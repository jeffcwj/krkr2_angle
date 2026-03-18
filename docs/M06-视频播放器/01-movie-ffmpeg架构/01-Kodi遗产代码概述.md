# 1.1 Kodi/XBMC 遗产代码概述

## 本节目标

理解 KrKr2 视频播放器的代码来源——XBMC/Kodi 开源媒体播放器项目。掌握这些遗产代码在 KrKr2 中的保留范围、裁剪策略和命名约定，为后续章节的深入分析奠定基础。

---

## 1. XBMC/Kodi 项目简介

XBMC（Xbox Media Center）最初是 2002 年为初代 Xbox 开发的开源媒体播放器。2014 年更名为 Kodi，发展为跨平台的家庭影院软件。XBMC/Kodi 的视频播放引擎经过十余年的迭代，是业界最成熟的开源视频播放实现之一。它的核心特点包括：

- **多线程流水线架构**：解封装、音频解码、视频解码各在独立线程中运行，通过消息队列通信
- **精确的音视频同步**：基于主时钟（CDVDClock）的 PTS 校正系统，支持多种同步策略
- **完善的消息系统**：CDVDMsg 层次化消息类型，支持播放控制、Seek、速度调节等操作
- **模块化解码器工厂**：通过 CDVDFactoryCodec 动态选择最优解码器（软解/硬解）

KrKr2 选择从 Kodi 移植视频播放代码，而非从零实现，是一个务实的工程决策。视频播放涉及大量的边界情况处理（时间戳不连续、解码器错误恢复、音视频同步漂移），这些都是 Kodi 在多年实践中积累的经验。

### 1.1 Kodi 源码中的 DVD 前缀

在 KrKr2 的视频播放代码中，你会频繁看到 `DVD` 前缀的类名和常量：

```cpp
// 源码: movie/ffmpeg/Message.h
// 这些类名都保留了 Kodi 的原始命名

class CDVDMsg;                    // DVD 消息基类
class CDVDMsgPlayerSeek;          // 播放器 Seek 消息
class CDVDMsgDemuxerPacket;       // 解封装数据包消息

// 源码: movie/ffmpeg/Clock.h
#define DVD_NOPTS_VALUE 0xFFF0000000000000LL  // 无效时间戳哨兵值
#define DVD_TIME_BASE   1000000               // 时间基准（微秒）
```

`DVD` 前缀源自 XBMC 最初的 DVD 播放功能。尽管 KrKr2 不需要播放 DVD 光盘，这些前缀被原样保留，因为：

1. **降低移植成本**：修改命名会引发大量连锁改动，增加移植出错风险
2. **方便对照原版**：保持命名一致使开发者可以直接参考 Kodi 源码和文档
3. **历史惯例**：Kodi 社区自身也保留了这些前缀，它已成为事实上的接口标识

### 1.2 KRMovie 命名空间

KrKr2 将所有从 Kodi 移植的代码包裹在 `KRMovie` 命名空间中，通过宏定义实现：

```cpp
// 源码: movie/ffmpeg/KRMovieDef.h
// KrKr2 的命名空间隔离机制

#define NS_KRMOVIE_BEGIN  namespace KRMovie {
#define NS_KRMOVIE_END    }

// 使用示例（所有 movie 模块源文件的标准写法）：
NS_KRMOVIE_BEGIN

class BasePlayer : public CThread {
    // ... 播放器实现
};

NS_KRMOVIE_END
```

这个命名空间隔离策略解决了一个关键问题：Kodi 的代码中有大量全局符号（类名、函数名、宏定义），直接引入会与 KrKr2 主引擎或其他第三方库产生命名冲突。`KRMovie` 命名空间确保所有移植代码的符号都被限定在独立的作用域内。

---

## 2. 移植代码的保留范围

KrKr2 从 Kodi 移植了视频播放的核心组件，同时大幅裁剪了不需要的功能。理解这个保留范围对于阅读源码至关重要——你会在代码中看到许多 `#if 0` 块和空实现，这些都是裁剪的痕迹。

### 2.1 保留的核心组件

```
┌─────────────────────────────────────────────────┐
│              已保留的 Kodi 组件                   │
├─────────────────────────────────────────────────┤
│  CDVDDemuxFFmpeg    - FFmpeg 解封装器            │
│  CDVDVideoCodecFFmpeg - FFmpeg 视频解码器        │
│  CDVDAudioCodecFFmpeg - FFmpeg 音频解码器        │
│  CDVDClock          - 主时钟/音视频同步          │
│  CVideoPlayerVideo  - 视频解码线程              │
│  CVideoPlayerAudio  - 音频解码线程              │
│  CRenderManager     - 渲染帧队列管理            │
│  CDVDMsg/CDVDMessageQueue - 消息系统            │
│  CThread/CEvent     - 线程和事件封装            │
│  BasePlayer         - 主播放循环                │
└─────────────────────────────────────────────────┘
```

以下代码示例展示了 BasePlayer 如何保留 Kodi 的核心播放循环结构：

```cpp
// 源码: movie/ffmpeg/VideoPlayer.cpp
// BasePlayer::Process() — 主播放循环（简化版）
// 这个结构与 Kodi 的 CVideoPlayer::Process() 基本一致

void BasePlayer::Process() {
    while (!m_bAbortRequest) {
        // 1. 处理消息（Play/Pause/Seek 等）
        HandleMessages();
        
        // 2. 处理播放速度变化
        HandlePlaySpeed();
        
        // 3. 检查队列是否已满
        if (m_CurrentVideo.id >= 0 && 
            m_VideoPlayerVideo->IsStalled())
            continue;
            
        // 4. 读取一个解封装数据包
        DemuxPacket* pkt = m_pDemuxer->Read();
        if (!pkt) {
            // EOF 处理
            HandleEOF();
            continue;
        }
        
        // 5. 将数据包分发到对应解码线程
        ProcessPacket(pkt);
    }
}
```

### 2.2 裁剪掉的功能

KrKr2 移除了 Kodi 中大量与视觉小说引擎无关的功能：

```cpp
// 源码: movie/ffmpeg/VideoPlayer.cpp
// 这些功能在 KrKr2 中被 #if 0 注释或空实现替代

// ❌ 已裁剪：字幕渲染系统
#if 0
void BasePlayer::AddSubtitleStream(/* ... */) {
    // Kodi 支持 SRT/ASS/SSA 等多种字幕格式
    // KrKr2 不需要外挂字幕——视觉小说有自己的文本系统
}
#endif

// ❌ 已裁剪：DVD 菜单导航
// Kodi 原版支持 DVD 菜单（libdvdnav），KrKr2 完全不需要

// ❌ 已裁剪：Teletext（图文电视）
// 欧洲广播标准，与视觉小说毫无关系

// ❌ 已裁剪：Radio RDS（无线电数据系统）
// 广播电台元数据，同样不需要

// ❌ 已裁剪：硬件加速解码（VDPAU/VAAPI/DXVA2/VideoToolbox）
// KrKr2 只使用 FFmpeg 软解码，简化了代码复杂度
```

### 2.3 裁剪对照表

| Kodi 功能 | KrKr2 状态 | 原因 |
|-----------|-----------|------|
| FFmpeg 软解码 | ✅ 保留 | 核心解码能力 |
| CDVDClock 时钟同步 | ✅ 保留 | 音视频同步必需 |
| CDVDMsg 消息系统 | ✅ 保留 | 线程间通信必需 |
| CThread 线程封装 | ✅ 保留 | 多线程架构必需 |
| 字幕渲染 | ❌ 裁剪 | 视觉小说有自己的文本系统 |
| DVD 菜单导航 | ❌ 裁剪 | 不播放 DVD 光盘 |
| 硬件加速解码 | ❌ 裁剪 | 简化移植复杂度 |
| 多音轨选择 | ⚠️ 部分保留 | 保留接口，简化实现 |
| 网络流播放 | ❌ 裁剪 | 只播放本地文件 |
| PVR/直播电视 | ❌ 裁剪 | 与视觉小说无关 |
| 蓝光播放 | ❌ 裁剪 | 不需要 |

---

## 3. 代码结构与文件组织

KrKr2 的视频播放代码集中在 `movie/ffmpeg/` 目录下，共包含约 85 个源文件（35 个 `.cpp` + 50 个 `.h`）。

### 3.1 目录结构概览

```cpp
// 文件组织结构
// 源码: movie/ffmpeg/ 目录

movie/ffmpeg/
├── krffmpeg.cpp              // KrKr2 插件入口（桥接层）
├── KRMoviePlayer.h/cpp       // Overlay 显示模式
├── KRMovieLayer.h/cpp        // Layer 显示模式
├── KRMovieDef.h              // 命名空间宏、通用工具宏
│
├── VideoPlayer.h/cpp         // BasePlayer 主播放循环
├── IVideoPlayer.h            // 播放器接口定义
├── VideoPlayerVideo.cpp      // 视频解码线程
├── VideoPlayerAudio.cpp      // 音频解码线程
│
├── DemuxFFmpeg.h/cpp         // FFmpeg 解封装器
├── VideoCodecFFmpeg.h/cpp    // FFmpeg 视频解码器
├── AudioCodecFFmpeg.h/cpp    // FFmpeg 音频解码器
├── FactoryCodec.h/cpp        // 解码器工厂
│
├── Clock.h/cpp               // CDVDClock 主时钟
├── Message.h                 // CDVDMsg 消息定义
├── MessageQueue.h/cpp        // CDVDMessageQueue 实现
│
├── VideoRenderer.h/cpp       // CRenderManager 渲染管理
├── BaseRenderer.h/cpp        // 渲染器基类
│
├── Thread.h/cpp              // CThread/CEvent 线程封装
├── InputStream.h/cpp         // IStream COM → AVIO 适配
├── AudioDevice.h/cpp         // 音频输出设备
│
├── AE*.h/cpp                 // Audio Engine（Kodi 音频引擎移植）
├── TVPMediaDemux.h/cpp       // TVP 特有解封装适配
├── VideoReferenceClock.h/cpp // 视频参考时钟
└── CMakeLists.txt            // 构建配置
```

### 3.2 分层架构

这些文件按功能可以分为四个层次：

```
┌───────────────────────────────────────────┐
│           桥接层 (Bridge Layer)            │
│  krffmpeg.cpp, KRMoviePlayer, KRMovieLayer │
│  职责：连接 KrKr2 引擎和视频播放核心       │
└─────────────────────┬─────────────────────┘
                      │
┌─────────────────────▼─────────────────────┐
│           控制层 (Control Layer)           │
│  BasePlayer, CDVDClock, CDVDMessageQueue   │
│  职责：播放流程控制、时钟同步、消息路由     │
└─────────────────────┬─────────────────────┘
                      │
┌─────────────────────▼─────────────────────┐
│           处理层 (Processing Layer)        │
│  CDVDDemuxFFmpeg, VideoCodecFFmpeg,       │
│  AudioCodecFFmpeg, CVideoPlayerVideo/Audio │
│  职责：解封装、解码、音视频流处理          │
└─────────────────────┬─────────────────────┘
                      │
┌─────────────────────▼─────────────────────┐
│           输出层 (Output Layer)            │
│  CRenderManager, BaseRenderer,            │
│  AudioDevice, VideoPresentOverlay/Layer    │
│  职责：帧缓冲管理、纹理上传、音频输出      │
└───────────────────────────────────────────┘
```

---

## 4. 入口点分析

理解视频播放器的第一步是找到入口点——KrKr2 脚本调用视频播放功能时，代码执行从哪里开始？

### 4.1 插件注册机制

```cpp
// 源码: movie/ffmpeg/krffmpeg.cpp
// KrKr2 的视频播放器是一个插件模块，通过导出函数注册

// 入口点 1：创建 Overlay 模式的视频播放器
// 当 KrKr2 脚本使用 VideoOverlay 类时调用
extern "C" void GetVideoOverlayObject(
    HWND callbackwin,
    IStream* stream,
    const tjs_char* streamname,
    const tjs_char* type,
    uint64_t size,
    iTVPVideoOverlay** out  // 输出参数：返回播放器实例
) {
    *out = new KRMovie::MoviePlayerOverlay(
        callbackwin, stream, streamname, type, size
    );
}

// 入口点 2：创建 Layer 模式的视频播放器
// 当 KrKr2 脚本使用 VideoLayer 类时调用
extern "C" void GetVideoLayerObject(
    HWND callbackwin,
    IStream* stream,
    const tjs_char* streamname,
    const tjs_char* type,
    uint64_t size,
    iTVPVideoOverlay** out
) {
    *out = new KRMovie::MoviePlayerLayer(
        callbackwin, stream, streamname, type, size
    );
}
```

### 4.2 FFmpeg 全局初始化

```cpp
// 源码: movie/ffmpeg/krffmpeg.cpp
// FFmpeg 库的全局初始化（只执行一次）

extern "C" void TVPInitLibAVCodec() {
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // 注册所有编解码器（旧版 FFmpeg API）
    avcodec_register_all();
    
    // 注册所有容器格式（解封装器/封装器）
    av_register_all();
    
    // 注册线程安全锁管理器
    av_lockmgr_register(lockmgr);
}
```

> **跨平台说明**：
> - **Windows**：FFmpeg 库通过 vcpkg 静态链接，`TVPInitLibAVCodec()` 在 DLL 加载时调用
> - **Linux**：FFmpeg 通过系统包管理器安装（`apt install libavcodec-dev`），动态链接
> - **macOS**：FFmpeg 通过 Homebrew 安装（`brew install ffmpeg`），动态链接
> - **Android**：FFmpeg 交叉编译为 `.so`，打包在 APK 的 `jniLibs/` 目录中

### 4.3 两种显示模式

KrKr2 提供两种视频显示模式，选择哪种取决于脚本层的使用方式：

```cpp
// 两种显示模式的核心区别

// 模式 1: Overlay（覆盖层）
// 使用 Cocos2d Node 作为渲染目标
// YUV 数据直接上传到 TVPYUVSprite 纹理
// 优点：无像素格式转换开销
// 适用：独立的视频播放窗口
class MoviePlayerOverlay : public TVPMoviePlayer {
    // 继承自 VideoPresentOverlay
    // PresentPicture() 直接传递 YUV 平面数据
};

// 模式 2: Layer（图层）
// 使用 tTVPBaseTexture 作为渲染目标  
// YUV→RGBA 格式转换通过 sws_scale 完成
// 优点：可与其他图层混合渲染
// 适用：视频嵌入到视觉小说的图层系统中
class MoviePlayerLayer : public TVPMoviePlayer {
    // 继承自 VideoPresentLayer
    // AddVideoPicture() 执行 sws_scale YUV→RGBA 转换
};
```

---

## 5. 构建系统集成

### 5.1 CMake 配置

```cmake
# 源码: movie/ffmpeg/CMakeLists.txt
# 视频播放器编译为静态库

add_library(core_movie_module STATIC
    krffmpeg.cpp
    KRMoviePlayer.cpp
    KRMovieLayer.cpp
    VideoPlayer.cpp
    VideoPlayerVideo.cpp
    VideoPlayerAudio.cpp
    DemuxFFmpeg.cpp
    Clock.cpp
    VideoRenderer.cpp
    # ... 共 34 个源文件
)

# FFmpeg 依赖（5 个库）
target_link_libraries(core_movie_module
    FFmpeg::avcodec      # 编解码器
    FFmpeg::avformat     # 容器格式
    FFmpeg::avutil       # 通用工具
    FFmpeg::swscale      # 图像缩放/格式转换
    FFmpeg::swresample   # 音频重采样
)

# KrKr2 内部依赖
target_link_libraries(core_movie_module
    cocos2d              # 渲染引擎
    tjs2                 # 脚本引擎
    base                 # 基础库
    sound                # 音频系统
    utils                # 工具库
    environ              # 环境抽象
    visual               # 视觉系统
)
```

### 5.2 跨平台构建

```bash
# Windows (Visual Studio + vcpkg)
cmake -B build -G "Visual Studio 17 2022" \
    -DCMAKE_TOOLCHAIN_FILE=vcpkg/scripts/buildsystems/vcpkg.cmake
cmake --build build --target core_movie_module

# Linux (GCC + 系统包)
sudo apt install libavcodec-dev libavformat-dev libavutil-dev \
                 libswscale-dev libswresample-dev
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --target core_movie_module

# macOS (Clang + Homebrew)
brew install ffmpeg
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --target core_movie_module

# Android (NDK 交叉编译)
cmake -B build \
    -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake \
    -DANDROID_ABI=arm64-v8a \
    -DANDROID_PLATFORM=android-21
cmake --build build --target core_movie_module
```

---

## 动手实践

### 练习 1：追踪入口点调用链

在源码中找到 `GetVideoOverlayObject` 的调用者，画出从 KrKr2 脚本到视频播放器实例化的完整调用链：

```
TJS2 脚本: var v = new VideoOverlay(window)
    → C++ 桥接: TVPGetVideoOverlayObject()
        → krffmpeg.cpp: GetVideoOverlayObject()
            → new MoviePlayerOverlay(...)
                → TVPMoviePlayer 构造函数
                    → BasePlayer 构造函数
                        → TVPInitLibAVCodec()  // 首次调用时初始化 FFmpeg
```

### 练习 2：统计代码裁剪比例

使用以下命令统计 `#if 0` 块的数量，估算被裁剪的代码比例：

```bash
# 统计被注释掉的代码块数量
grep -r "#if 0" movie/ffmpeg/ | wc -l

# 统计保留的有效代码行数
find movie/ffmpeg/ -name "*.cpp" -o -name "*.h" | \
    xargs cat | grep -v "^$" | grep -v "^//" | wc -l
```

---

## 对照项目源码

| 本节概念 | 对应源文件 | 关键代码位置 |
|----------|-----------|-------------|
| 插件入口点 | `krffmpeg.cpp` | `GetVideoOverlayObject()`, `GetVideoLayerObject()` |
| 命名空间宏 | `KRMovieDef.h` | `NS_KRMOVIE_BEGIN`, `NS_KRMOVIE_END` |
| DVD 常量定义 | `Clock.h` | `DVD_NOPTS_VALUE`, `DVD_TIME_BASE` |
| 安全释放宏 | `KRMovieDef.h` | `SAFE_DELETE`, `SAFE_RELEASE` |
| Overlay 模式 | `KRMoviePlayer.h/cpp` | `MoviePlayerOverlay` 类 |
| Layer 模式 | `KRMovieLayer.h/cpp` | `MoviePlayerLayer` 类 |
| 构建配置 | `CMakeLists.txt` | `add_library(core_movie_module ...)` |

---

## 本节小结

1. KrKr2 的视频播放器源自 XBMC/Kodi 开源项目，保留了核心播放引擎并裁剪了不需要的功能
2. 代码中的 `DVD` 前缀是历史遗留命名，所有移植代码包裹在 `KRMovie` 命名空间中
3. 保留的核心组件包括：解封装器、解码器、主时钟、消息系统、线程封装和渲染管理器
4. 裁剪的功能包括：字幕、DVD 菜单、硬件加速、网络流等
5. 入口点通过 `GetVideoOverlayObject`/`GetVideoLayerObject` 导出函数提供
6. 两种显示模式：Overlay（YUV 直通 Cocos2d）和 Layer（YUV→RGBA 转换到 tTVPBaseTexture）

---

## 常见错误及解决方案

### 错误 1：FFmpeg 链接失败

```
error: undefined reference to 'avcodec_register_all'
```

**原因**：FFmpeg 库未正确链接，或使用了新版 FFmpeg（4.0+）已废弃的 API。

**解决方案**：
- 确保 CMakeLists.txt 中包含所有 5 个 FFmpeg 库的链接
- 如果使用 FFmpeg 4.0+，`avcodec_register_all()` 已自动调用，可安全移除

### 错误 2：命名空间冲突

```
error: 'CThread' is ambiguous
```

**原因**：全局作用域和 `KRMovie` 命名空间中都存在同名类。

**解决方案**：
- 始终使用 `KRMovie::CThread` 显式限定
- 或在文件头部使用 `NS_KRMOVIE_BEGIN` 宏进入命名空间

### 错误 3：IStream 接口不可用

```
error: 'IStream' undeclared
```

**原因**：`IStream` 是 Windows COM 接口，在非 Windows 平台上需要兼容层。

**解决方案**：
- KrKr2 在非 Windows 平台提供了 `IStream` 的兼容实现
- 检查 `base/` 目录中的平台抽象层是否正确包含

---

## 练习题与答案

### 练习 1：为什么 KrKr2 选择移植 Kodi 的播放器而不是从零实现？

<details>
<summary>查看答案</summary>

1. **成熟度**：Kodi 的播放引擎经过十余年迭代，处理了大量边界情况（时间戳不连续、解码器错误恢复、多种容器格式兼容）
2. **音视频同步**：正确的 A/V 同步是视频播放中最复杂的问题之一，Kodi 的 CDVDClock 系统已经过充分验证
3. **开发效率**：视觉小说引擎的视频播放需求相对简单（播放 OP/ED 动画），移植比从零实现更高效
4. **可维护性**：开发者可以参考 Kodi 社区的文档和讨论来理解代码逻辑

</details>

### 练习 2：列举 `KRMovie` 命名空间隔离的三个好处

<details>
<summary>查看答案</summary>

1. **避免命名冲突**：Kodi 移植代码中的 `CThread`、`CEvent` 等通用类名不会与 KrKr2 主引擎或其他第三方库的同名类冲突
2. **清晰的代码边界**：通过命名空间可以快速区分哪些代码属于视频播放模块，哪些属于引擎核心
3. **便于整体替换**：如果将来需要替换视频播放实现，只需替换 `KRMovie` 命名空间内的代码，不影响外部接口

</details>

### 练习 3：对比两种显示模式的性能差异

<details>
<summary>查看答案</summary>

| 对比维度 | Overlay 模式 | Layer 模式 |
|---------|-------------|-----------|
| 像素格式 | YUV 直通（无转换） | YUV→RGBA（sws_scale） |
| CPU 开销 | 低（只需上传纹理） | 中高（每帧需要格式转换） |
| GPU 利用 | YUV 着色器在 GPU 完成转换 | CPU 完成转换后上传 RGBA |
| 图层混合 | 受限（独立 Node） | 完整支持（tTVPBaseTexture） |
| 适用场景 | 全屏视频播放 | 视频嵌入游戏场景 |

Overlay 模式性能更优，因为 YUV→RGB 转换由 GPU 着色器完成；Layer 模式则牺牲性能换取与图层系统的完整集成。

</details>

---

## 下一步

下一节 [1.2 CDVDPlayer 架构](02-CDVDPlayer架构.md) 将深入分析 BasePlayer（即 Kodi 的 CDVDPlayer）的内部架构，包括播放状态机、消息处理循环和数据流管线。
