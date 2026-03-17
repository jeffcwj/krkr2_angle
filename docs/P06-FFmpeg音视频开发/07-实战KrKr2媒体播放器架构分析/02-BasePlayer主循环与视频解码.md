# BasePlayer 主循环：Process() 详解
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

