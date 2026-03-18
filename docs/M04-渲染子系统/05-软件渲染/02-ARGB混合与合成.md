# ARGB 混合与合成

> **所属模块：** M04-渲染子系统
> **前置知识：** [01-tvpgl像素操作库](./01-tvpgl像素操作库.md)、[P05-软件渲染原理](../../P05-软件渲染原理/README.md)
> **预计阅读时间：** 40 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 tTVPARGB 模板结构的设计与用途
2. 掌握标准 Alpha 与 Additive Alpha（预乘 Alpha）的区别和转换
3. 理解 Porter-Duff 合成模式的数学原理和实现
4. 能够使用 tvpps.inc 宏展开机制实现 Photoshop 混合模式
5. 理解查找表在复杂混合模式中的应用

## tTVPARGB 模板结构

`tTVPARGB` 是 KriKiri2 中用于操作 ARGB 像素的模板类，定义在 `argb.h` 中。它支持不同精度的像素操作，并处理字节序差异。

### 基础结构定义

```cpp
// argb.h 中的模板定义
template <typename base_type>
struct tTVPARGB {
    union {
        struct {
#if TJS_HOST_IS_LITTLE_ENDIAN
            base_type b;  // 蓝色通道
            base_type g;  // 绿色通道
            base_type r;  // 红色通道
            base_type a;  // Alpha 通道
#endif
#if TJS_HOST_IS_BIG_ENDIAN
            base_type a;  // 大端序：Alpha 在最前
            base_type r;
            base_type g;
            base_type b;
#endif
        };
        struct {
            base_type packed;  // 打包访问（仅当 base_type 为 uint32 时有意义）
        };
    };
    
    typedef base_type base_int_type;
    
    tTVPARGB() {}
    
    // 清零所有通道
    void Zero() { b = g = r = a = 0; }
    
    // 从打包的 uint32 赋值
    void operator=(tjs_uint32 v) {
        a = v >> 24;
        r = (v >> 16) & 0xff;
        g = (v >> 8) & 0xff;
        b = v & 0xff;
    }
    
    // 转换回打包的 uint32
    operator tjs_uint32() const {
        return b + (g << 8) + (r << 16) + (a << 24);
    }
    
    // 累加另一个像素
    void operator+=(const tTVPARGB &rhs) {
        b += rhs.b;
        g += rhs.g;
        r += rhs.r;
        a += rhs.a;
    }
    
    // 计算平均值（用于下采样）
    void average(tjs_int n) {
        b /= n;
        g /= n;
        r /= n;
        a /= n;
    }
};
```

### 模板特化

为常用类型提供了特化版本，优化性能：

```cpp
// 对 tjs_uint8（单个像素）的特化
template <>
void tTVPARGB<tjs_uint8>::Zero() {
    // 可以用单个 32 位写入清零整个结构
    *(tjs_uint32*)this = 0;
}

template <>
void tTVPARGB<tjs_uint8>::operator=(tjs_uint32 v) {
    // 直接内存拷贝，利用小端序
    *(tjs_uint32*)this = v;
}

template <>
tTVPARGB<tjs_uint8>::operator tjs_uint32() const {
    return *(tjs_uint32*)this;
}

// 对 tjs_uint16/tjs_uint32 的 average 特化
// 使用四舍五入而非截断
template <>
void tTVPARGB<tjs_uint16>::average(tjs_int n) {
    tjs_int half_n = n >> 1;
    b = (b + half_n) / n;  // 加半 n 实现四舍五入
    g = (g + half_n) / n;
    r = (r + half_n) / n;
    a = (a + half_n) / n;
}
```

### 使用场景示例

```cpp
#include "argb.h"

// 示例：计算多个像素的平均值（用于缩放/模糊）
tjs_uint32 AveragePixels(const tjs_uint32* pixels, int count) {
    // 使用 uint32 基类型避免溢出（8 位会溢出）
    tTVPARGB<tjs_uint32> sum;
    sum.Zero();
    
    for (int i = 0; i < count; i++) {
        sum += pixels[i];  // 自动解包并累加
    }
    
    sum.average(count);  // 计算平均
    return sum;          // 自动转换回 uint32
}

// 示例：2x2 下采样
tjs_uint32 Downsample2x2(const tjs_uint32* row0, const tjs_uint32* row1) {
    tTVPARGB<tjs_uint32> sum;
    sum.Zero();
    sum += row0[0];
    sum += row0[1];
    sum += row1[0];
    sum += row1[1];
    sum.average(4);
    return sum;
}
```

## Additive Alpha（预乘 Alpha）

Additive Alpha（也称预乘 Alpha、Premultiplied Alpha）是一种特殊的 Alpha 表示方式，在 KiriKiri2 中广泛使用。

### 标准 Alpha vs Additive Alpha

```
标准 Alpha (Straight Alpha):
  存储: (R, G, B, A)
  显示: R' = R, G' = G, B' = B
  混合: Result = Src * SrcA + Dst * (1 - SrcA)

Additive Alpha (Premultiplied Alpha):
  存储: (R*A, G*A, B*A, A)  // RGB 已乘以 Alpha
  显示: R' = R/A, G' = G/A, B' = B/A
  混合: Result = Src + Dst * (1 - SrcA)  // 更简单！
```

### 为什么使用 Additive Alpha

1. **混合计算更简单**：少一次乘法
2. **避免颜色溢出**：半透明边缘不会出现白边
3. **线性插值正确**：标准 Alpha 插值会产生伪影

### tTVPARGB_AA 结构

`tTVPARGB_AA` 是 `tTVPARGB` 的派生类，专门处理 Additive Alpha 格式的像素：

```cpp
// argb.h
template <typename base_type>
struct tTVPARGB_AA : public tTVPARGB<base_type> {
    
    // 重写：从标准格式加载时自动预乘
    void operator+=(tjs_uint32 v) {
        tjs_int aadj;
        this->a += (aadj = (v >> 24));
        aadj += aadj >> 7;  // 将 [0,255] 映射到 [0,256] 以便位移
        
        // RGB 通道乘以调整后的 Alpha
        this->b += (v & 0xff) * aadj >> 8;
        this->g += ((v >> 8) & 0xff) * aadj >> 8;
        this->r += ((v >> 16) & 0xff) * aadj >> 8;
    }
    
    void operator-=(tjs_uint32 v) {
        tjs_int aadj;
        this->a -= (aadj = (v >> 24));
        aadj += aadj >> 7;
        this->b -= (v & 0xff) * aadj >> 8;
        this->g -= ((v >> 8) & 0xff) * aadj >> 8;
        this->r -= ((v >> 16) & 0xff) * aadj >> 8;
    }
    
    // 从标准 Alpha 格式赋值（自动预乘）
    void operator=(tjs_uint32 v) {
        this->a = v >> 24;
        tjs_int aadj = this->a + (this->a >> 7);  // Alpha 调整
        this->r = ((v >> 16) & 0xff) * aadj >> 8;
        this->g = ((v >> 8) & 0xff) * aadj >> 8;
        this->b = (v & 0xff) * aadj >> 8;
    }
    
    // 转换回标准 Alpha 格式（使用除法表反预乘）
    operator tjs_uint32() const {
        // TVPDivTable[a * 256 + color] = color * 255 / a
        tjs_uint8 *t = TVPDivTable + (this->a << 8);
        return t[this->b] + 
               (t[this->g] << 8) + 
               (t[this->r] << 16) + 
               (this->a << 24);
    }
};
```

### Alpha 格式转换

```cpp
// tvpgl.cpp 中的格式转换函数

// 标准 Alpha -> Additive Alpha
TVP_GL_FUNC_DECL(void, TVPConvertAlphaToAdditiveAlpha_c,
    (tjs_uint32 *buf, tjs_int len))
{
    while (len--) {
        tjs_uint32 pixel = *buf;
        tjs_uint32 alpha = pixel >> 24;
        
        // 预乘：RGB *= Alpha/255
        // 优化：用 >>8 代替 /255
        alpha += alpha >> 7;  // 调整到 [0,256]
        
        tjs_uint32 r = ((pixel >> 16) & 0xff) * alpha >> 8;
        tjs_uint32 g = ((pixel >> 8) & 0xff) * alpha >> 8;
        tjs_uint32 b = (pixel & 0xff) * alpha >> 8;
        
        *buf++ = (pixel & 0xff000000) | (r << 16) | (g << 8) | b;
    }
}

// Additive Alpha -> 标准 Alpha（使用除法表）
TVP_GL_FUNC_DECL(void, TVPConvertAdditiveAlphaToAlpha_c,
    (tjs_uint32 *buf, tjs_int len))
{
    while (len--) {
        tjs_uint32 pixel = *buf;
        tjs_uint32 alpha = pixel >> 24;
        
        // 反预乘：RGB = RGB * 255 / Alpha
        // 使用查找表避免除法
        tjs_uint8 *divTable = TVPDivTable + (alpha << 8);
        
        tjs_uint32 r = divTable[(pixel >> 16) & 0xff];
        tjs_uint32 g = divTable[(pixel >> 8) & 0xff];
        tjs_uint32 b = divTable[pixel & 0xff];
        
        *buf++ = (alpha << 24) | (r << 16) | (g << 8) | b;
    }
}
```

## Porter-Duff 合成模式

Porter-Duff 合成模式定义了两个图像如何根据各自的 Alpha 通道组合在一起。

### 基础公式

对于源像素 `(Rs, Gs, Bs, As)` 和目标像素 `(Rd, Gd, Bd, Ad)`：

```
结果颜色 = Src * Fa + Dst * Fb
结果Alpha = As * Fa + Ad * Fb
```

其中 `Fa` 和 `Fb` 是混合因子，由合成模式决定：

| 模式 | Fa | Fb | 描述 |
|------|----|----|------|
| Clear | 0 | 0 | 清空 |
| Src | 1 | 0 | 只显示源 |
| Dst | 0 | 1 | 只显示目标 |
| Src Over | 1 | 1-As | 源覆盖目标 |
| Dst Over | 1-Ad | 1 | 目标覆盖源 |
| Src In | Ad | 0 | 源与目标相交部分 |
| Dst In | 0 | As | 目标与源相交部分 |
| Src Out | 1-Ad | 0 | 源减去目标 |
| Dst Out | 0 | 1-As | 目标减去源 |
| Src Atop | Ad | 1-As | 源在目标上 |
| Dst Atop | 1-Ad | As | 目标在源上 |
| Xor | 1-Ad | 1-As | 异或 |

### KiriKiri2 中的实现

最常用的是 **Src Over** 模式（标准 Alpha 混合）：

```cpp
// 标准 Alpha 混合（Src Over）
static tjs_uint32 TVP_INLINE_FUNC AlphaBlend_SrcOver(
    tjs_uint32 dest, tjs_uint32 src)
{
    tjs_uint32 src_alpha = src >> 24;
    
    if (src_alpha == 0) return dest;    // 完全透明
    if (src_alpha == 255) return src;   // 完全不透明
    
    // Fa = 1, Fb = 1 - As
    // Result = Src + Dst * (1 - As)
    
    tjs_uint32 inv_alpha = 255 - src_alpha;
    
    // 分离处理 R/B 和 G 通道
    tjs_uint32 d_rb = dest & 0xff00ff;
    tjs_uint32 d_g = dest & 0x00ff00;
    tjs_uint32 s_rb = src & 0xff00ff;
    tjs_uint32 s_g = src & 0x00ff00;
    
    // Result = Src * 1 + Dst * (1 - SrcA)
    tjs_uint32 r_rb = s_rb + ((d_rb * inv_alpha >> 8) & 0xff00ff);
    tjs_uint32 r_g = s_g + ((d_g * inv_alpha >> 8) & 0x00ff00);
    
    return (src_alpha << 24) | r_rb | r_g;
}

// 使用 Additive Alpha 的版本（更简单）
static tjs_uint32 TVP_INLINE_FUNC AdditiveAlphaBlend_SrcOver(
    tjs_uint32 dest, tjs_uint32 src_premul)
{
    // src_premul 已经是预乘格式
    tjs_uint32 inv_alpha = 255 - (src_premul >> 24);
    
    // Result = Src_premul + Dst * (1 - SrcA)
    tjs_uint32 d_rb = (((dest & 0xff00ff) * inv_alpha) >> 8) & 0xff00ff;
    tjs_uint32 d_g = (((dest & 0x00ff00) * inv_alpha) >> 8) & 0x00ff00;
    
    return src_premul + d_rb + d_g;
}
```

### Alpha 合成

当目标也有 Alpha 通道时，需要正确计算结果 Alpha：

```cpp
// Da' = As + Da * (1 - As)
static tjs_uint32 TVP_INLINE_FUNC AlphaBlend_a_a(
    tjs_uint32 dest, tjs_uint32 src)
{
    tjs_uint32 dest_alpha = dest >> 24;
    tjs_uint32 src_alpha = src >> 24;
    
    // 计算结果 Alpha
    // Da' = As + Da - As * Da / 255
    tjs_uint32 result_alpha = dest_alpha + src_alpha - 
                              (dest_alpha * src_alpha >> 8);
    result_alpha -= (result_alpha >> 8);  // 修正 >>8 vs /255 的误差
    
    // 计算颜色
    tjs_uint32 inv_src_alpha = 255 - src_alpha;
    tjs_uint32 d_rb = (((dest & 0xff00ff) * inv_src_alpha) >> 8) & 0xff00ff;
    tjs_uint32 d_g = (((dest & 0x00ff00) * inv_src_alpha) >> 8) & 0x00ff00;
    
    return (result_alpha << 24) | 
           ((src & 0xffffff) + d_rb + d_g);
}
```

## Photoshop 混合模式

KiriKiri2 实现了完整的 Photoshop 图层混合模式，通过 `tvpps.inc` 宏展开机制生成。

### 混合模式列表

| 模式 | 函数名 | 公式 |
|------|--------|------|
| Normal | TVPPsAlphaBlend | 标准 Alpha 混合 |
| Add | TVPPsAddBlend | min(S + D, 255) |
| Subtract | TVPPsSubBlend | max(D - S, 0) |
| Multiply | TVPPsMulBlend | S * D / 255 |
| Screen | TVPPsScreenBlend | 255 - (255-S)(255-D)/255 |
| Overlay | TVPPsOverlayBlend | D<128 ? 2SD/255 : 255-2(255-S)(255-D)/255 |
| Hard Light | TVPPsHardLightBlend | S<128 ? 2SD/255 : 255-2(255-S)(255-D)/255 |
| Soft Light | TVPPsSoftLightBlend | pow(D, (S>=128 ? 128/S : (1-S/255)/0.5)) |
| Color Dodge | TVPPsColorDodgeBlend | D / (255-S) * 255 |
| Color Burn | TVPPsColorBurnBlend | 255 - (255-D) / S * 255 |
| Lighten | TVPPsLightenBlend | max(S, D) |
| Darken | TVPPsDarkenBlend | min(S, D) |
| Difference | TVPPsDiffBlend | abs(S - D) |
| Exclusion | TVPPsExclusionBlend | S + D - 2*S*D/255 |

### 宏展开机制

`tvpps.inc` 使用宏展开自动生成四个变体函数：

```cpp
// tvpps.inc 结构

// 普通版本
#define OPERATION1 { \
    tjs_uint32 s = *buf++, d = *dest, a = s >> 24; \
    TVPPS_OPERATION;  /* 这里替换为具体的混合操作 */ \
    *dest++ = s; \
}
void TVPPS_FUNC_NORM(tjs_uint32 *dest, const tjs_uint32 *buf, tjs_int len) {
    TVPPS_MAINLOOP  // Duff's Device 循环
}

// 带不透明度版本
#define OPERATION1 { \
    tjs_uint32 s = *buf++, d = *dest, \
               a = ((s >> 24) * opa) >> 8;  /* Alpha 乘以全局不透明度 */ \
    TVPPS_OPERATION; \
    *dest++ = s; \
}
void TVPPS_FUNC_O(tjs_uint32 *dest, const tjs_uint32 *buf, 
                  tjs_int len, tjs_int opa) {
    TVPPS_MAINLOOP
}

// HDA 版本（Hold Destination Alpha）
#define OPERATION1 { \
    tjs_uint32 s = *buf++, d = *dest, a = s >> 24; \
    TVPPS_OPERATION; \
    *dest++ = s | (d & 0xff000000);  /* 保持目标 Alpha */ \
}
void TVPPS_FUNC_HDA(tjs_uint32 *dest, const tjs_uint32 *buf, tjs_int len) {
    TVPPS_MAINLOOP
}

// HDA + 不透明度版本
void TVPPS_FUNC_HDA_O(...) { ... }
```

### 混合操作定义

在 `tvpgl.cpp` 中定义具体的混合操作：

```cpp
// 标准 Alpha 混合操作
#define TVPPS_ALPHABLEND { \
    tjs_uint32 d1 = d & 0x00ff00ff, d2 = d & 0x0000ff00; \
    s = ((((((s & 0x00ff00ff) - d1) * a) >> 8) + d1) & 0x00ff00ff) | \
        ((((((s & 0x0000ff00) - d2) * a) >> 8) + d2) & 0x0000ff00); \
}

// Multiply 混合操作
#define TVPPsOperationMulBlend { \
    /* S * D / 255 */ \
    s = (((((d >> 16) & 0xff) * (s & 0x00ff0000)) & 0xff000000) | \
         ((((d >> 8) & 0xff) * (s & 0x0000ff00)) & 0x00ff0000) | \
         ((((d >> 0) & 0xff) * (s & 0x000000ff)))) >> 8; \
    TVPPS_ALPHABLEND  /* 最后应用 Alpha 混合 */ \
}

// Screen 混合操作
#define TVPPsOperationScreenBlend { \
    /* 255 - (255-S)(255-D)/255 = S + D - S*D/255 */ \
    tjs_uint32 sd1, sd2; \
    sd1 = (((((d >> 16) & 0xff) * (s & 0x00ff0000)) & 0xff000000) | \
           ((((d >> 0) & 0xff) * (s & 0x000000ff)))) >> 8; \
    sd2 = (((((d >> 8) & 0xff) * (s & 0x0000ff00)) & 0x00ff0000)) >> 8; \
    /* Result = S - S*D/255，然后 Alpha 混合 */ \
    s = ((((((s & 0x00ff00ff) - sd1) * a) >> 8) + (d & 0x00ff00ff)) & 0x00ff00ff) | \
        ((((((s & 0x0000ff00) - sd2) * a) >> 8) + (d & 0x0000ff00)) & 0x0000ff00); \
}
```

### 查找表加速

复杂混合模式使用预计算查找表：

```cpp
// 查找表声明（tvpgl.cpp）
unsigned char TVPPsTableSoftLight[256][256];   // Soft Light
unsigned char TVPPsTableColorDodge[256][256];  // Color Dodge
unsigned char TVPPsTableColorBurn[256][256];   // Color Burn
unsigned char TVPPsTableOverlay[256][256];     // Overlay（可选）

// 表初始化
void TVPPsMakeTable() {
    for (int s = 0; s < 256; s++) {
        for (int d = 0; d < 256; d++) {
            // Soft Light: pow(D/255, (S>=128 ? 128/S : (1-S/255)/0.5))
            TVPPsTableSoftLight[s][d] = (s >= 128)
                ? (unsigned char)(pow(d / 255.0, 128.0 / s) * 255.0)
                : (unsigned char)(pow(d / 255.0, (1.0 - s / 255.0) / 0.5) * 255.0);
            
            // Color Dodge: D / (255-S) * 255，饱和到 255
            TVPPsTableColorDodge[s][d] = 
                ((255 - s) <= d) ? 0xff : ((d * 255) / (255 - s));
            
            // Color Burn: 255 - (255-D) / S * 255，饱和到 0
            TVPPsTableColorBurn[s][d] = 
                (s <= (255 - d)) ? 0x00 : (255 - ((255 - d) * 255) / s);
            
            // Overlay: D<128 ? 2SD/255 : 255 - 2(255-S)(255-D)/255
            TVPPsTableOverlay[s][d] = (d < 128)
                ? ((s * d * 2) / 255)
                : (((s + d) * 2) - ((s * d * 2) / 255) - 255);
        }
    }
}

// 使用查找表的 Soft Light 混合
#define TVPPsOperationSoftLightBlend { \
    s = (TVPPsTableSoftLight[(s >> 16) & 0xff][(d >> 16) & 0xff] << 16) | \
        (TVPPsTableSoftLight[(s >> 8) & 0xff][(d >> 8) & 0xff] << 8) | \
        (TVPPsTableSoftLight[(s >> 0) & 0xff][(d >> 0) & 0xff] << 0); \
    TVPPS_ALPHABLEND \
}
```

### 生成函数实例

使用 `#include` 展开生成所有混合函数：

```cpp
// 生成 Multiply 混合函数
#define TVPPS_FUNC_NORM TVPPsMulBlend_c
#define TVPPS_FUNC_O TVPPsMulBlend_o_c
#define TVPPS_FUNC_HDA TVPPsMulBlend_HDA_c
#define TVPPS_FUNC_HDA_O TVPPsMulBlend_HDA_o_c
#define TVPPS_OPERATION TVPPsOperationMulBlend
#include "tvpps.inc"  // 展开生成 4 个函数

// 生成 Screen 混合函数
#define TVPPS_FUNC_NORM TVPPsScreenBlend_c
#define TVPPS_FUNC_O TVPPsScreenBlend_o_c
#define TVPPS_FUNC_HDA TVPPsScreenBlend_HDA_c
#define TVPPS_FUNC_HDA_O TVPPsScreenBlend_HDA_o_c
#define TVPPS_OPERATION TVPPsOperationScreenBlend
#include "tvpps.inc"

// ... 其他混合模式类似
```

## 动手实践

### 练习 1：实现 Multiply 混合

```cpp
#include <cstdint>
#include <cstdio>

// Multiply 混合：每个通道相乘后除以 255
uint32_t MultiplyBlend(uint32_t dest, uint32_t src) {
    // 提取各通道
    uint32_t sr = (src >> 16) & 0xff;
    uint32_t sg = (src >> 8) & 0xff;
    uint32_t sb = src & 0xff;
    uint32_t sa = src >> 24;
    
    uint32_t dr = (dest >> 16) & 0xff;
    uint32_t dg = (dest >> 8) & 0xff;
    uint32_t db = dest & 0xff;
    
    // Multiply: S * D / 255
    uint32_t mr = (sr * dr + 127) / 255;  // +127 实现四舍五入
    uint32_t mg = (sg * dg + 127) / 255;
    uint32_t mb = (sb * db + 127) / 255;
    
    // 应用 Alpha 混合：Result = Mul + (Dest - Mul) * (1 - Alpha)
    // 简化为：Result = Dest + (Mul - Dest) * Alpha
    uint32_t rr = dr + ((int)(mr - dr) * sa >> 8);
    uint32_t rg = dg + ((int)(mg - dg) * sa >> 8);
    uint32_t rb = db + ((int)(mb - db) * sa >> 8);
    
    return (dest & 0xff000000) | (rr << 16) | (rg << 8) | rb;
}

int main() {
    // 测试：白色 * 红色 = 红色
    uint32_t white = 0xffffffff;
    uint32_t red = 0x80ff0000;  // 50% 透明红色
    uint32_t result = MultiplyBlend(white, red);
    printf("White * Red(50%%) = 0x%08x\n", result);
    // 期望：接近 0xffff8080（粉红色）
    
    return 0;
}
```

### 练习 2：预乘 Alpha 转换

```cpp
#include <cstdint>
#include <cstdio>

// 标准 Alpha -> 预乘 Alpha
uint32_t ToPremultiplied(uint32_t pixel) {
    uint32_t alpha = pixel >> 24;
    if (alpha == 255) return pixel;  // 不需要转换
    if (alpha == 0) return 0;        // 完全透明
    
    // RGB *= Alpha / 255
    uint32_t adj_alpha = alpha + (alpha >> 7);  // [0,255] -> [0,256]
    
    uint32_t r = (((pixel >> 16) & 0xff) * adj_alpha) >> 8;
    uint32_t g = (((pixel >> 8) & 0xff) * adj_alpha) >> 8;
    uint32_t b = ((pixel & 0xff) * adj_alpha) >> 8;
    
    return (alpha << 24) | (r << 16) | (g << 8) | b;
}

// 预乘 Alpha -> 标准 Alpha
uint32_t FromPremultiplied(uint32_t pixel) {
    uint32_t alpha = pixel >> 24;
    if (alpha == 255) return pixel;
    if (alpha == 0) return 0;
    
    // RGB = RGB * 255 / Alpha
    uint32_t r = ((pixel >> 16) & 0xff) * 255 / alpha;
    uint32_t g = ((pixel >> 8) & 0xff) * 255 / alpha;
    uint32_t b = (pixel & 0xff) * 255 / alpha;
    
    // 防止溢出
    if (r > 255) r = 255;
    if (g > 255) g = 255;
    if (b > 255) b = 255;
    
    return (alpha << 24) | (r << 16) | (g << 8) | b;
}

int main() {
    uint32_t original = 0x80ff8040;  // 50% Alpha, R=255, G=128, B=64
    
    uint32_t premul = ToPremultiplied(original);
    printf("Original:      0x%08x\n", original);
    printf("Premultiplied: 0x%08x\n", premul);
    // 期望：约 0x80804020（RGB 减半）
    
    uint32_t restored = FromPremultiplied(premul);
    printf("Restored:      0x%08x\n", restored);
    // 期望：接近原始值
    
    return 0;
}
```

## 对照项目源码

### 关键文件

| 文件 | 行号范围 | 内容 |
|------|----------|------|
| `cpp/core/visual/argb.h` | 22-98 | tTVPARGB 模板定义 |
| `cpp/core/visual/argb.h` | 126-176 | tTVPARGB_AA 预乘 Alpha 结构 |
| `cpp/core/visual/argb.cpp` | 全部 | 模板特化实现 |
| `cpp/core/visual/tvpps.inc` | 1-57 | 混合函数宏展开模板 |
| `cpp/core/visual/tvpgl.cpp` | 12496-12503 | PS 混合模式查找表声明 |
| `cpp/core/visual/tvpgl.cpp` | 12526-12727 | PS 混合操作宏定义 |
| `cpp/core/visual/tvpgl.cpp` | 12733-12752 | TVPPsMakeTable 查找表初始化 |
| `cpp/core/visual/tvpgl.cpp` | 12758-12868 | PS 混合函数生成（#include tvpps.inc） |

## 本节小结

- **tTVPARGB** 模板提供了平台无关的 ARGB 像素操作接口
- **Additive Alpha（预乘 Alpha）** 简化了混合计算，避免了颜色溢出
- **Porter-Duff** 合成模式是所有 Alpha 混合的理论基础
- **tvpps.inc** 宏展开机制高效生成多个混合函数变体
- **查找表** 用于加速复杂的混合模式（Soft Light、Color Dodge 等）
- 每种混合模式有 4 个变体：普通、带不透明度、HDA、HDA+不透明度

## 练习题与答案

### 题目 1：Alpha 格式识别

给定以下像素值，判断它是标准 Alpha 还是预乘 Alpha 格式，并说明理由：

```
像素 A: 0x80FF8040  (A=128, R=255, G=128, B=64)
像素 B: 0x80804020  (A=128, R=128, G=64, B=32)
```

<details>
<summary>查看答案</summary>

**像素 A (0x80FF8040)** 是**标准 Alpha** 格式：
- 理由：R=255 超过了 A=128，这在预乘格式中是不可能的
- 预乘格式中，RGB 各通道值不能超过 Alpha 值
- 因为预乘后 `R' = R * A / 255`，最大值为 `255 * 128 / 255 ≈ 128`

**像素 B (0x80804020)** 可能是**预乘 Alpha** 格式：
- R=128, G=64, B=32 都不超过 A=128
- 如果从像素 A 转换：
  - R' = 255 * 128 / 255 ≈ 128 ✓
  - G' = 128 * 128 / 255 ≈ 64 ✓
  - B' = 64 * 128 / 255 ≈ 32 ✓
- 这正是像素 A 的预乘形式

</details>

### 题目 2：Porter-Duff 计算

使用 Src Over 模式混合以下两个像素，计算结果：

```
源像素 (Src): ARGB = (192, 255, 0, 0)    // 75% 不透明红色
目标像素 (Dst): ARGB = (255, 0, 0, 255)  // 不透明蓝色
```

<details>
<summary>查看答案</summary>

Src Over 公式：`Result = Src + Dst * (1 - SrcAlpha)`

1. **计算 (1 - SrcAlpha)**：
   - SrcAlpha = 192/255 ≈ 0.753
   - 1 - SrcAlpha ≈ 0.247

2. **计算 Dst * (1 - SrcAlpha)**：
   - R = 0 * 0.247 = 0
   - G = 0 * 0.247 = 0
   - B = 255 * 0.247 ≈ 63

3. **计算 Src + 结果**：
   - R = 255 + 0 = 255
   - G = 0 + 0 = 0
   - B = 0 + 63 = 63

4. **计算结果 Alpha**：
   - 使用公式：As + Ad * (1 - As)
   - = 192/255 + 1 * (1 - 192/255)
   - = 0.753 + 0.247 = 1.0 = 255

**最终结果**: ARGB = (255, 255, 0, 63) 或 0xFFFF003F

即：不透明的深红色（带一点蓝色）

</details>

### 题目 3：实现 Darken 混合模式

实现 Darken 混合模式函数，选择源和目标中较暗的值：

```cpp
uint32_t DarkenBlend(uint32_t dest, uint32_t src, int alpha);
```

<details>
<summary>查看答案</summary>

```cpp
#include <cstdint>
#include <algorithm>

uint32_t DarkenBlend(uint32_t dest, uint32_t src, int alpha) {
    // 提取源和目标的 RGB
    int sr = (src >> 16) & 0xff;
    int sg = (src >> 8) & 0xff;
    int sb = src & 0xff;
    
    int dr = (dest >> 16) & 0xff;
    int dg = (dest >> 8) & 0xff;
    int db = dest & 0xff;
    
    // Darken: 选择较小值
    int mr = std::min(sr, dr);
    int mg = std::min(sg, dg);
    int mb = std::min(sb, db);
    
    // 应用 Alpha 混合
    // Result = Dest + (Darken - Dest) * Alpha / 255
    int rr = dr + ((mr - dr) * alpha >> 8);
    int rg = dg + ((mg - dg) * alpha >> 8);
    int rb = db + ((mb - db) * alpha >> 8);
    
    // 保持目标 Alpha（HDA 风格）
    return (dest & 0xff000000) | (rr << 16) | (rg << 8) | rb;
}

// 优化版本（使用位操作，模仿 tvpgl 风格）
uint32_t DarkenBlend_Opt(uint32_t dest, uint32_t src, int alpha) {
    // 计算 d < s 的掩码
    // 如果 d < s，对应字节为 0xff，否则为 0x00
    uint32_t n = (((~dest & src) << 1) + ((~dest ^ src) & 0x00fefefe)) & 0x01010100;
    n = ((n >> 8) + 0x007f7f7f) ^ 0x007f7f7f;
    
    // 选择较小值：d < s 时选 d，否则选 s
    uint32_t darken = (dest & n) | (src & ~n);
    
    // Alpha 混合
    uint32_t d1 = dest & 0x00ff00ff;
    uint32_t d2 = dest & 0x0000ff00;
    uint32_t s1 = darken & 0x00ff00ff;
    uint32_t s2 = darken & 0x0000ff00;
    
    uint32_t r1 = (((s1 - d1) * alpha >> 8) + d1) & 0x00ff00ff;
    uint32_t r2 = (((s2 - d2) * alpha >> 8) + d2) & 0x0000ff00;
    
    return (dest & 0xff000000) | r1 | r2;
}
```

</details>

## 下一步

下一节 [03-SSE-ASM优化路径](./03-SSE-ASM优化路径.md) 将深入讲解：
- TVPGL_ASM_Init() 初始化机制
- CPU 特性检测（CPUID）
- SSE/NEON 优化实现
- 函数指针运行时切换
- SIMD 优化技巧与性能对比
