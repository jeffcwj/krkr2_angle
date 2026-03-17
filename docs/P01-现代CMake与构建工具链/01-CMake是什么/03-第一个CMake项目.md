# 03-第一个 CMake 项目

> **所属模块：** P01-现代CMake与构建工具链
> **前置知识：** [02-CMake核心概念](./02-CMake核心概念.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 在 Windows/Linux/macOS/Android 平台上安装并配置 CMake 环境
2. 编写并运行一个最简单的 CMake 项目
3. 学会处理多源文件项目及解决常见环境错误

## 安装 CMake (四平台指南)

### 1. Windows 平台
- **方法 A (官网安装)**: 前往 [cmake.org/download/](https://cmake.org/download/) 下载 `.msi` 安装包。安装时务必勾选 "Add CMake to the system PATH for all users"。
- **方法 B (包管理器)**: 
  - `winget install kitware.cmake`
  - `choco install cmake` (如果你安装了 Chocolatey)
  - `scoop install cmake` (如果你安装了 Scoop)

### 2. Linux 平台
- **Ubuntu/Debian**: `sudo apt update && sudo apt install cmake`
- **Fedora**: `sudo dnf install cmake`
- **Arch Linux**: `sudo pacman -S cmake`
- **如果版本太旧**: 某些旧版系统（如 Ubuntu 20.04）自带的 CMake 版本过低，无法支持现代 CMake 功能。你可以通过 [Snap](https://snapcraft.io/cmake) 安装最新版：`sudo snap install cmake --classic`。

### 3. macOS 平台
- **Homebrew**: `brew install cmake`
- **官网**: 下载 `.dmg` 文件并安装。

### 4. Android 平台
Android 开发者通常不需要手动安装独立的 CMake。
- 打开 **Android Studio** -> Settings (Preferences) -> Languages & Frameworks -> Android SDK -> SDK Tools。
- 勾选 **CMake** (推荐版本 3.22+) 和 **NDK** 进行安装。

### 验证安装
在终端/命令行（Windows 建议使用 PowerShell 或 CMD）输入：
```bash
cmake --version
```
如果能看到类似 `cmake version 3.28.3` 的输出，说明你已经成功安装！

## 创建第一个项目结构

在一个新目录下手动创建以下两个文件：

### 1. main.cpp
```cpp
#include <iostream>

int main() {
    std::cout << "Hello, CMake! Welcome to KrKr2." << std::endl;
    return 0;
}
```

### 2. CMakeLists.txt (入口文件)
```cmake
# 声明 CMake 的最低版本要求 (必须在第一行)
cmake_minimum_required(VERSION 3.10)

# 定义项目名称
project(HelloWorld)

# 添加一个可执行文件目标
# 参数 1: 目标名称 (HelloWorld)
# 参数 2: 包含的源文件列表 (main.cpp)
add_executable(HelloWorld main.cpp)

## 配置与构建

我们现在采用"源外构建"的方式来运行这个项目。

### 第一步：配置 (Configure)
打开终端，进入你的项目文件夹，运行以下命令：
```bash
cmake -B build
```
- `-B build`：告诉 CMake 将所有的中间文件和构建产物放在一个名为 `build` 的文件夹中（如果该文件夹不存在，它会自动创建）。
- 如果你想在 Windows 上使用 **Ninja** 生成器（推荐），可以运行：`cmake -B build -G Ninja`。

### 第二步：构建 (Build)
配置完成后，运行以下命令进行最终编译：
```bash
cmake --build build
```
- `--build build`：这是一个跨平台命令，它会自动调用适合你系统的底层构建工具（如 Makefile、Ninja 或 Visual Studio）。

## 运行程序

构建完成后，你会发现在 `build` 目录下生成了一个可执行文件。

### 1. Windows (CMD/PowerShell)
```powershell
.\build\HelloWorld.exe
# 或者如果你使用的是 VS 生成器，路径可能更深：
.\build\Debug\HelloWorld.exe
```

### 2. Linux / macOS
```bash
./build/HelloWorld
```

### 3. Android
Android 应用通常是通过 Gradle 自动调用 CMake 构建成共享库（`.so`）并打包进 `.apk` 中运行的。在后续教程中，我们会专门讲解如何为 KrKr2 开发 Android 版本的 C++ 代码。

## 添加第二个源文件

在实际项目中，我们会有很多源文件。现在我们来添加一个辅助工具文件：

### 1. 创建 utils.h
```cpp
#ifndef UTILS_H
#define UTILS_H

void sayHello();

#endif
```

### 2. 创建 utils.cpp
```cpp
#include <iostream>
#include "utils.h"

void sayHello() {
    std::cout << "This message is from utils.cpp!" << std::endl;
}
```

### 3. 修改 main.cpp
```cpp
#include <iostream>
#include "utils.h"

int main() {
    std::cout << "Hello, CMake!" << std::endl;
    sayHello(); // 调用 utils 里的函数
    return 0;
}
```

### 4. 更新 CMakeLists.txt
将新的源文件添加到 `add_executable` 目标中：
```cmake
cmake_minimum_required(VERSION 3.10)
project(HelloWorld)

# 将 utils.cpp 也加入构建
add_executable(HelloWorld main.cpp utils.cpp)
```

现在重新运行 `cmake --build build`，CMake 会自动增量编译新添加的文件。

## 常见错误排查

### 1. CMake Error: Could not find CMAKE_CXX_COMPILER
**原因：** 你的系统中没有安装 C++ 编译器，或者编译器不在环境变量中。
**解决方法：**
- **Windows**: 安装 Visual Studio（勾选 "C++ 桌面开发" 工作负载）。
- **Linux**: 安装 `build-essential` 包 (`sudo apt install build-essential`)。
- **macOS**: 安装 Xcode 命令行工具 (`xcode-select --install`)。

### 2. CMake Error: The source directory does not appear to contain CMakeLists.txt
**原因：** 你运行命令的路径不正确，或者 `CMakeLists.txt` 文件名拼写错误。
**解决方法：** 确认你正在项目根目录下运行命令，且文件名完全匹配（包括大小写）。

## 本节小结

- 安装 CMake 是迈向现代 C++ 开发的第一步。
- 典型的 CMake 构建流程是 `cmake -B build` + `cmake --build build`。
- 多源文件项目只需在 `add_executable` 中列出所有参与编译的 `.cpp` 文件。

## 练习题与答案

### 题目 1：在配置阶段，`-B build` 命令中的 `-B` 代表什么意思？

<details>
<summary>查看答案</summary>

`-B` 代表 "Binary directory"（二进制目录），也就是构建目录。它指定了 CMake 应该在哪里生成构建系统文件（如 Makefile 或 Ninja 文件）以及最终的可执行文件。

</details>

### 题目 2：为什么我们在 CMakeLists.txt 中添加新源文件（如 utils.cpp）后，只需要重新运行 `cmake --build build` 而通常不需要重新运行 `cmake -B build`？

<details>
<summary>查看答案</summary>

因为 CMake 生成的底层构建工具（如 Makefile 或 Ninja）具有自动触发重新配置的功能。当你运行构建命令时，构建工具会发现 `CMakeLists.txt` 已被修改，它会自动先运行 CMake 配置过程，然后再进行编译。

</details>

### 题目 3：请写出一个跨平台的 CMake 构建命令（不依赖于具体生成器）。

<details>
<summary>查看答案</summary>

`cmake --build <构建目录名>`，例如：`cmake --build build`。

</details>

## 下一步

→ 恭喜！你已经完成了第一章的学习。接下来我们将深入 [P02-核心模块剖析] 了解 KrKr2 的内部构造。

```
