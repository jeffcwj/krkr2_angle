# 本节目标
# 根 CMakeLists.txt 深度解读

> **所属模块：** M02-项目构建系统深度解析
> **前置知识：** [P01-现代CMake与构建工具链](../P01-现代CMake与构建工具链/README.md) 全部章节、[M01-项目导览与环境搭建](../M01-项目导览与环境搭建/README.md) Ch2 项目结构
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：

1. 逐行解释根 `CMakeLists.txt`（143 行）中每一段代码的作用和设计意图
2. 理解 ccache 集成如何加速增量编译
3. 掌握生成器表达式在编译定义中的实际应用
4. 理解 vcpkg toolchain 在 Android 与桌面平台的不同集成方式
5. 掌握四平台入口文件注册的条件分支逻辑
6. 理解资源拷贝、测试和工具的条件构建机制

## 文件概览

KrKr2 的根 `CMakeLists.txt` 位于 `krkr2/CMakeLists.txt`，共 143 行。这个文件是整个项目构建系统的**入口点**——CMake 执行时首先解析这个文件，然后递归进入各子目录。

我们按功能将其分为 **8 个逻辑段**：

```
┌─────────────────────────────────────────────────┐
│ 第 1 段（1-9 行）   ：CMake 版本要求 + ccache     │
│ 第 2 段（11-15 行）  ：全局编译定义（Debug/Release）│
│ 第 3 段（17-24 行）  ：Android/桌面 vcpkg 分支      │
│ 第 4 段（26-38 行）  ：项目声明 + 选项 + 标准        │
│ 第 5 段（40-84 行）  ：平台入口文件注册              │
│ 第 6 段（86-97 行）  ：子目录与链接                  │
│ 第 7 段（99-134 行） ：资源拷贝                     │
│ 第 8 段（136-143 行）：测试与工具                   │
└─────────────────────────────────────────────────┘
```

下面逐段分析。

---

## 第 1 段：CMake 版本要求与 ccache 集成（第 1-9 行）

### 源码

```cmake
# 文件：krkr2/CMakeLists.txt 第 1-9 行
cmake_minimum_required(VERSION 3.28)

find_program(CCACHE_PROGRAM ccache)

if (CCACHE_PROGRAM)
    set(CMAKE_C_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
    set(CMAKE_CXX_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
    message(STATUS "Ccache found: ${CCACHE_PROGRAM}")
endif ()
```

### 逐行解析

**第 1 行：`cmake_minimum_required(VERSION 3.28)`**

这是每个 CMakeLists.txt **必须**有的第一条命令（在 `project()` 之前）。它做两件事：

1. **版本检查**：如果用户安装的 CMake 低于 3.28，配置阶段立即报错退出
2. **策略设置**：自动启用 CMake 3.28 及之前所有版本引入的"新行为策略"（CMP 策略）

为什么是 3.28？因为项目使用了以下 3.28 特性：

| 特性 | 引入版本 | 在本项目中的使用 |
|------|----------|-----------------|
| CMakePresets.json v6 | 3.25 | `CMakePresets.json` 使用 version 6 |
| `CMAKE_EXPORT_COMPILE_COMMANDS` | 3.5 | 预设中启用 compile_commands.json |
| `cmake_path()` 命令 | 3.20 | 路径操作 |
| Ninja Multi-Config 改进 | 3.17+ | Ninja 生成器 |

> **常见错误：** 在 Ubuntu 22.04 上通过 `apt install cmake` 安装的是 3.22 版本，无法满足要求。解决方案见 [M01-Ch3/02-Linux环境搭建](../M01-项目导览与环境搭建/Ch3/02-Linux环境搭建.md)。

**第 3 行：`find_program(CCACHE_PROGRAM ccache)`**

`find_program()` 在系统 PATH 中搜索名为 `ccache` 的可执行文件。如果找到，变量 `CCACHE_PROGRAM` 被设为其完整路径（如 `/usr/bin/ccache`）；如果没找到，值为 `CCACHE_PROGRAM-NOTFOUND`。

**第 5-8 行：ccache 配置**

```cmake
if (CCACHE_PROGRAM)
    set(CMAKE_C_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
    set(CMAKE_CXX_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
    message(STATUS "Ccache found: ${CCACHE_PROGRAM}")
endif ()
```

`CMAKE_C_COMPILER_LAUNCHER` 和 `CMAKE_CXX_COMPILER_LAUNCHER` 是 CMake 内置变量。设置后，CMake 在调用编译器时会在前面加上 launcher 程序：

```
# 不用 ccache 时：
g++ -c main.cpp -o main.o

# 使用 ccache 时：
ccache g++ -c main.cpp -o main.o
```

ccache 的工作原理是**缓存编译结果**。当源文件和编译参数未变化时，ccache 直接从缓存返回 `.o` 文件，跳过实际编译。在大型 C++ 项目中（KrKr2 有数百个 .cpp 文件），这可以将增量编译时间从分钟级降低到秒级。

### 跨平台安装 ccache

| 平台 | 安装命令 | 验证 |
|------|----------|------|
| Windows | `choco install ccache` 或从 [GitHub Releases](https://github.com/ccache/ccache/releases) 下载 | `ccache --version` |
| Linux (Ubuntu/Debian) | `sudo apt install ccache` | `ccache --version` |
| Linux (Fedora) | `sudo dnf install ccache` | `ccache --version` |
| macOS | `brew install ccache` | `ccache --version` |

> **注意：** Android 构建使用 Gradle 调用 CMake，ccache 同样生效——只要系统 PATH 中能找到 ccache。

### 常见错误与解决方案

**问题 1：ccache 未安装但构建正常**

这不是错误。`if (CCACHE_PROGRAM)` 判断确保了 ccache 是**可选的**。没有 ccache 时，CMake 直接调用编译器，只是增量编译会慢一些。

**问题 2：ccache 缓存命中率低**

```bash
# 查看 ccache 统计信息
ccache -s

# 如果命中率低于 50%，检查：
# 1. 是否频繁切换构建类型（Debug/Release）
# 2. 是否频繁清理构建目录
# 3. ccache 缓存大小是否足够
ccache -M 10G  # 设置缓存上限为 10GB
```

---

## 第 2 段：全局编译定义（第 11-15 行）

### 源码

```cmake
# 文件：krkr2/CMakeLists.txt 第 11-15 行
add_compile_definitions(
        $<$<CONFIG:Debug>:DEBUG>
        $<$<CONFIG:Debug>:_DEBUG>   # 对于 MSVC 兼容性
        $<$<NOT:$<CONFIG:Debug>>:NDEBUG>
)
```

### 逐行解析

`add_compile_definitions()` 为**所有目标**（当前目录及子目录中后续添加的目标）添加预处理器宏定义。这里使用了**生成器表达式**（Generator Expressions），在 P01 第 2 章第 3 节已详细讲解。

让我们逐行拆解这三个定义：

**`$<$<CONFIG:Debug>:DEBUG>`** — 当构建类型为 Debug 时，定义宏 `DEBUG`

这是一个嵌套生成器表达式。从内向外解读：

1. `$<CONFIG:Debug>` — 如果当前构建配置是 Debug，求值为 `1`；否则为 `0`
2. `$<$<CONFIG:Debug>:DEBUG>` — 外层是 `$<condition:value>` 形式，当条件为 `1` 时展开为 `DEBUG`，否则展开为空字符串

效果等价于在 Debug 模式下给编译器传 `-DDEBUG`（GCC/Clang）或 `/DDEBUG`（MSVC）。

**`$<$<CONFIG:Debug>:_DEBUG>`** — 当构建类型为 Debug 时，定义宏 `_DEBUG`

`_DEBUG` 是 MSVC 生态系统的惯例。Microsoft 的 C 运行时库（CRT）在 `_DEBUG` 定义时会启用额外的调试检查（如堆内存验证、迭代器调试）。虽然这里注释写的是"对于 MSVC 兼容性"，但实际上这个宏对所有平台都有效——KrKr2 的某些代码也使用 `#ifdef _DEBUG` 做条件编译。

**`$<$<NOT:$<CONFIG:Debug>>:NDEBUG>`** — 当构建类型**不是** Debug 时，定义宏 `NDEBUG`

`NDEBUG` 是 C/C++ 标准定义的宏，用于禁用 `assert()` 宏。当定义了 `NDEBUG` 时：

```cpp
#include <cassert>
assert(ptr != nullptr);  // 在 Release 模式下，这行代码完全消失（零开销）
```

### 为什么使用 add_compile_definitions 而不是 target_compile_definitions？

`add_compile_definitions()` 是**目录级**命令，影响当前 CMakeLists.txt 及所有子目录中的目标。而 `target_compile_definitions()` 是**目标级**命令，只影响指定的目标。

在根 CMakeLists.txt 中使用目录级命令是合理的——这些 Debug/Release 宏应该在**所有**编译单元中保持一致。如果 krkr2core 在 Debug 模式但 krkr2plugin 在 Release 模式，运行时会出现 ABI 不兼容导致的神秘崩溃。

### 常见错误

**问题：assert 在 Release 模式下失效**

有开发者习惯用 `assert()` 做参数校验，然后发现 Release 版本跳过了校验直接崩溃。原因就是 `NDEBUG` 禁用了 `assert()`。正确做法是对用户输入使用 `if` 校验，只对"绝对不应该发生"的内部状态使用 `assert()`。

---

