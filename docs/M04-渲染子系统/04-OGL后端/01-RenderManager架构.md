# RenderManager 架构

> **所属模块：** M04-渲染子系统
> **前置知识：** [P04-OpenGL图形编程](../../P04-OpenGL图形编程/README.md)、[M04/01-visual模块总览](../01-visual模块总览/01-目录结构与CMake构建.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 KrKr2 渲染管理器的三层抽象接口设计（`iTVPTexture2D`、`iTVPRenderMethod`、`iTVPRenderManager`）
2. 掌握 OpenGL 纹理类层次结构及其使用场景（静态/可变/分块纹理）
3. 理解 GLSL 渲染方法的封装机制与 Shader 编译流程
4. 学会通过工厂模式注册自定义渲染后端

---

## 一、渲染管理器概述

KrKr2 的渲染子系统采用**后端无关设计**——核心逻辑通过抽象接口操作纹理和渲染方法，而具体实现由 OpenGL 或软件渲染后端提供。这种设计使得：

- 同一份游戏脚本可在 GPU 和 CPU 渲染之间切换
- 移动端可使用不同的纹理压缩策略
- 未来可扩展 Vulkan/Metal 后端

### 1.1 核心文件结构

```
cpp/core/visual/
├── RenderManager.h              # 抽象接口定义
└── ogl/
    ├── ogl_common.h             # 跨平台 GL 头文件包含
    ├── RenderManager_ogl.cpp    # OpenGL 后端实现（~5000行）
    ├── etcpak.h/.cpp            # ETC 纹理压缩
    ├── pvrtc.h/.cpp             # PVRTC 纹理压缩
    ├── astcrt.h/.cpp            # ASTC 纹理压缩
    └── imagepacker.h/.cpp       # 纹理图集打包
```

### 1.2 三层抽象架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    TJS2 脚本层 / Layer 系统                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│           iTVPRenderManager（渲染管理器抽象接口）                 │
│  ├── CreateTexture2D()     创建纹理                              │
│  ├── GetRenderMethod()     获取渲染方法                          │
│  ├── OperateRect()         矩形渲染操作                          │
│  ├── OperateTriangles()    三角形渲染操作                        │
│  └── OperatePerspective()  透视变换渲染                          │
└─────────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┴───────────────────┐
          ▼                                       ▼
┌─────────────────────┐               ┌─────────────────────────┐
│ TVPRenderManager_OGL│               │ TVPRenderManager_Software│
│   (OpenGL 后端)      │               │   (软件渲染后端)          │
└─────────────────────┘               └─────────────────────────┘
```

---

## 二、抽象接口详解

### 2.1 iTVPTexture2D：纹理抽象接口

`iTVPTexture2D` 是所有纹理对象的基类，定义了纹理的通用操作：

```cpp
// 文件：cpp/core/visual/RenderManager.h 第 99-147 行

class iTVPTexture2D {
protected:
    int RefCount;                    // 引用计数
    tjs_int Width;                   // 逻辑宽度
    tjs_int Height;                  // 逻辑高度
    
    iTVPTexture2D(tjs_int w, tjs_int h) 
        : Width(w), Height(h), RefCount(1) {}

public:
    virtual ~iTVPTexture2D() = default;
    
    // 引用计数管理
    void AddRef() { ++RefCount; }
    virtual void Release();
    bool IsIndependent() const { return RefCount == 1; }
    
    // 尺寸查询
    tjs_uint GetWidth() const { return Width; }
    tjs_uint GetHeight() const { return Height; }
    virtual tjs_uint GetInternalWidth() const { return Width; }
    virtual tjs_uint GetInternalHeight() const { return Height; }
    
    // 像素格式
    virtual TVPTextureFormat::e GetFormat() const = 0;
    
    // 像素数据访问（软件渲染用）
    virtual const void *GetScanLineForRead(tjs_uint l) { return nullptr; }
    virtual void *GetScanLineForWrite(tjs_uint l) { 
        return (void *)GetScanLineForRead(l); 
    }
    virtual tjs_int GetPitch() const { return 0x100000; }
    
    // 核心操作
    virtual void Update(const void *pixel, TVPTextureFormat::e format,
                        int pitch, const tTVPRect &rc) = 0;
    virtual uint32_t GetPoint(int x, int y) = 0;
    virtual void SetPoint(int x, int y, uint32_t clr) = 0;
    
    // 纹理属性
    virtual bool IsStatic() = 0;     // 是否只读
    virtual bool IsOpaque() = 0;     // 是否不透明
    
    // Cocos2d-x 适配器（用于 UI 渲染）
    virtual cocos2d::Texture2D *GetAdapterTexture(
        cocos2d::Texture2D *origTex) = 0;
    
    // 缩放因子（用于移动端纹理压缩）
    virtual bool GetScale(float &x, float &y) {
        x = 1.f; y = 1.f;
        return true;
    }
};
```

**关键设计点：**

| 特性 | 说明 |
|------|------|
| 引用计数 | 纹理可被多个 Layer 共享，自动释放 |
| 逻辑尺寸 vs 内部尺寸 | 移动端压缩后内部尺寸可能小于逻辑尺寸 |
| GetScale | 返回缩放因子，用于纹理坐标转换 |
| IsStatic | 静态纹理不可修改，可使用压缩格式 |

### 2.2 iTVPRenderMethod：渲染方法抽象接口

`iTVPRenderMethod` 封装了一种渲染操作（如 Copy、AlphaBlend、ColorDodge 等）：

```cpp
// 文件：cpp/core/visual/RenderManager.h 第 149-173 行

class iTVPRenderMethod {
protected:
    virtual ~iTVPRenderMethod() = default;
    std::string Name;               // 方法名称

public:
    // 参数枚举（返回参数的 ID，用于后续 SetParameter）
    virtual int EnumParameterID(const char *name) { return -1; }
    
    // 设置各类参数
    virtual void SetParameterUInt(int id, unsigned int Value) {}
    virtual void SetParameterInt(int id, int Value) {}
    virtual void SetParameterPtr(int id, const void *Value) {}
    virtual void SetParameterFloat(int id, float Value) {}
    virtual void SetParameterColor4B(int id, unsigned int clr) {}
    virtual void SetParameterOpa(int id, int Value) {}
    virtual void SetParameterFloatArray(int id, float *Value, int nElem) {}
    
    // 混合函数配置
    virtual iTVPRenderMethod *SetBlendFuncSeparate(
        int func, int srcRGB, int dstRGB, int srcAlpha, int dstAlpha) {
        return this;
    }
    
    // 是否需要混合目标（某些方法如 Copy 不需要读取目标像素）
    virtual bool IsBlendTarget() { return true; }
    
    void SetName(const std::string &name) { Name = name; }
    const std::string &GetName() { return Name; }
};
```

**使用示例：**

```cpp
// 获取 Alpha 混合方法
iTVPRenderMethod *method = renderManager->GetRenderMethod("AlphaBlend");

// 设置不透明度参数
int opaId = method->EnumParameterID("opacity");
method->SetParameterOpa(opaId, 128);  // 50% 不透明度

// 执行渲染
renderManager->OperateRect(method, target, refTarget, rect, textures);
```

### 2.3 iTVPRenderManager：渲染管理器抽象接口

`iTVPRenderManager` 是整个渲染子系统的入口：

```cpp
// 文件：cpp/core/visual/RenderManager.h 第 206-298 行

class iTVPRenderManager {
protected:
    virtual ~iTVPRenderManager() = default;
    
    // 注册渲染方法（内部使用）
    void RegisterRenderMethod(const char *name, iTVPRenderMethod *method);
    
    // 从脚本编译渲染方法
    virtual iTVPRenderMethod *GetRenderMethodFromScript(
        const char *script, int nTex, unsigned int flags) { return nullptr; }
    
    // 所有已注册的渲染方法
    std::unordered_map<uint32_t, iTVPRenderMethod *> AllMethods;

public:
    void Initialize();

    // ============ 纹理创建 ============
    // 创建标志：
    // - RENDER_CREATE_TEXTURE_FLAG_STATIC: 只读纹理（可压缩）
    // - RENDER_CREATE_TEXTURE_FLAG_NO_COMPRESS: 禁止压缩
    
    virtual iTVPTexture2D *CreateTexture2D(
        const void *pixel, int pitch, unsigned int w, unsigned int h,
        TVPTextureFormat::e format, 
        int flags = RENDER_CREATE_TEXTURE_FLAG_ANY) = 0;
    
    // 从 Bitmap 创建（用于省略图等大图）
    virtual iTVPTexture2D *CreateTexture2D(tTVPBitmap *bmp) = 0;
    
    // 从流创建（用于压缩格式）
    virtual iTVPTexture2D *CreateTexture2D(TJS::tTJSBinaryStream *s) = 0;
    
    // 复制并调整大小
    virtual iTVPTexture2D *CreateTexture2D(
        unsigned int neww, unsigned int newh, iTVPTexture2D *tex) = 0;

    // ============ 渲染方法 ============
    virtual iTVPRenderMethod *GetRenderMethod(
        const char *name, uint32_t *hint = nullptr);
    
    // 编译 GLSL 脚本为渲染方法
    iTVPRenderMethod *CompileRenderMethod(
        const char *name, const char *glsl_script, 
        int nTex, unsigned int flags = 0);
    
    // ============ 渲染操作 ============
    // 矩形渲染：dst × Tex1 × ... × TexN -> dst
    virtual void OperateRect(
        iTVPRenderMethod *method,
        iTVPTexture2D *tar,           // 目标纹理
        iTVPTexture2D *reftar,        // 参考目标（用于读取目标像素）
        const tTVPRect &rctar,        // 目标矩形
        const tRenderTexRectArray &textures) = 0;  // 源纹理数组
    
    // 三角形渲染：src × dst -> tar
    virtual void OperateTriangles(
        iTVPRenderMethod *method, int nTriangles,
        iTVPTexture2D *target, iTVPTexture2D *reftar,
        const tTVPRect &rcclip,
        const tTVPPointD *pttar,
        const tRenderTexQuadArray &textures) = 0;
    
    // 透视变换渲染
    virtual void OperatePerspective(
        iTVPRenderMethod *method, int nQuads,
        iTVPTexture2D *target, iTVPTexture2D *reftar,
        const tTVPRect &rcclip,
        const tTVPPointD *pttar,  // quad{lt,rt,lb,rb}
        const tRenderTexQuadArray &textures) = 0;

    // ============ 查询接口 ============
    virtual bool IsSoftware() { return false; }
    virtual const char *GetName() = 0;
    virtual bool GetRenderStat(unsigned int &drawCount, uint64_t &vmemsize) = 0;
};
```

---

## 三、OpenGL 纹理类层次

OpenGL 后端实现了四种纹理类型，继承自 `iTVPTexture2D`：

```
iTVPTexture2D（抽象基类）
    │
    └── tTVPOGLTexture2D（OGL 纹理基类）
            │
            ├── tTVPOGLTexture2D_static   （静态纹理：不可修改，可压缩）
            │
            ├── tTVPOGLTexture2D_mutable  （可变纹理：可修改，用作渲染目标）
            │
            └── tTVPOGLTexture2D_split    （分块纹理：超大图按需加载）
```

### 3.1 tTVPOGLTexture2D：OGL 纹理基类

```cpp
// 文件：cpp/core/visual/ogl/RenderManager_ogl.cpp 第 862-1145 行

class tTVPOGLTexture2D : public iTVPTexture2D {
protected:
    GLuint texture = 0;              // OpenGL 纹理句柄
    bool IsCompressed = false;       // 是否使用压缩格式
    TVPTextureFormat::e Format;      // 像素格式
    unsigned int internalW, internalH;  // 实际纹理尺寸（可能是 POT）
    unsigned char *PixelData = nullptr; // CPU 侧像素缓存
    float _scaleW = 1, _scaleH = 1;     // 缩放因子
    
    // 初始化纹理
    void InternalInit(const void *pixel, unsigned int intw, 
                      unsigned int inth, unsigned int pitch) {
        GLenum pixfmt = GL_RGBA, internalfmt = GL_RGBA;
        
        // 根据格式设置参数
        switch(Format) {
            case TVPTextureFormat::Gray:
                pixfmt = internalfmt = GL_LUMINANCE;
                break;
            case TVPTextureFormat::RGB:
                pixfmt = internalfmt = GL_RGB;
                break;
            case TVPTextureFormat::RGBA:
                pixfmt = internalfmt = GL_RGBA;
                break;
        }
        
        // 设置像素对齐
        switch(pitch & 7) {
            case 0: glPixelStorei(GL_UNPACK_ALIGNMENT, 8); break;
            case 4: glPixelStorei(GL_UNPACK_ALIGNMENT, 4); break;
            case 2: case 6: glPixelStorei(GL_UNPACK_ALIGNMENT, 2); break;
            default: glPixelStorei(GL_UNPACK_ALIGNMENT, 1); break;
        }
        
        // 上传纹理数据
        _glBindTexture2D(texture);
        glTexImage2D(GL_TEXTURE_2D, 0, internalfmt, intw, inth, 
                     0, pixfmt, GL_UNSIGNED_BYTE, pixel);
        
        internalW = intw;
        internalH = inth;
        _totalVMemSize += internalW * internalH * getPixelSize();
    }
    
    // 部分更新纹理
    void InternalUpdate(const void *pixel, int pitch, 
                        int x, int y, int w, int h) {
        // 如果有缩放，先恢复原始尺寸
        if(_scaleW != 1.f || _scaleH != 1.f) {
            RestoreNormalSize();
        }
        
        _glBindTexture2D(texture);
        
        // 处理非连续内存布局
        if(GL_CHECK_unpack_subimage) {
            // 使用 GL_UNPACK_ROW_LENGTH 扩展
            glPixelStorei(GL_UNPACK_ROW_LENGTH, pitch / pixsize);
            glTexSubImage2D(GL_TEXTURE_2D, 0, x, y, w, h, 
                            pixfmt, GL_UNSIGNED_BYTE, pixel);
            glPixelStorei(GL_UNPACK_ROW_LENGTH, 0);
        } else {
            // 手动重排内存
            unsigned char *arr = new unsigned char[w * h * pixsize];
            // ... 复制数据 ...
            glTexSubImage2D(GL_TEXTURE_2D, 0, x, y, w, h,
                            pixfmt, GL_UNSIGNED_BYTE, arr);
            delete[] arr;
        }
    }

public:
    // 顶点坐标计算（用于渲染）
    virtual void ApplyVertex(GLVertexInfo &vtx, const tTVPRect &rc) {
        vtx.tex = this;
        
        // 计算纹理坐标 [0,1]
        float tw = _scaleW / internalW;
        float th = _scaleH / internalH;
        
        GLfloat sminu = rc.left * tw;
        GLfloat smaxu = rc.right * tw;
        GLfloat sminv = rc.top * th;
        GLfloat smaxv = rc.bottom * th;
        
        // 生成两个三角形的顶点
        vtx.vtx.resize(12);
        GLfloat *pt = &vtx.vtx.front();
        pt[0] = sminu; pt[1] = sminv;   // 左上
        pt[2] = smaxu; pt[3] = sminv;   // 右上
        pt[4] = sminu; pt[5] = smaxv;   // 左下
        pt[6] = smaxu; pt[7] = sminv;   // 右上
        pt[8] = sminu; pt[9] = smaxv;   // 左下
        pt[10] = smaxu; pt[11] = smaxv; // 右下
    }
    
    // 绑定到纹理单元
    void Bind(unsigned int i) { 
        cocos2d::GL::bindTexture2DN(i, texture); 
    }
};
```

### 3.2 静态纹理 vs 可变纹理

| 特性 | tTVPOGLTexture2D_static | tTVPOGLTexture2D_mutable |
|------|-------------------------|--------------------------|
| 可修改 | 否 | 是 |
| 可作为渲染目标 | 否 | 是 |
| 支持压缩 | 是（ETC2/PVRTC/ASTC） | 否 |
| 典型用途 | 背景图、立绘 | Layer 缓冲区、特效 |
| 内存优化 | 可降采样、压缩 | 必须原始尺寸 |

```cpp
// 静态纹理：用于图片资源
// 文件：cpp/core/visual/ogl/RenderManager_ogl.cpp 第 1553-1679 行
class tTVPOGLTexture2D_static : public tTVPOGLTexture2D {
public:
    bool IsStatic() override { return true; }
    
    // 禁止写入
    void *GetScanLineForWrite(tjs_uint l) override {
        assert(false);
        return nullptr;
    }
    
    void SetSize(unsigned int w, unsigned int h) override { 
        assert(false); 
    }
};

// 可变纹理：用于渲染目标
// 文件：cpp/core/visual/ogl/RenderManager_ogl.cpp 第 1681-1819 行
class tTVPOGLTexture2D_mutable : public tTVPOGLTexture2D {
    bool IsTextureDirty;  // 是否有未同步的修改
    
public:
    bool IsStatic() override { return false; }
    
    // 作为渲染目标
    void AsTarget() override {
        SyncPixel();
        TVPSetRenderTarget(texture);
    }
    
    // 同步 CPU 缓存到 GPU
    void SyncPixel() override {
        if(PixelData && IsTextureDirty) {
            InternalUpdate(PixelData, internalW * 4, 
                           0, 0, internalW, internalH);
            IsTextureDirty = false;
            delete[] PixelData;
            PixelData = nullptr;
        }
    }
};
```

### 3.3 分块纹理：超大图支持

当图片尺寸超过 GPU 最大纹理尺寸（通常 4096×4096）时，使用分块纹理：

```cpp
// 文件：cpp/core/visual/ogl/RenderManager_ogl.cpp 第 1160-1551 行
class tTVPOGLTexture2D_split : public tTVPOGLTexture2D {
    // 分块缓存
    std::map<uint32_t, GLTextureInfo> CachedTexture;
    tTVPBitmap *Bitmap;  // 持有原始位图引用
    
public:
    // 按需创建分块纹理
    void ApplyVertex(GLVertexInfo &vtx, const tTVPRect &rc) override {
        // 如果请求区域超过最大纹理尺寸，降采样为单个纹理
        if(rc.get_width() > GetMaxTextureWidth() || 
           rc.get_height() > GetMaxTextureHeight()) {
            AsSingleTexture();
            return ApplyVertex(vtx, rc);
        }
        
        // 查找或创建分块
        GLTextureInfoIndexer indexer;
        indexer.Point.X = rc.left;
        indexer.Point.Y = rc.top;
        
        auto it = CachedTexture.find(indexer.Index);
        if(it == CachedTexture.end()) {
            // 创建新分块
            texture = FetchGLTexture();
            GLTextureInfo &texinfo = CachedTexture[indexer.Index];
            texinfo.Name = texture;
            UpdateTextureData(texinfo, rc);
        }
        
        // 计算分块内的纹理坐标
        // ...
    }
};
```

---

## 四、GLSL 渲染方法系统

### 4.1 渲染方法基类

```cpp
// 文件：cpp/core/visual/ogl/RenderManager_ogl.cpp 第 1821-1869 行

class tTVPOGLRenderMethod : public iTVPRenderMethod {
protected:
    GLint pos_attr_location;              // 顶点位置属性位置
    std::vector<GLint> tex_coord_attr_location;  // 纹理坐标属性位置
    GLenum program;                       // Shader 程序句柄
    
    // 混合状态
    int BlendFunc = 0;
    int BlendSrcRGB, BlendDstRGB;
    int BlendSrcA, BlendDstA;
    
    bool tar_as_src;  // 是否将目标纹理作为输入
    
public:
    virtual void Rebuild() = 0;  // 重建 Shader（GL 上下文丢失时）
    virtual void Apply() = 0;    // 应用渲染状态
    
    // 应用纹理和顶点
    virtual void ApplyTexture(unsigned int i, const GLVertexInfo &info) {
        info.tex->Bind(i);
        glVertexAttribPointer(GetTexCoordAttr(i), 2, GL_FLOAT, 
                              GL_FALSE, 0, &info.vtx.front());
    }
    
    // 自定义渲染过程（用于 FillARGB 等优化）
    bool (*CustomProc)(...) = nullptr;
};
```

### 4.2 脚本式渲染方法

大部分渲染方法通过 GLSL 脚本定义：

```cpp
// 文件：cpp/core/visual/ogl/RenderManager_ogl.cpp 第 1919-2057 行

class tTVPOGLRenderMethod_Script : public tTVPOGLRenderMethod {
    std::string m_strScript;  // Fragment Shader 代码
    int m_nTex;               // 输入纹理数量
    
public:
    bool init(const std::string &body, int nTex, 
              const char *prefix = nullptr) {
        m_nTex = nTex;
        
        // 构建 Fragment Shader
        std::ostringstream str;
        if(prefix) str << prefix;
        str << "#ifdef GL_ES\n"
               "precision mediump float;\n"
               "#endif\n";
        
        // 声明纹理坐标 varying
        for(int i = 0; i < nTex; ++i) {
            str << "varying vec2 v_texCoord" << i << ";\n";
        }
        
        // 声明纹理 uniform
        for(int i = 0; i < nTex; ++i) {
            str << "uniform sampler2D tex" << i << ";\n";
        }
        
        str << body;
        m_strScript = str.str();
        Rebuild();
        return true;
    }
    
    // 动态生成 Vertex Shader
    static GLenum GetVertShader(unsigned int nTex) {
        std::ostringstream shader;
        shader << "#ifdef GL_ES\n"
                  "precision mediump float;\n"
                  "#endif\n"
                  "attribute vec2 a_position;\n";
        
        for(unsigned int i = 0; i < nTex; ++i) {
            shader << "attribute vec2 a_texCoord" << i << ";\n";
            shader << "varying vec2 v_texCoord" << i << ";\n";
        }
        
        shader << "void main() {\n"
                  "    gl_Position = vec4(a_position, 0.0, 1.0);\n";
        
        for(unsigned int i = 0; i < nTex; ++i) {
            shader << "    v_texCoord" << i 
                   << " = a_texCoord" << i << ";\n";
        }
        
        shader << "}";
        return CompileShader(GL_VERTEX_SHADER, shader.str());
    }
    
    void Rebuild() override {
        program = CombineProgram(
            GetVertShader(m_nTex),
            CompileShader(GL_FRAGMENT_SHADER, m_strScript));
        
        // 获取属性位置
        cocos2d::GL::useProgram(program);
        for(int i = 0; i < m_nTex; ++i) {
            char name[32];
            sprintf(name, "tex%d", i);
            int loc = glGetUniformLocation(program, name);
            glUniform1i(loc, i);
            
            sprintf(name, "a_texCoord%d", i);
            loc = glGetAttribLocation(program, name);
            tex_coord_attr_location.push_back(loc);
        }
        pos_attr_location = glGetAttribLocation(program, "a_position");
    }
    
    void Apply() override {
        cocos2d::GL::useProgram(program);
        if(BlendFunc) {
            glEnable(GL_BLEND);
            glBlendEquation(BlendFunc);
            glBlendFuncSeparate(BlendSrcRGB, BlendDstRGB, 
                                BlendSrcA, BlendDstA);
        } else {
            glDisable(GL_BLEND);
        }
    }
};
```

### 4.3 内置渲染方法示例

```cpp
// 简单复制（无混合）
const std::string Copy_Script = R"(
void main() {
    gl_FragColor = texture2D(tex0, v_texCoord0);
}
)";

// Alpha 混合
const std::string AlphaBlend_Script = R"(
uniform float opacity;
void main() {
    vec4 s = texture2D(tex0, v_texCoord0);
    vec4 d = texture2D(tex1, v_texCoord1);
    float a = s.a * opacity;
    gl_FragColor = vec4(mix(d.rgb, s.rgb, a), d.a + a * (1.0 - d.a));
}
)";

// 填充颜色
class tTVPOGLRenderMethod_FillARGB : public tTVPOGLRenderMethod_Script {
    uint32_t m_color;
    
public:
    // 使用 glClearTexImage 优化（如果支持）
    static bool DoFill(tTVPOGLRenderMethod *method, 
                       tTVPOGLTexture2D *tar, ...) {
        uint32_t c = static_cast<tTVPOGLRenderMethod_FillARGB*>(method)->m_color;
        uint8_t clr[4] = { 
            (uint8_t)c, (uint8_t)(c >> 8), 
            (uint8_t)(c >> 16), (uint8_t)(c >> 24) 
        };
        GL::glClearTexImage(tar->texture, 0, GL_RGBA, 
                            GL_UNSIGNED_BYTE, clr);
        return true;
    }
};
```

---

## 五、工厂模式与后端注册

### 5.1 REGISTER_RENDERMANAGER 宏

```cpp
// 文件：cpp/core/visual/RenderManager.h 第 302-309 行

#define REGISTER_RENDERMANAGER(MGR, NAME)                              \
    static iTVPRenderManager *__##MGR##Factory() { return new MGR(); } \
    static class __##MGR##AutoRegister {                               \
    public:                                                            \
        __##MGR##AutoRegister() {                                      \
            TVPRegisterRenderManager(#NAME, __##MGR##Factory);         \
        }                                                              \
    } __##MGR##AutoRegister_instance;

// 使用示例（在 RenderManager_ogl.cpp 末尾）
REGISTER_RENDERMANAGER(TVPRenderManager_OpenGL, opengl)
```

### 5.2 获取渲染管理器

```cpp
// 获取当前活动的渲染管理器
iTVPRenderManager *TVPGetRenderManager();

// 获取指定名称的渲染管理器
iTVPRenderManager *TVPGetRenderManager(const TJS::tTJSString &name);

// 检查是否软件渲染
bool TVPIsSoftwareRenderManager();
```

---

## 动手实践

### 练习：创建一个简单的渲染方法

```cpp
// 1. 定义 Fragment Shader
const std::string Grayscale_Script = R"(
void main() {
    vec4 c = texture2D(tex0, v_texCoord0);
    float gray = dot(c.rgb, vec3(0.299, 0.587, 0.114));
    gl_FragColor = vec4(vec3(gray), c.a);
}
)";

// 2. 编译并注册
iTVPRenderManager *mgr = TVPGetRenderManager();
iTVPRenderMethod *grayscale = mgr->CompileRenderMethod(
    "Grayscale",    // 方法名
    Grayscale_Script.c_str(),
    1,              // 输入纹理数量
    0               // 标志
);

// 3. 使用
std::pair<iTVPTexture2D*, tTVPRect> textures[] = {
    { sourceTexture, srcRect }
};
mgr->OperateRect(
    grayscale, 
    targetTexture, 
    nullptr,        // 不需要读取目标
    dstRect,
    tRenderTexRectArray(textures)
);
```

---

## 对照项目源码

| 文件 | 行号 | 内容 |
|------|------|------|
| `cpp/core/visual/RenderManager.h` | 99-147 | iTVPTexture2D 定义 |
| `cpp/core/visual/RenderManager.h` | 149-173 | iTVPRenderMethod 定义 |
| `cpp/core/visual/RenderManager.h` | 206-298 | iTVPRenderManager 定义 |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 862-1145 | tTVPOGLTexture2D 基类 |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 1553-1679 | 静态纹理实现 |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 1681-1819 | 可变纹理实现 |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 1919-2057 | 脚本式渲染方法 |

---

## 本节小结

- KrKr2 渲染子系统采用三层抽象：纹理(`iTVPTexture2D`)、渲染方法(`iTVPRenderMethod`)、管理器(`iTVPRenderManager`)
- OpenGL 后端实现了静态/可变/分块三种纹理类型，适应不同使用场景
- 渲染方法通过 GLSL 脚本动态编译，支持自定义混合效果
- 工厂模式支持多后端切换（OpenGL/Software）

---

## 练习题与答案

### 题目 1：解释 `IsStatic()` 和 `IsOpaque()` 的作用及它们如何影响渲染性能

<details>
<summary>查看答案</summary>

**IsStatic()**：
- 返回 `true` 表示纹理内容不会改变
- 静态纹理可以使用压缩格式（ETC2/PVRTC/ASTC）减少显存占用
- 静态纹理不需要分配 CPU 侧写入缓冲区
- 典型用途：背景图、立绘、UI 元素

**IsOpaque()**：
- 返回 `true` 表示纹理完全不透明
- 不透明纹理可以使用 RGB 格式而非 RGBA，节省 25% 显存
- 不透明纹理在混合时可以跳过 Alpha 通道计算
- 可以使用更高效的压缩格式（如 ETC2 RGB 而非 ETC2 RGBA）

**性能影响**：
- 静态 + 不透明纹理可减少约 40-60% 的显存占用
- 减少纹理上传和同步操作
- 允许 GPU 使用更高效的纹理采样路径

</details>

### 题目 2：为什么 `tTVPOGLTexture2D_split` 需要按需创建分块？

<details>
<summary>查看答案</summary>

**原因分析**：

1. **GPU 纹理尺寸限制**：大多数 GPU 最大支持 4096×4096 或 8192×8192 纹理，而视觉小说的背景图可能达到 8000×6000 甚至更大

2. **显存效率**：如果将整张大图上传为多个最大尺寸纹理，会占用大量显存（如 8000×6000 需要约 183MB）

3. **按需加载策略**：
   - 只创建当前可见区域的分块纹理
   - 使用 `std::map<uint32_t, GLTextureInfo>` 缓存已创建的分块
   - 当渲染请求到达时，检查分块是否存在，不存在则创建

4. **典型场景**：
   ```cpp
   // 滚动背景时，只需要加载可见区域
   // 假设屏幕 1920×1080，背景 8000×6000
   // 传统方式：上传全部 183MB
   // 分块方式：只上传 ~8MB（当前可见 + 缓存）
   ```

5. **降级策略**：当使用 `ApplyVertex` 且请求区域需要多个分块时，会触发 `AsSingleTexture()` 将整张图降采样为 GPU 支持的最大尺寸

</details>

### 题目 3：实现一个简单的色相偏移渲染方法

<details>
<summary>查看答案</summary>

```cpp
// HueShift.glsl - Fragment Shader
const char* HueShift_Script = R"(
uniform float u_hue;  // 色相偏移量（0-1 对应 0-360 度）

// RGB 转 HSV
vec3 rgb2hsv(vec3 c) {
    vec4 K = vec4(0.0, -1.0/3.0, 2.0/3.0, -1.0);
    vec4 p = mix(vec4(c.bg, K.wz), vec4(c.gb, K.xy), step(c.b, c.g));
    vec4 q = mix(vec4(p.xyw, c.r), vec4(c.r, p.yzx), step(p.x, c.r));
    float d = q.x - min(q.w, q.y);
    float e = 1.0e-10;
    return vec3(abs(q.z + (q.w - q.y) / (6.0*d + e)), d / (q.x + e), q.x);
}

// HSV 转 RGB
vec3 hsv2rgb(vec3 c) {
    vec4 K = vec4(1.0, 2.0/3.0, 1.0/3.0, 3.0);
    vec3 p = abs(fract(c.xxx + K.xyz) * 6.0 - K.www);
    return c.z * mix(K.xxx, clamp(p - K.xxx, 0.0, 1.0), c.y);
}

void main() {
    vec4 c = texture2D(tex0, v_texCoord0);
    vec3 hsv = rgb2hsv(c.rgb);
    hsv.x = fract(hsv.x + u_hue);  // 偏移色相
    gl_FragColor = vec4(hsv2rgb(hsv), c.a);
}
)";

// 使用示例
void applyHueShift(iTVPRenderManager* mgr, 
                   iTVPTexture2D* target, iTVPTexture2D* source,
                   const tTVPRect& rect, float hueShift) {
    // 编译（只需一次）
    static iTVPRenderMethod* method = nullptr;
    static int hueParamId = -1;
    
    if(!method) {
        method = mgr->CompileRenderMethod("HueShift", HueShift_Script, 1, 0);
        hueParamId = method->EnumParameterID("u_hue");
    }
    
    // 设置参数
    method->SetParameterFloat(hueParamId, hueShift);
    
    // 执行渲染
    std::pair<iTVPTexture2D*, tTVPRect> textures[] = {
        { source, rect }
    };
    mgr->OperateRect(method, target, nullptr, rect, 
                     tRenderTexRectArray(textures));
}
```

</details>

---

## 下一步

[02-纹理上传与批量渲染](./02-纹理上传与批量渲染.md) — 深入了解纹理数据上传流程、FBO 管理和批量渲染优化
