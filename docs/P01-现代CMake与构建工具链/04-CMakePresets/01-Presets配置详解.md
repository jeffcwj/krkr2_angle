# 01-Presets配置详解

> **所属模块：** P01-现代CMake与构建工具链  
> **前置知识：** `../03-构建类型与编译选项/`、`../02-CMake目标与链接/`  
> **预计阅读时间：** 45 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 Presets 在团队构建中的价值。
2. 区分 `CMakePresets.json` 与 `CMakeUserPresets.json` 的职责。
3. 理解 JSON Schema 版本与 CMake 兼容性关系。
4. 系统掌握 `configurePresets`、`buildPresets`、`testPresets` 字段。
5. 使用 `inherits + hidden` 搭建可复用预设体系。
6. 使用 `condition`、`$env{}`、`$penv{}`、宏展开处理跨平台差异。
7. 能逐行读懂 KrKr2 的 `krkr2/CMakePresets.json`。

在前面的章节中，你已经学会了如何编写 `CMakeLists.txt`，管理变量、目标和属性。然而在实际开发中，你可能会发现每次配置项目时都要输入一大串命令：

```bash
cmake -B out/build -G Ninja -DCMAKE_BUILD_TYPE=Debug -DVCPKG_TARGET_TRIPLET=x64-windows-static-md -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

这不仅难以记忆，而且团队中的不同开发者可能会使用不同的参数，导致构建环境不一致。为了解决这个问题，CMake 引入了 **Presets**（预设）机制。

---

## 为什么需要 Presets

在没有 Presets 之前，开发者通常会编写脚本（如 `.sh` 或 `.bat`）来封装复杂的 CMake 命令。但脚本存在跨平台不兼容、IDE 难以识别等问题。

Presets 的核心价值在于：
1. **统一化**：将复杂的命令行参数保存到 JSON 文件中，全团队共享。
2. **自动化**：IDE（如 VS Code, CLion, Visual Studio）可以直接读取这些配置，提供图形化界面。
3. **标准化**：消除“在我电脑上能跑”的尴尬，确保每个人都在相同的配置下构建。

---

## CMakePresets.json vs CMakeUserPresets.json

CMake 识别两个预设文件：

1. **`CMakePresets.json`**：
   - **用途**：存储项目通用的配置，如 Release/Debug 模式、工具链路径等。
   - **版本控制**：**必须**提交到 Git 等版本控制系统，供所有开发者使用。
   - **优先级**：基础配置。

2. **`CMakeUserPresets.json`**：
   - **用途**：存储开发者个人的私有配置，例如你本地特定的编译器路径或临时的测试开关。
   - **版本控制**：**严禁**提交到版本控制系统（应加入 `.gitignore`）。
   - **优先级**：会覆盖或补充 `CMakePresets.json` 中的同名预设。

实战建议：

- 团队默认配置一定放在 `CMakePresets.json`，这样 CI、IDE、命令行都统一。
- 个人磁盘路径、本机 SDK 路径放在 `CMakeUserPresets.json`，避免污染仓库。

---

## JSON Schema 版本和兼容性

`CMakePresets.json` 顶层 `version` 字段表示 **Presets 文件格式版本**，不是 CMake 可执行程序版本。

KrKr2 使用的是：

```json
"version": 6
```

同时还声明了最低 CMake 版本：

```json
"cmakeMinimumRequired": {
  "major": 3,
  "minor": 28,
  "patch": 0
}
```

兼容性检查顺序：

1. `cmake --version` 是否满足 `3.28.0+`。
2. IDE 内置 CMake 版本是否一致。
3. 再检查 JSON 字段拼写。

跨平台提示：

- Windows：VS 内置 CMake 可能落后。
- Linux：系统仓库可能提供旧版 CMake。
- macOS：Homebrew 安装后注意 PATH 优先级。
- Android：上层工具链调用的 CMake 也要满足最低版本。

---

## configurePresets 完整字段详解

每一个配置预设都是一个对象，包含以下常用字段：

### 1. 基础信息
- **`name`**：预设唯一标识名。
- **`hidden`**：若为 `true`，预设会被隐藏，常用作模板。
- **`inherits`**：继承父预设（字符串或数组）。
- **`generator`**：生成器，如 `Ninja`。

### 2. 目录与变量
- **`binaryDir`**：构建目录，支持宏，例如 `${sourceDir}`、`${presetName}`。
- **`cacheVariables`**：等价命令行 `-D` 参数。

```json
"cacheVariables": {
  "CMAKE_BUILD_TYPE": "Debug",
  "CMAKE_EXPORT_COMPILE_COMMANDS": true
}
```

### 3. 环境与条件
- **`condition`**：决定预设是否可见/可用。
- **`architecture`**：目标架构声明。
- **`vendor`**：IDE 厂商扩展信息。
- **`environment`**：预设环境变量。
- **`toolchainFile`**：工具链文件路径。
- **`installDir`**：安装目录。

---

## buildPresets 和 testPresets 详解

`buildPresets` 常见最小写法：

```json
{
  "name": "Windows Debug Build",
  "configurePreset": "Windows Debug Config"
}
```

含义：构建阶段直接复用 configure 预设确定的生成器、目录、缓存变量。

`testPresets` 典型模板：

```json
{
  "name": "Linux Debug Test",
  "configurePreset": "Linux Debug Config",
  "output": {
    "outputOnFailure": true
  },
  "execution": {
    "jobs": 8
  }
}
```

在 KrKr2 当前文件中暂未定义 `testPresets`，后续可按上述模式扩展。

---

## 继承机制：inherits 与 hidden 组合

推荐结构：

1. 平台基座预设 `hidden: true`。
2. Debug/Release 子预设继承基座。
3. 子预设仅覆盖差异字段。

优点：

- 公共字段只维护一次。
- 改动影响面可预测。
- 用户入口更清晰。

KrKr2 的 Windows/Linux/macOS 都采用了该模式。

---

## 条件预设：condition 字段

典型写法：

```json
"condition": {
  "type": "equals",
  "lhs": "${hostSystemName}",
  "rhs": "Windows"
}
```

常见 `type`：

- `equals`
- `notEquals`
- `allOf`
- `anyOf`

KrKr2 使用值：

- Windows → `Windows`
- Linux → `Linux`
- macOS → `Darwin`

---

## 环境变量引用与宏展开

### 环境变量引用

- `$env{VAR}`：读取当前预设环境变量。
- `$penv{VAR}`：读取父进程环境变量。

常见用途：

1. 继承 `PATH`。
2. 读取 `VCPKG_ROOT`。
3. 读取 `ANDROID_NDK`。

### 宏展开

常见宏：

- `${sourceDir}`
- `${presetName}`
- `${hostSystemName}`

示例：

```json
"binaryDir": "${sourceDir}/out/build/${presetName}"
```

---

## KrKr2 的 CMakePresets.json 逐行解读（实战案例）

文件路径：`krkr2/CMakePresets.json`  
文件行数：156 行

### 顶层（1-7）

- 第2行：`version = 6`
- 第3-7行：`cmakeMinimumRequired = 3.28.0`

### configurePresets（8-129）

1. `Windows Config`（10）
   - `generator`: Ninja（11）
   - `hidden`: true（12）
   - `cacheVariables`: WINDOWS、VCPKG_TARGET_TRIPLET、CMAKE_EXPORT_COMPILE_COMMANDS（13-17）
   - `condition`: Windows（18-22）
   - `architecture`: x64（23-26）
   - `vendor`: Visual Studio 扩展（27-31）

2. `Windows Debug Config`（34）和 `Windows Release Config`（42）
   - 继承 `Windows Config`
   - 输出目录分别为 `out/windows/debug`、`out/windows/release`
   - 分别覆盖 `CMAKE_BUILD_TYPE`

3. `Windows MinGW Config`（50）
   - 在 Linux 主机交叉编译 Windows
   - 指定 MinGW 编译器
   - `condition` 限制 Linux

4. `Linux Config` 与子预设（68-96）
   - 基座隐藏
   - 子预设拆分 Debug/Release

5. `MacOS Config` 与子预设（98-128）
   - `condition` 使用 `Darwin`
   - 指定 `clang`、`clang++`

### buildPresets（130-155）

- 共 6 个 build 预设
- 覆盖三平台 Debug/Release
- 每个条目绑定对应 configure 预设

命令映射：

```bash
cmake --preset "Linux Debug Config"
cmake --build --preset "Linux Debug Build"
```

---

## 动手实践

目标：新增 Linux ASan 调试预设。

### 第1步：新增隐藏基座

```json
{
  "name": "Linux ASan Base",
  "inherits": "Linux Config",
  "hidden": true,
  "cacheVariables": {
    "CMAKE_BUILD_TYPE": "Debug",
    "CMAKE_C_FLAGS": "-fsanitize=address -fno-omit-frame-pointer",
    "CMAKE_CXX_FLAGS": "-fsanitize=address -fno-omit-frame-pointer",
    "CMAKE_EXE_LINKER_FLAGS": "-fsanitize=address"
  }
}
```

### 第2步：新增 configure 预设

```json
{
  "name": "Linux ASan Debug Config",
  "inherits": "Linux ASan Base",
  "binaryDir": "${sourceDir}/out/linux/asan-debug"
}
```

### 第3步：新增 build 预设

```json
{
  "name": "Linux ASan Debug Build",
  "configurePreset": "Linux ASan Debug Config"
}
```

### 第4步：执行验证

```bash
cmake --preset "Linux ASan Debug Config"
cmake --build --preset "Linux ASan Debug Build"
ctest --test-dir out/linux/asan-debug --output-on-failure
```

验证点：

1. `Linux ASan Debug Config` 可见。
2. `Linux ASan Base` 不可见。
3. 编译参数中出现 `-fsanitize=address`。
4. 输出目录是 `out/linux/asan-debug`。

---

## 本节小结

- Presets 把临时命令参数升级为可管理配置资产。
- `CMakePresets.json` 面向团队，`CMakeUserPresets.json` 面向个人。
- `configurePresets` 负责定义，`buildPresets/testPresets` 负责执行。
- `inherits + hidden` 是去重和分层关键模式。
- `condition`、环境变量引用、宏展开是跨平台关键能力。
- KrKr2 的预设文件结构可直接作为扩展模板。

---

## 练习题与答案

### 题目1：基座预设为什么设置 hidden

为什么 `Windows Config`、`Linux Config` 这类预设通常设置 `hidden: true`？

<details>
<summary>查看答案</summary>

因为它们主要承载公共字段，不是最终执行入口。设置 hidden 可以避免误选模板预设，让用户只看到可直接执行的 Debug/Release 入口。

</details>

### 题目2：个人路径应写在哪个文件

你要把本机构建目录改为 `D:/build/krkr2`，应修改哪个文件？

<details>
<summary>查看答案</summary>

应写在 `CMakeUserPresets.json`。这是个人机器差异，不应写入团队共享的 `CMakePresets.json`。

</details>

### 题目3：写出 macOS 的 condition

请给出仅在 macOS 可见的 condition 片段。

<details>
<summary>查看答案</summary>

```json
"condition": {
  "type": "equals",
  "lhs": "${hostSystemName}",
  "rhs": "Darwin"
}
```

</details>

---

## 下一步

继续阅读：[02-多平台预设管理.md](./02-多平台预设管理.md)。

下一节重点：

1. 多平台预设命名规范。
2. 平台层与功能层拆分策略。
3. 本地预设与 CI 预设对齐方法。
