# Alpha 通道与标准混合

> **所属模块：** P05-软件渲染原理
> **前置知识：** [01-像素本质与格式](../01-像素与颜色空间/01-像素本质与格式.md)、[02-预乘Alpha与颜色空间](../01-像素与颜色空间/02-预乘Alpha与颜色空间.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 Alpha 混合（Alpha Blending，将前景半透明叠加到背景上）的数学原理
2. 手写 SrcOver（Source Over，源覆盖目标——最常用的图层叠加方式）公式的整数实现
3. 掌握双通道并行位运算技巧——一次乘法同时处理 R 和 B 两个颜色通道
4. 理解 HDA（Hold Destination Alpha，保持目标Alpha不变）变体的用途和实现
5. 理解 Opacity（全局不透明度，控制整个图层的透明程度）的叠加机制
6. 读懂 KrKr2 中 `alpha_blend_func`、`translucent_op`、`hda_op` 等核心代码

---

## 1. Alpha 混合是什么

### 1.1 问题场景

游戏画面通常由多个图层叠加而成：最底层是背景图，上面叠加角色立绘，再上面叠加对话框、文字、UI 按钮。每个图层的像素都可能是半透明的——比如对话框边缘的渐变淡出、角色立绘的抗锯齿（Anti-aliasing，让边缘平滑而非锯齿状的技术）边缘。

**Alpha 混合就是将半透明前景像素叠加到背景像素上，计算出最终颜色的过程。**

```
背景（不透明蓝色）    前景（半透明红色）      混合结果
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  R=0         │    │  R=255       │    │  R=128       │
│  G=0         │ +  │  G=0         │ =  │  G=0         │
│  B=255       │    │  B=0         │    │  B=128       │
│  A=255(不透明)│    │  A=128(50%)  │    │  紫色        │
└──────────────┘    └──────────────┘    └──────────────┘
```

### 1.2 数学公式：Over 运算

最常用的混合方式是 **SrcOver（源覆盖）**运算。数学上，对每个颜色通道（R、G、B）：

```
C_out = C_src × α_src + C_dst × (1 - α_src)
```

其中：
- `C_src`：前景（源）像素的颜色值
- `C_dst`：背景（目标）像素的颜色值
- `α_src`：前景像素的不透明度，范围 [0.0, 1.0]
- `C_out`：混合后的输出颜色

这个公式的直觉理解：**前景贡献 α 比例的颜色，背景贡献 (1-α) 比例的颜色**。当 α=1 时完全覆盖背景，α=0 时背景完全保留。

### 1.3 整数化：从浮点到位运算

在实际像素处理中，Alpha 值不是浮点数而是 0~255 的整数（0 对应 0.0，255 对应 1.0）。公式需要转换为整数运算：

```cpp
// 最朴素的逐通道实现
uint8_t blend_channel(uint8_t src, uint8_t dst, uint8_t alpha) {
    // src * alpha / 255 + dst * (255 - alpha) / 255
    return (src * alpha + dst * (255 - alpha)) / 255;
}
```

但这有两个性能问题：
1. **除以 255 很慢**——整数除法在多数 CPU 上需要 20~40 个时钟周期
2. **逐通道处理**——处理一个 ARGB 像素需要调用 3 次（R、G、B 各一次）

KrKr2 用两个关键优化解决这些问题：用 `>> 8`（右移 8 位，即除以 256）代替 `/ 255`，以及用位掩码技巧将两个通道合并为一次运算。

---

## 2. KrKr2 的 Alpha 混合实现

### 2.1 核心代码：alpha_blend_func

KrKr2 中最基础的 Alpha 混合函数是 `alpha_blend_func`，定义在 `blend_functor_c.h` 第 67-76 行。这是一个仿函数（Functor，即重载了 `operator()` 的结构体，可以像函数一样调用）：

```cpp
// 文件: cpp/core/visual/gl/blend_functor_c.h, 第 67-76 行
struct alpha_blend_func {
    inline tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s,
                                 tjs_uint32 a) const {
        // d = 目标像素 (ARGB), s = 源像素 (ARGB), a = alpha [0..255]
        tjs_uint32 d1 = d & 0xff00ff;
        // 对 R 和 B 两个通道并行计算线性插值
        d1 = (d1 + (((s & 0xff00ff) - d1) * a >> 8)) & 0xff00ff;
        d &= 0xff00;   // 提取 G 通道
        s &= 0xff00;   // 提取 G 通道
        return d1 + ((d + ((s - d) * a >> 8)) & 0xff00);
    }
};
```

这段代码只有 5 行运算，却同时处理了 R、G、B 三个通道的混合。下面逐步拆解。

### 2.2 线性插值的变形

标准公式 `C_out = C_src × α + C_dst × (1 - α)` 可以变形为：

```
C_out = C_dst + (C_src - C_dst) × α
```

这是线性插值（Linear Interpolation，也叫 Lerp）的标准形式，好处是只需要**一次乘法**（原公式需要两次）。

在整数域中：

```cpp
// 单通道线性插值
uint8_t result = dst + (src - dst) * alpha / 256;
// 注意 (src - dst) 可能是负数，但在位运算中不影响结果
```

### 2.3 双通道并行技巧（核心优化）

这是 KrKr2 像素运算中最精妙的技巧。观察 ARGB 像素的内存布局：

```
一个 32 位 ARGB 像素:
bit 31..24  bit 23..16  bit 15..8  bit 7..0
    A           R           G         B
```

用掩码 `0x00FF00FF` 提取 R 和 B：

```
原始:     0x  AA  RR  GG  BB
& 0x00FF00FF:
结果:     0x  00  RR  00  BB
               ↑       ↑
             R在高16位   B在低8位
```

关键洞察：**R 在第 16~23 位，B 在第 0~7 位，中间隔着 8 位的间隙**。当我们执行 `value * alpha`（alpha 最大 255）时：

```
    0x  00  RR  00  BB
  × alpha (最大 255)
  = 0x  00  RR×α 00  BB×α

  R×α 最大 = 255×255 = 65025 = 0xFE01，占 16 位
  B×α 最大 = 255×255 = 65025 = 0xFE01，占 16 位
  
  R×α 存放在 bit 16~31
  B×α 存放在 bit 0~15
  它们不会互相溢出！（中间有 8 位间隙提供了缓冲）
```

所以**一次 32 位乘法就同时处理了 R 和 B 两个通道**，速度翻倍。

完整的双通道并行计算流程：

```cpp
tjs_uint32 d_rb = d & 0xff00ff;           // 提取 dst 的 R+B
tjs_uint32 s_rb = s & 0xff00ff;           // 提取 src 的 R+B
tjs_uint32 diff = s_rb - d_rb;            // 计算差值（可能为负）
tjs_uint32 blended = diff * a >> 8;       // 差值 × alpha / 256
tjs_uint32 result = (d_rb + blended) & 0xff00ff;  // 加回基准，掩码裁剪
```

最后的 `& 0xff00ff` 是必要的——它清除了 R 和 B 之间的"缓冲区"中可能残留的溢出位。

### 2.4 G 通道单独处理

G 通道在第 8~15 位，夹在 R 和 B 之间，无法与其他通道合并计算。所以单独处理：

```cpp
d &= 0xff00;   // 只保留 G（bit 8~15）
s &= 0xff00;   // 只保留 G
// 同样的线性插值，但只操作 G 通道
tjs_uint32 g_result = (d + ((s - d) * a >> 8)) & 0xff00;
```

### 2.5 完整流程图

```
输入: dst = 0xAARRGGBB, src = 0xAARRGGBB, alpha = 0~255

步骤 1: 拆分通道
  d_rb = dst & 0x00FF00FF  →  0x00RR00BB
  d_g  = dst & 0x0000FF00  →  0x0000GG00
  s_rb = src & 0x00FF00FF  →  0x00RR00BB
  s_g  = src & 0x0000FF00  →  0x0000GG00

步骤 2: 并行计算 R+B
  d_rb = d_rb + ((s_rb - d_rb) * alpha >> 8)
  d_rb &= 0x00FF00FF   ← 裁剪溢出位

步骤 3: 单独计算 G
  d_g = d_g + ((s_g - d_g) * alpha >> 8)
  d_g &= 0x0000FF00    ← 裁剪溢出位

步骤 4: 合并
  result = d_rb | d_g   →  0x00RRGGBB
```

### 2.6 完整可运行示例

```cpp
#include <cstdint>
#include <cstdio>

using tjs_uint32 = uint32_t;

// KrKr2 风格的 Alpha 混合（双通道并行）
tjs_uint32 alpha_blend(tjs_uint32 d, tjs_uint32 s, tjs_uint32 a) {
    // R+B 并行
    tjs_uint32 d1 = d & 0xff00ff;
    d1 = (d1 + (((s & 0xff00ff) - d1) * a >> 8)) & 0xff00ff;
    // G 单独
    d &= 0xff00;
    s &= 0xff00;
    return d1 + ((d + ((s - d) * a >> 8)) & 0xff00);
}

// 朴素逐通道实现（用于对照验证）
tjs_uint32 alpha_blend_naive(tjs_uint32 d, tjs_uint32 s, tjs_uint32 a) {
    auto blend = [a](uint8_t dc, uint8_t sc) -> uint8_t {
        return (uint8_t)(dc + (int)(sc - dc) * (int)a / 256);
    };
    uint8_t r = blend((d >> 16) & 0xff, (s >> 16) & 0xff);
    uint8_t g = blend((d >> 8) & 0xff, (s >> 8) & 0xff);
    uint8_t b = blend(d & 0xff, s & 0xff);
    return (r << 16) | (g << 8) | b;
}

void print_pixel(const char* label, tjs_uint32 p) {
    printf("%-12s: R=%3d G=%3d B=%3d  (0x%06X)\n", label,
           (p >> 16) & 0xff, (p >> 8) & 0xff, p & 0xff, p & 0xffffff);
}

int main() {
    tjs_uint32 dst = 0xFF0000FF;  // 不透明蓝色
    tjs_uint32 src = 0x80FF0000;  // 半透明红色
    tjs_uint32 alpha = 128;       // 50% 不透明

    printf("=== Alpha 混合对比验证 ===\n");
    print_pixel("DST (蓝)", dst);
    print_pixel("SRC (红)", src);
    printf("Alpha = %u (%.0f%%)\n\n", alpha, alpha / 255.0 * 100);

    tjs_uint32 fast = alpha_blend(dst, src, alpha);
    tjs_uint32 naive = alpha_blend_naive(dst, src, alpha);

    print_pixel("位运算版", fast);
    print_pixel("朴素版", naive);
    printf("结果一致: %s\n", fast == naive ? "是" : "否");

    // 测试边界情况
    printf("\n=== 边界测试 ===\n");
    print_pixel("alpha=0", alpha_blend(dst, src, 0));    // 应等于 dst
    print_pixel("alpha=255", alpha_blend(dst, src, 255)); // 应接近 src

    return 0;
}
```

**CMakeLists.txt**：

```cmake
cmake_minimum_required(VERSION 3.16)
project(alpha_blend_demo LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
add_executable(alpha_blend_demo main.cpp)
```

```bash
# Windows
cmake -B build -G Ninja && cmake --build build && build\alpha_blend_demo.exe

# Linux / macOS
cmake -B build -G Ninja && cmake --build build && ./build/alpha_blend_demo
```

预期输出：

```
=== Alpha 混合对比验证 ===
DST (蓝)    : R=  0 G=  0 B=255  (0x0000FF)
SRC (红)    : R=255 G=  0 B=  0  (0xFF0000)
Alpha = 128 (50%)

位运算版    : R=128 G=  0 B=127  (0x80007F)
朴素版      : R=128 G=  0 B=127  (0x80007F)
结果一致: 是

=== 边界测试 ===
alpha=0     : R=  0 G=  0 B=255  (0x0000FF)
alpha=255   : R=255 G=  0 B=  0  (0xFF0000)
```

---

## 3. HDA 变体：保持目标 Alpha

### 3.1 什么是 HDA

HDA 全称 **Hold Destination Alpha**（保持目标Alpha），意思是在颜色混合时**不修改目标像素的 Alpha 通道**。

为什么需要这个？在多图层合成系统中，每个图层本身也有 Alpha 通道记录该图层的透明度信息。当我们把一个前景图层混合到目标图层上时，通常只想改变目标的**颜色**（RGB），而不想影响目标自身的**透明度信息**（Alpha）。

### 3.2 实现方式

HDA 的实现非常简单——混合后把目标原来的 Alpha 值放回去：

```cpp
// HDA: 混合 RGB，但保持 dst 的 Alpha 不变
tjs_uint32 alpha_blend_hda(tjs_uint32 d, tjs_uint32 s, tjs_uint32 a) {
    tjs_uint32 result = alpha_blend(d, s, a);           // 正常混合
    return (result & 0x00ffffff) | (d & 0xff000000);    // 替换回原 alpha
}
```

### 3.3 KrKr2 中的 HDA 模板

KrKr2 用模板类 `hda_op` 包装任意混合函数，自动添加 HDA 行为：

```cpp
// 文件: cpp/core/visual/gl/blend_variation.h
template<typename FUNC>
struct hda_op {
    FUNC func_;
    tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        tjs_uint32 result = func_(d, s);
        // 保留 dst 的高 8 位（Alpha）
        return (result & 0x00ffffff) | (d & 0xff000000);
    }
};
```

这样，任何混合 functor 都可以通过 `hda_op<xxx_func>` 获得 HDA 变体，无需重写代码。

### 3.4 实际使用场景

```
场景：角色立绘叠加到背景上

背景图层 (dst):
  - RGB = 天空蓝色
  - Alpha = 255（完全不透明）
  
角色立绘 (src):
  - RGB = 角色颜色
  - Alpha = 各种值（边缘渐变）

不使用 HDA:
  混合后背景的 Alpha 可能被修改 → 后续图层看到的"背景透明度"错误

使用 HDA (bmAlpha + HDA):
  混合后背景的 Alpha 保持 255 → 后续图层叠加时背景仍然完全不透明
```

---

## 4. Opacity：全局不透明度

### 4.1 概念说明

除了每个像素自身的 Alpha 值，KrKr2 还支持给**整个图层**指定一个全局不透明度（Opacity），范围 0~255。Opacity 和逐像素 Alpha 的关系是**相乘**：

```
实际 alpha = 像素 alpha × 图层 opacity / 255
```

例如：一个像素 Alpha=200（约 78%），图层 Opacity=128（约 50%），则实际不透明度 ≈ 200×128/255 ≈ 100（约 39%）。

### 4.2 KrKr2 的 translucent_op 模板

在 KrKr2 中，带 `_o` 后缀的函数处理 opacity。实现方式是用模板包装：

```cpp
// 文件: cpp/core/visual/gl/blend_variation.h
// translucent_op: 在执行混合前先将 alpha 乘以 opacity
template<typename FUNC>
struct translucent_op {
    FUNC func_;
    tjs_int opa_;                              // 全局不透明度 [0..255]
    translucent_op(tjs_int opa) : opa_(opa) {}
    tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        tjs_uint32 a = (s >> 24) * opa_ >> 8;  // 像素alpha × opacity / 256
        return func_(d, s, a);                  // 使用调制后的 alpha 执行混合
    }
};
```

### 4.3 完整示例：带 Opacity 的混合

```cpp
#include <cstdint>
#include <cstdio>

using tjs_uint32 = uint32_t;

// Alpha 混合核心（同前）
tjs_uint32 alpha_blend(tjs_uint32 d, tjs_uint32 s, tjs_uint32 a) {
    tjs_uint32 d1 = d & 0xff00ff;
    d1 = (d1 + (((s & 0xff00ff) - d1) * a >> 8)) & 0xff00ff;
    d &= 0xff00;
    s &= 0xff00;
    return d1 + ((d + ((s - d) * a >> 8)) & 0xff00);
}

// 带 opacity 的混合（模拟 translucent_op）
tjs_uint32 alpha_blend_with_opacity(tjs_uint32 d, tjs_uint32 s,
                                     tjs_uint32 opacity) {
    tjs_uint32 pixel_alpha = s >> 24;              // 提取像素 alpha
    tjs_uint32 effective_alpha = pixel_alpha * opacity >> 8;  // 实际 alpha
    return alpha_blend(d, s, effective_alpha);
}

void print_pixel(const char* label, tjs_uint32 p) {
    printf("%-20s: R=%3d G=%3d B=%3d  (0x%06X)\n", label,
           (p >> 16) & 0xff, (p >> 8) & 0xff, p & 0xff, p & 0xffffff);
}

int main() {
    tjs_uint32 dst = 0xFF00FF00;  // 不透明绿色
    tjs_uint32 src = 0xFFFF0000;  // 不透明红色，像素 alpha=255

    printf("=== Opacity 效果演示 ===\n");
    print_pixel("DST (绿)", dst);
    print_pixel("SRC (红, A=255)", src);
    printf("\n");

    // 不同 opacity 值的效果
    tjs_uint32 opacities[] = {255, 192, 128, 64, 0};
    for (tjs_uint32 opa : opacities) {
        char label[32];
        snprintf(label, sizeof(label), "opacity=%3d (%2d%%)", opa,
                 (int)(opa / 255.0 * 100 + 0.5));
        print_pixel(label, alpha_blend_with_opacity(dst, src, opa));
    }

    return 0;
}
```

预期输出：

```
=== Opacity 效果演示 ===
DST (绿)            : R=  0 G=255 B=  0  (0x00FF00)
SRC (红, A=255)     : R=255 G=  0 B=  0  (0xFF0000)

opacity=255 (100%)  : R=255 G=  0 B=  0  (0xFF0000)
opacity=192 ( 75%)  : R=192 G= 63 B=  0  (0xC03F00)
opacity=128 ( 50%)  : R=128 G=127 B=  0  (0x807F00)
opacity= 64 ( 25%)  : R= 64 G=191 B=  0  (0x40BF00)
opacity=  0 (  0%)  : R=  0 G=255 B=  0  (0x00FF00)
```

### 4.4 组合变体：HDA + Opacity

KrKr2 通过模板嵌套实现所有组合。例如 HDA + Opacity：

```cpp
// hda_translucent_op: 先乘 opacity，再保持 dst alpha
template<typename FUNC>
struct hda_translucent_op {
    FUNC func_;
    tjs_int opa_;
    hda_translucent_op(tjs_int opa) : opa_(opa) {}
    tjs_uint32 operator()(tjs_uint32 d, tjs_uint32 s) const {
        tjs_uint32 a = (s >> 24) * opa_ >> 8;
        tjs_uint32 result = func_(d, s, a);
        return (result & 0x00ffffff) | (d & 0xff000000);  // 保持 dst alpha
    }
};
```

这些模板最终被宏 `DEFINE_BLEND_VARIATION` 自动展开，为每种混合模式一次性生成所有变体（见下一节）。

---

## 5. >> 8 近似的精度分析

### 5.1 问题

上面所有代码都用 `>> 8`（除以 256）代替了理论上正确的 `/ 255`。这引入了多大误差？

```cpp
// 精确计算 vs 近似计算
int exact  = src * alpha / 255;     // 正确但慢
int approx = src * alpha >> 8;      // 快但有误差
// 最大误差出现在 src=255, alpha=255:
//   exact  = 255 * 255 / 255 = 255
//   approx = 255 * 255 / 256 = 254
// 相对误差 = 1/255 ≈ 0.4%
```

### 5.2 精确除以 255 的技巧

如果需要精确结果（比如图像编辑软件），可以用以下公式代替除法：

```cpp
// 精确的 x / 255（无除法指令）
// 原理：(x + 128) / 255 ≈ (x + 128 + ((x + 128) >> 8)) >> 8
inline uint32_t div255(uint32_t x) {
    return (x + 128 + ((x + 128) >> 8)) >> 8;
}
```

### 5.3 KrKr2 的选择

KrKr2 全局使用 `>> 8` 近似。原因：

1. **性能**：`>> 8` 是单条移位指令，`/ 255` 需要整数除法（20~40 周期）
2. **视觉不可察**：0.4% 的颜色误差人眼完全无法分辨
3. **一致性**：所有混合模式统一使用相同的近似，不会出现"模式 A 和模式 B 的同色叠加结果不同"
4. **SIMD 友好**：移位指令可以直接用 SSE2/NEON 的向量移位指令

---

## 6. 从像素到扫描线：批量混合

### 6.1 实际接口

前面展示的都是单像素混合。在实际引擎中，混合操作总是**批量处理整行像素**（一条扫描线，即图像的一行像素数据）：

```cpp
// 文件: cpp/core/visual/tvpgl.h
// 声明：批量 Alpha 混合函数（函数指针，支持运行时替换为 SIMD 版本）
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAlphaBlend,
    (tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len));

// 调用示例：将 src 行的 len 个像素混合到 dest 行
TVPAlphaBlend(dest_scanline, src_scanline, width);
```

### 6.2 C 语言默认实现

```cpp
// 文件: cpp/core/visual/tvpgl.cpp（简化）
void TVPAlphaBlend_c(tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len) {
    for (tjs_int i = 0; i < len; i++) {
        tjs_uint32 s = src[i];
        tjs_uint32 a = s >> 24;         // 提取源像素的 alpha
        if (a == 0) continue;            // 完全透明，跳过
        if (a == 255) {
            dest[i] = s;                 // 完全不透明，直接覆盖
            continue;
        }
        // 半透明，执行混合
        tjs_uint32 d = dest[i];
        tjs_uint32 d1 = d & 0xff00ff;
        d1 = (d1 + (((s & 0xff00ff) - d1) * a >> 8)) & 0xff00ff;
        d &= 0xff00;
        s &= 0xff00;
        dest[i] = d1 + ((d + ((s - d) * a >> 8)) & 0xff00);
    }
}
```

注意两个**短路优化**：
- `a == 0`（完全透明）：直接跳过，不做任何运算
- `a == 255`（完全不透明）：直接复制源像素，无需混合

这两个优化在实际游戏中非常有效——大部分像素要么完全透明（背景区域），要么完全不透明（角色/UI 的主体区域），真正需要混合的半透明像素占比通常不超过 20%。

### 6.3 函数指针与运行时分发

`TVPAlphaBlend` 是一个**函数指针**，启动时根据 CPU 能力指向不同实现：

```cpp
// 文件: cpp/core/visual/tvpgl.h
// 函数指针声明宏（展开后是一个全局函数指针变量）
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAlphaBlend,
    (tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len));

// 初始化时（tvpgl.cpp）:
// 默认指向 C 语言实现
TVPAlphaBlend = TVPAlphaBlend_c;
// 如果 CPU 支持 SSE2，替换为 SIMD 版本
if (cpu_has_sse2)
    TVPAlphaBlend = TVPAlphaBlend_sse2;
```

这种机制确保：同一份代码在高端机器上用 SIMD 加速，在低端机器上用 C 版本作为兜底（Fallback，备用方案）。第 4 章将详细讲解 SIMD 版本的实现。

---

## 动手实践

### 练习：实现带完整变体的混合器

创建一个程序，实现 4 种变体并验证结果：

```cpp
#include <cstdint>
#include <cstdio>

using tjs_uint32 = uint32_t;

// 基础 Alpha 混合
tjs_uint32 blend(tjs_uint32 d, tjs_uint32 s, tjs_uint32 a) {
    tjs_uint32 d1 = d & 0xff00ff;
    d1 = (d1 + (((s & 0xff00ff) - d1) * a >> 8)) & 0xff00ff;
    d &= 0xff00;
    s &= 0xff00;
    return d1 + ((d + ((s - d) * a >> 8)) & 0xff00);
}

// 变体 1: normal（使用像素自身 alpha）
tjs_uint32 blend_normal(tjs_uint32 d, tjs_uint32 s) {
    return blend(d, s, s >> 24);
}

// 变体 2: HDA（保持 dst alpha）
tjs_uint32 blend_hda(tjs_uint32 d, tjs_uint32 s) {
    tjs_uint32 r = blend(d, s, s >> 24);
    return (r & 0x00ffffff) | (d & 0xff000000);
}

// 变体 3: opacity（全局不透明度调制）
tjs_uint32 blend_opacity(tjs_uint32 d, tjs_uint32 s, tjs_uint32 opa) {
    tjs_uint32 a = (s >> 24) * opa >> 8;
    return blend(d, s, a);
}

// 变体 4: HDA + opacity
tjs_uint32 blend_hda_opacity(tjs_uint32 d, tjs_uint32 s, tjs_uint32 opa) {
    tjs_uint32 a = (s >> 24) * opa >> 8;
    tjs_uint32 r = blend(d, s, a);
    return (r & 0x00ffffff) | (d & 0xff000000);
}

void print_argb(const char* label, tjs_uint32 p) {
    printf("%-16s: A=%3d R=%3d G=%3d B=%3d\n", label,
           (p>>24)&0xff, (p>>16)&0xff, (p>>8)&0xff, p&0xff);
}

int main() {
    tjs_uint32 dst = 0x80008000;  // 半透明绿色 (A=128, G=128)
    tjs_uint32 src = 0xC0FF0000;  // 75%不透明红色 (A=192, R=255)

    printf("=== 混合变体对比 ===\n");
    print_argb("DST", dst);
    print_argb("SRC", src);
    printf("\n");

    print_argb("normal", blend_normal(dst, src));
    print_argb("HDA", blend_hda(dst, src));
    print_argb("opacity=128", blend_opacity(dst, src, 128));
    print_argb("HDA+opa=128", blend_hda_opacity(dst, src, 128));

    return 0;
}
```

编译运行后观察：
1. `normal` 的 Alpha 值被混合计算覆盖（不保留原 dst alpha）
2. `HDA` 的 Alpha 值保持为 dst 的 128
3. `opacity=128` 的混合程度减半（红色更淡）
4. `HDA+opa=128` 兼具两种效果

---

## 常见错误及解决方案

### 错误 1：混淆 Straight Alpha 和 Premultiplied Alpha 的混合公式

**症状**：画面出现黑边、半透明区域颜色偏暗。

**原因**：对预乘格式的像素使用了 Straight Alpha 的混合公式，导致 Alpha 被重复乘了两次。

```cpp
// ❌ 错误：对预乘像素用标准混合
tjs_uint32 premul_pixel = 0x80800000;  // 预乘格式的 50% 红色
// 标准混合公式会再乘一次 alpha → 颜色变为 ~25%

// ✅ 正确：预乘格式使用专用公式
// C_out = C_src + C_dst × (1 - α_src)
```

**规则**：在 KrKr2 中，`_a` 后缀的函数处理预乘源，`bmAddAlpha` 系列使用预乘混合。

### 错误 2：忘记 & 0xff00ff 裁剪

**症状**：画面出现随机闪烁的绿色条纹或噪点。

**原因**：双通道并行计算后不裁剪，R 通道的溢出位污染了 G 通道。

```cpp
// ❌ 错误：缺少掩码裁剪
d1 = d1 + (((s & 0xff00ff) - d1) * a >> 8);  // 可能溢出到 G 位

// ✅ 正确：必须裁剪
d1 = (d1 + (((s & 0xff00ff) - d1) * a >> 8)) & 0xff00ff;
```

### 错误 3：Alpha=0 或 Alpha=255 时未短路

**症状**：性能低于预期，逐像素混合速度慢。

**原因**：对完全透明或完全不透明的像素仍执行了完整的混合运算。

```cpp
// ✅ 正确：加入短路判断
tjs_uint32 a = s >> 24;
if (a == 0) continue;     // 透明，跳过
if (a == 255) { d = s; continue; }  // 不透明，直接覆盖
// 只有真正半透明的像素才执行混合
```

---

## 对照项目源码

| 文件 | 行号 | 内容 |
|------|------|------|
| `cpp/core/visual/gl/blend_functor_c.h` | 67-76 | `alpha_blend_func` — 核心混合 functor |
| `cpp/core/visual/gl/blend_variation.h` | — | `translucent_op` / `hda_op` / `hda_translucent_op` 模板 |
| `cpp/core/visual/tvpgl.h` | — | `TVPAlphaBlend` 等函数指针声明 |
| `cpp/core/visual/tvpgl.cpp` | — | C 语言默认实现 + 查找表初始化 |
| `cpp/core/visual/LayerBitmapIntf.h` | 28-61 | `tTVPBBBltMethod` 枚举（所有混合模式列表） |

**阅读建议**：打开 `blend_functor_c.h`，从第 67 行的 `alpha_blend_func` 开始阅读。理解了这个核心 functor 后，再看同文件中其他混合 functor（如 `add_blend_functor`、`sub_blend_functor`），你会发现它们都使用相同的双通道并行技巧。

---

## 本节小结

- **Alpha 混合**将半透明前景叠加到背景上，核心公式 `C_out = C_dst + (C_src - C_dst) × α`
- KrKr2 用**双通道并行**位运算一次处理 R+B 两个通道，利用 32 位整数中通道间的 8 位间隙避免溢出
- **HDA（Hold Destination Alpha）**在混合后恢复目标的 Alpha 值，用于保护图层透明度信息
- **Opacity（全局不透明度）**与逐像素 Alpha 相乘，实现图层级别的透明度控制
- `>> 8` 近似 `/ 255` 有 0.4% 最大误差，视觉上不可察，但性能提升显著
- 批量混合函数通过函数指针实现运行时分发，可在 C/SSE2/NEON 之间无缝切换

---

## 练习题与答案

### 题目 1：手动计算 Alpha 混合

给定 dst = `0xFF804020`（不透明暗橙色），src = `0x80FF0000`（半透明红色），alpha = 128。手动用双通道并行技巧计算混合结果。

<details>
<summary>查看答案</summary>

```
步骤 1: 提取 R+B
  d_rb = 0xFF804020 & 0x00FF00FF = 0x00800020
  s_rb = 0x80FF0000 & 0x00FF00FF = 0x00FF0000

步骤 2: 计算差值
  diff = s_rb - d_rb = 0x00FF0000 - 0x00800020 = 0x007FFFE0
  注意：B 通道 0x00 - 0x20 = 0xE0（在无符号域看是一个大数，
        但后续乘法和掩码会正确处理）

步骤 3: 乘以 alpha 并右移
  diff * 128 >> 8 = 0x007FFFE0 * 128 >> 8
  = 0x007FFFE0 * 0x80 >> 8
  R 部分: 0x7F * 0x80 = 0x3F80 → >> 8 = 0x3F
  B 部分: 0xE0 * 0x80 = 0x7000 → >> 8 = 0x70（但需要掩码）

步骤 4: 加回基准并掩码
  d_rb + (diff * 128 >> 8)  & 0x00FF00FF
  = 0x00800020 + 结果 & 0x00FF00FF

实际用代码验证:
  alpha_blend(0xFF804020, 0x80FF0000, 128)
  → R: 128 + (255-128)*128/256 = 128 + 63 = 191 ≈ 0xBF
  → G: 64 + (0-64)*128/256 = 64 - 32 = 32 = 0x20
  → B: 32 + (0-32)*128/256 = 32 - 16 = 16 = 0x10
  结果 ≈ 0x00BF2010
```

</details>

### 题目 2：为什么 G 通道不能与 R/B 并行？

解释为什么 KrKr2 把 R+B 合在一起计算，但 G 通道必须单独处理。如果强行把 G 和 B 合在一起（掩码 `0x0000FFFF`），会发生什么？

<details>
<summary>查看答案</summary>

R 在 bit 16-23，B 在 bit 0-7，它们之间有 8 位间隙（G 通道）。这个间隙保证了乘法结果不会互相溢出——R×α 最大 255×255=65025（16位），不会侵入 B 的位域。

如果用 `0x0000FFFF` 提取 G+B（bit 0-15）：
```
G 在 bit 8-15, B 在 bit 0-7，它们之间没有间隙！

0x0000GGBB × alpha:
  B×α 最大 = 255×255 = 65025 = 0xFE01（需要 16 位）
  这个结果会直接溢出到 G 通道的位域！
  
示例: G=0, B=255, alpha=255
  0x000000FF × 255 = 0x0000FE01
  高 8 位 0xFE 污染了 G 通道的位置
```

所以必须保证并行处理的两个通道之间至少有 8 位间隙。在 ARGB 布局中，只有 R 和 B 满足这个条件。

</details>

### 题目 3：实现精确除以 255 的混合

修改 `alpha_blend` 函数，使用 `div255` 代替 `>> 8`，对比两个版本在 `(src=200, dst=100, alpha=200)` 时的结果差异。

<details>
<summary>查看答案</summary>

```cpp
#include <cstdint>
#include <cstdio>

using tjs_uint32 = uint32_t;

// 精确的 x / 255
inline uint32_t div255(uint32_t x) {
    return (x + 128 + ((x + 128) >> 8)) >> 8;
}

// 精确版 Alpha 混合（逐通道，因为 div255 不能双通道并行）
tjs_uint32 alpha_blend_exact(tjs_uint32 d, tjs_uint32 s, tjs_uint32 a) {
    auto lerp = [a](uint32_t dc, uint32_t sc) -> uint32_t {
        return div255(sc * a + dc * (255 - a));
    };
    uint32_t r = lerp((d >> 16) & 0xff, (s >> 16) & 0xff);
    uint32_t g = lerp((d >> 8) & 0xff, (s >> 8) & 0xff);
    uint32_t b = lerp(d & 0xff, s & 0xff);
    return (r << 16) | (g << 8) | b;
}

// 近似版（KrKr2 风格）
tjs_uint32 alpha_blend_approx(tjs_uint32 d, tjs_uint32 s, tjs_uint32 a) {
    tjs_uint32 d1 = d & 0xff00ff;
    d1 = (d1 + (((s & 0xff00ff) - d1) * a >> 8)) & 0xff00ff;
    d &= 0xff00;
    s &= 0xff00;
    return d1 + ((d + ((s - d) * a >> 8)) & 0xff00);
}

int main() {
    // src=200, dst=100, alpha=200 → 各通道都设为相同值便于计算
    tjs_uint32 dst = 0xFF646464;  // R=G=B=100
    tjs_uint32 src = 0xFFC8C8C8;  // R=G=B=200
    tjs_uint32 alpha = 200;

    tjs_uint32 exact  = alpha_blend_exact(dst, src, alpha);
    tjs_uint32 approx = alpha_blend_approx(dst, src, alpha);

    printf("精确版: R=%d G=%d B=%d\n",
           (exact>>16)&0xff, (exact>>8)&0xff, exact&0xff);
    printf("近似版: R=%d G=%d B=%d\n",
           (approx>>16)&0xff, (approx>>8)&0xff, approx&0xff);
    printf("差异: %d\n", (int)((exact&0xff) - (approx&0xff)));

    // 精确: 200*200 + 100*55 = 40000 + 5500 = 45500
    //        45500 / 255 = 178.43 → 178
    // 近似: 100 + (200-100)*200/256 = 100 + 78 = 178
    // 在这个例子中差异为 0，但某些值组合会有 ±1 的差异
    return 0;
}
```

结论：两个版本的差异最大为 ±1 个色阶，视觉上完全不可察。KrKr2 选择 `>> 8` 是正确的工程权衡。

</details>

---

## 下一步

下一节 [Porter-Duff 与 KrKr2 混合模式](./02-Porter-Duff与KrKr2混合模式.md)，我们将学习 Porter-Duff 12 种合成模式的数学定义，以及 KrKr2 引擎 31 种混合模式（`tTVPBBBltMethod` 枚举）的分类、Functor + 宏展开架构和查找表优化。
