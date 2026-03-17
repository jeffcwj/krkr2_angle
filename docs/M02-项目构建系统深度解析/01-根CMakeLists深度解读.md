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

## 第 3 段：Android 与桌面平台的 vcpkg 集成（第 17-24 行）

### 源码

```cmake
# 文件：krkr2/CMakeLists.txt 第 17-24 行
if(ANDROID)
    set(VCPKG_TARGET_ANDROID ON)
    set(ENV{ANDROID_NDK_HOME} "$ENV{ANDROID_NDK}")
    add_compile_options(-Wno-inconsistent-missing-override)
    include(cmake/vcpkg_android.cmake)
else()
    set(CMAKE_TOOLCHAIN_FILE $ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake)
endif()
```

### 逐行解析

这段代码处理一个关键问题：**如何在 Android 和桌面平台上分别集成 vcpkg**。

**第 17 行：`if(ANDROID)`**

`ANDROID` 变量由 Android NDK 的 toolchain 文件（`android.toolchain.cmake`）设置。当通过 Gradle 构建 Android 目标时，Gradle 会自动传入这个 toolchain 文件，因此 `ANDROID` 为 `TRUE`。在桌面平台上，这个变量未定义，`if(ANDROID)` 为假。

**第 18 行：`set(VCPKG_TARGET_ANDROID ON)`**

设置一个自定义标志变量，用于在后面包含的 `vcpkg_android.cmake` 中做条件判断。注意这**不是**vcpkg 官方变量——它是项目自定义的触发器。

**第 19 行：`set(ENV{ANDROID_NDK_HOME} "$ENV{ANDROID_NDK}")`**

这行做的是**环境变量重映射**。Android SDK 标准设置的环境变量名是 `ANDROID_NDK`，但 `vcpkg_android.cmake` 脚本期望的是 `ANDROID_NDK_HOME`。这行代码将前者的值复制到后者，确保两个工具链都能找到 NDK。

`ENV{}` 语法用于在 CMake 中读写**进程环境变量**（不是 CMake 缓存变量）：

```cmake
set(ENV{VAR_NAME} "value")   # 写入环境变量
message("$ENV{VAR_NAME}")    # 读取环境变量
```

**第 20 行：`add_compile_options(-Wno-inconsistent-missing-override)`**

Android 的 NDK Clang 编译器在编译 KrKr2 的某些代码时会报 `-Winconsistent-missing-override` 警告——这是因为继承的虚函数有些加了 `override` 有些没加。`-Wno-` 前缀关闭这个警告。

> **设计权衡：** 更好的做法是修复所有缺少 `override` 的地方，但由于 KrKr2 移植了大量原版 KiriKiri2 代码，全面修复工作量大且可能引入 bug，所以暂时选择抑制警告。

**第 21 行：`include(cmake/vcpkg_android.cmake)`**

包含自定义的 Android vcpkg 集成脚本。这个脚本在第 05 章详细分析。它的核心功能是**将 vcpkg toolchain 和 Android NDK toolchain 叠加**——通过 `VCPKG_CHAINLOAD_TOOLCHAIN_FILE` 机制实现。

**第 23 行：`set(CMAKE_TOOLCHAIN_FILE $ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake)`**

在桌面平台（Windows/Linux/macOS）上，直接设置 `CMAKE_TOOLCHAIN_FILE` 指向 vcpkg 的 toolchain 文件。这个文件会在 `project()` 命令执行时被加载，接管包查找机制。

### 为什么 Android 不能直接设置 CMAKE_TOOLCHAIN_FILE？

Android 构建已经有一个 toolchain 文件（`android.toolchain.cmake`），CMake 在一次配置中只能有一个 `CMAKE_TOOLCHAIN_FILE`。vcpkg 的解决方案是**链式加载**：

```
CMAKE_TOOLCHAIN_FILE = vcpkg.cmake
    └── VCPKG_CHAINLOAD_TOOLCHAIN_FILE = android.toolchain.cmake
```

vcpkg.cmake 在执行完自己的逻辑后，会自动 `include()` chainload 的 toolchain 文件。

### 常见错误

**问题 1：`VCPKG_ROOT` 环境变量未设置**

```
CMake Error at CMakeLists.txt:23:
  set CMAKE_TOOLCHAIN_FILE to "/scripts/buildsystems/vcpkg.cmake"
```

路径中缺少 VCPKG_ROOT 前缀，说明环境变量为空。解决方案：

```bash
# Linux/macOS
export VCPKG_ROOT=/path/to/vcpkg

# Windows (PowerShell)
$env:VCPKG_ROOT = "C:\src\vcpkg"

# Windows (CMD)
set VCPKG_ROOT=C:\src\vcpkg
```

**问题 2：Android 构建找不到包**

如果 vcpkg 安装了桌面版本的包但没有 Android 版本，`find_package()` 会失败。需要用正确的 triplet 安装：

```bash
vcpkg install --triplet arm64-android
```

---

## 第 4 段：项目声明与全局设置（第 26-38 行）

### 源码

```cmake
# 文件：krkr2/CMakeLists.txt 第 26-38 行
set(APP_NAME krkr2)

project(${APP_NAME})

option(ENABLE_TESTS "enable tests execute build(exclude android)" ON)
option(BUILD_TOOLS "build tools execute build(exclude android ios)" ON)

set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

set(KRKR2CORE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cpp/core)
set(KRKR2PLUGIN_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cpp/plugins)
```

### 逐行解析

**第 26 行：`set(APP_NAME krkr2)`**

将项目名存入变量，避免后续硬编码字符串。在 CMake 中，使用变量引用项目名是最佳实践——如果未来需要改名（比如从 `krkr2` 改为 `krkr2-emu`），只需修改这一处。

**第 28 行：`project(${APP_NAME})`**

`project()` 命令是 CMake 构建的**核心声明**。它做的事情远比看起来多：

1. 设置 `PROJECT_NAME` 变量为 `krkr2`
2. 设置 `CMAKE_PROJECT_NAME` 变量（仅在根 CMakeLists.txt 中）
3. **触发 toolchain 文件加载**——`CMAKE_TOOLCHAIN_FILE` 在此时被 include
4. 检测编译器（C/CXX），设置 `CMAKE_C_COMPILER`、`CMAKE_CXX_COMPILER`
5. 设置项目级变量：`PROJECT_SOURCE_DIR`、`PROJECT_BINARY_DIR`

> **重要：** 在 `project()` 之前设置 `CMAKE_TOOLCHAIN_FILE` 是因为 toolchain 必须在编译器检测**之前**加载。如果顺序反了，vcpkg 的 toolchain 就不会生效。

**第 30-31 行：选项定义**

```cmake
option(ENABLE_TESTS "enable tests execute build(exclude android)" ON)
option(BUILD_TOOLS "build tools execute build(exclude android ios)" ON)
```

`option()` 定义布尔缓存变量，可在配置时通过命令行覆盖：

```bash
cmake -B build -DENABLE_TESTS=OFF -DBUILD_TOOLS=OFF
```

两个选项默认都是 `ON`，但在 Android/iOS 构建中会被后面的条件判断跳过（第 136-143 行）。

**第 33-35 行：语言标准和 PIC**

```cmake
set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
```

- `CMAKE_CXX_STANDARD 17`：要求所有目标使用 C++17 标准。KrKr2 使用了 `std::optional`、`std::filesystem`、结构化绑定等 C++17 特性
- `CMAKE_C_STANDARD 17`：C 语言也设为 C17（C11 的 bug 修复版本）
- `CMAKE_POSITION_INDEPENDENT_CODE ON`：生成位置无关代码（`-fPIC`）。这在 Linux 上构建共享库时是**必须的**，在 Android 上构建 `.so` 也需要

**第 37-38 行：路径变量**

```cmake
set(KRKR2CORE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cpp/core)
set(KRKR2PLUGIN_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cpp/plugins)
```

定义两个路径变量供后续使用。`${CMAKE_CURRENT_SOURCE_DIR}` 是当前 CMakeLists.txt 所在目录的绝对路径。这些变量在子目录的 CMakeLists.txt 中也可访问（因为 CMake 变量默认向子作用域传播）。

---

## 第 4.5 段：MSVC 编译选项（第 40-44 行）

### 源码

```cmake
# 文件：krkr2/CMakeLists.txt 第 40-44 行
if(MSVC)
    add_compile_options("$<$<COMPILE_LANGUAGE:CXX>:/EHsc>" "/MP" "/utf-8")
    add_link_options("/ignore:4099" "/INCREMENTAL" "/DEBUG:FASTLINK")
    # set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()
```

### 逐行解析

**`if(MSVC)`** — 仅在使用 MSVC 编译器时生效。MinGW、Clang-CL 等不会进入此分支。

**编译选项解析：**

| 选项 | 含义 | 为什么需要 |
|------|------|-----------|
| `$<$<COMPILE_LANGUAGE:CXX>:/EHsc>` | 仅对 C++ 文件启用标准 C++ 异常处理 | KrKr2 的 TJS2 引擎使用 try/catch，没有这个选项 MSVC 不会生成异常处理代码 |
| `/MP` | 多进程并行编译（每个 .cpp 一个进程） | 加速编译，类似于 `make -j` 但在 MSVC 的 cl.exe 层面实现 |
| `/utf-8` | 将源文件和执行字符集都设为 UTF-8 | KrKr2 源码包含中文注释和 Unicode 字符串字面量 |

> **注意：** `$<$<COMPILE_LANGUAGE:CXX>:/EHsc>` 使用了生成器表达式确保 `/EHsc` 只传给 C++ 编译，不传给 C 编译。C 语言没有异常机制，传 `/EHsc` 会产生警告。

**链接选项解析：**

| 选项 | 含义 | 为什么需要 |
|------|------|-----------|
| `/ignore:4099` | 忽略 LNK4099 警告（找不到 PDB 文件） | 第三方库（vcpkg 安装的）经常没有附带 PDB，这个警告无害 |
| `/INCREMENTAL` | 启用增量链接 | 只重新链接改动的 .obj，大幅缩短链接时间 |
| `/DEBUG:FASTLINK` | 使用快速链接的调试信息格式 | 调试信息留在各 .obj 中不合并，链接更快（需 VS 调试器支持） |

**被注释掉的行：`CMAKE_MSVC_RUNTIME_LIBRARY`**

```cmake
# set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
```

这行如果启用，会将 CRT（C 运行时库）从默认的 DLL 版本（`/MD`）切换为静态版本（`/MT`）。当前被注释说明项目使用的是 DLL 版 CRT——这与 `CMakePresets.json` 中的 triplet `x64-windows-static-md` 一致（static 库 + MD 运行时）。

---

## 第 5 段：四平台入口文件注册（第 46-84 行）

### 源码

```cmake
# 文件：krkr2/CMakeLists.txt 第 46-84 行
if(ANDROID)
    add_library(${PROJECT_NAME} SHARED
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/android/cpp/krkr2_android.cpp)
    find_package(unofficial-breakpad CONFIG REQUIRED)
    target_link_libraries(${PROJECT_NAME} PUBLIC
        unofficial::breakpad::libbreakpad_client)
elseif(LINUX)
    add_executable(${PROJECT_NAME}
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/linux/main.cpp)
elseif(WINDOWS)
    add_executable(${PROJECT_NAME}
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/windows/main.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/windows/game.rc)
elseif(IOS)
    # ... (完整代码已注释掉，iOS 平台暂未维护)
elseif(MACOS)
    set(APP_UI_RES
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/Icon.icns
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/Info.plist)
    add_executable(${PROJECT_NAME}
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/main.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/Prefix.pch
        ${APP_UI_RES})
endif()
```

### 关键设计决策分析

**1. Android 是共享库（SHARED），其他平台是可执行文件（EXECUTABLE）**

这是最重要的架构差异。Android 应用由 Java/Kotlin 层驱动，C++ 代码编译为 `.so` 共享库，在运行时通过 JNI 加载。而桌面平台直接生成可执行文件。

```
Android:    Java Activity → System.loadLibrary("krkr2") → .so
桌面平台:    操作系统 → 直接运行 krkr2.exe / krkr2
```

**2. Android 额外链接 Breakpad**

```cmake
find_package(unofficial-breakpad CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PUBLIC unofficial::breakpad::libbreakpad_client)
```

[Breakpad](https://chromium.googlesource.com/breakpad/breakpad/) 是 Google 开发的崩溃报告框架。Android 上 C++ 崩溃不会产生 Java 异常，默认只有一个 SIGSEGV 信号然后进程静默退出。Breakpad 能捕获 native crash 并生成 minidump 文件，用于事后调试。

桌面平台为什么不需要？因为：
- Windows 有 WER（Windows Error Reporting）和直接的调试器附加
- Linux 有 core dump（`ulimit -c unlimited`）
- macOS 有 CrashReporter

**3. Windows 包含 .rc 资源文件**

```cmake
${CMAKE_CURRENT_SOURCE_DIR}/platforms/windows/game.rc
```

`.rc`（Resource Script）文件定义了 Windows 应用的图标、版本信息、字符串表等。没有这个文件，生成的 .exe 在资源管理器中会显示默认图标。

**4. macOS 包含 App Bundle 资源**

```cmake
set(APP_UI_RES
    ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/Icon.icns
    ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/Info.plist)
```

macOS 应用以 `.app` Bundle 形式分发。`Icon.icns` 是应用图标（macOS 专用格式），`Info.plist` 定义 Bundle 元数据（应用名、版本号、最低系统版本等）。

**5. iOS 代码被完整注释掉**

```cmake
elseif(IOS)
#        list(APPEND GAME_HEADER ...)
#        set(APP_UI_RES ...)
#        list(APPEND GAME_SOURCE ...)
```

这表明 iOS 移植曾经开始过但未完成。注释保留了代码结构作为未来参考。

### 自定义平台变量 vs CMAKE_SYSTEM_NAME

注意这里用的是 `WINDOWS`、`LINUX`、`MACOS` 等**自定义变量**，而不是 CMake 内置的 `CMAKE_SYSTEM_NAME`。这些变量在 `CMakePresets.json` 中通过 `cacheVariables` 设置：

```json
{
  "name": "Windows Config",
  "cacheVariables": { "WINDOWS": true }
}
```

```json
{
  "name": "Linux Config",
  "cacheVariables": { "LINUX": true }
}
```

为什么不用标准的 `CMAKE_SYSTEM_NAME`？因为 `CMAKE_SYSTEM_NAME` 的值在不同环境下不完全一致（如 `Darwin` vs `macOS`），而自定义布尔变量更简洁直观。

---

## 第 6 段：子目录与链接（第 86-97 行）

### 源码

```cmake
# 文件：krkr2/CMakeLists.txt 第 86-97 行
target_include_directories(${PROJECT_NAME} PUBLIC ${KRKR2CORE_PATH}/environ/cocos2d)

# external lib
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/cpp/external)

# build library
add_subdirectory(${KRKR2CORE_PATH})
add_subdirectory(${KRKR2PLUGIN_PATH})

target_link_libraries(${PROJECT_NAME} PUBLIC
    krkr2plugin krkr2core
)
```

### 解析

**第 86 行：公开头文件目录**

`${KRKR2CORE_PATH}/environ/cocos2d` 目录包含 `AppDelegate.h` 等 Cocos2d-x 桥接头文件。使用 `PUBLIC` 意味着链接 `krkr2` 的目标也能访问这些头文件（虽然作为最终可执行文件，这里 PUBLIC 和 PRIVATE 效果相同）。

**第 89-93 行：三个 add_subdirectory 调用**

```
cpp/external/  → 第三方库（libbpg、minizip）
cpp/core/      → 核心引擎（INTERFACE 库 krkr2core）
cpp/plugins/   → 插件集合（STATIC 库 krkr2plugin）
```

`add_subdirectory()` 让 CMake 进入子目录解析其 `CMakeLists.txt`，子目录中定义的目标自动注册到全局。顺序很重要——`external` 在前，因为 `core` 和 `plugins` 可能依赖它。

**第 95-97 行：链接核心库**

```cmake
target_link_libraries(${PROJECT_NAME} PUBLIC
    krkr2plugin krkr2core
)
```

最终可执行文件（或 Android 上的 .so）链接两个库。由于 `krkr2core` 是 INTERFACE 库，链接它实际上是继承它传播的所有编译定义、包含目录和下游库依赖（如 cocos2dx、OpenMP、FFmpeg 等）。详见第 02 章。

---

## 第 7 段：资源拷贝（第 99-134 行）

### 源码概要

```cmake
# 文件：krkr2/CMakeLists.txt 第 99-134 行
if(NOT ANDROID)
    include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/CocosBuildHelpers.cmake")
    set(GAME_RES_FOLDER "${CMAKE_CURRENT_SOURCE_DIR}/ui/cocos-studio")

    if(WINDOWS OR APPLE)
        cocos_mark_multi_resources(common_res_files RES_TO "Resources"
            FOLDERS ${GAME_RES_FOLDER})
    endif()

    setup_cocos_app_config(${APP_NAME})

    if(APPLE)
        set_target_properties(${APP_NAME} PROPERTIES RESOURCE "${APP_UI_RES}")
        target_sources(${APP_NAME} PRIVATE ${common_res_files})
        # macOS bundle 和 iOS 设置...
    elseif(WINDOWS)
        target_sources(${PROJECT_NAME} PRIVATE ${common_res_files})
        cocos_copy_target_dll(${APP_NAME})
    endif()

    if(LINUX OR WINDOWS)
        cocos_get_resource_path(APP_RES_DIR ${APP_NAME})
        cocos_copy_target_res(${APP_NAME} LINK_TO ${APP_RES_DIR}
            FOLDERS ${GAME_RES_FOLDER})
    endif()
endif()
```

### 解析

**为什么 Android 不需要资源拷贝？**

Android 的资源文件通过 Gradle 的 `assets.srcDirs` 直接打包进 APK：

```groovy
// platforms/android/app/build.gradle 第 81 行
assets.srcDirs = ['../../../ui/cocos-studio']
```

所以 CMake 层面不需要额外处理。

**资源处理流程图：**

```
┌──────────────────────────────────────────────────┐
│ ui/cocos-studio/  (Cocos Studio 导出的 UI 资源)   │
└──────────┬───────────────────────────────────────┘
           │
    ┌──────┴──────┐
    │  平台判断    │
    └──┬──────┬───┘
       │      │
  ┌────▼──┐ ┌─▼────────┐
  │Windows│ │  macOS    │
  │ Linux │ │           │
  └───┬───┘ └────┬──────┘
      │          │
      ▼          ▼
  拷贝到       嵌入到
  bin/krkr2/   .app Bundle
  Resources/   Resources/
```

**关键函数说明（来自 CocosBuildHelpers.cmake）：**

| 函数 | 作用 |
|------|------|
| `cocos_mark_multi_resources()` | 扫描文件夹，为每个文件设置 `MACOSX_PACKAGE_LOCATION` 属性 |
| `setup_cocos_app_config()` | 配置输出目录、macOS bundle 属性、IDE 代码分组 |
| `cocos_copy_target_dll()` | 递归查找所有依赖 DLL，拷贝到可执行文件旁边（仅 Windows） |
| `cocos_copy_target_res()` | 使用 Python 脚本同步资源文件夹到构建目录 |

> **常见问题：** 如果运行时报"Resource not found"，首先检查 `bin/krkr2/Resources/` 目录是否存在。在 Windows 上，`cocos_copy_target_res` 依赖 Python（`find_package(Python REQUIRED)`），确保系统安装了 Python 并在 PATH 中。

---

## 第 8 段：测试与工具（第 136-143 行）

### 源码

```cmake
# 文件：krkr2/CMakeLists.txt 第 136-143 行
if(ENABLE_TESTS AND NOT (IOS OR ANDROID))
    enable_testing()
    add_subdirectory(tests)
endif()

if(BUILD_TOOLS AND NOT (IOS OR ANDROID))
    add_subdirectory(tools)
endif()
```

### 解析

这两段使用了前面定义的 `option()` 变量（第 30-31 行），并额外排除了移动平台。

**`enable_testing()`** — 启用 CTest 测试框架。执行后才能在 `tests/` 子目录中使用 `add_test()` 注册测试用例。

**为什么排除 Android 和 iOS？**

1. **测试框架不适用**：Catch2 单元测试需要在命令行运行，移动平台没有标准的命令行入口
2. **交叉编译限制**：Android 构建产生的是 ARM/x86_64 二进制，在 x86_64 构建机上无法直接运行
3. **工具无意义**：`tools/xp3/`（XP3 档案提取器）是开发辅助工具，只在桌面使用

```bash
# 禁用测试和工具的构建（加速编译）
cmake -B build -DENABLE_TESTS=OFF -DBUILD_TOOLS=OFF

# 运行测试
cmake --build build
ctest --test-dir build --output-on-failure
```

---

## 动手实践

### 实验 1：观察 ccache 效果

```bash
# 1. 确保安装了 ccache
ccache --version

# 2. 清空 ccache 统计
ccache -z

# 3. 完整构建项目
cmake --preset="Windows Debug Config"    # 或对应平台
cmake --build out/windows/debug

# 4. 查看缓存命中情况（第一次构建应该全部 miss）
ccache -s

# 5. 清理构建目录后重新构建
cmake --build out/windows/debug --clean-first
cmake --build out/windows/debug

# 6. 再次查看（这次应该看到大量 hit）
ccache -s
```

### 实验 2：修改编译定义观察效果

在 `krkr2/CMakeLists.txt` 第 11-15 行的 `add_compile_definitions` 中临时添加一个自定义宏：

```cmake
add_compile_definitions(
        $<$<CONFIG:Debug>:DEBUG>
        $<$<CONFIG:Debug>:_DEBUG>
        $<$<NOT:$<CONFIG:Debug>>:NDEBUG>
        MY_CUSTOM_MACRO=42    # 临时添加，用于验证
)
```

然后在任意 .cpp 文件中添加：

```cpp
#include <cstdio>
// 在某个函数中
printf("MY_CUSTOM_MACRO = %d\n", MY_CUSTOM_MACRO);  // 应输出 42
```

编译运行后确认宏值被正确传递，然后**删除临时添加的代码**。

---

## 对照项目源码

本节分析的是以下文件：

| 文件路径 | 行数 | 本节覆盖的行范围 |
|----------|------|-----------------|
| `krkr2/CMakeLists.txt` | 143 | 全部（第 1-143 行） |

关联文件（在后续章节详细分析）：

| 文件路径 | 关联段落 | 详细分析章节 |
|----------|----------|-------------|
| `krkr2/cpp/core/CMakeLists.txt` | 第 6 段 add_subdirectory | 第 02 章 |
| `krkr2/cpp/plugins/CMakeLists.txt` | 第 6 段 add_subdirectory | 第 02 章 |
| `krkr2/cmake/vcpkg_android.cmake` | 第 3 段 include | 第 05 章 |
| `krkr2/cmake/CocosBuildHelpers.cmake` | 第 7 段 include | 第 05 章 |
| `krkr2/CMakePresets.json` | 第 5 段自定义平台变量 | 第 03 章 |

---

## 本节小结

- 根 `CMakeLists.txt` 共 143 行，分为 8 个逻辑段，每段职责清晰
- **ccache** 通过 `CMAKE_CXX_COMPILER_LAUNCHER` 集成，是可选的编译加速工具
- **生成器表达式** `$<CONFIG:Debug>` 实现了 Debug/Release 编译定义的自动切换
- **vcpkg 集成** 在 Android 和桌面平台使用不同策略（链式 toolchain vs 直接设置）
- **平台入口** 通过 `if/elseif` 分支为四平台注册不同的源文件和目标类型
- **MSVC 选项** 处理了异常模型（`/EHsc`）、并行编译（`/MP`）和字符编码（`/utf-8`）
- **资源拷贝** 区分了 macOS Bundle 嵌入、Windows/Linux 文件拷贝、Android Gradle 打包三种方式
- **测试和工具** 通过 `option()` + 平台排除实现条件构建

---

## 练习题与答案

### 题目 1：分析 ccache 条件判断

如果系统中没有安装 ccache，`find_program(CCACHE_PROGRAM ccache)` 之后 `CCACHE_PROGRAM` 的值是什么？`if (CCACHE_PROGRAM)` 会走哪个分支？这会导致构建失败吗？

<details>
<summary>查看答案</summary>

`CCACHE_PROGRAM` 的值会被设置为 `CCACHE_PROGRAM-NOTFOUND`（这是 CMake 的 `find_program` 在找不到程序时的默认行为）。

`if (CCACHE_PROGRAM)` 会走 **false 分支**（即跳过 if 块），因为以 `-NOTFOUND` 结尾的字符串在 CMake 布尔上下文中被视为假值。

这**不会**导致构建失败。ccache 集成是完全可选的——没有 ccache 时，CMake 直接调用编译器，只是不会享受到编译缓存带来的加速。这是一个优秀的设计模式：可选优化不应该成为构建的硬性依赖。

</details>

### 题目 2：理解生成器表达式嵌套

解释以下生成器表达式在 Debug 和 Release 两种构建类型下分别展开为什么值：

```cmake
$<$<NOT:$<CONFIG:Debug>>:NDEBUG>
```

<details>
<summary>查看答案</summary>

**Debug 模式下的求值过程：**

1. `$<CONFIG:Debug>` → 当前是 Debug 配置，求值为 `1`
2. `$<NOT:1>` → 对 `1` 取反，求值为 `0`
3. `$<0:NDEBUG>` → 条件为 `0`，展开为**空字符串** `""`

结果：Debug 模式下不定义 `NDEBUG`。

**Release 模式下的求值过程：**

1. `$<CONFIG:Debug>` → 当前不是 Debug 配置，求值为 `0`
2. `$<NOT:0>` → 对 `0` 取反，求值为 `1`
3. `$<1:NDEBUG>` → 条件为 `1`，展开为 `NDEBUG`

结果：Release 模式下定义 `NDEBUG`，相当于编译器参数 `-DNDEBUG`。

这确保了 `assert()` 宏只在 Debug 模式下生效，Release 模式下完全消除（零运行时开销）。

</details>

### 题目 3：平台入口文件设计分析

为什么 Android 平台使用 `add_library(... SHARED ...)` 而桌面平台使用 `add_executable(...)`？如果把 Android 也改成 `add_executable`，会发生什么？

<details>
<summary>查看答案</summary>

**原因：** Android 应用的架构决定了这个设计。

Android 应用的入口是 Java/Kotlin 的 `Activity`，由 Android 运行时（ART）启动。C++ 代码必须编译为**共享库**（`.so` 文件），在运行时通过 `System.loadLibrary("krkr2")` 动态加载。Java 层通过 JNI（Java Native Interface）调用 C++ 函数。

桌面平台（Windows/Linux/macOS）的程序直接由操作系统加载执行，入口点是 `main()` 或 `WinMain()`，因此需要 `add_executable()`。

**如果把 Android 改成 `add_executable()`：**

1. CMake 会生成一个 ELF 可执行文件而不是 `.so`
2. Gradle 的 `externalNativeBuild` 期望找到 `.so` 文件，找不到会导致**打包失败**
3. 即使手动放入 APK，Android 也无法通过 `System.loadLibrary()` 加载一个可执行文件
4. 可执行文件有 `main()` 入口，但 Android 上没有直接运行 native 可执行文件的标准方式（需要 root 权限或通过 adb shell）

简言之：Android 的应用模型要求 native 代码以共享库形式存在，这是平台架构的硬性约束。

</details>

---

## 下一步

下一章 [02-INTERFACE库与模块链接](./02-INTERFACE库与模块链接.md) 将深入分析 `cpp/core/CMakeLists.txt` 和 `cpp/plugins/CMakeLists.txt`，理解 INTERFACE 库 `krkr2core` 如何将 9 个子模块聚合为一个统一的链接目标。
