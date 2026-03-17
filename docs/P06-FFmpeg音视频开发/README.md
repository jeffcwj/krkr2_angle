# P06 — FFmpeg 音视频开发

> **前置知识：** C++ 基础（指针、内存管理、多线程基础）
> **难度：** ★★★★
> **目标：** 掌握 FFmpeg C API，能理解 KrKr2 的完整媒体播放器栈

## 模块简介

FFmpeg 是全球使用最广泛的开源音视频处理框架，KrKr2 项目中的视频播放器（`cpp/core/movie/ffmpeg/`）和部分音频解码器（`cpp/core/sound/`）均基于 FFmpeg 构建。本模块从零教你理解音视频基础概念，掌握 FFmpeg 的 C API，最终能读懂并修改项目中的媒体播放器代码。

## 章节目录

| 章 | 标题 | 内容要点 | 预计阅读 |
|----|------|---------|---------|
| 01 | [音视频基础概念](01-音视频基础概念.md) | 容器与编码格式、采样率与比特率、PTS/DTS 时间戳、关键帧与 GOP | 30 分钟 |
| 02 | [FFmpeg架构与核心库](02-FFmpeg架构与核心库.md) | libavformat/libavcodec/libavutil/libswscale/libswresample 五大库的职责与协作 | 35 分钟 |
| 03 | [解封装：从容器到数据包](03-解封装从容器到数据包.md) | avformat_open_input→avformat_find_stream_info→av_read_frame 完整流程 | 40 分钟 |
| 04 | [解码：从数据包到原始帧](04-解码从数据包到原始帧.md) | avcodec_find_decoder→avcodec_send_packet→avcodec_receive_frame；硬件加速简介 | 40 分钟 |
| 05 | [音视频同步](05-音视频同步.md) | PTS 时钟、主时钟选择、drift 修正、帧丢弃策略 | 45 分钟 |
| 06 | [实战：FFmpeg+SDL2视频播放器](06-实战FFmpeg-SDL2视频播放器.md) | 从零用 FFmpeg+SDL2 实现完整的视频播放器（~500行） | 60 分钟 |
| 07 | [实战：KrKr2媒体播放器架构分析](07-实战KrKr2媒体播放器架构分析.md) | 对照分析 movie/ffmpeg 架构（Kodi 遗产、CDVDMsg 消息系统、CThread 线程模型） | 50 分钟 |

## 学习路线

```
本模块学习顺序（线性推进）：

01-基础概念 → 02-FFmpeg架构 → 03-解封装 → 04-解码
                                                ↓
                              05-音视频同步 ← ─ ┘
                                    ↓
                              06-实战播放器
                                    ↓
                              07-KrKr2架构分析

学完后可继续：
  → M05-音频子系统（了解 KrKr2 音频解码器如何使用 FFmpeg）
  → M06-视频播放器（深入 KrKr2 的 movie/ffmpeg 模块实现）
```

## 项目关联

P06 是以下项目模块的核心前置知识：

- **M05-音频子系统** — `cpp/core/sound/` 中的 FFmpeg 音频解码器
- **M06-视频播放器** — `cpp/core/movie/ffmpeg/` 完整媒体播放器栈

项目中的 FFmpeg 相关源码：

| 目录 | 说明 |
|------|------|
| `cpp/core/movie/ffmpeg/` | 完整视频播放器（基于 Kodi/XBMC 架构移植） |
| `cpp/core/sound/` | 音频解码器（含 FFmpeg 解码后端） |

## 环境准备

学习本模块需要安装 FFmpeg 开发库：

```bash
# Windows (vcpkg)
vcpkg install ffmpeg[core,avcodec,avformat,swscale,swresample]:x64-windows

# Linux (Ubuntu/Debian)
sudo apt install libavcodec-dev libavformat-dev libswscale-dev libswresample-dev libavutil-dev

# macOS (Homebrew)
brew install ffmpeg

# 验证安装
pkg-config --modversion libavcodec  # Linux/macOS
```
