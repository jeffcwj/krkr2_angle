# SSE/ASM 优化路径

> **所属模块：** M04-渲染子系统
> **前置知识：** [P05-软件渲染原理](../../P05-软件渲染原理/README.md)、[01-tvpgl像素操作库](./01-tvpgl像素操作库.md)、[02-ARGB混合与合成](./02-ARGB混合与合成.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：

1. 理解原版 KiriKiri2 的 CPU 检测与 SIMD 分派机制
2. 掌握 KrKr2 模拟器当前的纯 C 实现策略及其设计取舍
3. 编写基于 SSE2 intrinsics 的像素混合优化函数
4. 理解 ARM NEON 在 Android 平台上的对应优化方案
5. 能够为 tvpgl 函数指针系统添加 SIMD 优化路径

## CPU 特性检测机制

### cpu_types.h — CPU 能力位掩码

KiriKiri2 引擎定义了一套 CPU 特性标志系统，用于在运行时检测处理器支持哪些指令集扩展。所有标志定义在 `cpp/core/environ/cpu_types.h` 中：

```cpp
// 文件：cpp/core/environ/cpu_types.h

// CPU 特性标志 — 每个标志占用高 16 位中的一个比特
#define TVP_CPU_HAS_FPU   0x00010000  // 浮点运算单元
#define TVP_CPU_HAS_MMX   0x00020000  // MMX 指令集（64位整数SIMD）
#define TVP_CPU_HAS_3DN   0x00040000  // 3DNow!（AMD 浮点SIMD）
#define TVP_CPU_HAS_SSE   0x00080000  // SSE（128位浮点SIMD）
#define TVP_CPU_HAS_CMOV  0x00100000  // 条件移动指令
#define TVP_CPU_HAS_E3DN  0x00200000  // 扩展 3DNow!
#define TVP_CPU_HAS_EMMX  0x00400000  // 扩展 MMX（即 SSE 整数部分）
#define TVP_CPU_HAS_SSE2  0x00800000  // SSE2（128位整数SIMD）
#define TVP_CPU_HAS_TSC   0x01000000  // 时间戳计数器
#define TVP_CPU_HAS_NEON  0x02000000  // ARM NEON（128位SIMD）

// CPU 厂商标识 — 占用中间位
#define TVP_CPU_IS_INTEL      0x00000010
#define TVP_CPU_IS_AMD        0x00000020
#define TVP_CPU_IS_UNKNOWN    0x00000000

// CPU 架构族 — 占用低 4 位
#define TVP_CPU_FAMILY_X86    0x00000001
#define TVP_CPU_FAMILY_X64    0x00000002
#define TVP_CPU_FAMILY_ARM    0x00000003

// 位掩码用于提取各字段
#define TVP_CPU_FEATURE_MASK  0xffff0000  // 高16位=特性
#define TVP_CPU_VENDOR_MASK   0x00000ff0  // 中间位=厂商
#define TVP_CPU_FAMILY_MASK   0x0000000f  // 低4位=架构族

// 全局变量：运行时存储检测到的 CPU 特性
extern unsigned int TVPCPUFeatures;
```

这个位掩码设计非常紧凑。一个 32 位整数同时编码了三类信息：CPU 架构族（x86/x64/ARM）、厂商（Intel/AMD 等）和指令集特性（MMX/SSE/NEON 等）。通过位运算即可快速查询任意组合：

```cpp
#include <cstdint>
#include <cstdio>

// 模拟 cpu_types.h 中的定义
#define TVP_CPU_HAS_SSE2  0x00800000
#define TVP_CPU_HAS_NEON  0x02000000
#define TVP_CPU_FAMILY_X64 0x00000002
#define TVP_CPU_FAMILY_ARM 0x00000003
#define TVP_CPU_FEATURE_MASK 0xffff0000
#define TVP_CPU_FAMILY_MASK  0x0000000f

// 检测函数示例
bool HasSSE2(uint32_t features) {
    return (features & TVP_CPU_HAS_SSE2) != 0;
}

bool HasNEON(uint32_t features) {
    return (features & TVP_CPU_HAS_NEON) != 0;
}

bool IsX64(uint32_t features) {
    return (features & TVP_CPU_FAMILY_MASK) == TVP_CPU_FAMILY_X64;
}

int main() {
    // 模拟 x64 + SSE2 的 CPU
    uint32_t features = TVP_CPU_FAMILY_X64 | TVP_CPU_HAS_SSE2;
    printf("SSE2: %s\n", HasSSE2(features) ? "yes" : "no");   // yes
    printf("NEON: %s\n", HasNEON(features) ? "yes" : "no");   // no
    printf("x64:  %s\n", IsX64(features)   ? "yes" : "no");   // yes
    return 0;
}
```

### DetectCPU.cpp — 运行时 CPU 检测

`cpp/core/environ/DetectCPU.cpp` 实现了运行时 CPU 特性检测。在 KrKr2 模拟器中，这个文件经过了简化处理：

```cpp
// 文件：cpp/core/environ/DetectCPU.cpp（简化展示核心逻辑）

// 全局变量存储检测结果
extern "C" {
    tjs_uint32 TVPCPUType = 0;      // CPU 类型（含厂商+架构）
    tjs_uint32 TVPCPUFeatures = 0;  // CPU 特性标志
}

static bool TVPCPUChecked = false;

// 核心检测函数
void TVPDetectCPU() {
    if (TVPCPUChecked) return;  // 只检测一次
    TVPCPUChecked = true;

#ifdef __APPLE__
    // Apple 平台（iOS/macOS on ARM）直接标记 NEON 支持
    TVPCPUFeatures |= TVP_CPU_FAMILY_ARM | TVP_CPU_HAS_NEON;
#endif

    // 将特性位同步到 TVPCPUType
    tjs_uint32 features = (TVPCPUFeatures & TVP_CPU_FEATURE_MASK);
    TVPCPUType &= ~TVP_CPU_FEATURE_MASK;
    TVPCPUType |= features;
}
```

**关键观察**：原版 KiriKiri2 在此处使用 x86 汇编指令 `CPUID` 来检测 MMX/SSE/SSE2 等特性。KrKr2 模拟器面向多平台（包括 ARM Android），因此将 x86 专用的 CPUID 检测代码注释掉了，仅保留了 Apple 平台的 NEON 标记。

下面是原版 KiriKiri2 的 CPUID 检测原理（注释中保留的逻辑）：

```cpp
#include <cstdint>
#include <cstdio>
#include <cstring>

// 跨平台 CPUID 封装
#if defined(_MSC_VER)
    #include <intrin.h>  // Windows: __cpuid intrinsic
#elif defined(__GNUC__) && (defined(__x86_64__) || defined(__i386__))
    #include <cpuid.h>   // GCC/Clang: __get_cpuid
#endif

struct CPUInfo {
    char vendor[13];     // CPU 厂商字符串
    bool hasMMX;         // MMX 支持
    bool hasSSE;         // SSE 支持
    bool hasSSE2;        // SSE2 支持
    bool hasNEON;        // ARM NEON 支持
};

CPUInfo DetectCPU() {
    CPUInfo info = {};

#if defined(__x86_64__) || defined(__i386__) || defined(_M_X64) || defined(_M_IX86)
    // x86/x64 平台：使用 CPUID 指令
    uint32_t eax, ebx, ecx, edx;

    // CPUID 功能 0：获取厂商字符串
#if defined(_MSC_VER)
    int cpuInfo[4];
    __cpuid(cpuInfo, 0);  // MSVC intrinsic
    eax = cpuInfo[0]; ebx = cpuInfo[1];
    ecx = cpuInfo[2]; edx = cpuInfo[3];
#else
    __get_cpuid(0, &eax, &ebx, &ecx, &edx);  // GCC intrinsic
#endif
    // 厂商字符串存储在 EBX-EDX-ECX 中（注意顺序）
    memcpy(info.vendor + 0, &ebx, 4);
    memcpy(info.vendor + 4, &edx, 4);
    memcpy(info.vendor + 8, &ecx, 4);
    info.vendor[12] = '\0';

    // CPUID 功能 1：获取特性标志
#if defined(_MSC_VER)
    __cpuid(cpuInfo, 1);
    edx = cpuInfo[3];
#else
    __get_cpuid(1, &eax, &ebx, &ecx, &edx);
#endif
    info.hasMMX  = (edx & (1 << 23)) != 0;  // EDX bit 23 = MMX
    info.hasSSE  = (edx & (1 << 25)) != 0;  // EDX bit 25 = SSE
    info.hasSSE2 = (edx & (1 << 26)) != 0;  // EDX bit 26 = SSE2

#elif defined(__aarch64__) || defined(_M_ARM64)
    // ARM64 平台：NEON 是必备指令集
    strcpy(info.vendor, "ARM");
    info.hasNEON = true;

#elif defined(__ARM_NEON)
    // ARM32 + NEON 编译标志
    strcpy(info.vendor, "ARM");
    info.hasNEON = true;
#endif

    return info;
}

int main() {
    CPUInfo info = DetectCPU();
    printf("厂商: %s\n", info.vendor);
    printf("MMX:  %s\n", info.hasMMX  ? "支持" : "不支持");
    printf("SSE:  %s\n", info.hasSSE  ? "支持" : "不支持");
    printf("SSE2: %s\n", info.hasSSE2 ? "支持" : "不支持");
    printf("NEON: %s\n", info.hasNEON ? "支持" : "不支持");
    return 0;
}
```

> **跨平台说明**：
> - **Windows（MSVC）**：使用 `__cpuid()` intrinsic，需 `#include <intrin.h>`
> - **Linux/macOS（GCC/Clang）**：使用 `__get_cpuid()`，需 `#include <cpuid.h>`
> - **Android ARM64**：NEON 是 ARMv8-A 的必备指令集，无需运行时检测
> - **Android ARM32**：需检查编译器是否启用 `-mfpu=neon` 标志

### 常见错误与排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| `__cpuid` 未声明 | 缺少头文件 | MSVC 用 `<intrin.h>`，GCC 用 `<cpuid.h>` |
| ARM64 上 CPUID 编译失败 | CPUID 是 x86 专用指令 | 用 `#if defined(__x86_64__)` 条件编译 |
| NEON 检测不到 | 编译未启用 NEON | CMake 中添加 `-mfpu=neon`（ARM32） |

## 函数指针分派架构

### 设计原理：运行时多态 vs 编译时多态

tvpgl 采用 C 函数指针实现运行时多态（Runtime Polymorphism），这与 C++ 虚函数表类似，但更轻量。其核心思路：

1. **声明阶段**：同时声明函数指针和 C 实现函数
2. **初始化阶段**：根据 CPU 特性将函数指针指向最优实现
3. **调用阶段**：所有调用方通过函数指针间接调用，无需关心具体实现

```
┌──────────────────────────────────────────────────────┐
│                  调用方代码                            │
│         TVPAlphaBlend(dest, src, len)                 │
│                    │                                  │
│                    ▼                                  │
│          ┌──────────────────┐                         │
│          │   函数指针表      │                         │
│          │  TVPAlphaBlend ──┼──┐                      │
│          └──────────────────┘  │                      │
│                                │ 运行时指向            │
│               ┌────────────────┤                      │
│               │                │                      │
│               ▼                ▼                      │
│    ┌─────────────────┐  ┌─────────────────┐           │
│    │ TVPAlphaBlend_c  │  │ TVPAlphaBlend_  │           │
│    │   （纯 C 实现）   │  │ SSE2/NEON 实现  │           │
│    └─────────────────┘  └─────────────────┘           │
└──────────────────────────────────────────────────────┘
```

### TVP_GL_FUNC_PTR 宏系统

tvpgl.h 中定义了四个关键宏，构成函数指针分派的基础设施：

```cpp
// 文件：cpp/core/visual/tvpgl.h 第 47-56 行

// 普通函数声明（不经过函数指针）
#define TVP_GL_FUNC_DECL(rettype, funcname, arg) \
    rettype funcname arg

// 普通函数的 extern 声明
#define TVP_GL_FUNC_EXTERN_DECL(rettype, funcname, arg) \
    extern rettype funcname arg

// 函数指针定义（用于 .cpp 文件中定义全局函数指针变量）
#define TVP_GL_FUNC_PTR_DECL(rettype, funcname, arg) \
    rettype (*funcname) arg

// 函数指针 extern 声明 + C 实现函数声明
#define TVP_GL_FUNC_PTR_EXTERN_DECL_(rettype, funcname, arg) \
    extern rettype (*funcname) arg;      /* 函数指针声明 */   \
    extern rettype funcname##_c arg;     /* C 实现声明 */
#define TVP_GL_FUNC_PTR_EXTERN_DECL TVP_GL_FUNC_PTR_EXTERN_DECL_
```

`TVP_GL_FUNC_PTR_EXTERN_DECL` 是最重要的宏。它一次声明生成两个符号：

```cpp
// 展开前：
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPAlphaBlend,
    (tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len));

// 展开后：
extern void (*TVPAlphaBlend)(tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len);
extern void TVPAlphaBlend_c(tjs_uint32 *dest, const tjs_uint32 *src, tjs_int len);
```

- `TVPAlphaBlend`：函数指针，运行时指向实际实现
- `TVPAlphaBlend_c`：纯 C 实现，始终可用作后备

下面的自包含示例演示了这套机制的完整工作流程：

```cpp
#include <cstdint>
#include <cstdio>
#include <cstring>

// ========== 第一步：宏定义 ==========
#define BLEND_FUNC_PTR_DECL(rettype, funcname, arg) \
    rettype (*funcname) arg

#define BLEND_FUNC_PTR_EXTERN_DECL(rettype, funcname, arg) \
    extern rettype (*funcname) arg;   \
    extern rettype funcname##_c arg;

// ========== 第二步：声明（相当于 tvpgl.h）==========
void (*BlendPixels)(uint32_t *dest, const uint32_t *src, int len);
void BlendPixels_c(uint32_t *dest, const uint32_t *src, int len);

// ========== 第三步：C 实现（相当于 tvpgl.cpp 中的函数体）==========
void BlendPixels_c(uint32_t *dest, const uint32_t *src, int len) {
    for (int i = 0; i < len; ++i) {
        uint32_t sa = (src[i] >> 24) & 0xFF;  // 提取源 Alpha
        uint32_t da = 255 - sa;                 // 目标权重
        uint32_t sr = (src[i] >> 16) & 0xFF;
        uint32_t sg = (src[i] >> 8)  & 0xFF;
        uint32_t sb = src[i] & 0xFF;
        uint32_t dr = (dest[i] >> 16) & 0xFF;
        uint32_t dg = (dest[i] >> 8)  & 0xFF;
        uint32_t db = dest[i] & 0xFF;
        uint32_t r = (sr * sa + dr * da) >> 8;
        uint32_t g = (sg * sa + dg * da) >> 8;
        uint32_t b = (sb * sa + db * da) >> 8;
        dest[i] = (0xFF << 24) | (r << 16) | (g << 8) | b;
    }
}

// ========== 第四步：初始化（相当于 TVPInitTVPGL）==========
void InitBlendLib() {
    BlendPixels = BlendPixels_c;
    printf("[初始化] BlendPixels -> C 实现\n");
}

int main() {
    InitBlendLib();

    uint32_t dest[4] = {0xFF000000, 0xFF000000, 0xFF000000, 0xFF000000};
    uint32_t src[4]  = {0x80FF0000, 0x80FF0000, 0x80FF0000, 0x80FF0000};

    BlendPixels(dest, src, 4);  // 通过函数指针调用

    for (int i = 0; i < 4; ++i) {
        printf("dest[%d] = 0x%08X\n", i, dest[i]);
    }
    return 0;
}
```
