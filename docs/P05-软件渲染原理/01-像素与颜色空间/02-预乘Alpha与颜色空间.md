## 3. 预乘 Alpha（Premultiplied Alpha）

### 3.1 标准 Alpha vs 预乘 Alpha

**标准 Alpha（Straight Alpha）**存储原始颜色值：

```
像素 = (A, R, G, B) = (128, 255, 0, 0)
含义：50% 透明的纯红色
```

**预乘 Alpha（Premultiplied Alpha）**将颜色通道预先乘以 Alpha：

```
像素 = (A, R*A/255, G*A/255, B*A/255) = (128, 128, 0, 0)
含义：颜色值已经包含透明度信息
```

### 3.2 为什么预乘 Alpha 更好？

**标准 Alpha 混合公式（Porter-Duff Over）：**

```
Out.R = Src.R * Src.A / 255 + Dst.R * (255 - Src.A) / 255
Out.G = Src.G * Src.A / 255 + Dst.G * (255 - Src.A) / 255
Out.B = Src.B * Src.A / 255 + Dst.B * (255 - Src.A) / 255
```

每个通道需要 **2 次乘法 + 1 次加法 + 2 次除法**。

**预乘 Alpha 混合公式：**

```
Out.R = Src.R + Dst.R * (255 - Src.A) / 255
Out.G = Src.G + Dst.G * (255 - Src.A) / 255
Out.B = Src.B + Dst.B * (255 - Src.A) / 255
```

每个通道只需 **1 次乘法 + 1 次加法 + 1 次除法**——节省了 `Src.R * Src.A / 255` 这一步（因为已经预乘了）。

### 3.3 预乘转换代码

```cpp
#include <cstdint>

// 标准 Alpha → 预乘 Alpha
inline uint32_t toPremultiplied(uint32_t pixel) {
    uint32_t a = (pixel >> 24) & 0xFF;
    if (a == 255) return pixel;  // 完全不透明，无需处理
    if (a == 0)   return 0;     // 完全透明
    
    uint32_t r = ((pixel >> 16) & 0xFF) * a / 255;
    uint32_t g = ((pixel >> 8)  & 0xFF) * a / 255;
    uint32_t b = ( pixel        & 0xFF) * a / 255;
    
    return (a << 24) | (r << 16) | (g << 8) | b;
}

// 预乘 Alpha → 标准 Alpha（需要查表避免除零）
inline uint32_t toStraight(uint32_t pixel) {
    uint32_t a = (pixel >> 24) & 0xFF;
    if (a == 0)   return 0;
    if (a == 255) return pixel;
    
    // KrKr2 使用 TVPDivTable 查找表加速这个除法
    // TVPDivTable[a * 256 + c] ≈ c * 255 / a
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

### 3.4 KrKr2 的两套 Alpha 体系

KrKr2 同时支持两种 Alpha 模式，函数命名体现了这一设计：

```
TVPAlphaBlend_d    — d = dest has alpha（目标有标准 Alpha）
TVPAlphaBlend_a    — a = dest has additive-alpha（目标有预乘 Alpha）
TVPAlphaBlend_o    — o = with opacity（额外的全局透明度参数）
TVPAlphaBlend_HDA  — HDA = Hold Dest Alpha（保持目标 Alpha 不变）
```

来源：`krkr2/cpp/core/visual/tvpgl.h` 第 17-21 行的注释：
```
key to blending suffix:
d : destination has alpha
a : destination has additive-alpha
o : blend with opacity
```

---

## 4. 颜色空间：sRGB 与线性

### 4.1 人眼的非线性感知

人眼对暗部的亮度变化比亮部更敏感。如果用线性编码（数值翻倍=亮度翻倍），暗部的精度会严重不足（0~50 之间的微小变化人眼可以分辨，但 200~255 之间几乎分辨不出）。

**sRGB 编码**通过一条 Gamma 曲线（约 γ=2.2）将更多编码空间分配给暗部：

```
sRGB → 线性：
  if (sRGB <= 0.04045)
      线性 = sRGB / 12.92
  else
      线性 = ((sRGB + 0.055) / 1.055) ^ 2.4

线性 → sRGB：
  if (线性 <= 0.0031308)
      sRGB = 线性 * 12.92
  else
      sRGB = 1.055 * 线性^(1/2.4) - 0.055
```

### 4.2 Gamma 校正对混合的影响

在 sRGB 空间直接做 Alpha 混合会导致**中间色调偏暗**（因为非线性编码空间中的线性插值不等于真实亮度的线性插值）。正确流程是：

```
1. sRGB → 线性空间（去 Gamma）
2. 在线性空间做混合运算
3. 线性空间 → sRGB（加 Gamma）
```

但这样每次混合都需要两次 Gamma 转换，开销很大。实际工程中（包括 KrKr2），**通常直接在 sRGB 空间混合**，接受轻微的颜色偏差以换取性能。

### 4.3 KrKr2 的 Gamma 校正系统

KrKr2 提供了可配置的 Gamma 校正结构体（来源：`tvpgl.h` 第 31-44 行）：

```cpp
// Gamma 校正参数（每个通道独立调节）
typedef struct {
    float RGamma;     // R 通道 Gamma 值（0.10 ~ 9.99，默认 1.0）
    tjs_int RFloor;   // R 输出最小值（0 ~ 255）
    tjs_int RCeil;    // R 输出最大值（0 ~ 255）
    float GGamma;     // G 通道
    tjs_int GFloor;
    tjs_int GCeil;
    float BGamma;     // B 通道
    tjs_int BFloor;
    tjs_int BCeil;
} tTVPGLGammaAdjustData;

// 默认值：不做任何 Gamma 校正
// 来源：LayerBitmapIntf.cpp 第 43 行
tTVPGLGammaAdjustData TVPIntactGammaAdjustData = {
    1.0, 0, 255,   // R: gamma=1.0, floor=0, ceil=255
    1.0, 0, 255,   // G
    1.0, 0, 255    // B
};
```

### 4.4 完整 Gamma 转换实现

```cpp
#include <cmath>
#include <cstdint>

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
    
    // 钳制到 [0, 255] 并四舍五入
    int result = static_cast<int>(s * 255.0f + 0.5f);
    if (result < 0) return 0;
    if (result > 255) return 255;
    return static_cast<uint8_t>(result);
}

// 在线性空间做正确的 Alpha 混合
uint32_t blendLinear(uint32_t src, uint32_t dst) {
    float sa = ((src >> 24) & 0xFF) / 255.0f;  // 源 Alpha
    
    // 提取并转换到线性空间
    float sr = srgbToLinear((src >> 16) & 0xFF);
    float sg = srgbToLinear((src >> 8)  & 0xFF);
    float sb = srgbToLinear( src        & 0xFF);
    
    float dr = srgbToLinear((dst >> 16) & 0xFF);
    float dg = srgbToLinear((dst >> 8)  & 0xFF);
    float db = srgbToLinear( dst        & 0xFF);
    
    // 线性空间混合
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
```

---

## 5. 查找表（LUT）优化

### 5.1 为什么需要查找表？

`c * 255 / a` 这样的整数除法在 CPU 上非常昂贵（比乘法慢 20~40 倍）。KrKr2 使用预计算查找表避免运行时除法：

```cpp
// 来源：krkr2/cpp/core/visual/tvpgl.cpp 第 29-45 行

// TVPDivTable[a * 256 + b] = b * 255 / a（256KB 查找表）
unsigned char TVPDivTable[256 * 256];

// TVPOpacityOnOpacityTable：合成两个半透明层时的结果 Alpha
unsigned char TVPOpacityOnOpacityTable[256 * 256];

// TVPNegativeMulTable[a * 256 + b] = 255 - (255 - a) * (255 - b) / 255
// 用于 Screen 混合模式
unsigned char TVPNegativeMulTable[256 * 256];

// 倒数表：TVPRecipTable256[i] = 65536 / i（用乘法+移位替代除法）
tjs_uint32 TVPRecipTable256[256];
```

### 5.2 查找表初始化代码

```cpp
// 来源：krkr2/cpp/core/visual/tvpgl.cpp 第 224-295 行（简化）
static void TVPCreateTable() {
    for (int a = 0; a < 256; a++) {
        for (int b = 0; b < 256; b++) {
            int addr = b * 256 + a;
            
            // 计算 TVPOpacityOnOpacityTable：
            // 两个独立半透明层叠加后的等效透明度
            if (a) {
                float at = a / 255.0f, bt = b / 255.0f;
                float c = bt / at;
                c /= (1.0f - bt + c);
                TVPOpacityOnOpacityTable[addr] = (unsigned char)(c * 255);
            } else {
                TVPOpacityOnOpacityTable[addr] = 255;
            }
            
            // 计算 Screen 混合：255 - (255-a)(255-b)/255
            TVPNegativeMulTable[addr] =
                (unsigned char)(255 - (255 - a) * (255 - b) / 255);
        }
    }
    
    // 除法查找表
    for (int b = 0; b < 256; b++) {
        TVPDivTable[(0 << 8) + b] = 0;  // 除以 0 特殊处理
        for (int a = 1; a < 256; a++) {
            int tmp = b * 255 / a;
            if (tmp > 255) tmp = 255;
            TVPDivTable[(a << 8) + b] = (uint8_t)tmp;
        }
    }
    
    // 倒数表（用移位乘法替代除法）
    TVPRecipTable256[0] = 65536;
    for (int i = 1; i < 256; i++) {
        TVPRecipTable256[i] = 65536 / i;
        // 使用时：x / i ≈ x * TVPRecipTable256[i] >> 16
    }
}
```

---

