# tTJSBinaryStream 体系

> **所属模块：** M08-归档与IO系统
> **前置知识：** [归档系统概述](../01-归档系统概述/01-KrKr2存储架构总览.md)、[tTVPArchive基类设计](../01-归档系统概述/02-tTVPArchive基类设计.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 `tTJSBinaryStream` 作为整个引擎 IO 基础设施的核心地位
2. 掌握 `Seek`/`Read`/`Write`/`GetSize` 四个纯虚函数的语义与实现约定
3. 理解辅助方法 `SetPosition`/`GetPosition`/`ReadBuffer`/`WriteBuffer`/`ReadI64LE` 等的便捷封装
4. 掌握流模式标志（`TJS_BS_READ`/`WRITE`/`APPEND`/`UPDATE`）的含义与使用场景
5. 理解 `TArchiveStream` 如何在归档文件中实现偏移量+长度的流窗口
6. 掌握 `tTVPStreamHolder` RAII 包装器的用法与所有权管理
7. 能够自己编写一个 `tTJSBinaryStream` 的子类来适配自定义数据源

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 二进制流 | Binary Stream | 以字节为单位读写数据的抽象通道，不关心数据内容是文本还是图片 |
| 纯虚函数 | Pure Virtual Function | C++ 中用 `= 0` 标记的虚函数，强制子类必须实现，否则无法实例化 |
| Seek 操作 | Seek Operation | 移动流内部的"读写指针"到指定位置，类似于 C 标准库的 `fseek()` |
| RAII | Resource Acquisition Is Initialization | C++ 编程范式：在构造函数中获取资源、在析构函数中释放资源，确保资源不泄漏 |
| 流窗口 | Stream Window | 在一个大流中划定一段范围 [start, start+length)，让使用者以为这就是整个流 |
| 模式标志 | Mode Flags | 用位掩码（bitmask）表示的打开方式，如只读、只写、追加、更新等 |
| 字节序 | Byte Order / Endianness | 多字节数据在内存中的排列顺序，小端（Little-Endian）是低位字节在前 |

## 为什么需要统一的流抽象？

在一个游戏引擎中，数据来源五花八门：

```
┌─────────────────────────────────────────────────────┐
│                   数据消费者                         │
│  图片加载器 / 音频解码器 / 脚本编译器 / 存档读取    │
└───────────────────────┬─────────────────────────────┘
                        │ 统一接口
                        ▼
              ┌─────────────────────┐
              │  tTJSBinaryStream   │  ← 抽象基类
              └─────────┬───────────┘
          ┌─────────────┼──────────────────┐
          ▼             ▼                  ▼
   ┌──────────┐  ┌─────────────┐  ┌───────────────┐
   │ 磁盘文件 │  │ 归档内文件  │  │   内存块      │
   │ (file:)  │  │ (XP3/ZIP等) │  │ (MemoryStream)│
   └──────────┘  └─────────────┘  └───────────────┘
```

如果没有统一的流抽象，每个数据消费者（Consumer，指需要读取数据的模块）都要知道数据来自哪里——是磁盘文件？是压缩包里的某个条目？还是内存中的一块缓冲区？这会导致消费者代码充满 `if/else` 分支，耦合度极高。

`tTJSBinaryStream` 就是 KrKr2 引擎的"流抽象基类"，它定义了 4 个纯虚函数（Seek、Read、Write、GetSize），所有数据源都实现这 4 个接口。消费者只需要面对 `tTJSBinaryStream*` 指针，完全不关心底层实现。

## tTJSBinaryStream 类定义

### 源码位置

`tTJSBinaryStream` 定义在 TJS2 脚本引擎的头文件中：

```
cpp/core/tjs2/tjs.h  第 263-299 行
```

这个类定义在 `TJS` 命名空间（namespace）内，但由于引擎全局使用了 `using namespace TJS`，所以在 base 模块的代码中可以直接使用 `tTJSBinaryStream` 而不需要写 `TJS::tTJSBinaryStream`。

### 完整类声明

```cpp
// 源码：cpp/core/tjs2/tjs.h 第 263-299 行
class tTJSBinaryStream {
public:
    //-- 必须实现的纯虚函数
    virtual tjs_uint64 Seek(tjs_int64 offset, tjs_int whence) = 0;
    /* if error, position is not changed */

    virtual tjs_uint Read(void *buffer, tjs_uint read_size) = 0;
    /* returns actually read size */

    virtual tjs_uint Write(const void *buffer, tjs_uint write_size) = 0;
    /* returns actually written size */

    virtual void SetEndOfStorage();
    // the default behavior is raising a exception

    virtual tjs_uint64 GetSize() = 0;

    virtual ~tTJSBinaryStream() = default;

    //-- 辅助方法（已在基类实现，子类无需重写）
    tjs_uint64 GetPosition();       // 获取当前位置
    void SetPosition(tjs_uint64 pos); // 设置当前位置

    void ReadBuffer(void *buffer, tjs_uint read_size);  // 读取并检查完整性
    void WriteBuffer(const void *buffer, tjs_uint write_size); // 写入并检查完整性

    tjs_uint64 ReadI64LE();  // 读取 64 位小端整数
    tjs_uint32 ReadI32LE();  // 读取 32 位小端整数
    tjs_uint16 ReadI16LE();  // 读取 16 位小端整数
    tjs_uint8  ReadI8LE();   // 读取 8 位整数
};
```

### 设计解读

这个基类的设计遵循了几个重要原则：

**1. 最小接口原则**

只有 4 个纯虚函数（`Seek`、`Read`、`Write`、`GetSize`）需要子类实现。这是能支撑所有操作的最小集合——有了这四个，所有其他操作（获取位置、设置位置、按字节序读整数等）都可以在基类中组合实现。

**2. 可选覆盖**

`SetEndOfStorage()` 是虚函数但不是纯虚的（没有 `= 0`），默认实现会抛出异常。只有支持截断操作的流（如文件流、内存流）需要覆盖它。只读流（如 `TArchiveStream`、`tTVPPartialStream`）可以直接使用默认行为。

**3. 虚析构函数**

`virtual ~tTJSBinaryStream() = default;` 确保通过基类指针 `delete` 时，能正确调用子类的析构函数。这是 C++ 多态类的标准做法——如果析构函数不是虚的，`delete basePtr` 只会调用基类析构，子类中分配的资源就泄漏了。

## 四个纯虚函数详解

### Seek — 移动读写位置

```cpp
virtual tjs_uint64 Seek(tjs_int64 offset, tjs_int whence) = 0;
```

`Seek` 函数的作用是移动流内部的"读写指针"（也叫文件游标、位置指示器）。它的语义与 C 标准库的 `fseek()` 完全一致。

**参数说明：**

| 参数 | 类型 | 含义 |
|------|------|------|
| `offset` | `tjs_int64` (有符号64位) | 偏移量，可正可负。正值向后移动，负值向前移动 |
| `whence` | `tjs_int` | 基准位置，决定 `offset` 从哪里开始计算 |

**`whence` 取值：**

```cpp
// 源码：cpp/core/tjs2/tjs.h 第 253-255 行
#define TJS_BS_SEEK_SET  SEEK_SET   // 0: 从流的开头开始计算
#define TJS_BS_SEEK_CUR  SEEK_CUR   // 1: 从当前位置开始计算
#define TJS_BS_SEEK_END  SEEK_END   // 2: 从流的末尾开始计算
```

这三个宏直接映射到 C 标准库的 `SEEK_SET`/`SEEK_CUR`/`SEEK_END`，语义完全相同：

```
TJS_BS_SEEK_SET:  新位置 = offset
TJS_BS_SEEK_CUR:  新位置 = 当前位置 + offset
TJS_BS_SEEK_END:  新位置 = 流大小 + offset
```

**返回值：** 移动后的绝对位置（从流开头算起的字节偏移）。如果移动失败，位置不变，返回当前位置。

**实现约定：** 子类实现时应注意边界检查——不要让位置变成负数，也不要超过流的大小。看看 `tTVPMemoryStream` 的实现就能理解这种约定：

```cpp
// 源码：cpp/core/base/UtilStreams.cpp 第 113-140 行
tjs_uint64 tTVPMemoryStream::Seek(tjs_int64 offset, tjs_int whence) {
    tjs_int64 newpos;
    switch(whence) {
        case TJS_BS_SEEK_SET:
            if(offset >= 0) {
                if(offset <= Size)             // 不能超过末尾
                    CurrentPos = static_cast<tjs_uint>(offset);
            }
            return CurrentPos;

        case TJS_BS_SEEK_CUR:
            if((newpos = offset + (tjs_int64)CurrentPos) >= 0) {  // 不能为负
                tjs_uint np = (tjs_uint)newpos;
                if(np <= Size)                 // 不能超过末尾
                    CurrentPos = np;
            }
            return CurrentPos;

        case TJS_BS_SEEK_END:
            if((newpos = offset + (tjs_int64)Size) >= 0) {  // 从末尾往前算
                tjs_uint np = (tjs_uint)newpos;
                if(np <= Size)
                    CurrentPos = np;
            }
            return CurrentPos;
    }
    return CurrentPos;
}
```

注意这段代码的关键细节：如果 `offset` 不合法（如 `SEEK_SET` 时为负数、或结果超过 `Size`），**什么都不做**，直接返回 `CurrentPos`。这就是注释中 "if error, position is not changed" 的含义。

### Read — 读取数据

```cpp
virtual tjs_uint Read(void *buffer, tjs_uint read_size) = 0;
```

从流的当前位置读取最多 `read_size` 个字节到 `buffer` 中。

**参数说明：**

| 参数 | 类型 | 含义 |
|------|------|------|
| `buffer` | `void*` | 输出缓冲区，调用者负责分配足够的空间 |
| `read_size` | `tjs_uint` (无符号32位) | 期望读取的字节数 |

**返回值：** 实际读取的字节数。可能小于 `read_size`（到达流末尾时）。

**关键语义：** 读取后，流的当前位置自动前移实际读取的字节数。如果返回 0，表示已经到达流末尾。

```cpp
// 源码：cpp/core/base/UtilStreams.cpp 第 143-153 行
tjs_uint tTVPMemoryStream::Read(void *buffer, tjs_uint read_size) {
    if(CurrentPos + read_size >= Size) {
        read_size = Size - CurrentPos;   // 截断到剩余大小
    }

    memcpy(buffer, (tjs_uint8 *)Block + CurrentPos, read_size);  // 复制数据

    CurrentPos += read_size;  // 前移游标

    return read_size;   // 返回实际读取的字节数
}
```

### Write — 写入数据

```cpp
virtual tjs_uint Write(const void *buffer, tjs_uint write_size) = 0;
```

将 `buffer` 中的 `write_size` 个字节写入流的当前位置。

**参数说明：**

| 参数 | 类型 | 含义 |
|------|------|------|
| `buffer` | `const void*` | 输入缓冲区，包含要写入的数据 |
| `write_size` | `tjs_uint` | 要写入的字节数 |

**返回值：** 实际写入的字节数。

**注意事项：** 只读流（如 `TArchiveStream`、`tTVPPartialStream`）的 `Write` 实现直接返回 0，表示不支持写入：

```cpp
// 源码：cpp/core/base/StorageIntf.h 第 376-378 行
// TArchiveStream::Write
tjs_uint Write(const void *buffer, tjs_uint write_size) override {
    return 0;  // 归档内文件只读，不支持写入
}
```

而可写流（如 `tTVPMemoryStream`）的 `Write` 实现会自动扩容内部缓冲区：

```cpp
// 源码：cpp/core/base/UtilStreams.cpp 第 156-195 行（简化）
tjs_uint tTVPMemoryStream::Write(const void *buffer, tjs_uint write_size) {
    if(Reference)  // 引用模式下不允许写入
        TVPThrowExceptionMessage(TVPWriteError);

    tjs_uint newpos = CurrentPos + write_size;
    if(newpos >= AllocSize) {
        // 需要扩容：根据当前大小选择增量
        tjs_uint onesize;
        if(AllocSize < 64 * 1024)       onesize = 4 * 1024;     // 4KB
        else if(AllocSize < 512 * 1024) onesize = 16 * 1024;    // 16KB
        else if(AllocSize < 4096 * 1024) onesize = 256 * 1024;  // 256KB
        else                             onesize = 2024 * 1024;  // ~2MB
        AllocSize += onesize;

        if(CurrentPos + write_size >= AllocSize)  // 增量仍不够？
            AllocSize = CurrentPos + write_size;  // 精确分配

        Block = Realloc(Block, AllocSize);
        if(AllocSize && !Block)
            TVPThrowExceptionMessage(TVPInsufficientMemory);
    }

    memcpy((tjs_uint8 *)Block + CurrentPos, buffer, write_size);
    CurrentPos = newpos;
    if(CurrentPos > Size) Size = CurrentPos;  // 更新流大小

    return write_size;
}
```

这个扩容策略值得注意：**增量随当前大小递增**。小文件用小增量（4KB），大文件用大增量（2MB），在内存效率和分配次数之间取了平衡。

### GetSize — 获取流大小

```cpp
virtual tjs_uint64 GetSize() = 0;
```

返回流的总大小（字节数）。这个函数不改变当前位置。

**返回值类型是 `tjs_uint64`**（无符号64位），理论上支持到 16 EB（Exabyte，百亿亿字节）大小的流。在实际使用中，一般游戏资源不会超过几个 GB。

## 辅助方法详解

基类中的辅助方法都是对四个纯虚函数的封装，子类不需要（也不应该）重写它们。

### GetPosition 和 SetPosition

```cpp
tjs_uint64 GetPosition();
void SetPosition(tjs_uint64 pos);
```

这两个方法是对 `Seek` 的简单封装：

```cpp
// GetPosition 的等价实现：
tjs_uint64 GetPosition() {
    return Seek(0, TJS_BS_SEEK_CUR);  // 偏移 0，从当前位置算 → 返回当前位置
}

// SetPosition 的等价实现：
void SetPosition(tjs_uint64 pos) {
    Seek(pos, TJS_BS_SEEK_SET);  // 从开头偏移 pos 字节
}
```

为什么需要这两个封装？因为 `Seek(0, TJS_BS_SEEK_CUR)` 这种写法不直观，新手看到会一头雾水。而 `GetPosition()` 意图一目了然。这是接口设计中"说人话"的原则。

### ReadBuffer 和 WriteBuffer

```cpp
void ReadBuffer(void *buffer, tjs_uint read_size);
void WriteBuffer(const void *buffer, tjs_uint write_size);
```

这两个方法与 `Read`/`Write` 的区别在于：**它们会检查实际读写的字节数是否等于请求的字节数**。如果不等，说明出错了（比如读到文件末尾但还有数据未读完），会抛出异常。

```
Read(buf, 100)     → 可能只读了 50 字节（返回 50），调用者需要自己检查
ReadBuffer(buf, 100) → 如果只读了 50 字节，直接抛异常
```

**使用场景选择：**

- 知道数据一定够（如读取文件头、读取固定长度的结构体）→ 用 `ReadBuffer`
- 不确定数据是否够（如循环读取直到 EOF）→ 用 `Read`

### 小端整数读取系列

```cpp
tjs_uint64 ReadI64LE();  // 读 8 字节，解释为 64 位小端无符号整数
tjs_uint32 ReadI32LE();  // 读 4 字节，解释为 32 位小端无符号整数
tjs_uint16 ReadI16LE();  // 读 2 字节，解释为 16 位小端无符号整数
tjs_uint8  ReadI8LE();   // 读 1 字节
```

这些方法读取指定字节数的数据，并按小端字节序（Little-Endian，低位字节在前）解释为整数。KrKr2 引擎中几乎所有二进制格式（XP3、TJS2 字节码等）都使用小端格式，所以这些方法使用频率非常高。

**什么是小端字节序（Little-Endian）？** 多字节整数在文件/内存中的存放顺序——低位字节在前（低地址），高位字节在后（高地址）。例如 32 位整数 `0x12345678` 在小端序下的字节排列是：

```
地址  →  低 ──────────────── 高
字节     0x78  0x56  0x34  0x12
         ↑最低有效字节       ↑最高有效字节
```

x86/x64 处理器和 ARM 处理器（默认模式）都使用小端序，所以在这些平台上可以直接用 `memcpy` 把字节拷贝到整数变量中。`ReadI32LE()` 的典型实现就是这样做的：

```cpp
// ReadI32LE 等价实现：
tjs_uint32 ReadI32LE() {
    tjs_uint8 buf[4];
    ReadBuffer(buf, 4);  // 读 4 字节
    return (tjs_uint32)buf[0]
         | ((tjs_uint32)buf[1] << 8)
         | ((tjs_uint32)buf[2] << 16)
         | ((tjs_uint32)buf[3] << 24);
}
```

## 流模式标志

打开一个流时，需要指定访问模式。KrKr2 用宏定义了一组模式标志：

```cpp
// 源码：cpp/core/tjs2/tjs.h 第 243-251 行
#define TJS_BS_READ          0     // 只读模式
#define TJS_BS_WRITE         1     // 只写模式（创建新文件或清空已有文件）
#define TJS_BS_APPEND        2     // 追加模式（在文件末尾写入）
#define TJS_BS_UPDATE        3     // 更新模式（读写，不清空已有内容）

#define TJS_BS_DELETE_ON_CLOSE  0x10  // 关闭时删除文件

#define TJS_BS_ACCESS_MASK   0x0f  // 访问模式掩码（提取低 4 位）
#define TJS_BS_OPTION_MASK   0xf0  // 选项掩码（提取高 4 位）
```

### 访问模式对照表

| 标志 | 值 | 语义 | 对应 C 标准 | 使用场景 |
|------|-----|------|-------------|---------|
| `TJS_BS_READ` | 0 | 只读 | `fopen("rb")` | 加载资源：图片、音频、脚本 |
| `TJS_BS_WRITE` | 1 | 只写（截断） | `fopen("wb")` | 保存存档、导出数据 |
| `TJS_BS_APPEND` | 2 | 追加 | `fopen("ab")` | 日志文件 |
| `TJS_BS_UPDATE` | 3 | 读写 | `fopen("r+b")` | 就地修改文件 |

### 选项标志

`TJS_BS_DELETE_ON_CLOSE`（值 0x10）表示流关闭时自动删除底层文件。这用于临时文件场景——例如从远程存储下载到本地的缓存文件，用完就删。

模式标志通过位或（`|`）组合使用：

```cpp
// 只读打开
auto *stream = TVPCreateStream(name, TJS_BS_READ);

// 只写打开 + 关闭时删除
auto *stream = TVPCreateStream(name, TJS_BS_WRITE | TJS_BS_DELETE_ON_CLOSE);

// 提取访问模式：
tjs_uint32 access = flags & TJS_BS_ACCESS_MASK;  // 得到 0/1/2/3
// 提取选项部分：
tjs_uint32 options = flags & TJS_BS_OPTION_MASK;  // 得到 0x10 等
```

### 模式字符串解析

除了数值标志外，KrKr2 的文本流和二进制流工厂还支持一种字符串模式（mode string），用于 TJS2 脚本层面指定复杂的打开参数。相关解析函数 `parseModeNumber()` 定义在 `BinaryStream.h` 中：

```cpp
// 源码：cpp/core/base/BinaryStream.h 第 22-38 行
inline std::optional<std::uint32_t> parseModeNumber(
    const tjs_char *mode,  // 模式字符串，如 "c1z5o0"
    tjs_char key,          // 模式字符，如 'c', 'z', 'o'
    int maxDigits,         // 最多解析几位数字
    int defaultValue)      // 没有指定该模式时的默认值
{
    if(const tjs_char *p = TJS_strchr(mode, key)) {
        // 找到了模式字符
        if(!(p[1] >= TJS_W('0') && p[1] <= TJS_W('9'))) {
            return {};  // 模式字符后没有数字 → 返回 empty
        }
        int value = 0;
        p++;  // 跳过模式字符本身
        for(int i = 0;
            i < maxDigits && p[i] >= TJS_W('0') && p[i] <= TJS_W('9'); i++) {
            value = value * 10 + (p[i] - TJS_W('0'));
        }
        return value;  // 返回解析出的数字
    }
    return defaultValue;  // 未找到模式字符 → 返回默认值
}
```

支持的模式字符：

| 字符 | 含义 | 示例 |
|------|------|------|
| `o` | 偏移量（offset） | `o1024` — 从第 1024 字节开始读/写 |
| `c` | 加密模式（cipher） | `c1` — 使用简单加密模式 1 |
| `z` | 压缩级别（zlib） | `z5` — 使用 zlib 压缩级别 5 |

这些模式字符在二进制流工厂函数 `TVPCreateBinaryStreamForRead`/`Write` 中使用：

```cpp
// 源码：cpp/core/base/BinaryStream.cpp 第 27-49 行
tTJSBinaryStream *TVPCreateBinaryStreamForRead(
    const ttstr &name, const ttstr &modestr)
{
    tTJSBinaryStream *stream = TVPCreateStream(name, TJS_BS_READ);

    const tjs_char *o_ofs = TJS_strchr(modestr.c_str(), TJS_W('o'));
    if(o_ofs != nullptr) {
        // 有 'o' 模式：解析偏移量数字
        o_ofs++;
        tjs_char buf[256];
        int i;
        for(i = 0; i < 255; i++) {
            if(o_ofs[i] >= TJS_W('0') && o_ofs[i] <= TJS_W('9'))
                buf[i] = o_ofs[i];
            else
                break;
        }
        buf[i] = 0;
        tjs_uint64 ofs = ttstr(buf).AsInteger();
        stream->SetPosition(ofs);  // 跳过指定的字节数
    }
    return stream;
}
```

## TArchiveStream — 归档内文件流

### 是什么？

`TArchiveStream` 是嵌入在 `StorageIntf.h` 中的一个小而精的流类，它在一个已有的底层流上建立一个"窗口"：指定起始偏移量和数据长度，让使用者以为这就是一个独立的流。

**使用场景：** 归档文件（如 TAR、7z）中的每个文件条目都是归档文件的一部分。`TArchiveStream` 把归档文件流中 `[StartPos, StartPos + DataLength)` 这个范围包装成一个独立的流，让消费者可以透明地读取。

```
归档文件：
┌──────┬───────────────────────┬──────────────────┬───────────┐
│ 头部 │  文件A 的数据         │  文件B 的数据    │  文件C... │
│      │  ↑                   ↑│                  │           │
│      │  StartPos    StartPos+DataLength          │           │
└──────┴───────────────────────┴──────────────────┴───────────┘
         └── TArchiveStream 的窗口 ──┘
```

### 源码分析

```cpp
// 源码：cpp/core/base/StorageIntf.h 第 331-381 行
class TArchiveStream : public tTJSBinaryStream {
    tTVPArchive *Owner;           // 所属归档对象（持有引用计数）
    tjs_int64 CurrentPos;         // 流内部的当前位置（相对于窗口起点）
    tjs_uint64 StartPos, DataLength; // 窗口在底层流中的起始位置和长度
    tTJSBinaryStream *_instr;     // 底层流（归档文件的完整流）

public:
    TArchiveStream(tTVPArchive *owner, tjs_uint64 off, tjs_uint64 len);
    ~TArchiveStream() override;

    // Seek：在窗口范围内移动
    tjs_uint64 Seek(tjs_int64 offset, tjs_int whence) override {
        switch(whence) {
            case TJS_BS_SEEK_SET:
                CurrentPos = offset;
                break;
            case TJS_BS_SEEK_CUR:
                CurrentPos = offset + CurrentPos;
                break;
            case TJS_BS_SEEK_END:
                CurrentPos = offset + DataLength; // 注意：以窗口大小为末尾
                break;
        }
        // 边界钳制（clamp）
        if(CurrentPos < 0) CurrentPos = 0;
        else if(CurrentPos > (tjs_int64)DataLength) CurrentPos = DataLength;

        // 关键：将窗口内位置转换为底层流的绝对位置
        _instr->SetPosition(CurrentPos + StartPos);
        return CurrentPos;
    }

    // Read：限制在窗口范围内读取
    tjs_uint Read(void *buffer, tjs_uint read_size) override {
        if(CurrentPos + read_size >= (tjs_int64)DataLength) {
            read_size = (tjs_uint)(DataLength - CurrentPos);  // 截断
        }
        _instr->ReadBuffer(buffer, read_size);  // 从底层流读取
        CurrentPos += read_size;
        return read_size;
    }

    // Write：不支持
    tjs_uint Write(const void *buffer, tjs_uint write_size) override {
        return 0;
    }

    // GetSize：返回窗口大小，而非底层流大小
    tjs_uint64 GetSize() override { return DataLength; }
};
```

### 关键设计要点

**1. 位置转换**

使用者调用 `Seek(50, TJS_BS_SEEK_SET)` 时，看到的是"跳到第 50 字节"。但实际上，`TArchiveStream` 在底层流上执行的是 `_instr->SetPosition(50 + StartPos)`——加上了窗口的起始偏移量。这个转换对使用者完全透明。

**2. 边界钳制**

`Seek` 实现中有一个 clamp（钳制）操作：如果计算出的新位置小于 0，强制设为 0；如果大于 `DataLength`，强制设为 `DataLength`。这防止使用者误操作导致读到窗口外面的数据。

**3. 所有权管理**

`TArchiveStream` 在构造时会对 `Owner`（归档对象）调用 `AddRef()` 增加引用计数，在析构时调用 `Release()` 减少引用计数。这确保只要有流在使用归档文件，归档对象就不会被提前释放。

**4. GetSize 返回窗口大小**

这是流窗口模式的核心——`GetSize()` 返回的是 `DataLength`（窗口大小），而不是底层流的总大小。消费者永远不知道这个"文件"其实是一个大归档文件的一小部分。

## tTVPStreamHolder — RAII 流持有器

### 是什么？

`tTVPStreamHolder` 是一个 RAII（Resource Acquisition Is Initialization，资源获取即初始化）包装器，自动管理 `tTJSBinaryStream*` 的生命周期。

**RAII 的核心思想：** 在构造函数中获取资源（打开流），在析构函数中释放资源（删除流）。利用 C++ 的确定性析构（destructors are called when scope exits），确保无论是正常返回还是异常退出，流都会被正确关闭。

### 源码

```cpp
// 源码：cpp/core/base/UtilStreams.h 第 20-49 行
class tTVPStreamHolder {
    tTJSBinaryStream *Stream;

public:
    // 默认构造：不打开任何流
    tTVPStreamHolder() { Stream = nullptr; }

    // 构造时打开流
    tTVPStreamHolder(const ttstr &name, tjs_uint32 mode = 0) :
        Stream{ TVPCreateStream(name, mode) } {}

    // 析构时自动关闭流
    ~tTVPStreamHolder() { delete Stream; }

    // 箭头操作符：直接调用流的方法
    tTJSBinaryStream *operator->() const { return Stream; }

    // 获取原始指针
    [[nodiscard]] tTJSBinaryStream *Get() const { return Stream; }

    // 手动关闭
    void Close() {
        if(Stream) { delete Stream; Stream = nullptr; }
    }

    // 放弃所有权（不删除流）
    void Disown() { Stream = nullptr; }

    // 重新打开另一个流
    void Open(const ttstr &name, tjs_uint32 flag = 0) {
        if(Stream) delete Stream, Stream = nullptr;
        Stream = TVPCreateStream(name, flag);
    }
};
```

### 使用示例

```cpp
// 不用 tTVPStreamHolder 的写法（需要手动管理生命周期）：
tTJSBinaryStream *stream = TVPCreateStream(name, TJS_BS_READ);
try {
    tjs_uint8 header[16];
    stream->ReadBuffer(header, 16);
    // ... 处理数据 ...
} catch(...) {
    delete stream;   // 必须在异常时也释放！
    throw;
}
delete stream;       // 正常退出也要释放

// 用 tTVPStreamHolder 的写法（RAII，简洁安全）：
tTVPStreamHolder stream(name, TJS_BS_READ);
tjs_uint8 header[16];
stream->ReadBuffer(header, 16);  // 通过 operator-> 直接调用
// ... 处理数据 ...
// 无论正常退出还是异常退出，析构函数自动 delete Stream
```

### Disown 方法

`Disown()` 方法将内部指针设为 `nullptr`，放弃所有权但不释放资源。这用于将流的所有权转移给其他对象的场景：

```cpp
tTVPStreamHolder holder(name, TJS_BS_READ);
tTJSBinaryStream *raw = holder.Get();
// 将流传递给一个会自己管理生命周期的对象
SomeObject *obj = new SomeObject(raw);  // obj 接管流的所有权
holder.Disown();  // 告诉 holder 不要在析构时 delete
```

## 继承体系总览

整个流的继承体系在 KrKr2 引擎中呈"一棵树"结构：

```
tTJSBinaryStream  (抽象基类，定义在 tjs2/tjs.h)
├── TArchiveStream        (归档内文件流 — 偏移+长度窗口)
│   定义在 StorageIntf.h
│
├── tTVPMemoryStream      (内存流 — 可增长的内存缓冲区)
│   定义在 UtilStreams.h/cpp
│
├── tTVPPartialStream     (部分流 — 只读流窗口)
│   定义在 UtilStreams.h/cpp
│
├── tTVPXP3ArchiveStream  (XP3 归档流 — 分段解压)
│   定义在 XP3Archive.h/cpp
│
├── 平台文件流            (磁盘文件 — 各平台实现)
│   定义在 impl/StorageImpl.*
│
└── SevenZipStreamWrap    (7z I/O 适配器)
    定义在 7zArchive.cpp
```

### 各子类的定位

| 子类 | 可读 | 可写 | 可Seek | 典型用途 |
|------|------|------|--------|---------|
| TArchiveStream | ✅ | ❌ | ✅ | TAR/7z 归档中的未压缩文件 |
| tTVPMemoryStream | ✅ | ✅ | ✅ | 解压缩缓冲区、内存中的临时数据 |
| tTVPPartialStream | ✅ | ❌ | ✅ | 大流中的一个片段（如 XP3 段） |
| tTVPXP3ArchiveStream | ✅ | ❌ | ✅ | XP3 归档（分段、可能压缩/加密） |
| 平台文件流 | ✅ | ✅ | ✅ | 直接操作磁盘文件 |
| SevenZipStreamWrap | ✅ | ❌ | ✅ | 将 tTJSBinaryStream 适配给 LZMA SDK C API |

### TArchiveStream vs tTVPPartialStream

这两个类功能看起来很相似——都是在一个大流上划定一个窗口。但它们有关键区别：

| 对比项 | TArchiveStream | tTVPPartialStream |
|--------|---------------|-------------------|
| 定义位置 | StorageIntf.h | UtilStreams.h |
| 所有权 | 持有 `tTVPArchive*` 引用 | 拥有底层 `tTJSBinaryStream*` |
| 析构行为 | 对归档 `Release()` | `delete` 底层流 |
| 底层流获取 | 从归档对象获取 | 构造时传入 |
| 使用场景 | 简单归档（TAR, 7z 未压缩）| XP3 分段读取 |

`TArchiveStream` 面向归档系统设计，与 `tTVPArchive` 紧耦合；而 `tTVPPartialStream` 是通用的部分流工具，可以包装任意 `tTJSBinaryStream`。

## 动手实践

### 实践 1：实现一个只读的字节数组流

以下是一个最小的 `tTJSBinaryStream` 子类，从一个固定的字节数组中读取数据：

```cpp
#include <cstring>   // memcpy
#include <cstdint>
#include <iostream>
#include <algorithm>  // std::min

// 模拟 KrKr2 类型定义（实际项目中来自 tjsTypes.h）
using tjs_uint64 = uint64_t;
using tjs_int64  = int64_t;
using tjs_uint   = unsigned int;
using tjs_int    = int;

// Seek 常量
constexpr int TJS_BS_SEEK_SET = 0;
constexpr int TJS_BS_SEEK_CUR = 1;
constexpr int TJS_BS_SEEK_END = 2;

// 简化的基类（实际项目中继承自 tTJSBinaryStream）
class SimpleByteArrayStream {
    const uint8_t *data_;   // 数据指针（不拥有所有权）
    size_t size_;           // 数据大小
    size_t pos_;            // 当前读取位置

public:
    // 构造函数：接收外部数据指针和大小
    SimpleByteArrayStream(const uint8_t *data, size_t size)
        : data_(data), size_(size), pos_(0) {}

    // Seek：移动读取位置
    tjs_uint64 Seek(tjs_int64 offset, tjs_int whence) {
        int64_t newpos = 0;
        switch (whence) {
            case TJS_BS_SEEK_SET: newpos = offset; break;
            case TJS_BS_SEEK_CUR: newpos = static_cast<int64_t>(pos_) + offset; break;
            case TJS_BS_SEEK_END: newpos = static_cast<int64_t>(size_) + offset; break;
        }
        // 边界钳制
        if (newpos < 0) newpos = 0;
        if (newpos > static_cast<int64_t>(size_)) newpos = static_cast<int64_t>(size_);
        pos_ = static_cast<size_t>(newpos);
        return pos_;
    }

    // Read：从当前位置读取数据
    tjs_uint Read(void *buffer, tjs_uint read_size) {
        size_t available = size_ - pos_;
        size_t actual = std::min(static_cast<size_t>(read_size), available);
        memcpy(buffer, data_ + pos_, actual);
        pos_ += actual;
        return static_cast<tjs_uint>(actual);
    }

    // Write：只读流，不支持写入
    tjs_uint Write(const void *, tjs_uint) { return 0; }

    // GetSize：返回数据总大小
    tjs_uint64 GetSize() { return size_; }

    // 辅助：获取当前位置
    tjs_uint64 GetPosition() { return Seek(0, TJS_BS_SEEK_CUR); }
};

int main() {
    // 准备测试数据
    uint8_t data[] = {
        0x48, 0x65, 0x6C, 0x6C, 0x6F,  // "Hello"
        0x2C, 0x20,                      // ", "
        0x57, 0x6F, 0x72, 0x6C, 0x64,   // "World"
        0x21                             // "!"
    };

    SimpleByteArrayStream stream(data, sizeof(data));

    // 测试 1：顺序读取
    char buf[6] = {};
    tjs_uint read = stream.Read(buf, 5);
    std::cout << "读取 " << read << " 字节: " << buf << std::endl;
    // 输出：读取 5 字节: Hello

    // 测试 2：获取当前位置
    std::cout << "当前位置: " << stream.GetPosition() << std::endl;
    // 输出：当前位置: 5

    // 测试 3：Seek 到末尾附近
    stream.Seek(-6, TJS_BS_SEEK_END);  // 从末尾往前 6 字节
    read = stream.Read(buf, 5);
    buf[read] = '\0';
    std::cout << "Seek 后读取: " << buf << std::endl;
    // 输出：Seek 后读取: World

    // 测试 4：流大小
    std::cout << "流大小: " << stream.GetSize() << " 字节" << std::endl;
    // 输出：流大小: 13 字节

    // 测试 5：超出范围读取
    stream.Seek(10, TJS_BS_SEEK_SET);
    read = stream.Read(buf, 100);  // 请求 100 字节，但只剩 3 字节
    buf[read] = '\0';
    std::cout << "越界读取: 请求 100, 实际 " << read
              << ", 内容: " << buf << std::endl;
    // 输出：越界读取: 请求 100, 实际 3, 内容: ld!

    return 0;
}
```

编译运行：

```bash
# Linux / macOS
g++ -std=c++17 -o byte_stream byte_stream.cpp && ./byte_stream

# Windows (MSVC)
cl /std:c++17 /EHsc byte_stream.cpp && byte_stream.exe
```

### 实践 2：实现 ReadBuffer 安全包装

```cpp
#include <cstring>
#include <cstdint>
#include <iostream>
#include <stdexcept>

using tjs_uint = unsigned int;

// 模拟 Read 方法（故意返回不完整的读取量来测试）
class MockStream {
    const uint8_t *data_;
    size_t size_, pos_;

public:
    MockStream(const uint8_t *data, size_t size)
        : data_(data), size_(size), pos_(0) {}

    tjs_uint Read(void *buffer, tjs_uint read_size) {
        size_t avail = size_ - pos_;
        size_t actual = (read_size < avail) ? read_size : avail;
        memcpy(buffer, data_ + pos_, actual);
        pos_ += actual;
        return static_cast<tjs_uint>(actual);
    }

    // ReadBuffer：带完整性检查的读取
    void ReadBuffer(void *buffer, tjs_uint read_size) {
        tjs_uint actual = Read(buffer, read_size);
        if (actual != read_size) {
            throw std::runtime_error(
                "ReadBuffer: 期望读取 " + std::to_string(read_size) +
                " 字节，实际只读取了 " + std::to_string(actual) + " 字节");
        }
    }

    size_t GetPosition() const { return pos_; }
};

int main() {
    uint8_t data[] = { 0x01, 0x02, 0x03, 0x04, 0x05 };
    MockStream stream(data, sizeof(data));

    // 正常情况：ReadBuffer 成功
    uint8_t buf[3];
    stream.ReadBuffer(buf, 3);
    std::cout << "ReadBuffer 成功，读取 3 字节: ";
    for (int i = 0; i < 3; i++) {
        std::cout << "0x" << std::hex << (int)buf[i] << " ";
    }
    std::cout << std::dec << std::endl;
    // 输出：ReadBuffer 成功，读取 3 字节: 0x1 0x2 0x3

    // 错误情况：剩余数据不够
    try {
        uint8_t big_buf[10];
        stream.ReadBuffer(big_buf, 10);  // 只剩 2 字节，但请求 10 字节
    } catch (const std::runtime_error &e) {
        std::cout << "捕获异常: " << e.what() << std::endl;
        // 输出：捕获异常: ReadBuffer: 期望读取 10 字节，实际只读取了 2 字节
    }

    return 0;
}
```

### 实践 3：小端整数读取器

```cpp
#include <cstdint>
#include <cstring>
#include <iostream>
#include <iomanip>

// 通用的小端整数读取函数
// 与 tTJSBinaryStream::ReadI32LE() 等价
class LittleEndianReader {
    const uint8_t *data_;
    size_t size_, pos_;

public:
    LittleEndianReader(const uint8_t *data, size_t size)
        : data_(data), size_(size), pos_(0) {}

    void ReadBuffer(void *buf, size_t n) {
        if (pos_ + n > size_)
            throw std::runtime_error("读取越界");
        memcpy(buf, data_ + pos_, n);
        pos_ += n;
    }

    // 读取 8 位无符号整数
    uint8_t ReadI8LE() {
        uint8_t val;
        ReadBuffer(&val, 1);
        return val;
    }

    // 读取 16 位小端无符号整数
    uint16_t ReadI16LE() {
        uint8_t buf[2];
        ReadBuffer(buf, 2);
        return static_cast<uint16_t>(buf[0])
             | (static_cast<uint16_t>(buf[1]) << 8);
    }

    // 读取 32 位小端无符号整数
    uint32_t ReadI32LE() {
        uint8_t buf[4];
        ReadBuffer(buf, 4);
        return static_cast<uint32_t>(buf[0])
             | (static_cast<uint32_t>(buf[1]) << 8)
             | (static_cast<uint32_t>(buf[2]) << 16)
             | (static_cast<uint32_t>(buf[3]) << 24);
    }

    // 读取 64 位小端无符号整数
    uint64_t ReadI64LE() {
        uint8_t buf[8];
        ReadBuffer(buf, 8);
        uint64_t val = 0;
        for (int i = 0; i < 8; i++) {
            val |= static_cast<uint64_t>(buf[i]) << (i * 8);
        }
        return val;
    }
};

int main() {
    // 模拟一个 XP3 文件头的部分数据
    uint8_t header[] = {
        // 16 位值 0x0201 (little-endian: 01 02)
        0x01, 0x02,
        // 32 位值 0x78563412 (little-endian: 12 34 56 78)
        0x12, 0x34, 0x56, 0x78,
        // 64 位值 0x00000000DEADBEEF
        0xEF, 0xBE, 0xAD, 0xDE, 0x00, 0x00, 0x00, 0x00
    };

    LittleEndianReader reader(header, sizeof(header));

    uint16_t val16 = reader.ReadI16LE();
    std::cout << "16位值: 0x" << std::hex << std::setfill('0')
              << std::setw(4) << val16 << std::endl;
    // 输出：16位值: 0x0201

    uint32_t val32 = reader.ReadI32LE();
    std::cout << "32位值: 0x" << std::setw(8) << val32 << std::endl;
    // 输出：32位值: 0x78563412

    uint64_t val64 = reader.ReadI64LE();
    std::cout << "64位值: 0x" << std::setw(16) << val64 << std::endl;
    // 输出：64位值: 0x00000000deadbeef

    return 0;
}
```

### 实践 4：流窗口（模拟 TArchiveStream）

```cpp
#include <cstdint>
#include <cstring>
#include <iostream>
#include <algorithm>
#include <vector>

constexpr int BS_SEEK_SET = 0;
constexpr int BS_SEEK_CUR = 1;
constexpr int BS_SEEK_END = 2;

// 模拟底层完整流
class FullStream {
    std::vector<uint8_t> data_;
    size_t pos_ = 0;

public:
    FullStream(const std::vector<uint8_t> &data) : data_(data) {}

    void SetPosition(size_t pos) { pos_ = pos; }
    size_t GetPosition() const { return pos_; }

    void ReadBuffer(void *buf, size_t n) {
        memcpy(buf, data_.data() + pos_, n);
        pos_ += n;
    }

    size_t GetSize() const { return data_.size(); }
};

// 模拟 TArchiveStream 的窗口流
class WindowStream {
    FullStream *inner_;       // 底层流（不拥有所有权）
    int64_t currentPos_;      // 窗口内的当前位置
    uint64_t startPos_;       // 窗口在底层流中的起始位置
    uint64_t dataLength_;     // 窗口大小

public:
    WindowStream(FullStream *inner, uint64_t start, uint64_t length)
        : inner_(inner), currentPos_(0), startPos_(start), dataLength_(length)
    {
        inner_->SetPosition(start);
    }

    uint64_t Seek(int64_t offset, int whence) {
        switch (whence) {
            case BS_SEEK_SET: currentPos_ = offset; break;
            case BS_SEEK_CUR: currentPos_ += offset; break;
            case BS_SEEK_END: currentPos_ = static_cast<int64_t>(dataLength_) + offset; break;
        }
        // 边界钳制
        if (currentPos_ < 0) currentPos_ = 0;
        if (currentPos_ > static_cast<int64_t>(dataLength_))
            currentPos_ = static_cast<int64_t>(dataLength_);
        // 转换为底层流的绝对位置
        inner_->SetPosition(static_cast<size_t>(currentPos_ + startPos_));
        return static_cast<uint64_t>(currentPos_);
    }

    unsigned int Read(void *buffer, unsigned int readSize) {
        int64_t avail = static_cast<int64_t>(dataLength_) - currentPos_;
        if (avail <= 0) return 0;
        unsigned int actual = std::min(static_cast<int64_t>(readSize), avail);
        inner_->ReadBuffer(buffer, actual);
        currentPos_ += actual;
        return actual;
    }

    uint64_t GetSize() const { return dataLength_; }
};

int main() {
    // 模拟一个归档文件，包含三段数据
    //   [0-4]:   "HEAD_"     (归档头)
    //   [5-16]:  "Hello World" (文件 A，11 字节)
    //   [17-26]: "KiriKiri!!"  (文件 B，10 字节)
    std::vector<uint8_t> archive = {
        'H','E','A','D','_',                         // 0-4: 头部
        'H','e','l','l','o',' ','W','o','r','l','d',  // 5-15: 文件 A
        'K','i','r','i','K','i','r','i','!','!'       // 16-25: 文件 B
    };

    FullStream fullStream(archive);

    // 创建文件 A 的窗口流：从偏移 5 开始，长度 11
    WindowStream fileA(&fullStream, 5, 11);
    std::cout << "文件 A 大小: " << fileA.GetSize() << " 字节" << std::endl;

    char buf[32] = {};
    unsigned int n = fileA.Read(buf, 20);  // 请求 20 但只有 11
    buf[n] = '\0';
    std::cout << "文件 A 内容: \"" << buf << "\" (实际读取 " << n << " 字节)" << std::endl;
    // 输出：文件 A 内容: "Hello World" (实际读取 11 字节)

    // 创建文件 B 的窗口流：从偏移 16 开始，长度 10
    WindowStream fileB(&fullStream, 16, 10);
    n = fileB.Read(buf, 20);
    buf[n] = '\0';
    std::cout << "文件 B 内容: \"" << buf << "\" (实际读取 " << n << " 字节)" << std::endl;
    // 输出：文件 B 内容: "KiriKiri!!" (实际读取 10 字节)

    // 测试 Seek
    fileA.Seek(6, BS_SEEK_SET);  // 跳到文件 A 的第 6 字节
    n = fileA.Read(buf, 5);
    buf[n] = '\0';
    std::cout << "文件 A 偏移 6 处: \"" << buf << "\"" << std::endl;
    // 输出：文件 A 偏移 6 处: "World"

    return 0;
}
```

### 实践 5：带缓冲的流包装器

在实际项目中，频繁调用底层 `Read` 会导致性能问题（尤其是磁盘文件）。下面实现一个带读取缓冲（Read Buffer）的流包装器，减少底层调用次数：

```cpp
#include <cstdint>
#include <cstring>
#include <iostream>
#include <algorithm>
#include <vector>

// 简化的流接口
class IStream {
public:
    virtual ~IStream() = default;
    virtual unsigned int Read(void *buf, unsigned int size) = 0;
    virtual uint64_t Seek(int64_t offset, int whence) = 0;
    virtual uint64_t GetSize() = 0;
};

// 模拟的底层流（记录 Read 调用次数）
class RawStream : public IStream {
    std::vector<uint8_t> data_;
    size_t pos_ = 0;
    int readCallCount_ = 0;   // 调试用：记录底层 Read 调用次数

public:
    RawStream(size_t size) : data_(size) {
        // 用递增字节填充
        for (size_t i = 0; i < size; i++)
            data_[i] = static_cast<uint8_t>(i & 0xFF);
    }

    unsigned int Read(void *buf, unsigned int size) override {
        readCallCount_++;  // 记录调用次数
        size_t avail = data_.size() - pos_;
        unsigned int actual = static_cast<unsigned int>(
            std::min(static_cast<size_t>(size), avail));
        memcpy(buf, data_.data() + pos_, actual);
        pos_ += actual;
        return actual;
    }

    uint64_t Seek(int64_t offset, int whence) override {
        switch (whence) {
            case 0: pos_ = static_cast<size_t>(offset); break;
            case 1: pos_ += offset; break;
            case 2: pos_ = data_.size() + offset; break;
        }
        return pos_;
    }

    uint64_t GetSize() override { return data_.size(); }
    int GetReadCallCount() const { return readCallCount_; }
};

// 带缓冲的流包装器
class BufferedStream : public IStream {
    IStream *inner_;           // 底层流（不拥有所有权）
    std::vector<uint8_t> buf_; // 内部缓冲区
    size_t bufStart_ = 0;     // 缓冲区对应的流中起始位置
    size_t bufLen_ = 0;       // 缓冲区中有效数据长度
    size_t pos_ = 0;          // 当前逻辑读取位置

    static constexpr size_t BUF_SIZE = 4096;  // 缓冲区大小

    // 重新填充缓冲区
    void refill() {
        inner_->Seek(static_cast<int64_t>(pos_), 0);
        unsigned int n = inner_->Read(buf_.data(), BUF_SIZE);
        bufStart_ = pos_;
        bufLen_ = n;
    }

public:
    BufferedStream(IStream *inner)
        : inner_(inner), buf_(BUF_SIZE) {}

    unsigned int Read(void *buffer, unsigned int size) override {
        auto *dst = static_cast<uint8_t *>(buffer);
        unsigned int totalRead = 0;

        while (size > 0) {
            // 检查当前位置是否在缓冲区范围内
            if (pos_ < bufStart_ || pos_ >= bufStart_ + bufLen_) {
                refill();  // 缓冲区未命中，需要重新填充
                if (bufLen_ == 0) break;  // 底层流没有更多数据
            }

            // 从缓冲区中复制数据
            size_t bufOffset = pos_ - bufStart_;
            size_t avail = bufLen_ - bufOffset;
            size_t toCopy = std::min(static_cast<size_t>(size), avail);
            memcpy(dst, buf_.data() + bufOffset, toCopy);

            dst += toCopy;
            pos_ += toCopy;
            size -= static_cast<unsigned int>(toCopy);
            totalRead += static_cast<unsigned int>(toCopy);
        }

        return totalRead;
    }

    uint64_t Seek(int64_t offset, int whence) override {
        switch (whence) {
            case 0: pos_ = static_cast<size_t>(offset); break;
            case 1: pos_ += offset; break;
            case 2: pos_ = static_cast<size_t>(inner_->GetSize()) + offset; break;
        }
        return pos_;
    }

    uint64_t GetSize() override { return inner_->GetSize(); }
};

int main() {
    // 创建一个 10000 字节的底层流
    RawStream raw(10000);

    // 不使用缓冲：每次读 1 字节，读 1000 次
    uint8_t byte;
    for (int i = 0; i < 1000; i++) {
        raw.Read(&byte, 1);
    }
    std::cout << "无缓冲：1000 次 1 字节读取 → 底层调用 "
              << raw.GetReadCallCount() << " 次" << std::endl;
    // 输出：无缓冲：1000 次 1 字节读取 → 底层调用 1000 次

    // 使用缓冲：同样的操作
    RawStream raw2(10000);
    BufferedStream buffered(&raw2);
    for (int i = 0; i < 1000; i++) {
        buffered.Read(&byte, 1);
    }
    std::cout << "有缓冲：1000 次 1 字节读取 → 底层调用 "
              << raw2.GetReadCallCount() << " 次" << std::endl;
    // 输出：有缓冲：1000 次 1 字节读取 → 底层调用 1 次
    // （因为 4096 字节缓冲区一次就能覆盖 1000 字节的读取）

    return 0;
}
```

## 对照项目源码

### tTJSBinaryStream 基类

相关文件：
- `cpp/core/tjs2/tjs.h` 第 243-299 行 — 基类定义、模式标志、Seek 常量

基类定义在 TJS2 脚本引擎模块中而非 base 模块，这是因为 `tTJSBinaryStream` 是 TJS2 与外部世界交互的核心接口之一。TJS2 的 `Array.save()`、`Dictionary.save()` 等方法需要通过这个接口读写文件。

### TArchiveStream

相关文件：
- `cpp/core/base/StorageIntf.h` 第 331-381 行 — 完整定义（头文件中内联）

注意 `TArchiveStream` 的 `Seek` 和 `Read` 方法都是内联实现的（直接写在头文件的类定义中）。这种做法在 KrKr2 代码中偶尔出现，通常是因为这些方法很短小，内联可以避免函数调用开销。但 AGENTS.md 中标注了"maintain but don't extend this pattern"——也就是说，新代码不应该再这样做。

### tTVPStreamHolder

相关文件：
- `cpp/core/base/UtilStreams.h` 第 20-49 行 — RAII 包装器定义

`tTVPStreamHolder` 在引擎各处广泛使用，任何需要打开流并在作用域退出时自动关闭的场景都会用到它。例如在 `tTVPLocalTempStorageHolder` 中复制文件到本地时就使用了两个 `tTVPStreamHolder`：

```cpp
// 源码：cpp/core/base/UtilStreams.cpp 第 47-64 行
tTVPStreamHolder src(name);                           // 源流
tTVPStreamHolder dest(LocalName, TJS_BS_WRITE | TJS_BS_DELETE_ON_CLOSE); // 目标流
tjs_uint8 *buffer = new tjs_uint8[TVP_LOCAL_TEMP_COPY_BLOCK_SIZE];
try {
    tjs_uint read;
    while(true) {
        read = src->Read(buffer, TVP_LOCAL_TEMP_COPY_BLOCK_SIZE);
        if(read == 0) break;
        dest->WriteBuffer(buffer, read);
    }
} catch(...) {
    delete[] buffer;
    throw;
}
delete[] buffer;
```

### BinaryStream 工厂函数

相关文件：
- `cpp/core/base/BinaryStream.h` 第 7-10, 22-38 行 — 工厂声明 + `parseModeNumber`
- `cpp/core/base/BinaryStream.cpp` 第 27-77 行 — 工厂实现

`TVPCreateBinaryStreamForRead`/`Write` 是 TJS2 脚本层（通过函数指针 `TJSCreateBinaryStreamForRead`/`Write`）调用的入口，负责创建流并处理偏移量模式。

## 常见错误及解决方案

### 错误 1：忘记检查 Read 返回值

```cpp
// ❌ 错误：假设总是读到请求的字节数
char header[16];
stream->Read(header, 16);  // 可能只读到 8 字节！
ParseHeader(header);       // 使用不完整的数据

// ✅ 正确做法 1：用 ReadBuffer（会在不足时抛异常）
stream->ReadBuffer(header, 16);

// ✅ 正确做法 2：手动检查
tjs_uint n = stream->Read(header, 16);
if (n != 16) {
    // 处理数据不足的情况
}
```

### 错误 2：Seek 后不检查返回值

```cpp
// ❌ 错误：假设 Seek 一定成功
stream->Seek(offset, TJS_BS_SEEK_SET);
// 如果 offset 超过流大小，位置不变，但代码继续执行

// ✅ 正确：检查 Seek 返回的实际位置
tjs_uint64 actual = stream->Seek(offset, TJS_BS_SEEK_SET);
if (actual != offset) {
    // Seek 失败，offset 超出范围
}
```

### 错误 3：对只读流调用 Write

```cpp
// ❌ 错误：TArchiveStream 的 Write 返回 0 但不报错
TArchiveStream *stream = archive->CreateStreamByIndex(0);
stream->Write(data, size);  // 静默返回 0，数据没有写入！

// ✅ 检查写入是否成功
tjs_uint written = stream->Write(data, size);
if (written == 0 && size > 0) {
    // 流可能是只读的
}

// ✅ 或者使用 WriteBuffer（在写入不完整时抛异常）
stream->WriteBuffer(data, size);  // 只读流会抛异常
```

## 本节小结

- `tTJSBinaryStream` 是 KrKr2 引擎所有 IO 操作的核心抽象基类
- 4 个纯虚函数 `Seek`/`Read`/`Write`/`GetSize` 构成最小接口，所有子类必须实现
- 辅助方法 `ReadBuffer`/`WriteBuffer` 提供带完整性检查的读写，比裸 `Read`/`Write` 更安全
- `ReadI64LE`/`I32LE`/`I16LE`/`I8LE` 系列方法简化了小端整数的读取，是解析二进制格式的高频工具
- `TArchiveStream` 通过偏移量+长度实现流窗口，对外表现为独立流
- `tTVPStreamHolder` 提供 RAII 包装，确保流的生命周期安全
- 流模式标志 `TJS_BS_READ`/`WRITE`/`APPEND`/`UPDATE` 控制打开方式

## 练习题与答案

### 题目 1：Seek 行为分析

给定以下代码片段，分析每一步的位置变化和最终输出：

```cpp
tTVPMemoryStream stream;
// 假设流中有 100 字节数据
stream.SetSize(100);

uint64_t pos;
pos = stream.Seek(50, TJS_BS_SEEK_SET);   // (1)
pos = stream.Seek(20, TJS_BS_SEEK_CUR);   // (2)
pos = stream.Seek(-10, TJS_BS_SEEK_END);  // (3)
pos = stream.Seek(-200, TJS_BS_SEEK_SET); // (4)
pos = stream.Seek(200, TJS_BS_SEEK_SET);  // (5)
```

<details>
<summary>查看答案</summary>

```
(1) Seek(50, SEEK_SET)：
    从开头偏移 50 → 新位置 = 50
    offset >= 0 且 50 <= 100 → 合法，CurrentPos = 50
    返回 50

(2) Seek(20, SEEK_CUR)：
    从当前位置(50)偏移 +20 → newpos = 70
    newpos >= 0 且 70 <= 100 → 合法，CurrentPos = 70
    返回 70

(3) Seek(-10, SEEK_END)：
    从末尾(100)偏移 -10 → newpos = 90
    newpos >= 0 且 90 <= 100 → 合法，CurrentPos = 90
    返回 90

(4) Seek(-200, SEEK_SET)：
    从开头偏移 -200
    offset < 0 → 不合法！什么都不做
    CurrentPos 保持为 90
    返回 90

(5) Seek(200, SEEK_SET)：
    从开头偏移 200
    offset >= 0 但 200 > 100（Size）→ 不合法！什么都不做
    CurrentPos 保持为 90
    返回 90
```

关键点：`tTVPMemoryStream::Seek` 在位置不合法时**静默忽略**，不抛异常，不改变位置。这与 C 标准库 `fseek()` 的行为不同——`fseek()` 允许 seek 到文件末尾之后（会创建空洞），但 `tTVPMemoryStream::Seek` 不允许。

</details>

### 题目 2：实现一个反向流

实现一个 `ReverseStream` 类，它包装另一个流，使得从头部读取时实际上是从底层流的末尾倒着读取。即：`Read(buf, 5)` 读取的是底层流最后 5 字节（倒序排列）。

<details>
<summary>查看答案</summary>

```cpp
#include <cstdint>
#include <cstring>
#include <vector>
#include <iostream>
#include <algorithm>

class IStream {
public:
    virtual ~IStream() = default;
    virtual unsigned int Read(void *buf, unsigned int size) = 0;
    virtual uint64_t Seek(int64_t offset, int whence) = 0;
    virtual uint64_t GetSize() = 0;
};

class ArrayStream : public IStream {
    std::vector<uint8_t> data_;
    size_t pos_ = 0;
public:
    ArrayStream(std::initializer_list<uint8_t> init) : data_(init) {}
    unsigned int Read(void *buf, unsigned int size) override {
        size_t avail = data_.size() - pos_;
        unsigned int n = static_cast<unsigned int>(std::min(static_cast<size_t>(size), avail));
        memcpy(buf, data_.data() + pos_, n);
        pos_ += n;
        return n;
    }
    uint64_t Seek(int64_t offset, int whence) override {
        switch (whence) {
            case 0: pos_ = static_cast<size_t>(offset); break;
            case 1: pos_ += offset; break;
            case 2: pos_ = data_.size() + offset; break;
        }
        return pos_;
    }
    uint64_t GetSize() override { return data_.size(); }
};

// 反向流：逻辑位置 0 对应底层流末尾，逻辑位置 N 对应底层流 Size-N
class ReverseStream : public IStream {
    IStream *inner_;
    size_t pos_ = 0;  // 逻辑位置

public:
    ReverseStream(IStream *inner) : inner_(inner) {}

    unsigned int Read(void *buf, unsigned int size) override {
        uint64_t totalSize = inner_->GetSize();
        size_t avail = static_cast<size_t>(totalSize) - pos_;
        unsigned int n = static_cast<unsigned int>(std::min(static_cast<size_t>(size), avail));

        auto *dst = static_cast<uint8_t *>(buf);
        // 从底层流倒着读取每个字节
        for (unsigned int i = 0; i < n; i++) {
            // 逻辑位置 pos_+i 对应底层流位置 totalSize - 1 - (pos_+i)
            size_t realPos = static_cast<size_t>(totalSize) - 1 - (pos_ + i);
            inner_->Seek(static_cast<int64_t>(realPos), 0);
            inner_->Read(dst + i, 1);
        }
        pos_ += n;
        return n;
    }

    uint64_t Seek(int64_t offset, int whence) override {
        uint64_t totalSize = inner_->GetSize();
        switch (whence) {
            case 0: pos_ = static_cast<size_t>(offset); break;
            case 1: pos_ += offset; break;
            case 2: pos_ = static_cast<size_t>(totalSize) + offset; break;
        }
        return pos_;
    }

    uint64_t GetSize() override { return inner_->GetSize(); }
};

int main() {
    ArrayStream inner({0x01, 0x02, 0x03, 0x04, 0x05});
    ReverseStream rev(&inner);

    uint8_t buf[5];
    unsigned int n = rev.Read(buf, 5);
    std::cout << "反向读取 " << n << " 字节: ";
    for (unsigned int i = 0; i < n; i++) {
        std::cout << "0x" << std::hex << (int)buf[i] << " ";
    }
    std::cout << std::endl;
    // 输出：反向读取 5 字节: 0x5 0x4 0x3 0x2 0x1

    return 0;
}
```

注意：这个实现效率很低（每个字节都 Seek + Read 一次），实际项目中应该一次性读取一块然后反转。但它清晰地展示了流包装的概念。

</details>

### 题目 3：流模式标志解析

给定模式标志值 `0x13`，用位运算提取出访问模式和选项部分，并说明其含义。

<details>
<summary>查看答案</summary>

```cpp
#include <cstdint>
#include <iostream>

#define TJS_BS_READ          0
#define TJS_BS_WRITE         1
#define TJS_BS_APPEND        2
#define TJS_BS_UPDATE        3
#define TJS_BS_DELETE_ON_CLOSE  0x10
#define TJS_BS_ACCESS_MASK   0x0f
#define TJS_BS_OPTION_MASK   0xf0

int main() {
    uint32_t flags = 0x13;

    // 提取访问模式
    uint32_t access = flags & TJS_BS_ACCESS_MASK;
    // 0x13 & 0x0F = 0x03 = TJS_BS_UPDATE

    // 提取选项部分
    uint32_t options = flags & TJS_BS_OPTION_MASK;
    // 0x13 & 0xF0 = 0x10 = TJS_BS_DELETE_ON_CLOSE

    std::cout << "标志值: 0x" << std::hex << flags << std::endl;
    std::cout << "访问模式: 0x" << access;
    switch (access) {
        case TJS_BS_READ:   std::cout << " (只读)"; break;
        case TJS_BS_WRITE:  std::cout << " (只写)"; break;
        case TJS_BS_APPEND: std::cout << " (追加)"; break;
        case TJS_BS_UPDATE: std::cout << " (读写更新)"; break;
    }
    std::cout << std::endl;

    std::cout << "选项: 0x" << options;
    if (options & TJS_BS_DELETE_ON_CLOSE)
        std::cout << " (关闭时删除)";
    std::cout << std::endl;

    // 总结：0x13 = TJS_BS_UPDATE | TJS_BS_DELETE_ON_CLOSE
    // 含义：以读写更新模式打开，关闭时自动删除文件
    std::cout << std::endl << "含义：以读写更新模式打开，关闭时自动删除文件" << std::endl;

    return 0;
}
```

位运算过程：

```
0x13 的二进制:  0001 0011
                ││││ ││││
TJS_BS_ACCESS_MASK (0x0F):  0000 1111
AND 结果:                    0000 0011 = 0x03 = TJS_BS_UPDATE

TJS_BS_OPTION_MASK (0xF0):  1111 0000
AND 结果:                    0001 0000 = 0x10 = TJS_BS_DELETE_ON_CLOSE
```

所以 `0x13 = TJS_BS_UPDATE | TJS_BS_DELETE_ON_CLOSE`，表示以读写更新模式打开文件，且文件会在流关闭时被自动删除。

</details>

## 下一步

下一节 [内存流与部分流](./02-内存流与部分流.md) 将深入分析 `tTVPMemoryStream` 的可增长内存缓冲区实现（包括引用模式、分级扩容策略、自定义分配器）和 `tTVPPartialStream` 的只读窗口机制。

