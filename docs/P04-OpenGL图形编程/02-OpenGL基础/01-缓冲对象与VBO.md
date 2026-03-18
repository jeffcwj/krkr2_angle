# 缓冲对象与 VBO

> **所属模块：** P04-OpenGL图形编程  
> **前置知识：** [GPU架构与OpenGL简介](../01-GPU渲染管线概述/01-GPU架构与OpenGL简介.md)、[OpenGL渲染管线详解](../01-GPU渲染管线概述/02-OpenGL渲染管线详解.md)、C++ 基础  
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 OpenGL 缓冲对象（Buffer Object）的设计动机与核心概念
2. 掌握 VBO 的完整生命周期：生成、绑定、填充、使用、删除
3. 区分 `glBufferData` 与 `glBufferSubData` 的使用场景
4. 理解交错布局与分离布局的内存差异及性能影响
5. 使用 `glMapBuffer` / `glMapBufferRange` 实现零拷贝数据更新

---

## 场景导入：为什么 GPU 需要"专用仓库"？

想象你是一个画家。每天早上你要画一幅画，颜料和画布都放在隔壁仓库里。如果每画一笔都要跑去仓库拿颜料，效率极低。聪明的做法是：**一次性把今天要用的颜料全搬到画桌上**，然后随取随用。

GPU 渲染也是同样的道理：

- **CPU 内存**就是"隔壁仓库"——容量大但 GPU 访问慢
- **GPU 显存**就是"画桌"——容量有限但 GPU 读取极快
- **缓冲对象**就是搬运工具——把数据从 CPU 内存搬到 GPU 显存

在 OpenGL 中，**缓冲对象**（Buffer Object）是 GPU 显存中一块由驱动程序管理的内存区域。你把顶点数据上传到缓冲对象后，GPU 就能直接从自己的显存中读取数据，不再需要每帧通过 CPU→GPU 总线传输。

---

## 从立即模式到缓冲对象：一段历史

### 立即模式（已废弃）

在 OpenGL 1.x/2.x 时代，程序员使用**立即模式**（Immediate Mode）绘制图形。这种方式直观但效率极低——每个顶点都要通过一次 CPU→GPU 的函数调用传输：

```cpp
// ❌ 立即模式（OpenGL 1.x）——已废弃，仅供理解历史
// 每帧都要通过 CPU→GPU 总线逐顶点发送数据
glBegin(GL_TRIANGLES);
    glColor3f(1.0f, 0.0f, 0.0f);    // 发送颜色到 GPU（一次总线传输）
    glVertex3f(-0.5f, -0.5f, 0.0f); // 发送位置到 GPU（一次总线传输）
    glColor3f(0.0f, 1.0f, 0.0f);    // 又一次总线传输
    glVertex3f( 0.5f, -0.5f, 0.0f); // 又一次总线传输
    glColor3f(0.0f, 0.0f, 1.0f);    // 又一次总线传输
    glVertex3f( 0.0f,  0.5f, 0.0f); // 又一次总线传输
glEnd();
// 一个三角形 = 6 次函数调用 = 6 次 CPU→GPU 数据传输
// 一个有 10000 个三角形的场景 = 60000 次传输 = 卡顿
```

问题很明显：一个三角形就需要 6 次 CPU→GPU 通信。真实游戏场景有成千上万个三角形，每帧这样做会让 CPU→GPU 总线成为瓶颈。

### 缓冲对象方案

现代 OpenGL（3.0+）的解决方案是：**一次性把所有数据打包上传到 GPU 显存，后续绘制直接从显存读取**。

```cpp
// ✅ 现代方式：缓冲对象
// 初始化时（仅一次）：把数据上传到 GPU 显存
float vertices[] = { ... };              // CPU 端准备数据
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
// 一次批量传输，完成！

// 每帧绘制时（重复执行）：GPU 直接从自己的显存读取
glDrawArrays(GL_TRIANGLES, 0, 3);        // 不再有 CPU→GPU 数据传输
```

两种方式的性能对比：

| 特性 | 立即模式 | 缓冲对象 |
|------|---------|----------|
| 数据存储位置 | CPU 内存 | GPU 显存 |
| 每帧 CPU→GPU 传输 | 全部顶点数据 | 无（除非数据变化） |
| 10000 三角形开销 | 60000 次总线传输 | 1 次 `glDrawArrays` 调用 |
| OpenGL ES 支持 | ❌ 从未支持 | ✅ 唯一方式 |
| 适用场景 | 学习/原型（已废弃） | 所有生产环境 |

> **跨平台关键点：** OpenGL ES（Android/iOS）从 ES 2.0 开始就**只有**缓冲对象方式，从未支持立即模式。KrKr2 作为跨平台项目，所有渲染代码都使用缓冲对象。如果你之前学过 OpenGL 1.x 的立即模式，从现在开始请彻底忘记它。

---

## 缓冲对象的类型

OpenGL 支持多种缓冲对象，每种对应不同的绑定目标（Binding Target）。**绑定目标**告诉 OpenGL 这块缓冲用来做什么：

| 缓冲类型 | 绑定目标 | 用途 | 本教程覆盖位置 |
|----------|---------|------|----------------|
| 顶点缓冲对象（VBO） | `GL_ARRAY_BUFFER` | 存储顶点属性（位置、颜色、UV…） | **本节** |
| 元素缓冲对象（EBO/IBO） | `GL_ELEMENT_ARRAY_BUFFER` | 存储顶点索引 | [下一节](./02-VAO与EBO.md) |
| 帧缓冲对象（FBO） | `GL_FRAMEBUFFER` | 离屏渲染目标 | [第5章](../05-帧缓冲与后处理/01-帧缓冲基础与创建.md) |
| 渲染缓冲对象（RBO） | `GL_RENDERBUFFER` | FBO 的深度/模板附件 | [第5章](../05-帧缓冲与后处理/01-帧缓冲基础与创建.md) |
| Uniform 缓冲对象（UBO） | `GL_UNIFORM_BUFFER` | 共享 Uniform 数据（GL 3.1+） | [第3章](../03-着色器编程/02-Uniform与平台差异.md) |
| 像素缓冲对象（PBO） | `GL_PIXEL_PACK/UNPACK_BUFFER` | 异步像素传输 | [第4章](../04-纹理与采样/02-多纹理与像素传输.md) |

本节聚焦最基础也最常用的 **VBO（Vertex Buffer Object）**——顶点缓冲对象。

---

## VBO 的完整生命周期

VBO 的使用遵循严格的 5 步流程：**生成 → 绑定 → 填充 → 使用 → 删除**。把每一步想象成仓库管理：

```
生成(glGenBuffers)     → 向仓库管理员申请一个编号牌
绑定(glBindBuffer)     → 把编号牌挂到工作台上（"接下来的操作都针对这个编号"）
填充(glBufferData)     → 往这个编号对应的货架上放货物
使用(glDrawArrays)     → 工人从货架上取货加工
删除(glDeleteBuffers)  → 退还编号牌，清空货架
```

### 第 1 步：生成（glGenBuffers）

```cpp
GLuint vbo;               // 变量接收缓冲对象的"名字"（本质是无符号整数 ID）
glGenBuffers(1, &vbo);    // 向 OpenGL 申请 1 个缓冲对象名字
// 此时 vbo 只是一个"空名字"——对应的 GPU 显存还没有分配
// 类似 malloc 返回指针但还没写入数据

// 也可以一次申请多个：
GLuint buffers[3];
glGenBuffers(3, buffers);  // 一次性申请 3 个缓冲对象名字
```

> **底层细节：** `glGenBuffers` 只是分配了一个整数 ID（名字），并没有在 GPU 上分配任何内存。实际的 GPU 内存分配发生在第一次调用 `glBufferData` 时。这是 OpenGL 的惰性分配策略。

### 第 2 步：绑定（glBindBuffer）

OpenGL 是一个**状态机**——操作总是针对"当前绑定"的对象。绑定就是告诉 OpenGL："从现在起，所有对 `GL_ARRAY_BUFFER` 的操作，都针对这个 VBO"。

```cpp
glBindBuffer(GL_ARRAY_BUFFER, vbo);
// 从现在起：
//   glBufferData(GL_ARRAY_BUFFER, ...)   → 操作的是 vbo
//   glBufferSubData(GL_ARRAY_BUFFER, ...) → 操作的是 vbo
//   glVertexAttribPointer(...)            → 从 vbo 读取数据
```

绑定目标 `GL_ARRAY_BUFFER` 表示这是一个顶点属性缓冲。不同的绑定目标是独立的槽位——你可以同时绑定一个 VBO 到 `GL_ARRAY_BUFFER`，再绑定一个 EBO 到 `GL_ELEMENT_ARRAY_BUFFER`，它们互不干扰。

```
OpenGL 状态机绑定槽位：
┌──────────────────────────┐
│ GL_ARRAY_BUFFER    ← vbo │  ← 当前绑定的 VBO
│ GL_ELEMENT_..._BUF ← ebo │  ← 当前绑定的 EBO
│ GL_UNIFORM_BUFFER  ← ubo │  ← 当前绑定的 UBO
│ ...其他槽位...            │
└──────────────────────────┘
```

### 第 3 步：填充（glBufferData）

这是最关键的一步——把 CPU 内存中的顶点数据**复制**到 GPU 显存：

```cpp
// 定义三角形的 3 个顶点
// 每个顶点包含：位置(x,y,z) + 颜色(r,g,b) = 6 个 float
float vertices[] = {
    // --- 位置 ---        --- 颜色 ---
    -0.5f, -0.5f, 0.0f,   1.0f, 0.0f, 0.0f,  // 顶点0：左下角，红色
     0.5f, -0.5f, 0.0f,   0.0f, 1.0f, 0.0f,  // 顶点1：右下角，绿色
     0.0f,  0.5f, 0.0f,   0.0f, 0.0f, 1.0f   // 顶点2：顶部，蓝色
};

glBufferData(
    GL_ARRAY_BUFFER,      // 绑定目标：操作当前绑定到此目标的缓冲
    sizeof(vertices),     // 数据大小：18 × 4 = 72 字节
    vertices,             // 数据源指针：CPU 端数组
    GL_STATIC_DRAW        // 使用模式提示（后文详解）
);
```

**glBufferData 做了三件事：**
1. 在 GPU 显存中**分配**一块指定大小的内存
2. 将 CPU 端的数据**复制**到这块 GPU 内存
3. 如果该缓冲之前已有数据，**先释放旧内存**再分配新的

> **注意：** 每次调用 `glBufferData` 都会重新分配 GPU 内存。如果你只想更新部分数据而不重新分配，应该用 `glBufferSubData`（后文详解）。

### 第 4 步：使用

数据上传后，通过 `glVertexAttribPointer` 告诉 GPU 如何解读这些数据，然后调用绘制命令。这部分将在 [下一节 VAO 与 EBO](./02-VAO与EBO.md) 中详细讲解。

### 第 5 步：删除（glDeleteBuffers）

当缓冲对象不再需要时（例如场景卸载），必须手动释放 GPU 内存：

```cpp
glDeleteBuffers(1, &vbo);   // 释放 GPU 显存，回收 ID
// 删除后 vbo 的值仍然是那个整数，但已经无效了
// 再次使用会产生 GL_INVALID_OPERATION 错误

// 批量删除：
glDeleteBuffers(3, buffers); // 一次释放 3 个缓冲对象
```

> **资源泄漏警告：** 如果你 `glGenBuffers` 了但没有 `glDeleteBuffers`，这块 GPU 显存就泄漏了。与 CPU 内存泄漏不同，GPU 显存泄漏更难检测（不会触发段错误），但会导致显存逐渐耗尽，最终引发渲染异常或崩溃。

### 完整生命周期示例

```cpp
// === 完整示例：VBO 的创建、使用与销毁 ===
#include <glad/glad.h>  // 或 <GLES3/gl3.h> (Android)

void setupTriangle(GLuint& vbo) {
    float vertices[] = {
        -0.5f, -0.5f, 0.0f,  1.0f, 0.0f, 0.0f,
         0.5f, -0.5f, 0.0f,  0.0f, 1.0f, 0.0f,
         0.0f,  0.5f, 0.0f,  0.0f, 0.0f, 1.0f
    };

    glGenBuffers(1, &vbo);                                    // 1. 生成
    glBindBuffer(GL_ARRAY_BUFFER, vbo);                       // 2. 绑定
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertices),           // 3. 填充
                 vertices, GL_STATIC_DRAW);
    glBindBuffer(GL_ARRAY_BUFFER, 0);                         // 解绑（可选）
}

void renderFrame(GLuint vbo, GLuint shaderProgram) {
    glUseProgram(shaderProgram);
    glBindBuffer(GL_ARRAY_BUFFER, vbo);                       // 4. 使用前重新绑定
    // ... 配置顶点属性 + 绘制 ...
}

void cleanup(GLuint vbo) {
    glDeleteBuffers(1, &vbo);                                 // 5. 删除
}
```

---

## glBufferData 的使用模式（Usage Hint）

`glBufferData` 的第四个参数是**使用模式提示**。它告诉驱动程序你打算如何使用这块缓冲，驱动据此选择最优的内存分配策略。

使用模式由两部分组成：**频率 × 用途**：

| | DRAW（CPU→GPU） | READ（GPU→CPU） | COPY（GPU→GPU） |
|---|---|---|---|
| **STATIC**（设置一次） | `GL_STATIC_DRAW` | `GL_STATIC_READ` | `GL_STATIC_COPY` |
| **DYNAMIC**（经常修改） | `GL_DYNAMIC_DRAW` | `GL_DYNAMIC_READ` | `GL_DYNAMIC_COPY` |
| **STREAM**（每帧替换） | `GL_STREAM_DRAW` | `GL_STREAM_READ` | `GL_STREAM_COPY` |

实际开发中最常用的 3 个：

| 使用模式 | 含义 | 典型场景 | KrKr2 中的应用 |
|---------|------|---------|----------------|
| `GL_STATIC_DRAW` | 上传一次，反复绘制 | 静态网格、地形、UI 背景 | 静态精灵顶点 |
| `GL_DYNAMIC_DRAW` | 频繁修改 | 粒子系统、骨骼动画、视频帧 | `YUVSprite.cpp` 视频顶点 |
| `GL_STREAM_DRAW` | 每帧完全替换 | 即时文本渲染、调试线条 | 即时绘制的 UI 元素 |

> **重要提示：** 使用模式只是**提示**（hint），不是约束。即使你用 `GL_STATIC_DRAW` 创建的缓冲，仍然可以调用 `glBufferSubData` 修改数据——只是驱动可能把它放在不太适合频繁写入的内存区域，导致性能下降。选错模式不会报错，但会默默拖慢性能。

---

## 数据布局：交错 vs 分离

顶点数据在内存中有两种常见的组织方式。理解这两种布局对于正确配置 `glVertexAttribPointer` 至关重要。

### 交错布局（Interleaved）— 推荐

所有属性**按顶点交替排列**。同一个顶点的所有属性在内存中连续存放：

```
内存地址 →
┌─────────────────────┐┌─────────────────────┐┌─────────────────────┐
│ 顶点0               ││ 顶点1               ││ 顶点2               │
│ Px Py Pz Cr Cg Cb   ││ Px Py Pz Cr Cg Cb   ││ Px Py Pz Cr Cg Cb   │
│ ├──位置──┤├──颜色──┤ ││ ├──位置──┤├──颜色──┤ ││ ├──位置──┤├──颜色──┤ │
└─────────────────────┘└─────────────────────┘└─────────────────────┘
stride = 6 × sizeof(float) = 24 字节
位置 offset = 0
颜色 offset = 3 × sizeof(float) = 12 字节
```

```cpp
// 交错布局的数据定义
float vertices[] = {
    // 位置           颜色
    -0.5f,-0.5f,0.0f, 1.0f,0.0f,0.0f,  // 顶点0
     0.5f,-0.5f,0.0f, 0.0f,1.0f,0.0f,  // 顶点1
     0.0f, 0.5f,0.0f, 0.0f,0.0f,1.0f   // 顶点2
};

// 交错布局的属性指针配置
GLsizei stride = 6 * sizeof(float);  // 一个顶点的总大小

// 位置属性（location=0）：从字节0开始，每隔 stride 读取 3 个 float
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, stride, (void*)0);
glEnableVertexAttribArray(0);

// 颜色属性（location=1）：从字节12开始，每隔 stride 读取 3 个 float
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, stride,
                      (void*)(3 * sizeof(float)));
glEnableVertexAttribArray(1);
```

### 分离布局（Separate）

每种属性**各自连续存放**：

```
内存地址 →
┌──────────────────────────┐┌──────────────────────────┐
│ 所有位置                 ││ 所有颜色                 │
│ P0x P0y P0z P1x P1y P1z ││ C0r C0g C0b C1r C1g C1b │
│ │  P2x P2y P2z           ││ │  C2r C2g C2b           │
└──────────────────────────┘└──────────────────────────┘
stride = 0（或 3 × sizeof(float)，紧密排列）
位置 offset = 0
颜色 offset = 9 × sizeof(float) = 36 字节（3个顶点 × 3个分量）
```

```cpp
// 分离布局的数据定义
float positions[] = {
    -0.5f, -0.5f, 0.0f,  // 顶点0 位置
     0.5f, -0.5f, 0.0f,  // 顶点1 位置
     0.0f,  0.5f, 0.0f   // 顶点2 位置
};
float colors[] = {
    1.0f, 0.0f, 0.0f,    // 顶点0 颜色（红）
    0.0f, 1.0f, 0.0f,    // 顶点1 颜色（绿）
    0.0f, 0.0f, 1.0f     // 顶点2 颜色（蓝）
};

// 方案A：两种属性放在同一个 VBO 中（拼接后上传）
float combined[18];
memcpy(combined, positions, sizeof(positions));          // 前9个float
memcpy(combined + 9, colors, sizeof(colors));            // 后9个float
glBufferData(GL_ARRAY_BUFFER, sizeof(combined), combined, GL_STATIC_DRAW);

glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 0, (void*)0);
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 0,
                      (void*)(9 * sizeof(float)));

// 方案B：两种属性分别放在两个 VBO 中
GLuint vboPos, vboColor;
glGenBuffers(1, &vboPos);
glBindBuffer(GL_ARRAY_BUFFER, vboPos);
glBufferData(GL_ARRAY_BUFFER, sizeof(positions), positions, GL_STATIC_DRAW);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 0, (void*)0);
glEnableVertexAttribArray(0);

glGenBuffers(1, &vboColor);
glBindBuffer(GL_ARRAY_BUFFER, vboColor);
glBufferData(GL_ARRAY_BUFFER, sizeof(colors), colors, GL_STATIC_DRAW);
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 0, (void*)0);
glEnableVertexAttribArray(1);
```

### 交错 vs 分离对比

| 特性 | 交错布局 | 分离布局 |
|------|---------|---------|
| 缓存友好性 | ✅ 极好（同一顶点的属性连续） | ❌ 差（属性分散在不同内存区域） |
| 单独更新某属性 | ❌ 困难（需要跳过其他属性） | ✅ 简单（整块替换） |
| 代码可读性 | 中等 | ✅ 好（数据分离清晰） |
| 实际使用频率 | 90%+ 生产代码使用 | 少量特殊场景 |

> **KrKr2 实践：** KrKr2 通过 Cocos2d-x 使用交错布局。`V3F_C4B_T2F` 结构体定义在 Cocos2d-x 头文件中，每个顶点包含位置（`Vec3`）、颜色（`Color4B`）、纹理坐标（`Tex2F`）三种属性交错存放。

---

## 数据更新：glBufferSubData 与 glMapBuffer

实际开发中，VBO 的数据经常需要更新——比如粒子位置变化、视频帧切换、UI 元素移动等。OpenGL 提供了两种更新方式。

### 方式一：glBufferSubData — 局部更新

`glBufferSubData` 更新缓冲中**指定范围**的数据，**不重新分配内存**：

```cpp
// 假设 VBO 中有 4 个顶点（矩形），需要更新第 2 个顶点的位置
// 每个顶点 = 6 float（位置3 + 颜色3），第 2 个顶点偏移 = 1 × 6 × 4 = 24 字节

float newPosition[] = { 0.8f, -0.3f, 0.0f };  // 新的位置数据

glBindBuffer(GL_ARRAY_BUFFER, vbo);
glBufferSubData(
    GL_ARRAY_BUFFER,          // 绑定目标
    24,                       // 偏移量（字节）：第 2 个顶点的起始位置
    sizeof(newPosition),      // 更新大小：3 × 4 = 12 字节
    newPosition               // 新数据指针
);
```

**glBufferData vs glBufferSubData 对比：**

| | glBufferData | glBufferSubData |
|---|---|---|
| 内存分配 | 重新分配（释放旧的） | 不分配（原地修改） |
| 可更新范围 | 整个缓冲 | 任意子范围 |
| 性能（大量数据） | 较慢（涉及内存分配） | 较快（仅数据复制） |
| 数据源要求 | 可以传 `nullptr`（只分配不填充） | 必须传有效数据 |

```cpp
// 常用模式：先用 glBufferData 分配空间，再用 glBufferSubData 分段填充
glBufferData(GL_ARRAY_BUFFER, totalSize, nullptr, GL_DYNAMIC_DRAW);
// 此时 GPU 分配了内存但内容未定义

glBufferSubData(GL_ARRAY_BUFFER, 0, chunk1Size, chunk1Data);
glBufferSubData(GL_ARRAY_BUFFER, chunk1Size, chunk2Size, chunk2Data);
// 分两次填充不同区域
```

### 方式二：glMapBuffer / glMapBufferRange — 零拷贝映射

`glMapBuffer` 将 GPU 显存直接映射到 CPU 可访问的地址空间。你可以像操作普通数组一样直接写入 GPU 内存，避免了一次额外的内存拷贝：

```cpp
// === glMapBuffer 示例：直接写入 GPU 显存 ===
glBindBuffer(GL_ARRAY_BUFFER, vbo);

// 将缓冲映射为可写：返回指向 GPU 显存的指针
void* ptr = glMapBuffer(GL_ARRAY_BUFFER, GL_WRITE_ONLY);
if (ptr) {
    float* vertices = static_cast<float*>(ptr);
    // 直接修改 GPU 显存中的数据，就像操作普通数组
    vertices[0] = -0.3f;  // 修改顶点0的 X 坐标
    vertices[1] =  0.7f;  // 修改顶点0的 Y 坐标
    
    glUnmapBuffer(GL_ARRAY_BUFFER);  // 必须解除映射！
    // 解除映射后 ptr 变为无效指针，不可再使用
}
```

**glMapBufferRange（推荐）：** 更精确的版本，可以指定映射范围和访问标志：

```cpp
// === glMapBufferRange 示例：映射部分区域 ===
// 只映射第 2 个顶点（偏移 24 字节，大小 24 字节）
GLsizeiptr offset = 1 * 6 * sizeof(float);  // 24 字节
GLsizeiptr length = 6 * sizeof(float);       // 24 字节

void* ptr = glMapBufferRange(
    GL_ARRAY_BUFFER,
    offset,
    length,
    GL_MAP_WRITE_BIT | GL_MAP_INVALIDATE_RANGE_BIT
    // GL_MAP_WRITE_BIT: 允许写入
    // GL_MAP_INVALIDATE_RANGE_BIT: 告诉驱动这段旧数据不需要保留
);

if (ptr) {
    float* vertex1 = static_cast<float*>(ptr);
    vertex1[0] = 0.8f;   // 新的 X
    vertex1[1] = -0.3f;  // 新的 Y
    vertex1[2] = 0.0f;   // 新的 Z
    vertex1[3] = 1.0f;   // 新的 R
    vertex1[4] = 1.0f;   // 新的 G
    vertex1[5] = 0.0f;   // 新的 B（改为黄色）
    
    glUnmapBuffer(GL_ARRAY_BUFFER);
}
```

**跨平台注意：**

| API | Desktop GL | OpenGL ES 2.0 | OpenGL ES 3.0 |
|-----|-----------|---------------|---------------|
| `glMapBuffer` | ✅ GL 1.5+ | ❌ 不支持 | ❌ 不支持 |
| `glMapBufferRange` | ✅ GL 3.0+ | ❌ 不支持 | ✅ 支持 |
| `glMapBufferOES` | N/A | ⚠️ 需扩展 | N/A |

```cpp
// 跨平台安全的缓冲映射封装
void* mapBuffer(GLenum target, GLintptr offset,
                GLsizeiptr length, GLbitfield access) {
#if defined(GL_ES_VERSION_3_0)
    // OpenGL ES 3.0: 使用 glMapBufferRange
    return glMapBufferRange(target, offset, length, access);
#elif defined(GL_OES_mapbuffer)
    // OpenGL ES 2.0 + 扩展: 只能映射整个缓冲
    return glMapBufferOES(target, GL_WRITE_ONLY_OES);
#else
    // Desktop GL: 完整支持
    return glMapBufferRange(target, offset, length, access);
#endif
}
```

### 三种更新方式对比

| 方式 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| `glBufferData` | 整体替换/重新分配 | 简单直接 | 重新分配内存开销大 |
| `glBufferSubData` | 局部更新 | 不重新分配 | 涉及一次 CPU→GPU 拷贝 |
| `glMapBufferRange` | 高频更新/大量数据 | 零拷贝（直接写显存） | 映射期间 GPU 可能等待 |

---

## 动手实践

### 实践 1：创建并验证 VBO

编写一个程序，创建 VBO 并验证数据上传是否成功。通过 `glGetBufferParameteriv` 查询缓冲大小来确认：

```cpp
// === 完整可运行示例：VBO 创建与验证 ===
#include <glad/glad.h>
#include <GLFW/glfw3.h>
#include <iostream>

int main() {
    // 初始化 GLFW + OpenGL 上下文
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
#ifdef __APPLE__
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
#endif
    GLFWwindow* window = glfwCreateWindow(
        800, 600, "VBO 验证", nullptr, nullptr);
    glfwMakeContextCurrent(window);
    gladLoadGLLoader((GLADloadproc)glfwGetProcAddress);

    // 准备顶点数据
    float vertices[] = {
        -0.5f, -0.5f, 0.0f,   1.0f, 0.0f, 0.0f,
         0.5f, -0.5f, 0.0f,   0.0f, 1.0f, 0.0f,
         0.0f,  0.5f, 0.0f,   0.0f, 0.0f, 1.0f
    };
    GLsizeiptr expectedSize = sizeof(vertices);  // 72 字节

    // 创建 VBO
    GLuint vbo;
    glGenBuffers(1, &vbo);
    glBindBuffer(GL_ARRAY_BUFFER, vbo);
    glBufferData(GL_ARRAY_BUFFER, expectedSize,
                 vertices, GL_STATIC_DRAW);

    // 验证：查询 GPU 端的缓冲大小
    GLint actualSize = 0;
    glGetBufferParameteriv(GL_ARRAY_BUFFER,
                           GL_BUFFER_SIZE, &actualSize);

    std::cout << "期望大小: " << expectedSize << " 字节" << std::endl;
    std::cout << "实际大小: " << actualSize   << " 字节" << std::endl;

    if (actualSize == expectedSize) {
        std::cout << "✅ VBO 创建成功！" << std::endl;
    } else {
        std::cout << "❌ VBO 创建失败！" << std::endl;
    }

    // 清理
    glDeleteBuffers(1, &vbo);
    glfwTerminate();
    return 0;
}
```

**预期输出：**
```
期望大小: 72 字节
实际大小: 72 字节
✅ VBO 创建成功！
```

### 实践 2：动态更新 VBO 数据

创建一个矩形，然后用 `glBufferSubData` 动态修改其中一个顶点的位置：

```cpp
// === 动态更新示例 ===
// 1. 创建矩形的 4 个顶点
float rectVertices[] = {
    // 位置             颜色
    -0.5f,  0.5f, 0.0f, 1.0f, 0.0f, 0.0f,  // 左上
     0.5f,  0.5f, 0.0f, 0.0f, 1.0f, 0.0f,  // 右上
     0.5f, -0.5f, 0.0f, 0.0f, 0.0f, 1.0f,  // 右下
    -0.5f, -0.5f, 0.0f, 1.0f, 1.0f, 0.0f   // 左下
};

GLuint vbo;
glGenBuffers(1, &vbo);
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glBufferData(GL_ARRAY_BUFFER, sizeof(rectVertices),
             rectVertices, GL_DYNAMIC_DRAW);  // 注意：DYNAMIC

// 2. 每帧更新右上角顶点的位置（制造动画效果）
float time = glfwGetTime();
float newVertex[] = {
    0.5f + 0.2f * sinf(time),   // X 随时间摆动
    0.5f + 0.2f * cosf(time),   // Y 随时间摆动
    0.0f,
    0.0f, 1.0f, 0.0f            // 颜色不变
};

// 更新第 2 个顶点（偏移 = 1 × 6 × 4 = 24 字节）
glBufferSubData(GL_ARRAY_BUFFER, 24, sizeof(newVertex), newVertex);
```

---

## 对照项目源码

KrKr2 项目中 VBO 相关代码集中在以下文件：

| 文件 | 行号范围 | 内容 |
|------|---------|------|
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 64-76 | `setupVBOAndVAO()` 中 VBO 的创建和数据上传 |
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 73-76 | 使用 `GL_DYNAMIC_DRAW` 创建视频帧 VBO |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 862-900 | `tTVPOGLTexture2D` 类中的顶点信息管理 |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 464-468 | `GLVertexInfo` 结构体定义（交错布局） |

**关键模式观察：**

1. **DYNAMIC 用于视频：** YUVSprite 的 VBO 使用 `GL_DYNAMIC_DRAW`，因为每切换一帧视频就需要更新顶点的纹理坐标
2. **交错布局：** KrKr2 通过 Cocos2d-x 的 `V3F_C4B_T2F` 结构体使用交错布局，与本节推荐的做法一致
3. **缓冲清理：** 在 `YUVSprite` 析构函数中通过 `glDeleteBuffers` 释放 VBO，防止 GPU 显存泄漏

---

## 常见错误与排查

### 错误 1：忘记绑定就操作缓冲

```cpp
// ❌ 错误：没有绑定 VBO 就上传数据
GLuint vbo;
glGenBuffers(1, &vbo);
// 漏了 glBindBuffer(GL_ARRAY_BUFFER, vbo)!
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
// 结果：数据上传到了"当前绑定"的缓冲（可能是 0 = 无缓冲）
// GL 报错：GL_INVALID_OPERATION

// ✅ 正确：先绑定再操作
glGenBuffers(1, &vbo);
glBindBuffer(GL_ARRAY_BUFFER, vbo);  // 先绑定！
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
```

### 错误 2：glBufferSubData 的偏移或大小越界

```cpp
// ❌ 错误：更新范围超出缓冲大小
// 假设 VBO 大小 = 72 字节（3个顶点 × 6 float × 4 byte）
float data[6] = { ... };
glBufferSubData(GL_ARRAY_BUFFER, 60, sizeof(data), data);
// 偏移60 + 大小24 = 84 > 72 → 越界！
// GL 报错：GL_INVALID_VALUE

// ✅ 正确：确保 offset + size <= buffer_size
glBufferSubData(GL_ARRAY_BUFFER, 48, sizeof(data), data);
// 偏移48 + 大小24 = 72 = buffer_size → 刚好合法
```

### 错误 3：glMapBuffer 后忘记 Unmap

```cpp
// ❌ 错误：映射后忘记解除
void* ptr = glMapBuffer(GL_ARRAY_BUFFER, GL_WRITE_ONLY);
// ... 写入数据 ...
// 忘了 glUnmapBuffer！
// 结果：缓冲保持映射状态，后续任何绘制命令都会失败
// glDrawArrays → GL_INVALID_OPERATION

// ✅ 正确：映射和解除映射必须配对
void* ptr = glMapBuffer(GL_ARRAY_BUFFER, GL_WRITE_ONLY);
if (ptr) {
    // ... 写入数据 ...
    glUnmapBuffer(GL_ARRAY_BUFFER);  // 必须调用！
}
```

---

## 本节小结

- **缓冲对象**是 GPU 显存中的内存区域，用于存储顶点数据，避免每帧 CPU→GPU 传输
- **VBO**（顶点缓冲对象）是最基础的缓冲类型，绑定目标为 `GL_ARRAY_BUFFER`
- VBO 生命周期：`glGenBuffers` → `glBindBuffer` → `glBufferData` → 使用 → `glDeleteBuffers`
- **使用模式**（STATIC/DYNAMIC/STREAM）是性能提示，帮助驱动优化内存分配
- **交错布局**缓存友好，90%+ 的生产代码使用；**分离布局**适合需要单独更新某属性的场景
- **glBufferSubData** 用于局部更新（不重新分配内存）
- **glMapBufferRange** 用于零拷贝映射（直接写入 GPU 显存），ES 2.0 不支持
- 忘记绑定、越界更新、忘记 Unmap 是三大常见错误

---

## 练习题与答案

### 题目 1：缓冲对象生命周期

以下代码有 2 处错误，请找出并修复：

```cpp
GLuint vbo;
glGenBuffers(1, &vbo);
float data[] = { 1.0f, 2.0f, 3.0f };
glBufferData(GL_ARRAY_BUFFER, sizeof(data), data, GL_STATIC_DRAW);
// ... 使用 vbo 绘制 ...
// 程序退出时忘记调用 glDeleteBuffers
```

<details>
<summary>查看答案</summary>

**错误 1：** 调用 `glBufferData` 前没有 `glBindBuffer`。`glGenBuffers` 只生成了一个名字（ID），没有绑定到 `GL_ARRAY_BUFFER` 目标上。

**错误 2：** 程序退出前没有调用 `glDeleteBuffers` 释放 GPU 显存（资源泄漏）。

```cpp
// 修复后的代码：
GLuint vbo;
glGenBuffers(1, &vbo);
glBindBuffer(GL_ARRAY_BUFFER, vbo);  // 修复1：绑定！
float data[] = { 1.0f, 2.0f, 3.0f };
glBufferData(GL_ARRAY_BUFFER, sizeof(data), data, GL_STATIC_DRAW);
// ... 使用 vbo 绘制 ...
glDeleteBuffers(1, &vbo);            // 修复2：释放！
```

</details>

### 题目 2：使用模式选择

为以下场景选择最合适的 `glBufferData` 使用模式，并说明理由：

1. 一个 3D 模型的网格数据，加载后永远不变
2. 粒子系统中 1000 个粒子的位置，每帧更新约 30% 的粒子
3. 一个文字渲染器，每帧生成完全不同的字符四边形

<details>
<summary>查看答案</summary>

1. **`GL_STATIC_DRAW`**——数据上传一次后不再修改，驱动会把它放在 GPU 最快的只读内存区域。

2. **`GL_DYNAMIC_DRAW`**——数据频繁修改但不是每帧全部替换。配合 `glBufferSubData` 只更新变化的粒子，避免重新分配。也可以使用 `glMapBufferRange` 配合 `GL_MAP_INVALIDATE_RANGE_BIT` 来更新。

3. **`GL_STREAM_DRAW`**——每帧数据完全不同（上一帧的字符位置对下一帧没有意义）。驱动知道旧数据不需要保留，可以使用双缓冲或环形缓冲策略优化写入。

</details>

### 题目 3：交错布局计算

一个顶点包含以下属性：
- 位置：`vec3`（3 × float = 12 字节）
- 纹理坐标：`vec2`（2 × float = 8 字节）
- 法线：`vec3`（3 × float = 12 字节）

采用交错布局，请回答：
1. 每个顶点的 stride 是多少字节？
2. 每个属性的 offset 分别是多少字节？
3. 100 个顶点需要多大的 VBO？

<details>
<summary>查看答案</summary>

1. **stride = (3 + 2 + 3) × 4 = 32 字节**

2. **offset 计算：**
   - 位置：offset = **0** 字节
   - 纹理坐标：offset = 3 × 4 = **12** 字节
   - 法线：offset = (3 + 2) × 4 = **20** 字节

3. **100 个顶点 × 32 字节/顶点 = 3200 字节（约 3.1 KB）**

```cpp
GLsizei stride = 32;  // 8 个 float × 4 字节
// 位置 (location = 0)
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, stride, (void*)0);
// 纹理坐标 (location = 1)
glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, stride, (void*)12);
// 法线 (location = 2)
glVertexAttribPointer(2, 3, GL_FLOAT, GL_FALSE, stride, (void*)20);
```

</details>

---

## 下一步

下一节 [VAO 与 EBO](./02-VAO与EBO.md) 将讲解如何用 VAO 打包顶点属性配置、用 EBO 实现索引绘制，以及完整的 OpenGL 绘制命令体系。
