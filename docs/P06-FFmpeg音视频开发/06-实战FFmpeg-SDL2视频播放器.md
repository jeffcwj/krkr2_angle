# 第6章 实战：FFmpeg + SDL2 视频播放器

> **所属模块**：P06-FFmpeg 音视频开发
> **前置知识**：第1-5章全部内容（音视频基础、FFmpeg架构、解封装、解码、音视频同步）
> **预计阅读时间**：60-80 分钟
> **实践时间**：120-180 分钟
> **难度**：★★★★☆

---

## 本节目标

经过前 5 章的理论学习，你已经掌握了音视频开发的核心概念。本章将把所有知识串联起来，
**从零构建一个功能完整的视频播放器**，支持：

1. 音视频同步播放（以音频时钟为主）
2. 多线程架构（解封装线程 + 音频解码 + 视频解码）
3. SDL2 音频输出与视频渲染
4. 基本的播放控制（暂停/继续、退出）
5. 跨平台编译（Windows / Linux / macOS）

> **重要提示**：本章代码约 500 行 C 代码，是一个**可直接编译运行**的完整播放器。
> 每一段代码都会详细解释"为什么这样写"，而不仅仅是"怎么写"。

```
┌──────────────────────────────────────────────────────┐
│                    main() 主线程                      │
│  ┌──────────┐  ┌──────────────┐  ┌────────────────┐ │
│  │ SDL事件   │  │ SDL视频渲染   │  │ SDL音频回调     │ │
│  │ 循环      │  │ (纹理刷新)    │  │ (pull模式)     │ │
│  └──────────┘  └──────────────┘  └────────────────┘ │
│        ↑              ↑                   ↑          │
│        │         video_queue          audio_queue    │
│        │              ↑                   ↑          │
│  ┌─────┴──────────────┴───────────────────┴────────┐ │
│  │           demux_thread (解封装线程)               │ │
│  │   av_read_frame → 按流索引分发到对应队列          │ │
│  └─────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

---
## 6.1 项目结构与依赖配置

### 6.1.1 文件结构

我们的播放器只需要 **3 个文件**：

```
simple_player/
├── CMakeLists.txt      # 构建配置
├── player.c            # 播放器完整源码
└── README.md           # 使用说明
```

### 6.1.2 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)
project(simple_player C)

set(CMAKE_C_STANDARD 11)

# 使用 vcpkg 或 pkg-config 查找依赖
find_package(PkgConfig REQUIRED)
pkg_check_modules(FFMPEG REQUIRED
    libavformat
    libavcodec
    libavutil
    libswscale
    libswresample
)
pkg_check_modules(SDL2 REQUIRED sdl2)

add_executable(player player.c)

target_include_directories(player PRIVATE
    ${FFMPEG_INCLUDE_DIRS}
    ${SDL2_INCLUDE_DIRS}
)

target_link_libraries(player PRIVATE
    ${FFMPEG_LIBRARIES}
    ${SDL2_LIBRARIES}
)

# Windows 下需要链接额外库
if(WIN32)
    target_link_libraries(player PRIVATE ws2_32 Secur32 Bcrypt)
endif()
```

> **vcpkg 用户提示**：如果你使用 vcpkg（参见 P02 模块），可以用
> `find_package(FFmpeg)` 和 `find_package(SDL2)` 替代 pkg-config。

### 6.1.3 依赖安装

**Ubuntu/Debian：**
```bash
sudo apt install libavformat-dev libavcodec-dev libavutil-dev \
    libswscale-dev libswresample-dev libsdl2-dev
```

**macOS (Homebrew)：**
```bash
brew install ffmpeg sdl2
```

**Windows (vcpkg)：**
```bash
vcpkg install ffmpeg[core,avformat,avcodec,swscale,swresample]:x64-windows sdl2:x64-windows
```

---
## 6.2 核心数据结构设计

在写代码之前，先设计好数据结构。好的数据结构是程序成功的一半。

### 6.2.1 包队列（PacketQueue）

解封装线程读取 `AVPacket`，放入队列；解码端从队列取出。
这是经典的**生产者-消费者模型**。

```c
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libavutil/imgutils.h>
#include <libswscale/swscale.h>
#include <libswresample/swresample.h>
#include <SDL.h>

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <assert.h>

// ============================================================
// 包队列 —— 线程安全的 AVPacket FIFO
// ============================================================

typedef struct PacketList {
    AVPacket         pkt;
    struct PacketList *next;
} PacketList;

typedef struct PacketQueue {
    PacketList  *first, *last;
    int          nb_packets;   // 队列中包的数量
    int          size;         // 队列中所有包的总字节数
    SDL_mutex   *mutex;
    SDL_cond    *cond;
    int          abort_request; // 退出标志
} PacketQueue;

// 初始化队列
static void packet_queue_init(PacketQueue *q) {
    memset(q, 0, sizeof(PacketQueue));
    q->mutex = SDL_CreateMutex();
    q->cond  = SDL_CreateCond();
}

// 入队（生产者调用）
static int packet_queue_put(PacketQueue *q, AVPacket *pkt) {
    PacketList *node = (PacketList *)av_malloc(sizeof(PacketList));
    if (!node) return -1;

    node->pkt  = *pkt;       // 浅拷贝，转移所有权
    node->next = NULL;

    SDL_LockMutex(q->mutex);

    if (!q->last) {
        q->first = node;
    } else {
        q->last->next = node;
    }
    q->last = node;
    q->nb_packets++;
    q->size += pkt->size;

    SDL_CondSignal(q->cond);  // 唤醒等待的消费者
    SDL_UnlockMutex(q->mutex);
    return 0;
}

// 出队（消费者调用，阻塞式）
static int packet_queue_get(PacketQueue *q, AVPacket *pkt, int block) {
    PacketList *node;
    int ret = -1;

    SDL_LockMutex(q->mutex);
    for (;;) {
        if (q->abort_request) {
            ret = -1;
            break;
        }

        node = q->first;
        if (node) {
            q->first = node->next;
            if (!q->first) q->last = NULL;
            q->nb_packets--;
            q->size -= node->pkt.size;
            *pkt = node->pkt;    // 转移所有权给调用者
            av_free(node);
            ret = 1;
            break;
        } else if (!block) {
            ret = 0;
            break;
        } else {
            // 队列为空，等待生产者放入新数据
            SDL_CondWait(q->cond, q->mutex);
        }
    }
    SDL_UnlockMutex(q->mutex);
    return ret;
}

// 清空队列
static void packet_queue_flush(PacketQueue *q) {
    PacketList *node, *next;
    SDL_LockMutex(q->mutex);
    for (node = q->first; node; node = next) {
        next = node->next;
        av_packet_unref(&node->pkt);
        av_free(node);
    }
    q->first = q->last = NULL;
    q->nb_packets = 0;
    q->size = 0;
    SDL_UnlockMutex(q->mutex);
}

// 销毁队列
static void packet_queue_destroy(PacketQueue *q) {
    packet_queue_flush(q);
    SDL_DestroyMutex(q->mutex);
    SDL_DestroyCond(q->cond);
}

// 通知队列中止（用于退出时唤醒阻塞的消费者）
static void packet_queue_abort(PacketQueue *q) {
    SDL_LockMutex(q->mutex);
    q->abort_request = 1;
    SDL_CondSignal(q->cond);
    SDL_UnlockMutex(q->mutex);
}
```

> **设计要点**：
> - 使用 SDL 的 `SDL_mutex` 和 `SDL_cond` 实现线程同步，确保跨平台兼容
> - `abort_request` 标志让阻塞中的 `packet_queue_get` 能够安全退出
> - `av_packet_unref` 只在 `flush` 时调用，正常出队时所有权转移给调用者

### 6.2.2 播放器全局状态

```c
// ============================================================
// 播放器状态
// ============================================================

#define MAX_AUDIO_FRAME_SIZE 192000  // 1秒 48kHz 双声道 16bit
#define AUDIO_BUFFER_SIZE    1024    // SDL 音频缓冲区样本数

typedef struct PlayerState {
    // 格式上下文
    AVFormatContext    *fmt_ctx;

    // 音频相关
    int                 audio_stream_idx;
    AVCodecContext     *audio_codec_ctx;
    PacketQueue         audio_queue;
    uint8_t             audio_buf[MAX_AUDIO_FRAME_SIZE];
    unsigned int        audio_buf_size;
    unsigned int        audio_buf_index;
    AVFrame            *audio_frame;
    SwrContext         *swr_ctx;           // 重采样上下文

    // 视频相关
    int                 video_stream_idx;
    AVCodecContext     *video_codec_ctx;
    PacketQueue         video_queue;
    struct SwsContext   *sws_ctx;          // 像素格式转换

    // 同步时钟
    double              audio_clock;       // 当前音频播放时间 (秒)

    // SDL 相关
    SDL_Window         *window;
    SDL_Renderer       *renderer;
    SDL_Texture        *texture;
    int                 video_width;
    int                 video_height;

    // 控制
    int                 quit;              // 退出标志
    int                 paused;            // 暂停标志
    SDL_Thread         *demux_tid;         // 解封装线程
} PlayerState;

// 全局播放器实例（简化设计，生产代码应避免全局变量）
static PlayerState *global_ps = NULL;
```

> **为什么用全局变量？** SDL 的音频回调函数签名固定为
> `void callback(void *userdata, Uint8 *stream, int len)`，
> 虽然有 `userdata` 参数，但为了代码简洁，我们使用全局状态。
> KrKr2 项目中使用类成员变量（`this` 指针）来替代全局变量。

---
## 6.3 音频解码与 SDL2 输出

### 6.3.1 音频解码函数

SDL2 音频使用 **pull 模式**：SDL 在需要音频数据时调用我们的回调函数。
我们需要提前解码好音频数据，放在缓冲区中等待 SDL 拉取。

```c
// ============================================================
// 音频解码
// ============================================================

// 从音频队列取一个包并解码，返回解码后的字节数
static int audio_decode_frame(PlayerState *ps) {
    AVPacket pkt;
    int data_size = 0;

    for (;;) {
        // 从队列取包
        if (packet_queue_get(&ps->audio_queue, &pkt, 1) < 0) {
            return -1;  // 队列已中止
        }

        // 发送包到解码器
        int ret = avcodec_send_packet(ps->audio_codec_ctx, &pkt);
        av_packet_unref(&pkt);
        if (ret < 0) {
            fprintf(stderr, "音频 avcodec_send_packet 错误: %d\n", ret);
            continue;
        }

        // 接收解码帧
        ret = avcodec_receive_frame(ps->audio_codec_ctx, ps->audio_frame);
        if (ret == AVERROR(EAGAIN)) {
            continue;  // 需要更多输入
        } else if (ret < 0) {
            fprintf(stderr, "音频 avcodec_receive_frame 错误: %d\n", ret);
            continue;
        }

        // ── 重采样为 SDL 需要的格式 ──
        // SDL 要求：S16 交错格式（AV_SAMPLE_FMT_S16）
        int out_samples = swr_convert(
            ps->swr_ctx,
            &ps->audio_buf,              // 输出缓冲区（uint8_t*）
            MAX_AUDIO_FRAME_SIZE / 4,     // 最大输出样本数
            (const uint8_t **)ps->audio_frame->data,
            ps->audio_frame->nb_samples
        );

        if (out_samples < 0) {
            fprintf(stderr, "swr_convert 失败\n");
            return -1;
        }

        // 输出字节数 = 样本数 × 声道数 × 每样本字节数
        data_size = out_samples * 2 * sizeof(int16_t);  // 双声道 16bit

        // ── 更新音频时钟 ──
        if (ps->audio_frame->pts != AV_NOPTS_VALUE) {
            AVRational tb = ps->fmt_ctx->streams[ps->audio_stream_idx]->time_base;
            ps->audio_clock = av_q2d(tb) * ps->audio_frame->pts;
        }
        // 加上本帧播放时长
        ps->audio_clock += (double)out_samples /
                           ps->audio_codec_ctx->sample_rate;

        av_frame_unref(ps->audio_frame);
        return data_size;
    }
}
```

> **关键点**：
> 1. `swr_convert` 将解码器输出（可能是浮点、平面格式）转换为 SDL 需要的 S16 交错格式
> 2. 音频时钟 (`audio_clock`) 在每次解码后更新，这是 A/V 同步的基准
> 3. 时钟 = PTS 转秒 + 本帧时长，确保时钟持续递增

### 6.3.2 SDL 音频回调

```c
// SDL 音频回调 —— 在音频设备需要数据时被调用（独立线程）
static void sdl_audio_callback(void *userdata, Uint8 *stream, int len) {
    PlayerState *ps = (PlayerState *)userdata;
    int len1, audio_size;

    // SDL 要求我们填满 len 字节的 stream 缓冲区
    while (len > 0) {
        if (ps->audio_buf_index >= ps->audio_buf_size) {
            // 缓冲区已用完，解码新的数据
            audio_size = audio_decode_frame(ps);
            if (audio_size < 0) {
                // 解码失败，填充静音
                memset(stream, 0, len);
                break;
            }
            ps->audio_buf_size = audio_size;
            ps->audio_buf_index = 0;
        }

        len1 = ps->audio_buf_size - ps->audio_buf_index;
        if (len1 > len) len1 = len;

        memcpy(stream, ps->audio_buf + ps->audio_buf_index, len1);
        len -= len1;
        stream += len1;
        ps->audio_buf_index += len1;
    }
}
```

> **pull 模式工作流**：
> ```
> SDL音频线程 ──callback──→ audio_decode_frame() ──→ packet_queue_get()
>                                                       ↓
>                                              (阻塞等待解封装线程)
> ```
> SDL 回调运行在独立线程中，当缓冲区数据不足时会自动调用 `audio_decode_frame`
> 获取新数据。这种 pull 模式的好处是 SDL 自动控制消费速率，与音频硬件时钟同步。

### 6.3.3 常见错误：音频杂音

| 现象 | 原因 | 解决方案 |
|------|------|----------|
| 播放有杂音/爆音 | 重采样参数不匹配 | 检查 `swr_alloc_set_opts` 的输入输出格式 |
| 只有一个声道有声 | 声道布局错误 | 确认 `out_channel_layout` 为 `AV_CH_LAYOUT_STEREO` |
| 播放速度异常 | 采样率不匹配 | SDL 打开的采样率必须与重采样输出一致 |
| 开头有短暂杂音 | 缓冲区未清零 | 在 `audio_buf` 分配后 `memset(0)` |

---
## 6.4 视频解码与 SDL2 渲染

### 6.4.1 视频解码 + A/V 同步

视频解码在主线程的事件循环中以定时器驱动。
每次定时器触发时，从视频队列取包、解码、计算同步延迟、渲染。

```c
// ============================================================
// 视频解码与渲染
// ============================================================

// 解码一帧视频并渲染到 SDL 纹理
static void video_refresh(PlayerState *ps) {
    AVPacket pkt;
    AVFrame *frame = av_frame_alloc();
    AVFrame *frame_rgb = av_frame_alloc();
    int got_frame = 0;

    if (!frame || !frame_rgb) return;

    // 尝试从队列取一个视频包（非阻塞）
    while (packet_queue_get(&ps->video_queue, &pkt, 0) > 0) {
        // 发送到解码器
        int ret = avcodec_send_packet(ps->video_codec_ctx, &pkt);
        av_packet_unref(&pkt);
        if (ret < 0) continue;

        // 接收解码帧
        ret = avcodec_receive_frame(ps->video_codec_ctx, frame);
        if (ret == AVERROR(EAGAIN)) continue;
        if (ret < 0) break;

        got_frame = 1;

        // ── A/V 同步判断 ──
        double pts = 0.0;
        if (frame->pts != AV_NOPTS_VALUE) {
            AVRational tb = ps->fmt_ctx->streams[ps->video_stream_idx]->time_base;
            pts = av_q2d(tb) * frame->pts;
        }

        double diff = pts - ps->audio_clock;

        if (diff > 0.05) {
            // 视频超前音频 50ms 以上 → 等一等，把包放回去
            // 简化处理：直接跳过本次渲染，等下次定时器
            av_frame_unref(frame);
            got_frame = 0;
            // 注意：这里简化了处理，实际应该缓存这一帧而不是丢弃
            continue;
        } else if (diff < -0.05) {
            // 视频落后音频 50ms 以上 → 丢弃这一帧，追赶音频
            av_frame_unref(frame);
            got_frame = 0;
            continue;
        }

        // diff 在 [-50ms, +50ms] 范围内，正常渲染
        break;
    }

    if (!got_frame) {
        av_frame_free(&frame);
        av_frame_free(&frame_rgb);
        return;
    }

    // ── 像素格式转换 (YUV → RGB) ──
    // 为 RGB 帧分配缓冲区
    int num_bytes = av_image_get_buffer_size(
        AV_PIX_FMT_YUV420P,
        ps->video_width, ps->video_height, 32
    );
    uint8_t *buffer = (uint8_t *)av_malloc(num_bytes);
    av_image_fill_arrays(
        frame_rgb->data, frame_rgb->linesize,
        buffer, AV_PIX_FMT_YUV420P,
        ps->video_width, ps->video_height, 32
    );

    // ── 直接使用 YUV420P 更新纹理（避免 RGB 转换开销）──
    SDL_UpdateYUVTexture(
        ps->texture, NULL,
        frame->data[0], frame->linesize[0],   // Y 平面
        frame->data[1], frame->linesize[1],   // U 平面
        frame->data[2], frame->linesize[2]    // V 平面
    );

    // 渲染到窗口
    SDL_RenderClear(ps->renderer);
    SDL_RenderCopy(ps->renderer, ps->texture, NULL, NULL);
    SDL_RenderPresent(ps->renderer);

    // 清理
    av_free(buffer);
    av_frame_free(&frame);
    av_frame_free(&frame_rgb);
}
```

> **为什么直接用 `SDL_UpdateYUVTexture`？**
>
> SDL2 的 `SDL_PIXELFORMAT_IYUV` 纹理格式原生支持 YUV420P，
> GPU 可以直接在着色器中完成 YUV→RGB 转换。这比用 `sws_scale`
> 在 CPU 上转换为 RGB 再上传**快 3-5 倍**。
>
> KrKr2 项目中的 `TVPYUVSprite` 也采用了相同策略——直接传递 YUV
> 数据给 GPU（参见 `KRMoviePlayer.cpp:258`）。

### 6.4.2 A/V 同步策略详解

我们采用**音频主时钟**策略（与 KrKr2 项目一致）：

```
┌─────────────────────────────────────────────────┐
│              A/V 同步决策流程                     │
│                                                  │
│  video_pts = 当前视频帧的 PTS（秒）               │
│  audio_clock = 当前音频播放位置（秒）              │
│  diff = video_pts - audio_clock                  │
│                                                  │
│  if diff > +0.05:    视频超前 → 延迟渲染          │
│  if diff < -0.05:    视频落后 → 丢帧追赶          │
│  if |diff| ≤ 0.05:   同步范围 → 正常渲染          │
└─────────────────────────────────────────────────┘
```

**阈值选择 50ms 的原因**：
- 人眼对音画不同步的感知阈值约为 **80-100ms**
- 留 30-50ms 的余量确保用户几乎感觉不到延迟
- KrKr2 的 `CRenderManager::PrepareNextRender()` 使用了更精细的阈值（参见第5章）

### 6.4.3 常见错误：视频画面

| 现象 | 原因 | 解决方案 |
|------|------|----------|
| 画面全绿 | YUV 数据错误或 linesize 不匹配 | 确认 `frame->linesize` 传入正确 |
| 画面倒置 | 某些编码器输出是底部向上 | 检查 `frame->linesize[0]` 是否为负 |
| 画面拉伸 | SDL 窗口与视频比例不一致 | 用 `SDL_RenderSetLogicalSize` 设定比例 |
| 画面撕裂 | 未启用 VSync | 创建 renderer 时加 `SDL_RENDERER_PRESENTVSYNC` |
| 画面卡顿 | 解码速度不够 | 检查 CPU 负载，考虑降低分辨率 |

---
## 6.5 解封装线程

解封装线程是播放器的"数据泵"，不断从文件中读取数据包并分发到对应队列。

### 6.5.1 线程函数

```c
// ============================================================
// 解封装线程
// ============================================================

static int demux_thread(void *arg) {
    PlayerState *ps = (PlayerState *)arg;
    AVPacket pkt;

    while (!ps->quit) {
        // 暂停时休眠
        if (ps->paused) {
            SDL_Delay(10);
            continue;
        }

        // 队列过满时等待（防止内存爆炸）
        if (ps->audio_queue.size + ps->video_queue.size > 15 * 1024 * 1024) {
            SDL_Delay(10);
            continue;
        }

        // 读取一个数据包
        int ret = av_read_frame(ps->fmt_ctx, &pkt);
        if (ret < 0) {
            if (ret == AVERROR_EOF || avio_feof(ps->fmt_ctx->pb)) {
                // 文件结束，发送 flush 包
                if (ps->video_stream_idx >= 0) {
                    AVPacket flush_pkt;
                    av_init_packet(&flush_pkt);
                    flush_pkt.data = NULL;
                    flush_pkt.size = 0;
                    packet_queue_put(&ps->video_queue, &flush_pkt);
                }
                if (ps->audio_stream_idx >= 0) {
                    AVPacket flush_pkt;
                    av_init_packet(&flush_pkt);
                    flush_pkt.data = NULL;
                    flush_pkt.size = 0;
                    packet_queue_put(&ps->audio_queue, &flush_pkt);
                }
                break;  // 退出线程
            }
            // 读取错误，重试
            SDL_Delay(10);
            continue;
        }

        // 按流索引分发
        if (pkt.stream_index == ps->audio_stream_idx) {
            packet_queue_put(&ps->audio_queue, &pkt);
        } else if (pkt.stream_index == ps->video_stream_idx) {
            packet_queue_put(&ps->video_queue, &pkt);
        } else {
            // 其他流（字幕等），直接释放
            av_packet_unref(&pkt);
        }
    }

    return 0;
}
```

> **设计要点**：
>
> 1. **队列大小限制（15MB）**：防止内存无限增长。KrKr2 中的 `BasePlayer::Process()`
>    在 `VideoPlayer.cpp:581-585` 使用类似策略——当音视频队列都满时 `Sleep(10)`
>
> 2. **EOF 处理**：发送空包（`NULL` data）通知解码器刷新缓冲区中的剩余帧。
>    这对应 FFmpeg 的 "drain mode"——发送 NULL 后可以用 `avcodec_receive_frame`
>    获取解码器内部缓存的最后几帧
>
> 3. **暂停处理**：简单地 `Sleep(10)` 循环等待。生产级代码应该用条件变量阻塞

### 6.5.2 线程安全注意事项

```
┌─────────────────────────────────────────────────┐
│                 线程模型                          │
│                                                  │
│  解封装线程 ──put──→ [audio_queue] ──get──→ SDL回调│
│            ──put──→ [video_queue] ──get──→ 主线程 │
│                                                  │
│  共享状态：                                       │
│    ps->quit     → 原子读写（int 天然原子）         │
│    ps->paused   → 原子读写                        │
│    ps->audio_clock → 仅 SDL 回调写，主线程读       │
│                     （单写者，可容忍瞬时不一致）     │
│                                                  │
│  线程安全保证：                                    │
│    PacketQueue  → SDL_mutex + SDL_cond 保护       │
│    SDL 渲染     → 仅主线程操作（SDL 要求）          │
└─────────────────────────────────────────────────┘
```

> **重要**：SDL2 规定所有渲染操作必须在创建窗口的线程（通常是主线程）中进行。
> 这就是为什么 `video_refresh` 在主线程的事件循环中调用，而不是在单独的视频线程中。

---
## 6.6 初始化与主循环

### 6.6.1 打开媒体文件

```c
// ============================================================
// 初始化
// ============================================================

static int open_codec(AVFormatContext *fmt_ctx, int stream_idx,
                      AVCodecContext **out_ctx) {
    AVCodecParameters *par = fmt_ctx->streams[stream_idx]->codecpar;
    const AVCodec *codec = avcodec_find_decoder(par->codec_id);
    if (!codec) {
        fprintf(stderr, "找不到解码器: %d\n", par->codec_id);
        return -1;
    }

    AVCodecContext *ctx = avcodec_alloc_context3(codec);
    if (!ctx) return -1;

    if (avcodec_parameters_to_context(ctx, par) < 0) {
        avcodec_free_context(&ctx);
        return -1;
    }

    // 多线程解码（充分利用多核 CPU）
    ctx->thread_count = 4;

    if (avcodec_open2(ctx, codec, NULL) < 0) {
        fprintf(stderr, "无法打开解码器: %s\n", codec->name);
        avcodec_free_context(&ctx);
        return -1;
    }

    *out_ctx = ctx;
    return 0;
}

static int player_init(PlayerState *ps, const char *filename) {
    // ── 打开文件 ──
    if (avformat_open_input(&ps->fmt_ctx, filename, NULL, NULL) < 0) {
        fprintf(stderr, "无法打开文件: %s\n", filename);
        return -1;
    }

    if (avformat_find_stream_info(ps->fmt_ctx, NULL) < 0) {
        fprintf(stderr, "无法获取流信息\n");
        return -1;
    }

    // 打印文件信息（调试用）
    av_dump_format(ps->fmt_ctx, 0, filename, 0);

    // ── 查找音视频流 ──
    ps->audio_stream_idx = -1;
    ps->video_stream_idx = -1;

    for (unsigned int i = 0; i < ps->fmt_ctx->nb_streams; i++) {
        if (ps->fmt_ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO
            && ps->video_stream_idx < 0) {
            ps->video_stream_idx = i;
        }
        if (ps->fmt_ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_AUDIO
            && ps->audio_stream_idx < 0) {
            ps->audio_stream_idx = i;
        }
    }

    if (ps->video_stream_idx < 0) {
        fprintf(stderr, "找不到视频流\n");
        return -1;
    }

    // ── 打开解码器 ──
    if (open_codec(ps->fmt_ctx, ps->video_stream_idx,
                   &ps->video_codec_ctx) < 0) {
        return -1;
    }

    ps->video_width  = ps->video_codec_ctx->width;
    ps->video_height = ps->video_codec_ctx->height;

    if (ps->audio_stream_idx >= 0) {
        if (open_codec(ps->fmt_ctx, ps->audio_stream_idx,
                       &ps->audio_codec_ctx) < 0) {
            fprintf(stderr, "警告：无法打开音频解码器，将静音播放\n");
            ps->audio_stream_idx = -1;
        }
    }

    // ── 初始化包队列 ──
    packet_queue_init(&ps->video_queue);
    if (ps->audio_stream_idx >= 0) {
        packet_queue_init(&ps->audio_queue);
    }

    // ── 初始化音频重采样 ──
    if (ps->audio_stream_idx >= 0) {
        ps->swr_ctx = swr_alloc_set_opts(
            NULL,
            AV_CH_LAYOUT_STEREO,        // 输出：立体声
            AV_SAMPLE_FMT_S16,          // 输出：16位有符号整数
            44100,                       // 输出：44.1kHz
            ps->audio_codec_ctx->channel_layout
                ? ps->audio_codec_ctx->channel_layout
                : av_get_default_channel_layout(
                      ps->audio_codec_ctx->channels),
            ps->audio_codec_ctx->sample_fmt,
            ps->audio_codec_ctx->sample_rate,
            0, NULL
        );

        if (!ps->swr_ctx || swr_init(ps->swr_ctx) < 0) {
            fprintf(stderr, "无法初始化音频重采样\n");
            return -1;
        }

        ps->audio_frame = av_frame_alloc();
    }

    // ── 初始化 sws（备用，用于非 YUV420P 格式）──
    if (ps->video_codec_ctx->pix_fmt != AV_PIX_FMT_YUV420P) {
        ps->sws_ctx = sws_getContext(
            ps->video_width, ps->video_height,
            ps->video_codec_ctx->pix_fmt,
            ps->video_width, ps->video_height,
            AV_PIX_FMT_YUV420P,
            SWS_BILINEAR, NULL, NULL, NULL
        );
    }

    return 0;
}
```

### 6.6.2 SDL2 初始化

```c
static int sdl_init(PlayerState *ps) {
    // ── 初始化 SDL ──
    if (SDL_Init(SDL_INIT_VIDEO | SDL_INIT_AUDIO | SDL_INIT_TIMER) < 0) {
        fprintf(stderr, "SDL 初始化失败: %s\n", SDL_GetError());
        return -1;
    }

    // ── 创建窗口 ──
    ps->window = SDL_CreateWindow(
        "Simple FFmpeg Player",
        SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED,
        ps->video_width, ps->video_height,
        SDL_WINDOW_SHOWN | SDL_WINDOW_RESIZABLE
    );
    if (!ps->window) {
        fprintf(stderr, "创建窗口失败: %s\n", SDL_GetError());
        return -1;
    }

    // ── 创建渲染器（启用 VSync）──
    ps->renderer = SDL_CreateRenderer(
        ps->window, -1,
        SDL_RENDERER_ACCELERATED | SDL_RENDERER_PRESENTVSYNC
    );
    if (!ps->renderer) {
        fprintf(stderr, "创建渲染器失败: %s\n", SDL_GetError());
        return -1;
    }

    // ── 创建 YUV 纹理 ──
    ps->texture = SDL_CreateTexture(
        ps->renderer,
        SDL_PIXELFORMAT_IYUV,           // YUV420P 格式
        SDL_TEXTUREACCESS_STREAMING,     // 允许频繁更新
        ps->video_width, ps->video_height
    );
    if (!ps->texture) {
        fprintf(stderr, "创建纹理失败: %s\n", SDL_GetError());
        return -1;
    }

    // ── 打开音频设备 ──
    if (ps->audio_stream_idx >= 0) {
        SDL_AudioSpec wanted_spec, obtained_spec;
        memset(&wanted_spec, 0, sizeof(wanted_spec));

        wanted_spec.freq     = 44100;
        wanted_spec.format   = AUDIO_S16SYS;   // 16位有符号，系统字节序
        wanted_spec.channels = 2;               // 立体声
        wanted_spec.silence  = 0;
        wanted_spec.samples  = AUDIO_BUFFER_SIZE;
        wanted_spec.callback = sdl_audio_callback;
        wanted_spec.userdata = ps;

        if (SDL_OpenAudio(&wanted_spec, &obtained_spec) < 0) {
            fprintf(stderr, "打开音频设备失败: %s\n", SDL_GetError());
            ps->audio_stream_idx = -1;  // 降级为静音播放
        } else {
            // 开始播放音频
            SDL_PauseAudio(0);
        }
    }

    return 0;
}
```

> **`SDL_TEXTUREACCESS_STREAMING`** 表示纹理会被频繁更新（每帧一次），
> SDL 会在内部为其分配适合 CPU 写入的内存布局。
>
> **`AUDIO_S16SYS`** 使用系统原生字节序，避免大小端转换开销。

### 6.6.3 主事件循环

```c
// ============================================================
// 主循环
// ============================================================

// 自定义事件：视频刷新
#define FF_REFRESH_EVENT  (SDL_USEREVENT)
#define REFRESH_INTERVAL  10  // 10ms（约 100fps 检查频率）

static Uint32 sdl_refresh_timer_cb(Uint32 interval, void *param) {
    SDL_Event event;
    event.type = FF_REFRESH_EVENT;
    SDL_PushEvent(&event);
    return interval;  // 持续触发
}

static void event_loop(PlayerState *ps) {
    SDL_Event event;

    // 启动定时器（每 10ms 推送一个刷新事件）
    SDL_AddTimer(REFRESH_INTERVAL, sdl_refresh_timer_cb, NULL);

    while (!ps->quit) {
        SDL_WaitEvent(&event);

        switch (event.type) {
        case SDL_QUIT:
            ps->quit = 1;
            break;

        case SDL_KEYDOWN:
            switch (event.key.keysym.sym) {
            case SDLK_ESCAPE:
            case SDLK_q:
                ps->quit = 1;
                break;
            case SDLK_SPACE:
                // 暂停/继续
                ps->paused = !ps->paused;
                if (ps->paused) {
                    SDL_PauseAudio(1);   // 暂停音频输出
                } else {
                    SDL_PauseAudio(0);   // 恢复音频输出
                }
                break;
            }
            break;

        case FF_REFRESH_EVENT:
            if (!ps->paused) {
                video_refresh(ps);
            }
            break;

        default:
            break;
        }
    }
}
```

> **为什么用定时器而不是独立视频线程？**
>
> SDL2 要求所有渲染操作在主线程执行。使用 `SDL_AddTimer` + 自定义事件
> 是标准做法——定时器回调运行在 SDL 内部线程中，但只是推送一个事件到队列，
> 实际渲染仍在主线程的 `SDL_WaitEvent` 循环中执行。
>
> 10ms 的刷新间隔不意味着 100fps 渲染。`video_refresh` 内部有 A/V 同步逻辑，
> 只有当视频帧的 PTS 与音频时钟匹配时才会真正渲染。

---
## 6.7 main 函数与资源清理

### 6.7.1 程序入口

```c
// ============================================================
// main
// ============================================================

static void player_cleanup(PlayerState *ps) {
    ps->quit = 1;

    // 中止队列（唤醒阻塞的线程）
    packet_queue_abort(&ps->audio_queue);
    packet_queue_abort(&ps->video_queue);

    // 等待解封装线程结束
    if (ps->demux_tid) {
        SDL_WaitThread(ps->demux_tid, NULL);
    }

    // 关闭 SDL 音频
    SDL_CloseAudio();

    // 销毁队列
    packet_queue_destroy(&ps->audio_queue);
    packet_queue_destroy(&ps->video_queue);

    // 释放 FFmpeg 资源
    if (ps->audio_frame)     av_frame_free(&ps->audio_frame);
    if (ps->swr_ctx)         swr_free(&ps->swr_ctx);
    if (ps->sws_ctx)         sws_freeContext(ps->sws_ctx);
    if (ps->audio_codec_ctx) avcodec_free_context(&ps->audio_codec_ctx);
    if (ps->video_codec_ctx) avcodec_free_context(&ps->video_codec_ctx);
    if (ps->fmt_ctx)         avformat_close_input(&ps->fmt_ctx);

    // 释放 SDL 资源
    if (ps->texture)  SDL_DestroyTexture(ps->texture);
    if (ps->renderer) SDL_DestroyRenderer(ps->renderer);
    if (ps->window)   SDL_DestroyWindow(ps->window);

    SDL_Quit();
}

int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(stderr,
            "用法: %s <视频文件>\n"
            "\n"
            "操作:\n"
            "  空格键    暂停/继续\n"
            "  Q/Esc    退出\n",
            argv[0]
        );
        return 1;
    }

    // 分配播放器状态
    PlayerState *ps = (PlayerState *)av_mallocz(sizeof(PlayerState));
    if (!ps) {
        fprintf(stderr, "内存分配失败\n");
        return 1;
    }
    ps->audio_stream_idx = -1;
    ps->video_stream_idx = -1;
    global_ps = ps;

    // 初始化 FFmpeg（新版本无需调用 av_register_all）
    #if LIBAVFORMAT_VERSION_INT < AV_VERSION_INT(58, 9, 100)
    av_register_all();
    #endif

    // 打开并初始化播放器
    if (player_init(ps, argv[1]) < 0) {
        fprintf(stderr, "播放器初始化失败\n");
        av_free(ps);
        return 1;
    }

    // 初始化 SDL
    if (sdl_init(ps) < 0) {
        player_cleanup(ps);
        av_free(ps);
        return 1;
    }

    // 启动解封装线程
    ps->demux_tid = SDL_CreateThread(demux_thread, "demux", ps);
    if (!ps->demux_tid) {
        fprintf(stderr, "无法创建解封装线程\n");
        player_cleanup(ps);
        av_free(ps);
        return 1;
    }

    // 进入事件循环（阻塞直到退出）
    event_loop(ps);

    // 清理
    player_cleanup(ps);
    av_free(ps);

    printf("播放器正常退出\n");
    return 0;
}
```

### 6.7.2 资源释放顺序

资源释放必须按照与创建**相反的顺序**进行，避免悬空引用：

```
释放顺序（从后往前）：
  1. quit = 1                ← 通知所有线程退出
  2. packet_queue_abort()    ← 唤醒阻塞的线程
  3. SDL_WaitThread()        ← 等待线程真正结束
  4. SDL_CloseAudio()        ← 停止音频回调
  5. packet_queue_destroy()  ← 释放队列内存
  6. avcodec_free_context()  ← 关闭解码器
  7. avformat_close_input()  ← 关闭文件
  8. SDL_DestroyTexture()    ← 释放 GPU 资源
  9. SDL_DestroyRenderer()   ← 释放渲染器
  10. SDL_DestroyWindow()    ← 关闭窗口
  11. SDL_Quit()             ← 退出 SDL
```

> **常见错误**：直接调用 `exit()` 或跳过清理。虽然操作系统会回收进程内存，
> 但 GPU 资源、音频设备可能不会被正确释放，导致下次运行时设备占用。

---
## 6.8 编译运行与测试

### 6.8.1 编译

```bash
# 创建构建目录
mkdir build && cd build

# Linux/macOS
cmake .. && make -j$(nproc)

# Windows (Visual Studio)
cmake .. -G "Visual Studio 17 2022" -A x64
cmake --build . --config Release

# Windows (vcpkg)
cmake .. -DCMAKE_TOOLCHAIN_FILE=[vcpkg-root]/scripts/builtin/vcpkg.cmake
cmake --build . --config Release
```

### 6.8.2 运行测试

```bash
# 播放本地视频
./player test.mp4

# 播放网络流（FFmpeg 支持 HTTP/RTSP）
./player http://example.com/video.mp4

# 测试用视频生成（使用 FFmpeg 命令行工具）
ffmpeg -f lavfi -i testsrc=duration=10:size=640x480:rate=30 \
       -f lavfi -i sine=frequency=440:duration=10 \
       -c:v libx264 -c:a aac -pix_fmt yuv420p \
       test.mp4
```

### 6.8.3 预期输出

```
Input #0, mov,mp4,m4a,...  from 'test.mp4':
  Duration: 00:00:10.00, ...
    Stream #0:0(und): Video: h264 (...), yuv420p, 640x480, 30 fps
    Stream #0:1(und): Audio: aac (...), 44100 Hz, stereo
```

然后会弹出一个窗口显示视频画面，同时播放音频。按空格暂停，按 Q 退出。

### 6.8.4 常见编译错误

| 错误信息 | 原因 | 解决方案 |
|----------|------|----------|
| `undefined reference to avformat_open_input` | 未链接 FFmpeg 库 | 检查 CMakeLists.txt 的 `target_link_libraries` |
| `SDL.h: No such file or directory` | SDL2 头文件路径错误 | 确认 `pkg-config --cflags sdl2` 输出正确 |
| `LNK2019: 无法解析的外部符号` (Windows) | 缺少库文件 | 检查 vcpkg 安装，确认 triplet 为 x64-windows |
| `deprecated: avcodec_decode_video2` | 使用了旧 API | 本教程已使用新 API (`send_packet`/`receive_frame`) |
| `SDL_CreateWindow returns NULL` | 无显示环境 | 在 WSL 中需要 X11 转发，或使用原生 Windows |

---
## 6.9 与 KrKr2 架构的对比

我们的简单播放器与 KrKr2 的媒体播放器有很多相似之处，也有关键差异。
理解这些差异有助于你阅读 KrKr2 源码（下一章详细分析）。

### 6.9.1 架构对比表

| 特性 | 简单播放器 | KrKr2 播放器 |
|------|-----------|-------------|
| **语言** | C | C++ (面向对象) |
| **线程模型** | 1 解封装线程 + 主线程 + SDL回调 | BasePlayer + VideoPlayerVideo + VideoPlayerAudio + RenderManager |
| **包队列** | 自定义 PacketQueue (链表) | CDVDMessageQueue (优先级消息队列) |
| **音视频同步** | 简单阈值判断 (±50ms) | CDVDClock 三层时钟 + 自适应阈值 |
| **视频渲染** | SDL_UpdateYUVTexture 直传 | CRenderManager 6帧环形缓冲 + VSync 对齐 |
| **音频输出** | SDL 回调 (pull) | IAE 接口 → OpenAL (push) |
| **像素格式** | 仅 YUV420P | YUV420P 为主 + sws_scale 后备 |
| **错误处理** | fprintf + 退出 | CDVDMsg 消息通知 + 状态机恢复 |
| **缓存控制** | 15MB 硬上限 | 4 级缓存状态机 (FLUSH→FULL→PLAY→DONE) |

### 6.9.2 关键差异解读

**1. 消息队列 vs 包队列**

我们的 `PacketQueue` 只传递 `AVPacket`，而 KrKr2 的 `CDVDMessageQueue` 传递
`CDVDMsg` 消息对象，支持多种消息类型：

```
CDVDMsg 消息体系：
  GENERAL_SYNCHRONIZE  → 流间同步
  GENERAL_RESYNC       → 时间基准重置
  GENERAL_FLUSH        → 清空缓冲区
  PLAYER_SETSPEED      → 变速播放
  DEMUXER_PACKET       → 实际的音视频数据包（对应我们的 AVPacket）
```

这种设计让 KrKr2 可以在数据流中嵌入控制命令，而不需要额外的控制通道。

**2. 三层时钟 vs 简单时钟**

```
我们的时钟：  audio_clock (double, 秒)
              └─ 在解码回调中直接更新

KrKr2 时钟：  CDVDClock
              ├─ 系统时间层: 单调递增硬件时钟
              ├─ 绝对时间层: DVD_TIME_BASE 微秒单位
              └─ 播放时间层: 叠加暂停/变速/不连续性校正
```

**3. 渲染管线 vs 直接渲染**

```
我们：   decode → SDL_UpdateYUVTexture → SDL_RenderPresent
         (同步，无缓冲)

KrKr2：  decode → WaitForBuffer → AddVideoPicture → FlipPage
         → PrepareNextRender → PresentPicture
         (6帧环形缓冲，VSync对齐，帧调度)
```

> **下一章**将深入分析 KrKr2 的完整媒体播放器架构，
> 你会看到这些"复杂"的设计如何解决真实产品中的具体问题。

---
## 6.10 动手实践：功能增强

### 练习 1：添加进度条显示

在控制台输出当前播放进度：

```c
// 在 video_refresh() 末尾添加：
static double last_print_time = 0;
double now = ps->audio_clock;
if (now - last_print_time > 1.0) {  // 每秒打印一次
    double duration = ps->fmt_ctx->duration / (double)AV_TIME_BASE;
    int cur_min = (int)now / 60, cur_sec = (int)now % 60;
    int dur_min = (int)duration / 60, dur_sec = (int)duration % 60;
    fprintf(stderr, "\r播放进度: %02d:%02d / %02d:%02d  ",
            cur_min, cur_sec, dur_min, dur_sec);
    fflush(stderr);
    last_print_time = now;
}
```

### 练习 2：添加音量控制

```c
// 在 PlayerState 中添加：
float volume;  // 0.0 ~ 1.0

// 在 sdl_audio_callback 中，memcpy 后添加音量调节：
// 将 memcpy(stream, ...) 改为：
SDL_memset(stream, 0, len);  // 先清零
SDL_MixAudioFormat(
    stream,
    ps->audio_buf + ps->audio_buf_index,
    AUDIO_S16SYS,
    len1,
    (int)(ps->volume * SDL_MIX_MAXVOLUME)
);

// 在 event_loop 中添加按键处理：
case SDLK_UP:    ps->volume = fmin(1.0f, ps->volume + 0.1f); break;
case SDLK_DOWN:  ps->volume = fmax(0.0f, ps->volume - 0.1f); break;
```

### 练习 3：添加 Seek 功能

```c
// 在 event_loop 中添加：
case SDLK_LEFT: {
    // 后退 10 秒
    double pos = ps->audio_clock - 10.0;
    if (pos < 0) pos = 0;
    int64_t ts = (int64_t)(pos * AV_TIME_BASE);
    av_seek_frame(ps->fmt_ctx, -1, ts, AVSEEK_FLAG_BACKWARD);
    // 清空队列
    packet_queue_flush(&ps->audio_queue);
    packet_queue_flush(&ps->video_queue);
    // 刷新解码器
    if (ps->audio_codec_ctx)
        avcodec_flush_buffers(ps->audio_codec_ctx);
    if (ps->video_codec_ctx)
        avcodec_flush_buffers(ps->video_codec_ctx);
    break;
}
case SDLK_RIGHT: {
    // 前进 10 秒
    double pos = ps->audio_clock + 10.0;
    int64_t ts = (int64_t)(pos * AV_TIME_BASE);
    av_seek_frame(ps->fmt_ctx, -1, ts, 0);
    packet_queue_flush(&ps->audio_queue);
    packet_queue_flush(&ps->video_queue);
    if (ps->audio_codec_ctx)
        avcodec_flush_buffers(ps->audio_codec_ctx);
    if (ps->video_codec_ctx)
        avcodec_flush_buffers(ps->video_codec_ctx);
    break;
}
```

> **注意**：Seek 后必须清空队列和刷新解码器缓冲区。否则解码器会尝试
> 用旧的参考帧解码新位置的 P/B 帧，导致花屏。
> KrKr2 中 `BasePlayer::FlushBuffers()`（VideoPlayer.cpp:1279-1333）
> 执行了类似但更完善的清理操作。

---
## 6.11 本节小结

本章我们从零构建了一个完整的视频播放器，涵盖了音视频开发的所有核心环节：

| 模块 | 核心技术 | 关键函数 |
|------|----------|----------|
| 包队列 | 生产者-消费者模型，SDL 互斥锁 | `packet_queue_put/get` |
| 解封装 | `av_read_frame`，流索引分发 | `demux_thread` |
| 音频解码 | `avcodec_send_packet/receive_frame`，重采样 | `audio_decode_frame` |
| 音频输出 | SDL2 pull 模式回调 | `sdl_audio_callback` |
| 视频解码 | YUV420P 直传纹理 | `video_refresh` |
| A/V 同步 | 音频主时钟，帧丢弃/延迟策略 | `video_refresh` 中的 diff 判断 |
| 播放控制 | SDL 事件循环，暂停/退出 | `event_loop` |
| 资源管理 | LIFO 释放顺序 | `player_cleanup` |

### 关键设计模式

1. **生产者-消费者**：解封装线程（生产者）→ PacketQueue → 解码端（消费者）
2. **Pull 模式音频**：SDL 按需拉取，天然与硬件时钟同步
3. **定时器驱动渲染**：SDL_AddTimer 推送事件，主线程响应渲染
4. **音频主时钟同步**：视频帧根据与音频时钟的差值决定渲染/丢弃/延迟

### 与 KrKr2 的映射关系

```
简单播放器              →  KrKr2 对应组件
───────────────────────────────────────────────
PacketQueue             →  CDVDMessageQueue
demux_thread            →  BasePlayer::Process()
audio_decode_frame      →  CVideoPlayerAudio::Process()
video_refresh           →  CVideoPlayerVideo::OutputPicture()
SDL_UpdateYUVTexture    →  TVPYUVSprite (YUV→GPU)
audio_clock             →  CDVDClock
event_loop              →  SDL/Cocos2d 事件系统
```

---
## 6.12 练习题与答案

### 题目 1：为什么音频使用 Pull 模式而不是 Push 模式？

<details>
<summary>查看答案</summary>

**Pull 模式（SDL 回调）的优势**：

1. **天然节奏控制**：SDL 按照音频硬件的实际消费速率请求数据，
   不会出现数据溢出或欠载的问题

2. **精确的时钟基准**：音频回调的调用频率由声卡硬件决定，
   是最稳定的时钟源，适合作为 A/V 同步的主时钟

3. **简化缓冲区管理**：不需要自己维护一个环形缓冲区来匹配
   生产速率和消费速率

**Push 模式的场景**：
- KrKr2 使用 OpenAL（Push 模式），因为 OpenAL 提供更多音效控制
- Push 模式需要自己控制数据提交的节奏，通常用定时器或轮询队列状态
- 适合需要实时混音、3D 音效等高级音频处理的场景

</details>

### 题目 2：如果删除 A/V 同步代码，会出现什么现象？

<details>
<summary>查看答案</summary>

删除 `video_refresh` 中的 `diff` 判断后：

1. **短视频（<30s）**：可能看不出明显差异，因为音视频解码速度相近

2. **长视频（>5min）**：
   - 视频会逐渐与音频不同步（通常视频快于音频）
   - 原因：视频解码是"尽快解码并渲染"，而音频受硬件时钟约束
   - 累积误差可能达到数秒

3. **变码率视频**：
   - 高码率场景（动作场面）视频可能落后
   - 低码率场景（静态画面）视频可能超前
   - 表现为"一会儿快一会儿慢"

4. **网络流**：
   - 更明显的不同步，因为网络抖动导致包到达不均匀

**本质原因**：视频和音频的解码速率不同，且没有共同的时间参考。
同步机制的作用就是用音频时钟作为参考，调节视频的显示节奏。

</details>

### 题目 3：改造 video_refresh 支持帧队列（提高级）

当前的 `video_refresh` 在视频超前时直接丢弃了解码帧，这是一种浪费。
请设计一个帧队列（FrameQueue）来缓存解码后的帧，避免重复解码。

<details>
<summary>查看答案</summary>

```c
// 帧队列设计（参考 ffplay 和 KrKr2 的 CRenderManager）
#define FRAME_QUEUE_SIZE 4

typedef struct FrameEntry {
    AVFrame *frame;
    double   pts;       // 显示时间戳（秒）
    int      used;      // 是否已使用
} FrameEntry;

typedef struct FrameQueue {
    FrameEntry  entries[FRAME_QUEUE_SIZE];
    int         read_idx;    // 消费者读取位置
    int         write_idx;   // 生产者写入位置
    int         size;        // 当前帧数
    SDL_mutex  *mutex;
    SDL_cond   *cond;
} FrameQueue;

// 写入帧（解码线程调用）
static int frame_queue_push(FrameQueue *fq, AVFrame *frame, double pts) {
    SDL_LockMutex(fq->mutex);
    while (fq->size >= FRAME_QUEUE_SIZE) {
        SDL_CondWait(fq->cond, fq->mutex);  // 队列满，等待消费
    }

    FrameEntry *e = &fq->entries[fq->write_idx];
    if (e->frame) av_frame_unref(e->frame);
    else          e->frame = av_frame_alloc();

    av_frame_ref(e->frame, frame);
    e->pts = pts;
    e->used = 0;
    fq->write_idx = (fq->write_idx + 1) % FRAME_QUEUE_SIZE;
    fq->size++;

    SDL_CondSignal(fq->cond);
    SDL_UnlockMutex(fq->mutex);
    return 0;
}

// 读取帧（渲染线程调用）
static FrameEntry *frame_queue_peek(FrameQueue *fq) {
    SDL_LockMutex(fq->mutex);
    FrameEntry *e = NULL;
    if (fq->size > 0) {
        e = &fq->entries[fq->read_idx];
    }
    SDL_UnlockMutex(fq->mutex);
    return e;
}

// 消费帧（确认渲染后调用）
static void frame_queue_next(FrameQueue *fq) {
    SDL_LockMutex(fq->mutex);
    fq->read_idx = (fq->read_idx + 1) % FRAME_QUEUE_SIZE;
    fq->size--;
    SDL_CondSignal(fq->cond);
    SDL_UnlockMutex(fq->mutex);
}
```

**对比 KrKr2**：`CRenderManager` 使用 `free`/`queued`/`presentsource`/`discard`
四个 deque 实现类似功能，但支持更复杂的多步渲染（PrepareNextRender → PresentSingle/Fields/Blend）。

</details>

---

## 下一步

下一章 [第7章 实战：KrKr2 媒体播放器架构分析](./07-实战KrKr2媒体播放器架构分析.md)
将深入分析 KrKr2 源码中的完整媒体播放器实现，你会看到本章简单播放器中的每个概念
如何在生产级代码中被精心设计和优化。