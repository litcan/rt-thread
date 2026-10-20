# 11. 构建与运行

RT-Thread 的构建体系由三部分组成：

- **Kconfig**：描述“有哪些配置项、默认值、依赖关系”，产出 `.config`；
- **SCons + Python 脚本**：把 `.config` 转换成 `rtconfig.h`，再据此决定“编译哪些文件、带哪些宏与头文件路径”；
- **BSP 的 `rtconfig.py` / 链接脚本**：提供工具链、编译选项、链接参数与内存布局。

本章按“配置 → 收集 → 编译 → 链接 → 运行”的顺序，说明一次 `scons` 到底做了什么。

相关源码与脚本：

- [bsp/qemu-vexpress-a9/SConstruct](file:///workspace/bsp/qemu-vexpress-a9/SConstruct)：BSP 构建入口
- [tools/building.py](file:///workspace/tools/building.py)：构建核心库
- [tools/options.py](file:///workspace/tools/options.py)：`scons` 命令行选项
- [tools/menukconfig.py](file:///workspace/tools/menukconfig.py)：menuconfig 前端与 `rtconfig.h` 生成
- [tools/kconfiglib/](file:///workspace/tools/kconfiglib)：Kconfig 解析库
- [bsp/qemu-vexpress-a9/rtconfig.py](file:///workspace/bsp/qemu-vexpress-a9/rtconfig.py)：工具链与编译/链接选项
- [bsp/qemu-vexpress-a9/rtconfig.h](file:///workspace/bsp/qemu-vexpress-a9/rtconfig.h)：由配置生成的实际宏集合
- [bsp/qemu-vexpress-a9/link.lds](file:///workspace/bsp/qemu-vexpress-a9/link.lds)：链接脚本
- [bsp/qemu-vexpress-a9/qemu.sh](file:///workspace/bsp/qemu-vexpress-a9/qemu.sh)：QEMU 运行脚本

---

## 11.1 构建系统组成

| 角色 | 文件/目录 | 作用 |
| --- | --- | --- |
| 工程入口 | `bsp/<board>/SConstruct` | 创建 SCons `Environment`，调用 `PrepareBuilding` / `DoBuilding` |
| 构建框架 | `tools/building.py` | 解析 `rtconfig.h`、递归 `SConscript`、定义编译组、链接、生成 IDE 工程 |
| 命令行参数 | `tools/options.py` | 注册 `--target`、`--menuconfig`、`--buildlib`、`--strict` 等 |
| 工具链描述 | `bsp/<board>/rtconfig.py` | `ARCH/CPU/PLATFORM/CC/CFLAGS/LFLAGS/POST_ACTION` 等 |
| 配置系统 | `Kconfig`、`.config`、`rtconfig.h` | 功能裁剪与宏定义 |
| 模块清单 | 各目录 `SConscript` | 用 `DefineGroup()` 声明“本目录要编译哪些源文件” |
| 链接布局 | `bsp/<board>/link.lds` | 代码/数据段、自动初始化段、符号表段的位置 |
| 工程生成器 | `tools/keil.py`、`iar.py`、`cmake.py`、`vsc.py` … | 把收集到的 `Projects` 转成 IDE 工程文件 |

整体数据流：

```mermaid
flowchart LR
    K["Kconfig 树"] -->|menuconfig / guiconfig / defconfig| C[".config"]
    C -->|mk_rtconfig| H["rtconfig.h"]
    H -->|SCons PreProcessor 解析| B["BuildOptions 字典"]
    B -->|GetDepend| S["各目录 SConscript"]
    S -->|DefineGroup| P["Projects 编译组"]
    P -->|DoBuilding| O["Object 文件"]
    O -->|Env.Program + link.lds| E["rtthread.elf"]
    E -->|POST_ACTION| X["rtthread.bin / size 统计"]
```

---

## 11.2 一次构建的完整流程

以 `bsp/qemu-vexpress-a9` 为例，执行 `scons -j4` 后依次发生：

1. **SCons 读取 `SConstruct`**，导入 `rtconfig.py`，得到 `ARCH`、`CPU`、`CC`、`CFLAGS`、`LFLAGS` 等；
2. 创建 `Environment`，并把 `RTT_ROOT/tools` 加入 `sys.path`，从而 `from building import *`；
3. 调用 [PrepareBuilding](file:///workspace/tools/building.py#L123-L403)：
   - 注册命令行选项；
   - 设置 `RTT_DIR`、`BSP_DIR`、`PKGS_DIR` 等环境变量；
   - 处理 `--target` 带来的工具链切换；
   - 自动探测/校验工具链路径；
   - 解析 `rtconfig.h` 得到 `BuildOptions`；
   - 依次执行 BSP、`src`、`libcpu`、`components`、`utest` 的 `SConscript`，收集 `objs`；
4. 每个 `SConscript` 内部用 `GetDepend()` 判断配置，用 `DefineGroup()` 把源文件、头文件路径、局部编译选项登记为一个“组”；
5. 调用 [DoBuilding](file:///workspace/tools/building.py#L781-L855)：合并对象、处理 `LOCAL_*` 选项、`Env.Program()` 生成链接目标；
6. 调用 [EndBuilding](file:///workspace/tools/building.py#L938-L988)：挂接 `POST_ACTION`、可选生成 IDE 工程、可选打包；
7. SCons 执行编译与链接，最后运行 `POST_ACTION`（如 `objcopy -O binary`、`size`）。

BSP 的 `SConstruct` 非常短，只做“环境准备 + 两次调用”：

```python
TARGET = 'rtthread.' + rtconfig.TARGET_EXT

DefaultEnvironment(tools=[])
env = Environment(tools = ['mingw'],
    AS   = rtconfig.AS, ASFLAGS = rtconfig.AFLAGS,
    CC   = rtconfig.CC, CFLAGS = rtconfig.CFLAGS,
    CXX  = rtconfig.CXX, CXXFLAGS = rtconfig.CXXFLAGS,
    AR   = rtconfig.AR, ARFLAGS = '-rc',
    LINK = rtconfig.LINK, LINKFLAGS = rtconfig.LFLAGS)

objs = PrepareBuilding(env, RTT_ROOT)
DoBuilding(TARGET, objs)
```

对应文件：[SConstruct](file:///workspace/bsp/qemu-vexpress-a9/SConstruct#L13-L36)。

> 注意：`RT_USING_SMART` 打开时，`SConstruct` 会把 `link.lds` 替换成 `link_smart.lds`（[SConstruct#L31-L33](file:///workspace/bsp/qemu-vexpress-a9/SConstruct#L31-L33)），因为用户态与内核态需要不同的地址布局。

### 11.2.1 构建目录

`PrepareBuilding` 为 BSP 与内核使用了不同的 `variant_dir`（[building.py#L378-L401](file:///workspace/tools/building.py#L378-L401)）：

```python
bsp_vdir    = 'build'
kernel_vdir = 'build/kernel'

objs = SConscript('SConscript', variant_dir=bsp_vdir, duplicate=0)
objs.extend(SConscript(Rtt_Root + '/src/SConscript', variant_dir=kernel_vdir + '/src', duplicate=0))
objs.extend(SConscript(Rtt_Root + '/libcpu/SConscript', variant_dir=kernel_vdir + '/libcpu', duplicate=0))
objs.extend(SConscript(Rtt_Root + '/components/SConscript', variant_dir=kernel_vdir + '/components', duplicate=0))
```

- `duplicate=0`：只在 `build/` 下放 `.o`，不复制源码；
- BSP 与内核分开，避免 BSP 自带的同名组件与内核组件互相覆盖；
- `SConscript` 的**执行顺序**决定了 `Projects` 中组的登记顺序，但最终链接顺序还受组名排序与 SCons 依赖图影响。

---

## 11.3 配置阶段：从 Kconfig 到 rtconfig.h

### 11.3.1 配置文件层次

顶层 [Kconfig](file:///workspace/Kconfig#L1-L4) 只做 `source`：

```
source "$RTT_DIR/src/Kconfig"
source "$RTT_DIR/libcpu/Kconfig"
source "$RTT_DIR/components/Kconfig"
source "$RTT_DIR/examples/utest/testcases/Kconfig"
```

BSP 的 [Kconfig](file:///workspace/bsp/qemu-vexpress-a9/Kconfig#L1-L20) 则定义 `BSP_DIR`/`RTT_DIR`/`PKGS_DIR` 三个环境变量型符号，再引入根 Kconfig、软件包 Kconfig、板级驱动 Kconfig：

```
mainmenu "RT-Thread Project Configuration"

config BSP_DIR
    string
    option env="BSP_ROOT"
    default "."

config RTT_DIR
    string
    option env="RTT_ROOT"
    default "../.."

source "$RTT_DIR/Kconfig"
source "$PKGS_DIR/Kconfig"
source "$BSP_DIR/drivers/Kconfig"
```

因此**配置树 = 内核 + libcpu + components + 板级驱动 + 软件包**。

### 11.3.2 配置命令

| 命令 | 前端 | 行为 |
| --- | --- | --- |
| `scons --menuconfig` | `tools/kconfiglib/menuconfig.py` | 终端菜单配置，生成/更新 `.config` 与 `rtconfig.h` |
| `scons --pyconfig` | `guiconfig.py` | 图形化配置 |
| `scons --pyconfig-silent` | `defconfig.py` | 按 Kconfig 默认值静默生成 `.config`，并强制重写 `rtconfig.h` |
| `scons --genconfig` | 反向解析 | 从现有 `rtconfig.h` 生成 `.config` |
| `scons --useconfig=<file>` | `mk_rtconfig` | 用指定配置文件生成 `rtconfig.h` |
| `scons --target=...` | `GenTargetProject` | 不编译，只生成 IDE/工程文件 |

这些参数在 [options.py](file:///workspace/tools/options.py#L104-L138) 注册，在 `PrepareBuilding` 中被分派（[building.py#L330-L359](file:///workspace/tools/building.py#L330-L359)）。

### 11.3.3 `.config` → `rtconfig.h` 的映射规则

[menukconfig.mk_rtconfig](file:///workspace/tools/menukconfig.py#L47-L103) 的转换非常直接：

| `.config` 行 | `rtconfig.h` 结果 |
| --- | --- |
| `CONFIG_RT_USING_FINSH=y` | `#define RT_USING_FINSH` |
| `CONFIG_RT_NAME_MAX=8` | `#define RT_NAME_MAX 8` |
| `CONFIG_RT_CONSOLE_DEVICE_NAME="uart0"` | `#define RT_CONSOLE_DEVICE_NAME "uart0"` |
| `# CONFIG_RT_USING_SMP is not set` | `/* RT_USING_SMP is not set */` |
| `CONFIG_PKG_XXX_PATH=...` / `_VER` | 跳过，不进入 `rtconfig.h` |

同时会补上 `RT_CONFIG_H__` 头保护，并在文件末尾加入：

```c
#include "rtconfig_project.h"
```

因此 BSP 若有少量“配置之外的工程私有宏”，可以写在 `rtconfig_project.h` 中，不会被 `menuconfig` 覆盖。

实际产物可参考 [rtconfig.h](file:///workspace/bsp/qemu-vexpress-a9/rtconfig.h#L1-L20)：`#define RT_TICK_PER_SECOND 100`、`#define RT_USING_DFS_V2` 等都以宏的形式出现，C 代码里直接 `#ifdef` 使用。

> 注意：BSP 仓库里携带的 `.config` / `rtconfig.h` 是“上次生成的快照”。例如该 BSP 的 `RT_VER_NUM` 为 `0x50100`，而当前内核 Kconfig 默认值已是 `0x50200`。重新执行 `menuconfig` 后这些值会按当前 Kconfig 刷新。

---

## 11.4 PrepareBuilding 详解

### 11.4.1 环境准备

[PrepareBuilding](file:///workspace/tools/building.py#L123-L403) 首先固定三类路径：

| 变量 | 来源 | 用途 |
| --- | --- | --- |
| `RTT_ROOT` / `RTT_DIR` | `PrepareBuilding` 的 `root_directory` | 定位内核、libcpu、components |
| `BSP_ROOT` / `BSP_DIR` | `Dir('#').abspath` | 定位当前 BSP 的 `rtconfig.h`、驱动 |
| `PKGS_DIR` | `ENV_ROOT` 或 `$HOME/.env/packages` | 软件包 Kconfig 与源码 |

随后把 `tools/` 与 `tools/kconfiglib/` 加入 `sys.path`，使 `menuconfig`、`building` 等模块可直接导入。

### 11.4.2 工具链处理

`--target` 会同时改变 `CROSS_TOOL` 与 `PLATFORM`，映射表在 [building.py#L156-L174](file:///workspace/tools/building.py#L156-L174)：

| `--target` | `CROSS_TOOL` | `PLATFORM` |
| --- | --- | --- |
| `mdk` / `mdk4` / `mdk5` | `keil` | `armcc` |
| `iar` | `iar` | `iccarm` |
| `vs` / `vs2012` | `msvc` | `cl` |
| `vsc` / `ua` / `cdk` / `makefile` / `eclipse` / `ses` / `cmake` / `xmake` / `codelite` | `gcc` | `gcc` |
| `cmake-armclang` | `keil` | `armclang` |

设置 `--target` 时会同时 `SetOption('no_exec', 1)`（[building.py#L184](file:///workspace/tools/building.py#L184)），即**只生成工程、不执行编译**。

如果 `rtconfig.EXEC_PATH` 下找不到编译器，`PrepareBuilding` 会尝试从 RT-Thread Env 的 `pkgs.json` 中探测 SDK 路径（[building.py#L197-L217](file:///workspace/tools/building.py#L197-L217)），失败则保持原设置，最终在 `EndBuilding` 中报错并退出。

### 11.4.3 解析 rtconfig.h

关键代码只有几行（[building.py#L286-L292](file:///workspace/tools/building.py#L286-L292)）：

```python
PreProcessor = PatchedPreProcessor()
f = open('rtconfig.h', 'r')
contents = f.read()
f.close()
PreProcessor.process_contents(contents)
BuildOptions = PreProcessor.cpp_namespace
```

这里复用了 SCons 的 **C 预处理器**，只处理 `#define` / `#undef`（通过 patch 关闭了 `#include` 展开），最终得到：

```python
{
    'RT_NAME_MAX': '8',
    'RT_USING_FINSH': '',
    'RT_TICK_PER_SECOND': '100',
    ...
}
```

这个字典就是后续 `GetDepend()` 的唯一依据，因此 **`rtconfig.h` 必须在执行 `scons` 之前已经存在**。

> 注意：`rtconfig.h` 里被注释掉的配置项（`/* RT_USING_SMP is not set */`）不会出现在 `BuildOptions` 中，`GetDepend` 会将其视为“未开启”。

### 11.4.4 事件钩子与内核组

`PrepareBuilding` 还会：

- 注册 `BuildLib` builder，供 `--buildlib` 打包单个组件；
- 若 `--strict` 且工具链为 GCC，追加 `-Werror`；
- 若链接参数含 `nano.specs`，自动追加 `_REENT_SMALL`；
- 若开启 `RT_USING_CPLUSPLUS`，把链接器从 `gcc` 换成 `g++`；
- 建立名为 `Kernel` 的组，并把 **BSP 根目录**加入 `CPPPATH`（[building.py#L279](file:///workspace/tools/building.py#L279)），保证所有源码都能 `#include "rtconfig.h"`。

---

## 11.5 GetDepend 与 DefineGroup

### 11.5.1 GetDepend 的判定语义

[GetDepend](file:///workspace/tools/building.py#L448-L464) 支持字符串与列表两种入参：

```python
def GetDepend(depend):
    building = True
    if type(depend) == type('str'):
        if not depend in BuildOptions or BuildOptions[depend] == 0:
            building = False
        elif BuildOptions[depend] != '':
            return BuildOptions[depend]
        return building

    for item in depend:
        if item != '':
            if not item in BuildOptions or BuildOptions[item] == 0:
                building = False
    return building
```

由此可总结出：

| `rtconfig.h` 内容 | `GetDepend('X')` |
| --- | --- |
| `#define X` | `True`（返回空字符串之外的布尔真） |
| `#define X 1` | 返回字符串 `'1'`（可当数值/字符串用） |
| `#define X 1000` | 返回 `'1000'`，可用于取配置值 |
| 未定义 / 被注释 | `False` |
| 列表 `['A','B']` | 仅当 A、B 都开启时为 `True` |

想取“配置的数值”时通常写成 `GetConfigValue('FINSH_THREAD_STACK_SIZE')`（[building.py#L441-L446](file:///workspace/tools/building.py#L441-L446)），例如 [finsh/SConscript](file:///workspace/components/finsh/SConscript#L14-L15) 用它生成 `-Wstack-usage=` 警告阈值。

### 11.5.2 DefineGroup 做了什么

[DefineGroup](file:///workspace/tools/building.py#L622-L739) 是每个 `SConscript` 的出口，典型用法：

```python
group = DefineGroup('Finsh', src, depend=['RT_USING_FINSH'],
                    CPPPATH=CPPPATH, LOCAL_CFLAGS=LOCAL_CFLAGS)
Return('group')
```

其行为可概括为：

1. `GetDepend(depend)` 为假 → 直接返回空列表，**该组完全不参与编译**；
2. 同名组已存在 → 走 `MergeGroup()`，把 `src`、`CPPPATH`、`CFLAGS` 等**追加合并**（[building.py#L529-L609](file:///workspace/tools/building.py#L529-L609)）；
3. 新组 → 按组名（忽略大小写）**字典序插入** `Projects`，这会影响 IDE 工程中的分组展示；
4. `CFLAGS`/`CPPPATH`/`CPPDEFINES`/`LIBS` 等**全局**参数会 `AppendUnique` 到 `Env`，对后续所有组生效；
5. `LOCAL_CFLAGS`/`LOCAL_CPPPATH`/`LOCAL_CPPDEFINES` 只记录在组内，由 `DoBuilding` 在编译该组源文件时临时拼接；
6. 带 `LIBRARY=...` 时改为 `Env.Library()` 预打包。

`LOCAL_*` 的存在非常关键：它让某个组件可以使用更严格的告警或私有宏，而不会污染整个工程。内核 `src/SConscript` 就是这样使用 `-Wmissing-prototypes`、`-Wstrict-prototypes` 等选项的（[src/SConscript#L32-L50](file:///workspace/src/SConscript#L32-L50)）。

---

## 11.6 DoBuilding 与最终链接

### 11.6.1 对象合并与局部选项

[DoBuilding](file:///workspace/tools/building.py#L781-L855) 先做三件事：

1. `one_list()` 把 `SConscript` 返回的嵌套列表拍平；
2. 若某组带 `LOCAL_*`，先把它的源文件从默认对象列表中移除；
3. 再用 `Env.Object(source, CFLAGS=..., CPPPATH=..., CPPDEFINES=...)` 以“组内附加选项”重新编译这些文件。

最后统一 `Env.Program(target, objects)`，由 SCons 依据 `LINKFLAGS`（其中含 `-T link.lds`）完成链接。

### 11.6.2 EndBuilding 的收尾工作

[EndBuilding](file:///workspace/tools/building.py#L938-L988) 负责：

| 动作 | 说明 |
| --- | --- |
| `Env.AddPostAction(target, rtconfig.POST_ACTION)` | 链接后执行 BSP 自定义命令（objcopy、size 等） |
| `Clean(...)` | 把 `cconfig.h`、`rtua.py`、`.sconsign.dblite` 纳入 `scons -c` 清理范围 |
| `GenTargetProject()` | 当指定 `--target` 时生成 Keil/IAR/VS/CMake… 工程并退出 |
| `MkDist()` | `--dist` / `--dist-ide` 打包发布 |
| `CscopeDatabase()` | `--cscope` 生成代码索引 |
| 工具链路径检查 | `EXEC_PATH` 不存在时打印错误并退出 |

`POST_ACTION` 来自 `rtconfig.py`。以 qemu-vexpress-a9 为例：

```python
POST_ACTION = OBJCPY + ' -O binary $TARGET rtthread.bin\n' + SIZE + ' $TARGET \n'
```

即链接出 `rtthread.elf` 后，再生成裸二进制 `rtthread.bin` 并打印段大小（[rtconfig.py#L74-L77](file:///workspace/bsp/qemu-vexpress-a9/rtconfig.py#L74-L77)）。

---

## 11.7 生成 IDE / 第三方工程

`scons --target=<name>` 会复用同一套 `Projects` 数据，生成不同格式的工程文件（[building.py#L857-L936](file:///workspace/tools/building.py#L857-L936)）：

| `--target` | 生成器 | 产物 |
| --- | --- | --- |
| `mdk5` | `tools/keil.py` | `project.uvprojx` |
| `mdk4` | `tools/keil.py` | `project.uvproj` |
| `iar` | `tools/iar.py` | `project.ewp` |
| `vs` / `vs2012` | `tools/vs.py` / `vs2012.py` | `.vcproj` / `.vcxproj` |
| `vsc` | `tools/vsc.py` | `.vscode` 配置（含调试/IntelliSense） |
| `cdk` | `tools/cdk.py` | `.cdkproj` |
| `ses` | `tools/ses.py` | SEGGER Embedded Studio 工程 |
| `makefile` | `tools/makefile.py` | `Makefile` |
| `eclipse` | `tools/eclipse.py` | Eclipse CDT 工程 |
| `cmake` / `cmake-armclang` | `tools/cmake.py` | `CMakeLists.txt` |
| `xmake` | `tools/xmake.py` | `xmake.lua` |
| `codelite` | `tools/codelite.py` | `.project` |
| `esp-idf` | `tools/esp_idf.py` | ESP-IDF 组件工程 |

常用组合：

```bash
scons --target=mdk5 -s
scons --target=cmake --project-name=my_proj
scons --target=vsc
```

`--project-name` 控制工程文件名，`--project-path` 控制输出目录（[options.py#L40-L49](file:///workspace/tools/options.py#L40-L49)）。生成工程时 SCons 处于 `no_exec` 模式，不会真正调用编译器。

---

## 11.8 运行实例：qemu-vexpress-a9

### 11.8.1 rtconfig.py 关键变量

| 变量 | 值（本 BSP） | 作用 |
| --- | --- | --- |
| `ARCH` | `arm` | 选择 `libcpu/arm` |
| `CPU` | `cortex-a` | 选择 `libcpu/arm/cortex-a` |
| `PLATFORM` | `gcc` | 编译器族，决定告警/库参数 |
| `PREFIX` | `arm-none-eabi-` | 工具链前缀，可由 `RTT_CC_PREFIX` 覆盖 |
| `EXEC_PATH` | `/usr/bin` | 工具链目录，可由 `RTT_EXEC_PATH` 覆盖 |
| `CFLAGS` | `-march=armv7-a -msoft-float -std=gnu99 ...` | 架构与语言选项 |
| `LFLAGS` | `-Wl,--gc-sections,-Map=rtthread.map -T link.lds ...` | 链接脚本与映射文件 |
| `LINK_SCRIPT` | `link.lds` | Smart 模式会替换为 `link_smart.lds` |
| `POST_ACTION` | `objcopy -O binary` + `size` | 生成 `.bin` 与尺寸报告 |

对应文件：[rtconfig.py](file:///workspace/bsp/qemu-vexpress-a9/rtconfig.py#L28-L77)。

### 11.8.2 编译

```bash
cd bsp/qemu-vexpress-a9
scons -j4
```

成功后会得到：

| 文件 | 说明 |
| --- | --- |
| `rtthread.elf` | 带调试信息的 ELF，QEMU 可直接加载 |
| `rtthread.bin` | 裸二进制，用于 `-kernel` 启动 |
| `rtthread.map` | 链接映射，含各段地址与符号 |
| `build/` | 所有 `.o` 与中间文件 |

### 11.8.3 启动 QEMU

[qemu.sh](file:///workspace/bsp/qemu-vexpress-a9/qemu.sh#L1-L6) 的内容：

```bash
if [ ! -f "sd.bin" ]; then
dd if=/dev/zero of=sd.bin bs=1024 count=65536
fi

qemu-system-arm --version
qemu-system-arm -M vexpress-a9 -smp cpus=2 -kernel rtthread.bin -serial stdio -sd sd.bin
```

逐项解释：

- `sd.bin`：64 MiB 的空磁盘镜像，供 DFS + elmfat 挂载；首次挂载失败时，BSP 的 `mnt.c` 会自动 `mkfs` 再挂载；
- `-M vexpress-a9`：机器型号；
- `-smp cpus=2`：向 QEMU 暴露 2 个 CPU。是否真正启用 SMP 仍由 `rtconfig.h` 的 `RT_USING_SMP` / `RT_CPUS_NR` 决定，本 BSP 当前配置为单核；
- `-kernel rtthread.bin`：直接加载裸二进制；
- `-serial stdio`：串口重定向到终端，因此能看到 FinSH 输出；
- `-sd sd.bin`：挂载 SD 卡镜像。

运行后会看到启动横幅并进入 `msh />` 提示符；在 `msh` 下按 `Ctrl+C` 并按提示输入 `y` 可退出 QEMU。

### 11.8.4 调试模式

`qemu-dbg.sh` 会以 `-s -S` 方式启动 QEMU，等待 GDB 连接（默认端口 1234）。配合 `arm-none-eabi-gdb rtthread.elf` 即可单步调试；`rtthread.elf` 中的符号与 `rtthread.map` 可用于定位地址。

---

## 11.9 常用命令与排错

### 11.9.1 常用命令

| 命令 | 作用 |
| --- | --- |
| `scons -j4` | 多线程编译 |
| `scons -c` | 清理对象与目标文件 |
| `scons --verbose` | 打印完整编译命令行（默认只显示 `CC xxx`） |
| `scons --strict` | GCC 下把告警当错误 |
| `scons --buildlib=Finsh` | 只把某个组编译成静态库 |
| `scons --cleanlib` | 清理 `--buildlib` 产生的库 |
| `scons --stackanalysis` | 线程栈静态分析（`tools/WCS.py`） |
| `scons --cscope` | 生成 cscope 索引 |
| `scons --menuconfig` | 修改配置并重新生成 `rtconfig.h` |

### 11.9.2 常见错误

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `FileNotFoundError: rtconfig.h` | 新 BSP 尚未生成配置 | 先执行 `scons --pyconfig-silent` 或 `scons --menuconfig` |
| `Error: the toolchain path (...) is not exist` | `EXEC_PATH` 错误 | 设置 `RTT_EXEC_PATH`，或修改 `rtconfig.py` |
| `arm-none-eabi-gcc: command not found` | 工具链未加入 `PATH` | 安装工具链并导出 `RTT_EXEC_PATH` |
| 修改 Kconfig 后宏没生效 | 只改了 `Kconfig`，未重新生成 `rtconfig.h` | 重新 `menuconfig`，或检查 `.config` |
| `undefined reference to rt_hw_...` | `libcpu`/BSP 未实现对应移植接口 | 参考第 10 章补齐接口 |
| 链接报段溢出 | `link.lds` 内存布局与实际芯片不符 | 修改 `MEMORY`/`SECTIONS` 或裁剪功能 |

> 注意：不要手工长期维护 `rtconfig.h`。它会在每次 `menuconfig` 时被整体重写；需要“配置系统之外”的私有宏，应放入 `rtconfig_project.h` 或 `rtconfig.py` 的编译选项。

---

## 11.10 小结

- `scons` 的本质是：**Kconfig 决定宏 → 宏决定 `SConscript` 收集哪些源文件 → SCons 依据 `rtconfig.py` 与 `link.lds` 编译链接**。
- `PrepareBuilding` 负责“环境 + 工具链 + 配置解析 + 递归收集”，`DoBuilding` 负责“对象编译 + 链接”，`EndBuilding` 负责“后处理 + 工程生成”。
- `GetDepend()` 是配置与构建之间的唯一开关；`DefineGroup()` 是每个目录对外声明编译内容的唯一入口。
- BSP 只需提供 `SConstruct`、`rtconfig.py`、`link.lds`、`SConscript` 与驱动，就能接入整套构建体系。
- 运行阶段依赖 BSP 的启动代码、链接脚本与 `POST_ACTION`；QEMU BSP 还额外依赖 `sd.bin` 提供块设备。

下一章将从“配置项”视角出发，系统梳理 Kconfig 的写法、组件之间的 `select`/`depends on` 关系，以及一张可直接查阅的依赖矩阵。
