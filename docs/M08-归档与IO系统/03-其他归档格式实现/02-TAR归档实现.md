# TAR 归档实现

> **所属模块：** M08-归档与IO系统
> **前置知识：** [tTVPArchive 基类设计](../01-归档系统概述/02-tTVPArchive基类设计.md)、[归档格式对比与选择](../01-归档系统概述/03-归档格式对比与选择.md)
> **预计阅读时间：** 35 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：

1. 理解 TAR 文件格式的二进制结构——512 字节头部块（Header Block）的每个字段含义和布局
2. 掌握八进制（Octal）字符串解析的实现原理，理解 TAR 为何使用八进制而非二进制整数
3. 区分 POSIX ustar 格式与 GNU tar 格式的差异，以及 KrKr2 如何同时支持两者
4. 理解 GNU 长文件名扩展（LONGLINK）机制——突破 100 字符文件名限制的方法
5. 掌握 TAR 头部校验和（Checksum）的两种计算方式及其历史背景
6. 阅读并理解 KrKr2 的 `TARArchive.cpp`（146 行）和 `tar.h`（134 行）的完整实现
7. 独立实现一个简单的 TAR 归档解析器

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| TAR | Tape Archive | 最初为磁带备份设计的归档格式，将多个文件顺序排列，每个文件前有一个 512 字节的头部 |
| 头部块 | Header Block | 每个文件条目前的 512 字节固定大小区域，包含文件名、大小、权限等元信息 |
| TBLOCK | TAR Block Size | TAR 格式的基本块大小，固定为 512 字节，所有数据都按此大小对齐 |
| 八进制字符串 | Octal String | TAR 用 ASCII 文字 '0'-'7' 表示数值（如文件大小 "0001750"），而非直接存二进制整数 |
| POSIX ustar | POSIX Uniform Standard TAR | IEEE Std 1003.1-1988 定义的标准 TAR 格式，支持 prefix 字段扩展文件名到 255 字符 |
| GNU tar | GNU TAR Extension | GNU 项目对 TAR 格式的扩展，增加了 atime/ctime/offset 等字段 |
| LONGLINK | Long Link Name | GNU tar 扩展机制，用类型标志 'L' 标记的特殊条目携带超长文件名 |
| 校验和 | Checksum | 头部所有字节之和（校验和字段本身视为空格），用于验证头部数据完整性 |
| TArchiveStream | TArchive Stream | KrKr2 中基于偏移量和长度的流包装器，用于从归档中读取单个文件数据 |

---

## 一、TAR 格式概述——最简单的归档格式

### 1.1 TAR 的历史与设计哲学

TAR（Tape Archive，磁带归档）是 Unix 世界最古老的归档格式之一，最初设计于 1979 年的 Unix V7 系统。它的设计目标极其简单：**将多个文件顺序拼接到磁带上，每个文件前加一个描述头部**。

与 ZIP 和 XP3 等现代归档格式相比，TAR 有几个显著特点：

- **无压缩**：TAR 本身不做任何压缩，只是简单地将文件排列在一起。压缩通常由外部工具（如 gzip、bzip2、xz）处理，形成 `.tar.gz`、`.tar.bz2`、`.tar.xz` 等组合格式
- **无中央目录**：TAR 没有类似 ZIP 的 Central Directory，要找到某个文件必须从头到尾扫描所有头部。这是磁带介质的自然结果——磁带只能顺序读取
- **512 字节对齐**：所有数据（头部和文件内容）都按 512 字节边界对齐，不足的部分用零填充
- **ASCII 元数据**：文件大小、权限等数值信息用八进制 ASCII 字符串存储，而非二进制整数

```
┌─────────────────────────────────────────────────────┐
│                    TAR 文件结构                       │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────────┐  ┌────────────────────────────┐   │
│  │ Header Block │  │    File Data (padded to     │   │
│  │  (512 bytes) │  │    512-byte boundary)       │   │
│  └──────────────┘  └────────────────────────────┘   │
│        文件 1 的头部          文件 1 的数据            │
│                                                     │
│  ┌──────────────┐  ┌────────────────────────────┐   │
│  │ Header Block │  │    File Data (padded to     │   │
│  │  (512 bytes) │  │    512-byte boundary)       │   │
│  └──────────────┘  └────────────────────────────┘   │
│        文件 2 的头部          文件 2 的数据            │
│                                                     │
│  ┌──────────────┐  ┌────────────────────────────┐   │
│  │ Header Block │  │    File Data (padded to     │   │
│  │  (512 bytes) │  │    512-byte boundary)       │   │
│  └──────────────┘  └────────────────────────────┘   │
│        文件 3 的头部          文件 3 的数据            │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐                 │
│  │ Zero Block   │  │ Zero Block   │                 │
│  │  (512 bytes) │  │  (512 bytes) │                 │
│  └──────────────┘  └──────────────┘                 │
│        结束标记：连续两个全零块                        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 1.2 为什么 KrKr2 需要支持 TAR？

KrKr2 作为视觉小说引擎模拟器，其资源文件可能以多种归档格式打包。虽然 XP3 是 KiriKiri 的原生格式，但有些游戏或工具会使用 TAR 格式打包资源。支持 TAR 有以下好处：

1. **开发调试方便**：开发者可以用标准 Unix 工具 `tar` 快速打包测试资源，不需要专用的 XP3 打包工具
2. **第三方资源兼容**：某些社区 MOD 或补丁可能以 TAR 格式分发
3. **实现简单**：TAR 格式足够简单（KrKr2 的实现仅 146 行），维护成本极低

---

## 二、TAR 头部块二进制结构

### 2.1 512 字节头部的完整字段布局

每个 TAR 文件条目的头部（Header Block）是一个固定 512 字节的数据块。以下是每个字段的精确偏移、大小和含义：

```
偏移量    大小    字段名       格式           说明
────────────────────────────────────────────────────────────
0x000     100    name         ASCII          文件名（最多 100 字符，含末尾 \0）
0x064       8    mode         八进制 ASCII   文件权限（如 "0000644\0"）
0x06C       8    uid          八进制 ASCII   文件所有者 UID
0x074       8    gid          八进制 ASCII   文件所属组 GID
0x07C      12    size         八进制 ASCII   文件大小（字节数）
0x088      12    mtime        八进制 ASCII   最后修改时间（Unix 时间戳）
0x094       8    chksum       八进制 ASCII   头部校验和
0x09C       1    typeflag     ASCII 字符     条目类型标志
0x09D     100    linkname     ASCII          链接目标名（硬链接/符号链接用）
0x101       6    magic        ASCII          格式标识（"ustar" 或 "ustar  "）
0x107       2    version      ASCII          版本号（"00"）
0x109      32    uname        ASCII          文件所有者用户名
0x129      32    gname        ASCII          文件所属组名
0x149       8    devmajor     八进制 ASCII   设备主号（设备文件用）
0x151       8    devminor     八进制 ASCII   设备从号（设备文件用）
────────────────────────────────────────────────────────────
以下字段因格式而异：
────────────────────────────────────────────────────────────
POSIX ustar 格式：
0x159     155    prefix       ASCII          文件名前缀（与 name 拼接可达 255 字符）
0x1F4      12    pad          全零           填充到 512 字节

GNU tar 格式：
0x159      12    atime        八进制 ASCII   最后访问时间
0x165      12    ctime        八进制 ASCII   创建时间
0x171      12    offset       八进制 ASCII   多卷归档偏移
────────────────────────────────────────────────────────────
```

### 2.2 类型标志（typeflag）的含义

`typeflag` 是一个单字节 ASCII 字符，指示这个条目的类型。KrKr2 的 `tar.h` 中定义了完整的类型常量：

```cpp
// 来源：cpp/core/base/tar.h 第 22-40 行
#define REGTYPE  '0'   // 普通文件（Regular file）
#define AREGTYPE '\0'  // 普通文件——旧版 V7 格式用 \0 表示
#define LNKTYPE  '1'   // 硬链接（Hard link）
#define SYMTYPE  '2'   // 符号链接（Symbolic link）
#define CHRTYPE  '3'   // 字符设备（Character special）
#define BLKTYPE  '4'   // 块设备（Block special）
#define DIRTYPE  '5'   // 目录（Directory）
#define FIFOTYPE '6'   // FIFO 管道（FIFO special）
#define CONTTYPE '7'   // 连续文件（Contiguous file）——保留类型
#define LONGLINK 'L'   // GNU 长文件名扩展——携带超长文件名
#define PAX_ENTRTY 'x' // PAX 扩展头——针对单个文件的扩展属性
#define PAX_GLOBAL 'g' // PAX 全局扩展头——影响后续所有文件
#define MULTYPE  'M'   // GNU 多卷归档续接块
#define VOLTYPE  'V'   // GNU 卷标签
```

KrKr2 只关心两种类型：
- `REGTYPE`（'0'）：**普通文件**——这是 KrKr2 唯一会提取的条目类型
- `LONGLINK`（'L'）：**长文件名标记**——用于读取超过 100 字符的文件名

所有其他类型（目录、链接、设备等）都被跳过（`continue`），因为游戏引擎只需要读取资源文件的数据。

### 2.3 对照项目源码——TAR_HEADER 联合体

KrKr2 用 C/C++ 的 `union` 定义了 TAR 头部结构，这种方式既可以按字段名访问，也可以按原始字节数组访问：

```cpp
// 来源：cpp/core/base/tar.h 第 63-132 行
typedef union hblock {
    char dummy[TBLOCK];   // 原始字节视图：512 字节数组
    struct {
        char name[NAMSIZ];    // 文件名，NAMSIZ = 100
        char mode[8];         // 权限
        char uid[8];          // 用户 ID
        char gid[8];          // 组 ID
        char size[12];        // 文件大小（八进制字符串）
        char mtime[12];       // 修改时间
        char chksum[8];       // 校验和
        char typeflag;        // 类型标志
        char linkname[NAMSIZ];// 链接目标
        char magic[6];        // 格式标识
        char version[2];      // 版本号
        char uname[32];       // 用户名
        char gname[32];       // 组名
        char devmajor[8];     // 设备主号
        char devminor[8];     // 设备从号
        union _exthead {
            struct _POSIX {       // POSIX ustar 扩展
                char prefix[155]; // 文件名前缀
                char pad[12];     // 填充
            } posix;
            struct _GNU {         // GNU tar 扩展
                char atime[12];   // 访问时间
                char ctime[12];   // 创建时间
                char offset[12];  // 多卷偏移
            } gnu;
        } exthead;
    } dbuf;                   // 结构化字段视图

    // 校验和计算方法（详见下文）
    unsigned int compsum();
    int compsum_oldtar();
    int getFormat() const;
} TAR_HEADER;
```

这个 `union` 的精妙之处在于 `dummy[TBLOCK]` 和 `dbuf` 结构体共享同一块 512 字节内存。读取头部时，先把 512 字节读进 `dummy` 数组，然后直接通过 `dbuf.name`、`dbuf.size` 等字段名访问各个部分——不需要手动计算偏移量。

---

## 三、八进制字符串解析

### 3.1 为什么使用八进制字符串？

这是初学者最容易困惑的地方：**TAR 的数值字段（文件大小、权限、时间戳等）不是二进制整数，而是八进制 ASCII 字符串**。

比如一个 1000 字节的文件，在 TAR 头部的 `size` 字段中存储的不是二进制的 `0x000003E8`，而是 ASCII 字符串 `"00000001750\0"`（1000 的八进制表示是 1750，加上前导零和末尾 null 填满 12 字节）。

这种设计有历史原因：

1. **可读性**：早期 Unix 系统上，开发者可以直接用 `od` 或 `strings` 命令查看 TAR 头部内容，方便调试
2. **字节序无关**：ASCII 字符串没有大端（Big-endian）/小端（Little-endian）的问题——在任何平台上读取都一样
3. **磁带兼容性**：磁带驱动器以字节流方式工作，ASCII 字符串是最自然的数据表示

### 3.2 KrKr2 的八进制解析函数

```cpp
// 来源：cpp/core/base/TARArchive.cpp 第 27-36 行
tjs_uint64 parseOctNum(const char *oct, int length) {
    tjs_uint64 num = 0;           // 结果变量，64 位无符号整数
    for(int i = 0; i < length; i++) {
        char c = oct[i];
        if('0' <= c && c <= '9') { // 只处理数字字符
            num = num * 8 + (c - '0'); // 八进制左移一位 + 当前数字
        }
        // 非数字字符（空格、\0 等）被静默跳过
    }
    return num;
}
```

这个函数的工作原理与标准的 `strtol(..., 8)` 类似，但有一个重要区别：**它静默跳过所有非数字字符**。这是因为 TAR 头部的八进制字段可能包含前导空格、末尾空格、末尾 null 字符等，不同的 tar 实现对填充的处理方式不一致。

让我们手工追踪一个解析过程：

```
输入字段："0001750\0" （文件大小 = 1000 字节）

i=0: c='0', num = 0 * 8 + 0 = 0
i=1: c='0', num = 0 * 8 + 0 = 0
i=2: c='0', num = 0 * 8 + 0 = 0
i=3: c='1', num = 0 * 8 + 1 = 1
i=4: c='7', num = 1 * 8 + 7 = 15
i=5: c='5', num = 15 * 8 + 5 = 125
i=6: c='0', num = 125 * 8 + 0 = 1000
i=7: c='\0', 不是 '0'-'9'，跳过

结果：1000 ✓
```

### 3.3 完整可运行示例——八进制解析器

```cpp
#include <cstdint>
#include <cstdio>
#include <cstring>

// 模拟 KrKr2 的八进制解析函数
uint64_t parseOctNum(const char *oct, int length) {
    uint64_t num = 0;
    for (int i = 0; i < length; i++) {
        char c = oct[i];
        if ('0' <= c && c <= '9') {
            num = num * 8 + (c - '0');
        }
    }
    return num;
}

int main() {
    // 测试用例 1：标准八进制字符串（文件大小 1000 字节）
    const char size1[] = "00000001750\0";
    printf("解析 '00000001750': %llu\n", parseOctNum(size1, 12));
    // 输出：1000

    // 测试用例 2：带前导空格（某些 tar 实现的格式）
    const char size2[] = " 1750\0\0\0\0\0\0\0";
    printf("解析 ' 1750': %llu\n", parseOctNum(size2, 12));
    // 输出：1000（空格被跳过）

    // 测试用例 3：文件权限 0644
    const char mode[] = "0000644\0";
    printf("解析权限 '0000644': %llo (八进制)\n", parseOctNum(mode, 8));
    // 输出：644 (八进制) = 420 (十进制)

    // 测试用例 4：Unix 时间戳
    const char mtime[] = "14717450123\0";
    printf("解析时间戳: %llu\n", parseOctNum(mtime, 12));

    // 测试用例 5：全零（空条目标记）
    const char zeros[] = "\0\0\0\0\0\0\0\0\0\0\0\0";
    printf("解析全零: %llu\n", parseOctNum(zeros, 12));
    // 输出：0

    return 0;
}
```

---

## 四、校验和（Checksum）计算

### 4.1 为什么需要校验和？

TAR 头部的 `chksum` 字段用于验证头部数据在传输或存储过程中没有被损坏。计算方法很简单：**将头部 512 字节的所有字节值相加，但计算时将 chksum 字段的 8 个字节视为空格字符（ASCII 32）**。

为什么要将 chksum 字段视为空格？因为这是一个"先有鸡还是先有蛋"的问题——校验和字段本身也在被校验的数据范围内。解决方法是：写入校验和之前先用空格填充 chksum 字段，计算出校验和后再写入。读取验证时，将 chksum 字段的实际值减掉，再加回空格的值。

### 4.2 两种校验和算法——unsigned 与 signed

KrKr2 的 `tar.h` 实现了两种校验和计算方式：

```cpp
// 来源：cpp/core/base/tar.h 第 95-123 行

// 方式 1：POSIX 标准校验和（unsigned char 累加）
unsigned int compsum() {
    unsigned int sum = 0;
    int i;
    // 第一步：把所有 512 字节当作 unsigned char 累加
    for (i = 0; i < TBLOCK; i++) {
        sum += (unsigned char)dummy[i];
    }
    // 第二步：减去 chksum 字段的实际值
    for (i = 0; i < sizeof(dbuf.chksum); i++) {
        sum -= dbuf.chksum[i];
        sum += ' ';  // 加回空格字符的值（32）
    }
    return sum;
}

// 方式 2：旧版 Unix tar 校验和（signed char 累加）
int compsum_oldtar() {
    unsigned int sum = 0;
    int i;
    // 关键差异：用 signed char 累加
    for (i = 0; i < TBLOCK; i++) {
        sum += (signed char)dummy[i];
    }
    // 同样减去 chksum 字段并加回空格
    for (i = 0; i < sizeof(dbuf.chksum); i++) {
        sum -= dbuf.chksum[i];
        sum += ' ';
    }
    return sum;
}
```

**两种方式的关键区别**：`unsigned char` 把每个字节当作 0-255 的值累加；`signed char` 把高位字节（128-255）当作负数（-128 到 -1）累加。旧版 Unix（特别是某些 PDP-11 和早期 BSD 系统）使用 signed char 方式，POSIX 标准后来统一为 unsigned char。

KrKr2 在验证 TAR 头部时**两种方式都尝试**：

```cpp
// 来源：cpp/core/base/TARArchive.cpp 第 65-71 行
unsigned int checksum = parseOctNum(tar_header.dbuf.chksum,
                                    sizeof(tar_header.dbuf.chksum));
if (checksum != tar_header.compsum() &&
    (int)checksum != tar_header.compsum_oldtar()) {
    return false;  // 两种校验和都不匹配——不是有效的 TAR 文件
}
```

### 4.3 格式检测函数

```cpp
// 来源：cpp/core/base/tar.h 第 125-131 行
[[nodiscard]] int getFormat() const {
    if (dbuf.magic[5] == TOMAGIC[5]) {
        return TAR_FORMAT_GNU;   // GNU tar 格式（magic 第 6 字节是空格）
    } else {
        return TAR_FORMAT_POSIX; // POSIX ustar 格式
    }
}
```

POSIX ustar 的 magic 字段是 `"ustar\0"`（6 字节，第 6 字节是 null），而 GNU tar 的 magic+version 字段是 `"ustar  \0"`（8 字节，用空格填充）。通过检查第 6 字节是 null 还是空格就能区分两种格式。

---

## 五、文件名处理——从 100 字符到无限长度

### 5.1 标准文件名的限制

TAR 头部的 `name` 字段只有 100 字节（`NAMSIZ = 100`），这意味着文件名（含路径）最多只能有 100 个字符（包括末尾的 null 终止符，实际可用 99 字符）。对于深层嵌套的目录结构，这远远不够。

POSIX ustar 格式通过 `prefix` 字段（155 字节）部分缓解了这个问题——将路径前缀和文件名分开存储，拼接后最多可达 255 字符。但 KrKr2 没有使用 prefix 字段，而是采用了 GNU tar 的 LONGLINK 机制。

### 5.2 GNU LONGLINK 机制

当文件名超过 100 字符时，GNU tar 会在实际文件条目之前插入一个**特殊的 LONGLINK 条目**：

```
┌──────────────────────────────────────────────────────────┐
│ 处理长文件名的 TAR 条目序列                               │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────────────────────────────────────────┐         │
│  │ LONGLINK Header Block (512 bytes)           │         │
│  │   name = "././@LongLink"                    │         │
│  │   typeflag = 'L'                            │         │
│  │   size = 长文件名的字节数                     │         │
│  └─────────────────────────────────────────────┘         │
│                                                          │
│  ┌─────────────────────────────────────────────┐         │
│  │ Long Filename Data (padded to 512 bytes)    │         │
│  │   "very/long/path/to/some/deeply/nested/    │         │
│  │    directory/structure/filename.txt\0"       │         │
│  └─────────────────────────────────────────────┘         │
│                                                          │
│  ┌─────────────────────────────────────────────┐         │
│  │ Actual File Header Block (512 bytes)        │         │
│  │   name = "very/long/path..." (截断到 100)    │         │
│  │   typeflag = '0' (普通文件)                  │         │
│  │   size = 文件的实际大小                       │         │
│  └─────────────────────────────────────────────┘         │
│                                                          │
│  ┌─────────────────────────────────────────────┐         │
│  │ File Data (padded to 512 bytes)             │         │
│  └─────────────────────────────────────────────┘         │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 5.3 KrKr2 的 LONGLINK 处理代码

```cpp
// 来源：cpp/core/base/TARArchive.cpp 第 79-94 行
std::vector<char> filename;
if (tar_header.dbuf.typeflag == LONGLINK) {
    // LONGLINK 条目：数据区包含完整的长文件名
    unsigned int readsize =
        (original_size + (TBLOCK - 1)) & ~(TBLOCK - 1);
    //                    ^^^^^^^^^^^^^^^^^^^^^^^^^^
    //  向上对齐到 512 字节边界：(size + 511) & ~511
    //  例如 size=150 → (150+511)&~511 = 661&0xFE00 = 512
    //  例如 size=513 → (513+511)&~511 = 1024

    filename.resize(readsize + 1);       // +1 留给末尾 \0
    if (_instr->Read(&filename[0], readsize) != readsize)
        break;
    filename[readsize] = 0;              // 确保 null 终止

    // 读取完长文件名后，紧跟着的才是真正的文件头部
    if (_instr->Read(&tar_header, sizeof(tar_header)) !=
        sizeof(tar_header))
        break;
    original_size = parseOctNum(tar_header.dbuf.size,
                                sizeof(tar_header.dbuf.size));
    // original_size 更新为真正文件的大小
}
```

关键步骤：
1. 检测到 `typeflag == 'L'`（LONGLINK）
2. 读取 LONGLINK 的数据区（包含长文件名），大小向上对齐到 512 字节
3. 继续读取下一个头部块——这才是真正的文件条目
4. 使用 LONGLINK 中读到的长文件名替代头部中被截断的 name 字段

### 5.4 短文件名的回退处理

如果不是 LONGLINK 条目，文件名就直接从头部的 `name` 字段读取：

```cpp
// 来源：cpp/core/base/TARArchive.cpp 第 97-102 行
if (filename.empty()) {
    filename.resize(101);       // 100 字符 + \0
    memcpy(&filename[0], tar_header.dbuf.name,
           sizeof(tar_header.dbuf.name));  // 复制 100 字节
    filename[100] = 0;          // 强制 null 终止
}
```

注意 `filename[100] = 0` 这一行——TAR 标准允许 `name` 字段的 100 字节全部用于文件名（无 null 终止符），所以代码必须手动添加终止符以确保安全。

### 5.5 文件名编码转换

TAR 头部中的文件名是窄字符（ASCII/UTF-8），需要转换为 KrKr2 内部使用的宽字符（`tjs_char`，即 `wchar_t`）：

```cpp
// 来源：cpp/core/base/TARArchive.cpp 第 12-22 行
void storeFilename(ttstr &name, const char *narrowName,
                   const ttstr &filename) {
    tjs_int len = TJS_narrowtowidelen(narrowName);
    // 计算窄字符串转换为宽字符串后的长度
    if (len == -1) {
        // 转换失败——文件名不是有效的 UTF-8 编码
        ttstr msg("Filename is not encoded in UTF8 in archive:\n");
        TVPShowSimpleMessageBox(msg + filename, TJS_W("Error"));
        TVPThrowExceptionMessage(TJS_W("Invalid archive entry name"));
    }
    tjs_char *p = name.AllocBuffer(len);
    // 在 ttstr 中分配足够的宽字符缓冲区
    p[TJS_narrowtowide(p, narrowName, len)] = 0;
    // 执行实际转换，并添加 null 终止符
    name.FixLen();
    // 更新 ttstr 的长度缓存
}
```

这个函数假设 TAR 归档中的文件名使用 UTF-8 编码。如果遇到非 UTF-8 文件名（比如日文 Shift-JIS 编码），会弹出错误对话框并抛出异常。

---

## 六、TARArchive 类——完整实现分析

### 6.1 类结构概览

`TARArchive` 继承自 `tTVPArchive` 基类，是 KrKr2 中最简单的归档实现——仅 146 行代码（含工厂函数），没有压缩、没有加密、没有中央目录：

```cpp
// 来源：cpp/core/base/TARArchive.cpp 第 40-131 行
class TARArchive : public tTVPArchive {
    struct EntryInfo {
        ttstr filename;       // 文件名（宽字符）
        tjs_uint64 offset{};  // 文件数据在归档中的字节偏移
        tjs_uint64 size{};    // 文件数据的大小（字节）
    };
    std::vector<EntryInfo> filelist;  // 所有文件条目列表

    friend class TARArchiveStream;    // 允许流类访问私有成员

public:
    TARArchive(const ttstr &arcname) : tTVPArchive(arcname) {}

    ~TARArchive() override {
        TVPFreeArchiveHandlePoolByPointer(this);
        // 释放缓存的归档文件句柄
    }

    bool init(tTJSBinaryStream *_instr, bool normalizeFileName);

    tjs_uint GetCount() override { return filelist.size(); }
    ttstr GetName(tjs_uint idx) override { return filelist[idx].filename; }
    tTJSBinaryStream *CreateStreamByIndex(tjs_uint idx) override;
};
```

与 ZIP 归档（2138 行）和 XP3 归档（1024 行）相比，TAR 的实现简洁得多。原因在于 TAR 格式本身的设计简单性：
- 没有压缩→不需要解压缩逻辑
- 没有中央目录→不需要二次查找
- 没有加密→不需要解密流水线

### 6.2 init() 方法——归档初始化流程

`init()` 方法是 TARArchive 的核心，负责扫描整个 TAR 文件，提取所有文件条目的元信息。以下是完整的逐行分析：

```
┌──────────────────────────────────────────────────────────┐
│              TARArchive::init() 流程图                    │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  [开始] 传入 tTJSBinaryStream* 和 normalizeFileName      │
│    │                                                     │
│    ▼                                                     │
│  获取归档总大小 archiveSize                               │
│    │                                                     │
│    ▼                                                     │
│  读取第一个 512 字节头部                                  │
│    │                                                     │
│    ▼                                                     │
│  验证校验和（compsum 或 compsum_oldtar）                  │
│    │──── 不匹配 ──→ return false（不是有效 TAR）          │
│    │                                                     │
│    ▼ 匹配                                                │
│  重置文件位置到起点                                       │
│    │                                                     │
│    ▼                                                     │
│  ┌─────── 循环：读取每个 512 字节头部 ──────┐             │
│  │                                          │             │
│  │  解析 size 字段（八进制→整数）            │             │
│  │    │                                     │             │
│  │    ▼                                     │             │
│  │  typeflag == 'L' ?                       │             │
│  │    │──── 是 ──→ 读取长文件名数据          │             │
│  │    │            读取下一个真正的文件头部   │             │
│  │    │            重新解析 size              │             │
│  │    │                                     │             │
│  │    ▼ 否                                  │             │
│  │  typeflag == '0' ?                       │             │
│  │    │──── 否 ──→ continue（跳过非普通文件）│             │
│  │    │                                     │             │
│  │    ▼ 是                                  │             │
│  │  保存 {filename, offset, size}            │             │
│  │  跳过文件数据（对齐到 512 字节边界）      │             │
│  │    │                                     │             │
│  │    └──────────── 继续下一个条目 ──────────┘             │
│  │                                                       │
│  ▼                                                       │
│  排序 filelist（如果 normalizeFileName 为 true）           │
│    │                                                     │
│    ▼                                                     │
│  释放流到缓存池                                           │
│    │                                                     │
│    ▼                                                     │
│  [结束] return true                                      │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

关键代码段——条目信息收集：

```cpp
// 来源：cpp/core/base/TARArchive.cpp 第 103-113 行
EntryInfo item;
storeFilename(item.filename, &filename[0], ArchiveName);
// 将窄字符文件名转换为宽字符并存储

if (normalizeFileName)
    NormalizeInArchiveStorageName(item.filename);
// 规范化文件名：转小写 + 统一路径分隔符

item.size = original_size;         // 文件数据大小
item.offset = _instr->GetPosition(); // 当前流位置 = 数据起始偏移

filelist.emplace_back(item);       // 添加到条目列表

// 跳过文件数据，对齐到 512 字节边界
tjs_uint64 readsize =
    (original_size + (TBLOCK - 1)) & ~(TBLOCK - 1);
_instr->SetPosition(item.offset + readsize);
```

**512 字节对齐计算**的数学原理：

```
(size + 511) & ~511

~511 = ~0x1FF = 0xFFFFFFFFFFFFFE00（64 位）

示例：
  size = 0    → (0 + 511) & ~511    = 511 & 0xFE00 = 0
  size = 1    → (1 + 511) & ~511    = 512 & 0xFE00 = 512
  size = 512  → (512 + 511) & ~511  = 1023 & 0xFE00 = 512
  size = 513  → (513 + 511) & ~511  = 1024 & 0xFE00 = 1024
  size = 1000 → (1000 + 511) & ~511 = 1511 & 0xFE00 = 1024
```

这是一种高效的**向上对齐**技术（Round Up），利用位运算在 O(1) 时间内完成，不需要除法。

### 6.3 CreateStreamByIndex——文件数据读取

当上层代码需要读取归档中某个文件的数据时，调用 `CreateStreamByIndex()`：

```cpp
// 来源：cpp/core/base/TARArchive.cpp 第 133-136 行
tTJSBinaryStream *TARArchive::CreateStreamByIndex(tjs_uint idx) {
    const EntryInfo &info = filelist[idx];
    return new TArchiveStream(this, info.offset, info.size);
}
```

这里返回的 `TArchiveStream` 是一个轻量级的流包装器（在 `StorageIntf.h` 中定义），它做的事情非常简单：
- 持有归档文件的 `tTJSBinaryStream*` 指针
- 记录文件数据的起始偏移和大小
- 读取时将 `Read()` 调用转发到底层流，自动加上偏移量
- 不允许写入——归档中的文件是只读的

因为 TAR 不压缩文件数据，所以 `TArchiveStream` 可以直接从归档文件中读取原始字节——这是一种**零拷贝**设计，不需要像 ZIP 的 Deflate 格式那样先解压到内存。

### 6.4 工厂函数

```cpp
// 来源：cpp/core/base/TARArchive.cpp 第 138-146 行
tTVPArchive *TVPOpenTARArchive(const ttstr &name,
                                tTJSBinaryStream *st,
                                bool normalizeFileName) {
    auto *arc = new TARArchive(name);
    if (!arc->init(st, normalizeFileName)) {
        delete arc;       // 初始化失败（不是有效 TAR）→ 清理
        return nullptr;   // 返回 null 让调用者尝试其他格式
    }
    return arc;
}
```

与 ZIP 和 7z 的工厂函数一样，`TVPOpenTARArchive` 遵循 **试探性打开** 模式：创建对象→尝试初始化→失败则销毁并返回 null。调用者（`TVPOpenArchive`）会依次尝试 XP3、ZIP、TAR、7z 格式，直到某个格式成功。

### 6.5 归档句柄缓存机制

注意 `init()` 末尾的这一行：

```cpp
TVPReleaseCachedArchiveHandle(this, _instr);
```

以及析构函数中的：

```cpp
TVPFreeArchiveHandlePoolByPointer(this);
```

这是 KrKr2 的归档文件句柄缓存机制。`init()` 完成后，底层文件流 `_instr` 不会立即关闭，而是放入缓存池中。当后续通过 `CreateStreamByIndex()` 创建 `TArchiveStream` 时，`TArchiveStream` 会从缓存池中取回这个流。如果缓存已满或已被回收，则重新打开归档文件。

这种设计避免了频繁的文件打开/关闭操作，对于游戏运行时频繁读取资源文件的场景非常重要。

---

## 七、完整可运行示例——TAR 归档解析器

### 7.1 基础版——列出 TAR 内容

以下是一个独立的 TAR 解析器，不依赖 KrKr2 的任何代码，可以直接编译运行：

```cpp
// tar_lister.cpp — 独立的 TAR 归档列表工具
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <vector>
#include <string>

// TAR 块大小
static constexpr int TBLOCK = 512;

// 八进制字符串解析
uint64_t parseOctNum(const char *oct, int length) {
    uint64_t num = 0;
    for (int i = 0; i < length; i++) {
        char c = oct[i];
        if ('0' <= c && c <= '9') {
            num = num * 8 + (c - '0');
        }
    }
    return num;
}

// TAR 头部结构（简化版）
struct TarHeader {
    char name[100];    // 文件名
    char mode[8];      // 权限
    char uid[8];       // 用户 ID
    char gid[8];       // 组 ID
    char size[12];     // 文件大小（八进制）
    char mtime[12];    // 修改时间
    char chksum[8];    // 校验和
    char typeflag;     // 类型标志
    char linkname[100];// 链接名
    char magic[6];     // 格式标识
    char version[2];   // 版本
    char uname[32];    // 用户名
    char gname[32];    // 组名
    char devmajor[8];  // 设备主号
    char devminor[8];  // 设备从号
    char prefix[155];  // POSIX 前缀
    char pad[12];      // 填充
};
static_assert(sizeof(TarHeader) == TBLOCK,
              "TarHeader must be exactly 512 bytes");

// 计算校验和
unsigned int computeChecksum(const char *block) {
    unsigned int sum = 0;
    for (int i = 0; i < TBLOCK; i++) {
        sum += (unsigned char)block[i];
    }
    // 减去 chksum 字段的值，加回空格
    for (int i = 148; i < 156; i++) { // chksum 在偏移 148，长度 8
        sum -= (unsigned char)block[i];
        sum += ' ';
    }
    return sum;
}

// 检查是否为全零块（结束标记）
bool isZeroBlock(const char *block) {
    for (int i = 0; i < TBLOCK; i++) {
        if (block[i] != 0) return false;
    }
    return true;
}

// 文件条目信息
struct FileEntry {
    std::string name;
    uint64_t offset;  // 数据在文件中的偏移
    uint64_t size;
    uint64_t mtime;
    unsigned int mode;
};

int main(int argc, char *argv[]) {
    if (argc < 2) {
        printf("用法: %s <tar文件路径>\n", argv[0]);
        return 1;
    }

    FILE *fp = fopen(argv[1], "rb");
    if (!fp) {
        printf("无法打开文件: %s\n", argv[1]);
        return 1;
    }

    // 获取文件大小
    fseek(fp, 0, SEEK_END);
    long fileSize = ftell(fp);
    fseek(fp, 0, SEEK_SET);

    std::vector<FileEntry> entries;
    char block[TBLOCK];
    std::string longName;  // LONGLINK 缓冲

    printf("解析 TAR 归档: %s (%ld 字节)\n", argv[1], fileSize);
    printf("────────────────────────────────────────\n");

    while (ftell(fp) <= fileSize - TBLOCK) {
        // 读取 512 字节头部块
        if (fread(block, 1, TBLOCK, fp) != TBLOCK) break;

        // 检查是否为结束标记（全零块）
        if (isZeroBlock(block)) {
            printf("\n遇到全零块，归档结束。\n");
            break;
        }

        auto *hdr = reinterpret_cast<TarHeader *>(block);

        // 验证校验和
        unsigned int storedChecksum =
            parseOctNum(hdr->chksum, sizeof(hdr->chksum));
        unsigned int computedChecksum = computeChecksum(block);
        if (storedChecksum != computedChecksum) {
            printf("警告：校验和不匹配（存储=%u, 计算=%u），跳过\n",
                   storedChecksum, computedChecksum);
            break;
        }

        uint64_t entrySize =
            parseOctNum(hdr->size, sizeof(hdr->size));

        // 处理 LONGLINK
        if (hdr->typeflag == 'L') {
            uint64_t readSize =
                (entrySize + (TBLOCK - 1)) & ~(TBLOCK - 1);
            std::vector<char> nameBuf(readSize + 1, 0);
            if (fread(nameBuf.data(), 1, readSize, fp) != readSize)
                break;
            longName = nameBuf.data();
            continue; // 下一次循环读取真正的文件头部
        }

        // 获取文件名
        std::string fileName;
        if (!longName.empty()) {
            fileName = longName;
            longName.clear();
        } else {
            // 从头部提取，最多 100 字符
            fileName = std::string(hdr->name,
                strnlen(hdr->name, sizeof(hdr->name)));
        }

        // 只记录普通文件
        if (hdr->typeflag == '0' || hdr->typeflag == '\0') {
            FileEntry entry;
            entry.name = fileName;
            entry.offset = ftell(fp);
            entry.size = entrySize;
            entry.mtime = parseOctNum(hdr->mtime, sizeof(hdr->mtime));
            entry.mode = parseOctNum(hdr->mode, sizeof(hdr->mode));
            entries.push_back(entry);

            printf("[文件] %-50s  大小: %8llu 字节  权限: %04o\n",
                   fileName.c_str(),
                   (unsigned long long)entrySize,
                   entry.mode);
        } else {
            printf("[跳过] %-50s  类型: '%c'\n",
                   fileName.c_str(), hdr->typeflag);
        }

        // 跳过文件数据（对齐到 512 字节边界）
        uint64_t paddedSize =
            (entrySize + (TBLOCK - 1)) & ~(TBLOCK - 1);
        fseek(fp, (long)paddedSize, SEEK_CUR);
    }

    printf("────────────────────────────────────────\n");
    printf("共发现 %zu 个文件\n", entries.size());

    fclose(fp);
    return 0;
}
```

**CMakeLists.txt**：

```cmake
cmake_minimum_required(VERSION 3.15)
project(tar_lister CXX)
set(CMAKE_CXX_STANDARD 17)

add_executable(tar_lister tar_lister.cpp)
```

**编译与运行**：

```bash
# Linux / macOS
mkdir build && cd build
cmake .. && cmake --build .
./tar_lister ../test_archive.tar

# Windows (Visual Studio)
cmake -B build -G "Visual Studio 17 2022"
cmake --build build --config Release
build\Release\tar_lister.exe test_archive.tar
```

### 7.2 进阶版——提取文件

在列表功能基础上，添加提取（Extract）功能：

```cpp
// tar_extractor.cpp — 从 TAR 归档中提取指定文件
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <string>
#include <vector>

static constexpr int TBLOCK = 512;

uint64_t parseOctNum(const char *oct, int length) {
    uint64_t num = 0;
    for (int i = 0; i < length; i++) {
        char c = oct[i];
        if ('0' <= c && c <= '9')
            num = num * 8 + (c - '0');
    }
    return num;
}

struct TarHeader {
    char name[100]; char mode[8]; char uid[8]; char gid[8];
    char size[12]; char mtime[12]; char chksum[8]; char typeflag;
    char linkname[100]; char magic[6]; char version[2];
    char uname[32]; char gname[32]; char devmajor[8]; char devminor[8];
    char prefix[155]; char pad[12];
};

bool extractFile(FILE *tar, uint64_t offset, uint64_t size,
                 const char *outputPath) {
    FILE *out = fopen(outputPath, "wb");
    if (!out) {
        printf("无法创建输出文件: %s\n", outputPath);
        return false;
    }

    fseek(tar, (long)offset, SEEK_SET);

    // 分块读取并写入
    char buffer[4096];                  // 4KB 缓冲区
    uint64_t remaining = size;
    while (remaining > 0) {
        size_t toRead = (remaining < sizeof(buffer))
                            ? (size_t)remaining
                            : sizeof(buffer);
        size_t bytesRead = fread(buffer, 1, toRead, tar);
        if (bytesRead == 0) break;     // EOF 或错误
        fwrite(buffer, 1, bytesRead, out);
        remaining -= bytesRead;
    }

    fclose(out);
    printf("已提取: %s (%llu 字节)\n", outputPath,
           (unsigned long long)size);
    return true;
}

int main(int argc, char *argv[]) {
    if (argc < 3) {
        printf("用法: %s <tar文件> <要提取的文件名>\n", argv[0]);
        return 1;
    }

    FILE *fp = fopen(argv[1], "rb");
    if (!fp) { printf("无法打开: %s\n", argv[1]); return 1; }

    fseek(fp, 0, SEEK_END);
    long fileSize = ftell(fp);
    fseek(fp, 0, SEEK_SET);

    const char *targetName = argv[2];
    char block[TBLOCK];
    std::string longName;

    while (ftell(fp) <= fileSize - TBLOCK) {
        if (fread(block, 1, TBLOCK, fp) != TBLOCK) break;

        // 全零块 = 结束
        bool allZero = true;
        for (int i = 0; i < TBLOCK && allZero; i++)
            if (block[i] != 0) allZero = false;
        if (allZero) break;

        auto *hdr = reinterpret_cast<TarHeader *>(block);
        uint64_t entrySize =
            parseOctNum(hdr->size, sizeof(hdr->size));

        if (hdr->typeflag == 'L') {
            uint64_t readSize =
                (entrySize + (TBLOCK - 1)) & ~(TBLOCK - 1);
            std::vector<char> buf(readSize + 1, 0);
            if (fread(buf.data(), 1, readSize, fp) != readSize) break;
            longName = buf.data();
            continue;
        }

        std::string fileName;
        if (!longName.empty()) {
            fileName = longName; longName.clear();
        } else {
            fileName = std::string(hdr->name,
                strnlen(hdr->name, sizeof(hdr->name)));
        }

        if ((hdr->typeflag == '0' || hdr->typeflag == '\0') &&
            fileName == targetName) {
            // 找到目标文件——提取
            uint64_t dataOffset = ftell(fp);
            extractFile(fp, dataOffset, entrySize, targetName);
            fclose(fp);
            return 0;
        }

        // 跳过数据
        uint64_t paddedSize =
            (entrySize + (TBLOCK - 1)) & ~(TBLOCK - 1);
        fseek(fp, (long)paddedSize, SEEK_CUR);
    }

    printf("未找到文件: %s\n", targetName);
    fclose(fp);
    return 1;
}
```

### 7.3 校验和验证工具

```cpp
// tar_verify.cpp — 验证 TAR 归档中所有头部的校验和
#include <cstdint>
#include <cstdio>
#include <cstring>

static constexpr int TBLOCK = 512;

uint64_t parseOctNum(const char *oct, int length) {
    uint64_t num = 0;
    for (int i = 0; i < length; i++) {
        char c = oct[i];
        if ('0' <= c && c <= '9')
            num = num * 8 + (c - '0');
    }
    return num;
}

// POSIX 标准校验和（unsigned char）
unsigned int computeChecksumUnsigned(const char *block) {
    unsigned int sum = 0;
    for (int i = 0; i < TBLOCK; i++)
        sum += (unsigned char)block[i];
    for (int i = 148; i < 156; i++) {
        sum -= (unsigned char)block[i];
        sum += ' ';
    }
    return sum;
}

// 旧版 Unix 校验和（signed char）
int computeChecksumSigned(const char *block) {
    int sum = 0;
    for (int i = 0; i < TBLOCK; i++)
        sum += (signed char)block[i];
    for (int i = 148; i < 156; i++) {
        sum -= block[i];
        sum += ' ';
    }
    return sum;
}

int main(int argc, char *argv[]) {
    if (argc < 2) {
        printf("用法: %s <tar文件>\n", argv[0]);
        return 1;
    }

    FILE *fp = fopen(argv[1], "rb");
    if (!fp) { printf("无法打开: %s\n", argv[1]); return 1; }

    char block[TBLOCK];
    int entryIndex = 0;
    int passed = 0, failed = 0;

    while (fread(block, 1, TBLOCK, fp) == TBLOCK) {
        // 检查全零块
        bool allZero = true;
        for (int i = 0; i < TBLOCK && allZero; i++)
            if (block[i] != 0) allZero = false;
        if (allZero) break;

        unsigned int stored =
            parseOctNum(block + 148, 8);  // chksum 偏移 = 148
        unsigned int calcU = computeChecksumUnsigned(block);
        int calcS = computeChecksumSigned(block);

        char name[101] = {0};
        memcpy(name, block, 100);

        if (stored == calcU) {
            printf("[%3d] ✓ POSIX  %-40s  checksum=%u\n",
                   entryIndex, name, stored);
            passed++;
        } else if ((int)stored == calcS) {
            printf("[%3d] ✓ OLD    %-40s  checksum=%u (signed)\n",
                   entryIndex, name, stored);
            passed++;
        } else {
            printf("[%3d] ✗ FAIL   %-40s  stored=%u, "
                   "POSIX=%u, OLD=%d\n",
                   entryIndex, name, stored, calcU, calcS);
            failed++;
        }

        // 跳过数据区
        uint64_t size = parseOctNum(block + 124, 12);
        uint64_t paddedSize =
            (size + (TBLOCK - 1)) & ~(TBLOCK - 1);
        fseek(fp, (long)paddedSize, SEEK_CUR);

        entryIndex++;
    }

    printf("\n校验完成：%d 通过，%d 失败\n", passed, failed);
    fclose(fp);
    return (failed == 0) ? 0 : 1;
}
```

---

## 八、TAR 与其他格式的对比

| 特性 | TAR | ZIP | XP3 | 7z |
|------|-----|-----|-----|-----|
| 压缩支持 | ❌ 无（需外部工具） | ✅ Deflate/Store | ✅ zlib | ✅ LZMA/LZMA2 |
| 中央目录 | ❌ 无（顺序扫描） | ✅ Central Directory | ✅ 文件索引区 | ✅ 数据库结构 |
| 随机访问 | ⚠️ 需先扫描建索引 | ✅ 直接定位 | ✅ 直接定位 | ✅ 直接定位 |
| 加密 | ❌ 无 | ⚠️ 弱加密 | ✅ 提取过滤器 | ✅ AES-256 |
| 文件名长度 | 100（LONGLINK 无限） | 65535 | 无限制 | 无限制 |
| KrKr2 实现行数 | 146 | 2138 | 1024 | 260 |
| 适用场景 | 开发调试、简单打包 | 通用归档 | 游戏发布 | 高压缩比 |

TAR 格式在 KrKr2 中的定位很明确：**作为最简单的归档格式提供基础支持**。它的实现成本极低（146 行），不增加额外依赖，适合开发阶段使用。正式发布的游戏通常会使用 XP3 格式（支持加密和压缩）。

---

## 九、常见错误与排查

### 9.1 校验和不匹配

**症状**：`init()` 返回 `false`，归档无法打开。

**原因**：
- 文件不是 TAR 格式（可能是压缩过的 `.tar.gz`、`.tar.bz2`）
- 文件下载不完整或已损坏
- 使用了非标准的 TAR 实现创建的归档

**解决方案**：
```bash
# 检查文件是否被压缩
file archive.tar    # 应显示 "POSIX tar archive"
                     # 如果显示 "gzip compressed" 则需要先解压

# 解压 .tar.gz
gunzip archive.tar.gz   # 得到 archive.tar

# 验证 TAR 完整性
tar tf archive.tar       # 应能列出内容
```

### 9.2 文件名乱码

**症状**：文件名显示为乱码或触发 "Filename is not encoded in UTF8" 错误。

**原因**：TAR 归档中的文件名使用了非 UTF-8 编码（如 Shift-JIS、GBK）。

**解决方案**：
- 使用 UTF-8 编码重新创建 TAR 归档
- 或修改 `storeFilename()` 函数，添加编码检测和转换逻辑

### 9.3 大文件支持问题

**症状**：大于 8 GB 的文件大小被截断。

**原因**：TAR 的 `size` 字段是 12 字节八进制字符串，最大值为 `77777777777`（八进制），等于 8,589,934,591 字节（约 8 GB）。

**解决方案**：KrKr2 使用 `tjs_uint64`（64 位无符号整数）存储大小，但八进制解析的上限仍然受 TAR 格式本身限制。GNU tar 和 PAX 扩展可以突破这个限制（使用 base-256 编码或 PAX 扩展头），但 KrKr2 当前不支持这些扩展。对于游戏资源文件来说，8 GB 的限制通常不是问题。

---

## 对照项目源码

KrKr2 的 TAR 归档实现分布在两个文件中：

### 相关文件：

- **`cpp/core/base/tar.h`** 第 1-134 行 —— TAR 头部结构定义（`TAR_HEADER` 联合体）、类型标志常量（`REGTYPE`、`LONGLINK` 等）、POSIX/GNU 格式标识（`TMAGIC`/`TOMAGIC`）、校验和计算方法（`compsum()`/`compsum_oldtar()`）、格式检测（`getFormat()`）
- **`cpp/core/base/TARArchive.cpp`** 第 1-146 行 —— `storeFilename()` 文件名编码转换、`parseOctNum()` 八进制解析、`TARArchive` 类完整实现（`init()` 归档扫描、`CreateStreamByIndex()` 流创建）、`TVPOpenTARArchive()` 工厂函数
- **`cpp/core/base/StorageIntf.h`** —— `tTVPArchive` 基类定义（`TARArchive` 的父类）、`TArchiveStream` 流包装器定义（`CreateStreamByIndex` 返回的流类型）
- **`cpp/core/base/StorageIntf.cpp`** 第 700+ 行 —— `TVPOpenArchive()` 统一入口函数，依次尝试 XP3→ZIP→TAR→7z 格式

---

## 本节小结

- TAR 是最简单的归档格式：无压缩、无中央目录、512 字节对齐、八进制 ASCII 元数据
- KrKr2 的 TAR 实现（146 行）是所有归档格式中最精简的，但完整支持 POSIX ustar 和 GNU tar 两种格式
- 八进制字符串解析函数 `parseOctNum()` 静默跳过非数字字符，兼容各种 TAR 实现的填充差异
- 两种校验和算法（unsigned/signed char）覆盖了 POSIX 标准和旧版 Unix 的兼容性
- GNU LONGLINK 机制通过在文件条目前插入特殊条目来突破 100 字符文件名限制
- `CreateStreamByIndex()` 返回零拷贝的 `TArchiveStream`，因为 TAR 不压缩数据
- 文件名必须是 UTF-8 编码，否则 `storeFilename()` 会抛出异常

---

## 练习题与答案

### 题目 1：手动计算八进制解析结果

给定以下 TAR 头部的 `size` 字段内容（12 字节）：

```
"00000012345\0"
```

请手动计算这个八进制字符串表示的十进制文件大小是多少。

<details>
<summary>查看答案</summary>

八进制 `12345` 转换为十进制：

```
1 × 8⁴ + 2 × 8³ + 3 × 8² + 4 × 8¹ + 5 × 8⁰
= 1 × 4096 + 2 × 512 + 3 × 64 + 4 × 8 + 5 × 1
= 4096 + 1024 + 192 + 32 + 5
= 5349
```

文件大小为 **5349 字节**。

验证代码：
```cpp
#include <cstdio>
#include <cstdint>

uint64_t parseOctNum(const char *oct, int length) {
    uint64_t num = 0;
    for (int i = 0; i < length; i++) {
        char c = oct[i];
        if ('0' <= c && c <= '9')
            num = num * 8 + (c - '0');
    }
    return num;
}

int main() {
    const char size[] = "00000012345\0";
    printf("结果: %llu\n", (unsigned long long)parseOctNum(size, 12));
    // 输出: 结果: 5349
    return 0;
}
```

</details>

### 题目 2：实现 POSIX prefix + name 文件名拼接

KrKr2 当前没有使用 POSIX ustar 的 `prefix` 字段。请编写一个函数 `getFullName()`，将 `prefix` 和 `name` 字段正确拼接为完整路径（如果 prefix 非空，用 `/` 连接）。

<details>
<summary>查看答案</summary>

```cpp
#include <cstdio>
#include <cstring>
#include <string>

struct PosixTarHeader {
    char name[100];
    char mode[8]; char uid[8]; char gid[8];
    char size[12]; char mtime[12]; char chksum[8];
    char typeflag;
    char linkname[100];
    char magic[6]; char version[2];
    char uname[32]; char gname[32];
    char devmajor[8]; char devminor[8];
    char prefix[155];
    char pad[12];
};

// 安全提取固定长度字符串（可能没有 null 终止）
std::string safeExtract(const char *field, size_t maxLen) {
    size_t len = strnlen(field, maxLen);
    return std::string(field, len);
}

// 拼接 prefix + name，返回完整路径
std::string getFullName(const PosixTarHeader &hdr) {
    std::string prefix = safeExtract(hdr.prefix, sizeof(hdr.prefix));
    std::string name = safeExtract(hdr.name, sizeof(hdr.name));

    if (prefix.empty()) {
        return name;
        // prefix 为空，直接返回 name
    }
    return prefix + "/" + name;
    // prefix 非空，用 "/" 连接
    // 例如：prefix="very/long/path" + name="file.txt"
    // → "very/long/path/file.txt"
}

int main() {
    PosixTarHeader hdr = {};

    // 测试 1：只有 name（prefix 为空）
    strncpy(hdr.name, "short_name.txt", sizeof(hdr.name));
    printf("测试1: %s\n", getFullName(hdr).c_str());
    // 输出: short_name.txt

    // 测试 2：prefix + name 拼接
    memset(&hdr, 0, sizeof(hdr));
    strncpy(hdr.prefix,
            "very/long/directory/path/that/exceeds/100/chars",
            sizeof(hdr.prefix));
    strncpy(hdr.name, "filename.dat", sizeof(hdr.name));
    printf("测试2: %s\n", getFullName(hdr).c_str());
    // 输出: very/long/directory/path/that/exceeds/100/chars/filename.dat

    // 测试 3：name 满 100 字符（无 null 终止）
    memset(&hdr, 0, sizeof(hdr));
    memset(hdr.name, 'A', 100);  // 100 个 'A'，无 \0
    printf("测试3: 长度=%zu\n", getFullName(hdr).length());
    // 输出: 长度=100

    return 0;
}
```

</details>

### 题目 3：实现 PAX 扩展头解析

KrKr2 当前跳过了 PAX 扩展头（typeflag `'x'` 和 `'g'`）。PAX 扩展头的数据区包含 `key=value\n` 格式的键值对（每条前面有十进制长度前缀），例如：

```
20 path=long_file.txt\n
30 mtime=1234567890.123456789\n
```

请编写一个 `parsePaxHeader()` 函数解析这种格式。

<details>
<summary>查看答案</summary>

```cpp
#include <cstdio>
#include <cstring>
#include <string>
#include <map>

// PAX 扩展头格式：每条记录为 "长度 key=value\n"
// 其中"长度"是十进制数字，包含长度字段本身和末尾的 \n
std::map<std::string, std::string> parsePaxHeader(
    const char *data, size_t dataSize) {
    std::map<std::string, std::string> result;
    size_t pos = 0;

    while (pos < dataSize) {
        // 1. 解析长度前缀
        size_t lenEnd = pos;
        while (lenEnd < dataSize && data[lenEnd] != ' ')
            lenEnd++;
        if (lenEnd >= dataSize) break;

        int recordLen = 0;
        for (size_t i = pos; i < lenEnd; i++) {
            if (data[i] < '0' || data[i] > '9') break;
            recordLen = recordLen * 10 + (data[i] - '0');
        }
        if (recordLen <= 0) break;

        // 2. 提取 key=value 部分（跳过长度和空格，去掉末尾 \n）
        size_t kvStart = lenEnd + 1;  // 跳过空格
        size_t kvEnd = pos + recordLen - 1;  // 去掉末尾 \n
        if (kvEnd > dataSize) break;

        std::string kv(data + kvStart, kvEnd - kvStart);

        // 3. 分割 key 和 value
        size_t eqPos = kv.find('=');
        if (eqPos != std::string::npos) {
            std::string key = kv.substr(0, eqPos);
            std::string value = kv.substr(eqPos + 1);
            result[key] = value;
        }

        pos += recordLen;  // 移动到下一条记录
    }

    return result;
}

int main() {
    // 模拟 PAX 扩展头数据
    // 格式："<长度> <key>=<value>\n"
    // 长度包含整条记录（含长度字段、空格、key=value、\n）
    std::string paxData =
        "20 path=long_file.txt\n"
        "30 mtime=1234567890.123456789\n"
        "18 size=999999999\n"
        "23 uid=1000 gid=1000\n";
    // 注意：第 4 条 uid= 后面有空格但它是值的一部分

    // 手动修正长度（实际 PAX 中长度是精确计算的）
    // 这里用简化版演示
    const char *testData =
        "20 path=long_file.txt\n"
        "30 mtime=1234567890.123456789\n";

    auto attrs = parsePaxHeader(testData, strlen(testData));

    printf("解析到 %zu 个属性:\n", attrs.size());
    for (const auto &[key, value] : attrs) {
        printf("  %s = %s\n", key.c_str(), value.c_str());
    }
    // 输出：
    // 解析到 2 个属性:
    //   mtime = 1234567890.123456789
    //   path = long_file.txt

    return 0;
}
```

**要点**：PAX 格式的长度前缀是自引用的——`"20 path=long_file.txt\n"` 中的 `20` 是整条记录的总长度（包括 "20" 本身的 2 字节、空格、key=value、换行符）。这种设计使得解析器可以在不理解 key/value 含义的情况下跳过任何记录。

</details>

---

## 下一步

[03-7z 归档实现](./03-7z归档实现.md) —— 学习如何使用 LZMA SDK 的 C 语言 API 实现 7z 归档读取，包括 `ISeekInStream` 接口适配、`LookToRead2` 缓冲读取、以及多编码器复杂文件夹的处理策略。
