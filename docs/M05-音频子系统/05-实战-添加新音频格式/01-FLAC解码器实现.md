# FLAC 解码器实现

> **所属模块：** M05-音频子系统
> **前置知识：** [02-解码器注册机制](../02-解码器注册机制/01-tTVPWaveDecoder接口详解.md)、[P06-FFmpeg音视频开发](../../P06-FFmpeg音视频开发/README.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 libFLAC 的流式解码器架构与回调机制
2. 实现完整的 `FlacWaveDecoder` 类，继承 `tTVPWaveDecoder` 接口
3. 正确处理 FLAC 元数据（采样率、通道数、位深度）到 `tTVPWaveFormat` 的映射
4. 实现基于回调的流式解码，支持 Seek 操作
5. 处理 8/16/24/32 位 FLAC 到 16 位或浮点 PCM 的转换

---

## 5.1 FLAC 格式与 libFLAC 简介

### 5.1.1 为什么选择 FLAC

FLAC（Free Lossless Audio Codec）是最流行的无损音频压缩格式：

| 特性 | FLAC | WAV | Vorbis |
|------|------|-----|--------|
| 压缩类型 | 无损 | 无压缩 | 有损 |
| 典型压缩率 | 50-70% | 100% | 10-15% |
| 音质 | 完美还原 | 完美还原 | 有损失 |
| 元数据支持 | 完善（Vorbis Comment） | 有限（INFO chunk） | 完善 |
| 解码复杂度 | 中等 | 极低 | 中等 |

KiriKiri2 游戏中常见高品质 BGM 采用 FLAC 格式存储，因此添加 FLAC 支持能显著提升兼容性。

### 5.1.2 libFLAC 架构概览

libFLAC 提供两种解码 API：

```
┌─────────────────────────────────────────────────────────────────┐
│                        libFLAC API                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────┐    ┌───────────────────────────────┐ │
│  │  Stream Decoder API   │    │      File Decoder API         │ │
│  │  (FLAC__stream_decoder)│    │  (封装了 Stream + 文件 I/O)    │ │
│  │                       │    │                               │ │
│  │  - 用户提供回调函数     │    │  - 直接传入文件路径           │ │
│  │  - 适合自定义流源      │    │  - 仅支持标准文件系统         │ │
│  │  - KrKr2 采用此方式    │    │  - 不适合虚拟文件系统         │ │
│  └───────────────────────┘    └───────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

由于 KiriKiri2 使用虚拟文件系统（XP3 归档），我们必须使用 **Stream Decoder API**，通过回调函数桥接 `tTJSBinaryStream`。

### 5.1.3 回调函数签名

Stream Decoder 需要实现以下回调：

```cpp
// 读取回调：从流中读取数据
typedef FLAC__StreamDecoderReadStatus (*FLAC__StreamDecoderReadCallback)(
    const FLAC__StreamDecoder *decoder,
    FLAC__byte buffer[],           // 输出缓冲区
    size_t *bytes,                 // 输入：期望字节数，输出：实际读取字节数
    void *client_data              // 用户数据指针
);

// Seek 回调：定位到指定位置
typedef FLAC__StreamDecoderSeekStatus (*FLAC__StreamDecoderSeekCallback)(
    const FLAC__StreamDecoder *decoder,
    FLAC__uint64 absolute_byte_offset,  // 目标位置（字节）
    void *client_data
);

// Tell 回调：返回当前位置
typedef FLAC__StreamDecoderTellStatus (*FLAC__StreamDecoderTellCallback)(
    const FLAC__StreamDecoder *decoder,
    FLAC__uint64 *absolute_byte_offset,  // 输出：当前位置
    void *client_data
);

// Length 回调：返回流总长度
typedef FLAC__StreamDecoderLengthStatus (*FLAC__StreamDecoderLengthCallback)(
    const FLAC__StreamDecoder *decoder,
    FLAC__uint64 *stream_length,  // 输出：总字节数
    void *client_data
);

// EOF 回调：检查是否到达末尾
typedef FLAC__bool (*FLAC__StreamDecoderEofCallback)(
    const FLAC__StreamDecoder *decoder,
    void *client_data
);

// 写入回调：接收解码后的 PCM 数据（最重要！）
typedef FLAC__StreamDecoderWriteStatus (*FLAC__StreamDecoderWriteCallback)(
    const FLAC__StreamDecoder *decoder,
    const FLAC__Frame *frame,      // 帧信息（采样数、通道数等）
    const FLAC__int32 *const buffer[],  // 解码后的样本（每通道一个数组）
    void *client_data
);

// 元数据回调：接收 STREAMINFO、VORBIS_COMMENT 等
typedef void (*FLAC__StreamDecoderMetadataCallback)(
    const FLAC__StreamDecoder *decoder,
    const FLAC__StreamMetadata *metadata,
    void *client_data
);

// 错误回调：处理解码错误
typedef void (*FLAC__StreamDecoderErrorCallback)(
    const FLAC__StreamDecoder *decoder,
    FLAC__StreamDecoderErrorStatus status,
    void *client_data
);
```

---

## 5.2 FlacWaveDecoder 类设计

### 5.2.1 类定义

```cpp
// FlacWaveDecoder.h
#pragma once

#include "WaveIntf.h"
#include <FLAC/stream_decoder.h>
#include <vector>

class FlacWaveDecoder : public tTVPWaveDecoder {
public:
    FlacWaveDecoder();
    ~FlacWaveDecoder() override;

    // tTVPWaveDecoder 接口实现
    void GetFormat(tTVPWaveFormat &format) override;
    bool Render(void *buf, tjs_uint bufsamplelen, tjs_uint &rendered) override;
    bool SetPosition(tjs_uint64 samplepos) override;
    bool DesiredFormat(const tTVPWaveFormat &format) override;

    // 初始化方法
    bool SetStream(const ttstr &storagename);

private:
    // FLAC 回调函数（静态成员）
    static FLAC__StreamDecoderReadStatus ReadCallback(
        const FLAC__StreamDecoder *decoder,
        FLAC__byte buffer[], size_t *bytes, void *client_data);

    static FLAC__StreamDecoderSeekStatus SeekCallback(
        const FLAC__StreamDecoder *decoder,
        FLAC__uint64 absolute_byte_offset, void *client_data);

    static FLAC__StreamDecoderTellStatus TellCallback(
        const FLAC__StreamDecoder *decoder,
        FLAC__uint64 *absolute_byte_offset, void *client_data);

    static FLAC__StreamDecoderLengthStatus LengthCallback(
        const FLAC__StreamDecoder *decoder,
        FLAC__uint64 *stream_length, void *client_data);

    static FLAC__bool EofCallback(
        const FLAC__StreamDecoder *decoder, void *client_data);

    static FLAC__StreamDecoderWriteStatus WriteCallback(
        const FLAC__StreamDecoder *decoder,
        const FLAC__Frame *frame,
        const FLAC__int32 *const buffer[], void *client_data);

    static void MetadataCallback(
        const FLAC__StreamDecoder *decoder,
        const FLAC__StreamMetadata *metadata, void *client_data);

    static void ErrorCallback(
        const FLAC__StreamDecoder *decoder,
        FLAC__StreamDecoderErrorStatus status, void *client_data);

private:
    FLAC__StreamDecoder *Decoder;       // FLAC 解码器实例
    tTJSBinaryStream *InputStream;      // 输入流
    tTVPWaveFormat Format;              // 输出格式

    // 解码缓冲区
    std::vector<tjs_int16> OutputBuffer; // 16 位 PCM 输出
    size_t BufferReadPos;                // 当前读取位置
    size_t BufferWritePos;               // 当前写入位置
    
    // 状态标志
    bool MetadataReceived;               // 是否已收到 STREAMINFO
    bool DecoderError;                   // 是否发生解码错误
    tjs_uint64 CurrentSamplePos;         // 当前样本位置
};
```

### 5.2.2 成员变量说明

| 成员 | 类型 | 作用 |
|------|------|------|
| `Decoder` | `FLAC__StreamDecoder*` | libFLAC 解码器句柄 |
| `InputStream` | `tTJSBinaryStream*` | KrKr2 虚拟文件流 |
| `Format` | `tTVPWaveFormat` | 输出音频格式描述 |
| `OutputBuffer` | `vector<tjs_int16>` | 解码后的 PCM 缓冲区 |
| `BufferReadPos/WritePos` | `size_t` | 环形缓冲区读写指针 |
| `MetadataReceived` | `bool` | 防止在元数据前调用 Render |
| `DecoderError` | `bool` | 错误状态标志 |
| `CurrentSamplePos` | `tjs_uint64` | 跟踪播放位置用于 Seek |

---

## 5.3 回调函数实现

### 5.3.1 读取回调

```cpp
FLAC__StreamDecoderReadStatus FlacWaveDecoder::ReadCallback(
    const FLAC__StreamDecoder *decoder,
    FLAC__byte buffer[], size_t *bytes, void *client_data) {
    
    FlacWaveDecoder *self = static_cast<FlacWaveDecoder*>(client_data);
    
    if(!self->InputStream) {
        *bytes = 0;
        return FLAC__STREAM_DECODER_READ_STATUS_ABORT;
    }
    
    // 尝试读取请求的字节数
    tjs_uint read = self->InputStream->Read(buffer, static_cast<tjs_uint>(*bytes));
    
    if(read == 0) {
        // 没有读到任何数据
        *bytes = 0;
        // 检查是否真的到达文件末尾
        tjs_uint64 pos = self->InputStream->GetPosition();
        tjs_uint64 size = self->InputStream->GetSize();
        if(pos >= size) {
            return FLAC__STREAM_DECODER_READ_STATUS_END_OF_STREAM;
        }
        // 可能是临时错误，返回 CONTINUE 让 libFLAC 重试
        return FLAC__STREAM_DECODER_READ_STATUS_CONTINUE;
    }
    
    *bytes = read;
    return FLAC__STREAM_DECODER_READ_STATUS_CONTINUE;
}
```

**关键点：**
- 返回 `END_OF_STREAM` 必须确认真的到达文件末尾
- 返回 `ABORT` 会立即终止解码
- `*bytes` 必须设置为实际读取的字节数

### 5.3.2 Seek/Tell/Length/EOF 回调

```cpp
FLAC__StreamDecoderSeekStatus FlacWaveDecoder::SeekCallback(
    const FLAC__StreamDecoder *decoder,
    FLAC__uint64 absolute_byte_offset, void *client_data) {
    
    FlacWaveDecoder *self = static_cast<FlacWaveDecoder*>(client_data);
    
    if(!self->InputStream) {
        return FLAC__STREAM_DECODER_SEEK_STATUS_ERROR;
    }
    
    // 使用 KrKr2 流的 Seek 方法
    tjs_uint64 result = self->InputStream->Seek(
        static_cast<tjs_int64>(absolute_byte_offset), TJS_BS_SEEK_SET);
    
    if(result == absolute_byte_offset) {
        return FLAC__STREAM_DECODER_SEEK_STATUS_OK;
    }
    
    return FLAC__STREAM_DECODER_SEEK_STATUS_ERROR;
}

FLAC__StreamDecoderTellStatus FlacWaveDecoder::TellCallback(
    const FLAC__StreamDecoder *decoder,
    FLAC__uint64 *absolute_byte_offset, void *client_data) {
    
    FlacWaveDecoder *self = static_cast<FlacWaveDecoder*>(client_data);
    
    if(!self->InputStream) {
        return FLAC__STREAM_DECODER_TELL_STATUS_ERROR;
    }
    
    *absolute_byte_offset = self->InputStream->GetPosition();
    return FLAC__STREAM_DECODER_TELL_STATUS_OK;
}

FLAC__StreamDecoderLengthStatus FlacWaveDecoder::LengthCallback(
    const FLAC__StreamDecoder *decoder,
    FLAC__uint64 *stream_length, void *client_data) {
    
    FlacWaveDecoder *self = static_cast<FlacWaveDecoder*>(client_data);
    
    if(!self->InputStream) {
        return FLAC__STREAM_DECODER_LENGTH_STATUS_ERROR;
    }
    
    *stream_length = self->InputStream->GetSize();
    return FLAC__STREAM_DECODER_LENGTH_STATUS_OK;
}

FLAC__bool FlacWaveDecoder::EofCallback(
    const FLAC__StreamDecoder *decoder, void *client_data) {
    
    FlacWaveDecoder *self = static_cast<FlacWaveDecoder*>(client_data);
    
    if(!self->InputStream) {
        return true;  // 没有流视为已结束
    }
    
    tjs_uint64 pos = self->InputStream->GetPosition();
    tjs_uint64 size = self->InputStream->GetSize();
    return pos >= size;
}
```

### 5.3.3 元数据回调（最重要）

```cpp
void FlacWaveDecoder::MetadataCallback(
    const FLAC__StreamDecoder *decoder,
    const FLAC__StreamMetadata *metadata, void *client_data) {
    
    FlacWaveDecoder *self = static_cast<FlacWaveDecoder*>(client_data);
    
    // 只处理 STREAMINFO 块（包含音频格式信息）
    if(metadata->type == FLAC__METADATA_TYPE_STREAMINFO) {
        const FLAC__StreamMetadata_StreamInfo &info = metadata->data.stream_info;
        
        // 填充 tTVPWaveFormat 结构
        self->Format.SamplesPerSec = info.sample_rate;
        self->Format.Channels = info.channels;
        self->Format.BitsPerSample = info.bits_per_sample;
        
        // 计算每样本字节数（向上取整到字节边界）
        self->Format.BytesPerSample = (info.bits_per_sample + 7) / 8;
        
        // FLAC 总是整数 PCM，不是浮点
        self->Format.IsFloat = false;
        
        // 总样本数和时长
        self->Format.TotalSamples = info.total_samples;
        if(info.sample_rate > 0) {
            self->Format.TotalTime = 
                (info.total_samples * 1000) / info.sample_rate;
        } else {
            self->Format.TotalTime = 0;
        }
        
        // FLAC 流通常可 Seek（除非是纯流式输入）
        self->Format.Seekable = true;
        
        // 默认无特殊扬声器配置
        self->Format.SpeakerConfig = 0;
        
        // 预分配输出缓冲区（至少容纳一帧，FLAC 最大 blocksize=65535）
        size_t bufferSize = info.max_blocksize * info.channels;
        self->OutputBuffer.resize(bufferSize);
        
        self->MetadataReceived = true;
        
        TVPAddLog(TJS_W("FLAC: STREAMINFO received - ") +
                  ttstr(info.sample_rate) + TJS_W("Hz, ") +
                  ttstr(info.channels) + TJS_W("ch, ") +
                  ttstr(info.bits_per_sample) + TJS_W("bit"));
    }
}
```

**STREAMINFO 块结构：**

```
STREAMINFO 元数据块（必须是第一个块）
├── min_blocksize      : 最小块大小（16-65535）
├── max_blocksize      : 最大块大小
├── min_framesize      : 最小帧字节数（0=未知）
├── max_framesize      : 最大帧字节数
├── sample_rate        : 采样率（1-655350 Hz）
├── channels           : 通道数（1-8）
├── bits_per_sample    : 位深度（4-32）
├── total_samples      : 总样本数（0=未知）
└── md5sum[16]         : 原始音频 MD5 校验
```

### 5.3.4 写入回调（解码核心）

```cpp
FLAC__StreamDecoderWriteStatus FlacWaveDecoder::WriteCallback(
    const FLAC__StreamDecoder *decoder,
    const FLAC__Frame *frame,
    const FLAC__int32 *const buffer[], void *client_data) {
    
    FlacWaveDecoder *self = static_cast<FlacWaveDecoder*>(client_data);
    
    // 确保元数据已接收
    if(!self->MetadataReceived) {
        return FLAC__STREAM_DECODER_WRITE_STATUS_ABORT;
    }
    
    const unsigned int channels = frame->header.channels;
    const unsigned int blocksize = frame->header.blocksize;
    const unsigned int bps = frame->header.bits_per_sample;
    
    // 确保缓冲区足够大
    size_t requiredSize = (self->BufferWritePos + blocksize) * channels;
    if(self->OutputBuffer.size() < requiredSize) {
        self->OutputBuffer.resize(requiredSize);
    }
    
    // 将 FLAC 的 int32 样本转换为 16 位 PCM
    // FLAC 解码出的样本是左对齐的，需要根据位深度调整
    tjs_int16 *out = self->OutputBuffer.data() + 
                     self->BufferWritePos * channels;
    
    for(unsigned int i = 0; i < blocksize; i++) {
        for(unsigned int ch = 0; ch < channels; ch++) {
            FLAC__int32 sample = buffer[ch][i];
            
            // 根据位深度转换到 16 位
            tjs_int16 sample16;
            if(bps == 16) {
                // 16 位直接复制
                sample16 = static_cast<tjs_int16>(sample);
            } else if(bps == 24) {
                // 24 位右移 8 位
                sample16 = static_cast<tjs_int16>(sample >> 8);
            } else if(bps == 8) {
                // 8 位左移 8 位
                sample16 = static_cast<tjs_int16>(sample << 8);
            } else if(bps == 32) {
                // 32 位右移 16 位
                sample16 = static_cast<tjs_int16>(sample >> 16);
            } else {
                // 通用转换：归一化到 16 位
                int shift = bps - 16;
                if(shift > 0) {
                    sample16 = static_cast<tjs_int16>(sample >> shift);
                } else {
                    sample16 = static_cast<tjs_int16>(sample << (-shift));
                }
            }
            
            *out++ = sample16;
        }
    }
    
    self->BufferWritePos += blocksize;
    
    return FLAC__STREAM_DECODER_WRITE_STATUS_CONTINUE;
}
```

**位深度转换原理：**

```
FLAC 样本是"左对齐"的，即有效位在高位：

8 位样本：  [XXXXXXXX] 00000000 00000000 00000000  (存储在低 8 位)
16 位样本：[XXXXXXXX XXXXXXXX] 00000000 00000000  (存储在低 16 位)
24 位样本：[XXXXXXXX XXXXXXXX XXXXXXXX] 00000000  (存储在低 24 位)
32 位样本：[XXXXXXXX XXXXXXXX XXXXXXXX XXXXXXXX]

转换到 16 位时：
- 8 位 → 左移 8 位，填充低位
- 16 位 → 直接使用
- 24 位 → 右移 8 位，截断低位
- 32 位 → 右移 16 位，截断低位
```

### 5.3.5 错误回调

```cpp
void FlacWaveDecoder::ErrorCallback(
    const FLAC__StreamDecoder *decoder,
    FLAC__StreamDecoderErrorStatus status, void *client_data) {
    
    FlacWaveDecoder *self = static_cast<FlacWaveDecoder*>(client_data);
    self->DecoderError = true;
    
    const char *errorStr = "Unknown error";
    switch(status) {
        case FLAC__STREAM_DECODER_ERROR_STATUS_LOST_SYNC:
            errorStr = "Lost sync";
            break;
        case FLAC__STREAM_DECODER_ERROR_STATUS_BAD_HEADER:
            errorStr = "Bad header";
            break;
        case FLAC__STREAM_DECODER_ERROR_STATUS_FRAME_CRC_MISMATCH:
            errorStr = "Frame CRC mismatch";
            break;
        case FLAC__STREAM_DECODER_ERROR_STATUS_UNPARSEABLE_STREAM:
            errorStr = "Unparseable stream";
            break;
    }
    
    TVPAddLog(ttstr(TJS_W("FLAC decode error: ")) + ttstr(errorStr));
}
```

---

## 5.4 核心方法实现

### 5.4.1 构造与析构

```cpp
FlacWaveDecoder::FlacWaveDecoder()
    : Decoder(nullptr)
    , InputStream(nullptr)
    , BufferReadPos(0)
    , BufferWritePos(0)
    , MetadataReceived(false)
    , DecoderError(false)
    , CurrentSamplePos(0) {
    
    memset(&Format, 0, sizeof(Format));
}

FlacWaveDecoder::~FlacWaveDecoder() {
    if(Decoder) {
        FLAC__stream_decoder_finish(Decoder);
        FLAC__stream_decoder_delete(Decoder);
        Decoder = nullptr;
    }
    
    if(InputStream) {
        delete InputStream;
        InputStream = nullptr;
    }
}
```

### 5.4.2 流初始化

```cpp
bool FlacWaveDecoder::SetStream(const ttstr &storagename) {
    // 创建 KrKr2 流
    InputStream = TVPCreateStream(storagename, TJS_BS_READ);
    if(!InputStream) {
        return false;
    }
    
    // 创建 FLAC 解码器
    Decoder = FLAC__stream_decoder_new();
    if(!Decoder) {
        delete InputStream;
        InputStream = nullptr;
        return false;
    }
    
    // 设置元数据响应（我们只需要 STREAMINFO）
    FLAC__stream_decoder_set_metadata_respond(
        Decoder, FLAC__METADATA_TYPE_STREAMINFO);
    
    // 初始化流解码器
    FLAC__StreamDecoderInitStatus initStatus = 
        FLAC__stream_decoder_init_stream(
            Decoder,
            ReadCallback,
            SeekCallback,
            TellCallback,
            LengthCallback,
            EofCallback,
            WriteCallback,
            MetadataCallback,
            ErrorCallback,
            this  // client_data 传递 this 指针
        );
    
    if(initStatus != FLAC__STREAM_DECODER_INIT_STATUS_OK) {
        TVPAddLog(TJS_W("FLAC: Failed to init decoder, status=") +
                  ttstr(static_cast<int>(initStatus)));
        FLAC__stream_decoder_delete(Decoder);
        Decoder = nullptr;
        delete InputStream;
        InputStream = nullptr;
        return false;
    }
    
    // 处理元数据（这会触发 MetadataCallback）
    if(!FLAC__stream_decoder_process_until_end_of_metadata(Decoder)) {
        TVPAddLog(TJS_W("FLAC: Failed to process metadata"));
        FLAC__stream_decoder_finish(Decoder);
        FLAC__stream_decoder_delete(Decoder);
        Decoder = nullptr;
        delete InputStream;
        InputStream = nullptr;
        return false;
    }
    
    // 确认元数据已接收
    if(!MetadataReceived) {
        TVPAddLog(TJS_W("FLAC: No STREAMINFO found"));
        FLAC__stream_decoder_finish(Decoder);
        FLAC__stream_decoder_delete(Decoder);
        Decoder = nullptr;
        delete InputStream;
        InputStream = nullptr;
        return false;
    }
    
    return true;
}
```

### 5.4.3 GetFormat 实现

```cpp
void FlacWaveDecoder::GetFormat(tTVPWaveFormat &format) {
    format = Format;
}
```

### 5.4.4 Render 实现（核心解码循环）

```cpp
bool FlacWaveDecoder::Render(void *buf, tjs_uint bufsamplelen, 
                              tjs_uint &rendered) {
    if(!Decoder || !MetadataReceived || DecoderError) {
        rendered = 0;
        return false;
    }
    
    tjs_int16 *output = static_cast<tjs_int16*>(buf);
    tjs_uint samplesWritten = 0;
    
    while(samplesWritten < bufsamplelen) {
        // 1. 先从缓冲区读取已解码的样本
        if(BufferReadPos < BufferWritePos) {
            size_t availableSamples = BufferWritePos - BufferReadPos;
            size_t samplesToRead = std::min(
                availableSamples, 
                static_cast<size_t>(bufsamplelen - samplesWritten));
            
            size_t srcOffset = BufferReadPos * Format.Channels;
            size_t dstOffset = samplesWritten * Format.Channels;
            size_t copyCount = samplesToRead * Format.Channels;
            
            memcpy(output + dstOffset, 
                   OutputBuffer.data() + srcOffset,
                   copyCount * sizeof(tjs_int16));
            
            BufferReadPos += samplesToRead;
            samplesWritten += static_cast<tjs_uint>(samplesToRead);
            CurrentSamplePos += samplesToRead;
            
            continue;
        }
        
        // 2. 缓冲区已空，重置并解码下一帧
        BufferReadPos = 0;
        BufferWritePos = 0;
        
        // 解码单帧
        FLAC__bool ok = FLAC__stream_decoder_process_single(Decoder);
        
        if(!ok || DecoderError) {
            // 解码错误
            break;
        }
        
        // 检查解码器状态
        FLAC__StreamDecoderState state = 
            FLAC__stream_decoder_get_state(Decoder);
        
        if(state == FLAC__STREAM_DECODER_END_OF_STREAM) {
            // 文件结束
            break;
        }
        
        if(state == FLAC__STREAM_DECODER_ABORTED ||
           state == FLAC__STREAM_DECODER_MEMORY_ALLOCATION_ERROR) {
            // 严重错误
            DecoderError = true;
            break;
        }
        
        // 如果没有解码出任何样本，可能是流有问题
        if(BufferWritePos == 0) {
            // 尝试继续
            continue;
        }
    }
    
    rendered = samplesWritten;
    return samplesWritten >= bufsamplelen;
}
```

### 5.4.5 SetPosition 实现（Seek）

```cpp
bool FlacWaveDecoder::SetPosition(tjs_uint64 samplepos) {
    if(!Decoder || !MetadataReceived) {
        return false;
    }
    
    // 检查范围
    if(samplepos >= Format.TotalSamples && Format.TotalSamples > 0) {
        return false;
    }
    
    // 清空解码缓冲区
    BufferReadPos = 0;
    BufferWritePos = 0;
    DecoderError = false;
    
    // 执行样本级 Seek
    FLAC__bool ok = FLAC__stream_decoder_seek_absolute(Decoder, samplepos);
    
    if(!ok) {
        // Seek 失败，检查状态
        FLAC__StreamDecoderState state = 
            FLAC__stream_decoder_get_state(Decoder);
        
        if(state == FLAC__STREAM_DECODER_SEEK_ERROR) {
            // 尝试重置解码器并重新 Seek
            FLAC__stream_decoder_flush(Decoder);
            ok = FLAC__stream_decoder_seek_absolute(Decoder, samplepos);
        }
        
        if(!ok) {
            TVPAddLog(TJS_W("FLAC: Seek failed to sample ") + 
                      ttstr(static_cast<tjs_int64>(samplepos)));
            return false;
        }
    }
    
    CurrentSamplePos = samplepos;
    return true;
}
```

### 5.4.6 DesiredFormat 实现

```cpp
bool FlacWaveDecoder::DesiredFormat(const tTVPWaveFormat &format) {
    // FLAC 解码器输出固定 16 位整数 PCM
    // 如果请求浮点格式，返回 false 让上层做转换
    return false;
}
```

---

## 5.5 完整实现代码

### FlacWaveDecoder.h

```cpp
// cpp/core/sound/FlacWaveDecoder.h
#pragma once

#include "WaveIntf.h"
#include <FLAC/stream_decoder.h>
#include <vector>

class FlacWaveDecoder : public tTVPWaveDecoder {
public:
    FlacWaveDecoder();
    ~FlacWaveDecoder() override;

    void GetFormat(tTVPWaveFormat &format) override;
    bool Render(void *buf, tjs_uint bufsamplelen, tjs_uint &rendered) override;
    bool SetPosition(tjs_uint64 samplepos) override;
    bool DesiredFormat(const tTVPWaveFormat &format) override;

    bool SetStream(const ttstr &storagename);

private:
    static FLAC__StreamDecoderReadStatus ReadCallback(
        const FLAC__StreamDecoder*, FLAC__byte[], size_t*, void*);
    static FLAC__StreamDecoderSeekStatus SeekCallback(
        const FLAC__StreamDecoder*, FLAC__uint64, void*);
    static FLAC__StreamDecoderTellStatus TellCallback(
        const FLAC__StreamDecoder*, FLAC__uint64*, void*);
    static FLAC__StreamDecoderLengthStatus LengthCallback(
        const FLAC__StreamDecoder*, FLAC__uint64*, void*);
    static FLAC__bool EofCallback(const FLAC__StreamDecoder*, void*);
    static FLAC__StreamDecoderWriteStatus WriteCallback(
        const FLAC__StreamDecoder*, const FLAC__Frame*,
        const FLAC__int32 *const[], void*);
    static void MetadataCallback(
        const FLAC__StreamDecoder*, const FLAC__StreamMetadata*, void*);
    static void ErrorCallback(
        const FLAC__StreamDecoder*, FLAC__StreamDecoderErrorStatus, void*);

private:
    FLAC__StreamDecoder *Decoder;
    tTJSBinaryStream *InputStream;
    tTVPWaveFormat Format;
    std::vector<tjs_int16> OutputBuffer;
    size_t BufferReadPos;
    size_t BufferWritePos;
    bool MetadataReceived;
    bool DecoderError;
    tjs_uint64 CurrentSamplePos;
};

class FlacWaveDecoderCreator : public tTVPWaveDecoderCreator {
public:
    tTVPWaveDecoder *Create(const ttstr &storagename,
                            const ttstr &extension) override;
};
```

### FlacWaveDecoder.cpp（Creator 部分）

```cpp
// cpp/core/sound/FlacWaveDecoder.cpp
#include "FlacWaveDecoder.h"
#include "StorageIntf.h"
#include "DebugIntf.h"

// ... [上述所有方法实现] ...

// Creator 实现
tTVPWaveDecoder *FlacWaveDecoderCreator::Create(
    const ttstr &storagename, const ttstr &extension) {
    
    static const ttstr flac_ext = TJS_W(".flac");
    static const ttstr fla_ext = TJS_W(".fla");
    
    // 只处理 .flac 和 .fla 扩展名
    if(extension != flac_ext && extension != fla_ext) {
        return nullptr;
    }
    
    FlacWaveDecoder *decoder = new FlacWaveDecoder();
    if(!decoder->SetStream(storagename)) {
        delete decoder;
        return nullptr;
    }
    
    return decoder;
}
```

---

## 动手实践

### 实践 1：编译测试 FLAC 解码器

1. **添加 vcpkg 依赖**：

```json
// vcpkg.json
{
  "dependencies": [
    "flac"
  ]
}
```

2. **创建测试程序**：

```cpp
// tests/unit-tests/core/sound/test_flac_decoder.cpp
#include <catch2/catch_all.hpp>
#include "FlacWaveDecoder.h"

TEST_CASE("FlacWaveDecoder basic test", "[sound][flac]") {
    FlacWaveDecoderCreator creator;
    
    // 需要一个测试 FLAC 文件
    ttstr testFile = TJS_W("test.flac");
    ttstr ext = TJS_W(".flac");
    
    tTVPWaveDecoder *decoder = creator.Create(testFile, ext);
    
    if(decoder) {
        tTVPWaveFormat format;
        decoder->GetFormat(format);
        
        REQUIRE(format.SamplesPerSec > 0);
        REQUIRE(format.Channels >= 1);
        REQUIRE(format.BitsPerSample >= 8);
        
        // 尝试解码一帧
        std::vector<tjs_int16> buffer(4096);
        tjs_uint rendered = 0;
        decoder->Render(buffer.data(), 1024, rendered);
        
        REQUIRE(rendered > 0);
        
        delete decoder;
    }
}
```

### 实践 2：位深度转换验证

```cpp
// 验证不同位深度转换的正确性
void TestBitDepthConversion() {
    // 24 位最大值应该转换为 16 位最大值
    FLAC__int32 sample24_max = 0x7FFFFF;  // 24 位有符号最大值
    tjs_int16 result16 = static_cast<tjs_int16>(sample24_max >> 8);
    assert(result16 == 0x7FFF);  // 应该得到 16 位最大值
    
    // 8 位半幅值应该扩展正确
    FLAC__int32 sample8_half = 64;  // 8 位有符号中间值
    tjs_int16 result8to16 = static_cast<tjs_int16>(sample8_half << 8);
    assert(result8to16 == 0x4000);  // 应该是 16384
}
```

---

## 对照项目源码

相关参考文件：

- `cpp/core/sound/VorbisWaveDecoder.cpp` 第 29-130 行 — Vorbis 解码器回调实现模式
- `cpp/core/sound/WaveIntf.cpp` 第 97-140 行 — PCM 格式转换函数
- `cpp/core/sound/WaveIntf.h` 第 45-55 行 — `tTVPWaveFormat` 结构定义
- `cpp/core/sound/VorbisWaveDecoder.cpp` 第 211-312 行 — `SetStream` 初始化模式

项目中 `VorbisWaveDecoder` 的回调桥接模式可直接复用于 FLAC：

```cpp
// VorbisWaveDecoder 中的回调模式
static size_t read_func(void *ptr, size_t size, size_t nmemb, void *datasource) {
    VorbisWaveDecoder *decoder = (VorbisWaveDecoder *)datasource;
    return decoder->InputStream->Read(ptr, size * nmemb) / size;
}
```

对比 FLAC 版本：

```cpp
// FLAC 回调使用相同模式
static FLAC__StreamDecoderReadStatus ReadCallback(..., void *client_data) {
    FlacWaveDecoder *self = static_cast<FlacWaveDecoder*>(client_data);
    tjs_uint read = self->InputStream->Read(buffer, *bytes);
    // ...
}
```

---

## 本节小结

- libFLAC 使用 **回调式流解码器**，与 KrKr2 虚拟文件系统完美适配
- 必须实现 7 个回调函数：Read、Seek、Tell、Length、EOF、Write、Metadata、Error
- **元数据回调** 中获取 STREAMINFO 并填充 `tTVPWaveFormat`
- **写入回调** 是解码核心，负责将 `FLAC__int32` 样本转换为 16 位 PCM
- 位深度转换遵循 **右移高位截断** 原则（24→16 右移 8 位）
- Seek 操作使用 `FLAC__stream_decoder_seek_absolute`，失败时需 flush 重试

---

## 练习题与答案

### 题目 1：FLAC 解码器为什么使用 Stream Decoder API 而不是 File Decoder API？

<details>
<summary>查看答案</summary>

KiriKiri2 使用虚拟文件系统（XP3 归档），文件并非直接存在于操作系统文件系统中。File Decoder API 只接受文件路径，无法处理虚拟流。Stream Decoder API 允许我们通过回调函数桥接任意数据源（包括 `tTJSBinaryStream`），因此必须使用 Stream Decoder API。

关键代码：
```cpp
// Stream Decoder 允许自定义所有 I/O 操作
FLAC__stream_decoder_init_stream(
    Decoder,
    ReadCallback,   // 自定义读取
    SeekCallback,   // 自定义 Seek
    TellCallback,   // 自定义 Tell
    // ...
);
```

</details>

### 题目 2：将 24 位 FLAC 样本 `0x7FABCD` 转换为 16 位 PCM，结果是多少？写出计算过程。

<details>
<summary>查看答案</summary>

24 位到 16 位转换是右移 8 位：

```
原始值：0x7FABCD (24 位有符号整数)
       = 0111 1111 1010 1011 1100 1101 (二进制)

右移 8 位：
       0x7FABCD >> 8 
     = 0x007FAB
     = 0111 1111 1010 1011 (保留高 16 位)

结果：0x7FAB = 32683 (16 位有符号整数)
```

代码实现：
```cpp
FLAC__int32 sample24 = 0x7FABCD;
tjs_int16 sample16 = static_cast<tjs_int16>(sample24 >> 8);
// sample16 = 0x7FAB = 32683
```

注意：这里丢失了低 8 位 `0xCD`，这是无损转有损的必然代价。

</details>

### 题目 3：如果 `FLAC__stream_decoder_seek_absolute` 返回失败，应该如何处理？

<details>
<summary>查看答案</summary>

Seek 失败时应按以下步骤处理：

1. **检查解码器状态**，确认是 `FLAC__STREAM_DECODER_SEEK_ERROR`
2. **调用 `FLAC__stream_decoder_flush`** 清空内部缓冲区
3. **重试 Seek 操作**
4. **如果仍然失败，记录日志并返回 false**

完整代码：
```cpp
bool FlacWaveDecoder::SetPosition(tjs_uint64 samplepos) {
    // 清空解码缓冲区
    BufferReadPos = 0;
    BufferWritePos = 0;
    
    FLAC__bool ok = FLAC__stream_decoder_seek_absolute(Decoder, samplepos);
    
    if(!ok) {
        FLAC__StreamDecoderState state = 
            FLAC__stream_decoder_get_state(Decoder);
        
        if(state == FLAC__STREAM_DECODER_SEEK_ERROR) {
            // 步骤 2：Flush 解码器
            FLAC__stream_decoder_flush(Decoder);
            // 步骤 3：重试
            ok = FLAC__stream_decoder_seek_absolute(Decoder, samplepos);
        }
        
        if(!ok) {
            // 步骤 4：记录日志，返回失败
            TVPAddLog(TJS_W("FLAC: Seek failed"));
            return false;
        }
    }
    
    CurrentSamplePos = samplepos;
    return true;
}
```

</details>

---

## 下一步

[02-解码器注册与测试](./02-解码器注册与测试.md) — 学习如何将 FLAC 解码器注册到 KrKr2 音频系统，编写 CMake 集成和单元测试。
