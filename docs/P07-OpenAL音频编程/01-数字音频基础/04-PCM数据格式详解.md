# PCM 数据格式详解

> **所属模块：** P07-OpenAL音频编程
> **前置知识：** [01-声音物理与采样原理](01-声音物理与采样原理.md)、[02-位深与动态范围](02-位深与动态范围.md)、[03-声道布局与码率计算](03-声道布局与码率计算.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：

1. 解释 PCM（Pulse Code Modulation，脉冲编码调制）的 6 种常见编码变体及其应用场景
2. 手动解析 WAV/RIFF 文件的完整二进制结构（含 fmt 和 data chunk）
3. 理解字节序（Endianness）对多字节音频样本的影响
4. 用 C++ 从零实现一个健壮的 WAV 文件读取器
5. 说明 KrKr2 项目中 `tTVPWaveFormat` 和 OpenAL 格式常量的对应关系

## PCM 编码变体

### 什么是 PCM

PCM（Pulse Code Modulation）是最基础的数字音频编码方式。前面几节中，我们已经知道模拟声波经过 **采样** 和 **量化** 后变成了数字信号——这个数字信号的存储格式就是 PCM。PCM 数据就是"原始采样值的序列"，没有任何压缩，每个数字代表某个时刻某个声道的振幅值。

### 6 种常见 PCM 编码格式

| 编码格式 | 字节数 | 取值范围 | 静音值 | 常见用途 |
|----------|--------|----------|--------|----------|
| 无符号 8 位（`uint8`） | 1 | 0 ~ 255 | 128 | 古老的 WAV、低质量音效 |
| 有符号 16 位（`int16`） | 2 | -32768 ~ 32767 | 0 | **CD 音质标准**、游戏音频 |
| 有符号 24 位（`int24`） | 3 | ±8388607 | 0 | 专业录音、母带制作 |
| 有符号 32 位（`int32`） | 4 | ±2147483647 | 0 | 专业 DAW 内部处理 |
| 32 位浮点（`float32`） | 4 | [-1.0, +1.0]（可超出） | 0.0 | 现代音频引擎、混音 |
| 64 位浮点（`float64`） | 8 | [-1.0, +1.0]（可超出） | 0.0 | 科学计算、极高精度 |

> **注意：** 8 位 PCM 是唯一使用**无符号**编码的常见格式，16 位及以上都是**有符号**的。这是历史遗留设计，初学者容易搞混。

### PCM 格式转换代码

不同格式之间的转换在音频编程中非常常见：

```cpp
#include <cstdint>
#include <cmath>

// ====== uint8 ↔ int16 ======
inline int16_t pcm_u8_to_s16(uint8_t sample) {
    return static_cast<int16_t>((sample - 128) << 8);  // 减偏移、左移扩展
}
inline uint8_t pcm_s16_to_u8(int16_t sample) {
    return static_cast<uint8_t>((sample >> 8) + 128);   // 右移、加偏移
}

// ====== int16 ↔ float32 ======
inline float pcm_s16_to_f32(int16_t sample) {
    return sample / 32768.0f;  // 用 32768 而非 32767（见下文解释）
}
inline int16_t pcm_f32_to_s16(float sample) {
    sample = std::fmax(-1.0f, std::fmin(1.0f, sample));  // 限幅
    return static_cast<int16_t>(sample * 32767.0f);
}

// ====== int24（3字节小端） → int32 ======
inline int32_t pcm_s24_to_s32(const uint8_t* bytes) {
    int32_t value = bytes[0] | (bytes[1] << 8) | (bytes[2] << 16);
    if (value & 0x800000) value |= 0xFF000000;  // 符号扩展
    return value;
}
```

### 常见错误 1：8 位 PCM 的无符号陷阱

```cpp
// ❌ 错误：把 uint8 当 int8 处理
int8_t sample = wav_data[i];          // 静音时值为 -128！
float normalized = sample / 128.0f;   // 静音变成 -1.0，完全错误

// ✅ 正确：uint8 减 128 偏移
uint8_t sample = wav_data[i];         // 静音时值为 128
float normalized = (sample - 128) / 128.0f;  // 静音 = 0.0 ✓
```

### 常见错误 2：float → int16 的除数选择

```cpp
// ❌ 用 32767.0f 做除数 → -32768 得到 -1.000031（超出范围）
float normalized = sample / 32767.0f;
// ✅ 用 32768.0f（二补码负端多一个值）→ 范围 [-1.0, 0.999969]
float normalized = sample / 32768.0f;
```

> **为什么用 32768？** 二补码中 int16 范围是 [-32768, 32767]，用 32768 做除数保证 -32768 精确映射到 -1.0，这是 WAV 标准和 OpenAL 遵循的约定。

## WAV/RIFF 文件格式

### RIFF 容器结构

WAV 文件使用 Microsoft 的 **RIFF**（Resource Interchange File Format）容器格式。文件由多个 **chunk**（块）组成，每个 chunk 有 4 字节 ID + 4 字节长度，可按需跳过不认识的 chunk。

```
┌─────────────────────────────────────┐
│  RIFF Header                         │
│  ├── ChunkID:   "RIFF" (4 bytes)    │
│  ├── ChunkSize: 文件大小-8 (4 bytes) │
│  └── Format:    "WAVE" (4 bytes)    │
├─────────────────────────────────────┤
│  fmt Sub-chunk                       │
│  ├── SubchunkID:   "fmt " (4 bytes) │  ← 注意 fmt 后有空格
│  ├── SubchunkSize: 16/18/40 (4 B)   │
│  ├── AudioFormat:  1=PCM (2 bytes)  │
│  ├── NumChannels:  1/2 (2 bytes)    │
│  ├── SampleRate:   44100 (4 bytes)  │
│  ├── ByteRate:     176400 (4 bytes) │
│  ├── BlockAlign:   4 (2 bytes)      │
│  └── BitsPerSample: 16 (2 bytes)   │
├─────────────────────────────────────┤
│  data Sub-chunk                      │
│  ├── SubchunkID:   "data" (4 bytes) │
│  ├── SubchunkSize: N (4 bytes)      │
│  └── Data:         PCM 样本... (N B) │
└─────────────────────────────────────┘
```

### fmt 子块字段详解

| 偏移 | 长度 | 名称 | 说明 |
|------|------|------|------|
| 12 | 4 | Subchunk1ID | ASCII `"fmt "` |
| 16 | 4 | Subchunk1Size | PCM=16，扩展=18 或 40 |
| 20 | 2 | AudioFormat | `1`=PCM, `3`=IEEE float, `0xFFFE`=Extensible |
| 22 | 2 | NumChannels | 声道数 |
| 24 | 4 | SampleRate | 采样率（Hz） |
| 28 | 4 | ByteRate | = SampleRate × NumChannels × BitsPerSample/8 |
| 32 | 2 | BlockAlign | = NumChannels × BitsPerSample/8（一帧字节数） |
| 34 | 2 | BitsPerSample | 位深 |

> **冗余字段：** ByteRate 和 BlockAlign 可从其他字段计算，但 WAV 格式要求显式写入。读取时应验证一致性。

### AudioFormat 标志

| 值 | 含义 | BitsPerSample 对应 |
|----|------|---------------------|
| `1` (0x0001) | 线性 PCM（整数） | 8→uint8, 16→int16, 24→int24, 32→int32 |
| `3` (0x0003) | IEEE 754 浮点 | 32→float32, 64→float64 |
| `0xFFFE` | 扩展格式 | 需读取 SubFormat GUID；>2ch 必用 |

### 字节序注意事项

WAV 中所有多字节整数均为**小端序**（Little-Endian）。x86/x64/ARM 上与内存自然顺序一致，可直接用 `memcpy` 或结构体映射读写。跨平台可移植代码应使用显式函数：

```cpp
inline uint16_t read_le16(const uint8_t* p) {
    return static_cast<uint16_t>(p[0] | (p[1] << 8));
}
inline uint32_t read_le32(const uint8_t* p) {
    return p[0] | (p[1] << 8) | (p[2] << 16) | (p[3] << 24);
}
```

> **常见错误：** 在大端平台用 `reinterpret_cast<int16_t*>(raw_bytes)` 直接读 WAV 数据，会导致字节序反转，播放出刺耳噪声。

## 用 C++ 实现 WAV 文件解析

### 健壮的 WAV 读取器（支持跳过未知 chunk）

```cpp
#include <cstdint>
#include <cstring>
#include <fstream>
#include <iostream>
#include <vector>
#include <stdexcept>

struct WavFile {
    uint16_t audio_format;     // 1=PCM, 3=float
    uint16_t num_channels;
    uint32_t sample_rate;
    uint16_t bits_per_sample;
    uint16_t block_align;
    std::vector<uint8_t> data; // 原始 PCM 数据

    size_t frame_count() const {
        return block_align > 0 ? data.size() / block_align : 0;
    }
    double duration_seconds() const {
        return sample_rate > 0
            ? static_cast<double>(frame_count()) / sample_rate : 0.0;
    }
};

WavFile load_wav(const std::string& path) {
    std::ifstream file(path, std::ios::binary);
    if (!file) throw std::runtime_error("无法打开文件: " + path);

    // 1. 读取 RIFF 头部（12 字节）
    char riff_header[12];
    file.read(riff_header, 12);
    if (!file || std::memcmp(riff_header, "RIFF", 4) != 0
              || std::memcmp(riff_header + 8, "WAVE", 4) != 0) {
        throw std::runtime_error("不是有效的 WAV 文件");
    }

    WavFile wav{};
    bool found_fmt = false, found_data = false;

    // 2. 逐 chunk 解析（不假设顺序）
    while (file && !(found_fmt && found_data)) {
        char chunk_id[4];
        uint32_t chunk_size;
        file.read(chunk_id, 4);
        file.read(reinterpret_cast<char*>(&chunk_size), 4);
        if (!file) break;

        if (std::memcmp(chunk_id, "fmt ", 4) == 0) {
            if (chunk_size < 16)
                throw std::runtime_error("fmt chunk 太小");
            file.read(reinterpret_cast<char*>(&wav.audio_format), 2);
            file.read(reinterpret_cast<char*>(&wav.num_channels), 2);
            file.read(reinterpret_cast<char*>(&wav.sample_rate), 4);
            uint32_t byte_rate;
            file.read(reinterpret_cast<char*>(&byte_rate), 4);
            file.read(reinterpret_cast<char*>(&wav.block_align), 2);
            file.read(reinterpret_cast<char*>(&wav.bits_per_sample), 2);
            if (chunk_size > 16)  // 跳过扩展格式字段
                file.seekg(chunk_size - 16, std::ios::cur);
            found_fmt = true;
        } else if (std::memcmp(chunk_id, "data", 4) == 0) {
            wav.data.resize(chunk_size);
            file.read(reinterpret_cast<char*>(wav.data.data()), chunk_size);
            found_data = true;
        } else {
            file.seekg(chunk_size, std::ios::cur);  // 跳过未知 chunk
        }
        if (chunk_size % 2 != 0)  // RIFF 要求 2 字节对齐
            file.seekg(1, std::ios::cur);
    }

    if (!found_fmt)  throw std::runtime_error("缺少 fmt chunk");
    if (!found_data) throw std::runtime_error("缺少 data chunk");
    if (wav.audio_format != 1 && wav.audio_format != 3)
        throw std::runtime_error("不支持的格式: " +
            std::to_string(wav.audio_format));
    return wav;
}
```

**关键设计要点：**
- **chunk 遍历** —— 不假设 fmt 和 data 的顺序，也不假设它们之间没有其他 chunk（LIST、fact、bext 等）
- **2 字节对齐** —— RIFF 规范要求奇数长度的 chunk 后填充 1 字节
- **扩展格式兼容** —— fmt chunk 大于 16 字节时跳过多余部分

### WAV 文件写入器

```cpp
void save_wav(const std::string& path, const WavFile& wav) {
    std::ofstream file(path, std::ios::binary);
    if (!file) throw std::runtime_error("无法创建文件: " + path);

    uint32_t data_size = static_cast<uint32_t>(wav.data.size());
    uint32_t riff_size = 36 + data_size;
    uint32_t fmt_size = 16;
    uint32_t byte_rate = wav.sample_rate * wav.block_align;

    file.write("RIFF", 4);
    file.write(reinterpret_cast<const char*>(&riff_size), 4);
    file.write("WAVE", 4);
    file.write("fmt ", 4);
    file.write(reinterpret_cast<const char*>(&fmt_size), 4);
    file.write(reinterpret_cast<const char*>(&wav.audio_format), 2);
    file.write(reinterpret_cast<const char*>(&wav.num_channels), 2);
    file.write(reinterpret_cast<const char*>(&wav.sample_rate), 4);
    file.write(reinterpret_cast<const char*>(&byte_rate), 4);
    file.write(reinterpret_cast<const char*>(&wav.block_align), 2);
    file.write(reinterpret_cast<const char*>(&wav.bits_per_sample), 2);
    file.write("data", 4);
    file.write(reinterpret_cast<const char*>(&data_size), 4);
    file.write(reinterpret_cast<const char*>(wav.data.data()), data_size);
}
```

## 动手实践

### 完整示例：WAV 文件信息查看器

```cpp
// wav_info.cpp
#include <cstdint>
#include <cstring>
#include <fstream>
#include <iostream>
#include <vector>
#include <stdexcept>
#include <cmath>
#include <iomanip>

// [上面的 WavFile 结构体和 load_wav 函数]

int main(int argc, char* argv[]) {
    if (argc < 2) { std::cerr << "用法: wav_info <file.wav>\n"; return 1; }
    try {
        WavFile wav = load_wav(argv[1]);
        std::cout << "格式:    " << (wav.audio_format == 1 ? "PCM" : "Float")
                  << "\n声道:    " << wav.num_channels
                  << "\n采样率:  " << wav.sample_rate << " Hz"
                  << "\n位深:    " << wav.bits_per_sample << " bit"
                  << "\n帧数:    " << wav.frame_count()
                  << std::fixed << std::setprecision(3)
                  << "\n时长:    " << wav.duration_seconds() << " 秒"
                  << "\n码率:    " << wav.sample_rate * wav.num_channels
                                     * wav.bits_per_sample / 1000.0
                  << " kbps" << std::endl;

        // 16 位 PCM 峰值分析
        if (wav.bits_per_sample == 16 && wav.audio_format == 1) {
            auto* samples = reinterpret_cast<const int16_t*>(wav.data.data());
            size_t count = wav.data.size() / 2;
            int16_t peak = 0;
            double sum_sq = 0.0;
            for (size_t i = 0; i < count; ++i) {
                if (std::abs(samples[i]) > peak) peak = std::abs(samples[i]);
                double n = samples[i] / 32768.0;
                sum_sq += n * n;
            }
            std::cout << "峰值:    " << 20.0 * std::log10(peak / 32768.0)
                      << " dBFS\nRMS:     "
                      << 20.0 * std::log10(std::sqrt(sum_sq / count))
                      << " dBFS" << std::endl;
        }
    } catch (const std::exception& e) {
        std::cerr << "错误: " << e.what() << std::endl;
        return 1;
    }
}
```

```cmake
cmake_minimum_required(VERSION 3.19)
project(wav_info LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
add_executable(wav_info wav_info.cpp)
```

**运行示例：**

```
$ ./wav_info test.wav
格式:    PCM
声道:    2
采样率:  44100 Hz
位深:    16 bit
帧数:    220500
时长:    5.000 秒
码率:    1411.200 kbps
峰值:    0.000 dBFS
RMS:     -3.010 dBFS
```

## 对照项目源码

### KrKr2 的音频格式定义

```cpp
// cpp/core/sound/WaveIntf.h 第 43-53 行
struct tTVPWaveFormat {
    tjs_uint SamplesPerSec;   // 采样率（如 44100）
    tjs_uint Channels;        // 声道数
    tjs_uint BitsPerSample;   // 位深（8/16）
    tjs_uint BytesPerSample;  // = BitsPerSample / 8
    tjs_uint64 TotalSamples;  // 总帧数
    tjs_uint64 TotalTime;     // 总时长（毫秒）
    tjs_uint32 SpeakerConfig; // 声道布局
};
```

### 格式到 OpenAL 常量的映射

```cpp
// cpp/core/sound/win32/WaveMixer.cpp 第 515-524 行（等效逻辑）
ALenum get_al_format(uint16_t channels, uint16_t bits) {
    if (channels == 1 && bits == 8)  return AL_FORMAT_MONO8;
    if (channels == 1 && bits == 16) return AL_FORMAT_MONO16;
    if (channels == 2 && bits == 8)  return AL_FORMAT_STEREO8;
    if (channels == 2 && bits == 16) return AL_FORMAT_STEREO16;
    return 0;  // 不支持的格式
}
```

> **OpenAL 格式限制：** 标准 OpenAL 仅支持 Mono8/Mono16/Stereo8/Stereo16 四种格式。超 16 位或超 2 声道需先转换。OpenAL Soft 扩展支持 float32，但 KrKr2 未使用。

### KrKr2 默认音频参数

```cpp
// cpp/core/sound/win32/WaveImpl.cpp 第 62-63 行
tjs_int TVPPrimarySBFrequency = 44100;  // 默认采样率
tjs_int TVPPrimarySBBits = 16;          // 默认位深
// 但 iTVPAudioRenderer 实际使用 48000 Hz（WaveMixer.cpp 第 245 行）
```

KrKr2 的音频管线：**解码器输出 → 采样率转换 → 16 位 PCM → OpenAL 缓冲区**。

## 本节小结

- PCM 有 6 种常见编码，8 位是无符号（静音=128），16 位及以上有符号（静音=0）
- WAV 使用 RIFF 容器格式，标准头 44 字节 = RIFF 头(12B) + fmt 块(24B) + data 块头(8B)
- 字节序：WAV 全部小端，x86/ARM 无需转换，大端平台必须用显式字节序函数
- 健壮的 WAV 解析器需支持跳过未知 chunk、处理扩展 fmt、2 字节对齐
- KrKr2 仅用 OpenAL 的 4 种基础格式，不使用 float 扩展

## 练习题与答案

### 题目 1：手动计算 WAV 文件大小

一个 3 秒的 WAV 文件，44100 Hz、16 位、立体声。计算 (a) data chunk 大小、(b) 文件总大小、(c) riff_size 字段值。

<details>
<summary>查看答案</summary>

```
(a) data = 44100 × 3 × 2 × 2 = 529200 字节
(b) 文件 = 44 头 + 529200 = 529244 字节 ≈ 516.8 KB
(c) riff_size = 529244 - 8 = 529236（减去 "RIFF" + size 本身的 8 字节）
```

</details>

### 题目 2：修复有 Bug 的 WAV 读取代码

以下代码有 3 个 Bug，找出并修复：

```cpp
bool load_simple_wav(const char* path, std::vector<int16_t>& out) {
    FILE* f = fopen(path, "r");           // Bug 1
    WavHeader header;
    fread(&header, sizeof(header), 1, f);
    if (header.audio_format != 1) return false;
    int sample_count = header.data_size;  // Bug 2
    out.resize(sample_count);
    fread(out.data(), 2, sample_count, f);
    fclose(f);                            // Bug 3 位置
    return true;
}
```

<details>
<summary>查看答案</summary>

```cpp
bool load_simple_wav(const char* path, std::vector<int16_t>& out) {
    FILE* f = fopen(path, "rb");  // Bug 1: "r"→"rb"，文本模式会破坏二进制数据
    if (!f) return false;
    WavHeader header;
    fread(&header, sizeof(header), 1, f);
    if (header.audio_format != 1) { fclose(f); return false; }  // Bug 3: 泄漏句柄
    size_t sample_count = header.data_size / sizeof(int16_t);   // Bug 2: 字节→样本
    out.resize(sample_count);
    fread(out.data(), sizeof(int16_t), sample_count, f);
    fclose(f);
    return true;
}
```

1. `fopen("r")` → `fopen("rb")`：Windows 文本模式将 `0x0D 0x0A` 转换为 `0x0A`
2. `data_size` 是**字节数**不是样本数，需 `/ sizeof(int16_t)`
3. `audio_format != 1` 的 early return 未调用 `fclose(f)`，泄漏文件句柄

</details>

### 题目 3：实现 PCM 格式转换

编写函数将 16 位立体声 WAV 转换为 8 位单声道（左右平均 + 正确偏移）：

<details>
<summary>查看答案</summary>

```cpp
WavFile convert_stereo16_to_mono8(const WavFile& src) {
    if (src.num_channels != 2 || src.bits_per_sample != 16)
        throw std::runtime_error("输入必须是 16 位立体声");

    WavFile dst{};
    dst.audio_format = 1;
    dst.num_channels = 1;
    dst.sample_rate = src.sample_rate;
    dst.bits_per_sample = 8;
    dst.block_align = 1;  // 1声道 × 1字节

    size_t frames = src.frame_count();
    dst.data.resize(frames);
    const int16_t* in = reinterpret_cast<const int16_t*>(src.data.data());

    for (size_t i = 0; i < frames; ++i) {
        // 立体声→单声道：左右平均（用 int32 防溢出）
        int32_t mono = (static_cast<int32_t>(in[i*2]) + in[i*2+1]) / 2;
        // 16位有符号 → 8位无符号：右移8位 + 加128偏移
        dst.data[i] = static_cast<uint8_t>((mono >> 8) + 128);
    }
    return dst;  // 数据量 = 原始的 1/4（声道减半 × 位深减半）
}
```

</details>

---

## 下一步

[下一节：音频缓冲区与延迟](05-音频缓冲区与延迟.md) — 缓冲区大小的选择策略、延迟来源分析、双缓冲/三缓冲机制。
