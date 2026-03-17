# 5.4 CDVDClock 时钟系统详解
## 5.4 CDVDClock 时钟系统详解

### 5.4.1 时钟架构

`CDVDClock` 是 KrKr2 播放器的核心时钟组件，维护一个**可暂停、可变速、可跳变**的播放时钟：

```
┌──────────────────────────────────────────────┐
│                  CDVDClock                     │
│                                               │
│  CVideoReferenceClock (系统时钟源)             │
│      │                                        │
│      ▼ GetTime()                              │
│  m_systemOffset ──┐                           │
│                   ▼                           │
│  SystemToAbsolute(system)                     │
│      = (system - m_systemOffset)              │
│        / m_systemFrequency * DVD_TIME_BASE    │
│                   │                           │
│                   ▼                           │
│  SystemToPlaying(system)                      │
│      = (current - m_startClock + m_systemAdjust)│
│        / m_systemUsed * DVD_TIME_BASE + m_iDisc│
│                                               │
│  关键变量：                                    │
│  ├── m_startClock    起始系统时间              │
│  ├── m_systemUsed    有效频率（受速度影响）     │
│  ├── m_iDisc         不连续跳变量              │
│  ├── m_systemAdjust  累计微调量                │
│  ├── m_speedAdjust   速度微调系数              │
│  ├── m_pauseClock    暂停时冻结的时间          │
│  └── m_vSyncAdjust   VSync 校正量             │
└──────────────────────────────────────────────┘
```

### 5.4.2 三层时间模型

`CDVDClock` 维护了三层时间概念：

| 层次 | 方法 | 含义 |
|------|------|------|
| 系统时间 | `m_videoRefClock->GetTime()` | 单调递增的系统时钟（纳秒/微秒级） |
| 绝对时间 | `SystemToAbsolute()` | 从系统时间换算为 `DVD_TIME_BASE` 单位 |
| 播放时间 | `SystemToPlaying()` | 考虑暂停、变速、跳变后的实际播放位置 |

```cpp
// 来源：Clock.cpp 第 234-237 行
// 系统时间 → 绝对时间
double CDVDClock::SystemToAbsolute(int64_t system) {
    return DVD_TIME_BASE * (double)(system - m_systemOffset) /
        m_systemFrequency;
}

// 来源：Clock.cpp 第 244-267 行
// 系统时间 → 播放时间
double CDVDClock::SystemToPlaying(int64_t system) {
    // 首次调用时重置
    if (m_bReset) {
        m_startClock = system;
        m_systemUsed = m_systemFrequency;
        m_iDisc = 0;
        m_systemAdjust = 0;
        m_speedAdjust = 0;
        m_bReset = false;
    }

    int64_t current;
    if (m_pauseClock)
        current = m_pauseClock;   // 暂停时使用冻结时间
    else
        current = system;         // 正常播放使用当前时间

    // 核心公式：播放时间 = 经过的系统时间 / 有效频率 * 基准 + 不连续量
    return DVD_TIME_BASE * (double)(current - m_startClock + m_systemAdjust) /
        m_systemUsed + m_iDisc;
}
```

### 5.4.3 速度控制

播放速度通过调整 `m_systemUsed`（有效频率）实现：

```cpp
// 来源：Clock.cpp 第 91-119 行
void CDVDClock::SetSpeed(int iSpeed) {
    // iSpeed: DVD_PLAYSPEED_NORMAL=1000 代表 1.0x
    //         500 代表 0.5x，2000 代表 2.0x
    //         DVD_PLAYSPEED_PAUSE=0 代表暂停

    if (iSpeed == DVD_PLAYSPEED_PAUSE) {
        if (!m_pauseClock)
            m_pauseClock = m_videoRefClock->GetTime();  // 冻结当前时间
        return;
    }

    // 计算新的有效频率
    int64_t newfreq = m_systemFrequency * DVD_PLAYSPEED_NORMAL / iSpeed;
    // 例：2x 速度 → newfreq = freq * 1000 / 2000 = freq / 2
    // 分母减半，同样的系统时间流逝，播放时间增长 2 倍

    current = m_videoRefClock->GetTime();

    // 从暂停恢复时，补偿暂停期间的时间
    if (m_pauseClock) {
        m_startClock += current - m_pauseClock;
        m_pauseClock = 0;
    }

    // 平滑切换速度：重新计算 startClock 使得时间连续
    m_startClock = current -
        (int64_t)((double)(current - m_startClock) * newfreq / m_systemUsed);
    m_systemUsed = newfreq;
}
```

### 5.4.4 误差校正（ErrorAdjust）

这是 SYNC_DISCON 模式的核心——当检测到音视频偏差时，通过跳变时钟来纠正：

```cpp
// 来源：Clock.cpp 第 134-169 行
double CDVDClock::ErrorAdjust(double error, const char *log) {
    double clock, absolute, adjustment;
    clock = GetClock(absolute);

    // 当 speedAdjust 活跃且误差较小（<100ms）时跳过微调
    // 避免与速度调整冲突
    if (m_speedAdjust != 0 && error < DVD_MSEC_TO_TIME(100))
        return 0;

    adjustment = error;

    if (m_vSyncAdjust != 0) {
        // VSync 模式下使用帧时间为单位进行校正
        // 音频超前 >20ms → 向前调整一帧时间
        // 音频落后 >27ms → 向后调整一帧时间
        // 20ms/27ms 的不对称阈值正是因为人耳对音频领先更敏感
        if (error > 0.02 * DVD_TIME_BASE)
            adjustment = m_frameTime;       // +1 帧
        else if (error < -0.027 * DVD_TIME_BASE)
            adjustment = -m_frameTime;      // -1 帧
        else
            adjustment = 0;                 // 在容差内，不调整
    }

    if (adjustment == 0)
        return 0;

    // 执行不连续跳变
    Discontinuity(clock + adjustment, absolute);
    return adjustment;
}
```

### 5.4.5 不连续跳变（Discontinuity）

`Discontinuity` 方法直接重置时钟到新的播放位置，通常在 seek、流切换或误差校正时调用：

```cpp
// 来源：Clock.cpp 第 171-180 行
void CDVDClock::Discontinuity(double clock, double absolute) {
    m_startClock = AbsoluteToSystem(absolute);  // 重置起始点
    if (m_pauseClock)
        m_pauseClock = m_startClock;
    m_iDisc = clock;              // 设置新的基准偏移
    m_bReset = false;
    m_systemAdjust = 0;           // 清除累计微调
    m_speedAdjust = 0;            // 清除速度微调
}
```

---
## 5.5 音频同步状态机

### 5.5.1 CVideoPlayerAudio 的同步状态

`CVideoPlayerAudio` 维护了一个三状态的同步状态机：

```
                 首帧输出
SYNC_STARTING ──────────> SYNC_WAITSYNC
    │                         │
    │ flush/reset             │ 收到 RESYNC 消息
    │                         ▼
    └──────────────────── SYNC_INSYNC
                              │
                              │ flush/reset
                              ▼
                         SYNC_STARTING
```

| 状态 | 含义 | 行为 |
|------|------|------|
| `SYNC_STARTING` | 刚开始或重置后 | 解码并填充音频缓冲区，等待足够数据 |
| `SYNC_WAITSYNC` | 音频就绪，等待视频同步 | 向主循环发送 `PLAYER_STARTED` 消息 |
| `SYNC_INSYNC` | 正常同步播放 | 持续解码+输出，检测同步误差 |

### 5.5.2 音频处理主循环

`CVideoPlayerAudio::Process` 是音频线程的主循环（第 194-496 行），采用消息驱动架构：

```
┌─────────────────────────────────────────────────┐
│            CVideoPlayerAudio::Process            │
│                                                  │
│  while (!m_bStop) {                              │
│      msg = m_messageQueue.Get(timeout, priority) │
│      │                                           │
│      ├── GENERAL_SYNCHRONIZE → 等待同步信号       │
│      ├── GENERAL_RESYNC → 设置 audioClock + INSYNC│
│      ├── GENERAL_RESET → 重置解码器 + STARTING    │
│      ├── GENERAL_FLUSH → 清空缓冲区               │
│      ├── PLAYER_SETSPEED → 调整播放速度           │
│      ├── AUDIO_SILENCE → 静音控制                 │
│      ├── GENERAL_STREAMCHANGE → 切换编解码器       │
│      ├── GENERAL_PAUSE → 暂停/恢复                │
│      └── DEMUXER_PACKET → ★ 核心解码路径 ★        │
│          │                                       │
│          ├── 1. Decode(pData, iSize, dts, pts)    │
│          ├── 2. GetData(audioframe)               │
│          ├── 3. 更新 audioClock                    │
│          ├── 4. 检查同步状态                       │
│          ├── 5. OutputPacket(audioframe)           │
│          │      └── 检测 syncerror                │
│          │      └── ErrorAdjust 修正时钟          │
│          └── 6. audioClock += duration             │
│  }                                               │
└─────────────────────────────────────────────────┘
```

### 5.5.3 同步误差检测与修正

在 `OutputPacket` 中，音频线程检测并修正同步误差：

```cpp
// 来源：VideoPlayerAudio.cpp 第 525-538 行
bool CVideoPlayerAudio::OutputPacket(DVDAudioFrame &audioframe) {
    // 获取音频输出与时钟之间的同步误差
    double syncerror = m_dvdAudio.GetSyncError();

    // SYNC_DISCON 模式：误差超过 10ms 时通过时钟跳变修正
    if (m_synctype == SYNC_DISCON && fabs(syncerror) > DVD_MSEC_TO_TIME(10)) {
        double correction = m_pClock->ErrorAdjust(
            syncerror, "CVideoPlayerAudio::OutputPacket");
        if (correction != 0) {
            m_dvdAudio.SetSyncErrorCorrection(-correction);
        }
    }

    // 将音频帧送入音频设备
    m_dvdAudio.AddPackets(audioframe);
    return true;
}
```

整个同步修正流程：

```
音频设备报告 syncerror (当前播放位置 vs 期望位置)
    │
    ▼
|syncerror| > 10ms ?
    │
    ├── 否 → 误差在容忍范围内，不调整
    │
    └── 是 → CDVDClock::ErrorAdjust(syncerror)
              │
              ├── VSync 模式：按帧时间步进调整（±1帧）
              │     20ms < error  → +1 帧
              │     error < -27ms → -1 帧
              │
              └── 非 VSync：直接跳变 error 量
                    │
                    ▼
              Discontinuity(clock + adjustment)
              → 时钟瞬间跳变，视频线程跟随
```

### 5.5.4 音频时钟的维护

音频播放时钟 `m_audioClock` 通过两种方式更新：

```cpp
// 方式 1：从解码帧的 PTS 更新（精确）
// 来源：VideoPlayerAudio.cpp 第 360-366 行
if (audioframe.pts == DVD_NOPTS_VALUE) {
    audioframe.pts = m_audioClock;       // 无 PTS 时用预测值
    audioframe.hasTimestamp = false;
} else {
    m_audioClock = audioframe.pts;       // 有 PTS 时直接采用
}

// 方式 2：根据帧持续时间递增（预测下一帧）
// 来源：VideoPlayerAudio.cpp 第 473 行
m_audioClock += audioframe.duration;
// duration = nb_samples / sample_rate * DVD_TIME_BASE
```

---
