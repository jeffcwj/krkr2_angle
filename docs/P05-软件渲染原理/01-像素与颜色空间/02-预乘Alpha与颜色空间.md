# 预乘 Alpha 与颜色空间

> **所属模块：** P05-软件渲染原理
> **前置知识：** [01-像素本质与格式](./01-像素本质与格式.md)
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：

1. 解释预乘 Alpha（Premultiplied Alpha）与标准 Alpha（Straight Alpha）的区别
2. 理解预乘 Alpha 在混合运算中的性能优势
3. 手写标准 Alpha 与预乘 Alpha 的转换代码
4. 区分 sRGB 与线性颜色空间，理解 Gamma 校正的原理
5. 读懂 KrKr2 中 `TVPDivTable`、`TVPOpacityOnOpacityTable` 等查找表的设计

---

## 1. 标准 Alpha 与预乘 Alpha

### 1.1 两种存储约定

**标准 Alpha（Straight Alpha，也叫 Non-premultiplied Alpha）**直接存储原始颜色值，Alpha 独立表示不透明度：

```
像素 = (A, R, G, B) = (128, 255, 0, 0)
含义：50% 透明的纯红色
存储的 R=255 就是"这个像素本来的红色亮度是 255"
```

**预乘 Alpha（Premultiplied Alpha）**则在存储时已经将颜色通道乘以了 Alpha：

```
像素 = (A, R×A/255, G×A/255, B×A/255) = (128, 128, 0, 0)
含义：颜色值已经包含了透明度信息
存储的 R=128 是"红色 255 乘以透明度 128/255 后的结果"
```

### 1.2 为什么预乘 Alpha 更高效？

预乘 Alpha 的核心优势在于简化混合运算。来看最常用的 **Porter-Duff Over 合成**（将前景叠加到背景上，下一节详讲）：

**标准 Alpha 的混合公式：**

```
Out.R = Src.R × Src.A / 255 + Dst.R × (255 - Src.A) / 255
Out.G = Src.G × Src.A / 255 + Dst.G × (255 - Src.A) / 255
Out.B = Src.B × Src.A / 255 + Dst.B × (255 - Src.A) / 255
```

每个通道需要 **2 次乘法 + 1 次加法 + 2 次除法**（共 6 个运算/通道）。

**预乘 Alpha 的混合公式：**

```
Out.R = Src.R_premul + Dst.R × (255 - Src.A) / 255
Out.G = Src.G_premul + Dst.G × (255 - Src.A) / 255
Out.B = Src.B_premul + Dst.B × (255 - Src.A) / 255
```

因为 `Src.R_premul` 已经等于 `Src.R × Src.A / 255`，所以每个通道只需 **1 次乘法 + 1 次加法 + 1 次除法**（共 3 个运算/通道）。运算量几乎减半。

对于每帧需要混合数百万像素的游戏引擎来说，这个差异非常显著。

### 1.3 预乘 Alpha 的另一个优势：滤波正确性

当对半透明图像做缩放、旋转等插值（Interpolation，通过相邻像素的加权平均来计算新位置的颜色，第 3 章详讲）时，标准 Alpha 会产生**黑边伪影（Black Fringe）**。

原因：标准 Alpha 的像素 `(A=0, R=255, G=0, B=0)` 表示"完全透明，但记录了红色信息"。当这个像素和相邻像素做线性插值时，R=255 会"泄漏"到结果中，产生不应出现的颜色。

预乘 Alpha 中同样的像素是 `(A=0, R=0, G=0, B=0)` —— 完全透明的像素颜色值为零，插值时不会泄漏任何颜色。

---

## 2. 转换代码

### 2.1 标准 Alpha → 预乘 Alpha

```cpp
#include <cstdint>

// 标准 Alpha → 预乘 Alpha
inline uint32_t toPremultiplied(uint32_t pixel) {
    uint32_t a = (pixel >> 24) & 0xFF;  // 提取 Alpha
    if (a == 255) return pixel;         // 完全不透明，无需处理
    if (a == 0)   return 0;             // 完全透明，所有通道归零

    // 各颜色通道乘以 Alpha，再除以 255
    uint32_t r = ((pixel >> 16) & 0xFF) * a / 255;
    uint32_t g = ((pixel >> 8)  & 0xFF) * a / 255;
    uint32_t b = ( pixel        & 0xFF) * a / 255;

    return (a << 24) | (r << 16) | (g << 8) | b;
}
```

KrKr2 的实际实现使用了位运算加速（`blend_functor_c.h` 中的 `alpha_to_premulalpha_func`）：

```cpp
// 文件: cpp/core/visual/gl/blend_functor_c.h
tjs_uint32 straight_to_premul(tjs_uint32 pixel) {
    tjs_uint32 a = pixel >> 24;          // 提取 Alpha
    tjs_uint32 rb = pixel & 0x00ff00ff;  // 同时提取 R 和 B
    tjs_uint32 g  = pixel & 0x0000ff00;  // 提取 G
    rb = (rb * a >> 8) & 0x00ff00ff;     // R、B 同时乘以 Alpha（双通道并行）
    g  = (g  * a >> 8) & 0x0000ff00;     // G 乘以 Alpha
    return rb | g | (a << 24);           // 重组像素
}
```

> **注意**：这里用 `>> 8`（除以 256）近似 `/ 255`（除以 255），有约 0.4% 的误差。KrKr2 为了性能统一使用 `>> 8`，这是游戏引擎中的常见做法。精确除以 255 的技巧是 `(x * a + 128) / 255`。

### 2.2 预乘 Alpha → 标准 Alpha（反转）

反转操作需要将颜色通道除以 Alpha，这涉及整数除法——比乘法慢 20~40 倍。KrKr2 使用查找表加速（见第 4 节）。

```cpp
// 预乘 Alpha → 标准 Alpha
inline uint32_t toStraight(uint32_t pixel) {
    uint32_t a = (pixel >> 24) & 0xFF;
    if (a == 0)   return 0;       // 完全透明
    if (a == 255) return pixel;   // 完全不透明

    // KrKr2 使用 TVPDivTable 查找表加速除法（见第 4 节）
    // 这里用标准除法演示
    uint32_t r = ((pixel >> 16) & 0xFF) * 255 / a;
    uint32_t g = ((pixel >> 8)  & 0xFF) * 255 / a;
    uint32_t b = ( pixel        & 0xFF) * 255 / a;

    // 防止溢出（预乘后再反转可能略超 255）
    if (r > 255) r = 255;
    if (g > 255) g = 255;
    if (b > 255) b = 255;

    return (a << 24) | (r << 16) | (g << 8) | b;
}
```

### 2.3 完整转换示例

```cpp
#include <cstdint>
#include <cstdio>

uint32_t toPremultiplied(uint32_t pixel) {
    uint32_t a = (pixel >> 24) & 0xFF;
    if (a == 255) return pixel;
    if (a == 0) return 0;
    uint32_t r = ((pixel >> 16) & 0xFF) * a / 255;
    uint32_t g = ((pixel >> 8)  & 0xFF) * a / 255;
    uint32_t b = ( pixel        & 0xFF) * a / 255;
    return (a << 24) | (r << 16) | (g << 8) | b;
}

uint32_t toStraight(uint32_t pixel) {
    uint32_t a = (pixel >> 24) & 0xFF;
    if (a == 0) return 0;
    if (a == 255) return pixel;
    uint32_t r = std::min(255u, ((pixel >> 16) & 0xFF) * 255u / a);
    uint32_t g = std::min(255u, ((pixel >> 8)  & 0xFF) * 255u / a);
    uint32_t b = std::min(255u, ( pixel        & 0xFF) * 255u / a);
    return (a << 24) | (r << 16) | (g << 8) | b;
}

void printPixel(const char* label, uint32_t p) {
    printf("%s: A=%3u R=%3u G=%3u B=%3u (0x%08X)\n", label,
           (p >> 24) & 0xFF, (p >> 16) & 0xFF,
           (p >> 8)  & 0xFF,  p        & 0xFF, p);
}

int main() {
    uint32_t straight = 0x80FF8040;  // 半透明的 R=255 G=128 B=64
    printPixel("Straight   ", straight);

    uint32_t premul = toPremultiplied(straight);
    printPixel("Premultiplied", premul);

    uint32_t back = toStraight(premul);
    printPixel("Back to Str", back);

    // 验证往返转换（会有微小精度损失）
    printf("往返误差: R=%d G=%d B=%d\n",
           (int)((straight >> 16) & 0xFF) - (int)((back >> 16) & 0xFF),
           (int)((straight >> 8) & 0xFF)  - (int)((back >> 8) & 0xFF),
           (int)(straight & 0xFF)         - (int)(back & 0xFF));
    return 0;
}
```

预期输出：
```
Straight   : A=128 R=255 G=128 B= 64 (0x80FF8040)
Premultiplied: A=128 R=128 G= 64 B= 32 (0x80804020)
Back to Str: A=128 R=255 G=127 B= 63 (0x80FF7F3F)
往返误差: R=0 G=1 B=1
```

注意往返转换会有 ±1 的精度损失，这是整数除法截断导致的。

---

## 3. KrKr2 的双 Alpha 体系

KrKr2 同时支持标准 Alpha 和预乘 Alpha 两种图层模式。函数命名通过后缀区分：

```
TVPAlphaBlend_d    — d = dest has alpha（目标有标准 Alpha 通道）
TVPAlphaBlend_a    — a = dest has additive-alpha（目标有预乘 Alpha）
TVPAlphaBlend_o    — o = with opacity（附加全局不透明度参数）
TVPAlphaBlend_HDA  — HDA = Hold Dest Alpha（保持目标 Alpha 不变）
```

来源：`krkr2/cpp/core/visual/tvpgl.h` 第 17-21 行的注释：

```cpp
// key to blending suffix:
// d : destination has alpha
// a : destination has additive-alpha (premultiplied)
// o : blend with opacity
```

**混合模式对应关系**（`LayerBitmapIntf.h` 中定义）：

| 枚举值 | 含义 | Alpha 类型 |
|--------|------|-----------|
| `bmAlpha` | 标准 Alpha 混合 | Straight → Straight |
| `bmAddAlpha` | 预乘 Alpha 混合 | Premultiplied → Premultiplied |
| `bmAddAlphaOnAlpha` | 预乘 → 标准 | Premultiplied → Straight |
| `bmAlphaOnAddAlpha` | 标准 → 预乘 | Straight → Premultiplied |

引擎根据源图层和目标图层各自的 Alpha 类型，自动选择正确的混合函数。

---

## 4. 查找表（LUT）优化

### 4.1 为什么需要查找表？

`c * 255 / a` 这样的整数除法在 CPU 上非常昂贵（比乘法慢 20~40 倍）。KrKr2 使用预计算的查找表（Look-Up Table，简称 LUT）将除法转化为数组访问，实现"空间换时间"。

### 4.2 除法表 TVPDivTable

```cpp
// 来源：krkr2/cpp/core/visual/tvpgl.cpp 第 29-45 行

// TVPDivTable[(a << 8) + b] = b * 255 / a（256KB 查找表）
unsigned char TVPDivTable[256 * 256];

// 初始化（TVPCreateTable 函数中）:
for (int b = 0; b < 256; b++) {
    TVPDivTable[(0 << 8) + b] = 0;  // 除以 0 特殊处理为 0
    for (int a = 1; a < 256; a++) {
        int tmp = b * 255 / a;
        if (tmp > 255) tmp = 255;    // 饱和到 255
        TVPDivTable[(a << 8) + b] = (uint8_t)tmp;
    }
}
```

使用方式：

```cpp
// 预乘 Alpha 反转时，不再调用昂贵的除法
uint8_t a = pixel >> 24;
uint8_t r_premul = (pixel >> 16) & 0xFF;
uint8_t r_straight = TVPDivTable[(a << 8) + r_premul];
// 等价于 r_premul * 255 / a，但只是一次数组查表
```

### 4.3 不透明度合成表

两层半透明叠加后的总不透明度计算：`result = 1 - (1-a1)(1-a2)`

```cpp
// TVPOpacityOnOpacityTable[a1 * 256 + a2] = 合成后的不透明度
unsigned char TVPOpacityOnOpacityTable[256 * 256];

// 初始化:
for (int a = 0; a < 256; a++) {
    for (int b = 0; b < 256; b++) {
        int addr = b * 256 + a;
        if (a) {
            float at = a / 255.0f, bt = b / 255.0f;
            float c = bt / at;
            c /= (1.0f - bt + c);
            TVPOpacityOnOpacityTable[addr] = (unsigned char)(c * 255);
        } else {
            TVPOpacityOnOpacityTable[addr] = 255;
        }
    }
}
```

### 4.4 Screen 混合表与倒数表

```cpp
// Screen 混合的单通道结果：255 - (255-a)(255-b)/255
unsigned char TVPNegativeMulTable[256 * 256];

// 倒数表：用乘法+移位替代除法
// TVPRecipTable256[i] = 65536 / i
tjs_uint32 TVPRecipTable256[256];

// 使用时：x / i ≈ x * TVPRecipTable256[i] >> 16
```

### 4.5 查找表内存分析

| 查找表 | 大小 | 用途 |
|--------|------|------|
| `TVPDivTable` | 256×256×1 = 64 KB | 预乘反转的除法 |
| `TVPOpacityOnOpacityTable` | 256×256×1 = 64 KB | 半透明合成 |
| `TVPNegativeMulTable` | 256×256×1 = 64 KB | Screen 混合 |
| `TVPRecipTable256` | 256×4 = 1 KB | 通用倒数 |
| **合计** | **~193 KB** | |

这些查找表在引擎启动时由 `TVPCreateTable()` 一次性初始化，之后全程只读。193 KB 在现代系统中微不足道，但要注意 L1 缓存（一级缓存，CPU 最快的缓存，通常 32-64 KB）一次只能容纳一张 64 KB 表。频繁在多张表之间切换会导致缓存未命中（Cache Miss），所以 KrKr2 的混合函数通常一次批量处理一整行像素，减少表切换。

---

## 5. 颜色空间：sRGB 与线性

### 5.1 人眼的非线性感知

人眼对暗部的亮度变化比亮部更敏感。如果用线性编码（数值翻倍 = 亮度翻倍），0~50 之间的微小变化人眼可以分辨，但 200~255 之间几乎看不出差别。这意味着线性编码在暗部"浪费了精度"。

**sRGB 编码**通过一条 **Gamma 曲线**（Gamma，希腊字母 γ，此处指一种非线性映射函数，约为 2.2 次幂）将更多编码空间分配给暗部，使得每一级亮度变化在人眼看来都是均匀的。

```
sRGB → 线性空间（去 Gamma）：
  if (sRGB <= 0.04045)
      线性 = sRGB / 12.92
  else
      线性 = ((sRGB + 0.055) / 1.055) ^ 2.4

线性空间 → sRGB（加 Gamma）：
  if (线性 <= 0.0031308)
      sRGB = 线性 * 12.92
  else
      sRGB = 1.055 * 线性^(1/2.4) - 0.055
```

### 5.2 Gamma 校正对混合的影响

在 sRGB 空间直接做 Alpha 混合会导致**中间色调偏暗**。因为 sRGB 是非线性编码，在非线性空间里做线性插值，结果不等于真实亮度的线性插值。正确流程是：

```
1. sRGB → 线性空间（去 Gamma）
2. 在线性空间做混合运算
3. 线性空间 → sRGB（加 Gamma）
```

但这样每次混合需要两次 pow() 运算，开销很大。实际工程中（包括 KrKr2），**通常直接在 sRGB 空间混合**，接受轻微的颜色偏差以换取性能。

### 5.3 完整 Gamma 转换实现

```cpp
#include <cmath>
#include <cstdint>
#include <cstdio>
#include <algorithm>

// sRGB 通道值（0~255）→ 线性浮点值（0.0~1.0）
inline float srgbToLinear(uint8_t srgb) {
    float s = srgb / 255.0f;
    if (s <= 0.04045f)
        return s / 12.92f;
    else
        return std::pow((s + 0.055f) / 1.055f, 2.4f);
}

// 线性浮点值（0.0~1.0）→ sRGB 通道值（0~255）
inline uint8_t linearToSrgb(float linear) {
    float s;
    if (linear <= 0.0031308f)
        s = linear * 12.92f;
    else
        s = 1.055f * std::pow(linear, 1.0f / 2.4f) - 0.055f;
    int result = static_cast<int>(s * 255.0f + 0.5f);
    return static_cast<uint8_t>(std::clamp(result, 0, 255));
}

// 在线性空间做正确的 Alpha 混合
uint32_t blendLinear(uint32_t src, uint32_t dst) {
    float sa = ((src >> 24) & 0xFF) / 255.0f;  // 源 Alpha

    // 提取 RGB 并转换到线性空间
    float sr = srgbToLinear((src >> 16) & 0xFF);
    float sg = srgbToLinear((src >> 8)  & 0xFF);
    float sb = srgbToLinear( src        & 0xFF);
    float dr = srgbToLinear((dst >> 16) & 0xFF);
    float dg = srgbToLinear((dst >> 8)  & 0xFF);
    float db = srgbToLinear( dst        & 0xFF);

    // 线性空间混合（SrcOver）
    float inv_sa = 1.0f - sa;
    float or_ = sr * sa + dr * inv_sa;
    float og  = sg * sa + dg * inv_sa;
    float ob  = sb * sa + db * inv_sa;

    // 转回 sRGB
    return (0xFF << 24) |
           (linearToSrgb(or_) << 16) |
           (linearToSrgb(og) << 8) |
            linearToSrgb(ob);
}

// 对比：直接在 sRGB 空间混合（KrKr2 的做法）
uint32_t blendSrgb(uint32_t src, uint32_t dst) {
    uint32_t a = (src >> 24) & 0xFF;
    auto lerp = [&](int shift) -> uint8_t {
        uint32_t s = (src >> shift) & 0xFF;
        uint32_t d = (dst >> shift) & 0xFF;
        return static_cast<uint8_t>(d + (s - d) * a / 255);
    };
    return (0xFF << 24) | (lerp(16) << 16) | (lerp(8) << 8) | lerp(0);
}

int main() {
    uint32_t red   = 0x80FF0000;  // 半透明红
    uint32_t green = 0xFF00FF00;  // 不透明绿

    uint32_t resultLinear = blendLinear(red, green);
    uint32_t resultSrgb   = blendSrgb(red, green);

    printf("线性空间混合: R=%u G=%u B=%u\n",
           (resultLinear >> 16) & 0xFF,
           (resultLinear >> 8) & 0xFF,
           resultLinear & 0xFF);
    printf("sRGB直接混合: R=%u G=%u B=%u\n",
           (resultSrgb >> 16) & 0xFF,
           (resultSrgb >> 8) & 0xFF,
           resultSrgb & 0xFF);
    printf("差异（线性更准确）: ΔR=%d ΔG=%d\n",
           (int)((resultLinear >> 16) & 0xFF) - (int)((resultSrgb >> 16) & 0xFF),
           (int)((resultLinear >> 8) & 0xFF) - (int)((resultSrgb >> 8) & 0xFF));
    return 0;
}
```

### 5.4 KrKr2 的 Gamma 校正系统

KrKr2 提供了可配置的 Gamma 校正参数，允许每个颜色通道独立调节（来源：`tvpgl.h` 第 31-44 行）：

```cpp
typedef struct {
    float RGamma;     // R 通道 Gamma 值（0.10 ~ 9.99，默认 1.0）
    tjs_int RFloor;   // R 输出最小值（0 ~ 255）
    tjs_int RCeil;    // R 输出最大值（0 ~ 255）
    float GGamma;     // G 通道 Gamma 值
    tjs_int GFloor;
    tjs_int GCeil;
    float BGamma;     // B 通道 Gamma 值
    tjs_int BFloor;
    tjs_int BCeil;
} tTVPGLGammaAdjustData;

// 默认值：不做任何 Gamma 校正（gamma=1.0，输出范围 0~255）
// 来源：LayerBitmapIntf.cpp 第 43 行
tTVPGLGammaAdjustData TVPIntactGammaAdjustData = {
    1.0, 0, 255,   // R: gamma=1.0, floor=0, ceil=255
    1.0, 0, 255,   // G
    1.0, 0, 255    // B
};
```

---

## 动手实践

编写一个程序，对比标准 Alpha 和预乘 Alpha 在缩放场景下的边缘质量差异：

```cpp
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <vector>
#include <fstream>
#include <algorithm>

using Pixel = uint32_t;

// --- 通道操作 ---
inline uint8_t getA(Pixel p) { return (p >> 24) & 0xFF; }
inline uint8_t getR(Pixel p) { return (p >> 16) & 0xFF; }
inline uint8_t getG(Pixel p) { return (p >> 8)  & 0xFF; }
inline uint8_t getB(Pixel p) { return  p        & 0xFF; }
inline Pixel make(uint8_t a, uint8_t r, uint8_t g, uint8_t b) {
    return (uint32_t(a) << 24) | (uint32_t(r) << 16) |
           (uint32_t(g) << 8) | b;
}

// --- 简易 2x2 缩小（平均4个像素）---
// 在标准 Alpha 空间做缩小
Pixel downsample2x2_straight(Pixel tl, Pixel tr, Pixel bl, Pixel br) {
    // 直接对 4 个像素取平均——错误做法！
    uint32_t a = (getA(tl) + getA(tr) + getA(bl) + getA(br)) / 4;
    uint32_t r = (getR(tl) + getR(tr) + getR(bl) + getR(br)) / 4;
    uint32_t g = (getG(tl) + getG(tr) + getG(bl) + getG(br)) / 4;
    uint32_t b = (getB(tl) + getB(tr) + getB(bl) + getB(br)) / 4;
    return make(a, r, g, b);
}

// 在预乘 Alpha 空间做缩小——正确做法
Pixel toPremul(Pixel p) {
    uint32_t a = getA(p);
    if (a == 255) return p;
    if (a == 0) return 0;
    return make(a, getR(p)*a/255, getG(p)*a/255, getB(p)*a/255);
}

Pixel toStraight(Pixel p) {
    uint32_t a = getA(p);
    if (a == 0) return 0;
    if (a == 255) return p;
    return make(a,
        std::min(255u, (uint32_t)getR(p)*255/a),
        std::min(255u, (uint32_t)getG(p)*255/a),
        std::min(255u, (uint32_t)getB(p)*255/a));
}

Pixel downsample2x2_premul(Pixel tl, Pixel tr, Pixel bl, Pixel br) {
    // 先转预乘，取平均，再转回
    Pixel ptl = toPremul(tl), ptr = toPremul(tr);
    Pixel pbl = toPremul(bl), pbr = toPremul(br);
    uint32_t a = (getA(ptl) + getA(ptr) + getA(pbl) + getA(pbr)) / 4;
    uint32_t r = (getR(ptl) + getR(ptr) + getR(pbl) + getR(pbr)) / 4;
    uint32_t g = (getG(ptl) + getG(ptr) + getG(pbl) + getG(pbr)) / 4;
    uint32_t b = (getB(ptl) + getB(ptr) + getB(pbl) + getB(pbr)) / 4;
    Pixel premulResult = make(a, r, g, b);
    return toStraight(premulResult);
}

int main() {
    // 模拟红色圆的边缘：3 个不透明红 + 1 个完全透明
    Pixel red_opaque = 0xFFFF0000;    // 不透明红
    Pixel transparent = 0x00000000;   // 完全透明（标准 Alpha 中 RGB 为 0）

    // 但如果透明像素记录了"黑色"（某些图片编辑器会这样）
    Pixel transparent_black = 0x00000000;
    // 或者记录了"绿色"（这在标准 Alpha 中是合法的！）
    Pixel transparent_green = 0x0000FF00;

    printf("=== 透明像素为纯黑 (0x00000000) ===\n");
    Pixel s1 = downsample2x2_straight(red_opaque, red_opaque,
                                       red_opaque, transparent_black);
    Pixel p1 = downsample2x2_premul(red_opaque, red_opaque,
                                     red_opaque, transparent_black);
    printf("Straight: A=%u R=%u G=%u B=%u\n", getA(s1), getR(s1), getG(s1), getB(s1));
    printf("Premul:   A=%u R=%u G=%u B=%u\n", getA(p1), getR(p1), getG(p1), getB(p1));

    printf("\n=== 透明像素记录了绿色 (0x0000FF00) ===\n");
    Pixel s2 = downsample2x2_straight(red_opaque, red_opaque,
                                       red_opaque, transparent_green);
    Pixel p2 = downsample2x2_premul(red_opaque, red_opaque,
                                     red_opaque, transparent_green);
    printf("Straight: A=%u R=%u G=%u B=%u  ← 绿色泄漏！\n",
           getA(s2), getR(s2), getG(s2), getB(s2));
    printf("Premul:   A=%u R=%u G=%u B=%u  ← 正确，无泄漏\n",
           getA(p2), getR(p2), getG(p2), getB(p2));

    return 0;
}
```

---

## 常见错误及解决方案

### 错误 1：混用标准 Alpha 和预乘 Alpha

```cpp
// ❌ 错误：对预乘格式的像素再乘一次 Alpha
uint32_t premul_pixel = 0x80800000;  // 预乘格式：A=128, R=128
uint32_t a = premul_pixel >> 24;     // a = 128
// 混合时又乘了一次 a：R = 128 * 128 / 255 ≈ 64（偏暗！）

// ✅ 正确：识别像素是否已预乘，选择正确的混合函数
// KrKr2 中 bmAlpha 用于 straight，bmAddAlpha 用于 premultiplied
```

**现象**：半透明对象比预期更暗，边缘出现深色"光晕"。

### 错误 2：预乘反转时除以零

```cpp
// ❌ 错误：Alpha=0 时执行除法
uint8_t r = (pixel >> 16 & 0xFF) * 255 / (pixel >> 24 & 0xFF);

// ✅ 正确：先判断 Alpha，为 0 时直接返回
uint8_t a = (pixel >> 24) & 0xFF;
if (a == 0) return 0;  // 完全透明像素，所有通道为 0
uint8_t r = std::min(255u, ((pixel >> 16) & 0xFF) * 255u / a);
```

### 错误 3：在 sRGB 空间取平均导致偏暗

```cpp
// ❌ 错误：在 sRGB 空间直接取两个颜色的中间值
uint8_t mid = (srgb_dark + srgb_bright) / 2;
// 视觉上这个"中间值"会偏暗

// ✅ 正确（如果需要精确结果）：转线性空间再取平均
float linear_a = srgbToLinear(srgb_dark);
float linear_b = srgbToLinear(srgb_bright);
uint8_t mid = linearToSrgb((linear_a + linear_b) * 0.5f);
```

---

## 本节小结

- **标准 Alpha** 存储原始颜色，**预乘 Alpha** 存储颜色×透明度。预乘格式混合运算量减半，且滤波时不会产生颜色泄漏
- KrKr2 同时支持两种 Alpha 模式，通过函数后缀 `_a`（预乘）和 `_d`（标准 Alpha 目标）区分
- **查找表优化**：`TVPDivTable`（64KB）替代除法，`TVPOpacityOnOpacityTable`（64KB）预计算不透明度合成，`TVPRecipTable256`（1KB）提供快速倒数
- **sRGB** 是非线性编码，分配更多精度给人眼敏感的暗部。正确的混合应在线性空间进行，但 KrKr2 等引擎通常在 sRGB 直接混合以换取性能
- **Gamma 校正**：KrKr2 通过 `tTVPGLGammaAdjustData` 结构体支持每通道独立的 Gamma 调节

---

## 练习题与答案

### 题目 1：预乘 Alpha 手动计算

给定一个标准 Alpha 像素 `0xC0336699`（A=192, R=51, G=102, B=153），将它转换为预乘 Alpha 格式。然后再转回标准 Alpha，验证是否存在精度损失。

<details>
<summary>查看答案</summary>

**标准 → 预乘**：
- A' = 192（不变）
- R' = 51 × 192 / 255 = 9792 / 255 ≈ **38**
- G' = 102 × 192 / 255 = 19584 / 255 ≈ **76**
- B' = 153 × 192 / 255 = 29376 / 255 ≈ **115**

预乘后：`0xC0264C73`

**预乘 → 标准**（反转）：
- A = 192
- R = 38 × 255 / 192 = 9690 / 192 ≈ **50**（原值 51，损失 1）
- G = 76 × 255 / 192 = 19380 / 192 ≈ **100**（原值 102，损失 2）
- B = 115 × 255 / 192 = 29325 / 192 ≈ **152**（原值 153，损失 1）

**结论**：往返转换有 ±1~2 的精度损失，这是整数除法截断导致的，在实际渲染中不可见。

</details>

### 题目 2：查找表空间分析

KrKr2 使用了 `TVPDivTable[256*256]`、`TVPOpacityOnOpacityTable[256*256]`、`TVPNegativeMulTable[256*256]` 三张查找表。如果要增加 16 位精度版本（每通道 65536 级），每张表需要多少内存？这种方案是否可行？

<details>
<summary>查看答案</summary>

**当前 8 位查找表**：
- 每张表：256 × 256 × 1 字节 = **64 KB**
- 三张表合计：**192 KB**

**16 位精度版本**：
- 每张表：65536 × 65536 × 2 字节 = **8 GB**
- 完全不可行！

**更实际的方案**（65536 × 256）：
- 每张表：65536 × 256 × 2 字节 = **32 MB**
- 仍然不可行——远超 L3 缓存（一般 8-32 MB），查表会频繁引发缓存未命中

**正确做法**：16 位精度直接用定点数乘法+移位：
```cpp
// x / a ≈ x * (65536 / a) >> 16，只需 256 项的倒数表
uint16_t recipTable[256];  // 仅 512 字节
```

</details>

### 题目 3：验证 TVP_REVRGB 的自反性

`TVP_REVRGB` 将 ARGB 的 R 和 B 通道互换。请用代码证明对同一个像素值连续调用两次 `TVP_REVRGB`，结果等于原值。

<details>
<summary>查看答案</summary>

```cpp
#include <cstdint>
#include <cstdio>
#include <cassert>

#define TVP_REVRGB(v) \
    ((v & 0xFF00FF00) | ((v >> 16) & 0xFF) | ((v & 0xFF) << 16))

int main() {
    // 测试多个不同的像素值
    uint32_t testCases[] = {
        0xFFFF0000,   // 不透明红
        0x80336699,   // 半透明
        0x00000000,   // 全零
        0xFFFFFFFF,   // 全满
        0xABCDEF01,   // 随机值
    };

    for (auto pixel : testCases) {
        uint32_t once = TVP_REVRGB(pixel);
        uint32_t twice = TVP_REVRGB(once);
        printf("原值: 0x%08X → 一次: 0x%08X → 两次: 0x%08X  %s\n",
               pixel, once, twice,
               (pixel == twice) ? "✅" : "❌");
        assert(pixel == twice);  // 断言自反性
    }
    printf("所有测试通过！TVP_REVRGB 是自反操作。\n");
    return 0;
}
```

**数学证明**：

`TVP_REVRGB` 的本质是交换字节 0（B）和字节 2（R），保持字节 1（G）和字节 3（A）不变。交换操作执行两次等于不操作，所以是自反的。

设 `v = 0xAARRGGBB`：
1. 第一次：`0xAA00GG00 | 0x000000RR | 0x00BB0000` = `0xAABBGGRR`
2. 第二次：`0xAA00GG00 | 0x000000BB | 0x00RR0000` = `0xAARRGGBB` = 原值 ✅

</details>

---

## 下一步

下一节 [跨平台差异与实践](./03-跨平台差异与实践.md) 将详细讲解不同操作系统和图形 API 的像素格式差异，以及 KrKr2 如何在 Windows/Linux/macOS/Android 四个平台之间统一像素处理流程。
