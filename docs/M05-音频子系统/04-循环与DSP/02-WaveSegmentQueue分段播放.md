# WaveSegmentQueue分段播放

> **所属模块：** M05-音频子系统
> **前置知识：** [01-WaveLoopManager循环管理](./01-WaveLoopManager循环管理.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 理解WaveSegmentQueue的设计目的与应用场景
2. 掌握tTVPWaveSegment与tTVPWaveLabel数据结构
3. 理解分段队列的入队（Enqueue）与出队（Dequeue）操作
4. 实现DSP处理后的位置映射（FilteredPosition → DecodePosition）
5. 编写自定义分段播放逻辑

---

## 1. 分段队列概述

### 1.1 为什么需要WaveSegmentQueue？

在上一节中，我们学习了WaveLoopManager如何管理循环点跳转。但当音频经过DSP处理（如变速、时间拉伸）后，会出现一个问题：**处理后的时间轴与原始时间轴不再对应**。

| 场景 | 问题 | WaveSegmentQueue解决方案 |
|------|------|-------------------------|
| 变速播放 | 1.5x播放时，1秒音频只播放0.67秒 | 记录Length→FilteredLength映射 |
| 时间拉伸 | 变速不变调后，时长改变 | 线性插值计算原始位置 |
| 循环跳转 | 跳转后位置需要重新计算 | 分段记录跳转前后的映射关系 |
| 标签触发 | 需要知道标签在处理后时间轴的位置 | Label.Offset存储处理后偏移 |

### 1.2 核心概念

```
原始解码时间轴 (Decode Position)
├─────────────────────────────────────────────────────────┤
│  Segment 1          │  Segment 2      │  Segment 3     │
│  Start=0, Len=1000  │  Start=5000     │  Start=8000    │
│                     │  Len=2000       │  Len=1500      │
├─────────────────────────────────────────────────────────┤
                              ↓ DSP处理（如1.5x变速）
├─────────────────────────────────────────────────────────┤
│  FilteredLen=667    │  FilteredLen=1333│ FilteredLen=1000│
├─────────────────────────────────────────────────────────┤
过滤后时间轴 (Filtered Position)
```

**关键比率：** `FilteredLength / Length` 表示时间压缩/拉伸比率

### 1.3 架构位置

```
┌─────────────────────────────────────────────────────────────────┐
│                    音频播放管道                                   │
├─────────────────────────────────────────────────────────────────┤
│  tTVPWaveDecoder ──▶ WaveLoopManager ──▶ PhaseVocoder ──▶ Output│
│        │                   │                  │                 │
│    原始PCM              循环跳转            变速DSP              │
│                            │                  │                 │
│                    ┌───────▼──────────────────▼───────┐         │
│                    │     tTVPWaveSegmentQueue         │         │
│                    │  - 跟踪每个分段的位置映射        │         │
│                    │  - 记录Label在处理后的偏移       │         │
│                    └──────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心数据结构

### 2.1 tTVPWaveSegment —— 播放分段

```cpp
// 文件: cpp/core/sound/WaveSegmentQueue.h 第21-41行

//! @brief 再生セグメント情報（播放分段信息）
struct tTVPWaveSegment {
    // 构造函数1：无DSP处理时，FilteredLength = Length
    tTVPWaveSegment(tjs_int64 start, tjs_int64 length) {
        Start = start;
        Length = FilteredLength = length;  // 1:1映射
    }

    // 构造函数2：有DSP处理时，长度不同
    tTVPWaveSegment(tjs_int64 start, tjs_int64 length,
                    tjs_int64 filteredlength) {
        Start = start;
        Length = length;
        FilteredLength = filteredlength;
    }

    tjs_int64 Start;          // 原始解码器上的起始位置（样本数）
    tjs_int64 Length;         // 原始解码器上的长度（样本数）
    tjs_int64 FilteredLength; // DSP处理后的长度（样本数）
};
```

**示例：变速播放**

```cpp
// 原始1000样本，1.5x播放后变成667样本
tTVPWaveSegment seg(0, 1000, 667);
// Start = 0       : 从原始位置0开始
// Length = 1000   : 原始长度1000样本
// FilteredLength = 667 : 处理后输出667样本

// 比率计算
double ratio = (double)seg.FilteredLength / seg.Length;  // = 0.667
// 这意味着1.5x变速（播放速度=1/0.667=1.5）
```

### 2.2 tTVPWaveLabel —— 播放标签

```cpp
// 文件: cpp/core/sound/WaveSegmentQueue.h 第47-107行

//! @brief 再生ラベル情報（播放标签信息）
struct tTVPWaveLabel {
    tjs_int64 Position;  // 原始解码器上的标签位置（样本数）
    ttstr Name;          // 标签名称
    tjs_int Offset;      // 处理后的偏移（Render调用时设置）

    // 默认构造
    tTVPWaveLabel() {
        Position = 0;
        Offset = 0;
    }

    // 带参构造
    tTVPWaveLabel(tjs_int64 position, const ttstr &name, tjs_int offset)
        : Position(position), Name(name), Offset(offset) {}

    // 排序函数对象 —— 按Position排序
    struct tSortByPositionFuncObj {
        bool operator()(const tTVPWaveLabel &lhs,
                        const tTVPWaveLabel &rhs) const {
            return lhs.Position < rhs.Position;
        }
    };

    // 排序函数对象 —— 按Offset排序
    struct tSortByOffsetFuncObj {
        bool operator()(const tTVPWaveLabel &lhs,
                        const tTVPWaveLabel &rhs) const {
            return lhs.Offset < rhs.Offset;
        }
    };
};

// 全局比较运算符
bool inline operator<(const tTVPWaveLabel &lhs, const tTVPWaveLabel &rhs) {
    return lhs.Position < rhs.Position;
}
```

**Label的两个位置字段说明：**

| 字段 | 含义 | 设置时机 |
|------|------|----------|
| Position | 原始解码位置 | 创建Label时指定 |
| Offset | 处理后输出位置 | Enqueue到队列时计算 |

### 2.3 tTVPWaveSegmentQueue —— 分段队列类

```cpp
// 文件: cpp/core/sound/WaveSegmentQueue.h 第118-187行

//! @brief Wave的分段/标签队列管理类
class tTVPWaveSegmentQueue {
    // 使用deque存储（虽然实际数据量不大，vector也够用）
    std::deque<tTVPWaveSegment> Segments;  // 分段数组
    std::deque<tTVPWaveLabel> Labels;      // 标签数组

public:
    // 清空队列
    void Clear();
    
    // 获取分段数组（只读）
    [[nodiscard]] const std::deque<tTVPWaveSegment>& GetSegments() const;
    
    // 获取标签数组（只读）
    [[nodiscard]] const std::deque<tTVPWaveLabel>& GetLabels() const;
    
    // === 入队操作 ===
    void Enqueue(const tTVPWaveSegmentQueue &queue);  // 合并另一个队列
    void Enqueue(const tTVPWaveSegment &segment);     // 添加单个分段
    void Enqueue(const tTVPWaveLabel &label);         // 添加单个标签
    void Enqueue(const std::deque<tTVPWaveSegment> &segments);  // 批量添加分段
    void Enqueue(const std::deque<tTVPWaveLabel> &labels);      // 批量添加标签
    
    // === 出队操作 ===
    void Dequeue(tTVPWaveSegmentQueue &dest, tjs_int64 length);
    
    // === 查询操作 ===
    [[nodiscard]] tjs_int64 GetFilteredLength() const;  // 获取总处理后长度
    
    // === 缩放操作 ===
    void Scale(tjs_int64 new_total_filtered_length);  // 缩放所有分段
    
    // === 位置映射 ===
    [[nodiscard]] tjs_int64 FilteredPositionToDecodePosition(tjs_int64 pos) const;
};
```

---

## 3. 入队操作详解

### 3.1 基本Enqueue —— 添加分段

```cpp
// 文件: cpp/core/sound/WaveSegmentQueue.cpp 第32-52行

void tTVPWaveSegmentQueue::Enqueue(const tTVPWaveSegment &segment) {
    if(Segments.size() > 0) {
        // 已有分段，检查是否可以合并
        tTVPWaveSegment &last = Segments.back();
        
        // 合并条件：
        // 1. 新分段紧接上一个分段（Start == last.Start + last.Length）
        // 2. 两个分段的压缩比率相同
        if(last.Start + last.Length == segment.Start &&
           (double)last.FilteredLength / last.Length ==
               (double)segment.FilteredLength / segment.Length) {
            // 合并：延长现有分段
            last.FilteredLength += segment.FilteredLength;
            last.Length += segment.Length;
            return;  // 不需要新增元素
        }
    }

    // 不能合并，直接添加
    Segments.push_back(segment);
}
```

**合并优化的意义：**

```cpp
// 无合并优化时，连续添加会产生大量小分段
queue.Enqueue(tTVPWaveSegment(0, 100, 100));
queue.Enqueue(tTVPWaveSegment(100, 100, 100));
queue.Enqueue(tTVPWaveSegment(200, 100, 100));
// 结果：3个分段

// 有合并优化时，连续同比率分段自动合并
// 结果：1个分段 [Start=0, Length=300, FilteredLength=300]
```

### 3.2 批量Enqueue —— 添加分段数组

```cpp
// 文件: cpp/core/sound/WaveSegmentQueue.cpp 第62-68行

void tTVPWaveSegmentQueue::Enqueue(
    const std::deque<tTVPWaveSegment> &segments) {
    // 逐个调用单分段Enqueue，触发合并优化
    for(const auto &segment : segments)
        Enqueue(segment);
}
```

### 3.3 Enqueue标签 —— 带偏移修正

```cpp
// 文件: cpp/core/sound/WaveSegmentQueue.cpp 第71-80行

void tTVPWaveSegmentQueue::Enqueue(const std::deque<tTVPWaveLabel> &Labels) {
    // 获取当前队列的处理后总长度（作为新标签的偏移基准）
    tjs_int64 Label_offset = GetFilteredLength();

    // 逐个添加标签，修正Offset
    for(auto one_Label : Labels) {
        one_Label.Offset += static_cast<tjs_int>(Label_offset);  // 加上偏移
        Enqueue(one_Label);  // 调用单标签Enqueue
    }
}
```

**标签偏移修正示例：**

```cpp
tTVPWaveSegmentQueue queue;
// 队列当前长度 = 0

queue.Enqueue(tTVPWaveSegment(0, 1000, 1000));  // FilteredLength = 1000

// 现在添加标签数组
std::deque<tTVPWaveLabel> labels;
labels.push_back(tTVPWaveLabel(500, "label1", 0));   // Offset=0
labels.push_back(tTVPWaveLabel(800, "label2", 100)); // Offset=100

queue.Enqueue(labels);
// 结果：
// label1.Offset = 0 + 1000 = 1000
// label2.Offset = 100 + 1000 = 1100
```

### 3.4 合并两个队列

```cpp
// 文件: cpp/core/sound/WaveSegmentQueue.cpp 第25-28行

void tTVPWaveSegmentQueue::Enqueue(const tTVPWaveSegmentQueue &queue) {
    Enqueue(queue.Labels);    // 先合并标签（会修正Offset）
    Enqueue(queue.Segments);  // 再合并分段（会触发合并优化）
}
```

**注意顺序：** 必须先合并Labels，因为Labels的Offset修正依赖于当前队列的`GetFilteredLength()`，如果先合并Segments会改变这个值。

---

## 4. 出队操作详解

### 4.1 Dequeue —— 切出指定长度

```cpp
// 文件: cpp/core/sound/WaveSegmentQueue.cpp 第84-142行

void tTVPWaveSegmentQueue::Dequeue(tTVPWaveSegmentQueue &dest,
                                   tjs_int64 length) {
    tjs_int64 remain;
    dest.Clear();  // 清空目标队列

    // === 第一步：切出Segments ===
    remain = length;
    while(Segments.size() > 0 && remain > 0) {
        if(Segments.front().FilteredLength <= remain) {
            // 情况1：整个分段都在切出范围内
            remain -= Segments.front().FilteredLength;
            dest.Enqueue(Segments.front());
            Segments.pop_front();  // 从源队列移除
        } else {
            // 情况2：分段需要从中间切断
            // 按FilteredLength比例计算原始Length
            tjs_int64 newlength = static_cast<tjs_int64>(
                (double)Segments.front().Length /
                (double)Segments.front().FilteredLength * remain);
            
            if(newlength > 0)
                dest.Enqueue(tTVPWaveSegment(
                    Segments.front().Start, newlength, remain));

            // 修正源分段的起始位置和长度
            Segments.front().Start += newlength;
            Segments.front().Length -= newlength;
            Segments.front().FilteredLength -= remain;
            
            // 处理边界情况：切断后长度为0
            if(Segments.front().Length == 0 ||
               Segments.front().FilteredLength == 0) {
                Segments.pop_front();
            }
            remain = 0;  // 退出循环
        }
    }

    // === 第二步：切出Labels ===
    size_t Labels_to_dequeue = 0;
    for(auto &Label : Labels) {
        tjs_int64 newoffset = Label.Offset - length;
        if(newoffset < 0) {
            // 标签在切出范围内，移到dest
            dest.Enqueue(Label);
            Labels_to_dequeue++;
        } else {
            // 标签在切出范围外，修正Offset
            Label.Offset = static_cast<tjs_int>(newoffset);
        }
    }

    // 删除已切出的标签
    while(Labels_to_dequeue--)
        Labels.pop_front();
}
```

### 4.2 Dequeue图解

```
切出前：
Queue: [Seg1: 0-1000, Filtered=1000] [Seg2: 1000-2000, Filtered=1000]
Labels: [L1@500] [L2@1200]
                ↑         ↑
             Offset=500  Offset=1200

Dequeue(dest, 800):
                │
    ┌───────────┴───────────┐
    │      切出length=800   │
    │                       │
    ▼                       ▼
dest:  [Seg1': 0-800, Filtered=800]
       [L1@500]

source: [Seg1'': 800-1000, Filtered=200] [Seg2: 1000-2000, Filtered=1000]
        [L2: Offset = 1200-800 = 400]
```

### 4.3 完整Dequeue示例

```cpp
#include <iostream>
#include <deque>

// 模拟结构体
struct Segment {
    int64_t Start, Length, FilteredLength;
    Segment(int64_t s, int64_t l, int64_t f) 
        : Start(s), Length(l), FilteredLength(f) {}
};

struct Label {
    int64_t Position;
    std::string Name;
    int64_t Offset;
    Label(int64_t p, const std::string& n, int64_t o)
        : Position(p), Name(n), Offset(o) {}
};

class SegmentQueue {
    std::deque<Segment> segments;
    std::deque<Label> labels;

public:
    void Enqueue(const Segment& seg) {
        segments.push_back(seg);
    }
    
    void Enqueue(const Label& lbl) {
        labels.push_back(lbl);
    }
    
    int64_t GetFilteredLength() const {
        int64_t total = 0;
        for (const auto& seg : segments)
            total += seg.FilteredLength;
        return total;
    }
    
    void Dequeue(SegmentQueue& dest, int64_t length) {
        dest.Clear();
        int64_t remain = length;
        
        // 切出分段
        while (!segments.empty() && remain > 0) {
            auto& front = segments.front();
            if (front.FilteredLength <= remain) {
                remain -= front.FilteredLength;
                dest.Enqueue(front);
                segments.pop_front();
            } else {
                // 线性插值计算原始长度
                int64_t newlen = static_cast<int64_t>(
                    (double)front.Length / front.FilteredLength * remain);
                dest.Enqueue(Segment(front.Start, newlen, remain));
                front.Start += newlen;
                front.Length -= newlen;
                front.FilteredLength -= remain;
                remain = 0;
            }
        }
        
        // 切出标签
        auto it = labels.begin();
        while (it != labels.end()) {
            if (it->Offset < length) {
                dest.Enqueue(*it);
                it = labels.erase(it);
            } else {
                it->Offset -= length;
                ++it;
            }
        }
    }
    
    void Clear() {
        segments.clear();
        labels.clear();
    }
    
    void Print(const std::string& name) const {
        std::cout << name << " Queue:\n";
        std::cout << "  Segments:\n";
        for (const auto& seg : segments) {
            std::cout << "    [Start=" << seg.Start 
                      << ", Len=" << seg.Length 
                      << ", Filtered=" << seg.FilteredLength << "]\n";
        }
        std::cout << "  Labels:\n";
        for (const auto& lbl : labels) {
            std::cout << "    [" << lbl.Name 
                      << " @Offset=" << lbl.Offset << "]\n";
        }
    }
};

int main() {
    SegmentQueue queue;
    
    // 添加分段和标签
    queue.Enqueue(Segment(0, 1000, 1000));
    queue.Enqueue(Segment(1000, 2000, 2000));
    queue.Enqueue(Label(500, "intro", 500));
    queue.Enqueue(Label(1500, "verse", 1500));
    queue.Enqueue(Label(2500, "chorus", 2500));
    
    std::cout << "=== 初始状态 ===\n";
    queue.Print("Source");
    
    // 切出前800样本
    SegmentQueue dest;
    queue.Dequeue(dest, 800);
    
    std::cout << "\n=== Dequeue(800)后 ===\n";
    dest.Print("Dest");
    queue.Print("Source");
    
    return 0;
}
```

**输出：**
```
=== 初始状态 ===
Source Queue:
  Segments:
    [Start=0, Len=1000, Filtered=1000]
    [Start=1000, Len=2000, Filtered=2000]
  Labels:
    [intro @Offset=500]
    [verse @Offset=1500]
    [chorus @Offset=2500]

=== Dequeue(800)后 ===
Dest Queue:
  Segments:
    [Start=0, Len=800, Filtered=800]
  Labels:
    [intro @Offset=500]
Source Queue:
  Segments:
    [Start=800, Len=200, Filtered=200]
    [Start=1000, Len=2000, Filtered=2000]
  Labels:
    [verse @Offset=700]
    [chorus @Offset=1700]
```

---

## 5. 长度与缩放操作

### 5.1 GetFilteredLength —— 获取总长度

```cpp
// 文件: cpp/core/sound/WaveSegmentQueue.cpp 第146-153行

tjs_int64 tTVPWaveSegmentQueue::GetFilteredLength() const {
    // 队列长度 = 所有分段的FilteredLength之和
    tjs_int64 length = 0;
    for(const auto &Segment : Segments)
        length += Segment.FilteredLength;
    return length;
}
```

### 5.2 Scale —— 缩放队列长度

`Scale()`用于DSP处理后调整所有分段和标签的长度，保持比例关系：

```cpp
// 文件: cpp/core/sound/WaveSegmentQueue.cpp 第157-200行

void tTVPWaveSegmentQueue::Scale(tjs_int64 new_total_filtered_length) {
    tjs_int64 total_length_was = GetFilteredLength();  // 变化前长度
    
    if(total_length_was == 0)
        return;  // 空队列无法缩放

    // === 缩放Segments ===
    tjs_int64 offset_was = 0;  // 变化前累计偏移
    tjs_int64 offset_is = 0;   // 变化后累计偏移

    for(auto &Segment : Segments) {
        tjs_int64 old_end = offset_was + Segment.FilteredLength;
        offset_was += Segment.FilteredLength;

        // 计算该分段终点在全局的比例
        double ratio = static_cast<double>(old_end) /
            static_cast<double>(total_length_was);

        // 新终点位置
        tjs_int64 new_end =
            static_cast<tjs_int64>(ratio * new_total_filtered_length);

        // 新FilteredLength = 新终点 - 当前偏移
        Segment.FilteredLength = new_end - offset_is;
        offset_is += Segment.FilteredLength;
    }

    // === 删除空分段 ===
    for(auto i = Segments.begin(); i != Segments.end();) {
        if(i->FilteredLength == 0 || i->Length == 0)
            i = Segments.erase(i);
        else
            i++;
    }

    // === 缩放Labels ===
    double ratio = (double)new_total_filtered_length / (double)total_length_was;
    for(auto &Label : Labels) {
        Label.Offset = static_cast<tjs_int64>(Label.Offset * ratio);
    }
}
```

**Scale使用场景：**

```cpp
// 场景：音频原本3000样本，PhaseVocoder处理后变成2000样本
tTVPWaveSegmentQueue queue;
queue.Enqueue(tTVPWaveSegment(0, 1000, 1000));
queue.Enqueue(tTVPWaveSegment(1000, 1000, 1000));
queue.Enqueue(tTVPWaveSegment(2000, 1000, 1000));
// 当前FilteredLength = 3000

// DSP处理结果是2000样本
queue.Scale(2000);
// 现在每个分段的FilteredLength按比例缩小
// Seg1: FilteredLength = 667
// Seg2: FilteredLength = 666
// Seg3: FilteredLength = 667
```

---

## 6. 位置映射

### 6.1 FilteredPositionToDecodePosition

这是WaveSegmentQueue最重要的功能——将处理后的播放位置映射回原始解码位置：

```cpp
// 文件: cpp/core/sound/WaveSegmentQueue.cpp 第204-230行

tjs_int64 tTVPWaveSegmentQueue::FilteredPositionToDecodePosition(
    tjs_int64 pos) const {
    tjs_int64 offset_filtered = 0;

    for(const auto &Segment : Segments) {
        // 检查pos是否在当前分段范围内
        if(offset_filtered <= pos &&
           pos < offset_filtered + Segment.FilteredLength) {
            // 找到对应分段，线性插值计算原始位置
            return static_cast<tjs_int64>(
                Segment.Start +
                (pos - offset_filtered) *
                    (double)Segment.Length /
                    (double)Segment.FilteredLength);
        }
        offset_filtered += Segment.FilteredLength;
    }

    // 未找到对应分段
    if(pos < 0)
        return 0;  // 负位置返回0
    if(Segments.size() == 0)
        return 0;  // 空队列返回0
    
    // 超出范围，返回最后位置
    return Segments.back().Start + Segments.back().Length;
}
```

### 6.2 位置映射图解

```
原始时间轴:
  0        1000      2000      3000
  ├──────────┼─────────┼─────────┤
  │  Seg1    │  Seg2   │  Seg3   │
  │ 0-1000   │1000-2000│2000-3000│

处理后时间轴（1.5x变速）:
  0    667    1333    2000
  ├──────┼──────┼──────┤
  │Fil=667│Fil=666│Fil=667│

FilteredPositionToDecodePosition(1000):
  1. 累计offset: Seg1=667, Seg2=667+666=1333
  2. 1000 在 [667, 1333) 范围内 → 属于Seg2
  3. 计算: Seg2.Start + (1000-667) * (1000/666)
         = 1000 + 333 * 1.5
         = 1000 + 500
         = 1500

所以播放位置1000样本对应原始位置1500样本
```

### 6.3 完整位置映射示例

```cpp
#include <iostream>
#include <vector>
#include <cstdint>

struct Segment {
    int64_t Start, Length, FilteredLength;
};

class SegmentQueue {
    std::vector<Segment> segments;
    
public:
    void AddSegment(int64_t start, int64_t len, int64_t filtered) {
        segments.push_back({start, len, filtered});
    }
    
    int64_t FilteredToOriginal(int64_t filtered_pos) const {
        int64_t offset = 0;
        
        for (const auto& seg : segments) {
            if (offset <= filtered_pos && 
                filtered_pos < offset + seg.FilteredLength) {
                // 线性插值
                double local_pos = filtered_pos - offset;
                double ratio = (double)seg.Length / seg.FilteredLength;
                return seg.Start + static_cast<int64_t>(local_pos * ratio);
            }
            offset += seg.FilteredLength;
        }
        
        // 边界处理
        if (filtered_pos < 0) return 0;
        if (segments.empty()) return 0;
        return segments.back().Start + segments.back().Length;
    }
    
    void PrintMapping() const {
        int64_t total_filtered = 0;
        for (const auto& seg : segments)
            total_filtered += seg.FilteredLength;
        
        std::cout << "位置映射表:\n";
        std::cout << "Filtered -> Original\n";
        for (int64_t pos = 0; pos <= total_filtered; pos += 100) {
            std::cout << "  " << pos << " -> " 
                      << FilteredToOriginal(pos) << "\n";
        }
    }
};

int main() {
    SegmentQueue queue;
    
    // 模拟变速播放：原始3000样本 → 处理后2000样本
    queue.AddSegment(0, 1000, 667);      // Seg1
    queue.AddSegment(1000, 1000, 666);   // Seg2
    queue.AddSegment(2000, 1000, 667);   // Seg3
    
    queue.PrintMapping();
    
    // 验证几个关键点
    std::cout << "\n关键点验证:\n";
    std::cout << "  播放位置0 -> 原始位置" 
              << queue.FilteredToOriginal(0) << "\n";
    std::cout << "  播放位置667 -> 原始位置" 
              << queue.FilteredToOriginal(667) << "\n";
    std::cout << "  播放位置1000 -> 原始位置" 
              << queue.FilteredToOriginal(1000) << "\n";
    std::cout << "  播放位置2000 -> 原始位置" 
              << queue.FilteredToOriginal(2000) << "\n";
    
    return 0;
}
```

---

## 7. 动手实践

### 实践1：实现简单的分段队列

创建一个支持基本入队、出队和位置映射的分段队列：

```cpp
// simple_segment_queue.cpp
#include <iostream>
#include <deque>
#include <string>
#include <cstdint>

struct SimpleSegment {
    int64_t Start;
    int64_t Length;
    int64_t FilteredLength;
    
    SimpleSegment(int64_t s, int64_t l, int64_t f = -1) 
        : Start(s), Length(l), FilteredLength(f < 0 ? l : f) {}
};

struct SimpleLabel {
    std::string Name;
    int64_t Position;
    int64_t Offset;
    
    SimpleLabel(const std::string& n, int64_t p, int64_t o = 0)
        : Name(n), Position(p), Offset(o) {}
};

class SimpleSegmentQueue {
    std::deque<SimpleSegment> segments_;
    std::deque<SimpleLabel> labels_;
    
public:
    // 清空
    void Clear() {
        segments_.clear();
        labels_.clear();
    }
    
    // 添加分段（带合并优化）
    void Enqueue(const SimpleSegment& seg) {
        if (!segments_.empty()) {
            auto& last = segments_.back();
            // 检查是否连续且比率相同
            if (last.Start + last.Length == seg.Start) {
                double last_ratio = (double)last.FilteredLength / last.Length;
                double seg_ratio = (double)seg.FilteredLength / seg.Length;
                if (std::abs(last_ratio - seg_ratio) < 0.0001) {
                    // 合并
                    last.Length += seg.Length;
                    last.FilteredLength += seg.FilteredLength;
                    return;
                }
            }
        }
        segments_.push_back(seg);
    }
    
    // 添加标签
    void Enqueue(const SimpleLabel& lbl) {
        SimpleLabel adjusted = lbl;
        adjusted.Offset = lbl.Offset + GetFilteredLength();
        labels_.push_back(adjusted);
    }
    
    // 获取处理后总长度
    int64_t GetFilteredLength() const {
        int64_t total = 0;
        for (const auto& seg : segments_)
            total += seg.FilteredLength;
        return total;
    }
    
    // 位置映射
    int64_t FilteredToOriginal(int64_t pos) const {
        int64_t offset = 0;
        for (const auto& seg : segments_) {
            if (pos >= offset && pos < offset + seg.FilteredLength) {
                double local = pos - offset;
                double ratio = (double)seg.Length / seg.FilteredLength;
                return seg.Start + static_cast<int64_t>(local * ratio);
            }
            offset += seg.FilteredLength;
        }
        return segments_.empty() ? 0 : 
               segments_.back().Start + segments_.back().Length;
    }
    
    // 打印队列状态
    void Print() const {
        std::cout << "Segments (" << segments_.size() << "):\n";
        for (const auto& seg : segments_) {
            std::cout << "  [" << seg.Start << "-" 
                      << (seg.Start + seg.Length)
                      << ", Filtered=" << seg.FilteredLength << "]\n";
        }
        std::cout << "Labels (" << labels_.size() << "):\n";
        for (const auto& lbl : labels_) {
            std::cout << "  " << lbl.Name << " @" << lbl.Offset << "\n";
        }
        std::cout << "Total FilteredLength: " << GetFilteredLength() << "\n";
    }
};

int main() {
    SimpleSegmentQueue queue;
    
    // 测试1：普通入队
    std::cout << "=== 测试1：普通入队 ===\n";
    queue.Enqueue(SimpleSegment(0, 1000));
    queue.Enqueue(SimpleSegment(1000, 500));
    queue.Print();
    
    // 测试2：合并入队
    std::cout << "\n=== 测试2：合并入队 ===\n";
    queue.Clear();
    queue.Enqueue(SimpleSegment(0, 100, 100));
    queue.Enqueue(SimpleSegment(100, 100, 100));  // 应该合并
    queue.Enqueue(SimpleSegment(200, 100, 50));   // 比率不同，不合并
    queue.Print();
    
    // 测试3：位置映射
    std::cout << "\n=== 测试3：位置映射 ===\n";
    queue.Clear();
    queue.Enqueue(SimpleSegment(0, 1000, 500));   // 2x变速
    queue.Enqueue(SimpleSegment(1000, 1000, 1000)); // 1x变速
    
    std::cout << "FilteredPos -> OriginalPos:\n";
    std::cout << "  0 -> " << queue.FilteredToOriginal(0) << "\n";
    std::cout << "  250 -> " << queue.FilteredToOriginal(250) << "\n";
    std::cout << "  500 -> " << queue.FilteredToOriginal(500) << "\n";
    std::cout << "  1000 -> " << queue.FilteredToOriginal(1000) << "\n";
    
    return 0;
}
```

**编译运行：**
```bash
g++ -std=c++17 -o simple_segment_queue simple_segment_queue.cpp
./simple_segment_queue
```

---

## 8. 对照项目源码

### 8.1 相关文件

| 文件 | 行数 | 说明 |
|------|------|------|
| `cpp/core/sound/WaveSegmentQueue.h` | 1-190 | 类定义与数据结构 |
| `cpp/core/sound/WaveSegmentQueue.cpp` | 1-231 | 全部实现代码 |
| `cpp/core/sound/WaveLoopManager.cpp` | ~800行 | 使用SegmentQueue跟踪循环跳转 |
| `cpp/core/sound/PhaseVocoderFilter.cpp` | ~300行 | DSP处理后调用Scale() |

### 8.2 关键调用链

```cpp
// WaveLoopManager::Render()中使用SegmentQueue
void tTVPWaveLoopManager::Render(void* dest, tjs_uint samples,
                                  tTVPWaveSegmentQueue& segments) {
    // ...解码音频...
    
    // 记录当前分段
    segments.Enqueue(tTVPWaveSegment(decode_start, decoded_samples));
    
    // 如果发生循环跳转
    if (link_triggered) {
        // 新分段从跳转目标位置开始
        segments.Enqueue(tTVPWaveSegment(link->To, remaining_samples));
    }
}

// PhaseVocoder处理后缩放
void ProcessWithPhaseVocoder(tTVPWaveSegmentQueue& segments,
                              int64_t output_samples) {
    // DSP处理改变了输出长度
    segments.Scale(output_samples);
}
```

---

## 9. 本节小结

- **tTVPWaveSegment** 记录原始位置(Start, Length)与处理后长度(FilteredLength)的映射
- **tTVPWaveLabel** 的Offset在入队时自动修正，反映在处理后时间轴的位置
- **Enqueue** 支持自动合并连续同比率分段，减少内存开销
- **Dequeue** 使用线性插值精确切分分段，同时调整标签偏移
- **Scale** 用于DSP处理后整体缩放队列长度
- **FilteredPositionToDecodePosition** 是核心位置映射函数，支持seek操作

---

## 10. 练习题与答案

### 题目1：分段合并条件

以下代码执行后，队列中有几个分段？

```cpp
tTVPWaveSegmentQueue queue;
queue.Enqueue(tTVPWaveSegment(0, 100, 100));
queue.Enqueue(tTVPWaveSegment(100, 100, 100));
queue.Enqueue(tTVPWaveSegment(200, 100, 50));
queue.Enqueue(tTVPWaveSegment(300, 100, 50));
```

<details>
<summary>查看答案</summary>

**答案：3个分段**

分析：
1. 第一个分段：`[Start=0, Length=100, Filtered=100]`，比率=1.0
2. 第二个分段与第一个**连续**(100==0+100)且**比率相同**(1.0)，合并为`[0, 200, 200]`
3. 第三个分段与第二个连续(200==0+200)但**比率不同**(0.5≠1.0)，不合并，新增分段
4. 第四个分段与第三个连续(300==200+100)且**比率相同**(0.5)，合并

最终：
- 分段1: `[Start=0, Length=200, FilteredLength=200]`
- 分段2: `[Start=200, Length=200, FilteredLength=100]`

等等，让我重新计算：
- Seg1+Seg2合并: `[0, 200, 200]`
- Seg3: `[200, 100, 50]` (比率0.5)
- Seg4: 起点300，Seg3终点=200+100=300，连续；比率50/100=0.5相同，合并
- Seg3+Seg4合并: `[200, 200, 100]`

**正确答案：2个分段**
- `[Start=0, Length=200, FilteredLength=200]`
- `[Start=200, Length=200, FilteredLength=100]`

</details>

### 题目2：位置映射计算

给定以下队列，计算`FilteredPositionToDecodePosition(450)`的返回值：

```cpp
tTVPWaveSegmentQueue queue;
queue.Enqueue(tTVPWaveSegment(0, 1000, 500));    // 2x变速
queue.Enqueue(tTVPWaveSegment(1000, 500, 500));  // 1x变速
```

<details>
<summary>查看答案</summary>

**答案：900**

计算过程：
1. 分段1: Filtered范围[0, 500)，原始范围[0, 1000)
2. 分段2: Filtered范围[500, 1000)，原始范围[1000, 1500)
3. 位置450在分段1范围[0, 500)内
4. 本地偏移: 450 - 0 = 450
5. 比率: Length/FilteredLength = 1000/500 = 2
6. 原始位置: Start + local * ratio = 0 + 450 * 2 = **900**

</details>

### 题目3：实现Dequeue函数

补全以下Dequeue实现的核心逻辑（处理分段切分的部分）：

```cpp
void Dequeue(SegmentQueue& dest, int64_t length) {
    dest.Clear();
    int64_t remain = length;
    
    while (!segments_.empty() && remain > 0) {
        auto& front = segments_.front();
        if (front.FilteredLength <= remain) {
            // 情况1：整个分段都在范围内
            remain -= front.FilteredLength;
            dest.Enqueue(front);
            segments_.pop_front();
        } else {
            // 情况2：需要切分分段
            // TODO: 补全此处代码
        }
    }
}
```

<details>
<summary>查看答案</summary>

```cpp
void Dequeue(SegmentQueue& dest, int64_t length) {
    dest.Clear();
    int64_t remain = length;
    
    while (!segments_.empty() && remain > 0) {
        auto& front = segments_.front();
        if (front.FilteredLength <= remain) {
            remain -= front.FilteredLength;
            dest.Enqueue(front);
            segments_.pop_front();
        } else {
            // 情况2：需要切分分段
            // 1. 计算原始长度（线性插值）
            int64_t newLength = static_cast<int64_t>(
                (double)front.Length / front.FilteredLength * remain);
            
            // 2. 创建新分段并入队
            if (newLength > 0) {
                dest.Enqueue(Segment(front.Start, newLength, remain));
            }
            
            // 3. 修正源分段
            front.Start += newLength;
            front.Length -= newLength;
            front.FilteredLength -= remain;
            
            // 4. 边界处理：如果切分后长度为0则删除
            if (front.Length == 0 || front.FilteredLength == 0) {
                segments_.pop_front();
            }
            
            // 5. 退出循环
            remain = 0;
        }
    }
}
```

关键点：
- 使用`(double)front.Length / front.FilteredLength * remain`计算原始长度
- 修正源分段的Start、Length、FilteredLength
- 处理长度为0的边界情况
- 设置`remain = 0`退出循环

</details>

---

## 下一步

[03-PhaseVocoder变速不变调](./03-PhaseVocoder变速不变调.md) — 学习如何使用相位声码器实现音频变速不变调效果
