# 02-CMake 核心概念

> **所属模块：** P01-现代CMake与构建工具链
> **前置知识：** [01-构建系统的演进](./01-构建系统的演进.md)
> **预计阅读时间：** 20 分钟

## 元数据块
| 字段 | 内容 |
|---|---|
| 章节编号 | P01 / 01 / 02 |
| 章节名称 | CMake 核心概念 |
| 适用平台 | Windows / Linux / macOS / Android |
| 推荐生成器 | Ninja（跨平台一致） |
| 参考项目 | `krkr2/CMakeLists.txt` |
| 学完可达成 | 能读懂中大型 CMake 项目的主配置结构 |

## 本节目标

读完本节后，你将能够：
1. 掌握 `CMakeLists.txt` 的作用与命名规范
2. 理解什么是"源外构建（Out-of-source Build）"及其重要性
3. 理解 CMake 的三个基本阶段：配置、生成和构建

## CMakeLists.txt：项目的剧本

在 CMake 的世界里，一切都始于一个名为 `CMakeLists.txt` 的文件。

- **它是入口**：CMake 会首先寻找并读取这个文件。
- **它是大小写敏感的**：必须叫 `CMakeLists.txt`，不能叫 `cmakelists.txt`。
- **它是声明式的**：你在里面告诉 CMake "我想要一个叫 hello 的可执行文件，它由 main.cpp 编译而来"，而不是写具体的编译命令。

### CMakeLists.txt 的角色与解析规则

在中大型项目里，`CMakeLists.txt` 是构建模型定义文件、平台分发入口和工程聚合器。

解析规则抓住三点即可：

- **顺序执行**：按文件顺序解释。
- **作用域层级**：目录、函数、缓存作用域会影响变量可见性。
- **列表语义**：参数本质是列表，变量展开后分号也会分割列表。

```cmake
# 示例 2：最小可运行 CMakeLists（含注释）
cmake_minimum_required(VERSION 3.28)         # 指定最低 CMake 版本
project(hello_cmake LANGUAGES C CXX)         # 定义工程名与语言

set(CMAKE_CXX_STANDARD 17)                   # 设定 C++ 标准
add_executable(hello main.cpp)               # 声明可执行目标 hello

target_compile_definitions(hello PRIVATE APP_NAME="hello")
target_compile_options(hello PRIVATE
    $<$<CXX_COMPILER_ID:MSVC>:/W4>           # MSVC 警告级别
    $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall>  # GCC/Clang 警告级别
)
```

这个例子可直接配合 `main.cpp` 运行，能快速理解“目标驱动”思想。

## 源外构建（Out-of-source Build）

这是一个非常重要的习惯，也是 CMake 强烈推荐的做法。

### 什么是"源内构建"（In-source Build）？
如果你在代码所在的目录下直接运行 CMake，它会生成大量的临时文件（如 `CMakeFiles/`, `CMakeCache.txt`, `Makefile` 等），这些文件会和你的 `.cpp`、`.h` 源码混杂在一起。这会污染你的源码目录，让版本控制（如 Git）变得非常痛苦。

### 什么是"源外构建"？
我们将所有的编译产物放在一个独立的文件夹里（通常叫 `build`）。

**操作步骤：**
1. 在项目根目录下创建一个 `build` 文件夹。
2. 进入 `build` 文件夹运行 CMake。
3. 所有的临时文件和最终生成的可执行文件都会留在 `build` 文件夹内。

如果你想重新开始，只需要删除 `build` 文件夹，你的源码依然干干净净。

```bash
# 典型的操作流程
mkdir build
cd build
cmake ..  # ".." 代表告诉 CMake 去上一级目录找 CMakeLists.txt
```

### Out-of-source 的原理和好处

当你执行 `cmake -S . -B build` 时，`-S` 是源码输入目录，`-B` 是构建输出目录。

CMake 会把缓存、生成器文件、中间产物都写到 `build`，收益是：

1. **多配置并存**：同一份源码可同时有 `build-debug`、`build-release`。
2. **多平台并存**：同一份源码可同时有 `build-linux`、`build-windows`。
3. **可重复构建**：删除构建目录即可回到干净状态，排查问题更快。

```bash
# 示例 6：同一源码目录并行维护多个构建目录
cmake -S . -B build-debug -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake -S . -B build-release -G Ninja -DCMAKE_BUILD_TYPE=Release

cmake --build build-debug
cmake --build build-release
```

## 生成器（Generator）

还记得我们说过 CMake 不直接编译代码吗？它需要一个**生成器**来生成真正的构建文件。

- **Unix Makefiles**：生成经典的 Makefile（Linux/macOS 默认）。
- **Visual Studio 17 2022**：生成 Windows 下的 `.sln` 和 `.vcxproj`。
- **Xcode**：生成 macOS/iOS 下的 `.xcodeproj`。
- **Ninja**：一个跨平台、极速的构建系统，这也是 KrKr2 项目推荐使用的生成器。

你可以通过 `-G` 参数指定生成器，例如：`cmake -G Ninja ..`

### 生成器对比：Makefiles / Ninja / Visual Studio / Xcode

- **Unix Makefiles**：Linux/macOS 常见，生态传统稳定。
- **Ninja**：跨平台一致、速度快，适合本地与 CI。
- **Visual Studio**：Windows 原生 IDE 工程。
- **Xcode**：macOS 原生工程。

```bash
# 示例 5：常见生成器切换
cmake -S . -B build-ninja -G Ninja
cmake -S . -B build-vs -G "Visual Studio 17 2022" -A x64
cmake -S . -B build-xcode -G Xcode
```

KrKr2 教程里优先推荐 Ninja，因为它在本地开发和 CI 中都更一致。

## 目标（Target）：CMake 的灵魂

在现代 CMake（3.0+）中，**一切皆目标（Everything is a target）**。一个 CMake 项目实际上是多个目标的集合。

### 目标类型深入：什么时候该选哪一种

- `add_executable`：最终程序入口。
- `add_library(名称 STATIC 源文件)`：编译后打包进最终产物。
- `add_library(名称 SHARED 源文件)`：运行时动态装载。
- `add_library(名称 INTERFACE)`：不产出二进制，只传播编译属性。

```cmake
# 示例 3：四类目标完整声明
cmake_minimum_required(VERSION 3.28)
project(target_showcase LANGUAGES CXX)

add_library(math_static STATIC src/math.cpp)
add_library(render_shared SHARED src/render.cpp)
add_library(config_header_only INTERFACE)
add_executable(game_app src/main.cpp)

target_include_directories(config_header_only INTERFACE include)
target_compile_features(config_header_only INTERFACE cxx_std_17)

target_link_libraries(game_app PRIVATE
    math_static
    render_shared
    config_header_only
)
```

上面这个配置的关键点是：`config_header_only` 虽然不产出库文件，但它把头文件路径和编译标准传播给 `game_app`。

### Property 系统：目标属性、目录属性、全局属性

Property（属性）是 CMake 的“状态存储机制”。

1. **目标属性（Target Property）**：只作用于某个目标，最常用。
2. **目录属性（Directory Property）**：作用于当前目录和子目录。
3. **全局属性（Global Property）**：跨目录共享的全局状态。

```cmake
# 示例 4：设置与读取不同层级的属性
add_executable(demo_app main.cpp)

# 目标属性：为 demo_app 单独设置 C++ 标准
set_property(TARGET demo_app PROPERTY CXX_STANDARD 17)
get_property(app_std TARGET demo_app PROPERTY CXX_STANDARD)
message(STATUS "demo_app CXX_STANDARD = ${app_std}")

# 目录属性：给当前目录设置额外包含路径
set_property(DIRECTORY PROPERTY INCLUDE_DIRECTORIES "${CMAKE_CURRENT_SOURCE_DIR}/include")

# 全局属性：记录一个构建元信息
set_property(GLOBAL PROPERTY KRKR2_TUTORIAL_STAGE "configure")
get_property(stage GLOBAL PROPERTY KRKR2_TUTORIAL_STAGE)
message(STATUS "Global stage = ${stage}")
```

如果你调试复杂工程，`get_property()` 配合 `message(STATUS "变量名=${变量}")` 是定位问题的高频组合。

## CMake 的三个阶段

理解这三个阶段，能帮你快速定位 90% 的构建错误。

### 三阶段模型详解：每个阶段到底在做什么

很多初学者把 `cmake` 当成“编译命令”，这是最容易混淆的点。准确说：

- `cmake -S . -B build` 触发 **配置（Configure）+ 生成（Generate）**。
- `cmake --build build` 才触发 **构建（Build）**。

你可以把它理解成“先写施工图，再让施工队施工”。

```bash
# 示例 1：显式三阶段（推荐教学写法）
cmake -S . -B build -G Ninja            # 配置 + 生成
cmake --build build                     # 构建
cmake --build build --target clean      # 清理当前生成器定义的产物
```

#### Configure（配置阶段）会做什么

1. 读取顶层和子目录的 `CMakeLists.txt`。
2. 解析 `project()`、`option()`、`find_package()` 等指令。
3. 检测编译器、系统、工具链是否可用。
4. 生成或更新 `CMakeCache.txt`。

这一步失败，常见日志是：找不到编译器、找不到包、变量配置冲突。

#### Generate（生成阶段）会做什么

1. 把“目标关系图”转换成生成器可识别的文件。
2. 例如 Ninja 生成 `build.ninja`，Visual Studio 生成 `.sln` 和 `.vcxproj`。
3. 处理平台差异与配置差异（Debug/Release）。

这一步失败，常见原因是目标之间依赖环、属性配置不完整。

#### Build（构建阶段）会做什么

1. 真正调用编译器（如 `cl`、`g++`、`clang++`）与链接器。
2. 按依赖顺序构建目标。
3. 输出可执行文件、静态库、动态库。

这一阶段出错通常就是 C++ 编译错误或链接错误，比如未定义符号。

## 本节小结

- `CMakeLists.txt` 是项目的核心配置文件。
- 始终坚持使用"源外构建"（Out-of-source Build），保持源码整洁。
- 目标（Target）是 CMake 的核心管理单位。
- CMake 构建分为配置、生成和构建三个阶段。
- Property 系统是定位复杂构建问题的关键抓手。
- 生成器与工具链决定了“产物格式”和“目标平台能力”。

## 工具链（Toolchain）概念：编译器、链接器与交叉编译

工具链（Toolchain）可以理解为“把源码变成目标平台二进制的一整套工具”，核心组成包括：

1. **编译器（Compiler）**：如 `clang++`、`g++`、`cl`，把 `.cpp` 编译成目标文件。
2. **链接器（Linker）**：把多个目标文件和库链接成最终产物。
3. **系统头文件与运行库**：决定 ABI、平台调用约定和可用 API。

KrKr2 的顶层配置在 Android 与非 Android 路径上有明显分流：

- Android 分支会 `include(cmake/vcpkg_android.cmake)`。
- 其他平台通过 `CMAKE_TOOLCHAIN_FILE` 使用 vcpkg 工具链。

这正是“工具链切换”的典型做法。

```cmake
# 示例 7：常见工具链入口（节选思想）
if(ANDROID)
    set(VCPKG_TARGET_ANDROID ON)
    include(cmake/vcpkg_android.cmake)
else()
    set(CMAKE_TOOLCHAIN_FILE $ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake)
endif()
```

#### 交叉编译（Cross Compilation）

交叉编译是指：在 A 平台上生成 B 平台可运行的程序。例如在 Windows 主机上生成 Android `arm64-v8a` 动态库。

在 CMake 中，交叉编译通常意味着你要明确：

- 目标系统（`CMAKE_SYSTEM_NAME`）
- 目标架构（如 arm64、x86_64）
- 对应 SDK/NDK 与 sysroot

KrKr2 的 Android 构建就是一个现实案例。

## 对照 KrKr2 项目源码

下面用 `krkr2/CMakeLists.txt` 的真实配置对照本节概念：

- `cmake_minimum_required(VERSION 3.28)`：定义最低 CMake 版本。
- `project(${APP_NAME})`：建立工程作用域和语言上下文。
- `if(ANDROID) elseif(LINUX) elseif(WINDOWS)`：平台分支。
- `add_library(${PROJECT_NAME} SHARED 平台入口文件)`：Android 下主目标是共享库。
- `add_executable(${PROJECT_NAME} 平台入口文件)`：Linux/Windows/macOS 下主目标是可执行文件。
- `add_subdirectory(${KRKR2CORE_PATH})` 与 `add_subdirectory(${KRKR2PLUGIN_PATH})`：模块化组织。
- `target_link_libraries(${PROJECT_NAME} PUBLIC krkr2plugin krkr2core)`：目标依赖连接。

这说明 KrKr2 已经采用了现代 CMake 的“目标 + 属性 + 目录分层”思路。

## 练习题与答案

### 题目 1：在项目根目录下创建一个名为 build 的文件夹，并在其中运行 cmake ..，这属于哪种构建方式？为什么要这么做？

<details>
<summary>查看答案</summary>

这属于"源外构建"（Out-of-source Build）。这样做是为了防止 CMake 生成的临时文件（如缓存、中间生成脚本等）污染源码目录。这也有利于版本管理（不需要在 .gitignore 中写一长串临时文件名），并且清理项目非常简单：直接删除整个 build 文件夹即可。

</details>

### 题目 2：如果你在控制台看到 "CMake Error: Could not create named generator" 错误，这通常发生在 CMake 的哪个阶段？

<details>
<summary>查看答案</summary>

这发生在"配置阶段"（Configure）。这个错误通常意味着你指定的生成器（如 -G "Ninja"）在当前系统中没有安装或不在 PATH 路径中。

</details>

### 题目 3：请列举出至少三种 CMake 目标的类型。

<details>
<summary>查看答案</summary>

1. 可执行文件（add_executable）
2. 静态库（add_library STATIC）
3. 共享库/动态库（add_library SHARED）
4. 接口库/头文件库（add_library INTERFACE）

</details>

## 动手实践

下面给你一个“最小但完整”的实践任务：创建一个同时包含可执行文件、静态库与接口库的小项目。

### 步骤 1：目录结构

```text
cmake-practice/
├─ CMakeLists.txt
├─ include/
│  └─ config.hpp
├─ src/
│  ├─ math.cpp
│  └─ main.cpp
```

### 步骤 2：写入代码
`include/config.hpp`：

```cpp
#pragma once

constexpr const char* APP_TITLE = "CMakePractice"; // 接口库传播头文件
int add_int(int a, int b);                          // 由静态库实现
```
`src/math.cpp`：

```cpp
#include "config.hpp"

int add_int(int a, int b) {
    return a + b; // 简单可验证逻辑
}
```
`src/main.cpp`：

```cpp
#include <iostream>
#include "config.hpp"

int main() {
    std::cout << APP_TITLE << " => 3 + 4 = " << add_int(3, 4) << "\n";
    return 0;
}
```
### 步骤 3：写入 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.28)
project(cmake_practice LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)

add_library(practice_headers INTERFACE)
target_include_directories(practice_headers INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include)

add_library(practice_math STATIC src/math.cpp)
target_link_libraries(practice_math PUBLIC practice_headers)

add_executable(practice_app src/main.cpp)
target_link_libraries(practice_app PRIVATE practice_math)
```
### 步骤 4：构建与运行
```bash
cmake -S . -B build -G Ninja
cmake --build build
# Windows
./build/practice_app.exe
# Linux/macOS
./build/practice_app
```
## 下一步

→ 继续阅读 [03-第一个CMake项目](./03-第一个CMake项目.md)
