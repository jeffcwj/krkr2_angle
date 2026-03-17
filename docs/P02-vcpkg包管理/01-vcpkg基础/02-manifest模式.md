# 02-manifest 模式

## Classic 模式 vs Manifest 模式

vcpkg 有两种运行模式，理解它们的区别对于构建健壮的项目至关重要。

### 经典模式 (Classic Mode)

在经典模式下，你通过 `vcpkg install <package>` 命令将库安装到 vcpkg 的中央目录中。
- **优点**：简单直接，适合快速试验。
- **缺点**：难以在团队间共享，不同项目可能需要同一库的不同版本，而经典模式通常只能安装一个版本。

### 清单模式 (Manifest Mode)

在清单模式下，你的项目根目录中有一个 `vcpkg.json` 文件。
- **优点**：
  - **可重复构建**：任何人克隆你的项目后，运行同样的构建命令，都能得到相同版本的依赖库。
  - **项目级隔离**：每个项目可以拥有独立的依赖列表和版本。
  - **易于版本控制**：只需将 `vcpkg.json` 提交到 Git，团队其他成员就能自动同步依赖。

## vcpkg.json 详解

`vcpkg.json` 是清单模式的核心，它是一个 JSON 格式的文件，描述了项目的依赖关系。

### 基础结构示例

```json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/vcpkg-tool/main/docs/vcpkg.schema.json",
  "name": "my-cool-project",
  "version-string": "1.0.0",
  "dependencies": [
    "zlib",
    "nlohmann-json",
    "fmt"
  ]
}
```

- **$schema**：提供编辑器的智能提示，帮助你纠正拼写错误。
- **name**：你的项目名称（必须全小写，不能有空格）。
- **version-string**：项目的版本号。
- **dependencies**：一个数组，列出你项目需要的所有包。

## vcpkg-configuration.json 详解

对于更高级的配置（如锁定特定版本或使用私有仓库），我们需要 `vcpkg-configuration.json`。

### 核心配置项

```json
{
  "default-registry": {
    "kind": "builtin",
    "baseline": "d644865_your_commit_hash"
  }
}
```

- **default-registry**：指定默认的包来源。
- **baseline**：基准版本。它是一个 Git 提交哈希（Commit Hash），定义了在该时间点 vcpkg 仓库中所有包的版本。这确保了团队成员使用的包版本完全一致。

---
(待续)

## 创建 Manifest 项目的完整步骤

让我们在不同平台上创建一个真实的项目并管理它的依赖。

### 1. 准备目录与代码

在任意平台上，创建一个空目录并进入：
```bash
mkdir my_project && cd my_project
```

### 2. 初始化清单文件

手动创建一个名为 `vcpkg.json` 的文件，填入以下内容：
```json
{
  "name": "json-test",
  "version-string": "0.1.0",
  "dependencies": [
    "nlohmann-json"
  ]
}
```

### 3. 配置集成（以 CMake 为例）

这是关键步骤。你需要告诉 CMake 使用 vcpkg 作为依赖解析器。

**Windows / Linux / macOS 通用命令**：
```bash
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE="$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"
```
或者，在你的 `CMakeLists.txt` 第一行（`project()` 之前）添加：
```cmake
set(CMAKE_TOOLCHAIN_FILE "$ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake" CACHE STRING "")
```

### 4. 自动下载与构建

当你运行 `cmake --build build` 时，vcpkg 会检测到 `vcpkg.json`，自动下载 `nlohmann-json` 及其依赖，并针对你的当前系统（Windows/Linux/macOS）完成编译。

## 依赖版本管理与 Overrides

有时候你可能需要特定版本的包。vcpkg 提供了 `overrides`（覆盖）机制。

```json
{
  "name": "custom-version-project",
  "version-string": "1.0.0",
  "dependencies": [
    "fmt"
  ],
  "overrides": [
    {
      "name": "fmt",
      "version": "8.1.1"
    }
  ]
}
```
这段代码强制 vcpkg 安装 `fmt` 库的 8.1.1 版本，而不管 `baseline` 是如何规定的。

## 练习题与答案

### 题目 1：Manifest 模式的优势

为什么在团队开发中，我们应该优先选择清单模式（Manifest Mode）而不是经典模式（Classic Mode）？

<details>
<summary>查看答案</summary>

清单模式将依赖描述文件 `vcpkg.json` 纳入版本控制（Git），能够确保每一位开发者在不同机器上克隆项目后，通过相同的构建指令自动下载并编译完全一致的依赖库。它消除了“我的电脑上能跑，你的电脑上跑不了”的配置不一问题。

</details>

### 题目 2：vcpkg.json 语法纠错

以下 `vcpkg.json` 存在一处致命拼写错误，请指出：
```json
{
  "name": "project_A",
  "version-string": "1.0.1",
  "dependency": ["zlib"]
}
```

<details>
<summary>查看答案</summary>

关键字应为 `dependencies`（复数形式），而非 `dependency`（单数形式）。

</details>

