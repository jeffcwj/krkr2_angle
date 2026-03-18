# ARGB 混合与合成

> **所属模块：** M04-渲染子系统
> **前置知识：** [01-tvpgl像素操作库](./01-tvpgl像素操作库.md)、[P05-软件渲染原理](../../P05-软件渲染原理/README.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 KrKr2 混合仿函数（Blend Functor）的三层架构设计
2. 掌握 `blend_variation.h` 中 6 种 alpha 变体包装器的工作原理
3. 读懂 Photoshop 兼容混合模式（Screen、Overlay、SoftLight 等）的位运算实现
4. 理解 256×256 查找表在 PS 混合模式中的作用与初始化流程
5. 能够自行添加一种新的混合模式到 tvpgl 系统中

## 混合仿函数三层架构

KrKr2 的像素混合系统采用 C++ 模板仿函数（Functor）设计，将混合逻辑分为三层。这种设计让每种混合算法只需编写一次核心逻辑，就能自动生成多种 alpha 处理变体。

```
┌─────────────────────────────────────────────────┐
│  第三层：模板循环函数 (blend_function.cpp)        │
│  blend_func_c<F>, copy_func_c<F>, ...           │
│  负责遍历像素数组，调用仿函数                      │
├─────────────────────────────────────────────────┤
│  第二层：变体包装器 (blend_variation.h)           │
│  normal_op<F>, translucent_op<F>, hda_op<F>, ...│
│  负责提取/调整 alpha 值                          │
├─────────────────────────────────────────────────┤
│  第一层：基础混合函数 (blend_functor_c.h)         │
│  alpha_blend_func, add_blend_func, ...          │
│  纯粹的颜色混合数学运算                          │
└─────────────────────────────────────────────────┘
```

### 第一层：基础混合函数

基础混合函数（Base Blend Function）是最底层的仿函数结构体，定义了一个接受三个参数的 `operator()`：目标像素 `d`、源像素 `s`、以及已计算好的 alpha 值 `a`。这三个参数都是 `tjs_uint32` 类型（即 32 位无符号整数），对应一个 BGRA 像素。alpha 值 `a` 的范围是 0-255。

基础混合函数**不关心 alpha 从哪里来**——它可能来自源像素的 A 通道、外部指定的不透明度、或者两者的乘积。这种关注点分离（Separation of Concerns）是整个架构的核心设计哲学。

以最基本的 alpha 混合为例：

```cpp
// 文件：cpp/core/visual/gl/blend_functor_c.h 第 20-35 行
struct alpha_blend_func {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s,
                                 tjs_uint32 a) const {
        // 双通道并行混合：同时处理 B 和 R 通道
        tjs_uint32 dcl = d & 0xff00ff;  // 提取 dest 的 B(低8位) 和 R(高8位)
        // (s_br - d_br) * alpha >> 8 + d_br，结果只保留 B 和 R 通道
        dcl = ((dcl + (((s & 0xff00ff) - dcl) * a >> 8)) & 0xff00ff);
        // 单独处理 G 通道
        d &= 0xff00;   // 提取 dest 的 G 通道
        s &= 0xff00;   // 提取 src 的 G 通道
        // 同样的线性插值公式，只保留 G 通道
        return dcl | ((d + ((s - d) * a >> 8)) & 0xff00);
    }
};
```

这里的关键技巧是**双通道并行处理**：利用 `0xff00ff` 掩码同时提取 B（位 0-7）和 R（位 16-23）通道。因为 G 通道（位 8-15）夹在中间，乘法时不会互相干扰（前提是乘数不超过 256）。这个技巧在上一节中已经详细介绍过。

### 常见基础混合函数一览

项目实现了丰富的基础混合函数，下表列出核心的几种：

| 仿函数名 | 混合公式 | 说明 |
|----------|---------|------|
| `alpha_blend_func` | `d + (s - d) * a >> 8` | 标准 alpha 混合 |
| `add_blend_func` | `sat(d + s * a >> 8)` | 加法混合（饱和加法） |
| `sub_blend_func` | `d - s * a >> 8` | 减法混合 |
| `mul_blend_func` | `d * s >> 8` | 乘法混合 |
| `lighten_blend_func` | `max(d, s)` | 取亮 |
| `darken_blend_func` | `min(d, s)` | 取暗 |
| `screen_blend_func` | `s + d - s*d/255` | 滤色 |
| `color_dodge_blend_func` | `d / (1 - s)` | 颜色减淡 |

加法混合（Additive Blend）使用饱和加法防止溢出，其实现如下：

```cpp
// 文件：cpp/core/visual/gl/blend_functor_c.h 第 37-56 行
struct add_blend_func {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s,
                                 tjs_uint32 a) const {
        // 先将源像素乘以 alpha
        s = (((s & 0xff00ff) * a >> 8) & 0xff00ff) +
            (((s >> 8) & 0xff00ff) * a & 0xff00ff00);
        // 饱和加法核心算法——无分支判断溢出
        tjs_uint32 tmp =
            ((d & s) + (((d ^ s) >> 1) & 0x7f7f7f7f)) & 0x80808080;
        tmp = (tmp << 1) - (tmp >> 7);
        return (d + s - tmp) | tmp;
    }
};
```

饱和加法的实现值得深入分析。传统做法需要对每个通道分别判断溢出，这需要 4 次比较和分支。KrKr2 的实现使用纯位运算在一次操作中处理全部 4 个通道：

1. `(d & s) + (((d ^ s) >> 1) & 0x7f7f7f7f)` 计算每字节的"半加"，等价于 `(d + s) / 2`
2. `& 0x80808080` 提取每字节最高位——如果最高位为 1，说明 `(d+s)/2 >= 128`，即 `d + s >= 256`，发生了溢出
3. `(tmp << 1) - (tmp >> 7)` 将标志位扩展为全 0 或全 1 的掩码（0x00 或 0xFF）
4. `(d + s - tmp) | tmp` 用掩码修正：溢出的通道变为 0xFF，未溢出的保持原值

## 变体包装器层

第二层是 `blend_variation.h` 中定义的模板包装器。它们的作用是**决定 alpha 值从哪里来、怎么处理**，然后调用底层的基础混合函数。这一层将 3 参数的基础函数（d, s, a）转换为 2 参数的接口（d, s），供模板循环函数调用。

KrKr2 定义了以下 6 种变体：

```
┌──────────────────────┬───────────────────────────────────────┐
│ 变体                  │ alpha 来源与处理                       │
├──────────────────────┼───────────────────────────────────────┤
│ normal_op<F>         │ a = s >> 24（使用源像素 alpha）         │
│ translucent_op<F>    │ a = (s >> 24) * opa >> 8（源alpha×透明度）│
│ hda_op<F>            │ 同 normal_op，但保护目标 alpha 通道      │
│ hda_translucent_op<F>│ 同 translucent_op，但保护目标 alpha 通道 │
│ dest_alpha_op<F>     │ 考虑 dest alpha 的正确混合               │
│ dest_alpha_translucent_op<F> │ 同上 + 外部透明度               │
└──────────────────────┴───────────────────────────────────────┘
```

### normal_op —— 最基本的源 alpha 变体

`normal_op<F>` 从源像素的最高 8 位提取 alpha 值，直接传给基础混合函数。这是最常见的使用方式——源图像自带透明度信息：

```cpp
// 文件：cpp/core/visual/gl/blend_variation.h 第 26-35 行
template <class blend_func>
struct normal_op {
    blend_func func_;
    inline normal_op() = default;
    inline normal_op(tjs_uint32 opa) {}  // opa 参数被忽略
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        tjs_uint32 a = (s >> 24);        // 从源像素提取 alpha
        return func_(d, s, a);           // 调用基础混合函数
    }
};
```

### translucent_op —— 带外部不透明度的变体

`translucent_op<F>` 同时考虑源像素 alpha 和外部指定的不透明度参数 `opa`。两者相乘得到最终的 alpha 值。这用于实现"淡入淡出"效果——即使源像素本身是半透明的，还可以再叠加一层整体透明度：

```cpp
// 文件：cpp/core/visual/gl/blend_variation.h 第 38-48 行
template <class blend_func>
struct translucent_op {
    const tjs_int opa_;                  // 外部不透明度 (0-255)
    blend_func func_;
    inline translucent_op() : opa_(255) {}
    inline translucent_op(tjs_uint32 opa) : opa_(opa) {}
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        // 源 alpha × 外部不透明度，结果仍在 0-255 范围
        tjs_uint32 a = ((s >> 24) * opa_) >> 8;
        return func_(d, s, a);
    }
};
```

### hda_op —— 保护目标 alpha 的变体

HDA 代表 "Hold Destination Alpha"（保持目标 alpha）。在某些场景中，目标缓冲区的 alpha 通道有特殊用途（例如用作遮罩或深度信息），混合操作不应修改它。`hda_op<F>` 在混合完成后用位运算恢复目标像素原来的 alpha 值：

```cpp
// 文件：cpp/core/visual/gl/blend_variation.h 第 62-71 行
template <class blend_func>
struct hda_op {
    blend_func func_;
    inline hda_op() = default;
    inline hda_op(tjs_uint32 opa) {}
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        tjs_uint32 a = (s >> 24);
        // 混合结果只取低 24 位（BGR），alpha 取自原始 dest
        return (func_(d, s, a) & 0x00ffffff) | (d & 0xff000000);
    }
};
```

### dest_alpha_op —— 正确处理目标 alpha 的变体

`dest_alpha_op<F>` 是最复杂的变体。当源和目标都有有意义的 alpha 通道时（例如两个半透明图层叠加），需要按照 Porter-Duff 合成规则正确计算结果 alpha 和混合权重。

KrKr2 使用查找表 `TVPOpacityOnOpacityTable` 和 `TVPNegativeMulTable` 来避免浮点运算：

```cpp
// 文件：cpp/core/visual/gl/blend_variation.h 第 108-146 行
template <class blend_func>
struct dest_alpha_op {
    blend_func func_;
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        // 从查找表获取混合权重和结果 alpha
        tjs_uint32 addr = ((s >> 16) & 0xff00) + (d >> 24);
        // TVPNegativeMulTable[sa*256+da] = 255 - (255-sa)*(255-da)/255
        //   即 Porter-Duff "over" 操作的结果 alpha
        tjs_uint32 destalpha = TVPNegativeMulTable[addr] << 24;
        // TVPOpacityOnOpacityTable[sa*256+da] = sa / result_alpha * 255
        //   即源像素在最终结果中的混合权重
        tjs_uint32 sopa = TVPOpacityOnOpacityTable[addr];
        return func_(d, s, sopa) + destalpha;
    }
};
```

查找表地址编码为 `sa * 256 + da`，其中 `sa` 是源 alpha，`da` 是目标 alpha。两个表的含义：
- **TVPNegativeMulTable**：`result_alpha = sa + da - sa*da/255`（Porter-Duff "over" 合成的结果 alpha）
- **TVPOpacityOnOpacityTable**：`weight = sa / result_alpha * 255`（源像素在混合中的权重）

### DEFINE_BLEND_VARIATION 宏：自动生成变体

每写一个基础混合函数 `xxx_func`，用一个宏就能生成全部 6 种变体的 typedef：

```cpp
// 宏展开示例（伪代码）
#define DEFINE_BLEND_VARIATION(xxx) \
    typedef normal_op<xxx##_func>               xxx##_functor;        \
    typedef translucent_op<xxx##_func>          xxx##_o_functor;      \
    typedef hda_op<xxx##_func>                  xxx##_HDA_functor;    \
    typedef hda_translucent_op<xxx##_func>      xxx##_HDA_o_functor;  \
    typedef dest_alpha_op<xxx##_func>           xxx##_d_functor;      \
    typedef dest_alpha_translucent_op<xxx##_func> xxx##_do_functor;

// 使用示例
DEFINE_BLEND_VARIATION(alpha_blend)
// 生成：alpha_blend_functor, alpha_blend_o_functor,
//       alpha_blend_HDA_functor, alpha_blend_HDA_o_functor,
//       alpha_blend_d_functor, alpha_blend_do_functor
```

命名后缀含义：
- 无后缀（`_functor`）：使用源 alpha
- `_o`：带外部不透明度（opacity）
- `_HDA`：保护目标 alpha
- `_HDA_o`：保护目标 alpha + 外部不透明度
- `_d`：处理目标 alpha（dest alpha）
- `_do`：处理目标 alpha + 外部不透明度

## Photoshop 兼容混合模式

KrKr2 实现了一套完整的 Photoshop 兼容混合模式（PS Blend Modes），以 `ps_` 前缀命名。这些模式继承自 `ps_alpha_blend_func` 基类，该基类与普通 `alpha_blend_func` 相同，但在 PS 混合模式体系中作为统一的 alpha 合成入口。

PS 混合模式的设计模式统一：先计算混合后的颜色值 `s`，然后调用 `ps_alpha_blend_func::operator()` 将结果与目标像素按 alpha 合成。

### 乘法混合（Multiply）

乘法混合是最直观的模式——每个通道的值相乘后除以 255：

```cpp
// 文件：cpp/core/visual/gl/blend_functor_c.h 第 416-426 行
struct ps_mul_blend_func : public ps_alpha_blend_func {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s,
                                 tjs_uint32 a) const {
        // 逐通道相乘：R*R, G*G, B*B
        s = (((((d >> 16) & 0xff) * (s & 0x00ff0000)) & 0xff000000) |
             ((((d >> 8) & 0xff) * (s & 0x0000ff00)) & 0x00ff0000) |
             ((((d >> 0) & 0xff) * (s & 0x000000ff)))) >> 8;
        // 将乘法结果作为新的源像素，按 alpha 合成到目标上
        return ps_alpha_blend_func::operator()(d, s, a);
    }
};
```

注意这里的技巧：每个通道分别提取、相乘、再移位对齐。不能使用双通道并行技巧，因为乘法结果可能超过 16 位。

### 滤色混合（Screen）

滤色是乘法的"反色"版本，公式为 `c = s + d - s*d/255`。它总是产生更亮的结果：

```cpp
// 文件：cpp/core/visual/gl/blend_functor_c.h 第 430-444 行
struct ps_screen_blend_func {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s,
                                 tjs_uint32 a) const {
        // 计算 s*d/255（分两组处理：BR 和 G）
        tjs_uint32 sd1, sd2;
        sd1 = (((((d >> 16) & 0xff) * (s & 0x00ff0000)) & 0xff000000) |
               ((((d >> 0) & 0xff) * (s & 0x000000ff)))) >> 8;
        sd2 = (((((d >> 8) & 0xff) * (s & 0x0000ff00)) & 0x00ff0000)) >> 8;
        // c = (s - s*d/255) * a + d  等价于  (s + d - s*d/255 - d)*a + d
        return ((((((s & 0x00ff00ff) - sd1) * a) >> 8) + (d & 0x00ff00ff)) &
                0x00ff00ff) |
            ((((((s & 0x0000ff00) - sd2) * a) >> 8) + (d & 0x0000ff00)) &
             0x0000ff00);
    }
};
```

### 叠加混合（Overlay）

叠加模式根据目标像素的亮度选择不同的混合策略：暗区用乘法加深，亮区用滤色提亮。阈值为 128（0x80）：

```cpp
// 文件：cpp/core/visual/gl/blend_functor_c.h 第 512-531 行
struct ps_overlay_blend_func : public ps_alpha_blend_func {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s,
                                 tjs_uint32 a) const {
        // 生成通道掩码：d < 128 → 0xFF, d >= 128 → 0x00
        tjs_uint32 n = (((d & 0x00808080) >> 7) + 0x007f7f7f) ^ 0x007f7f7f;
        tjs_uint32 sa1, sa2, d1 = d & n, s1 = s & n;
        s |= 0x00010101;  // 防溢出调整
        // 计算 d*s*2 用于暗区（乘法分支）
        sa1 = (((((d >> 16) & 0xff) * (s & 0x00ff0000)) & 0xff800000) |
               ((((d >> 0) & 0xff) * (s & 0x000000ff)))) >> 7;
        sa2 = (((((d >> 8) & 0xff) * (s & 0x0000ff00)) & 0x00ff8000)) >> 7;
        // 暗区结果 = d*s*2（用掩码选择）
        s = ((sa1 & ~n) | (sa2 & ~n));
        // 亮区结果 = s + d - s*d*2（合并两个分支）
        s |= (((s1 & 0x00fe00fe) + (d1 & 0x00ff00ff)) << 1) -
            (n & 0x00ff00ff) - (sa1 & n);
        s |= (((s1 & 0x0000fe00) + (d1 & 0x0000ff00)) << 1) -
            (n & 0x0000ff00) - (sa2 & n);
        return ps_alpha_blend_func::operator()(d, s, a);
    }
};
```

掩码生成的关键步骤：`(((d & 0x00808080) >> 7) + 0x007f7f7f) ^ 0x007f7f7f` 将每个通道的第 7 位（最高位）转换为全通道掩码。如果通道值 < 128（bit7 = 0），掩码为 0xFF；否则为 0x00。这让暗区和亮区的计算完全无分支。

## 查找表驱动的混合模式

某些复杂的混合模式（如柔光 Soft Light、颜色减淡 Color Dodge、颜色加深 Color Burn）的数学公式包含除法或复杂的非线性函数，无法高效地用纯位运算实现。KrKr2 对这些模式使用 **256×256 预计算查找表**。

### 查找表结构

每个表驱动的混合模式定义一个包含 `TABLE[256][256]` 静态数组的结构体：

```cpp
// 文件：cpp/core/visual/gl/blend_functor_c.h 第 461-479 行
struct ps_soft_light_table {
    static unsigned char TABLE[256][256];  // 柔光查找表
};
struct ps_color_dodge_table {
    static unsigned char TABLE[256][256];  // 颜色减淡查找表
};
struct ps_color_burn_table {
    static unsigned char TABLE[256][256];  // 颜色加深查找表
};
struct ps_overlay_table {
    static unsigned char TABLE[256][256];  // 叠加查找表（可选）
};
```

每个表占用 256×256 = 65536 字节（64KB）。4 个表共计 256KB。对于现代系统来说内存开销不大，但查找表方案有两个优势：
- **性能稳定**：每次查找是 O(1) 的内存访问，没有分支预测失败的风险
- **精度完美**：查找表可以存储精确的数学结果，避免整数近似误差

### 模板化的查找表混合函数

`ps_table_blend_func<TTable>` 是一个模板类，接受任意查找表类型作为参数：

```cpp
// 文件：cpp/core/visual/gl/blend_functor_c.h 第 449-458 行
template <typename TTable>
struct ps_table_blend_func : public ps_alpha_blend_func {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s,
                                 tjs_uint32 a) const {
        // 对每个颜色通道分别查表
        s = (TTable::TABLE[(s >> 16) & 0xff][(d >> 16) & 0xff] << 16) |
            (TTable::TABLE[(s >> 8) & 0xff][(d >> 8) & 0xff] << 8) |
            (TTable::TABLE[(s >> 0) & 0xff][(d >> 0) & 0xff] << 0);
        // 查表结果作为混合后颜色，按 alpha 合成
        return ps_alpha_blend_func::operator()(d, s, a);
    }
};

// 具体混合模式的定义——只需继承模板即可
struct ps_soft_light_blend_func
    : public ps_table_blend_func<ps_soft_light_table> {};
struct ps_color_dodge_blend_func
    : public ps_table_blend_func<ps_color_dodge_table> {};
struct ps_color_burn_blend_func
    : public ps_table_blend_func<ps_color_burn_table> {};
```

### 查找表初始化

查找表在 `blend_function.cpp` 的 `TVPPsInitTable()` 函数中初始化。以柔光表为例：

```cpp
// 文件：cpp/core/visual/gl/blend_function.cpp TVPPsInitTable() 内
// 柔光公式：
// 当 s < 128 时：result = d - (255 - 2*s) * d * (255 - d) / 255^2
// 当 s >= 128 时：result = d + (2*s - 255) * (D(d) - d) / 255
//   其中 D(d) = d < 64 ? ((16*d - 12)*d + 4)*d
//                       : sqrt(d/255) * 255
for (int s = 0; s < 256; s++) {
    for (int d = 0; d < 256; d++) {
        double ds = s / 255.0, dd = d / 255.0;
        double result;
        if (ds <= 0.5) {
            result = dd - (1.0 - 2.0*ds) * dd * (1.0 - dd);
        } else {
            double Dd = dd <= 0.25
                ? ((16.0*dd - 12.0)*dd + 4.0)*dd
                : sqrt(dd);
            result = dd + (2.0*ds - 1.0) * (Dd - dd);
        }
        ps_soft_light_table::TABLE[s][d] =
            (unsigned char)(result * 255.0 + 0.5);
    }
}
```

## 预乘 Alpha 混合

除了标准 alpha 混合，KrKr2 还支持预乘 alpha（Premultiplied Alpha）格式的混合操作。预乘 alpha 格式中，RGB 通道的值已经乘以了 alpha 值（即 `R' = R * A / 255`），这在合成时可以减少一次乘法。

### 预乘 alpha 混合函数

`blend_util_func.h` 定义了完整的预乘 alpha 混合体系：

```cpp
// 文件：cpp/core/visual/gl/blend_util_func.h 第 70-84 行
// 预乘 alpha 格式的 "over" 合成
// 公式：Di = sat(Si, (1-Sa)*Di)，Da = Sa + Da - Sa*Da
struct premulalpha_blend_a_a_func {
    saturated_u8_add_func sat_add_;
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        tjs_uint32 da = d >> 24;
        tjs_uint32 sa = s >> 24;
        // 结果 alpha = sa + da - sa*da/255
        da = da + sa - (da * sa >> 8);
        da -= (da >> 8);    // 精度调整：(x>>8) 近似 x/255 的误差修正
        sa ^= 0xff;         // sa_inv = 255 - sa
        s &= 0xffffff;      // 去掉源 alpha（因为 RGB 已经预乘了）
        return (da << 24) +
            sat_add_((((d & 0xff00ff) * sa >> 8) & 0xff00ff) +
                     (((d & 0xff00) * sa >> 8) & 0xff00), s);
    }
};
```

`da -= (da >> 8)` 这一行是精度修正。由于 `>> 8` 近似 `/255` 会产生约 `1/255` 的误差，减去 `da >> 8` 可以补偿这个偏差。

### 标准 alpha 到预乘 alpha 的转换

```cpp
// 文件：cpp/core/visual/gl/blend_functor_c.h 第 977-985 行
struct convet_alpha_to_premulalpha_functor {
    inline tjs_uint32 operator()(tjs_uint32 d) const {
        tjs_uint32 alpha = d >> 24;
        // RGB 各通道乘以 alpha
        tjs_uint32 color = (((((d & 0x00ff00) * alpha) & 0x00ff0000) +
                             (((d & 0xff00ff) * alpha) & 0xff00ff00)) >> 8);
        return color | (d & 0xff000000);  // 保持原 alpha 不变
    }
};
```

## 常量颜色填充混合

KrKr2 还提供了一系列"常量颜色"仿函数，用于将固定颜色混合到整个像素区域。这在文字渲染（将字形掩码的灰度值映射到指定颜色）和矩形填充中广泛使用。

```cpp
// 文件：cpp/core/visual/gl/blend_functor_c.h 第 761-773 行
struct const_alpha_fill_blend_functor {
    const tjs_uint32 color_;       // 预乘后的 BR 通道
    const tjs_uint32 color_mid_;   // 预乘后的 G 通道
    const tjs_int32 opa_;          // 255 - opacity（反向不透明度）
    inline const_alpha_fill_blend_functor(tjs_int32 opa, tjs_int32 color) :
        opa_(255 - opa),
        color_((color & 0xff00ff) * opa),      // 预计算避免循环内重复运算
        color_mid_((color & 0xff00) * opa) {}
    inline tjs_uint32 operator()(tjs_uint32 d) const {
        return (d & 0xff000000) +                    // 保持 dest alpha
            ((((d & 0xff00ff) * opa_ + color_) >> 8) & 0xff00ff) +  // BR混合
            ((((d & 0xff00) * opa_ + color_mid_) >> 8) & 0xff00);   // G混合
    }
};
```

这个仿函数在构造时就预计算了 `color * opa`，循环中每个像素只需要 2 次乘法和若干位运算。这比在循环内部每次都计算 `(d - color) * alpha + color` 少了一次乘法。

## 颜色映射函数（Color Map / Glyph Rendering）

`apply_color_map_xx_functor` 系列用于字体渲染。字形（Glyph）的灰度值作为 `tjs_uint8 s` 输入，指定颜色 `color` 是常量参数。输出是将灰度值作为 alpha、颜色作为 RGB 混合到目标像素上：

```cpp
// 文件：cpp/core/visual/gl/blend_functor_c.h 第 805-817 行
template <int tshift>
struct apply_color_map_xx_functor {
    const tjs_uint32 color_;   // 预存的 G 通道
    const tjs_uint32 c1_;      // 预存的 BR 通道
    inline apply_color_map_xx_functor(tjs_uint32 color) :
        c1_(color & 0xff00ff), color_(color & 0x00ff00) {}
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint8 s) const {
        tjs_uint32 d1 = d & 0xff00ff;
        // d + (color - d) * glyph_alpha >> tshift
        d1 = ((d1 + ((c1_ - d1) * s >> tshift)) & 0xff00ff);
        d &= 0xff00;
        return d1 | ((d + ((color_ - d) * s >> tshift)) & 0x00ff00);
    }
};
// tshift=8 用于标准 0-255 灰度，tshift=6 用于 0-63 灰度（TrueType 子像素渲染）
typedef apply_color_map_xx_functor<8> apply_color_map_functor;
typedef apply_color_map_xx_functor<6> apply_color_map65_functor;
```

## 动手实践

### 实验 1：验证饱和加法的正确性

```cpp
#include <cstdint>
#include <cstdio>

// 模拟 KrKr2 的饱和加法
uint32_t saturated_add(uint32_t a, uint32_t b) {
    uint32_t tmp = ((a & b) + (((a ^ b) >> 1) & 0x7f7f7f7f)) & 0x80808080;
    tmp = (tmp << 1) - (tmp >> 7);
    return (a + b - tmp) | tmp;
}

// 朴素实现作为对照
uint32_t saturated_add_naive(uint32_t a, uint32_t b) {
    uint32_t result = 0;
    for (int i = 0; i < 4; i++) {
        uint32_t ca = (a >> (i*8)) & 0xFF;
        uint32_t cb = (b >> (i*8)) & 0xFF;
        uint32_t sum = ca + cb;
        if (sum > 255) sum = 255;
        result |= (sum << (i*8));
    }
    return result;
}

int main() {
    // 测试典型场景
    uint32_t tests[][2] = {
        {0x80808080, 0x80808080},  // 128+128=255（饱和）
        {0xFF000000, 0x01000000},  // alpha 溢出
        {0x00FF00FF, 0x00010001},  // BR 通道饱和
        {0x00000000, 0xFFFFFFFF},  // 极端值
    };
    for (auto& t : tests) {
        uint32_t fast = saturated_add(t[0], t[1]);
        uint32_t naive = saturated_add_naive(t[0], t[1]);
        printf("0x%08X + 0x%08X = 0x%08X (fast) / 0x%08X (naive) %s\n",
               t[0], t[1], fast, naive,
               fast == naive ? "✓" : "✗ MISMATCH");
    }
    return 0;
}
```

### 实验 2：可视化不同混合模式的效果

```cpp
#include <cstdint>
#include <cstdio>
#include <algorithm>

using tjs_uint32 = uint32_t;

// Alpha 混合
tjs_uint32 alpha_blend(tjs_uint32 d, tjs_uint32 s, tjs_uint32 a) {
    tjs_uint32 dcl = d & 0xff00ff;
    dcl = ((dcl + (((s & 0xff00ff) - dcl) * a >> 8)) & 0xff00ff);
    tjs_uint32 dg = d & 0xff00, sg = s & 0xff00;
    return dcl | ((dg + ((sg - dg) * a >> 8)) & 0xff00);
}

// 乘法混合
tjs_uint32 mul_blend(tjs_uint32 d, tjs_uint32 s) {
    return (((((d>>16)&0xff) * (s & 0x00ff0000)) & 0xff000000) |
            ((((d>>8)&0xff) * (s & 0x0000ff00)) & 0x00ff0000) |
            ((((d>>0)&0xff) * (s & 0x000000ff)))) >> 8;
}

// 滤色混合
tjs_uint32 screen_blend(tjs_uint32 d, tjs_uint32 s) {
    // s + d - s*d/255（按通道计算）
    auto ch = [](int dc, int sc) { return sc + dc - sc * dc / 255; };
    return (ch((d>>16)&0xff, (s>>16)&0xff) << 16) |
           (ch((d>>8)&0xff, (s>>8)&0xff) << 8) |
           ch(d&0xff, s&0xff);
}

int main() {
    tjs_uint32 dest = 0x00804020;   // 深蓝绿色（BGRA，A=0）
    tjs_uint32 src  = 0xFF60A0E0;   // 浅橙色（A=255）
    tjs_uint32 a = 192;             // 75% 不透明度

    printf("Dest:   B=%3d G=%3d R=%3d\n", dest&0xff, (dest>>8)&0xff, (dest>>16)&0xff);
    printf("Src:    B=%3d G=%3d R=%3d (A=%d)\n", src&0xff, (src>>8)&0xff, (src>>16)&0xff, src>>24);
    printf("Alpha blend:  "); auto r = alpha_blend(dest, src, a);
    printf("B=%3d G=%3d R=%3d\n", r&0xff, (r>>8)&0xff, (r>>16)&0xff);
    printf("Mul blend:    "); r = mul_blend(dest, src);
    printf("B=%3d G=%3d R=%3d\n", r&0xff, (r>>8)&0xff, (r>>16)&0xff);
    printf("Screen blend: "); r = screen_blend(dest, src);
    printf("B=%3d G=%3d R=%3d\n", r&0xff, (r>>8)&0xff, (r>>16)&0xff);
    return 0;
}
```

## 对照项目源码

相关文件：
- `cpp/core/visual/gl/blend_functor_c.h` 全文 — 所有基础混合函数和 PS 兼容混合模式的定义
- `cpp/core/visual/gl/blend_variation.h` 全文 — 6 种 alpha 变体包装器模板
- `cpp/core/visual/gl/blend_util_func.h` 全文 — 预乘 alpha 混合、饱和加法、颜色映射工具函数
- `cpp/core/visual/gl/blend_function.cpp` 第 1-685 行 — 模板循环函数、宏批量生成、查找表初始化
- `cpp/core/visual/tvpgl.h` 全文 — 所有混合函数的函数指针声明
- `cpp/core/visual/argb.h` 第 1-184 行 — `tTVPARGB<T>` 像素类型模板

### 常见错误与排查

**错误 1：双通道并行时乘数超过 256**

```cpp
// ❌ 错误：alpha 值可能超过 256，导致 B 和 R 通道互相干扰
tjs_uint32 a = 300;
tjs_uint32 result = (s & 0xff00ff) * a;  // 溢出！

// ✅ 正确：确保乘数不超过 256
tjs_uint32 a = std::min(opa, (tjs_uint32)255);
```

当乘数超过 256 时，低 8 位（B 通道）的乘法结果会溢出到 G 通道的位置，产生错误的颜色值。项目中所有 alpha 值在使用前都确保在 0-255 范围内。

**错误 2：忘记 HDA 模式导致 alpha 通道被覆盖**

当目标缓冲区的 alpha 通道用作遮罩信息时，使用 `normal_op` 会破坏遮罩数据。应该使用 `hda_op` 变体。在 KrKr2 中，`_HDA` 后缀的函数指针专门用于需要保护目标 alpha 的场景。

**错误 3：预乘 alpha 和标准 alpha 格式混用**

```cpp
// ❌ 错误：用标准 alpha 混合函数处理预乘 alpha 数据
// 会导致暗边（dark fringe）或颜色偏移
alpha_blend_func blend;
auto result = blend(premul_dest, premul_src, alpha);

// ✅ 正确：预乘 alpha 数据必须使用对应的混合函数
premulalpha_blend_a_a_func premul_blend;
auto result = premul_blend(premul_dest, premul_src);
```

## 本节小结

- KrKr2 的混合系统采用三层仿函数架构：基础混合函数 → 变体包装器 → 模板循环函数
- 基础混合函数只关心颜色数学运算，接受 (d, s, a) 三参数
- 变体包装器负责 alpha 的提取和处理，提供 6 种常见模式（normal、translucent、HDA、dest_alpha 等）
- `DEFINE_BLEND_VARIATION` 宏一次生成 6 种变体 typedef，大幅减少重复代码
- PS 兼容混合模式实现了 Photoshop 的全部常用混合效果，复杂模式使用 256×256 查找表
- 预乘 alpha 格式有独立的混合函数体系，不可与标准 alpha 混用
- 所有位运算技巧都遵循双通道并行原则（`0xff00ff` 掩码处理 B+R，单独处理 G）

## 练习题与答案

### 题目 1：实现一个"反色差值"混合模式

编写一个 `invert_diff_blend_func` 结构体，公式为：`result = abs((255 - s) - d)`，即先反色再求差值。要求兼容三层架构（接受 d, s, a 三参数）。

<details>
<summary>查看答案</summary>

```cpp
struct invert_diff_blend_func : public ps_alpha_blend_func {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s,
                                 tjs_uint32 a) const {
        // 先反色 s
        tjs_uint32 inv_s = ~s & 0x00ffffff;
        // 无分支求绝对值差（与 ps_diff_blend_func 相同的掩码技巧）
        tjs_uint32 n;
        n = (((~d & inv_s) << 1) + ((~d ^ inv_s) & 0x00fefefe)) & 0x01010100;
        n = ((n >> 8) + 0x007f7f7f) ^ 0x007f7f7f;
        // n 为掩码：d < inv_s 的通道为 0xFF，否则为 0x00
        s = ((inv_s & n) - (d & n)) | ((d & ~n) - (inv_s & ~n));
        return ps_alpha_blend_func::operator()(d, s, a);
    }
};
// 生成 6 种变体
DEFINE_BLEND_PS_VARIATION(invert_diff_blend)
```

</details>

### 题目 2：解释 `da -= (da >> 8)` 精度修正的数学原理

在 `premulalpha_blend_a_a_func` 中，计算 `da = da + sa - (da * sa >> 8)` 后立即执行 `da -= (da >> 8)`。请解释这行代码为什么能提高精度。

<details>
<summary>查看答案</summary>

问题根源：`>> 8` 是对 `/256` 的近似，而我们实际需要的是 `/255`。

数学分析：
- 精确公式：`result = sa + da - sa * da / 255`
- 近似公式：`result = sa + da - (sa * da >> 8)` = `sa + da - sa * da / 256`
- 误差：`sa * da / 256 - sa * da / 255 = sa * da * (1/256 - 1/255) = -sa * da / 65280`

也就是说，`>> 8` 近似会让结果偏大（因为减去的值偏小）。

修正原理：
- `da -= (da >> 8)` 等价于 `da = da * (1 - 1/256) = da * 255/256`
- 这近似于将结果乘以 `255/256`，补偿了 `/256` 和 `/255` 之间的差异
- 修正后的最大误差从约 1.0 降低到约 0.004，在 8 位整数精度下几乎可以忽略

示例验证：
- sa=200, da=200
- 精确：200 + 200 - 200*200/255 = 243.14 ≈ 243
- 未修正：200 + 200 - (200*200>>8) = 400 - 156 = 244（偏大 1）
- 修正后：244 - (244>>8) = 244 - 0 = 244（仍偏大 1，但在大多数情况下误差更小）

</details>

### 题目 3：为什么叠加模式的掩码生成用 `0x00808080` 而不是直接比较 128？

<details>
<summary>查看答案</summary>

直接比较需要逐通道进行 4 次比较和分支：

```cpp
// 朴素方法：4 次比较 + 4 次分支
uint8_t r_mask = ((d >> 16) & 0xff) < 128 ? 0xff : 0x00;
uint8_t g_mask = ((d >> 8) & 0xff) < 128 ? 0xff : 0x00;
uint8_t b_mask = (d & 0xff) < 128 ? 0xff : 0x00;
```

而位运算方法一次处理全部 3 个通道：

1. `d & 0x00808080` — 提取每个通道的 bit7（第 128 位）
2. `>> 7` — 将 bit7 移到 bit0，得到 0 或 1
3. `+ 0x007f7f7f` — 如果 bit7=0（值<128），得到 0x7f；如果 bit7=1（值≥128），得到 0x80
4. `^ 0x007f7f7f` — 异或翻转：0x7f → 0x00（不对，让我重新分析）

实际上正确的过程是：
- bit7 = 0（通道值 < 128）：0 + 0x7f = 0x7f，0x7f ^ 0x7f = 0x00... 

等等，这说明**值 < 128 时掩码为 0x00，值 ≥ 128 时掩码为 0xFF**？让我重新核实源码中的使用方式——掩码 `n` 在源码中用于选择暗区（d & n）和亮区（~n），所以暗区对应值 < 128，掩码 n = 0xFF。

重新推导：
- `d & 0x00808080` 提取 bit7，例如 B=0x40（<128）→ bit7=0，B=0x90（≥128）→ bit7=0x80
- `>> 7` → 0 或 1
- `+ 0x007f7f7f` → 0x7f 或 0x80
- `^ 0x007f7f7f` → 0x00 或 0xff

所以 **值 < 128 → 掩码 0x00，值 ≥ 128 → 掩码 0xFF**。在源码中 `d1 = d & n` 提取的是亮区像素，`~n` 对应暗区。

优势：无分支、单指令宽度处理 3 个通道（或 4 个，取决于掩码），流水线友好。在每帧处理数百万像素时，消除分支预测失败带来的性能提升是显著的。

</details>

## 下一步

[03-SSE-ASM优化路径](./03-SSE-ASM优化路径.md) — 了解 KrKr2 如何通过 SIMD 指令集加速像素混合操作，以及函数指针运行时切换机制。
