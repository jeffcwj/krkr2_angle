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

