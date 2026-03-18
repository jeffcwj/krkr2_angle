# CMake 传递平台信息与 KrKr2 实例

> **所属模块：** P03-跨平台 C++ 开发
> **前置知识：** [01-预处理器基础与平台宏](./01-预处理器基础与平台宏.md)、[P01-现代 CMake 与构建工具链](../../P01-现代CMake与构建工具链/)
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：
1. 使用 `target_compile_definitions()` 从 CMake 向 C++ 代码传递自定义宏
2. 理解 `PRIVATE`、`PUBLIC`、`INTERFACE` 三种可见性的区别，并在实际项目中正确选择
3. 使用 Generator Expressions 根据平台、构建类型等条件动态设置编译定义
4. 使用 `configure_file()` 和 `#cmakedefine` 生成包含构建时信息的配置头文件
5. 分析 KrKr2 项目中 CMake 传递平台信息的真实用法

---

## 为什么不直接用编译器预定义宏

上一节我们学到，每个编译器都会自动定义平台相关的宏（`_WIN32`、`__linux__` 等）。既然如此，为什么还需要通过 CMake 传递平台信息？

考虑这个场景：

```cpp
// ❌ 直接使用编译器预定义宏
#if defined(__linux__) && !defined(__ANDROID__)
    #include <X11/Xlib.h>
    void createWindow() {
        // 使用 X11 创建窗口...
    }
#endif
```

问题出现了：你的项目在 Linux 上既支持 X11 也支持 Wayland，或者用 SDL2 来抽象窗口系统。编译器预定义宏只告诉你"这是 Linux"，但**无法告诉你项目选择了哪个窗口后端**。

这就是 CMake 传递自定义宏的价值：

```cmake
# CMakeLists.txt — 项目级的配置选择
option(USE_X11 "Use X11 window system" ON)
option(USE_SDL2 "Use SDL2 window system" OFF)

if(USE_X11)
    target_compile_definitions(my_target PRIVATE USE_X11=1)
elseif(USE_SDL2)
    target_compile_definitions(my_target PRIVATE USE_SDL2=1)
endif()
```

```cpp
// ✅ 使用项目级的宏——构建系统告诉你"用哪个后端"
#if defined(USE_X11)
    #include <X11/Xlib.h>
    void createWindow() { /* X11 实现 */ }
#elif defined(USE_SDL2)
    #include <SDL2/SDL.h>
    void createWindow() { /* SDL2 实现 */ }
#endif
```

**总结**：编译器预定义宏告诉你"目标平台是什么"，CMake 传递的宏告诉你"项目决定怎么做"。两者配合使用。

---

## `target_compile_definitions()` 详解

### 基本语法

```cmake
target_compile_definitions(<target> <PRIVATE|PUBLIC|INTERFACE>
    MACRO1                # 定义无值宏（等价于 #define MACRO1）
    MACRO2=value          # 定义有值宏（等价于 #define MACRO2 value）
    MACRO3="string"       # 定义字符串宏
)
```

这条命令的效果等价于在编译命令行中添加 `-DMACRO1 -DMACRO2=value`（GCC/Clang）或 `/DMACRO1 /DMACRO2=value`（MSVC）。

### 可见性关键字

这是很多初学者困惑的地方。三个关键字决定了宏的"传播范围"：

| 关键字 | 当前目标看得到？ | 链接到此目标的其他目标看得到？ | 使用场景 |
|--------|:-:|:-:|---------|
| `PRIVATE` | ✅ | ❌ | 仅当前目标内部使用的宏 |
| `PUBLIC` | ✅ | ✅ | 当前目标和使用者都需要的宏 |
| `INTERFACE` | ❌ | ✅ | 仅暴露给使用者的宏（头文件库常用） |

让我们用一个具体的例子来理解：

```cmake
# 假设有两个目标：engine（库）和 game（可执行文件，链接 engine）

# 场景 1：PRIVATE — engine 内部调试宏
target_compile_definitions(engine PRIVATE
    ENGINE_INTERNAL_DEBUG   # 只有 engine 的 .cpp 文件能看到
)

# 场景 2：PUBLIC — 引擎版本号，engine 和 game 都需要
target_compile_definitions(engine PUBLIC
    ENGINE_VERSION=2        # engine 的代码和 game 的代码都能看到
)

# 场景 3：INTERFACE — KrKr2 的 krkr2core 是 INTERFACE 库
# INTERFACE 库不编译任何 .cpp，只提供头文件和编译定义给使用者
add_library(krkr2core INTERFACE)
target_compile_definitions(krkr2core INTERFACE
    TJS_TEXT_OUT_CRLF       # krkr2core 自己不编译代码，
    USE_UNICODE_FSTRING     # 但链接到它的 krkr2 可执行文件会继承这些定义
)
```

> **KrKr2 为什么用 INTERFACE？**
>
> KrKr2 的 `krkr2core` 是一个 `INTERFACE` 库——它不编译任何代码，只是把一堆子模块（`tjs2`、`core_base_module` 等）的链接关系和编译定义"打包"在一起。当 `krkr2` 可执行文件链接 `krkr2core` 时，它自动继承所有子模块的头文件路径、编译定义和链接库。

### 实际效果演示

创建以下文件来验证可见性的效果：

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.28)
project(visibility_demo)
set(CMAKE_CXX_STANDARD 17)

# 库目标
add_library(mylib mylib.cpp)
target_compile_definitions(mylib PRIVATE   LIB_PRIVATE_MACRO)
target_compile_definitions(mylib PUBLIC    LIB_PUBLIC_MACRO)
target_compile_definitions(mylib INTERFACE LIB_INTERFACE_MACRO)

# 可执行目标，链接 mylib
add_executable(myapp main.cpp)
target_link_libraries(myapp PRIVATE mylib)
```

```cpp
// mylib.cpp
#include <iostream>

void libFunction() {
#ifdef LIB_PRIVATE_MACRO
    std::cout << "LIB_PRIVATE_MACRO: 可见 ✅" << std::endl;
#endif
#ifdef LIB_PUBLIC_MACRO
    std::cout << "LIB_PUBLIC_MACRO: 可见 ✅" << std::endl;
#endif
#ifdef LIB_INTERFACE_MACRO
    std::cout << "LIB_INTERFACE_MACRO: 可见 ✅" << std::endl;
#else
    std::cout << "LIB_INTERFACE_MACRO: 不可见 ❌" << std::endl;
#endif
}
```

```cpp
// main.cpp
#include <iostream>
extern void libFunction();

int main() {
    std::cout << "=== 在 mylib.cpp 中 ===" << std::endl;
    libFunction();

    std::cout << "\n=== 在 main.cpp 中 ===" << std::endl;
#ifdef LIB_PRIVATE_MACRO
    std::cout << "LIB_PRIVATE_MACRO: 可见 ✅" << std::endl;
#else
    std::cout << "LIB_PRIVATE_MACRO: 不可见 ❌" << std::endl;
#endif
#ifdef LIB_PUBLIC_MACRO
    std::cout << "LIB_PUBLIC_MACRO: 可见 ✅" << std::endl;
#endif
#ifdef LIB_INTERFACE_MACRO
    std::cout << "LIB_INTERFACE_MACRO: 可见 ✅" << std::endl;
#endif
    return 0;
}
```

运行结果：

```
=== 在 mylib.cpp 中 ===
LIB_PRIVATE_MACRO: 可见 ✅
LIB_PUBLIC_MACRO: 可见 ✅
LIB_INTERFACE_MACRO: 不可见 ❌

=== 在 main.cpp 中 ===
LIB_PRIVATE_MACRO: 不可见 ❌
LIB_PUBLIC_MACRO: 可见 ✅
LIB_INTERFACE_MACRO: 可见 ✅
```

这清楚地展示了三种可见性的传播行为。

---

## Generator Expressions（生成器表达式）

Generator Expressions 是 CMake 的一个强大特性，允许你在构建时（而非配置时）根据条件动态生成值。语法是 `$<条件:值>`。

### 基本语法

```cmake
# 基本形式：如果条件为真，展开为 value；否则展开为空字符串
$<条件:value>

# 布尔条件：如果变量为真值（TRUE、ON、1、非空）
$<$<BOOL:${MY_VAR}>:MACRO_NAME>

# 配置类型条件
$<$<CONFIG:Debug>:_DEBUG>         # Debug 构建时定义 _DEBUG
$<$<CONFIG:Release>:NDEBUG>      # Release 构建时定义 NDEBUG

# 平台条件
$<$<PLATFORM_ID:Windows>:WIN32_LEAN_AND_MEAN>  # Windows 时定义

# 组合条件（AND、OR、NOT）
$<$<AND:$<CONFIG:Debug>,$<PLATFORM_ID:Windows>>:WIN_DEBUG>
$<$<OR:$<PLATFORM_ID:Linux>,$<PLATFORM_ID:Darwin>>:UNIX_LIKE>
$<$<NOT:$<PLATFORM_ID:Windows>>:NOT_WINDOWS>
```

### KrKr2 中的实际用法

```cmake
# KrKr2 根 CMakeLists.txt 中的平台定义（简化版）

# 先设置 CMake 变量
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    set(WINDOWS TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(LINUX TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set(MACOS TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Android")
    set(ANDROID TRUE)
endif()

# 再通过 Generator Expression 传递给 C++ 代码
target_compile_definitions(krkr2core INTERFACE
    # 平台宏：只在对应平台上定义
    $<$<BOOL:${WINDOWS}>:WINDOWS=1>
    $<$<BOOL:${LINUX}>:LINUX=1>
    $<$<BOOL:${MACOS}>:MACOS=1>
    $<$<BOOL:${ANDROID}>:ANDROID=1>

    # 通用定义：所有平台都需要
    TJS_TEXT_OUT_CRLF
    __STDC_CONSTANT_MACROS
    USE_UNICODE_FSTRING
)
```

这里用 `$<$<BOOL:${WINDOWS}>:WINDOWS=1>` 而不是简单的 `if(WINDOWS)` + `target_compile_definitions`，原因是 Generator Expression 在**生成构建系统时**求值，而 `if()` 在**配置时**求值。对于像 `CONFIG` 这样只有生成时才知道的条件，必须用 Generator Expression。

### 常用 Generator Expression 模式

```cmake
# 模式 1：Debug/Release 差异化定义
target_compile_definitions(my_target PRIVATE
    $<$<CONFIG:Debug>:_DEBUG>
    $<$<CONFIG:Debug>:KR2_LOG_LEVEL=0>     # Debug: 全部日志
    $<$<CONFIG:Release>:NDEBUG>
    $<$<CONFIG:Release>:KR2_LOG_LEVEL=3>   # Release: 仅 Error
)

# 模式 2：编译器特定定义
target_compile_definitions(my_target PRIVATE
    $<$<CXX_COMPILER_ID:MSVC>:_CRT_SECURE_NO_WARNINGS>
    $<$<CXX_COMPILER_ID:MSVC>:_SILENCE_ALL_CXX17_DEPRECATION_WARNINGS>
)

# 模式 3：多配置（Visual Studio / Xcode）安全写法
# 不要用 if(CMAKE_BUILD_TYPE STREQUAL "Debug")
# 因为多配置生成器中 CMAKE_BUILD_TYPE 在配置时为空
target_compile_definitions(my_target PRIVATE
    $<$<CONFIG:Debug>:ENABLE_PROFILING>     # ✅ 正确
)
```

---

## `configure_file()` 生成配置头文件

### 为什么需要 configure_file

有些信息在 CMake 配置时才知道（如版本号、Git 哈希、功能开关），你想把它们嵌入到 C++ 代码中。`configure_file()` 可以基于模板文件生成 C++ 头文件。

### 基本用法

```cmake
# CMakeLists.txt
set(PROJECT_VERSION_MAJOR 2)
set(PROJECT_VERSION_MINOR 1)
set(PROJECT_VERSION_PATCH 0)
set(HAS_OPENGL TRUE)
set(HAS_VULKAN FALSE)

# 从模板生成头文件
configure_file(
    ${CMAKE_SOURCE_DIR}/config.h.in      # 输入模板
    ${CMAKE_BINARY_DIR}/generated/config.h  # 输出文件
)

# 让代码能找到生成的头文件
target_include_directories(my_target PRIVATE
    ${CMAKE_BINARY_DIR}/generated
)
```

### 模板文件语法

```cpp
// config.h.in — 模板文件
#pragma once

// @VAR@ 语法：直接替换为 CMake 变量的值
#define PROJECT_VERSION_MAJOR @PROJECT_VERSION_MAJOR@
#define PROJECT_VERSION_MINOR @PROJECT_VERSION_MINOR@
#define PROJECT_VERSION_PATCH @PROJECT_VERSION_PATCH@

// #cmakedefine 语法：
// 如果变量为 TRUE/ON/非空 → 生成 #define MACRO
// 如果变量为 FALSE/OFF/空/未定义 → 生成 /* #undef MACRO */
#cmakedefine HAS_OPENGL
#cmakedefine HAS_VULKAN

// #cmakedefine01 语法：
// 变量为真 → #define MACRO 1
// 变量为假 → #define MACRO 0
#cmakedefine01 HAS_OPENGL
```

生成的 `config.h`：

```cpp
// config.h — 自动生成，不要手动编辑！
#pragma once

#define PROJECT_VERSION_MAJOR 2
#define PROJECT_VERSION_MINOR 1
#define PROJECT_VERSION_PATCH 0

#define HAS_OPENGL
/* #undef HAS_VULKAN */

#define HAS_OPENGL 1
```

### 在代码中使用

```cpp
#include "config.h"
#include <iostream>

int main() {
    std::cout << "版本: " << PROJECT_VERSION_MAJOR << "."
              << PROJECT_VERSION_MINOR << "."
              << PROJECT_VERSION_PATCH << std::endl;

#ifdef HAS_OPENGL
    std::cout << "OpenGL: 已启用" << std::endl;
    // 初始化 OpenGL...
#endif

#ifdef HAS_VULKAN
    std::cout << "Vulkan: 已启用" << std::endl;
    // 初始化 Vulkan...
#else
    std::cout << "Vulkan: 未启用" << std::endl;
#endif

    return 0;
}
```

### 进阶：嵌入 Git 信息

```cmake
# 获取 Git 提交哈希
execute_process(
    COMMAND git rev-parse --short HEAD
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_HASH
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET  # Git 不可用时不报错
)

if(NOT GIT_HASH)
    set(GIT_HASH "unknown")
endif()

set(BUILD_TIMESTAMP "${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR}")

configure_file(build_info.h.in ${CMAKE_BINARY_DIR}/generated/build_info.h)
```

```cpp
// build_info.h.in
#pragma once
#define GIT_HASH "@GIT_HASH@"
#define BUILD_PLATFORM "@BUILD_TIMESTAMP@"
```

```cpp
// 使用
#include "build_info.h"
std::cout << "Build: " << GIT_HASH << " on " << BUILD_PLATFORM << std::endl;
// 输出: Build: a3b2c1d on Windows-AMD64
```

---

## `target_compile_definitions` vs `configure_file` 对比

两种方法都能将信息传递给 C++ 代码，各有适用场景：

| 特性 | `target_compile_definitions` | `configure_file` |
|------|:---:|:---:|
| 无值宏（开关） | ✅ 简单 | ✅ 可以 |
| 数值宏 | ✅ 简单 | ✅ 可以 |
| 字符串宏 | ⚠️ 引号转义复杂 | ✅ 自然 |
| 复杂配置（多个相关宏） | ❌ 冗长 | ✅ 模板清晰 |
| IDE 支持（代码补全） | ❌ IDE 可能不识别 | ✅ 头文件，IDE 完整支持 |
| Generator Expression | ✅ 支持 | ❌ 不支持 |
| 按构建类型切换 | ✅ `$<CONFIG:Debug>` | ❌ 配置时固定 |
| 典型用途 | 平台宏、功能开关 | 版本号、Git 哈希、特性检测 |

**实践建议**：
- 简单的开关宏（`WINDOWS`、`_DEBUG`） → `target_compile_definitions`
- 版本号、路径、字符串 → `configure_file`
- 需要根据 Debug/Release 切换 → `target_compile_definitions` + Generator Expression

---

## KrKr2 条件编译实例分析

### 实例 1：平台入口点——文件级条件编译

KrKr2 没有在一个 `main.cpp` 中用 `#ifdef` 区分平台，而是为每个平台提供独立的入口文件：

```
platforms/
├── windows/main.cpp     ← WinMain() 入口
├── linux/main.cpp       ← int main() 入口
├── apple/macos/main.cpp ← int main() + NSApplication
└── android/cpp/
    └── krkr2_android.cpp ← JNI_OnLoad() 入口
```

CMake 根据平台变量选择编译哪个文件：

```cmake
# 根据平台选择入口文件（KrKr2 的实际做法简化版）
if(WINDOWS)
    # WIN32 参数让 MSVC 使用 WinMain 入口而非 console main
    add_executable(krkr2 WIN32 platforms/windows/main.cpp)
elseif(LINUX)
    add_executable(krkr2 platforms/linux/main.cpp)
elseif(MACOS)
    # MACOSX_BUNDLE 创建 .app 应用包
    add_executable(krkr2 MACOSX_BUNDLE platforms/apple/macos/main.cpp)
elseif(ANDROID)
    # Android 上是共享库（.so），不是可执行文件
    add_library(krkr2 SHARED platforms/android/cpp/krkr2_android.cpp)
endif()
```

这种方式叫做**文件级条件编译**——比在一个文件里塞满 `#ifdef` 更清晰。每个文件只包含一个平台的代码，更容易阅读和维护。

### 实例 2：krkr2core 的编译定义传递

```cmake
# cpp/core/CMakeLists.txt（实际代码简化版）
add_library(krkr2core INTERFACE)

target_compile_definitions(krkr2core INTERFACE
    TJS_TEXT_OUT_CRLF           # TJS2 脚本输出使用 CRLF 换行
    __STDC_CONSTANT_MACROS      # 启用 C99 常量宏（INT64_C 等）
    USE_UNICODE_FSTRING         # 使用 Unicode 格式化字符串
)

# krkr2core 链接所有子模块
target_link_libraries(krkr2core INTERFACE
    tjs2
    core_base_module
    core_environ_module
    core_visual_module
    core_sound_module
    core_movie_module
    # ...
)
```

当 `krkr2` 链接 `krkr2core` 时：
```cmake
target_link_libraries(krkr2 PRIVATE krkr2core)
```

`krkr2` 自动继承了 `TJS_TEXT_OUT_CRLF`、`USE_UNICODE_FSTRING` 等所有定义，以及所有子模块的链接关系。这就是 INTERFACE 库的威力。

### 实例 3：OpenMP 的条件启用

KrKr2 在音频解码和图像处理中使用 OpenMP 并行化。但 Apple Clang 默认不支持 OpenMP：

```cmake
# 根 CMakeLists.txt
if(NOT APPLE)
    find_package(OpenMP)
    if(OpenMP_CXX_FOUND)
        target_link_libraries(krkr2core INTERFACE OpenMP::OpenMP_CXX)
    endif()
endif()
```

对应的 C++ 代码通过 `_OPENMP` 宏检查 OpenMP 是否可用（这个宏由编译器在启用 OpenMP 时自动定义）：

```cpp
// 图像像素处理中的 OpenMP 并行化
void processImage(uint32_t* pixels, int width, int height) {
    int totalPixels = width * height;

#ifdef _OPENMP
    // OpenMP 可用：并行处理像素
    #pragma omp parallel for schedule(dynamic, 1024)
    for (int i = 0; i < totalPixels; ++i) {
        pixels[i] = applyFilter(pixels[i]);
    }
#else
    // OpenMP 不可用（macOS）：串行处理
    for (int i = 0; i < totalPixels; ++i) {
        pixels[i] = applyFilter(pixels[i]);
    }
#endif
}
```

### 实例 4：environ 目录的文件级分离

```
cpp/core/environ/
├── cocos2d/       ← 所有平台共用的 Cocos2d-x 桥接层
├── win32/         ← Windows 专用实现（Win32 API）
├── linux/         ← Linux 专用实现（X11/Wayland）
├── android/       ← Android 专用实现（JNI 桥接）
├── apple/         ← macOS 专用实现（Cocoa）
├── sdl/           ← SDL2 后端（可选替代方案）
└── ui/            ← Cocos2d-x UI 组件（跨平台）
```

CMake 只编译当前平台对应目录中的源文件。这种"每个平台一个目录"的模式是大型跨平台项目的标准做法，我们将在第三章"平台抽象层设计模式"中详细讲解。

---

## 常见错误与排查

### 错误 1：add_definitions 的陷阱

```cmake
# ❌ 旧式写法（CMake 2.x 遗留）——不推荐
add_definitions(-DWINDOWS=1)
# 问题：对所有目标生效，无法控制可见性
```

```cmake
# ✅ 现代写法——精确控制每个目标
target_compile_definitions(krkr2 PRIVATE WINDOWS=1)
```

`add_definitions()` 是全局的，会影响当前目录下的**所有**目标。在有多个目标的项目中（如 KrKr2 有 `krkr2`、`krkr2core`、`krkr2plugin`、测试目标等），这会导致不需要的宏泄漏到其他目标。

### 错误 2：字符串宏的引号问题

```cmake
# ❌ 错误：引号在某些平台/生成器上可能被吞掉
target_compile_definitions(my_target PRIVATE
    VERSION_STRING="2.0.1"
)

# ✅ 更安全的方式：使用 configure_file
# 或者使用转义（但可移植性差）：
target_compile_definitions(my_target PRIVATE
    VERSION_STRING=\"2.0.1\"
)
```

字符串宏在命令行传递时，引号的处理在不同系统（bash/cmd/powershell）和不同生成器（Ninja/Make/VS）之间行为不一致。**最安全的做法是用 `configure_file()` 生成包含字符串常量的头文件**。

### 错误 3：混淆配置时和生成时

```cmake
# ❌ 错误：CMAKE_BUILD_TYPE 在多配置生成器中为空
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_definitions(my_target PRIVATE _DEBUG)
endif()
# 在 Visual Studio 和 Xcode 中，CMAKE_BUILD_TYPE 在配置时未设置！
# 用户在 IDE 中选择 Debug/Release 是在构建时才确定的

# ✅ 正确：使用 Generator Expression
target_compile_definitions(my_target PRIVATE
    $<$<CONFIG:Debug>:_DEBUG>
    $<$<CONFIG:Release>:NDEBUG>
)
```

---

## 动手实践

### 练习：创建一个带版本信息的跨平台项目

创建以下目录结构：

```
my_project/
├── CMakeLists.txt
├── version.h.in
└── main.cpp
```

**步骤 1**：编写 `version.h.in` 模板

```cpp
// version.h.in
#pragma once

// 版本信息（由 CMake 生成）
#define APP_NAME "@PROJECT_NAME@"
#define APP_VERSION_MAJOR @PROJECT_VERSION_MAJOR@
#define APP_VERSION_MINOR @PROJECT_VERSION_MINOR@

// 平台检测结果
#cmakedefine IS_WINDOWS
#cmakedefine IS_LINUX
#cmakedefine IS_MACOS
#cmakedefine IS_ANDROID

// 功能开关
#cmakedefine01 ENABLE_LOGGING
```

**步骤 2**：编写 `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.28)
project(my_project VERSION 1.0)
set(CMAKE_CXX_STANDARD 17)

# 平台检测
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    set(IS_WINDOWS TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(IS_LINUX TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set(IS_MACOS TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Android")
    set(IS_ANDROID TRUE)
endif()

# 功能开关
option(ENABLE_LOGGING "Enable debug logging" ON)

# 生成版本头文件
configure_file(
    ${CMAKE_SOURCE_DIR}/version.h.in
    ${CMAKE_BINARY_DIR}/generated/version.h
)

# 创建可执行文件
add_executable(my_project main.cpp)

# 让代码能找到生成的 version.h
target_include_directories(my_project PRIVATE
    ${CMAKE_BINARY_DIR}/generated
)

# 通过 target_compile_definitions 传递构建类型
target_compile_definitions(my_project PRIVATE
    $<$<CONFIG:Debug>:BUILD_TYPE="Debug">
    $<$<CONFIG:Release>:BUILD_TYPE="Release">
    $<$<CONFIG:RelWithDebInfo>:BUILD_TYPE="RelWithDebInfo">
)
```

**步骤 3**：编写 `main.cpp`

```cpp
// main.cpp
#include "version.h"
#include <iostream>

int main() {
    std::cout << APP_NAME << " v"
              << APP_VERSION_MAJOR << "."
              << APP_VERSION_MINOR << std::endl;

    // 平台信息
    std::cout << "平台: ";
#ifdef IS_WINDOWS
    std::cout << "Windows" << std::endl;
#elif defined(IS_LINUX)
    std::cout << "Linux" << std::endl;
#elif defined(IS_MACOS)
    std::cout << "macOS" << std::endl;
#elif defined(IS_ANDROID)
    std::cout << "Android" << std::endl;
#else
    std::cout << "Unknown" << std::endl;
#endif

    // 功能检测
#if ENABLE_LOGGING
    std::cout << "日志: 已启用" << std::endl;
#else
    std::cout << "日志: 已禁用" << std::endl;
#endif

    // 构建类型（通过 Generator Expression 传入）
#ifdef BUILD_TYPE
    std::cout << "构建: " << BUILD_TYPE << std::endl;
#endif

    return 0;
}
```

**步骤 4**：编译并运行

```bash
# 配置（Linux/macOS）
cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
./build/my_project

# 配置（Windows MSVC）
cmake -B build -G "Visual Studio 17 2022"
cmake --build build --config Debug
build\Debug\my_project.exe

# 禁用日志功能重新构建
cmake -B build -DENABLE_LOGGING=OFF
cmake --build build
```

---

## 对照项目源码

### 关键文件

- `krkr2/CMakeLists.txt` — 根 CMake 文件，定义了 `WINDOWS`/`LINUX`/`MACOS`/`ANDROID` 自定义变量和 Generator Expression 传递
- `krkr2/cpp/core/CMakeLists.txt` — `krkr2core` INTERFACE 库的定义，展示了 `INTERFACE` 可见性的实际用法
- `krkr2/platforms/` — 每个平台的入口文件，展示了文件级条件编译模式
- `krkr2/tests/unit-tests/` — 测试目标使用 `configure_file` 传递 `TEST_FILES_PATH` 测试数据路径

---

## 本节小结

- **`target_compile_definitions()`** 是从 CMake 向 C++ 传递宏定义的标准方法，替代了旧式的 `add_definitions()`
- **可见性三选一**：`PRIVATE`（仅当前目标）、`PUBLIC`（当前 + 使用者）、`INTERFACE`（仅使用者）
- **Generator Expressions** 让你在构建时动态设置宏，支持 `$<CONFIG:Debug>`、`$<PLATFORM_ID:Windows>` 等条件
- **`configure_file()`** 适合传递版本号、路径、字符串等复杂信息，生成的头文件对 IDE 友好
- KrKr2 综合使用了两种方式：`target_compile_definitions` 传递平台和功能开关，`configure_file` 传递测试路径
- **文件级条件编译**（每个平台一个文件）比行内 `#ifdef` 更清晰，是大型项目的标准做法

---

## 练习题与答案

### 题目 1：可见性选择

一个项目有三个 CMake 目标：`core_lib`（库）、`plugin_lib`（库，链接 `core_lib`）、`app`（可执行文件，链接 `plugin_lib`）。现在需要定义宏 `CORE_VERSION=3`，要求：
- `core_lib` 的代码能看到
- `plugin_lib` 的代码能看到
- `app` 的代码**看不到**

应该用哪个可见性关键字？

<details>
<summary>查看答案</summary>

使用 `PUBLIC`：

```cmake
target_compile_definitions(core_lib PUBLIC CORE_VERSION=3)
```

**解析**：
- `PUBLIC` 让 `core_lib` 自己和直接链接它的 `plugin_lib` 都能看到 → ✅
- `plugin_lib` 链接 `core_lib` 时用的是 `PRIVATE`（`target_link_libraries(plugin_lib PRIVATE core_lib)`），所以 `CORE_VERSION` 不会传播到 `app` → ✅
- 如果 `plugin_lib` 用 `PUBLIC` 链接 `core_lib`，则 `app` 也能看到（这时需要改用 `PRIVATE`）

**如果需要阻止传播**，可以在 `plugin_lib` 中用 `PRIVATE` 链接：
```cmake
target_link_libraries(plugin_lib PRIVATE core_lib)
# core_lib 的 PUBLIC 定义不会传播到 app
```

</details>

### 题目 2：修复多配置生成器 Bug

以下 CMake 代码在 Ninja（单配置生成器）上正常工作，但在 Visual Studio（多配置生成器）上 Debug 构建时 `_DEBUG` 未被定义。找出原因并修复。

```cmake
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_definitions(my_target PRIVATE _DEBUG)
endif()
```

<details>
<summary>查看答案</summary>

**原因**：在多配置生成器（Visual Studio、Xcode）中，`CMAKE_BUILD_TYPE` 在 CMake 配置阶段是**空的**。用户在 IDE 中选择 Debug/Release 是在构建阶段才确定的，此时 CMake 的 `if()` 已经执行完了。

**修复**：使用 Generator Expression，它在构建时求值：

```cmake
target_compile_definitions(my_target PRIVATE
    $<$<CONFIG:Debug>:_DEBUG>
)
```

`$<$<CONFIG:Debug>:_DEBUG>` 的含义是："如果当前构建配置是 Debug，则定义 `_DEBUG`"。这在构建时才求值，所以在多配置生成器中也能正确工作。

</details>

### 题目 3：configure_file 模板

编写一个 `config.h.in` 模板文件，满足以下需求：
1. 包含项目名称（字符串）
2. 包含版本号（三段式：major.minor.patch）
3. 如果 CMake 变量 `ENABLE_OPENGL` 为 TRUE，定义 `HAS_OPENGL` 宏
4. 如果 CMake 变量 `ENABLE_VULKAN` 为 TRUE，定义 `HAS_VULKAN` 宏；否则生成注释

<details>
<summary>查看答案</summary>

```cpp
// config.h.in
#pragma once

// 项目信息
#define PROJECT_NAME "@PROJECT_NAME@"
#define VERSION_MAJOR @PROJECT_VERSION_MAJOR@
#define VERSION_MINOR @PROJECT_VERSION_MINOR@
#define VERSION_PATCH @PROJECT_VERSION_PATCH@

// 完整版本字符串
#define VERSION_STRING "@PROJECT_VERSION_MAJOR@.@PROJECT_VERSION_MINOR@.@PROJECT_VERSION_PATCH@"

// 渲染后端
#cmakedefine HAS_OPENGL
#cmakedefine HAS_VULKAN
```

对应的 CMakeLists.txt：

```cmake
cmake_minimum_required(VERSION 3.28)
project(my_game VERSION 2.1.3)

option(ENABLE_OPENGL "Enable OpenGL renderer" ON)
option(ENABLE_VULKAN "Enable Vulkan renderer" OFF)

# 将 option 值映射到 cmakedefine 能识别的变量
if(ENABLE_OPENGL)
    set(HAS_OPENGL TRUE)
endif()
if(ENABLE_VULKAN)
    set(HAS_VULKAN TRUE)
endif()

configure_file(config.h.in ${CMAKE_BINARY_DIR}/generated/config.h)
```

生成的 `config.h`（默认选项下）：

```cpp
#pragma once

#define PROJECT_NAME "my_game"
#define VERSION_MAJOR 2
#define VERSION_MINOR 1
#define VERSION_PATCH 3

#define VERSION_STRING "2.1.3"

#define HAS_OPENGL
/* #undef HAS_VULKAN */
```

</details>

---

## 下一步

本节讲解了 CMake 如何向 C++ 代码传递信息。在实际开发中，条件编译的滥用会导致代码变成"预处理器地狱"——到处是 `#ifdef`，难以阅读和维护。

下一节 [最佳实践与常见错误](./03-最佳实践与常见错误.md) 将讲解：

- 何时用行内 `#ifdef`，何时用文件级分离
- 避免"预处理器地狱"的实战策略
- 宏定义的命名规范和作用域控制
- KrKr2 项目中好的和不好的条件编译实践
