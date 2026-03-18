# M05 — 音频子系统

> **前置依赖：** M01-项目导览与环境搭建、P06-FFmpeg 音视频开发、P07-OpenAL 音频编程
> **目标：** 掌握 KrKr2 的音频架构，能添加新格式支持和修改 DSP 处理

## 模块概述

KrKr2 的音频子系统位于 `cpp/core/sound/`，CMake 目标名为 `core_sound_module`。它实现了完整的音频播放栈：从文件解码（Vorbis/Opus/FFmpeg）、PCM 格式转换、循环管理、DSP 滤波（PhaseVocoder 变速不变调），到最终通过 OpenAL 输出到硬件。Android 平台额外使用 Oboe 作为底层音频后端。

整个模块通过 TJS2 脚本引擎暴露 `WaveSoundBuffer` 类，游戏脚本可以用 TJS2 代码控制音频的播放、暂停、音量、淡入淡出和循环行为。

## 架构概览

```
TJS2 脚本层
  │  WaveSoundBuffer.open("bgm.ogg")
  │  WaveSoundBuffer.play()
  ▼
tTJSNC_WaveSoundBuffer          ← TJS2 原生类绑定
  │
  ▼
tTJSNI_WaveSoundBuffer          ← 播放器实现（win32/WaveImpl）
  ├── tTVPWaveDecoder            ← 解码器接口
  │   ├── FFWaveDecoder          ← FFmpeg 通用解码
  │   ├── VorbisWaveDecoder      ← Ogg Vorbis 解码
  │   └── OpusWaveDecoder        ← Opus 解码
  ├── tTVPWaveLoopManager        ← 循环/标签管理
  ├── tTVPSampleAndLabelSource   ← DSP 滤波链
  │   └── PhaseVocoderFilter     ← 变速不变调
  └── iTVPSoundBuffer            ← OpenAL 输出缓冲
      └── TVPCreateSoundBuffer() ← 平台音频输出
```

## 学习路径

```
sound 模块架构（目录结构 / 类层次 / 数据流）
  │
  ▼
解码器注册机制（WaveDecoder 接口 / Creator 注册 / 格式探测）
  │
  ▼
OpenAL 混音器（iTVPSoundBuffer / 缓冲管理 / 音量声道 / Oboe）
  │
  ▼
循环与 DSP（LoopManager / SegmentQueue / PhaseVocoder / FFT）
  │
  ▼
实战：添加新音频格式的完整流程
```

## 章节目录

### [01 — sound 模块架构](01-sound模块架构/)

| 节 | 标题 | 内容 |
|----|------|------|
| 01 | 目录结构与 CMake 构建 | sound/ 文件组织、CMakeLists.txt 依赖关系、平台条件编译 |
| 02 | 核心类层次与继承链 | tTVPWaveDecoder → tTJSNI_BaseSoundBuffer → tTJSNI_WaveSoundBuffer 完整继承链 |
| 03 | 音频播放数据流 | 从 TJS2 脚本 open()/play() 到 OpenAL 输出的完整调用链路 |

### [02 — 解码器注册机制](02-解码器注册机制/)

| 节 | 标题 | 内容 |
|----|------|------|
| 01 | tTVPWaveDecoder 接口详解 | 解码器抽象接口、tTVPWaveFormat PCM 格式结构、Render/SetPosition 协议 |
| 02 | Creator 注册与格式探测 | tTVPWaveDecoderCreator、TVPRegisterWaveDecoderCreator、按扩展名匹配流程 |
| 03 | Vorbis 与 FFmpeg 解码器实现 | VorbisWaveDecoder/OpusWaveDecoder/FFWaveDecoder 的具体实现分析 |

### [03 — OpenAL 混音器](03-OpenAL混音器/)

| 节 | 标题 | 内容 |
|----|------|------|
| 01 | iTVPSoundBuffer 接口与 OpenAL 后端 | iTVPSoundBuffer 抽象层、TVPCreateSoundBuffer 工厂、OpenAL Source/Buffer 管理 |
| 02 | 双层缓冲与解码线程 | L1/L2 缓冲区设计、tTVPWaveSoundBufferDecodeThread、FillBuffer 流程 |
| 03 | 音量控制与 Android Oboe 后端 | Volume/Pan/3D 定位、GlobalVolume、FocusMode、Android Oboe 集成 |

### [04 — 循环与 DSP](04-循环与DSP/)

| 节 | 标题 | 内容 |
|----|------|------|
| 01 | WaveLoopManager 循环管理 | 循环点/标签/链接/条件跳转、CrossFade 实现 |
| 02 | WaveSegmentQueue 分段播放 | 段队列模型、标签事件传递、与 LoopManager 的协作 |
| 03 | PhaseVocoder 变速不变调 | DSP 滤波链、FFT/IFFT 处理、MathAlgorithms 数学基础 |

### [05 — 实战：添加新音频格式](05-实战-添加新音频格式/)

| 节 | 标题 | 内容 |
|----|------|------|
| 01 | FLAC 解码器实现 | 从零实现 FLACWaveDecoder：接口对接、libFLAC 集成、CMake 配置 |
| 02 | 解码器注册与测试 | Creator 注册、格式探测优先级、单元测试编写 |
| 03 | PCM 格式转换与边界情况 | 多采样率/位深/声道的转换处理、性能优化、常见陷阱 |

## 源码文件速查

| 文件 | 职责 |
|------|------|
| `WaveIntf.h/cpp` | 解码器接口、Creator 注册表、PCM 格式转换 |
| `SoundBufferBaseIntf.h/cpp` | 基础音频缓冲：状态机、淡入淡出、音量控制 |
| `FFWaveDecoder.h/cpp` | FFmpeg 通用解码器（支持任何 FFmpeg 可解码格式） |
| `VorbisWaveDecoder.h/cpp` | Ogg Vorbis + Opus 解码器 |
| `WaveLoopManager.h/cpp` | 循环点管理：标签、链接、条件跳转 |
| `WaveSegmentQueue.h/cpp` | 分段播放队列 |
| `PhaseVocoderDSP.h/cpp` | Phase Vocoder 时间拉伸 DSP |
| `PhaseVocoderFilter.h/cpp` | DSP 滤波链适配器 |
| `MathAlgorithms.h/cpp` | FFT、窗函数等数学工具 |
| `RingBuffer.h` | 环形缓冲模板 |
| `WaveFormatConverter.cpp` | PCM 位深/声道转换 |
| `win32/WaveImpl.h/cpp` | tTJSNI_WaveSoundBuffer 播放器实现 |
| `win32/WaveMixer.h/cpp` | OpenAL 混音器后端 |
| `win32/SoundBufferBaseImpl.h/cpp` | tTJSNI_SoundBuffer 平台实现 |
| `CDDAIntf.h/cpp` / `MIDIIntf.h/cpp` | CD 音频和 MIDI 桩实现 |
