# M05 — 音频子系统

> **前置依赖：** M01-项目导览与环境搭建、P06-FFmpeg 音视频开发、P07-OpenAL 音频编程
> **目标：** 掌握 KrKr2 的音频架构，能添加新格式支持和修改 DSP 处理

## 模块概述

KrKr2 的音频子系统位于 `cpp/core/sound/`，CMake 目标名为 `core_sound_module`。它实现了完整的音频播放栈：从文件解码（Vorbis/Opus/FFmpeg 等多种音频格式解码器）、PCM（脉冲编码调制，音频的原始数字采样数据）格式转换、循环管理、DSP 滤波（PhaseVocoder 变速不变调），到最终通过 OpenAL（跨平台音频输出库）输出到硬件。Android 平台额外使用 Oboe（Google 的高性能音频库）作为底层音频后端。

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

## 术语预览

本模块涉及大量音频子系统的专业术语，先在这里建立印象，各章正文会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| PCM | Pulse Code Modulation | 脉冲编码调制——音频的最原始数字格式，存储每个时间点的声波采样值 |
| Vorbis | Ogg Vorbis | 开源有损音频编码格式，KrKr2 的 BGM 和音效常用 `.ogg` 文件 |
| Opus | — | 新一代开源音频编码格式，低延迟高音质，KrKr2 也内置了解码支持 |
| FLAC | Free Lossless Audio Codec | 开源无损音频格式，实战章节将从零实现 FLAC 解码器 |
| DSP | Digital Signal Processing | 数字信号处理——对 PCM 音频数据进行变速、变调、滤波等数学处理 |
| PhaseVocoder | 相位声码器 | 一种 DSP 算法——通过 FFT 频域处理实现"变速不变调"（加快/减慢播放但不改变音高） |
| FFT / IFFT | Fast Fourier Transform / Inverse | 快速傅里叶变换——把时域音频信号转换到频域（和反向转换），是 PhaseVocoder 的数学基础 |
| OpenAL | Open Audio Library | 跨平台 3D 音频 API，KrKr2 在 Windows/Linux/macOS 上用它输出音频 |
| Oboe | — | Google 推出的 Android 高性能音频库，KrKr2 在 Android 平台用它替代 OpenAL |
| Creator 注册 | WaveDecoderCreator Registration | KrKr2 的解码器插件机制——每种音频格式注册一个"创建器"，播放时按格式自动选择对应解码器 |
| CrossFade | 交叉淡入淡出 | 两段音频重叠播放时，前一段淡出同时后一段淡入，实现平滑过渡 |
| WaveSoundBuffer | — | KrKr2 暴露给 TJS2 脚本的音频播放器类，游戏脚本用它控制播放、暂停、音量等 |

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
