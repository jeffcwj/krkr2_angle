# tvpgl 像素操作库

> **所属模块：** M04-渲染子系统
> **前置知识：** [位图操作与绘制](../02-图层系统/03-位图操作与绘制.md)、[渲染管线概览](../01-visual模块总览/02-渲染管线概览.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 tvpgl（T Visual Presenter Graphics Library）的整体架构与设计目标
2. 掌握像素内存布局（ARGB 打包格式）与位操作技巧
3. 理解预计算查找表（Lookup Table）系统的原理与初始化流程
4. 掌握函数指针分发机制（`TVP_GL_FUNC_PTR`）的运作方式
5. 理解 Duff's Device 循环展开优化技巧在像素操作中的应用

## tvpgl 库概述

tvpgl（T Visual Presenter Graphics Library）是 KrKr2 引擎中负责 **CPU 端像素级图形操作** 的底层 C 库。它的名字中 "gl" 代表 Graphics Library，而非 OpenGL。这个库由 Perl 脚本 `gengl.pl` 自动生成，提供了 100 多个像素操作函数，覆盖混合（blend）、拷贝（copy）、填充（fill）、颜色转换、抖动（dither）等操作。

tvpgl 的设计遵循三个核心原则：

1. **极致性能**：所有函数都针对逐扫描线（scanline）操作设计，使用查找表避免运行时除法，使用 Duff's Device 循环展开减少分支开销
2. **可替换实现**：通过函数指针机制，每个操作都可在运行时切换 C 纯软实现或 SSE/ASM 加速实现
3. **零依赖**：纯 C 实现，不依赖任何外部库，仅需 `<string.h>` 和 `<math.h>`

### 文件组织

```
visual/
├── tvpgl.h              # 函数声明、内联辅助函数、结构体定义
├── tvpgl.cpp             # C 实现（14000+ 行，由 gengl.pl 生成）
├── tvpgl_asm_init.h      # ASM/SSE 初始化接口声明
├── argb.h                # ARGB 像素类型模板
├── gl/
│   ├── blend_functor_c.h # 混合操作 functor 定义
│   ├── blend_function.cpp# 模板混合函数 + 函数指针初始化
│   ├── blend_util_func.h # 底层混合辅助函数
│   └── blend_variation.h # 混合变体模板（HDA/opacity/dest-alpha）
└── tvpps.inc             # Photoshop 混合模式查找表
```

## 像素内存布局

KrKr2 使用 32 位 ARGB 打包格式存储像素，每个像素占 4 字节（`tjs_uint32`）。在内存中的字节序排列为：

```
位域布局（高位 → 低位）：
┌──────────┬──────────┬──────────┬──────────┐
│  Alpha   │   Red    │  Green   │   Blue   │
│ [31:24]  │ [23:16]  │ [15:8]   │  [7:0]   │
└──────────┴──────────┴──────────┴──────────┘
   8 bit      8 bit      8 bit      8 bit
```

以下是提取和组装各通道值的基本位操作：

```cpp
#include <cstdint>
#include <cstdio>

// KrKr2 内部类型
using tjs_uint32 = uint32_t;

// 从打包像素中提取各通道
void extract_channels(tjs_uint32 pixel) {
    tjs_uint32 alpha = pixel >> 24;              // 右移 24 位得到 A
    tjs_uint32 red   = (pixel >> 16) & 0xFF;     // 右移 16 位，掩码取低 8 位
    tjs_uint32 green = (pixel >> 8)  & 0xFF;     // 右移 8 位，掩码取低 8 位
    tjs_uint32 blue  = pixel & 0xFF;             // 直接掩码取低 8 位
    printf("A=%u R=%u G=%u B=%u\n", alpha, red, green, blue);
}

// 从各通道组装打包像素
tjs_uint32 pack_pixel(uint8_t a, uint8_t r, uint8_t g, uint8_t b) {
    return (static_cast<tjs_uint32>(a) << 24) |  // A 放到高 8 位
           (static_cast<tjs_uint32>(r) << 16) |  // R 放到次高 8 位
           (static_cast<tjs_uint32>(g) << 8)  |  // G 放到次低 8 位
           static_cast<tjs_uint32>(b);            // B 放到最低 8 位
}

int main() {
    tjs_uint32 pixel = 0x80FF3300; // A=128, R=255, G=51, B=0
    extract_channels(pixel);        // 输出：A=128 R=255 G=51 B=0

    tjs_uint32 rebuilt = pack_pixel(128, 255, 51, 0);
    printf("rebuilt=0x%08X\n", rebuilt); // 输出：rebuilt=0x80FF3300
    return 0;
}
```

### 并行位操作技巧

tvpgl 大量使用 **双通道并行** 技巧来减少指令数。`0xff00ff` 掩码可以同时操作 R 和 B 两个通道（它们在 32 位字中被 G 通道隔开），而 `0xff00` 掩码操作 G 通道：

```cpp
// 项目源码：tvpgl.cpp 第 319-324 行
// 一次操作同时处理 R 和 B 两个通道
tjs_uint32 d1 = d & 0xff00ff;        // 提取 dest 的 R+B
// (s & 0xff00ff) - d1 计算 src 与 dest 的 R+B 差值
// * sopa >> 8 乘以不透明度后除以 256
// & 0xff00ff 掩码确保通道间不会溢出
d1 = (d1 + (((s & 0xff00ff) - d1) * sopa >> 8)) & 0xff00ff;

// 单独处理 G 通道
d &= 0xff00;                         // 提取 dest 的 G
s &= 0xff00;                         // 提取 src 的 G
// 最终结果 = R+B 结果 + G 结果
*dest = d1 + ((d + ((s - d) * sopa >> 8)) & 0xff00);
```

这种"双通道并行"之所以可行，是因为 R（位 23:16）和 B（位 7:0）在 32 位字中相隔 8 位，中间被 G 通道隔开。乘法结果不会跨通道溢出，前提是乘数不超过 256。

## 预计算查找表系统

tvpgl 使用多个预计算查找表（Lookup Table, LUT）来避免运行时的浮点除法和复杂计算。这些表在引擎初始化时一次性计算完成，之后所有像素操作直接查表获取结果。

### 核心查找表一览

| 表名 | 大小 | 用途 |
|------|------|------|
| `TVPDivTable[256*256]` | 64 KB | `b * 255 / a` 的查表，避免除法运算 |
| `TVPOpacityOnOpacityTable[256*256]` | 64 KB | 半透明叠半透明时的有效不透明度 |
| `TVPNegativeMulTable[256*256]` | 64 KB | `255 - (255-a)*(255-b)/255` 合并不透明度 |
| `TVPRecipTable256[256]` | 1 KB | `65536 / x` 倒数表，用于覆盖焼き（color dodge） |
| `TVPOpacityOnOpacityTable65[65*256]` | 16 KB | 65 级反走样（anti-aliased）字体混合用 |
| `TVPDitherTable_5_6[8][4][2][256]` | 16 KB | 8bit→5bit/6bit 有序抖动表 |
| `TVP252DitherPalette[3][256]` | 768 B | 252 色（6×7×6）调色板抖动表 |

### 查找表初始化流程

所有查找表在 `TVPCreateTable()` 中统一初始化（`tvpgl.cpp` 第 224-301 行）：

```cpp
// 项目源码：tvpgl.cpp 第 224-301 行（精简版）
static void TVPCreateTable() {
    int a, b;
    // 1. 构建不透明度-on-不透明度表
    for (a = 0; a < 256; a++) {
        for (b = 0; b < 256; b++) {
            int addr = b * 256 + a;
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
            // 负乘表：合并两层不透明度的结果
            TVPNegativeMulTable[addr] =
                (unsigned char)(255 - (255 - a) * (255 - b) / 255);
        }
    }

    // 2. 构建除法表
    for (b = 0; b < 256; b++) {
        TVPDivTable[(0 << 8) + b] = 0;  // 除以 0 返回 0
        for (a = 1; a < 256; a++) {
            auto tmp = (int)(b * 255 / a);
            if (tmp > 255) tmp = 255;
            TVPDivTable[(a << 8) + b] = (uint8_t)(tmp);
        }
    }

    // 3. 构建倒数表（用于 color dodge 混合）
    TVPRecipTable256[0] = 65536;
    for (int i = 1; i < 256; i++) {
        TVPRecipTable256[i] = 65536 / i;   // 16.16 定点倒数
    }

    // 4. 有序抖动表、TLG Golomb 表等
    TVPInitDitherTable();
    TVPTLG6InitLeadingZeroTable();
    TVPTLG6InitGolombTable();
    TVPPsMakeTable();  // Photoshop 混合查找表
}
```

### TVPDivTable 使用示例

`TVPDivTable` 将除法 `b * 255 / a` 预计算为查表操作。索引方式为 `TVPDivTable[(a << 8) + b]`，其中 `a` 是除数（高字节），`b` 是被除数（低字节）：

```cpp
#include <cstdint>
#include <cstdio>

// 模拟 TVPDivTable 的构建和使用
static unsigned char TVPDivTable[256 * 256];

void init_div_table() {
    for (int b = 0; b < 256; b++) {
        TVPDivTable[b] = 0;  // a=0 时返回 0
        for (int a = 1; a < 256; a++) {
            int tmp = b * 255 / a;
            if (tmp > 255) tmp = 255;
            TVPDivTable[(a << 8) + b] = (uint8_t)tmp;
        }
    }
}

// 使用查表代替除法：计算 pixel_value * 255 / alpha
uint8_t div_by_alpha(uint8_t pixel_value, uint8_t alpha) {
    return TVPDivTable[(alpha << 8) + pixel_value];
}

int main() {
    init_div_table();
    // 等价于 200 * 255 / 128 = 398 → 截断到 255
    printf("200/128 = %d\n", div_by_alpha(200, 128)); // 输出 255
    // 等价于 100 * 255 / 200 = 127
    printf("100/200 = %d\n", div_by_alpha(100, 200)); // 输出 127
    return 0;
}
```

## 函数指针分发机制

tvpgl 的核心设计之一是 **函数指针分发**（Function Pointer Dispatch）。每个像素操作函数都声明为函数指针，而非直接的函数实现。这样做的目的是：运行时可根据 CPU 支持的指令集（如 SSE2）将函数指针切换到优化实现。

### TVP_GL_FUNC_PTR 宏体系

`tvpgl.h` 中定义了一组宏来声明函数指针和对应的 C 默认实现：

```cpp
// 项目源码：tvpgl.h 第 48-56 行
// 声明函数的实现（直接函数）
#define TVP_GL_FUNC_DECL(rettype, funcname, arg) \
    rettype funcname arg

// 声明函数指针 + C 默认实现
// 展开后产生两个符号：
//   rettype (*funcname)(args);          // 函数指针
//   rettype funcname_c(args);           // C 参考实现
#define TVP_GL_FUNC_PTR_EXTERN_DECL_(rettype, funcname, arg) \
    extern rettype (*funcname) arg;   \
    extern rettype funcname##_c arg;
```

以 `TVPAlphaBlend` 为例，宏展开后得到：

```cpp
// 展开结果
extern void (*TVPAlphaBlend)(tjs_uint32* dest,
                             const tjs_uint32* src,
                             tjs_int len);
extern void TVPAlphaBlend_c(tjs_uint32* dest,
                            const tjs_uint32* src,
                            tjs_int len);
```

`TVPAlphaBlend` 是函数指针，默认指向 `TVPAlphaBlend_c`（C 实现），但可在初始化时重定向到 SSE 版本。调用代码只需写 `TVPAlphaBlend(dest, src, len)`，无需关心底层是哪个实现。

### 函数命名后缀规约

tvpgl 函数使用系统化的后缀命名来表示不同的混合变体：

| 后缀 | 含义 | 示例 |
|------|------|------|
| （无后缀） | 基本混合，忽略目标 Alpha | `TVPAlphaBlend` |
| `_HDA` | Hold Destination Alpha，保持目标的 Alpha 通道不变 | `TVPAlphaBlend_HDA` |
| `_o` | with Opacity，带全局不透明度参数 | `TVPAlphaBlend_o` |
| `_d` | Destination has alpha，目标有 Alpha 通道参与计算 | `TVPAlphaBlend_d` |
| `_a` | Additive alpha（pre-multiplied alpha），加法预乘 Alpha | `TVPAlphaBlend_a` |
| `_do` | Destination alpha + opacity | `TVPAlphaBlend_do` |
| `_ao` | Additive alpha + opacity | `TVPAlphaBlend_ao` |
| `_SD` | Source-Destination，三缓冲区混合（dest = src1 op src2） | `TVPConstAlphaBlend_SD` |

这种命名规约使得每种混合模式都能精确匹配图层的 Alpha 类型和混合需求。

### TVPGL_C_Init：函数指针绑定

所有函数指针在 `TVPGL_C_Init()` 中统一绑定到 C 实现：

```cpp
// 项目源码：blend_function.cpp 第 590-685 行（精简版）
void TVPGL_C_Init() {
    // 混合函数族 —— 通过 SET_BLEND_FUNCTIONS 宏批量设置
    // 展开后等价于：
    //   TVPAlphaBlend     = TVP_alpha_blend;
    //   TVPAlphaBlend_HDA = TVP_alpha_blend_HDA;
    //   TVPAlphaBlend_o   = TVP_alpha_blend_o;
    //   ... (共 8 个变体)
    SET_BLEND_FUNCTIONS(AlphaBlend, alpha_blend);
    SET_BLEND_FUNCTIONS(AdditiveAlphaBlend, premulalpha_blend);

    // 最小变体族（4 个变体：普通/HDA/o/HDA_o）
    SET_BLEND_MIN_FUNCTIONS(AddBlend, add_blend);
    SET_BLEND_MIN_FUNCTIONS(SubBlend, sub_blend);
    SET_BLEND_MIN_FUNCTIONS(MulBlend, mul_blend);

    // Photoshop 兼容混合模式
    SET_BLEND_MIN_FUNCTIONS(PsSoftLightBlend, ps_soft_light_blend);
    SET_BLEND_MIN_FUNCTIONS(PsOverlayBlend, ps_overlay_blend);
    // ... 共 16 种 PS 混合模式

    // 拷贝/填充操作
    TVPCopyColor = TVP_color_copy;
    TVPCopyMask  = TVP_alpha_copy;
    TVPFillARGB  = TVP_fill_argb;
    TVPFillColor = TVP_const_color_copy;

    // 颜色映射（文字渲染用）
    TVPApplyColorMap   = TVP_apply_color_map;
    TVPApplyColorMap65 = TVP_apply_color_map65; // 65 级反走样

    // Photoshop 混合查找表初始化
    TVPPsInitTable();
}
```

## Duff's Device 循环展开

tvpgl.cpp 中最显著的代码模式是 **Duff's Device**——一种利用 C 语言 `switch-case` 穿透特性实现循环展开的经典技巧。这不是现代编译器的自动向量化，而是手工展开，确保每次循环处理 4 个像素。

### 原理解析

```cpp
// 标准循环（每次迭代处理 1 个像素）
for (int i = 0; i < len; i++) {
    process(dest[i], src[i]);
}

// Duff's Device 展开（每次迭代处理 4 个像素）
// 项目源码：tvpgl.cpp 第 309-368 行 TVPAlphaBlend_c
if (len > 0) {
    int lu_n = (len + 3) / 4;  // 向上取整的循环次数
    switch (len % 4) {          // 根据余数跳到对应 case
        case 0: do {
                    BLEND_ONE_PIXEL;  // 处理第 1 个
            case 3: BLEND_ONE_PIXEL;  // 处理第 2 个
            case 2: BLEND_ONE_PIXEL;  // 处理第 3 个
            case 1: BLEND_ONE_PIXEL;  // 处理第 4 个
                } while (--lu_n);
    }
}
```

工作流程（假设 `len = 7`）：
1. `lu_n = (7+3)/4 = 2`，循环 2 次
2. `len % 4 = 3`，首次进入跳到 `case 3`
3. 首次循环：执行 case 3→2→1，处理 3 个像素
4. 第二次循环：从 case 0 开始，执行 0→3→2→1，处理 4 个像素
5. 总计 3 + 4 = 7 个像素，与 `len` 一致

### 完整示例：模拟 Duff's Device 混合

```cpp
#include <cstdint>
#include <cstdio>
#include <cstring>

using tjs_uint32 = uint32_t;

// 简化的 Alpha 混合（单像素）
static inline tjs_uint32 blend_pixel(tjs_uint32 d, tjs_uint32 s) {
    tjs_uint32 sopa = s >> 24;  // 源不透明度
    tjs_uint32 d1 = d & 0xff00ff;
    d1 = (d1 + (((s & 0xff00ff) - d1) * sopa >> 8)) & 0xff00ff;
    tjs_uint32 dg = d & 0xff00;
    tjs_uint32 sg = s & 0xff00;
    return d1 + ((dg + ((sg - dg) * sopa >> 8)) & 0xff00);
}

// Duff's Device 版本
void alpha_blend_duff(tjs_uint32* dest, const tjs_uint32* src, int len) {
    if (len > 0) {
        int lu_n = (len + 3) / 4;
        switch (len % 4) {
            case 0: do {
                        *dest = blend_pixel(*dest, *src); dest++; src++;
                case 3: *dest = blend_pixel(*dest, *src); dest++; src++;
                case 2: *dest = blend_pixel(*dest, *src); dest++; src++;
                case 1: *dest = blend_pixel(*dest, *src); dest++; src++;
                    } while (--lu_n);
        }
    }
}

int main() {
    // 4 个半透明红色像素（A=128, R=255, G=0, B=0）
    tjs_uint32 src[7] = {
        0x80FF0000, 0x80FF0000, 0x80FF0000, 0x80FF0000,
        0x80FF0000, 0x80FF0000, 0x80FF0000
    };
    // 4 个不透明蓝色像素（A=255, R=0, G=0, B=255）
    tjs_uint32 dest[7] = {
        0xFF0000FF, 0xFF0000FF, 0xFF0000FF, 0xFF0000FF,
        0xFF0000FF, 0xFF0000FF, 0xFF0000FF
    };

    alpha_blend_duff(dest, src, 7); // 处理 7 个像素

    for (int i = 0; i < 7; i++) {
        printf("pixel[%d] = 0x%08X (R=%u G=%u B=%u)\n",
               i, dest[i],
               (dest[i] >> 16) & 0xFF,
               (dest[i] >> 8) & 0xFF,
               dest[i] & 0xFF);
    }
    // 每个像素：50% 红色 + 50% 蓝色 → 紫色 (R≈128, B≈127)
    return 0;
}
```

### 与现代循环展开的对比

| 方法 | 优点 | 缺点 |
|------|------|------|
| Duff's Device | 保证精确的展开因子；无需额外的尾部处理 | 代码可读性差；现代编译器可能无法自动向量化 |
| 手动展开（主循环+尾循环） | 清晰明了；编译器友好 | 需要单独处理余数像素 |
| 编译器自动向量化 | 零代码量；编译器选择最优 SIMD | 依赖编译器优化水平；不可控 |

KrKr2 在 `blend_function.cpp` 中的模板方案已经改用标准 `for` 循环，依靠编译器自动向量化（使用 `__restrict` 提示无别名）。而 `tvpgl.cpp` 中的 Duff's Device 是历史遗留实现，作为参考和回退路径保留。

## 饱和加法：TVPSaturatedAdd

tvpgl 中另一个核心的位操作技巧是 **无分支饱和加法**（Branchless Saturated Addition）。当两个 8 位通道值相加超过 255 时，需要钳位到 255 而非溢出：

```cpp
// 项目源码：tvpgl.h 第 73-80 行
static tjs_uint32 TVPSaturatedAdd(tjs_uint32 a, tjs_uint32 b) {
    // 同时对 4 个通道做饱和加法，无分支
    tjs_uint32 tmp = ((a & b) + (((a ^ b) >> 1) & 0x7f7f7f7f))
                     & 0x80808080;
    tmp = (tmp << 1) - (tmp >> 7);
    return (a + b - tmp) | tmp;
}
```

算法分解（以单通道为例）：
1. `(a & b) + ((a ^ b) >> 1)`：无溢出地计算 `(a + b) / 2`
2. `& 0x80`：检查最高位——如果为 1，说明 `a+b >= 256`（溢出）
3. `(tmp << 1) - (tmp >> 7)`：溢出位扩展为 `0xFF` 掩码
4. `(a + b - tmp) | tmp`：如果溢出则结果为 255（`0xFF`）

```cpp
#include <cstdint>
#include <cstdio>

using tjs_uint32 = uint32_t;

// 项目中的无分支饱和加法
static inline tjs_uint32 TVPSaturatedAdd(tjs_uint32 a, tjs_uint32 b) {
    tjs_uint32 tmp = ((a & b) + (((a ^ b) >> 1) & 0x7f7f7f7f)) & 0x80808080;
    tmp = (tmp << 1) - (tmp >> 7);
    return (a + b - tmp) | tmp;
}

int main() {
    // 正常加法：100 + 50 = 150
    tjs_uint32 r1 = TVPSaturatedAdd(0x00640000, 0x00320000);
    printf("100+50 = %u\n", (r1 >> 16) & 0xFF); // 输出 150

    // 饱和加法：200 + 100 = 255（不是 300）
    tjs_uint32 r2 = TVPSaturatedAdd(0x00C80000, 0x00640000);
    printf("200+100 = %u\n", (r2 >> 16) & 0xFF); // 输出 255

    // 多通道同时运算
    tjs_uint32 pixel_a = 0x80C86432; // A=128 R=200 G=100 B=50
    tjs_uint32 pixel_b = 0x40644B1E; // A=64  R=100 G=75  B=30
    tjs_uint32 result = TVPSaturatedAdd(pixel_a, pixel_b);
    printf("multi: A=%u R=%u G=%u B=%u\n",
           result >> 24,
           (result >> 16) & 0xFF,
           (result >> 8) & 0xFF,
           result & 0xFF);
    // 输出：A=192 R=255(饱和) G=175 B=80
    return 0;
}
```

## 动手实践

### 实践 1：手写一个查找表加速的混合函数

```cpp
#include <cstdint>
#include <cstdio>
#include <cstring>

using tjs_uint32 = uint32_t;

// 构建简化版 NegativeMul 表
// 公式：result = 255 - (255-a)*(255-b)/255
// 用途：合并两层不透明度
static unsigned char NegMulTable[256 * 256];

void init_negmul_table() {
    for (int a = 0; a < 256; a++) {
        for (int b = 0; b < 256; b++) {
            NegMulTable[b * 256 + a] =
                (unsigned char)(255 - (255 - a) * (255 - b) / 255);
        }
    }
}

// 使用查表合并两个不透明度
uint8_t combine_opacity(uint8_t opa1, uint8_t opa2) {
    return NegMulTable[opa2 * 256 + opa1];
}

int main() {
    init_negmul_table();
    // 两层各 50% 不透明度叠加
    printf("128+128 -> %u\n", combine_opacity(128, 128)); // ≈192 (75%)
    // 两层各 100% 不透明度
    printf("255+255 -> %u\n", combine_opacity(255, 255)); // =255 (100%)
    // 0% + 任何 = 原值
    printf("0+200 -> %u\n", combine_opacity(0, 200));     // =200
    return 0;
}
```

## 对照项目源码

相关文件：
- `cpp/core/visual/tvpgl.h` 第 48-56 行 —— `TVP_GL_FUNC_PTR_EXTERN_DECL` 宏定义，函数指针与 C 默认实现的双重声明
- `cpp/core/visual/tvpgl.h` 第 73-80 行 —— `TVPSaturatedAdd` 无分支饱和加法内联函数
- `cpp/core/visual/tvpgl.h` 第 215-238 行 —— `TVPAlphaBlend` 系列 8 个变体的函数指针声明
- `cpp/core/visual/tvpgl.cpp` 第 29-45 行 —— 全局查找表定义（DivTable, OpacityOnOpacityTable 等）
- `cpp/core/visual/tvpgl.cpp` 第 224-301 行 —— `TVPCreateTable()` 查找表初始化
- `cpp/core/visual/tvpgl.cpp` 第 306-368 行 —— `TVPAlphaBlend_c` Duff's Device 实现
- `cpp/core/visual/gl/blend_function.cpp` 第 590-685 行 —— `TVPGL_C_Init()` 函数指针绑定

### 常见错误与解决方案

**错误 1：并行位操作中的通道溢出**

当两个通道的差值乘以不透明度后，如果掩码操作遗漏，一个通道的高位会溢出到相邻通道：

```cpp
// ❌ 错误：遗漏掩码导致 R 通道溢出到 G
tjs_uint32 d1 = d & 0xff00ff;
d1 = d1 + ((s & 0xff00ff) - d1) * sopa >> 8;  // 缺少 & 0xff00ff

// ✅ 正确：先乘再掩码
d1 = (d1 + (((s & 0xff00ff) - d1) * sopa >> 8)) & 0xff00ff;
```

**错误 2：Duff's Device 中忘记处理 len == 0**

Duff's Device 要求 `len > 0`，否则 `(len + 3) / 4` 仍为 0，`do-while` 至少执行一次，导致非法内存访问：

```cpp
// ❌ 错误：未检查 len
int lu_n = (len + 3) / 4;
switch (len % 4) { ... }

// ✅ 正确：先检查 len
if (len > 0) {
    int lu_n = (len + 3) / 4;
    switch (len % 4) { ... }
}
```

**错误 3：查找表索引计算错误**

`TVPDivTable` 的索引是 `(除数 << 8) + 被除数`，而非 `(被除数 << 8) + 除数`。搞反会导致计算结果完全错误：

```cpp
// ❌ 错误：被除数和除数位置颠倒
uint8_t result = TVPDivTable[(pixel_value << 8) + alpha];

// ✅ 正确：除数在高字节
uint8_t result = TVPDivTable[(alpha << 8) + pixel_value];
```

## 本节小结

- tvpgl 是 KrKr2 的底层像素操作库，由 `gengl.pl` 脚本生成，提供 100+ 个 CPU 端像素处理函数
- 像素采用 32 位 ARGB 打包格式，通过 `0xff00ff` 掩码实现 R+B 双通道并行处理
- 预计算查找表（TVPDivTable、TVPOpacityOnOpacityTable 等）在初始化时一次构建，避免运行时除法和浮点运算
- 函数指针分发（`TVP_GL_FUNC_PTR`）允许运行时在 C 实现和 SSE/ASM 优化实现之间切换
- Duff's Device 循环展开是历史遗留的手工优化，在模板方案（`blend_function.cpp`）中已改用标准循环

## 练习题与答案

### 题目 1：实现一个使用查找表的 ConstAlphaBlend 函数

实现 `const_alpha_blend` 函数：将 `src` 扫描线以固定不透明度 `opa`（0-255）混合到 `dest` 上。不使用浮点运算。

<details>
<summary>查看答案</summary>

```cpp
#include <cstdint>
#include <cstdio>

using tjs_uint32 = uint32_t;

void const_alpha_blend(tjs_uint32* dest, const tjs_uint32* src,
                       int len, int opa) {
    for (int i = 0; i < len; i++) {
        tjs_uint32 s = src[i];
        tjs_uint32 d = dest[i];
        // R+B 双通道并行
        tjs_uint32 d1 = d & 0xff00ff;
        d1 = (d1 + (((s & 0xff00ff) - d1) * opa >> 8)) & 0xff00ff;
        // G 通道单独处理
        tjs_uint32 dg = d & 0xff00;
        tjs_uint32 sg = s & 0xff00;
        dest[i] = d1 + ((dg + ((sg - dg) * opa >> 8)) & 0xff00);
    }
}

int main() {
    tjs_uint32 src[4]  = {0xFFFF0000, 0xFFFF0000, 0xFFFF0000, 0xFFFF0000};
    tjs_uint32 dest[4] = {0xFF0000FF, 0xFF0000FF, 0xFF0000FF, 0xFF0000FF};
    const_alpha_blend(dest, src, 4, 128); // 50% 混合
    for (int i = 0; i < 4; i++) {
        printf("pixel[%d] R=%u G=%u B=%u\n", i,
               (dest[i] >> 16) & 0xFF,
               (dest[i] >> 8) & 0xFF,
               dest[i] & 0xFF);
    }
    // 输出：每个像素 R≈128 G=0 B≈127
    return 0;
}
```

</details>

### 题目 2：解释 TVPSaturatedAdd 中 `(tmp << 1) - (tmp >> 7)` 的作用

`tmp` 的值是 `0x80808080` 格式（每个通道只有最高位）。解释这行代码如何将单个溢出标志位扩展为全通道掩码。

<details>
<summary>查看答案</summary>

`tmp` 在每个通道中只有 bit 7 有意义（`0x80` 或 `0x00`）：

- 如果通道未溢出：bit 7 = 0 → `tmp` 该字节为 `0x00`
- 如果通道溢出：bit 7 = 1 → `tmp` 该字节为 `0x80`

`(tmp << 1) - (tmp >> 7)` 的作用：
- `tmp << 1`：`0x80` → `0x100`（进位到下一通道），但只看当前通道等于 `0x00`
- `tmp >> 7`：`0x80` → `0x01`
- `0x00 - 0x01`（模 256）= `0xFF`

所以每个溢出通道得到 `0xFF` 掩码，非溢出通道保持 `0x00`。

最终 `(a + b - tmp) | tmp`：
- 溢出通道：`(a+b-0xFF) | 0xFF = 0xFF`（饱和到 255）
- 非溢出通道：`(a+b-0x00) | 0x00 = a+b`（正常相加）

实际上 `tmp << 1` 会跨通道进位，但因为后续是 `|` 操作，溢出位只会让高位通道更倾向于饱和，而不会产生错误结果。这是一种利用整数溢出特性的巧妙无分支编程。

</details>

### 题目 3：将 Duff's Device 版本改写为主循环+尾循环的等价实现

<details>
<summary>查看答案</summary>

```cpp
#include <cstdint>
#include <cstdio>

using tjs_uint32 = uint32_t;

static inline tjs_uint32 blend_one(tjs_uint32 d, tjs_uint32 s) {
    tjs_uint32 sopa = s >> 24;
    tjs_uint32 d1 = d & 0xff00ff;
    d1 = (d1 + (((s & 0xff00ff) - d1) * sopa >> 8)) & 0xff00ff;
    tjs_uint32 dg = d & 0xff00;
    tjs_uint32 sg = s & 0xff00;
    return d1 + ((dg + ((sg - dg) * sopa >> 8)) & 0xff00);
}

// 主循环 + 尾循环版本（等价于 Duff's Device 4x 展开）
void alpha_blend_unrolled(tjs_uint32* dest, const tjs_uint32* src, int len) {
    int i = 0;
    // 主循环：每次处理 4 个像素
    int main_count = len & ~3;  // len / 4 * 4
    for (; i < main_count; i += 4) {
        dest[i+0] = blend_one(dest[i+0], src[i+0]);
        dest[i+1] = blend_one(dest[i+1], src[i+1]);
        dest[i+2] = blend_one(dest[i+2], src[i+2]);
        dest[i+3] = blend_one(dest[i+3], src[i+3]);
    }
    // 尾循环：处理剩余 0-3 个像素
    for (; i < len; i++) {
        dest[i] = blend_one(dest[i], src[i]);
    }
}

int main() {
    tjs_uint32 src[7], dest[7];
    for (int i = 0; i < 7; i++) {
        src[i]  = 0x80FF0000; // 半透明红
        dest[i] = 0xFF0000FF; // 不透明蓝
    }
    alpha_blend_unrolled(dest, src, 7);
    for (int i = 0; i < 7; i++) {
        printf("pixel[%d] = 0x%08X\n", i, dest[i]);
    }
    return 0;
}
```

主循环+尾循环的优势：代码可读性更好，编译器更容易自动向量化（用 `-O2 -mavx2` 编译时，主循环可能被优化为 SIMD 指令）。

</details>

## 下一步

[ARGB 混合与合成](./02-ARGB混合与合成.md) —— 深入 blend_functor_c.h 的模板混合系统，了解加算合成、Photoshop 兼容混合等 20+ 种混合模式的实现原理。
