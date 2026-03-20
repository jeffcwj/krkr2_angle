# ZIP 归档实现

> **所属模块：** M08-归档与IO系统
> **前置知识：** [tTVPArchive 基类设计](../01-归档系统概述/02-tTVPArchive基类设计.md)、[归档格式对比与选择](../01-归档系统概述/03-归档格式对比与选择.md)
> **预计阅读时间：** 45 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：
1. 理解 ZIP 文件格式的二进制结构（本地文件头、中央目录、结束记录）
2. 掌握 KrKr2 如何重新实现 minizip 库以适配自有流系统
3. 分析 I/O 回调桥接机制——将 `tTJSBinaryStream` 接入 minizip 的 `zlib_filefunc` 接口
4. 理解 ZIP64 扩展对大文件（>4GB）的支持原理
5. 掌握 `ZipArchive` 类如何实现 `tTVPArchive` 三个纯虚函数

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 本地文件头 | Local File Header | ZIP 中每个文件数据前的元信息头，包含压缩方法、CRC、大小等 |
| 中央目录 | Central Directory | ZIP 文件末尾的"总目录"，汇总所有文件的元信息和偏移量 |
| 结束记录 | End of Central Directory Record (EOCD) | ZIP 文件最末尾的定位结构，告诉解析器中央目录在哪里 |
| ZIP64 | ZIP64 Extended Information | 突破 4GB/65535 文件数限制的扩展，使用 64 位字段 |
| Deflate | Deflate Compression | ZIP 最常用的压缩算法，基于 LZ77 + Huffman 编码 |
| minizip | minizip Library | zlib 附带的轻量 ZIP 解压库，KrKr2 对其进行了深度定制 |
| I/O 回调 | I/O Callback Functions | 让 minizip 不依赖 `FILE*`，而是通过函数指针操作自定义流 |
| ZPOS64_T | 64-bit Position Type | minizip 中表示文件内 64 位偏移量的类型，实际是 `uint64_t` |

---

## 一、ZIP 文件格式基础

### 1.1 为什么 KrKr2 需要支持 ZIP

KrKr2 的原生归档格式是 XP3（在前一章已详细讲解），但实际游戏开发和分发中，ZIP 格式的使用也非常普遍：

- **第三方资源分发**：许多游戏资源以 ZIP 格式打包交付
- **调试便利**：开发阶段用 ZIP 打包资源比 XP3 更方便（任何压缩工具都能创建）
- **兼容性**：部分旧版 KiriKiri 游戏使用 ZIP 而非 XP3 存储资源
- **Android 平台**：APK 本身就是 ZIP 格式，可能需要直接从 APK 内读取资源

### 1.2 ZIP 文件整体结构

ZIP 文件的结构是"先数据、后目录"，与 XP3 类似。整体布局如下：

```
┌──────────────────────────────────────────────┐
│              ZIP 文件整体结构                  │
├──────────────────────────────────────────────┤
│                                              │
│  ┌─────────────────────────────────────┐     │
│  │ [本地文件头 1] + [文件数据 1]         │     │
│  ├─────────────────────────────────────┤     │
│  │ [本地文件头 2] + [文件数据 2]         │     │
│  ├─────────────────────────────────────┤     │
│  │ ...                                 │     │
│  ├─────────────────────────────────────┤     │
│  │ [本地文件头 N] + [文件数据 N]         │     │
│  └─────────────────────────────────────┘     │
│                                              │
│  ┌─────────────────────────────────────┐     │
│  │ [中央目录条目 1]                     │     │
│  │ [中央目录条目 2]                     │     │
│  │ ...                                 │     │
│  │ [中央目录条目 N]                     │     │
│  └─────────────────────────────────────┘     │
│                                              │
│  ┌─────────────────────────────────────┐     │
│  │ [ZIP64 结束记录] (可选)              │     │
│  │ [ZIP64 结束定位器] (可选)            │     │
│  │ [结束记录 EOCD]                     │     │
│  └─────────────────────────────────────┘     │
│                                              │
└──────────────────────────────────────────────┘
```

### 1.3 本地文件头（Local File Header）

每个文件数据前都有一个本地文件头，签名为 `0x04034B50`（即 ASCII `PK\x03\x04`）：

```
偏移  大小  说明
───────────────────────────────────────
 0    4    签名 = 0x04034B50 ('PK' + 03 04)
 4    2    解压所需版本
 6    2    通用位标志（bit flags）
 8    2    压缩方法（0=存储, 8=Deflate）
10    2    最后修改时间（DOS 格式）
12    2    最后修改日期（DOS 格式）
14    4    CRC-32 校验值
18    4    压缩后大小
22    4    原始大小
26    2    文件名长度 (n)
28    2    扩展字段长度 (m)
30    n    文件名（不以 \0 结尾）
30+n  m    扩展字段
30+n+m ... 文件数据（压缩或原始）
```

对应 KrKr2 源码中的常量定义：

```cpp
// ZIPArchive.cpp 第 108 行
#define SIZEZIPLOCALHEADER (0x1e)  // 30 字节 = 本地文件头固定部分大小
```

### 1.4 中央目录条目（Central Directory Entry）

中央目录位于所有文件数据之后，每个条目的签名为 `0x02014B50`：

```
偏移  大小  说明
───────────────────────────────────────
 0    4    签名 = 0x02014B50 ('PK' + 01 02)
 4    2    创建版本
 6    2    解压所需版本
 8    2    通用位标志
10    2    压缩方法
12    4    DOS 日期时间
16    4    CRC-32
20    4    压缩后大小
24    4    原始大小
28    2    文件名长度 (n)
30    2    扩展字段长度 (m)
32    2    注释长度 (k)
34    2    起始磁盘编号
36    2    内部文件属性
38    4    外部文件属性
42    4    本地文件头的相对偏移
46    n    文件名
46+n  m    扩展字段
46+n+m k   注释
```

对应 KrKr2 中的常量：

```cpp
// ZIPArchive.cpp 第 107 行
#define SIZECENTRALDIRITEM (0x2e)  // 46 字节 = 中央目录条目固定部分大小
```

KrKr2 将中央目录条目的信息解析到 `unz_file_info64` 结构体中：

```cpp
// ZIPArchive.cpp 第 159-183 行（简化）
typedef struct unz_file_info64_s {
    uLong version;              // 创建版本
    uLong version_needed;       // 解压所需版本
    uLong flag;                 // 通用位标志
    uLong compression_method;   // 压缩方法（0=存储, 8=Deflate）
    uLong dosDate;              // DOS 日期时间
    uLong crc;                  // CRC-32 校验值
    ZPOS64_T compressed_size;   // 压缩后大小（ZIP64 下为 64 位）
    ZPOS64_T uncompressed_size; // 原始大小（ZIP64 下为 64 位）
    uLong size_filename;        // 文件名长度
    uLong size_file_extra;      // 扩展字段长度
    uLong size_file_comment;    // 注释长度
    ZPOS64_T offset_curfile;    // 本地文件头偏移（ZIP64 扩展后）
    uLong disk_num_start;       // 起始磁盘编号
    uLong internal_fa;          // 内部文件属性
    uLong external_fa;          // 外部文件属性
    tm_unz tmu_date;            // 解析后的日期时间
} unz_file_info64;
```

### 1.5 结束记录（End of Central Directory Record）

结束记录是 ZIP 解析的入口点，签名为 `0x06054B50`：

```
偏移  大小  说明
───────────────────────────────────────
 0    4    签名 = 0x06054B50
 4    2    当前磁盘编号
 6    2    中央目录起始磁盘编号
 8    2    本磁盘中央目录条目数
10    2    中央目录总条目数
12    4    中央目录大小（字节）
16    4    中央目录起始偏移
20    2    注释长度 (n)
22    n    注释内容
```

> **关键理解**：ZIP 解析总是从文件末尾向前搜索 EOCD 签名开始，因为只有找到 EOCD 才能知道中央目录的位置。这与 XP3 从头部读取索引偏移的方式不同。

---

## 二、KrKr2 的 minizip 重实现

### 2.1 为什么不直接用标准 minizip

KrKr2 没有直接链接标准 minizip 库，而是将 minizip 的核心代码直接内嵌到 `ZIPArchive.cpp` 中（整个文件 2138 行）。原因有三：

1. **I/O 适配**：标准 minizip 基于 `FILE*` 或 `zlib_filefunc_def` 的文件 I/O，KrKr2 需要用自己的 `tTJSBinaryStream` 替代
2. **精简裁剪**：KrKr2 只需要解压功能（unzip），不需要创建 ZIP 的功能
3. **类型定制**：部分类型定义（如 `tm_unz`、`unz_file_info`）需要重新定义以避免与其他库冲突

```cpp
// ZIPArchive.cpp 开头的类型重定义（第 16-60 行）
#undef ZEXPORT
#define ZEXPORT
typedef uint64_t ZPOS64_T;  // 强制使用 64 位位置类型

// 重新定义时间结构，避免与系统 tm 冲突
typedef struct tm_unz_s {
    uInt tm_sec;   // 秒 [0,59]
    uInt tm_min;   // 分 [0,59]
    uInt tm_hour;  // 时 [0,23]
    uInt tm_mday;  // 日 [1,31]
    uInt tm_mon;   // 月 [0,11]
    uInt tm_year;  // 年 [1980..2044]
} tm_unz;
```

### 2.2 字节序读取工具函数

ZIP 格式使用小端序（Little-Endian）存储所有多字节字段。KrKr2 实现了一组逐字节读取的工具函数，保证在任何平台上都能正确解析：

```cpp
// 读取单个字节（ZIPArchive.cpp 第 263-276 行）
local int unz64local_getByte(
    const zlib_filefunc64_32_def *pzlib_filefunc_def,
    voidpf filestream, int *pi) 
{
    unsigned char c;
    // 通过 I/O 回调读取 1 字节
    int err = (int)zip_readfile(nullptr, filestream, &c, 1);
    if(err == 1) {
        *pi = (int)c;       // 成功：返回字节值
        return UNZ_OK;
    } else {
        return UNZ_EOF;     // 失败：文件结束
    }
}

// 读取 16 位小端整数（第 285-303 行）
local int unz64local_getShort(
    const zlib_filefunc64_32_def *pzlib_filefunc_def,
    voidpf filestream, uLong *pX) 
{
    uLong x;
    int i = 0;
    int err;
    err = unz64local_getByte(pzlib_filefunc_def, filestream, &i);
    x = (uLong)i;                      // 低字节
    if(err == UNZ_OK)
        err = unz64local_getByte(pzlib_filefunc_def, filestream, &i);
    x |= ((uLong)i) << 8;             // 高字节左移 8 位
    if(err == UNZ_OK) *pX = x;
    else *pX = 0;
    return err;
}

// 读取 32 位小端整数（第 309-335 行）
local int unz64local_getLong(...) {
    // 类似，依次读取 4 个字节，按 0/8/16/24 位移位合并
    x = byte0 | (byte1 << 8) | (byte2 << 16) | (byte3 << 24);
}

// 读取 64 位小端整数（第 341-383 行）
local int unz64local_getLong64(...) {
    // 依次读取 8 个字节，按 0/8/16/24/32/40/48/56 位移位合并
    x = byte0 | (byte1<<8) | ... | (byte7<<56);
}
```

> **为什么逐字节读取而不是直接 `memcpy` + 字节序转换？** 逐字节读取是最安全的跨平台方式——无论 CPU 是大端（Big-Endian，如某些 ARM 配置）还是小端，行为完全一致。虽然性能稍差，但 ZIP 头部解析是一次性操作，不在热路径上。

---

## 三、I/O 回调桥接机制

### 3.1 桥接架构概览

这是 KrKr2 ZIP 实现中最精巧的设计。标准 minizip 通过 `zlib_filefunc64_32_def` 结构体提供 I/O 回调函数指针，KrKr2 将这些函数指针指向自定义实现，内部操作 `tTJSBinaryStream`：

```
┌─────────────────────────────────────────────────────────┐
│                   I/O 桥接架构                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   minizip 解压引擎                                       │
│       │                                                 │
│       ├── open    → zip_open64file()  → 返回 this 指针  │
│       ├── read    → zip_readfile()    → _st->Read()     │
│       ├── write   → zip_writefile()   → _st->Write()    │
│       ├── tell    → zip_tell64file()  → _st->GetPosition() │
│       ├── seek    → zip_seek64file()  → _st->Seek()     │
│       └── close   → zip_closefile()   → 空操作           │
│                                                         │
│   其中 s（filestream）实际上是 ZipArchive* 指针           │
│   通过 ((ZipArchive*)s)->_st 访问底层流                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 3.2 回调函数实现详解

```cpp
// 打开文件：不真正打开，只是返回传入的指针（第 226-230 行）
static voidpf zip_open64file(voidpf opaque, const void *filename, int mode) {
    if(mode == (ZLIB_FILEFUNC_MODE_READ | ZLIB_FILEFUNC_MODE_EXISTING))
        return (voidpf)filename;  // filename 实际上是 ZipArchive* this 指针
    return nullptr;
}

// 读取数据：转发到 tTJSBinaryStream::Read()（第 2030-2032 行）
static uint32_t zip_readfile(voidpf, voidpf s, void *buf, uint32_t size) {
    return ((ZipArchive *)s)->_st->Read(buf, size);
}

// 写入数据：转发到 tTJSBinaryStream::Write()（第 2034-2037 行）
static uint32_t zip_writefile(voidpf, voidpf s, const void *buf, uint32_t size) {
    return ((ZipArchive *)s)->_st->Write(buf, size);
}

// 获取当前位置：转发到 tTJSBinaryStream::GetPosition()（第 2039-2041 行）
static ZPOS64_T zip_tell64file(voidpf, voidpf s) {
    return ((ZipArchive *)s)->_st->GetPosition();
}

// 定位：转发到 tTJSBinaryStream::Seek()（第 2043-2046 行）
static long zip_seek64file(voidpf, voidpf s, ZPOS64_T offset, int origin) {
    ((ZipArchive *)s)->_st->Seek(offset, origin);
    return 0;
}

// 关闭文件：空操作，流的生命周期由 ZipArchive 管理（第 240-243 行）
static int zip_closefile(voidpf, voidpf s) {
    // 注意：不删除流，因为 ZipArchive 析构函数会处理
    return 0;
}
```

### 3.3 回调函数表注册

所有回调函数通过一个静态结构体注册：

```cpp
// ZIPArchive.cpp 第 245-251 行
static zlib_filefunc64_32_def zipfunc = {
    { zip_open64file,   // 打开
      nullptr,          // opaque（未使用）
      zip_readfile,     // 读取
      zip_writefile,    // 写入
      zip_tell64file,   // 获取位置
      zip_seek64file,   // 定位
      zip_closefile,    // 关闭
      nullptr,          // 错误检测（未使用）
      nullptr           // 预留
    },
    nullptr,            // 32 位打开（未使用）
    nullptr,            // 32 位 tell（未使用）
    nullptr             // 32 位 seek（未使用）
};
```

> **设计巧妙之处**：`zip_open64file` 收到的 `filename` 参数实际上是 `ZipArchive*` 的 `this` 指针（见后面的 `unzOpenInternal` 调用）。打开后返回的 `filestream` 就是这个 `this` 指针，后续所有读写操作通过它访问 `_st` 成员（`tTJSBinaryStream*`）。这样 minizip 引擎完全不知道底层流的存在，实现了完美解耦。

---

## 四、中央目录搜索

### 4.1 搜索 EOCD 记录

ZIP 解析的第一步是从文件末尾向前搜索 EOCD 签名（`0x06054B50`）。这是必须的，因为 ZIP 文件可能在开头嵌入其他数据（如自解压可执行文件），只有 EOCD 能准确定位中央目录：

```cpp
// 搜索标准 EOCD（ZIPArchive.cpp 第 448-501 行，简化注释版）
local ZPOS64_T unz64local_SearchCentralDir(
    const zlib_filefunc64_32_def *pzlib_filefunc_def, 
    voidpf filestream) 
{
    unsigned char *buf;
    ZPOS64_T uSizeFile;
    ZPOS64_T uBackRead;
    ZPOS64_T uMaxBack = 0xffff;   // 最大回溯 64KB（ZIP 注释最大长度）
    ZPOS64_T uPosFound = 0;

    // 1. 获取文件总大小
    zip_seek64file(nullptr, filestream, 0, ZLIB_FILEFUNC_SEEK_END);
    uSizeFile = zip_tell64file(nullptr, filestream);

    if(uMaxBack > uSizeFile)
        uMaxBack = uSizeFile;     // 小文件：最多回溯到文件开头

    buf = (unsigned char *)ALLOC(BUFREADCOMMENT + 4);  // 1KB + 4 缓冲区

    // 2. 从文件末尾向前分块扫描
    uBackRead = 4;
    while(uBackRead < uMaxBack) {
        // 每次多读 BUFREADCOMMENT (1024) 字节
        if(uBackRead + BUFREADCOMMENT > uMaxBack)
            uBackRead = uMaxBack;
        else
            uBackRead += BUFREADCOMMENT;
        
        ZPOS64_T uReadPos = uSizeFile - uBackRead;
        uLong uReadSize = min(BUFREADCOMMENT + 4, uSizeFile - uReadPos);

        // 读取这块数据
        zip_seek64file(nullptr, filestream, uReadPos, ZLIB_FILEFUNC_SEEK_SET);
        zip_readfile(nullptr, filestream, buf, uReadSize);

        // 3. 在缓冲区内从后向前搜索签名 PK\x05\x06
        for(int i = (int)uReadSize - 3; (i--) > 0;) {
            if(buf[i]     == 0x50 &&   // 'P'
               buf[i + 1] == 0x4b &&   // 'K'
               buf[i + 2] == 0x05 &&   // \x05
               buf[i + 3] == 0x06) {   // \x06
                uPosFound = uReadPos + i;
                break;
            }
        }
        if(uPosFound != 0) break;
    }
    TRYFREE(buf);
    return uPosFound;  // 返回 EOCD 在文件中的偏移量
}
```

### 4.2 搜索 ZIP64 EOCD 定位器

如果 ZIP 使用了 ZIP64 扩展，还需要搜索 ZIP64 EOCD 定位器（签名 `0x07064B50`）：

```cpp
// 搜索 ZIP64 结束定位器（ZIPArchive.cpp 第 510-608 行，简化）
local ZPOS64_T unz64local_SearchCentralDir64(...) {
    // 类似搜索过程，但寻找签名 PK\x06\x07 (0x07064B50)
    
    // 找到后，从定位器中读取 ZIP64 EOCD 的偏移
    // ZIP64 EOCD 的签名是 0x06064B50
    
    // ZIP64 EOCD 定位器结构：
    // 偏移 0: 签名 0x07064B50
    // 偏移 4: ZIP64 EOCD 所在磁盘号（4 字节）
    // 偏移 8: ZIP64 EOCD 的相对偏移（8 字节，64 位！）
    // 偏移 16: 总磁盘数（4 字节）
    
    unz64local_getLong64(pzlib_filefunc_def, filestream, &relativeOffset);
    
    // 跳转到 ZIP64 EOCD，验证签名 0x06064B50
    zip_seek64file(nullptr, filestream, relativeOffset, ZLIB_FILEFUNC_SEEK_SET);
    unz64local_getLong(pzlib_filefunc_def, filestream, &uL);
    if(uL != 0x06064b50) return 0;  // 签名不匹配
    
    return relativeOffset;
}
```

### 4.3 打开 ZIP 文件：unzOpenInternal

这是 ZIP 解析的核心函数，负责定位中央目录并初始化内部状态：

```cpp
// ZIPArchive.cpp 第 621-794 行（简化注释版）
local unzFile unzOpenInternal(const void *path, ...) {
    unz64_s us;    // ZIP 内部状态结构
    ZPOS64_T central_pos;
    
    // 1. "打开"文件（实际上 path 就是 ZipArchive* this 指针）
    us.filestream = zip_open64file(nullptr, path, 
        ZLIB_FILEFUNC_MODE_READ | ZLIB_FILEFUNC_MODE_EXISTING);
    
    // 2. 先尝试 ZIP64 格式
    central_pos = unz64local_SearchCentralDir64(nullptr, us.filestream);
    if(central_pos) {
        us.isZip64 = 1;
        // 从 ZIP64 EOCD 读取：
        // - 中央目录条目数（64 位）
        // - 中央目录大小（64 位）
        // - 中央目录偏移（64 位）
        unz64local_getLong64(nullptr, us.filestream, &us.gi.number_entry);
        unz64local_getLong64(nullptr, us.filestream, &us.size_central_dir);
        unz64local_getLong64(nullptr, us.filestream, &us.offset_central_dir);
    } else {
        // 3. 回退到标准 ZIP 格式
        central_pos = unz64local_SearchCentralDir(nullptr, us.filestream);
        us.isZip64 = 0;
        // 从标准 EOCD 读取（字段较短，2/4 字节）
        unz64local_getShort(nullptr, us.filestream, &uL);
        us.gi.number_entry = uL;         // 条目数（16 位，最多 65535）
        unz64local_getLong(nullptr, us.filestream, &uL);
        us.size_central_dir = uL;        // 目录大小（32 位）
        unz64local_getLong(nullptr, us.filestream, &uL);
        us.offset_central_dir = uL;      // 目录偏移（32 位，最多 4GB）
    }
    
    // 4. 计算 ZIP 数据前可能存在的前缀偏移（自解压 EXE 等场景）
    us.byte_before_the_zipfile = 
        central_pos - (us.offset_central_dir + us.size_central_dir);
    
    // 5. 分配并初始化状态，跳转到第一个文件
    s = (unz64_s *)ALLOC(sizeof(unz64_s));
    *s = us;
    unzGoToFirstFile64((unzFile)s, nullptr, nullptr, 0);
    return (unzFile)s;
}
```

> **`byte_before_the_zipfile` 的作用**：如果 ZIP 数据不是从文件偏移 0 开始（比如自解压 EXE 中 ZIP 数据在 EXE 代码之后），这个值记录了 ZIP 数据前的偏移量。后续所有偏移计算都需要加上这个值。

---

## 五、ZipArchive 类实现

### 5.1 类定义

`ZipArchive` 类继承自 `tTVPArchive`，是 KrKr2 存储系统与 minizip 之间的桥梁：

```cpp
// ZIPArchive.cpp 第 2009-2028 行
class ZipArchive : public tTVPArchive {
    unzFile uf;    // minizip 内部句柄
    
    // 文件条目列表：文件名 + 在中央目录中的位置
    typedef std::pair<ttstr, unz64_file_pos> FileEntry;
    std::vector<FileEntry> filelist;

public:
    ~ZipArchive() override;
    
    // 构造函数：解析整个 ZIP 目录
    ZipArchive(const ttstr &name, tTJSBinaryStream *st, bool normalizeFileName);
    
    // 检查是否成功打开
    bool isValid() { return uf != nullptr; }
    
    // --- tTVPArchive 纯虚函数实现 ---
    tjs_uint GetCount() override { return filelist.size(); }
    ttstr GetName(tjs_uint idx) override { return filelist[idx].first; }
    tTJSBinaryStream *CreateStreamByIndex(tjs_uint idx) override;
    
    tTJSBinaryStream *_st = nullptr;  // 底层二进制流（public，供 I/O 回调访问）
};
```

### 5.2 构造函数——解析中央目录

```cpp
// ZIPArchive.cpp 第 2108-2138 行
ZipArchive::ZipArchive(const ttstr &name, tTJSBinaryStream *st,
                       bool normalizeFileName) : tTVPArchive(name) {
    if(!st)
        st = TVPCreateStream(name);    // 如果没有提供流，根据名称创建
    _st = st;
    
    // 1. 通过 minizip 打开 ZIP（this 指针作为"文件名"传入）
    if((uf = unzOpenInternal(this, &zipfunc, 1)) != nullptr) {
        unz_file_info file_info;
        
        // 2. 遍历中央目录的所有条目
        do {
            unz64_file_pos entry;
            if(unzGetFilePos64(uf, &entry) == UNZ_OK) {
                ttstr filename;
                char filename_inzip[1024];
                
                // 读取文件信息和文件名
                if(unzGetCurrentFileInfo(uf, &file_info, filename_inzip,
                                         sizeof(filename_inzip), 
                                         nullptr, 0) == UNZ_OK) {
                    // 将窄字符文件名转换为宽字符
                    storeFilename(filename, filename_inzip, name);
                    
                    // 标准化文件名（小写 + 统一分隔符）
                    if(normalizeFileName)
                        NormalizeInArchiveStorageName(filename);
                    
                    // 存储文件名和中央目录位置
                    filelist.emplace_back(filename, entry);
                }
            }
        } while(unzGoToNextFile64(uf, nullptr, nullptr, 0) == UNZ_OK);
        
        // 3. 按文件名排序，支持 tTVPArchive 基类的二分查找
        if(normalizeFileName) {
            std::sort(filelist.begin(), filelist.end(),
                      [](const FileEntry &a, const FileEntry &b) {
                          return a.first < b.first;
                      });
        }
    }
}
```

> **关键点**：`unzGetFilePos64` 记录的是当前文件在中央目录中的位置（`pos_in_zip_directory`）和文件序号（`num_of_file`），后续 `CreateStreamByIndex` 通过 `unzGoToFilePos64` 快速定位到指定文件。

### 5.3 CreateStreamByIndex——按需创建流

这是 `ZipArchive` 最核心的方法，根据压缩状态选择不同的策略：

```cpp
// ZIPArchive.cpp 第 2064-2093 行
tTJSBinaryStream *ZipArchive::CreateStreamByIndex(tjs_uint idx) {
    // 1. 跳转到中央目录中的对应条目
    if(unzGoToFilePos64(uf, &filelist[idx].second) != UNZ_OK)
        return nullptr;
    
    // 2. 获取文件信息（特别是压缩方法和大小）
    unz_file_info file_info;
    if(unzGetCurrentFileInfo(uf, &file_info, nullptr, 0, nullptr, 0) != UNZ_OK)
        return nullptr;
    
    // 3. 根据压缩方法选择策略
    if(file_info.compression_method == 0) {
        // === 未压缩（Store）：直接用 TArchiveStream 包装 ===
        uInt iSizeVar;
        ZPOS64_T offset_local_extrafield;
        uInt size_local_extrafield;
        
        // 验证本地文件头一致性
        if(unz64local_CheckCurrentFileCoherencyHeader(
               (unz64_s *)uf, &iSizeVar, 
               &offset_local_extrafield, &size_local_extrafield) != UNZ_OK)
            return nullptr;
        
        // 计算数据起始偏移 = 本地头偏移 + 固定头大小(30) + 变长字段
        return new TArchiveStream(
            this,
            file_info.offset_curfile + SIZEZIPLOCALHEADER + iSizeVar,
            file_info.uncompressed_size);
    } else {
        // === 已压缩（Deflate 等）：解压到内存流 ===
        if(unzOpenCurrentFile(uf) != UNZ_OK)
            return nullptr;
        
        // 创建内存流并预分配空间
        tTVPMemoryStream *mem = new tTVPMemoryStream();
        mem->SetSize(file_info.uncompressed_size);
        
        // 一次性读取全部解压数据到内存
        unzReadCurrentFile(uf, mem->GetInternalBuffer(),
                           file_info.uncompressed_size);
        unzCloseCurrentFile(uf);
        
        return mem;  // 返回内存流，支持随机访问
    }
}
```

这里有两个关键的性能优化策略：

| 策略 | 条件 | 流类型 | 优势 |
|------|------|--------|------|
| 零拷贝读取 | `compression_method == 0` | `TArchiveStream` | 不需要解压，直接从 ZIP 文件中读取原始数据，节省内存 |
| 全量解压 | `compression_method != 0` | `tTVPMemoryStream` | 一次解压到内存，后续随机访问无需重复解压 |

> **与 7z 实现的对比**：7z 归档也使用相同的双策略（参见后续 03-7z归档实现.md），但 7z 的"无需解压"判断条件更复杂（需要检查 coder 数量和类型）。

### 5.4 析构函数与资源清理

```cpp
// ZIPArchive.cpp 第 2095-2104 行
ZipArchive::~ZipArchive() {
    if(uf) {
        unzClose(uf);       // 关闭 minizip 句柄（释放内部分配的内存）
        uf = nullptr;
    }
    if(_st) {
        delete _st;         // 删除底层二进制流
        _st = nullptr;
    }
}
```

### 5.5 格式检测与工厂函数

KrKr2 通过 `TVPOpenZIPArchive` 函数检测文件是否为 ZIP 格式并创建归档对象：

```cpp
// ZIPArchive.cpp 第 2048-2062 行
tTVPArchive *TVPOpenZIPArchive(const ttstr &name, tTJSBinaryStream *st,
                               bool normalizeFileName) {
    tjs_uint64 pos = st->GetPosition();
    
    // 检查前两个字节是否为 'PK'（0x504B）
    bool checkZIP = st->ReadI16LE() == 0x4B50;  // 注意小端序：P=0x50, K=0x4B
    st->SetPosition(pos);     // 恢复读取位置
    
    if(!checkZIP)
        return nullptr;       // 不是 ZIP 文件
    
    ZipArchive *arc = new ZipArchive(name, st, normalizeFileName);
    if(!arc->isValid()) {
        arc->_st = nullptr;   // 防止析构时 double free
        delete arc;
        return nullptr;       // ZIP 解析失败
    }
    return arc;
}
```

> **注意 `0x4B50` 的字节序问题**：`ReadI16LE()` 以小端序读取 16 位整数。文件中的字节序是 `0x50, 0x4B`（即 ASCII `'P', 'K'`），小端读取后得到 `0x4B50`。这是正确的——小端序中低地址存放低字节。

---

## 六、Deflate 解压流程

### 6.1 打开当前文件进行读取

`unzOpenCurrentFile3` 初始化解压状态：

```cpp
// ZIPArchive.cpp 第 1382-1550 行（简化）
int ZEXPORT unzOpenCurrentFile3(unzFile file, int *method, int *level, 
                                int raw, const char *password) {
    unz64_s *s = (unz64_s *)file;
    
    // 1. 验证本地文件头与中央目录一致性
    unz64local_CheckCurrentFileCoherencyHeader(s, &iSizeVar, ...);
    
    // 2. 分配读取状态结构
    pfile_in_zip_read_info = ALLOC(sizeof(file_in_zip64_read_info_s));
    pfile_in_zip_read_info->read_buffer = ALLOC(UNZ_BUFSIZE);  // 16KB 缓冲
    
    // 3. 初始化 zlib 解压器（Deflate）
    if(s->cur_file_info.compression_method == Z_DEFLATED) {
        // -MAX_WBITS 表示 raw deflate（无 zlib 头部）
        err = inflateInit2(&pfile_in_zip_read_info->stream, -MAX_WBITS);
        pfile_in_zip_read_info->stream_initialised = Z_DEFLATED;
    }
    
    // 4. 记录待读取的字节数
    pfile_in_zip_read_info->rest_read_compressed = 
        s->cur_file_info.compressed_size;
    pfile_in_zip_read_info->rest_read_uncompressed = 
        s->cur_file_info.uncompressed_size;
    
    // 5. 计算压缩数据的起始偏移
    pfile_in_zip_read_info->pos_in_zipfile = 
        s->cur_file_info_internal.offset_curfile + SIZEZIPLOCALHEADER + iSizeVar;
    
    s->pfile_in_zip_read = pfile_in_zip_read_info;
    return UNZ_OK;
}
```

### 6.2 读取解压数据

`unzReadCurrentFile` 是实际的解压读取函数，处理分块读取和流式解压：

```cpp
// ZIPArchive.cpp 第 1583-1785 行（简化注释版）
int ZEXPORT unzReadCurrentFile(unzFile file, voidp buf, unsigned len) {
    file_in_zip64_read_info_s *pinfo = s->pfile_in_zip_read;
    
    pinfo->stream.next_out = (Bytef *)buf;      // 输出缓冲区
    pinfo->stream.avail_out = (uInt)len;         // 请求的字节数
    
    // 限制输出不超过未压缩剩余量
    if(len > pinfo->rest_read_uncompressed)
        pinfo->stream.avail_out = (uInt)pinfo->rest_read_uncompressed;
    
    while(pinfo->stream.avail_out > 0) {
        // --- 阶段 1：补充输入缓冲区 ---
        if(pinfo->stream.avail_in == 0 && pinfo->rest_read_compressed > 0) {
            uInt uReadThis = min(UNZ_BUFSIZE, pinfo->rest_read_compressed);
            
            // 定位并读取压缩数据
            zip_seek64file(nullptr, pinfo->filestream,
                pinfo->pos_in_zipfile + pinfo->byte_before_the_zipfile,
                ZLIB_FILEFUNC_SEEK_SET);
            zip_readfile(nullptr, pinfo->filestream, 
                pinfo->read_buffer, uReadThis);
            
            pinfo->pos_in_zipfile += uReadThis;
            pinfo->rest_read_compressed -= uReadThis;
            pinfo->stream.next_in = (Bytef *)pinfo->read_buffer;
            pinfo->stream.avail_in = uReadThis;
        }
        
        // --- 阶段 2：处理数据 ---
        if(pinfo->compression_method == 0) {
            // 未压缩：直接复制
            uInt uDoCopy = min(pinfo->stream.avail_out, pinfo->stream.avail_in);
            memcpy(pinfo->stream.next_out, pinfo->stream.next_in, uDoCopy);
            // 更新 CRC 和各种计数器...
        } else {
            // Deflate 压缩：调用 zlib 解压
            err = inflate(&pinfo->stream, Z_SYNC_FLUSH);
            // 更新 CRC 和计数器...
            
            if(err == Z_STREAM_END) return iRead;  // 解压完成
            if(err != Z_OK) break;                 // 解压错误
        }
    }
    return iRead;  // 返回实际读取的字节数
}
```

流式解压的数据流图：

```
ZIP 文件中的压缩数据
    │
    ▼
┌──────────────────────────┐
│  read_buffer (16KB)      │  ← 从 ZIP 文件分块读取压缩数据
│  [compressed chunks]     │
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│  zlib inflate()          │  ← 流式解压（Z_SYNC_FLUSH）
│  stream.avail_in → out   │
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│  用户缓冲区 buf          │  ← 解压后的数据写入用户提供的缓冲区
│  [uncompressed data]     │
└──────────────────────────┘
```

---

## 七、ZIP64 扩展支持

### 7.1 为什么需要 ZIP64

标准 ZIP 格式有三个硬性限制：
- 单个文件大小最大 **4GB**（32 位 `compressed_size` / `uncompressed_size`）
- 归档文件总大小最大 **4GB**（32 位偏移量）
- 最多 **65535** 个文件（16 位 `number_entry`）

ZIP64 扩展通过将这些字段扩展到 64 位来解决这些限制。

### 7.2 KrKr2 中的 ZIP64 处理

ZIP64 信息存储在扩展字段中，Header ID 为 `0x0001`：

```cpp
// ZIPArchive.cpp 第 1005-1056 行（中央目录解析中的 ZIP64 扩展字段处理）
while(acc < file_info.size_file_extra) {
    uLong headerId;
    uLong dataSize;
    
    unz64local_getShort(nullptr, s->filestream, &headerId);
    unz64local_getShort(nullptr, s->filestream, &dataSize);
    
    if(headerId == 0x0001) {
        // ZIP64 扩展字段！
        
        // 当原始大小标记为 0xFFFFFFFF 时，从扩展字段读取 64 位值
        if(file_info.uncompressed_size == (ZPOS64_T)(unsigned long)-1) {
            unz64local_getLong64(nullptr, s->filestream, 
                &file_info.uncompressed_size);
        }
        
        // 压缩大小同理
        if(file_info.compressed_size == (ZPOS64_T)(unsigned long)-1) {
            unz64local_getLong64(nullptr, s->filestream, 
                &file_info.compressed_size);
        }
        
        // 本地头偏移同理
        if(file_info_internal.offset_curfile == (ZPOS64_T)(unsigned long)-1) {
            unz64local_getLong64(nullptr, s->filestream, 
                &file_info_internal.offset_curfile);
        }
    } else {
        // 跳过其他扩展字段
        zip_seek64file(nullptr, s->filestream, dataSize, ZLIB_FILEFUNC_SEEK_CUR);
    }
    
    acc += 2 + 2 + dataSize;  // headerId(2) + dataSize(2) + data
}
```

> **哨兵值机制**：标准字段中的值 `0xFFFFFFFF`（32 位全 1）和 `0xFFFF`（16 位全 1）是 ZIP64 的哨兵值（Sentinel Value），表示"真实值在 ZIP64 扩展字段中"。KrKr2 只在检测到哨兵值时才读取对应的 64 位扩展字段。

### 7.3 65535 文件数溢出处理

KrKr2 还处理了一个边界情况——当文件数恰好为 65535 时的遍历逻辑：

```cpp
// ZIPArchive.cpp 第 1174-1199 行
int ZEXPORT unzGoToNextFile64(unzFile file, ...) {
    unz64_s *s = (unz64_s *)file;
    
    if(!s->current_file_ok)
        return UNZ_END_OF_LIST_OF_FILE;
    
    // 当 number_entry 为 0xFFFF 时跳过文件数检查（ZIP64 的哨兵值）
    if(s->gi.number_entry != 0xffff)
        if(s->num_file + 1 == s->gi.number_entry)
            return UNZ_END_OF_LIST_OF_FILE;
    
    // 移动到下一个中央目录条目
    s->pos_in_central_dir += SIZECENTRALDIRITEM + 
        s->cur_file_info.size_filename + 
        s->cur_file_info.size_file_extra + 
        s->cur_file_info.size_file_comment;
    s->num_file++;
    
    // 读取下一个条目信息
    err = unz64local_GetCurrentFileInfoInternal(file, ...);
    s->current_file_ok = (err == UNZ_OK);
    return err;
}
```

---

## 八、DOS 日期时间转换

ZIP 格式使用 MS-DOS 风格的日期时间编码。KrKr2 将其转换为更易读的 `tm_unz` 结构：

```cpp
// ZIPArchive.cpp 第 844-854 行
local void unz64local_DosDateToTmuDate(ZPOS64_T ulDosDate, tm_unz *ptm) {
    ZPOS64_T uDate;
    uDate = (ZPOS64_T)(ulDosDate >> 16);   // 高 16 位 = 日期部分
    
    // 日期解析
    ptm->tm_mday = (uInt)(uDate & 0x1f);                    // 位 0-4: 日
    ptm->tm_mon  = (uInt)((((uDate) & 0x1E0) / 0x20) - 1);  // 位 5-8: 月(1-12→0-11)
    ptm->tm_year = (uInt)(((uDate & 0x0FE00) / 0x0200) + 1980); // 位 9-15: 年(+1980)
    
    // 时间解析（低 16 位）
    ptm->tm_hour = (uInt)((ulDosDate & 0xF800) / 0x800);    // 位 11-15: 时
    ptm->tm_min  = (uInt)((ulDosDate & 0x7E0) / 0x20);      // 位 5-10: 分
    ptm->tm_sec  = (uInt)(2 * (ulDosDate & 0x1f));           // 位 0-4: 秒/2
}
```

DOS 日期时间编码的位布局：

```
日期（高 16 位）：
  位 15-9: 年份 - 1980 (0-127, 对应 1980-2107)
  位 8-5:  月份 (1-12)
  位 4-0:  日 (1-31)

时间（低 16 位）：
  位 15-11: 小时 (0-23)
  位 10-5:  分钟 (0-59)
  位 4-0:   秒/2 (0-29, 对应 0-58，精度为 2 秒)
```

---

## 九、完整示例：手动解析 ZIP 文件

以下是一个独立可编译的示例，演示如何从零开始手动解析 ZIP 文件结构：

```cpp
#include <cstdint>
#include <cstring>
#include <fstream>
#include <iostream>
#include <string>
#include <vector>

// ZIP 签名常量
constexpr uint32_t ZIP_LOCAL_HEADER_SIG = 0x04034B50;    // PK\x03\x04
constexpr uint32_t ZIP_CENTRAL_DIR_SIG  = 0x02014B50;    // PK\x01\x02
constexpr uint32_t ZIP_EOCD_SIG         = 0x06054B50;    // PK\x05\x06

// 从小端序字节流读取 16 位整数
uint16_t readU16LE(std::ifstream &f) {
    uint8_t buf[2];
    f.read(reinterpret_cast<char*>(buf), 2);
    return buf[0] | (buf[1] << 8);  // 小端：低字节在前
}

// 从小端序字节流读取 32 位整数
uint32_t readU32LE(std::ifstream &f) {
    uint8_t buf[4];
    f.read(reinterpret_cast<char*>(buf), 4);
    return buf[0] | (buf[1] << 8) | (buf[2] << 16) | (buf[3] << 24);
}

// 搜索 EOCD 记录
int64_t findEOCD(std::ifstream &f) {
    f.seekg(0, std::ios::end);
    int64_t fileSize = f.tellg();
    int64_t maxBack = std::min<int64_t>(fileSize, 0xFFFF + 22);  // EOCD 最小 22 字节
    
    // 从文件末尾向前搜索
    std::vector<uint8_t> buf(maxBack);
    f.seekg(fileSize - maxBack);
    f.read(reinterpret_cast<char*>(buf.data()), maxBack);
    
    // 从后向前扫描签名 0x06054B50
    for(int64_t i = maxBack - 4; i >= 0; --i) {
        if(buf[i]   == 0x50 && buf[i+1] == 0x4B &&
           buf[i+2] == 0x05 && buf[i+3] == 0x06) {
            return (fileSize - maxBack) + i;
        }
    }
    return -1;  // 未找到
}

int main(int argc, char* argv[]) {
    if(argc < 2) {
        std::cerr << "用法: " << argv[0] << " <zip文件路径>" << std::endl;
        return 1;
    }
    
    std::ifstream f(argv[1], std::ios::binary);
    if(!f.is_open()) {
        std::cerr << "无法打开文件: " << argv[1] << std::endl;
        return 1;
    }
    
    // 1. 搜索 EOCD
    int64_t eocdPos = findEOCD(f);
    if(eocdPos < 0) {
        std::cerr << "无法找到 EOCD 记录，不是有效的 ZIP 文件" << std::endl;
        return 1;
    }
    std::cout << "EOCD 位置: 0x" << std::hex << eocdPos << std::dec << std::endl;
    
    // 2. 解析 EOCD
    f.seekg(eocdPos + 4);  // 跳过签名
    uint16_t diskNum       = readU16LE(f);  // 当前磁盘号
    uint16_t cdDisk        = readU16LE(f);  // 中央目录起始磁盘号
    uint16_t cdEntriesThis = readU16LE(f);  // 本磁盘条目数
    uint16_t cdEntriesAll  = readU16LE(f);  // 总条目数
    uint32_t cdSize        = readU32LE(f);  // 中央目录大小
    uint32_t cdOffset      = readU32LE(f);  // 中央目录偏移
    uint16_t commentLen    = readU16LE(f);  // 注释长度
    
    std::cout << "文件总数: " << cdEntriesAll << std::endl;
    std::cout << "中央目录偏移: 0x" << std::hex << cdOffset << std::dec << std::endl;
    std::cout << "中央目录大小: " << cdSize << " 字节" << std::endl;
    
    // 3. 遍历中央目录
    f.seekg(cdOffset);
    std::cout << "\n--- 文件列表 ---" << std::endl;
    
    for(int i = 0; i < cdEntriesAll; ++i) {
        uint32_t sig = readU32LE(f);
        if(sig != ZIP_CENTRAL_DIR_SIG) {
            std::cerr << "错误：中央目录签名不匹配！" << std::endl;
            break;
        }
        
        uint16_t versionMade   = readU16LE(f);  // 创建版本
        uint16_t versionNeeded = readU16LE(f);  // 解压所需版本
        uint16_t flags         = readU16LE(f);  // 通用位标志
        uint16_t method        = readU16LE(f);  // 压缩方法
        uint32_t dosDateTime   = readU32LE(f);  // DOS 日期时间
        uint32_t crc32         = readU32LE(f);  // CRC-32
        uint32_t compSize      = readU32LE(f);  // 压缩大小
        uint32_t uncompSize    = readU32LE(f);  // 原始大小
        uint16_t nameLen       = readU16LE(f);  // 文件名长度
        uint16_t extraLen      = readU16LE(f);  // 扩展字段长度
        uint16_t commentL      = readU16LE(f);  // 注释长度
        uint16_t diskStart     = readU16LE(f);  // 起始磁盘号
        uint16_t intAttr       = readU16LE(f);  // 内部属性
        uint32_t extAttr       = readU32LE(f);  // 外部属性
        uint32_t localOffset   = readU32LE(f);  // 本地文件头偏移
        
        // 读取文件名
        std::string filename(nameLen, '\0');
        f.read(&filename[0], nameLen);
        
        // 跳过扩展字段和注释
        f.seekg(extraLen + commentL, std::ios::cur);
        
        // 输出文件信息
        const char* methodStr = (method == 0) ? "存储" : 
                                (method == 8) ? "Deflate" : "其他";
        std::cout << "  [" << i << "] " << filename 
                  << "  大小:" << uncompSize 
                  << "  压缩:" << compSize
                  << "  方法:" << methodStr
                  << "  CRC:0x" << std::hex << crc32 << std::dec
                  << std::endl;
    }
    
    return 0;
}
```

编译和运行：

```bash
# Linux / macOS
g++ -std=c++17 -o zip_parser zip_parser.cpp
./zip_parser test.zip

# Windows (MSVC)
cl /std:c++17 /EHsc zip_parser.cpp
zip_parser.exe test.zip

# 预期输出示例：
# EOCD 位置: 0x1a3f
# 文件总数: 5
# 中央目录偏移: 0x19b7
# 中央目录大小: 136 字节
#
# --- 文件列表 ---
#   [0] readme.txt  大小:1024  压缩:512  方法:Deflate  CRC:0xa1b2c3d4
#   [1] data/config.json  大小:256  压缩:256  方法:存储  CRC:0xe5f6a7b8
#   ...
```

---

## 十、动手实践

### 实践 1：追踪 ZIP 打开流程

在 KrKr2 源码中，从 `TVPOpenZIPArchive` 开始追踪完整的 ZIP 打开流程：

1. 打开 `cpp/core/base/ZIPArchive.cpp`，定位到第 2048 行 `TVPOpenZIPArchive`
2. 观察 `ReadI16LE() == 0x4B50` 的签名检测
3. 进入 `ZipArchive` 构造函数（第 2108 行），观察 `unzOpenInternal(this, &zipfunc, 1)` 调用
4. 注意 `this` 指针如何作为"文件名"传入，又如何在 `zip_open64file` 中被原样返回
5. 追踪 `unzGoToFirstFile64` → `unz64local_GetCurrentFileInfoInternal` 的调用链

### 实践 2：对比未压缩/压缩文件的读取路径

1. 创建一个测试 ZIP 文件，包含一个未压缩文件和一个 Deflate 压缩文件：
   ```bash
   echo "hello" > test.txt
   zip -0 mixed.zip test.txt       # -0 表示不压缩（Store）
   echo "world" > test2.txt
   zip mixed.zip test2.txt         # 默认 Deflate 压缩
   ```
2. 在 `CreateStreamByIndex` 中设置断点
3. 观察 `compression_method == 0` 分支返回 `TArchiveStream`，而压缩分支返回 `tTVPMemoryStream`

---

## 对照项目源码

相关文件：
- `cpp/core/base/ZIPArchive.cpp` 第 1-2138 行 — 完整的 ZIP 归档实现，包含内嵌的 minizip 解压引擎
- `cpp/core/base/StorageIntf.h` 第 50-80 行 — `tTVPArchive` 基类定义和 `TArchiveStream` 类
- `cpp/core/base/StorageIntf.cpp` 第 280-320 行 — `TVPOpenArchive` 函数，按顺序尝试各种归档格式
- `cpp/core/base/UtilStreams.h` 第 30-60 行 — `tTVPMemoryStream` 定义（ZIP 解压的目标容器）

---

## 本节小结

- ZIP 格式采用"先数据后目录"的结构，解析从文件末尾的 EOCD 记录开始
- KrKr2 将 minizip 解压代码直接内嵌到 `ZIPArchive.cpp`（2138 行），不依赖外部库
- I/O 桥接通过 `zlib_filefunc64_32_def` 回调函数表实现，将 `tTJSBinaryStream` 伪装为文件操作
- 巧妙地利用 `this` 指针作为 minizip 的"文件名"参数，实现了对象与 C 风格 API 的桥接
- 未压缩文件使用 `TArchiveStream`（零拷贝），压缩文件解压到 `tTVPMemoryStream`（全量解压）
- ZIP64 扩展通过哨兵值（`0xFFFFFFFF`）触发 64 位字段读取，突破 4GB/65535 限制

---

## 练习题与答案

### 题目 1：I/O 桥接机制分析

在 KrKr2 的 ZIP 实现中，`zip_open64file` 函数接收的 `filename` 参数实际上是什么？为什么可以这样做？如果 minizip 对 `filename` 做了字符串操作（如 `strlen`），会发生什么问题？

<details>
<summary>查看答案</summary>

`filename` 实际上是 `ZipArchive*` 的 `this` 指针（通过 `unzOpenInternal(this, &zipfunc, 1)` 传入）。

可以这样做的原因：
1. `zip_open64file` 的 `filename` 参数类型是 `const void*`，可以接受任何指针
2. 函数内部只是将指针原样返回，不对其做任何字符串解释
3. 后续的 `zip_readfile` 等回调将收到的 `filestream`（即返回的 `filename`）强制转换回 `ZipArchive*` 使用

如果 minizip 对 `filename` 做了字符串操作（如 `strlen`），会导致**未定义行为**——`this` 指针指向的内存是 `ZipArchive` 对象，不是以 `\0` 结尾的字符串，`strlen` 会一直读取内存直到碰巧遇到 `\0` 字节或触发访问违规（Segmentation Fault）。KrKr2 的实现之所以安全，正是因为它完全重写了所有 I/O 回调函数，确保没有任何地方将 `filename` 当作字符串处理。

</details>

### 题目 2：未压缩文件优化原理

`CreateStreamByIndex` 对未压缩文件（`compression_method == 0`）返回 `TArchiveStream` 而非 `tTVPMemoryStream`。请解释这两种方式的内存占用差异，并说明在什么场景下 `TArchiveStream` 的优势最明显。

<details>
<summary>查看答案</summary>

**内存占用对比：**

| 方式 | 内存占用 | 说明 |
|------|----------|------|
| `TArchiveStream` | 几十字节（对象本身） | 不复制数据，直接在原 ZIP 文件的底层流上设置偏移和长度窗口 |
| `tTVPMemoryStream` | `uncompressed_size` 字节 | 需要将整个文件内容复制到内存中 |

**`TArchiveStream` 优势最明显的场景：**

1. **大文件**：如果 ZIP 中包含未压缩的大文件（如 100MB 的 BGM 音频），使用 `TArchiveStream` 完全不需要额外内存，而 `tTVPMemoryStream` 需要分配 100MB 内存
2. **顺序读取**：音频播放等场景通常是顺序读取，`TArchiveStream` 直接从底层流读取，效率等同于直接读文件
3. **内存受限环境**：Android 设备内存有限，避免大文件的内存复制非常重要
4. **文件仅被部分读取**：如果只需要读取文件的前几KB（如检查文件头），`TArchiveStream` 不会浪费资源读取整个文件

**`tTVPMemoryStream` 优于 `TArchiveStream` 的场景：**
- 需要频繁随机访问的小文件（如纹理元数据），内存流的随机访问更快（无需系统调用）

</details>

### 题目 3：实现一个简化的 ZIP 文件列表器

编写一个 C++ 程序，使用 KrKr2 类似的逐字节读取方法解析 ZIP 文件的 EOCD 和中央目录，输出每个文件的名称、压缩方法和大小。要求：支持 ZIP64 扩展字段中的大文件大小。

<details>
<summary>查看答案</summary>

```cpp
#include <cstdint>
#include <fstream>
#include <iostream>
#include <string>
#include <vector>

// 逐字节读取，与 KrKr2 的 unz64local_getByte 原理相同
bool getByte(std::ifstream &f, uint8_t &out) {
    return f.read(reinterpret_cast<char*>(&out), 1).good();
}

// 小端序读取 16 位（对应 unz64local_getShort）
bool getU16(std::ifstream &f, uint16_t &out) {
    uint8_t b0, b1;
    if(!getByte(f, b0) || !getByte(f, b1)) return false;
    out = b0 | (static_cast<uint16_t>(b1) << 8);
    return true;
}

// 小端序读取 32 位（对应 unz64local_getLong）
bool getU32(std::ifstream &f, uint32_t &out) {
    uint8_t b[4];
    for(int i = 0; i < 4; ++i)
        if(!getByte(f, b[i])) return false;
    out = b[0] | (b[1] << 8) | (b[2] << 16) | (static_cast<uint32_t>(b[3]) << 24);
    return true;
}

// 小端序读取 64 位（对应 unz64local_getLong64）
bool getU64(std::ifstream &f, uint64_t &out) {
    uint8_t b[8];
    for(int i = 0; i < 8; ++i)
        if(!getByte(f, b[i])) return false;
    out = 0;
    for(int i = 7; i >= 0; --i)
        out = (out << 8) | b[i];
    return true;
}

int main(int argc, char* argv[]) {
    if(argc < 2) {
        std::cerr << "用法: " << argv[0] << " <zip文件>" << std::endl;
        return 1;
    }
    
    std::ifstream f(argv[1], std::ios::binary);
    if(!f.is_open()) { std::cerr << "打开失败" << std::endl; return 1; }
    
    // 搜索 EOCD（从文件末尾向前扫描）
    f.seekg(0, std::ios::end);
    int64_t fileSize = f.tellg();
    int64_t searchLen = std::min<int64_t>(fileSize, 0xFFFF + 22);
    std::vector<uint8_t> tail(searchLen);
    f.seekg(fileSize - searchLen);
    f.read(reinterpret_cast<char*>(tail.data()), searchLen);
    
    int64_t eocdOff = -1;
    for(int64_t i = searchLen - 22; i >= 0; --i) {
        if(tail[i]==0x50 && tail[i+1]==0x4B && tail[i+2]==0x05 && tail[i+3]==0x06) {
            eocdOff = (fileSize - searchLen) + i;
            break;
        }
    }
    if(eocdOff < 0) { std::cerr << "无效 ZIP" << std::endl; return 1; }
    
    // 解析 EOCD
    f.seekg(eocdOff + 4);
    uint16_t tmp16; uint32_t tmp32;
    getU16(f, tmp16); // disk
    getU16(f, tmp16); // cd disk
    getU16(f, tmp16); uint16_t numEntries = tmp16;
    getU16(f, tmp16); // total entries
    getU32(f, tmp32); uint32_t cdSize = tmp32;
    getU32(f, tmp32); uint32_t cdOffset = tmp32;
    
    std::cout << "文件数: " << numEntries << std::endl;
    
    // 遍历中央目录
    f.seekg(cdOffset);
    for(int i = 0; i < numEntries; ++i) {
        uint32_t sig; getU32(f, sig);
        if(sig != 0x02014B50) { std::cerr << "目录签名错误" << std::endl; break; }
        
        uint16_t verMade, verNeed, flags, method;
        uint32_t dosDate, crc, compSize32, uncompSize32;
        uint16_t nameLen, extraLen, commentLen, diskStart, intAttr;
        uint32_t extAttr, localOff32;
        
        getU16(f, verMade); getU16(f, verNeed);
        getU16(f, flags); getU16(f, method);
        getU32(f, dosDate); getU32(f, crc);
        getU32(f, compSize32); getU32(f, uncompSize32);
        getU16(f, nameLen); getU16(f, extraLen); getU16(f, commentLen);
        getU16(f, diskStart); getU16(f, intAttr);
        getU32(f, extAttr); getU32(f, localOff32);
        
        std::string name(nameLen, '\0');
        f.read(&name[0], nameLen);
        
        // 处理 ZIP64 扩展字段
        uint64_t uncompSize = uncompSize32;
        uint64_t compSize = compSize32;
        
        uint16_t extraRead = 0;
        while(extraRead < extraLen) {
            uint16_t hid, dsz;
            getU16(f, hid); getU16(f, dsz);
            if(hid == 0x0001) {
                // ZIP64 扩展
                if(uncompSize32 == 0xFFFFFFFF && dsz >= 8) {
                    getU64(f, uncompSize);
                    dsz -= 8; extraRead += 8;
                }
                if(compSize32 == 0xFFFFFFFF && dsz >= 8) {
                    getU64(f, compSize);
                    dsz -= 8; extraRead += 8;
                }
                if(dsz > 0) f.seekg(dsz, std::ios::cur);
            } else {
                f.seekg(dsz, std::ios::cur);
            }
            extraRead += 4 + dsz;
        }
        f.seekg(commentLen, std::ios::cur);
        
        const char* mstr = (method == 0) ? "Store" : (method == 8) ? "Deflate" : "Other";
        std::cout << "  " << name << "  解压:" << uncompSize 
                  << "  压缩:" << compSize << "  方法:" << mstr << std::endl;
    }
    return 0;
}
```

编译运行：
```bash
g++ -std=c++17 -o zip64_list zip64_list.cpp
./zip64_list large_archive.zip
```

</details>

---

## 下一步

[TAR 归档实现](./02-TAR归档实现.md) — 学习 KrKr2 如何解析 POSIX/GNU TAR 格式，包括长文件名支持和八进制数解析。
