# Linux 环境搭建

本章节将引导你在 Linux 系统上搭建 KR2 项目的开发环境。我们以 Ubuntu 22.04/24.04 LTS 为主要示例，同时也会提及 Fedora 和 Arch Linux 的安装方法。

## 1. 前提条件

- 推荐系统：Ubuntu 22.04 LTS 或 Ubuntu 24.04 LTS。
- 基础权限：需要 `sudo` 权限以安装软件包。
- 编译工具：Linux 环境下通常使用 GCC 编译器。

## 2. 安装 GCC (build-essential)

首先更新软件包列表并安装基础构建工具。

在 Ubuntu/Debian 上运行：
```bash
sudo apt update
sudo apt install build-essential g++ gdb git
```

在 Fedora 上运行：
```bash
sudo dnf groupinstall "Development Tools" "C Development Tools and Libraries"
```

在 Arch Linux 上运行：
```bash
sudo pacman -S nasm yasm
```

## 9. 验证工具安装

在完成以上所有步骤后，请执行以下命令进行检查：

```bash
# 检查 CMake 版本
cmake --version

# 检查 Ninja 版本
ninja --version

# 检查 vcpkg
vcpkg version

# 检查 Bison
bison --version

# 检查 Python
python3 --version

# 检查 NASM
nasm --version
```

## 10. 常见问题

- **缺少 dev 包**：在 Linux 下编译一些外部依赖时（特别是通过 vcpkg 编译），可能会遇到类似 `X11/Xlib.h not found` 的错误。这通常是因为缺少系统开发库。你可能需要安装 `libx11-dev`, `libxft-dev`, `libxext-dev` 等包。
- **权限问题**：请尽量将 vcpkg 安装在用户主目录下（如 `~/vcpkg`），以避免在编译依赖时出现权限不足的问题。

## 11. 练习题与答案

1.  Linux 上通常预装了较旧版本的 CMake。如果 KR2 项目要求 CMake 3.31.1+，而你的 Ubuntu 系统自带的版本是 3.22，最推荐的升级方案是什么？
2.  在 Linux 上设置 `VCPKG_ROOT` 环境变量时，通常是在哪个文件中进行的？如何使修改立即生效？
3.  编译过程中提示 `yasm command not found`，你应该执行什么命令来修复？

<details>
<summary>查看答案</summary>

1.  **回答**：最推荐的方案是使用 Kitware 官方维护的 PPA (Personal Package Archive)，因为它能通过 `apt` 直接获取官方最新的稳定版 CMake。另外，使用 Snap 镜像也是一种非常便捷且不会影响系统原有包管理的方案。
2.  **回答**：
    - 通常在用户家目录下的 `.bashrc`（针对 bash）或 `.zshrc`（针对 zsh）文件中添加 `export VCPKG_ROOT=$HOME/vcpkg`。
    - 使其立即生效的命令是 `source ~/.bashrc`（或重启终端窗口）。
3.  **回答**：执行 `sudo apt install yasm`（针对 Ubuntu/Debian 系统）或相应的包管理器安装命令。

</details>


## 5. 安装 vcpkg

1.  **克隆 vcpkg 仓库**：
    ```bash
    git clone https://github.com/microsoft/vcpkg.git ~/vcpkg
    cd ~/vcpkg
    ```
2.  **引导 vcpkg**：
    ```bash
    ./bootstrap-vcpkg.sh
    ```
3.  **配置环境变量**：
    将以下内容添加到你的 `~/.bashrc` 或 `~/.profile` 文件末尾：
    ```bash
    export VCPKG_ROOT=$HOME/vcpkg
    export PATH=$VCPKG_ROOT:$PATH
    ```
    然后执行 `source ~/.bashrc` 使其立即生效。

## 6. 安装 bison 3.8.2+

KR2 需要较新版本的 bison 工具。

- **Ubuntu 22.04+**：
    Ubuntu 22.04 自带的 bison 版本通常已经是 3.8.2，可以直接安装：
    ```bash
    sudo apt install bison flex
    ```
- **手动编译（如果版本过旧）**：
    如果你的系统版本较低且没有高版本的 bison，可以从 GNU 官网下载源码并执行 `./configure && make && sudo make install`。

## 7. 安装 Python3

大多数现代 Linux 发行版都预装了 Python 3。如果没装，请执行：

```bash
# Ubuntu/Debian
sudo apt install python3 python3-pip

# Fedora
sudo dnf install python3

# Arch Linux
sudo pacman -S python
```

## 8. 安装 NASM 和 YASM

为了支持一些库的汇编加速优化，我们需要安装这两个汇编器：

```bash
# Ubuntu/Debian
sudo apt install nasm yasm

# Fedora
sudo dnf install nasm yasm

# Arch Linux
sudo pacman -S nasm yasm
```


## 3. 安装 CMake 3.31.1+

Ubuntu 默认软件仓库中的 CMake 版本可能较旧。为了确保 KR2 能够正确构建，我们建议安装 3.31.1 或更新版本。

- **方法 A（Kitware 官方 PPA，推荐）**：
    ```bash
    sudo apt install gpg wget
    wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/dev/null | gpg --dearmor - | sudo tee /usr/share/keyrings/kitware-archive-keyring.gpg >/dev/null
    echo 'deb [signed-by=/usr/share/keyrings/kitware-archive-keyring.gpg] https://apt.kitware.com/ubuntu/ jammy main' | sudo tee /etc/apt/sources.list.d/kitware.list >/dev/null
    sudo apt update
    sudo apt install cmake
    ```
- **方法 B（Snap 安装）**：
    ```bash
    sudo snap install cmake --classic
    ```

## 4. 安装 Ninja

Ninja 在 Linux 上通常可以通过包管理器直接安装：

```bash
# Ubuntu/Debian
sudo apt install ninja-build

# Fedora
sudo dnf install ninja-build

# Arch Linux
sudo pacman -S ninja
```
