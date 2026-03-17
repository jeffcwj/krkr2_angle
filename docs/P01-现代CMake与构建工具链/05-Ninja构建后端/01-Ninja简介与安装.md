# Ninja 简介与安装

> **所属模块：** P01-现代 CMake 与构建工具链
> **前置知识：** 第 4 章全部（CMake 基础、变量、目标、CMakePresets）
> **预计阅读时间：** 15 分钟

## 本节目标
读完本节后，你将能够：
1. 理解 Ninja 的定位以及它与 CMake 的关系。
2. 掌握 Ninja 相比于传统 Make 的核心优势。
3. 在 Windows、Linux、macOS、Android 平台上完成 Ninja 的安装与验证。
4. 了解 KrKr2 项目选择 Ninja 作为统一构建后端的原因。

## 什么是 Ninja
Ninja 是由 Google 工程师开发的轻量级构建工具（Build System），它的口号是“专注于速度”。在 C++ 项目开发中，它通常扮演“构建后端”（Build Backend）的角色。

我们可以用建筑施工来打个比方：
* **CMake 是“建筑设计师”：** 它负责绘制蓝图。你通过编写 `CMakeLists.txt` 来告诉它有哪些源文件、库依赖、编译选项。CMake 会根据你的环境（Windows 还是 Linux），把蓝图翻译成“施工方案”。
* **Ninja 是“施工队”：** 它不关心如何设计，只负责盯着蓝图，以最快的速度把砖头（源文件）垒成大楼（可执行文件）。

Ninja 的设计初衷是为了解决大型项目（如 Chromium 或 LLVM）中构建速度缓慢的问题。它只做一件事：高效地执行构建命令，并跟踪文件依赖关系。

## Ninja vs Make
在 Unix 系平台上，Make 是历史悠久的构建工具。虽然 CMake 默认在 Linux 上生成 Makefile，但在现代开发中，Ninja 已经逐渐取代了 Make 的地位。

| 特性 | Make | Ninja |
| :--- | :--- | :--- |
| **设计目标** | 通用、功能丰富（支持复杂逻辑） | 简单、极速（专为机器生成设计） |
| **启动速度** | 慢（需要解析复杂的 Makefile） | 极快（几乎瞬间启动） |
| **增量构建** | 随项目规模变大而变慢 | 即使是万级文件，检查更新也只需零点几秒 |
| **并行处理** | 需要手动指定 `-j` 参数 | 默认根据 CPU 核心数自动并行构建 |
| **平台支持** | 主要针对 Unix/POSIX | 跨平台一致性极佳（Win/Linux/Mac） |

Ninja 为什么比 Make 快？
1. **最小化 I/O：** Ninja 在构建开始前会构建一个极其精简的依赖图，减少磁盘读取次数。
2. **零逻辑：** Ninja 不支持条件判断或复杂的脚本逻辑。这些复杂的“脑力活”都交给了 CMake 处理。Ninja 只需无脑执行命令。
3. **高效的依赖检查：** Ninja 使用极快的时间戳对比算法来决定哪些文件需要重编。

## Ninja vs MSBuild vs Xcode
如果你在 Windows 上使用 Visual Studio，默认会使用 MSBuild；在 macOS 上使用 Xcode，默认会使用 Xcode 构建系统。

虽然这些原生系统对特定 IDE 支持很好，但在跨平台开发（如 KrKr2）中，它们存在以下问题：
* **行为不统一：** 同样的工程在 Windows 和 Mac 上的构建逻辑可能因为构建器的差异而产生细微不同。
* **速度劣势：** 对于大型 C++ 项目，Ninja 的调度性能通常优于 MSBuild。
* **CI 环境统一：** 在自动化构建（CI/CD）服务器上，统一使用 Ninja 可以让脚本更加简洁且易于维护。

## 四平台安装指南

### 1. Windows 平台
在 Windows 上安装 Ninja 有多种快捷方式：

* **方式 A: winget (推荐)**
  ```bash
  winget install Ninja-build.Ninja
  ```
* **方式 B: scoop**
  ```bash
  scoop install ninja
  ```
* **方式 C: Chocolatey**
  ```bash
  choco install ninja
  ```
* **方式 D: 手动安装**
  1. 访问 [Ninja GitHub Releases](https://github.com/ninja-build/ninja/releases)。
  2. 下载 `ninja-win.zip`，解压得到 `ninja.exe`。
  3. 将 `ninja.exe` 所在的文件夹路径添加到系统的 `PATH` 环境变量中。

### 2. Linux 平台
根据你的发行版选择相应的包管理器：

* **Ubuntu / Debian / Linux Mint:**
  ```bash
  sudo apt update
  sudo apt install ninja-build
  ```
* **Fedora / RHEL / CentOS:**
  ```bash
  sudo dnf install ninja-build
  ```
* **Arch Linux:**
  ```bash
  sudo pacman -S ninja
  ```

### 3. macOS 平台
推荐使用 Homebrew 安装：
```bash
brew install ninja
```

### 4. Android 平台
如果你在进行移动端开发，通常不需要手动安装 Ninja。Android NDK（Native Development Kit）内置了 Ninja 副本。当你使用 Android Studio 或 CMake 交叉编译时，它会自动调用。

### 5. 验证安装
在任意终端（Command Prompt, PowerShell, Bash）中输入：
```bash
ninja --version
```
如果你看到类似 `1.11.1` 的版本号，说明安装成功。

## Ninja 文件格式简介
虽然我们不需要手动编写 Ninja 脚本（因为 CMake 会为我们代劳），但了解它的基本结构能帮你更好地调试。

Ninja 的主配置文件通常叫作 `build.ninja`。它的语法非常直接：

```ninja
# 定义一个变量
cxx = g++
cflags = -Wall -O2

# 定义一个构建规则（Rule）
rule compile
  command = $cxx $cflags -c $in -o $out
  description = Compiling $in...

# 定义一个构建目标（Build Statement）
build main.o: compile main.cpp

# 定义最终生成物
rule link
  command = $cxx $in -o $out

build my_app: link main.o
```

这段简单的代码展示了 Ninja 的三大核心概念：
1. **Variables（变量）：** 用来存储通用的设置。
2. **Rules（规则）：** 告诉 Ninja 如何执行具体的命令。
3. **Build Statements（构建声明）：** 明确具体的输入文件和输出文件。

因为这种格式极其扁平，Ninja 能够以闪电般的速度加载它，这正是它快过 Make 的秘诀之一。

## 为什么 KrKr2 选择 Ninja
在 KrKr2 项目中，我们坚持使用 Ninja 作为核心构建后端。原因如下：

1. **跨平台体验一致：** 无论你在 Windows、Linux 还是 Mac 上，输入同样的 `ninja` 命令都能得到一致的结果。
2. **极速的增量构建：** 在开发过程中，我们经常只修改一个源文件。Ninja 可以在毫秒级判断出只需重编这一个文件，让反馈周期大幅缩短。
3. **原生支持 CMakePresets：** 我们所有的预设文件 `CMakePresets.json` 都默认配置了 `"generator": "Ninja"`。

你可以查看 KrKr2 的根目录下的 `CMakePresets.json`，你会看到类似这样的代码：

```json
{
  "version": 3,
  "configurePresets": [
    {
      "name": "windows-base",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/${presetName}",
      "condition": {
        "type": "equals",
        "lhs": "${hostSystemName}",
        "rhs": "Windows"
      }
    }
  ]
}
```

这意味着你不再需要纠结到底是用 Visual Studio 2022 还是 Makefile。在 KrKr2 的世界里，Ninja 就是我们的官方语言。

## 本节小结
本节中，你了解了 Ninja 作为一个高速构建工具的定位，它与 CMake 的“设计者-施工者”关系，以及它在性能上对传统工具的超越。你已经在各个平台上成功安装了它。

## 练习题与答案

### 题目 1：Ninja 与 CMake 之间是什么关系？
<details><summary>查看答案</summary>
CMake 是构建生成器（Meta-Build System），负责根据 `CMakeLists.txt` 生成构建脚本。Ninja 是构建执行器（Build System），负责解析构建脚本并调用编译器（如 cl.exe, g++, clang++）完成实际的构建任务。
</details>

### 题目 2：在 Windows 上，如果你手动下载了 `ninja.exe` 但在终端运行 `ninja --version` 提示命令找不到，最可能的原因是什么？
<details><summary>查看答案</summary>
原因是你没有将 `ninja.exe` 所在的文件夹路径添加到系统的 `PATH` 环境变量中。只有添加后，系统才能在任何目录下识别出 `ninja` 这个命令。
</details>

### 题目 3：相比于 Make，Ninja 为什么在大型项目中表现更出色？
<details><summary>查看答案</summary>
Ninja 采用了更简单的依赖跟踪逻辑，去除了复杂的脚本语法，最小化了磁盘 I/O。它能更高效地利用多核 CPU 进行并行构建，且其增量构建的时间复杂度更低，响应更快。
</details>

## 下一步
→ 继续阅读 [02-CMake 搭配 Ninja](./02-CMake搭配Ninja.md)

