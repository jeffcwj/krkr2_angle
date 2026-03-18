# 后处理与多 Pass 管线

> **所属模块：** P04-OpenGL 图形编程  
> **前置知识：** [帧缓冲基础与创建](./01-帧缓冲基础与创建.md)、[着色器编程](../03-着色器编程/01-GLSL语言与着色器阶段.md)、纹理采样基础  
> **预计阅读时间：** 50 分钟

## 本节目标

读完本节后，你将能够：

1. 区分 RBO（Renderbuffer Object，渲染缓冲对象——一种只写的离屏缓冲区）与纹理附件的适用场景。
2. 创建深度纹理附件用于阴影映射（shadow mapping，一种通过从光源视角渲染深度图来计算阴影的技术）等高级效果。
3. 实现灰度化、核卷积（锐化）、高斯模糊、边缘检测四种后处理效果。
4. 使用 Ping-Pong FBO（帧缓冲对象，Frame Buffer Object）技术实现多 Pass（多次渲染遍历）后处理管线。
5. 处理窗口大小变化时 FBO 的重建逻辑。
6. 理解 FBO 在 Desktop GL 与 OpenGL ES 2.0/3.0 之间的差异及兼容写法。

## 渲染缓冲对象（RBO）详解

### RBO vs 纹理附件的选择

渲染缓冲对象（Renderbuffer Object, RBO）是一种不可采样的离屏缓冲区。与纹理附件相比：

```
           纹理附件                    RBO
    ┌──────────────────┐      ┌──────────────────┐
    │  可读可写          │      │  只写（写入优化）  │
    │  可作为采样器输入   │      │  不可采样          │
    │  格式灵活          │      │  格式受限          │
    │  适合：颜色附件     │      │  适合：深度/模板   │
    └──────────────────┘      └──────────────────┘
```

**选择原则**：
- 如果后续需要**读取**该附件的数据（如后处理读取颜色） → 用**纹理**
- 如果只需要 OpenGL 内部**写入**（如深度测试、模板测试） → 用 **RBO**（性能更优）

### 创建深度纹理附件

某些场景需要读取深度信息（如阴影映射），此时要用纹理而非 RBO：

```cpp
GLuint depthTexture;
glGenTextures(1, &depthTexture);
glBindTexture(GL_TEXTURE_2D, depthTexture);
// GL_DEPTH_COMPONENT24：24位深度精度
glTexImage2D(GL_TEXTURE_2D, 0, GL_DEPTH_COMPONENT24,
             SCR_WIDTH, SCR_HEIGHT, 0,
             GL_DEPTH_COMPONENT, GL_FLOAT, nullptr);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
// 深度纹理边缘用白色（最大深度），避免阴影边缘伪影
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_BORDER);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_BORDER);
float borderColor[] = {1.0f, 1.0f, 1.0f, 1.0f};
glTexParameterfv(GL_TEXTURE_2D, GL_TEXTURE_BORDER_COLOR, borderColor);
// 附加到 FBO 深度附件
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT,
                       GL_TEXTURE_2D, depthTexture, 0);
```

> **跨平台注意**：`GL_CLAMP_TO_BORDER` 在 OpenGL ES 2.0 中**不可用**，ES 3.2 才支持。在移动端需要改用 `GL_CLAMP_TO_EDGE` 并在着色器中手动处理边界。

## 后处理效果实战

后处理的核心思路：先把场景渲染到 FBO 的颜色纹理上，再用一个**全屏四边形 + 后处理着色器**读取该纹理并输出到默认帧缓冲。不同的后处理效果只是片段着色器不同。

### 后处理顶点着色器（通用）

所有后处理效果共享同一个顶点着色器：

```glsl
#version 330 core
layout(location = 0) in vec2 aPos;      // NDC 坐标
layout(location = 1) in vec2 aTexCoord; // 纹理坐标

out vec2 TexCoord;

void main() {
    gl_Position = vec4(aPos, 0.0, 1.0);  // 直接使用 NDC 坐标
    TexCoord = aTexCoord;
}
```

### 效果一：灰度化（Grayscale）

将彩色图像转为灰度图。使用人眼感知的加权公式（亮度 = 0.2126R + 0.7152G + 0.0722B）：

```glsl
#version 330 core
in vec2 TexCoord;
out vec4 FragColor;

uniform sampler2D screenTexture;  // FBO 颜色纹理

void main() {
    vec4 color = texture(screenTexture, TexCoord);
    // ITU-R BT.709 亮度系数，人眼对绿色最敏感
    float gray = dot(color.rgb, vec3(0.2126, 0.7152, 0.0722));
    FragColor = vec4(gray, gray, gray, 1.0);
}
```

### 效果二：核卷积（Kernel Convolution）

卷积是图像处理的基石。对每个像素，取其周围 3×3 邻域，用**卷积核**加权求和：

```
  核（Kernel）3×3 示例 — 锐化：
  ┌────┬────┬────┐
  │  0 │ -1 │  0 │
  ├────┼────┼────┤
  │ -1 │  5 │ -1 │
  ├────┼────┼────┤
  │  0 │ -1 │  0 │
  └────┴────┴────┘
```

通用核卷积片段着色器：

```glsl
#version 330 core
in vec2 TexCoord;
out vec4 FragColor;

uniform sampler2D screenTexture;

// 纹理尺寸的倒数，用于计算相邻像素的 UV 偏移
uniform vec2 texelSize;  // = vec2(1.0/width, 1.0/height)

void main() {
    // 定义不同效果的卷积核 — 这里用锐化核
    float kernel[9] = float[](
         0.0, -1.0,  0.0,
        -1.0,  5.0, -1.0,
         0.0, -1.0,  0.0
    );

    // 3×3 邻域的偏移量
    vec2 offsets[9] = vec2[](
        vec2(-1, 1),  vec2(0, 1),  vec2(1, 1),   // 上方一行
        vec2(-1, 0),  vec2(0, 0),  vec2(1, 0),   // 当前行
        vec2(-1,-1),  vec2(0,-1),  vec2(1,-1)    // 下方一行
    );

    vec3 result = vec3(0.0);
    for (int i = 0; i < 9; i++) {
        // 采样相邻像素并乘以对应的核权重
        vec3 sample = texture(screenTexture,
                              TexCoord + offsets[i] * texelSize).rgb;
        result += sample * kernel[i];
    }

    FragColor = vec4(result, 1.0);
}
```

### 效果三：高斯模糊（Gaussian Blur）

高斯模糊是最常用的模糊算法。直接用 N×N 核开销大，实际常用**两步分离式模糊**（先水平后垂直），将 O(N²) 降为 O(2N)：

```
完整高斯模糊 = 水平 pass + 垂直 pass

Pass 1（水平）:        Pass 2（垂直）:
FBO_A → FBO_B         FBO_B → 默认帧缓冲
━━━━━━━━━━━━━         ━━━━━━━━━━━━━━━━━━
对每行做 1D 模糊       对每列做 1D 模糊
```

这需要**两个 FBO**（或一个 FBO 的 Ping-Pong 切换）。水平模糊着色器：

```glsl
#version 330 core
in vec2 TexCoord;
out vec4 FragColor;

uniform sampler2D image;
uniform vec2 texelSize;

// 5-tap 高斯权重（sigma ≈ 1.5）
const float weight[5] = float[](
    0.227027, 0.1945946, 0.1216216, 0.054054, 0.016216
);

void main() {
    vec3 result = texture(image, TexCoord).rgb * weight[0];
    // 水平方向采样
    for (int i = 1; i < 5; i++) {
        vec2 offset = vec2(texelSize.x * float(i), 0.0);
        result += texture(image, TexCoord + offset).rgb * weight[i];
        result += texture(image, TexCoord - offset).rgb * weight[i];
    }
    FragColor = vec4(result, 1.0);
}
```

垂直模糊着色器与上述类似，只需把偏移方向改为 `vec2(0.0, texelSize.y * float(i))`。

### 效果四：边缘检测（Sobel 算子）

使用 Sobel 算子检测图像中的边缘。Sobel 核分水平和垂直两个方向：

```glsl
#version 330 core
in vec2 TexCoord;
out vec4 FragColor;

uniform sampler2D screenTexture;
uniform vec2 texelSize;

void main() {
    // Sobel 水平核 Gx
    float Gx[9] = float[](
        -1, 0, 1,
        -2, 0, 2,
        -1, 0, 1
    );
    // Sobel 垂直核 Gy
    float Gy[9] = float[](
        -1, -2, -1,
         0,  0,  0,
         1,  2,  1
    );

    vec2 offsets[9] = vec2[](
        vec2(-1, 1), vec2(0, 1), vec2(1, 1),
        vec2(-1, 0), vec2(0, 0), vec2(1, 0),
        vec2(-1,-1), vec2(0,-1), vec2(1,-1)
    );

    vec3 sumX = vec3(0.0), sumY = vec3(0.0);
    for (int i = 0; i < 9; i++) {
        vec3 s = texture(screenTexture,
                         TexCoord + offsets[i] * texelSize).rgb;
        sumX += s * Gx[i];
        sumY += s * Gy[i];
    }
    // 梯度幅值 = sqrt(Gx² + Gy²)
    vec3 edge = sqrt(sumX * sumX + sumY * sumY);
    FragColor = vec4(edge, 1.0);
}
```

## 多 Pass 后处理管线

实际项目中经常需要串联多个后处理效果。这就需要**Ping-Pong FBO**技术：

```
场景 ──→ FBO_A ──→ FBO_B ──→ FBO_A ──→ 屏幕
         (渲染)    (效果1)   (效果2)   (最终)
```

```cpp
// Ping-Pong FBO 管线示例
GLuint pingpongFBO[2], pingpongTex[2];

void initPingPongFBOs(int width, int height) {
    glGenFramebuffers(2, pingpongFBO);
    glGenTextures(2, pingpongTex);
    for (int i = 0; i < 2; i++) {
        glBindFramebuffer(GL_FRAMEBUFFER, pingpongFBO[i]);
        glBindTexture(GL_TEXTURE_2D, pingpongTex[i]);
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA,
                     width, height, 0,
                     GL_RGBA, GL_UNSIGNED_BYTE, nullptr);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
        glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,
                               GL_TEXTURE_2D, pingpongTex[i], 0);
    }
}

void renderWithMultiPass() {
    // Pass 0: 场景渲染到 FBO_A
    glBindFramebuffer(GL_FRAMEBUFFER, pingpongFBO[0]);
    renderScene();

    // Pass 1: FBO_A → 高斯模糊水平 → FBO_B
    glBindFramebuffer(GL_FRAMEBUFFER, pingpongFBO[1]);
    blurHShader.use();
    glBindTexture(GL_TEXTURE_2D, pingpongTex[0]);
    drawFullscreenQuad();

    // Pass 2: FBO_B → 高斯模糊垂直 → FBO_A
    glBindFramebuffer(GL_FRAMEBUFFER, pingpongFBO[0]);
    blurVShader.use();
    glBindTexture(GL_TEXTURE_2D, pingpongTex[1]);
    drawFullscreenQuad();

    // 最终 Pass: FBO_A → 色调映射 → 屏幕
    glBindFramebuffer(GL_FRAMEBUFFER, 0);
    tonemapShader.use();
    glBindTexture(GL_TEXTURE_2D, pingpongTex[0]);
    drawFullscreenQuad();
}
```

### FBO 尺寸与窗口尺寸

当窗口大小改变时，FBO 也需要重建：

```cpp
void onWindowResize(int newWidth, int newHeight) {
    // 重新分配颜色纹理
    glBindTexture(GL_TEXTURE_2D, colorTexture);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA,
                 newWidth, newHeight, 0,
                 GL_RGBA, GL_UNSIGNED_BYTE, nullptr);
    // 重新分配 RBO
    glBindRenderbuffer(GL_RENDERBUFFER, rbo);
    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH24_STENCIL8,
                          newWidth, newHeight);
    // 重新检查完整性
    glBindFramebuffer(GL_FRAMEBUFFER, fbo);
    if (glCheckFramebufferStatus(GL_FRAMEBUFFER)
        != GL_FRAMEBUFFER_COMPLETE) {
        std::cerr << "FBO resize 后不完整！" << std::endl;
    }
    glBindFramebuffer(GL_FRAMEBUFFER, 0);
}
```

## 跨平台 FBO 差异

| 特性 | Desktop GL 3.3+ | OpenGL ES 2.0 | OpenGL ES 3.0+ |
|------|-----------------|---------------|-----------------|
| FBO 核心支持 | ✅ 核心 | ✅ 核心 | ✅ 核心 |
| MRT（多渲染目标） | ✅ 最多 8 个 | ❌（需扩展 `NV_draw_buffers`） | ✅ 最多 4 个 |
| `GL_DEPTH24_STENCIL8` | ✅ | ❌（需 `OES_packed_depth_stencil`） | ✅ |
| 深度纹理 | ✅ | 需 `OES_depth_texture` | ✅ |
| `glBlitFramebuffer` | ✅ | ❌ | ✅ |
| `GL_READ_FRAMEBUFFER` / `GL_DRAW_FRAMEBUFFER` | ✅ | ❌ | ✅ |
| MSAA FBO | ✅ `glRenderbufferStorageMultisample` | 需扩展 | ✅ |

### ES 2.0 兼容写法

在 ES 2.0 中，深度+模板需要分开创建：

```cpp
#ifdef GL_ES_VERSION_2_0
    // ES 2.0：分别创建深度和模板 RBO
    GLuint depthRBO, stencilRBO;
    glGenRenderbuffers(1, &depthRBO);
    glBindRenderbuffer(GL_RENDERBUFFER, depthRBO);
    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT16,
                          width, height);
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT,
                              GL_RENDERBUFFER, depthRBO);

    glGenRenderbuffers(1, &stencilRBO);
    glBindRenderbuffer(GL_RENDERBUFFER, stencilRBO);
    glRenderbufferStorage(GL_RENDERBUFFER, GL_STENCIL_INDEX8,
                          width, height);
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_STENCIL_ATTACHMENT,
                              GL_RENDERBUFFER, stencilRBO);
#else
    // Desktop GL / ES 3.0+：合并深度+模板
    GLuint rbo;
    glGenRenderbuffers(1, &rbo);
    glBindRenderbuffer(GL_RENDERBUFFER, rbo);
    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH24_STENCIL8,
                          width, height);
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT,
                               GL_RENDERBUFFER, rbo);
#endif
```

## 动手实践

### 实践 1：实现灰度化后处理

按以下步骤实现：

1. 创建一个 FBO，附加一张颜色纹理和一个 RBO 深度/模板附件。
2. 第一遍（Pass 1）：将 3D 场景（或简单的彩色三角形）渲染到 FBO。
3. 第二遍（Pass 2）：绑定默认帧缓冲（`glBindFramebuffer(GL_FRAMEBUFFER, 0)`），用全屏四边形 + 灰度化片段着色器绘制 FBO 的颜色纹理。

```cpp
// 全屏四边形顶点数据（NDC 坐标 + 纹理坐标）
float quadVertices[] = {
    // 位置x, 位置y,  纹理u, 纹理v
    -1.0f,  1.0f,     0.0f, 1.0f,   // 左上
    -1.0f, -1.0f,     0.0f, 0.0f,   // 左下
     1.0f, -1.0f,     1.0f, 0.0f,   // 右下

    -1.0f,  1.0f,     0.0f, 1.0f,   // 左上
     1.0f, -1.0f,     1.0f, 0.0f,   // 右下
     1.0f,  1.0f,     1.0f, 1.0f    // 右上
};

// 设置全屏四边形 VAO/VBO
GLuint quadVAO, quadVBO;
glGenVertexArrays(1, &quadVAO);
glGenBuffers(1, &quadVBO);
glBindVertexArray(quadVAO);
glBindBuffer(GL_ARRAY_BUFFER, quadVBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(quadVertices), quadVertices, GL_STATIC_DRAW);
// 位置属性
glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 4 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
// 纹理坐标属性
glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, 4 * sizeof(float), (void*)(2 * sizeof(float)));
glEnableVertexAttribArray(1);
glBindVertexArray(0);

// 绘制全屏四边形
void drawFullscreenQuad() {
    glBindVertexArray(quadVAO);
    glDrawArrays(GL_TRIANGLES, 0, 6);  // 两个三角形组成全屏矩形
    glBindVertexArray(0);
}
```

### 实践 2：Ping-Pong 高斯模糊

在实践 1 的基础上，创建两个 FBO（FBO_A 和 FBO_B），先水平模糊再垂直模糊。可以通过增加迭代次数（多次水平+垂直）来增强模糊强度。

## 对照项目源码

KrKr2 项目中帧缓冲与后处理相关代码：

| 文件 | 行号范围 | 内容 |
|------|---------|------|
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 469-471 | 全局 FBO 变量 `_FBO` 和 `_CurrentRenderTarget`，引擎使用单个全局 FBO 进行离屏渲染 |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 463-468 | `GLVertexInfo` 结构体，存储纹理指针与顶点坐标，用于后续绘制 |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 2300-2371 | 纹理恢复路径使用硬编码 NDC 全屏四边形坐标 + `glDrawArrays(GL_TRIANGLES, 0, 6)`，本质就是一个后处理 Pass |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 1860-2060 | 动态着色器编译系统（`CompileShader`、`CombineProgram`），支持运行时组合多种后处理效果 |

**关键模式观察：**

1. **KrKr2 不直接使用多 Pass 后处理：** 引擎主要通过 Cocos2d-x 的渲染命令队列实现效果叠加，而非手动 Ping-Pong FBO。
2. **FBO 管理是全局的：** `TVPSetRenderTarget` 函数控制渲染目标切换，简化了 FBO 生命周期管理。
3. **全屏四边形模式一致：** 引擎中 NDC 四边形的写法（两个三角形，6 个顶点）与本节讲解的方式完全相同。

## 常见错误及解决方案

### 错误 1：后处理画面上下颠倒

**现象：** 后处理效果应用后，画面上下翻转。

**原因：** FBO 纹理的坐标原点在左下角（OpenGL 默认），但渲染循环可能假设左上角为原点。全屏四边形的纹理坐标映射反了。

**解决方案：** 检查全屏四边形的纹理坐标 V 分量是否正确。如果画面颠倒，将 V 坐标翻转（`1.0 - v`），或在顶点着色器中做翻转：
```glsl
TexCoord = vec2(aTexCoord.x, 1.0 - aTexCoord.y); // 翻转 V 轴
```

### 错误 2：Ping-Pong 模糊结果全黑

**现象：** 第一个 Pass 正常，切换到第二个 FBO 后输出全黑。

**原因：** 在 Pass 2 绑定 FBO_B 之前没有解绑 FBO_A 的纹理，导致"同时读写同一纹理"——这在 OpenGL 中是未定义行为。

**解决方案：** 确保每个 Pass 绑定的**输入纹理**和**输出 FBO 的附件纹理**不是同一个对象：
```cpp
// Pass 1: FBO_A → 处理 → FBO_B
glBindFramebuffer(GL_FRAMEBUFFER, pingpongFBO[1]);    // 输出到 FBO_B
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, pingpongTex[0]);          // 读取 FBO_A 的纹理
// ↑ pingpongTex[0] 和 pingpongFBO[1] 的附件是不同纹理 ✅
```

### 错误 3：窗口缩放后后处理效果拉伸或模糊

**现象：** 调整窗口大小后，后处理效果出现拉伸、模糊或分辨率不匹配。

**原因：** 窗口大小变了但 FBO 纹理没有重建，仍然是旧分辨率。

**解决方案：** 在窗口 resize 回调中重建所有 FBO 纹理和 RBO：
```cpp
void onResize(int w, int h) {
    glViewport(0, 0, w, h);                      // 更新视口
    // 重建每个 FBO 的颜色纹理
    for (int i = 0; i < 2; i++) {
        glBindTexture(GL_TEXTURE_2D, pingpongTex[i]);
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, w, h, 0,
                     GL_RGBA, GL_UNSIGNED_BYTE, nullptr);
    }
    // 别忘了重建 RBO（如果有深度/模板）
    glBindRenderbuffer(GL_RENDERBUFFER, rbo);
    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH24_STENCIL8, w, h);
}
```

## 本节小结

- **RBO vs 纹理附件**：RBO 只写不可采样，适合深度/模板附件；纹理附件可采样，适合颜色附件和需要后续读取的场景。
- **后处理核心模式**：渲染到 FBO → 全屏四边形 + 后处理着色器 → 输出到默认帧缓冲。
- **四种基础后处理效果**：灰度化（亮度加权）、锐化（核卷积）、高斯模糊（分离式两 Pass）、边缘检测（Sobel 算子）。
- **Ping-Pong FBO**：串联多个后处理效果的标准技术，交替使用两个 FBO 避免同时读写同一纹理。
- **窗口缩放时必须重建 FBO**：纹理和 RBO 都需要按新分辨率重新分配。
- **ES 2.0 兼容性**：深度+模板需分开创建；不支持 `GL_CLAMP_TO_BORDER`；不支持 MRT；不支持 `glBlitFramebuffer`。

## 练习题与答案

### 题目 1：实现色调反转后处理

编写一个片段着色器，将输入图像的颜色进行反转（即 `1.0 - color`），同时保持 Alpha 通道不变。

<details>
<summary>查看答案</summary>

```glsl
#version 330 core
in vec2 TexCoord;
out vec4 FragColor;
uniform sampler2D screenTexture;

void main() {
    vec4 color = texture(screenTexture, TexCoord); // 采样 FBO 颜色纹理
    FragColor = vec4(1.0 - color.rgb, color.a);    // RGB 反转，Alpha 保持
}
```

**关键点：** `1.0 - color.rgb` 利用了 GLSL 的标量-向量运算广播。如果写成 `vec3(1.0) - color.rgb` 效果完全相同，但前者更简洁。

</details>

### 题目 2：分析 Ping-Pong 迭代次数对模糊强度的影响

如果把高斯模糊的 Ping-Pong 迭代从 2 次（水平+垂直各 1 次）增加到 10 次（水平+垂直各 5 次），请回答：

1. 模糊的视觉效果有什么变化？
2. 性能开销如何变化？
3. 有没有比增加迭代次数更高效的方式来增强模糊强度？

<details>
<summary>查看答案</summary>

1. **视觉变化：** 模糊半径显著增大，图像细节更加模糊。多次迭代高斯模糊等价于使用更大 sigma 值的单次高斯模糊（根据卷积定理，多次高斯卷积的 sigma 按 `sqrt(σ1² + σ2² + ...)` 叠加）。

2. **性能开销：** 线性增长。每次迭代都是一个完整的全屏 Pass（绑定 FBO → 绑定纹理 → 绘制全屏四边形），所以 10 次迭代的开销约为 2 次的 5 倍。在 4K 分辨率下，每个 Pass 处理约 830 万像素，10 次 = 8300 万次片段着色器调用。

3. **更高效的替代方案：**
   - **增大采样半径**：在着色器中增加采样点数（如 5-tap → 13-tap），单次 Pass 就能覆盖更大范围。
   - **降采样模糊**：先将图像缩小到 1/2 或 1/4 分辨率，在低分辨率上做模糊，再放大回来。像素数减少 4-16 倍，性能大幅提升。
   - **Kawase 模糊**：一种迭代模糊算法，每次迭代使用不同的采样偏移距离，用更少的迭代次数达到相似的模糊效果。

</details>

### 题目 3：ES 2.0 深度+模板创建

在 OpenGL ES 2.0 环境下，编写代码为一个 FBO 添加深度和模板附件。说明为什么不能使用 `GL_DEPTH24_STENCIL8`。

<details>
<summary>查看答案</summary>

```cpp
// ES 2.0 不支持 GL_DEPTH24_STENCIL8 合并格式
// 原因：ES 2.0 规范中没有定义 packed depth-stencil 格式
// 需要扩展 OES_packed_depth_stencil 才支持，不是所有设备都有

// 标准做法：分别创建深度 RBO 和模板 RBO
GLuint depthRBO, stencilRBO;

// 创建深度 RBO
glGenRenderbuffers(1, &depthRBO);                          // 生成 RBO 名称
glBindRenderbuffer(GL_RENDERBUFFER, depthRBO);              // 绑定
glRenderbufferStorage(GL_RENDERBUFFER,
                      GL_DEPTH_COMPONENT16,                 // ES 2.0 只保证 16 位深度
                      width, height);                       // 尺寸
glFramebufferRenderbuffer(GL_FRAMEBUFFER,
                          GL_DEPTH_ATTACHMENT,               // 附加到深度附件点
                          GL_RENDERBUFFER, depthRBO);

// 创建模板 RBO
glGenRenderbuffers(1, &stencilRBO);
glBindRenderbuffer(GL_RENDERBUFFER, stencilRBO);
glRenderbufferStorage(GL_RENDERBUFFER,
                      GL_STENCIL_INDEX8,                    // 8 位模板
                      width, height);
glFramebufferRenderbuffer(GL_FRAMEBUFFER,
                          GL_STENCIL_ATTACHMENT,             // 附加到模板附件点
                          GL_RENDERBUFFER, stencilRBO);

// 检查 FBO 完整性
if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE) {
    // 处理错误
}
```

**为什么不能用 `GL_DEPTH24_STENCIL8`：** 该格式在 ES 2.0 核心规范中未定义。ES 2.0 只要求支持 `GL_DEPTH_COMPONENT16` 和 `GL_STENCIL_INDEX8`。合并的深度-模板格式需要 `OES_packed_depth_stencil` 扩展，而这个扩展并非所有 ES 2.0 设备都支持。ES 3.0 才将 `GL_DEPTH24_STENCIL8` 纳入核心规范。

</details>

## 下一步

下一节 [实践与常见错误](./03-实践与常见错误.md) 将综合本章所有 FBO 知识，通过一个完整的多效果后处理 Demo 进行实战练习，并深入分析 FBO 相关的调试技巧与跨平台陷阱。

