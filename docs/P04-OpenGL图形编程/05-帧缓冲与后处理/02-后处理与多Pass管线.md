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

