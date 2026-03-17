# Linux 环境搭建
> **所属模块：** M01-项目导览与环境搭建  
> **所属章节：** 03-环境搭建  
> **前置知识：** [01-Windows环境搭建](./01-Windows环境搭建.md)  
> **预计阅读时间：** 45-60 分钟  
> **适用系统：** Ubuntu 22.04/24.04、Fedora、Arch、Debian、WSL2、Docker  
> **最后更新：** 2026-03-17

## 本节目标

读完本节后，你将能够：

1. 根据 KR2 真实配置准备 Linux 开发机。
2. 在主流发行版安装完整系统依赖。
3. 在 GCC 与 Clang 之间做合理选择。
4. 配置 CMake、Ninja、vcpkg、bison、flex。
5. 用验证清单快速定位环境问题。
6. 在 WSL2 与 Docker 中复用同一流程。

## 1. 先看项目真实配置

安装环境前，先读三份文件：

- `krkr2/CMakePresets.json`
- `krkr2/vcpkg.json`
- `krkr2/.github/workflows/build-linux.yml`

你需要确认：

1. Linux preset 使用 `Ninja`。
2. Linux 常用预设是 `Linux Debug Config`、`Linux Release Config`。
3. CI 里明确安装了 `gcc g++ bison python3 nasm ninja-build`。
4. CI 里明确安装了 X11/OpenGL/GTK 开发包。
5. CI 固定 CMake 版本为 `3.31.1`。

## 2. 支持的发行版详解

| 发行版 | 特点 | 建议 |
| :--- | :--- | :--- |
| Ubuntu 22.04 LTS | 文档最多，最稳妥 | 新手与团队统一环境首选 |
| Ubuntu 24.04 LTS | 工具链更现代 | 适合长期主力开发机 |
| Fedora | GCC/Clang 更新快 | 适合偏 LLVM 工作流 |
| Arch Linux | 版本新、更新快 | 适合有经验用户 |
| Debian | 稳定保守 | 适合企业与长期维护场景 |


## 3. GCC vs Clang 选择和版本要求

- GCC 推荐 `11+`，更建议 `12/13`。
- Clang 推荐 `15+`，更建议 `16/17`。
- CMake 最低 `3.28`，建议 `3.31.1+`。

### 3.1 GCC 安装

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install -y build-essential gcc g++ gdb

# Fedora
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y gcc gcc-c++ gdb

# Arch
sudo pacman -S --needed base-devel gcc gdb
```

### 3.2 Clang 安装（可选）

```bash
# Ubuntu / Debian
sudo apt install -y clang lldb

# Fedora
sudo dnf install -y clang lldb

# Arch
sudo pacman -S --needed clang lldb
```

### 3.3 编译器验证

```bash
gcc --version
g++ --version
clang --version
clang++ --version
```

## 4. 系统依赖包安装（按发行版）

下面的依赖集合覆盖 CI 中 Linux 构建主链路。

### 4.1 Ubuntu / Debian 依赖

```bash
sudo apt update
sudo apt install -y \
  git curl wget unzip zip tar pkg-config perl python3 python3-pip \
  gcc g++ gdb bison flex nasm yasm ninja-build \
  libx11-dev libxext-dev libxft-dev libxmu-dev libxi-dev libxxf86vm-dev \
  libglu1-mesa-dev libgl1-mesa-dev libgl2ps-dev libglew-dev \
  libasound2-dev libpulse-dev libgtk-3-dev \
  libfontconfig1-dev libsqlite3-dev libzip-dev libssl-dev \
  libcurl4-gnutls-dev autoconf automake libtool libltdl-dev
```

### 4.2 Fedora 依赖

```bash
sudo dnf groupinstall -y "Development Tools" "C Development Tools and Libraries"
sudo dnf install -y \
  git curl wget unzip zip tar pkgconf-pkg-config perl python3 python3-pip \
  gcc gcc-c++ gdb bison flex nasm yasm ninja-build \
  libX11-devel libXext-devel libXft-devel libXmu-devel libXi-devel libXxf86vm-devel \
  mesa-libGL-devel mesa-libGLU-devel glew-devel alsa-lib-devel \
  pulseaudio-libs-devel gtk3-devel fontconfig-devel sqlite-devel \
  libzip-devel openssl-devel libcurl-devel autoconf automake libtool
```

### 4.3 Arch 依赖

```bash
sudo pacman -Syu --needed \
  git curl wget unzip zip tar pkgconf perl python python-pip \
  base-devel gcc gdb bison flex nasm yasm ninja cmake \
  libx11 libxext libxft libxmu libxi libxxf86vm \
  mesa glu glew alsa-lib libpulse gtk3 \
  fontconfig sqlite libzip openssl curl autoconf automake libtool
```

### 4.4 Debian 说明

Debian 大多数场景可沿用 Ubuntu 依赖清单。

## 5. CMake 安装（系统包 vs pip vs snap vs 源码）

### 5.1 方案对比

| 方案 | 优点 | 风险 |
| :--- | :--- | :--- |
| 系统包 | 稳定、易维护 | 老版本可能偏旧 |
| pip | 安装快 | PATH 可能冲突 |
| snap | 获取新版方便 | 依赖 snapd |
| 源码编译 | 灵活可控 | 编译耗时 |

### 5.2 Ubuntu（Kitware APT）

```bash
sudo apt install -y gpg wget ca-certificates
wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/dev/null \
  | gpg --dearmor - \
  | sudo tee /usr/share/keyrings/kitware-archive-keyring.gpg >/dev/null

echo 'deb [signed-by=/usr/share/keyrings/kitware-archive-keyring.gpg] https://apt.kitware.com/ubuntu/ jammy main' \
  | sudo tee /etc/apt/sources.list.d/kitware.list >/dev/null

sudo apt update
sudo apt install -y cmake
```

Ubuntu 24.04 请把 `jammy` 改为 `noble`。

### 5.3 pip 安装

```bash
python3 -m pip install --user --upgrade cmake
```

### 5.4 snap 安装

```bash
sudo snap install cmake --classic
```

### 5.5 源码编译

```bash
wget https://github.com/Kitware/CMake/releases/download/v3.31.1/cmake-3.31.1.tar.gz
tar -xzf cmake-3.31.1.tar.gz
cd cmake-3.31.1
./bootstrap
make -j"$(nproc)"
sudo make install
```

### 5.6 CMake 验证

```bash
cmake --version
which cmake
```

## 6. Ninja 安装

```bash
# Ubuntu / Debian
sudo apt install -y ninja-build

# Fedora
sudo dnf install -y ninja-build

# Arch
sudo pacman -S --needed ninja
```

验证：

```bash
ninja --version
which ninja
```

## 7. vcpkg 安装和配置

```bash
git clone https://github.com/microsoft/vcpkg.git "$HOME/vcpkg"
"$HOME/vcpkg"/bootstrap-vcpkg.sh
```

将以下内容写入 `~/.bashrc` 或 `~/.zshrc`：

```bash
export VCPKG_ROOT="$HOME/vcpkg"
export PATH="$VCPKG_ROOT:$PATH"
```

生效与验证：

```bash
source ~/.bashrc
vcpkg version
test -f "$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake" && echo "toolchain OK"
```

## 8. bison / flex 安装

```bash
# Ubuntu / Debian
sudo apt install -y bison flex

# Fedora
sudo dnf install -y bison flex

# Arch
sudo pacman -S --needed bison flex
```

验证：

```bash
bison --version
flex --version
```

## 9. Python、NASM、YASM 安装

```bash
# Ubuntu / Debian
sudo apt install -y python3 python3-pip nasm yasm

# Fedora
sudo dnf install -y python3 nasm yasm

# Arch
sudo pacman -S --needed python nasm yasm
```

验证：

```bash
python3 --version
nasm --version
yasm --version
```

## 10. 环境变量配置（.bashrc/.zshrc）

推荐写入：

```bash
export VCPKG_ROOT="$HOME/vcpkg"
export PATH="$VCPKG_ROOT:$HOME/.local/bin:$PATH"
```

生效并检查：

```bash
source ~/.bashrc
echo "$VCPKG_ROOT"
which vcpkg
```

## 11. WSL2 特殊注意事项

- 仓库建议放 Linux 文件系统，不放 `/mnt/c`。
- 脚本行尾保持 LF，权限报错时执行 `chmod +x`。

## 12. Docker 开发环境（可选）

```bash
docker build -f dockers/linux.Dockerfile -t krkr2-linux-dev .
docker run --rm -it -v "$(pwd)":/workspace/krkr2 -w /workspace/krkr2 krkr2-linux-dev bash
```

## 13. 验证命令清单

```bash
gcc --version
cmake --version
ninja --version
vcpkg version
bison --version
flex --version
python3 --version
nasm --version
```

首次构建验证：

```bash
cmake --preset="Linux Debug Config"
cmake --build --preset="Linux Debug Build"
```

## 14. 常见 Linux 环境问题

### 14.1 `X11/Xlib.h not found`

安装 `libx11-dev`（Fedora 为 `libX11-devel`）。

### 14.2 `GL/gl.h not found`

安装 `libgl1-mesa-dev` 与 `libglu1-mesa-dev`。

### 14.3 `pulse/pulseaudio.h not found`

安装 `libpulse-dev`（Fedora 为 `pulseaudio-libs-devel`）。

### 14.4 `Could not find CMAKE_MAKE_PROGRAM`

安装 Ninja 并确认 `which ninja` 有输出。

### 14.5 `Could not find toolchain file vcpkg.cmake`

检查 `VCPKG_ROOT` 是否正确，确认 `scripts/buildsystems/vcpkg.cmake` 存在。

## 15. 动手实践

> 目标：完整跑通一次 Linux Debug 构建。

```bash
mkdir -p "$HOME/workspace"
cd "$HOME/workspace"
# 已 clone 可跳过
# git clone <你的仓库地址>
cd krkr2

cmake --preset="Linux Debug Config"
cmake --build --preset="Linux Debug Build"
```

## 16. 对照项目源码

1. `krkr2/CMakePresets.json`：Linux preset、Ninja 生成器。
2. `krkr2/vcpkg.json`：依赖清单、opencv4 override。
3. `krkr2/.github/workflows/build-linux.yml`：CI Linux 依赖与构建流程。

## 17. 本节小结

- Linux 环境搭建要先读项目配置。
- 发行版不同，但核心工具链流程一致。
- CMake、Ninja、VCPKG_ROOT 是最关键检查点。

## 18. 练习题与答案

### 题目 1：为什么 Linux 必须安装 Ninja？
<details><summary>查看答案</summary>

Linux preset 指定 Ninja 为生成器，缺少 Ninja 会导致 configure 失败。

</details>

### 题目 2：出现 `X11/Xlib.h not found` 应安装什么？
<details><summary>查看答案</summary>

安装 X11 开发包：Ubuntu/Debian 常用 `libx11-dev`，Fedora 常用 `libX11-devel`。

</details>

### 题目 3：`VCPKG_ROOT` 通常写在哪个文件？
<details><summary>查看答案</summary>

写在 `~/.bashrc` 或 `~/.zshrc`，保存后执行 `source` 立即生效。

</details>

## 19. 下一步

- [03-macOS环境搭建](./03-macOS环境搭建.md)
