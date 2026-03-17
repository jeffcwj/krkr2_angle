# Windows 环境搭建

> **所属模块：** M01-项目导览与环境搭建  
> **前置知识：** [01-目录结构与模块职责](../02-项目架构总览/01-目录结构与模块职责.md)  
> **预计阅读时间：** 45 分钟  
> **适用系统：** Windows 10 / Windows 11 x64  
> **最后更新：** 2026-03-17

## 本节目标

读完本节后，你将能够：

1. 安装 Visual Studio 2022 的必要工作负载和组件。
2. 验证 MSVC 编译器满足 C++17 要求。
3. 在 Windows 上配置 CMake、Ninja、vcpkg、Git、winflexbison。
4. 正确设置 `VCPKG_ROOT` 与 `PATH`。
5. 用项目 preset 完成首次 configure 与 build。

## 1. 先核对项目真实配置

开始安装前，先看仓库配置，避免装错版本：

- `CMakePresets.json` 的 Windows preset 使用 `Ninja`。
- Windows triplet 固定为 `x64-windows-static-md`。
- 根 `CMakeLists.txt` 要求 `cmake_minimum_required(VERSION 3.28)`。
- 根 `CMakeLists.txt` 设置 `set(CMAKE_CXX_STANDARD 17)`。
- 非 Android 分支会读取 `$ENV{VCPKG_ROOT}`。
- `cpp/core/tjs2/CMakeLists.txt` 会查找 `win_bison.exe`。

结论：

1. Ninja 不是可选项。
2. `VCPKG_ROOT` 必须配置正确。
3. `winflexbison` 必须可在 PATH 中找到。

## 2. 前提条件

- 系统：Windows 10 22H2 或 Windows 11 23H2 及以上。
- 架构：x64。
- 磁盘：建议预留 30GB。
- 网络：可访问 GitHub 与常见源码镜像站。
- 权限：安装阶段建议管理员权限。

## 3. Visual Studio 2022 安装详解

Visual Studio 在本项目里不只是 IDE，
它还是 MSVC 与 SDK 的官方来源。

### 3.1 下载

1. 打开：https://visualstudio.microsoft.com/zh-hans/downloads/
2. 下载 **Visual Studio 2022 Community**。
3. 运行安装器 `VisualStudioSetup.exe`。

### 3.2 工作负载选择

在“工作负载”页至少勾选：

- **使用 C++ 的桌面开发**。

建议额外勾选：

- **使用 C++ 的游戏开发**（可选，但有助于后续图形调试）。

### 3.3 组件选择（非常关键）

切到“单个组件”，确认以下条目存在：

1. `MSVC v143 - VS 2022 C++ x64/x86 生成工具`。
2. `Windows 11 SDK`（或最新 Windows 10 SDK）。
3. `C++ CMake tools for Windows`。
4. `MSBuild`。

可选增强：

1. AddressSanitizer。
2. 调试工具（Windows）。

### 3.4 像看截图一样检查安装页面

安装页面通常分为：

- 左侧：工作负载列表。
- 右侧：安装详细信息。
- 底部：安装位置、下载大小、安装后占用。

请重点确认：

1. 右侧有 `MSVC v143`。
2. 右侧有 `Windows SDK`。
3. 底部可用空间充足。

### 3.5 安装后验证

打开 **x64 Native Tools Command Prompt for VS 2022**，执行：

```bat
cl
```

输出包含 `Microsoft (R) C/C++ Optimizing Compiler` 即为正常。

## 4. MSVC 版本要求（C++17）

项目根 `CMakeLists.txt` 设置了：

```cmake
set(CMAKE_CXX_STANDARD 17)
```

推荐基线：

- VS 2022 17.x
- 工具集 v143

在终端执行 `cl` 可以查看版本信息。
常见 `19.3x.xxxxx` 已可稳定支持 C++17。

## 5. CMake 安装方式对比

本项目最低需要 CMake 3.28。

### 5.1 方式 A：VS 自带 CMake

优点：安装一步到位。
缺点：在外部终端的 PATH 可见性可能不一致。

### 5.2 方式 B：官网独立安装（推荐）

1. 访问 https://cmake.org/download/
2. 下载 Windows x64 MSI。
3. 安装时勾选“Add CMake to the system PATH for all users”。

### 5.3 方式 C：包管理器

```powershell
winget install Kitware.CMake
```

或：

```powershell
scoop install cmake
```

### 5.4 验证

```powershell
cmake --version
```

## 6. Ninja 安装与配置

Windows preset 使用 Ninja，因此必须安装。

安装方式：

1. 使用 VS 自带 Ninja。
2. 下载 `ninja-win.zip` 手动解压并加 PATH。
3. 通过 winget 安装。

```powershell
winget install Ninja-build.Ninja
```

验证：

```powershell
ninja --version
```

## 7. vcpkg 安装与 `VCPKG_ROOT` 配置

KR2 使用 vcpkg manifest（根目录有 `vcpkg.json`）。
依赖在 configure/build 过程中自动解析。

### 7.1 安装

建议安装到短路径，避免空格：

```powershell
git clone https://github.com/microsoft/vcpkg.git D:/vcpkg
D:/vcpkg/bootstrap-vcpkg.bat
```

### 7.2 配置环境变量

变量名：`VCPKG_ROOT`

示例值：`D:/vcpkg`

图形界面路径：

1. 打开“编辑系统环境变量”。
2. 点击“环境变量”。
3. 新建变量 `VCPKG_ROOT`。

### 7.3 路径写法建议

- 推荐：`D:/vcpkg`
- 可用：`D:\\vcpkg`
- 不推荐在 CMake 字符串场景直接写 `D:\vcpkg`

### 7.4 为什么这是硬性要求

根 `CMakeLists.txt` 写了：

```cmake
set(CMAKE_TOOLCHAIN_FILE $ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake)
```

变量错误会导致 configure 失败。

## 8. Git for Windows 配置

### 8.1 安装

下载地址：https://git-scm.com/download/win

安装时建议选择：

- `Git from the command line and also from 3rd-party software`

### 8.2 基础配置

```powershell
git --version
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
```

### 8.3 行尾建议

```powershell
git config --global core.autocrlf true
```

## 9. winflexbison 安装与 PATH 配置

KR2 在 `cpp/core/tjs2/CMakeLists.txt` 中搜索 `win_bison.exe`。
因此 Windows 下必须正确安装 winflexbison。

### 9.1 安装步骤

1. 打开发布页：https://github.com/lexxmark/winflexbison/releases
2. 下载 zip（例如 2.5.25）。
3. 解压到 `C:\tools\winflexbison`。
4. 将该目录加入 PATH。

### 9.2 验证

```powershell
win_bison --version
win_flex --version
```

## 10. Python 与 NASM（可选）

如果你后续要扩展依赖编译链，建议提前安装 Python 与 NASM：

- Python 下载：https://www.python.org/downloads/windows/
- NASM 下载：https://www.nasm.us/

验证命令：`python --version`、`nasm --version`

## 11. PowerShell vs CMD vs Git Bash

| 终端 | 优点 | 缺点 | 推荐场景 |
| :--- | :--- | :--- | :--- |
| PowerShell | 与 Windows 集成好、脚本能力强 | 语法与 Bash 不同 | **默认推荐：构建与排障** |
| CMD | 兼容传统批处理 | 功能较弱 | 快速执行旧脚本 |
| Git Bash | 接近 Linux 命令习惯 | 与 VS 环境变量偶有差异 | Git 操作、类 Unix 命令 |

建议同一次排障固定一种终端，
避免路径转义和变量语法混用。

## 12. 环境变量汇总表

| 变量名 | 必需 | 示例值 | 作用 |
| :--- | :---: | :--- | :--- |
| `VCPKG_ROOT` | 是 | `D:/vcpkg` | 指向 vcpkg 根目录，供 CMake 注入 toolchain |
| `PATH` | 是 | 包含 cmake/ninja/git/winflexbison/python/nasm | 让命令在全局可执行 |

建议加入 PATH 的目录：

1. CMake 可执行目录。
2. Ninja 可执行目录。
3. Git 可执行目录。
4. `C:\tools\winflexbison`。
5. Python 可执行目录。
6. NASM 可执行目录。

## 13. 验证工具安装成功的命令清单

```powershell
cmake --version
ninja --version
git --version
cl
python --version
nasm --version
win_bison --version
win_flex --version
echo $env:VCPKG_ROOT
Test-Path "$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"
```

判断标准：

1. 各工具命令都能返回版本。
2. `VCPKG_ROOT` 不为空。
3. `Test-Path` 返回 `True`。

## 14. 动手实践

> 目标：按项目标准流程完成 Windows Debug 构建。

### 步骤 1：进入仓库目录

```powershell
cd C:/Users/Administrator/Downloads/code/KR2/krkr2
```

### 步骤 2：执行 configure

```powershell
cmake --preset="Windows Debug Config" -DDISABLE_TEST=ON
```

### 步骤 3：执行 build

```powershell
cmake --build --preset="Windows Debug Build"
```

### 步骤 4：确认产物目录

应出现：`out/windows/debug`

该流程与 `scripts/build-windows.bat` 保持一致。

## 15. 常见 Windows 环境问题排查

| 现象 | 快速检查命令 | 常见根因 | 处理建议 |
| :--- | :--- | :--- | :--- |
| `cmake` 不存在 | `where cmake` | CMake 未安装或 PATH 未生效 | 安装 CMake，重开终端 |
| `ninja` 不存在 | `where ninja` | Ninja 未安装 | 安装 Ninja 并加入 PATH |
| 找不到 toolchain | `echo $env:VCPKG_ROOT` / `Test-Path "$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"` | `VCPKG_ROOT` 错误 | 修正变量并重开终端 |
| `win_bison` 找不到 | `where win_bison` | winflexbison 未加入 PATH | 加入 `C:\tools\winflexbison` |
| `cl` 不可用 | `cl` | VS 组件缺失或环境未激活 | 在 VS Installer 补齐 v143+SDK |
| triplet 混用 | 查看 configure 日志 | 手工参数与 preset 冲突 | 统一使用 preset，清理 build |

## 16. 对照项目源码

- `CMakePresets.json`：Windows 生成器 `Ninja`，triplet `x64-windows-static-md`。
- `CMakeLists.txt`：最低 CMake 3.28、C++17、`VCPKG_ROOT` toolchain 注入。
- `cpp/core/tjs2/CMakeLists.txt`：`find_program(BISON_EXECUTABLE NAMES win_bison.exe bison.exe bison)`。
- `scripts/build-windows.bat`：Debug configure/build 标准命令。

## 17. 本节小结

- Windows 基础链路：MSVC + CMake + Ninja + vcpkg + winflexbison。
- 最高频问题：PATH 与 `VCPKG_ROOT`。
- 最稳妥流程：先“版本验证”，再“preset configure/build”。

## 18. 练习题与答案

### 题目 1：为什么 KR2 在 Windows 上必须安装 Ninja？

<details>
<summary>查看答案</summary>

因为项目在 `CMakePresets.json` 的 Windows 预设里明确使用 `Ninja` 作为生成器。
没有 Ninja，`cmake --preset` 无法生成构建系统。

</details>

### 题目 2：`VCPKG_ROOT` 配置错误会导致什么典型错误？

<details>
<summary>查看答案</summary>

会在 configure 阶段出现 `Could not find toolchain file` 或类似报错。
因为根 `CMakeLists.txt` 会从 `$ENV{VCPKG_ROOT}` 拼接 vcpkg toolchain 路径。

</details>

### 题目 3：为什么要单独安装 winflexbison？

<details>
<summary>查看答案</summary>

KR2 的 TJS2 模块在 CMake 中会查找 `win_bison.exe`。
Windows 下若没有 winflexbison 或 PATH 未配置，会导致语法文件生成失败。

</details>

## 19. 下一步
- [02-Linux环境搭建](./02-Linux环境搭建.md)
你将看到同一项目在 Linux 平台上的工具安装与排障差异。
