# 2.2 CDVDStreamInfo 与流管理

## 本节目标

理解 KrKr2 视频播放器中的流管理机制。掌握 CDVDStreamInfo 如何传递编解码参数、CDemuxStream 层次结构如何描述音视频流，以及 BasePlayer 如何选择和切换活跃流。

---

## 1. 流管理概述

一个多媒体容器文件通常包含多个流：至少一个视频流和一个音频流，可能还有字幕流、章节信息等附加数据。流管理的职责是：识别所有可用的流、选择默认播放的流、传递流的编解码参数给解码器。

在 KrKr2 的视频播放器中，流管理涉及三个关键类：CDemuxStream（解封装层的流描述）、CDVDStreamInfo（编解码参数的传输载体）和 CCurrentStream（播放层的流状态跟踪）。这三个类分别工作在不同的层次，通过参数传递形成完整的流信息链。

流管理是连接解封装器和解码器的桥梁——解封装器知道容器中有哪些流及其格式参数，解码器需要这些参数来正确初始化。CDVDStreamInfo 就是这个桥梁的具体实现。

### 1.1 流信息传递链

```
┌──────────────────────┐
│   AVStream (FFmpeg)   │  FFmpeg 内部的流描述
│   AVCodecParameters   │  包含编解码器 ID、分辨率等
└──────────┬───────────┘
           │ CreateStreams()
           ▼
┌──────────────────────┐
│   CDemuxStream        │  KrKr2 的流描述
│   CDemuxStreamVideo   │  从 AVStream 提取关键参数
│   CDemuxStreamAudio   │  供 BasePlayer 选择流
└──────────┬───────────┘
           │ OpenStream()
           ▼
┌──────────────────────┐
│   CDVDStreamInfo      │  编解码参数传输载体
│                       │  传递给解码器初始化
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   CDVDVideoCodecFFmpeg│  解码器使用这些参数
│   CDVDAudioCodecFFmpeg│  初始化 AVCodecContext
└──────────────────────┘
```

---

## 2. CDemuxStream 层次结构

### 2.1 基类定义

```cpp
// 源码: movie/ffmpeg/DemuxFFmpeg.h (概念性定义)
// CDemuxStream 是解封装器输出的流描述基类

class CDemuxStream {
public:
    int uniqueId;              // 流的唯一标识
    StreamType type;           // 流类型 (VIDEO/AUDIO/SUBTITLE)
    AVCodecID codec;           // FFmpeg 编解码器 ID
    std::string codecName;     // 编解码器名称
    std::string language;      // 语言标识 (如 "eng", "jpn")
    
    // 获取流信息的字符串表示
    virtual std::string GetStreamInfo() const;
};

// 流类型枚举
enum StreamType {
    STREAM_NONE,
    STREAM_VIDEO,
    STREAM_AUDIO,
    STREAM_SUBTITLE,
    STREAM_DATA
};
```

### 2.2 视频流描述

```cpp
// CDemuxStreamVideo — 视频流特有属性

class CDemuxStreamVideo : public CDemuxStream {
public:
    int iWidth;                // 视频宽度（像素）
    int iHeight;               // 视频高度（像素）
    float fFps;                // 帧率（帧/秒）
    int iOrientation;          // 旋转角度 (0/90/180/270)
    float fAspect;             // 宽高比 (如 16.0/9.0)
    bool bVFR;                 // 是否为可变帧率
    
    // 从 AVStream 提取参数
    void SetFromAVStream(AVStream* avStream) {
        AVCodecParameters* par = avStream->codecpar;
        iWidth = par->width;
        iHeight = par->height;
        codec = par->codec_id;
        
        // 计算帧率
        if (avStream->avg_frame_rate.den > 0)
            fFps = av_q2d(avStream->avg_frame_rate);
        else if (avStream->r_frame_rate.den > 0)
            fFps = av_q2d(avStream->r_frame_rate);
        else
            fFps = 25.0f;  // 默认 25fps
        
        // 计算宽高比
        if (par->sample_aspect_ratio.num > 0)
            fAspect = (float)iWidth * 
                par->sample_aspect_ratio.num /
                (iHeight * par->sample_aspect_ratio.den);
        else
            fAspect = (float)iWidth / iHeight;
    }
};
```

### 2.3 音频流描述

```cpp
// CDemuxStreamAudio — 音频流特有属性

class CDemuxStreamAudio : public CDemuxStream {
public:
    int iChannels;             // 声道数 (1=单声道, 2=立体声)
    int iSampleRate;           // 采样率 (如 44100, 48000)
    int iBitsPerSample;        // 每采样位数 (如 16, 24)
    int iBlockAlign;           // 块对齐（字节）
    int iBitRate;              // 比特率 (bps)
    uint64_t iChannelLayout;   // 声道布局 (FFmpeg channel_layout)
    
    // 从 AVStream 提取参数
    void SetFromAVStream(AVStream* avStream) {
        AVCodecParameters* par = avStream->codecpar;
        iChannels = par->channels;
        iSampleRate = par->sample_rate;
        iBitsPerSample = par->bits_per_coded_sample;
        iBlockAlign = par->block_align;
        iBitRate = par->bit_rate;
        iChannelLayout = par->channel_layout;
        codec = par->codec_id;
        
        // 如果声道布局未知，根据声道数推断
        if (iChannelLayout == 0) {
            iChannelLayout = av_get_default_channel_layout(
                iChannels);
        }
    }
};
```

---

## 3. CDVDStreamInfo 参数传递

CDVDStreamInfo 是从 CDemuxStream 到解码器的参数传输载体。它包含初始化解码器所需的全部信息，包括编解码器 ID、extradata（编解码器配置数据）和流特有的属性。

### 3.1 CDVDStreamInfo 定义

```cpp
// 源码: movie/ffmpeg/VideoPlayer.h (概念性定义)
// CDVDStreamInfo — 编解码参数传输结构

struct CDVDStreamInfo {
    // ── 通用属性 ──
    AVCodecID codec;           // 编解码器 ID (如 AV_CODEC_ID_H264)
    StreamType type;           // 流类型
    
    // 编解码器配置数据（extradata）
    // 对于 H.264: SPS/PPS 参数集
    // 对于 AAC: AudioSpecificConfig
    uint8_t* extradata;        
    int extrasize;             
    
    // ── 视频特有 ──
    int width;                 // 宽度
    int height;                // 高度
    float fps;                 // 帧率
    float aspect;              // 宽高比
    AVPixelFormat pixfmt;      // 像素格式 (如 AV_PIX_FMT_YUV420P)
    
    // ── 音频特有 ──
    int channels;              // 声道数
    int samplerate;            // 采样率
    int bitspersample;         // 每采样位数
    int blockalign;            // 块对齐
    uint64_t channellayout;    // 声道布局
    
    // 从 CDemuxStream 填充
    void FillFromDemuxStream(CDemuxStream* stream);
    
    // 从 AVCodecParameters 填充
    void FillFromCodecPar(AVCodecParameters* par);
};
```

### 3.2 参数填充流程

```cpp
// CDVDStreamInfo 的填充示例

void CDVDStreamInfo::FillFromDemuxStream(CDemuxStream* stream) {
    codec = stream->codec;
    type = stream->type;
    
    if (stream->type == STREAM_VIDEO) {
        CDemuxStreamVideo* vs = 
            static_cast<CDemuxStreamVideo*>(stream);
        width = vs->iWidth;
        height = vs->iHeight;
        fps = vs->fFps;
        aspect = vs->fAspect;
    }
    else if (stream->type == STREAM_AUDIO) {
        CDemuxStreamAudio* as = 
            static_cast<CDemuxStreamAudio*>(stream);
        channels = as->iChannels;
        samplerate = as->iSampleRate;
        bitspersample = as->iBitsPerSample;
        blockalign = as->iBlockAlign;
        channellayout = as->iChannelLayout;
    }
}

// 填充 extradata（编解码器配置数据）
void CDVDStreamInfo::FillExtradata(AVCodecParameters* par) {
    if (par->extradata && par->extradata_size > 0) {
        extrasize = par->extradata_size;
        extradata = new uint8_t[extrasize];
        memcpy(extradata, par->extradata, extrasize);
    } else {
        extradata = nullptr;
        extrasize = 0;
    }
}
```

---

## 4. 流选择策略

### 4.1 BasePlayer 的流选择

```cpp
// 源码: movie/ffmpeg/VideoPlayer.cpp
// OpenDemuxStream() 中的流选择逻辑

void BasePlayer::SelectDefaultStreams() {
    int bestVideoStream = -1;
    int bestAudioStream = -1;
    int bestVideoScore = -1;
    int bestAudioScore = -1;
    
    for (int i = 0; i < m_pDemuxer->GetNrOfStreams(); i++) {
        CDemuxStream* stream = m_pDemuxer->GetStream(i);
        
        if (stream->type == STREAM_VIDEO) {
            // 视频流评分策略：
            // 优先选择分辨率最高的流
            CDemuxStreamVideo* vs = 
                static_cast<CDemuxStreamVideo*>(stream);
            int score = vs->iWidth * vs->iHeight;
            
            if (score > bestVideoScore) {
                bestVideoScore = score;
                bestVideoStream = i;
            }
        }
        else if (stream->type == STREAM_AUDIO) {
            // 音频流评分策略：
            // 优先选择声道数最多的流
            // 如果声道数相同，优先选择日语音轨
            CDemuxStreamAudio* as = 
                static_cast<CDemuxStreamAudio*>(stream);
            int score = as->iChannels * 1000;
            
            // 日语优先（视觉小说常见需求）
            if (as->language == "jpn" || 
                as->language == "ja")
                score += 500;
            
            if (score > bestAudioScore) {
                bestAudioScore = score;
                bestAudioStream = i;
            }
        }
    }
    
    // 打开选中的流
    if (bestVideoStream >= 0)
        OpenStream(bestVideoStream, STREAM_VIDEO);
    if (bestAudioStream >= 0)
        OpenStream(bestAudioStream, STREAM_AUDIO);
}
```

### 4.2 OpenStream 流程

```cpp
// 源码: movie/ffmpeg/VideoPlayer.cpp
// OpenStream() — 打开一个流并初始化对应的解码器

bool BasePlayer::OpenStream(int streamId, StreamType type) {
    CDemuxStream* stream = m_pDemuxer->GetStream(streamId);
    if (!stream) return false;
    
    // 创建 CDVDStreamInfo 传输参数
    CDVDStreamInfo hint;
    hint.FillFromDemuxStream(stream);
    
    // 从 FFmpeg AVStream 获取 extradata
    AVStream* avStream = m_pDemuxer->GetAVStream(streamId);
    if (avStream)
        hint.FillExtradata(avStream->codecpar);
    
    if (type == STREAM_VIDEO) {
        // 初始化视频解码器
        m_CurrentVideo.id = streamId;
        m_CurrentVideo.hint = hint;
        
        // 通知视频播放器打开新流
        m_VideoPlayerVideo->OpenStream(hint);
        
        // 配置渲染管理器
        m_renderManager->Configure(
            hint.width, hint.height, 
            hint.fps, hint.aspect
        );
    }
    else if (type == STREAM_AUDIO) {
        // 初始化音频解码器
        m_CurrentAudio.id = streamId;
        m_CurrentAudio.hint = hint;
        
        m_VideoPlayerAudio->OpenStream(hint);
    }
    
    return true;
}
```

---

## 5. DemuxPacket 数据包结构

DemuxPacket 是解封装器输出的基本数据单元，每个包包含一帧（或部分帧）的压缩数据以及时间戳信息。

### 5.1 数据包定义

```cpp
// DemuxPacket — 解封装后的数据包

struct DemuxPacket {
    uint8_t* pData;        // 压缩数据指针
    int iSize;             // 数据大小（字节）
    int iStreamId;         // 所属流的索引
    
    double pts;            // 显示时间戳（DVD_TIME_BASE 微秒）
    double dts;            // 解码时间戳（DVD_TIME_BASE 微秒）
    double duration;       // 持续时间（DVD_TIME_BASE 微秒）
    
    int64_t demuxerId;     // 解封装器标识
    
    // 关键帧标志
    // 关键帧（I 帧）可以独立解码
    // 非关键帧（P/B 帧）依赖前面的帧
    bool isKeyFrame;
};

// 数据包分配与释放
class CDVDDemuxUtils {
public:
    static DemuxPacket* AllocateDemuxPacket(int dataSize) {
        DemuxPacket* pkt = new DemuxPacket();
        if (dataSize > 0) {
            // 使用 FFmpeg 的内存分配器（保证对齐）
            pkt->pData = (uint8_t*)av_malloc(
                dataSize + AV_INPUT_BUFFER_PADDING_SIZE);
            // 填充 padding（FFmpeg 要求数据末尾有填充字节）
            memset(pkt->pData + dataSize, 0, 
                   AV_INPUT_BUFFER_PADDING_SIZE);
        }
        pkt->iSize = dataSize;
        pkt->pts = DVD_NOPTS_VALUE;
        pkt->dts = DVD_NOPTS_VALUE;
        pkt->duration = 0;
        pkt->isKeyFrame = false;
        return pkt;
    }
    
    static void FreeDemuxPacket(DemuxPacket* pkt) {
        if (pkt) {
            av_free(pkt->pData);
            delete pkt;
        }
    }
};
```

### 5.2 PTS 与 DTS 的区别

```
PTS（Presentation Time Stamp）= 显示时间
DTS（Decode Time Stamp）= 解码时间

对于 I 帧和 P 帧：PTS == DTS（解码后立即可显示）

对于 B 帧：PTS != DTS（B 帧参考前后帧，需要提前解码）

时间线示例（含 B 帧的 MPEG 流）：

显示顺序：  I₁  B₂  B₃  P₄  B₅  B₆  P₇
解码顺序：  I₁  P₄  B₂  B₃  P₇  B₅  B₆

帧     DTS      PTS
I₁     0ms      0ms
P₄     33ms     100ms    ← P₄ 需要先解码，B₂B₃ 才能参考它
B₂     67ms     33ms     ← DTS > PTS（先解码 P₄ 再显示 B₂）
B₃     100ms    67ms
P₇     133ms    200ms
B₅     167ms    133ms
B₆     200ms    167ms

在 KrKr2 中，视觉小说的 OP/ED 视频通常使用 MPEG-1/2，
这些格式的 B 帧使用较少，大部分情况 PTS ≈ DTS。
```

---

## 动手实践

### 练习 1：打印流信息

```cpp
// 编写函数打印所有流的详细信息

void PrintStreamInfo(CDVDDemuxFFmpeg* demuxer) {
    printf("=== Stream Information ===\n");
    printf("Number of streams: %d\n", 
           demuxer->GetNrOfStreams());
    
    for (int i = 0; i < demuxer->GetNrOfStreams(); i++) {
        CDemuxStream* stream = demuxer->GetStream(i);
        
        printf("\nStream #%d: ", i);
        
        if (stream->type == STREAM_VIDEO) {
            CDemuxStreamVideo* vs = 
                static_cast<CDemuxStreamVideo*>(stream);
            printf("VIDEO\n");
            printf("  Codec: %s (%d)\n", 
                   avcodec_get_name(vs->codec), vs->codec);
            printf("  Resolution: %dx%d\n", 
                   vs->iWidth, vs->iHeight);
            printf("  FPS: %.2f\n", vs->fFps);
            printf("  Aspect: %.2f\n", vs->fAspect);
        }
        else if (stream->type == STREAM_AUDIO) {
            CDemuxStreamAudio* as = 
                static_cast<CDemuxStreamAudio*>(stream);
            printf("AUDIO\n");
            printf("  Codec: %s (%d)\n",
                   avcodec_get_name(as->codec), as->codec);
            printf("  Channels: %d\n", as->iChannels);
            printf("  Sample Rate: %d Hz\n", as->iSampleRate);
            printf("  Bits/Sample: %d\n", as->iBitsPerSample);
            printf("  Language: %s\n", 
                   as->language.empty() ? "unknown" : 
                   as->language.c_str());
        }
    }
}
```

### 练习 2：实现流切换

```cpp
// 模拟音轨切换的完整流程

bool BasePlayer::SwitchAudioStream(int newStreamId) {
    CDemuxStream* stream = m_pDemuxer->GetStream(newStreamId);
    if (!stream || stream->type != STREAM_AUDIO)
        return false;
    
    // 1. 刷新当前音频缓冲
    m_VideoPlayerAudio->SendMessage(
        new CDVDMsg(CDVDMsg::GENERAL_FLUSH));
    
    // 2. 关闭当前音频解码器
    m_VideoPlayerAudio->CloseStream();
    
    // 3. 更新流信息
    CDVDStreamInfo hint;
    hint.FillFromDemuxStream(stream);
    
    // 4. 打开新的音频解码器
    m_CurrentAudio.id = newStreamId;
    m_CurrentAudio.hint = hint;
    m_VideoPlayerAudio->OpenStream(hint);
    
    return true;
}
```

---

## 对照项目源码

| 本节概念 | 对应源文件 | 关键代码位置 |
|----------|-----------|-------------|
| CDemuxStream 基类 | `DemuxFFmpeg.h` | `class CDemuxStream` |
| CDemuxStreamVideo | `DemuxFFmpeg.h` | `class CDemuxStreamVideo` |
| CDemuxStreamAudio | `DemuxFFmpeg.h` | `class CDemuxStreamAudio` |
| CDVDStreamInfo | `VideoPlayer.h` | `struct CDVDStreamInfo` |
| DemuxPacket | `DemuxFFmpeg.h` | `struct DemuxPacket` |
| CreateStreams | `DemuxFFmpeg.cpp` | `CDVDDemuxFFmpeg::CreateStreams()` |
| OpenStream | `VideoPlayer.cpp` | `BasePlayer::OpenStream()` |
| CCurrentStream | `VideoPlayer.h` | `struct CCurrentStream` |

---

## 本节小结

1. CDemuxStream 层次结构描述容器中的各个流，区分视频流和音频流的特有属性
2. CDVDStreamInfo 是从解封装层到解码层的参数传输载体，包含初始化解码器所需的全部信息
3. extradata 包含编解码器的配置数据（如 H.264 的 SPS/PPS），是正确初始化解码器的关键
4. BasePlayer 通过分辨率（视频）和声道数+语言（音频）来选择默认播放的流
5. DemuxPacket 包含压缩数据和时间戳（PTS/DTS），是解码器的输入单元

---

## 常见错误及解决方案

### 错误 1：解码器初始化失败——缺少 extradata

```
avcodec_open2 failed: "no codec parameters"
```

**原因**：H.264/H.265 等编解码器需要 extradata（SPS/PPS）才能初始化。如果 CDVDStreamInfo 没有正确传递 extradata，解码器无法打开。

**解决方案**：
1. 确保 `FillExtradata()` 正确复制了 `AVCodecParameters::extradata`
2. 检查 `avformat_find_stream_info()` 是否成功（它负责探测 extradata）

### 错误 2：视频宽高比异常

```
视频画面被拉伸或压缩
```

**原因**：未正确处理 `sample_aspect_ratio`（像素宽高比）。某些视频使用非方形像素。

**解决方案**：
1. 在计算显示宽高比时乘以 `sample_aspect_ratio`
2. 公式：`display_aspect = width * SAR_num / (height * SAR_den)`

---

## 练习题与答案

### 练习 1：为什么 DemuxPacket 需要 AV_INPUT_BUFFER_PADDING_SIZE 的填充？

<details>
<summary>查看答案</summary>

FFmpeg 的解码器在内部使用 SIMD（单指令多数据）优化来加速解码。SIMD 指令通常一次处理 16 或 32 字节的数据。如果数据缓冲区恰好结束在某个位置，SIMD 读取可能会越界访问。

`AV_INPUT_BUFFER_PADDING_SIZE`（通常为 64 字节）在数据末尾添加填充：
1. **防止越界**：SIMD 指令越界读取时读到的是填充的零字节，不会访问非法内存
2. **简化边界检查**：解码器不需要在每个 SIMD 操作前检查是否到达数据末尾
3. **填充零字节**：确保越界读取不会导致异常数据被当作输入处理

</details>

### 练习 2：在什么情况下一个容器会有多个视频流？

<details>
<summary>查看答案</summary>

1. **多角度视频**：DVD/蓝光中的多角度功能，同一场景有不同摄像角度的视频流
2. **缩略图流**：某些 MKV 文件包含一个小分辨率的缩略图视频流用于快速预览
3. **字幕图片流**：DVD 的字幕实际上是图片流（VOBSUB），以视频流的形式存在
4. **多分辨率自适应**：HLS/DASH 自适应流包含不同分辨率的视频流（但在 KrKr2 中不适用）

在视觉小说的 OP/ED 视频中，通常只有一个视频流和一个音频流，多流的情况很罕见。

</details>

### 练习 3：CDVDStreamInfo 中的 codec 字段使用 AVCodecID 而非字符串，有什么优势？

<details>
<summary>查看答案</summary>

1. **性能**：整数比较比字符串比较快得多，流选择和解码器工厂查找更高效
2. **类型安全**：枚举值在编译期检查，防止拼写错误（如 "h264" vs "H264" vs "h.264"）
3. **统一标识**：FFmpeg 对同一编解码器可能有多个名称（如 "aac" 和 "libfdk_aac"），但 AVCodecID 唯一
4. **switch 友好**：可以直接在 switch 语句中使用，代码结构更清晰
5. **空间效率**：一个整数占 4 字节，而编解码器名称字符串可能占 10+ 字节

</details>

---

## 下一步

下一节 [2.3 音视频解码器](03-音视频解码器.md) 将分析 CDVDVideoCodecFFmpeg 和 CDVDAudioCodecFFmpeg 的实现，理解如何使用 FFmpeg 的解码 API 将压缩数据解码为原始的 YUV 帧和 PCM 采样。
