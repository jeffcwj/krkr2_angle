# CMake 搭配 Ninja

> **所属模块：** P01-现代 CMake 与构建工具链
> **前置知识：** 01-Ninja 简介与安装
> **预计阅读时间：** 20 分钟

## 本节目标
读完本节后，你将能够：
1. 学会如何在 CMake 中指定 Ninja 作为生成器。
2. 掌握 Windows 上针对 Ninja 的开发环境配置。
3. 实现从配置到构建的完整工作流。
4. 运用 Ninja 的并行、增量及详细日志等高级构建特性。

## 指定 Ninja 生成器
在使用 CMake 时，如果你想生成 `build.ninja` 而不是默认的 Makefile 或解决方案文件，你需要通过 `-G` 参数指定生成器。

### 命令行指定
```bash
# -B 指定输出目录为 build，-G 指定生成器为 Ninja
cmake -B build -G Ninja
```

### 使用 CMakePresets.json 指定 (KrKr2 的做法)
在 `CMakePresets.json` 中，我们可以为不同的平台定义生成器。这样你就不用每次都敲长长的命令行参数了。

```json
{
  "version": 3,
  "configurePresets": [
    {
      "name": "default-ninja",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build"
    }
  ]
}
```
运行预设命令：
```bash
cmake --preset default-ninja
```

## 完整构建流程

### 1. Windows 平台（环境要求）
在 Windows 上使用 Ninja 时，Ninja 本身并不包含编译器。通常我们需要搭配 **MSVC** (Microsoft Visual C++ Compiler, `cl.exe`)。

**关键点：环境变量**
`cl.exe` 不能在普通的命令行中直接运行，因为它需要一系列环境变量（如 `INCLUDE`, `LIB`）来找到 SDK 路径。

* **正确做法：使用 Developer Command Prompt**
  你可以在开始菜单搜索“Developer Command Prompt for VS 2022”或“x64 Native Tools Command Prompt for VS 2022”。

在开发者命令行中执行：
```powershell
# 1. 配置工程
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release

# 2. 执行构建
cmake --build build
```

如果你在普通命令行中运行，CMake 会报错提示 `cl.exe` 找不到，或者编译一个简单的测试程序失败。

### 2. Linux 和 macOS 平台
相比 Windows，类 Unix 系统（Linux/macOS）的环境通常更加简单。只要你安装了 `gcc` 或 `clang`，Ninja 就能直接上手。

```bash
# 1. 配置工程
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug

# 2. 执行构建
cmake --build build
```

### 3. 使用 CMakePresets 的统一工作流
在 KrKr2 项目中，我们提倡使用统一的预设工作流：

```bash
# Windows / Linux / macOS 通用
cmake --preset "Windows-Debug-Config"
cmake --build --preset "Windows-Debug-Build"
```
（具体的预设名称请参考项目根目录下的 `CMakePresets.json`）

## 高级构建特性

### 并行构建
Ninja 最显著的特点就是它的**自动并行化**。

当你执行 `cmake --build build` 时，CMake 会在后台调用 `ninja`。默认情况下，Ninja 会检测你的 CPU 核心数，并自动开启相应数量的线程。

如果你想手动控制并行程度（例如在内存较小的机器上）：
```bash
# 使用 CMake 通用命令传递 -j 参数给底层构建器
cmake --build build -- -j4

# 或者直接调用 ninja 命令
ninja -C build -j4
```
注：`-C build` 告诉 Ninja 进入 `build` 目录执行构建任务。

### 增量构建
增量构建（Incremental Build）是开发中最常用的功能。当你只修改了一个源文件时，Ninja 只需要重新编译这一个文件。

**实验环节：**
1. 运行 `cmake --build build` 完成初次构建。
2. 打开项目中的任意 `.cpp` 文件，加一行空注释。
3. 再次运行 `cmake --build build`。
4. 你会看到 Ninja 几乎瞬间就完成了，并提示 `[1/1] Linking...`。

### 清理构建
如果你想删除所有生成的可执行文件、库以及中间产物（`.obj`, `.o`）：

```bash
# 使用 CMake 通用命令
cmake --build build --target clean

# 直接调用 ninja
ninja -C build clean
```

### 查看详细日志 (Verbose Mode)
在某些时候，编译报错，但你看不出是因为什么参数导致。这时你需要查看完整的编译命令。

```bash
# 启用详细输出
cmake --build build -- -v

# 或者直接调用 ninja
ninja -C build -v
```
它会输出类似 `cl.exe /c /Iinclude ... main.cpp` 的完整参数，方便你排查包含路径、宏定义等问题。

## 常见问题排查

### 1. "ninja: error: loading 'build.ninja': 系统找不到指定的文件"
**现象：** 运行 `ninja` 或 `cmake --build build` 报错。
**解决：** 你还没有进行“配置（Configure）”阶段。请先运行 `cmake -B build -G Ninja` 确保 `build.ninja` 被生成。

### 2. "cl is not recognized" (Windows)
**现象：** CMake 配置阶段提示找不到 C 编译器。
**解决：** 你没有在“开发者命令行（Developer Command Prompt）”中运行。请从 VS 菜单中打开专用命令行，或者在普通命令行中手动执行 `vcvarsall.bat`。

### 3. 构建卡住或系统响应缓慢
**现象：** 电脑风扇狂转，屏幕卡死。
**解决：** Ninja 默认并行度过高，导致内存占满。尝试减少并行数，如使用 `-j2` 或 `-j4`。

## 对照 KrKr2 构建脚本
为了简化操作，KrKr2 提供了一些方便的脚本。你可以阅读 `scripts/build-windows.bat`，你会发现它本质上也是在做同样的几件事：

```batch
:: scripts/build-windows.bat (示例片段)
@echo off
set BUILD_DIR=build/windows-release
cmake -B %BUILD_DIR% -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build %BUILD_DIR%
```

而在 Linux 脚本 `scripts/build-linux.sh` 中：

```bash
#!/bin/bash
# scripts/build-linux.sh (示例片段)
BUILD_DIR="build/linux-debug"
cmake -B "$BUILD_DIR" -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build "$BUILD_DIR"
```

## 本节小结
本节你掌握了 CMake 与 Ninja 的“黄金搭档”用法。你学会了如何在 Windows 开发者环境下配置工程，以及如何利用并行构建和详细日志来加速和优化你的开发流程。

## 练习题与答案

### 题目 1：如果你已经在 CMakePresets.json 中设置了 `"generator": "Ninja"`，在配置阶段还需要手动敲 `-G Ninja` 吗？
<details><summary>查看答案</summary>
不需要。当你使用 `cmake --preset <name>` 时，CMake 会自动读取预设中的 `generator` 设置。
</details>

### 题目 2：在 Windows 上，直接在 PowerShell 中运行 `cmake -B build -G Ninja` 提示 `cl.exe` 找不到，应该如何解决？
<details><summary>查看答案</summary>
应该在“Visual Studio 开发者命令行（Developer Command Prompt）”中运行，或者在当前 PowerShell 窗口中先执行 Visual Studio 提供的 `vcvarsall.bat` 脚本来初始化环境变量。
</details>

### 题目 3：如何查看 Ninja 构建时具体的编译参数（例如包含了哪些头文件路径）？
<details><summary>查看答案</summary>
使用详细模式（Verbose Mode），在构建时添加 `-v` 参数，即：`cmake --build build -- -v` 或 `ninja -C build -v`。
</details>

## 下一步
→ 恭喜！你已完成第五章“Ninja 构建后端”的学习。接下来的章节我们将深入探讨 P01 模块的其他高级特性。
