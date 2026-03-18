# tvpgl 像素操作库

> **所属模块：** M04-渲染子系统
> **前置知识：** [P05-软件渲染原理](../../P05-软件渲染原理/README.md)、[01-visual模块总览](../01-visual模块总览/)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 tvpgl 库的整体架构和函数指针分派机制
2. 掌握查找表（Lookup Table）的设计原理和性能优化意义
3. 理解 Duff's Device 循环展开技术及其在像素处理中的应用
4. 能够阅读和修改 tvpgl 中的混合函数实现

## tvpgl 概述

tvpgl（T Visual Presenter Graphics Library）是 KiriKiri2 引擎的核心像素操作库，负责所有 2D 图像的混合、合成、变换等底层操作。它采用纯 C 实现以保证最大的可移植性和性能优化空间。

### 架构设计理念

tvpgl 的设计遵循以下原则：

1. **运行时可切换实现**：通过函数指针，可在运行时选择 C 实现或 SSE/NEON 优化实现
2. **查找表优化**：预计算复杂运算结果，用空间换时间
3. **循环展开**：使用 Duff's Device 减少循环开销
4. **SIMD 友好**：数据布局和算法设计便于向量化

### 文件结构

```
cpp/core/visual/
├── tvpgl.h              # 函数声明、内联混合函数、宏定义
├── tvpgl.cpp            # C 实现（14000+ 行）
├── tvpgl_asm_init.h     # SSE/ASM 初始化声明
└── tvpps.inc            # Photoshop 混合模式宏
```

## 函数指针分派机制

tvpgl 最核心的设计是函数指针分派（Function Pointer Dispatch）。这允许在运行时根据 CPU 能力选择最优实现。

### 宏定义系统

在 `tvpgl.h` 中定义了一套完整的函数声明宏：

```cpp
// 普通函数声明（用于内部实现）
#define TVP_GL_FUNC_DECL(rettype, funcname, arg) \
    rettype funcname arg

// 外部函数声明（用于头文件导出）
#define TVP_GL_FUNC_EXTERN_DECL(rettype, funcname, arg) \
    extern rettype funcname arg

// 函数指针声明（用于运行时分派）
#define TVP_GL_FUNC_PTR_DECL(rettype, funcname, arg) \
    rettype(*funcname) arg

// 函数指针外部声明（同时声明指针和 C 实现）
#define TVP_GL_FUNC_PTR_EXTERN_DECL_(rettype, funcname, arg) \
    extern rettype(*funcname) arg;                           \
    extern rettype funcname##_c arg;

#define TVP_GL_FUNC_PTR_EXTERN_DECL TVP_GL_FUNC_PTR_EXTERN_DECL_
```

### 函数声明示例

以 Alpha 混合函数为例：

```cpp
// 在 tvpgl.h 中声明
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAlphaBlend,
    (tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len));

// 宏展开后等价于：
extern void (*TVPAlphaBlend)(tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len);
extern void TVPAlphaBlend_c(tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len);
```

这样设计的好处：

1. `TVPAlphaBlend` 是函数指针，可在运行时指向不同实现
2. `TVPAlphaBlend_c` 是 C 语言实现，作为默认/备选方案
3. SSE 实现会命名为 `TVPAlphaBlend_sse`（在其他文件中）

### 运行时初始化

在程序启动时，根据 CPU 支持的指令集选择实现：

```cpp
// 伪代码示意
void TVPGL_Init() {
    // 默认使用 C 实现
    TVPAlphaBlend = TVPAlphaBlend_c;
    TVPConstAlphaBlend = TVPConstAlphaBlend_c;
    // ... 更多函数
    
    // 检测 CPU 并切换到优化实现
    if (CPU_HasSSE2()) {
        TVPGL_ASM_Init();  // 切换到 SSE2 实现
    }
}

// 在 tvpgl_asm_init.h 中声明
extern void TVPGL_ASM_Init();
```

### 函数命名约定

tvpgl 函数名包含丰富的语义信息：

```
TVP[操作类型][Alpha模式][目标模式][附加选项]
```

| 后缀 | 含义 | 示例 |
|------|------|------|
| `_d` | 目标有 Alpha（destination has alpha） | `TVPAlphaBlend_d` |
| `_a` | 目标有 Additive Alpha | `TVPAlphaBlend_a` |
| `_o` | 带不透明度参数（with opacity） | `TVPAlphaBlend_o` |
| `_HDA` | 保持目标 Alpha 不变（Hold Dest Alpha） | `TVPAlphaBlend_HDA` |
| `_c` | C 语言实现 | `TVPAlphaBlend_c` |
| `_sse` | SSE 优化实现 | `TVPAlphaBlend_sse` |

完整函数列表示例：

```cpp
// 基础 Alpha 混合系列
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAlphaBlend, (...));      // 基础版
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAlphaBlend_HDA, (...));  // 保持目标Alpha
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAlphaBlend_o, (...));    // 带全局不透明度
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAlphaBlend_HDA_o, (...));// 组合
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAlphaBlend_d, (...));    // 目标有Alpha
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAlphaBlend_a, (...));    // Additive Alpha
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAlphaBlend_do, (...));   // d + o 组合
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAlphaBlend_ao, (...));   // a + o 组合
```

## 查找表系统

查找表（Lookup Table，LUT）是 tvpgl 性能优化的核心手段。通过预计算，将复杂的数学运算转化为简单的数组查找。

### 主要查找表

在 `tvpgl.cpp` 中定义了多个全局查找表：

```cpp
// 除法表 - 用于 Alpha 恢复（从预乘格式转回）
unsigned char TVPDivTable[256 * 256];
// 用法：result = TVPDivTable[alpha * 256 + color]
// 等价于：result = color * 255 / alpha（但避免了除法运算）

// 不透明度叠加表 - 计算两层 Alpha 叠加后的有效不透明度
unsigned char TVPOpacityOnOpacityTable[256 * 256];
// 用法：result = TVPOpacityOnOpacityTable[srcAlpha * 256 + destAlpha]

// 负片乘法表 - 用于 Screen 混合模式
unsigned char TVPNegativeMulTable[256 * 256];
// Screen(a, b) = 1 - (1-a)(1-b) = 255 - (255-a)(255-b)/255
// 用法：result = TVPNegativeMulTable[a * 256 + b]

// 65 级表 - 用于抗锯齿文字渲染（65 级灰度）
unsigned char TVPOpacityOnOpacityTable65[65 * 256];
unsigned char TVPNegativeMulTable65[65 * 256];
unsigned char TVPTable65_255[65];

// 抖动表 - 用于真彩色到调色板颜色转换
unsigned char TVPDitherTable_5_6[8][4][2][256];    // 5/6 位抖动
unsigned char TVPDitherTable_676[3][4][4][256];    // 6-7-6 调色板抖动
unsigned char TVP252DitherPalette[3][256];         // 252 色调色板

// 倒数表 - 用于快速除法
tjs_uint32 TVPRecipTable256[256];          // 1/x * 65536
tjs_uint16 TVPRecipTable256_16[256];       // 限制到 16 位
tjs_uint16 TVPRecipTableForOpacityOnOpacity[256];
```

### 查找表初始化

`TVPCreateTable()` 函数负责初始化所有查找表：

```cpp
static void TVPCreateTable() {
    int a, b;
    
    // 初始化 TVPOpacityOnOpacityTable 和 TVPNegativeMulTable
    for (a = 0; a < 256; a++) {
        for (b = 0; b < 256; b++) {
            int addr = b * 256 + a;  // 行优先索引
            
            // 不透明度叠加计算
            // 公式：c = bt / at / (1 - bt + bt/at)
            // 其中 at = a/255, bt = b/255
            if (a) {
                float at = a / 255.0f, bt = b / 255.0f;
                float c = bt / at;
                c /= (1.0f - bt + c);
                int ci = (int)(c * 255);
                if (ci >= 256) ci = 255;
                TVPOpacityOnOpacityTable[addr] = (unsigned char)ci;
            } else {
                TVPOpacityOnOpacityTable[addr] = 255;
            }
            
            // 负片乘法（Screen 模式）
            // Screen(a, b) = 255 - (255-a)(255-b)/255
            TVPNegativeMulTable[addr] = 
                (unsigned char)(255 - (255 - a) * (255 - b) / 255);
        }
    }
    
    // 初始化除法表
    for (b = 0; b < 256; b++) {
        TVPDivTable[(0 << 8) + b] = 0;  // 除以 0 返回 0
        for (a = 1; a < 256; a++) {
            int tmp = b * 255 / a;
            if (tmp > 255) tmp = 255;
            TVPDivTable[(a << 8) + b] = (tjs_uint8)tmp;
        }
    }
    
    // 初始化倒数表
    TVPRecipTable256[0] = 65536;
    TVPRecipTable256_16[0] = 0x7fff;
    for (int i = 1; i < 256; i++) {
        TVPRecipTable256[i] = 65536 / i;
        TVPRecipTable256_16[i] = 
            TVPRecipTable256[i] > 0x7fff ? 0x7fff : TVPRecipTable256[i];
    }
    
    // 初始化其他表...
    TVPInitDitherTable();
    TVPTLG6InitLeadingZeroTable();
    TVPTLG6InitGolombTable();
    TVPPsMakeTable();
}
```

### 查找表使用示例

以 Alpha 混合为例，展示如何使用查找表优化：

```cpp
// 不使用查找表的版本（需要除法）
inline uint8_t BlendWithAlpha_Slow(uint8_t src, uint8_t dst, uint8_t alpha) {
    // 线性插值：result = dst + (src - dst) * alpha / 255
    return dst + (src - dst) * alpha / 255;
}

// 使用乘法近似（避免除法，但有精度损失）
inline uint8_t BlendWithAlpha_Approx(uint8_t src, uint8_t dst, uint8_t alpha) {
    // 用 >>8 代替 /255，误差约 0.4%
    return dst + ((src - dst) * alpha >> 8);
}

// 从预乘格式恢复时使用除法表
inline uint8_t RecoverFromPremultiplied(uint8_t premul_color, uint8_t alpha) {
    // 等价于：premul_color * 255 / alpha
    // 但使用查找表避免除法
    return TVPDivTable[(alpha << 8) + premul_color];
}

// 计算两层叠加后的有效 Alpha
inline uint8_t CombineOpacity(uint8_t src_alpha, uint8_t dst_alpha) {
    return TVPOpacityOnOpacityTable[(src_alpha << 8) + dst_alpha];
}
```

### 查找表性能分析

| 操作 | 不用 LUT | 使用 LUT | 加速比 |
|------|----------|----------|--------|
| 除法 `a/b` | ~20-40 周期 | ~3-5 周期 | 6-10x |
| 复杂公式 | 50-100 周期 | ~3-5 周期 | 15-30x |
| 内存访问 | - | L1 命中 ~4 周期 | - |

查找表的代价是内存占用：

| 表名 | 大小 | 用途 |
|------|------|------|
| TVPDivTable | 64 KB | Alpha 恢复 |
| TVPOpacityOnOpacityTable | 64 KB | 不透明度叠加 |
| TVPNegativeMulTable | 64 KB | Screen 混合 |
| TVPRecipTable256 | 1 KB | 快速倒数 |
| 合计 | ~200 KB | - |

## Duff's Device 循环展开

Duff's Device 是 tvpgl 中大量使用的循环展开技术，由 Tom Duff 于 1983 年发明。它利用 C 语言 switch-case 的"穿透"特性实现循环展开。

### 原理解释

传统循环每次迭代都需要：
1. 检查循环条件
2. 更新循环变量
3. 跳转回循环开始

这些开销在处理大量像素时累积显著。Duff's Device 将循环体复制多次，减少迭代次数。

### 标准实现

以 4 倍展开为例：

```cpp
// 原始循环
void copy_pixels_simple(uint32_t* dest, const uint32_t* src, int len) {
    for (int i = 0; i < len; i++) {
        dest[i] = src[i];
    }
}

// Duff's Device 4 倍展开
void copy_pixels_duff(uint32_t* dest, const uint32_t* src, int len) {
    if (len > 0) {
        int n = (len + 3) / 4;  // 向上取整的迭代次数
        switch (len % 4) {      // 处理余数
            case 0: do { *dest++ = *src++;
            case 3:      *dest++ = *src++;
            case 2:      *dest++ = *src++;
            case 1:      *dest++ = *src++;
                    } while (--n > 0);
        }
    }
}
```

### tvpgl 中的实际应用

以 `TVPAlphaBlend_c` 为例，展示完整的 Duff's Device 应用：

```cpp
TVP_GL_FUNC_DECL(void, TVPAlphaBlend_c,
    (tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len))
{
    tjs_uint32 d1, s, d, sopa;
    
    if (len > 0) {
        int lu_n = (len + 3) / 4;  // 计算迭代次数
        
        switch (len % 4) {
            case 0:
                do {
                    {   // 第 1 个像素
                        s = *src++;           // 读取源像素
                        d = *dest;            // 读取目标像素
                        sopa = s >> 24;       // 提取源 Alpha
                        
                        // 混合 R 和 B 通道（它们在 0xff00ff 位置）
                        d1 = d & 0xff00ff;
                        d1 = (d1 + (((s & 0xff00ff) - d1) * sopa >> 8)) & 0xff00ff;
                        
                        // 混合 G 通道（在 0xff00 位置）
                        d &= 0xff00;
                        s &= 0xff00;
                        *dest++ = d1 + ((d + ((s - d) * sopa >> 8)) & 0xff00);
                    }
                    case 3: {   // 第 2 个像素
                        s = *src++;
                        d = *dest;
                        sopa = s >> 24;
                        d1 = d & 0xff00ff;
                        d1 = (d1 + (((s & 0xff00ff) - d1) * sopa >> 8)) & 0xff00ff;
                        d &= 0xff00;
                        s &= 0xff00;
                        *dest++ = d1 + ((d + ((s - d) * sopa >> 8)) & 0xff00);
                    }
                    case 2: {   // 第 3 个像素
                        s = *src++;
                        d = *dest;
                        sopa = s >> 24;
                        d1 = d & 0xff00ff;
                        d1 = (d1 + (((s & 0xff00ff) - d1) * sopa >> 8)) & 0xff00ff;
                        d &= 0xff00;
                        s &= 0xff00;
                        *dest++ = d1 + ((d + ((s - d) * sopa >> 8)) & 0xff00);
                    }
                    case 1: {   // 第 4 个像素
                        s = *src++;
                        d = *dest;
                        sopa = s >> 24;
                        d1 = d & 0xff00ff;
                        d1 = (d1 + (((s & 0xff00ff) - d1) * sopa >> 8)) & 0xff00ff;
                        d &= 0xff00;
                        s &= 0xff00;
                        *dest++ = d1 + ((d + ((s - d) * sopa >> 8)) & 0xff00);
                    }
                } while (--lu_n);
        }
    }
}
```

### 性能分析

Duff's Device 4 倍展开的效果：

| 指标 | 原始循环 | 4 倍展开 | 改善 |
|------|----------|----------|------|
| 循环开销 | 每像素 1 次 | 每 4 像素 1 次 | 75% ↓ |
| 分支预测失败 | 频繁 | 大幅减少 | - |
| 指令缓存 | 小循环体 | 较大但可接受 | - |

### 现代替代方案

现代编译器（GCC/Clang -O2 及以上）通常能自动展开简单循环，但 Duff's Device 仍有价值：

```cpp
// 编译器自动展开（需要 -funroll-loops 或 #pragma）
#pragma GCC unroll 4
for (int i = 0; i < len; i++) {
    // 循环体
}

// 或使用 SIMD 内在函数（SSE/NEON）
// 一次处理 4 个像素
__m128i pixels = _mm_loadu_si128((__m128i*)src);
// ... SIMD 运算
_mm_storeu_si128((__m128i*)dest, result);
```

## 内联混合函数

`tvpgl.h` 中定义了多个内联函数，用于单像素操作或作为批量函数的构建块。

### 饱和加法

```cpp
// 饱和加法：每个字节通道独立相加，溢出时饱和到 255
static tjs_uint32 TVP_INLINE_FUNC TVPSaturatedAdd(tjs_uint32 a, tjs_uint32 b) {
    // 检测哪些字节会溢出
    // 原理：如果两个数的某位都是 1，加法后该位可能进位
    tjs_uint32 tmp = ((a & b) + (((a ^ b) >> 1) & 0x7f7f7f7f)) & 0x80808080;
    
    // 将溢出标志扩展为掩码（0x80 -> 0xff）
    tmp = (tmp << 1) - (tmp >> 7);
    
    // 相加后用掩码将溢出字节设为 0xff
    return (a + b - tmp) | tmp;
}

// 使用示例
uint32_t pixel1 = 0x80FF8040;  // ARGB
uint32_t pixel2 = 0x40808080;
uint32_t result = TVPSaturatedAdd(pixel1, pixel2);
// result = 0xC0FF8040 + 0x40808080 = 0xFFFFFFFF（每通道饱和）
```

### Additive Alpha 混合

```cpp
// Additive Alpha 混合：源像素已预乘 Alpha
// dest/src 命名约定：n(无Alpha), a(additive), d(普通Alpha)
static tjs_uint32 TVP_INLINE_FUNC TVPAddAlphaBlend_n_a(
    tjs_uint32 dest,    // 无 Alpha 的目标
    tjs_uint32 src      // 预乘 Alpha 的源（Additive Alpha）
) {
    // sopa = 255 - src_alpha（源的"透明度"）
    tjs_uint32 sopa = (~src) >> 24;
    
    // dest * (1 - src_alpha) + src
    // 因为 src 已预乘，直接加即可
    return TVPSaturatedAdd(
        (((dest & 0xff00ff) * sopa >> 8) & 0xff00ff) +   // R, B 通道
        (((dest & 0xff00) * sopa >> 8) & 0xff00),        // G 通道
        src
    );
}

// HDA 版本：保持目标 Alpha 不变（Hold Destination Alpha）
static tjs_uint32 TVP_INLINE_FUNC TVPAddAlphaBlend_HDA_n_a(
    tjs_uint32 dest, tjs_uint32 src
) {
    return (dest & 0xff000000) + (TVPAddAlphaBlend_n_a(dest, src) & 0xffffff);
}

// 带全局不透明度的版本
static tjs_uint32 TVP_INLINE_FUNC TVPAddAlphaBlend_n_a_o(
    tjs_uint32 dest, tjs_uint32 src, tjs_int opa
) {
    // 先将源像素乘以全局不透明度
    src = (((src & 0xff00ff) * opa >> 8) & 0xff00ff) +
          (((src >> 8) & 0xff00ff) * opa & 0xff00ff00);
    return TVPAddAlphaBlend_n_a(dest, src);
}
```

### 颜色乘法

```cpp
// 颜色乘法：每个通道乘以一个系数
static tjs_uint32 TVP_INLINE_FUNC TVPMulColor(tjs_uint32 color, tjs_uint32 fac) {
    // 分离处理 G 和 R/B 通道
    return (
        ((((color & 0x00ff00) * fac) & 0x00ff0000) +   // G 通道
         (((color & 0xff00ff) * fac) & 0xff00ff00))    // R, B 通道
        >> 8
    );
}

// 将 Alpha 和颜色转换为 Additive Alpha 格式
static tjs_uint32 TVP_INLINE_FUNC TVPAlphaAndColorToAdditiveAlpha(
    tjs_uint32 alpha, tjs_uint32 color
) {
    // Additive Alpha 格式：RGB 已预乘 Alpha
    return TVPMulColor(color, alpha) + (color & 0xff000000);
}

// 从普通 Alpha 格式转换
static tjs_uint32 TVP_INLINE_FUNC TVPAlphaToAdditiveAlpha(tjs_uint32 a) {
    return TVPAlphaAndColorToAdditiveAlpha(a >> 24, a);
}
```

### ARGB 线性插值

```cpp
// 两个 ARGB 颜色的线性插值
static tjs_uint32 TVP_INLINE_FUNC TVPBlendARGB(
    tjs_uint32 b,       // 颜色 B
    tjs_uint32 a,       // 颜色 A
    tjs_int ratio       // 混合比例 [0, 256]，256 = 100% A
) {
    // result = b + (a - b) * ratio / 256
    tjs_uint32 b2;
    tjs_uint32 t;
    
    // 处理 R 和 B 通道
    b2 = b & 0x00ff00ff;
    t = (b2 + (((a & 0x00ff00ff) - b2) * ratio >> 8)) & 0x00ff00ff;
    
    // 处理 G 和 A 通道
    b2 = (b & 0xff00ff00) >> 8;
    return t + (((b2 + ((((a & 0xff00ff00) >> 8) - b2) * ratio >> 8)) << 8) 
                & 0xff00ff00);
}
```

## 动手实践

### 练习 1：实现简化版 Alpha 混合

创建一个简化版的 Alpha 混合函数，理解核心算法：

```cpp
#include <cstdint>
#include <cstdio>

// 简化版 Alpha 混合（单像素）
uint32_t AlphaBlendPixel(uint32_t dest, uint32_t src) {
    // 提取源 Alpha
    uint32_t src_alpha = src >> 24;
    
    if (src_alpha == 0) return dest;    // 完全透明
    if (src_alpha == 255) return src;   // 完全不透明
    
    // 提取各通道
    uint32_t dest_r = (dest >> 16) & 0xff;
    uint32_t dest_g = (dest >> 8) & 0xff;
    uint32_t dest_b = dest & 0xff;
    
    uint32_t src_r = (src >> 16) & 0xff;
    uint32_t src_g = (src >> 8) & 0xff;
    uint32_t src_b = src & 0xff;
    
    // 线性插值
    uint32_t out_r = dest_r + ((src_r - dest_r) * src_alpha >> 8);
    uint32_t out_g = dest_g + ((src_g - dest_g) * src_alpha >> 8);
    uint32_t out_b = dest_b + ((src_b - dest_b) * src_alpha >> 8);
    
    return (src_alpha << 24) | (out_r << 16) | (out_g << 8) | out_b;
}

// 使用位操作优化的版本（模仿 tvpgl）
uint32_t AlphaBlendPixel_Opt(uint32_t dest, uint32_t src) {
    uint32_t sopa = src >> 24;
    
    // 同时处理 R 和 B 通道（它们不相邻，不会互相影响）
    uint32_t d1 = dest & 0xff00ff;
    d1 = (d1 + (((src & 0xff00ff) - d1) * sopa >> 8)) & 0xff00ff;
    
    // 处理 G 通道
    uint32_t d_g = dest & 0xff00;
    uint32_t s_g = src & 0xff00;
    uint32_t out_g = (d_g + ((s_g - d_g) * sopa >> 8)) & 0xff00;
    
    return d1 | out_g;
}

int main() {
    // 测试：半透明红色覆盖蓝色
    uint32_t blue = 0xff0000ff;      // 不透明蓝色
    uint32_t red_50 = 0x80ff0000;    // 50% 透明红色
    
    uint32_t result1 = AlphaBlendPixel(blue, red_50);
    uint32_t result2 = AlphaBlendPixel_Opt(blue, red_50);
    
    printf("Simple: 0x%08x\n", result1);
    printf("Optimized: 0x%08x\n", result2);
    // 期望结果：紫色（红+蓝混合）
    
    return 0;
}
```

### 练习 2：创建简单的查找表

```cpp
#include <cstdint>
#include <cstdio>

// 256x256 乘法表（预计算 a * b / 255）
static uint8_t MulTable[256][256];

void InitMulTable() {
    for (int a = 0; a < 256; a++) {
        for (int b = 0; b < 256; b++) {
            MulTable[a][b] = (uint8_t)((a * b + 127) / 255);  // 四舍五入
        }
    }
}

// 使用查找表的乘法
inline uint8_t MulColor(uint8_t color, uint8_t factor) {
    return MulTable[color][factor];
}

// 使用查找表的 Alpha 混合
uint32_t AlphaBlendWithTable(uint32_t dest, uint32_t src) {
    uint8_t alpha = src >> 24;
    uint8_t inv_alpha = 255 - alpha;
    
    uint8_t r = MulTable[(src >> 16) & 0xff][alpha] + 
                MulTable[(dest >> 16) & 0xff][inv_alpha];
    uint8_t g = MulTable[(src >> 8) & 0xff][alpha] + 
                MulTable[(dest >> 8) & 0xff][inv_alpha];
    uint8_t b = MulTable[src & 0xff][alpha] + 
                MulTable[dest & 0xff][inv_alpha];
    
    return (alpha << 24) | (r << 16) | (g << 8) | b;
}

int main() {
    InitMulTable();
    
    // 测试乘法表
    printf("128 * 128 / 255 = %d (table: %d)\n", 
           128 * 128 / 255, MulTable[128][128]);
    printf("200 * 100 / 255 = %d (table: %d)\n", 
           200 * 100 / 255, MulTable[200][100]);
    
    return 0;
}
```

## 对照项目源码

### 关键文件

| 文件 | 行号范围 | 内容 |
|------|----------|------|
| `cpp/core/visual/tvpgl.h` | 1-56 | 函数指针宏定义 |
| `cpp/core/visual/tvpgl.h` | 73-203 | 内联混合函数 |
| `cpp/core/visual/tvpgl.cpp` | 29-55 | 全局查找表声明 |
| `cpp/core/visual/tvpgl.cpp` | 224-301 | `TVPCreateTable()` 查找表初始化 |
| `cpp/core/visual/tvpgl.cpp` | 306-369 | `TVPAlphaBlend_c` Duff's Device 实现 |

### 函数数量统计

tvpgl 库包含约 100+ 个像素操作函数：

| 类别 | 数量 | 示例 |
|------|------|------|
| Alpha 混合 | ~20 | TVPAlphaBlend, TVPAlphaBlend_HDA, ... |
| Additive 混合 | ~10 | TVPAdditiveAlphaBlend, ... |
| 拉伸混合 | ~15 | TVPStretchAlphaBlend, ... |
| 仿射变换 | ~8 | TVPAffineBlend, ... |
| ColorMap | ~10 | TVPApplyColorMap, ... |
| 颜色调整 | ~8 | TVPAdjustGamma, TVPChBlend, ... |
| PS 混合模式 | ~25 | TVPPsMulBlend, TVPPsScreenBlend, ... |

## 本节小结

- tvpgl 使用**函数指针分派**实现运行时可切换的优化路径
- **查找表**将复杂数学运算转化为数组查找，大幅提升性能
- **Duff's Device** 是一种循环展开技术，减少循环开销
- 函数命名约定（`_d`, `_a`, `_o`, `_HDA`, `_c`）清晰表达功能变体
- 内联函数处理单像素操作，批量函数处理像素数组
- 位操作技巧（如 `& 0xff00ff`）允许同时处理多个通道

## 练习题与答案

### 题目 1：函数命名解析

解释以下函数名的含义：
1. `TVPAlphaBlend_HDA_o`
2. `TVPAdditiveAlphaBlend_ao`
3. `TVPStretchAlphaBlend_d`

<details>
<summary>查看答案</summary>

1. **TVPAlphaBlend_HDA_o**
   - `TVPAlphaBlend`: 基础 Alpha 混合
   - `_HDA`: Hold Destination Alpha（保持目标 Alpha 不变）
   - `_o`: 带全局不透明度参数（opacity）
   - 完整含义：带全局不透明度的 Alpha 混合，混合后保持目标像素的 Alpha 通道

2. **TVPAdditiveAlphaBlend_ao**
   - `TVPAdditiveAlphaBlend`: Additive Alpha 混合（源已预乘 Alpha）
   - `_a`: 目标也有 Additive Alpha
   - `_o`: 带全局不透明度参数
   - 完整含义：源和目标都是 Additive Alpha 格式，带全局不透明度的混合

3. **TVPStretchAlphaBlend_d**
   - `TVPStretchAlphaBlend`: 带拉伸（缩放）的 Alpha 混合
   - `_d`: 目标有普通 Alpha（destination has alpha）
   - 完整含义：处理缩放的 Alpha 混合，目标像素包含 Alpha 通道

</details>

### 题目 2：Duff's Device 分析

给定以下代码，分析当 `len = 7` 时的执行流程：

```cpp
int n = (len + 3) / 4;
switch (len % 4) {
    case 0: do { printf("0 ");
    case 3:      printf("3 ");
    case 2:      printf("2 ");
    case 1:      printf("1 ");
            } while (--n > 0);
}
```

<details>
<summary>查看答案</summary>

当 `len = 7` 时：

1. `n = (7 + 3) / 4 = 2`
2. `len % 4 = 3`，进入 `case 3`
3. 第一轮循环：从 `case 3` 开始穿透
   - 打印 "3 "
   - 打印 "2 "
   - 打印 "1 "
   - `--n = 1 > 0`，继续循环
4. 第二轮循环：从 `case 0` 开始（do-while 的 do）
   - 打印 "0 "
   - 打印 "3 "
   - 打印 "2 "
   - 打印 "1 "
   - `--n = 0`，退出循环

**输出结果**：`3 2 1 0 3 2 1 `

**处理像素数**：第一轮 3 个 + 第二轮 4 个 = 7 个（正好等于 len）

</details>

### 题目 3：实现饱和减法

参考 `TVPSaturatedAdd` 的实现，编写一个饱和减法函数：

```cpp
// 饱和减法：每个字节通道独立相减，下溢时饱和到 0
uint32_t TVPSaturatedSub(uint32_t a, uint32_t b);
```

<details>
<summary>查看答案</summary>

```cpp
#include <cstdint>
#include <cstdio>

uint32_t TVPSaturatedSub(uint32_t a, uint32_t b) {
    // 检测哪些字节会下溢
    // 如果 a 的某字节小于 b 的对应字节，结果应为 0
    
    // 方法：检查 a < b 的条件
    // 如果 a >= b，则 a - b + 0x80 的最高位为 1
    // 如果 a < b，则 a - b + 0x80 的最高位为 0（因为借位）
    
    // 简单实现（逐字节处理，供理解）
    uint32_t result = 0;
    for (int i = 0; i < 4; i++) {
        int shift = i * 8;
        int va = (a >> shift) & 0xff;
        int vb = (b >> shift) & 0xff;
        int diff = va - vb;
        if (diff < 0) diff = 0;  // 饱和到 0
        result |= (diff << shift);
    }
    return result;
}

// 优化版本（位操作，模仿 TVPSaturatedAdd）
uint32_t TVPSaturatedSub_Opt(uint32_t a, uint32_t b) {
    // 计算 a - b，可能有借位
    uint32_t diff = a - b;
    
    // 检测下溢：如果 a < b，则 (a | ~b) 的最高位为 0
    // 而 (a - b) 会借位导致最高位为 1
    // 比较两者可检测借位
    
    // 另一种方法：先计算差值，再用掩码清除下溢字节
    // 下溢条件：a 的字节 < b 的字节
    // 等价于：(~a & b) 的最高位为 1
    
    // 找出哪些字节下溢
    uint32_t borrow = ((~a & b) | ((~a | b) & diff)) & 0x80808080;
    // 将借位标志扩展为掩码
    uint32_t mask = (borrow >> 7) * 0xff;
    
    // 清除下溢字节
    return diff & ~mask;
}

int main() {
    // 测试
    uint32_t a = 0x80404020;
    uint32_t b = 0x40608010;
    
    printf("a = 0x%08x\n", a);
    printf("b = 0x%08x\n", b);
    printf("Simple:    0x%08x\n", TVPSaturatedSub(a, b));
    printf("Optimized: 0x%08x\n", TVPSaturatedSub_Opt(a, b));
    // 期望：0x40 0x00 0x00 0x10
    //       (0x80-0x40, 0x40-0x60饱和0, 0x20-0x80饱和0, 0x20-0x10)
    
    return 0;
}
```

</details>

## 下一步

下一节 [02-ARGB混合与合成](./02-ARGB混合与合成.md) 将深入讲解：
- tTVPARGB 模板结构的设计
- Porter-Duff 合成模式的实现
- Photoshop 混合模式（tvpps.inc）
- Additive Alpha 与标准 Alpha 的转换
