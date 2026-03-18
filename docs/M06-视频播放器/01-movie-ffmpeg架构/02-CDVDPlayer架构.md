# 1.2 CDVDPlayer 架构解析

## 本节目标

深入分析 BasePlayer（即 Kodi 中的 CDVDPlayer）的内部架构。理解播放器的状态机设计、消息处理循环、数据流管线以及关键的生命周期方法，掌握从打开文件到播放完成的完整流程。

---

## 1. BasePlayer 类概述

BasePlayer 是 KrKr2 视频播放器的核心类，对应 Kodi 中的 `CDVDPlayer`（后更名为 `CVideoPlayer`）。它继承自 `CThread`，在独立线程中运行主播放循环，负责协调解封装、分发数据包、管理播放状态。

BasePlayer 是整个视频播放系统的"指挥中心"。它不直接执行解码或渲染操作，而是创建和管理各个子组件，通过消息队列与它们通信。这种设计使得各个组件可以在独立的线程中并行运行，充分利用多核 CPU 的能力。

BasePlayer 类的设计遵循了一个经典的多媒体架构模式：将控制逻辑集中在一个主线程中，将计算密集的解码和渲染工作分配到独立的工作线程中。

### 1.1 类继承关系

```cpp
// 源码: movie/ffmpeg/VideoPlayer.h
// BasePlayer 的继承层次

class BasePlayer : public CThread {
    // CThread 提供线程创建/销毁/消息循环能力
    // BasePlayer 重写 Process() 方法作为线程入口
    
protected:
    // 解封装器
    CDVDDemuxFFmpeg* m_pDemuxer;
    
    // 子播放器（音频/视频解码线程）
    CVideoPlayerVideo* m_VideoPlayerVideo;
    CVideoPlayerAudio* m_VideoPlayerAudio;
    
    // 主时钟
    CDVDClock m_clock;
    
    // 消息队列
    CDVDMessageQueue m_messenger;
    
    // 当前活跃的音视频流
    CCurrentStream m_CurrentVideo;
    CCurrentStream m_CurrentAudio;
    
    // 播放器状态
    SPlayerState m_State;
    
    // 输入流（IStream COM 适配器）
    InputStream* m_pInputStream;
    
    // 渲染管理器
    CRenderManager* m_renderManager;
};
```

### 1.2 关键成员结构

```cpp
// 源码: movie/ffmpeg/VideoPlayer.h
// CCurrentStream — 当前活跃流的状态跟踪

struct CCurrentStream {
    int id;                    // 流索引（-1 表示未选择）
    int demuxerId;             // 解封装器中的流 ID
    int source;                // 数据源标识
    double dts;                // 最近一帧的解码时间戳
    double dur;                // 最近一帧的持续时间
    
    CDVDStreamInfo hint;       // 流的编解码参数
    bool started;              // 流是否已开始播放
    bool inited;               // 解码器是否已初始化
    bool startpts;             // 起始 PTS 是否已确定
    double lastdts;            // 上一帧 DTS（用于增量计算）
};

// SPlayerState — 播放器全局状态
struct SPlayerState {
    double time;               // 当前播放时间（秒）
    double time_total;         // 总时长（秒）
    double time_offset;        // 时间偏移
    float speed;               // 播放速度（1.0 = 正常）
    bool canseek;              // 是否支持 Seek
    bool canpause;             // 是否支持暂停
};
```

---

## 2. 播放器生命周期

### 2.1 生命周期状态图

```
 ┌──────────┐   Open()    ┌──────────┐   Play()   ┌──────────┐
 │  Created  │──────────→│  Opening  │─────────→│  Playing  │
 └──────────┘            └──────────┘           └────┬─────┘
                                                     │
                              ┌───────────────────────┤
                              │                       │
                    Pause() ┌─▼──────┐    Seek()  ┌──▼──────┐
                            │ Paused │            │ Seeking │
                            └─┬──────┘            └──┬──────┘
                              │                       │
                    Play()    │                       │ 完成
                              ▼                       ▼
                         ┌──────────┐            ┌──────────┐
                         │  Playing │            │  Playing  │
                         └──────────┘            └──────────┘
                              │
                         EOF  │
                              ▼
                         ┌──────────┐   Close()  ┌──────────┐
                         │   Ended  │──────────→│  Closed   │
                         └──────────┘            └──────────┘
```

### 2.2 打开文件流程（OpenFromStream）

```cpp
// 源码: movie/ffmpeg/VideoPlayer.cpp
// OpenFromStream() — 打开视频文件的完整流程（简化版）

bool BasePlayer::OpenFromStream(IStream* pStream, 
                                 const tjs_char* name,
                                 uint64_t size) {
    // 步骤 1：创建输入流适配器
    // 将 IStream COM 接口包装为 FFmpeg 可用的 InputStream
    m_pInputStream = new InputStream(pStream, size);
    
    // 步骤 2：打开解封装器
    // CDVDDemuxFFmpeg::Open() 内部调用 avformat_open_input()
    if (!OpenDemuxStream()) {
        // 常见失败原因：
        // - 文件格式不支持（不是有效的多媒体容器）
        // - 数据损坏（文件头无法解析）
        return false;
    }
    
    // 步骤 3：创建子播放器
    // 根据检测到的音视频流创建解码线程
    CreatePlayers();
    
    // 步骤 4：启动主播放线程
    // CThread::Create() 创建新线程，入口为 Process()
    Create();
    
    return true;
}

// OpenDemuxStream() 的核心逻辑
bool BasePlayer::OpenDemuxStream() {
    m_pDemuxer = new CDVDDemuxFFmpeg();
    
    if (!m_pDemuxer->Open(m_pInputStream)) {
        delete m_pDemuxer;
        m_pDemuxer = nullptr;
        return false;
    }
    
    // 获取流信息并选择默认音视频流
    int nVideoStream = -1;
    int nAudioStream = -1;
    
    for (int i = 0; i < m_pDemuxer->GetNrOfStreams(); i++) {
        CDemuxStream* stream = m_pDemuxer->GetStream(i);
        if (stream->type == STREAM_VIDEO && nVideoStream < 0)
            nVideoStream = i;
        if (stream->type == STREAM_AUDIO && nAudioStream < 0)
            nAudioStream = i;
    }
    
    // 打开选中的流
    if (nVideoStream >= 0)
        OpenStream(nVideoStream, STREAM_VIDEO);
    if (nAudioStream >= 0)
        OpenStream(nAudioStream, STREAM_AUDIO);
    
    return true;
}
```

### 2.3 创建子播放器

```cpp
// 源码: movie/ffmpeg/VideoPlayer.cpp
// CreatePlayers() — 创建音视频解码线程

void BasePlayer::CreatePlayers() {
    // 创建视频解码线程
    // CVideoPlayerVideo 继承自 CThread
    // 它在独立线程中接收视频 DemuxPacket 并解码
    m_VideoPlayerVideo = new CVideoPlayerVideo(
        &m_clock,           // 主时钟引用（用于同步）
        this,               // 回调接口
        m_renderManager     // 渲染管理器（解码帧的输出目标）
    );
    
    // 创建音频解码线程  
    // CVideoPlayerAudio 继承自 CThread
    // 它在独立线程中接收音频 DemuxPacket 并解码
    m_VideoPlayerAudio = new CVideoPlayerAudio(
        &m_clock,           // 主时钟引用（音频同步基准）
        this                // 回调接口
    );
}
```

---

## 3. 主播放循环（Process）

`Process()` 是 BasePlayer 的心脏，在独立线程中无限循环运行，直到收到终止请求。每次循环迭代执行以下步骤：

### 3.1 循环流程图

```
┌──────────────────────────────────────────┐
│            Process() 主循环               │
│                                          │
│  while (!m_bAbortRequest) {              │
│    ┌──────────────────────┐              │
│    │ 1. HandleMessages()  │ ← 处理控制消息│
│    │    Play/Pause/Seek   │              │
│    └──────────┬───────────┘              │
│               │                          │
│    ┌──────────▼───────────┐              │
│    │ 2. HandlePlaySpeed() │ ← 处理速度变化│
│    │    时钟调整           │              │
│    └──────────┬───────────┘              │
│               │                          │
│    ┌──────────▼───────────┐              │
│    │ 3. 检查队列状态       │ ← 队列满则等待│
│    │    > 50% → Sleep     │              │
│    └──────────┬───────────┘              │
│               │                          │
│    ┌──────────▼───────────┐              │
│    │ 4. m_pDemuxer->Read()│ ← 读取数据包 │
│    │    返回 DemuxPacket   │              │
│    └──────────┬───────────┘              │
│               │                          │
│    ┌──────────▼───────────┐              │
│    │ 5. ProcessPacket()   │ ← 分发到解码器│
│    │    Video/Audio queue │              │
│    └──────────┬───────────┘              │
│               │                          │
│    └──────────┘ (循环继续)               │
│  }                                       │
└──────────────────────────────────────────┘
```

### 3.2 核心循环代码

```cpp
// 源码: movie/ffmpeg/VideoPlayer.cpp
// Process() — 主播放循环（实际代码简化版）

void BasePlayer::Process() {
    // 循环直到收到终止信号
    while (!m_bAbortRequest) {
        
        // ── 步骤 1: 处理控制消息 ──
        // 检查消息队列中是否有 Play/Pause/Seek/SetSpeed 等命令
        HandleMessages();
        
        // ── 步骤 2: 处理播放速度 ──
        // 如果速度发生变化，调整时钟频率
        HandlePlaySpeed();
        
        // ── 步骤 3: 流量控制 ──
        // 防止解封装速度远超解码速度导致内存溢出
        if (m_CurrentVideo.id >= 0) {
            bool full = m_VideoPlayerVideo->IsStalled();
            if (full) {
                Sleep(10);  // 队列满，暂停读取
                continue;
            }
        }
        
        // ── 步骤 4: 读取数据包 ──
        DemuxPacket* pPacket = m_pDemuxer->Read();
        
        if (!pPacket) {
            // 到达文件末尾
            // 发送 EOF 消息到子播放器
            if (m_CurrentVideo.id >= 0)
                m_VideoPlayerVideo->SendMessage(
                    new CDVDMsg(CDVDMsg::GENERAL_EOF));
            if (m_CurrentAudio.id >= 0)
                m_VideoPlayerAudio->SendMessage(
                    new CDVDMsg(CDVDMsg::GENERAL_EOF));
            
            // 等待子播放器处理完缓冲区中的剩余数据
            // 然后触发播放结束事件
            WaitForFinish();
            
            // 回到起始位置（用于循环播放）
            m_pDemuxer->SeekTime(0);
            
            // 通知上层播放结束
            FireEvent(KRMovieEvent::Ended);
            break;
        }
        
        // ── 步骤 5: 分发数据包 ──
        ProcessPacket(pPacket);
    }
}
```

### 3.3 数据包分发

```cpp
// 源码: movie/ffmpeg/VideoPlayer.cpp
// ProcessPacket() — 将解封装数据包分发到对应的解码线程

void BasePlayer::ProcessPacket(DemuxPacket* pPacket) {
    int streamId = pPacket->iStreamId;
    
    if (streamId == m_CurrentVideo.id) {
        // 视频数据包 → 发送到视频解码线程
        // 封装为 CDVDMsgDemuxerPacket 消息
        m_VideoPlayerVideo->SendMessage(
            new CDVDMsgDemuxerPacket(pPacket, false)
        );
        
        // 更新当前视频流状态
        m_CurrentVideo.dts = pPacket->dts;
        m_CurrentVideo.dur = pPacket->duration;
        
    } else if (streamId == m_CurrentAudio.id) {
        // 音频数据包 → 发送到音频解码线程
        m_VideoPlayerAudio->SendMessage(
            new CDVDMsgDemuxerPacket(pPacket, false)
        );
        
        // 更新当前音频流状态
        m_CurrentAudio.dts = pPacket->dts;
        m_CurrentAudio.dur = pPacket->duration;
        
    } else {
        // 其他流（字幕等）→ 丢弃
        CDVDDemuxUtils::FreeDemuxPacket(pPacket);
    }
}
```

---

## 4. 消息处理（HandleMessages）

HandleMessages 是 BasePlayer 处理外部控制命令的核心方法。它从消息队列中取出消息并执行对应的操作。

### 4.1 消息处理流程

```cpp
// 源码: movie/ffmpeg/VideoPlayer.cpp
// HandleMessages() — 处理控制消息

void BasePlayer::HandleMessages() {
    CDVDMsg* pMsg;
    
    // 非阻塞方式从队列中取消息
    while (m_messenger.Get(&pMsg, 0) == MSGQ_OK) {
        
        switch (pMsg->GetMessageType()) {
            
        case CDVDMsg::PLAYER_SEEK: {
            // ── Seek 处理 ──
            CDVDMsgPlayerSeek* msg = 
                static_cast<CDVDMsgPlayerSeek*>(pMsg);
            double seekTarget = msg->GetTime();
            
            // 1. 刷新所有缓冲区
            FlushBuffers(false);
            
            // 2. 在解封装器中执行 Seek
            m_pDemuxer->SeekTime(
                static_cast<int>(seekTarget)
            );
            
            // 3. 重置时钟
            m_clock.Discontinuity(seekTarget);
            
            // 4. 通知子播放器刷新
            if (m_VideoPlayerVideo)
                m_VideoPlayerVideo->SendMessage(
                    new CDVDMsg(CDVDMsg::GENERAL_FLUSH));
            if (m_VideoPlayerAudio)
                m_VideoPlayerAudio->SendMessage(
                    new CDVDMsg(CDVDMsg::GENERAL_FLUSH));
            break;
        }
        
        case CDVDMsg::PLAYER_SETSPEED: {
            // ── 速度调整 ──
            CDVDMsgInt* msg = static_cast<CDVDMsgInt*>(pMsg);
            int speed = msg->m_value;
            
            // 更新时钟速度
            m_clock.SetSpeed(speed);
            m_State.speed = speed / DVD_PLAYSPEED_NORMAL;
            break;
        }
        
        case CDVDMsg::GENERAL_PAUSE: {
            // ── 暂停/恢复 ──
            m_clock.SetSpeed(0);  // 暂停时钟
            break;
        }
        
        default:
            break;
        }
        
        pMsg->Release();  // 释放消息对象（引用计数）
    }
}
```

### 4.2 缓冲区刷新

Seek 操作时需要刷新所有缓冲区，丢弃已读取但尚未播放的数据：

```cpp
// 源码: movie/ffmpeg/VideoPlayer.cpp
// FlushBuffers() — 刷新所有数据缓冲区

void BasePlayer::FlushBuffers(bool queued) {
    // 1. 重置当前流状态
    m_CurrentVideo.dts = DVD_NOPTS_VALUE;
    m_CurrentVideo.started = false;
    m_CurrentAudio.dts = DVD_NOPTS_VALUE;
    m_CurrentAudio.started = false;
    
    // 2. 刷新解封装器内部缓冲
    if (m_pDemuxer)
        m_pDemuxer->Flush();
    
    // 3. 如果需要队列刷新，发送 FLUSH 消息到子播放器
    if (queued) {
        if (m_VideoPlayerVideo)
            m_VideoPlayerVideo->SendMessage(
                new CDVDMsg(CDVDMsg::GENERAL_FLUSH));
        if (m_VideoPlayerAudio)
            m_VideoPlayerAudio->SendMessage(
                new CDVDMsg(CDVDMsg::GENERAL_FLUSH));
    }
    
    // 4. 重置渲染管理器
    if (m_renderManager)
        m_renderManager->Flush();
}
```

---

## 5. 数据流管线

### 5.1 完整数据流路径

```
IStream (COM)
    │
    ▼
InputStream (AVIO 适配器)
    │  dvd_file_read() / dvd_file_seek()
    ▼
AVFormatContext (FFmpeg 容器解析)
    │  av_read_frame()
    ▼
DemuxPacket (解封装后的压缩数据包)
    │
    ├─── 视频包 ──→ CDVDMsgDemuxerPacket
    │                    │
    │                    ▼
    │              CVideoPlayerVideo (解码线程)
    │                    │  avcodec_send_packet() / receive_frame()
    │                    ▼
    │              AVFrame (解码后的 YUV 帧)
    │                    │
    │                    ▼
    │              CRenderManager.FlipPage()
    │                    │
    │                    ▼
    │              显示层 (Overlay/Layer)
    │
    └─── 音频包 ──→ CDVDMsgDemuxerPacket
                         │
                         ▼
                   CVideoPlayerAudio (解码线程)
                         │  avcodec_send_packet() / receive_frame()
                         ▼
                   PCM 采样数据
                         │
                         ▼
                   AudioDevice (音频输出)
```

### 5.2 流量控制机制

BasePlayer 通过检查子播放器的队列状态来控制读取速度，防止内存溢出：

```cpp
// 源码: movie/ffmpeg/VideoPlayer.cpp
// 流量控制的关键逻辑

// BasePlayer 检查视频队列是否已满
bool BasePlayer::CheckVideoQueueFull() {
    if (m_CurrentVideo.id < 0)
        return false;  // 没有视频流
    
    // 获取视频消息队列的填充状态
    int level = m_VideoPlayerVideo->GetLevel();
    int maxLevel = m_VideoPlayerVideo->GetMaxLevel();
    
    // 队列超过 50% 容量时暂停读取
    // 这给解码线程留出处理空间
    return (level > maxLevel / 2);
}

// CVideoPlayerVideo 中的队列状态查询
bool CVideoPlayerVideo::IsStalled() {
    // 消息队列中的数据包数量
    return m_messageQueue.GetDataSize() > 
           m_messageQueue.GetMaxDataSize() * 0.5;
}
```

---

## 动手实践

### 练习 1：跟踪一个 Seek 操作的完整路径

在代码中设置断点，跟踪一个 Seek 操作从外部调用到内部执行的完整路径：

```cpp
// 跟踪路径：
// 1. 外部调用
player->SetPosition(5000);  // 跳转到 5 秒

// 2. 发送消息
m_messenger.Put(new CDVDMsgPlayerSeek(5000));

// 3. HandleMessages() 处理
// case CDVDMsg::PLAYER_SEEK:
//   → FlushBuffers()     ← 清空所有缓冲
//   → SeekTime(5000)     ← FFmpeg seek
//   → Discontinuity()    ← 重置时钟
//   → GENERAL_FLUSH      ← 通知子播放器

// 4. 子播放器接收 FLUSH
// CVideoPlayerVideo::Process()
//   → 丢弃解码队列中的所有帧
//   → 重置解码器状态
//   → 等待新的数据包到来
```

### 练习 2：模拟播放速度调整

理解时钟速度与播放速度的关系：

```cpp
// DVD_PLAYSPEED_NORMAL = 1000

// 正常速度
m_clock.SetSpeed(1000);    // 1.0x
// → m_systemUsed = m_systemFrequency * 1000 / 1000

// 2 倍速
m_clock.SetSpeed(2000);    // 2.0x
// → m_systemUsed = m_systemFrequency * 1000 / 2000
// → SystemToPlaying() 返回值增速 2 倍

// 暂停
m_clock.SetSpeed(0);       // 0.0x
// → 时钟停止递增
// → 解码线程等待时钟恢复
```

---

## 对照项目源码

| 本节概念 | 对应源文件 | 关键代码位置 |
|----------|-----------|-------------|
| BasePlayer 类定义 | `VideoPlayer.h` | `class BasePlayer : public CThread` |
| CCurrentStream | `VideoPlayer.h` | `struct CCurrentStream` |
| SPlayerState | `VideoPlayer.h` | `struct SPlayerState` |
| OpenFromStream | `VideoPlayer.cpp` | `BasePlayer::OpenFromStream()` |
| Process 主循环 | `VideoPlayer.cpp` | `BasePlayer::Process()` |
| HandleMessages | `VideoPlayer.cpp` | `BasePlayer::HandleMessages()` |
| FlushBuffers | `VideoPlayer.cpp` | `BasePlayer::FlushBuffers()` |
| ProcessPacket | `VideoPlayer.cpp` | `BasePlayer::ProcessPacket()` |
| CreatePlayers | `VideoPlayer.cpp` | `BasePlayer::CreatePlayers()` |

---

## 本节小结

1. BasePlayer 是视频播放器的核心控制类，在独立线程中运行主播放循环
2. 播放器生命周期经历 Created → Opening → Playing → Ended → Closed 状态
3. `OpenFromStream()` 完成输入流创建、解封装器打开、子播放器创建的初始化流程
4. `Process()` 主循环包含五个步骤：消息处理 → 速度处理 → 流量控制 → 读包 → 分发
5. 消息处理通过 CDVDMessageQueue 实现，支持 Seek、速度调整、暂停等控制命令
6. 数据流从 IStream → InputStream → FFmpeg → DemuxPacket → 子播放器的完整路径

---

## 常见错误及解决方案

### 错误 1：OpenDemuxStream 返回 false

```
BasePlayer::OpenDemuxStream failed: could not open demuxer
```

**原因**：输入文件格式不受支持，或文件数据损坏。

**解决方案**：
1. 检查文件是否是有效的多媒体文件（尝试用 `ffprobe` 工具验证）
2. 确认 FFmpeg 编译时启用了对应格式的解封装器
3. 检查 InputStream 的 Read/Seek 操作是否正确实现

### 错误 2：子播放器创建失败

```
CreatePlayers: no video/audio stream found
```

**原因**：解封装器成功打开文件，但未检测到可用的音视频流。

**解决方案**：
1. 使用 `ffprobe -show_streams <file>` 检查文件中的流信息
2. 确认 FFmpeg 编译时启用了对应的解码器

### 错误 3：Seek 后画面卡住

```
Seek completed but video appears frozen
```

**原因**：Seek 目标位置不在关键帧上，解码器需要从最近的关键帧开始解码才能显示正确画面。

**解决方案**：
1. 这是正常现象——FFmpeg 的 `av_seek_frame()` 默认 Seek 到最近的关键帧
2. Seek 后需要等待解码器输出第一个完整帧
3. 检查 `FlushBuffers()` 是否正确清空了旧数据

---

## 练习题与答案

### 练习 1：为什么 BasePlayer 要在独立线程中运行主循环？

<details>
<summary>查看答案</summary>

1. **避免阻塞 UI 线程**：视频解封装是 I/O 密集操作，如果在 UI 线程中执行，会导致界面卡顿
2. **解耦控制与处理**：主循环需要持续运行以读取数据包和处理消息，如果与渲染帧率绑定，会导致解封装速度受限于屏幕刷新率
3. **并行处理**：BasePlayer 线程负责读取和分发，音视频解码线程并行执行解码，三个线程各司其职

</details>

### 练习 2：ProcessPacket 中丢弃非音视频流数据包的影响是什么？

<details>
<summary>查看答案</summary>

丢弃非音视频流的数据包意味着：
1. **字幕不可用**：即使视频文件中嵌入了字幕流，也不会被处理或显示
2. **元数据丢失**：附加的元数据流（如章节信息）被忽略
3. **内存及时释放**：通过 `CDVDDemuxUtils::FreeDemuxPacket()` 及时释放不需要的数据包，避免内存泄漏

这是 KrKr2 裁剪策略的体现——视觉小说不需要视频文件中的字幕功能。

</details>

### 练习 3：解释流量控制中 50% 阈值的设计考量

<details>
<summary>查看答案</summary>

队列超过 50% 时暂停读取的阈值选择是一个平衡设计：

1. **太低（如 20%）**：解封装频繁暂停和恢复，增加线程切换开销，可能导致解码线程饥饿
2. **50%（当前值）**：给解码线程留出足够的处理空间，同时保持解封装线程的连续性
3. **太高（如 90%）**：解码线程处理完缓冲区需要更长时间，可能导致内存使用峰值过高
4. **等待恢复**：当队列降到 50% 以下时，解封装线程自动恢复读取，形成自适应流量控制

这个自适应机制确保了数据供给与消费之间的平衡，既不会内存溢出，也不会让解码线程空等。

</details>

---

## 下一步

下一节 [1.3 线程模型](03-线程模型.md) 将详细分析视频播放器的三线程架构，包括线程间的消息传递机制、同步策略和生命周期管理。
