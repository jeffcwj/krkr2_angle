# WaveLoopManager循环管理

> **所属模块：** M05-音频子系统
> **前置知识：** [01-tTVPWaveDecoder接口详解](../02-解码器注册机制/01-tTVPWaveDecoder接口详解.md)、[03-音量控制与Android-Oboe后端](../03-OpenAL混音器/03-音量控制与Android-Oboe后端.md)
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：
1. 理解KrKr2循环点管理系统的设计架构与数据结构
2. 掌握Link（跳转链接）与Label（标签）的工作机制
3. 理解条件跳转与标志变量的表达式语法
4. 实现50ms平滑交叉渐变（Crossfade）循环
5. 编写自定义循环点配置文件

---

## 1. 循环管理系统概述

### 1.1 为什么需要专门的循环管理器？

在视觉小说和游戏中，BGM循环不是简单的"播放到结尾后从头开始"：

| 需求 | 简单循环 | WaveLoopManager |
|------|----------|-----------------|
| 无缝循环 | 有点击声 | 50ms交叉渐变 |
| 循环点控制 | 不支持 | 任意位置跳转 |
| 多循环段 | 不支持 | 多个Link定义 |
| 条件循环 | 不支持 | 基于Flag的条件跳转 |
| 标签事件 | 不支持 | Label触发脚本回调 |
| 动态修改 | 不支持 | 运行时修改Flag |

### 1.2 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    TJS2 脚本层                                   │
│   waveSoundBuffer.loopStart = 44100;  // 设置循环起点            │
│   waveSoundBuffer.loopEnd = 441000;   // 设置循环终点            │
│   waveSoundBuffer.flags[0] = 1;       // 设置条件标志            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              tTVPWaveLoopManager                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Links        │  │ Labels       │  │ Flags[16]    │          │
│  │ (跳转列表)   │  │ (标签列表)   │  │ (条件标志)   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                              │                                   │
│                    ┌─────────┴─────────┐                        │
│                    │ Decode() 核心逻辑 │                        │
│                    │ - 检查最近Link    │                        │
│                    │ - 评估条件        │                        │
│                    │ - 执行Crossfade   │                        │
│                    │ - 触发Label       │                        │
│                    └───────────────────┘                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              tTVPWaveDecoder                                     │
│              (底层解码器 - Vorbis/FFmpeg等)                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心数据结构

### 2.1 tTVPWaveLoopLink —— 跳转链接

```cpp
// 文件: cpp/core/sound/WaveLoopManager.h 第82-139行

// 条件类型枚举
enum tTVPWaveLoopLinkCondition {
    llcNone,           // 无条件（总是跳转）
    llcEqual,          // Flag[CondVar] == RefValue
    llcNotEqual,       // Flag[CondVar] != RefValue
    llcGreater,        // Flag[CondVar] >  RefValue
    llcGreaterOrEqual, // Flag[CondVar] >= RefValue
    llcLesser,         // Flag[CondVar] <  RefValue
    llcLesserOrEqual   // Flag[CondVar] <= RefValue
};

// 跳转链接结构
struct tTVPWaveLoopLink {
    tjs_int64 From;                     // 跳转起点（样本位置）
    tjs_int64 To;                       // 跳转终点（样本位置）
    bool Smooth;                        // 是否使用平滑渐变
    tTVPWaveLoopLinkCondition Condition;// 跳转条件
    tjs_int RefValue;                   // 条件比较值
    tjs_int CondVar;                    // 条件变量索引（0-15）

    tTVPWaveLoopLink() {
        From = To = 0;
        Smooth = false;
        Condition = llcNone;
        RefValue = CondVar = 0;
    }
};
```

### 2.2 tTVPWaveLabel —— 标签

```cpp
// 文件: cpp/core/sound/WaveSegmentQueue.h 第47-107行

struct tTVPWaveLabel {
    tjs_int64 Position;  // 标签位置（样本数）
    ttstr Name;          // 标签名称
    tjs_int Offset;      // 解码偏移（由Decode填充）

    tTVPWaveLabel() {
        Position = 0;
        Offset = 0;
    }

    tTVPWaveLabel(tjs_int64 position, const ttstr &name, tjs_int offset) 
        : Position(position), Name(name), Offset(offset) {}
};
```

### 2.3 常量定义

```cpp
// 文件: cpp/core/sound/WaveLoopManager.h 第21-28行

#define TVP_WL_MAX_FLAGS 16       // 最多16个条件标志
#define TVP_WL_MAX_FLAG_VALUE 9999 // 标志最大值
#define TVP_WL_SMOOTH_TIME 50     // 交叉渐变时间（毫秒）
#define TVP_WL_SMOOTH_TIME_HALF 25 // 渐变半时间
#define TVP_WL_MAX_ID_LEN 16      // 标签ID最大长度
```

---

## 3. 核心解码逻辑

### 3.1 Decode() 函数流程

```cpp
// 文件: cpp/core/sound/WaveLoopManager.cpp 第285-494行

void tTVPWaveLoopManager::Decode(void *dest, tjs_uint samples,
                                 tjs_uint &written,
                                 tTVPWaveSegmentQueue &segments) {
    // 加锁保护
    volatile tTJSCriticalSectionHolder CS(DataCS);
    
    written = 0;
    tjs_uint8 *d = (tjs_uint8 *)dest;
    tjs_int give_up_count = 0;  // 防止无限循环
    std::deque<tTVPWaveLabel> labels;

    while(written != samples) {
        labels.clear();
        
        // 步骤1: 查找最近的Link事件
        tTVPWaveLoopLink link;
        if(!IgnoreLinks && GetNearestEvent(Position, link, false)) {
            
            // 步骤2: 如果当前位置就是跳转点
            if(link.From == Position) {
                give_up_count++;
                if(give_up_count >= TVPWaveLoopLinkGiveUpCount)
                    break;  // 防止递归Link导致的无限循环
                
                Position = link.To;  // 执行跳转
                if(!CrossFadeSamples)
                    Decoder->SetPosition(Position);
                continue;
            }
            
            // 步骤3: 处理平滑渐变
            if(link.Smooth) {
                // 准备交叉渐变...（详见3.2节）
            }
        }
        
        // 步骤4: 计算本次解码量
        tjs_int one_unit = /* 到下一事件的样本数 */;
        
        // 步骤5: 处理经过的标签
        GetLabelAt(Position, Position + one_unit, labels);
        for(auto& label : labels) {
            if(label.Name.c_str()[0] == ':') {
                EvalLabelExpression(label.Name);  // 执行表达式
            }
        }
        
        // 步骤6: 将标签加入队列
        segments.Enqueue(labels);
        segments.Enqueue(tTVPWaveSegment(Position, one_unit));
        
        // 步骤7: 实际解码
        if(!CrossFadeSamples) {
            tjs_uint decoded;
            Decoder->Render(d, one_unit, decoded);
            Position += decoded;
            written += decoded;
            // 处理文件结束...
        } else {
            // 从预渲染的CrossFade缓冲区复制
            memcpy(d, CrossFadeSamples + CrossFadePosition * ..., ...);
        }
    }
}
```

### 3.2 流程图

```
┌──────────────────┐
│  开始 Decode()   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 查找最近 Link    │
│ GetNearestEvent()│
└────────┬─────────┘
         │
    ┌────┴────┐
    │ 找到?   │
    └────┬────┘
      是│    │否
         │    └──────────────────┐
         ▼                       │
┌──────────────────┐            │
│ Position==From?  │            │
└────────┬─────────┘            │
      是│    │否                 │
         │    │                  │
         ▼    │                  │
┌────────────┐│                  │
│ 执行跳转   ││                  │
│ Pos = To   ││                  │
└─────┬──────┘│                  │
      │       │                  │
      │       ▼                  │
      │  ┌───────────┐          │
      │  │ Smooth?   │          │
      │  └─────┬─────┘          │
      │     是│   │否            │
      │       │   │              │
      │       ▼   │              │
      │  ┌────────┴───────┐     │
      │  │ 准备CrossFade  │     │
      │  │ 预渲染两段音频 │     │
      │  │ 执行渐变混合   │     │
      │  └────────────────┘     │
      │                          │
      └──────────┬───────────────┘
                 │
                 ▼
        ┌────────────────┐
        │ 处理标签       │
        │ GetLabelAt()   │
        │ EvalLabel()    │
        └────────┬───────┘
                 │
                 ▼
        ┌────────────────┐
        │ 解码音频数据   │
        │ Decoder->Render│
        └────────┬───────┘
                 │
                 ▼
        ┌────────────────┐
        │ 循环直到完成   │
        └────────────────┘
```

---

## 4. 二分查找与条件评估

### 4.1 GetNearestEvent() 实现

```cpp
// 文件: cpp/core/sound/WaveLoopManager.cpp 第497-593行

bool tTVPWaveLoopManager::GetNearestEvent(tjs_int64 current,
                                          tTVPWaveLoopLink &link,
                                          bool ignore_conditions) {
    volatile tTJSCriticalSectionHolder CS(FlagsCS);

    if(Links.size() == 0)
        return false;

    // 确保Links已排序
    if(!IsLinksSorted) {
        std::sort(Links.begin(), Links.end());
        IsLinksSorted = true;
    }

    // 二分查找最近的Link
    tjs_int s = 0, e = (tjs_int)Links.size();
    while(e - s > 1) {
        tjs_int m = (s + e) / 2;
        if(Links[m].From <= current)
            s = m;
        else
            e = m;
    }

    // 调整到正确位置
    if(s < (int)Links.size() - 1 && Links[s].From < current)
        s++;

    if((tjs_uint)s >= Links.size() || Links[s].From < current)
        return false;

    // 回退到相同From位置的第一个Link
    tjs_int64 from = Links[s].From;
    while(s >= 1 && Links[s - 1].From == from)
        s--;

    // 条件检查
    if(!ignore_conditions) {
        do {
            if(Links[s].CondVar != -1) {
                bool match = false;
                switch(Links[s].Condition) {
                    case llcNone:
                        match = true;
                        break;
                    case llcEqual:
                        match = (Links[s].RefValue == Flags[Links[s].CondVar]);
                        break;
                    case llcNotEqual:
                        match = (Links[s].RefValue != Flags[Links[s].CondVar]);
                        break;
                    case llcGreater:
                        match = (Links[s].RefValue < Flags[Links[s].CondVar]);
                        break;
                    case llcGreaterOrEqual:
                        match = (Links[s].RefValue <= Flags[Links[s].CondVar]);
                        break;
                    case llcLesser:
                        match = (Links[s].RefValue > Flags[Links[s].CondVar]);
                        break;
                    case llcLesserOrEqual:
                        match = (Links[s].RefValue >= Flags[Links[s].CondVar]);
                        break;
                }
                if(match) break;
            } else {
                break;
            }
            s++;
        } while((tjs_uint)s < Links.size());
    }

    link = Links[s];
    return true;
}
```

### 4.2 Link排序规则

```cpp
// 文件: cpp/core/sound/WaveLoopManager.h 第142-154行

bool inline operator<(const tTVPWaveLoopLink &lhs,
                      const tTVPWaveLoopLink &rhs) {
    if(lhs.From < rhs.From)
        return true;
    if(lhs.From == rhs.From) {
        // 同一位置时，有条件的Link优先
        if(lhs.Condition != rhs.Condition)
            return lhs.Condition > rhs.Condition;
        // 条件变量索引大的优先
        return lhs.CondVar > rhs.CondVar;
    }
    return false;
}
```

这确保在同一位置有多个Link时，条件Link先被检查，无条件Link作为fallback。

---

## 5. 50ms平滑交叉渐变

### 5.1 为什么需要交叉渐变？

直接跳转（硬切）会产生"咔嗒"声：

```
音频波形（硬切）:
      ▲
      │    /\    /\
      │   /  \  /  \         突然跳转
      │  /    \/    \  ┃────────────────
      │ /            \ ┃  /\    /\
──────┼─────────────────/──\──/──\────▶
      │               ┃
      │               ↑ 跳转点（有点击声）
```

交叉渐变实现无缝过渡：

```
音频波形（交叉渐变）:
      ▲
      │    /\    /\           渐出
      │   /  \  /  \    ╲
      │  /    \/    \    ╲
      │ /            \    ╲    /\    /\
──────┼────────────────────╲──/──\──/──\──▶
      │                 ╱  渐入
      │           ╱╱╱╱
      │    50ms渐变区
```

### 5.2 交叉渐变实现

```cpp
// 文件: cpp/core/sound/WaveLoopManager.cpp 第325-406行

// 平滑链接处理
if(link.Smooth) {
    // 计算渐变前后的样本数
    int before_count = ShortCrossFadeHalfSamples;  // 25ms
    int after_count = ShortCrossFadeHalfSamples;   // 25ms
    
    // 边界调整：确保不超出文件范围
    if(link.From - before_count < 0)
        before_count = (tjs_int)link.From;
    if(link.To - before_count < 0)
        before_count = (tjs_int)link.To;
    
    // 分配缓冲区
    tjs_int alloc_size = (before_count + after_count) *
        Format->BytesPerSample * Format->Channels;
    CrossFadeSamples = new tjs_uint8[alloc_size];
    tjs_uint8* src1 = new tjs_uint8[alloc_size];  // 源1：跳转前
    tjs_uint8* src2 = new tjs_uint8[alloc_size];  // 源2：跳转后
    
    // 解码两段音频
    Decoder->Render(src1, before_count + after_count, decoded1);
    Decoder->SetPosition(link.To - before_count);
    Decoder->Render(src2, before_count + after_count, decoded2);
    
    // 执行交叉渐变
    // 前半段：src1(100%→50%) 与 src2(0%→50%) 混合
    DoCrossFade(CrossFadeSamples, src1, src2, before_count, 0, 50);
    
    // 后半段：src1(50%→0%) 与 src2(50%→100%) 混合
    DoCrossFade(CrossFadeSamples + after_offset,
                src1 + after_offset, src2 + after_offset,
                after_count, 50, 100);
    
    delete[] src1;
    delete[] src2;
    
    CrossFadePosition = 0;
    CrossFadeLen = before_count + after_count;
}
```

### 5.3 DoCrossFade 混合算法

```cpp
// 文件: cpp/core/sound/WaveLoopManager.cpp 第644-686行

void tTVPWaveLoopManager::DoCrossFade(void *dest, void *src1, void *src2,
                                       tjs_int samples, tjs_int ratiostart,
                                       tjs_int ratioend) {
    if(samples == 0) return;

    if(Format->IsFloat) {
        // 浮点格式
        float blend_step = (float)((ratioend - ratiostart) / 100.0 / samples);
        const float *s1 = (const float *)src1;
        const float *s2 = (const float *)src2;
        float *out = (float *)dest;
        float ratio = ratiostart / 100.0f;
        
        for(tjs_int i = 0; i < samples; i++) {
            for(tjs_int j = Format->Channels - 1; j >= 0; j--) {
                // 线性插值: out = s1 + (s2 - s1) * ratio
                *out = *s1 + (*s2 - *s1) * ratio;
                s1++; s2++; out++;
            }
            ratio += blend_step;
        }
    } else {
        // 整数格式 - 使用模板函数
        switch(Format->BytesPerSample) {
            case 1:
                TVPCrossFadeIntegerBlend<tTVPPCM8>(...);
                break;
            case 2:
                TVPCrossFadeIntegerBlend<tjs_int16>(...);
                break;
            case 3:
                TVPCrossFadeIntegerBlend<tTVPPCM24>(...);
                break;
            case 4:
                TVPCrossFadeIntegerBlend<tjs_int32>(...);
                break;
        }
    }
}
```

### 5.4 整数交叉渐变模板

```cpp
// 文件: cpp/core/sound/WaveLoopManager.cpp 第111-137行

template <typename T>
static void TVPCrossFadeIntegerBlend(void *dest, void *src1, void *src2,
                                     tjs_int ratiostart, tjs_int ratioend,
                                     tjs_int samples, tjs_int channels) {
    // 使用32位定点数计算比例步进
    tjs_uint blend_step = (tjs_int)(
        ((ratioend - ratiostart) * ((tjs_int64)1 << 32) / 100) / samples);
    
    const T *s1 = (const T *)src1;
    const T *s2 = (const T *)src2;
    T *out = (T *)dest;
    
    // 起始比例（32位定点数）
    tjs_uint ratio = (tjs_int)(ratiostart * ((tjs_int64)1 << 32) / 100);
    
    for(tjs_int i = 0; i < samples; i++) {
        for(tjs_int j = channels - 1; j >= 0; j--) {
            tjs_int si1 = (tjs_int)*s1;
            tjs_int si2 = (tjs_int)*s2;
            
            // 64位中间结果，防止溢出
            // o = si2 * ratio + si1 * (1 - ratio)
            tjs_int o = (tjs_int)(
                (((tjs_int64)si2 * (tjs_uint64)ratio) >> 32) +
                (((tjs_int64)si1 * (0x100000000ull - (tjs_uint64)ratio)) >> 32));
            
            *out = o;
            s1++; s2++; out++;
        }
        ratio += blend_step;
    }
}
```

---

## 6. 标签表达式系统

### 6.1 表达式语法

KrKr2支持在标签名称中嵌入表达式，用于动态修改Flag：

| 语法 | 含义 | 示例 |
|------|------|------|
| `:[n]=v` | 设置Flag[n]为v | `:[0]=1` |
| `:[n]+=v` | Flag[n]加v | `:[0]+=5` |
| `:[n]-=v` | Flag[n]减v | `:[1]-=2` |
| `:[n]++` | Flag[n]自增 | `:[2]++` |
| `:[n]--` | Flag[n]自减 | `:[3]--` |
| `:[n]=[m]` | Flag[n]=Flag[m] | `:[0]=[1]` |

### 6.2 表达式解析实现

```cpp
// 文件: cpp/core/sound/WaveLoopManager.cpp 第695-778行

bool tTVPWaveLoopManager::GetLabelExpression(
    const tTVPLabelStringType &label,
    tExpressionToken *ope, tjs_int *lv, tjs_int *rv,
    bool *is_rv_indirect) {
    
    const tTVPLabelCharType *p = label.c_str();
    
    // 必须以':'开头
    if(*p != ':') return false;
    p++;
    
    // 解析左值: [n]
    if(GetExpressionToken(p, nullptr) != etLBracket) return false;
    tjs_int lvalue;
    if(GetExpressionToken(p, &lvalue) != etInteger) return false;
    if(lvalue < 0 || lvalue >= TVP_WL_MAX_FLAGS) return false;
    if(GetExpressionToken(p, nullptr) != etRBracket) return false;
    
    // 解析操作符
    tExpressionToken operation = GetExpressionToken(p, nullptr);
    switch(operation) {
        case etEqual:
        case etPlusEqual:
        case etIncrement:
        case etMinusEqual:
        case etDecrement:
            break;
        default:
            return false;
    }
    
    // 解析右值（可能是直接值或间接值[m]）
    tjs_int rvalue = 0;
    bool rv_indirect = false;
    tExpressionToken token = GetExpressionToken(p, &rvalue);
    
    if(token == etLBracket) {
        // 间接值 [m]
        GetExpressionToken(p, &rvalue);
        // ... 验证范围
        rv_indirect = true;
    } else if(token == etInteger) {
        // 直接值
        rv_indirect = false;
    }
    
    // 输出结果
    if(ope) *ope = operation;
    if(lv) *lv = lvalue;
    if(rv) *rv = rvalue;
    if(is_rv_indirect) *is_rv_indirect = rv_indirect;
    
    return true;
}
```

### 6.3 表达式执行

```cpp
// 文件: cpp/core/sound/WaveLoopManager.cpp 第781-831行

bool tTVPWaveLoopManager::EvalLabelExpression(
    const tTVPLabelStringType &label) {
    
    volatile tTJSCriticalSectionHolder CS(FlagsCS);
    
    tExpressionToken operation;
    tjs_int lvalue, rvalue;
    bool is_rv_indirect;
    
    if(!GetLabelExpression(label, &operation, &lvalue, &rvalue, &is_rv_indirect))
        return false;
    
    // 解引用间接值
    if(is_rv_indirect)
        rvalue = Flags[rvalue];
    
    // 执行操作
    switch(operation) {
        case etEqual:
            Flags[lvalue] = rvalue;
            break;
        case etPlusEqual:
            Flags[lvalue] += rvalue;
            break;
        case etMinusEqual:
            Flags[lvalue] -= rvalue;
            break;
        case etIncrement:
            Flags[lvalue]++;
            break;
        case etDecrement:
            Flags[lvalue]--;
            break;
    }
    
    // 范围限制
    if(Flags[lvalue] < 0)
        Flags[lvalue] = 0;
    if(Flags[lvalue] > TVP_WL_MAX_FLAG_VALUE)
        Flags[lvalue] = TVP_WL_MAX_FLAG_VALUE;
    
    FlagsModifiedByLabelExpression = true;
    return true;
}
```

---

## 7. 动手实践

### 实践1：实现简单的循环点管理器

```cpp
#include <vector>
#include <algorithm>
#include <cstdint>
#include <cstring>

// 简化版循环链接
struct SimpleLoopLink {
    int64_t from;       // 跳转起点
    int64_t to;         // 跳转终点
    bool smooth;        // 是否平滑过渡
    
    bool operator<(const SimpleLoopLink& other) const {
        return from < other.from;
    }
};

// 简化版循环管理器
class SimpleLoopManager {
private:
    std::vector<SimpleLoopLink> links_;
    int64_t position_ = 0;
    bool links_sorted_ = false;
    
public:
    void addLink(int64_t from, int64_t to, bool smooth = false) {
        links_.push_back({from, to, smooth});
        links_sorted_ = false;
    }
    
    void setPosition(int64_t pos) {
        position_ = pos;
    }
    
    // 获取下一个跳转点
    bool getNextJump(int64_t current, SimpleLoopLink& link) {
        if(links_.empty()) return false;
        
        // 确保排序
        if(!links_sorted_) {
            std::sort(links_.begin(), links_.end());
            links_sorted_ = true;
        }
        
        // 二分查找
        auto it = std::lower_bound(links_.begin(), links_.end(), 
            SimpleLoopLink{current, 0, false});
        
        if(it == links_.end()) return false;
        if(it->from < current) return false;
        
        link = *it;
        return true;
    }
    
    // 模拟解码过程
    int64_t decode(int64_t samples) {
        int64_t decoded = 0;
        
        while(decoded < samples) {
            SimpleLoopLink link;
            if(getNextJump(position_, link)) {
                if(link.from == position_) {
                    // 执行跳转
                    position_ = link.to;
                    continue;
                }
                
                // 解码到跳转点
                int64_t chunk = std::min(
                    link.from - position_, 
                    samples - decoded);
                position_ += chunk;
                decoded += chunk;
            } else {
                // 无跳转，直接解码
                int64_t chunk = samples - decoded;
                position_ += chunk;
                decoded += chunk;
            }
        }
        
        return decoded;
    }
    
    int64_t getPosition() const { return position_; }
};

// 使用示例
int main() {
    SimpleLoopManager mgr;
    
    // 添加循环: 从样本100000跳转到样本10000
    mgr.addLink(100000, 10000, true);
    
    // 模拟播放
    mgr.setPosition(0);
    
    for(int i = 0; i < 5; ++i) {
        int64_t before = mgr.getPosition();
        mgr.decode(50000);  // 解码50000个样本
        int64_t after = mgr.getPosition();
        
        printf("Chunk %d: %lld -> %lld\n", i, before, after);
    }
    
    return 0;
}
```

### 实践2：实现交叉渐变

```cpp
#include <cmath>
#include <vector>

// 16位PCM交叉渐变
void crossfade16(int16_t* dest, 
                 const int16_t* src1, 
                 const int16_t* src2,
                 int samples, 
                 int channels,
                 float ratio_start,  // 0.0 = 100% src1
                 float ratio_end) {  // 1.0 = 100% src2
    
    float ratio = ratio_start;
    float step = (ratio_end - ratio_start) / samples;
    
    for(int i = 0; i < samples; ++i) {
        for(int ch = 0; ch < channels; ++ch) {
            float s1 = src1[i * channels + ch];
            float s2 = src2[i * channels + ch];
            
            // 线性插值
            float result = s1 * (1.0f - ratio) + s2 * ratio;
            
            // 限幅
            if(result > 32767) result = 32767;
            if(result < -32768) result = -32768;
            
            dest[i * channels + ch] = (int16_t)result;
        }
        ratio += step;
    }
}

// 测试
int main() {
    const int samples = 1000;
    const int channels = 2;
    
    std::vector<int16_t> src1(samples * channels);
    std::vector<int16_t> src2(samples * channels);
    std::vector<int16_t> dest(samples * channels);
    
    // 生成测试数据: src1 = 440Hz, src2 = 880Hz
    for(int i = 0; i < samples; ++i) {
        float t = i / 44100.0f;
        int16_t val1 = (int16_t)(sin(2 * M_PI * 440 * t) * 20000);
        int16_t val2 = (int16_t)(sin(2 * M_PI * 880 * t) * 20000);
        
        src1[i * 2] = src1[i * 2 + 1] = val1;
        src2[i * 2] = src2[i * 2 + 1] = val2;
    }
    
    // 执行交叉渐变: 0% -> 100%
    crossfade16(dest.data(), src1.data(), src2.data(), 
                samples, channels, 0.0f, 1.0f);
    
    printf("Crossfade complete. First/Mid/Last samples:\n");
    printf("  First: %d (src1=%d, src2=%d)\n", 
           dest[0], src1[0], src2[0]);
    printf("  Mid:   %d (src1=%d, src2=%d)\n", 
           dest[samples], src1[samples], src2[samples]);
    printf("  Last:  %d (src1=%d, src2=%d)\n", 
           dest[(samples-1)*2], src1[(samples-1)*2], src2[(samples-1)*2]);
    
    return 0;
}
```

### 实践3：解析循环信息文件

```cpp
#include <string>
#include <vector>
#include <sstream>
#include <cstdio>

struct LoopInfo {
    std::vector<SimpleLoopLink> links;
    
    bool parse(const std::string& content) {
        std::istringstream iss(content);
        std::string line;
        
        while(std::getline(iss, line)) {
            // 跳过注释和空行
            if(line.empty() || line[0] == '#') continue;
            
            // 解析 Link: from=X to=Y smooth=true/false
            if(line.find("Link:") == 0) {
                SimpleLoopLink link = {0, 0, false};
                
                // 简单解析
                sscanf(line.c_str(), "Link: from=%lld to=%lld",
                       &link.from, &link.to);
                
                if(line.find("smooth=true") != std::string::npos) {
                    link.smooth = true;
                }
                
                links.push_back(link);
            }
        }
        
        return true;
    }
};

int main() {
    const char* loopFile = R"(
# BGM循环配置
# 采样率: 44100Hz

Link: from=1323000 to=44100 smooth=true
Link: from=2646000 to=88200 smooth=false
)";
    
    LoopInfo info;
    info.parse(loopFile);
    
    printf("Parsed %zu links:\n", info.links.size());
    for(const auto& link : info.links) {
        printf("  From %lld (%.2fs) -> To %lld (%.2fs), Smooth=%s\n",
               link.from, link.from / 44100.0,
               link.to, link.to / 44100.0,
               link.smooth ? "true" : "false");
    }
    
    return 0;
}
```

---

## 8. 对照项目源码

### 关键文件清单

| 文件路径 | 行号范围 | 内容说明 |
|----------|----------|----------|
| `cpp/core/sound/WaveLoopManager.h` | 82-139 | Link结构与条件枚举 |
| `cpp/core/sound/WaveLoopManager.h` | 181-260 | 管理器类声明 |
| `cpp/core/sound/WaveLoopManager.cpp` | 111-137 | 整数交叉渐变模板 |
| `cpp/core/sound/WaveLoopManager.cpp` | 285-494 | Decode()核心逻辑 |
| `cpp/core/sound/WaveLoopManager.cpp` | 497-593 | GetNearestEvent二分查找 |
| `cpp/core/sound/WaveLoopManager.cpp` | 644-686 | DoCrossFade混合实现 |
| `cpp/core/sound/WaveLoopManager.cpp` | 695-831 | 标签表达式解析与执行 |
| `cpp/core/sound/WaveSegmentQueue.h` | 21-41 | Segment结构 |
| `cpp/core/sound/WaveSegmentQueue.h` | 47-107 | Label结构 |

---

## 9. 本节小结

- **Link结构**：定义跳转点(From/To)、条件(Condition/CondVar/RefValue)、平滑标志(Smooth)
- **二分查找**：O(log n)快速定位最近的跳转事件
- **条件评估**：支持6种比较条件，16个Flag变量
- **50ms交叉渐变**：前25ms渐出 + 后25ms渐入，消除点击声
- **标签表达式**：`:[]`语法动态修改Flag，实现复杂循环逻辑
- **防无限循环**：give_up_count限制递归跳转次数

---

## 10. 练习题与答案

### 题目1：设计多段循环

一首BGM需要以下循环行为：
- 播放A段(0-30秒)一次
- 循环B段(30-60秒)3次
- 之后永久循环C段(60-90秒)

请设计Links和Labels配置。

<details>
<summary>查看答案</summary>

```
# 假设采样率44100Hz
# A段: 0 - 1323000
# B段: 1323000 - 2646000  
# C段: 2646000 - 3969000

# 标签设置Flag[0]为循环计数器
Label: position=1323000 name=":[0]=0"      # B段开始时初始化计数器

# B段结束时检查计数器
Label: position=2646000 name=":[0]++"      # 计数器+1

# 条件跳转：Flag[0] < 3 时跳回B段开始
Link: from=2646000 to=1323000 smooth=true condition=llcLesser condvar=0 refvalue=3

# 无条件跳转：C段结束后回到C段开始（永久循环）
Link: from=3969000 to=2646000 smooth=true condition=llcNone
```

**执行流程**：
1. 播放A段 → 到达1323000，`:[0]=0`设置计数器
2. 播放B段 → 到达2646000，`:[0]++`计数器变为1，触发跳转回1323000
3. 重复B段3次（Flag[0]从0→1→2→3）
4. 当Flag[0]=3时，条件不满足，继续到C段
5. C段结束后，无条件跳回2646000，永久循环

</details>

### 题目2：交叉渐变数学推导

证明：当使用线性交叉渐变混合两个相同频率、相同振幅但相位相差180°的正弦波时，在50%混合点输出为0。

<details>
<summary>查看答案</summary>

设：
- `s1(t) = A·sin(ωt)`
- `s2(t) = A·sin(ωt + π) = -A·sin(ωt)`（相位差180°）

在50%混合点，`ratio = 0.5`：
```
out = s1 * (1 - ratio) + s2 * ratio
    = A·sin(ωt) * 0.5 + (-A·sin(ωt)) * 0.5
    = 0.5·A·sin(ωt) - 0.5·A·sin(ωt)
    = 0
```

**物理意义**：两个反相信号在等比例混合时完全抵消。这也是为什么交叉渐变需要选择合适的跳转点——如果跳转点的波形恰好反相，中间会出现静音区。KrKr2通过短时间(50ms)的渐变降低了这种影响。

</details>

### 题目3：实现条件跳转

使用C++实现一个evaluateCondition函数，根据给定的条件类型、标志值和参考值判断是否满足跳转条件。

<details>
<summary>查看答案</summary>

```cpp
#include <iostream>

enum ConditionType {
    COND_NONE,          // 无条件
    COND_EQUAL,         // ==
    COND_NOT_EQUAL,     // !=
    COND_GREATER,       // >
    COND_GREATER_EQUAL, // >=
    COND_LESSER,        // <
    COND_LESSER_EQUAL   // <=
};

bool evaluateCondition(ConditionType cond, int flagValue, int refValue) {
    switch(cond) {
        case COND_NONE:
            return true;  // 无条件总是满足
        case COND_EQUAL:
            return flagValue == refValue;
        case COND_NOT_EQUAL:
            return flagValue != refValue;
        case COND_GREATER:
            return flagValue > refValue;
        case COND_GREATER_EQUAL:
            return flagValue >= refValue;
        case COND_LESSER:
            return flagValue < refValue;
        case COND_LESSER_EQUAL:
            return flagValue <= refValue;
        default:
            return false;
    }
}

int main() {
    int flags[16] = {0};
    flags[0] = 5;
    
    // 测试各种条件
    std::cout << std::boolalpha;
    std::cout << "Flag[0] = 5\n";
    std::cout << "NONE:          " << evaluateCondition(COND_NONE, flags[0], 0) << "\n";
    std::cout << "EQUAL(5):      " << evaluateCondition(COND_EQUAL, flags[0], 5) << "\n";
    std::cout << "EQUAL(3):      " << evaluateCondition(COND_EQUAL, flags[0], 3) << "\n";
    std::cout << "NOT_EQUAL(3):  " << evaluateCondition(COND_NOT_EQUAL, flags[0], 3) << "\n";
    std::cout << "GREATER(3):    " << evaluateCondition(COND_GREATER, flags[0], 3) << "\n";
    std::cout << "GREATER(5):    " << evaluateCondition(COND_GREATER, flags[0], 5) << "\n";
    std::cout << "LESSER(10):    " << evaluateCondition(COND_LESSER, flags[0], 10) << "\n";
    
    return 0;
}

/* 输出:
Flag[0] = 5
NONE:          true
EQUAL(5):      true
EQUAL(3):      false
NOT_EQUAL(3):  true
GREATER(3):    true
GREATER(5):    false
LESSER(10):    true
*/
```

</details>

---

## 下一步

[02-WaveSegmentQueue分段播放](./02-WaveSegmentQueue分段播放.md) — 学习分段播放队列的实现与时间轴管理
