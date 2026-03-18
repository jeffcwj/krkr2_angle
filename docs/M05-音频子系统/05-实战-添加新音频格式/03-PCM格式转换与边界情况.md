# PCM 格式转换与边界情况

> **所属模块：** M05-音频子系统
> **前置知识：** [01-FLAC解码器实现](./01-FLAC解码器实现.md)、[03-音频播放数据流](../01-sound模块架构/03-音频播放数据流.md)
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：

1. 深入理解 KrKr2 音频系统的 PCM 格式转换函数实现原理
2. 正确处理 8/16/24/32 位整数和 32 位浮点 PCM 之间的转换
3. 处理字节序（大端/小端）差异与跨平台适配
4. 识别并正确处理边界情况：溢出、下溢、静音、极端值
5. 实现高效的 SIMD 优化转换路径

---

## 5.11 PCM 格式转换概述

### 5.11.1 KrKr2 音频管线中的格式转换点

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        KrKr2 音频解码管线                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌───────────┐ │
│  │   FLAC       │    │   Filter     │    │   Mixer      │    │  OpenAL   │ │
│  │  Decoder     │───▶│   Chain      │───▶│  (SDL)       │───▶│  Output   │ │
│  │ (8/16/24/32) │    │ (Float/Int)  │    │  (16-bit)    │    │           │ │
│  └──────────────┘    └──────────────┘    └──────────────┘    └───────────┘ │
│         │                   │                   │                          │
│         │                   │                   │                          │
│         ▼                   ▼                   ▼                          │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                  │
│  │ 位深度归一化  │    │ Float↔Int   │    │ 下混声道     │                  │
│  │ (→16 or →F)  │    │  转换        │    │ (Downmix)    │                  │
│  └──────────────┘    └──────────────┘    └──────────────┘                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.11.2 核心转换函数

KrKr2 在 `WaveIntf.cpp` 中提供四个核心转换函数：

| 函数 | 输入 | 输出 | 用途 |
|------|------|------|------|
| `TVPConvertPCMTo16bits` | 任意格式 | 16 位整数 | 送给 SDL/OpenAL 混音器 |
| `TVPConvertPCMToFloat` | 任意格式 | 32 位浮点 | 送给 DSP 滤波器（如 PhaseVocoder） |
| `TVPConvertFloatPCMTo16bits` | 浮点 | 16 位 | DSP 后转回整数 |
| `TVPConvertIntegerPCMTo16bits` | 整数(8/16/24/32) | 16 位 | 各种位深度归一化 |

---

## 5.12 整数 PCM 转换详解

### 5.12.1 位深度转换原理

PCM 样本的位深度转换本质是 **数值范围映射**：

```
位深度    有符号范围              无符号范围
8 位      -128 ~ +127            0 ~ 255
16 位     -32768 ~ +32767        0 ~ 65535
24 位     -8388608 ~ +8388607    0 ~ 16777215
32 位     -2^31 ~ +2^31-1        0 ~ 2^32-1
浮点      -1.0 ~ +1.0            N/A
```

**向上转换（扩展位深度）：** 左移填零

```cpp
// 8 位 → 16 位：左移 8 位
int8_t sample8 = 64;           // 二进制: 0100 0000
int16_t sample16 = sample8 << 8;  // 0100 0000 0000 0000 = 16384
```

**向下转换（缩减位深度）：** 右移截断

```cpp
// 24 位 → 16 位：右移 8 位
int32_t sample24 = 0x7FABCD;      // 8,367,053
int16_t sample16 = sample24 >> 8;  // 0x7FAB = 32683
// 丢失低 8 位精度
```

### 5.12.2 TVPConvertIntegerPCMTo16bits 实现分析

```cpp
// cpp/core/sound/WaveIntf.cpp 第 143-268 行
static void TVPConvertIntegerPCMTo16bits(
    tjs_int16 *output, const void *input,
    tjs_int bytespersample,      // 每样本字节数 (1/2/3/4)
    tjs_int validbits,           // 有效位数 (可能小于 bytespersample*8)
    tjs_int channels,            // 通道数
    tjs_int count,               // 样本数
    bool downmix) {              // 是否下混到单声道
    
    // 8 位处理（无符号→有符号转换）
    if(bytespersample == 1) {
        const tjs_int8 *p = (tjs_int8 *)input;
        if(!downmix || channels == 1) {
            tjs_int total = channels * count;
            while(total--)
                // 8 位 PCM 是无符号的，需要减去 0x80 转为有符号
                // 然后左移 8 位扩展到 16 位
                *(output++) = (tjs_int16)(((tjs_int)*(p++) - 0x80) * 0x100);
        } else {
            // 下混处理...
        }
    }
    
    // 16 位处理（应用有效位掩码）
    else if(bytespersample == 2) {
        // 计算掩码，只保留有效位
        tjs_uint16 mask = ~((1 << (16 - validbits)) - 1);
        const tjs_int16 *p = (const tjs_int16 *)input;
        if(!downmix || channels == 1) {
            tjs_int total = channels * count;
            while(total--)
                *(output++) = (tjs_int16)(*(p++) & mask);
        } else {
            // 下混处理...
        }
    }
    
    // 24 位处理（字节序敏感）
    else if(bytespersample == 3) {
        tjs_uint32 mask = ~((1 << (24 - validbits)) - 1);
        const tjs_uint8 *p = (const tjs_uint8 *)input;
        
        tjs_int total = channels * count;
        while(total--) {
            // 手动组装 24 位值（小端序）
            tjs_int32 t = p[0] + (p[1] << 8) + (p[2] << 16);
            p += 3;
            t |= -(t & 0x800000);  // 符号扩展
            t &= mask;
            t >>= 8;  // 转换到 16 位
            *(output++) = (tjs_int16)t;
        }
    }
    
    // 32 位处理
    else if(bytespersample == 4) {
        tjs_int32 mask = ~((1 << (32 - validbits)) - 1);
        const tjs_int32 *p = (const tjs_int32 *)input;
        tjs_int total = channels * count;
        while(total--)
            *(output++) = (tjs_int16)((*(p++) & mask) >> 16);
    }
}
```

### 5.12.3 关键技巧解析

**1. 8 位无符号转有符号**

WAV 格式中 8 位 PCM 是无符号的（0-255），静音点在 128：

```cpp
// 无符号 8 位：0 = 最小, 128 = 静音, 255 = 最大
// 有符号 16 位：-32768 = 最小, 0 = 静音, +32767 = 最大

uint8_t sample_u8 = 200;        // 无符号
int16_t sample_s16 = ((int)sample_u8 - 0x80) * 0x100;
// = (200 - 128) * 256 = 72 * 256 = 18432
```

**2. 24 位符号扩展**

24 位整数存储在 32 位变量中时，需要手动扩展符号位：

```cpp
tjs_int32 t = p[0] + (p[1] << 8) + (p[2] << 16);  // 组装 24 位
// 如果 t 的第 23 位是 1（负数），需要将高 8 位设为全 1

// 巧妙的符号扩展：
t |= -(t & 0x800000);
// 解释：
// - (t & 0x800000) 取出符号位（0 或 0x800000）
// - 取负：0 → 0，0x800000 → 0xFF800000
// - OR 运算：正数不变，负数高位填 1

// 示例：
// t = 0x800001（24 位负数 -8388607）
// t & 0x800000 = 0x800000
// -(0x800000) = 0xFF800000
// t | 0xFF800000 = 0xFF800001（32 位表示的 -8388607）
```

**3. 有效位掩码**

某些格式的有效位数小于存储位数（如 20 位存在 24 位容器中）：

```cpp
// validbits = 20, bytespersample = 3 (24 位)
tjs_uint32 mask = ~((1 << (24 - 20)) - 1);
// = ~((1 << 4) - 1) = ~0xF = 0xFFFFFFF0

// 掩码保留高 20 位，清零低 4 位（无效位）
```

---

## 5.13 浮点 PCM 转换

### 5.13.1 浮点范围与整数映射

```
浮点 PCM 范围：-1.0 ~ +1.0（静音 = 0.0）

浮点    →    16 位整数
+1.0    →    +32767
 0.0    →    0
-1.0    →    -32768

转换公式：
int16 = float * 32768.0  （向下取整，并 clamp 到范围内）
float = int16 / 32768.0
```

### 5.13.2 TVPConvertFloatPCMTo16bits 实现

```cpp
// cpp/core/sound/WaveIntf.cpp 第 97-140 行
static void TVPConvertFloatPCMTo16bits(
    tjs_int16 *output, const float *input,
    tjs_int channels, tjs_int count, bool downmix) {
    
    if(!downmix) {
        tjs_int total = channels * count;
        // 可能使用 SIMD 优化版本
        PCMConvertLoopFloat32ToInt16(output, input, total);
    } else {
        // 下混到单声道
        float nc = 32768.0f / (float)channels;  // 归一化系数
        while(count--) {
            tjs_int n = channels;
            float t = 0;
            while(n--)
                t += *(input++) * nc;  // 累加所有通道
            
            // 四舍五入 + 钳位
            if(t > 0) {
                int i = (int)(t + 0.5);
                if(i > 32767) i = 32767;  // 钳位上限
                *(output++) = (tjs_int16)i;
            } else {
                int i = (int)(t - 0.5);
                if(i < -32768) i = -32768;  // 钳位下限
                *(output++) = (tjs_int16)i;
            }
        }
    }
}
```

### 5.13.3 标量转换实现

当 SIMD 不可用时，使用标量循环：

```cpp
// 标量版本的 Float32 → Int16
void PCMConvertLoopFloat32ToInt16_scalar(
    void *dest, const void *src, size_t numsamples) {
    
    tjs_int16 *d = (tjs_int16 *)dest;
    const float *s = (const float *)src;
    
    for(size_t i = 0; i < numsamples; i++) {
        float sample = s[i] * 32768.0f;
        
        // 钳位（Clamp）
        if(sample > 32767.0f) sample = 32767.0f;
        if(sample < -32768.0f) sample = -32768.0f;
        
        // 四舍五入转换
        d[i] = (tjs_int16)(sample > 0 ? sample + 0.5f : sample - 0.5f);
    }
}
```

### 5.13.4 TVPConvertPCMToFloat 实现

```cpp
// cpp/core/sound/WaveIntf.cpp 第 305-372 行
static void TVPConvertIntegerPCMToFloat(
    float *output, const void *input,
    tjs_int bytespersample, tjs_int validbits,
    tjs_int channels, tjs_int count) {
    
    if(bytespersample == 1) {
        const tjs_int8 *p = (tjs_int8 *)input;
        tjs_int total = channels * count;
        while(total--)
            // 8 位：减 0x80 转有符号，除 128 归一化到 [-1, 1)
            *(output++) = (float)(((tjs_int)*(p++) - 0x80) * (1.0 / 128));
    }
    else if(bytespersample == 2) {
        const tjs_int16 *p = (const tjs_int16 *)input;
        tjs_int total = channels * count;
        
        if(validbits == 16) {
            // 最常见情况，可能使用 SIMD
            PCMConvertLoopInt16ToFloat32(output, p, total);
        } else {
            tjs_uint16 mask = ~((1 << (16 - validbits)) - 1);
            while(total--)
                *(output++) = (float)((*(p++) & mask) * (1.0 / 32768));
        }
    }
    else if(bytespersample == 3) {
        // 24 位处理
        const tjs_uint8 *p = (const tjs_uint8 *)input;
        tjs_uint32 mask = ~((1 << (24 - validbits)) - 1);
        tjs_int total = channels * count;
        while(total--) {
            tjs_int32 t = p[0] + (p[1] << 8) + (p[2] << 16);
            p += 3;
            t |= -(t & 0x800000);  // 符号扩展
            t &= mask;
            *(output++) = (float)(t * (1.0 / (1 << 23)));
        }
    }
    else if(bytespersample == 4) {
        // 32 位处理
        tjs_int32 mask = ~((1 << (32 - validbits)) - 1);
        const tjs_int32 *p = (const tjs_int32 *)input;
        tjs_int total = channels * count;
        while(total--)
            *(output++) = (float)(((*(p++) & mask) >> 0) * (1.0 / (1 << 31)));
    }
}
```

---

## 5.14 字节序处理

### 5.14.1 大端与小端

```
16 位值 0x1234 在内存中的存储：

小端序（Little-Endian，x86/ARM）：
地址:    [0]  [1]
内容:    0x34 0x12  （低字节在低地址）

大端序（Big-Endian，PowerPC/SPARC）：
地址:    [0]  [1]
内容:    0x12 0x34  （高字节在低地址）
```

### 5.14.2 KrKr2 字节序处理

```cpp
// cpp/core/sound/WaveIntf.cpp 第 452-479 行（RIFFWave Render 方法）

#if TJS_HOST_IS_BIG_ENDIAN
// 大端机器上需要交换 WAV 文件（小端格式）的字节序

if(Format.BytesPerSample == 2) {
    tjs_uint16 *p = (tjs_uint16 *)buf;
    tjs_uint16 *plim = (tjs_uint16 *)((tjs_uint8 *)buf + read);
    while(p < plim) {
        // 16 位字节交换
        *p = (*p >> 8) + (*p << 8);
        p++;
    }
}
else if(Format.BytesPerSample == 3) {
    tjs_uint8 *p = (tjs_uint8 *)buf;
    tjs_uint8 *plim = (tjs_uint8 *)buf + read;
    while(p < plim) {
        // 24 位字节交换（交换首尾字节）
        tjs_uint8 tmp = p[0];
        p[0] = p[2];
        p[2] = tmp;
        p += 3;
    }
}
else if(Format.BytesPerSample == 4) {
    tjs_uint32 *p = (tjs_uint32 *)buf;
    tjs_uint32 *plim = (tjs_uint32 *)((tjs_uint8 *)buf + read);
    while(p < plim) {
        // 32 位字节交换
        *p = ((*p & 0xff000000) >> 24) +
             ((*p & 0x00ff0000) >> 8)  +
             ((*p & 0x0000ff00) << 8)  +
             ((*p & 0x000000ff) << 24);
        p++;
    }
}
#endif
```

### 5.14.3 跨平台编译宏

```cpp
// 检测主机字节序（通常在配置头文件中定义）
#if defined(__BYTE_ORDER__) && __BYTE_ORDER__ == __ORDER_BIG_ENDIAN__
    #define TJS_HOST_IS_BIG_ENDIAN 1
#elif defined(__BIG_ENDIAN__) || defined(_BIG_ENDIAN)
    #define TJS_HOST_IS_BIG_ENDIAN 1
#elif defined(__LITTLE_ENDIAN__) || defined(_LITTLE_ENDIAN)
    #define TJS_HOST_IS_BIG_ENDIAN 0
#else
    // 默认假设小端（x86/ARM 都是小端）
    #define TJS_HOST_IS_BIG_ENDIAN 0
#endif
```

**平台字节序：**

| 平台 | 架构 | 字节序 |
|------|------|--------|
| Windows | x86/x64 | 小端 |
| Linux | x86/x64/ARM | 小端 |
| macOS | x64/ARM64 | 小端 |
| Android | ARM/ARM64 | 小端 |
| 网络传输 | - | 大端（网络序） |

KrKr2 目标平台全部是小端，字节序代码通常不会执行。

---

## 5.15 边界情况处理

### 5.15.1 溢出与钳位

浮点转整数时可能超出目标范围：

```cpp
// 错误示例：直接截断导致环绕
float sample = 1.5f;  // 超出 [-1, 1] 范围
int16_t bad = (int16_t)(sample * 32768.0f);  // = 49152，但截断为 -16384！

// 正确示例：钳位（Clamp）
float sample = 1.5f;
float scaled = sample * 32768.0f;  // = 49152
if(scaled > 32767.0f) scaled = 32767.0f;  // 钳位到最大值
int16_t good = (int16_t)scaled;  // = 32767
```

### 5.15.2 静音检测

```cpp
// 检测缓冲区是否为静音（用于优化）
bool IsSilent(const tjs_int16 *buffer, size_t samples) {
    for(size_t i = 0; i < samples; i++) {
        if(buffer[i] != 0) return false;
    }
    return true;
}

// 浮点版本（允许微小误差）
bool IsSilentFloat(const float *buffer, size_t samples) {
    const float threshold = 1e-6f;
    for(size_t i = 0; i < samples; i++) {
        if(fabsf(buffer[i]) > threshold) return false;
    }
    return true;
}
```

### 5.15.3 DC 偏移处理

某些音频文件可能有 DC 偏移（静音时不为零）：

```cpp
// 计算 DC 偏移
float CalculateDCOffset(const float *buffer, size_t samples) {
    double sum = 0;
    for(size_t i = 0; i < samples; i++) {
        sum += buffer[i];
    }
    return (float)(sum / samples);
}

// 移除 DC 偏移
void RemoveDCOffset(float *buffer, size_t samples, float offset) {
    for(size_t i = 0; i < samples; i++) {
        buffer[i] -= offset;
    }
}
```

### 5.15.4 极端值测试

```cpp
TEST_CASE("PCM conversion edge cases", "[sound][pcm]") {
    SECTION("16-bit max positive") {
        float input = 1.0f;
        tjs_int16 output;
        // 期望：32767（不是 32768，因为有符号 16 位最大是 32767）
        ConvertFloat32ToInt16(&output, &input, 1);
        REQUIRE(output == 32767);
    }
    
    SECTION("16-bit max negative") {
        float input = -1.0f;
        tjs_int16 output;
        // 期望：-32768
        ConvertFloat32ToInt16(&output, &input, 1);
        REQUIRE(output == -32768);
    }
    
    SECTION("overflow clipping") {
        float input = 2.0f;  // 超出范围
        tjs_int16 output;
        ConvertFloat32ToInt16(&output, &input, 1);
        REQUIRE(output == 32767);  // 应该钳位到最大值
    }
    
    SECTION("underflow clipping") {
        float input = -2.0f;  // 超出范围
        tjs_int16 output;
        ConvertFloat32ToInt16(&output, &input, 1);
        REQUIRE(output == -32768);  // 应该钳位到最小值
    }
    
    SECTION("NaN handling") {
        float input = NAN;
        tjs_int16 output;
        // NaN 应该转换为 0 或者不崩溃
        REQUIRE_NOTHROW(ConvertFloat32ToInt16(&output, &input, 1));
    }
    
    SECTION("Infinity handling") {
        float input = INFINITY;
        tjs_int16 output;
        ConvertFloat32ToInt16(&output, &input, 1);
        REQUIRE(output == 32767);  // 正无穷钳位到最大值
    }
}
```

---

## 5.16 多平台适配

### 5.16.1 SIMD 优化（SSE/NEON）

KrKr2 源码中有 SIMD 优化的注释代码（目前禁用）：

```cpp
// cpp/core/sound/WaveIntf.cpp 第 107-119 行
if(!downmix) {
    tjs_int total = channels * count;
#if 0  // SSE 路径（已禁用）
    bool use_sse =
        (TVPCPUType & TVP_CPU_HAS_MMX) &&
        (TVPCPUType & TVP_CPU_HAS_SSE) &&
        (TVPCPUType & TVP_CPU_HAS_CMOV);
    
    if(use_sse)
        PCMConvertLoopFloat32ToInt16_sse(output, input, total);
    else
        PCMConvertLoopFloat32ToInt16(output, input, total);
#else
    PCMConvertLoopFloat32ToInt16(output, input, total);
#endif
}
```

**SSE 实现示例（参考）：**

```cpp
// SSE 版本的 Float32 → Int16
void PCMConvertLoopFloat32ToInt16_sse(
    void *dest, const void *src, size_t numsamples) {
    
    tjs_int16 *d = (tjs_int16 *)dest;
    const float *s = (const float *)src;
    
    __m128 scale = _mm_set1_ps(32768.0f);
    __m128 max_val = _mm_set1_ps(32767.0f);
    __m128 min_val = _mm_set1_ps(-32768.0f);
    
    size_t i = 0;
    // 每次处理 8 个样本（2 个 __m128）
    for(; i + 7 < numsamples; i += 8) {
        // 加载 8 个 float
        __m128 f0 = _mm_loadu_ps(s + i);
        __m128 f1 = _mm_loadu_ps(s + i + 4);
        
        // 缩放
        f0 = _mm_mul_ps(f0, scale);
        f1 = _mm_mul_ps(f1, scale);
        
        // 钳位
        f0 = _mm_min_ps(_mm_max_ps(f0, min_val), max_val);
        f1 = _mm_min_ps(_mm_max_ps(f1, min_val), max_val);
        
        // 转换为 int32
        __m128i i0 = _mm_cvtps_epi32(f0);
        __m128i i1 = _mm_cvtps_epi32(f1);
        
        // 打包为 int16
        __m128i packed = _mm_packs_epi32(i0, i1);
        
        // 存储
        _mm_storeu_si128((__m128i *)(d + i), packed);
    }
    
    // 处理剩余样本
    for(; i < numsamples; i++) {
        float sample = s[i] * 32768.0f;
        if(sample > 32767.0f) sample = 32767.0f;
        if(sample < -32768.0f) sample = -32768.0f;
        d[i] = (tjs_int16)sample;
    }
}
```

### 5.16.2 平台特定实现

```cpp
// 根据平台选择实现
#if defined(__x86_64__) || defined(_M_X64)
    #define USE_SSE_CONVERSION 1
#elif defined(__aarch64__) || defined(_M_ARM64)
    #define USE_NEON_CONVERSION 1
#else
    #define USE_SCALAR_CONVERSION 1
#endif

// 函数指针初始化
void (*PCMConvertLoopFloat32ToInt16)(void*, const void*, size_t) = nullptr;

void InitPCMConversion() {
#if USE_SSE_CONVERSION
    PCMConvertLoopFloat32ToInt16 = PCMConvertLoopFloat32ToInt16_sse;
#elif USE_NEON_CONVERSION
    PCMConvertLoopFloat32ToInt16 = PCMConvertLoopFloat32ToInt16_neon;
#else
    PCMConvertLoopFloat32ToInt16 = PCMConvertLoopFloat32ToInt16_scalar;
#endif
}
```

---

## 动手实践

### 实践 1：实现 24 位浮点转换

补全以下函数，将 24 位整数 PCM 转换为浮点：

```cpp
void Convert24BitToFloat(float *output, const uint8_t *input, 
                         size_t samples) {
    for(size_t i = 0; i < samples; i++) {
        // TODO: 组装 24 位值，符号扩展，归一化到 [-1, 1]
        
    }
}
```

<details>
<summary>查看答案</summary>

```cpp
void Convert24BitToFloat(float *output, const uint8_t *input, 
                         size_t samples) {
    for(size_t i = 0; i < samples; i++) {
        // 组装 24 位值（小端序）
        int32_t sample = input[0] + (input[1] << 8) + (input[2] << 16);
        input += 3;
        
        // 符号扩展：如果第 23 位是 1，扩展到高 8 位
        if(sample & 0x800000) {
            sample |= 0xFF000000;
        }
        
        // 归一化到 [-1, 1)
        // 24 位有符号最大值是 0x7FFFFF = 8388607
        output[i] = (float)sample / 8388608.0f;
    }
}
```

</details>

### 实践 2：验证钳位逻辑

```cpp
// 测试你的钳位实现
void TestClipping() {
    float inputs[] = {0.0f, 0.5f, 1.0f, 1.5f, -1.0f, -1.5f, NAN, INFINITY};
    int16_t expected[] = {0, 16384, 32767, 32767, -32768, -32768, 0, 32767};
    
    for(int i = 0; i < 8; i++) {
        int16_t result = ConvertWithClipping(inputs[i]);
        assert(result == expected[i]);
    }
}
```

---

## 对照项目源码

相关参考文件：

- `cpp/core/sound/WaveIntf.cpp` 第 97-140 行 — `TVPConvertFloatPCMTo16bits`
- `cpp/core/sound/WaveIntf.cpp` 第 143-268 行 — `TVPConvertIntegerPCMTo16bits`
- `cpp/core/sound/WaveIntf.cpp` 第 305-372 行 — `TVPConvertIntegerPCMToFloat`
- `cpp/core/sound/WaveIntf.cpp` 第 452-479 行 — 大端字节序处理
- `cpp/core/sound/WaveIntf.cpp` 第 775-860 行 — 滤波器链中的格式转换

滤波器链中的格式转换实现：

```cpp
// cpp/core/sound/WaveIntf.cpp 第 807-819 行
class SoundBufferConvertFloatToPCM16 : public iSoundBufferConvertToPCM16 {
    std::vector<float> buffer;

    void Decode(void *dest, tjs_uint samples, tjs_uint &written,
                tTVPWaveSegmentQueue &segments) override {
        tjs_uint numsamples = samples * OutputFormat.Channels;
        if(buffer.size() < numsamples)
            buffer.resize(numsamples);
        float *p = &buffer[0];
        Source->Decode(p, samples, written, segments);
        numsamples = written * OutputFormat.Channels;
        PCMConvertLoopFloat32ToInt16(dest, p, numsamples);
    }
};
```

---

## 本节小结

- PCM 格式转换本质是 **数值范围映射**：整数位移/浮点缩放
- 8 位 PCM 在 WAV 中是 **无符号**，需减 0x80 转有符号
- 24 位值需要 **手动符号扩展**：`t |= -(t & 0x800000)`
- 浮点转整数必须 **钳位**，防止溢出导致环绕
- 字节序处理仅在大端机器上需要，KrKr2 目标平台全部小端
- SIMD 优化可提升转换性能，但需要平台适配

---

## 练习题与答案

### 题目 1：为什么 8 位 PCM 转 16 位要减去 0x80 再乘 0x100？

<details>
<summary>查看答案</summary>

WAV 格式中 8 位 PCM 是**无符号**的，范围 0-255，静音点在 128：

| 无符号 8 位 | 含义 | 有符号 16 位 |
|-------------|------|--------------|
| 0 | 最小幅度 | -32768 |
| 128 | 静音 | 0 |
| 255 | 最大幅度 | +32512 |

转换公式：`int16 = (uint8 - 128) * 256`

- 减 128：将无符号 [0, 255] 转为有符号 [-128, 127]
- 乘 256（即左移 8 位）：扩展到 16 位范围 [-32768, +32512]

注意：由于 127 * 256 = 32512，最大正值比 32767 小，这是历史设计的结果。

</details>

### 题目 2：浮点值 0.75 转换为 16 位整数应该是多少？写出计算过程。

<details>
<summary>查看答案</summary>

```
浮点值：0.75
范围映射：[-1.0, +1.0] → [-32768, +32767]

计算：0.75 * 32768 = 24576

验证：
- 0.75 是 3/4 满幅
- 32767 * 0.75 ≈ 24575（四舍五入）

最终结果：24576

代码实现：
float sample = 0.75f;
int16_t result = (int16_t)(sample * 32768.0f);
// result = 24576
```

</details>

### 题目 3：写出将 32 位整数 PCM 转换为浮点的函数。

<details>
<summary>查看答案</summary>

```cpp
void Convert32BitIntToFloat(float *output, const int32_t *input, 
                             size_t samples) {
    // 32 位有符号整数最大值：2^31 - 1 = 2147483647
    // 归一化系数：1.0 / 2^31 = 1.0 / 2147483648
    const float scale = 1.0f / 2147483648.0f;
    
    for(size_t i = 0; i < samples; i++) {
        output[i] = (float)input[i] * scale;
    }
}

// 测试
void TestConvert32Bit() {
    int32_t input[] = {0, 2147483647, -2147483648, 1073741824};
    float output[4];
    
    Convert32BitIntToFloat(output, input, 4);
    
    // 期望值：
    // 0 → 0.0
    // 2147483647 → ~0.9999999995 (接近 1.0)
    // -2147483648 → -1.0
    // 1073741824 → 0.5
    
    assert(fabsf(output[0] - 0.0f) < 1e-6f);
    assert(fabsf(output[1] - 1.0f) < 1e-6f);
    assert(fabsf(output[2] - (-1.0f)) < 1e-6f);
    assert(fabsf(output[3] - 0.5f) < 1e-6f);
}
```

</details>

---

## 模块总结

恭喜完成 M05-音频子系统的全部内容！你现在掌握了：

1. **sound 模块架构**：目录结构、核心类层次、数据流
2. **解码器注册机制**：tTVPWaveDecoder 接口、Creator 链、格式探测
3. **OpenAL 混音器**：iTVPSoundBuffer 接口、双层缓冲、解码线程
4. **循环与 DSP**：WaveLoopManager、WaveSegmentQueue、PhaseVocoder
5. **实战添加新格式**：完整 FLAC 解码器实现、注册集成、PCM 转换

下一步建议学习 **M04-渲染子系统** 或 **M06-视频播放器**，继续深入 KrKr2 引擎核心。
