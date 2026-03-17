## OpenGL 渲染管线详解

OpenGL 的渲染管线（Rendering Pipeline）描述了从"应用程序提交顶点数据"到"屏幕上显示像素"的完整流程。

### 管线总览

```
应用程序（C++ 代码）
    │
    │  glDrawArrays / glDrawElements
    ↓
┌──────────────┐
│ 1. 顶点获取    │  从 VBO 读取顶点数据（位置、颜色、纹理坐标等）
│    (Vertex    │
│     Fetch)    │
└──────┬───────┘
       ↓
┌──────────────┐
│ 2. 顶点着色器  │  ★ 可编程阶段 ★
│   (Vertex     │  变换顶点坐标（模型→世界→裁剪空间）
│    Shader)    │  传递数据给后续阶段
└──────┬───────┘
       ↓
┌──────────────┐
│ 3. 图元装配    │  将顶点组装为几何图元（三角形/线段/点）
│  (Primitive   │
│   Assembly)   │
└──────┬───────┘
       ↓
┌──────────────┐
│ 4. 光栅化      │  将三角形转换为片段（Fragment）
│ (Rasterization│  每个片段对应屏幕上一个潜在的像素
│              )│  插值顶点属性（颜色、纹理坐标等）
└──────┬───────┘
       ↓
┌──────────────┐
│ 5. 片段着色器  │  ★ 可编程阶段 ★
│  (Fragment    │  计算每个片段的最终颜色
│   Shader)     │  纹理采样、光照计算、颜色混合
└──────┬───────┘
       ↓
┌──────────────┐
│ 6. 逐片段操作  │  深度测试（Z-buffer）
│  (Per-Fragment│  模板测试（Stencil Test）
│   Operations) │  Alpha 混合（Blending）
└──────┬───────┘
       ↓
┌──────────────┐
│ 7. 帧缓冲      │  最终像素颜色写入帧缓冲
│ (Framebuffer) │  显示到屏幕（或用于后续处理）
└──────────────┘
```

### 阶段 1：顶点获取

GPU 从**顶点缓冲对象**（VBO, Vertex Buffer Object）中读取顶点数据。顶点数据通常包含：

```cpp
// 一个典型的 2D 顶点结构
struct Vertex2D {
    float x, y;      // 位置坐标
    float u, v;       // 纹理坐标
    float r, g, b, a; // 颜色
};

// 绘制一个矩形需要 4 个顶点 + 6 个索引（2 个三角形）
Vertex2D vertices[] = {
    // 位置           纹理坐标    颜色
    {-0.5f, -0.5f,   0.0f, 0.0f, 1, 1, 1, 1},  // 左下
    { 0.5f, -0.5f,   1.0f, 0.0f, 1, 1, 1, 1},  // 右下
    { 0.5f,  0.5f,   1.0f, 1.0f, 1, 1, 1, 1},  // 右上
    {-0.5f,  0.5f,   0.0f, 1.0f, 1, 1, 1, 1},  // 左上
};

// 索引数组：两个三角形组成一个矩形
unsigned int indices[] = {
    0, 1, 2,  // 第一个三角形
    0, 2, 3,  // 第二个三角形
};
```

> **为什么用三角形而不是矩形？** GPU 硬件只支持三角形作为基本图元（除了点和线），因为三角形永远是平面的、凸的，便于光栅化。矩形是两个三角形的组合。

### 阶段 2：顶点着色器（可编程）

顶点着色器用 GLSL（OpenGL Shading Language）编写，**对每个顶点执行一次**。最核心的任务是将顶点从**模型空间**变换到**裁剪空间**：

```glsl
// 一个简单的 2D 顶点着色器
#version 330 core                    // OpenGL 3.3 对应 GLSL 330

layout(location = 0) in vec2 aPos;   // 从 VBO 获取的位置
layout(location = 1) in vec2 aTexCoord; // 纹理坐标
layout(location = 2) in vec4 aColor; // 颜色

out vec2 vTexCoord;                  // 传给片段着色器
out vec4 vColor;

uniform mat4 uProjection;           // 投影矩阵（由 CPU 传入）

void main() {
    // gl_Position 是内置输出变量，表示裁剪空间坐标
    gl_Position = uProjection * vec4(aPos, 0.0, 1.0);
    vTexCoord = aTexCoord;
    vColor = aColor;
}
```

**坐标空间变换链**（2D 场景简化版）：

```
模型空间（Model Space）    — 物体自身的坐标系
    ↓ × 模型矩阵（Model Matrix）
世界空间（World Space）    — 场景中的绝对位置
    ↓ × 视图矩阵（View Matrix）
观察空间（View Space）     — 摄像机视角
    ↓ × 投影矩阵（Projection Matrix）
裁剪空间（Clip Space）     — NDC 标准化设备坐标
    ↓ GPU 自动执行透视除法 + 视口变换
屏幕空间（Screen Space）   — 最终的像素坐标
```

对于 2D 游戏（如 KrKr2），通常只需要一个正交投影矩阵（Orthographic Projection），将屏幕像素坐标直接映射到 NDC：

```
正交投影：将 [0, 1920] × [0, 1080] 映射到 [-1, 1] × [-1, 1]
```

### 阶段 3-4：图元装配与光栅化

图元装配将顶点组装为三角形。光栅化将三角形覆盖的每个像素位置生成一个**片段**（Fragment）。

```
三角形 ABC：
A(100, 50) ─── B(200, 50)
    \              /
     \            /
      \          /
       \        /
        C(150, 150)

光栅化后生成片段：
..........▓▓▓▓▓▓..........    y=50
.........▓▓▓▓▓▓▓▓.........    y=60
........▓▓▓▓▓▓▓▓▓▓........    y=70
.......▓▓▓▓▓▓▓▓▓▓▓▓.......    y=80
......▓▓▓▓▓▓▓▓▓▓▓▓▓▓......    y=90
.....▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓.....    y=100
每个 ▓ 就是一个片段，需要经过片段着色器处理
```

光栅化还会对顶点属性进行**重心插值**（Barycentric Interpolation）：如果三角形三个顶点的颜色分别是红、绿、蓝，那么三角形中间某一点的颜色会是三者的平滑混合。

### 阶段 5：片段着色器（可编程）

片段着色器**对每个片段执行一次**，决定该片段的最终颜色：

```glsl
// 一个简单的 2D 片段着色器
#version 330 core

in vec2 vTexCoord;    // 从顶点着色器插值得到
in vec4 vColor;

out vec4 FragColor;   // 最终输出颜色

uniform sampler2D uTexture;  // 纹理采样器

void main() {
    // 从纹理采样颜色，乘以顶点颜色
    vec4 texColor = texture(uTexture, vTexCoord);
    FragColor = texColor * vColor;

    // 如果 Alpha 接近 0，丢弃该片段（透明裁剪）
    if (FragColor.a < 0.01)
        discard;  // 该片段不会写入帧缓冲
}
```

> **KrKr2 的渲染特点**：KrKr2 是视觉小说引擎，大量使用 Alpha 混合（透明度混合）来实现立绘叠加、过渡效果。理解片段着色器中的 Alpha 处理对理解 KrKr2 渲染至关重要。

### 阶段 6-7：逐片段操作与帧缓冲

经过片段着色器后，片段还要经过以下测试：

1. **深度测试**（Depth Test）：比较片段的 Z 值与深度缓冲中的值，离摄像机更近的片段才被保留。2D 游戏通常按绘制顺序决定前后关系，可以关闭深度测试
2. **模板测试**（Stencil Test）：按模板缓冲的掩码决定是否绘制，用于实现遮罩效果
3. **Alpha 混合**（Blending）：将新片段颜色与帧缓冲中已有颜色混合

```cpp
// 典型的 Alpha 混合公式
// result = src_color * src_alpha + dst_color * (1 - src_alpha)

// OpenGL 设置 Alpha 混合
glEnable(GL_BLEND);
glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);

// 示例：半透明红色(1,0,0,0.5) 覆盖在白色(1,1,1,1)上
// R = 1.0 * 0.5 + 1.0 * (1 - 0.5) = 1.0
// G = 0.0 * 0.5 + 1.0 * (1 - 0.5) = 0.5
// B = 0.0 * 0.5 + 1.0 * (1 - 0.5) = 0.5
// 结果：浅粉色 (1.0, 0.5, 0.5)
```

## OpenGL 状态机模型

OpenGL 的 API 设计基于**有限状态机**（Finite State Machine）模型。这意味着：

1. OpenGL 维护一个全局状态对象（当前绑定的 VBO、当前使用的着色器、当前激活的纹理单元等）
2. API 调用**修改状态**或**执行绘制**，两者分开
3. 绘制命令使用**当前状态**，而不是显式传参

```cpp
// OpenGL 的"状态-绘制"模式
// 第一步：设置状态
glBindVertexArray(vao);          // 绑定 VAO（状态改变）
glUseProgram(shaderProgram);     // 使用着色器程序（状态改变）
glBindTexture(GL_TEXTURE_2D, tex); // 绑定纹理（状态改变）
glUniformMatrix4fv(loc, 1, GL_FALSE, &proj[0][0]); // 设置 uniform

// 第二步：执行绘制（使用当前状态）
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);

// 如果要画另一个物体，需要切换状态
glBindTexture(GL_TEXTURE_2D, anotherTex);  // 切换纹理
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);  // 再次绘制
```

> **状态切换的开销**：每次 `glBind*` 都有 CPU 开销（驱动需要验证和记录状态变化），频繁切换纹理/着色器是性能杀手。这就是为什么 Cocos2d-x（以及所有游戏引擎）都实现了**批量渲染**（Batching）——把使用相同状态的绘制调用合并。

### 常见错误：忘记状态是全局的

```cpp
// 危险代码！
void DrawSpriteA() {
    glBindTexture(GL_TEXTURE_2D, texA);
    glDrawElements(...);
    // 注意：texA 仍然处于绑定状态！
}

void DrawSpriteB() {
    // 忘记绑定 texB，结果用了 texA 的纹理！
    glDrawElements(...);  // BUG：使用了上一个函数绑定的 texA
}

// 正确做法：每次绘制前显式设置所有需要的状态
void DrawSpriteB() {
    glBindTexture(GL_TEXTURE_2D, texB);  // 显式绑定
    glDrawElements(...);
}
```

