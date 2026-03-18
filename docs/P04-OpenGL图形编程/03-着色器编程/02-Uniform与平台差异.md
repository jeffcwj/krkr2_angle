# Uniform 与平台差异

> **所属模块：** P04-OpenGL图形编程  
> **前置知识：** [GLSL语言与着色器阶段](./01-GLSL语言与着色器阶段.md)、[缓冲对象与VBO](../02-OpenGL基础/01-缓冲对象与VBO.md)  
> **预计阅读时间：** 40 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 Uniform 变量的概念：CPU→GPU 的每帧参数传递通道
2. 掌握各种 Uniform 设置函数（标量、向量、矩阵、采样器）
3. 编写完整的 Uniform 传值代码，包含位置缓存与错误处理
4. 理解 GLSL 版本差异（ES 2.0 / ES 3.0 / Desktop 3.3）及其对 Uniform 的影响
5. 使用预处理器编写跨平台兼容的着色器代码

---

## 场景导入：着色器怎么知道"当前时间"？

回忆一下你已经写过的着色器——顶点着色器通过 `in`（或 `attribute`）接收来自 VBO 的顶点数据。但有些信息不是顶点级别的：

- **变换矩阵**：所有顶点共用同一个 MVP 矩阵
- **全局颜色**：整个物体统一染色
- **当前时间**：用于动画效果
- **纹理单元编号**：告诉采样器从哪个纹理读取

这些"对所有顶点/片段都相同"的参数，就是 **Uniform**。

```
数据流向对比：
┌─────────┐                    ┌─────────────┐
│  CPU 端  │                    │  GPU 着色器  │
│          │                    │             │
│ 顶点数组  │ ─── VBO ────────→ │ in/attribute│  ← 每个顶点不同
│          │                    │             │
│ 矩阵/颜色 │ ─── Uniform ───→ │ uniform     │  ← 所有顶点相同
│ 时间/标志 │                    │             │
│          │                    │             │
│ 纹理对象  │ ─── 纹理单元 ──→  │ sampler2D   │  ← 通过 Uniform 绑定
└─────────┘                    └─────────────┘
```

**Uniform**（统一变量）就是 CPU 每帧（或按需）传给 GPU 着色器的**只读常量**。着色器执行期间，Uniform 的值不会变——对当前绘制调用（Draw Call）中的所有顶点和片段都相同。

---

## Uniform 基本规则

在使用 Uniform 之前，必须理解 5 条核心规则：

### 规则 1：必须在 glUseProgram 之后设置

Uniform 的设置针对的是**当前激活的着色器程序**。如果没有先调用 `glUseProgram`，设置的值不知道传给谁：

```cpp
// ❌ 错误：没有先激活程序
glUniform1f(loc, 1.0f);  // 传给谁？GL_INVALID_OPERATION

// ✅ 正确：先激活，再设置
glUseProgram(shaderProgram);
glUniform1f(loc, 1.0f);  // 传给 shaderProgram 中的 uniform
```

### 规则 2：位置应该在初始化时缓存

每次调用 `glGetUniformLocation` 都会在驱动内部做字符串查找，开销不小。正确做法是初始化时查询一次，之后直接使用缓存的位置：

```cpp
// ❌ 低效：每帧查询
void renderFrame() {
    GLint loc = glGetUniformLocation(prog, "uMVP");  // 每帧字符串查找！
    glUniformMatrix4fv(loc, 1, GL_FALSE, mvpData);
}

// ✅ 高效：初始化时缓存
class Renderer {
    GLint m_locMVP = -1;
    GLint m_locColor = -1;
    GLint m_locTime = -1;

    void init() {
        glUseProgram(m_prog);
        m_locMVP   = glGetUniformLocation(m_prog, "uMVP");
        m_locColor = glGetUniformLocation(m_prog, "uTintColor");
        m_locTime  = glGetUniformLocation(m_prog, "uTime");
    }

    void renderFrame(float time) {
        glUseProgram(m_prog);
        if (m_locMVP >= 0)   glUniformMatrix4fv(m_locMVP, 1, GL_FALSE, mvp);
        if (m_locTime >= 0)  glUniform1f(m_locTime, time);
        if (m_locColor >= 0) glUniform4f(m_locColor, 1.0f, 1.0f, 1.0f, 1.0f);
    }
};
```

### 规则 3：未使用的 Uniform 会被优化掉

如果着色器代码中声明了 `uniform float uUnused;` 但从未在计算中使用它，GLSL 编译器会将其优化掉。此时 `glGetUniformLocation` 返回 **-1**：

```cpp
GLint loc = glGetUniformLocation(prog, "uUnused");
// loc == -1（被编译器优化掉了）

// 对 loc=-1 调用 glUniform* 不会报错，只是静默忽略
// 但你应该检查 loc 是否有效，便于调试
if (loc < 0) {
    std::cerr << "警告: uniform 'uUnused' 未找到（可能被优化）"
              << std::endl;
}
```

### 规则 4：Uniform 值在程序切换后保留

每个着色器程序维护自己的 Uniform 值。设置后即使切换到另一个程序再切回来，Uniform 值仍然保留：

```cpp
glUseProgram(progA);
glUniform1f(locTime, 3.14f);  // progA 的 uTime = 3.14

glUseProgram(progB);          // 切换到 progB
// progB 有自己的 uniform 集

glUseProgram(progA);          // 切回 progA
// progA 的 uTime 仍然是 3.14（保留！）
```

### 规则 5：Uniform 有数量上限

每个着色器阶段能使用的 Uniform 数量是有限的。可以查询上限：

```cpp
GLint maxVertUniforms, maxFragUniforms;
glGetIntegerv(GL_MAX_VERTEX_UNIFORM_VECTORS, &maxVertUniforms);
glGetIntegerv(GL_MAX_FRAGMENT_UNIFORM_VECTORS, &maxFragUniforms);
std::cout << "顶点着色器最大 uniform 向量: " << maxVertUniforms << std::endl;
std::cout << "片段着色器最大 uniform 向量: " << maxFragUniforms << std::endl;
// 典型值: Desktop GL ≥ 1024, ES 2.0 ≥ 128, ES 3.0 ≥ 256
```

---

## Uniform 设置函数全家族

OpenGL 为不同数据类型提供了对应的 `glUniform*` 函数：

### 标量与向量

```cpp
// === 标量 ===
glUniform1f(loc, 1.0f);              // float
glUniform1i(loc, 5);                 // int（也用于 sampler）
glUniform1ui(loc, 10u);             // unsigned int (GL 3.0+)

// === 向量 ===
glUniform2f(loc, 0.5f, 0.5f);       // vec2
glUniform3f(loc, 1.0f, 0.0f, 0.0f); // vec3（如颜色 RGB）
glUniform4f(loc, 1.0f, 0.85f, 0.85f, 1.0f); // vec4（如 RGBA 染色）

// === 数组形式（传指针）===
float color[4] = {1.0f, 0.0f, 0.0f, 1.0f};
glUniform4fv(loc, 1, color);        // 传 1 个 vec4
// 第二个参数 count：传多少个 vec4（用于 uniform 数组）

float positions[6] = {0.0f,0.0f, 1.0f,0.0f, 0.5f,1.0f};
glUniform2fv(loc, 3, positions);    // 传 3 个 vec2
```

### 矩阵

```cpp
// === 矩阵 ===
// float mvp[16] = { ... }; // 列主序 4x4 矩阵
glUniformMatrix4fv(
    loc,            // uniform 位置
    1,              // 矩阵数量（通常 1）
    GL_FALSE,       // 是否转置：GL_FALSE = 列主序（OpenGL 默认）
    mvp             // 数据指针
);

// 其他矩阵类型
glUniformMatrix2fv(loc, 1, GL_FALSE, mat2Data);   // mat2
glUniformMatrix3fv(loc, 1, GL_FALSE, mat3Data);   // mat3
glUniformMatrix3x2fv(loc, 1, GL_FALSE, data);     // mat3x2 (GL 2.1+)
```

> **transpose 参数：** 第三个参数 `transpose`。如果你的矩阵是行主序（如 DirectX 风格），传 `GL_TRUE` 让 OpenGL 自动转置。但注意 **OpenGL ES 2.0 不支持 GL_TRUE**——必须传 `GL_FALSE`，这意味着你必须在 CPU 端保证矩阵是列主序。

### 采样器（纹理单元）

采样器 Uniform 比较特殊——它的值不是纹理 ID，而是**纹理单元编号**：

```cpp
// 着色器中声明：
// uniform sampler2D uTexture;  // 采样器

// CPU 端设置：
glActiveTexture(GL_TEXTURE0);           // 激活纹理单元 0
glBindTexture(GL_TEXTURE_2D, texId);    // 把纹理对象绑定到单元 0
glUniform1i(locTexture, 0);             // 告诉 sampler：从单元 0 采样
//                      ↑ 注意：这里是单元编号 0，不是纹理 ID！
```

完整的多纹理 Uniform 设置示例（YUV 三平面）：

```cpp
// === 完整示例：YUV 三纹理绑定 ===
// 着色器中：
//   uniform sampler2D uTexY;   // Y 亮度平面
//   uniform sampler2D uTexU;   // U 色差平面
//   uniform sampler2D uTexV;   // V 色差平面

GLint locY = glGetUniformLocation(prog, "uTexY");
GLint locU = glGetUniformLocation(prog, "uTexU");
GLint locV = glGetUniformLocation(prog, "uTexV");

// 每帧绘制时：
glUseProgram(prog);

// Y 平面 → 纹理单元 0
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, texIdY);
glUniform1i(locY, 0);

// U 平面 → 纹理单元 1
glActiveTexture(GL_TEXTURE1);
glBindTexture(GL_TEXTURE_2D, texIdU);
glUniform1i(locU, 1);

// V 平面 → 纹理单元 2
glActiveTexture(GL_TEXTURE2);
glBindTexture(GL_TEXTURE_2D, texIdV);
glUniform1i(locV, 2);

// 绘制视频帧
glBindVertexArray(vao);
glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
```

> **KrKr2 实践：** `YUVSprite.cpp` 第 219-235 行正是这个模式——绑定 Y/U/V 三张纹理到三个纹理单元，然后通过 Cocos2d-x 的 `_glProgramState->apply()` 统一设置所有 Uniform。

---

## GLSL 版本与平台差异

跨平台 OpenGL 开发中最大的痛点之一就是 GLSL 版本差异。同一个功能在不同版本中语法不同，如果不处理好兼容性，着色器在 A 设备上编译通过，在 B 设备上就失败。

### 版本对照速查表

| 特性 | ES 2.0 (`#version 100`) | ES 3.0 (`#version 300 es`) | Desktop 3.3 (`#version 330 core`) |
|------|------------------------|---------------------------|----------------------------------|
| **顶点输入** | `attribute` | `in` | `in` + `layout(location=N)` |
| **阶段间传递** | `varying` | `in`/`out` | `in`/`out` |
| **片段输出** | `gl_FragColor` | 自定义 `out vec4` | 自定义 `out vec4` |
| **纹理采样** | `texture2D()` | `texture()` | `texture()` |
| **精度限定符** | ✅ 必须（片段） | ✅ 必须（片段） | ⚠️ 可选（有默认） |
| **layout 限定符** | ❌ 不支持 | ✅ 支持 | ✅ 支持 |

### 三个版本的同一个着色器

同一个"纹理采样 + 颜色染色"效果，在三个版本中的写法对比：

**ES 2.0 (`#version 100`)：**

```glsl
// === 顶点着色器 (ES 2.0) ===
#version 100
attribute vec3 aPos;           // 老语法: attribute
attribute vec2 aTexCoord;
varying vec2 vTexCoord;        // 老语法: varying
uniform mat4 uMVP;

void main() {
    gl_Position = uMVP * vec4(aPos, 1.0);
    vTexCoord = aTexCoord;
}
```

```glsl
// === 片段着色器 (ES 2.0) ===
#version 100
precision mediump float;       // ES 必须声明精度！
varying vec2 vTexCoord;
uniform sampler2D uTexture;
uniform vec4 uTintColor;

void main() {
    vec4 texColor = texture2D(uTexture, vTexCoord);  // 老函数名
    gl_FragColor = texColor * uTintColor;             // 老输出名
}
```

**ES 3.0 (`#version 300 es`)：**

```glsl
// === 顶点着色器 (ES 3.0) ===
#version 300 es
layout(location = 0) in vec3 aPos;      // 新语法: in + layout
layout(location = 1) in vec2 aTexCoord;
out vec2 vTexCoord;                      // 新语法: out
uniform mat4 uMVP;

void main() {
    gl_Position = uMVP * vec4(aPos, 1.0);
    vTexCoord = aTexCoord;
}
```

```glsl
// === 片段着色器 (ES 3.0) ===
#version 300 es
precision mediump float;          // ES 仍然需要精度声明
in vec2 vTexCoord;                // 新语法: in
out vec4 FragColor;               // 自定义输出变量名
uniform sampler2D uTexture;
uniform vec4 uTintColor;

void main() {
    vec4 texColor = texture(uTexture, vTexCoord);  // 新函数名
    FragColor = texColor * uTintColor;              // 自定义输出
}
```

**Desktop 3.3 (`#version 330 core`)：**

```glsl
// === 顶点着色器 (Desktop 3.3) ===
#version 330 core
layout(location = 0) in vec3 aPos;
layout(location = 1) in vec2 aTexCoord;
out vec2 vTexCoord;
uniform mat4 uMVP;

void main() {
    gl_Position = uMVP * vec4(aPos, 1.0);
    vTexCoord = aTexCoord;
}
```

```glsl
// === 片段着色器 (Desktop 3.3) ===
#version 330 core
in vec2 vTexCoord;
out vec4 FragColor;
uniform sampler2D uTexture;
uniform vec4 uTintColor;

void main() {
    vec4 texColor = texture(uTexture, vTexCoord);
    FragColor = texColor * uTintColor;
}
// 注意：Desktop 不需要 precision 声明
```

### 预处理器技巧：一份代码兼容多平台

在实际项目中，维护三份着色器太费力。常用预处理器技巧减少代码重复：

**方案 1：`#ifdef GL_ES` 区分 ES 和 Desktop**

```glsl
// === 跨平台着色器模板 ===
#ifdef GL_ES
    #ifdef GL_FRAGMENT_PRECISION_HIGH
        precision highp float;    // 如果设备支持 highp
    #else
        precision mediump float;  // 否则用 mediump
    #endif
#endif

// ES 2.0 和 Desktop/ES 3.0 的关键字差异通过 #define 统一
#if __VERSION__ < 130
    // ES 2.0 (#version 100) 的语法映射
    #define VERT_IN   attribute
    #define VERT_OUT  varying
    #define FRAG_IN   varying
    #define TEX2D     texture2D
#else
    // ES 3.0+ / Desktop 3.3+ 的语法映射
    #define VERT_IN   in
    #define VERT_OUT  out
    #define FRAG_IN   in
    #define TEX2D     texture
#endif
```

然后着色器这样写：

```glsl
// 顶点着色器（兼容 ES 2.0 和 3.0+）
VERT_IN vec3 aPos;
VERT_IN vec2 aTexCoord;
VERT_OUT vec2 vTexCoord;
uniform mat4 uMVP;

void main() {
    gl_Position = uMVP * vec4(aPos, 1.0);
    vTexCoord = aTexCoord;
}
```

**方案 2：C++ 端动态注入版本头**

更灵活的做法是在 C++ 端根据运行时 GL 版本动态拼接着色器源码：

```cpp
// === C++ 端：动态生成版本头 ===
std::string getShaderHeader() {
#ifdef __ANDROID__
    // Android: 检查 GL ES 版本
    const char* ver = (const char*)glGetString(GL_VERSION);
    if (strstr(ver, "OpenGL ES 3")) {
        return "#version 300 es\nprecision mediump float;\n";
    } else {
        return "#version 100\nprecision mediump float;\n";
    }
#elif defined(__APPLE__)
    // macOS: Core Profile 3.2+
    return "#version 330 core\n";
#else
    // Windows/Linux: 按需选择
    return "#version 330 core\n";
#endif
}

// 编译着色器时拼接
std::string fullSource = getShaderHeader() + shaderBody;
const char* src = fullSource.c_str();
glShaderSource(shader, 1, &src, nullptr);
glCompileShader(shader);
```

> **KrKr2 实践：** `YUVSprite.cpp` 第 7-38 行中，YUV 片段着色器源码包含 `#ifdef GL_ES` 和 `precision` 声明，并使用 `texture2D`（ES 2.0 兼容）。`RenderManager_ogl.cpp` 第 84-102 行维护了扩展名清单，运行时检测设备能力。

---

## 精度限定符（ES 特有）

OpenGL ES 的一个核心差异是**精度限定符**（Precision Qualifier）。移动 GPU 硬件支持不同精度级别的浮点运算，选对精度可以显著提升性能。

### 三种精度级别

| 精度 | 最小范围 | 最小精度 | 用途 | 性能 |
|------|---------|---------|------|------|
| `highp` | ±2^62 | 2^-16 | 坐标变换、深度计算 | 最慢 |
| `mediump` | ±2^14 | 2^-10 | 颜色计算、UV 坐标 | 中等 |
| `lowp` | ±2 | 2^-8 | 简单颜色标志 | 最快 |

```glsl
// ES 片段着色器中的精度声明
precision mediump float;    // 设置 float 的默认精度为 mediump

// 也可以对单个变量指定精度
highp float depth;          // 深度值用高精度
lowp vec4 flatColor;        // 简单颜色用低精度
mediump vec2 uv;            // UV 坐标用中精度
```

### 精度陷阱：为什么移动端颜色"抖动"？

```glsl
// ❌ 在 mediump 下做大范围坐标运算可能出问题
precision mediump float;
uniform float uTime;  // 如果 uTime > 16384，mediump 会溢出！

void main() {
    // mediump 的精度只有 2^-10 ≈ 0.001
    // 当 uTime 很大时，sin(uTime) 的输入精度不够
    float wave = sin(uTime * 10.0);  // 抖动！
    FragColor = vec4(wave, wave, wave, 1.0);
}

// ✅ 对时间和坐标相关运算使用 highp
#ifdef GL_ES
    #ifdef GL_FRAGMENT_PRECISION_HIGH
        precision highp float;
    #else
        precision mediump float;
        // 注意：某些低端设备不支持片段着色器的 highp
        // 此时需要重构算法避免大数值运算
    #endif
#endif
```

> **重要：** 并非所有移动 GPU 都支持片段着色器的 `highp`。可以通过检查 `GL_FRAGMENT_PRECISION_HIGH` 宏来确认。如果不支持，需要在算法层面避免大数值运算（例如对时间取模后再传入着色器）。

### 各平台精度默认值

| 着色器阶段 | ES 2.0 默认 | ES 3.0 默认 | Desktop 默认 |
|-----------|------------|------------|-------------|
| 顶点 float | `highp` | `highp` | 无精度概念（32位） |
| 顶点 int | `highp` | `highp` | 无精度概念（32位） |
| 片段 float | **无默认！** | **无默认！** | 无精度概念（32位） |
| 片段 int | `mediump` | `mediump` | 无精度概念（32位） |

> **关键：** ES 片段着色器的 float **没有默认精度**，必须显式声明 `precision mediump float;`。忘了写会导致编译错误——这是 ES 和 Desktop 最常见的不兼容来源。

---

## 扩展检测与降级路径

跨平台项目不能假设所有设备支持相同的扩展。KrKr2 通过运行时扩展检测实现能力探测：

```cpp
// === 扩展检测示例 ===
bool hasFramebufferFetch = false;
bool hasTextureFloat = false;

void detectExtensions() {
    const char* extensions =
        (const char*)glGetString(GL_EXTENSIONS);
    
    // 检查 shader framebuffer fetch（某些移动 GPU 支持）
    if (strstr(extensions,
               "GL_EXT_shader_framebuffer_fetch")) {
        hasFramebufferFetch = true;
        // 可以在片段着色器中直接读取当前帧缓冲颜色
        // 实现低开销的 Alpha 混合
    }
    
    // 检查浮点纹理支持
    if (strstr(extensions, "GL_OES_texture_float")) {
        hasTextureFloat = true;
        // 可以使用 32 位浮点纹理
    }
    
    std::cout << "Framebuffer Fetch: "
              << (hasFramebufferFetch ? "✅" : "❌")
              << std::endl;
    std::cout << "Texture Float: "
              << (hasTextureFloat ? "✅" : "❌")
              << std::endl;
}

// 根据能力选择着色器变体
const char* getFragmentShader() {
    if (hasFramebufferFetch) {
        return fragShaderWithFetch;   // 高性能路径
    } else {
        return fragShaderStandard;    // 标准路径
    }
}
```

> **KrKr2 实践：** `RenderManager_ogl.cpp` 第 84-102 行维护了扩展名清单，第 217-221 行运行时检测 `GL_EXT_shader_framebuffer_fetch` 是否可用。这种"检测→选择→降级"的模式是跨平台 OpenGL 开发的标准做法。

---

## 动手实践

### 实践 1：Uniform 传值全流程

编写一个程序，用 Uniform 控制三角形的颜色和位置偏移：

```cpp
// === 着色器源码 ===
const char* vertSrc = R"(
#version 330 core
layout(location=0) in vec3 aPos;
uniform vec2 uOffset;     // 位置偏移
uniform float uScale;     // 缩放因子

void main() {
    vec3 pos = aPos * uScale;          // 缩放
    pos.xy += uOffset;                 // 平移
    gl_Position = vec4(pos, 1.0);
}
)";

const char* fragSrc = R"(
#version 330 core
out vec4 FragColor;
uniform vec4 uColor;      // 染色颜色
uniform float uTime;       // 时间（用于动画）

void main() {
    float pulse = 0.5 + 0.5 * sin(uTime * 3.0);
    FragColor = uColor * pulse;        // 颜色随时间脉动
}
)";

// === 主程序 ===
// 初始化时缓存 uniform 位置
GLint locOffset, locScale, locColor, locTime;
void initUniforms(GLuint prog) {
    glUseProgram(prog);
    locOffset = glGetUniformLocation(prog, "uOffset");
    locScale  = glGetUniformLocation(prog, "uScale");
    locColor  = glGetUniformLocation(prog, "uColor");
    locTime   = glGetUniformLocation(prog, "uTime");
    
    // 打印位置信息用于调试
    std::cout << "uOffset=" << locOffset
              << " uScale=" << locScale
              << " uColor=" << locColor
              << " uTime="  << locTime << std::endl;
}

// 每帧更新 uniform
void updateUniforms(float time) {
    glUniform2f(locOffset,
                0.3f * sinf(time),
                0.3f * cosf(time));        // 圆形运动
    glUniform1f(locScale, 0.5f);           // 缩小到一半
    glUniform4f(locColor,
                1.0f, 0.5f, 0.2f, 1.0f);  // 橙色
    glUniform1f(locTime, time);            // 当前时间
}
```

### 实践 2：编写跨 ES 2.0 / Desktop 的着色器

同一个着色器效果，分别用 ES 2.0 和 Desktop 3.3 语法写一遍，验证两边都能编译运行。使用上文的预处理器模板减少重复代码。

---

## 对照项目源码

| 文件 | 行号范围 | 内容 |
|------|---------|------|
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 7-38 | YUV 片段着色器：`#ifdef GL_ES` + `precision` + `texture2D` |
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 48-53 | 着色器编译链接：`GLProgram::createWithByteArrays` + `link` + `updateUniforms` |
| `cpp/core/environ/cocos2d/YUVSprite.cpp` | 219-235 | 三纹理 Uniform 绑定：Y/U/V 分别绑定到纹理单元 0/1/2 |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 84-102 | 扩展名清单（`GL_EXT_shader_framebuffer_fetch` 等） |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 217-221 | 运行时扩展检测与能力探测 |

**关键实战结论：**
- 教程里的标准 OpenGL Uniform 流程，在 KrKr2 中被 Cocos2d-x 封装了一层（`GLProgram` / `GLProgramState`），但本质完全一致
- 跨平台不只是语法兼容，还要考虑精度、扩展支持、能力降级

---

## 常见错误与排查

### 错误 1：精度问题导致颜色抖动或偏色

**现象：** 仅移动端出现带状色阶、颜色偏灰、渐变断层。Desktop 正常。

**原因：** 片段着色器使用 `lowp` / `mediump` 导致精度不足。

```glsl
// ❌ mediump 下的坐标运算精度不够
precision mediump float;
uniform float uTime;  // uTime > 16384 时 mediump 溢出

// ✅ 对关键路径使用 highp
#ifdef GL_ES
precision highp float;  // 颜色/坐标变换用 highp
#endif
```

### 错误 2：varying / in / out 不匹配导致链接失败

**现象：** 编译成功但链接失败。

```glsl
// ❌ 顶点和片段的接口类型不匹配
// 顶点: out vec2 vUV;
// 片段: in vec3 vUV;  ← 类型不同！

// ✅ 必须完全一致：类型、名称、数组维度
// 顶点: out vec2 vUV;
// 片段: in vec2 vUV;  ← 完全匹配
```

**排查方法：** 链接后检查 `GL_LINK_STATUS`，失败时打印 `glGetProgramInfoLog`。

### 错误 3：Sampler 绑定错纹理单元

**现象：** 画面纹理错位、YUV 颜色严重异常。

```cpp
// ❌ 错误：sampler 说用单元 1，但纹理绑定在单元 2
glUniform1i(locTexU, 1);              // sampler 指向单元 1
glActiveTexture(GL_TEXTURE2);         // 但纹理绑在单元 2！
glBindTexture(GL_TEXTURE_2D, texU);

// ✅ 正确：sampler 编号和 glActiveTexture 单元编号必须一致
glUniform1i(locTexU, 1);              // sampler 指向单元 1
glActiveTexture(GL_TEXTURE1);         // 纹理也绑在单元 1 ✅
glBindTexture(GL_TEXTURE_2D, texU);
```

**建议：** 建立"单元号→语义"映射表，初始化时一次性配置并打印日志验证。

---

## 本节小结

- **Uniform** 是 CPU 向 GPU 传递"每帧不变"参数的通道（矩阵、颜色、时间、纹理单元）
- 必须在 `glUseProgram` 之后设置 Uniform，位置应初始化时缓存
- Uniform 值在程序切换后保留，未使用的 Uniform 会被编译器优化掉（location=-1）
- 采样器 Uniform 的值是**纹理单元编号**（0/1/2...），不是纹理对象 ID
- GLSL 版本差异：ES 2.0 用 `attribute/varying/texture2D/gl_FragColor`；ES 3.0+ 和 Desktop 3.3+ 用 `in/out/texture/自定义输出`
- ES 片段着色器必须声明 `precision`，Desktop 不需要
- 精度选择影响性能和正确性：`highp` 用于坐标/深度，`mediump` 用于颜色/UV
- 跨平台项目要做扩展检测和能力降级，不能假设所有设备支持同一功能

---

## 练习题与答案

### 题目 1：为什么 ES 2.0 着色器常见 `#ifdef GL_ES` 和 `precision`，但 Desktop 330 不写？

<details>
<summary>查看答案</summary>

OpenGL ES 面向移动设备，片段着色器的精度等级（`lowp/mediump/highp`）是语言规范的一部分。ES 2.0/3.0 **要求**片段着色器显式声明 float 精度，否则编译报错。

Desktop GLSL 没有精度等级概念——所有浮点运算都是 32 位 IEEE 754。`precision` 关键字在 Desktop GLSL 中虽然不会报错（会被忽略），但不是必须的。

因此跨平台写法用 `#ifdef GL_ES` 包裹精度声明：ES 编译器会处理它，Desktop 编译器会跳过它。

</details>

### 题目 2：以下代码有什么问题？如何修复？

```cpp
// 着色器中：uniform sampler2D uTex;
glActiveTexture(GL_TEXTURE3);
glBindTexture(GL_TEXTURE_2D, myTexture);
glUniform1i(locTex, 3);
glActiveTexture(GL_TEXTURE0);  // 切回默认

// 绘制
glDrawArrays(GL_TRIANGLES, 0, 3);
```

<details>
<summary>查看答案</summary>

代码**本身没有错误**——sampler 指向单元 3，纹理也绑定在单元 3，逻辑上是正确的。

但有一个**潜在陷阱**：`glActiveTexture(GL_TEXTURE0)` 切回默认后，如果后续代码在不知情的情况下调用 `glBindTexture(GL_TEXTURE_2D, otherTex)` 绑定到单元 0，不会影响单元 3 上的纹理——这是安全的。

然而，如果后续代码错误地认为"当前活跃单元"还是 3 并绑定新纹理，就会把 `myTexture` 从单元 3 上替换掉，导致采样结果错误。

**最佳实践：** 保持 sampler 编号从 0 开始连续分配（0, 1, 2...），减少混乱。不需要使用跳跃编号（如 3）除非有特殊原因。

</details>

### 题目 3：YUV 三平面采样中，为什么 sampler 与纹理单元错位会导致"颜色像坏掉"？

<details>
<summary>查看答案</summary>

YUV 到 RGB 的转换公式是固定的数学关系：

```
R = Y + 1.402 × (V - 128)
G = Y - 0.344 × (U - 128) - 0.714 × (V - 128)
B = Y + 1.772 × (U - 128)
```

Y 表示亮度，U/V 表示色差。如果 sampler 错位（例如把 U 平面当作 Y 平面采样），转换矩阵的输入完全错乱：

- Y 通道收到的是 U 的数据（色差当亮度→图像偏灰/偏色）
- U 通道收到的是 V 的数据（色差信号互换→颜色偏移）

结果表现为严重的偏绿、偏紫、对比度异常。

**解决方法：** 建立固定的映射规则（Y→单元0, U→单元1, V→单元2），初始化时打印各 sampler 的 location 和绑定单元进行核对。

</details>

---

## 下一步

下一节 [纹理基础与参数配置](../04-纹理与采样/01-纹理基础与参数配置.md) 将讲解纹理对象的创建、过滤模式、环绕模式与 Mipmap，把本节学到的 `sampler2D` 和 Uniform 基础直接应用到纹理渲染中。
