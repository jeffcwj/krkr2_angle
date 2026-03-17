# 本节目标
# 实战：KrKr2 媒体播放器架构分析

> **所属模块：** P06-FFmpeg音视频开发
> **前置知识：** [03-解封装](03-解封装从容器到数据包.md)、[04-解码](04-解码从数据包到原始帧.md)、[05-音视频同步](05-音视频同步.md)、[06-实战播放器](06-实战FFmpeg-SDL2视频播放器.md)
> **预计阅读时间：** 60 分钟

## 本节目标

读完本节后，你将能够：
1. 说出 KrKr2 媒体播放器的整体架构（BasePlayer → Demux → Decode → Render 四层管线）
2. 解释 Kodi/XBMC 遗产代码在 KrKr2 中的保留与裁剪策略
3. 追踪一帧视频从文件读取到屏幕显示的完整数据流
4. 理解 CDVDMsg 消息系统如何驱动线程间通信
5. 分析 CRenderManager 的帧缓冲队列与 VSync 对齐机制
6. 定位并修改播放器中的关键参数（缓冲区大小、丢帧阈值等）

## 架构总览：从 Kodi 到 KrKr2

KrKr2 的视频播放器位于 `cpp/core/movie/ffmpeg/` 目录，源自 Kodi（原 XBMC）媒体中心的播放器内核。
这不是一个简单的 FFmpeg 封装——它是一套完整的多线程媒体播放管线，包含解封装、解码、音视频同步、
帧缓冲管理和渲染输出。

### 为什么是 Kodi 架构？

KrKr2 需要播放视觉小说中的过场动画（OP/ED 视频、背景动画等），要求：
- 支持多种视频格式（MPEG、H.264、VP9 等）
- 音视频精确同步
- 可在游戏引擎内部渲染（输出到 Cocos2d-x 纹理）
- 跨平台（Windows/Linux/macOS/Android）

Kodi 的播放器内核恰好满足这些需求，且经过了十多年的生产环境验证。
KrKr2 团队将其核心播放逻辑移植过来，裁剪掉了字幕、网络流、蓝光等不需要的功能。

### 目录结构

```
cpp/core/movie/ffmpeg/
├── KRMoviePlayer.cpp/h    ← KrKr2 入口：TVPMoviePlayer（iTVPVideoOverlay 接口实现）
├── VideoPlayer.cpp/h      ← BasePlayer：主控线程，驱动整条管线
├── DemuxFFmpeg.cpp/h      ← CDVDDemuxFFmpeg：解封装层（AVFormatContext 封装）
├── VideoCodecFFmpeg.cpp/h ← CDVDVideoCodecFFmpeg：视频解码（含硬件加速接口）
├── AudioCodecFFmpeg.cpp/h ← CDVDAudioCodecFFmpeg：音频解码
├── VideoPlayerVideo.cpp/h ← CVideoPlayerVideo：视频解码线程
├── VideoPlayerAudio.cpp/h ← CVideoPlayerAudio：音频解码线程
├── VideoRenderer.cpp/h    ← CRenderManager：帧缓冲管理 + 渲染调度
├── Clock.cpp/h            ← CDVDClock：主时钟（微秒精度）
├── MessageQueue.cpp/h     ← CDVDMessageQueue：线程间消息队列
├── KRMovieLayer.cpp/h     ← VideoPresentLayer：Cocos2d-x 图层渲染
├── AE.h                   ← IAE 音频引擎接口
└── OMXClock.h             ← 空桩（Kodi 遗留，勿删）
```

### 层次关系图

```
┌─────────────────────────────────────────────────────┐
│                   KrKr2 引擎层                        │
│  TVPMoviePlayer (KRMoviePlayer.cpp)                  │
│  ├── 实现 iTVPVideoOverlay 接口                      │
│  ├── 管理 YUV 环形缓冲区 (MAX_BUFFER_COUNT=4)        │
│  └── 通过 cocos2d Scheduler 触发渲染                  │
├─────────────────────────────────────────────────────┤
│                   播放控制层                           │
│  BasePlayer (VideoPlayer.cpp)                        │
│  ├── 主循环线程 Process()                             │
│  ├── 缓存状态机 (FLUSH→FULL→PLAY→DONE)               │
│  └── 协调所有子组件                                   │
├─────────────────────────────────────────────────────┤
│              数据处理层（多线程）                       │
│  ┌──────────────┐  ┌──────────────┐                  │
│  │CDVDDemuxFFmpeg│  │  CDVDClock   │                  │
│  │  解封装线程   │  │  主时钟管理   │                  │
│  └──────┬───────┘  └──────────────┘                  │
│         │ AVPacket                                    │
│  ┌──────┴───────┐  ┌──────────────┐                  │
│  │VideoPlayerVid │  │VideoPlayerAud│                  │
│  │  视频解码线程 │  │  音频解码线程 │                  │
│  └──────┬───────┘  └──────────────┘                  │
│         │ AVFrame (YUV420P)                           │
├─────────┴───────────────────────────────────────────┤
│                   渲染管理层                           │
│  CRenderManager (VideoRenderer.cpp)                  │
│  ├── 6 帧缓冲队列 (free→queued→present→discard)      │
│  ├── VSync 对齐 + 延迟补偿                            │
│  └── 丢帧判定                                         │
└─────────────────────────────────────────────────────┘
```

## Kodi 遗产：保留了什么、裁剪了什么

理解哪些是 Kodi 原始代码、哪些是 KrKr2 定制代码，对阅读源码至关重要。

### 保留的 Kodi 特征

| 特征 | 说明 | 源码位置 |
|------|------|----------|
| `DVD_` 前缀命名 | DVD 时代的命名约定，如 `DVD_NOPTS_VALUE`、`DVD_TIME_BASE` | `Clock.h` |
| `CDVDMsg` 消息系统 | 线程间通信的消息类型层次 | `MessageQueue.cpp` |
| `CThread` 线程基类 | 所有工作线程的基类 | BasePlayer/VideoPlayerVideo/Audio |
| 缓存状态机 | `CACHESTATE_FLUSH/FULL/INIT/PLAY/DONE` | `VideoPlayer.cpp` |
| `OMXClock` 空桩 | 树莓派硬件时钟接口，保留为空实现 | `OMXClock.h` |
| 老版 FFmpeg API | `avcodec_decode_video2` / `avcodec_decode_audio4` | `VideoCodecFFmpeg.cpp` |

### 裁剪的 Kodi 功能

| 功能 | 原因 |
|------|------|
| 字幕渲染 (SubtitleRenderer) | 视觉小说不需要外挂字幕 |
| 网络流 (RTSP/HLS) | 只需播放本地文件 |
| 蓝光/DVD 导航 | 不适用 |
| 多音轨/多字幕切换 UI | 简化为单音轨 |
| 复杂的播放列表管理 | 单文件播放即可 |

### KrKr2 新增的定制代码

```
KRMoviePlayer.cpp  ← 全新：桥接 Kodi 播放器与 KrKr2 引擎
KRMovieLayer.cpp   ← 全新：Cocos2d-x 图层渲染
```

**关键定制点：** `TVPMoviePlayer` 类同时继承了 `iTVPVideoOverlay`（KrKr2 视频叠加接口）
和 `CBaseRenderer`（Kodi 渲染器接口），充当两个系统之间的桥梁。

## CDVDMsg 消息系统：线程间通信的核心

整个播放器的线程协调依赖 `CDVDMsg` 消息系统。理解它是读懂源码的前提。

### 消息类型层次

```cpp
// MessageQueue.cpp 中定义的消息类型（摘自实际源码）

// 通用消息 —— 所有线程都能处理
GENERAL_SYNCHRONIZE    // 线程同步屏障
GENERAL_RESYNC         // 重新同步（seek 后）
GENERAL_FLUSH          // 清空缓冲区
GENERAL_STREAMCHANGE   // 流切换
GENERAL_EOF            // 文件结束

// 播放器控制消息
PLAYER_SETSPEED        // 设置播放速度
PLAYER_STARTED         // 播放已开始
PLAYER_AVCHANGE        // 音视频参数变化

// 解封装器消息
DEMUXER_PACKET         // 携带 AVPacket 数据
```

### 消息队列实现

`CDVDMessageQueue` 是一个优先级消息队列，支持：

```cpp
// 源码：MessageQueue.cpp 第 58-118 行
// CDVDMessageQueue::Put() 实现

void CDVDMessageQueue::Put(CDVDMsg* pMsg, int priority) {
    // 1. 加锁
    std::unique_lock<std::mutex> lock(m_section);

    // 2. 优先级消息插入队首
    if (priority > 0) {
        m_messages.push_front({pMsg, priority});
    } else {
        m_messages.push_back({pMsg, priority});
    }

    // 3. 唤醒等待的消费者线程
    m_event.notify_one();
}

// CDVDMessageQueue::Get() —— 带超时的阻塞获取
// 源码：MessageQueue.cpp 第 120-180 行
MsgQueueReturnCode CDVDMessageQueue::Get(
    CDVDMsg** pMsg, unsigned int iTimeoutInMilliSeconds) {
    
    std::unique_lock<std::mutex> lock(m_section);
    
    // 优先处理高优先级消息
    for (auto it = m_messages.begin(); it != m_messages.end(); ++it) {
        if (it->priority > 0) {
            *pMsg = it->pMsg;
            m_messages.erase(it);
            return MSGQ_OK;
        }
    }
    
    // 无高优先级消息时，取队首
    if (!m_messages.empty()) {
        *pMsg = m_messages.front().pMsg;
        m_messages.pop_front();
        return MSGQ_OK;
    }
    
    // 队列为空，等待超时
    if (iTimeoutInMilliSeconds > 0) {
        m_event.wait_for(lock, 
            std::chrono::milliseconds(iTimeoutInMilliSeconds));
    }
    
    return MSGQ_TIMEOUT;
}
```

**设计要点：**
- 优先级消息（priority > 0）总是先于普通消息被处理
- `GENERAL_FLUSH` 和 `GENERAL_RESYNC` 通常以高优先级发送，确保 seek 操作立即生效
- `DEMUXER_PACKET` 以普通优先级发送，因为数据包按顺序处理即可

### 消息流转图

```
BasePlayer::Process()
    │
    ├── ReadPacket() 从 DemuxFFmpeg 读取 AVPacket
    │       │
    │       ├── 视频包 → CDVDMsgDemuxerPacket → VideoPlayerVideo 消息队列
    │       │                                        │
    │       │                                        ├── Get() 取出
    │       │                                        ├── Decode()
    │       │                                        └── OutputPicture() → RenderManager
    │       │
    │       └── 音频包 → CDVDMsgDemuxerPacket → VideoPlayerAudio 消息队列
    │                                                │
    │                                                ├── Get() 取出
    │                                                ├── Decode()
    │                                                └── AddSamples() → AudioEngine
    │
    ├── HandleMessages() 处理控制消息
    │       ├── GENERAL_FLUSH → 转发给所有子线程
    │       └── PLAYER_SETSPEED → 更新时钟速率
    │
    └── HandlePlaySpeed() 管理缓存与同步状态
```

