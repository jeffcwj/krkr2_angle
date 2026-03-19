# 5.3 与原版 Kodi 的差异

## 本节目标

- 系统性了解 KrKr2 视频播放器相对于原版 XBMC/Kodi 的裁剪和修改
- 识别源码中的 `#if 0` 死代码块及其原始功能
- 理解哪些 FFmpeg API 被降级使用及其影响
- 掌握 KrKr2 特有的渲染集成层（Overlay/Layer 模式）
- 能够在阅读源码时快速区分"活代码"与"死代码"

---

## 5.3.1 为什么需要了解差异

KrKr2 的视频播放器并非从零编写，而是从 XBMC/Kodi 的 CDVDPlayer 模块
移植而来。移植过程中，大量与 Kodi 媒体中心功能相关的代码被裁剪或禁用。
了解这些差异有三个实际意义：

1. **避免困惑**：阅读源码时不会被 `#if 0` 块中的代码误导
2. **定位问题**：调试时知道哪些功能路径是不可达的
3. **评估可行性**：如果需要恢复某个功能，知道需要多少工作量

### 差异全景图

```
原版 Kodi CDVDPlayer
├── 视频解码 ✅ 保留
├── 音频解码 ✅ 保留
├── A/V 同步 ✅ 保留（部分简化）
├── 解封装器 ✅ 保留
├── 字幕系统 ❌ 完全禁用
├── DVD/蓝光菜单 ❌ 完全禁用
├── Teletext/RDS ❌ 完全禁用
├── 视频捕获 ❌ 完全禁用
├── 硬件加速渲染 ❌ 替换为自有渲染层
├── 帧率计算 ❌ 禁用（硬编码 60fps）
├── VBlank 同步 ❌ 禁用
├── 视频滤镜 ❌ 禁用（SetFilters no-op）
├── 多渲染器 ❌ 简化为单一路径
├── Codec 工厂 ⚠️ 简化（保留链式但裁剪条目）
└── FFmpeg API ⚠️ 使用已弃用版本
```

---

## 5.3.2 #if 0 禁用代码块清单

KrKr2 源码中大量使用 `#if 0` 来禁用不需要的功能。以下是完整清单：

### VideoPlayer.cpp 中的禁用块

```cpp
// ===== 1. 字幕处理（完全禁用） =====
#if 0  // 原始位置：VideoPlayer.cpp 多处
// 字幕流的打开、关闭、渲染
void CVideoPlayer::OpenSubtitleStream(int iStream) { ... }
void CVideoPlayer::CloseSubtitleStream(bool bKeepOverlays) { ... }
// Kodi 支持内嵌字幕、外挂字幕（SRT/ASS/SSA）、蓝光 PGS 字幕
// KrKr2 的视觉小说不需要字幕功能
#endif

// ===== 2. Teletext/RDS 数据（完全禁用） =====
#if 0
// Teletext：模拟电视图文信息
// RDS：FM 广播数据系统
// 这些是欧洲电视广播的遗留功能
void CVideoPlayer::OpenTeletextStream(int iStream) { ... }
void CVideoPlayer::OpenRadioRDSStream(int iStream) { ... }
#endif

// ===== 3. DVD/蓝光菜单导航（完全禁用） =====
#if 0
// DVD 菜单按钮高亮、章节导航、角度切换
// Kodi 作为全功能媒体播放器需要这些，KrKr2 不需要
void CVideoPlayer::OnDVDNavResult(void* pData, int iMessage) { ... }
void CVideoPlayer::SetDVDMenuHighlight(int button) { ... }
#endif

// ===== 4. 帧率计算（完全禁用） =====
#if 0
// CalcFrameRate：通过统计解码帧的时间间隔估算实际帧率
// 用于动态调整丢帧策略
// KrKr2 禁用后 m_bAllowDrop 永远为 false
void CVideoPlayer::CalcFrameRate() {
    // 统计最近 N 帧的时间间隔
    // 计算平均帧率
    // 如果帧率稳定，设置 m_bAllowDrop = true
}
#endif

// ===== 5. 视频捕获/截图（完全禁用） =====
#if 0
// 从解码帧中截取画面
// Kodi 用于生成缩略图和截图功能
bool CVideoPlayer::GetCurrentFrame(DVDVideoPicture& picture) { ... }
#endif
```

### VideoPlayerVideo.cpp 中的禁用块

```cpp
// ===== 6. VBlank 同步（完全禁用） =====
#if 0
// CVideoReferenceClock：通过垂直同步信号校准渲染时钟
// Kodi 使用 VBlank 中断实现精确的帧呈现时间控制
// KrKr2 使用 Cocos2d 的渲染循环替代
class CVideoReferenceClock {
    void Start();     // 启动 VBlank 监控线程
    void Stop();
    double GetTime(); // 获取 VBlank 校准后的时间
};
#endif

// ===== 7. 丢帧策略中的帧率依赖（部分禁用） =====
#if 0
// CalcDropRequirement 中依赖 CalcFrameRate 的部分
// 因为 CalcFrameRate 被禁用，相关的丢帧逻辑也失效
if (m_bAllowDrop) {
    // 基于帧率计算的丢帧判断
    // m_bAllowDrop 在 KrKr2 中永远为 false
}
#endif
```

### VideoRenderer.cpp 中的禁用块

```cpp
// ===== 8. 多渲染器选择（完全禁用） =====
#if 0
// Kodi 支持多种渲染后端：
// - DXVA2（Windows DirectX 硬件加速）
// - VAAPI（Linux VA-API 硬件加速）
// - VDPAU（Linux NVIDIA 硬件加速）
// - VideoToolbox（macOS 硬件加速）
// - OpenGL ES（移动端）
// KrKr2 统一使用自己的 CBaseRenderer 接口
CBaseRenderer* CRenderManager::CreateRenderer() {
    // 原版根据平台和硬件选择最优渲染器
    // KrKr2 直接返回 KRMoviePlayer 自身（实现了 CBaseRenderer）
}
#endif
```

---

## 5.3.3 API 降级与简化

### FFmpeg 已弃用 API

KrKr2 使用了部分已在新版 FFmpeg 中弃用的 API：

```cpp
// ===== avcodec_decode_video2（已弃用） =====
// 源码：VideoCodecFFmpeg.cpp

// KrKr2 使用的旧 API（FFmpeg 3.x 风格）
int CDVDVideoCodecFFmpeg::Decode(
    uint8_t* pData, int iSize, double dts, double pts)
{
    AVPacket avpkt;
    av_init_packet(&avpkt);    // 也已弃用！
    avpkt.data = pData;
    avpkt.size = iSize;

    int gotPicture = 0;
    // avcodec_decode_video2 在 FFmpeg 3.1 中标记为 deprecated
    // 新 API 应使用 avcodec_send_packet + avcodec_receive_frame
    int len = avcodec_decode_video2(
        m_pCodecContext, m_pFrame,
        &gotPicture, &avpkt
    );

    return gotPicture ? VC_PICTURE : VC_BUFFER;
}

// 如果要升级到新版 FFmpeg，应改为：
int CDVDVideoCodecFFmpeg::DecodeModern(
    uint8_t* pData, int iSize, double dts, double pts)
{
    AVPacket* avpkt = av_packet_alloc();
    avpkt->data = pData;
    avpkt->size = iSize;

    int ret = avcodec_send_packet(m_pCodecContext, avpkt);
    av_packet_free(&avpkt);

    if (ret < 0) return VC_ERROR;

    ret = avcodec_receive_frame(m_pCodecContext, m_pFrame);
    if (ret == 0) return VC_PICTURE;
    if (ret == AVERROR(EAGAIN)) return VC_BUFFER;
    return VC_ERROR;
}
```

### 新旧 API 对比

| 旧 API (KrKr2 使用) | 新 API (FFmpeg 4.0+) | 差异 |
|---------------------|---------------------|------|
| `av_init_packet()` | `av_packet_alloc()` | 新 API 堆分配，旧 API 栈初始化 |
| `avcodec_decode_video2()` | `avcodec_send_packet()` + `avcodec_receive_frame()` | 新 API 解耦输入输出，支持异步 |
| `avcodec_decode_audio4()` | `avcodec_send_packet()` + `avcodec_receive_frame()` | 同上 |
| `avpicture_fill()` | `av_image_fill_arrays()` | 函数重命名 |
| `PIX_FMT_*` | `AV_PIX_FMT_*` | 枚举值重命名 |

### g_graphicsContext.GetFPS() 硬编码

原版 Kodi 通过系统 API 检测显示器刷新率，KrKr2 直接硬编码为 60fps：

```cpp
// === 原版 Kodi ===
float fps = g_graphicsContext.GetFPS();
// 查询系统显示刷新率
// Windows: EnumDisplaySettings → dmDisplayFrequency
// Linux:   XRandR → refresh rate
// macOS:   CGDisplayModeGetRefreshRate
// 返回 60/75/120/144 等实际值

// === KrKr2 版本 ===
float CGraphicContext::GetFPS() {
    return 60.0f;  // 硬编码
}
// 因为：
// 1. 视觉小说的视频通常是 24fps/30fps，远低于任何显示器
// 2. Cocos2d 引擎自己管理帧率，不需要播放器关心显示器频率
// 3. VBlank 同步已禁用，帧率信息不再用于渲染调度
```

### SetFilters() 空实现

Kodi 的视频滤镜系统在 KrKr2 中被完全清空：

```cpp
// === 原版 Kodi ===
void CVideoPlayerVideo::SetFilters() {
    // 添加反交错滤镜（deinterlace）
    // 添加缩放滤镜（scale）
    // 添加后处理滤镜（postprocessing）
    // 添加色彩空间转换滤镜
    m_filters.push_back(new CDeinterlaceFilter());
    m_filters.push_back(new CScaleFilter());
}

// === KrKr2 版本 ===
void CVideoPlayerVideo::SetFilters() {
    // 空函数，不添加任何滤镜
    // 视觉小说的视频是逐行扫描的，不需要反交错
    // 缩放由 Cocos2d 的 Sprite 处理
}
```

---

## 5.3.4 Codec 工厂简化

原版 Kodi 的 Codec 工厂支持多种硬件加速解码器，KrKr2 大幅简化了这个链路：

```cpp
// === 原版 Kodi 的 Codec 优先级链 ===
// 1. DXVA2     (Windows DirectX 硬件加速)
// 2. VAAPI     (Linux Intel/AMD 硬件加速)
// 3. VDPAU     (Linux NVIDIA 硬件加速)
// 4. VideoToolbox (macOS 硬件加速)
// 5. MediaCodec (Android 硬件加速)
// 6. OpenMax   (嵌入式硬件加速)
// 7. Amlogic   (Amlogic SoC 硬件加速)
// 8. MMAL      (Raspberry Pi 硬件加速)
// 9. FFmpeg    (纯软件解码)

// === KrKr2 的 Codec 优先级链 ===
// 源码：FactoryCodec.cpp
CDVDVideoCodec* CDVDFactoryCodec::CreateVideoCodec(
    CDVDStreamInfo& hint, CProcessInfo& processInfo)
{
    std::vector<CDVDVideoCodec*> codecs;

    // 1. Amlogic（Amlogic SoC 设备专用）
    #ifdef HAS_AMLOGIC
    codecs.push_back(new CDVDVideoCodecAmlogic(processInfo));
    #endif

    // 2. Android MediaCodec（Android 平台硬件加速）
    #ifdef HAS_MEDIACODEC
    codecs.push_back(new CDVDVideoCodecAndroidMediaCodec(
        processInfo));
    #endif

    // 3. OpenMax（某些嵌入式平台）
    #ifdef HAS_OPENMAX
    codecs.push_back(new CDVDVideoCodecOpenMax(processInfo));
    #endif

    // 4. MMAL（Raspberry Pi）
    #ifdef HAS_MMAL
    codecs.push_back(new CDVDVideoCodecMMAL(processInfo));
    #endif

    // 5. FFmpeg（始终可用的软件解码后备）
    codecs.push_back(new CDVDVideoCodecFFmpeg(processInfo));

    // 尝试按优先级打开
    for (auto* codec : codecs) {
        if (codec->Open(hint, processInfo)) {
            // 释放未使用的 codec
            for (auto* other : codecs) {
                if (other != codec) delete other;
            }
            return codec;
        }
        delete codec;
    }

    return nullptr;  // 所有 codec 都失败
}
```

### 裁剪对比表

| Codec | 原版 Kodi | KrKr2 | 说明 |
|-------|----------|-------|------|
| DXVA2 | ✅ | ❌ | Windows 硬件加速，需要 DirectX 集成 |
| VAAPI | ✅ | ❌ | Linux Intel/AMD 硬件加速 |
| VDPAU | ✅ | ❌ | Linux NVIDIA 硬件加速 |
| VideoToolbox | ✅ | ❌ | macOS 硬件加速 |
| MediaCodec | ✅ | ✅ | Android 硬件加速（保留） |
| OpenMax | ✅ | ✅ | 嵌入式硬件加速（保留） |
| Amlogic | ✅ | ✅ | Amlogic SoC（保留） |
| MMAL | ✅ | ✅ | Raspberry Pi（保留） |
| FFmpeg | ✅ | ✅ | 软件解码（保留） |

> **设计考量**：KrKr2 保留了移动端和嵌入式平台的硬件加速（MediaCodec、
> Amlogic），因为这些平台的 CPU 性能有限，纯软件解码可能无法满足
> 720p/1080p 视频的实时播放需求。

---

## 5.3.5 渲染层替换

这是 KrKr2 与原版 Kodi 最大的差异——整个渲染管线被替换为
KrKr2 自己的实现：

### 原版 Kodi 渲染架构

```
原版 Kodi:
┌─────────────────────────────────────────────┐
│            CRenderManager                    │
│  ┌─────────────┐   ┌──────────────────┐     │
│  │ Frame Queue  │──▶│ CBaseRenderer    │     │
│  │ (ring buffer)│   │  ├─ DXVA2Renderer│     │
│  └─────────────┘   │  ├─ VDPAURenderer│     │
│                     │  ├─ VAAPIRenderer│     │
│                     │  └─ GLRenderer   │     │
│                     └──────────┬───────┘     │
│                                │              │
│                     ┌──────────▼───────┐     │
│                     │  GUI Window       │     │
│                     │  (独立窗口系统)    │     │
│                     └──────────────────┘     │
└─────────────────────────────────────────────┘
```

### KrKr2 渲染架构

```
KrKr2:
┌─────────────────────────────────────────────────┐
│            CRenderManager                        │
│  ┌─────────────┐   ┌────────────────────────┐   │
│  │ Frame Queue  │──▶│ CBaseRenderer          │   │
│  │ (ring buffer)│   │  = KRMoviePlayer 自身  │   │
│  └─────────────┘   └──────────┬─────────────┘   │
│                                │                  │
│              ┌─────────────────┼───────────┐     │
│              ▼                 ▼           │     │
│  ┌──────────────────┐  ┌──────────────┐   │     │
│  │ VideoPresentLayer │  │VideoPresentOL│   │     │
│  │ (Layer 模式)      │  │(Overlay 模式)│   │     │
│  │                   │  │              │   │     │
│  │ sws_scale         │  │ TVPYUVSprite │   │     │
│  │ YUV→RGBA          │  │ GPU YUV→RGB  │   │     │
│  │ ↓                 │  │ ↓            │   │     │
│  │ tTVPBaseTexture   │  │ Cocos2d Node │   │     │
│  └──────────────────┘  └──────────────┘   │     │
│              │                 │           │     │
│              ▼                 ▼           │     │
│         KrKr2 图层系统    Cocos2d 场景树    │     │
└─────────────────────────────────────────────────┘
```

### KRMoviePlayer 如何实现 CBaseRenderer

KRMoviePlayer 类同时实现了 iTVPVideoOverlay（KrKr2 接口）和
CBaseRenderer（Kodi 渲染器接口），充当了两个系统之间的桥梁：

```cpp
// 源码：KRMoviePlayer.h
class KRMoviePlayer
    : public iTVPVideoOverlay  // KrKr2 视频覆盖接口
    , public CBaseRenderer     // Kodi 渲染器接口
{
    // CBaseRenderer 接口实现
    void AddVideoPicture(DVDVideoPicture& pic) override {
        // 将解码帧交给 VideoPresentOverlay 或 VideoPresentLayer
        m_presenter->AddVideoPicture(pic);
    }

    void FlipPage() override {
        // 通知渲染层更新显示
        m_presenter->FlipPage();
    }

    bool IsConfigured() override {
        return m_presenter != nullptr;
    }

    // iTVPVideoOverlay 接口实现
    void Play() override {
        m_player->SetSpeed(DVD_PLAYSPEED_NORMAL);
    }

    void Pause() override {
        m_player->SetSpeed(DVD_PLAYSPEED_PAUSE);
    }

    void Stop() override {
        m_player->StopThread(true);
    }
};
```

---

## 5.3.6 Stubbed 方法清单

以下方法在 KrKr2 中存在但实现为空或返回固定值：

```cpp
// === TVPMoviePlayer.cpp 中的 Stub 方法 ===

// 字幕相关（全部为空）
void SetSubtitleState(bool on) {}          // 无字幕支持
void SetSubtitleStream(int stream) {}      // 无字幕轨切换
int  GetSubtitleCount() { return 0; }      // 字幕数为 0

// 视频属性（固定返回值）
float GetVideoAspectRatio() { return 0; }  // 不报告宽高比
int   GetVideoWidth() { return 0; }        // 不报告分辨率
int   GetVideoHeight() { return 0; }       // 不报告分辨率

// 频道相关（全部为空）
void ChannelNext() {}                      // 无频道概念
void ChannelPrev() {}                      // 无频道概念
void SelectChannel(int ch) {}              // 无频道概念

// 视频滤镜（空实现）
void SetFilters() {}                       // 不使用滤镜链

// 帧率估算（禁用）
void CalcFrameRate() {}                    // 不估算帧率
```

### 跨平台差异在 Stub 中的体现

| Stub 方法 | Windows | Linux | macOS | Android |
|-----------|---------|-------|-------|---------|
| GetFPS() | 返回 60 | 返回 60 | 返回 60 | 返回 60 |
| GetSubtitleCount() | 返回 0 | 返回 0 | 返回 0 | 返回 0 |
| SetFilters() | 空函数 | 空函数 | 空函数 | 空函数 |
| CalcFrameRate() | 空函数 | 空函数 | 空函数 | 空函数 |

> **注意**：所有平台的 Stub 行为完全相同，因为这些功能在 KrKr2 中
> 被统一裁剪，不存在平台特定的差异。

---

## 5.3.7 ErrorAdjust 阈值差异

A/V 同步中的 ErrorAdjust 阈值在 KrKr2 中与原版 Kodi 存在细微差异：

```cpp
// === 原版 Kodi 的 ErrorAdjust ===
// 使用 VBlank 校准的精确阈值
// 阈值基于显示器实际帧间隔动态计算
double threshold = 1.0 / g_graphicsContext.GetFPS();
// 例如：60Hz → 16.67ms, 120Hz → 8.33ms

// === KrKr2 的 ErrorAdjust ===
// 源码：VideoPlayerVideo.cpp
void CVideoPlayerVideo::CalcDropRequirement() {
    // 音频超前视频的阈值（固定值）
    double errorAhead  = 0.020;  // 20ms
    // 音频落后视频的阈值（固定值）
    double errorBehind = 0.027;  // 27ms

    // 不对称阈值的原因：
    // - 音频超前 20ms 就开始追赶（人耳对提前更敏感）
    // - 音频落后 27ms 才开始丢帧（允许更大的滞后容忍）

    double error = m_clock - m_audioClock;
    if (error > errorBehind) {
        // 视频落后太多，需要丢帧追赶
        m_lateframes++;
    } else if (error < -errorAhead) {
        // 视频超前太多，需要等待
        Sleep(1);
    }
}
```

### 阈值对比

| 参数 | 原版 Kodi | KrKr2 | 影响 |
|------|----------|-------|------|
| 超前阈值 | 动态计算 (1/fps) | 固定 20ms | KrKr2 可能在低帧率视频中过早追赶 |
| 落后阈值 | 动态计算 | 固定 27ms | 相对宽松，适合视觉小说 |
| m_bAllowDrop | 可变 | 始终 false | KrKr2 不主动丢帧 |
| CalcFrameRate | 运行时估算 | 禁用 | 无帧率感知能力 |

---

## 5.3.8 保留不变的核心组件

虽然大量功能被裁剪，但以下核心组件与原版 Kodi 几乎完全一致：

```
✅ 保留不变的组件：
├── CDVDDemuxFFmpeg       — FFmpeg 解封装器封装
├── CDVDStreamInfo        — 流信息描述结构
├── CDVDClock             — A/V 同步主时钟
├── CDVDMsg / CDVDMessageQueue — 消息系统
├── CThread / CEvent      — 线程基础设施
├── IRef<T>               — 引用计数模板
├── VideoPlayer::Process()— 主循环逻辑
├── CVideoPlayerVideo     — 视频解码线程
├── CVideoPlayerAudio     — 音频解码线程
├── CDVDVideoCodecFFmpeg  — FFmpeg 视频解码器（API 旧但逻辑一致）
├── CDVDAudioCodecFFmpeg  — FFmpeg 音频解码器
├── CRenderManager        — 帧队列管理器
└── DemuxPacket           — 数据包结构
```

这些组件保留了 Kodi 久经考验的多线程架构和 A/V 同步算法，
是 KrKr2 视频播放稳定性的基石。

---

## 常见错误及解决方案

### 错误 1：取消 #if 0 恢复功能时遗漏依赖的成员变量/类

**现象**：将某个 `#if 0` 块改为 `#if 1` 试图恢复被裁剪的 Kodi 功能后，编译报大量 `undeclared identifier` 或 `incomplete type` 错误，涉及的符号散布在好几个文件里。

**原因**：KrKr2 裁剪 Kodi 代码时，不仅用 `#if 0` 禁用了功能代码块，还同步删除或注释了该功能依赖的成员变量、辅助类、头文件包含。恢复代码块时只取消了 `#if 0`，但这些依赖已经不存在了。

**解决**：恢复 `#if 0` 块之前，先做依赖扫描：

```bash
# 步骤 1：提取 #if 0 块中引用的所有标识符
# Windows (PowerShell)
$block = Get-Content CDVDVideoCodecFFmpeg.cpp -Raw
# 手动定位 #if 0 块，列出其中用到的类名/变量名

# 步骤 2：在整个 movie/ffmpeg/ 目录搜索这些标识符
grep -rn "CDVDVideoCodecDRMPRIME" cpp/core/movie/ffmpeg/
# 如果搜不到定义 → 说明该类已被整个删除，需要从原版 Kodi 回搬

# 步骤 3：从原版 Kodi 对应版本的源码中找到缺失的定义
# 参考 Kodi v19 (Matrix) 的 xbmc/cores/VideoPlayer/ 目录
```

> 💡 建议用 `git diff` 对比 KrKr2 版本与原版 Kodi 对应文件，可以快速看到所有被删除的内容。

### 错误 2：使用已弃用的 FFmpeg API 导致新版 FFmpeg 编译失败

**现象**：升级 FFmpeg 版本（如从 4.x 到 5.x/6.x）后编译报错：`'av_init_packet' is deprecated`、`'avcodec_decode_video2' undeclared` 等。

**原因**：KrKr2 保留的 Kodi 遗产代码中仍在使用 FFmpeg 4.x 时代的旧 API。这些 API 在 FFmpeg 5.0+ 中被标记为 deprecated，在 6.0+ 中被直接移除：

| 旧 API（已弃用） | 新 API（替代） | 变更版本 |
|------------------|----------------|----------|
| `av_init_packet(&pkt)` | 栈上零初始化 `AVPacket pkt = {}` 或 `av_packet_alloc()` | FFmpeg 4.4 deprecated, 6.0 removed |
| `avcodec_decode_video2()` | `avcodec_send_packet()` + `avcodec_receive_frame()` | FFmpeg 3.1 deprecated, 5.0 removed |
| `avpicture_fill()` | `av_image_fill_arrays()` | FFmpeg 3.4 deprecated |

**解决**：逐一替换弃用调用。以 `av_init_packet` 为例：

```cpp
// ❌ 旧写法（FFmpeg 6.0 编译失败）
AVPacket pkt;
av_init_packet(&pkt);
pkt.data = nullptr;
pkt.size = 0;

// ✅ 新写法（兼容 FFmpeg 4.x-7.x）
AVPacket* pkt = av_packet_alloc();
if (!pkt) { /* 内存分配失败处理 */ }
// 使用完毕后：
av_packet_free(&pkt);
```

### 错误 3：硬编码 GetFPS()=60 导致在非标准刷新率下同步异常

**现象**：在 75Hz/120Hz/144Hz 显示器上播放视频，画面出现周期性的微卡顿（judder），但在 60Hz 显示器上完全正常。

**原因**：KrKr2 中 `CRenderManager::GetFPS()` 被简化为直接返回 `60.0`（硬编码），而原版 Kodi 会查询显示器实际刷新率。当显示器是 144Hz 时，渲染器以为刷新率是 60Hz，`VSYNC` 时序计算全部错误，导致每隔几帧就多等或少等一个刷新周期。

**解决**：

```cpp
// ❌ KrKr2 当前的硬编码实现
double CRenderManager::GetFPS() {
    return 60.0;  // 永远返回 60，不管实际显示器
}

// ✅ 修复：查询 Cocos2d-x 的实际帧率设置
double CRenderManager::GetFPS() {
    // Cocos2d-x Director 记录了当前的动画间隔
    float interval = cocos2d::Director::getInstance()
                         ->getAnimationInterval();
    if (interval > 0.0f) {
        return 1.0 / interval;  // 如 interval=1/60 → 返回 60
    }
    return 60.0;  // fallback
}
```

> ⚠️ 即使修复了 `GetFPS()`，还需要检查 `FlipPage()` 和 `WaitForBuffer()` 中是否有基于 60fps 假设的硬编码等待时间（如 `sleep(16ms)`）。

---

## 动手实践

### 实践 1：#if 0 块统计

在 KrKr2 视频播放器源码目录中，搜索所有 `#if 0` 块，
统计每个文件中被禁用的代码行数：

```bash
# Windows (PowerShell)
Get-ChildItem -Path "cpp/core/movie/ffmpeg/" -Filter "*.cpp" |
    ForEach-Object {
        $content = Get-Content $_.FullName -Raw
        $matches = [regex]::Matches($content, '(?s)#if 0.*?#endif')
        $lines = ($matches | ForEach-Object {
            ($_.Value -split "`n").Count
        } | Measure-Object -Sum).Sum
        Write-Output "$($_.Name): $lines 行被禁用"
    }

# Linux / macOS
find cpp/core/movie/ffmpeg/ -name "*.cpp" -exec sh -c '
    file="$1"
    lines=$(awk "/#if 0/{f=1} f{c++} /#endif/{if(f)f=0}" "$file"
           echo "$c")
    echo "$(basename $file): ${lines:-0} 行被禁用"
' _ {} \;

# Android (adb shell)
# 同 Linux 命令，通过 adb shell 执行
```

### 实践 2：尝试恢复 CalcFrameRate

作为实验练习，尝试移除 CalcFrameRate() 周围的 `#if 0`，
观察编译错误和运行时行为：

```cpp
// 步骤 1：移除 #if 0 / #endif
// 在 VideoPlayer.cpp 中找到 CalcFrameRate 的 #if 0 块

// 步骤 2：可能遇到的编译错误
// - m_fFrameRate 成员可能已被删除
// - g_graphicsContext.GetRefreshRate() 可能不存在
// - 相关的统计变量可能缺失

// 步骤 3：添加缺失的成员变量
class CVideoPlayer {
    // 需要添加：
    double m_fFrameRate;
    int    m_frameRateCount;
    double m_frameRateTimeStamps[100];
};

// 步骤 4：在 Process() 中调用
void CVideoPlayer::Process() {
    while (!m_bStop) {
        // ... 现有代码 ...

        // 每 100 帧计算一次帧率
        CalcFrameRate();

        // 打印结果
        if (m_fFrameRate > 0) {
            printf("Estimated frame rate: %.2f fps\n",
                   m_fFrameRate);
        }
    }
}
```

### 实践 3：对比新旧 FFmpeg API

将 CDVDVideoCodecFFmpeg::Decode() 改写为使用新版 API，
并对比两种实现的差异：

```cpp
// 旧版实现（当前 KrKr2 使用）
int DecodeOld(uint8_t* pData, int iSize) {
    AVPacket avpkt;
    av_init_packet(&avpkt);
    avpkt.data = pData;
    avpkt.size = iSize;

    int gotPicture = 0;
    avcodec_decode_video2(m_pCodecContext, m_pFrame,
                          &gotPicture, &avpkt);
    return gotPicture ? VC_PICTURE : VC_BUFFER;
}

// 新版实现（FFmpeg 4.0+ 推荐）
int DecodeNew(uint8_t* pData, int iSize) {
    AVPacket* avpkt = av_packet_alloc();
    avpkt->data = pData;
    avpkt->size = iSize;

    int ret = avcodec_send_packet(m_pCodecContext, avpkt);
    av_packet_free(&avpkt);
    if (ret < 0) return VC_ERROR;

    ret = avcodec_receive_frame(m_pCodecContext, m_pFrame);
    if (ret == 0)              return VC_PICTURE;
    if (ret == AVERROR(EAGAIN)) return VC_BUFFER;
    return VC_ERROR;
}

// 关键差异：
// 1. 新 API 解耦了发送和接收，支持 1:N 的包到帧映射
// 2. 新 API 自动处理解码器内部缓冲
// 3. 新 API 在 flush 时发送 NULL packet（avcodec_send_packet(ctx, NULL)）
// 4. 旧 API 的 gotPicture 参数在新 API 中由返回值替代
```

---

## 对照项目源码

| 差异类别 | 源码位置 | 关键特征 |
|---------|---------|---------|
| 字幕禁用 | `VideoPlayer.cpp` 多处 | `#if 0` 包裹 Subtitle 相关代码 |
| Teletext/RDS | `VideoPlayer.cpp` | `#if 0` 包裹 |
| DVD 菜单 | `VideoPlayer.cpp` | `#if 0` 包裹 OnDVDNavResult |
| CalcFrameRate | `VideoPlayer.cpp` | `#if 0` 包裹，m_bAllowDrop=false |
| 视频捕获 | `VideoPlayer.cpp` | `#if 0` 包裹 GetCurrentFrame |
| VBlank 同步 | `VideoReferenceClock.h` | 类定义为空壳 |
| SetFilters | `VideoPlayerVideo.cpp` | 空函数体 |
| GetFPS | `CGraphicContext` | 硬编码返回 60 |
| Codec 工厂 | `FactoryCodec.cpp` | 只保留 5 种 codec |
| 旧 FFmpeg API | `VideoCodecFFmpeg.cpp` | avcodec_decode_video2 |
| 渲染器替换 | `KRMoviePlayer.h` | 实现 CBaseRenderer |
| ErrorAdjust | `VideoPlayerVideo.cpp` | 固定 20ms/27ms 阈值 |

---

## 本节小结

- KrKr2 从 XBMC/Kodi 移植了 CDVDPlayer 架构，但裁剪了约 60% 的功能代码
- 主要裁剪项：字幕、DVD 菜单、Teletext/RDS、视频捕获、VBlank 同步、帧率计算、视频滤镜
- `#if 0` 是主要的禁用手段，源码中保留了原始代码供参考
- FFmpeg API 使用了已弃用的 avcodec_decode_video2，若升级 FFmpeg 需要迁移
- g_graphicsContext.GetFPS() 硬编码为 60fps，所有帧率相关的动态计算失效
- Codec 工厂保留了移动端硬件加速（MediaCodec、Amlogic），裁剪了桌面端（DXVA2、VAAPI、VDPAU、VideoToolbox）
- 渲染管线完全替换为 KrKr2 自有的 Layer/Overlay 双模式架构
- 核心组件（解封装、解码、消息系统、线程模型、A/V 同步）保持与原版 Kodi 一致

---

## 练习题与答案

### 练习 1：功能恢复评估

**题目**：如果要为 KrKr2 添加字幕显示功能（SRT 格式），
需要恢复哪些被禁用的组件？列出至少 3 个必须修改的文件和具体工作项。

**答案**：

需要恢复/修改的组件：

1. **VideoPlayer.cpp** — 恢复字幕流的 OpenStream/CloseStream
   - 移除 `#if 0` 围绕的 `OpenSubtitleStream()`
   - 在 Process() 主循环中添加字幕数据包路由
   - 添加 `DEMUXER_PACKET` 到字幕队列的分发逻辑

2. **新建 VideoPlayerSubtitle.h/cpp** — 字幕解码线程
   - 继承 CThread 实现 Process() 循环
   - 解析 SRT 时间戳和文本内容
   - 维护当前显示字幕的缓冲区

3. **KRMoviePlayer.h/cpp** — 字幕渲染接口
   - 添加 SetSubtitleState()/SetSubtitleStream() 的实际实现
   - 在 VideoPresentOverlay/VideoPresentLayer 中叠加字幕文本
   - 对于 Overlay 模式：创建 Cocos2d Label 节点
   - 对于 Layer 模式：将字幕文本渲染到 tTVPBaseTexture

4. **CDVDDemuxFFmpeg.cpp** — 启用字幕流解封装
   - 确保 GetStream() 返回字幕流信息
   - 在 AddStream() 中处理 AVMEDIA_TYPE_SUBTITLE 类型

### 练习 2：API 迁移风险评估

**题目**：将 `avcodec_decode_video2()` 迁移到
`avcodec_send_packet()` / `avcodec_receive_frame()` 时，
有一个容易被忽略的行为差异。这个差异是什么？它会导致什么问题？

**答案**：

关键差异在于**多帧输出**：

- `avcodec_decode_video2()` 每次调用最多输出 1 帧
- `avcodec_receive_frame()` 可能在一次 `send_packet` 后
  输出多帧（例如 B 帧参考链导致的延迟帧释放）

```cpp
// 旧 API：1 次调用 = 最多 1 帧
avcodec_decode_video2(ctx, frame, &gotPicture, &pkt);
if (gotPicture) OutputPicture(frame);

// 新 API：需要循环接收所有可用帧
avcodec_send_packet(ctx, pkt);
while (true) {
    int ret = avcodec_receive_frame(ctx, frame);
    if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
        break;
    if (ret < 0) HandleError(ret);
    OutputPicture(frame);  // 可能执行多次！
}
```

如果迁移时只调用一次 `avcodec_receive_frame()`，会导致帧丢失，
表现为视频卡顿或画面不连续。

### 练习 3：识别活代码与死代码

**题目**：以下代码片段来自 KrKr2 源码，判断哪些是"活代码"，
哪些是"死代码"，并说明理由：

```cpp
// 片段 A
if (m_bAllowDrop && m_lateframes > 3) {
    DropFrame();
}

// 片段 B
void SetFilters() {}

// 片段 C
double fps = g_graphicsContext.GetFPS();
double frameInterval = 1.0 / fps;

// 片段 D
#if 0
void CalcFrameRate() { ... }
#endif
```

**答案**：

| 片段 | 类型 | 理由 |
|------|------|------|
| A | **死代码**（逻辑死代码） | `m_bAllowDrop` 始终为 `false`（CalcFrameRate 被禁用），条件永远不满足 |
| B | **活代码**（但无效） | 函数会被调用，但函数体为空，不产生任何效果 |
| C | **活代码**（但语义降级） | GetFPS() 返回硬编码的 60，frameInterval 始终为 16.67ms，代码执行但结果不准确 |
| D | **死代码**（编译死代码） | `#if 0` 块中的代码不参与编译，完全不存在于二进制中 |

---

## 下一步

下一章 [6.1 AV 不同步排查](../06-实战-调试视频播放问题/01-AV不同步排查.md)
将进入实战调试环节——当视频播放出现音画不同步时，如何利用 CDVDClock、
ErrorAdjust 阈值、m_lateframes 计数器等工具，系统性地定位和解决同步问题。
