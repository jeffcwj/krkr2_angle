# 4.1 RenderManager 架构

## 本节目标

- 理解 `iTVPRenderManager` 接口的设计哲学与职责划分
- 掌握渲染后端的工厂注册机制（`REGISTER_RENDERMANAGER` 宏）
- 了解 OpenGL 后端 `TVPRenderManager_OpenGL` 的初始化流程与 GL 扩展探测
- 学会通过接口抽象实现软件/硬件渲染后端的无缝切换

---

## 4.1.1 为什么需要 RenderManager 抽象层

KrKr2 同时支持 OpenGL 硬件渲染和纯软件渲染两种后端。为了让上层图层系统（`tTJSNI_BaseLayer`）不必关心底层实现细节，引擎在 `RenderManager.h` 中定义了一套纯虚接口体系。这套体系的核心设计原则是**依赖倒转**——上层代码只依赖抽象接口，具体后端通过工厂模式在运行时注入。

这种设计带来三个关键好处：

1. **可移植性**：Android/iOS 使用 OpenGL ES，桌面平台可选 OpenGL 或软件渲染，无需修改上层代码
2. **可测试性**：可以编写 Mock 后端来测试图层逻辑而不依赖 GPU
3. **可扩展性**：未来可添加 Vulkan/Metal 后端而不影响现有代码

接口体系的层次结构如下：

```
┌─────────────────────────────────────────────┐
│           上层代码 (LayerManager)             │
│         只使用 iTVPRenderManager*            │
├─────────────────────────────────────────────┤
│         iTVPRenderManager (纯虚接口)         │
│  ┌──────────────┐  ┌─────────────────────┐  │
│  │iTVPTexture2D │  │ iTVPRenderMethod    │  │
│  │  (纹理抽象)  │  │ (着色器/混合抽象)    │  │
│  └──────────────┘  └─────────────────────┘  │
├──────────────┬──────────────────────────────┤
│  OGL 后端    │     软件渲染后端              │
│  (opengl)    │     (software)               │
└──────────────┴──────────────────────────────┘
```

## 4.1.2 iTVPRenderManager 接口详解

`iTVPRenderManager` 定义在 `visual/RenderManager.h` 第 206-298 行，是整个渲染系统的总入口。我们逐组分析其方法：

### 纹理创建组

引擎提供四种纹理创建方式，覆盖不同的输入源：

```cpp
// 源码位置：RenderManager.h:216-240
class iTVPRenderManager {
public:
    // 方式1：从原始像素数据创建
    // 参数 pitch 为每行字节数，可能大于 w * bpp（存在行对齐填充）
    virtual iTVPTexture2D* CreateTexture2D(
        const void *pixel, int pitch,
        unsigned int w, unsigned int h,
        TVPTextureFormat::e format, int flags) = 0;

    // 方式2：从 tTVPBitmap 对象创建（最常用）
    // tTVPBitmap 是引擎内部的位图封装，已包含尺寸和格式信息
    virtual iTVPTexture2D* CreateTexture2D(
        tTVPBitmap* bmp, int flags) = 0;

    // 方式3：从流创建（用于压缩纹理如 ETC2/PVRTC）
    virtual iTVPTexture2D* CreateTexture2D(
        tTJSBinaryStream* stream, int flags) = 0;

    // 方式4：复制已有纹理
    virtual iTVPTexture2D* CreateTexture2D(
        iTVPTexture2D* orig, int flags) = 0;
};
```

`flags` 参数控制纹理创建行为，常见标志包括：

| 标志位 | 含义 | 使用场景 |
|--------|------|----------|
| `RENDER_CREATE_STATIC` | 静态纹理，不再修改 | 背景图、立绘 |
| `RENDER_CREATE_HALF` | 创建半尺寸纹理 | 大背景节省显存 |
| `RENDER_CREATE_COMPRESSED` | 使用 GPU 压缩格式 | ETC2/PVRTC |

### 渲染方法组

渲染方法（`iTVPRenderMethod`）封装了 GPU 着色器程序和混合状态：

```cpp
// 源码位置：RenderManager.h:245-260
class iTVPRenderManager {
public:
    // 按名称获取预编译的渲染方法
    // name 如 "AlphaBlend", "AddBlend", "CopyColor" 等
    virtual iTVPRenderMethod* GetRenderMethod(const char *name) = 0;

    // 从脚本代码编译渲染方法（GLSL 片段着色器）
    virtual iTVPRenderMethod* CompileRenderMethod(
        const char *name, const char *body,
        int nTextures, const char *prefix = nullptr) = 0;

    // 先查找已有，未找到则编译（带缓存）
    virtual iTVPRenderMethod* GetOrCompileRenderMethod(
        const char *name, const char *body,
        int nTextures, const char *prefix = nullptr) = 0;
};
```

### 绘制操作组

实际的绘制调用通过三种几何模式完成：

```cpp
// 源码位置：RenderManager.h:268-290
class iTVPRenderManager {
public:
    // 矩形绘制（最常用）——源矩形到目标矩形的映射
    virtual void OperateRect(
        iTVPTexture2D *tar,              // 目标纹理（渲染目标）
        iTVPTexture2D *ref,              // 引用纹理（可选，用于混合）
        const tTVPRect &rctar,           // 目标区域
        const tTVPRect &rcsrc,           // 源区域
        iTVPRenderMethod* method,        // 渲染方法
        const tRenderTexRectArray &textures  // 源纹理数组
    ) = 0;

    // 三角形绘制——自由变形
    virtual void OperateTriangles(
        iTVPTexture2D *tar,
        iTVPTexture2D *ref,
        const tTVPRect &rcclip,          // 裁剪区域
        iTVPRenderMethod* method,
        const tRenderTexQuadArray &textures  // 带四边形坐标的纹理
    ) = 0;

    // 透视变换绘制
    virtual void OperatePerspective(
        iTVPTexture2D *tar, iTVPTexture2D *ref,
        const tTVPRect &rcclip,
        iTVPRenderMethod* method,
        const tRenderTexQuadArray &textures
    ) = 0;
};
```

### 辅助功能组

```cpp
// 源码位置：RenderManager.h:260-268
class iTVPRenderManager {
public:
    // 后端身份查询
    virtual bool IsSoftware() = 0;        // 是否为软件渲染
    virtual const char* GetName() = 0;    // 后端名称，如 "opengl"

    // 渲染统计
    virtual void GetRenderStat(const char *name, float &v) = 0;

    // 模板缓冲（用于裁剪遮罩）
    virtual void BeginStencil(iTVPTexture2D *tar) = 0;
    virtual void EndStencil(iTVPTexture2D *tar, bool inv) = 0;

    // 设置渲染目标（FBO 绑定）
    virtual void SetRenderTarget(iTVPTexture2D *tar) = 0;
};
```

## 4.1.3 工厂注册机制

KrKr2 使用经典的**自注册工厂模式**来管理渲染后端。核心是 `REGISTER_RENDERMANAGER` 宏：

```cpp
// 源码位置：RenderManager.h:302-309
#define REGISTER_RENDERMANAGER(cls, name) \
    static struct _##cls##_register { \
        _##cls##_register() { \
            TVPRegisterRenderManager( \
                #name, []() -> iTVPRenderManager* { return new cls(); }); \
        } \
    } _##cls##_reg;

// 使用示例（RenderManager_ogl.cpp:4950）
REGISTER_RENDERMANAGER(TVPRenderManager_OpenGL, opengl);
```

这个宏的工作原理分三步：

**第一步：静态变量触发构造**。宏展开后生成一个全局静态结构体实例 `_TVPRenderManager_OpenGL_reg`，其构造函数在程序启动时（`main` 之前）自动执行。

**第二步：注册到工厂表**。构造函数调用 `TVPRegisterRenderManager`，将后端名称 `"opengl"` 和创建 lambda 注册到全局工厂映射表中。

**第三步：运行时查询创建**。引擎初始化时根据配置选择后端名称，从工厂表中查找对应的创建函数，调用它获得 `iTVPRenderManager*` 实例。

```
程序启动
  │
  ├─ 静态初始化阶段
  │   ├─ _TVPRenderManager_OpenGL_reg 构造
  │   │   └─ TVPRegisterRenderManager("opengl", lambda)
  │   └─ _TVPRenderManager_Software_reg 构造
  │       └─ TVPRegisterRenderManager("software", lambda)
  │
  ├─ main() 执行
  │   └─ 读取配置 → renderer = "opengl"
  │       └─ factory["opengl"]() → new TVPRenderManager_OpenGL()
  │
  └─ 全局指针 TVPGetRenderManager() 返回实例
```

访问全局渲染管理器实例的方式非常简洁：

```cpp
// 任何位置获取当前渲染后端
iTVPRenderManager* mgr = TVPGetRenderManager();

// 判断是否为软件渲染（常用于条件分支）
if (TVPIsSoftwareRenderManager()) {
    // 软件渲染路径：使用 CPU 像素操作
} else {
    // GPU 渲染路径：使用 shader
}
```

## 4.1.4 OpenGL 后端初始化流程

`TVPRenderManager_OpenGL` 的初始化涉及 GL 扩展探测、纹理格式查询和函数指针加载三个阶段。

### GL 扩展探测

引擎在首次使用 OGL 后端时调用 `TVPInitGLExtensionInfo()`：

```cpp
// 源码位置：RenderManager_ogl.cpp:48-95
struct _UsedGLExtInfo {
    // 关键扩展检测标志
    bool unpack_subimage;          // GL_EXT_unpack_subimage
    bool shader_framebuffer_fetch_ARM;  // GL_ARM_shader_framebuffer_fetch
    bool shader_framebuffer_fetch_EXT;  // GL_EXT_shader_framebuffer_fetch
    bool shader_framebuffer_fetch_NV;   // GL_NV_shader_framebuffer_fetch
    bool copy_image;               // GL_ARB_copy_image
    bool clear_texture;            // GL_ARB_clear_texture
    bool alpha_test_QCOM;          // GL_QCOM_alpha_test
};

// 扩展信息存储在全局 unordered_set 中
static std::unordered_set<std::string> sTVPGLExtensions;

void TVPInitGLExtensionInfo() {
    const char* ext = (const char*)glGetString(GL_EXTENSIONS);
    // 解析空格分隔的扩展字符串，逐个插入集合
    // 同时检查 IndividualConfigManager 中的用户覆盖设置
}
```

### 跨平台差异

GL 扩展探测在不同平台上有显著差异：

| 平台 | GL 版本 | 关键扩展 | 函数加载方式 |
|------|---------|----------|-------------|
| Windows | OpenGL 3.x+ | ARB_copy_image, ARB_clear_texture | GLEW / wglGetProcAddress |
| Linux | OpenGL 3.x+ | ARB_copy_image | glXGetProcAddress |
| macOS | OpenGL 3.2 Core | 大部分内置 | dlsym |
| Android | OpenGL ES 2.0/3.0 | EXT_shader_framebuffer_fetch | eglGetProcAddress |

Android 平台上 `shader_framebuffer_fetch` 扩展尤为关键——它允许片段着色器直接读取帧缓冲区当前值（`gl_LastFragData[0]`），从而将原本需要两张纹理的混合操作优化为只需一张纹理。这对移动端性能影响巨大。

### 纹理格式查询

```cpp
// 源码位置：RenderManager_ogl.cpp:100-120
void TVPInitTextureFormatList() {
    GLint numFormats = 0;
    glGetIntegerv(GL_NUM_COMPRESSED_TEXTURE_FORMATS, &numFormats);

    std::vector<GLint> formats(numFormats);
    glGetIntegerv(GL_COMPRESSED_TEXTURE_FORMATS, formats.data());

    for (GLint fmt : formats) {
        TVPTextureFormats.insert(fmt);  // 存入全局 set
    }
    // 后续创建压缩纹理时，先检查格式是否在 set 中
}
```

### 函数指针加载

部分 GL 函数不在基础版本中，需要动态加载：

```cpp
// 源码位置：RenderManager_ogl.cpp:130-180
void TVPInitGLExtensionFunc() {
    // Windows 平台使用 wglGetProcAddress
    #ifdef _WIN32
    if (GL_CHECK_copy_image) {
        glCopyImageSubData = (PFNGLCOPYIMAGESUBDATAPROC)
            wglGetProcAddress("glCopyImageSubData");
    }
    if (GL_CHECK_clear_texture) {
        glClearTexImage = (PFNGLCLEARTEXIMAGE)
            wglGetProcAddress("glClearTexImage");
    }
    #endif

    // Android 平台使用 eglGetProcAddress
    #ifdef ANDROID
    if (GL_CHECK_copy_image) {
        glCopyImageSubData = (PFNGLCOPYIMAGESUBDATAPROC)
            eglGetProcAddress("glCopyImageSubDataEXT");
    }
    #endif
}
```

**常见错误1：忘记检查扩展就直接调用函数指针**

```cpp
// ❌ 错误：未检查扩展可用性，函数指针可能为 nullptr
glCopyImageSubData(srcTex, GL_TEXTURE_2D, ...);

// ✅ 正确：先检查扩展标志
if (GL_CHECK_copy_image && glCopyImageSubData) {
    glCopyImageSubData(srcTex, GL_TEXTURE_2D, ...);
} else {
    // 回退路径：使用 FBO + glCopyTexSubImage2D
    FallbackCopyImage(srcTex, dstTex, ...);
}
```

**常见错误2：在错误的线程调用 GL 函数**

```cpp
// ❌ 错误：在非渲染线程创建纹理
std::thread bgThread([&]() {
    glGenTextures(1, &tex);  // 崩溃！无 GL 上下文
});

// ✅ 正确：所有 GL 调用必须在拥有 GL 上下文的线程中执行
// KrKr2 中所有渲染操作都在主线程进行
RunOnRenderThread([&]() {
    glGenTextures(1, &tex);
});
```

## 4.1.5 渲染目标管理（FBO）

OpenGL 后端使用帧缓冲对象（FBO）实现渲染到纹理（Render-to-Texture），这是图层合成的基础：

```cpp
// 源码位置：RenderManager_ogl.cpp:490-530
static GLuint _FBO = 0;                        // 全局 FBO 句柄
static iTVPTexture2D* _CurrentRenderTarget = nullptr;

void TVPSetRenderTarget(GLuint textureName) {
    if (_FBO == 0) {
        glGenFramebuffers(1, &_FBO);
    }
    glBindFramebuffer(GL_FRAMEBUFFER, _FBO);
    glFramebufferTexture2D(
        GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,
        GL_TEXTURE_2D, textureName, 0);

    // 调试模式下检查 FBO 完整性
    #ifdef _DEBUG
    GLenum status = glCheckFramebufferStatus(GL_FRAMEBUFFER);
    assert(status == GL_FRAMEBUFFER_COMPLETE);
    #endif
}
```

FBO 的核心思路是将一张纹理"挂载"为渲染目标，后续所有 `glDraw*` 调用的输出不再写入屏幕，而是写入这张纹理。KrKr2 的图层合成就是反复切换 FBO 目标，将各图层逐一绘制到上层图层的纹理上。

```
合成流程：
  1. SetRenderTarget(图层A的纹理)
  2. OperateRect(图层A, null, ..., 图层B, AlphaBlend)
     → 将图层B alpha混合到图层A上
  3. SetRenderTarget(屏幕纹理)
  4. OperateRect(屏幕, null, ..., 图层A, CopyColor)
     → 将合成结果拷贝到屏幕
```

**常见错误3：FBO 绑定后忘记恢复**

```cpp
// ❌ 错误：渲染完成后未恢复默认帧缓冲
TVPSetRenderTarget(offscreenTex);
RenderSomething();
// 后续渲染到屏幕的操作会错误地写入 offscreenTex

// ✅ 正确：完成后恢复到默认帧缓冲
TVPSetRenderTarget(offscreenTex);
RenderSomething();
glBindFramebuffer(GL_FRAMEBUFFER, 0);  // 恢复默认帧缓冲
```

---

## 动手实践

### 实践1：实现一个最小渲染后端骨架

```cpp
// MinimalRenderManager.h
#include "RenderManager.h"

class MinimalRenderManager : public iTVPRenderManager {
public:
    // --- 纹理创建 ---
    iTVPTexture2D* CreateTexture2D(
        const void *pixel, int pitch,
        unsigned int w, unsigned int h,
        TVPTextureFormat::e format, int flags) override
    {
        // 最简实现：分配 CPU 内存，拷贝像素数据
        auto* tex = new MinimalTexture2D(w, h, format);
        tex->CopyPixels(pixel, pitch);
        return tex;
    }

    iTVPTexture2D* CreateTexture2D(tTVPBitmap* bmp, int flags) override {
        return CreateTexture2D(
            bmp->GetScanLine(0), bmp->GetPitch(),
            bmp->GetWidth(), bmp->GetHeight(),
            TVPTextureFormat::RGBA, flags);
    }

    iTVPTexture2D* CreateTexture2D(tTJSBinaryStream* s, int flags) override {
        return nullptr;  // 最小实现不支持流加载
    }

    iTVPTexture2D* CreateTexture2D(iTVPTexture2D* orig, int flags) override {
        return nullptr;  // 最小实现不支持复制
    }

    // --- 渲染方法 ---
    iTVPRenderMethod* GetRenderMethod(const char *name) override {
        return &_defaultMethod;  // 返回默认方法
    }

    iTVPRenderMethod* CompileRenderMethod(
        const char *name, const char *body,
        int nTex, const char *prefix) override { return nullptr; }

    iTVPRenderMethod* GetOrCompileRenderMethod(
        const char *name, const char *body,
        int nTex, const char *prefix) override {
        return GetRenderMethod(name);
    }

    // --- 身份 ---
    bool IsSoftware() override { return true; }
    const char* GetName() override { return "minimal"; }

    // --- 绘制操作（留空） ---
    void OperateRect(...) override { /* TODO */ }
    void OperateTriangles(...) override { /* TODO */ }
    void OperatePerspective(...) override { /* TODO */ }

    // --- 辅助 ---
    void GetRenderStat(const char*, float&) override {}
    void BeginStencil(iTVPTexture2D*) override {}
    void EndStencil(iTVPTexture2D*, bool) override {}
    void SetRenderTarget(iTVPTexture2D*) override {}

private:
    MinimalRenderMethod _defaultMethod;
};

// 注册到工厂
REGISTER_RENDERMANAGER(MinimalRenderManager, minimal);
```

### 实践2：查询当前后端信息

```cpp
// 运行时查询渲染后端状态
void PrintRenderInfo() {
    iTVPRenderManager* mgr = TVPGetRenderManager();

    printf("渲染后端: %s\n", mgr->GetName());
    printf("软件渲染: %s\n", mgr->IsSoftware() ? "是" : "否");

    float fps = 0, drawCalls = 0;
    mgr->GetRenderStat("fps", fps);
    mgr->GetRenderStat("draw_calls", drawCalls);
    printf("FPS: %.1f, Draw Calls: %.0f\n", fps, drawCalls);
}
```

---

## 对照项目源码

| 概念 | 源码文件 | 行号 |
|------|----------|------|
| iTVPRenderManager 接口 | `visual/RenderManager.h` | 206-298 |
| iTVPTexture2D 接口 | `visual/RenderManager.h` | 99-147 |
| iTVPRenderMethod 接口 | `visual/RenderManager.h` | 149-173 |
| REGISTER_RENDERMANAGER 宏 | `visual/RenderManager.h` | 302-309 |
| OGL 后端注册 | `visual/ogl/RenderManager_ogl.cpp` | 4950 |
| GL 扩展探测 | `visual/ogl/RenderManager_ogl.cpp` | 48-180 |
| FBO 管理 | `visual/ogl/RenderManager_ogl.cpp` | 490-530 |
| 纹理格式查询 | `visual/ogl/RenderManager_ogl.cpp` | 100-120 |

---

## 本节小结

1. `iTVPRenderManager` 是渲染系统的顶层抽象，定义了纹理创建、渲染方法管理、绘制操作三大职责
2. `REGISTER_RENDERMANAGER` 宏利用 C++ 静态初始化机制实现自注册工厂模式，零侵入地添加新后端
3. OpenGL 后端初始化经历扩展探测→格式查询→函数指针加载三阶段，确保在不同 GPU/驱动上安全运行
4. FBO 是图层合成的基础设施，通过切换渲染目标实现多图层的逐层合成
5. `shader_framebuffer_fetch` 扩展是移动端的关键优化，将混合操作的纹理需求从 2 张降为 1 张

---

## 练习题与答案

### 练习1：接口设计分析

**问题**：`CreateTexture2D` 为什么要提供 4 个重载而不是一个通用版本？从性能和使用场景角度分析。

<details>
<summary>参考答案</summary>

四个重载分别针对不同的输入源优化：

1. **原始像素重载**：直接内存拷贝，零解析开销，适合程序生成的纹理
2. **tTVPBitmap 重载**：最常用路径，bitmap 已包含格式信息，后端可利用 bitmap 的内存布局优化上传（如 OGL 后端的 `glTexSubImage2D` 直接使用 bitmap 内存）
3. **流重载**：用于 GPU 原生压缩格式（ETC2/PVRTC），数据直接传给 `glCompressedTexImage2D`，无需 CPU 解码
4. **复制重载**：利用 GPU 端 `glCopyImageSubData` 实现零 CPU 开销的纹理复制

如果统一为一个通用接口，要么需要运行时类型判断（性能损失），要么需要先统一解码到像素数据（失去流加载和 GPU 复制的优化机会）。

</details>

### 练习2：扩展工厂模式

**问题**：如果要添加一个 Vulkan 渲染后端，需要修改哪些文件？`REGISTER_RENDERMANAGER` 宏如何确保不需要修改现有代码？

<details>
<summary>参考答案</summary>

添加 Vulkan 后端只需：

1. 创建新文件 `RenderManager_vk.cpp`，实现 `TVPRenderManager_Vulkan` 类继承 `iTVPRenderManager`
2. 在文件末尾添加 `REGISTER_RENDERMANAGER(TVPRenderManager_Vulkan, vulkan);`
3. 在构建系统中将新文件加入编译

**不需要修改任何现有文件**。`REGISTER_RENDERMANAGER` 利用 C++ 的静态初始化机制：编译器为每个翻译单元中的全局静态对象生成初始化代码，链接器将其汇总到全局初始化列表中。程序启动时，C Runtime 遍历此列表执行所有静态构造函数，从而自动完成注册。这就是"自注册"模式的核心——注册逻辑与实现代码放在同一个编译单元中，耦合度为零。

</details>

### 练习3：FBO 调试

**问题**：在调试渲染问题时，如何验证 FBO 绑定是否正确？写出检查代码。

<details>
<summary>参考答案</summary>

```cpp
void DebugCheckFBO() {
    // 1. 检查当前绑定的 FBO
    GLint currentFBO = 0;
    glGetIntegerv(GL_FRAMEBUFFER_BINDING, &currentFBO);
    printf("当前 FBO: %d\n", currentFBO);

    // 2. 检查 FBO 完整性状态
    GLenum status = glCheckFramebufferStatus(GL_FRAMEBUFFER);
    switch (status) {
    case GL_FRAMEBUFFER_COMPLETE:
        printf("FBO 状态: 完整\n"); break;
    case GL_FRAMEBUFFER_INCOMPLETE_ATTACHMENT:
        printf("FBO 错误: 附件不完整\n"); break;
    case GL_FRAMEBUFFER_INCOMPLETE_MISSING_ATTACHMENT:
        printf("FBO 错误: 缺少附件\n"); break;
    case GL_FRAMEBUFFER_UNSUPPORTED:
        printf("FBO 错误: 格式组合不支持\n"); break;
    default:
        printf("FBO 错误: 未知状态 0x%X\n", status);
    }

    // 3. 检查附加的纹理
    GLint attachedTex = 0;
    glGetFramebufferAttachmentParameteriv(
        GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,
        GL_FRAMEBUFFER_ATTACHMENT_OBJECT_NAME, &attachedTex);
    printf("附加纹理: %d\n", attachedTex);
}
```

</details>

---

## 下一步

下一节 [4.2 纹理管理与压缩](02-纹理管理与压缩.md) 将深入 `tTVPOGLTexture2D` 和 `tTVPOGLTexture2D_split` 的实现，讲解 GPU 纹理的上传、缓存、压缩（ETC2/PVRTC）和超大纹理的分块处理策略。
