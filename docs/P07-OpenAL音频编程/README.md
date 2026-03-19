# P07 — OpenAL 音频编程

> **前置知识：** [P03-跨平台C++开发](../P03-跨平台C++开发/README.md)、[P06-FFmpeg音视频开发](../P06-FFmpeg音视频开发/README.md) 第1章（音视频基础概念）
> **难度：** ★★★（中级）
> **预计总学时：** 12-16 小时

## 模块概述

OpenAL（Open Audio Library）是一套跨平台的 3D 音频 API，设计风格与 OpenGL 高度一致。KrKr2 项目在**所有非 Android 平台**（Windows/Linux/macOS）上使用 OpenAL Soft 作为音频输出后端，Android 平台则优先使用 Oboe、回退到 SDL2 Audio。

本模块从数字音频的物理基础出发，逐步讲解 OpenAL 的核心对象模型（Device → Context → Source → Buffer）、流式播放机制（buffer queuing）、3D 空间音频，最终实战构建一个完整的音频播放器。

## 学习目标

完成本模块后，你将能够：

1. 理解数字音频的采样、量化、编码原理，能手动计算 PCM 数据的字节大小和播放时长
2. 掌握 OpenAL 的对象层次结构，能独立完成 Device/Context/Source/Buffer 的创建与管理
3. 实现静态缓冲播放和流式队列播放（buffer queuing），理解两者的适用场景
4. 设计并实现环形缓冲区（Ring Buffer），用于音频流的生产者-消费者模型
5. 阅读并理解 KrKr2 项目中 `cpp/core/sound/win32/WaveMixer.cpp` 的 OpenAL 音频渲染器实现
6. 构建一个支持 WAV 加载、PCM 解码、流式播放、音量控制的完整音频播放器

## 章节导航

### [第1章：数字音频基础](01-数字音频基础/)

| 节 | 标题 | 核心内容 |
|----|------|----------|
| [01](01-数字音频基础/01-声音物理与采样原理.md) | 声音物理与采样原理 | 声波三要素、模拟→数字转换、奈奎斯特定理、采样率选择 |
| [02](01-数字音频基础/02-位深与动态范围.md) | 位深与动态范围 | 量化精度、信噪比计算、8/16/24/32-bit 对比、抖动与噪声整形 |
| [03](01-数字音频基础/03-声道布局与码率计算.md) | 声道布局与码率计算 | 单声道/立体声/环绕声布局、码率公式、存储大小估算 |
| [04](01-数字音频基础/04-PCM数据格式详解.md) | PCM 数据格式详解 | 交错存储 vs 平面存储、大端/小端、整数 vs 浮点、WAV 文件头解析 |
| [05](01-数字音频基础/05-音频缓冲区与延迟.md) | 音频缓冲区与延迟 | 缓冲区大小计算、延迟（latency）成因与优化、实时 vs 非实时场景 |

### [第2章：OpenAL 核心概念](02-OpenAL核心概念/)

| 节 | 标题 | 核心内容 |
|----|------|----------|
| [01](02-OpenAL核心概念/01-OpenAL环境搭建与架构总览.md) | OpenAL 环境搭建与架构总览 | vcpkg 安装 openal-soft、CMake 集成、API 设计哲学对比 OpenGL |
| [02](02-OpenAL核心概念/02-Device-Context-Source-Buffer.md) | Device → Context → Source → Buffer | 四层对象模型详解、生命周期管理、多 Context 场景 |
| [03](02-OpenAL核心概念/03-坐标系与空间音频.md) | 坐标系与空间音频 | 3D 音频原理、Listener/Source 位置设置、距离衰减模型、多普勒效应 |

### [第3章：缓冲与流式播放](03-缓冲与流式播放/)

| 节 | 标题 | 核心内容 |
|----|------|----------|
| [01](03-缓冲与流式播放/01-静态缓冲播放.md) | 静态缓冲播放 | alBufferData 一次性加载、适用短音效、内存计算与管理 |
| [02](03-缓冲与流式播放/02-流式队列播放.md) | 流式队列播放 | alSourceQueueBuffers/alSourceUnqueueBuffers 流程、双缓冲/三缓冲策略 |
| [03](03-缓冲与流式播放/03-环形缓冲区设计.md) | 环形缓冲区设计 | 环形缓冲区原理、KrKr2 的 tRisaRingBuffer 实现分析、线程安全设计 |

### [第4章：实战 — 完整音频播放器](04-实战-完整音频播放器/)

| 节 | 标题 | 核心内容 |
|----|------|----------|
| [01](04-实战-完整音频播放器/01-WAV文件加载与PCM解码.md) | WAV 文件加载与 PCM 解码 | WAV 格式解析器实现、PCM 格式转换、错误处理 |
| [02](04-实战-完整音频播放器/02-流式播放引擎实现.md) | 流式播放引擎实现 | 多线程解码+播放架构、Buffer Queue 管理、状态机设计 |
| [03](04-实战-完整音频播放器/03-音量控制与完整播放器.md) | 音量控制与完整播放器 | 音量/声像控制、淡入淡出、播放进度查询、KrKr2 源码对照分析 |

## 术语预览

本模块涉及大量音频编程术语，先在这里建立印象，各章正文会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| PCM | Pulse Code Modulation | 脉冲编码调制——把模拟声波变成数字采样值的最基本方式，几乎所有数字音频的"原始格式" |
| 采样率 | Sample Rate | 每秒对声波取样的次数（如 44100 Hz），决定能还原的最高频率 |
| 奈奎斯特定理 | Nyquist Theorem | 采样率必须 ≥ 信号最高频率的 2 倍，否则会产生走样失真 |
| 位深 | Bit Depth | 每个采样值用多少位表示（如 16-bit），位数越高动态范围越大 |
| OpenAL | Open Audio Library | 跨平台 3D 音频 API，设计风格仿照 OpenGL，是 KrKr2 非 Android 平台的音频输出后端 |
| Device / Context | — | OpenAL 的两层初始化对象：Device 代表声卡硬件，Context 是渲染状态容器 |
| Source / Buffer | — | OpenAL 的播放对象：Buffer 装 PCM 数据，Source 控制播放位置和音量 |
| 缓冲队列 | Buffer Queuing | 把多个小 Buffer 排成队列喂给一个 Source，实现长音频流式播放而不需一次加载整首歌 |
| 环形缓冲区 | Ring Buffer | 首尾相连的固定大小数组，生产者往里写、消费者往外读，用于音频流的线程安全数据传递 |
| 延迟 | Latency | 从发出播放指令到扬声器出声的时间差，缓冲区越大延迟越高 |
| Oboe | — | Google 推出的 Android 高性能音频库，KrKr2 在 Android 平台用它替代 OpenAL |
| WAV | Waveform Audio | Windows 标准无压缩音频格式，内部存储 PCM 数据加一个文件头 |

## 项目源码对照

本模块将频繁引用以下 KrKr2 源文件：

| 文件 | 内容 |
|------|------|
| `cpp/core/sound/win32/WaveMixer.cpp` | OpenAL 音频渲染器（`tTVPAudioRendererAL`）、Buffer Queue 管理（`tTVPSoundBufferAL`）、SDL 混音器 |
| `cpp/core/sound/RingBuffer.h` | 环形缓冲区模板类 `tRisaRingBuffer` |
| `cpp/core/sound/SoundBufferBaseIntf.h` | 音频缓冲区基类、淡入淡出逻辑 |
| `cpp/core/sound/WaveIntf.h` | 波形解码器接口 `tTVPWaveDecoder`、PCM 格式定义 `tTVPWaveFormat` |
| `cpp/core/sound/FFWaveDecoder.cpp` | FFmpeg 解码器实现 |
| `cpp/core/sound/CMakeLists.txt` | 音频模块构建配置、OpenAL/Vorbis/Opus 依赖链接 |

## 环境准备

```bash
# vcpkg 安装 OpenAL Soft
vcpkg install openal-soft

# CMakeLists.txt 配置
find_package(OpenAL CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE OpenAL::OpenAL)
```

## 下一步

完成本模块后，建议继续学习：
- [M05-音频子系统解析](../M05-音频子系统解析/README.md)（深入分析 KrKr2 完整音频管线）
- [P08-编译原理与脚本引擎](../P08-编译原理与脚本引擎/README.md)（TJS2 脚本引擎实现）
