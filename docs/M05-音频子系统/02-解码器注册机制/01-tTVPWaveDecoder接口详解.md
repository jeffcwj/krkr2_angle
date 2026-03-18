# tTVPWaveDecoder 接口详解

> **所属模块：** M05-音频子系统  
> **前置知识：** [01-sound模块架构](../01-sound模块架构/README.md)、C++ 虚函数与多态  
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 `tTVPWaveDecoder` 抽象接口的设计意图和职责边界
2. 掌握 `tTVPWaveFormat` 结构体各字段的含义和取值范围
3. 实现符合接口规范的自定义解码器
4. 正确处理 PCM 数据的渲染与定位操作
5. 理解解码器与播放器之间的数据交互协议

---

## tTVPWaveDecoder 接口概述

### 设计理念

KiriKiri2 音频系统采用**策略模式（Strategy Pattern）**设计解码器接口。`tTVPWaveDecoder` 是所有音频解码器的抽象基类，定义了解码器必须实现的核心操作：

```
┌─────────────────────────────────────────────────────────────┐
│                    tTVPWaveDecoder (抽象)                    │
├─────────────────────────────────────────────────────────────┤
│  + GetFormat(format&)      : void     // 获取音频格式        │
│  + Render(buf, len, &out)  : bool     // 渲染 PCM 数据       │
│  + SetPosition(pos)        : bool     // 定位到指定采样位置   │
│  + DesiredFormat(format&)  : bool     // 请求输出格式(可选)   │
├─────────────────────────────────────────────────────────────┤
│  <<interface>> 职责：                                        │
│  1. 打开并解析音频文件头                                      │
│  2. 按需解码 PCM 数据块                                       │
│  3. 支持随机定位（如果格式支持）                              │
│  4. 报告音频格式信息                                          │
└─────────────────────────────────────────────────────────────┘
           ▲               ▲                ▲
           │               │                │
   ┌───────┴───┐   ┌───────┴───┐   ┌────────┴────┐
   │tTVPWD_    │   │VorbisWave │   │ FFWave      │
   │RIFFWave   │   │Decoder    │   │ Decoder     │
   └───────────┘   └───────────┘   └─────────────┘
```

### 接口定义（源码）

以下是 `WaveIntf.h` 中的接口定义：

```cpp
// 文件: cpp/core/sound/WaveIntf.h 第 84-113 行
class tTVPWaveDecoder {
public:
    virtual ~tTVPWaveDecoder() = default;

    // 获取音频格式信息
    virtual void GetFormat(tTVPWaveFormat &format) = 0;

    // 渲染 PCM 数据到缓冲区
    // 参数:
    //   buf          - 目标缓冲区
    //   bufsamplelen - 缓冲区大小（以采样粒度为单位）
    //   rendered     - 输出：实际渲染的采样数
    // 返回值:
    //   true  - 还有更多数据可解码
    //   false - 已到达文件末尾
    virtual bool Render(void *buf, tjs_uint bufsamplelen,
                        tjs_uint &rendered) = 0;

    // 定位到指定采样位置
    // 参数:
    //   samplepos - 目标位置（以采样粒度为单位）
    // 返回值:
    //   true  - 定位成功
    //   false - 定位失败（格式不支持或位置无效）
    virtual bool SetPosition(tjs_uint64 samplepos) = 0;

    // 请求输出格式（可选实现）
    // 某些解码器可按请求格式输出（如重采样）
    virtual bool DesiredFormat(const tTVPWaveFormat &format) { 
        return false;  // 默认不支持
    }
};
```

### 采样粒度（Sample Granule）概念

理解"采样粒度"是使用此接口的关键。在 KiriKiri2 中：

```
采样粒度 = 一个时间点上所有声道的采样值集合

例如: 立体声 16-bit 音频
┌───────────────────────────────────────────────────┐
│ 时间点 0      │ 时间点 1      │ 时间点 2      │...│
├───────────────┼───────────────┼───────────────┤   │
│ L0 (2B) R0(2B)│ L1 (2B) R1(2B)│ L2 (2B) R2(2B)│   │
└───────────────┴───────────────┴───────────────┴───┘
     1个粒度          1个粒度         1个粒度

每个粒度大小 = Channels × BytesPerSample = 2 × 2 = 4 字节
```

这意味着：
- `Render(buf, 1024, rendered)` 请求 1024 个粒度，不是 1024 字节
- `SetPosition(44100)` 定位到第 44100 个粒度，即 1 秒位置（44.1kHz 时）

---

## tTVPWaveFormat 结构体详解

### 结构体定义

```cpp
// 文件: cpp/core/sound/WaveIntf.h 第 45-55 行
struct tTVPWaveFormat {
    tjs_uint SamplesPerSec;   // 采样率（每秒采样粒度数）
    tjs_uint Channels;         // 声道数
    tjs_uint BitsPerSample;    // 每采样有效位数
    tjs_uint BytesPerSample;   // 每采样字节数（含填充）
    tjs_uint64 TotalSamples;   // 总采样粒度数（0 表示未知）
    tjs_uint64 TotalTime;      // 总时长（毫秒，0 表示未知）
    tjs_uint32 SpeakerConfig;  // 扬声器配置（位掩码）
    bool IsFloat;              // 是否为 IEEE 浮点格式
    bool Seekable;             // 是否支持随机定位
};
```

### 字段详解与取值范围

#### 1. SamplesPerSec（采样率）

```cpp
// 常见取值
tjs_uint SamplesPerSec;

// 标准采样率
const tjs_uint RATE_8000   = 8000;    // 电话质量
const tjs_uint RATE_11025  = 11025;   // 低质量语音
const tjs_uint RATE_22050  = 22050;   // FM 广播
const tjs_uint RATE_44100  = 44100;   // CD 质量（最常见）
const tjs_uint RATE_48000  = 48000;   // DVD/专业音频
const tjs_uint RATE_96000  = 96000;   // 高清音频
const tjs_uint RATE_192000 = 192000;  // 超高清音频

// 验证示例
bool IsValidSampleRate(tjs_uint rate) {
    // KiriKiri2 支持 1Hz ~ 192kHz
    return rate >= 1 && rate <= 192000;
}
```

#### 2. Channels（声道数）

```cpp
tjs_uint Channels;

// 常见配置
// 1 = 单声道（Mono）
// 2 = 立体声（Stereo）
// 4 = 四声道（Quadraphonic）
// 6 = 5.1 环绕声
// 8 = 7.1 环绕声

// 扬声器配置常量（来自 Windows ksmedia.h）
#define SPEAKER_FRONT_LEFT            0x1
#define SPEAKER_FRONT_RIGHT           0x2
#define SPEAKER_FRONT_CENTER          0x4
#define SPEAKER_LOW_FREQUENCY         0x8    // 低音炮（LFE）
#define SPEAKER_BACK_LEFT             0x10
#define SPEAKER_BACK_RIGHT            0x20
#define SPEAKER_SIDE_LEFT             0x200
#define SPEAKER_SIDE_RIGHT            0x400

// 5.1 声道配置示例
tjs_uint32 speaker51 = SPEAKER_FRONT_LEFT | SPEAKER_FRONT_RIGHT |
                       SPEAKER_FRONT_CENTER | SPEAKER_LOW_FREQUENCY |
                       SPEAKER_BACK_LEFT | SPEAKER_BACK_RIGHT;
```

#### 3. BitsPerSample 与 BytesPerSample

```cpp
// 两者的关系
// BitsPerSample: 有效数据位数
// BytesPerSample: 存储占用字节数（可能有填充）

// 示例 1: 标准 16-bit 音频
format.BitsPerSample = 16;
format.BytesPerSample = 2;  // 16/8 = 2，无填充

// 示例 2: 24-bit 音频（紧凑存储）
format.BitsPerSample = 24;
format.BytesPerSample = 3;  // 3 字节紧凑存储

// 示例 3: 24-bit 音频（32-bit 容器）
format.BitsPerSample = 24;
format.BytesPerSample = 4;  // 高 8 位为填充

// 示例 4: 32-bit 浮点
format.BitsPerSample = 32;
format.BytesPerSample = 4;
format.IsFloat = true;

// 计算每粒度字节数
size_t bytesPerGranule = format.Channels * format.BytesPerSample;
```

#### 4. TotalSamples 与 TotalTime

```cpp
tjs_uint64 TotalSamples;  // 总粒度数
tjs_uint64 TotalTime;     // 总毫秒数

// 关系: TotalTime = TotalSamples * 1000 / SamplesPerSec

// 流式音频（无法预知总长度）
if (format.TotalSamples == 0 && format.TotalTime == 0) {
    // 这是流式音频，长度未知
    // 播放器应显示 "--:--" 而非 "00:00"
}

// 计算示例：3 分钟 CD 音频
// SamplesPerSec = 44100
// TotalSamples = 44100 * 180 = 7,938,000
// TotalTime = 180,000 ms
```

#### 5. IsFloat 与数据范围

```cpp
bool IsFloat;

// 整数 PCM 数据范围
// 8-bit:  0 ~ 255      (无符号，128 为静音)
// 16-bit: -32768 ~ 32767 (有符号)
// 24-bit: -8388608 ~ 8388607
// 32-bit: -2147483648 ~ 2147483647

// 浮点 PCM 数据范围
// 32-bit float: -1.0 ~ +1.0 (可能超出，需裁剪)

// 浮点转整数示例
int16_t FloatToInt16(float sample) {
    // 裁剪到 [-1.0, 1.0]
    if (sample > 1.0f) sample = 1.0f;
    if (sample < -1.0f) sample = -1.0f;
    
    // 缩放到 16-bit 范围
    return static_cast<int16_t>(sample * 32767.0f);
}
```

---

## 核心方法实现指南

### GetFormat() 方法

`GetFormat()` 在解码器构造后被调用，用于获取音频格式信息。

```cpp
// 实现示例：简单 WAV 解码器
class SimpleWavDecoder : public tTVPWaveDecoder {
private:
    tTVPWaveFormat format_;
    FILE* file_;
    size_t dataOffset_;
    size_t dataSize_;

public:
    SimpleWavDecoder(const char* filename) {
        file_ = fopen(filename, "rb");
        if (!file_) throw std::runtime_error("无法打开文件");
        
        // 解析 WAV 头
        ParseWavHeader();
    }
    
    void GetFormat(tTVPWaveFormat& format) override {
        // 直接复制预解析的格式信息
        format = format_;
    }
    
private:
    void ParseWavHeader() {
        // 读取 RIFF 头
        char riff[4];
        fread(riff, 1, 4, file_);
        if (memcmp(riff, "RIFF", 4) != 0) {
            throw std::runtime_error("不是有效的 WAV 文件");
        }
        
        // 跳过文件大小
        fseek(file_, 4, SEEK_CUR);
        
        // 验证 WAVE 标识
        char wave[4];
        fread(wave, 1, 4, file_);
        if (memcmp(wave, "WAVE", 4) != 0) {
            throw std::runtime_error("不是有效的 WAV 文件");
        }
        
        // 查找 fmt 块
        while (true) {
            char chunkId[4];
            uint32_t chunkSize;
            if (fread(chunkId, 1, 4, file_) != 4) break;
            if (fread(&chunkSize, 4, 1, file_) != 1) break;
            
            if (memcmp(chunkId, "fmt ", 4) == 0) {
                // 解析格式块
                uint16_t audioFormat;
                fread(&audioFormat, 2, 1, file_);
                
                uint16_t channels;
                fread(&channels, 2, 1, file_);
                format_.Channels = channels;
                
                uint32_t sampleRate;
                fread(&sampleRate, 4, 1, file_);
                format_.SamplesPerSec = sampleRate;
                
                fseek(file_, 4, SEEK_CUR);  // 跳过 byteRate
                fseek(file_, 2, SEEK_CUR);  // 跳过 blockAlign
                
                uint16_t bitsPerSample;
                fread(&bitsPerSample, 2, 1, file_);
                format_.BitsPerSample = bitsPerSample;
                format_.BytesPerSample = bitsPerSample / 8;
                
                format_.IsFloat = (audioFormat == 3);  // IEEE float
                format_.SpeakerConfig = 0;
                
                // 跳过剩余格式数据
                fseek(file_, chunkSize - 16, SEEK_CUR);
            }
            else if (memcmp(chunkId, "data", 4) == 0) {
                dataOffset_ = ftell(file_);
                dataSize_ = chunkSize;
                
                // 计算总样本数
                size_t bytesPerGranule = 
                    format_.Channels * format_.BytesPerSample;
                format_.TotalSamples = dataSize_ / bytesPerGranule;
                format_.TotalTime = 
                    format_.TotalSamples * 1000 / format_.SamplesPerSec;
                format_.Seekable = true;
                break;
            }
            else {
                // 跳过未知块
                fseek(file_, chunkSize, SEEK_CUR);
            }
        }
    }
};
```

### Render() 方法

`Render()` 是解码器的核心方法，负责将压缩音频解码为 PCM 数据。

```cpp
// 实现要点
bool Render(void* buf, tjs_uint bufsamplelen, tjs_uint& rendered) override {
    // 1. 计算请求的字节数
    size_t bytesPerGranule = format_.Channels * format_.BytesPerSample;
    size_t requestedBytes = bufsamplelen * bytesPerGranule;
    
    // 2. 计算剩余可读字节数
    size_t currentPos = ftell(file_) - dataOffset_;
    size_t remainingBytes = dataSize_ - currentPos;
    size_t bytesToRead = std::min(requestedBytes, remainingBytes);
    
    // 3. 读取数据
    size_t bytesRead = fread(buf, 1, bytesToRead, file_);
    
    // 4. 计算实际渲染的粒度数
    rendered = static_cast<tjs_uint>(bytesRead / bytesPerGranule);
    
    // 5. 返回是否还有更多数据
    // 注意：即使 rendered < bufsamplelen，只要还有数据就返回 true
    return (currentPos + bytesRead) < dataSize_;
}
```

**关键点：返回值语义**

```cpp
// 返回值含义解析
// true:  还有更多数据可解码（即使本次 rendered < bufsamplelen）
// false: 已经到达文件末尾（解码完成）

// 错误示例：过早返回 false
bool Render_WRONG(void* buf, tjs_uint bufsamplelen, tjs_uint& rendered) {
    // ... 解码 ...
    if (rendered < bufsamplelen) {
        return false;  // 错误！可能只是临时数据不足
    }
    return true;
}

// 正确示例：只在真正结束时返回 false
bool Render_CORRECT(void* buf, tjs_uint bufsamplelen, tjs_uint& rendered) {
    // ... 解码 ...
    return !IsEndOfStream();  // 只有真正结束才返回 false
}
```

### SetPosition() 方法

```cpp
// 定位实现示例
bool SetPosition(tjs_uint64 samplepos) override {
    if (!format_.Seekable) {
        return false;  // 格式不支持定位
    }
    
    // 计算字节偏移
    size_t bytesPerGranule = format_.Channels * format_.BytesPerSample;
    size_t byteOffset = static_cast<size_t>(samplepos * bytesPerGranule);
    
    // 边界检查
    if (byteOffset > dataSize_) {
        return false;  // 超出范围
    }
    
    // 执行定位
    int result = fseek(file_, 
                       static_cast<long>(dataOffset_ + byteOffset), 
                       SEEK_SET);
    return (result == 0);
}

// 对于压缩格式（如 Vorbis），定位更复杂
bool VorbisSetPosition(tjs_uint64 samplepos) {
    // Vorbis 使用 ov_pcm_seek
    int result = ov_pcm_seek(&vorbisFile_, samplepos);
    return (result == 0);
}
```

---

## 完整解码器实现示例

以下是一个完整的原始 PCM 文件解码器实现：

```cpp
// raw_pcm_decoder.h
#pragma once
#include "WaveIntf.h"
#include <cstdio>
#include <cstring>
#include <stdexcept>

/**
 * 原始 PCM 解码器
 * 支持读取无头部的原始 PCM 数据文件
 * 格式参数通过构造函数指定
 */
class RawPCMDecoder : public tTVPWaveDecoder {
private:
    FILE* file_;
    tTVPWaveFormat format_;
    size_t dataSize_;
    size_t currentPosition_;

public:
    /**
     * 构造原始 PCM 解码器
     * @param filename 文件路径
     * @param sampleRate 采样率
     * @param channels 声道数
     * @param bitsPerSample 位深度
     * @param isFloat 是否为浮点格式
     */
    RawPCMDecoder(const char* filename,
                  tjs_uint sampleRate,
                  tjs_uint channels,
                  tjs_uint bitsPerSample,
                  bool isFloat = false) 
        : file_(nullptr), currentPosition_(0) 
    {
        // 打开文件
        file_ = fopen(filename, "rb");
        if (!file_) {
            throw std::runtime_error("无法打开 PCM 文件");
        }
        
        // 获取文件大小
        fseek(file_, 0, SEEK_END);
        dataSize_ = ftell(file_);
        fseek(file_, 0, SEEK_SET);
        
        // 设置格式信息
        format_.SamplesPerSec = sampleRate;
        format_.Channels = channels;
        format_.BitsPerSample = bitsPerSample;
        format_.BytesPerSample = bitsPerSample / 8;
        format_.IsFloat = isFloat;
        format_.SpeakerConfig = 0;  // 默认配置
        format_.Seekable = true;
        
        // 计算总样本数和时长
        size_t bytesPerGranule = channels * format_.BytesPerSample;
        format_.TotalSamples = dataSize_ / bytesPerGranule;
        format_.TotalTime = format_.TotalSamples * 1000 / sampleRate;
    }
    
    ~RawPCMDecoder() override {
        if (file_) {
            fclose(file_);
            file_ = nullptr;
        }
    }
    
    void GetFormat(tTVPWaveFormat& format) override {
        format = format_;
    }
    
    bool Render(void* buf, tjs_uint bufsamplelen, tjs_uint& rendered) override {
        if (!file_) {
            rendered = 0;
            return false;
        }
        
        // 计算请求字节数
        size_t bytesPerGranule = format_.Channels * format_.BytesPerSample;
        size_t requestedBytes = bufsamplelen * bytesPerGranule;
        
        // 计算可读字节数
        size_t remainingBytes = dataSize_ - currentPosition_;
        size_t bytesToRead = std::min(requestedBytes, remainingBytes);
        
        // 读取数据
        size_t bytesRead = fread(buf, 1, bytesToRead, file_);
        currentPosition_ += bytesRead;
        
        // 计算渲染的粒度数
        rendered = static_cast<tjs_uint>(bytesRead / bytesPerGranule);
        
        // 判断是否还有更多数据
        return currentPosition_ < dataSize_;
    }
    
    bool SetPosition(tjs_uint64 samplepos) override {
        if (!file_ || !format_.Seekable) {
            return false;
        }
        
        // 计算目标字节位置
        size_t bytesPerGranule = format_.Channels * format_.BytesPerSample;
        size_t targetPos = static_cast<size_t>(samplepos * bytesPerGranule);
        
        // 边界检查
        if (targetPos > dataSize_) {
            return false;
        }
        
        // 执行定位
        if (fseek(file_, static_cast<long>(targetPos), SEEK_SET) != 0) {
            return false;
        }
        
        currentPosition_ = targetPos;
        return true;
    }
};
```

---

## 对照项目源码

### tTVPWD_RIFFWave（WAV 解码器）

项目内置的 WAV 解码器位于 `WaveIntf.cpp`，展示了标准实现模式：

**相关文件：**
- `cpp/core/sound/WaveIntf.cpp` 第 400-704 行

```cpp
// 文件: cpp/core/sound/WaveIntf.cpp 第 400-430 行
class tTVPWD_RIFFWave : public tTVPWaveDecoder {
    tTJSBinaryStream *Stream;  // 文件流
    tTVPWaveFormat Format;      // 格式信息
    tjs_uint64 DataStart;       // 数据起始位置
    tjs_uint64 CurrentPos;      // 当前位置
    tjs_uint64 DataLength;      // 数据长度

public:
    tTVPWD_RIFFWave(tTJSBinaryStream *stream, 
                    tjs_uint64 datastart,
                    const tTVPWaveFormat &format) {
        Stream = stream;
        DataStart = datastart;
        CurrentPos = 0;
        Format = format;
        DataLength = Format.TotalSamples * 
                     Format.Channels * Format.BytesPerSample;
    }

    void GetFormat(tTVPWaveFormat &format) override { 
        format = Format; 
    }
    
    // ... Render 和 SetPosition 实现 ...
};
```

### PCM 格式转换函数

项目提供了丰富的 PCM 格式转换函数：

**相关文件：**
- `cpp/core/sound/WaveIntf.cpp` 第 97-394 行

```cpp
// 浮点转 16-bit 整数
// 文件: cpp/core/sound/WaveIntf.cpp 第 97-140 行
static void TVPConvertFloatPCMTo16bits(
    tjs_int16 *output, 
    const float *input,
    tjs_int channels, 
    tjs_int count,
    bool downmix)  // downmix=true 时混合为单声道
{
    if (!downmix) {
        // 直接转换
        tjs_int total = channels * count;
        PCMConvertLoopFloat32ToInt16(output, input, total);
    } else {
        // 混合到单声道
        float nc = 32768.0f / (float)channels;
        while (count--) {
            tjs_int n = channels;
            float t = 0;
            while (n--)
                t += *(input++) * nc;
            // 裁剪和量化
            if (t > 0) {
                int i = (int)(t + 0.5);
                if (i > 32767) i = 32767;
                *(output++) = (tjs_int16)i;
            } else {
                int i = (int)(t - 0.5);
                if (i < -32768) i = -32768;
                *(output++) = (tjs_int16)i;
            }
        }
    }
}
```

---

## 常见错误与排查

### 错误 1：渲染数据对齐问题

```cpp
// 错误：直接使用字节数作为粒度数
bool Render_WRONG(void* buf, tjs_uint bufsamplelen, tjs_uint& rendered) {
    size_t bytesRead = fread(buf, 1, bufsamplelen, file_);  // 错！
    rendered = bytesRead;  // 单位混淆
    return true;
}

// 正确：按粒度计算
bool Render_CORRECT(void* buf, tjs_uint bufsamplelen, tjs_uint& rendered) {
    size_t bytesPerGranule = format_.Channels * format_.BytesPerSample;
    size_t bytesToRead = bufsamplelen * bytesPerGranule;
    size_t bytesRead = fread(buf, 1, bytesToRead, file_);
    rendered = bytesRead / bytesPerGranule;  // 转换回粒度数
    return true;
}
```

### 错误 2：定位时未考虑数据起始偏移

```cpp
// 错误：直接使用采样位置
bool SetPosition_WRONG(tjs_uint64 samplepos) {
    fseek(file_, samplepos * bytesPerGranule, SEEK_SET);  // 错！
    return true;
}

// 正确：加上数据区起始偏移
bool SetPosition_CORRECT(tjs_uint64 samplepos) {
    size_t offset = dataOffset_ + samplepos * bytesPerGranule;
    fseek(file_, offset, SEEK_SET);
    return true;
}
```

### 错误 3：浮点数据未裁剪

```cpp
// 错误：直接缩放
int16_t FloatToInt16_WRONG(float sample) {
    return (int16_t)(sample * 32767.0f);  // 可能溢出！
}

// 正确：先裁剪再缩放
int16_t FloatToInt16_CORRECT(float sample) {
    sample = std::clamp(sample, -1.0f, 1.0f);
    return (int16_t)(sample * 32767.0f);
}
```

---

## 本节小结

- `tTVPWaveDecoder` 是所有音频解码器的抽象基类，定义了 `GetFormat`、`Render`、`SetPosition` 三个核心方法
- `tTVPWaveFormat` 结构体描述音频格式，包含采样率、声道数、位深度、浮点标志等字段
- 采样粒度（Sample Granule）是数据单位，表示一个时间点上所有声道的采样值
- `Render()` 返回 `false` 仅表示文件结束，不表示错误或数据不足
- 实现解码器时需注意字节/粒度单位转换、数据起始偏移、浮点数据裁剪等细节

---

## 练习题与答案

### 题目 1：计算音频缓冲区大小

一段 48kHz、双声道、24-bit 的音频，需要缓冲 500ms 的数据，请计算：
1. 需要多少个采样粒度？
2. 需要多大的缓冲区（字节）？

<details>
<summary>查看答案</summary>

```cpp
// 已知参数
tjs_uint sampleRate = 48000;   // 48kHz
tjs_uint channels = 2;          // 双声道
tjs_uint bytesPerSample = 3;    // 24-bit = 3 字节
tjs_uint durationMs = 500;      // 500 毫秒

// 1. 计算采样粒度数
tjs_uint granules = sampleRate * durationMs / 1000;
// granules = 48000 * 500 / 1000 = 24000 个粒度

// 2. 计算缓冲区大小
size_t bytesPerGranule = channels * bytesPerSample;  // 2 * 3 = 6 字节
size_t bufferSize = granules * bytesPerGranule;
// bufferSize = 24000 * 6 = 144000 字节 = 140.625 KB

#include <iostream>

int main() {
    tjs_uint sampleRate = 48000;
    tjs_uint channels = 2;
    tjs_uint bytesPerSample = 3;
    tjs_uint durationMs = 500;
    
    tjs_uint granules = sampleRate * durationMs / 1000;
    size_t bytesPerGranule = channels * bytesPerSample;
    size_t bufferSize = granules * bytesPerGranule;
    
    std::cout << "采样粒度数: " << granules << std::endl;
    std::cout << "缓冲区大小: " << bufferSize << " 字节" << std::endl;
    
    return 0;
}
// 输出:
// 采样粒度数: 24000
// 缓冲区大小: 144000 字节
```

</details>

### 题目 2：实现 TotalTime 计算

给定以下 `tTVPWaveFormat`，编写函数计算并验证 `TotalTime` 字段：

```cpp
tTVPWaveFormat format;
format.SamplesPerSec = 44100;
format.Channels = 2;
format.BitsPerSample = 16;
format.BytesPerSample = 2;
format.TotalSamples = 13230000;  // 待验证的总时长（毫秒）
format.TotalTime = ?;
```

<details>
<summary>查看答案</summary>

```cpp
#include <iostream>
#include <cstdint>

// 简化的 tTVPWaveFormat 定义
struct tTVPWaveFormat {
    uint32_t SamplesPerSec;
    uint32_t Channels;
    uint32_t BitsPerSample;
    uint32_t BytesPerSample;
    uint64_t TotalSamples;
    uint64_t TotalTime;
};

uint64_t CalculateTotalTime(const tTVPWaveFormat& format) {
    // 公式: TotalTime(ms) = TotalSamples * 1000 / SamplesPerSec
    return format.TotalSamples * 1000 / format.SamplesPerSec;
}

std::string FormatDuration(uint64_t ms) {
    uint64_t seconds = ms / 1000;
    uint64_t minutes = seconds / 60;
    uint64_t hours = minutes / 60;
    
    char buf[64];
    if (hours > 0) {
        snprintf(buf, sizeof(buf), "%llu:%02llu:%02llu.%03llu",
                 hours, minutes % 60, seconds % 60, ms % 1000);
    } else {
        snprintf(buf, sizeof(buf), "%02llu:%02llu.%03llu",
                 minutes, seconds % 60, ms % 1000);
    }
    return std::string(buf);
}

int main() {
    tTVPWaveFormat format;
    format.SamplesPerSec = 44100;
    format.Channels = 2;
    format.BitsPerSample = 16;
    format.BytesPerSample = 2;
    format.TotalSamples = 13230000;
    
    format.TotalTime = CalculateTotalTime(format);
    
    std::cout << "总采样数: " << format.TotalSamples << std::endl;
    std::cout << "总时长: " << format.TotalTime << " ms" << std::endl;
    std::cout << "格式化: " << FormatDuration(format.TotalTime) << std::endl;
    
    return 0;
}
// 输出:
// 总采样数: 13230000
// 总时长: 300000 ms
// 格式化: 05:00.000
// 结论: 这是一段 5 分钟整的音频
```

</details>

### 题目 3：实现简单的 Render 方法

为以下解码器骨架实现 `Render` 方法，要求正确处理文件末尾情况：

```cpp
class TestDecoder : public tTVPWaveDecoder {
    std::vector<int16_t> data_;  // 内存中的 PCM 数据（单声道 16-bit）
    size_t position_;             // 当前读取位置（粒度）
    tTVPWaveFormat format_;
    
public:
    TestDecoder(const std::vector<int16_t>& data, tjs_uint sampleRate) 
        : data_(data), position_(0) {
        format_.SamplesPerSec = sampleRate;
        format_.Channels = 1;
        format_.BitsPerSample = 16;
        format_.BytesPerSample = 2;
        format_.TotalSamples = data.size();
        format_.TotalTime = format_.TotalSamples * 1000 / sampleRate;
        format_.IsFloat = false;
        format_.Seekable = true;
    }
    
    void GetFormat(tTVPWaveFormat& format) override { format = format_; }
    bool SetPosition(tjs_uint64 samplepos) override { 
        if (samplepos > data_.size()) return false;
        position_ = samplepos;
        return true;
    }
    
    bool Render(void* buf, tjs_uint bufsamplelen, tjs_uint& rendered) override {
        // TODO: 实现此方法
    }
};
```

<details>
<summary>查看答案</summary>

```cpp
#include <vector>
#include <cstdint>
#include <cstring>
#include <algorithm>
#include <iostream>

// 简化的类型定义
using tjs_uint = uint32_t;
using tjs_uint64 = uint64_t;

struct tTVPWaveFormat {
    tjs_uint SamplesPerSec;
    tjs_uint Channels;
    tjs_uint BitsPerSample;
    tjs_uint BytesPerSample;
    tjs_uint64 TotalSamples;
    tjs_uint64 TotalTime;
    bool IsFloat;
    bool Seekable;
};

class tTVPWaveDecoder {
public:
    virtual ~tTVPWaveDecoder() = default;
    virtual void GetFormat(tTVPWaveFormat& format) = 0;
    virtual bool Render(void* buf, tjs_uint bufsamplelen, tjs_uint& rendered) = 0;
    virtual bool SetPosition(tjs_uint64 samplepos) = 0;
};

class TestDecoder : public tTVPWaveDecoder {
    std::vector<int16_t> data_;
    size_t position_;
    tTVPWaveFormat format_;
    
public:
    TestDecoder(const std::vector<int16_t>& data, tjs_uint sampleRate) 
        : data_(data), position_(0) {
        format_.SamplesPerSec = sampleRate;
        format_.Channels = 1;
        format_.BitsPerSample = 16;
        format_.BytesPerSample = 2;
        format_.TotalSamples = data.size();
        format_.TotalTime = format_.TotalSamples * 1000 / sampleRate;
        format_.IsFloat = false;
        format_.Seekable = true;
    }
    
    void GetFormat(tTVPWaveFormat& format) override { format = format_; }
    
    bool SetPosition(tjs_uint64 samplepos) override { 
        if (samplepos > data_.size()) return false;
        position_ = static_cast<size_t>(samplepos);
        return true;
    }
    
    bool Render(void* buf, tjs_uint bufsamplelen, tjs_uint& rendered) override {
        // 计算剩余可读的粒度数
        size_t remaining = data_.size() - position_;
        
        // 确定实际要读取的粒度数
        size_t toRead = std::min(static_cast<size_t>(bufsamplelen), remaining);
        
        if (toRead > 0) {
            // 复制数据到输出缓冲区
            // 单声道 16-bit，每粒度 = 1 个 int16_t = 2 字节
            std::memcpy(buf, &data_[position_], toRead * sizeof(int16_t));
            position_ += toRead;
        }
        
        rendered = static_cast<tjs_uint>(toRead);
        
        // 返回是否还有更多数据
        return position_ < data_.size();
    }
};

// 测试代码
int main() {
    // 创建测试数据：1 秒的 1kHz 正弦波
    std::vector<int16_t> testData(44100);
    for (size_t i = 0; i < testData.size(); ++i) {
        testData[i] = static_cast<int16_t>(
            32767.0 * std::sin(2.0 * 3.14159 * 1000.0 * i / 44100.0)
        );
    }
    
    TestDecoder decoder(testData, 44100);
    
    // 测试渲染
    std::vector<int16_t> buffer(1024);
    tjs_uint rendered;
    int totalRendered = 0;
    
    while (decoder.Render(buffer.data(), 1024, rendered)) {
        totalRendered += rendered;
        std::cout << "本次渲染: " << rendered << " 粒度" << std::endl;
    }
    totalRendered += rendered;  // 最后一次
    
    std::cout << "总共渲染: " << totalRendered << " 粒度" << std::endl;
    std::cout << "期望: " << testData.size() << " 粒度" << std::endl;
    
    // 测试定位
    decoder.SetPosition(22050);  // 定位到 0.5 秒
    decoder.Render(buffer.data(), 100, rendered);
    std::cout << "定位后渲染: " << rendered << " 粒度" << std::endl;
    
    return 0;
}
// 输出:
// 本次渲染: 1024 粒度
// ... (重复多次)
// 总共渲染: 44100 粒度
// 期望: 44100 粒度
// 定位后渲染: 100 粒度
```

</details>

---

## 下一步

[02-Creator注册与格式探测](./02-Creator注册与格式探测.md) — 学习解码器工厂模式与音频格式自动检测机制
