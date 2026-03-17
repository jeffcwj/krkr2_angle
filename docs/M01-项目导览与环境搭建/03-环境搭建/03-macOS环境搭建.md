# macOS 环境搭建

本章节将引导你在 macOS 系统上搭建 KR2 项目的开发环境。我们以 macOS 14+ (Sonoma) 和 Apple Silicon (arm64) 架构为主要推荐。

## 1. 前提条件

- 推荐系统：macOS 14 (Sonoma) 或更高版本。
- 架构：Apple Silicon (M1/M2/M3) 或 Intel。
- 基础权限：需要管理员权限。

## 2. 安装 Xcode 和 Command Line Tools

Xcode 是 macOS 下必备的开发工具包。

1.  从 App Store 下载并安装 **Xcode**。
2.  安装 Xcode 命令行工具 (Command Line Tools)：
    在终端输入：
    ```bash
    xcode-select --install
    ```
    在弹出的对话框中点击“安装”，等待下载并完成安装。

## 3. 安装 Homebrew

Homebrew 是 macOS 下最流行的包管理器，我们将用它安装大部分工具。

在终端执行以下命令安装：
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
安装完成后，根据终端提示将 Homebrew 添加到 PATH 环境。

## 4. 安装 CMake 3.31.1+

使用 Homebrew 安装最新版本的 CMake：

```bash
brew install cmake
```
输入 `cmake --version` 验证版本。

## 5. 安装 Ninja

Ninja 可以加速 CMake 的构建过程：

```bash
brew install ninja
```
输入 `ninja --version` 验证。

## 6. 安装 vcpkg

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
    macOS 默认使用 zsh，因此我们将环境变量添加到 `~/.zshrc`：
    ```bash
    export VCPKG_ROOT=$HOME/vcpkg
    export PATH=$VCPKG_ROOT:$PATH
    ```
    执行 `source ~/.zshrc` 使其生效。

## 7. 安装 bison 3.8.2+

**重要提示**：macOS 系统自带的 `/usr/bin/bison` 版本非常陈旧（通常是 2.3 版本），无法满足 KR2 的编译需求。我们必须通过 Homebrew 安装新版本，并配置 PATH 优先级。

1.  安装新版本 bison：
    ```bash
    brew install bison
    ```
2.  配置 PATH 优先级（确保 shell 优先找到 Homebrew 版本的 bison）：
    在 `~/.zshrc` 中添加：
    ```bash
    export PATH="/opt/homebrew/opt/bison/bin:$PATH"
    ```
    然后执行 `source ~/.zshrc`。

## 8. 安装 Python3

macOS 系统自带的 Python 版本可能不全，建议通过 Homebrew 安装：

```bash
brew install python3
```

## 9. 安装 NASM

为了支持汇编加速，需要安装 NASM：

```bash
brew install nasm
```

## 10. 验证工具安装

在终端执行以下命令，确保所有工具安装正确且版本符合要求：

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

## 11. 练习题与答案

1.  在 macOS 上，即使你已经运行了 `brew install bison`，终端输入 `bison --version` 时仍然显示版本为 2.3。这通常是什么原因造成的？
2.  在 Apple Silicon (M1/M2/M3) 架构的 macOS 上，Homebrew 的默认安装路径在哪里？
3.  KR2 项目在 macOS 下通常使用哪个 shell？环境变量应写在哪个配置文件中？

<details>
<summary>查看答案</summary>

1.  **回答**：这是因为 macOS 系统自带了一个非常旧版本的 bison，其路径（通常是 `/usr/bin/bison`）在环境变量 **PATH** 中的优先级高于 Homebrew 安装的路径。你需要手动将 Homebrew 的 bison 路径（例如 `/opt/homebrew/opt/bison/bin`）添加到 PATH 的最前面。
2.  **回答**：默认安装在 `/opt/homebrew` 目录下。而在传统的 Intel 架构 macOS 上，Homebrew 则默认安装在 `/usr/local` 目录下。
3.  **回答**：现代 macOS (10.15+) 默认使用 **zsh**。因此，环境变量通常应当添加到用户主目录下的 `.zshrc` 文件中。

</details>


