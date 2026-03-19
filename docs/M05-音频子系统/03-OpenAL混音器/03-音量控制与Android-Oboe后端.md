# 音量控制与Android-Oboe后端

> **所属模块：** M05-音频子系统
> **前置知识：** [01-iTVPSoundBuffer接口与OpenAL后端](./01-iTVPSoundBuffer接口与OpenAL后端.md)、[02-双层缓冲与解码线程](./02-双层缓冲与解码线程.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 理解KrKr2音量控制的14位定点数实现与非线性混音算法
2. 掌握Pan（声像）控制的立体声衰减与OpenAL 3D定位两种实现
3. 理解Android Oboe低延迟音频后端的回调模型与优势
4. 分析渲染器选择策略与多后端架构设计
5. 实现自定义音量曲线与淡入淡出效果

---

## 1. 音量控制架构概述

KrKr2的音量控制分布在两个层次：

```
┌─────────────────────────────────────────────────────────────┐
│                     TJS2 脚本层                              │
│   waveSoundBuffer.volume = 0.8;  // 0.0 ~ 1.0 浮点数        │
│   waveSoundBuffer.pan = -0.5;    // -1.0(左) ~ +1.0(右)     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  iTVPSoundBuffer 接口层                      │
│   SetVolume(float v);   // 存储为 _volume 成员              │
│   SetPan(float v);      // 存储为 _pan 成员                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    混音器实现层                              │
│   ┌───────────────────┐  ┌───────────────────────────────┐ │
│   │  软件混音器        │  │  OpenAL 混音器               │ │
│   │ (tTVPSoundBuffer)  │  │ (tTVPSoundBufferAL)          │ │
│   │  14位定点数运算    │  │  alSourcef(AL_GAIN, vol)     │ │
│   │  RecalcVolume()    │  │  alSourcefv(AL_POSITION,pos) │ │
│   └───────────────────┘  └───────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 1.1 两种混音器的音量处理差异

| 特性 | 软件混音器 (tTVPSoundBuffer) | OpenAL混音器 (tTVPSoundBufferAL) |
|------|------------------------------|----------------------------------|
| 音量精度 | 14位定点数 (0-16384) | 32位浮点数 (0.0-1.0+) |
| Pan实现 | 左右声道独立衰减 | 3D空间位置 (x, 0, 0) |
| 混音算法 | 非线性软削波 | OpenAL内部线性混音 |
| CPU占用 | 较高（逐样本运算） | 较低（硬件加速） |
| 精度控制 | 完全可控 | 依赖OpenAL实现 |

---

## 2. 软件混音器的14位定点数音量

### 2.1 为什么选择14位定点数？

在音频处理中，浮点数运算虽然精度高，但有以下问题：
1. **性能开销**：每个样本都需浮点乘法
2. **精度损失**：多次浮点运算累积误差
3. **跨平台差异**：不同CPU的浮点单元行为可能不同

KrKr2采用14位定点数作为折中方案：

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第101-125行

class tTVPSoundBuffer : public iTVPSoundBuffer {
public:
    float _volume = 1;                  // 用户设置的音量 (0.0 ~ 1.0)
    float _pan = 0;                     // 用户设置的声像 (-1.0 ~ +1.0)
    const signed int MAX_VOLUME = 16384; // 14位最大值 = 2^14
    int16_t _volume_raw[8];             // 预计算的各声道音量（支持7.1声道）

    void RecalcVolume() {
        // 左声道：当pan>0时衰减，pan<=0时全量
        if(_pan > 0) {
            _volume_raw[0] = (1.0f - _pan) * _volume * MAX_VOLUME;
        } else {
            _volume_raw[0] = _volume * MAX_VOLUME;
        }
        // 右声道：当pan<0时衰减，pan>=0时全量
        if(_pan < 0) {
            _volume_raw[1] = (_pan + 1.0f) * _volume * MAX_VOLUME;
        } else {
            _volume_raw[1] = _volume * MAX_VOLUME;
        }
        // SIMD优化：复制到[2][3]供并行处理
        _volume_raw[2] = _volume_raw[0];
        _volume_raw[3] = _volume_raw[1];
    }
};
```

### 2.2 定点数运算详解

```cpp
// 16位有符号音频样本范围: -32768 ~ +32767
// 14位音量范围: 0 ~ 16384
// 乘积范围: -536,870,912 ~ +536,854,528 (需要32位存储)

// 音量应用公式 (整数运算):
int src_sample = *src16;                     // 原始样本 (16位)
src_sample = (src_sample * volume[i]) >> 14; // 乘音量后右移14位
// 等效于: src_sample = src_sample * (volume[i] / 16384.0)
```

**示例计算**：
```
原始样本: 20000 (约61%最大值)
音量设置: 0.5 → volume_raw = 8192
计算过程: (20000 * 8192) >> 14 = 163840000 >> 14 = 10000
结果: 10000 (正好是原始值的50%)
```

### 2.3 完整的S16混音模板实现

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第36-57行

template <int ch>  // ch = 声道数，编译期确定
void MixAudioS16CPP(void *dst, const void *src, int samples, int16_t *volume) {
    int16_t *dst16 = (int16_t *)dst;      // 目标缓冲区（已有数据）
    const int16_t *src16 = (const int16_t *)src;  // 源缓冲区
    
    while(samples--) {
        for(int i = 0; i < ch; ++i) {
            // 步骤1: 读取源样本并应用音量
            int src_sample = *src16++;
            src_sample = (src_sample * volume[i]) >> 14;
            
            // 步骤2: 读取目标缓冲区已有数据
            int dest_sample = *dst16;
            
            // 步骤3: 非线性混音（防削波）
            if(src_sample > 0 && dest_sample > 0) {
                // 两个正值：降低混合强度防止溢出
                dest_sample = src_sample + dest_sample -
                    ((dest_sample * src_sample + 0x8000) >> 15);
            } else if(src_sample < 0 && dest_sample < 0) {
                // 两个负值：同样降低强度
                dest_sample = src_sample + dest_sample +
                    ((dest_sample * src_sample) >> 15);
            } else {
                // 一正一负或零：直接相加（不会溢出）
                dest_sample += src_sample;
            }
            
            // 步骤4: 写回结果
            *dst16++ = dest_sample;
        }
    }
}
```

### 2.4 非线性混音算法原理

传统线性混音 `result = A + B` 在两个信号都接近最大值时会溢出。KrKr2使用的非线性公式：

```
当 A > 0 且 B > 0:
    result = A + B - (A * B) / 32768
    
当 A < 0 且 B < 0:
    result = A + B + (A * B) / 32768
    
否则:
    result = A + B
```

**数学分析**：
- 两个最大正值相加：`32767 + 32767 - (32767 * 32767 / 32768) = 65534 - 32766 = 32768` ≈ 最大值
- 衰减因子随幅度增大而增大，实现"软削波"效果
- 保留了混音的动态范围，避免失真

```cpp
// 示例：两个50%幅度信号混音
int A = 16384, B = 16384;  // 各50%
int linear = A + B;         // = 32768 (溢出!)
int nonlinear = A + B - ((A * B + 0x8000) >> 15);
// = 32768 - ((268435456 + 32768) >> 15)
// = 32768 - 8192 = 24576 (安全范围内)
```

---

## 3. F32浮点混音实现

对于浮点音频格式，KrKr2提供了对应的混音器：

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第59-83行

template <int ch>
void MixAudioF32CPP(void *dst, const void *src, int samples, int16_t *volume) {
    float *dst32 = (float *)dst;
    const float *src32 = (const float *)src;
    
    // 将14位定点音量转换为浮点数
    const float fmaxvolume = 1.0f / 16384;  // = 1/MAX_VOLUME
    float fvolume[ch];
    for(int i = 0; i < ch; ++i)
        fvolume[i] = volume[i] * fmaxvolume;
    
    while(samples--) {
        for(int i = 0; i < ch; ++i) {
            // 读取并应用音量（处理字节序）
            float src_sample = SDL_SwapFloatLE(*src32++) * fvolume[i];
            float dest_sample = SDL_SwapFloatLE(*dst32);
            
            // 非线性混音（浮点版本更简洁）
            if(src_sample > 0 && dest_sample > 0) {
                dest_sample = src_sample + dest_sample - dest_sample * src_sample;
            } else if(src_sample < 0 && dest_sample < 0) {
                dest_sample = src_sample + dest_sample + dest_sample * src_sample;
            } else {
                dest_sample += src_sample;
            }
            
            // 写回（处理字节序）
            *(dst32++) = SDL_SwapFloatLE(dest_sample);
        }
    }
}
```

### 3.1 字节序处理

`SDL_SwapFloatLE` 确保在大端机器上正确处理小端格式的浮点数：

```cpp
// SDL2 内部实现 (简化版)
static inline float SDL_SwapFloatLE(float x) {
#if SDL_BYTEORDER == SDL_BIG_ENDIAN
    union { float f; uint32_t u; } val;
    val.f = x;
    val.u = SDL_Swap32(val.u);
    return val.f;
#else
    return x;  // 小端机器直接返回
#endif
}
```

---

## 4. 混音器函数表与动态选择

KrKr2使用函数指针数组支持1-8声道：

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第85-97行

typedef void(FAudioMix)(void *dst, const void *src, int samples, int16_t *volume);

// S16格式函数表
static FAudioMix *_AudioMixS16[8] = {
    &MixAudioS16CPP<1>, &MixAudioS16CPP<2>, &MixAudioS16CPP<3>,
    &MixAudioS16CPP<4>, &MixAudioS16CPP<5>, &MixAudioS16CPP<6>,
    &MixAudioS16CPP<7>, &MixAudioS16CPP<8>
};

// F32格式函数表
static FAudioMix *_AudioMixF32[8] = {
    &MixAudioF32CPP<1>, &MixAudioF32CPP<2>, &MixAudioF32CPP<3>,
    &MixAudioF32CPP<4>, &MixAudioF32CPP<5>, &MixAudioF32CPP<6>,
    &MixAudioF32CPP<7>, &MixAudioF32CPP<8>
};
```

渲染器初始化时选择正确的混音函数：

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第266-275行

void SetupMixer() {
    if(_spec.format == AUDIO_S16LSB) {
        DoMixAudio = _AudioMixS16[_spec.channels - 1];
    } else if(_spec.format == AUDIO_F32LSB) {
        DoMixAudio = _AudioMixF32[_spec.channels - 1];
    } else {
        // 不支持的格式：空操作
        DoMixAudio = [](void*, const void*, int, int16_t*) {};
    }
}
```

---

## 5. OpenAL音量与3D定位

### 5.1 OpenAL音量控制

OpenAL后端直接使用OpenAL的音量API：

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第631-640行

void SetVolume(float volume) override {
    alSourcef(_alSource, AL_GAIN, volume);  // 直接设置增益
    checkerr("SetVolume");
}

float GetVolume() override {
    float volume = 0;
    alGetSourcef(_alSource, AL_GAIN, &volume);
    return volume;
}
```

**AL_GAIN 参数说明**：
- 范围：0.0（静音）到无上限（但>1.0可能导致失真）
- 默认值：1.0
- 线性比例：0.5 = 50% 音量

### 5.2 Pan通过3D位置实现

OpenAL没有直接的Pan参数，KrKr2巧妙地使用3D位置模拟：

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第642-651行

void SetPan(float pan) override {
    // 将pan (-1 ~ +1) 映射到X坐标
    float sourcePosAL[] = { pan, 0.0f, 0.0f };
    alSourcefv(_alSource, AL_POSITION, sourcePosAL);
}

float GetPan() override {
    float sourcePosAL[3];
    alGetSourcefv(_alSource, AL_POSITION, sourcePosAL);
    return sourcePosAL[0];  // 返回X坐标作为pan值
}
```

**工作原理**：
```
              听者位置 (0, 0, 0)
                     |
    左(-1,0,0) ←─────┼─────→ 右(+1,0,0)
                     |
                     
当声源在 (-1, 0, 0) 时，左耳听到更多
当声源在 (+1, 0, 0) 时，右耳听到更多
当声源在 (0, 0, 0) 时，两耳均衡
```

### 5.3 完整3D定位支持

OpenAL后端还支持完整的3D空间定位：

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第659-663行

void SetPosition(float x, float y, float z) override {
    float sourcePosAL[] = { x, y, z };
    alSourcefv(_alSource, AL_POSITION, sourcePosAL);
    checkerr("SetPosition");
}
```

---

## 6. Android Oboe低延迟后端

### 6.1 为什么需要Oboe？

Android音频系统历史上存在高延迟问题：

| 平台 | 典型延迟 | 原因 |
|------|----------|------|
| iOS | ~5ms | 底层AudioUnit直接访问 |
| Windows | ~10-20ms | WASAPI独占模式 |
| Linux | ~10-30ms | ALSA/PulseAudio |
| **Android (传统)** | **100-200ms** | AudioTrack多层缓冲 |
| **Android (Oboe)** | **~20ms** | AAudio/OpenSL ES优化路径 |

Google开发的Oboe库统一了AAudio（Android 8.0+）和OpenSL ES（旧版本）的接口。

### 6.2 Oboe渲染器实现

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第411-485行

#ifdef __ANDROID__

class tTVPAudioRendererOboe : public iTVPAudioRenderer,
                              public oboe::AudioStreamCallback {
    oboe::AudioStream *_oboeAudioStream = nullptr;

public:
    virtual ~tTVPAudioRendererOboe() {
        if(_oboeAudioStream)
            delete _oboeAudioStream;
    }

    bool Init() override {
        InitMixer();  // 初始化SDL格式转换器
        
        // 使用Builder模式配置音频流
        oboe::AudioStreamBuilder builder;
        builder.setChannelCount(2);           // 立体声
        builder.setCallback(this);            // 回调模式
        // builder.setPerformanceMode(PerformanceMode::LowLatency);
        // builder.setSharingMode(SharingMode::Exclusive);
        
        oboe::Result result = builder.openStream(&_oboeAudioStream);
        
        if(result == oboe::Result::OK) {
            // 获取实际参数（可能与请求不同）
            _spec.freq = _oboeAudioStream->getSampleRate();
            
            // 确定音频格式
            switch(_oboeAudioStream->getFormat()) {
                case oboe::AudioFormat::I16:
                    _spec.format = AUDIO_S16LSB;
                    break;
                case oboe::AudioFormat::Float:
                    _spec.format = AUDIO_F32LSB;
                    break;
            }
            
            _frame_size = SDL_AUDIO_BITSIZE(_spec.format) / 8 * _spec.channels;
            _oboeAudioStream->requestStart();
            SDL_Log("Audio Device: Oboe @%dHz", _spec.freq);
            SetupMixer();
            return true;
        }
        
        SDL_Log("Fail to open Oboe audio");
        return false;
    }
```

### 6.3 Oboe回调模型

与OpenAL的轮询式buffer queue不同，Oboe使用回调模型：

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第461-472行

virtual oboe::DataCallbackResult onAudioReady(
    oboe::AudioStream *oboeStream,
    void *audioData,
    int32_t numFrames) override 
{
    // 计算需要填充的字节数
    int len = _frame_size * numFrames;
    
    // 清零缓冲区
    memset(audioData, 0, len);
    
    // 混音填充
    FillBuffer((uint8_t *)audioData, len);
    
    // 继续播放
    return oboe::DataCallbackResult::Continue;
}
```

**回调模型优势**：
1. **低延迟**：数据直接写入硬件缓冲区
2. **精确控制**：系统告知确切需要的帧数
3. **无轮询开销**：不需要检查缓冲区状态
4. **实时优先级**：回调线程通常有提升的优先级

### 6.4 精确延迟计算

Oboe提供硬件级别的时间戳，可精确计算延迟：

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第474-484行

virtual int32_t GetUnprocessedSamples() {
    int64_t hardwareFrameIndex;
    int64_t timeNanoseconds;
    
    // 获取硬件已播放的帧位置
    oboe::Result result = _oboeAudioStream->getTimestamp(
        CLOCK_MONOTONIC, 
        &hardwareFrameIndex, 
        &timeNanoseconds);
    
    if(result != oboe::Result::OK) {
        return 0;  // OpenSL ES可能不支持
    }
    
    // 应用层已写入的帧数
    int64_t appFrameIndex = _oboeAudioStream->getFramesWritten();
    
    // 差值即为未播放的样本数
    return appFrameIndex - hardwareFrameIndex;
}
```

---

## 7. 渲染器选择策略

### 7.1 工厂函数实现

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第769-784行

static iTVPAudioRenderer *CreateAudioRenderer() {
    iTVPAudioRenderer *renderer = nullptr;
    
#ifdef __ANDROID__
    // Android优先使用Oboe
    renderer = new tTVPAudioRendererOboe;
    if(renderer->Init())
        return renderer;
    delete renderer;  // Oboe失败则回退
#elif defined(_MSC_VER) && 0
    // Windows可选SDL（当前禁用）
    renderer = new tTVPAudioRendererSDL;
    renderer->Init();
    return renderer;
#endif

    // 默认使用OpenAL（所有平台通用）
    renderer = new tTVPAudioRendererAL;
    renderer->Init();
    return renderer;
}
```

### 7.2 平台选择矩阵

| 平台 | 首选渲染器 | 回退方案 | 混音模式 |
|------|------------|----------|----------|
| Android | Oboe | OpenAL | 软件混音 |
| Windows | OpenAL | SDL (禁用) | OpenAL硬件 |
| Linux | OpenAL | - | OpenAL硬件 |
| macOS | OpenAL | - | OpenAL硬件 |

---

## 8. 动手实践

### 实践1：实现音量渐变效果

```cpp
#include <cmath>
#include <thread>
#include <chrono>

// 线性渐变
void fadeLinear(iTVPSoundBuffer* buffer, 
                float from, float to, int durationMs) {
    const int steps = durationMs / 10;  // 每10ms一步
    const float delta = (to - from) / steps;
    
    float vol = from;
    for(int i = 0; i < steps; ++i) {
        buffer->SetVolume(vol);
        vol += delta;
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
    buffer->SetVolume(to);
}

// 对数渐变（更符合人耳感知）
void fadeLogarithmic(iTVPSoundBuffer* buffer, 
                     float from, float to, int durationMs) {
    const int steps = durationMs / 10;
    
    for(int i = 0; i <= steps; ++i) {
        float t = static_cast<float>(i) / steps;
        // 使用对数曲线
        float logT = std::log10(1 + t * 9) / std::log10(10);
        float vol = from + (to - from) * logT;
        buffer->SetVolume(vol);
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
}

// 使用示例
void example() {
    // 假设已创建buffer
    iTVPSoundBuffer* buffer = nullptr;
    
    // 淡入：0% → 100%，1秒
    fadeLogarithmic(buffer, 0.0f, 1.0f, 1000);
    
    // 播放5秒...
    
    // 淡出：100% → 0%，2秒
    fadeLogarithmic(buffer, 1.0f, 0.0f, 2000);
}
```

### 实践2：实现立体声旋转效果

```cpp
#include <cmath>

// 声音绕听者旋转
void panRotation(iTVPSoundBuffer* buffer, int durationMs) {
    const float PI = 3.14159265f;
    const int steps = durationMs / 20;  // 50fps
    
    for(int i = 0; i < steps; ++i) {
        float angle = 2 * PI * i / steps;  // 0 ~ 2π
        float pan = std::sin(angle);        // -1 ~ +1
        buffer->SetPan(pan);
        std::this_thread::sleep_for(std::chrono::milliseconds(20));
    }
}

// 使用OpenAL 3D位置实现更复杂的运动
void circularMotion3D(tTVPSoundBufferAL* buffer, int durationMs) {
    const float PI = 3.14159265f;
    const int steps = durationMs / 20;
    const float radius = 1.0f;
    
    for(int i = 0; i < steps; ++i) {
        float angle = 2 * PI * i / steps;
        float x = radius * std::sin(angle);   // 左右
        float z = radius * std::cos(angle);   // 前后
        buffer->SetPosition(x, 0, z);
        std::this_thread::sleep_for(std::chrono::milliseconds(20));
    }
}
```

### 实践3：自定义混音器（支持超过8声道）

```cpp
// 扩展支持更多声道
template <int ch>
void MixAudioS16Extended(void *dst, const void *src, 
                         int samples, int16_t *volume) {
    int16_t *dst16 = (int16_t *)dst;
    const int16_t *src16 = (const int16_t *)src;
    
    while(samples--) {
        for(int i = 0; i < ch; ++i) {
            int src_sample = *src16++;
            src_sample = (src_sample * volume[i % 8]) >> 14;  // 音量循环使用
            
            int dest_sample = *dst16;
            // 非线性混音
            if((src_sample > 0) == (dest_sample > 0) && 
               src_sample != 0 && dest_sample != 0) {
                int product = (dest_sample * src_sample);
                if(src_sample > 0) {
                    dest_sample = src_sample + dest_sample - 
                                  ((product + 0x8000) >> 15);
                } else {
                    dest_sample = src_sample + dest_sample + 
                                  (product >> 15);
                }
            } else {
                dest_sample += src_sample;
            }
            *dst16++ = dest_sample;
        }
    }
}
```

---

## 常见错误及解决方案

### 错误 1：14 位定点音量乘法溢出导致爆音

**现象**：音量设置为最大值（100%）时，部分高振幅采样出现严重削波（clipping），波形方波化。

**原因**：S16 格式中采样值范围为 -32768~32767。如果音量因子使用线性乘法 `sample * volume`，当 volume 接近 16384（14 位定点的 1.0）且 sample 接近 32767 时，中间结果 `32767 * 16384 = 536854528` 超出 int32 精度后右移 14 位仍可能产生非预期值。更常见的错误是用 `int16_t` 存储中间结果导致溢出。

**解决**：

```cpp
// 错误：中间结果用 16 位存储
int16_t mixed = (int16_t)(sample * volume);  // 溢出！

// 正确：用 32 位存储中间结果，再右移并饱和截断
int32_t product = (int32_t)sample * volume;   // volume 是 14 位定点
int32_t result = product >> 14;               // 还原到 16 位范围

// 饱和截断（防止极端值溢出）
if (result > 32767)  result = 32767;
if (result < -32768) result = -32768;
int16_t output = (int16_t)result;
```

> 项目中 `MixAudioS16CPP`（`WaveMixer.cpp` 第 36-57 行）使用非线性混音公式避免直接乘法溢出，将两路信号的混合结果约束在合理范围内。

### 错误 2：Oboe 回调线程中执行阻塞操作导致音频卡顿

**现象**：Android 设备上播放音频时出现不规则的卡顿和延迟，Logcat 中可见 `W/Oboe: underrun detected` 警告。

**原因**：Oboe 的 `onAudioReady` 回调在高优先级实时线程中执行，该线程有严格的时间限制（通常 < 5ms）。如果在回调中执行内存分配（`new`/`malloc`）、文件 I/O、日志输出、锁竞争等阻塞操作，会导致缓冲区欠载（underrun）。

**解决**：

```cpp
// 错误：回调中分配内存和打印日志
oboe::DataCallbackResult onAudioReady(
    oboe::AudioStream *stream, void *audioData, int32_t numFrames) {
    
    auto *buffer = new float[numFrames * channels];  // 禁止！内存分配
    LOGI("填充 %d 帧", numFrames);                    // 禁止！日志 I/O
    
    std::lock_guard<std::mutex> lock(dataMutex);     // 危险！可能阻塞
    // ...
    delete[] buffer;
    return oboe::DataCallbackResult::Continue;
}

// 正确：预分配缓冲区，使用无锁队列
class AudioCallback : public oboe::AudioStreamDataCallback {
    float preAllocBuffer[MAX_FRAMES * MAX_CHANNELS];  // 预分配
    LockFreeQueue<AudioChunk> dataQueue;               // 无锁队列

    oboe::DataCallbackResult onAudioReady(
        oboe::AudioStream *stream, void *audioData, int32_t numFrames) {
        
        auto *output = static_cast<float *>(audioData);
        AudioChunk chunk;
        if (dataQueue.tryPop(chunk)) {
            memcpy(output, chunk.data, numFrames * channels * sizeof(float));
        } else {
            memset(output, 0, numFrames * channels * sizeof(float));  // 静音
        }
        return oboe::DataCallbackResult::Continue;
    }
};
```

> 项目中 `tTVPAudioRendererOboe`（`WaveMixer.cpp` 第 411-485 行）的 `onAudioReady` 直接从预填充的混音缓冲区复制数据，不做任何分配或 I/O。

### 错误 3：Pan 值计算错误导致左右声道音量不平衡

**现象**：设置 Pan=0（居中）时左右声道音量不一致，或设置 Pan 到极端值时另一个声道没有完全静音。

**原因**：Pan 值通常范围为 -10000~10000（KiriKiri 规范），需要正确映射到左右声道的增益因子。常见错误是线性映射而非使用等功率曲线（equal-power panning），或者 Pan 范围边界处理不当。

**解决**：

```cpp
// 错误：线性映射（Pan 居中时总功率下降 3dB）
float leftGain  = (10000.0f - pan) / 20000.0f;   // pan=0 → 0.5
float rightGain = (10000.0f + pan) / 20000.0f;    // pan=0 → 0.5
// 问题：居中时每声道只有 50% 音量，听感偏小

// 正确：等功率曲线（居中时保持原始响度）
float panNorm = (pan + 10000.0f) / 20000.0f;     // 归一化到 [0, 1]
float angle = panNorm * M_PI * 0.5f;              // 映射到 [0, π/2]
float leftGain  = cosf(angle);                     // 居中时 cos(π/4) ≈ 0.707
float rightGain = sinf(angle);                     // 居中时 sin(π/4) ≈ 0.707
// 0.707² + 0.707² = 1.0，总功率恒定

// 项目中 OpenAL 后端的做法（WaveMixer.cpp 第 631-663 行）：
// 直接使用 alSource3f(AL_POSITION, x, 0, z) 设置空间位置
// 让 OpenAL 的距离衰减模型自动处理 Pan
```

---

## 9. 对照项目源码

### 关键文件清单

| 文件路径 | 行号范围 | 内容说明 |
|----------|----------|----------|
| `cpp/core/sound/win32/WaveMixer.cpp` | 36-57 | S16非线性混音模板 |
| `cpp/core/sound/win32/WaveMixer.cpp` | 59-83 | F32浮点混音模板 |
| `cpp/core/sound/win32/WaveMixer.cpp` | 101-125 | 音量/Pan存储与RecalcVolume |
| `cpp/core/sound/win32/WaveMixer.cpp` | 368-389 | FillBuffer混音调用 |
| `cpp/core/sound/win32/WaveMixer.cpp` | 411-485 | Oboe渲染器完整实现 |
| `cpp/core/sound/win32/WaveMixer.cpp` | 631-663 | OpenAL音量/Pan/Position |
| `cpp/core/sound/win32/WaveMixer.cpp` | 769-784 | 渲染器工厂函数 |

### 代码导读

1. **混音核心**: 第36-57行的`MixAudioS16CPP`是理解整个音量系统的关键，非线性混音公式在此实现

2. **音量预计算**: 第112-125行的`RecalcVolume()`将浮点音量/Pan转换为定点数数组，避免运行时浮点运算

3. **Oboe集成**: 第411-485行展示了如何使用Google Oboe库，注意Builder模式和回调接口

---

## 10. 本节小结

- **14位定点数音量**：平衡精度与性能，使用`>> 14`代替浮点除法
- **非线性混音**：公式`A + B - A*B/32768`防止削波，保持动态范围
- **Pan实现差异**：软件混音用声道衰减，OpenAL用3D位置模拟
- **Oboe优势**：Android低延迟音频，回调模式，精确时间戳
- **渲染器选择**：Android优先Oboe，其他平台使用OpenAL

---

## 11. 练习题与答案

### 题目1：定点数精度分析

已知14位定点数音量`MAX_VOLUME = 16384`，请计算：
1. 音量最小可调分辨率是多少dB？
2. 当用户设置`volume = 0.001`时，实际的定点数值是多少？这个精度够用吗？

<details>
<summary>查看答案</summary>

1. **最小分辨率计算**：
   - 定点数步长：1/16384 ≈ 0.000061
   - dB计算：20 * log10(1/16384) ≈ -84.3 dB
   - 每步约为 84.3/16384 ≈ 0.005 dB
   - 人耳可分辨约1dB变化，所以精度足够

2. **volume = 0.001的情况**：
   - 定点数值：0.001 * 16384 = 16.384 → 取整为16
   - 实际音量：16/16384 = 0.000977
   - 误差：(0.001 - 0.000977) / 0.001 = 2.3%
   - 在极低音量时精度下降，但通常不影响使用（接近静音）

</details>

### 题目2：非线性混音算法验证

编写程序验证非线性混音不会溢出16位范围。测试两个最大正值样本的混合结果。

<details>
<summary>查看答案</summary>

```cpp
#include <iostream>
#include <cstdint>
#include <cassert>

int mixNonLinear(int a, int b) {
    int result;
    if(a > 0 && b > 0) {
        result = a + b - ((a * b + 0x8000) >> 15);
    } else if(a < 0 && b < 0) {
        result = a + b + ((a * b) >> 15);
    } else {
        result = a + b;
    }
    return result;
}

int main() {
    // 测试最大正值
    int maxPos = 32767;
    int resultPP = mixNonLinear(maxPos, maxPos);
    std::cout << "Max + Max = " << resultPP << std::endl;
    // 输出: 32768 (略超，实际会被截断到32767)
    
    // 测试最大负值
    int maxNeg = -32768;
    int resultNN = mixNonLinear(maxNeg, maxNeg);
    std::cout << "Min + Min = " << resultNN << std::endl;
    // 输出: -32768 (恰好在范围内)
    
    // 测试一正一负（直接相加）
    int resultPN = mixNonLinear(maxPos, maxNeg);
    std::cout << "Max + Min = " << resultPN << std::endl;
    // 输出: -1 (抵消)
    
    // 验证范围
    assert(resultPP <= 32768 && resultPP >= -32768);
    assert(resultNN <= 32768 && resultNN >= -32768);
    assert(resultPN <= 32767 && resultPN >= -32768);
    
    std::cout << "All tests passed!" << std::endl;
    return 0;
}
```

</details>

### 题目3：实现Oboe回调

假设你需要在Oboe回调中实现一个简单的正弦波发生器，频率440Hz，采样率48000Hz，立体声输出。请实现`onAudioReady`回调函数。

<details>
<summary>查看答案</summary>

```cpp
#include <cmath>

class SineGenerator : public oboe::AudioStreamCallback {
    double _phase = 0.0;
    const double _frequency = 440.0;      // Hz
    const double _sampleRate = 48000.0;   // Hz
    const double _amplitude = 0.5;        // 50% volume
    
public:
    oboe::DataCallbackResult onAudioReady(
        oboe::AudioStream *stream,
        void *audioData,
        int32_t numFrames) override 
    {
        // 计算相位增量
        double phaseIncrement = 2.0 * M_PI * _frequency / _sampleRate;
        
        // 根据格式选择处理方式
        if(stream->getFormat() == oboe::AudioFormat::Float) {
            float* buffer = static_cast<float*>(audioData);
            for(int i = 0; i < numFrames; ++i) {
                float sample = static_cast<float>(_amplitude * std::sin(_phase));
                
                // 立体声：左右声道相同
                *buffer++ = sample;  // 左声道
                *buffer++ = sample;  // 右声道
                
                // 更新相位
                _phase += phaseIncrement;
                if(_phase >= 2.0 * M_PI) {
                    _phase -= 2.0 * M_PI;
                }
            }
        } else {  // I16格式
            int16_t* buffer = static_cast<int16_t*>(audioData);
            for(int i = 0; i < numFrames; ++i) {
                int16_t sample = static_cast<int16_t>(
                    _amplitude * 32767.0 * std::sin(_phase));
                
                *buffer++ = sample;  // 左声道
                *buffer++ = sample;  // 右声道
                
                _phase += phaseIncrement;
                if(_phase >= 2.0 * M_PI) {
                    _phase -= 2.0 * M_PI;
                }
            }
        }
        
        return oboe::DataCallbackResult::Continue;
    }
};
```

</details>

---

## 下一步

[01-WaveLoopManager循环管理](../04-循环与DSP/01-WaveLoopManager循环管理.md) — 学习音频循环点管理与无缝循环实现
