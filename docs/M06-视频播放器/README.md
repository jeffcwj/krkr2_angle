# M06 — 视频播放器 (movie/ffmpeg)

## 模块概述

本模块深入解析 KrKr2 模拟器的完整视频播放栈。KrKr2 的视频播放子系统源自 XBMC/Kodi 开源媒体播放器项目，经过大幅裁剪和适配，集成到吉里吉里引擎的插件架构中。该子系统使用 FFmpeg（开源音视频处理框架）进行音视频解封装（将容器格式拆分为独立的音频流和视频流）与解码（将压缩数据还原为原始像素/采样数据），通过多线程流水线实现高效的实时播放，并与 Cocos2d-x 渲染引擎无缝对接。

### 学习目标

完成本模块后，你将能够：

1. **理解视频播放器的整体架构**：掌握从 IStream 数据源（统一的流式数据读取接口）到屏幕显示的完整数据流
2. **掌握解封装与解码流水线**：理解 CDVDDemuxFFmpeg（基于 FFmpeg 的解封装器）如何将容器格式拆分为音视频流并交给解码器处理
3. **深入音视频同步机制**：理解 CDVDClock 主时钟、PTS 校正（显示时间戳修正）、帧丢弃策略的工作原理
4. **掌握帧缓冲与渲染集成**：理解解码帧如何经过纹理上传最终在 Cocos2d 中显示
5. **阅读 XBMC-Kodi 遗产代码**：理解 CDVDMsg 消息系统（线程间通信机制）、CThread 线程模型（各组件的独立线程封装）等移植代码
6. **具备调试视频播放问题的能力**：能够排查 A/V 不同步、解码失败、性能瓶颈等常见问题

## 术语预览

本模块涉及大量视频播放和 Kodi 遗产代码术语，先在这里建立印象，各章正文会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| IStream | — | Windows COM 接口风格的数据流抽象，KrKr2 用它统一读取本地文件和存档中的视频数据 |
| AVIO | FFmpeg I/O Context | FFmpeg 的自定义 I/O 接口，KrKr2 通过它把 IStream 桥接给 FFmpeg 的解封装器 |
| CDVDDemuxFFmpeg | — | 基于 FFmpeg 的解封装器类（源自 Kodi），负责将视频容器（如 MP4/AVI）拆分为独立的音频流和视频流 |
| 解封装 | Demuxing | 从容器格式中分离出独立的音频、视频数据包（packet）的过程 |
| PTS | Presentation Time Stamp | 显示时间戳——每一帧应该在什么时刻显示到屏幕上，是音视频同步的核心依据 |
| CDVDClock | — | 视频播放器的主时钟类（源自 Kodi），负责维护全局播放时间并作为 A/V 同步的基准 |
| CDVDMsg | — | Kodi 风格的消息系统类，各线程之间通过消息队列传递控制指令和数据包 |
| CThread | — | Kodi 风格的线程封装类，为每个解码器/播放器组件提供独立线程和生命周期管理 |
| YUV | — | 视频的原生颜色空间（Y=亮度，U/V=色度），解码器输出 YUV 格式的帧数据 |
| sws_scale | — | FFmpeg 的像素格式转换函数，KrKr2 用它将解码出的 YUV 帧转换为 RGBA 以供渲染 |
| Overlay 模式 | — | 视频播放的高性能模式——YUV 数据直接通过专用着色器渲染，跳过 YUV→RGBA 转换 |
| Layer 模式 | — | 视频播放的兼容模式——先用 sws_scale 将 YUV 转为 RGBA，再作为普通图层纹理绘制 |
| CRenderManager | — | 帧缓冲管理器，维护 free→queued→displayed→discard 四状态队列，解码线程和渲染线程通过它交换帧 |
| A/V 同步 | Audio/Video Synchronization | 确保视频画面和音频声音在时间上对齐的机制——通常以音频时钟为基准调整视频帧的显示时机 |

### 源码位置

```
krkr2/cpp/core/movie/ffmpeg/
├── krffmpeg.cpp             # 插件入口点（GetVideoOverlayObject/GetVideoLayerObject）
├── KRMoviePlayer.h/cpp      # TVPMoviePlayer 主类（Overlay 模式）
├── KRMovieLayer.h/cpp       # TVPMoviePlayer 主类（Layer 模式）
├── VideoPlayer.h/cpp        # BasePlayer 核心播放循环
├── DemuxFFmpeg.h/cpp        # FFmpeg 解封装器
├── VideoPlayerVideo.cpp     # 视频解码线程
├── VideoPlayerAudio.cpp     # 音频解码线程
├── Clock.h/cpp              # CDVDClock 主时钟
├── VideoRenderer.h/cpp      # CRenderManager 渲染管理器
├── Message.h                # CDVDMsg 消息定义
├── Thread.h                 # CThread 线程封装
├── InputStream.cpp          # IStream COM → FFmpeg AVIO 适配器
└── ... (共 35 个 .cpp + 50 个 .h 文件)
```

### 整体架构图

```
┌─────────────────────────────────────────────────────────┐
│                    KrKr2 脚本层                          │
│  VideoOverlay.open("movie.mpg")                         │
└────────────────────────┬────────────────────────────────┘
                         │ iTVPVideoOverlay 接口
┌────────────────────────▼────────────────────────────────┐
│              KRMoviePlayer / KRMovieLayer                │
│         (TVPMoviePlayer + CBaseRenderer)                │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                  BasePlayer (CThread)                    │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │InputStream│→│CDVDDemuxFFmpeg│→│  CDVDMessageQueue │  │
│  │(IStream)  │  │(AVFormat)    │  │  (packet分发)    │  │
│  └──────────┘  └──────────────┘  └────┬────────┬────┘  │
│                                       │        │        │
│  ┌────────────────────┐  ┌───────────▼──┐ ┌──▼──────┐ │
│  │   CDVDClock        │  │CVideoPlayer  │ │CVideoP. │ │
│  │   (主时钟/同步)     │  │Video(解码)   │ │Audio    │ │
│  └────────────────────┘  └──────┬───────┘ └─────────┘ │
│                                  │                      │
│  ┌──────────────────────────────▼──────────────────┐   │
│  │          CRenderManager (帧队列管理)              │   │
│  │     free → queued → displayed → discard          │   │
│  └──────────────────────────────┬──────────────────┘   │
└─────────────────────────────────┼──────────────────────┘
                                  │
┌─────────────────────────────────▼──────────────────────┐
│              显示层（二选一）                             │
│  Overlay: Cocos2d Node + TVPYUVSprite (YUV 直通)       │
│  Layer:   tTVPBaseTexture (YUV→RGBA sws_scale)         │
└────────────────────────────────────────────────────────┘
```

## 章节目录

### 第 1 章：movie/ffmpeg 架构

| 小节 | 内容 | 文件 |
|------|------|------|
| 1.1 | Kodi/XBMC 遗产代码概述 | [01-Kodi遗产代码概述.md](01-movie-ffmpeg架构/01-Kodi遗产代码概述.md) |
| 1.2 | CDVDPlayer 架构解析 | [02-CDVDPlayer架构.md](01-movie-ffmpeg架构/02-CDVDPlayer架构.md) |
| 1.3 | 线程模型详解 | [03-线程模型.md](01-movie-ffmpeg架构/03-线程模型.md) |

### 第 2 章：解封装与解码流水线

| 小节 | 内容 | 文件 |
|------|------|------|
| 2.1 | CDVDDemuxFFmpeg 解封装器 | [01-CDVDDemuxFFmpeg解封装器.md](02-解封装与解码流水线/01-CDVDDemuxFFmpeg解封装器.md) |
| 2.2 | CDVDStreamInfo 与流管理 | [02-CDVDStreamInfo与流管理.md](02-解封装与解码流水线/02-CDVDStreamInfo与流管理.md) |
| 2.3 | 音视频解码器 | [03-音视频解码器.md](02-解封装与解码流水线/03-音视频解码器.md) |

### 第 3 章：音视频同步机制

| 小节 | 内容 | 文件 |
|------|------|------|
| 3.1 | CDVDClock 主时钟 | [01-CDVDClock主时钟.md](03-音视频同步机制/01-CDVDClock主时钟.md) |
| 3.2 | PTS 校正与时间戳转换 | [02-PTS校正与时间戳转换.md](03-音视频同步机制/02-PTS校正与时间戳转换.md) |
| 3.3 | 帧丢弃策略 | [03-帧丢弃策略.md](03-音视频同步机制/03-帧丢弃策略.md) |

### 第 4 章：帧缓冲与渲染集成

| 小节 | 内容 | 文件 |
|------|------|------|
| 4.1 | 解码帧到纹理上传 | [01-解码帧到纹理上传.md](04-帧缓冲与渲染集成/01-解码帧到纹理上传.md) |
| 4.2 | CRenderManager 渲染管线 | [02-CRenderManager渲染管线.md](04-帧缓冲与渲染集成/02-CRenderManager渲染管线.md) |
| 4.3 | Cocos2d 显示集成 | [03-Cocos2d显示集成.md](04-帧缓冲与渲染集成/03-Cocos2d显示集成.md) |

### 第 5 章：XBMC-Kodi 遗产代码导读

| 小节 | 内容 | 文件 |
|------|------|------|
| 5.1 | CDVDMsg 消息系统 | [01-CDVDMsg消息系统.md](05-XBMC-Kodi遗产代码导读/01-CDVDMsg消息系统.md) |
| 5.2 | CThread 线程模型 | [02-CThread线程模型.md](05-XBMC-Kodi遗产代码导读/02-CThread线程模型.md) |
| 5.3 | 与原版 Kodi 的差异 | [03-与原版Kodi的差异.md](05-XBMC-Kodi遗产代码导读/03-与原版Kodi的差异.md) |

### 第 6 章：实战——调试视频播放问题

| 小节 | 内容 | 文件 |
|------|------|------|
| 6.1 | A/V 不同步排查 | [01-AV不同步排查.md](06-实战-调试视频播放问题/01-AV不同步排查.md) |
| 6.2 | 解码失败分析 | [02-解码失败分析.md](06-实战-调试视频播放问题/02-解码失败分析.md) |
| 6.3 | 性能瓶颈定位 | [03-性能瓶颈定位.md](06-实战-调试视频播放问题/03-性能瓶颈定位.md) |

## 前置知识

- **M04-渲染子系统**：理解 Cocos2d-x 渲染管线和纹理系统
- **M05-音频子系统**：理解音频输出设备和缓冲区管理
- **P06-FFmpeg 音视频开发**：FFmpeg 基础 API（AVFormatContext、AVCodecContext 等）
- **P03-跨平台 C++ 开发**：多线程编程基础（mutex、condition variable、atomic）

## 关键概念速查

| 概念 | 说明 |
|------|------|
| `DVD_NOPTS_VALUE` | `0xFFF0000000000000`，表示无效时间戳的哨兵值 |
| `DVD_TIME_BASE` | `1000000`（微秒），与 FFmpeg 的 `AV_TIME_BASE` 相同 |
| `KRMovie` 命名空间 | 通过 `NS_KRMOVIE_BEGIN`/`NS_KRMOVIE_END` 宏定义 |
| Overlay 模式 | 使用 Cocos2d Node + TVPYUVSprite，YUV 数据直通显示 |
| Layer 模式 | 使用 tTVPBaseTexture，通过 sws_scale 将 YUV 转为 RGBA |
| `MAX_BUFFER_COUNT` | 帧缓冲区大小 = 4（KRMoviePlayer）/ 6（CRenderManager） |

## 构建目标

视频播放器编译为静态库 `core_movie_module`，链接依赖：

```cmake
target_link_libraries(core_movie_module
    FFmpeg::avcodec FFmpeg::avformat FFmpeg::avutil
    FFmpeg::swscale FFmpeg::swresample
    cocos2d tjs2 base sound utils environ visual
)
```
