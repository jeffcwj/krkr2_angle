# CRenderManager 渲染管线

> **所属模块：** M06-视频播放器
> **前置知识：** [01-解码帧到纹理上传](./01-解码帧到纹理上传.md)、[01-CDVDClock主时钟](../03-音视频同步机制/01-CDVDClock主时钟.md)
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 `CRenderManager` 中三个帧队列（free、queued、discard）的状态流转
2. 掌握 `EPRESENTSTEP` 状态机的五种状态及其转换条件
3. 理解 `FlipPage()` 如何将帧从解码线程提交到渲染线程
4. 深入分析 `PrepareNextRender()` 的延迟帧跳过算法和容忍度收紧机制
5. 理解 `FrameMove()` 主循环如何驱动整个渲染流水线

## CRenderManager 架构概览

### 核心职责

`CRenderManager`（渲染管理器）是视频播放管线中连接解码器和显示输出的中枢组件。它继承自 Kodi/XBMC 的设计，负责管理帧队列、调度渲染时机、处理迟到帧的跳过，以及与时钟系统协调 V-Sync 对齐。在 KrKr2 中，CRenderManager 的大部分 Kodi 原有功能被简化——它主要充当解码线程和主线程之间的帧传递中间层。

来源文件：`cpp/core/movie/ffmpeg/VideoRenderer.h` 和 `cpp/core/movie/ffmpeg/VideoRenderer.cpp`

```
CRenderManager 在播放器中的位置：

CVideoPlayerVideo (解码线程)              CRenderManager               主线程
┌────────────────────┐         ┌──────────────────────────┐   ┌──────────────┐
│ OutputPicture()    │──────▶  │ AddVideoPicture()        │   │              │
│   → WaitForBuffer()│◀──────  │ WaitForBuffer()          │   │              │
│   → AddVideoPicture│──────▶  │   → m_pRenderer->...     │   │              │
│   → FlipPage()     │──────▶  │ FlipPage() → 入队 m_queued│   │ FrameMove() │
│                    │         │                          │   │  → Configure │
│                    │         │ PrepareNextRender()       │◀──│  → Prepare   │
│                    │         │   → 迟到帧跳过           │   │  → FlipPage  │
│                    │         │   → m_presentsource 设置  │   │  → Render    │
│                    │         │                          │   │  → 释放 disc │
└────────────────────┘         │ Render()                 │──▶│              │
                               │   → PresentSingle()      │   │              │
                               └──────────────────────────┘   └──────────────┘
```

### 关键成员变量

```cpp
// VideoRenderer.h 第 46-279 行
class CRenderManager {
    CBaseRenderer* m_pRenderer;    // 底层渲染器（KrKr2 中是 TVPMoviePlayer 自身）

    // 三个帧队列（使用 deque 实现）
    std::deque<int> m_free;        // 空闲缓冲索引队列
    std::deque<int> m_queued;      // 等待显示的帧队列
    std::deque<int> m_discard;     // 等待释放的已显示帧队列

    // 帧数据存储（NUM_BUFFERS=6 个槽位）
    struct SPresent {
        double pts;                // 帧的显示时间戳
        EPRESENTMETHOD presentmethod;  // 呈现方式（单帧/交错/混合）
    } m_Queue[NUM_BUFFERS];

    // 状态机
    EPRESENTSTEP m_presentstep;    // 当前呈现步骤
    int m_presentsource;           // 当前正在显示的缓冲索引
    bool m_forceNext;              // 强制推进到下一帧

    // 延迟检测
    int m_lateframes;              // 连续迟到帧计数器
    double m_presentpts;           // 当前显示帧的 PTS
    int m_QueueSize;               // 有效队列大小
    int m_QueueSkip;               // 累计跳过帧数

    // 时钟与同步
    CDVDClock& m_dvdClock;         // 主时钟引用
    double m_displayLatency;       // 显示延迟补偿
    std::atomic_int m_videoDelay;  // 视频延迟偏移
    CClockSync m_clockSync;        // VSync 时钟同步

    // 同步原语
    CCriticalSection m_statelock;  // 状态锁
    CCriticalSection m_presentlock; // 呈现锁
    CCriticalSection m_datalock;   // 数据锁
    std::condition_variable_any m_presentevent;  // 状态变化通知
};
```

## 三队列帧管理模型

### 帧队列状态流转

CRenderManager 使用三个 `std::deque<int>` 管理 `NUM_BUFFERS`（6）个帧缓冲的生命周期。每个元素是缓冲索引（0-5），帧数据存储在 `m_Queue[index]` 中。

```
帧缓冲生命周期：

              FlipPage()           PrepareNextRender()        Render() 后释放
    free ──────────────▶ queued ──────────────────▶ presentsource ──▶ discard ──▶ free
   (空闲)    入队等待显示  (等待)    选中最佳帧显示    (显示中)      已显示     (回收)

详细状态流：

  ┌──────┐  FlipPage()  ┌────────┐  PrepareNextRender()  ┌───────────┐  FrameMove() ┌─────────┐
  │ free │─────────────▶│ queued │─────────────────────▶│ presentsrc│────────────▶│ discard │
  │      │              │        │  跳过的迟到帧 ──────▶│ (显示中)  │              │         │
  └──────┘              └────────┘                      └───────────┘              └─────────┘
     ▲                                                                                │
     │                                                                                │
     └────────────────────────────── NeedBuffer()==false ──────────────────────────────┘
```

### 初始化与队列填充

`Configure()` 方法初始化队列。索引 0 保留给 `m_presentsource`（当前显示帧），其余索引进入 free 队列：

```cpp
// VideoRenderer.cpp 第 244-249 行
m_queued.clear();
m_discard.clear();
m_free.clear();
m_presentsource = 0;          // 索引 0 作为初始显示槽
for (int i = 1; i < m_QueueSize; i++)
    m_free.push_back(i);      // 索引 1..QueueSize-1 进入空闲队列

// 假设 QueueSize=4，初始状态：
// free: [1, 2, 3]
// queued: []
// discard: []
// presentsource: 0
```

### requeue 帮助函数

```cpp
// VideoRenderer.cpp 第 23-26 行
// 从源队列头部取出一个索引，追加到目标队列尾部
static void requeue(std::deque<int>& trg, std::deque<int>& src) {
    trg.push_back(src.front());
    src.pop_front();
}
// 示例：requeue(m_queued, m_free) → 从 free 取头部，放到 queued 尾部
```

## EPRESENTSTEP 状态机

### 五种状态

```cpp
// VideoRenderer.h 第 201-207 行
enum EPRESENTSTEP {
    PRESENT_IDLE  = 0,  // 空闲：无帧待显示
    PRESENT_FLIP  = 1,  // 翻转：通知渲染器交换缓冲
    PRESENT_FRAME = 2,  // 帧显示中：第一场
    PRESENT_FRAME2= 3,  // 帧显示中：第二场（隔行扫描）
    PRESENT_READY = 4,  // 就绪：有帧在队列中等待
};
```

### 状态转换图

```
                                      queued 非空
                    ┌─────────────────────────────────────┐
                    │                                     │
                    ▼                                     │
              ┌──────────┐                          ┌──────────┐
              │  READY   │                          │   IDLE   │
              │ (有帧等待)│                          │ (无帧)   │
              └────┬─────┘                          └──────────┘
                   │                                     ▲
                   │ PrepareNextRender()                  │
                   │ renderPts >= nextFramePts            │
                   ▼                                     │
              ┌──────────┐                               │
              │   FLIP   │                               │
              │ (翻转中) │                               │
              └────┬─────┘                               │
                   │                                     │
                   │ FrameMove() 调用                     │
                   │ m_pRenderer->FlipPage()             │
                   ▼                                     │
              ┌──────────┐                               │
              │  FRAME   │                               │
              │ (第一场) │                               │
              └────┬─────┘                               │
                   │                                     │
                   │ Render() 完成                        │
                   │ (逐行扫描: 直接→IDLE)               │
                   │ (隔行扫描: →FRAME2→IDLE)            │
                   └─────────────────────────────────────┘
```

### 状态转换时序示例

```cpp
// 模拟一帧的完整生命周期
// 假设初始状态：free=[1,2,3], queued=[], presentsource=0, step=IDLE

// 1. 解码线程调用 FlipPage() — 帧入队
//    free=[2,3], queued=[1], step=READY
FlipPage(bStop, pts, false);
// → source = m_free.front() = 1
// → m_Queue[1].pts = pts
// → requeue(m_queued, m_free)
// → m_presentstep = PRESENT_READY

// 2. 主线程调用 FrameMove() — 检查是否该显示
FrameMove();
// → m_presentstep == PRESENT_READY → PrepareNextRender()
//   → renderPts >= m_Queue[1].pts → 选中帧 1
//   → m_presentstep = PRESENT_FLIP
//   → m_discard.push_back(0)  // 旧帧 0 进入 discard
//   → m_presentsource = 1     // 新帧 1 成为显示源

// 3. FrameMove() 继续处理 FLIP 状态
// → m_pRenderer->FlipPage(1)
// → m_presentstep = PRESENT_FRAME

// 4. 主线程调用 Render() — 实际渲染
Render(clear, flags, alpha, true);
// → PresentSingle() → m_pRenderer->RenderUpdate()
// → m_presentstep = PRESENT_IDLE (逐行扫描直接转 IDLE)
// → queued 非空则转 PRESENT_READY（检查是否有更多帧）

// 5. FrameMove() 释放 discard 队列
// → m_pRenderer->ReleaseBuffer(0)
// → m_free.push_back(0)
// → discard 清空
```

## FlipPage：帧入队

`FlipPage()` 是解码线程调用的接口，将已完成解码的帧提交到 CRenderManager 的等待队列。来源文件：`VideoRenderer.cpp` 第 708-756 行。

```cpp
void CRenderManager::FlipPage(volatile std::atomic_bool& bStop,
                               double pts, bool wait) {
    {
        CSingleLock lock(m_statelock);
        if (bStop) return;         // 停止标志检查
        if (!m_pRenderer) return;  // 渲染器未初始化
    }

    CSingleLock lock(m_presentlock);

    if (m_free.empty()) return;  // 无空闲缓冲，静默返回

    // 1. 从 free 队列取出头部索引
    int source = m_free.front();

    // 2. 填充帧元数据
    SPresent& m = m_Queue[source];
    m.presentmethod = PRESENT_METHOD_SINGLE;  // KrKr2 只用单帧模式
    m.pts = pts;                              // 设置显示时间戳

    // 3. 移动到 queued 队列
    requeue(m_queued, m_free);

    // 4. 如果当前空闲，触发 READY
    if (m_presentstep == PRESENT_IDLE) {
        m_presentstep = PRESENT_READY;
        m_presentevent.notify_all();  // 唤醒等待的渲染线程
    }

    // 5. 如果需要同步等待（wait=true）
    if (wait) {
        m_forceNext = true;   // 强制 PrepareNextRender 立即推进
        Timer endtime(200);   // 最多等 200ms
        while (m_presentstep == PRESENT_READY) {
            m_presentevent.wait_for(lock, std::chrono::milliseconds(20));
            if (endtime.IsTimePast() || bStop) break;
        }
        m_forceNext = false;
    }
}
```

**关键设计点**：
- `FlipPage()` 在解码线程中调用，需要通过锁保护与主线程的并发访问
- `wait=true` 时解码线程会阻塞等待主线程处理完当前帧，这在需要精确帧同步的场景下使用
- 如果 free 队列为空，`FlipPage()` 静默返回，帧被丢弃

## PrepareNextRender：迟到帧跳过

`PrepareNextRender()` 是 CRenderManager 最核心的调度算法。它决定从 queued 队列中选取哪一帧进行显示，跳过已经迟到的帧。来源文件：`VideoRenderer.cpp` 第 1141-1222 行。

### 算法流程

```
PrepareNextRender() 流程：

1. 获取当前渲染时间 = 时钟值 + 显示延迟补偿
2. 获取队列头部帧的 PTS
3. 如果 VSync 时钟同步启用：
   a. 计算与帧时间的取模误差
   b. 累积误差，每 30 帧计算平均值
   c. 调整 VSync 偏移
4. 判断是否该显示：renderPts >= nextFramePts？
5. 如果该显示：
   a. 遍历 queued，查找可跳过的迟到帧
   b. 容忍度随 m_lateframes 收紧
   c. 跳过的帧转移到 discard
   d. 更新 m_lateframes
   e. 设置 PRESENT_FLIP
```

### 核心代码详解

```cpp
void CRenderManager::PrepareNextRender() {
    if (m_queued.empty()) {
        m_presentstep = PRESENT_IDLE;
        m_presentevent.notify_all();
        return;
    }

    // 1. 计算当前渲染时间
    double frameOnScreen = m_dvdClock.GetClock();
    double frametime = 1.0 / g_graphicsContext.GetFPS() * DVD_TIME_BASE;
    // 注意：g_graphicsContext.GetFPS() 在 KrKr2 中硬编码返回 60

    // 2. 显示延迟补偿
    // totalLatency = 驱动内部缓冲延迟 - 用户偏移 + 2帧管线延迟
    double totalLatency = DVD_SEC_TO_TIME(m_displayLatency)
                        - DVD_MSEC_TO_TIME(m_videoDelay)
                        + 2 * frametime;

    double renderPts = frameOnScreen + totalLatency;

    // 3. 获取队列头部帧的 PTS
    double nextFramePts = m_Queue[m_queued.front()].pts;
    if (m_dvdClock.GetClockSpeed() < 0)  // 倒放时不做比较
        nextFramePts = renderPts;

    // 4. VSync 时钟同步（可选，在 KrKr2 中通常禁用）
    if (m_clockSync.m_enabled) {
        double err = fmod(renderPts - nextFramePts, frametime);
        m_clockSync.m_error += err;
        m_clockSync.m_errCount++;
        if (m_clockSync.m_errCount > 30) {
            double average = m_clockSync.m_error / m_clockSync.m_errCount;
            m_clockSync.m_syncOffset = average;
            m_clockSync.m_error = 0;
            m_clockSync.m_errCount = 0;
            m_dvdClock.SetVsyncAdjust(-average);
        }
        renderPts += frametime / 2 - m_clockSync.m_syncOffset;
    }

    // 5. 判断是否到了显示时间
    if (renderPts >= nextFramePts || m_forceNext) {
        auto iter = m_queued.begin();
        int idx = *iter;
        ++iter;

        // 6. 遍历队列，跳过迟到帧
        while (iter != m_queued.end()) {
            // 容忍度系数 x：
            // m_lateframes <= 6 → x = 0.98（宽容，允许帧稍微迟到）
            // m_lateframes > 6  → x = 0  （零容忍，任何迟到都跳过）
            double x = (m_lateframes <= 6) ? 0.98 : 0;

            // 如果下一帧还没到时间（考虑容忍度），停止跳过
            if (renderPts < m_Queue[*iter].pts + x * frametime)
                break;

            idx = *iter;  // 跳到这一帧
            ++iter;
        }

        // 7. 将跳过的帧移到 discard
        while (m_queued.front() != idx) {
            requeue(m_discard, m_queued);
            m_QueueSkip++;  // 累计跳过计数
        }

        // 8. 更新 m_lateframes
        int lateframes = (renderPts - m_Queue[idx].pts) * m_fps / DVD_TIME_BASE;
        if (lateframes)
            m_lateframes += lateframes;  // 累加延迟
        else
            m_lateframes = 0;            // 准时则立即归零

        // 9. 切换显示源
        m_presentstep = PRESENT_FLIP;
        m_discard.push_back(m_presentsource);  // 旧帧进入 discard
        m_presentsource = idx;                  // 新帧成为显示源
        m_queued.pop_front();                   // 从 queued 中移除
        m_presentpts = m_Queue[idx].pts - totalLatency;
        m_presentevent.notify_all();
    }
}
```

### 容忍度收紧机制详解

这是 PrepareNextRender 最精妙的设计——通过 `m_lateframes` 动态调整对迟到帧的容忍度：

```
容忍度收紧时序示例（30fps 视频，frametime ≈ 33.3ms）：

帧序号  renderPts-framePts  m_lateframes  容忍度 x    行为
────────────────────────────────────────────────────────────
  1       +5ms              0→0           0.98        显示（准时）
  2       +35ms             0→1           0.98        显示（稍迟但在容忍度内）
  3       +40ms             1→2           0.98        显示
  4       +50ms             2→3           0.98        跳过一帧
  5       +55ms             3→5           0.98        跳过一帧
  6       +60ms             5→7           → 0         切换到零容忍！
  7       +10ms             7→0           0           显示（准时归零）

关键转折点：m_lateframes > 6 时进入零容忍模式
- 正常模式（x=0.98）：允许帧迟到 0.98 * 33.3ms ≈ 32.6ms
- 零容忍模式（x=0）：帧迟到任何时间都会被跳过
- 一旦有帧准时到达，m_lateframes 立即归零，恢复正常模式
```

### GetStats 的低通滤波

外部消费者（如 `CalcDropRequirement`）通过 `GetStats()` 获取延迟帧数。该方法对 `m_lateframes` 做除以 10 的低通滤波，防止过度反应：

```cpp
// VideoRenderer.cpp 第 1236-1245 行
bool CRenderManager::GetStats(int& lateframes, double& pts,
                              int& queued, int& discard) {
    CSingleLock lock(m_presentlock);
    lateframes = m_lateframes / 10;  // 低通滤波：除以 10
    pts = m_dvdClock.GetClock();
    queued = m_queued.size();
    discard = m_discard.size();
    return true;
}
// 效果：m_lateframes 需要累积到 10 以上，CalcDropRequirement 才会
// 看到 lateframes >= 1，从而触发更激进的丢帧策略
```

## FrameMove：主循环驱动

`FrameMove()` 是在主线程中每帧调用的核心驱动函数。它串联了配置检查、帧选择、缓冲翻转和废弃帧回收的全部流程。来源文件：`VideoRenderer.cpp` 第 302-355 行。

```cpp
void CRenderManager::FrameMove() {
    UpdateResolution();  // 检查分辨率变化（KrKr2 中为空操作）

    // 1. 配置状态检查
    {
        CSingleLock lock(m_statelock);
        if (m_renderState == STATE_UNCONFIGURED)
            return;  // 未配置，直接返回
        else if (m_renderState == STATE_CONFIGURING) {
            lock.unlock();
            if (!Configure())  // 执行配置（创建渲染器等）
                return;
            FrameWait(50);     // 等待最多 50ms 让帧就绪
        }
    }

    // 2. 帧调度
    {
        CSingleLock lock2(m_presentlock);

        // 队列为空时回到空闲状态
        if (m_queued.empty()) {
            m_presentstep = PRESENT_IDLE;
        }

        // READY 状态：执行帧选择
        if (m_presentstep == PRESENT_READY)
            PrepareNextRender();  // 选取最佳帧，可能跳过迟到帧

        // FLIP 状态：执行缓冲翻转
        if (m_presentstep == PRESENT_FLIP) {
            m_pRenderer->FlipPage(m_presentsource);  // 通知渲染器切换
            m_presentstep = PRESENT_FRAME;
            m_presentevent.notify_all();
        }

        // 3. 回收 discard 队列
        for (auto it = m_discard.begin(); it != m_discard.end();) {
            // 渲染器可能仍需要该缓冲做后处理
            if (!m_pRenderer->NeedBuffer(*it) || !m_bRenderGUI) {
                m_pRenderer->ReleaseBuffer(*it);  // 释放渲染器资源
                m_free.push_back(*it);            // 归还到 free 队列
                it = m_discard.erase(it);
            } else {
                ++it;  // 渲染器仍需要，保留在 discard
            }
        }
        m_bRenderGUI = true;
    }
}
```

**调用频率**：`FrameMove()` 由 Cocos2d 的主循环调度（通常 60fps），与视频帧率（通常 24/30fps）是异步的。这种异步设计允许渲染帧率高于视频帧率时正确处理帧重复，低于视频帧率时自动跳过迟到帧。

## Render：实际渲染执行

`Render()` 方法执行实际的画面渲染，并在完成后推进状态机。来源文件：`VideoRenderer.cpp` 第 774-860 行：

```cpp
void CRenderManager::Render(bool clear, uint32_t flags,
                            uint32_t alpha, bool gui) {
    // 1. 状态检查
    {
        CSingleLock lock(m_statelock);
        if (m_renderState != STATE_CONFIGURED) return;
    }

    // 2. GUI 层检查
    if (!gui && m_pRenderer->IsGuiLayer()) return;

    // 3. 执行渲染（根据呈现方法选择策略）
    if (!gui || m_pRenderer->IsGuiLayer()) {
        SPresent& m = m_Queue[m_presentsource];
        if (m.presentmethod == PRESENT_METHOD_BOB)
            PresentFields(clear, flags, alpha);      // 场间插值
        else if (m.presentmethod == PRESENT_METHOD_WEAVE)
            PresentFields(clear, flags | RENDER_FLAG_WEAVE, alpha);
        else if (m.presentmethod == PRESENT_METHOD_BLEND)
            PresentBlend(clear, flags, alpha);       // 混合呈现
        else
            PresentSingle(clear, flags, alpha);      // ← KrKr2 走这条路径
    }

    // 4. 状态机推进
    {
        CSingleLock lock(m_presentlock);
        if (m_presentstep == PRESENT_FRAME) {
            if (m.presentmethod == PRESENT_METHOD_BOB ||
                m.presentmethod == PRESENT_METHOD_WEAVE)
                m_presentstep = PRESENT_FRAME2;   // 隔行：进入第二场
            else
                m_presentstep = PRESENT_IDLE;     // 逐行：直接空闲
        } else if (m_presentstep == PRESENT_FRAME2) {
            m_presentstep = PRESENT_IDLE;         // 第二场完成
        }

        // 如果空闲但队列非空，重新进入 READY
        if (m_presentstep == PRESENT_IDLE) {
            if (!m_queued.empty())
                m_presentstep = PRESENT_READY;
        }
        m_presentevent.notify_all();
    }
}
```

### PresentSingle 简单呈现

KrKr2 中所有视频都使用逐行扫描（progressive scan），走 `PresentSingle()` 路径：

```cpp
// VideoRenderer.cpp 第 893-902 行
void CRenderManager::PresentSingle(bool clear, uint32_t flags, uint32_t alpha) {
    SPresent& m = m_Queue[m_presentsource];
    // 直接调用渲染器的 RenderUpdate
    m_pRenderer->RenderUpdate(clear, flags, alpha);
}
```

## KrKr2 中的简化：直通渲染器

在 KrKr2 中，CRenderManager 的许多 Kodi 功能被绕过了。关键简化点：

### AddVideoPicture 直通

```cpp
// VideoRenderer.cpp 第 999-1001 行
int CRenderManager::AddVideoPicture(DVDVideoPicture& pic) {
    return m_pRenderer->AddVideoPicture(pic, 0);
    // ↑ 直接返回，下面的代码（第 1002-1051 行）是死代码
    // 原 Kodi 逻辑：从 free 队列取索引 → GetImage → CopyPicture → ReleaseImage
}
```

### WaitForBuffer 直通

```cpp
// VideoRenderer.cpp 第 1086-1088 行
int CRenderManager::WaitForBuffer(volatile std::atomic_bool& bStop, int timeout) {
    return m_pRenderer->WaitForBuffer(bStop, timeout);
    // ↑ 直接委托给底层渲染器（TVPMoviePlayer::WaitForBuffer）
    // 下面第 1090-1138 行是死代码
}
```

### CreateRenderer 简化

```cpp
// VideoRenderer.cpp 第 443-521 行
void CRenderManager::CreateRenderer() {
    if (!m_pRenderer) {
#if 0
        // 原 Kodi 代码：根据格式创建 VAAPI/VDPAU/MediaCodec 等渲染器
        // 全部被 #if 0 禁用
#endif
        // KrKr2：直接通过播放器接口创建
        m_pRenderer = m_playerPort->CreateRenderer();
        if (m_pRenderer) m_pRenderer->PreInit();
    }
}
```

这意味着在 KrKr2 中，CRenderManager 的帧队列管理（free/queued/discard）虽然存在，但 AddVideoPicture 和 WaitForBuffer 绕过了这套队列，直接使用 TVPMoviePlayer 自己的 BitmapPicture 环形缓冲。CRenderManager 在 KrKr2 中的主要价值是 **PrepareNextRender 的迟到帧检测算法** 和 **GetStats 的延迟帧统计**。

## Configure：渲染器配置

`Configure()` 是渲染器的初始化入口。来源文件：`VideoRenderer.cpp` 第 118-200 行（双参数版）和第 203-272 行（无参数版）：

```cpp
// 第一阶段：保存参数，标记为 CONFIGURING
bool CRenderManager::Configure(DVDVideoPicture& picture, float fps,
                               unsigned flags, unsigned int orientation,
                               int buffers) {
    // 检查参数是否变化，未变则直接返回
    CSingleLock lock(m_statelock);
    if (m_width == picture.iWidth && m_height == picture.iHeight &&
        m_fps == fps && m_format == picture.format &&
        m_pRenderer != nullptr) {
        return true;  // 无变化，不需要重新配置
    }

    // 等待当前帧处理完成（最多 5 秒）
    // ...

    // 保存参数
    m_width = picture.iWidth;
    m_height = picture.iHeight;
    m_fps = fps;
    m_format = picture.format;
    m_renderState = STATE_CONFIGURING;  // 标记为配置中

    // 立即进入 READY，让 FrameMove 驱动配置
    m_presentstep = PRESENT_READY;
    m_presentevent.notify_all();
    return true;
}

// 第二阶段：实际执行配置（在 FrameMove 中调用）
bool CRenderManager::Configure() {
    CSingleLock lock(m_statelock);
    CSingleLock lock2(m_presentlock);
    CSingleLock lock3(m_datalock);

    // 创建渲染器
    if (!m_pRenderer) {
        CreateRenderer();
        if (!m_pRenderer) return false;
    }

    // 配置渲染器
    bool result = m_pRenderer->Configure(
        m_width, m_height, m_dwidth, m_dheight,
        m_fps, m_flags, m_format, m_extended_format, m_orientation);

    if (result) {
        // 初始化队列
        CRenderInfo info = m_pRenderer->GetRenderInfo();
        m_QueueSize = std::min({
            info.optimal_buffer_size,
            (int)info.max_buffer_size,
            NUM_BUFFERS,
            m_NumberBuffers > 0 ? m_NumberBuffers : NUM_BUFFERS
        });
        m_QueueSize = std::max(m_QueueSize, 2);  // 最少 2 个缓冲

        // 重置队列状态
        m_queued.clear();
        m_discard.clear();
        m_free.clear();
        m_presentsource = 0;
        for (int i = 1; i < m_QueueSize; i++)
            m_free.push_back(i);

        m_presentstep = PRESENT_IDLE;
        m_lateframes = -1.0;          // 初始值 -1 表示尚未开始计数
        m_renderState = STATE_CONFIGURED;
    }
    return result;
}
```

## VSync 时钟同步

CRenderManager 包含一个可选的 VSync 时钟同步模块 `CClockSync`，用于将视频帧的 PTS 与显示器的刷新周期对齐：

```cpp
// VideoRenderer.cpp 第 1247-1273 行
void CRenderManager::CheckEnableClockSync() {
    double diff = 1.0;
    if (m_fps != 0) {
        float fps = m_fps;
        // 检查显示刷新率是否是视频帧率的整数倍
        if (g_graphicsContext.GetFPS() >= fps)
            diff = fmod(g_graphicsContext.GetFPS(), fps);
        else
            diff = fps - g_graphicsContext.GetFPS();
    }

    // 差异 < 0.01 时启用同步
    // 例如：60Hz 显示 / 30fps 视频 → fmod(60, 30) = 0 → 启用
    // 例如：60Hz 显示 / 24fps 视频 → fmod(60, 24) = 12 → 禁用
    if (diff < 0.01) {
        m_clockSync.m_enabled = true;
    } else {
        m_clockSync.m_enabled = false;
        m_dvdClock.SetVsyncAdjust(0);
    }
}
```

在 KrKr2 中，`g_graphicsContext.GetFPS()` 硬编码返回 60，因此 VSync 同步只在视频帧率为 60 的因子（30、20、15、12、10 等）时才启用。对于常见的 24fps 视频，VSync 同步是禁用的。

## 动手实践

### 实验 1：模拟三队列帧管理

```cpp
#include <cstdio>
#include <deque>
#include <string>

#define NUM_BUFFERS 6

struct SPresent {
    double pts = 0.0;
    bool valid = false;
};

class SimpleRenderManager {
    std::deque<int> m_free, m_queued, m_discard;
    SPresent m_Queue[NUM_BUFFERS];
    int m_presentsource = 0;
    int m_QueueSkip = 0;

public:
    void Init(int queueSize) {
        m_free.clear(); m_queued.clear(); m_discard.clear();
        m_presentsource = 0;
        for (int i = 1; i < queueSize; i++)
            m_free.push_back(i);
        PrintState("初始化");
    }

    bool FlipPage(double pts) {
        if (m_free.empty()) {
            printf("FlipPage: 无空闲缓冲！帧被丢弃 (PTS=%.1f)\n", pts);
            return false;
        }
        int source = m_free.front();
        m_Queue[source].pts = pts;
        m_Queue[source].valid = true;
        m_queued.push_back(m_free.front());
        m_free.pop_front();
        printf("FlipPage: 帧 PTS=%.1f 入队到槽位 %d\n", pts, source);
        PrintState("FlipPage后");
        return true;
    }

    void PrepareNextRender(double renderPts) {
        if (m_queued.empty()) {
            printf("PrepareNextRender: 队列空\n");
            return;
        }

        auto iter = m_queued.begin();
        int idx = *iter;
        ++iter;

        // 简化版：跳过所有已迟到的帧
        while (iter != m_queued.end()) {
            if (renderPts < m_Queue[*iter].pts) break;
            idx = *iter;
            ++iter;
        }

        // 跳过迟到帧
        while (m_queued.front() != idx) {
            printf("  跳过迟到帧 PTS=%.1f (槽位 %d)\n",
                   m_Queue[m_queued.front()].pts, m_queued.front());
            m_discard.push_back(m_queued.front());
            m_queued.pop_front();
            m_QueueSkip++;
        }

        // 切换显示源
        m_discard.push_back(m_presentsource);
        m_presentsource = idx;
        m_queued.pop_front();
        printf("PrepareNextRender: 选中帧 PTS=%.1f (槽位 %d), renderPts=%.1f\n",
               m_Queue[idx].pts, idx, renderPts);
        PrintState("Prepare后");
    }

    void ReleaseDiscard() {
        while (!m_discard.empty()) {
            int idx = m_discard.front();
            m_discard.pop_front();
            m_free.push_back(idx);
        }
        PrintState("释放后");
    }

    void PrintState(const char* label) {
        printf("  [%s] free=[", label);
        for (int i : m_free) printf("%d ", i);
        printf("] queued=[");
        for (int i : m_queued) printf("%d ", i);
        printf("] discard=[");
        for (int i : m_discard) printf("%d ", i);
        printf("] present=%d skip=%d\n", m_presentsource, m_QueueSkip);
    }
};

int main() {
    SimpleRenderManager mgr;
    mgr.Init(4);

    // 模拟解码线程提交帧
    mgr.FlipPage(100.0);  // 帧 1
    mgr.FlipPage(133.3);  // 帧 2
    mgr.FlipPage(166.6);  // 帧 3

    // 模拟渲染线程处理（假设渲染延迟导致 renderPts=150）
    printf("\n--- 渲染时间 150ms，帧 1 迟到 ---\n");
    mgr.PrepareNextRender(150.0);
    mgr.ReleaseDiscard();

    // 继续提交
    mgr.FlipPage(200.0);
    printf("\n--- 渲染时间 170ms ---\n");
    mgr.PrepareNextRender(170.0);
    mgr.ReleaseDiscard();

    return 0;
}
```

### 实验 2：观察容忍度收紧效果

```cpp
#include <cstdio>

int main() {
    int m_lateframes = 0;
    double frametime = 33333.33;  // 30fps，单位微秒

    // 模拟 10 帧的延迟积累
    struct Frame {
        double renderPts;  // 渲染时间
        double framePts;   // 帧 PTS
    };

    Frame frames[] = {
        {100000, 100000},   // 准时
        {135000, 133333},   // 稍迟 1.7ms
        {170000, 166667},   // 稍迟 3.3ms
        {210000, 200000},   // 迟 10ms
        {250000, 233333},   // 迟 16.7ms
        {295000, 266667},   // 迟 28.3ms
        {340000, 300000},   // 迟 40ms
        {390000, 333333},   // 迟 56.7ms → lateframes > 6
        {430000, 366667},   // 迟 63.3ms
        {370000, 370000},   // 准时！→ 归零
    };

    for (int i = 0; i < 10; i++) {
        double diff = frames[i].renderPts - frames[i].framePts;
        int late = (int)(diff * 30.0 / 1000000.0);  // 转换为帧数
        if (late > 0) m_lateframes += late;
        else m_lateframes = 0;

        double x = (m_lateframes <= 6) ? 0.98 : 0;
        double tolerance_ms = x * frametime / 1000.0;

        printf("帧%2d: diff=%+7.1fus late=%d m_lateframes=%2d x=%.2f "
               "容忍度=%.1fms GetStats=%d\n",
               i, diff, late, m_lateframes, x, tolerance_ms,
               m_lateframes / 10);
    }
    return 0;
}
```

## 对照项目源码

相关文件：
- `cpp/core/movie/ffmpeg/VideoRenderer.h` 第 46-279 行 — CRenderManager 完整类定义，含队列、状态机、同步原语
- `cpp/core/movie/ffmpeg/VideoRenderer.cpp` 第 83-98 行 — 构造函数，初始值设置
- `cpp/core/movie/ffmpeg/VideoRenderer.cpp` 第 118-200 行 — `Configure(DVDVideoPicture&)` 保存参数版
- `cpp/core/movie/ffmpeg/VideoRenderer.cpp` 第 203-272 行 — `Configure()` 执行配置版，初始化三队列
- `cpp/core/movie/ffmpeg/VideoRenderer.cpp` 第 302-355 行 — `FrameMove()` 主循环驱动
- `cpp/core/movie/ffmpeg/VideoRenderer.cpp` 第 708-756 行 — `FlipPage()` 帧入队
- `cpp/core/movie/ffmpeg/VideoRenderer.cpp` 第 774-860 行 — `Render()` 渲染执行和状态推进
- `cpp/core/movie/ffmpeg/VideoRenderer.cpp` 第 999-1001 行 — `AddVideoPicture()` 直通简化
- `cpp/core/movie/ffmpeg/VideoRenderer.cpp` 第 1086-1088 行 — `WaitForBuffer()` 直通简化
- `cpp/core/movie/ffmpeg/VideoRenderer.cpp` 第 1141-1222 行 — `PrepareNextRender()` 核心调度算法
- `cpp/core/movie/ffmpeg/VideoRenderer.cpp` 第 1236-1245 行 — `GetStats()` 低通滤波

## 常见错误与解决方案

### 错误 1：忘记释放 discard 队列导致 free 耗尽

**现象**：视频播放几秒后停住，解码线程阻塞在 `WaitForBuffer()`。

**原因**：主线程的 `FrameMove()` 未被正确调用，discard 队列中的帧不能回收到 free。

```cpp
// ❌ 错误：不调用 FrameMove 就直接 Render
while (playing) {
    mgr.Render(true, 0, 255, true);
    // 忘记调用 FrameMove()！
    // discard 队列越来越长，free 队列为空
}

// ✅ 正确：每帧先 FrameMove 再 Render
while (playing) {
    mgr.FrameMove();  // 回收 discard，推进状态
    mgr.Render(true, 0, 255, true);
}
```

### 错误 2：m_lateframes 不归零导致持续丢帧

**现象**：视频播放中偶尔卡顿后，即使帧恢复准时也持续丢帧。

**原因**：`m_lateframes` 使用累加模式，如果迟到帧数一直增加但不出现准时帧，它就一直增长。一旦超过 6，进入零容忍模式后更难恢复。

**解决**：这是 Kodi 原版设计的一个特性——一旦延迟严重，强制零容忍来快速追上。如果需要更宽松的策略，可修改归零条件。

### 错误 3：多线程竞争导致状态不一致

**现象**：偶发崩溃，通常在 `PrepareNextRender()` 中访问空队列。

**原因**：`FlipPage()`（解码线程）和 `FrameMove()`（主线程）同时操作队列但锁的粒度不够。

```cpp
// CRenderManager 使用三把锁保护不同资源：
// m_statelock:   保护 m_renderState, m_pRenderer
// m_presentlock: 保护 m_queued, m_free, m_discard, m_presentstep
// m_datalock:    保护帧数据读写

// 两个线程总是先获取 m_presentlock 再操作队列，保证互斥
```

## 本节小结

- CRenderManager 使用三个 deque（free、queued、discard）管理 NUM_BUFFERS=6 个帧缓冲
- EPRESENTSTEP 状态机有五种状态：IDLE→READY→FLIP→FRAME→IDLE
- FlipPage 在解码线程入队帧，FrameMove 在主线程驱动整个流水线
- PrepareNextRender 的容忍度随 m_lateframes 收紧：≤6 → x=0.98，>6 → x=0
- GetStats 对 m_lateframes 做 /10 低通滤波，防止外部模块过度反应
- KrKr2 中 AddVideoPicture 和 WaitForBuffer 直通底层渲染器，绕过了 Kodi 原有的队列管理

## 练习题与答案

### 题目 1：队列状态推演

给定 QueueSize=4，初始状态 free=[1,2,3], presentsource=0。依次执行以下操作，写出每步后 free/queued/discard/presentsource 的状态：
1. FlipPage(pts=100)
2. FlipPage(pts=133)
3. PrepareNextRender(renderPts=120) → 选中哪一帧？
4. FlipPage(pts=166)
5. ReleaseDiscard

<details>
<summary>查看答案</summary>

```
初始：free=[1,2,3] queued=[] discard=[] present=0

1. FlipPage(100):
   free=[2,3] queued=[1] discard=[] present=0
   (从 free 取 1，放入 queued，m_Queue[1].pts=100)

2. FlipPage(133):
   free=[3] queued=[1,2] discard=[] present=0
   (从 free 取 2，放入 queued，m_Queue[2].pts=133)

3. PrepareNextRender(120):
   renderPts=120 >= m_Queue[1].pts=100 → 满足显示条件
   遍历后续帧：renderPts=120 < m_Queue[2].pts=133 → 不跳过帧 2
   选中帧 1（索引 1）

   free=[3] queued=[2] discard=[0] present=1
   (旧 presentsource=0 进 discard，帧 1 成为新 presentsource)

4. FlipPage(166):
   free=[] queued=[2,3] discard=[0] present=1
   (从 free 取 3，放入 queued)

5. ReleaseDiscard:
   free=[0] queued=[2,3] discard=[] present=1
   (索引 0 从 discard 回到 free)
```

</details>

### 题目 2：为什么 PrepareNextRender 的容忍度阈值是 0.98 而不是 1.0？

<details>
<summary>查看答案</summary>

使用 0.98 而不是 1.0 的原因是为了留出 2% 的安全边际：

```
frametime（30fps）= 33333 微秒
0.98 * frametime = 32667 微秒
差值 = 666 微秒（约 0.67ms）
```

这 0.67ms 的余量用于补偿以下误差：
1. **时钟读取延迟**：`GetClock()` 返回的时间和实际渲染时间有微小差异
2. **线程调度抖动**：主线程可能因操作系统调度略微延迟
3. **浮点精度**：DVD_TIME_BASE 运算的精度损失

如果使用 1.0，一帧刚好在 frametime 边界上的帧可能因为微小的计时误差被错误地跳过。0.98 确保只有真正迟到超过 2% 帧时间的帧才会被跳过。

</details>

### 题目 3：修改 GetStats 的低通滤波系数

当前 GetStats 使用 `m_lateframes / 10` 作为低通滤波。如果你觉得这个阈值太高（需要 10 个迟到帧才触发 CalcDropRequirement），请设计一个改进方案。

<details>
<summary>查看答案</summary>

```cpp
// 方案 1：使用指数移动平均（EMA）替代简单除法
// 优点：对最近的变化更敏感，同时平滑噪声
class SmoothedLateFrames {
    double m_smoothed = 0.0;
    static constexpr double ALPHA = 0.3;  // 平滑系数，越大越敏感

public:
    void Update(int rawLateframes) {
        if (rawLateframes > 0) {
            m_smoothed = ALPHA * rawLateframes + (1.0 - ALPHA) * m_smoothed;
        } else {
            // 准时时快速衰减但不立即归零
            m_smoothed *= (1.0 - ALPHA);
        }
    }

    int GetSmoothed() const {
        return static_cast<int>(m_smoothed);
    }
};

// 方案 2：使用滑动窗口平均
class WindowedLateFrames {
    static constexpr int WINDOW_SIZE = 5;
    int m_history[WINDOW_SIZE] = {};
    int m_pos = 0;
    int m_count = 0;

public:
    void Update(int rawLateframes) {
        m_history[m_pos] = rawLateframes;
        m_pos = (m_pos + 1) % WINDOW_SIZE;
        if (m_count < WINDOW_SIZE) m_count++;
    }

    int GetAverage() const {
        int sum = 0;
        for (int i = 0; i < m_count; i++) sum += m_history[i];
        return sum / m_count;
    }
};

// 使用示例
int main() {
    SmoothedLateFrames ema;
    WindowedLateFrames win;

    int rawValues[] = {0, 0, 3, 5, 8, 2, 0, 0, 4, 0};
    printf("%-6s %-8s %-8s %-10s\n", "Raw", "/10", "EMA", "Window");
    for (int raw : rawValues) {
        ema.Update(raw);
        win.Update(raw);
        printf("%-6d %-8d %-8d %-10d\n",
               raw, raw / 10, ema.GetSmoothed(), win.GetAverage());
    }
    return 0;
}
// EMA 方案在 raw=3 时就能产生非零输出，比 /10 更早触发丢帧策略
```

</details>

## 下一步

[Cocos2d 显示集成](./03-Cocos2d显示集成.md) — 深入分析 Overlay 模式的 TVPYUVSprite 渲染和 Layer 模式的 tTVPBaseTexture 双缓冲机制，了解视频帧如何最终显示到屏幕上。
