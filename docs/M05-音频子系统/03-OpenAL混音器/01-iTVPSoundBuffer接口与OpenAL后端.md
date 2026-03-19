# iTVPSoundBuffer 接口与 OpenAL 后端

> **所属模块：** M05-音频子系统
> **前置知识：** [P07-OpenAL音频编程](../../P07-OpenAL音频编程/README.md)、[02-解码器注册机制](../02-解码器注册机制/README.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 `iTVPSoundBuffer` 抽象接口的设计意图与方法契约
2. 掌握 KrKr2 中三种音频渲染器（SDL、OpenAL、Oboe）的选择策略
3. 深入理解 `tTVPSoundBufferAL` 类如何封装 OpenAL Source/Buffer 机制
4. 实现一个自定义的音频缓冲区类

## 接口设计哲学

KrKr2 音频子系统采用 **策略模式（Strategy Pattern）** 分离接口与实现。`iTVPSoundBuffer` 定义了与具体音频后端无关的统一操作契约，而 `tTVPSoundBuffer`（软件混音）和 `tTVPSoundBufferAL`（OpenAL 硬件加速）提供具体实现。

```
┌─────────────────────────────────────────────────────────────┐
│                    上层（TJS2 脚本）                         │
│               WaveSoundBuffer.play() 等                     │
└─────────────────────────┬───────────────────────────────────┘
                          │ 调用
                          ▼
┌─────────────────────────────────────────────────────────────┐
│               iTVPSoundBuffer 接口                          │
│  Play() / Pause() / Stop() / SetVolume() / AppendBuffer()  │
└─────────────────────────┬───────────────────────────────────┘
                          │ 实现
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ tTVPSoundBuffer │ │tTVPSoundBufferAL│ │   (Oboe 变体)   │
│   (软件混音)     │ │ (OpenAL 后端)   │ │  (Android 专用) │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

### 为什么需要接口抽象

1. **跨平台兼容**：不同平台的音频 API 差异巨大（Windows DirectSound、Android Oboe、桌面 OpenAL），统一接口屏蔽这些差异
2. **运行时切换**：可根据设备能力在软件混音和硬件加速之间选择
3. **测试友好**：可注入 Mock 实现进行单元测试
4. **扩展性**：未来新增音频后端（如 PulseAudio、WASAPI）只需实现接口

## iTVPSoundBuffer 接口详解

接口定义在 `cpp/core/sound/win32/WaveMixer.h` 第 15-57 行：

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.h
class iTVPSoundBuffer {
public:
    virtual ~iTVPSoundBuffer() = default;

    // 生命周期管理
    virtual void Release() = 0;         // 释放资源，等同 delete this

    // 播放控制
    virtual void Play() = 0;            // 开始播放
    virtual void Pause() = 0;           // 暂停（保持播放位置）
    virtual void Stop() = 0;            // 停止（重置播放位置）
    virtual void Reset() = 0;           // 清空所有缓冲区数据

    // 状态查询
    virtual bool IsPlaying() = 0;       // 是否正在播放
    virtual bool IsBufferValid() = 0;   // 是否还能接受新数据

    // 音量与声像
    virtual void SetVolume(float v) = 0;  // 音量 [0.0, 1.0]
    virtual float GetVolume() = 0;
    virtual void SetPan(float v) = 0;     // 声像 [-1.0(左), 1.0(右)]
    virtual float GetPan() = 0;

    // 缓冲区操作（核心方法）
    virtual void AppendBuffer(const void *buf, unsigned int len) = 0;

    // 播放进度与延迟
    virtual tjs_uint GetCurrentPlaySamples() = 0;  // 已播放采样数
    virtual tjs_uint GetLatencySamples() = 0;      // 缓冲延迟（采样数）
    virtual float GetLatencySeconds() { return 0; }
    virtual int GetRemainBuffers() = 0;            // 剩余待播放缓冲区数

    // 3D 音频定位（可选实现）
    virtual void SetPosition(float x, float y, float z) {
        // 默认不实现（2D 音频）
    }
};
```

### 方法契约详解

#### 播放控制三元组：Play/Pause/Stop

这三个方法的语义必须严格遵守：

| 方法 | 当前状态 | 调用后状态 | 播放位置 | 缓冲区数据 |
|------|----------|------------|----------|------------|
| `Play()` | 任意 | 播放中 | 从当前位置继续 | 保持 |
| `Pause()` | 播放中 | 暂停 | 冻结当前位置 | 保持 |
| `Stop()` | 任意 | 停止 | 重置为 0 | **清空** |

关键点：`Stop()` 不仅停止播放，还会调用 `Reset()` 清空所有缓冲数据。这意味着 Stop 后若要重新播放，必须重新填充数据。

```cpp
// Stop() 的典型实现
void tTVPSoundBuffer::Stop() override {
    _playing = false;
    Reset();  // 关键：清空缓冲区
}
```

#### AppendBuffer：流式数据填充

这是接口中最重要的方法，支持流式音频播放：

```cpp
virtual void AppendBuffer(const void *buf, unsigned int len) = 0;
```

**参数说明：**
- `buf`：PCM 数据指针，格式必须与创建时指定的 `tTVPWaveFormat` 一致
- `len`：数据长度（字节数），不是采样数！

**调用时机：**
1. 播放前：预填充 2-4 个缓冲区
2. 播放中：在回调/轮询中持续填充

**返回缓冲区可用性：**

```cpp
// 典型的流式填充循环
while(playing) {
    if(soundBuffer->IsBufferValid()) {
        int decoded = decoder->Render(pcmBuffer, bufferSize);
        if(decoded > 0) {
            soundBuffer->AppendBuffer(pcmBuffer, decoded);
        }
    }
    std::this_thread::sleep_for(10ms);
}
```

#### GetLatencySamples：延迟计算

音频延迟（Latency）是从数据送入缓冲区到实际从扬声器播出的时间差。对于同步音画至关重要：

```cpp
// 计算实际播放位置（考虑延迟）
tjs_uint actualPosition = GetCurrentPlaySamples() - GetLatencySamples();

// 转换为时间（秒）
float positionSeconds = actualPosition / (float)sampleRate;
```

## 音频渲染器层次结构

KrKr2 使用 `iTVPAudioRenderer` 作为渲染器基类，管理多个 `iTVPSoundBuffer` 实例：

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第 235-343 行
class iTVPAudioRenderer {
protected:
    SDL_AudioSpec _spec;                           // 输出格式规格
    std::mutex _streams_mtx;                       // 流列表互斥锁
    std::unordered_set<tTVPSoundBuffer *> _streams; // 活跃音频流集合
    int _frame_size = 0;                           // 单帧字节数

public:
    iTVPAudioRenderer() {
        memset(&_spec, 0, sizeof(_spec));
        _spec.freq = 48000;       // 默认 48kHz
        _spec.format = AUDIO_S16; // 默认 16-bit signed
        _spec.channels = 2;       // 立体声
        // 注册回调函数
        _spec.callback = [](void *p, Uint8 *s, int l) {
            memset(s, 0, l);  // 先静音
            ((iTVPAudioRenderer *)p)->FillBuffer(s, l);  // 再混音
        };
        _spec.userdata = this;
    }

    // 创建音频流（虚方法，子类可重写）
    virtual tTVPSoundBuffer *CreateStream(tTVPWaveFormat &fmt, int bufcount);

    // 混音回调：遍历所有流并混合
    void FillBuffer(Uint8 *buf, int len) {
        std::lock_guard<std::mutex> lk(_streams_mtx);
        for(tTVPSoundBuffer *s : _streams) {
            s->FillBuffer(buf, len);  // 每个流独立混合
        }
    }
};
```

### 渲染器选择策略

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第 769-784 行
static iTVPAudioRenderer *CreateAudioRenderer() {
    iTVPAudioRenderer *renderer = nullptr;
#ifdef __ANDROID__
    // Android 优先使用 Oboe（低延迟）
    renderer = new tTVPAudioRendererOboe;
    if(renderer->Init())
        return renderer;
    delete renderer;  // Oboe 失败则回退
#elif defined(_MSC_VER) && 0
    // Windows SDL 路径（当前禁用）
    renderer = new tTVPAudioRendererSDL;
    renderer->Init();
    return renderer;
#endif
    // 默认：OpenAL（跨平台）
    renderer = new tTVPAudioRendererAL;
    renderer->Init();
    return renderer;
}
```

**平台选择总结：**

| 平台 | 首选渲染器 | 回退方案 | 原因 |
|------|-----------|----------|------|
| Android | Oboe | OpenAL | Oboe 延迟更低（AAudio/OpenSL ES） |
| Windows | OpenAL | SDL | DirectSound 废弃，OpenAL 兼容性好 |
| macOS | OpenAL | - | macOS 原生支持 OpenAL |
| Linux | OpenAL | - | PulseAudio/ALSA 通过 OpenAL 封装 |

## tTVPSoundBufferAL：OpenAL 后端实现

这是 KrKr2 桌面平台的主力音频后端，位于 `WaveMixer.cpp` 第 489-701 行：

```cpp
class tTVPSoundBufferAL : public tTVPSoundBuffer {
    typedef tTVPSoundBuffer inherit;

    // OpenAL 资源
    ALuint _alSource;           // OpenAL 音源
    ALenum _alFormat;           // 音频格式（AL_FORMAT_STEREO16 等）
    ALuint *_bufferIds;         // 缓冲区 ID 数组（主）
    ALuint *_bufferIds2;        // 缓冲区 ID 数组（用于 Unqueue）
    tjs_uint *_bufferSize;      // 每个缓冲区的实际数据大小
    tjs_uint _bufferCount;      // 缓冲区总数
    int _bufferIdx = -1;        // 当前填充的缓冲区索引
    tTVPWaveFormat _format;     // 音频格式信息
};
```

### 构造与初始化

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第 501-536 行
tTVPSoundBufferAL(tTVPWaveFormat &desired, int bufcount) :
    tTVPSoundBuffer(desired.BytesPerSample * desired.Channels, nullptr),
    _bufferCount(bufcount) 
{
    // 分配缓冲区 ID 数组
    _bufferIds = new ALuint[bufcount];
    _bufferIds2 = new ALuint[bufcount];
    _bufferSize = new tjs_uint[bufcount];
    _format = desired;

    // 创建 OpenAL 资源
    alGenSources(1, &_alSource);           // 创建一个音源
    alGenBuffers(_bufferCount, _bufferIds); // 创建 N 个缓冲区
    alSourcef(_alSource, AL_GAIN, 1.0f);   // 初始音量 100%

    // 根据通道数和位深选择 OpenAL 格式
    if(desired.Channels == 1) {
        switch(desired.BitsPerSample) {
            case 8:  _alFormat = AL_FORMAT_MONO8; break;
            case 16: _alFormat = AL_FORMAT_MONO16; break;
            default: assert(false);  // 不支持的格式
        }
    } else if(desired.Channels == 2) {
        switch(desired.BitsPerSample) {
            case 8:  _alFormat = AL_FORMAT_STEREO8; break;
            case 16: _alFormat = AL_FORMAT_STEREO16; break;
            default: assert(false);
        }
    }
}
```

**OpenAL 格式映射表：**

| 通道 | 位深 | OpenAL 格式常量 |
|------|------|-----------------|
| 1 | 8 | `AL_FORMAT_MONO8` |
| 1 | 16 | `AL_FORMAT_MONO16` |
| 2 | 8 | `AL_FORMAT_STEREO8` |
| 2 | 16 | `AL_FORMAT_STEREO16` |

> ⚠️ **注意**：标准 OpenAL 不支持 32-bit 浮点和多通道（5.1/7.1）格式。KrKr2 对这些格式使用软件混音器（`tTVPSoundBuffer`）并转换为 16-bit 立体声输出。

### AppendBuffer：环形缓冲队列

OpenAL 使用 **队列缓冲区（Buffer Queuing）** 模式，`AppendBuffer` 的实现是整个类的核心：

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第 556-594 行
void AppendBuffer(const void *buf, unsigned int len) override {
    if(len <= 0) return;
    std::lock_guard<std::mutex> lk(_buffer_mtx);

    // 第一步：回收已播放完的缓冲区
    ALint processed = 0;
    alGetSourcei(_alSource, AL_BUFFERS_PROCESSED, &processed);
    if(processed > 0) {
        // 从队列头部移除已播放的缓冲区
        alSourceUnqueueBuffers(_alSource, processed, _bufferIds2);
        
        // 更新已发送采样数统计
        for(int i = 0; i < processed; ++i) {
            for(int j = 0; j < _bufferCount; ++j) {
                if(_bufferIds[j] == _bufferIds2[i]) {
                    _sendedSamples += _bufferSize[j] / _frame_size;
                    break;
                }
            }
        }
    }

    // 第二步：检查队列是否已满
    ALint queued = 0;
    alGetSourcei(_alSource, AL_BUFFERS_QUEUED, &queued);
    if(queued >= _bufferCount) return;  // 队列已满，拒绝新数据

    // 第三步：填充下一个缓冲区
    ++_bufferIdx;
    if(_bufferIdx >= _bufferCount) _bufferIdx = 0;  // 环形索引
    
    ALuint bufid = _bufferIds[_bufferIdx];
    alBufferData(bufid, _alFormat, buf, len, _format.SamplesPerSec);
    alSourceQueueBuffers(_alSource, 1, &bufid);
    _bufferSize[_bufferIdx] = len;  // 记录实际大小
}
```

**工作流程图：**

```
初始状态：所有 4 个缓冲区空闲
┌───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │  _bufferIds
└───┴───┴───┴───┘
  ↑
 空闲

AppendBuffer() x 3 后：
┌───┬───┬───┬───┐
│ Q │ Q │ Q │   │  Q = Queued（已入队）
└───┴───┴───┴───┘
          ↑
        下次填充位置

Play() 开始播放后：
┌───┬───┬───┬───┐
│ P │ Q │ Q │   │  P = Playing（正在播放）
└───┴───┴───┴───┘
  ↑
 硬件正在读取

缓冲区 0 播放完毕：
┌───┬───┬───┬───┐
│ ✓ │ P │ Q │   │  ✓ = Processed（可回收）
└───┴───┴───┴───┘

下次 AppendBuffer() 时：
1. alSourceUnqueueBuffers() 回收缓冲区 0
2. alBufferData() 填充缓冲区 0（或新数据到缓冲区 3）
3. alSourceQueueBuffers() 重新入队
```

### 播放控制实现

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第 611-629 行
void Play() override {
    ALenum state;
    alGetSourcei(_alSource, AL_SOURCE_STATE, &state);
    if(state != AL_PLAYING) {
        alSourcePlay(_alSource);  // 仅在非播放状态时启动
    }
    _playing = true;
}

void Pause() override {
    alSourcePause(_alSource);
    _playing = false;
}

void Stop() override {
    alSourceStop(_alSource);
    Reset();          // 清空缓冲区
    _bufferIdx = -1;  // 重置索引
    _playing = false;
}

void Reset() override {
    inherit::Reset();
    std::lock_guard<std::mutex> lk(_buffer_mtx);
    alSourceRewind(_alSource);         // 倒回起点
    alSourcei(_alSource, AL_BUFFER, 0); // 解绑所有缓冲区
}
```

### 延迟计算

OpenAL 提供精确的播放位置查询：

```cpp
// 文件: cpp/core/sound/win32/WaveMixer.cpp 第 672-700 行
tjs_uint GetLatencySamples() override {
    std::lock_guard<std::mutex> lk(_buffer_mtx);
    ALint offset = 0, queued = 0;
    
    // 获取当前缓冲区内的字节偏移
    alGetSourcei(_alSource, AL_BYTE_OFFSET, &offset);
    // 获取队列中的缓冲区数量
    alGetSourcei(_alSource, AL_BUFFERS_QUEUED, &queued);
    
    int remainBuffers = queued;
    if(remainBuffers == 0) return 0;
    
    // 计算所有待播放数据的总字节数
    tjs_int total = -offset;  // 减去当前缓冲区已播放部分
    for(int i = 0; i < remainBuffers; ++i) {
        int idx = _bufferIdx + 1 - remainBuffers + i;
        // 环形索引处理
        if(idx >= _bufferCount) idx -= _bufferCount;
        else if(idx < 0) idx += _bufferCount;
        total += _bufferSize[idx];
    }
    return total / _frame_size;  // 转换为采样数
}

tjs_uint GetCurrentPlaySamples() override {
    ALint offset = 0;
    alGetSourcei(_alSource, AL_SAMPLE_OFFSET, &offset);
    return _sendedSamples + offset;  // 已完成缓冲区 + 当前缓冲区偏移
}
```

## 动手实践：实现简化版 OpenAL 播放器

下面我们从零实现一个简化版的 OpenAL 播放器，帮助理解接口设计：

```cpp
// simple_al_player.cpp
#include <AL/al.h>
#include <AL/alc.h>
#include <cstdint>
#include <cstring>
#include <iostream>
#include <thread>
#include <chrono>
#include <cmath>

// 简化版 SoundBuffer 接口
class ISimpleSoundBuffer {
public:
    virtual ~ISimpleSoundBuffer() = default;
    virtual void play() = 0;
    virtual void stop() = 0;
    virtual bool isPlaying() = 0;
    virtual void appendBuffer(const void* data, size_t bytes) = 0;
    virtual void setVolume(float v) = 0;
};

// OpenAL 实现
class ALSoundBuffer : public ISimpleSoundBuffer {
    ALuint source_;
    ALuint buffers_[4];       // 4 个缓冲区
    int bufferIdx_ = 0;
    ALenum format_;
    ALsizei sampleRate_;
    int frameSize_;
    
public:
    ALSoundBuffer(int channels, int bitsPerSample, int sampleRate)
        : sampleRate_(sampleRate)
        , frameSize_(channels * bitsPerSample / 8)
    {
        // 选择格式
        if (channels == 1) {
            format_ = (bitsPerSample == 8) ? AL_FORMAT_MONO8 : AL_FORMAT_MONO16;
        } else {
            format_ = (bitsPerSample == 8) ? AL_FORMAT_STEREO8 : AL_FORMAT_STEREO16;
        }
        
        // 创建 OpenAL 资源
        alGenSources(1, &source_);
        alGenBuffers(4, buffers_);
        alSourcef(source_, AL_GAIN, 1.0f);
        
        std::cout << "[ALSoundBuffer] 创建完成: " 
                  << channels << "ch, " 
                  << bitsPerSample << "bit, " 
                  << sampleRate << "Hz\n";
    }
    
    ~ALSoundBuffer() override {
        stop();
        alDeleteBuffers(4, buffers_);
        alDeleteSources(1, &source_);
    }
    
    void play() override {
        ALenum state;
        alGetSourcei(source_, AL_SOURCE_STATE, &state);
        if (state != AL_PLAYING) {
            alSourcePlay(source_);
        }
    }
    
    void stop() override {
        alSourceStop(source_);
        // 清空队列
        ALint queued;
        alGetSourcei(source_, AL_BUFFERS_QUEUED, &queued);
        if (queued > 0) {
            ALuint temp[4];
            alSourceUnqueueBuffers(source_, queued, temp);
        }
        bufferIdx_ = 0;
    }
    
    bool isPlaying() override {
        ALenum state;
        alGetSourcei(source_, AL_SOURCE_STATE, &state);
        return state == AL_PLAYING;
    }
    
    void appendBuffer(const void* data, size_t bytes) override {
        // 回收已播放缓冲区
        ALint processed;
        alGetSourcei(source_, AL_BUFFERS_PROCESSED, &processed);
        if (processed > 0) {
            ALuint temp[4];
            alSourceUnqueueBuffers(source_, processed, temp);
        }
        
        // 检查队列是否已满
        ALint queued;
        alGetSourcei(source_, AL_BUFFERS_QUEUED, &queued);
        if (queued >= 4) {
            std::cout << "[警告] 缓冲区队列已满，丢弃数据\n";
            return;
        }
        
        // 填充并入队
        ALuint buf = buffers_[bufferIdx_];
        alBufferData(buf, format_, data, bytes, sampleRate_);
        alSourceQueueBuffers(source_, 1, &buf);
        
        bufferIdx_ = (bufferIdx_ + 1) % 4;
    }
    
    void setVolume(float v) override {
        alSourcef(source_, AL_GAIN, v);
    }
};

// 生成正弦波测试数据
void generateSineWave(int16_t* buffer, int samples, int sampleRate, float frequency) {
    for (int i = 0; i < samples; ++i) {
        float t = (float)i / sampleRate;
        float value = std::sin(2.0f * 3.14159f * frequency * t);
        buffer[i * 2] = buffer[i * 2 + 1] = (int16_t)(value * 32767 * 0.5f);
    }
}

int main() {
    // 初始化 OpenAL
    ALCdevice* device = alcOpenDevice(nullptr);
    if (!device) {
        std::cerr << "无法打开音频设备\n";
        return 1;
    }
    
    ALCcontext* context = alcCreateContext(device, nullptr);
    alcMakeContextCurrent(context);
    
    std::cout << "OpenAL 初始化成功\n";
    std::cout << "设备: " << alcGetString(device, ALC_DEVICE_SPECIFIER) << "\n\n";
    
    // 创建 SoundBuffer（立体声 16-bit 48kHz）
    ALSoundBuffer soundBuffer(2, 16, 48000);
    
    // 生成测试数据（1 秒 440Hz 正弦波）
    const int samplesPerBuffer = 4800;  // 100ms
    const int bufferBytes = samplesPerBuffer * 4;  // 立体声 16-bit
    int16_t* pcmBuffer = new int16_t[samplesPerBuffer * 2];
    
    // 预填充 3 个缓冲区
    for (int i = 0; i < 3; ++i) {
        generateSineWave(pcmBuffer, samplesPerBuffer, 48000, 440.0f);
        soundBuffer.appendBuffer(pcmBuffer, bufferBytes);
    }
    
    // 开始播放
    soundBuffer.play();
    std::cout << "开始播放 440Hz 正弦波...\n";
    
    // 持续填充（模拟流式播放）
    for (int round = 0; round < 20; ++round) {  // 2 秒
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        
        // 变化频率（演示动态效果）
        float freq = 440.0f + round * 20.0f;
        generateSineWave(pcmBuffer, samplesPerBuffer, 48000, freq);
        soundBuffer.appendBuffer(pcmBuffer, bufferBytes);
        
        std::cout << "填充缓冲区: " << freq << "Hz\n";
    }
    
    // 等待播放完毕
    while (soundBuffer.isPlaying()) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
    
    std::cout << "播放结束\n";
    
    // 清理
    delete[] pcmBuffer;
    alcMakeContextCurrent(nullptr);
    alcDestroyContext(context);
    alcCloseDevice(device);
    
    return 0;
}
```

**编译与运行：**

```bash
# Linux/macOS
g++ -std=c++17 simple_al_player.cpp -o player -lopenal -lm
./player

# Windows (MSYS2)
g++ -std=c++17 simple_al_player.cpp -o player.exe -lopenal32

# 使用 CMake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(simple_al_player)
find_package(OpenAL REQUIRED)
add_executable(player simple_al_player.cpp)
target_link_libraries(player PRIVATE OpenAL::OpenAL)
```

## 常见错误及解决方案

### 错误 1：OpenAL Source 状态查询与缓冲区填充不同步

**现象**：播放过程中偶尔出现短暂静音或爆音，调试发现 `alSourceUnqueueBuffers` 返回 `AL_INVALID_OPERATION`。

**原因**：在 Source 处于 `AL_PLAYING` 状态时，尝试 Unqueue 尚未播放完毕的 Buffer。`alSourceUnqueueBuffers` 只能取回已经播放完成的 Buffer，但代码没有先查询 `AL_BUFFERS_PROCESSED` 来确认有多少 Buffer 可以取回。

**解决**：

```cpp
// 错误写法：直接 Unqueue 全部 Buffer
alSourceUnqueueBuffers(source, NUM_BUFFERS, bufferIds);
// 可能触发 AL_INVALID_OPERATION，因为部分 Buffer 还在播放中

// 正确写法：先查询已处理数量，再逐个取回
ALint processed = 0;
alGetSourcei(source, AL_BUFFERS_PROCESSED, &processed);
while (processed > 0) {
    ALuint buf;
    alSourceUnqueueBuffers(source, 1, &buf);
    // 重新填充 buf 并 Queue 回去
    fillBuffer(buf);
    alSourceQueueBuffers(source, 1, &buf);
    processed--;
}
```

> 这正是项目中 `tTVPSoundBufferAL::AppendBuffer()`（`WaveMixer.cpp` 第 560-590 行）采用的策略：先用 `alGetSourcei(AL_BUFFERS_PROCESSED)` 查询，再安全 Unqueue。

### 错误 2：OpenAL 设备 / 上下文泄漏导致后续音频功能失效

**现象**：应用退出后重新启动，OpenAL 报 `ALC_INVALID_DEVICE` 或系统音频服务被占用。在 Android 上表现为第二次进入游戏时完全无声。

**原因**：`alcCloseDevice` / `alcDestroyContext` 调用顺序错误，或在 Context 仍为 Current 时直接关闭 Device。OpenAL 规范要求：先取消当前 Context → 销毁 Context → 关闭 Device。

**解决**：

```cpp
// 错误顺序：
alcCloseDevice(device);      // Context 还在引用 Device！
alcDestroyContext(context);   // 为时已晚

// 正确顺序：
alcMakeContextCurrent(nullptr);   // 1. 取消当前 Context
alcDestroyContext(context);        // 2. 销毁 Context
alcCloseDevice(device);            // 3. 关闭 Device
context = nullptr;
device = nullptr;
```

> 项目中 `tTVPAudioRendererAL` 析构函数（`WaveMixer.cpp` 第 730-750 行）严格按此顺序执行清理。

### 错误 3：NUM_BUFFERS 设置过小导致 Source 饥饿停止

**现象**：CPU 负载较高时播放频繁卡顿，日志中可见 Source 状态从 `AL_PLAYING` 自动变为 `AL_STOPPED`，需要手动调用 `alSourcePlay` 重启。

**原因**：流式播放使用的 Buffer 队列数量不足（如仅 2 个），解码线程稍有延迟就导致所有 Buffer 都被播放完毕，OpenAL 自动将 Source 停止（这是 OpenAL 规范行为——当队列中所有 Buffer 都已处理且没有新 Buffer 入队时，Source 进入 STOPPED 状态）。

**解决**：

```cpp
// 项目默认值（WaveMixer.cpp 第 489 行）
static const int NUM_BUFFERS = 4;   // 4 个缓冲区轮转

// 如果在低端设备上仍然卡顿，可适当增加：
static const int NUM_BUFFERS = 6;   // 给解码线程更多缓冲余量

// 同时需要检测并恢复 Source 饥饿：
ALint state;
alGetSourcei(source, AL_SOURCE_STATE, &state);
if (state == AL_STOPPED && !userRequestedStop) {
    // Source 因缓冲区饥饿而停止，重新启动
    alSourcePlay(source);
    printf("警告: Source 饥饿停止，已自动重启\n");
}
```

> 项目的 `tTVPSoundBufferAL::AppendBuffer()` 末尾（`WaveMixer.cpp` 第 588-590 行）正是通过检测 `AL_STOPPED` 状态来自动恢复播放。

---

## 对照项目源码

### 核心文件

| 文件 | 行号 | 内容 |
|------|------|------|
| `cpp/core/sound/win32/WaveMixer.h` | 15-57 | `iTVPSoundBuffer` 接口定义 |
| `cpp/core/sound/win32/WaveMixer.cpp` | 101-233 | `tTVPSoundBuffer` 软件混音实现 |
| `cpp/core/sound/win32/WaveMixer.cpp` | 489-701 | `tTVPSoundBufferAL` OpenAL 实现 |
| `cpp/core/sound/win32/WaveMixer.cpp` | 703-753 | `tTVPAudioRendererAL` OpenAL 设备管理 |
| `cpp/core/sound/win32/WaveMixer.cpp` | 769-784 | `CreateAudioRenderer()` 渲染器选择 |

### 关键设计点

1. **`_bufferIds` 与 `_bufferIds2` 双数组**：`_bufferIds` 存储所有缓冲区 ID，`_bufferIds2` 作为 `alSourceUnqueueBuffers` 的输出数组，避免破坏原始顺序

2. **`_bufferSize` 数组**：OpenAL 不存储每个缓冲区的实际数据大小，需要自行维护以计算延迟

3. **互斥锁保护**：所有涉及 `_bufferIdx` 和 OpenAL API 的操作都用 `_buffer_mtx` 保护，因为 `FillBuffer` 可能在音频回调线程中调用

## 本节小结

- `iTVPSoundBuffer` 是 KrKr2 音频播放的核心抽象，定义了播放控制、音量、缓冲区操作等统一接口
- KrKr2 根据平台选择不同渲染器：Android 优先 Oboe，其他平台使用 OpenAL
- `tTVPSoundBufferAL` 使用 OpenAL 的 Buffer Queuing 机制实现流式播放
- 缓冲区回收、填充、入队的循环是流式音频的核心模式
- 延迟计算需要累加所有待播放缓冲区的数据量，减去当前播放偏移

## 练习题与答案

### 题目 1：为什么 Stop() 要调用 Reset()？

如果 `Stop()` 只是暂停播放而不清空缓冲区，会有什么问题？

<details>
<summary>查看答案</summary>

`Stop()` 必须调用 `Reset()` 清空缓冲区，原因：

1. **语义一致性**：Stop 表示"完全停止"，用户再次 Play 时期望从头开始，而不是从上次暂停位置继续

2. **避免脏数据**：如果不清空，下次播放可能先播放旧的残留数据，造成"杂音"

3. **资源管理**：及时释放已缓冲的 PCM 数据，避免内存占用

4. **与 Pause 区分**：
   - `Pause()` = 暂停，保持位置和数据，可恢复
   - `Stop()` = 停止，重置一切，需重新填充

如果只想暂停而保留数据，应该使用 `Pause()`。

</details>

### 题目 2：实现 SetPan 的 3D 定位

当前代码使用 `AL_POSITION` 实现声像（Pan），这样做有什么优缺点？如何改进？

```cpp
void SetPan(float pan) override {
    float sourcePosAL[] = { pan, 0.0f, 0.0f };
    alSourcefv(_alSource, AL_POSITION, sourcePosAL);
}
```

<details>
<summary>查看答案</summary>

**当前实现分析：**

使用 3D 位置模拟 2D Pan 是一种简便方法，但有局限：

优点：
- 代码简单，利用 OpenAL 内置的 3D 衰减模型
- 不需要手动计算左右声道音量

缺点：
- 依赖 Listener 位置（默认在原点），如果 Listener 被移动会出问题
- 距离衰减会影响音量（非纯 Pan 效果）
- 无法精确控制衰减曲线

**改进方案：使用 AL_EXT_STEREO_ANGLES 或手动混音**

```cpp
// 方案 1：禁用距离衰减，纯位置 Pan
void SetPan(float pan) override {
    // 禁用距离模型
    alSourcef(_alSource, AL_ROLLOFF_FACTOR, 0.0f);
    // 设置位置
    float sourcePosAL[] = { pan, 0.0f, 0.0f };
    alSourcefv(_alSource, AL_POSITION, sourcePosAL);
}

// 方案 2：使用 AL_GAIN 和 AL_SOURCE_RELATIVE（最精确）
// 需要支持 AL_EXT_source_distance_model 扩展
void SetPan(float pan) override {
    // 切换到相对定位模式
    alSourcei(_alSource, AL_SOURCE_RELATIVE, AL_TRUE);
    // 使用恒等增益模型
    alSourcef(_alSource, AL_ROLLOFF_FACTOR, 0.0f);
    
    // 计算左右增益（等功率 Pan）
    float angle = (pan + 1.0f) * 0.25f * 3.14159f;  // 0 到 π/2
    float leftGain = std::cos(angle);
    float rightGain = std::sin(angle);
    
    // OpenAL 原生不支持分通道增益，需要软件混音
    // 或使用 OpenAL Soft 的 AL_SOFT_direct_channels 扩展
}
```

KrKr2 当前方案是足够实用的折中，对于视觉小说场景（对话、BGM、SE）已经够用。

</details>

### 题目 3：设计一个支持淡入淡出的 SoundBuffer 包装器

基于 `iTVPSoundBuffer` 设计一个装饰器类，支持音量淡入/淡出效果。

<details>
<summary>查看答案</summary>

```cpp
#include <chrono>
#include <thread>
#include <atomic>

class FadingSoundBuffer : public iTVPSoundBuffer {
    iTVPSoundBuffer* inner_;  // 被装饰的原始缓冲区
    
    std::atomic<float> targetVolume_{1.0f};
    std::atomic<float> currentVolume_{1.0f};
    std::atomic<bool> fading_{false};
    std::thread fadeThread_;
    
public:
    FadingSoundBuffer(iTVPSoundBuffer* inner) : inner_(inner) {}
    
    ~FadingSoundBuffer() override {
        fading_ = false;
        if (fadeThread_.joinable()) {
            fadeThread_.join();
        }
        inner_->Release();
    }
    
    void Release() override { delete this; }
    
    // 代理所有基本方法
    void Play() override { inner_->Play(); }
    void Pause() override { inner_->Pause(); }
    void Stop() override { inner_->Stop(); }
    void Reset() override { inner_->Reset(); }
    bool IsPlaying() override { return inner_->IsPlaying(); }
    bool IsBufferValid() override { return inner_->IsBufferValid(); }
    void SetPan(float v) override { inner_->SetPan(v); }
    float GetPan() override { return inner_->GetPan(); }
    void AppendBuffer(const void* buf, unsigned int len) override {
        inner_->AppendBuffer(buf, len);
    }
    tjs_uint GetCurrentPlaySamples() override { 
        return inner_->GetCurrentPlaySamples(); 
    }
    tjs_uint GetLatencySamples() override { 
        return inner_->GetLatencySamples(); 
    }
    int GetRemainBuffers() override { 
        return inner_->GetRemainBuffers(); 
    }
    
    // 音量方法重写
    void SetVolume(float v) override {
        targetVolume_ = v;
        if (!fading_) {
            currentVolume_ = v;
            inner_->SetVolume(v);
        }
    }
    
    float GetVolume() override { return currentVolume_; }
    
    // 新增：淡入淡出
    void FadeTo(float targetVol, int durationMs) {
        // 停止之前的淡变
        fading_ = false;
        if (fadeThread_.joinable()) {
            fadeThread_.join();
        }
        
        targetVolume_ = targetVol;
        fading_ = true;
        
        fadeThread_ = std::thread([this, durationMs]() {
            const int steps = durationMs / 10;  // 每 10ms 更新一次
            float startVol = currentVolume_;
            float endVol = targetVolume_;
            
            for (int i = 1; i <= steps && fading_; ++i) {
                float t = (float)i / steps;
                // 使用指数曲线（更自然的听感）
                float vol = startVol + (endVol - startVol) * (1.0f - std::exp(-3.0f * t));
                currentVolume_ = vol;
                inner_->SetVolume(vol);
                std::this_thread::sleep_for(std::chrono::milliseconds(10));
            }
            
            if (fading_) {
                currentVolume_ = endVol;
                inner_->SetVolume(endVol);
            }
            fading_ = false;
        });
    }
    
    void FadeIn(int durationMs) {
        currentVolume_ = 0.0f;
        inner_->SetVolume(0.0f);
        FadeTo(1.0f, durationMs);
    }
    
    void FadeOut(int durationMs) {
        FadeTo(0.0f, durationMs);
    }
    
    void FadeOutAndStop(int durationMs) {
        FadeTo(0.0f, durationMs);
        // 启动监控线程
        std::thread([this, durationMs]() {
            std::this_thread::sleep_for(std::chrono::milliseconds(durationMs + 50));
            Stop();
        }).detach();
    }
};

// 使用示例
void example() {
    iTVPSoundBuffer* rawBuffer = /* 创建原始缓冲区 */;
    FadingSoundBuffer* fadingBuffer = new FadingSoundBuffer(rawBuffer);
    
    fadingBuffer->Play();
    fadingBuffer->FadeIn(1000);   // 1 秒淡入
    
    // ... 播放中 ...
    
    fadingBuffer->FadeOutAndStop(2000);  // 2 秒淡出后停止
}
```

</details>

## 下一步

下一节 [02-双层缓冲与解码线程](./02-双层缓冲与解码线程.md) 将深入分析 KrKr2 的 L1/L2 双层缓冲架构，以及解码线程如何与播放线程协作实现无缝流式播放。
