# 架构总览与 Kodi 遗产

> **所属模块：** P06-FFmpeg音视频开发
> **前置知识：** [解封装](../03-解封装从容器到数据包/01-解封装概念与数据结构.md)、[解码](../04-解码从数据包到原始帧/01-解码器基础与视频解码.md)、[音视频同步](../05-音视频同步/01-同步基础与时间戳.md)、[实战播放器](../06-实战FFmpeg-SDL2视频播放器/01-项目结构与数据结构设计.md)
> **预计阅读时间：** 45 分钟

## 本节目标

读完本节后，你将能够：
1. 说出 KrKr2 媒体播放器的整体架构——从 TVPMoviePlayer 入口层到 BasePlayer 主控层、再到 Demux/Decode/Render 数据处理层的四层结构
2. 解释 Kodi（一个开源媒体中心软件，原名 XBMC）播放器内核在 KrKr2 中的保留与裁剪策略
3. 画出一帧视频从文件读取到屏幕显示的完整数据流路径
4. 理解 CDVDMsg 消息系统如何驱动线程间通信，以及消息优先级的实际意义
5. 列出源码目录中每个文件的职责，快速定位想看的功能

---

## 为什么 KrKr2 不自己写播放器？

在第 6 章中，我们从零实现了一个约 500 行的 SDL2 播放器。它能工作，但存在明显不足：

- **丢帧策略粗糙**：只有简单的"跳过延迟帧"，没有基于缓冲区水位的智能判断
- **缓存管理缺失**：没有预缓冲（prebuffering，即播放前预先读取一定量的数据到内存中以防止卡顿）机制，网络抖动或磁盘 IO 慢时直接卡顿
- **Seek 处理简陋**：没有处理 seek 后的缓冲区残留数据问题
- **无播放速度控制**：不支持快进/慢放
- **硬件加速**：没有对接硬件解码器（如 DXVA2、VideoToolbox 等）的框架

KrKr2 需要在视觉小说中播放 OP/ED 视频和背景动画，要求音视频精确同步、支持多种格式（MPEG、H.264、VP9 等）、能输出到 Cocos2d-x（KrKr2 使用的游戏渲染引擎）纹理、且跨平台运行。

自己从头写？工作量巨大，且难以达到生产级质量。更明智的选择是——站在巨人肩上。

### Kodi 播放器内核

Kodi（原名 XBMC，Xbox Media Center）是一个有十多年历史的开源媒体中心软件，其播放器内核经过了海量用户的生产环境验证。KrKr2 团队将 Kodi 的核心播放逻辑移植过来，裁剪掉了视觉小说不需要的功能（字幕渲染、网络流、蓝光导航等），保留了其精髓：

- **成熟的多线程架构**：解封装、视频解码、音频解码各自独立线程
- **完善的缓存状态机**：FLUSH → FULL → INIT → PLAY → DONE 五状态管理
- **精确的时钟同步**：基于 DVD_TIME_BASE（即 `1000000.0`，单位为微秒）的高精度时钟
- **智能丢帧算法**：根据缓冲区水位和延迟帧数动态调整丢帧策略

> **识别 Kodi 遗产代码的标志**：源码中大量使用 `DVD_` 前缀（如 `DVD_NOPTS_VALUE`、`DVD_TIME_BASE`），这是 Kodi 早期以 DVD 播放为核心功能时的命名遗留。看到 `DVD_` 开头的常量或类名，就知道这是 Kodi 血统。

---

## 源码目录结构

KrKr2 的视频播放器源码位于 `cpp/core/movie/ffmpeg/` 目录下。我们先逐文件了解每个文件的职责：

```
cpp/core/movie/ffmpeg/
├── KRMoviePlayer.cpp/h    ← 入口层：TVPMoviePlayer 类
│                             实现 iTVPVideoOverlay 接口（KrKr2 引擎定义的视频叠加接口）
│                             同时继承 CBaseRenderer（Kodi 定义的渲染器基类）
│                             管理 YUV 环形缓冲区，是两个系统之间的"桥梁"
│
├── VideoPlayer.cpp/h      ← 主控层：BasePlayer 类
│                             继承 CThread（Kodi 的线程基类），运行主循环 Process()
│                             协调解封装、解码、渲染所有子组件
│                             包含缓存状态机和 Seek 处理逻辑
│
├── DemuxFFmpeg.cpp/h      ← 解封装层：CDVDDemuxFFmpeg 类
│                             封装 AVFormatContext（FFmpeg 的容器格式上下文）
│                             负责打开文件、读取 AVPacket（压缩数据包）、Seek 操作
│
├── VideoCodecFFmpeg.cpp/h ← 视频解码器：CDVDVideoCodecFFmpeg 类
│                             封装 AVCodecContext（FFmpeg 的编解码器上下文）
│                             使用旧版 API：avcodec_decode_video2()（注意不是新版的
│                             avcodec_send_packet/avcodec_receive_frame）
│
├── AudioCodecFFmpeg.cpp/h ← 音频解码器：CDVDAudioCodecFFmpeg 类
│                             同样封装 AVCodecContext，使用 avcodec_decode_audio4()
│
├── VideoPlayerVideo.cpp/h ← 视频解码线程：CVideoPlayerVideo 类
│                             独立线程，从消息队列取数据包 → 解码 → 送渲染管理器
│                             包含丢帧判定逻辑（CalcDropRequirement）
│
├── VideoPlayerAudio.cpp/h ← 音频解码线程：CVideoPlayerAudio 类
│                             独立线程，从消息队列取数据包 → 解码 → 送音频引擎
│                             包含音频时钟同步逻辑
│
├── VideoRenderer.cpp/h    ← 渲染管理器：CRenderManager 类
│                             管理帧缓冲队列（free → queued → present → discard 循环）
│                             负责 VSync（垂直同步，即让画面刷新与显示器刷新率对齐）对齐
│                             和延迟补偿
│
├── Clock.cpp/h            ← 主时钟：CDVDClock 类
│                             微秒精度的播放时钟，支持速度调整和不连续点（discontinuity，
│                             即时间线跳跃，通常发生在 seek 操作后）处理
│
├── MessageQueue.cpp/h     ← 消息队列：CDVDMessageQueue 类
│                             线程间通信的核心机制，支持优先级消息
│
├── KRMovieLayer.cpp/h     ← Cocos2d-x 图层：VideoPresentLayer / VideoPresentOverlay 类
│                             负责将 YUV 数据转换为纹理并在 Cocos2d-x 场景中渲染
│
├── AE.h                   ← 音频引擎接口：IAE 抽象接口
│                             KrKr2 实现了自己的音频引擎来替代 Kodi 原有的
│
└── OMXClock.h             ← 空桩文件（Kodi 遗留）
                              原本用于树莓派（Raspberry Pi）的硬件时钟接口
                              在 KrKr2 中保留为空实现以维持编译通过，勿删
```

### 文件职责速查表

| 文件 | 核心类 | 线程模型 | 一句话职责 |
|------|--------|----------|-----------|
| `KRMoviePlayer.cpp` | TVPMoviePlayer | 被调用方（主线程 + 渲染线程） | KrKr2 ↔ Kodi 桥梁，管理 YUV 环形缓冲 |
| `VideoPlayer.cpp` | BasePlayer | 独立线程（Process 主循环） | 总指挥：读包、分发、管缓存、处理 Seek |
| `DemuxFFmpeg.cpp` | CDVDDemuxFFmpeg | 被 BasePlayer 调用 | 打开文件、读 AVPacket、Seek |
| `VideoCodecFFmpeg.cpp` | CDVDVideoCodecFFmpeg | 被 CVideoPlayerVideo 调用 | 视频解码：压缩数据 → YUV 帧 |
| `AudioCodecFFmpeg.cpp` | CDVDAudioCodecFFmpeg | 被 CVideoPlayerAudio 调用 | 音频解码：压缩数据 → PCM 采样 |
| `VideoPlayerVideo.cpp` | CVideoPlayerVideo | 独立线程 | 视频解码 + 丢帧判定 + 送渲染 |
| `VideoPlayerAudio.cpp` | CVideoPlayerAudio | 独立线程 | 音频解码 + 时钟同步 + 送音频引擎 |
| `VideoRenderer.cpp` | CRenderManager | 帧队列管理（跨线程） | 帧缓冲队列 + VSync 对齐 |
| `Clock.cpp` | CDVDClock | 被多线程读取 | 播放时钟：时间基准 + 速度调整 |
| `MessageQueue.cpp` | CDVDMessageQueue | 线程安全队列 | 优先级消息队列（生产者-消费者） |

---

## 四层架构详解

KrKr2 播放器不是一个扁平结构，而是清晰的四层分工。理解这四层是读懂全部源码的基础：

```
┌─────────────────────────────────────────────────────────────┐
│                    第 1 层：引擎入口层                         │
│                                                              │
│  TVPMoviePlayer (KRMoviePlayer.cpp)                         │
│  ├── 实现 iTVPVideoOverlay 接口，供 KrKr2 TJS2 脚本调用     │
│  ├── 同时继承 CBaseRenderer，让 Kodi 播放管线能输出画面到这  │
│  ├── 管理 YUV420P 环形缓冲区（MAX_BUFFER_COUNT = 4 帧）     │
│  └── 通过 Cocos2d-x Scheduler 定期触发渲染（PresentPicture）│
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                    第 2 层：播放控制层                         │
│                                                              │
│  BasePlayer (VideoPlayer.cpp)                               │
│  ├── 主循环线程 Process()：读包 → 分发 → 管理状态            │
│  ├── 缓存状态机：FLUSH → FULL → INIT → PLAY → DONE          │
│  ├── HandleMessages()：处理 Seek、速度变更等控制命令          │
│  └── HandlePlaySpeed()：管理缓存水位、协调音视频同步启动      │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                    第 3 层：数据处理层                         │
│                                                              │
│  ┌───────────────────┐  ┌────────────────────────────────┐  │
│  │  CDVDDemuxFFmpeg   │  │       CDVDClock                │  │
│  │  解封装            │  │       主时钟                    │  │
│  │  文件 → AVPacket   │  │       微秒精度时间基准          │  │
│  └────────┬──────────┘  └────────────────────────────────┘  │
│           │                                                  │
│           │ AVPacket（压缩数据包，通过 CDVDMsg 消息分发）     │
│           │                                                  │
│  ┌────────┴──────────┐  ┌────────────────────────────────┐  │
│  │ CVideoPlayerVideo  │  │    CVideoPlayerAudio           │  │
│  │ 视频解码线程       │  │    音频解码线程                 │  │
│  │ AVPacket → YUV帧   │  │    AVPacket → PCM采样           │  │
│  └────────┬──────────┘  └────────────────────────────────┘  │
│           │                                                  │
│           │ VideoPicture（解码后的 YUV 帧数据）              │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                    第 4 层：渲染管理层                         │
│                                                              │
│  CRenderManager (VideoRenderer.cpp)                         │
│  ├── 帧缓冲队列：free → queued → present → discard 循环     │
│  ├── PrepareNextRender()：选择最接近当前时钟的帧显示          │
│  ├── 智能丢帧：延迟 ≤6 帧时有宽限期，>10 帧时零容忍         │
│  └── VSync 时钟同步：累积 30 帧渲染误差后微调时钟            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 各层之间的调用关系

```
KrKr2 TJS2 脚本
    │  调用 Movie.open("video.mp4")
    ▼
TVPMoviePlayer::Open()
    │  创建 BasePlayer 实例并启动线程
    ▼
BasePlayer::Process()  ←── 独立线程，死循环运行
    │
    ├── ReadPacket()
    │   └── CDVDDemuxFFmpeg::Read()  →  返回 AVPacket
    │
    ├── ProcessPacket()
    │   ├── 视频包 → CDVDMsgDemuxerPacket → CVideoPlayerVideo 的消息队列
    │   └── 音频包 → CDVDMsgDemuxerPacket → CVideoPlayerAudio 的消息队列
    │
    ├── CVideoPlayerVideo::Process()  ←── 独立线程
    │   ├── 从消息队列取出 CDVDMsgDemuxerPacket
    │   ├── CDVDVideoCodecFFmpeg::Decode()  →  解码为 YUV 帧
    │   └── OutputPicture()  →  CRenderManager::AddVideoPicture()
    │                              └── TVPMoviePlayer::AddVideoPicture()
    │                                   （写入 YUV 环形缓冲区）
    │
    └── CVideoPlayerAudio::Process()  ←── 独立线程
        ├── 从消息队列取出 CDVDMsgDemuxerPacket
        ├── CDVDAudioCodecFFmpeg::Decode()  →  解码为 PCM 采样
        └── 送入音频引擎播放

Cocos2d-x 渲染循环（每帧调用）
    │
    └── VideoPresentOverlay::PresentPicture()
        └── 从 YUV 环形缓冲区读取帧 → 创建纹理 → 绘制到屏幕
```

> **关键发现**：在 KrKr2 的实现中，`CRenderManager::AddVideoPicture()` 直接委托给了 `TVPMoviePlayer::AddVideoPicture()`（即 `m_pRenderer->AddVideoPicture()`），绕过了 CRenderManager 自身的帧队列（free/queued/discard 双端队列）。也就是说 TVPMoviePlayer **就是**渲染器，它直接管理环形缓冲区。CRenderManager 的队列逻辑在这个架构中被旁路了。

---

## Kodi 遗产：保留了什么、裁剪了什么

理解哪些是 Kodi 原始代码、哪些是 KrKr2 定制代码，对阅读源码至关重要——看到 Kodi 风格的代码就知道去 Kodi 源码找参考，看到 KrKr2 风格的代码就知道是定制逻辑。

### 保留的 Kodi 特征

| 特征 | 说明 | 源码位置 | 为什么保留 |
|------|------|----------|-----------|
| `DVD_` 前缀命名 | DVD 时代的命名约定，如 `DVD_NOPTS_VALUE`（表示"无有效时间戳"的特殊值，值为 `0`）、`DVD_TIME_BASE`（时间基准，值为 `1000000.0` 微秒） | `Clock.h` | 修改命名会导致大量代码变动，收益不大 |
| `CDVDMsg` 消息体系 | 线程间通信的消息类型层次结构 | `MessageQueue.cpp` | 整个多线程架构的基石，无法轻易替换 |
| `CThread` 线程基类 | 所有工作线程继承此基类，提供 `Create()`/`StopThread()` 等方法 | BasePlayer / VideoPlayerVideo / Audio | 线程管理的统一接口 |
| 缓存状态机 | 5 个状态：FLUSH → FULL → INIT → PLAY → DONE | `VideoPlayer.cpp` HandlePlaySpeed() | 确保播放流畅的核心机制 |
| 老版 FFmpeg API | `avcodec_decode_video2()` / `avcodec_decode_audio4()` | `VideoCodecFFmpeg.cpp` / `AudioCodecFFmpeg.cpp` | 移植时的 FFmpeg 版本较老（3.3.9），之后未更新 |
| `OMXClock` 空桩 | 树莓派硬件时钟接口 | `OMXClock.h` | 保留空实现以免修改大量 `#include` |

> **有趣的对比**：KrKr2 的音频模块（`cpp/core/sound/`）中的 FFWaveDecoder 使用的是**新版** FFmpeg API（`avcodec_send_packet()` / `avcodec_receive_frame()`），而视频播放器模块用的是**旧版** API。这说明两个模块是不同时期编写或移植的。

### 裁剪的 Kodi 功能

| 被裁剪的功能 | Kodi 中的用途 | 裁剪原因 |
|-------------|-------------|---------|
| 字幕渲染（SubtitleRenderer） | 加载 SRT/ASS 字幕并渲染叠加 | 视觉小说不使用外挂字幕 |
| 网络流（RTSP/HLS/DASH） | 播放网络视频流 | KrKr2 只需播放本地文件 |
| 蓝光/DVD 导航 | 蓝光菜单、DVD 章节跳转 | 不适用于视觉小说场景 |
| 多音轨/多字幕切换 | 用户在界面上选择不同语言音轨 | 简化为单音轨播放 |
| 播放列表管理 | 连续播放多个文件 | 视觉小说中一次只播一个视频 |
| 复杂的 GUI 交互 | OSD（On-Screen Display，屏幕叠加显示，如进度条、音量条）控制 | KrKr2 有自己的 UI 系统 |

### KrKr2 新增的定制代码

只有两个文件是 KrKr2 全新编写的：

```
KRMoviePlayer.cpp/h  ← 全新：桥接 Kodi 播放器与 KrKr2 引擎
KRMovieLayer.cpp/h   ← 全新：Cocos2d-x 图层渲染
```

**核心设计**：TVPMoviePlayer 类通过**双重继承**实现桥接——

```cpp
// 源码：KRMoviePlayer.h（简化）
class TVPMoviePlayer : public iTVPVideoOverlay,  // KrKr2 侧接口
                       public CBaseRenderer       // Kodi 侧接口
{
    // KrKr2 侧：被 TJS2 脚本通过 iTVPVideoOverlay 接口调用
    //   Open(), Play(), Stop(), SetPosition() 等
    
    // Kodi 侧：被 CRenderManager 作为渲染器回调
    //   AddVideoPicture(), WaitForBuffer(), FlipPage() 等
};
```

这种设计让两个系统无需了解对方的内部实现——Kodi 的播放管线只知道它在向一个 CBaseRenderer 输出帧，KrKr2 引擎只知道它在通过 iTVPVideoOverlay 控制一个视频叠加层。

---

## CDVDMsg 消息系统：线程通信的核心

整个播放器的多线程协调依赖 CDVDMsg 消息系统。不理解它，就看不懂任何线程间的交互代码。

### 什么是 CDVDMsg？

CDVDMsg（DVD Message 的缩写）是一个消息基类，每种线程间通信操作对应一个消息子类。所有消息通过 CDVDMessageQueue（一个线程安全的优先级队列）在线程间传递。

这种设计模式叫做**消息传递并发模型**（Message Passing Concurrency）——线程之间不共享可变状态，而是通过发送消息来协调工作，从根本上避免了数据竞争（data race，即多个线程同时读写同一块内存导致的不确定行为）。

### 消息类型一览

```cpp
// 源码摘录：MessageQueue.cpp 中定义的消息类型

// ═══ 通用消息（所有工作线程都能处理）═══
GENERAL_SYNCHRONIZE    // 同步屏障：等所有线程都到达这个点后再继续
GENERAL_RESYNC         // 重新同步：Seek 后通知各线程以新时间点重新开始
GENERAL_FLUSH          // 清空缓冲区：丢弃所有已缓冲的数据
GENERAL_RESET          // 重置解码器状态（比 FLUSH 更彻底）
GENERAL_STREAMCHANGE   // 流切换：音轨或视频轨发生变化
GENERAL_EOF            // 文件结束：通知该流已没有更多数据

// ═══ 播放器控制消息 ═══
PLAYER_SETSPEED        // 设置播放速度（如 2x 快进、0.5x 慢放）
PLAYER_STARTED         // 通知主控：某个子线程已开始播放（解码到了足够的数据）
PLAYER_AVCHANGE        // 音视频参数变化（如分辨率改变）
PLAYER_SEEK            // 跳转到指定时间点

// ═══ 数据消息 ═══
DEMUXER_PACKET         // 携带一个 AVPacket（压缩数据包），从解封装层传给解码层
```

### 消息优先级机制

CDVDMessageQueue 支持两级优先级——高优先级消息**总是**先于普通消息被处理：

```cpp
// 源码：MessageQueue.cpp（简化后的核心逻辑）

void CDVDMessageQueue::Put(CDVDMsg* pMsg, int priority) {
    std::unique_lock<std::mutex> lock(m_section);  // 加互斥锁，保证线程安全
    
    if (priority > 0) {
        // 高优先级消息：插入队列头部，下次 Get() 时最先被取出
        m_messages.push_front({pMsg, priority});
    } else {
        // 普通优先级消息：插入队列尾部，按先进先出（FIFO）顺序处理
        m_messages.push_back({pMsg, priority});
    }
    
    // 唤醒正在等待消息的消费者线程
    // （如果没有线程在等待，这个通知会被忽略，不会丢失消息）
    m_event.notify_one();
}

MsgQueueReturnCode CDVDMessageQueue::Get(
    CDVDMsg** pMsg,
    unsigned int iTimeoutInMilliSeconds)
{
    std::unique_lock<std::mutex> lock(m_section);
    
    // 第 1 步：扫描队列，优先取出高优先级消息
    for (auto it = m_messages.begin(); it != m_messages.end(); ++it) {
        if (it->priority > 0) {
            *pMsg = it->pMsg;
            m_messages.erase(it);
            return MSGQ_OK;  // 找到高优先级消息，立即返回
        }
    }
    
    // 第 2 步：没有高优先级消息时，取队首的普通消息
    if (!m_messages.empty()) {
        *pMsg = m_messages.front().pMsg;
        m_messages.pop_front();
        return MSGQ_OK;
    }
    
    // 第 3 步：队列为空，阻塞等待直到超时
    if (iTimeoutInMilliSeconds > 0) {
        m_event.wait_for(lock,
            std::chrono::milliseconds(iTimeoutInMilliSeconds));
    }
    
    return MSGQ_TIMEOUT;  // 超时后返回，让调用者决定下一步
}
```

### 为什么需要优先级？

考虑一个实际场景——用户在播放过程中按下 Seek（跳转）按钮：

```
时间线：
  ──────────────────────────────────────────→
  [普通包][普通包][普通包] ... [普通包]  ← 队列中已积压了 40MB 的数据包
                                           
  用户按下 Seek 跳转到 01:30:00 位置

  如果 Seek 消息以普通优先级插入队尾：
  → 需要等队列中所有包处理完才能执行 Seek
  → 延迟可能达到数秒，用户体验极差

  实际做法：GENERAL_FLUSH 和 GENERAL_RESYNC 以高优先级（priority=1）发送
  → 立即被取出执行
  → 清空队列中的旧数据包
  → 从新位置开始读取
  → Seek 响应时间 < 100ms
```

**设计规则总结**：
- 控制消息（FLUSH / RESYNC / SETSPEED / SEEK）→ **高优先级**（priority=1），立即响应
- 数据消息（DEMUXER_PACKET）→ **普通优先级**（priority=0），按顺序处理

### 消息流转全景图

下面这张图展示了一次完整播放过程中，消息如何在各线程间流转：

```
BasePlayer::Process() 主循环                    （线程 A）
    │
    │  ① ReadPacket()
    │     CDVDDemuxFFmpeg::Read()
    │     └── 返回 AVPacket（压缩视频/音频数据）
    │
    │  ② ProcessPacket() — 根据包的 stream_id 分发
    │     │
    │     ├── 视频包 ─→ CDVDMsgDemuxerPacket ─→ m_VideoPlayerVideo.m_messageQueue
    │     │                                         │
    │     │                                         │  （线程 B）
    │     │                                         ▼
    │     │                              CVideoPlayerVideo::Process()
    │     │                                  ├── Get() 取出消息
    │     │                                  ├── Decode() 调用 FFmpeg 解码
    │     │                                  └── OutputPicture()
    │     │                                       └── CRenderManager::AddVideoPicture()
    │     │                                            └── TVPMoviePlayer::AddVideoPicture()
    │     │                                                 └── 写入 YUV 环形缓冲区
    │     │
    │     └── 音频包 ─→ CDVDMsgDemuxerPacket ─→ m_VideoPlayerAudio.m_messageQueue
    │                                               │
    │                                               │  （线程 C）
    │                                               ▼
    │                                    CVideoPlayerAudio::Process()
    │                                        ├── Get() 取出消息
    │                                        ├── Decode() 调用 FFmpeg 解码
    │                                        └── OutputPacket()
    │                                             └── 送入音频引擎（IAE）播放
    │
    │  ③ HandleMessages() — 处理控制消息
    │     ├── PLAYER_SEEK → FlushBuffers()
    │     │                  ├── 向视频线程发送 GENERAL_RESET（高优先级）
    │     │                  ├── 向音频线程发送 GENERAL_RESET（高优先级）
    │     │                  ├── 向两者发送 GENERAL_FLUSH（高优先级）
    │     │                  └── 向两者发送 GENERAL_SYNCHRONIZE（高优先级）
    │     │
    │     └── PLAYER_SETSPEED → m_clock.SetSpeed(newSpeed)
    │
    │  ④ HandlePlaySpeed() — 管理缓存状态与同步
    │     ├── 监控缓存水位，驱动状态机转换
    │     └── 检测音视频同步启动（SYNC_WAITSYNC → SYNC_INSYNC）
    │
    └── 循环 ①②③④

Cocos2d-x 渲染循环                              （主线程 / 渲染线程）
    │
    └── VideoPresentOverlay::PresentPicture(dt)
        ├── 累加时间 m_curpts += dt
        ├── 比较 m_curpts 与环形缓冲区中帧的 PTS
        ├── 到时间了 → 创建 TVPYUVSprite → 缩放绘制到屏幕
        └── 还没到 → 跳过本帧，等下一次调用
```

---

## 与第 6 章播放器的对比

读到这里，你可能想知道：KrKr2 的这套架构和我们在第 6 章写的 SDL2 播放器有什么本质区别？

| 对比维度 | 第 6 章 SDL2 播放器 | KrKr2 Kodi 架构播放器 |
|---------|-------------------|---------------------|
| 代码量 | ~500 行 | ~7000 行（含所有 .cpp） |
| 线程模型 | 1 解封装线程 + 音频回调 + 主线程 | 1 主控线程 + 1 视频解码线程 + 1 音频解码线程 + 渲染线程 |
| 线程通信 | 共享变量 + 互斥锁 | CDVDMsg 消息队列（消息传递模型） |
| 缓存管理 | 无预缓冲，直接播放 | 五状态缓存状态机（FLUSH→FULL→INIT→PLAY→DONE） |
| 丢帧策略 | 简单跳过 | 基于 CDroppingStats 统计的智能丢帧 |
| Seek 处理 | 基础 seek + 简单 flush | 多阶段 flush（RESET→FLUSH→SYNCHRONIZE）+ 重新同步 |
| 时钟精度 | SDL 毫秒定时器 | CDVDClock 微秒精度 + VSync 校正 |
| 渲染输出 | SDL2 纹理（直接控制窗口） | YUV 环形缓冲区 → Cocos2d-x 纹理（嵌入游戏引擎） |
| 播放速度 | 不支持 | 支持（通过 CDVDClock::SetSpeed()） |
| 循环播放 | 不支持 | 支持（m_iLoopSegmentBegin 段循环） |

> **核心差距在于"鲁棒性"**：第 6 章的播放器在理想条件下工作正常，但遇到 IO 抖动、格式异常、用户快速操作（连续 seek、快进切换）等边界情况就会出问题。Kodi 架构的播放器通过消息队列解耦、缓存状态机保护、多阶段同步等机制，将这些边界情况都妥善处理了。

---

## 常见错误与排查

### 错误 1：搞混"谁在哪个线程运行"

```
错误理解：BasePlayer::ReadPacket() 和 CVideoPlayerVideo::Process() 在同一个线程
正确理解：
  - BasePlayer::Process()       → 线程 A（主控线程）
  - CVideoPlayerVideo::Process() → 线程 B（视频解码线程）
  - CVideoPlayerAudio::Process() → 线程 C（音频解码线程）
  
  它们之间唯一的通信渠道是 CDVDMessageQueue 消息队列。
  BasePlayer 通过 Put() 放入数据包，解码线程通过 Get() 取出。
```

**排查方法**：在源码中看到 `CThread` 的子类，就知道它是一个独立线程。看到 `m_messageQueue.Put()` / `m_messageQueue.Get()`，就知道这是跨线程通信。

### 错误 2：以为 CRenderManager 管理帧缓冲队列

```
错误理解：CRenderManager 像 Kodi 原版一样用 free/queued/discard 三个双端队列管理帧
正确理解：在 KrKr2 中，CRenderManager::AddVideoPicture() 和 WaitForBuffer()
         直接委托给了 TVPMoviePlayer（即 m_pRenderer）。
         
  // VideoRenderer.cpp 第 999 行
  void CRenderManager::AddVideoPicture(const VideoPicture& pic, ...) {
      m_pRenderer->AddVideoPicture(pic, 0);  // 直接委托，不经过自己的队列
  }
  
  // VideoRenderer.cpp 第 1086-1088 行
  int CRenderManager::WaitForBuffer(...) {
      return m_pRenderer->WaitForBuffer(...);  // 同样直接委托
  }
  
  TVPMoviePlayer 用自己的 YUV 环形缓冲区（m_curPicture / m_usedPicture 双索引）
  管理帧数据，而非 CRenderManager 的 free/queued/discard 队列。
```

### 错误 3：看到 DVD_ 前缀就以为是 DVD 相关功能

```
错误理解：DVD_TIME_BASE 是用于 DVD 光盘播放的时间基准
正确理解：DVD_ 前缀只是 Kodi 早期命名遗留，与 DVD 光盘无关。
  - DVD_TIME_BASE = 1000000.0，即以微秒为单位的时间基准
  - DVD_NOPTS_VALUE = 0，表示"此时间戳无效"
  - 这些常量在所有视频格式（MP4/MKV/AVI 等）的播放中通用
```

---

## 动手实践

### 实践 1：跟踪一帧视频的完整旅程

打开源码，沿着以下路径跟踪一帧视频数据从文件到屏幕的完整旅程：

1. **起点** — `VideoPlayer.cpp` 第 603 行 `ReadPacket()`：
   - 调用 `m_CurrentDemuxer->Read()`，返回一个 `DemuxPacket`
   - 这个包里有压缩的视频数据（比如 H.264 的 NAL 单元）

2. **分发** — `VideoPlayer.cpp` 第 638 行 `ProcessPacket()`：
   - 根据 `pPacket->iStreamId` 判断是视频包还是音频包
   - 视频包被包装成 `CDVDMsgDemuxerPacket`，通过 `m_VideoPlayerVideo.SendMessage()` 放入消息队列

3. **解码** — `VideoPlayerVideo.cpp` 第 353 行：
   - 从消息队列 `Get()` 取出 `CDVDMsgDemuxerPacket`
   - 调用 `m_pVideoCodec->Decode(pData, iSize, dts, pts)` 进行解码
   - 返回值是位掩码：`VC_PICTURE`（有解码好的帧）、`VC_BUFFER`（需要更多数据）、`VC_ERROR`（解码出错）

4. **输出** — `VideoPlayerVideo.cpp` 第 714 行 `OutputPicture()`：
   - 检查是否需要丢帧（CalcDropRequirement）
   - 调用 `CRenderManager::AddVideoPicture()`
   - 最终到达 `TVPMoviePlayer::AddVideoPicture()`

5. **缓冲** — `KRMoviePlayer.cpp` 第 176 行 `AddVideoPicture()`：
   - 只接受 YUV420P 格式
   - 将 Y/U/V 三个平面数据拷贝到环形缓冲区的当前槽位
   - 使用 `TJSAlignedAlloc(size, 4)` 分配 4 字节对齐的内存
   - 推进 `m_curPicture` 索引

6. **终点** — `KRMoviePlayer.cpp` 第 239 行 `PresentPicture(dt)`：
   - 累加时间 `m_curpts += dt`
   - 当 `m_curpts >= 当前帧PTS` 时，创建 `TVPYUVSprite` 纹理
   - 缩放到 `GetBounds()` 矩形区域并绘制到屏幕

### 实践 2：统计消息类型使用频率

在 `VideoPlayer.cpp` 的 `HandleMessages()` 函数中（第 1210-1669 行），找到所有 `CDVDMsg::Is()` 的调用，列出该函数处理了哪些消息类型。提示：搜索 `pMsg->Is(` 或 `IsInited(CDVDMsg::` 模式。

---

## 对照项目源码

本节涉及的所有源码文件及关键位置：

- `cpp/core/movie/ffmpeg/VideoPlayer.cpp` — BasePlayer 主控逻辑
  - 第 273-343 行：`OpenFromStream()` 播放器初始化入口
  - 第 538-741 行：`Process()` 主循环
  - 第 1210-1669 行：`HandleMessages()` 消息处理
  - 第 1865-2183 行：`HandlePlaySpeed()` 缓存状态机 + 同步
- `cpp/core/movie/ffmpeg/KRMoviePlayer.cpp` — TVPMoviePlayer 桥接层
  - 第 176-228 行：`AddVideoPicture()` YUV 环形缓冲写入
  - 第 239-295 行：`PresentPicture()` 渲染输出
- `cpp/core/movie/ffmpeg/VideoRenderer.cpp` — CRenderManager
  - 第 999 行：`AddVideoPicture()` 委托给 TVPMoviePlayer
  - 第 1086-1088 行：`WaitForBuffer()` 委托给 TVPMoviePlayer
- `cpp/core/movie/ffmpeg/MessageQueue.cpp` — CDVDMessageQueue 实现
- `cpp/core/movie/ffmpeg/Clock.cpp` — CDVDClock 时钟
- `cpp/core/movie/ffmpeg/KRMoviePlayer.h` — TVPMoviePlayer 类定义（双重继承）

---

## 本节小结

- KrKr2 的视频播放器**不是从头编写的**，而是移植自 Kodi（原 XBMC）播放器内核，这是一个经过十多年生产验证的成熟架构
- 源码分为**四层**：入口层（TVPMoviePlayer）→ 控制层（BasePlayer）→ 数据处理层（Demux + Decode + Clock）→ 渲染管理层（CRenderManager）
- **TVPMoviePlayer 通过双重继承**（iTVPVideoOverlay + CBaseRenderer）桥接 KrKr2 引擎和 Kodi 播放管线
- **CDVDMsg 消息系统**是线程间通信的唯一渠道，采用消息传递并发模型避免数据竞争
- 消息队列支持**两级优先级**——控制消息（Seek/Flush）高优先级立即处理，数据消息（AVPacket）按顺序处理
- 识别 Kodi 遗产代码的标志：`DVD_` 前缀命名、`CDVDMsg` 消息类型、`CThread` 基类
- 在 KrKr2 实现中，CRenderManager 的帧队列被**旁路**，TVPMoviePlayer 直接管理 YUV 环形缓冲区

---

## 练习题与答案

### 题目 1：画出 Seek 操作的消息流转图

当用户在播放过程中跳转到视频的 1:30:00 位置时，BasePlayer 需要通知所有子线程清空旧数据并从新位置开始。请画出从 `HandleMessages()` 收到 `PLAYER_SEEK` 消息开始，到所有线程准备好从新位置播放为止的完整消息流转图，包括消息类型和优先级。

<details>
<summary>查看答案</summary>

```
HandleMessages() 收到 PLAYER_SEEK 消息
    │
    ├── 调用 FlushBuffers(true)  // DVD_MSGT_SEEK 触发
    │   │
    │   ├── 向 CVideoPlayerVideo 发送（全部高优先级 priority=1）：
    │   │   ├── GENERAL_RESET        ← 重置解码器状态
    │   │   ├── GENERAL_FLUSH        ← 清空消息队列中的旧数据包
    │   │   └── GENERAL_SYNCHRONIZE  ← 同步屏障，等待线程处理完
    │   │
    │   └── 向 CVideoPlayerAudio 发送（全部高优先级 priority=1）：
    │       ├── GENERAL_RESET
    │       ├── GENERAL_FLUSH
    │       └── GENERAL_SYNCHRONIZE
    │
    ├── 调用 CDVDDemuxFFmpeg::SeekTime(90*60*1000)  // Seek 到 1:30:00
    │   └── 内部调用 av_seek_frame() 或 avformat_seek_file()
    │
    ├── 设置缓存状态 → CACHESTATE_FLUSH
    │   └── HandlePlaySpeed() 会将其转为 CACHESTATE_FULL 或 CACHESTATE_INIT
    │
    └── 恢复正常读包循环
        └── ReadPacket() 从新位置读取数据包

各子线程收到高优先级消息后的处理：
  CVideoPlayerVideo::Process():
    ├── GENERAL_RESET    → 调用 m_pVideoCodec->Reset()，清除解码器内部缓存
    ├── GENERAL_FLUSH    → 丢弃消息队列中所有剩余的 DEMUXER_PACKET
    └── GENERAL_SYNCHRONIZE → 向 BasePlayer 发回确认

  CVideoPlayerAudio::Process():
    ├── GENERAL_RESET    → 调用 m_pAudioCodec->Reset()
    ├── GENERAL_FLUSH    → 丢弃消息队列中所有剩余的 DEMUXER_PACKET
    └── GENERAL_SYNCHRONIZE → 向 BasePlayer 发回确认

同步恢复流程（HandlePlaySpeed 中）：
  ├── 两个子线程各自发送 PLAYER_STARTED（syncState = SYNC_WAITSYNC）
  ├── HandlePlaySpeed() 检测到两者都到达 SYNC_WAITSYNC
  ├── 计算公共时钟点，调用 m_clock.Discontinuity(clock)
  ├── 向两者发送 GENERAL_RESYNC（携带新时钟值）
  └── 两者进入 SYNC_INSYNC，正常播放恢复
```

**关键理解**：
- 所有控制消息以**高优先级**发送，确保立即被处理（在积压的数据包之前）
- `GENERAL_SYNCHRONIZE` 是一个"屏障"消息——BasePlayer 等待它被处理完才继续
- Seek 后需要经历完整的同步启动流程（SYNC_WAITSYNC → SYNC_INSYNC），与首次开始播放时相同

</details>

### 题目 2：为什么 TVPMoviePlayer 要用双重继承？

解释 TVPMoviePlayer 同时继承 `iTVPVideoOverlay` 和 `CBaseRenderer` 的设计原因。如果只用其中一个基类，会带来什么问题？能否用组合（composition）替代继承来实现同样的功能？

<details>
<summary>查看答案</summary>

**双重继承的原因：**

TVPMoviePlayer 需要同时满足两个系统的接口要求：
1. **KrKr2 引擎侧**：通过 `iTVPVideoOverlay` 接口，TJS2 脚本和引擎核心可以调用 `Open()`、`Play()`、`Stop()` 等方法控制播放
2. **Kodi 播放管线侧**：CRenderManager 需要一个 `CBaseRenderer` 对象来输出解码后的帧数据，它会调用 `AddVideoPicture()`、`WaitForBuffer()` 等方法

**如果只继承一个基类：**
- 只继承 `iTVPVideoOverlay`：Kodi 的 CRenderManager 无法将 TVPMoviePlayer 作为渲染器使用，需要额外的适配器类
- 只继承 `CBaseRenderer`：KrKr2 引擎无法通过标准的视频叠加接口控制播放器，需要修改引擎核心代码

**能否用组合替代？**

可以，但会增加复杂度：

```cpp
// 组合方案（理论上可行但更复杂）
class TVPMoviePlayer : public iTVPVideoOverlay {
    class RendererAdapter : public CBaseRenderer {
        TVPMoviePlayer* m_owner;
        void AddVideoPicture(...) override {
            m_owner->doAddVideoPicture(...);  // 转发给外层类
        }
    };
    RendererAdapter m_renderer;  // 持有适配器实例
};
```

组合方案需要额外的适配器类和转发调用，而双重继承直接让一个对象同时满足两个接口，代码更简洁。在这种"一个对象确实同时扮演两个角色"的场景下，双重继承是合理的设计选择。

</details>

### 题目 3：指出旧版 FFmpeg API 的问题

KrKr2 视频播放器使用 `avcodec_decode_video2()` 而非新版的 `avcodec_send_packet()` / `avcodec_receive_frame()` API。请解释：(a) 这两套 API 的核心区别是什么？ (b) 使用旧版 API 会带来什么实际问题？ (c) 如果要升级到新版 API，`CVideoPlayerVideo::Process()` 中的解码循环需要做哪些改动？

<details>
<summary>查看答案</summary>

**(a) 核心区别：**

旧版 API（同步，一进一出）：
```cpp
// 输入一个包，立即输出一帧（或不输出）
int got_frame = 0;
int ret = avcodec_decode_video2(ctx, frame, &got_frame, &packet);
// got_frame != 0 表示解码出了一帧
```

新版 API（异步，解耦输入和输出）：
```cpp
// 第 1 步：送入一个包（可能不会立即产出帧）
avcodec_send_packet(ctx, &packet);

// 第 2 步：尝试取出帧（可能取出 0 帧、1 帧或多帧）
while (avcodec_receive_frame(ctx, frame) == 0) {
    // 处理 frame
}
```

新版 API 将输入和输出解耦，支持一个 packet 解码出多帧（B 帧重排序时常见），也支持先送多个 packet 再批量取帧。

**(b) 实际问题：**

1. `avcodec_decode_video2()` 在 FFmpeg 3.1 后被标记为 deprecated（已弃用），新版 FFmpeg（≥4.0）中可能被移除
2. 某些编解码器（如 H.265/HEVC 的某些配置）在旧 API 下行为不正确
3. 无法利用解码器内部的流水线优化——新 API 允许解码器批量处理，提高吞吐量
4. KrKr2 使用固定的 FFmpeg 3.3.9 版本（通过 vcpkg overlay port 锁定），避免了兼容性问题，但也无法享受新版本的性能和格式支持改进

**(c) 升级改动：**

`CVideoPlayerVideo::Process()` 中目前的解码循环（`VideoPlayerVideo.cpp` 第 353 行附近）：
```cpp
// 当前：旧 API
int iDecoderState = m_pVideoCodec->Decode(pData, iSize, dts, pts);
if (iDecoderState & VC_PICTURE) {
    OutputPicture();
}
```

需要改为：
```cpp
// 升级后：新 API
m_pVideoCodec->SendPacket(pData, iSize, dts, pts);
while (m_pVideoCodec->ReceiveFrame() == VC_PICTURE) {
    OutputPicture();  // 可能一个包产出多帧
}
```

主要改动点：
1. `CDVDVideoCodecFFmpeg::Decode()` 拆分为 `SendPacket()` + `ReceiveFrame()` 两个方法
2. `Process()` 中的解码结果处理从 `if` 改为 `while` 循环（一个包可能产出多帧）
3. 需要处理 `EAGAIN`（解码器需要更多输入）和 `EOF`（冲刷完成）两个新返回值
4. 冲刷（flush）逻辑：`SendPacket(NULL)` 后循环 `ReceiveFrame()` 直到返回 `EOF`

</details>

---

## 下一步

[BasePlayer 主循环与视频解码](./02-BasePlayer主循环与视频解码.md) — 深入 BasePlayer::Process() 主循环的每一步：ReadPacket → ProcessPacket → HandleMessages → HandlePlaySpeed 四阶段循环，以及 CVideoPlayerVideo 视频解码线程的 OutputPicture、CalcDropRequirement 丢帧判定逻辑。
