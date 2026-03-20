# Docker 基础与 Dockerfile 语法

> **所属模块：** M11-CI/CD与Docker
> **前置知识：** [GitHub Actions 基础与 YAML 语法](../01-GitHub-Actions工作流/01-GitHub-Actions基础与YAML语法.md)
> **预计阅读时间：** 40 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 Docker 的核心概念（镜像、容器、层、Registry）及其在 C++ 项目中的价值
2. 掌握 Dockerfile 的完整指令集，能独立编写多阶段构建的 Dockerfile
3. 理解镜像层（Layer）的缓存机制，并应用最佳实践优化构建速度
4. 使用 BuildKit 的高级特性（缓存挂载、Secret 挂载）提升构建效率
5. 针对 C++ 项目的特殊需求（大型编译、交叉编译）编写高效的 Dockerfile

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 镜像 | Image | 一个只读的文件系统快照，包含了运行程序所需的所有文件、库和配置 |
| 容器 | Container | 镜像的运行实例——在镜像上加了一个可写层，形成一个隔离的运行环境 |
| 层 | Layer | 镜像由多层只读文件系统叠加而成，每条 Dockerfile 指令生成一层 |
| Dockerfile | Dockerfile | 描述如何构建镜像的文本文件，包含一系列按顺序执行的指令 |
| BuildKit | BuildKit | Docker 的新一代构建引擎，支持并行构建、缓存挂载等高级特性 |
| 多阶段构建 | Multi-stage Build | 在一个 Dockerfile 中使用多个 `FROM` 指令，每个阶段可以从前一阶段复制产物，最终镜像只包含需要的文件 |
| 缓存挂载 | Cache Mount | BuildKit 特性，允许在构建过程中挂载持久化的缓存目录，避免重复下载依赖 |
| Registry | Registry | 存储和分发 Docker 镜像的服务，如 Docker Hub、GitHub Container Registry |

## Docker 核心概念

### 为什么 C++ 项目需要 Docker

C++ 项目的构建环境搭建是出了名的复杂。以 KrKr2 为例，Linux 构建需要：GCC/Clang、CMake 3.28+、Ninja、vcpkg、libX11-dev、libGL-dev、Mono（用于 NuGet）、ccache 等十几个工具和库。不同开发者的机器上这些工具的版本可能不同，导致"在我这里能编译"的经典问题。

Docker 通过**容器化（Containerization）** 解决这个问题：把整个构建环境（操作系统 + 工具链 + 依赖库）打包成一个**镜像（Image）**，任何人只要安装了 Docker，就能在完全相同的环境中构建代码。

Docker 在 C++ 项目中的典型应用场景：

```
┌─────────────────────────────────────────────────────────────┐
│                    Docker 在 C++ 项目中的角色                 │
├─────────────────┬───────────────────────────────────────────┤
│ 场景            │ 说明                                       │
├─────────────────┼───────────────────────────────────────────┤
│ 一致的构建环境   │ 所有开发者和 CI 使用完全相同的编译器/工具版本 │
│ 交叉编译        │ 在 Linux 容器中为 Android/ARM 构建           │
│ CI/CD 加速      │ 预装工具链的镜像省去每次安装依赖的时间         │
│ 多版本测试      │ 同时用 GCC 12、GCC 13、Clang 17 等测试       │
│ 隔离构建        │ 不污染宿主机环境，随时丢弃重建               │
└─────────────────┴───────────────────────────────────────────┘
```

### 镜像与容器

**镜像（Image）** 是一个只读的文件系统模板。你可以把它想象成一个"系统安装盘"——它包含了操作系统的文件、安装好的工具、配置好的环境变量，但它本身不能运行。

**容器（Container）** 是镜像的运行实例。当你用 `docker run` 启动一个镜像时，Docker 会在镜像的只读层上面加一个可写层（Copy-on-Write Layer），形成一个隔离的运行环境。容器销毁后，可写层的修改也随之消失（除非你主动保存）。

```
镜像与容器的关系：

    镜像（只读模板）              容器（运行实例）
   ┌──────────────┐          ┌──────────────────┐
   │  Ubuntu 22.04 │          │ 可写层（临时修改） │ ← docker run 时创建
   │  + GCC 12     │  ──→    ├──────────────────┤
   │  + CMake 3.31 │          │  Ubuntu 22.04     │
   │  + vcpkg      │          │  + GCC 12         │ ← 来自镜像（只读）
   └──────────────┘          │  + CMake 3.31     │
                              │  + vcpkg          │
   一个镜像可以创建            └──────────────────┘
   多个容器实例                 容器销毁后可写层消失
```

### 镜像层（Layer）

Docker 镜像不是一个单一的大文件，而是由多个**层（Layer）** 叠加而成。Dockerfile 中的每条指令（`FROM`、`RUN`、`COPY` 等）都会生成一个新层。层是只读的、可共享的、可缓存的：

```
Dockerfile 指令          生成的镜像层
─────────────────       ─────────────────
FROM ubuntu:22.04   →   Layer 1: Ubuntu 基础文件系统（~77MB）
RUN apt-get update  →   Layer 2: 更新的包索引（~30MB）
RUN apt-get install →   Layer 3: 安装的工具（~200MB）
COPY . /src         →   Layer 4: 复制的源代码（~50MB）
RUN cmake --build   →   Layer 5: 编译产物（~100MB）
                        ─────────────────
                        总计：~457MB
```

**层缓存的工作原理：** Docker 在构建时会检查每一层的输入是否与之前构建时相同。如果相同（指令文本一样、`COPY` 的文件内容一样），就直接复用缓存的层，不需要重新执行。一旦某一层的缓存失效，该层及其之后的所有层都需要重新构建：

```
第二次构建（只改了源代码）：

Layer 1: FROM ubuntu:22.04      → ✅ 缓存命中（不变）
Layer 2: RUN apt-get update      → ✅ 缓存命中（不变）
Layer 3: RUN apt-get install     → ✅ 缓存命中（不变）
Layer 4: COPY . /src             → ❌ 缓存失效（源代码变了）
Layer 5: RUN cmake --build       → ❌ 必须重新构建（前一层失效）
```

这就是为什么 Dockerfile 的指令顺序非常重要——**变化频率低的指令要放在前面，变化频率高的放在后面**，这样前面的层可以尽可能多地复用缓存。

## Dockerfile 指令详解

### FROM——指定基础镜像

`FROM` 是每个 Dockerfile 的第一条指令（注释和 `ARG` 除外），指定构建的基础镜像：

```dockerfile
# 示例 1：基本的 FROM 用法
# 使用 Ubuntu 22.04 作为基础镜像
FROM ubuntu:22.04

# 使用特定的 Debian 版本
FROM debian:bookworm-slim

# 使用 Alpine Linux（极小的基础镜像，约 5MB）
FROM alpine:3.19

# 使用特定的 GCC 编译器镜像（预装了 GCC）
FROM gcc:13

# 使用空镜像（scratch）——用于部署纯静态链接的二进制
FROM scratch
```

常用的 C++ 构建基础镜像对比：

| 基础镜像 | 大小 | 包管理器 | 适用场景 |
|----------|------|----------|----------|
| `ubuntu:22.04` | ~77MB | apt | 通用，生态最完整 |
| `debian:bookworm-slim` | ~80MB | apt | 类似 Ubuntu，更精简 |
| `alpine:3.19` | ~5MB | apk | 极小镜像，但 musl libc 可能有兼容问题 |
| `gcc:13` | ~1.2GB | apt | 预装 GCC，省去安装步骤 |
| `scratch` | 0MB | 无 | 仅放静态二进制，最终部署用 |

KrKr2 项目的两个 Dockerfile 都使用 `ubuntu:22.04`，因为它提供了最完整的工具链支持和库兼容性。

### RUN——执行命令

`RUN` 在构建时执行 Shell 命令，是安装工具和配置环境的主力指令。每条 `RUN` 指令生成一个新层：

```dockerfile
# 示例 2：RUN 的基本用法

# 方式 1：Shell 形式（通过 /bin/sh -c 执行）
RUN apt-get update && apt-get install -y gcc g++ cmake

# 方式 2：Exec 形式（直接执行，不经过 Shell 解析）
RUN ["apt-get", "install", "-y", "gcc"]

# 方式 3：多行命令（用 \ 换行，&& 连接——推荐写法）
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        gcc \
        g++ \
        cmake \
        ninja-build \
        git \
        curl \
    && rm -rf /var/lib/apt/lists/*    # 清理缓存，减小镜像体积
```

**最佳实践：合并 RUN 指令减少层数。** 每条 `RUN` 都创建一层。如果在一条 `RUN` 中安装了包，在下一条 `RUN` 中删除缓存，删除操作不会真正减小镜像——因为安装时的层已经保存了那些文件。应该在同一条 `RUN` 中完成安装和清理：

```dockerfile
# ❌ 不好：2 层，第一层包含了 apt 缓存
RUN apt-get update && apt-get install -y gcc
RUN rm -rf /var/lib/apt/lists/*    # 这个清理在新层中，不减小总镜像大小

# ✅ 好：1 层，缓存在同一层中被清除
RUN apt-get update \
    && apt-get install -y --no-install-recommends gcc \
    && rm -rf /var/lib/apt/lists/*
```

### COPY 与 ADD——复制文件到镜像

`COPY` 将文件或目录从构建上下文（通常是项目根目录）复制到镜像中。`ADD` 功能类似但有额外能力（解压 tar、远程 URL），但官方推荐在大多数情况下使用更简单明确的 `COPY`：

```dockerfile
# 示例 3：COPY 与 ADD 的用法

# 复制单个文件
COPY CMakeLists.txt /workspace/CMakeLists.txt

# 复制整个目录
COPY cpp/ /workspace/cpp/

# 复制多个文件到同一目标目录（目标以 / 结尾）
COPY CMakeLists.txt CMakePresets.json vcpkg.json /workspace/

# 使用通配符
COPY *.cmake /workspace/cmake/

# 改变文件所有者（--chown 仅 Linux 有效）
COPY --chown=1000:1000 . /workspace/

# ADD 可以自动解压 tar 文件
ADD toolchain.tar.gz /opt/    # 自动解压到 /opt/

# ADD 可以从 URL 下载（但推荐用 RUN curl 替代，可以控制缓存）
ADD https://example.com/file.txt /opt/file.txt
```

**COPY vs ADD 选择建议：**

| 功能 | COPY | ADD |
|------|------|-----|
| 复制本地文件 | ✅ | ✅ |
| 自动解压 tar.gz | ❌ | ✅ |
| 从 URL 下载 | ❌ | ✅ |
| 语义明确性 | ✅ 推荐 | ⚠️ 有隐式行为 |

**实际建议：** 始终用 `COPY`，除非你明确需要自动解压。下载文件用 `RUN curl` 或 `RUN wget`——这样可以在同一个 `RUN` 中下载、验证哈希、安装、清理。

### WORKDIR——设置工作目录

`WORKDIR` 设置后续 `RUN`、`CMD`、`COPY` 等指令的工作目录。如果目录不存在会自动创建：

```dockerfile
# 示例 4：WORKDIR 用法
FROM ubuntu:22.04

# 设置工作目录——后续指令都在此目录下执行
WORKDIR /workspace

# 这里的 . 指的是 /workspace
COPY . .

# 这条 RUN 在 /workspace 中执行
RUN cmake -B build

# 可以多次切换
WORKDIR /workspace/build
RUN cmake --build .

# 相对路径会基于上一个 WORKDIR
WORKDIR subdir    # 实际路径：/workspace/build/subdir
```

### ENV——设置环境变量

`ENV` 设置在构建时和运行时都生效的环境变量：

```dockerfile
# 示例 5：ENV 用法
FROM ubuntu:22.04

# 单个变量
ENV VCPKG_ROOT=/opt/vcpkg

# 多个变量（每个 ENV 一层，多个变量可合并）
ENV CC=gcc \
    CXX=g++ \
    CMAKE_BUILD_TYPE=Release

# 在后续指令中引用
RUN echo "vcpkg root: $VCPKG_ROOT"
WORKDIR $VCPKG_ROOT

# 使用 ARG + ENV 组合——ARG 用于构建时参数，ENV 持久化到运行时
ARG CMAKE_VERSION=3.31.6
ENV CMAKE_VERSION=$CMAKE_VERSION
RUN echo "将安装 CMake $CMAKE_VERSION"
```

### ARG——构建参数

`ARG` 定义只在构建时可用的变量（不会保存到最终镜像中），可以在 `docker build --build-arg` 时覆盖：

```dockerfile
# 示例 6：ARG 构建参数
# ARG 在 FROM 之前声明——可以影响基础镜像选择
ARG UBUNTU_VERSION=22.04
FROM ubuntu:${UBUNTU_VERSION}

# FROM 之后需要重新声明 ARG（FROM 会重置 ARG 作用域）
ARG CMAKE_VERSION=3.31.6
ARG NINJA_VERSION=1.12.1

RUN echo "安装 CMake ${CMAKE_VERSION} 和 Ninja ${NINJA_VERSION}"

# 构建时覆盖：
# docker build --build-arg CMAKE_VERSION=3.30.0 --build-arg UBUNTU_VERSION=24.04 .
```

### CMD 与 ENTRYPOINT——容器启动命令

`CMD` 定义容器启动时默认执行的命令。`ENTRYPOINT` 定义容器的"入口点"——更不容易被 `docker run` 的参数覆盖：

```dockerfile
# 示例 7：CMD 和 ENTRYPOINT
FROM ubuntu:22.04
WORKDIR /workspace

# CMD 方式：容器启动时默认执行的命令，可被 docker run 的参数覆盖
CMD ["cmake", "--build", "build"]
# docker run myimage                → 执行 cmake --build build
# docker run myimage bash           → 执行 bash（CMD 被覆盖）

# ENTRYPOINT 方式：固定的入口点，docker run 的参数会追加到后面
ENTRYPOINT ["cmake"]
CMD ["--help"]
# docker run myimage                → 执行 cmake --help
# docker run myimage --build build  → 执行 cmake --build build（CMD 被替换）

# 实际 C++ 项目中常见的模式：
# 构建镜像——默认执行构建脚本
COPY scripts/build-linux.sh /build.sh
RUN chmod +x /build.sh
ENTRYPOINT ["/build.sh"]
```

## 多阶段构建（Multi-stage Build）

### 为什么需要多阶段构建

C++ 项目的构建环境通常很大——GCC、CMake、vcpkg、头文件、静态库加起来可能有好几个 GB。但最终需要部署的只是编译出来的二进制文件（通常只有几十 MB）。如果把所有构建工具都打包在最终镜像中，不仅浪费存储空间，还会增加安全攻击面（多余的工具可能有漏洞）。

多阶段构建（Multi-stage Build）允许你在同一个 Dockerfile 中定义多个阶段，每个阶段有独立的 `FROM` 基础镜像。前面的阶段负责编译，最后一个阶段只从前面的阶段中复制需要的产物：

```dockerfile
# 示例 8：多阶段构建——典型的 C++ 项目模式
# ========== 第一阶段：构建环境 ==========
FROM ubuntu:22.04 AS builder

# 安装所有构建工具（这些不会出现在最终镜像中）
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        gcc g++ cmake ninja-build git curl \
        libx11-dev libgl-dev \
    && rm -rf /var/lib/apt/lists/*

# 安装 vcpkg
RUN git clone https://github.com/microsoft/vcpkg.git /opt/vcpkg \
    && /opt/vcpkg/bootstrap-vcpkg.sh -disableMetrics
ENV VCPKG_ROOT=/opt/vcpkg

# 复制源代码并编译
WORKDIR /workspace
COPY . .
RUN cmake -B build \
        -G Ninja \
        -DCMAKE_BUILD_TYPE=Release \
        -DENABLE_TESTS=OFF \
        -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
    && cmake --build build

# ========== 第二阶段：运行环境（精简） ==========
FROM ubuntu:22.04 AS runtime

# 只安装运行时依赖（不需要编译器和头文件）
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libx11-6 libgl1 libstdc++6 \
    && rm -rf /var/lib/apt/lists/*

# 从 builder 阶段复制编译好的二进制
COPY --from=builder /workspace/build/krkr2 /usr/local/bin/krkr2

# 运行时配置
WORKDIR /app
ENTRYPOINT ["krkr2"]
```

最终镜像大小对比：

```
单阶段构建：~2.5GB（包含 GCC、CMake、vcpkg、源码、中间文件等）
多阶段构建：~150MB（只有运行时库 + 编译好的二进制）
                                        节省 ~94% 的空间！
```

### 多阶段构建的进阶技巧

```dockerfile
# 示例 9：三阶段构建——工具链 / 依赖 / 编译
# 阶段 1：工具链（变化最少，缓存命中率最高）
FROM ubuntu:22.04 AS build-tools
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        gcc g++ cmake ninja-build git curl ca-certificates pkg-config \
    && rm -rf /var/lib/apt/lists/*

# 安装 CMake 特定版本
ARG CMAKE_VERSION=3.31.6
RUN curl -sSL "https://github.com/Kitware/CMake/releases/download/v${CMAKE_VERSION}/cmake-${CMAKE_VERSION}-linux-x86_64.tar.gz" \
    | tar -xz -C /opt/ \
    && ln -s /opt/cmake-${CMAKE_VERSION}-linux-x86_64/bin/* /usr/local/bin/

# 安装 vcpkg
RUN git clone https://github.com/microsoft/vcpkg.git /opt/vcpkg \
    && /opt/vcpkg/bootstrap-vcpkg.sh -disableMetrics
ENV VCPKG_ROOT=/opt/vcpkg

# 阶段 2：安装项目依赖（vcpkg.json 变化时才重建）
FROM build-tools AS dependencies
WORKDIR /workspace
# 先只复制依赖描述文件——利用层缓存
COPY vcpkg.json vcpkg-configuration.json* ./
COPY vcpkg/ ./vcpkg/
# 预安装依赖（源码还没复制，所以源码变化不会使这层失效）
RUN $VCPKG_ROOT/vcpkg install --triplet x64-linux

# 阶段 3：编译源码（源码变化时重建，但依赖层已缓存）
FROM dependencies AS build
COPY . .
RUN cmake -B build -G Ninja \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
    && cmake --build build

# 最终阶段：只取编译产物
FROM ubuntu:22.04 AS final
RUN apt-get update \
    && apt-get install -y --no-install-recommends libstdc++6 \
    && rm -rf /var/lib/apt/lists/*
COPY --from=build /workspace/build/krkr2 /usr/local/bin/
ENTRYPOINT ["krkr2"]
```

这种三阶段模式的缓存效率极高：

```
修改了源代码？  → 只重建阶段 3（编译），阶段 1-2 全部缓存
修改了 vcpkg.json？ → 重建阶段 2-3，阶段 1 缓存
修改了基础工具？ → 全部重建
```

## BuildKit 高级特性

### 什么是 BuildKit

BuildKit 是 Docker 18.09 引入的新一代构建引擎，从 Docker 23.0 开始成为默认构建器。它比旧的构建引擎有以下优势：

- **并行构建**：多阶段构建中无依赖的阶段可以并行执行
- **增量传输**：只传输变化的文件到构建守护进程
- **缓存挂载**：`--mount=type=cache`，构建过程中使用持久化缓存
- **Secret 挂载**：`--mount=type=secret`，安全传递密钥而不留在层中
- **SSH 挂载**：`--mount=type=ssh`，转发 SSH 密钥用于克隆私有仓库

启用 BuildKit（Docker 23.0 以下版本需要显式启用）：

```bash
# 方式 1：环境变量
export DOCKER_BUILDKIT=1
docker build .

# 方式 2：使用 docker buildx（推荐）
docker buildx build .

# 方式 3：在 Docker 配置中全局启用
# 编辑 /etc/docker/daemon.json：
# { "features": { "buildkit": true } }
```

### 缓存挂载（--mount=type=cache）

缓存挂载是 BuildKit 对 C++ 项目最有价值的特性。它允许 `RUN` 指令在构建时访问一个持久化的缓存目录，这个目录在构建之间保持不变，但不会被打包到最终镜像中：

```dockerfile
# 示例 10：缓存挂载——vcpkg 和 CMake 构建缓存
# 需要在 Dockerfile 第一行声明使用 BuildKit 语法
# syntax=docker/dockerfile:1

FROM ubuntu:22.04 AS builder

RUN apt-get update \
    && apt-get install -y gcc g++ cmake ninja-build git \
    && rm -rf /var/lib/apt/lists/*

ENV VCPKG_ROOT=/opt/vcpkg
RUN git clone https://github.com/microsoft/vcpkg.git $VCPKG_ROOT \
    && $VCPKG_ROOT/bootstrap-vcpkg.sh -disableMetrics

WORKDIR /workspace
COPY . .

# 使用缓存挂载加速 vcpkg 依赖安装和 CMake 编译
# --mount=type=cache,target=<路径> 将指定路径映射到持久缓存
RUN --mount=type=cache,target=/opt/vcpkg/buildtrees \
    --mount=type=cache,target=/opt/vcpkg/downloads \
    --mount=type=cache,target=/opt/vcpkg/packages \
    --mount=type=cache,target=/workspace/build \
    cmake -B build -G Ninja \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
    && cmake --build build
```

**缓存挂载与层缓存的区别：**

| 特性 | 层缓存（Layer Cache） | 缓存挂载（Cache Mount） |
|------|----------------------|------------------------|
| 作用范围 | 整个 RUN 指令 | 指定的目录 |
| 失效条件 | 任何输入变化（指令文本、COPY 的文件） | 目录内容独立管理 |
| 是否打包到镜像 | 是 | 否 |
| 跨构建持久化 | 是（但容易被新层覆盖） | 是（独立于镜像层） |
| 典型用途 | 工具安装、配置 | 编译缓存、包管理器缓存 |

### Secret 挂载（--mount=type=secret）

构建 C++ 项目时经常需要私有的认证信息（如 vcpkg 二进制缓存的 NuGet API Key）。直接写在 `ENV` 或 `ARG` 中不安全——它们会保存在镜像的层历史中。Secret 挂载在构建时临时提供密钥文件，不留痕迹：

```dockerfile
# 示例 11：Secret 挂载——安全传递 NuGet API Key
# syntax=docker/dockerfile:1

FROM ubuntu:22.04

RUN apt-get update && apt-get install -y mono-complete nuget

# Secret 挂载：密钥文件在构建时挂载到 /run/secrets/<id>
# 构建完成后自动消失，不会保存在任何镜像层中
RUN --mount=type=secret,id=nuget_key \
    nuget sources add \
        -Source "https://nuget.pkg.github.com/jeffcwj/index.json" \
        -Name "GitHub" \
        -UserName "jeffcwj" \
        -Password "$(cat /run/secrets/nuget_key)" \
        -StorePasswordInClearText

# 构建时传递 Secret：
# docker build --secret id=nuget_key,src=./my-api-key.txt .
```

## Docker 命令速查

### 构建与运行

```bash
# 示例 12：常用 Docker 命令

# ===== 构建镜像 =====
# 基本构建（在包含 Dockerfile 的目录中执行）
docker build -t krkr2-builder .

# 指定 Dockerfile 路径（KrKr2 项目的 Dockerfile 在 dockers/ 下）
docker build -f dockers/linux.Dockerfile -t krkr2-linux .

# 使用构建参数
docker build --build-arg CMAKE_VERSION=3.31.6 -t krkr2-linux .

# 多平台构建（需要 buildx）
docker buildx build --platform linux/amd64,linux/arm64 -t krkr2-multi .

# ===== 运行容器 =====
# 交互式运行（进入 bash）
docker run -it krkr2-builder bash

# 挂载本地源码到容器中
docker run -v $(pwd):/workspace krkr2-builder cmake --build /workspace/build

# 后台运行
docker run -d --name my-builder krkr2-builder

# ===== 管理镜像和容器 =====
# 列出所有镜像
docker images

# 列出运行中的容器
docker ps

# 列出所有容器（包括停止的）
docker ps -a

# 删除镜像
docker rmi krkr2-builder

# 删除所有停止的容器
docker container prune

# 删除所有未使用的镜像
docker image prune -a

# ===== 跨平台说明 =====
# Windows：Docker Desktop 必须启用 WSL 2 后端
# Linux：直接安装 docker-ce
# macOS：Docker Desktop 或 colima
# Android：不直接运行 Docker（在 CI 的 Linux Runner 中用 Docker 构建 Android）
```

### 安装 Docker

```bash
# Windows：
# 1. 下载安装 Docker Desktop：https://www.docker.com/products/docker-desktop
# 2. 安装时勾选 "Use WSL 2 instead of Hyper-V"
# 3. 安装后重启，在 PowerShell 中验证：
docker --version

# Linux（Ubuntu/Debian）：
sudo apt-get update
sudo apt-get install -y docker.io
sudo usermod -aG docker $USER    # 将当前用户加入 docker 组
newgrp docker                    # 刷新组权限
docker --version

# macOS：
# 方式 1：下载 Docker Desktop for Mac
# 方式 2：使用 Homebrew
brew install --cask docker
# 方式 3：使用 colima（轻量级替代方案）
brew install colima docker
colima start
docker --version
```

## 动手实践

### 实践 1：编写一个最小的 C++ 构建 Dockerfile

创建一个从零开始的 Dockerfile，编译一个简单的 C++ 程序：

**步骤 1：** 创建测试项目

```bash
# 创建目录
mkdir docker-cpp-demo && cd docker-cpp-demo

# 创建 C++ 源文件
cat > main.cpp << 'EOF'
#include <iostream>
#include <string>

int main() {
    std::string message = "Hello from Docker!";
    std::cout << message << std::endl;
    std::cout << "Compiler: "
#if defined(__GNUC__)
              << "GCC " << __GNUC__ << "." << __GNUC_MINOR__
#elif defined(__clang__)
              << "Clang " << __clang_major__ << "." << __clang_minor__
#elif defined(_MSC_VER)
              << "MSVC " << _MSC_VER
#endif
              << std::endl;
    return 0;
}
EOF

# 创建 CMakeLists.txt
cat > CMakeLists.txt << 'EOF'
cmake_minimum_required(VERSION 3.20)
project(docker-demo LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
add_executable(demo main.cpp)
EOF
```

**步骤 2：** 编写 Dockerfile

```dockerfile
# 文件名：Dockerfile
FROM ubuntu:22.04 AS builder

# 安装构建工具
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        gcc g++ cmake make \
    && rm -rf /var/lib/apt/lists/*

# 复制源码并编译
WORKDIR /workspace
COPY CMakeLists.txt main.cpp ./
RUN cmake -B build -DCMAKE_BUILD_TYPE=Release \
    && cmake --build build

# 运行阶段
FROM ubuntu:22.04
COPY --from=builder /workspace/build/demo /usr/local/bin/demo
ENTRYPOINT ["demo"]
```

**步骤 3：** 构建并运行

```bash
# 构建镜像
docker build -t cpp-demo .

# 运行容器——应该输出 "Hello from Docker!" 和编译器信息
docker run --rm cpp-demo

# 查看镜像大小
docker images cpp-demo
```

### 实践 2：使用缓存挂载优化构建

在实践 1 的基础上，添加缓存挂载加速重复构建：

```dockerfile
# 文件名：Dockerfile.cached
# syntax=docker/dockerfile:1
FROM ubuntu:22.04 AS builder

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        gcc g++ cmake make \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /workspace
COPY CMakeLists.txt main.cpp ./

# 使用缓存挂载——build 目录在构建之间保持
RUN --mount=type=cache,target=/workspace/build \
    cmake -B build -DCMAKE_BUILD_TYPE=Release \
    && cmake --build build \
    && cp build/demo /tmp/demo    # 必须拷出来，因为缓存目录不会进入镜像

FROM ubuntu:22.04
COPY --from=builder /tmp/demo /usr/local/bin/demo
ENTRYPOINT ["demo"]
```

```bash
# 使用 buildx 构建（确保 BuildKit 启用）
docker buildx build -f Dockerfile.cached -t cpp-demo-cached .

# 修改 main.cpp 后重新构建——CMake 增量编译会很快
docker buildx build -f Dockerfile.cached -t cpp-demo-cached .
```

## 对照项目源码

KrKr2 项目有两个 Dockerfile，都在 `dockers/` 目录下：

- `dockers/linux.Dockerfile` 第 1-62 行 — 使用多阶段构建（`build-tools` → `generate`），安装 CMake 3.31.6 和 vcpkg，使用 BuildKit 缓存挂载优化编译
- `dockers/android.Dockerfile` 第 1-67 行 — 单阶段构建，安装 JDK 17、Android SDK/NDK r28 和 vcpkg，使用 Gradle 构建 APK

两个 Dockerfile 展示了本节介绍的多个核心概念：

| 本节概念 | linux.Dockerfile 中的应用 | android.Dockerfile 中的应用 |
|----------|---------------------------|----------------------------|
| `FROM ... AS` | `FROM ubuntu:22.04 AS build-tools`（多阶段） | 只用 `FROM ubuntu:22.04`（单阶段） |
| `RUN` 合并 | 工具安装合并为一条 `RUN` | 同样合并 |
| `ARG` | 定义 `CMAKE_VERSION` | 定义 `SDK_VERSION`、`NDK_VERSION` |
| `ENV` | 设置 `VCPKG_ROOT`、`PATH` | 设置 `ANDROID_SDK`、`ANDROID_NDK` 等 |
| `COPY` | `COPY . /workspace/krkr2` | `COPY . /workspace/krkr2` |
| 缓存挂载 | 5 个 `--mount=type=cache`（vcpkg 4 目录 + build） | 6 个 `--mount=type=cache`（vcpkg 4 + gradle + build） |
| `WORKDIR` | `/workspace/krkr2` | `/workspace/krkr2` |

下一节将逐行深入分析这两个 Dockerfile 的实现细节。

## 常见错误与排查

### 错误 1：COPY 找不到文件

```
COPY failed: file not found in build context
```

**原因：** `COPY` 的源路径是相对于**构建上下文**的，不是相对于 Dockerfile 的位置。

```bash
# ❌ 在 dockers/ 目录中构建——构建上下文是 dockers/，找不到项目根目录的文件
cd dockers && docker build -f linux.Dockerfile .

# ✅ 在项目根目录构建——构建上下文是项目根目录
docker build -f dockers/linux.Dockerfile .
#                                        ^ 这个 . 是构建上下文
```

### 错误 2：层缓存不生效

**原因：** `COPY . .` 放在了 `RUN apt-get install` 前面。任何源文件变化都会使 `COPY` 层失效，导致后面的所有层（包括耗时的包安装）都要重新执行。

```dockerfile
# ❌ 不好的顺序
FROM ubuntu:22.04
COPY . /workspace          # 源码变了 → 这层失效
RUN apt-get install -y gcc  # 前面失效 → 这层也要重建

# ✅ 好的顺序
FROM ubuntu:22.04
RUN apt-get install -y gcc  # 不受源码变化影响 → 保持缓存
COPY . /workspace          # 只有这层和之后的层需要重建
```

### 错误 3：镜像体积过大

**排查方法：**

```bash
# 查看每一层的大小
docker history krkr2-builder

# 输出示例：
# IMAGE          CREATED       SIZE      COMMENT
# a1b2c3d4       2 min ago     500MB     RUN cmake --build build
# e5f6g7h8       5 min ago     1.2GB     RUN apt-get install -y ...
# i9j0k1l2       10 min ago    77MB      /bin/sh -c #(nop) FROM ubuntu:22.04
```

**常见原因和解决方案：**
1. **没用多阶段构建** → 添加第二阶段，只复制二进制
2. **没清理 apt 缓存** → 在 `RUN` 末尾加 `&& rm -rf /var/lib/apt/lists/*`
3. **安装了不需要的推荐包** → 使用 `--no-install-recommends`
4. **COPY 复制了不需要的文件** → 使用 `.dockerignore`

## 本节小结

- Docker 通过**容器化**解决 C++ 项目"在我这能编译"的环境一致性问题
- **镜像**是只读模板，**容器**是镜像的运行实例；镜像由多个**层**叠加而成
- **层缓存**是 Docker 构建速度的关键——变化少的指令放前面，变化多的放后面
- **多阶段构建**是 C++ 项目的标配——构建阶段和运行阶段分离，镜像体积可缩小 90%+
- **BuildKit** 提供缓存挂载、Secret 挂载等高级特性，对 vcpkg/CMake 构建缓存特别有价值
- KrKr2 的 `dockers/linux.Dockerfile` 和 `dockers/android.Dockerfile` 综合运用了多阶段构建和缓存挂载

## 练习题与答案

### 题目 1：优化 Dockerfile 的层缓存

以下 Dockerfile 在每次修改源代码后都要重新安装所有系统包（耗时约 3 分钟）。请重新排列指令顺序，使得源代码变化时不需要重新安装包：

```dockerfile
FROM ubuntu:22.04
COPY . /workspace
RUN apt-get update && apt-get install -y gcc g++ cmake
WORKDIR /workspace
RUN cmake -B build && cmake --build build
```

<details>
<summary>查看答案</summary>

```dockerfile
FROM ubuntu:22.04

# 1. 先安装系统包（几乎不变，缓存命中率高）
RUN apt-get update \
    && apt-get install -y --no-install-recommends gcc g++ cmake \
    && rm -rf /var/lib/apt/lists/*

# 2. 设置工作目录
WORKDIR /workspace

# 3. 先复制依赖描述文件（变化少）
COPY CMakeLists.txt .

# 4. 最后复制源代码（变化频繁）
COPY . .

# 5. 构建
RUN cmake -B build && cmake --build build
```

优化要点：
- `apt-get install` 移到 `COPY` 前面，源码变化不影响包安装层
- 添加 `--no-install-recommends` 减少安装不必要的包
- 添加 `rm -rf /var/lib/apt/lists/*` 清理缓存
- 可以进一步优化：先只 COPY CMakeLists.txt 和 vcpkg.json，安装依赖后再 COPY 源码

</details>

### 题目 2：编写多阶段 Dockerfile

为以下项目编写一个两阶段的 Dockerfile：第一阶段编译，第二阶段只包含可执行文件。要求最终镜像不包含任何编译工具。

```cpp
// main.cpp
#include <iostream>
int main() {
    std::cout << "Multi-stage build works!" << std::endl;
    return 0;
}
```

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(multistage LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
add_executable(app main.cpp)
```

<details>
<summary>查看答案</summary>

```dockerfile
# ========== 阶段 1：编译 ==========
FROM ubuntu:22.04 AS builder

# 安装编译工具
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        gcc g++ cmake make \
    && rm -rf /var/lib/apt/lists/*

# 复制源码
WORKDIR /build
COPY CMakeLists.txt main.cpp ./

# 静态链接，确保运行时不依赖编译器的动态库
RUN cmake -B out -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_EXE_LINKER_FLAGS="-static-libgcc -static-libstdc++" \
    && cmake --build out

# ========== 阶段 2：运行 ==========
FROM ubuntu:22.04

# 只安装最小运行时依赖
RUN apt-get update \
    && apt-get install -y --no-install-recommends libstdc++6 \
    && rm -rf /var/lib/apt/lists/*

# 从 builder 阶段复制编译产物
COPY --from=builder /build/out/app /usr/local/bin/app

# 设置入口点
ENTRYPOINT ["app"]
```

构建和验证：

```bash
# 构建
docker build -t multistage-demo .

# 运行——应输出 "Multi-stage build works!"
docker run --rm multistage-demo

# 验证最终镜像不包含编译工具
docker run --rm multistage-demo which gcc    # 应该找不到
docker run --rm multistage-demo which cmake  # 应该找不到

# 查看镜像大小——应该远小于包含编译器的版本
docker images multistage-demo
```

</details>

### 题目 3：使用 BuildKit 缓存挂载

修改题目 2 的 Dockerfile，添加 BuildKit 缓存挂载，使得重复构建（只修改源码）时 CMake 可以增量编译而不是从头开始。

<details>
<summary>查看答案</summary>

```dockerfile
# syntax=docker/dockerfile:1
# ========== 阶段 1：编译 ==========
FROM ubuntu:22.04 AS builder

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        gcc g++ cmake make \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build
COPY CMakeLists.txt main.cpp ./

# 使用缓存挂载：
# target=/build/out 缓存 CMake 构建目录（增量编译）
# 注意：缓存目录不会进入镜像，所以必须 cp 到非缓存路径
RUN --mount=type=cache,target=/build/out \
    cmake -B out -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_EXE_LINKER_FLAGS="-static-libgcc -static-libstdc++" \
    && cmake --build out \
    && cp out/app /tmp/app    # 复制到非缓存路径

# ========== 阶段 2：运行 ==========
FROM ubuntu:22.04

RUN apt-get update \
    && apt-get install -y --no-install-recommends libstdc++6 \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /tmp/app /usr/local/bin/app
ENTRYPOINT ["app"]
```

关键要点：
1. 第一行 `# syntax=docker/dockerfile:1` 启用 BuildKit 语法
2. `--mount=type=cache,target=/build/out` 使 CMake 构建目录在多次构建间持久化
3. 必须用 `cp out/app /tmp/app` 把产物复制到非缓存路径，因为缓存挂载的目录不进入镜像层
4. 使用 `docker buildx build` 确保 BuildKit 启用

构建并测试增量编译效果：

```bash
# 第一次构建（全量编译）
time docker buildx build -f Dockerfile.cached -t demo-cached .

# 修改 main.cpp 中的输出文字
# 第二次构建（增量编译——只重编译 main.cpp，速度明显更快）
time docker buildx build -f Dockerfile.cached -t demo-cached .
```

</details>

## 下一步

[项目 Dockerfile 解析](./02-项目Dockerfile解析.md) — 逐行分析 KrKr2 项目的 `linux.Dockerfile` 和 `android.Dockerfile`，理解多阶段构建和缓存挂载在实际项目中的应用。

