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

