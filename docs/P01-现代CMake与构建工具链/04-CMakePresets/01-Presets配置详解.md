# 01-Presets配置详解

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

---

## JSON 结构详解

一个标准的 `CMakePresets.json` 文件由几个核心字段组成：

```json
{
  "version": 6,
  "cmakeMinimumRequired": {
    "major": 3,
    "minor": 28,
    "patch": 0
  },
  "configurePresets": [],
  "buildPresets": [],
  "testPresets": [],
  "workflowPresets": []
}
```

- **`version`**：JSON 格式的版本号。建议使用较新版本（如 6），以支持更多特性。
- **`cmakeMinimumRequired`**：指定运行此预设所需的最低 CMake 版本。
- **`configurePresets`**：**最核心的部分**，定义配置阶段（生成 Makefile/Ninja 文件的阶段）的参数。
- **`buildPresets`**：定义构建阶段（编译代码的阶段）的参数，必须关联一个 `configurePreset`。

---

## configurePreset 字段详解

每一个配置预设都是一个对象，包含以下常用字段：

### 1. 基础信息
- **`name`**：预设的唯一标识名。
- **`hidden`**：布尔值。如果为 `true`，该预设不会出现在 IDE 的选择列表中，通常用作被其他预设继承的基础预设。
- **`inherits`**：字符串或数组，表示继承自哪些预设。
- **`generator`**：指定生成器，如 `"Ninja"` 或 `"Visual Studio 17 2022"`。

### 2. 目录与变量
- **`binaryDir`**：指定构建目录（输出目录），支持变量替换，如 `"${sourceDir}/out/build/${presetName}"`。
- **`cacheVariables`**：一个对象，定义 CMake 缓存变量（即 `-D` 参数）。
  ```json
  "cacheVariables": {
    "CMAKE_BUILD_TYPE": "Debug",
    "MY_FEATURE_ENABLED": "ON"
  }
  ```

### 3. 环境与条件
- **`condition`**：条件表达式。只有满足条件时，该预设才可用。例如只在 Windows 上启用。
- **`architecture`**：指定目标架构，主要用于 Visual Studio 生成器（如 `x64`, `arm64`）。
- **`vendor`**：供应商特定设置，例如为特定的 IDE 提供额外提示。

---

## 继承（inherits）与代码复用

在实际项目中，你可能会发现 Windows Debug 和 Windows Release 有很多重复的配置（如都要用 Ninja、都要定义特定的宏）。通过 `inherits` 机制，你可以消除重复代码。

```json
{
  "configurePresets": [
    {
      "name": "Base Config",
      "hidden": true,
      "generator": "Ninja",
      "cacheVariables": {
        "PROJECT_VERSION": "1.0.0"
      }
    },
    {
      "name": "Windows Debug",
      "inherits": "Base Config",
      "binaryDir": "${sourceDir}/out/debug",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug"
      }
    }
  ]
}
```

在上例中，`Windows Debug` 预设会自动继承 `Base Config` 中的生成器（Ninja）和项目版本变量。

---

## condition 字段详解

为了实现跨平台支持，你可以使用 `condition` 字段让预设只在特定环境下生效。

```json
"condition": {
  "type": "equals",
  "lhs": "${hostSystemName}",
  "rhs": "Windows"
}
```

- **`type`**：条件类型。常用有 `equals`（等于）、`notEquals`（不等于）、`anyOf`（任一满足）、`allOf`（全部满足）。
- **`lhs`**（Left Hand Side）：左值，通常是一个内置变量（如 `${hostSystemName}`）。
- **`rhs`**（Right Hand Side）：右值。

常用的内置变量包括：
- `${sourceDir}`：项目源码根目录。
- `${presetName}`：当前预设的名称。
- `${hostSystemName}`：宿主系统名称，如 `"Windows"`, `"Linux"`, `"Darwin"` (macOS)。

---

## 使用 Presets 进行构建

配置好 JSON 后，你再也不需要输入长长的参数。只需使用以下命令：

1. **配置（Configure）**：
   ```bash
   cmake --preset "Windows Debug Config"
   ```
   这会自动根据预设名在 JSON 中查找对应的生成器、变量和构建目录。

2. **构建（Build）**：
   ```bash
   cmake --build --preset "Windows Debug Build"
   ```
   构建预设会自动关联其对应的配置预设，从而知道去哪个目录下寻找构建文件。

---

## 练习题与答案

### 题目 1：理解 `hidden` 属性

如果你在 `CMakePresets.json` 中定义了一个 `hidden: true` 的预设，为什么在 VS Code 的 CMake Tools 插件的下拉列表中找不到它？

<details>
<summary>查看答案</summary>

`hidden` 属性的作用就是将该预设标记为"内部预设"。它主要用于作为其他预设的基础模板（通过 `inherits` 继承），而不直接供用户手动选择和运行。IDE 和 `cmake --list-presets` 都会自动隐藏这些预设。

</details>

### 题目 2：配置构建目录

请写出一个 `configurePreset` 片段，要求该预设继承自 `base`，并且将构建输出目录设置为项目根目录下的 `build_output` 文件夹。

<details>
<summary>查看答案</summary>

```json
{
  "name": "my-preset",
  "inherits": "base",
  "binaryDir": "${sourceDir}/build_output"
}
```

关键点：`${sourceDir}` 是 CMake Presets 内置变量，指向项目根目录（即 `CMakePresets.json` 所在目录）。

</details>

### 题目 3：跨平台条件判断

如何配置一个预设，使其仅在 macOS 系统上生效？

<details>
<summary>查看答案</summary>

在 `configurePresets` 的对象中加入如下 `condition` 字段：

```json
"condition": {
  "type": "equals",
  "lhs": "${hostSystemName}",
  "rhs": "Darwin"
}
```

注意：CMake 将 macOS 识别为 `Darwin`（其底层内核名称），而不是 `MacOS` 或 `macOS`。

</details>

