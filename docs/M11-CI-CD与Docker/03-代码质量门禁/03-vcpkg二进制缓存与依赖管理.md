# vcpkg 二进制缓存与依赖管理

> **所属模块：** M11-CI/CD 与 Docker
> **前置知识：** [P02-vcpkg 包管理](../../P02-vcpkg包管理/README.md)、[02-clang-tidy 静态分析](./02-clang-tidy静态分析.md)
> **预计阅读时间：** 40 分钟（约 8000 字）

## 本节目标

读完本节后，你将能够：
1. 理解 KrKr2 的 `vcpkg.json` 清单文件中每个依赖的作用和平台条件
2. 解释 vcpkg 二进制缓存的原理以及 NuGet 缓存在 CI 中如何运作
3. 阅读并理解 KrKr2 CI 工作流中 `VCPKG_BINARY_SOURCES` 的配置
4. 掌握 overlay ports（覆盖端口）的概念，理解 KrKr2 为何自定义了 6 个端口
5. 编写自定义 vcpkg triplet 文件并理解 Android 交叉编译 triplet 的配置

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 清单模式 | Manifest Mode | vcpkg 的一种使用模式，通过项目根目录的 `vcpkg.json` 文件声明依赖，类似 npm 的 `package.json` |
| 二进制缓存 | Binary Caching | 将编译好的库文件缓存起来，下次构建时直接复用，避免重复编译 |
| NuGet | NuGet | 微软的包管理系统，vcpkg 借用它作为二进制缓存的存储后端 |
| 覆盖端口 | Overlay Ports | 项目本地的自定义 vcpkg 端口定义，优先级高于 vcpkg 官方注册表中的版本 |
| Triplet | Triplet | vcpkg 中描述目标平台配置的文件，包含架构、操作系统、链接方式等信息 |
| 端口 | Port | vcpkg 中一个库的构建配方，包含 `portfile.cmake`（构建脚本）和 `vcpkg.json`（元数据） |

---

## KrKr2 的 `vcpkg.json` 清单文件

KrKr2 使用 vcpkg 的**清单模式**（Manifest Mode）管理 C++ 依赖。项目根目录的 `vcpkg.json` 文件（94 行）声明了所有需要的第三方库：

```json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/vcpkg-tool/main/docs/vcpkg.schema.json",
  "name": "krkr2",
  "version-string": "1.5.0",
  "dependencies": [
    "7zip",
    {
      "name": "argparse",
      "platform": "linux | windows | osx"
    },
    "boost-iostreams",
    ...
  ],
  "overrides": [
    {
      "name": "opencv4",
      "version": "4.7.0#6"
    }
  ]
}
```

### 依赖列表完整解读

KrKr2 共声明了 **28 个**直接依赖。下面按功能分组逐一说明：

#### 核心引擎依赖

| 依赖名 | 功能 | 平台限制 | 说明 |
|--------|------|----------|------|
| `cocos2dx` | 游戏引擎框架 | 全平台 | KrKr2 的渲染和 UI 基础，使用 **overlay port** 自定义构建 |
| `ffmpeg` | 音视频编解码 | 全平台 | 视频播放器核心，使用 overlay port 自定义编译选项 |
| `sdl2` | 跨平台多媒体层 | 全平台 | 窗口管理、输入处理、音频输出；`default-features: false` 禁用不需要的组件 |
| `openal-soft` | 3D 音频引擎 | 全平台 | 空间音效支持 |

#### 图像处理依赖

| 依赖名 | 功能 | 平台限制 | 说明 |
|--------|------|----------|------|
| `libjpeg-turbo` | JPEG 解码 | 全平台 | 使用 SIMD 加速的 JPEG 库 |
| `libpng` | PNG 解码 | `!linux` | Linux 上使用系统自带的 libpng |
| `libwebp` | WebP 解码 | 全平台 | Google 的 WebP 图像格式 |
| `jxrlib` | JPEG XR 解码 | 全平台 | 微软的 JPEG XR 格式（用于某些游戏资源） |
| `opencv4` | 计算机视觉 | 全平台 | 仅启用 `openmp` 特性；版本锁定为 `4.7.0#6` |

#### 压缩与归档依赖

| 依赖名 | 功能 | 平台限制 | 说明 |
|--------|------|----------|------|
| `7zip` | 7z 压缩 | 全平台 | 支持 XP3 归档中的 7z 压缩 |
| `libarchive` | 通用归档库 | 全平台 | 支持多种归档格式；使用 overlay port |
| `lz4` | 快速压缩 | 全平台 | 用于高速数据压缩 |
| `zstd` | Zstandard 压缩 | 全平台 | Facebook 的现代压缩算法 |
| `unrar` | RAR 解压 | 全平台 | 使用 overlay port 自定义构建 |

#### 文本与编码依赖

| 依赖名 | 功能 | 平台限制 | 说明 |
|--------|------|----------|------|
| `oniguruma` | 正则表达式 | 全平台 | 支持日文等多字节字符的正则引擎 |
| `uchardet` | 编码检测 | 全平台 | 自动检测文本文件的字符编码 |
| `fmt` | 格式化输出 | 全平台 | 现代 C++ 格式化库（`spdlog` 的依赖） |
| `spdlog` | 日志库 | 全平台 | KrKr2 全局使用的结构化日志库 |

#### Boost 与工具依赖

| 依赖名 | 功能 | 平台限制 | 说明 |
|--------|------|----------|------|
| `boost-iostreams` | IO 流扩展 | 全平台 | 支持压缩流、内存映射文件等 |
| `boost-locale` | 国际化 | 全平台 | 字符编码转换；`default-features: false` 禁用 ICU 后端 |
| `boost-spirit` | 解析器框架 | 全平台 | 用于 KAG 脚本解析等文本处理 |
| `boost-phoenix` | 函数对象库 | 全平台 | Boost.Spirit 的依赖 |
| `bullet3` | 物理引擎 | 全平台 | Cocos2d-x 的物理模拟后端 |

#### 平台特定依赖

| 依赖名 | 功能 | 平台限制 | 说明 |
|--------|------|----------|------|
| `argparse` | 命令行解析 | `linux | windows | osx` | 解析启动参数；Android 不需要 |
| `glfw3` | 窗口/OpenGL 上下文 | `linux | windows | osx` | 桌面平台的窗口创建；Android 用原生窗口 |
| `catch2` | 单元测试框架 | `!android & !ios` | 仅桌面平台构建测试 |
| `dirent` | 目录遍历 | `windows` | Windows 缺少 POSIX dirent.h，需要这个兼容层 |
| `breakpad` | 崩溃报告 | `android` | Google 的崩溃收集工具，仅 Android 启用 |
| `libgdiplus` | GDI+ 兼容层 | `!windows` | 非 Windows 平台上提供 GDI+ API |
| `libogg` | Ogg 容器 | `android` | Android 上独立提供 Ogg 支持 |
| `opus` / `opusfile` | Opus 音频 | `android` / 全平台 | 高质量语音和音乐编解码器 |
| `vcpkg-cmake` | vcpkg CMake 辅助 | 宿主工具 | `"host": true` 表示在构建主机上运行 |

### 平台条件语法

vcpkg.json 中的 `"platform"` 字段使用布尔表达式语法：

```
"linux | windows | osx"     → Linux 或 Windows 或 macOS（排除 Android/iOS）
"!android & !ios"           → 除了 Android 和 iOS 之外的所有平台
"!windows"                  → 除了 Windows 的所有平台
"!linux"                    → 除了 Linux 的所有平台
"android"                   → 仅 Android
"windows"                   → 仅 Windows
```

这些条件在 vcpkg 安装依赖时自动评估——如果当前目标平台不满足条件，该依赖会被**完全跳过**（不下载、不编译、不安装）。

### 版本锁定（overrides）

```json
"overrides": [
    {
      "name": "opencv4",
      "version": "4.7.0#6"
    }
]
```

`overrides`（覆盖，vcpkg 中用于强制指定某个依赖的版本号，忽略版本解析器的自动选择）将 `opencv4` 锁定在 `4.7.0#6`。`#6` 是 vcpkg 的**端口修订号**（Port Revision，表示同一个库版本的第 N 次构建配方更新）。

**为什么要锁定 opencv4？** OpenCV 是一个巨大的库，新版本可能引入 API 变更或新的依赖要求。KrKr2 只使用 OpenCV 的少量功能（图像处理和 OpenMP 加速），锁定版本可以避免意外的编译失败或行为变更。

---

## vcpkg 二进制缓存原理

### 为什么需要二进制缓存？

vcpkg 从源码编译所有依赖库。KrKr2 有 28 个直接依赖，加上传递依赖（Transitive Dependencies，被直接依赖的库所依赖的其他库，如 `ffmpeg` 依赖 `zlib`）可能超过 100 个库。从源码编译全部依赖可能需要 **30-60 分钟**——这对于 CI 来说太慢了。

**二进制缓存**（Binary Caching）的核心思想：如果某个库的"构建指纹"（版本号 + 编译选项 + 平台 + 依赖版本的哈希值）没有变化，就直接下载上次编译好的二进制文件，跳过编译。

### 缓存工作流程

```
首次构建：
  源码 → vcpkg 编译 → 二进制产物 → 上传到缓存存储
                                        ↓
后续构建：                             缓存存储
  计算构建指纹 → 查询缓存 → 命中 → 下载二进制 → 完成（秒级）
                           ↓
                         未命中 → 源码编译 → 上传缓存
```

### 缓存存储后端

vcpkg 支持多种缓存后端：

| 后端 | 说明 | 适用场景 |
|------|------|----------|
| 本地文件系统 | 缓存在本地磁盘（`~/.cache/vcpkg/archives/`） | 开发者本机 |
| NuGet | 使用 NuGet 包管理协议存储 | CI（GitHub Packages、Azure Artifacts） |
| GitHub Actions Cache | 使用 GitHub Actions 的内置缓存 | GitHub Actions CI |
| AWS S3 / Azure Blob | 云对象存储 | 大型团队、自建 CI |
| GCS (Google Cloud) | Google 云存储 | GCP 环境 |

KrKr2 使用 **NuGet + GitHub Packages** 作为缓存后端——这是 GitHub 项目最常见的选择。

---

## KrKr2 CI 中的二进制缓存配置

以 `build-linux.yml` 为例，逐行解读二进制缓存相关配置：

### 环境变量声明

```yaml
env:
  # VCPKG_BINARY_SOURCES: 控制二进制缓存的来源和写入方式
  VCPKG_BINARY_SOURCES: "clear;nuget,https://nuget.pkg.github.com/${{ github.repository_owner }}/index.json,readwrite"
  VCPKG_INSTALL_OPTIONS: "--debug"
```

**`VCPKG_BINARY_SOURCES`** 是一个用分号 `;` 分隔的指令列表：

| 部分 | 含义 |
|------|------|
| `clear` | 先清除所有默认缓存源（防止意外使用本地缓存） |
| `nuget,<URL>,readwrite` | 添加 NuGet 缓存源，URL 指向 GitHub Packages 的 NuGet 仓库 |
| `readwrite` | 既可以读取（下载）缓存，也可以写入（上传）新编译的二进制 |

`${{ github.repository_owner }}` 会被替换为仓库所有者（如 `jeffcwj`），最终 URL 形如：
```
https://nuget.pkg.github.com/jeffcwj/index.json
```

### NuGet 源注册

```yaml
- name: Add NuGet sources
  shell: bash
  run: |
    mono `${{ env.VCPKG_ROOT }}/vcpkg fetch nuget | tail -n 1` \
      sources add \
      -Source "https://nuget.pkg.github.com/${{ github.repository_owner }}/index.json" \
      -StorePasswordInClearText \
      -Name GitHubPackages \
      -UserName "${{ github.actor }}" \
      -Password "${{ secrets.NUGET_API_KEY }}"
    mono `${{ env.VCPKG_ROOT }}/vcpkg fetch nuget | tail -n 1` \
      setapikey "${{ secrets.NUGET_API_KEY }}" \
      -Source "https://nuget.pkg.github.com/${{ github.repository_owner }}/index.json"
```

这段脚本做了两件事：

1. **注册 NuGet 源**：通过 `sources add` 命令将 GitHub Packages 添加为 NuGet 包源，使用 `NUGET_API_KEY` 密钥认证
2. **设置 API Key**：通过 `setapikey` 命令配置上传权限

**关键细节：**
- `mono`：NuGet CLI 是 .NET 应用，Linux 上需要通过 Mono（.NET 的开源跨平台实现）运行
- `vcpkg fetch nuget`：vcpkg 内置的 NuGet 客户端下载器，输出 NuGet 可执行文件的路径
- `${{ secrets.NUGET_API_KEY }}`：存储在 GitHub 仓库 Settings → Secrets 中的密钥，具有 `write:packages` 权限的 Personal Access Token（个人访问令牌）

### 缓存命中时的效果

当缓存完全命中时，vcpkg 安装 28 个依赖的时间从 **30-60 分钟**缩短到 **2-5 分钟**——几乎只是下载和解压的时间。

```
构建日志对比：
# 无缓存：
Installing 1/28 7zip:x64-linux...
Building 7zip:x64-linux...        ← 从源码编译，耗时 2 分钟
...
Total install time: 45 minutes

# 有缓存：
Installing 1/28 7zip:x64-linux...
Restored from nuget source         ← 从缓存下载，耗时 3 秒
...
Total install time: 3 minutes
```

---

## Overlay Ports（覆盖端口）

### 什么是 Overlay Ports？

vcpkg 的官方注册表（Registry，vcpkg 维护的数千个库的构建配方集合）提供了大多数常用 C++ 库的端口。但有时官方端口不满足需求：

- 需要特定的编译选项
- 官方端口的版本太新或太旧
- 需要额外的补丁修复兼容性问题
- 库完全不在官方注册表中（如 Cocos2d-x）

**Overlay Ports** 允许你在项目内维护自定义的端口定义。当 vcpkg 安装依赖时，会**优先使用** overlay 目录中的端口，而非官方注册表中的版本。

### KrKr2 的 Overlay Ports

KrKr2 在 `vcpkg/ports/` 目录下维护了 **6 个**自定义端口：

```
vcpkg/ports/
├── 7zip/          # 7-Zip 压缩库
├── cocos2dx/      # Cocos2d-x 游戏引擎（官方注册表中没有）
├── ffmpeg/        # FFmpeg 音视频库（需要自定义编译选项）
├── libarchive/    # 归档库（需要特定配置）
├── libgdiplus/    # GDI+ 兼容层（需要补丁）
└── unrar/         # RAR 解压库（可能未在官方注册表）
```

### 以 cocos2dx 端口为例

Cocos2d-x 不在 vcpkg 官方注册表中，KrKr2 完全自建了这个端口。端口结构：

```
vcpkg/ports/cocos2dx/
├── vcpkg.json              # 端口元数据（名称、版本、依赖）
├── portfile.cmake          # 构建脚本（67 行）
├── DownloadDeps.cmake      # 额外依赖下载脚本
├── cocos2dx-config.cmake.in # CMake 配置文件模板
└── patch/                  # 兼容性补丁目录
    ├── 0001-add-cstdint-header.patch
    ├── fix-iconv-cast.patch
    ├── fix-mac-audio-build.patch
    └── ... (10 个补丁)
```

`portfile.cmake` 的核心逻辑：

```cmake
# 1. 下载 Cocos2d-x 源码
vcpkg_download_distfile(
    ARCHIVE
    URLS https://github.com/cocos2d/cocos2d-x/archive/refs/tags/${COCOS2D_VERSION}.tar.gz
    FILENAME ${COCOS2D_VERSION}.tar.gz
    SHA512 b2d5ac968...  # 校验和确保下载完整性
)

# 2. 解压并应用补丁
vcpkg_extract_source_archive(
    SOURCE_PATH
    ARCHIVE "${ARCHIVE}"
    PATCHES
        patch/0001-add-cstdint-header.patch   # 修复 C++17 编译
        patch/fix-iconv-cast.patch            # 修复 iconv 类型转换
        patch/fix-mac-audio-build.patch       # 修复 macOS 音频
        patch/fix-win64.patch                 # 修复 Windows 64 位
        # ... 共 10 个补丁
)

# 3. 使用 CMake 构建
vcpkg_cmake_configure(
    SOURCE_PATH "${SOURCE_PATH}"
    OPTIONS
        -DBUILD_TESTS=OFF    # 不构建测试
        -DBUILD_JS_LIBS=OFF  # 不构建 JavaScript 绑定
        -DBUILD_LUA_LIBS=OFF # 不构建 Lua 绑定
)

# 4. 安装
vcpkg_cmake_install()
vcpkg_copy_pdbs()           # 复制调试符号（Windows PDB 文件）
vcpkg_cmake_config_fixup()  # 修复 CMake 配置文件的路径
```

**10 个补丁**说明 Cocos2d-x 的原始代码需要大量修改才能在现代编译器和 KrKr2 的环境下正确编译。如果使用官方发布的 Cocos2d-x 而不应用这些补丁，编译会失败。

---

## 自定义 Triplet

### Triplet 是什么？

Triplet（三元组，vcpkg 中描述目标平台的配置文件）定义了目标平台的三个基本属性：
1. **架构**（Architecture）：x64、arm64、x86 等
2. **操作系统**（OS）：Windows、Linux、Android 等
3. **链接方式**（Linkage）：静态库（static）或动态库（dynamic）

vcpkg 内置了常见 triplet（如 `x64-windows`、`x64-linux`、`arm64-android`），但 KrKr2 需要为 Android 自定义 triplet——因为需要**条件性的动态链接**。

### KrKr2 的自定义 Android Triplet

KrKr2 在 `vcpkg/triplets/` 目录下维护了 5 个 Android triplet 文件：

```
vcpkg/triplets/
├── android-dynamic-libs.cmake    # 共享的动态链接规则
├── arm-android.cmake             # 32 位 ARM (armeabi-v7a)
├── arm64-android.cmake           # 64 位 ARM (arm64-v8a)
├── x64-android.cmake             # 64 位 x86 (x86_64)
└── x86-android.cmake             # 32 位 x86
```

以 `arm64-android.cmake` 为例（9 行）：

```cmake
set(VCPKG_TARGET_ARCHITECTURE arm64)        # 目标架构
set(VCPKG_CRT_LINKAGE static)              # C 运行时静态链接
set(VCPKG_LIBRARY_LINKAGE static)          # 所有库默认静态链接
set(VCPKG_CMAKE_SYSTEM_NAME Android)       # 目标操作系统
set(VCPKG_MAKE_BUILD_TRIPLET "--host=aarch64-linux-android")  # autotools 交叉编译参数
set(VCPKG_CMAKE_CONFIGURE_OPTIONS -DANDROID_ABI=arm64-v8a)    # CMake Android ABI
set(VCPKG_CMAKE_SYSTEM_VERSION 28)         # Android API Level 28 (Android 9.0)

include(${CMAKE_CURRENT_LIST_DIR}/android-dynamic-libs.cmake) # 引入动态链接例外
```

### 条件性动态链接

`android-dynamic-libs.cmake`（5 行）定义了哪些库必须使用动态链接：

```cmake
set(DYNAMIC_LIBRARIES sdl2)   # SDL2 必须是动态库

if(PORT IN_LIST DYNAMIC_LIBRARIES)
    set(VCPKG_LIBRARY_LINKAGE dynamic)  # 覆盖默认的 static
endif()
```

**为什么 SDL2 必须动态链接？** Android 的 Java 层通过 JNI（Java Native Interface，Java 和 C/C++ 代码之间的桥接接口）加载原生库。SDL2 在 Android 上有自己的 Java Activity（`SDLActivity`），需要作为独立的 `.so` 文件被 Java 类加载器发现。如果 SDL2 被静态链接到主程序中，Java 层无法找到它。

---

## 动手实践

### 实践 1：查看本地二进制缓存

vcpkg 默认将编译好的二进制包缓存到本地目录。我们来找到并查看它：

**Windows：**

```powershell
# 查看默认缓存目录
dir "$env:LOCALAPPDATA\vcpkg\archives"

# 列出所有 .zip 缓存文件（每个文件对应一个已编译的库）
Get-ChildItem "$env:LOCALAPPDATA\vcpkg\archives" -Recurse -Filter "*.zip" |
    Select-Object Name, Length, LastWriteTime |
    Sort-Object LastWriteTime -Descending |
    Format-Table -AutoSize

# 查看缓存总大小
$size = (Get-ChildItem "$env:LOCALAPPDATA\vcpkg\archives" -Recurse |
    Measure-Object -Property Length -Sum).Sum / 1MB
Write-Host "缓存总大小: $([math]::Round($size, 2)) MB"
```

**Linux / macOS：**

```bash
# 查看默认缓存目录
ls -la ~/.cache/vcpkg/archives/

# 列出所有缓存文件并按时间排序
find ~/.cache/vcpkg/archives/ -name "*.zip" -exec ls -lh {} \; | sort -k6,7

# 查看缓存总大小
du -sh ~/.cache/vcpkg/archives/
```

**预期输出示例：**

```
缓存总大小: 847.32 MB
```

> **观察要点：** 缓存文件名是一串哈希值（如 `a3b7c9d2e1f4...zip`），这个哈希由库名、版本、编译选项、triplet 等因素计算而来。相同的输入一定产生相同的哈希，这就是缓存命中的原理。

### 实践 2：创建一个最小的 Overlay Port

我们来创建一个自定义的 overlay port，体验端口的基本结构：

**步骤 1 — 创建端口目录：**

```bash
# 在项目根目录下
mkdir -p vcpkg/ports/my-hello-lib
```

**步骤 2 — 编写 vcpkg.json（端口清单文件）：**

```json
{
    "name": "my-hello-lib",
    "version": "1.0.0",
    "description": "一个用于学习 overlay port 的示例库",
    "homepage": "https://example.com",
    "license": "MIT",
    "dependencies": []
}
```

将上面的内容保存为 `vcpkg/ports/my-hello-lib/vcpkg.json`。

**步骤 3 — 编写 portfile.cmake（构建脚本）：**

```cmake
# portfile.cmake — vcpkg 的构建入口文件
# 每个端口必须有这个文件，告诉 vcpkg 如何获取和编译源码

# 从 GitHub 下载源码（这里用一个假的 URL 演示格式）
vcpkg_from_github(
    OUT_SOURCE_PATH SOURCE_PATH    # 下载后源码的存放路径
    REPO example/hello-lib         # GitHub 仓库地址（用户名/仓库名）
    REF v1.0.0                     # Git 标签或提交哈希
    SHA512 0                       # 源码压缩包的 SHA512 校验值（0 表示跳过校验，仅供测试）
    HEAD_REF main                  # 开发分支名
)

# 使用 CMake 构建
vcpkg_cmake_configure(
    SOURCE_PATH "${SOURCE_PATH}"   # 源码路径
)

vcpkg_cmake_install()             # 执行 cmake --install
vcpkg_cmake_config_fixup()        # 修正 CMake 配置文件路径

# 清理多余文件
file(REMOVE_RECURSE
    "${CURRENT_PACKAGES_DIR}/debug/include"    # debug 版不需要头文件
    "${CURRENT_PACKAGES_DIR}/debug/share"      # debug 版不需要 share
)

# 安装版权文件（vcpkg 要求每个端口必须提供）
vcpkg_install_copyright(FILE_LIST "${SOURCE_PATH}/LICENSE")
```

将上面的内容保存为 `vcpkg/ports/my-hello-lib/portfile.cmake`。

> **关键理解：** portfile.cmake 就是一个 CMake 脚本，vcpkg 在安装端口时会执行它。它定义了"从哪里下载源码"、"如何编译"、"如何安装"三个步骤。KrKr2 的 cocos2dx 端口也是这个结构，只不过更复杂（有 10 个补丁文件）。

### 实践 3：查看 vcpkg 安装某个库的完整日志

当 vcpkg 编译一个库时，它会在构建目录下生成详细日志。学会看日志能帮你排查安装失败的问题：

```bash
# 安装一个小型库并查看详细输出
vcpkg install fmt --triplet x64-windows 2>&1

# 如果安装失败，查看构建日志
# 日志路径格式：<vcpkg-root>/buildtrees/<port>/build-<triplet>-out.log
cat $VCPKG_ROOT/buildtrees/fmt/build-x64-windows-rel-out.log
```

**日志中的关键信息：**

```
-- Downloading https://github.com/fmtlib/fmt/archive/refs/tags/10.2.1.tar.gz
-- Extracting source /path/to/vcpkg/downloads/fmtlib-fmt-...tar.gz
-- Using cached binary package: /path/to/archives/ab/abcdef123456.zip
-- Configuring x64-windows-rel
-- Building x64-windows-rel
-- Installing x64-windows-rel
-- Fixing pkgconfig file: /path/to/packages/fmt_x64-windows/lib/pkgconfig/fmt.pc
```

> **要点：** 如果看到 `Using cached binary package`，说明命中了二进制缓存，跳过了编译步骤。如果没看到这一行，说明是从源码重新编译的。

### 实践 4：对比不同 Triplet 的编译差异

同一个库在不同 triplet 下编译会产生完全不同的结果。我们来观察差异：

```bash
# 安装 zlib 的两个不同 triplet 版本
vcpkg install zlib:x64-windows        # 默认动态链接
vcpkg install zlib:x64-windows-static  # 静态链接

# 对比安装结果
echo "=== 动态链接版本 ==="
ls -la $VCPKG_ROOT/installed/x64-windows/lib/zlib*
ls -la $VCPKG_ROOT/installed/x64-windows/bin/zlib*

echo "=== 静态链接版本 ==="
ls -la $VCPKG_ROOT/installed/x64-windows-static/lib/zlib*
```

**预期观察：**

| 文件 | x64-windows（动态） | x64-windows-static（静态） |
|------|---------------------|---------------------------|
| `lib/zlib.lib` | 导入库（很小，几 KB） | 完整静态库（几百 KB） |
| `bin/zlib1.dll` | ✅ 存在（运行时需要） | ❌ 不存在 |
| 缓存哈希 | 哈希 A | 哈希 B（不同！） |

> **核心理解：** triplet 决定了编译选项，不同 triplet 产生不同的二进制文件和不同的缓存哈希。这就是为什么 KrKr2 为 Android 的四种 CPU 架构各创建了一个 triplet——每种架构需要独立编译、独立缓存。

**跨平台验证命令对照：**

```bash
# Windows (PowerShell)
Get-ChildItem "$env:VCPKG_ROOT\installed\x64-windows\lib\zlib*"
Get-ChildItem "$env:VCPKG_ROOT\installed\x64-windows\bin\zlib*"

# Linux
ls -la $VCPKG_ROOT/installed/x64-linux/lib/libz*

# macOS
ls -la $VCPKG_ROOT/installed/x64-osx/lib/libz*

# Android (通过项目的自定义 triplet)
ls -la $VCPKG_ROOT/installed/arm64-android/lib/libz*
```

---

## 对照项目源码

KrKr2 项目中与 vcpkg 依赖管理相关的文件一览：

| 文件路径 | 行数 | 功能说明 |
|----------|------|----------|
| `vcpkg.json` | 94 行 | 项目依赖清单，声明 28 个直接依赖和平台条件 |
| `.github/workflows/build-linux.yml` | 100 行 | Linux CI 流水线，第 35-52 行配置 NuGet 二进制缓存 |
| `.github/workflows/build-windows.yml` | 84 行 | Windows CI 流水线，第 28-42 行配置二进制缓存 |
| `.github/workflows/build-android.yml` | 169 行 | Android CI 流水线，第 45-70 行配置缓存 + 多架构构建 |
| `vcpkg/ports/cocos2dx/portfile.cmake` | 67 行 | Cocos2d-x 自定义端口，完整的 CMake 构建脚本 |
| `vcpkg/ports/cocos2dx/vcpkg.json` | — | Cocos2d-x 端口的依赖声明 |
| `vcpkg/ports/7zip/portfile.cmake` | — | 7-Zip 自定义端口 |
| `vcpkg/ports/ffmpeg/portfile.cmake` | — | FFmpeg 自定义端口（添加额外编解码器选项） |
| `vcpkg/ports/libarchive/portfile.cmake` | — | libarchive 自定义端口（启用 RAR 支持） |
| `vcpkg/ports/libgdiplus/portfile.cmake` | — | libgdiplus 自定义端口（Linux/macOS GDI+ 替代） |
| `vcpkg/ports/unrar/portfile.cmake` | — | UnRAR 自定义端口（官方仓库不含此库） |
| `vcpkg/triplets/arm64-android.cmake` | 9 行 | ARM64 Android triplet 定义 |
| `vcpkg/triplets/arm-android.cmake` | — | ARM32 Android triplet 定义 |
| `vcpkg/triplets/x64-android.cmake` | — | x86_64 Android triplet 定义 |
| `vcpkg/triplets/x86-android.cmake` | — | x86 Android triplet 定义 |
| `vcpkg/triplets/android-dynamic-libs.cmake` | 5 行 | SDL2 动态链接例外规则 |
| `CMakePresets.json` | 156 行 | 第 8-12 行配置 `VCPKG_ROOT` 和 toolchain 路径 |
| `cmake/vcpkg_android.cmake` | — | Android 构建时的 vcpkg 集成辅助脚本 |

**核心联动关系：**

```
vcpkg.json（声明依赖）
    │
    ├── vcpkg/ports/（overlay ports 覆盖官方端口）
    │   └── portfile.cmake → 定义如何下载、编译、安装
    │
    ├── vcpkg/triplets/（自定义 triplet）
    │   └── arm64-android.cmake → 定义目标架构和编译选项
    │       └── include android-dynamic-libs.cmake → SDL2 特殊处理
    │
    ├── CMakePresets.json（指定 VCPKG_ROOT 和 toolchain）
    │   └── CMAKE_TOOLCHAIN_FILE = vcpkg/scripts/buildsystems/vcpkg.cmake
    │
    └── .github/workflows/（CI 中配置二进制缓存）
        └── VCPKG_BINARY_SOURCES = "clear;nuget,<URL>,readwrite"
```

---

## 常见错误与排查

### 错误 1：`error: in triplet arm64-android: Unable to find a valid toolchain`

**完整错误信息：**

```
CMake Error at scripts/cmake/vcpkg_configure_cmake.cmake:245 (message):
  error: in triplet arm64-android:
  Unable to find a valid Android NDK toolchain.
  Expected ANDROID_NDK_HOME or ANDROID_NDK environment variable to be set.
```

**原因：** vcpkg 在使用 Android triplet 编译库时，需要 Android NDK（Native Development Kit，Android 原生开发工具包）提供交叉编译工具链。如果环境变量没有正确设置，vcpkg 找不到编译器。

**解决方案：**

```bash
# 方案 1：设置环境变量（推荐）
# Windows (PowerShell)
$env:ANDROID_NDK_HOME = "C:\Users\<你的用户名>\AppData\Local\Android\Sdk\ndk\26.1.10909125"

# Linux / macOS
export ANDROID_NDK_HOME="$HOME/Android/Sdk/ndk/26.1.10909125"

# 方案 2：在 CMakePresets.json 中指定（项目已使用此方式）
# 见 CMakePresets.json 第 45-50 行的 Android 预设配置

# 验证：确认 NDK 路径下有 toolchains/ 目录
ls $ANDROID_NDK_HOME/toolchains/llvm/prebuilt/
```

### 错误 2：`error: failed to push to NuGet source`

**完整错误信息：**

```
error: failed to push to NuGet source https://nuget.pkg.github.com/jeffcwj/index.json:
Response status code does not indicate success: 403 (Forbidden).
```

**原因：** CI 在推送二进制缓存到 GitHub Packages NuGet 源时，API Key（API 密钥，用于身份验证的令牌）无效或过期。GitHub Actions 的 `GITHUB_TOKEN` 需要有 `packages: write` 权限。

**解决方案：**

```yaml
# 在 workflow 文件中确保权限声明正确
permissions:
  contents: read
  packages: write    # 必须有这一行！

# 如果使用自定义 PAT（Personal Access Token，个人访问令牌）
# 确保令牌有 write:packages 权限范围
# 在 GitHub → Settings → Developer settings → Personal access tokens 中检查
```

**验证步骤：**

```bash
# 手动测试 NuGet 源的可达性
# Windows
nuget sources list
nuget push test.nupkg -Source "https://nuget.pkg.github.com/jeffcwj/index.json" -ApiKey $GITHUB_TOKEN

# Linux（需要 Mono）
mono $(which nuget) sources list
```

### 错误 3：`error: overlay port 'cocos2dx' has a conflicting version`

**完整错误信息：**

```
error: while resolving cocos2dx:x64-windows:
  the overlay port in vcpkg/ports/cocos2dx declares version 0.0.1,
  but the baseline expects version 2024-03-15.
  Use "overrides" in vcpkg.json to force a specific version.
```

**原因：** 当 overlay port（本地覆盖端口）的版本号与 vcpkg.json 中的 baseline（基线，vcpkg 的全局版本快照）不一致时，vcpkg 的版本解析器（resolver）会拒绝安装。这通常在升级 vcpkg 注册表后出现——新的 baseline 期望更高的版本号。

**解决方案：**

```json
// 方案 1：在 vcpkg.json 中添加 overrides 强制指定版本
{
    "overrides": [
        {
            "name": "cocos2dx",
            "version": "0.0.1"    // 与 overlay port 中声明的版本一致
        }
    ]
}

// 方案 2：更新 overlay port 的 vcpkg.json 中的版本号
// 修改 vcpkg/ports/cocos2dx/vcpkg.json 中的 "version" 字段
// 使其与 baseline 期望的版本匹配
```

> **预防措施：** KrKr2 项目中的 cocos2dx 是完全自定义的 overlay port（官方 vcpkg 仓库中不存在此包），所以版本号完全由项目自己控制。只要不随意修改 overlay port 的版本号，就不会遇到此错误。

---

## 本节小结

- **vcpkg.json 清单模式**：KrKr2 通过 `vcpkg.json` 声明式地管理 28 个直接依赖，按功能分为核心引擎库（SDL2、Cocos2d-x）、图像处理库（opencv4、libpng 等）、压缩归档库（zlib、libarchive 等）、文本编码库（ICU）、Boost 工具库和平台特定库（libgdiplus）
- **平台条件语法**：通过 `"platform"` 字段的布尔表达式（如 `"!windows & !android"`）控制哪些依赖在哪些平台上安装，避免不必要的编译
- **版本锁定**：`overrides` 字段强制指定特定依赖的版本（如 opencv4 锁定 4.7.0#6），防止上游更新导致兼容性问题
- **二进制缓存原理**：vcpkg 通过哈希（库名 + 版本 + triplet + 编译选项）唯一标识编译结果，缓存命中时直接下载预编译包，跳过耗时的源码编译
- **CI 二进制缓存配置**：通过 `VCPKG_BINARY_SOURCES` 环境变量配置 NuGet 后端，实现跨 CI 运行的缓存共享。Linux CI 需要 Mono 来运行 NuGet CLI
- **Overlay Ports**：当官方 vcpkg 仓库不包含某个库（如 cocos2dx、unrar）或需要自定义编译选项时，通过 overlay port 提供本地端口定义。portfile.cmake 定义下载、编译、安装三个步骤
- **自定义 Triplet**：KrKr2 为 Android 的四种 CPU 架构（arm、arm64、x86、x64）各创建了一个 triplet，通过 `include` 共享动态链接例外规则（SDL2 必须动态链接）
- **缓存后端选择**：本地文件系统缓存适合个人开发，NuGet/GitHub Packages 适合团队 CI，AWS S3/GCS/Azure Blob 适合大规模企业环境

---

## 练习题与答案

### 题目 1：分析 vcpkg.json 中的平台条件

阅读以下 vcpkg.json 片段，回答问题：

```json
{
    "dependencies": [
        {
            "name": "libgdiplus",
            "platform": "!windows & !android"
        },
        {
            "name": "sdl2",
            "platform": "android"
        },
        {
            "name": "openal-soft"
        }
    ]
}
```

**问题：**
1. `libgdiplus` 会在哪些平台上安装？为什么 Windows 不需要？
2. `sdl2` 为什么只在 Android 上安装？
3. `openal-soft` 没有 `platform` 字段意味着什么？

<details>
<summary>查看答案</summary>

1. **`libgdiplus` 会在 Linux 和 macOS 上安装。** 条件 `!windows & !android` 排除了 Windows 和 Android 两个平台。Windows 不需要 libgdiplus 是因为 Windows 自带原生的 GDI+（Graphics Device Interface Plus，图形设备接口增强版）库，它是 Windows 操作系统的内置组件。而 Linux 和 macOS 没有 GDI+，所以需要安装开源的 libgdiplus 作为替代实现。

2. **`sdl2` 只在 Android 上安装，因为 Android 平台使用 SDL2 作为窗口管理和输入处理的基础层。** 在其他平台上，KrKr2 使用平台原生的窗口系统（Windows 用 Win32 API，Linux 用 X11/Wayland，macOS 用 Cocoa）。Android 上没有直接可用的原生窗口 API 给 C++ 使用，所以通过 SDL2 提供统一的窗口和输入抽象。

3. **没有 `platform` 字段意味着该依赖在所有平台上都安装。** `openal-soft`（OpenAL 音频库的软件实现）是全平台通用的音频后端，Windows、Linux、macOS、Android 都需要它来播放音频。

</details>

### 题目 2：编写一个自定义 Triplet

假设你需要为 RISC-V 64 位 Linux 目标创建一个 vcpkg triplet（RISC-V 是一种开源指令集架构，类似于 ARM 但完全开放），请编写 triplet 文件并解释每一行。

<details>
<summary>查看答案</summary>

创建文件 `vcpkg/triplets/riscv64-linux.cmake`：

```cmake
# riscv64-linux.cmake — RISC-V 64 位 Linux 交叉编译 triplet

# 目标 CPU 架构：RISC-V 64 位
# vcpkg 用这个值来选择正确的编译器标志和库路径
set(VCPKG_TARGET_ARCHITECTURE riscv64)

# C 运行时链接方式：动态链接
# Linux 上通常使用动态链接的 glibc，减小二进制体积
set(VCPKG_CRT_LINKAGE dynamic)

# 第三方库链接方式：静态链接
# 静态链接简化部署——不需要在目标设备上安装这些库
set(VCPKG_LIBRARY_LINKAGE static)

# 目标操作系统：Linux
# vcpkg 据此决定使用 Unix-style 的路径和编译器
set(VCPKG_CMAKE_SYSTEM_NAME Linux)

# 交叉编译三元组：用于 autotools 构建系统的 --host 参数
# riscv64-unknown-linux-gnu 是 GCC 工具链的标准命名
set(VCPKG_MAKE_BUILD_TRIPLET "--host=riscv64-unknown-linux-gnu")

# CMake 交叉编译选项：指定 RISC-V 工具链路径
# TOOLCHAIN_PREFIX 告诉 CMake 编译器的前缀（如 riscv64-unknown-linux-gnu-gcc）
set(VCPKG_CMAKE_CONFIGURE_OPTIONS
    -DCMAKE_C_COMPILER=riscv64-unknown-linux-gnu-gcc
    -DCMAKE_CXX_COMPILER=riscv64-unknown-linux-gnu-g++
)
```

**对比 KrKr2 的 `arm64-android.cmake`：**

| 配置项 | arm64-android | riscv64-linux |
|--------|---------------|---------------|
| `TARGET_ARCHITECTURE` | arm64 | riscv64 |
| `CRT_LINKAGE` | static | dynamic |
| `LIBRARY_LINKAGE` | static | static |
| `CMAKE_SYSTEM_NAME` | Android | Linux |
| `MAKE_BUILD_TRIPLET` | `--host=aarch64-linux-android` | `--host=riscv64-unknown-linux-gnu` |

Android triplet 使用 static CRT 是因为 Android 上静态链接更可靠（避免系统 libc 版本差异），而 Linux 上通常使用 dynamic CRT 因为系统的 glibc 是标准化的。

</details>

### 题目 3：排查二进制缓存未命中的原因

你的 CI 流水线每次构建都要重新编译所有 28 个依赖，耗时 45 分钟。你已经配置了 `VCPKG_BINARY_SOURCES="clear;nuget,https://nuget.pkg.github.com/myorg/index.json,readwrite"`，但缓存似乎完全没有生效。请列出至少 4 个可能的原因并给出排查步骤。

<details>
<summary>查看答案</summary>

**可能原因及排查步骤：**

**原因 1：NuGet API Key 无效或权限不足**

```bash
# 排查：查看 vcpkg 输出中是否有 401/403 错误
# 在 CI 日志中搜索 "error" 或 "unauthorized"
grep -i "error\|unauthorized\|forbidden" ci-build.log

# 修复：确保 workflow 有 packages: write 权限
# permissions:
#   packages: write
```

**原因 2：vcpkg 版本不一致**

```bash
# 排查：不同 CI 运行使用了不同版本的 vcpkg
# vcpkg 版本变化会改变内部编译逻辑，导致哈希不同
vcpkg version  # 检查版本号

# 修复：在 CI 中固定 vcpkg 版本
# git clone https://github.com/microsoft/vcpkg.git
# cd vcpkg && git checkout 2024.01.12  # 固定到具体版本
```

**原因 3：编译器版本变化导致哈希不同**

```bash
# 排查：CI 环境的编译器更新了
gcc --version   # 检查 GCC 版本
cl.exe /?       # 检查 MSVC 版本（Windows）

# 二进制缓存的哈希包含编译器版本信息
# 如果 CI runner 的编译器从 GCC 13.1 升级到 13.2，所有缓存都会失效

# 修复：在 CI 中固定编译器版本
# - name: Install GCC
#   run: sudo apt-get install gcc-13=13.1.0-* g++-13=13.1.0-*
```

**原因 4：`clear` 关键字清除了默认缓存但 NuGet 源不可达**

```bash
# 排查：VCPKG_BINARY_SOURCES 以 "clear;" 开头会清除默认的本地缓存
# 如果 NuGet 源不可达，就没有任何缓存可用
curl -I "https://nuget.pkg.github.com/myorg/index.json"  # 测试 NuGet 源可达性

# 修复：添加本地缓存作为后备
# VCPKG_BINARY_SOURCES="clear;nuget,...,readwrite;files,$HOME/.cache/vcpkg/archives,readwrite"
```

**原因 5：overlay port 的源码文件被修改**

```bash
# 排查：如果 portfile.cmake 或补丁文件有任何变化，哈希会改变
git diff --name-only HEAD~1 -- vcpkg/ports/

# 修复：确保只在必要时修改 overlay port
```

**快速诊断脚本：**

```bash
#!/bin/bash
# diagnose-vcpkg-cache.sh — 诊断二进制缓存问题

echo "=== 1. vcpkg 版本 ==="
vcpkg version

echo "=== 2. 编译器版本 ==="
gcc --version 2>/dev/null || cl.exe /? 2>/dev/null | head -1

echo "=== 3. VCPKG_BINARY_SOURCES ==="
echo "$VCPKG_BINARY_SOURCES"

echo "=== 4. NuGet 源可达性 ==="
# 提取 URL（从 VCPKG_BINARY_SOURCES 中解析）
NUGET_URL=$(echo "$VCPKG_BINARY_SOURCES" | grep -oP 'nuget,\K[^,]+')
curl -s -o /dev/null -w "%{http_code}" "$NUGET_URL"

echo "=== 5. 本地缓存状态 ==="
du -sh ~/.cache/vcpkg/archives/ 2>/dev/null || echo "无本地缓存"

echo "=== 6. overlay port 最近修改 ==="
git log --oneline -5 -- vcpkg/ports/
```

</details>

---

## 下一步

恭喜你完成了 **M11-CI/CD 与 Docker** 模块的全部内容！

在本模块中，你学习了：
- **GitHub Actions 工作流**：YAML 语法、CI 流水线结构、矩阵构建
- **Docker 构建环境**：Dockerfile 编写、多阶段构建、本地 Docker 调试
- **代码质量门禁**：clang-format 格式化、clang-tidy 静态分析、vcpkg 二进制缓存与依赖管理

接下来请进入 [P12-现代跨平台 UI](../../P12-现代跨平台UI/README.md)，学习 Flutter 和 Compose Multiplatform 等现代 UI 框架，为后续的 M13-UI 框架替换实战做准备。
