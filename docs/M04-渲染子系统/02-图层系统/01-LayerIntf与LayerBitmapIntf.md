# LayerIntf 与 LayerBitmapIntf

> **所属模块：** M04-渲染子系统
> **前置知识：** [01-visual模块总览](../01-visual模块总览/)、[P03-跨平台C++开发](../../P03-跨平台C++开发/)
> **预计阅读时间：** 40 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 `tTJSNI_BaseLayer` 类的核心成员变量及其职责
2. 掌握图层的 **双缓冲图像系统**（MainImage + ProvinceImage）
3. 解释图层坐标系（Layer 坐标 vs Image 坐标）的区别与转换
4. 使用 `iTVPBaseBitmap` 接口进行像素级操作
5. 追踪图层图像的创建、分配与释放流程

---

## 1. tTJSNI_BaseLayer 类总览

### 1.1 类的定位与职责

`tTJSNI_BaseLayer` 是 KrKr2 图层系统的**核心基类**，定义在 `cpp/core/visual/LayerIntf.h` 中。它承担以下职责：

| 职责 | 说明 |
|------|------|
| **TJS 脚本绑定** | 继承 `tTJSNativeInstance`，使图层可被 TJS 脚本创建和操作 |
| **树状结构管理** | 维护父子关系（Parent/Children），实现图层嵌套 |
| **图像缓冲管理** | 持有 MainImage（主图像）和 ProvinceImage（区域图像） |
| **渲染参与** | 实现 `tTVPDrawable` 接口，参与渲染管线 |
| **事件处理** | 处理鼠标、键盘、触摸等输入事件 |
| **过渡效果** | 支持图层间的过渡动画（Transition） |

### 1.2 继承关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    tTJSNI_BaseLayer 继承体系                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  tTJSNativeInstance          tTVPDrawable     tTVPCompactEventCallbackIntf
│         │                         │                    │            │
│         │   TJS 脚本对象基类       │   可绘制接口        │  内存紧凑回调  │
│         └─────────────┬───────────┴────────────────────┘            │
│                       │                                             │
│                       ▼                                             │
│              ┌─────────────────┐                                    │
│              │tTJSNI_BaseLayer │  ← 图层基类（平台无关）              │
│              └────────┬────────┘                                    │
│                       │                                             │
│                       │ #include "LayerImpl.h"                      │
│                       ▼                                             │
│              ┌─────────────────┐                                    │
│              │  tTJSNI_Layer   │  ← 平台实现类                       │
│              └─────────────────┘                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.3 核心成员变量分类

以下是 `tTJSNI_BaseLayer` 中最关键的成员变量，按功能分类：

```cpp
// 文件: cpp/core/visual/LayerIntf.h（第 165-430 行摘录）

class tTJSNI_BaseLayer : public tTJSNativeInstance,
                         public tTVPDrawable,
                         public tTVPCompactEventCallbackIntf {
protected:
    // ============ 对象所有权 ============
    iTJSDispatch2 *Owner;           // 拥有此图层的 TJS 对象指针
    tTJSVariantClosure ActionOwner; // 事件处理器闭包（用于回调 TJS 脚本）

    // ============ 树状结构 ============
    tTVPLayerManager *Manager;      // 所属的图层管理器
    tTJSNI_BaseLayer *Parent;       // 父图层指针（nullptr 表示根图层）
    tObjectList<tTJSNI_BaseLayer> Children;  // 子图层列表
    ttstr Name;                     // 图层名称（用于调试和脚本访问）
    bool Visible;                   // 可见性标志
    tjs_int Opacity;                // 不透明度（0-255）
    tjs_uint OrderIndex;            // 在父图层中的顺序索引

    // ============ 图层类型 ============
    tTVPLayerType Type;             // 用户设置的图层类型
    tTVPLayerType DisplayType;      // 实际显示类型（可能与 Type 不同）

    // ============ 几何信息 ============
    tTVPRect Rect;                  // 图层矩形（相对于父图层的位置和大小）
    tTVPComplexRect ExposedRegion;  // 暴露区域（未被子图层遮挡的部分）
    tTVPComplexRect OverlappedRegion; // 被遮挡区域

    // ============ 图像缓冲 ============
    tTVPBaseTexture *MainImage;     // 主图像（ARGB 像素数据）
    tTVPBaseBitmap *ProvinceImage;  // 区域图像（用于点击测试等）
    bool CanHaveImage;              // 是否可以拥有图像
    tjs_int ImageLeft;              // 图像在图层中的 X 偏移
    tjs_int ImageTop;               // 图像在图层中的 Y 偏移
    tjs_uint32 NeutralColor;        // 中性色（用户可设置）
    tjs_uint32 TransparentColor;    // 透明色（由图层类型决定）

    // ============ 缓存管理 ============
    tTVPBaseTexture *CacheBitmap;   // 缓存位图（用于加速渲染）
    tjs_uint CacheEnabledCount;     // 缓存启用计数
    bool Cached;                    // 脚本控制的缓存状态

    // ============ 绘图状态 ============
    tTVPDrawFace DrawFace;          // 实际绘图面（Main/Mask/Province）
    tTVPDrawFace Face;              // 对外暴露的绘图面
    bool HoldAlpha;                 // 是否保持 Alpha 通道
    tTVPRect ClipRect;              // 裁剪矩形
    bool ImageModified;             // 图像是否已修改
};
```

**关键点解析**：

1. **双图像系统**：`MainImage` 存储可见像素，`ProvinceImage` 存储区域标识（用于点击检测）
2. **坐标偏移**：`ImageLeft`/`ImageTop` 允许图像在图层内部滚动
3. **缓存机制**：`CacheBitmap` 用于存储预合成的渲染结果，避免重复绘制

---

## 2. 图像缓冲系统

### 2.1 MainImage 与 ProvinceImage

KrKr2 采用**双图像架构**来分离显示数据和元数据：

```
┌─────────────────────────────────────────────────────────────────────┐
│                       图层双图像系统                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────┐         ┌─────────────────────┐           │
│  │     MainImage       │         │   ProvinceImage     │           │
│  │  (tTVPBaseTexture)  │         │  (tTVPBaseBitmap)   │           │
│  ├─────────────────────┤         ├─────────────────────┤           │
│  │ 格式: 32bpp ARGB    │         │ 格式: 8bpp 索引     │           │
│  │ 内容: 可见像素数据   │         │ 内容: 区域标识符     │           │
│  │ 用途: 渲染到屏幕    │         │ 用途: 点击测试       │           │
│  │      文本绘制       │         │      区域判定        │           │
│  │      图形操作       │         │      事件路由        │           │
│  └─────────────────────┘         └─────────────────────┘           │
│           │                               │                         │
│           │ 同步尺寸                       │                         │
│           └───────────────┬───────────────┘                         │
│                           ▼                                         │
│                  ┌─────────────────┐                                │
│                  │  Layer Rect     │                                │
│                  │  (图层边界)      │                                │
│                  └─────────────────┘                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**MainImage 详解**：

```cpp
// 文件: cpp/core/visual/LayerIntf.h（第 476-479 行）

tTVPBaseTexture *GetMainImage() {
    ApplyFont();       // 确保字体已应用
    return MainImage;  // 返回主图像指针
}
```

- **类型**：`tTVPBaseTexture`（支持 GPU 纹理上传）
- **像素格式**：32 位 ARGB（每像素 4 字节）
- **用途**：所有可见内容的绘制目标

**ProvinceImage 详解**：

```cpp
// 文件: cpp/core/visual/LayerIntf.h（第 480-484 行）

tTVPBaseBitmap *GetProvinceImage() {
    ApplyFont();          // 确保字体已应用
    return ProvinceImage; // 返回区域图像指针
}
```

- **类型**：`tTVPBaseBitmap`（纯 CPU 位图）
- **像素格式**：8 位索引（每像素 1 字节）
- **用途**：存储区域 ID，用于判断鼠标点击的目标区域

### 2.2 图像分配与释放

图层图像的生命周期由以下方法管理：

```cpp
// 文件: cpp/core/visual/LayerIntf.cpp（第 1500-1580 行概要）

// 分配主图像
void tTJSNI_BaseLayer::AllocateImage() {
    if(!CanHaveImage) return;  // 检查是否允许分配
    if(MainImage) return;      // 已存在则跳过
    
    // 创建与图层尺寸匹配的纹理
    tjs_uint w = Rect.get_width();
    tjs_uint h = Rect.get_height();
    MainImage = new tTVPBaseTexture(w, h, 32);
    
    // 填充透明色
    MainImage->Fill(tTVPRect(0, 0, w, h), TransparentColor);
}

// 释放主图像
void tTJSNI_BaseLayer::DeallocateImage() {
    if(MainImage) {
        delete MainImage;
        MainImage = nullptr;
    }
}

// 分配区域图像
void tTJSNI_BaseLayer::AllocateProvinceImage() {
    if(!CanHaveImage) return;
    if(ProvinceImage) return;
    
    tjs_uint w = Rect.get_width();
    tjs_uint h = Rect.get_height();
    ProvinceImage = new tTVPBaseBitmap(w, h, 8);  // 8bpp
    ProvinceImage->Fill(tTVPRect(0, 0, w, h), 0); // 填充 0（无区域）
}

// 释放区域图像
void tTJSNI_BaseLayer::DeallocateProvinceImage() {
    if(ProvinceImage) {
        delete ProvinceImage;
        ProvinceImage = nullptr;
    }
}
```

**生命周期流程**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    图像生命周期管理                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   new Layer()          SetHasImage(true)      SetSize(w, h)        │
│       │                      │                      │              │
│       ▼                      ▼                      ▼              │
│  AllocateDefaultImage   AllocateImage        ChangeImageSize       │
│       │                      │                      │              │
│       │                      │                      │              │
│       └──────────────────────┴──────────────────────┘              │
│                              │                                      │
│                              ▼                                      │
│                      MainImage 存在                                  │
│                              │                                      │
│       ┌──────────────────────┴──────────────────────┐              │
│       │                                             │              │
│       ▼                                             ▼              │
│  SetHasImage(false)                           Invalidate()         │
│       │                                             │              │
│       ▼                                             ▼              │
│  DeallocateImage                              DeallocateImage      │
│       │                                       DeallocateCache      │
│       ▼                                             │              │
│  MainImage = nullptr                                ▼              │
│                                               对象销毁              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 图像尺寸变更

当图层大小改变时，图像需要同步调整：

```cpp
// 文件: cpp/core/visual/LayerIntf.cpp（第 1620-1680 行概要）

void tTJSNI_BaseLayer::ChangeImageSize(tjs_uint width, tjs_uint height) {
    // 检查是否需要变更
    if(!MainImage) return;
    if(MainImage->GetWidth() == width && 
       MainImage->GetHeight() == height) return;
    
    // 调整主图像大小（使用填充扩展）
    MainImage->SetSizeWithFill(width, height, NeutralColor);
    
    // 同步调整区域图像
    if(ProvinceImage) {
        ProvinceImage->SetSizeWithFill(width, height, 0);
    }
    
    // 重置裁剪区域
    ResetClip();
    
    // 标记图像已修改
    ImageModified = true;
}
```

---

## 3. 坐标系统详解

### 3.1 三种坐标系

KrKr2 图层系统使用三种坐标系，理解它们的关系是正确操作图层的基础：

```
┌─────────────────────────────────────────────────────────────────────┐
│                       坐标系统关系图                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─── Primary (Window) 坐标 ────────────────────────────────────┐  │
│  │                                                               │  │
│  │   (0,0)──────────────────────────────────────────▶ X         │  │
│  │     │                                                         │  │
│  │     │    ┌─── Parent Layer 坐标 ──────────────────────────┐  │  │
│  │     │    │                                                 │  │  │
│  │     │    │   (Rect.left, Rect.top)                        │  │  │
│  │     │    │        │                                        │  │  │
│  │     │    │        ▼                                        │  │  │
│  │     │    │   ┌─── Layer 坐标 ─────────────────────────┐   │  │  │
│  │     │    │   │                                         │   │  │  │
│  │     │    │   │  (0,0)───────────────────────▶          │   │  │  │
│  │     │    │   │    │                                    │   │  │  │
│  │     │    │   │    │  ┌─── Image 坐标 ──────────────┐  │   │  │  │
│  │     │    │   │    │  │                              │  │   │  │  │
│  │     │    │   │    │  │ (ImageLeft, ImageTop)       │  │   │  │  │
│  │     │    │   │    │  │       │                      │  │   │  │  │
│  │     │    │   │    │  │       ▼                      │  │   │  │  │
│  │     │    │   │    │  │  (0,0)──────▶ 图像像素       │  │   │  │  │
│  │     │    │   │    │  │       │                      │  │   │  │  │
│  │     │    │   │    │  │       ▼                      │  │   │  │  │
│  │     │    │   │    │  └──────────────────────────────┘  │   │  │  │
│  │     │    │   │    ▼                                    │   │  │  │
│  │     │    │   └─────────────────────────────────────────┘   │  │  │
│  │     │    │                                                 │  │  │
│  │     │    └─────────────────────────────────────────────────┘  │  │
│  │     ▼                                                         │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

| 坐标系 | 原点 | 用途 |
|--------|------|------|
| **Primary 坐标** | 窗口左上角 | 鼠标事件、最终渲染输出 |
| **Layer 坐标** | 图层左上角（相对于父图层） | 图层定位、子图层布局 |
| **Image 坐标** | 图像左上角 | 像素绘制、位图操作 |

### 3.2 坐标转换方法

```cpp
// 文件: cpp/core/visual/LayerIntf.h（第 409-413 行）

// 将坐标转换为 Primary（窗口）坐标系
void ToPrimaryCoordinates(tjs_int &x, tjs_int &y) const;

// 从 Primary 坐标系转换回图层坐标系
void FromPrimaryCoordinates(tjs_int &x, tjs_int &y) const;
void FromPrimaryCoordinates(tjs_real &x, tjs_real &y) const;  // 浮点版本
```

**实现原理**：

```cpp
// 文件: cpp/core/visual/LayerIntf.cpp（概念实现）

void tTJSNI_BaseLayer::ToPrimaryCoordinates(tjs_int &x, tjs_int &y) const {
    // 递归向上累加父图层的偏移
    const tTJSNI_BaseLayer *layer = this;
    while(layer) {
        x += layer->Rect.left;
        y += layer->Rect.top;
        layer = layer->Parent;
    }
}

void tTJSNI_BaseLayer::FromPrimaryCoordinates(tjs_int &x, tjs_int &y) const {
    // 递归向上减去父图层的偏移
    const tTJSNI_BaseLayer *layer = this;
    while(layer) {
        x -= layer->Rect.left;
        y -= layer->Rect.top;
        layer = layer->Parent;
    }
}
```

### 3.3 Layer 坐标到 Image 坐标

图像可能在图层内部有偏移（例如实现滚动效果）：

```cpp
// 从 Layer 坐标转换为 Image 坐标
inline tjs_int LayerToImageX(tjs_int layerX) const {
    return layerX - ImageLeft;
}

inline tjs_int LayerToImageY(tjs_int layerY) const {
    return layerY - ImageTop;
}

// 从 Image 坐标转换为 Layer 坐标
inline tjs_int ImageToLayerX(tjs_int imageX) const {
    return imageX + ImageLeft;
}

inline tjs_int ImageToLayerY(tjs_int imageY) const {
    return imageY + ImageTop;
}
```

**应用示例**：

```cpp
// 在图层的 (100, 50) 位置绘制文本
// 但图像偏移为 ImageLeft=20, ImageTop=10
// 则实际绘制到图像的 (80, 40) 位置

tjs_int layerX = 100, layerY = 50;
tjs_int imageX = layerX - ImageLeft;  // 100 - 20 = 80
tjs_int imageY = layerY - ImageTop;   // 50 - 10 = 40

MainImage->DrawText(imageX, imageY, text, color);
```

---

## 4. iTVPBaseBitmap 接口

### 4.1 接口概述

`iTVPBaseBitmap` 是所有位图操作的基础接口，定义在 `LayerBitmapIntf.h` 中：

```cpp
// 文件: cpp/core/visual/LayerBitmapIntf.h（第 139-242 行摘录）

class iTVPBaseBitmap : public tTVPNativeBaseBitmap {
public:
    // ============ 尺寸操作 ============
    void SetSizeWithFill(tjs_uint w, tjs_uint h, tjs_uint32 fillvalue);
    
    // ============ 像素访问 ============
    tjs_uint32 GetPoint(tjs_int x, tjs_int y) const;      // 读取像素
    bool SetPoint(tjs_int x, tjs_int y, tjs_uint32 value); // 设置像素
    bool SetPointMain(tjs_int x, tjs_int y, tjs_uint32 color);  // 设置颜色
    bool SetPointMask(tjs_int x, tjs_int y, tjs_int mask);      // 设置透明度
    
    // ============ 填充操作 ============
    virtual bool Fill(tTVPRect rect, tjs_uint32 value);
    bool FillColor(tTVPRect rect, tjs_uint32 color, tjs_int opa);
    bool FillColorOnAlpha(tTVPRect rect, tjs_uint32 color, tjs_int opa);
    bool FillMask(tTVPRect rect, tjs_int value);
    
    // ============ 复制操作 ============
    virtual bool CopyRect(tjs_int x, tjs_int y, 
                          const iTVPBaseBitmap *ref, tTVPRect refrect);
    bool Copy9Patch(const iTVPBaseBitmap *ref, tTVPRect &margin);
    
    // ============ 混合操作 ============
    bool Blt(tjs_int x, tjs_int y, const iTVPBaseBitmap *ref, 
             tTVPRect refrect, tTVPBBBltMethod method, 
             tjs_int opa, bool hda = true);
    
    // ============ 缩放操作 ============
    bool StretchBlt(tTVPRect cliprect, tTVPRect destrect,
                    const iTVPBaseBitmap *ref, tTVPRect refrect,
                    tTVPBBBltMethod method, tjs_int opa, bool hda = true,
                    tTVPBBStretchType mode = stNearest, tjs_real typeopt = 0.0);
    
    // ============ 仿射变换 ============
    bool AffineBlt(tTVPRect destrect, const iTVPBaseBitmap *ref,
                   tTVPRect refrect, const tTVPPointD *points,
                   tTVPBBBltMethod method, tjs_int opa, ...);
    
    // ============ 效果处理 ============
    bool DoBoxBlur(const tTVPRect &rect, const tTVPRect &area);
    void DoGrayScale(tTVPRect rect);
    void AdjustGamma(tTVPRect rect, const tTVPGLGammaAdjustData &data);
    void UDFlip(const tTVPRect &rect);  // 上下翻转
    void LRFlip(const tTVPRect &rect);  // 左右翻转
};
```

### 4.2 像素访问方法

**读取像素**：

```cpp
// 文件: cpp/core/visual/LayerBitmapIntf.cpp（第 165-177 行）

tjs_uint32 iTVPBaseBitmap::GetPoint(tjs_int x, tjs_int y) const {
    // 边界检查
    if(x < 0 || y < 0 || 
       x >= (tjs_int)GetWidth() || y >= (tjs_int)GetHeight())
        TVPThrowExceptionMessage(TVPOutOfRectangle);
    
    // 从底层位图获取像素值
    return Bitmap->GetPoint(x, y);
}
```

**设置像素**：

```cpp
// 文件: cpp/core/visual/LayerBitmapIntf.cpp（第 179-226 行）

bool iTVPBaseBitmap::SetPoint(tjs_int x, tjs_int y, tjs_uint32 value) {
    // 边界检查
    if(x < 0 || y < 0 || 
       x >= (tjs_int)GetWidth() || y >= (tjs_int)GetHeight())
        TVPThrowExceptionMessage(TVPOutOfRectangle);
    
    // 颜色格式转换（BGR ↔ RGB）
    Bitmap->SetPoint(x, y, TVP_REVRGB(value));
    return true;
}

// 只设置颜色（保持透明度不变）
bool iTVPBaseBitmap::SetPointMain(tjs_int x, tjs_int y, tjs_uint32 color) {
    if(!Is32BPP())
        TVPThrowExceptionMessage(TVPInvalidOperationFor8BPP);
    
    // 保留原有 Alpha，替换 RGB
    tjs_uint32 clr = Bitmap->GetPoint(x, y);
    clr &= 0xff000000;                    // 保留 Alpha
    clr += TVP_REVRGB(color) & 0xffffff;  // 设置 RGB
    Bitmap->SetPoint(x, y, clr);
    return true;
}

// 只设置透明度（保持颜色不变）
bool iTVPBaseBitmap::SetPointMask(tjs_int x, tjs_int y, tjs_int mask) {
    if(!Is32BPP())
        TVPThrowExceptionMessage(TVPInvalidOperationFor8BPP);
    
    tjs_uint32 clr = Bitmap->GetPoint(x, y);
    clr &= 0x00ffffff;               // 保留 RGB
    clr += (mask & 0xff) << 24;      // 设置 Alpha
    Bitmap->SetPoint(x, y, clr);
    return true;
}
```

### 4.3 混合方法枚举

`tTVPBBBltMethod` 定义了位图混合的各种模式：

```cpp
// 文件: cpp/core/visual/LayerBitmapIntf.h（第 28-61 行）

enum tTVPBBBltMethod {
    // ===== 基础混合 =====
    bmCopy,             // 直接复制
    bmCopyOnAlpha,      // 复制到 Alpha 图层
    bmAlpha,            // Alpha 混合
    bmAlphaOnAlpha,     // Alpha 混合到 Alpha 图层
    
    // ===== 数学混合 =====
    bmAdd,              // 加法混合
    bmSub,              // 减法混合
    bmMul,              // 乘法混合
    bmDodge,            // 颜色减淡
    bmDarken,           // 变暗
    bmLighten,          // 变亮
    bmScreen,           // 滤色
    
    // ===== AddAlpha 系列 =====
    bmAddAlpha,              // 加法 Alpha 混合
    bmAddAlphaOnAddAlpha,    // AddAlpha 到 AddAlpha
    bmAddAlphaOnAlpha,       // AddAlpha 到 Alpha
    bmAlphaOnAddAlpha,       // Alpha 到 AddAlpha
    bmCopyOnAddAlpha,        // Copy 到 AddAlpha
    
    // ===== Photoshop 兼容混合 =====
    bmPsNormal,         // PS 正常
    bmPsAdditive,       // PS 线性减淡（添加）
    bmPsSubtractive,    // PS 减去
    bmPsMultiplicative, // PS 正片叠底
    bmPsScreen,         // PS 滤色
    bmPsOverlay,        // PS 叠加
    bmPsHardLight,      // PS 强光
    bmPsSoftLight,      // PS 柔光
    bmPsColorDodge,     // PS 颜色减淡
    bmPsColorDodge5,    // PS 颜色减淡（版本5）
    bmPsColorBurn,      // PS 颜色加深
    bmPsLighten,        // PS 变亮
    bmPsDarken,         // PS 变暗
    bmPsDifference,     // PS 差值
    bmPsDifference5,    // PS 差值（版本5）
    bmPsExclusion       // PS 排除
};
```

### 4.4 缩放插值模式

```cpp
// 文件: cpp/core/visual/LayerBitmapIntf.h（第 63-92 行）

enum tTVPBBStretchType {
    stNearest = 0,       // 最近邻插值（像素风格）
    stFastLinear = 1,    // 快速双线性
    stLinear = 2,        // 严格双线性
    stCubic = 3,         // 双三次插值
    stSemiFastLinear = 4,
    stFastCubic = 5,
    stLanczos2 = 6,      // Lanczos 2 插值
    stFastLanczos2 = 7,
    stLanczos3 = 8,      // Lanczos 3 插值
    stFastLanczos3 = 9,
    stSpline16 = 10,     // Spline16 插值
    stFastSpline16 = 11,
    stSpline36 = 12,     // Spline36 插值
    stFastSpline36 = 13,
    stAreaAvg = 14,      // 面积平均（适合缩小）
    stFastAreaAvg = 15,
    stGaussian = 16,     // 高斯插值
    stFastGaussian = 17,
    stBlackmanSinc = 18, // Blackman-Sinc 插值
    stFastBlackmanSinc = 19,
};
```

---

## 5. 动手实践

### 实践 1：创建图层并设置图像

```cpp
// 示例：从 C++ 代码创建图层并绘制内容

#include "LayerIntf.h"
#include "LayerBitmapIntf.h"

void CreateAndDrawLayer(tTVPLayerManager* manager) {
    // 1. 获取或创建图层对象
    // 注意：实际中通过 TJS 脚本创建，这里仅演示概念
    
    // 假设 layer 已创建并设置了父图层
    tTJSNI_BaseLayer* layer = /* 获取图层指针 */;
    
    // 2. 确保图层有图像
    layer->SetHasImage(true);  // 分配 MainImage
    
    // 3. 设置图层尺寸
    layer->SetSize(320, 240);
    
    // 4. 获取主图像进行绘制
    tTVPBaseTexture* image = layer->GetMainImage();
    
    // 5. 填充背景色（半透明蓝色）
    tTVPRect fullRect(0, 0, 320, 240);
    // ARGB: 0x800000FF = 半透明蓝色
    image->Fill(fullRect, 0x800000FF);
    
    // 6. 绘制一个红色矩形
    tTVPRect redRect(50, 50, 200, 150);
    image->FillColor(redRect, 0xFF0000, 255);  // 不透明红色
    
    // 7. 标记图像已修改（触发重绘）
    layer->Update();
}
```

### 实践 2：像素级操作

```cpp
// 示例：逐像素处理图像

void InvertColors(tTJSNI_BaseLayer* layer) {
    tTVPBaseTexture* image = layer->GetMainImage();
    if(!image) return;
    
    tjs_uint width = image->GetWidth();
    tjs_uint height = image->GetHeight();
    
    // 逐像素反转颜色
    for(tjs_uint y = 0; y < height; y++) {
        for(tjs_uint x = 0; x < width; x++) {
            // 读取原像素 (ARGB)
            tjs_uint32 pixel = image->GetPoint(x, y);
            
            // 提取分量
            tjs_uint8 a = (pixel >> 24) & 0xFF;
            tjs_uint8 r = (pixel >> 16) & 0xFF;
            tjs_uint8 g = (pixel >> 8) & 0xFF;
            tjs_uint8 b = pixel & 0xFF;
            
            // 反转 RGB（保持 Alpha）
            tjs_uint32 inverted = (a << 24) | 
                                  ((255 - r) << 16) | 
                                  ((255 - g) << 8) | 
                                  (255 - b);
            
            // 写回
            image->SetPoint(x, y, inverted);
        }
    }
    
    // 触发更新
    layer->Update();
}
```

### 实践 3：坐标转换

```cpp
// 示例：处理鼠标点击事件中的坐标转换

void HandleMouseClick(tTJSNI_BaseLayer* layer, 
                      tjs_int primaryX, tjs_int primaryY) {
    // 1. 从 Primary（窗口）坐标转换为 Layer 坐标
    tjs_int layerX = primaryX;
    tjs_int layerY = primaryY;
    layer->FromPrimaryCoordinates(layerX, layerY);
    
    // 2. 检查是否在图层边界内
    if(layerX < 0 || layerY < 0 ||
       layerX >= (tjs_int)layer->GetWidth() ||
       layerY >= (tjs_int)layer->GetHeight()) {
        // 点击在图层外部
        return;
    }
    
    // 3. 转换为 Image 坐标
    tjs_int imageX = layerX - layer->GetImageLeft();
    tjs_int imageY = layerY - layer->GetImageTop();
    
    // 4. 检查是否在图像边界内
    tTVPBaseTexture* image = layer->GetMainImage();
    if(!image) return;
    
    if(imageX < 0 || imageY < 0 ||
       imageX >= (tjs_int)image->GetWidth() ||
       imageY >= (tjs_int)image->GetHeight()) {
        // 点击在图像外部（但在图层内）
        return;
    }
    
    // 5. 获取点击位置的像素
    tjs_uint32 pixel = image->GetPoint(imageX, imageY);
    tjs_uint8 alpha = (pixel >> 24) & 0xFF;
    
    // 6. 判断是否点击到了有效内容（Alpha > 阈值）
    if(alpha > layer->GetHitThreshold()) {
        // 有效点击！
        printf("Clicked at image (%d, %d), color = 0x%08X\n",
               imageX, imageY, pixel);
    }
}
```

### 实践 4：位图复制与混合

```cpp
// 示例：将一个图层的内容混合到另一个图层

void BlendLayers(tTJSNI_BaseLayer* destLayer,
                 tTJSNI_BaseLayer* srcLayer,
                 tjs_int destX, tjs_int destY,
                 tjs_int opacity) {
    tTVPBaseTexture* destImage = destLayer->GetMainImage();
    tTVPBaseTexture* srcImage = srcLayer->GetMainImage();
    
    if(!destImage || !srcImage) return;
    
    // 定义源矩形（整个源图像）
    tTVPRect srcRect(0, 0, 
                     srcImage->GetWidth(), 
                     srcImage->GetHeight());
    
    // 执行 Alpha 混合
    destImage->Blt(
        destX, destY,           // 目标位置
        srcImage,               // 源位图
        srcRect,                // 源矩形
        bmAlpha,                // 混合方法：Alpha 混合
        opacity,                // 不透明度（0-255）
        true                    // hda: 保持目标 Alpha
    );
    
    // 触发更新
    destLayer->Update();
}
```

### 实践 5：缩放绘制

```cpp
// 示例：将图像缩放绘制到目标位置

void StretchDrawImage(tTJSNI_BaseLayer* layer,
                      iTVPBaseBitmap* srcBitmap,
                      tTVPRect destRect) {
    tTVPBaseTexture* image = layer->GetMainImage();
    if(!image) return;
    
    // 源矩形（整个源图像）
    tTVPRect srcRect(0, 0, 
                     srcBitmap->GetWidth(), 
                     srcBitmap->GetHeight());
    
    // 裁剪矩形（目标图像边界）
    tTVPRect clipRect(0, 0, 
                      image->GetWidth(), 
                      image->GetHeight());
    
    // 执行缩放绘制
    image->StretchBlt(
        clipRect,       // 裁剪区域
        destRect,       // 目标矩形
        srcBitmap,      // 源位图
        srcRect,        // 源矩形
        bmAlpha,        // 混合方法
        255,            // 不透明度
        true,           // hda
        stLinear        // 插值方法：双线性
    );
    
    layer->Update();
}
```

---

## 6. 对照项目源码

### 相关文件

| 文件 | 行数 | 关键内容 |
|------|------|----------|
| `cpp/core/visual/LayerIntf.h` | 1205 | `tTJSNI_BaseLayer` 类定义 |
| `cpp/core/visual/LayerIntf.cpp` | 11922 | 图层核心实现 |
| `cpp/core/visual/LayerBitmapIntf.h` | 275 | `iTVPBaseBitmap` 接口定义 |
| `cpp/core/visual/LayerBitmapIntf.cpp` | 4820 | 位图操作实现 |

### 关键代码位置

1. **图层构造函数**：`LayerIntf.cpp` 第 304-410 行
   - 初始化所有成员变量
   - 分配默认图像

2. **图像分配**：`LayerIntf.cpp` 第 1500-1600 行
   - `AllocateImage()` / `DeallocateImage()`
   - `AllocateProvinceImage()` / `DeallocateProvinceImage()`

3. **坐标转换**：`LayerIntf.cpp` 搜索 `ToPrimaryCoordinates`
   - 递归累加父图层偏移

4. **像素操作**：`LayerBitmapIntf.cpp` 第 165-226 行
   - `GetPoint()` / `SetPoint()`
   - `SetPointMain()` / `SetPointMask()`

5. **填充操作**：`LayerBitmapIntf.cpp` 第 228-303 行
   - `Fill()` 的 GPU 加速实现

---

## 7. 本节小结

- **tTJSNI_BaseLayer** 是图层系统的核心基类，管理树状结构、图像缓冲、渲染参与
- **双图像系统**：MainImage 存储可见像素，ProvinceImage 存储区域标识
- **三种坐标系**：Primary（窗口）、Layer（图层）、Image（图像），需正确转换
- **iTVPBaseBitmap** 提供丰富的像素操作 API，支持多种混合模式和插值算法
- **图像生命周期**：由 `AllocateImage()` / `DeallocateImage()` 管理，随图层创建/销毁

---

## 8. 练习题与答案

### 题目 1：图层坐标转换

假设有如下图层结构：
- Primary 图层位于窗口 (0, 0)
- Layer A 位于 Primary 的 (100, 50)
- Layer B 位于 Layer A 的 (30, 20)
- Layer B 的 ImageLeft=10, ImageTop=5

问：在 Layer B 的图像坐标 (15, 25) 处绘制的像素，对应窗口的哪个位置？

<details>
<summary>查看答案</summary>

**计算过程**：

1. **Image → Layer 坐标**
   - Layer 坐标 X = ImageX + ImageLeft = 15 + 10 = 25
   - Layer 坐标 Y = ImageY + ImageTop = 25 + 5 = 30

2. **Layer B → Layer A 坐标**
   - Layer A 坐标 X = 25 + 30 = 55（加上 Layer B 在 A 中的位置）
   - Layer A 坐标 Y = 30 + 20 = 50

3. **Layer A → Primary 坐标**
   - Primary 坐标 X = 55 + 100 = 155
   - Primary 坐标 Y = 50 + 50 = 100

**答案**：窗口坐标 **(155, 100)**

</details>

### 题目 2：像素颜色操作

编写一个函数，将图层中所有像素的红色分量减半，同时保持绿色、蓝色和透明度不变。

<details>
<summary>查看答案</summary>

```cpp
void HalveRedChannel(tTJSNI_BaseLayer* layer) {
    tTVPBaseTexture* image = layer->GetMainImage();
    if(!image || !image->Is32BPP()) return;
    
    tjs_uint width = image->GetWidth();
    tjs_uint height = image->GetHeight();
    
    for(tjs_uint y = 0; y < height; y++) {
        for(tjs_uint x = 0; x < width; x++) {
            // 读取像素 (ARGB)
            tjs_uint32 pixel = image->GetPoint(x, y);
            
            // 提取分量
            tjs_uint8 a = (pixel >> 24) & 0xFF;
            tjs_uint8 r = (pixel >> 16) & 0xFF;
            tjs_uint8 g = (pixel >> 8) & 0xFF;
            tjs_uint8 b = pixel & 0xFF;
            
            // 红色分量减半
            r = r / 2;
            
            // 重组像素
            tjs_uint32 newPixel = (a << 24) | (r << 16) | (g << 8) | b;
            
            // 写回
            image->SetPoint(x, y, newPixel);
        }
    }
    
    // 标记修改并触发更新
    layer->SetImageModified(true);
    layer->Update();
}
```

**要点**：
- 使用位运算提取和重组 ARGB 分量
- 操作完成后调用 `Update()` 触发重绘

</details>

### 题目 3：混合模式选择

你需要实现以下效果，应该选择哪种 `tTVPBBBltMethod`？

1. 将 UI 按钮图像叠加到背景上，按钮有透明边缘
2. 实现图像发光效果（颜色叠加变亮）
3. 将两张照片合成，使重叠部分变暗

<details>
<summary>查看答案</summary>

1. **UI 按钮叠加** → `bmAlpha`
   - Alpha 混合是最常用的透明叠加方式
   - 按钮的透明边缘会自然融合到背景中

2. **发光效果** → `bmAdd` 或 `bmPsScreen`
   - `bmAdd`（加法混合）：直接相加，适合明亮的发光
   - `bmPsScreen`（滤色）：更柔和的发光效果，不会过曝

3. **照片重叠变暗** → `bmDarken` 或 `bmPsDarken`
   - `bmDarken`：取两个像素中较暗的那个
   - `bmPsMultiplicative`（正片叠底）：更真实的变暗混合

**代码示例**：

```cpp
// 1. UI 按钮叠加
image->Blt(x, y, buttonImage, srcRect, bmAlpha, 255);

// 2. 发光效果
image->Blt(x, y, glowImage, srcRect, bmAdd, 128);  // 半透明发光

// 3. 照片变暗
image->Blt(x, y, photo2, srcRect, bmDarken, 255);
```

</details>

---

## 9. 下一步

下一节 [02-LayerManager与图层树](./02-LayerManager与图层树.md) 将深入讲解：

- `tTVPLayerManager` 如何管理整个图层树
- 图层的 Z 序管理（OrderIndex、AbsoluteOrderMode）
- 焦点管理与事件分发
- 图层更新机制与脏区追踪
