# Windows 环境搭建

本章节将引导你在 Windows 10 或 Windows 11 (x64) 系统上搭建 KR2 项目的开发环境。我们将安装编译器、构建工具以及包管理器。

## 1. 前提条件

在开始之前，请确保你的系统满足以下要求：
- 操作系统：Windows 10 或 Windows 11 (64位)
- 硬盘空间：至少预留 20GB 空间（Visual Studio 和 vcpkg 比较占用空间）
- 网络连接：需要稳定的互联网连接以执行在线安装和下载依赖。

## 2. 安装 Visual Studio 2022

Visual Studio 是 Windows 平台首选的 C++ 开发环境。KR2 项目推荐使用 Visual Studio 2022。

1.  访问 [Visual Studio 官网](https://visualstudio.microsoft.com/zh-hans/downloads/)。
2.  下载 **Community** 版本（对个人开发者免费）。
3.  运行安装程序。在“工作负载”选项卡中，勾选 **“使用 C++ 的桌面开发” (Desktop development with C++)**。
4.  在右侧的“安装详细信息”窗格中，确保选中以下关键组件：
    - **MSVC v143 - VS 2022 C++ x64/x86 生成工具** (最新版本)
    - **Windows 11 SDK (10.0.xxxxx.0)** 或 Windows 10 SDK
    - **C++ CMake 工具 for Windows**
5.  点击“安装”，等待安装完成。

## 3. 安装 CMake 3.31.1+

CMake 是一个跨平台的构建系统生成器，KR2 使用 CMake 管理工程。

- **方法 A（官网下载）**：访问 [CMake 官网下载页面](https://cmake.org/download/)，下载 Windows x64 Installer (.msi)。安装时，建议选择“Add CMake to the system PATH for all users”。
- **方法 B（命令行）**：打开 PowerShell，运行：
    ```powershell
    winget install kitware.cmake
    ```

安装完成后，打开命令行输入 `cmake --version` 检查版本，确保版本大于等于 3.31.1。

## 9. 环境变量总结表格

为了方便检查，以下是 KR2 相关的关键环境变量总结：

| 变量名 | 推荐设置方式 | 说明 |
| :--- | :--- | :--- |
| `PATH` | 添加 VS 安装目录、vcpkg、CMake、Ninja、winflexbison、Python、NASM 所在的目录 | 系统路径变量，用于全局执行命令 |
| `VCPKG_ROOT` | 在“高级系统设置”中手动添加。路径格式示例：`D:/vcpkg` | vcpkg 的安装根目录 |

## 10. 验证工具安装

在完成以上所有步骤后，请务必执行以下命令进行检查：

```powershell
# 检查 CMake 版本
cmake --version

# 检查 Ninja 版本
ninja --version

# 检查 vcpkg
vcpkg version

# 检查 win_flex 和 win_bison
win_flex --version
win_bison --version

# 检查 Python
python --version

# 检查 NASM
nasm --version
```

如果所有命令都能输出正确的版本号，说明你的 Windows 开发环境已经搭建完毕。

## 11. 练习题与答案

以下是关于 Windows 环境搭建的几个小问题：

1.  在 Windows 下配置 `VCPKG_ROOT` 环境变量时，最需要注意的路径格式问题是什么？
2.  如果在命令行输入 `cmake --version` 提示“不是内部或外部命令”，你应该如何排查？
3.  为什么 KR2 项目需要安装 winflexbison？

<details>
<summary>查看答案</summary>

1.  **回答**：在 Windows 上，`VCPKG_ROOT` 的路径分隔符必须使用正斜杠 `/` 或双反斜杠 `\\`（例如 `D:/vcpkg`），而不能直接使用单反斜杠 `\`（例如 `D:\vcpkg`），否则 CMake 可能无法正确处理该路径。
2.  **回答**：
    - 首先确认 CMake 是否已安装。
    - 其次确认 CMake 的 `bin` 目录（例如 `C:\Program Files\CMake\bin`）是否已添加到系统的 **PATH** 环境变量中。
    - 如果是刚修改了环境变量，请关闭并重新打开命令行终端，让新配置生效。
3.  **回答**：KR2 项目中包含了一些需要进行语法分析或词法分析的代码（通常是自定义的脚本引擎或解析器），这些代码生成需要依赖 `flex` 和 `bison` 工具。而在 Windows 上，winflexbison 是这两个工具的推荐移植版本。

</details>


## 5. 安装 vcpkg

vcpkg 是 Microsoft 开发的 C++ 库管理器。在 KR2 中，绝大部分外部依赖（如 glib、libpng 等）都是通过 vcpkg 自动下载并编译的。

1.  **克隆 vcpkg 仓库**：
    打开 PowerShell 或 Git Bash，切换到你想放置 vcpkg 的目录（例如 `D:\dev`），运行：
    ```bash
    git clone https://github.com/microsoft/vcpkg.git
    cd vcpkg
    ```
2.  **引导 vcpkg (Bootstrap)**：
    运行 vcpkg 目录下的引导脚本：
    ```powershell
    .\bootstrap-vcpkg.bat
    ```
3.  **配置 VCPKG_ROOT 环境变量**：
    - 右键“此电脑” -> “属性” -> “高级系统设置” -> “环境变量”。
    - 在“用户变量”中新建变量名 `VCPKG_ROOT`，值为你的 vcpkg 根目录路径。
    - **特别注意**：在 Windows 上设置 `VCPKG_ROOT` 时，路径必须使用 `/` 或 `\\` 作为分隔符，不能直接使用单 `\`。
        - ❌ 错误示例：`D:\vcpkg` (CMake 可能无法正确转义路径)
        - ✅ 正确示例：`D:/vcpkg` 或 `D:\\vcpkg`
    - 点击“确定”并重启所有已打开的命令行终端。

## 6. 安装 winflexbison 2.5.25

winflexbison 是词法分析器和语法分析生成器的 Windows 移植版本。

1.  从 [GitHub Releases](https://github.com/lexxmark/winflexbison/releases/tag/v2.5.25) 下载 `win_flex_bison-2.5.25.zip`。
2.  将其解压到一个固定位置，例如 `C:\tools\winflexbison`。
3.  将该目录添加到你的系统 **PATH** 环境变量中。

## 7. 安装 Python3

KR2 的构建脚本和一些构建工具依赖 Python。

- 访问 [Python 官网](https://www.python.org/downloads/) 下载最新的 Python 3.x 安装器。
- 安装时务必勾选 **“Add Python to PATH”**。
- 你也可以通过 Microsoft Store 直接安装 Python。

## 8. 安装 NASM

NASM 是一个 80x86 汇编器，用于编译某些高性能计算库的汇编代码。

1.  访问 [NASM 官网](https://www.nasm.us/) 下载 Windows 安装程序。
2.  运行安装程序并完成安装。
3.  找到 NASM 的安装路径（通常是 `C:\Users\用户名\AppData\Local\bin\NASM`），将该路径添加到 **PATH**。


## 4. 安装 Ninja

Ninja 是一个小巧、极速的构建工具，常与 CMake 搭配使用。

- **获取途径**：如果你安装了 Visual Studio 的 CMake 支持，Ninja 通常已经包含在内。
- **手动安装**：可以从 [Ninja GitHub Releases](https://github.com/ninja-build/ninja/releases) 下载 `ninja-win.zip`，解压后将 `ninja.exe` 所在目录添加到系统的 PATH 环境变量中。
- **验证**：在命令行输入 `ninja --version`。
