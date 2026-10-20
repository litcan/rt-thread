# 6. 内存管理

RT-Thread 提供四类动态内存分配器，外加页级分配与 MMU 地址空间管理（Smart）。理解它们的关键是：**`rt_malloc()` 是一个“可替换入口”**，具体走哪套算法由 `rtconfig.h` 中的宏决定。

相关源码：

- [kservice.c](file:///workspace/src/kservice.c)：`rt_malloc/realloc/calloc/free`、堆入口、钩子、对齐分配
- [mem.c](file:///workspace/src/mem.c)：小内存管理（`rt_smem_*`）
- [memheap.c](file:///workspace/src/memheap.c)：多区域内存堆（`rt_memheap_*`）
- [slab.c](file:///workspace/src/slab.c)：Slab 分配器（`rt_slab_*`）
- [mempool.c](file:///workspace/src/mempool.c)：固定块内存池（`rt_mp_*`）
- `components/mm/`：页表、地址空间、匿名映射（Smart）

---

## 6.1 全局堆入口与选择机制

### 6.1.1 对外 API

```c
void  *rt_malloc(rt_size_t size);
void  *rt_realloc(void *ptr, rt_size_t newsize);
void  *rt_calloc(rt_size_t count, rt_size_t size);
void   rt_free(void *ptr);
void  *rt_malloc_align(rt_size_t size, rt_size_t align);
void   rt_free_align(void *ptr);

void   rt_system_heap_init(void *begin_addr, void *end_addr);
void   rt_memory_info(rt_size_t *total, rt_size_t *used, rt_size_t *max_used);
```

这些函数在 [kservice.c](file:///workspace/src/kservice.c) 中声明为 `rt_weak`，允许 BSP 或应用**重写**（例如接入外部 SDRAM 或 TLSF 分配器）。

### 6.1.2 编译期选择

`_MEM_INIT/_MEM_MALLOC/_MEM_FREE/_MEM_REALLOC/_MEM_INFO` 宏根据配置切换实现：

```c
#if defined(RT_USING_SMALL_MEM_AS_HEAP)
    system_heap = rt_smem_init("heap", start, size);
    #define _MEM_MALLOC(size) rt_smem_alloc(system_heap, size)
#elif defined(RT_USING_MEMHEAP_AS_HEAP)
    rt_memheap_init(&system_heap, "heap", start, size);
    #define _MEM_MALLOC(size) _memheap_alloc(&system_heap, size)
#elif defined(RT_USING_SLAB_AS_HEAP)
    system_heap = rt_slab_init("heap", start, size);
    #define _MEM_MALLOC(size) rt_slab_alloc(system_heap, size)
#else
    /* 未启用堆：rt_malloc 返回 RT_NULL */
#endif
```

| 配置 | 堆实现 | 典型场景 |
| --- | --- | --- |
| `RT_USING_SMALL_MEM_AS_HEAP` | 小内存算法 | 默认，资源受限 MCU |
| `RT_USING_MEMHEAP_AS_HEAP` | memheap | 多块不连续 RAM（如内部 SRAM + 外部 SDRAM） |
| `RT_USING_SLAB_AS_HEAP` | slab | 频繁分配/释放、追求性能 |
| 都未定义 | 无堆 | 仅静态分配，`rt_malloc` 恒返回 `NULL` |

> 注意：`RT_USING_SMALL_MEM` 只是**编译**小内存模块，`RT_USING_SMALL_MEM_AS_HEAP` 才把它**作为全局堆**。两者可独立配置。

### 6.1.3 堆初始化

```c
static void _rt_system_heap_init(void *begin_addr, void *end_addr)
{
    rt_ubase_t begin_align = RT_ALIGN((rt_ubase_t)begin_addr, RT_ALIGN_SIZE);
    rt_ubase_t end_align   = RT_ALIGN_DOWN((rt_ubase_t)end_addr, RT_ALIGN_SIZE);

    RT_ASSERT(end_align > begin_align);
    _MEM_INIT("heap", (void *)begin_align, end_align - begin_align);
    _heap_lock_init();
}
```

- 起止地址按 `RT_ALIGN_SIZE` 对齐（默认 8 字节）；
- 必须在创建任何动态对象**之前**调用（通常由 `rt_hw_board_init()` 完成）；
- `rt_system_heap_init()` 是 `rt_weak`，BSP 可重写以支持多区域初始化。

### 6.1.4 并发保护与钩子

```c
static rt_base_t _heap_lock(void);   /* SMP：自旋锁；UP：调度锁/关中断 */
static void     _heap_unlock(rt_base_t level);
```

- `rt_malloc/free` 全程持堆锁，保证多线程安全；
- **不要在堆锁内做阻塞操作**（当前实现不会，但重写 `rt_malloc` 时需注意）；
- 每个分配/释放都会调用钩子：

```c
void rt_malloc_sethook(void (*hook)(void **ptr, rt_size_t size));
void rt_free_sethook(void (*hook)(void *ptr));
void rt_realloc_sethook(void (*hook)(void **ptr, rt_size_t size));
```

可用于内存泄漏检测、越界哨兵、分配统计。

---

## 6.2 小内存管理（small memory）

### 数据结构

```c
struct rt_small_mem_item
{
    rt_ubase_t pool_ptr;   /* 所属 rt_small_mem 指针 + 最低位 used 标志 */
    rt_size_t  next;       /* 下一个空闲块偏移（相对 heap_ptr） */
    rt_size_t  prev;       /* 上一个空闲块偏移 */
#ifdef RT_USING_MEMTRACE
    rt_uint8_t thread[4 或 8];  /* 分配线程名，便于追踪 */
#endif
};

struct rt_small_mem
{
    struct rt_memory          parent;    /* 统计信息 total/used/max */
    rt_uint8_t               *heap_ptr;
    struct rt_small_mem_item *heap_end;
    struct rt_small_mem_item *lfree;     /* 最近发现的空闲块，加速分配 */
    rt_size_t                 mem_size_aligned;
};
```

- 每个内存块前有块头，`pool_ptr` 的最低位表示“已用/空闲”；
- 空闲块之间用 `next/prev` 组成**双向空闲链表**；
- 已用块的 `next` 记录块结束偏移，用于计算块大小；
- `lfree` 指向最近一次释放/分裂产生的空闲块，分配时从这里开始找，减少遍历。

### 关键算法

**分配 `rt_smem_alloc()`**：

1. 对齐请求大小（加块头、最小块限制）；
2. 从 `lfree` 开始沿空闲链表找第一个足够大的块（first fit）；
3. 若剩余空间还能构成一个最小块，则**分裂**：前半给用户，后半成为新空闲块；
4. 标记 `used`，更新 `lfree`，返回块头之后的数据地址。

**释放 `rt_smem_free()`**：

1. 由指针反推块头，校验 `used` 位；
2. 标记为空闲；
3. 调用 `plug_holes()` 与**前后相邻空闲块合并**，避免碎片；
4. 更新 `lfree` 与统计信息。

**重分配 `rt_smem_realloc()`**：

- 缩小：原地分裂；
- 扩大：若后邻块空闲且足够则合并，否则 `malloc + memcpy + free`。

### 特点

| 优点 | 缺点 |
| --- | --- |
| 实现简单、可移植 | 空闲链表遍历，最坏 O(n) |
| 支持任意大小 | 长期运行易产生外部碎片 |
| 可开启 memtrace 定位泄漏 | 块头有额外开销（约 3 个机器字） |

---

## 6.3 多区域内存堆（memheap）

### 适用场景

当系统有**多块不连续的 RAM**（内部 SRAM、CCM、外部 SDRAM），小内存算法只能管理一块连续区域，此时用 memheap：

```c
static struct rt_memheap heap1;
static struct rt_memheap heap2;

rt_memheap_init(&heap1, "sram", (void *)0x20000000, 128 * 1024);
rt_memheap_init(&heap2, "sdram", (void *)0xC0000000, 8 * 1024 * 1024);
```

### 数据结构

```c
struct rt_memheap_item
{
    rt_uint32_t            magic;      /* 魔数，检测越界/野指针 */
    struct rt_memheap_item *next;      /* 空闲链表 next */
    struct rt_memheap_item *prev;
    struct rt_memheap_item *next_free; /* 物理相邻块 */
    struct rt_memheap_item *prev_free;
    struct rt_memheap      *pool_ptr;
    rt_uint8_t              pool_id;
    /* ... */
};

struct rt_memheap
{
    struct rt_memory  parent;
    rt_uint32_t       *start_addr;
    rt_uint32_t        pool_size;
    struct rt_memheap_item *header_free_list;
    /* ... */
};
```

### 分配策略

- `rt_memheap_alloc()` 使用 **fast fit**：从空闲链表头开始找第一个满足大小的块；
- 块头带 `magic`，释放时校验，可发现重复释放与越界写；
- 支持 `rt_memheap_realloc()`、`rt_memheap_free()`；
- `rt_memheap_info()` 返回总量/已用/峰值。

### 特点

| 优点 | 缺点 |
| --- | --- |
| 支持多块不连续内存 | 元数据比 small mem 大 |
| 带 magic 校验，易调试 | 仍是 first-fit，碎片问题存在 |
| 可独立管理多个堆 | 需要为每个区域显式初始化 |

---

## 6.4 Slab 分配器

### 设计思想

Slab 将内存按**固定大小档位**切分，分配时直接从对应档位的空闲链表取块，避免遍历与分裂，适合大量小对象的高频分配。

### 结构

```c
struct rt_slab_zone
{
    struct rt_slab_zone *next;       /* 下一个 zone */
    rt_size_t            size;       /* 本 zone 的块大小 */
    rt_size_t            page_count; /* 占用页数 */
    /* 空闲块链表等 */
};

struct rt_slab
{
    struct rt_memory      parent;
    struct rt_slab_zone  *zones;     /* 按大小分档 */
    rt_uint8_t           *heap_ptr;
    /* ... */
};
```

### API

```c
rt_slab_t rt_slab_init(const char *name, void *begin_addr, rt_size_t size);
void     *rt_slab_alloc(rt_slab_t slab, rt_size_t size);
void     *rt_slab_realloc(rt_slab_t slab, void *ptr, rt_size_t size);
void      rt_slab_free(rt_slab_t slab, void *ptr);
```

### 页分配配合

Slab 扩容时通过页分配器获取内存：

```c
void *rt_page_alloc(rt_size_t npages);
void  rt_page_free(void *addr, rt_size_t npages);
```

- `rt_page_alloc()` 以页为单位向系统申请（页大小由架构/配置决定）；
- 页分配器之上还可以构建 MMU 映射（`components/mm`）。

### 特点

| 优点 | 缺点 |
| --- | --- |
| 分配/释放接近 O(1) | 内部碎片（按档位取整） |
| 适合小对象、高频场景 | 元数据与页管理复杂 |
| 不易产生外部碎片 | 内存占用峰值较高 |

---

## 6.5 固定块内存池（mempool）

### 适用场景

任务间传递固定大小数据块、需要**确定性分配时间**且**不允许碎片**的场景。内存池在初始化时一次性切分好，运行时只做链表摘挂。

### 数据结构

```c
struct rt_mempool
{
    struct rt_object parent;
    void            *start_address;
    rt_size_t        size;
    rt_size_t        block_size;
    rt_size_t        block_total_count;
    rt_size_t        block_free_count;
    rt_list_t        block_list;      /* 空闲块链表 */
    rt_list_t        suspend_thread;  /* 等待线程 */
};
```

### API

```c
rt_err_t rt_mp_init(rt_mempool_t mp, const char *name, void *start, rt_size_t size,
                    rt_size_t block_size);
rt_mempool_t rt_mp_create(const char *name, rt_size_t block_count, rt_size_t block_size);
rt_err_t rt_mp_delete(rt_mempool_t mp);
rt_err_t rt_mp_detach(rt_mempool_t mp);
void    *rt_mp_alloc(rt_mempool_t mp, rt_int32_t time);
void     rt_mp_free(void *block);
```

### 行为

- 初始化时把内存切成 `block_total_count` 个块，每块前部存放指向下一空闲块的指针；
- `rt_mp_alloc(mp, time)`：有空闲块直接返回；否则按 `time` 阻塞等待（`RT_WAITING_FOREVER` 永久等待）；
- `rt_mp_free()`：块归还空闲链表并唤醒等待线程；
- `rt_mp_alloc` 可在中断中调用（`time = 0`），`rt_mp_free` 也可在 ISR 中使用。

> 注意：块大小会被 `RT_ALIGN_SIZE` 对齐；每个块额外需要一个指针空间，实际内存占用为 `count * (block_size + sizeof(void*))`。

---

## 6.6 对齐分配

DMA、Cache 行、SIMD 等场景要求缓冲区按特定边界对齐：

```c
void *rt_malloc_align(rt_size_t size, rt_size_t align);
void  rt_free_align(void *ptr);
```

实现方式：多申请 `align + sizeof(void*)` 字节，把返回地址向上对齐，并在前一个机器字保存原始指针；`rt_free_align()` 读取原始指针后释放。

> 注意：`rt_free_align()` 只能释放 `rt_malloc_align()` 的返回值，二者不可混用。

---

## 6.7 堆统计与调试

```c
rt_size_t total, used, max_used;
rt_memory_info(&total, &used, &max_used);
```

MSH 命令：

| 命令 | 作用 |
| --- | --- |
| `free` | 查看系统堆总量/已用/峰值 |
| `memcheck` | 小内存堆完整性检查（需 `RT_USING_MEMCHECKER`） |
| `memtrace` | 按线程统计分配（需 `RT_USING_MEMTRACE`） |
| `memheaptrace` | memheap 版本的分配追踪 |

内存泄漏排查建议：

1. 打开 `RT_USING_MEMTRACE`，用 `memtrace` 看哪个线程分配最多；
2. 注册 `rt_malloc_sethook` 记录每次分配的大小与调用栈；
3. 用 `rt_memory_info()` 在关键路径前后对比 `used`；
4. 检查是否在中断中误用 `rt_malloc`（应使用内存池）。

---

## 6.8 Smart 的 MMU 内存管理

在 `RT_USING_SMART` 下，`components/mm/` 提供进程地址空间能力：

| 文件 | 职责 |
| --- | --- |
| `mm_aspace.c` | 地址空间创建/销毁、区域映射 |
| `mm_page.c` | 物理页分配与引用计数 |
| `mm_anon.c` | 匿名页（按需分页） |
| `mm_memblock.c` | 物理内存块管理 |
| `mm_fault.c` | 缺页异常处理 |
| `mm_kmem.c` | 内核虚拟内存分配 |
| `ioremap.c` | 外设寄存器映射 |

配合 `components/lwp/`，每个用户进程拥有独立的 `rt_aspace`，通过页表实现隔离；`rt_page_alloc/free` 是这一层的物理页来源之一。

---

## 6.9 选型对照

| 分配器 | 分配复杂度 | 碎片 | 元数据开销 | 适用 |
| --- | --- | --- | --- | --- |
| small mem | O(n) | 外部碎片 | 中 | 通用、内存紧张 |
| memheap | O(n) | 外部碎片 | 较大 | 多块不连续 RAM |
| slab | 近似 O(1) | 内部碎片 | 中 | 高频小对象 |
| mempool | O(1) | 无 | 每块一个指针 | 固定大小、确定性场景 |

实践建议：

1. MCU 默认用 small mem 即可；
2. 有外部 SDRAM 且需要统一管理时用 memheap；
3. 网络/文件系统等高频小对象可评估 slab；
4. 线程间传数据块优先用 mempool 或消息队列，避免频繁 `malloc/free`；
5. 嵌入式系统应尽量**静态分配**，动态分配只用于启动阶段或数量可控的对象。

---

## 6.10 常见问题

1. **堆未初始化就 `rt_malloc`**：返回 `NULL` 或触发断言。确认 `rt_hw_board_init()` 中调用了 `rt_system_heap_init()`。
2. **堆与栈重叠**：链接脚本中 `HEAP_BEGIN/HEAP_END` 必须避开 `.bss`、栈和已用 RAM。
3. **中断中 `rt_malloc`**：多数配置下堆锁不可在 ISR 获取，禁止使用；改用 mempool。
4. **反复 `realloc` 导致碎片**：尽量预估大小一次性分配，或使用内存池。
5. **`rt_free` 野指针**：memheap 的 magic、small mem 的 used 位能捕获部分错误，但无法覆盖所有越界；建议开启 memtrace/哨兵。
6. **`rt_malloc_align` 与 `rt_free` 混用**：会破坏堆，必须成对使用 `rt_free_align`。
