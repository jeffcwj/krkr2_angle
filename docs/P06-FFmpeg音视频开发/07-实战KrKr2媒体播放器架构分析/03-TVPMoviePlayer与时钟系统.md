# TVPMoviePlayer 与时钟系统

> **所属模块：** P06-FFmpeg音视频开发
> **前置知识：** [架构总览与 Kodi 遗产](./01-架构总览与Kodi遗产.md)、[BasePlayer 主循环与视频解码](./02-BasePlayer主循环与视频解码.md)
> **预计阅读时间：** 45 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 TVPMoviePlayer 如何通过双重继承桥接 KrKr2 引擎和 Kodi 播放管线
2. 解释 YUV 环形缓冲区的写入（AddVideoPicture）和读取（PresentPicture）机制
3. 分析 CDVDClock 的时间计算公式、速度调整和不连续点处理
4. 理解 CRenderManager 的 PrepareNextRender 帧调度算法和 VSync 时钟校正
5. 追踪一帧视频从解码完成到显示在屏幕上的完整流程

---

## TVPMoviePlayer：两个世界的桥梁

TVPMoviePlayer 是 KrKr2 团队编写的全新代码（非 Kodi 移植），它的唯一使命就是连接两个独立的系统：

- **KrKr2 引擎侧**：通过 `iTVPVideoOverlay` 接口，TJS2 脚本可以调用 `Open()`、`Play()`、`Stop()` 等方法
- **Kodi 播放管线侧**：CRenderManager 需要一个 `CBaseRenderer` 对象来接收解码后的帧数据

**源码位置**：`cpp/core/movie/ffmpeg/KRMoviePlayer.cpp`（382 行）+ `KRMoviePlayer.h`（286 行）

### 双重继承的类定义

```cpp
// 源码：KRMoviePlayer.h（关键部分）
class TVPMoviePlayer
    : public iTVPVideoOverlay    // KrKr2 引擎定义的视频叠加接口
    , public CBaseRenderer       // Kodi 定义的渲染器基类
{
public:
    // ═══ iTVPVideoOverlay 接口方法（KrKr2 侧调用）═══
    void Open(const char* filename);    // 打开视频文件
    void Play();                        // 开始播放
    void Stop();                        // 停止播放
    void SetPosition(double pos);       // 跳转到指定位置
    void SetVisible(bool visible);      // 控制是否可见
    // ... 更多控制方法
    
    // ═══ CBaseRenderer 接口方法（Kodi 播放管线回调）═══
    bool AddVideoPicture(const VideoPicture& pic, int index) override;
    int WaitForBuffer(volatile bool& bStop) override;
    void FlipPage(volatile bool& bStop) override;
    bool IsConfigured() override;
    // ... 更多渲染器方法

private:
    // Kodi 播放器实例
    BasePlayer* m_pPlayer;
    CRenderManager m_renderManager;
    
    // YUV 环形缓冲区
    static constexpr int MAX_BUFFER_COUNT = 4;
    
    uint8_t* m_pBufY[MAX_BUFFER_COUNT];   // Y 平面数据（亮度）
    uint8_t* m_pBufU[MAX_BUFFER_COUNT];   // U 平面数据（色差蓝）
    uint8_t* m_pBufV[MAX_BUFFER_COUNT];   // V 平面数据（色差红）
    double   m_pts[MAX_BUFFER_COUNT];      // 每帧的显示时间戳
    int      m_linesize[3];               // 每行的字节数
    
    int m_curPicture;   // 写入索引（下一帧要写入的位置）
    int m_usedPicture;  // 读取索引（下一帧要显示的位置）
    
    int m_width;        // 视频宽度
    int m_height;       // 视频高度
    
    std::mutex m_mutex;
    std::condition_variable m_cond;
};
```

### 桥接工作流

```
KrKr2 TJS2 脚本
    │
    │  Movie.open("video.mp4")
    │  Movie.play()
    │
    ▼
iTVPVideoOverlay 接口
    │
    ▼
TVPMoviePlayer::Open() / Play()
    │  创建 BasePlayer 实例
    │  配置 CRenderManager（将自身作为 CBaseRenderer 传入）
    │  启动 BasePlayer 线程
    ▼
BasePlayer::Process() 开始运行
    │  读取数据包 → 分发给解码线程
    │  解码线程解码出帧 → 调用 CRenderManager::AddVideoPicture()
    │
    ▼
CRenderManager::AddVideoPicture()
    │  直接委托给 m_pRenderer->AddVideoPicture()
    │  而 m_pRenderer 就是 TVPMoviePlayer 自身
    │
    ▼
TVPMoviePlayer::AddVideoPicture()  ← CBaseRenderer 接口回调
    │  将 YUV 数据写入环形缓冲区
    ▼
    
Cocos2d-x 渲染循环
    │  每帧调用 VideoPresentOverlay::PresentPicture()
    │
    ▼
从环形缓冲区读取帧 → 创建 TVPYUVSprite → 显示在屏幕上
```

---

## YUV 环形缓冲区

环形缓冲区（Ring Buffer / Circular Buffer）是一种固定大小的数据结构，当写入到末尾时自动回绕到开头，形成循环。它在播放器中用于解耦解码线程（生产者）和渲染线程（消费者）的速率差异。

### 缓冲区布局

```
MAX_BUFFER_COUNT = 4

索引:    [0]     [1]     [2]     [3]
状态:   已显示   已显示   当前写入  空闲
         ↑                  ↑
    m_usedPicture=2    m_curPicture=2

写入一帧后：
索引:    [0]     [1]     [2]     [3]
状态:   已显示   已显示   待显示   当前写入
         ↑                          ↑
    m_usedPicture=2            m_curPicture=3

显示一帧后：
索引:    [0]     [1]     [2]     [3]
状态:   已显示   已显示   已显示   当前写入
                          ↑          ↑
                  m_usedPicture=3  m_curPicture=3

可用帧数 = m_curPicture - m_usedPicture
缓冲区满 = (m_curPicture - m_usedPicture) >= MAX_BUFFER_COUNT
缓冲区空 = m_curPicture == m_usedPicture
```

### AddVideoPicture：写入帧数据

**源码位置**：`cpp/core/movie/ffmpeg/KRMoviePlayer.cpp` 第 176-228 行

```cpp
// KRMoviePlayer.cpp TVPMoviePlayer::AddVideoPicture()
// 由视频解码线程（CVideoPlayerVideo）通过 CRenderManager 间接调用

bool TVPMoviePlayer::AddVideoPicture(const VideoPicture& pic, int index) {
    // ════════════════════════════
    // 第 1 步：格式检查
    // ════════════════════════════
    // 只接受 YUV420P 格式（Y/U/V 三个独立平面，U/V 分辨率是 Y 的 1/4）
    // 这是 FFmpeg 解码 H.264 等编码最常见的输出格式
    if (pic.videoBuffer->GetFormat() != AV_PIX_FMT_YUV420P) {
        return false;
    }
    
    // ════════════════════════════
    // 第 2 步：加锁保护
    // ════════════════════════════
    std::unique_lock<std::mutex> lock(m_mutex);
    
    // ════════════════════════════
    // 第 3 步：计算写入位置
    // ════════════════════════════
    int idx = m_curPicture % MAX_BUFFER_COUNT;
    // m_curPicture 是单调递增的计数器
    // 通过 % MAX_BUFFER_COUNT 得到实际的数组索引
    // 例如：m_curPicture=7 → idx=7%4=3
    
    // ════════════════════════════
    // 第 4 步：分配对齐内存（首次或尺寸变化时）
    // ════════════════════════════
    int ySize = m_width * m_height;            // Y 平面大小
    int uvSize = (m_width / 2) * (m_height / 2);  // U/V 平面大小（宽高各为 Y 的一半）
    
    if (m_pBufY[idx] == nullptr) {
        // TJSAlignedAlloc(size, alignment)
        // alignment=4（4 字节对齐）
        // 注意：不是 32 字节对齐，TJSAlignedAlloc 的第二个参数是对齐值
        m_pBufY[idx] = (uint8_t*)TJSAlignedAlloc(ySize, 4);
        m_pBufU[idx] = (uint8_t*)TJSAlignedAlloc(uvSize, 4);
        m_pBufV[idx] = (uint8_t*)TJSAlignedAlloc(uvSize, 4);
    }
    
    // ════════════════════════════
    // 第 5 步：逐行拷贝 YUV 数据
    // ════════════════════════════
    // 为什么逐行拷贝而不用一次 memcpy？
    // 因为 FFmpeg 解码器输出的每行可能有填充字节（padding）
    // 即 pic.iLineSize[0] >= pic.iWidth
    // 需要只拷贝有效数据，跳过填充
    
    uint8_t* srcY = pic.data[0];
    uint8_t* srcU = pic.data[1];
    uint8_t* srcV = pic.data[2];
    
    // Y 平面：逐行拷贝 width 字节（跳过源的填充）
    for (int row = 0; row < m_height; row++) {
        memcpy(m_pBufY[idx] + row * m_width,    // 目标：紧凑存储
               srcY + row * pic.iLineSize[0],    // 源：可能有填充
               m_width);                          // 只拷贝有效宽度
    }
    
    // U 平面：宽高各为 Y 的一半
    int halfWidth = m_width / 2;
    int halfHeight = m_height / 2;
    for (int row = 0; row < halfHeight; row++) {
        memcpy(m_pBufU[idx] + row * halfWidth,
               srcU + row * pic.iLineSize[1],
               halfWidth);
    }
    
    // V 平面：同 U
    for (int row = 0; row < halfHeight; row++) {
        memcpy(m_pBufV[idx] + row * halfWidth,
               srcV + row * pic.iLineSize[2],
               halfWidth);
    }
    
    // ════════════════════════════
    // 第 6 步：保存 PTS 并推进写入索引
    // ════════════════════════════
    m_pts[idx] = pic.pts;
    m_linesize[0] = m_width;
    m_linesize[1] = halfWidth;
    m_linesize[2] = halfWidth;
    
    m_curPicture++;  // 写入索引 +1
    
    // 通知等待中的渲染线程有新帧可用
    m_cond.notify_one();
    
    return true;
}
```

### WaitForBuffer：等待空闲槽位

**源码位置**：`cpp/core/movie/ffmpeg/KRMoviePlayer.cpp` 第 148-159 行

```cpp
// KRMoviePlayer.cpp TVPMoviePlayer::WaitForBuffer()
// 由视频解码线程在 OutputPicture() 中调用

int TVPMoviePlayer::WaitForBuffer(volatile bool& bStop) {
    // 当环形缓冲区满时，解码线程需要等待渲染线程消费掉一帧
    
    while (!bStop) {
        // 检查是否有空闲槽位
        int buffered = m_curPicture - m_usedPicture;
        if (buffered < MAX_BUFFER_COUNT) {
            return MAX_BUFFER_COUNT - buffered;  // 返回剩余空闲槽位数
        }
        
        // 缓冲区满 → 等待 10ms 后重试
        // 这里用轮询（polling）而非条件变量等待
        // 因为 10ms 的间隔足够短，不会造成明显延迟
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
    
    return 0;  // bStop=true，被中断
}
```

**为什么用轮询而不用条件变量？**

条件变量（condition_variable）的 `wait()` 需要持有互斥锁，而 WaitForBuffer 的调用者（视频解码线程）并不需要持有缓冲区的锁。用条件变量会增加锁的持有时间，可能影响渲染线程的读取。10ms 的轮询间隔在实践中完全足够——视频通常是 24-60fps，即每帧 16-42ms，10ms 的轮询不会造成帧延迟。

---

## PresentPicture：渲染到屏幕

`VideoPresentOverlay::PresentPicture()` 是整个管线的最后一环，由 Cocos2d-x 的调度器（Scheduler）在每一帧调用。

**源码位置**：`cpp/core/movie/ffmpeg/KRMoviePlayer.cpp` 第 239-295 行

### 核心渲染逻辑

```cpp
// KRMoviePlayer.cpp VideoPresentOverlay::PresentPicture(float dt)
// 参数 dt = 距上次调用经过的时间（秒）
// 由 Cocos2d-x Scheduler 自动传入

void VideoPresentOverlay::PresentPicture(float dt) {
    // ════════════════════════════
    // 第 1 步：累加时间
    // ════════════════════════════
    m_curpts += dt;
    // 注意：这里使用 delta time 累加，而不是查询播放器时钟
    // 这是一个重要的设计选择——PresentPicture 独立于 CDVDClock，
    // 用自己的累计时间来控制帧的显示节奏
    
    // ════════════════════════════
    // 第 2 步：检查是否有可显示的帧
    // ════════════════════════════
    if (m_usedPicture >= m_pPlayer->m_curPicture) {
        return;  // 环形缓冲区为空，无帧可显示
    }
    
    // ════════════════════════════
    // 第 3 步：跳过已过期的帧
    // ════════════════════════════
    // 如果累积延迟导致多帧过期，跳到最新的可显示帧
    while (m_usedPicture + 1 < m_pPlayer->m_curPicture) {
        int nextIdx = (m_usedPicture + 1) % MAX_BUFFER_COUNT;
        double nextPts = m_pPlayer->m_pts[nextIdx];
        
        // 将 PTS 从微秒（DVD_TIME_BASE）转换为秒
        double nextPtsSec = nextPts / DVD_TIME_BASE;
        
        if (nextPtsSec <= m_curpts) {
            // 下一帧也已经到了显示时间 → 跳过当前帧
            m_usedPicture++;
        } else {
            break;  // 下一帧还没到时间 → 显示当前帧
        }
    }
    
    // ════════════════════════════
    // 第 4 步：获取当前帧数据
    // ════════════════════════════
    int idx = m_usedPicture % MAX_BUFFER_COUNT;
    
    // ════════════════════════════
    // 第 5 步：创建或更新 YUV 精灵
    // ════════════════════════════
    if (m_pYUVSprite == nullptr) {
        // 首次显示：创建 TVPYUVSprite
        // TVPYUVSprite 内部使用 OpenGL shader 做 YUV→RGB 转换
        m_pYUVSprite = TVPYUVSprite::create(
            m_pPlayer->m_width, m_pPlayer->m_height);
        this->addChild(m_pYUVSprite);
    }
    
    // 更新 YUV 纹理数据
    m_pYUVSprite->updateYUV(
        m_pPlayer->m_pBufY[idx], m_pPlayer->m_linesize[0],  // Y 平面
        m_pPlayer->m_pBufU[idx], m_pPlayer->m_linesize[1],  // U 平面
        m_pPlayer->m_pBufV[idx], m_pPlayer->m_linesize[2],  // V 平面
        m_pPlayer->m_width, m_pPlayer->m_height
    );
    
    // ════════════════════════════
    // 第 6 步：缩放到显示区域
    // ════════════════════════════
    // GetBounds() 返回视频应该显示的屏幕区域
    auto bounds = GetBounds();
    float scaleX = bounds.size.width / m_pPlayer->m_width;
    float scaleY = bounds.size.height / m_pPlayer->m_height;
    m_pYUVSprite->setScale(scaleX, scaleY);
    m_pYUVSprite->setPosition(bounds.origin);
    
    // ════════════════════════════
    // 第 7 步：推进读取索引
    // ════════════════════════════
    m_usedPicture++;
}
```

### Delta-Time vs 直接查询时钟

PresentPicture 使用 `m_curpts += dt`（delta-time 累加）而非直接查询 `CDVDClock::GetClock()` 来控制渲染时机。这两种方式的区别：

| 对比维度 | Delta-Time 累加 | 直接查询时钟 |
|---------|----------------|-------------|
| 精度 | 取决于帧率（60fps → 16ms 精度） | 微秒精度 |
| 暂停行为 | Cocos2d-x 暂停时 dt=0 自动停止 | 需要手动检查暂停状态 |
| 速度控制 | 不受 CDVDClock 速度影响 | 自动跟随时钟速度 |
| 复杂度 | 简单，无跨线程访问 | 需要线程安全的时钟读取 |
| Seek 后 | 需要手动重置 m_curpts | 自动从新位置开始 |

**KrKr2 选择 delta-time 的原因**：PresentPicture 运行在 Cocos2d-x 主线程中，与游戏逻辑共享帧循环。使用 dt 累加可以与 Cocos2d-x 的暂停/恢复机制无缝配合——当游戏暂停时，Scheduler 停止调用 PresentPicture，视频自然也暂停了。

---

## CDVDClock：主时钟系统

CDVDClock 是播放器的时间基准，所有音视频同步判断都依赖它。

**源码位置**：`cpp/core/movie/ffmpeg/Clock.cpp`（276 行）+ `Clock.h`（97 行）

### 关键常量

```cpp
// Clock.h 中定义的时钟常量
#define DVD_TIME_BASE       1000000.0    // 时间基准：1秒 = 1,000,000（微秒）
#define DVD_NOPTS_VALUE     0            // 无效 PTS 标记（注意是 0，不是负数）
#define DVD_PLAYSPEED_NORMAL 1000        // 正常播放速度
#define DVD_PLAYSPEED_PAUSE  0           // 暂停

// 构造函数中的默认帧时间
// m_frameTime = DVD_TIME_BASE / 60.0
// 即默认假设 60fps
```

### 时钟的核心算法

CDVDClock 的 `GetClock()` 返回当前播放位置（以微秒为单位）。它不存储"当前时间"，而是通过基准时间 + 经过时间来动态计算：

```cpp
// Clock.cpp CDVDClock::GetClock()
// 源码：简化后的核心逻辑

double CDVDClock::GetClock() {
    std::lock_guard<std::mutex> lock(m_critSection);
    
    // 获取系统高精度时钟
    int64_t now = CurrentHostCounter();
    
    // 计算自上次更新以来经过的系统时间
    int64_t delta = now - m_lastSystemTime;
    
    // 累积速度调整
    // m_speedAdjust 是 VSync 校正引入的微调系数
    m_systemAdjust += m_speedAdjust * delta;
    m_lastSystemTime = now;
    
    // 计算经过的播放时间（考虑播放速度）
    // m_systemFrequency 是系统时钟频率（每秒的 tick 数）
    // 当速度为 DVD_PLAYSPEED_NORMAL (1000) 时，播放速度 = 1x
    double elapsed = (double)(now - m_startClock + m_systemAdjust)
                     / m_systemFrequency;
    
    // 最终时钟 = 不连续点的值 + 经过的时间 × (速度因子)
    return m_iDisc + elapsed * DVD_TIME_BASE;
}
```

### 速度调整 SetSpeed

```cpp
// Clock.cpp CDVDClock::SetSpeed(int iSpeed)
// iSpeed 的单位：1000 = 正常速度，2000 = 2倍速，500 = 0.5倍速

void CDVDClock::SetSpeed(int iSpeed) {
    std::lock_guard<std::mutex> lock(m_critSection);
    
    // 速度变化时，需要重新计算时钟参数以保证时间连续性
    // 否则速度切换的瞬间时钟会出现跳变
    
    if (m_iSpeed != iSpeed) {
        // 1. 在旧速度下记录当前时钟值
        double clock = GetClockInternal();
        
        // 2. 计算新的系统频率
        // 原理：通过改变"每秒多少个系统 tick"来实现速度变化
        // 正常速度：m_systemFrequency 不变
        // 2倍速：m_systemFrequency 减半 → 同样数量的 tick 对应 2 倍的播放时间
        // 0.5倍速：m_systemFrequency 翻倍 → 同样数量的 tick 对应 0.5 倍的播放时间
        int64_t newFreq = m_systemFrequency * DVD_PLAYSPEED_NORMAL / iSpeed;
        
        // 3. 重新计算 m_startClock 以保证时间连续性
        // 让 GetClock() 在速度切换前后返回相同的值
        m_startClock = CurrentHostCounter();
        m_iDisc = clock;  // 以当前时钟值作为新的不连续点
        m_systemAdjust = 0;
        
        m_iSpeed = iSpeed;
    }
}
```

### 不连续点 Discontinuity

当播放器执行 Seek 操作时，播放位置发生跳变（从 00:30:00 跳到 01:30:00），时钟需要"不连续地"跳转到新位置：

```cpp
// Clock.cpp CDVDClock::Discontinuity(double clock)
// 参数 clock = 新的播放位置（微秒）

void CDVDClock::Discontinuity(double clock) {
    std::lock_guard<std::mutex> lock(m_critSection);
    
    // 重置时钟到新的起点
    m_startClock = CurrentHostCounter();  // 记录当前系统时间
    m_iDisc = clock;                       // 设置新的基准值
    m_systemAdjust = 0;                    // 清除所有累积的微调
    
    // Discontinuity 之后，GetClock() 会返回：
    //   clock + (当前系统时间 - m_startClock) / m_systemFrequency * DVD_TIME_BASE
    // 由于 m_startClock 刚刚被设置为当前时间，差值 ≈ 0
    // 所以 GetClock() ≈ clock
    // 随着时间推移，返回值以播放速度线性增长
}
```

### ErrorAdjust：VSync 时钟微调

当渲染器检测到系统帧率偏差时，会通过 `ErrorAdjust()` 微调时钟，使画面与显示器刷新率（VSync）对齐：

```cpp
// Clock.cpp CDVDClock::ErrorAdjust(double error)
// error = 期望显示时间 - 实际显示时间（微秒）

void CDVDClock::ErrorAdjust(double error) {
    std::lock_guard<std::mutex> lock(m_critSection);
    
    if (m_bVsyncMode) {
        // VSync 模式：使用非对称阈值
        // 允许画面稍微提前（人眼不敏感），但不允许太晚
        
        if (error > 20000) {        // 提前超过 20ms
            m_speedAdjust -= 0.01;  // 稍微减慢时钟
        }
        else if (error < -27000) {  // 延迟超过 27ms
            m_speedAdjust += 0.01;  // 稍微加快时钟
        }
        
        // 非对称阈值的原因：
        // - 画面提前 20ms 到达：人眼几乎察觉不到
        // - 画面延迟 27ms 到达：可能导致丢帧或画面撕裂
        // 所以对延迟的容忍度更高一些（27ms > 20ms）
    }
    else {
        // 非 VSync 模式：直接跳转
        // 不做渐进调整，而是直接修正时钟偏差
        m_startClock += (int64_t)(error / DVD_TIME_BASE * m_systemFrequency);
    }
}
```

### 时钟精度分析

```
CDVDClock 时间精度层次：

  1. 系统层：CurrentHostCounter() — 纳秒级精度
     Windows:  QueryPerformanceCounter()  → ~100ns 精度
     Linux:    clock_gettime(MONOTONIC)   → ~1ns 精度
     macOS:    mach_absolute_time()       → ~1ns 精度
     Android:  clock_gettime(MONOTONIC)   → ~1ns 精度

  2. 时钟层：DVD_TIME_BASE = 1000000 → 微秒精度
     足够区分 1μs 的时间差
     1080p@60fps 的帧间隔 = 16,667μs → 远大于精度

  3. 渲染层：VSync 周期 = 16.67ms（60Hz）或 6.94ms（144Hz）
     ErrorAdjust 的阈值 = 20ms/27ms → 远大于 VSync 周期
     说明它是在"帧"级别做调整，不是精确到像素行
```

---

## CRenderManager：帧调度与 VSync 对齐

虽然在 KrKr2 中 CRenderManager 的帧队列被旁路（AddVideoPicture/WaitForBuffer 直接委托给 TVPMoviePlayer），但 `PrepareNextRender()` 和 `FrameMove()` 的帧调度逻辑仍然在运行。

**源码位置**：`cpp/core/movie/ffmpeg/VideoRenderer.cpp`（1274 行）

### FrameMove：每帧调用

```cpp
// VideoRenderer.cpp CRenderManager::FrameMove()
// 源码：第 302-355 行
// 每帧被 Cocos2d-x 调度器调用

void CRenderManager::FrameMove() {
    
    // ─── 状态机驱动 ───
    switch (m_presentstate) {
        
    case PRESENT_IDLE:
        // 空闲 → 没有帧需要渲染
        break;
        
    case PRESENT_READY:
        // 有帧等待显示 → 调用 PrepareNextRender
        if (PrepareNextRender()) {
            m_presentstate = PRESENT_FLIP;
        }
        break;
        
    case PRESENT_FLIP:
        // 执行翻页（将新帧提交给 GPU）
        FlipPage();
        m_presentstate = PRESENT_READY;
        break;
    }
    
    // ─── 回收已显示的帧 ───
    // 将 discard 队列中的帧移回 free 队列
    while (!m_discard.empty()) {
        m_free.push_back(m_discard.front());
        m_discard.pop_front();
    }
}
```

### PrepareNextRender：核心调度算法

`PrepareNextRender()` 决定应该显示哪一帧。它的核心任务是：在时钟指定的时间点选择最合适的帧，跳过已经迟到的帧。

**源码位置**：`cpp/core/movie/ffmpeg/VideoRenderer.cpp` 第 1141-1222 行

```cpp
// VideoRenderer.cpp CRenderManager::PrepareNextRender()
// 核心帧调度算法

bool CRenderManager::PrepareNextRender() {
    // ════════════════════════════
    // 第 1 步：检查队列
    // ════════════════════════════
    if (m_queued.empty()) return false;  // 没有帧可显示
    
    // ════════════════════════════
    // 第 2 步：计算渲染目标时间
    // ════════════════════════════
    double clock = m_dvdClock.GetClock();
    double renderPts = clock;
    
    // 加上显示延迟补偿
    // 帧从提交到实际显示在屏幕上有延迟（通常 1-2 个 VSync 周期）
    renderPts += m_displayLatency;
    
    // ════════════════════════════
    // 第 3 步：遍历队列，找到最合适的帧
    // ════════════════════════════
    // 从队列头开始扫描，跳过所有已经迟到的帧
    
    while (m_queued.size() > 1) {
        // 取队首帧的 PTS
        double framePts = m_pBuffers[m_queued.front()].pts;
        double diff = renderPts - framePts;
        // diff > 0 表示帧已迟到（当前时钟已经超过了帧的显示时间）
        
        if (diff <= 0) {
            break;  // 帧还没到显示时间，停止扫描
        }
        
        // ─── 智能迟到帧处理 ───
        double frametime = m_presentstep;  // 帧间隔
        
        // 计算宽限时间（grace period）
        // 根据连续迟到帧数动态调整
        double x;  // 宽限系数
        
        if (m_lateframes <= 6) {
            x = 0.98;  // 轻微迟到：允许约 1 帧的宽限
        }
        else if (m_lateframes <= 10) {
            // 线性递减：6帧时 0.98 → 10帧时 0
            x = 0.98 * (1.0 - (m_lateframes - 6) / 4.0);
        }
        else {
            x = 0;  // 严重迟到：零容忍
        }
        
        double grace = x * frametime;
        
        if (diff > grace) {
            // 迟到超过宽限 → 跳过此帧
            m_lateframes++;
            
            // 将此帧移到 discard 队列
            m_discard.push_back(m_queued.front());
            m_queued.pop_front();
            continue;  // 检查下一帧
        }
        
        break;  // 在宽限范围内 → 显示此帧
    }
    
    // ════════════════════════════
    // 第 4 步：选定帧
    // ════════════════════════════
    if (m_queued.empty()) return false;
    
    // 将当前显示帧移到 discard
    if (m_presentsource >= 0) {
        m_discard.push_back(m_presentsource);
    }
    
    // 取出队首帧作为下一个显示帧
    m_presentsource = m_queued.front();
    m_queued.pop_front();
    m_presentpts = m_pBuffers[m_presentsource].pts;
    
    // 如果没有迟到，重置迟到计数
    if (renderPts - m_presentpts <= 0) {
        m_lateframes = 0;
    }
    
    return true;
}
```

### VSync 时钟同步

CRenderManager 还负责 VSync 时钟同步——通过统计实际渲染时间与理想渲染时间的偏差，微调 CDVDClock：

```cpp
// VideoRenderer.cpp — VSync 同步逻辑（简化）
// 运行在 PrepareNextRender() 或 FlipPage() 附近

void CRenderManager::UpdateClockSync() {
    // 统计最近 30 帧的渲染时间偏差
    static const int SYNC_WINDOW = 30;
    
    // 当前帧的实际渲染时间
    double actualTime = CurrentHostCounter() / m_systemFrequency;
    
    // 期望的渲染时间（基于 PTS 和帧率）
    double expectedTime = m_presentpts / DVD_TIME_BASE;
    
    // 累积偏差
    m_syncError += (actualTime - expectedTime);
    m_syncFrameCount++;
    
    if (m_syncFrameCount >= SYNC_WINDOW) {
        // 每 30 帧计算一次平均偏差
        double avgError = m_syncError / SYNC_WINDOW;
        
        // 通过 CDVDClock::ErrorAdjust() 微调时钟
        // 让时钟速度略微调整以对齐 VSync
        m_dvdClock.ErrorAdjust(avgError * DVD_TIME_BASE);
        
        // 重置统计
        m_syncError = 0;
        m_syncFrameCount = 0;
    }
}
```

### Configure：初始化渲染器

```cpp
// VideoRenderer.cpp CRenderManager::Configure()
// 源码：第 203-272 行
// 首次收到帧或视频参数变化时调用

void CRenderManager::Configure(const VideoPicture& picture) {
    // 1. 配置渲染器（TVPMoviePlayer）
    m_pRenderer->Configure(picture);
    
    // 2. 计算帧缓冲队列大小
    // 取以下三个值的最小值：
    //   - 渲染器建议的最优值
    //   - NumberBuffers（配置参数）
    //   - NUM_BUFFERS（硬编码上限，值为 6）
    m_QueueSize = std::min({
        m_pRenderer->GetOptimalBufferSize(),
        m_NumberBuffers,
        NUM_BUFFERS
    });
    
    // 3. 初始化 free 队列
    // 所有缓冲区初始状态为 free
    m_free.clear();
    m_queued.clear();
    m_discard.clear();
    for (int i = 0; i < m_QueueSize; i++) {
        m_free.push_back(i);
    }
    
    // 4. 设置帧时间
    m_presentstep = (double)DVD_TIME_BASE / picture.iFrameRate;
}
```

### FlipPage：提交帧

```cpp
// VideoRenderer.cpp CRenderManager::FlipPage()
// 源码：第 708-756 行

void CRenderManager::FlipPage(volatile bool& bStop, double pts) {
    std::unique_lock<std::mutex> lock(m_presentlock);
    
    // 1. 将帧从 free 队列移到 queued 队列
    if (!m_free.empty()) {
        int idx = m_free.front();
        m_free.pop_front();
        
        // 记录此帧的 PTS
        m_pBuffers[idx].pts = pts;
        
        // 加入显示队列
        m_queued.push_back(idx);
    }
    
    // 2. 标记有新帧可渲染
    m_presentstate = PRESENT_READY;
    
    // 3. 如果配置了等待渲染完成，在这里阻塞
    // 直到 FrameMove() 调用了 PrepareNextRender()
}
```

---

## 帧缓冲队列生命周期图

```
CRenderManager 的帧在四个队列之间流转：

  ┌─────┐   FlipPage()   ┌────────┐  PrepareNextRender()  ┌─────────────┐
  │ free │ ─────────────→ │ queued │ ────────────────────→ │ presentsource│
  └──┬──┘                └────────┘                       └──────┬──────┘
     ↑                                                           │
     │                    ┌─────────┐                            │
     └────────────────────│ discard │←───────────────────────────┘
        FrameMove()       └─────────┘    PrepareNextRender()
        回收完成                         旧帧进入 discard

但是！在 KrKr2 中：

  CRenderManager::AddVideoPicture() → m_pRenderer->AddVideoPicture()
  CRenderManager::WaitForBuffer()   → m_pRenderer->WaitForBuffer()

  m_pRenderer 就是 TVPMoviePlayer，它有自己的 YUV 环形缓冲区。
  所以 CRenderManager 的 free/queued/discard 队列在实际帧数据流中被旁路了。
  
  PrepareNextRender() 和 FrameMove() 的调度逻辑仍然运行，
  但它们处理的是"帧元数据"（PTS、时序信息），不是帧像素数据。
```

---

## 常见错误与排查

### 错误 1：环形缓冲区溢出

```
症状：AddVideoPicture 写入的帧被尚未显示的帧覆盖，画面出现帧混乱

原因分析：
  m_curPicture - m_usedPicture >= MAX_BUFFER_COUNT 时
  写入的帧会覆盖还未被 PresentPicture 读取的帧。

  正常情况下 WaitForBuffer() 应该阻塞直到有空闲槽位。
  如果 WaitForBuffer() 被 bStop=true 中断后
  仍然调用了 AddVideoPicture()，就可能发生溢出。

排查步骤：
  1. 在 AddVideoPicture 入口处添加断言：
     assert(m_curPicture - m_usedPicture < MAX_BUFFER_COUNT);
  2. 检查 WaitForBuffer 的返回值是否被正确检查
  3. 检查 OutputPicture 中 WaitForBuffer 返回 0 后是否跳过了 AddVideoPicture

修复思路：
  在 AddVideoPicture 入口处增加防御性检查：
  if (m_curPicture - m_usedPicture >= MAX_BUFFER_COUNT) return false;
```

### 错误 2：PresentPicture 的 dt 累加漂移

```
症状：长时间播放后音画逐渐不同步（每分钟偏移约 10-50ms）

原因分析：
  PresentPicture 使用 m_curpts += dt 累加时间，
  而 dt 是 Cocos2d-x 调度器提供的帧间隔。
  由于浮点数精度限制，长时间累加会产生舍入误差。
  
  例如 60fps 时，每帧 dt = 0.016667 秒
  经过 3600 帧（1 分钟）：
    理想值 = 60.0 秒
    实际累加 ≈ 59.99988... 秒 或 60.00012... 秒
    误差 ≈ 0.1-0.2ms

  1 小时后误差可达 6-12ms，虽然通常可接受，
  但如果 Cocos2d-x 的 dt 本身就有波动，误差会更大。

排查步骤：
  1. 在 PresentPicture 中记录每次的 m_curpts 和对应的系统时间
  2. 比较累积的 m_curpts 与实际经过的系统时间
  3. 如果偏差持续增长，说明是 dt 累加漂移

修复思路：
  定期（每 N 帧）用系统时间校正 m_curpts：
  if (frameCount % 300 == 0) {
      m_curpts = (CurrentHostCounter() - m_startTime) / 1e6;
  }
```

### 错误 3：TJSAlignedAlloc 对齐值不足

```
症状：在某些 ARM 平台（Android）上，YUV→RGB 转换出现随机崩溃

原因分析：
  KrKr2 使用 TJSAlignedAlloc(size, 4) 分配 4 字节对齐的内存。
  但某些 NEON（ARM 的 SIMD 指令集）指令要求 16 字节对齐。
  如果 YUV→RGB 的 shader 或 CPU 路径使用了 NEON 加速，
  4 字节对齐的缓冲区可能导致对齐异常（alignment fault）。

排查步骤：
  1. 检查崩溃日志中的地址是否不满足 16 字节对齐
  2. 确认 TVPYUVSprite 的 updateYUV 是否使用了 NEON 指令
  3. 在 x86 平台上通常不会崩溃（SSE 指令可以处理非对齐访问，只是较慢）

修复思路：
  将 TJSAlignedAlloc 的对齐值从 4 改为 16 或 32：
  m_pBufY[idx] = (uint8_t*)TJSAlignedAlloc(ySize, 16);
```

---

## 动手实践

### 实践 1：分析环形缓冲区的容量选择

`MAX_BUFFER_COUNT = 4` 意味着环形缓冲区最多缓存 4 帧。计算以下场景下，这 4 帧能覆盖多少时间：

- 24fps 电影：每帧 ___ms，4 帧 = ___ms
- 30fps 视频：每帧 ___ms，4 帧 = ___ms
- 60fps 视频：每帧 ___ms，4 帧 = ___ms

思考：如果 Cocos2d-x 渲染线程因为其他 UI 操作卡顿了 100ms，哪种帧率的视频会出现缓冲区溢出？

### 实践 2：修改环形缓冲区大小

将 `MAX_BUFFER_COUNT` 从 4 改为 8，分析：
1. 内存开销增加多少？（假设 1080p 视频，YUV420P 格式）
2. 对抗渲染线程卡顿的能力提升多少？
3. 有什么副作用？（提示：考虑延迟）

---

## 对照项目源码

本节涉及的所有源码文件及关键位置：

- `cpp/core/movie/ffmpeg/KRMoviePlayer.h` — TVPMoviePlayer 类定义
  - 第 1-286 行：完整头文件，包含双重继承声明

- `cpp/core/movie/ffmpeg/KRMoviePlayer.cpp` — TVPMoviePlayer 实现
  - 第 148-159 行：`WaitForBuffer()` 轮询等待
  - 第 176-228 行：`AddVideoPicture()` YUV 环形缓冲写入
  - 第 239-295 行：`PresentPicture()` 渲染到屏幕
  - 第 315-328 行：`SetWindow()` Cocos2d-x 调度器注册

- `cpp/core/movie/ffmpeg/Clock.h` — CDVDClock 类定义 + 常量
  - 第 1-97 行：DVD_TIME_BASE、DVD_NOPTS_VALUE 等

- `cpp/core/movie/ffmpeg/Clock.cpp` — CDVDClock 实现
  - `GetClock()`：时间计算公式
  - `SetSpeed()`：速度调整 + 时间连续性保证
  - `Discontinuity()`：Seek 后的时钟重置
  - `ErrorAdjust()`：VSync 微调（非对称阈值 20ms/27ms）

- `cpp/core/movie/ffmpeg/VideoRenderer.cpp` — CRenderManager
  - 第 203-272 行：`Configure()` 渲染器初始化
  - 第 302-355 行：`FrameMove()` 每帧调用
  - 第 708-756 行：`FlipPage()` 帧提交
  - 第 999 行：`AddVideoPicture()` 委托
  - 第 1086-1088 行：`WaitForBuffer()` 委托
  - 第 1141-1222 行：`PrepareNextRender()` 帧调度核心

---

## 本节小结

- **TVPMoviePlayer** 通过双重继承（`iTVPVideoOverlay` + `CBaseRenderer`）桥接 KrKr2 和 Kodi 两个系统
- **YUV 环形缓冲区**（4 帧容量）解耦解码线程和渲染线程，写入端（AddVideoPicture）使用 mutex 保护，读取端（PresentPicture）通过 delta-time 累加控制帧节奏
- **AddVideoPicture** 只接受 YUV420P 格式，使用 `TJSAlignedAlloc(size, 4)` 分配 4 字节对齐内存，逐行拷贝以跳过 FFmpeg 的行填充
- **PresentPicture** 使用 delta-time 累加（`m_curpts += dt`）而非直接查询时钟，这与 Cocos2d-x 的暂停/恢复机制无缝配合
- **CDVDClock** 基于系统高精度时钟计算播放位置，支持速度调整（SetSpeed）和不连续点处理（Discontinuity），VSync 校正使用非对称阈值（20ms 提前 / 27ms 延迟）
- **CRenderManager** 的 PrepareNextRender 根据连续迟到帧数动态调整丢帧宽限度（≤6 帧宽容，>10 帧零容忍），但在 KrKr2 中其帧队列被旁路，TVPMoviePlayer 直接管理帧数据

---

## 练习题与答案

### 题目 1：环形缓冲区容量计算

一个 1080p（1920×1080）YUV420P 视频，MAX_BUFFER_COUNT = 4 帧。计算环形缓冲区的总内存开销。

提示：YUV420P 格式中，Y 平面大小 = width × height，U 平面大小 = V 平面大小 = width/2 × height/2。

<details>
<summary>查看答案</summary>

```
每帧内存：
  Y 平面 = 1920 × 1080 = 2,073,600 字节 ≈ 1.98 MB
  U 平面 = 960 × 540 = 518,400 字节 ≈ 0.49 MB
  V 平面 = 960 × 540 = 518,400 字节 ≈ 0.49 MB
  
  每帧总计 = 2,073,600 + 518,400 + 518,400 = 3,110,400 字节 ≈ 2.97 MB

4 帧总内存：
  4 × 3,110,400 = 12,441,600 字节 ≈ 11.87 MB

对于 4K（3840×2160）视频：
  每帧 = 3840 × 2160 × 1.5 = 12,441,600 字节 ≈ 11.87 MB
  4 帧 = 49,766,400 字节 ≈ 47.46 MB

结论：1080p 的 4 帧缓冲约 12MB，在现代设备上完全可接受。
但 4K 的 4 帧缓冲约 47MB，在移动设备上可能较大。
```

</details>

### 题目 2：CDVDClock 速度计算

CDVDClock 的 `GetClock()` 返回的时间以微秒为单位（DVD_TIME_BASE = 1000000.0）。如果：
- `m_iDisc = 60000000`（即 60 秒位置）
- `m_startClock = T0`
- 当前系统时间 = T0 + 500000 个系统时钟 tick
- `m_systemFrequency = 1000000`（每秒 100万 tick）
- `m_iSpeed = 2000`（2 倍速）

此时 `GetClock()` 返回多少？对应视频的哪个时间点？

<details>
<summary>查看答案</summary>

```
代入 GetClock() 公式：

elapsed = (now - m_startClock) / m_systemFrequency
        = 500000 / 1000000
        = 0.5 秒

但速度是 2 倍速，所以播放时间前进了 2 × 0.5 = 1.0 秒

返回值 = m_iDisc + elapsed * DVD_TIME_BASE * (m_iSpeed / DVD_PLAYSPEED_NORMAL)
       = 60000000 + 0.5 * 1000000 * (2000 / 1000)
       = 60000000 + 0.5 * 1000000 * 2
       = 60000000 + 1000000
       = 61000000 微秒

对应视频时间点 = 61000000 / 1000000 = 61 秒 = 1分01秒

解释：从 60 秒位置开始，以 2 倍速播放了 0.5 秒（实际时间），
播放位置前进了 1.0 秒（播放时间），到达 61 秒。

注意：实际代码中速度是通过修改 m_systemFrequency 实现的
（SetSpeed 将频率改为 m_systemFrequency * 1000 / iSpeed），
而不是在 GetClock 中直接乘以速度因子。但最终效果等价。
```

</details>

### 题目 3：PresentPicture 的帧跳过分析

假设视频帧率 30fps（帧间隔 33.33ms），环形缓冲区中有以下 3 帧：

| 帧索引 | PTS (秒) |
|--------|---------|
| 0 | 1.000 |
| 1 | 1.033 |
| 2 | 1.067 |

如果 Cocos2d-x 渲染线程因为 UI 操作卡顿了 80ms，导致 PresentPicture 被调用时 `m_curpts = 1.080`。PresentPicture 会显示哪一帧？为什么？

<details>
<summary>查看答案</summary>

PresentPicture 会显示**帧 2**（PTS = 1.067 秒）。

**流程分析：**

1. 进入 PresentPicture 时 `m_curpts = 1.080`
2. 第 3 步的 while 循环检查是否可以跳过帧：
   - 当前帧 = 帧 0（PTS = 1.000），下一帧 = 帧 1（PTS = 1.033）
   - 1.033 ≤ 1.080 → 帧 1 也已过期，跳过帧 0
   - 当前帧 = 帧 1（PTS = 1.033），下一帧 = 帧 2（PTS = 1.067）
   - 1.067 ≤ 1.080 → 帧 2 也已过期，跳过帧 1
   - 当前帧 = 帧 2，没有更多帧了 → 退出循环
3. 显示帧 2

**结果**：帧 0 和帧 1 被跳过（从未显示），帧 2 被显示。

**用户感知**：卡顿 80ms 后画面直接跳到帧 2，跳过了中间 2 帧。如果内容是快速运动的场景，可能会看到轻微的"跳跃"；如果是静态或缓慢变化的场景，几乎感知不到。

**如果缓冲区中没有更多帧（帧 2 是最后一帧）**：直接显示帧 2，不等待新帧。下次 PresentPicture 调用时如果仍然没有新帧，会直接返回（环形缓冲区为空）。

</details>

---

## 下一步

[调试技巧与实践](./04-调试技巧与实践.md) — 播放器常见问题诊断方法、性能调优、内存泄漏排查，以及如何在 KrKr2 项目中实际修改和测试播放器代码。
