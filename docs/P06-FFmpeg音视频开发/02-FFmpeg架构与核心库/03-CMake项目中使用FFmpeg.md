# CMake 项目中集成 FFmpeg

> **所属模块：** P06-FFmpeg音视频开发 > 02-FFmpeg架构与核心库
> **前置知识：** [P01-现代CMake与构建工具链](../../P01-现代CMake与构建工具链/)、[P02-vcpkg包管理](../../P02-vcpkg包管理/)、[02-六大核心库详解](./02-六大核心库详解.md)
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：
1. 通过 vcpkg（跨平台 C++ 包管理器）安装 FFmpeg 并在 CMake 中自动发现
2. 掌握三种 CMake 集成方式（`find_package`、`pkg-config`、手动查找）的适用场景与写法
3. 理解 KrKr2 项目实际的 FFmpeg CMake 集成方式，并能复现到自己的项目
4. 处理跨平台（Windows/Linux/macOS/Android）链接差异和常见编译错误

---

## 一、通过 vcpkg 安装 FFmpeg

### 1.1 为什么用 vcpkg

在 C++ 项目中引入 FFmpeg 的最大难题不是写代码，而是**把库装好、链接好**。FFmpeg 自身依赖几十个第三方库（x264、x265、opus、lame 等），手动编译非常痛苦。vcpkg（微软开源的 C++ 包管理器）能帮你一行命令搞定所有依赖的下载、编译和安装。

KrKr2 项目就使用 vcpkg 管理 FFmpeg 和其他 40 多个依赖库。

### 1.2 vcpkg.json 清单模式

KrKr2 使用 vcpkg 的清单模式（Manifest Mode），在项目根目录的 `vcpkg.json` 中声明依赖：

```json
{
  "name": "krkr2",
  "version-string": "1.5.0",
  "dependencies": [
    "ffmpeg",
    "openal-soft",
    "spdlog",
    "cocos2dx"
  ]
}
```

注意 KrKr2 的 `"ffmpeg"` 没有指定 features（功能特性），因为项目使用了自定义的 overlay port（覆盖端口）。这是一个重要细节——标准 vcpkg 的 FFmpeg 端口和 KrKr2 的不一样。

### 1.3 Overlay Port（覆盖端口）

KrKr2 在 `vcpkg/ports/ffmpeg/` 目录下放置了自己的 FFmpeg 端口定义：

```json
// vcpkg/ports/ffmpeg/vcpkg.json
{
  "name": "ffmpeg",
  "version": "3.3.9",
  "dependencies": [
    { "name": "vcpkg-cmake-get-vars", "host": true },
    { "name": "vcpkg-pkgconfig-get-modules", "host": true },
    "libiconv"
  ]
}
```

**为什么要用 overlay port？** 标准 vcpkg 中的 FFmpeg 版本可能是 6.x 或 7.x，但 KrKr2 的视频播放代码基于 Kodi/XBMC 架构，需要特定版本（3.3.9）的 API。overlay port 让项目锁定版本，避免上游更新导致 API 不兼容。

使用 overlay port 时，CMake 配置需要指定覆盖路径：

```bash
# 手动指定 overlay port 路径
cmake -B build -S . \
  -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
  -DVCPKG_OVERLAY_PORTS=./vcpkg/ports

# 或者在 CMakePresets.json 中配置（KrKr2 的做法）
```

### 1.4 安装命令（经典模式）

如果你不使用清单模式，也可以用命令行安装：

```bash
# Windows
vcpkg install ffmpeg:x64-windows

# Linux
vcpkg install ffmpeg:x64-linux

# macOS (Apple Silicon)
vcpkg install ffmpeg:arm64-osx

# macOS (Intel)
vcpkg install ffmpeg:x64-osx

# Android (交叉编译)
vcpkg install ffmpeg:arm64-android
```

安装完成后，vcpkg 会输出库的安装路径和可用的 CMake 集成方式。

---

## 二、CMake 集成方式一：find_package（推荐）

### 2.1 基本用法

`find_package` 是 CMake 查找外部库的标准机制。vcpkg 安装的 FFmpeg 提供了 CMake 配置文件（Config 模式），可以直接使用：

```cmake
cmake_minimum_required(VERSION 3.20)
project(ffmpeg_demo LANGUAGES CXX)

# 查找 FFmpeg，指定需要的组件（Component）
# COMPONENTS 后面列出你要用的库名（不带 lib 前缀）
find_package(FFMPEG COMPONENTS
    avutil        # 基础工具库（几乎必需）
    avformat      # 封装/解封装（读写容器格式）
    avcodec       # 编解码器
    swscale       # 图像缩放和像素格式转换
    swresample    # 音频重采样
    REQUIRED      # 找不到就报错终止
)

add_executable(demo main.cpp)

# 添加头文件搜索路径
# FFMPEG_INCLUDE_DIRS 包含所有组件的头文件目录
target_include_directories(demo PRIVATE ${FFMPEG_INCLUDE_DIRS})

# 链接库
# FFMPEG_LIBRARIES 包含所有请求组件的库文件
target_link_libraries(demo PRIVATE ${FFMPEG_LIBRARIES})
```

`find_package(FFMPEG ...)` 成功后，CMake 会设置以下变量：

| 变量 | 含义 | 示例值 |
|------|------|--------|
| `FFMPEG_FOUND` | 是否找到 | `TRUE` |
| `FFMPEG_INCLUDE_DIRS` | 头文件目录列表 | `/usr/include;/usr/include/x86_64-linux-gnu` |
| `FFMPEG_LIBRARIES` | 库文件路径列表 | `/usr/lib/x86_64-linux-gnu/libavformat.so;...` |
| `FFMPEG_VERSION` | FFmpeg 版本 | `6.1.1` |

### 2.2 KrKr2 的实际写法

KrKr2 的 `cpp/core/movie/CMakeLists.txt` 中的 FFmpeg 集成是本教程最重要的参考：

```cmake
# 文件：cpp/core/movie/CMakeLists.txt（简化版）
cmake_minimum_required(VERSION 3.19)
project(core_movie_module LANGUAGES CXX)

# 收集源文件
set(MOVIE_SOURCE_FILES
    ${MOVIE_PATH}/ffmpeg/DemuxFFmpeg.cpp      # 解封装器
    ${MOVIE_PATH}/ffmpeg/VideoCodecFFmpeg.cpp  # 视频解码
    ${MOVIE_PATH}/ffmpeg/AudioCodecFFmpeg.cpp  # 音频解码
    ${MOVIE_PATH}/ffmpeg/VideoPlayer.cpp       # 主播放循环
    ${MOVIE_PATH}/ffmpeg/Clock.cpp             # 时钟同步
    # ... 共 30+ 个源文件
)

# 构建为静态库
add_library(${PROJECT_NAME} STATIC ${MOVIE_SOURCE_FILES})

# 查找 FFmpeg —— 注意：没有 avformat！
# KrKr2 的解封装器直接包含 avformat 头文件，
# 但链接通过 ${FFMPEG_LIBRARIES} 自动包含
find_package(FFMPEG COMPONENTS
    avutil
    avfilter     # 视频滤镜（反交错等）
    avcodec
    swscale
    swresample
    REQUIRED
)

# 头文件路径设为 PUBLIC —— 依赖此模块的其他模块也能用
target_include_directories(${PROJECT_NAME}
    PUBLIC ${FFMPEG_INCLUDE_DIRS}
)

# 库链接设为 PRIVATE —— 只有本模块直接链接 FFmpeg
target_link_libraries(${PROJECT_NAME} PRIVATE
    ${FFMPEG_LIBRARIES}
)
```

**关键细节解读**：
- **COMPONENTS 没有列出 `avformat`**，但代码中大量使用 `<libavformat/avformat.h>`。这是因为 vcpkg 的 FFmpeg 包在安装 avcodec 时会自动安装 avformat 作为依赖，`${FFMPEG_LIBRARIES}` 实际包含了 avformat
- **头文件路径是 PUBLIC**：`core_sound_module` 等其他模块通过 `target_link_libraries(... PRIVATE core_movie_module)` 链接 movie 模块时，会自动获得 FFmpeg 头文件路径
- **库链接是 PRIVATE**：防止 FFmpeg 库符号泄露到上层模块的链接命令中

---

## 三、CMake 集成方式二：pkg-config

### 3.1 什么是 pkg-config

pkg-config（包配置工具）是 Linux/macOS 上的传统库发现机制。每个安装的库会在系统中注册一个 `.pc` 文件，描述自己的头文件路径、库路径和依赖关系。CMake 通过 `FindPkgConfig` 模块调用 pkg-config。

### 3.2 基本用法

```cmake
cmake_minimum_required(VERSION 3.20)
project(ffmpeg_demo LANGUAGES CXX)

# 加载 CMake 的 PkgConfig 支持模块
find_package(PkgConfig REQUIRED)

# 通过 pkg-config 查找 FFmpeg 各库
# IMPORTED_TARGET 会创建一个可直接链接的 CMake 目标
pkg_check_modules(LIBAV REQUIRED IMPORTED_TARGET
    libavformat      # 注意：pkg-config 名称带 lib 前缀
    libavcodec
    libavutil
    libswscale
    libswresample
)

add_executable(demo main.cpp)

# 链接 PkgConfig::LIBAV 即可——头文件路径自动包含
target_link_libraries(demo PRIVATE PkgConfig::LIBAV)
```

### 3.3 find_package vs pkg-config 对比

| 特性 | find_package | pkg-config |
|------|-------------|------------|
| **Windows 支持** | ✅ 原生支持 | ⚠️ 需要额外安装 pkg-config |
| **vcpkg 集成** | ✅ 自动配置 | ✅ vcpkg 也生成 .pc 文件 |
| **Android 交叉编译** | ✅ 通过 toolchain file 自动处理 | ⚠️ 需要设置 `PKG_CONFIG_PATH` |
| **创建 IMPORTED 目标** | ✅ 有些包提供 | ✅ `IMPORTED_TARGET` 关键字 |
| **KrKr2 使用** | ✅ 是 | ❌ 否 |
| **推荐场景** | 跨平台项目 | Linux/macOS 原生包 |

**建议**：跨平台项目优先用 `find_package`（KrKr2 的做法）。如果只需要在 Linux 上快速编译原型，`pkg-config` 更简单。

---

## 四、CMake 集成方式三：手动查找

当 `find_package` 和 `pkg-config` 都不可用时（比如使用预编译二进制包），可以手动查找每个库：

```cmake
cmake_minimum_required(VERSION 3.20)
project(ffmpeg_demo LANGUAGES CXX)

# 逐个查找头文件目录
# find_path 在系统路径中搜索包含指定文件的目录
find_path(AVFORMAT_INCLUDE_DIR
    libavformat/avformat.h    # 要查找的头文件
    HINTS ${FFMPEG_ROOT}/include  # 额外的搜索提示路径
)
find_path(AVCODEC_INCLUDE_DIR libavcodec/avcodec.h
    HINTS ${FFMPEG_ROOT}/include
)
find_path(AVUTIL_INCLUDE_DIR libavutil/avutil.h
    HINTS ${FFMPEG_ROOT}/include
)

# 逐个查找库文件
# find_library 在系统路径中搜索指定名称的库文件
find_library(AVFORMAT_LIBRARY avformat
    HINTS ${FFMPEG_ROOT}/lib
)
find_library(AVCODEC_LIBRARY avcodec
    HINTS ${FFMPEG_ROOT}/lib
)
find_library(AVUTIL_LIBRARY avutil
    HINTS ${FFMPEG_ROOT}/lib
)
find_library(SWSCALE_LIBRARY swscale
    HINTS ${FFMPEG_ROOT}/lib
)
find_library(SWRESAMPLE_LIBRARY swresample
    HINTS ${FFMPEG_ROOT}/lib
)

# 检查是否全部找到
include(FindPackageHandleStandardArgs)
find_package_handle_standard_args(FFmpeg DEFAULT_MSG
    AVFORMAT_LIBRARY AVCODEC_LIBRARY AVUTIL_LIBRARY
    SWSCALE_LIBRARY SWRESAMPLE_LIBRARY
    AVFORMAT_INCLUDE_DIR
)

add_executable(demo main.cpp)

target_include_directories(demo PRIVATE
    ${AVFORMAT_INCLUDE_DIR}
    ${AVCODEC_INCLUDE_DIR}
    ${AVUTIL_INCLUDE_DIR}
)

# 链接顺序很重要！高层库在前，底层库在后
target_link_libraries(demo PRIVATE
    ${AVFORMAT_LIBRARY}     # 依赖 avcodec
    ${AVCODEC_LIBRARY}      # 依赖 avutil
    ${SWSCALE_LIBRARY}      # 依赖 avutil
    ${SWRESAMPLE_LIBRARY}   # 依赖 avutil
    ${AVUTIL_LIBRARY}       # 最底层，放最后
)
```

**手动查找的缺点**：
- 代码量大，每个库都要写 `find_path` + `find_library`
- 容易遗漏传递依赖（比如 avformat 内部依赖的 zlib、bz2 等）
- 跨平台维护困难

**何时使用**：仅在无法使用 vcpkg 且系统没有 pkg-config 时（比如嵌入式设备或定制化 SDK）才考虑。

---

## 五、跨平台链接注意事项

### 5.1 平台差异总览

| 平台 | 安装方式 | 库文件格式 | 特殊注意 |
|------|---------|-----------|---------|
| **Windows** | vcpkg 或预编译包 | `.lib`（静态）/ `.dll`（动态） | 静态链接需额外系统库；动态链接需部署 DLL |
| **Linux** | vcpkg / apt / pacman | `.a`（静态）/ `.so`（动态） | 系统包版本可能偏旧（Ubuntu 22.04 默认 FFmpeg 4.x） |
| **macOS** | vcpkg / Homebrew | `.a`（静态）/ `.dylib`（动态） | Apple Silicon 路径为 `/opt/homebrew/`，Intel 为 `/usr/local/` |
| **Android** | vcpkg 交叉编译 | `.a`（静态）/ `.so`（动态） | 需要 NDK toolchain；Gradle 中配置 CMake 路径 |

### 5.2 Windows 静态链接的额外依赖

FFmpeg 在 Windows 上静态链接时，需要链接若干 Windows 系统库。这些库提供了 FFmpeg 内部使用的 Windows API：

```cmake
if(WIN32)
    target_link_libraries(demo PRIVATE
        bcrypt      # 加密 API（FFmpeg 的随机数生成和哈希计算）
        secur32     # 安全 API（TLS/SSL 支持）
        ws2_32      # Winsock 网络库（RTSP/HTTP 流媒体）
        Mfplat      # Media Foundation 平台库（硬件解码器支持）
        Mfuuid      # Media Foundation GUID 定义
        Strmiids    # DirectShow GUID 定义
    )
endif()
```

如果不链接这些库，会出现大量 `undefined reference` 或 `unresolved external symbol` 错误，而且报错信息指向的是 Windows API 函数名，很难直接关联到 FFmpeg。

### 5.3 Linux 系统包安装

如果不使用 vcpkg，可以用系统包管理器安装（但版本可能偏旧）：

```bash
# Ubuntu / Debian
sudo apt install libavformat-dev libavcodec-dev libavutil-dev \
                 libswscale-dev libswresample-dev libavfilter-dev

# Fedora / RHEL
sudo dnf install ffmpeg-devel

# Arch Linux
sudo pacman -S ffmpeg
```

**版本警告**：Ubuntu 22.04 LTS 默认仓库中的 FFmpeg 是 4.4.x，不包含一些新版 API（如 `AVChannelLayout` 结构体在 5.1+ 才引入）。如果需要新版，使用 vcpkg 或添加第三方 PPA。

### 5.4 macOS Homebrew 安装

```bash
# Apple Silicon (M1/M2/M3/M4)
brew install ffmpeg
# 安装路径：/opt/homebrew/Cellar/ffmpeg/

# Intel Mac
brew install ffmpeg
# 安装路径：/usr/local/Cellar/ffmpeg/
```

如果 CMake 找不到 Homebrew 安装的 FFmpeg，需要设置提示路径：

```cmake
if(APPLE)
    # Homebrew 在 Apple Silicon 和 Intel 上的路径不同
    execute_process(
        COMMAND brew --prefix
        OUTPUT_VARIABLE HOMEBREW_PREFIX
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    list(APPEND CMAKE_PREFIX_PATH "${HOMEBREW_PREFIX}")
endif()
```

### 5.5 Android 交叉编译

Android 平台不自带 FFmpeg，必须交叉编译。使用 vcpkg 可以大幅简化：

```cmake
# Android 的 CMakeLists.txt
if(ANDROID)
    find_package(FFMPEG COMPONENTS
        avutil avcodec avformat swscale swresample
        REQUIRED
    )
    target_link_libraries(demo PRIVATE
        ${FFMPEG_LIBRARIES}
        log         # Android 日志库（__android_log_print）
        android     # Android NDK 核心库
    )
endif()
```

在 Gradle 中配置 CMake 使用 vcpkg：

```groovy
// app/build.gradle
android {
    defaultConfig {
        externalNativeBuild {
            cmake {
                arguments "-DCMAKE_TOOLCHAIN_FILE=${vcpkgRoot}/scripts/buildsystems/vcpkg.cmake",
                          "-DVCPKG_TARGET_TRIPLET=arm64-android",
                          "-DVCPKG_OVERLAY_PORTS=../../vcpkg/ports"
            }
        }
    }
    externalNativeBuild {
        cmake {
            path "src/main/cpp/CMakeLists.txt"
        }
    }
}
```

---

## 六、FFmpeg 头文件的正确包含方式

FFmpeg 的头文件使用子目录结构，每个库的头文件在对应的 `libavXXX/` 子目录下：

```cpp
// ✅ 正确：使用 libavXXX/ 子目录前缀
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libavutil/avutil.h>
#include <libavutil/imgutils.h>
#include <libswscale/swscale.h>
#include <libswresample/swresample.h>
#include <libavfilter/avfilter.h>

// ❌ 错误：缺少子目录前缀
#include <avformat.h>     // 编译报错：找不到文件
#include <avcodec.h>      // 编译报错：找不到文件
```

**C++ 项目中的特殊处理**：FFmpeg 是纯 C 库，在 C++ 中包含其头文件时必须用 `extern "C"` 包裹，否则链接时会因 C++ 名称修饰（Name Mangling，编译器给函数名加前缀以支持重载）而找不到符号：

```cpp
// ✅ 正确：C++ 中包含 FFmpeg 头文件
extern "C" {
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libavutil/avutil.h>
}

// 💡 现代 FFmpeg（≥4.0）的头文件内部已包含 extern "C" 宏
// 所以大多数情况下不需要手动包裹，但加上不会出错
```

KrKr2 项目中，`DemuxFFmpeg.h` 的实际写法：

```cpp
// 文件：cpp/core/movie/ffmpeg/DemuxFFmpeg.h（简化版）
extern "C" {
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
}

class CDVDDemuxFFmpeg {
    AVFormatContext* m_pFormatContext = nullptr;
    AVIOContext* m_ioContext = nullptr;
    // ...
};
```

---

## 动手实践

### 实践：从零搭建一个使用 FFmpeg 的 CMake 项目

**目标**：创建一个最小的 CMake + vcpkg + FFmpeg 项目，编译通过并打印 FFmpeg 版本信息。

**步骤 1**：创建项目目录结构

```
ffmpeg_hello/
├── CMakeLists.txt
├── vcpkg.json
└── main.cpp
```

**步骤 2**：编写 `vcpkg.json`

```json
{
  "name": "ffmpeg-hello",
  "version-string": "1.0.0",
  "dependencies": [
    "ffmpeg"
  ]
}
```

**步骤 3**：编写 `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.20)
project(ffmpeg_hello LANGUAGES CXX)

# 设置 C++ 标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 查找 FFmpeg
find_package(FFMPEG COMPONENTS
    avutil
    avformat
    avcodec
    REQUIRED
)

# 打印找到的信息（调试用，确认路径正确）
message(STATUS "FFmpeg include dirs: ${FFMPEG_INCLUDE_DIRS}")
message(STATUS "FFmpeg libraries: ${FFMPEG_LIBRARIES}")

add_executable(ffmpeg_hello main.cpp)
target_include_directories(ffmpeg_hello PRIVATE ${FFMPEG_INCLUDE_DIRS})
target_link_libraries(ffmpeg_hello PRIVATE ${FFMPEG_LIBRARIES})

# Windows 静态链接额外依赖
if(WIN32)
    target_link_libraries(ffmpeg_hello PRIVATE
        bcrypt secur32 ws2_32
    )
endif()
```

**步骤 4**：编写 `main.cpp`

```cpp
#include <cstdio>

extern "C" {
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libavutil/avutil.h>
}

int main() {
    // 打印 FFmpeg 各库的版本号
    // avformat_version() 返回一个整数，格式为 (major << 16 | minor << 8 | micro)
    unsigned int ver_format = avformat_version();
    unsigned int ver_codec  = avcodec_version();
    unsigned int ver_util   = avutil_version();

    printf("========== FFmpeg 版本信息 ==========\n");
    printf("libavformat: %d.%d.%d\n",
           ver_format >> 16,             // 主版本号
           (ver_format >> 8) & 0xFF,     // 次版本号
           ver_format & 0xFF);           // 修订号
    printf("libavcodec:  %d.%d.%d\n",
           ver_codec >> 16,
           (ver_codec >> 8) & 0xFF,
           ver_codec & 0xFF);
    printf("libavutil:   %d.%d.%d\n",
           ver_util >> 16,
           (ver_util >> 8) & 0xFF,
           ver_util & 0xFF);

    // 打印编译时配置信息
    printf("\nlibavformat 构建配置:\n%s\n",
           avformat_configuration());

    // 打印 FFmpeg 许可证
    printf("\n许可证: %s\n", avformat_license());

    // 列出所有支持的容器格式（取前 10 个作为演示）
    printf("\n========== 支持的容器格式（前 10 个） ==========\n");
    void* opaque = nullptr;
    const AVInputFormat* ifmt = nullptr;
    int count = 0;
    while ((ifmt = av_demuxer_iterate(&opaque)) != nullptr && count < 10) {
        printf("  %-15s  %s\n", ifmt->name, ifmt->long_name);
        count++;
    }

    // 列出所有支持的编解码器（取前 10 个作为演示）
    printf("\n========== 支持的编解码器（前 10 个） ==========\n");
    opaque = nullptr;
    const AVCodec* codec = nullptr;
    count = 0;
    while ((codec = av_codec_iterate(&opaque)) != nullptr && count < 10) {
        const char* type_str = "其他";
        if (codec->type == AVMEDIA_TYPE_VIDEO) type_str = "视频";
        if (codec->type == AVMEDIA_TYPE_AUDIO) type_str = "音频";
        printf("  %-15s  [%s]  %s\n",
               codec->name,
               type_str,
               codec->long_name ? codec->long_name : "");
        count++;
    }

    printf("\n✅ FFmpeg 集成成功！\n");
    return 0;
}
```

**步骤 5**：编译运行（四个平台）

```bash
# Windows（需要已安装 vcpkg 并设置 VCPKG_ROOT 环境变量）
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%/scripts/buildsystems/vcpkg.cmake
cmake --build build --config Release
.\build\Release\ffmpeg_hello.exe

# Linux
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake
cmake --build build
./build/ffmpeg_hello

# macOS
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake
cmake --build build
./build/ffmpeg_hello

# Android（交叉编译后推送到设备运行）
cmake -B build -S . \
  -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
  -DVCPKG_TARGET_TRIPLET=arm64-android \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-24
cmake --build build
adb push build/ffmpeg_hello /data/local/tmp/
adb shell /data/local/tmp/ffmpeg_hello
```

**预期输出**（版本号因安装版本而异）：

```
========== FFmpeg 版本信息 ==========
libavformat: 57.83.100
libavcodec:  57.107.100
libavutil:   55.78.100

libavformat 构建配置:
--enable-shared --enable-pic ...

许可证: LGPL version 2.1 or later

========== 支持的容器格式（前 10 个） ==========
  aa               Audible AA format
  aac              raw ADTS AAC
  ...

✅ FFmpeg 集成成功！
```

---

## 对照项目源码

KrKr2 中 FFmpeg CMake 集成的完整链路：

| 文件 | 作用 | 关键行 |
|------|------|--------|
| `vcpkg.json` 第 34 行 | 声明 FFmpeg 依赖 | `"ffmpeg"` |
| `vcpkg/ports/ffmpeg/vcpkg.json` | 锁定 FFmpeg 3.3.9 版本 | `"version": "3.3.9"` |
| `cpp/core/movie/CMakeLists.txt` 第 52-58 行 | 查找 FFmpeg 5 个组件 | `find_package(FFMPEG COMPONENTS ...)` |
| `cpp/core/movie/CMakeLists.txt` 第 61 行 | PUBLIC 头文件路径 | `target_include_directories(... PUBLIC ${FFMPEG_INCLUDE_DIRS})` |
| `cpp/core/movie/CMakeLists.txt` 第 63-67 行 | PRIVATE 链接库 | `target_link_libraries(... PRIVATE ${FFMPEG_LIBRARIES})` |
| `cpp/core/sound/CMakeLists.txt` 第 38 行 | 通过 movie 模块间接获得 FFmpeg | `target_link_libraries(... PRIVATE core_movie_module)` |

**注意**：`core_sound_module` 中的 `FFWaveDecoder.cpp` 使用了 FFmpeg，但 sound 模块的 CMakeLists.txt 中并没有 `find_package(FFMPEG ...)`。这是因为 sound 模块通过 `target_link_libraries(... PRIVATE core_movie_module)` 链接了 movie 模块，而 movie 模块把 FFmpeg 头文件路径设为了 PUBLIC，所以 sound 模块自动获得了 FFmpeg 的头文件搜索路径和库链接。

这是 CMake 的**传递性（Transitivity）**特性：PUBLIC 依赖会传播给所有使用者。这也是为什么 KrKr2 中只需要在 movie 模块一处配置 FFmpeg，其他模块就能自动使用。

---

## 常见错误及解决方案

### 错误 1：静态链接时的 undefined reference

```
/usr/bin/ld: libavformat.a: undefined reference to `avcodec_parameters_copy'
```

**原因**：静态链接时，链接器从左到右处理库文件。如果 `avformat` 在 `avcodec` 之前被处理，但 `avformat` 依赖 `avcodec` 中的符号，链接器会报找不到符号。

**解决**：确保链接顺序从高层到底层（依赖者在前，被依赖者在后）：

```cmake
# ✅ 正确顺序：avformat → avcodec → swscale → swresample → avutil
target_link_libraries(demo PRIVATE
    ${AVFORMAT_LIBRARY}
    ${AVCODEC_LIBRARY}
    ${SWSCALE_LIBRARY}
    ${SWRESAMPLE_LIBRARY}
    ${AVUTIL_LIBRARY}        # avutil 是所有库的基础，放最后
)
```

使用 `find_package(FFMPEG ...)` 或 `pkg-config` 时通常不需要手动排序，CMake 会自动处理。

### 错误 2：头文件路径不对

```
fatal error: libavformat/avformat.h: No such file or directory
```

**解决**：检查两件事：
1. FFmpeg 头文件确实已安装
2. `target_include_directories` 指向了正确的目录

```bash
# 检查 FFmpeg 头文件位置
find /usr -name "avformat.h" 2>/dev/null
# 应输出类似：/usr/include/x86_64-linux-gnu/libavformat/avformat.h

# 如果使用 vcpkg，检查 vcpkg 安装路径
ls $VCPKG_ROOT/installed/x64-linux/include/libavformat/
```

### 错误 3：API 版本不匹配

```
error: 'avcodec_decode_video2' was not declared in this scope
```

**原因**：代码使用了旧版 FFmpeg API，但安装的是新版 FFmpeg（≥5.0 已移除这些函数）。

**解决**：将旧 API 替换为新 API：

```cpp
// 旧 API（已废弃/移除）        →  新 API
avcodec_decode_video2()          →  avcodec_send_packet() + avcodec_receive_frame()
avcodec_decode_audio4()          →  avcodec_send_packet() + avcodec_receive_frame()
avpicture_fill()                 →  av_image_fill_arrays()
avpicture_get_size()             →  av_image_get_buffer_size()
av_register_all()                →  不再需要（FFmpeg 4.0+ 自动注册）
avcodec_register_all()           →  不再需要（FFmpeg 4.0+ 自动注册）
```

---

## 本节小结

- **vcpkg 是跨平台 FFmpeg 集成的最佳方式**：一行 JSON 声明依赖，CMake 自动发现
- **KrKr2 使用 overlay port 锁定 FFmpeg 3.3.9**，确保 API 兼容性
- **三种 CMake 集成方式**：`find_package`（推荐，KrKr2 的做法）> `pkg-config`（Linux 简单项目）> 手动查找（最后手段）
- **链接顺序很重要**：静态链接时必须从高层库到底层库（avformat → avcodec → avutil）
- **跨平台差异**：Windows 需要额外系统库；macOS 注意 Homebrew 路径；Android 需要交叉编译 triplet
- **CMake 传递性**：KrKr2 只在 movie 模块配置 FFmpeg，其他模块通过 PUBLIC 依赖自动获得

---

## 练习题与答案

### 题目 1：分析 CMake 配置错误

以下 `CMakeLists.txt` 有 3 处问题，找出并修正：

```cmake
cmake_minimum_required(VERSION 3.20)
project(my_player LANGUAGES CXX)

find_package(FFMPEG REQUIRED)  # 问题 1

add_executable(player main.cpp)
target_link_libraries(player PRIVATE ${FFMPEG_LIBRARIES})
# 问题 2：缺少什么？

if(WIN32)
    # 问题 3：静态链接，缺少什么？
endif()
```

<details>
<summary>查看答案</summary>

**问题 1**：`find_package(FFMPEG REQUIRED)` 没有指定 COMPONENTS。虽然某些 vcpkg 配置下能工作，但最佳实践是明确指定需要的组件：

```cmake
find_package(FFMPEG COMPONENTS avutil avformat avcodec swscale swresample REQUIRED)
```

**问题 2**：缺少 `target_include_directories`。没有指定头文件搜索路径，`#include <libavformat/avformat.h>` 会找不到文件：

```cmake
target_include_directories(player PRIVATE ${FFMPEG_INCLUDE_DIRS})
```

**问题 3**：Windows 静态链接缺少系统库依赖：

```cmake
if(WIN32)
    target_link_libraries(player PRIVATE
        bcrypt secur32 ws2_32 Mfplat Mfuuid Strmiids
    )
endif()
```

完整修正版：

```cmake
cmake_minimum_required(VERSION 3.20)
project(my_player LANGUAGES CXX)

find_package(FFMPEG COMPONENTS
    avutil avformat avcodec swscale swresample
    REQUIRED
)

add_executable(player main.cpp)
target_include_directories(player PRIVATE ${FFMPEG_INCLUDE_DIRS})
target_link_libraries(player PRIVATE ${FFMPEG_LIBRARIES})

if(WIN32)
    target_link_libraries(player PRIVATE
        bcrypt secur32 ws2_32 Mfplat Mfuuid Strmiids
    )
endif()
```

</details>

### 题目 2：理解 CMake 传递性

KrKr2 中 `core_sound_module` 的 `FFWaveDecoder.cpp` 使用了 FFmpeg 的头文件和库。请回答：

1. `core_sound_module` 的 CMakeLists.txt 中有 `find_package(FFMPEG ...)` 吗？
2. 如果没有，`FFWaveDecoder.cpp` 是如何成功编译和链接的？
3. 如果把 `core_movie_module` 的 `target_include_directories` 从 `PUBLIC` 改为 `PRIVATE`，会发生什么？

<details>
<summary>查看答案</summary>

**1.** 没有。`core_sound_module` 的 CMakeLists.txt 中只有 `find_package(Vorbis ...)`, `find_package(Opus ...)`, `find_package(OpenAL ...)` 等，没有 FFmpeg。

**2.** 通过 CMake 的传递性机制：
- `core_sound_module` 的 CMakeLists.txt 有 `target_link_libraries(... PRIVATE core_movie_module)`
- `core_movie_module` 的 CMakeLists.txt 有 `target_include_directories(... PUBLIC ${FFMPEG_INCLUDE_DIRS})`
- `PUBLIC` 意味着任何链接 `core_movie_module` 的目标都自动获得 FFmpeg 头文件路径
- 同理，`${FFMPEG_LIBRARIES}` 的链接也通过模块依赖关系传递

**3.** `FFWaveDecoder.cpp` 的编译会失败，报错 `libavformat/avformat.h: No such file or directory`。因为 `PRIVATE` 意味着头文件路径只对 `core_movie_module` 自身有效，不传播给依赖者。

修复方法有两种：
- （推荐）保持 movie 模块的 `PUBLIC`，因为多个模块都需要 FFmpeg
- （不推荐）在 sound 模块中也添加 `find_package(FFMPEG ...)`，但这导致重复配置

</details>

### 题目 3：编写完整的 CMake 配置

为以下项目结构编写完整的 `CMakeLists.txt`，要求支持四个平台：

```
video_info/
├── CMakeLists.txt     ← 需要编写
├── vcpkg.json         ← 已有
├── src/
│   ├── main.cpp       ← 使用 avformat + avcodec + avutil
│   └── decoder.cpp    ← 使用 avcodec + swscale
└── include/
    └── decoder.h
```

<details>
<summary>查看答案</summary>

```cmake
cmake_minimum_required(VERSION 3.20)
project(video_info LANGUAGES CXX)

# C++ 标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 查找 FFmpeg（指定所有用到的组件）
find_package(FFMPEG COMPONENTS
    avutil
    avformat
    avcodec
    swscale
    REQUIRED
)

# 创建可执行目标
add_executable(video_info
    src/main.cpp
    src/decoder.cpp
)

# 头文件路径
target_include_directories(video_info PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/include   # 项目自己的头文件
    ${FFMPEG_INCLUDE_DIRS}                # FFmpeg 头文件
)

# 链接 FFmpeg 库
target_link_libraries(video_info PRIVATE
    ${FFMPEG_LIBRARIES}
)

# 平台特定配置
if(WIN32)
    # Windows 静态链接需要的系统库
    target_link_libraries(video_info PRIVATE
        bcrypt secur32 ws2_32 Mfplat Mfuuid Strmiids
    )
elseif(ANDROID)
    # Android 需要的 NDK 库
    target_link_libraries(video_info PRIVATE
        log android
    )
endif()
# Linux 和 macOS 不需要额外配置
```

</details>

---

## 下一步

下一节 [04-动手实践与总结](./04-动手实践与总结.md) 将通过一个完整的 ffprobe 迷你工具实践本章所有知识，包括：
- 使用 FFmpeg API 读取媒体文件信息
- 对照 KrKr2 项目源码中的 FFmpeg 使用
- 第二章的完整知识回顾与练习
