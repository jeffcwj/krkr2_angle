# FBO 与批量渲染

> **所属模块：** M04-渲染子系统
> **前置知识：** [01-纹理数据上传](./01-纹理数据上传.md)、[P04-OpenGL图形编程](../../../P04-OpenGL图形编程/README.md)
> **预计阅读时间：** 20 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 FBO（帧缓冲对象）的管理机制及渲染目标切换
2. 掌握 `OperateRect` 矩形区域渲染的实现原理
3. 理解 `OperateTriangles` 三角形网格渲染的坐标转换
4. 掌握 `ApplyVertex` 顶点数据生成的算法
5. 能够分析和优化批量渲染的性能瓶颈

---

## FBO 管理机制

### FBO 基本概念

帧缓冲对象（Framebuffer Object, FBO）允许渲染到纹理而非屏幕。KrKr2 使用 FBO 实现：

- **离屏渲染**：将图层合成到临时纹理
- **后处理效果**：对渲染结果应用滤镜
- **过渡动画**：在两个场景之间切换

```
┌─────────────────────────────────────────────────────────────────┐
│                    FBO 渲染流程                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   默认帧缓冲（屏幕）          FBO（离屏纹理）                      │
│   ┌─────────────────┐        ┌─────────────────┐                │
│   │                 │        │                 │                │
│   │   最终显示画面   │◀───────│   图层合成结果   │                │
│   │                 │        │                 │                │
│   └─────────────────┘        └─────────────────┘                │
│           ▲                          ▲                          │
│           │                          │                          │
│   glBindFramebuffer          glBindFramebuffer                  │
│   (GL_FRAMEBUFFER, 0)        (GL_FRAMEBUFFER, fbo)              │
│                                      │                          │
│                              glFramebufferTexture2D             │
│                              (附加纹理作为渲染目标)               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### KrKr2 的 FBO 架构

KrKr2 使用单一共享 FBO 实现所有离屏渲染，避免频繁创建/销毁：

```cpp
// 来源：cpp/core/visual/ogl/RenderManager_ogl.cpp 第 469-491 行
static bool _CurrentFBOValid = false;      // FBO 是否已绑定
static GLuint _FBO;                         // 共享帧缓冲对象
static GLuint _stencil_FBO;                 // 模板缓冲 FBO（特殊用途）
static GLuint _CurrentRenderTarget = 0;    // 当前渲染目标纹理
static GLint _screenFrameBuffer = 0;       // 屏幕帧缓冲（保存恢复用）
```

### TVPSetRenderTarget 实现

这是渲染目标切换的核心函数：

```cpp
// 来源：cpp/core/visual/ogl/RenderManager_ogl.cpp 第 493-509 行
void TVPSetRenderTarget(GLuint t) {
    if(t) {
        // 切换到 FBO 渲染
        if(!_CurrentFBOValid) {
            _CurrentFBOValid = true;
            // 保存当前屏幕帧缓冲（可能不是 0，如在 Cocos2d 环境中）
            glGetIntegerv(GL_FRAMEBUFFER_BINDING, &_screenFrameBuffer);
            // 绑定离屏 FBO
            glBindFramebuffer(GL_FRAMEBUFFER, _FBO);
        }
        
        // 避免重复绑定相同目标（优化）
        if(_CurrentRenderTarget == t)
            return;
            
        // 将纹理附加为颜色附件
        glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,
                               GL_TEXTURE_2D, t, 0);
    } else {
        // 恢复屏幕渲染
        glBindFramebuffer(GL_FRAMEBUFFER, _screenFrameBuffer);
        _CurrentFBOValid = false;
    }
    _CurrentRenderTarget = t;
}
```

### FBO 切换流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    TVPSetRenderTarget 流程                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   TVPSetRenderTarget(texID)                                     │
│           │                                                      │
│           ▼                                                      │
│   ┌───────────────────┐                                         │
│   │   texID != 0 ?    │                                         │
│   └─────────┬─────────┘                                         │
│         Yes │  No                                               │
│             │    │                                               │
│   ┌─────────▼────│──────────────────┐                           │
│   │ _CurrentFBOValid == false?      │                           │
│   │ (首次切换到 FBO)                │                           │
│   └─────────┬───────────────────────┘                           │
│         Yes │  No                                               │
│             │    │                                               │
│   ┌─────────▼────│──────────────────┐                           │
│   │ 保存 _screenFrameBuffer         │                           │
│   │ 绑定 _FBO                       │                           │
│   │ _CurrentFBOValid = true         │                           │
│   └─────────┬───────────────────────┘                           │
│             │                                                    │
│   ┌─────────▼───────────────────────┐   ┌───────────────────┐   │
│   │ _CurrentRenderTarget != texID?  │   │ 恢复屏幕 FBO      │   │
│   │ (避免重复绑定)                   │   │ _CurrentFBOValid  │   │
│   └─────────┬───────────────────────┘   │     = false       │   │
│         Yes │  No                        └─────────┬─────────┘   │
│             │    │                                 │             │
│   ┌─────────▼────│──────────────────┐              │             │
│   │ glFramebufferTexture2D          │              │             │
│   │ (附加纹理到颜色附件)             │              │             │
│   └─────────┬───────────────────────┘              │             │
│             │                                       │             │
│   ┌─────────▼───────────────────────────────────────▼─────────┐ │
│   │            _CurrentRenderTarget = texID                    │ │
│   └────────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 状态恢复

渲染完成后，需要恢复 GL 状态以免影响后续操作：

```cpp
// 来源：cpp/core/visual/ogl/RenderManager_ogl.cpp 第 511-518 行
static void _RestoreGLStatues() {
    // 重置行宽度设置
    if(GL_CHECK_unpack_subimage) {
        glPixelStorei(GL_UNPACK_ROW_LENGTH, 0);
    }
    
    // 重置混合状态缓存（Cocos2d-x 内部状态管理）
    cocos2d::GL::blendResetToCache();
    
    // 恢复屏幕渲染目标
    TVPSetRenderTarget(0);
    
    // 恢复视口
    cocos2d::Director::getInstance()->setViewport();
}
```

---

## 批量渲染实现

### OperateRect：矩形区域渲染

`OperateRect` 是最常用的渲染接口，用于处理矩形区域的纹理混合。适用场景：

- 图层叠加（背景、角色、UI）
- 矩形裁剪区域渲染
- 2D 精灵绘制

#### 函数签名

```cpp
void OperateRect(
    iTVPRenderMethod *_method,      // 渲染方法（Shader）
    iTVPTexture2D *_tar,            // 目标纹理
    iTVPTexture2D *reftar,          // 参考目标（用于复制避免读写冲突）
    const tTVPRect &rctar,          // 目标矩形区域
    const tRenderTexRectArray &textures  // 源纹理数组
);
```

#### 核心实现分析

```cpp
// 来源：cpp/core/visual/ogl/RenderManager_ogl.cpp 第 4498-4636 行
void OperateRect(iTVPRenderMethod *_method, iTVPTexture2D *_tar,
                 iTVPTexture2D *reftar, const tTVPRect &rctar,
                 const tRenderTexRectArray &textures) override {
    ++_drawCount;  // 统计绘制调用次数（性能分析用）
    
    tTVPOGLRenderMethod *method = (tTVPOGLRenderMethod *)_method;
    tTVPOGLTexture2D *tar = (tTVPOGLTexture2D *)_tar;
    
    // 如果引用目标和目标相同，清除引用
    if(reftar == _tar)
        reftar = nullptr;
    
    // 检查是否有自定义处理过程（特殊效果）
    if(method->CustomProc &&
       method->CustomProc(method, tar, (tTVPOGLTexture2D *)reftar, 
                          &rctar, textures)) {
        return;  // 自定义处理完成，直接返回
    }

    // 准备纹理顶点数据
    std::vector<GLVertexInfo> texlist;
    texlist.resize(textures.size() + (method->tar_as_src ? 1 : 0));
    
    for(unsigned int i = 0; i < textures.size(); ++i) {
        tTVPOGLTexture2D *tex = (tTVPOGLTexture2D *)(textures[i].first);
        tex->SyncPixel();  // 同步 CPU 像素数据（如有待上传数据）
        
        GLVertexInfo &texitem = texlist[i];
        
        // 特殊处理：源纹理和目标纹理相同（读写冲突）
        if(tex == tar) {
            // 需要复制目标纹理以避免 GPU 读写冲突
            tTVPOGLTexture2D *newtex;
            tTVPRect rc = textures[i].second;
            if(reftar) {
                newtex = (tTVPOGLTexture2D *)reftar;
            } else {
                newtex = GetTempTexture2D(tex, rc);  // 获取临时纹理
                rc.set_offsets(0, 0);
            }
            newtex->ApplyVertex(texitem, rc);
        } else {
            tex->ApplyVertex(texitem, textures[i].second);
        }
    }
    
    // 标准化顶点坐标（NDC：-1 到 1）
    // 两个三角形构成一个矩形
    static const GLfloat vertices[12] = {
        -1, -1,  1, -1, -1,  1,   // 三角形 1：左下、右下、左上
         1, -1, -1,  1,  1,  1    // 三角形 2：右下、左上、右上
    };
    
    // 应用渲染方法（绑定 Shader 程序）
    method->Apply();
    
    // 设置渲染目标
    tar->AsTarget();
    
    // 设置视口（考虑纹理缩放）
    glViewport(rctar.left * tar->_scaleW, rctar.top * tar->_scaleH,
               rctar.get_width() * tar->_scaleW,
               rctar.get_height() * tar->_scaleH);
    
    // 启用顶点属性
    int VA_flag = 1 << method->GetPosAttr();
    for(unsigned int i = 0; i < texlist.size(); ++i) {
        VA_flag |= 1 << method->GetTexCoordAttr(i);
    }
    cocos2d::GL::enableVertexAttribs(VA_flag);
    
    // 设置顶点位置数据
    glVertexAttribPointer(method->GetPosAttr(), 2, GL_FLOAT, GL_FALSE, 0,
                          vertices);
    
    // 绑定纹理并设置纹理坐标
    for(unsigned int i = 0; i < texlist.size(); ++i) {
        method->ApplyTexture(i, texlist[i]);
    }
    
    // 执行绘制（6 个顶点 = 2 个三角形）
    glDrawArrays(GL_TRIANGLES, 0, 6);
    
    // 清理
    method->onFinish();
    CHECK_GL_ERROR_DEBUG();
}
```

### OperateTriangles：三角形网格渲染

用于更复杂的几何变换，如透视、扭曲、仿射变换等效果：

```cpp
// 来源：cpp/core/visual/ogl/RenderManager_ogl.cpp 第 4639-4799 行
void OperateTriangles(iTVPRenderMethod *_method, int nTriangles,
                      iTVPTexture2D *_tar, iTVPTexture2D *reftar,
                      const tTVPRect &rcclip, const tTVPPointD *_pttar,
                      const tRenderTexQuadArray &textures) override {
    ++_drawCount;
    
    tTVPOGLRenderMethod *method = (tTVPOGLRenderMethod *)_method;
    tTVPOGLTexture2D *tar = (tTVPOGLTexture2D *)_tar;
    int ptcount = nTriangles * 3;  // 总顶点数
    
    // 准备纹理顶点数据
    std::vector<GLVertexInfo> texlist;
    texlist.resize(textures.size() + (method->tar_as_src ? 1 : 0));
    
    for(unsigned int i = 0; i < textures.size(); ++i) {
        tTVPOGLTexture2D *tex = (tTVPOGLTexture2D *)(textures[i].first);
        tex->SyncPixel();
        GLVertexInfo &texitem = texlist[i];
        
        if(tex == tar) {
            // 处理读写冲突
            tTVPOGLTexture2D *newtex;
            if(reftar)
                newtex = (tTVPOGLTexture2D *)reftar;
            else {
                newtex = GetTempTexture2D(
                    tex, tTVPRect(0, 0, tex->GetWidth(), tex->GetHeight()));
            }
            newtex->ApplyVertex(texitem, textures[i].second, ptcount);
        } else {
            tex->ApplyVertex(texitem, textures[i].second, ptcount);
        }
    }
    
    // 转换目标坐标到 NDC（核心算法）
    std::vector<GLfloat> pttar;
    float rcw = rcclip.get_width(), rch = rcclip.get_height();
    float rcl = rcclip.left, rct = rcclip.top;
    pttar.resize(ptcount * 2);
    
    for(int i = 0; i < ptcount; ++i) {
        // 从像素坐标转换到 [-1, 1] NDC
        // 公式：ndc = ((pixel - clip_origin) / clip_size - 0.5) * 2
        pttar[i * 2 + 0] = (((GLfloat)_pttar[i].x - rcl) / rcw - 0.5f) * 2.f;
        pttar[i * 2 + 1] = (((GLfloat)_pttar[i].y - rct) / rch - 0.5f) * 2.f;
    }
    
    // 设置渲染目标和视口
    tar->AsTarget();
    glViewport(rcl * tar->_scaleW, rct * tar->_scaleH,
               rcw * tar->_scaleW, rch * tar->_scaleH);
    
    // 应用 Shader
    method->Apply();
    
    // 启用顶点属性
    int VA_flag = 1 << method->GetPosAttr();
    for(unsigned int i = 0; i < texlist.size(); ++i) {
        VA_flag |= 1 << method->GetTexCoordAttr(i);
    }
    cocos2d::GL::enableVertexAttribs(VA_flag);
    
    // 设置顶点位置
    glVertexAttribPointer(method->GetPosAttr(), 2, GL_FLOAT, GL_FALSE, 0,
                          &pttar.front());
    
    // 设置纹理坐标
    for(unsigned int i = 0; i < texlist.size(); ++i) {
        method->ApplyTexture(i, texlist[i]);
    }
    
    // 绘制三角形
    glDrawArrays(GL_TRIANGLES, 0, ptcount);
    
    method->onFinish();
    CHECK_GL_ERROR_DEBUG();
}
```

---

## 顶点数据计算

### ApplyVertex：矩形顶点生成

`ApplyVertex` 负责根据源矩形区域生成纹理坐标：

```cpp
// 来源：cpp/core/visual/ogl/RenderManager_ogl.cpp 第 1104-1132 行
virtual void ApplyVertex(GLVertexInfo &vtx, const tTVPRect &rc) {
    vtx.tex = this;
    
    // 计算纹理坐标边界（像素坐标）
    GLfloat sminu, smaxu, sminv, smaxv;
    sminu = (GLfloat)rc.left;
    smaxu = (GLfloat)rc.right;
    sminv = (GLfloat)rc.top;
    smaxv = (GLfloat)rc.bottom;
    
    // 归一化到 [0, 1] 并考虑缩放
    float tw = _scaleW / internalW;  // 每像素的纹理坐标增量
    float th = _scaleH / internalH;
    sminu *= tw;
    smaxu *= tw;
    sminv *= th;
    smaxv *= th;
    
    // 生成 6 个顶点（2 个三角形）的纹理坐标
    vtx.vtx.resize(6 * 2);  // 6 顶点 * 2 分量(u, v)
    GLfloat *pt = &vtx.vtx.front();
    
    // 三角形 1：左上、右上、左下
    pt[0] = sminu;  pt[1] = sminv;   // 顶点 0
    pt[2] = smaxu;  pt[3] = sminv;   // 顶点 1
    pt[4] = sminu;  pt[5] = smaxv;   // 顶点 2
    
    // 三角形 2：右上、左下、右下
    pt[6] = smaxu;  pt[7] = sminv;   // 顶点 3
    pt[8] = sminu;  pt[9] = smaxv;   // 顶点 4
    pt[10] = smaxu; pt[11] = smaxv;  // 顶点 5
}
```

### 三角形构成示意图

```
纹理坐标空间：

(sminu, sminv)───────────(smaxu, sminv)
      │  ╲                     │
      │    ╲   三角形 1        │
      │      ╲                 │
      │        ╲               │
      │          ╲             │
      │   三角形 2  ╲           │
      │              ╲         │
(sminu, smaxv)───────────(smaxu, smaxv)

三角形 1: 顶点 0, 1, 2 → (sminu,sminv), (smaxu,sminv), (sminu,smaxv)
三角形 2: 顶点 3, 4, 5 → (smaxu,sminv), (sminu,smaxv), (smaxu,smaxv)

注意：顶点顺序决定正面/背面（背面剔除）
```

### NDC 坐标转换公式

`OperateTriangles` 中将像素坐标转换为 NDC 坐标的公式：

```cpp
ndc_x = ((pixel_x - clip_left) / clip_width - 0.5) * 2.0
ndc_y = ((pixel_y - clip_top) / clip_height - 0.5) * 2.0
```

**推导过程**：

| 步骤 | 公式 | 范围 |
|------|------|------|
| 1. 相对坐标 | `(pixel - origin) / size` | [0, 1] |
| 2. 中心化 | `... - 0.5` | [-0.5, 0.5] |
| 3. NDC 缩放 | `... * 2.0` | [-1, 1] |

---

## 对照项目源码

### 关键文件定位

| 文件 | 行号 | 功能 |
|------|------|------|
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 469-518 | FBO 管理相关全局变量和函数 |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 493-509 | `TVPSetRenderTarget` |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 4498-4636 | `OperateRect` |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 4639-4799 | `OperateTriangles` |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 1104-1132 | `ApplyVertex` |

### GLVertexInfo 结构

```cpp
// 顶点信息结构
struct GLVertexInfo {
    tTVPOGLTexture2D *tex;    // 关联的纹理对象
    std::vector<GLfloat> vtx;  // 纹理坐标数组
};
```

---

## 本节小结

- **FBO 管理**使用单一共享 FBO，通过 `TVPSetRenderTarget` 切换渲染目标
- **避免重复绑定**：通过 `_CurrentRenderTarget` 缓存减少 GL 调用
- **OperateRect** 用于矩形区域渲染，固定 6 顶点（2 三角形）
- **OperateTriangles** 用于任意三角形网格，支持复杂几何变换
- **读写冲突处理**：当源纹理等于目标纹理时，使用临时纹理避免未定义行为
- **NDC 转换**：像素坐标通过归一化、中心化、缩放转换到 [-1, 1]

---

## 练习题与答案

### 题目 1：为什么需要临时纹理？

**问题**：在 `OperateRect` 中，当源纹理和目标纹理相同时，为什么需要 `GetTempTexture2D` 获取临时纹理？

<details>
<summary>查看答案</summary>

**原因**：在 GPU 中，同一纹理不能同时作为渲染目标（写入）和采样源（读取）。这会导致**读写冲突（Read-After-Write Hazard）**。

**具体问题**：
```cpp
// 错误示例：src == tar
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, 
                       GL_TEXTURE_2D, tar->texture, 0);  // 写入目标
glBindTexture(GL_TEXTURE_2D, tar->texture);  // 同时采样
glDrawArrays(...);  // 未定义行为！
```

**解决方案**：
1. 将源纹理内容复制到临时纹理
2. 使用临时纹理作为采样源
3. 渲染到原始目标纹理

```cpp
if(tex == tar) {
    tTVPOGLTexture2D *newtex = GetTempTexture2D(tex, rc);
    newtex->ApplyVertex(texitem, ...);
}
```

**性能考虑**：
- 临时纹理增加显存使用和复制开销
- KrKr2 使用纹理池缓存临时纹理以减少分配次数
- `reftar` 参数允许外部提供临时纹理，避免重复创建

</details>

### 题目 2：NDC 坐标转换

**问题**：一个 200x200 的裁剪区域，左上角在 (100, 50)。像素点 (200, 150) 在 NDC 中的坐标是多少？

<details>
<summary>查看答案</summary>

**给定**：
- 裁剪区域：left=100, top=50, width=200, height=200
- 像素点：(200, 150)

**计算**：
```cpp
// X 轴
rel_x = (200 - 100) / 200.0 = 0.5      // 相对坐标
ndc_x = (0.5 - 0.5) * 2.0 = 0.0        // NDC

// Y 轴
rel_y = (150 - 50) / 200.0 = 0.5       // 相对坐标
ndc_y = (0.5 - 0.5) * 2.0 = 0.0        // NDC
```

**答案**：NDC 坐标为 **(0.0, 0.0)**，即裁剪区域的正中心。

**验证**：
- 像素 (200, 150) 相对于 (100, 50) 的偏移是 (100, 100)
- 区域尺寸 200x200，所以 (100, 100) 正好是中心点
- 中心点在 NDC 中映射到原点 (0, 0) ✓

</details>

---

## 下一步

[03-动手实践与练习](./03-动手实践与练习.md) — 通过完整代码示例巩固 FBO 和批量渲染知识
