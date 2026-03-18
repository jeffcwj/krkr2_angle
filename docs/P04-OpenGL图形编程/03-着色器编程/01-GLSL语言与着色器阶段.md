# 着色器编程

> **所属模块：** P04-OpenGL 图形编程  
> **前置知识：** [02-OpenGL基础](../02-OpenGL基础/01-OpenGL基础.md)、线性代数基础（向量与矩阵乘法）  
> **预计阅读时间：** 55 分钟

## 本节目标

读完本节后，你将能够：

1. 解释 GLSL 的核心数据类型与变量限定符，并区分 ES 2.0、ES 3.0、Desktop 3.3 的写法。
2. 独立写出一个可运行的顶点着色器与片段着色器，完成坐标变换、颜色计算与纹理采样。
3. 在 C++ 侧完成着色器编译、链接、Uniform 传值、错误日志输出的完整流程。
4. 使用 `#ifdef GL_ES` 与 `precision` 做跨平台兼容，避免移动端精度与版本陷阱。
5. 排查常见着色器问题：编译失败、varying 不匹配、精度导致的颜色异常。

## 1. GLSL 语言基础：类型、限定符与版本语法

### 1.1 为什么着色器语言和 C++ 不一样

GLSL（OpenGL Shading Language）是运行在 GPU 上的语言。它的目标不是通用编程，而是高吞吐并行计算。

- 你写一份顶点着色器代码，GPU 会对每个顶点并行执行。
- 你写一份片段着色器代码，GPU 会对每个片段（fragment，即光栅化后的像素候选者——一个顶点组成的三角形覆盖屏幕上的每个像素位置都会产生一个片段，片段着色器决定该位置的最终颜色）并行执行。
- 因为执行模型不同，GLSL 提供了向量、矩阵、采样器等图形专用类型。

### 1.2 必会数据类型

| 类型 | 含义 | 常见用途 |
|---|---|---|
| `float` | 浮点标量 | 透明度、时间、系数 |
| `vec2` | 2 维浮点向量 | UV 坐标、2D 位移 |
| `vec3` | 3 维浮点向量 | RGB 颜色、法线 |
| `vec4` | 4 维浮点向量 | RGBA、齐次坐标 |
| `mat4` | 4x4 浮点矩阵 | 模型/视图/投影变换 |
| `sampler2D` | 二维纹理采样器 | `texture`/`texture2D` 取样 |

示例：GLSL 中常见构造方式。

```glsl
float a = 0.5;                   // 标量
vec2 uv = vec2(0.25, 0.75);      // 二维向量
vec3 rgb = vec3(1.0, 0.4, 0.2);  // 三维颜色
vec4 rgba = vec4(rgb, 1.0);      // 由 vec3 + alpha 组合
mat4 mvp = mat4(1.0);            // 单位矩阵
```

### 1.3 变量限定符：老语法与新语法

GLSL 最大的易错点之一就是版本差异。

#### ES 2.0 / GLSL ES 100（旧写法）

- 顶点输入：`attribute`
- 顶点到片段插值：`varying`
- 外部常量：`uniform`
- 片段输出：内置 `gl_FragColor`

#### ES 3.0 / GLSL ES 300 与 Desktop 330（新写法）

- 顶点输入：`in`
- 阶段间插值：顶点 `out` + 片段 `in`
- 外部常量：`uniform`
- 片段输出：自定义 `out vec4 FragColor;`
- 位置绑定：`layout(location = N)`

### 1.4 代码示例 1：ES 2.0（#version 100）最小着色器对

```glsl
// 顶点着色器：OpenGL ES 2.0
// 文件内字符串通常由 C++ 传入 glShaderSource
#version 100
attribute vec3 aPos;        // 顶点位置
attribute vec2 aTexCoord;   // 顶点纹理坐标
varying vec2 vTexCoord;     // 传给片段着色器
uniform mat4 uMVP;          // 模型-视图-投影矩阵

void main() {
    gl_Position = uMVP * vec4(aPos, 1.0); // 计算裁剪空间坐标（clip space，经 MVP 变换后的坐标，GPU 据此裁剪不可见部分）
    vTexCoord = aTexCoord;                // 透传 UV
}
```

```glsl
// 片段着色器：OpenGL ES 2.0
#version 100
#ifdef GL_ES
precision mediump float;    // ES 片段着色器通常必须声明精度
#endif
varying vec2 vTexCoord;     // 接收顶点着色器插值结果
uniform sampler2D uTex0;    // 纹理采样器

void main() {
    vec4 texColor = texture2D(uTex0, vTexCoord); // ES 2.0 使用 texture2D
    gl_FragColor = texColor;                     // 输出最终颜色
}
```

### 1.5 代码示例 2：ES 3.0 / Desktop 3.3 新语法

```glsl
// 顶点着色器：GLSL ES 300
#version 300 es
layout(location = 0) in vec3 aPos;      // 指定位置槽 0
layout(location = 1) in vec2 aTexCoord; // 指定位置槽 1
out vec2 vTexCoord;                     // 传递到片段阶段
uniform mat4 uMVP;                      // 变换矩阵

void main() {
    gl_Position = uMVP * vec4(aPos, 1.0); // 计算裁剪空间坐标
    vTexCoord = aTexCoord;                // 传递 UV
}
```

```glsl
// 片段着色器：GLSL ES 300
#version 300 es
precision mediump float;      // ES 3.0 仍建议显式声明
in vec2 vTexCoord;            // 从顶点阶段接收
uniform sampler2D uTex0;      // 采样器
out vec4 FragColor;           // 自定义片段输出

void main() {
    FragColor = texture(uTex0, vTexCoord); // ES 3.0 使用 texture
}
```

Desktop 版只需把版本改为 `#version 330 core`，并移除 `precision` 语句。

## 2. 顶点着色器：内置变量与坐标变换

### 2.1 gl_Position 与 gl_PointSize

- `gl_Position`：顶点着色器必须写入，表示裁剪空间坐标（clip space coordinate，即经过模型/视图/投影变换后的四维齐次坐标，GPU 会根据该值判断顶点是否在可视范围内）。
- `gl_PointSize`：当你绘制 `GL_POINTS` 时有效，控制点精灵大小。

如果忘记写 `gl_Position`，程序一般会链接失败或渲染异常。

### 2.2 模型、视图、投影矩阵（M/V/P）

经典链路如下：

```
模型空间 --(Model)--> 世界空间 --(View)--> 观察空间 --(Projection)--> 裁剪空间
```

2D 项目常把 `View` 固定为单位矩阵，但 `Model` 与 `Projection` 仍很关键。

### 2.3 代码示例 3：带 gl_PointSize 的顶点着色器

```glsl
// 该示例用于点渲染调试
#version 330 core
layout(location = 0) in vec3 aPos;   // 顶点位置
uniform mat4 uModel;                 // 模型矩阵
uniform mat4 uView;                  // 视图矩阵
uniform mat4 uProj;                  // 投影矩阵
uniform float uPointSize;            // 点大小（像素）

void main() {
    mat4 mvp = uProj * uView * uModel;     // 注意矩阵乘法顺序
    gl_Position = mvp * vec4(aPos, 1.0);   // 写入必须输出
    gl_PointSize = uPointSize;             // 仅 GL_POINTS 生效
}
```

### 2.4 坐标变换实战提醒

1. 传入 C++ 端矩阵时要确认列主序与库约定一致。
2. `Projection` 错误会导致图形“看不见”或拉伸。
3. 2D 正交投影常用范围：左上原点映射到 NDC（Normalized Device Coordinates，标准化设备坐标，即裁剪空间经过透视除法后各轴归一化到 [-1, 1] 的坐标空间）。

## 3. 片段着色器：颜色计算、纹理采样与内置变量

### 3.1 gl_FragColor 与 gl_FragCoord

- `gl_FragColor`：旧版本输出颜色（ES 2.0 / 部分兼容配置）。
- `gl_FragCoord`：当前片段窗口坐标，常用于屏幕空间效果。

### 3.2 颜色计算常见模式

- 纹理色 * 顶点色（2D 引擎最常见）
- 基础色 + 时间函数（闪烁、呼吸）
- 使用 `mix` 在两个颜色间插值

### 3.3 代码示例 4：基于 gl_FragCoord 的条纹叠加

```glsl
#version 330 core
in vec2 vTexCoord;                    // 从顶点着色器传来的 UV
uniform sampler2D uTex0;              // 基础纹理
uniform vec3 uTint;                   // 颜色染色
out vec4 FragColor;                   // 输出颜色

void main() {
    vec4 baseColor = texture(uTex0, vTexCoord);          // 采样纹理
    float stripe = step(0.5, fract(gl_FragCoord.y * 0.1)); // 屏幕空间条纹
    vec3 mixed = mix(baseColor.rgb, baseColor.rgb * uTint, 0.5); // 颜色混合
    mixed *= mix(1.0, 0.85, stripe);                     // 条纹轻微压暗
    FragColor = vec4(mixed, baseColor.a);               // 保持原 alpha
}
```

### 3.4 结合项目的 YUV 采样思路

KrKr2 的 `YUVSprite.cpp` 里展示了三纹理采样并做 YUV→RGB 的真实流程，这类片段着色器在视频渲染中很常见。

```glsl
// 思路与项目一致：Y/U/V 分别来自不同纹理单元
#version 100
#ifdef GL_ES
precision lowp float;                 // 项目中使用 lowp 以兼容旧设备
#endif
varying vec2 v_texCoord;              // 插值 UV
uniform sampler2D CC_Texture0;        // Y 平面
uniform sampler2D CC_Texture1;        // U 平面
uniform sampler2D CC_Texture2;        // V 平面

void main() {
    vec3 yuv;                                              // 保存 YUV
    yuv.x = texture2D(CC_Texture0, v_texCoord).r - 0.0625; // Y 偏移
    yuv.y = texture2D(CC_Texture1, v_texCoord).r - 0.5;    // U 偏移
    yuv.z = texture2D(CC_Texture2, v_texCoord).r - 0.5;    // V 偏移

    mat3 m = mat3(1.164, 1.164, 1.164,
                  0.0,  -0.392, 2.017,
                  1.596, -0.813, 0.0);                    // BT.601 常见系数
    vec3 rgb = m * yuv;                                    // 转 RGB
    gl_FragColor = vec4(rgb, 1.0);                         // 输出颜色
}
```

## 4. 着色器编译与链接：从字符串到可执行 Program

### 4.1 标准流程

必须掌握的调用顺序：

1. `glCreateShader`
2. `glShaderSource`
3. `glCompileShader`
4. `glCreateProgram`
5. `glAttachShader`
6. `glLinkProgram`

如果某一步失败，后续调用不会自动修复，必须立即打印日志。

### 4.2 代码示例 5：编译、链接与错误日志

```cpp
#include <glad/glad.h>
#include <string>
#include <vector>
#include <iostream>

static GLuint CompileShader(GLenum type, const char* src) {
    GLuint shader = glCreateShader(type);                 // 1) 创建 shader 对象
    glShaderSource(shader, 1, &src, nullptr);             // 2) 绑定源码
    glCompileShader(shader);                              // 3) 编译

    GLint ok = GL_FALSE;
    glGetShaderiv(shader, GL_COMPILE_STATUS, &ok);       // 查询编译状态
    if (ok != GL_TRUE) {
        GLint logLen = 0;
        glGetShaderiv(shader, GL_INFO_LOG_LENGTH, &logLen);
        std::vector<char> log(logLen > 1 ? logLen : 1);
        glGetShaderInfoLog(shader, (GLsizei)log.size(), nullptr, log.data()); // 关键：打印编译日志
        std::cerr << "Shader 编译失败: " << log.data() << std::endl;
        glDeleteShader(shader);                           // 失败后释放对象
        return 0;
    }
    return shader;
}

static GLuint CreateProgram(const char* vsSrc, const char* fsSrc) {
    GLuint vs = CompileShader(GL_VERTEX_SHADER, vsSrc);
    if (!vs) return 0;
    GLuint fs = CompileShader(GL_FRAGMENT_SHADER, fsSrc);
    if (!fs) {
        glDeleteShader(vs);
        return 0;
    }

    GLuint program = glCreateProgram();                  // 4) 创建 program
    glAttachShader(program, vs);                         // 5) 附加顶点着色器
    glAttachShader(program, fs);                         // 6) 附加片段着色器
    glLinkProgram(program);                              // 7) 链接

    GLint linked = GL_FALSE;
    glGetProgramiv(program, GL_LINK_STATUS, &linked);    // 查询链接状态
    if (linked != GL_TRUE) {
        GLint logLen = 0;
        glGetProgramiv(program, GL_INFO_LOG_LENGTH, &logLen);
        std::vector<char> log(logLen > 1 ? logLen : 1);
        glGetProgramInfoLog(program, (GLsizei)log.size(), nullptr, log.data()); // 关键：打印链接日志
        std::cerr << "Program 链接失败: " << log.data() << std::endl;
        glDeleteProgram(program);
        program = 0;
    }

    glDeleteShader(vs);                                  // 链接后可删除 shader 对象
    glDeleteShader(fs);
    return program;
}
```

> 在 KrKr2 的 Cocos2d-x 封装中，你会看到 `GLProgram::initWithByteArrays`、`link`、`updateUniforms` 这样的封装接口，本质仍是上述 GL 原生步骤。

### 4.3 编译错误排查最短路径

1. 检查 `#version` 是否和当前上下文兼容。
2. 检查旧/新语法是否混用（如 `attribute` + `#version 330`）。
3. 检查顶点 `out` 与片段 `in` 的类型/名称是否一致。
4. 立即打印 `glGetShaderInfoLog` 与 `glGetProgramInfoLog`，不要猜。

## 常见错误及解决方案

### 错误 1：`#version` 与上下文不匹配

**现象：** 着色器编译失败，日志提示 `version '330' is not supported` 或 `#version must be first line`。

**原因：** 在 OpenGL ES 2.0 设备上使用了 `#version 330 core`，或源码字符串开头有 BOM / 空行导致 `#version` 不在第一行。

**解决方案：**
```cpp
// 根据当前上下文动态选择版本头
const char* versionHeader;
#ifdef GL_ES_VERSION_2_0
    versionHeader = "#version 100\n";       // ES 2.0
#elif defined(GL_ES_VERSION_3_0)
    versionHeader = "#version 300 es\n";    // ES 3.0
#else
    versionHeader = "#version 330 core\n";  // Desktop GL 3.3+
#endif
// 拼接版本头与着色器正文
std::string fullSrc = std::string(versionHeader) + shaderBody;
```

### 错误 2：顶点 `out` 与片段 `in` 名称/类型不匹配

**现象：** 编译通过但链接失败，日志出现 `varying variable ... not found` 或 `type mismatch`。片段着色器收到全黑或随机值。

**原因：** 顶点着色器声明 `out vec3 vColor;`，片段着色器写成了 `in vec4 vColor;`（类型不同）或 `in vec3 vertexColor;`（名字不同）。

**解决方案：** 保证名称、类型、精度限定完全一致。推荐把 varying 声明放在公共 `#include` 头文件中：
```glsl
// common_varying.glsl —— 顶点和片段着色器都引用
// 注意：GLSL 没有标准 #include，需要在 C++ 侧拼接
// 或者使用 GL_ARB_shading_language_include 扩展

// 声明共享变量
vec2 vTexCoord;
vec3 vColor;
```

### 错误 3：ES 片段着色器缺少 `precision` 导致编译失败

**现象：** 桌面端正常，Android/iOS 编译失败，日志提示 `No precision specified for type 'float'`。

**原因：** OpenGL ES 的片段着色器中 `float` 没有默认精度，必须显式声明。

**解决方案：**
```glsl
#version 100
#ifdef GL_ES
precision mediump float;  // 必须加！mediump 适合大多数场景
#endif
// ... 其余代码 ...
```

## 本节小结

- GLSL 是运行在 GPU 上的着色器语言，核心数据类型包括 `float`、`vec2`/`vec3`/`vec4`、`mat4`、`sampler2D`。
- 变量限定符在 ES 2.0 和 ES 3.0/Desktop 3.3 之间有语法差异：`attribute`/`varying` (旧) → `in`/`out` (新)。
- 顶点着色器的核心输出是 `gl_Position`（裁剪空间坐标），片段着色器的核心输出是颜色值（`gl_FragColor` 或自定义 `out`）。
- 模型/视图/投影（MVP）矩阵链完成从模型空间到裁剪空间的坐标变换。
- 着色器编译必须检查 `GL_COMPILE_STATUS`，链接必须检查 `GL_LINK_STATUS`，失败时**立即打印日志**。
- ES 片段着色器必须显式声明 `precision`，否则在移动端编译失败。
- KrKr2 的 `YUVSprite.cpp` 展示了三纹理 YUV→RGB 采样的真实片段着色器模式。

## 练习题与答案

### 题目 1：将 ES 2.0 着色器升级为 ES 3.0 语法

以下是一对 ES 2.0 着色器代码，请将其改写为 ES 3.0（`#version 300 es`）语法，保持功能不变：

```glsl
// 顶点着色器 (ES 2.0)
#version 100
attribute vec3 aPos;
attribute vec2 aUV;
varying vec2 vUV;
uniform mat4 uMVP;
void main() {
    gl_Position = uMVP * vec4(aPos, 1.0);
    vUV = aUV;
}
```

```glsl
// 片段着色器 (ES 2.0)
#version 100
precision mediump float;
varying vec2 vUV;
uniform sampler2D uTex;
void main() {
    gl_FragColor = texture2D(uTex, vUV);
}
```

<details>
<summary>查看答案</summary>

```glsl
// 顶点着色器 (ES 3.0)
#version 300 es
layout(location = 0) in vec3 aPos;       // attribute → in，并加 layout
layout(location = 1) in vec2 aUV;        // attribute → in
out vec2 vUV;                            // varying → out
uniform mat4 uMVP;                       // uniform 不变
void main() {
    gl_Position = uMVP * vec4(aPos, 1.0);
    vUV = aUV;
}
```

```glsl
// 片段着色器 (ES 3.0)
#version 300 es
precision mediump float;                  // ES 仍需精度声明
in vec2 vUV;                              // varying → in
uniform sampler2D uTex;                   // uniform 不变
out vec4 FragColor;                       // 自定义输出替代 gl_FragColor
void main() {
    FragColor = texture(uTex, vUV);       // texture2D → texture
}
```

**改动要点：**
1. `#version 100` → `#version 300 es`
2. `attribute` → `in`（顶点输入）
3. `varying` → `out`（顶点输出）/ `in`（片段输入）
4. `gl_FragColor` → 自定义 `out vec4 FragColor`
5. `texture2D()` → `texture()`
6. 可选加 `layout(location = N)` 精确绑定属性位置

</details>

### 题目 2：分析 KrKr2 YUV 着色器的精度选择

在本节代码示例 4 中，KrKr2 的 YUV 片段着色器使用了 `precision lowp float` 而非 `mediump`。回答以下问题：

1. `lowp`、`mediump`、`highp` 三种精度在 ES 规范中的**最低保证精度范围**分别是什么？
2. `lowp` 能正确表示 YUV 到 RGB 转换系数（如 `1.164`、`-0.392`）吗？可能有什么问题？
3. 如果在高分辨率屏幕（如 4K）上发现颜色出现条带（banding），应该如何改？

<details>
<summary>查看答案</summary>

1. **精度范围（ES 规范最低保证）：**
   - `lowp`：范围 [-2, 2]，精度 1/256（8 位定点或等价浮点）
   - `mediump`：范围 [-2¹⁴, 2¹⁴]，精度 1/1024（10 位尾数浮点或等价）
   - `highp`：范围 [-2⁶², 2⁶²]，精度 1/65536（IEEE 754 float32 等价）

2. **能否表示 YUV 系数：**
   `lowp` 范围是 [-2, 2]，系数 `1.164`、`1.596`、`2.017` 均在此范围内，所以**数值上可表示**。但精度只有 1/256，这意味着颜色分量计算精度约为 8 位。对于视频帧来说勉强够用（视频源本身通常是 8-bit YUV），但中间运算的**累积精度误差**可能导致颜色偏差。尤其是 `2.017` 接近 `lowp` 范围上限，乘法后容易溢出或精度损失。

3. **解决色带问题：**
   将 `precision lowp float` 改为 `precision mediump float`。这将精度提升到 10+ 位，足以消除大多数可见色带。如果仍有问题（极端情况），可在特定变量上声明 `highp`：
   ```glsl
   precision mediump float;
   // 对 YUV 转换矩阵使用更高精度
   highp mat3 m = mat3(1.164, 1.164, 1.164,
                        0.0,  -0.392, 2.017,
                        1.596, -0.813, 0.0);
   highp vec3 rgb = m * yuv;
   ```

</details>

### 题目 3：着色器编译错误排查

以下代码在 Desktop GL 3.3 上编译失败。找出所有错误并修复：

```glsl
#version 330 core
attribute vec3 aPos;          // ← 错误 1
varying vec2 vTexCoord;       // ← 错误 2
uniform mat4 uMVP;

void main() {
    gl_Position = uMVP * vec4(aPos, 1.0);
    vTexCoord = vec2(0.0, 0.0);
}
```

```glsl
#version 330 core
precision mediump float;      // ← 错误 3
varying vec2 vTexCoord;       // ← 错误 4

void main() {
    gl_FragColor = vec4(1.0);  // ← 错误 5
}
```

<details>
<summary>查看答案</summary>

**错误分析与修复：**

| 行 | 错误 | 原因 | 修复 |
|---|---|---|---|
| VS: `attribute` | 旧语法用在 `#version 330 core` | 330 core 不支持 `attribute` | 改为 `layout(location = 0) in vec3 aPos;` |
| VS: `varying` | 旧语法用在 `#version 330 core` | 330 core 不支持 `varying` | 改为 `out vec2 vTexCoord;` |
| FS: `precision mediump float` | Desktop GL 不需要/不支持 | `precision` 是 ES 专用语法 | 删除这行（或用 `#ifdef GL_ES` 包裹） |
| FS: `varying` | 同上，旧语法 | 330 core 不支持 | 改为 `in vec2 vTexCoord;` |
| FS: `gl_FragColor` | 旧内置变量 | 330 core 中已移除 | 改为自定义 `out vec4 FragColor;`，`main()` 中写 `FragColor = ...;` |

**修复后完整代码：**

```glsl
// 顶点着色器（修复后）
#version 330 core
layout(location = 0) in vec3 aPos;
out vec2 vTexCoord;
uniform mat4 uMVP;

void main() {
    gl_Position = uMVP * vec4(aPos, 1.0);
    vTexCoord = vec2(0.0, 0.0);
}
```

```glsl
// 片段着色器（修复后）
#version 330 core
in vec2 vTexCoord;
out vec4 FragColor;

void main() {
    FragColor = vec4(1.0);
}
```

</details>

## 下一步

下一节 [Uniform 与平台差异](./02-Uniform与平台差异.md) 将讲解如何从 C++ 侧向着色器传递 Uniform 数据（矩阵、颜色、纹理采样器等），以及不同平台（Desktop GL / ES 2.0 / ES 3.0）在 Uniform 处理上的差异和兼容策略。

