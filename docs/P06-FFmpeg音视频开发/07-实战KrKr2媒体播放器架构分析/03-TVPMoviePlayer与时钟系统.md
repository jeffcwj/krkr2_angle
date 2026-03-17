# TVPMoviePlayer：KrKr2 的桥梁层
## TVPMoviePlayer：KrKr2 的桥梁层

`TVPMoviePlayer` 是 KrKr2 与 Kodi 播放器之间的桥梁，也是你修改播放器行为时最常接触的文件。

源码位置：`cpp/core/movie/ffmpeg/KRMoviePlayer.cpp`（382 行）

### 类继承关系

```cpp
// KRMoviePlayer.h 中的定义
class TVPMoviePlayer 
    : public iTVPVideoOverlay    // KrKr2 引擎接口
    , public CBaseRenderer       // Kodi 渲染器接口
{
    // KrKr2 侧
    iTVPVideoOverlay* m_pOverlay;   // 上层叠加层
    
    // Kodi 侧
    BasePlayer* m_pPlayer;           // 播放器实例
    CRenderManager m_renderManager;  // 渲染管理器
    
    // YUV 环形缓冲
    struct YUVBuffer {
        uint8_t* data[3];   // Y, U, V 三个平面
        int linesize[3];    // 每行字节数
        double pts;          // 显示时间戳
    };
    YUVBuffer m_buffers[MAX_BUFFER_COUNT];  // MAX_BUFFER_COUNT=4
    int m_writeIdx;
    int m_readIdx;
};
```

### AddVideoPicture：从解码器到环形缓冲

当视频解码线程解码出一帧后，通过 `AddVideoPicture()` 将 YUV 数据拷贝到环形缓冲区。

```cpp
// 源码：KRMoviePlayer.cpp 第 176-228 行
bool TVPMoviePlayer::AddVideoPicture(DVDVideoPicture& pic) {
    // 1. 仅接受 YUV420P 格式
    if (pic.format != RENDER_FMT_YUV420P) {
        return false;
    }
    
    // 2. 获取写入槽位
    int idx = m_writeIdx % MAX_BUFFER_COUNT;
    YUVBuffer& buf = m_buffers[idx];
    
    // 3. 分配对齐内存（首次或尺寸变化时）
    int ySize = pic.iWidth * pic.iHeight;
    int uvSize = (pic.iWidth / 2) * (pic.iHeight / 2);
    
    if (!buf.data[0]) {
        buf.data[0] = (uint8_t*)TJSAlignedAlloc(ySize, 32);
        buf.data[1] = (uint8_t*)TJSAlignedAlloc(uvSize, 32);
        buf.data[2] = (uint8_t*)TJSAlignedAlloc(uvSize, 32);
    }
    
    // 4. 逐行拷贝 YUV 数据
    // Y 平面
    for (int i = 0; i < pic.iHeight; i++) {
        memcpy(buf.data[0] + i * pic.iWidth,
               pic.data[0] + i * pic.iLineSize[0],
               pic.iWidth);
    }
    // U 平面
    for (int i = 0; i < pic.iHeight / 2; i++) {
        memcpy(buf.data[1] + i * (pic.iWidth / 2),
               pic.data[1] + i * pic.iLineSize[1],
               pic.iWidth / 2);
    }
    // V 平面
    for (int i = 0; i < pic.iHeight / 2; i++) {
        memcpy(buf.data[2] + i * (pic.iWidth / 2),
               pic.data[2] + i * pic.iLineSize[2],
               pic.iWidth / 2);
    }
    
    // 5. 记录 PTS
    buf.pts = pic.pts;
    buf.linesize[0] = pic.iWidth;
    buf.linesize[1] = pic.iWidth / 2;
    buf.linesize[2] = pic.iWidth / 2;
    
    // 6. 推进写指针
    m_writeIdx++;
    
    return true;
}
```

**为什么用 `TJSAlignedAlloc`？** 对齐内存（32字节对齐）有助于 SIMD 指令（SSE/NEON）
加速后续的 YUV→RGB 转换操作。

### PresentPicture：渲染到屏幕

`VideoPresentOverlay::PresentPicture()` 由 Cocos2d-x 的调度器在主线程调用，
负责将 YUV 缓冲区中的帧渲染到屏幕上。

```cpp
// 源码：KRMoviePlayer.cpp 第 239-295 行
void VideoPresentOverlay::PresentPicture() {
    // 这个函数在 Cocos2d-x 主线程中被调度器调用
    
    // 1. 获取当前播放时钟
    double clock = m_pPlayer->GetClock();
    
    // 2. 查找最合适的帧（PTS 最接近当前时钟的）
    int bestIdx = -1;
    double bestDiff = DBL_MAX;
    
    for (int i = m_readIdx; i < m_pPlayer->m_writeIdx; i++) {
        int idx = i % MAX_BUFFER_COUNT;
        double diff = m_pPlayer->m_buffers[idx].pts - clock;
        
        // 选择已经到显示时间且最接近当前时钟的帧
        if (diff <= 0 && fabs(diff) < bestDiff) {
            bestDiff = fabs(diff);
            bestIdx = idx;
        }
    }
    
    if (bestIdx < 0) return;  // 没有可显示的帧
    
    // 3. 跳过中间的帧（帧跳跃）
    // 如果积压了多帧，直接跳到最新的可显示帧
    m_readIdx = bestIdx + 1;
    
    // 4. 使用 TVPYUVSprite 渲染 YUV 数据
    YUVBuffer& buf = m_pPlayer->m_buffers[bestIdx];
    m_pYUVSprite->updateYUV(
        buf.data[0], buf.linesize[0],
        buf.data[1], buf.linesize[1],
        buf.data[2], buf.linesize[2],
        m_width, m_height
    );
}
```

**关键设计决策：**
- **PTS 比较选帧**而非简单的 FIFO，允许在延迟时跳过中间帧
- **YUVSprite** 使用 GPU shader 做 YUV→RGB 转换，CPU 零开销
- **环形缓冲区**解耦了解码线程和渲染线程的速率差异

## CDVDClock：时钟系统

时钟是音视频同步的基准。KrKr2 使用微秒精度的时钟系统。

源码位置：`cpp/core/movie/ffmpeg/Clock.cpp`（276 行）

### 关键常量

```cpp
// Clock.h 中定义的常量
#define DVD_TIME_BASE      1000000  // 微秒为单位（1秒 = 1000000）
#define DVD_NOPTS_VALUE    0xFFF0000000000000  // 无效 PTS 标记
#define DVD_PLAYSPEED_NORMAL 1000   // 正常速度 = 1000（支持 0.5x~2x）
```

### 时钟实现

```cpp
// Clock.cpp 核心方法

double CDVDClock::GetClock() {
    std::lock_guard<std::mutex> lock(m_critSection);
    
    // 当前时间 = 基准时间 + (系统时钟 - 上次更新时的系统时钟) × 播放速率
    int64_t now = CurrentHostCounter();
    double elapsed = (double)(now - m_startClock) / m_systemFrequency;
    
    return m_iDisc + elapsed * m_iSpeed / DVD_PLAYSPEED_NORMAL;
}

void CDVDClock::Discontinuity(double pts) {
    std::lock_guard<std::mutex> lock(m_critSection);
    
    // seek 后重置时钟基准
    m_startClock = CurrentHostCounter();
    m_iDisc = pts;  // 新的基准 PTS
}
```

**`DVD_NOPTS_VALUE` 的用法：** 值为 `0xFFF0000000000000`（一个极大的正数），
用于标记"无有效时间戳"。代码中大量使用：

```cpp
// 常见模式
if (pts == DVD_NOPTS_VALUE) {
    // 该数据包没有时间戳，用帧率推算
    pts = m_lastPts + frameDuration;
}
```

## 一帧视频的完整旅程

现在让我们追踪一帧视频从文件到屏幕的完整路径：

```
步骤 1: 文件 → AVPacket
━━━━━━━━━━━━━━━━━━━
CDVDDemuxFFmpeg::Read()
  └── av_read_frame(m_pFormatContext, &pkt)
      // 从容器中读取一个压缩数据包
      // 文件：DemuxFFmpeg.cpp

步骤 2: AVPacket → 消息队列
━━━━━━━━━━━━━━━━━━━
BasePlayer::ProcessPacket()
  └── CDVDMsgDemuxerPacket* msg = new CDVDMsgDemuxerPacket(pkt)
      m_VideoPlayerVideo->SendMessage(msg)
      // 数据包封装为消息，发送到视频线程
      // 文件：VideoPlayer.cpp

步骤 3: 消息队列 → 解码
━━━━━━━━━━━━━━━━━━━
CVideoPlayerVideo::Process()
  └── m_messageQueue.Get(&pMsg, 1000)
      m_pVideoCodec->Decode(pData, iSize, dts, pts)
      // 视频线程取出消息，调用 FFmpeg 解码
      // 文件：VideoPlayerVideo.cpp

步骤 4: FFmpeg 解码
━━━━━━━━━━━━━━━━━━━
CDVDVideoCodecFFmpeg::Decode()
  └── avcodec_decode_video2(m_pCodecContext, m_pFrame, &got, &pkt)
      // 注意：使用的是旧版 API（非 send_packet/receive_frame）
      // 文件：VideoCodecFFmpeg.cpp

步骤 5: 解码帧 → 时序控制
━━━━━━━━━━━━━━━━━━━
CVideoPlayerVideo::OutputPicture()
  ├── 计算 PTS 与主时钟偏差
  ├── CalcDropRequirement() → 判断是否丢帧
  ├── WaitForBuffer() → 等待渲染队列有空位
  └── AddVideoPicture() + FlipPage()
      // 文件：VideoPlayerVideo.cpp

步骤 6: 渲染队列 → KrKr2 环形缓冲
━━━━━━━━━━━━━━━━━━━
TVPMoviePlayer::AddVideoPicture()
  └── memcpy YUV420P 数据到 m_buffers[idx]
      // 仅接受 YUV420P 格式
      // 使用 TJSAlignedAlloc 32 字节对齐
      // 文件：KRMoviePlayer.cpp

步骤 7: 环形缓冲 → 屏幕
━━━━━━━━━━━━━━━━━━━
VideoPresentOverlay::PresentPicture()
  ├── 按 PTS 选取最合适的帧
  ├── 跳过已过期的中间帧
  └── m_pYUVSprite->updateYUV(Y, U, V)
      // GPU shader 完成 YUV→RGB 转换并显示
      // 由 Cocos2d-x 调度器在主线程调用
      // 文件：KRMoviePlayer.cpp
```

### 跨线程数据流总结

```
┌──────────┐    AVPacket    ┌───────────────┐    CDVDMsg    ┌─────────────────┐
│ DemuxFFmpeg │──────────────→│ BasePlayer    │──────────────→│ VideoPlayerVideo │
│ (主线程)   │              │ (主线程)      │              │ (视频线程)       │
└──────────┘              └───────────────┘              └────────┬────────┘
                                                                  │ DVDVideoPicture
                                                                  ▼
┌──────────────┐   YUV memcpy   ┌───────────────┐   PTS 选帧    ┌──────────────┐
│ YUVSprite    │←───────────────│ TVPMoviePlayer │←──────────────│ RenderManager │
│ (GPU 渲染)   │              │ (环形缓冲)    │              │ (帧调度)      │
└──────────────┘              └───────────────┘              └──────────────┘
```

