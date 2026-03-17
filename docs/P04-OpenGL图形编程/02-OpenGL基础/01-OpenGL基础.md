# OpenGL 基础绑定与绘制

> **所属模块：** P04-OpenGL图形编程
> **前置知识：** [P04/01-GPU渲染管线概述](../01-GPU渲染管线概述/01-GPU渲染管线概述.md)、C++ 基础、基本线性代数概念
> **预计阅读时间：** 40 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 OpenGL 缓冲对象（Buffer Object）的概念与作用
2. 掌握 VAO（顶点数组对象）、VBO（顶点缓冲对象）、EBO（元素缓冲对象）的创建与使用
3. 编写完整的 OpenGL 绘制流程：从数据准备到屏幕显示
4. 理解着色器程序的编译、链接与使用基础
5. 识别并解决 OpenGL 初学者常见的绘制问题

## 缓冲对象概述

在现代 OpenGL（3.0+）中，所有顶点数据必须通过**缓冲对象**（Buffer Object）从 CPU 传输到 GPU。缓冲对象是 GPU 显存中的一块内存区域，由驱动程序管理。

### 为什么需要缓冲对象？

早期 OpenGL（1.x/2.x）使用**立即模式**（Immediate Mode），每帧通过 `glBegin`/`glEnd` 逐顶点发送数据：

```cpp
// ❌ 已废弃的立即模式（仅供理解历史）
glBegin(GL_TRIANGLES);
    glColor3f(1.0f, 0.0f, 0.0f);   // 设置颜色为红色
    glVertex3f(-0.5f, -0.5f, 0.0f); // 左下角顶点
    glColor3f(0.0f, 1.0f, 0.0f);   // 设置颜色为绿色
    glVertex3f( 0.5f, -0.5f, 0.0f); // 右下角顶点
    glColor3f(0.0f, 0.0f, 1.0f);   // 设置颜色为蓝色
    glVertex3f( 0.0f,  0.5f, 0.0f); // 顶部顶点
glEnd();
```

这种方式每帧都要通过 CPU→GPU 总线传输所有顶点数据，效率极低。现代 OpenGL 的缓冲对象方案是：**一次性将数据上传到 GPU 显存，后续绘制直接从显存读取**，避免了反复的数据传输。

| 特性 | 立即模式 | 缓冲对象 |
|------|---------|----------|
| 数据存储位置 | CPU 内存 | GPU 显存 |
| 每帧数据传输 | 全部重传 | 仅修改部分 |
| API 复杂度 | 简单但低效 | 稍复杂但高效 |
| OpenGL ES 支持 | ❌ 不支持 | ✅ 唯一方式 |
| 适用场景 | 学习/原型 | 生产环境 |

> **重要提示：** OpenGL ES（Android/iOS）从未支持立即模式，从 ES 2.0 开始就只有缓冲对象方式。KrKr2 作为跨平台项目，所有渲染代码都使用缓冲对象。

### 缓冲对象的类型

OpenGL 中常用的缓冲对象类型：

| 缓冲类型 | 绑定目标 | 用途 |
|----------|---------|------|
| 顶点缓冲对象（VBO） | `GL_ARRAY_BUFFER` | 存储顶点属性数据（位置、颜色、纹理坐标等） |
| 元素缓冲对象（EBO/IBO） | `GL_ELEMENT_ARRAY_BUFFER` | 存储顶点索引，实现索引绘制 |
| 帧缓冲对象（FBO） | `GL_FRAMEBUFFER` | 离屏渲染目标（第 5 章详述） |
| 渲染缓冲对象（RBO） | `GL_RENDERBUFFER` | FBO 的深度/模板附件 |

## VBO — 顶点缓冲对象

VBO（Vertex Buffer Object）是最基础的缓冲对象，用来存储顶点的各种属性数据。

### 创建与填充 VBO

VBO 的生命周期：**生成 → 绑定 → 填充 → 使用 → 删除**。

```cpp
// === 完整示例：创建一个三角形的 VBO ===

// 定义三角形的 3 个顶点，每个顶点包含位置(x,y,z)和颜色(r,g,b)
// 交错布局：[位置x, 位置y, 位置z, 颜色r, 颜色g, 颜色b] × 3 个顶点
float vertices[] = {
    // --- 位置 ---      --- 颜色 ---
    -0.5f, -0.5f, 0.0f,  1.0f, 0.0f, 0.0f,  // 左下角：红色
     0.5f, -0.5f, 0.0f,  0.0f, 1.0f, 0.0f,  // 右下角：绿色
     0.0f,  0.5f, 0.0f,  0.0f, 0.0f, 1.0f   // 顶部：蓝色
};

GLuint vbo;
glGenBuffers(1, &vbo);              // 第1步：生成缓冲对象，获得ID
glBindBuffer(GL_ARRAY_BUFFER, vbo); // 第2步：绑定到 GL_ARRAY_BUFFER 目标

// 第3步：将顶点数据从 CPU 复制到 GPU 显存
glBufferData(
    GL_ARRAY_BUFFER,      // 绑定目标
    sizeof(vertices),     // 数据大小（字节）
    vertices,             // 数据指针
    GL_STATIC_DRAW        // 使用模式提示
);
```

### glBufferData 的使用模式

第四个参数 `usage` 是对驱动程序的性能提示，帮助它决定内存分配策略：

| 使用模式 | 数据修改频率 | 典型场景 |
|---------|-------------|---------|
| `GL_STATIC_DRAW` | 设置一次，不再修改 | 静态网格、地形 |
| `GL_DYNAMIC_DRAW` | 经常修改 | 粒子系统、动画顶点 |
| `GL_STREAM_DRAW` | 每帧都替换 | 即时生成的几何体 |

> **KrKr2 实践：** 在 `YUVSprite.cpp` 第 73-76 行中，视频精灵的 VBO 使用 `GL_DYNAMIC_DRAW`，因为视频帧的顶点数据需要频繁更新。

### 数据布局：交错 vs 分离

顶点数据在内存中有两种组织方式：

```
交错布局（Interleaved）— 推荐：
[位置0][颜色0][UV0] [位置1][颜色1][UV1] [位置2][颜色2][UV2]

分离布局（Separate）：
[位置0][位置1][位置2] [颜色0][颜色1][颜色2] [UV0][UV1][UV2]
```

交错布局的优势在于**缓存友好性**：同一个顶点的所有属性在内存中连续存放，GPU 读取时缓存命中率更高。现代引擎（包括 KrKr2 使用的 Cocos2d-x）几乎都使用交错布局。

## VAO — 顶点数组对象

VAO（Vertex Array Object）是 OpenGL 3.0 引入的核心概念。它**不存储**顶点数据本身，而是**记录**顶点属性的配置状态。

### 为什么需要 VAO？

没有 VAO 时，每次绘制前都需要重新配置所有顶点属性指针。VAO 的作用是：**将一组顶点属性配置打包保存，绘制时只需绑定 VAO 即可恢复所有配置**。

```
不使用 VAO 的绘制流程（每帧重复）：
  绑定 VBO → 设置属性指针0 → 启用属性0
            → 设置属性指针1 → 启用属性1
            → 绘制
            → 设置属性指针0 → 启用属性0（另一个物体）
            → ...

使用 VAO 的绘制流程：
  初始化时：创建 VAO → 绑定 VAO → 配置所有属性 → 解绑
  每帧绘制：绑定 VAO → 绘制 → 绑定另一个 VAO → 绘制
```

### 创建与使用 VAO

```cpp
// === 完整示例：VAO + VBO 配合使用 ===

GLuint vao, vbo;

// 创建 VAO
glGenVertexArrays(1, &vao);  // 生成 VAO
glBindVertexArray(vao);       // 绑定 VAO — 之后的所有属性配置都会记录到这个 VAO

// 创建并填充 VBO（同上例）
glGenBuffers(1, &vbo);
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

// 配置顶点属性指针
// 属性 0：位置（location = 0）
glVertexAttribPointer(
    0,                  // 属性索引（对应着色器中 layout(location = 0)）
    3,                  // 分量数（vec3 = 3 个 float）
    GL_FLOAT,           // 数据类型
    GL_FALSE,           // 是否归一化
    6 * sizeof(float),  // 步长：每个顶点占 6 个 float（位置3 + 颜色3）
    (void*)0            // 偏移：位置数据从头开始
);
glEnableVertexAttribArray(0); // 启用属性 0

// 属性 1：颜色（location = 1）
glVertexAttribPointer(
    1,                          // 属性索引
    3,                          // 分量数（vec3）
    GL_FLOAT,                   // 数据类型
    GL_FALSE,                   // 不归一化
    6 * sizeof(float),          // 步长同上
    (void*)(3 * sizeof(float))  // 偏移：颜色数据从第 4 个 float 开始
);
glEnableVertexAttribArray(1); // 启用属性 1

// 解绑 VAO（可选，防止后续误修改）
glBindVertexArray(0);
```

### glVertexAttribPointer 参数详解

这是 OpenGL 中最容易出错的函数之一，必须理解每个参数的含义：

```
glVertexAttribPointer(index, size, type, normalized, stride, offset)
                        │      │     │       │          │       │
                        │      │     │       │          │       └─ 属性在顶点结构中的字节偏移
                        │      │     │       │          └──── 相邻顶点之间的字节间距（0=紧密排列）
                        │      │     │       └───── 整数类型是否映射到 [0,1] 或 [-1,1]
                        │      │     └──────── 每个分量的数据类型
                        │      └────────── 每个顶点该属性的分量数（1-4）
                        └───────────── 属性位置索引
```

**stride（步长）计算方法：**
- 交错布局：stride = 一个完整顶点的总字节数
- 紧密排列（同一属性连续存放）：stride = 0（OpenGL 自动计算）

**offset（偏移）计算方法：**
- 该属性在一个顶点结构中的起始字节位置
- 必须强制转换为 `(void*)` 类型（历史遗留 API 设计）

> **KrKr2 实践：** 在 `YUVSprite.cpp` 第 80-97 行中，KrKr2 使用 Cocos2d-x 的 `V3F_C4B_T2F` 结构体（包含顶点位置 Vec3、颜色 Color4B、纹理坐标 Tex2F），通过 `offsetof` 宏计算每个属性的偏移量：
> ```cpp
> // 位置属性：offset = V3F_C4B_T2F::vertices 的偏移
> glVertexAttribPointer(VERTEX_ATTRIB_POSITION, 3, GL_FLOAT, GL_FALSE,
>     sizeof(V3F_C4B_T2F), (GLvoid*)offsetof(V3F_C4B_T2F, vertices));
> // 颜色属性：offset = V3F_C4B_T2F::colors 的偏移
> glVertexAttribPointer(VERTEX_ATTRIB_COLOR, 4, GL_UNSIGNED_BYTE, GL_TRUE,
>     sizeof(V3F_C4B_T2F), (GLvoid*)offsetof(V3F_C4B_T2F, colors));
> ```

### OpenGL ES 2.0 中的 VAO

OpenGL ES 2.0 **不原生支持 VAO**（ES 3.0 才引入）。在 ES 2.0 上需要通过扩展 `GL_OES_vertex_array_object` 使用，或者每次绘制时手动重新配置属性指针。

```cpp
// 跨平台 VAO 用法示例
#ifdef GL_ES_VERSION_3_0
    // ES 3.0+：原生支持
    glGenVertexArrays(1, &vao);
    glBindVertexArray(vao);
#elif defined(GL_OES_vertex_array_object)
    // ES 2.0 扩展方式
    glGenVertexArraysOES(1, &vao);
    glBindVertexArrayOES(vao);
#else
    // 无 VAO 支持：每次绘制前手动配置属性
    // KrKr2 的 YUVSprite 在不使用 VAO 时采用此方案
#endif
```

> **KrKr2 实践：** `YUVSprite.cpp` 的 `setupVBOAndVAO()` 函数被 `#ifdef USE_VAO` 包裹，默认不使用 VAO。在 `onDraw()` 方法中（第 230-256 行），当 VAO 不可用时，每次绘制都手动调用 `glVertexAttribPointer` 设置属性。

## EBO — 元素缓冲对象

EBO（Element Buffer Object），也叫 IBO（Index Buffer Object），用来存储顶点**索引**。它的核心作用是**避免重复存储共享顶点**。

### 索引绘制 vs 非索引绘制

绘制一个矩形需要 2 个三角形（6 个顶点），但其中 2 个顶点是重复的：

```
非索引绘制（6 个顶点 = 6 × 6 float = 144 字节）：
顶点0 → 顶点1 → 顶点2（三角形1）
顶点3 → 顶点4 → 顶点5（三角形2）
其中顶点0=顶点3, 顶点2=顶点4（重复数据！）

索引绘制（4 个顶点 + 6 个索引 = 4×6×4 + 6×2 = 108 字节）：
顶点: [0, 1, 2, 3]（仅 4 个唯一顶点）
索引: [0, 1, 2, 2, 1, 3]（指定三角形组装顺序）
```

对于复杂模型，索引绘制可以节省 **40-60%** 的显存占用。

### 创建与使用 EBO

```cpp
// === 完整示例：用 EBO 绘制矩形 ===

// 4 个唯一顶点（矩形的 4 个角）
float vertices[] = {
    // --- 位置 ---        --- 颜色 ---
     0.5f,  0.5f, 0.0f,   1.0f, 0.0f, 0.0f,  // 右上角：红
     0.5f, -0.5f, 0.0f,   0.0f, 1.0f, 0.0f,  // 右下角：绿
    -0.5f, -0.5f, 0.0f,   0.0f, 0.0f, 1.0f,  // 左下角：蓝
    -0.5f,  0.5f, 0.0f,   1.0f, 1.0f, 0.0f   // 左上角：黄
};

// 6 个索引，定义 2 个三角形
unsigned int indices[] = {
    0, 1, 3,  // 第一个三角形：右上→右下→左上
    1, 2, 3   // 第二个三角形：右下→左下→左上
};

GLuint vao, vbo, ebo;

// 创建 VAO
glGenVertexArrays(1, &vao);
glBindVertexArray(vao);

// 创建 VBO
glGenBuffers(1, &vbo);
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

// 创建 EBO — 必须在 VAO 绑定的状态下创建
glGenBuffers(1, &ebo);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);

// 配置顶点属性（同前例）
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float),
    (void*)(3 * sizeof(float)));
glEnableVertexAttribArray(1);

// 解绑 VAO
glBindVertexArray(0);
// 注意：不要在 VAO 绑定时解绑 EBO！EBO 的绑定状态保存在 VAO 中
```

> **关键陷阱：** 不要在 VAO 绑定时调用 `glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0)` 来解绑 EBO。EBO 的绑定存储在 VAO 状态中，解绑会导致 VAO 丢失索引信息。VBO 则可以安全解绑（属性指针已记录了引用）。

## 绘制命令

OpenGL 提供两类绘制命令：**非索引绘制**和**索引绘制**。

### glDrawArrays — 非索引绘制

```cpp
// 语法：glDrawArrays(图元类型, 起始顶点索引, 顶点数量)

glBindVertexArray(vao);  // 绑定 VAO

// 绘制三角形：从第 0 个顶点开始，使用 3 个顶点
glDrawArrays(GL_TRIANGLES, 0, 3);

// 绘制三角形带：从第 0 个顶点开始，使用 4 个顶点
glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
```

> **KrKr2 实践：** `YUVSprite.cpp` 第 256 行使用 `glDrawArrays(GL_TRIANGLE_STRIP, 0, 4)` 绘制视频帧。三角形带用 4 个顶点形成一个矩形，比 `GL_TRIANGLES` 需要的 6 个顶点更高效。

### glDrawElements — 索引绘制

```cpp
// 语法：glDrawElements(图元类型, 索引数量, 索引类型, 索引偏移)

glBindVertexArray(vao);  // 绑定 VAO（包含 EBO 绑定信息）

// 绘制矩形：2 个三角形 = 6 个索引
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
```

> **KrKr2 实践：** `YUVSprite.cpp` 第 224 行在 VAO 模式下使用 `glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_SHORT, nullptr)` 进行索引绘制。注意索引类型是 `GL_UNSIGNED_SHORT`（16位无符号），在移动设备上比 `GL_UNSIGNED_INT` 更节省内存。

### 图元类型一览

```
GL_POINTS           ●  ●  ●              每个顶点绘制一个点
GL_LINES            ●──●  ●──●           每 2 个顶点绘制一条线段
GL_LINE_STRIP       ●──●──●──●           连续线段
GL_LINE_LOOP        ●──●──●──●──●        首尾相连的线段
GL_TRIANGLES        ▲  ▲                 每 3 个顶点绘制一个三角形
GL_TRIANGLE_STRIP   ▲▲▲                  三角形带（共享边）
GL_TRIANGLE_FAN     ▲▲▲                  三角形扇（共享中心点）
```

2D 游戏引擎中最常用的是 `GL_TRIANGLES` 和 `GL_TRIANGLE_STRIP`。

## 着色器程序基础

在现代 OpenGL 中，必须提供**着色器程序**（Shader Program）才能绘制任何东西。着色器是运行在 GPU 上的小程序，至少需要**顶点着色器**和**片段着色器**两个阶段。

### 最小着色器对

```glsl
// === 顶点着色器（vertex shader） ===
#version 330 core                    // OpenGL 3.3 核心模式
layout (location = 0) in vec3 aPos;  // 从 VBO 读取位置属性
layout (location = 1) in vec3 aColor;// 从 VBO 读取颜色属性
out vec3 vertexColor;                // 传递给片段着色器

void main() {
    gl_Position = vec4(aPos, 1.0);   // 输出裁剪空间坐标
    vertexColor = aColor;            // 传递颜色
}
```

```glsl
// === 片段着色器（fragment shader） ===
#version 330 core
in vec3 vertexColor;                 // 从顶点着色器接收
out vec4 FragColor;                  // 输出最终颜色

void main() {
    FragColor = vec4(vertexColor, 1.0); // 使用插值后的颜色
}
```

### 编译与链接着色器

```cpp
// === 完整示例：编译并链接着色器程序 ===

// 顶点着色器源码
const char* vertexShaderSource = R"(
    #version 330 core
    layout (location = 0) in vec3 aPos;
    layout (location = 1) in vec3 aColor;
    out vec3 vertexColor;
    void main() {
        gl_Position = vec4(aPos, 1.0);
        vertexColor = aColor;
    }
)";

// 片段着色器源码
const char* fragmentShaderSource = R"(
    #version 330 core
    in vec3 vertexColor;
    out vec4 FragColor;
    void main() {
        FragColor = vec4(vertexColor, 1.0);
    }
)";

// 编译顶点着色器
GLuint vertexShader = glCreateShader(GL_VERTEX_SHADER);  // 创建着色器对象
glShaderSource(vertexShader, 1, &vertexShaderSource, nullptr); // 设置源码
glCompileShader(vertexShader);                            // 编译

// 检查编译结果
GLint success;
glGetShaderiv(vertexShader, GL_COMPILE_STATUS, &success);
if (!success) {
    char infoLog[512];
    glGetShaderInfoLog(vertexShader, 512, nullptr, infoLog); // 获取错误信息
    std::cerr << "顶点着色器编译失败：\n" << infoLog << std::endl;
}

// 编译片段着色器（流程相同）
GLuint fragmentShader = glCreateShader(GL_FRAGMENT_SHADER);
glShaderSource(fragmentShader, 1, &fragmentShaderSource, nullptr);
glCompileShader(fragmentShader);
// ... 同样检查编译结果 ...

// 链接着色器程序
GLuint shaderProgram = glCreateProgram();           // 创建程序对象
glAttachShader(shaderProgram, vertexShader);         // 附加顶点着色器
glAttachShader(shaderProgram, fragmentShader);       // 附加片段着色器
glLinkProgram(shaderProgram);                        // 链接

// 检查链接结果
glGetProgramiv(shaderProgram, GL_LINK_STATUS, &success);
if (!success) {
    char infoLog[512];
    glGetProgramInfoLog(shaderProgram, 512, nullptr, infoLog);
    std::cerr << "着色器程序链接失败：\n" << infoLog << std::endl;
}

// 链接成功后，着色器对象可以删除（已复制到程序中）
glDeleteShader(vertexShader);
glDeleteShader(fragmentShader);
```

> **KrKr2 实践：** `YUVSprite.cpp` 第 58 行通过 Cocos2d-x 的 `GLProgram::createWithByteArrays()` 封装了上述编译链接流程。传入顶点着色器源码和片段着色器源码，返回编译好的 GLProgram 对象。

### 使用着色器程序绘制

```cpp
// 完整的绘制循环
glUseProgram(shaderProgram);   // 激活着色器程序
glBindVertexArray(vao);        // 绑定 VAO
glDrawArrays(GL_TRIANGLES, 0, 3); // 绘制
// 或索引绘制：
// glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
```

## 动手实践

### 练习 1：绘制彩色三角形

将本节学到的 VBO、VAO、着色器程序组合起来，完成一个彩色三角形的绘制。完整步骤：

1. 初始化 OpenGL 上下文（使用 GLFW + GLAD，或 SDL2 + GLEW）
2. 编写并编译顶点着色器和片段着色器
3. 创建 VAO 和 VBO，配置顶点属性
4. 在渲染循环中绘制三角形

以下是使用 GLFW + GLAD 的完整可运行代码框架：

```cpp
// main.cpp — 最小 OpenGL 程序框架
// CMakeLists.txt 需要链接 glfw、glad（或通过 vcpkg 安装）
#include <glad/glad.h>
#include <GLFW/glfw3.h>
#include <iostream>

// 着色器源码（同上例）
const char* vertSrc = "#version 330 core\n"
    "layout (location = 0) in vec3 aPos;\n"
    "layout (location = 1) in vec3 aColor;\n"
    "out vec3 vColor;\n"
    "void main() { gl_Position = vec4(aPos, 1.0); vColor = aColor; }\n";

const char* fragSrc = "#version 330 core\n"
    "in vec3 vColor;\n"
    "out vec4 FragColor;\n"
    "void main() { FragColor = vec4(vColor, 1.0); }\n";

int main() {
    // 初始化 GLFW
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
#ifdef __APPLE__
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE); // macOS 必需
#endif

    GLFWwindow* window = glfwCreateWindow(800, 600, "OpenGL 三角形", nullptr, nullptr);
    glfwMakeContextCurrent(window);
    gladLoadGLLoader((GLADloadproc)glfwGetProcAddress); // 加载 GL 函数指针

    // 编译着色器（省略错误检查，完整版见上文）
    GLuint vs = glCreateShader(GL_VERTEX_SHADER);
    glShaderSource(vs, 1, &vertSrc, nullptr);
    glCompileShader(vs);
    GLuint fs = glCreateShader(GL_FRAGMENT_SHADER);
    glShaderSource(fs, 1, &fragSrc, nullptr);
    glCompileShader(fs);
    GLuint prog = glCreateProgram();
    glAttachShader(prog, vs);
    glAttachShader(prog, fs);
    glLinkProgram(prog);
    glDeleteShader(vs);
    glDeleteShader(fs);

    // 顶点数据
    float verts[] = {
        -0.5f, -0.5f, 0.0f,  1.0f, 0.0f, 0.0f,
         0.5f, -0.5f, 0.0f,  0.0f, 1.0f, 0.0f,
         0.0f,  0.5f, 0.0f,  0.0f, 0.0f, 1.0f
    };

    GLuint vao, vbo;
    glGenVertexArrays(1, &vao);
    glGenBuffers(1, &vbo);
    glBindVertexArray(vao);
    glBindBuffer(GL_ARRAY_BUFFER, vbo);
    glBufferData(GL_ARRAY_BUFFER, sizeof(verts), verts, GL_STATIC_DRAW);
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6*sizeof(float), (void*)0);
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 6*sizeof(float), (void*)(3*sizeof(float)));
    glEnableVertexAttribArray(1);
    glBindVertexArray(0);

    // 渲染循环
    while (!glfwWindowShouldClose(window)) {
        glClearColor(0.1f, 0.1f, 0.1f, 1.0f); // 深灰色背景
        glClear(GL_COLOR_BUFFER_BIT);
        glUseProgram(prog);
        glBindVertexArray(vao);
        glDrawArrays(GL_TRIANGLES, 0, 3);       // 绘制三角形
        glfwSwapBuffers(window);
        glfwPollEvents();
    }

    // 清理资源
    glDeleteVertexArrays(1, &vao);
    glDeleteBuffers(1, &vbo);
    glDeleteProgram(prog);
    glfwTerminate();
    return 0;
}
```

### 练习 2：用 EBO 绘制矩形

在练习 1 的基础上，添加 EBO 实现索引绘制，将三角形改为矩形。

## 对照项目源码

KrKr2 项目中 OpenGL 绑定和绘制相关代码主要集中在以下文件：

| 文件 | 行号范围 | 内容 |
|------|---------|------|
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 64-112 | VAO/VBO 创建（`setupVBOAndVAO`），展示了完整的 VAO+VBO+EBO 设置流程 |
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 200-257 | 两种绘制路径：VAO 模式（`glDrawElements`）和非 VAO 模式（`glDrawArrays`） |
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 230-256 | 非 VAO 路径：手动设置 `glVertexAttribPointer` + `glDrawArrays(GL_TRIANGLE_STRIP)` |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 862-1145 | `tTVPOGLTexture2D` 类：纹理对象管理、顶点信息结构（`GLVertexInfo`） |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 464-468 | `GLVertexInfo` 结构体：存储纹理指针和顶点坐标数组 |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 469-471 | FBO 管理：全局 `_FBO` 变量和 `_CurrentRenderTarget` |

**关键模式观察：**

1. **条件 VAO：** KrKr2 通过 `#ifdef USE_VAO` 控制是否使用 VAO，因为 OpenGL ES 2.0 设备可能不支持
2. **Cocos2d-x 状态缓存：** 使用 `cocos2d::GL::bindTexture2D()` 而非直接 `glBindTexture`，避免冗余状态切换
3. **三角形带优化：** 矩形绘制统一使用 `GL_TRIANGLE_STRIP`（4 顶点）而非 `GL_TRIANGLES`（6 顶点）

## 常见错误与排查

### 错误 1：绘制不出任何东西（黑屏）

**现象：** 调用了绘制命令但屏幕一片空白。

**排查清单：**

```cpp
// 1. 着色器编译/链接是否成功？
GLint success;
glGetProgramiv(program, GL_LINK_STATUS, &success);
assert(success && "着色器链接失败！");

// 2. 是否调用了 glUseProgram？
glUseProgram(shaderProgram); // 必须在绘制前激活

// 3. VAO 是否正确绑定？
glBindVertexArray(vao); // 必须在绘制前绑定

// 4. 顶点坐标是否在裁剪空间范围内 [-1, 1]？
// 如果坐标超出此范围，顶点会被裁剪掉

// 5. glClear 是否在绘制之前调用？
glClear(GL_COLOR_BUFFER_BIT); // 先清屏
// ... 然后绘制 ...

// 6. 是否调用了 SwapBuffers？
glfwSwapBuffers(window); // 双缓冲交换
```

### 错误 2：顶点属性数据错乱（颜色/位置不对）

**原因：** `glVertexAttribPointer` 的 stride 或 offset 参数计算错误。

**调试方法：** 画出内存布局图，手动计算每个属性的偏移量。

```
内存布局（交错，每顶点 6 个 float）：
字节偏移: 0    4    8    12   16   20   24
         [Px] [Py] [Pz] [Cr] [Cg] [Cb] [Px] ...
         |--- 位置 ---|  |--- 颜色 ---|
stride = 6 × 4 = 24 字节
位置 offset = 0
颜色 offset = 3 × 4 = 12 字节
```

### 错误 3：EBO 数据丢失导致绘制异常

**原因：** 在 VAO 绑定状态下解绑了 EBO。

```cpp
// ❌ 错误：在 VAO 激活时解绑 EBO
glBindVertexArray(vao);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0); // 这会把 VAO 中的 EBO 引用清除！

// ✅ 正确：先解绑 VAO，再解绑 EBO（或干脆不解绑 EBO）
glBindVertexArray(0);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0); // 此时不影响 VAO
```

## 本节小结

- **VBO** 是 GPU 显存中的数据存储区，用于存放顶点属性数据
- **VAO** 记录顶点属性的配置状态，避免每帧重复设置
- **EBO** 存储顶点索引，通过共享顶点减少数据冗余
- 现代 OpenGL 绘制的核心流程：`glUseProgram` → `glBindVertexArray` → `glDrawArrays`/`glDrawElements`
- 着色器程序必须经过 **编译 → 附加 → 链接** 三步才能使用
- OpenGL ES 2.0 不原生支持 VAO，需要扩展或手动配置属性
- `glVertexAttribPointer` 是最易出错的函数，必须准确计算 stride 和 offset
- KrKr2 使用条件编译控制 VAO 使用，并通过 Cocos2d-x 封装简化 GL 状态管理

## 练习题与答案

### 题目 1：stride 和 offset 计算

假设一个顶点包含以下属性：
- 位置：`vec3`（3 个 float）
- 法线：`vec3`（3 个 float）
- 纹理坐标：`vec2`（2 个 float）
- 颜色：`vec4`（4 个 float）

采用交错布局存储。请计算：
1. 每个顶点的 stride 是多少字节？
2. 每个属性的 offset 是多少字节？
3. 写出 4 个 `glVertexAttribPointer` 调用

<details>
<summary>查看答案</summary>

**stride 计算：**
一个顶点包含 3 + 3 + 2 + 4 = 12 个 float，stride = 12 × 4 = **48 字节**

**offset 计算：**
- 位置：offset = 0
- 法线：offset = 3 × 4 = 12 字节
- 纹理坐标：offset = (3 + 3) × 4 = 24 字节
- 颜色：offset = (3 + 3 + 2) × 4 = 32 字节

```cpp
GLsizei stride = 12 * sizeof(float); // 48 字节

// 位置属性 (location = 0)
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, stride, (void*)0);
glEnableVertexAttribArray(0);

// 法线属性 (location = 1)
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, stride, (void*)(3 * sizeof(float)));
glEnableVertexAttribArray(1);

// 纹理坐标属性 (location = 2)
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, stride, (void*)(6 * sizeof(float)));
glEnableVertexAttribArray(2);

// 颜色属性 (location = 3)
glVertexAttribPointer(3, 4, GL_FLOAT, GL_FALSE, stride, (void*)(8 * sizeof(float)));
glEnableVertexAttribArray(3);
```

</details>

### 题目 2：VAO 状态保存

以下代码存在一个关键错误，会导致 `glDrawElements` 绘制时崩溃或绘制不出任何东西。请找出错误并修复。

```cpp
GLuint vao, vbo, ebo;
glGenVertexArrays(1, &vao);
glBindVertexArray(vao);

glGenBuffers(1, &vbo);
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

glGenBuffers(1, &ebo);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);

glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3*sizeof(float), (void*)0);
glEnableVertexAttribArray(0);

// 解绑
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0); // ← 错误！
glBindVertexArray(0);
```

<details>
<summary>查看答案</summary>

**错误：** 在 VAO 仍然绑定时解绑了 EBO（`glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0)`）。

EBO 的绑定状态是 VAO 状态的一部分。在 VAO 绑定期间解绑 EBO，等于把 VAO 中记录的 EBO 引用清除了。后续 `glDrawElements` 时，VAO 中没有 EBO，无法读取索引数据。

**修复方法：** 先解绑 VAO，再解绑 EBO（或者不解绑 EBO）。

```cpp
// 正确顺序：
glBindVertexArray(0);                        // 先解绑 VAO
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);    // 然后可以安全解绑 EBO
glBindBuffer(GL_ARRAY_BUFFER, 0);            // VBO 随时可以解绑
```

注意：`GL_ARRAY_BUFFER` 的绑定不保存在 VAO 中（因为 `glVertexAttribPointer` 调用时已经记录了 VBO 引用），所以 VBO 可以在任何时候安全解绑。

</details>

### 题目 3：分析 KrKr2 绘制路径

阅读 `cpp/core/environ/cocos2d/YUVSprite.cpp` 的 `onDraw()` 方法（第 200-257 行），回答以下问题：
1. 该方法支持哪两种绘制路径？什么条件下使用哪种？
2. VAO 路径使用什么图元类型和绘制命令？
3. 非 VAO 路径使用什么图元类型和绘制命令？
4. 为什么非 VAO 路径需要创建局部数组（如 `vertices[4]`、`colors[4]`）？

<details>
<summary>查看答案</summary>

1. **两种路径：**
   - **VAO 路径**（`#ifdef USE_VAO` 内，第 202-228 行）：使用 VAO + VBO + EBO 的标准现代 GL 流程
   - **非 VAO 路径**（`#ifdef` 之后，第 230-257 行）：每帧手动设置顶点属性指针，直接传入客户端数组

   使用哪种取决于编译时宏 `USE_VAO` 是否定义。默认未定义（注释掉了 `#define USE_VAO`），即默认使用非 VAO 路径。

2. **VAO 路径：** 使用 `GL_TRIANGLES` 图元，调用 `glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_SHORT, nullptr)`（索引绘制，6 个索引组成 2 个三角形）

3. **非 VAO 路径：** 使用 `GL_TRIANGLE_STRIP` 图元，调用 `glDrawArrays(GL_TRIANGLE_STRIP, 0, 4)`（非索引绘制，4 个顶点组成三角形带）

4. **为什么需要局部数组：** 非 VAO 路径直接使用 `glVertexAttribPointer` 指向客户端内存（CPU 端），不使用 GPU 端的 VBO。因此需要把顶点的各个属性（位置、颜色、纹理坐标）从 Cocos2d-x 的交错结构体 `V3F_C4B_T2F` 中提取出来，分别存放到独立的局部数组中，然后分别设置属性指针指向这些数组。这是因为没有 VBO 时无法使用 stride/offset 方式访问交错数据——`glVertexAttribPointer` 在没有 VBO 绑定时会把指针当作直接的客户端内存地址。

</details>

## 下一步

下一节 [着色器编程](../03-着色器编程/01-着色器编程.md) 将深入讲解 GLSL 着色器语言，包括数据类型、变量限定符、内置函数，以及如何编写高质量的跨平台着色器代码。
