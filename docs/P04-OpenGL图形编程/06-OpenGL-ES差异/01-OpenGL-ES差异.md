# OpenGL ES 差异与兼容开发
> **所属模块：** P04-OpenGL图形编程 ｜ **前置知识：** [05-帧缓冲与后处理](../05-帧缓冲与后处理/01-帧缓冲与后处理.md) ｜ **预计阅读时间：** 90 分钟
## 本节目标
1. 理解 OpenGL ES 1.1、2.0、3.0、3.2 的演进路径。
2. 区分 ES 2.0 与 Desktop OpenGL 3.x Core 的关键差异。
3. 掌握 GLSL `#version 100`、`#version 300 es`、`#version 330 core` 的映射关系。
4. 掌握 `attribute/varying` 到 `in/out` 的迁移方法。
5. 掌握 `highp/mediump/lowp` 的声明时机与性能影响。
6. 理解扩展命名体系 `GL_OES_*`、`GL_EXT_*`、`GL_KHR_*`。
7. 掌握 EGL 与 WGL/GLX/CGL 的上下文创建差异。
8. 掌握 NPOT、纹理上限、压缩格式、VAO、FBO 的兼容策略。
9. 学会写一套同时支持 Desktop GL 与 ES 的渲染兼容层。
10. 对照 KrKr2 实际源码理解工程落地方案。
## 一、版本演进全景：ES 1.1 → ES 2.0 → ES 3.0 → ES 3.2
OpenGL ES 不是桌面 OpenGL 的缩小版。
它是面向移动与嵌入式设备的独立标准路线。
设计重点是低功耗、实现可控、跨设备一致性。
ES 1.1 主要是固定功能时代。
ES 2.0 开始进入可编程管线时代。
ES 3.0 补齐大量现代能力。
ES 3.2 进一步整合高级特性与扩展成果。
### 1.1 ES 1.1 的定位
ES 1.1 提供固定变换与固定光照流程。
开发者通过状态机配置渲染过程。
优点是简单。
缺点是不灵活。
对复杂特效支持有限。
### 1.2 ES 2.0 的转折意义
ES 2.0 移除了固定功能主路径。
开发者必须编写顶点着色器和片元着色器。
没有 `glBegin/glEnd`。
没有固定矩阵栈。
没有固定光照接口。
这迫使渲染架构从“状态机拼接”变为“数据驱动+shader 驱动”。
### 1.3 ES 3.0 的工程价值
ES 3.0 不是简单加函数。
它是移动端工程化能力跃迁。
它让移动端能够实现更稳健的后处理、批量绘制和数据组织。
### 1.4 ES 3.2 的成熟特征
ES 3.2 把更多扩展能力标准化。
减少厂商特定路径带来的维护成本。
为跨平台引擎提供更稳定的能力基线。
### 1.5 版本对照表
| 版本 | 管线模型 | 着色器语义 | 关键能力 |
|---|---|---|---|
| ES 1.1 | 固定功能 | 无 GLSL 强制 | 固定变换、固定光照 |
| ES 2.0 | 可编程起点 | GLSL ES 1.00 | attribute/varying、片元精度声明 |
| ES 3.0 | 可编程增强 | GLSL ES 3.00 | UBO、TBO、MRT、Instancing、Transform Feedback |
| ES 3.2 | 高级整合 | GLSL ES 3.20 | 更多现代特性并入主规范 |
## 二、ES 2.0 vs Desktop OpenGL 3.x Core 核心差异
同样叫 OpenGL。
但 Desktop 与 ES 的默认假设不同。
错误通常来自“以为完全一致”。
### 2.1 固定管线差异
ES 2.0 完全没有固定管线接口。
Desktop 3.x Core 也不鼓励固定管线。
但桌面环境常有历史兼容上下文。
这会误导开发者形成错误预期。
### 2.2 立即模式差异
ES 2.0 无 `glBegin/glEnd`。
Desktop Core 同样不支持。
但很多老教程来自兼容上下文时代。
迁移时必须统一采用 VBO/IBO/VAO 思维。
### 2.3 精度限定符差异
Desktop GLSL 通常不要求片元默认精度声明。
GLSL ES 片元阶段常要求显式默认精度。
漏写时常见编译失败。
### 2.4 纹理与格式差异
ES 2.0 对 NPOT 纹理、过滤组合、内部格式更严格。
Desktop 驱动通常更宽松。
这导致“桌面正常、手机异常”的典型跨端问题。
### 2.5 功能对照表
| 维度 | Desktop OpenGL 3.x Core | OpenGL ES 2.0 | 兼容建议 |
|---|---|---|---|
| 固定管线 | Core 不推荐 | 不支持 | 全量 shader 化 |
| `glBegin/glEnd` | Core 不支持 | 不支持 | 统一 VBO/IBO 绘制 |
| 片元默认精度 | 通常宽松 | 常需显式声明 | 统一精度模板 |
| NPOT 行为 | 常较宽松 | 受限明显 | 保守采样参数 |
| 扩展入口加载 | WGL/GLX/CGL | EGL 常见 | 封装统一加载器 |
| FBO 完整性 | 驱动差异大 | 规则更严格 | 强制状态检查 |
### 2.6 代码示例 1：ES2 基础绘制模板
```cpp
// 文件：DrawMesh_ES2.cpp
#include <GLES2/gl2.h>
struct Vertex {
    float x, y, z;
    float u, v;
};
void DrawMeshES2(GLuint program, GLuint vbo, GLsizei count) {
    glUseProgram(program); // 绑定程序
    glBindBuffer(GL_ARRAY_BUFFER, vbo); // 绑定顶点缓冲
    GLint aPos = glGetAttribLocation(program, "aPos");
    GLint aUV = glGetAttribLocation(program, "aUV");
    glEnableVertexAttribArray(aPos); // 开启位置属性
    glVertexAttribPointer(aPos, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), (const void*)0); // 位置偏移
    glEnableVertexAttribArray(aUV); // 开启UV属性
    glVertexAttribPointer(aUV, 2, GL_FLOAT, GL_FALSE, sizeof(Vertex), (const void*)(sizeof(float) * 3)); // UV偏移
    glDrawArrays(GL_TRIANGLES, 0, count); // 绘制
    glDisableVertexAttribArray(aPos); // 清理状态
    glDisableVertexAttribArray(aUV);
    glBindBuffer(GL_ARRAY_BUFFER, 0);
}
```
### 2.7 代码示例 2：ES2 下 NPOT 保守策略
```cpp
// 文件：TextureSafe_ES2.cpp
#include <GLES2/gl2.h>
void SetupTextureSafeES2(GLuint tex) {
    glBindTexture(GL_TEXTURE_2D, tex); // 绑定纹理
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE); // 避免 NPOT repeat 风险
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE); // 避免 NPOT repeat 风险
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR); // 保守最小过滤
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR); // 保守最大过滤
}
```
## 三、ES 3.0 新增特性详解
### 3.1 UBO
UBO 用于统一管理 uniform 块。
优势是减少频繁 `glUniform*` 调用。
优势是数据组织更稳定。
### 3.2 TBO
TBO 适合按索引读取大量结构化数据。
常见于骨骼矩阵、查表数据、实例参数。
### 3.3 FBO MRT
MRT 允许一次 pass 输出多个颜色附件。
这是延迟渲染和复杂后处理的基础能力。
### 3.4 Instancing
Instancing 能显著减少重复 draw call。
特别适合草地、粒子、重复网格场景。
### 3.5 Transform Feedback
Transform Feedback 可捕获顶点阶段输出。
有利于 GPU 侧迭代计算。
### 3.6 代码示例 3：UBO 初始化
```cpp
// 文件：SetupUBO_ES30.cpp
#include <GLES3/gl3.h>
void SetupCameraUBO(GLuint program, GLuint& ubo) {
    glGenBuffers(1, &ubo); // 创建 UBO
    glBindBuffer(GL_UNIFORM_BUFFER, ubo);
    glBufferData(GL_UNIFORM_BUFFER, 2 * 16 * sizeof(float), nullptr, GL_DYNAMIC_DRAW); // 预留两个mat4
    GLuint block = glGetUniformBlockIndex(program, "CameraBlock");
    glUniformBlockBinding(program, block, 0); // 绑定到槽位0
    glBindBufferBase(GL_UNIFORM_BUFFER, 0, ubo); // 绑定UBO
    glBindBuffer(GL_UNIFORM_BUFFER, 0);
}
```
### 3.7 代码示例 4：MRT 创建
```cpp
// 文件：CreateMRT_ES30.cpp
#include <GLES3/gl3.h>
bool CreateMRT(GLuint tex0, GLuint tex1, GLuint& fbo) {
    glGenFramebuffers(1, &fbo);
    glBindFramebuffer(GL_FRAMEBUFFER, fbo);
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, tex0, 0); // 输出0
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT1, GL_TEXTURE_2D, tex1, 0); // 输出1
    GLenum bufs[2] = {GL_COLOR_ATTACHMENT0, GL_COLOR_ATTACHMENT1};
    glDrawBuffers(2, bufs); // 启用双输出
    bool ok = (glCheckFramebufferStatus(GL_FRAMEBUFFER) == GL_FRAMEBUFFER_COMPLETE);
    glBindFramebuffer(GL_FRAMEBUFFER, 0);
    return ok;
}
```
### 3.8 代码示例 5：Instancing 绘制
```cpp
// 文件：DrawInstanced_ES30.cpp
#include <GLES3/gl3.h>
void DrawInstanced(GLuint vao, GLsizei indexCount, GLsizei instanceCount) {
    glBindVertexArray(vao); // 绑定VAO
    glDrawElementsInstanced(GL_TRIANGLES, indexCount, GL_UNSIGNED_SHORT, (const void*)0, instanceCount); // 实例化绘制
    glBindVertexArray(0);
}
```
## 四、GLSL 版本映射与语法迁移
### 4.1 GLSL 版本映射
| 目标平台 | 版本行 | 关键语法 |
|---|---|---|
| ES 2.0 | `#version 100` | `attribute`、`varying`、`gl_FragColor` |
| ES 3.0 | `#version 300 es` | `in`、`out`、显式片元输出 |
| Desktop 3.3 | `#version 330 core` | `in`、`out`、Core 输出 |
### 4.2 attribute/varying 与 in/out
ES2 顶点输入使用 `attribute`。
ES2 跨阶段变量使用 `varying`。
ES3/Desktop 使用 `in/out`。
片元输出从 `gl_FragColor` 迁移到自定义 `out vec4`。
### 4.3 代码示例 6：ES2 shader
```glsl
// 文件：shader_es2.vert
#version 100
attribute vec3 aPos;      // 顶点位置
attribute vec2 aUV;       // 顶点UV
varying vec2 vUV;         // 输出UV
uniform mat4 uMVP;        // 变换矩阵
void main() {
    vUV = aUV;            // 传递UV
    gl_Position = uMVP * vec4(aPos, 1.0); // 输出位置
}
```
```glsl
// 文件：shader_es2.frag
#version 100
precision mediump float;  // ES2 默认片元精度
varying vec2 vUV;         // 输入UV
uniform sampler2D uTex;   // 纹理
void main() {
    gl_FragColor = texture2D(uTex, vUV); // 输出颜色
}
```
### 4.4 代码示例 7：ES3 与 Desktop 片元
```glsl
// 文件：shader_es3.frag
#version 300 es
precision mediump float;  // ES3 默认精度
in vec2 vUV;              // 输入UV
uniform sampler2D uTex;   // 纹理
out vec4 FragColor;       // 显式输出
void main() {
    FragColor = texture(uTex, vUV); // 输出采样结果
}
```
```glsl
// 文件：shader_gl330.frag
#version 330 core
in vec2 vUV;              // 输入UV
uniform sampler2D uTex;   // 纹理
out vec4 FragColor;       // 输出
void main() {
    FragColor = texture(uTex, vUV); // Desktop输出
}
```
## 五、精度限定符：highp / mediump / lowp
### 5.1 何时必须声明
GLSL ES 片元着色器通常需要默认精度声明。
推荐统一写 `precision mediump float;`。
### 5.2 各平台默认精度实践
Android 设备差异大。
建议始终显式声明。
iOS 虽然相对一致。
也建议显式声明保持统一行为。
Desktop 通常不依赖 ES 默认精度规则。
### 5.3 精度对性能影响
`highp` 精度高但代价通常更大。
`mediump` 是多数移动场景的折中方案。
`lowp` 适合低动态范围颜色逻辑。
### 5.4 代码示例 8：跨平台精度宏
```glsl
// 文件：precision_common.glsl
#ifdef GL_ES
    precision mediump float;      // ES默认精度
    #define TVP_HIGHP highp
    #define TVP_MEDIUMP mediump
    #define TVP_LOWP lowp
#else
    #define TVP_HIGHP
    #define TVP_MEDIUMP
    #define TVP_LOWP
#endif
TVP_MEDIUMP vec2 gUV;             // 统一精度写法
```
## 六、扩展机制与上下文创建差异
### 6.1 扩展命名体系
常见扩展前缀包括 `GL_OES_*`、`GL_EXT_*`、`GL_KHR_*`。
同一能力可能有多个后缀实现。
应做多名称回退探测。
### 6.2 代码示例 9：扩展检测与函数加载
```cpp
// 文件：ExtProbe.cpp
#include <string>
#include <unordered_set>
#include <GLES2/gl2.h>
#include <EGL/egl.h>
static std::unordered_set<std::string> gExtSet; // 扩展集合
void BuildExtSet() {
    const char* ext = (const char*)glGetString(GL_EXTENSIONS);
    if (!ext) return;
    std::string all(ext);
    size_t p = 0;
    while (p < all.size()) {
        size_t n = all.find(' ', p);
        if (n == std::string::npos) n = all.size();
        if (n > p) gExtSet.emplace(all.substr(p, n - p)); // 保存扩展名
        p = n + 1;
    }
}
bool HasExt(const char* name) {
    return gExtSet.find(name) != gExtSet.end();
}
using PFN_CopyImage = void(*)(GLuint, GLenum, GLint, GLint, GLint, GLint,
                              GLuint, GLenum, GLint, GLint, GLint, GLint,
                              GLsizei, GLsizei, GLsizei);
PFN_CopyImage LoadCopyImage() {
    if (HasExt("GL_EXT_copy_image")) return (PFN_CopyImage)eglGetProcAddress("glCopyImageSubDataEXT");
    if (HasExt("GL_OES_copy_image")) return (PFN_CopyImage)eglGetProcAddress("glCopyImageSubDataOES");
    return nullptr; // 不支持则回退
}
```
### 6.3 EGL vs WGL/GLX/CGL
OpenGL 函数负责绘制。
上下文 API 负责创建与绑定上下文。
Windows 常见 WGL。
Linux 常见 GLX 或 EGL。
macOS 传统路径是 CGL。
Android 主路径是 EGL。
## 七、纹理限制、VAO 与 FBO
### 7.1 NPOT 与纹理上限
ES2 下 NPOT 在 Repeat/mipmap 组合容易异常。
`GL_MAX_TEXTURE_SIZE` 在设备间差异巨大。
应运行时读取并做资源策略。
### 7.2 压缩格式差异
Desktop 常见 BC/S3TC 生态。
移动常见 ETC/ETC2/PVRTC/ASTC。
应动态枚举 `GL_COMPRESSED_TEXTURE_FORMATS`。
### 7.3 VAO 在 ES2 的特殊性
ES3 原生支持 VAO。
ES2 依赖 `OES_vertex_array_object` 扩展。
必须保留无 VAO 回退路径。
### 7.4 FBO 在 ES 的限制
ES 下 FBO 完整性规则更严格。
通常需要有效 color attachment。
depth/stencil 格式组合要符合规范。
每次创建后都要 `glCheckFramebufferStatus`。
### 7.5 代码示例 10：FBO 检查模板
```cpp
// 文件：CreateFBO_ES.cpp
#include <GLES2/gl2.h>
bool CreateFBO(GLuint colorTex, GLuint depthRb, GLuint& fbo) {
    glGenFramebuffers(1, &fbo); // 创建FBO
    glBindFramebuffer(GL_FRAMEBUFFER, fbo); // 绑定FBO
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, colorTex, 0); // 颜色附件
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_RENDERBUFFER, depthRb); // 深度附件
    GLenum st = glCheckFramebufferStatus(GL_FRAMEBUFFER); // 检查完整性
    glBindFramebuffer(GL_FRAMEBUFFER, 0); // 解绑
    return st == GL_FRAMEBUFFER_COMPLETE;
}
```
## 八、兼容编码策略与 KrKr2 实战分析
### 8.1 同时支持 Desktop 与 ES 的策略
建议三层架构：
第一层：上下文与函数入口加载层。
第二层：能力探测层（版本、扩展、上限）。
第三层：渲染实现层（读取能力开关）。
这种结构可以显著降低平台分支污染。
### 8.2 常见错误及解决方案
错误一：片元 shader 缺默认精度声明导致编译失败。
解决：统一注入 `precision mediump float;`。
错误二：NPOT + Repeat + mipmap 在 ES2 采样异常。
解决：降级 `CLAMP_TO_EDGE` 并关闭 mipmap。
错误三：扩展函数未检测直接调用导致崩溃。
解决：先检测扩展，再加载函数并判空。
### 8.3 代码示例 11：能力结构体
```cpp
// 文件：GLCaps.h
#pragma once
struct GLCaps {
    bool isGLES = false;            // 是否ES
    bool hasVAO = false;            // VAO能力
    bool hasCopyImage = false;      // copy image扩展
    bool hasUnpackSubimage = false; // unpack扩展
    int maxTextureSize = 0;         // 最大纹理尺寸
};
inline bool NeedNPOTFallback(const GLCaps& caps) {
    return caps.isGLES;             // ES默认保守策略
}
```
### 8.4 KrKr2 中的实际兼容策略
`RenderManager_ogl.cpp` 里的关键点：
1. 扩展名集中声明并统一管理。
2. 扩展字符串解析后放入集合。
3. 平台级函数加载分流到 `wglGetProcAddress/eglGetProcAddress`。
4. 扩展函数按多后缀回退加载。
5. 压缩纹理格式运行时枚举。
6. 读取 `GL_MAX_TEXTURE_SIZE` 作为资源策略边界。
`YUVSprite.cpp` 里的关键点：
1. shader 字符串中 `#ifdef GL_ES` + 精度声明。
2. VAO 路径可选。
3. 无 VAO 回退路径保留。
通过项目检索 `GL_ES|GLES` 相关文件：
- `krkr2/cpp/core/visual/ogl/RenderManager_ogl.cpp`
- `krkr2/cpp/core/visual/ogl/ogl_common.h`
- `krkr2/cpp/core/environ/cocos2d/YUVSprite.cpp`
- `krkr2/cpp/core/visual/impl/LayerImpl.cpp`
- `krkr2/cpp/core/visual/impl/BasicDrawDevice.cpp`
- `krkr2/cpp/core/movie/ffmpeg/VideoRenderer.cpp`
## 动手实践
1. 启动时打印 `GL_VERSION`、`GL_RENDERER`、`GL_VENDOR`、`GL_MAX_TEXTURE_SIZE`。
2. 构建扩展集合并检测 `GL_EXT_unpack_subimage`、`GL_OES_vertex_array_object`、`GL_EXT_copy_image`。
3. 给所有 ES 片元 shader 统一注入默认精度声明。
4. 为 ES2 的 NPOT 纹理添加 `CLAMP_TO_EDGE` 与非 mipmap 降级路径。
5. 封装 FBO 完整性检查函数并记录状态码。
6. 为 VAO 加入扩展缺失时的手动属性绑定回退。
7. 输出 capability dump 给 QA 做机型差异验证。
## 对照项目源码
- `krkr2/cpp/core/visual/ogl/RenderManager_ogl.cpp`：
  - 84-103 行：扩展名声明。
  - 114-147 行：扩展集合初始化。
  - 153-162 行：函数加载类型定义。
  - 211-215 行：平台函数绑定。
  - 228-264 行：扩展函数多后缀回退加载。
  - 193-208 行：压缩纹理格式枚举。
  - 303-307 行：`GL_MAX_TEXTURE_SIZE` 查询。
- `krkr2/cpp/core/environ/cocos2d/YUVSprite.cpp`：
  - 8-13 行：ES 条件精度声明。
  - 64-112 行：VAO 路径。
  - 231-257 行：无 VAO 回退路径。
- `krkr2/cpp/core/visual/ogl/ogl_common.h`：
  - 18-27 行：iOS/Android GLES 与 EGL 头文件选择。
## 本节小结
- OpenGL ES 的演进主线是固定功能到可编程再到现代能力补齐。
- ES2 与 Desktop 的核心差异集中在固定管线、精度规则、纹理限制和扩展机制。
- ES3 提供的 UBO/TBO/MRT/Instancing/TF 是移动端现代渲染基础。
- GLSL 迁移关键是版本声明、语法关键字、片元输出语义同步迁移。
- 精度限定符必须纳入工程规范，不可临时处理。
- KrKr2 展示了运行时能力探测与回退路径的工程化范式。
## 练习题与答案（≥3题，答案用 <details> 折叠）
### 题目 1
为什么 ES2 片元 shader 常写 `precision mediump float;`？
<details>
<summary>查看答案</summary>
因为 GLSL ES 片元阶段通常要求默认浮点精度声明。
不声明时很多移动设备会在编译阶段报错。
`mediump` 是兼容性与性能之间的常见平衡点。
</details>
### 题目 2
ES2 中如何安全使用 VAO？
<details>
<summary>查看答案</summary>
先检测 `GL_OES_vertex_array_object` 扩展。
可用时启用 VAO 路径。
不可用时回退到手动 `glVertexAttribPointer` 路径。
VAO 应作为优化项而非硬依赖。
</details>
### 题目 3
NPOT 在 ES2 的典型问题与降级方案是什么？
<details>
<summary>查看答案</summary>
典型问题是 NPOT + Repeat + mipmap 在部分设备采样异常。
降级方案是使用 `GL_CLAMP_TO_EDGE` 并关闭 mipmap。
必要时在资源阶段转换为 POT 纹理。
</details>
### 题目 4
为什么 KrKr2 运行时枚举压缩格式而不是写死平台表？
<details>
<summary>查看答案</summary>
同平台下不同 GPU/驱动能力差异很大。
写死平台表容易误判。
运行时枚举 `GL_COMPRESSED_TEXTURE_FORMATS` 才能得到当前设备真实能力。
</details>
## 下一步
下一章：[07-纹理压缩格式](../07-纹理压缩格式/01-纹理压缩格式.md)。
下一节将系统比较 ETC2、PVRTC、ASTC 的画质、体积、性能和平台支持差异。
