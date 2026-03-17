# 6.6 初始化与主循环
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
