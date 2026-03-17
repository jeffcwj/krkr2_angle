# 本节目标
# FFmpeg 架构与核心库

> **所属模块：** P06-FFmpeg 音视频开发
> **前置知识：** [01-音视频基础概念](./01-音视频基础概念.md)、C++ 基础（指针、结构体、动态内存）
> **预计阅读时间：** 35 分钟（约 7000 字）

## 本节目标

读完本节后，你将能够：
1. 画出 FFmpeg 的整体架构图，说明各核心库的职责边界
2. 解释 **libavformat**、**libavcodec**、**libavutil**、**libswscale**、**libswresample**、**libavfilter** 六大库的功能
3. 使用 FFmpeg API 完成最基本的"打开文件 → 读取信息 → 关闭"流程
4. 在 CMake 项目中正确链接 FFmpeg 库（含 vcpkg 集成）
5. 理解 KrKr2 项目中如何使用 FFmpeg 各库

---

## 一、FFmpeg 项目概览

### 1.1 FFmpeg 是什么？

FFmpeg 是一个开源的跨平台音视频处理框架，提供了从编解码到封装/解封装、滤镜处理、格式转换的完整工具链。它由以下部分组成：

| 组件 | 类型 | 说明 |
|------|------|------|
| `ffmpeg` | 命令行工具 | 格式转换、编码、滤镜处理 |
| `ffplay` | 命令行工具 | 简易播放器（基于 SDL） |
| `ffprobe` | 命令行工具 | 媒体文件信息探测 |
| `libavformat` | 库 | 封装/解封装（容器格式处理） |
| `libavcodec` | 库 | 编解码（所有编解码器） |
| `libavutil` | 库 | 通用工具（内存管理、数学、像素格式等） |
| `libswscale` | 库 | 图像缩放与像素格式转换 |
| `libswresample` | 库 | 音频重采样与格式转换 |
| `libavfilter` | 库 | 音视频滤镜框架 |
| `libavdevice` | 库 | 设备输入/输出（摄像头、屏幕录制等） |

> **KrKr2 使用了其中 6 个库**：libavformat、libavcodec、libavutil、libswscale、libswresample、libavfilter。未使用 libavdevice（模拟器不需要设备采集）。

### 1.2 FFmpeg 的版本与 API 演进

FFmpeg 的 API 经历了多次重大变化，了解这些变化对阅读 KrKr2 源码非常重要：

| 时期 | API 特征 | 关键变化 |
|------|---------|----------|
| FFmpeg 2.x 及更早 | `avcodec_decode_video2()` | 解码函数一步完成（同步 API） |
| FFmpeg 3.1+ (2016) | `avcodec_send_packet()` / `avcodec_receive_frame()` | 引入异步编解码 API |
| FFmpeg 4.x | 废弃旧 API | `avcodec_decode_video2()` 被标记为 deprecated |
| FFmpeg 5.0+ (2022) | 删除旧 API | 旧的解码函数被完全移除 |
| FFmpeg 6.x-7.x | 稳定的新 API | `send/receive` 成为唯一方式 |

KrKr2 使用的是**新式 API**（`avcodec_send_packet` / `avcodec_receive_frame`），这意味着代码是现代风格的，你在网上搜到的旧教程中的 `avcodec_decode_video2` 不适用于本项目。

```cpp
// ❌ 旧 API（KrKr2 不使用）
int got_frame = 0;
avcodec_decode_video2(codec_ctx, frame, &got_frame, &packet);

// ✅ 新 API（KrKr2 使用）
avcodec_send_packet(codec_ctx, &packet);
avcodec_receive_frame(codec_ctx, frame);
```

### 1.3 FFmpeg 在 KrKr2 中的角色

在 KrKr2 模拟器中，FFmpeg 承担两大职责：

1. **视频播放**（`cpp/core/movie/ffmpeg/`）：播放视觉小说的 OP/ED 动画
2. **音频解码**（`cpp/core/sound/FFWaveDecoder.cpp`）：解码 FFmpeg 支持的所有音频格式

```
┌─────────────────────────────────────────────────────┐
│                    KrKr2 模拟器                       │
├─────────────────────┬───────────────────────────────┤
│  视频播放模块        │  音频解码模块                  │
│  movie/ffmpeg/      │  sound/FFWaveDecoder.cpp      │
│                     │                               │
│  ┌─ DemuxFFmpeg ────┤  ┌─ FFWaveDecoder ───────────┐│
│  │  (libavformat)   │  │  (libavformat)            ││
│  │                  │  │  (libavcodec)             ││
│  ├─ VideoCodec ─────┤  │  (libswresample)          ││
│  │  (libavcodec)    │  └────────────────────────────┘│
│  │  (libswscale)    │                               │
│  │  (libavfilter)   │                               │
│  │                  │                               │
│  ├─ AudioCodec ─────┤                               │
│  │  (libavcodec)    │                               │
│  │  (libswresample) │                               │
│  └──────────────────┘                               │
└─────────────────────────────────────────────────────┘
```

---

