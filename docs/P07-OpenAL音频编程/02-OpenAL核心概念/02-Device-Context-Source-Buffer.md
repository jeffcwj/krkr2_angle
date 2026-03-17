# Device → Context → Source → Buffer

> **所属模块：** P07-OpenAL音频编程 | **前置知识：** [OpenAL 环境搭建与架构总览](01-OpenAL环境搭建与架构总览.md) | **阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：

1. 画出 OpenAL 四层对象模型（Device/Context/Source/Buffer）的层次关系
2. 正确管理创建与销毁顺序，理解 Source 状态机
3. 阅读 KrKr2 中 `tTVPAudioRendererAL` 和 `tTVPSoundBufferAL` 的对象管理代码

## 四层对象模型总览

OpenAL 的对象层次结构如下：

```
ALCdevice (硬件连接)
└── ALCcontext (状态容器)
    ├── Listener (唯一)
    ├── Source 1 → Buffer A
    ├── Source 2 → Buffer B
    └── Source 3 → Buffer A  ← 多 Source 可共享同一 Buffer
```

| 层级 | 类型 | 职责 | 创建 / 销毁 |
|------|------|------|------------|
| Device | `ALCdevice*` | 连接音频硬件 | `alcOpenDevice` / `alcCloseDevice` |
| Context | `ALCcontext*` | 保存所有音频状态 | `alcCreateContext` / `alcDestroyContext` |
| Source | `ALuint` | 声音发射点 | `alGenSources` / `alDeleteSources` |
| Buffer | `ALuint` | PCM 数据容器 | `alGenBuffers` / `alDeleteBuffers` |

## Device（设备）

Device 代表与底层音频硬件的连接，通常只打开一个。

### 打开与关闭

```cpp
ALCdevice* device = alcOpenDevice(nullptr);          // 默认设备
ALCdevice* device = alcOpenDevice("OpenAL Soft on HDMI"); // 指定设备
alcCloseDevice(device);  // 必须先销毁所有 Context
```

### 查询设备属性

```cpp
const char* name = alcGetString(device, ALC_DEVICE_SPECIFIER);
ALCint freq = 0, mono = 0, stereo = 0;
alcGetIntegerv(device, ALC_FREQUENCY, 1, &freq);
alcGetIntegerv(device, ALC_MONO_SOURCES, 1, &mono);
alcGetIntegerv(device, ALC_STEREO_SOURCES, 1, &stereo);
std::printf("频率=%dHz  Mono上限=%d  Stereo上限=%d\n", freq, mono, stereo);
```

> **典型值：** OpenAL Soft 默认 256 Mono Sources、44100Hz 混音频率。

## Context（上下文）

Context 是 OpenAL 的**状态容器**，保存 Listener 属性、距离模型等。一个 Device 可创建多个 Context，但每个线程只能有一个"当前 Context"。

### 创建与激活

```cpp
ALCcontext* context = alcCreateContext(device, nullptr);  // nullptr = 默认属性
alcMakeContextCurrent(context);  // 之后所有 al* 调用作用于此 Context
```

### 带属性创建

```cpp
ALCint attrs[] = {
    ALC_FREQUENCY, 48000,  // 混音采样率
    ALC_REFRESH,   100,    // 每秒混音次数（影响延迟）
    0                      // 以 0 结尾
};
ALCcontext* ctx = alcCreateContext(device, attrs);
```

### 上下文操作

```cpp
alcSuspendContext(context);   // 暂停处理（批量更新时用）
alcProcessContext(context);   // 恢复
ALCcontext* cur = alcGetCurrentContext();
ALCdevice* dev  = alcGetContextsDevice(context);
```

> **多线程：** 每个线程独立维护"当前 Context"，工作线程操作 OpenAL 前必须先 `alcMakeContextCurrent`。

## Source（声源）

Source 是 OpenAL 的**声音发射体**，每个 Source 有独立的位置、速度、增益和播放状态。

### 创建与删除

```cpp
ALuint source;
alGenSources(1, &source);    // 生成
alDeleteSources(1, &source); // 删除
// 批量：ALuint s[8]; alGenSources(8, s); alDeleteSources(8, s);
```

### Source 属性

| 属性 | 函数 | 默认值 | 说明 |
|------|------|--------|------|
| `AL_BUFFER` | `alSourcei` | 0 | 绑定的 Buffer |
| `AL_GAIN` | `alSourcef` | 1.0 | 音量（0=静音，>1=放大）|
| `AL_PITCH` | `alSourcef` | 1.0 | 播放速度/音高 |
| `AL_POSITION` | `alSourcefv` | {0,0,0} | 3D 位置 |
| `AL_VELOCITY` | `alSourcefv` | {0,0,0} | 速度（多普勒效应）|
| `AL_LOOPING` | `alSourcei` | AL_FALSE | 循环播放 |
| `AL_SOURCE_RELATIVE` | `alSourcei` | AL_FALSE | 相对 Listener 定位 |

```cpp
alSourcef(source, AL_GAIN, 0.5f);          // 音量 50%
alSourcef(source, AL_PITCH, 1.2f);         // 加速 20%
float pos[] = {1.0f, 0.0f, -3.0f};
alSourcefv(source, AL_POSITION, pos);      // 3D 定位
alSourcei(source, AL_LOOPING, AL_TRUE);    // 循环
```

### Source 状态机

Source 有四种状态，通过播放控制函数转换：

```
Initial ──Play──→ Playing ──Pause──→ Paused
  ↑                  │                  │
  │←──Rewind────── Stopped ←──Stop──────┘
  │                  ↑
  │←──── 播放结束 ───┘（非循环时自动进入 Stopped）
```

```cpp
alSourcePlay(source);   // → Playing
alSourcePause(source);  // → Paused
alSourceStop(source);   // → Stopped
alSourceRewind(source); // → Initial

ALint state;
alGetSourcei(source, AL_SOURCE_STATE, &state);
// state ∈ {AL_INITIAL, AL_PLAYING, AL_PAUSED, AL_STOPPED}
```

### 示例 1：Source 状态转换测试

```cpp
// source_states.cpp — 验证 Source 状态机
#include <AL/al.h>
#include <AL/alc.h>
#include <cstdio>
#include <cstdint>
#include <cmath>
#include <vector>
#include <thread>
#include <chrono>

const char* stateStr(ALint s) {
    switch (s) {
        case AL_INITIAL: return "INITIAL"; case AL_PLAYING: return "PLAYING";
        case AL_PAUSED:  return "PAUSED";  case AL_STOPPED: return "STOPPED";
        default: return "UNKNOWN";
    }
}
void printState(ALuint src, const char* label) {
    ALint s; alGetSourcei(src, AL_SOURCE_STATE, &s);
    std::printf("  %-16s → %s\n", label, stateStr(s));
}

int main() {
    ALCdevice* dev = alcOpenDevice(nullptr);
    ALCcontext* ctx = alcCreateContext(dev, nullptr);
    alcMakeContextCurrent(ctx);

    constexpr int kRate = 44100, kN = kRate / 2;
    std::vector<int16_t> pcm(kN);
    for (int i = 0; i < kN; ++i)
        pcm[i] = int16_t(0.5f * 32767 * std::sin(6.2832f * 440.f * i / kRate));

    ALuint buf, src;
    alGenBuffers(1, &buf);
    alBufferData(buf, AL_FORMAT_MONO16, pcm.data(), kN * 2, kRate);
    alGenSources(1, &src);
    alSourcei(src, AL_BUFFER, buf);

    printState(src, "创建后");       // INITIAL
    alSourcePlay(src);  printState(src, "Play");   // PLAYING
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    alSourcePause(src); printState(src, "Pause");  // PAUSED
    alSourcePlay(src);  printState(src, "再 Play"); // PLAYING
    alSourceStop(src);  printState(src, "Stop");   // STOPPED
    alSourceRewind(src);printState(src, "Rewind"); // INITIAL

    alDeleteSources(1, &src); alDeleteBuffers(1, &buf);
    alcMakeContextCurrent(nullptr); alcDestroyContext(ctx); alcCloseDevice(dev);
}
```

## Buffer（缓冲区）

Buffer 是 PCM 音频数据容器，可被多个 Source 共享。

### 创建与填充

```cpp
ALuint buffer;
alGenBuffers(1, &buffer);
alBufferData(buffer, AL_FORMAT_MONO16, pcmData, dataSize, sampleRate);
// alBufferData 会复制数据，调用后原始指针可释放
```

### 音频格式

| 格式常量 | 声道数 | 位深 | 每采样字节数 | 说明 |
|---------|--------|------|-------------|------|
| `AL_FORMAT_MONO8` | 1 | 8 bit | 1 | 无符号 0-255，128=静音 |
| `AL_FORMAT_MONO16` | 1 | 16 bit | 2 | 有符号 -32768~32767 |
| `AL_FORMAT_STEREO8` | 2 | 8 bit | 2 | 左右交替无符号 |
| `AL_FORMAT_STEREO16` | 2 | 16 bit | 4 | 左右交替有符号 |

> **KrKr2 实现：** `WaveMixer.cpp` 第 537-549 行根据声道数和位深选择格式：
> ```cpp
> ALenum fmt;
> if (channels == 1)
>     fmt = (bitWidth == 8) ? AL_FORMAT_MONO8 : AL_FORMAT_MONO16;
> else
>     fmt = (bitWidth == 8) ? AL_FORMAT_STEREO8 : AL_FORMAT_STEREO16;
> ```
> 注意 OpenAL 原生只支持这四种整数格式。浮点格式需要 `AL_EXT_float32` 扩展。

### 查询 Buffer 属性

```cpp
ALint freq, bits, channels, size;
alGetBufferi(buffer, AL_FREQUENCY, &freq);
alGetBufferi(buffer, AL_BITS, &bits);
alGetBufferi(buffer, AL_CHANNELS, &channels);
alGetBufferi(buffer, AL_SIZE, &size);
double duration = double(size) / (freq * (bits / 8) * channels);  // 时长（秒）
```

### Buffer 共享与生命周期

多个 Source 可同时绑定同一 Buffer（节省内存），但不能删除被引用的 Buffer：

```cpp
alSourcei(source, AL_BUFFER, 0);    // 先解绑
alDeleteBuffers(1, &buffer);         // 再删除
```

## 资源生命周期管理

### 创建与销毁顺序

OpenAL 资源必须按层次顺序创建，按**逆序**销毁：

```
创建：Device → Context → MakeCurrent → Buffer → Source（绑定）
销毁：Source → Buffer → MakeCurrent(nullptr) → DestroyContext → CloseDevice
```

> **违反顺序的后果：**
> - 未取消激活 Context 就销毁 → OpenAL Soft 内部泄漏
> - 未解绑 Buffer 就删除 → `AL_INVALID_OPERATION`
> - 未销毁 Context 就关设备 → 行为未定义

### RAII 封装

手动管理创建/销毁顺序容易出错。C++ 的 RAII 模式可以自动保证销毁顺序和异常安全：

```cpp
// al_raii_demo.cpp — RAII 封装四层对象 + 使用
#include <AL/al.h>
#include <AL/alc.h>
#include <stdexcept>
#include <cstdio>
#include <cstdint>
#include <cmath>
#include <vector>
#include <thread>
#include <chrono>

// 每个 struct 构造时获取资源，析构时释放；禁止拷贝
struct ALDevice {
    ALCdevice* handle;
    explicit ALDevice(const char* n = nullptr) : handle(alcOpenDevice(n)) {
        if (!handle) throw std::runtime_error("alcOpenDevice 失败");
    }
    ~ALDevice() { if (handle) alcCloseDevice(handle); }
    ALDevice(const ALDevice&) = delete; ALDevice& operator=(const ALDevice&) = delete;
};
struct ALContext {
    ALCcontext* handle;
    explicit ALContext(ALCdevice* d) : handle(alcCreateContext(d, nullptr)) {
        if (!handle) throw std::runtime_error("alcCreateContext 失败");
        alcMakeContextCurrent(handle);
    }
    ~ALContext() { if (handle) { alcMakeContextCurrent(nullptr); alcDestroyContext(handle); } }
    ALContext(const ALContext&) = delete; ALContext& operator=(const ALContext&) = delete;
};
struct ALBuffer {
    ALuint id = 0;
    ALBuffer() { alGenBuffers(1, &id); }
    ~ALBuffer() { if (id) alDeleteBuffers(1, &id); }
    ALBuffer(const ALBuffer&) = delete; ALBuffer& operator=(const ALBuffer&) = delete;
};
struct ALSource {
    ALuint id = 0;
    ALSource() { alGenSources(1, &id); }
    ~ALSource() { if (id) { alSourceStop(id); alSourcei(id,AL_BUFFER,0); alDeleteSources(1,&id); } }
    ALSource(const ALSource&) = delete; ALSource& operator=(const ALSource&) = delete;
};

int main() {
    ALDevice dev;  ALContext ctx(dev.handle);  // ①② 自动打开+激活
    constexpr int kRate = 44100, kN = kRate / 2;
    std::vector<int16_t> pcm(kN);
    for (int i = 0; i < kN; ++i)
        pcm[i] = int16_t(0.3f * 32767 * std::sin(6.2832f * 660.f * i / kRate));
    ALBuffer buf;  alBufferData(buf.id, AL_FORMAT_MONO16, pcm.data(), kN*2, kRate);
    ALSource src;  alSourcei(src.id, AL_BUFFER, buf.id);
    alSourcePlay(src.id);
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    // 离开作用域 → 自动逆序销毁：Source → Buffer → Context → Device
}
```

> **手动管理 vs RAII：** 手动需 12 步、异常时泄漏、顺序易错。RAII 只需声明 4 个变量，C++ 栈逆序析构自动保证顺序和异常安全。

## 多 Context 场景

一个 Device 上可创建多个 Context 隔离音频状态（如游戏 3D 音效用 `AL_EXPONENT_DISTANCE`，UI 音效用 `AL_NONE`）：

```cpp
ALCcontext* ctxGame = alcCreateContext(dev, nullptr);
alcMakeContextCurrent(ctxGame);
alDistanceModel(AL_EXPONENT_DISTANCE);

ALCcontext* ctxUI = alcCreateContext(dev, nullptr);
alcMakeContextCurrent(ctxUI);
alDistanceModel(AL_NONE);

alcMakeContextCurrent(ctxGame);  // 切换回游戏音效
// ... 操作 Source ...
```

> Source 属于创建时的 Context，切换后不可访问。Buffer 在同一 Device 的不同 Context 间可共享。

## 常见错误与解决方案

### 错误 1：未激活 Context

```cpp
ALCcontext* ctx = alcCreateContext(dev, nullptr);
// 漏了 alcMakeContextCurrent(ctx) → 后续 al* 调用全部 AL_INVALID_OPERATION
```

### 错误 2：删除被引用的 Buffer

```cpp
// ❌ alDeleteBuffers 时 Source 仍绑定 → AL_INVALID_OPERATION
// ✅ 先 alSourcei(src, AL_BUFFER, 0) 解绑，再删
```

### 错误 3：销毁顺序错误

```cpp
// ❌ 先 alcCloseDevice 再 alcDestroyContext → 未定义行为
// ✅ MakeCurrent(nullptr) → DestroyContext → CloseDevice
```

## 动手实践

### 练习 A：对象生命周期验证器

编写程序用 `alGetError()` 逐步验证：正确创建顺序、故意不解绑就删 Buffer（应报错）、最后正确清理。关键工具函数：

```cpp
void checkAL(const char* op) {
    ALenum e = alGetError();
    std::printf("%-30s → %s\n", op, e == AL_NO_ERROR ? "OK" : "ERROR");
}
```

预期输出中，`alDeleteBuffers（未解绑）` 应显示 ERROR，解绑后再删应显示 OK。

### 练习 B：多 Source 共享 Buffer

创建 1 个 Buffer（任意频率正弦波），创建 3 个 Source 绑定同一 Buffer，分别设置 GAIN 为 0.3、0.6、1.0，同时播放验证音量差异。

## 对照项目源码

KrKr2 的 `tTVPSoundBufferAL`（`WaveMixer.cpp` 第 489-701 行）是四层模型的完整实例：

**初始化**（`tTVPAudioRendererAL` 构造函数，第 716-724 行）——标准 Device → Context → MakeCurrent 三步：

```cpp
ALCdevice* _device = alcOpenDevice(nullptr);
ALCcontext* _context = alcCreateContext(_device, nullptr);
alcMakeContextCurrent(_context);
```

**Source/Buffer 管理**（第 499-506 行）——用 4 个 Buffer 做流式队列（下一章详细讲）：

```cpp
alGenSources(1, &_alSource);
alGenBuffers(bufcount, _bufferIds);  // bufcount=4
```

**播放控制**（第 605-609 行）——与本节 Source 状态机一致：`alSourcePlay/_alSource`、`alSourcePause`、`alSourceStop`。

**属性设置**（第 644-656 行）——用 `AL_POSITION` 的 x 分量模拟左右声像（pan）：

```cpp
alSourcef(_alSource, AL_GAIN, volume);        // 音量
float pos[3] = { pan, 0.0f, 0.0f };
alSourcefv(_alSource, AL_POSITION, pos);       // 声像
```

## 本节小结

- 四层对象：Device → Context → Source → Buffer，创建自顶向下，销毁逆序
- Source 四态状态机：Initial/Playing/Paused/Stopped
- Buffer 支持 MONO8/MONO16/STEREO8/STEREO16，可被多 Source 共享
- RAII 封装自动保证销毁顺序和异常安全

## 练习题与答案

### 题目 1：对象层次关系

回答：(1) 一个 Device 最多能创建几个 Context？(2) 一个 Context 最多几个 Source？(3) 一个 Buffer 最多被几个 Source 引用？(4) 删除 Buffer 前必须做什么？

<details>
<summary>查看答案</summary>

1. **无上限**。每个线程同一时刻只能有一个"当前 Context"。
2. **取决于实现**。OpenAL Soft 默认 256 Mono，`alcGetIntegerv(dev, ALC_MONO_SOURCES, ...)` 查询。
3. **无上限**。Buffer 与 Source 分离是核心设计。
4. 必须先解绑所有 Source（`alSourcei(src, AL_BUFFER, 0)`），否则报 `AL_INVALID_OPERATION`。

</details>

### 题目 2：修复错误代码

找出以下代码的 3 处错误并修复：

```cpp
ALCdevice* dev = alcOpenDevice(nullptr);
ALCcontext* ctx = alcCreateContext(dev, nullptr);
// 错误 1
ALuint buf, src;
alGenBuffers(1, &buf);
int16_t data[44100] = {};
alBufferData(buf, AL_FORMAT_MONO16, data, sizeof(data), 44100);
alGenSources(1, &src);
alSourcei(src, AL_BUFFER, buf);
alSourcePlay(src);
// 清理
alDeleteBuffers(1, &buf);   // 错误 2
alDeleteSources(1, &src);   // 错误 3
alcDestroyContext(ctx);
alcCloseDevice(dev);
```

<details>
<summary>查看答案</summary>

**错误 1：** 缺少 `alcMakeContextCurrent(ctx);`——未激活 Context，后续 `al*` 调用全部失败。

**错误 2+3：** 销毁顺序不对且 Source 仍在播放。正确清理：

```cpp
alSourceStop(src);              // 先停止
alSourcei(src, AL_BUFFER, 0);   // 解绑 Buffer
alDeleteSources(1, &src);       // 先删 Source
alDeleteBuffers(1, &buf);       // 再删 Buffer
alcMakeContextCurrent(nullptr); // 取消激活
alcDestroyContext(ctx);
alcCloseDevice(dev);
```

</details>

### 题目 3：ALBuffer 移动语义

为上面的 `ALBuffer` 添加移动构造和移动赋值（可放入 `std::vector`）。

<details>
<summary>查看答案</summary>

```cpp
ALBuffer(ALBuffer&& o) noexcept : id(o.id) { o.id = 0; }
ALBuffer& operator=(ALBuffer&& o) noexcept {
    if (this != &o) { if (id) alDeleteBuffers(1, &id); id = o.id; o.id = 0; }
    return *this;
}
// 使用：vector<ALBuffer> v; v.emplace_back(); ALBuffer b = std::move(v[0]);
```

</details>

## 下一步

[下一节：坐标系与空间音频](03-坐标系与空间音频.md) — 学习 Listener 和 Source 的 3D 定位、距离衰减模型、多普勒效应。
