# 为什么像素处理需要 SIMD

> **所属模块：** P05-软件渲染原理 · 第 4 章 · 第 1 节  
> **前置知识：** [像素本质与格式](../01-像素与颜色空间/01-像素本质与格式.md)、[Alpha 混合](../02-Alpha混合与合成/01-Alpha通道与标准混合.md)、C++ 基础、位运算  
> **预计阅读时间：** 30 分钟

---

## 本节目标

读完本节后，你将能够：

1. 理解标量像素处理的性能瓶颈所在
2. 掌握 SIMD（Single Instruction Multiple Data，单指令多数据）的核心思想
3. 了解 x86 平台 SIMD 指令集从 MMX 到 AVX-512 的完整演进历程
4. 理解 SIMD 寄存器宽度与像素处理吞吐量的关系
5. 知道 KrKr2 项目选择 SSE2 作为基线的硬件依据
6. 了解 ARM NEON 在移动平台的地位

---

## 1. 标量处理的性能瓶颈

### 1.1 一个真实的性能问题

在前面章节中，我们实现了逐像素的 Alpha 混合。回顾一下标量版本：

```cpp
// 标量 Alpha 混合 —— 每次只处理 1 个像素
void alpha_blend_scalar(uint32_t* dst, const uint32_t* src, int count) {
    for (int i = 0; i < count; ++i) {
        uint32_t s = src[i];  // 源像素（ARGB 格式，参见第 1 章）
        uint32_t d = dst[i];  // 目标像素

        // 提取源像素的 Alpha 通道（最高 8 位）
        uint32_t sa = (s >> 24) & 0xFF;
        uint32_t inv_sa = 255 - sa;  // 反转 Alpha

        // 分别提取 R、G、B 三个颜色通道
        uint32_t sr = (s >> 16) & 0xFF;
        uint32_t sg = (s >> 8)  & 0xFF;
        uint32_t sb =  s        & 0xFF;

        uint32_t dr = (d >> 16) & 0xFF;
        uint32_t dg = (d >> 8)  & 0xFF;
        uint32_t db =  d        & 0xFF;

        // Alpha 混合公式：result = src * sa + dst * (255 - sa)
        // 然后除以 255 归一化回 [0, 255] 范围
        uint32_t or_ = (sr * sa + dr * inv_sa) / 255;
        uint32_t og  = (sg * sa + dg * inv_sa) / 255;
        uint32_t ob  = (sb * sa + db * inv_sa) / 255;

        // 重新组装为 ARGB 像素，Alpha 设为不透明
        dst[i] = (0xFFu << 24) | (or_ << 16) | (og << 8) | ob;
    }
}
```

### 1.2 数一数运算量

对于一张 1920×1080 的画面，需要处理 **207 万个像素**。每个像素需要：

| 操作类型 | 每像素次数 | 说明 |
|----------|-----------|------|
| 移位（`>>`） | 6 次 | 提取 A/R/G/B 通道 |
| 掩码（`& 0xFF`） | 6 次 | 确保值在 [0,255] |
| 乘法 | 6 次 | 3 通道 × 2 项（src×α + dst×(1-α)） |
| 加法 | 3 次 | 每通道合并两项 |
| 除法 | 3 次 | ÷255 归一化 |
| 减法 | 1 次 | 计算 255 - sa |
| 组装移位 | 3 次 | 重新拼装为 ARGB |

总计每像素约 **28 条运算**。整帧共需 ~5800 万次运算。在 60 FPS（每秒 60 帧）下，每帧只有 **16.7 毫秒**，而且 CPU 还有大量其他工作（脚本解析、事件处理、音频解码等）。

### 1.3 瓶颈分析

标量处理的核心问题是**吞吐量不足**：

```
标量循环：
  第 1 次迭代 → 处理像素 #0 的 R 通道
  第 2 次迭代 → 处理像素 #0 的 G 通道
  第 3 次迭代 → 处理像素 #0 的 B 通道
  ...反复 207 万次...
```

CPU 的 ALU（算术逻辑单元，Arithmetic Logic Unit——执行加减乘除等运算的硬件电路）一次只操作 32 位或 64 位数据，但实际上我们的 R/G/B 通道每个只有 8 位。**32 位寄存器中有 24 位是浪费的！**

```
32 位寄存器处理 1 个 8 位通道值：

  [00000000 00000000 00000000 RRRRRRRR]
   ↑ 浪费     ↑ 浪费     ↑ 浪费    ↑ 实际使用
         75% 的硬件能力被闲置
```

这就是 SIMD 要解决的问题。

---

## 2. SIMD 的核心思想

### 2.1 什么是 SIMD

SIMD 是 **Single Instruction, Multiple Data** 的缩写，中文译为"单指令多数据"。它是现代 CPU 的**向量指令集扩展**（Vector Instruction Set Extension），允许一条 CPU 指令同时对多个数据执行相同的运算。

对比标量（Scalar，一次处理一个数据的常规模式）与 SIMD：

```
标量模式：一条指令处理 1 个数据
  ADD  r1, r2  →  1 个结果

SIMD 模式：一条指令同时处理 N 个数据
  PADDB xmm1, xmm2  →  16 个字节同时相加！
```

> **PADDB** 是 x86 SSE2 指令集中的一条指令。P = Packed（打包），ADD = 加法，B = Byte（字节）。它将两个 128 位寄存器中的 16 个字节逐一相加。

### 2.2 SIMD 与像素处理天然契合

一个 ARGB 像素恰好是 4 个字节（32 位）。不同宽度的 SIMD 寄存器可以装下不同数量的像素：

| 寄存器宽度 | 指令集 | 可容纳像素数 | 平台 |
|-----------|--------|-------------|------|
| 64 位 | MMX（已淘汰） | 2 个 | x86 |
| 128 位 | SSE2 / NEON | **4 个** | x86 / ARM |
| 256 位 | AVX2 | **8 个** | x86（2013+） |
| 512 位 | AVX-512 | 16 个 | x86 服务器 |

这意味着使用 128 位 SSE2 寄存器，理论上可以获得 **4 倍加速**；使用 256 位 AVX2 则可以获得 **8 倍加速**。

```
SSE2 寄存器（128 位 = 16 字节 = 4 个 ARGB 像素）：

  [A3 R3 G3 B3 | A2 R2 G2 B2 | A1 R1 G1 B1 | A0 R0 G0 B0]
   ←── 像素 3 ──→ ←── 像素 2 ──→ ←── 像素 1 ──→ ←── 像素 0 ──→

一条 _mm_adds_epu8 指令：16 个字节同时做饱和加法！
```

### 2.3 实际加速效果

下面是 Alpha 混合在不同实现方式下的典型性能数据（1920×1080 图像，100 次迭代平均）：

| 实现方式 | 像素/秒（百万） | 相对加速比 | 说明 |
|----------|-----------------|------------|------|
| 纯 C 标量 | ~120 M | 1.0× | 基线，逐像素处理 |
| SSE2（128-bit） | ~450 M | 3.7× | 4 像素并行，所有 x86-64 支持 |
| AVX2（256-bit） | ~800 M | 6.7× | 8 像素并行，2013+ CPU |
| ARM NEON（128-bit） | ~400 M | 3.3× | 4 像素并行，所有 ARMv8 支持 |

> **为什么实际加速比达不到理论的 4× / 8×？** 因为 SIMD 处理需要额外的数据打包/解包操作，还受内存带宽（Memory Bandwidth，CPU 与内存之间的数据传输速度上限）和缓存命中率（Cache Hit Rate，数据恰好在 CPU 高速缓存中的概率）限制。

---

## 3. x86 SIMD 指令集演进

x86 平台的 SIMD 指令集经历了 20 多年的演进。了解这段历史有助于理解为什么 KrKr2 选择 SSE2 作为基线：

```
MMX (1997)     →  64 位寄存器（mm0-mm7）
                  8× int8 或 4× int16，但与 x87 浮点共用寄存器
                  致命缺陷：用 MMX 时不能做浮点运算，已被淘汰

SSE (1999)     →  128 位寄存器（xmm0-xmm7）
                  4× float32（单精度浮点）
                  首次引入独立的 XMM 寄存器组，解决了 MMX 的冲突问题

SSE2 (2001)    →  128 位整数支持 ★ 像素处理的最低基线
                  16× int8, 8× int16, 4× int32, 2× int64
                  ★ 所有 x86-64（64 位 x86）CPU 都必须支持 SSE2
                  这是 KrKr2 项目的 SIMD 基线选择

SSE3 (2004)    →  新增水平加法 HADDPS
                  （将寄存器内相邻元素相加，而非两个寄存器对应元素相加）

SSSE3 (2006)   →  字节重排 PSHUFB ★ 像素通道重排利器
                  可以用一条指令完成任意字节的重排列

SSE4.1 (2008)  →  条件混合 BLENDVPS ★ 像素条件选择
                  按掩码选择两个寄存器中的元素

SSE4.2 (2008)  →  字符串搜索指令（与像素处理关系不大）

AVX (2011)     →  256 位浮点向量（ymm0-ymm15）

AVX2 (2013)    →  256 位整数支持 ★ 像素处理的进阶选择
                  32× int8, 16× int16, 8× int32
                  注意"lane"陷阱（后续章节详讲）

AVX-512 (2016) →  512 位（zmm0-zmm31）
                  主要用于服务器和科学计算，桌面 CPU 支持有限
```

### 3.1 为什么 KrKr2 选择 SSE2 而不是更新的指令集

**关键原因：SSE2 是 x86-64 架构的强制要求。**

当 AMD 在 2003 年发布 AMD64（x86-64 的原始设计）时，将 SSE2 列为必须支持的指令集。这意味着：

- 任何能运行 64 位 Windows/Linux 的 x86 CPU 都支持 SSE2
- 不需要运行时检测，可以无条件使用
- KrKr2 的目标平台（64 位 Windows/Linux）100% 覆盖

对于 AVX2，由于 2013 年之前的 CPU 不支持，KrKr2 将其作为**可选加速路径**——运行时检测 CPU 是否支持，支持则启用，不支持则回退到 SSE2。

### 3.2 ARM 平台的 SIMD：NEON

ARM 处理器使用 **NEON**（也叫 Advanced SIMD）作为其向量指令集。NEON 同样提供 128 位寄存器：

| 特性 | SSE2（x86） | NEON（ARM） |
|------|------------|------------|
| 寄存器宽度 | 128 位 | 128 位 |
| 寄存器数量 | 16 个（xmm0-xmm15） | 32 个（v0-v31，ARMv8） |
| 整数支持 | int8/16/32/64 | int8/16/32/64 |
| 浮点支持 | float32/64 | float32/64 |
| 必然可用 | 所有 x86-64 CPU | 所有 ARMv8（AArch64）CPU |
| 头文件 | `<emmintrin.h>` | `<arm_neon.h>` |

**KrKr2 的跨平台策略**：x86 用 SSE2/AVX2，ARM（Android/macOS Apple Silicon）用 NEON，两者通过条件编译（`#if defined(__x86_64__)` / `#elif defined(__aarch64__)`）隔离。

---

## 4. SIMD 编程的基本概念

在开始写 SIMD 代码之前，需要理解几个关键概念：

### 4.1 Intrinsic 函数

SIMD 代码通常不直接写汇编，而是使用 **intrinsic 函数**（编译器内建函数）。intrinsic 函数看起来像普通的 C/C++ 函数调用，但编译器会将它们直接翻译为对应的 SIMD 汇编指令：

```cpp
#include <emmintrin.h>  // SSE2 的 intrinsic 头文件

// 这个"函数调用"会被编译器直接翻译为 PADDB 汇编指令
__m128i result = _mm_add_epi8(a, b);

// 命名规则：_mm_<操作>_<数据类型>
// _mm    → 128 位 SSE 操作（256 位用 _mm256，512 位用 _mm512）
// add    → 加法操作
// epi8   → 有符号 8 位整数（epu8 = 无符号 8 位，epi16 = 有符号 16 位...）
```

### 4.2 数据类型

SSE2 提供三种 128 位数据类型：

```cpp
__m128i  reg_int;    // 128 位整数向量（像素处理最常用）
                     // 可以解释为 16×int8、8×int16、4×int32 等
__m128   reg_float;  // 128 位单精度浮点向量（4× float）
__m128d  reg_double; // 128 位双精度浮点向量（2× double）
```

### 4.3 对齐要求

SIMD 指令通常要求数据在内存中**对齐**（Aligned）到寄存器宽度的边界：

```cpp
// 128 位 SSE 需要 16 字节对齐
// 意思是：指针地址必须是 16 的整数倍

// 对齐加载（要求地址 % 16 == 0）—— 速度最快
__m128i data = _mm_load_si128((__m128i*)aligned_ptr);

// 非对齐加载（任意地址）—— 稍慢但安全
__m128i data = _mm_loadu_si128((__m128i*)any_ptr);
// 名字中的 'u' 代表 unaligned（未对齐）
```

> **实际建议**：现代 CPU（Haswell 2013+）上对齐与非对齐加载的性能差距已经很小。在不确定对齐情况时，优先使用 `_mm_loadu_si128` 保证安全性。

### 4.4 饱和运算 vs 回绕运算

普通整数加法会**回绕**（Wrap Around）：200 + 100 = 300，但 8 位无符号整数只能存 0-255，所以结果变成 300 - 256 = 44。这对像素处理是灾难性的——白色(255) + 任何值都会回绕成暗色。

**饱和运算**（Saturated Arithmetic）会在溢出时截断到最大值：200 + 100 = 255（而不是 44）。

```cpp
// 回绕加法（普通）—— 像素处理中通常不想要这个
__m128i sum_wrap = _mm_add_epi8(a, b);     // 200+100=44 ❌

// 饱和加法 —— 像素处理的正确选择
__m128i sum_sat = _mm_adds_epu8(a, b);     // 200+100=255 ✅
// 函数名中 'adds' = add saturated, 'epu8' = unsigned packed 8-bit

// 饱和减法
__m128i diff_sat = _mm_subs_epu8(a, b);    // 50-100=0 ✅（而非 206）
```

---

## 动手实践

### 实践 1：测量标量 Alpha 混合的实际耗时

创建一个简单的基准测试，感受标量处理的性能极限：

```cpp
// 文件：benchmark_scalar.cpp
#include <chrono>
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <vector>

// 标量 Alpha 混合（与上面相同）
static void alpha_blend_scalar(uint32_t* dst,
                               const uint32_t* src,
                               int count) {
    for (int i = 0; i < count; ++i) {
        uint32_t s = src[i];
        uint32_t d = dst[i];
        uint32_t sa = (s >> 24) & 0xFF;
        uint32_t inv_sa = 255 - sa;

        uint32_t sr = (s >> 16) & 0xFF;
        uint32_t sg = (s >> 8)  & 0xFF;
        uint32_t sb =  s        & 0xFF;
        uint32_t dr = (d >> 16) & 0xFF;
        uint32_t dg = (d >> 8)  & 0xFF;
        uint32_t db =  d        & 0xFF;

        uint32_t or_ = (sr * sa + dr * inv_sa) / 255;
        uint32_t og  = (sg * sa + dg * inv_sa) / 255;
        uint32_t ob  = (sb * sa + db * inv_sa) / 255;

        dst[i] = (0xFFu << 24) | (or_ << 16) | (og << 8) | ob;
    }
}

int main() {
    const int W = 1920, H = 1080;
    const int PIXELS = W * H;       // 207 万像素
    const int ITERATIONS = 100;

    // 分配源和目标缓冲区
    std::vector<uint32_t> src(PIXELS);
    std::vector<uint32_t> dst(PIXELS);

    // 填充测试数据：半透明红色叠加到白色背景
    for (int i = 0; i < PIXELS; ++i) {
        src[i] = 0x80FF0000u;  // Alpha=128, R=255（半透明红色）
        dst[i] = 0xFFFFFFFFu;  // 不透明白色背景
    }

    // 预热一次（让 CPU 缓存就绪）
    alpha_blend_scalar(dst.data(), src.data(), PIXELS);

    // 正式测量
    auto t0 = std::chrono::high_resolution_clock::now();
    for (int r = 0; r < ITERATIONS; ++r) {
        alpha_blend_scalar(dst.data(), src.data(), PIXELS);
    }
    auto t1 = std::chrono::high_resolution_clock::now();

    double ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
    double mpixels_sec = (double)PIXELS * ITERATIONS / (ms / 1000.0) / 1e6;

    std::printf("标量 Alpha 混合:\n");
    std::printf("  %d 次 × %d 像素 = %.2f ms\n", ITERATIONS, PIXELS, ms);
    std::printf("  吞吐量: %.1f M 像素/秒\n", mpixels_sec);
    std::printf("  每帧(1080p): %.2f ms\n", ms / ITERATIONS);

    // 验证结果正确性：半透明红叠白 → 应该是浅红
    uint32_t sample = dst[0];
    std::printf("  验证: 结果像素 = 0x%08X\n", sample);
    std::printf("    R=%d G=%d B=%d (期望: R≈255, G≈128, B≈128)\n",
                (sample >> 16) & 0xFF,
                (sample >> 8) & 0xFF,
                sample & 0xFF);

    return 0;
}
```

对应的 CMakeLists.txt：

```cmake
cmake_minimum_required(VERSION 3.20)
project(benchmark_scalar LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)

add_executable(benchmark_scalar benchmark_scalar.cpp)

if(MSVC)
    target_compile_options(benchmark_scalar PRIVATE /O2)
else()
    target_compile_options(benchmark_scalar PRIVATE -O2)
endif()
```

### 实践 2：观察 SIMD 饱和加法的效果

```cpp
// 文件：saturation_demo.cpp
// 演示回绕加法与饱和加法的区别
#include <cstdint>
#include <cstdio>
#include <emmintrin.h>  // SSE2

int main() {
    // 准备两组像素数据（4 个像素，每像素 ARGB）
    // 像素值较大，普通加法会溢出
    alignas(16) uint32_t a[4] = {
        0xFFC08060u, 0xFF906050u, 0xFFE0D0C0u, 0xFF707070u
    };
    alignas(16) uint32_t b[4] = {
        0xFF604020u, 0xFF806040u, 0xFF403020u, 0xFFA0A0A0u
    };
    alignas(16) uint32_t result_wrap[4];
    alignas(16) uint32_t result_sat[4];

    // 加载到 SSE2 寄存器
    __m128i va = _mm_load_si128((__m128i*)a);
    __m128i vb = _mm_load_si128((__m128i*)b);

    // 普通加法（回绕）
    __m128i wrap = _mm_add_epi8(va, vb);
    _mm_store_si128((__m128i*)result_wrap, wrap);

    // 饱和加法（截断到 255）
    __m128i sat = _mm_adds_epu8(va, vb);
    _mm_store_si128((__m128i*)result_sat, sat);

    // 打印对比
    std::printf("像素   |  A 值  |  R 值            |  G 值            |  B 值\n");
    std::printf("-------+--------+------------------+------------------+------------------\n");
    for (int i = 0; i < 4; ++i) {
        uint32_t av = (a[i] >> 16) & 0xFF;
        uint32_t bv = (b[i] >> 16) & 0xFF;
        uint32_t wv = (result_wrap[i] >> 16) & 0xFF;
        uint32_t sv = (result_sat[i] >> 16) & 0xFF;
        std::printf("像素 %d | a=%3d  | 回绕: %d+%d=%3d | 饱和: %d+%d=%3d |\n",
                    i, (a[i] >> 24) & 0xFF,
                    av, bv, wv,
                    av, bv, sv);
    }

    return 0;
}
```

运行后你会看到：回绕加法中 `0xC0 + 0x60 = 0x20`（192+96=32），而饱和加法中 `0xC0 + 0x60 = 0xFF`（192+96=255）。对于像素叠加效果（如 KrKr2 的 `TVPSaturatedAdd`），饱和加法才是正确行为。

---

## 对照项目源码

### KrKr2 中的 SIMD 使用概览

KrKr2 项目大量使用 SIMD 优化像素处理。以下是关键文件：

```
krkr2/cpp/core/visual/
├── tvpgl.h              ← 函数指针声明（TVP_GL_FUNC_PTR_EXTERN_DECL 宏）
│                          所有像素操作函数都通过函数指针间接调用
├── tvpgl.cpp            ← 查找表初始化 + 标量版（纯 C）实现
│                          TVPDivTable（÷255 查找表）、TVPOpacityOnOpacityTable 等
├── tvpgl_asm_init.h     ← SIMD 分发入口声明
│                          TVPGL_ASM_Init() 在启动时选择最优实现
├── gl/
│   ├── blend_functor_c.h   ← 30+ 混合模式的纯 C 仿函数（functor）
│   │                         每个仿函数对应一种 Photoshop 风格混合模式
│   ├── blend_function.cpp  ← 宏展开生成各混合函数的入口
│   └── ResampleImage.cpp   ← 图像缩放的 AVX2/SSE2 加速声明
└── ../environ/
    └── DetectCPU.cpp       ← CPU 特性检测（CPUID 指令封装）
```

**设计决策**：KrKr2 使用函数指针而非虚函数来实现运行时分发。函数指针的调用开销等同于普通函数调用（一次间接跳转），而虚函数调用需要先查虚表（vtable）再跳转，多一次内存访问。对于每秒调用数百万次的像素操作，这个差异是可测量的。

---

## 常见错误及解决方案

### 错误 1：在 32 位模式下假设 SSE2 可用

**现象**：代码在 64 位系统上正常运行，编译为 32 位后在老旧 CPU 上崩溃。

**原因**：SSE2 只在 x86-64 上是强制的。32 位 x86 程序可能运行在不支持 SSE2 的古老 CPU（如 Pentium III）上。

**解决**：

```cpp
// ❌ 32 位模式下直接使用 SSE2
void process(uint32_t* data, int n) {
    __m128i v = _mm_loadu_si128((__m128i*)data);  // 老 CPU 崩溃
}

// ✅ 运行时检测或编译期限定架构
#if defined(__x86_64__) || defined(_M_X64)
    // 64 位模式：SSE2 始终可用，直接使用
    void process_sse2(uint32_t* data, int n) { /* ... */ }
#elif defined(__i386__) || defined(_M_IX86)
    // 32 位模式：需要运行时检测
    void process(uint32_t* data, int n) {
        if (cpu_has_sse2()) process_sse2(data, n);
        else                process_scalar(data, n);
    }
#endif
```

### 错误 2：混淆 SIMD 指令集与平台/操作系统

**现象**：在 Apple Silicon Mac 上编译含 `<emmintrin.h>` 的代码失败。

**原因**：开发者错误地按操作系统判断 SIMD 支持，而不是按 CPU 架构。Apple Silicon（M1/M2/M3）是 ARM 架构，使用 NEON 而非 SSE2。

**解决**：

```cpp
// ❌ 错误：按操作系统判断
#ifdef __APPLE__
    #include <emmintrin.h>  // M1 Mac 上编译失败！
#endif

// ✅ 正确：按 CPU 架构判断
#if defined(__x86_64__) || defined(_M_X64) || defined(__i386__) || defined(_M_IX86)
    #include <emmintrin.h>  // x86 CPU 用 SSE2
#elif defined(__aarch64__) || defined(__ARM_NEON__) || defined(__ARM_NEON)
    #include <arm_neon.h>   // ARM CPU（含 Apple Silicon）用 NEON
#endif
```

### 错误 3：对 SIMD 加速比有不切实际的期望

**现象**：SSE2 理论应该 4 倍加速，但实测只有 2-3 倍。

**原因**：

1. SIMD 需要额外的打包/解包操作（8 位 ↔ 16 位转换）
2. 内存带宽可能成为瓶颈（1080p 图像 = 8MB 数据需要读写）
3. 尾部像素仍需标量处理
4. 分支预测（Branch Prediction，CPU 猜测分支走向的机制）对标量代码有优化效果

**解决**：不追求理论倍数，而是用基准测试验证实际收益。只要加速比 > 2×，在 207 万像素的规模下就是值得的。

---

## 本节小结

| 概念 | 要点 |
|------|------|
| 标量瓶颈 | 32 位寄存器处理 8 位通道值，75% 硬件能力被浪费 |
| SIMD 原理 | 一条指令同时操作多个数据，像素处理天然适合 |
| SSE2 | 128 位，4 像素并行，所有 x86-64 CPU 强制支持 |
| AVX2 | 256 位，8 像素并行，2013+ CPU 可选加速 |
| NEON | ARM 的 128 位 SIMD，所有 ARMv8(AArch64) CPU 支持 |
| Intrinsic | 编译器内建函数，像 C 函数一样调用但直接翻译为 SIMD 指令 |
| 饱和运算 | 溢出截断到最大值而非回绕，像素处理必备 |
| KrKr2 策略 | SSE2 基线 + AVX2 可选 + NEON 移动端，函数指针分发 |

---

## 练习题与答案

### 题目 1：SIMD 寄存器容量计算

**问题**：一个 256 位的 AVX2 寄存器（`__m256i`）可以同时存放多少个 ARGB 像素？如果要处理 1920×1080 的图像，AVX2 主循环需要迭代多少次？

<details>
<summary>查看答案</summary>

256 位 = 32 字节。一个 ARGB 像素 = 4 字节。所以可以存放 **32 / 4 = 8 个像素**。

1920 × 1080 = 2,073,600 像素。每次处理 8 个：2,073,600 / 8 = **259,200 次迭代**。

因为 1920 是 8 的整数倍（1920 / 8 = 240），如果按行处理，每行 240 次迭代，没有尾部像素。

</details>

### 题目 2：为什么 KrKr2 不直接用 AVX-512

**问题**：AVX-512 可以一次处理 16 个像素，理论加速比更高。为什么 KrKr2 不以 AVX-512 为基线？

<details>
<summary>查看答案</summary>

1. **覆盖率不足**：AVX-512 主要出现在服务器 CPU（Xeon）和部分高端桌面 CPU。大量用户的 CPU（AMD Ryzen 7000 以下、Intel 第 11 代以下笔记本）不支持。以 AVX-512 为基线会排除大部分用户。

2. **降频问题**：许多 CPU 在执行 AVX-512 指令时会降低核心频率（Intel 称为"License-based downclocking"），可能反而导致整体性能下降。

3. **维护成本**：每增加一个指令集版本就需要多一套实现和测试。SSE2 基线 + AVX2 可选已经覆盖了绝大多数场景。

4. **KrKr2 是游戏引擎**：面向普通用户的桌面 PC 和移动设备，不是面向数据中心。

</details>

### 题目 3：判断 SIMD 适用场景

**问题**：以下哪些场景适合用 SIMD 加速？哪些不适合？说明理由。

1. 对图像每个像素加上固定亮度偏移
2. 解析 JSON 配置文件
3. 计算两张图的像素差值（逐像素相减取绝对值）
4. 遍历链表查找特定节点

<details>
<summary>查看答案</summary>

**适合 SIMD：**

1. ✅ **加亮度偏移**：每个像素做相同的加法运算，数据连续存储，完美匹配 `_mm_adds_epu8` 饱和加法。
3. ✅ **像素差值**：逐像素减法 + 取绝对值，数据连续、操作统一。SSE2 的 `_mm_subs_epu8` + `_mm_max_epu8` 可以高效实现。

**不适合 SIMD：**

2. ❌ **解析 JSON**：JSON 解析涉及大量分支判断（是字符串？数字？花括号？）和不规则的数据访问模式。每个字符的处理逻辑可能不同，无法批量执行相同操作。
4. ❌ **链表遍历**：链表节点在内存中不连续（随机分布），SIMD 需要连续内存。而且每个节点的"下一步"取决于当前节点的 `next` 指针，无法并行。

**判断原则**：SIMD 适合"对连续内存中的大量数据做相同运算"的场景。

</details>

---

## 下一步

下一节 [SSE2 基础与像素并行](./02-SSE2基础与像素并行.md) 将深入 SSE2 的具体指令，手把手实现 4 像素并行 Alpha 混合，并讲解 KrKr2 使用的 ÷255 近似技巧。
