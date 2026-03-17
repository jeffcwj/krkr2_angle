## 5. Uniform 传值：CPU 到 GPU 的关键桥梁

`uniform` 是 CPU 每帧（或按需）传给 GPU 的只读参数。典型内容：矩阵、颜色、时间、纹理单元编号。

### 5.1 Uniform 基本规则

- 必须在 `glUseProgram(program)` 后设置。
- `glGetUniformLocation` 建议在初始化阶段缓存。
- 未被着色器使用的 uniform 可能被编译器优化掉，location 会是 `-1`。

### 5.2 代码示例 6：矩阵与颜色 Uniform 传值

```cpp
// 假设 program 已经链接成功
glUseProgram(program); // 先激活 program

GLint locMVP = glGetUniformLocation(program, "uMVP");       // 查询矩阵位置
GLint locTint = glGetUniformLocation(program, "uTintColor"); // 查询颜色位置
GLint locTex0 = glGetUniformLocation(program, "uTex0");      // 查询采样器位置

if (locMVP >= 0) {
    glUniformMatrix4fv(locMVP, 1, GL_FALSE, mvpPtr);          // 传 4x4 矩阵
}
if (locTint >= 0) {
    glUniform4f(locTint, 1.0f, 0.85f, 0.85f, 1.0f);           // 传 RGBA 染色
}
if (locTex0 >= 0) {
    glActiveTexture(GL_TEXTURE0);                               // 激活纹理单元 0
    glBindTexture(GL_TEXTURE_2D, texId);                        // 绑定纹理对象
    glUniform1i(locTex0, 0);                                    // 告诉 sampler 用单元 0
}
```

### 5.3 多纹理 Uniform：对应 YUV 三平面

YUV 方案里常写成：

- `CC_Texture0 -> GL_TEXTURE0`（Y）
- `CC_Texture1 -> GL_TEXTURE1`（U）
- `CC_Texture2 -> GL_TEXTURE2`（V）

CPU 侧只要保证纹理绑定顺序和 `sampler2D` location 对应一致即可。

## 6. GLSL 版本与平台差异：Desktop OpenGL vs OpenGL ES

### 6.1 版本对照速查表

| 场景 | 版本声明 | 输入输出关键字 | 片段输出 | 采样函数 |
|---|---|---|---|---|
| OpenGL ES 2.0 | `#version 100` | `attribute`/`varying` | `gl_FragColor` | `texture2D` |
| OpenGL ES 3.0 | `#version 300 es` | `in`/`out` | 自定义 `out` | `texture` |
| Desktop OpenGL 3.3 | `#version 330 core` | `in`/`out` + `layout` | 自定义 `out` | `texture` |

### 6.2 预处理器技巧：`#ifdef GL_ES` 与 `precision`

在跨平台项目里，常把同一份 shader 兼容 Desktop 和 ES：

```glsl
// 代码示例 7：跨平台预处理头
#ifdef GL_ES
precision mediump float;  // ES 片段阶段必须关注精度
#define VARYING varying
#define ATTR attribute
#else
#define VARYING varying
#define ATTR attribute
#endif

// 对于 #version 100 时代，这种写法可减少平台分叉
ATTR vec3 aPos;           // 顶点输入
VARYING vec2 vUV;         // 阶段间插值
```

对于 ES 3.0/Desktop 3.3，通常直接维护两份简短 shader，更清晰、可读性更高。

### 6.3 RenderManager_ogl.cpp 的跨平台思路

项目中 `RenderManager_ogl.cpp` 会检查扩展（如 `GL_EXT_shader_framebuffer_fetch`），并按平台动态启用能力。

这告诉我们：

1. 不能假设所有设备都支持同一扩展。
2. 着色器效果要准备降级路径。
3. 稳定性优先于“炫技特性”。

## 7. 常见错误与解决方案（至少先会这 3 类）

### 错误 A：精度问题导致颜色抖动或偏色

**现象：** 仅移动端出现带状色阶、颜色偏灰、某些渐变断层。  
**原因：** 片段着色器使用 `lowp`/`mediump` 时精度不足。  
**解决：** 对关键路径改 `highp`（若设备支持），或重构计算避免大范围乘加。

```glsl
#ifdef GL_ES
precision highp float; // 对颜色变换/坐标变换敏感时提高精度
#endif
```

### 错误 B：varying/in/out 不匹配导致链接失败

**现象：** 编译成功但 Program 链接失败。  
**原因：** 顶点 `out vec2 vUV;`，片段写成 `in vec3 vUV;` 或名称不同。  
**解决：** 保证类型、名称、数组维度一致；版本语法一致。

### 错误 C：编译失败但没有任何输出

**现象：** 屏幕全黑，只知道“失败”，不知道哪一行错。  
**原因：** 没有调用 `glGetShaderInfoLog` / `glGetProgramInfoLog`。  
**解决：** 编译和链接后立即打印日志，并附源码版本号与平台信息。

### 错误 D：Sampler 绑定错纹理单元

**现象：** 画面纹理错位、YUV 颜色严重异常。  
**原因：** `glUniform1i(uTex1, 1)` 但实际绑定在 `GL_TEXTURE2`。  
**解决：** 建立“单元号-语义”映射表，初始化时一次性绑定并打印日志。

## 动手实践

按下面步骤做一遍，你会真正掌握“从源码到画面”的闭环。

1. 写一组 ES 2.0 shader（`#version 100`），输出纯色。
2. 改成采样纹理（`sampler2D + texture2D`）。
3. 加入 `uMVP`，让四边形可平移缩放。
4. 把 shader 改写为 ES 3.0（`#version 300 es` + `in/out`）。
5. 在 Desktop 上用 `#version 330 core` 跑通同样功能。
6. 故意制造一个错误（比如 varying 类型不匹配），用 InfoLog 定位并修复。

建议至少记录：GLSL 版本、编译日志、链接日志，便于跨设备复现。

## 对照项目源码

以下文件是本节最关键的参考。

- `krkr2/cpp/core/environ/cocos2d/YUVSprite.cpp` 第 7-38 行：YUV 片段着色器源码，包含 `#ifdef GL_ES`、`precision`、`texture2D`、`gl_FragColor`。
- `krkr2/cpp/core/environ/cocos2d/YUVSprite.cpp` 第 48-53 行：`initWithByteArrays` 之后 `link` 与 `updateUniforms`，对应编译/链接后的程序初始化。
- `krkr2/cpp/core/environ/cocos2d/YUVSprite.cpp` 第 219-235 行：绑定三张纹理并调用 `_glProgramState->apply`，对应 Uniform 与纹理单元生效阶段。
- `krkr2/cpp/core/visual/ogl/RenderManager_ogl.cpp` 第 84-102 行：维护扩展名清单，包含 `GL_EXT_shader_framebuffer_fetch` 等扩展。
- `krkr2/cpp/core/visual/ogl/RenderManager_ogl.cpp` 第 217-221 行：运行时检测 `shader_framebuffer_fetch` 扩展支持，体现跨设备能力探测。

从这两份源码可以得到一个实战结论：**教程里的标准 OpenGL 流程，在项目里只是被 Cocos2d-x 包装了一层，本质完全一致。**

## 本节小结

- GLSL 核心类型要熟：`float/vec2/vec3/vec4/mat4/sampler2D`。
- 变量限定符要按版本写：ES2 用 `attribute/varying`，ES3+ 与 Desktop 用 `in/out/layout`。
- 顶点着色器重点是 `gl_Position` 与 M/V/P 变换；点渲染还会用 `gl_PointSize`。
- 片段着色器重点是颜色计算、纹理采样与 `gl_FragCoord` 的屏幕空间用法。
- 编译与链接必须形成“状态检查 + InfoLog 输出”的固定模板。
- Uniform 传值要做位置缓存，并保证 sampler 与纹理单元严格对应。
- 跨平台不只靠语法兼容，还要考虑精度与扩展支持差异。

## 练习题与答案

### 题目 1：为什么 ES 2.0 着色器常见 `#ifdef GL_ES` 和 `precision`，但 Desktop 330 往往不写？

<details>
<summary>查看答案</summary>

OpenGL ES 面向移动设备，片段着色器精度等级（`lowp/mediump/highp`）是语言与硬件协同的一部分。很多 ES 驱动要求显式精度声明，否则编译报错。Desktop GLSL 一般有默认精度模型，不强制写这类声明。因此跨平台写法常用 `#ifdef GL_ES` 包裹精度声明，避免 Desktop 编译器报“未知限定符”或冗余警告。

</details>

### 题目 2：给定顶点着色器 `out vec2 vUV;`，片段着色器写成 `in vec3 vUV;` 会在什么阶段报错？如何定位？

<details>
<summary>查看答案</summary>

通常在 Program 链接阶段报错，因为跨阶段接口校验发生在链接器。定位方法：在 `glLinkProgram` 后读取 `GL_LINK_STATUS`，失败时调用 `glGetProgramInfoLog` 输出日志。日志会指出 varying/in/out 接口类型不兼容。修复方式是让两端类型、名称、数组维度完全一致。

</details>

### 题目 3：YUV 三平面采样中，为什么 sampler 与纹理单元一旦错位会“颜色看起来像坏掉”？

<details>
<summary>查看答案</summary>

Y、U、V 分量在转换矩阵里有固定物理意义：Y 代表亮度，U/V 代表色差信号。若 sampler 错位（如把 U 当 Y），矩阵输入被打乱，输出 RGB 会严重失真，表现为偏绿、偏紫、对比异常。解决要点是：统一映射表（Y->0, U->1, V->2），并在初始化后打印 location 与绑定单元进行核对。

</details>

## 下一步

下一节：[04-纹理与采样/01-纹理与采样.md](../04-纹理与采样/01-纹理与采样.md)。你会把本节的 `sampler2D` 和 Uniform 基础直接用于纹理过滤、环绕模式、Mipmap 与多纹理混合，进一步接近 KrKr2 的实际渲染路径。
