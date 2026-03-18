# CDVDClock 主时钟

> **所属模块：** M06-视频播放器
> **前置知识：** [线程模型](../01-movie-ffmpeg架构/03-线程模型.md)、[CDVDPlayer 架构](../01-movie-ffmpeg架构/02-CDVDPlayer架构.md)
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 CDVDClock 作为音视频同步核心的设计原理和工作机制
2. 掌握 DVD_TIME_BASE、DVD_NOPTS_VALUE 等时间常量的含义与用途
3. 深入分析 SystemToPlaying() 时间映射公式的数学原理
4. 理解播放速度控制（SetSpeed）的频率比实现方式
5. 掌握暂停/恢复、速度微调（SpeedAdjust）的完整工作流程

## 为什么需要主时钟？

在视频播放系统中，音频和视频是两个独立的解码线程，各自以不同速率产出数据。音频采样率通常是 44100Hz 或 48000Hz，视频帧率通常是 23.976fps、24fps、25fps 或 29.97fps。如果没有统一的时间参考，音频和视频会各跑各的，很快就会出现"嘴型不同步"（lip sync）问题。

**主时钟（Master Clock）** 就是解决这个问题的核心组件。它提供一个统一的时间基准，所有流播放器（音频线程、视频线程）都向这个时钟对齐自己的输出时机。KrKr2 的 `CDVDClock` 继承自 Kodi/XBMC 的同名类，是整个播放器的"心跳"。

主时钟的三个核心职责：

1. **时间映射（Time Mapping）**：将操作系统的物理时间（system time）转换为播放时间（playing time）
2. **速度控制（Speed Control）**：通过调整频率比来实现变速播放、暂停等功能
3. **误差校正（Error Correction）**：当音视频出现偏差时，通过 Discontinuity 机制修正时钟

```
┌──────────────────────────────────────────────────────────────────┐
│                     CDVDClock 主时钟架构                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐    SystemToAbsolute()    ┌──────────────┐  │
│  │ CVideoReference │ ──── GetTime() ─────────>│ 绝对时间      │  │
│  │ Clock           │                          │ (Absolute)    │  │
│  │ (系统时钟源)     │                          └──────┬───────┘  │
│  └─────────────────┘                                  │          │
│                                                       │          │
│                                          SystemToPlaying()       │
│                                                       │          │
│                                                       ▼          │
│  ┌──────────────┐   GetClock()            ┌──────────────────┐  │
│  │ 音频线程      │ <─────────────────────── │ 播放时间          │  │
│  │ (Audio)      │                          │ (Playing Time)   │  │
│  ├──────────────┤   GetClock()            │                  │  │
│  │ 视频线程      │ <─────────────────────── │ = (current -     │  │
│  │ (Video)      │                          │   startClock +   │  │
│  ├──────────────┤                          │   systemAdjust)  │  │
│  │ BasePlayer   │   ErrorAdjust()          │   * TIME_BASE /  │  │
│  │ (主循环)      │ ───────────────────────> │   systemUsed     │  │
│  └──────────────┘   Discontinuity()        │   + iDisc        │  │
│                                            └──────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

## 时间常量体系

KrKr2 的时间系统继承自 Kodi，使用微秒（microsecond）作为基本单位。以下常量定义在 `Clock.h` 中：

```cpp
// 源码位置：cpp/core/movie/ffmpeg/Clock.h 第 8-16 行

#define DVD_TIME_BASE 1000000       // 时间基准：1秒 = 1,000,000 微秒
#define DVD_NOPTS_VALUE 0xFFF0000000000000  // 无效时间戳哨兵值

// 时间单位转换宏
#define DVD_TIME_TO_MSEC(x) ((int)((double)(x) * 1000 / DVD_TIME_BASE))
// 将播放时间转为毫秒：pts_us * 1000 / 1000000 = pts_ms

#define DVD_SEC_TO_TIME(x)  ((double)(x) * DVD_TIME_BASE)
// 将秒转为播放时间：seconds * 1000000 = microseconds

#define DVD_MSEC_TO_TIME(x) ((double)(x) * DVD_TIME_BASE / 1000)
// 将毫秒转为播放时间：ms * 1000000 / 1000 = ms * 1000

// 播放速度常量
#define DVD_PLAYSPEED_PAUSE  0     // 暂停状态（帧步进模式）
#define DVD_PLAYSPEED_NORMAL 1000  // 正常速度（1.0x = 1000）
```

### DVD_TIME_BASE 的设计考量

为什么选择 1,000,000（微秒）而不是毫秒或纳秒？这是因为：

1. **与 FFmpeg 一致**：FFmpeg 的 `AV_TIME_BASE` 也是 1,000,000，避免了额外转换
2. **精度足够**：1μs 的精度对于 60fps 视频（每帧 16,667μs）绰绰有余
3. **int64_t 范围充足**：64位整数可以表示约 292,471 年的微秒时间，不会溢出

```cpp
// 实际使用示例：时间单位转换

#include <cstdio>
#include <cstdint>

#define DVD_TIME_BASE 1000000
#define DVD_NOPTS_VALUE 0xFFF0000000000000
#define DVD_TIME_TO_MSEC(x) ((int)((double)(x) * 1000 / DVD_TIME_BASE))
#define DVD_SEC_TO_TIME(x)  ((double)(x) * DVD_TIME_BASE)
#define DVD_MSEC_TO_TIME(x) ((double)(x) * DVD_TIME_BASE / 1000)

int main() {
    // 1. 秒 -> 播放时间
    double pts_5sec = DVD_SEC_TO_TIME(5.0);
    printf("5 秒 = %.0f DVD_TIME_BASE 单位\n", pts_5sec);
    // 输出：5 秒 = 5000000 DVD_TIME_BASE 单位

    // 2. 播放时间 -> 毫秒
    int ms = DVD_TIME_TO_MSEC(pts_5sec);
    printf("5000000 单位 = %d 毫秒\n", ms);
    // 输出：5000000 单位 = 5000 毫秒

    // 3. 毫秒 -> 播放时间
    double pts_100ms = DVD_MSEC_TO_TIME(100);
    printf("100 毫秒 = %.0f DVD_TIME_BASE 单位\n", pts_100ms);
    // 输出：100 毫秒 = 100000 DVD_TIME_BASE 单位

    // 4. 检查无效时间戳
    double pts = DVD_NOPTS_VALUE;
    if (pts == DVD_NOPTS_VALUE) {
        printf("时间戳无效（哨兵值）\n");
    }
    // 输出：时间戳无效（哨兵值）

    // 5. 帧间隔计算
    double fps = 23.976;
    double frame_interval = DVD_SEC_TO_TIME(1.0 / fps);
    printf("23.976fps 帧间隔 = %.2f μs (%.2f ms)\n",
           frame_interval, frame_interval / 1000.0);
    // 输出：23.976fps 帧间隔 = 41708.38 μs (41.71 ms)

    return 0;
}
```

### DVD_NOPTS_VALUE 哨兵值

`DVD_NOPTS_VALUE`（值为 `0xFFF0000000000000`，即十进制约 1.1529 × 10^18）是一个特殊的标记值，表示"此时间戳无效/未设置"。它广泛用于：

- 解复用器返回的 DemuxPacket 中，当某个包没有 PTS/DTS 时
- 音频解码帧没有时间戳时的默认值
- 条件判断中区分"有效 PTS"和"无 PTS"

```cpp
// 源码模式：DVD_NOPTS_VALUE 的典型用法
// 来自 VideoPlayerAudio.cpp 第 361-366 行

if (audioframe.pts == DVD_NOPTS_VALUE) {
    // 解码器没有产出时间戳，使用音频时钟估算
    audioframe.pts = m_audioClock;
    audioframe.hasTimestamp = false;
} else {
    // 有有效时间戳，更新音频时钟
    m_audioClock = audioframe.pts;
}
```

> ⚠️ **常见错误**：不要将 DVD_NOPTS_VALUE 与 0 混淆。PTS=0 是合法的（表示文件开头），而 DVD_NOPTS_VALUE 表示"不存在时间戳"。在比较时必须用 `==` 精确比较，不能用 `<` 或 `>`。

## CDVDClock 成员变量详解

理解 CDVDClock 的核心在于理解它的成员变量。每个变量都参与时间映射公式的计算：

```cpp
// 源码位置：cpp/core/movie/ffmpeg/Clock.h 第 73-94 行

class CDVDClock {
protected:
    // ========== 播放时间计算相关 ==========
    CCriticalSection m_critSection;    // 主锁：保护 GetClock/SetSpeed/Pause 等操作
    int64_t m_systemUsed;             // 当前使用的系统频率（用于速度缩放）
                                       // 正常速度时 = m_systemFrequency
                                       // 2x 速度时 = m_systemFrequency / 2
    int64_t m_startClock;             // 播放起始时的系统时间戳
    int64_t m_pauseClock;             // 暂停时刻的系统时间戳（0 = 未暂停）
    double  m_iDisc;                  // Discontinuity 偏移量（时钟跳变后的补偿值）
    bool    m_bReset;                 // 重置标志：为 true 时下次 GetClock 会重新初始化
    bool    m_paused;                 // 暂停状态标志
    int     m_speedAfterPause;        // 暂停前记录的速度，恢复时使用

    // ========== 系统时钟源 ==========
    std::unique_ptr<CVideoReferenceClock> m_videoRefClock;  // 视频参考时钟
    int64_t m_systemFrequency;        // 系统时钟频率（每秒 tick 数）
    int64_t m_systemOffset;           // 系统时钟偏移（构造时记录的起始时间）
    CCriticalSection m_systemsection; // 系统时钟锁

    // ========== 速度微调 ==========
    int64_t m_systemAdjust;           // 累积的速度微调量（SpeedAdjust 积分结果）
    int64_t m_lastSystemTime;         // 上次 GetClock 时的系统时间
    double  m_speedAdjust;            // 速度微调系数（由音频同步设置）
    double  m_vSyncAdjust;            // VSync 调整量
    double  m_frameTime;              // 当前帧间隔（DVD_TIME_BASE / fps）

    // ========== 帧率匹配 ==========
    double  m_maxspeedadjust;         // 最大允许的速度调整比例
    CCriticalSection m_speedsection;  // 速度调整锁
};
```

### 成员变量之间的关系

```
系统时钟层：
  m_videoRefClock.GetTime() → 原始系统 tick
  m_systemFrequency        → 每秒多少 tick
  m_systemOffset            → 构造时的偏移基准

时间映射层：
  m_startClock    → 播放开始时的 tick（或最近一次 Discontinuity 时的 tick）
  m_systemUsed    → 缩放后的频率（决定播放速度）
  m_systemAdjust  → 音频同步的微调积分
  m_iDisc         → Discontinuity 后的时间偏移

速度控制层：
  m_speedAdjust   → 每 tick 的微调量（由音频 SYNC_RESAMPLE 模式设置）
  m_pauseClock    → 暂停时冻结的 tick 值
  m_paused        → 暂停标志
```

## SystemToPlaying() —— 核心时间映射公式

`SystemToPlaying()` 是 CDVDClock 最核心的方法。它将操作系统的物理时间（system tick）转换为播放时间（以 DVD_TIME_BASE 为单位的微秒）：

```cpp
// 源码位置：cpp/core/movie/ffmpeg/Clock.cpp 第 244-267 行

double CDVDClock::SystemToPlaying(int64_t system) {
    int64_t current;

    // 首次调用或 Reset 后：初始化所有状态
    if (m_bReset) {
        m_startClock = system;          // 以当前时间为起点
        m_systemUsed = m_systemFrequency; // 恢复正常速度
        if (m_pauseClock)
            m_pauseClock = m_startClock;
        m_iDisc = 0;                    // 清除时钟跳变偏移
        m_systemAdjust = 0;             // 清除微调积分
        m_speedAdjust = 0;              // 清除微调速率
        m_vSyncAdjust = 0;
        m_bReset = false;
    }

    // 暂停时使用冻结的时间
    if (m_pauseClock)
        current = m_pauseClock;         // 时间停止流逝
    else
        current = system;               // 正常使用当前系统时间

    // 核心公式：
    // playing_time = (elapsed_ticks + adjust) / scaled_frequency * TIME_BASE + disc_offset
    return DVD_TIME_BASE * (double)(current - m_startClock + m_systemAdjust)
           / m_systemUsed
           + m_iDisc;
}
```

### 公式拆解

核心公式可以分解为四个部分：

```
playing_time = DVD_TIME_BASE × (current - m_startClock + m_systemAdjust) / m_systemUsed + m_iDisc
               ─────────────   ─────────────────────────────────────────   ────────────   ───────
                   ①                              ②                            ③           ④

① DVD_TIME_BASE = 1,000,000：将比值转换为微秒单位
② current - m_startClock + m_systemAdjust：已流逝的 tick 数（含微调）
③ m_systemUsed：缩放后的频率（控制播放速度）
④ m_iDisc：Discontinuity 偏移（时钟跳变补偿）
```

**数学本质**：`elapsed_ticks / frequency = elapsed_seconds`，然后乘以 `DVD_TIME_BASE` 转微秒。通过改变 `m_systemUsed`（分母），可以加速或减速时间流逝。

```cpp
// 示例：理解 SystemToPlaying 公式

#include <cstdio>
#include <cstdint>

#define DVD_TIME_BASE 1000000
#define DVD_PLAYSPEED_NORMAL 1000

int main() {
    // 模拟 CDVDClock 内部状态
    int64_t systemFrequency = 1000000;  // 假设系统频率 1MHz
    int64_t startClock = 0;
    int64_t systemAdjust = 0;
    double  iDisc = 0;

    // ===== 场景 1：正常速度播放 1 秒 =====
    int64_t systemUsed = systemFrequency;  // 正常速度
    int64_t current = 1000000;             // 1 秒后的 tick

    double playing = DVD_TIME_BASE * (double)(current - startClock + systemAdjust)
                     / systemUsed + iDisc;
    printf("正常 1x: playing = %.0f μs (%.3f 秒)\n", playing, playing / DVD_TIME_BASE);
    // 输出：正常 1x: playing = 1000000 μs (1.000 秒)

    // ===== 场景 2：2 倍速播放 1 秒 =====
    systemUsed = systemFrequency * DVD_PLAYSPEED_NORMAL / 2000;  // 2x 速度
    // systemUsed = 1000000 * 1000 / 2000 = 500000

    playing = DVD_TIME_BASE * (double)(current - startClock + systemAdjust)
              / systemUsed + iDisc;
    printf("2x 速度: playing = %.0f μs (%.3f 秒)\n", playing, playing / DVD_TIME_BASE);
    // 输出：2x 速度: playing = 2000000 μs (2.000 秒) — 物理 1 秒 = 播放 2 秒

    // ===== 场景 3：0.5 倍速播放 1 秒 =====
    systemUsed = systemFrequency * DVD_PLAYSPEED_NORMAL / 500;  // 0.5x 速度
    // systemUsed = 1000000 * 1000 / 500 = 2000000

    playing = DVD_TIME_BASE * (double)(current - startClock + systemAdjust)
              / systemUsed + iDisc;
    printf("0.5x 速度: playing = %.0f μs (%.3f 秒)\n", playing, playing / DVD_TIME_BASE);
    // 输出：0.5x 速度: playing = 500000 μs (0.500 秒) — 物理 1 秒 = 播放 0.5 秒

    // ===== 场景 4：Discontinuity 偏移 =====
    systemUsed = systemFrequency;
    iDisc = 5.0 * DVD_TIME_BASE;  // 跳到 5 秒位置
    current = 500000;              // 跳转后又过了 0.5 秒

    playing = DVD_TIME_BASE * (double)(current - startClock + systemAdjust)
              / systemUsed + iDisc;
    printf("Disc 跳转: playing = %.0f μs (%.3f 秒)\n", playing, playing / DVD_TIME_BASE);
    // 输出：Disc 跳转: playing = 5500000 μs (5.500 秒)

    return 0;
}
```

## SetSpeed() —— 速度控制实现

`SetSpeed()` 通过修改 `m_systemUsed`（缩放频率）来改变播放速度，同时保证速度切换瞬间时钟不跳变：

```cpp
// 源码位置：cpp/core/movie/ffmpeg/Clock.cpp 第 91-119 行

void CDVDClock::SetSpeed(int iSpeed) {
    CSingleLock lock(m_critSection);

    // 暂停状态下，只记录恢复后的速度
    if (m_paused) {
        m_speedAfterPause = iSpeed;
        return;
    }

    // 进入暂停：记录暂停时刻
    if (iSpeed == DVD_PLAYSPEED_PAUSE) {
        if (!m_pauseClock)
            m_pauseClock = m_videoRefClock->GetTime();  // 冻结时间
        return;
    }

    int64_t current;
    // 核心：计算新的缩放频率
    // 速度 1000 (1x) → newfreq = systemFrequency
    // 速度 2000 (2x) → newfreq = systemFrequency / 2（时间流逝更快）
    // 速度  500 (0.5x) → newfreq = systemFrequency * 2（时间流逝更慢）
    int64_t newfreq = m_systemFrequency * DVD_PLAYSPEED_NORMAL / iSpeed;

    current = m_videoRefClock->GetTime();
    // 从暂停恢复时，补偿暂停期间流逝的系统时间
    if (m_pauseClock) {
        m_startClock += current - m_pauseClock;
        m_pauseClock = 0;
    }

    // 关键：重新计算 m_startClock 使得速度切换瞬间 playing_time 不变
    // 推导：
    //   old_playing = (current - old_start) * TIME_BASE / old_freq + iDisc
    //   new_playing = (current - new_start) * TIME_BASE / new_freq + iDisc
    //   要求 old_playing == new_playing，解出 new_start：
    //   new_start = current - (current - old_start) * new_freq / old_freq
    m_startClock = current
        - (int64_t)((double)(current - m_startClock) * newfreq / m_systemUsed);
    m_systemUsed = newfreq;
}
```

### 速度切换的连续性保证

速度切换时最关键的问题是：**不能让播放时间发生跳变**。如果直接修改 `m_systemUsed` 而不调整 `m_startClock`，那么 `SystemToPlaying()` 的返回值会瞬间改变，导致画面卡顿或跳帧。

```cpp
// 示例：演示速度切换的连续性

#include <cstdio>
#include <cstdint>

#define DVD_TIME_BASE 1000000
#define DVD_PLAYSPEED_NORMAL 1000

// 简化版 SystemToPlaying
double SystemToPlaying(int64_t current, int64_t startClock,
                       int64_t systemUsed, double iDisc) {
    return DVD_TIME_BASE * (double)(current - startClock) / systemUsed + iDisc;
}

int main() {
    int64_t systemFrequency = 1000000;
    int64_t startClock = 0;
    int64_t systemUsed = systemFrequency;  // 1x 速度
    double iDisc = 0;

    // 播放 2 秒后，切换到 2x 速度
    int64_t current = 2000000;  // 2 秒
    double before = SystemToPlaying(current, startClock, systemUsed, iDisc);
    printf("切换前: playing = %.3f 秒\n", before / DVD_TIME_BASE);
    // 输出：切换前: playing = 2.000 秒

    // 计算新频率
    int64_t newfreq = systemFrequency * DVD_PLAYSPEED_NORMAL / 2000;  // 2x = 500000

    // 关键：调整 startClock 保持连续
    startClock = current
        - (int64_t)((double)(current - startClock) * newfreq / systemUsed);
    systemUsed = newfreq;

    double after = SystemToPlaying(current, startClock, systemUsed, iDisc);
    printf("切换后: playing = %.3f 秒\n", after / DVD_TIME_BASE);
    // 输出：切换后: playing = 2.000 秒 — 连续！无跳变

    // 又过了 1 物理秒
    current += 1000000;
    double later = SystemToPlaying(current, startClock, systemUsed, iDisc);
    printf("1秒后:  playing = %.3f 秒\n", later / DVD_TIME_BASE);
    // 输出：1秒后:  playing = 4.000 秒 — 2x 速度：物理 1 秒 = 播放 2 秒

    return 0;
}
```

## Pause() 与暂停/恢复机制

暂停的实现非常精巧：通过冻结 `m_pauseClock` 来阻止时间流逝，恢复时通过 `SetSpeed()` 中的补偿逻辑跳过暂停期间的物理时间。

```cpp
// 源码位置：cpp/core/movie/ffmpeg/Clock.cpp 第 73-89 行

void CDVDClock::Pause(bool pause) {
    CSingleLock lock(m_critSection);

    if (pause && !m_paused) {
        // 进入暂停
        if (!m_pauseClock)
            // 记录暂停前的速度，以便恢复
            m_speedAfterPause =
                m_systemFrequency * DVD_PLAYSPEED_NORMAL / m_systemUsed;
        else
            // 已经处于帧步进模式（speed=0），恢复后仍然暂停
            m_speedAfterPause = DVD_PLAYSPEED_PAUSE;

        SetSpeed(DVD_PLAYSPEED_PAUSE);  // 冻结时钟
        m_paused = true;
    } else if (!pause && m_paused) {
        // 恢复播放
        m_paused = false;
        SetSpeed(m_speedAfterPause);    // 恢复之前的速度
    }
}
```

### 暂停/恢复的时序图

```
时间线：
  t0        t1(暂停)     t2(暂停中)    t3(恢复)     t4
  │           │             │            │            │
  ├───────────┤             │            ├────────────┤
  │ 正常播放   │             │            │ 继续播放    │
  │           │─────────────│────────────│            │
  │           │  m_pauseClock = t1 时的 tick           │
  │           │  SystemToPlaying 冻结在 t1 的值        │
  │           │                          │            │
  │           │                   SetSpeed() 执行：    │
  │           │                   m_startClock += (t3 - t1)
  │           │                   相当于"跳过"暂停期间  │

播放时间：
  0s ──────── 5s                         5s ──────── 10s
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^
              暂停期间播放时间不动
```

关键细节：`SetSpeed()` 中当 `m_pauseClock != 0` 时，会执行 `m_startClock += current - m_pauseClock`。这行代码的效果是将 `m_startClock` 向前推进暂停期间流逝的物理时间，使得恢复后 `SystemToPlaying()` 从暂停点继续计算，而不是出现一个巨大的时间跳变。

## GetClock() —— 带微调的时间查询

`GetClock()` 是所有线程查询播放时间的入口，它在 `SystemToPlaying()` 的基础上增加了 `m_speedAdjust` 微调积分：

```cpp
// 源码位置：cpp/core/movie/ffmpeg/Clock.cpp 第 41-49 行

double CDVDClock::GetClock(bool interpolated /*= true*/) {
    CSingleLock lock(m_critSection);

    int64_t current = m_videoRefClock->GetTime(interpolated);
    // 累积速度微调：每个 tick 增加 speedAdjust 的调整量
    m_systemAdjust += m_speedAdjust * (current - m_lastSystemTime);
    m_lastSystemTime = current;

    return SystemToPlaying(current);
}
```

### SpeedAdjust 微调机制

`m_speedAdjust` 是一个非常小的浮点数（通常在 ±0.01 范围内），由音频同步的 SYNC_RESAMPLE 模式设置。它的作用是对时钟进行微幅加速或减速，使音频采样率与播放时钟精确匹配。

每次调用 `GetClock()` 时：
1. 计算距上次调用的 tick 差：`delta = current - m_lastSystemTime`
2. 累积微调量：`m_systemAdjust += m_speedAdjust * delta`
3. 这个微调量参与 `SystemToPlaying()` 的计算

```cpp
// 示例：SpeedAdjust 的效果

#include <cstdio>
#include <cstdint>

#define DVD_TIME_BASE 1000000

int main() {
    // 模拟 GetClock 连续调用
    int64_t systemFrequency = 1000000;
    int64_t startClock = 0;
    int64_t systemUsed = systemFrequency;
    int64_t systemAdjust = 0;
    int64_t lastSystemTime = 0;
    double speedAdjust = 0.001;  // 0.1% 加速
    double iDisc = 0;

    for (int i = 1; i <= 5; i++) {
        int64_t current = i * 1000000;  // 每秒采样一次

        // 累积微调
        systemAdjust += (int64_t)(speedAdjust * (current - lastSystemTime));
        lastSystemTime = current;

        // 计算播放时间
        double playing = DVD_TIME_BASE
            * (double)(current - startClock + systemAdjust)
            / systemUsed + iDisc;

        printf("物理 %d 秒 → 播放 %.6f 秒 (微调累积: %lld ticks)\n",
               i, playing / DVD_TIME_BASE, (long long)systemAdjust);
    }
    // 输出：
    // 物理 1 秒 → 播放 1.001000 秒 (微调累积: 1000 ticks)
    // 物理 2 秒 → 播放 2.002000 秒 (微调累积: 2000 ticks)
    // 物理 3 秒 → 播放 3.003000 秒 (微调累积: 3000 ticks)
    // 物理 4 秒 → 播放 4.004000 秒 (微调累积: 4000 ticks)
    // 物理 5 秒 → 播放 5.005000 秒 (微调累积: 5000 ticks)
    // → 0.1% 的加速，5 秒后播放时间快了 5ms

    return 0;
}
```

## GetClockSpeed() —— 当前实际播放速率

`GetClockSpeed()` 返回当前时钟相对于正常速度的比率，包含了基础速度和微调：

```cpp
// 源码位置：cpp/core/movie/ffmpeg/Clock.cpp 第 269-274 行

double CDVDClock::GetClockSpeed() {
    CSingleLock lock(m_critSection);

    // 基础速度：systemFrequency / systemUsed
    // 正常时 = 1.0，2x 时 = 2.0，0.5x 时 = 0.5
    double speed = (double)m_systemFrequency / m_systemUsed;

    // 叠加视频参考时钟的速度和微调
    return m_videoRefClock->GetSpeed() * speed + m_speedAdjust;
}
```

## CVideoReferenceClock —— 系统时钟源

`CVideoReferenceClock` 是 CDVDClock 底层的时钟源，负责提供原始的系统时间：

```cpp
// 源码位置：cpp/core/movie/ffmpeg/VideoReferenceClock.h 第 7-55 行

class CVideoReferenceClock /*: public CThread*/ {
public:
    int64_t GetTime(bool interpolated = true);
    // 获取当前系统时间（tick 单位）
    // interpolated=true 时做插值平滑（减少抖动）

    void SetSpeed(double Speed);
    // 设置时钟速度（用于帧率匹配）

    double GetSpeed();
    // 获取当前速度

    double GetRefreshRate(double* interval = nullptr);
    // 获取显示器刷新率（用于 VSync 匹配）
    // KrKr2 中此功能基本未启用

private:
    int64_t  m_CurrTime;       // 当前时钟值
    int64_t  m_LastIntTime;    // 上次插值时间（防止时钟倒退）
    double   m_CurrTimeFract;  // 舍入累积的小数部分
    double   m_ClockSpeed;     // 播放器设置的时钟速度
    int64_t  m_ClockOffset;    // VBlank 时钟与系统时钟的偏差
    int64_t  m_SystemFrequency;// 系统频率
    bool     m_UseVblank;      // 是否使用 VBlank 作为时钟源
    double   m_RefreshRate;    // 刷新率
};
```

> 💡 **KrKr2 特殊情况**：原版 Kodi 中 `CVideoReferenceClock` 是一个独立线程（继承 CThread），用于追踪 VBlank 信号实现精确帧同步。但在 KrKr2 中，`CThread` 继承被注释掉了（`/*: public CThread*/`），VSync 追踪功能未启用。时钟源回退为普通的 `CurrentHostCounter()` 系统计时器。

### 跨平台时钟源差异

`CurrentHostFrequency()` 和 `CurrentHostCounter()` 在不同平台上使用不同的底层 API：

| 平台 | 时钟函数 | 精度 | 说明 |
|------|---------|------|------|
| **Windows** | `QueryPerformanceCounter` / `QueryPerformanceFrequency` | ~100ns | 高精度计时器，所有现代 Windows 版本均支持 |
| **Linux** | `clock_gettime(CLOCK_MONOTONIC)` | ~1ns | 单调递增时钟，不受系统时间调整影响 |
| **macOS** | `mach_absolute_time()` + `mach_timebase_info` | ~1ns | Mach 内核提供的高精度计时器 |
| **Android** | `clock_gettime(CLOCK_MONOTONIC)` | ~1ns | 与 Linux 相同（基于 Linux 内核） |

```cpp
// 各平台时钟源实现概览

// Windows:
int64_t CurrentHostCounter() {
    LARGE_INTEGER counter;
    QueryPerformanceCounter(&counter);
    return counter.QuadPart;
}
int64_t CurrentHostFrequency() {
    LARGE_INTEGER freq;
    QueryPerformanceFrequency(&freq);
    return freq.QuadPart;  // 通常为 10,000,000 (10MHz)
}

// Linux / Android:
int64_t CurrentHostCounter() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return (int64_t)ts.tv_sec * 1000000000LL + ts.tv_nsec;
}
int64_t CurrentHostFrequency() {
    return 1000000000LL;  // 纳秒精度
}

// macOS:
int64_t CurrentHostCounter() {
    return mach_absolute_time();
}
int64_t CurrentHostFrequency() {
    mach_timebase_info_data_t info;
    mach_timebase_info(&info);
    return 1000000000LL * info.denom / info.numer;
}
```

> ⚠️ **常见错误 1**：在 Windows 上误用 `GetTickCount()` 或 `timeGetTime()` 作为时钟源。这些函数精度只有 ~15ms，完全不够音视频同步使用。必须用 `QueryPerformanceCounter`。

> ⚠️ **常见错误 2**：在 Linux 上使用 `CLOCK_REALTIME` 而非 `CLOCK_MONOTONIC`。`CLOCK_REALTIME` 会被 NTP 调整，可能导致时钟倒退。音视频同步必须使用单调递增时钟。

## 动手实践

### 实验 1：模拟完整的时钟生命周期

```cpp
// 完整的 CDVDClock 模拟器 — 体验时钟的初始化、播放、暂停、变速、恢复

#include <cstdio>
#include <cstdint>
#include <thread>
#include <chrono>

#define DVD_TIME_BASE 1000000
#define DVD_PLAYSPEED_PAUSE 0
#define DVD_PLAYSPEED_NORMAL 1000

struct SimpleClock {
    int64_t systemFrequency;
    int64_t systemUsed;
    int64_t startClock;
    int64_t pauseClock;
    int64_t systemAdjust;
    double  iDisc;
    bool    bReset;

    SimpleClock() : systemFrequency(1000000), systemUsed(1000000),
                    startClock(0), pauseClock(0), systemAdjust(0),
                    iDisc(0), bReset(true) {}

    int64_t GetSystemTime() {
        auto now = std::chrono::steady_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(
            now.time_since_epoch());
        return us.count();
    }

    double GetPlayingTime() {
        int64_t current = GetSystemTime();
        if (bReset) {
            startClock = current;
            systemUsed = systemFrequency;
            pauseClock = 0;
            iDisc = 0;
            systemAdjust = 0;
            bReset = false;
        }
        int64_t effective = pauseClock ? pauseClock : current;
        return DVD_TIME_BASE * (double)(effective - startClock + systemAdjust)
               / systemUsed + iDisc;
    }

    void SetSpeed(int speed) {
        if (speed == DVD_PLAYSPEED_PAUSE) {
            if (!pauseClock)
                pauseClock = GetSystemTime();
            return;
        }
        int64_t current = GetSystemTime();
        int64_t newfreq = systemFrequency * DVD_PLAYSPEED_NORMAL / speed;
        if (pauseClock) {
            startClock += current - pauseClock;
            pauseClock = 0;
        }
        startClock = current
            - (int64_t)((double)(current - startClock) * newfreq / systemUsed);
        systemUsed = newfreq;
    }

    void Discontinuity(double clock) {
        startClock = GetSystemTime();
        iDisc = clock;
        systemAdjust = 0;
    }
};

int main() {
    SimpleClock clock;

    printf("=== 阶段 1：正常播放 2 秒 ===\n");
    for (int i = 0; i < 4; i++) {
        double pt = clock.GetPlayingTime();
        printf("  播放时间: %.3f 秒\n", pt / DVD_TIME_BASE);
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }

    printf("\n=== 阶段 2：暂停 1 秒 ===\n");
    clock.SetSpeed(DVD_PLAYSPEED_PAUSE);
    for (int i = 0; i < 2; i++) {
        double pt = clock.GetPlayingTime();
        printf("  播放时间: %.3f 秒 (暂停中)\n", pt / DVD_TIME_BASE);
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }

    printf("\n=== 阶段 3：恢复 + 2x 速度 ===\n");
    clock.SetSpeed(2000);  // 2x 速度
    for (int i = 0; i < 4; i++) {
        double pt = clock.GetPlayingTime();
        printf("  播放时间: %.3f 秒\n", pt / DVD_TIME_BASE);
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }

    printf("\n=== 阶段 4：Discontinuity 跳转到 30 秒 ===\n");
    clock.Discontinuity(30.0 * DVD_TIME_BASE);
    for (int i = 0; i < 4; i++) {
        double pt = clock.GetPlayingTime();
        printf("  播放时间: %.3f 秒\n", pt / DVD_TIME_BASE);
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }

    return 0;
}
```

编译命令（四平台通用）：

```bash
# Windows (MSVC)
cl /EHsc /std:c++17 clock_sim.cpp /Fe:clock_sim.exe

# Linux / macOS
g++ -std=c++17 -o clock_sim clock_sim.cpp -lpthread

# Android (NDK)
$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android30-clang++ \
    -std=c++17 -o clock_sim clock_sim.cpp
```

## 对照项目源码

理解 CDVDClock 在项目中的使用方式：

相关文件：
- `cpp/core/movie/ffmpeg/Clock.h` 第 18-95 行 — CDVDClock 完整声明，所有成员变量和方法
- `cpp/core/movie/ffmpeg/Clock.cpp` 第 1-276 行 — CDVDClock 完整实现（276 行）
- `cpp/core/movie/ffmpeg/VideoReferenceClock.h` 第 7-55 行 — 系统时钟源（VBlank 被禁用）
- `cpp/core/movie/ffmpeg/VideoPlayerVideo.cpp` 第 771 行 — 视频线程调用 `UpdateFramerate()` 通知帧率
- `cpp/core/movie/ffmpeg/VideoPlayerVideo.cpp` 第 778 行 — 视频线程调用 `GetClock()` 获取播放时间做同步
- `cpp/core/movie/ffmpeg/VideoPlayerAudio.cpp` 第 525-538 行 — 音频线程通过 `ErrorAdjust()` 校正同步误差
- `cpp/core/movie/ffmpeg/TimeUtils.h` 第 10-12 行 — `CurrentHostCounter()` 和 `CurrentHostFrequency()` 声明

## 本节小结

- **CDVDClock** 是整个视频播放器的时间核心，继承自 Kodi/XBMC，提供统一的播放时间基准
- **DVD_TIME_BASE**（1,000,000）是微秒单位，与 FFmpeg 的 AV_TIME_BASE 一致
- **DVD_NOPTS_VALUE** 是无效时间戳哨兵值，必须用 `==` 精确比较
- **SystemToPlaying()** 是核心公式：`TIME_BASE × (elapsed + adjust) / scaledFreq + disc`
- **SetSpeed()** 通过修改 `m_systemUsed`（缩放频率）实现变速，同时调整 `m_startClock` 保证连续性
- **Pause()** 通过冻结 `m_pauseClock` 停止时间流逝，恢复时跳过暂停期间
- **SpeedAdjust** 是微幅调整机制，由音频 SYNC_RESAMPLE 模式驱动
- **CVideoReferenceClock** 在 KrKr2 中退化为简单系统计时器（VBlank 追踪被禁用）
- 跨平台时钟源：Windows 用 QPC，Linux/Android 用 CLOCK_MONOTONIC，macOS 用 mach_absolute_time

## 练习题与答案

### 题目 1：如果 DVD_PLAYSPEED_NORMAL 改为 10000（而非 1000），SetSpeed 的计算需要怎样修改？

<details>
<summary>查看答案</summary>

`SetSpeed()` 中计算新频率的公式为：

```cpp
int64_t newfreq = m_systemFrequency * DVD_PLAYSPEED_NORMAL / iSpeed;
```

如果 `DVD_PLAYSPEED_NORMAL` 改为 10000，那么：
- 正常速度 1x 对应 `iSpeed = 10000`
- 2x 速度对应 `iSpeed = 20000`
- 0.5x 速度对应 `iSpeed = 5000`

公式本身不需要修改——`DVD_PLAYSPEED_NORMAL / iSpeed` 的比值仍然代表速度因子的倒数。只需要修改调用方传入的 `iSpeed` 值即可。

精度影响：更大的 `DVD_PLAYSPEED_NORMAL` 允许更细粒度的速度控制。原来 1000 只能以 0.1% 为步长调整，改为 10000 后可以以 0.01% 为步长调整。这在音频重采样同步中可能有用。

</details>

### 题目 2：为什么暂停时 SystemToPlaying() 使用 m_pauseClock 而不是直接返回上一次的结果？

<details>
<summary>查看答案</summary>

使用 `m_pauseClock` 而非缓存上一次结果有三个原因：

1. **线程安全简洁性**：`SystemToPlaying()` 被多个线程同时调用（音频线程、视频线程、主线程）。如果缓存结果，需要额外的同步来保证所有线程看到同一个缓存值。而使用 `m_pauseClock` 只需在暂停时设置一次，之后每次调用自然计算出相同结果。

2. **systemAdjust 仍在变化**：即使暂停了，`GetClock()` 中的 `m_systemAdjust` 仍然可能被修改（虽然在暂停时通常不会）。使用实时计算确保这些调整不会丢失。

3. **Discontinuity 兼容**：暂停期间可能发生 `Discontinuity()`（例如用户拖动进度条）。使用公式计算可以正确反映 `m_iDisc` 的变化，而缓存值则不行。

```cpp
// 对比两种方式：
// 方式 A（项目采用）：冻结 current = m_pauseClock
double playing = DVD_TIME_BASE * (current - m_startClock + m_systemAdjust)
                 / m_systemUsed + m_iDisc;
// → 如果暂停期间 m_iDisc 改变了，结果会反映新的偏移

// 方式 B（缓存）：
double playing = m_cachedPlayingTime;
// → 即使 m_iDisc 改了，返回的还是旧值
```

</details>

### 题目 3：编写一个函数，给定当前的 CDVDClock 状态，计算需要等待多少毫秒才能到达目标 PTS。

<details>
<summary>查看答案</summary>

```cpp
#include <cstdio>
#include <cstdint>
#include <algorithm>

#define DVD_TIME_BASE 1000000
#define DVD_TIME_TO_MSEC(x) ((int)((double)(x) * 1000 / DVD_TIME_BASE))
#define DVD_PLAYSPEED_NORMAL 1000

// 计算到达目标 PTS 还需等待的毫秒数
// 返回值：正数 = 需要等待，0 = 已到达或已过，-1 = 时钟暂停无法计算
int CalcWaitTime(double targetPts,
                 double currentPlaying,  // 当前 GetClock() 返回值
                 int speed) {            // 当前播放速度
    if (speed == 0) {
        return -1;  // 暂停中，无法到达目标
    }

    double diff = targetPts - currentPlaying;
    if (diff <= 0) {
        return 0;   // 已到达或已过
    }

    // 考虑播放速度：实际等待时间 = 播放时间差 / 速度因子
    // 速度因子 = speed / NORMAL
    double speedFactor = (double)speed / DVD_PLAYSPEED_NORMAL;
    double waitUs = diff / speedFactor;  // 微秒

    return std::max(1, DVD_TIME_TO_MSEC(waitUs));
}

int main() {
    // 场景 1：正常速度，距目标 100ms
    printf("1x, 100ms ahead: wait %d ms\n",
           CalcWaitTime(1100000, 1000000, 1000));
    // 输出：1x, 100ms ahead: wait 100 ms

    // 场景 2：2x 速度，距目标 100ms（物理只需等 50ms）
    printf("2x, 100ms ahead: wait %d ms\n",
           CalcWaitTime(1100000, 1000000, 2000));
    // 输出：2x, 100ms ahead: wait 50 ms

    // 场景 3：0.5x 速度，距目标 100ms（物理需等 200ms）
    printf("0.5x, 100ms ahead: wait %d ms\n",
           CalcWaitTime(1100000, 1000000, 500));
    // 输出：0.5x, 100ms ahead: wait 200 ms

    // 场景 4：已过目标
    printf("past target: wait %d ms\n",
           CalcWaitTime(900000, 1000000, 1000));
    // 输出：past target: wait 0 ms

    // 场景 5：暂停中
    printf("paused: wait %d ms\n",
           CalcWaitTime(1100000, 1000000, 0));
    // 输出：paused: wait -1 ms

    return 0;
}
```

</details>

## 下一步

[PTS 校正与时间戳转换](./02-PTS校正与时间戳转换.md) —— 深入分析 ErrorAdjust() 的误差校正机制、Discontinuity() 的时钟跳变处理，以及 ConvertTimestamp() 如何将 FFmpeg 时间戳转换为 DVD 播放时间。

