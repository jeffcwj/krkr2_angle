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

