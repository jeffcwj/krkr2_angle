# GPU 渲染管线概述

> **所属模块：** P04-OpenGL 图形编程
> **前置知识：** [P03-跨平台基础](../../P03-跨平台C++开发/01-跨平台基础/01-操作系统差异.md)（理解平台差异）、C++ 基础（指针、结构体）
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：

1. **描述 GPU 的基本架构**，解释为什么 GPU 适合图形渲染而 CPU 不适合
2. **画出 OpenGL 渲染管线的各阶段**，说明数据在每个阶段如何变换
3. **理解 OpenGL 状态机模型**，解释为什么 OpenGL API 看起来是"全局函数"
4. **区分 OpenGL 上下文与窗口系统**的关系，理解为什么创建 GL 上下文需要平台特定代码
5. **解释 KrKr2 为什么选择 Cocos2d-x** 而不是直接调用 OpenGL API

## 为什么要学 OpenGL？

KrKr2 模拟器的渲染路径可以简化为：

```
TJS2 脚本指令
    ↓
KrKr2 visual 层（图层管理、过渡效果）
    ↓
Cocos2d-x 渲染引擎
    ↓
OpenGL / OpenGL ES API
    ↓
GPU 硬件
```

虽然 KrKr2 不直接调用 OpenGL（而是通过 Cocos2d-x），但理解 OpenGL 是理解 Cocos2d-x 行为的基础。当你遇到以下问题时，OpenGL 知识是必需的：

- **纹理显示异常**——需要理解纹理格式、采样器参数
- **渲染性能问题**——需要理解 draw call、批量渲染、状态切换开销
- **移动端适配**——需要理解 OpenGL ES 的限制
- **添加后处理效果**——需要理解帧缓冲、离屏渲染

## GPU 架构基础

### CPU vs GPU：为什么需要专用硬件？

| 特性 | CPU | GPU |
|------|-----|-----|
| 核心数 | 4-32 个强核心 | 数千个弱核心 |
| 单核性能 | 极高（复杂分支、乱序执行） | 较低（简单 ALU） |
| 并行能力 | 有限（线程切换开销大） | 极强（SIMT 架构，天然并行） |
| 缓存 | 大（L1/L2/L3 共数十 MB） | 小（侧重带宽而非延迟） |
| 内存带宽 | 中等（50-100 GB/s） | 极高（200-1000 GB/s） |
| 适合任务 | 串行复杂逻辑 | 大规模并行计算 |

图形渲染的核心特征是**对大量像素做相同的计算**。一个 1920×1080 的屏幕有 207 万个像素，每个像素都需要执行：采样纹理 → 计算光照 → 混合颜色。这种"数据并行"（Data Parallelism）正是 GPU 的设计目标。

```
CPU 方式（串行处理 207 万像素）：
for (int y = 0; y < 1080; ++y)
    for (int x = 0; x < 1920; ++x)
        framebuffer[y][x] = shade(x, y);  // 一个一个来，慢！

GPU 方式（并行处理 207 万像素）：
// 每个 GPU 核心独立处理一个像素
// 数千个核心同时工作，几毫秒完成
void fragment_shader(int x, int y) {
    output = shade(x, y);  // 所有像素同时执行
}
```

### GPU 的基本组成

现代 GPU 通常包含以下组件：

```
┌─────────────────────────────────────────┐
│                  GPU                     │
│                                          │
│  ┌─────────────────────────────────┐    │
│  │     命令处理器（Command Proc.）    │    │
│  │   解析 CPU 提交的 GL 命令队列     │    │
│  └──────────┬──────────────────────┘    │
│             ↓                            │
│  ┌──────────────────────────────────┐   │
│  │    顶点处理单元（Vertex Shader）    │   │
│  │  × N 个 Shader Core 并行执行      │   │
│  └──────────┬───────────────────────┘   │
│             ↓                            │
│  ┌──────────────────────────────────┐   │
│  │    光栅化器（Rasterizer）          │   │
│  │  三角形 → 像素片段（固定功能）      │   │
│  └──────────┬───────────────────────┘   │
│             ↓                            │
│  ┌──────────────────────────────────┐   │
│  │    片段处理单元（Fragment Shader）  │   │
│  │  × N 个 Shader Core 并行执行      │   │
│  └──────────┬───────────────────────┘   │
│             ↓                            │
│  ┌──────────────────────────────────┐   │
│  │    输出合并（Output Merger）       │   │
│  │  深度测试 / Alpha 混合 / 写入帧缓冲│   │
│  └──────────────────────────────────┘   │
│                                          │
│  ┌──────────────────────────────────┐   │
│  │          显存（VRAM）              │   │
│  │  纹理、顶点缓冲、帧缓冲等数据     │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

> **对照 KrKr2**：KrKr2 的 `visual` 模块将图像数据（精灵、背景、文字）组织为图层，Cocos2d-x 将这些图层转换为 GPU 可以处理的顶点和纹理数据，然后通过 OpenGL 提交给 GPU 渲染。

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

## OpenGL 上下文与窗口系统

### 什么是 GL 上下文？

OpenGL 上下文（GL Context）是 OpenGL 状态机的实例。所有 GL 函数调用都隐式操作**当前线程绑定的上下文**。

创建 GL 上下文**不是 OpenGL 标准的一部分**——它由**窗口系统**（Window System）负责：

| 平台 | 窗口系统 API | 创建 GL 上下文的 API |
|------|-------------|---------------------|
| Windows | Win32 | WGL（`wglCreateContext`） |
| Linux | X11 / Wayland | GLX（`glXCreateContext`）/ EGL |
| macOS | Cocoa | CGL（`CGLCreateContext`）/ NSOpenGLContext |
| Android | EGL | EGL（`eglCreateContext`） |
| 跨平台 | SDL2 / GLFW | 封装上述所有 API |

```cpp
// Windows 上创建 GL 上下文的简化流程
// 1. 创建窗口
HWND hwnd = CreateWindowEx(...);
HDC hdc = GetDC(hwnd);

// 2. 设置像素格式
PIXELFORMATDESCRIPTOR pfd = { ... };
int format = ChoosePixelFormat(hdc, &pfd);
SetPixelFormat(hdc, format, &pfd);

// 3. 创建 GL 上下文
HGLRC hglrc = wglCreateContext(hdc);
wglMakeCurrent(hdc, hglrc);  // 绑定到当前线程

// 现在可以调用 GL 函数了
glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
glClear(GL_COLOR_BUFFER_BIT);
SwapBuffers(hdc);  // 双缓冲交换
```

> **这就是为什么 KrKr2 使用 Cocos2d-x**——Cocos2d-x 封装了所有平台的窗口创建和 GL 上下文管理，开发者不需要写平台特定代码。`AppDelegate.cpp` 中的 `applicationDidFinishLaunching` 调用 `Director::getInstance()` 时，Cocos2d-x 已经完成了 GL 上下文的创建。

### GL 上下文与线程

**关键规则：一个 GL 上下文在同一时刻只能绑定到一个线程。**

```
线程 A: wglMakeCurrent(hdc, context) → glDraw*() ✅
线程 B: glDraw*() → 没有绑定上下文 → 崩溃或未定义行为 ❌

// 如果需要在线程 B 使用 GL：
线程 A: wglMakeCurrent(hdc, nullptr)     // 先解绑
线程 B: wglMakeCurrent(hdc, context)     // 再绑定到 B
```

大多数游戏引擎（包括 Cocos2d-x）的解决方案是：**所有 GL 调用都在主线程（渲染线程）执行**，其他线程通过消息队列提交渲染命令。

## OpenGL 版本简史

| 版本 | 年份 | 关键特性 | KrKr2 相关性 |
|------|------|---------|-------------|
| OpenGL 1.0 | 1992 | 固定管线 | ❌ 已废弃 |
| OpenGL 2.0 | 2004 | 可编程着色器（GLSL 1.10） | ⚠️ 最低兼容版本 |
| OpenGL 3.3 | 2010 | Core Profile, VAO 必须 | ✅ 桌面端常用基线 |
| OpenGL 4.6 | 2017 | SPIR-V, 计算着色器 | ❌ 超出需求 |
| OpenGL ES 2.0 | 2007 | 移动端可编程管线 | ✅ Android 基线 |
| OpenGL ES 3.0 | 2012 | FBO, 实例化, 3D 纹理 | ✅ 现代移动端 |

> **KrKr2 的选择**：通过 Cocos2d-x 间接使用 OpenGL，桌面端使用 OpenGL 2.1+，移动端使用 OpenGL ES 2.0+。Cocos2d-x 的渲染器自动处理版本差异。

## 常见错误与排查

### 错误 1：GL 函数调用返回全黑/什么都不显示

**可能原因**：
1. 没有调用 `glClear` 清除帧缓冲
2. 顶点坐标超出 NDC 范围 [-1, 1]，被裁剪掉
3. 投影矩阵不正确
4. 着色器编译失败但没检查错误日志

**排查方法**：
```cpp
// 始终检查着色器编译状态
GLint success;
glGetShaderiv(shader, GL_COMPILE_STATUS, &success);
if (!success) {
    char log[512];
    glGetShaderInfoLog(shader, 512, nullptr, log);
    printf("着色器编译失败: %s\n", log);
}
```

### 错误 2：glGetError 返回 GL_INVALID_OPERATION

**常见原因**：在没有 GL 上下文的线程调用 GL 函数。

```cpp
// 错误：在工作线程直接调用 GL
std::thread worker([]() {
    glGenTextures(1, &tex);  // ❌ 此线程没有 GL 上下文
});

// 正确：在渲染线程执行 GL 操作
void OnRenderThread() {
    glGenTextures(1, &tex);  // ✅ 渲染线程有 GL 上下文
}
```

### 错误 3：移动端显示异常但桌面正常

**常见原因**：OpenGL ES 不支持某些桌面 GL 特性。

| 特性 | 桌面 GL | ES 2.0 | ES 3.0 |
|------|---------|--------|--------|
| `GL_QUADS` 图元 | ✅ | ❌ | ❌ |
| 整数属性 `glVertexAttribI*` | ✅ | ❌ | ✅ |
| `glTexImage2D` 内部格式 `GL_RGBA` | ✅ | 必须精确匹配 | ✅ |
| `highp` 精度 | 总是可用 | 片段着色器中可能不支持 | ✅ |

## 动手实践

完成以下思维练习（不需要写代码，理解概念即可）：

### 练习 1：追踪一个像素

假设 KrKr2 正在显示一个视觉小说场景，屏幕左半边是背景图，右半边叠加了一个半透明的角色立绘。追踪屏幕正中间某个像素的渲染路径：

1. 哪些顶点数据会影响这个像素？（背景矩形和立绘矩形）
2. 顶点着色器做了什么？（坐标变换）
3. 光栅化时这个像素属于哪些三角形？（可能同时属于背景和立绘的三角形）
4. 片段着色器执行了几次？（两次——背景一次，立绘一次）
5. Alpha 混合如何决定最终颜色？（先画背景，再用立绘的 Alpha 值混合到背景上）

### 练习 2：画出你最熟悉的应用的渲染管线

选择一个你日常使用的应用（浏览器、游戏、文本编辑器），思考它的界面是如何渲染的：
- 哪些元素需要 GPU 渲染？
- 哪些可以用软件渲染？
- 为什么现代 UI 框架（Chrome、Flutter）都转向 GPU 渲染？

## 对照项目源码

| 概念 | KrKr2 对应代码 | 说明 |
|------|---------------|------|
| GL 上下文创建 | `cpp/core/environ/cocos2d/AppDelegate.cpp` 第 36-50 行 | Cocos2d-x Director 初始化时创建 GLView |
| 渲染循环 | `cpp/core/environ/cocos2d/MainScene.cpp` | Cocos2d-x Scene 的 `update` 方法驱动每帧渲染 |
| 纹理加载 | `cpp/core/visual/LoadPNG.cpp`, `LoadJPEG.cpp` 等 | 图像解码后上传到 GPU 纹理 |
| 2D 精灵渲染 | Cocos2d-x `Sprite` 类 | 封装了 VBO/VAO/着色器/纹理绑定 |
| Alpha 混合 | `cpp/core/visual/` 的图层系统 | 多图层叠加使用 Alpha 混合 |

## 本节小结

- **GPU 是专为并行计算设计的硬件**，数千个核心同时处理像素，适合图形渲染
- **渲染管线** 是从顶点数据到屏幕像素的固定流程：顶点获取 → 顶点着色器 → 图元装配 → 光栅化 → 片段着色器 → 逐片段操作 → 帧缓冲
- **顶点着色器和片段着色器**是可编程阶段，用 GLSL 编写，分别处理顶点变换和像素着色
- **OpenGL 是状态机**：先设置状态（绑定纹理、VAO、着色器），再执行绘制
- **GL 上下文**由窗口系统创建，绑定到单个线程；KrKr2 通过 Cocos2d-x 封装了这些平台差异
- **2D 游戏渲染**的核心是：矩形（两个三角形）+ 纹理采样 + Alpha 混合

## 练习题与答案

### 题目 1：为什么 GPU 有数千个核心但单个核心性能比 CPU 弱？这种设计取舍对图形渲染有什么好处？

<details>
<summary>查看答案</summary>

GPU 的设计目标是**吞吐量**（Throughput）而非**延迟**（Latency）。

CPU 核心为了降低单任务延迟，配备了深流水线、分支预测、乱序执行、大容量缓存等复杂机制，这些都需要大量晶体管。因此在相同芯片面积下，CPU 只能放置少量核心。

GPU 核心放弃了这些复杂机制，只保留基本的算术逻辑单元（ALU），因此同样面积可以放置数千个核心。虽然单个核心处理单个任务更慢，但数千个核心同时工作，**总吞吐量**远超 CPU。

对图形渲染的好处：屏幕上有数百万个像素，每个像素的着色计算是**独立的**（不依赖其他像素的结果）。这种"尴尬并行"（Embarrassingly Parallel）的特点完美匹配了 GPU 的多核架构。CPU 串行处理 200 万像素需要几十毫秒，GPU 并行处理只需要几毫秒。

</details>

### 题目 2：解释 OpenGL 渲染管线中哪些阶段是可编程的，哪些是固定功能的。为什么光栅化阶段不是可编程的？

<details>
<summary>查看答案</summary>

**可编程阶段**（用 GLSL 着色器编写）：
- 顶点着色器（Vertex Shader）—— 必须提供
- 片段着色器（Fragment Shader）—— 必须提供
- 几何着色器（Geometry Shader）—— 可选（OpenGL 3.2+）
- 曲面细分着色器（Tessellation Shader）—— 可选（OpenGL 4.0+）
- 计算着色器（Compute Shader）—— 可选（OpenGL 4.3+）

**固定功能阶段**（GPU 硬件执行，不可编程）：
- 顶点获取（Vertex Fetch）
- 图元装配（Primitive Assembly）
- 光栅化（Rasterization）
- 逐片段操作（深度测试、模板测试、Alpha 混合）

**光栅化不可编程的原因**：光栅化的任务是确定三角形覆盖了哪些像素位置，这是一个纯几何计算，算法（扫描线/边函数/层次化覆盖测试）高度优化且固定。将其硬件化可以获得极高的执行效率和确定性结果。如果允许可编程光栅化，一方面会引入巨大的性能开销（光栅化频率极高），另一方面在实践中很少需要自定义光栅化行为。需要自定义效果（如非光真实渲染）通常在片段着色器阶段就可以实现。

不过值得注意的是，现代 GPU（如 NVIDIA Turing 架构的 Mesh Shaders）正在逐步打破固定管线的边界，但这超出了 OpenGL 的范畴。

</details>

### 题目 3：KrKr2 的渲染场景中，一帧画面包含：背景图（不透明）、3 个角色立绘（半透明）、文字层（半透明背景框 + 文字）。请计算至少需要多少次 draw call，并解释绘制顺序为什么重要。

<details>
<summary>查看答案</summary>

**最少 draw call 次数**：

假设每个元素使用不同的纹理：
1. 背景图：1 次 draw call
2. 角色立绘 A：1 次 draw call
3. 角色立绘 B：1 次 draw call
4. 角色立绘 C：1 次 draw call
5. 文字背景框：1 次 draw call
6. 文字内容：1 次 draw call

**最少 6 次 draw call**。

如果使用了纹理图集（Texture Atlas）将多个立绘合并到一张纹理，3 个立绘可以合并为 1 次 draw call，最少降至 4 次。Cocos2d-x 的 SpriteBatchNode 正是这种优化。

**绘制顺序为什么重要**：

Alpha 混合不满足交换律。混合公式是：

```
result = src_color × src_alpha + dst_color × (1 - src_alpha)
```

其中 `dst_color` 是帧缓冲中已有的颜色。因此：
- 必须**先画不透明物体**（背景），再画半透明物体（立绘、文字）
- 半透明物体之间需要**从后往前**绘制（距离摄像机远的先画）
- 如果顺序反了，例如先画半透明立绘再画背景，立绘的颜色会被背景完全覆盖

正确顺序：`背景 → 最远的立绘 → 中间立绘 → 最近的立绘 → 文字背景框 → 文字`

KrKr2 的图层系统（Layer）通过 Z-order 管理绘制顺序，确保从后往前正确渲染。

</details>

## 下一步

下一节 [OpenGL 基础](../02-OpenGL基础/01-OpenGL基础.md) 将动手编写第一个 OpenGL 程序——创建窗口、编译着色器、上传顶点数据、绘制三角形。你将从"理论理解"进入"代码实践"。
