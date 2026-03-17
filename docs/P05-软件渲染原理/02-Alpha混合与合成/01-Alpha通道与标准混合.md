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

