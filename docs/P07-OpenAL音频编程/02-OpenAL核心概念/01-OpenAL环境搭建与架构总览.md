# OpenAL 环境搭建与架构总览

> **所属模块：** P07-OpenAL音频编程
> **前置知识：** [P01-现代CMake与构建工具链](../../P01-现代CMake与构建工具链/README.md)、[P02-vcpkg包管理](../../P02-vcpkg包管理/README.md)、[本模块第1章](../01-数字音频基础/)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：

1. 在 Windows/Linux/macOS 上通过 vcpkg 安装 OpenAL Soft 并配置 CMake 项目
2. 理解 OpenAL 与 OpenGL 的 API 设计对应关系，举一反三
3. 编写并成功运行第一个 OpenAL 程序：枚举音频设备并播放正弦波
4. 理解 OpenAL 的状态机模型和错误检查机制
5. 阅读 KrKr2 项目中的 OpenAL 集成代码并理解其 CMake 配置

## OpenAL 是什么

### 历史与定位

OpenAL（Open Audio Library）最初由 Loki Software 于 2000 年设计，目标是为 3D 音频提供与 **OpenGL 设计风格一致的** 跨平台 API。原始参考实现由 Creative Labs 维护，已多年未更新。目前业界广泛使用的是 **OpenAL Soft** —— 开源纯软件实现，支持 Windows、Linux、macOS、Android、iOS，不依赖特定硬件。

| 特性 | 说明 |
|------|------|
| 跨平台 | Windows (WASAPI/DirectSound)、Linux (PulseAudio/ALSA/JACK)、macOS (CoreAudio)、Android (OpenSL ES) |
| 3D 音频 | HRTF（Head-Related Transfer Function）支持，模拟真实空间听觉 |
| 多后端 | 自动检测最佳音频后端，也可手动指定 |
| 浮点支持 | 原生支持 32-bit float 格式（通过扩展） |
| 低延迟 | 可配置缓冲区大小以满足实时需求 |

### OpenAL vs OpenGL：API 设计对应

如果你学过 OpenGL，理解 OpenAL 非常轻松。两者 API 设计高度一致：

| OpenGL 概念 | OpenAL 对应 | 说明 |
|-------------|-------------|------|
| Display Connection | `ALCdevice` | 连接到音频/图形硬件 |
| GL Context | `ALCcontext` | 渲染/音频状态容器 |
| Texture | `ALuint` (Buffer) | 保存图像/音频数据 |
| Geometry/Mesh | `ALuint` (Source) | 发出图形/声音的实体 |
| Camera | Listener | 观察者/听者 |
| `glGen*` / `glDelete*` | `alGen*` / `alDelete*` | 生成/删除对象 |
| `glBind*` | — (直接操作 ID) | OpenAL 不需要 bind |
| `glGetError` | `alGetError` | 错误检查 |

关键差异：OpenGL 使用 bind-to-edit 模式（先绑定再操作），而 **OpenAL 直接通过对象 ID 操作**，更接近现代 Vulkan/DirectX 12 的设计。

### 前缀命名规范

| 前缀 | 含义 | 示例 |
|------|------|------|
| `al` | 核心 AL 函数（Source/Buffer/Listener） | `alGenSources`, `alBufferData` |
| `alc` | AL Context 函数（Device/Context） | `alcOpenDevice`, `alcCreateContext` |
| `AL_` | AL 常量 | `AL_POSITION`, `AL_FORMAT_MONO16` |
| `ALC_` | ALC 常量 | `ALC_DEVICE_SPECIFIER` |

> **记忆技巧：** 涉及设备和上下文管理用 `alc`（c = context），操作音频对象用 `al`。

### 状态机模型

与 OpenGL 类似，OpenAL 也是**状态机架构**：

```
全局状态
├── 当前 ALCdevice*    ← alcOpenDevice 设置
├── 当前 ALCcontext*   ← alcMakeContextCurrent 设置
└── AL 线程本地状态
    ├── 最后一次错误码  ← alGetError() 读取并清除
    ├── Listener 属性   ← alListenerf/fv 设置
    ├── 距离模型        ← alDistanceModel 设置
    └── 多普勒参数      ← alDopplerFactor/Velocity 设置
```

**关键规则：** 一个线程在同一时刻只能有一个"当前上下文"，所有 `al*` 函数都隐式作用于当前上下文。这与 OpenGL 的 `glMakeCurrent` 机制完全一致。

## 环境搭建

### vcpkg 安装（推荐）

```bash
# Windows / Linux / macOS 通用
vcpkg install openal-soft
vcpkg list | grep openal   # 验证: openal-soft:x64-windows  1.23.1#2
```

**CMakeLists.txt 配置（与 KrKr2 项目一致）：**

```cmake
cmake_minimum_required(VERSION 3.19)
project(openal_tutorial LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
find_package(OpenAL CONFIG REQUIRED)                          # vcpkg 自动处理路径
add_executable(${PROJECT_NAME} main.cpp)
target_link_libraries(${PROJECT_NAME} PRIVATE OpenAL::OpenAL) # 同 KrKr2 sound/CMakeLists.txt:52
```

### 系统包管理器安装（备选）

```bash
sudo apt install libopenal-dev          # Ubuntu / Debian
sudo dnf install openal-soft-devel      # Fedora / RHEL
brew install openal-soft                 # macOS — 注意系统自带 OpenAL.framework 已废弃
vcpkg install openal-soft:arm64-android  # Android
```

### 平台音频后端一览

| 平台 | 默认后端 | 注意事项 |
|------|---------|---------|
| Windows | WASAPI (Win7+) / DirectSound (XP) | 开箱即用 |
| Linux | PulseAudio → ALSA（自动回退） | 需 `libasound2-dev` 或 `libpulse-dev` |
| macOS | CoreAudio | 系统 OpenAL.framework 已废弃，用 vcpkg 版 |
| Android | OpenSL ES | KrKr2 用 Oboe 而非 OpenAL（`WaveMixer.cpp:769`） |

## 第一个 OpenAL 程序

### 示例 1：枚举音频设备

```cpp
// enumerate_devices.cpp — 列出所有可用 OpenAL 音频设备
#include <AL/al.h>
#include <AL/alc.h>
#include <cstdio>
#include <cstring>

int main() {
    // 检查设备枚举扩展
    if (alcIsExtensionPresent(nullptr, "ALC_ENUMERATION_EXT") == ALC_FALSE) {
        std::printf("设备枚举扩展不可用\n");
        return 1;
    }
    // 设备名称列表：连续字符串，以双 '\0' 结尾
    const ALCchar* ptr = alcGetString(nullptr, ALC_DEVICE_SPECIFIER);
    std::printf("=== 可用音频输出设备 ===\n");
    for (int i = 0; *ptr; ++i, ptr += std::strlen(ptr) + 1)
        std::printf("  [%d] %s\n", i, ptr);

    std::printf("\n默认设备: %s\n",
                alcGetString(nullptr, ALC_DEFAULT_DEVICE_SPECIFIER));

    // 打开默认设备并查询参数
    ALCdevice* dev = alcOpenDevice(nullptr);
    if (!dev) { std::printf("错误：无法打开音频设备\n"); return 1; }

    ALCint freq = 0;
    alcGetIntegerv(dev, ALC_FREQUENCY, 1, &freq);
    std::printf("设备采样率: %d Hz\n", freq);

    alcCloseDevice(dev);
    return 0;
}
```

**典型输出（Windows）：**
```
=== 可用音频输出设备 ===
  [0] OpenAL Soft on Speakers (Realtek High Definition Audio)
  [1] OpenAL Soft on HDMI Audio Output

默认设备: OpenAL Soft on Speakers (Realtek High Definition Audio)
设备采样率: 44100 Hz
```

### 示例 2：Hello World — 播放 440Hz 正弦波

```cpp
// beep.cpp — 生成 440Hz 正弦波并通过 OpenAL 播放 1 秒
#include <AL/al.h>
#include <AL/alc.h>
#include <cmath>
#include <cstdio>
#include <cstdint>
#include <vector>
#include <thread>
#include <chrono>

#define AL_CHECK(stmt) do { stmt; ALenum err = alGetError(); \
    if (err != AL_NO_ERROR) std::printf("AL error 0x%04X at %s:%d\n", \
        err, __FILE__, __LINE__); } while(0)

int main() {
    ALCdevice*  device  = alcOpenDevice(nullptr);       // 1. 打开设备
    if (!device) { std::printf("无法打开设备\n"); return 1; }
    ALCcontext* context = alcCreateContext(device, nullptr); // 2. 创建上下文
    if (!context || alcMakeContextCurrent(context) == ALC_FALSE) {
        alcCloseDevice(device); return 1;
    }
    alGetError();  // 清除初始错误

    // 3. 生成 PCM：44100Hz，1秒，单声道 16-bit
    constexpr int kRate = 44100, kSamples = kRate;
    constexpr float kFreq = 440.0f, kPI = 3.14159265f;
    std::vector<int16_t> pcm(kSamples);
    for (int i = 0; i < kSamples; ++i)
        pcm[i] = int16_t(0.8f * 32767.0f * std::sin(2.0f*kPI*kFreq*float(i)/kRate));

    // 4. Buffer → 填数据 → Source → 绑定 → 播放
    ALuint buffer = 0, source = 0;
    AL_CHECK(alGenBuffers(1, &buffer));
    AL_CHECK(alBufferData(buffer, AL_FORMAT_MONO16, pcm.data(),
                          ALsizei(pcm.size()*sizeof(int16_t)), kRate));
    AL_CHECK(alGenSources(1, &source));
    AL_CHECK(alSourcei(source, AL_BUFFER, ALint(buffer)));
    AL_CHECK(alSourcePlay(source));
    std::printf("正在播放 440Hz 正弦波...\n");

    // 5. 轮询等待播放完成
    for (ALint st = AL_PLAYING; st == AL_PLAYING; ) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        alGetSourcei(source, AL_SOURCE_STATE, &st);
    }
    // 6. 清理（Source → Buffer → Context → Device）
    alDeleteSources(1, &source);  alDeleteBuffers(1, &buffer);
    alcMakeContextCurrent(nullptr); alcDestroyContext(context); alcCloseDevice(device);
    std::printf("播放完成\n");
}
```

**CMakeLists.txt（编译两个示例）：**

```cmake
cmake_minimum_required(VERSION 3.19)
project(openal_ch02_01 LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
find_package(OpenAL CONFIG REQUIRED)

add_executable(enumerate_devices enumerate_devices.cpp)
target_link_libraries(enumerate_devices PRIVATE OpenAL::OpenAL)
add_executable(beep beep.cpp)
target_link_libraries(beep PRIVATE OpenAL::OpenAL)
```

## 错误检查机制

### alGetError() — 读取即清除

```cpp
ALenum err = alGetError();
// 1. 返回上一次错误码（或 AL_NO_ERROR）
// 2. 将内部错误状态重置为 AL_NO_ERROR
```

**所有错误码：**

| 错误码 | 值 | 含义 | 常见原因 |
|--------|-----|------|---------|
| `AL_NO_ERROR` | 0x0000 | 无错误 | — |
| `AL_INVALID_NAME` | 0xA001 | 无效对象 ID | 未生成或已删除的 Buffer/Source |
| `AL_INVALID_ENUM` | 0xA002 | 无效枚举值 | 错误的 format 或 param |
| `AL_INVALID_VALUE` | 0xA003 | 无效参数值 | 负数频率、超范围 gain |
| `AL_INVALID_OPERATION` | 0xA004 | 非法操作 | 无当前上下文 |
| `AL_OUT_OF_MEMORY` | 0xA005 | 内存不足 | 缓冲区太大 |

### 示例 3：错误检查工具封装

```cpp
// al_utils.h — Debug 模式自动启用错误检查
#pragma once
#include <AL/al.h>
#include <AL/alc.h>
#include <cstdio>

inline const char* alErrStr(ALenum e) {
    switch (e) {
        case AL_INVALID_NAME:      return "INVALID_NAME";
        case AL_INVALID_ENUM:      return "INVALID_ENUM";
        case AL_INVALID_VALUE:     return "INVALID_VALUE";
        case AL_INVALID_OPERATION: return "INVALID_OPERATION";
        case AL_OUT_OF_MEMORY:     return "OUT_OF_MEMORY";
        default:                   return "NO_ERROR";
    }
}
inline const char* alcErrStr(ALCenum e) {
    switch (e) {
        case ALC_INVALID_DEVICE:  return "INVALID_DEVICE";
        case ALC_INVALID_CONTEXT: return "INVALID_CONTEXT";
        case ALC_INVALID_ENUM:    return "INVALID_ENUM";
        case ALC_INVALID_VALUE:   return "INVALID_VALUE";
        case ALC_OUT_OF_MEMORY:   return "OUT_OF_MEMORY";
        default:                  return "NO_ERROR";
    }
}
#ifndef NDEBUG
  #define AL_CHECK(s) do { s; ALenum _e = alGetError(); \
      if (_e != AL_NO_ERROR) std::fprintf(stderr, \
          "[AL] %s (0x%04X) at %s:%d\n", alErrStr(_e), _e, __FILE__, __LINE__); } while(0)
  #define ALC_CHECK(dev, s) do { s; ALCenum _e = alcGetError(dev); \
      if (_e != ALC_NO_ERROR) std::fprintf(stderr, \
          "[ALC] %s (0x%04X) at %s:%d\n", alcErrStr(_e), _e, __FILE__, __LINE__); } while(0)
#else
  #define AL_CHECK(s)        s
  #define ALC_CHECK(dev, s)  s
#endif
```

> **KrKr2 做法：** 仅 debug 模式启用 `alGetError()` 检查（`WaveMixer.cpp` `AppendBuffer` 末尾 `#ifdef _DEBUG` 块），Release 省略以提升性能。

### 示例 4：故意触发各种错误

```cpp
// error_demo.cpp — 演示 OpenAL 错误码
#include "al_utils.h"
int main() {
    ALCdevice* dev = alcOpenDevice(nullptr);
    ALCcontext* ctx = alcCreateContext(dev, nullptr);
    alcMakeContextCurrent(ctx);
    alGetError();

    AL_CHECK(alSourcePlay(99999));                              // AL_INVALID_NAME
    ALuint buf; alGenBuffers(1, &buf);
    AL_CHECK(alBufferData(buf, 0xFFFF, nullptr, 0, 44100));     // AL_INVALID_ENUM
    int16_t d = 0;
    AL_CHECK(alBufferData(buf, AL_FORMAT_MONO16, &d, 2, -1));   // AL_INVALID_VALUE
    alcMakeContextCurrent(nullptr);
    AL_CHECK(alGenBuffers(1, &buf));                             // AL_INVALID_OPERATION

    alcMakeContextCurrent(ctx);
    alDeleteBuffers(1, &buf);
    alcMakeContextCurrent(nullptr);
    alcDestroyContext(ctx);
    alcCloseDevice(dev);
}
```

### 示例 5：设备信息查询

```cpp
// device_info.cpp — 查询设备名称、Source 上限、采样率
#include <AL/al.h>
#include <AL/alc.h>
#include <cstdio>
int main() {
    ALCdevice* dev = alcOpenDevice(nullptr);
    if (!dev) return 1;
    ALCcontext* ctx = alcCreateContext(dev, nullptr);
    alcMakeContextCurrent(ctx);

    std::printf("设备: %s\n", alcGetString(dev, ALC_DEVICE_SPECIFIER));
    ALCint mono = 0, stereo = 0, freq = 0;
    alcGetIntegerv(dev, ALC_MONO_SOURCES, 1, &mono);
    alcGetIntegerv(dev, ALC_STEREO_SOURCES, 1, &stereo);
    alcGetIntegerv(dev, ALC_FREQUENCY, 1, &freq);
    std::printf("Mono Sources: %d  Stereo Sources: %d  采样率: %d Hz\n",
                mono, stereo, freq);

    alcMakeContextCurrent(nullptr);
    alcDestroyContext(ctx);
    alcCloseDevice(dev);
}
```

## 常见错误与排查

| 症状 | 原因 | 解决 |
|------|------|------|
| `undefined reference to 'alcOpenDevice'` | CMake 缺少链接配置 | 添加 `find_package(OpenAL CONFIG REQUIRED)` + `target_link_libraries(... OpenAL::OpenAL)` |
| 程序运行但无声音 | 未设置上下文 / 系统静音 / Linux 缺 PulseAudio | 检查返回值 + `alGetError()` + 系统音量 + `systemctl --user status pulseaudio` |
| `AL_INVALID_OPERATION` (0xA004) | 调用 `al*` 前未 `alcMakeContextCurrent` | 确保上下文激活在所有 `al*` 调用之前 |

## 动手实践

创建 `openal_tutorial/` 目录，放入上述 5 个 `.cpp` 文件和 `al_utils.h`，编译运行：

```bash
# Windows — cmake + vcpkg
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%/scripts/buildsystems/vcpkg.cmake
cmake --build build --config Release
# Linux / macOS — 同理，用 $VCPKG_ROOT
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake && cmake --build build
```

验证：`enumerate_devices` 能列出设备 → `beep` 能听到 440Hz → `error_demo` 显示错误码 → `device_info` 输出系统参数。

## 对照项目源码

KrKr2 中 OpenAL 初始化位于 `cpp/core/sound/win32/WaveMixer.cpp`（第 716-730 行）：

```cpp
// tTVPAudioRendererAL 构造函数
alcIsExtensionPresent(NULL, "ALC_ENUMERATION_EXT"); // 检查扩展，但直接开默认设备
ALCdevice* _device = alcOpenDevice(NULL);
ALCcontext* _context = alcCreateContext(_device, NULL);
alcMakeContextCurrent(_context);
// 注意：未检查 alcCreateContext 返回值——潜在隐患
```

| 方面 | 本节教程 | KrKr2 实际代码 |
|------|---------|---------------|
| 设备选择 | 枚举所有 + 打开默认 | 仅打开默认设备 |
| 错误检查 | 每步都检查 | 仅 `#ifdef _DEBUG` |
| 资源释放 | 完整反序释放 | 在析构函数中释放 |
| Context 检查 | 检查返回值 | 未检查（风险） |

## 本节小结

- **OpenAL Soft** 是当前业界标准实现，支持 Windows/Linux/macOS/Android
- API 与 **OpenGL 高度一致**：al/alc 前缀、状态机模型、`GetError` 错误检查
- 关键区别：OpenAL **不需要 bind**，直接通过对象 ID 操作
- 环境搭建推荐 **vcpkg + CMake**：`find_package(OpenAL CONFIG REQUIRED)`
- 错误检查用 **alGetError()**（读取即清除），开发阶段建议用 `AL_CHECK` 宏包装
- **alcGetError(device)** 与 **alGetError()** 是不同函数：前者需要 device 指针

## 练习题与答案

### 题目 1：OpenAL 与 OpenGL 的 API 差异

列出 OpenAL 与 OpenGL 在 API 设计上至少 3 个相同点和 2 个不同点。

<details>
<summary>查看答案</summary>

**相同点：** ① Device/Context 模型管理硬件 ② `Gen/Delete` 创建销毁对象 ③ `GetError()` 读取即清除错误检查

**不同点：** ① OpenGL bind-to-edit，OpenAL 直接通过 ID 操作 ② OpenAL 有全局唯一 Listener，OpenGL 无对应"观察者"

</details>

### 题目 2：修复错误代码

以下代码有 3 处错误，找出并修复：

```cpp
int main() {
    ALCdevice* device = alcOpenDevice(nullptr);
    ALCcontext* ctx = alcCreateContext(device, nullptr);
    // 错误 1: 缺少什么？
    ALuint buf; alGenBuffers(1, &buf);
    int16_t data[44100] = {};
    alBufferData(buf, AL_FORMAT_MONO16, data, sizeof(data), 44100);
    ALuint src; alGenSources(1, &src);
    alSourcei(src, AL_BUFFER, buf); alSourcePlay(src);
    // 错误 2: 播放后立即退出会怎样？
    alcCloseDevice(device); alcDestroyContext(ctx);  // 错误 3: 顺序？
    alDeleteSources(1, &src); alDeleteBuffers(1, &buf);
}
```

<details>
<summary>查看答案</summary>

**错误 1：** 缺少 `alcMakeContextCurrent(ctx);`，后续 `al*` 调用全部产生 `AL_INVALID_OPERATION`。

**错误 2：** 播放后立即退出，Source 来不及发声就被销毁。需要等待：
```cpp
ALint state = AL_PLAYING;
while (state == AL_PLAYING) {
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    alGetSourcei(src, AL_SOURCE_STATE, &state);
}
```

**错误 3：** 释放顺序完全颠倒！正确顺序（先创建的后释放）：
```cpp
alDeleteSources(1, &src);
alDeleteBuffers(1, &buf);
alcMakeContextCurrent(nullptr);
alcDestroyContext(ctx);
alcCloseDevice(device);
```

</details>

### 题目 3：扩展设备枚举

修改 `enumerate_devices.cpp`，使其同时列出**输出设备**和**录音设备**。提示：录音设备用 `ALC_CAPTURE_DEVICE_SPECIFIER` 和 `ALC_CAPTURE_DEFAULT_DEVICE_SPECIFIER`。

<details>
<summary>查看答案</summary>

```cpp
#include <AL/al.h>
#include <AL/alc.h>
#include <cstdio>
#include <cstring>

void listDevices(const char* label, ALCenum specifier, ALCenum defaultSpec) {
    const ALCchar* ptr = alcGetString(nullptr, specifier);
    std::printf("=== %s ===\n", label);
    for (int i = 0; *ptr; ++i, ptr += std::strlen(ptr) + 1)
        std::printf("  [%d] %s\n", i, ptr);
    std::printf("  默认: %s\n\n", alcGetString(nullptr, defaultSpec));
}

int main() {
    if (!alcIsExtensionPresent(nullptr, "ALC_ENUMERATION_EXT")) {
        std::printf("枚举扩展不可用\n"); return 1;
    }
    listDevices("输出设备", ALC_DEVICE_SPECIFIER,
                ALC_DEFAULT_DEVICE_SPECIFIER);
    listDevices("录音设备", ALC_CAPTURE_DEVICE_SPECIFIER,
                ALC_CAPTURE_DEFAULT_DEVICE_SPECIFIER);
}
```

</details>

---

## 下一步

[下一节：Device → Context → Source → Buffer](02-Device-Context-Source-Buffer.md) — 深入理解 OpenAL 四层对象模型、生命周期管理和多上下文场景。
