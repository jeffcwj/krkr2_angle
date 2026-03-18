# 2.1 CDVDDemuxFFmpeg 解封装器

## 本节目标

深入分析 CDVDDemuxFFmpeg 解封装器的实现。理解它如何将多媒体容器文件（MPEG、AVI、MKV 等）拆分为独立的音视频数据流，掌握 IStream COM 到 FFmpeg AVIO 的适配机制、容器格式探测、流信息获取和数据包读取的完整流程。

---

## 1. 解封装器概述

解封装（Demuxing）是多媒体播放的第一步——将容器文件中交错存储的音频和视频数据分离为独立的流。容器格式（如 MPEG-PS、AVI、MKV）将音频帧和视频帧按时间顺序交错排列，解封装器的任务是读取这些交错数据，按流类型分类输出。

CDVDDemuxFFmpeg 是对 FFmpeg `libavformat` 库的封装。FFmpeg 是世界上最全面的开源多媒体框架，其 `libavformat` 支持超过 300 种容器格式的解封装。KrKr2 通过 CDVDDemuxFFmpeg 利用 FFmpeg 的容器解析能力，同时隐藏 FFmpeg 的复杂 API 细节。

解封装器在播放流水线中的位置是"数据入口"——所有后续的解码、同步和渲染都依赖于解封装器输出的数据包。因此解封装器的正确性和效率直接影响整个播放链路。

### 1.1 解封装数据流

```
┌──────────────────────────────────────────────────────────┐
│                    容器文件（如 MPEG）                     │
│  ┌─V─┐┌─A─┐┌─V─┐┌─V─┐┌─A─┐┌─V─┐┌─A─┐┌─V─┐           │
│  │ 1 ││ 1 ││ 2 ││ 3 ││ 2 ││ 4 ││ 3 ││ 5 │ ...        │
│  └───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘           │
└────────────────────────┬─────────────────────────────────┘
                         │ CDVDDemuxFFmpeg::Read()
                         │ av_read_frame()
                         ▼
┌────────────────────────────────────────────────────────┐
│                    CDVDDemuxFFmpeg                       │
│                                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │ AVFormatContext (FFmpeg 容器解析上下文)           │   │
│  │  • 容器格式探测                                  │   │
│  │  • 流信息解析                                    │   │
│  │  • 数据包读取和时间戳转换                         │   │
│  └──────────────────────┬──────────────────────────┘   │
│                         │                              │
│  ┌──────────────────────▼──────────────────────────┐   │
│  │         ConvertTimestamp()                        │   │
│  │  AVStream 时基 → DVD_TIME_BASE (微秒)             │   │
│  └──────────┬────────────────────────┬─────────────┘   │
│             │                        │                  │
│   ┌─────────▼────────┐    ┌─────────▼────────┐        │
│   │   视频 DemuxPacket │    │   音频 DemuxPacket │        │
│   │   streamId = 0    │    │   streamId = 1    │        │
│   └──────────────────┘    └──────────────────┘        │
└────────────────────────────────────────────────────────┘
```

---

## 2. IStream 到 AVIO 适配

KrKr2 的文件系统使用 Windows COM 的 IStream 接口。FFmpeg 的 `libavformat` 使用自己的 AVIO（AVIOContext）接口进行 I/O 操作。InputStream 类负责在这两套接口之间架起桥梁，使 FFmpeg 能够透明地读取 KrKr2 管理的文件数据。

### 2.1 InputStream 适配器

```cpp
// 源码: movie/ffmpeg/InputStream.cpp
// IStream COM → FFmpeg AVIO 适配器

NS_KRMOVIE_BEGIN

class InputStream {
public:
    InputStream(IStream* stream, uint64_t size)
        : m_stream(stream), m_size(size) {
        // IStream 是 KrKr2 提供的文件读取接口
        // 它可能指向磁盘文件、压缩包中的文件、
        // 或内存中的数据
        m_stream->AddRef();  // 增加 COM 引用计数
    }
    
    ~InputStream() {
        m_stream->Release();  // 释放 COM 引用
    }
    
    // 读取数据（FFmpeg AVIO 回调）
    int Read(uint8_t* buf, int size) {
        ULONG bytesRead = 0;
        HRESULT hr = m_stream->Read(buf, size, &bytesRead);
        if (FAILED(hr))
            return -1;  // 读取失败
        return bytesRead;
    }
    
    // 跳转位置（FFmpeg AVIO 回调）
    int64_t Seek(int64_t offset, int whence) {
        LARGE_INTEGER li;
        ULARGE_INTEGER newPos;
        
        switch (whence) {
        case SEEK_SET:
            li.QuadPart = offset;
            break;
        case SEEK_CUR:
            li.QuadPart = offset;
            break;
        case SEEK_END:
            li.QuadPart = offset;
            break;
        case AVSEEK_SIZE:
            // FFmpeg 查询文件总大小
            return m_size;
        default:
            return -1;
        }
        
        HRESULT hr = m_stream->Seek(li, whence, &newPos);
        if (FAILED(hr))
            return -1;
        return newPos.QuadPart;
    }

private:
    IStream* m_stream;   // COM 流接口
    uint64_t m_size;     // 文件总大小
};

NS_KRMOVIE_END
```

### 2.2 AVIO 回调注册

```cpp
// 源码: movie/ffmpeg/DemuxFFmpeg.cpp
// 将 InputStream 的方法注册为 FFmpeg 的 I/O 回调

// C 风格的回调函数（FFmpeg 需要 C 函数指针）
static int dvd_file_read(void* opaque, uint8_t* buf, int size) {
    // opaque 指向 InputStream 实例
    InputStream* input = static_cast<InputStream*>(opaque);
    return input->Read(buf, size);
}

static int64_t dvd_file_seek(void* opaque, 
                              int64_t offset, int whence) {
    InputStream* input = static_cast<InputStream*>(opaque);
    return input->Seek(offset, whence);
}

// 在 CDVDDemuxFFmpeg::Open() 中创建 AVIO 上下文
bool CDVDDemuxFFmpeg::Open(InputStream* pInput) {
    m_pInput = pInput;
    
    // 分配 AVIO 缓冲区（32KB）
    const int bufferSize = 32768;
    uint8_t* buffer = (uint8_t*)av_malloc(bufferSize);
    
    // 创建 AVIOContext
    // 将 dvd_file_read/seek 注册为 I/O 回调
    m_ioContext = avio_alloc_context(
        buffer,          // I/O 缓冲区
        bufferSize,      // 缓冲区大小
        0,               // 只读模式
        m_pInput,        // opaque 指针（传给回调函数）
        dvd_file_read,   // 读取回调
        nullptr,         // 写入回调（不需要）
        dvd_file_seek    // Seek 回调
    );
    
    // ... 继续打开容器 ...
}
```

> **跨平台说明**：
> - **Windows**：IStream 是原生 COM 接口，无需适配
> - **Linux/macOS**：KrKr2 提供了 IStream 的兼容实现（基于 `FILE*` 或 `fstream`）
> - **Android**：IStream 通过 JNI 桥接到 Java 的 `InputStream`/`AssetManager`
> - 所有平台上 FFmpeg 的 AVIO 回调接口是相同的，差异被 InputStream 抽象层屏蔽

---

## 3. 容器打开流程

### 3.1 Open() 方法详解

```cpp
// 源码: movie/ffmpeg/DemuxFFmpeg.cpp
// CDVDDemuxFFmpeg::Open() — 打开容器的完整流程

bool CDVDDemuxFFmpeg::Open(InputStream* pInput) {
    m_pInput = pInput;
    
    // ── 步骤 1: 创建 AVIO 上下文 ──
    // （见 2.2 节代码）
    SetupAVIO();
    
    // ── 步骤 2: 分配 AVFormatContext ──
    m_pFormatContext = avformat_alloc_context();
    m_pFormatContext->pb = m_ioContext;  // 关联自定义 I/O
    
    // 禁止 FFmpeg 自己打开文件（我们已提供 I/O 回调）
    m_pFormatContext->flags |= AVFMT_FLAG_CUSTOM_IO;
    
    // ── 步骤 3: 探测容器格式并打开 ──
    // avformat_open_input 会：
    //   1. 读取文件头（通过 dvd_file_read 回调）
    //   2. 探测容器格式（MPEG-PS? AVI? MKV?）
    //   3. 解析容器头信息
    AVInputFormat* inputFormat = nullptr;
    int ret = avformat_open_input(
        &m_pFormatContext,
        "",              // 文件名（已通过 AVIO 打开，传空）
        inputFormat,     // 容器格式（nullptr = 自动探测）
        nullptr          // 格式选项
    );
    
    if (ret < 0) {
        char errbuf[256];
        av_strerror(ret, errbuf, sizeof(errbuf));
        // 常见错误: "Invalid data found when processing input"
        return false;
    }
    
    // ── 步骤 4: 获取流信息 ──
    // avformat_find_stream_info 会：
    //   1. 读取若干帧数据
    //   2. 分析每个流的编解码参数
    //   3. 计算流的时长、帧率等信息
    ret = avformat_find_stream_info(
        m_pFormatContext, 
        nullptr
    );
    
    if (ret < 0) {
        return false;
    }
    
    // ── 步骤 5: 创建流描述对象 ──
    CreateStreams();
    
    return true;
}
```

### 3.2 流创建

```cpp
// 源码: movie/ffmpeg/DemuxFFmpeg.cpp
// CreateStreams() — 为每个流创建描述对象

void CDVDDemuxFFmpeg::CreateStreams() {
    for (unsigned int i = 0; i < m_pFormatContext->nb_streams; i++) {
        AVStream* avStream = m_pFormatContext->streams[i];
        AVCodecParameters* codecpar = avStream->codecpar;
        
        CDemuxStream* stream = nullptr;
        
        switch (codecpar->codec_type) {
        case AVMEDIA_TYPE_VIDEO: {
            CDemuxStreamVideo* videoStream = 
                new CDemuxStreamVideo();
            videoStream->iWidth = codecpar->width;
            videoStream->iHeight = codecpar->height;
            videoStream->fFps = 
                av_q2d(avStream->avg_frame_rate);
            videoStream->codec = codecpar->codec_id;
            stream = videoStream;
            break;
        }
        
        case AVMEDIA_TYPE_AUDIO: {
            CDemuxStreamAudio* audioStream = 
                new CDemuxStreamAudio();
            audioStream->iChannels = codecpar->channels;
            audioStream->iSampleRate = codecpar->sample_rate;
            audioStream->iBitsPerSample = 
                codecpar->bits_per_coded_sample;
            audioStream->codec = codecpar->codec_id;
            stream = audioStream;
            break;
        }
        
        default:
            // 字幕、数据等其他流类型 → 跳过
            continue;
        }
        
        // 设置通用属性
        stream->uniqueId = i;
        stream->type = (codecpar->codec_type == AVMEDIA_TYPE_VIDEO)
            ? STREAM_VIDEO : STREAM_AUDIO;
        
        m_streams.push_back(stream);
    }
}
```

---

## 4. 数据包读取

### 4.1 Read() 方法

```cpp
// 源码: movie/ffmpeg/DemuxFFmpeg.cpp
// CDVDDemuxFFmpeg::Read() — 读取一个解封装数据包

DemuxPacket* CDVDDemuxFFmpeg::Read() {
    AVPacket avpkt;
    av_init_packet(&avpkt);
    avpkt.data = nullptr;
    avpkt.size = 0;
    
    // 从容器中读取下一个数据包
    // av_read_frame 返回的是一个完整的帧（可能跨越多个容器块）
    int ret = av_read_frame(m_pFormatContext, &avpkt);
    
    if (ret < 0) {
        if (ret == AVERROR_EOF || 
            avio_feof(m_pFormatContext->pb)) {
            // 文件结束
            return nullptr;
        }
        // 读取错误
        return nullptr;
    }
    
    // 检查流索引是否有效
    if (avpkt.stream_index < 0 || 
        avpkt.stream_index >= (int)m_pFormatContext->nb_streams) {
        av_packet_unref(&avpkt);
        return nullptr;
    }
    
    AVStream* stream = m_pFormatContext->streams[avpkt.stream_index];
    
    // 创建 DemuxPacket（KrKr2 内部数据包格式）
    DemuxPacket* pPacket = CDVDDemuxUtils::AllocateDemuxPacket(
        avpkt.size);
    
    // 复制数据
    memcpy(pPacket->pData, avpkt.data, avpkt.size);
    pPacket->iSize = avpkt.size;
    pPacket->iStreamId = avpkt.stream_index;
    
    // ── 时间戳转换 ──
    // 将 AVStream 的时基转换为内部时基（DVD_TIME_BASE = 微秒）
    pPacket->pts = ConvertTimestamp(
        avpkt.pts, 
        stream->time_base.den, 
        stream->time_base.num
    );
    pPacket->dts = ConvertTimestamp(
        avpkt.dts,
        stream->time_base.den,
        stream->time_base.num
    );
    pPacket->duration = ConvertTimestamp(
        avpkt.duration,
        stream->time_base.den,
        stream->time_base.num
    );
    
    // 释放 FFmpeg 数据包
    av_packet_unref(&avpkt);
    
    return pPacket;
}
```

### 4.2 时间戳转换

```cpp
// 源码: movie/ffmpeg/DemuxFFmpeg.cpp  
// ConvertTimestamp() — AVStream 时基到内部时基的转换

// FFmpeg 中每个流有自己的时基（time_base），例如：
// 视频流: time_base = 1/90000 (90kHz，MPEG 标准)
// 音频流: time_base = 1/44100 (采样率)
// 
// KrKr2 内部统一使用 DVD_TIME_BASE = 1000000 (微秒)

double CDVDDemuxFFmpeg::ConvertTimestamp(
    int64_t pts, int den, int num) {
    
    if (pts == AV_NOPTS_VALUE)
        return DVD_NOPTS_VALUE;  // 无效时间戳保持无效
    
    // 转换公式：
    // result = (pts * num / den - starttime) * DVD_TIME_BASE
    //
    // 例如：视频流 time_base = {1, 90000}
    //   pts = 450000 (90kHz 时基)
    //   num = 1, den = 90000
    //   result = (450000 * 1 / 90000) * 1000000
    //          = 5.0 * 1000000 = 5000000 微秒 = 5 秒
    
    double timestamp = (double)pts * num / den;
    
    // 减去起始时间偏移
    if (m_pFormatContext->start_time != AV_NOPTS_VALUE) {
        timestamp -= (double)m_pFormatContext->start_time / 
                     AV_TIME_BASE;
    }
    
    // 转换为 DVD_TIME_BASE（微秒）
    return timestamp * DVD_TIME_BASE;
}
```

---

## 5. Seek 操作

### 5.1 SeekTime() 方法

```cpp
// 源码: movie/ffmpeg/DemuxFFmpeg.cpp
// CDVDDemuxFFmpeg::SeekTime() — 跳转到指定时间

bool CDVDDemuxFFmpeg::SeekTime(int time_ms) {
    // 将毫秒转换为 FFmpeg 的 AV_TIME_BASE（微秒）
    int64_t seekTarget = (int64_t)time_ms * 1000;
    
    // 加上起始时间偏移
    if (m_pFormatContext->start_time != AV_NOPTS_VALUE)
        seekTarget += m_pFormatContext->start_time;
    
    // 执行 Seek
    // AVSEEK_FLAG_BACKWARD: 向后 Seek 到最近的关键帧
    // 这确保解码器从关键帧开始，能正确重建完整画面
    int ret = av_seek_frame(
        m_pFormatContext,
        -1,              // 流索引（-1 = 自动选择）
        seekTarget,      // 目标位置
        AVSEEK_FLAG_BACKWARD  // 向后到最近关键帧
    );
    
    if (ret < 0) {
        // Seek 失败（某些格式不支持 Seek）
        return false;
    }
    
    // 刷新 FFmpeg 内部缓冲区
    // 清除解封装器已缓存但尚未返回的数据包
    avformat_flush(m_pFormatContext);
    
    return true;
}
```

### 5.2 关键帧与 Seek 精度

```
关键帧（I 帧）──→ P 帧 ──→ P 帧 ──→ P 帧 ──→ 关键帧
  0ms            33ms      67ms      100ms      133ms

Seek 目标 = 60ms
实际 Seek 到 = 0ms (最近的前一个关键帧)

原因：P 帧依赖前面的帧才能解码，
      必须从关键帧开始解码才能得到正确画面。
      
这就是为什么 Seek 后可能有短暂的"解码追赶"：
  - FFmpeg Seek 到 0ms 的关键帧
  - 解码器解码 0ms → 33ms → 67ms
  - 只有 67ms 帧及之后的帧才会显示
  - 0ms 和 33ms 的帧被丢弃（它们在目标时间之前）
```

---

## 动手实践

### 练习 1：分析 AVIO 缓冲区大小的影响

```cpp
// 修改缓冲区大小，观察对性能的影响
const int bufferSize = 32768;  // 默认 32KB

// 尝试不同的值：
// 4096  (4KB)  — 更多 I/O 调用，但内存占用低
// 32768 (32KB) — 平衡的选择（默认值）  
// 262144 (256KB) — 更少 I/O 调用，适合大文件

// 使用 profiler 测量不同缓冲区大小下的：
// 1. av_read_frame() 平均耗时
// 2. dvd_file_read() 调用次数
// 3. 内存使用峰值
```

### 练习 2：实现一个简易解封装器

```cpp
// 使用 FFmpeg API 实现最小化的解封装器
// 不依赖 KrKr2 的任何组件

#include <libavformat/avformat.h>
#include <cstdio>

int main(int argc, char* argv[]) {
    const char* filename = argv[1];
    
    // 1. 打开容器
    AVFormatContext* fmt_ctx = nullptr;
    int ret = avformat_open_input(&fmt_ctx, filename, 
                                   nullptr, nullptr);
    if (ret < 0) {
        fprintf(stderr, "Cannot open: %s\n", filename);
        return 1;
    }
    
    // 2. 获取流信息
    avformat_find_stream_info(fmt_ctx, nullptr);
    
    // 3. 打印流信息
    printf("Format: %s\n", fmt_ctx->iformat->name);
    printf("Duration: %.2f seconds\n", 
           fmt_ctx->duration / (double)AV_TIME_BASE);
    
    for (unsigned i = 0; i < fmt_ctx->nb_streams; i++) {
        AVStream* s = fmt_ctx->streams[i];
        printf("Stream %d: %s, codec=%s\n", 
               i,
               av_get_media_type_string(
                   s->codecpar->codec_type),
               avcodec_get_name(s->codecpar->codec_id));
    }
    
    // 4. 读取前 10 个数据包
    AVPacket pkt;
    for (int n = 0; n < 10; n++) {
        ret = av_read_frame(fmt_ctx, &pkt);
        if (ret < 0) break;
        
        AVStream* s = fmt_ctx->streams[pkt.stream_index];
        double pts_sec = pkt.pts * 
            av_q2d(s->time_base);
        
        printf("Packet %d: stream=%d, size=%d, "
               "pts=%.3fs, key=%d\n",
               n, pkt.stream_index, pkt.size,
               pts_sec, (pkt.flags & AV_PKT_FLAG_KEY));
        
        av_packet_unref(&pkt);
    }
    
    // 5. 清理
    avformat_close_input(&fmt_ctx);
    return 0;
}

// 编译命令（Linux/macOS）:
// gcc -o demux_test demux_test.c \
//     $(pkg-config --cflags --libs libavformat libavcodec libavutil)
//
// Windows (vcpkg):
// cl demux_test.c /I vcpkg/installed/x64-windows/include \
//    /link vcpkg/installed/x64-windows/lib/avformat.lib \
//          vcpkg/installed/x64-windows/lib/avcodec.lib \
//          vcpkg/installed/x64-windows/lib/avutil.lib
```

---

## 对照项目源码

| 本节概念 | 对应源文件 | 关键代码位置 |
|----------|-----------|-------------|
| CDVDDemuxFFmpeg 类 | `DemuxFFmpeg.h` | `class CDVDDemuxFFmpeg` |
| Open() 方法 | `DemuxFFmpeg.cpp` | `CDVDDemuxFFmpeg::Open()` |
| Read() 方法 | `DemuxFFmpeg.cpp` | `CDVDDemuxFFmpeg::Read()` |
| ConvertTimestamp | `DemuxFFmpeg.cpp` | `CDVDDemuxFFmpeg::ConvertTimestamp()` |
| SeekTime | `DemuxFFmpeg.cpp` | `CDVDDemuxFFmpeg::SeekTime()` |
| CreateStreams | `DemuxFFmpeg.cpp` | `CDVDDemuxFFmpeg::CreateStreams()` |
| InputStream 适配器 | `InputStream.cpp` | `class InputStream` |
| AVIO 回调 | `DemuxFFmpeg.cpp` | `dvd_file_read()`, `dvd_file_seek()` |

---

## 本节小结

1. CDVDDemuxFFmpeg 封装了 FFmpeg 的 `libavformat`，负责将容器文件拆分为音视频数据流
2. InputStream 类将 KrKr2 的 IStream COM 接口适配为 FFmpeg 的 AVIO 回调接口
3. 打开流程：AVIO 创建 → avformat_open_input（格式探测）→ avformat_find_stream_info → CreateStreams
4. Read() 通过 av_read_frame 读取一个完整数据包，ConvertTimestamp 转换时基为 DVD_TIME_BASE
5. Seek 通过 av_seek_frame 实现，总是向后 Seek 到最近的关键帧以确保正确解码

---

## 常见错误及解决方案

### 错误 1：avformat_open_input 返回 -5 (I/O error)

```
avformat_open_input failed: Input/output error
```

**原因**：AVIO 回调函数返回了错误值，通常是 IStream::Read() 失败。

**解决方案**：
1. 检查 IStream 对象是否有效（是否已被释放？）
2. 验证文件是否存在且可读
3. 在 `dvd_file_read` 中添加日志，检查每次读取的返回值

### 错误 2：avformat_find_stream_info 卡住很久

```
avformat_find_stream_info takes > 5 seconds
```

**原因**：某些容器格式需要读取大量数据才能确定流信息（如缺少索引的 MPEG-TS）。

**解决方案**：
1. 设置分析时长限制：`m_pFormatContext->max_analyze_duration = 5 * AV_TIME_BASE`
2. 设置探测大小限制：`m_pFormatContext->probesize = 5000000`（5MB）

### 错误 3：时间戳不连续

```
Warning: pts jumped from 5.0s to 15.0s (gap = 10s)
```

**原因**：容器文件中存在时间戳不连续（常见于录制中断后重新开始的文件）。

**解决方案**：
1. 在 ConvertTimestamp 中检测大的 PTS 跳变
2. 当检测到不连续时，触发 CDVDClock::Discontinuity() 重置时钟

---

## 练习题与答案

### 练习 1：解释为什么需要 AVSEEK_FLAG_BACKWARD 标志

<details>
<summary>查看答案</summary>

`AVSEEK_FLAG_BACKWARD` 告诉 FFmpeg 将 Seek 位置调整到目标时间之前最近的关键帧。这是必要的，因为：

1. **帧间依赖**：视频压缩中，P 帧和 B 帧依赖前面的关键帧（I 帧）才能正确解码
2. **解码完整性**：如果 Seek 到一个 P 帧，解码器没有参考帧，会输出花屏
3. **向后保证**：即使目标位置恰好是关键帧，BACKWARD 标志也不会有副作用
4. **解码追赶**：解码器从关键帧开始解码，跳过目标时间之前的帧，直到到达目标位置

</details>

### 练习 2：分析 ConvertTimestamp 中为什么要减去 start_time

<details>
<summary>查看答案</summary>

某些容器格式的时间戳不从 0 开始。例如：

1. **MPEG-TS 流**：时间戳可能从一个很大的值开始（如 126000000），因为它代表广播时的实时时间
2. **录制文件**：从直播流录制的文件可能保留了原始的播放时间戳
3. **编辑后的文件**：多个片段拼接后，时间戳可能有偏移

减去 `start_time` 的作用是将文件的时间轴归零，使播放器的 0 秒对应文件的实际开始位置。

如果不减去 `start_time`，播放器会认为文件从一个很大的时间点开始，导致：
- 进度条显示异常
- Seek 计算错误
- 音视频同步基准偏移

</details>

### 练习 3：如果 dvd_file_read 返回 0 但不是文件结束，会发生什么？

<details>
<summary>查看答案</summary>

在 FFmpeg 的 AVIO 协议中，返回 0 表示文件结束（EOF）。如果 `dvd_file_read` 在文件中间错误地返回 0：

1. FFmpeg 会认为文件已经读完
2. `av_read_frame` 会返回 `AVERROR_EOF`
3. CDVDDemuxFFmpeg::Read() 返回 nullptr
4. BasePlayer 触发 EOF 处理逻辑
5. 播放提前结束，用户看到视频突然停止

**正确做法**：`dvd_file_read` 应该在没有数据可读但文件未结束时返回 `AVERROR(EAGAIN)`，表示暂时无数据可用。但在 KrKr2 中，由于都是本地文件读取，Read() 不应该返回 0 除非真的到达文件末尾。

</details>

---

## 下一步

下一节 [2.2 CDVDStreamInfo 与流管理](02-CDVDStreamInfo与流管理.md) 将分析如何管理容器中的多个流，包括流选择策略、编解码参数传递和动态流切换机制。
