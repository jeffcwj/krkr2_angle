# 初始化、主循环与 main 函数

> **所属模块：** P06-FFmpeg音视频开发
> **前置知识：** [上一节：音视频解码与渲染](./02-音视频解码与渲染.md)、[第2章：FFmpeg 架构与核心库](../02-FFmpeg架构与核心库/01-FFmpeg项目概览.md)
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：
1. 实现 `open_codec` 辅助函数，封装解码器的查找、分配、配置、打开四步操作
2. 实现 `player_init` 函数，完成文件打开、流查找、解码器初始化、重采样器配置
3. 实现 `sdl_init` 函数，创建窗口/渲染器/纹理/音频设备
4. 实现主事件循环 `event_loop`，理解定时器驱动渲染的设计
5. 编写完整的 `main` 函数和 `player_cleanup` 资源清理函数
6. 能够编译运行整个播放器项目

---

## 解码器打开辅助函数 open_codec

### 为什么封装 open_codec

打开一个 FFmpeg 解码器需要四个步骤：查找解码器 → 分配上下文 → 复制参数 → 打开解码器。音频和视频各需要执行一次，因此封装为通用函数避免代码重复。

### 完整实现

```c
// ============================================================
// open_codec —— 打开指定流的解码器
// 参数：
//   fmt_ctx    —— AVFormatContext*，已打开的容器上下文
//   stream_idx —— 要打开的流索引
//   out_ctx    —— 输出参数，成功时写入分配好的解码器上下文
// 返回值：0 表示成功，-1 表示失败
// ============================================================
static int open_codec(AVFormatContext *fmt_ctx, int stream_idx,
                      AVCodecContext **out_ctx) {
    // 第1步：获取流的编解码器参数
    // AVCodecParameters 包含解码所需的全部参数（codec_id、宽高、采样率等）
    // 它存储在 AVStream 的 codecpar 字段中
    AVCodecParameters *par = fmt_ctx->streams[stream_idx]->codecpar;

    // 第2步：根据 codec_id 查找解码器
    // avcodec_find_decoder 在 FFmpeg 注册的解码器列表中查找
    // 参数：
    //   par->codec_id —— 编解码器标识（如 AV_CODEC_ID_H264、AV_CODEC_ID_AAC）
    // 返回值：找到则返回 AVCodec*，否则返回 NULL
    const AVCodec *codec = avcodec_find_decoder(par->codec_id);
    if (!codec) {
        fprintf(stderr, "找不到解码器: %d\n", par->codec_id);
        return -1;
    }

    // 第3步：分配解码器上下文
    // AVCodecContext 是解码器的运行时状态，保存解码过程中的所有信息
    // avcodec_alloc_context3 分配并初始化一个新的上下文
    // 参数：codec —— 要使用的解码器（用于设置默认值）
    AVCodecContext *ctx = avcodec_alloc_context3(codec);
    if (!ctx) return -1;

    // 第4步：将容器中的编解码器参数复制到上下文
    // avcodec_parameters_to_context 将 AVCodecParameters 的字段
    // （如 width、height、sample_rate、channels 等）复制到 AVCodecContext
    // 这是 FFmpeg 3.x+ 的标准做法（旧版直接使用 AVStream::codec）
    if (avcodec_parameters_to_context(ctx, par) < 0) {
        avcodec_free_context(&ctx);
        return -1;
    }

    // 第5步：配置多线程解码
    // thread_count 设置解码器使用的线程数
    // 设为 4 在大多数平台上是合理的默认值
    // 对于 H.264/H.265 等支持切片级并行的编解码器效果明显
    ctx->thread_count = 4;

    // 第6步：打开解码器
    // avcodec_open2 初始化解码器，分配内部资源
    // 调用后，ctx 就可以用于 avcodec_send_packet/receive_frame
    // 参数：
    //   ctx   —— 解码器上下文
    //   codec —— 解码器（必须与 alloc_context3 时一致）
    //   NULL  —— 可选的 AVDictionary* 参数（如配置硬件加速）
    if (avcodec_open2(ctx, codec, NULL) < 0) {
        fprintf(stderr, "无法打开解码器: %s\n", codec->name);
        avcodec_free_context(&ctx);
        return -1;
    }

    *out_ctx = ctx;
    return 0;
}
```

### 函数调用流程图

```
open_codec(fmt_ctx, stream_idx, &out_ctx)
    │
    ├─ avcodec_find_decoder(codec_id)     ← 查找解码器
    │     找到 H.264 解码器 / AAC 解码器等
    │
    ├─ avcodec_alloc_context3(codec)       ← 分配上下文
    │     分配 ~2KB 的 AVCodecContext 结构
    │
    ├─ avcodec_parameters_to_context()     ← 复制参数
    │     width=1920, height=1080, pix_fmt=YUV420P 等
    │
    ├─ ctx->thread_count = 4               ← 配置多线程
    │
    └─ avcodec_open2(ctx, codec, NULL)     ← 打开解码器
          初始化内部表、分配缓冲区等
```

### KrKr2 对照

KrKr2 中解码器打开分散在各个解码器类的 `Open()` 方法中：
- 视频：`CDVDVideoCodecFFmpeg::Open()`（`VideoCodecFFmpeg.cpp` 第 170-260 行）
- 音频：`CDVDAudioCodecFFmpeg::Open()`（`AudioCodecFFmpeg.cpp`）

与我们的区别是 KrKr2 额外支持硬件加速解码（`GetFormat` 回调函数协商 GPU 解码格式）。

---

## 播放器初始化 player_init

### 初始化阶段总览

`player_init` 是播放器的核心初始化函数，按以下顺序完成所有准备工作：

```
player_init 执行流程：

  1. avformat_open_input      ← 打开文件/URL
  2. avformat_find_stream_info ← 探测流信息
  3. 遍历流，找到音频和视频   ← 确定 stream_index
  4. open_codec × 2           ← 打开音频/视频解码器
  5. packet_queue_init × 2    ← 初始化包队列
  6. swr_alloc_set_opts       ← 配置音频重采样器
  7. sws_getContext（可选）    ← 配置像素格式转换器
```

### 完整实现

```c
// ============================================================
// player_init —— 初始化播放器
// 打开文件、查找流、初始化解码器、配置重采样器
// 参数：
//   ps       —— 播放器状态（已 av_mallocz 清零）
//   filename —— 媒体文件路径或 URL
// 返回值：0 成功，-1 失败
// ============================================================
static int player_init(PlayerState *ps, const char *filename) {

    // ──────── 第1步：打开媒体文件 ────────
    // avformat_open_input 打开文件并读取文件头，确定容器格式
    // 参数：
    //   &ps->fmt_ctx —— 输出参数，成功时分配 AVFormatContext
    //   filename     —— 文件路径、URL（http://）、管道（pipe:）等
    //   NULL         —— 强制指定容器格式（NULL = 自动检测）
    //   NULL         —— AVDictionary* 额外选项（如超时设置）
    if (avformat_open_input(&ps->fmt_ctx, filename, NULL, NULL) < 0) {
        fprintf(stderr, "无法打开文件: %s\n", filename);
        return -1;
    }

    // ──────── 第2步：探测流信息 ────────
    // avformat_find_stream_info 读取一段数据来分析流参数
    // 某些容器（如 MPEG-TS）的头部不包含完整信息，
    // 需要实际读取几秒数据才能确定编码格式、帧率等
    if (avformat_find_stream_info(ps->fmt_ctx, NULL) < 0) {
        fprintf(stderr, "无法获取流信息\n");
        return -1;
    }

    // 打印文件信息到 stderr（调试用）
    // av_dump_format 输出类似 ffprobe 的详细格式信息
    av_dump_format(ps->fmt_ctx, 0, filename, 0);

    // ──────── 第3步：查找音视频流 ────────
    // 遍历所有流，找到第一个视频流和第一个音频流
    ps->audio_stream_idx = -1;  // -1 表示未找到
    ps->video_stream_idx = -1;

    for (unsigned int i = 0; i < ps->fmt_ctx->nb_streams; i++) {
        // codec_type 标识流的类型：
        //   AVMEDIA_TYPE_VIDEO = 0（视频）
        //   AVMEDIA_TYPE_AUDIO = 1（音频）
        //   AVMEDIA_TYPE_SUBTITLE = 3（字幕）
        if (ps->fmt_ctx->streams[i]->codecpar->codec_type
            == AVMEDIA_TYPE_VIDEO && ps->video_stream_idx < 0) {
            ps->video_stream_idx = i;
        }
        if (ps->fmt_ctx->streams[i]->codecpar->codec_type
            == AVMEDIA_TYPE_AUDIO && ps->audio_stream_idx < 0) {
            ps->audio_stream_idx = i;
        }
    }

    if (ps->video_stream_idx < 0) {
        fprintf(stderr, "找不到视频流\n");
        return -1;
    }

    // ──────── 第4步：打开解码器 ────────
    // 视频解码器（必须）
    if (open_codec(ps->fmt_ctx, ps->video_stream_idx,
                   &ps->video_codec_ctx) < 0) {
        return -1;
    }

    // 记录视频尺寸（后续创建窗口和纹理用）
    ps->video_width  = ps->video_codec_ctx->width;
    ps->video_height = ps->video_codec_ctx->height;

    // 音频解码器（可选，失败时降级为静音播放）
    if (ps->audio_stream_idx >= 0) {
        if (open_codec(ps->fmt_ctx, ps->audio_stream_idx,
                       &ps->audio_codec_ctx) < 0) {
            fprintf(stderr, "警告：无法打开音频解码器，将静音播放\n");
            ps->audio_stream_idx = -1;  // 标记为"无音频"
        }
    }

    // ──────── 第5步：初始化包队列 ────────
    packet_queue_init(&ps->video_queue);
    if (ps->audio_stream_idx >= 0) {
        packet_queue_init(&ps->audio_queue);
    }

    // ──────── 第6步：初始化音频重采样器 ────────
    if (ps->audio_stream_idx >= 0) {
        // swr_alloc_set_opts 创建并配置一个 SwrContext（重采样上下文）
        // 参数（按顺序）：
        //   NULL                        —— 传 NULL 表示新建（非复用已有的）
        //   AV_CH_LAYOUT_STEREO         —— 输出声道布局：立体声（2声道）
        //   AV_SAMPLE_FMT_S16           —— 输出采样格式：16位有符号整数交错
        //   44100                       —— 输出采样率：44100Hz（CD 品质）
        //   channel_layout              —— 输入声道布局（从解码器获取）
        //   sample_fmt                  —— 输入采样格式（可能是 FLTP 等）
        //   sample_rate                 —— 输入采样率（可能是 48000 等）
        //   0, NULL                     —— 日志偏移和日志上下文（不使用）
        ps->swr_ctx = swr_alloc_set_opts(
            NULL,
            AV_CH_LAYOUT_STEREO,        // 输出：立体声
            AV_SAMPLE_FMT_S16,          // 输出：16位有符号整数
            44100,                       // 输出：44.1kHz
            // 输入声道布局：优先使用解码器报告的，否则根据声道数推断
            ps->audio_codec_ctx->channel_layout
                ? ps->audio_codec_ctx->channel_layout
                : av_get_default_channel_layout(
                      ps->audio_codec_ctx->channels),
            ps->audio_codec_ctx->sample_fmt,    // 输入格式
            ps->audio_codec_ctx->sample_rate,   // 输入采样率
            0, NULL
        );

        if (!ps->swr_ctx || swr_init(ps->swr_ctx) < 0) {
            fprintf(stderr, "无法初始化音频重采样\n");
            return -1;
        }

        // 分配用于接收解码帧的 AVFrame
        ps->audio_frame = av_frame_alloc();
    }

    // ──────── 第7步：初始化像素格式转换器（可选）────────
    // 如果视频不是 YUV420P 格式，需要转换后才能用 SDL_UpdateYUVTexture
    // sws_getContext 创建 SwsContext（像素格式转换上下文）
    // 参数：
    //   srcW, srcH, srcFormat —— 源尺寸和格式
    //   dstW, dstH, dstFormat —— 目标尺寸和格式
    //   flags —— 缩放算法（SWS_BILINEAR = 双线性插值，速度与质量平衡）
    //   后三个 NULL —— 源/目标过滤器和参数（高级用法，通常不需要）
    if (ps->video_codec_ctx->pix_fmt != AV_PIX_FMT_YUV420P) {
        ps->sws_ctx = sws_getContext(
            ps->video_width, ps->video_height,
            ps->video_codec_ctx->pix_fmt,   // 源格式
            ps->video_width, ps->video_height,
            AV_PIX_FMT_YUV420P,             // 目标格式
            SWS_BILINEAR,                    // 缩放算法
            NULL, NULL, NULL
        );
    }

    return 0;
}
```

### 初始化参数选择说明

**为什么输出固定 44100Hz / S16 / Stereo？**

| 参数 | 选择 | 原因 |
|------|------|------|
| 采样率 44100Hz | CD 标准采样率 | 所有声卡都支持，与 SDL 默认配置匹配 |
| 格式 S16 | 16位有符号整数 | SDL 的 `AUDIO_S16SYS` 直接对应，无需额外转换 |
| 声道 Stereo | 双声道 | 覆盖 99% 的音频设备，单声道源会被 upmix（上混）|

生产级播放器（如 KrKr2）会查询音频设备实际支持的格式，动态选择输出参数。

**为什么要处理 channel_layout 为 0 的情况？**

某些编码器不设置 `channel_layout` 字段（留 0）。`av_get_default_channel_layout(channels)` 根据声道数推断布局：
- 1 声道 → `AV_CH_LAYOUT_MONO`
- 2 声道 → `AV_CH_LAYOUT_STEREO`
- 6 声道 → `AV_CH_LAYOUT_5POINT1`

---

## SDL2 初始化 sdl_init

### SDL 子系统

SDL2 采用模块化设计，每个功能是一个独立的"子系统"（subsystem）。我们需要初始化三个子系统：

| 子系统 | 标志 | 用途 |
|--------|------|------|
| Video | `SDL_INIT_VIDEO` | 窗口创建、渲染、纹理 |
| Audio | `SDL_INIT_AUDIO` | 音频设备打开、回调 |
| Timer | `SDL_INIT_TIMER` | 定时器（驱动视频刷新） |

### 完整实现

```c
// 音频缓冲区大小（样本数，不是字节数）
// 1024 样本 ÷ 44100Hz ≈ 23ms 的音频数据
// 值越小延迟越低，但 CPU 开销越大（回调更频繁）
#define AUDIO_BUFFER_SIZE 1024

// ============================================================
// sdl_init —— 初始化 SDL2 子系统、窗口、渲染器、纹理、音频设备
// 参数：ps —— 播放器状态（video_width/height 已设置）
// 返回值：0 成功，-1 失败
// ============================================================
static int sdl_init(PlayerState *ps) {
    // ── 初始化 SDL 子系统 ──
    // SDL_Init 初始化指定的子系统，可用 | 组合多个
    // 返回值：0 成功，负值失败（错误信息通过 SDL_GetError 获取）
    if (SDL_Init(SDL_INIT_VIDEO | SDL_INIT_AUDIO | SDL_INIT_TIMER) < 0) {
        fprintf(stderr, "SDL 初始化失败: %s\n", SDL_GetError());
        return -1;
    }

    // ── 创建窗口 ──
    // SDL_CreateWindow 创建一个操作系统窗口
    // 参数：
    //   title  —— 窗口标题栏文字
    //   x, y   —— 窗口位置（SDL_WINDOWPOS_CENTERED = 屏幕居中）
    //   w, h   —— 窗口尺寸（使用视频的原始分辨率）
    //   flags  —— SDL_WINDOW_SHOWN（立即显示）
    //            + SDL_WINDOW_RESIZABLE（允许用户拖拽调整大小）
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

    // ── 创建渲染器 ──
    // SDL_CreateRenderer 创建一个 2D 渲染器（底层可能是 OpenGL/DirectX/Metal）
    // 参数：
    //   window  —— 关联的窗口
    //   -1      —— 渲染驱动索引（-1 = 自动选择最佳）
    //   flags   —— SDL_RENDERER_ACCELERATED（GPU 加速）
    //            + SDL_RENDERER_PRESENTVSYNC（启用垂直同步，防止画面撕裂）
    ps->renderer = SDL_CreateRenderer(
        ps->window, -1,
        SDL_RENDERER_ACCELERATED | SDL_RENDERER_PRESENTVSYNC
    );
    if (!ps->renderer) {
        fprintf(stderr, "创建渲染器失败: %s\n", SDL_GetError());
        return -1;
    }

    // ── 创建 YUV 纹理 ──
    // SDL_CreateTexture 创建一个 GPU 纹理
    // 参数：
    //   renderer —— 关联的渲染器
    //   format   —— 像素格式
    //              SDL_PIXELFORMAT_IYUV = YUV420P（与 FFmpeg 的 AV_PIX_FMT_YUV420P 对应）
    //   access   —— 访问模式
    //              SDL_TEXTUREACCESS_STREAMING = 允许频繁更新（每帧更新一次）
    //              SDL 会为其分配适合 CPU 写入的内存布局
    //   w, h     —— 纹理尺寸
    ps->texture = SDL_CreateTexture(
        ps->renderer,
        SDL_PIXELFORMAT_IYUV,
        SDL_TEXTUREACCESS_STREAMING,
        ps->video_width, ps->video_height
    );
    if (!ps->texture) {
        fprintf(stderr, "创建纹理失败: %s\n", SDL_GetError());
        return -1;
    }

    // ── 打开音频设备 ──
    if (ps->audio_stream_idx >= 0) {
        // SDL_AudioSpec 描述音频设备的参数
        SDL_AudioSpec wanted_spec, obtained_spec;
        memset(&wanted_spec, 0, sizeof(wanted_spec));

        wanted_spec.freq     = 44100;          // 采样率（Hz）
        wanted_spec.format   = AUDIO_S16SYS;   // 16位有符号，系统原生字节序
        wanted_spec.channels = 2;              // 双声道
        wanted_spec.silence  = 0;              // 静音值（S16 格式下为 0）
        wanted_spec.samples  = AUDIO_BUFFER_SIZE; // 缓冲区大小（样本数）
        wanted_spec.callback = sdl_audio_callback; // 音频回调函数
        wanted_spec.userdata = ps;             // 传给回调的自定义数据

        // SDL_OpenAudio 打开默认音频设备
        // wanted_spec：我们期望的参数
        // obtained_spec：设备实际支持的参数（可能与 wanted 不同）
        if (SDL_OpenAudio(&wanted_spec, &obtained_spec) < 0) {
            fprintf(stderr, "打开音频设备失败: %s\n", SDL_GetError());
            ps->audio_stream_idx = -1;  // 降级为静音播放
        } else {
            // SDL_PauseAudio(0) 开始播放（SDL 打开设备后默认是暂停状态）
            // 参数：0 = 取消暂停（开始播放），1 = 暂停
            SDL_PauseAudio(0);
        }
    }

    return 0;
}
```

### SDL 纹理格式对照

| SDL 格式 | FFmpeg 格式 | 说明 |
|----------|-------------|------|
| `SDL_PIXELFORMAT_IYUV` | `AV_PIX_FMT_YUV420P` | YUV 4:2:0 平面格式 |
| `SDL_PIXELFORMAT_NV12` | `AV_PIX_FMT_NV12` | Y + UV 交错（硬件解码常用） |
| `SDL_PIXELFORMAT_RGB24` | `AV_PIX_FMT_RGB24` | RGB 无 Alpha |
| `SDL_PIXELFORMAT_ARGB8888` | `AV_PIX_FMT_BGRA` | 含 Alpha 通道（注意 BGRA 顺序） |

### 常见初始化错误

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| `SDL_CreateWindow returns NULL` | 无显示环境（如 SSH/WSL） | Windows 用原生环境；Linux 需 X11 或 Wayland |
| `SDL_CreateRenderer returns NULL` | GPU 驱动不支持 | 去掉 `SDL_RENDERER_ACCELERATED` 使用软件渲染 |
| 窗口创建成功但黑屏 | 未调用 `SDL_RenderPresent` | 确保事件循环中调用了渲染流程 |
| 音频设备打开失败 | 设备被占用或不支持 S16 | 检查 `obtained_spec` 使用实际参数 |
| `AUDIO_S16SYS` 是什么 | 系统原生字节序的 16 位有符号整数 | 大端系统 = `AUDIO_S16MSB`，小端 = `AUDIO_S16LSB` |

---

## 主事件循环 event_loop

### 定时器驱动渲染

SDL2 的渲染 API 必须在主线程调用。我们使用 `SDL_AddTimer` 定时器每隔固定间隔向事件队列推送自定义事件，主线程在事件循环中响应这些事件来触发视频渲染。

```
定时器驱动渲染流程：

  SDL 定时器线程 ──每 10ms──→ SDL_PushEvent(FF_REFRESH_EVENT)
                                    │
                                    ↓
  主线程 ──SDL_WaitEvent()──→ 收到 FF_REFRESH_EVENT
                                    │
                                    ↓
                             video_refresh(ps) → 解码+同步+渲染
```

### 完整实现

```c
// 自定义事件类型（必须 >= SDL_USEREVENT）
// SDL_USEREVENT 是 SDL 预留给用户的事件类型起始值
#define FF_REFRESH_EVENT  (SDL_USEREVENT)

// 刷新间隔（毫秒）
// 10ms ≈ 100Hz 的检查频率，远高于常见的 24/30/60fps
// 这保证了足够的刷新精度，实际渲染帧率由 A/V 同步逻辑控制
#define REFRESH_INTERVAL  10

// ============================================================
// sdl_refresh_timer_cb —— SDL 定时器回调
// 运行在 SDL 内部的定时器线程中（不是主线程！）
// 功能：向事件队列推送一个刷新事件，由主线程处理
// 参数：
//   interval —— 定时器间隔（毫秒）
//   param    —— 用户数据（未使用）
// 返回值：返回 interval 表示继续定时，返回 0 表示停止
// ============================================================
static Uint32 sdl_refresh_timer_cb(Uint32 interval, void *param) {
    SDL_Event event;
    event.type = FF_REFRESH_EVENT;
    SDL_PushEvent(&event);  // 线程安全地将事件推入队列
    return interval;        // 返回 interval = 持续触发
}

// ============================================================
// event_loop —— 主事件循环
// 处理用户输入（键盘、关闭窗口）和视频刷新事件
// 参数：ps —— 播放器状态
// ============================================================
static void event_loop(PlayerState *ps) {
    SDL_Event event;

    // 启动定时器：每 REFRESH_INTERVAL 毫秒触发一次
    // SDL_AddTimer 返回一个定时器 ID（可用于 SDL_RemoveTimer 取消）
    SDL_AddTimer(REFRESH_INTERVAL, sdl_refresh_timer_cb, NULL);

    while (!ps->quit) {
        // SDL_WaitEvent 阻塞等待下一个事件
        // 与 SDL_PollEvent 的区别：
        //   WaitEvent —— 无事件时休眠，CPU 占用极低
        //   PollEvent —— 无事件时立即返回，需要手动 Sleep 控制帧率
        SDL_WaitEvent(&event);

        switch (event.type) {
        case SDL_QUIT:
            // 用户关闭窗口（点击 × 按钮）
            ps->quit = 1;
            break;

        case SDL_KEYDOWN:
            switch (event.key.keysym.sym) {
            case SDLK_ESCAPE:
            case SDLK_q:
                // ESC 或 Q 键退出
                ps->quit = 1;
                break;

            case SDLK_SPACE:
                // 空格键切换暂停/继续
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
            // 定时器触发的视频刷新事件
            if (!ps->paused) {
                video_refresh(ps);  // 解码 + A/V 同步 + 渲染
            }
            break;

        default:
            break;
        }
    }
}
```

### 为什么用定时器而不是独立视频线程

| 方案 | 优点 | 缺点 |
|------|------|------|
| 定时器 + 事件（我们用的） | 渲染在主线程，符合 SDL 要求 | 定时精度受事件队列拥堵影响 |
| 独立视频线程 | 不受事件队列影响 | SDL 渲染 API 不能在非主线程调用 |
| 主线程轮询（PollEvent） | 最简单 | CPU 占用高（busy loop），不推荐 |

**10ms 间隔不意味着 100fps**：`video_refresh` 内部有 A/V 同步逻辑，只有当视频帧的 PTS 与音频时钟在 ±50ms 范围内时才会渲染。10ms 的高频检查确保"不错过"任何应该渲染的时刻。

### 与 KrKr2 的事件循环对比

KrKr2 使用 Cocos2d-x 的主循环（`Director::mainLoop()`），而不是 SDL 事件循环：

```
我们的事件循环：
  SDL_WaitEvent → switch(event.type) → video_refresh / 键盘处理

KrKr2 的主循环：
  Director::mainLoop()
    → Scene::update(dt)        ← Cocos2d 场景更新
    → Renderer::render()       ← Cocos2d 渲染管线
    → TVPMoviePlayer::OnTimer() ← 视频帧更新（如果正在播放）
```

KrKr2 的视频渲染集成在 Cocos2d-x 的渲染管线中，通过 `TVPYUVSprite`（一个 Cocos2d Sprite 子类）将视频帧作为纹理显示。

---

## main 函数与资源清理

### 程序入口

`main` 函数是整个程序的入口点，按照"初始化 → 启动线程 → 进入事件循环 → 清理退出"的流程运行。

```c
// ============================================================
// player_cleanup —— 资源清理（析构函数角色）
// 按照与创建相反的顺序释放所有资源
// ============================================================
static void player_cleanup(PlayerState *ps) {
    // 1. 通知所有线程退出
    ps->quit = 1;

    // 2. 中止包队列 —— 唤醒阻塞在 packet_queue_get 中的线程
    //    如果不先中止，SDL_WaitThread 会死锁（线程永远阻塞在队列上）
    packet_queue_abort(&ps->audio_queue);
    packet_queue_abort(&ps->video_queue);

    // 3. 等待解封装线程结束
    //    SDL_WaitThread 阻塞直到指定线程退出
    //    必须在释放共享资源之前等待，否则线程访问已释放的内存 → 崩溃
    if (ps->demux_tid) {
        SDL_WaitThread(ps->demux_tid, NULL);
    }

    // 4. 关闭 SDL 音频 —— 停止音频回调
    //    必须在释放解码器之前调用，否则回调仍会访问 audio_codec_ctx
    SDL_CloseAudio();

    // 5. 销毁包队列（释放队列中剩余的 AVPacket）
    packet_queue_destroy(&ps->audio_queue);
    packet_queue_destroy(&ps->video_queue);

    // 6. 释放 FFmpeg 资源（按依赖关系逆序释放）
    if (ps->audio_frame)     av_frame_free(&ps->audio_frame);
    if (ps->swr_ctx)         swr_free(&ps->swr_ctx);
    if (ps->sws_ctx)         sws_freeContext(ps->sws_ctx);
    if (ps->audio_codec_ctx) avcodec_free_context(&ps->audio_codec_ctx);
    if (ps->video_codec_ctx) avcodec_free_context(&ps->video_codec_ctx);
    if (ps->fmt_ctx)         avformat_close_input(&ps->fmt_ctx);

    // 7. 释放 SDL 资源（按创建逆序）
    //    纹理 → 渲染器 → 窗口（子对象先于父对象释放）
    if (ps->texture)  SDL_DestroyTexture(ps->texture);
    if (ps->renderer) SDL_DestroyRenderer(ps->renderer);
    if (ps->window)   SDL_DestroyWindow(ps->window);

    // 8. 退出 SDL（释放所有子系统）
    SDL_Quit();
}

// ============================================================
// main —— 程序入口
// ============================================================
int main(int argc, char *argv[]) {
    // 检查命令行参数
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

    // 分配播放器状态（av_mallocz 分配内存并清零）
    // 使用 av_mallocz 而非 malloc 是为了与 FFmpeg 的内存管理系统一致
    // av_mallocz 保证内存对齐（通常 32 字节），某些 SIMD 指令需要对齐
    PlayerState *ps = (PlayerState *)av_mallocz(sizeof(PlayerState));
    if (!ps) {
        fprintf(stderr, "内存分配失败\n");
        return 1;
    }
    ps->audio_stream_idx = -1;  // 初始化为"未找到"
    ps->video_stream_idx = -1;
    global_ps = ps;  // 全局指针，用于信号处理等场景

    // FFmpeg 版本兼容处理
    // FFmpeg 4.0 之前需要调用 av_register_all() 注册所有编解码器
    // FFmpeg 4.0+ 自动注册，不再需要此调用
    #if LIBAVFORMAT_VERSION_INT < AV_VERSION_INT(58, 9, 100)
    av_register_all();
    #endif

    // 初始化 FFmpeg 部分（打开文件、找流、开解码器）
    if (player_init(ps, argv[1]) < 0) {
        fprintf(stderr, "播放器初始化失败\n");
        av_free(ps);
        return 1;
    }

    // 初始化 SDL 部分（窗口、渲染器、纹理、音频设备）
    if (sdl_init(ps) < 0) {
        player_cleanup(ps);
        av_free(ps);
        return 1;
    }

    // 启动解封装线程
    // SDL_CreateThread 创建一个新线程并立即开始执行
    // 参数：
    //   demux_thread —— 线程函数
    //   "demux"      —— 线程名称（调试用，在任务管理器中可见）
    //   ps           —— 传给线程函数的参数
    ps->demux_tid = SDL_CreateThread(demux_thread, "demux", ps);
    if (!ps->demux_tid) {
        fprintf(stderr, "无法创建解封装线程\n");
        player_cleanup(ps);
        av_free(ps);
        return 1;
    }

    // 进入主事件循环（阻塞直到用户退出）
    event_loop(ps);

    // 清理所有资源
    player_cleanup(ps);
    av_free(ps);

    printf("播放器正常退出\n");
    return 0;
}
```

### 资源释放顺序图

```
释放顺序（从后往前，与创建顺序相反）：

  1. ps->quit = 1              ← 通知所有线程退出
  2. packet_queue_abort() ×2    ← 唤醒阻塞在队列上的线程
  3. SDL_WaitThread()          ← 等待解封装线程真正结束
  ────── 线程已全部停止，下面可以安全释放资源 ──────
  4. SDL_CloseAudio()          ← 停止音频回调（释放音频设备）
  5. packet_queue_destroy() ×2 ← 释放队列中残留的 AVPacket
  6. av_frame_free()           ← 释放音频帧
  7. swr_free()                ← 释放重采样器
  8. sws_freeContext()         ← 释放像素转换器
  9. avcodec_free_context() ×2 ← 关闭并释放解码器
  10. avformat_close_input()   ← 关闭文件/URL
  ────── FFmpeg 资源已释放 ──────
  11. SDL_DestroyTexture()     ← 释放 GPU 纹理
  12. SDL_DestroyRenderer()    ← 释放渲染器
  13. SDL_DestroyWindow()      ← 关闭窗口
  14. SDL_Quit()               ← 退出所有 SDL 子系统
  15. av_free(ps)              ← 释放 PlayerState 自身
```

**为什么顺序如此重要？**

错误的释放顺序会导致：

| 错误顺序 | 后果 |
|----------|------|
| 先释放 codec_ctx 再 SDL_CloseAudio | 音频回调仍在运行，访问已释放的解码器 → 崩溃 |
| 先释放 fmt_ctx 再 SDL_WaitThread | 解封装线程仍在 av_read_frame → 段错误 |
| 先 SDL_DestroyRenderer 再 SDL_DestroyTexture | 纹理依赖渲染器，先删渲染器导致未定义行为 |
| 直接 exit() 不清理 | GPU 资源泄漏，音频设备被占用，下次启动可能失败 |

---

## 编译运行

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)
project(simple_player C)

# 查找 FFmpeg 和 SDL2
find_package(PkgConfig REQUIRED)
pkg_check_modules(FFMPEG REQUIRED
    libavformat libavcodec libavutil libswscale libswresample)
pkg_check_modules(SDL2 REQUIRED sdl2)

add_executable(player
    main.c          # 包含所有函数的单文件
)

target_include_directories(player PRIVATE
    ${FFMPEG_INCLUDE_DIRS}
    ${SDL2_INCLUDE_DIRS}
)

target_link_libraries(player PRIVATE
    ${FFMPEG_LIBRARIES}
    ${SDL2_LIBRARIES}
)
```

### 编译命令

```bash
# ── Linux / macOS ──
mkdir build && cd build
cmake ..
make -j$(nproc)

# ── Windows（使用 vcpkg）──
mkdir build && cd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%/scripts/buildsystems/vcpkg.cmake
cmake --build . --config Release

# ── Windows（Visual Studio 生成器）──
cmake .. -G "Visual Studio 17 2022" -A x64 ^
  -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%/scripts/buildsystems/vcpkg.cmake
cmake --build . --config Release
```

### 测试运行

```bash
# 生成测试视频（需要 ffmpeg 命令行工具）
ffmpeg -f lavfi -i testsrc=duration=10:size=640x480:rate=30 \
       -f lavfi -i sine=frequency=440:duration=10 \
       -c:v libx264 -c:a aac -pix_fmt yuv420p \
       test.mp4

# 运行播放器
./player test.mp4

# 预期输出：
# Input #0, mov,mp4,m4a,... from 'test.mp4':
#   Duration: 00:00:10.00, ...
#     Stream #0:0: Video: h264 (...), yuv420p, 640x480, 30 fps
#     Stream #0:1: Audio: aac (...), 44100 Hz, stereo
# （弹出窗口显示测试图案 + 440Hz 正弦音）
```

### 常见编译错误

| 错误信息 | 平台 | 解决方案 |
|----------|------|----------|
| `undefined reference to avformat_open_input` | Linux | 检查 `pkg-config --libs libavformat` 输出 |
| `SDL.h: No such file or directory` | Linux | `sudo apt install libsdl2-dev` |
| `LNK2019: 无法解析的外部符号` | Windows | 检查 vcpkg 安装，确认 triplet 为 `x64-windows` |
| `deprecated: avcodec_decode_video2` | 全平台 | 本教程已使用新 API，此警告来自旧代码 |
| `SDL_CreateWindow returns NULL` | WSL | WSL 需要 X11 转发（`export DISPLAY=:0`），或用原生 Windows |
| `cmake: PkgConfig not found` | macOS | `brew install pkg-config`；或用 `find_package(SDL2)` 替代 |

---

## 对照项目源码

### 初始化对比

| 步骤 | 简单播放器 | KrKr2 |
|------|-----------|-------|
| 打开文件 | `avformat_open_input` | `CDVDDemuxFFmpeg::Open()` |
| 查找流 | 遍历 `nb_streams` | `av_find_best_stream` + 手动筛选 |
| 打开解码器 | `open_codec` 通用函数 | 各解码器类各自的 `Open()` 方法 |
| 重采样器 | `swr_alloc_set_opts` | 在 `CVideoPlayerAudio::Process()` 中按需创建 |
| SDL 初始化 | `sdl_init` 集中初始化 | Cocos2d-x 框架自动处理 |

### 主循环对比

```
简单播放器：
  main()
    → player_init()     ← FFmpeg 初始化
    → sdl_init()        ← SDL 初始化
    → SDL_CreateThread(demux_thread)  ← 启动解封装
    → event_loop()      ← 阻塞运行，处理事件和渲染
    → player_cleanup()  ← 退出清理

KrKr2：
  main()
    → TVPAppDelegate::run()
      → applicationDidFinishLaunching()
        → Director::getInstance()->runWithScene(scene)
          → mainLoop()   ← Cocos2d 主循环（包含渲染）
            → TVPMoviePlayer::OnTimer()  ← 视频帧更新
```

关键差异：KrKr2 的视频播放嵌入在 Cocos2d-x 的主循环中，视频帧作为 `TVPYUVSprite`（一个 Cocos2d Sprite 子类）参与场景渲染。我们的简单播放器使用独立的 SDL 事件循环。

相关文件：
- `cpp/core/movie/ffmpeg/VideoPlayer.cpp` —— `BasePlayer::OpenInputStream()`（初始化入口）
- `cpp/core/movie/ffmpeg/DemuxFFmpeg.cpp` —— `CDVDDemuxFFmpeg::Open()`（解封装器初始化）
- `cpp/core/environ/cocos2d/AppDelegate.cpp` —— 应用入口和 Cocos2d 初始化

---

## 本节小结

- `open_codec`：封装了解码器查找→分配→参数复制→打开四步操作，音视频共用
- `player_init`：完成文件打开、流查找、解码器初始化、重采样器配置，是播放器的"构造函数"
- `sdl_init`：初始化 SDL 子系统、创建窗口/渲染器/YUV纹理、打开音频设备
- `event_loop`：定时器驱动的主事件循环，处理用户输入和视频刷新
- `player_cleanup`：按创建逆序释放所有资源，先停线程再释放资源
- 资源释放顺序至关重要：线程停止 → 音频关闭 → FFmpeg 资源 → SDL 资源

## 常见错误

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| 初始化时崩溃 | `PlayerState` 未清零 | 使用 `av_mallocz`（清零分配）而非 `malloc` |
| 退出时段错误 | 释放顺序错误 | 先停线程，再关音频，最后释放解码器和 SDL |
| 音频不播放 | 忘记 `SDL_PauseAudio(0)` | SDL 默认暂停状态，必须显式取消暂停 |
| 窗口闪退 | `event_loop` 未调用 | 确保 `main` 中在 `SDL_CreateThread` 之后调用 `event_loop` |

## 练习题与答案

### 题目 1：`av_register_all()` 在什么情况下需要调用？为什么新版 FFmpeg 不需要？

<details>
<summary>查看答案</summary>

**FFmpeg 3.x 及之前版本**：必须在使用任何 FFmpeg 函数之前调用 `av_register_all()`。这个函数将所有编解码器、解封装器、协议处理器注册到全局链表中。不调用的话，`avcodec_find_decoder` 会找不到任何解码器。

**FFmpeg 4.0（libavformat 58.9.100）及之后版本**：FFmpeg 改用了编译期静态注册机制。所有编解码器在编译时就被注册到全局数组中，运行时无需额外注册。`av_register_all()` 被标记为 deprecated（已弃用），调用它不会有害但也没有作用。

**我们的兼容代码**：

```c
#if LIBAVFORMAT_VERSION_INT < AV_VERSION_INT(58, 9, 100)
av_register_all();  // 仅在旧版本调用
#endif
```

`AV_VERSION_INT(58, 9, 100)` 是 FFmpeg 4.0 对应的 libavformat 版本号。通过条件编译，我们在新旧版本上都能正确运行。

</details>

### 题目 2：为什么 `player_cleanup` 必须先调用 `packet_queue_abort` 再调用 `SDL_WaitThread`？

<details>
<summary>查看答案</summary>

**场景分析**：

解封装线程 `demux_thread` 可能在两个地方阻塞：
1. `packet_queue_get` 内部的 `SDL_CondWait`（等待消费者消费数据）— 虽然解封装线程是生产者不会调用 get，但音频回调线程在 `audio_decode_frame` 中会阻塞在 `packet_queue_get` 上
2. `av_read_frame`（等待网络数据到达）

如果直接调用 `SDL_WaitThread` 而不先 abort 队列：

```
错误顺序：
  主线程: SDL_WaitThread(demux_tid)  ← 等待解封装线程结束
  解封装线程: 正在运行...
  音频回调线程: packet_queue_get(&audio_queue) ← 阻塞等待
  
  → 解封装线程想退出，但先要等 quit=1 检查通过
  → 如果此时音频队列满了，解封装线程在 "队列过满" 处 SDL_Delay
  → 但音频回调线程阻塞在空队列上
  → 形成间接死锁
```

正确顺序：

```
正确顺序：
  1. ps->quit = 1                      ← 设置退出标志
  2. packet_queue_abort(&audio_queue)   ← 唤醒阻塞的线程
  3. packet_queue_abort(&video_queue)   ← 唤醒阻塞的线程
  4. SDL_WaitThread(demux_tid)          ← 安全等待
```

`packet_queue_abort` 内部将队列标记为"已中止"并调用 `SDL_CondBroadcast` 唤醒所有等待在该队列上的线程。被唤醒的线程检查到中止标志后会返回 -1，从而退出循环。

</details>

### 题目 3：修改 `event_loop`，添加左右方向键的快进/快退功能（每次 10 秒）

<details>
<summary>查看答案</summary>

```c
static void event_loop(PlayerState *ps) {
    SDL_Event event;
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
                ps->paused = !ps->paused;
                SDL_PauseAudio(ps->paused ? 1 : 0);
                break;

            case SDLK_LEFT: {
                // ── 快退 10 秒 ──
                double pos = ps->audio_clock - 10.0;
                if (pos < 0) pos = 0;

                // 将秒数转换为 AV_TIME_BASE 单位的时间戳
                // AV_TIME_BASE = 1000000（微秒），FFmpeg 内部使用
                int64_t ts = (int64_t)(pos * AV_TIME_BASE);

                // av_seek_frame 执行随机访问（seek）
                // 参数：
                //   fmt_ctx —— 格式上下文
                //   -1      —— 流索引（-1 = 默认流）
                //   ts      —— 目标时间戳（AV_TIME_BASE 单位）
                //   AVSEEK_FLAG_BACKWARD —— 向前查找最近的关键帧
                //                         （确保 seek 后能正确解码）
                av_seek_frame(ps->fmt_ctx, -1, ts, AVSEEK_FLAG_BACKWARD);

                // seek 后必须清空队列中的旧数据包
                // 否则解码器会继续处理 seek 前的数据
                packet_queue_flush(&ps->audio_queue);
                packet_queue_flush(&ps->video_queue);

                // 刷新解码器缓冲区
                // avcodec_flush_buffers 清空解码器内部的参考帧缓存
                // 如果不刷新，解码器会用旧的参考帧解码新位置的
                // P 帧/B 帧，导致花屏
                if (ps->audio_codec_ctx)
                    avcodec_flush_buffers(ps->audio_codec_ctx);
                if (ps->video_codec_ctx)
                    avcodec_flush_buffers(ps->video_codec_ctx);

                printf("快退到 %.1f 秒\n", pos);
                break;
            }

            case SDLK_RIGHT: {
                // ── 快进 10 秒 ──
                double pos = ps->audio_clock + 10.0;
                int64_t ts = (int64_t)(pos * AV_TIME_BASE);

                // 快进不需要 AVSEEK_FLAG_BACKWARD
                // FFmpeg 会自动找到最近的关键帧
                av_seek_frame(ps->fmt_ctx, -1, ts, 0);

                packet_queue_flush(&ps->audio_queue);
                packet_queue_flush(&ps->video_queue);
                if (ps->audio_codec_ctx)
                    avcodec_flush_buffers(ps->audio_codec_ctx);
                if (ps->video_codec_ctx)
                    avcodec_flush_buffers(ps->video_codec_ctx);

                printf("快进到 %.1f 秒\n", pos);
                break;
            }
            }
            break;

        case FF_REFRESH_EVENT:
            if (!ps->paused) {
                video_refresh(ps);
            }
            break;
        }
    }
}
```

**Seek 三步骤解释**：

1. **`av_seek_frame`**：在容器层面移动读取位置到指定时间点附近的关键帧
2. **`packet_queue_flush`**：清空队列中 seek 前的旧数据包（如果不清空，解码器会先把旧包解完才开始处理新位置的包）
3. **`avcodec_flush_buffers`**：清空解码器内部的参考帧缓存。视频编码中 P 帧（前向预测帧）和 B 帧（双向预测帧）依赖于之前的参考帧，seek 后这些参考帧已失效，必须清除

KrKr2 中 `BasePlayer::FlushBuffers()`（`VideoPlayer.cpp` 第 1279-1333 行）执行了更完善的清理操作，包括重置时钟、清空渲染缓冲区、发送 FLUSH 消息等。

</details>

---

## 下一步

下一节 [动手实践与总结](./04-动手实践与总结.md) 将提供更多进阶练习（进度条显示、字幕渲染等），并对整个第6章进行系统总结，建立与 KrKr2 源码的完整映射关系。
