# VAO 与 EBO

> **所属模块：** P04-OpenGL图形编程  
> **前置知识：** [缓冲对象与VBO](./01-缓冲对象与VBO.md)、[OpenGL渲染管线详解](../01-GPU渲染管线概述/02-OpenGL渲染管线详解.md)  
> **预计阅读时间：** 40 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 VAO（顶点数组对象）的状态记录机制与设计动机
2. 掌握 `glVertexAttribPointer` 的每个参数含义及计算方法
3. 理解 EBO（元素缓冲对象）的索引绘制原理与显存节省
4. 区分 `glDrawArrays` 和 `glDrawElements` 的使用场景
5. 编写跨平台兼容的 VAO 代码（Desktop GL / ES 2.0 / ES 3.0）

---

## 场景导入：每帧都要"填表"太烦了

上一节我们学会了用 VBO 把顶点数据上传到 GPU。但有个问题：GPU 拿到这些数据后，**不知道怎么解读它们**。

想象一下：你把一堆混在一起的零件送到工厂的传送带上。工人问你：

- "每个零件从哪里开始？"（stride）
- "位置信息从第几个字节开始？"（offset）
- "每个属性有几个分量？"（size）
- "数据类型是什么？"（type）

如果每次开工（每帧绘制）都要重新回答这些问题（调用 `glVertexAttribPointer`），效率很低。**VAO 的作用就是把这些回答"录"下来**——配置一次，以后只要按"播放"（绑定 VAO），所有配置就自动恢复了。

---

## VAO — 顶点数组对象

### VAO 是什么？

VAO（Vertex Array Object，顶点数组对象）是 OpenGL 3.0 引入的核心概念。它有两个关键特征：

1. **VAO 不存储顶点数据**——数据仍然在 VBO 中
2. **VAO 记录的是"配置状态"**——哪些属性从哪个 VBO 读取、怎么读取

把 VAO 想象成一个"录音机"，它录下了你对顶点属性的所有配置操作：

```
VAO 记录的状态：
┌─────────────────────────────────────────────┐
│ VAO #1 "彩色三角形"                          │
│                                             │
│  属性 0（位置）:                             │
│    ├─ 数据来源: VBO #7                       │
│    ├─ 分量数: 3 (vec3)                       │
│    ├─ 类型: GL_FLOAT                         │
│    ├─ stride: 24 字节                        │
│    ├─ offset: 0 字节                         │
│    └─ 已启用: ✅                             │
│                                             │
│  属性 1（颜色）:                             │
│    ├─ 数据来源: VBO #7                       │
│    ├─ 分量数: 3 (vec3)                       │
│    ├─ 类型: GL_FLOAT                         │
│    ├─ stride: 24 字节                        │
│    ├─ offset: 12 字节                        │
│    └─ 已启用: ✅                             │
│                                             │
│  EBO 绑定: EBO #3                            │
└─────────────────────────────────────────────┘
```

### 没有 VAO vs 有 VAO 的绘制流程

```
━━━ 没有 VAO（每帧重复配置）━━━

帧 1:
  绑定 VBO → 设置属性0 → 启用属性0 → 设置属性1 → 启用属性1 → 绘制物体A
  绑定 VBO → 设置属性0 → 启用属性0 → 设置属性1 → 启用属性1 → 绘制物体B
  绑定 VBO → 设置属性0 → 启用属性0 → 设置属性1 → 启用属性1 → 绘制物体C

帧 2:（完全重复！）
  绑定 VBO → 设置属性0 → 启用属性0 → 设置属性1 → 启用属性1 → 绘制物体A
  ...

━━━ 使用 VAO（配置一次，反复使用）━━━

初始化（仅一次）:
  创建 VAO_A → 绑定 → 配置物体A的所有属性 → 解绑
  创建 VAO_B → 绑定 → 配置物体B的所有属性 → 解绑
  创建 VAO_C → 绑定 → 配置物体C的所有属性 → 解绑

每帧绘制:
  绑定 VAO_A → 绘制    ← 一行搞定！
  绑定 VAO_B → 绘制
  绑定 VAO_C → 绘制
```

### 创建与使用 VAO

VAO 的生命周期与 VBO 类似：**生成 → 绑定 → 配置 → 使用 → 删除**。

```cpp
// === 完整示例：创建 VAO 并配置顶点属性 ===

// 先准备顶点数据（上一节已讲）
float vertices[] = {
    // --- 位置 ---        --- 颜色 ---
    -0.5f, -0.5f, 0.0f,   1.0f, 0.0f, 0.0f,  // 顶点0：红色
     0.5f, -0.5f, 0.0f,   0.0f, 1.0f, 0.0f,  // 顶点1：绿色
     0.0f,  0.5f, 0.0f,   0.0f, 0.0f, 1.0f   // 顶点2：蓝色
};

GLuint vao, vbo;

// === 第 1 步：创建 VAO ===
glGenVertexArrays(1, &vao);  // 生成一个 VAO 名字（ID）
glBindVertexArray(vao);       // 绑定 VAO — 开始"录制"
// 从现在起，所有 glVertexAttribPointer 和 glEnableVertexAttribArray
// 的调用都会被记录到这个 VAO 中

// === 第 2 步：在 VAO "录制"状态下创建并填充 VBO ===
glGenBuffers(1, &vbo);
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

// === 第 3 步：配置顶点属性指针（被 VAO 录制）===
// 属性 0：位置
glVertexAttribPointer(
    0,                    // 属性索引（对应着色器 layout(location=0)）
    3,                    // 分量数（vec3 = 3 个 float）
    GL_FLOAT,             // 数据类型
    GL_FALSE,             // 是否归一化
    6 * sizeof(float),    // stride：每个顶点 6 个 float = 24 字节
    (void*)0              // offset：位置从字节 0 开始
);
glEnableVertexAttribArray(0);  // 启用属性 0

// 属性 1：颜色
glVertexAttribPointer(
    1,                          // 属性索引
    3,                          // 分量数
    GL_FLOAT,                   // 数据类型
    GL_FALSE,                   // 不归一化
    6 * sizeof(float),          // stride 同上
    (void*)(3 * sizeof(float))  // offset：颜色从字节 12 开始
);
glEnableVertexAttribArray(1);  // 启用属性 1

// === 第 4 步：解绑 VAO（停止"录制"）===
glBindVertexArray(0);
// 现在 VAO 中已经保存了所有属性配置
// VBO 可以安全解绑（属性指针已记录了 VBO 引用）
glBindBuffer(GL_ARRAY_BUFFER, 0);
```

绘制时只需绑定 VAO：

```cpp
// === 绘制：绑定 VAO 后直接绘制 ===
glUseProgram(shaderProgram);  // 激活着色器
glBindVertexArray(vao);       // 一行恢复所有属性配置！
glDrawArrays(GL_TRIANGLES, 0, 3);  // 绘制
glBindVertexArray(0);         // 可选：解绑
```

---

## glVertexAttribPointer 深度解析

这是 OpenGL 中**最容易出错**的函数。每个参数都必须精确理解：

```
glVertexAttribPointer(index, size, type, normalized, stride, offset)
                        │      │     │       │          │       │
                        │      │     │       │          │       └─ 该属性在顶点中的字节偏移
                        │      │     │       │          └──── 相邻顶点间的字节距离
                        │      │     │       └───── 整数是否映射到浮点范围
                        │      │     └──────── 每个分量的数据类型
                        │      └────────── 每个顶点该属性的分量个数(1-4)
                        └───────────── 属性位置索引(对应着色器layout)
```

### 参数详解

**index（属性索引）：** 对应着色器中 `layout(location = N)` 的 N 值。
```glsl
layout(location = 0) in vec3 aPos;    // index = 0
layout(location = 1) in vec3 aColor;  // index = 1
layout(location = 2) in vec2 aTexCoord; // index = 2
```

**size（分量数）：** 1-4 之间的整数。`vec2` = 2，`vec3` = 3，`vec4` = 4。
> **注意：** 如果着色器中声明的是 `vec4` 但你传入的 size = 3，OpenGL 会自动补齐第 4 个分量为 1.0（w 分量默认为 1.0，其余默认 0.0）。

**type（数据类型）：**

| 类型 | C++ 对应 | 大小 | 常见用途 |
|------|---------|------|---------|
| `GL_FLOAT` | `float` | 4 字节 | 位置、法线、UV |
| `GL_UNSIGNED_BYTE` | `uint8_t` | 1 字节 | 颜色（0-255） |
| `GL_SHORT` | `int16_t` | 2 字节 | 压缩顶点数据 |
| `GL_HALF_FLOAT` | `__fp16` | 2 字节 | 移动端优化 |

**normalized（归一化）：** 仅对整数类型有效。
- `GL_TRUE`：将整数映射到 [0, 1]（无符号）或 [-1, 1]（有符号）
- `GL_FALSE`：将整数直接转为浮点数

```cpp
// 实际案例：颜色用 4 个 unsigned byte 存储（每通道 0-255）
// normalized = GL_TRUE → 255 映射为 1.0，0 映射为 0.0
glVertexAttribPointer(1, 4, GL_UNSIGNED_BYTE, GL_TRUE, stride, offset);
// 着色器中收到的是 vec4(1.0, 0.5, 0.0, 1.0) 而非 vec4(255, 128, 0, 255)
```

> **KrKr2 实践：** `YUVSprite.cpp` 第 91-93 行中，颜色属性使用 `GL_UNSIGNED_BYTE` + `GL_TRUE`：
> ```cpp
> glVertexAttribPointer(VERTEX_ATTRIB_COLOR, 4, GL_UNSIGNED_BYTE, GL_TRUE,
>     sizeof(V3F_C4B_T2F), (GLvoid*)offsetof(V3F_C4B_T2F, colors));
> ```
> Cocos2d-x 的 `Color4B` 用 4 个 `uint8_t` 存储 RGBA 颜色（共 4 字节），比 4 个 float（16 字节）节省 75% 显存。

**stride（步长）：** 相邻两个顶点之间的字节距离。

```
stride = 一个完整顶点的总字节数

例子：位置(vec3) + 颜色(vec3) = 6 × sizeof(float) = 24 字节

字节地址:  0                24                48
          [P0 C0]           [P1 C1]           [P2 C2]
          ├── stride=24 ──→├── stride=24 ──→│
```

> **stride = 0 的特殊含义：** 当 stride 传 0 时，OpenGL 认为数据是"紧密排列"的，自动计算 stride = size × sizeof(type)。这只在每个 VBO 只存一种属性时才正确。

**offset（偏移）：** 该属性在一个顶点中的起始字节位置。

```
一个顶点的内存布局：
字节 0      12     20     32
│ 位置(12B) │ UV(8B) │ 颜色(12B) │
│ offset=0  │ off=12 │ off=20    │

// 必须强制转换为 (void*) —— 历史遗留 API 设计
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 32, (void*)0);   // 位置
glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, 32, (void*)12);  // UV
glVertexAttribPointer(2, 3, GL_FLOAT, GL_FALSE, 32, (void*)20);  // 颜色
```

### 使用 offsetof 宏（推荐）

当使用结构体定义顶点格式时，`offsetof` 宏比手动计算更安全：

```cpp
// 定义顶点结构体
struct Vertex {
    float position[3];   // 12 字节
    float texCoord[2];   // 8 字节
    uint8_t color[4];    // 4 字节
};  // 总计 24 字节

GLsizei stride = sizeof(Vertex);  // 24

// 使用 offsetof 自动计算偏移
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE,
    stride, (void*)offsetof(Vertex, position));    // 0
glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE,
    stride, (void*)offsetof(Vertex, texCoord));    // 12
glVertexAttribPointer(2, 4, GL_UNSIGNED_BYTE, GL_TRUE,
    stride, (void*)offsetof(Vertex, color));       // 20
```

> **KrKr2 实践：** KrKr2 在 `YUVSprite.cpp` 第 80-97 行中使用 `offsetof` 计算 Cocos2d-x 结构体 `V3F_C4B_T2F` 各属性的偏移量。这是生产代码中的标准做法。

---

## EBO — 元素缓冲对象

### 为什么需要索引绘制？

考虑绘制一个矩形——由 2 个三角形组成。没有索引时，你需要 6 个顶点：

```
非索引方式（6 个顶点，其中 2 个重复）：

顶点数组:  V0  V1  V2  V3  V4  V5
          [A] [B] [C] [D] [E] [F]

三角形 1: A─B─D
三角形 2: B─C─D
但 A=D, B=E → 重复存储了 2 个顶点的数据！

一个矩形: 6 顶点 × 24 字节/顶点 = 144 字节
```

有了索引，只需 4 个唯一顶点 + 6 个索引：

```
索引方式（4 个唯一顶点 + 6 个索引）：

唯一顶点:   V0    V1    V2    V3
           [左上] [右上] [右下] [左下]

索引数组:  [0, 1, 3,    // 三角形 1
            1, 2, 3]    // 三角形 2

一个矩形: 4 顶点 × 24 字节 + 6 索引 × 4 字节 = 120 字节
节省: (144 - 120) / 144 ≈ 17%
```

矩形只节省了 17%，但对于复杂 3D 模型，每个顶点平均被 4-6 个三角形共享，索引绘制可以节省 **40-60%** 的显存：

| 场景 | 非索引 | 索引 | 节省 |
|------|--------|------|------|
| 矩形（4顶点） | 6 × 24 = 144B | 4 × 24 + 6 × 4 = 120B | 17% |
| 立方体（8顶点） | 36 × 24 = 864B | 8 × 24 + 36 × 4 = 336B | 61% |
| 球体（1000面） | 3000 × 32 = 96KB | 500 × 32 + 3000 × 4 = 28KB | 71% |

### 创建与使用 EBO

EBO（Element Buffer Object）也叫 IBO（Index Buffer Object），创建流程与 VBO 类似，但绑定目标是 `GL_ELEMENT_ARRAY_BUFFER`：

```cpp
// === 完整示例：VAO + VBO + EBO 绘制矩形 ===

// 4 个唯一顶点
float vertices[] = {
    // --- 位置 ---        --- 颜色 ---
     0.5f,  0.5f, 0.0f,   1.0f, 0.0f, 0.0f,  // V0：右上角（红）
     0.5f, -0.5f, 0.0f,   0.0f, 1.0f, 0.0f,  // V1：右下角（绿）
    -0.5f, -0.5f, 0.0f,   0.0f, 0.0f, 1.0f,  // V2：左下角（蓝）
    -0.5f,  0.5f, 0.0f,   1.0f, 1.0f, 0.0f   // V3：左上角（黄）
};

// 6 个索引：定义 2 个三角形的组装顺序
unsigned int indices[] = {
    0, 1, 3,  // 三角形 1：右上 → 右下 → 左上
    1, 2, 3   // 三角形 2：右下 → 左下 → 左上
};

GLuint vao, vbo, ebo;

// 创建 VAO（开始录制）
glGenVertexArrays(1, &vao);
glBindVertexArray(vao);

// 创建 VBO
glGenBuffers(1, &vbo);
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices),
             vertices, GL_STATIC_DRAW);

// 创建 EBO — ⚠️ 必须在 VAO 绑定状态下创建！
glGenBuffers(1, &ebo);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices),
             indices, GL_STATIC_DRAW);

// 配置顶点属性
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE,
    6 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE,
    6 * sizeof(float), (void*)(3 * sizeof(float)));
glEnableVertexAttribArray(1);

// 解绑 VAO
glBindVertexArray(0);
// ⚠️ 关键：不要在 VAO 绑定时解绑 EBO！
```

### EBO 绑定与 VAO 的关系（关键陷阱）

EBO 的绑定状态保存在 VAO 中，这意味着：

```cpp
// ❌ 严重错误：在 VAO 绑定时解绑 EBO
glBindVertexArray(vao);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices),
             indices, GL_STATIC_DRAW);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);  // 💥 这会清除 VAO 中的 EBO 引用！
glBindVertexArray(0);
// 后续 glDrawElements 会失败——VAO 中没有 EBO 了

// ✅ 正确：先解绑 VAO，再解绑 EBO
glBindVertexArray(0);                        // 先停止 VAO 录制
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);    // 然后可以安全解绑 EBO
```

为什么 VBO 不受这个限制？因为 `glVertexAttribPointer` 调用时已经"快照"了当前绑定的 VBO 引用。但 EBO 不同——VAO 存储的是 `GL_ELEMENT_ARRAY_BUFFER` 绑定点的**实时引用**。

```
VAO 状态保存规则：
┌──────────────────────────────────────────┐
│ GL_ARRAY_BUFFER 绑定          → 不保存   │  ← VBO 随时可解绑
│ GL_ELEMENT_ARRAY_BUFFER 绑定  → 保存！   │  ← EBO 不可在 VAO 内解绑
│ glVertexAttribPointer 配置    → 保存     │
│ glEnableVertexAttribArray     → 保存     │
└──────────────────────────────────────────┘
```

---

## 绘制命令

OpenGL 提供两大类绘制命令：**非索引绘制**和**索引绘制**。

### glDrawArrays — 非索引绘制

按顺序从 VBO 中取出顶点进行绘制，不使用索引：

```cpp
// 函数签名
void glDrawArrays(GLenum mode, GLint first, GLsizei count);
//                  │            │            │
//                  │            │            └─ 使用几个顶点
//                  │            └─────── 从第几个顶点开始（0-based）
//                  └──────────── 图元类型

// 绘制三角形（3 个顶点 → 1 个三角形）
glBindVertexArray(vao);
glDrawArrays(GL_TRIANGLES, 0, 3);

// 绘制矩形（三角形带：4 个顶点 → 2 个三角形）
glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);

// 绘制第二组三角形（从第 6 个顶点开始，取 3 个）
glDrawArrays(GL_TRIANGLES, 6, 3);
```

> **KrKr2 实践：** `YUVSprite.cpp` 第 256 行使用 `glDrawArrays(GL_TRIANGLE_STRIP, 0, 4)` 绘制视频帧矩形。三角形带只需 4 个顶点就能形成 2 个三角形（矩形），比 `GL_TRIANGLES` 的 6 个顶点更高效。

### glDrawElements — 索引绘制

根据 EBO 中的索引从 VBO 中取出顶点进行绘制：

```cpp
// 函数签名
void glDrawElements(GLenum mode, GLsizei count,
                    GLenum type, const void* indices);
//                    │            │          │            │
//                    │            │          │            └─ 索引偏移（通常为0）
//                    │            │          └─ 索引数据类型
//                    │            └──────── 索引数量
//                    └───────── 图元类型

// 绘制矩形（6 个索引 → 2 个三角形）
glBindVertexArray(vao);  // VAO 中已包含 EBO 绑定
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
```

**索引类型选择：**

| 索引类型 | C++ 类型 | 范围 | 用途 |
|----------|---------|------|------|
| `GL_UNSIGNED_BYTE` | `uint8_t` | 0-255 | ≤256 个顶点的小模型 |
| `GL_UNSIGNED_SHORT` | `uint16_t` | 0-65535 | 大多数 2D/3D 模型 |
| `GL_UNSIGNED_INT` | `uint32_t` | 0-4G | 超大模型 |

> **移动端优化：** 在 OpenGL ES 中，优先使用 `GL_UNSIGNED_SHORT`。KrKr2 在 `YUVSprite.cpp` 第 224 行就使用 `GL_UNSIGNED_SHORT` 类型的索引。16 位索引比 32 位索引节省 50% 的索引缓冲空间，而且移动 GPU 对 16 位索引有硬件优化。

---

## 图元类型一览

`glDrawArrays` 和 `glDrawElements` 的第一个参数 `mode` 指定了如何将顶点组装成图元：

```
图元类型                  图示                        说明
────────────────────────────────────────────────────────────────
GL_POINTS              ●  ●  ●  ●              每个顶点画一个点

GL_LINES               ●──●  ●──●              每 2 个顶点画一条线段
                                               （4个顶点 = 2条线段）

GL_LINE_STRIP          ●──●──●──●              连续线段
                                               （4个顶点 = 3条线段）

GL_LINE_LOOP           ●──●──●──●──●           首尾相连的线段
                       └──────────────┘        （自动连接最后→第一个）

GL_TRIANGLES           ▲     ▲                 每 3 个顶点画一个三角形
                                               （6个顶点 = 2个三角形）

GL_TRIANGLE_STRIP      ▲▲▲▲                    三角形带（共享前两个顶点）
                       0─1                     （4个顶点 = 2个三角形）
                       │╲│                     （N个顶点 = N-2个三角形）
                       3─2

GL_TRIANGLE_FAN        ▲▲▲▲                    三角形扇（共享第一个顶点）
                        ╱│╲                    （适合圆形/扇形）
                       2─0─1
                        ╲│╱
                         3
```

**2D 游戏引擎最常用的：**
- `GL_TRIANGLES`：通用，配合 EBO 使用
- `GL_TRIANGLE_STRIP`：矩形绘制最优（4 个顶点画矩形）

---

## 跨平台 VAO 兼容

VAO 在不同 OpenGL 版本中的支持情况不同，跨平台项目必须处理这个差异：

| 平台 | OpenGL 版本 | VAO 支持 |
|------|------------|---------|
| Windows/Linux（Desktop） | GL 3.0+ | ✅ 原生支持 |
| macOS | GL 3.2+ Core | ✅ 原生支持（且 Core Profile **必须**使用 VAO） |
| Android（高端） | ES 3.0 | ✅ 原生支持 |
| Android（低端） | ES 2.0 | ⚠️ 需扩展 `GL_OES_vertex_array_object` |
| iOS | ES 2.0+ | ⚠️ ES 2.0 需扩展；ES 3.0 原生 |

```cpp
// === 跨平台 VAO 封装 ===

// 封装 VAO 创建/绑定/删除，处理各平台差异
class CrossPlatformVAO {
public:
    void create() {
#if defined(GL_ES_VERSION_3_0) || !defined(GL_ES_VERSION_2_0)
        // ES 3.0+ 或 Desktop GL：原生支持
        glGenVertexArrays(1, &m_vao);
#elif defined(GL_OES_vertex_array_object)
        // ES 2.0 + 扩展
        glGenVertexArraysOES(1, &m_vao);
#else
        // 无 VAO 支持：标记为无效
        m_vao = 0;
        m_supported = false;
#endif
    }

    void bind() {
        if (!m_supported) return;  // 无 VAO 时跳过
#if defined(GL_ES_VERSION_3_0) || !defined(GL_ES_VERSION_2_0)
        glBindVertexArray(m_vao);
#elif defined(GL_OES_vertex_array_object)
        glBindVertexArrayOES(m_vao);
#endif
    }

    void unbind() {
        if (!m_supported) return;
#if defined(GL_ES_VERSION_3_0) || !defined(GL_ES_VERSION_2_0)
        glBindVertexArray(0);
#elif defined(GL_OES_vertex_array_object)
        glBindVertexArrayOES(0);
#endif
    }

    void destroy() {
        if (!m_supported || m_vao == 0) return;
#if defined(GL_ES_VERSION_3_0) || !defined(GL_ES_VERSION_2_0)
        glDeleteVertexArrays(1, &m_vao);
#elif defined(GL_OES_vertex_array_object)
        glDeleteVertexArraysOES(1, &m_vao);
#endif
        m_vao = 0;
    }

    bool isSupported() const { return m_supported; }

private:
    GLuint m_vao = 0;
    bool m_supported = true;
};
```

**无 VAO 时的回退方案：** 每帧手动配置属性指针。

```cpp
// === 无 VAO 设备的绘制流程 ===
void drawWithoutVAO(GLuint vbo, GLuint program) {
    glUseProgram(program);
    glBindBuffer(GL_ARRAY_BUFFER, vbo);

    // 每帧都要重新设置属性指针（没有 VAO 记录状态）
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE,
        6 * sizeof(float), (void*)0);
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE,
        6 * sizeof(float), (void*)(3 * sizeof(float)));
    glEnableVertexAttribArray(1);

    glDrawArrays(GL_TRIANGLES, 0, 3);

    // 可选：禁用属性（防止影响后续绘制）
    glDisableVertexAttribArray(0);
    glDisableVertexAttribArray(1);
}
```

> **KrKr2 实践：** `YUVSprite.cpp` 的 `setupVBOAndVAO()` 函数被 `#ifdef USE_VAO` 包裹，默认不使用 VAO。在 `onDraw()` 方法（第 230-256 行）中，当 VAO 不可用时，每次绘制都手动调用 `glVertexAttribPointer` 设置属性——正是上面演示的回退方案。

---

## 完整绘制流程：从数据到屏幕

将 VBO、VAO、EBO、着色器组合起来的完整流程：

```cpp
// === 完整可运行示例：用 VAO+VBO+EBO 绘制彩色矩形 ===
#include <glad/glad.h>
#include <GLFW/glfw3.h>
#include <iostream>

// 着色器源码
const char* vertSrc = R"(
#version 330 core
layout(location=0) in vec3 aPos;
layout(location=1) in vec3 aColor;
out vec3 vColor;
void main() {
    gl_Position = vec4(aPos, 1.0);
    vColor = aColor;
}
)";

const char* fragSrc = R"(
#version 330 core
in vec3 vColor;
out vec4 FragColor;
void main() {
    FragColor = vec4(vColor, 1.0);
}
)";

// 着色器编译辅助函数
GLuint compileShader(GLenum type, const char* src) {
    GLuint shader = glCreateShader(type);
    glShaderSource(shader, 1, &src, nullptr);
    glCompileShader(shader);
    GLint ok;
    glGetShaderiv(shader, GL_COMPILE_STATUS, &ok);
    if (!ok) {
        char log[512];
        glGetShaderInfoLog(shader, 512, nullptr, log);
        std::cerr << "着色器编译失败: " << log << std::endl;
    }
    return shader;
}

int main() {
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
#ifdef __APPLE__
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
#endif
    auto* win = glfwCreateWindow(800, 600, "矩形", nullptr, nullptr);
    glfwMakeContextCurrent(win);
    gladLoadGLLoader((GLADloadproc)glfwGetProcAddress);

    // 编译着色器
    GLuint vs = compileShader(GL_VERTEX_SHADER, vertSrc);
    GLuint fs = compileShader(GL_FRAGMENT_SHADER, fragSrc);
    GLuint prog = glCreateProgram();
    glAttachShader(prog, vs);
    glAttachShader(prog, fs);
    glLinkProgram(prog);
    glDeleteShader(vs);
    glDeleteShader(fs);

    // 顶点数据
    float verts[] = {
         0.5f,  0.5f, 0.0f,  1.0f, 0.0f, 0.0f,
         0.5f, -0.5f, 0.0f,  0.0f, 1.0f, 0.0f,
        -0.5f, -0.5f, 0.0f,  0.0f, 0.0f, 1.0f,
        -0.5f,  0.5f, 0.0f,  1.0f, 1.0f, 0.0f
    };
    unsigned int idx[] = { 0,1,3, 1,2,3 };

    // 创建 VAO + VBO + EBO
    GLuint vao, vbo, ebo;
    glGenVertexArrays(1, &vao);
    glGenBuffers(1, &vbo);
    glGenBuffers(1, &ebo);

    glBindVertexArray(vao);
    glBindBuffer(GL_ARRAY_BUFFER, vbo);
    glBufferData(GL_ARRAY_BUFFER, sizeof(verts),
                 verts, GL_STATIC_DRAW);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(idx),
                 idx, GL_STATIC_DRAW);

    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE,
        24, (void*)0);
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE,
        24, (void*)12);
    glEnableVertexAttribArray(1);
    glBindVertexArray(0);

    // 渲染循环
    while (!glfwWindowShouldClose(win)) {
        glClearColor(0.15f, 0.15f, 0.15f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT);

        glUseProgram(prog);
        glBindVertexArray(vao);
        glDrawElements(GL_TRIANGLES, 6,
                       GL_UNSIGNED_INT, 0);

        glfwSwapBuffers(win);
        glfwPollEvents();
    }

    // 清理
    glDeleteVertexArrays(1, &vao);
    glDeleteBuffers(1, &vbo);
    glDeleteBuffers(1, &ebo);
    glDeleteProgram(prog);
    glfwTerminate();
    return 0;
}
```

---

## 动手实践

### 实践 1：切换图元类型观察效果

在上面的完整示例基础上，将 `glDrawElements` 的图元类型从 `GL_TRIANGLES` 依次改为以下类型，观察绘制结果：

```cpp
// 试验 1：线框模式
glPolygonMode(GL_FRONT_AND_BACK, GL_LINE);  // 在绘制前添加
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
// 你会看到矩形的三角形线框

// 试验 2：仅绘制顶点
glPointSize(10.0f);  // 设置点大小
glDrawArrays(GL_POINTS, 0, 4);
// 你会看到 4 个大圆点

// 试验 3：线段连接
glDrawArrays(GL_LINE_LOOP, 0, 4);
// 你会看到矩形的四条边（首尾相连）
```

### 实践 2：用 EBO 绘制六边形

用索引绘制一个正六边形（6 个三角形扇形组成）：

```cpp
// 正六边形：中心点 + 6 个外围顶点 = 7 个顶点
float hexVerts[] = {
    // 中心点
     0.0f,  0.0f, 0.0f,   1.0f, 1.0f, 1.0f,  // V0：白色
    // 外围 6 个顶点（60° 间隔）
     0.5f,  0.0f, 0.0f,   1.0f, 0.0f, 0.0f,  // V1：红
     0.25f, 0.43f,0.0f,   1.0f, 1.0f, 0.0f,  // V2：黄
    -0.25f, 0.43f,0.0f,   0.0f, 1.0f, 0.0f,  // V3：绿
    -0.5f,  0.0f, 0.0f,   0.0f, 1.0f, 1.0f,  // V4：青
    -0.25f,-0.43f,0.0f,   0.0f, 0.0f, 1.0f,  // V5：蓝
     0.25f,-0.43f,0.0f,   1.0f, 0.0f, 1.0f   // V6：紫
};

// 6 个三角形，每个共享中心点 V0
unsigned int hexIndices[] = {
    0, 1, 2,  // 三角形 1
    0, 2, 3,  // 三角形 2
    0, 3, 4,  // 三角形 3
    0, 4, 5,  // 三角形 4
    0, 5, 6,  // 三角形 5
    0, 6, 1   // 三角形 6（闭合）
};
// 7 个顶点 + 18 个索引 → 完整六边形
// 非索引方式需要 18 个顶点（3×6），节省了 61%！
```

---

## 对照项目源码

KrKr2 项目中 VAO/EBO 相关代码主要集中在以下文件：

| 文件 | 行号范围 | 内容 |
|------|---------|------|
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 64-112 | `setupVBOAndVAO()`：完整的 VAO+VBO+EBO 创建流程 |
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 200-228 | VAO 路径的绘制：`glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_SHORT, nullptr)` |
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 230-256 | 非 VAO 路径的绘制：手动设置属性 + `glDrawArrays(GL_TRIANGLE_STRIP, 0, 4)` |
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 80-97 | 顶点属性配置：使用 `offsetof` + `V3F_C4B_T2F` 结构体 |

**关键模式观察：**

1. **条件 VAO：** 通过 `#ifdef USE_VAO` 编译时决定使用哪种路径。默认**不使用** VAO（为了兼容 ES 2.0 设备）
2. **图元差异：** VAO 路径用 `GL_TRIANGLES`（6索引），非 VAO 路径用 `GL_TRIANGLE_STRIP`（4顶点）——两种方式绘制同样的矩形
3. **16位索引：** 使用 `GL_UNSIGNED_SHORT` 而非 `GL_UNSIGNED_INT`，节省索引缓冲空间（2D 精灵不需要超过 65535 个顶点）
4. **Cocos2d-x 状态缓存：** 使用 `cocos2d::GL::bindTexture2D()` 而非直接 `glBindTexture`，避免冗余的 GL 状态切换

---

## 常见错误与排查

### 错误 1：在 VAO 绑定状态下解绑 EBO

```cpp
// ❌ 错误：EBO 引用被清除！
glBindVertexArray(vao);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices),
             indices, GL_STATIC_DRAW);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);  // 💥 VAO 丢失 EBO！
glBindVertexArray(0);

// ✅ 正确：先解绑 VAO
glBindVertexArray(0);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);  // 此时安全
```

### 错误 2：macOS Core Profile 不用 VAO 导致黑屏

```cpp
// ❌ macOS Core Profile 必须使用 VAO，否则一切绘制静默失败
// 没有报错，但什么都不画——非常难调试！
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 12, (void*)0);
glDrawArrays(GL_TRIANGLES, 0, 3);  // macOS: 黑屏，无错误

// ✅ 正确：macOS 必须创建并绑定 VAO
GLuint vao;
glGenVertexArrays(1, &vao);
glBindVertexArray(vao);
// ... 然后配置属性和绘制 ...
```

### 错误 3：stride 计算错误导致顶点错乱

```cpp
// ❌ 错误：stride 只算了位置，忘了颜色
// 顶点格式: 位置(3 float) + 颜色(3 float) = 6 float
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE,
    3 * sizeof(float), (void*)0);  // stride=12，应该是24！
// 结果：GPU 把颜色数据当成下一个顶点的位置，图形完全错乱

// ✅ 正确：stride = 整个顶点的总大小
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE,
    6 * sizeof(float), (void*)0);  // stride=24 ✅
```

---

## 本节小结

- **VAO** 记录顶点属性配置状态（不存储数据），避免每帧重复设置属性指针
- **EBO** 存储顶点索引，通过共享顶点减少 40-60% 的数据冗余
- `glVertexAttribPointer` 是最易出错的 API——必须精确计算 stride 和 offset
- 使用 `offsetof` 宏比手动计算偏移更安全、更可维护
- **EBO 绑定保存在 VAO 中**，不可在 VAO 绑定时解绑 EBO（VBO 则可以）
- `glDrawArrays` 用于非索引绘制，`glDrawElements` 用于索引绘制
- 2D 引擎最常用 `GL_TRIANGLES`（通用）和 `GL_TRIANGLE_STRIP`（矩形最优）
- 跨平台项目需要处理 ES 2.0 的 VAO 兼容性（扩展或手动回退）
- macOS Core Profile **强制要求** VAO，不用 VAO 会静默黑屏

---

## 练习题与答案

### 题目 1：stride 和 offset 综合计算

一个顶点包含以下属性（交错布局）：
- 位置：`vec3`（3 × float）
- 法线：`vec3`（3 × float）
- 纹理坐标：`vec2`（2 × float）
- 颜色：`vec4`（4 × unsigned byte）

请计算：
1. 每个顶点的 stride 是多少字节？
2. 每个属性的 offset 是多少字节？
3. 写出所有 `glVertexAttribPointer` 调用

<details>
<summary>查看答案</summary>

**stride 计算：**
- 位置：3 × 4 = 12 字节
- 法线：3 × 4 = 12 字节
- 纹理坐标：2 × 4 = 8 字节
- 颜色：4 × 1 = 4 字节
- **stride = 12 + 12 + 8 + 4 = 36 字节**

**offset 计算：**
- 位置：**0** 字节
- 法线：12 字节
- 纹理坐标：12 + 12 = **24** 字节
- 颜色：12 + 12 + 8 = **32** 字节

```cpp
GLsizei stride = 36;

// 位置 (location = 0): vec3, float
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE,
    stride, (void*)0);
glEnableVertexAttribArray(0);

// 法线 (location = 1): vec3, float
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE,
    stride, (void*)12);
glEnableVertexAttribArray(1);

// 纹理坐标 (location = 2): vec2, float
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE,
    stride, (void*)24);
glEnableVertexAttribArray(2);

// 颜色 (location = 3): vec4, unsigned byte, 归一化
glVertexAttribPointer(3, 4, GL_UNSIGNED_BYTE, GL_TRUE,
    stride, (void*)32);
glEnableVertexAttribArray(3);
```

注意颜色使用 `GL_UNSIGNED_BYTE` + `GL_TRUE`（归一化），4 个字节的颜色比 4 × 4 = 16 字节的 float 颜色节省 75%。

</details>

### 题目 2：VAO 状态问题诊断

以下代码在 macOS 上绘制黑屏，在 Windows 上正常。请找出原因并修复：

```cpp
GLuint vbo;
glGenBuffers(1, &vbo);
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices),
             vertices, GL_STATIC_DRAW);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE,
    3*sizeof(float), (void*)0);
glEnableVertexAttribArray(0);

// 渲染循环
while (...) {
    glUseProgram(prog);
    glDrawArrays(GL_TRIANGLES, 0, 3);  // macOS: 黑屏！
}
```

<details>
<summary>查看答案</summary>

**原因：** macOS Core Profile **强制要求使用 VAO**。没有绑定 VAO 时，所有绘制命令静默失败（不报错，只是什么都不画）。Windows 的 Compatibility Profile 允许不使用 VAO。

**修复：** 添加 VAO 创建和绑定。

```cpp
GLuint vao, vbo;

// 添加 VAO 创建
glGenVertexArrays(1, &vao);
glBindVertexArray(vao);  // 在配置属性之前绑定 VAO

glGenBuffers(1, &vbo);
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices),
             vertices, GL_STATIC_DRAW);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE,
    3*sizeof(float), (void*)0);
glEnableVertexAttribArray(0);

glBindVertexArray(0);  // 配置完成后解绑

// 渲染循环
while (...) {
    glUseProgram(prog);
    glBindVertexArray(vao);  // 绘制前绑定 VAO
    glDrawArrays(GL_TRIANGLES, 0, 3);  // 现在 macOS 也正常了
}
```

**教训：** 始终使用 VAO，即使当前平台不强制要求——这是跨平台兼容的最安全做法。

</details>

### 题目 3：分析 KrKr2 的双路径绘制

阅读 `cpp/core/environ/cocos2d/YUVSprite.cpp` 的 `onDraw()` 方法，回答：

1. VAO 路径使用什么图元类型？非 VAO 路径呢？为什么不同？
2. 非 VAO 路径中为什么要创建局部数组（`vertices[4]`、`colors[4]`）而不是直接传 VBO？
3. 如果要让 KrKr2 在 macOS Core Profile 上运行，`USE_VAO` 宏应该设为什么？

<details>
<summary>查看答案</summary>

1. **VAO 路径：** `GL_TRIANGLES`（6 个索引绘制 2 个三角形）
   **非 VAO 路径：** `GL_TRIANGLE_STRIP`（4 个顶点绘制 2 个三角形）
   **原因：** VAO 路径搭配 EBO 使用索引绘制，用 `GL_TRIANGLES` 更灵活。非 VAO 路径没有 EBO，用 `GL_TRIANGLE_STRIP` 只需 4 个顶点就能画矩形，比 `GL_TRIANGLES` 需要的 6 个顶点更节省。

2. **原因：** 非 VAO 路径中没有绑定 VBO，`glVertexAttribPointer` 在没有 VBO 绑定时会把 offset 参数解释为**客户端内存地址**（而非 GPU 缓冲内偏移）。因此需要把 Cocos2d-x 交错结构体 `V3F_C4B_T2F` 中的各属性**提取到独立数组**中，让 `glVertexAttribPointer` 直接指向这些 CPU 端数组。这是没有 VBO 时唯一的数据传递方式。

3. **必须定义 `USE_VAO`**。macOS Core Profile 强制要求 VAO，如果 `USE_VAO` 未定义，绘制会走非 VAO 路径，在 macOS 上静默失败（黑屏）。

</details>

---

## 下一步

下一节 [着色器程序与实践](./03-着色器程序与实践.md) 将讲解如何编写、编译、链接着色器程序，以及将 VBO + VAO + 着色器组合成完整的渲染管线。
