# Cocos2d 显示集成

> **所属模块：** M06-视频播放器
> **前置知识：** [02-CRenderManager渲染管线](./02-CRenderManager渲染管线.md)、[P10-Cocos2d-x框架](../../P10-Cocos2d-x框架/README.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 KrKr2 中视频画面如何最终显示到屏幕上（Overlay 模式 vs Layer 模式）
2. 掌握 TVPYUVSprite 的 YUV→RGB GPU 着色器工作原理
3. 理解 VideoPresentOverlay 基于 dt 的帧调度机制
4. 理解 VideoPresentLayer 基于 CDVDClock 的帧调度与双缓冲交换
5. 掌握事件驱动的视频更新流程（EC_UPDATE / EC_COMPLETE）

## 两种显示模式概述

KrKr2 继承了 KiriKiri2 原版的两种视频播放模式——**Overlay 模式**和 **Layer 模式**。这两种模式在入口函数 `krffmpeg.cpp` 中分别通过不同的工厂函数创建：

```cpp
// cpp/core/movie/ffmpeg/krffmpeg.cpp 第 65-83 行

// Overlay 模式 —— 在 Cocos2d 场景树上直接叠加 YUV 精灵
void GetVideoOverlayObject(tTJSNI_VideoOverlay *callbackwin, IStream *stream,
                           const tjs_char *streamname, const tjs_char *type,
                           uint64_t size, iTVPVideoOverlay **out) {
    *out = new KRMovie::MoviePlayerOverlay;  // 创建 Overlay 模式播放器
    if (*out)
        static_cast<KRMovie::MoviePlayerOverlay *>(*out)->BuildGraph(
            callbackwin, stream, streamname, type, size);
}

// Layer 模式 —— 将视频帧写入 tTVPBaseTexture，由引擎图层系统管理
void GetVideoLayerObject(tTJSNI_VideoOverlay *callbackwin, IStream *stream,
                         const tjs_char *streamname, const tjs_char *type,
                         uint64_t size, iTVPVideoOverlay **out) {
    *out = new KRMovie::MoviePlayerLayer;    // 创建 Layer 模式播放器
    if (*out)
        static_cast<KRMovie::MoviePlayerLayer *>(*out)->BuildGraph(
            callbackwin, stream, streamname, type, size);
}
```

两种模式的核心差异如下：

| 特性 | Overlay 模式 (VideoPresentOverlay) | Layer 模式 (VideoPresentLayer) |
|------|-----------------------------------|-------------------------------|
| **显示载体** | Cocos2d 场景节点 + TVPYUVSprite | tTVPBaseTexture 双缓冲 |
| **像素格式** | YUV420P（GPU 着色器转 RGB） | RGBA（CPU sws_scale 转换） |
| **调度方式** | Cocos2d schedule 回调（dt 驱动） | TVPAddContinuousEventHook（引擎事件驱动） |
| **帧跳过** | PresentPicture 内 do-while 循环 | OnContinuousCallback 中 PTS 判断 |
| **内存占用** | 1.5 bytes/pixel (YUV) | 4 bytes/pixel (RGBA) |
| **GPU 开销** | YUV→RGB 着色器 | 无（CPU 已转好） |
| **适用场景** | 全屏播放、OP/ED 动画 | 图层合成、半透明叠加 |

```
┌──────────────────────────────────────────────────────┐
│                 解码线程 (CVideoPlayerVideo)           │
│  OutputPicture() → AddVideoPicture(DVDVideoPicture)  │
└──────────────────────┬───────────────────────────────┘
                       │
           ┌───────────┴───────────┐
           ▼                       ▼
┌─────────────────────┐  ┌─────────────────────┐
│  TVPMoviePlayer::    │  │ VideoPresentLayer:: │
│  AddVideoPicture()   │  │ AddVideoPicture()   │
│  (YUV 直通复制)      │  │ (sws_scale→RGBA)    │
└──────────┬──────────┘  └──────────┬──────────┘
           │                        │
           ▼                        ▼
┌─────────────────────┐  ┌─────────────────────┐
│ BitmapPicture 环形   │  │ BitmapPicture 环形   │
│ 缓冲区 (YUV×3平面)   │  │ 缓冲区 (RGBA×1缓冲)  │
└──────────┬──────────┘  └──────────┬──────────┘
           │                        │
    Cocos2d schedule         TVPContinuousEvent
           │                        │
           ▼                        ▼
┌─────────────────────┐  ┌─────────────────────┐
│ PresentPicture(dt)  │  │ OnContinuousCallback│
│ → TVPYUVSprite      │  │ → GetFrontBuffer()  │
│ → GPU YUV→RGB       │  │ → tTVPBaseTexture   │
│ → 屏幕显示          │  │ → 图层合成显示       │
└─────────────────────┘  └─────────────────────┘
```

### 类继承体系

理解显示集成需要首先理清类的继承关系。KrKr2 的视频播放器类层次如下：

```
                    iTVPVideoOverlay          CBaseRenderer
                        │                        │
                        └──────────┬──────────────┘
                                   │
                           TVPMoviePlayer
                        (环形缓冲区、WaitForBuffer、
                         AddVideoPicture 基础实现)
                                   │
                    ┌──────────────┴──────────────┐
                    │                             │
           VideoPresentOverlay            VideoPresentLayer
        (Cocos2d 节点 + YUV精灵)     (tTVPBaseTexture 双缓冲)
        (PresentPicture(dt))        (OnContinuousCallback)
        (GetBounds() 纯虚)          (AddVideoPicture 覆写)
                    │                             │
         ┌──────────┴──────┐              MoviePlayerLayer
         │                 │           (BuildGraph, 事件回调)
  MoviePlayerOverlay  VideoPresentOverlay2
  (窗口关联模式)      (通用 Cocos2d 节点)
```

> **概念说明：iTVPVideoOverlay** 是 KiriKiri2 引擎定义的视频叠加层接口（Video Overlay Interface），包含 Play/Stop/Pause/SetPosition 等方法。CBaseRenderer 是从 Kodi/XBMC 继承来的渲染器基类，提供 AddVideoPicture/WaitForBuffer/Flush 等帧接收接口。TVPMoviePlayer 同时实现了这两个接口，成为连接 KiriKiri 脚本层与 Kodi 解码引擎的桥梁。

## TVPYUVSprite：GPU YUV→RGB 渲染

TVPYUVSprite 是 KrKr2 实现的一个自定义 Cocos2d 精灵（Custom Sprite），专门用于在 GPU 上将 YUV420P 视频帧转换为 RGB 并显示。它继承自 `cocos2d::Sprite`，通过自定义 GLSL 着色器（Shader）实现高效的色彩空间转换（Color Space Conversion）。

### 为什么需要 GPU 转换

将 YUV→RGB 转换放在 GPU 上执行有以下优势：

| 方案 | CPU 占用 | 内存带宽 | 延迟 |
|------|---------|---------|------|
| CPU sws_scale 转换 (Layer 模式) | 高（逐像素矩阵乘法） | 高（写 RGBA 4 bytes/pixel） | 高 |
| GPU 着色器转换 (Overlay 模式) | 低（仅上传纹理） | 低（上传 YUV 1.5 bytes/pixel） | 低 |

对于 1920×1080 的视频帧：
- **YUV420P 数据量**：1920×1080×1.5 = 3,110,400 字节（约 3 MB）
- **RGBA 数据量**：1920×1080×4 = 8,294,400 字节（约 8 MB）

GPU 方案减少了 60% 的 CPU→GPU 数据传输量。

### 三纹理架构

TVPYUVSprite 使用三个独立的 Cocos2d Texture2D 对象分别存储 Y、U、V 平面数据：

```cpp
// cpp/core/environ/cocos2d/YUVSprite.h 第 4-11 行

class TVPYUVSprite : public cocos2d::Sprite {
    // 继承自 Sprite 的 _texture 存储 Y 平面
    cocos2d::Texture2D *_textureU = nullptr;   // U (Cb) 色度平面
    cocos2d::Texture2D *_textureV = nullptr;   // V (Cr) 色度平面
    cocos2d::Texture2D::PixelFormat _texfmtY =
        cocos2d::Texture2D::PixelFormat::NONE; // Y 纹理格式
    cocos2d::Texture2D::PixelFormat _texfmtU =
        cocos2d::Texture2D::PixelFormat::NONE; // U 纹理格式
    cocos2d::Texture2D::PixelFormat _texfmtV =
        cocos2d::Texture2D::PixelFormat::NONE; // V 纹理格式
    // ...
};
```

> **YUV420P（也称 I420）** 是最常见的视频像素格式。"420" 表示色度（U/V）在水平和垂直方向上均以 2:1 的比例下采样——即每 4 个亮度（Y）像素共享 1 组色度（U/V）像素。因此 U 和 V 平面的宽高都是 Y 平面的一半。这是人眼对亮度敏感度远高于色度的生物学特性的直接应用。

初始化时，三个纹理均被创建并绑定到 GL 纹理单元（Texture Unit）0、1、2：

```cpp
// cpp/core/environ/cocos2d/YUVSprite.cpp 第 162-172 行

bool TVPYUVSprite::init() {
    initWithTexture(new cocos2d::Texture2D); // Y 平面纹理（继承自 Sprite）
    setupVBOAndVAO();                         // 可选的 VAO/VBO 设置
    _drawCommand.func = [this] { onDraw(); }; // 注册自定义绘制命令
    _textureU = new cocos2d::Texture2D;       // 创建 U 平面纹理
    _textureU->retain();                      // 防止自动释放
    _textureV = new cocos2d::Texture2D;       // 创建 V 平面纹理
    _textureV->retain();
    setGLProgram(_getYUVRenderProgram());      // 设置自定义 YUV 着色器
    return true;
}
```

### YUV→RGB 片段着色器

核心的色彩空间转换发生在 GLSL 片段着色器（Fragment Shader）中：

```glsl
// cpp/core/environ/cocos2d/YUVSprite.cpp 第 7-38 行

#ifdef GL_ES
precision lowp float;  // 移动端使用低精度浮点
#endif

varying vec4 v_fragmentColor;  // 顶点颜色（用于整体调色）
varying vec2 v_texCoord;       // 纹理坐标

void main() {
    vec3 yuv;
    // 从三个纹理单元分别采样 Y/U/V 值
    yuv.x = texture2D(CC_Texture0, v_texCoord).r - 0.0625;  // Y - 16/255
    yuv.y = texture2D(CC_Texture1, v_texCoord).r - 0.5;     // U - 128/255
    yuv.z = texture2D(CC_Texture2, v_texCoord).r - 0.5;     // V - 128/255

    // BT.601 标准的 YUV→RGB 转换矩阵
    vec3 rgb = mat3(
        1.164,  1.164, 1.164,    // Y 系数（三个通道相同）
        0.0,   -0.392, 2.017,    // U 系数 (Cb)
        1.596, -0.813, 0.0       // V 系数 (Cr)
    ) * yuv;

    gl_FragColor = vec4(rgb, 1.0) * v_fragmentColor;
}
```

这个着色器实现了 BT.601 标准（ITU-R BT.601，用于标清视频）的转换公式：

```
R = 1.164 × (Y - 16) + 1.596 × (V - 128)
G = 1.164 × (Y - 16) - 0.392 × (U - 128) - 0.813 × (V - 128)
B = 1.164 × (Y - 16) + 2.017 × (U - 128)
```

> **注意**：Y 减去 0.0625（= 16/255）是因为 BT.601 标准中 Y 的有效范围是 [16, 235]，U/V 的有效范围是 [16, 240]，中心值为 128（即 0.5 × 255）。这些偏移将有效范围映射到着色器中合适的浮点数域。

### 纹理数据更新

TVPYUVSprite 提供两种纹理更新方法——RGBA 直接更新和 YUV 分平面更新：

```cpp
// cpp/core/environ/cocos2d/YUVSprite.cpp 第 174-198 行

// 方法 1：RGBA 直接更新（Layer 模式使用，虽然 Overlay 模式不直接调用此版本）
void TVPYUVSprite::updateTextureData(const void *data, int width, int height) {
    cocos2d::Texture2D *pTex = getTexture();       // 获取 Y 纹理（此处作为 RGBA）
    const cocos2d::Size &size = pTex->getContentSize();
    updateTextureDataInternal(getTexture(), data, width, height,
                              cocos2d::Texture2D::PixelFormat::RGBA8888);
    if (size.width != width || size.height != height) {
        setTextureRect(cocos2d::Rect(0, 0, width, height)); // 更新精灵矩形
    }
}

// 方法 2：YUV 分平面更新（Overlay 模式使用）
void TVPYUVSprite::updateTextureData(
    const void *Y, int YW, int YH,    // Y 平面：全分辨率
    const void *U, int UW, int UH,    // U 平面：宽高各半
    const void *V, int VW, int VH) {  // V 平面：宽高各半
    cocos2d::Texture2D *pTex = getTexture();
    const cocos2d::Size &size = pTex->getContentSize();
    // 使用 I8（单通道 8 位）格式上传每个平面
    updateTextureDataInternal(getTexture(), Y, YW, YH,
                              cocos2d::Texture2D::PixelFormat::I8);
    updateTextureDataInternal(_textureU, U, UW, UH,
                              cocos2d::Texture2D::PixelFormat::I8);
    updateTextureDataInternal(_textureV, V, VW, VH,
                              cocos2d::Texture2D::PixelFormat::I8);
    if (size.width != YW || size.height != YH) {
        setTextureRect(cocos2d::Rect(0, 0, YW, YH));
    }
}
```

> **I8 像素格式**（也称 Luminance8 或 Alpha8）是单通道 8 位纹理格式，每个像素仅占 1 字节。用于存储 YUV 的单个平面非常合适，因为 Y/U/V 每个分量本身就是 8 位。在着色器中通过 `.r` 分量读取。

`updateTextureDataInternal` 处理了纹理尺寸变化时的重新初始化逻辑：

```cpp
// cpp/core/environ/cocos2d/YUVSprite.cpp 第 114-148 行

void TVPYUVSprite::updateTextureDataInternal(
    cocos2d::Texture2D *pTex, const void *data,
    int width, int height,
    cocos2d::Texture2D::PixelFormat pixfmt) {
    const cocos2d::Size &size = pTex->getContentSize();
    cocos2d::Size videoSize(width, height);
    // 检查是否需要重新创建纹理
    if (size.width != videoSize.width || size.height != videoSize.height ||
        pTex->getPixelFormat() != pixfmt) {
        if (size.width < videoSize.width || size.height < videoSize.height ||
            pTex->getPixelFormat() != pixfmt) {
            ssize_t datasize = width * height;
            switch (pixfmt) {
                case Texture2D::PixelFormat::RGBA8888:
                    datasize *= 4; break;
                case Texture2D::PixelFormat::RGB888:
                    datasize *= 3; break;
                case Texture2D::PixelFormat::RGB565:
                case Texture2D::PixelFormat::RGBA4444:
                    datasize *= 2; break;
                default: break; // I8 = 1 byte/pixel，不需要乘
            }
            // 纹理变大或格式变化 → 重建纹理
            pTex->initWithData(data, datasize, pixfmt, width, height,
                               cocos2d::Size::ZERO);
            pTex = nullptr; // 标记已重建，跳过下面的 updateWithData
        }
    }
    if (pTex) {
        // 纹理尺寸不变 → 仅更新数据（更快）
        pTex->updateWithData(data, 0, 0, width, height);
    }
}
```

### 自定义绘制命令

TVPYUVSprite 覆写了 Cocos2d 的 `draw()` 方法来注入自定义绘制命令：

```cpp
// cpp/core/environ/cocos2d/YUVSprite.cpp 第 259-264 行

void TVPYUVSprite::draw(cocos2d::Renderer *renderer,
                        const cocos2d::Mat4 &transform, uint32_t flags) {
    _mv = transform;                        // 保存模型-视图矩阵
    _drawCommand.init(_globalZOrder);        // 按 Z 顺序排序
    renderer->addCommand(&_drawCommand);     // 提交到渲染队列
}
```

实际的 GL 绘制在 `onDraw()` 中执行：

```cpp
// cpp/core/environ/cocos2d/YUVSprite.cpp 第 200-257 行（非 VAO 路径）

void TVPYUVSprite::onDraw() {
    // 绑定 Y/U/V 纹理到三个纹理单元
    cocos2d::GL::bindTexture2DN(0, _texture->getName());   // CC_Texture0 = Y
    cocos2d::GL::bindTexture2DN(1, _textureU->getName());  // CC_Texture1 = U
    cocos2d::GL::bindTexture2DN(2, _textureV->getName());  // CC_Texture2 = V
    // 设置混合模式
    cocos2d::GL::blendFunc(_blendFunc.src, _blendFunc.dst);
    // 应用着色器 uniform（包括 MVP 矩阵）
    _glProgramState->apply(_mv);
    // 启用顶点属性
    cocos2d::GL::enableVertexAttribs(
        cocos2d::GL::VERTEX_ATTRIB_FLAG_POS_COLOR_TEX);

    // 提取四个顶点的位置、颜色、纹理坐标
    cocos2d::Vec3 vertices[4] = { /* 四角顶点 */ };
    glVertexAttribPointer(GLProgram::VERTEX_ATTRIB_POSITION,
                          3, GL_FLOAT, GL_FALSE, 0, vertices);

    cocos2d::Color4B colors[4] = { /* 四角颜色 */ };
    glVertexAttribPointer(GLProgram::VERTEX_ATTRIB_COLOR,
                          4, GL_UNSIGNED_BYTE, GL_TRUE, 0, colors);

    cocos2d::Tex2F texCoords[4] = { /* 四角纹理坐标 */ };
    glVertexAttribPointer(GLProgram::VERTEX_ATTRIB_TEX_COORD,
                          2, GL_FLOAT, GL_FALSE, 0, texCoords);

    // 以三角形带方式绘制四边形
    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
}
```

> **GL_TRIANGLE_STRIP** 用 4 个顶点绘制 2 个三角形组成的矩形。顺序为左上→右上→左下→右下（或类似），第一个三角形由顶点 0-1-2 组成，第二个由顶点 1-2-3 组成。

### 常见错误与解决方案

**错误 1：视频画面绿屏/紫屏**

```
症状：视频播放但画面显示为全绿色或紫色条纹
原因：YUV 纹理绑定顺序错误，U/V 平面对调
```

检查 `onDraw()` 中的纹理绑定顺序：
```cpp
// 正确顺序：
cocos2d::GL::bindTexture2DN(0, _texture->getName());   // Y
cocos2d::GL::bindTexture2DN(1, _textureU->getName());  // U
cocos2d::GL::bindTexture2DN(2, _textureV->getName());  // V

// 错误示例（U/V 对调导致颜色反转）：
cocos2d::GL::bindTexture2DN(1, _textureV->getName());  // 错！
cocos2d::GL::bindTexture2DN(2, _textureU->getName());  // 错！
```

**错误 2：视频画面有水平撕裂线**

```
症状：画面中出现规律性的水平亮线或暗线
原因：lineSize（步长）与实际宽度不匹配
```

FFmpeg 解码器可能将每行数据对齐到 32 或 64 字节边界，导致 `iLineSize > iWidth`。AddVideoPicture 中的逐行复制逻辑正是为了处理这个问题：

```cpp
// KRMoviePlayer.cpp 第 196-207 行
if (yuvwidth[i] == pic.iLineSize[i]) {
    memcpy(yuvdata[i], pic.data[i], size);    // 步长 == 宽度：整块复制
} else {
    uint8_t *d = yuvdata[i], *s = pic.data[i];
    for (int y = 0; y < yuvheight[i]; ++y) {
        memcpy(d, s, yuvwidth[i]);            // 逐行复制有效像素
        d += yuvwidth[i];                     // 目标步进 = 宽度
        s += pic.iLineSize[i];                // 源步进 = lineSize
    }
}
```

**错误 3：GL 上下文丢失后黑屏**

Android 应用切到后台再恢复时，GL 上下文可能被销毁重建。TVPYUVSprite 通过监听 `EVENT_RENDERER_RECREATED` 事件来重新编译着色器：

```cpp
// YUVSprite.cpp 第 43-57 行
_backgroundListener = cocos2d::EventListenerCustom::create(
    EVENT_RENDERER_RECREATED, [](cocos2d::EventCustom *) {
        _program->reset();
        _program->initWithByteArrays(
            cocos2d::ccPositionTextureColor_vert,
            _yuvRenderProgram_fsh);
        _program->link();
        _program->updateUniforms();
    });
```

## VideoPresentOverlay：Cocos2d Overlay 模式

VideoPresentOverlay 是 Overlay 模式的核心类，负责将 BitmapPicture 环形缓冲区中的 YUV 帧取出并通过 TVPYUVSprite 显示到 Cocos2d 场景树上。它使用 Cocos2d 的 `schedule()` 机制，由引擎每帧回调 `PresentPicture(float dt)` 方法。

### 场景树挂载

当用户调用 `SetWindow()` 设置播放窗口时，Overlay 模式会在窗口的 Cocos2d 节点树下创建一个子节点，并注册定时调度器：

```cpp
// cpp/core/movie/ffmpeg/KRMoviePlayer.cpp 第 315-328 行

void MoviePlayerOverlay::SetWindow(tTJSNI_Window *window) {
    ClearNode();                          // 清理旧节点
    m_pOwnerWindow = window;
    // 获取窗口的主显示区域节点
    cocos2d::Node *parent = m_pOwnerWindow->GetForm()->GetPrimaryArea();
    // 创建一个空的容器节点并挂载到场景树
    parent->addChild((m_pRootNode = cocos2d::Node::create()));
    m_pRootNode->setContentSize(cocos2d::Size::ZERO);
    // 注册每帧回调 —— Cocos2d 每次渲染循环都会调用 lambda
    const static std::string sckey("update video");
    m_pRootNode->schedule(
        [this](float dt) {
            PresentPicture(dt);      // dt = 上一帧到当前帧的时间间隔（秒）
        },
        sckey);                      // 用字符串 key 标识调度器
}
```

> **Cocos2d schedule() 机制**：Cocos2d-x 的节点调度系统允许将回调函数绑定到渲染循环中。每帧（默认 60 FPS）会调用所有注册的调度回调，参数 `dt` 是距上一帧的时间间隔（单位：秒）。例如在 60 FPS 下，dt ≈ 0.01667 秒。这是游戏引擎中经典的"更新循环"（Update Loop）模式。

### PresentPicture 帧调度算法

`PresentPicture(float dt)` 是 Overlay 模式的核心调度函数。它基于累计时间来判断何时应该显示下一帧：

```cpp
// cpp/core/movie/ffmpeg/KRMoviePlayer.cpp 第 239-295 行

void VideoPresentOverlay::PresentPicture(float dt) {
    BitmapPicture pic;
    if (!m_usedPicture) {    // 环形缓冲区为空，无帧可显示
        return;
    } else {
        std::lock_guard<std::mutex> lk(m_mtxPicture);
        BitmapPicture &picbuf = m_picture[m_curPicture]; // 取队首帧
        // PTS 初始化逻辑
        if (m_curpts == 0.0) {
            m_curpts = picbuf.pts;  // 第一帧：将当前时间设为帧的 PTS
        } else {
            m_curpts += dt;         // 后续帧：累加 dt 推进播放时间
            if (picbuf.pts > m_curpts) { // 帧的显示时间还没到
                return;                  // 跳过本次回调
            }
        }
        // 帧跳过循环 —— 跳过所有已过期的帧
        do {
            pic.Clear();
            m_picture[m_curPicture].swap(pic);  // 取出当前帧
            // 环形缓冲区指针前进
            m_curPicture = (m_curPicture + 1) & (MAX_BUFFER_COUNT - 1);
            --m_usedPicture;
        } while (m_usedPicture > 0 && m_curpts >= m_picture[m_curPicture].pts);
        assert(m_usedPicture >= 0);
        m_condPicture.notify_all();  // 通知生产者（解码线程）有空位了
    }
    FrameMove();                     // 通知 BasePlayer 推进内部状态
    if (!pic.rgba) {                 // 安全检查
        return;
    }
    // 处理可见性
    if (!Visible) {
        m_pRootNode->setVisible(false);
        return;
    } else {
        m_pRootNode->setVisible(true);
    }
    // 首帧时创建 TVPYUVSprite
    if (!m_pSprite) {
        m_pSprite = TVPYUVSprite::create();
        m_pSprite->setAnchorPoint(cocos2d::Vec2(0, 1)); // 左上角锚点
        m_pRootNode->addChild(m_pSprite);
    }
    // 更新 YUV 纹理数据
    cocos2d::Size videoSize(pic.width, pic.height);
    m_pSprite->updateTextureData(
        pic.data[0], pic.width, pic.height,           // Y 平面
        pic.data[1], pic.width / 2, pic.height / 2,   // U 平面（宽高各半）
        pic.data[2], pic.width / 2, pic.height / 2);  // V 平面
    // 缩放到目标区域
    const tTVPRect &rc = GetBounds();
    float scaleX = rc.get_width() / videoSize.width;
    float scaleY = rc.get_height() / videoSize.height;
    if (scaleX != m_pSprite->getScaleX())
        m_pSprite->setScaleX(scaleX);
    if (scaleY != m_pSprite->getScaleY())
        m_pSprite->setScaleY(scaleY);
    // 定位精灵
    cocos2d::Vec2 pos = m_pSprite->getPosition();
    int top = m_pRootNode->getParent()->getContentSize().height - rc.top;
    if ((int)pos.x != rc.left || (int)pos.y != top) {
        m_pSprite->setPosition(rc.left, top);
    }
}
```

PresentPicture 的帧调度时序图：

```
时间线 →
        0ms    16ms   33ms   50ms   66ms   83ms   100ms
dt:     ─      16ms   17ms   17ms   16ms   17ms   17ms
m_curpts: 0.0   0.016  0.033  0.050  0.066  0.083  0.100

帧 PTS:  0.000         0.033         0.067         0.100
         │             │             │             │
         ▼             ▼             ▼             ▼
回调1:  显示帧0   ─  回调3:显示帧1  ─  回调5:显示帧2  ─  回调7:显示帧3
回调2:  跳过(未到)   回调4:跳过(未到)   回调6:跳过(未到)

(假设视频为 30fps，帧间隔 33.3ms；显示为 60fps，回调间隔 ~16.7ms)
```

> **关键理解**：`m_curpts` 是基于 dt 累加的"软时钟"，而非来自 CDVDClock 的"硬时钟"。这意味着 Overlay 模式的帧同步精度取决于 Cocos2d 渲染循环的稳定性。如果渲染帧率不稳定（如 GC 卡顿），dt 可能出现跳变，导致连续跳帧。

### 帧跳过机制详解

do-while 循环是 PresentPicture 中最精妙的部分。当系统卡顿导致 `m_curpts` 大幅前进时，可能一次跳过多个已过期的帧：

```cpp
do {
    pic.Clear();
    m_picture[m_curPicture].swap(pic);  // 取出帧（swap 而非 copy）
    m_curPicture = (m_curPicture + 1) & (MAX_BUFFER_COUNT - 1);
    --m_usedPicture;
} while (m_usedPicture > 0 && m_curpts >= m_picture[m_curPicture].pts);
```

```
场景：系统卡顿 100ms，缓冲区有 3 帧

Before:
  m_curpts = 2.500  （已累加 100ms 的 dt）
  帧 [0]: pts=2.400  ← m_curPicture 指向这里
  帧 [1]: pts=2.433
  帧 [2]: pts=2.467

迭代 1: swap 帧[0] (pts=2.400), m_curPicture→1
         检查: 2.500 >= 2.433 → true，继续
迭代 2: swap 帧[1] (pts=2.433), m_curPicture→2
         检查: 2.500 >= 2.467 → true，继续
迭代 3: swap 帧[2] (pts=2.467), m_curPicture→3
         检查: m_usedPicture=0 → false，退出

After: pic 持有帧[2]，帧[0]和[1]被丢弃
       屏幕显示最新的帧[2]，跳过了帧[0]和[1]
```

> **swap 而非 copy**：`BitmapPicture::swap()` 仅交换指针和元数据，不复制像素数据。这确保了每次迭代中 `pic.Clear()` 能正确释放上一次迭代取出的帧内存，而最后一次迭代保留的帧被送去显示。

## VideoPresentLayer：Layer 模式

Layer 模式与 Overlay 模式在帧调度和显示方式上有根本区别。Layer 模式将视频帧写入 KiriKiri 引擎的 `tTVPBaseTexture` 双缓冲（Double Buffer），通过引擎的图层系统（Layer System）参与正常的图层合成渲染。

### AddVideoPicture 的 YUV→RGBA 转换

Layer 模式覆写了 `AddVideoPicture()` 方法，在解码线程中用 CPU（而非 GPU）将 YUV420P 转换为 RGBA：

```cpp
// cpp/core/movie/ffmpeg/KRMovieLayer.cpp 第 69-109 行

int VideoPresentLayer::AddVideoPicture(DVDVideoPicture &pic, int index) {
    if (pic.format != RENDER_FMT_YUV420P)
        return -2;               // 仅接受 YUV420P
    if (pic.pts == DVD_NOPTS_VALUE)
        return 0;                // 无效 PTS，跳过

    // 等待环形缓冲区有空位
    if (m_usedPicture >= MAX_BUFFER_COUNT) {
        std::unique_lock<std::mutex> lk(m_mtxPicture);
        m_condPicture.wait(lk);  // 阻塞等待消费者取走帧
    }
    if (m_usedPicture >= MAX_BUFFER_COUNT)
        return -1;               // 仍然满，放弃

    int width = pic.iWidth, height = pic.iHeight;
    // 分配 RGBA 缓冲区
    uint8_t *data = (uint8_t *)TJSAlignedAlloc(width * height * 4, 4);
    int datasize = width * 4;    // RGBA 行步长

    // 获取或创建 sws_scale 上下文
    img_convert_ctx = sws_getCachedContext(
        img_convert_ctx,
        width, height, AV_PIX_FMT_YUV420P,     // 源格式
        width, height, AV_PIX_FMT_RGBA,          // 目标格式
        SWS_FAST_BILINEAR,                        // 缩放算法（这里不缩放）
        nullptr, nullptr, nullptr);
    assert(img_convert_ctx);

    // 执行色彩空间转换
    int processed = sws_scale(
        img_convert_ctx,
        pic.data, pic.iLineSize, 0, pic.iHeight, // 源 YUV 数据
        &data, &datasize);                         // 目标 RGBA 数据

    // 存入环形缓冲区
    {
        std::lock_guard<std::mutex> lk(m_mtxPicture);
        BitmapPicture &picbuf =
            m_picture[(m_curPicture + m_usedPicture) & (MAX_BUFFER_COUNT - 1)];
        picbuf.Clear();
        picbuf.width = width;
        picbuf.height = height;
        picbuf.data[0] = data;              // 仅存一个 RGBA 缓冲
        picbuf.pts = pic.pts / DVD_TIME_BASE;
        ++m_usedPicture;
    }
    return MAX_BUFFER_COUNT - m_usedPicture;
}
```

> **SWS_FAST_BILINEAR** 是 libswscale 最快的双线性缩放算法标志。虽然这里源和目标尺寸相同（不需要缩放），但 sws_scale 仍然可以高效地执行纯色彩空间转换。其他选项如 SWS_BICUBIC 质量更高但速度更慢，对于实时视频播放不推荐。

### GetFrontBuffer 双缓冲交换

Layer 模式通过 `GetFrontBuffer()` 将 RGBA 帧数据写入 tTVPBaseTexture，并使用双缓冲避免撕裂：

```cpp
// cpp/core/movie/ffmpeg/KRMovieLayer.cpp 第 15-35 行

tTVPBaseTexture *VideoPresentLayer::GetFrontBuffer() {
    BitmapPicture pic;
    if (!m_usedPicture) {        // 无帧可用
        return nullptr;
    }
    {
        std::lock_guard<std::mutex> lk(m_mtxPicture);
        BitmapPicture &picbuf = m_picture[m_curPicture]; // 取队首
        picbuf.swap(pic);                                 // 零拷贝交换
        m_curPicture = (m_curPicture + 1) & (MAX_BUFFER_COUNT - 1);
        --m_usedPicture;
        assert(m_usedPicture >= 0);
        m_condPicture.notify_all(); // 唤醒生产者
    }
    FrameMove();                    // 通知 BasePlayer
    int n = m_nCurBmpBuff;          // 当前写入缓冲索引
    m_nCurBmpBuff = !m_nCurBmpBuff; // 切换到另一个缓冲
    // 将 RGBA 数据写入纹理
    m_BmpBits[n]->Update(
        pic.data[0],     // RGBA 像素数据
        pic.width * 4,   // 行步长（pitch = width × 4 bytes）
        0, 0,            // 写入起始位置 (x, y)
        pic.width,       // 写入宽度
        pic.height);     // 写入高度
    return m_BmpBits[n]; // 返回给引擎图层系统
}
```

双缓冲工作原理：

```
时刻 T1:
  m_nCurBmpBuff = 0
  m_BmpBits[0] ← 正在被 GPU 读取（上一帧）
  m_BmpBits[1] ← 空闲

  GetFrontBuffer():
    n = 0
    m_nCurBmpBuff = 1  （切换）
    写入 m_BmpBits[0]  ← 新帧数据
    返回 m_BmpBits[0]

时刻 T2:
  m_nCurBmpBuff = 1
  m_BmpBits[0] ← 正在被 GPU 读取（当前帧）
  m_BmpBits[1] ← 空闲

  GetFrontBuffer():
    n = 1
    m_nCurBmpBuff = 0  （切换回）
    写入 m_BmpBits[1]  ← 新帧数据
    返回 m_BmpBits[1]
```

> **注意**：这里的"双缓冲"与通常的前后缓冲（Front/Back Buffer）稍有不同。`m_BmpBits[0]` 和 `m_BmpBits[1]` 是两个 tTVPBaseTexture 对象，交替使用以确保写入操作不会干扰正在被引擎读取的纹理。`SetVideoBuffer()` 在播放开始前由引擎调用来设置这两个缓冲。

### OnContinuousCallback 事件驱动调度

Layer 模式不使用 Cocos2d 的 schedule 机制，而是通过 KiriKiri 引擎的连续事件钩子（Continuous Event Hook）驱动帧更新：

```cpp
// cpp/core/movie/ffmpeg/KRMovieLayer.cpp 第 45-67 行

void VideoPresentLayer::OnContinuousCallback(tjs_uint64 tick) {
    if (!m_usedPicture)          // 无帧可用
        return;
    // 使用 CDVDClock 获取当前播放时间（比 dt 累加更精确）
    double m_curpts = m_pPlayer->GetClock() / DVD_TIME_BASE;
    {
        std::lock_guard<std::mutex> lk(m_mtxPicture);
        BitmapPicture &picbuf = m_picture[m_curPicture];
        if (picbuf.pts > m_curpts) { // 帧显示时间还没到
            return;
        }
    }
    // 时间已到 → 触发更新事件
    OnPlayEvent(KRMovieEvent::Update, nullptr);
}
```

与 Overlay 模式的关键区别：

| 特性 | Overlay (PresentPicture) | Layer (OnContinuousCallback) |
|------|-------------------------|------------------------------|
| **时间源** | dt 累加（"软时钟"） | CDVDClock（"硬时钟"） |
| **帧消费** | 直接在回调中取帧并渲染 | 仅触发事件，由引擎调用 GetFrontBuffer |
| **帧跳过** | do-while 循环跳过过期帧 | 被注释掉（`#if 0`），依赖引擎节奏 |
| **线程模型** | Cocos2d 主线程 | KiriKiri 连续事件线程 |

> **为什么 Layer 模式禁用了帧跳过？** 观察源码中 `OnContinuousCallback` 的 `#if 0` 块——帧跳过逻辑被注释掉了。这是因为 Layer 模式通过事件驱动 `GetFrontBuffer()` 调用，引擎会在合适的时机（图层合成前）调用该方法取帧。如果 OnContinuousCallback 自己跳帧，可能导致 GetFrontBuffer 取到"跳过后"的帧，与引擎的期望不符。

### Play 与事件注册

Layer 模式在 `Play()` 时注册连续事件钩子，在析构时移除：

```cpp
// cpp/core/movie/ffmpeg/KRMovieLayer.cpp 第 138-141 行

void MoviePlayerLayer::Play() {
    inherit::Play();                  // 调用父类 Play() → m_pPlayer->Play()
    TVPAddContinuousEventHook(this);  // 注册到引擎事件系统
}

// 析构时自动移除
VideoPresentLayer::~VideoPresentLayer() {
    TVPRemoveContinuousEventHook(this); // 第 13 行
}
```

> **TVPAddContinuousEventHook** 是 KiriKiri 引擎提供的连续事件注册函数。它将实现了 `tTVPContinuousEventCallbackIntf` 接口的对象注册到引擎的事件循环中，引擎会周期性（每帧）调用 `OnContinuousCallback(tick)`。这类似于 Cocos2d 的 schedule 但运行在 KiriKiri 的事件系统中。

## 事件流：从视频帧到脚本层

视频播放的事件传播链涉及多个层次，最终将播放状态通知到 TJS2 脚本层：

### 事件类型

KrKr2 定义了两个主要的视频播放事件：

```
EC_UPDATE   —— 有新帧可显示（仅 Layer 模式使用）
EC_COMPLETE —— 播放结束（两种模式都使用）
```

### Layer 模式事件流

```
解码线程                    KiriKiri 引擎事件循环              TJS2 脚本层
    │                              │                              │
    │  AddVideoPicture()           │                              │
    │──────────────────►          │                              │
    │  (写入环形缓冲区)            │                              │
    │                              │                              │
    │                    OnContinuousCallback()                   │
    │                    PTS 检查通过                              │
    │                              │                              │
    │                    OnPlayEvent(Update)                      │
    │                              │                              │
    │                    NativeEvent(WM_GRAPHNOTIFY)              │
    │                    ev.WParam = EC_UPDATE                    │
    │                    ev.LParam = frame                        │
    │                              │                              │
    │                    m_pCallbackWin->WndProc(ev)              │
    │                              │──────────────►               │
    │                              │         onPeriod 回调         │
    │                              │         GetFrontBuffer()     │
    │                              │◄──────────────               │
    │                              │         图层更新显示          │
```

```cpp
// cpp/core/movie/ffmpeg/KRMovieLayer.cpp 第 122-136 行

void MoviePlayerLayer::OnPlayEvent(KRMovieEvent msg, void *p) {
    if (msg == KRMovieEvent::Update) {
        NativeEvent ev(WM_GRAPHNOTIFY);
        ev.WParam = EC_UPDATE;            // 帧更新事件
        int frame;
        GetFrame(&frame);                 // 获取当前帧号
        ev.LParam = frame;
        m_pCallbackWin->WndProc(ev);      // 同步调用（同线程）
    } else if (msg == KRMovieEvent::Ended) {
        NativeEvent ev(WM_GRAPHNOTIFY);
        ev.WParam = EC_COMPLETE;          // 播放结束事件
        ev.LParam = 0;
        m_pCallbackWin->PostEvent(ev);    // 异步投递（跨线程安全）
    }
}
```

> **WndProc vs PostEvent**：注意 EC_UPDATE 使用 `WndProc()`（同步调用），而 EC_COMPLETE 使用 `PostEvent()`（异步投递）。这是因为 EC_UPDATE 在连续事件回调中触发，处于安全的引擎线程上下文中；而 EC_COMPLETE 可能从 BasePlayer 线程触发，需要异步投递到主线程避免竞态条件。

### Overlay 模式事件流

Overlay 模式更简单，因为帧显示直接在 PresentPicture 中完成，不需要 EC_UPDATE 事件：

```cpp
// cpp/core/movie/ffmpeg/KRMoviePlayer.cpp 第 352-359 行

void MoviePlayerOverlay::OnPlayEvent(KRMovieEvent msg, void *p) {
    if (msg == KRMovieEvent::Ended) {
        NativeEvent ev(WM_GRAPHNOTIFY);
        ev.WParam = EC_COMPLETE;       // 仅处理播放结束
        ev.LParam = 0;
        m_pCallbackWin->PostEvent(ev); // 异步投递
    }
}
```

### 跨平台注意事项

视频显示集成涉及 GL 调用，在不同平台上有以下差异：

| 平台 | GL API | 着色器精度 | GL 上下文恢复 |
|------|--------|-----------|--------------|
| **Windows** | OpenGL 3.3+ | highp（默认） | 无需处理 |
| **Linux** | OpenGL 3.3+ | highp（默认） | 无需处理（X11/Wayland） |
| **macOS** | OpenGL 2.1 (deprecated) | highp（默认） | 无需处理 |
| **Android** | OpenGL ES 2.0/3.0 | lowp/mediump | 必须处理 EVENT_RENDERER_RECREATED |

Android 特有问题：
1. **精度限制**：着色器中的 `precision lowp float` 在某些 GPU 上可能导致颜色精度不够，出现色带（Banding）
2. **GL 上下文丢失**：应用切后台时 GL 上下文可能被销毁，需要重新编译着色器
3. **EGL Surface**：某些设备在屏幕旋转时会重建 EGL Surface

## 动手实践

### 练习 1：实现简化版 YUV→RGB 转换器

创建一个独立的 YUV→RGB CPU 转换函数，理解色彩空间转换的原理：

```cpp
// yuv_converter.cpp
#include <cstdint>
#include <cstdio>
#include <cmath>
#include <vector>

// BT.601 YUV→RGB 转换（与 TVPYUVSprite 着色器相同的数学公式）
struct RGBAPixel {
    uint8_t r, g, b, a;
};

// 将单个 YUV 值转换为 RGBA
// Y: [16, 235], U/V: [16, 240]
RGBAPixel yuv_to_rgba(uint8_t y, uint8_t u, uint8_t v) {
    // 与着色器公式等价的整数定点运算版本
    int C = (int)y - 16;    // Y 偏移
    int D = (int)u - 128;   // U 偏移（中心化）
    int E = (int)v - 128;   // V 偏移（中心化）

    // BT.601 转换系数（定点运算，右移 8 位 = 除以 256）
    int r = (298 * C + 409 * E + 128) >> 8;
    int g = (298 * C - 100 * D - 208 * E + 128) >> 8;
    int b = (298 * C + 516 * D + 128) >> 8;

    // 钳位到 [0, 255]
    auto clamp = [](int val) -> uint8_t {
        return val < 0 ? 0 : (val > 255 ? 255 : val);
    };
    return { clamp(r), clamp(g), clamp(b), 255 };
}

// 将 YUV420P 图像转换为 RGBA
// Y 平面: width × height
// U 平面: width/2 × height/2
// V 平面: width/2 × height/2
std::vector<RGBAPixel> convert_yuv420p_to_rgba(
    const uint8_t *yPlane, int yStride,
    const uint8_t *uPlane, int uStride,
    const uint8_t *vPlane, int vStride,
    int width, int height) {

    std::vector<RGBAPixel> rgba(width * height);

    for (int row = 0; row < height; ++row) {
        for (int col = 0; col < width; ++col) {
            // Y 平面：每个像素一个值
            uint8_t y = yPlane[row * yStride + col];
            // U/V 平面：每 2×2 像素共享一个值
            uint8_t u = uPlane[(row / 2) * uStride + (col / 2)];
            uint8_t v = vPlane[(row / 2) * vStride + (col / 2)];

            rgba[row * width + col] = yuv_to_rgba(y, u, v);
        }
    }
    return rgba;
}

int main() {
    // 创建 4×4 的测试 YUV 图像
    const int W = 4, H = 4;
    // Y 平面 (4×4) —— 从暗到亮的渐变
    uint8_t yData[W * H] = {
        16,  76, 135, 195,    // 第 0 行
        16,  76, 135, 195,    // 第 1 行
        135, 195, 235, 235,   // 第 2 行
        135, 195, 235, 235    // 第 3 行
    };
    // U 平面 (2×2)
    uint8_t uData[2 * 2] = {
        128, 128,   // 中性色度
        128, 128
    };
    // V 平面 (2×2)
    uint8_t vData[2 * 2] = {
        128, 128,   // 中性色度
        128, 128
    };

    auto result = convert_yuv420p_to_rgba(
        yData, W, uData, 2, vData, 2, W, H);

    // 打印转换结果
    printf("YUV420P → RGBA 转换结果 (%d×%d):\n", W, H);
    for (int row = 0; row < H; ++row) {
        for (int col = 0; col < W; ++col) {
            auto &p = result[row * W + col];
            printf("(%3d,%3d,%3d) ", p.r, p.g, p.b);
        }
        printf("\n");
    }
    return 0;
}
```

编译和运行：

```bash
# Windows
cl /EHsc /std:c++17 yuv_converter.cpp /Fe:yuv_converter.exe
yuv_converter.exe

# Linux / macOS
g++ -std=c++17 -o yuv_converter yuv_converter.cpp
./yuv_converter

# 预期输出（中性色度下 Y→灰度）：
# YUV420P → RGBA 转换结果 (4×4):
# (  0,  0,  0) ( 70, 70, 70) (138,138,138) (208,208,208)
# ...
```

### 练习 2：模拟环形缓冲区的生产者-消费者模式

```cpp
// ring_buffer_sim.cpp
#include <cstdio>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <chrono>
#include <atomic>

#define MAX_BUFFER_COUNT 4

struct Frame {
    int id = -1;
    double pts = 0.0;
    bool valid = false;

    void Clear() { id = -1; pts = 0.0; valid = false; }
};

// 模拟 KrKr2 的 BitmapPicture 环形缓冲区
class RingBuffer {
    Frame m_frames[MAX_BUFFER_COUNT];
    int m_curFrame = 0;   // 消费者读取位置
    int m_usedFrame = 0;  // 已用帧数
    std::mutex m_mtx;
    std::condition_variable m_cond;

public:
    // 生产者：添加帧（模拟 AddVideoPicture）
    bool produce(int id, double pts) {
        if (m_usedFrame >= MAX_BUFFER_COUNT) {
            std::unique_lock<std::mutex> lk(m_mtx);
            m_cond.wait_for(lk, std::chrono::milliseconds(100));
        }
        if (m_usedFrame >= MAX_BUFFER_COUNT) {
            printf("[生产者] 缓冲区满，丢弃帧 %d\n", id);
            return false;
        }
        {
            std::lock_guard<std::mutex> lk(m_mtx);
            int idx = (m_curFrame + m_usedFrame) & (MAX_BUFFER_COUNT - 1);
            m_frames[idx] = { id, pts, true };
            ++m_usedFrame;
            printf("[生产者] 添加帧 %d (pts=%.3f), 缓冲区使用 %d/%d\n",
                   id, pts, m_usedFrame, MAX_BUFFER_COUNT);
        }
        return true;
    }

    // 消费者：取出帧（模拟 PresentPicture 的 swap 逻辑）
    Frame consume(double curpts) {
        Frame result;
        if (m_usedFrame == 0) return result;
        std::lock_guard<std::mutex> lk(m_mtx);
        Frame &front = m_frames[m_curFrame];
        if (front.pts > curpts) return result; // 还没到显示时间
        // do-while 跳帧逻辑
        do {
            result = m_frames[m_curFrame];
            m_frames[m_curFrame].Clear();
            m_curFrame = (m_curFrame + 1) & (MAX_BUFFER_COUNT - 1);
            --m_usedFrame;
        } while (m_usedFrame > 0 && curpts >= m_frames[m_curFrame].pts);
        m_cond.notify_all();
        return result;
    }
};

int main() {
    RingBuffer buf;
    std::atomic<bool> done{false};

    // 生产者线程（模拟解码线程，30fps）
    std::thread producer([&] {
        for (int i = 0; i < 20; ++i) {
            double pts = i * (1.0 / 30.0);   // 30fps
            buf.produce(i, pts);
            std::this_thread::sleep_for(std::chrono::milliseconds(33));
        }
        done = true;
    });

    // 消费者线程（模拟渲染线程，60fps）
    std::thread consumer([&] {
        double curpts = 0.0;
        while (!done || curpts < 0.65) {
            Frame f = buf.consume(curpts);
            if (f.valid) {
                printf("[消费者] 显示帧 %d (pts=%.3f, curpts=%.3f)\n",
                       f.id, f.pts, curpts);
            }
            std::this_thread::sleep_for(std::chrono::milliseconds(16));
            curpts += 0.016;  // 模拟 dt 累加
        }
    });

    producer.join();
    consumer.join();
    return 0;
}
```

## 对照项目源码

### 关键文件清单

| 文件路径 | 行号 | 内容 |
|----------|------|------|
| `cpp/core/environ/cocos2d/YUVSprite.h` | 1-42 | TVPYUVSprite 类定义，三纹理架构 |
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 7-38 | YUV→RGB GLSL 片段着色器 |
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 162-172 | init() 纹理创建与着色器绑定 |
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 184-198 | updateTextureData() YUV 分平面更新 |
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 200-257 | onDraw() 自定义 GL 绘制 |
| `cpp/core/movie/ffmpeg/KRMoviePlayer.h` | 225-245 | VideoPresentOverlay 类定义 |
| `cpp/core/movie/ffmpeg/KRMoviePlayer.cpp` | 239-295 | PresentPicture() 帧调度与显示 |
| `cpp/core/movie/ffmpeg/KRMoviePlayer.cpp` | 315-328 | SetWindow() 场景树挂载 |
| `cpp/core/movie/ffmpeg/KRMovieLayer.h` | 8-27 | VideoPresentLayer 类定义 |
| `cpp/core/movie/ffmpeg/KRMovieLayer.cpp` | 15-35 | GetFrontBuffer() 双缓冲交换 |
| `cpp/core/movie/ffmpeg/KRMovieLayer.cpp` | 45-67 | OnContinuousCallback() 事件驱动调度 |
| `cpp/core/movie/ffmpeg/KRMovieLayer.cpp` | 69-109 | AddVideoPicture() YUV→RGBA 转换 |
| `cpp/core/movie/ffmpeg/KRMovieLayer.cpp` | 122-136 | OnPlayEvent() 事件分发 |
| `cpp/core/movie/ffmpeg/krffmpeg.cpp` | 65-83 | GetVideoOverlayObject/GetVideoLayerObject 工厂函数 |

### 阅读建议

1. **先读 KRMoviePlayer.h**（第 193-223 行）理解 BitmapPicture 和环形缓冲区
2. **再读 KRMoviePlayer.cpp**（第 176-228 行）理解 AddVideoPicture 的 YUV 数据复制
3. **然后读 PresentPicture**（第 239-295 行）理解 Overlay 模式的帧调度
4. **对比读 KRMovieLayer.cpp**（全文 143 行）理解 Layer 模式的差异
5. **最后读 YUVSprite.cpp** 理解 GPU 渲染管线

## 本节小结

- KrKr2 提供两种视频显示模式：**Overlay**（Cocos2d YUV 精灵）和 **Layer**（tTVPBaseTexture 双缓冲）
- **TVPYUVSprite** 使用三个 I8 纹理 + BT.601 GLSL 着色器在 GPU 上完成 YUV→RGB 转换，减少 60% 数据传输
- **Overlay 模式**通过 Cocos2d schedule 的 dt 参数驱动帧调度，使用 do-while 循环跳过过期帧
- **Layer 模式**通过 KiriKiri 连续事件钩子和 CDVDClock 驱动帧调度，使用双缓冲避免撕裂
- 事件传播链：AddVideoPicture → BitmapPicture 缓冲 → PresentPicture/OnContinuousCallback → OnPlayEvent → NativeEvent → 脚本层
- BitmapPicture 使用 swap 而非 copy，实现零拷贝帧传递
- Android 平台需额外处理 GL 上下文丢失和着色器重建

## 练习题与答案

### 题目 1：为什么 Overlay 模式使用 dt 累加作为时间源而不是 CDVDClock？

<details>
<summary>查看答案</summary>

Overlay 模式的帧显示直接在 Cocos2d 的渲染循环中完成（`PresentPicture` 由 `schedule` 调度）。使用 dt 累加有以下原因：

1. **简单性**：dt 直接来自 Cocos2d 引擎，不需要额外的时钟查询开销
2. **自然同步**：dt 反映了实际的显示间隔，帧调度自然与显示刷新率对齐
3. **解耦**：Overlay 模式不参与 KiriKiri 引擎的图层合成，不需要与引擎的渲染节奏同步

而 Layer 模式使用 CDVDClock 是因为它需要与引擎的图层合成系统协调——引擎通过 EC_UPDATE 事件知道何时有新帧，然后在合成时调用 `GetFrontBuffer()`。CDVDClock 提供与音频同步的精确时间基准，确保 Layer 模式的帧显示时机与音视频同步机制一致。

两种模式的取舍：
- dt 累加：简单但精度受渲染帧率波动影响
- CDVDClock：精确但需要额外的时钟查询

</details>

### 题目 2：分析以下场景中 PresentPicture 的行为

假设环形缓冲区中有 4 帧（已满），PTS 分别为 1.000、1.033、1.067、1.100。当前 `m_curpts = 0.0`，此时调用 PresentPicture(dt=0)。接下来连续两次调用 PresentPicture(dt=1.050) 和 PresentPicture(dt=0.020)，分析每次调用的结果。

<details>
<summary>查看答案</summary>

**第一次调用 PresentPicture(dt=0)**：
```
m_curpts == 0.0 → 进入首帧逻辑
m_curpts = picbuf.pts = 1.000  (设为第一帧的 PTS)
do-while:
  swap 帧[0] (pts=1.000), m_curPicture→1
  检查: 1.000 >= 1.033 → false，退出
结果: 显示帧[0]，m_usedPicture=3
```

**第二次调用 PresentPicture(dt=1.050)**：
```
m_curpts += 1.050 → m_curpts = 2.050
picbuf.pts = 1.033 < 2.050 → 不 return
do-while:
  迭代1: swap 帧[1] (pts=1.033), m_curPicture→2
         检查: 2.050 >= 1.067 → true
  迭代2: swap 帧[2] (pts=1.067), m_curPicture→3
         检查: 2.050 >= 1.100 → true
  迭代3: swap 帧[3] (pts=1.100), m_curPicture→0
         检查: m_usedPicture=0 → false，退出
结果: 显示帧[3]（最新帧），帧[1]和[2]被跳过
      m_usedPicture=0
```

**第三次调用 PresentPicture(dt=0.020)**：
```
m_usedPicture == 0 → 直接 return
结果: 无操作（环形缓冲区已空）
```

关键观察：第二次调用中由于 dt 极大（1.05秒），一次性跳过了所有缓冲帧，只显示最后一帧。这模拟了系统严重卡顿后的恢复行为——跳到最新帧以追上实时播放进度。

</details>

### 题目 3：如果要支持 BT.709（高清视频）色彩空间转换，需要修改哪些代码？

<details>
<summary>查看答案</summary>

需要修改 `YUVSprite.cpp` 中的 GLSL 片段着色器。BT.709 标准使用不同的转换矩阵：

```glsl
// 修改前（BT.601）：
vec3 rgb = mat3(
    1.164,  1.164, 1.164,
    0.0,   -0.392, 2.017,
    1.596, -0.813, 0.0
) * yuv;

// 修改后（BT.709）：
vec3 rgb = mat3(
    1.164,  1.164,  1.164,
    0.0,   -0.213,  2.112,
    1.793, -0.533,  0.0
) * yuv;
```

BT.709 对应的公式为：
```
R = 1.164 × (Y-16) + 1.793 × (V-128)
G = 1.164 × (Y-16) - 0.213 × (U-128) - 0.533 × (V-128)
B = 1.164 × (Y-16) + 2.112 × (U-128)
```

完整的实现方案：

1. **着色器支持两种标准**：在着色器中通过 uniform 变量选择矩阵
2. **运行时检测**：根据视频流的 `color_space` 元数据自动选择
3. **修改文件清单**：
   - `YUVSprite.h`：添加 `void setColorSpace(int cs)` 方法
   - `YUVSprite.cpp`：着色器中添加 uniform mat3 和条件逻辑
   - `KRMoviePlayer.cpp`：在创建 sprite 后根据 AVCodecContext::colorspace 设置

```cpp
// 示例：扩展着色器支持 uniform 矩阵
static const char *_yuvRenderProgram_fsh =
    "#ifdef GL_ES\n"
    "precision lowp float;\n"
    "#endif\n"
    "varying vec4 v_fragmentColor;\n"
    "varying vec2 v_texCoord;\n"
    "uniform mat3 u_yuvMatrix;\n"   // 新增：YUV 转换矩阵
    "void main() {\n"
    "    vec3 yuv;\n"
    "    yuv.x = texture2D(CC_Texture0, v_texCoord).r - 0.0625;\n"
    "    yuv.y = texture2D(CC_Texture1, v_texCoord).r - 0.5;\n"
    "    yuv.z = texture2D(CC_Texture2, v_texCoord).r - 0.5;\n"
    "    vec3 rgb = u_yuvMatrix * yuv;\n"  // 使用 uniform 矩阵
    "    gl_FragColor = vec4(rgb, 1.0) * v_fragmentColor;\n"
    "}\n";
```

</details>

## 下一步

[05-XBMC-Kodi遗产代码导读/01-CDVDMsg消息系统](../05-XBMC-Kodi遗产代码导读/01-CDVDMsg消息系统.md) —— 深入分析从 Kodi 移植来的 CDVDMsg 消息系统，理解解码管线中的消息传递机制。

