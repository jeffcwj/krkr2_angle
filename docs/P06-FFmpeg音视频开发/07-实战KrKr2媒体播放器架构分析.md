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

## BasePlayer 主循环：Process() 详解

`BasePlayer::Process()` 是整个播放器的心脏，运行在独立线程中，驱动所有子组件。

源码位置：`cpp/core/movie/ffmpeg/VideoPlayer.cpp` 第 538-741 行

### 主循环伪代码

```cpp
// VideoPlayer.cpp BasePlayer::Process() 简化逻辑
void BasePlayer::Process() {
    // === 阶段 1：初始化 ===
    OpenInputStream(m_filename);      // 打开文件
    OpenDemuxStream();                // 初始化解封装器
    OpenDefaultStreams();             // 选择默认音视频流
    
    // === 阶段 2：主循环 ===
    while (!m_bAbortRequest) {
        // 步骤 A：处理控制消息（seek、速度变更等）
        HandleMessages();
        
        // 步骤 B：管理缓存状态和音视频同步
        HandlePlaySpeed();
        
        // 步骤 C：检查连续性（PTS 跳变检测）
        CheckContinuity();
        
        // 步骤 D：读取并分发数据包
        DemuxPacket* pkt = ReadPacket();
        if (pkt) {
            ProcessPacket(pkt);  // 按流类型分发
        }
        
        // 步骤 E：更新播放状态
        UpdatePlayState();
    }
    
    // === 阶段 3：清理 ===
    DestroyPlayers();
    CloseInputStream();
}
```

### 缓存状态机

BasePlayer 维护一个缓存状态机，控制数据填充和播放启动的时机：

```
    ┌──────────────────────────────────────────────┐
    │                                              │
    ▼                                              │
CACHESTATE_FLUSH ──→ CACHESTATE_FULL ──→ CACHESTATE_PLAY ──→ CACHESTATE_DONE
    │                     │                   │
    │                     │                   │
    │                     ▼                   │
    │              CACHESTATE_INIT            │
    │                     │                   │
    │                     └───────────────────┘
    │                                              
    └──── seek 时重置回 FLUSH ─────────────────────
```

各状态含义：

| 状态 | 含义 | 行为 |
|------|------|------|
| `CACHESTATE_FLUSH` | 缓冲区刚被清空（seek 后） | 开始填充数据 |
| `CACHESTATE_INIT` | 首次启动，等待初始缓冲 | 填充到阈值后转入 FULL |
| `CACHESTATE_FULL` | 缓冲区已满 | 准备开始播放 |
| `CACHESTATE_PLAY` | 正在播放 | 正常读取和解码 |
| `CACHESTATE_DONE` | 文件读取完毕 | 等待缓冲区消耗完 |

```cpp
// 源码：VideoPlayer.cpp HandlePlaySpeed() 中的缓存状态转换
// 第 1920-2094 行（简化）

void BasePlayer::HandlePlaySpeed() {
    if (m_caching == CACHESTATE_FLUSH) {
        // seek 后重置：清空所有子线程缓冲区
        FlushBuffers(false);
        m_caching = CACHESTATE_FULL;
    }
    
    if (m_caching == CACHESTATE_FULL) {
        // 检查缓冲是否足够
        if (GetCacheFillLevel() > 80) {  // 填充度 > 80%
            m_caching = CACHESTATE_PLAY;
            // 启动音视频同步
            SynchronizePlayers();
        }
    }
    
    if (m_caching == CACHESTATE_PLAY) {
        // 正常播放状态
        // 检查是否需要重新缓冲（网络流场景，本地文件通常不会）
    }
}
```

### 音视频同步启动过程

音视频同步的启动是最精密的部分之一。当两个解码线程都准备好后：

```cpp
// 源码：VideoPlayer.cpp 第 1992-2094 行（简化）
// 同步启动流程

void BasePlayer::SynchronizePlayers() {
    // 1. 两个流都必须到达 SYNC_WAITSYNC 状态
    if (m_CurrentVideo.syncState != SYNC_WAITSYNC) return;
    if (m_CurrentAudio.syncState != SYNC_WAITSYNC) return;
    
    // 2. 设置时钟不连续标记
    m_clock.Discontinuity(pts);
    
    // 3. 向两个线程发送 GENERAL_RESYNC（高优先级）
    m_VideoPlayerVideo->SendMessage(
        new CDVDMsgGeneralResync(pts), 1);  // priority=1
    m_VideoPlayerAudio->SendMessage(
        new CDVDMsgGeneralResync(pts), 1);
    
    // 4. 标记为已同步
    m_CurrentVideo.syncState = SYNC_INSYNC;
    m_CurrentAudio.syncState = SYNC_INSYNC;
}
```

**为什么需要这个同步过程？** 因为视频和音频解码速度不同，第一个 I 帧解码完毕的时间点不一致。
如果不等待两者都准备好就开始播放，会出现"画面先出来但没声音"或"声音先出来但黑屏"的问题。

## 视频解码线程：CVideoPlayerVideo

### 线程主循环

`CVideoPlayerVideo` 运行在独立线程中，从消息队列获取数据包，解码后输出到渲染管理器。

源码位置：`cpp/core/movie/ffmpeg/VideoPlayerVideo.cpp` 第 209-473 行

```cpp
// VideoPlayerVideo.cpp CVideoPlayerVideo::Process() 简化
void CVideoPlayerVideo::Process() {
    while (!m_bStop) {
        CDVDMsg* pMsg;
        // 从消息队列阻塞获取（超时 1000ms）
        MsgQueueReturnCode ret = m_messageQueue.Get(&pMsg, 1000);
        
        if (ret == MSGQ_TIMEOUT) continue;
        
        // 根据消息类型处理
        switch (pMsg->GetType()) {
        
        case CDVDMsg::GENERAL_SYNCHRONIZE:
            // 同步屏障：等待其他线程到达同一点
            break;
            
        case CDVDMsg::GENERAL_RESYNC:
            // seek 后重新同步：重置内部 PTS
            m_iCurrentPts = DVD_NOPTS_VALUE;
            break;
            
        case CDVDMsg::GENERAL_FLUSH:
            // 清空解码器缓冲
            m_pVideoCodec->Reset();
            break;
            
        case CDVDMsg::PLAYER_SETSPEED:
            // 更新播放速度
            m_speed = static_cast<CDVDMsgInt*>(pMsg)->m_value;
            break;
            
        case CDVDMsg::DEMUXER_PACKET: {
            // 核心：解码数据包
            DemuxPacket* pPacket = 
                static_cast<CDVDMsgDemuxerPacket*>(pMsg)->GetPacket();
            
            // 调用 FFmpeg 解码
            int result = m_pVideoCodec->Decode(
                pPacket->pData, pPacket->iSize,
                pPacket->dts, pPacket->pts);
            
            // 如果解码出帧，输出到渲染器
            if (result & VC_PICTURE) {
                ProcessDecoderOutput();
            }
            break;
        }
        }
        
        pMsg->Release();  // 释放消息（引用计数）
    }
}
```

### OutputPicture：帧输出与时序控制

`OutputPicture()` 是视频解码线程中最复杂的函数，负责将解码后的帧按正确时序输出。

源码位置：`cpp/core/movie/ffmpeg/VideoPlayerVideo.cpp` 第 590-780 行

```cpp
// VideoPlayerVideo.cpp OutputPicture() 简化
int CVideoPlayerVideo::OutputPicture(DVDVideoPicture* pPicture) {
    // === 1. PTS 计算 ===
    double pts = pPicture->pts;
    if (pts == DVD_NOPTS_VALUE) {
        pts = m_iCurrentPts + m_fFrameTime;  // 无 PTS 时用帧率推算
    }
    m_iCurrentPts = pts;
    
    // === 2. 计算与主时钟的偏差 ===
    double clock = m_pClock->GetClock();
    double frametime = (double)DVD_TIME_BASE / m_fFrameRate;
    double error = pts - clock;
    
    // === 3. 丢帧判定 ===
    // CalcDropRequirement() 分析当前延迟情况
    int dropReq = CalcDropRequirement(clock);
    if (dropReq > 0 && m_speed == DVD_PLAYSPEED_NORMAL) {
        // 视频落后太多，丢弃此帧
        m_droppingStats.m_dropcount++;
        return EOS_DROPPED;
    }
    
    // === 4. 等待渲染缓冲区可用 ===
    // WaitForBuffer() 在 CRenderManager 的帧队列满时阻塞
    if (!m_renderManager.WaitForBuffer(m_bAbortRequest)) {
        return EOS_ABORT;
    }
    
    // === 5. 提交帧到渲染管理器 ===
    m_renderManager.AddVideoPicture(pPicture, m_bAbortRequest);
    
    // === 6. 触发渲染 ===
    m_renderManager.FlipPage(m_bAbortRequest, pts, 
                             -1, -1, mDisplayField);
    
    return EOS_OK;
}
```

### 丢帧策略

```cpp
// VideoPlayerVideo.cpp CalcDropRequirement() 简化
// 源码：第 480-588 行

int CVideoPlayerVideo::CalcDropRequirement(double clock) {
    double iDecoderPts = m_iCurrentPts;
    double iClock = clock;
    
    // 计算视频比音频落后了多少
    double iLateness = iClock - iDecoderPts;
    
    // 落后超过 1 帧时间 → 建议丢帧
    if (iLateness > m_fFrameTime) {
        return 1;  // 丢 1 帧
    }
    
    // 落后超过 4 帧时间 → 紧急丢帧
    if (iLateness > m_fFrameTime * 4) {
        return 2;  // 紧急丢帧模式
    }
    
    return 0;  // 不需要丢帧
}
```

**丢帧统计** 通过 `CDroppingStats` 类跟踪：

```cpp
// 源码：VideoPlayerVideo.cpp CDroppingStats 结构
struct CDroppingStats {
    int m_dropcount;       // 总丢帧数
    double m_lastPts;      // 上次丢帧的 PTS
    int m_lateframes;      // 连续迟到帧计数
    double m_latecount;    // 总迟到计数
};
```

## CRenderManager：帧缓冲管理

渲染管理器是播放器中最精密的组件之一，负责帧缓冲队列、VSync 对齐和帧调度。

源码位置：`cpp/core/movie/ffmpeg/VideoRenderer.cpp`（1274 行）

### 帧缓冲队列设计

CRenderManager 使用 4 个 deque 管理 6 个帧缓冲槽位：

```
NUM_BUFFERS = 6

帧的生命周期：

  free → queued → presentsource → discard
  (空闲)  (等待显示)  (正在显示)    (显示完毕)
    ↑                                  │
    └──────────── 回收 ────────────────┘
```

```cpp
// 源码：VideoRenderer.cpp 帧队列成员
// 第 30-50 行附近

class CRenderManager {
    std::deque<int> m_free;          // 空闲缓冲区索引
    std::deque<int> m_queued;        // 已入队等待显示
    std::deque<int> m_discard;       // 已显示待回收
    int m_presentsource = -1;        // 当前正在显示的缓冲区
    
    // 帧数据存储
    DVDVideoPicture m_pBuffers[NUM_BUFFERS];
    
    // 时序信息
    double m_presentpts;             // 当前显示帧的 PTS
    double m_presentstep;            // 帧间隔
    double m_clock_framefinish;      // 帧结束时钟
};
```

### PrepareNextRender：帧调度核心

`PrepareNextRender()` 是渲染调度的核心算法，决定何时从队列取出下一帧显示。

源码位置：`cpp/core/movie/ffmpeg/VideoRenderer.cpp` 第 1141-1222 行

```cpp
// VideoRenderer.cpp PrepareNextRender() 简化
bool CRenderManager::PrepareNextRender() {
    // === 1. 检查是否有帧等待显示 ===
    if (m_queued.empty()) return false;
    
    // === 2. 获取当前时钟 ===
    double clock = m_pClock->GetClock();
    double frametime = m_presentstep;  // 帧间隔
    
    // === 3. 计算显示延迟补偿 ===
    // 考虑 VSync 对齐：帧不是立即显示的，
    // 需要等到下一个 VSync 信号
    double display_latency = m_displayLatency;  // 通常 1-2 个 VSync 周期
    
    // === 4. 判断是否该显示下一帧 ===
    double nextframepts = m_pBuffers[m_queued.front()].pts;
    double diff = nextframepts - clock;
    
    // 考虑延迟后的实际显示时间
    double target = diff - display_latency;
    
    if (target <= 0) {
        // 该帧已经到了显示时间（或已迟到）
        
        // === 5. 迟到帧检测 ===
        if (target < -frametime) {
            m_lateframes++;
            // 如果连续多帧迟到，说明解码跟不上
        } else {
            m_lateframes = 0;
        }
        
        // === 6. 从队列取出帧 ===
        if (m_presentsource >= 0) {
            m_discard.push_back(m_presentsource);  // 旧帧入丢弃队列
        }
        m_presentsource = m_queued.front();
        m_queued.pop_front();
        m_presentpts = nextframepts;
        
        return true;
    }
    
    return false;  // 还没到显示时间
}
```

### 渲染流水线

```cpp
// VideoRenderer.cpp Render() 调用链
// FrameMove() → PrepareNextRender() → Render() → PresentSingle()

void CRenderManager::FrameMove() {
    // 每帧调用（由 Cocos2d-x 调度器驱动）
    
    // 1. 回收已丢弃的缓冲区
    while (!m_discard.empty()) {
        m_free.push_back(m_discard.front());
        m_discard.pop_front();
    }
    
    // 2. 准备下一帧
    PrepareNextRender();
    
    // 3. 执行渲染
    if (m_presentsource >= 0) {
        Render();
    }
}

void CRenderManager::Render() {
    // 根据模式选择渲染方式
    PresentSingle();  // 逐帧渲染（最常用）
    // 或 PresentFields() — 隔行扫描
    // 或 PresentBlend() — 帧混合
}
```

## TVPMoviePlayer：KrKr2 的桥梁层

`TVPMoviePlayer` 是 KrKr2 与 Kodi 播放器之间的桥梁，也是你修改播放器行为时最常接触的文件。

源码位置：`cpp/core/movie/ffmpeg/KRMoviePlayer.cpp`（382 行）

### 类继承关系

```cpp
// KRMoviePlayer.h 中的定义
class TVPMoviePlayer 
    : public iTVPVideoOverlay    // KrKr2 引擎接口
    , public CBaseRenderer       // Kodi 渲染器接口
{
    // KrKr2 侧
    iTVPVideoOverlay* m_pOverlay;   // 上层叠加层
    
    // Kodi 侧
    BasePlayer* m_pPlayer;           // 播放器实例
    CRenderManager m_renderManager;  // 渲染管理器
    
    // YUV 环形缓冲
    struct YUVBuffer {
        uint8_t* data[3];   // Y, U, V 三个平面
        int linesize[3];    // 每行字节数
        double pts;          // 显示时间戳
    };
    YUVBuffer m_buffers[MAX_BUFFER_COUNT];  // MAX_BUFFER_COUNT=4
    int m_writeIdx;
    int m_readIdx;
};
```

### AddVideoPicture：从解码器到环形缓冲

当视频解码线程解码出一帧后，通过 `AddVideoPicture()` 将 YUV 数据拷贝到环形缓冲区。

```cpp
// 源码：KRMoviePlayer.cpp 第 176-228 行
bool TVPMoviePlayer::AddVideoPicture(DVDVideoPicture& pic) {
    // 1. 仅接受 YUV420P 格式
    if (pic.format != RENDER_FMT_YUV420P) {
        return false;
    }
    
    // 2. 获取写入槽位
    int idx = m_writeIdx % MAX_BUFFER_COUNT;
    YUVBuffer& buf = m_buffers[idx];
    
    // 3. 分配对齐内存（首次或尺寸变化时）
    int ySize = pic.iWidth * pic.iHeight;
    int uvSize = (pic.iWidth / 2) * (pic.iHeight / 2);
    
    if (!buf.data[0]) {
        buf.data[0] = (uint8_t*)TJSAlignedAlloc(ySize, 32);
        buf.data[1] = (uint8_t*)TJSAlignedAlloc(uvSize, 32);
        buf.data[2] = (uint8_t*)TJSAlignedAlloc(uvSize, 32);
    }
    
    // 4. 逐行拷贝 YUV 数据
    // Y 平面
    for (int i = 0; i < pic.iHeight; i++) {
        memcpy(buf.data[0] + i * pic.iWidth,
               pic.data[0] + i * pic.iLineSize[0],
               pic.iWidth);
    }
    // U 平面
    for (int i = 0; i < pic.iHeight / 2; i++) {
        memcpy(buf.data[1] + i * (pic.iWidth / 2),
               pic.data[1] + i * pic.iLineSize[1],
               pic.iWidth / 2);
    }
    // V 平面
    for (int i = 0; i < pic.iHeight / 2; i++) {
        memcpy(buf.data[2] + i * (pic.iWidth / 2),
               pic.data[2] + i * pic.iLineSize[2],
               pic.iWidth / 2);
    }
    
    // 5. 记录 PTS
    buf.pts = pic.pts;
    buf.linesize[0] = pic.iWidth;
    buf.linesize[1] = pic.iWidth / 2;
    buf.linesize[2] = pic.iWidth / 2;
    
    // 6. 推进写指针
    m_writeIdx++;
    
    return true;
}
```

**为什么用 `TJSAlignedAlloc`？** 对齐内存（32字节对齐）有助于 SIMD 指令（SSE/NEON）
加速后续的 YUV→RGB 转换操作。

### PresentPicture：渲染到屏幕

`VideoPresentOverlay::PresentPicture()` 由 Cocos2d-x 的调度器在主线程调用，
负责将 YUV 缓冲区中的帧渲染到屏幕上。

```cpp
// 源码：KRMoviePlayer.cpp 第 239-295 行
void VideoPresentOverlay::PresentPicture() {
    // 这个函数在 Cocos2d-x 主线程中被调度器调用
    
    // 1. 获取当前播放时钟
    double clock = m_pPlayer->GetClock();
    
    // 2. 查找最合适的帧（PTS 最接近当前时钟的）
    int bestIdx = -1;
    double bestDiff = DBL_MAX;
    
    for (int i = m_readIdx; i < m_pPlayer->m_writeIdx; i++) {
        int idx = i % MAX_BUFFER_COUNT;
        double diff = m_pPlayer->m_buffers[idx].pts - clock;
        
        // 选择已经到显示时间且最接近当前时钟的帧
        if (diff <= 0 && fabs(diff) < bestDiff) {
            bestDiff = fabs(diff);
            bestIdx = idx;
        }
    }
    
    if (bestIdx < 0) return;  // 没有可显示的帧
    
    // 3. 跳过中间的帧（帧跳跃）
    // 如果积压了多帧，直接跳到最新的可显示帧
    m_readIdx = bestIdx + 1;
    
    // 4. 使用 TVPYUVSprite 渲染 YUV 数据
    YUVBuffer& buf = m_pPlayer->m_buffers[bestIdx];
    m_pYUVSprite->updateYUV(
        buf.data[0], buf.linesize[0],
        buf.data[1], buf.linesize[1],
        buf.data[2], buf.linesize[2],
        m_width, m_height
    );
}
```

**关键设计决策：**
- **PTS 比较选帧**而非简单的 FIFO，允许在延迟时跳过中间帧
- **YUVSprite** 使用 GPU shader 做 YUV→RGB 转换，CPU 零开销
- **环形缓冲区**解耦了解码线程和渲染线程的速率差异

## CDVDClock：时钟系统

时钟是音视频同步的基准。KrKr2 使用微秒精度的时钟系统。

源码位置：`cpp/core/movie/ffmpeg/Clock.cpp`（276 行）

### 关键常量

```cpp
// Clock.h 中定义的常量
#define DVD_TIME_BASE      1000000  // 微秒为单位（1秒 = 1000000）
#define DVD_NOPTS_VALUE    0xFFF0000000000000  // 无效 PTS 标记
#define DVD_PLAYSPEED_NORMAL 1000   // 正常速度 = 1000（支持 0.5x~2x）
```

### 时钟实现

```cpp
// Clock.cpp 核心方法

double CDVDClock::GetClock() {
    std::lock_guard<std::mutex> lock(m_critSection);
    
    // 当前时间 = 基准时间 + (系统时钟 - 上次更新时的系统时钟) × 播放速率
    int64_t now = CurrentHostCounter();
    double elapsed = (double)(now - m_startClock) / m_systemFrequency;
    
    return m_iDisc + elapsed * m_iSpeed / DVD_PLAYSPEED_NORMAL;
}

void CDVDClock::Discontinuity(double pts) {
    std::lock_guard<std::mutex> lock(m_critSection);
    
    // seek 后重置时钟基准
    m_startClock = CurrentHostCounter();
    m_iDisc = pts;  // 新的基准 PTS
}
```

**`DVD_NOPTS_VALUE` 的用法：** 值为 `0xFFF0000000000000`（一个极大的正数），
用于标记"无有效时间戳"。代码中大量使用：

```cpp
// 常见模式
if (pts == DVD_NOPTS_VALUE) {
    // 该数据包没有时间戳，用帧率推算
    pts = m_lastPts + frameDuration;
}
```

## 一帧视频的完整旅程

现在让我们追踪一帧视频从文件到屏幕的完整路径：

```
步骤 1: 文件 → AVPacket
━━━━━━━━━━━━━━━━━━━
CDVDDemuxFFmpeg::Read()
  └── av_read_frame(m_pFormatContext, &pkt)
      // 从容器中读取一个压缩数据包
      // 文件：DemuxFFmpeg.cpp

步骤 2: AVPacket → 消息队列
━━━━━━━━━━━━━━━━━━━
BasePlayer::ProcessPacket()
  └── CDVDMsgDemuxerPacket* msg = new CDVDMsgDemuxerPacket(pkt)
      m_VideoPlayerVideo->SendMessage(msg)
      // 数据包封装为消息，发送到视频线程
      // 文件：VideoPlayer.cpp

步骤 3: 消息队列 → 解码
━━━━━━━━━━━━━━━━━━━
CVideoPlayerVideo::Process()
  └── m_messageQueue.Get(&pMsg, 1000)
      m_pVideoCodec->Decode(pData, iSize, dts, pts)
      // 视频线程取出消息，调用 FFmpeg 解码
      // 文件：VideoPlayerVideo.cpp

步骤 4: FFmpeg 解码
━━━━━━━━━━━━━━━━━━━
CDVDVideoCodecFFmpeg::Decode()
  └── avcodec_decode_video2(m_pCodecContext, m_pFrame, &got, &pkt)
      // 注意：使用的是旧版 API（非 send_packet/receive_frame）
      // 文件：VideoCodecFFmpeg.cpp

步骤 5: 解码帧 → 时序控制
━━━━━━━━━━━━━━━━━━━
CVideoPlayerVideo::OutputPicture()
  ├── 计算 PTS 与主时钟偏差
  ├── CalcDropRequirement() → 判断是否丢帧
  ├── WaitForBuffer() → 等待渲染队列有空位
  └── AddVideoPicture() + FlipPage()
      // 文件：VideoPlayerVideo.cpp

步骤 6: 渲染队列 → KrKr2 环形缓冲
━━━━━━━━━━━━━━━━━━━
TVPMoviePlayer::AddVideoPicture()
  └── memcpy YUV420P 数据到 m_buffers[idx]
      // 仅接受 YUV420P 格式
      // 使用 TJSAlignedAlloc 32 字节对齐
      // 文件：KRMoviePlayer.cpp

步骤 7: 环形缓冲 → 屏幕
━━━━━━━━━━━━━━━━━━━
VideoPresentOverlay::PresentPicture()
  ├── 按 PTS 选取最合适的帧
  ├── 跳过已过期的中间帧
  └── m_pYUVSprite->updateYUV(Y, U, V)
      // GPU shader 完成 YUV→RGB 转换并显示
      // 由 Cocos2d-x 调度器在主线程调用
      // 文件：KRMoviePlayer.cpp
```

### 跨线程数据流总结

```
┌──────────┐    AVPacket    ┌───────────────┐    CDVDMsg    ┌─────────────────┐
│ DemuxFFmpeg │──────────────→│ BasePlayer    │──────────────→│ VideoPlayerVideo │
│ (主线程)   │              │ (主线程)      │              │ (视频线程)       │
└──────────┘              └───────────────┘              └────────┬────────┘
                                                                  │ DVDVideoPicture
                                                                  ▼
┌──────────────┐   YUV memcpy   ┌───────────────┐   PTS 选帧    ┌──────────────┐
│ YUVSprite    │←───────────────│ TVPMoviePlayer │←──────────────│ RenderManager │
│ (GPU 渲染)   │              │ (环形缓冲)    │              │ (帧调度)      │
└──────────────┘              └───────────────┘              └──────────────┘
```

## 常见问题与调试技巧

### 问题 1：视频播放卡顿

**症状：** 视频播放时画面不流畅，有明显的顿挫感。

**排查步骤：**

```cpp
// 1. 检查丢帧统计
// 在 VideoPlayerVideo.cpp 的 CalcDropRequirement() 中添加日志
SPDLOG_DEBUG("drop stats: count={}, lateframes={}, lateness={}ms",
    m_droppingStats.m_dropcount,
    m_droppingStats.m_lateframes,
    iLateness / 1000.0);

// 2. 检查渲染队列状态
// 在 VideoRenderer.cpp 的 GetStats() 中查看
// free.size()=0 表示渲染跟不上解码
// queued.size() 持续增长表示显示跟不上

// 3. 常见原因
// - CPU 不够快，解码速度跟不上帧率
// - 环形缓冲区太小（MAX_BUFFER_COUNT=4）
// - Cocos2d-x 调度器频率太低
```

**解决方案：**

```cpp
// 方案 A：增加环形缓冲区大小
// KRMoviePlayer.cpp
#define MAX_BUFFER_COUNT 8  // 原值 4，增加到 8

// 方案 B：调整丢帧阈值（更积极地丢帧以保持同步）
// VideoPlayerVideo.cpp CalcDropRequirement()
if (iLateness > m_fFrameTime * 0.5) {  // 原值 1.0，改为 0.5
    return 1;
}
```

### 问题 2：音视频不同步

**症状：** 画面和声音对不上，越播越明显。

**排查步骤：**

```cpp
// 1. 检查时钟漂移
// 在 Clock.cpp 的 GetClock() 中添加日志
double clockValue = GetClock();
double audioPts = m_audioPlayer->GetCurrentPts();
SPDLOG_DEBUG("clock drift: clock={:.3f}s, audio={:.3f}s, diff={:.3f}ms",
    clockValue / DVD_TIME_BASE,
    audioPts / DVD_TIME_BASE,
    (clockValue - audioPts) / 1000.0);

// 2. 检查同步启动是否正常
// VideoPlayer.cpp SynchronizePlayers()
// 两个流的 syncState 是否都到达了 SYNC_WAITSYNC
```

### 问题 3：seek 后黑屏或花屏

**症状：** 拖动进度条后画面黑屏，或显示花屏（马赛克）。

**原因分析：**
```
seek 后的正常流程：
1. BasePlayer 发送 GENERAL_FLUSH 给所有子线程
2. 子线程清空缓冲区和解码器状态
3. DemuxFFmpeg 执行 av_seek_frame()
4. 从新位置开始读取，等待下一个 I 帧
5. I 帧解码后画面恢复正常

黑屏原因：
- FLUSH 消息没有正确传达（检查消息队列是否被 Abort）
- seek 后没有等到 I 帧就开始显示（花屏）

花屏原因：
- seek 目标位置不在 I 帧边界
- 解码器状态没有正确重置（avcodec_flush_buffers 未调用）
```

```cpp
// 修复花屏：确保 seek 后重置解码器
// VideoCodecFFmpeg.cpp Reset() 方法
void CDVDVideoCodecFFmpeg::Reset() {
    if (m_pCodecContext) {
        avcodec_flush_buffers(m_pCodecContext);  // 必须调用！
    }
    m_iLastKeyframe = 0;
}
```

### 问题 4：内存泄漏

```cpp
// CDVDMsg 使用引用计数管理生命周期
// 每个消息创建时 refcount=1
// Get() 取出后由消费者负责 Release()

// 常见泄漏模式：
CDVDMsg* pMsg;
m_messageQueue.Get(&pMsg, 1000);
if (pMsg->GetType() == CDVDMsg::DEMUXER_PACKET) {
    // 处理...
    return;  // ❌ 忘记 Release！
}
pMsg->Release();

// 正确做法：使用 RAII 或确保所有路径都 Release
CDVDMsg* pMsg;
m_messageQueue.Get(&pMsg, 1000);
// ... 处理 ...
pMsg->Release();  // ✅ 所有路径都必须到达这里
```

## 动手实践

### 实践 1：添加播放统计信息输出

在 `VideoRenderer.cpp` 的 `GetStats()` 方法基础上，添加一个周期性日志输出：

```cpp
// 在 CRenderManager::FrameMove() 中添加
void CRenderManager::FrameMove() {
    // 原有代码...
    
    // 每 60 帧输出一次统计
    static int frameCount = 0;
    frameCount++;
    if (frameCount % 60 == 0) {
        SPDLOG_INFO(
            "[PlayStats] free={}, queued={}, discard={}, "
            "lateframes={}, pts={:.3f}s",
            m_free.size(), m_queued.size(), m_discard.size(),
            m_lateframes,
            m_presentpts / DVD_TIME_BASE
        );
    }
}
```

### 实践 2：调整缓冲策略

尝试修改 `NUM_BUFFERS` 和 `MAX_BUFFER_COUNT`，观察对播放流畅度的影响：

```cpp
// 实验步骤：
// 1. 在 VideoRenderer.cpp 中修改 NUM_BUFFERS
//    原值：6，尝试改为 3 和 12，对比效果

// 2. 在 KRMoviePlayer.cpp 中修改 MAX_BUFFER_COUNT
//    原值：4，尝试改为 2 和 8，对比效果

// 3. 观察指标：
//    - 内存占用变化（YUV 缓冲区大小 = width×height×1.5×count）
//    - 播放启动延迟（缓冲区越大，首帧等待越久）
//    - 丢帧率（缓冲区越小，丢帧可能越多）

// 对于 1920×1080 视频：
// NUM_BUFFERS=6:  内存 ≈ 1920×1080×1.5×6 ≈ 18.7MB
// NUM_BUFFERS=12: 内存 ≈ 1920×1080×1.5×12 ≈ 37.3MB
```

## 对照项目源码

本节分析的源码文件清单：

| 文件 | 行数 | 核心内容 |
|------|------|----------|
| `cpp/core/movie/ffmpeg/VideoPlayer.cpp` | 2906 | BasePlayer 主循环、缓存状态机、同步启动 |
| `cpp/core/movie/ffmpeg/VideoPlayerVideo.cpp` | 1104 | 视频解码线程、OutputPicture、丢帧策略 |
| `cpp/core/movie/ffmpeg/VideoPlayerAudio.cpp` | 606 | 音频解码线程 |
| `cpp/core/movie/ffmpeg/VideoRenderer.cpp` | 1274 | CRenderManager 帧缓冲管理、PrepareNextRender |
| `cpp/core/movie/ffmpeg/KRMoviePlayer.cpp` | 382 | TVPMoviePlayer 桥梁层、YUV 环形缓冲 |
| `cpp/core/movie/ffmpeg/MessageQueue.cpp` | 277 | CDVDMessageQueue 优先级消息队列 |
| `cpp/core/movie/ffmpeg/Clock.cpp` | 276 | CDVDClock 主时钟 |
| `cpp/core/movie/ffmpeg/VideoCodecFFmpeg.cpp` | 1113 | FFmpeg 视频解码封装 |
| `cpp/core/movie/ffmpeg/AudioCodecFFmpeg.cpp` | 360 | FFmpeg 音频解码封装 |
| `cpp/core/movie/ffmpeg/DemuxFFmpeg.cpp` | 1656 | FFmpeg 解封装封装 |

**重点阅读建议：**
1. 先读 `KRMoviePlayer.cpp`（382 行）—— 最短，理解桥梁层设计
2. 再读 `MessageQueue.cpp`（277 行）—— 理解线程通信机制
3. 然后 `VideoPlayerVideo.cpp`（1104 行）—— 理解解码和时序控制
4. 最后 `VideoPlayer.cpp`（2906 行）—— 理解主控逻辑

## 本节小结

- KrKr2 的媒体播放器移植自 **Kodi/XBMC**，保留了多线程管线架构，裁剪了字幕、网络流等不需要的功能
- **CDVDMsg 消息系统**是线程间通信的核心，支持优先级消息确保控制命令（seek、flush）优先处理
- **BasePlayer::Process()** 是主循环，负责解封装和数据包分发，通过**缓存状态机**管理播放流程
- **CVideoPlayerVideo** 在独立线程中解码，通过 **OutputPicture()** 控制帧时序和丢帧策略
- **CRenderManager** 管理 6 帧缓冲队列（free→queued→present→discard），实现 **VSync 对齐**
- **TVPMoviePlayer** 是桥梁层，用 **YUV 环形缓冲区**连接 Kodi 播放器和 Cocos2d-x 渲染
- 使用的是**旧版 FFmpeg API**（`avcodec_decode_video2`），源自较早版本的 Kodi 移植
- 时钟精度为**微秒**（`DVD_TIME_BASE = 1000000`），无效时间戳用 `DVD_NOPTS_VALUE` 标记

## 练习题与答案

### 题目 1：消息优先级分析

假设视频正在播放中，用户点击了 seek 按钮。此时 `VideoPlayerVideo` 的消息队列中有以下消息：

```
队列内容（队首→队尾）：
1. DEMUXER_PACKET (priority=0, PTS=5.0s)
2. DEMUXER_PACKET (priority=0, PTS=5.033s)
3. DEMUXER_PACKET (priority=0, PTS=5.067s)
4. GENERAL_FLUSH  (priority=1)    ← seek 触发
5. GENERAL_RESYNC (priority=1, PTS=30.0s)  ← seek 目标
```

请问：这 5 条消息的实际处理顺序是什么？为什么？

<details>
<summary>查看答案</summary>

**实际处理顺序：**

```
1. GENERAL_FLUSH  (priority=1)    ← 第一个被处理
2. GENERAL_RESYNC (priority=1)    ← 第二个被处理
3. DEMUXER_PACKET (PTS=5.0s)      ← 但这些会被 FLUSH 清空
4. DEMUXER_PACKET (PTS=5.033s)    ← 实际上不会被处理
5. DEMUXER_PACKET (PTS=5.067s)    ← 因为 FLUSH 会重置状态
```

**原因：** `CDVDMessageQueue::Get()` 优先返回 `priority > 0` 的消息。
即使 `GENERAL_FLUSH` 在队列的第 4 位置，它会被优先取出。

处理 `GENERAL_FLUSH` 后，解码器状态被重置，剩余的旧 `DEMUXER_PACKET`
即使被取出也会因为 PTS 不匹配（它们是 seek 前的数据）而被丢弃。

`GENERAL_RESYNC` 设置新的基准 PTS（30.0s），之后只有 seek 目标位置的新数据包才会被正常解码。

</details>

### 题目 2：帧缓冲队列状态分析

CRenderManager 的 `NUM_BUFFERS=6`，在某一时刻队列状态如下：

```
free:    [0, 1]        (2个空闲)
queued:  [2, 3, 4]     (3个等待显示)
present: 5             (1个正在显示)
discard: []            (0个待回收)
```

问题：
1. 此时如果解码线程调用 `WaitForBuffer()`，会发生什么？
2. 如果此时又解码出 3 帧新帧，会发生什么？
3. 这种队列状态说明什么问题？

<details>
<summary>查看答案</summary>

**1. `WaitForBuffer()` 的行为：**

`WaitForBuffer()` 检查 `m_free` 是否为空。当前 `m_free` 有 2 个空闲槽位（0 和 1），
所以 `WaitForBuffer()` **立即返回 true**，不会阻塞。

**2. 解码出 3 帧新帧：**

- 第 1 帧：从 `m_free` 取出索引 0，填入数据，放入 `m_queued` → `free=[1], queued=[2,3,4,0]`
- 第 2 帧：从 `m_free` 取出索引 1，填入数据，放入 `m_queued` → `free=[], queued=[2,3,4,0,1]`
- 第 3 帧：`m_free` 为空！`WaitForBuffer()` **阻塞**，等待渲染线程消费帧并回收到 `m_free`

此时需要 `FrameMove()` 被调用，执行：
```
PrepareNextRender():  present=5 → discard, present=2 (从queued取出)
回收: discard=[5] → free=[5]
```
然后解码线程的 `WaitForBuffer()` 才能返回。

**3. 队列状态分析：**

`queued` 有 3 帧等待显示，说明**渲染速度低于解码速度**。可能原因：
- Cocos2d-x 调度器频率低于视频帧率
- `PrepareNextRender()` 的时序计算导致帧积压
- 如果这种状态持续，说明需要更积极的丢帧策略

正常情况下 `queued` 应保持 1-2 帧（`QueueSize` 的默认值是 2）。
</details>

### 题目 3：修改播放器支持新的像素格式

当前 `TVPMoviePlayer::AddVideoPicture()` 仅接受 `RENDER_FMT_YUV420P` 格式。
如果要支持 `RENDER_FMT_NV12`（Y + 交错 UV 的双平面格式，常见于硬件解码器输出），
需要修改哪些地方？请写出关键代码修改。

<details>
<summary>查看答案</summary>

需要修改 3 个位置：

**1. `KRMoviePlayer.cpp` — `AddVideoPicture()` 添加 NV12 支持：**

```cpp
bool TVPMoviePlayer::AddVideoPicture(DVDVideoPicture& pic) {
    // 支持 YUV420P 和 NV12 两种格式
    if (pic.format != RENDER_FMT_YUV420P && 
        pic.format != RENDER_FMT_NV12) {
        return false;
    }
    
    int idx = m_writeIdx % MAX_BUFFER_COUNT;
    YUVBuffer& buf = m_buffers[idx];
    
    int ySize = pic.iWidth * pic.iHeight;
    
    if (pic.format == RENDER_FMT_YUV420P) {
        // 原有的 YUV420P 处理代码（3 个平面）
        int uvSize = (pic.iWidth / 2) * (pic.iHeight / 2);
        // ... 原代码不变 ...
    } 
    else if (pic.format == RENDER_FMT_NV12) {
        // NV12: Y 平面 + 交错 UV 平面
        int uvSize = pic.iWidth * (pic.iHeight / 2);  // UV 交错，宽度不减半
        
        if (!buf.data[0]) {
            buf.data[0] = (uint8_t*)TJSAlignedAlloc(ySize, 32);
            buf.data[1] = (uint8_t*)TJSAlignedAlloc(uvSize, 32);
            buf.data[2] = nullptr;  // NV12 没有第三个平面
        }
        
        // 拷贝 Y 平面
        for (int i = 0; i < pic.iHeight; i++) {
            memcpy(buf.data[0] + i * pic.iWidth,
                   pic.data[0] + i * pic.iLineSize[0],
                   pic.iWidth);
        }
        // 拷贝 UV 交错平面
        for (int i = 0; i < pic.iHeight / 2; i++) {
            memcpy(buf.data[1] + i * pic.iWidth,
                   pic.data[1] + i * pic.iLineSize[1],
                   pic.iWidth);
        }
    }
    
    buf.pts = pic.pts;
    buf.format = pic.format;  // 需要在 YUVBuffer 中新增 format 字段
    m_writeIdx++;
    return true;
}
```

**2. `YUVBuffer` 结构体添加 format 字段：**

```cpp
struct YUVBuffer {
    uint8_t* data[3];
    int linesize[3];
    double pts;
    int format;  // 新增：记录像素格式
};
```

**3. `VideoPresentOverlay::PresentPicture()` 根据格式选择 shader：**

```cpp
void VideoPresentOverlay::PresentPicture() {
    // ... 选帧逻辑不变 ...
    
    YUVBuffer& buf = m_pPlayer->m_buffers[bestIdx];
    
    if (buf.format == RENDER_FMT_YUV420P) {
        // 原有的 YUV420P 渲染（3 平面）
        m_pYUVSprite->updateYUV(
            buf.data[0], buf.linesize[0],
            buf.data[1], buf.linesize[1],
            buf.data[2], buf.linesize[2],
            m_width, m_height);
    } else if (buf.format == RENDER_FMT_NV12) {
        // NV12 渲染（2 平面，需要不同的 shader）
        m_pYUVSprite->updateNV12(
            buf.data[0], buf.linesize[0],  // Y
            buf.data[1], buf.linesize[1],  // UV 交错
            m_width, m_height);
    }
}
```

**注意：** 还需要在 `TVPYUVSprite` 中实现 `updateNV12()` 方法和对应的 NV12 fragment shader。
NV12 shader 的主要区别是从 UV 纹理中同时采样 U 和 V（而非两个独立纹理）。
</details>

## 下一步

恭喜你完成了 P06 模块的全部 7 章！你现在已经具备了：
- 音视频编解码的基础理论知识
- FFmpeg C API 的实际使用能力
- 完整视频播放器的开发经验
- KrKr2 媒体播放器架构的深入理解

接下来推荐学习：
- [P07-OpenAL音频编程](../P07-OpenAL音频编程/README.md) — 了解 3D 音频和音频引擎
- [M05-音频子系统](../M05-音频子系统/README.md) — 深入 KrKr2 的音频解码器和 DSP 链路
- [M06-视频播放器](../M06-视频播放器/README.md) — 更深入的 movie/ffmpeg 模块分析
