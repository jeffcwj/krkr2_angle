# macOS 环境搭建
> **所属模块：** M01-项目导览与环境搭建  
> **所属章节：** 03-环境搭建  
> **前置知识：** [01-Windows环境搭建](./01-Windows环境搭建.md)、[02-Linux环境搭建](./02-Linux环境搭建.md)  
> **下一节：** [04-Android环境搭建](./04-Android环境搭建.md)  
> **预计阅读时间：** 60-80 分钟  
> **适用系统：** macOS 13/14/15（Intel 与 Apple Silicon）  
> **最后更新：** 2026-03-17

## 本节目标
读完本节后，你将能够：
1. 根据 KR2 仓库配置明确 macOS 构建要求，而不是凭经验安装。
2. 完成 Xcode、Command Line Tools、Homebrew 的安装与验证。
3. 对比并选择 CMake 安装方式（brew / dmg / 源码）。
4. 完成 Ninja、vcpkg、bison、flex、Python、NASM 的安装与配置。
5. 理解 Apple Silicon 与 Intel 在路径、架构、二进制产物上的差异。
6. 理解 Framework 路径、Code Signing、Universal Binary 的基本概念。
7. 使用项目预设完成首次 configure/build，并具备常见问题排查能力。

## 0. 先对齐项目真实配置（必须）
先读取以下三处配置，再开始安装：
- `krkr2/CMakePresets.json`
- `krkr2/vcpkg.json`
- `krkr2/.github/workflows/`

### 0.1 `CMakePresets.json` 的 macOS 关键信息
你可以确认到以下事实：
1. macOS 预设名是 `MacOS Debug Config`、`MacOS Release Config`。
2. 生成器是 `Ninja`。
3. 编译器是 `clang` 与 `clang++`。
4. 生效条件是 `hostSystemName == Darwin`。
5. Debug 输出目录是 `out/macos/debug`。
6. Release 输出目录是 `out/macos/release`。
7. 对应 build 预设为 `MacOS Debug Build` 与 `MacOS Release Build`。

### 0.2 `vcpkg.json` 的含义
该项目使用 vcpkg manifest 模式，依赖中包含 `ffmpeg`、`cocos2dx`、`openal-soft`、`spdlog`、`libarchive`、`opencv4` 等。
并且有 `osx` 平台条件条目。
这意味着：
1. configure 阶段会自动处理依赖。
2. 首次构建时间长是正常现象。
3. `VCPKG_ROOT` 与 PATH 错误会直接导致失败。

### 0.3 `.github/workflows/` 的现状
当前仓库有 Windows、Linux、Android 工作流，没有独立 `build-macos.yml`。
这不代表 macOS 不支持，而是代表本地环境验证要做得更严格。

## 1. macOS 版本要求和兼容性
### 1.1 推荐版本
- 推荐：macOS 14（Sonoma）/ 15（Sequoia）
- 可用：macOS 13（Ventura）
- 不建议：macOS 12 及更低

### 1.2 为什么建议新版本
1. 系统 SDK 更完整。
2. Apple Clang 更现代，C++17 兼容性更稳。
3. Homebrew 与 vcpkg 在新系统遇到的兼容问题更少。

### 1.3 资源建议
- 磁盘可用空间建议 ≥ 35GB。
- 内存建议 ≥ 16GB。
- 首次依赖拉取时建议稳定网络。

## 2. Xcode 与 Command Line Tools
### 2.1 安装 Xcode
1. 从 App Store 安装 Xcode。
2. 首次启动并接受协议。
3. 确认 Xcode 可正常启动一次。

### 2.2 安装 CLT
```bash
xcode-select --install
```

### 2.3 验证工具链
```bash
xcode-select -p
clang --version
```
如果 `clang --version` 输出 Apple clang 版本号，说明 CLI 工具可用。

### 2.4 常见许可问题
若构建时报 license 错误，执行：
```bash
sudo xcodebuild -license accept
```

## 3. Homebrew 安装与配置
### 3.1 安装命令
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 3.2 Apple Silicon 与 Intel 前缀差异
- Apple Silicon 默认前缀：`/opt/homebrew`
- Intel 默认前缀：`/usr/local`

### 3.3 初始化 shell 环境
Apple Silicon 常见：
```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```
Intel 常见：
```bash
eval "$(/usr/local/bin/brew shellenv)"
```

### 3.4 验证 Homebrew
```bash
brew --version
brew doctor
```

## 4. Apple Silicon（M1/M2/M3）vs Intel 差异
### 4.1 检查当前终端架构
```bash
uname -m
arch
```
输出含义：
- `arm64`：Apple Silicon 原生终端
- `x86_64`：Intel 终端或 Rosetta 终端

### 4.2 Rosetta 2 的定位
Rosetta 2 用于兼容运行 x86_64 程序，不建议作为默认开发路径。
安装命令（按需）：
```bash
softwareupdate --install-rosetta --agree-to-license
```

### 4.3 架构混用风险
Apple Silicon 常见错误链路：
1. arm64 终端安装一套依赖。
2. Rosetta 终端再安装一套依赖。
3. 链接阶段报 architecture mismatch。
建议统一使用 arm64 终端和 `/opt/homebrew`。

## 5. CMake 安装（brew vs dmg vs 源码）
项目最低要求是 CMake 3.28.0，建议对齐 3.31.x。

### 5.1 方案 A：brew（推荐）
```bash
brew install cmake
cmake --version
```
优点：升级方便、维护成本低。

### 5.2 方案 B：官方 dmg
访问 <https://cmake.org/download/> 下载并安装。
优点：版本精确可控；缺点：升级偏手工。

### 5.3 方案 C：源码安装
仅在需要定制 patch 时考虑，教程阶段不推荐。

### 5.4 验证 cmake 来源
```bash
cmake --version
which cmake
```

## 6. Ninja 安装
```bash
brew install ninja
ninja --version
which ninja
```

## 7. vcpkg 安装与配置
### 7.1 克隆并引导
```bash
git clone https://github.com/microsoft/vcpkg.git "$HOME/vcpkg"
"$HOME/vcpkg"/bootstrap-vcpkg.sh
```

### 7.2 配置环境变量（zsh）
在 `~/.zshrc` 添加：
```bash
export VCPKG_ROOT="$HOME/vcpkg"
export PATH="$VCPKG_ROOT:$PATH"
```
生效：
```bash
source ~/.zshrc
```

### 7.3 验证
```bash
echo "$VCPKG_ROOT"
vcpkg version
```

## 8. bison/flex 安装（macOS 自带版本过旧）
### 8.1 问题背景
macOS 自带 `/usr/bin/bison` 常见为 2.3，版本偏旧。

### 8.2 安装命令
```bash
brew install bison flex
```

### 8.3 PATH 前置（Apple Silicon）
```bash
export PATH="/opt/homebrew/opt/bison/bin:/opt/homebrew/opt/flex/bin:$PATH"
```

### 8.4 PATH 前置（Intel）
```bash
export PATH="/usr/local/opt/bison/bin:/usr/local/opt/flex/bin:$PATH"
```

### 8.5 验证命中版本
```bash
which bison
bison --version
which flex
flex --version
```

## 9. Python3 与 NASM
```bash
brew install python
brew install nasm
python3 --version
nasm --version
```

## 10. Framework 路径与 Code Signing 概念
### 10.1 Framework 路径
常见路径：
- `/System/Library/Frameworks`
- `/Library/Frameworks`

### 10.2 Code Signing 基础
本地调试可先不做完整签名流程，但 `.app` 分发通常需要签名。
```bash
codesign -dv --verbose=4 /path/to/app_or_binary
```

## 11. 环境变量配置（.zshrc）
推荐集中配置（Apple Silicon 示例）：
```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
export VCPKG_ROOT="$HOME/vcpkg"
export PATH="$VCPKG_ROOT:$PATH"
export PATH="/opt/homebrew/opt/bison/bin:/opt/homebrew/opt/flex/bin:$PATH"
```
执行：
```bash
source ~/.zshrc
```

## 12. Universal Binary（arm64 + x86_64）
Universal Binary 指同一二进制同时包含 arm64 与 x86_64 机器码。
开发阶段建议先保持单架构一致，发布阶段再考虑双架构。
检查命令：
```bash
file /path/to/binary
lipo -info /path/to/binary
```

## 13. 使用 preset 构建
Debug：
```bash
cmake --preset="MacOS Debug Config"
cmake --build --preset="MacOS Debug Build"
```
Release：
```bash
cmake --preset="MacOS Release Config"
cmake --build --preset="MacOS Release Build"
```

## 14. 验证命令清单
完成搭建后建议依次执行：
```bash
sw_vers
uname -m
xcode-select -p
clang --version
brew --version
cmake --version
ninja --version
echo "$VCPKG_ROOT"
vcpkg version
bison --version
flex --version
python3 --version
nasm --version
```

## 15. 常见 macOS 环境问题
### 15.1 `bison --version` 仍是 2.3
原因：命中系统 `/usr/bin/bison`。  
修复：将 Homebrew bison 路径前置并 `source ~/.zshrc`。

### 15.2 `cmake` 版本过低
原因：PATH 指向旧版本。  
修复：重装 CMake 并用 `which cmake` 检查来源。

### 15.3 `vcpkg` 找不到
原因：`VCPKG_ROOT` 或 PATH 配置不完整。  
修复：补全 `.zshrc` 并重载 shell。

### 15.4 Apple Silicon 架构冲突
原因：x86_64 与 arm64 依赖混用。  
修复：统一终端架构，清理并重建依赖。

### 15.5 `xcode-select` 指向异常
```bash
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
```

## 16. 动手实践
### 步骤 1：安装核心工具
```bash
brew install cmake ninja bison flex python nasm
```

### 步骤 2：安装并配置 vcpkg
```bash
git clone https://github.com/microsoft/vcpkg.git "$HOME/vcpkg"
"$HOME/vcpkg"/bootstrap-vcpkg.sh
source ~/.zshrc
vcpkg version
```

### 步骤 3：执行 Debug 构建
```bash
cmake --preset="MacOS Debug Config"
cmake --build --preset="MacOS Debug Build"
```

### 步骤 4：记录环境快照
记录以下信息：
1. 系统版本
2. CPU 架构
3. cmake/ninja 版本
4. bison/flex 版本
5. 首次构建结果

## 17. 对照项目源码
相关文件与说明：
- `krkr2/CMakePresets.json` 第 98-127 行：macOS configure preset。
- `krkr2/CMakePresets.json` 第 148-153 行：macOS build preset。
- `krkr2/vcpkg.json` 第 1-94 行：manifest 依赖清单。
- `krkr2/.github/workflows/build-linux.yml` 第 54-58 行：CMake 版本参考基线。

## 18. 本节小结
- 核心工具链是 Xcode/CLT + Homebrew + CMake + Ninja + vcpkg。
- Apple Silicon 与 Intel 差异主要在路径与架构一致性。
- bison/flex 是高频坑点，PATH 顺序决定能否命中新版本。
- 先稳定单架构，再考虑 Universal Binary 发布。

## 19. 练习题与答案
### 题目 1：Apple Silicon 上 Homebrew 默认路径是什么？
<details>
<summary>查看答案</summary>

默认路径是 `/opt/homebrew`。

</details>

### 题目 2：为什么安装了 bison 后仍可能显示 2.3？
<details>
<summary>查看答案</summary>

因为 shell 命中了系统 `/usr/bin/bison`。需要把 Homebrew bison 目录前置到 PATH，并重新加载 shell。

</details>

### 题目 3：KR2 在 macOS 上的 Debug 构建命令是什么？
<details>
<summary>查看答案</summary>

```bash
cmake --preset="MacOS Debug Config"
cmake --build --preset="MacOS Debug Build"
```

</details>

### 题目 4：什么是 Universal Binary？
<details>
<summary>查看答案</summary>

Universal Binary 是同一个可执行文件同时包含 arm64 与 x86_64 两种机器码，可在两类 Mac 上运行。

</details>

## 20. 下一步
继续阅读 [04-Android环境搭建](./04-Android环境搭建.md)。
下一节会把本节的工具链思路迁移到 Android NDK/SDK、Gradle、ABI 与签名流程。
