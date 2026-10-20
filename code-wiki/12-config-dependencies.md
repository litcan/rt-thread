# 第 12 章 配置与依赖关系

> 本章回答两个问题：**一个源文件凭什么被编译进来？** 以及 **打开一个开关，为什么会连带打开十几个组件？**
>
> 第 11 章从"构建流程"的视角讲了 `.config → rtconfig.h → Project` 这条流水线。本章换到"配置项"的视角：Kconfig 该怎么写、`select` 与 `depends on` 到底谁说了算、组件之间的依赖关系是一张什么样的网，以及 SConscript 里那些五花八门的裁剪写法分别意味着什么。

---

## 12.1 两套裁剪机制：一个管"有没有"，一个管"编不编"

RT-Thread 的裁剪发生在**两个完全不同的时刻**，由**两套完全不同的语言**表达。把它们混为一谈，是配置类问题最常见的认知误区。

```
        Kconfig 文件树（声明式）
              │
              │ ① menuconfig / guiconfig 解析，用户勾选
              ▼
           .config                  ← 用户选择 + select 推导 + default 计算
              │
              │ ② mk_rtconfig() 逐行转换
              ▼
          rtconfig.h                ← C 预处理器可见的唯一事实来源
              │
              │ ③ PatchedPreProcessor 解析成 dict
              ▼
        BuildOptions(dict)
              │
              │ ④ GetDepend() / GetConfigValue()
              ▼
        SConscript（命令式 Python）
              │
              │ ⑤ DefineGroup() 登记
              ▼
         Projects[]  →  .o  →  .elf
```

| 阶段 | 时刻 | 输入 | 输出 | 决策者 | 能决定什么 |
|---|---|---|---|---|---|
| ① 配置解析 | 执行 `menuconfig` 时 | `Kconfig` 树 | `.config` | 用户 + `select`/`default` | 符号的**值** |
| ② 头文件生成 | 退出 `menuconfig` 时 | `.config` | `rtconfig.h` | `mk_rtconfig()` | 符号的**C 形态** |
| ③ 头文件解析 | `scons` 启动时 | `rtconfig.h` | `BuildOptions` | `PatchedPreProcessor` | 符号的**Python 视图** |
| ④⑤ 脚本求值 | `scons` 遍历时 | `SConscript` | `Projects` | `GetDepend`/`DefineGroup` | 哪些**文件**进编译 |

由此可以得出三个必须记住的结论：

1. **符号 `=y` 不等于源码被编译。** 例如 `RT_USING_DEVICE_IPC=y` 只保证 ipc 组存在，但 [drivers/ipc/SConscript:7-9](file:///workspace/components/drivers/ipc/SConscript#L7-L9) 还会在 `RT_USING_HEAP` 为假时摘掉 `dataqueue.c` 与 `pipe.c`。
2. **源码存在不等于被编译。** 目录里躺着 `.c` 文件，只要 `SConscript` 没把它放进 `src` 列表（或被 `SrcRemove` 摘掉），它就不会进镜像。
3. **`rtconfig.h` 是唯一契约。** `scons` 从不读 `.config`，它只读 `rtconfig.h`（[building.py:286-292](file:///workspace/tools/building.py#L286-L292)）。所以"我改了 `.config` 但没重新生成 `rtconfig.h`"必然无效。

---

## 12.2 Kconfig 语法速查

`Kconfig` 不是 RT-Thread 发明的，而是 Linux 内核那套语言；RT-Thread 在 `tools/kconfiglib/` 里内置了一份 Python 实现，`menukconfig.py` 通过 `import menuconfig` 调用它。

### 12.2.1 八种语句

| 语句 | 作用 | 本仓库实例 |
|---|---|---|
| `mainmenu "..."` | 整个配置树的根标题，一个工程只能有一个 | [bsp/qemu-vexpress-a9/Kconfig:1](file:///workspace/bsp/qemu-vexpress-a9/Kconfig#L1) |
| `menu "..."` / `endmenu` | 纯 UI 分组，**不产生任何依赖** | [src/Kconfig:2](file:///workspace/src/Kconfig#L2) |
| `config <SYM>` | 声明一个符号 | [src/Kconfig:4](file:///workspace/src/Kconfig#L4) |
| `menuconfig <SYM>` | 声明符号且该符号在菜单里可整块折叠 | [components/drivers/Kconfig:35](file:///workspace/components/drivers/Kconfig#L35) |
| `choice` / `endchoice` | 多选一，成员之间自动互斥 | [components/dfs/Kconfig:37-49](file:///workspace/components/dfs/Kconfig#L37-L49) |
| `if <EXPR>` / `endif` | 条件包含，块内所有符号都被加上依赖 | [components/Kconfig:32-41](file:///workspace/components/Kconfig#L32-L41) |
| `source "<path>"` | 包含另一个 Kconfig 文件 | [components/net/Kconfig:3-6](file:///workspace/components/net/Kconfig#L3-L6) |
| `# 注释` | 行注释 | 全仓通用 |

`menu` 与 `if` 的区别值得单独强调：

- `menu` 只是把若干个条目收进一个可折叠的界面分组，**不改变任何符号的可见性或取值**。
- `if` 是真正的依赖注入，等价于给块内每个 `config` 追加一条 `depends on`。

`components/Kconfig` 把两者用得很好：

```kconfig
if RT_USING_CONSOLE
source "$RTT_DIR/components/finsh/Kconfig"
endif

if !RT_USING_NANO
source "$RTT_DIR/components/dfs/Kconfig"
...
endif

if RT_USING_SMART
source "$RTT_DIR/components/lwp/Kconfig"
source "$RTT_DIR/components/mm/Kconfig"
endif
```

这段 `source` 的门控解释了为什么 Nano 版本"打开 menuconfig 看不到文件系统"——不是被隐藏，而是整个 Kconfig 文件根本没被解析，符号在配置树里不存在。

### 12.2.2 五种类型与五种属性

| 类型 | 取值形态 | 在 `rtconfig.h` 中的样子 | 典型用途 |
|---|---|---|---|
| `bool` | `y` / 未设置 | `#define X` / `/* X is not set */` | 功能开关 |
| `int` | 十进制整数 | `#define X 256` | 栈大小、缓冲长度 |
| `hex` | `0x...` | `#define X 0x50200` | 版本号、地址 |
| `string` | 带引号字符串 | `#define X "192.168.1.30"` | IP、设备名 |
| 无类型 | 只能被 `select` 或 `default` 赋值 | 同上 | 纯推导符号 |

| 属性 | 作用 |
|---|---|
| `prompt "..."` | 等价于 `bool "..."` 里的那个字符串；有 prompt 才能被用户直接改 |
| `default <值> [if <条件>]` | 默认值，可写多条，**第一条条件成立者胜出** |
| `range <min> <max> [if <条件>]` | 数值区间约束 |
| `help` / `---help---` | 帮助文本，紧随其后缩进书写 |
| `option env="VAR"` | 从环境变量取值，BSP 入口靠它定位源码树 |

`range` 的一个真实用例：[src/Kconfig:148-151](file:///workspace/src/Kconfig#L148-L151) 把空闲钩子链表长度限制在 1~16。

### 12.2.3 `choice` 的写法与"默认值也带条件"

`choice` 是唯一能表达"互斥"的结构。DFS v1/v2 的版本选择是教科书级例子：

```kconfig
choice
    prompt "The version of DFS"
    default RT_USING_DFS_V1
    default RT_USING_DFS_V2 if RT_USING_SMART

    config RT_USING_DFS_V1
        bool "DFS v1.0"
        depends on !RT_USING_SMART

    config RT_USING_DFS_V2
        bool "DFS v2.0"
        select RT_USING_DEVICE_OPS
endchoice
```

见 [components/dfs/Kconfig:37-49](file:///workspace/components/dfs/Kconfig#L37-L49)。三个细节：

1. `default` 可以写多条，第二条带 `if RT_USING_SMART`，于是"开了 Smart 就默认 v2"。
2. 成员里的 `depends on !RT_USING_SMART` 让 v1 在 Smart 下**不可选**，从机制上保证不会出现 v1+v2 同时成立。
3. `select RT_USING_DEVICE_OPS` 写在成员内部，意味着"选了 v2 就必须有 device ops"，这是一个单向强制。

### 12.2.4 书写规范

本仓库的 Kconfig 大体遵循以下约定，新写 Kconfig 时建议对齐：

- 4 空格缩进，`config` 顶格，属性缩进一级，`if` 块内再缩一级。
- `select` / `depends on` 统一放在类型声明之后、`default` 之前。
- 一个 `config` 只表达一件事；需要联动就用 `select`，不要写"用户必须自己再勾一个"。
- 没有 prompt 的 `config` 是**静默符号**，应在 `help` 或注释里写清"由谁 select"。
- 跨文件引用一律用 `$RTT_DIR` / `$BSP_DIR` / `$PKGS_DIR` 变量，不写相对路径。

**反例**：[libcpu/Kconfig:236-238](file:///workspace/libcpu/Kconfig#L236-L238) 把 `select` 写在了 `bool` 之前：

```kconfig
config ARCH_RISCV_FPU_S
    select ARCH_RISCV_FPU
    bool
```

解析器能容忍，但可读性差，且容易让人误以为这是个无类型符号。

---

## 12.3 配置树与 BSP 入口

### 12.3.1 根 Kconfig：只有 4 行

[Kconfig](file:///workspace/Kconfig#L1-L4) 全文：

```kconfig
source "$RTT_DIR/src/Kconfig"
source "$RTT_DIR/libcpu/Kconfig"
source "$RTT_DIR/components/Kconfig"
source "$RTT_DIR/examples/utest/testcases/Kconfig"
```

它是"组件视角"的入口：内核、架构、组件、测试用例四块。注意这里**没有** `mainmenu`——`mainmenu` 由 BSP 提供。

### 12.3.2 BSP 入口：`mainmenu` + 三个环境变量

[bsp/qemu-vexpress-a9/Kconfig](file:///workspace/bsp/qemu-vexpress-a9/Kconfig#L1-L20) 全文：

```kconfig
mainmenu "RT-Thread Project Configuration"

config BSP_DIR
    string
    option env="BSP_ROOT"
    default "."

config RTT_DIR
    string
    option env="RTT_ROOT"
    default "../.."

config PKGS_DIR
    string
    option env="PKGS_ROOT"
    default "packages"

source "$RTT_DIR/Kconfig"
source "$PKGS_DIR/Kconfig"
source "$BSP_DIR/drivers/Kconfig"
```

这段 20 行是整个配置体系的"接线盒"：

- `option env="BSP_ROOT"` 让 `BSP_DIR` 的值来自环境变量，`default "."` 是兜底。
- `source "$RTT_DIR/Kconfig"` 把 12.3.1 的根文件拉进来——**顺序很重要**，`src/` 在最前，因此内核符号先定义，组件后定义时可以引用它们。
- `source "$BSP_DIR/drivers/Kconfig"` 放在最后，让板级开关（`SOC_*`、`BSP_USING_*`）在主菜单底部出现，同时可以 `select` 前面定义的所有符号。

`PKGS_ROOT` 由 `menuconfig()` 主动设置：[menukconfig.py:256-258](file:///workspace/tools/menukconfig.py#L256-L258) 把 `ENV_ROOT/packages` 写进环境变量。

### 12.3.3 完整配置树

```
bsp/<board>/Kconfig            mainmenu + BSP_DIR/RTT_DIR/PKGS_DIR
├── $RTT_DIR/Kconfig
│   ├── src/Kconfig                RT-Thread Kernel
│   │   ├── menu "Inter-Thread communication"
│   │   ├── menu "Memory Management"
│   │   └── (device / console / version)
│   ├── libcpu/Kconfig             ARCH_* / CPU 端口 / MMU / GIC
│   └── components/Kconfig         RT-Thread Components
│       ├── finsh/Kconfig              (if RT_USING_CONSOLE)
│       ├── dfs/Kconfig                (if !RT_USING_NANO)
│       ├── fal/Kconfig
│       ├── drivers/Kconfig            Device Drivers
│       ├── libc/Kconfig
│       │   ├── compilers/common/Kconfig
│       │   ├── posix/Kconfig
│       │   └── cplusplus/Kconfig
│       ├── net/Kconfig
│       │   ├── sal/Kconfig
│       │   ├── netdev/Kconfig
│       │   ├── lwip/Kconfig
│       │   └── at/Kconfig
│       ├── mprotect/ utilities/ vbus/
│       └── lwp/ mm/                   (if RT_USING_SMART)
├── $PKGS_DIR/Kconfig             软件包（需 package 环境）
└── $BSP_DIR/drivers/Kconfig      SOC_* / BSP_USING_*
```

### 12.3.4 一个真实约束：`packages/Kconfig` 不在源码树里

`source "$PKGS_DIR/Kconfig"` 要求 `packages/Kconfig` 存在。但在纯 `git clone` 的源码树中，`packages/` 目录通常并不存在（它由 `pkgs --update` 或 Env 工具生成）。因此在没有 Env 环境时直接执行 `menuconfig`，可能在 `source` 阶段报找不到文件。

这不影响 `scons`：BSP 目录下已经有一份可用的 `rtconfig.h`，构建直接拿它用，完全绕过 Kconfig 解析。这也再次印证了 12.1 的结论——**构建不依赖 Kconfig**。

---

## 12.4 计算型符号：没有 prompt 的 `config`

有一类符号，用户永远在菜单里看不到，但它们是整张依赖网的关键节点。它们的值完全由 `select` 或 `default` 推导。理解这类符号，才能读懂"为什么我只开了一个开关，却多了一堆宏"。

### 12.4.1 形态一：纯 `select` 目标

[src/Kconfig:418-421](file:///workspace/src/Kconfig#L418-L421)：

```kconfig
config RT_USING_DEVICE_OPS
    bool "Using ops for each device object"
    depends on RT_USING_DEVICE
    default n
```

它有 prompt（用户理论上能改），但真正打开它的通常是别人：[components/dfs/Kconfig:48](file:///workspace/components/dfs/Kconfig#L48) 的 `RT_USING_DFS_V2 select RT_USING_DEVICE_OPS`。语义是"v2 的文件系统必须走 `rt_device_ops` 那套接口"。

### 12.4.2 形态二：默认值链

[src/Kconfig:404-410](file:///workspace/src/Kconfig#L404-L410)：

```kconfig
config RT_USING_HEAP
    bool
    default n if RT_USING_NOHEAP
    default y if RT_USING_SMALL_MEM
    default y if RT_USING_SLAB
    default y if RT_USING_MEMHEAP_AS_HEAP
    default y if RT_USING_USERHEAP
```

这是"内存管理器选择"的收口：前面 `choice` 让用户在小内存/SLAB/memheap/用户堆/无堆之间选一个，这里再把选择结果折叠成单一布尔量 `RT_USING_HEAP`，供 `#ifdef RT_USING_HEAP` 使用。

链式 `default` 的语义是**从上往下匹配，第一个条件成立者生效**。所以 `RT_USING_NOHEAP` 必须排在最前面，否则其他分支的 `y` 会先命中。

### 12.4.3 形态三：条件默认（无 prompt 的推导）

[src/Kconfig:429-439](file:///workspace/src/Kconfig#L429-L439)：

```kconfig
config RT_USING_THREADSAFE_PRINTF
    bool "Enable thread safe kernel print service"
    default y if RT_USING_SMP && RT_USING_SMART

config RT_USING_SCHED_THREAD_CTX
    bool "Using the scheduler thread context"
    help
        Using the scheduler thread context embedded in the thread object.
        This options is only for backward compatible codes. Maybe use as a
        mandatory option in the future.
    default y if RT_USING_SMP
```

这两条把"多核必须怎样"固化进了配置系统：SMP 下调度器上下文必须内嵌到线程对象里；SMP + Smart 下打印必须加锁。用户不需要知道这些细节，也不该能关掉它们。

同样的手法见 [src/Kconfig:143-145](file:///workspace/src/Kconfig#L143-L145)：`RT_USING_IDLE_HOOK` 的 `default y if RT_USING_HOOK`，而 `RT_USING_HOOK` 又 `select RT_USING_IDLE_HOOK`——两条路径指向同一结果，形成冗余保险。

### 12.4.4 形态四：引用另一个符号作为默认值

[src/Kconfig:161-164](file:///workspace/src/Kconfig#L161-L164)：

```kconfig
config SYSTEM_THREAD_STACK_SIZE
    int "The stack size of system thread (for defunct etc.)"
    depends on RT_USING_SMP
    default IDLE_THREAD_STACK_SIZE
```

`default` 的值可以是一个符号名而非字面量。结果是：用户改了空闲线程栈大小，系统线程栈跟着变，除非显式覆盖。这种"默认值跟随"比硬编码 1024 更不容易出错。

---

## 12.5 `select` 与 `depends on` 的语义与陷阱

这两个关键字是 Kconfig 里最容易用错的一对。

### 12.5.1 语义对比

| 维度 | `depends on` | `select` |
|---|---|---|
| 依赖方向 | 我依赖别人（我需要它） | 别人因我而开（我强制它） |
| 书写位置 | 写在**被依赖方**？不，写在**依赖方** | 写在**触发方** |
| 效果 | 条件不成立时，本符号不可见/不可选 | 条件成立时，目标符号被写成 `y` |
| 是否写回 `.config` | 否 | 是 |
| 传递性 | 不传递，只约束本符号 | 逐级传递（A select B，B select C） |
| 典型用途 | 表达"没有 DFS 就别谈 POSIX 文件接口" | 表达"开了 lwIP 就必须有 device 和 IPC" |

一句话记法：**`depends on` 是"我要"，`select` 是"你必须"**。

### 12.5.2 陷阱一：`select` 会绕过目标的 `depends on`

`RT_USING_LWIP` 的第一行就 `select RT_USING_DEVICE`（[components/net/lwip/Kconfig:3](file:///workspace/components/net/lwip/Kconfig#L3)），而 `RT_USING_DEVICE` 有 `depends on !RT_USING_NANO`（[src/Kconfig:413-416](file:///workspace/src/Kconfig#L413-L416)）。

如果某个 BSP 在 Nano 配置下又 `select RT_USING_LWIP`，就会出现：

- `.config` 里 `CONFIG_RT_USING_DEVICE=y`；
- 但 menuconfig 里 `RT_USING_DEVICE` 这一项因为依赖不满足而**根本看不到**。

kconfiglib 在写 `.config` 时可能给出 `unmet direct dependencies` 一类警告，但**不会阻止写入**。所以看到"符号存在但菜单里找不到"，第一反应应该是查它被谁 `select` 了。

### 12.5.3 陷阱二：条件 `select`

[components/net/lwip/Kconfig:5](file:///workspace/components/net/lwip/Kconfig#L5)：

```kconfig
select SAL_USING_LWIP if RT_USING_SAL
```

含义是"如果系统里有 SAL，就把 lwIP 注册成 SAL 的一个协议栈实现；如果没 SAL，就不要凭空造一个 SAL 开关"。这比无条件 `select SAL_USING_LWIP` 正确得多——后者会把 SAL 的半套符号拉进一个根本不用 SAL 的工程。

同样，`RT_USING_LWIP_IPV6` 里有 `select NETDEV_IPV6_SCOPES if RT_USING_LWIP212`（[lwip/Kconfig:51](file:///workspace/components/net/lwip/Kconfig#L51)），表达"只有 2.1.2 版本才有这个能力"。

**写 `select` 时先问自己：这个目标在什么前提下才成立？** 有前提就加 `if`。

### 12.5.4 陷阱三：`depends on !X` 与 `choice` 的配合

`depends on` 支持取反，常用来做互斥。DFS 的例子（12.2.3）已经展示过：`RT_USING_DFS_V1 depends on !RT_USING_SMART`。

再看 SAL 的 socket 数量（[components/net/sal/Kconfig](file:///workspace/components/net/sal/Kconfig)）：

```kconfig
config SAL_SOCKETS_NUM
    int "The number of sockets"
    depends on !SAL_USING_POSIX
    default 16
```

含义是"如果已经走 POSIX socket（由 libc 提供 fd 管理），就不再需要 SAL 自己的 socket 表"。这是一个"二选一"的隐式表达——用 `depends on !X` 代替 `choice`，代价是可读性稍差，好处是不用新增 `choice` 块。

### 12.5.5 陷阱四：循环依赖

`select` 是逐级传递的，一旦形成环，结果取决于解析顺序，而不是逻辑。典型链路：

```
RT_USING_SMART
  ├─ select RT_USING_DFS          (src/Kconfig:25)
  ├─ select RT_USING_POSIX_FS     (src/Kconfig:34)
  └─ select RT_USING_LWP          (src/Kconfig:24)

RT_USING_POSIX_FS
  ├─ select RT_USING_DFS          (posix/Kconfig:5)
  └─ select DFS_USING_POSIX       (posix/Kconfig:6)

RT_USING_DFS
  └─ select RT_USING_MUTEX        (dfs/Kconfig:5)
```

这条链目前是单向的（SMART → POSIX → DFS），没有环。但如果哪天给 `RT_USING_DFS` 加上 `select RT_USING_POSIX_FS`，就立刻成环：SMART→DFS→POSIX_FS→DFS。

Kconfig **不会报错**，只会让符号的最终值依赖解析顺序。所以维护依赖时，建议在纸上画出 `select` 的箭头方向，确认它是一张有向无环图。

### 12.5.6 陷阱五：重复定义

`libcpu/Kconfig` 里有两个符号被定义了两次：

| 符号 | 第一次 | 第二次 |
|---|---|---|
| `ARCH_ARM_CORTEX_R` | [L57](file:///workspace/libcpu/Kconfig#L57) | [L111](file:///workspace/libcpu/Kconfig#L111) |
| `ARCH_CPU_STACK_GROWS_UPWARD` | [L36](file:///workspace/libcpu/Kconfig#L36) | [L267](file:///workspace/libcpu/Kconfig#L267) |

第二次定义会与第一次合并：**`select` 和 `default` 会叠加，但 prompt 与类型以某一次为准**。实际后果是"看起来加了个 `select`，实际上把前面那条 `select` 的顺序打乱了"，排查时极易误判。新增符号前先 `grep -n "^config <SYM>$"` 全树确认，是低成本的自我保护。

---

## 12.6 组件依赖矩阵

下表是"查表式"的速查矩阵。**"被谁 select"一列是反向索引**——当你想知道"我打开 X 会不会连带打开 Y"时，从这一列找。

### 12.6.1 内核与架构

| 组件 | 关键符号 | `depends on` | 被谁 `select` | DefineGroup 组名 | 源码位置 |
|---|---|---|---|---|---|
| 内核核心 | —（无总开关） | — | — | `Kernel` | [src/](file:///workspace/src) |
| 设备对象 | `RT_USING_DEVICE` | `!RT_USING_NANO` | `SMART`、`LWIP`、`SERIAL` 及少量 BSP | `Kernel` | `src/device.c` |
| 设备 ops | `RT_USING_DEVICE_OPS` | `RT_USING_DEVICE` | `DFS_V2` | `Kernel` | `src/device.c` |
| SMP | `RT_USING_SMP` | — | BSP | `Kernel` | `src/scheduler_mp.c` |
| Smart | `RT_USING_SMART` | `ARCH_ARM_CORTEX_M`\|`ARM9`\|`CORTEX_A`\|`ARMV8`\|`RISCV64` 之一，且 `!RT_USING_NANO` | BSP | `lwP`、`mm` | `components/lwp`、`components/mm` |
| 内核堆 | `RT_USING_HEAP` | —（默认链推导） | — | `Kernel` | `src/mem.c`、`slab.c` 等 |
| 架构符号 | `ARCH_*` | — | `SOC_*`、`CORTEX_*` | `libcpu` | `libcpu/<arch>` |

### 12.6.2 设备驱动

| 组件 | 关键符号 | `depends on` | 被谁 `select` | DefineGroup 组名 | 源码位置 |
|---|---|---|---|---|---|
| 驱动框架 | `RT_USING_DEVICE` | `!RT_USING_NANO` | 见上 | `DeviceDrivers` | `components/drivers/core` |
| 设备 IPC | `RT_USING_DEVICE_IPC` | — | `SERIAL`、`LWIP` | `DeviceDrivers` | `components/drivers/ipc` |
| 串口 | `RT_USING_SERIAL` | — | BSP | `DeviceDrivers` | `components/drivers/serial` |
| SPI | `RT_USING_SPI` | — | BSP | `DeviceDrivers` | `components/drivers/spi` |
| SPI 软件模拟 | `RT_USING_SPI_BITOPS` | — | 部分 BSP（引脚配置为软件 SPI 时） | `DeviceDrivers` | 同上 |
| I2C | `RT_USING_I2C` | — | BSP | `DeviceDrivers` | `components/drivers/i2c` |
| PIN | `RT_USING_PIN` | — | `SPI_BITOPS`、`SOFT_I2C`、`SENSOR` 及大量 BSP | `DeviceDrivers` | `components/drivers/pin` |
| 设备树模型 | `RT_USING_DM` | — | BSP | `DeviceDrivers` | `components/drivers/core` |
| ktime | `RT_USING_KTIME` | — | `SMART`、`POSIX_DELAY`、`POSIX_TIMER` | `ktime` | `components/drivers/ktime` |
| RTC | `RT_USING_RTC` | — | `SMART`、`CPLUSPLUS11` 及大量 BSP | `DeviceDrivers` | `components/drivers/rtc` |

### 12.6.3 文件系统与 POSIX

| 组件 | 关键符号 | `depends on` | 被谁 `select` | DefineGroup 组名 | 源码位置 |
|---|---|---|---|---|---|
| DFS | `RT_USING_DFS` | — | `SMART`、`POSIX_FS`、`SPI_MSD`、`ULOG_BACKEND_USING_FILE` | `Filesystem` | `components/dfs` |
| DFS POSIX 接口 | `DFS_USING_POSIX` | `RT_USING_DFS`（`if` 块） | `POSIX_FS` | `Filesystem` | 同上 |
| DFS v1 | `RT_USING_DFS_V1` | `!RT_USING_SMART` | —（`choice` 默认） | `Filesystem` | `components/dfs/dfs_v1` |
| DFS v2 | `RT_USING_DFS_V2` | — | —（`choice` 默认 if SMART） | `Filesystem` | `components/dfs/dfs_v2` |
| devfs | `RT_USING_DFS_DEVFS` | `RT_USING_DFS`（`if` 块） | `POSIX_DEVIO` | `Filesystem` | `dfs_v*/filesystems/devfs` |
| elmfat | `RT_USING_DFS_ELMFAT` | `RT_USING_DFS`（`if` 块） | — | `Filesystem` | `dfs_v*/filesystems/elmfat` |
| POSIX 文件接口 | `RT_USING_POSIX_FS` | — | `SMART`、`CPLUSPLUS11`、`POSIX_IPC` | `POSIX` | `components/libc/posix` |
| POSIX socket | `RT_USING_POSIX_SOCKET` | — | BSP EMAC | `POSIX` | 同上 |
| pthread | `RT_USING_PTHREADS` | — | `CPLUSPLUS11` | `POSIX` | `components/libc/posix/pthreads` |
| libc | `RT_USING_LIBC` | — | — | `Compiler` | `components/libc/compilers` |
| C++ | `RT_USING_CPLUSPLUS` | — | — | `CPP` | `components/libc/cplusplus` |
| C++11 | `RT_USING_CPLUSPLUS11` | `RT_USING_CPLUSPLUS`（`if` 块） | — | `CPP` | `components/libc/cplusplus/cpp11` |

### 12.6.4 网络与工具

| 组件 | 关键符号 | `depends on` | 被谁 `select` | DefineGroup 组名 | 源码位置 |
|---|---|---|---|---|---|
| SAL | `RT_USING_SAL` | — | `POSIX_SOCKET`、`AT_USING_SOCKET` | `SAL` | `components/net/sal` |
| netdev | `RT_USING_NETDEV` | — | `SAL` 及大量 BSP | `SAL`（注意不是 `netdev`） | `components/net/netdev` |
| lwIP | `RT_USING_LWIP` | — | BSP EMAC、`ENC28J60`/`SPI_WIFI`、`WLAN_PROT_LWIP` | `lwIP` | `components/net/lwip` |
| AT | `RT_USING_AT` | — | 少量 BSP | `AT` | `components/net/at` |
| FinSH | `RT_USING_FINSH` | `RT_USING_CONSOLE`（`source` 门控） | — | `Finsh` | `components/finsh` |
| FAL | `RT_USING_FAL` | — | — | `Fal` | `components/fal` |
| ulog | `RT_USING_ULOG` | — | — | `Utilities` | `components/utilities/ulog` |
| utest | `RT_USING_UTEST` | — | — | `UTest` | `components/utilities/utest` |
| VBus | `RT_USING_VBUS` | — | — | `VBus` | `components/vbus` |
| legacy | `RT_USING_LEGACY` | — | — | `Legacy` | `components/legacy` |
| 内存保护 | `RT_USING_MEM_PROTECTION` | — | `HW_STACK_GUARD` | `mprotect` | `components/mprotect` |

### 12.6.5 十条最常用的 `select` 边

| 触发符号 | 被强制打开 | 位置 |
|---|---|---|
| `SOC_VEXPRESS_A9` | `ARCH_ARM_CORTEX_A9`、`RT_USING_CACHE`、`ARCH_ARM_MMU`、`RT_USING_COMPONENTS_INIT`、`RT_USING_USER_MAIN`、`RT_USING_GIC_V2` | [drivers/Kconfig:3-11](file:///workspace/bsp/qemu-vexpress-a9/drivers/Kconfig#L3-L11) |
| `ARCH_ARM_CORTEX_A9` | `ARCH_ARM_CORTEX_A` | [libcpu/Kconfig:186-189](file:///workspace/libcpu/Kconfig#L186-L189) |
| `ARCH_ARM_CORTEX_A` | `ARCH_ARM`、`ARCH_MM_MMU`、`RT_USING_CPU_FFS`、`RT_USING_HW_ATOMIC` | [libcpu/Kconfig:149-155](file:///workspace/libcpu/Kconfig#L149-L155) |
| `ARCH_ARMV8` | `ARCH_ARM`、`ARCH_MM_MMU`、`RT_USING_CPU_FFS` | [libcpu/Kconfig:204-209](file:///workspace/libcpu/Kconfig#L204-L209) |
| `ARCH_ARM_MMU` | `RT_USING_CACHE`、`ARCH_MM_MMU` | [libcpu/Kconfig:120-125](file:///workspace/libcpu/Kconfig#L120-L125) |
| `RT_USING_SMP` | `RT_USING_SCHED_THREAD_CTX` | [src/Kconfig:66-69](file:///workspace/src/Kconfig#L66-L69) |
| `RT_USING_HOOK` | `RT_USING_IDLE_HOOK` | [src/Kconfig:122-125](file:///workspace/src/Kconfig#L122-L125) |
| `RT_USING_SIGNALS` | `RT_USING_MEMPOOL` | [src/Kconfig:279-281](file:///workspace/src/Kconfig#L279-L281) |
| `RT_USING_SMART` | 14 个符号（LWP/DFS/POSIX_*/DEVICE/NULL/ZERO/RANDOM/RTC/KTIME/STDC_ATOMIC） | [src/Kconfig:21-39](file:///workspace/src/Kconfig#L21-L39) |
| `RT_USING_DFS_V2` | `RT_USING_DEVICE_OPS` | [dfs/Kconfig:46-48](file:///workspace/components/dfs/Kconfig#L46-L48) |

---

## 12.7 SConscript 裁剪的六种模式

Kconfig 决定"符号在不在"，SConscript 决定"文件编不编"。本仓库的 SConscript 里，裁剪有六种固定写法。

### 模式一：桥接递归（bridge）

[components/SConscript](file:///workspace/components/SConscript#L1-L17) 全文：

```python
import os
from building import *

cwd = GetCurrentDir()
objs = []
list = os.listdir(cwd)

Import('remove_components')

for d in list:
    path = os.path.join(cwd, d)
    if os.path.isfile(os.path.join(path, 'SConscript')):
        if d in remove_components:
            continue
        objs = objs + SConscript(os.path.join(d, 'SConscript'))

Return('objs')
```

特点：**不含任何 `GetDepend`**，只负责"发现子目录并把返回值拼起来"。

关键点是 `remove_components`：它由 `SConstruct` 传给 `PrepareBuilding()`，用来在启动阶段把某些顶层目录整体拉黑。**这也是唯一一种"绕过 Kconfig 的裁剪"**——即使 `RT_USING_FOO=y`，只要 `foo` 在 `remove_components` 里，它的 SConscript 根本不会被求值。

### 模式二：架构 / 平台分派

[libcpu/SConscript](file:///workspace/libcpu/SConscript#L1-L15)：

```python
Import('rtconfig')

cwd   = GetCurrentDir()
group = []
list  = os.listdir(cwd)

if rtconfig.ARCH in list:
    group = group + SConscript(os.path.join(rtconfig.ARCH, 'SConscript'))
```

[libcpu/arm/SConscript](file:///workspace/libcpu/arm/SConscript#L11-L16) 再分派一层：

```python
# add common code files
group = SConscript(os.path.join('common', 'SConscript'))

# cpu porting code files
if rtconfig.CPU in list:
    group = group + SConscript(os.path.join(rtconfig.CPU, 'SConscript'))
```

特点：用 `rtconfig.ARCH` / `rtconfig.CPU` / `rtconfig.PLATFORM` 这三个**字符串**做分派，而不是 `GetDepend`。它们来自 `rtconfig.h` 里的 `#define ARCH "arm"` 一类宏。

同一个模式在 [components/lwp/SConscript](file:///workspace/components/lwp/SConscript#L1-L46) 里更复杂：用 `support_arch` / `platform_file` 两个字典把 `PLATFORM`、`ARCH`、`CPU` 映射到具体源文件，再对 risc-v 做了 `cpu` 大小写修正。

**为什么不用 `GetDepend`？** 因为 `ARCH`/`CPU` 是字符串宏，`GetDepend('ARCH_ARM')` 判断的是布尔宏。两者用途不同：布尔宏回答"是不是 ARM"，字符串宏回答"具体是 arm 下的哪个 CPU"。

### 模式三：文件级移除

[src/SConscript](file:///workspace/src/SConscript#L1-L30) 用 `SrcRemove` 做同组内的加减：

```python
if not GetDepend(['RT_USING_MEMHEAP']):
    SrcRemove(src, ['memheap.c'])

if GetDepend(['RT_USING_SMP']) == False:
    SrcRemove(src, ['cpu_mp.c', 'scheduler_mp.c'])
else:
    SrcRemove(src, ['cpu_up.c', 'scheduler_up.c'])
```

特点：先 `Glob('*.c')` 拿到全集，再按条件摘除。优点是新增文件不用改脚本；缺点是"文件存在但没编"这件事变得非常隐蔽。

`SrcRemove` 的另一个用例在 [components/mm/SConscript:11-12](file:///workspace/components/mm/SConscript#L11-L12)：

```python
if not GetDepend('RT_USING_MEMBLOCK'):
    SrcRemove(src, ['mm_memblock.c'])
```

### 模式四：早退 `Return`

[components/drivers/serial/SConscript](file:///workspace/components/drivers/serial/SConscript#L1-L24)：

```python
if not GetDepend(['RT_USING_SERIAL']):
    Return('group')
...
group = DefineGroup('DeviceDrivers', src, depend=[''], CPPPATH=CPPPATH)

Return('group')
```

特点：在文件开头就把整个 SConscript 短路掉，连 `DefineGroup` 都不执行。

**为什么要早退，而不是靠 `DefineGroup(depend=...)`？** 因为中间那些 `if GetDepend(...)` 分支可能引用了未定义的变量或做了不必要的 `Glob`。早退能避免这些副作用，也让脚本意图更直白。

### 模式五：互斥分支

dfs_v1 与 dfs_v2 是同一组件的两个实现，靠两个 SConscript 的守卫条件互斥：

```python
# components/dfs/dfs_v1/SConscript
if GetDepend('RT_USING_DFS') and not GetDepend('RT_USING_DFS_V2'):
    ...
    group = DefineGroup('Filesystem', src, depend=['RT_USING_DFS'], CPPPATH=CPPPATH)

# components/dfs/dfs_v2/SConscript
group = DefineGroup('Filesystem', src,
                    depend=['RT_USING_DFS', 'RT_USING_DFS_V2'], CPPPATH=CPPPATH)
if GetDepend(['RT_USING_DFS', 'RT_USING_DFS_V2']):
    ...  # 递归子目录
```

见 [dfs_v1/SConscript](file:///workspace/components/dfs/dfs_v1/SConscript#L1-L25) 与 [dfs_v2/SConscript](file:///workspace/components/dfs/dfs_v2/SConscript#L1-L22)。

注意 v2 的 `DefineGroup` 里写的是 `depend=['RT_USING_DFS', 'RT_USING_DFS_V2']`——列表是**逻辑与**，两个都成立才登记。这比在 Python 里写 `if a and b` 更省事，因为 `DefineGroup` 内部会做判断。

### 模式六：依赖守卫 `if` 包裹

[components/mm/SConscript](file:///workspace/components/mm/SConscript#L1-L24)：

```python
if GetDepend('ARCH_ARM_CORTEX_A') or GetDepend('ARCH_ARMV8') or GetDepend('ARCH_RISCV64'):
    src = Glob('*.c') + Glob('*_gcc.S')
    ...
    group = DefineGroup('mm', src, depend=['ARCH_MM_MMU'], CPPPATH=CPPPATH)

    list = os.listdir(cwd)
    for d in list:
        path = os.path.join(cwd, d)
        if os.path.isfile(os.path.join(path, 'SConscript')):
            group = group + SConscript(os.path.join(path, 'SConscript'))

Return('objs')
```

特点：外层 `if` 决定"这个目录要不要整体处理"，内层 `DefineGroup(depend=...)` 再兜一道。两层条件必须同时满足，`mm` 才会出现在最终工程里。

### 六种模式对照

| 模式 | 判定手段 | 作用粒度 | 典型文件 | 风险点 |
|---|---|---|---|---|
| 桥接递归 | `os.listdir` + `remove_components` | 整个顶层目录 | `components/SConscript` | 目录被拉黑后无任何提示 |
| 架构分派 | `rtconfig.ARCH/CPU/PLATFORM` | 整个架构/CPU 目录 | `libcpu/SConscript` | 字符串拼错则静默不编 |
| 文件级移除 | `SrcRemove` | 单个 `.c` | `src/SConscript` | 文件存在但不编，难发现 |
| 早退 | `Return()` | 整个 SConscript | `drivers/serial` | 早退后 `group` 为未定义/空 |
| 互斥分支 | `GetDepend` + `not` | 两套实现二选一 | `dfs_v1` / `dfs_v2` | 守卫写漏则两个都编，符号冲突 |
| 依赖守卫 | `if` 包裹 `DefineGroup` | 整个目录 | `mm/SConscript` | 外层条件与 `depend` 不一致 |

---

## 12.8 Group 机制与作用域

### 12.8.1 `DefineGroup` 干了三件事

第 11 章已经拆过它的六条显式行为，这里从"作用域"角度补三点：

1. **依赖闸门**：`GetDepend(depend)` 为假时直接 `return []`，该组连 `Projects` 都进不去。
2. **参数注入**：`CFLAGS` / `CPPPATH` / `CPPDEFINES` 等走 `Env.AppendUnique`，写进**全局构建环境**；`LOCAL_*` 只存进 group 字典。
3. **同名合并**：同名组走 `MergeGroup`，源文件取并集。这是 `DeviceDrivers` 能由三十来个 SConscript 拼成一个组的原因。

### 12.8.2 全局参数 vs `LOCAL_*`

| 全局键 | 局部键 | 生效范围 | 全仓使用情况 |
|---|---|---|---|
| `CFLAGS` | `LOCAL_CFLAGS` | 全局：之后所有组 | 核心树 3 处，`bsp/` 25 处 |
| `CPPPATH` | `LOCAL_CPPPATH` | 全局：之后所有组 | 极少 |
| `CPPDEFINES` | `LOCAL_CPPDEFINES` | 全局：之后所有组 | 核心树 2 处，`bsp/` 7 处 |
| `LINKFLAGS` | `LOCAL_LINKFLAGS` | 全局 / 单组 | — |

`src/` 与 `components/` 里 `LOCAL_CFLAGS` 只有三个使用者，且每个都有明确理由：

- [src/SConscript:36-44](file:///workspace/src/SConscript#L36-L44)：给内核源码加一组严格告警（`-Wmissing-prototypes`、`-Wimplicit-fallthrough` 等）。这些告警如果全局生效，会把组件和三方库全部炸掉。
- [components/finsh/SConscript:15](file:///workspace/components/finsh/SConscript#L15)：`-Wstack-usage=<FINSH_THREAD_STACK_SIZE>`，**用配置值反过来约束编译期检查**：

  ```python
  LOCAL_CFLAGS += ' -Wstack-usage=' + str(GetConfigValue('FINSH_THREAD_STACK_SIZE'))
  ```

  这是 `GetConfigValue` 最典型的用法——它返回的是 `rtconfig.h` 里宏的**字符串**，可以直接拼进命令行。
- [components/drivers/spi/SConscript:34-36](file:///workspace/components/drivers/spi/SConscript#L34-L36)：给 SFUD 加 `-std=c99` / `--c99`，因为这份第三方代码用了 C99 语法。

`LOCAL_CPPDEFINES` 在核心树的两处是"源码归属标记"：[src/SConscript:50](file:///workspace/src/SConscript#L50) 的 `__RT_KERNEL_SOURCE__` 和 [components/drivers/ipc/SConscript:11](file:///workspace/components/drivers/ipc/SConscript#L11) 的 `__RT_IPC_SOURCE__`。

而 `bsp/` 下 25 个 `LOCAL_CFLAGS` 使用者几乎都是同一个用途：给厂商 HAL 库关掉告警、换 C 标准。它们把"脏活"限制在单个组里，说明 `LOCAL_*` 的真正价值在于**隔离不可控的第三方代码**。

### 12.8.3 `Env.AppendUnique` 的泄漏问题

`DeviceDrivers` 组由三十多个 SConscript 合并而成。如果其中任何一个在 `DefineGroup` 里传了**全局** `CFLAGS`，这个 flag 会通过 `Env.AppendUnique` 永久留在构建环境里，影响**之后求值**的所有组。

由于 SConscript 的求值顺序依赖 `os.listdir` 的返回顺序（在多数文件系统上是字典序，但不保证），这种泄漏的后果是"部分文件编译参数不一致"，且很难复现。

**实践建议**：组件级 SConscript 一律用 `LOCAL_*`；只有确实需要全局生效（如 `-mfloat-abi`、`-mcpu`）时才写全局键。

### 12.8.4 命名现状

`components/` 下 `DefineGroup` 共 106 处调用（全仓含 BSP 与测试用例共 1413 处），组名分布在三个层次：

| 层次 | 组名 |
|---|---|
| 系统层 | `Kernel`、`libcpu`、`DeviceDrivers`、`Filesystem`、`POSIX`、`Compiler`、`CPP`、`Finsh`、`lwIP`、`SAL`、`AT`、`Fal`、`VBus`、`Legacy`、`Utilities`、`LIBADT`、`UTest`、`FDT`、`ktime`、`rt-link`、`rt_usbd`、`rt_usbh`、`mm`、`lwP`、`lwp-riscv`、`lwp-x86-i386`、`mprotect` |
| BSP 层 | `Drivers`、`Applications`、`Libraries`、`CMSIS`、`xip`、`video` |
| 测试层 | `utestcases`、`rtt_posix_testcase` |

几个值得注意的不一致（读代码时容易踩）：

- `components/lwp/SConscript` 用的是 **`lwP`**（大写 P），而 arch 子目录用 `lwp-riscv` / `lwp-x86-i386`。
- `components/net/netdev/SConscript` 把组名写成 **`SAL`**，和 `components/net/sal` 合并，而不是 `netdev`。
- `libcpu/arm/cortex-r52/SConscript` 用的是 **`CPU`**，其余 libcpu 目录都是 `libcpu`。
- `components/fal` → `Fal`，`components/utilities/libadt` → `LIBADT`，大小写风格不统一。

组名只是字符串标识，**不参与依赖判定**（`GetDepend` 只看宏）。所以拼写不一致不会导致功能错误，但会让 `--buildlib=<组名>` 和 IDE 工程里的分组名变得难以猜测。

---

## 12.9 实例：从 `SOC_VEXPRESS_A9=y` 推导编译集合

以 `bsp/qemu-vexpress-a9` 为例，走一遍完整推导。这个 BSP 的 `SOC_VEXPRESS_A9` 默认就是 `y`（[drivers/Kconfig:3-11](file:///workspace/bsp/qemu-vexpress-a9/drivers/Kconfig#L3-L11)）。

### 第一步：板级 `select` 直接展开

```kconfig
config SOC_VEXPRESS_A9
    bool
    select ARCH_ARM_CORTEX_A9
    select RT_USING_CACHE
    select ARCH_ARM_MMU
    select RT_USING_COMPONENTS_INIT
    select RT_USING_USER_MAIN
    select RT_USING_GIC_V2
    default y
```

一个开关，带出 6 个符号。

### 第二步：架构 `select` 逐级传递

```
SOC_VEXPRESS_A9
├─ ARCH_ARM_CORTEX_A9
│   └─ ARCH_ARM_CORTEX_A
│       ├─ ARCH_ARM
│       ├─ ARCH_MM_MMU
│       ├─ RT_USING_CPU_FFS
│       └─ RT_USING_HW_ATOMIC
├─ ARCH_ARM_MMU
│   ├─ RT_USING_CACHE
│   └─ ARCH_MM_MMU（冗余，已由上面带上）
├─ RT_USING_CACHE
├─ RT_USING_COMPONENTS_INIT
├─ RT_USING_USER_MAIN
└─ RT_USING_GIC_V2
```

`ARCH_ARM_CORTEX_A` 的定义见 [libcpu/Kconfig:149-155](file:///workspace/libcpu/Kconfig#L149-L155)，`ARCH_ARM_CORTEX_A9` 见 [L186-189](file:///workspace/libcpu/Kconfig#L186-L189)。这两跳是典型的"层层收窄"：板子 → 具体核 → 核家族 → 通用 ARM。

### 第三步：默认值补齐

关键符号中默认值就是 `y` 的那几个——它们不需要任何人 `select`，只要没被用户关掉就在编译集合里：

| 符号 | 来源 | 原因 |
|---|---|---|
| `RT_USING_DEVICE` | [src/Kconfig:413-416](file:///workspace/src/Kconfig#L413-L416) | `default y`（且非 Nano），另被 `RT_USING_SERIAL` 再 `select` 一次 |
| `RT_USING_DEVICE_IPC` | [drivers/Kconfig:10-12](file:///workspace/components/drivers/Kconfig#L10-L12) | `default y`，另被 `RT_USING_SERIAL` 再 `select` 一次 |
| `RT_USING_SERIAL` | [drivers/Kconfig:35-39](file:///workspace/components/drivers/Kconfig#L35-L39) | `default y` |
| `RT_USING_DFS` | [dfs/Kconfig:3-6](file:///workspace/components/dfs/Kconfig#L3-L6) | `default y` |
| `DFS_USING_POSIX` | [dfs/Kconfig:11-13](file:///workspace/components/dfs/Kconfig#L11-L13) | `default y` |

### 第四步：`select` 与默认值的相互作用

`RT_USING_SERIAL` 会 `select RT_USING_DEVICE_IPC` 和 `RT_USING_DEVICE`——即使 `RT_USING_DEVICE_IPC` 的默认值改成 `n`，开了串口仍然会被强制打开。这就是 12.5.1 表格里"写回 `.config`"的实际含义。

`RT_USING_DFS_V2` 属于 `choice`（[dfs/Kconfig:37-49](file:///workspace/components/dfs/Kconfig#L37-L49)），它的默认其实是 `RT_USING_DFS_V1`，只有 `RT_USING_SMART` 时才默认 v2。但这个 BSP 的 `.config` 明确写着 `CONFIG_RT_USING_DFS_V2=y`——**BSP 自带的 `.config` 是快照，优先级高于 Kconfig 默认值**。一旦选中 v2，它 `select RT_USING_DEVICE_OPS`，于是 `.config` 里出现 `CONFIG_RT_USING_DEVICE_OPS=y`，尽管用户在菜单里从未点过它。

### 第五步：最终参与编译的组

把上面的符号代入 12.7 的六种模式，逐个 SConscript 判断：

| SConscript | 判定条件 | 结果 | 理由 |
|---|---|---|---|
| `src/SConscript` | 无条件 | ✅ `Kernel` | 内核永远编 |
| `libcpu/SConscript` | `rtconfig.ARCH == "arm"` | ✅ `libcpu` | 分派到 `arm/SConscript` |
| `libcpu/arm/SConscript` | `rtconfig.CPU == "cortex-a"` | ✅ `libcpu` | 分派到 `cortex-a/SConscript` |
| `components/SConscript` | 遍历 | ✅ 递归 | — |
| `components/drivers/core` | `RT_USING_DEVICE` | ✅ `DeviceDrivers` | `=y` |
| `components/drivers/ipc` | `RT_USING_DEVICE_IPC` | ✅ `DeviceDrivers` | `=y` |
| `components/drivers/serial` | 早退检查 `RT_USING_SERIAL` | ✅ `DeviceDrivers` | `=y` |
| `components/drivers/spi` | `RT_USING_SPI` | ✅ `DeviceDrivers` | `.config` 里 `=y`（Kconfig 默认 `n`，由 BSP 配置快照打开） |
| `components/drivers/i2c` | `RT_USING_I2C` | ✅ `DeviceDrivers` | 同上 |
| `components/drivers/pin` | `RT_USING_PIN` | ✅ `DeviceDrivers` | 同上 |
| `components/dfs/dfs_v2` | `RT_USING_DFS && RT_USING_DFS_V2` | ✅ `Filesystem` | `.config` 已选 v2 |
| `components/dfs/dfs_v1` | `RT_USING_DFS && !RT_USING_DFS_V2` | ❌ | v2 已选，互斥 |
| `components/dfs/dfs_v2/filesystems/elmfat` | `RT_USING_DFS && RT_USING_DFS_ELMFAT` | ✅（若 elmfat 开） | — |
| `components/libc/compilers/newlib` | `rtconfig.PLATFORM` | ✅ `Compiler` | gcc 走 newlib |
| `components/libc/posix/io/*` | 各自符号 | 部分 ✅ | `POSIX_FS` 默认 y |
| `components/finsh` | `RT_USING_FINSH` | ✅（若开） | `RT_USING_CONSOLE` 门控 |
| `components/net/lwip` | `RT_USING_LWIP` | ❌ | 默认 `n` |
| `components/net/sal` | `RT_USING_SAL` | ❌ | 默认 `n` |
| `components/lwp` | `RT_USING_SMART` | ❌ | Smart 未开 |
| `components/mm` | `ARCH_MM_MMU` + 外层 `if` | ⚠️ | 有 MMU，但 `mm` 目录只在 `RT_USING_SMART` 下被 `source`/递归；本 BSP 未开 Smart，故不编 |
| `components/mprotect` | `RT_USING_MEM_PROTECTION` | ❌ | 未开 |
| `components/fal` | `RT_USING_FAL` | ❌ | 未开 |
| `bsp/qemu-vexpress-a9/drivers` | `depend=['']` | ✅ `Drivers` | BSP 驱动永远编，内部再按 `BSP_USING_*` 裁 |

这张表就是"配置 → 编译集合"的完整映射。注意 `mm` 那一行：符号层面 `ARCH_MM_MMU=y` 成立，理论上 `mm/SConscript` 的 `DefineGroup` 会成功，但 `components/mm` 目录本身只在 `RT_USING_SMART` 时被 `source` 进配置树、并被 `components/SConscript` 递归——**这就是 Kconfig 与 SConscript 双重裁剪的典型案例**。

---

## 12.10 常见配置问题排查表

| 现象 | 可能原因 | 定位方法 | 修复 |
|---|---|---|---|
| 改了 `Kconfig` 默认值，退出 menuconfig 后 `rtconfig.h` 没变 | `.config` 未变，`mk_rtconfig` 被 md5 门控跳过（[menukconfig.py:266-277](file:///workspace/tools/menukconfig.py#L266-L277)） | `diff .config .config.old` | 删掉 `.config.old`，或改动任意一项使 `.config` 变化 |
| 执行 `scons` 报 `No such file: rtconfig.h` | 从未执行过 `menuconfig`，或手工删了头文件 | `ls rtconfig.h` | 执行一次 `menuconfig`（即使不改任何项） |
| `menuconfig` 打不开配置树 | `source "$PKGS_DIR/Kconfig"` 找不到文件 | 看异常里缺失的路径 | 配置 Env 环境 / `pkgs --update` 生成 `packages/` |
| 开了 `X=y`，但对应源文件没编 | ① `DefineGroup` 的 `depend` 不满足；② `SrcRemove` 摘掉了；③ SConscript 早退；④ 父目录没被递归 | 在该 SConscript 里临时 `print(GetDepend(...))`；或 `scons --verbose` 看是否加载了该目录 | 按 12.7 的六种模式逐条对照 |
| 符号在 `.config` 里有，菜单里却找不到 | 被 `select` 强制打开，但自身 `depends on` 不满足 | `grep -rn "select <SYM>" --include=Kconfig` | 补齐依赖，或改用条件 `select ... if ...` |
| 打开 A 却意外带开 B | A 有 `select B` | `grep -n "select B" -r --include=Kconfig` | 若不应联动，改成 `select B if 条件` |
| `Kconfig` 里明明写了 `select`，`.config` 里没有 | 该符号被**重复定义**，后一次的 `default` 覆盖了 | `grep -rn "^config <SYM>$" --include=Kconfig` | 删除重复定义 |
| 链接报 `undefined reference to xxx` | 符号开了但组没进 `Projects`；或 `.a` 未被链接 | `scons --verbose` 看链接命令行是否含该组 | 检查 `DefineGroup` 的 `depend` 与早退条件 |
| 同一份代码在不同 BSP 上编译参数不同 | 某个 SConscript 用了全局 `CFLAGS`，`Env.AppendUnique` 泄漏 | `grep -rn "CFLAGS *=" --include=SConscript components/` | 改为 `LOCAL_CFLAGS` |
| `GetConfigValue` 返回 `None` | 该宏在 `rtconfig.h` 里不存在（符号未定义，或名字拼错） | `grep <SYM> rtconfig.h` | 确认符号名与 Kconfig 中的 `config` 名完全一致 |
| `rtconfig.h` 里的版本号与 `Kconfig` 默认值不一致 | BSP 的 `rtconfig.h` 是历史快照，未随源码升级 | 对比 [src/Kconfig:456-460](file:///workspace/src/Kconfig#L456-L460) 的 `default 0x50200` 与 BSP 头文件 | 重新执行 `menuconfig` 生成，或手工同步 |

### 一条通用的排查顺序

```
① 符号在 .config 里吗？        → 不在：查 Kconfig 依赖/是否被 source
② 符号在 rtconfig.h 里吗？     → 不在：菜单退出时是否真的重新生成
③ GetDepend 认得它吗？         → 不认得：名字大小写、是否 bool
④ SConscript 被加载了吗？      → 没加载：父目录守卫、remove_components
⑤ DefineGroup 返回值非空吗？   → 空：depend 不满足或提前 Return
⑥ 文件在 src 列表里吗？        → 不在：Glob 模式、SrcRemove
⑦ 链接命令行里有这个 .o 吗？   → 没有：DoBuilding 的 LOCAL_* 过滤逻辑
```

这七步覆盖了从"配置项"到"目标文件"的全部环节，每一环都可以独立验证。

---

## 12.11 小结

本章的核心可以压缩成四句话：

1. **两套裁剪、两个时刻**：Kconfig 在配置期决定符号的值，SConscript 在构建期决定文件的去留；二者通过 `rtconfig.h` 单向连接，构建从不回头读 `.config`。
2. **`select` 是强制，`depends on` 是许可**：前者写值、可传递、会绕过后者的约束；后者只控制可见性。写 `select` 时优先加 `if` 条件，写完检查是否成环。
3. **六种 SConscript 裁剪模式**：桥接递归、架构分派、文件级移除、早退、互斥分支、依赖守卫。看到一个组件没编进来，就按这六种逐一排除。
4. **组名只是标签**：`DeviceDrivers` 由三十多个脚本合并，`lwP` 的大小写是个历史遗留；真正决定编译的是 `GetDepend`，不是组名。

至此，十二个章节走完了 RT-Thread v5.2.0 的主干：

```
架构与对象  →  调度与时钟  →  IPC 与内存  →  中断与启动
     ↓
设备框架  →  组件层  →  libcpu/BSP  →  构建
     ↓
配置与依赖（本章）——把前面所有模块串成一张可裁剪的网
```

读到这里，再回头看 `bsp/qemu-vexpress-a9` 的 `.config`，每一行 `=y` 背后都能指认出：是哪个 `select` 带出来的、哪个 `default` 补上的、最终由哪个 `SConscript` 落成目标文件。这张"配置 → 依赖 → 编译"的完整链路，就是 RT-Thread 能够同时跑在 8 位 MCU 和 64 位多核应用处理器上的根本原因。
