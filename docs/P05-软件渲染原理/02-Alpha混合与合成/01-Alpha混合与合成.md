# Alpha 混合与合成

> **所属模块：** P05-软件渲染原理
> **前置知识：** [P05-01 像素与颜色空间](../01-像素与颜色空间/01-像素与颜色空间.md)、C++ 位运算基础
> **预计阅读时间：** 45 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 Alpha 通道的含义及其在 ARGB 像素中的存储方式
2. 手写标准 Alpha 混合公式（straight alpha 与 premultiplied alpha）
3. 掌握 Porter-Duff 12 种合成模式的数学定义与应用场景
4. 理解 KrKr2 引擎 30+ 种混合模式（`tTVPBBBltMethod` 枚举）的设计思路
5. 看懂 KrKr2 `blend_functor_c.h` 中 functor + 宏展开的模板架构
6. 使用查找表（LUT）优化除法运算的技巧

---

## 1. Alpha 通道基础

### 1.1 什么是 Alpha

Alpha 通道表示像素的不透明度。在 32 位 ARGB 格式中：

```
像素 = 0xAARRGGBB
         ↑↑  ↑↑  ↑↑  ↑↑
         A   R   G   B
```

- `A = 0x00`：完全透明
- `A = 0xFF`：完全不透明
- `A = 0x80`：半透明（约 50%）

### 1.2 Straight Alpha vs Premultiplied Alpha

这是两种完全不同的像素存储约定，混用会导致画面出错。

**Straight Alpha（非预乘）**：

颜色通道存储的是原始颜色值，Alpha 独立存储：

```
R_stored = R_original
G_stored = G_original
B_stored = B_original
A_stored = opacity
```

**Premultiplied Alpha（预乘）**：

颜色通道已经乘以 Alpha：

```
R_stored = R_original × (A / 255)
G_stored = G_original × (A / 255)
B_stored = B_original × (A / 255)
A_stored = opacity
```

预乘格式的优势在于合成运算更简单、更快（少一次乘法）。KrKr2 引擎中两种格式并存：

- **Straight Alpha**：大部分常规混合模式使用（`bmAlpha` 等）
- **Premultiplied Alpha**（引擎中称 "Additive Alpha"）：`bmAddAlpha` 系列使用

在 KrKr2 源码中，函数名后缀 `_a` 表示源为 premultiplied alpha，`_d` 表示目标有独立 alpha 通道：

```cpp
// 文件: cpp/core/visual/tvpgl.h
// _n_a: normal (straight) source, premul alpha dest (实际是 add alpha blend)
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAddAlphaBlend_n_a,
    (tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len));
// _a_a: premul alpha source, premul alpha dest
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAddAlphaBlend_a_a,
    (tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len));
```

### 1.3 转换公式

Straight → Premultiplied：

```cpp
// 文件: cpp/core/visual/gl/blend_functor_c.h 中的 alpha_to_premulalpha_func
tjs_uint32 straight_to_premul(tjs_uint32 pixel) {
    tjs_uint32 a = pixel >> 24;          // 提取 alpha
    tjs_uint32 rb = pixel & 0x00ff00ff;  // 提取 R 和 B
    tjs_uint32 g  = pixel & 0x0000ff00;  // 提取 G
    rb = (rb * a >> 8) & 0x00ff00ff;     // R、B 各乘 alpha
    g  = (g  * a >> 8) & 0x0000ff00;     // G 乘 alpha
    return rb | g | (a << 24);           // 重组
}
```

> **常见错误：** `>> 8` 而非 `/ 255`。严格来说 `x * a / 255` 与 `x * a >> 8` 有约 0.4% 的误差。KrKr2 为了性能统一使用 `>> 8`，这是游戏引擎中的常见做法。精确除以 255 的技巧是 `(x * a + 128) / 255` 或使用查找表（见第 5 节）。

---

## 2. 标准 Alpha 混合

### 2.1 Over 运算公式

最常用的混合模式——将前景叠加到背景上：

```
C_out = C_src × A_src + C_dst × (1 - A_src)
```

用整数运算（0~255 范围）实现：

```cpp
// 最基础的 Alpha Blend（逐通道，straight alpha）
tjs_uint32 alpha_blend(tjs_uint32 dst, tjs_uint32 src) {
    tjs_uint32 a = src >> 24;  // 源 alpha [0..255]
    // 分离 R+B 和 G 通道，用位掩码技巧并行计算两个通道
    tjs_uint32 d_rb = dst & 0xff00ff;   // dst 的 R 和 B
    tjs_uint32 s_rb = src & 0xff00ff;   // src 的 R 和 B
    // d_rb + (s_rb - d_rb) * a / 256 —— 本质是线性插值
    d_rb = (d_rb + ((s_rb - d_rb) * a >> 8)) & 0xff00ff;

    tjs_uint32 d_g = dst & 0xff00;      // dst 的 G
    tjs_uint32 s_g = src & 0xff00;      // src 的 G
    d_g = (d_g + ((s_g - d_g) * a >> 8)) & 0xff00;

    return d_rb | d_g;
}
```

这正是 KrKr2 中 `alpha_blend_func` 的实现（`blend_functor_c.h` 第 67-76 行）：

```cpp
// 文件: cpp/core/visual/gl/blend_functor_c.h, 第 67-76 行
struct alpha_blend_func {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s,
                                 tjs_uint32 a) const {
        tjs_uint32 d1 = d & 0xff00ff;
        d1 = (d1 + (((s & 0xff00ff) - d1) * a >> 8)) & 0xff00ff;
        d &= 0xff00;
        s &= 0xff00;
        return d1 + ((d + ((s - d) * a >> 8)) & 0xff00);
    }
};
```

### 2.2 位运算技巧解析

上面的代码隐含了一个巧妙的优化——**双通道并行计算**：

```
像素布局: 0x__RR__BB  (掩码 0x00ff00ff)
```

因为 R 在第 16-23 位、B 在第 0-7 位，中间隔了 8 位的 G，所以 `(R * a >> 8)` 和 `(B * a >> 8)` 不会互相溢出（只要 a ≤ 255）。这样一次乘法就处理了两个通道，速度翻倍。

```
     R 通道 (16-23位)    [空隙8位]    B 通道 (0-7位)
     ├───────┤            ├──┤         ├───────┤
0x   00  RR  00  BB
  ×  a
  =  00 RR×a 00 BB×a     // R×a 最大 255×255=65025，占 16 位
  >> 8
  =  00  RR' 00  BB'     // 右移8位后各回到原位
  &  0x00ff00ff           // 裁剪，防止溢出
```

### 2.3 HDA 变体（Hold Dest Alpha）

HDA = Hold Destination Alpha，即混合时保留目标像素的 Alpha 通道不被修改。在图层合成中，目标图层的透明度信息需要保持不变时使用：

```cpp
// HDA: 混合颜色，但保持 dst 的 alpha
tjs_uint32 alpha_blend_hda(tjs_uint32 dst, tjs_uint32 src) {
    tjs_uint32 result = alpha_blend(dst, src);
    return (result & 0x00ffffff) | (dst & 0xff000000);  // 替换回原 alpha
}
```

### 2.4 Opacity（全局不透明度）

除了每像素的 Alpha，还可以给整个图层指定一个全局不透明度（opacity），范围 0~255。在 KrKr2 中，带 `_o` 后缀的函数处理 opacity：

```cpp
// 文件: cpp/core/visual/gl/blend_variation.h（通过 blend_functor_c.h 引入）
// translucent_op: 在执行混合前先将 alpha 乘以 opacity
template<typename FUNC>
struct translucent_op {
    FUNC func_;
    tjs_int opa_;
    translucent_op(tjs_int opa) : opa_(opa) {}
    tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        tjs_uint32 a = (s >> 24) * opa_ >> 8;  // alpha × opacity
        return func_(d, s, a);
    }
};
```

---

## 3. Porter-Duff 合成模式

Thomas Porter 和 Tom Duff 于 1984 年提出了 12 种像素合成运算（"Compositing Digital Images"），是所有现代 2D 图形系统的理论基础。

### 3.1 基本概念

对于源（S）和目标（D）两个图层，每个像素区域可分为四个部分：

```
┌─────────────────┐
│  S∩D  │  S only │
│───────│─────────│
│D only │ Neither │
└─────────────────┘
```

12 种模式通过选择保留/丢弃这四个区域来定义：

### 3.2 完整列表

| 模式 | 保留区域 | 公式 (Co = 输出颜色, Ao = 输出Alpha) |
|------|----------|--------------------------------------|
| Clear | 无 | Co=0, Ao=0 |
| Src | S | Co=Cs, Ao=As |
| Dst | D | Co=Cd, Ao=Ad |
| SrcOver | S∩D + S + D | Co=Cs + Cd(1-As), Ao=As + Ad(1-As) |
| DstOver | S∩D + S + D | Co=Cs(1-Ad) + Cd, Ao=As(1-Ad) + Ad |
| SrcIn | S∩D (src部分) | Co=Cs·Ad, Ao=As·Ad |
| DstIn | S∩D (dst部分) | Co=Cd·As, Ao=Ad·As |
| SrcOut | S only | Co=Cs(1-Ad), Ao=As(1-Ad) |
| DstOut | D only | Co=Cd(1-As), Ao=Ad(1-As) |
| SrcAtop | S∩D(src) + D | Co=Cs·Ad + Cd(1-As), Ao=Ad |
| DstAtop | S∩D(dst) + S | Co=Cs(1-Ad) + Cd·As, Ao=As |
| Xor | S only + D only | Co=Cs(1-Ad) + Cd(1-As), Ao=As(1-Ad) + Ad(1-As) |

KrKr2 的 `bmAlpha` 模式对应 **SrcOver**，这是最常用的图层叠加方式。

### 3.3 SrcOver 实现（premultiplied alpha 版）

在预乘格式下，SrcOver 公式极为简洁：

```cpp
// Premultiplied Alpha 的 SrcOver（KrKr2 中的 premulalpha_blend_n_a_func）
tjs_uint32 premul_src_over(tjs_uint32 dst, tjs_uint32 src) {
    // dst' = src + dst × (1 - src_alpha)
    tjs_uint32 s_alpha = (~src) >> 24;  // 255 - src_alpha（取反后的高8位）
    // dst 各通道乘以 (1-src_alpha)
    tjs_uint32 d_rb = ((dst & 0xff00ff) * s_alpha >> 8) & 0xff00ff;
    tjs_uint32 d_g  = ((dst & 0x00ff00) * s_alpha >> 8) & 0x00ff00;
    // 饱和加法: src + d_scaled
    return saturated_add(src, d_rb | d_g);
}
```

---

## 4. KrKr2 的 30+ 混合模式

### 4.1 混合模式枚举

KrKr2 定义了完整的混合模式枚举（`LayerBitmapIntf.h` 第 28-61 行）：

```cpp
enum tTVPBBBltMethod {
    bmCopy,                // 直接复制（忽略 alpha）
    bmCopyOnAlpha,         // 复制到 alpha 图层
    bmAlpha,               // 标准 Alpha 混合 (SrcOver)
    bmAlphaOnAlpha,        // Alpha 混合到 alpha 图层
    bmAdd,                 // 加算合成（饱和加法）
    bmSub,                 // 减算合成
    bmMul,                 // 乘算合成
    bmDodge,               // 覆い焼き（Color Dodge 简化版）
    bmDarken,              // 比较暗
    bmLighten,             // 比较亮
    bmScreen,              // 滤色
    bmAddAlpha,            // Premultiplied Alpha 混合
    bmAddAlphaOnAddAlpha,  // Premul → Premul 图层
    bmAddAlphaOnAlpha,     // Premul → Straight 图层
    bmAlphaOnAddAlpha,     // Straight → Premul 图层
    bmCopyOnAddAlpha,      // 复制 → Premul 图层
    // Photoshop 兼容混合模式
    bmPsNormal,            // PS 正常
    bmPsAdditive,          // PS 线性减淡（加法）
    bmPsSubtractive,       // PS 线性加深（减法）
    bmPsMultiplicative,    // PS 乘法
    bmPsScreen,            // PS 滤色
    bmPsOverlay,           // PS 叠加
    bmPsHardLight,         // PS 强光
    bmPsSoftLight,         // PS 柔光
    bmPsColorDodge,        // PS 颜色减淡
    bmPsColorDodge5,       // PS 5.x 兼容颜色减淡
    bmPsColorBurn,         // PS 焼き込み（颜色加深）
    bmPsLighten,           // PS 变亮
    bmPsDarken,            // PS 变暗
    bmPsDifference,        // PS 差值
    bmPsDifference5,       // PS 5.x 兼容差值
    bmPsExclusion          // PS 排除
};
```

### 4.2 分类与数学公式

这些模式可以分为几大类：

**基础类（位运算优化）**：

| 模式 | 公式 | 说明 |
|------|------|------|
| bmCopy | D = S | 直接覆盖 |
| bmAlpha | D = S×As + D×(1-As) | 标准 SrcOver |
| bmAdd | D = min(D+S, 255) | 饱和加法，使画面变亮 |
| bmSub | D = max(D+S-255, 0) | 饱和减法，使画面变暗 |
| bmMul | D = D×S/255 | 乘法，两暗色叠加更暗 |
| bmScreen | D = 1-(1-D)(1-S) | 滤色，两亮色叠加更亮 |

**Photoshop 兼容类（查找表优化）**：

KrKr2 中 PS 系列使用 256×256 查找表（`blend_functor_c.h` 第 461-479 行）：

```cpp
// 256×256 的预计算表，避免运行时除法
struct ps_soft_light_table {
    static unsigned char TABLE[256][256];  // TABLE[src][dst] = result
};
struct ps_color_dodge_table {
    static unsigned char TABLE[256][256];
};
struct ps_color_burn_table {
    static unsigned char TABLE[256][256];
};
```

柔光、颜色减淡、颜色加深这三种模式包含除法运算，直接计算开销大，所以用预计算的 64KB 查找表（256×256 字节）替代。

### 4.3 Functor + 宏展开架构

KrKr2 用 C++ 模板 functor（仿函数）定义每种混合的核心算法，再用宏自动生成所有变体：

```cpp
// 步骤1: 定义核心混合仿函数（只管 RGB，不管 alpha 策略）
struct mul_blend_functor {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        // 逐通道乘法: R = Rd × Rs / 255, G = ..., B = ...
        tjs_uint32 tmp = (d & 0xff) * (s & 0xff) & 0xff00;
        tmp |= ((d & 0xff00) >> 8) * (s & 0xff00) & 0xff0000;
        tmp |= ((d & 0xff0000) >> 16) * (s & 0xff0000) & 0xff000000;
        return tmp >> 8;
    }
};

// 步骤2: 宏自动展开 6 种变体
DEFINE_BLEND_VARIATION(mul_blend)
// 展开为:
//   mul_blend_functor          — 基本版（无 alpha）
//   mul_blend_o_functor        — 带 opacity
//   mul_blend_HDA_functor      — 保持 dst alpha
//   mul_blend_HDA_o_functor    — 保持 dst alpha + opacity
//   mul_blend_d_functor        — dst 有独立 alpha
//   mul_blend_do_functor       — dst 有独立 alpha + opacity
```

变体后缀说明：

| 后缀 | 含义 | 场景 |
|------|------|------|
| (无) | 基本混合 | dst alpha 被忽略 |
| `_o` | + opacity | 图层有全局不透明度 |
| `_HDA` | Hold Dest Alpha | 保持 dst alpha 不被修改 |
| `_HDA_o` | HDA + opacity | 两者结合 |
| `_d` | Dest Alpha | dst 有独立 alpha 通道参与运算 |
| `_do` | Dest Alpha + opacity | 两者结合 |
| `_a` | Additive Alpha (src) | src 是 premultiplied alpha |
| `_ao` | Additive Alpha + opacity | premul src + 全局不透明度 |

---

## 5. 查找表（LUT）优化

### 5.1 除法表 TVPDivTable

整数除以 255 是 Alpha 混合中最耗时的操作之一。KrKr2 用查找表避免除法：

```cpp
// 文件: cpp/core/visual/tvpgl.cpp, 第 13-15 行
tjs_uint32 TVPDivTable[256]; // TVPDivTable[x] ≈ (1/x) × 65536

// 初始化（TVPCreateTable 函数中）:
for(int i = 1; i < 256; i++)
    TVPDivTable[i] = 65536 / i;
TVPDivTable[0] = 65536;  // 防止除零
```

### 5.2 不透明度合成表

两层半透明叠加后的总不透明度需要计算 `1 - (1-a1)(1-a2)`。KrKr2 预计算了这张 256×256 表：

```cpp
// 文件: cpp/core/visual/tvpgl.cpp
unsigned char TVPOpacityOnOpacityTable[256 * 256];

// 初始化:
for(int a1 = 0; a1 < 256; a1++) {
    for(int a2 = 0; a2 < 256; a2++) {
        // 结果 = 1 - (1-a1/255)(1-a2/255) 映射到 [0,255]
        float result = 1.0f - (1.0f - a1/255.0f) * (1.0f - a2/255.0f);
        TVPOpacityOnOpacityTable[a1 * 256 + a2] = (unsigned char)(result * 255);
    }
}
```

### 5.3 负乘表

```cpp
// TVPNegativeMulTable[a][b] = 255 - (255-a)(255-b)/255
// 等价于 Screen 混合的单通道结果
unsigned char TVPNegativeMulTable[256 * 256];
```

### 5.4 倒数表

```cpp
// 文件: cpp/core/visual/tvpgl.cpp
tjs_uint32 TVPRecipTable256[256];
// TVPRecipTable256[x] = 65536 / x（256 级精度的倒数表）
// 用于 Color Dodge 混合: result = dst / (1 - src)
//   = dst * TVPRecipTable256[255 - src] >> 8
```

这就是为什么 `color_dodge_blend_functor` 中使用 `TVPRecipTable256`（`blend_functor_c.h` 第 246 行）：

```cpp
struct color_dodge_blend_functor {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        tjs_uint32 tmp2 = ~s;  // 255 - s（各通道）
        // R 通道: dst_r * (65536 / (255-src_r)) >> 8
        tjs_uint32 tmp = (d & 0xff) * TVPRecipTable256[tmp2 & 0xff] >> 8;
        // 饱和到 255（用带符号比较技巧）
        tjs_uint32 tmp3 = (tmp | ((tjs_int32)~(tmp - 0x100) >> 31)) & 0xff;
        // G、B 通道类似...
        return tmp3;
    }
};
```

---

## 6. 性能权重因子

KrKr2 为每种混合模式分配了性能权重因子（`LayerBitmapIntf.cpp` 第 46-79 行），用于多线程调度时估算工作量：

```cpp
const static float sBmFactor[] = {
    59, // bmCopy         — 最简单，仅复制
    52, // bmAlpha        — 标准混合
    61, // bmAdd          — 饱和加法
    45, // bmMul          — 逐通道乘法
    10, // bmDodge        — 需要除法/查表，最慢
    10, // bmPsSoftLight  — 需要 256×256 查表
    10, // bmPsColorDodge — 需要 256×256 查表
    66, // bmPsExclusion  — 公式简单，最快的 PS 模式
};
```

数值越大表示单位时间能处理越多像素（即越快）。`bmDodge`/`bmPsSoftLight` 等需要查表的模式只有 10，而简单的 `bmPsExclusion` 高达 66。引擎据此决定是否为该混合操作启用多线程。

---

## 动手实践

### 练习：实现一个迷你混合器

创建一个程序，支持 bmAlpha、bmAdd、bmMul、bmScreen 四种模式：

```cpp
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <algorithm>

using tjs_uint32 = uint32_t;

// 饱和加法（不使用分支）
tjs_uint32 saturated_add(tjs_uint32 a, tjs_uint32 b) {
    tjs_uint32 tmp = ((a & b) + (((a ^ b) >> 1) & 0x7f7f7f7f)) & 0x80808080;
    tmp = (tmp << 1) - (tmp >> 7);  // 溢出位扩展为 0xFF 掩码
    return (a + b - tmp) | tmp;     // 溢出处饱和到 255
}

// bmAlpha: 标准 SrcOver
tjs_uint32 blend_alpha(tjs_uint32 d, tjs_uint32 s) {
    tjs_uint32 a = s >> 24;
    tjs_uint32 d1 = d & 0xff00ff;
    d1 = (d1 + (((s & 0xff00ff) - d1) * a >> 8)) & 0xff00ff;
    tjs_uint32 d2 = d & 0xff00;
    tjs_uint32 s2 = s & 0xff00;
    return d1 + ((d2 + ((s2 - d2) * a >> 8)) & 0xff00);
}

// bmAdd: 饱和加法（源 alpha 调制后相加）
tjs_uint32 blend_add(tjs_uint32 d, tjs_uint32 s) {
    tjs_uint32 a = s >> 24;
    // 源颜色乘以源 alpha
    tjs_uint32 s_mod = (((s & 0x00ff00) * a >> 8) & 0x00ff00) +
                        (((s & 0xff00ff) * a >> 8) & 0xff00ff);
    return saturated_add(d, s_mod);
}

// bmMul: 逐通道乘法
tjs_uint32 blend_mul(tjs_uint32 d, tjs_uint32 s) {
    tjs_uint32 r = ((d & 0xff0000) >> 16) * ((s & 0xff0000) >> 16) / 255;
    tjs_uint32 g = ((d & 0x00ff00) >>  8) * ((s & 0x00ff00) >>  8) / 255;
    tjs_uint32 b = ((d & 0x0000ff)      ) * ((s & 0x0000ff)      ) / 255;
    return (r << 16) | (g << 8) | b;
}

// bmScreen: 1 - (1-D)(1-S)
tjs_uint32 blend_screen(tjs_uint32 d, tjs_uint32 s) {
    tjs_uint32 id = ~d, is = ~s;  // 取反
    tjs_uint32 r = ((id & 0xff0000) >> 16) * ((is & 0xff0000) >> 16) / 255;
    tjs_uint32 g = ((id & 0x00ff00) >>  8) * ((is & 0x00ff00) >>  8) / 255;
    tjs_uint32 b = ((id & 0x0000ff)      ) * ((is & 0x0000ff)      ) / 255;
    return ~((r << 16) | (g << 8) | b) & 0x00ffffff;
}

void print_pixel(const char* label, tjs_uint32 p) {
    printf("%s: A=%3d R=%3d G=%3d B=%3d (0x%08X)\n", label,
           (p>>24)&0xff, (p>>16)&0xff, (p>>8)&0xff, p&0xff, p);
}

int main() {
    tjs_uint32 dst = 0xFF804020;  // 不透明的暗橙色
    tjs_uint32 src = 0x80FF0000;  // 半透明红色 (alpha=128)

    print_pixel("DST     ", dst);
    print_pixel("SRC     ", src);
    print_pixel("bmAlpha ", blend_alpha(dst, src));
    print_pixel("bmAdd   ", blend_add(dst, src));
    print_pixel("bmMul   ", blend_mul(dst, src));
    print_pixel("bmScreen", blend_screen(dst, src));

    return 0;
}
```

**CMakeLists.txt**：

```cmake
cmake_minimum_required(VERSION 3.16)
project(blend_demo LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
add_executable(blend_demo main.cpp)
```

编译运行：

```bash
# Windows
cmake -B build -G Ninja && cmake --build build && build\blend_demo.exe

# Linux / macOS
cmake -B build -G Ninja && cmake --build build && ./build/blend_demo
```

预期输出：

```
DST     : A=255 R=128 G= 64 B= 32 (0xFF804020)
SRC     : A=128 R=255 G=  0 B=  0 (0x80FF0000)
bmAlpha : A=  0 R=192 G= 32 B= 16 (0x00C02010)
bmAdd   : A=  0 R=255 G= 64 B= 32 (0x00FF4020)
bmMul   : A=  0 R=128 G=  0 B=  0 (0x00800000)
bmScreen: A=  0 R=255 G= 64 B= 32 (0x00FF4020)
```

---

## 对照项目源码

以下是 KrKr2 Alpha 混合相关的核心源文件：

| 文件 | 行数 | 内容 |
|------|------|------|
| `cpp/core/visual/gl/blend_functor_c.h` | 1027 | 所有混合模式的 functor 实现 |
| `cpp/core/visual/gl/blend_function.cpp` | 685 | 宏展开生成所有混合函数变体 |
| `cpp/core/visual/gl/blend_variation.h` | — | `normal_op`/`translucent_op`/`hda_op` 等模板定义 |
| `cpp/core/visual/gl/blend_util_func.h` | — | 基础混合工具函数 |
| `cpp/core/visual/tvpgl.h` | 1107 | 函数指针声明（运行时分发） |
| `cpp/core/visual/tvpgl.cpp` | 14137 | 查找表初始化 + 默认 C 实现 |
| `cpp/core/visual/LayerBitmapIntf.h` | 275 | `tTVPBBBltMethod` 枚举定义 |
| `cpp/core/visual/LayerBitmapIntf.cpp` | 4820 | 混合模式调度 + 性能权重表 |

**阅读建议**：从 `blend_functor_c.h` 的 `alpha_blend_func`（第 67 行）开始，理解最基础的混合，然后看 `DEFINE_BLEND_VARIATION` 宏如何展开变体，最后看 `blend_function.cpp` 的 `DEFINE_BLEND_FUNCTION_VARIATION` 宏如何把 functor 包装成 C 风格函数指针。

---

## 本节小结

- Alpha 通道表示不透明度，有 **Straight** 和 **Premultiplied** 两种存储约定
- 标准 Alpha 混合（SrcOver）是最常用的合成操作，KrKr2 用**双通道并行位运算**优化
- Porter-Duff 定义了 12 种合成模式，其中 SrcOver 最常用
- KrKr2 定义了 31 种混合模式，包括基础类（位运算）和 Photoshop 兼容类（查找表）
- **Functor + 宏展开**架构：核心算法只写一次，宏自动生成 6-8 种变体（_o, _HDA, _d, _a 等）
- **查找表优化**：`TVPRecipTable256`（除法）、`TVPOpacityOnOpacityTable`（透明度合成）、`ps_*_table`（PS 混合）避免昂贵的运行时除法
- **性能权重 `sBmFactor`**：引擎根据混合模式复杂度决定是否多线程并行

---

## 练习题与答案

### 题目 1：Straight 与 Premultiplied 的区别

给定一个半透明红色像素（RGBA = 255, 0, 0, 128）：

(a) 在 Straight Alpha 格式下，32 位 ARGB 值是多少？
(b) 在 Premultiplied Alpha 格式下，32 位 ARGB 值是多少？
(c) 如果将 Premultiplied 格式的像素错误地当作 Straight 格式进行 Alpha 混合，画面会出现什么问题？

<details>
<summary>查看答案</summary>

**(a)** Straight Alpha: `0x80FF0000`
- A=128 (0x80), R=255 (0xFF), G=0, B=0

**(b)** Premultiplied Alpha: `0x80800000`
- A=128 (0x80)
- R = 255 × (128/255) ≈ 128 (0x80)
- G = 0 × (128/255) = 0
- B = 0 × (128/255) = 0

**(c)** 画面会出现颜色偏暗/偏灰的问题。因为预乘格式的 R=128 会被再乘一次 Alpha（128/255），变成 R≈64，相当于 Alpha 被重复应用了两次，导致半透明对象比预期更加透明。这在游戏中表现为精灵边缘出现深色"光晕"。

</details>

### 题目 2：实现饱和减法

KrKr2 的 `sub_blend_functor` 使用纯位运算实现了饱和减法（`max(D+S-255, 0)` 的等价运算）。请用更易读的方式实现同样功能，然后对比 KrKr2 的位运算版本，分析后者的优势。

<details>
<summary>查看答案</summary>

```cpp
// 易读版本（使用分支）
tjs_uint32 sub_blend_readable(tjs_uint32 d, tjs_uint32 s) {
    int r = std::max(0, (int)((d>>16)&0xff) + (int)((s>>16)&0xff) - 255);
    int g = std::max(0, (int)((d>>8)&0xff)  + (int)((s>>8)&0xff)  - 255);
    int b = std::max(0, (int)(d&0xff)       + (int)(s&0xff)       - 255);
    return (r << 16) | (g << 8) | b;
}

// KrKr2 的位运算版本 (blend_functor_c.h 第 202-207 行):
struct sub_blend_functor {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        tjs_uint32 tmp = ((s & d) + (((s ^ d) >> 1) & 0x7f7f7f7f)) & 0x80808080;
        tmp = (tmp << 1) - (tmp >> 7);
        return (s + d - tmp) & tmp;
    }
};
```

**KrKr2 版本的优势：**
1. **零分支**：没有 if/max，不会导致 CPU 分支预测失败
2. **四通道并行**：一条表达式同时处理 A/R/G/B 四个字节
3. **SIMD 友好**：无分支的位运算能被编译器自动向量化
4. 易读版本需要 12 次移位/掩码 + 3 次 max 比较 + 3 次解包/打包；位运算版只需约 8 条运算

</details>

### 题目 3：查找表设计

假设你需要为一个自定义混合模式实现 256×256 的查找表，公式为 `result = sqrt(src × dst / 255.0) × 255`。

(a) 写出表的初始化代码
(b) 写出使用该表的混合 functor
(c) 该表占用多少内存？

<details>
<summary>查看答案</summary>

```cpp
#include <cmath>
#include <cstdint>

// (a) 初始化
unsigned char my_blend_table[256][256];

void init_table() {
    for (int s = 0; s < 256; s++) {
        for (int d = 0; d < 256; d++) {
            double val = std::sqrt(s * d / 255.0) * 255.0;
            my_blend_table[s][d] = (unsigned char)std::min(255.0, val + 0.5);
        }
    }
}

// (b) 使用表的 functor
struct my_blend_func {
    inline uint32_t operator()(uint32_t d, uint32_t s, uint32_t a) const {
        uint32_t result =
            (my_blend_table[(s >> 16) & 0xff][(d >> 16) & 0xff] << 16) |
            (my_blend_table[(s >>  8) & 0xff][(d >>  8) & 0xff] <<  8) |
            (my_blend_table[(s >>  0) & 0xff][(d >>  0) & 0xff] <<  0);
        // 与 alpha 混合（参考 ps_alpha_blend_func 的做法）
        uint32_t d1 = d & 0x00ff00ff;
        uint32_t d2 = d & 0x0000ff00;
        return ((((((result & 0x00ff00ff) - d1) * a) >> 8) + d1) & 0x00ff00ff) |
               ((((((result & 0x0000ff00) - d2) * a) >> 8) + d2) & 0x0000ff00);
    }
};
```

**(c)** 内存占用 = 256 × 256 × 1 字节 = **65,536 字节 = 64 KB**。这在现代系统中微不足道，但刚好能放进 L1 缓存（通常 32-64KB），所以查表的性能非常好。KrKr2 的 `ps_soft_light_table`、`ps_color_dodge_table`、`ps_color_burn_table` 各占 64KB，共 192KB。

</details>

---

## 下一步

下一节 [图像缩放与插值](../03-图像缩放与插值/01-图像缩放与插值.md)，我们将学习如何对像素图像进行高质量的缩放，包括最近邻、双线性、双三次、Lanczos 等插值算法，以及 KrKr2 中 `ResampleImage.cpp` 的 SIMD 加速实现。
