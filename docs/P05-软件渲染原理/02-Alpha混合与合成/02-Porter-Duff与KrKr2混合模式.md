# Porter-Duff 与 KrKr2 混合模式

> **所属模块：** P05-软件渲染原理
> **前置知识：** [01-Alpha通道与标准混合](./01-Alpha通道与标准混合.md)
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 Porter-Duff 12 种合成模式的数学定义，说出每种模式保留/丢弃了哪些像素区域
2. 看懂 KrKr2 引擎 31 种混合模式（`tTVPBBBltMethod` 枚举）的完整列表和分类逻辑
3. 理解 Functor + 宏展开架构——如何用一个核心 functor 自动生成 6~8 种变体
4. 掌握查找表（LUT，Look-Up Table，预计算的结果表用于替代运行时计算）优化除法的原理
5. 理解性能权重因子（`sBmFactor`）在多线程调度中的作用

---

## 1. Porter-Duff 合成模式

### 1.1 历史背景

1984 年，Thomas Porter 和 Tom Duff 在论文"Compositing Digital Images"中提出了 12 种像素合成运算。这篇论文奠定了现代 2D 图形合成的理论基础——从 Photoshop 到浏览器的 CSS、从 Android 的 `Canvas` 到 iOS 的 `CGBlendMode`，所有 2D 图形系统的图层合成都基于这套理论。

### 1.2 四区域模型

对于源像素（Source，前景）和目标像素（Destination，背景），Porter-Duff 模型将合成区域分为四个部分：

```
┌─────────────────────────────────┐
│                                 │
│   ┌───────────────┐             │
│   │               │             │
│   │   S ∩ D       │  S only     │
│   │  (两者重叠)    │ (只有源)    │
│   │               │             │
│   ├───────────────┤             │
│   │               │             │
│   │   D only      │  Neither    │
│   │  (只有目标)    │ (都没有)    │
│   │               │             │
│   └───────────────┘             │
│                                 │
└─────────────────────────────────┘
```

- **S ∩ D**：源和目标都有内容的区域
- **S only**：只有源有内容的区域
- **D only**：只有目标有内容的区域
- **Neither**：两者都没有内容的区域

12 种模式的区别在于：**保留哪些区域的颜色，丢弃哪些**。

### 1.3 完整 12 种模式

| 模式 | 保留区域 | 颜色公式 | Alpha 公式 | 直觉说明 |
|------|----------|----------|------------|----------|
| **Clear** | 无 | Co=0 | Ao=0 | 清空——全部变透明 |
| **Src** | S | Co=Cs | Ao=As | 只显示源，无视目标 |
| **Dst** | D | Co=Cd | Ao=Ad | 只显示目标，无视源 |
| **SrcOver** | S∩D + S + D | Co=Cs + Cd(1-As) | Ao=As + Ad(1-As) | 源叠加在目标上（最常用） |
| **DstOver** | S∩D + S + D | Co=Cs(1-Ad) + Cd | Ao=As(1-Ad) + Ad | 目标叠加在源上（反向叠加） |
| **SrcIn** | S∩D (源色) | Co=Cs·Ad | Ao=As·Ad | 源只在目标不透明处可见 |
| **DstIn** | S∩D (目标色) | Co=Cd·As | Ao=Ad·As | 目标只在源不透明处可见 |
| **SrcOut** | S only | Co=Cs(1-Ad) | Ao=As(1-Ad) | 源只在目标透明处可见 |
| **DstOut** | D only | Co=Cd(1-As) | Ao=Ad(1-As) | 目标被源"挖去"（蒙版裁剪） |
| **SrcAtop** | S∩D(源) + D | Co=Cs·Ad + Cd(1-As) | Ao=Ad | 源贴在目标上，但不扩展边界 |
| **DstAtop** | S∩D(目标) + S | Co=Cs(1-Ad) + Cd·As | Ao=As | 目标贴在源上，但不扩展边界 |
| **Xor** | S only + D only | Co=Cs(1-Ad) + Cd(1-As) | Ao=As(1-Ad) + Ad(1-As) | 只保留非重叠部分 |

> **符号说明**：Cs = 源颜色，Cd = 目标颜色，As = 源 Alpha，Ad = 目标 Alpha，Co = 输出颜色，Ao = 输出 Alpha。

### 1.4 SrcOver：最常用的模式

KrKr2 的 `bmAlpha` 模式对应 **SrcOver**，是游戏引擎中使用频率最高的合成方式。上一节已详细讲解了它的 Straight Alpha 实现，这里补充 **Premultiplied Alpha**（预乘Alpha，颜色通道已预先乘以Alpha值）版本：

```cpp
// Premultiplied Alpha 的 SrcOver
// 公式: dst' = src + dst × (1 - src_alpha)
// 在预乘格式下，src 的颜色已经包含了 alpha，所以直接相加
tjs_uint32 premul_src_over(tjs_uint32 dst, tjs_uint32 src) {
    tjs_uint32 s_alpha = (~src) >> 24;  // 255 - src_alpha（位取反后取高8位）
    // dst 各通道乘以 (1 - src_alpha)
    tjs_uint32 d_rb = ((dst & 0xff00ff) * s_alpha >> 8) & 0xff00ff;
    tjs_uint32 d_g  = ((dst & 0x00ff00) * s_alpha >> 8) & 0x00ff00;
    // src + dst_scaled（需要饱和加法防止溢出）
    return saturated_add(src, d_rb | d_g);
}
```

预乘版本的优势：**公式只有一次乘法 + 一次加法**，而 Straight Alpha 版本需要两次乘法。这就是为什么 Photoshop、浏览器等高性能图形系统内部都使用预乘格式。

### 1.5 DstOut：实现蒙版裁剪

DstOut 模式有一个非常实用的应用——**蒙版裁剪**（Mask Clipping）。它的公式 `Co = Cd × (1 - As)` 意味着：源越不透明的地方，目标越被"擦除"。

```cpp
// DstOut 实现
tjs_uint32 dst_out(tjs_uint32 dst, tjs_uint32 src) {
    tjs_uint32 inv_a = 255 - (src >> 24);  // 1 - 源alpha
    tjs_uint32 d_rb = ((dst & 0xff00ff) * inv_a >> 8) & 0xff00ff;
    tjs_uint32 d_g  = ((dst & 0x00ff00) * inv_a >> 8) & 0x00ff00;
    tjs_uint32 d_a  = ((dst >> 24) * inv_a >> 8) << 24;
    return d_rb | d_g | d_a;
}
```

在视觉小说中，DstOut 常用于实现"文字淡出"效果——用一个渐变的蒙版图层叠加到文字图层上，文字就会渐渐消失。

### 1.6 完整可运行示例：Porter-Duff 模式对比

```cpp
#include <cstdint>
#include <cstdio>
#include <algorithm>

using tjs_uint32 = uint32_t;

// 通用 Porter-Duff 合成（逐通道，便于理解）
struct PorterDuff {
    // 系数: result = src * Fs + dst * Fd
    // 各模式只是 Fs 和 Fd 不同
    static tjs_uint32 compose(tjs_uint32 dst, tjs_uint32 src,
                               float Fs, float Fd) {
        auto ch = [](tjs_uint32 p, int shift) -> float {
            return ((p >> shift) & 0xff) / 255.0f;
        };
        auto out = [&](int shift) -> uint8_t {
            float s = ch(src, shift), d = ch(dst, shift);
            float r = s * Fs + d * Fd;
            return (uint8_t)(std::min(1.0f, std::max(0.0f, r)) * 255 + 0.5f);
        };
        return (out(24) << 24) | (out(16) << 16) | (out(8) << 8) | out(0);
    }

    static tjs_uint32 src_over(tjs_uint32 d, tjs_uint32 s) {
        float as = (s >> 24) / 255.0f;
        return compose(d, s, 1.0f, 1.0f - as);
    }
    static tjs_uint32 dst_over(tjs_uint32 d, tjs_uint32 s) {
        float ad = (d >> 24) / 255.0f;
        return compose(d, s, 1.0f - ad, 1.0f);
    }
    static tjs_uint32 src_in(tjs_uint32 d, tjs_uint32 s) {
        float ad = (d >> 24) / 255.0f;
        return compose(d, s, ad, 0.0f);
    }
    static tjs_uint32 dst_out(tjs_uint32 d, tjs_uint32 s) {
        float as = (s >> 24) / 255.0f;
        return compose(d, s, 0.0f, 1.0f - as);
    }
    static tjs_uint32 xor_mode(tjs_uint32 d, tjs_uint32 s) {
        float as = (s >> 24) / 255.0f;
        float ad = (d >> 24) / 255.0f;
        return compose(d, s, 1.0f - ad, 1.0f - as);
    }
};

void print_pixel(const char* label, tjs_uint32 p) {
    printf("%-12s: A=%3d R=%3d G=%3d B=%3d\n", label,
           (p>>24)&0xff, (p>>16)&0xff, (p>>8)&0xff, p&0xff);
}

int main() {
    tjs_uint32 dst = 0xC0006400;  // 75% 不透明深绿
    tjs_uint32 src = 0x80FF0000;  // 50% 不透明红色

    printf("=== Porter-Duff 合成模式对比 ===\n");
    print_pixel("DST", dst);
    print_pixel("SRC", src);
    printf("\n");

    print_pixel("SrcOver", PorterDuff::src_over(dst, src));
    print_pixel("DstOver", PorterDuff::dst_over(dst, src));
    print_pixel("SrcIn",   PorterDuff::src_in(dst, src));
    print_pixel("DstOut",  PorterDuff::dst_out(dst, src));
    print_pixel("Xor",     PorterDuff::xor_mode(dst, src));

    return 0;
}
```

**CMakeLists.txt**：

```cmake
cmake_minimum_required(VERSION 3.16)
project(porter_duff_demo LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
add_executable(porter_duff_demo main.cpp)
```

---

## 2. KrKr2 的 31 种混合模式

### 2.1 完整枚举

KrKr2 在 `LayerBitmapIntf.h` 第 28-61 行定义了完整的混合模式枚举：

```cpp
// 文件: cpp/core/visual/LayerBitmapIntf.h, 第 28-61 行
enum tTVPBBBltMethod {
    // === 基础类（位运算优化） ===
    bmCopy,                // 直接复制（忽略 alpha，D = S）
    bmCopyOnAlpha,         // 复制到带 alpha 的图层
    bmAlpha,               // 标准 Alpha 混合（对应 Porter-Duff SrcOver）
    bmAlphaOnAlpha,        // Alpha 混合，目标有独立 alpha
    bmAdd,                 // 加算合成（饱和加法，min(D+S, 255)）
    bmSub,                 // 减算合成（饱和减法，max(D+S-255, 0)）
    bmMul,                 // 乘算合成（D×S/255，使暗色更暗）
    bmDodge,               // 覆い焼き（Color Dodge 简化版）
    bmDarken,              // 比较暗（取 D 和 S 中较暗的）
    bmLighten,             // 比较亮（取 D 和 S 中较亮的）
    bmScreen,              // 滤色（1-(1-D)(1-S)，使亮色更亮）

    // === Alpha 格式转换类 ===
    bmAddAlpha,            // Premultiplied Alpha 混合（预乘→普通目标）
    bmAddAlphaOnAddAlpha,  // Premul 源 → Premul 目标
    bmAddAlphaOnAlpha,     // Premul 源 → Straight 目标
    bmAlphaOnAddAlpha,     // Straight 源 → Premul 目标
    bmCopyOnAddAlpha,      // 直接复制 → Premul 目标

    // === Photoshop 兼容混合模式（查找表优化） ===
    bmPsNormal,            // PS 正常（等同 bmAlpha）
    bmPsAdditive,          // PS 线性减淡/加法（Linear Dodge）
    bmPsSubtractive,       // PS 线性加深/减法（Linear Burn）
    bmPsMultiplicative,    // PS 乘法（Multiply）
    bmPsScreen,            // PS 滤色（Screen）
    bmPsOverlay,           // PS 叠加（Overlay）
    bmPsHardLight,         // PS 强光（Hard Light）
    bmPsSoftLight,         // PS 柔光（Soft Light）
    bmPsColorDodge,        // PS 颜色减淡（Color Dodge）
    bmPsColorDodge5,       // PS 5.x 兼容颜色减淡
    bmPsColorBurn,         // PS 颜色加深（Color Burn）
    bmPsLighten,           // PS 变亮（Lighten）
    bmPsDarken,            // PS 变暗（Darken）
    bmPsDifference,        // PS 差值（Difference）
    bmPsDifference5,       // PS 5.x 兼容差值
    bmPsExclusion          // PS 排除（Exclusion）
};
```

### 2.2 分类与数学公式

这 31 种模式可以分为三大类：

**第一类：基础算术混合**

| 模式 | 公式 | 说明 | 性能 |
|------|------|------|------|
| bmCopy | D = S | 直接内存复制 | 最快 |
| bmAlpha | D = S×As + D×(1-As) | SrcOver，标准图层叠加 | 快 |
| bmAdd | D = min(D+S, 255) | 饱和加法，画面整体变亮 | 快 |
| bmSub | D = max(D+S-255, 0) | 饱和减法，画面整体变暗 | 快 |
| bmMul | D = D×S/255 | 乘法，暗色叠暗色更暗 | 中等 |
| bmScreen | D = 1-(1-D)(1-S) | 滤色（Multiply 的反转），亮色叠亮色更亮 | 中等 |
| bmDarken | D = min(D, S) | 取较暗值 | 快 |
| bmLighten | D = max(D, S) | 取较亮值 | 快 |

**第二类：Photoshop 兼容混合**

这些模式的公式更复杂，包含除法或条件分支：

| 模式 | 公式 | 说明 |
|------|------|------|
| bmPsOverlay | D<128: 2DS/255; D≥128: 255-2(255-D)(255-S)/255 | 暗部用乘法、亮部用滤色 |
| bmPsHardLight | S<128: 2DS/255; S≥128: 255-2(255-D)(255-S)/255 | Overlay 的镜像（以 S 判断） |
| bmPsSoftLight | 复杂多段函数 | 柔和的光照效果 |
| bmPsColorDodge | D / (1-S) | 颜色减淡（使亮部更亮） |
| bmPsColorBurn | 1 - (1-D) / S | 颜色加深（使暗部更暗） |
| bmPsDifference | \|D-S\| | 差值（用于对比两个图层） |
| bmPsExclusion | D+S-2DS/255 | 类似差值但对比度更低 |

**第三类：Alpha 格式转换**

处理 Straight Alpha 和 Premultiplied Alpha 之间的互操作，在上一节已介绍。

### 2.3 Overlay 模式详解

Overlay 是视觉小说中常用的光影效果模式。它的特殊之处在于**根据目标亮度切换算法**：

```cpp
// Overlay: 暗部用乘法加深，亮部用滤色提亮
struct overlay_blend_functor {
    inline uint8_t blend_ch(uint8_t d, uint8_t s) const {
        if (d < 128) {
            // 暗部：乘法（使更暗）
            return (uint8_t)(2 * d * s / 255);
        } else {
            // 亮部：滤色（使更亮）
            return (uint8_t)(255 - 2 * (255 - d) * (255 - s) / 255);
        }
    }
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        uint8_t r = blend_ch((d >> 16) & 0xff, (s >> 16) & 0xff);
        uint8_t g = blend_ch((d >> 8) & 0xff, (s >> 8) & 0xff);
        uint8_t b = blend_ch(d & 0xff, s & 0xff);
        return (r << 16) | (g << 8) | b;
    }
};
```

在视觉小说中，Overlay 常用于：
- 给场景添加光照贴图（明亮区域更亮，阴影区域更暗）
- 给角色立绘添加环境光效果（根据背景明暗调整角色明暗）

---

## 3. Functor + 宏展开架构

### 3.1 设计问题

KrKr2 有 31 种混合模式，每种还需要多个变体（带/不带 opacity、带/不带 HDA、straight/premultiplied alpha）。如果每种组合都单独写一个函数，需要 31×8 = 248 个函数——这显然不可维护。

KrKr2 的解决方案：**核心算法只写一次（functor），变体通过模板 + 宏自动生成**。

### 3.2 核心 Functor

每种混合模式定义一个 functor（仿函数），只负责**最核心的颜色运算**，不关心 alpha 策略、opacity、HDA 等：

```cpp
// 文件: cpp/core/visual/gl/blend_functor_c.h
// 示例：乘法混合的核心 functor
struct mul_blend_functor {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        // 逐通道乘法: result = D × S / 255
        tjs_uint32 tmp = (d & 0xff) * (s & 0xff) & 0xff00;      // B
        tmp |= ((d & 0xff00) >> 8) * (s & 0xff00) & 0xff0000;   // G
        tmp |= ((d >> 16) & 0xff) * ((s >> 16) & 0xff) << 8     // R
               & 0xff000000;
        return tmp >> 8;
    }
};
```

### 3.3 变体模板

上一节已介绍的 `translucent_op`（opacity）、`hda_op`（HDA）等模板，用于包装任意 functor：

```cpp
// blend_variation.h 中的模板（简化）

// 基本版：直接使用像素 alpha
template<typename FUNC>
struct normal_op {
    FUNC func_;
    tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        tjs_uint32 a = s >> 24;  // 提取像素 alpha
        // 先用 functor 计算混合颜色，再与 alpha 线性插值
        tjs_uint32 blended = func_(d, s);
        return alpha_blend(d, blended, a);  // 按 alpha 混合
    }
};

// opacity 版：alpha × opacity
template<typename FUNC>
struct translucent_op {
    FUNC func_;
    tjs_int opa_;
    tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        tjs_uint32 a = (s >> 24) * opa_ >> 8;
        tjs_uint32 blended = func_(d, s);
        return alpha_blend(d, blended, a);
    }
};

// HDA 版：保持 dst alpha
template<typename FUNC>
struct hda_op {
    FUNC func_;
    tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        tjs_uint32 result = func_(d, s);
        return (result & 0x00ffffff) | (d & 0xff000000);
    }
};
```

### 3.4 宏展开

`DEFINE_BLEND_VARIATION` 宏为一个 functor 自动生成所有变体：

```cpp
// 文件: cpp/core/visual/gl/blend_function.cpp（简化）
#define DEFINE_BLEND_VARIATION(NAME) \
    /* 基本版 */                                                     \
    void TVP##NAME(tjs_uint32 *d, const tjs_uint32 *s, tjs_int len) { \
        normal_op<NAME##_functor> op;                                 \
        for (tjs_int i = 0; i < len; i++) d[i] = op(d[i], s[i]);    \
    }                                                                 \
    /* opacity 版 */                                                  \
    void TVP##NAME##_o(tjs_uint32 *d, const tjs_uint32 *s,           \
                       tjs_int len, tjs_int opa) {                    \
        translucent_op<NAME##_functor> op(opa);                       \
        for (tjs_int i = 0; i < len; i++) d[i] = op(d[i], s[i]);    \
    }                                                                 \
    /* HDA 版 */                                                      \
    void TVP##NAME##_HDA(tjs_uint32 *d, const tjs_uint32 *s,         \
                         tjs_int len) {                                \
        hda_op<NAME##_functor> op;                                    \
        for (tjs_int i = 0; i < len; i++) d[i] = op(d[i], s[i]);    \
    }                                                                 \
    /* HDA + opacity 版 */                                            \
    void TVP##NAME##_HDA_o(tjs_uint32 *d, const tjs_uint32 *s,       \
                           tjs_int len, tjs_int opa) {                \
        hda_translucent_op<NAME##_functor> op(opa);                   \
        for (tjs_int i = 0; i < len; i++) d[i] = op(d[i], s[i]);    \
    }
    // ... 还有 _d、_do、_a、_ao 等变体

// 使用：一行宏为乘法混合生成所有变体
DEFINE_BLEND_VARIATION(mul_blend)
// 展开为: TVPmul_blend, TVPmul_blend_o, TVPmul_blend_HDA,
//         TVPmul_blend_HDA_o, TVPmul_blend_d, TVPmul_blend_do
```

### 3.5 变体后缀速查表

| 后缀 | 含义 | 场景 |
|------|------|------|
| _(无)_ | 基本混合 | 目标 alpha 被忽略 |
| `_o` | + Opacity（全局不透明度） | 图层有透明度设置 |
| `_HDA` | Hold Dest Alpha | 保持目标 alpha 不被修改 |
| `_HDA_o` | HDA + Opacity | 两者结合 |
| `_d` | Dest Alpha | 目标有独立 alpha 通道参与运算 |
| `_do` | Dest Alpha + Opacity | 两者结合 |
| `_a` | Additive Alpha（预乘源） | 源是 premultiplied alpha 格式 |
| `_ao` | Additive Alpha + Opacity | 预乘源 + 全局不透明度 |

这种架构意味着：**添加一种新的混合模式只需写一个 functor（~10 行），宏自动生成所有变体和包装函数。** 极大减少了代码重复和维护成本。

---

## 4. 查找表（LUT）优化

### 4.1 为什么需要查找表

某些混合模式（如 Color Dodge、Color Burn、Soft Light）的公式包含**除法运算**。在整数域中，除法是最昂贵的算术操作之一——一次整数除法需要 20~40 个 CPU 时钟周期，而加法/移位只需 1 个周期。

KrKr2 的策略：**把所有可能的输入→输出映射预先计算好存入表中，运行时直接查表**。

### 4.2 TVPDivTable：除法倒数表

```cpp
// 文件: cpp/core/visual/tvpgl.cpp, 第 13-15 行
tjs_uint32 TVPDivTable[256];
// TVPDivTable[x] ≈ (1/x) × 65536（定点数，Fixed-point，
// 用整数模拟小数，通过左移16位将小数部分存储在整数的低16位中）

// 初始化:
for (int i = 1; i < 256; i++)
    TVPDivTable[i] = 65536 / i;
TVPDivTable[0] = 65536;  // 防止除零

// 使用: a / b ≈ a * TVPDivTable[b] >> 16
// 例: 200 / 128 ≈ 200 * (65536/128) >> 16 = 200 * 512 >> 16 = 1.5625
```

这张 256 项的表只占 **1KB** 内存，完全在 CPU L1 缓存（Cache，CPU 芯片内最快的存储区，通常 32-64KB）中，查表延迟接近 1 个周期。

### 4.3 TVPRecipTable256：倒数表

```cpp
// 文件: cpp/core/visual/tvpgl.cpp
tjs_uint32 TVPRecipTable256[256];
// TVPRecipTable256[x] = 65536 / x（与 TVPDivTable 类似，
// 但专门用于 Color Dodge 混合模式）

// Color Dodge 公式: result = dst / (1 - src)
// 整数版: result = dst * TVPRecipTable256[255 - src] >> 8
```

在 `color_dodge_blend_functor` 中的实际用法：

```cpp
// 文件: cpp/core/visual/gl/blend_functor_c.h, 第 246 行附近
struct color_dodge_blend_functor {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        tjs_uint32 tmp2 = ~s;  // 255 - s（各通道取反）
        // R 通道: dst_r × (65536 / (255 - src_r)) >> 8
        tjs_uint32 tmp = (d & 0xff) * TVPRecipTable256[tmp2 & 0xff] >> 8;
        // 饱和到 255（无分支技巧）:
        // 如果 tmp < 256，高位全 0；如果 tmp >= 256，通过位运算饱和到 0xFF
        tjs_uint32 tmp3 = (tmp | ((tjs_int32)~(tmp - 0x100) >> 31)) & 0xff;
        // G、B 通道同理...
        return tmp3;
    }
};
```

### 4.4 TVPOpacityOnOpacityTable：不透明度合成表

当两层半透明图层叠加时，需要计算合成后的总不透明度：

```
合成公式: A_out = 1 - (1 - A1)(1 - A2)
```

这个公式包含浮点乘法。KrKr2 预计算了 256×256 的查找表：

```cpp
// 文件: cpp/core/visual/tvpgl.cpp
unsigned char TVPOpacityOnOpacityTable[256 * 256];  // 64KB

// 初始化:
for (int a1 = 0; a1 < 256; a1++) {
    for (int a2 = 0; a2 < 256; a2++) {
        float result = 1.0f - (1.0f - a1 / 255.0f) * (1.0f - a2 / 255.0f);
        TVPOpacityOnOpacityTable[a1 * 256 + a2] = (unsigned char)(result * 255);
    }
}

// 使用:
unsigned char combined = TVPOpacityOnOpacityTable[alpha1 * 256 + alpha2];
// 例: alpha1=128, alpha2=128
//   → 1 - (1-0.502)(1-0.502) = 1 - 0.498×0.498 = 1 - 0.248 = 0.752 ≈ 192
```

### 4.5 TVPNegativeMulTable：负乘表

```cpp
// 文件: cpp/core/visual/tvpgl.cpp
unsigned char TVPNegativeMulTable[256 * 256];  // 64KB

// TVPNegativeMulTable[a][b] = 255 - (255-a)(255-b)/255
// 等价于 Screen 混合的单通道结果
// 使用此表可以将 Screen 混合从 3 次除法变为 3 次查表
```

### 4.6 Photoshop 混合查找表

Soft Light、Color Dodge、Color Burn 三种模式各自有 256×256 的查找表：

```cpp
// 文件: cpp/core/visual/gl/blend_functor_c.h, 第 461-479 行
struct ps_soft_light_table {
    static unsigned char TABLE[256][256];  // 64KB
};
struct ps_color_dodge_table {
    static unsigned char TABLE[256][256];  // 64KB
};
struct ps_color_burn_table {
    static unsigned char TABLE[256][256];  // 64KB
};
// 三张表共 192KB
```

查表版的混合 functor 极其简单：

```cpp
// 使用查找表的 Soft Light 混合
struct ps_soft_light_blend_functor {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        return (ps_soft_light_table::TABLE[(s >> 16) & 0xff][(d >> 16) & 0xff] << 16)
             | (ps_soft_light_table::TABLE[(s >>  8) & 0xff][(d >>  8) & 0xff] <<  8)
             | (ps_soft_light_table::TABLE[ s        & 0xff][ d        & 0xff]);
    }
};
```

### 4.7 查找表内存总结

| 查找表 | 大小 | 用途 |
|--------|------|------|
| TVPDivTable | 1 KB | 整数除法替代 |
| TVPRecipTable256 | 1 KB | Color Dodge 除法 |
| TVPOpacityOnOpacityTable | 64 KB | 两层 Alpha 合成 |
| TVPNegativeMulTable | 64 KB | Screen 混合 |
| ps_soft_light_table | 64 KB | PS Soft Light |
| ps_color_dodge_table | 64 KB | PS Color Dodge |
| ps_color_burn_table | 64 KB | PS Color Burn |
| **合计** | **~322 KB** | — |

322 KB 在现代系统中微不足道，但需要注意：64 KB 的表刚好在 L1 缓存（通常 32-64 KB/核心）的边界上。如果同时访问多张大表，可能导致缓存抖动（Cache Thrashing，多个数据块反复争抢同一块缓存空间）。KrKr2 的设计中，同一时刻通常只使用一种 PS 混合模式，所以实际只有一张 64KB 表在热路径上。

---

## 5. 性能权重因子

### 5.1 设计思路

KrKr2 为每种混合模式分配了一个**性能权重因子**（`sBmFactor`），数值越大表示该模式越快（单位时间能处理越多像素）。引擎的多线程调度器用这个因子决定**是否值得为当前混合操作启用多线程**。

```cpp
// 文件: cpp/core/visual/LayerBitmapIntf.cpp, 第 46-79 行
const static float sBmFactor[] = {
    59,  // bmCopy         — 纯内存复制，很快
    52,  // bmAlpha        — 标准混合，快
    61,  // bmAdd          — 饱和加法，快
    45,  // bmMul          — 逐通道乘法，中等
    10,  // bmDodge        — 需要除法/查表，慢
    10,  // bmPsSoftLight  — 需要 256×256 查表，慢
    10,  // bmPsColorDodge — 需要 256×256 查表，慢
    66,  // bmPsExclusion  — 公式简单 (D+S-2DS/255)，最快的 PS 模式
};
```

### 5.2 调度逻辑

```cpp
// 简化的多线程调度决策（伪代码）
void dispatch_blend(tTVPBBBltMethod method, int pixel_count) {
    float factor = sBmFactor[method];
    // 工作量估算 = 像素数 / 速度因子
    float work_estimate = pixel_count / factor;

    if (work_estimate > THREAD_THRESHOLD) {
        // 工作量大（因为模式慢或像素多），启用多线程
        parallel_blend(method, pixel_count);
    } else {
        // 工作量小，单线程更划算（避免线程创建开销）
        single_thread_blend(method, pixel_count);
    }
}
```

这意味着：bmDodge（因子=10）处理同样数量的像素时，比 bmPsExclusion（因子=66）更可能触发多线程——因为它更慢，需要更多计算时间。

---

## 动手实践

### 练习：实现 5 种基础混合模式

```cpp
#include <cstdint>
#include <cstdio>
#include <algorithm>

using tjs_uint32 = uint32_t;

// 饱和加法（无分支版本，第 4 章将详细讲解位运算原理）
tjs_uint32 saturated_add(tjs_uint32 a, tjs_uint32 b) {
    tjs_uint32 tmp = ((a & b) + (((a ^ b) >> 1) & 0x7f7f7f7f)) & 0x80808080;
    tmp = (tmp << 1) - (tmp >> 7);
    return (a + b - tmp) | tmp;
}

// bmAlpha: 标准混合
tjs_uint32 blend_alpha(tjs_uint32 d, tjs_uint32 s) {
    tjs_uint32 a = s >> 24;
    tjs_uint32 d1 = d & 0xff00ff;
    d1 = (d1 + (((s & 0xff00ff) - d1) * a >> 8)) & 0xff00ff;
    tjs_uint32 d2 = d & 0xff00;
    return d1 + ((d2 + (((s & 0xff00) - d2) * a >> 8)) & 0xff00);
}

// bmAdd: 饱和加法（源 alpha 调制后相加）
tjs_uint32 blend_add(tjs_uint32 d, tjs_uint32 s) {
    tjs_uint32 a = s >> 24;
    tjs_uint32 s_rb = ((s & 0xff00ff) * a >> 8) & 0xff00ff;
    tjs_uint32 s_g  = ((s & 0x00ff00) * a >> 8) & 0x00ff00;
    return saturated_add(d, s_rb | s_g);
}

// bmMul: 逐通道乘法
tjs_uint32 blend_mul(tjs_uint32 d, tjs_uint32 s) {
    uint8_t r = ((d >> 16) & 0xff) * ((s >> 16) & 0xff) / 255;
    uint8_t g = ((d >>  8) & 0xff) * ((s >>  8) & 0xff) / 255;
    uint8_t b = ( d        & 0xff) * ( s        & 0xff) / 255;
    return (r << 16) | (g << 8) | b;
}

// bmScreen: 1 - (1-D)(1-S)
tjs_uint32 blend_screen(tjs_uint32 d, tjs_uint32 s) {
    tjs_uint32 id = ~d, is = ~s;
    uint8_t r = 255 - ((id >> 16) & 0xff) * ((is >> 16) & 0xff) / 255;
    uint8_t g = 255 - ((id >>  8) & 0xff) * ((is >>  8) & 0xff) / 255;
    uint8_t b = 255 - ( id        & 0xff) * ( is        & 0xff) / 255;
    return (r << 16) | (g << 8) | b;
}

// bmDifference: |D - S|
tjs_uint32 blend_difference(tjs_uint32 d, tjs_uint32 s) {
    auto abs_diff = [](uint8_t a, uint8_t b) -> uint8_t {
        return a > b ? a - b : b - a;
    };
    uint8_t r = abs_diff((d >> 16) & 0xff, (s >> 16) & 0xff);
    uint8_t g = abs_diff((d >>  8) & 0xff, (s >>  8) & 0xff);
    uint8_t b = abs_diff( d        & 0xff,  s        & 0xff);
    return (r << 16) | (g << 8) | b;
}

void print_pixel(const char* label, tjs_uint32 p) {
    printf("%-14s: R=%3d G=%3d B=%3d  (0x%06X)\n", label,
           (p >> 16) & 0xff, (p >> 8) & 0xff, p & 0xff, p & 0xffffff);
}

int main() {
    tjs_uint32 dst = 0xFF804020;  // 不透明暗橙色
    tjs_uint32 src = 0x80FF8040;  // 半透明橙黄色

    printf("=== 基础混合模式对比 ===\n");
    print_pixel("DST", dst);
    print_pixel("SRC", src);
    printf("\n");

    print_pixel("bmAlpha",      blend_alpha(dst, src));
    print_pixel("bmAdd",        blend_add(dst, src));
    print_pixel("bmMul",        blend_mul(dst, src));
    print_pixel("bmScreen",     blend_screen(dst, src));
    print_pixel("bmDifference", blend_difference(dst, src));

    printf("\n--- 观察要点 ---\n");
    printf("bmAdd:  整体偏亮（饱和加法）\n");
    printf("bmMul:  整体偏暗（暗色×暗色更暗）\n");
    printf("bmScreen: 整体偏亮（亮色+亮色更亮）\n");
    printf("bmDifference: 相似色→暗，不同色→亮\n");

    return 0;
}
```

**CMakeLists.txt**：

```cmake
cmake_minimum_required(VERSION 3.16)
project(blend_modes_demo LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
add_executable(blend_modes_demo main.cpp)
```

```bash
# Windows
cmake -B build -G Ninja && cmake --build build && build\blend_modes_demo.exe

# Linux / macOS
cmake -B build -G Ninja && cmake --build build && ./build/blend_modes_demo
```

---

## 常见错误及解决方案

### 错误 1：混淆基础混合模式和 PS 混合模式

**症状**：用 `bmAdd` 期望得到 Photoshop "线性减淡" 的效果，但颜色不对。

**原因**：`bmAdd` 是引擎自己的简化加法（饱和加法 + alpha 调制），`bmPsAdditive` 才是 Photoshop 兼容的线性减淡。两者的 alpha 处理方式不同。

**解决**：如果需要 Photoshop 兼容效果，使用 `bmPs*` 系列；如果只需要简单的颜色运算，用基础系列。

### 错误 2：自定义 functor 忘记处理溢出

**症状**：混合结果出现错误的亮色或暗色条带。

**原因**：逐通道计算后没有限制到 [0, 255] 范围。

```cpp
// ❌ 错误：未处理溢出
uint8_t r = d_r + s_r;  // 如果 d_r=200, s_r=100 → r=300 → 溢出为 44

// ✅ 正确：饱和处理
uint8_t r = std::min(255, (int)d_r + (int)s_r);
// 或使用位运算无分支饱和（见第 4 章）
```

### 错误 3：查找表未初始化就使用

**症状**：程序启动后第一次混合操作结果全是 0 或随机值。

**原因**：KrKr2 的查找表在 `TVPCreateTable()` 中统一初始化。如果在引擎初始化完成前就调用了混合函数，表还是全零。

**解决**：确保在引擎初始化流程（`TVPAppDelegate::run()`）中调用 `TVPCreateTable()` 后再进行任何图形操作。

---

## 对照项目源码

| 文件 | 内容 |
|------|------|
| `cpp/core/visual/LayerBitmapIntf.h` 第 28-61 行 | `tTVPBBBltMethod` 完整枚举 |
| `cpp/core/visual/gl/blend_functor_c.h` | 所有混合模式的 functor 实现（1027 行） |
| `cpp/core/visual/gl/blend_function.cpp` | `DEFINE_BLEND_VARIATION` 宏展开（685 行） |
| `cpp/core/visual/gl/blend_variation.h` | `normal_op` / `translucent_op` / `hda_op` 等模板 |
| `cpp/core/visual/tvpgl.cpp` | 查找表初始化（`TVPCreateTable` 函数）+ C 默认实现 |
| `cpp/core/visual/LayerBitmapIntf.cpp` 第 46-79 行 | 性能权重因子 `sBmFactor` |

**阅读建议**：
1. 先看 `blend_functor_c.h` 中的 `alpha_blend_func`（第 67 行）理解最基础的混合
2. 然后看同文件中的 `mul_blend_functor`、`screen_blend_functor` 等，了解不同混合模式的核心算法
3. 再看 `blend_variation.h` 理解模板如何包装 functor
4. 最后看 `blend_function.cpp` 的 `DEFINE_BLEND_VARIATION` 宏如何把一切粘合在一起

---

## 本节小结

- **Porter-Duff** 12 种合成模式通过选择保留/丢弃四个像素区域来定义，SrcOver 是最常用的模式
- KrKr2 定义了 **31 种混合模式**，分为基础类（位运算优化）、Alpha 格式转换类、Photoshop 兼容类（查找表优化）三大类
- **Functor + 宏展开架构**：核心算法只写一次（functor），宏自动生成 6~8 种变体（`_o`、`_HDA`、`_d`、`_a` 等），极大减少代码重复
- **查找表**（TVPDivTable、TVPRecipTable256、TVPOpacityOnOpacityTable、ps_*_table）用预计算结果替代运行时除法，总计约 322 KB
- **性能权重因子** `sBmFactor` 用于多线程调度——慢的模式更倾向于启用多线程

---

## 练习题与答案

### 题目 1：Porter-Duff 模式选择

给定以下需求，应该使用哪种 Porter-Duff 模式？

(a) 将角色立绘叠加到背景上，角色半透明边缘与背景自然过渡
(b) 用一个圆形蒙版裁剪头像，只保留圆形内部的头像内容
(c) 制作"镂空"效果——在背景上按字体形状挖出透明区域

<details>
<summary>查看答案</summary>

**(a) SrcOver**
标准图层叠加。前景的半透明像素与背景混合，不透明像素覆盖背景。对应 KrKr2 的 `bmAlpha`。

**(b) DstIn**
`DstIn` 公式：`Co = Cd × As`。蒙版（源）不透明的地方保留头像（目标），蒙版透明的地方头像也变透明。效果等于"头像只在蒙版范围内可见"。

**(c) DstOut**
`DstOut` 公式：`Co = Cd × (1 - As)`。源（字体形状）不透明的地方，目标变透明——实现"挖洞"效果。

</details>

### 题目 2：分析 Functor 架构的优势

假设不使用 Functor + 宏展开架构，而是为每种混合模式的每种变体单独写函数：

(a) 31 种混合模式 × 8 种变体 = 多少个函数？
(b) 如果发现 alpha 混合公式有 bug 需要修复，需要修改多少处代码？
(c) Functor 架构下同样的修复需要改多少处？

<details>
<summary>查看答案</summary>

**(a)** 31 × 8 = **248 个函数**。每个函数至少 5-10 行，共 1240-2480 行几乎重复的代码。

**(b)** 如果 bug 在 alpha 混合逻辑中（比如 `>> 8` 应改为更精确的公式），需要修改**所有 248 个函数**中与 alpha 相关的代码行。实际上人工逐个修改 248 处极易遗漏。

**(c)** Functor 架构下只需修改 **1 处**——`blend_variation.h` 中的 `normal_op`/`translucent_op` 模板里的 alpha 混合代码。所有使用该模板的变体自动获得修复。

这就是 KrKr2 选择 Functor + 宏架构的核心原因：**单点维护，全局生效**。

</details>

### 题目 3：设计自定义查找表

设计一个查找表实现 Exclusion 混合模式（`result = src + dst - 2 × src × dst / 255`）。

(a) 该表需要多大？
(b) 写出初始化代码
(c) 写出使用该表的 functor

<details>
<summary>查看答案</summary>

```cpp
#include <cstdint>
#include <cstdio>

// (a) 256 × 256 × 1 字节 = 64 KB（与 ps_soft_light_table 相同）

// (b) 初始化
unsigned char exclusion_table[256][256];

void init_exclusion_table() {
    for (int s = 0; s < 256; s++) {
        for (int d = 0; d < 256; d++) {
            int result = s + d - 2 * s * d / 255;
            // 理论上 result 范围 [0, 255]，但因整数截断可能有 ±1 误差
            if (result < 0) result = 0;
            if (result > 255) result = 255;
            exclusion_table[s][d] = (unsigned char)result;
        }
    }
}

// (c) 使用查找表的 functor
struct exclusion_blend_functor {
    inline uint32_t operator()(uint32_t d, uint32_t s) const {
        return (exclusion_table[(s >> 16) & 0xff][(d >> 16) & 0xff] << 16)
             | (exclusion_table[(s >>  8) & 0xff][(d >>  8) & 0xff] <<  8)
             | (exclusion_table[ s        & 0xff][ d        & 0xff]);
    }
};

int main() {
    init_exclusion_table();

    // 验证: s=200, d=100 → 200 + 100 - 2×200×100/255 = 300 - 156 = 143
    printf("exclusion(200, 100) = %d (期望 ≈143)\n", exclusion_table[200][100]);
    // s=0, d=0 → 0
    printf("exclusion(0, 0) = %d (期望 0)\n", exclusion_table[0][0]);
    // s=255, d=255 → 255+255-2×255 = 255
    printf("exclusion(255, 255) = %d (期望 255)\n", exclusion_table[255][255]);

    return 0;
}
```

注意：KrKr2 中 `bmPsExclusion` 实际上**没有**使用查找表，因为它的公式 `D+S-2DS/255` 可以用位运算高效计算（不含除法——`2DS/255` 可以用 `DS >> 7` 近似），所以它的性能因子高达 66（最快的 PS 模式）。查找表主要用于包含真正除法的模式（Color Dodge/Burn/Soft Light）。

</details>

---

## 下一步

下一节 [动手实践与总结](./03-动手实践与总结.md)，我们将实现一个完整的迷你混合器程序，支持多种混合模式和图层合成，并对照 KrKr2 源码验证实现的正确性。
