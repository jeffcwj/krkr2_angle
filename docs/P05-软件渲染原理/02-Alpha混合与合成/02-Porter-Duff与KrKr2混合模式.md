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

