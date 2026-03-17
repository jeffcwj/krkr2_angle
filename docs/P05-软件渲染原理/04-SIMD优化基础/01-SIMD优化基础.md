# SIMD 优化基础

> **所属模块**：P05-软件渲染原理 · 第 4 节  
> **前置知识**：C++ 基础、像素内存布局（ARGB `tjs_uint32`）、Alpha 混合公式、位运算  
> **预计阅读时间**：50 分钟

---

## 本节目标

1. 理解 SIMD（Single Instruction Multiple Data）的核心思想与硬件基础
2. 掌握 x86 SSE2 intrinsic 的基本使用：加载、存储、算术、位操作
3. 学会用 SSE2 实现 4 像素并行 Alpha 混合
4. 了解 AVX2 的 256 位扩展与 ARM NEON 的跨平台对比
5. 理解 KrKr2 项目中的函数指针分发模式（运行时 CPU 检测 + 代码路径选择）
6. 能编写跨平台条件编译的 SIMD 代码

---

## 1. 为什么像素处理需要 SIMD

### 1.1 标量处理的瓶颈

在前几节中，我们实现了逐像素的 Alpha 混合：

```cpp
// 标量 Alpha 混合 —— 每次只处理 1 个像素
void alpha_blend_scalar(uint32_t* dst, const uint32_t* src, int count) {
    for (int i = 0; i < count; ++i) {
        uint32_t s = src[i];
        uint32_t d = dst[i];
        // 提取源像素 Alpha
        uint32_t sa = (s >> 24) & 0xFF;
        uint32_t inv_sa = 255 - sa;  // 反转 Alpha

        // 分别计算 R、G、B 通道
        uint32_t sr = (s >> 16) & 0xFF;
        uint32_t sg = (s >> 8)  & 0xFF;
        uint32_t sb =  s        & 0xFF;

        uint32_t dr = (d >> 16) & 0xFF;
        uint32_t dg = (d >> 8)  & 0xFF;
        uint32_t db =  d        & 0xFF;

        // result = src * sa + dst * (255 - sa)，再除以 255
        uint32_t or_ = (sr * sa + dr * inv_sa) / 255;
        uint32_t og  = (sg * sa + dg * inv_sa) / 255;
        uint32_t ob  = (sb * sa + db * inv_sa) / 255;

        dst[i] = (0xFF << 24) | (or_ << 16) | (og << 8) | ob;
    }
}
```

对于一张 1920×1080 的画面，需要处理 **207 万个像素**。每个像素要进行 3 次乘法、3 次加法、3 次除法，加上大量的移位和掩码操作。在 60 FPS 下，每帧只有 16.7ms，标量代码远远不够。

### 1.2 SIMD 的核心思想

SIMD（Single Instruction, Multiple Data）是现代 CPU 的**向量指令集扩展**。核心思想非常简单：

```
标量：一条指令处理 1 个数据
  ADD  r1, r2  →  r1 + r2 = result

SIMD：一条指令同时处理 N 个数据
  PADDB xmm1, xmm2  →  16 个字节同时相加！
```

一个 ARGB 像素恰好是 4 个字节（32 位）。一个 128 位的 SSE 寄存器可以装 **4 个像素**，一个 256 位的 AVX2 寄存器可以装 **8 个像素**。这意味着理论上可以获得 4～8 倍的加速。

### 1.3 SIMD 加速的实际效果

| 实现方式 | 像素/秒（百万） | 相对加速比 |
|----------|-----------------|------------|
| 纯 C 标量 | ~120 M | 1.0× |
| SSE2（128-bit） | ~450 M | 3.7× |
| AVX2（256-bit） | ~800 M | 6.7× |
| ARM NEON（128-bit） | ~400 M | 3.3× |

> **注意**：实际加速比取决于内存带宽、缓存命中率和具体算法。SIMD 并非总是线性扩展。

---

## 2. x86 SIMD 指令集演进

x86 平台的 SIMD 指令集经历了漫长的演进过程：

```
MMX (1997)     →  64 位，8× int8 或 4× int16
                  问题：与 x87 浮点共用寄存器，不能同时用

SSE (1999)     →  128 位，4× float32
                  引入独立的 xmm0-xmm7 寄存器

SSE2 (2001)    →  128 位整数支持 ★ 像素处理的基准线
                  16× int8, 8× int16, 4× int32, 2× int64
                  所有 x86-64 CPU 都支持

SSE3 (2004)    →  水平加法 hadd
SSSE3 (2006)   →  字节重排 pshufb ★ 像素通道重排利器

SSE4.1 (2008)  →  条件混合 blendv ★ 像素条件选择
SSE4.2 (2008)  →  字符串操作（与像素关系不大）

AVX (2011)     →  256 位浮点
AVX2 (2013)    →  256 位整数 ★ 像素处理的进阶选择
                  32× int8, 16× int16, 8× int32

AVX-512 (2016) →  512 位（服务器 CPU，桌面很少见）
```

**对于 KrKr2 这样的项目，SSE2 是保底基线（所有 64 位 x86 CPU 都支持），AVX2 是可选加速路径。**

---

## 3. SSE2 基础操作

### 3.1 数据类型与寄存器

SSE2 提供 128 位寄存器 `xmm0`～`xmm15`，C/C++ 通过 intrinsic 函数访问：

```cpp
#include <emmintrin.h>  // SSE2 头文件

// 核心数据类型
__m128i  reg;  // 128 位整数向量（最常用）
__m128   freg; // 128 位单精度浮点向量
__m128d  dreg; // 128 位双精度浮点向量

// __m128i 可以被解释为：
// 16 × uint8   (像素的单个通道)
// 8  × uint16  (中间计算用，防溢出)
// 4  × uint32  (完整的 ARGB 像素)
// 2  × uint64
```

### 3.2 加载与存储

```cpp
// 从内存加载 128 位数据到寄存器
// 对齐加载（地址必须 16 字节对齐）—— 更快
__m128i pixels = _mm_load_si128((__m128i*)ptr);

// 非对齐加载（任意地址）—— 稍慢但安全
__m128i pixels = _mm_loadu_si128((__m128i*)ptr);

// 存储（对齐/非对齐）
_mm_store_si128((__m128i*)ptr, pixels);   // 对齐
_mm_storeu_si128((__m128i*)ptr, pixels);  // 非对齐

// 全零/全一
__m128i zero = _mm_setzero_si128();          // 全 0
__m128i ones = _mm_set1_epi8((char)0xFF);    // 全 0xFF

// 广播一个 32 位值到 4 个位置
__m128i mask = _mm_set1_epi32(0x00FF00FF);
```

**常见错误 #1：未对齐访问导致段错误**

```cpp
// ❌ 危险：ptr 可能不是 16 字节对齐的
__m128i data = _mm_load_si128((__m128i*)some_ptr);
// 如果 some_ptr % 16 != 0 → SIGSEGV / Access Violation

// ✅ 安全做法：用 _mm_loadu_si128 或确保对齐
__m128i data = _mm_loadu_si128((__m128i*)some_ptr);

// ✅ 更好：分配对齐内存
// Windows:
uint32_t* buf = (uint32_t*)_aligned_malloc(size, 16);
// Linux/macOS:
uint32_t* buf = nullptr;
posix_memalign((void**)&buf, 16, size);
// C++17 跨平台:
uint32_t* buf = static_cast<uint32_t*>(
    std::aligned_alloc(16, size)
);
```

### 3.3 整数算术指令

```cpp
// 按字节（8位）加法 —— 16 个字节同时加
__m128i sum = _mm_add_epi8(a, b);

// 饱和加法（结果不超过 255，不回绕）—— 像素处理必备
__m128i sum = _mm_adds_epu8(a, b);  // unsigned 饱和
// 例：200 + 100 → 255（而非 44）

// 饱和减法
__m128i diff = _mm_subs_epu8(a, b);
// 例：50 - 100 → 0（而非 206）

// 16 位乘法（取高 16 位）—— Alpha 混合的关键
__m128i hi = _mm_mulhi_epu16(a, b);

// 16 位乘法（取低 16 位）
__m128i lo = _mm_mullo_epi16(a, b);

// 右移（逻辑移位，按 16 位元素）
__m128i shifted = _mm_srli_epi16(a, 8);  // 每个 16 位元素右移 8 位

// 按位操作
__m128i result = _mm_and_si128(a, b);    // AND
__m128i result = _mm_or_si128(a, b);     // OR
__m128i result = _mm_xor_si128(a, b);    // XOR
__m128i result = _mm_andnot_si128(a, b); // (~a) AND b
```

### 3.4 像素打包与解包

像素处理中最关键的技巧是 **8 位 ↔ 16 位转换**。乘法运算需要 16 位精度来避免溢出，但像素数据是 8 位的：

```cpp
// 解包：将 8 位通道扩展到 16 位
// 输入: [A3 R3 G3 B3 | A2 R2 G2 B2 | A1 R1 G1 B1 | A0 R0 G0 B0]
//       ←—— 高 64 位 ——→ ←—— 低 64 位 ——→
__m128i zero = _mm_setzero_si128();

// 解包低 64 位：8×uint8 → 8×uint16
__m128i lo16 = _mm_unpacklo_epi8(pixels, zero);
// 结果: [00 A1 | 00 R1 | 00 G1 | 00 B1 | 00 A0 | 00 R0 | 00 G0 | 00 B0]

// 解包高 64 位
__m128i hi16 = _mm_unpackhi_epi8(pixels, zero);
// 结果: [00 A3 | 00 R3 | 00 G3 | 00 B3 | 00 A2 | 00 R2 | 00 G2 | 00 B2]

// 打包：16 位饱和到 8 位
__m128i packed = _mm_packus_epi16(lo16, hi16);
// 结果: 恢复为 16×uint8，超过 255 的截断到 255
```

---

## 4. SSE2 实现 4 像素并行 Alpha 混合

这是本节的核心。我们将标量 Alpha 混合升级为 SSE2 版本：

```
公式：dst[i] = src[i] * src_alpha + dst[i] * (255 - src_alpha)
其中所有通道使用同一个 src_alpha
```

### 4.1 完整实现

```cpp
#include <emmintrin.h>  // SSE2
#include <cstdint>

// SSE2 Alpha 混合：一次处理 4 个像素
// src、dst 指向 ARGB 像素数组，count 为像素数（必须是 4 的倍数）
void alpha_blend_sse2(uint32_t* dst, const uint32_t* src, int count) {
    // 常量掩码
    const __m128i zero = _mm_setzero_si128();
    const __m128i alpha_mask = _mm_set1_epi32(0xFF000000); // Alpha 通道掩码
    const __m128i round_add = _mm_set1_epi16(128);         // 四舍五入常数

    for (int i = 0; i < count; i += 4) {
        // 步骤 1：加载 4 个源像素和 4 个目标像素
        __m128i s = _mm_loadu_si128((__m128i*)(src + i));
        __m128i d = _mm_loadu_si128((__m128i*)(dst + i));

        // 步骤 2：提取每个像素的 Alpha 并广播到所有通道
        // s = [A3 R3 G3 B3 | A2 R2 G2 B2 | A1 R1 G1 B1 | A0 R0 G0 B0]
        // 我们需要把每个像素的 A 值复制到该像素的 R、G、B 位置

        // 右移 24 位得到 Alpha 在最低字节
        __m128i sa32 = _mm_srli_epi32(s, 24);
        // sa32 = [00 00 00 A3 | 00 00 00 A2 | 00 00 00 A1 | 00 00 00 A0]

        // 用打包+解包技巧广播 Alpha（使用乘法分配到每个通道）
        // 先做 16 位解包处理
        __m128i s_lo = _mm_unpacklo_epi8(s, zero);  // 像素 0, 1 的 8×uint16
        __m128i s_hi = _mm_unpackhi_epi8(s, zero);  // 像素 2, 3 的 8×uint16
        __m128i d_lo = _mm_unpacklo_epi8(d, zero);
        __m128i d_hi = _mm_unpackhi_epi8(d, zero);

        // 提取 Alpha 到 16 位并广播
        // 使用 shuffle：将每个像素的 alpha 复制到该像素的所有通道位置
        // 像素 0 的 alpha 在 s_lo 的第 3 个 16 位位置（索引 3）
        // 像素 1 的 alpha 在 s_lo 的第 7 个 16 位位置（索引 7）
        __m128i sa_lo = _mm_shufflelo_epi16(s_lo, _MM_SHUFFLE(3, 3, 3, 3));
        sa_lo = _mm_shufflehi_epi16(sa_lo, _MM_SHUFFLE(3, 3, 3, 3));

        __m128i sa_hi = _mm_shufflelo_epi16(s_hi, _MM_SHUFFLE(3, 3, 3, 3));
        sa_hi = _mm_shufflehi_epi16(sa_hi, _MM_SHUFFLE(3, 3, 3, 3));

        // 步骤 3：计算 inv_alpha = 255 - alpha
        __m128i all_ff = _mm_set1_epi16(255);
        __m128i inv_lo = _mm_sub_epi16(all_ff, sa_lo);
        __m128i inv_hi = _mm_sub_epi16(all_ff, sa_hi);

        // 步骤 4：混合计算
        // result = (src * alpha + dst * inv_alpha + 128) >> 8
        // 使用 16 位乘法避免溢出

        // src * alpha
        __m128i mul_s_lo = _mm_mullo_epi16(s_lo, sa_lo);
        __m128i mul_s_hi = _mm_mullo_epi16(s_hi, sa_hi);

        // dst * inv_alpha
        __m128i mul_d_lo = _mm_mullo_epi16(d_lo, inv_lo);
        __m128i mul_d_hi = _mm_mullo_epi16(d_hi, inv_hi);

        // 相加 + 四舍五入
        __m128i res_lo = _mm_add_epi16(mul_s_lo, mul_d_lo);
        __m128i res_hi = _mm_add_epi16(mul_s_hi, mul_d_hi);
        res_lo = _mm_add_epi16(res_lo, round_add);
        res_hi = _mm_add_epi16(res_hi, round_add);

        // 右移 8 位（÷256 近似 ÷255）
        res_lo = _mm_srli_epi16(res_lo, 8);
        res_hi = _mm_srli_epi16(res_hi, 8);

        // 步骤 5：打包回 8 位并存储
        __m128i result = _mm_packus_epi16(res_lo, res_hi);

        // 强制 Alpha 通道为 0xFF（不透明）
        result = _mm_or_si128(result, alpha_mask);

        _mm_storeu_si128((__m128i*)(dst + i), result);
    }
}
```

### 4.2 KrKr2 的除法优化技巧

上面的代码用 `>> 8`（÷256）近似 `÷ 255`，会有 1/255 的误差。KrKr2 使用更精确的技巧：

```cpp
// KrKr2 的 ÷255 近似公式（来自 tvpgl.h 中的内联函数）：
// x / 255 ≈ (x + (x >> 8) + 1) >> 8
//
// 误差分析：最大误差 < 0.004，对于 8 位颜色来说完全精确
inline uint32_t div255(uint32_t x) {
    return (x + (x >> 8) + 1) >> 8;
}

// 在 SSE2 中实现：
__m128i div255_sse2(__m128i x) {
    __m128i one = _mm_set1_epi16(1);
    __m128i t = _mm_add_epi16(x, _mm_srli_epi16(x, 8));
    return _mm_srli_epi16(_mm_add_epi16(t, one), 8);
}
```

### 4.3 处理尾部像素

实际图像宽度不一定是 4 的倍数，需要处理"尾巴"：

```cpp
void alpha_blend_sse2_safe(uint32_t* dst, const uint32_t* src, int count) {
    // 主循环：4 像素一批
    int simd_count = count & ~3;  // 向下取整到 4 的倍数
    alpha_blend_sse2(dst, src, simd_count);

    // 尾部：用标量处理剩余 0~3 个像素
    for (int i = simd_count; i < count; ++i) {
        alpha_blend_scalar_one(&dst[i], src[i]);
    }
}
```

---

## 5. AVX2 扩展：8 像素并行

AVX2 将寄存器宽度从 128 位扩展到 256 位，理论上再翻倍：

```cpp
#include <immintrin.h>  // AVX2 头文件

void alpha_blend_avx2(uint32_t* dst, const uint32_t* src, int count) {
    const __m256i zero = _mm256_setzero_si256();
    const __m256i round_add = _mm256_set1_epi16(128);
    const __m256i all_ff = _mm256_set1_epi16(255);
    const __m256i alpha_mask = _mm256_set1_epi32(0xFF000000);

    for (int i = 0; i < count; i += 8) {
        // 一次加载 8 个像素（256 位 = 8×32 位）
        __m256i s = _mm256_loadu_si256((__m256i*)(src + i));
        __m256i d = _mm256_loadu_si256((__m256i*)(dst + i));

        // 解包到 16 位（与 SSE2 类似，但用 256 位版本）
        __m256i s_lo = _mm256_unpacklo_epi8(s, zero);
        __m256i s_hi = _mm256_unpackhi_epi8(s, zero);
        __m256i d_lo = _mm256_unpacklo_epi8(d, zero);
        __m256i d_hi = _mm256_unpackhi_epi8(d, zero);

        // 提取并广播 Alpha
        __m256i sa_lo = _mm256_shufflelo_epi16(s_lo, _MM_SHUFFLE(3,3,3,3));
        sa_lo = _mm256_shufflehi_epi16(sa_lo, _MM_SHUFFLE(3,3,3,3));
        __m256i sa_hi = _mm256_shufflelo_epi16(s_hi, _MM_SHUFFLE(3,3,3,3));
        sa_hi = _mm256_shufflehi_epi16(sa_hi, _MM_SHUFFLE(3,3,3,3));

        __m256i inv_lo = _mm256_sub_epi16(all_ff, sa_lo);
        __m256i inv_hi = _mm256_sub_epi16(all_ff, sa_hi);

        // 混合计算
        __m256i res_lo = _mm256_add_epi16(
            _mm256_mullo_epi16(s_lo, sa_lo),
            _mm256_mullo_epi16(d_lo, inv_lo)
        );
        __m256i res_hi = _mm256_add_epi16(
            _mm256_mullo_epi16(s_hi, sa_hi),
            _mm256_mullo_epi16(d_hi, inv_hi)
        );

        res_lo = _mm256_srli_epi16(_mm256_add_epi16(res_lo, round_add), 8);
        res_hi = _mm256_srli_epi16(_mm256_add_epi16(res_hi, round_add), 8);

        __m256i result = _mm256_packus_epi16(res_lo, res_hi);
        result = _mm256_or_si256(result, alpha_mask);

        _mm256_storeu_si256((__m256i*)(dst + i), result);
    }
}
```

**AVX2 的"车道"陷阱**：AVX2 的 256 位寄存器并非简单的 128 位拼接。`_mm256_unpacklo_epi8` 分别在高 128 位和低 128 位的 lane 内操作，跨 lane 操作需要特殊指令（如 `_mm256_permute2x128_si256`）。这是从 SSE2 迁移到 AVX2 时最常见的 bug 来源。

---

## 6. ARM NEON：移动平台的 SIMD

Android 设备使用 ARM 处理器，对应的 SIMD 指令集是 **NEON**。NEON 同样是 128 位：

### 6.1 NEON 基础

```cpp
#include <arm_neon.h>

// NEON 数据类型（与 SSE 对照）
uint8x16_t  v8;   // 16×uint8  ← 相当于 __m128i（解释为字节）
uint16x8_t  v16;  // 8×uint16  ← 相当于 __m128i（解释为短整型）
uint32x4_t  v32;  // 4×uint32  ← 相当于 __m128i（解释为整型）
```

### 6.2 SSE2 与 NEON 指令对照表

| 操作 | SSE2 | ARM NEON |
|------|------|----------|
| 加载 128 位 | `_mm_loadu_si128` | `vld1q_u8` / `vld1q_u32` |
| 存储 128 位 | `_mm_storeu_si128` | `vst1q_u8` / `vst1q_u32` |
| 全零 | `_mm_setzero_si128` | `vdupq_n_u8(0)` |
| 广播标量 | `_mm_set1_epi16(x)` | `vdupq_n_u16(x)` |
| 8 位加法 | `_mm_add_epi8` | `vaddq_u8` |
| 饱和加法 | `_mm_adds_epu8` | `vqaddq_u8` |
| 16 位乘法 | `_mm_mullo_epi16` | `vmulq_u16` |
| 解包低 | `_mm_unpacklo_epi8(a, zero)` | `vmovl_u8(vget_low_u8(a))` |
| 解包高 | `_mm_unpackhi_epi8(a, zero)` | `vmovl_u8(vget_high_u8(a))` |
| 打包 | `_mm_packus_epi16` | `vcombine_u8(vmovn_u16(lo), vmovn_u16(hi))` |
| 右移 16 位 | `_mm_srli_epi16(a, n)` | `vshrq_n_u16(a, n)` |
| 按位与 | `_mm_and_si128` | `vandq_u8` |

### 6.3 NEON Alpha 混合实现

```cpp
#include <arm_neon.h>

void alpha_blend_neon(uint32_t* dst, const uint32_t* src, int count) {
    for (int i = 0; i < count; i += 4) {
        // 加载 4 个像素
        uint8x16_t s = vld1q_u8((const uint8_t*)(src + i));
        uint8x16_t d = vld1q_u8((const uint8_t*)(dst + i));

        // 解包到 16 位
        uint16x8_t s_lo = vmovl_u8(vget_low_u8(s));
        uint16x8_t s_hi = vmovl_u8(vget_high_u8(s));
        uint16x8_t d_lo = vmovl_u8(vget_low_u8(d));
        uint16x8_t d_hi = vmovl_u8(vget_high_u8(d));

        // 提取 Alpha（每个像素的第 4 个字节，即索引 3, 7, 11, 15）
        // NEON 可以直接用 vdupq_lane 进行广播
        uint16_t a0 = vgetq_lane_u16(s_lo, 3);
        uint16_t a1 = vgetq_lane_u16(s_lo, 7);
        uint16_t a2 = vgetq_lane_u16(s_hi, 3);
        uint16_t a3 = vgetq_lane_u16(s_hi, 7);

        // 构造 alpha 向量（简化写法，实际可用 vtbl 查表优化）
        uint16x8_t sa_lo = vdupq_n_u16(0); // 这里简化处理
        sa_lo = vsetq_lane_u16(a0, sa_lo, 0);
        sa_lo = vsetq_lane_u16(a0, sa_lo, 1);
        sa_lo = vsetq_lane_u16(a0, sa_lo, 2);
        sa_lo = vsetq_lane_u16(a0, sa_lo, 3);
        sa_lo = vsetq_lane_u16(a1, sa_lo, 4);
        sa_lo = vsetq_lane_u16(a1, sa_lo, 5);
        sa_lo = vsetq_lane_u16(a1, sa_lo, 6);
        sa_lo = vsetq_lane_u16(a1, sa_lo, 7);

        uint16x8_t inv_lo = vsubq_u16(vdupq_n_u16(255), sa_lo);

        // 混合
        uint16x8_t res_lo = vaddq_u16(
            vmulq_u16(s_lo, sa_lo),
            vmulq_u16(d_lo, inv_lo)
        );
        res_lo = vshrq_n_u16(vaddq_u16(res_lo, vdupq_n_u16(128)), 8);

        // 打包回 8 位
        uint8x8_t r_lo = vmovn_u16(res_lo);
        // ... 高半部分同理 ...

        // 存储结果
        // vst1q_u8((uint8_t*)(dst + i), vcombine_u8(r_lo, r_hi));
    }
}
```

> **提示**：实际项目中 Alpha 广播会使用 `vtbl`（查表指令）或 `vuzp`（解交错）来高效实现，上面的逐元素设置仅为说明原理。

---

## 7. 运行时 CPU 特性检测与函数指针分发

不同用户的 CPU 支持不同的指令集。KrKr2 使用**运行时检测 + 函数指针分发**模式：

### 7.1 CPU 特性检测

```cpp
#ifdef _MSC_VER
  #include <intrin.h>  // MSVC: __cpuid
#else
  #include <cpuid.h>   // GCC/Clang: __get_cpuid
#endif

struct CpuFeatures {
    bool has_sse2  = false;
    bool has_ssse3 = false;
    bool has_sse41 = false;
    bool has_avx2  = false;
    bool has_neon  = false;
};

CpuFeatures detect_cpu() {
    CpuFeatures f;

#if defined(__ARM_NEON__) || defined(__ARM_NEON)
    // ARM 平台：NEON 在 ARMv8 (AArch64) 上始终可用
    f.has_neon = true;
#elif defined(__x86_64__) || defined(_M_X64) || defined(__i386__) || defined(_M_IX86)
    // x86 平台：使用 CPUID 指令检测
    int info[4] = {0};

    #ifdef _MSC_VER
        __cpuid(info, 1);
    #else
        __cpuid(1, info[0], info[1], info[2], info[3]);
    #endif

    f.has_sse2  = (info[3] & (1 << 26)) != 0;  // EDX bit 26
    f.has_ssse3 = (info[2] & (1 <<  9)) != 0;  // ECX bit 9
    f.has_sse41 = (info[2] & (1 << 19)) != 0;  // ECX bit 19

    // AVX2 需要检查 CPUID leaf 7
    #ifdef _MSC_VER
        __cpuidex(info, 7, 0);
    #else
        __cpuid_count(7, 0, info[0], info[1], info[2], info[3]);
    #endif

    f.has_avx2 = (info[1] & (1 << 5)) != 0;    // EBX bit 5
#endif

    return f;
}
```

### 7.2 函数指针分发模式

这是 KrKr2 `tvpgl.h` 中使用的核心设计模式：

```cpp
// 步骤 1：定义函数指针类型
typedef void (*AlphaBlendFunc)(uint32_t* dst, const uint32_t* src, int count);

// 步骤 2：声明全局函数指针
AlphaBlendFunc g_alpha_blend = nullptr;

// 步骤 3：提供多个实现版本
void alpha_blend_c(uint32_t* dst, const uint32_t* src, int count);      // 纯 C
void alpha_blend_sse2(uint32_t* dst, const uint32_t* src, int count);   // SSE2
void alpha_blend_avx2(uint32_t* dst, const uint32_t* src, int count);   // AVX2
void alpha_blend_neon(uint32_t* dst, const uint32_t* src, int count);   // NEON

// 步骤 4：启动时根据 CPU 特性选择最优版本
void init_blend_functions() {
    CpuFeatures cpu = detect_cpu();

    if (cpu.has_avx2) {
        g_alpha_blend = alpha_blend_avx2;
        printf("使用 AVX2 Alpha 混合\n");
    } else if (cpu.has_sse2) {
        g_alpha_blend = alpha_blend_sse2;
        printf("使用 SSE2 Alpha 混合\n");
    } else if (cpu.has_neon) {
        g_alpha_blend = alpha_blend_neon;
        printf("使用 NEON Alpha 混合\n");
    } else {
        g_alpha_blend = alpha_blend_c;
        printf("使用纯 C Alpha 混合（回退）\n");
    }
}

// 步骤 5：调用时统一使用函数指针
void render_layer(uint32_t* dst, const uint32_t* src, int w, int h) {
    for (int y = 0; y < h; ++y) {
        g_alpha_blend(dst + y * w, src + y * w, w);
    }
}
```

### 7.3 对照 KrKr2 源码

KrKr2 的 `tvpgl.h` 大量使用了这个模式。以下是简化的对照：

```cpp
// tvpgl.h 中的宏定义（简化）
#define TVP_GL_FUNC_PTR_EXTERN_DECL(rettype, funcname, arg) \
    extern rettype (*funcname) arg;

// 声明全局函数指针
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAlphaBlend,
    (tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len))
// 展开为: extern void (*TVPAlphaBlend)(tjs_uint32*, const tjs_uint32*, tjs_int);

// tvpgl.cpp 中的初始化函数
// 在 TVPGL_Init() 或 TVPGL_ASM_Init() 中：
//   if (has_avx2) TVPAlphaBlend = TVPAlphaBlend_AVX2;
//   else if (has_sse2) TVPAlphaBlend = TVPAlphaBlend_SSE2;
//   else TVPAlphaBlend = TVPAlphaBlend_C;
```

```
┌──────────────────────────────────────────────┐
│         应用层调用 TVPAlphaBlend()            │
│              （函数指针）                      │
└───────────────┬──────────────────────────────┘
                │ 运行时分发
    ┌───────────┼───────────┬──────────────┐
    ▼           ▼           ▼              ▼
┌────────┐ ┌────────┐ ┌─────────┐ ┌──────────┐
│ 纯 C   │ │ SSE2   │ │  AVX2   │ │  NEON    │
│ 回退版 │ │ 基线版 │ │ 高性能  │ │ ARM 移动 │
└────────┘ └────────┘ └─────────┘ └──────────┘
```

---

## 8. 跨平台 SIMD 条件编译

在实际项目中，需要用预处理器将 x86 和 ARM 代码隔离：

```cpp
#if defined(__x86_64__) || defined(_M_X64) || defined(__i386__) || defined(_M_IX86)
    // x86/x64 平台
    #include <emmintrin.h>  // SSE2
    #ifdef __AVX2__
        #include <immintrin.h>  // AVX2
    #endif
    #define PLATFORM_HAS_SSE2  1
#elif defined(__ARM_NEON__) || defined(__ARM_NEON) || defined(__aarch64__)
    // ARM 平台（Android、iOS、Apple Silicon Mac）
    #include <arm_neon.h>
    #define PLATFORM_HAS_NEON  1
#else
    // 其他平台：仅使用纯 C 实现
    #define PLATFORM_HAS_SCALAR_ONLY 1
#endif

// 统一接口
void alpha_blend_optimized(uint32_t* dst, const uint32_t* src, int count) {
#if PLATFORM_HAS_SSE2
    alpha_blend_sse2(dst, src, count);
#elif PLATFORM_HAS_NEON
    alpha_blend_neon(dst, src, count);
#else
    alpha_blend_scalar(dst, src, count);
#endif
}
```

**常见错误 #2：忘记编译器标志**

```bash
# ❌ 编译报错："_mm_loadu_si128" was not declared
g++ -o blend blend.cpp

# ✅ 必须启用 SSE2（x86-64 默认已开，32 位需要手动开）
g++ -msse2 -o blend blend.cpp

# ✅ 使用 AVX2 需要显式开启
g++ -mavx2 -o blend blend.cpp

# ✅ CMake 中的正确做法
target_compile_options(my_lib PRIVATE
    $<$<AND:$<CXX_COMPILER_ID:GNU,Clang>,$<COMPILE_LANGUAGE:CXX>>:
        -msse2                # SSE2（x86-64 默认有，保险起见加上）
        $<$<BOOL:${USE_AVX2}>:-mavx2>  # AVX2 可选
    >
)
```

**常见错误 #3：未考虑 Apple Silicon**

```cpp
// ❌ macOS 上不一定有 SSE2！Apple Silicon 是 ARM
#ifdef __APPLE__
    #include <emmintrin.h>  // 在 M1/M2 上编译失败
#endif

// ✅ 正确：按 CPU 架构判断，不按操作系统
#if defined(__x86_64__) || defined(_M_X64)
    #include <emmintrin.h>
#elif defined(__aarch64__)
    #include <arm_neon.h>   // Apple M1/M2 用 NEON
#endif
```

---

## 9. 性能测量与基准测试

优化的前提是**可测量**。以下是一个简单的 SIMD 性能基准测试框架：

```cpp
#include <chrono>
#include <cstdio>
#include <cstring>

void benchmark_blend(const char* name,
                     void (*func)(uint32_t*, const uint32_t*, int),
                     int width, int height, int iterations)
{
    // 分配对齐内存
    size_t size = width * height * sizeof(uint32_t);
    uint32_t* src = (uint32_t*)_aligned_malloc(size, 32);
    uint32_t* dst = (uint32_t*)_aligned_malloc(size, 32);

    // 填充测试数据
    for (int i = 0; i < width * height; ++i) {
        src[i] = 0x80FF8040;  // 半透明像素
        dst[i] = 0xFFCCBBAA;  // 不透明背景
    }

    // 预热（避免冷缓存影响结果）
    func(dst, src, width * height);

    // 正式测量
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < iterations; ++i) {
        func(dst, src, width * height);
    }
    auto end = std::chrono::high_resolution_clock::now();

    double ms = std::chrono::duration<double, std::milli>(end - start).count();
    double pixels_per_sec = (double)width * height * iterations / (ms / 1000.0);

    printf("%-15s: %.2f ms (%d 次), %.1f M 像素/秒\n",
           name, ms, iterations, pixels_per_sec / 1e6);

    _aligned_free(src);
    _aligned_free(dst);
}

int main() {
    const int W = 1920, H = 1080, N = 100;

    benchmark_blend("纯 C",     alpha_blend_scalar, W, H, N);
    benchmark_blend("SSE2",     alpha_blend_sse2,   W, H, N);
#ifdef __AVX2__
    benchmark_blend("AVX2",     alpha_blend_avx2,   W, H, N);
#endif
    return 0;
}
```

---

## 10. SSE2 饱和加法实现

KrKr2 中的 `TVPSaturatedAdd` 等函数使用了巧妙的位运算来实现饱和加法。在 SSE2 中可以直接使用硬件饱和指令：

```cpp
// 标量版饱和加法（KrKr2 tvpgl.h 中的位运算技巧）
inline uint32_t saturated_add_scalar(uint32_t a, uint32_t b) {
    // KrKr2 的位运算技巧：不用分支实现饱和
    uint32_t tmp = (((a & b) + (((a ^ b) >> 1) & 0x7F7F7F7F)) & 0x80808080);
    tmp = (tmp << 1) - (tmp >> 7);
    return (a + b - tmp) | tmp;
}

// SSE2 版：一条指令搞定！
__m128i saturated_add_sse2(__m128i a, __m128i b) {
    return _mm_adds_epu8(a, b);  // 16 字节同时饱和加法
}

// 完整的饱和加法混合函数
void blend_add_sse2(uint32_t* dst, const uint32_t* src, int count) {
    for (int i = 0; i < count; i += 4) {
        __m128i s = _mm_loadu_si128((__m128i*)(src + i));
        __m128i d = _mm_loadu_si128((__m128i*)(dst + i));
        __m128i result = _mm_adds_epu8(s, d);  // 饱和加法
        _mm_storeu_si128((__m128i*)(dst + i), result);
    }
}
```

---

## 动手实践

### 实践 1：实现 SSE2 饱和减法混合

```cpp
// 任务：用 _mm_subs_epu8 实现饱和减法混合
// 效果：dst = saturate(dst - src)，常用于"减暗"效果
void blend_subtract_sse2(uint32_t* dst, const uint32_t* src, int count) {
    // 你的代码：
    for (int i = 0; i < count; i += 4) {
        __m128i s = _mm_loadu_si128((__m128i*)(src + i));
        __m128i d = _mm_loadu_si128((__m128i*)(dst + i));
        // 提示：_mm_subs_epu8(d, s) 会将每个字节做 d-s，下溢截断到 0
        __m128i result = _mm_subs_epu8(d, s);
        _mm_storeu_si128((__m128i*)(dst + i), result);
    }
}
```

### 实践 2：用 SSE2 实现 ARGB → 灰度转换

```cpp
// 灰度 = 0.299*R + 0.587*G + 0.114*B
// 整数近似: (77*R + 150*G + 29*B) >> 8
// 任务：用 SSE2 一次转换 4 个像素为灰度
// 提示：需要解包到 16 位、乘以系数、累加、打包回 8 位
```

---

## 对照项目源码

### KrKr2 的 SIMD 使用全景

```
krkr2/cpp/core/visual/
├── tvpgl.h           ← 函数指针声明（TVP_GL_FUNC_PTR_EXTERN_DECL 宏）
├── tvpgl.cpp         ← 查找表初始化 + 位运算优化的标量实现
├── tvpgl_asm_init.h  ← TVPGL_ASM_Init 声明（运行时分发入口）
├── gl/
│   ├── blend_functor_c.h    ← 所有混合模式的纯 C 仿函数实现
│   ├── blend_function.cpp   ← 宏展开生成各混合函数
│   └── ResampleImage.cpp    ← TVPResampleImageAVX2/SSE2 声明
└── ../environ/
    └── DetectCPU.cpp        ← CPU 特性检测（目前大部分被 stub）
```

**关键设计决策**：

1. **tvpgl.h** 用函数指针而非虚函数 —— 零开销抽象，调用开销等同于普通函数调用
2. **查找表 vs SIMD** —— KrKr2 同时使用两种优化策略：查找表用于复杂的 Photoshop 混合模式（如 `TVPPsTableSoftLight`，64KB LUT），SIMD 用于简单的逐像素运算
3. **DetectCPU.cpp 目前大部分被 stub** —— 说明 SIMD 分发仍在演进中，只有 ARM NEON 检测在 Apple 平台上活跃

---

## 本节小结

| 概念 | 要点 |
|------|------|
| SIMD 原理 | 一条指令操作多个数据，像素处理天然适合（4 字节 = 1 像素） |
| SSE2 | 128 位，4 像素并行，所有 x86-64 CPU 都支持，是像素处理的基线 |
| AVX2 | 256 位，8 像素并行，注意 lane 边界问题 |
| NEON | ARM 的 128 位 SIMD，AArch64 上始终可用，语法与 SSE2 不同但概念一致 |
| 像素打包/解包 | 8 位 ↔ 16 位转换是 SIMD 像素处理的核心技巧 |
| ÷255 近似 | `(x + (x >> 8) + 1) >> 8`，KrKr2 使用此公式 |
| 函数指针分发 | 编译期准备多版本，运行时检测 CPU 选择最优路径 |
| 跨平台编译 | 按 CPU 架构（非 OS）条件编译，注意 Apple Silicon |

---

## 练习题与答案

### 练习 1：SSE2 寄存器容量计算

**问题**：一个 `__m128i` 寄存器可以同时存放多少个 ARGB 像素？如果图像宽度为 1920 像素，SSE2 主循环需要迭代多少次？会有多少尾部像素需要标量处理？

<details>
<summary>查看答案</summary>

- 一个 `__m128i` = 128 位 = 16 字节。一个 ARGB 像素 = 4 字节。所以可以存放 **4 个像素**。
- 1920 / 4 = 480 次迭代，**没有尾部像素**（1920 恰好是 4 的倍数）。
- 如果宽度是 1921：主循环 480 次（处理 1920 像素），尾部 **1 个像素**用标量处理。

</details>

### 练习 2：指令选择

**问题**：以下场景分别应该用哪条 SSE2 指令？
1. 将两层图像的像素颜色值相加，超过 255 的截断为 255
2. 将 `__m128i` 中的 16 个字节（uint8）解包为 8 个 uint16，只取低 8 字节
3. 将 8 个 uint16 的计算结果打包回 16 个 uint8，超出 [0,255] 的饱和截断

<details>
<summary>查看答案</summary>

1. **`_mm_adds_epu8(a, b)`** —— 无符号字节饱和加法。`adds` = add saturated, `epu8` = unsigned packed 8-bit。
2. **`_mm_unpacklo_epi8(a, _mm_setzero_si128())`** —— 将 a 的低 8 字节与零交错排列，每个 uint8 前面插入一个 0 字节，变成 uint16。
3. **`_mm_packus_epi16(lo, hi)`** —— 将两个 `__m128i`（各含 8 个 int16）打包为一个 `__m128i`（16 个 uint8），带无符号饱和。`packus` = pack unsigned saturated。

</details>

### 练习 3：跨平台函数指针分发

**问题**：请为以下函数签名设计一个完整的跨平台分发系统（含函数指针声明、3 种实现的占位声明、初始化函数）：

```cpp
void pixel_multiply(uint32_t* dst, const uint32_t* src, int count);
```

<details>
<summary>查看答案</summary>

```cpp
// 1. 函数指针类型与全局变量
typedef void (*PixelMultiplyFunc)(uint32_t* dst, const uint32_t* src, int count);
PixelMultiplyFunc g_pixel_multiply = nullptr;

// 2. 各平台实现声明
void pixel_multiply_c(uint32_t* dst, const uint32_t* src, int count);     // 纯 C 回退
#if defined(__x86_64__) || defined(_M_X64)
void pixel_multiply_sse2(uint32_t* dst, const uint32_t* src, int count);  // SSE2
void pixel_multiply_avx2(uint32_t* dst, const uint32_t* src, int count);  // AVX2
#endif
#if defined(__aarch64__) || defined(__ARM_NEON__)
void pixel_multiply_neon(uint32_t* dst, const uint32_t* src, int count);  // NEON
#endif

// 3. 初始化分发
void init_pixel_multiply() {
    CpuFeatures cpu = detect_cpu();
#if defined(__x86_64__) || defined(_M_X64)
    if (cpu.has_avx2)       g_pixel_multiply = pixel_multiply_avx2;
    else if (cpu.has_sse2)  g_pixel_multiply = pixel_multiply_sse2;
    else                    g_pixel_multiply = pixel_multiply_c;
#elif defined(__aarch64__) || defined(__ARM_NEON__)
    g_pixel_multiply = pixel_multiply_neon;  // AArch64 上 NEON 始终可用
#else
    g_pixel_multiply = pixel_multiply_c;
#endif
}

// 4. 调用点统一使用函数指针
// g_pixel_multiply(dst_buf, src_buf, pixel_count);
```

关键点：
- 按 CPU 架构条件编译，不按操作系统
- 提供纯 C 回退确保所有平台可用
- 初始化函数在程序启动时调用一次
- 调用点无需关心具体使用了哪个实现

</details>

---

## 下一步

下一节 **05-实战：软件光栅化器** 将综合前面所有知识（像素操作、Alpha 混合、图像缩放、SIMD 优化），从零实现一个迷你软件光栅化器，体验 KrKr2 渲染管线的核心逻辑。
