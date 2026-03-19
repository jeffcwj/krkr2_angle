# CDVDClock 时钟系统与音频同步

> **所属模块：** P06-FFmpeg音视频开发
> **前置知识：** [同步基础与时间戳](./01-同步基础与时间戳.md)
> **预计阅读时间：** 40 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 CDVDClock 的三层时间模型（系统时间→绝对时间→播放时间）及其数学关系
2. 解释播放速度变化、暂停、seek 跳变时时钟内部状态如何切换
3. 说明 VSync（垂直同步，显示器刷新与画面输出对齐的机制）模式和非 VSync 模式下误差校正的差异
4. 描述 CVideoPlayerAudio 的三状态同步状态机（STARTING→WAITSYNC→INSYNC）的工作流程
5. 自己实现一个支持暂停、变速、跳变的简化时钟类

---

## CDVDClock 架构总览

### 时钟在播放器中的位置

上一节我们了解了音视频同步的三种策略。KrKr2 默认使用 **SYNC_DISCON**（音频主时钟 + 不连续跳变修正）。而 `CDVDClock` 就是实现这个策略的核心组件——它是整个播放器的"心脏"，所有线程（视频解码线程、音频解码线程、主循环）都通过它获取当前的播放时间。

```
┌─────────────────────────────────────────────────────────────┐
│                      KrKr2 播放器                            │
│                                                              │
│  ┌──────────────────┐     ┌──────────────┐                   │
│  │ CVideoPlayerVideo │     │ CVideoPlayer- │                  │
│  │  （视频线程）       │     │  Audio（音频）│                  │
│  └────────┬─────────┘     └──────┬───────┘                   │
│           │ GetClock()           │ GetClock() / ErrorAdjust() │
│           │                      │                            │
│           ▼                      ▼                            │
│  ┌──────────────────────────────────────────┐                │
│  │              CDVDClock                     │                │
│  │  ┌─────────────────────────────────────┐  │                │
│  │  │  CVideoReferenceClock（系统时钟源）  │  │                │
│  │  │  提供单调递增的高精度系统时间         │  │                │
│  │  └─────────────────────────────────────┘  │                │
│  │                                            │                │
│  │  核心功能：                                 │                │
│  │  • SystemToAbsolute() — 系统→绝对时间      │                │
│  │  • SystemToPlaying() — 系统→播放时间       │                │
│  │  • SetSpeed()        — 变速/暂停控制       │                │
│  │  • ErrorAdjust()     — 同步误差修正        │                │
│  │  • Discontinuity()   — 时钟跳变           │                │
│  └──────────────────────────────────────────┘                │
│           ▲                                                   │
│           │ GetClock()                                        │
│  ┌────────┴─────────┐                                        │
│  │  BasePlayer 主循环 │                                       │
│  │  （控制线程）      │                                       │
│  └──────────────────┘                                        │
└─────────────────────────────────────────────────────────────┘
```

### 头文件定义

CDVDClock 定义在 `cpp/core/movie/ffmpeg/Clock.h`（97 行）。先看关键常量和成员变量：

```cpp
// 来源：Clock.h 第 15-20 行
// DVD_TIME_BASE 是 KrKr2 播放器内部的时间单位
// 1 秒 = 1,000,000 个 DVD_TIME_BASE 单位（即微秒精度）
#define DVD_TIME_BASE 1000000

// 时间转换宏：秒 ↔ DVD_TIME_BASE
#define DVD_SEC_TO_TIME(x)  ((double)(x) * DVD_TIME_BASE)
#define DVD_MSEC_TO_TIME(x) ((double)(x) * DVD_TIME_BASE / 1000.0)
#define DVD_TIME_TO_SEC(x)  ((double)(x) / DVD_TIME_BASE)
#define DVD_TIME_TO_MSEC(x) ((double)(x) * 1000.0 / DVD_TIME_BASE)

// 播放速度常量
// 速度值以 1000 为基准：1000 = 正常速度，500 = 半速，2000 = 双速
#define DVD_PLAYSPEED_PAUSE    0     // 暂停
#define DVD_PLAYSPEED_NORMAL   1000  // 1.0x 正常速度
```

```cpp
// 来源：Clock.h 第 30-65 行（简化展示）
class CDVDClock {
public:
    CDVDClock();

    // 获取当前播放时间，同时输出绝对时间
    double GetClock(double &absolute);
    double GetClock();

    // 时间坐标转换
    double SystemToAbsolute(int64_t system);  // 系统时间→绝对时间
    double SystemToPlaying(int64_t system);   // 系统时间→播放时间

    // 播放控制
    void SetSpeed(int iSpeed);     // 设置播放速度
    void Pause(bool pause);        // 暂停/恢复

    // 同步修正
    double ErrorAdjust(double error, const char *log);
    void Discontinuity(double clock, double absolute);

    // 帧率相关
    void UpdateFramerate(double fps, double *interval = nullptr);

    // 速度微调（用于长期漂移修正）
    void SetSpeedAdjust(double adjust);
    double GetSpeedAdjust();

private:
    CVideoReferenceClock *m_videoRefClock;  // 系统时钟源

    int64_t m_systemOffset;     // 系统时钟的初始偏移量
    int64_t m_systemFrequency;  // 系统时钟频率（每秒多少 tick）
    int64_t m_startClock;       // 本次播放段的起始系统时间
    int64_t m_pauseClock;       // 暂停时冻结的系统时间（0=未暂停）
    int64_t m_systemUsed;       // 有效频率（受播放速度影响）

    double m_iDisc;         // 不连续跳变量（Discontinuity 偏移）
    double m_systemAdjust;  // 累计微调量（由 GetClock 渐进累加）
    double m_speedAdjust;   // 速度微调系数（用于长期漂移修正）
    double m_frameTime;     // 一帧的时间长度（DVD_TIME_BASE 单位）
    double m_vSyncAdjust;   // VSync 校正量

    bool m_bReset;          // 是否需要重置状态
    int64_t m_lastSystemTime; // 上次 GetClock 调用的系统时间
    int m_speedAfterPause;    // 暂停前的播放速度（恢复时使用）
};
```

每个成员变量都在时钟的数学模型中有明确的角色。接下来我们逐层分析。

---

## 三层时间模型

CDVDClock 维护了三层相互关联的时间概念。理解这三层是理解整个时钟系统的关键：

| 层次 | 方法 | 含义 | 特征 |
|------|------|------|------|
| **系统时间** | `m_videoRefClock->GetTime()` | 操作系统提供的高精度时钟 | 单调递增，不可暂停，不可变速 |
| **绝对时间** | `SystemToAbsolute()` | 换算为 DVD_TIME_BASE 单位的时间 | 单调递增，单位统一，但不可暂停 |
| **播放时间** | `SystemToPlaying()` | 考虑暂停、变速、跳变后的实际播放位置 | 可暂停、可变速、可跳变 |

为什么需要三层？直觉上，一个简单的时钟只需要"当前时间"。但播放器面临的真实场景是复杂的：

- **暂停**：用户按暂停键，播放时间必须停止但系统时间不会停
- **变速**：用户选择 2 倍速，播放时间必须以 2 倍速率增长
- **seek 跳变**：用户拖动进度条，播放时间必须瞬间跳到新位置
- **误差修正**：音视频不同步时，需要微调播放时间来修正

三层模型将这些复杂性分层处理：

```
系统时间 ──[单位转换]──▶ 绝对时间 ──[暂停+变速+跳变]──▶ 播放时间
  │                        │                              │
  │ 纯硬件时钟              │ 统一单位                      │ 实际播放位置
  │ 不可控                  │ 不可控                        │ 完全可控
  └────────────────────────┴──────────────────────────────┘
```

### 系统时间→绝对时间：SystemToAbsolute

这是最简单的一层——纯粹的单位换算：

```cpp
// 来源：Clock.cpp 第 234-237 行
double CDVDClock::SystemToAbsolute(int64_t system) {
    // system: 操作系统时钟的原始 tick 值
    // m_systemOffset: 系统时钟的初始偏移（构造时记录）
    // m_systemFrequency: 系统时钟每秒多少 tick
    //
    // 公式含义：
    //   (经过的 tick 数) / (每秒 tick 数) * (每秒的 DVD_TIME_BASE 单位数)
    //   = 经过的秒数 * 1000000 = 经过的微秒数
    return DVD_TIME_BASE * (double)(system - m_systemOffset)
                         / m_systemFrequency;
}
```

举例：假设 `m_systemFrequency = 10000000`（Windows 的 `QueryPerformanceFrequency` 通常是 10MHz），系统启动后经过了 50000000 tick：

```
绝对时间 = 1000000 * (50000000 - 0) / 10000000
         = 1000000 * 5.0
         = 5000000.0  （即 5 秒）
```

### 绝对时间→播放时间：SystemToPlaying

这是时钟系统最核心的公式，处理了暂停、变速、跳变的全部逻辑：

```cpp
// 来源：Clock.cpp 第 244-267 行
double CDVDClock::SystemToPlaying(int64_t system) {
    // 首次调用时重置所有状态
    if (m_bReset) {
        m_startClock = system;        // 记录起始时间
        m_systemUsed = m_systemFrequency; // 有效频率=系统频率（1.0x速度）
        m_iDisc = 0;                  // 无跳变
        m_systemAdjust = 0;           // 无微调
        m_speedAdjust = 0;            // 无速度微调
        m_bReset = false;
    }

    int64_t current;
    if (m_pauseClock)
        current = m_pauseClock;   // 暂停时：使用冻结的时间点
    else
        current = system;         // 正常播放：使用当前系统时间

    // ★ 核心公式 ★
    // 播放时间 = DVD_TIME_BASE × (当前时间 - 起始时间 + 累计微调)
    //            ÷ 有效频率 + 跳变偏移
    return DVD_TIME_BASE
           * (double)(current - m_startClock + m_systemAdjust)
           / m_systemUsed
           + m_iDisc;
}
```

我们把核心公式拆开来看每一项的作用：

```
播放时间 = DVD_TIME_BASE × (current - m_startClock + m_systemAdjust)
                           ─────────────────────────────────────────
                                         m_systemUsed
           + m_iDisc

各项含义：
┌──────────────────┬──────────────────────────────────────────────┐
│ current           │ 当前系统时间（暂停时被冻结为 m_pauseClock）   │
│ m_startClock      │ 本次播放段的起始系统时间                      │
│ m_systemAdjust    │ 累计微调量（GetClock 中渐进累加的小修正）     │
│ m_systemUsed      │ 有效频率 = m_systemFrequency * 1000 / speed  │
│ m_iDisc           │ 不连续跳变偏移（seek 或 ErrorAdjust 设置）   │
│ DVD_TIME_BASE     │ 1000000，将秒转为微秒单位                    │
└──────────────────┴──────────────────────────────────────────────┘
```

### 公式如何实现变速

关键在 `m_systemUsed`。正常速度时 `m_systemUsed = m_systemFrequency`，公式简化为：

```
播放时间 ≈ DVD_TIME_BASE × 经过tick / 每秒tick = 经过的秒数（微秒单位）
```

当播放速度变为 2 倍时，`m_systemUsed = m_systemFrequency / 2`，同样的 tick 差值除以更小的分母，结果变为 2 倍——系统过了 1 秒，播放时间走了 2 秒。

### 公式如何实现暂停

暂停时 `current = m_pauseClock`（冻结值），分子 `current - m_startClock + m_systemAdjust` 变为常量，播放时间停止增长。

### 公式如何实现跳变

`m_iDisc` 是一个加法偏移。调用 `Discontinuity(clock, absolute)` 时：
- `m_startClock` 被重置为当前时间 → 分子变为约 0
- `m_iDisc = clock` → 播放时间直接跳到 `clock`

从跳变点之后，分子从 0 开始重新增长，等效于从 `clock` 位置继续播放。

---

## 播放控制：SetSpeed 与 Pause

### SetSpeed —— 变速的数学

`SetSpeed` 是变速和暂停的统一入口。速度值以 1000 为基准（DVD_PLAYSPEED_NORMAL = 1000）：

```cpp
// 来源：Clock.cpp 第 91-119 行
void CDVDClock::SetSpeed(int iSpeed) {
    // iSpeed 示例值：
    //   1000 = 1.0x 正常速度
    //   500  = 0.5x 半速
    //   2000 = 2.0x 双速
    //   0    = DVD_PLAYSPEED_PAUSE 暂停

    if (iSpeed == DVD_PLAYSPEED_PAUSE) {
        // 暂停：冻结当前系统时间
        if (!m_pauseClock)
            m_pauseClock = m_videoRefClock->GetTime();
        return;
    }

    // 计算新的有效频率
    // newfreq = systemFrequency * 1000 / speed
    // 2x速度 → newfreq = freq/2 → 同样时间，播放进度翻倍
    int64_t newfreq = m_systemFrequency * DVD_PLAYSPEED_NORMAL / iSpeed;

    int64_t current = m_videoRefClock->GetTime();

    // 从暂停恢复时，补偿暂停期间的时间流逝
    if (m_pauseClock) {
        // 将暂停期间的时间差加到 startClock 上
        // 这样 (current - m_startClock) 不包含暂停的那段时间
        m_startClock += current - m_pauseClock;
        m_pauseClock = 0;  // 清除暂停标记
    }

    // ★ 关键：速度切换时保证播放时间连续 ★
    // 重新计算 m_startClock，使得切换前后 SystemToPlaying 返回相同值
    //
    // 推导：设切换前播放时间为 T
    //   T = (current - old_start) / old_freq * DVD_TIME_BASE + m_iDisc
    // 切换后要求 T 不变：
    //   T = (current - new_start) / new_freq * DVD_TIME_BASE + m_iDisc
    // 解出：
    //   new_start = current - (current - old_start) * new_freq / old_freq
    m_startClock = current
        - (int64_t)((double)(current - m_startClock) * newfreq / m_systemUsed);
    m_systemUsed = newfreq;
}
```

**为什么重算 m_startClock？** 如果只改 `m_systemUsed` 而不调整 `m_startClock`，播放时间会在切换瞬间发生跳变。通过重算 `m_startClock`，保证了 `SystemToPlaying` 在速度切换前后返回完全相同的值——用户感知不到任何中断。

### Pause —— 暂停的实现

```cpp
// 来源：Clock.cpp 第 82-89 行
void CDVDClock::Pause(bool pause) {
    if (pause) {
        // 记住暂停前的速度，将来恢复用
        m_speedAfterPause = m_currentSpeed;
        SetSpeed(DVD_PLAYSPEED_PAUSE);  // 冻结时钟
    } else {
        // 恢复暂停前的速度
        SetSpeed(m_speedAfterPause);    // 内部会补偿暂停时间
    }
}
```

暂停/恢复的完整时间线：

```
时间轴 ──────────────────────────────────────────────────▶

系统时间:   0    1    2    3    4    5    6    7    8
            ├────┼────┼────┼────┼────┼────┼────┼────┤

事件:       开始播放    暂停              恢复         
            ↓           ↓                 ↓
播放时间:   0    1    2  [冻结在2.0]      2    3    4
                         2    2    2    2

内部状态：
  m_pauseClock:          设为 t=2 的系统时间
  m_startClock:                            += (t5 - t2) 补偿暂停
  m_pauseClock:                            清零
```

---

## 同步误差修正：GetClock 与 ErrorAdjust

### GetClock —— 带微调的时钟读取

`GetClock` 不只是简单调用 `SystemToPlaying`，它还实现了渐进式的速度微调：

```cpp
// 来源：Clock.cpp 第 50-80 行
double CDVDClock::GetClock(double &absolute) {
    int64_t current = m_videoRefClock->GetTime();

    // 渐进式速度微调：
    // m_speedAdjust 是一个很小的系数（如 0.001），
    // 每次调用 GetClock 时，将经过的时间乘以这个系数累加到 m_systemAdjust
    // 效果：时钟速度比标准值略快或略慢，用于修正长期漂移
    m_systemAdjust += m_speedAdjust * (current - m_lastSystemTime);
    m_lastSystemTime = current;

    absolute = SystemToAbsolute(current);
    return SystemToPlaying(current);
}
```

为什么需要 `m_speedAdjust`？声卡的实际采样率（Sample Rate，每秒采集/播放的音频样本数）与标称值总有微小差异。比如标称 48000Hz 的声卡实际可能是 47998Hz。长时间播放后这个差异会累积：

```
48000Hz vs 47998Hz 的漂移：
  1 分钟：0.04 秒偏差（可感知）
  10 分钟：0.4 秒偏差（严重不同步）
  1 小时：2.5 秒偏差（完全无法观看）
```

`m_speedAdjust` 通过微量调整时钟速率来补偿这种漂移，而不是等到偏差累积后再做大幅跳变。

### ErrorAdjust —— 同步误差的主动修正

当音频线程检测到同步误差超过阈值（Threshold，触发动作的边界值）时，调用 `ErrorAdjust` 进行修正：

```cpp
// 来源：Clock.cpp 第 134-169 行
double CDVDClock::ErrorAdjust(double error, const char *log) {
    double clock, absolute, adjustment;
    clock = GetClock(absolute);

    // 当速度微调正在工作且误差较小（<100ms）时，跳过跳变修正
    // 避免两种修正机制冲突——让渐进修正先工作
    if (m_speedAdjust != 0 && error < DVD_MSEC_TO_TIME(100))
        return 0;

    adjustment = error;  // 默认：直接跳变整个误差量

    if (m_vSyncAdjust != 0) {
        // ★ VSync 模式：使用帧时间为单位进行微量校正 ★
        //
        // VSync（Vertical Sync，垂直同步）模式下，视频输出与显示器刷新率
        // 严格对齐，不能随意跳变——跳太多会造成画面撕裂或卡顿。
        // 因此使用"每次调整一帧时间"的保守策略。
        //
        // 不对称阈值的原因：
        //   音频超前 >20ms → 调整（人耳对音频领先非常敏感）
        //   音频落后 >27ms → 调整（人耳对音频略微落后不太敏感）
        if (error > 0.02 * DVD_TIME_BASE)        // 超前 > 20ms
            adjustment = m_frameTime;              // 向前跳一帧时间
        else if (error < -0.027 * DVD_TIME_BASE)  // 落后 > 27ms
            adjustment = -m_frameTime;             // 向后跳一帧时间
        else
            adjustment = 0;                        // 在容差内不调整
    }

    if (adjustment == 0)
        return 0;

    // 执行不连续跳变
    Discontinuity(clock + adjustment, absolute);
    return adjustment;
}
```

两种模式的对比：

| 特性 | VSync 模式 | 非 VSync 模式 |
|------|-----------|--------------|
| 调整单位 | 一帧时间（如 16.67ms@60fps） | 整个误差量 |
| 调整频率 | 每次只调整一帧，多次调用渐进修正 | 一次到位 |
| 画面影响 | 最小——与显示器刷新对齐 | 可能出现短暂跳帧 |
| 收敛速度 | 慢（可能需要多帧才能修正完） | 快（一次修正） |
| 超前阈值 | 20ms | 无阈值，直接修正 |
| 落后阈值 | 27ms | 无阈值，直接修正 |
| 使用场景 | 显示器刷新率已知且稳定 | 一般播放场景 |

**为什么 20ms 和 27ms 不对称？** 这与人类感知特性有关。心理声学（Psychoacoustics，研究人类如何感知声音的学科）研究表明：
- 音频**领先**于画面超过 20ms，观众会明显感觉"嘴还没动声音就出来了"
- 音频**落后**于画面超过 40-50ms，观众才会感觉到明显的延迟

因此对音频领先更敏感，使用更小的阈值（20ms）来更早触发修正。

### Discontinuity —— 时钟跳变

`Discontinuity` 是 ErrorAdjust 和 seek 操作的底层实现，直接将时钟重置到新位置：

```cpp
// 来源：Clock.cpp 第 171-180 行
void CDVDClock::Discontinuity(double clock, double absolute) {
    // 将绝对时间反转为系统时间，作为新的起始点
    m_startClock = AbsoluteToSystem(absolute);
    if (m_pauseClock)
        m_pauseClock = m_startClock;  // 暂停状态下也同步更新

    m_iDisc = clock;        // 设置新的播放时间基准
    m_bReset = false;       // 防止下次 SystemToPlaying 再次重置
    m_systemAdjust = 0;     // 清除所有累计微调
    m_speedAdjust = 0;      // 清除速度微调
}
```

跳变后的效果：
```
跳变前：播放时间 = 30.5 秒，需要跳到 90.0 秒

Discontinuity(90.0 * DVD_TIME_BASE, absolute) 执行后：
  m_startClock = 当前系统时间        → 分子归零
  m_iDisc = 90000000.0              → 播放时间 ≈ 0 + 90000000 = 90.0秒

下一帧：播放时间 = 很小的增量 + 90000000 ≈ 90.0秒 ✓
```

---

## 帧率与 VSync 调整：UpdateFramerate

当视频帧率与显示器刷新率有特定关系时（如 24fps 视频在 60Hz 显示器上），`UpdateFramerate` 会微调视频参考时钟的速度：

```cpp
// 来源：Clock.cpp 第 195-228 行
void CDVDClock::UpdateFramerate(double fps, double *interval) {
    // fps: 视频的帧率（如 23.976, 24.0, 29.97, 30.0, 60.0）
    // m_frameTime: 更新为一帧的 DVD_TIME_BASE 时间

    if (fps > 0)
        m_frameTime = DVD_TIME_BASE / fps;
    else
        m_frameTime = DVD_TIME_BASE / 60.0;  // 默认 60fps

    if (interval)
        *interval = m_frameTime;

    // 计算 weight = round(显示器刷新率) / round(视频帧率)
    // 当 weight 是整数时（如 60Hz/30fps = 2, 60Hz/24fps = 2.5≈不是整数）
    // 说明帧率与刷新率有整数倍关系，可以精确对齐

    // 通过调整 m_videoRefClock 的速度，让视频参考时钟与
    // 显示器刷新严格同步，避免因微小的帧率不匹配造成画面抖动
    // （这种抖动叫做 judder，即画面不均匀运动的视觉效果）
}
```

---

## CVideoPlayerAudio 同步状态机

CDVDClock 提供了时钟基础设施，而 `CVideoPlayerAudio`（音频播放线程）是使用这些设施进行同步的主要消费者。

### 三个同步状态

音频线程维护了一个三状态的有限状态机（FSM，Finite State Machine，一种用有限个状态和状态转换来描述行为的模型）：

```
                  ┌─────────────────────────────────────────┐
                  │                                         │
                  ▼                                         │
          ┌──────────────┐    首帧输出    ┌──────────────┐  │
  启动 ──▶│ SYNC_STARTING │─────────────▶│ SYNC_WAITSYNC │  │
          └──────────────┘              └──────┬───────┘  │
                  ▲                            │           │
                  │                   收到 RESYNC 消息      │
                  │                            │           │
                  │                            ▼           │
                  │                    ┌──────────────┐    │
                  │ flush/reset        │  SYNC_INSYNC  │    │
                  └────────────────────┤              ├────┘
                                       └──────────────┘
                                              │
                                       持续监测 syncerror
                                       并调用 ErrorAdjust
```

| 状态 | 英文 | 含义 | 进入条件 | 行为 |
|------|------|------|---------|------|
| STARTING | 启动中 | 刚开始或刚重置 | 初始化 / flush / reset | 解码并填充音频缓冲区，等待足够数据 |
| WAITSYNC | 等待同步 | 音频就绪，等视频 | 首帧输出后 | 向主循环发送 `PLAYER_STARTED` 消息 |
| INSYNC | 已同步 | 正常播放 | 收到 RESYNC 消息后 | 持续解码+输出，检测并修正 syncerror |

### 音频处理主循环

`CVideoPlayerAudio::Process` 是音频线程的入口（在独立线程中运行），采用消息驱动架构：

```cpp
// 来源：VideoPlayerAudio.cpp（伪代码简化）
void CVideoPlayerAudio::Process() {
    while (!m_bStop) {
        // 从消息队列取消息（阻塞等待或超时返回）
        CDVDMsg *msg = m_messageQueue.Get(timeout, priority);

        switch (msg->GetMessageType()) {
        case GENERAL_SYNCHRONIZE:
            // 等待其他线程的同步信号（如视频线程就绪）
            break;

        case GENERAL_RESYNC:
            // 主循环通知：音视频已对齐，进入 INSYNC 状态
            m_syncState = SYNC_INSYNC;
            m_audioClock = msg->GetTime();  // 重设音频时钟
            break;

        case GENERAL_RESET:
            // 重置：清空解码器，回到 STARTING
            m_codec->Reset();
            m_syncState = SYNC_STARTING;
            break;

        case GENERAL_FLUSH:
            // 清空缓冲区（seek 时触发）
            m_dvdAudio.Flush();
            m_codec->Reset();
            m_syncState = SYNC_STARTING;
            break;

        case DEMUXER_PACKET:
            // ★ 核心解码路径 ★
            ProcessPacket(msg);
            break;
        }
    }
}
```

### 核心解码路径

当收到 `DEMUXER_PACKET`（解复用器发来的音频数据包）时：

```cpp
// 来源：VideoPlayerAudio.cpp（伪代码简化核心流程）
void CVideoPlayerAudio::ProcessPacket(CDVDMsg *msg) {
    // 步骤 1：解码音频数据包
    int consumed = m_codec->Decode(pData, iSize, dts, pts);

    // 步骤 2：获取解码后的 PCM 音频帧
    DVDAudioFrame audioframe;
    m_codec->GetData(audioframe);

    // 步骤 3：更新音频时钟
    if (audioframe.pts == DVD_NOPTS_VALUE) {
        // 没有 PTS 时，使用预测值（上一帧的 PTS + 上一帧的持续时间）
        audioframe.pts = m_audioClock;
        audioframe.hasTimestamp = false;
    } else {
        // 有 PTS 时，直接采用（精确校正）
        m_audioClock = audioframe.pts;
    }

    // 步骤 4：检查同步状态转换
    if (m_syncState == SYNC_STARTING) {
        // 首帧输出后转为 WAITSYNC
        m_syncState = SYNC_WAITSYNC;
        // 发送 PLAYER_STARTED 消息给主循环
        m_messageParent.Put(new CDVDMsgPlayerStarted());
    }

    // 步骤 5：输出音频帧（同时检测同步误差）
    OutputPacket(audioframe);

    // 步骤 6：递增音频时钟（预测下一帧）
    // duration = 帧中样本数 / 采样率 * DVD_TIME_BASE
    m_audioClock += audioframe.duration;
}
```

### 同步误差检测与修正

`OutputPacket` 是同步修正的关键环节：

```cpp
// 来源：VideoPlayerAudio.cpp 第 525-538 行
bool CVideoPlayerAudio::OutputPacket(DVDAudioFrame &audioframe) {
    // 从音频输出设备获取同步误差
    // syncerror = 音频设备的实际播放位置 - 时钟期望的播放位置
    // 正值 = 音频超前，负值 = 音频落后
    double syncerror = m_dvdAudio.GetSyncError();

    // SYNC_DISCON 模式：误差超过 10ms 时启动修正
    if (m_synctype == SYNC_DISCON
        && fabs(syncerror) > DVD_MSEC_TO_TIME(10))
    {
        // 调用时钟的 ErrorAdjust 进行修正
        double correction = m_pClock->ErrorAdjust(
            syncerror, "CVideoPlayerAudio::OutputPacket");

        if (correction != 0) {
            // 将修正量反向应用到音频输出设备
            // 负号是因为：时钟跳变后，音频设备需要反向补偿
            m_dvdAudio.SetSyncErrorCorrection(-correction);
        }
    }

    // 将音频帧送入音频输出设备
    m_dvdAudio.AddPackets(audioframe);
    return true;
}
```

完整的同步修正数据流：

```
音频设备报告 syncerror（实际播放位置 vs 期望位置）
    │
    ▼
|syncerror| > 10ms ?
    │
    ├── 否 → 在容忍范围内，不调整
    │
    └── 是 → CDVDClock::ErrorAdjust(syncerror)
              │
              ├── speedAdjust 活跃且 error < 100ms ?
              │     └── 是 → 跳过（让渐进修正先工作）
              │
              ├── VSync 模式：
              │     ├── error > +20ms → adjustment = +m_frameTime
              │     ├── error < -27ms → adjustment = -m_frameTime
              │     └── 否则 → adjustment = 0
              │
              └── 非 VSync 模式：
                    └── adjustment = error（直接跳变全部误差）
              │
              ▼
        Discontinuity(clock + adjustment, absolute)
        → 时钟瞬间跳变
        → 视频线程下次 GetClock() 时感知到变化
        → 音频设备收到反向补偿 SetSyncErrorCorrection(-correction)
```

---

## 音频时钟的维护策略

音频时钟 `m_audioClock` 采用**双源更新**策略，兼顾精度和连续性：

### 精确更新：从 PTS

当解码帧携带 PTS（Presentation Time Stamp，显示时间戳）时，直接采用：

```cpp
// 有 PTS 时：精确校正
m_audioClock = audioframe.pts;  // 直接跳到精确位置
```

### 预测更新：从 duration

当帧没有 PTS 或在帧间需要推算时，用持续时间递增：

```cpp
// 帧结束后：预测下一帧位置
m_audioClock += audioframe.duration;
// duration = nb_samples / sample_rate * DVD_TIME_BASE
// 例：1024 样本 / 48000Hz * 1000000 = 21333.3 微秒 ≈ 21.3ms
```

这两种方式交替配合：

```
帧 1 (有PTS=0)     帧 2 (无PTS)        帧 3 (有PTS=63333)
     │                   │                    │
     ▼                   ▼                    ▼
audioClock = 0      audioClock += 21333   audioClock = 63333
             ──▶ 21333               ──▶ 42666            ──▶ 精确校正！
                (预测)                (预测)               (PTS 校正)
                                                           差值 = 63333 - 42666
                                                                = 20667（正常范围）
```

如果累积的预测误差过大（比如解码器丢弃了几个包），PTS 校正会把时钟拉回正确位置。

---

## 动手实践：实现简化版 CDVDClock

下面我们从零实现一个包含三层时间模型、暂停、变速、跳变功能的简化时钟：

```cpp
// simple_clock.cpp — 简化版 CDVDClock 实现
// 编译：g++ -std=c++17 -o simple_clock simple_clock.cpp -lpthread
//       cl /std:c++17 /EHsc simple_clock.cpp (Windows MSVC)
#include <cstdio>
#include <cstdint>
#include <cmath>
#include <chrono>
#include <thread>

// 模拟 KrKr2 的时间基准
static constexpr double TIME_BASE = 1000000.0;  // 1秒 = 1000000单位

class SimpleDVDClock {
public:
    SimpleDVDClock() {
        m_systemFrequency = 1000000; // 微秒精度
        m_systemUsed = m_systemFrequency;
        m_startClock = getSystemTime();
        m_systemOffset = m_startClock;
        m_pauseClock = 0;
        m_iDisc = 0.0;
        m_systemAdjust = 0.0;
        m_speedAdjust = 0.0;
        m_lastSystemTime = m_startClock;
        m_frameTime = TIME_BASE / 60.0;  // 默认 60fps
    }

    // 获取当前播放时间（微秒单位），同时输出绝对时间
    double getClock(double &absolute) {
        int64_t current = getSystemTime();

        // 渐进式速度微调
        m_systemAdjust += m_speedAdjust * (current - m_lastSystemTime);
        m_lastSystemTime = current;

        absolute = systemToAbsolute(current);
        return systemToPlaying(current);
    }

    double getClock() {
        double abs;
        return getClock(abs);
    }

    // 设置播放速度（1000 = 正常，500 = 半速，2000 = 双速，0 = 暂停）
    void setSpeed(int speed) {
        if (speed == 0) {
            if (!m_pauseClock)
                m_pauseClock = getSystemTime();
            return;
        }

        int64_t newfreq = m_systemFrequency * 1000 / speed;
        int64_t current = getSystemTime();

        // 从暂停恢复
        if (m_pauseClock) {
            m_startClock += current - m_pauseClock;
            m_pauseClock = 0;
        }

        // 保证速度切换时播放时间连续
        m_startClock = current
            - (int64_t)((double)(current - m_startClock)
                        * newfreq / m_systemUsed);
        m_systemUsed = newfreq;
        printf("  [时钟] 速度设为 %.1fx (有效频率=%lld)\n",
               speed / 1000.0, (long long)m_systemUsed);
    }

    // 不连续跳变
    void discontinuity(double clock, double absolute) {
        m_startClock = absoluteToSystem(absolute);
        if (m_pauseClock)
            m_pauseClock = m_startClock;
        m_iDisc = clock;
        m_systemAdjust = 0;
        m_speedAdjust = 0;
        printf("  [时钟] 跳变到 %.3f 秒\n", clock / TIME_BASE);
    }

    // 误差校正（简化版，非 VSync 模式）
    double errorAdjust(double error) {
        double clock, absolute;
        clock = getClock(absolute);

        if (fabs(error) < 0.01 * TIME_BASE)  // 10ms 以内不修正
            return 0;

        printf("  [时钟] 修正误差 %.1fms\n", error / TIME_BASE * 1000);
        discontinuity(clock + error, absolute);
        return error;
    }

    // 设置速度微调系数
    void setSpeedAdjust(double adjust) {
        m_speedAdjust = adjust;
    }

private:
    static int64_t getSystemTime() {
        auto t = std::chrono::steady_clock::now().time_since_epoch();
        return std::chrono::duration_cast<
            std::chrono::microseconds>(t).count();
    }

    double systemToAbsolute(int64_t system) {
        return TIME_BASE * (double)(system - m_systemOffset)
                         / m_systemFrequency;
    }

    double systemToPlaying(int64_t system) {
        int64_t current = m_pauseClock ? m_pauseClock : system;
        return TIME_BASE
               * (double)(current - m_startClock + m_systemAdjust)
               / m_systemUsed
               + m_iDisc;
    }

    int64_t absoluteToSystem(double absolute) {
        return (int64_t)(absolute / TIME_BASE * m_systemFrequency)
               + m_systemOffset;
    }

    int64_t m_systemFrequency;
    int64_t m_systemUsed;
    int64_t m_startClock;
    int64_t m_systemOffset;
    int64_t m_pauseClock;
    int64_t m_lastSystemTime;
    double m_iDisc;
    double m_systemAdjust;
    double m_speedAdjust;
    double m_frameTime;
};

int main() {
    SimpleDVDClock clock;
    auto sleepMs = [](int ms) {
        std::this_thread::sleep_for(std::chrono::milliseconds(ms));
    };

    printf("=== 测试 1：正常播放 ===\n");
    sleepMs(500);
    printf("  播放时间: %.3f 秒（期望≈0.5秒）\n",
           clock.getClock() / TIME_BASE);

    printf("\n=== 测试 2：暂停与恢复 ===\n");
    double t1 = clock.getClock() / TIME_BASE;
    printf("  暂停前: %.3f 秒\n", t1);
    clock.setSpeed(0);  // 暂停
    sleepMs(1000);       // 等待 1 秒
    printf("  暂停中: %.3f 秒（应该不变）\n",
           clock.getClock() / TIME_BASE);
    clock.setSpeed(1000);  // 恢复
    sleepMs(500);
    printf("  恢复后: %.3f 秒（应≈暂停前+0.5秒）\n",
           clock.getClock() / TIME_BASE);

    printf("\n=== 测试 3：2倍速 ===\n");
    double t2 = clock.getClock() / TIME_BASE;
    printf("  切换前: %.3f 秒\n", t2);
    clock.setSpeed(2000);  // 2倍速
    sleepMs(500);           // 实际等 0.5 秒
    double t3 = clock.getClock() / TIME_BASE;
    printf("  0.5秒后: %.3f 秒（增量应≈1.0秒）\n", t3);
    printf("  实际增量: %.3f 秒\n", t3 - t2);
    clock.setSpeed(1000);  // 恢复正常速度

    printf("\n=== 测试 4：seek 跳变 ===\n");
    printf("  跳变前: %.3f 秒\n", clock.getClock() / TIME_BASE);
    double abs;
    clock.getClock(abs);
    clock.discontinuity(90.0 * TIME_BASE, abs);
    sleepMs(500);
    printf("  跳变后: %.3f 秒（应≈90.5秒）\n",
           clock.getClock() / TIME_BASE);

    printf("\n=== 测试 5：误差修正 ===\n");
    printf("  修正前: %.3f 秒\n", clock.getClock() / TIME_BASE);
    clock.errorAdjust(0.05 * TIME_BASE);  // 修正 50ms
    printf("  修正后: %.3f 秒（应增加≈0.05秒）\n",
           clock.getClock() / TIME_BASE);

    // 小误差不修正
    double noCorrection = clock.errorAdjust(0.005 * TIME_BASE); // 5ms
    printf("  5ms误差: %s（应跳过）\n",
           noCorrection == 0 ? "未修正" : "已修正");

    printf("\n=== 所有测试完成 ===\n");
    return 0;
}
```

编译和运行：

```bash
# Linux / macOS
g++ -std=c++17 -O2 -o simple_clock simple_clock.cpp -lpthread
./simple_clock

# Windows (MSVC)
cl /std:c++17 /EHsc /O2 simple_clock.cpp
simple_clock.exe

# Windows (MinGW)
g++ -std=c++17 -O2 -o simple_clock.exe simple_clock.cpp

# Android NDK
$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android30-clang++ \
    -std=c++17 -O2 -o simple_clock simple_clock.cpp
```

预期输出（时间值会有微小波动）：

```
=== 测试 1：正常播放 ===
  播放时间: 0.500 秒（期望≈0.5秒）

=== 测试 2：暂停与恢复 ===
  暂停前: 0.500 秒
  [时钟] 速度设为 0.0x (暂停)
  暂停中: 0.500 秒（应该不变）
  [时钟] 速度设为 1.0x (有效频率=1000000)
  恢复后: 1.000 秒（应≈暂停前+0.5秒）

=== 测试 3：2倍速 ===
  切换前: 1.000 秒
  [时钟] 速度设为 2.0x (有效频率=500000)
  0.5秒后: 2.000 秒（增量应≈1.0秒）
  实际增量: 1.000 秒

=== 测试 4：seek 跳变 ===
  跳变前: 2.000 秒
  [时钟] 跳变到 90.000 秒
  跳变后: 90.500 秒（应≈90.5秒）

=== 测试 5：误差修正 ===
  修正前: 90.500 秒
  [时钟] 修正误差 50.0ms
  [时钟] 跳变到 90.550 秒
  修正后: 90.550 秒（应增加≈0.05秒）
  5ms误差: 未修正（应跳过）

=== 所有测试完成 ===
```

---

## 对照项目源码

### 相关源文件

| 文件 | 行数 | 内容 |
|------|------|------|
| `cpp/core/movie/ffmpeg/Clock.h` | 1-97 | CDVDClock 完整头文件，所有成员变量和方法声明 |
| `cpp/core/movie/ffmpeg/Clock.cpp` | 1-276 | CDVDClock 完整实现 |
| `cpp/core/movie/ffmpeg/Clock.cpp` 第 50-80 行 | — | `GetClock()`：带渐进微调的时钟读取 |
| `cpp/core/movie/ffmpeg/Clock.cpp` 第 91-119 行 | — | `SetSpeed()`：变速与暂停 |
| `cpp/core/movie/ffmpeg/Clock.cpp` 第 134-169 行 | — | `ErrorAdjust()`：VSync/非VSync误差修正 |
| `cpp/core/movie/ffmpeg/Clock.cpp` 第 171-180 行 | — | `Discontinuity()`：时钟跳变 |
| `cpp/core/movie/ffmpeg/Clock.cpp` 第 195-228 行 | — | `UpdateFramerate()`：帧率与VSync调整 |
| `cpp/core/movie/ffmpeg/Clock.cpp` 第 234-267 行 | — | `SystemToAbsolute()` / `SystemToPlaying()`：核心时间转换 |

### KrKr2 与 Kodi 的关系

KrKr2 的媒体播放器代码源自 **Kodi**（前身 XBMC，一个著名的开源媒体中心）。CDVDClock 的设计几乎完全继承了 Kodi 的 `CDVDClock` 类，包括：

- 三层时间模型的设计
- VSync 模式下的不对称阈值（20ms/27ms）
- `m_speedAdjust` 渐进微调机制
- SYNC_DISCON 同步策略

KrKr2 对 Kodi 代码做了以下适配：
1. 移除了网络流相关的同步逻辑（视觉小说不需要网络播放）
2. 简化了音频输出层（使用自己的 OpenAL 音频系统）
3. 保留了核心时钟和同步算法不变

---

## 常见错误与排查

### 错误 1：速度切换后播放时间跳变

```
现象：从 1x 切换到 2x 时，播放时间突然跳了几秒
原因：修改 m_systemUsed 时没有同步重算 m_startClock
      新旧频率下同一个 (current - startClock) 值产生不同的播放时间

解决：必须在修改 m_systemUsed 之前重算 m_startClock（见 SetSpeed 实现）
      公式：new_start = current - (current - old_start) * new_freq / old_freq
```

### 错误 2：暂停恢复后时间"快进"

```
现象：暂停 10 秒后恢复，播放时间瞬间跳了 10 秒
原因：恢复时没有补偿 m_startClock
      (current - m_startClock) 包含了暂停的 10 秒

解决：恢复时执行 m_startClock += (current - m_pauseClock)
      这样 (current - m_startClock) 就只包含实际播放时间
```

### 错误 3：seek 后音画不同步

```
现象：拖动进度条后，音频和视频不对齐
原因：seek 时只重置了时钟，没有通知音频/视频线程

完整 seek 流程（KrKr2 实际做法）：
1. CDVDClock::Discontinuity() — 重置时钟到目标位置
2. 向音频线程发送 GENERAL_FLUSH — 清空音频缓冲区
3. 向视频线程发送 GENERAL_FLUSH — 清空视频缓冲区
4. avcodec_flush_buffers() — 重置 FFmpeg 解码器状态
5. 音频线程回到 SYNC_STARTING 状态
6. 重新走一遍 STARTING → WAITSYNC → INSYNC 流程
```

---

## 本节小结

- **CDVDClock** 是 KrKr2 播放器的核心时钟组件，所有线程通过它获取播放时间
- **三层时间模型**将硬件时钟的不可控性逐层转化为可控的播放时间：系统时间（不可控）→ 绝对时间（统一单位）→ 播放时间（可暂停/变速/跳变）
- `SystemToPlaying` 的核心公式通过 `m_systemUsed`（有效频率）实现变速，通过 `m_pauseClock` 实现暂停，通过 `m_iDisc` 实现跳变
- **SetSpeed** 在切换速度时重算 `m_startClock`，保证播放时间不发生跳变
- **ErrorAdjust** 在 VSync 模式下使用不对称阈值（20ms/27ms）按帧步进修正，非 VSync 模式直接跳变
- **CVideoPlayerAudio** 维护三状态同步状态机（STARTING→WAITSYNC→INSYNC），通过 `OutputPacket` 检测 syncerror 并调用 ErrorAdjust 修正
- 音频时钟采用 PTS 精确更新 + duration 预测递增的双源策略

---

## 练习题与答案

### 题目 1：SystemToPlaying 公式推导

`CDVDClock::SystemToPlaying` 的核心公式是：

```
playing = DVD_TIME_BASE × (current - m_startClock + m_systemAdjust)
                         ÷ m_systemUsed + m_iDisc
```

请回答：
1. 当播放速度从 1.0x 变为 2.0x 时，`m_systemUsed` 如何变化？为什么这能实现 2 倍速？
2. `m_iDisc` 的作用是什么？什么情况下它会被修改？
3. 为什么暂停时使用 `m_pauseClock` 而不是简单地让 `m_systemUsed = ∞`？

<details>
<summary>查看答案</summary>

1. **m_systemUsed 变化**：
   - 1.0x 速度：`m_systemUsed = m_systemFrequency × 1000 / 1000 = m_systemFrequency`
   - 2.0x 速度：`m_systemUsed = m_systemFrequency × 1000 / 2000 = m_systemFrequency / 2`

   当 `m_systemUsed` 减半时，同样的分子 `(current - m_startClock)` 除以更小的分母，结果变为 2 倍。即系统过了 1 秒，播放时间走了 2 秒。

2. **m_iDisc 的作用**：它是时钟跳变的目标偏移量。当调用 `Discontinuity(clock, absolute)` 时，`m_iDisc = clock`，同时 `m_startClock` 被重置为当前时间。此时分子 ≈ 0，`SystemToPlaying` 立即返回 `m_iDisc` 值。

   修改场景：seek 跳转、`ErrorAdjust` 修正同步误差、流切换、首次播放重置。

3. **为什么不用 m_systemUsed = ∞**：
   - 整数类型无法表示无穷大，且即使用极大值，分母极大但非零，播放时间仍会缓慢增长
   - `m_pauseClock` 方案更优：直接冻结分子为常量值，播放时间精确停止
   - 恢复时通过 `m_startClock += (current - m_pauseClock)` 补偿暂停时间，数学上完全精确

</details>

### 题目 2：ErrorAdjust 行为分析

某播放器在 VSync 模式下运行（60fps 显示器，`m_frameTime = 16667 微秒`），音频线程报告以下同步误差。请判断每种情况下 ErrorAdjust 的行为：

| 情况 | syncerror | m_speedAdjust |
|------|-----------|---------------|
| A | +25000（超前 25ms） | 0 |
| B | -30000（落后 30ms） | 0 |
| C | +15000（超前 15ms） | 0 |
| D | +80000（超前 80ms） | 0.001 |
| E | -50000（落后 50ms） | 0 |

<details>
<summary>查看答案</summary>

| 情况 | 行为 | 原因 |
|------|------|------|
| A | `adjustment = +16667`（向前跳一帧） | error=25ms > 20ms 阈值，VSync 模式按帧步进 |
| B | `adjustment = -16667`（向后跳一帧） | error=-30ms < -27ms 阈值，VSync 模式按帧步进 |
| C | `adjustment = 0`（不调整） | error=15ms，在 [-27ms, +20ms] 容差范围内 |
| D | `adjustment = +16667`（向前跳一帧） | error=80ms < 100ms 但 speedAdjust≠0 → 等等，80ms < 100ms，所以跳过！`return 0` |
| E | `adjustment = -16667`（向后跳一帧） | error=-50ms < -27ms，speedAdjust=0 不触发跳过 |

**情况 D 解析**：虽然 80ms 超过了 VSync 的 20ms 阈值，但代码中先检查 `m_speedAdjust != 0 && error < 100ms`，80ms < 100ms 条件成立，所以直接返回 0，让渐进微调机制先处理。

</details>

### 题目 3：实现 VSync 模式 ErrorAdjust

在上面的 `SimpleDVDClock` 实现基础上，修改 `errorAdjust` 方法，增加 VSync 模式支持：
- 新增 `m_vsyncMode` 成员变量（bool）
- VSync 模式下使用 `m_frameTime` 为单位调整
- 超前阈值 20ms，落后阈值 27ms

<details>
<summary>查看答案</summary>

```cpp
// 在 SimpleDVDClock 类中修改/新增：

class SimpleDVDClock {
    // ... 现有成员 ...
    bool m_vsyncMode = false;  // 新增

public:
    void setVSyncMode(bool enable) {
        m_vsyncMode = enable;
        printf("  [时钟] VSync模式: %s\n", enable ? "开启" : "关闭");
    }

    double errorAdjust(double error) {
        double clock, absolute;
        clock = getClock(absolute);

        // speedAdjust 活跃且误差 < 100ms 时跳过
        if (m_speedAdjust != 0
            && fabs(error) < 0.1 * TIME_BASE)
            return 0;

        double adjustment = error;  // 默认：非VSync，直接跳变

        if (m_vsyncMode) {
            // VSync 模式：按帧步进
            if (error > 0.02 * TIME_BASE)          // 超前 > 20ms
                adjustment = m_frameTime;            // +1 帧
            else if (error < -0.027 * TIME_BASE)    // 落后 > 27ms
                adjustment = -m_frameTime;           // -1 帧
            else
                adjustment = 0;                      // 容差内不调整
        } else {
            // 非 VSync：10ms 阈值
            if (fabs(error) < 0.01 * TIME_BASE)
                return 0;
        }

        if (adjustment == 0)
            return 0;

        printf("  [时钟] %s模式修正: error=%.1fms, adjustment=%.1fms\n",
               m_vsyncMode ? "VSync" : "直接",
               error / TIME_BASE * 1000,
               adjustment / TIME_BASE * 1000);
        discontinuity(clock + adjustment, absolute);
        return adjustment;
    }
};

// 测试代码：
int main() {
    SimpleDVDClock clock;
    clock.setVSyncMode(true);  // 开启 VSync（60fps）

    double abs;
    clock.getClock(abs);

    // 测试各种误差
    printf("25ms超前: ");
    clock.errorAdjust(25000);   // 应调整 +16667
    printf("15ms超前: ");
    clock.errorAdjust(15000);   // 应跳过（容差内）
    printf("30ms落后: ");
    clock.errorAdjust(-30000);  // 应调整 -16667

    return 0;
}
```

</details>

---

## 下一步

本节深入分析了 CDVDClock 的内部实现和 CVideoPlayerAudio 的同步机制。下一节我们将通过完整的动手实践来巩固这些知识，实现一个完整的音视频同步演示程序。

→ [动手实践与总结](./03-动手实践与总结.md)
