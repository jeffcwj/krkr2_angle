# PhaseVocoder变速不变调

> **所属模块：** M05-音频子系统
> **前置知识：** [02-WaveSegmentQueue分段播放](./02-WaveSegmentQueue分段播放.md)、[P06-FFmpeg音视频开发](../../P06-FFmpeg音视频开发/README.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：
1. 理解相位声码器（Phase Vocoder）的基本原理与应用场景
2. 掌握KrKr2中tRisaPhaseVocoderDSP的架构与核心算法
3. 理解FFT分析-变换-合成的完整流程
4. 学会使用TJS2脚本控制音频变速与变调
5. 实现自定义DSP滤波器链

---

## 1. Phase Vocoder概述

### 1.1 什么是Phase Vocoder？

**Phase Vocoder（相位声码器）** 是一种基于FFT的音频时间拉伸/音调变换技术，可以实现：

| 效果 | 说明 | 应用场景 |
|------|------|----------|
| 变速不变调 | 播放速度改变，音高不变 | 视觉小说快进、语速调整 |
| 变调不变速 | 音高改变，播放速度不变 | 角色音色变换、音效处理 |
| 同时变速变调 | 两者同时改变 | 特殊音效 |

### 1.2 为什么不用简单的重采样？

| 方法 | 变速 | 变调 | 质量 |
|------|------|------|------|
| 简单丢弃/重复采样 | ✓ | ✗ | 低（有断裂感） |
| 线性插值重采样 | ✓ | ✗ | 中（有混叠） |
| 改变采样率 | ✓ | ✓（联动） | 高（但变调） |
| Phase Vocoder | ✓ | ✓（独立） | 高 |

**Phase Vocoder的独特优势：** 时间轴和频率轴可以独立控制。

### 1.3 基本原理图解

```
原始音频信号 ──▶ 分帧 ──▶ 加窗 ──▶ FFT分析 ──▶ 极坐标转换
                                        │
                                        ▼
                                   频谱数据
                                   (幅度+相位)
                                        │
        ┌───────────────────────────────┴───────────────────────────────┐
        │                          变换处理                              │
        │  • 时间拉伸：调整帧重叠率（HopSize）                          │
        │  • 音调变换：频率轴重采样                                      │
        │  • 相位修正：保持相位连续性                                    │
        └───────────────────────────────┬───────────────────────────────┘
                                        │
                                        ▼
                               直角坐标转换 ──▶ IFFT合成 ──▶ 加窗 ──▶ 重叠相加
                                                                      │
                                                                      ▼
                                                                输出音频信号
```

---

## 2. 核心类架构

### 2.1 类层次结构

```
tRisaPhaseVocoderDSP          ← 底层DSP处理引擎
        │
        └── 被使用 ──▶ tTJSNI_PhaseVocoder    ← TJS2绑定包装类
                              │
                              ├── 实现 iTVPBasicWaveFilter    ← 滤波器接口
                              └── 继承 tTVPSampleAndLabelSource ← 数据源接口
```

### 2.2 tRisaPhaseVocoderDSP —— DSP引擎

```cpp
// 文件: cpp/core/sound/PhaseVocoderDSP.h 第21-198行

class tRisaPhaseVocoderDSP {
protected:
    // === 工作缓冲区 ===
    float **AnalWork;       // 分析工作缓冲区 [Channels][FrameSize]
    float **SynthWork;      // 合成工作缓冲区 [Channels][FrameSize]
    float **LastAnalPhase;  // 上次分析相位 [Channels][FrameSize/2]
    float **LastSynthPhase; // 上次合成相位 [Channels][FrameSize/2]

    // === FFT相关 ===
    int *FFTWorkIp;         // FFT工作参数ip
    float *FFTWorkW;        // FFT工作参数w
    float *InputWindow;     // 输入窗函数
    float *OutputWindow;    // 输出窗函数

    // === 参数 ===
    unsigned int FrameSize;     // FFT大小（2的幂，如4096）
    unsigned int OverSampling;  // 重叠系数（如8）
    unsigned int Frequency;     // 采样率
    unsigned int Channels;      // 声道数
    unsigned int InputHopSize;  // 输入跳跃大小 = FrameSize/OverSampling
    unsigned int OutputHopSize; // 输出跳跃大小 = InputHopSize * TimeScale

    float TimeScale;        // 时间缩放因子（>1变慢，<1变快）
    float FrequencyScale;   // 频率缩放因子（>1升调，<1降调）

    // === 环形缓冲区 ===
    tRisaRingBuffer<float> InputBuffer;   // 输入缓冲
    tRisaRingBuffer<float> OutputBuffer;  // 输出缓冲

public:
    // 处理状态枚举
    enum tStatus {
        psNoError,         // 正常
        psInputNotEnough,  // 输入不足
        psOutputFull       // 输出已满
    };

    // 核心方法
    tRisaPhaseVocoderDSP(unsigned int framesize, unsigned int frequency,
                         unsigned int channels);
    tStatus Process();           // 执行一步处理
    void ProcessCore(int ch);    // 单声道核心处理

    // 参数设置
    void SetTimeScale(float v);
    void SetFrequencyScale(float v);
    void SetOverSampling(unsigned int v);
};
```

### 2.3 关键参数说明

| 参数 | 典型值 | 说明 |
|------|--------|------|
| FrameSize | 4096 | FFT窗口大小（音乐用4096，语音用256） |
| OverSampling | 8 | 重叠系数（越大质量越好，计算量越大） |
| TimeScale | 1.0 | 时间缩放（2.0=慢2倍，0.5=快2倍） |
| FrequencyScale | 1.0 | 频率缩放（2.0=升1个八度，0.5=降1个八度） |
| InputHopSize | 512 | FrameSize/OverSampling |
| OutputHopSize | 512 | InputHopSize * TimeScale（偶数对齐） |

### 2.4 tTJSNI_PhaseVocoder —— TJS2绑定

```cpp
// 文件: cpp/core/sound/PhaseVocoderFilter.h 第23-95行

class tTJSNI_PhaseVocoder : public tTJSNativeInstance,
                            public iTVPBasicWaveFilter,
                            public tTVPSampleAndLabelSource {
private:
    int Window;    // 窗口大小（64~32768，2的幂）
    int Overlap;   // 重叠系数（0,2,4,8,16,32）
    float Pitch;   // 音调缩放
    float Time;    // 时间缩放

    tTVPSampleAndLabelSource *Source;         // 上游数据源
    tRisaPhaseVocoderDSP *PhaseVocoder;        // DSP实例
    char *FormatConvertBuffer;                 // 格式转换缓冲

    tTVPWaveFormat InputFormat;   // 输入格式
    tTVPWaveFormat OutputFormat;  // 输出格式（始终为32位float）

    tTVPWaveSegmentQueue InputSegments;   // 输入分段队列
    tTVPWaveSegmentQueue OutputSegments;  // 输出分段队列

public:
    // TJS2脚本可访问的属性
    int GetWindow() const;
    void SetWindow(int window);
    int GetOverlap() const;
    void SetOverlap(int overlap);
    float GetPitch() const;
    void SetPitch(float pitch);
    float GetTime() const;
    void SetTime(float time);

    // 核心解码方法
    void Decode(void *dest, tjs_uint samples, tjs_uint &written,
                tTVPWaveSegmentQueue &segments) override;
};
```

---

## 3. 核心算法详解

### 3.1 Process() —— 主处理循环

```cpp
// 文件: cpp/core/sound/PhaseVocoderDSP.cpp 第299-430行

tRisaPhaseVocoderDSP::tStatus tRisaPhaseVocoderDSP::Process() {
    // === 步骤1：参数重建（如果需要） ===
    if(RebuildParams) {
        // 计算Vorbis I窗函数
        float output_volume = TimeScale / FrameSize / sqrt(FrequencyScale) /
            OverSampling * 2 * 2.0f;  // 2.0 = Vorbis窗补偿
        for(unsigned int i = 0; i < FrameSize; i++) {
            double x = ((double)i + 0.5) / FrameSize;
            // Vorbis I 窗: sin(π/2 * sin²(πx))
            double window = sin(M_PI / 2 * sin(M_PI * x) * sin(M_PI * x));
            InputWindow[i] = (float)window;
            OutputWindow[i] = (float)(window * output_volume);
        }
        // 计算其他参数
        OverSamplingRadian = (float)((2.0 * M_PI) / OverSampling);
        ExactTimeScale = (float)OutputHopSize / InputHopSize;
        RebuildParams = false;
    }

    // === 步骤2：检查缓冲区 ===
    if(InputBuffer.GetDataSize() < FrameSize * Channels)
        return psInputNotEnough;  // 输入不足
    if(OutputBuffer.GetFreeSize() < FrameSize * Channels)
        return psOutputFull;      // 输出满

    // === 步骤3：清零输出区域的最后OutputHopSize样本 ===
    // （用于重叠相加时的边界处理）
    {
        float *p1, *p2;
        size_t p1len, p2len;
        OutputBuffer.GetWritePointer(OutputHopSize * Channels, p1, p1len, p2,
                                     p2len, (FrameSize - OutputHopSize) * Channels);
        memset(p1, 0, p1len * sizeof(float));
        if(p2) memset(p2, 0, p2len * sizeof(float));
    }

    // === 步骤4：读取输入并加窗 ===
    {
        const float *p1, *p2;
        size_t p1len, p2len;
        InputBuffer.GetReadPointer(FrameSize * Channels, p1, p1len, p2, p2len);
        // 解交织并应用输入窗
        DeinterleaveApplyingWindow(AnalWork, p1, InputWindow, Channels, 0, p1len/Channels);
        if(p2)
            DeinterleaveApplyingWindow(AnalWork, p2, InputWindow + p1len/Channels,
                                       Channels, p1len/Channels, p2len/Channels);
    }

    // === 步骤5：逐声道处理 ===
    for(unsigned int ch = 0; ch < Channels; ch++) {
        ProcessCore(ch);  // 核心FFT处理
    }

    // === 步骤6：写入输出并加窗（重叠相加） ===
    {
        float *p1, *p2;
        size_t p1len, p2len;
        OutputBuffer.GetWritePointer(FrameSize * Channels, p1, p1len, p2, p2len);
        // 交织并应用输出窗（重叠相加）
        InterleaveOverlappingWindow(p1, SynthWork, OutputWindow, Channels, 0, p1len/Channels);
        if(p2)
            InterleaveOverlappingWindow(p2, SynthWork, OutputWindow + p1len/Channels,
                                        Channels, p1len/Channels, p2len/Channels);
    }

    // === 步骤7：更新缓冲区指针 ===
    OutputBuffer.AdvanceWritePos(OutputHopSize * Channels);
    InputBuffer.AdvanceReadPos(InputHopSize * Channels);

    return psNoError;
}
```

### 3.2 ProcessCore() —— FFT核心处理

```cpp
// 文件: cpp/core/sound/PhaseVocoderDSP.cpp 第434-634行（简化版）

void tRisaPhaseVocoderDSP::ProcessCore(int ch) {
    unsigned int framesize_d2 = FrameSize / 2;
    float *analwork = AnalWork[ch];
    float *synthwork = SynthWork[ch];

    // === 1. 正向FFT ===
    rdft(FrameSize, 1, analwork, FFTWorkIp, FFTWorkW);
    analwork[1] = 0.0;  // 清零奈奎斯特频率分量

    // === 2. 分析阶段：直角坐标→极坐标 ===
    for(unsigned int i = 0; i < framesize_d2; i++) {
        float re = analwork[i * 2];
        float im = analwork[i * 2 + 1];

        // 计算幅度和相位
        float mag = sqrt(re * re + im * im);      // 幅度 = √(re² + im²)
        float ang = VFast_arctan2(im, re);        // 相位 = atan2(im, re)

        // 计算相位差（与上一帧比较）
        float tmp = LastAnalPhase[ch][i] - ang;
        LastAnalPhase[ch][i] = ang;               // 保存当前相位

        // 去除预期相位推进（由于窗口重叠）
        tmp -= i * OverSamplingRadian;

        // 相位展开到 [-π, +π]
        tmp = WrapPi_F1(tmp);

        // 计算真实频率
        float freq = (i + tmp * OverSamplingRadianRecp) * FrequencyPerFilterBand;

        analwork[i * 2] = mag;      // 存储幅度
        analwork[i * 2 + 1] = freq; // 存储频率
    }

    // === 3. 频率轴重采样（变调） ===
    if(FrequencyScale != 1.0) {
        float FrequencyScale_rcp = 1.0f / FrequencyScale;
        for(unsigned int i = 0; i < framesize_d2; i++) {
            float fi = i * FrequencyScale_rcp;
            unsigned int index = (unsigned int)fi;
            float frac = fi - index;

            // 双线性插值
            if(index + 1 < framesize_d2) {
                synthwork[i * 2] = analwork[index * 2] +
                    frac * (analwork[index * 2 + 2] - analwork[index * 2]);
                synthwork[i * 2 + 1] = FrequencyScale *
                    (analwork[index * 2 + 1] +
                     frac * (analwork[index * 2 + 3] - analwork[index * 2 + 1]));
            } else {
                synthwork[i * 2] = 0.0;
                synthwork[i * 2 + 1] = 0.0;
            }
        }
    }

    // === 4. 合成阶段：极坐标→直角坐标 ===
    for(unsigned int i = 0; i < framesize_d2; i++) {
        float mag = synthwork[i * 2];
        float freq = synthwork[i * 2 + 1];

        // 计算相位增量
        float tmp = freq * FrequencyPerFilterBandRecp - (float)i;
        tmp = tmp * OverSamplingRadian;
        tmp += i * OverSamplingRadian;
        tmp *= ExactTimeScale;  // 时间缩放补偿

        // 累加相位
        LastSynthPhase[ch][i] -= tmp;
        float ang = LastSynthPhase[ch][i];

        // 极坐标→直角坐标
        float c, s;
        VFast_sincos(ang, s, c);
        synthwork[i * 2] = mag * c;
        synthwork[i * 2 + 1] = mag * s;
    }

    // === 5. 逆向FFT ===
    synthwork[1] = 0.0;
    rdft(FrameSize, -1, synthwork, FFTWorkIp, FFTWorkW);
}
```

### 3.3 算法流程图

```
输入信号 (时域)
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  分帧 + 加窗 (Vorbis I Window)                              │
│  • FrameSize = 4096 样本                                    │
│  • HopSize = 512 样本 (OverSampling = 8)                   │
│  • 75% 重叠                                                 │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  FFT (正向)                                                  │
│  • 时域 → 频域                                              │
│  • 输出: FrameSize/2 个复数 (re, im)                        │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  直角坐标 → 极坐标                                          │
│  • mag = √(re² + im²)                                       │
│  • ang = atan2(im, re)                                      │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  相位差计算 + 真实频率估计                                  │
│  • Δφ = 上次相位 - 当前相位                                │
│  • 去除预期相位推进: Δφ -= i × 2π/OverSampling             │
│  • 相位展开到 [-π, +π]                                      │
│  • 真实频率 = (i + Δφ/OverSamplingRadian) × FreqPerBand    │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  频率轴重采样 (变调)                                        │
│  • FrequencyScale > 1: 升调                                │
│  • FrequencyScale < 1: 降调                                │
│  • 双线性插值保持平滑                                       │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  相位累加 + 极坐标 → 直角坐标                               │
│  • 相位累加: LastPhase -= Δφ × ExactTimeScale              │
│  • re = mag × cos(phase)                                    │
│  • im = mag × sin(phase)                                    │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  IFFT (逆向)                                                │
│  • 频域 → 时域                                              │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  加窗 + 重叠相加 (Overlap-Add)                              │
│  • OutputHopSize = InputHopSize × TimeScale                 │
│  • TimeScale > 1: 变慢 (拉伸)                              │
│  • TimeScale < 1: 变快 (压缩)                              │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
输出信号 (时域)
```

---

## 4. TJS2脚本接口

### 4.1 PhaseVocoder类

```tjs
// TJS2脚本中使用PhaseVocoder
var pv = new PhaseVocoder();

// 设置参数
pv.window = 4096;   // 窗口大小 (64, 128, ..., 32768)
pv.overlap = 8;     // 重叠系数 (0, 2, 4, 8, 16, 32)
pv.pitch = 1.2;     // 音调 (1.2 = 升调约3个半音)
pv.time = 0.8;      // 时间 (0.8 = 加速25%)

// 连接到音频
waveSoundBuffer.setFilter(pv.interface);
```

### 4.2 常用参数组合

| 效果 | pitch | time | 说明 |
|------|-------|------|------|
| 快进（不变调） | 1.0 | 0.5 | 2倍速播放 |
| 慢放（不变调） | 1.0 | 2.0 | 0.5倍速播放 |
| 升调（不变速） | 1.5 | 1.0 | 升约7个半音 |
| 降调（不变速） | 0.75 | 1.0 | 降约5个半音 |
| 快进+降调 | 0.8 | 0.5 | 2倍速+降调 |

### 4.3 音调计算公式

```
半音数 = 12 × log₂(pitch)

例如:
  pitch = 2.0  → 12 × log₂(2) = 12 × 1 = +12半音 (升1个八度)
  pitch = 1.5  → 12 × log₂(1.5) ≈ 12 × 0.585 ≈ +7半音
  pitch = 0.5  → 12 × log₂(0.5) = 12 × (-1) = -12半音 (降1个八度)
```

---

## 5. 与WaveSegmentQueue的集成

### 5.1 Decode流程

```cpp
// 文件: cpp/core/sound/PhaseVocoderFilter.cpp 第295-376行

void tTJSNI_PhaseVocoder::Decode(void *dest, tjs_uint samples,
                                 tjs_uint &written,
                                 tTVPWaveSegmentQueue &segments) {
    // 创建PhaseVocoder（如果需要）
    if(!PhaseVocoder) {
        PhaseVocoder = new tRisaPhaseVocoderDSP(
            Window, InputFormat.SamplesPerSec, InputFormat.Channels);
        PhaseVocoder->SetFrequencyScale(Pitch);
        PhaseVocoder->SetTimeScale(Time);
        PhaseVocoder->SetOverSampling(Overlap);
    }

    size_t inputhopsize = PhaseVocoder->GetInputHopSize();
    size_t outputhopsize = PhaseVocoder->GetOutputHopSize();
    tTVPWaveSegmentQueue queue;

    float *dest_buf = (float *)dest;
    written = 0;

    while(samples > 0) {
        tRisaPhaseVocoderDSP::tStatus status;
        do {
            // 向输入缓冲填充数据
            size_t inputfree = PhaseVocoder->GetInputFreeSize();
            if(inputfree >= inputhopsize) {
                float *p1, *p2;
                size_t p1len, p2len;
                PhaseVocoder->GetInputBuffer(inputhopsize, p1, p1len, p2, p2len);
                
                tjs_uint filled = 0;
                Fill(p1, p1len, filled, InputSegments);
                if(p2) Fill(p2, p2len, filled, InputSegments);
                
                if(filled == 0) break;  // 输入结束
            }

            // 执行PhaseVocoder处理
            status = PhaseVocoder->Process();
            
            if(status == tRisaPhaseVocoderDSP::psNoError) {
                // 处理成功：从InputSegments取出inputhopsize
                // 缩放到outputhopsize后加入OutputSegments
                InputSegments.Dequeue(queue, inputhopsize);
                queue.Scale(outputhopsize);  // 关键：时间缩放
                OutputSegments.Enqueue(queue);
            }
        } while(status == tRisaPhaseVocoderDSP::psInputNotEnough);

        // 从PhaseVocoder输出缓冲读取数据
        size_t output_ready = PhaseVocoder->GetOutputReadySize();
        if(output_ready >= outputhopsize) {
            size_t copy_size = std::min((size_t)samples, outputhopsize);
            const float *p1, *p2;
            size_t p1len, p2len;
            PhaseVocoder->GetOutputBuffer(copy_size, p1, p1len, p2, p2len);
            
            memcpy(dest_buf, p1, p1len * sizeof(float) * OutputFormat.Channels);
            if(p2)
                memcpy(dest_buf + p1len * OutputFormat.Channels, p2,
                       p2len * sizeof(float) * OutputFormat.Channels);

            samples -= copy_size;
            written += copy_size;
            dest_buf += copy_size * OutputFormat.Channels;

            // 更新输出分段队列
            OutputSegments.Dequeue(queue, copy_size);
            segments.Enqueue(queue);
        } else {
            return;  // 无更多数据
        }
    }
}
```

### 5.2 SegmentQueue与TimeScale的关系

```
原始时间轴:
  ├────InputHopSize────┤  ← 每次消耗512样本
  
处理后时间轴:
  ├────OutputHopSize────┤  ← 每次输出512×TimeScale样本

例如 TimeScale = 2.0 (慢放):
  输入: [0, 512) → 输出: [0, 1024)
  输入长度 = 512, FilteredLength = 1024
  
Scale操作:
  InputSegments.Dequeue(queue, inputhopsize=512);
  queue.Scale(outputhopsize=1024);  // 拉伸到1024
  OutputSegments.Enqueue(queue);
```

---

## 6. 动手实践

### 实践1：简化版Phase Vocoder

```cpp
// simple_phase_vocoder.cpp
// 最简化的Phase Vocoder实现（仅变速，不变调）

#include <iostream>
#include <vector>
#include <cmath>
#include <complex>

const double PI = 3.14159265358979323846;

// 简单的DFT实现（实际应用中使用FFT）
void DFT(const std::vector<float>& in, std::vector<std::complex<float>>& out) {
    size_t N = in.size();
    out.resize(N);
    for (size_t k = 0; k < N; k++) {
        out[k] = 0;
        for (size_t n = 0; n < N; n++) {
            float angle = 2 * PI * k * n / N;
            out[k] += in[n] * std::complex<float>(cos(angle), -sin(angle));
        }
    }
}

void IDFT(const std::vector<std::complex<float>>& in, std::vector<float>& out) {
    size_t N = in.size();
    out.resize(N);
    for (size_t n = 0; n < N; n++) {
        std::complex<float> sum = 0;
        for (size_t k = 0; k < N; k++) {
            float angle = 2 * PI * k * n / N;
            sum += in[k] * std::complex<float>(cos(angle), sin(angle));
        }
        out[n] = sum.real() / N;
    }
}

// Hann窗函数
float HannWindow(int i, int N) {
    return 0.5f * (1.0f - cos(2.0f * PI * i / (N - 1)));
}

// 相位展开到 [-PI, PI]
float WrapPhase(float phase) {
    while (phase > PI) phase -= 2 * PI;
    while (phase < -PI) phase += 2 * PI;
    return phase;
}

class SimplePhaseVocoder {
    int frameSize;
    int hopSize;
    float timeScale;
    
    std::vector<float> lastAnalPhase;
    std::vector<float> lastSynthPhase;
    std::vector<float> outputAccum;
    int outputPos;
    
public:
    SimplePhaseVocoder(int fs = 1024, float ts = 1.0f) 
        : frameSize(fs), timeScale(ts) {
        hopSize = frameSize / 4;  // 75% 重叠
        lastAnalPhase.resize(frameSize, 0);
        lastSynthPhase.resize(frameSize, 0);
        outputAccum.resize(frameSize * 4, 0);
        outputPos = 0;
    }
    
    void Process(const std::vector<float>& input, std::vector<float>& output) {
        int outputHopSize = static_cast<int>(hopSize * timeScale);
        output.clear();
        
        for (size_t pos = 0; pos + frameSize <= input.size(); pos += hopSize) {
            // 1. 加窗
            std::vector<float> frame(frameSize);
            for (int i = 0; i < frameSize; i++) {
                frame[i] = input[pos + i] * HannWindow(i, frameSize);
            }
            
            // 2. FFT (使用简化DFT)
            std::vector<std::complex<float>> spectrum;
            DFT(frame, spectrum);
            
            // 3. 分析：计算幅度和相位
            std::vector<float> magnitude(frameSize);
            std::vector<float> frequency(frameSize);
            
            for (int k = 0; k < frameSize / 2; k++) {
                magnitude[k] = std::abs(spectrum[k]);
                float phase = std::arg(spectrum[k]);
                
                // 计算相位差
                float deltaPhi = phase - lastAnalPhase[k];
                lastAnalPhase[k] = phase;
                
                // 减去预期相位推进
                deltaPhi -= k * 2 * PI * hopSize / frameSize;
                deltaPhi = WrapPhase(deltaPhi);
                
                // 计算真实频率
                frequency[k] = k + deltaPhi * frameSize / (2 * PI * hopSize);
            }
            
            // 4. 合成：重建相位
            std::vector<std::complex<float>> synthSpectrum(frameSize);
            for (int k = 0; k < frameSize / 2; k++) {
                // 计算新相位
                float deltaPhi = frequency[k] * 2 * PI * outputHopSize / frameSize;
                lastSynthPhase[k] = WrapPhase(lastSynthPhase[k] + deltaPhi);
                
                // 极坐标→直角坐标
                synthSpectrum[k] = std::polar(magnitude[k], lastSynthPhase[k]);
                // 共轭对称
                if (k > 0 && k < frameSize / 2) {
                    synthSpectrum[frameSize - k] = std::conj(synthSpectrum[k]);
                }
            }
            
            // 5. IFFT
            std::vector<float> synthFrame;
            IDFT(synthSpectrum, synthFrame);
            
            // 6. 加窗 + 重叠相加
            for (int i = 0; i < frameSize; i++) {
                int outIdx = outputPos + i;
                if (outIdx < outputAccum.size()) {
                    outputAccum[outIdx] += synthFrame[i] * HannWindow(i, frameSize);
                }
            }
            
            // 输出
            for (int i = 0; i < outputHopSize && outputPos + i < outputAccum.size(); i++) {
                output.push_back(outputAccum[outputPos + i]);
            }
            
            outputPos += outputHopSize;
        }
    }
};

int main() {
    // 生成测试信号：440Hz正弦波
    const int sampleRate = 44100;
    const int duration = 1;  // 1秒
    const float freq = 440.0f;
    
    std::vector<float> input(sampleRate * duration);
    for (int i = 0; i < input.size(); i++) {
        input[i] = sin(2 * PI * freq * i / sampleRate);
    }
    
    std::cout << "原始信号: " << input.size() << " 样本\n";
    
    // 2倍慢放
    SimplePhaseVocoder pv(1024, 2.0f);
    std::vector<float> output;
    pv.Process(input, output);
    
    std::cout << "处理后信号: " << output.size() << " 样本\n";
    std::cout << "时间拉伸比例: " << (float)output.size() / input.size() << "\n";
    
    return 0;
}
```

**编译运行：**
```bash
g++ -std=c++17 -o simple_phase_vocoder simple_phase_vocoder.cpp -lm
./simple_phase_vocoder
```

---

## 7. 对照项目源码

### 7.1 相关文件

| 文件 | 行数 | 说明 |
|------|------|------|
| `cpp/core/sound/PhaseVocoderDSP.h` | 201 | DSP类声明 |
| `cpp/core/sound/PhaseVocoderDSP.cpp` | 875 | 核心算法实现 |
| `cpp/core/sound/PhaseVocoderFilter.h` | 112 | TJS2绑定声明 |
| `cpp/core/sound/PhaseVocoderFilter.cpp` | 377 | TJS2绑定实现 |
| `cpp/core/sound/RingBuffer.h` | 203 | 环形缓冲区模板 |
| `cpp/core/sound/MathAlgorithms.h` | 577 | 数学函数（快速sin/cos/atan2） |
| `cpp/core/sound/RealFFT.h/cpp` | ~500 | Real FFT实现 |

### 7.2 关键代码位置

| 功能 | 文件:行 |
|------|---------|
| Vorbis I窗函数 | PhaseVocoderDSP.cpp:317-322 |
| 快速atan2 | MathAlgorithms.h:33-49 |
| 快速sincos | MathAlgorithms.h:69-125 |
| 相位展开 | MathAlgorithms.h:132-140 |
| TJS2属性绑定 | PhaseVocoderFilter.cpp:55-125 |

---

## 8. 本节小结

- **Phase Vocoder** 通过FFT将时域信号转换到频域，独立控制时间和频率
- **核心流程**: 分帧→加窗→FFT→极坐标→相位差→变换→累加相位→IFFT→重叠相加
- **TimeScale** 控制OutputHopSize/InputHopSize比例，实现变速
- **FrequencyScale** 通过频率轴重采样实现变调
- **tTVPWaveSegmentQueue** 跟踪DSP处理前后的时间映射
- **TJS2脚本** 通过`PhaseVocoder`类的`pitch`和`time`属性控制效果

---

## 9. 练习题与答案

### 题目1：参数计算

如果`FrameSize=4096`，`OverSampling=8`，`TimeScale=1.5`，计算：
1. InputHopSize
2. OutputHopSize
3. 重叠百分比

<details>
<summary>查看答案</summary>

1. **InputHopSize** = FrameSize / OverSampling = 4096 / 8 = **512**
2. **OutputHopSize** = InputHopSize × TimeScale = 512 × 1.5 = 768，偶数对齐后 = **768**
3. **重叠百分比** = (FrameSize - InputHopSize) / FrameSize × 100%
   = (4096 - 512) / 4096 × 100% = **87.5%**

</details>

### 题目2：变调计算

用户想将音频升高5个半音，`pitch`属性应设置为多少？

<details>
<summary>查看答案</summary>

使用公式: `pitch = 2^(半音数/12)`

```
pitch = 2^(5/12) 
      = 2^0.4167
      ≈ 1.335
```

**答案: pitch ≈ 1.335**

验证:
```
半音数 = 12 × log₂(1.335) 
       = 12 × 0.4167 
       ≈ 5
```

</details>

### 题目3：代码理解

解释以下代码片段的作用：

```cpp
tmp -= i * OverSamplingRadian;
tmp = WrapPi_F1(tmp);
float freq = (i + tmp * OverSamplingRadianRecp) * FrequencyPerFilterBand;
```

<details>
<summary>查看答案</summary>

这三行代码实现了**真实频率估计**：

1. **`tmp -= i * OverSamplingRadian`**
   - `tmp`是当前帧与上一帧的相位差
   - 由于帧之间有重叠（OverSampling），每个频率bin的相位会有预期推进量
   - 预期推进量 = `i × 2π / OverSampling`（第i个bin推进i个周期的1/OverSampling）
   - 减去预期推进后，剩余的是"意外"相位变化

2. **`tmp = WrapPi_F1(tmp)`**
   - 将相位差展开到 [-π, +π] 范围
   - 避免2π的整数倍带来的歧义

3. **`freq = (i + tmp * OverSamplingRadianRecp) * FrequencyPerFilterBand`**
   - `tmp * OverSamplingRadianRecp` = 相位偏差转换为bin偏移量（-1.0到+1.0）
   - `i + 偏移量` = 真实的bin位置（非整数）
   - 乘以 `FrequencyPerFilterBand`（=采样率/帧大小）转换为Hz

**目的**: FFT的每个bin对应一个频带，但真实频率可能在bin中心之外。通过分析相位变化可以精确估计真实频率。

</details>

---

## 下一步

[05-实战-添加新音频格式/01-FLAC解码器实现](../05-实战-添加新音频格式/01-FLAC解码器实现.md) — 动手实现一个完整的FLAC音频解码器
