# BasePlayer 主循环与视频解码

> **所属模块：** P06-FFmpeg音视频开发
> **前置知识：** [架构总览与 Kodi 遗产](./01-架构总览与Kodi遗产.md)、[音视频同步](../05-音视频同步/01-同步基础与时间戳.md)
> **预计阅读时间：** 50 分钟

## 本节目标

读完本节后，你将能够：
1. 逐步跟踪 `BasePlayer::Process()` 主循环的四个阶段：HandleMessages → HandlePlaySpeed → ReadPacket → ProcessPacket
2. 理解缓存状态机的五个状态（FLUSH → FULL → INIT → PLAY → DONE）及其转换条件
3. 解释音视频同步启动的三阶段流程：SYNC_STARTING → SYNC_WAITSYNC → SYNC_INSYNC
4. 分析 CVideoPlayerVideo 视频解码线程的消息处理优先级机制
5. 理解 OutputPicture 的帧输出流程和 CalcDropRequirement 的智能丢帧算法

---

## BasePlayer::Process() 主循环总览

`BasePlayer::Process()` 是整个播放器的心脏。它运行在独立线程中（继承自 Kodi 的 `CThread` 基类），通过一个无限循环驱动所有子组件协同工作。

**源码位置**：`cpp/core/movie/ffmpeg/VideoPlayer.cpp` 第 538-741 行

### 三阶段生命周期

```cpp
// VideoPlayer.cpp BasePlayer::Process() — 整体结构
void BasePlayer::Process() {
    // ══════════════════════════════════════════════
    // 阶段 1：初始化（只执行一次）
    // ══════════════════════════════════════════════
    
    // OpenFromStream() 在 Process() 之前由 TVPMoviePlayer 调用
    // 它完成了：
    //   1. 打开输入流（CDVDInputStreamFile）
    //   2. 创建解封装器（CDVDDemuxFFmpeg）
    //   3. 创建视频/音频解码线程（CVideoPlayerVideo / CVideoPlayerAudio）
    //   4. 打开默认音视频流（OpenDefaultStreams）
    
    // ══════════════════════════════════════════════
    // 阶段 2：主循环（一直运行直到停止）
    // ══════════════════════════════════════════════
    while (!m_bAbortRequest) {  // m_bAbortRequest 由 Stop() 设置为 true
        
        // 步骤 A：处理控制消息
        HandleMessages();
        
        // 步骤 B：管理缓存状态和播放速度
        HandlePlaySpeed();
        
        // 步骤 C：读取数据包
        DemuxPacket* pkt = ReadPacket();
        
        // 步骤 D：分发数据包到对应的解码线程
        if (pkt) {
            ProcessPacket(pkt);
        }
        
        // 步骤 E：更新播放状态（进度、缓冲百分比等）
        UpdatePlayState();
    }
    
    // ══════════════════════════════════════════════
    // 阶段 3：清理（退出时执行一次）
    // ══════════════════════════════════════════════
    // 通知所有子线程停止
    // 关闭解码器、解封装器、输入流
    // 释放所有资源
}
```

### 四步循环详解

每一轮主循环执行四个步骤，顺序不能打乱——消息处理必须最先（确保 Seek 等控制操作立即生效），缓存管理其次（确保播放状态正确），然后才是读取和分发数据。

```
每一轮循环：

  HandleMessages()     ← 步骤 A：有 Seek 请求？速度变更？
       │                  如果有，立即执行（清空缓冲、重置时钟等）
       ▼
  HandlePlaySpeed()    ← 步骤 B：缓存状态是什么？
       │                  需要预缓冲？需要启动音视频同步？
       ▼
  ReadPacket()         ← 步骤 C：从解封装器读一个 AVPacket
       │                  如果 EOF → 发送 GENERAL_EOF 给子线程
       ▼
  ProcessPacket()      ← 步骤 D：视频包 → 送视频解码线程
                          音频包 → 送音频解码线程
```

---

## 步骤 A：HandleMessages() — 控制消息处理

`HandleMessages()` 处理来自外部（TVPMoviePlayer）和内部（子线程）的控制消息。

**源码位置**：`cpp/core/movie/ffmpeg/VideoPlayer.cpp` 第 1210-1669 行（约 460 行代码）

### 关键消息处理

```cpp
// VideoPlayer.cpp HandleMessages() — 核心消息处理逻辑（简化）
void BasePlayer::HandleMessages() {
    CDVDMsg* pMsg;
    
    // 非阻塞取消息（timeout=0），有消息就处理，没有就跳过
    while (m_messageQueue.Get(&pMsg, 0) == MSGQ_OK) {
        
        // ─── PLAYER_SEEK：跳转请求 ───
        if (pMsg->IsType(CDVDMsg::PLAYER_SEEK)) {
            auto* seekMsg = static_cast<CDVDMsgPlayerSeek*>(pMsg);
            double seekTarget = seekMsg->GetTime();
            
            // 1. 通知所有子线程清空缓冲（多阶段 flush）
            FlushBuffers(true);
            //   FlushBuffers 内部会依次发送：
            //   GENERAL_RESET     → 重置解码器内部状态
            //   GENERAL_FLUSH     → 清空消息队列中的旧数据包
            //   GENERAL_SYNCHRONIZE → 同步屏障，等子线程处理完
            
            // 2. 执行实际的 seek 操作
            m_CurrentDemuxer->SeekTime(seekTarget);
            //   内部调用 av_seek_frame() 或 avformat_seek_file()
            
            // 3. 重置缓存状态 → 重新开始缓冲
            SetCaching(CACHESTATE_FLUSH);
        }
        
        // ─── PLAYER_SETSPEED：播放速度变更 ───
        else if (pMsg->IsType(CDVDMsg::PLAYER_SETSPEED)) {
            int speed = static_cast<CDVDMsgInt*>(pMsg)->m_value;
            
            // 更新主时钟速度
            m_clock.SetSpeed(speed);
            // CDVDClock::SetSpeed() 内部会：
            //   1. 计算新的系统频率 newfreq = m_systemFrequency * 1000 / speed
            //   2. 重新计算 m_startClock 以保证时间连续性
            
            // 通知音视频解码线程
            m_VideoPlayerVideo->SendMessage(
                new CDVDMsgInt(CDVDMsg::PLAYER_SETSPEED, speed));
            m_VideoPlayerAudio->SendMessage(
                new CDVDMsgInt(CDVDMsg::PLAYER_SETSPEED, speed));
        }
        
        // ─── PLAYER_STARTED：子线程报告"已就绪" ───
        else if (pMsg->IsType(CDVDMsg::PLAYER_STARTED)) {
            auto* startMsg = static_cast<CDVDMsgPlayerStarted*>(pMsg);
            
            // 记录该流的同步状态
            if (startMsg->m_player == DVDPLAYER_VIDEO) {
                m_CurrentVideo.syncState = startMsg->m_syncState;
                // syncState 从 SYNC_STARTING 变为 SYNC_WAITSYNC
                // 表示"我已经解码到足够的数据，等待同步信号"
            } else if (startMsg->m_player == DVDPLAYER_AUDIO) {
                m_CurrentAudio.syncState = startMsg->m_syncState;
            }
        }
        
        // ─── GENERAL_SYNCHRONIZE：同步屏障完成 ───
        else if (pMsg->IsType(CDVDMsg::GENERAL_SYNCHRONIZE)) {
            // 子线程处理完同步屏障消息后会发回确认
            // BasePlayer 在这里等待所有线程都确认后才继续
        }
        
        pMsg->Release();  // 释放消息（CDVDMsg 使用引用计数管理内存）
    }
}
```

### FlushBuffers：多阶段缓冲区清空

Seek 操作的核心步骤是清空所有缓冲区，这通过 `FlushBuffers()` 实现。它不是简单地清空队列，而是一个精心设计的三阶段过程：

**源码位置**：`cpp/core/movie/ffmpeg/VideoPlayer.cpp` 第 1091-1208 行

```cpp
// VideoPlayer.cpp FlushBuffers() — 三阶段清空流程
void BasePlayer::FlushBuffers(bool bOnSeek) {
    // ════════════════════════════
    // 阶段 1：GENERAL_RESET
    // ════════════════════════════
    // 重置解码器内部状态（清除解码器内部缓存的参考帧等）
    // 必须在 FLUSH 之前，因为解码器可能持有对队列中数据的引用
    if (m_CurrentVideo.inited) {
        m_VideoPlayerVideo->SendMessage(
            new CDVDMsg(CDVDMsg::GENERAL_RESET), 1);  // priority=1（高优先级）
    }
    if (m_CurrentAudio.inited) {
        m_VideoPlayerAudio->SendMessage(
            new CDVDMsg(CDVDMsg::GENERAL_RESET), 1);
    }
    
    // ════════════════════════════
    // 阶段 2：GENERAL_FLUSH
    // ════════════════════════════
    // 清空消息队列中所有待处理的 DEMUXER_PACKET 消息
    // 这些是旧位置的数据，不再需要
    if (m_CurrentVideo.inited) {
        m_VideoPlayerVideo->SendMessage(
            new CDVDMsg(CDVDMsg::GENERAL_FLUSH), 1);
    }
    if (m_CurrentAudio.inited) {
        m_VideoPlayerAudio->SendMessage(
            new CDVDMsg(CDVDMsg::GENERAL_FLUSH), 1);
    }
    
    // ════════════════════════════
    // 阶段 3：GENERAL_SYNCHRONIZE
    // ════════════════════════════
    // 同步屏障：等待子线程确认已完成 RESET 和 FLUSH
    // 这保证了在 BasePlayer 开始从新位置读取数据前，
    // 所有子线程都已经清空了旧数据
    if (m_CurrentVideo.inited) {
        m_VideoPlayerVideo->SendMessage(
            new CDVDMsg(CDVDMsg::GENERAL_SYNCHRONIZE), 1);
    }
    if (m_CurrentAudio.inited) {
        m_VideoPlayerAudio->SendMessage(
            new CDVDMsg(CDVDMsg::GENERAL_SYNCHRONIZE), 1);
    }
    
    // 重置同步状态 → 需要重新走同步启动流程
    if (bOnSeek) {
        m_CurrentVideo.syncState = SYNC_STARTING;
        m_CurrentAudio.syncState = SYNC_STARTING;
    }
}
```

**三阶段顺序的原因**：

```
为什么不能直接 FLUSH？

假设队列中有：[PacketA] [PacketB] [PacketC]

如果先 FLUSH 再 RESET：
  1. FLUSH 清空队列 → 队列为空
  2. RESET 重置解码器 → 但解码器内部可能还在处理 PacketC
  → 解码器试图访问已被清空的数据 → 崩溃或数据损坏

正确顺序：先 RESET 再 FLUSH：
  1. RESET 重置解码器 → 解码器释放所有内部引用
  2. FLUSH 清空队列 → 安全地丢弃所有旧数据
  3. SYNCHRONIZE → 确认一切完成后再读取新数据

所有三条消息都以高优先级（priority=1）发送，
确保它们在队列中所有旧数据包之前被处理。
```

---

## 步骤 B：HandlePlaySpeed() — 缓存状态机

`HandlePlaySpeed()` 管理缓存状态和音视频同步启动，是主循环中最复杂的步骤。

**源码位置**：`cpp/core/movie/ffmpeg/VideoPlayer.cpp` 第 1865-2183 行（约 320 行代码）

### 五状态缓存状态机

```
                    seek / 开始播放
                          │
                          ▼
                  ┌───────────────┐
                  │ CACHESTATE_   │
                  │    FLUSH      │  seek 后的初始状态
                  └───────┬───────┘
                          │
              ┌───────────┴───────────┐
              │ 首次启动？             │
              │                       │
              ▼                       ▼
     ┌───────────────┐      ┌───────────────┐
     │ CACHESTATE_   │      │ CACHESTATE_   │
     │    INIT       │      │    FULL       │  非首次启动（seek 后）
     │ 首次预缓冲    │      │ 等待缓冲填满  │
     └───────┬───────┘      └───────┬───────┘
              │                       │
              │  缓冲达到阈值          │ 缓冲达到阈值
              │                       │
              ▼                       ▼
            ┌─────────────────────────┐
            │    CACHESTATE_PLAY      │  正常播放状态
            │  时钟正常运行，数据流动  │
            └────────────┬────────────┘
                         │
                         │ 文件读到 EOF
                         ▼
            ┌─────────────────────────┐
            │    CACHESTATE_DONE      │  等待缓冲区中的帧全部播完
            └─────────────────────────┘
```

### 状态转换源码分析

```cpp
// VideoPlayer.cpp HandlePlaySpeed() — 缓存状态机（简化）
// 源码：第 1865-2183 行

void BasePlayer::HandlePlaySpeed() {
    
    // ─── CACHESTATE_FLUSH → FULL 或 INIT ───
    if (m_caching == CACHESTATE_FLUSH) {
        // SetCaching() 根据当前上下文决定下一个状态
        // 如果是首次启动 → CACHESTATE_INIT
        // 如果是 seek 后  → CACHESTATE_FULL
        SetCaching(CACHESTATE_FULL);
        // SetCaching 的内部逻辑：
        //   FLUSH → FULL：设置 m_clock.SetSpeed(DVD_PLAYSPEED_PAUSE)
        //                  暂停时钟，等数据填满
        //   FLUSH → INIT：同上，但使用不同的缓冲阈值
    }
    
    // ─── CACHESTATE_FULL / CACHESTATE_INIT ───
    // 等待缓冲区填充到足够水位
    if (m_caching == CACHESTATE_FULL || m_caching == CACHESTATE_INIT) {
        // 检查缓冲水位
        // INIT 状态使用较低的阈值（更快开始播放）
        // FULL 状态使用较高的阈值（seek 后确保流畅）
        
        // 当缓冲足够时 → 进入 PLAY 状态
        SetCaching(CACHESTATE_PLAY);
        // SetCaching(PLAY) 内部：
        //   恢复时钟速度 m_clock.SetSpeed(m_playSpeed)
        //   开始正常播放
    }
    
    // ─── 音视频同步启动（在 PLAY 状态下）───
    // 这是最精密的部分
    // 源码：第 1991-2094 行
    
    if (m_caching == CACHESTATE_PLAY) {
        
        // 检查视频流是否已就绪
        bool videoReady = (!m_CurrentVideo.inited ||
                          m_CurrentVideo.syncState == SYNC_WAITSYNC);
        // 检查音频流是否已就绪
        bool audioReady = (!m_CurrentAudio.inited ||
                          m_CurrentAudio.syncState == SYNC_WAITSYNC);
        
        // 两者都就绪 → 启动同步
        if (videoReady && audioReady) {
            // 1. 计算公共时钟起点
            //    取音视频两个流当前 PTS 的合理值
            double clock = 0;
            // （具体计算逻辑取决于哪个流先就绪、PTS 是否有效等）
            
            // 2. 设置时钟不连续点
            //    这会重置时钟的起始时间
            m_clock.Discontinuity(clock);
            // CDVDClock::Discontinuity() 内部：
            //   m_startClock = 当前系统时间
            //   m_iDisc = clock（记录不连续点的值）
            //   清除所有累积的调整量
            
            // 3. 通知两个线程"从这个时钟点开始同步播放"
            if (m_CurrentVideo.inited) {
                m_VideoPlayerVideo->SendMessage(
                    new CDVDMsgGeneralResync(clock), 1);
                m_CurrentVideo.syncState = SYNC_INSYNC;
            }
            if (m_CurrentAudio.inited) {
                m_VideoPlayerAudio->SendMessage(
                    new CDVDMsgGeneralResync(clock), 1);
                m_CurrentAudio.syncState = SYNC_INSYNC;
            }
        }
    }
}
```

### 同步状态三阶段详解

音视频同步启动不是一个瞬间完成的操作，而是一个三阶段协商过程：

```
阶段 1：SYNC_STARTING
  ├── 触发：FlushBuffers() 将 syncState 重置为 STARTING
  ├── 含义：解码线程刚开始接收新位置的数据，还没解码出第一帧
  └── 持续时间：通常很短，取决于关键帧间距（I 帧间距越大，等待越久）

阶段 2：SYNC_WAITSYNC
  ├── 触发：解码线程解码出足够的数据后，发送 PLAYER_STARTED 消息
  │         消息中携带 syncState = SYNC_WAITSYNC
  ├── 含义："我已经准备好了，请告诉我从哪个时间点开始播放"
  └── 等待条件：等另一个流（音频/视频）也到达 WAITSYNC

阶段 3：SYNC_INSYNC
  ├── 触发：HandlePlaySpeed() 检测到两者都到达 WAITSYNC
  │         调用 m_clock.Discontinuity(clock)
  │         发送 GENERAL_RESYNC 给两个线程
  ├── 含义：两个线程已经对齐到同一个时钟起点，正常播放
  └── 持续到：下一次 Seek 或播放结束
```

**为什么需要三阶段？** 考虑这个场景：

```
Seek 到 01:30:00 位置后：

视频线程：
  - 需要找到最近的 I 帧（可能在 01:29:58 位置）
  - 解码 01:29:58 → 01:30:00 之间的所有 P/B 帧
  - 才能得到 01:30:00 这一帧的完整画面
  - 这个过程可能需要 200ms

音频线程：
  - 音频不依赖关键帧，任意位置都能直接解码
  - 几乎立即就准备好了（< 10ms）

如果不等视频线程就开始播放音频：
  → 用户听到 01:30:00 的声音，但屏幕还是黑的
  → 200ms 后画面突然跳出来
  → 用户感觉画面"闪"了一下

三阶段同步确保：
  → 两者都准备好后才同时开始
  → 画面和声音完美对齐
```

---

## 步骤 C & D：ReadPacket() 与 ProcessPacket()

### ReadPacket：读取数据包

**源码位置**：`cpp/core/movie/ffmpeg/VideoPlayer.cpp` 第 603 行附近

```cpp
// VideoPlayer.cpp ReadPacket() — 从解封装器读取一个数据包
DemuxPacket* BasePlayer::ReadPacket() {
    // 调用解封装器读取下一个包
    DemuxPacket* pPacket = m_CurrentDemuxer->Read();
    
    if (pPacket == nullptr) {
        // 读取到 EOF（End of File，文件结束）
        
        // 检查是否需要循环播放
        if (m_iLoopSegmentBegin != DVD_NOPTS_VALUE) {
            // 有循环段设置 → seek 回循环起点
            // m_iLoopSegmentBegin 是由 TVPMoviePlayer 设置的循环起点
            SeekTime(m_iLoopSegmentBegin);
            return nullptr;  // 本轮循环不处理数据
        }
        
        // 不循环 → 通知所有子线程文件结束
        m_VideoPlayerVideo->SendMessage(
            new CDVDMsg(CDVDMsg::GENERAL_EOF));
        m_VideoPlayerAudio->SendMessage(
            new CDVDMsg(CDVDMsg::GENERAL_EOF));
        
        // 最终：停止播放器
        SetSpeed(0);    // 暂停
        SeekTime(0);    // 回到开头
        // 触发 KRMovieEvent::Ended 回调通知 KrKr2 引擎
        
        return nullptr;
    }
    
    return pPacket;
}
```

### ProcessPacket：按流类型分发

```cpp
// VideoPlayer.cpp ProcessPacket() — 将数据包送入对应的解码线程
void BasePlayer::ProcessPacket(DemuxPacket* pPacket) {
    // 根据包的 iStreamId 判断是哪个流的数据
    
    if (pPacket->iStreamId == m_CurrentVideo.id) {
        // 视频包 → 包装成 CDVDMsgDemuxerPacket → 送入视频解码线程
        m_VideoPlayerVideo->SendMessage(
            new CDVDMsgDemuxerPacket(pPacket, false),
            0);  // priority=0（普通优先级）
        // 注意：数据包以普通优先级发送
        // 这样控制消息（FLUSH/RESET）总能优先被处理
    }
    else if (pPacket->iStreamId == m_CurrentAudio.id) {
        // 音频包 → 送入音频解码线程
        m_VideoPlayerAudio->SendMessage(
            new CDVDMsgDemuxerPacket(pPacket, false),
            0);
    }
    else {
        // 既不是视频也不是音频（可能是字幕或数据流）
        // KrKr2 裁剪了字幕支持，直接丢弃
        CDVDDemuxUtils::FreeDemuxPacket(pPacket);
    }
}
```

### 消息队列的容量限制

视频和音频解码线程的消息队列有容量上限，防止内存无限增长：

```cpp
// VideoPlayerVideo.cpp 消息队列配置
// 最大数据量：40MB
// 最大时间跨度：8.0 秒
m_messageQueue.SetMaxDataSize(40 * 1024 * 1024);  // 40MB
m_messageQueue.SetMaxTimeSize(8.0);                // 8 秒

// 当队列满时，BasePlayer 的 SendMessage() 会阻塞
// 直到解码线程消费掉一些消息腾出空间
// 这形成了自然的背压（backpressure）机制：
//   解码慢 → 队列满 → 读取暂停 → 避免内存溢出
```

---

## CVideoPlayerVideo：视频解码线程

视频解码线程是独立运行的线程，从消息队列获取压缩数据包，解码为 YUV 帧后输出到渲染管理器。

**源码位置**：`cpp/core/movie/ffmpeg/VideoPlayerVideo.cpp`

### 线程主循环

```cpp
// VideoPlayerVideo.cpp CVideoPlayerVideo::Process()
// 源码：第 209-473 行

void CVideoPlayerVideo::Process() {
    while (!m_bStop) {
        CDVDMsg* pMsg;
        
        // 从消息队列获取消息
        // 超时 1000ms —— 即使没有消息也每秒醒一次检查 m_bStop
        MsgQueueReturnCode ret = m_messageQueue.Get(&pMsg, 1000);
        
        if (ret == MSGQ_TIMEOUT) {
            continue;  // 超时 → 回到循环顶部检查是否该退出
        }
        
        if (ret != MSGQ_OK) {
            break;  // 队列出错 → 退出线程
        }
        
        // ═══ 消息处理（按类型分支）═══
        
        // 高优先级控制消息——这些因为以 priority=1 入队，
        // 会在所有 DEMUXER_PACKET 之前被 Get() 取出
        
        if (pMsg->IsType(CDVDMsg::GENERAL_SYNCHRONIZE)) {
            // 同步屏障：确认已处理完之前的所有消息
            // 向 BasePlayer 发回确认
        }
        
        else if (pMsg->IsType(CDVDMsg::GENERAL_RESYNC)) {
            // Seek 后重新同步：用新时钟点重置内部 PTS 跟踪
            auto* resyncMsg = static_cast<CDVDMsgDouble*>(pMsg);
            m_iCurrentPts = resyncMsg->m_value;
            // 清空丢帧统计
            m_droppingStats.Reset();
        }
        
        else if (pMsg->IsType(CDVDMsg::GENERAL_FLUSH)) {
            // 清空解码器内部缓冲
            m_pVideoCodec->Reset();
            // avcodec_flush_buffers() 内部调用
            m_iCurrentPts = DVD_NOPTS_VALUE;
        }
        
        else if (pMsg->IsType(CDVDMsg::GENERAL_RESET)) {
            // 重置解码器（比 FLUSH 更彻底）
            m_pVideoCodec->Reset();
        }
        
        else if (pMsg->IsType(CDVDMsg::GENERAL_EOF)) {
            // 文件结束：解码器中可能还有缓存的帧
            // 需要 "flush" 解码器（送空包让它输出剩余帧）
        }
        
        else if (pMsg->IsType(CDVDMsg::PLAYER_SETSPEED)) {
            m_speed = static_cast<CDVDMsgInt*>(pMsg)->m_value;
        }
        
        else if (pMsg->IsType(CDVDMsg::DEMUXER_PACKET)) {
            // ═══ 核心：解码数据包 ═══
            DemuxPacket* pPacket =
                static_cast<CDVDMsgDemuxerPacket*>(pMsg)->GetPacket();
            
            // 调用 FFmpeg 解码
            // 注意：使用旧版 API avcodec_decode_video2()
            int iDecoderState = m_pVideoCodec->Decode(
                pPacket->pData, pPacket->iSize,
                pPacket->dts, pPacket->pts);
            
            // 处理解码结果
            // iDecoderState 是位掩码（bitmask），可以同时包含多个标志
            ProcessDecoderOutput(iDecoderState);
        }
        
        pMsg->Release();  // 释放消息（引用计数 -1）
    }
}
```

### ProcessDecoderOutput：处理解码结果

**源码位置**：`cpp/core/movie/ffmpeg/VideoPlayerVideo.cpp` 第 475-627 行

```cpp
// VideoPlayerVideo.cpp ProcessDecoderOutput() — 处理解码器返回的状态
void CVideoPlayerVideo::ProcessDecoderOutput(int iDecoderState) {
    
    // ─── VC_FLUSHED：解码器被冲刷 ───
    if (iDecoderState & VC_FLUSHED) {
        // 解码器在处理某个包时内部执行了 flush
        // 需要将之前缓冲但未送出的包重新送入解码器
        // （某些编解码器在格式变化时会自动 flush）
    }
    
    // ─── VC_REOPEN：需要重新打开解码器 ───
    if (iDecoderState & VC_REOPEN) {
        // 视频参数发生变化（分辨率、像素格式等）
        // 需要关闭当前解码器并重新打开
    }
    
    // ─── VC_ERROR：解码错误 ───
    if (iDecoderState & VC_ERROR) {
        // 记录错误，但不停止播放
        // 某些损坏的帧可以被跳过
    }
    
    // ─── VC_PICTURE：成功解码出一帧 ───
    if (iDecoderState & VC_PICTURE) {
        // 获取解码后的帧数据
        DVDVideoPicture picture;
        if (m_pVideoCodec->GetPicture(&picture)) {
            // 输出到渲染管理器
            OutputPicture(&picture);
        }
    }
    
    // ─── VC_BUFFER：解码器需要更多数据 ───
    if (iDecoderState & VC_BUFFER) {
        // 当前包已被解码器消费完
        // 下一轮循环会从消息队列取新的包
    }
}
```

### OutputPicture：帧输出与丢帧判定

`OutputPicture()` 是视频解码线程中最复杂的函数，负责将解码后的帧按正确时序输出。

**源码位置**：`cpp/core/movie/ffmpeg/VideoPlayerVideo.cpp` 第 714-872 行

```cpp
// VideoPlayerVideo.cpp OutputPicture() — 帧输出核心流程
int CVideoPlayerVideo::OutputPicture(DVDVideoPicture* pPicture) {
    
    // ════════════════════════════
    // 第 1 步：PTS 处理
    // ════════════════════════════
    double pts = pPicture->pts;
    if (pts == DVD_NOPTS_VALUE) {
        // 没有有效的 PTS → 用帧率推算
        // pts = 上一帧的 PTS + 帧间隔
        pts = m_iCurrentPts + (double)DVD_TIME_BASE / m_fFrameRate;
    }
    m_iCurrentPts = pts;
    
    // ════════════════════════════
    // 第 2 步：配置渲染器
    // ════════════════════════════
    // 如果视频参数（分辨率、像素格式）发生变化
    // 需要重新配置 CRenderManager
    if (m_bConfigureNeeded) {
        m_renderManager.Configure(pPicture);
        // Configure() 会设置帧缓冲队列大小、渲染器参数等
    }
    
    // ════════════════════════════
    // 第 3 步：帧率计算
    // ════════════════════════════
    CalcFrameRate();
    // 通过最近 N 帧的 PTS 差值统计实际帧率
    // 用于后续的丢帧判定和 VSync 对齐
    
    // ════════════════════════════
    // 第 4 步：丢帧判定
    // ════════════════════════════
    int dropReq = CalcDropRequirement();
    
    if (dropReq > 0 && m_speed == DVD_PLAYSPEED_NORMAL) {
        // 正常速度播放时，如果视频落后于时钟 → 丢帧
        m_droppingStats.m_dropcount++;
        return EOS_DROPPED;
        // 丢弃此帧，不送入渲染器
        // 这样解码线程可以更快地追上时钟
    }
    
    // 快进/快退时使用不同的丢帧策略
    if (m_speed != DVD_PLAYSPEED_NORMAL) {
        double clock = m_pClock->GetClock();
        double frametime = (double)DVD_TIME_BASE / m_fFrameRate;
        
        if (m_speed > 0) {
            // 快进：只显示每 N 帧中的 1 帧
            // N 取决于快进倍率
        } else {
            // 快退（rewind）：从后向前显示
            // 由于视频压缩的方向性（I→P→B），快退实际上是
            // 反复 seek 到前面的关键帧 → 正向解码 → 只显示最后一帧
        }
    }
    
    // ════════════════════════════
    // 第 5 步：等待渲染缓冲区可用
    // ════════════════════════════
    if (!m_renderManager.WaitForBuffer(m_bAbortRequest)) {
        return EOS_ABORT;
        // 如果被中断（停止播放），返回 ABORT
    }
    // WaitForBuffer() 会阻塞直到渲染器有空闲缓冲区
    // 在 KrKr2 中，这直接委托给 TVPMoviePlayer::WaitForBuffer()
    // 它检查环形缓冲区是否有空闲槽位
    
    // ════════════════════════════
    // 第 6 步：提交帧到渲染管理器
    // ════════════════════════════
    m_renderManager.AddVideoPicture(*pPicture, m_bAbortRequest);
    // 在 KrKr2 中，这直接委托给 TVPMoviePlayer::AddVideoPicture()
    // 将 YUV 数据拷贝到环形缓冲区的当前槽位
    
    // ════════════════════════════
    // 第 7 步：通知渲染器翻页
    // ════════════════════════════
    m_renderManager.FlipPage(m_bAbortRequest, pts);
    // FlipPage() 将帧从 free 队列移到 queued 队列
    // 标记 PRESENT_READY，通知渲染循环有新帧可用
    
    return EOS_OK;
}
```

### CalcDropRequirement：智能丢帧算法

**源码位置**：`cpp/core/movie/ffmpeg/VideoPlayerVideo.cpp` 第 1013-1090 行

这不是简单的"落后就丢"，而是基于统计数据的智能决策：

```cpp
// VideoPlayerVideo.cpp CalcDropRequirement() — 丢帧决策算法
int CVideoPlayerVideo::CalcDropRequirement() {
    
    // ─── 第 1 步：计算当前延迟 ───
    double iDecoderPts = m_iCurrentPts;        // 当前解码到的 PTS
    double iClock = m_pClock->GetClock();       // 当前播放时钟
    double iLateness = iClock - iDecoderPts;    // 正值 = 视频落后于时钟
    
    double frametime = (double)DVD_TIME_BASE / m_fFrameRate;
    // frametime = 一帧的持续时间（微秒）
    // 例如 30fps → frametime = 33333 微秒 ≈ 33ms
    
    // ─── 第 2 步：更新丢帧统计 ───
    // CDroppingStats 跟踪最近一段时间的丢帧情况
    // 包括：连续迟到帧数（m_lateframes）、累计增益（gain）等
    
    if (iLateness > 0) {
        m_droppingStats.m_lateframes++;
        m_droppingStats.AddOutputDropGain(iLateness);
        // AddOutputDropGain 累积"通过丢帧节省了多少时间"
    } else {
        m_droppingStats.m_lateframes = 0;
        // 不再落后 → 重置连续迟到计数
    }
    
    // ─── 第 3 步：计算丢帧所需的增益 ───
    // 需要丢多少帧才能追上时钟？
    double gain = iLateness;  // 需要追回的时间
    
    // ─── 第 4 步：智能阈值判定 ───
    // 根据连续迟到帧数调整丢帧的"宽容度"
    
    int lateframes = m_droppingStats.m_lateframes;
    double allowedLateness;
    
    if (lateframes <= 6) {
        // 轻微落后：允许最多 0.98 倍帧时间的宽限
        // 因为可能只是偶尔的 IO 抖动，不需要丢帧
        allowedLateness = 0.98 * frametime;
    }
    else if (lateframes <= 10) {
        // 持续落后：逐渐收紧宽限
        // 线性插值：从 0.98 * frametime 降到 0
        double factor = 1.0 - (lateframes - 6) / 4.0;
        allowedLateness = factor * 0.98 * frametime;
    }
    else {
        // 严重落后（>10 帧）：零容忍
        // 任何落后都会触发丢帧
        allowedLateness = 0;
    }
    
    // ─── 第 5 步：最终判定 ───
    if (gain > allowedLateness) {
        return 1;  // 需要丢帧
    }
    
    return 0;  // 不需要丢帧
}
```

**丢帧策略可视化**：

```
连续迟到帧数:  0  1  2  3  4  5  6  7  8  9  10  11  12 ...
宽限系数:      ─────── 0.98 ──────  ↘ 递减 ↘   ──── 0 ────
丢帧阈值:     [        约 32ms        ] [递减] [  0ms（零容忍）]

行为示例（30fps，帧时间 ≈ 33ms）：

  落后 20ms + 连续迟到 3 帧 → 宽限 32ms → 不丢帧
  落后 20ms + 连续迟到 8 帧 → 宽限 16ms → 需要丢帧
  落后  5ms + 连续迟到 11帧 → 宽限  0ms → 必须丢帧

为什么这样设计？
  - 偶尔迟到（≤6 帧）很正常，可能是 IO 抖动
    → 宽容处理，避免不必要的丢帧导致画面跳跃
  - 持续迟到（7-10 帧）说明解码确实跟不上
    → 逐渐收紧，开始丢帧追赶
  - 严重迟到（>10 帧）需要紧急恢复
    → 零容忍，快速丢帧直到追上时钟
```

---

## EOF 与循环播放处理

当文件读到末尾时，BasePlayer 有两条路径：

```cpp
// VideoPlayer.cpp — EOF 处理逻辑
// 源码：第 607-615 行

// 路径 1：循环播放
if (m_iLoopSegmentBegin != DVD_NOPTS_VALUE) {
    // TVPMoviePlayer 设置了循环段
    // m_iLoopSegmentBegin 是循环起点（微秒）
    
    // seek 回循环起点，继续播放
    SeekTime(m_iLoopSegmentBegin);
    // 这会触发完整的 Seek 流程：
    //   FlushBuffers → CDVDDemuxFFmpeg::SeekTime → 重新缓冲 → 同步启动
    return;
}

// 路径 2：正常结束
// 通知所有子线程文件已结束
m_VideoPlayerVideo->SendMessage(
    new CDVDMsg(CDVDMsg::GENERAL_EOF));
m_VideoPlayerAudio->SendMessage(
    new CDVDMsg(CDVDMsg::GENERAL_EOF));

// 停止并回到开头
SetSpeed(0);    // 速度设为 0 → 暂停
SeekTime(0);    // 回到文件开头

// 触发 KrKr2 事件回调
// → KRMovieEvent::Ended
// → TJS2 脚本的 onPlayCompleted 回调
```

**循环播放在视觉小说中的典型用途**：
- 标题画面的背景动画循环播放
- 某些场景的环境视频（窗外的雨、飘动的窗帘等）

---

## 常见错误与排查

### 错误 1：Seek 后画面花屏

```
症状：跳转后前几帧出现花屏（马赛克块、颜色错乱）

原因分析：
  视频编码使用了 B 帧（双向预测帧），它依赖前后的 I 帧和 P 帧。
  Seek 后 av_seek_frame() 定位到最近的关键帧（I 帧），
  但如果 FlushBuffers() 没有正确执行 GENERAL_RESET，
  解码器内部的参考帧缓存可能残留旧数据。

排查步骤：
  1. 确认 FlushBuffers() 中的三条消息（RESET → FLUSH → SYNCHRONIZE）
     都以高优先级发送
  2. 检查 CDVDVideoCodecFFmpeg::Reset() 是否调用了 avcodec_flush_buffers()
  3. 检查是否有竞争条件：RESET 消息在 FLUSH 之前被处理

修复思路：
  确保 GENERAL_RESET 在解码线程中正确调用 avcodec_flush_buffers()，
  该函数会清除解码器内部所有参考帧缓存。
```

### 错误 2：音视频不同步（画面超前或滞后）

```
症状：画面和声音对不上，差距随时间增大

原因分析：
  HandlePlaySpeed() 中的同步启动流程可能没有正确执行。
  如果 m_clock.Discontinuity() 没有被调用，
  或者 GENERAL_RESYNC 消息的时钟值不正确，
  音视频会从不同的起点开始播放。

排查步骤：
  1. 在 HandlePlaySpeed() 的同步代码处加日志，
     打印 videoSyncState 和 audioSyncState
  2. 检查是否两者都到达了 SYNC_WAITSYNC
  3. 检查 Discontinuity() 传入的 clock 值是否合理

修复思路：
  在 SynchronizePlayers 逻辑中打印日志：
  - 视频流的 startPts、音频流的 startPts
  - 最终选择的 clock 值
  - 两者的差值
  差值应该 < 100ms，否则说明某个流的起始 PTS 不正确。
```

### 错误 3：播放卡顿但 CPU 使用率不高

```
症状：播放时偶尔卡顿 100-200ms，但 CPU 占用率只有 30%

原因分析：
  可能是缓存状态机的问题——CACHESTATE 在 PLAY 和 FULL 之间震荡。
  每次进入 FULL 状态时时钟会暂停（SetSpeed(PAUSE)），
  导致画面短暂停顿。

排查步骤：
  1. 在 SetCaching() 中加日志，打印每次状态转换
  2. 如果看到 PLAY → FULL → PLAY 频繁切换，
     说明缓冲阈值设置不合理
  3. 检查磁盘 IO 速度——可能是读取速度不够导致缓冲区频繁变空

修复思路：
  调整缓冲阈值——降低"重新缓冲"的触发条件，
  或增大消息队列的 MaxDataSize（默认 40MB）。
```

---

## 对照项目源码

本节涉及的所有源码文件及关键位置：

- `cpp/core/movie/ffmpeg/VideoPlayer.cpp` — BasePlayer 主控逻辑
  - 第 273-343 行：`OpenFromStream()` 初始化入口
  - 第 538-741 行：`Process()` 主循环
  - 第 603-615 行：`ReadPacket()` + EOF/循环处理
  - 第 1091-1208 行：`FlushBuffers()` 三阶段缓冲清空
  - 第 1210-1669 行：`HandleMessages()` 消息处理
  - 第 1671-1718 行：`SetCaching()` 状态转换
  - 第 1865-2183 行：`HandlePlaySpeed()` 缓存状态机 + 同步启动
  - 第 1991-2094 行：音视频同步启动逻辑

- `cpp/core/movie/ffmpeg/VideoPlayerVideo.cpp` — 视频解码线程
  - 第 209-473 行：`Process()` 线程主循环
  - 第 475-627 行：`ProcessDecoderOutput()` 解码结果处理
  - 第 714-872 行：`OutputPicture()` 帧输出
  - 第 1013-1090 行：`CalcDropRequirement()` 丢帧算法

---

## 动手实践

### 实践 1：跟踪缓存状态机转换

在 `VideoPlayer.cpp` 中找到 `SetCaching()` 函数（第 1671-1718 行）。回答以下问题：

1. `SetCaching(CACHESTATE_FLUSH)` 时，时钟速度被设为什么？
2. `SetCaching(CACHESTATE_PLAY)` 时，时钟速度被恢复为什么值？
3. `SetCaching(CACHESTATE_DONE)` 时，与 PLAY 状态有什么区别？

### 实践 2：修改丢帧阈值

找到 `CalcDropRequirement()` 中的宽限系数 `0.98`。将其改为 `0.5`（更激进的丢帧）和 `1.5`（更保守的丢帧），预测每种情况下播放器行为会有什么变化。

### 实践 3：添加调试日志

在 `HandlePlaySpeed()` 的同步启动代码（第 1991 行附近）添加一行日志，打印当前的 `m_CurrentVideo.syncState` 和 `m_CurrentAudio.syncState`。使用 spdlog（KrKr2 项目使用的日志库）的格式：

```cpp
spdlog::get("core")->debug(
    "SyncState: video={}, audio={}",
    static_cast<int>(m_CurrentVideo.syncState),
    static_cast<int>(m_CurrentAudio.syncState));
```

---

## 本节小结

- `BasePlayer::Process()` 主循环按固定顺序执行四步：**HandleMessages → HandlePlaySpeed → ReadPacket → ProcessPacket**
- `HandleMessages()` 负责处理 Seek、速度变更等控制消息，其中 `FlushBuffers()` 是 Seek 的核心，按 **RESET → FLUSH → SYNCHRONIZE** 三阶段执行
- 缓存状态机有五个状态：**FLUSH → FULL/INIT → PLAY → DONE**，通过暂停/恢复时钟来控制播放节奏
- 音视频同步启动是三阶段协商：**SYNC_STARTING → SYNC_WAITSYNC → SYNC_INSYNC**，确保音画同时开始
- `CVideoPlayerVideo` 在独立线程中运行，通过消息队列接收数据包，使用旧版 FFmpeg API 解码
- `OutputPicture()` 在提交帧前会调用 `CalcDropRequirement()` 进行智能丢帧判定
- 丢帧策略基于连续迟到帧数动态调整宽限度：≤6 帧宽容（0.98 倍帧时间），>10 帧零容忍
- EOF 时支持循环播放（通过 `m_iLoopSegmentBegin` seek 回起点）

---

## 练习题与答案

### 题目 1：缓存状态机设计分析

为什么缓存状态机需要区分 `CACHESTATE_INIT` 和 `CACHESTATE_FULL` 两个状态？它们的缓冲阈值有什么不同？如果合并为一个状态会有什么问题？

<details>
<summary>查看答案</summary>

**区别：**
- `CACHESTATE_INIT` 用于首次启动播放，使用**较低**的缓冲阈值，让用户更快看到画面
- `CACHESTATE_FULL` 用于 Seek 之后的重新缓冲，使用**较高**的缓冲阈值，确保 seek 后播放流畅

**为什么不同？**
- 首次启动时，用户的心理预期是"点击播放后尽快看到画面"，可以容忍初始几百毫秒的轻微卡顿
- Seek 后，用户已经在观看中，对卡顿的容忍度更低。更高的缓冲阈值确保 seek 后有足够的数据储备，避免刚恢复播放就因缓冲不足再次暂停

**如果合并为一个状态：**
- 如果统一用低阈值：seek 后可能频繁出现"播放 → 缓冲 → 播放 → 缓冲"的震荡
- 如果统一用高阈值：首次启动等待时间过长，用户以为播放器卡死了

这种区分是 Kodi 多年实践经验的结晶——同一个功能（缓冲）在不同场景下的最优参数是不同的。

</details>

### 题目 2：消息优先级的实际场景

假设视频解码线程的消息队列中当前有以下消息（按入队顺序）：

```
[DEMUXER_PACKET_1(p=0)] [DEMUXER_PACKET_2(p=0)] [DEMUXER_PACKET_3(p=0)]
```

此时 BasePlayer 执行了 FlushBuffers()，依次发送了：

```
[GENERAL_RESET(p=1)] [GENERAL_FLUSH(p=1)] [GENERAL_SYNCHRONIZE(p=1)]
```

请问 CDVDMessageQueue::Get() 的调用者会以什么顺序取到这 6 条消息？

<details>
<summary>查看答案</summary>

取出顺序：

```
1. GENERAL_RESET        (p=1) — 高优先级，最先被取出
2. GENERAL_FLUSH         (p=1) — 高优先级，第二个取出
3. GENERAL_SYNCHRONIZE   (p=1) — 高优先级，第三个取出
4. DEMUXER_PACKET_1      (p=0) — 普通优先级
5. DEMUXER_PACKET_2      (p=0) — 普通优先级
6. DEMUXER_PACKET_3      (p=0) — 普通优先级
```

**解释**：CDVDMessageQueue::Get() 首先扫描整个队列寻找 priority > 0 的消息。由于三条高优先级消息是用 `push_front()` 插入的（在队列头部），它们会按照 RESET → FLUSH → SYNCHRONIZE 的顺序被依次取出。

**实际效果**：
- RESET 先执行：解码器释放内部参考帧
- FLUSH 再执行：清空队列中剩余的旧数据包（PACKET_1/2/3 被丢弃）
- SYNCHRONIZE 最后：确认清理完成

注意：如果 FLUSH 正确实现了"清空队列中所有 DEMUXER_PACKET"的逻辑，那么 PACKET_1/2/3 实际上在第 2 步就被丢弃了，第 4-6 步不会发生。上面列出 6 步是假设 Get() 的行为——在实际代码中，FLUSH 的处理逻辑会直接清空队列中的普通消息。

</details>

### 题目 3：修改丢帧策略的影响

将 `CalcDropRequirement()` 中的连续迟到帧阈值从 6 帧改为 2 帧（即 lateframes > 2 时就开始收紧宽限），分析以下两种场景的影响：

场景 A：720p 30fps 视频，在性能良好的桌面电脑上播放
场景 B：4K 60fps 视频，在性能较弱的移动设备上播放

<details>
<summary>查看答案</summary>

**场景 A（720p 30fps，高性能电脑）：**
- 解码速度远快于帧率要求，很少出现连续迟到
- 将阈值从 6 改为 2 影响很小——因为连续迟到很少超过 1-2 帧
- 偶尔的 IO 抖动（磁盘忙碌）可能导致 2-3 帧迟到
- 改为 2 帧阈值后，这种偶尔的抖动也会触发丢帧
- **影响**：画面可能偶尔出现不必要的丢帧跳跃，但整体体验变化不大

**场景 B（4K 60fps，弱设备）：**
- 解码刚好能跟上或略慢于帧率要求
- 经常出现 3-6 帧的连续迟到（正常波动范围内）
- 原始阈值 6 帧允许这种波动不触发丢帧
- 改为 2 帧阈值后：
  - 几乎每次波动都会触发丢帧
  - 丢帧 → 短暂追上时钟 → 解码再次略落后 → 再次丢帧
  - 形成"丢帧震荡"：画面不流畅，持续有跳帧感
- **影响**：严重影响观看体验。用户会看到画面频繁跳跃

**结论**：6 帧的阈值是在"及时响应真正的性能问题"和"容忍正常的解码波动"之间的平衡点。降低阈值会导致过度敏感，特别是在性能边缘的设备上。

</details>

---

## 下一步

[TVPMoviePlayer 与时钟系统](./03-TVPMoviePlayer与时钟系统.md) — 深入 KrKr2 定制层：TVPMoviePlayer 的双重继承桥接设计、YUV 环形缓冲区的无锁写入、PresentPicture 的 delta-time 渲染调度、CDVDClock 的速度调整与不连续点处理。
