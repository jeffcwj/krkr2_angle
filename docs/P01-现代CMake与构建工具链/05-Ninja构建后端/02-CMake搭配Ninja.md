# CMake 搭配 Ninja

> **所属模块：** P01-现代 CMake 与构建工具链  
> **前置知识：** [01-Ninja简介与安装](./01-Ninja简介与安装.md)  
> **预计阅读时间：** 25 分钟  
> **适用平台：** Windows / Linux / macOS / Android（交叉编译链路）

## 本节目标

读完本节后，你将能够：

1. 说清 CMake 与 Ninja 在构建流程中的分工。
2. 正确使用 `-G "Ninja"` 与 `-G "Ninja Multi-Config"`。
3. 基于 KrKr2 的 `CMakePresets.json` 执行统一构建。
4. 理解单配置与多配置对 Debug/Release 管理的差异。
5. 生成并使用 `compile_commands.json` 完成 IDE 集成。
6. 用 `ccache + Ninja` 进一步减少重复编译开销。
7. 用 `-v` 与 `-d explain` 快速定位构建问题。

## CMake 与 Ninja 的协作关系

可以把流程拆成两步：

1. **配置阶段（Configure）**：CMake 读取 CMakeLists，生成 `build.ninja`。
2. **构建阶段（Build）**：Ninja 执行 `build.ninja` 中的规则。

因此：

- `cmake -S . -B out -G Ninja` 负责“生成规则”。
- `cmake --build out` 负责“执行规则”。

这个分层让你能在保持 CMakeLists 不变的情况下切换后端。

## `-G "Ninja"` 与 `-G "Ninja Multi-Config"` 的区别

### `-G "Ninja"`：单配置

单配置模式下，一个构建目录只对应一个构建类型。
你通常用 `CMAKE_BUILD_TYPE` 指定类型。

```bash
cmake -S . -B out/linux/debug -G "Ninja" -DCMAKE_BUILD_TYPE=Debug
cmake --build out/linux/debug

cmake -S . -B out/linux/release -G "Ninja" -DCMAKE_BUILD_TYPE=Release
cmake --build out/linux/release
```

### `-G "Ninja Multi-Config"`：多配置

多配置模式下，一个构建目录中可以同时管理多种配置。
构建时用 `--config` 选择具体配置。

```bash
cmake -S . -B out/linux/multi -G "Ninja Multi-Config"
cmake --build out/linux/multi --config Debug
cmake --build out/linux/multi --config Release
```

### 对照结论

| 维度 | Ninja | Ninja Multi-Config |
|---|---|---|
| 配置目录 | 每个配置一个目录 | 一个目录多个配置 |
| 主要控制参数 | `CMAKE_BUILD_TYPE` | `--config` |
| 切换方式 | 切目录 | 同目录切换 |

KrKr2 当前预设采用的是单配置 `Ninja`。

## KrKr2 的 CMakePresets.json 中 Ninja 实际配置

根据项目当前 `krkr2/CMakePresets.json`：

1. `Windows Config` 生成器是 `Ninja`。
2. `Linux Config` 生成器是 `Ninja`。
3. `MacOS Config` 生成器是 `Ninja`。
4. 三平台基础预设都打开了 `CMAKE_EXPORT_COMPILE_COMMANDS=true`。

这说明 Ninja 是项目当前默认后端，而非文档示例。

### Debug / Release 的组织

项目通过继承预设并拆分目录：

- Linux Debug：`out/linux/debug`
- Linux Release：`out/linux/release`
- Windows 与 macOS 也是同样模式

这是单配置 Ninja 的标准管理方式。

## 用 CMakePresets 驱动 Ninja

### Windows

在 Visual Studio 开发者命令行执行：

```powershell
cmake --preset "Windows Debug Config"
cmake --build --preset "Windows Debug Build"
```

### Linux

```bash
cmake --preset "Linux Debug Config"
cmake --build --preset "Linux Debug Build"
```

### macOS

```bash
cmake --preset "MacOS Debug Config"
cmake --build --preset "MacOS Debug Build"
```

### Android（交叉编译链路）

KrKr2 Android 主要入口是 Gradle：

```bash
./platforms/android/gradlew -p ./platforms/android assembleDebug
```

你可以把它理解为“Gradle 驱动 CMake，再由 Ninja 执行 native 构建”。

## Ninja Multi-Config 的多构建类型管理

虽然 KrKr2 当前未使用 Multi-Config，但你应知道其适用场景。

适合：

1. 频繁在 Debug/Release 间切换。
2. 希望一个目录内同时保留多配置产物。

迁移注意：

1. 脚本中若依赖 `CMAKE_BUILD_TYPE`，需改为 `--config`。
2. CI 与缓存目录策略需要同步调整。

## compile_commands.json 与 IDE 集成

`compile_commands.json` 是编译命令数据库。
它记录每个源文件的真实编译命令和参数。

KrKr2 已通过预设默认开启：

```json
"CMAKE_EXPORT_COMPILE_COMMANDS": true
```

常见输出位置：

- `out/windows/debug/compile_commands.json`
- `out/linux/debug/compile_commands.json`
- `out/macos/debug/compile_commands.json`

### VS Code + clangd 示例

```json
{
  "clangd.arguments": [
    "--compile-commands-dir=out/linux/debug"
  ]
}
```

### 使用收益

1. 补全与跳转更准确。
2. 诊断更贴近真实构建参数。
3. 跨文件重构误报更少。

## ccache + Ninja 加速编译

Ninja 负责快调度，ccache 负责快复用。

```bash
cmake -S . -B out/linux/debug -G "Ninja" \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_C_COMPILER_LAUNCHER=ccache \
  -DCMAKE_CXX_COMPILER_LAUNCHER=ccache

cmake --build out/linux/debug
ccache -s
```

Windows + MSVC 场景一般用 `sccache` 思路。

## Ninja 调试构建问题的技巧

### `-v`：输出完整构建命令

```bash
cmake --build out/linux/debug -- -v
```

用于检查 include、宏、链接顺序和编译器路径。

### `-d explain`：解释重构建原因

```bash
ninja -C out/linux/debug -d explain
```

用于回答“为什么这个目标被重建”。

### 联合使用

```bash
ninja -C out/linux/debug -v -d explain
```

## 并行度控制与资源管理

Ninja 默认并行通常较高。
你可以显式控制并发线程：

```bash
cmake --build out/linux/debug -- -j8
ninja -C out/linux/debug -j8
```

建议：

1. 16GB 内存可从 `-j4` 起步。
2. 32GB 内存可尝试 `-j8`。
3. 链接阶段内存紧张时应下调并发。

## KrKr2 项目中的实际流程演示

### Linux 开发流程

```bash
cmake --preset "Linux Debug Config"
cmake --build --preset "Linux Debug Build"
cmake --build --preset "Linux Debug Build" -- -v
ninja -C out/linux/debug -d explain
```

### Windows 发布前流程

```powershell
cmake --preset "Windows Release Config"
cmake --build --preset "Windows Release Build"
```

### macOS 对比流程

```bash
cmake --preset "MacOS Debug Config"
cmake --build --preset "MacOS Debug Build"
cmake --preset "MacOS Release Config"
cmake --build --preset "MacOS Release Build"
```

## 对照项目源码

相关文件：

- `krkr2/CMakePresets.json` 第 8-129 行：`configurePresets` 主体。
- `krkr2/CMakePresets.json` 第 10-12、68-70、98-100 行：三平台统一 `generator: Ninja`。
- `krkr2/CMakePresets.json` 第 34-47、82-95、114-127 行：Debug/Release 分目录与构建类型。
- `krkr2/CMakePresets.json` 第 13-17、71-74、101-106 行：开启 `CMAKE_EXPORT_COMPILE_COMMANDS`。
- `krkr2/CMakePresets.json` 第 130-154 行：`buildPresets` 映射构建入口。

## 动手实践

请按步骤实际执行一次：

### 步骤 1：列出预设

```bash
cmake --list-presets
```

### 步骤 2：执行一次 Debug 构建

```bash
cmake --preset "Linux Debug Config"
cmake --build --preset "Linux Debug Build"
```

### 步骤 3：检查命令数据库

确认 `out/linux/debug/compile_commands.json` 已生成。

### 步骤 4：触发增量构建

修改一个源文件中的注释后再次构建，观察只重编译少量目标。

### 步骤 5：排查重建原因

```bash
cmake --build --preset "Linux Debug Build" -- -v
ninja -C out/linux/debug -d explain
```

## 本节小结

- KrKr2 当前统一采用单配置 Ninja 后端。
- Debug/Release 通过 preset + 分目录管理，清晰且稳定。
- `compile_commands.json` 已默认开启，IDE 与 LSP 集成成本低。
- `-v` 与 `-d explain` 是 Ninja 排障核心命令。
- 合理并行度与缓存策略能显著优化构建体验。

## 练习题与答案

### 题目 1：KrKr2 当前使用的是哪种 Ninja 生成器模式？依据是什么？

<details>
<summary>查看答案</summary>

是单配置模式。  
依据：`CMakePresets.json` 的 `generator` 使用 `Ninja`，不是 `Ninja Multi-Config`。  
并且 Debug/Release 通过不同 `binaryDir` 与 `CMAKE_BUILD_TYPE` 管理。

</details>

### 题目 2：为什么 `compile_commands.json` 对编辑器体验影响很大？

<details>
<summary>查看答案</summary>

因为它提供了真实编译参数。  
clangd 等工具据此解析 include、宏与标准选项。  
没有它时，补全、跳转和诊断常与真实构建不一致。

</details>

### 题目 3：当构建意外大面积重编译时，应先执行什么命令？

<details>
<summary>查看答案</summary>

优先执行：

```bash
cmake --build <build_dir> -- -v
ninja -C <build_dir> -d explain
```

`-v` 查看具体执行命令，`-d explain` 查看触发重建原因。

</details>

## 下一步

完成本节后，进入 P01 第六章实战：

- [01-根CMakeLists解读](../06-实战-分析KrKr2的CMake/01-根CMakeLists解读.md)
