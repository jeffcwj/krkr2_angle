# ARM NEON 与运行时 CPU 分发

> **所属模块：** P05-软件渲染原理 · 第 4 章 · 第 3 节  
> **前置知识：** [为什么需要 SIMD](./01-为什么需要SIMD.md)、[SSE2 基础与像素并行](./02-SSE2基础与像素并行.md)、Alpha 混合公式、C/C++ 预处理器  
> **预计阅读时间：** 40 分钟

---

## 本节目标

读完本节后，你将能够：

1. 理解 ARM NEON 指令集的定位——移动/嵌入式平台（Android、iOS、Apple Silicon Mac）的 128 位 SIMD
2. 掌握 NEON 的三种核心数据类型及其与 SSE2 的对照关系
3. 使用 NEON intrinsic 函数完成像素加载、解包、混合、打包全流程
4. 理解 NEON Alpha 广播的 `vtbl`/`vuzp` 优化手法
5. 实现运行时 CPU 特性检测（x86 CPUID + ARM 编译期检测）
6. 掌握 KrKr2 使用的"函数指针分发"架构模式
7. 正确编写跨平台 SIMD 条件编译代码，避免 Apple Silicon 陷阱

---

## 1. 为什么要学 NEON

### 1.1 ARM 处理器的 SIMD

在上两节中，我们学习了 x86 平台的 SSE2 指令集。但 KrKr2 的目标平台不止 PC——它还要在 **Android**（ARM 处理器）和 **Apple Silicon Mac**（ARM 架构的 M1/M2/M3 芯片）上运行。ARM 处理器没有 SSE2，取而代之的是 **NEON**（Advanced SIMD Extension，ARM 的高级 SIMD 扩展指令集）。

NEON 与 SSE2 的核心思想完全一致——用一条指令同时处理多个数据。不同之处在于：

| 对比项 | SSE2 (x86) | NEON (ARM) |
|--------|-----------|------------|
| 寄存器宽度 | 128 位（`xmm0`-`xmm15`） | 128 位（`q0`-`q15`，64 位模式下有 `q0`-`q31`） |
| 寄存器数量 | 16 个（64 位模式） | 16 个（32 位）/ 32 个（64 位 AArch64） |
| 头文件 | `<emmintrin.h>` | `<arm_neon.h>` |
| 可用性 | x86-64 默认支持 | ARMv8（AArch64）**始终可用**，ARMv7 需运行时检测 |
| 数据类型命名 | 统一 `__m128i` | 按元素类型区分：`uint8x16_t`、`uint16x8_t` 等 |

> **关键区别**：SSE2 只有一个整数类型 `__m128i`，加载出来是"无类型"的 128 位数据，靠指令决定怎么解释。NEON 的类型名直接告诉你"里面装的是什么"——`uint8x16_t` 就是 16 个 `uint8_t`，`uint32x4_t` 就是 4 个 `uint32_t`。这让 NEON 代码比 SSE2 更易读。

### 1.2 KrKr2 的 ARM NEON 需求

KrKr2 的 Android 版需要在 ARM64 设备上高效渲染。像素混合、图像缩放等热点操作如果只用标量 C 代码，在移动设备上会比 PC 更慢（ARM 处理器主频通常低于桌面 x86）。因此 KrKr2 在 `tvpgl.h` 中预留了 NEON 实现的函数指针插槽——虽然当前主要实现是 SSE2/AVX2，但整体架构支持随时插入 NEON 版本。

---

## 2. NEON 数据类型详解

### 2.1 类型命名规则

NEON 的类型命名遵循严格的模式：`<基本类型>x<通道数>_t`。

```cpp
#include <arm_neon.h>

// ── 8 位元素：每个通道 1 字节 ──
uint8x16_t  v8;    // 16 个 uint8_t，共 128 位 —— 正好装 16 个像素通道
uint8x8_t   v8h;   // 8 个 uint8_t，共 64 位（半宽寄存器，用 d0-d31 表示）
int8x16_t   s8;    // 16 个 int8_t（有符号版本）

// ── 16 位元素：每个通道 2 字节 ──
uint16x8_t  v16;   // 8 个 uint16_t，共 128 位 —— 像素解包后的中间结果
uint16x4_t  v16h;  // 4 个 uint16_t，共 64 位

// ── 32 位元素：每个通道 4 字节 ──
uint32x4_t  v32;   // 4 个 uint32_t，共 128 位 —— 正好装 4 个 ARGB 像素
uint32x2_t  v32h;  // 2 个 uint32_t，共 64 位

// ── 浮点元素 ──
float32x4_t vf;    // 4 个 float，共 128 位 —— 用于浮点运算（纹理坐标等）
```

> **与 SSE2 对照**：SSE2 中 `__m128i` 可以当 16×uint8 用也可以当 4×uint32 用，全靠你选什么指令。NEON 在**类型层面**就区分好了——编译器能帮你检查类型是否匹配，减少 bug。

### 2.2 半宽寄存器（64 位）

NEON 的一个独特设计是支持 **64 位半宽操作**。一个 128 位 `q` 寄存器可以拆成两个 64 位 `d` 寄存器：

```cpp
uint8x16_t full;        // 128 位完整寄存器（q 寄存器）

// 取出低 64 位和高 64 位
uint8x8_t lo = vget_low_u8(full);   // 低 8 字节（d 寄存器）
uint8x8_t hi = vget_high_u8(full);  // 高 8 字节（d 寄存器）

// 合并两个半宽回完整寄存器
uint8x16_t combined = vcombine_u8(lo, hi);
```

这在像素处理中非常有用——例如解包（Unpack，将 8 位元素扩展到 16 位）时，先取低半部分扩展，再取高半部分扩展：

```cpp
uint8x16_t pixels;      // 16 个 uint8 通道值

// 解包低 8 个通道到 16 位（零扩展）
uint16x8_t lo16 = vmovl_u8(vget_low_u8(pixels));
// 解包高 8 个通道到 16 位（零扩展）
uint16x8_t hi16 = vmovl_u8(vget_high_u8(pixels));
```

> **`vmovl`** = Vector MOVe Long，将窄类型扩展为宽类型（8→16、16→32）。这等价于 SSE2 的 `_mm_unpacklo_epi8(a, zero)` + `_mm_unpackhi_epi8(a, zero)` 组合，但语义更清晰。

---

## 3. SSE2 与 NEON 指令对照表

下表列出像素处理中最常用的操作。**建议收藏**——编写跨平台 SIMD 代码时随时查阅：

| 操作 | SSE2 intrinsic | NEON intrinsic | 说明 |
|------|---------------|----------------|------|
| 加载 128 位（非对齐） | `_mm_loadu_si128` | `vld1q_u8` / `vld1q_u32` | NEON 按元素类型区分 |
| 存储 128 位（非对齐） | `_mm_storeu_si128` | `vst1q_u8` / `vst1q_u32` | 同上 |
| 对齐加载 | `_mm_load_si128` | `vld1q_u8`（NEON 不区分对齐） | NEON 硬件处理非对齐，无需区分 |
| 全零寄存器 | `_mm_setzero_si128()` | `vdupq_n_u8(0)` | `vdup` = 广播标量到全部通道 |
| 广播标量到全部通道 | `_mm_set1_epi16(x)` | `vdupq_n_u16(x)` | 像素混合时广播 Alpha 值 |
| 8 位无符号加法 | `_mm_add_epi8` | `vaddq_u8` | 会回绕溢出 |
| 8 位饱和加法 | `_mm_adds_epu8` | `vqaddq_u8` | 结果钳制在 [0,255]，`q`=saturating（饱和） |
| 16 位乘法（低 16 位） | `_mm_mullo_epi16` | `vmulq_u16` | Alpha × 通道值 |
| 解包低半（8→16） | `_mm_unpacklo_epi8(a, zero)` | `vmovl_u8(vget_low_u8(a))` | NEON 语义更直接 |
| 解包高半（8→16） | `_mm_unpackhi_epi8(a, zero)` | `vmovl_u8(vget_high_u8(a))` | 同上 |
| 打包（16→8 饱和） | `_mm_packus_epi16(lo, hi)` | `vcombine_u8(vqmovn_u16(lo), vqmovn_u16(hi))` | `vqmovn` = 饱和窄化 |
| 右移 16 位通道 | `_mm_srli_epi16(a, n)` | `vshrq_n_u16(a, n)` | 除以 2^n |
| 按位与 | `_mm_and_si128` | `vandq_u8` | 掩码操作 |
| 按位或 | `_mm_or_si128` | `vorrq_u8` | 合并结果 |
| 16 位减法 | `_mm_sub_epi16` | `vsubq_u16` | 计算 `255 - alpha` |

> **NEON 的"无对齐惩罚"**：在 SSE2 中，`_mm_load_si128` 要求地址 16 字节对齐，否则会触发段错误（Segmentation Fault，程序因非法内存访问被操作系统终止）；`_mm_loadu_si128` 不要求对齐但可能稍慢。NEON 的 `vld1q` 系列不区分对齐/非对齐——ARMv8 硬件自动处理非对齐访问，几乎无性能损失。

---

## 4. NEON Alpha 混合完整实现

### 4.1 基础版本（逐元素广播 Alpha）

回顾 Alpha 混合公式（上一节已推导）：

```
result = src × srcAlpha + dst × (255 - srcAlpha)
```

SSE2 版本的核心挑战是"把每个像素的 Alpha 值广播到该像素的 4 个通道"。NEON 也面临同样挑战，但解决方式不同。先看一个最直白的实现：

```cpp
#include <arm_neon.h>
#include <cstdint>

// NEON Alpha 混合 —— 每次处理 4 个 ARGB 像素（128 位）
// 参数: dst/src 为 ARGB 像素数组，count 为像素个数（应为 4 的倍数）
void alpha_blend_neon(uint32_t* dst, const uint32_t* src, int count) {
    // 预先准备常量向量
    uint16x8_t v255 = vdupq_n_u16(255);    // 8 个 255
    uint16x8_t v128 = vdupq_n_u16(128);    // 用于 ÷255 近似的舍入偏移

    for (int i = 0; i < count; i += 4) {
        // ── 步骤 1：加载 4 个像素（16 字节 = 128 位）──
        // vld1q_u8: 按 uint8 加载 16 个字节
        uint8x16_t s = vld1q_u8((const uint8_t*)(src + i));
        uint8x16_t d = vld1q_u8((const uint8_t*)(dst + i));

        // ── 步骤 2：解包到 16 位，防止乘法溢出 ──
        // 像素内存布局（ARGB）: [B0 G0 R0 A0 | B1 G1 R1 A1 | B2 G2 R2 A2 | B3 G3 R3 A3]
        //                          ^^^^^^^^^^^ 低 8 字节       ^^^^^^^^^^^ 高 8 字节
        uint16x8_t s_lo = vmovl_u8(vget_low_u8(s));   // 前 2 像素的 8 通道 → 16 位
        uint16x8_t s_hi = vmovl_u8(vget_high_u8(s));  // 后 2 像素的 8 通道 → 16 位
        uint16x8_t d_lo = vmovl_u8(vget_low_u8(d));
        uint16x8_t d_hi = vmovl_u8(vget_high_u8(d));

        // ── 步骤 3：提取 Alpha 并广播 ──
        // ARGB 中 Alpha 在每像素的第 4 个字节（索引 3, 7）
        // s_lo 包含 [B0 G0 R0 A0 B1 G1 R1 A1] 共 8 个 uint16
        //            0   1  2  3  4  5  6  7
        uint16_t a0 = vgetq_lane_u16(s_lo, 3);  // 像素 0 的 Alpha
        uint16_t a1 = vgetq_lane_u16(s_lo, 7);  // 像素 1 的 Alpha
        uint16_t a2 = vgetq_lane_u16(s_hi, 3);  // 像素 2 的 Alpha
        uint16_t a3 = vgetq_lane_u16(s_hi, 7);  // 像素 3 的 Alpha

        // 广播 Alpha 到每像素的 4 个通道
        // 像素 0 和 1 共用 sa_lo（8 个 uint16）
        uint16x8_t sa_lo = vdupq_n_u16(0);
        sa_lo = vsetq_lane_u16(a0, sa_lo, 0);  // B0 通道用 A0
        sa_lo = vsetq_lane_u16(a0, sa_lo, 1);  // G0 通道用 A0
        sa_lo = vsetq_lane_u16(a0, sa_lo, 2);  // R0 通道用 A0
        sa_lo = vsetq_lane_u16(a0, sa_lo, 3);  // A0 通道用 A0
        sa_lo = vsetq_lane_u16(a1, sa_lo, 4);  // B1 通道用 A1
        sa_lo = vsetq_lane_u16(a1, sa_lo, 5);  // G1 通道用 A1
        sa_lo = vsetq_lane_u16(a1, sa_lo, 6);  // R1 通道用 A1
        sa_lo = vsetq_lane_u16(a1, sa_lo, 7);  // A1 通道用 A1

        // 像素 2 和 3 同理
        uint16x8_t sa_hi = vdupq_n_u16(0);
        sa_hi = vsetq_lane_u16(a2, sa_hi, 0);
        sa_hi = vsetq_lane_u16(a2, sa_hi, 1);
        sa_hi = vsetq_lane_u16(a2, sa_hi, 2);
        sa_hi = vsetq_lane_u16(a2, sa_hi, 3);
        sa_hi = vsetq_lane_u16(a3, sa_hi, 4);
        sa_hi = vsetq_lane_u16(a3, sa_hi, 5);
        sa_hi = vsetq_lane_u16(a3, sa_hi, 6);
        sa_hi = vsetq_lane_u16(a3, sa_hi, 7);

        // ── 步骤 4：计算 (255 - alpha) ──
        uint16x8_t inv_lo = vsubq_u16(v255, sa_lo);
        uint16x8_t inv_hi = vsubq_u16(v255, sa_hi);

        // ── 步骤 5：混合公式 result = (src*alpha + dst*inv_alpha + 128) >> 8 ──
        // 这里使用 KrKr2 的 ÷255 近似：(x + (x>>8) + 1) >> 8
        // 但简化版用 (x + 128) >> 8 也可接受（误差 ≤1）
        uint16x8_t res_lo = vaddq_u16(
            vmulq_u16(s_lo, sa_lo),       // src * alpha
            vmulq_u16(d_lo, inv_lo)       // dst * (255 - alpha)
        );
        res_lo = vshrq_n_u16(vaddq_u16(res_lo, v128), 8);  // (x + 128) >> 8

        uint16x8_t res_hi = vaddq_u16(
            vmulq_u16(s_hi, sa_hi),
            vmulq_u16(d_hi, inv_hi)
        );
        res_hi = vshrq_n_u16(vaddq_u16(res_hi, v128), 8);

        // ── 步骤 6：打包回 8 位并存储 ──
        // vqmovn_u16: 将 uint16 饱和窄化为 uint8（超过 255 钳制为 255）
        uint8x8_t r_lo = vqmovn_u16(res_lo);  // 前 2 像素
        uint8x8_t r_hi = vqmovn_u16(res_hi);  // 后 2 像素
        uint8x16_t result = vcombine_u8(r_lo, r_hi);  // 合并为 128 位

        // vst1q_u8: 存储 16 字节回内存
        vst1q_u8((uint8_t*)(dst + i), result);
    }
}
```

上面的代码能正确工作，但 `vsetq_lane_u16` 逐通道设置效率较低——每个 `vsetq_lane` 在某些 ARM 核心上会产生一次寄存器写入。下面介绍更高效的 Alpha 广播方法。

### 4.2 优化版本：使用 vtbl 查表广播 Alpha

**`vtbl`**（Vector Table Lookup，向量查表指令）是 NEON 的"字节洗牌"指令，类似 SSE 中的 `_mm_shuffle_epi8`（SSSE3）。它根据索引表从源寄存器中任意选取字节：

```cpp
// vtbl 的作用：根据 idx 中的索引，从 tbl 中挑选字节
// idx 中每个字节的值是 tbl 中的字节位置（0-15）
// 如果索引超出范围（≥16），结果字节为 0

// 用 vtbl 实现 Alpha 广播（ARMv8 版本使用 vqtbl1q_u8）
// 目标：把 [B0 G0 R0 A0 B1 G1 R1 A1 ...] 中的 A0 广播到 B0/G0/R0 位置
static const uint8_t alpha_shuffle_table[16] = {
    3, 3, 3, 3,     // 像素 0：全部取索引 3（即 A0）
    7, 7, 7, 7,     // 像素 1：全部取索引 7（即 A1）
    11, 11, 11, 11, // 像素 2：全部取索引 11（即 A2）
    15, 15, 15, 15  // 像素 3：全部取索引 15（即 A3）
};

void alpha_blend_neon_fast(uint32_t* dst, const uint32_t* src, int count) {
    // 加载洗牌表（只需加载一次，循环外）
    uint8x16_t shuffle = vld1q_u8(alpha_shuffle_table);
    uint16x8_t v255 = vdupq_n_u16(255);
    uint16x8_t v128 = vdupq_n_u16(128);

    for (int i = 0; i < count; i += 4) {
        uint8x16_t s = vld1q_u8((const uint8_t*)(src + i));
        uint8x16_t d = vld1q_u8((const uint8_t*)(dst + i));

        // 一条指令完成 Alpha 广播！
        // vqtbl1q_u8: ARMv8 的 128 位查表指令
        // 结果: [A0 A0 A0 A0 | A1 A1 A1 A1 | A2 A2 A2 A2 | A3 A3 A3 A3]
        uint8x16_t alpha_bytes = vqtbl1q_u8(s, shuffle);

        // 解包源/目标到 16 位
        uint16x8_t s_lo = vmovl_u8(vget_low_u8(s));
        uint16x8_t s_hi = vmovl_u8(vget_high_u8(s));
        uint16x8_t d_lo = vmovl_u8(vget_low_u8(d));
        uint16x8_t d_hi = vmovl_u8(vget_high_u8(d));

        // 解包 Alpha 到 16 位
        uint16x8_t a_lo = vmovl_u8(vget_low_u8(alpha_bytes));
        uint16x8_t a_hi = vmovl_u8(vget_high_u8(alpha_bytes));

        // 计算 (255 - alpha)
        uint16x8_t inv_lo = vsubq_u16(v255, a_lo);
        uint16x8_t inv_hi = vsubq_u16(v255, a_hi);

        // 混合
        uint16x8_t res_lo = vshrq_n_u16(
            vaddq_u16(
                vaddq_u16(vmulq_u16(s_lo, a_lo), vmulq_u16(d_lo, inv_lo)),
                v128
            ), 8);
        uint16x8_t res_hi = vshrq_n_u16(
            vaddq_u16(
                vaddq_u16(vmulq_u16(s_hi, a_hi), vmulq_u16(d_hi, inv_hi)),
                v128
            ), 8);

        // 打包并存储
        vst1q_u8((uint8_t*)(dst + i),
                 vcombine_u8(vqmovn_u16(res_lo), vqmovn_u16(res_hi)));
    }
}
```

> **性能差距**：`vsetq_lane` 版本的 Alpha 广播需要 8-16 条指令（提取 + 逐个设置），`vqtbl1q_u8` 只需 **1 条指令**。在 1920×1080 全屏混合中，这个差距会累积为数毫秒——对 60fps 渲染来说很显著。

### 4.3 尾部像素处理

与 SSE2 一样，当像素数量不是 4 的倍数时需要处理尾部。策略也相同——回退到标量：

```cpp
void alpha_blend_neon_safe(uint32_t* dst, const uint32_t* src, int count) {
    int simd_count = count & ~3;  // 向下取整到 4 的倍数

    // NEON 主循环
    alpha_blend_neon_fast(dst, src, simd_count);

    // 标量处理剩余 0-3 个像素
    for (int i = simd_count; i < count; ++i) {
        uint32_t s = src[i], d = dst[i];
        uint32_t sa = (s >> 24) & 0xFF;        // 提取 Alpha
        uint32_t inv = 255 - sa;
        uint32_t rb = (((s & 0xFF00FF) * sa + (d & 0xFF00FF) * inv + 0x800080) >> 8) & 0xFF00FF;
        uint32_t g  = (((s & 0x00FF00) * sa + (d & 0x00FF00) * inv + 0x008000) >> 8) & 0x00FF00;
        dst[i] = rb | g | (sa << 24);
    }
}
```

---

## 5. 运行时 CPU 特性检测

### 5.1 为什么需要运行时检测

一个可执行文件可能在不同硬件上运行：

- **Windows PC**：可能是 2005 年只支持 SSE2 的老 CPU，也可能是 2023 年支持 AVX-512 的新 CPU
- **Android**：可能是只有 ARMv7（需要检测 NEON 是否可用）的旧手机，也可能是 ARMv8（NEON 始终可用）的新手机

如果程序编译时就硬编码使用 AVX2 指令，那么在不支持 AVX2 的 CPU 上运行会直接崩溃（触发"非法指令"异常，即 `SIGILL` 信号）。所以我们需要在**程序启动时**检测 CPU 支持哪些指令集，然后选择对应的实现版本。

### 5.2 x86 平台：CPUID 指令

x86 处理器提供 `CPUID` 指令来查询 CPU 特性。在 C/C++ 中通过编译器提供的 intrinsic 函数调用：

```cpp
#ifdef _MSC_VER
    #include <intrin.h>      // MSVC 编译器提供 __cpuid / __cpuidex
#elif defined(__GNUC__) || defined(__clang__)
    #include <cpuid.h>       // GCC/Clang 提供 __get_cpuid / __cpuid_count
#endif

struct CpuFeatures {
    bool has_sse2  = false;  // 2001 年 Pentium 4 引入，x86-64 必备
    bool has_ssse3 = false;  // 2006 年 Core 2 引入，含 pshufb 字节洗牌
    bool has_sse41 = false;  // 2007 年 Penryn 引入，含 blend/extract 指令
    bool has_avx2  = false;  // 2013 年 Haswell 引入，256 位整数 SIMD
    bool has_neon  = false;  // ARM 平台的 SIMD（ARMv8 始终可用）
};

CpuFeatures detect_cpu() {
    CpuFeatures f;

#if defined(__ARM_NEON__) || defined(__ARM_NEON)
    // ── ARM 平台 ──
    // ARMv8（AArch64）上 NEON 是强制的，编译期就能确定
    // ARMv7 上需要检查 /proc/cpuinfo 或 getauxval(AT_HWCAP)
    f.has_neon = true;

#elif defined(__x86_64__) || defined(_M_X64) || defined(__i386__) || defined(_M_IX86)
    // ── x86 平台：使用 CPUID 指令 ──
    int info[4] = {0};  // EAX, EBX, ECX, EDX

    // CPUID leaf 1：基本特性位
    #ifdef _MSC_VER
        __cpuid(info, 1);  // MSVC: __cpuid(输出数组, 功能号)
    #else
        __cpuid(1, info[0], info[1], info[2], info[3]);  // GCC: 参数顺序不同
    #endif

    // 检查各特性位（参考 Intel 手册 Vol.2 CPUID 指令说明）
    f.has_sse2  = (info[3] & (1 << 26)) != 0;  // EDX bit 26 = SSE2
    f.has_ssse3 = (info[2] & (1 <<  9)) != 0;  // ECX bit 9  = SSSE3
    f.has_sse41 = (info[2] & (1 << 19)) != 0;  // ECX bit 19 = SSE4.1

    // CPUID leaf 7：扩展特性位（AVX2 在这里）
    #ifdef _MSC_VER
        __cpuidex(info, 7, 0);  // MSVC: __cpuidex(输出, 功能号, 子功能号)
    #else
        __cpuid_count(7, 0, info[0], info[1], info[2], info[3]);
    #endif

    f.has_avx2 = (info[1] & (1 << 5)) != 0;    // EBX bit 5 = AVX2
#endif

    return f;
}
```

> **CPUID leaf**（CPUID 叶子/功能号）：CPUID 指令根据 EAX 寄存器的输入值（功能号）返回不同信息。`leaf 1` 返回基本特性标志，`leaf 7` 返回扩展特性。每个特性对应返回值中的特定位——例如 SSE2 对应 EDX 寄存器的第 26 位。

### 5.3 ARM 平台的检测策略

ARM 平台的情况比 x86 简单得多：

| ARM 版本 | NEON 可用性 | 检测方式 |
|----------|-----------|---------|
| ARMv7（32 位） | **可选**，需检测 | Linux: `getauxval(AT_HWCAP)` 检查 `HWCAP_NEON` 位；Android: `android_getCpuFeatures()` |
| ARMv8/AArch64（64 位） | **始终可用** | 编译期宏 `__ARM_NEON` 或 `__aarch64__` 即可确认 |

KrKr2 的 Android 版目标是 ARM64（`arm64-v8a`），所以 NEON 始终可用，不需要运行时检测——编译期条件编译就够了。但 x86 特性（SSE2/AVX2）仍需运行时检测，因为同一 PC 版本可能运行在不同代际的 CPU 上。

---

## 6. 函数指针分发模式

### 6.1 模式概述

"运行时检测 + 函数指针分发"是 KrKr2 处理跨平台 SIMD 的核心架构模式。思路很简单：

1. **定义统一的函数签名**（typedef 函数指针类型）
2. **声明全局函数指针变量**（初始为 `nullptr`）
3. **为每种指令集提供独立的实现**（纯 C / SSE2 / AVX2 / NEON）
4. **程序启动时**，检测 CPU 特性，把函数指针指向最优实现
5. **业务代码统一通过函数指针调用**——无需关心底层用了哪种指令集

```cpp
// ── 步骤 1：定义函数指针类型 ──
typedef void (*BlendFunc)(uint32_t* dst, const uint32_t* src, int count);

// ── 步骤 2：声明全局函数指针（初始为空） ──
BlendFunc g_alpha_blend = nullptr;
BlendFunc g_add_blend   = nullptr;
BlendFunc g_sub_blend   = nullptr;

// ── 步骤 3：各版本实现（分别在不同 .cpp 文件中） ──
// blend_scalar.cpp
void alpha_blend_c(uint32_t* dst, const uint32_t* src, int count) { /* 纯 C */ }
void add_blend_c(uint32_t* dst, const uint32_t* src, int count)   { /* 纯 C */ }

// blend_sse2.cpp（仅 x86 编译）
void alpha_blend_sse2(uint32_t* dst, const uint32_t* src, int count) { /* SSE2 */ }
void add_blend_sse2(uint32_t* dst, const uint32_t* src, int count)   { /* SSE2 */ }

// blend_neon.cpp（仅 ARM 编译）
void alpha_blend_neon(uint32_t* dst, const uint32_t* src, int count) { /* NEON */ }
void add_blend_neon(uint32_t* dst, const uint32_t* src, int count)   { /* NEON */ }

// ── 步骤 4：启动时初始化 ──
void init_blend_functions() {
    CpuFeatures cpu = detect_cpu();

    if (cpu.has_avx2) {
        g_alpha_blend = alpha_blend_avx2;
        g_add_blend   = add_blend_avx2;
    } else if (cpu.has_sse2) {
        g_alpha_blend = alpha_blend_sse2;
        g_add_blend   = add_blend_sse2;
    } else if (cpu.has_neon) {
        g_alpha_blend = alpha_blend_neon;
        g_add_blend   = add_blend_neon;
    } else {
        g_alpha_blend = alpha_blend_c;    // 回退到纯 C
        g_add_blend   = add_blend_c;
    }
}

// ── 步骤 5：业务代码统一调用 ──
void render_layer(uint32_t* canvas, const uint32_t* layer,
                  int width, int height, int blend_mode) {
    // 选择混合函数
    BlendFunc blend = (blend_mode == BLEND_ADD) ? g_add_blend : g_alpha_blend;

    // 逐行混合
    for (int y = 0; y < height; ++y) {
        blend(canvas + y * width, layer + y * width, width);
    }
}
```

> **为什么用函数指针而不用 `if` 判断？** 函数指针只在初始化时做一次判断，之后每次调用都是直接的间接跳转（indirect call）——没有分支预测开销。如果在每行混合时都 `if (has_sse2)` 判断，那 1920×1080 的图像就要做 1080 次无意义的条件判断。

### 6.2 KrKr2 源码中的分发架构

KrKr2 在 `tvpgl.h` 中使用宏来批量声明函数指针。以下是简化的对照：

```cpp
// ── tvpgl.h 中的宏（项目源码简化） ──
// TVP_GL_FUNC_PTR_EXTERN_DECL 宏生成 extern 函数指针声明
#define TVP_GL_FUNC_PTR_EXTERN_DECL(rettype, funcname, arg) \
    extern rettype (*funcname) arg;

// 使用宏声明大量函数指针
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAlphaBlend,
    (tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len))
// 宏展开后等价于:
// extern void (*TVPAlphaBlend)(tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len);

TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAdditiveAlphaBlend,
    (tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len))

TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPCopyColor,
    (tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len))

// ... 30+ 个类似声明，覆盖所有混合模式
```

初始化流程在 `tvpgl.cpp` / `tvpgl_asm_init.h` 中：

```
┌──────────────────────────────────────────────────────────┐
│              程序启动                                     │
│  main() → TVPAppDelegate::run() → ...                    │
└───────────────────┬──────────────────────────────────────┘
                    ▼
┌──────────────────────────────────────────────────────────┐
│         TVPGL_Init() / TVPGL_ASM_Init()                  │
│  1. 调用 detect_cpu() 检测 CPU 特性                       │
│  2. 根据检测结果设置所有函数指针                            │
└───────────────────┬──────────────────────────────────────┘
                    │
        ┌───────────┼───────────┬────────────┐
        ▼           ▼           ▼            ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
   │  纯 C   │ │  SSE2   │ │  AVX2   │ │  NEON   │
   │ 回退版  │ │ x86基线 │ │ x86高性能│ │ ARM移动 │
   └─────────┘ └─────────┘ └─────────┘ └─────────┘
```

相关源码文件：

| 文件 | 作用 |
|------|------|
| `cpp/core/visual/tvpgl.h` | 函数指针声明（`TVP_GL_FUNC_PTR_EXTERN_DECL` 宏） |
| `cpp/core/visual/tvpgl.cpp` | 函数指针定义 + `TVPGL_Init()` 初始化 |
| `cpp/core/visual/tvpgl_asm_init.h` | `TVPGL_ASM_Init()` —— SIMD 版本注册 |
| `cpp/core/visual/blend_functor_c.h` | 30+ 混合模式的纯 C 仿函数（Functor，一种可以像函数一样调用的对象/结构体） |
| `cpp/core/visual/blend_function.cpp` | 混合函数的实际调用入口 |
| `cpp/core/visual/DetectCPU.cpp` | CPU 特性检测实现 |

---

## 7. 跨平台 SIMD 条件编译

### 7.1 预处理器架构隔离

在实际项目中，x86 的 SIMD 头文件和 ARM 的 NEON 头文件**不能同时包含**——它们定义了不同架构的类型和函数。必须用预处理器将它们隔离：

```cpp
// ── simd_config.h —— 跨平台 SIMD 配置 ──
#pragma once

// 检测 CPU 架构（不是操作系统！）
#if defined(__x86_64__) || defined(_M_X64) || defined(__i386__) || defined(_M_IX86)
    // x86 / x64 架构
    #include <emmintrin.h>    // SSE2（x86-64 始终可用）
    #ifdef __SSSE3__
        #include <tmmintrin.h>    // SSSE3（含 _mm_shuffle_epi8）
    #endif
    #ifdef __AVX2__
        #include <immintrin.h>    // AVX2（256 位整数 SIMD）
    #endif
    #define SIMD_HAS_SSE2  1
    // AVX2 需要运行时检测，不在此定义

#elif defined(__aarch64__) || defined(__ARM_NEON__) || defined(__ARM_NEON)
    // ARM 架构（Android arm64-v8a、iOS、Apple Silicon Mac）
    #include <arm_neon.h>
    #define SIMD_HAS_NEON  1

#else
    // 其他架构（RISC-V、MIPS 等）：仅使用纯 C
    #define SIMD_SCALAR_ONLY 1
#endif
```

### 7.2 Apple Silicon 陷阱（常见错误 #1）

这是最容易犯的跨平台错误——**按操作系统判断 SIMD 可用性**：

```cpp
// ❌ 错误！macOS 不等于 x86
#ifdef __APPLE__
    #include <emmintrin.h>  // Apple Silicon (M1/M2/M3) 上编译失败！
#endif

// ❌ 错误！Windows 也可能是 ARM（Windows on ARM 设备）
#ifdef _WIN32
    #include <emmintrin.h>  // Surface Pro X (ARM) 上编译失败！
#endif

// ✅ 正确：按 CPU 架构判断
#if defined(__x86_64__) || defined(_M_X64)
    #include <emmintrin.h>  // 只有真正的 x86-64 CPU 才包含
#elif defined(__aarch64__)
    #include <arm_neon.h>   // Apple M1/M2/M3 走这个分支
#endif
```

> **原因**：Apple 在 2020 年将 Mac 从 Intel（x86）切换到自研 M 系列芯片（ARM 架构）。同样的 `__APPLE__` 宏在 Intel Mac 和 Apple Silicon Mac 上都为真，但 SSE2 只在 Intel Mac 上可用。Windows 也有类似情况——微软推出了基于 ARM 芯片的 Surface 设备。所以**永远按 CPU 架构判断，不按操作系统**。

### 7.3 编译器标志要求

即使代码中包含了正确的头文件，编译器也需要对应的标志来启用 SIMD 指令生成：

```bash
# ── x86-64 编译器标志 ──
# SSE2 在 x86-64 模式下默认启用，但 32 位模式需要手动开启
g++ -msse2 -o blend blend.cpp          # 启用 SSE2
g++ -mssse3 -o blend blend.cpp         # 启用到 SSSE3（含 SSE2）
g++ -mavx2 -o blend blend.cpp          # 启用到 AVX2（含所有更早的指令集）

# ── ARM 编译器标志 ──
# AArch64 (ARM64) 模式下 NEON 默认启用，通常不需要额外标志
# ARMv7 (32 位 ARM) 需要：
g++ -mfpu=neon -o blend blend.cpp      # 启用 NEON（32 位 ARM）

# ── CMake 中的跨平台写法 ──
target_compile_options(my_blend_lib PRIVATE
    $<$<AND:$<CXX_COMPILER_ID:GNU,Clang>,$<NOT:$<STREQUAL:${CMAKE_SYSTEM_PROCESSOR},aarch64>>>:
        -msse2
        $<$<BOOL:${ENABLE_AVX2}>:-mavx2>
    >
)
```

> **常见错误 #2**：忘记编译器标志导致"函数未声明"。如果你写了 AVX2 代码但编译时没加 `-mavx2`，编译器会报 `_mm256_loadu_si256 was not declared in this scope`——这不是代码错误，而是编译器在没有 AVX2 标志时不会处理 `<immintrin.h>` 中的 AVX2 部分。

---

## 8. 完整跨平台混合框架

下面是一个把前面所有知识整合在一起的完整示例——支持 x86（SSE2）和 ARM（NEON），带运行时分发：

```cpp
// ── cross_blend.h ──
#pragma once
#include <cstdint>

// 函数指针类型
typedef void (*BlendFunc)(uint32_t* dst, const uint32_t* src, int count);

// 全局函数指针
extern BlendFunc g_alpha_blend;

// 初始化（必须在第一次使用前调用）
void init_simd_dispatch();
```

```cpp
// ── cross_blend.cpp ──
#include "cross_blend.h"
#include <cstdio>

// ── 1. 纯 C 回退版本（所有平台都能编译） ──
static void alpha_blend_c(uint32_t* dst, const uint32_t* src, int count) {
    for (int i = 0; i < count; ++i) {
        uint32_t s = src[i], d = dst[i];
        uint32_t sa = (s >> 24) & 0xFF;
        if (sa == 0) continue;           // 完全透明，跳过
        if (sa == 255) { dst[i] = s; continue; }  // 完全不透明，直接覆盖
        uint32_t inv = 255 - sa;
        // 分离 RB 和 G 通道并行计算（参见第 2 章）
        uint32_t rb = (((s & 0xFF00FF) * sa + (d & 0xFF00FF) * inv + 0x800080) >> 8) & 0xFF00FF;
        uint32_t g  = (((s & 0x00FF00) * sa + (d & 0x00FF00) * inv + 0x008000) >> 8) & 0x00FF00;
        dst[i] = rb | g | (sa << 24);
    }
}

// ── 2. SSE2 版本（仅 x86 编译） ──
#if defined(__x86_64__) || defined(_M_X64) || defined(__i386__) || defined(_M_IX86)
#include <emmintrin.h>

static void alpha_blend_sse2(uint32_t* dst, const uint32_t* src, int count) {
    __m128i zero = _mm_setzero_si128();
    // ... SSE2 实现（详见上一节）
    // 这里省略完整实现以避免重复，实际项目中放完整代码
    // 尾部回退到 alpha_blend_c
    int simd_end = count & ~3;
    for (int i = simd_end; i < count; ++i) {
        // 标量处理尾部
    }
}
#endif

// ── 3. NEON 版本（仅 ARM 编译） ──
#if defined(__aarch64__) || defined(__ARM_NEON__) || defined(__ARM_NEON)
#include <arm_neon.h>

static void alpha_blend_neon_impl(uint32_t* dst, const uint32_t* src, int count) {
    static const uint8_t shuf[16] = {3,3,3,3, 7,7,7,7, 11,11,11,11, 15,15,15,15};
    uint8x16_t shuffle = vld1q_u8(shuf);
    uint16x8_t v255 = vdupq_n_u16(255);
    uint16x8_t v128 = vdupq_n_u16(128);

    int simd_end = count & ~3;
    for (int i = 0; i < simd_end; i += 4) {
        uint8x16_t s = vld1q_u8((const uint8_t*)(src + i));
        uint8x16_t d = vld1q_u8((const uint8_t*)(dst + i));
        uint8x16_t ab = vqtbl1q_u8(s, shuffle);

        uint16x8_t s_lo = vmovl_u8(vget_low_u8(s));
        uint16x8_t s_hi = vmovl_u8(vget_high_u8(s));
        uint16x8_t d_lo = vmovl_u8(vget_low_u8(d));
        uint16x8_t d_hi = vmovl_u8(vget_high_u8(d));
        uint16x8_t a_lo = vmovl_u8(vget_low_u8(ab));
        uint16x8_t a_hi = vmovl_u8(vget_high_u8(ab));

        uint16x8_t inv_lo = vsubq_u16(v255, a_lo);
        uint16x8_t inv_hi = vsubq_u16(v255, a_hi);

        uint16x8_t r_lo = vshrq_n_u16(vaddq_u16(
            vaddq_u16(vmulq_u16(s_lo, a_lo), vmulq_u16(d_lo, inv_lo)), v128), 8);
        uint16x8_t r_hi = vshrq_n_u16(vaddq_u16(
            vaddq_u16(vmulq_u16(s_hi, a_hi), vmulq_u16(d_hi, inv_hi)), v128), 8);

        vst1q_u8((uint8_t*)(dst + i),
                 vcombine_u8(vqmovn_u16(r_lo), vqmovn_u16(r_hi)));
    }
    // 尾部标量
    for (int i = simd_end; i < count; ++i) {
        uint32_t s = src[i], d = dst[i];
        uint32_t sa = (s >> 24) & 0xFF, inv = 255 - sa;
        uint32_t rb = (((s&0xFF00FF)*sa + (d&0xFF00FF)*inv + 0x800080)>>8) & 0xFF00FF;
        uint32_t g  = (((s&0x00FF00)*sa + (d&0x00FF00)*inv + 0x008000)>>8) & 0x00FF00;
        dst[i] = rb | g | (sa << 24);
    }
}
#endif

// ── 4. 全局函数指针定义 ──
BlendFunc g_alpha_blend = nullptr;

// ── 5. 分发初始化 ──
void init_simd_dispatch() {
#if defined(__aarch64__) || defined(__ARM_NEON__) || defined(__ARM_NEON)
    // ARM64：直接使用 NEON（始终可用）
    g_alpha_blend = alpha_blend_neon_impl;
    printf("[SIMD] 使用 ARM NEON Alpha 混合\n");

#elif defined(__x86_64__) || defined(_M_X64) || defined(__i386__) || defined(_M_IX86)
    // x86：运行时检测
    CpuFeatures cpu = detect_cpu();
    if (cpu.has_sse2) {
        g_alpha_blend = alpha_blend_sse2;
        printf("[SIMD] 使用 SSE2 Alpha 混合\n");
    } else {
        g_alpha_blend = alpha_blend_c;
        printf("[SIMD] 使用纯 C Alpha 混合（回退）\n");
    }

#else
    // 未知架构：纯 C
    g_alpha_blend = alpha_blend_c;
    printf("[SIMD] 使用纯 C Alpha 混合（未知架构）\n");
#endif
}
```

```cpp
// ── main.cpp —— 使用示例 ──
#include "cross_blend.h"
#include <cstdio>
#include <vector>

int main() {
    // 必须在使用前初始化分发
    init_simd_dispatch();

    // 创建测试数据
    const int W = 1920, H = 1080;
    std::vector<uint32_t> canvas(W * H, 0xFFCCBBAA);  // 不透明背景
    std::vector<uint32_t> layer(W * H, 0x80FF8040);   // 半透明前景

    // 通过函数指针调用——自动使用最优实现
    for (int y = 0; y < H; ++y) {
        g_alpha_blend(canvas.data() + y * W,
                      layer.data() + y * W, W);
    }

    printf("混合完成：%d x %d\n", W, H);
    return 0;
}
```

---

## 动手实践

### 练习 1：NEON 饱和加法

在 ARM 设备或模拟器上（也可以用 QEMU），实现一个 NEON 版的饱和加法函数：

```cpp
// 目标：使用 vqaddq_u8 实现 4 像素饱和加法
void add_blend_neon(uint32_t* dst, const uint32_t* src, int count) {
    int simd_end = count & ~3;
    for (int i = 0; i < simd_end; i += 4) {
        // 1. 用 vld1q_u8 加载 src 和 dst
        uint8x16_t s = vld1q_u8((const uint8_t*)(src + i));
        uint8x16_t d = vld1q_u8((const uint8_t*)(dst + i));

        // 2. 用 vqaddq_u8 执行饱和加法（一条指令！）
        uint8x16_t result = vqaddq_u8(s, d);

        // 3. 用 vst1q_u8 存储结果
        vst1q_u8((uint8_t*)(dst + i), result);
    }
    // 标量尾部
    for (int i = simd_end; i < count; ++i) {
        uint8_t* dp = (uint8_t*)(dst + i);
        const uint8_t* sp = (const uint8_t*)(src + i);
        for (int c = 0; c < 4; ++c) {
            int sum = dp[c] + sp[c];
            dp[c] = (sum > 255) ? 255 : sum;
        }
    }
}
```

### 练习 2：扩展分发框架

在上面的 `init_simd_dispatch()` 函数基础上，添加对 `g_add_blend`（加法混合）和 `g_sub_blend`（减法混合，使用 `_mm_subs_epu8` / `vqsubq_u8`）的分发支持。确保三个函数指针都在启动时被正确初始化。

---

## 对照项目源码

KrKr2 的函数指针分发架构分布在以下文件中：

| 文件路径 | 作用 | 关键点 |
|---------|------|--------|
| `cpp/core/visual/tvpgl.h` | 用 `TVP_GL_FUNC_PTR_EXTERN_DECL` 宏声明 30+ 个函数指针 | 统一函数签名，所有混合/拷贝/转换操作 |
| `cpp/core/visual/tvpgl.cpp` | 定义函数指针变量，实现 `TVPGL_Init()` | 初始化入口，设置默认（纯 C）实现 |
| `cpp/core/visual/tvpgl_asm_init.h` | `TVPGL_ASM_Init()` 注册 SIMD 版本 | 按 CPU 特性覆盖默认实现 |
| `cpp/core/visual/blend_functor_c.h` | 30+ 混合模式的纯 C 仿函数 | 宏展开架构，每个仿函数实现一种混合算法 |
| `cpp/core/visual/blend_function.cpp` | 业务层调用入口 | 通过函数指针间接调用 |
| `cpp/core/visual/DetectCPU.cpp` | CPU 特性检测 | 目前主要活跃代码是 ARM NEON（Apple 平台） |
| `cpp/core/visual/ResampleImage.cpp` | 图像缩放的 SIMD 版本 | 有 AVX2/SSE2 声明 |

> **学习建议**：打开 `tvpgl.h`，搜索 `TVP_GL_FUNC_PTR_EXTERN_DECL`，数一数有多少个函数指针声明——这就是 KrKr2 为 SIMD 加速预留的全部"插槽"。然后打开 `tvpgl_asm_init.h` 看哪些已经有了 SIMD 实现，哪些还只是纯 C 回退。

---

## 常见错误及解决方案

### 错误 1：按操作系统而非 CPU 架构判断 SIMD

```cpp
// ❌ macOS 从 2020 年开始有 ARM 芯片了
#ifdef __APPLE__
    #include <emmintrin.h>
#endif
```

**解决**：始终用 `__x86_64__` / `_M_X64` / `__aarch64__` 等 CPU 架构宏。

### 错误 2：忘记编译器标志

```bash
# ❌ 编译报错：_mm_loadu_si128 was not declared
g++ -o blend blend.cpp

# ✅ 必须启用对应指令集
g++ -msse2 -o blend blend.cpp         # SSE2
g++ -mavx2 -o blend blend.cpp         # AVX2
g++ -mfpu=neon -o blend blend.cpp     # NEON（32 位 ARM）
```

### 错误 3：ARM64 上多余的运行时 NEON 检测

```cpp
// ❌ ARMv8 上 NEON 始终可用，运行时检测是浪费
if (detect_neon()) {
    g_blend = blend_neon;
}

// ✅ 直接在编译期确定
#if defined(__aarch64__)
    g_blend = blend_neon;  // ARMv8 = NEON 始终可用
#endif
```

### 错误 4：函数指针未初始化就调用

```cpp
// ❌ 如果忘记调用 init_simd_dispatch()，g_alpha_blend 为 nullptr
g_alpha_blend(dst, src, count);  // 段错误（Segmentation Fault）！

// ✅ 防御性编程：提供默认实现
BlendFunc g_alpha_blend = alpha_blend_c;  // 默认纯 C，不会为空
// 或者在调用前检查
if (g_alpha_blend) {
    g_alpha_blend(dst, src, count);
}
```

---

## 本节小结

| 主题 | 要点 |
|------|------|
| NEON 定位 | ARM 平台的 128 位 SIMD，等价于 x86 的 SSE2 |
| 数据类型 | 按元素类型命名（`uint8x16_t`），比 SSE2 的 `__m128i` 更清晰 |
| 半宽寄存器 | `vget_low/high` + `vcombine` 实现 64/128 位切换 |
| Alpha 广播 | 基础版用 `vsetq_lane`（慢），优化版用 `vqtbl1q_u8`（快） |
| CPU 检测 | x86 用 CPUID 指令运行时检测；ARM64 的 NEON 编译期确定 |
| 函数指针分发 | 定义类型 → 声明指针 → 多版本实现 → 启动时初始化 → 统一调用 |
| 条件编译 | 按 CPU 架构（`__x86_64__`/`__aarch64__`）判断，**绝不按操作系统** |
| KrKr2 架构 | `tvpgl.h` 用宏声明 30+ 函数指针，`TVPGL_ASM_Init()` 注册 SIMD 版本 |

---

## 练习题与答案

### 题目 1：NEON 类型推断

以下 NEON 代码的输出是什么？

```cpp
uint8x16_t a = vdupq_n_u8(200);   // 全部通道 = 200
uint8x16_t b = vdupq_n_u8(100);   // 全部通道 = 100

uint8x16_t sum_wrap = vaddq_u8(a, b);     // 回绕加法
uint8x16_t sum_sat  = vqaddq_u8(a, b);    // 饱和加法

// sum_wrap 的每个通道值 = ?
// sum_sat 的每个通道值 = ?
```

<details>
<summary>查看答案</summary>

```
sum_wrap: 200 + 100 = 300，但 uint8 最大 255，回绕后 = 300 - 256 = 44
sum_sat:  200 + 100 = 300，饱和钳制为 255

所以：
- sum_wrap 每个通道 = 44
- sum_sat  每个通道 = 255
```

回绕加法 `vaddq_u8` 等价于 `(a + b) % 256`，结果可能"绕回"到小数值——这在像素处理中会导致颜色突变（亮色变暗）。饱和加法 `vqaddq_u8` 保证结果不超过 255，适合加法混合（Additive Blend）等场景。

</details>

### 题目 2：修复跨平台编译错误

以下代码在 Apple M2 MacBook 上编译失败，找出错误并修复：

```cpp
#ifdef __APPLE__
    #include <emmintrin.h>

    void process(uint32_t* data, int n) {
        for (int i = 0; i < n; i += 4) {
            __m128i v = _mm_loadu_si128((__m128i*)(data + i));
            v = _mm_add_epi32(v, _mm_set1_epi32(1));
            _mm_storeu_si128((__m128i*)(data + i), v);
        }
    }
#endif
```

<details>
<summary>查看答案</summary>

**错误原因**：Apple M2 是 ARM 芯片（AArch64），没有 SSE2 指令集。`#ifdef __APPLE__` 在 M2 上为真，但 `<emmintrin.h>` 和 SSE2 intrinsic 只在 x86 上存在。

**修复方案**：按 CPU 架构分支，同时提供 NEON 实现：

```cpp
#if defined(__x86_64__) || defined(_M_X64)
    #include <emmintrin.h>

    void process(uint32_t* data, int n) {
        for (int i = 0; i < n; i += 4) {
            __m128i v = _mm_loadu_si128((__m128i*)(data + i));
            v = _mm_add_epi32(v, _mm_set1_epi32(1));
            _mm_storeu_si128((__m128i*)(data + i), v);
        }
    }

#elif defined(__aarch64__)
    #include <arm_neon.h>

    void process(uint32_t* data, int n) {
        uint32x4_t one = vdupq_n_u32(1);
        for (int i = 0; i < n; i += 4) {
            uint32x4_t v = vld1q_u32(data + i);
            v = vaddq_u32(v, one);
            vst1q_u32(data + i, v);
        }
    }

#else
    // 纯 C 回退
    void process(uint32_t* data, int n) {
        for (int i = 0; i < n; ++i) {
            data[i] += 1;
        }
    }
#endif
```

**核心教训**：`__APPLE__` 只表示"这是 Apple 的操作系统"，不代表 CPU 架构。从 2020 年 Apple Silicon 开始，Mac 可以是 x86 也可以是 ARM。

</details>

### 题目 3：设计函数指针分发

为以下三种操作设计完整的函数指针分发框架：
- `pixel_copy`：像素拷贝（`memcpy` 的 SIMD 版本）
- `pixel_fill`：像素填充（用固定颜色填满缓冲区）
- `pixel_invert`：像素反色（每个通道 `c = 255 - c`）

要求：纯 C 回退 + SSE2 + NEON 三种实现，启动时自动选择。

<details>
<summary>查看答案</summary>

```cpp
#include <cstdint>
#include <cstring>
#include <cstdio>

// ── 1. 函数指针类型 ──
typedef void (*PixelCopyFunc)(uint32_t* dst, const uint32_t* src, int count);
typedef void (*PixelFillFunc)(uint32_t* dst, uint32_t color, int count);
typedef void (*PixelInvertFunc)(uint32_t* data, int count);

// ── 2. 全局函数指针 ──
PixelCopyFunc   g_pixel_copy   = nullptr;
PixelFillFunc   g_pixel_fill   = nullptr;
PixelInvertFunc g_pixel_invert = nullptr;

// ── 3. 纯 C 实现 ──
void pixel_copy_c(uint32_t* dst, const uint32_t* src, int count) {
    memcpy(dst, src, count * sizeof(uint32_t));
}
void pixel_fill_c(uint32_t* dst, uint32_t color, int count) {
    for (int i = 0; i < count; ++i) dst[i] = color;
}
void pixel_invert_c(uint32_t* data, int count) {
    for (int i = 0; i < count; ++i) {
        data[i] ^= 0x00FFFFFF;  // 反转 RGB，保留 Alpha
    }
}

// ── 4. SSE2 实现（x86） ──
#if defined(__x86_64__) || defined(_M_X64)
#include <emmintrin.h>
void pixel_invert_sse2(uint32_t* data, int count) {
    __m128i mask = _mm_set1_epi32(0x00FFFFFF);
    int simd_end = count & ~3;
    for (int i = 0; i < simd_end; i += 4) {
        __m128i v = _mm_loadu_si128((__m128i*)(data + i));
        v = _mm_xor_si128(v, mask);  // XOR 反转 RGB
        _mm_storeu_si128((__m128i*)(data + i), v);
    }
    for (int i = simd_end; i < count; ++i)
        data[i] ^= 0x00FFFFFF;
}
#endif

// ── 5. NEON 实现（ARM） ──
#if defined(__aarch64__)
#include <arm_neon.h>
void pixel_invert_neon(uint32_t* data, int count) {
    uint32x4_t mask = vdupq_n_u32(0x00FFFFFF);
    int simd_end = count & ~3;
    for (int i = 0; i < simd_end; i += 4) {
        uint32x4_t v = vld1q_u32(data + i);
        v = veorq_u32(v, mask);  // XOR 反转 RGB
        vst1q_u32(data + i, v);
    }
    for (int i = simd_end; i < count; ++i)
        data[i] ^= 0x00FFFFFF;
}
#endif

// ── 6. 分发初始化 ──
void init_pixel_ops() {
    // copy 和 fill 的 SIMD 版本收益小（memcpy 已经很快），用 C 版本即可
    g_pixel_copy = pixel_copy_c;
    g_pixel_fill = pixel_fill_c;

#if defined(__aarch64__)
    g_pixel_invert = pixel_invert_neon;
    printf("[Dispatch] pixel_invert → NEON\n");
#elif defined(__x86_64__) || defined(_M_X64)
    g_pixel_invert = pixel_invert_sse2;
    printf("[Dispatch] pixel_invert → SSE2\n");
#else
    g_pixel_invert = pixel_invert_c;
    printf("[Dispatch] pixel_invert → Scalar C\n");
#endif
}

int main() {
    init_pixel_ops();

    uint32_t pixels[8] = {
        0xFF112233, 0xFF445566, 0xFF778899, 0xFFAABBCC,
        0x80DDEEFF, 0x80001122, 0x80334455, 0x80667788
    };

    g_pixel_invert(pixels, 8);

    for (int i = 0; i < 8; ++i)
        printf("pixels[%d] = 0x%08X\n", i, pixels[i]);

    return 0;
}
```

**预期输出**（以 SSE2 为例）：
```
[Dispatch] pixel_invert → SSE2
pixels[0] = 0xFFEEDDCC    // 0xFF112233 ^ 0x00FFFFFF
pixels[1] = 0xFFBBAA99
pixels[2] = 0xFF887766
pixels[3] = 0xFF554433
pixels[4] = 0x80221100    // Alpha 0x80 保持不变
pixels[5] = 0x80FFEEDD
pixels[6] = 0x80CCBBAA
pixels[7] = 0x80998877
```

</details>

---

## 下一步

[饱和加法与实践](./04-饱和加法与实践.md) —— 学习 KrKr2 中 `TVPSaturatedAdd` 的标量位操作技巧、SSE2/NEON 一条指令实现饱和运算，以及 KrKr2 SIMD 文件架构全景图。
