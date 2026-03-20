# 项目 Dockerfile 解析

> **所属模块：** M11-CI/CD与Docker
> **前置知识：** [Docker 基础与 Dockerfile 语法](./01-Docker基础与Dockerfile语法.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：
1. 逐行理解 KrKr2 的 `linux.Dockerfile` 和 `android.Dockerfile` 的每一条指令
2. 分析多阶段构建在实际项目中的层级设计决策
3. 理解 BuildKit 缓存挂载在 vcpkg、CMake、Gradle 场景中的具体应用
4. 识别两个 Dockerfile 的设计差异及其原因
5. 评估现有 Dockerfile 的优化空间并提出改进方案

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 多阶段构建 | Multi-stage Build | 一个 Dockerfile 中使用多个 FROM，前面阶段安装工具和编译，后面阶段只取结果 |
| BuildKit 缓存挂载 | Cache Mount | `--mount=type=cache` 指令，让构建中的下载/编译缓存在多次构建间持久化 |
| vcpkg 清单模式 | Manifest Mode | vcpkg 通过 `vcpkg.json` 文件声明依赖，构建时自动安装——类似 npm 的 package.json |
| CMake 预设 | CMake Preset | `CMakePresets.json` 中定义的预配置方案，一条命令完成所有 CMake 配置 |
| NDK | Native Development Kit | Android 原生开发工具包，提供交叉编译器（Clang）和系统库头文件 |
| Gradle Wrapper | Gradle Wrapper | `gradlew` 脚本，自动下载并使用项目指定版本的 Gradle，确保构建一致性 |
| dos2unix | dos2unix | 将 Windows 的 CRLF 换行符转换为 Unix 的 LF——Git 在 Windows 上 checkout 可能引入 CRLF |

## linux.Dockerfile 逐行解析

### 完整文件概览

KrKr2 的 Linux Dockerfile 位于 `dockers/linux.Dockerfile`（共 62 行），采用两阶段构建：

```
阶段 1: build-tools     阶段 2: generate
┌─────────────────┐    ┌─────────────────┐
│ Ubuntu 22.04    │    │ 继承 build-tools │
│ + CMake 3.31.6  │ →  │ + 复制源代码      │
│ + GCC/G++       │    │ + cmake 构建      │
│ + vcpkg         │    │ + 产出二进制       │
│ + Ninja         │    └─────────────────┘
└─────────────────┘
```

### 第一阶段：build-tools

```dockerfile
# dockers/linux.Dockerfile 第 1-34 行

# 第一阶段：命名为 build-tools
# AS build-tools 给这个阶段一个名字，后面的阶段可以引用它
FROM ubuntu:22.04 AS build-tools
```

选择 `ubuntu:22.04`（Jammy Jellyfish）的原因：
- LTS 版本，支持到 2027 年，稳定性好
- 默认 GCC 11.4，完整支持 C++17
- apt 仓库齐全，安装开发库方便
- GitHub Actions 的 `ubuntu-latest` 也基于此版本，环境一致

```dockerfile
# 设置 DEBIAN_FRONTEND=noninteractive 避免 apt 安装时弹出交互式对话框
# 在 Docker 构建中没有终端，交互式提示会导致构建挂起
ENV DEBIAN_FRONTEND=noninteractive
```

```dockerfile
# 安装基础构建工具
# 这是一条合并的 RUN 指令——多个命令用 && 连接，减少镜像层数
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        # C/C++ 编译器套件
        gcc g++ \
        # 构建工具
        ninja-build \
        # 版本控制（vcpkg 需要 git 来克隆自身和端口）
        git \
        # 下载工具（安装 CMake 和 vcpkg 端口下载源码时需要）
        curl ca-certificates \
        # 解压工具
        unzip tar \
        # pkg-config 用于查找系统库的编译标志
        pkg-config \
        # Linux 图形开发库——KrKr2 使用 Cocos2d-x 渲染，需要 X11 和 OpenGL
        libx11-dev libgl-dev \
    && rm -rf /var/lib/apt/lists/*
```

关键设计决策分析：
- **`--no-install-recommends`**：不安装推荐包，减小镜像约 200MB
- **`rm -rf /var/lib/apt/lists/*`**：清理 apt 缓存，因为后续不需要再安装包
- **`libx11-dev libgl-dev`**：Cocos2d-x 的 Linux 渲染后端需要 X11 和 OpenGL 头文件

```dockerfile
# 安装特定版本的 CMake（3.31.6）
# 为什么不用 apt 安装？因为 Ubuntu 22.04 的 apt 仓库中 CMake 版本较旧（3.22）
# KrKr2 的 CMakeLists.txt 要求 cmake_minimum_required(VERSION 3.28)
ARG CMAKE_VERSION=3.31.6
RUN curl -sSL \
    "https://github.com/Kitware/CMake/releases/download/v${CMAKE_VERSION}/cmake-${CMAKE_VERSION}-linux-x86_64.tar.gz" \
    | tar -xz -C /opt/ \
    && ln -s /opt/cmake-${CMAKE_VERSION}-linux-x86_64/bin/* /usr/local/bin/
```

这里使用了管道技巧：`curl | tar` 直接将下载的 tar.gz 流式解压，不需要先保存到磁盘再解压，减少了一个文件清理步骤。`ln -s` 创建符号链接使 `cmake`、`ctest`、`cpack` 等命令在 PATH 中可用。

```dockerfile
# 安装 vcpkg
# vcpkg 是 KrKr2 的包管理器，管理 FFmpeg、OpenCV、SDL2 等 C++ 依赖
RUN git clone https://github.com/microsoft/vcpkg.git /opt/vcpkg \
    && /opt/vcpkg/bootstrap-vcpkg.sh -disableMetrics

# 设置环境变量，让 CMake 的 vcpkg 工具链文件能找到 vcpkg
ENV VCPKG_ROOT=/opt/vcpkg
ENV PATH="${VCPKG_ROOT}:${PATH}"
```

> **注意：** 这里克隆了 vcpkg 的 `main` 分支最新版本，没有固定到特定 commit。这意味着不同时间构建可能使用不同版本的 vcpkg，存在可重现性风险。理想做法是 `git clone --branch 2024.01.12` 或 `git checkout <commit-hash>`。

### 第二阶段：generate

```dockerfile
# dockers/linux.Dockerfile 第 36-62 行

# 第二阶段：基于 build-tools 阶段
# FROM build-tools 继承了第一阶段安装的所有工具
FROM build-tools AS generate
```

为什么用 `FROM build-tools` 而不是 `FROM ubuntu:22.04`？因为第二阶段需要第一阶段安装的所有编译工具来编译代码。这与"构建 + 运行分离"的经典多阶段不同——这里的多阶段主要目的是**利用层缓存**：工具链安装（阶段 1）很少变化，源代码（阶段 2）频繁变化。

```dockerfile
# 将整个项目源代码复制到容器中
WORKDIR /workspace/krkr2
COPY . .
```

`COPY . .` 将构建上下文（项目根目录）的所有文件复制到容器的 `/workspace/krkr2` 中。这也是为什么构建命令要在项目根目录执行：

```bash
# 必须在项目根目录执行，这样 COPY . . 才能复制到所有源文件
docker build -f dockers/linux.Dockerfile -t krkr2-linux .
```

```dockerfile
# 使用 BuildKit 缓存挂载执行 CMake 构建
# 这是整个 Dockerfile 最核心的部分
RUN --mount=type=cache,target=/opt/vcpkg/buildtrees \
    --mount=type=cache,target=/opt/vcpkg/downloads \
    --mount=type=cache,target=/opt/vcpkg/installed \
    --mount=type=cache,target=/opt/vcpkg/packages \
    --mount=type=cache,target=/workspace/out \
    cmake --preset="Linux Release Config" \
        -DENABLE_TESTS=OFF \
        -DBUILD_TOOLS=OFF \
    && cmake --build --preset="Linux Release Build"
```

这条 `RUN` 指令是整个 Dockerfile 的核心，包含了 5 个缓存挂载。逐一解析：

```
缓存挂载详解：
┌──────────────────────────────────┬───────────────────────────────────┐
│ 缓存目标                         │ 缓存的内容                        │
├──────────────────────────────────┼───────────────────────────────────┤
│ /opt/vcpkg/buildtrees            │ vcpkg 编译依赖时的中间文件          │
│                                  │ （FFmpeg、SDL2 等的编译过程文件）   │
├──────────────────────────────────┼───────────────────────────────────┤
│ /opt/vcpkg/downloads             │ vcpkg 下载的源码压缩包             │
│                                  │ （避免重复从网络下载）              │
├──────────────────────────────────┼───────────────────────────────────┤
│ /opt/vcpkg/installed             │ vcpkg 安装好的库文件               │
│                                  │ （头文件、静态库、动态库）          │
├──────────────────────────────────┼───────────────────────────────────┤
│ /opt/vcpkg/packages              │ vcpkg 的打包输出                   │
│                                  │ （中间产物，用于安装步骤）          │
├──────────────────────────────────┼───────────────────────────────────┤
│ /workspace/out                   │ CMake 构建目录                    │
│                                  │ （.o 文件、最终二进制等编译产物）   │
└──────────────────────────────────┴───────────────────────────────────┘
```

CMake 参数解析：
- `--preset="Linux Release Config"`：使用 `CMakePresets.json` 中定义的 Linux Release 预设
- `-DENABLE_TESTS=OFF`：禁用测试编译（加快构建速度，Docker 构建通常不跑测试）
- `-DBUILD_TOOLS=OFF`：禁用工具编译（如 XP3 解压工具，Docker 构建不需要）

## android.Dockerfile 逐行解析

### 完整文件概览

Android Dockerfile 位于 `dockers/android.Dockerfile`（共 67 行），采用单阶段构建：

```
单阶段构建（无多阶段）
┌─────────────────────────┐
│ Ubuntu 22.04            │
│ + JDK 17                │
│ + Android SDK           │
│ + Android NDK r28       │
│ + vcpkg                 │
│ + 复制源代码             │
│ + Gradle 构建 APK       │
└─────────────────────────┘
```

为什么不用多阶段？因为 Android 构建通过 Gradle 管理，Gradle 本身就是一个完整的构建系统，产物（APK）的提取比 CMake 二进制复杂得多。单阶段对于"用 Docker 构建 APK"的场景已经够用。

### 逐行解析

```dockerfile
# dockers/android.Dockerfile 第 1-8 行
FROM ubuntu:22.04

# 同样禁用交互式提示
ENV DEBIAN_FRONTEND=noninteractive

# 安装基础工具——注意比 Linux Dockerfile 多了 dos2unix
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        # JDK 17——Android Gradle Plugin 8.x 要求 JDK 17+
        openjdk-17-jdk \
        # 基础工具
        git curl unzip \
        # dos2unix——将 Windows CRLF 换行符转为 Unix LF
        # Git 在 Windows 上 checkout 时可能给 gradlew 脚本引入 CRLF
        # CRLF 的 shell 脚本在 Linux 上会报 "/bin/bash^M: bad interpreter"
        dos2unix \
    && rm -rf /var/lib/apt/lists/*
```

**为什么需要 `dos2unix`？** 这是一个容易被忽视但非常关键的细节。`gradlew`（Gradle Wrapper 脚本）是一个 shell 脚本。如果某个开发者在 Windows 上修改并提交了这个文件，Git 可能会将换行符从 LF 转换为 CRLF。在 Linux 容器中运行带有 CRLF 的 shell 脚本会得到 `bad interpreter` 错误。

```dockerfile
# dockers/android.Dockerfile 第 10-28 行

# Android SDK/NDK 版本通过 ARG 参数化
ARG SDK_VERSION=11076708
ARG NDK_VERSION=28.0.13004108
ARG BUILD_TOOLS_VERSION=35.0.0
ARG PLATFORM_VERSION=35

# 设置 Android SDK 相关环境变量
ENV ANDROID_HOME=/opt/android-sdk
ENV ANDROID_SDK=${ANDROID_HOME}
ENV ANDROID_NDK=${ANDROID_HOME}/ndk/${NDK_VERSION}
ENV PATH="${ANDROID_HOME}/cmdline-tools/latest/bin:${ANDROID_HOME}/platform-tools:${PATH}"
```

这里将版本号定义为 `ARG`，方便在构建时通过 `--build-arg` 覆盖，比如升级到新版 NDK：
```bash
docker build --build-arg NDK_VERSION=29.0.xxxxx -f dockers/android.Dockerfile .
```

```dockerfile
# 安装 Android SDK
RUN mkdir -p ${ANDROID_HOME}/cmdline-tools \
    && curl -sSL \
        "https://dl.google.com/android/repository/commandlinetools-linux-${SDK_VERSION}_latest.zip" \
        -o /tmp/sdk.zip \
    && unzip -q /tmp/sdk.zip -d ${ANDROID_HOME}/cmdline-tools \
    && mv ${ANDROID_HOME}/cmdline-tools/cmdline-tools ${ANDROID_HOME}/cmdline-tools/latest \
    && rm /tmp/sdk.zip
```

Android SDK 的安装比较繁琐——需要先下载命令行工具包，然后用 `sdkmanager` 安装具体组件。注意 `mv cmdline-tools cmdline-tools/latest` 这一步是 Google 的惯例目录结构要求。

```dockerfile
# 接受 Android SDK 许可证并安装组件
RUN yes | sdkmanager --licenses > /dev/null 2>&1 \
    && sdkmanager \
        "platform-tools" \
        "platforms;android-${PLATFORM_VERSION}" \
        "build-tools;${BUILD_TOOLS_VERSION}" \
        "ndk;${NDK_VERSION}" \
        "cmake;3.31.1"
```

`yes | sdkmanager --licenses` 自动接受所有许可证（交互式操作在 Docker 中不可行）。安装的组件包括：
- `platform-tools`：adb 等调试工具
- `platforms;android-35`：Android API 35 的系统库
- `build-tools;35.0.0`：aapt2、dx 等构建工具
- `ndk;28.0.13004108`：NDK r28，包含交叉编译器（Clang）
- `cmake;3.31.1`：Android SDK 自带的 CMake（用于 NDK 的 CMake 构建）

```dockerfile
# 安装 vcpkg
RUN git clone https://github.com/microsoft/vcpkg.git /opt/vcpkg \
    && /opt/vcpkg/bootstrap-vcpkg.sh -disableMetrics
ENV VCPKG_ROOT=/opt/vcpkg
```

与 Linux Dockerfile 完全一样的 vcpkg 安装步骤。

```dockerfile
# dockers/android.Dockerfile 第 48-67 行

# 复制源代码
WORKDIR /workspace/krkr2
COPY . .

# 使用 BuildKit 缓存挂载执行 Gradle 构建
RUN --mount=type=cache,target=/opt/vcpkg/buildtrees \
    --mount=type=cache,target=/opt/vcpkg/downloads \
    --mount=type=cache,target=/opt/vcpkg/installed \
    --mount=type=cache,target=/opt/vcpkg/packages \
    --mount=type=cache,target=/workspace/out \
    --mount=type=cache,target=/root/.gradle \
    dos2unix platforms/android/gradlew \
    && chmod +x platforms/android/gradlew \
    && platforms/android/gradlew -p platforms/android assembleRelease
```

比 Linux Dockerfile 多了一个缓存挂载和两个预处理步骤：

```
Android 独有的细节：
┌──────────────────────────────┬─────────────────────────────────────┐
│ 多出的缓存挂载               │ 说明                                 │
├──────────────────────────────┼─────────────────────────────────────┤
│ /root/.gradle                │ Gradle 缓存目录，包含下载的依赖 JAR、│
│                              │ 构建缓存和 wrapper 发行版             │
└──────────────────────────────┴─────────────────────────────────────┘

┌──────────────────────────────┬─────────────────────────────────────┐
│ 多出的构建步骤               │ 说明                                 │
├──────────────────────────────┼─────────────────────────────────────┤
│ dos2unix gradlew             │ 修复可能的 CRLF 问题                 │
│ chmod +x gradlew             │ 确保脚本有执行权限                    │
└──────────────────────────────┴─────────────────────────────────────┘
```

Gradle 构建命令 `assembleRelease` 会触发完整的 Android 构建流水线：
1. Gradle 下载自身和插件依赖
2. CMake（通过 Android Gradle Plugin 的 `externalNativeBuild`）编译 C++ 代码
3. 将 .so 文件打包到 APK 中
4. 编译 Java/Kotlin 代码
5. 资源打包、签名、对齐

## 两个 Dockerfile 的对比分析

理解完各自的实现后，我们把两个 Dockerfile 放在一起对比，看看它们的设计决策差异：

### 结构对比表

```
┌─────────────────────┬──────────────────────────┬──────────────────────────┐
│ 对比维度             │ linux.Dockerfile          │ android.Dockerfile        │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ 总行数               │ 62 行                     │ 67 行                     │
│ 构建阶段数           │ 2 阶段（build-tools →    │ 1 阶段（单 FROM）          │
│                     │ generate）                │                          │
│ 基础镜像             │ ubuntu:22.04              │ ubuntu:22.04              │
│ 编译器               │ GCC 11.4（apt 安装）      │ NDK r28 Clang（SDK 安装） │
│ 构建系统             │ CMake + Ninja             │ Gradle + CMake（NDK）     │
│ CMake 来源           │ 官方 tarball 3.31.6       │ Android SDK 内置 3.31.1   │
│ 缓存挂载数           │ 5 个                      │ 6 个（多 .gradle）        │
│ 特殊处理             │ 无                        │ dos2unix + chmod          │
│ 最终产物             │ ELF 二进制文件            │ APK 文件                  │
│ 预估镜像大小         │ ~1.5 GB                   │ ~4.5 GB（含 Android SDK） │
└─────────────────────┴──────────────────────────┴──────────────────────────┘
```

### 为什么 Linux 用多阶段而 Android 不用？

**Linux 的多阶段设计理由：**

多阶段构建（Multi-stage Build）的核心价值是**层缓存隔离**。linux.Dockerfile 把"安装工具"和"编译代码"分为两个阶段，理由是：

1. **工具链变化频率低**：GCC、CMake、vcpkg 很少需要升级，`build-tools` 阶段的层缓存可以长期复用
2. **源码变化频率高**：每次代码改动都会使 `COPY . .` 的层失效，但不影响 `build-tools` 阶段
3. **Docker 层缓存机制**：Docker 按指令顺序检查缓存，一旦某层失效，后续所有层都必须重建。分阶段可以让前面的层保持有效

```
层缓存失效链（单阶段 vs 多阶段）：

单阶段（如果不分阶段）：
  FROM ubuntu → apt install → cmake install → vcpkg → COPY . . → cmake build
                                                        ↑
                                               代码改动导致此处失效
                                               但 apt/cmake/vcpkg 层缓存仍有效 ✓

多阶段（实际设计）：
  阶段 1: FROM ubuntu → apt → cmake → vcpkg    ← 整个阶段缓存有效 ✓
  阶段 2: FROM build-tools → COPY . . → build  ← 只重建这个阶段 ✓
```

实际上在这个项目中，由于 `FROM build-tools` 继承了第一阶段的全部内容，多阶段的主要价值是**代码组织清晰**和**潜在的独立使用**（可以单独 `docker build --target build-tools` 获得一个纯工具链镜像）。

**Android 不用多阶段的原因：**

1. **Gradle 是一体化构建系统**：Gradle 管理从依赖下载、NDK 编译到 APK 打包签名的全流程，拆分阶段反而会增加复杂度
2. **APK 产物位置复杂**：APK 输出在 Gradle 管理的深层目录中（如 `platforms/android/app/build/outputs/apk/release/`），从一个阶段 COPY 到另一个阶段比较麻烦
3. **Android SDK 无法轻量化**：即使分阶段，运行 Gradle 仍需要完整的 SDK/NDK，无法像 Go 那样只拷贝一个二进制到 alpine 镜像

### 缓存挂载效果量化

两个 Dockerfile 都大量使用了 `--mount=type=cache`。以下是有缓存和无缓存的构建时间对比（基于典型 CI 环境，仅改动一个 .cpp 文件的增量构建）：

```
构建时间对比（估算值，实际因网速和机器性能而异）：

┌───────────────────┬──────────────┬──────────────┬───────────┐
│ 场景               │ 无缓存（冷） │ 有缓存（热） │ 节省      │
├───────────────────┼──────────────┼──────────────┼───────────┤
│ Linux 全量构建     │ ~25 分钟     │ ~3 分钟      │ ~88%      │
│ Linux 增量构建     │ ~25 分钟     │ ~1 分钟      │ ~96%      │
│ Android 全量构建   │ ~35 分钟     │ ~5 分钟      │ ~86%      │
│ Android 增量构建   │ ~35 分钟     │ ~2 分钟      │ ~94%      │
└───────────────────┴──────────────┴──────────────┴───────────┘

主要耗时分布（冷构建）：
- vcpkg 依赖下载 + 编译：~15 分钟（FFmpeg 编译尤其耗时）
- CMake 配置 + C++ 编译：~8 分钟
- Gradle 依赖下载（仅 Android）：~5 分钟
- APK 打包签名（仅 Android）：~2 分钟
```

缓存挂载的关键价值是**跨构建持久化**：即使 Docker 层缓存失效（因为 `COPY . .` 改变了），`--mount=type=cache` 挂载的目录内容仍然保留。这意味着 vcpkg 不需要重新下载和编译所有依赖，CMake 不需要重新编译所有 .o 文件。

## 动手实践

### 实践 1：构建 Linux Docker 镜像

```bash
# 进入项目根目录
cd /path/to/krkr2

# 确认 Docker 已安装且 BuildKit 已启用
docker --version
# Docker version 24.x 或更高

# 设置 BuildKit 环境变量（如果 Docker 版本低于 23.0）
export DOCKER_BUILDKIT=1

# 构建 Linux 镜像
# -f 指定 Dockerfile 路径
# -t 给镜像打标签
# . 表示构建上下文是当前目录（项目根目录）
docker build -f dockers/linux.Dockerfile -t krkr2-linux .

# 查看构建好的镜像
docker images | grep krkr2
# REPOSITORY    TAG       IMAGE ID       CREATED          SIZE
# krkr2-linux   latest    abc123def456   30 seconds ago   ~1.5GB
```

**预期结果：** 构建成功，镜像大小约 1.5GB。首次构建需要 20-30 分钟（取决于网速），后续增量构建约 1-3 分钟。

**常见问题：** 如果在中国大陆，vcpkg 下载可能很慢。可以配置代理：
```bash
docker build \
    --build-arg http_proxy=http://your-proxy:port \
    --build-arg https_proxy=http://your-proxy:port \
    -f dockers/linux.Dockerfile -t krkr2-linux .
```

### 实践 2：构建 Android Docker 镜像

```bash
# 构建 Android 镜像
docker build -f dockers/android.Dockerfile -t krkr2-android .

# 查看镜像大小（比 Linux 大很多，因为包含 Android SDK + NDK）
docker images | grep krkr2
# REPOSITORY       TAG       IMAGE ID       CREATED          SIZE
# krkr2-android    latest    def456abc789   30 seconds ago   ~4.5GB
# krkr2-linux      latest    abc123def456   5 minutes ago    ~1.5GB
```

**预期结果：** 构建成功，镜像大小约 4.5GB。首次构建需要 30-40 分钟。

### 实践 3：验证缓存挂载的增量构建效果

```bash
# 第一步：做一次小改动
echo "// test change" >> cpp/core/base/XP3Archive.cpp

# 第二步：重新构建 Linux 镜像，观察耗时
time docker build -f dockers/linux.Dockerfile -t krkr2-linux .

# 第三步：对比首次构建和增量构建的耗时
# 首次：约 25 分钟
# 增量：约 1-3 分钟（因为 vcpkg 缓存仍在，只重新编译改动的文件）

# 第四步：撤销测试改动
git checkout cpp/core/base/XP3Archive.cpp
```

**观察要点：** 在增量构建的输出中，你会看到 vcpkg 的步骤几乎瞬间完成（显示 "Already installed"），CMake 也只编译被改动影响的目标文件。这就是 `--mount=type=cache` 的效果。

### 实践 4：创建 .dockerignore 优化构建上下文

目前项目没有 `.dockerignore` 文件。`COPY . .` 会把项目根目录下的所有文件复制到容器中，包括 `.git`（可能数百 MB）、`out/`（构建产物）、IDE 配置等无用文件。创建一个 `.dockerignore` 可以显著减小构建上下文大小：

```bash
# 在项目根目录创建 .dockerignore
cat > .dockerignore << 'EOF'
# 版本控制
.git
.gitignore

# 构建产物
out/
build/

# IDE 配置
.idea/
.vscode/
*.swp
*.swo

# Docker 相关（避免递归）
dockers/

# 文档（构建不需要）
docs/
doc/

# 操作系统文件
.DS_Store
Thumbs.db

# 其他
*.log
EOF

# 查看构建上下文大小的变化
# 无 .dockerignore 时，docker build 开头会显示：
#   Sending build context to Docker daemon  XXX MB
# 有 .dockerignore 后，这个数字会显著减小
```

**预期结果：** 构建上下文大小从数百 MB 减小到约 50-100 MB，`COPY . .` 步骤的耗时也会明显缩短。

## 对照项目源码

本节涉及的所有项目文件及其在 Dockerfile 中的作用：

```
项目文件与 Dockerfile 的关联：

┌─────────────────────────────────┬───────────┬──────────────────────────────┐
│ 文件路径                         │ 行数      │ 在 Dockerfile 中的作用        │
├─────────────────────────────────┼───────────┼──────────────────────────────┤
│ dockers/linux.Dockerfile         │ 62 行     │ Linux 平台 Docker 构建脚本    │
│ dockers/android.Dockerfile       │ 67 行     │ Android 平台 Docker 构建脚本  │
│ CMakePresets.json                │ 156 行    │ 定义 CMake 预设，Dockerfile   │
│                                 │           │ 中 --preset 参数引用          │
│ CMakeLists.txt（根目录）         │ 143 行    │ 定义 ENABLE_TESTS、           │
│                                 │           │ BUILD_TOOLS 等选项            │
│ vcpkg.json                      │ -         │ vcpkg 清单文件，定义 C++ 依赖 │
│ platforms/android/gradlew        │ -         │ Gradle Wrapper 脚本，         │
│                                 │           │ android.Dockerfile 用         │
│                                 │           │ dos2unix 处理后执行            │
│ platforms/android/build.gradle   │ -         │ Android 构建配置，定义         │
│                                 │           │ externalNativeBuild 调用 CMake│
└─────────────────────────────────┴───────────┴──────────────────────────────┘
```

### Dockerfile 与 CI 工作流的对比

Dockerfile 和 CI 工作流（`.github/workflows/build-linux.yml` 等）做的是类似的事情——在干净环境中编译项目。但它们的设计目标不同：

```
┌─────────────────────┬──────────────────────────┬──────────────────────────┐
│ 对比维度             │ Dockerfile                │ CI 工作流                 │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ 运行环境             │ 任何安装了 Docker 的机器  │ GitHub Actions Runner     │
│ 环境复用             │ 镜像可分发、离线使用       │ 每次运行从零开始          │
│ 缓存机制             │ BuildKit cache mount      │ actions/cache + ccache    │
│ vcpkg 缓存           │ --mount=type=cache        │ NuGet 二进制缓存          │
│ 适用场景             │ 本地开发、离线构建         │ 自动化 CI/CD              │
│ 触发方式             │ 手动 docker build         │ Git push/PR 自动触发      │
│ 产物获取             │ docker cp 或 volume       │ actions/upload-artifact   │
└─────────────────────┴──────────────────────────┴──────────────────────────┘
```

## 常见错误与排查

### 错误 1：构建上下文路径错误

```
错误信息：
COPY failed: file not found in build context or excluded by .dockerignore

原因：在错误的目录执行了 docker build。
```

```bash
# 错误示范：在 dockers/ 目录下执行
cd dockers/
docker build -f linux.Dockerfile -t krkr2-linux .
# COPY . . 只会复制 dockers/ 目录的内容，找不到源代码

# 正确做法：在项目根目录执行
cd /path/to/krkr2
docker build -f dockers/linux.Dockerfile -t krkr2-linux .
# COPY . . 复制整个项目根目录的内容
```

**排查步骤：**
1. 确认当前目录是项目根目录（`ls` 能看到 `cpp/`、`platforms/`、`dockers/` 等）
2. 确认 `-f` 参数指向正确的 Dockerfile 相对路径
3. 确认构建上下文（最后的 `.`）是项目根目录

### 错误 2：BuildKit 未启用

```
错误信息：
the --mount option requires BuildKit. Refer to
https://docs.docker.com/go/buildkit/ to learn how to enable BuildKit

原因：Docker 版本低于 23.0 且未设置 DOCKER_BUILDKIT 环境变量。
```

```bash
# 解决方案 1：设置环境变量（临时）
export DOCKER_BUILDKIT=1
docker build -f dockers/linux.Dockerfile -t krkr2-linux .

# 解决方案 2：修改 Docker 配置（永久）
# Linux/macOS: 编辑 /etc/docker/daemon.json
{
    "features": {
        "buildkit": true
    }
}

# Windows (Docker Desktop): Settings → Docker Engine → 添加上述配置

# 解决方案 3：升级 Docker 到 23.0+（推荐）
# Docker 23.0 起 BuildKit 默认启用
```

**排查步骤：**
1. 检查 Docker 版本：`docker --version`
2. 如果低于 23.0，设置 `DOCKER_BUILDKIT=1`
3. 验证 BuildKit 是否生效：构建输出应显示 `[+] Building` 而非旧版的 `Step 1/N`

### 错误 3：gradlew 权限错误（Android 专用）

```
错误信息：
/bin/bash: ./platforms/android/gradlew: /bin/bash^M: bad interpreter:
No such file or directory

原因：gradlew 脚本包含 Windows CRLF 换行符。
```

android.Dockerfile 中的 `dos2unix` 和 `chmod +x` 就是为了解决这个问题。如果你修改了 Dockerfile 去掉了这两步，或者用了不同的 Dockerfile，就会遇到这个错误。

```bash
# 手动修复（在容器内或宿主机上）
dos2unix platforms/android/gradlew
chmod +x platforms/android/gradlew

# 或者用 sed 替代 dos2unix
sed -i 's/\r$//' platforms/android/gradlew

# 预防措施：在 .gitattributes 中强制 gradlew 使用 LF
echo "gradlew text eol=lf" >> platforms/android/.gitattributes
```

### 错误 4：磁盘空间不足

```
错误信息：
no space left on device

原因：Docker 镜像和构建缓存占用大量磁盘空间。
Android 镜像约 4.5GB，加上中间层和缓存可能需要 15-20GB 可用空间。
```

```bash
# 查看 Docker 磁盘使用情况
docker system df
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          5         2         12.5GB    8.3GB (66%)
# Containers      2         0         1.2GB     1.2GB (100%)
# Build Cache     15        0         5.6GB     5.6GB (100%)

# 清理未使用的资源
docker system prune -a
# WARNING! This will remove all stopped containers, unused networks,
# unused images, and build cache.

# 只清理构建缓存（保留镜像）
docker builder prune

# 只清理悬空镜像（无标签的中间层）
docker image prune
```

**预防建议：** 确保 Docker 的存储目录所在分区至少有 30GB 可用空间，尤其在构建 Android 镜像时。

## 现有 Dockerfile 的改进建议

项目当前的 Dockerfile 已经比较完善，但仍有优化空间：

### 改进 1：固定 vcpkg 版本

```dockerfile
# 当前写法（不固定版本，存在可重现性风险）
RUN git clone https://github.com/microsoft/vcpkg.git /opt/vcpkg

# 改进写法（固定到特定 release tag）
ARG VCPKG_VERSION=2024.12.16
RUN git clone --branch "${VCPKG_VERSION}" --depth 1 \
    https://github.com/microsoft/vcpkg.git /opt/vcpkg \
    && /opt/vcpkg/bootstrap-vcpkg.sh -disableMetrics
```

`--depth 1` 执行浅克隆（Shallow Clone，只获取最新一次提交的文件，不包含完整历史记录），减少下载量约 90%。`--branch` 固定版本确保不同时间的构建结果一致。

### 改进 2：添加 .dockerignore

如"动手实践 4"所示，添加 `.dockerignore` 文件排除 `.git`、`out/`、`docs/` 等不参与构建的目录，减小构建上下文传输时间。

### 改进 3：分离 vcpkg install 步骤

```dockerfile
# 当前写法：vcpkg install 和 cmake build 在同一个 RUN 中
# 如果 cmake build 失败，vcpkg install 的结果不会被 Docker 层缓存

# 改进写法：把 vcpkg install 独立出来
# 先只复制 vcpkg 相关文件
COPY vcpkg.json vcpkg-configuration.json ./
COPY vcpkg/ ./vcpkg/

# 单独执行 vcpkg install（这一步的结果可以被层缓存）
RUN --mount=type=cache,target=/opt/vcpkg/buildtrees \
    --mount=type=cache,target=/opt/vcpkg/downloads \
    --mount=type=cache,target=/opt/vcpkg/installed \
    --mount=type=cache,target=/opt/vcpkg/packages \
    vcpkg install --triplet x64-linux

# 再复制全部源码并构建
COPY . .
RUN --mount=type=cache,target=/workspace/out \
    cmake --preset="Linux Release Config" \
        -DENABLE_TESTS=OFF -DBUILD_TOOLS=OFF \
    && cmake --build --preset="Linux Release Build"
```

这样做的好处：只要 `vcpkg.json` 没变，vcpkg install 步骤的 Docker 层缓存就不会失效，即使源代码改动了也不需要重新安装依赖。

### 改进 4：添加元数据标签

```dockerfile
# 在 Dockerfile 开头添加 OCI 标准标签
LABEL org.opencontainers.image.source="https://github.com/jeffcwj/krkr2_angle"
LABEL org.opencontainers.image.description="KrKr2 Linux build environment"
LABEL org.opencontainers.image.licenses="MIT"
```

元数据标签（Metadata Label）让镜像自带描述信息，在 Docker Hub 或 GitHub Container Registry 上查看时能直接看到来源和用途。

## 本节小结

- linux.Dockerfile 采用**两阶段构建**（build-tools → generate），将工具链安装和代码编译分离，利用层缓存加速增量构建
- android.Dockerfile 采用**单阶段构建**，因为 Gradle 是一体化构建系统，拆分阶段收益不大
- 两个 Dockerfile 都基于 **Ubuntu 22.04 LTS**，确保与 GitHub Actions 环境一致
- **BuildKit 缓存挂载**（`--mount=type=cache`）是性能关键——跨构建持久化 vcpkg 依赖和编译中间文件，增量构建可节省 88%-96% 的时间
- Android Dockerfile 需要额外处理 **dos2unix**（修复 gradlew 的 CRLF 换行符问题）和 **chmod**（确保脚本有执行权限）
- vcpkg 未固定版本是一个**可重现性风险**，建议使用 `--branch <tag>` 固定版本
- 添加 **.dockerignore** 文件可以显著减小构建上下文，缩短 `COPY . .` 的传输时间

## 练习题与答案

### 题目 1：为什么 linux.Dockerfile 的 `build-tools` 阶段使用 `curl | tar` 管道安装 CMake，而不是先 `curl -o cmake.tar.gz` 下载再 `tar xzf cmake.tar.gz` 解压？

<details>
<summary>查看答案</summary>

使用管道 `curl | tar` 有两个好处：

1. **减少磁盘 I/O 和临时文件**：管道方式将下载的数据直接流式传给 tar 解压，不需要先将 tar.gz 文件写入磁盘。在 Docker 构建中，每个文件写入都会增加镜像层的大小。如果先下载到磁盘，即使后续 `rm` 删除了文件，由于 Docker 的联合文件系统（Union Filesystem，将多个目录层叠合并为一个文件系统的技术）特性，该文件仍然存在于前面的层中，占用空间。

2. **减少 RUN 指令中的步骤**：不需要额外的 `rm cmake.tar.gz` 清理命令，代码更简洁。

如果不用管道而要避免层大小问题，需要把下载、解压、删除放在同一条 RUN 指令中：
```dockerfile
RUN curl -sSL "https://...cmake.tar.gz" -o /tmp/cmake.tar.gz \
    && tar -xzf /tmp/cmake.tar.gz -C /opt/ \
    && rm /tmp/cmake.tar.gz \
    && ln -s /opt/cmake-*/bin/* /usr/local/bin/
```

这比管道方式更冗长，效果相同。因此管道是更优雅的写法。

</details>

### 题目 2：android.Dockerfile 使用了 6 个 `--mount=type=cache` 挂载，请列出这 6 个挂载目标并解释每个缓存的内容。如果去掉 `/root/.gradle` 的缓存挂载，会有什么影响？

<details>
<summary>查看答案</summary>

6 个缓存挂载目标：

| # | 挂载目标 | 缓存内容 |
|---|---------|---------|
| 1 | `/opt/vcpkg/buildtrees` | vcpkg 编译依赖库（如 FFmpeg、SDL2）时的中间构建文件 |
| 2 | `/opt/vcpkg/downloads` | vcpkg 从网络下载的源码压缩包 |
| 3 | `/opt/vcpkg/installed` | vcpkg 安装完成的库文件（头文件、静态库、动态库） |
| 4 | `/opt/vcpkg/packages` | vcpkg 的打包中间产物 |
| 5 | `/workspace/out` | CMake 构建目录（.o 文件、最终二进制等编译产物） |
| 6 | `/root/.gradle` | Gradle 缓存目录（依赖 JAR 包、构建缓存、Wrapper 发行版） |

如果去掉 `/root/.gradle` 的缓存挂载：

- **每次构建都需要重新下载 Gradle Wrapper**（约 100MB 的 Gradle 发行版 zip）
- **每次构建都需要重新下载所有 Gradle 插件和依赖**（Android Gradle Plugin、Kotlin 编译器等，约 500MB）
- **Gradle 的构建缓存失效**：增量编译信息丢失，即使只改了一个文件也要全量编译 Java/Kotlin 代码
- **估计影响**：增量构建时间从 ~2 分钟增加到 ~10 分钟（主要是下载时间）

</details>

### 题目 3：假设你要为 KrKr2 添加 macOS 平台的 Docker 构建支持。请说明这个方案是否可行，并解释原因。如果不可行，提供替代方案。

<details>
<summary>查看答案</summary>

**不可行。** macOS Docker 构建在技术上有根本性限制：

1. **macOS 许可证限制**：Apple 的 EULA（End-User License Agreement，终端用户许可协议）禁止在非 Apple 硬件上运行 macOS。Docker 容器运行在 Linux 内核上，在非 Apple 硬件的 Linux 服务器上运行 macOS 容器违反许可证。

2. **内核不兼容**：Docker 容器共享宿主机的 Linux 内核。macOS 基于 Darwin/XNU 内核，无法在 Linux 内核上运行。不存在官方的 macOS Docker 镜像。

3. **Xcode 工具链限制**：macOS 编译需要 Xcode 和 Apple 的系统框架，这些都受 Apple 许可证保护，不能分发到非 Apple 平台。

**替代方案：**

1. **GitHub Actions macOS Runner**：使用 `runs-on: macos-latest`，这是官方支持的 Apple 硬件虚拟机。KrKr2 可以添加类似 `build-linux.yml` 的 `build-macos.yml` 工作流。

2. **自建 macOS CI**：购买 Mac Mini 作为 CI 构建节点，运行 self-hosted GitHub Actions Runner。

3. **交叉编译（有限）**：使用 [osxcross](https://github.com/tpoechtrager/osxcross)（一个在 Linux 上交叉编译 macOS 应用的开源工具链）在 Linux Docker 中交叉编译，但这无法处理需要 Apple 框架的代码，且不支持签名和公证。

对于 KrKr2 项目，方案 1（GitHub Actions macOS Runner）是最实际的选择。

</details>

## 下一步

下一节 [Docker 实战——本地构建与调试](./03-Docker实战-本地构建与调试.md) 将带你亲手操作：在本地使用 Docker 构建 KrKr2、从容器中提取构建产物、使用 Volume 挂载实现快速开发迭代、以及排查 Docker 构建中的各种问题。









