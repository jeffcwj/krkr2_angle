# 01-vcpkg.json解读

在前几章中，我们学习了 vcpkg 的基础用法。本章我们将深入剖析 KrKr2 项目的 `vcpkg.json` 配置文件。这是一个典型的现代 C++ 项目清单文件，涵盖了多平台构建、复杂依赖管理以及版本控制。

## 1. 核心清单配置

我们先来看 KrKr2 真实的 `vcpkg.json` 内容：

```json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/vcpkg-tool/main/docs/vcpkg.schema.json",
  "name": "krkr2",
  "version-string": "1.5.0",
  "dependencies": [
    "7zip",
    {"name": "argparse", "platform": "linux | windows | osx"},
    "boost-iostreams",
    {"name": "boost-locale", "default-features": false},
    "bullet3",
    "boost-phoenix",
    "boost-spirit",
    {"name": "breakpad", "default-features": false, "platform": "android"},
    {"name": "catch2", "default-features": false, "platform": "!android & !ios"},
    "cocos2dx",
    {"name": "dirent", "platform": "windows"},
    "ffmpeg",
    "fmt",
    {"name": "glfw3", "platform": "linux | windows | osx"},
    "jxrlib",
    "libarchive",
    {"name": "libgdiplus", "platform": "!windows"},
    "libjpeg-turbo",
    {"name": "libogg", "platform": "android"},
    {"name": "libpng", "platform": "!linux"},
    "libvorbis",
    "libwebp",
    "lz4",
    "oniguruma",
    "openal-soft",
    {"name": "opencv4", "default-features": false, "features": ["openmp"]},
    {"name": "opus", "platform": "android"},
    {"name": "opusfile", "default-features": false},
    {"name": "sdl2", "default-features": false},
    "spdlog",
    "uchardet",
    "unrar",
    {"name": "vcpkg-cmake", "host": true},
    "zstd"
  ],
  "overrides": [
    {"name": "opencv4", "version": "4.7.0#6"}
  ]
}
```

## 2. 依赖项分类详解

KrKr2 作为一个跨平台游戏框架，依赖库非常多。为了方便理解，我们可以将其分为以下几类：

### 核心框架类
- **cocos2dx**: 这是 KrKr2 的图形渲染核心。KrKr2 通过 vcpkg 集成了自定义版本的 Cocos2d-x，负责处理基础的精灵渲染、动作系统和跨平台窗口生命周期。

### 图像处理类
- **opencv4**: 用于复杂的图像处理，KrKr2 启用了其 `openmp` 特性以提升多核性能。
- **libjpeg-turbo / libpng / libwebp**: 基础图片格式编解码库，分别负责 JPG、PNG 和 WebP 格式。
- **jxrlib**: 微软开发的 JPEG-XR 编码库，在某些老旧资源中会用到。

### 音频与视频类
- **openal-soft**: 跨平台音频 API，负责 3D 音效和音频输出。
- **libvorbis / libogg / opus / opusfile**: 游戏音频常用的压缩格式。其中 `libogg` 和 `opus` 仅在 **Android** 平台引入。
- **ffmpeg**: 万能多媒体框架，用于处理视频播放和复杂的流媒体解码。

### 压缩与归档类
- **7zip / unrar**: 处理各种压缩包。
- **libarchive**: 一个可以读写多种归档格式（tar, zip, iso 等）的综合库。
- **lz4 / zstd**: 现代的高性能压缩库，常用于资源文件的实时解压。

### 脚本、文本与工具类
- **oniguruma**: 强大的正则表达式引擎，用于脚本解析。
- **boost-spirit / boost-phoenix**: 复杂的表达式解析引擎。
- **uchardet**: 文本编码自动识别。
- **argparse**: 命令行参数解析。

### 平台兼容与物理引擎
- **bullet3**: 高性能物理引擎，用于碰撞检测和物理模拟。
- **sdl2 / glfw3**: 窗口和输入管理。
- **dirent**: 在 Windows 下模拟 POSIX 的目录操作 API。
- **libgdiplus**: 在非 Windows 平台提供类似 GDI+ 的图形操作能力。

### 调试与测试类
- **catch2**: 现代 C++ 测试框架。
- **breakpad**: 谷歌开发的崩溃收集系统，仅用于 **Android** 捕获 Native 崩溃。
- **spdlog / fmt**: 高性能日志库和字符串格式化库。

## 3. 核心语法解析

### 3.1 平台过滤器 (platform)
在 `vcpkg.json` 中，我们可以看到类似 `"platform": "linux | windows | osx"` 的语法。这允许我们根据编译的目标平台按需安装依赖。
- **linux | windows | osx**: 表示该库在 **Windows**、**Linux** 和 **macOS** 平台都会安装。
- **!windows**: 表示除了 Windows 之外的所有平台。例如 `libgdiplus` 需要在 Linux、macOS 和 Android 上提供类似图形能力。
- **!android & !ios**: 表示在非 Android 且非 iOS 的桌面平台上安装。

### 3.2 依赖特性的控制 (features)
`"default-features": false` 是一个常用的选项。很多库（如 OpenCV、SDL2）默认会开启大量冗余特性。为了减小安装后的包大小并减少冲突，KrKr2 手动禁用了默认特性，仅按需开启。
- 例如：`{"name": "opencv4", "default-features": false, "features": ["openmp"]}` 仅保留核心功能并开启 OpenMP 支持。

### 3.3 版本重写 (overrides)
`"overrides"` 用于锁定特定库的版本。
- **为什么 opencv4 要锁定到 4.7.0#6？**
  - 在大型项目中，库的版本升级可能会引入 API 变更或不兼容。vcpkg 的 baseline 机制虽然能锁定大部分版本，但如果我们需要某个特定的修订版（如带 `#6` 修订补丁的），则必须使用 `overrides` 强制锁定。这确保了在不同开发者环境下编译结果的 100% 一致性。

### 3.4 Host 依赖项 (host)
在配置中有一行：`{"name": "vcpkg-cmake", "host": true}`。
- **Host 依赖**：指的是在“构建机器”上运行的工具，而不是链接到“目标程序”的库。
- 比如在 Windows 上交叉编译 Android 应用时，`vcpkg-cmake` 工具需要运行在 Windows 环境下，而其他的库（如 `zstd`）则需要编译为 Android 的 `.so` 文件。

## 练习题与答案

### 题目 1：在 `vcpkg.json` 中，如果我们想配置一个库仅在 Windows 和 macOS 下安装，`platform` 字段该如何填写？

<details>
<summary>查看答案</summary>

应当填写 `"platform": "windows | osx"`。

</details>

### 题目 2：为什么在跨平台项目中通常建议将 `sdl2` 的 `default-features` 设为 `false`？

<details>
<summary>查看答案</summary>

SDL2 的默认特性非常庞大，包含了很多可能在特定平台不需要的音频和视频后端。手动关闭默认特性并根据平台需求（如 Vulkan、DirectX、OpenGL）按需开启，可以显著加快编译速度并减少最终程序包的体积。

</details>

### 题目 3：`overrides` 字段的作用是什么？它与 `version-string` 有什么区别？

<details>
<summary>查看答案</summary>

`overrides` 是一个强制指令，用于覆盖 baseline 或依赖链中自动推导出的版本。`version-string` 仅定义当前项目的版本号。即使 baseline 指定了一个更新的版本，`overrides` 也会强制 vcpkg 回退或锁定到你指定的那个特定版本。

</details>

