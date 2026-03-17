# Ninja 简介与安装

## 元数据块

> **所属模块：** P01-现代 CMake 与构建工具链  
> **章节位置：** 第 5 章第 1 节（Ninja 构建后端）  
> **前置知识：** 第 4 章全部（CMake 基础、变量、目标、CMakePresets）  
> **适用平台：** Windows / Linux / macOS / CI 环境  
> **预计阅读时间：** 12 分钟  
> **本节产出：** 完成 Ninja 安装、验证、常用命令实操并理解 KrKr2 默认选择原因

## 本节目标

读完本节后，你将能够：

1. 解释 Ninja 的诞生背景与 Chromium 项目关系。
2. 对比 Ninja 与 Make 在启动时间上的差异。
3. 对比 Ninja 与 Make 在增量构建上的差异。
4. 对比 Ninja 与 Make 在并行化上的差异。
5. 理解 Ninja 的设计哲学与使用边界。
6. 在 Windows、Linux、macOS、CI 环境安装 Ninja。
7. 使用完整命令链验证 Ninja 安装结果。
8. 理解 `build.ninja` 的 `rule`、`build`、`default`。
9. 使用 `ninja -j`、`ninja -v`、`ninja -t targets`、`ninja -t deps`、`ninja -t clean`。
10. 说明 KrKr2 项目为什么默认使用 Ninja。

## 什么是 Ninja

Ninja 是一个以执行速度为核心目标的构建工具。
它通常与 CMake 等生成器配合使用，而不是独立承担复杂工程表达。

典型流程是：

1. 使用 `CMakeLists.txt` 定义工程。
2. CMake 生成 `build.ninja`。
3. Ninja 读取并执行构建命令。

这个分工中：

- CMake 负责描述和生成。
- Ninja 负责执行和加速。

## Ninja 的诞生背景

Ninja 起源于 Google 工程实践。
其核心动机是解决 Chromium 超大工程中的构建效率问题。

Chromium 场景具有典型特征：

1. 文件和目标规模很大。
2. 增量构建频率很高。
3. 需要更高效利用多核并行。

Ninja 通过极简输入和低开销执行路径，降低启动和调度成本。
因此它不是“功能最多”，而是“执行最快”类型的构建后端。

## Ninja vs Make：启动、增量、并行

| 维度 | Make | Ninja | 影响 |
| :--- | :--- | :--- | :--- |
| 启动时间 | 解析通常更重 | 启动更轻 | 小改动反馈更快 |
| 增量构建 | 大项目检查成本上升 | 判定路径更短 | 迭代更顺畅 |
| 并行化 | 常需更多手动调优 | 调度开销低 | 多核利用更高 |

## Ninja 的设计哲学

Ninja 的核心哲学是：

**复杂逻辑放在生成阶段，执行阶段只做快速调度。**

因此：

1. 不建议手写复杂 `build.ninja`。
2. 建议通过 CMake 生成 Ninja 文件。
3. 条件分支与平台逻辑应放在 CMake 阶段。

## 四平台安装方式

### Windows（winget / scoop / 手动）

#### winget

```bash
winget install Ninja-build.Ninja
```

#### scoop

```bash
scoop install ninja
```

#### 手动安装

1. 访问 `https://github.com/ninja-build/ninja/releases`。
2. 下载 `ninja-win.zip`。
3. 解压得到 `ninja.exe`。
4. 把目录加入系统 `PATH`。
5. 重新打开终端执行 `ninja --version`。

### Linux（apt / dnf / pacman / 源码）

#### Debian / Ubuntu / Linux Mint

```bash
sudo apt update
sudo apt install -y ninja-build
```

#### Fedora / RHEL / CentOS Stream

```bash
sudo dnf install -y ninja-build
```

#### Arch / Manjaro

```bash
sudo pacman -S --noconfirm ninja
```

#### 源码安装

```bash
git clone https://github.com/ninja-build/ninja.git
cd ninja
python3 configure.py --bootstrap
sudo install -m 755 ninja /usr/local/bin/ninja
```

### macOS（Homebrew）

```bash
brew update
brew install ninja
```

### CI 环境（GitHub Actions / GitLab CI / Jenkins）

#### GitHub Actions

```yaml
- name: Install Ninja
  run: |
    sudo apt update
    sudo apt install -y ninja-build
    ninja --version
```

#### GitLab CI

```yaml
before_script:
  - apt-get update
  - apt-get install -y ninja-build cmake
  - ninja --version
```

#### Jenkins Linux Agent

```bash
sudo apt update
sudo apt install -y ninja-build
ninja --version
```

### Android 补充

Android NDK 常包含 Ninja，默认 Android Studio 流程中通常无需单独安装。

## 验证安装：完整命令链

### Windows PowerShell

```powershell
ninja --version
Get-Command ninja
cmake --version
cmake -G Ninja -S . -B build\verify-ninja
cmake --build build\verify-ninja -v
```

### Linux / macOS / CI Shell

```bash
ninja --version
command -v ninja
cmake --version
cmake -G Ninja -S . -B build/verify-ninja
cmake --build build/verify-ninja -v
```

通过标准：

1. 能输出 Ninja 版本号。
2. 能定位 Ninja 可执行路径。
3. CMake 能生成 Ninja 构建目录。
4. Ninja 能完成真实构建。

## `build.ninja` 文件结构简介

```ninja
cxx = g++
cflags = -Wall -Wextra -O2

rule compile
  command = $cxx $cflags -MMD -MF $out.d -c $in -o $out
  depfile = $out.d
  description = CXX $out

rule link
  command = $cxx $in -o $out
  description = LINK $out

build main.o: compile main.cpp
build app: link main.o

default app
```

### `rule`

定义命令模板。

### `build`

定义输入输出依赖关系。

### `default`

定义默认构建目标。

## Ninja 常用命令

### `ninja -j`

```bash
ninja -j 8
```

### `ninja -v`

```bash
ninja -v
```

### `ninja -t targets`

```bash
ninja -t targets
```

### `ninja -t deps`

```bash
ninja -t deps
```

### `ninja -t clean`

```bash
ninja -t clean
```

## KrKr2 项目为什么默认选择 Ninja

KrKr2 构建系统的核心诉求是：

1. 跨平台行为一致。
2. 增量构建反馈快。

Ninja 在这两点上表现稳定：

1. Windows、Linux、macOS、CI 命令入口统一。
2. 小改动迭代速度快。
3. 本地与 CI 差异更小。
4. 与 CMakePresets 配合自然。

示例：

```json
{
  "version": 3,
  "configurePresets": [
    {
      "name": "windows-base",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/${presetName}"
    }
  ]
}
```

## 动手实践

### 步骤 1：创建目录结构

```text
ninja-demo/
├── CMakeLists.txt
└── main.cpp
```

### 步骤 2：编写 `main.cpp`

```cpp
#include <iostream>

int main() {
    std::cout << "Hello Ninja Tutorial" << std::endl;
    return 0;
}
```

### 步骤 3：编写 `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.20)
project(ninja_demo LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
add_executable(ninja_demo main.cpp)
```

### 步骤 4：配置并构建

```bash
cmake -G Ninja -S . -B build
cmake --build build -v
```

### 步骤 5：运行程序

Windows：

```powershell
.\build\ninja_demo.exe
```

Linux/macOS：

```bash
./build/ninja_demo
```

### 步骤 6：观察增量构建

1. 修改输出文本。
2. 重新执行 `cmake --build build -v`。
3. 观察仅受影响文件重编译。

## 本节小结

- Ninja 来自 Chromium 大型工程的效率需求。
- 相比 Make，Ninja 在启动、增量、并行上更偏重执行效率。
- Ninja 更适合作为后端执行器，而非复杂脚本前端。
- 你已掌握四平台安装和完整验证命令链。
- 你已掌握 `build.ninja` 结构与常用命令。
- 你已理解 KrKr2 默认选择 Ninja 的工程化原因。

## 练习题与答案

### 题目 1：为什么 Ninja 与 Chromium 场景高度匹配？

<details>
<summary>查看答案</summary>

Chromium 工程规模大、增量构建频繁、并行需求高。Ninja 通过低开销调度和简洁执行路径降低等待时间，因此更匹配这类场景。

</details>

### 题目 2：`build.ninja` 中 `rule`、`build`、`default` 的作用分别是什么？

<details>
<summary>查看答案</summary>

`rule` 定义命令模板，`build` 定义输入输出依赖关系，`default` 定义默认构建目标。

</details>

### 题目 3：请写出 Linux 下安装并验证 Ninja 的完整命令链。

<details>
<summary>查看答案</summary>

```bash
sudo apt update
sudo apt install -y ninja-build cmake
command -v ninja
ninja --version
cmake -G Ninja -S . -B build/verify-ninja
cmake --build build/verify-ninja -v
```

</details>

## 下一步

继续阅读 [02-CMake 搭配 Ninja](./02-CMake搭配Ninja.md)，学习如何在 CMakePresets 中组织 Ninja 的多平台构建流程。
