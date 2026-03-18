# Vorbis 与 FFmpeg 解码器实现

> **所属模块：** M05-音频子系统  
> **前置知识：** [01-tTVPWaveDecoder接口详解](./01-tTVPWaveDecoder接口详解.md)、[02-Creator注册与格式探测](./02-Creator注册与格式探测.md)  
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 libvorbisfile 和 libopus 的回调机制与集成方式
2. 掌握 FFmpeg 音频解码的完整流程（解复用、解码、格式处理）
3. 分析 Planar 与 Interleaved 采样格式的区别与转换方法
4. 实现自定义流式数据源与解码器的绑定
5. 处理解码过程中的定位、缓冲和边界情况

---

## Vorbis 解码器实现分析

### 架构概述

KiriKiri2 的 Vorbis 解码器使用 `libvorbisfile` 库，通过回调函数实现流式读取：

```
┌────────────────────────────────────────────────────────────┐
│                   VorbisWaveDecoder                        │
├────────────────────────────────────────────────────────────┤
│  - InputFile : OggVorbis_File    // libvorbis 文件句柄     │
│  - InputStream : tTJSBinaryStream*  // KiriKiri 流         │
│  - Format : tTVPWaveFormat       // 输出格式信息           │
│  - CurrentSection : int          // 当前流段               │
│  - pcmsize : int                 // 每采样字节数           │
├────────────────────────────────────────────────────────────┤
│  回调函数（静态成员）:                                      │
│  - read_func()   → InputStream->Read()                     │
│  - seek_func()   → InputStream->Seek()                     │
│  - tell_func()   → InputStream->GetPosition()              │
│  - close_func()  → delete InputStream                      │
└────────────────────────────────────────────────────────────┘
              │
              ▼
┌────────────────────────────────────────────────────────────┐
│                   libvorbisfile                            │
│  ov_open_callbacks() → ov_read() → ov_pcm_seek()          │
└────────────────────────────────────────────────────────────┘
```

### 回调函数实现

libvorbisfile 通过 `ov_callbacks` 结构体接收 I/O 回调：

```cpp
// 文件: cpp/core/sound/VorbisWaveDecoder.cpp 第 83-129 行

// 读取回调 — 从 KiriKiri 流读取数据
static size_t read_func(void *ptr, size_t size, size_t nmemb, 
                        void *datasource) {
    VorbisWaveDecoder *decoder = (VorbisWaveDecoder *)datasource;
    if (!decoder->InputStream) return 0;
    
    // KiriKiri 的 Read 返回实际读取字节数
    // 需要转换为元素个数返回
    return decoder->InputStream->Read(ptr, size * nmemb) / size;
}

// 定位回调 — 支持 SEEK_SET/CUR/END
static int seek_func(void *datasource, ogg_int64_t offset, int whence) {
    VorbisWaveDecoder *decoder = (VorbisWaveDecoder *)datasource;
    if (!decoder->InputStream) return -1;
    
    // 映射标准 whence 到 KiriKiri 常量
    int seek_type = TJS_BS_SEEK_SET;
    switch (whence) {
        case SEEK_SET: seek_type = TJS_BS_SEEK_SET; break;
        case SEEK_CUR: seek_type = TJS_BS_SEEK_CUR; break;
        case SEEK_END: seek_type = TJS_BS_SEEK_END; break;
    }
    
    decoder->InputStream->Seek(offset, seek_type);
    return 0;  // 成功返回 0
}

// 关闭回调 — 释放 KiriKiri 流
static int close_func(void *datasource) {
    VorbisWaveDecoder *decoder = (VorbisWaveDecoder *)datasource;
    if (!decoder->InputStream) return EOF;
    
    delete decoder->InputStream;
    decoder->InputStream = nullptr;
    return 0;
}

// 位置查询回调
static long tell_func(void *datasource) {
    VorbisWaveDecoder *decoder = (VorbisWaveDecoder *)datasource;
    if (!decoder->InputStream) return EOF;
    
    // GetPosition 等价于 Seek(0, SEEK_CUR)
    return decoder->InputStream->Seek(0, TJS_BS_SEEK_CUR);
}
```

### 流初始化

```cpp
// 文件: cpp/core/sound/VorbisWaveDecoder.cpp 第 211-257 行
bool VorbisWaveDecoder::SetStream(const ttstr &url) {
    // 1. 打开 KiriKiri 存储流
    InputStream = TVPCreateStream(url, TJS_BS_READ);
    if (!InputStream) return false;
    
    // 2. 设置回调结构
    ov_callbacks callbacks = { 
        read_func, 
        seek_func, 
        close_func, 
        tell_func 
    };
    
    // 3. 通过回调打开 Vorbis 文件
    //    this 指针作为 datasource 传递给回调
    if (ov_open_callbacks(this, &InputFile, nullptr, 0, callbacks) < 0) {
        delete InputStream;
        InputStream = nullptr;
        return false;
    }
    InputFileInit = true;
    
    // 4. 获取 Vorbis 流信息
    vorbis_info *vi = ov_info(&InputFile, -1);  // -1 表示当前链接
    if (!vi) return false;
    
    // 5. 填充 tTVPWaveFormat
    Format.BitsPerSample = FloatExtraction ? 32 : 16;
    Format.BytesPerSample = Format.BitsPerSample / 8;
    Format.Channels = vi->channels;
    Format.IsFloat = FloatExtraction;
    Format.SamplesPerSec = vi->rate;
    Format.Seekable = true;  // Vorbis 总是可定位
    Format.SpeakerConfig = 0;
    
    // 6. 获取总时长
    ogg_int64_t pcmtotal = ov_pcm_total(&InputFile, -1);
    Format.TotalSamples = (pcmtotal < 0) ? 0 : pcmtotal;
    
    double timetotal = ov_time_total(&InputFile, -1);  // 秒
    Format.TotalTime = (timetotal < 0) ? 0 : timetotal * 1000.0;
    
    return true;
}
```

### 渲染实现

Vorbis 解码器支持两种输出格式：16-bit 整数和 32-bit 浮点：

```cpp
// 文件: cpp/core/sound/VorbisWaveDecoder.cpp 第 146-197 行
bool VorbisWaveDecoder::Render(void *buf, tjs_uint bufsamplelen, 
                               tjs_uint &rendered) {
    if (!InputFileInit) return false;
    
    int pos = 0;  // 已解码粒度数
    
    if (Format.IsFloat) {
        // 浮点输出模式 — 使用 ov_read_float
        float *p = (float *)buf;
        int remain = bufsamplelen;
        
        while (remain) {
            float **pcm;  // 各声道的浮点数组指针
            int current_section;
            int samples;
            
            do {
                // ov_read_float 返回 planar 格式数据
                samples = ov_read_float(&InputFile, &pcm, remain, 
                                        &current_section);
            } while (samples < 0);  // 负数表示临时错误，重试
            
            if (samples == 0) break;  // EOF
            
            // 交织 planar 数据到输出缓冲区
            for (int j = 0; j < samples; j++) {
                for (int i = 0; i < Format.Channels; i++) {
                    *p++ = pcm[i][j];  // pcm[channel][sample]
                }
            }
            
            pos += samples;
            remain -= samples;
        }
    } else {
        // 16-bit 整数输出模式 — 使用 ov_read
        int remain = bufsamplelen * Format.Channels * pcmsize;  // 字节
        
        while (remain) {
            int res;
            do {
                // ov_read 直接输出 interleaved 格式
                res = ov_read(&InputFile, 
                              (char *)buf + pos, 
                              remain,
                              0,           // 小端序
                              pcmsize,     // 每采样字节数
                              1,           // 有符号
                              &CurrentSection);
            } while (res < 0);
            
            if (res == 0) break;  // EOF
            
            pos += res;
            remain -= res;
        }
        
        // 转换为粒度数
        pos /= (Format.Channels * pcmsize);
    }
    
    rendered = pos;
    return (unsigned int)pos >= bufsamplelen;  // 是否还有更多数据
}
```

### 动态格式切换

Vorbis 解码器支持运行时切换输出格式：

```cpp
// 文件: cpp/core/sound/VorbisWaveDecoder.cpp 第 62-77 行
bool VorbisWaveDecoder::DesiredFormat(const tTVPWaveFormat &format) {
    // 如果请求的格式与当前相同，无需切换
    if (format.IsFloat == Format.IsFloat) return false;
    
    if (format.IsFloat) {
        // 切换到浮点输出
        pcmsize = 4;
        Format.IsFloat = true;
        Format.BitsPerSample = 32;
        Format.BytesPerSample = 4;
    } else {
        // 切换到整数输出
        pcmsize = 2;
        Format.IsFloat = false;
        Format.BitsPerSample = 16;
        Format.BytesPerSample = 2;
    }
    
    return true;  // 表示格式已切换
}
```

---

## Opus 解码器实现分析

### 与 Vorbis 的区别

Opus 是更现代的音频编解码器，KiriKiri2 使用 `libopusfile` 库：

| 特性 | Vorbis | Opus |
|------|--------|------|
| 库 | libvorbisfile | libopusfile |
| 采样率 | 可变（常见 44.1/48kHz） | 固定 48kHz |
| 输出格式 | 16-bit 或 float | 仅 16-bit |
| API 函数 | `ov_read`, `ov_read_float` | `op_read` |
| 内部缓冲 | 无需 | 需要 5760 采样缓冲 |

### Opus 特有的缓冲机制

```cpp
// 文件: cpp/core/sound/VorbisWaveDecoder.cpp 第 316-325 行
class OpusWaveDecoder : public tTVPWaveDecoder {
    OggOpusFile *InputFile;
    tTJSBinaryStream *InputStream;
    tTVPWaveFormat Format;
    
    // Opus 特有：内部缓冲区
    opus_int16 *Buffer;   // 5760 采样 × 声道数
    opus_int16 *pBuffer;  // 当前读取位置
    int BufferRemain;     // 缓冲区剩余采样数
    
    // ...
};
```

为什么需要内部缓冲？Opus 的 `op_read` 要求至少 5760 采样（120ms @ 48kHz）的缓冲区，但调用者可能请求更少：

```cpp
// 文件: cpp/core/sound/VorbisWaveDecoder.cpp 第 403-444 行
bool OpusWaveDecoder::Render(void *_buf, tjs_uint bufsamplelen,
                             tjs_uint &rendered) {
    if (!InputFile) return false;
    
    opus_int16 *buf = (opus_int16 *)_buf;
    int pos = 0;
    int remain = bufsamplelen;
    
    while (remain) {
        int res;
        
        if (pBuffer) {
            // 情况 1：从内部缓冲区读取剩余数据
            res = BufferRead(buf, remain);
        } 
        else if (remain >= 5760) {
            // 情况 2：请求量足够大，直接解码到用户缓冲区
            res = op_read(InputFile, buf, remain * Format.Channels, nullptr);
        } 
        else {
            // 情况 3：请求量太小，先解码到内部缓冲区
            res = op_read(InputFile, Buffer, 5760 * Format.Channels, nullptr);
            
            if (res > remain) {
                // 只复制请求的量，剩余保存在缓冲区
                memcpy(buf, Buffer, remain * Format.Channels * sizeof(opus_int16));
                BufferRemain = res - remain;
                pBuffer = Buffer + remain * Format.Channels;
                res = remain;
            } else {
                // 全部复制
                memcpy(buf, Buffer, res * Format.Channels * sizeof(opus_int16));
            }
        }
        
        if (res <= 0) break;
        
        pos += res;
        remain -= res;
        buf += res * Format.Channels;
    }
    
    rendered = pos;
    return !remain;
}
```

---

## FFmpeg 解码器实现分析

### 架构概述

FFmpeg 解码器是通用后备，支持几乎所有音频格式：

```
┌─────────────────────────────────────────────────────────────────┐
│                      FFWaveDecoder                              │
├─────────────────────────────────────────────────────────────────┤
│  文件层:                                                        │
│  - FormatCtx : AVFormatContext*   // 容器格式上下文             │
│  - InputStream : tTJSBinaryStream*                              │
│  - AVIOContext (自定义 I/O)                                     │
├─────────────────────────────────────────────────────────────────┤
│  流层:                                                          │
│  - StreamIdx : int                // 音频流索引                 │
│  - AudioStream : AVStream*        // 音频流指针                 │
├─────────────────────────────────────────────────────────────────┤
│  解码层:                                                        │
│  - Packet : AVPacket              // 压缩数据包                 │
│  - frame : AVFrame*               // 解码后帧                   │
│  - AVFmt : AVSampleFormat         // 采样格式                   │
│  - IsPlanar : bool                // 是否为 planar 格式         │
├─────────────────────────────────────────────────────────────────┤
│  缓冲管理:                                                      │
│  - audio_buf_index : int          // 当前帧读取位置             │
│  - audio_buf_samples : int        // 当前帧总采样数             │
│  - audio_frame_next_pts : int64_t // 下一帧时间戳               │
└─────────────────────────────────────────────────────────────────┘
```

### 自定义 I/O 回调

FFmpeg 通过 `AVIOContext` 实现自定义数据源：

```cpp
// 文件: cpp/core/sound/FFWaveDecoder.cpp 第 85-98 行

// 读取回调
static int AVReadFunc(void *opaque, uint8_t *buf, int buf_size) {
    TJS::tTJSBinaryStream *stream = (TJS::tTJSBinaryStream *)opaque;
    return stream->Read(buf, buf_size);
}

// 定位回调（支持 AVSEEK_SIZE 查询文件大小）
static int64_t AVSeekFunc(void *opaque, int64_t offset, int whence) {
    TJS::tTJSBinaryStream *stream = (TJS::tTJSBinaryStream *)opaque;
    
    switch (whence) {
        case AVSEEK_SIZE:
            // FFmpeg 查询文件总大小
            return stream->GetSize();
        default:
            // 标准定位操作
            return stream->Seek(offset, whence & 0xFF);
    }
}
```

### 流初始化

```cpp
// 文件: cpp/core/sound/FFWaveDecoder.cpp 第 256-362 行
bool FFWaveDecoder::SetStream(const ttstr &url) {
    Clear();
    
    // 1. 打开 KiriKiri 流
    InputStream = TVPCreateBinaryStreamForRead(url, TJS_W(""));
    if (!InputStream) return false;
    
    // 2. 创建自定义 AVIOContext
    int bufSize = 32 * 1024;  // 32KB 缓冲
    AVIOContext *pIOCtx = avio_alloc_context(
        (unsigned char *)av_malloc(bufSize + AVPROBE_PADDING_SIZE),
        bufSize,
        0,               // 只读
        InputStream,     // opaque 指针
        AVReadFunc,      // 读取回调
        nullptr,         // 无写入回调
        AVSeekFunc       // 定位回调
    );
    
    // 3. 探测格式
    AVInputFormat *fmt = nullptr;
    tTJSNarrowStringHolder holder(url.c_str());
    av_probe_input_buffer2(pIOCtx, &fmt, holder, nullptr, 0, 0);
    
    // 4. 打开容器
    AVFormatContext *ic = FormatCtx = avformat_alloc_context();
    ic->pb = pIOCtx;
    if (avformat_open_input(&ic, "", fmt, nullptr) < 0) {
        FormatCtx = nullptr;
        return false;
    }
    
    // 5. 查找流信息
    if (avformat_find_stream_info(ic, nullptr) < 0) {
        return false;
    }
    
    // 6. 找到最佳音频流
    StreamIdx = av_find_best_stream(ic, AVMEDIA_TYPE_AUDIO, -1, -1, 
                                    nullptr, 0);
    if (StreamIdx < 0) return false;
    
    AVCodecContext *avctx = ic->streams[StreamIdx]->codec;
    
    // 7. 查找并打开解码器
    AVCodec *codec = avcodec_find_decoder(avctx->codec_id);
    if (!codec) return false;
    
    if (avcodec_open2(avctx, codec, nullptr) < 0) {
        return false;
    }
    
    // 8. 填充格式信息
    Format.SamplesPerSec = avctx->sample_rate;
    Format.Channels = avctx->channels;
    
    // 判断是否可定位
    Format.Seekable = (FormatCtx->iformat->flags & 
        (AVFMT_NOBINSEARCH | AVFMT_NOGENSEARCH | AVFMT_NO_BYTE_SEEK)) !=
        (AVFMT_NOBINSEARCH | AVFMT_NOGENSEARCH | AVFMT_NO_BYTE_SEEK);
    
    // 处理不同的采样格式
    switch (AVFmt = avctx->sample_fmt) {
        case AV_SAMPLE_FMT_S16P:
        case AV_SAMPLE_FMT_S16:
            Format.BitsPerSample = 16;
            Format.IsFloat = false;
            break;
        case AV_SAMPLE_FMT_FLTP:
        case AV_SAMPLE_FMT_FLT:
            Format.BitsPerSample = 32;
            Format.IsFloat = true;
            break;
        case AV_SAMPLE_FMT_S32P:
        case AV_SAMPLE_FMT_S32:
            Format.BitsPerSample = 32;
            Format.IsFloat = false;
            break;
        default:
            return false;  // 不支持的格式
    }
    
    // 判断是否为 planar 格式
    IsPlanar = (AVFmt == AV_SAMPLE_FMT_S16P || 
                AVFmt == AV_SAMPLE_FMT_FLTP ||
                AVFmt == AV_SAMPLE_FMT_S32P);
    
    Format.BytesPerSample = Format.BitsPerSample / 8;
    
    // 计算总时长
    AudioStream = FormatCtx->streams[StreamIdx];
    Format.TotalTime = av_q2d(AudioStream->time_base) * 
                       AudioStream->duration * 1000;
    Format.TotalSamples = av_q2d(AudioStream->time_base) * 
                          AudioStream->duration * Format.SamplesPerSec;
    
    return true;
}
```

### Planar 与 Interleaved 格式

FFmpeg 解码后的数据可能是 planar（各声道分离）或 interleaved（交织）格式：

```
Interleaved 格式（期望输出）:
┌──────────────────────────────────────────────────────┐
│ L0 R0 L1 R1 L2 R2 L3 R3 ...                         │
└──────────────────────────────────────────────────────┘

Planar 格式（某些编解码器输出）:
┌──────────────────────────────────────────────────────┐
│ frame->data[0]: L0 L1 L2 L3 ...                     │
│ frame->data[1]: R0 R1 R2 R3 ...                     │
└──────────────────────────────────────────────────────┘
```

项目实现了高效的格式转换：

```cpp
// 文件: cpp/core/sound/FFWaveDecoder.cpp 第 112-158 行

// 通用版本（任意声道数）
template <typename T>
static unsigned char *_CopySamples(unsigned char *dst, AVFrame *frame,
                                   int samples, int buf_index) {
    int buf_pos = buf_index * sizeof(T);
    T *pDst = (T *)dst;
    
    for (int i = 0; i < samples; ++i, buf_pos += sizeof(T)) {
        for (int j = 0; j < frame->channels; ++j) {
            // 从各声道数组中取出对应位置的采样
            *pDst++ = *(T *)(frame->data[j] + buf_pos);
        }
    }
    
    return (unsigned char *)pDst;
}

// 立体声优化版本（展开循环）
template <typename T>
static unsigned char *_CopySamples2(unsigned char *dst, AVFrame *frame,
                                    int samples, int buf_index) {
    int buf_pos = buf_index * sizeof(T);
    T *pDst = (T *)dst;
    
    for (int i = 0; i < samples; ++i, buf_pos += sizeof(T)) {
        *pDst++ = *(T *)(frame->data[0] + buf_pos);  // 左声道
        *pDst++ = *(T *)(frame->data[1] + buf_pos);  // 右声道
    }
    
    return (unsigned char *)pDst;
}

// 调度函数：根据格式和声道数选择实现
static unsigned char *CopySamples(unsigned char *dst, AVFrame *frame,
                                  int samples, int buf_index) {
    switch (frame->format) {
        case AV_SAMPLE_FMT_FLTP:
        case AV_SAMPLE_FMT_S32P:
            if (frame->channels == 2)
                return _CopySamples2<tjs_uint32>(dst, frame, samples, buf_index);
            else
                return _CopySamples<tjs_uint32>(dst, frame, samples, buf_index);
            
        case AV_SAMPLE_FMT_S16P:
            if (frame->channels == 2)
                return _CopySamples2<tjs_uint16>(dst, frame, samples, buf_index);
            else
                return _CopySamples<tjs_uint16>(dst, frame, samples, buf_index);
    }
    
    return nullptr;
}
```

### 解码主循环

```cpp
// 文件: cpp/core/sound/FFWaveDecoder.cpp 第 160-196 行
bool FFWaveDecoder::Render(void *buf, tjs_uint bufsamplelen,
                           tjs_uint &rendered) {
    if (!InputStream) return false;
    
    int remain = bufsamplelen;
    int sample_size = av_samples_get_buffer_size(nullptr, Format.Channels, 
                                                  1, AVFmt, 1);
    unsigned char *stream = (unsigned char *)buf;
    
    while (remain) {
        // 1. 检查当前帧是否用完
        if (audio_buf_index >= audio_buf_samples) {
            // 解码新帧
            int decoded_samples = audio_decode_frame();
            if (decoded_samples < 0) break;  // EOF 或错误
            
            audio_buf_samples = decoded_samples;
            audio_buf_index = 0;
        }
        
        // 2. 计算本次可复制的采样数
        int samples = audio_buf_samples - audio_buf_index;
        if (samples > remain) samples = remain;
        
        // 3. 复制数据（处理 planar/interleaved 差异）
        if (!IsPlanar || Format.Channels == 1) {
            // Interleaved 或单声道：直接 memcpy
            memcpy(stream, 
                   frame->data[0] + audio_buf_index * sample_size,
                   samples * sample_size);
            stream += samples * sample_size;
        } else {
            // Planar 多声道：需要交织
            stream = CopySamples(stream, frame, samples, audio_buf_index);
        }
        
        remain -= samples;
        audio_buf_index += samples;
    }
    
    rendered = bufsamplelen - remain;
    return !remain;
}
```

### 帧解码函数

```cpp
// 文件: cpp/core/sound/FFWaveDecoder.cpp 第 365-436 行
int FFWaveDecoder::audio_decode_frame() {
    AVStream *audio_st = AudioStream;
    AVCodecContext *dec = audio_st->codec;
    
    for (;;) {
        // 1. 处理当前包中剩余的帧
        while (pkt_temp.stream_index != -1) {
            if (!frame) {
                frame = av_frame_alloc();
            } else {
                av_frame_unref(frame);
            }
            
            int got_frame;
            int len1 = avcodec_decode_audio4(dec, frame, &got_frame, &pkt_temp);
            
            if (len1 < 0) {
                // 解码错误，跳过当前包
                pkt_temp.size = 0;
                break;
            }
            
            // 更新包指针
            pkt_temp.data += len1;
            pkt_temp.size -= len1;
            
            // 包用完了
            if ((pkt_temp.data && pkt_temp.size <= 0) ||
                (!pkt_temp.data && !got_frame)) {
                pkt_temp.stream_index = -1;
            }
            
            if (!got_frame) continue;
            
            // 2. 更新时间戳
            AVRational tb = { 1, frame->sample_rate };
            
            if (frame->pts != AV_NOPTS_VALUE) {
                frame->pts = av_rescale_q(frame->pts, dec->time_base, tb);
            } else if (frame->pkt_pts != AV_NOPTS_VALUE) {
                frame->pts = av_rescale_q(frame->pkt_pts, 
                                          audio_st->time_base, tb);
            } else if (audio_frame_next_pts != AV_NOPTS_VALUE) {
                AVRational a = { 1, (int)Format.SamplesPerSec };
                frame->pts = av_rescale_q(audio_frame_next_pts, a, tb);
            }
            
            if (frame->pts != AV_NOPTS_VALUE) {
                audio_frame_next_pts = frame->pts + frame->nb_samples;
            }
            
            return frame->nb_samples;
        }
        
        // 3. 释放旧包，读取新包
        if (Packet.data) av_free_packet(&Packet);
        pkt_temp.stream_index = -1;
        
        if (!ReadPacket()) return -1;  // EOF
        
        pkt_temp = Packet;
    }
    
    return -1;
}
```

---

## 对照项目源码

### Creator 实现对比

| Creator | 扩展名过滤 | 文件验证 | 文件位置 |
|---------|-----------|---------|---------|
| VorbisWaveDecoderCreator | `.ogg` | 通过 ov_open 验证 | `VorbisWaveDecoder.cpp:132-144` |
| OpusWaveDecoderCreator | 无（尝试所有文件） | 通过 op_open 验证 | `VorbisWaveDecoder.cpp:504-512` |
| FFWaveDecoderCreator | 无（通用后备） | 通过 avformat_open 验证 | `FFWaveDecoder.cpp:100-110` |

**相关文件：**
- `cpp/core/sound/VorbisWaveDecoder.cpp` — Vorbis 和 Opus 解码器
- `cpp/core/sound/FFWaveDecoder.cpp` — FFmpeg 通用解码器
- `cpp/core/sound/WaveIntf.cpp` 第 710-729 行 — 注册机制

---

## 常见错误与排查

### 错误 1：回调函数签名不匹配

```cpp
// 错误：libvorbis 期望 size_t 返回值
int read_func_WRONG(void *ptr, size_t size, size_t nmemb, void *datasource);

// 正确：
size_t read_func(void *ptr, size_t size, size_t nmemb, void *datasource);

// libopus 回调与 libvorbis 不同
// Opus:
int read_func(void *datasource, unsigned char *ptr, int size);
// Vorbis:
size_t read_func(void *ptr, size_t size, size_t nmemb, void *datasource);
```

### 错误 2：Opus 缓冲区过小

```cpp
// 错误：Opus 要求至少 5760 采样
opus_int16 buffer[1024];  // 太小！
op_read(file, buffer, 1024 * channels, nullptr);  // 可能失败

// 正确：
opus_int16 buffer[5760 * 2];  // 立体声 120ms
op_read(file, buffer, 5760 * channels, nullptr);
```

### 错误 3：忽略 Planar 格式

```cpp
// 错误：假设所有数据都是 interleaved
memcpy(output, frame->data[0], samples * channels * bytes_per_sample);

// 正确：检查 planar 标志
if (av_sample_fmt_is_planar(format)) {
    // 需要交织各声道数据
    for (int i = 0; i < samples; i++) {
        for (int ch = 0; ch < channels; ch++) {
            memcpy(output, frame->data[ch] + i * bytes_per_sample, 
                   bytes_per_sample);
            output += bytes_per_sample;
        }
    }
} else {
    memcpy(output, frame->data[0], samples * channels * bytes_per_sample);
}
```

---

## 本节小结

- Vorbis 解码器通过 `ov_callbacks` 实现自定义 I/O，支持 16-bit 和浮点输出切换
- Opus 解码器固定 48kHz/16-bit 输出，需要 5760 采样的内部缓冲区
- FFmpeg 解码器是通用后备，需处理 planar/interleaved 格式差异
- 所有解码器通过回调函数桥接 KiriKiri 的 `tTJSBinaryStream` 与第三方库
- Planar 格式转换可通过模板函数和立体声特化优化性能

---

## 练习题与答案

### 题目 1：分析回调参数

Vorbis 的 `read_func` 返回 `stream->Read(...) / size`，为什么要除以 `size`？

<details>
<summary>查看答案</summary>

`libvorbisfile` 的读取回调签名模仿标准 `fread`：

```cpp
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
// 返回值：成功读取的 **元素个数**，不是字节数
```

而 KiriKiri 的 `Read` 返回实际读取的 **字节数**：

```cpp
tjs_uint tTJSBinaryStream::Read(void *buf, tjs_uint size);
// 返回值：实际读取的字节数
```

因此需要转换：

```cpp
// stream->Read(ptr, size * nmemb) 返回读取的字节数
// 除以 size 得到元素个数，符合 fread 语义
return stream->Read(ptr, size * nmemb) / size;
```

例如：请求读取 10 个 4 字节元素（总 40 字节）
- KiriKiri Read 返回 32（实际读取 32 字节）
- 回调返回 32 / 4 = 8（成功读取 8 个元素）

</details>

### 题目 2：实现 FLAC 回调

为 libFLAC 实现读取回调。libFLAC 的回调签名为：

```cpp
FLAC__StreamDecoderReadStatus read_callback(
    const FLAC__StreamDecoder *decoder,
    FLAC__byte buffer[],
    size_t *bytes,  // 输入/输出：请求/实际字节数
    void *client_data
);
```

<details>
<summary>查看答案</summary>

```cpp
#include <FLAC/stream_decoder.h>

// 假设 client_data 指向包含 tTJSBinaryStream* 的解码器对象
class FLACWaveDecoder {
    tTJSBinaryStream *InputStream;
    // ...
};

FLAC__StreamDecoderReadStatus flac_read_callback(
    const FLAC__StreamDecoder *decoder,
    FLAC__byte buffer[],
    size_t *bytes,
    void *client_data)
{
    FLACWaveDecoder *self = (FLACWaveDecoder *)client_data;
    
    if (!self->InputStream) {
        *bytes = 0;
        return FLAC__STREAM_DECODER_READ_STATUS_ABORT;
    }
    
    // 尝试读取请求的字节数
    size_t requested = *bytes;
    size_t actual = self->InputStream->Read(buffer, requested);
    
    *bytes = actual;  // 返回实际读取的字节数
    
    if (actual == 0) {
        // 检查是真正的 EOF 还是错误
        // 假设流末尾 GetPosition() == GetSize()
        if (self->InputStream->GetPosition() >= 
            self->InputStream->GetSize()) {
            return FLAC__STREAM_DECODER_READ_STATUS_END_OF_STREAM;
        } else {
            return FLAC__STREAM_DECODER_READ_STATUS_ABORT;
        }
    }
    
    return FLAC__STREAM_DECODER_READ_STATUS_CONTINUE;
}

// 定位回调
FLAC__StreamDecoderSeekStatus flac_seek_callback(
    const FLAC__StreamDecoder *decoder,
    FLAC__uint64 absolute_byte_offset,
    void *client_data)
{
    FLACWaveDecoder *self = (FLACWaveDecoder *)client_data;
    
    if (!self->InputStream) {
        return FLAC__STREAM_DECODER_SEEK_STATUS_ERROR;
    }
    
    tjs_uint64 result = self->InputStream->Seek(
        absolute_byte_offset, TJS_BS_SEEK_SET);
    
    if (result == absolute_byte_offset) {
        return FLAC__STREAM_DECODER_SEEK_STATUS_OK;
    } else {
        return FLAC__STREAM_DECODER_SEEK_STATUS_ERROR;
    }
}

// 位置回调
FLAC__StreamDecoderTellStatus flac_tell_callback(
    const FLAC__StreamDecoder *decoder,
    FLAC__uint64 *absolute_byte_offset,
    void *client_data)
{
    FLACWaveDecoder *self = (FLACWaveDecoder *)client_data;
    
    if (!self->InputStream) {
        return FLAC__STREAM_DECODER_TELL_STATUS_ERROR;
    }
    
    *absolute_byte_offset = self->InputStream->GetPosition();
    return FLAC__STREAM_DECODER_TELL_STATUS_OK;
}

// 长度回调
FLAC__StreamDecoderLengthStatus flac_length_callback(
    const FLAC__StreamDecoder *decoder,
    FLAC__uint64 *stream_length,
    void *client_data)
{
    FLACWaveDecoder *self = (FLACWaveDecoder *)client_data;
    
    if (!self->InputStream) {
        return FLAC__STREAM_DECODER_LENGTH_STATUS_ERROR;
    }
    
    *stream_length = self->InputStream->GetSize();
    return FLAC__STREAM_DECODER_LENGTH_STATUS_OK;
}

// EOF 检测回调
FLAC__bool flac_eof_callback(
    const FLAC__StreamDecoder *decoder,
    void *client_data)
{
    FLACWaveDecoder *self = (FLACWaveDecoder *)client_data;
    
    if (!self->InputStream) return true;
    
    return self->InputStream->GetPosition() >= 
           self->InputStream->GetSize();
}
```

</details>

### 题目 3：优化立体声交织

当前的 `_CopySamples2` 每次复制一个采样，请优化为批量复制：

<details>
<summary>查看答案</summary>

```cpp
#include <cstring>
#include <cstdint>

// 原始实现：逐采样复制
template <typename T>
static unsigned char *_CopySamples2_Original(
    unsigned char *dst, AVFrame *frame,
    int samples, int buf_index)
{
    int buf_pos = buf_index * sizeof(T);
    T *pDst = (T *)dst;
    
    for (int i = 0; i < samples; ++i, buf_pos += sizeof(T)) {
        *pDst++ = *(T *)(frame->data[0] + buf_pos);
        *pDst++ = *(T *)(frame->data[1] + buf_pos);
    }
    
    return (unsigned char *)pDst;
}

// 优化实现 1：使用 SIMD（需要 SSE/NEON）
#ifdef __SSE2__
#include <emmintrin.h>

// 专门针对 16-bit 立体声优化
static unsigned char *_CopySamples2_SSE2_S16(
    unsigned char *dst, AVFrame *frame,
    int samples, int buf_index)
{
    int16_t *left = (int16_t *)(frame->data[0] + buf_index * 2);
    int16_t *right = (int16_t *)(frame->data[1] + buf_index * 2);
    int16_t *out = (int16_t *)dst;
    
    // 一次处理 8 个采样
    int simd_samples = samples / 8;
    int remainder = samples % 8;
    
    for (int i = 0; i < simd_samples; ++i) {
        // 加载 8 个左声道采样
        __m128i l = _mm_loadu_si128((__m128i *)left);
        // 加载 8 个右声道采样
        __m128i r = _mm_loadu_si128((__m128i *)right);
        
        // 交织：L0R0L1R1...
        __m128i lo = _mm_unpacklo_epi16(l, r);  // L0R0L1R1L2R2L3R3
        __m128i hi = _mm_unpackhi_epi16(l, r);  // L4R4L5R5L6R6L7R7
        
        // 存储
        _mm_storeu_si128((__m128i *)out, lo);
        _mm_storeu_si128((__m128i *)(out + 8), hi);
        
        left += 8;
        right += 8;
        out += 16;
    }
    
    // 处理剩余
    for (int i = 0; i < remainder; ++i) {
        *out++ = *left++;
        *out++ = *right++;
    }
    
    return (unsigned char *)out;
}
#endif

// 优化实现 2：减少指针计算（纯 C++）
template <typename T>
static unsigned char *_CopySamples2_Optimized(
    unsigned char *dst, AVFrame *frame,
    int samples, int buf_index)
{
    T *left = (T *)(frame->data[0]) + buf_index;
    T *right = (T *)(frame->data[1]) + buf_index;
    T *out = (T *)dst;
    
    // 4 路展开
    int unrolled = samples / 4;
    int remainder = samples % 4;
    
    for (int i = 0; i < unrolled; ++i) {
        out[0] = left[0];
        out[1] = right[0];
        out[2] = left[1];
        out[3] = right[1];
        out[4] = left[2];
        out[5] = right[2];
        out[6] = left[3];
        out[7] = right[3];
        
        left += 4;
        right += 4;
        out += 8;
    }
    
    for (int i = 0; i < remainder; ++i) {
        *out++ = *left++;
        *out++ = *right++;
    }
    
    return (unsigned char *)out;
}

// 测试
#include <iostream>
#include <chrono>
#include <vector>

int main() {
    const int SAMPLES = 1024 * 1024;  // 1M 采样
    const int ITERATIONS = 100;
    
    // 模拟 AVFrame
    struct MockFrame {
        int16_t *data[2];
        int channels = 2;
        int format = 1;  // AV_SAMPLE_FMT_S16P
    };
    
    std::vector<int16_t> left(SAMPLES), right(SAMPLES), output(SAMPLES * 2);
    
    // 填充测试数据
    for (int i = 0; i < SAMPLES; ++i) {
        left[i] = i & 0x7FFF;
        right[i] = ~i & 0x7FFF;
    }
    
    MockFrame frame;
    frame.data[0] = left.data();
    frame.data[1] = right.data();
    
    // 测试优化版本
    auto start = std::chrono::high_resolution_clock::now();
    for (int iter = 0; iter < ITERATIONS; ++iter) {
        _CopySamples2_Optimized<int16_t>(
            (unsigned char *)output.data(), 
            (AVFrame *)&frame, 
            SAMPLES, 
            0);
    }
    auto end = std::chrono::high_resolution_clock::now();
    
    auto duration = std::chrono::duration_cast<std::chrono::microseconds>(
        end - start).count();
    
    std::cout << "优化版本: " << duration / ITERATIONS << " μs/iter" << std::endl;
    std::cout << "吞吐量: " << (SAMPLES * ITERATIONS * 4.0 / duration) 
              << " MB/s" << std::endl;
    
    return 0;
}
```

</details>

---

## 下一步

[03-OpenAL混音器](../03-OpenAL混音器/01-iTVPSoundBuffer接口与OpenAL后端.md) — 学习 OpenAL 音频输出与混音实现
