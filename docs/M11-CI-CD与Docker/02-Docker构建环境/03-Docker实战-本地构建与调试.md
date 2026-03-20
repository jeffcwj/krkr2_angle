# Docker 实战——本地构建与调试

> **所属模块：** M11-CI/CD与Docker
> **前置知识：** [Docker 基础与 Dockerfile 语法](./01-Docker基础与Dockerfile语法.md)、[项目 Dockerfile 解析](./02-项目Dockerfile解析.md)
> **预计阅读时间：** 40 分钟

## 本节目标

读完本节后，你将能够：
1. 使用 Docker 在本地完整构建 KrKr2 的 Linux 和 Android 版本
2. 从 Docker 容器中提取构建产物（二进制文件、APK）
3. 使用 Volume 挂载实现源码修改后的快速增量构建
4. 利用 `docker run` 进入容器交互式调试构建问题
5. 使用 Docker Compose 管理多平台构建环境
6. 排查 Docker 构建过程中的网络、权限、磁盘空间等常见问题

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 构建上下文 | Build Context | `docker build` 命令末尾的 `.` 指向的目录，Docker 会把这个目录的所有文件发送给 Docker Engine 用于构建 |
| Volume 挂载 | Volume Mount | 将宿主机的目录映射到容器内，容器和宿主机共享同一份文件 |
| Bind Mount | Bind Mount | Volume 的一种类型，直接把宿主机的某个路径挂载到容器中——开发时最常用 |
| 交互式容器 | Interactive Container | 用 `docker run -it` 启动的容器，你可以在其中使用 shell 执行命令 |
| Docker Compose | Docker Compose | 用 YAML 文件定义和管理多个容器的工具，一条命令启动整个开发环境 |
| 多阶段目标 | Multi-stage Target | 用 `docker build --target <name>` 只构建 Dockerfile 中的某个阶段，跳过后续阶段 |

## 本地构建完整流程

### 环境准备

在开始之前，确认你的环境满足以下条件：

```bash
# 1. Docker 已安装（推荐 23.0+ 版本，默认启用 BuildKit）
docker --version
# Docker version 27.5.1, build 9f9e405

# 2. 磁盘空间充足
# Linux 构建需要约 5GB，Android 构建需要约 15GB
# 检查可用空间：
# Linux/macOS:
df -h /var/lib/docker
# Windows (Docker Desktop):
# Settings → Resources → Disk image size

# 3. 项目代码已获取
git clone https://github.com/jeffcwj/krkr2_angle.git krkr2
cd krkr2
```

### Windows 用户的额外注意事项

在 Windows 上使用 Docker Desktop 时，有几个特殊注意点：

```
Windows Docker Desktop 配置建议：
┌──────────────────────────┬────────────────────────────────────┐
│ 配置项                    │ 推荐值                              │
├──────────────────────────┼────────────────────────────────────┤
│ WSL 2 后端               │ 启用（Settings → General）          │
│ 内存限制                  │ 至少 4GB（Settings → Resources）    │
│ 磁盘镜像大小              │ 至少 50GB（构建 Android 需要更多）   │
│ 文件共享                  │ 确保项目目录在共享列表中             │
└──────────────────────────┴────────────────────────────────────┘
```

```powershell
# Windows PowerShell 中设置 BuildKit（如果 Docker 版本低于 23.0）
$env:DOCKER_BUILDKIT = 1

# 验证 Docker 是否通过 WSL 2 运行
wsl --list --verbose
# NAME                   STATE           VERSION
# docker-desktop         Running         2
# docker-desktop-data    Running         2
```

### macOS 用户的额外注意事项

```bash
# macOS 上 Docker Desktop 使用 HyperKit 或 Apple Virtualization 框架
# 确认虚拟化后端（推荐使用 Apple Virtualization）：
# Docker Desktop → Settings → General → "Use Virtualization framework"

# Apple Silicon (M1/M2/M3) 用户注意：
# KrKr2 的 Dockerfile 基于 x86_64 架构的 Ubuntu
# 在 ARM Mac 上构建时，Docker 会自动使用 QEMU 模拟 x86_64
# 这会导致构建速度慢 3-5 倍
# 可以通过 --platform 明确指定：
docker build --platform linux/amd64 -f dockers/linux.Dockerfile -t krkr2-linux .
```

## 构建 Linux 版本

### 一键构建

```bash
# 在项目根目录执行
# -f 指定 Dockerfile 路径（相对于当前目录）
# -t 给镜像命名为 krkr2-linux，标签为 latest
# . 是构建上下文（Build Context），Docker 会把当前目录的所有文件发送给 Docker Engine
docker build -f dockers/linux.Dockerfile -t krkr2-linux .
```

构建输出解读：

```
[+] Building 1523.4s (12/12) FINISHED                docker:default
 => [internal] load build definition from linux.Dockerfile      0.0s
 => [internal] load .dockerignore                               0.0s
 => [internal] load metadata for docker.io/library/ubuntu:22.04 1.2s
 => [build-tools 1/4] FROM ubuntu:22.04@sha256:abc123...       15.3s
 => [build-tools 2/4] RUN apt-get update && apt-get install... 45.6s
 => [build-tools 3/4] RUN curl -sSL ... | tar -xz ...          8.2s
 => [build-tools 4/4] RUN git clone ... vcpkg ...              22.1s
 => [generate 1/3] WORKDIR /workspace/krkr2                     0.1s
 => [generate 2/3] COPY . .                                     3.4s
 => [generate 3/3] RUN --mount=type=cache ... cmake ...      1427.5s  ← 主要耗时
 => exporting to image                                          0.1s

 ↑ 每行前面的标签表示阶段名和步骤序号：
 [build-tools 1/4] = build-tools 阶段的第 1 步，共 4 步
 [generate 3/3]    = generate 阶段的第 3 步，共 3 步
```

### 只构建工具链镜像（不编译代码）

利用多阶段目标（Multi-stage Target），可以只构建到 `build-tools` 阶段：

```bash
# --target build-tools 告诉 Docker 只构建到 build-tools 阶段就停止
# 不执行 generate 阶段（不复制源码、不编译）
docker build --target build-tools \
    -f dockers/linux.Dockerfile \
    -t krkr2-toolchain .

# 这个镜像包含：Ubuntu 22.04 + GCC + CMake 3.31.6 + vcpkg + Ninja
# 但不包含项目源码和编译产物
# 用途：可以作为基础镜像供其他项目使用，或用于交互式调试
docker images krkr2-toolchain
# REPOSITORY        TAG       IMAGE ID       CREATED          SIZE
# krkr2-toolchain   latest    xyz789abc012   10 seconds ago   ~1.2GB
```

### 从容器中提取构建产物

Docker 构建完成后，编译好的二进制文件在镜像内部。要把它取出来：

```bash
# 方法 1：使用 docker cp（最简单）
# 先用镜像创建一个临时容器（不运行任何命令）
docker create --name krkr2-extract krkr2-linux

# 将容器内的构建产物复制到宿主机
# 假设构建产物在 /workspace/out/linux/release/ 目录下
docker cp krkr2-extract:/workspace/out/ ./docker-output/

# 查看提取的文件
ls -la ./docker-output/
# 应该能看到编译好的 krkr2 二进制文件

# 清理临时容器
docker rm krkr2-extract
```

```bash
# 方法 2：使用 docker run 挂载 Volume 直接输出
# 将宿主机的 ./output 目录挂载到容器的 /output
# 在容器内把构建产物复制到 /output，就等于复制到了宿主机
docker run --rm \
    -v "$(pwd)/output:/output" \
    krkr2-linux \
    cp -r /workspace/out/ /output/

# 查看输出
ls -la ./output/
```

```bash
# 方法 3：多阶段 + COPY --from（推荐用于 CI）
# 创建一个专门的 "提取" Dockerfile
cat > Dockerfile.extract << 'EOF'
# 引用已构建好的镜像
FROM krkr2-linux AS builder

# 用一个最小镜像只取需要的文件
FROM alpine:3.19
COPY --from=builder /workspace/out/linux/release/krkr2 /app/krkr2
# 此镜像只有几 MB，只包含最终的二进制文件
CMD ["/app/krkr2"]
EOF

docker build -f Dockerfile.extract -t krkr2-runtime .
# 这个镜像只有几 MB，非常适合分发
```

## 构建 Android 版本

### 完整构建 APK

```bash
# 构建 Android 镜像
docker build -f dockers/android.Dockerfile -t krkr2-android .

# 构建完成后，APK 在容器内的 Gradle 输出目录中
# 提取 APK 文件
docker create --name android-extract krkr2-android
docker cp android-extract:/workspace/krkr2/platforms/android/app/build/outputs/apk/release/ ./apk-output/
docker rm android-extract

# 查看 APK
ls -la ./apk-output/
# app-release-unsigned.apk  (未签名的 APK)
```

### 使用自定义签名构建

```bash
# Android APK 需要签名才能安装到设备上
# 可以通过 ARG 传入签名相关参数
# 注意：项目 CI 在运行时动态生成 keystore，本地构建也可以这样做

# 生成一个测试用的 keystore
keytool -genkey -v \
    -keystore debug.keystore \
    -storepass android \
    -alias androiddebugkey \
    -keypass android \
    -keyalg RSA \
    -keysize 2048 \
    -validity 10000 \
    -dname "CN=Debug, OU=Debug, O=Debug, L=Debug, S=Debug, C=US"

# 对 APK 签名
# apksigner 需要 Android SDK Build Tools
# 如果本地没有 Android SDK，可以进入容器内签名
docker run --rm \
    -v "$(pwd)/apk-output:/apk" \
    -v "$(pwd)/debug.keystore:/keystore/debug.keystore" \
    krkr2-android \
    bash -c '/opt/android-sdk/build-tools/35.0.0/apksigner sign \
        --ks /keystore/debug.keystore \
        --ks-pass pass:android \
        /apk/app-release-unsigned.apk'
```

## 使用 Volume 挂载加速开发迭代

每次修改代码后重新 `docker build` 会重新执行 `COPY . .`，虽然有缓存挂载加速编译，但构建上下文传输和层校验仍需时间。使用 Volume 挂载（Bind Mount）可以跳过 `COPY . .`，让容器直接访问宿主机的源代码目录。

### 原理对比

```
传统 docker build 流程：
  修改代码 → docker build → COPY . . → cmake build → 提取产物
             ↑ 需要传输整个构建上下文（数百 MB）

Volume 挂载开发流程：
  修改代码 → 容器内直接 cmake build → 产物直接在宿主机可见
             ↑ 零传输，直接读取宿主机文件
```

### 用工具链镜像 + Volume 挂载做增量开发

```bash
# 第一步：构建工具链镜像（只需执行一次）
docker build --target build-tools \
    -f dockers/linux.Dockerfile \
    -t krkr2-toolchain .

# 第二步：启动交互式容器，挂载源码目录
# -it：交互模式 + 终端（Interactive + TTY）
# -v：将宿主机的项目目录挂载到容器的 /workspace/krkr2
# --name：给容器命名，方便后续操作
docker run -it \
    --name krkr2-dev \
    -v "$(pwd):/workspace/krkr2" \
    -w /workspace/krkr2 \
    krkr2-toolchain \
    bash

# 现在你在容器的 bash 中，可以直接执行构建命令
# 容器内的 /workspace/krkr2 就是宿主机的项目目录

# 第三步：在容器内执行构建
cmake --preset="Linux Debug Config" \
    -DENABLE_TESTS=ON
cmake --build --preset="Linux Debug Build"

# 构建产物会出现在宿主机的 out/ 目录中（因为 Volume 挂载是双向的）

# 第四步：修改代码后，直接在容器内重新构建（增量编译，通常几秒钟）
# 在宿主机编辑 cpp/core/base/XP3Archive.cpp
# 回到容器终端：
cmake --build --preset="Linux Debug Build"
# 只重新编译改动的文件，非常快

# 退出容器
exit

# 下次继续使用同一个容器
docker start -ai krkr2-dev
```

### Windows 上使用 Volume 挂载的性能注意事项

在 Windows + WSL 2 + Docker Desktop 的组合下，Volume 挂载的性能取决于文件存储的位置：

```
性能对比（Windows Docker Desktop + WSL 2）：
┌────────────────────────────┬───────────────────┬─────────────┐
│ 文件位置                    │ I/O 性能           │ 推荐程度     │
├────────────────────────────┼───────────────────┼─────────────┤
│ WSL 2 文件系统内            │ ★★★★★ 原生速度    │ 强烈推荐     │
│ (如 /home/user/krkr2)     │                   │             │
├────────────────────────────┼───────────────────┼─────────────┤
│ Windows NTFS 盘            │ ★★☆☆☆ 慢 3-10 倍 │ 不推荐       │
│ (如 /mnt/c/Users/.../krkr2)│                   │             │
└────────────────────────────┴───────────────────┴─────────────┘
```

```powershell
# 推荐做法：把项目放在 WSL 2 文件系统中
# 在 WSL 2 终端中：
cd ~
git clone https://github.com/jeffcwj/krkr2_angle.git krkr2
cd krkr2

# 然后在 WSL 2 终端中运行 Docker 命令
docker run -it -v "$(pwd):/workspace/krkr2" krkr2-toolchain bash
```

## 交互式调试构建问题

当 `docker build` 失败时，你需要进入容器内部调查原因。以下是几种调试方法：

### 方法 1：进入失败步骤之前的状态

```bash
# 假设 docker build 在 generate 阶段的 cmake 步骤失败了
# 可以用 --target 只构建到 build-tools 阶段
docker build --target build-tools \
    -f dockers/linux.Dockerfile \
    -t krkr2-debug .

# 然后进入这个镜像，手动执行后续步骤
docker run -it \
    -v "$(pwd):/workspace/krkr2" \
    -w /workspace/krkr2 \
    krkr2-debug bash

# 在容器内手动执行 cmake，观察详细错误输出
cmake --preset="Linux Release Config" \
    -DENABLE_TESTS=OFF \
    -DBUILD_TOOLS=OFF \
    --log-level=DEBUG 2>&1 | tee /tmp/cmake-output.log

# 如果 cmake 配置成功，再尝试构建
cmake --build --preset="Linux Release Build" --verbose 2>&1 | tee /tmp/build-output.log

# 查看具体编译错误
grep -n "error:" /tmp/build-output.log
```

### 方法 2：使用 docker exec 附加到运行中的容器

```bash
# 如果你有一个正在运行的容器（比如前面创建的 krkr2-dev）
docker exec -it krkr2-dev bash

# 现在你有两个终端：
# 终端 1：正在运行构建
# 终端 2：可以实时查看构建日志、检查文件系统等

# 查看 vcpkg 安装了哪些包
ls /opt/vcpkg/installed/x64-linux/lib/
# 应该能看到 libavcodec.a、libSDL2.a 等

# 查看 cmake 缓存
cat /workspace/out/linux/release/CMakeCache.txt | grep VCPKG
# VCPKG_TARGET_TRIPLET:STRING=x64-linux
# CMAKE_TOOLCHAIN_FILE:STRING=/opt/vcpkg/scripts/buildsystems/vcpkg.cmake

# 检查编译器版本
gcc --version
cmake --version
ninja --version
```

### 方法 3：查看构建中间文件

```bash
# 进入容器后，检查关键目录

# vcpkg 下载缓存
ls /opt/vcpkg/downloads/
# ffmpeg-n7.1.tar.gz、SDL2-2.30.10.tar.gz 等

# vcpkg 构建日志（排查依赖编译失败）
ls /opt/vcpkg/buildtrees/ffmpeg/
# config-x64-linux-dbg-out.log、build-x64-linux-dbg-out.log 等
cat /opt/vcpkg/buildtrees/ffmpeg/build-x64-linux-rel-err.log

# CMake 构建产物
ls /workspace/out/linux/release/
# CMakeCache.txt、build.ninja、compile_commands.json 等
```

## Docker Compose 管理多平台构建

当你需要同时管理 Linux 和 Android 的构建环境时，Docker Compose 可以用一个 YAML 文件统一管理：

### 编写 docker-compose.yml

```yaml
# docker-compose.yml（放在项目根目录）
# Docker Compose 配置文件，定义多个构建服务

version: "3.9"  # Compose 文件格式版本

services:
  # Linux 构建服务
  linux-build:
    build:
      context: .                              # 构建上下文 = 项目根目录
      dockerfile: dockers/linux.Dockerfile    # 指定 Dockerfile
    image: krkr2-linux:latest                 # 构建后的镜像名
    volumes:
      - ./output/linux:/output                # 挂载输出目录
    # 构建完成后将产物复制到 /output
    command: >
      bash -c "cp -r /workspace/out/linux/release/* /output/ 2>/dev/null || echo 'No build output found'"

  # Android 构建服务
  android-build:
    build:
      context: .
      dockerfile: dockers/android.Dockerfile
    image: krkr2-android:latest
    volumes:
      - ./output/android:/output
    command: >
      bash -c "cp -r /workspace/krkr2/platforms/android/app/build/outputs/apk/release/* /output/ 2>/dev/null || echo 'No APK found'"

  # 开发环境服务（交互式）
  dev:
    build:
      context: .
      dockerfile: dockers/linux.Dockerfile
      target: build-tools                     # 只构建到 build-tools 阶段
    image: krkr2-toolchain:latest
    volumes:
      - .:/workspace/krkr2                    # 挂载源码目录（实时同步）
    working_dir: /workspace/krkr2
    stdin_open: true                          # 等同于 docker run -i
    tty: true                                 # 等同于 docker run -t
    command: bash                             # 启动 bash shell
```

### 使用 Docker Compose

```bash
# 构建所有服务的镜像
docker compose build

# 只构建 Linux 版本
docker compose build linux-build

# 运行 Linux 构建并提取产物
docker compose run --rm linux-build
# 产物会出现在 ./output/linux/ 目录中

# 同时构建 Linux 和 Android（并行）
docker compose up linux-build android-build

# 启动开发环境
docker compose run --rm dev
# 进入交互式 bash，可以手动执行 cmake 命令

# 清理所有容器和镜像
docker compose down --rmi all
```

### 使用 Compose Profiles 按需启动

```yaml
# 增强版 docker-compose.yml（使用 profiles）
services:
  linux-build:
    profiles: ["build", "linux"]
    # ... 同上

  android-build:
    profiles: ["build", "android"]
    # ... 同上

  dev:
    profiles: ["dev"]
    # ... 同上
```

```bash
# 只启动 linux 相关服务
docker compose --profile linux up

# 只启动开发环境
docker compose --profile dev run --rm dev

# 启动所有构建服务
docker compose --profile build up
```

## 网络问题排查

Docker 构建中最常见的问题之一是网络——下载超时、DNS 解析失败、代理配置等。

### 配置代理

```bash
# 方法 1：通过 --build-arg 传入（临时）
docker build \
    --build-arg http_proxy=http://proxy.example.com:7890 \
    --build-arg https_proxy=http://proxy.example.com:7890 \
    --build-arg no_proxy=localhost,127.0.0.1 \
    -f dockers/linux.Dockerfile -t krkr2-linux .

# 方法 2：Docker Desktop 全局代理配置（永久）
# Docker Desktop → Settings → Resources → Proxies
# HTTP Proxy:  http://proxy.example.com:7890
# HTTPS Proxy: http://proxy.example.com:7890
# No Proxy:    localhost,127.0.0.1

# 方法 3：Docker daemon 配置（Linux 服务器）
# 创建或编辑 /etc/systemd/system/docker.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:7890"
Environment="HTTPS_PROXY=http://proxy.example.com:7890"
Environment="NO_PROXY=localhost,127.0.0.1"

# 重新加载配置
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### DNS 解析问题

```bash
# 如果构建中出现 "Could not resolve host: github.com" 等错误
# 检查 Docker 的 DNS 配置

# 查看容器内的 DNS 设置
docker run --rm ubuntu:22.04 cat /etc/resolv.conf

# 手动指定 DNS 服务器
docker build \
    --network host \
    -f dockers/linux.Dockerfile -t krkr2-linux .
# --network host 让容器使用宿主机的网络栈（包括 DNS）

# 或者在 Docker daemon 配置中指定 DNS
# /etc/docker/daemon.json
{
    "dns": ["8.8.8.8", "8.8.4.4"]
}
```

### 下载超时处理

```bash
# vcpkg 下载大型依赖（如 FFmpeg）可能超时
# 可以预先下载到宿主机，然后通过 Volume 挂载传入

# 第一步：查看 vcpkg 需要下载的文件
# 在容器内执行：
vcpkg install --dry-run --triplet x64-linux
# 输出会列出需要下载的 URL

# 第二步：在宿主机用更快的工具下载
mkdir -p vcpkg-downloads
wget -P vcpkg-downloads/ https://github.com/FFmpeg/FFmpeg/archive/refs/tags/n7.1.tar.gz

# 第三步：挂载下载目录到容器
docker run -it \
    -v "$(pwd):/workspace/krkr2" \
    -v "$(pwd)/vcpkg-downloads:/opt/vcpkg/downloads" \
    krkr2-toolchain bash
# vcpkg 会优先使用 downloads/ 目录中的文件，跳过下载
```

## 动手实践

### 实践 1：完整的 Linux 本地构建与产物提取

从零开始，走通 Linux Docker 构建全流程：

```bash
# 0. 准备工作
cd /path/to/krkr2
mkdir -p output/linux

# 1. 构建镜像（首次约 25 分钟，后续约 1-3 分钟）
docker build -f dockers/linux.Dockerfile -t krkr2-linux .

# 2. 创建临时容器并提取产物
docker create --name linux-extract krkr2-linux
docker cp linux-extract:/workspace/out/ ./output/linux/
docker rm linux-extract

# 3. 验证产物
ls -la ./output/linux/
# 应该能看到 linux/release/ 目录下的二进制文件

# 4. 查看构建信息
docker inspect krkr2-linux --format='{{.Created}}'
# 输出镜像创建时间
docker image inspect krkr2-linux --format='{{.Size}}' | numfmt --to=iec
# 输出镜像大小（人类可读格式）
```

### 实践 2：使用开发容器做增量编译

```bash
# 1. 构建工具链镜像
docker build --target build-tools \
    -f dockers/linux.Dockerfile -t krkr2-toolchain .

# 2. 启动开发容器
docker run -it --name krkr2-dev \
    -v "$(pwd):/workspace/krkr2" \
    -w /workspace/krkr2 \
    krkr2-toolchain bash

# === 在容器内 ===

# 3. 首次完整构建
cmake --preset="Linux Debug Config" -DENABLE_TESTS=ON
cmake --build --preset="Linux Debug Build"

# 4. 运行测试
ctest --test-dir out/linux/debug --output-on-failure

# 5. 模拟代码修改（在宿主机编辑器中修改文件）
# 然后在容器内增量构建：
cmake --build --preset="Linux Debug Build"
# 只编译改动涉及的文件，通常几秒钟完成

# 6. 退出容器
exit

# 7. 下次继续使用
docker start -ai krkr2-dev
```

### 实践 3：清理 Docker 资源

```bash
# 查看 Docker 总体磁盘使用
docker system df

# 清理已停止的容器
docker container prune
# WARNING! This will remove all stopped containers.

# 清理悬空镜像（没有标签的中间层镜像）
docker image prune

# 清理构建缓存
docker builder prune
# 注意：这会删除 --mount=type=cache 的缓存数据！
# 下次构建将从零开始编译 vcpkg 依赖

# 核弹选项：清理所有未使用的资源
docker system prune -a --volumes
# 这会删除所有停止的容器、所有未使用的镜像、所有未使用的 volume
# 慎用！
```

## 对照项目源码

本节操作涉及的项目文件：

```
项目文件与 Docker 实战操作的关联：

┌──────────────────────────────┬───────────────────────────────────────────┐
│ 文件路径                      │ 在本节中的使用场景                         │
├──────────────────────────────┼───────────────────────────────────────────┤
│ dockers/linux.Dockerfile      │ docker build 命令直接引用                  │
│ (62 行)                      │ --target build-tools 构建工具链镜像        │
├──────────────────────────────┼───────────────────────────────────────────┤
│ dockers/android.Dockerfile    │ docker build 构建 Android 镜像             │
│ (67 行)                      │ 提取 APK 产物                              │
├──────────────────────────────┼───────────────────────────────────────────┤
│ CMakePresets.json             │ 容器内 cmake --preset 引用                 │
│ (156 行)                     │ "Linux Debug Config" 等预设                │
├──────────────────────────────┼───────────────────────────────────────────┤
│ scripts/build-linux.sh        │ 参考脚本，Docker 构建方式的替代方案         │
│                              │ 适用于不使用 Docker 的本地构建              │
├──────────────────────────────┼───────────────────────────────────────────┤
│ .github/workflows/            │ CI 使用不同的缓存策略（actions/cache）      │
│ build-linux.yml 等           │ 与 Docker 的 --mount=type=cache 形成对比   │
└──────────────────────────────┴───────────────────────────────────────────┘
```

### Docker 构建 vs 本地脚本构建

项目提供了两种构建方式，对比如下：

```
┌─────────────────────┬──────────────────────────┬──────────────────────────┐
│ 对比维度             │ Docker 构建               │ scripts/ 脚本构建         │
├─────────────────────┼──────────────────────────┼──────────────────────────┤
│ 环境要求             │ 只需 Docker               │ 需要本地安装所有工具链     │
│ 环境一致性           │ 完全一致（镜像固定）       │ 依赖本地系统版本          │
│ 首次构建速度         │ 慢（需要下载镜像+编译）    │ 快（直接使用本地工具）     │
│ 增量构建速度         │ 快（缓存挂载）             │ 快（本地缓存）            │
│ 磁盘占用             │ 大（镜像 1.5-4.5GB）      │ 小（只有编译产物）         │
│ 跨平台               │ 可以在 Linux 上构建        │ 只能在对应平台构建         │
│                     │ Android 版本              │                          │
│ 团队协作             │ 环境一致，避免"我机器能跑" │ 需要每人手动配置环境       │
│ CI 适用性            │ 可直接用于 CI 流水线       │ CI 也可用但环境配置复杂    │
└─────────────────────┴──────────────────────────┴──────────────────────────┘
```

## 常见错误与排查

### 错误 1：Volume 挂载后权限错误

```
错误信息：
cmake: /workspace/krkr2/out/linux/debug/CMakeCache.txt:
Permission denied

原因：容器内以 root 身份运行，但挂载的文件属于宿主机用户，
或反过来——容器内创建的文件在宿主机上属于 root。
```

```bash
# 解决方案 1：指定容器内的用户 ID 与宿主机一致
docker run -it \
    --user "$(id -u):$(id -g)" \
    -v "$(pwd):/workspace/krkr2" \
    krkr2-toolchain bash

# 解决方案 2：在容器内修改目录权限
docker run -it \
    -v "$(pwd):/workspace/krkr2" \
    krkr2-toolchain bash -c "
        chown -R $(id -u):$(id -g) /workspace/krkr2/out 2>/dev/null
        exec bash
    "

# 解决方案 3：构建完成后在宿主机修改权限
sudo chown -R $(whoami) ./out/
```

### 错误 2：Apple Silicon 架构不匹配

```
错误信息：
exec format error

原因：在 ARM64 Mac 上运行了 x86_64 架构的容器，且 QEMU 模拟未正确配置。
```

```bash
# 解决方案：确保安装了 QEMU 多架构支持
# Docker Desktop for Mac 通常自带，但有时需要更新

# 检查是否支持 x86_64 模拟
docker run --rm --platform linux/amd64 ubuntu:22.04 uname -m
# 应输出：x86_64

# 如果不支持，尝试安装 binfmt
docker run --privileged --rm tonistiigi/binfmt --install all

# 然后明确指定平台
docker build --platform linux/amd64 \
    -f dockers/linux.Dockerfile -t krkr2-linux .
```

### 错误 3：Windows 路径格式问题

```
错误信息：
error during connect: ... the docker command must be run elevated

或：
invalid mount config: mount path must be absolute
```

```powershell
# Windows 上 Volume 挂载需要使用正确的路径格式

# PowerShell 正确写法：
docker run -it -v "${PWD}:/workspace/krkr2" krkr2-toolchain bash

# CMD 正确写法：
docker run -it -v "%cd%:/workspace/krkr2" krkr2-toolchain bash

# Git Bash 正确写法（需要双斜杠前缀）：
docker run -it -v "/$(pwd):/workspace/krkr2" krkr2-toolchain bash
# 或使用 MSYS_NO_PATHCONV 环境变量
MSYS_NO_PATHCONV=1 docker run -it -v "$(pwd):/workspace/krkr2" krkr2-toolchain bash
```

## 本节小结

- **完整构建流程**：`docker build -f dockers/linux.Dockerfile -t krkr2-linux .` 一条命令即可在干净环境中构建项目
- **产物提取**：使用 `docker create` + `docker cp` 从镜像中提取编译好的二进制文件或 APK
- **多阶段目标**：`--target build-tools` 可以只构建工具链镜像，跳过编译步骤
- **Volume 挂载开发**：将宿主机源码挂载到容器内，实现零传输的增量编译
- **Windows 性能陷阱**：WSL 2 环境下应将项目放在 WSL 文件系统内（`/home/`），而非 Windows 盘（`/mnt/c/`）
- **Docker Compose**：用 YAML 文件统一管理 Linux、Android 构建服务和开发环境
- **网络问题**：代理配置通过 `--build-arg` 或 Docker daemon 设置；DNS 问题用 `--network host` 解决
- **权限问题**：Volume 挂载时使用 `--user "$(id -u):$(id -g)"` 避免权限不匹配
- **资源清理**：定期执行 `docker system prune` 释放磁盘空间

## 练习题与答案

### 题目 1：你需要在本地修改 KrKr2 的 `cpp/core/visual/LoadPNG.cpp` 文件并验证编译通过。请描述使用 Docker 开发容器完成此操作的完整步骤，要求增量编译（不重新编译所有文件）。

<details>
<summary>查看答案</summary>

完整步骤：

```bash
# 1. 确保工具链镜像已构建
docker build --target build-tools \
    -f dockers/linux.Dockerfile -t krkr2-toolchain .

# 2. 启动开发容器，挂载源码
docker run -it --name krkr2-dev \
    -v "$(pwd):/workspace/krkr2" \
    -w /workspace/krkr2 \
    krkr2-toolchain bash

# 3. 在容器内执行首次完整构建（建立编译数据库）
cmake --preset="Linux Debug Config" -DENABLE_TESTS=OFF
cmake --build --preset="Linux Debug Build"

# 4. 退出容器（保持容器存在）
exit

# 5. 在宿主机使用你喜欢的编辑器修改 LoadPNG.cpp
# 例如：vim cpp/core/visual/LoadPNG.cpp

# 6. 重新进入容器
docker start -ai krkr2-dev

# 7. 增量编译（只重新编译 LoadPNG.cpp 及其依赖）
cmake --build --preset="Linux Debug Build"
# 输出应该显示只编译了 1-2 个文件，耗时几秒钟

# 8. 如果编译成功，说明修改没有引入编译错误
echo $?
# 0 表示成功
```

关键点：
- Volume 挂载使容器和宿主机共享文件——在宿主机修改的文件立即反映在容器中
- CMake 的增量编译基于文件修改时间，不需要重新 configure，直接 `cmake --build` 即可
- 使用 `docker start -ai` 复用已有容器，不需要每次创建新容器

</details>

### 题目 2：为什么在 Windows + Docker Desktop (WSL 2) 环境下，把项目放在 `/mnt/c/Users/...` 目录中使用 Volume 挂载会导致构建速度慢 3-10 倍？请从文件系统层面解释原因，并给出解决方案。

<details>
<summary>查看答案</summary>

**原因分析：**

WSL 2 的文件系统架构导致了跨文件系统访问的性能瓶颈：

1. **WSL 2 使用轻量级 VM**：WSL 2 运行在 Hyper-V 虚拟机中，有自己的 ext4 文件系统
2. **`/mnt/c/` 是 9P 协议挂载**：Windows 的 NTFS 分区通过 9P 网络文件系统协议（一种将远程文件系统挂载为本地目录的网络协议）挂载到 WSL 2 中
3. **Docker 容器运行在 WSL 2 内**：Docker Desktop 使用 WSL 2 作为后端
4. **跨文件系统链路**：容器 → WSL 2 ext4 → 9P 协议 → Windows NTFS

每次文件 I/O 操作（读取头文件、写入 .o 文件、检查文件修改时间等）都需要经过 9P 协议的网络转换。C++ 编译过程中有大量的文件 I/O（一个 .cpp 文件可能 include 数百个头文件），因此性能损失巨大。

```
文件访问路径对比：

项目在 /mnt/c/（慢）：
  容器进程 → WSL 2 VFS → 9P Client → Hyper-V → 9P Server → Windows NTFS
                                        ↑ 瓶颈：网络协议转换

项目在 /home/user/（快）：
  容器进程 → WSL 2 VFS → ext4 文件系统
                          ↑ 直接访问，无协议转换
```

**解决方案：**

```bash
# 方案 1（推荐）：将项目克隆到 WSL 2 内部文件系统
# 在 WSL 2 终端中：
cd ~
git clone https://github.com/jeffcwj/krkr2_angle.git krkr2
cd krkr2
docker run -it -v "$(pwd):/workspace/krkr2" krkr2-toolchain bash

# 方案 2：使用 Docker Named Volume（中等速度）
docker volume create krkr2-src
docker run --rm -v krkr2-src:/workspace/krkr2 -v "$(pwd):/host" \
    ubuntu:22.04 cp -r /host/. /workspace/krkr2/
# Named Volume 存储在 Docker 管理的 WSL 2 文件系统中

# 方案 3：使用 Docker 的 delegated 一致性选项（macOS 适用，WSL 2 无效）
# docker run -v "$(pwd):/workspace:delegated" ...
# 注意：这对 WSL 2 无效，因为瓶颈在 9P 协议而非缓存一致性
```

</details>

### 题目 3：假设你的团队决定为 KrKr2 项目创建一个 `docker-compose.yml` 文件。请编写一个包含三个服务的 Compose 文件：(1) Linux Release 构建、(2) Linux Debug 构建 + 测试、(3) 开发环境。要求使用 profiles 隔离构建和开发服务。

<details>
<summary>查看答案</summary>

```yaml
# docker-compose.yml
version: "3.9"

services:
  # 服务 1：Linux Release 构建
  linux-release:
    profiles: ["build", "release"]
    build:
      context: .
      dockerfile: dockers/linux.Dockerfile
    image: krkr2-linux-release:latest
    volumes:
      - ./output/release:/output
    command: >
      bash -c "
        mkdir -p /output &&
        cp -r /workspace/out/linux/release/* /output/ 2>/dev/null &&
        echo 'Release build artifacts copied to ./output/release/' ||
        echo 'No release output found'
      "

  # 服务 2：Linux Debug 构建 + 测试
  linux-debug-test:
    profiles: ["build", "test"]
    build:
      context: .
      dockerfile: dockers/linux.Dockerfile
      target: build-tools
    image: krkr2-toolchain:latest
    volumes:
      - .:/workspace/krkr2
      - ./output/debug:/output
    working_dir: /workspace/krkr2
    command: >
      bash -c "
        cmake --preset='Linux Debug Config' -DENABLE_TESTS=ON &&
        cmake --build --preset='Linux Debug Build' &&
        ctest --test-dir out/linux/debug --output-on-failure &&
        cp -r out/linux/debug/* /output/ 2>/dev/null &&
        echo 'Debug build + tests passed!'
      "

  # 服务 3：开发环境（交互式）
  dev:
    profiles: ["dev"]
    build:
      context: .
      dockerfile: dockers/linux.Dockerfile
      target: build-tools
    image: krkr2-toolchain:latest
    volumes:
      - .:/workspace/krkr2
    working_dir: /workspace/krkr2
    stdin_open: true
    tty: true
    command: bash

# 使用方法：
# 构建 Release 版本：    docker compose --profile release up linux-release
# Debug 构建 + 测试：    docker compose --profile test up linux-debug-test
# 启动开发环境：          docker compose --profile dev run --rm dev
# 构建所有：              docker compose --profile build up
```

**设计说明：**
- `linux-release` 使用完整的 Dockerfile（两阶段构建），产物在镜像内
- `linux-debug-test` 使用 `target: build-tools`，通过 Volume 挂载源码在容器内构建和测试
- `dev` 也用 `target: build-tools`，提供交互式 bash 环境
- profiles 隔离确保 `docker compose up` 不会意外启动所有服务

</details>

## 下一步

Docker 构建环境到此介绍完毕。下一章 [clang-format 代码格式化](../03-代码质量门禁/01-clang-format代码格式化.md) 将讲解 KrKr2 如何用 clang-format 统一代码风格，以及 CI 中的格式检查门禁机制。








