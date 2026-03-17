# WAV 文件加载与 PCM 解码

> **所属模块：** P07-OpenAL 音频编程
> **前置知识：** [静态缓冲播放](../03-缓冲与流式播放/01-静态缓冲播放.md)、[流式队列播放](../03-缓冲与流式播放/02-流式队列播放.md)
> **预计阅读时间：** 20 分钟

## 本节目标

读完本节后，你将能够：
1. 实现健壮的 WAV 文件解析器，正确处理各种 chunk 布局
2. 将 WAV 中的 PCM 数据映射为 OpenAL 格式常量
3. 实现一次性加载和内存映射两种加载策略
4. 处理 WAV 文件中常见的异常情况（非 PCM 编码、奇数大小 chunk 等）

## WAV 格式快速回顾

WAV 基于 RIFF 容器格式，由嵌套的 chunk 组成。核心结构：

```
RIFF header:  "RIFF" + fileSize + "WAVE"
  ├── fmt  chunk:  音频格式参数（编码、声道、采样率、位深）
  ├── data chunk:  原始 PCM 样本数据
  └── 其他 chunk:  LIST/INFO/fact 等（可选，需跳过）
```

**关键限制：** `fmt` 不一定紧跟在 RIFF header 之后，`data` 也可能不在 `fmt` 之后。健壮的解析器必须遍历所有 chunk。

### fmt chunk 字段

```cpp
struct WavFmt {
    uint16_t audioFormat;    // 1=PCM, 3=IEEE float, 其他=压缩
    uint16_t numChannels;    // 声道数
    uint32_t sampleRate;     // 采样率 (Hz)
    uint32_t byteRate;       // 每秒字节数
    uint16_t blockAlign;     // 每帧字节数 = channels * bitsPerSample/8
    uint16_t bitsPerSample;  // 位深：8/16/24/32
};
```

### PCM 格式到 OpenAL 的映射

OpenAL 原生只支持 4 种 PCM 格式：

| 声道 | 位深 | OpenAL 常量 | blockAlign |
|------|------|-------------|------------|
| 1 (Mono) | 8 | `AL_FORMAT_MONO8` | 1 |
| 1 (Mono) | 16 | `AL_FORMAT_MONO16` | 2 |
| 2 (Stereo) | 8 | `AL_FORMAT_STEREO8` | 2 |
| 2 (Stereo) | 16 | `AL_FORMAT_STEREO16` | 4 |

24-bit 和 32-bit PCM 需要转换为 16-bit 后才能用于 OpenAL（除非使用 AL_EXT_FLOAT32 扩展）。

## 示例 1：健壮的 WAV 解析器

```cpp
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <vector>
#include <string>

struct WavFile {
    uint16_t format;       // 1=PCM, 3=float
    uint16_t channels;
    uint32_t sampleRate;
    uint16_t bitsPerSample;
    uint16_t blockAlign;
    std::vector<uint8_t> data;  // 原始 PCM 数据
};

// 从 4 字节读取小端 uint32
static uint32_t readU32(const uint8_t* p) {
    return p[0] | (p[1]<<8) | (p[2]<<16) | (p[3]<<24);
}
static uint16_t readU16(const uint8_t* p) {
    return p[0] | (p[1]<<8);
}

bool loadWav(const char* path, WavFile& wav) {
    FILE* f = fopen(path, "rb");
    if (!f) return false;

    // 读取 RIFF header (12 bytes)
    uint8_t hdr[12];
    if (fread(hdr, 1, 12, f) != 12) { fclose(f); return false; }
    if (memcmp(hdr, "RIFF", 4) != 0 || memcmp(hdr+8, "WAVE", 4) != 0) {
        fclose(f); return false;
    }

    bool hasFmt = false, hasData = false;

    // 遍历所有 chunk
    while (!feof(f)) {
        uint8_t chunkHdr[8];
        if (fread(chunkHdr, 1, 8, f) != 8) break;
        uint32_t chunkSize = readU32(chunkHdr + 4);

        if (memcmp(chunkHdr, "fmt ", 4) == 0) {
            uint8_t fmt[16];
            if (chunkSize < 16 || fread(fmt, 1, 16, f) != 16) {
                fclose(f); return false;
            }
            wav.format = readU16(fmt);
            wav.channels = readU16(fmt + 2);
            wav.sampleRate = readU32(fmt + 4);
            wav.bitsPerSample = readU16(fmt + 14);
            wav.blockAlign = readU16(fmt + 12);
            hasFmt = true;
            // 跳过 fmt 中多余的字节（如扩展格式）
            if (chunkSize > 16) fseek(f, chunkSize - 16, SEEK_CUR);
        } else if (memcmp(chunkHdr, "data", 4) == 0) {
            wav.data.resize(chunkSize);
            size_t got = fread(wav.data.data(), 1, chunkSize, f);
            wav.data.resize(got);
            hasData = true;
        } else {
            // 跳过未知 chunk
            fseek(f, chunkSize, SEEK_CUR);
        }
        // RIFF 规范：chunk 大小为奇数时需要 1 字节填充对齐
        if (chunkSize & 1) fseek(f, 1, SEEK_CUR);
    }
    fclose(f);
    return hasFmt && hasData;
}
```

关键细节：
- **chunk 遍历**：不假设 chunk 顺序，`fmt` 和 `data` 可能被 LIST/INFO 等 chunk 隔开
- **奇数对齐**：RIFF 规范要求 chunk body 为奇数字节时补 1 字节填充
- **扩展格式**：`fmt` chunk 可能大于 16 字节（WAVEFORMATEX），多余部分需跳过

## 示例 2：格式映射与加载函数

```cpp
#include <AL/al.h>
#include <AL/alc.h>

// 将 WAV 参数映射为 OpenAL 格式常量
ALenum wavToALFormat(const WavFile& wav) {
    if (wav.format != 1) return 0;  // 仅支持 PCM
    if (wav.channels == 1 && wav.bitsPerSample == 8)
        return AL_FORMAT_MONO8;
    if (wav.channels == 1 && wav.bitsPerSample == 16)
        return AL_FORMAT_MONO16;
    if (wav.channels == 2 && wav.bitsPerSample == 8)
        return AL_FORMAT_STEREO8;
    if (wav.channels == 2 && wav.bitsPerSample == 16)
        return AL_FORMAT_STEREO16;
    return 0;  // 不支持的组合
}

// 加载 WAV 到 OpenAL Buffer
ALuint loadWavToALBuffer(const char* path) {
    WavFile wav;
    if (!loadWav(path, wav)) {
        fprintf(stderr, "无法加载: %s\n", path);
        return 0;
    }
    ALenum fmt = wavToALFormat(wav);
    if (fmt == 0) {
        fprintf(stderr, "不支持的格式: ch=%d bits=%d fmt=%d\n",
                wav.channels, wav.bitsPerSample, wav.format);
        return 0;
    }
    ALuint buf;
    alGenBuffers(1, &buf);
    alBufferData(buf, fmt, wav.data.data(),
                 (ALsizei)wav.data.size(), wav.sampleRate);
    ALenum err = alGetError();
    if (err != AL_NO_ERROR) {
        fprintf(stderr, "alBufferData 错误: 0x%X\n", err);
        alDeleteBuffers(1, &buf);
        return 0;
    }
    return buf;
}
```

## 示例 3：24-bit PCM 转 16-bit

很多专业音频工具导出 24-bit WAV。OpenAL 不支持 24-bit，需转换：

```cpp
// 将 24-bit PCM 数据就地转换为 16-bit
std::vector<uint8_t> convert24to16(const uint8_t* src, size_t bytes) {
    size_t samples = bytes / 3;  // 每样本 3 字节
    std::vector<uint8_t> dst(samples * 2);
    for (size_t i = 0; i < samples; ++i) {
        // 24-bit 小端：[low, mid, high]，取高 16 位
        int16_t val = (int16_t)((src[i*3+2] << 8) | src[i*3+1]);
        dst[i*2]     = (uint8_t)(val & 0xFF);
        dst[i*2 + 1] = (uint8_t)((val >> 8) & 0xFF);
    }
    return dst;
}
```

集成到加载流程：

```cpp
ALenum wavToALFormatEx(WavFile& wav) {
    if (wav.format != 1) return 0;
    // 24-bit → 16-bit 转换
    if (wav.bitsPerSample == 24) {
        wav.data = convert24to16(wav.data.data(), wav.data.size());
        wav.bitsPerSample = 16;
        wav.blockAlign = wav.channels * 2;
    }
    return wavToALFormat(wav);
}
```

## 示例 4：完整播放程序

```cpp
#include <AL/al.h>
#include <AL/alc.h>
#include <cstdio>
#include <thread>
#include <chrono>

int main(int argc, char** argv) {
    if (argc < 2) {
        printf("用法: %s <file.wav>\n", argv[0]);
        return 1;
    }
    // 初始化 OpenAL
    ALCdevice* dev = alcOpenDevice(nullptr);
    if (!dev) { fprintf(stderr, "无法打开设备\n"); return 1; }
    ALCcontext* ctx = alcCreateContext(dev, nullptr);
    alcMakeContextCurrent(ctx);

    // 加载 WAV
    ALuint buf = loadWavToALBuffer(argv[1]);
    if (!buf) { return 1; }

    // 查询 Buffer 信息
    ALint size, freq, bits, channels;
    alGetBufferi(buf, AL_SIZE, &size);
    alGetBufferi(buf, AL_FREQUENCY, &freq);
    alGetBufferi(buf, AL_BITS, &bits);
    alGetBufferi(buf, AL_CHANNELS, &channels);
    float duration = (float)size / (freq * channels * bits / 8);
    printf("已加载: %dHz, %dch, %dbit, %.2f秒\n",
           freq, channels, bits, duration);

    // 播放
    ALuint src;
    alGenSources(1, &src);
    alSourcei(src, AL_BUFFER, buf);
    alSourcePlay(src);

    // 等待播放完成
    ALint state;
    do {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        alGetSourcei(src, AL_SOURCE_STATE, &state);
    } while (state == AL_PLAYING);

    // 清理
    alDeleteSources(1, &src);
    alDeleteBuffers(1, &buf);
    alcMakeContextCurrent(nullptr);
    alcDestroyContext(ctx);
    alcCloseDevice(dev);
    printf("播放完成\n");
    return 0;
}
```

### 示例 5：CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(wav_player)
find_package(OpenAL CONFIG REQUIRED)
add_executable(wav_player main.cpp)
target_link_libraries(wav_player PRIVATE OpenAL::OpenAL)
target_compile_features(wav_player PRIVATE cxx_std_17)
```

## 常见错误与解决方案

### 错误 1：假设 fmt 紧跟 RIFF header

```cpp
// 错误：直接跳到偏移 12 读 fmt
fseek(f, 12, SEEK_SET);
fread(&fmt, sizeof(fmt), 1, f);  // 如果中间有 JUNK chunk 就读错了
```

**解决：** 遍历所有 chunk 按 ID 匹配，不假设位置。参考示例 1 的 `while (!feof(f))` 循环。

### 错误 2：忽略 blockAlign 导致静态噪音

```cpp
// 错误：用错误的帧大小计算 data 字节数
size_t frameSize = channels * bitsPerSample / 8;  // 可能与 blockAlign 不一致
```

某些 WAV 编码器在 `blockAlign` 中写入与计算值不同的数字（如包含填充字节）。**解决：** 始终使用 `fmt` chunk 中的 `blockAlign` 字段，不要自己计算。

### 错误 3：不处理 chunk 对齐

```cpp
// 错误：跳过 chunk 后直接读下一个
fseek(f, chunkSize, SEEK_CUR);  // 缺少奇数对齐处理
```

RIFF 规范要求每个 chunk 的 body 起始地址 2 字节对齐。若 `chunkSize` 为奇数，后面有 1 字节填充。**解决：** `if (chunkSize & 1) fseek(f, 1, SEEK_CUR);`

## 动手实践

1. **编译示例 4**，用任意 WAV 文件测试播放
2. **测试 24-bit WAV**：用 Audacity 导出一个 24-bit WAV，验证 `convert24to16` 转换后能否正常播放
3. **添加 32-bit float 支持**：扩展 `wavToALFormatEx`，将 `audioFormat==3`（IEEE float）的 32-bit 数据转换为 16-bit PCM
4. **错误处理增强**：给 `loadWav` 添加详细错误信息（用 `std::string& errorMsg` 输出参数），区分"文件打开失败"、"不是 WAV"、"缺少 fmt"、"缺少 data"

## 对照项目源码

- `cpp/core/sound/win32/WaveMixer.cpp` 第 523-546 行 — KrKr2 的音频格式处理，使用 `AL_FORMAT_MONO8/MONO16/STEREO8/STEREO16` 四种格式，与本节映射表完全对应
- `cpp/core/sound/WaveIntf.h` 第 46-52 行 — `tTVPWaveFormat` 结构体定义了采样率、声道数、位深等字段，与 WAV fmt chunk 字段一一对应

## 本节小结

- WAV 是 RIFF 容器格式，通过遍历 chunk 解析 `fmt`（格式参数）和 `data`（PCM 数据）
- 健壮的解析器不假设 chunk 顺序，处理奇数对齐和扩展格式
- OpenAL 原生支持 Mono/Stereo × 8/16-bit 共 4 种格式，24-bit 需转换
- `alBufferData` 一次性将 PCM 数据提交到 AL Buffer，适合小文件播放

## 练习题与答案

### 题目 1：格式判断

一个 WAV 文件的 fmt chunk 显示：`audioFormat=1, channels=2, bitsPerSample=24, sampleRate=96000`。能否直接用于 OpenAL？如果不能，如何处理？

<details>
<summary>查看答案</summary>

不能直接使用。OpenAL 不支持 24-bit 格式。处理方案：将 24-bit PCM 转换为 16-bit（取每样本的高 16 位），然后用 `AL_FORMAT_STEREO16` 提交。采样率 96000 可直接传给 `alBufferData` 的 `freq` 参数。

</details>

### 题目 2：计算播放时长

WAV 文件 `data` chunk 大小为 705600 字节，格式为 Stereo 16-bit 44100Hz。计算播放时长。

<details>
<summary>查看答案</summary>

每帧字节数 = 2（声道）× 2（16-bit）= 4 字节
总帧数 = 705600 / 4 = 176400 帧
时长 = 176400 / 44100 = **4.0 秒**

</details>

### 题目 3：实现 float32 转 int16

编写函数将 IEEE float32 PCM（范围 -1.0~+1.0）转换为 int16 PCM：

<details>
<summary>查看答案</summary>

```cpp
std::vector<uint8_t> convertFloat32toInt16(const uint8_t* src, size_t bytes) {
    size_t samples = bytes / 4;  // float32 每样本 4 字节
    std::vector<uint8_t> dst(samples * 2);
    const float* fp = reinterpret_cast<const float*>(src);
    int16_t* out = reinterpret_cast<int16_t*>(dst.data());
    for (size_t i = 0; i < samples; ++i) {
        float v = fp[i];
        if (v > 1.0f) v = 1.0f;      // 钳位
        if (v < -1.0f) v = -1.0f;
        out[i] = (int16_t)(v * 32767.0f);
    }
    return dst;
}
```

</details>

## 下一步

[下一节：流式播放引擎实现](02-流式播放引擎实现.md) — 将 WAV 解析器与环形缓冲区结合，实现支持大文件的流式播放引擎。