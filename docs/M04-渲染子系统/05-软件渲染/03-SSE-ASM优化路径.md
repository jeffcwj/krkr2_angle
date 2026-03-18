# SSE/ASM 优化路径

> **所属模块：** M04-渲染子系统
> **前置知识：** [tvpgl像素操作库](./01-tvpgl像素操作库.md)、[ARGB混合与合成](./02-ARGB混合与合成.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 KrKr2 的 SIMD 分发架构（函数指针绑定模式）
2. 掌握 CPU 特性检测系统的实现原理（CPUID 指令与平台差异）
3. 了解对齐内存分配器在 SIMD 中的作用
4. 分析原版 KiriKiri2 ASM 优化路径被禁用的原因及重新启用策略

## SIMD 分发架构：函数指针绑定

### 核心思想

KrKr2 的像素操作库 tvpgl 使用**函数指针分发**（Function Pointer Dispatch）来实现运行时 SIMD 优化切换。所有像素操作函数在 `tvpgl.h` 中声明为全局函数指针，而非直接函数：

```cpp
// tvpgl.h — 函数指针声明（非函数原型）
// 每个操作都是一个可在运行时重绑定的指针
extern void (*TVPAlphaBlend)(
    tjs_uint32 *dest,         // 目标像素缓冲区
    const tjs_uint32 *src,    // 源像素缓冲区
    tjs_int len               // 像素数量
);

extern void (*TVPAlphaBlend_o)(
    tjs_uint32 *dest,
    const tjs_uint32 *src,
    tjs_int len,
    tjs_int opa                // 附加不透明度参数
);

extern void (*TVPAlphaBlend_HDA)(
    tjs_uint32 *dest,
    const tjs_uint32 *src,
    tjs_int len                // Hold Dest Alpha 变体
);
```

这种设计的本质是**策略模式**（Strategy Pattern）应用于底层像素操作：函数指针就是策略槽（Strategy Slot），不同的初始化函数将不同实现绑定到这些槽中。调用者无需关心底层使用的是纯 C 实现还是 SSE2/AVX2 加速版本。

### 两阶段初始化

系统提供两个初始化函数，按优先级调用：

```cpp
// blend_function.cpp — 纯 C 实现初始化
void TVPGL_C_Init() {
    // 第一阶段：绑定所有函数指针到纯 C 实现
    // 这是基准实现，保证在所有平台上可用

    // Alpha 混合系列
    TVPAlphaBlend       = TVPAlphaBlend_c;
    TVPAlphaBlend_o     = TVPAlphaBlend_o_c;
    TVPAlphaBlend_HDA   = TVPAlphaBlend_HDA_c;
    TVPAlphaBlend_HDA_o = TVPAlphaBlend_HDA_o_c;
    TVPAlphaBlend_d     = TVPAlphaBlend_d_c;
    TVPAlphaBlend_d_o   = TVPAlphaBlend_d_o_c;

    // 加法混合系列
    TVPAdditiveAlphaBlend     = TVPAdditiveAlphaBlend_c;
    TVPAdditiveAlphaBlend_o   = TVPAdditiveAlphaBlend_o_c;
    TVPAdditiveAlphaBlend_HDA = TVPAdditiveAlphaBlend_HDA_c;

    // 常量 Alpha 混合
    TVPConstAlphaBlend       = TVPConstAlphaBlend_c;
    TVPConstAlphaBlend_SD    = TVPConstAlphaBlend_SD_c;
    TVPConstAlphaBlend_SD_d  = TVPConstAlphaBlend_SD_d_c;
    TVPConstAlphaBlend_SD_a  = TVPConstAlphaBlend_SD_a_c;

    // PS 混合模式
    TVPPsMulBlend     = TVPPsMulBlend_c;
    TVPPsScreenBlend  = TVPPsScreenBlend_c;
    TVPPsOverlayBlend = TVPPsOverlayBlend_c;

    // 转换系列
    TVPBLConvert24BitTo32Bit = TVPBLConvert24BitTo32Bit_c;
    TVPBLConvert32BitTo24Bit = TVPBLConvert32BitTo24Bit_c;

    // Universal Transition 混合
    TVPUnivTransBlend   = TVPUnivTransBlend_c;
    TVPUnivTransBlend_d = TVPUnivTransBlend_d_c;
    TVPUnivTransBlend_a = TVPUnivTransBlend_a_c;

    // ... 共计 100+ 个函数指针绑定
}
```

```cpp
// tvpgl_asm_init.h — ASM/SIMD 优化初始化（声明）
// 第二阶段：用 SIMD 版本覆盖 C 版本
extern void TVPGL_ASM_Init();

// 调用顺序示意（引擎启动时）：
// 1. TVPGL_C_Init();      ← 必须先调用，建立基准
// 2. TVPGL_ASM_Init();    ← 可选，用 SIMD 版覆盖
```

### 函数指针分发的优势

```
┌─────────────────────────────────────────────────┐
│              调用层 (Layer / Transition)          │
│    TVPAlphaBlend(dest, src, len);                │
│    ↓ 通过函数指针间接调用                         │
├─────────────────────────────────────────────────┤
│           函数指针表 (tvpgl.h)                    │
│    TVPAlphaBlend → ???                           │
├─────────┬──────────┬───────────┬────────────────┤
│  C 实现  │ SSE2实现  │ AVX2实现  │  NEON实现       │
│  (基准)  │ (x86加速) │ (x86加速) │  (ARM加速)      │
└─────────┴──────────┴───────────┴────────────────┘
     ↑ TVPGL_C_Init()  ↑ TVPGL_ASM_Init() 选择性覆盖
```

这种架构相比编译期 `#ifdef` 切换的优势：
1. **运行时适配** — 同一个二进制文件可以在不同 CPU 上选择最优路径
2. **渐进优化** — 只需优化热点函数，未优化的自动回退到 C 实现
3. **易于测试** — 可以强制使用 C 实现来验证 SIMD 实现的正确性
4. **热插拔** — 理论上可以在运行时切换实现（用于基准测试对比）

## CPU 特性检测系统

### CPU 特性标志定义

```cpp
// cpu_types.h — CPU 特性位掩码
// 每个标志对应一组 SIMD 指令集
enum {
    TVP_CPU_HAS_MMX   = 0x01,  // MMX（已过时，x86 早期 SIMD）
    TVP_CPU_HAS_SSE   = 0x02,  // SSE（128位浮点 SIMD）
    TVP_CPU_HAS_SSE2  = 0x04,  // SSE2（128位整数 SIMD，像素操作主力）
    TVP_CPU_HAS_SSE3  = 0x08,  // SSE3（水平运算增强）
    TVP_CPU_HAS_SSSE3 = 0x10,  // SSSE3（字节级 shuffle）
    TVP_CPU_HAS_SSE41 = 0x20,  // SSE4.1（blend/extract 指令）
    TVP_CPU_HAS_SSE42 = 0x40,  // SSE4.2（字符串比较指令）
    TVP_CPU_HAS_NEON  = 0x80,  // ARM NEON（128位 ARM SIMD）
    TVP_CPU_HAS_AVX   = 0x100, // AVX（256位浮点 SIMD）
    TVP_CPU_HAS_AVX2  = 0x200, // AVX2（256位整数 SIMD）
};

// 全局变量，存储检测到的 CPU 特性
extern tjs_uint32 TVPCPUType;
```

### Windows 平台：完整 CPUID 检测

Windows 版本提供了最完整的 CPU 特性检测，使用 x86 的 `CPUID` 指令：

```cpp
// win32/DetectCPU.cpp — Windows CPU 检测核心逻辑

#include <intrin.h>   // __cpuid, __cpuidex 内联汇编

static void TVPDetectCPUType() {
    // 使用 CPUID 指令查询 CPU 特性
    int cpuInfo[4];  // EAX, EBX, ECX, EDX

    // CPUID 功能 1：基础特性标志
    __cpuid(cpuInfo, 1);
    int ecx = cpuInfo[2];  // ECX 寄存器包含 SSE3+ 标志
    int edx = cpuInfo[3];  // EDX 寄存器包含 MMX/SSE/SSE2 标志

    // 从 EDX 提取基础特性
    if (edx & (1 << 23)) TVPCPUType |= TVP_CPU_HAS_MMX;   // bit 23
    if (edx & (1 << 25)) TVPCPUType |= TVP_CPU_HAS_SSE;   // bit 25
    if (edx & (1 << 26)) TVPCPUType |= TVP_CPU_HAS_SSE2;  // bit 26

    // 从 ECX 提取扩展特性
    if (ecx & (1 << 0))  TVPCPUType |= TVP_CPU_HAS_SSE3;  // bit 0
    if (ecx & (1 << 9))  TVPCPUType |= TVP_CPU_HAS_SSSE3; // bit 9
    if (ecx & (1 << 19)) TVPCPUType |= TVP_CPU_HAS_SSE41; // bit 19
    if (ecx & (1 << 20)) TVPCPUType |= TVP_CPU_HAS_SSE42; // bit 20

    // AVX 检测需要额外检查 OS 支持（XSAVE/OSXSAVE）
    if ((ecx & (1 << 28)) && (ecx & (1 << 27))) {
        // CPU 支持 AVX 且 OS 已启用 XSAVE
        unsigned long long xcr0 = _xgetbv(0);
        if ((xcr0 & 0x6) == 0x6) {
            // OS 已保存 YMM 寄存器状态
            TVPCPUType |= TVP_CPU_HAS_AVX;

            // CPUID 功能 7：扩展特性（AVX2 在此）
            __cpuidex(cpuInfo, 7, 0);
            if (cpuInfo[1] & (1 << 5)) {  // EBX bit 5
                TVPCPUType |= TVP_CPU_HAS_AVX2;
            }
        }
    }
}
```

Windows 版本还有一个重要特性 — **多核检测**，使用线程亲和性掩码逐核检测：

```cpp
// win32/DetectCPU.cpp — 逐核检测（处理大小核架构）
static void TVPDetectCPUAllCores() {
    DWORD_PTR processAffinity, systemAffinity;
    GetProcessAffinityMask(GetCurrentProcess(),
                           &processAffinity, &systemAffinity);

    tjs_uint32 commonFeatures = 0xFFFFFFFF; // 所有核心共有的特性

    for (int i = 0; i < 64; i++) {
        DWORD_PTR mask = (DWORD_PTR)1 << i;
        if (!(processAffinity & mask)) continue;

        // 将当前线程固定到第 i 个核心
        SetThreadAffinityMask(GetCurrentThread(), mask);
        Sleep(0); // 确保线程已迁移到目标核心

        // 在该核心上执行 CPUID
        tjs_uint32 coreType = 0;
        TVPDetectCPUTypeOnCore(&coreType);

        // 取所有核心的交集（最保守策略）
        commonFeatures &= coreType;
    }

    TVPCPUType = commonFeatures;
    // 恢复原始亲和性
    SetThreadAffinityMask(GetCurrentThread(), processAffinity);
}
```

### 命令行参数控制

Windows 版本支持通过命令行强制启用或禁用特定指令集：

```cpp
// win32/DetectCPU.cpp — 命令行覆盖
// 用法示例：
//   krkr2.exe -cpusse2       # 强制启用 SSE2
//   krkr2.exe -nocpusse2     # 强制禁用 SSE2
//   krkr2.exe -cpuavx2       # 强制启用 AVX2

static struct {
    const char *name;      // 命令行参数名
    tjs_uint32 flag;       // 对应的 CPU 特性标志
} CPUFlags[] = {
    { "mmx",   TVP_CPU_HAS_MMX   },
    { "sse",   TVP_CPU_HAS_SSE   },
    { "sse2",  TVP_CPU_HAS_SSE2  },
    { "sse3",  TVP_CPU_HAS_SSE3  },
    { "ssse3", TVP_CPU_HAS_SSSE3 },
    { "sse41", TVP_CPU_HAS_SSE41 },
    { "sse42", TVP_CPU_HAS_SSE42 },
    { "avx",   TVP_CPU_HAS_AVX   },
    { "avx2",  TVP_CPU_HAS_AVX2  },
    { nullptr, 0 }
};

void TVPApplyCPUCommandLine() {
    for (int i = 0; CPUFlags[i].name; i++) {
        // 检查 -cpuXXX 参数（强制启用）
        ttstr enableOpt = TJS_W("-cpu") + ttstr(CPUFlags[i].name);
        if (TVPGetCommandLine(enableOpt)) {
            TVPCPUType |= CPUFlags[i].flag;
        }
        // 检查 -nocpuXXX 参数（强制禁用）
        ttstr disableOpt = TJS_W("-nocpu") + ttstr(CPUFlags[i].name);
        if (TVPGetCommandLine(disableOpt)) {
            TVPCPUType &= ~CPUFlags[i].flag;
        }
    }
}
```

### 非 Windows 平台：最小检测

在 KrKr2 模拟器的跨平台移植中，非 Windows 平台的 CPU 检测几乎被完全禁用：

```cpp
// DetectCPU.cpp — 非 Windows 平台
// 大部分检测代码被注释掉

void TVPDetectCPU() {
    TVPCPUType = 0;

#if defined(__APPLE__) && defined(__ARM_NEON)
    // Apple ARM (M1/M2/iPhone) — 仅设置 NEON 标志
    TVPCPUType |= TVP_CPU_HAS_NEON;
#endif

    // Linux x86、Android ARM — 不做任何检测
    // 所有 SIMD 路径均不会被激活
}
```

### 跨平台检测对比

| 平台 | 检测方式 | 支持的指令集 | 状态 |
|------|----------|-------------|------|
| Windows x64 | CPUID 指令 + 逐核检测 | MMX/SSE/SSE2/SSE3/SSSE3/SSE4.1/SSE4.2/AVX/AVX2 | ✅ 完整 |
| macOS ARM | 编译期宏 | NEON | ⚠️ 仅标志位 |
| Linux x86 | 无 | 无 | ❌ 未实现 |
| Android ARM | 无 | 无 | ❌ 未实现 |

## 对齐内存分配器

SIMD 指令要求操作数按特定字节边界对齐。SSE 需要 16 字节对齐，AVX 需要 32 字节对齐。未对齐的访问会导致崩溃（`SIGBUS`）或严重性能下降。

### aligned_allocator 实现

```cpp
// aligned_allocator.h — 跨平台对齐内存分配器
// 用作 STL 容器的自定义分配器

template <typename T, std::size_t Alignment = 16>
class aligned_allocator {
public:
    using value_type = T;

    // 分配对齐内存
    T* allocate(std::size_t n) {
        void* ptr = nullptr;
        std::size_t bytes = n * sizeof(T);

#if defined(_MSC_VER) || defined(__MINGW32__)
        // Windows：使用 _mm_malloc（SSE 内在函数提供）
        ptr = _mm_malloc(bytes, Alignment);
#else
        // POSIX（Linux/macOS/Android）：使用 posix_memalign
        if (posix_memalign(&ptr, Alignment, bytes) != 0) {
            ptr = nullptr;
        }
#endif
        if (!ptr) throw std::bad_alloc();
        return static_cast<T*>(ptr);
    }

    // 释放对齐内存（必须使用对应的释放函数）
    void deallocate(T* ptr, std::size_t) {
#if defined(_MSC_VER) || defined(__MINGW32__)
        _mm_free(ptr);    // 配对 _mm_malloc
#else
        free(ptr);        // posix_memalign 的内存可以用 free 释放
#endif
    }
};
```

### 使用示例

```cpp
#include "aligned_allocator.h"
#include <vector>
#include <cstdint>
#include <iostream>
#include <cassert>

int main() {
    // 使用对齐分配器创建像素缓冲区
    // 16 字节对齐，适合 SSE 指令
    std::vector<uint32_t, aligned_allocator<uint32_t, 16>> pixels(1024);

    // 验证对齐
    uintptr_t addr = reinterpret_cast<uintptr_t>(pixels.data());
    assert((addr % 16) == 0);  // 必须 16 字节对齐
    std::cout << "地址: 0x" << std::hex << addr
              << " 对齐: " << (addr % 16 == 0 ? "OK" : "FAIL")
              << std::endl;

    // 32 字节对齐版本（AVX 需要）
    std::vector<uint32_t, aligned_allocator<uint32_t, 32>> avx_buf(1024);
    uintptr_t addr2 = reinterpret_cast<uintptr_t>(avx_buf.data());
    assert((addr2 % 32) == 0);

    return 0;
}
```

### 常见错误：使用错误的释放函数

```cpp
// ❌ 错误：混用分配/释放函数会导致堆损坏
void* ptr = _mm_malloc(1024, 16);
free(ptr);           // 错误！必须用 _mm_free

// ❌ 错误：new 分配的内存不保证对齐
uint32_t* buf = new uint32_t[256];
// buf 可能不是 16 字节对齐的，传给 SSE 指令会崩溃

// ✅ 正确用法
void* ptr = _mm_malloc(1024, 16);
_mm_free(ptr);       // 配对使用

// ✅ 或者使用 C++17 aligned new
uint32_t* buf = static_cast<uint32_t*>(
    ::operator new(256 * sizeof(uint32_t),
                   std::align_val_t{16}));
::operator delete(buf, std::align_val_t{16});
```

## ASM 优化路径的当前状态

### 为什么 ASM 被禁用？

原版 KiriKiri2 引擎包含大量手写 x86 汇编优化（通过 NASM 编写的 `.nas` 文件），但在 KrKr2 跨平台模拟器中，这些优化代码被系统性地禁用了。原因包括：

1. **跨平台兼容性** — 原版 ASM 仅支持 x86/x86_64，无法在 ARM（Android/Apple Silicon）上运行
2. **编译工具链** — `.nas` 文件需要 NASM 汇编器，增加构建复杂度
3. **GPU 渲染替代** — KrKr2 使用 Cocos2d-x OpenGL 后端，大部分像素操作通过 GPU shader 完成，CPU 路径仅作备选
4. **维护成本** — 手写汇编极难维护和调试

### 证据：被禁用的代码

ResampleImage.cpp 中的 AVX2/SSE2 路径被 `#if 0` 包裹：

```cpp
// ResampleImage.cpp — 第 824 行附近
#if 0  // ← 整段 SIMD 优化被禁用
static void ResampleImageSSE2(
    tjs_uint32* dest, tjs_int destWidth,
    const tjs_uint32* src, tjs_int srcWidth, tjs_int srcHeight,
    float scaleX, float scaleY)
{
    // SSE2 双线性插值实现
    __m128i zero = _mm_setzero_si128();
    // ... 完整的 SSE2 优化代码
}

static void ResampleImageAVX2(
    tjs_uint32* dest, tjs_int destWidth,
    const tjs_uint32* src, tjs_int srcWidth, tjs_int srcHeight,
    float scaleX, float scaleY)
{
    // AVX2 版本 — 一次处理 8 个像素
    __m256i zero = _mm256_setzero_si256();
    // ...
}
#endif

// 实际使用的是纯 C 实现
static void ResampleImageC(/*...*/) {
    // 逐像素双线性插值，无任何 SIMD 优化
    for (tjs_int y = 0; y < destHeight; y++) {
        for (tjs_int x = 0; x < destWidth; x++) {
            // 计算源坐标和权重
            // 四邻域采样 + 加权平均
        }
    }
}
```

`TVPGL_ASM_Init()` 在头文件中声明但从未被调用：

```cpp
// tvpgl_asm_init.h — 仅 12 行
#ifndef TVPGL_ASM_INIT_H
#define TVPGL_ASM_INIT_H
extern void TVPGL_ASM_Init();
#endif

// 在整个代码库中搜索 TVPGL_ASM_Init 的调用 → 无结果
// 仅在 blend_function.cpp 中有 TVPGL_C_Init() 的调用
```

## 动手实践：实现一个 SIMD 函数并注册到分发表

以下是一个完整示例，展示如何为 tvpgl 函数指针表添加 SSE2 优化实现：

```cpp
// 文件：simd_alpha_blend_demo.cpp
// 演示：SSE2 优化的 Alpha 混合 + 函数指针注册

#include <cstdint>
#include <cstring>
#include <iostream>
#include <chrono>

#ifdef _MSC_VER
#include <intrin.h>
#else
#include <x86intrin.h>  // GCC/Clang SSE2 内在函数
#endif

// 模拟 tvpgl 的函数指针声明
using BlendFunc = void (*)(uint32_t* dest,
                           const uint32_t* src,
                           int len);

BlendFunc TVPAlphaBlend = nullptr; // 全局函数指针

// ===== 纯 C 实现（基准） =====
void TVPAlphaBlend_c(uint32_t* dest, const uint32_t* src, int len) {
    for (int i = 0; i < len; i++) {
        uint32_t s = src[i];
        uint32_t d = dest[i];
        uint32_t sa = s >> 24;  // 源 Alpha

        if (sa == 255) {
            dest[i] = s;       // 完全不透明，直接覆盖
        } else if (sa > 0) {
            // 0xFF00FF 掩码技巧：同时处理 R 和 B 通道
            uint32_t d_rb = d & 0x00FF00FF;
            uint32_t d_g  = d & 0x0000FF00;
            uint32_t s_rb = s & 0x00FF00FF;
            uint32_t s_g  = s & 0x0000FF00;

            uint32_t rb = (d_rb + (((s_rb - d_rb) * sa) >> 8)) & 0x00FF00FF;
            uint32_t g  = (d_g  + (((s_g  - d_g)  * sa) >> 8)) & 0x0000FF00;

            dest[i] = rb | g | 0xFF000000;
        }
    }
}

// ===== SSE2 实现 =====
#if defined(__SSE2__) || defined(_M_X64)
void TVPAlphaBlend_sse2(uint32_t* dest, const uint32_t* src, int len) {
    __m128i zero = _mm_setzero_si128();

    int i = 0;
    // SSE2 一次处理 4 个像素（128位 / 32位 = 4像素）
    for (; i + 3 < len; i += 4) {
        __m128i s = _mm_loadu_si128((__m128i*)(src + i));
        __m128i d = _mm_loadu_si128((__m128i*)(dest + i));

        // 提取 Alpha 通道并广播到所有通道
        // 分离为 16 位通道进行乘法运算
        __m128i s_lo = _mm_unpacklo_epi8(s, zero); // 像素 0,1 扩展到 16 位
        __m128i s_hi = _mm_unpackhi_epi8(s, zero); // 像素 2,3
        __m128i d_lo = _mm_unpacklo_epi8(d, zero);
        __m128i d_hi = _mm_unpackhi_epi8(d, zero);

        // 提取 Alpha 到每个 16 位通道
        __m128i a_lo = _mm_shufflelo_epi16(s_lo, _MM_SHUFFLE(3,3,3,3));
        a_lo = _mm_shufflehi_epi16(a_lo, _MM_SHUFFLE(3,3,3,3));
        __m128i a_hi = _mm_shufflelo_epi16(s_hi, _MM_SHUFFLE(3,3,3,3));
        a_hi = _mm_shufflehi_epi16(a_hi, _MM_SHUFFLE(3,3,3,3));

        // blend = dest + (src - dest) * alpha / 256
        __m128i diff_lo = _mm_sub_epi16(s_lo, d_lo);
        __m128i diff_hi = _mm_sub_epi16(s_hi, d_hi);

        diff_lo = _mm_mullo_epi16(diff_lo, a_lo);
        diff_hi = _mm_mullo_epi16(diff_hi, a_hi);

        diff_lo = _mm_srli_epi16(diff_lo, 8);
        diff_hi = _mm_srli_epi16(diff_hi, 8);

        __m128i r_lo = _mm_add_epi16(d_lo, diff_lo);
        __m128i r_hi = _mm_add_epi16(d_hi, diff_hi);

        // 重新打包为 8 位
        __m128i result = _mm_packus_epi16(r_lo, r_hi);
        _mm_storeu_si128((__m128i*)(dest + i), result);
    }

    // 处理剩余像素（不足 4 个的尾部）
    for (; i < len; i++) {
        TVPAlphaBlend_c(dest + i, src + i, 1);
    }
}
#endif

// ===== 分发初始化 =====
void InitBlendFunctions() {
    // 第一阶段：C 基准
    TVPAlphaBlend = TVPAlphaBlend_c;
    std::cout << "[INIT] 绑定 C 实现" << std::endl;

#if defined(__SSE2__) || defined(_M_X64)
    // 第二阶段：SSE2 覆盖
    TVPAlphaBlend = TVPAlphaBlend_sse2;
    std::cout << "[INIT] 覆盖为 SSE2 实现" << std::endl;
#endif
}

// ===== 基准测试 =====
int main() {
    const int SIZE = 1024 * 1024; // 1M 像素
    auto* dest = new uint32_t[SIZE];
    auto* src  = new uint32_t[SIZE];

    // 初始化测试数据
    for (int i = 0; i < SIZE; i++) {
        src[i]  = 0x80FF8040;  // 半透明红绿像素
        dest[i] = 0xFF204060;  // 不透明背景
    }

    // 测试 C 版本
    TVPAlphaBlend = TVPAlphaBlend_c;
    auto t0 = std::chrono::high_resolution_clock::now();
    TVPAlphaBlend(dest, src, SIZE);
    auto t1 = std::chrono::high_resolution_clock::now();
    double ms_c = std::chrono::duration<double, std::milli>(t1 - t0).count();
    std::cout << "C 实现: " << ms_c << " ms" << std::endl;

#if defined(__SSE2__) || defined(_M_X64)
    // 重置数据
    for (int i = 0; i < SIZE; i++) dest[i] = 0xFF204060;

    TVPAlphaBlend = TVPAlphaBlend_sse2;
    t0 = std::chrono::high_resolution_clock::now();
    TVPAlphaBlend(dest, src, SIZE);
    t1 = std::chrono::high_resolution_clock::now();
    double ms_sse = std::chrono::duration<double, std::milli>(t1 - t0).count();
    std::cout << "SSE2 实现: " << ms_sse << " ms" << std::endl;
    std::cout << "加速比: " << (ms_c / ms_sse) << "x" << std::endl;
#endif

    delete[] dest;
    delete[] src;
    return 0;
}
```

编译命令（四平台）：

```bash
# Windows (MSVC) — SSE2 默认启用于 x64
cl /O2 /EHsc /std:c++17 simd_alpha_blend_demo.cpp

# Linux (GCC/Clang)
g++ -O2 -msse2 -std=c++17 simd_alpha_blend_demo.cpp -o demo

# macOS (Apple Clang) — x86_64 Mac
clang++ -O2 -msse2 -std=c++17 simd_alpha_blend_demo.cpp -o demo

# Android (NDK, x86_64 模拟器)
$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/x86_64-linux-android31-clang++ \
    -O2 -msse2 -std=c++17 simd_alpha_blend_demo.cpp -o demo
```

## 对照项目源码

相关文件：
- `cpp/core/visual/tvpgl.h` 全文 — 所有 100+ 个函数指针声明
- `cpp/core/visual/blend_function.cpp` 第 1-685 行 — `TVPGL_C_Init()` 完整绑定表
- `cpp/core/visual/tvpgl_asm_init.h` 全文（12 行） — `TVPGL_ASM_Init()` 声明
- `cpp/core/visual/cpu_types.h` 全文（54 行） — CPU 特性标志枚举
- `cpp/core/visual/win32/DetectCPU.cpp` 全文（347 行） — Windows CPUID + 逐核检测
- `cpp/core/visual/DetectCPU.cpp` 全文（176 行） — 非 Windows 平台（大部分注释）
- `cpp/core/visual/aligned_allocator.h` 全文（70 行） — 对齐内存分配器
- `cpp/core/visual/ResampleImage.cpp` 第 820-919 行 — `#if 0` 禁用的 AVX2/SSE2 路径

## 常见错误及解决方案

### 错误 1：AVX2 代码在旧 CPU 上崩溃

```
症状：程序在某些机器上启动即崩溃，信号 SIGILL (Illegal Instruction)
原因：编译时启用了 -mavx2，但运行的 CPU 不支持 AVX2 指令
```

解决方案：**不要在全局编译选项中启用高级 SIMD 标志**，而是使用函数指针分发：

```cpp
// ❌ 错误：全局启用 AVX2 — 所有代码都可能使用 AVX2 指令
// CMakeLists.txt: target_compile_options(mylib PRIVATE -mavx2)

// ✅ 正确：仅对特定文件启用
// CMakeLists.txt:
// set_source_files_properties(simd_avx2.cpp PROPERTIES
//     COMPILE_FLAGS "-mavx2")

// 并在运行时检查 CPU 支持后才调用
if (TVPCPUType & TVP_CPU_HAS_AVX2) {
    TVPAlphaBlend = TVPAlphaBlend_avx2;
}
```

### 错误 2：未对齐内存导致 SSE 崩溃

```
症状：_mm_load_si128 导致段错误
原因：传入的指针不是 16 字节对齐的
```

解决方案：使用 `_mm_loadu_si128`（unaligned load）替代，或确保内存对齐：

```cpp
// ❌ 要求对齐的加载 — 不对齐会崩溃
__m128i data = _mm_load_si128((__m128i*)ptr);

// ✅ 安全的非对齐加载 — 性能略低但不会崩溃
__m128i data = _mm_loadu_si128((__m128i*)ptr);

// ✅ 或者确保内存对齐
alignas(16) uint32_t buffer[4]; // C++11 对齐语法
__m128i data = _mm_load_si128((__m128i*)buffer); // 安全
```

### 错误 3：大小核架构下 SIMD 特性不一致

```
症状：ARM big.LITTLE 或 Intel Alder Lake 混合架构上
      程序随机崩溃（取决于线程被调度到哪个核心）
原因：大核支持 AVX2，小核不支持
```

解决方案：检测所有核心的交集（KrKr2 Windows 版已实现此策略）：

```cpp
// 取所有核心的特性交集，确保安全
tjs_uint32 commonFeatures = 0xFFFFFFFF;
for (int core = 0; core < numCores; core++) {
    pin_to_core(core);
    tjs_uint32 coreFeatures = detect_cpu_on_this_core();
    commonFeatures &= coreFeatures; // 交集
}
TVPCPUType = commonFeatures;
```

## 本节小结

- tvpgl 使用**函数指针分发**实现 SIMD 优化的运行时切换，所有像素操作通过全局函数指针调用
- 初始化分两阶段：`TVPGL_C_Init()` 绑定 C 基准实现，`TVPGL_ASM_Init()` 用 SIMD 版本覆盖
- CPU 检测在 Windows 上使用 CPUID 指令并支持逐核检测和命令行覆盖，其他平台几乎未实现
- `aligned_allocator` 提供跨平台的 SIMD 对齐内存分配（Windows 用 `_mm_malloc`，POSIX 用 `posix_memalign`）
- KrKr2 模拟器中 ASM 优化路径已全部禁用（`#if 0`），因为 GPU 渲染后端已替代大部分 CPU 像素操作

## 练习题与答案

### 题目 1：为什么 KrKr2 使用函数指针而不是 `#ifdef` 编译期切换来实现 SIMD 分发？

<details>
<summary>查看答案</summary>

函数指针分发相比 `#ifdef` 编译期切换有以下优势：

1. **运行时适配**：同一个二进制文件可在不同 CPU 上自动选择最优路径。`#ifdef` 在编译时就固定了，分发给用户的二进制要么全用 SSE2，要么全不用，无法适配不同硬件。

2. **渐进优化**：只需为性能关键的函数编写 SIMD 版本，其他函数自动使用 C 基准。`#ifdef` 需要为每个函数都写两份代码。

3. **测试便利**：可以在运行时强制使用 C 实现来验证 SIMD 实现的正确性（通过命令行参数 `-nocpusse2`），无需重新编译。

4. **函数指针的开销可忽略**：间接调用增加约 1-2 个时钟周期，相比像素操作处理成千上万个像素的总耗时，这个开销微乎其微。

</details>

### 题目 2：编写一个完整的 CPU 特性检测函数，使用 GCC/Clang 的 `__builtin_cpu_supports` 为 Linux 平台实现检测。

<details>
<summary>查看答案</summary>

```cpp
// linux_detect_cpu.cpp — Linux 平台 CPU 检测
#include <cstdint>
#include <iostream>

// 与 KrKr2 兼容的特性标志
enum {
    TVP_CPU_HAS_MMX   = 0x01,
    TVP_CPU_HAS_SSE   = 0x02,
    TVP_CPU_HAS_SSE2  = 0x04,
    TVP_CPU_HAS_SSE3  = 0x08,
    TVP_CPU_HAS_SSSE3 = 0x10,
    TVP_CPU_HAS_SSE41 = 0x20,
    TVP_CPU_HAS_SSE42 = 0x40,
    TVP_CPU_HAS_AVX   = 0x100,
    TVP_CPU_HAS_AVX2  = 0x200,
};

uint32_t TVPCPUType = 0;

void TVPDetectCPU() {
    TVPCPUType = 0;

#if defined(__x86_64__) || defined(__i386__)
    // GCC/Clang 提供的内建 CPU 检测函数
    __builtin_cpu_init(); // 必须先初始化

    if (__builtin_cpu_supports("mmx"))   TVPCPUType |= TVP_CPU_HAS_MMX;
    if (__builtin_cpu_supports("sse"))   TVPCPUType |= TVP_CPU_HAS_SSE;
    if (__builtin_cpu_supports("sse2"))  TVPCPUType |= TVP_CPU_HAS_SSE2;
    if (__builtin_cpu_supports("sse3"))  TVPCPUType |= TVP_CPU_HAS_SSE3;
    if (__builtin_cpu_supports("ssse3")) TVPCPUType |= TVP_CPU_HAS_SSSE3;
    if (__builtin_cpu_supports("sse4.1")) TVPCPUType |= TVP_CPU_HAS_SSE41;
    if (__builtin_cpu_supports("sse4.2")) TVPCPUType |= TVP_CPU_HAS_SSE42;
    if (__builtin_cpu_supports("avx"))   TVPCPUType |= TVP_CPU_HAS_AVX;
    if (__builtin_cpu_supports("avx2"))  TVPCPUType |= TVP_CPU_HAS_AVX2;

#elif defined(__ARM_NEON) || defined(__ARM_NEON__)
    TVPCPUType |= 0x80; // TVP_CPU_HAS_NEON
#endif
}

int main() {
    TVPDetectCPU();
    std::cout << "CPU 特性标志: 0x" << std::hex << TVPCPUType << std::endl;
    if (TVPCPUType & TVP_CPU_HAS_SSE2)
        std::cout << "  SSE2: 支持" << std::endl;
    if (TVPCPUType & TVP_CPU_HAS_AVX2)
        std::cout << "  AVX2: 支持" << std::endl;
    return 0;
}
// 编译: g++ -O2 -std=c++17 linux_detect_cpu.cpp -o detect
```

</details>

### 题目 3：为什么 KrKr2 的 Windows CPU 检测需要逐核执行 CPUID？单次检测不够吗？

<details>
<summary>查看答案</summary>

单次检测不够，原因是**异构多核架构**（Heterogeneous Multi-Core）的存在：

1. **Intel Alder Lake / Raptor Lake**（第 12/13/14 代）使用 P-Core（性能核）+ E-Core（效率核）。P-Core 支持 AVX-512（部分型号）和 AVX2，但 E-Core 可能不支持 AVX-512。如果只在 P-Core 上检测，会错误地认为所有核心都支持 AVX-512。

2. **ARM big.LITTLE**（Android 常见）：大核（Cortex-A7x）和小核（Cortex-A5x）的 SIMD 扩展可能不同。

3. **CPUID 结果是每核独立的**：在哪个核心上执行 CPUID，就返回那个核心的特性。操作系统线程调度器可能把检测线程调度到任意核心。

KrKr2 的解决方案是：通过 `SetThreadAffinityMask` 将线程固定到每个核心上分别执行 CPUID，然后取所有核心特性的**交集**（AND 运算），确保使用的指令集在所有核心上都可用。这是最保守但最安全的策略。

</details>

## 下一步

[过渡系统架构](../06-过渡效果/01-过渡系统架构.md) — 学习 KrKr2 的过渡效果系统，包括淡入淡出、滚动、万能过渡的实现原理。
