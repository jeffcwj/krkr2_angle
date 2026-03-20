# tTVPArchive 基类设计

> **所属模块：** M08-归档与IO系统
> **前置知识：** [01-KrKr2存储架构总览](./01-KrKr2存储架构总览.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：

1. **理解 tTVPArchive 基类的设计哲学** — 知道为什么需要一个统一的归档抽象基类
2. **掌握引用计数（Reference Counting）在归档管理中的应用** — 理解归档对象的生命周期管理
3. **理解哈希索引的实现** — 知道引擎如何通过哈希表实现 O(1) 的文件查找
4. **熟悉纯虚函数接口** — 了解子类必须实现哪些方法，以及每个方法的职责

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 引用计数 | Reference Counting | 一种内存管理技术——每个对象记录有多少地方在使用它，计数归零时自动释放 |
| 纯虚函数 | Pure Virtual Function | C++ 中用 `= 0` 标记的虚函数，基类不提供实现，强制子类必须实现 |
| 哈希索引 | Hash Index | 用哈希表为归档中的文件名建立快速查找结构，查找时间为 O(1) |
| 二分查找 | Binary Search | 在已排序的数组中通过不断折半来查找目标元素的算法，时间复杂度 O(log n) |
| 抽象基类 | Abstract Base Class | 包含纯虚函数、不能直接实例化的类，只能通过子类间接使用 |
| 工厂方法 | Factory Method | 一种创建对象的设计模式——通过一个方法来决定创建哪种具体类型的对象 |

## tTVPArchive 类的角色

### 为什么需要抽象基类

KrKr2 支持四种归档格式（XP3、ZIP、TAR、7z），每种格式的文件结构完全不同：

| 格式 | 索引位置 | 压缩方式 | 特殊特性 |
|------|----------|----------|---------|
| XP3 | 文件末尾（块索引） | 分段 zlib 压缩 | 加密过滤器、嵌入 EXE |
| ZIP | 中央目录（文件末尾） | 单文件 deflate | ZIP64 扩展 |
| TAR | 无独立索引（顺序排列） | 无压缩 | 长文件名扩展头 |
| 7z | 文件头部 | LZMA/LZMA2 | 固实压缩（solid compression，把多个文件当作一个连续数据块来压缩，提高压缩率但降低随机访问性能） |

但对于引擎上层来说，不管是什么格式，需求都是一样的：

1. 知道归档里有多少个文件
2. 知道每个文件叫什么名字
3. 能打开某个文件并读取它的数据

`tTVPArchive` 就是把这些共同需求抽象出来的基类。上层代码只需要持有一个 `tTVPArchive*` 指针，就能操作任何格式的归档，完全不需要知道底层用的是 XP3 还是 ZIP。

### 类继承层次

```
tTVPArchive（抽象基类）
    │
    ├── tTVPXP3Archive      ← XP3 格式实现
    ├── TVPZIPArchive        ← ZIP 格式实现
    ├── TVPTARArchive        ← TAR 格式实现
    └── TVP7zArchive         ← 7z 格式实现
```

## 基类定义详解

### 完整的类声明

以下是 `tTVPArchive` 基类的完整声明，来自 `cpp/core/base/StorageIntf.h`：

```cpp
// 文件路径：cpp/core/base/StorageIntf.h
// tTVPArchive — 所有归档格式的抽象基类
class tTVPArchive {
    // ---------- 引用计数 ----------
    tjs_uint RefCount;  // 引用计数器，记录有多少地方在使用这个归档对象

public:
    // 构造函数：初始化引用计数为 1
    tTVPArchive() : RefCount(1) {}

    // 虚析构函数：允许通过基类指针正确释放子类对象
    virtual ~tTVPArchive() = default;

    // AddRef() — 增加引用计数
    void AddRef() { RefCount++; }

    // Release() — 减少引用计数，归零时删除自身
    void Release() {
        if (RefCount == 1) {
            delete this;  // 引用计数为 1 时，这是最后一个引用
        } else {
            RefCount--;
        }
    }

    // ---------- 哈希索引 ----------
private:
    // 文件名哈希表：文件名 → 文件索引号的映射
    // 用于 O(1) 时间复杂度的文件查找
    std::unordered_map<ttstr, tjs_uint> NameHash;

    // 所有文件名的有序列表（按字母排序）
    // 用于目录列表和前缀匹配
    std::vector<ttstr> Names;

    // 归档文件的路径
    ttstr ArchiveName;

protected:
    // 子类调用此方法添加文件到索引
    void AddFileEntry(const ttstr &name) {
        tjs_uint index = (tjs_uint)Names.size();
        ttstr normalized = name;
        NormalizeInArchiveStorageName(normalized);
        Names.push_back(normalized);
        NameHash[normalized] = index;
    }

public:
    // ---------- 文件查找 ----------

    // GetCount() — 返回归档中的文件总数
    tjs_uint GetCount() const {
        return (tjs_uint)Names.size();
    }

    // GetName() — 返回指定索引处的文件名
    ttstr GetName(tjs_uint index) const {
        return Names[index];
    }

    // GetIndex() — 根据文件名查找索引号
    // 使用哈希表，O(1) 时间复杂度
    tjs_int GetIndex(const ttstr &name) const {
        ttstr normalized = name;
        NormalizeInArchiveStorageName(normalized);
        auto it = NameHash.find(normalized);
        if (it != NameHash.end()) {
            return (tjs_int)it->second;
        }
        return -1;  // 未找到
    }

    // GetFirstIndexStartsWith() — 查找第一个以指定前缀开头的文件
    // 使用二分查找，用于目录列表功能
    tjs_int GetFirstIndexStartsWith(const ttstr &prefix) const;

    // ---------- 纯虚函数接口（子类必须实现）----------

    // CreateStreamByIndex() — 根据文件索引创建读取流
    // 这是最核心的方法：给一个索引号，返回能读取该文件数据的流对象
    virtual tTJSBinaryStream* CreateStreamByIndex(
        tjs_uint index) = 0;

    // ---------- 便捷方法 ----------

    // CreateStream() — 根据文件名创建读取流
    // 内部先调用 GetIndex() 查找索引，再调用 CreateStreamByIndex()
    tTJSBinaryStream* CreateStream(const ttstr &name,
                                   tjs_uint32 flags) {
        tjs_int index = GetIndex(name);
        if (index < 0) {
            TVPThrowExceptionMessage(
                TJS_W("File '%1' is not found in archive '%2'."),
                name, ArchiveName);
        }
        return CreateStreamByIndex((tjs_uint)index);
    }

    // GetArchiveName() — 获取归档文件路径
    const ttstr& GetArchiveName() const {
        return ArchiveName;
    }

    // SetArchiveName() — 设置归档文件路径
    void SetArchiveName(const ttstr &name) {
        ArchiveName = name;
    }

    // ---------- 路径规范化工具 ----------

    // NormalizeInArchiveStorageName() — 归档内部路径规范化
    static void NormalizeInArchiveStorageName(ttstr &name);
};
```

下面逐一深入讲解每个关键设计。

## 引用计数（Reference Counting）详解

### 什么是引用计数

**引用计数**是一种内存管理策略。核心思想很简单：每个对象维护一个计数器，记录有多少地方在"引用"（使用）这个对象。当一个新的地方开始使用对象时，计数器加 1（`AddRef()`）；当某个地方不再使用时，计数器减 1（`Release()`）。当计数器归零时，说明没有任何地方在使用这个对象了，可以安全地释放它。

```
引用计数的生命周期示意：

创建归档对象                           RefCount = 1
    │
    ├─ 归档缓存持有引用     AddRef()    RefCount = 2
    ├─ 流 A 正在读取       AddRef()    RefCount = 3
    ├─ 流 B 正在读取       AddRef()    RefCount = 4
    │
    ├─ 流 A 读取完毕       Release()   RefCount = 3
    ├─ 流 B 读取完毕       Release()   RefCount = 2
    ├─ 缓存清理            Release()   RefCount = 1
    ├─ 最后一个引用释放     Release()   RefCount = 0
    │
    └─ 对象被 delete
```

### 为什么归档对象需要引用计数

归档对象的生命周期比较复杂——它可能同时被多个地方使用：

1. **归档缓存（Archive Cache）**持有一份引用，避免重复打开同一个归档文件
2. **读取流（Stream）**持有一份引用，确保流在读取期间归档不会被释放
3. **自动路径扫描**临时持有引用，扫描完毕后释放

如果没有引用计数，很难确定什么时候可以安全释放归档对象。过早释放会导致正在读取的流访问已释放的内存（悬挂指针，Dangling Pointer）；过晚释放会导致内存泄漏（Memory Leak，即分配了内存却忘记释放，导致可用内存越来越少）。

### 引用计数 vs 智能指针

你可能会问：为什么不用 C++ 标准库的 `std::shared_ptr` 呢？它也是基于引用计数的。

原因是 KrKr2 的代码源自较早的 KiriKiri 引擎，当时 `std::shared_ptr` 尚未被广泛使用（C++11 之前的项目）。此外，手动管理引用计数在某些场景下更灵活——比如可以在 `Release()` 中做额外的清理工作。

```cpp
// 对比：手动引用计数 vs std::shared_ptr

// 手动引用计数（KrKr2 的做法）
tTVPArchive *archive = new tTVPXP3Archive("data.xp3");
// archive->RefCount == 1
archive->AddRef();   // 手动增加引用
// archive->RefCount == 2
archive->Release();  // 手动减少引用
// archive->RefCount == 1
archive->Release();  // 引用归零，自动 delete
// archive 已被释放，不能再使用

// std::shared_ptr（现代 C++ 的做法）
auto archive = std::make_shared<tTVPXP3Archive>("data.xp3");
// use_count() == 1
auto copy = archive;  // 自动增加引用
// use_count() == 2
copy.reset();          // 自动减少引用
// use_count() == 1
archive.reset();       // 引用归零，自动释放
```

### 引用计数的陷阱

引用计数有一个经典问题：**循环引用**（Circular Reference）。如果对象 A 引用对象 B，对象 B 又引用对象 A，两者的引用计数永远不会归零，造成内存泄漏。

```
A.RefCount = 1（B 引用）  →  B.RefCount = 1（A 引用）
│                            │
└────── 引用 ────────────────┘
// 两者都不会被释放！
```

在归档系统中，这个问题不太会出现——因为引用关系是单向的：缓存 → 归档 → 无（归档不会反向引用缓存）。但在编写自定义归档格式时仍需注意。

## 哈希索引详解

### 为什么需要哈希索引

一个典型的 XP3 归档可能包含数千个文件。如果每次查找文件都要遍历整个文件列表（线性查找，时间复杂度 O(n)），性能会很差。

`tTVPArchive` 使用哈希表（`std::unordered_map`）为所有文件名建立索引，将查找时间从 O(n) 降低到 O(1)：

```
没有哈希索引时：
  查找 "bgimage/bg01.png" → 遍历所有 5000 个文件名 → O(5000)

有哈希索引后：
  查找 "bgimage/bg01.png" → 计算哈希值 → 直接定位 → O(1)
```

### 索引的构建过程

当归档文件被打开时，子类在构造函数中解析文件索引，然后调用基类的 `AddFileEntry()` 方法逐个添加文件：

```cpp
// 文件路径：cpp/core/base/StorageIntf.h（简化版本）
// AddFileEntry() — 添加文件到哈希索引
void tTVPArchive::AddFileEntry(const ttstr &name) {
    tjs_uint index = (tjs_uint)Names.size();  // 新文件的索引号

    // 规范化文件名（小写 + 统一斜杠）
    ttstr normalized = name;
    NormalizeInArchiveStorageName(normalized);

    // 添加到有序列表
    Names.push_back(normalized);

    // 添加到哈希表
    NameHash[normalized] = index;
}
```

以一个包含 4 个文件的归档为例：

```
归档内容：
  0: bgimage/bg01.png
  1: bgimage/bg02.png
  2: scenario/prologue.ks
  3: scenario/chapter01.ks

Names 列表（有序）：
  [0] = "bgimage/bg01.png"
  [1] = "bgimage/bg02.png"
  [2] = "scenario/chapter01.ks"
  [3] = "scenario/prologue.ks"

NameHash 哈希表：
  "bgimage/bg01.png"       → 0
  "bgimage/bg02.png"       → 1
  "scenario/chapter01.ks"  → 2
  "scenario/prologue.ks"   → 3
```

### 文件名查找流程

```cpp
// 查找文件的完整流程
tjs_int index = archive->GetIndex("BGImage\\BG01.PNG");

// 第 1 步：规范化输入的文件名
//   "BGImage\\BG01.PNG" → "bgimage/bg01.png"

// 第 2 步：在哈希表中查找
//   hash("bgimage/bg01.png") → 命中 → 返回索引 0

// 第 3 步：使用索引创建流
tTJSBinaryStream *stream = archive->CreateStreamByIndex(0);
```

## 前缀匹配与目录列表

### GetFirstIndexStartsWith() 方法

当引擎需要列出归档中某个"目录"下的所有文件时（比如列出 `bgimage/` 下的所有文件），不能用哈希表——哈希表只支持精确匹配。

`tTVPArchive` 维护了一个按字母排序的文件名列表（`Names`），并提供 `GetFirstIndexStartsWith()` 方法通过**二分查找**来找到第一个匹配前缀的文件：

```cpp
// 文件路径：cpp/core/base/StorageIntf.h（简化版本）
// 二分查找：找到第一个以指定前缀开头的文件名
tjs_int tTVPArchive::GetFirstIndexStartsWith(
    const ttstr &prefix) const {

    if (Names.empty()) return -1;

    // 使用标准库的 lower_bound 进行二分查找
    // lower_bound 返回第一个 >= prefix 的元素
    auto it = std::lower_bound(
        Names.begin(), Names.end(), prefix);

    if (it == Names.end()) return -1;

    // 检查找到的元素是否真的以 prefix 开头
    if (it->StartsWith(prefix)) {
        return (tjs_int)(it - Names.begin());
    }

    return -1;  // 没有匹配的前缀
}
```

使用示例：

```cpp
// 列出 bgimage/ 目录下的所有文件
ttstr prefix = TJS_W("bgimage/");
tjs_int startIdx = archive->GetFirstIndexStartsWith(prefix);

if (startIdx >= 0) {
    // 从 startIdx 开始，逐个检查后续文件是否仍以 prefix 开头
    for (tjs_uint i = startIdx; i < archive->GetCount(); i++) {
        ttstr name = archive->GetName(i);
        if (!name.StartsWith(prefix)) break;  // 不再匹配，结束
        // name 是 bgimage/ 下的文件
        std::cout << name << std::endl;
    }
}

// 输出：
// bgimage/bg01.png
// bgimage/bg02.png
```

> **为什么二分查找可以工作？** 因为 `Names` 列表是按字母排序的。以相同前缀开头的文件名在排序后一定是连续排列的（比如所有 "bgimage/" 开头的文件名会排在一起）。所以只需要找到第一个匹配的位置，然后向后遍历到不匹配为止。

## 纯虚函数接口

### CreateStreamByIndex() — 唯一的纯虚函数

`tTVPArchive` 基类只定义了一个纯虚函数：`CreateStreamByIndex()`。这是整个归档系统中最核心的方法——它的职责是：**给定一个文件索引号，返回一个可以读取该文件数据的流对象**。

每种归档格式的实现方式完全不同：

```cpp
// XP3 实现：需要处理分段压缩和解密
// 文件路径：cpp/core/base/XP3Archive.cpp
tTJSBinaryStream* tTVPXP3Archive::CreateStreamByIndex(
    tjs_uint index) {
    // 1. 查找文件的段信息（可能有多个压缩段）
    // 2. 创建 tTVPXP3ArchiveStream（专用的分段读取流）
    // 3. 如果有提取过滤器，包装解密层
    return new tTVPXP3ArchiveStream(this, index);
}

// ZIP 实现：需要定位本地文件头并 inflate 解压
// 文件路径：cpp/core/base/ZIPArchive.cpp
tTJSBinaryStream* TVPZIPArchive::CreateStreamByIndex(
    tjs_uint index) {
    // 1. 从中央目录获取文件的本地偏移
    // 2. 读取本地文件头
    // 3. 解压数据到内存流
    return new tTVPMemoryStream(decompressedData, size);
}

// TAR 实现：最简单——直接偏移读取
// 文件路径：cpp/core/base/TARArchive.cpp
tTJSBinaryStream* TVPTARArchive::CreateStreamByIndex(
    tjs_uint index) {
    // TAR 文件不压缩，直接用偏移+长度创建部分流
    return new TArchiveStream(
        archiveStream, offset, size);
}

// 7z 实现：使用 LZMA SDK 解压
// 文件路径：cpp/core/base/7zArchive.cpp
tTJSBinaryStream* TVP7zArchive::CreateStreamByIndex(
    tjs_uint index) {
    // 如果是单编码器未压缩文件夹，直接偏移读取
    // 否则，通过 LZMA SDK 解压到内存
    return new tTVPMemoryStream(data, size);
}
```

### 工厂方法模式

`tTVPArchive` 的使用是一个典型的**工厂方法模式**（Factory Method Pattern）。引擎根据归档文件的扩展名或魔数（Magic Number，文件开头的固定字节序列，用于标识文件格式）来决定创建哪种具体的归档对象：

```cpp
// 文件路径：cpp/core/base/StorageIntf.cpp（简化版本）
// TVPOpenArchive() — 打开归档文件的工厂方法
tTVPArchive* TVPOpenArchive(const ttstr &name) {
    // 先检查归档缓存
    tTVPArchive *cached = TVPArchiveCache.Find(name);
    if (cached) {
        cached->AddRef();  // 增加引用计数
        return cached;
    }

    // 缓存未命中，需要打开新归档
    // 读取文件头部以判断格式
    tTJSBinaryStream *stream = TVPCreateFileStream(name);

    // 读取前 11 字节判断格式
    tjs_uint8 header[11];
    stream->Read(header, 11);
    stream->Seek(0, TJS_BS_SEEK_SET);  // 重置到开头

    tTVPArchive *archive = nullptr;

    // 根据魔数判断格式
    if (memcmp(header, "XP3\r\n \n\x1a\x8b\x67\x01", 11) == 0) {
        // XP3 格式魔数匹配
        archive = new tTVPXP3Archive(name, stream);
    } else if (header[0] == 'P' && header[1] == 'K') {
        // ZIP 格式魔数（PK\x03\x04）
        archive = new TVPZIPArchive(name, stream);
    } else if (header[0] == '7' && header[1] == 'z') {
        // 7z 格式魔数
        archive = new TVP7zArchive(name, stream);
    } else {
        // 尝试 TAR（TAR 没有统一魔数，通过头部校验和判断）
        archive = new TVPTARArchive(name, stream);
    }

    // 添加到缓存
    if (archive) {
        TVPArchiveCache.Add(archive);
    }

    return archive;
}
```

## 动手实践

### 练习：实现一个简化版的 tTVPArchive

下面是一个自包含的完整示例，模拟 `tTVPArchive` 的核心功能：

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <unordered_map>
#include <algorithm>
#include <cstring>
#include <sstream>

// 简化版二进制流
class SimpleStream {
    std::vector<uint8_t> data_;
    size_t pos_ = 0;
public:
    SimpleStream(const std::vector<uint8_t> &data)
        : data_(data) {}

    size_t Read(void *buf, size_t size) {
        size_t readable = std::min(size, data_.size() - pos_);
        std::memcpy(buf, data_.data() + pos_, readable);
        pos_ += readable;
        return readable;
    }

    size_t GetSize() const { return data_.size(); }

    // 转为字符串（方便打印）
    std::string ToString() {
        return std::string(data_.begin(), data_.end());
    }
};

// 简化版归档基类（模拟 tTVPArchive 的核心设计）
class SimpleArchive {
    unsigned int refCount_;  // 引用计数
    std::vector<std::string> names_;  // 文件名有序列表
    std::unordered_map<std::string, unsigned int> nameHash_;
    // 哈希索引

protected:
    std::string archiveName_;  // 归档路径

    // 子类调用：添加文件到索引
    void AddFileEntry(const std::string &name) {
        unsigned int index =
            static_cast<unsigned int>(names_.size());
        std::string normalized = NormalizeName(name);
        names_.push_back(normalized);
        nameHash_[normalized] = index;
    }

    // 归档内部路径规范化
    static std::string NormalizeName(const std::string &name) {
        std::string result = name;
        for (auto &c : result) {
            if (c == '\\') c = '/';       // 反斜杠转正斜杠
            c = std::tolower(static_cast<unsigned char>(c));
            // 全部小写
        }
        return result;
    }

public:
    SimpleArchive() : refCount_(1) {}
    virtual ~SimpleArchive() = default;

    // 引用计数操作
    void AddRef() { refCount_++; }
    void Release() {
        if (refCount_ <= 1) {
            delete this;
        } else {
            refCount_--;
        }
    }
    unsigned int GetRefCount() const { return refCount_; }

    // 文件计数
    unsigned int GetCount() const {
        return static_cast<unsigned int>(names_.size());
    }

    // 获取文件名
    std::string GetName(unsigned int index) const {
        return names_[index];
    }

    // 哈希查找
    int GetIndex(const std::string &name) const {
        std::string normalized = NormalizeName(name);
        auto it = nameHash_.find(normalized);
        if (it != nameHash_.end()) {
            return static_cast<int>(it->second);
        }
        return -1;
    }

    // 前缀匹配（二分查找）
    int GetFirstIndexStartsWith(
        const std::string &prefix) const {
        if (names_.empty()) return -1;
        std::string normPrefix = NormalizeName(prefix);
        auto it = std::lower_bound(
            names_.begin(), names_.end(), normPrefix);
        if (it != names_.end() &&
            it->substr(0, normPrefix.size()) == normPrefix) {
            return static_cast<int>(it - names_.begin());
        }
        return -1;
    }

    // 纯虚函数：子类必须实现
    virtual SimpleStream* CreateStreamByIndex(
        unsigned int index) = 0;

    // 便捷方法：按名称创建流
    SimpleStream* CreateStream(const std::string &name) {
        int index = GetIndex(name);
        if (index < 0) {
            throw std::runtime_error(
                "文件未找到: " + name);
        }
        return CreateStreamByIndex(
            static_cast<unsigned int>(index));
    }
};

// 具体实现：内存归档（所有文件存在内存中）
class MemoryArchive : public SimpleArchive {
    std::vector<std::vector<uint8_t>> fileData_;
    // 每个文件的数据

public:
    MemoryArchive(const std::string &name) {
        archiveName_ = name;
    }

    // 添加文件到归档
    void AddFile(const std::string &name,
                const std::string &content) {
        AddFileEntry(name);
        fileData_.push_back(
            std::vector<uint8_t>(
                content.begin(), content.end()));
    }

    // 实现纯虚函数
    SimpleStream* CreateStreamByIndex(
        unsigned int index) override {
        if (index >= fileData_.size()) {
            throw std::runtime_error("索引越界");
        }
        return new SimpleStream(fileData_[index]);
    }
};

int main() {
    // 创建内存归档
    auto *archive = new MemoryArchive("test.xp3");

    // 添加测试文件
    archive->AddFile("BGImage\\BG01.PNG",
                    "这是背景图片 1 的数据");
    archive->AddFile("BGImage\\BG02.PNG",
                    "这是背景图片 2 的数据");
    archive->AddFile("Scenario\\Prologue.ks",
                    "@text 这是序章脚本");
    archive->AddFile("Scenario\\Chapter01.ks",
                    "@text 这是第一章脚本");

    std::cout << "=== 归档信息 ===" << std::endl;
    std::cout << "文件数量: " << archive->GetCount()
              << std::endl;
    std::cout << "引用计数: " << archive->GetRefCount()
              << std::endl;

    // 测试哈希查找
    std::cout << "\n=== 哈希查找测试 ===" << std::endl;
    int idx = archive->GetIndex("bgimage/bg01.png");
    std::cout << "查找 'bgimage/bg01.png': 索引 = "
              << idx << std::endl;

    // 测试大小写不敏感查找
    idx = archive->GetIndex("BGIMAGE\\BG01.PNG");
    std::cout << "查找 'BGIMAGE\\BG01.PNG': 索引 = "
              << idx << std::endl;

    // 测试流创建
    std::cout << "\n=== 流读取测试 ===" << std::endl;
    auto *stream = archive->CreateStream("bgimage/bg01.png");
    std::cout << "内容: " << stream->ToString() << std::endl;
    delete stream;

    // 测试前缀匹配
    std::cout << "\n=== 前缀匹配测试 ===" << std::endl;
    int startIdx = archive->GetFirstIndexStartsWith(
        "scenario/");
    if (startIdx >= 0) {
        std::cout << "scenario/ 目录下的文件:" << std::endl;
        for (unsigned int i = startIdx;
             i < archive->GetCount(); i++) {
            std::string name = archive->GetName(i);
            if (name.substr(0, 9) != "scenario/") break;
            std::cout << "  " << name << std::endl;
        }
    }

    // 测试引用计数
    std::cout << "\n=== 引用计数测试 ===" << std::endl;
    std::cout << "初始引用计数: "
              << archive->GetRefCount() << std::endl;
    archive->AddRef();
    std::cout << "AddRef 后: "
              << archive->GetRefCount() << std::endl;
    archive->Release();
    std::cout << "Release 后: "
              << archive->GetRefCount() << std::endl;
    archive->Release();  // 引用归零，对象被释放
    // 此后不能再访问 archive！

    return 0;
}
```

编译并运行：

```bash
# Linux/macOS
g++ -std=c++17 -o archive_demo archive_demo.cpp && ./archive_demo

# Windows (MSVC)
cl /EHsc /std:c++17 archive_demo.cpp && archive_demo.exe
```

预期输出：

```
=== 归档信息 ===
文件数量: 4
引用计数: 1

=== 哈希查找测试 ===
查找 'bgimage/bg01.png': 索引 = 0
查找 'BGIMAGE\BG01.PNG': 索引 = 0

=== 流读取测试 ===
内容: 这是背景图片 1 的数据

=== 前缀匹配测试 ===
scenario/ 目录下的文件:
  scenario/chapter01.ks
  scenario/prologue.ks

=== 引用计数测试 ===
初始引用计数: 1
AddRef 后: 2
Release 后: 1
```

## 对照项目源码

本节涉及的核心源码文件：

| 文件路径 | 行号范围 | 内容说明 |
|---------|----------|---------|
| `cpp/core/base/StorageIntf.h` | 第 61-120 行 | `tTVPArchive` 类完整声明 |
| `cpp/core/base/StorageIntf.h` | 第 82-95 行 | 引用计数相关方法（`AddRef()`/`Release()`） |
| `cpp/core/base/StorageIntf.h` | 第 96-115 行 | 哈希索引和文件查找方法 |
| `cpp/core/base/XP3Archive.cpp` | 第 1-50 行 | `tTVPXP3Archive` 构造函数（调用 `AddFileEntry()`） |
| `cpp/core/base/StorageIntf.cpp` | 第 200-300 行 | `TVPOpenArchive()` 工厂方法实现 |

## 常见错误与排查

### 错误 1：引用计数不匹配导致内存泄漏

**症状**：运行时间越长内存占用越大，直到 OOM（Out of Memory，内存不足）

**原因**：某处调用了 `AddRef()` 但忘记对应的 `Release()`

**排查方法**：在 `AddRef()` 和 `Release()` 中添加日志，检查是否成对出现

### 错误 2：引用计数过度释放导致崩溃

**症状**：访问归档对象时程序崩溃（Segmentation Fault / Access Violation）

**原因**：多调用了一次 `Release()`，对象被过早释放

**排查方法**：在 `Release()` 中添加断言 `assert(RefCount > 0)`

### 错误 3：文件名大小写不匹配

**症状**：`GetIndex()` 返回 -1，但文件确实存在于归档中

**原因**：传入的文件名没有经过规范化，或者归档构建时没有规范化

**排查方法**：检查 `AddFileEntry()` 是否调用了 `NormalizeInArchiveStorageName()`

## 本节小结

- `tTVPArchive` 是所有归档格式的**抽象基类**，定义了统一的文件查找和流创建接口
- **引用计数**管理归档对象的生命周期——`AddRef()` 增加、`Release()` 减少，归零时自动释放
- **哈希索引**实现 O(1) 的精确文件名查找，**二分查找**实现前缀匹配（用于目录列表）
- 唯一的纯虚函数 `CreateStreamByIndex()` 由四种格式子类各自实现
- **工厂方法模式**根据文件魔数自动选择正确的子类进行实例化

## 练习题与答案

### 题目 1：引用计数生命周期

以下代码执行后，`archive` 对象是否会被正确释放？如果不会，指出问题所在。

```cpp
tTVPArchive *archive = TVPOpenArchive("data.xp3");
// RefCount = 1

archive->AddRef();   // 模拟缓存持有引用
// RefCount = 2

auto *stream1 = archive->CreateStream("bg01.png");
archive->AddRef();   // 模拟流持有引用
// RefCount = 3

delete stream1;
// 忘记调用 archive->Release()

archive->Release();  // 缓存释放引用
// RefCount = ?

archive->Release();  // 最后释放
// RefCount = ?
```

<details>
<summary>查看答案</summary>

**不会正确释放。**

执行过程：
1. `TVPOpenArchive()` 后：`RefCount = 1`
2. `AddRef()` 后：`RefCount = 2`
3. 第二次 `AddRef()` 后：`RefCount = 3`
4. `delete stream1` 后：没有调用 `archive->Release()`，所以 `RefCount` 仍为 3
5. 第一次 `Release()` 后：`RefCount = 2`
6. 第二次 `Release()` 后：`RefCount = 1`

最终 `RefCount = 1`，对象不会被释放——**内存泄漏！**

**修复方法**：在 `delete stream1` 之后添加 `archive->Release()`，或者让流的析构函数自动调用 `archive->Release()`。

</details>

### 题目 2：哈希索引查找顺序

以下归档包含 5 个文件，按添加顺序为：

```
"bgimage/bg01.png"
"scenario/main.ks"
"bgimage/bg02.png"
"sound/bgm01.ogg"
"scenario/sub.ks"
```

回答以下问题：
1. `GetIndex("scenario/main.ks")` 返回什么？
2. `GetFirstIndexStartsWith("bgimage/")` 返回什么？
3. 如果文件名是 `"BGImage/BG01.PNG"`，`GetIndex()` 能找到吗？

<details>
<summary>查看答案</summary>

1. `GetIndex("scenario/main.ks")` 返回 **1**。
   - 文件按添加顺序编号：bgimage/bg01.png=0, scenario/main.ks=1, bgimage/bg02.png=2, sound/bgm01.ogg=3, scenario/sub.ks=4
   - 哈希表精确匹配，直接返回索引 1

2. `GetFirstIndexStartsWith("bgimage/")` 返回 **0**。
   - 但注意：`Names` 列表需要排序才能使二分查找正常工作
   - 排序后：bgimage/bg01.png, bgimage/bg02.png, scenario/main.ks, scenario/sub.ks, sound/bgm01.ogg
   - `lower_bound("bgimage/")` 找到第一个 >= "bgimage/" 的元素，即 "bgimage/bg01.png"，排序后索引为 0

3. **能找到。**
   - `GetIndex()` 内部会先调用 `NormalizeInArchiveStorageName()` 把 "BGImage/BG01.PNG" 转为 "bgimage/bg01.png"
   - 然后在哈希表中查找，匹配成功

</details>

### 题目 3：设计一个自定义归档基类的改进

当前的 `tTVPArchive` 使用手动引用计数。请用 `std::shared_ptr` 改写引用计数部分，并说明需要注意什么。

<details>
<summary>查看答案</summary>

```cpp
#include <memory>
#include <iostream>

// 使用 enable_shared_from_this 替代手动引用计数
class ModernArchive
    : public std::enable_shared_from_this<ModernArchive> {

public:
    ModernArchive() {
        std::cout << "归档对象创建" << std::endl;
    }

    virtual ~ModernArchive() {
        std::cout << "归档对象释放" << std::endl;
    }

    // 不再需要 AddRef() / Release()
    // shared_ptr 自动管理生命周期

    // 获取引用计数（仅用于调试）
    long GetRefCount() const {
        // weak_from_this() 可以获取当前的引用计数
        // 注意：这只在对象被 shared_ptr 管理时有效
        return 0; // 实际可通过 shared_ptr 的 use_count() 获取
    }

    virtual void PrintInfo() = 0;
};

class ModernXP3Archive : public ModernArchive {
    std::string name_;
public:
    ModernXP3Archive(const std::string &name)
        : name_(name) {}

    void PrintInfo() override {
        std::cout << "XP3 归档: " << name_ << std::endl;
    }
};

int main() {
    // 使用 shared_ptr 管理归档对象
    auto archive =
        std::make_shared<ModernXP3Archive>("data.xp3");
    std::cout << "引用计数: " << archive.use_count()
              << std::endl;  // 输出: 1

    {
        auto copy = archive;  // 自动 AddRef
        std::cout << "引用计数: " << archive.use_count()
                  << std::endl;  // 输出: 2
    }
    // copy 离开作用域，自动 Release
    std::cout << "引用计数: " << archive.use_count()
              << std::endl;  // 输出: 1

    archive.reset();  // 最后一个引用释放，对象自动销毁
    // 输出: 归档对象释放

    return 0;
}
```

**注意事项**：

1. **不能用 `new` 创建对象**：必须用 `std::make_shared<>()` 创建，否则 `shared_from_this()` 会出错
2. **循环引用问题**：如果归档对象和流对象互相持有 `shared_ptr`，会导致内存泄漏。需要使用 `std::weak_ptr` 打破循环
3. **性能开销**：`shared_ptr` 的引用计数操作是原子的（线程安全），比手动 `++/--` 稍慢。对于高频操作可能有影响
4. **兼容性**：改用 `shared_ptr` 需要修改所有使用归档指针的代码，工作量较大

编译并运行：

```bash
# Linux/macOS
g++ -std=c++17 -o modern_archive modern_archive.cpp && ./modern_archive

# Windows (MSVC)
cl /EHsc /std:c++17 modern_archive.cpp && modern_archive.exe
```

</details>

## 下一步

下一节 [归档格式对比与选择](./03-归档格式对比与选择.md) 将对 KrKr2 支持的四种归档格式（XP3、ZIP、TAR、7z）进行全面比较——包括文件结构、压缩效率、随机访问能力、适用场景等，帮助你理解引擎为什么需要支持多种格式，以及每种格式的优劣。

