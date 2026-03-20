# M11 — CI/CD 与 Docker

> **前置知识：** [M01-项目导览与环境搭建](../M01-项目导览与环境搭建/README.md)、[M02-项目构建系统深度解析](../M02-项目构建系统深度解析/README.md)
> **难度：** ★★☆（中等）
> **预计总学习时间：** 约 6-8 小时

## 模块简介

当你在本地完成代码修改后，如何确保代码质量、自动化构建和持续交付？这就是 CI/CD（持续集成/持续交付，Continuous Integration/Continuous Delivery）要解决的问题。CI/CD 是现代软件工程的核心实践——每次代码推送都会自动触发一系列检查和构建流程，确保新代码不会破坏已有功能。

KrKr2 项目使用 **GitHub Actions**（GitHub 提供的免费 CI/CD 平台）作为自动化引擎，配合 **Docker**（一种轻量级容器化技术，把构建环境打包成可复现的"镜像"）提供一致的构建环境。项目的 CI 流水线包含代码格式检查（clang-format 门禁）、三平台自动构建（Linux/Windows/Android）、以及 vcpkg 依赖缓存等完整的自动化体系。

本模块将带你从 CI/CD 基础概念出发，逐行解读项目的 4 个 GitHub Actions 工作流文件和 2 个 Dockerfile，最后深入代码质量门禁（clang-format 格式化、clang-tidy 静态分析、vcpkg 二进制缓存）的实现细节。学完本模块后，你将能够独立修改和扩展项目的 CI 配置，甚至为项目添加新的自动化流程。

## 术语预览

本模块将涉及以下核心术语，先有个印象，各章正文会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 持续集成 | Continuous Integration (CI) | 开发者频繁地将代码合并到主分支，每次合并都自动运行测试和构建来尽早发现问题 |
| 持续交付 | Continuous Delivery (CD) | 在 CI 基础上，让代码随时可以一键部署到生产环境（本项目主要关注 CI 部分） |
| 工作流 | Workflow | GitHub Actions 中的自动化流程定义，用 YAML 文件描述，存放在 `.github/workflows/` 目录 |
| 作业 | Job | 工作流中的一个独立执行单元，运行在一台虚拟机上，包含多个步骤（Step） |
| 步骤 | Step | 作业中的最小执行单位，可以是运行一条 shell 命令，也可以是调用一个 Action |
| Action | Action | GitHub 社区或官方提供的可复用自动化组件，如 `actions/checkout@v4`（检出代码）|
| 触发器 | Trigger | 决定工作流何时运行的事件，如 `push`（代码推送）、`pull_request`（PR 创建）、`workflow_run`（其他工作流完成后） |
| 门禁检查 | Gate Check | CI 中的一种拦截机制——如果检查不通过，后续流程不会执行（如格式检查失败则不构建） |
| 容器 | Container | Docker 创建的轻量级隔离运行环境，共享宿主机内核但有独立的文件系统和进程空间 |
| 镜像 | Image | Docker 容器的只读模板，包含运行环境的完整快照（操作系统 + 依赖 + 工具链） |
| Dockerfile | Dockerfile | 定义如何构建 Docker 镜像的文本文件，包含一系列指令（FROM、RUN、COPY 等） |
| 多阶段构建 | Multi-stage Build | Dockerfile 中使用多个 `FROM` 指令，前一阶段安装工具，后一阶段只复制需要的产物，减小最终镜像体积 |
| 缓存挂载 | Cache Mount | Docker BuildKit 的 `--mount=type=cache` 特性，让构建缓存在多次构建间持久化，加速重复构建 |
| clang-format | clang-format | LLVM 项目提供的 C/C++ 代码格式化工具，根据配置文件自动统一代码风格 |
| clang-tidy | clang-tidy | LLVM 项目提供的 C/C++ 静态分析工具，能检测 bug、代码规范问题和现代化建议 |
| 二进制缓存 | Binary Caching | vcpkg 的加速机制，将编译好的依赖包缓存到远程存储（如 NuGet），避免每次 CI 都重新编译依赖 |
| NuGet | NuGet | .NET 生态的包管理器，vcpkg 借用它的协议来存储和分发预编译的 C++ 依赖包 |

## 章节目录

| 章 | 主题 | 小节数 | 主要内容 |
|----|------|--------|----------|
| 01 | [GitHub Actions 工作流](./01-GitHub-Actions工作流/) | 3 | GitHub Actions 基础概念与 YAML 语法、项目 4 个工作流文件逐行解读、自定义工作流与矩阵构建实战 |
| 02 | [Docker 构建环境](./02-Docker构建环境/) | 3 | Docker 基础概念与 Dockerfile 语法详解、项目两个 Dockerfile 逐行解析（多阶段构建 + 缓存挂载）、Docker 本地构建与调试实战 |
| 03 | [代码质量门禁](./03-代码质量门禁/) | 3 | clang-format 代码格式化（配置文件 55 个选项逐一解读）、clang-tidy 静态分析（142 条检查规则分类详解）、vcpkg 二进制缓存与 NuGet 集成 |

### 各节详细内容

#### 第一章：GitHub Actions 工作流

| 节 | 文件 | 内容概要 |
|----|------|----------|
| 01 | [GitHub Actions 基础与 YAML 语法](./01-GitHub-Actions工作流/01-GitHub-Actions基础与YAML语法.md) | CI/CD 核心概念；GitHub Actions 架构（工作流→作业→步骤→Action）；YAML 语法详解；触发器类型（push/PR/workflow_run/workflow_dispatch/schedule）；环境变量与密钥管理 |
| 02 | [项目 CI 流水线解析](./01-GitHub-Actions工作流/02-项目CI流水线解析.md) | 逐行解读 `code-format-check.yml`（40 行）、`build-linux.yml`（100 行）、`build-windows.yml`（84 行）、`build-android.yml`（169 行）；workflow_run 链式触发机制；四工作流依赖关系图 |
| 03 | [自定义工作流与矩阵构建](./01-GitHub-Actions工作流/03-自定义工作流与矩阵构建.md) | 矩阵构建（Matrix Strategy）实战；条件执行（if 表达式）；缓存策略（actions/cache + ccache）；构建产物上传（upload-artifact）；从零编写自定义工作流 |

#### 第二章：Docker 构建环境

| 节 | 文件 | 内容概要 |
|----|------|----------|
| 01 | [Docker 基础与 Dockerfile 语法](./02-Docker构建环境/01-Docker基础与Dockerfile语法.md) | 容器化核心概念（镜像 vs 容器 vs 仓库）；Docker 安装（四平台）；Dockerfile 指令详解（FROM/RUN/COPY/WORKDIR/ENV/ARG/LABEL/ENTRYPOINT/CMD）；构建上下文；镜像分层原理 |
| 02 | [项目 Dockerfile 解析](./02-Docker构建环境/02-项目Dockerfile解析.md) | 逐行解读 `linux.Dockerfile`（62 行）和 `android.Dockerfile`（67 行）；多阶段构建（`AS build-tools` → `AS generate`）；BuildKit 缓存挂载（`--mount=type=cache`）；vcpkg 容器内安装策略 |
| 03 | [Docker 实战：本地构建与调试](./02-Docker构建环境/03-Docker实战-本地构建与调试.md) | 用 Docker 在本地复现 CI 构建环境；调试构建失败的容器；自定义 Dockerfile 添加测试支持；Docker Compose 多容器编排；镜像优化（减小体积、加速构建） |

#### 第三章：代码质量门禁

| 节 | 文件 | 内容概要 |
|----|------|----------|
| 01 | [clang-format 代码格式化](./03-代码质量门禁/01-clang-format代码格式化.md) | clang-format 工作原理；项目 `.clang-format` 配置文件 55 个选项逐一解读；`DisableFormat: true` 豁免机制；本地使用（命令行 + IDE 集成）；CI 中的 `DoozyX/clang-format-lint-action` |
| 02 | [clang-tidy 静态分析](./03-代码质量门禁/02-clang-tidy静态分析.md) | 静态分析原理（AST 级别检查 vs 编译器警告）；项目 `.clang-tidy` 配置的 142 条规则分 8 大类详解（bugprone/cert/cppcoreguidelines/modernize/performance/readability 等）；本地运行与 CI 集成 |
| 03 | [vcpkg 二进制缓存与依赖管理](./03-代码质量门禁/03-vcpkg二进制缓存与依赖管理.md) | vcpkg 二进制缓存原理（哈希 → 查找 → 命中/未命中）；NuGet 协议集成；`VCPKG_BINARY_SOURCES` 环境变量；GitHub Packages 作为缓存后端；Mono 在 Linux CI 中的作用；overlay ports 与自定义 triplet |

## 学习路线

```
推荐阅读顺序：

01-GitHub-Actions工作流/
  ├── 01-GitHub-Actions基础与YAML语法     ← 从这里开始
  ├── 02-项目CI流水线解析                  ← 对照项目实际工作流
  └── 03-自定义工作流与矩阵构建            ← 动手编写

02-Docker构建环境/
  ├── 01-Docker基础与Dockerfile语法        ← Docker 入门
  ├── 02-项目Dockerfile解析                ← 对照项目实际 Dockerfile
  └── 03-Docker实战-本地构建与调试          ← 动手操作

03-代码质量门禁/
  ├── 01-clang-format代码格式化            ← 代码风格统一
  ├── 02-clang-tidy静态分析               ← 代码质量提升
  └── 03-vcpkg二进制缓存与依赖管理         ← 依赖加速
```

## 项目相关文件速查

| 文件 | 位置 | 说明 |
|------|------|------|
| 格式检查工作流 | `.github/workflows/code-format-check.yml` | 40 行，clang-format 20 门禁检查 |
| Linux 构建工作流 | `.github/workflows/build-linux.yml` | 100 行，ubuntu-latest + vcpkg + ccache |
| Windows 构建工作流 | `.github/workflows/build-windows.yml` | 84 行，windows-latest + MSVC + vcpkg |
| Android 构建工作流 | `.github/workflows/build-android.yml` | 169 行，ubuntu-latest + Android SDK/NDK + Gradle |
| Linux Dockerfile | `dockers/linux.Dockerfile` | 62 行，多阶段构建 + BuildKit 缓存 |
| Android Dockerfile | `dockers/android.Dockerfile` | 67 行，Android SDK/NDK + Gradle 构建 |
| clang-format 配置 | `.clang-format` | 55 行，LLVM 风格 + 自定义选项 |
| clang-tidy 配置 | `.clang-tidy` | 142 行，8 大类检查规则 |
| 格式化豁免 | `cpp/external/.clang-format`、`cpp/core/plugin/.clang-format` | `DisableFormat: true` |
| vcpkg 清单 | `vcpkg.json` | 依赖声明 |
| vcpkg overlay ports | `vcpkg/` | 自定义端口和 triplet |
