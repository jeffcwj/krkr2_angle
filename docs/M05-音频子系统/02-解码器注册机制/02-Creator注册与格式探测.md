# Creator 注册与格式探测

> **所属模块：** M05-音频子系统  
> **前置知识：** [01-tTVPWaveDecoder接口详解](./01-tTVPWaveDecoder接口详解.md)  
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 `tTVPWaveDecoderCreator` 工厂接口的设计模式
2. 掌握解码器注册机制的完整流程
3. 实现自定义解码器的工厂类并注册到系统
4. 理解格式探测的优先级与回退机制
5. 处理多格式支持与扩展名映射

---

## 工厂模式概述

### 为什么需要 Creator？

KiriKiri2 音频系统支持多种音频格式（WAV、OGG、Opus、MP3 等），每种格式需要不同的解码器实现。为了实现**格式透明**的音频播放，系统采用**抽象工厂模式（Abstract Factory Pattern）**：

```
┌─────────────────────────────────────────────────────────────────┐
│                     播放器层                                    │
│    WaveSoundBuffer.open("bgm.ogg")                             │
│         │                                                       │
│         ▼                                                       │
│    TVPCreateWaveDecoder("bgm.ogg")                             │
│         │                                                       │
└─────────┼───────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                     解码器管理器                                │
│    ┌─────────────────────────────────────────────────────────┐ │
│    │           Creators 注册表（后注册优先）                  │ │
│    ├─────────────────────────────────────────────────────────┤ │
│    │ [3] VorbisWaveDecoderCreator  ← 最后检查（先注册）       │ │
│    │ [2] RIFFWaveDecoderCreator                              │ │
│    │ [1] OpusWaveDecoderCreator                              │ │
│    │ [0] FFWaveDecoderCreator      ← 最先检查（后注册）       │ │
│    └─────────────────────────────────────────────────────────┘ │
│                                                                 │
│    逐个调用 Create()，返回首个非空解码器                        │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                     具体解码器                                  │
│    VorbisWaveDecoder / FFWaveDecoder / ...                     │
└─────────────────────────────────────────────────────────────────┘
```

### tTVPWaveDecoderCreator 接口

```cpp
// 文件: cpp/core/sound/WaveIntf.h 第 116-123 行
class tTVPWaveDecoderCreator {
public:
    virtual tTVPWaveDecoder *Create(const ttstr &storagename,
                                    const ttstr &extension) = 0;
    /*
        创建 tTVPWaveDecoder 实例
        参数:
            storagename - 完整存储路径（如 "bgm/title.ogg"）
            extension   - 小写扩展名（如 ".ogg"）
        返回值:
            成功时返回解码器指针
            失败时返回 nullptr（让下一个 Creator 尝试）
    */
};
```

**关键设计点：**

1. **参数分离**：同时传入完整路径和扩展名，Creator 可灵活选择检测方式
2. **静默失败**：返回 `nullptr` 表示"我不支持这个格式"，而不是抛出异常
3. **职责单一**：每个 Creator 只负责一种或一类格式

---

## 注册机制详解

### 注册表实现

```cpp
// 文件: cpp/core/sound/WaveIntf.cpp 第 710-729 行
bool TVPWaveDecoderManagerAvail = false;  // 管理器可用标志

struct tTVPWaveDecoderManager {
    std::vector<tTVPWaveDecoderCreator *> Creators;  // 注册表
    
    // 内置解码器实例（静态生命周期）
    tTVPWDC_RIFFWave RIFFWaveDecoderCreator;          // WAV
    VorbisWaveDecoderCreator vorbisWaveDecoderCreator; // OGG Vorbis
    FFWaveDecoderCreator ffWaveDecoderCreator;         // FFmpeg 通用
    OpusWaveDecoderCreator opusWaveDecoderCreator;     // Opus

    tTVPWaveDecoderManager() {
        TVPWaveDecoderManagerAvail = true;
        // 注册顺序决定探测优先级（后注册 = 先检查）
        TVPRegisterWaveDecoderCreator(&ffWaveDecoderCreator);    // 优先级 0
        TVPRegisterWaveDecoderCreator(&opusWaveDecoderCreator);  // 优先级 1
        TVPRegisterWaveDecoderCreator(&RIFFWaveDecoderCreator);  // 优先级 2
        TVPRegisterWaveDecoderCreator(&vorbisWaveDecoderCreator);// 优先级 3
    }

    ~tTVPWaveDecoderManager() { 
        TVPWaveDecoderManagerAvail = false; 
    }
} static TVPWaveDecoderManager;  // 全局单例
```

### 注册与注销函数

```cpp
// 文件: cpp/core/sound/WaveIntf.cpp 第 732-747 行

// 注册新的 Creator
void TVPRegisterWaveDecoderCreator(tTVPWaveDecoderCreator *d) {
    if (TVPWaveDecoderManagerAvail) {
        TVPWaveDecoderManager.Creators.push_back(d);
    }
}

// 注销已有的 Creator
void TVPUnregisterWaveDecoderCreator(tTVPWaveDecoderCreator *d) {
    if (TVPWaveDecoderManagerAvail) {
        auto it = std::find(
            TVPWaveDecoderManager.Creators.begin(),
            TVPWaveDecoderManager.Creators.end(),
            d
        );
        if (it != TVPWaveDecoderManager.Creators.end()) {
            TVPWaveDecoderManager.Creators.erase(it);
        }
    }
}
```

### 解码器创建流程

```cpp
// 文件: cpp/core/sound/WaveIntf.cpp 第 750-769 行
tTVPWaveDecoder *TVPCreateWaveDecoder(const ttstr &storagename) {
    // 检查管理器是否可用
    if (!TVPWaveDecoderManagerAvail)
        return nullptr;

    // 提取并转换扩展名为小写
    ttstr ext(TVPExtractStorageExt(storagename));
    ext.ToLowerCase();

    // 从后向前遍历（后注册的优先）
    tjs_int i = (tjs_int)(TVPWaveDecoderManager.Creators.size() - 1);
    for (; i >= 0; i--) {
        tTVPWaveDecoder *decoder;
        decoder = TVPWaveDecoderManager.Creators[i]->Create(storagename, ext);
        if (decoder)
            return decoder;  // 找到了！
    }

    // 所有 Creator 都失败了
    TVPThrowExceptionMessage(TVPUnknownWaveFormat, storagename);
    return nullptr;
}
```

**遍历顺序分析：**

```
注册顺序: FF → Opus → RIFF → Vorbis
存储顺序: [0]=FF, [1]=Opus, [2]=RIFF, [3]=Vorbis
检查顺序: [3]→[2]→[1]→[0]
         Vorbis→RIFF→Opus→FF

结论: 后注册的 Creator 优先级更高
```

---

## 格式探测策略

### 基于扩展名的快速探测

大多数 Creator 首先检查扩展名，快速判断是否值得尝试：

```cpp
// VorbisWaveDecoderCreator 示例
class VorbisWaveDecoderCreator : public tTVPWaveDecoderCreator {
public:
    tTVPWaveDecoder* Create(const ttstr& storagename, 
                            const ttstr& extension) override {
        // 扩展名快速过滤
        if (extension != ".ogg" && extension != ".oga") {
            return nullptr;  // 不是 OGG 文件，直接跳过
        }
        
        // 尝试打开并验证文件头
        try {
            return new VorbisWaveDecoder(storagename);
        } catch (...) {
            return nullptr;  // 静默失败
        }
    }
};
```

### 基于文件头的深度探测

当扩展名不可靠时（如通用 `.dat` 文件），需要读取文件头进行魔数验证：

```cpp
// WAV 格式魔数探测
class tTVPWDC_RIFFWave : public tTVPWaveDecoderCreator {
    tjs_uint8 riff_mark[4] = {'R', 'I', 'F', 'F'};
    tjs_uint8 wave_mark[4] = {'W', 'A', 'V', 'E'};
    
public:
    tTVPWaveDecoder* Create(const ttstr& storagename,
                            const ttstr& extension) override {
        // 扩展名提示（可选）
        if (extension != ".wav" && extension != ".wave") {
            // 不是 .wav 但也可以尝试（可能是改名的 WAV）
        }
        
        try {
            // 打开文件
            auto stream = TVPCreateStream(storagename, TJS_BS_READ);
            if (!stream) return nullptr;
            
            // 读取并验证 RIFF 头
            tjs_uint8 header[12];
            if (stream->Read(header, 12) != 12) {
                delete stream;
                return nullptr;
            }
            
            // 检查魔数: "RIFF....WAVE"
            if (memcmp(header, riff_mark, 4) != 0 ||
                memcmp(header + 8, wave_mark, 4) != 0) {
                delete stream;
                return nullptr;
            }
            
            // 魔数匹配，继续解析
            stream->SetPosition(0);  // 回退到开头
            return CreateRIFFWaveDecoder(stream);
            
        } catch (...) {
            return nullptr;
        }
    }
};
```

### 常见音频格式魔数

| 格式 | 魔数 (十六进制) | 偏移 | 说明 |
|------|-----------------|------|------|
| WAV  | `52 49 46 46` + `57 41 56 45` | 0, 8 | "RIFF" + "WAVE" |
| OGG  | `4F 67 67 53` | 0 | "OggS" |
| MP3  | `FF FB` 或 `49 44 33` | 0 | 帧同步 或 "ID3" |
| FLAC | `66 4C 61 43` | 0 | "fLaC" |
| Opus | `4F 67 67 53` + `4F 70 75 73` | 0, 28 | OGG 容器 + "Opus" |
| AAC  | `FF F1` 或 `FF F9` | 0 | ADTS 帧头 |

```cpp
// 魔数检测工具函数
bool CheckMagic(tTJSBinaryStream* stream, 
                const uint8_t* magic, 
                size_t length,
                size_t offset = 0) {
    // 保存当前位置
    tjs_uint64 savedPos = stream->GetPosition();
    
    // 定位到目标偏移
    stream->Seek(offset, TJS_BS_SEEK_SET);
    
    // 读取并比较
    std::vector<uint8_t> buffer(length);
    bool match = false;
    if (stream->Read(buffer.data(), length) == length) {
        match = (memcmp(buffer.data(), magic, length) == 0);
    }
    
    // 恢复位置
    stream->Seek(savedPos, TJS_BS_SEEK_SET);
    
    return match;
}

// 使用示例
const uint8_t OGG_MAGIC[] = {0x4F, 0x67, 0x67, 0x53};  // "OggS"
if (CheckMagic(stream, OGG_MAGIC, 4)) {
    // 这是一个 OGG 文件
}
```

---

## 实现自定义 Creator

### 完整示例：FLAC 解码器 Creator

```cpp
// flac_decoder_creator.h
#pragma once

#include "WaveIntf.h"
#include <FLAC/stream_decoder.h>

// 前向声明
class FLACWaveDecoder;

/**
 * FLAC 解码器工厂
 * 支持 .flac 和 .fla 扩展名
 */
class FLACWaveDecoderCreator : public tTVPWaveDecoderCreator {
private:
    // FLAC 魔数: "fLaC"
    static constexpr uint8_t FLAC_MAGIC[4] = {0x66, 0x4C, 0x61, 0x43};
    
public:
    tTVPWaveDecoder* Create(const ttstr& storagename,
                            const ttstr& extension) override {
        // 第一阶段：扩展名过滤
        bool extensionMatch = 
            (extension == TJS_W(".flac")) || 
            (extension == TJS_W(".fla"));
        
        // 如果扩展名不匹配，可以选择：
        // 1. 直接返回 nullptr（快速但可能漏检）
        // 2. 继续进行魔数检测（慢但准确）
        // 这里采用策略 1，因为 FLAC 文件通常有正确扩展名
        if (!extensionMatch) {
            return nullptr;
        }
        
        // 第二阶段：尝试打开文件
        tTJSBinaryStream* stream = nullptr;
        try {
            stream = TVPCreateStream(storagename, TJS_BS_READ);
            if (!stream) {
                return nullptr;
            }
            
            // 第三阶段：验证 FLAC 魔数
            uint8_t header[4];
            if (stream->Read(header, 4) != 4) {
                delete stream;
                return nullptr;
            }
            
            if (memcmp(header, FLAC_MAGIC, 4) != 0) {
                // 魔数不匹配，可能是误命名的文件
                delete stream;
                return nullptr;
            }
            
            // 第四阶段：创建解码器
            stream->Seek(0, TJS_BS_SEEK_SET);  // 回退到开头
            
            FLACWaveDecoder* decoder = new FLACWaveDecoder(stream);
            // 注意：stream 的所有权转移给 decoder
            
            return decoder;
            
        } catch (const std::exception& e) {
            // 记录错误（可选）
            // TVPAddLog(TJS_W("FLAC 解码器创建失败: ") + ttstr(e.what()));
            
            if (stream) delete stream;
            return nullptr;
            
        } catch (...) {
            if (stream) delete stream;
            return nullptr;
        }
    }
};

// 全局实例（在某个 .cpp 中定义）
// FLACWaveDecoderCreator gFLACWaveDecoderCreator;
```

### 注册时机选择

```cpp
// 方法 1：静态初始化（最早，推荐用于内置解码器）
namespace {
    struct FLACDecoderRegistrar {
        FLACWaveDecoderCreator creator;
        FLACDecoderRegistrar() {
            TVPRegisterWaveDecoderCreator(&creator);
        }
        ~FLACDecoderRegistrar() {
            TVPUnregisterWaveDecoderCreator(&creator);
        }
    } gFLACRegistrar;
}

// 方法 2：显式初始化函数（灵活，适合插件）
static FLACWaveDecoderCreator* gFLACCreator = nullptr;

void InitFLACDecoder() {
    if (!gFLACCreator) {
        gFLACCreator = new FLACWaveDecoderCreator();
        TVPRegisterWaveDecoderCreator(gFLACCreator);
    }
}

void ShutdownFLACDecoder() {
    if (gFLACCreator) {
        TVPUnregisterWaveDecoderCreator(gFLACCreator);
        delete gFLACCreator;
        gFLACCreator = nullptr;
    }
}

// 方法 3：使用 KiriKiri 插件系统（适合外部插件）
// 在 NCB_PRE_REGIST_CALLBACK 中调用 TVPRegisterWaveDecoderCreator
```

---

## 优先级与回退机制

### 设计考量

当多个 Creator 可能处理同一格式时，需要考虑优先级：

```cpp
// 场景：同时支持 Vorbis 和 FFmpeg 解码 OGG 文件
// 问题：谁应该优先？

// KiriKiri2 的策略：专用解码器优先于通用解码器
// 注册顺序（决定优先级）：
// 1. FFWaveDecoderCreator   → 优先级最低（通用后备）
// 2. OpusWaveDecoderCreator → 专用 Opus
// 3. RIFFWaveDecoderCreator → 专用 WAV
// 4. VorbisWaveDecoderCreator → 专用 Vorbis（优先级最高）

// 遍历顺序：[3]→[2]→[1]→[0]
// 效果：Vorbis 文件会先被 VorbisDecoder 处理
//      如果 Vorbis 失败，才会尝试 FFmpeg
```

### 回退链实现

```cpp
// 更精细的优先级控制
class SmartDecoderCreator : public tTVPWaveDecoderCreator {
private:
    int priority_;  // 显式优先级
    std::string supportedExtensions_;
    
public:
    SmartDecoderCreator(int priority, const std::string& exts)
        : priority_(priority), supportedExtensions_(exts) {}
    
    int GetPriority() const { return priority_; }
    
    tTVPWaveDecoder* Create(const ttstr& storagename,
                            const ttstr& extension) override {
        // 检查扩展名是否在支持列表中
        std::string ext = ttstr_to_string(extension);
        if (supportedExtensions_.find(ext) == std::string::npos) {
            return nullptr;
        }
        
        // 尝试创建解码器
        return TryCreate(storagename);
    }
    
protected:
    virtual tTVPWaveDecoder* TryCreate(const ttstr& storagename) = 0;
};

// 按优先级排序的注册函数
void RegisterWithPriority(SmartDecoderCreator* creator) {
    auto& creators = TVPWaveDecoderManager.Creators;
    
    // 找到插入位置（保持优先级降序）
    auto it = std::lower_bound(
        creators.begin(), creators.end(), creator,
        [](auto* a, auto* b) {
            auto* sa = dynamic_cast<SmartDecoderCreator*>(a);
            auto* sb = dynamic_cast<SmartDecoderCreator*>(b);
            int pa = sa ? sa->GetPriority() : 0;
            int pb = sb ? sb->GetPriority() : 0;
            return pa > pb;  // 高优先级在前
        }
    );
    
    creators.insert(it, creator);
}
```

---

## 对照项目源码

### FFWaveDecoderCreator 实现

FFmpeg 解码器作为通用后备，支持几乎所有音频格式：

**相关文件：**
- `cpp/core/sound/FFWaveDecoder.h`
- `cpp/core/sound/FFWaveDecoder.cpp`

```cpp
// 文件: cpp/core/sound/FFWaveDecoder.h（简化版）
class FFWaveDecoderCreator : public tTVPWaveDecoderCreator {
public:
    tTVPWaveDecoder* Create(const ttstr& storagename,
                            const ttstr& extension) override {
        // FFmpeg 不基于扩展名过滤，尝试打开任何文件
        try {
            // FFWaveDecoder 构造函数会：
            // 1. 调用 avformat_open_input 打开文件
            // 2. 调用 avformat_find_stream_info 探测格式
            // 3. 找到音频流并初始化解码器
            // 任何步骤失败都会抛出异常
            return new FFWaveDecoder(storagename);
        } catch (...) {
            return nullptr;
        }
    }
};
```

### VorbisWaveDecoderCreator 实现

Vorbis 解码器专门处理 OGG Vorbis 格式：

**相关文件：**
- `cpp/core/sound/VorbisWaveDecoder.h`
- `cpp/core/sound/VorbisWaveDecoder.cpp`

```cpp
// 文件: cpp/core/sound/VorbisWaveDecoder.h（简化版）
class VorbisWaveDecoderCreator : public tTVPWaveDecoderCreator {
public:
    tTVPWaveDecoder* Create(const ttstr& storagename,
                            const ttstr& extension) override {
        // 只处理 .ogg 文件
        if (extension != TJS_W(".ogg")) {
            return nullptr;
        }
        
        try {
            return new VorbisWaveDecoder(storagename);
        } catch (...) {
            return nullptr;
        }
    }
};

// Opus 解码器类似
class OpusWaveDecoderCreator : public tTVPWaveDecoderCreator {
public:
    tTVPWaveDecoder* Create(const ttstr& storagename,
                            const ttstr& extension) override {
        // 只处理 .opus 文件
        if (extension != TJS_W(".opus")) {
            return nullptr;
        }
        
        try {
            return new OpusWaveDecoder(storagename);
        } catch (...) {
            return nullptr;
        }
    }
};
```

---

## 动手实践

### 实验 1：模拟完整的 Creator 注册与格式探测链

```cpp
#include <cstdio>
#include <cstring>
#include <functional>
#include <memory>
#include <string>
#include <vector>

// 简化版 Creator 接口
class IWaveDecoderCreator {
public:
    virtual ~IWaveDecoderCreator() = default;
    // 检查是否支持该扩展名
    virtual bool CanHandle(const std::string& ext) const = 0;
    // 尝试用文件头魔数（magic bytes）探测格式
    virtual bool ProbeFormat(const uint8_t* header, size_t len) const = 0;
    // 创建解码器实例
    virtual std::string GetName() const = 0;
};

// WAV Creator：支持 .wav 扩展名和 RIFF 魔数
class WavCreator : public IWaveDecoderCreator {
public:
    bool CanHandle(const std::string& ext) const override {
        return ext == ".wav" || ext == ".wave";
    }
    bool ProbeFormat(const uint8_t* header, size_t len) const override {
        // WAV 文件以 "RIFF" 开头，偏移 8 处为 "WAVE"
        if (len < 12) return false;
        return memcmp(header, "RIFF", 4) == 0 &&
               memcmp(header + 8, "WAVE", 4) == 0;
    }
    std::string GetName() const override { return "WAV"; }
};

// OGG Creator：支持 .ogg 扩展名和 OggS 魔数
class OggCreator : public IWaveDecoderCreator {
public:
    bool CanHandle(const std::string& ext) const override {
        return ext == ".ogg" || ext == ".oga";
    }
    bool ProbeFormat(const uint8_t* header, size_t len) const override {
        if (len < 4) return false;
        return memcmp(header, "OggS", 4) == 0;
    }
    std::string GetName() const override { return "Vorbis/Opus"; }
};

// 注册表：管理所有 Creator
class CreatorRegistry {
    std::vector<std::unique_ptr<IWaveDecoderCreator>> creators_;
public:
    void Register(std::unique_ptr<IWaveDecoderCreator> c) {
        printf("注册解码器 Creator: %s\n", c->GetName().c_str());
        creators_.push_back(std::move(c));
    }

    // 策略 1：按扩展名匹配
    IWaveDecoderCreator* FindByExtension(const std::string& ext) {
        for (auto& c : creators_) {
            if (c->CanHandle(ext)) return c.get();
        }
        return nullptr;
    }

    // 策略 2：按文件头魔数探测
    IWaveDecoderCreator* FindByProbe(const uint8_t* header, size_t len) {
        for (auto& c : creators_) {
            if (c->ProbeFormat(header, len)) return c.get();
        }
        return nullptr;
    }

    // 完整探测流程：先按扩展名，失败则按魔数
    IWaveDecoderCreator* Resolve(const std::string& ext,
                                  const uint8_t* header, size_t len) {
        // 第一轮：扩展名匹配
        auto* c = FindByExtension(ext);
        if (c) {
            printf("  扩展名 '%s' 匹配 -> %s\n",
                   ext.c_str(), c->GetName().c_str());
            return c;
        }

        // 第二轮：魔数探测（处理扩展名缺失或错误的情况）
        c = FindByProbe(header, len);
        if (c) {
            printf("  魔数探测匹配 -> %s\n", c->GetName().c_str());
            return c;
        }

        printf("  未找到匹配的解码器\n");
        return nullptr;
    }
};

int main() {
    CreatorRegistry registry;
    registry.Register(std::make_unique<WavCreator>());
    registry.Register(std::make_unique<OggCreator>());

    // 模拟文件头
    uint8_t wavHeader[] = {'R','I','F','F', 0,0,0,0, 'W','A','V','E'};
    uint8_t oggHeader[] = {'O','g','g','S', 0,0,0,0, 0,0,0,0};
    uint8_t unknown[]   = {0x89,'P','N','G', 0,0,0,0, 0,0,0,0};

    printf("\n--- 测试 1: 正常扩展名匹配 ---\n");
    registry.Resolve(".wav", wavHeader, 12);

    printf("\n--- 测试 2: 扩展名错误，靠魔数探测 ---\n");
    registry.Resolve(".mp3", oggHeader, 12);  // 扩展名是 mp3 但内容是 OGG

    printf("\n--- 测试 3: 完全未知格式 ---\n");
    registry.Resolve(".png", unknown, 12);

    return 0;
}
```

**预期输出**：
```
注册解码器 Creator: WAV
注册解码器 Creator: Vorbis/Opus

--- 测试 1: 正常扩展名匹配 ---
  扩展名 '.wav' 匹配 -> WAV

--- 测试 2: 扩展名错误，靠魔数探测 ---
  魔数探测匹配 -> Vorbis/Opus

--- 测试 3: 完全未知格式 ---
  未找到匹配的解码器
```

### 实验 2：测试 Creator 注册顺序的优先级影响

在注册表中先注册一个"通配 Creator"（什么格式都声称能处理），再注册特定格式的 Creator，观察优先级行为：

```cpp
// 通配 Creator：声称支持所有扩展名
class FallbackCreator : public IWaveDecoderCreator {
public:
    bool CanHandle(const std::string& ext) const override {
        return true;  // 什么都能处理
    }
    bool ProbeFormat(const uint8_t* h, size_t l) const override {
        return true;
    }
    std::string GetName() const override { return "FFmpeg-Fallback"; }
};

// 测试注册顺序影响
void TestPriority() {
    // 场景 A：先注册特定格式，再注册通配
    CreatorRegistry regA;
    regA.Register(std::make_unique<OggCreator>());
    regA.Register(std::make_unique<FallbackCreator>());
    printf("场景A (.ogg): ");
    auto* c = regA.FindByExtension(".ogg");
    printf("%s\n", c ? c->GetName().c_str() : "null");
    // 输出: Vorbis/Opus（特定优先）

    // 场景 B：先注册通配，再注册特定格式
    CreatorRegistry regB;
    regB.Register(std::make_unique<FallbackCreator>());
    regB.Register(std::make_unique<OggCreator>());
    printf("场景B (.ogg): ");
    c = regB.FindByExtension(".ogg");
    printf("%s\n", c ? c->GetName().c_str() : "null");
    // 输出: FFmpeg-Fallback（通配抢先匹配）
}
```

> 💡 这就是 KrKr2 中 FFmpeg 解码器放在 Creator 链末尾的原因——它几乎能解码任何格式，如果放在前面会"吃掉"所有文件，导致专用解码器永远不会被使用。

---

## 常见错误与排查

### 错误 1：Creator 生命周期管理不当

```cpp
// 错误：使用局部变量注册
void RegisterMyDecoder() {
    MyDecoderCreator creator;  // 局部变量
    TVPRegisterWaveDecoderCreator(&creator);
}  // creator 被销毁，但指针仍在注册表中！

// 正确：使用静态或堆分配
static MyDecoderCreator gMyCreator;  // 静态生命周期

void RegisterMyDecoder() {
    TVPRegisterWaveDecoderCreator(&gMyCreator);
}
```

### 错误 2：Create() 抛出异常

```cpp
// 错误：让异常传播出去
tTVPWaveDecoder* Create(const ttstr& name, const ttstr& ext) override {
    return new MyDecoder(name);  // 可能抛出异常
}

// 正确：捕获所有异常
tTVPWaveDecoder* Create(const ttstr& name, const ttstr& ext) override {
    try {
        return new MyDecoder(name);
    } catch (...) {
        return nullptr;  // 静默失败，让其他 Creator 尝试
    }
}
```

### 错误 3：扩展名比较忽略大小写

```cpp
// 错误：区分大小写的比较
if (extension == ".OGG") {  // 永远不匹配，因为 extension 已转小写
    return new OggDecoder(name);
}

// 正确：使用小写比较
if (extension == ".ogg") {  // TVPCreateWaveDecoder 已将 extension 转为小写
    return new OggDecoder(name);
}
```

### 错误 4：忘记注销 Creator

```cpp
// 错误：只注册不注销
void PluginInit() {
    gMyCreator = new MyCreator();
    TVPRegisterWaveDecoderCreator(gMyCreator);
}
// 程序退出时，gMyCreator 泄漏

// 正确：配对注册注销
void PluginInit() {
    gMyCreator = new MyCreator();
    TVPRegisterWaveDecoderCreator(gMyCreator);
}

void PluginShutdown() {
    TVPUnregisterWaveDecoderCreator(gMyCreator);
    delete gMyCreator;
    gMyCreator = nullptr;
}
```

---

## 本节小结

- `tTVPWaveDecoderCreator` 是解码器工厂的抽象接口，`Create()` 方法返回解码器或 `nullptr`
- 注册表是 `std::vector`，**后注册的 Creator 优先级更高**（从后向前遍历）
- 格式探测通常分两阶段：扩展名快速过滤 + 魔数深度验证
- 内置解码器注册顺序：FFmpeg → Opus → RIFF → Vorbis，Vorbis 优先级最高
- Creator 必须静默失败（返回 `nullptr`），不能抛出异常
- 实现自定义 Creator 时需注意生命周期管理和正确的注销

---

## 练习题与答案

### 题目 1：分析优先级

假设注册了以下 Creator（按顺序）：
1. `GenericCreator` — 支持所有格式
2. `MP3Creator` — 只支持 .mp3
3. `OGGCreator` — 只支持 .ogg

问：打开 `music.ogg` 时，哪个 Creator 会被首先调用？

<details>
<summary>查看答案</summary>

**答案：OGGCreator**

```cpp
// 存储顺序: [0]=Generic, [1]=MP3, [2]=OGG
// 遍历顺序: [2]→[1]→[0]

// 调用顺序:
// 1. OGGCreator.Create("music.ogg", ".ogg")
//    → 扩展名匹配，尝试打开
//    → 成功则返回解码器，不再继续
//
// 2. 如果 OGGCreator 失败:
//    MP3Creator.Create("music.ogg", ".ogg")
//    → 扩展名不匹配 (.ogg != .mp3)，返回 nullptr
//
// 3. 继续尝试:
//    GenericCreator.Create("music.ogg", ".ogg")
//    → 支持所有格式，尝试打开

// 结论: 后注册的 Creator 优先，即 OGGCreator 首先被调用
```

</details>

### 题目 2：实现扩展名映射

编写一个工具函数，将常见音频扩展名映射到格式名称：

```cpp
std::string GetFormatName(const ttstr& extension);
// ".mp3" → "MPEG Audio Layer 3"
// ".ogg" → "Ogg Vorbis"
// ".wav" → "RIFF WAVE"
// ...
```

<details>
<summary>查看答案</summary>

```cpp
#include <string>
#include <unordered_map>

// 简化的 ttstr 转 std::string
std::string ttstr_to_string(const ttstr& s) {
    // 假设 ttstr 有 c_str() 方法返回宽字符
    std::wstring ws(s.c_str());
    return std::string(ws.begin(), ws.end());
}

std::string GetFormatName(const ttstr& extension) {
    static const std::unordered_map<std::string, std::string> formatMap = {
        // 无损格式
        {".wav",  "RIFF WAVE"},
        {".wave", "RIFF WAVE"},
        {".flac", "Free Lossless Audio Codec"},
        {".fla",  "Free Lossless Audio Codec"},
        {".ape",  "Monkey's Audio"},
        {".alac", "Apple Lossless Audio Codec"},
        {".wv",   "WavPack"},
        
        // 有损格式
        {".mp3",  "MPEG Audio Layer 3"},
        {".ogg",  "Ogg Vorbis"},
        {".oga",  "Ogg Audio"},
        {".opus", "Opus Audio"},
        {".aac",  "Advanced Audio Coding"},
        {".m4a",  "MPEG-4 Audio"},
        {".wma",  "Windows Media Audio"},
        
        // 模块音乐
        {".mod",  "ProTracker Module"},
        {".xm",   "FastTracker 2 Extended Module"},
        {".it",   "Impulse Tracker Module"},
        {".s3m",  "Scream Tracker 3 Module"},
        
        // 其他
        {".mid",  "MIDI Sequence"},
        {".midi", "MIDI Sequence"},
        {".aiff", "Audio Interchange File Format"},
        {".aif",  "Audio Interchange File Format"},
    };
    
    std::string ext = ttstr_to_string(extension);
    
    // 转小写
    for (char& c : ext) {
        c = std::tolower(c);
    }
    
    auto it = formatMap.find(ext);
    if (it != formatMap.end()) {
        return it->second;
    }
    
    return "Unknown Audio Format";
}

// 测试
#include <iostream>

int main() {
    // 模拟 ttstr（实际项目中使用真正的 ttstr）
    struct MockTtstr {
        std::wstring str;
        const wchar_t* c_str() const { return str.c_str(); }
    };
    
    MockTtstr ext1{L".mp3"};
    MockTtstr ext2{L".OGG"};  // 大写
    MockTtstr ext3{L".xyz"};  // 未知
    
    auto convert = [](const MockTtstr& s) {
        std::wstring ws(s.c_str());
        std::string result(ws.begin(), ws.end());
        for (char& c : result) c = std::tolower(c);
        return result;
    };
    
    std::cout << convert(ext1) << " → " << GetFormatName(ext1) << std::endl;
    std::cout << convert(ext2) << " → " << GetFormatName(ext2) << std::endl;
    std::cout << convert(ext3) << " → " << GetFormatName(ext3) << std::endl;
    
    return 0;
}
// 输出:
// .mp3 → MPEG Audio Layer 3
// .ogg → Ogg Vorbis
// .xyz → Unknown Audio Format
```

</details>

### 题目 3：实现多格式 Creator

实现一个支持多种扩展名的 Creator 基类：

```cpp
class MultiExtensionCreator : public tTVPWaveDecoderCreator {
    // 支持: .mp3, .mp2, .mp1
    // 子类只需实现 TryCreate()
};
```

<details>
<summary>查看答案</summary>

```cpp
#include <vector>
#include <string>
#include <algorithm>

// 简化类型定义
using ttstr = std::wstring;
using tjs_uint = unsigned int;

class tTVPWaveDecoder {
public:
    virtual ~tTVPWaveDecoder() = default;
};

class tTVPWaveDecoderCreator {
public:
    virtual ~tTVPWaveDecoderCreator() = default;
    virtual tTVPWaveDecoder* Create(const ttstr& storagename,
                                    const ttstr& extension) = 0;
};

/**
 * 多扩展名 Creator 基类
 * 子类只需指定支持的扩展名列表并实现 TryCreate()
 */
class MultiExtensionCreator : public tTVPWaveDecoderCreator {
protected:
    std::vector<std::wstring> supportedExtensions_;
    
public:
    /**
     * 添加支持的扩展名
     * @param ext 扩展名（如 ".mp3"）
     */
    void AddExtension(const std::wstring& ext) {
        // 确保小写
        std::wstring lower = ext;
        std::transform(lower.begin(), lower.end(), lower.begin(), ::towlower);
        supportedExtensions_.push_back(lower);
    }
    
    /**
     * 检查是否支持指定扩展名
     */
    bool SupportsExtension(const ttstr& extension) const {
        // extension 已是小写
        return std::find(supportedExtensions_.begin(),
                         supportedExtensions_.end(),
                         extension) != supportedExtensions_.end();
    }
    
    tTVPWaveDecoder* Create(const ttstr& storagename,
                            const ttstr& extension) override final {
        // 扩展名过滤
        if (!SupportsExtension(extension)) {
            return nullptr;
        }
        
        // 委托给子类实现
        try {
            return TryCreate(storagename, extension);
        } catch (...) {
            return nullptr;  // 静默失败
        }
    }
    
protected:
    /**
     * 子类实现：尝试创建解码器
     * 只有扩展名匹配时才会被调用
     * @param storagename 文件路径
     * @param extension 小写扩展名
     * @return 解码器指针，失败时抛异常或返回 nullptr
     */
    virtual tTVPWaveDecoder* TryCreate(const ttstr& storagename,
                                       const ttstr& extension) = 0;
};

// 示例：MP3 系列 Creator
class MPEGAudioCreator : public MultiExtensionCreator {
public:
    MPEGAudioCreator() {
        AddExtension(L".mp3");
        AddExtension(L".mp2");
        AddExtension(L".mp1");
        AddExtension(L".mpga");
    }
    
protected:
    tTVPWaveDecoder* TryCreate(const ttstr& storagename,
                               const ttstr& extension) override {
        // 这里创建 MPEG 音频解码器
        // return new MPEGAudioDecoder(storagename);
        return nullptr;  // 示例返回 nullptr
    }
};

// 示例：Ogg 系列 Creator
class OggAudioCreator : public MultiExtensionCreator {
public:
    OggAudioCreator() {
        AddExtension(L".ogg");
        AddExtension(L".oga");
        AddExtension(L".ogx");
    }
    
protected:
    tTVPWaveDecoder* TryCreate(const ttstr& storagename,
                               const ttstr& extension) override {
        // 根据实际内容判断是 Vorbis 还是其他编码
        // return new OggDecoder(storagename);
        return nullptr;
    }
};

// 测试
#include <iostream>

int main() {
    MPEGAudioCreator mpegCreator;
    OggAudioCreator oggCreator;
    
    // 测试扩展名支持
    std::wcout << L".mp3 supported by MPEG: " 
               << mpegCreator.SupportsExtension(L".mp3") << std::endl;
    std::wcout << L".ogg supported by MPEG: "
               << mpegCreator.SupportsExtension(L".ogg") << std::endl;
    std::wcout << L".ogg supported by OGG: "
               << oggCreator.SupportsExtension(L".ogg") << std::endl;
    
    // 测试 Create（会返回 nullptr 因为 TryCreate 未实现）
    auto* decoder = mpegCreator.Create(L"test.mp3", L".mp3");
    std::wcout << L"Decoder created: " << (decoder ? L"yes" : L"no") << std::endl;
    
    return 0;
}
// 输出:
// .mp3 supported by MPEG: 1
// .ogg supported by MPEG: 0
// .ogg supported by OGG: 1
// Decoder created: no
```

</details>

---

## 下一步

[03-Vorbis与FFmpeg解码器实现](./03-Vorbis与FFmpeg解码器实现.md) — 深入分析两种主流解码器的具体实现
