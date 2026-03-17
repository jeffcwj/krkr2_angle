# FFmpeg 架构与核心库

> **所属模块：** P06-FFmpeg 音视频开发
> **前置知识：** [01-音视频基础概念](./01-音视频基础概念.md)、C++ 基础（指针、结构体、动态内存）
> **预计阅读时间：** 35 分钟（约 7000 字）

## 本节目标

读完本节后，你将能够：
1. 画出 FFmpeg 的整体架构图，说明各核心库的职责边界
2. 解释 **libavformat**、**libavcodec**、**libavutil**、**libswscale**、**libswresample**、**libavfilter** 六大库的功能
3. 使用 FFmpeg API 完成最基本的"打开文件 → 读取信息 → 关闭"流程
4. 在 CMake 项目中正确链接 FFmpeg 库（含 vcpkg 集成）
5. 理解 KrKr2 项目中如何使用 FFmpeg 各库

---

## 一、FFmpeg 项目概览

### 1.1 FFmpeg 是什么？

FFmpeg 是一个开源的跨平台音视频处理框架，提供了从编解码到封装/解封装、滤镜处理、格式转换的完整工具链。它由以下部分组成：

| 组件 | 类型 | 说明 |
|------|------|------|
| `ffmpeg` | 命令行工具 | 格式转换、编码、滤镜处理 |
| `ffplay` | 命令行工具 | 简易播放器（基于 SDL） |
| `ffprobe` | 命令行工具 | 媒体文件信息探测 |
| `libavformat` | 库 | 封装/解封装（容器格式处理） |
| `libavcodec` | 库 | 编解码（所有编解码器） |
| `libavutil` | 库 | 通用工具（内存管理、数学、像素格式等） |
| `libswscale` | 库 | 图像缩放与像素格式转换 |
| `libswresample` | 库 | 音频重采样与格式转换 |
| `libavfilter` | 库 | 音视频滤镜框架 |
| `libavdevice` | 库 | 设备输入/输出（摄像头、屏幕录制等） |

> **KrKr2 使用了其中 6 个库**：libavformat、libavcodec、libavutil、libswscale、libswresample、libavfilter。未使用 libavdevice（模拟器不需要设备采集）。

### 1.2 FFmpeg 的版本与 API 演进

FFmpeg 的 API 经历了多次重大变化，了解这些变化对阅读 KrKr2 源码非常重要：

| 时期 | API 特征 | 关键变化 |
|------|---------|----------|
| FFmpeg 2.x 及更早 | `avcodec_decode_video2()` | 解码函数一步完成（同步 API） |
| FFmpeg 3.1+ (2016) | `avcodec_send_packet()` / `avcodec_receive_frame()` | 引入异步编解码 API |
| FFmpeg 4.x | 废弃旧 API | `avcodec_decode_video2()` 被标记为 deprecated |
| FFmpeg 5.0+ (2022) | 删除旧 API | 旧的解码函数被完全移除 |
| FFmpeg 6.x-7.x | 稳定的新 API | `send/receive` 成为唯一方式 |

KrKr2 使用的是**新式 API**（`avcodec_send_packet` / `avcodec_receive_frame`），这意味着代码是现代风格的，你在网上搜到的旧教程中的 `avcodec_decode_video2` 不适用于本项目。

```cpp
// ❌ 旧 API（KrKr2 不使用）
int got_frame = 0;
avcodec_decode_video2(codec_ctx, frame, &got_frame, &packet);

// ✅ 新 API（KrKr2 使用）
avcodec_send_packet(codec_ctx, &packet);
avcodec_receive_frame(codec_ctx, frame);
```

### 1.3 FFmpeg 在 KrKr2 中的角色

在 KrKr2 模拟器中，FFmpeg 承担两大职责：

1. **视频播放**（`cpp/core/movie/ffmpeg/`）：播放视觉小说的 OP/ED 动画
2. **音频解码**（`cpp/core/sound/FFWaveDecoder.cpp`）：解码 FFmpeg 支持的所有音频格式

```
┌─────────────────────────────────────────────────────┐
│                    KrKr2 模拟器                       │
├─────────────────────┬───────────────────────────────┤
│  视频播放模块        │  音频解码模块                  │
│  movie/ffmpeg/      │  sound/FFWaveDecoder.cpp      │
│                     │                               │
│  ┌─ DemuxFFmpeg ────┤  ┌─ FFWaveDecoder ───────────┐│
│  │  (libavformat)   │  │  (libavformat)            ││
│  │                  │  │  (libavcodec)             ││
│  ├─ VideoCodec ─────┤  │  (libswresample)          ││
│  │  (libavcodec)    │  └────────────────────────────┘│
│  │  (libswscale)    │                               │
│  │  (libavfilter)   │                               │
│  │                  │                               │
│  ├─ AudioCodec ─────┤                               │
│  │  (libavcodec)    │                               │
│  │  (libswresample) │                               │
│  └──────────────────┘                               │
└─────────────────────────────────────────────────────┘
```

---

## 二、六大核心库详解

### 2.1 libavutil —— 基础工具库

libavutil 是 FFmpeg 所有其他库的基础依赖，提供通用工具函数。其他五个库都链接了 libavutil。

**核心功能：**

| 功能域 | 关键类型/函数 | 说明 |
|--------|-------------|------|
| 内存管理 | `av_malloc()`, `av_free()`, `av_freep()` | FFmpeg 自己的内存分配器（对齐分配） |
| 错误处理 | `av_strerror()`, `AVERROR()`, `AVERROR_EOF` | 统一的错误码系统 |
| 像素格式 | `AVPixelFormat`, `av_get_pix_fmt_name()` | 枚举所有像素格式（YUV420P、RGB24 等） |
| 采样格式 | `AVSampleFormat`, `av_get_sample_fmt_name()` | 枚举所有音频采样格式 |
| 数学工具 | `AVRational`, `av_rescale_q()` | 有理数运算（时间基转换的核心） |
| 日志系统 | `av_log()`, `av_log_set_level()` | 统一日志输出 |
| 字典 | `AVDictionary`, `av_dict_set()`, `av_dict_get()` | 键值对存储（metadata、选项传递） |
| 缓冲区 | `AVBuffer`, `AVBufferRef`, `av_buffer_ref()` | 引用计数缓冲区（零拷贝的基础） |

**最重要的概念 —— AVRational 与时间基（time_base）：**

```cpp
#include <libavutil/rational.h>
#include <libavutil/mathematics.h>
#include <cstdio>

int main() {
    // AVRational 是一个分数：{分子, 分母}
    AVRational time_base = {1, 90000};  // 常见的 MPEG 时间基：1/90000 秒

    // 假设一个帧的 PTS 是 180000
    int64_t pts = 180000;

    // 将 PTS 转换为秒：pts * time_base = 180000 * (1/90000) = 2.0 秒
    double seconds = pts * av_q2d(time_base);  // av_q2d 将 AVRational 转为 double
    printf("PTS %lld = %.2f 秒\n", pts, seconds);
    // 输出：PTS 180000 = 2.00 秒

    // 在不同时间基之间转换 PTS
    AVRational src_tb = {1, 90000};     // 源时间基
    AVRational dst_tb = {1, 1000};      // 目标时间基（毫秒）
    int64_t dst_pts = av_rescale_q(pts, src_tb, dst_tb);
    printf("转换后 PTS = %lld（毫秒时间基）\n", dst_pts);
    // 输出：转换后 PTS = 2000（毫秒时间基）

    // av_rescale_q 内部使用 128 位运算避免溢出
    // 等价于：pts * src_tb.num * dst_tb.den / (src_tb.den * dst_tb.num)
    // 但直接算可能溢出 int64_t

    return 0;
}
```

> **KrKr2 中的使用**：`Clock.cpp` 中的 `CDVDClock` 大量使用 `av_rescale_q()` 进行时间基转换，将不同流的 PTS 统一到同一时钟域。

**错误处理模式：**

```cpp
#include <libavutil/error.h>
#include <cstdio>

// FFmpeg 函数返回负值表示错误
void check_error(int ret, const char* operation) {
    if (ret < 0) {
        char errbuf[AV_ERROR_MAX_STRING_SIZE];
        av_strerror(ret, errbuf, sizeof(errbuf));  // 将错误码转为可读字符串
        fprintf(stderr, "%s 失败: %s\n", operation, errbuf);
    }
}

// 常见错误码
// AVERROR_EOF          —— 文件末尾
// AVERROR(EAGAIN)      —— 需要更多输入数据
// AVERROR(ENOMEM)      —— 内存不足
// AVERROR(EINVAL)      —— 参数无效
// AVERROR_INVALIDDATA  —— 数据损坏
```

### 2.2 libavformat —— 封装/解封装库

libavformat 负责处理容器格式，它能识别文件格式、拆分出各个流（视频流、音频流、字幕流），并读取原始数据包（AVPacket）。

**核心数据结构：**

```
AVFormatContext（格式上下文 —— 最顶层容器）
├── nb_streams: 流数量
├── streams[]: AVStream 数组
│   ├── streams[0]: AVStream（视频流）
│   │   ├── codecpar: AVCodecParameters（编解码参数）
│   │   ├── time_base: AVRational（该流的时间基）
│   │   └── duration: 流时长
│   ├── streams[1]: AVStream（音频流）
│   └── streams[2]: AVStream（字幕流）
├── duration: 总时长
├── pb: AVIOContext*（I/O 上下文，自定义读取的关键）
└── iformat / oformat: 输入/输出格式描述
```

**关键 API 流程（解封装/读取）：**

```cpp
#include <libavformat/avformat.h>
#include <cstdio>

int main() {
    AVFormatContext* fmt_ctx = nullptr;

    // 第 1 步：打开文件（自动探测格式）
    int ret = avformat_open_input(&fmt_ctx, "test.mp4", nullptr, nullptr);
    if (ret < 0) {
        char errbuf[AV_ERROR_MAX_STRING_SIZE];
        av_strerror(ret, errbuf, sizeof(errbuf));
        fprintf(stderr, "无法打开文件: %s\n", errbuf);
        return 1;
    }
    // 此时 fmt_ctx 已被分配并填充了基本信息

    // 第 2 步：读取流信息（探测编码参数）
    ret = avformat_find_stream_info(fmt_ctx, nullptr);
    if (ret < 0) {
        fprintf(stderr, "无法获取流信息\n");
        avformat_close_input(&fmt_ctx);
        return 1;
    }

    // 第 3 步：遍历所有流
    printf("文件包含 %u 个流：\n", fmt_ctx->nb_streams);
    for (unsigned i = 0; i < fmt_ctx->nb_streams; i++) {
        AVStream* stream = fmt_ctx->streams[i];
        const char* type_name = av_get_media_type_string(
            stream->codecpar->codec_type);
        const char* codec_name = avcodec_get_name(
            stream->codecpar->codec_id);
        printf("  流 #%u: 类型=%s, 编码=%s, 时间基=%d/%d\n",
               i,
               type_name ? type_name : "unknown",
               codec_name,
               stream->time_base.num, stream->time_base.den);
    }

    // 第 4 步：读取数据包
    AVPacket* pkt = av_packet_alloc();  // 分配数据包
    int packet_count = 0;
    while (av_read_frame(fmt_ctx, pkt) >= 0) {
        packet_count++;
        if (packet_count <= 5) {  // 只打印前 5 个
            printf("  包 #%d: 流=%d, PTS=%lld, 大小=%d 字节\n",
                   packet_count,
                   pkt->stream_index,
                   pkt->pts,
                   pkt->size);
        }
        av_packet_unref(pkt);  // 释放包数据（重要！否则内存泄漏）
    }
    printf("共读取 %d 个数据包\n", packet_count);

    // 第 5 步：清理
    av_packet_free(&pkt);
    avformat_close_input(&fmt_ctx);  // 关闭文件并释放 fmt_ctx

    return 0;
}
```

**KrKr2 中的使用 —— `DemuxFFmpeg.cpp`**：

KrKr2 的 `CDVDDemuxFFmpeg` 类封装了上述流程，但有一个重要差异：它使用**自定义 I/O 回调**而非直接打开文件路径。这是因为 KrKr2 的数据可能来自 XP3 归档包：

```cpp
// 来源：cpp/core/movie/ffmpeg/DemuxFFmpeg.h
// CDVDDemuxFFmpeg 的关键成员
class CDVDDemuxFFmpeg : public CDVDDemux {
    AVFormatContext* m_pFormatContext = nullptr;  // 格式上下文
    AVIOContext* m_ioContext = nullptr;           // 自定义 I/O（从 IStream 读取）
    std::vector<CDemuxStream*> m_streams;         // 解析出的流列表

    // 自定义读取回调 —— 从 COM IStream 读取数据
    static int dvd_file_read(void* h, uint8_t* buf, int size);
    static int64_t dvd_file_seek(void* h, int64_t pos, int whence);
};
```

> **为什么需要自定义 I/O？** KrKr2 的媒体文件存储在 XP3 归档包中，不能用普通文件路径打开。通过 `avio_alloc_context()` 注册自定义的 read/seek 回调，FFmpeg 就能从任意数据源读取——这是 FFmpeg 架构灵活性的典型体现。

### 2.3 libavcodec —— 编解码库

libavcodec 是 FFmpeg 最核心的库，包含了上百种音视频编解码器的实现。

**核心数据结构：**

```
AVCodec（编解码器描述 —— 静态信息）
├── name: 编解码器名称（如 "h264", "aac"）
├── type: 类型（视频/音频/字幕）
├── id: AVCodecID（如 AV_CODEC_ID_H264）
└── capabilities: 能力标志

AVCodecContext（编解码器上下文 —— 运行时状态）
├── codec: 指向 AVCodec
├── width, height: 视频尺寸
├── pix_fmt: 像素格式
├── sample_rate: 音频采样率
├── channels: 音频通道数
├── time_base: 时间基
├── extradata: 编解码器额外数据（SPS/PPS 等）
└── opaque: 用户自定义数据指针

AVPacket（压缩数据包 —— 编码后的数据）
├── data: 压缩数据指针
├── size: 数据大小
├── pts: 显示时间戳
├── dts: 解码时间戳
├── stream_index: 所属流索引
└── flags: 标志（如 AV_PKT_FLAG_KEY 关键帧）

AVFrame（原始数据帧 —— 解码后的数据）
├── data[8]: 数据平面指针（YUV 有 3 个平面）
├── linesize[8]: 每行字节数（含对齐填充）
├── width, height: 视频帧尺寸
├── format: 像素/采样格式
├── pts: 显示时间戳
├── nb_samples: 音频帧的采样数
└── channels: 音频通道数
```

**解码流程（新 API）：**

```cpp
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <cstdio>

// 完整的视频解码示例
int decode_video(const char* filename) {
    AVFormatContext* fmt_ctx = nullptr;
    int ret;

    // 打开文件
    if ((ret = avformat_open_input(&fmt_ctx, filename, nullptr, nullptr)) < 0)
        return ret;
    if ((ret = avformat_find_stream_info(fmt_ctx, nullptr)) < 0)
        return ret;

    // 找到视频流
    int video_stream_idx = -1;
    for (unsigned i = 0; i < fmt_ctx->nb_streams; i++) {
        if (fmt_ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {
            video_stream_idx = i;
            break;
        }
    }
    if (video_stream_idx < 0) {
        fprintf(stderr, "找不到视频流\n");
        return -1;
    }

    AVStream* video_stream = fmt_ctx->streams[video_stream_idx];

    // 查找解码器
    const AVCodec* decoder = avcodec_find_decoder(
        video_stream->codecpar->codec_id);
    if (!decoder) {
        fprintf(stderr, "找不到解码器: %s\n",
                avcodec_get_name(video_stream->codecpar->codec_id));
        return -1;
    }
    printf("使用解码器: %s\n", decoder->name);

    // 创建解码器上下文
    AVCodecContext* codec_ctx = avcodec_alloc_context3(decoder);
    // 从流参数复制到解码器上下文
    avcodec_parameters_to_context(codec_ctx, video_stream->codecpar);
    // 打开解码器
    if ((ret = avcodec_open2(codec_ctx, decoder, nullptr)) < 0) {
        fprintf(stderr, "无法打开解码器\n");
        return ret;
    }

    printf("视频: %dx%d, 像素格式: %s\n",
           codec_ctx->width, codec_ctx->height,
           av_get_pix_fmt_name(codec_ctx->pix_fmt));

    // 解码循环
    AVPacket* pkt = av_packet_alloc();
    AVFrame* frame = av_frame_alloc();
    int frame_count = 0;

    while (av_read_frame(fmt_ctx, pkt) >= 0) {
        if (pkt->stream_index != video_stream_idx) {
            av_packet_unref(pkt);
            continue;  // 跳过非视频包
        }

        // 发送压缩包到解码器
        ret = avcodec_send_packet(codec_ctx, pkt);
        av_packet_unref(pkt);
        if (ret < 0) {
            fprintf(stderr, "发送包失败\n");
            break;
        }

        // 从解码器接收解码后的帧（一个包可能产生多个帧）
        while (ret >= 0) {
            ret = avcodec_receive_frame(codec_ctx, frame);
            if (ret == AVERROR(EAGAIN)) {
                break;  // 需要更多输入数据
            }
            if (ret == AVERROR_EOF) {
                goto end;  // 解码完毕
            }
            if (ret < 0) {
                fprintf(stderr, "接收帧失败\n");
                goto end;
            }

            frame_count++;
            if (frame_count <= 3) {
                printf("帧 #%d: PTS=%lld, 类型=%c, 尺寸=%dx%d\n",
                       frame_count,
                       frame->pts,
                       av_get_picture_type_char(frame->pict_type),
                       frame->width, frame->height);
            }

            av_frame_unref(frame);  // 释放帧数据
        }
    }

end:
    printf("共解码 %d 帧\n", frame_count);

    // 清理
    av_frame_free(&frame);
    av_packet_free(&pkt);
    avcodec_free_context(&codec_ctx);
    avformat_close_input(&fmt_ctx);

    return 0;
}

int main() {
    return decode_video("test.mp4");
}
```

**KrKr2 中的使用 —— `VideoCodecFFmpeg.cpp`**：

```cpp
// 来源：cpp/core/movie/ffmpeg/VideoCodecFFmpeg.h
// CDVDVideoCodecFFmpeg 封装了视频解码流程
class CDVDVideoCodecFFmpeg : public CDVDVideoCodec {
    AVCodecContext* m_pCodecContext = nullptr;  // 解码器上下文
    AVFrame* m_pFrame = nullptr;               // 解码输出帧
    AVFrame* m_pFilteredFrame = nullptr;        // 滤镜处理后的帧

    // 硬件解码抽象接口
    class IHardwareDecoder {
    public:
        virtual bool Open(AVCodecContext* avctx,
                         AVCodecContext* mainctx,
                         enum AVPixelFormat fmt) = 0;
        virtual CDVDVideoCodec::VCReturn Decode(AVCodecContext* avctx,
                                                AVFrame* frame) = 0;
    };
    std::shared_ptr<IHardwareDecoder> m_pHardware;
};
```

> **注意**：KrKr2 的视频解码器还集成了 `libavfilter` 用于视频后处理（反交错、缩放等），以及 `IHardwareDecoder` 接口用于硬件加速解码。这些高级特性我们在后续章节详细讲解。

### 2.4 libswscale —— 图像缩放与格式转换

libswscale 用于在不同像素格式之间转换，以及缩放视频帧。这在播放器中是必需的——解码器输出的格式（通常是 YUV420P）往往不是渲染器需要的格式（通常是 RGB 或 BGRA）。

**核心 API：**

```cpp
#include <libswscale/swscale.h>
#include <libavutil/imgutils.h>
#include <libavutil/frame.h>
#include <cstdio>

int main() {
    int src_w = 1920, src_h = 1080;  // 源尺寸
    int dst_w = 960,  dst_h = 540;   // 目标尺寸

    // 创建转换上下文：YUV420P → BGRA，同时缩放到一半
    SwsContext* sws_ctx = sws_getContext(
        src_w, src_h, AV_PIX_FMT_YUV420P,   // 源格式
        dst_w, dst_h, AV_PIX_FMT_BGRA,       // 目标格式
        SWS_BILINEAR,                          // 缩放算法
        nullptr, nullptr, nullptr);

    if (!sws_ctx) {
        fprintf(stderr, "无法创建 SwsContext\n");
        return 1;
    }

    // 分配源缓冲区（模拟解码器输出的 YUV420P 帧）
    uint8_t* src_data[4];
    int src_linesize[4];
    av_image_alloc(src_data, src_linesize, src_w, src_h,
                   AV_PIX_FMT_YUV420P, 32);  // 32 字节对齐

    // 分配目标缓冲区（BGRA 只有一个平面）
    uint8_t* dst_data[4];
    int dst_linesize[4];
    av_image_alloc(dst_data, dst_linesize, dst_w, dst_h,
                   AV_PIX_FMT_BGRA, 32);

    // 执行转换
    int output_height = sws_scale(
        sws_ctx,
        src_data, src_linesize,   // 源
        0, src_h,                  // 从第 0 行开始，处理 src_h 行
        dst_data, dst_linesize);   // 目标

    printf("转换完成，输出 %d 行\n", output_height);
    printf("源 linesize: Y=%d, U=%d, V=%d\n",
           src_linesize[0], src_linesize[1], src_linesize[2]);
    printf("目标 linesize: BGRA=%d\n", dst_linesize[0]);

    // 清理
    av_freep(&src_data[0]);
    av_freep(&dst_data[0]);
    sws_freeContext(sws_ctx);

    return 0;
}
```

**缩放算法选择：**

| 算法 | 常量 | 速度 | 质量 | 适用场景 |
|------|------|------|------|---------|
| 最近邻 | `SWS_POINT` | ★★★★★ | ★ | 像素风游戏、整数倍缩放 |
| 双线性 | `SWS_BILINEAR` | ★★★★ | ★★★ | 实时播放（KrKr2 的选择） |
| 双三次 | `SWS_BICUBIC` | ★★★ | ★★★★ | 高质量缩放 |
| Lanczos | `SWS_LANCZOS` | ★★ | ★★★★★ | 最高质量（视频转码） |
| 面积平均 | `SWS_AREA` | ★★★ | ★★★★ | 缩小图像 |

> **KrKr2 中的使用**：`VideoCodecFFmpeg.cpp` 使用 libswscale 将解码器输出的 YUV 帧转换为渲染器所需的像素格式。具体转换路径取决于渲染后端（OpenGL 使用 YUV420P 直接上传纹理，软件渲染使用 BGRA）。

### 2.5 libswresample —— 音频重采样库

libswresample 用于音频采样率转换、通道布局转换和采样格式转换。在播放器中，解码器输出的音频格式可能与音频设备期望的格式不同，需要用 libswresample 进行转换。

**核心 API：**

```cpp
#include <libswresample/swresample.h>
#include <libavutil/channel_layout.h>
#include <libavutil/samplefmt.h>
#include <cstdio>

int main() {
    SwrContext* swr_ctx = nullptr;
    int ret;

    // 使用新式 API（FFmpeg 5.1+）创建重采样上下文
    AVChannelLayout src_ch_layout = AV_CHANNEL_LAYOUT_STEREO;    // 源：立体声
    AVChannelLayout dst_ch_layout = AV_CHANNEL_LAYOUT_STEREO;    // 目标：立体声

    ret = swr_alloc_set_opts2(
        &swr_ctx,
        &dst_ch_layout,              // 目标通道布局
        AV_SAMPLE_FMT_S16,           // 目标采样格式：16 位有符号整数
        48000,                        // 目标采样率：48kHz
        &src_ch_layout,              // 源通道布局
        AV_SAMPLE_FMT_FLTP,         // 源采样格式：32 位浮点（planar）
        44100,                        // 源采样率：44.1kHz
        0, nullptr);

    if (ret < 0 || !swr_ctx) {
        fprintf(stderr, "无法创建 SwrContext\n");
        return 1;
    }

    // 初始化重采样器
    if ((ret = swr_init(swr_ctx)) < 0) {
        fprintf(stderr, "无法初始化 SwrContext\n");
        return 1;
    }

    // 模拟转换 1024 个采样
    int src_nb_samples = 1024;

    // 计算目标采样数（采样率不同，数量也不同）
    int dst_nb_samples = av_rescale_rnd(
        swr_get_delay(swr_ctx, 44100) + src_nb_samples,
        48000,   // 目标采样率
        44100,   // 源采样率
        AV_ROUND_UP);

    printf("源采样数: %d, 目标采样数: %d\n",
           src_nb_samples, dst_nb_samples);

    // 分配源缓冲区（FLTP 是 planar 格式，每个通道独立）
    uint8_t* src_data[2];  // 2 个通道
    src_data[0] = (uint8_t*)av_malloc(src_nb_samples * sizeof(float));
    src_data[1] = (uint8_t*)av_malloc(src_nb_samples * sizeof(float));

    // 分配目标缓冲区（S16 是 interleaved 格式，所有通道交织）
    uint8_t* dst_data[1];
    dst_data[0] = (uint8_t*)av_malloc(
        dst_nb_samples * 2 * sizeof(int16_t));  // 2 通道 × 16 位

    // 执行转换
    int converted = swr_convert(
        swr_ctx,
        dst_data, dst_nb_samples,                      // 目标
        (const uint8_t**)src_data, src_nb_samples);    // 源

    printf("转换了 %d 个采样\n", converted);

    // 清理
    av_free(src_data[0]);
    av_free(src_data[1]);
    av_free(dst_data[0]);
    swr_free(&swr_ctx);

    return 0;
}
```

**Planar vs Interleaved 格式（重要概念）：**

```
Interleaved（交织）格式，如 AV_SAMPLE_FMT_S16：
  [L0][R0][L1][R1][L2][R2]...
  所有通道数据混在一个数组中

Planar（平面）格式，如 AV_SAMPLE_FMT_FLTP：
  通道 0: [L0][L1][L2]...
  通道 1: [R0][R1][R2]...
  每个通道是独立的数组（AVFrame::data[0] 和 data[1]）
```

FFmpeg 解码器通常输出 planar 格式（以 P 结尾，如 FLTP、S16P），而音频设备通常期望 interleaved 格式（如 S16、FLT）。libswresample 能在两种格式之间转换。

> **KrKr2 中的使用**：`AudioCodecFFmpeg.cpp` 和 `FFWaveDecoder.cpp` 都使用 libswresample 将解码器输出转换为引擎音频系统期望的格式（通常是 16 位立体声交织格式）。

### 2.6 libavfilter —— 滤镜框架

libavfilter 提供音视频滤镜处理能力，可以构建复杂的滤镜链（filter graph）。

**滤镜图概念：**

```
                    滤镜图（Filter Graph）
┌──────────────────────────────────────────────────┐
│                                                  │
│  [buffer] ──→ [yadif] ──→ [scale] ──→ [buffersink]  │
│   (输入)     (反交错)    (缩放)      (输出)      │
│                                                  │
└──────────────────────────────────────────────────┘

buffer:     将 AVFrame 送入滤镜图
yadif:      反交错滤镜（将隔行扫描转为逐行）
scale:      缩放滤镜
buffersink: 从滤镜图取出处理后的 AVFrame
```

**基础用法：**

```cpp
#include <libavfilter/avfilter.h>
#include <libavfilter/buffersrc.h>
#include <libavfilter/buffersink.h>
#include <libavutil/opt.h>
#include <cstdio>

// 创建一个简单的视频滤镜图：缩放到 640x360
int create_filter_graph(int src_w, int src_h,
                        AVPixelFormat src_fmt) {
    AVFilterGraph* graph = avfilter_graph_alloc();
    if (!graph) return -1;

    // 创建输入滤镜（buffer）
    char args[512];
    snprintf(args, sizeof(args),
             "video_size=%dx%d:pix_fmt=%d:time_base=1/25",
             src_w, src_h, src_fmt);

    const AVFilter* buffersrc = avfilter_get_by_name("buffer");
    AVFilterContext* src_ctx = nullptr;
    int ret = avfilter_graph_create_filter(
        &src_ctx, buffersrc, "in", args, nullptr, graph);
    if (ret < 0) {
        fprintf(stderr, "无法创建 buffer 滤镜\n");
        avfilter_graph_free(&graph);
        return ret;
    }

    // 创建输出滤镜（buffersink）
    const AVFilter* buffersink = avfilter_get_by_name("buffersink");
    AVFilterContext* sink_ctx = nullptr;
    ret = avfilter_graph_create_filter(
        &sink_ctx, buffersink, "out", nullptr, nullptr, graph);
    if (ret < 0) {
        fprintf(stderr, "无法创建 buffersink 滤镜\n");
        avfilter_graph_free(&graph);
        return ret;
    }

    // 创建缩放滤镜
    const AVFilter* scale_filter = avfilter_get_by_name("scale");
    AVFilterContext* scale_ctx = nullptr;
    ret = avfilter_graph_create_filter(
        &scale_ctx, scale_filter, "scale",
        "640:360", nullptr, graph);  // 目标尺寸
    if (ret < 0) {
        fprintf(stderr, "无法创建 scale 滤镜\n");
        avfilter_graph_free(&graph);
        return ret;
    }

    // 连接滤镜：buffer → scale → buffersink
    avfilter_link(src_ctx, 0, scale_ctx, 0);
    avfilter_link(scale_ctx, 0, sink_ctx, 0);

    // 配置滤镜图
    ret = avfilter_graph_config(graph, nullptr);
    if (ret < 0) {
        fprintf(stderr, "滤镜图配置失败\n");
        avfilter_graph_free(&graph);
        return ret;
    }

    printf("滤镜图创建成功\n");
    printf("滤镜图描述: %s\n", avfilter_graph_dump(graph, nullptr));

    // 使用方式：
    // av_buffersrc_add_frame(src_ctx, decoded_frame);   // 送入帧
    // av_buffersink_get_frame(sink_ctx, filtered_frame); // 取出处理后的帧

    avfilter_graph_free(&graph);
    return 0;
}

int main() {
    create_filter_graph(1920, 1080, AV_PIX_FMT_YUV420P);
    return 0;
}
```

> **KrKr2 中的使用**：`VideoCodecFFmpeg.cpp` 使用 libavfilter 进行视频后处理。`m_pFilteredFrame` 成员存储滤镜处理后的帧。典型用途包括反交错（yadif 滤镜）和像素格式转换。

---

## 三、六大库的依赖关系

理解库之间的依赖关系对于正确链接至关重要：

```
┌────────────────────────────────────────────────┐
│              应用程序（KrKr2）                    │
├────────┬────────┬──────────┬───────────────────┤
│ libav  │ libav  │ libsw    │ libsw             │
│ format │ filter │ scale    │ resample          │
├────────┴────────┴──────────┴───────────────────┤
│                  libavcodec                      │
├──────────────────────────────────────────────────┤
│                  libavutil                        │
└──────────────────────────────────────────────────┘

依赖方向（从上到下）：
  libavformat  → libavcodec → libavutil
  libavfilter  → libavcodec → libavutil
  libswscale                → libavutil
  libswresample             → libavutil
```

**关键规则**：链接顺序必须从高层到底层。在 CMake 的 `target_link_libraries` 中，被依赖的库放在后面：

```cmake
# ✅ 正确的链接顺序
target_link_libraries(my_player PRIVATE
    avformat    # 最高层
    avfilter
    avcodec     # 中间层
    swscale
    swresample
    avutil      # 最底层
)

# ❌ 错误的顺序（静态链接时会导致符号未定义）
target_link_libraries(my_player PRIVATE
    avutil avcodec avformat  # 反了！
)
```

---

## 四、在 CMake 项目中使用 FFmpeg

### 4.1 通过 vcpkg 安装

KrKr2 使用 vcpkg 管理 FFmpeg 依赖。在 `vcpkg.json` 中：

```json
{
    "dependencies": [
        {
            "name": "ffmpeg",
            "features": ["avcodec", "avformat", "avfilter",
                         "swscale", "swresample"]
        }
    ]
}
```

### 4.2 CMake 中查找和链接

**方法一：使用 `pkg-config`（通用方式）：**

```cmake
cmake_minimum_required(VERSION 3.20)
project(ffmpeg_demo LANGUAGES CXX)

find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBAV REQUIRED IMPORTED_TARGET
    libavformat
    libavcodec
    libavutil
    libswscale
    libswresample
    libavfilter
)

add_executable(demo main.cpp)
target_link_libraries(demo PRIVATE PkgConfig::LIBAV)
```

**方法二：使用 vcpkg 的 FFmpeg CMake 集成：**

```cmake
cmake_minimum_required(VERSION 3.20)
project(ffmpeg_demo LANGUAGES CXX)

# vcpkg 会自动设置 CMAKE_TOOLCHAIN_FILE
find_package(PkgConfig REQUIRED)

# vcpkg 安装的 FFmpeg 同样通过 pkg-config 发现
pkg_check_modules(FFMPEG REQUIRED IMPORTED_TARGET
    libavformat
    libavcodec
    libavutil
    libswscale
    libswresample
)

add_executable(demo main.cpp)
target_link_libraries(demo PRIVATE PkgConfig::FFMPEG)
```

**方法三：手动查找（当 pkg-config 不可用时）：**

```cmake
cmake_minimum_required(VERSION 3.20)
project(ffmpeg_demo LANGUAGES CXX)

# 逐个查找 FFmpeg 库
find_path(AVFORMAT_INCLUDE_DIR libavformat/avformat.h)
find_library(AVFORMAT_LIBRARY avformat)
find_path(AVCODEC_INCLUDE_DIR libavcodec/avcodec.h)
find_library(AVCODEC_LIBRARY avcodec)
find_path(AVUTIL_INCLUDE_DIR libavutil/avutil.h)
find_library(AVUTIL_LIBRARY avutil)
find_library(SWSCALE_LIBRARY swscale)
find_library(SWRESAMPLE_LIBRARY swresample)

add_executable(demo main.cpp)
target_include_directories(demo PRIVATE
    ${AVFORMAT_INCLUDE_DIR}
    ${AVCODEC_INCLUDE_DIR}
    ${AVUTIL_INCLUDE_DIR}
)
target_link_libraries(demo PRIVATE
    ${AVFORMAT_LIBRARY}
    ${AVCODEC_LIBRARY}
    ${SWSCALE_LIBRARY}
    ${SWRESAMPLE_LIBRARY}
    ${AVUTIL_LIBRARY}
)
```

### 4.3 跨平台注意事项

| 平台 | 安装方式 | 注意事项 |
|------|---------|---------|
| **Windows** | vcpkg 或预编译二进制 | 需要同时部署 `.dll` 文件；静态链接需额外链接 `bcrypt`, `secur32`, `ws2_32` |
| **Linux** | vcpkg / apt / pacman | `apt install libavformat-dev libavcodec-dev ...`；注意版本（Ubuntu 22.04 默认 FFmpeg 4.x） |
| **macOS** | vcpkg / Homebrew | `brew install ffmpeg`；注意 Apple Silicon 的 Homebrew 路径 `/opt/homebrew/` |
| **Android** | vcpkg（交叉编译） | 使用 vcpkg 的 Android triplet；需要在 Gradle 中配置 CMake 路径 |

**Windows 静态链接的额外依赖：**

```cmake
if(WIN32)
    # FFmpeg 静态链接时需要这些 Windows 系统库
    target_link_libraries(demo PRIVATE
        bcrypt      # 加密
        secur32     # 安全
        ws2_32      # 网络
        Mfplat      # Media Foundation（某些硬件解码器需要）
        Mfuuid
        Strmiids    # DirectShow GUID
    )
endif()
```

**Android 的 CMake 配置：**

```cmake
if(ANDROID)
    # Android NDK 不自带 FFmpeg，通过 vcpkg 安装
    # vcpkg 的 Android triplet 会自动处理交叉编译
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(FFMPEG REQUIRED IMPORTED_TARGET
        libavformat libavcodec libavutil
        libswscale libswresample
    )
    target_link_libraries(demo PRIVATE
        PkgConfig::FFMPEG
        log         # Android 日志库
        android     # Android NDK 库
    )
endif()
```

---

## 五、常见错误与排查

### 错误 1：链接顺序导致 undefined reference

```
/usr/bin/ld: libavformat.a: undefined reference to `avcodec_parameters_copy'
```

**原因**：静态链接时，链接器从左到右处理库。如果 `avformat` 在 `avcodec` 后面，`avformat` 依赖的 `avcodec` 符号无法解析。

**解决**：确保链接顺序从高层到底层：
```cmake
target_link_libraries(demo PRIVATE avformat avcodec swscale swresample avutil)
```

### 错误 2：头文件路径不对

```
fatal error: libavformat/avformat.h: No such file or directory
```

**解决**：FFmpeg 头文件使用 `libavXXX/` 前缀，需要将 FFmpeg 的 include 目录添加到搜索路径：
```cmake
# 正确的 include 方式
#include <libavformat/avformat.h>   # ✅
#include <avformat.h>                # ❌ 缺少子目录前缀
```

### 错误 3：API 版本不匹配

```
error: 'avcodec_decode_video2' was not declared in this scope
```

**原因**：使用了旧 API，但 FFmpeg 版本 ≥ 5.0 已移除。

**解决**：使用新式 API：
```cpp
// 旧 → 新
avcodec_decode_video2()  →  avcodec_send_packet() + avcodec_receive_frame()
avcodec_decode_audio4()  →  avcodec_send_packet() + avcodec_receive_frame()
avpicture_fill()         →  av_image_fill_arrays()
avpicture_get_size()     →  av_image_get_buffer_size()
```

---

## 六、动手实践

### 实践 1：查看文件信息（类似 ffprobe）

创建项目结构：
```
ffprobe_mini/
├── CMakeLists.txt
└── main.cpp
```

**CMakeLists.txt：**
```cmake
cmake_minimum_required(VERSION 3.20)
project(ffprobe_mini LANGUAGES CXX)

find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBAV REQUIRED IMPORTED_TARGET
    libavformat
    libavcodec
    libavutil
)

add_executable(ffprobe_mini main.cpp)
target_link_libraries(ffprobe_mini PRIVATE PkgConfig::LIBAV)
```

**main.cpp：**
```cpp
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libavutil/avutil.h>
#include <cstdio>
#include <cstdlib>

void print_stream_info(AVFormatContext* fmt_ctx, int stream_idx) {
    AVStream* stream = fmt_ctx->streams[stream_idx];
    AVCodecParameters* par = stream->codecpar;

    const char* type = av_get_media_type_string(par->codec_type);
    const char* codec = avcodec_get_name(par->codec_id);

    printf("\n  --- 流 #%d ---\n", stream_idx);
    printf("  类型:     %s\n", type ? type : "unknown");
    printf("  编解码器: %s\n", codec);
    printf("  时间基:   %d/%d\n",
           stream->time_base.num, stream->time_base.den);

    if (stream->duration != AV_NOPTS_VALUE) {
        double duration_sec = stream->duration * av_q2d(stream->time_base);
        printf("  时长:     %.2f 秒\n", duration_sec);
    }

    if (par->codec_type == AVMEDIA_TYPE_VIDEO) {
        printf("  分辨率:   %dx%d\n", par->width, par->height);
        printf("  像素格式: %s\n",
               av_get_pix_fmt_name((AVPixelFormat)par->format));
        if (stream->avg_frame_rate.den != 0) {
            printf("  帧率:     %.2f fps\n",
                   av_q2d(stream->avg_frame_rate));
        }
        printf("  比特率:   %lld kbps\n", par->bit_rate / 1000);
    }

    if (par->codec_type == AVMEDIA_TYPE_AUDIO) {
        printf("  采样率:   %d Hz\n", par->sample_rate);
        printf("  通道数:   %d\n", par->ch_layout.nb_channels);
        printf("  采样格式: %s\n",
               av_get_sample_fmt_name((AVSampleFormat)par->format));
        printf("  比特率:   %lld kbps\n", par->bit_rate / 1000);
    }
}

int main(int argc, char* argv[]) {
    if (argc < 2) {
        fprintf(stderr, "用法: %s <媒体文件>\n", argv[0]);
        return 1;
    }

    const char* filename = argv[1];
    AVFormatContext* fmt_ctx = nullptr;

    // 打开文件
    int ret = avformat_open_input(&fmt_ctx, filename, nullptr, nullptr);
    if (ret < 0) {
        char errbuf[AV_ERROR_MAX_STRING_SIZE];
        av_strerror(ret, errbuf, sizeof(errbuf));
        fprintf(stderr, "无法打开 '%s': %s\n", filename, errbuf);
        return 1;
    }

    // 获取流信息
    if (avformat_find_stream_info(fmt_ctx, nullptr) < 0) {
        fprintf(stderr, "无法获取流信息\n");
        avformat_close_input(&fmt_ctx);
        return 1;
    }

    // 打印文件信息
    printf("========================================\n");
    printf("文件: %s\n", filename);
    printf("格式: %s (%s)\n",
           fmt_ctx->iformat->name,
           fmt_ctx->iformat->long_name);

    if (fmt_ctx->duration != AV_NOPTS_VALUE) {
        double total_sec = fmt_ctx->duration / (double)AV_TIME_BASE;
        int hours = (int)(total_sec / 3600);
        int minutes = ((int)(total_sec / 60)) % 60;
        int seconds = ((int)total_sec) % 60;
        printf("总时长: %02d:%02d:%02d\n", hours, minutes, seconds);
    }

    printf("比特率: %lld kbps\n", fmt_ctx->bit_rate / 1000);
    printf("流数量: %u\n", fmt_ctx->nb_streams);

    for (unsigned i = 0; i < fmt_ctx->nb_streams; i++) {
        print_stream_info(fmt_ctx, i);
    }

    // 打印 metadata
    printf("\n  --- Metadata ---\n");
    const AVDictionaryEntry* tag = nullptr;
    while ((tag = av_dict_iterate(fmt_ctx->metadata, tag))) {
        printf("  %s = %s\n", tag->key, tag->value);
    }

    printf("========================================\n");

    avformat_close_input(&fmt_ctx);
    return 0;
}
```

编译运行（四个平台）：

```bash
# Windows（vcpkg）
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%/scripts/buildsystems/vcpkg.cmake
cmake --build build --config Release
.\build\Release\ffprobe_mini.exe test.mp4

# Linux
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake
cmake --build build
./build/ffprobe_mini test.mp4

# macOS
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake
cmake --build build
./build/ffprobe_mini test.mp4

# Android（交叉编译后 adb push 到设备）
cmake -B build -S . \
    -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
    -DVCPKG_TARGET_TRIPLET=arm64-android \
    -DANDROID_ABI=arm64-v8a \
    -DANDROID_PLATFORM=android-24
cmake --build build
adb push build/ffprobe_mini /data/local/tmp/
adb shell /data/local/tmp/ffprobe_mini /sdcard/test.mp4
```

---

## 七、对照项目源码

KrKr2 项目中 FFmpeg 各库的使用分布：

| 源文件 | 使用的 FFmpeg 库 | 功能 |
|--------|-----------------|------|
| `cpp/core/movie/ffmpeg/DemuxFFmpeg.cpp` (1656行) | libavformat, libavutil | 解封装：打开媒体文件，读取 AVPacket |
| `cpp/core/movie/ffmpeg/VideoCodecFFmpeg.cpp` (1112行) | libavcodec, libswscale, libavfilter, libavutil | 视频解码 + 像素格式转换 + 滤镜 |
| `cpp/core/movie/ffmpeg/AudioCodecFFmpeg.cpp` (359行) | libavcodec, libswresample, libavutil | 音频解码 + 重采样 |
| `cpp/core/movie/ffmpeg/VideoPlayer.cpp` (2906行) | libavutil | 主循环：协调解封装、解码、渲染 |
| `cpp/core/movie/ffmpeg/Clock.cpp` (275行) | libavutil | 时钟管理：av_rescale_q 时间转换 |
| `cpp/core/sound/FFWaveDecoder.cpp` (451行) | libavformat, libavcodec, libswresample, libavutil | 独立的音频解码器（用于 BGM/SE） |

**关键发现**：
- 视频播放模块（`movie/ffmpeg/`）使用了全部 6 个库
- 音频解码模块（`sound/FFWaveDecoder.cpp`）使用了 4 个库（不需要 libswscale 和 libavfilter）
- 所有模块都依赖 libavutil（错误处理、时间转换、内存管理）

相关文件路径：
- `cpp/core/movie/ffmpeg/DemuxFFmpeg.h` 第 1-157 行 — 解封装器类定义
- `cpp/core/movie/ffmpeg/VideoCodecFFmpeg.h` 第 1-149 行 — 视频解码器类定义
- `cpp/core/movie/ffmpeg/AudioCodecFFmpeg.h` 第 1-72 行 — 音频解码器类定义

---

## 本节小结

- **FFmpeg 由 6 个核心库组成**，各有明确职责：
  - **libavutil**：基础工具（内存、错误码、时间基、像素/采样格式）
  - **libavformat**：容器格式处理（打开文件、读取流、拆分数据包）
  - **libavcodec**：编解码（将压缩数据 ↔ 原始数据）
  - **libswscale**：图像缩放和像素格式转换
  - **libswresample**：音频重采样和采样格式转换
  - **libavfilter**：滤镜处理（反交错、后处理等）
- **依赖关系**：avutil 在最底层，所有库都依赖它；avformat 和 avfilter 依赖 avcodec
- **KrKr2 使用新式 API**（`send_packet` / `receive_frame`），不使用已废弃的旧 API
- **CMake 链接顺序很重要**：高层库必须在前，底层库在后
- **KrKr2 使用自定义 I/O** 从 XP3 归档读取媒体数据，而非直接打开文件路径

---

## 练习题与答案

### 题目 1：FFmpeg 库的职责匹配

将以下操作与对应的 FFmpeg 库匹配：

| 操作 | 应使用哪个库？ |
|------|--------------|
| A. 将 H.264 压缩数据解码为 YUV 像素 | ? |
| B. 打开 MP4 文件并读取数据包 | ? |
| C. 将 YUV420P 转为 BGRA 格式 | ? |
| D. 将 44.1kHz 音频转为 48kHz | ? |
| E. 对视频帧进行反交错处理 | ? |
| F. 将 PTS 从一个时间基转换到另一个时间基 | ? |

<details>
<summary>查看答案</summary>

| 操作 | 库 | 说明 |
|------|-----|------|
| A. 将 H.264 压缩数据解码为 YUV 像素 | **libavcodec** | 编解码器库负责压缩↔原始数据的转换 |
| B. 打开 MP4 文件并读取数据包 | **libavformat** | 封装格式库负责容器解析和数据包读取 |
| C. 将 YUV420P 转为 BGRA 格式 | **libswscale** | 图像缩放和像素格式转换库 |
| D. 将 44.1kHz 音频转为 48kHz | **libswresample** | 音频重采样库 |
| E. 对视频帧进行反交错处理 | **libavfilter** | 滤镜框架库（yadif 滤镜） |
| F. 将 PTS 从一个时间基转换到另一个时间基 | **libavutil** | 基础工具库（av_rescale_q 函数） |

</details>

### 题目 2：修复错误的代码

以下代码有 3 处错误，找出并修正：

```cpp
#include <libavformat/avformat.h>

int main() {
    AVFormatContext* fmt_ctx;  // 错误 1
    avformat_open_input(&fmt_ctx, "test.mp4", NULL, NULL);
    avformat_find_stream_info(fmt_ctx, NULL);

    AVPacket pkt;  // 错误 2
    while (av_read_frame(fmt_ctx, &pkt) >= 0) {
        printf("stream=%d size=%d\n", pkt.stream_index, pkt.size);
        // 错误 3：缺少什么？
    }

    avformat_close_input(&fmt_ctx);
    return 0;
}
```

<details>
<summary>查看答案</summary>

**错误 1**：`fmt_ctx` 未初始化为 `nullptr`。`avformat_open_input` 接受 `AVFormatContext**`，如果 `*fmt_ctx` 不为 `nullptr`，它会尝试使用已有的上下文而非分配新的。

```cpp
AVFormatContext* fmt_ctx = nullptr;  // ✅ 必须初始化为 nullptr
```

**错误 2**：`AVPacket` 不应该在栈上直接声明（现代 FFmpeg 推荐使用 `av_packet_alloc`）。栈上的 `AVPacket` 需要手动用 `av_init_packet` 初始化，而且在新版 FFmpeg 中 `av_init_packet` 已被废弃。

```cpp
AVPacket* pkt = av_packet_alloc();  // ✅ 使用堆分配
// ... 循环使用 pkt 而非 &pkt
av_packet_free(&pkt);  // 最后释放
```

**错误 3**：循环内缺少 `av_packet_unref(pkt)` 调用。每次 `av_read_frame` 成功后，`pkt` 内部会分配数据缓冲区，如果不释放就会持续泄漏内存。

```cpp
while (av_read_frame(fmt_ctx, pkt) >= 0) {
    printf("stream=%d size=%d\n", pkt->stream_index, pkt->size);
    av_packet_unref(pkt);  // ✅ 释放包数据
}
```

完整修正代码：

```cpp
#include <libavformat/avformat.h>
#include <cstdio>

int main() {
    AVFormatContext* fmt_ctx = nullptr;
    int ret = avformat_open_input(&fmt_ctx, "test.mp4", nullptr, nullptr);
    if (ret < 0) {
        char errbuf[AV_ERROR_MAX_STRING_SIZE];
        av_strerror(ret, errbuf, sizeof(errbuf));
        fprintf(stderr, "打开失败: %s\n", errbuf);
        return 1;
    }

    if (avformat_find_stream_info(fmt_ctx, nullptr) < 0) {
        fprintf(stderr, "获取流信息失败\n");
        avformat_close_input(&fmt_ctx);
        return 1;
    }

    AVPacket* pkt = av_packet_alloc();
    while (av_read_frame(fmt_ctx, pkt) >= 0) {
        printf("stream=%d size=%d\n", pkt->stream_index, pkt->size);
        av_packet_unref(pkt);
    }

    av_packet_free(&pkt);
    avformat_close_input(&fmt_ctx);
    return 0;
}
```

</details>

### 题目 3：设计题 —— 自定义 I/O

KrKr2 的媒体文件存储在 XP3 归档包中，无法用文件路径直接打开。请回答：

1. FFmpeg 提供了什么机制来支持自定义数据源？
2. 需要实现哪些回调函数？
3. 使用哪个 API 创建自定义 I/O 上下文？

<details>
<summary>查看答案</summary>

**1. 自定义 I/O 机制**

FFmpeg 通过 `AVIOContext` 支持自定义 I/O。用户可以注册回调函数，FFmpeg 调用这些回调来读取/写入数据，而不是直接操作文件系统。

**2. 需要实现的回调函数**

```cpp
// 读取回调：从自定义数据源读取 size 字节到 buf
int read_callback(void* opaque, uint8_t* buf, int buf_size);

// Seek 回调：在数据源中定位
int64_t seek_callback(void* opaque, int64_t offset, int whence);
// whence 可以是 SEEK_SET、SEEK_CUR、SEEK_END 或 AVSEEK_SIZE（返回总大小）
```

- `opaque`：用户自定义指针，通常指向你的数据源对象（如 IStream*）
- 读取回调必须实现，Seek 回调可选但强烈建议实现（否则无法 seek）

**3. 创建自定义 I/O 上下文的 API**

```cpp
// 分配 I/O 缓冲区
int buffer_size = 32768;  // 32KB
uint8_t* buffer = (uint8_t*)av_malloc(buffer_size);

// 创建自定义 AVIOContext
AVIOContext* avio_ctx = avio_alloc_context(
    buffer,          // I/O 缓冲区
    buffer_size,     // 缓冲区大小
    0,               // 0 = 只读，1 = 可写
    my_stream,       // opaque 指针（传给回调的 void* 参数）
    read_callback,   // 读取回调
    nullptr,         // 写入回调（只读时为 nullptr）
    seek_callback);  // Seek 回调

// 将自定义 I/O 设置到格式上下文
AVFormatContext* fmt_ctx = avformat_alloc_context();
fmt_ctx->pb = avio_ctx;

// 然后正常调用 avformat_open_input，但 URL 参数可以传 nullptr
avformat_open_input(&fmt_ctx, nullptr, nullptr, nullptr);
```

KrKr2 的 `DemuxFFmpeg.cpp` 正是这样实现的，`opaque` 指向 COM 的 `IStream` 接口，回调函数通过 `IStream::Read()` 和 `IStream::Seek()` 读取 XP3 归档中的数据。

</details>

---

## 下一步

下一节 [03-解封装从容器到数据包](./03-解封装从容器到数据包.md) 将深入讲解 libavformat 的解封装流程，包括：
- `avformat_open_input` 的内部工作原理
- 自定义 I/O 回调的完整实现
- KrKr2 `CDVDDemuxFFmpeg` 类的源码逐行解析
- 流选择和多轨道处理
