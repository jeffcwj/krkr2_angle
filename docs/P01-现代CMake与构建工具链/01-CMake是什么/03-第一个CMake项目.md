# 03-第一个 CMake 项目

> **所属模块：** P01-现代CMake与构建工具链  
> **前置知识：** [02-CMake核心概念](./02-CMake核心概念.md)  
> **预计阅读时间：** 30 分钟  
> **适用平台：** Windows / Linux / macOS / Android

## 元数据块

| 字段 | 内容 |
|---|---|
| 教程类型 | P 系列前置知识 |
| 章节位置 | 01-CMake是什么 / 第3节 |
| 学习目标 | 独立创建并运行一个 CMake 项目 |
| 实操结果 | 完成单文件与多文件构建 |
| 后续衔接 | [01-变量与缓存](../02-CMake语法与命令/01-变量与缓存.md) |

## 本节目标

读完本节后，你将能够：
1. 从零创建一个 CMake 项目并跑通构建链路。
2. 理解 `cmake_minimum_required()` 的版本策略意义。
3. 掌握 `project()` 的 `VERSION`、`LANGUAGES`、`DESCRIPTION`、`HOMEPAGE_URL`。
4. 掌握 `add_executable()` 的核心用法和维护方式。
5. 完成头文件与源文件分离的多文件项目。
6. 看懂构建目录关键文件并快速排错。

## 正文内容

### 1. 从零创建项目（step-by-step）

先创建目录结构：

```text
hello-cmake/
├── CMakeLists.txt
└── src/
    └── main.cpp
```

这个结构简单但完整，足够演示 CMake 的核心流程。

#### 步骤 1：写 `main.cpp`

文件：`hello-cmake/src/main.cpp`

```cpp
#include <iostream> // 标准输出

int main() {
    std::cout << "你好，CMake！第一个项目运行成功。" << std::endl; // 验证输出
    return 0; // 正常结束
}
```

#### 步骤 2：写 `CMakeLists.txt`

文件：`hello-cmake/CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.20) # 最低版本与策略基线

project(
    HelloCMake
    VERSION 1.0.0
    DESCRIPTION "第一个 CMake 项目"
    HOMEPAGE_URL "https://example.local/hello-cmake"
    LANGUAGES CXX
)

add_executable(HelloCMake src/main.cpp)
```

#### 步骤 3：配置与构建

```bash
cmake -S . -B build
cmake --build build
```

解释：
- `-S .` 指源码目录。
- `-B build` 指构建目录。
- `cmake --build` 自动调用对应后端工具。

#### 步骤 4：运行程序

Linux / macOS：

```bash
./build/HelloCMake
```

Windows：

```powershell
.\build\HelloCMake.exe
```

如果是 Visual Studio 生成器，可执行文件常在 `build\Debug\`。

### 2. `cmake_minimum_required()` 版本策略详解

核心写法：

```cmake
cmake_minimum_required(VERSION 3.20)
```

它有两层作用：
1. **最低版本限制**：低于该版本直接报错，避免旧环境隐式兼容。
2. **策略基线设置**：让命令行为按较新的语义执行，减少历史行为差异。

为什么不建议写太低版本：
- 一些现代命令行为不可用。
- 团队成员之间更容易出现“同配置不同结果”。

常见建议：
- 教学和新项目：`3.20+`。
- 团队项目：本地和 CI 版本保持一致。
- 升级最低版本时，记录升级理由与影响范围。

### 3. `project()` 命令参数详解

推荐写法：

```cmake
project(
    DemoProject
    VERSION 2.3.1
    DESCRIPTION "演示 project 参数"
    HOMEPAGE_URL "https://example.local/demo-project"
    LANGUAGES C CXX
)
```

参数意义：
- `VERSION`：项目版本号，常用于发布、日志和打包。
- `LANGUAGES`：声明所需语言，避免无关探测。
- `DESCRIPTION`：项目说明，便于工具链和文档系统展示。
- `HOMEPAGE_URL`：项目主页地址，便于维护追踪。

读取自动变量示例：

```cmake
message(STATUS "项目名: ${PROJECT_NAME}")
message(STATUS "版本: ${PROJECT_VERSION}")
message(STATUS "主版本: ${PROJECT_VERSION_MAJOR}")
message(STATUS "次版本: ${PROJECT_VERSION_MINOR}")
message(STATUS "补丁版本: ${PROJECT_VERSION_PATCH}")
```

### 4. `add_executable()` 详解

基础语法：

```cmake
add_executable(目标名 源文件1 源文件2)
```

示例：

```cmake
add_executable(app src/main.cpp)
add_executable(tool src/tool_main.cpp src/tool_util.cpp)
```

理解重点：
1. 定义一个可执行目标。
2. 绑定编译输入文件。
3. 在生成阶段生成该目标对应规则。

维护建议：
- 小项目直接列文件最清晰。
- 文件增多后可以用 `target_sources()` 分组。

### 5. 多文件项目示例（头文件 + 源文件分离）

目录结构：

```text
hello-cmake/
├── CMakeLists.txt
├── include/
│   └── greeter.h
└── src/
    ├── greeter.cpp
    └── main.cpp
```

文件：`include/greeter.h`

```cpp
#ifndef GREETER_H
#define GREETER_H

#include <string>

std::string 获取问候语(); // 函数声明

#endif
```

文件：`src/greeter.cpp`

```cpp
#include "greeter.h"

std::string 获取问候语() {
    return "你好，来自多文件 CMake 项目！"; // 返回固定字符串
}
```

文件：`src/main.cpp`

```cpp
#include <iostream>
#include "greeter.h"

int main() {
    std::cout << 获取问候语() << std::endl; // 调用子模块
    return 0;
}
```

多文件版本 `CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.20)

project(
    HelloMultiFile
    VERSION 1.1.0
    DESCRIPTION "多文件 CMake 项目"
    HOMEPAGE_URL "https://example.local/hello-multi"
    LANGUAGES CXX
)

add_executable(HelloMultiFile
    src/main.cpp
    src/greeter.cpp
)

target_include_directories(HelloMultiFile
    PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include
)
```

### 6. 四平台构建命令对比

| 平台 | 推荐生成器 | 配置命令 | 构建命令 |
|---|---|---|---|
| Windows | Ninja / Visual Studio 17 2022 | `cmake -S . -B build -G Ninja` | `cmake --build build` |
| Linux | Ninja / Unix Makefiles | `cmake -S . -B build -G Ninja` | `cmake --build build` |
| macOS | Ninja / Xcode | `cmake -S . -B build -G Ninja` | `cmake --build build` |
| Android | Ninja + NDK 工具链 | `cmake -S . -B build-android -G Ninja -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-24` | `cmake --build build-android` |

补充：
- Android 常由 Gradle 调用 CMake，上述命令是等价底层流程。
- Visual Studio 是多配置生成器，产物通常带 `Debug/Release` 子目录。

### 7. 构建目录结构解释

第一次配置后，`build/` 目录常见文件如下：

```text
build/
├── CMakeCache.txt
├── CMakeFiles/
├── build.ninja
├── cmake_install.cmake
└── HelloCMake(.exe)
```

关键说明：
- `CMakeCache.txt`：缓存变量、编译器路径、探测结果。
- `CMakeFiles/`：内部中间文件目录。
- `build.ninja`：Ninja 后端规则文件。
- `cmake_install.cmake`：安装命令对应脚本。

排错建议：
- 切换生成器或编译器后，删除旧构建目录再配置。
- 路径问题优先检查 `CMakeCache.txt`。

### 8. 常见新手错误与解决方案

#### 错误 1：找不到 C++ 编译器

报错：`Could not find CMAKE_CXX_COMPILER`

解决：
- Windows：安装 Visual Studio 的 C++ 工作负载。
- Linux：安装 `build-essential` 或 `gcc g++`。
- macOS：执行 `xcode-select --install`。
- Android：检查 NDK 与工具链路径是否正确。

#### 错误 2：找不到 `CMakeLists.txt`

报错：`The source directory does not appear to contain CMakeLists.txt`

解决：
- 确认当前目录是项目根目录。
- 确认文件名大小写正确。
- 使用显式源码路径：`cmake -S <源码目录> -B build`。

#### 错误 3：头文件未找到

报错：`fatal error: greeter.h: No such file or directory`

解决：

```cmake
target_include_directories(HelloMultiFile PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include)
```

#### 错误 4：Windows 可以编译，Linux 失败

常见原因：文件名大小写不一致。

解决：
- `#include` 路径与实际文件名完全一致。
- 建议统一使用小写文件名。

## 动手实践

> 目标：创建一个“可输出版本号”的多文件项目。

### 实操步骤

1. 创建 `practice-cmake/`、`src/`、`include/`。
2. 在 `include/version.h` 声明 `获取版本字符串()`。
3. 在 `src/version.cpp` 返回 `"Practice 1.0.0"`。
4. 在 `src/main.cpp` 打印欢迎语与版本号。
5. 编写 `CMakeLists.txt`，包含：
   - `cmake_minimum_required(VERSION 3.20)`
   - `project(Practice VERSION 1.0.0 LANGUAGES CXX DESCRIPTION "实践项目" HOMEPAGE_URL "https://example.local/practice")`
   - `add_executable(Practice src/main.cpp src/version.cpp)`
   - `target_include_directories(Practice PRIVATE include)`
6. 执行 `cmake -S . -B build`。
7. 执行 `cmake --build build`。
8. 运行程序并核对输出。

参考输出：

```text
欢迎来到实践项目！
当前版本: Practice 1.0.0
```

## 本节小结

- 你已经掌握第一个 CMake 项目的完整闭环。
- 你已经理解最低版本与策略行为的关系。
- 你已经掌握 `project()` 核心参数与 `add_executable()` 用法。
- 你已经能搭建多文件项目并处理 include 路径。

## 练习题与答案

### 题目 1

为什么 `cmake_minimum_required(VERSION 3.20)` 不建议改成很低版本？

<details>
<summary>查看答案</summary>

版本过低会导致现代语义和新特性不可用，还可能触发旧策略行为，造成多人协作时构建结果不一致，排错成本更高。

</details>

### 题目 2

请写一个包含 `VERSION`、`LANGUAGES`、`DESCRIPTION`、`HOMEPAGE_URL` 的 `project()`。

<details>
<summary>查看答案</summary>

```cmake
project(
    MyApp
    VERSION 0.1.0
    DESCRIPTION "我的规范 CMake 项目"
    HOMEPAGE_URL "https://example.local/myapp"
    LANGUAGES CXX
)
```

</details>

### 题目 3

为什么推荐 `cmake -S . -B build` 而不是 `cmake .`？

<details>
<summary>查看答案</summary>

因为 `-S/-B` 对应源外构建：源码目录不被污染、可维护多个构建目录、清理构建状态更简单，适合长期项目维护。

</details>
## 下一步
→ [01-变量与缓存](../02-CMake语法与命令/01-变量与缓存.md)
