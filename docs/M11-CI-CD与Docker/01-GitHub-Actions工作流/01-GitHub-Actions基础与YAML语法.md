# GitHub Actions 基础与 YAML 语法

> **所属模块：** M11-CI/CD 与 Docker
> **前置知识：** [M01-项目导览与环境搭建](../../M01-项目导览与环境搭建/README.md)、[M02-项目构建系统深度解析](../../M02-项目构建系统深度解析/README.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 CI/CD 的核心概念，解释为什么现代项目需要自动化流水线
2. 掌握 GitHub Actions 的架构模型（工作流→作业→步骤→Action 四层结构）
3. 熟练编写 YAML 格式的工作流配置文件
4. 区分并使用不同类型的触发器（push、pull_request、workflow_run、schedule 等）
5. 理解环境变量、密钥（Secrets）和上下文（Context）的使用场景
6. 能够从零编写一个包含检出代码、安装依赖、编译运行的完整工作流

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 持续集成 | Continuous Integration (CI) | 开发者频繁提交代码到主分支，每次提交自动触发构建和测试 |
| 持续交付 | Continuous Delivery (CD) | CI 的延伸——构建和测试通过后，代码随时可以一键发布到生产环境 |
| 工作流 | Workflow | 一个自动化流程的完整定义，用 YAML 文件存放在 `.github/workflows/` 目录 |
| 作业 | Job | 工作流中的独立执行单元，运行在一台虚拟机（Runner）上 |
| 步骤 | Step | 作业中的最小执行单位，一条命令或一个 Action 调用 |
| Runner | Runner | 执行作业的虚拟机或物理机，GitHub 提供免费的托管 Runner（ubuntu-latest、windows-latest 等） |
| Action | Action | 可复用的自动化组件，如 `actions/checkout@v4` 用于检出代码 |
| 触发器 | Trigger/Event | 决定工作流何时启动的条件，如代码推送、PR 创建、定时任务等 |
| 上下文 | Context | GitHub Actions 提供的内置变量集合，如 `github.repository`、`runner.os` 等 |
| 密钥 | Secret | 存储在 GitHub 仓库设置中的加密敏感信息（如 API Key），通过 `${{ secrets.NAME }}` 引用 |

---

## 第一步：理解 CI/CD 核心概念

### 什么是 CI/CD？

想象你和团队的 5 个同事同时在修改一个 C++ 项目。如果每个人都在自己的分支上闷头开发一个月，最后一起合并到主分支，几乎必然会遇到大量的合并冲突、编译错误、甚至运行时 Bug。这就是所谓的"集成地狱"（Integration Hell）。

**持续集成（CI, Continuous Integration）** 的核心思想很简单：**不要等到最后才合并，而是每天（甚至每次提交）都把代码合并到主分支，并自动运行构建和测试**。这样一来，问题在产生的几分钟内就能被发现，而不是积累一个月后才爆发。

**持续交付（CD, Continuous Delivery）** 是 CI 的自然延伸：既然每次提交都经过了自动化测试，那理论上每次提交都可以直接部署到生产环境。CD 确保从代码提交到产品发布的整条链路都是自动化的。

对于 KrKr2 项目而言，CI 体现为：

```
开发者推送代码到 main 分支
       ↓
GitHub Actions 自动触发
       ↓
┌──────────────────────┐
│ 1. 代码格式检查       │ ← 格式不对？直接拒绝
│    (clang-format)     │
└──────────┬───────────┘
           ↓ 通过
┌──────────────────────┐
│ 2a. Linux 构建       │
│ 2b. Windows 构建     │ ← 三平台并行构建
│ 2c. Android 构建     │
└──────────┬───────────┘
           ↓ 全部通过
    构建产物上传到 GitHub Artifacts
```

### CI/CD 工具生态

市面上有很多 CI/CD 工具，以下是常见选项的对比：

| 工具 | 托管方式 | 与 GitHub 集成 | 免费额度 | 适用场景 |
|------|----------|---------------|---------|---------|
| **GitHub Actions** | 云端（GitHub 托管） | 原生集成 | 2000 分钟/月（公开仓库无限） | GitHub 项目首选 |
| Jenkins | 自托管 | 需插件 | 完全免费（自建服务器） | 大型企业、高度定制 |
| GitLab CI | 云端/自托管 | GitLab 原生 | 400 分钟/月 | GitLab 项目 |
| CircleCI | 云端 | 需配置 | 6000 分钟/月 | 复杂流水线 |
| Travis CI | 云端 | 需配置 | 有限免费 | 开源项目（老牌） |

KrKr2 使用 GitHub Actions，原因很直接：代码托管在 GitHub，Actions 提供原生集成、免费额度充足、社区生态丰富。

---

## 第二步：GitHub Actions 架构模型

### 四层结构

GitHub Actions 的架构分为四个层级，从大到小依次是：

```
Workflow（工作流）
  └── Job（作业）
        └── Step（步骤）
              └── Action（动作）或 Shell 命令
```

用一个生活类比来理解：

- **Workflow** = 一条完整的流水线（比如"汽车装配线"）
- **Job** = 流水线上的工位（"焊接工位"、"喷漆工位"、"质检工位"）
- **Step** = 工位上的操作步骤（"拿起焊枪"、"对准接缝"、"开始焊接"）
- **Action** = 一个标准化的操作工具（"3M 焊枪"——任何工位都可以拿来用）

### Workflow（工作流）

工作流是 GitHub Actions 的顶层概念。每个工作流是一个 **YAML 文件**，存放在仓库的 `.github/workflows/` 目录下。一个仓库可以有多个工作流文件，每个文件定义一条独立的自动化流水线。

KrKr2 项目有 4 个工作流文件：

```
.github/workflows/
├── code-format-check.yml    # 代码格式检查（门禁）
├── build-linux.yml          # Linux 平台构建
├── build-windows.yml        # Windows 平台构建
└── build-android.yml        # Android 平台构建
```

### Job（作业）

一个工作流可以包含多个作业。**每个作业运行在一台独立的虚拟机（Runner）上**——这意味着不同作业之间的文件系统是隔离的。作业默认并行执行，但可以通过 `needs` 关键字声明依赖关系来串行化。

```yaml
jobs:
  format-check:          # 作业 1：检查格式
    runs-on: ubuntu-latest
    steps: [...]

  build:                 # 作业 2：构建（依赖作业 1）
    needs: format-check  # 等作业 1 完成后才开始
    runs-on: ubuntu-latest
    steps: [...]

  test:                  # 作业 3：测试（依赖作业 2）
    needs: build
    runs-on: ubuntu-latest
    steps: [...]
```

GitHub 提供的免费托管 Runner 有三种操作系统：

| Runner 标签 | 操作系统 | CPU/内存 | 适用场景 |
|-------------|---------|---------|---------|
| `ubuntu-latest` | Ubuntu 22.04 或 24.04 | 4 核 / 16 GB | Linux 构建、通用任务 |
| `windows-latest` | Windows Server 2022 | 4 核 / 16 GB | MSVC 构建、Windows 专属工具 |
| `macos-latest` | macOS 14 (Sonoma) | 3 核 / 14 GB | Xcode 构建、iOS/macOS |

### Step（步骤）

步骤是作业内部的执行单位。步骤有两种形式：

1. **运行 Shell 命令**：使用 `run` 关键字
2. **调用 Action**：使用 `uses` 关键字

```yaml
steps:
  # 形式 1：调用 Action
  - name: 检出代码             # 步骤名称（可选，用于日志显示）
    uses: actions/checkout@v4  # 调用官方 checkout Action v4 版本

  # 形式 2：运行 Shell 命令
  - name: 安装依赖
    run: |
      sudo apt-get update
      sudo apt-get install -y ninja-build
```

### Action（动作）

Action 是 GitHub Actions 生态的核心——它是一个**可复用的自动化组件**。Action 可以用 JavaScript 或 Docker 容器实现，发布到 GitHub Marketplace 供所有人使用。

KrKr2 项目中使用了以下 Action：

| Action | 版本 | 用途 |
|--------|------|------|
| `actions/checkout` | v4 | 检出仓库代码到 Runner |
| `actions/upload-artifact` | v4 | 上传构建产物 |
| `actions/cache` | v4.2.3 | 缓存 Android Command Line Tools |
| `DoozyX/clang-format-lint-action` | v0.20 | 检查 C++ 代码格式 |
| `ilammy/msvc-dev-cmd` | v1 | 设置 MSVC 编译环境 |
| `hendrikmuhs/ccache-action` | v1.2.18 | 设置 ccache 编译缓存 |
| `actions/setup-java` | v3 | 安装 JDK（Android 构建需要） |
| `gradle/actions/setup-gradle` | v4 | 设置 Gradle 构建工具 |
| `lukka/get-cmake` | latest | 安装指定版本的 CMake |

---

## 第三步：YAML 语法详解

GitHub Actions 的配置文件使用 YAML（YAML Ain't Markup Language，一种递归缩写，意为"YAML 不是标记语言"）格式。YAML 是一种以**缩进表示层级关系**的数据序列化格式，比 JSON 更易读。如果你熟悉 Python 的缩进规则，YAML 会感觉很亲切。

### 基本数据类型

```yaml
# 字符串（三种写法）
name: Code Format Check        # 不加引号（最常用）
message: "Hello, World!"       # 双引号（支持转义字符 \n \t 等）
path: 'cpp/**/*.cpp'           # 单引号（字面值，不解析转义）

# 数字
timeout: 30         # 整数
version: 3.31       # 浮点数

# 布尔值
verbose: true       # true / false（也支持 yes / no，但不推荐）

# null
value: null         # 或者 ~，或者直接不写值
```

### 列表（数组）

```yaml
# 块样式列表（每项以 - 开头，推荐）
branches:
  - main
  - develop
  - 'release/**'

# 流样式列表（类似 JSON，适合短列表）
extensions: ['cpp', 'cc', 'h', 'hpp', 'inc']
```

### 映射（对象/字典）

```yaml
# 块样式映射（缩进表示嵌套）
env:
  VCPKG_ROOT: /opt/vcpkg
  VCPKG_INSTALL_OPTIONS: "--debug"
  CMAKE_BUILD_TYPE: Release

# 嵌套映射
jobs:
  build-linux:
    runs-on: ubuntu-latest
    env:
      CC: gcc
      CXX: g++
```

### 多行字符串

这是 YAML 中最容易混淆的部分，GitHub Actions 中经常用到：

```yaml
# | 保留换行符（Literal Block Scalar）
# 每行原样保留，末尾自动加换行
run: |
  echo "第一行"
  echo "第二行"
  echo "第三行"
# 等价于字符串："echo \"第一行\"\necho \"第二行\"\necho \"第三行\"\n"

# > 折叠换行符（Folded Block Scalar）
# 换行变空格，空行变换行
description: >
  这是一段很长的描述，
  写在多行上会被折叠成一行，
  但空行会保留为段落分隔。

  这是第二段。
# 等价于字符串："这是一段很长的描述， 写在多行上会被折叠成一行， 但空行会保留为段落分隔。\n这是第二段。\n"

# |- 和 >- 去掉末尾换行
run: |-
  echo "没有末尾换行"
```

### 锚点与别名（DRY 技巧）

```yaml
# & 定义锚点，* 引用别名
defaults: &default-env
  VCPKG_ROOT: /opt/vcpkg
  CC: gcc
  CXX: g++

jobs:
  build-debug:
    env:
      <<: *default-env    # 合并锚点的所有键值对
      CMAKE_BUILD_TYPE: Debug

  build-release:
    env:
      <<: *default-env
      CMAKE_BUILD_TYPE: Release
```

> **注意：** GitHub Actions 的 YAML 解析器对锚点支持有限，跨 job 的锚点可能不生效。如果需要跨 job 复用配置，推荐使用 **Reusable Workflows**（可复用工作流）或 **Composite Actions**（组合 Action）。

### YAML 常见陷阱

| 陷阱 | 错误示例 | 正确示例 | 说明 |
|------|---------|---------|------|
| Tab 缩进 | `\t runs-on:` | `  runs-on:` | YAML 只允许空格缩进 |
| 冒号后无空格 | `name:value` | `name: value` | 冒号后必须有空格 |
| 特殊字符未引用 | `on: true` | `on: "true"` 或 `"on": true` | `on`、`true`、`yes` 等是 YAML 保留词 |
| 版本号解析 | `version: 3.10` | `version: "3.10"` | `3.10` 会被解析为浮点数 3.1 |

---

## 第四步：触发器（Trigger）详解

触发器定义了工作流**何时运行**，是 YAML 文件中 `on` 字段的内容。

### push 触发器

当代码推送到指定分支时触发：

```yaml
on:
  push:
    branches: [ main, develop ]  # 只在 main 和 develop 分支触发
    paths:                       # 只在以下路径有改动时触发
      - 'cpp/**'
      - 'platforms/**'
      - 'CMakeLists.txt'
    paths-ignore:                # 排除以下路径（与 paths 互斥，不能同时用）
      - 'docs/**'
      - '*.md'
```

KrKr2 的 `code-format-check.yml` 就使用了 `push` + `paths` 过滤：

```yaml
# .github/workflows/code-format-check.yml 第 6-17 行
on:
  push:
    branches: [ main ]
    paths:
      - 'cmake/**'
      - 'cpp/**'
      - 'platforms/**'
      - 'vcpkg/**'
      - 'CMakeLists.txt'
      - 'CMakePresets.json'
      - 'vcpkg.json'
      - 'vcpkg-configuration.json'
      - '.github/workflows/**'
```

这意味着：只有修改了 C++ 代码、CMake 配置、vcpkg 配置或工作流文件本身时，格式检查才会运行。修改文档（docs/）或 README 不会触发。

### pull_request 触发器

当 PR 创建或更新时触发：

```yaml
on:
  pull_request:
    branches: [ main ]           # PR 目标分支是 main
    types: [ opened, synchronize, reopened ]  # 可选，默认就是这三种
    paths:
      - 'cpp/**'
```

### workflow_run 触发器（链式触发）

这是 KrKr2 项目的核心触发机制——**一个工作流在另一个工作流完成后触发**：

```yaml
# build-linux.yml 第 6-8 行
on:
  workflow_run:
    workflows: ["Code Format Check"]  # 监听 "Code Format Check" 工作流
    types: [ "completed" ]            # 当它完成时触发（不管成功还是失败）
```

注意 `completed` 表示工作流执行完毕（包括成功和失败），所以还需要在 job 级别加条件：

```yaml
jobs:
  build-linux:
    # 只有当格式检查成功时才真正执行
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
```

这形成了 KrKr2 的链式触发流程：

```
push 到 main
    ↓
Code Format Check 启动
    ↓ 完成
build-linux.yml 触发  ─── if: conclusion == 'success' ─→ 执行构建
build-windows.yml 触发 ── if: conclusion == 'success' ─→ 执行构建
build-android.yml 触发 ── if: conclusion == 'success' ─→ 执行构建
```

### workflow_dispatch 触发器（手动触发）

允许在 GitHub 网页上手动点击 "Run workflow" 按钮触发：

```yaml
on:
  workflow_dispatch:
    inputs:                      # 可选：定义输入参数
      build_type:
        description: '构建类型'
        required: true
        default: 'Debug'
        type: choice
        options:
          - Debug
          - Release
      enable_tests:
        description: '是否启用测试'
        required: false
        type: boolean
        default: false
```

KrKr2 的所有 4 个工作流都支持 `workflow_dispatch`，方便开发者手动触发重新构建。

### schedule 触发器（定时任务）

使用 cron 表达式定义定时触发：

```yaml
on:
  schedule:
    # ┌───────────── 分钟 (0-59)
    # │ ┌───────────── 小时 (0-23, UTC 时间)
    # │ │ ┌───────────── 日 (1-31)
    # │ │ │ ┌───────────── 月 (1-12)
    # │ │ │ │ ┌───────────── 星期 (0-6, 0=周日)
    # │ │ │ │ │
    - cron: '0 2 * * 1'    # 每周一 UTC 02:00（北京时间 10:00）
    - cron: '30 5 1 * *'   # 每月 1 号 UTC 05:30
```

### 多触发器组合

一个工作流可以同时响应多种触发器：

```yaml
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:     # 同时支持手动触发
  schedule:
    - cron: '0 0 * * 0'  # 每周日 UTC 00:00 自动运行
```

---

## 第五步：环境变量与密钥

### 环境变量的三个层级

GitHub Actions 的环境变量可以在三个层级定义，从大到小：

```yaml
# 1. 工作流级别 — 所有 job 都能访问
env:
  VCPKG_BINARY_SOURCES: "clear;nuget,https://nuget.pkg.github.com/..."

jobs:
  build:
    # 2. 作业级别 — 只有该 job 的步骤能访问
    env:
      VCPKG_ROOT: ${{ github.workspace }}/.act/vcpkg

    steps:
      - name: 构建
        # 3. 步骤级别 — 只有该步骤能访问
        env:
          CMAKE_BUILD_TYPE: Release
        run: cmake --preset="Linux Release Config"
```

优先级：步骤级 > 作业级 > 工作流级（内层覆盖外层）。

### GitHub 上下文（Context）

上下文是 GitHub Actions 提供的一组**内置变量**，通过 `${{ <context> }}` 语法引用：

```yaml
# 常用上下文
${{ github.repository }}       # 仓库名，如 "jeffcwj/krkr2_angle"
${{ github.repository_owner }} # 仓库所有者，如 "jeffcwj"
${{ github.actor }}            # 触发者用户名
${{ github.workspace }}        # Runner 上的工作目录路径
${{ github.ref }}              # 分支/标签引用，如 "refs/heads/main"
${{ github.sha }}              # 提交 SHA
${{ github.event_name }}       # 触发事件名，如 "push"、"workflow_run"

${{ runner.os }}               # Runner 操作系统：Linux/Windows/macOS
${{ runner.arch }}             # Runner 架构：X64/ARM64

${{ env.VCPKG_ROOT }}          # 引用环境变量
${{ secrets.NUGET_API_KEY }}   # 引用密钥

${{ matrix.os }}               # 矩阵构建中的变量（下一节详讲）
```

### 密钥管理（Secrets）

敏感信息（API Key、密码、Token）不能写在工作流文件中（因为 YAML 文件会提交到 Git），而是存储在 GitHub 仓库的 **Settings → Secrets and variables → Actions** 中。

KrKr2 项目使用了 `NUGET_API_KEY` 密钥，用于 vcpkg 二进制缓存的 NuGet 认证：

```yaml
# build-linux.yml 第 76 行
-Password "${{ secrets.NUGET_API_KEY }}"
```

设置密钥的步骤：

1. 进入 GitHub 仓库页面
2. 点击 **Settings** → **Secrets and variables** → **Actions**
3. 点击 **New repository secret**
4. 填写名称（如 `NUGET_API_KEY`）和值
5. 在工作流中通过 `${{ secrets.NUGET_API_KEY }}` 引用

> **安全提示：** 密钥的值在日志中会被自动遮蔽为 `***`。但是，如果你在 `run` 步骤中用 `echo` 打印密钥，GitHub 可能无法完全遮蔽所有变形（如 base64 编码后的值）。永远不要故意在日志中输出密钥。

---

## 第六步：完整工作流示例——从零开始

下面我们从零编写一个完整的 GitHub Actions 工作流，以加深理解。这个工作流将：检出代码 → 安装 CMake → 配置并构建一个简单的 C++ 项目 → 运行可执行文件。

### 示例 1：最小工作流

```yaml
# .github/workflows/hello-ci.yml
name: Hello CI                    # 工作流名称（显示在 GitHub Actions 页面）

on:                                # 触发条件
  push:
    branches: [ main ]            # 只在 main 分支的 push 触发
  workflow_dispatch:               # 允许手动触发

jobs:                              # 定义作业
  hello:                           # 作业 ID（自定义名称，只能用字母、数字、-、_）
    runs-on: ubuntu-latest         # 运行环境

    steps:                         # 步骤列表
      - name: 打招呼
        run: echo "Hello from GitHub Actions! 当前时间：$(date)"

      - name: 显示系统信息
        run: |
          echo "操作系统：$(uname -a)"
          echo "CPU 核心数：$(nproc)"
          echo "内存：$(free -h | grep Mem | awk '{print $2}')"
          echo "磁盘：$(df -h / | tail -1 | awk '{print $4}') 可用"
```

### 示例 2：C++ 构建工作流

```yaml
# .github/workflows/cpp-build.yml
name: C++ Build

on:
  push:
    branches: [ main ]
    paths:
      - '**.cpp'
      - '**.h'
      - '**/CMakeLists.txt'
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      # 步骤 1：检出代码
      - name: 检出代码
        uses: actions/checkout@v4
        # checkout Action 会将仓库代码克隆到 $GITHUB_WORKSPACE 目录

      # 步骤 2：安装构建工具
      - name: 安装 CMake 和 Ninja
        run: |
          sudo apt-get update
          sudo apt-get install -y cmake ninja-build
          cmake --version
          ninja --version

      # 步骤 3：配置 CMake
      - name: 配置项目
        run: |
          cmake -B build \
            -G Ninja \
            -DCMAKE_BUILD_TYPE=Release
          # -B build      → 构建目录为 ./build
          # -G Ninja      → 使用 Ninja 生成器（比 Make 快）
          # -DCMAKE_BUILD_TYPE=Release → Release 模式

      # 步骤 4：编译
      - name: 编译
        run: cmake --build build --parallel $(nproc)
        # --parallel $(nproc) → 使用所有 CPU 核心并行编译

      # 步骤 5：运行测试（如果有的话）
      - name: 运行测试
        run: ctest --test-dir build --output-on-failure
```

### 示例 3：多平台构建

```yaml
# .github/workflows/multi-platform.yml
name: Multi-Platform Build

on:
  push:
    branches: [ main ]

jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        build_type: [Debug, Release]
      fail-fast: false    # 一个平台失败不影响其他平台继续构建

    runs-on: ${{ matrix.os }}
    name: ${{ matrix.os }} (${{ matrix.build_type }})

    steps:
      - uses: actions/checkout@v4

      - name: 安装 CMake (Linux)
        if: runner.os == 'Linux'
        run: sudo apt-get install -y cmake ninja-build

      - name: 安装 CMake (macOS)
        if: runner.os == 'macOS'
        run: brew install cmake ninja

      - name: 设置 MSVC (Windows)
        if: runner.os == 'Windows'
        uses: ilammy/msvc-dev-cmd@v1
        with:
          arch: amd64

      - name: 配置和构建
        run: |
          cmake -B build -DCMAKE_BUILD_TYPE=${{ matrix.build_type }}
          cmake --build build --config ${{ matrix.build_type }}
```

### 示例 4：带缓存的构建

```yaml
# .github/workflows/cached-build.yml
name: Cached Build

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # 缓存 vcpkg 安装的包
      - name: 缓存 vcpkg
        uses: actions/cache@v4
        with:
          path: |
            ~/.cache/vcpkg
            ${{ github.workspace }}/vcpkg_installed
          key: vcpkg-${{ runner.os }}-${{ hashFiles('vcpkg.json') }}
          restore-keys: |
            vcpkg-${{ runner.os }}-

      # 缓存编译中间文件（ccache）
      - name: ccache
        uses: hendrikmuhs/ccache-action@v1.2.18
        with:
          key: ${{ github.job }}

      - name: 构建
        run: |
          cmake -B build -DCMAKE_BUILD_TYPE=Release
          cmake --build build
```

### 示例 5：上传构建产物

```yaml
# .github/workflows/build-and-upload.yml
name: Build and Upload

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: 构建
        run: |
          cmake -B build -DCMAKE_BUILD_TYPE=Release
          cmake --build build

      # 上传构建产物
      - name: 上传构建结果
        if: success()          # 只有构建成功时才上传
        uses: actions/upload-artifact@v4
        with:
          name: my-app-linux   # 产物名称
          path: |              # 要上传的文件路径
            build/bin/**
            build/lib/**
          retention-days: 30   # 保留 30 天
```

---

## 动手实践

按照以下步骤在你自己的 GitHub 仓库中创建并运行一个工作流：

1. **创建工作流文件：** 在仓库根目录下创建 `.github/workflows/test-ci.yml`
2. **粘贴示例 1** 的内容（最小工作流）
3. **提交并推送** 到 main 分支
4. **查看运行结果：** 打开 GitHub 仓库 → 点击 **Actions** 标签页 → 查看运行日志
5. **手动触发：** 点击工作流名称 → 点击 **Run workflow** 按钮 → 再次查看日志
6. **修改为示例 2**（C++ 构建工作流），在仓库中添加一个简单的 `CMakeLists.txt` 和 `main.cpp`，推送后观察构建结果

## 对照项目源码

KrKr2 项目的工作流文件位于以下位置：

- `.github/workflows/code-format-check.yml` 第 1-40 行 — 使用 `push` + `pull_request` + `workflow_dispatch` 三种触发器，`paths` 过滤只监听代码文件变更
- `.github/workflows/build-linux.yml` 第 1-12 行 — 使用 `workflow_run` 链式触发 + 工作流级别 `env` 定义 vcpkg 二进制缓存源
- `.github/workflows/build-windows.yml` 第 1-12 行 — 同样的 `workflow_run` 触发模式
- `.github/workflows/build-android.yml` 第 1-13 行 — 额外定义了 `ANDROID_NDK_VERSION` 环境变量
- `.github/workflows/build-linux.yml` 第 67-79 行 — 使用 `${{ secrets.NUGET_API_KEY }}` 引用密钥配置 NuGet

## 常见错误与解决方案

### 错误 1：YAML 缩进错误

```
Error: .github/workflows/test.yml (Line: 12, Col: 3):
  While parsing a block mapping, did not find expected key
```

**原因：** 使用了 Tab 缩进或缩进层级不一致。
**解决：** 统一使用 2 个空格缩进（GitHub Actions 的惯例）。使用支持 YAML 的编辑器（VS Code 安装 YAML 扩展）可以自动检测。

### 错误 2：工作流未触发

**症状：** 推送代码后 Actions 页面没有新的运行。
**排查步骤：**
1. 检查 `on.push.branches` 是否包含你推送的分支名
2. 检查 `on.push.paths` 是否包含你修改的文件路径
3. 确认工作流文件位于 `.github/workflows/` 目录（注意 `.github` 有个点号前缀）
4. 确认 YAML 文件没有语法错误（在 GitHub 网页上编辑时有实时语法检查）

### 错误 3：Secret 引用为空

```
Error: The template is not valid. 'secrets.MY_KEY' is not defined
```

**原因：** 密钥未在仓库 Settings 中设置，或者名称拼写错误（区分大小写）。
**解决：** 进入 Settings → Secrets and variables → Actions，确认密钥名称完全匹配。

## 本节小结

- CI/CD 是现代软件工程的核心实践，**持续集成**通过频繁合并代码和自动化测试来尽早发现问题
- GitHub Actions 采用**四层架构**：Workflow → Job → Step → Action，每一层有明确的职责
- 工作流配置使用 **YAML 格式**，关键语法包括缩进层级、列表（`-`）、映射（`key: value`）、多行字符串（`|` 和 `>`）
- **触发器**决定工作流何时运行，KrKr2 使用 `push`、`pull_request`、`workflow_run`（链式触发）和 `workflow_dispatch`（手动触发）
- 环境变量分三个层级（工作流/作业/步骤），内层覆盖外层
- 密钥通过 `${{ secrets.NAME }}` 引用，存储在仓库设置中，日志中自动遮蔽
- GitHub 提供免费的托管 Runner（ubuntu-latest、windows-latest、macos-latest），开源项目无分钟限制

## 练习题与答案

### 题目 1：编写一个每天自动运行的工作流

编写一个 GitHub Actions 工作流文件，要求：
- 每天北京时间 8:00 自动运行（提示：北京时间 = UTC + 8）
- 同时支持手动触发
- 打印当前日期和 Git 最新提交信息

<details>
<summary>查看答案</summary>

```yaml
# .github/workflows/daily-report.yml
name: Daily Report

on:
  schedule:
    # 北京时间 8:00 = UTC 0:00
    - cron: '0 0 * * *'
  workflow_dispatch:

jobs:
  report:
    runs-on: ubuntu-latest

    steps:
      - name: 检出代码
        uses: actions/checkout@v4
        with:
          fetch-depth: 5    # 获取最近 5 个提交

      - name: 打印报告
        run: |
          echo "=== 每日报告 ==="
          echo "日期：$(date -u '+%Y-%m-%d %H:%M:%S UTC')"
          echo "北京时间：$(TZ='Asia/Shanghai' date '+%Y-%m-%d %H:%M:%S')"
          echo ""
          echo "=== 最近 5 个提交 ==="
          git log --oneline -5
          echo ""
          echo "=== 仓库统计 ==="
          echo "文件数：$(find . -type f | wc -l)"
          echo "代码行数（.cpp）：$(find . -name '*.cpp' -exec cat {} + 2>/dev/null | wc -l)"
```

关键点：
- `cron: '0 0 * * *'` 表示 UTC 00:00，即北京时间 08:00
- `fetch-depth: 5` 限制克隆深度，加速检出
- 同时声明 `schedule` 和 `workflow_dispatch` 实现定时+手动双触发

</details>

### 题目 2：解释 KrKr2 的链式触发机制

请画出 KrKr2 项目 4 个工作流文件之间的触发关系图，并解释为什么使用 `workflow_run` 而不是直接在一个工作流中定义多个 job。

<details>
<summary>查看答案</summary>

**触发关系图：**

```
开发者 push 到 main 分支
         │
         ▼
┌─────────────────────────┐
│  Code Format Check      │ ← push + pull_request + workflow_dispatch 触发
│  (code-format-check.yml)│
└────────────┬────────────┘
             │ completed
    ┌────────┼────────┐
    ▼        ▼        ▼
┌────────┐ ┌────────┐ ┌────────┐
│ Build  │ │ Build  │ │ Build  │
│ Linux  │ │Windows │ │Android │
│        │ │        │ │        │
│if:     │ │if:     │ │if:     │
│success │ │success │ │success │
└────────┘ └────────┘ └────────┘
```

**使用 `workflow_run` 而非单工作流多 job 的原因：**

1. **不同 Runner 类型：** Linux 和 Android 构建需要 `ubuntu-latest`，Windows 构建需要 `windows-latest`。虽然单工作流的多 job 也支持不同 Runner，但分文件更清晰
2. **独立维护：** 每个平台的构建步骤差异很大（Linux 需要 apt 安装依赖、Windows 需要 MSVC 和 winflexbison、Android 需要 SDK/NDK），分成独立文件更容易维护和调试
3. **门禁隔离：** `workflow_run` 允许先运行格式检查，只有格式检查通过后才启动构建。如果格式检查失败，三个构建工作流虽然被触发但会因为 `if: conclusion == 'success'` 而跳过，节省 CI 资源
4. **独立重试：** 如果 Android 构建失败但 Linux 和 Windows 成功，可以单独重新运行 Android 构建工作流，而不需要重新运行所有平台

</details>

### 题目 3：修复以下工作流中的 3 个错误

```yaml
name: Broken Workflow

on:
	push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu
    steps:
      - uses: actions/checkout
      - name: 构建
        run:
          cmake -B build
          cmake --build build
```

<details>
<summary>查看答案</summary>

**错误 1：** 第 4 行使用了 Tab 缩进（`\t push:`），YAML 不允许 Tab。
**修复：** 改为 2 个空格缩进。

**错误 2：** 第 9 行 `runs-on: ubuntu` 不是有效的 Runner 标签。
**修复：** 改为 `runs-on: ubuntu-latest`（或 `ubuntu-22.04`、`ubuntu-24.04`）。

**错误 3：** 第 11 行 `uses: actions/checkout` 缺少版本标签。虽然技术上可以运行（默认使用最新版），但这是不安全的做法——可能因 Action 更新而突然失败。
**修复：** 改为 `uses: actions/checkout@v4`。

**错误 4（额外）：** 第 13-15 行 `run:` 后面的多行命令没有使用 `|` 管道符号，YAML 会将两行视为映射键而非命令。
**修复：** 改为 `run: |`。

修复后的完整工作流：

```yaml
name: Fixed Workflow

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: 构建
        run: |
          cmake -B build
          cmake --build build
```

</details>

## 下一步

[项目 CI 流水线解析](./02-项目CI流水线解析.md) — 逐行解读 KrKr2 项目的 4 个 GitHub Actions 工作流文件，理解完整的 CI 流水线设计。
