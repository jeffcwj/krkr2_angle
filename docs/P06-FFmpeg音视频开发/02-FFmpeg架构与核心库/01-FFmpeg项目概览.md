# FFmpeg 项目概览

> **所属模块：** P06-FFmpeg 音视频开发 > 02-FFmpeg 架构与核心库
> **前置知识：** [01-音视频基础概念](../01-音视频基础概念/01-容器格式与编码格式.md)（容器、编码、采样率、像素格式等基础术语）、C 语言基础（指针、结构体、动态内存分配）
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：
1. 说出 FFmpeg 是什么，它解决了音视频开发中的哪些核心问题
2. 列出 FFmpeg 的三个命令行工具和七个核心库，说明各自的用途
3. 画出 FFmpeg 的整体架构图，理解库之间的依赖关系
4. 解释 FFmpeg 的 API 演进历程，区分旧 API 和新 API
5. 说明 KrKr2 项目中如何使用 FFmpeg 及使用了哪些库

---

## 一、FFmpeg 是什么

### 1.1 一句话定义

FFmpeg 是一个**开源的跨平台音视频处理框架**，提供了从编解码（Codec，将原始数据压缩/解压缩的算法）到封装/解封装（Mux/Demux，将压缩数据放入/取出容器文件的过程）、滤镜处理（Filter，对音视频数据进行变换的处理步骤）、格式转换的完整工具链。几乎所有你听说过的音视频软件——VLC、OBS、Kodi、Chrome、Android 系统媒体栈——内部都在使用 FFmpeg 的代码。

### 1.2 FFmpeg 解决的问题

在没有 FFmpeg 的年代，音视频开发者需要面对一个残酷的现实：

```
视频格式：MP4, MKV, AVI, FLV, WebM, MOV, TS, WMV...    （几十种容器）
视频编码：H.264, H.265, VP9, AV1, MPEG-2, MPEG-4...     （几十种编码）
音频格式：MP3, AAC, Vorbis, Opus, FLAC, PCM, WMA...      （几十种编码）
像素格式：YUV420P, NV12, RGB24, BGRA, YUV444P...         （上百种像素排列）
采样格式：S16, S32, FLT, FLTP, DBL...                     （十几种采样精度）
```

每种组合都需要单独的处理代码。FFmpeg 把这些全部统一到一套框架中，开发者只需要学一套 API，就能处理几乎所有音视频格式。

### 1.3 FFmpeg 的名称由来

"FF" 代表 "Fast Forward"（快进），"mpeg" 指 MPEG（Moving Picture Experts Group，动态影像专家组）视频标准。项目始于 2000 年，由 Fabrice Bellard 创建，目前由社区维护，遵循 LGPL 2.1 / GPL 2 许可证。

---

## 二、FFmpeg 的组成部分

FFmpeg 项目由**三个命令行工具**和**七个库**组成：

### 2.1 命令行工具

| 工具 | 用途 | 典型用法 |
|------|------|----------|
| `ffmpeg` | 格式转换、编码、滤镜处理 | `ffmpeg -i input.avi output.mp4` |
| `ffplay` | 简易播放器（基于 SDL，Simple DirectMedia Layer，一个跨平台多媒体库） | `ffplay test.mp4` |
| `ffprobe` | 媒体文件信息探测 | `ffprobe -show_streams test.mp4` |

这三个工具本身就是使用 FFmpeg 库构建的应用程序。我们开发 KrKr2 时不直接使用这些命令行工具，而是直接调用底层库的 C API。但在调试时，这三个工具非常有用：

```bash
# 用 ffprobe 查看文件信息（调试时最常用）
ffprobe -v quiet -show_format -show_streams test.mp4

# 输出示例：
# [FORMAT]
# filename=test.mp4
# nb_streams=2
# format_name=mov,mp4,m4a,3gp,3g2,mj2
# duration=120.000000
# bit_rate=5000000
# [/FORMAT]
# [STREAM]
# index=0
# codec_name=h264
# codec_type=video
# width=1920
# height=1080
# ...
```

```bash
# 用 ffplay 快速播放文件验证内容
ffplay -autoexit test.mp4

# 用 ffmpeg 转码（测试解码器是否正常）
ffmpeg -i input.mkv -c:v libx264 -c:a aac output.mp4
# -i: 输入文件
# -c:v libx264: 视频编码器使用 x264（H.264 编码器的开源实现）
# -c:a aac: 音频编码器使用 AAC
```

### 2.2 七个核心库

| 库名 | 全称含义 | 职责 | KrKr2 使用 |
|------|---------|------|-----------|
| `libavutil` | Audio Video Utility | 基础工具：内存管理、数学运算、像素格式定义、错误码、日志 | ✅ |
| `libavformat` | Audio Video Format | 封装/解封装：处理容器格式（MP4、MKV 等），读写数据包 | ✅ |
| `libavcodec` | Audio Video Codec | 编解码：压缩/解压缩音视频数据（H.264、AAC 等） | ✅ |
| `libswscale` | Software Scale | 图像缩放与像素格式转换（如 YUV → RGB） | ✅ |
| `libswresample` | Software Resample | 音频重采样（Resample，改变采样率）与格式转换 | ✅ |
| `libavfilter` | Audio Video Filter | 音视频滤镜框架：反交错（Deinterlace，将隔行扫描视频转为逐行扫描）、缩放、裁剪等 | ✅ |
| `libavdevice` | Audio Video Device | 设备输入/输出：摄像头采集、屏幕录制、音频设备 | ❌ |

> **KrKr2 使用了其中 6 个库**，未使用 `libavdevice`——因为 KrKr2 是游戏模拟器，只需要播放预录制的媒体文件，不需要采集摄像头或麦克风数据。

### 2.3 库名命名规律

注意所有库名的前缀规律：
- `libav*`：Audio Video 的缩写，处理音视频数据本身
- `libsw*`：Software 的缩写，软件实现的转换功能（区别于硬件加速）

---

## 三、FFmpeg 的整体架构

### 3.1 架构全景图

FFmpeg 的库之间有严格的层次关系和依赖方向：

```
┌──────────────────────────────────────────────────────────────┐
│                    你的应用程序（如 KrKr2）                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐  ┌─────────────┐                           │
│  │ libavformat  │  │ libavfilter │   ← 高层库               │
│  │ (容器格式)   │  │ (滤镜处理)  │     负责"组织"和"处理"     │
│  └──────┬──────┘  └──────┬──────┘                           │
│         │                │                                   │
│         ▼                ▼                                   │
│  ┌─────────────────────────────┐                            │
│  │        libavcodec            │   ← 中间层                │
│  │     (编解码器)               │     负责"压缩/解压缩"      │
│  └──────────────┬──────────────┘                            │
│                 │                                            │
│  ┌──────────────┼──────────────┐                            │
│  │              │              │                             │
│  ▼              ▼              ▼                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐                │
│  │libswscale│ │libsw     │ │  libavutil    │  ← 底层库     │
│  │(图像转换)│ │resample  │ │ (基础工具)    │    所有库共用  │
│  └──────────┘ │(音频转换)│ └──────────────┘                │
│               └──────────┘                                  │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 依赖关系详解

```
libavformat  ──依赖──▶ libavcodec ──依赖──▶ libavutil
libavfilter  ──依赖──▶ libavcodec ──依赖──▶ libavutil
libswscale   ──────────────────────依赖──▶ libavutil
libswresample ─────────────────────依赖──▶ libavutil
```

**关键规则**：所有库都依赖 `libavutil`，它是整个 FFmpeg 的地基。`libavformat` 和 `libavfilter` 还依赖 `libavcodec`。这意味着：
- 链接时必须按照"高层 → 底层"的顺序排列库
- 不能只链接 `libavformat` 而不链接 `libavcodec` 和 `libavutil`

```cmake
# ✅ 正确：被依赖的库放在后面
target_link_libraries(my_app PRIVATE
    avformat avfilter avcodec swscale swresample avutil
)

# ❌ 错误：底层库在前面，静态链接时会报 "undefined reference"
target_link_libraries(my_app PRIVATE
    avutil avcodec avformat  # 反了！
)
```

### 3.3 数据在库之间的流动

一个典型的视频播放流程中，数据依次经过多个库的处理：

```
媒体文件（test.mp4）
    │
    ▼  libavformat（解封装）
    │  avformat_open_input() → avformat_find_stream_info() → av_read_frame()
    │
  AVPacket（压缩数据包，Packet，编码后的一段数据）
    │
    ▼  libavcodec（解码）
    │  avcodec_send_packet() → avcodec_receive_frame()
    │
  AVFrame（原始帧，Frame，解码后的像素/采样数据）
    │
    ├──▶ libswscale（像素格式转换，如 YUV420P → BGRA）
    │    sws_scale()
    │
    └──▶ libswresample（音频格式转换，如 FLTP 44.1kHz → S16 48kHz）
         swr_convert()
    │
    ▼  你的渲染/播放代码
```

---

## 四、FFmpeg 的版本与 API 演进

### 4.1 为什么要了解 API 演进

FFmpeg 的 API 经历了多次重大变化。如果你在网上搜"FFmpeg 解码教程"，很可能找到旧 API 的代码——这些代码在现代 FFmpeg（5.0+）中已被**删除**，无法编译。了解 API 演进能帮你区分哪些教程可用、哪些已过时。

### 4.2 关键 API 变化时间线

| 时期 | 解码 API | 状态 |
|------|---------|------|
| FFmpeg 2.x 及更早 | `avcodec_decode_video2(ctx, frame, &got, pkt)` | 同步 API，一次调用完成解码 |
| FFmpeg 3.1（2016 年） | `avcodec_send_packet()` / `avcodec_receive_frame()` | 引入异步 API，新旧共存 |
| FFmpeg 4.x | 旧 API 标记为 `deprecated`（弃用） | 编译时产生警告 |
| FFmpeg 5.0（2022 年） | 旧 API **被删除** | 无法编译旧代码 |
| FFmpeg 6.x-7.x | `send/receive` 是唯一方式 | 稳定 |

### 4.3 新旧 API 对比

```cpp
// ──────── 旧 API（FFmpeg 4.x 及以前）──────────────────
// ❌ KrKr2 不使用，现代 FFmpeg 中已删除
int got_frame = 0;
int ret = avcodec_decode_video2(codec_ctx, frame, &got_frame, &packet);
if (ret < 0) {
    // 处理错误
}
if (got_frame) {
    // 成功解码一帧，处理 frame
}
// 问题：一次调用只能解码一帧，实际上编解码器内部可能缓存了多帧

// ──────── 新 API（FFmpeg 3.1+ ，KrKr2 使用）──────────
// ✅ 正确的现代写法
int ret = avcodec_send_packet(codec_ctx, &packet);  // 发送压缩数据
if (ret < 0) {
    // 处理错误
}
// 一个 packet 可能产生多个 frame（如 B 帧重排序），所以用循环
while (ret >= 0) {
    ret = avcodec_receive_frame(codec_ctx, frame);   // 接收解码后的帧
    if (ret == AVERROR(EAGAIN)) {
        break;   // 解码器需要更多输入数据，继续发送下一个 packet
    }
    if (ret == AVERROR_EOF) {
        break;   // 所有帧已解码完毕
    }
    if (ret < 0) {
        // 真正的错误
        break;
    }
    // 成功解码一帧，处理 frame
}
```

新 API 的优势：
1. **更好地处理编解码器延迟**：某些编解码器（如 H.264 的 B 帧）会内部缓存多帧再输出，新 API 天然支持
2. **统一音视频接口**：音频和视频都用 `send_packet` / `receive_frame`，旧 API 是两套不同的函数
3. **更清晰的状态机**：`EAGAIN` 表示需要更多输入，`EOF` 表示结束，语义明确

### 4.4 其他常见的 API 变更

```cpp
// 内存分配相关
av_mallocz_array(n, size)  →  av_calloc(n, size)        // FFmpeg 6.0+

// 图像工具函数
avpicture_fill()           →  av_image_fill_arrays()     // FFmpeg 4.0+
avpicture_get_size()       →  av_image_get_buffer_size() // FFmpeg 4.0+

// 通道布局
codec_ctx->channels        →  codec_ctx->ch_layout.nb_channels  // FFmpeg 5.1+
codec_ctx->channel_layout  →  codec_ctx->ch_layout              // FFmpeg 5.1+

// 注册编解码器（新版自动注册，无需调用）
av_register_all()          →  （删除，自动完成）          // FFmpeg 4.0+
avcodec_register_all()     →  （删除，自动完成）          // FFmpeg 4.0+
```

> **记住**：如果你在教程或 Stack Overflow 上看到 `av_register_all()`、`avcodec_decode_video2`、`avpicture_fill` 这些函数名，说明那篇文章已过时，不要照搬。

---

## 五、FFmpeg 在 KrKr2 中的角色

### 5.1 两大使用场景

KrKr2 模拟器中，FFmpeg 承担两大职责：

| 模块 | 源码路径 | 职责 | 使用的 FFmpeg 库 |
|------|---------|------|-----------------|
| 视频播放 | `cpp/core/movie/ffmpeg/` | 播放视觉小说的 OP/ED 动画、过场视频 | 全部 6 个库 |
| 音频解码 | `cpp/core/sound/FFWaveDecoder.cpp` | 解码 FFmpeg 支持的音频格式（BGM、SE） | avformat + avcodec + swresample + avutil |

### 5.2 视频播放模块架构

视频播放模块位于 `cpp/core/movie/ffmpeg/`，其架构来源于 Kodi（原名 XBMC，一个著名的开源媒体中心）项目。这套架构的特点是：

```
┌─────────────────────────────────────────────────────────────┐
│                    KrKr2 视频播放模块                         │
│                  (cpp/core/movie/ffmpeg/)                    │
│                                                             │
│  ┌─────────────────┐     ┌──────────────────────┐          │
│  │  DemuxFFmpeg     │     │  CDVDClock            │         │
│  │  (解封装器)      │     │  (同步时钟)           │         │
│  │  libavformat     │     │  libavutil             │        │
│  │  + 自定义 I/O    │     │  (av_rescale_q)       │         │
│  └────────┬────────┘     └──────────┬───────────┘          │
│           │ AVPacket                │ 时间参考              │
│     ┌─────┴──────┐                  │                       │
│     ▼            ▼                  │                       │
│  ┌──────────┐ ┌──────────┐         │                       │
│  │VideoCodec│ │AudioCodec│         │                       │
│  │FFmpeg    │ │FFmpeg    │         │                       │
│  │(视频解码)│ │(音频解码)│         │                       │
│  │libavcodec│ │libavcodec│         │                       │
│  │libswscale│ │libswre-  │         │                       │
│  │libavfilter│ │sample   │         │                       │
│  └────┬─────┘ └────┬─────┘         │                       │
│       │ AVFrame     │ AVFrame       │                       │
│       ▼             ▼               │                       │
│  ┌──────────────────────────────────┴───┐                  │
│  │           VideoPlayer (主循环)         │                  │
│  │           协调解封装/解码/渲染/同步     │                  │
│  └──────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

关键类说明：

| 类名 | 源文件 | 行数 | 职责 |
|------|-------|------|------|
| `CDVDDemuxFFmpeg` | `DemuxFFmpeg.cpp` | ~1656 行 | 解封装：打开媒体文件，从 XP3 归档中通过自定义 I/O 读取数据，拆分出视频包和音频包 |
| `CDVDVideoCodecFFmpeg` | `VideoCodecFFmpeg.cpp` | ~1112 行 | 视频解码：将压缩的视频数据解码为原始像素帧，支持硬件加速 |
| `CDVDAudioCodecFFmpeg` | `AudioCodecFFmpeg.cpp` | ~359 行 | 音频解码：将压缩的音频数据解码为 PCM（Pulse-Code Modulation，脉冲编码调制，即未压缩的数字音频）采样 |
| `CDVDClock` | `Clock.cpp` | ~275 行 | 同步时钟：管理音视频同步，使用 `av_rescale_q` 做时间基转换 |
| `CVideoPlayer` | `VideoPlayer.cpp` | ~2906 行 | 主循环：协调解封装、解码、渲染和音视频同步 |

### 5.3 音频解码模块

独立的音频解码器 `FFWaveDecoder`（位于 `cpp/core/sound/FFWaveDecoder.cpp`，约 451 行）是一个更简单的 FFmpeg 使用案例。它不需要视频相关的库，只使用 4 个库：

```cpp
// FFWaveDecoder 的简化工作流程
// 1. avformat_open_input()     → 打开音频文件（来自 XP3 归档）
// 2. avformat_find_stream_info() → 读取流信息
// 3. avcodec_open2()           → 打开音频解码器
// 4. av_read_frame()           → 读取压缩数据包
// 5. avcodec_send_packet()     → 发送到解码器
// 6. avcodec_receive_frame()   → 接收解码后的 PCM 数据
// 7. swr_convert()             → 转换为引擎需要的格式（16 位立体声）
```

### 5.4 KrKr2 使用新式 API

KrKr2 的 FFmpeg 代码全部使用**新式 API**（`avcodec_send_packet` / `avcodec_receive_frame`），说明代码是基于 FFmpeg 3.1+ 编写的。这意味着：
- 代码风格是现代的，你学到的知识可以直接应用到其他现代项目
- 网上旧教程中的 `avcodec_decode_video2` 不适用于本项目的代码
- 如果需要升级 FFmpeg 版本，不需要做 API 迁移

---

## 动手实践

### 实践 1：安装 FFmpeg 命令行工具并探测文件

在四个平台上安装 FFmpeg 工具：

```bash
# Windows（推荐使用 winget 或 choco）
winget install Gyan.FFmpeg
# 或者下载预编译二进制：https://www.gyan.dev/ffmpeg/builds/

# Linux（Ubuntu/Debian）
sudo apt install ffmpeg

# macOS
brew install ffmpeg

# Android（Termux 环境）
pkg install ffmpeg
```

安装后验证：

```bash
ffmpeg -version
# 输出：ffmpeg version 7.x ...

ffprobe -version
# 输出：ffprobe version 7.x ...
```

使用 `ffprobe` 探测任意媒体文件：

```bash
# 探测文件基本信息
ffprobe -v quiet -print_format json -show_format -show_streams test.mp4
```

### 实践 2：编写最小的 FFmpeg C 程序

创建以下项目结构：

```
ffmpeg_hello/
├── CMakeLists.txt
└── main.cpp
```

**CMakeLists.txt：**

```cmake
cmake_minimum_required(VERSION 3.20)
project(ffmpeg_hello LANGUAGES CXX)

find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBAV REQUIRED IMPORTED_TARGET
    libavformat
    libavcodec
    libavutil
)

add_executable(ffmpeg_hello main.cpp)
target_link_libraries(ffmpeg_hello PRIVATE PkgConfig::LIBAV)
```

**main.cpp：**

```cpp
// 最小的 FFmpeg 程序：打印版本信息 + 列出所有支持的编解码器
#include <cstdio>

// FFmpeg 是 C 语言库，在 C++ 中需要 extern "C" 包裹头文件
extern "C" {
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libavutil/avutil.h>
}

int main() {
    // 打印各库版本
    printf("libavformat 版本: %s\n", av_version_info());
    printf("libavcodec  版本: %d.%d.%d\n",
           LIBAVCODEC_VERSION_MAJOR,
           LIBAVCODEC_VERSION_MINOR,
           LIBAVCODEC_VERSION_MICRO);
    printf("libavutil   版本: %d.%d.%d\n",
           LIBAVUTIL_VERSION_MAJOR,
           LIBAVUTIL_VERSION_MINOR,
           LIBAVUTIL_VERSION_MICRO);

    // 列出前 10 个视频解码器
    printf("\n支持的视频解码器（前 10 个）：\n");
    void* opaque = nullptr;
    const AVCodec* codec = nullptr;
    int count = 0;
    while ((codec = av_codec_iterate(&opaque)) != nullptr) {
        // av_codec_is_decoder: 判断是否为解码器（而非编码器）
        if (av_codec_is_decoder(codec) &&
            codec->type == AVMEDIA_TYPE_VIDEO) {
            printf("  %s - %s\n", codec->name,
                   codec->long_name ? codec->long_name : "");
            if (++count >= 10) break;
        }
    }

    // 列出支持的容器格式（前 10 个）
    printf("\n支持的输入格式（前 10 个）：\n");
    opaque = nullptr;
    const AVInputFormat* ifmt = nullptr;
    count = 0;
    while ((ifmt = av_demuxer_iterate(&opaque)) != nullptr) {
        printf("  %s - %s\n", ifmt->name,
               ifmt->long_name ? ifmt->long_name : "");
        if (++count >= 10) break;
    }

    return 0;
}
```

编译运行：

```bash
# Windows
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%/scripts/buildsystems/vcpkg.cmake
cmake --build build --config Release
.\build\Release\ffmpeg_hello.exe

# Linux / macOS
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake
cmake --build build
./build/ffmpeg_hello
```

预期输出：

```
libavformat 版本: 7.1
libavcodec  版本: 61.19.100
libavutil   版本: 59.39.100

支持的视频解码器（前 10 个）：
  h264 - H.264 / AVC / MPEG-4 AVC / MPEG-4 part 10
  hevc - HEVC (High Efficiency Video Coding)
  mpeg2video - MPEG-2 video
  ...

支持的输入格式（前 10 个）：
  mov,mp4,m4a,3gp,3g2,mj2 - QuickTime / MOV
  matroska,webm - Matroska / WebM
  avi - AVI (Audio Video Interleaved)
  ...
```

---

## 常见错误及解决方案

### 错误 1：C++ 中包含 FFmpeg 头文件报链接错误

```
undefined reference to `avformat_open_input'
```

**原因**：FFmpeg 是纯 C 语言库，C++ 编译器会对函数名做 name mangling（名称修饰，C++ 编译器为支持重载而给函数名加的后缀），导致链接器找不到对应的 C 函数符号。

**解决**：用 `extern "C"` 包裹 FFmpeg 头文件：

```cpp
// ✅ C++ 中正确的包含方式
extern "C" {
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libavutil/avutil.h>
}
```

> 注意：某些版本的 FFmpeg 头文件内部已经有 `#ifdef __cplusplus extern "C"` 保护，但为了兼容性，建议始终自己加上。

### 错误 2：调用已删除的旧 API

```
error: 'avcodec_decode_video2' was not declared in this scope
```

**原因**：使用了 FFmpeg 5.0 中已被删除的旧解码 API。

**解决**：参考本节 4.4 的 API 映射表，将旧函数替换为新函数。

### 错误 3：忘记初始化指针为 nullptr

```cpp
AVFormatContext* fmt_ctx;  // ❌ 未初始化
avformat_open_input(&fmt_ctx, "test.mp4", nullptr, nullptr);
// 可能崩溃！因为 avformat_open_input 会检查 *fmt_ctx 是否为 nullptr
// 如果不为 nullptr，它认为你传入了已有的 context，尝试复用
```

**解决**：始终初始化为 `nullptr`：

```cpp
AVFormatContext* fmt_ctx = nullptr;  // ✅ 正确
```

---

## 本节小结

- **FFmpeg** 是开源的跨平台音视频处理框架，提供从编解码到封装/解封装的完整工具链
- FFmpeg 由 **3 个命令行工具**（ffmpeg、ffplay、ffprobe）和 **7 个库** 组成
- **核心库的层次关系**：libavutil（基础）→ libavcodec（编解码）→ libavformat/libavfilter（高层），libswscale 和 libswresample 是独立的转换库
- **链接顺序**：高层库在前，底层库在后（avformat → avcodec → avutil）
- **API 演进**：新式 API 使用 `send_packet` / `receive_frame`，旧式 `decode_video2` 已在 FFmpeg 5.0 中删除
- **KrKr2 使用 6 个 FFmpeg 库**（不含 libavdevice），代码基于新式 API，分为视频播放模块和音频解码模块两个使用场景

---

## 练习题与答案

### 题目 1：FFmpeg 库的层次关系

画出 FFmpeg 7 个库的依赖关系图，回答以下问题：
1. 哪个库是所有其他库的基础依赖？
2. 如果只需要做图像格式转换（如 YUV → RGB），至少需要链接哪些库？
3. 为什么 KrKr2 不使用 `libavdevice`？

<details>
<summary>查看答案</summary>

1. **libavutil** 是所有其他库的基础依赖。它提供内存管理、错误码、数学工具、像素/采样格式定义等基础功能。

2. 至少需要链接 **libswscale** 和 **libavutil**。libswscale 负责像素格式转换，它只依赖 libavutil：
```cmake
target_link_libraries(my_app PRIVATE swscale avutil)
```

3. KrKr2 是游戏模拟器，只需要**播放预录制的媒体文件**（OP/ED 动画、BGM、音效），不需要从摄像头或麦克风采集实时数据，因此不需要 libavdevice。

</details>

### 题目 2：API 版本判断

以下代码片段分别属于旧 API 还是新 API？哪些在 FFmpeg 7.x 中可以编译通过？

```cpp
// 片段 A
av_register_all();
AVFormatContext* ctx = avformat_alloc_context();

// 片段 B
avcodec_send_packet(codec_ctx, pkt);
avcodec_receive_frame(codec_ctx, frame);

// 片段 C
int got_picture;
avcodec_decode_video2(codec_ctx, frame, &got_picture, pkt);

// 片段 D
int size = avpicture_get_size(AV_PIX_FMT_RGB24, 1920, 1080);
```

<details>
<summary>查看答案</summary>

| 片段 | API 类型 | FFmpeg 7.x 能否编译 | 说明 |
|------|---------|---------------------|------|
| A | 旧 API | ❌ 不能 | `av_register_all()` 在 FFmpeg 4.0+ 中已删除（编解码器自动注册），但 `avformat_alloc_context()` 仍有效 |
| B | 新 API | ✅ 可以 | 这是现代标准的解码方式 |
| C | 旧 API | ❌ 不能 | `avcodec_decode_video2()` 在 FFmpeg 5.0 中完全删除 |
| D | 旧 API | ❌ 不能 | `avpicture_get_size()` 已被 `av_image_get_buffer_size()` 替代 |

</details>

### 题目 3：KrKr2 FFmpeg 使用分析

根据本节内容，回答：
1. KrKr2 的视频播放模块为什么需要"自定义 I/O"？
2. `CDVDVideoCodecFFmpeg` 类使用了哪些 FFmpeg 库？它为什么需要 libavfilter？
3. `FFWaveDecoder` 使用了 4 个库但不需要 libswscale，为什么？

<details>
<summary>查看答案</summary>

1. KrKr2 的媒体文件存储在 **XP3 归档包**中，不是独立的文件系统文件，无法用普通文件路径 `"path/to/video.mp4"` 直接打开。通过 FFmpeg 的 `AVIOContext` 自定义 I/O 机制，注册读取和 Seek 回调函数，FFmpeg 就能从 XP3 归档中读取数据。

2. `CDVDVideoCodecFFmpeg` 使用了 **libavcodec**（视频解码）、**libswscale**（像素格式转换）、**libavfilter**（滤镜处理）和 **libavutil**（基础工具）。它需要 libavfilter 是因为要做**视频后处理**，最典型的是反交错（deinterlace）——将隔行扫描的视频帧转换为逐行扫描，使用的是 yadif 滤镜。

3. `FFWaveDecoder` 是**音频解码器**，不处理任何视频/图像数据，因此不需要 libswscale（图像缩放和像素格式转换库）。它需要 libswresample 来将解码后的音频格式转换为引擎统一使用的格式（16 位立体声交织格式）。

</details>

---

## 下一步

下一节 [02-六大核心库详解](./02-六大核心库详解.md) 将深入讲解 FFmpeg 六大核心库的 API 细节，包括：
- 每个库的核心数据结构和关键函数
- 完整可运行的代码示例
- KrKr2 源码中的实际使用案例
