# 03-03 TLG 格式深度解析

## 本节目标

通过本节学习，你将：

1. 理解 TLG（TVP Graphics）格式的三层封装结构：TLG0.0 SDS 容器 → TLG5.0/6.0 原始数据
2. 掌握 TLG5 的改进型 LZSS 逐通道压缩与 Delta 滤波原理
3. 掌握 TLG6 的 Golomb 熵编码与 8×8 块色度滤波机制
4. 能够读懂 `LoadTLG.cpp` 中的完整解码流程并定位关键函数
5. 学会利用元数据标签（tags）提取图像偏移、分辨率等附加信息

---

## 3.1 TLG 格式概述与版本演进

TLG（TVP/KiriKiri Graphics）是吉里吉里引擎专有的图像格式，专为视觉小说场景优化。与通用格式（PNG、JPEG）相比，TLG 在解码速度和压缩率之间做了针对性取舍。

**TLG 的三个版本：**

| 版本 | 标识符 | 压缩算法 | 特点 |
|------|--------|----------|------|
| TLG5.0 | `TLG5.0\x00raw\x1a` | 改进型 LZSS | 无损，逐通道压缩，适合渐变较少的图像 |
| TLG6.0 | `TLG6.0\x00raw\x1a` | Golomb 熵编码 | 无损，块式处理，压缩率更高 |
| TLG0.0 | `TLG0.0\x00sds\x1a` | 容器封装 | 包裹 TLG5/6 数据，附加元数据标签 |

在引擎源码中，所有 TLG 解码都集中在 `visual/LoadTLG.cpp` 一个文件中（约 656 行）。

```
┌─────────────────────────────────────────────┐
│              TLG0.0 SDS 容器                │
│  ┌───────────────────────────────────────┐  │
│  │  标签区 (Tag Block)                   │  │
│  │  "4:LEFT=2:20,3:TOP=3:120,..."        │  │
│  └───────────────────────────────────────┘  │
│  ┌───────────────────────────────────────┐  │
│  │  数据区 (rawlen 字节)                 │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │  TLG5.0 或 TLG6.0 原始数据     │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### 版本标识检测

引擎通过文件头的 11 字节魔数来判断 TLG 版本：

```cpp
// LoadTLG.cpp 中的版本检测逻辑
void TVPLoadTLG(void* formatdata,
                void *callbackdata,
                tTVPGraphicSizeCallback sizecallback,
                tTVPGraphicScanLineCallback scanlinecallback,
                tTVPMetaInfoPushCallback metainfopushcallback,
                tTJSBinaryStream *src,
                tjs_int keyidx,
                tTVPGraphicLoadMode mode)
{
    // 读取 11 字节文件头
    unsigned char mark[12];
    src->ReadBuffer(mark, 11);

    // TLG0.0 SDS 容器 —— 先解析标签，再递归解码内部数据
    if (!memcmp("TLG0.0\x00sds\x1a", mark, 11))
    {
        // 处理 SDS 封装 ...
    }
    // TLG5.0 裸数据
    else if (!memcmp("TLG5.0\x00raw\x1a", mark, 11))
    {
        TVPLoadTLG5(/* ... */);
    }
    // TLG6.0 裸数据
    else if (!memcmp("TLG6.0\x00raw\x1a", mark, 11))
    {
        TVPLoadTLG6(/* ... */);
    }
}
```

> **概念解释：SDS（Structured Data Stream）** 是 TLG0.0 引入的容器层。SDS 本身不做图像压缩，而是在 TLG5/6 数据外面包一层标签元数据。标签采用 `key_len:key=value_len:value` 的文本格式，用逗号分隔。这种设计让引擎能在不解码图像的情况下快速读取偏移量、原始尺寸等信息。

---

## 3.2 TLG5 解码：LZSS 逐通道压缩

TLG5 是较早的版本，使用改进型 LZSS（Lempel-Ziv-Storer-Szymanski）压缩算法。它的核心思路是**将 BGRA 四通道拆开，每个通道独立压缩**。

### 3.2.1 文件头结构

```cpp
// TLG5 文件头（紧跟版本标识之后）
struct TLG5Header {
    unsigned char colors;  // 通道数：3(BGR) 或 4(BGRA)
    tjs_int width;         // 图像宽度（32位小端）
    tjs_int height;        // 图像高度（32位小端）
    tjs_int blockcount;    // 块数量（每块最多4行）
};
```

TLG5 将图像按每 4 行分块（最后一块可能不足 4 行）。每块内每个通道独立用 LZSS 压缩。

### 3.2.2 LZSS 解压核心

```cpp
// 简化版 LZSS 解压（对应 LoadTLG.cpp 中的 TVPDecompressSlide 函数）
static tjs_int TVPDecompressSlide(
    tjs_uint8 *out,          // 输出缓冲区
    const tjs_uint8 *in,     // 压缩数据
    tjs_int insize,          // 压缩数据大小
    tjs_uint8 *text,         // 滑动窗口（4096字节）
    tjs_int initialr)        // 初始写入位置
{
    tjs_int r = initialr;    // 滑动窗口写入指针
    tjs_uint flags = 0;
    const tjs_uint8 *inlim = in + insize;
    
    while (in < inlim) {
        if (((flags >>= 1) & 256) == 0) {
            // 每8个操作读一个标志字节
            flags = 0[in++] | 0xff00;
        }
        
        if (flags & 1) {
            // 标志位=1：字面量字节，直接输出
            tjs_uint8 c = 0[in++];
            *out++ = c;
            text[r++] = c;
            r &= 4095;  // 滑动窗口取模（4096）
        } else {
            // 标志位=0：引用（position, length）对
            tjs_int p = 0[in++];           // 低8位位置
            tjs_int l = 0[in++];           // 高4位位置 + 低4位长度
            p = p | ((l & 0xf0) << 4);     // 12位偏移
            l = (l & 0x0f) + 3;            // 长度=低4位+3（最小匹配3）
            
            for (int i = 0; i < l; i++) {
                *out++ = text[r++] = text[p++];
                p &= 4095;
                r &= 4095;
            }
        }
    }
    return r;  // 返回新的滑动窗口位置
}
```

> **概念解释：滑动窗口（Sliding Window）** 是 LZSS 的核心数据结构。它是一个 4096 字节的循环缓冲区，保存最近解压的数据。当遇到引用对 (p, l) 时，从窗口位置 p 复制 l 个字节。12 位偏移 + 4 位长度的编码方式意味着最大回看距离 4096 字节，最大匹配长度 18 字节（4位+3）。

> **概念解释：标志字节机制**。每 8 个操作共用一个标志字节，每位控制一个操作：1 表示字面量（1字节），0 表示引用（2字节）。`flags >>= 1` 逐位消耗，`& 256` 检测第9位判断是否需要读新标志。这种方式比逐操作编码节省空间。

### 3.2.3 通道合成与 Delta 滤波

TLG5 解压完成后，需要将独立通道合成为 BGRA 像素，并应用**Delta 滤波**：

```cpp
// 通道合成逻辑（LoadTLG.cpp 简化）
// 3通道(BGR)场景 → 输出32位BGRA
if (colors == 3) {
    for (int x = 0; x < width; x++) {
        outbuf[x*4+0] = channelbufs[0][x];  // B
        outbuf[x*4+1] = channelbufs[1][x];  // G
        outbuf[x*4+2] = channelbufs[2][x];  // R
        outbuf[x*4+3] = 0xff;               // A = 不透明
    }
}
// 4通道(BGRA)场景
else {
    for (int x = 0; x < width; x++) {
        outbuf[x*4+0] = channelbufs[0][x];  // B
        outbuf[x*4+1] = channelbufs[1][x];  // G
        outbuf[x*4+2] = channelbufs[2][x];  // R
        outbuf[x*4+3] = channelbufs[3][x];  // A
    }
}

// Delta 滤波：当前行 += 上一行（逐字节累加）
// 这是 TLG5 的关键优化——存储差值而非绝对值
if (prevline) {
    for (int x = 0; x < width * 4; x++) {
        outbuf[x] += prevline[x];
    }
}
```

> **概念解释：Delta 滤波**。在连续的图像行中，相邻行的像素值通常非常接近。TLG5 存储的是当前行与上一行的差值（Delta），而不是原始像素值。差值通常集中在 0 附近，LZSS 对这种数据的压缩效率更高。解码时通过累加还原原始值。

### Delta 滤波效果示意

```
原始数据:        行0: [100, 102, 105, 103]
                 行1: [101, 103, 104, 102]
                 行2: [102, 105, 106, 104]

Delta编码后:     行0: [100, 102, 105, 103]  (首行不变)
                 行1: [  1,   1,  -1,  -1]  (差值接近0)
                 行2: [  1,   2,   2,   2]  (差值接近0)

LZSS对接近0的数据 → 压缩率显著提高
```

---

## 3.3 TLG6 解码：Golomb 熵编码

TLG6 是更先进的版本，使用 Golomb 熵编码和 8×8 块色度滤波，压缩率明显优于 TLG5。

### 3.3.1 文件头与块结构

```cpp
// TLG6 文件头
struct TLG6Header {
    unsigned char colors;      // 通道数（3或4）
    unsigned char data_flags;  // 数据标志
    unsigned char color_type;  // 颜色类型
    unsigned char external_golomb_table;  // 是否使用外部 Golomb 表
    tjs_int width;
    tjs_int height;
    tjs_int max_bit_length;    // 最大位长度
};
// 图像被划分为 8×8 的块进行处理
#define TVP_TLG6_H_BLOCK_SIZE 8
#define TVP_TLG6_W_BLOCK_SIZE 8
```

### 3.3.2 Golomb-Rice 编码原理

Golomb 编码是一种**变长前缀编码**，特别适合编码几何分布的非负整数（即 Delta 滤波后的残差）。

```cpp
// Golomb 解码核心（简化自 LoadTLG.cpp）
// k = Golomb 参数，控制编码效率
// 值 n 被编码为: 一元码(n >> k) + 二进制码(n 的低 k 位)
//
// 例如 k=2 时编码值 13:
//   13 >> 2 = 3  → 一元码 "1110"（3个1 + 终止0）
//   13 & 3  = 1  → 二进制 "01"
//   完整编码: "111001"

// 实际解码使用查表加速
static tjs_uint8 TVPTLG6GolombBitLengthTable
    [TVP_TLG6_GOLOMB_N_COUNT * 2 * 128][TVP_TLG6_GOLOMB_N_COUNT];

// 表在 tvpgl.cpp 中初始化
void TVPFillGolombTable() {
    for (int n = 0; n < TVP_TLG6_GOLOMB_N_COUNT; n++) {
        int a = 0;
        for (int i = 0; i < 9; i++) {
            for (int j = 0; j < (1 << i); j++) {
                TVPTLG6GolombBitLengthTable[a++][n] = 
                    (tjs_uint8)(i + n + 1);
            }
        }
    }
}
```

> **概念解释：Golomb 参数 k**。k 决定了编码的"粒度"。k 值越大，对较大数值的编码越短，但对小数值的编码变长。TLG6 根据每个 8×8 块的数据特征动态选择最优 k 值，使整体压缩率最大化。在源码中，k 值通过 `TVPTLG6GolombBitLengthTable` 查表获得，避免逐位运算的开销。

### 3.3.3 色度滤波器

TLG6 对每个 8×8 块应用色度滤波器，利用通道间的相关性进一步提高压缩率：

```cpp
// 色度滤波器类型（对应 TVPTLG6DecodeLineGeneric 中的 ft 参数）
// 滤波器将 (B,G,R) 转换为残差表示，解码时做逆变换

// 滤波器0: 无变换
//   B' = B, G' = G, R' = R

// 滤波器1: G 通道作为基准
//   B' = B - G, G' = G, R' = R - G
//   解码: B = B' + G, R = R' + G

// 滤波器2: BG 联合
//   B' = B - G, G' = G - R, R' = R
//   解码: G = G' + R, B = B' + G

// 源码中的色度滤波解码实现
#define TVP_TLG6_DO_CHROMA_DECODE(N, R, G, B) \
    case N: \
        /* 根据滤波器类型做逆变换 */ \
        break;

static void TVPTLG6DecodeLineGeneric(
    tjs_uint32 *prevline,
    tjs_uint32 *curline,
    tjs_int width,
    tjs_int start_block,
    tjs_int block_limit,
    tjs_uint8 *filtertypes,     // 每块的滤波器类型
    tjs_int skipblockbytes,
    tjs_uint32 *inbuf,
    tjs_uint8 *flagbuf,         // MED 预测标志
    tjs_int oddskip,
    tjs_int dir)                // 方向（左→右 或 右→左）
```

### 3.3.4 MED 预测与方向交替

TLG6 使用 **MED（Median Edge Detector）预测器**，并在相邻行之间交替解码方向：

```cpp
// MED 预测器：根据左(a)、上(b)、左上(c)三个像素预测当前值
// 预测值 p 计算规则：
//   if (c >= max(a, b))  p = min(a, b)    // 水平或垂直边缘
//   if (c <= min(a, b))  p = max(a, b)    // 对角边缘
//   else                 p = a + b - c    // 平滑区域

// 方向交替：偶数行从左到右，奇数行从右到左
// 这利用了2D空间局部性，提高预测精度
int dir = (y & 1) ? -1 : 1;  // 交替方向
int start = (dir == 1) ? 0 : width - 1;
```

> **概念解释：MED 预测器**。MED 是一种自适应预测方法，比简单的左邻预测或上邻预测更准确。它通过比较三个已知邻居像素 (a=左, b=上, c=左上)，自动检测边缘方向并选择最佳预测。预测残差（实际值 - 预测值）的幅度更小，使 Golomb 编码效率更高。

---

## 3.4 TLG0.0 SDS 容器与元数据标签

TLG0.0 SDS（Structured Data Stream）是一个容器格式，在 TLG5/6 数据外包裹元数据：

```cpp
// SDS 解析逻辑（LoadTLG.cpp 行 446-599，简化版）
if (!memcmp("TLG0.0\x00sds\x1a", mark, 11)) {
    // 1. 读取原始数据长度
    tjs_int rawlen = src->ReadI32LE();
    
    // 2. 读取标签块大小（循环读取所有 chunk）
    while (true) {
        tjs_int chunkname_len = src->ReadI32LE();
        if (chunkname_len == 0) break;
        
        char chunkname[256];
        src->ReadBuffer(chunkname, chunkname_len);
        tjs_int chunksize = src->ReadI32LE();
        
        if (!memcmp(chunkname, "tags", 4)) {
            // 解析标签字符串
            // 格式: "key_len:key=value_len:value,..."
            ParseTags(src, chunksize, metainfopushcallback);
        } else {
            src->SetPosition(src->GetPosition() + chunksize);
        }
    }
    
    // 3. 递归调用 TVPLoadTLG 解码内部数据
    TVPLoadTLG(formatdata, callbackdata,
               sizecallback, scanlinecallback,
               metainfopushcallback, src, keyidx, mode);
}
```

### 标签格式详解

```
标签字符串示例：
"4:LEFT=2:20,3:TOP=3:120,5:WIDTH=3:800,6:HEIGHT=3:600"

解析规则：
  key_len_digits : key_string = value_len_digits : value_string , ...
  
  "4:LEFT"   → key_len=4, key="LEFT"
  "2:20"     → value_len=2, value="20"
  逗号分隔多个标签对

常用标签：
  LEFT   - 图像在画面中的 X 偏移
  TOP    - 图像在画面中的 Y 偏移
  WIDTH  - 原始宽度（差分图可能小于画面宽度）
  HEIGHT - 原始高度
```

```cpp
// 标签解析代码示例
void ParseTLGTags(const char* tagstr, int len) {
    const char* p = tagstr;
    const char* end = tagstr + len;
    
    while (p < end) {
        // 读 key 长度
        int key_len = 0;
        while (*p != ':') { key_len = key_len * 10 + (*p - '0'); p++; }
        p++;  // 跳过 ':'
        
        // 读 key
        std::string key(p, key_len);
        p += key_len;
        p++;  // 跳过 '='
        
        // 读 value 长度
        int val_len = 0;
        while (*p != ':') { val_len = val_len * 10 + (*p - '0'); p++; }
        p++;  // 跳过 ':'
        
        // 读 value
        std::string value(p, val_len);
        p += val_len;
        
        if (p < end && *p == ',') p++;  // 跳过逗号
        
        // 通过回调传递给引擎
        // metainfopushcallback(callbackdata, key, value);
    }
}
```

---

## 3.5 Header-Only 加载

引擎支持仅读取 TLG 文件头而不解码像素数据，用于预取图像尺寸：

```cpp
// TVPLoadHeaderTLG（LoadTLG.cpp 行 601-656）
void TVPLoadHeaderTLG(void* formatdata,
                      tTJSBinaryStream *src,
                      iTVPGraphicHeaderInfo *info)
{
    unsigned char mark[12];
    src->ReadBuffer(mark, 11);
    
    // SDS 容器：跳过标签，递归读内部头
    if (!memcmp("TLG0.0\x00sds\x1a", mark, 11)) {
        // 跳过 rawlen + 所有 chunk
        // 再次调用 TVPLoadHeaderTLG
    }
    // TLG5: 读取 colors + width + height
    else if (!memcmp("TLG5.0\x00raw\x1a", mark, 11)) {
        info->ColorFormat = src->ReadI8();   // 3 或 4
        info->Width = src->ReadI32LE();
        info->Height = src->ReadI32LE();
        info->BitsPerPixel = info->ColorFormat * 8;
    }
    // TLG6: 读取 colors + data_flags + ... + width + height
    else if (!memcmp("TLG6.0\x00raw\x1a", mark, 11)) {
        info->ColorFormat = src->ReadI8();
        src->ReadI8();  // data_flags（跳过）
        src->ReadI8();  // color_type（跳过）
        src->ReadI8();  // external_golomb_table（跳过）
        info->Width = src->ReadI32LE();
        info->Height = src->ReadI32LE();
        info->BitsPerPixel = info->ColorFormat * 8;
    }
}
```

这在图层预加载、资源管理器预览等场景非常有用——无需解压整个图像就能获取尺寸信息。

---

## 动手实践

### 实践1：手动解析 TLG5 文件头

```cpp
#include <fstream>
#include <cstdint>
#include <cstring>
#include <iostream>

struct TLG5Info {
    uint8_t  version[11];
    uint8_t  colors;
    int32_t  width;
    int32_t  height;
    int32_t  blockcount;
};

bool ParseTLG5Header(const char* filename, TLG5Info& info) {
    std::ifstream fs(filename, std::ios::binary);
    if (!fs) return false;
    
    fs.read(reinterpret_cast<char*>(info.version), 11);
    
    // 检查是否为 TLG0.0 SDS 容器
    if (memcmp(info.version, "TLG0.0\x00sds\x1a", 11) == 0) {
        // 跳过 SDS 封装，读取内部 TLG5/6 数据
        int32_t rawlen;
        fs.read(reinterpret_cast<char*>(&rawlen), 4);
        
        // 跳过 chunk 列表直到 chunkname_len == 0
        while (true) {
            int32_t chunkname_len;
            fs.read(reinterpret_cast<char*>(&chunkname_len), 4);
            if (chunkname_len == 0) break;
            fs.seekg(chunkname_len, std::ios::cur);
            int32_t chunksize;
            fs.read(reinterpret_cast<char*>(&chunksize), 4);
            fs.seekg(chunksize, std::ios::cur);
        }
        
        // 重新读取内部版本标识
        fs.read(reinterpret_cast<char*>(info.version), 11);
    }
    
    if (memcmp(info.version, "TLG5.0\x00raw\x1a", 11) != 0) {
        std::cerr << "不是 TLG5 格式" << std::endl;
        return false;
    }
    
    fs.read(reinterpret_cast<char*>(&info.colors), 1);
    fs.read(reinterpret_cast<char*>(&info.width), 4);
    fs.read(reinterpret_cast<char*>(&info.height), 4);
    fs.read(reinterpret_cast<char*>(&info.blockcount), 4);
    
    return true;
}

int main() {
    TLG5Info info;
    if (ParseTLG5Header("test.tlg", info)) {
        std::cout << "通道数: " << (int)info.colors << std::endl;
        std::cout << "尺寸: " << info.width << " x " << info.height << std::endl;
        std::cout << "块数: " << info.blockcount << std::endl;
        std::cout << "预期块数: " << ((info.height - 1) / 4 + 1) << std::endl;
    }
    return 0;
}
```

### 实践2：LZSS 滑动窗口模拟

```cpp
#include <iostream>
#include <vector>
#include <cstdint>

// 简化版 LZSS 解压器——仅用于理解原理
class SimpleLZSS {
    uint8_t window_[4096];
    int r_ = 0;
    
public:
    SimpleLZSS() { memset(window_, 0, sizeof(window_)); }
    
    std::vector<uint8_t> Decompress(const uint8_t* in, int insize) {
        std::vector<uint8_t> out;
        int pos = 0;
        uint32_t flags = 0;
        
        while (pos < insize) {
            if (((flags >>= 1) & 256) == 0) {
                flags = in[pos++] | 0xff00;
            }
            
            if (flags & 1) {
                // 字面量
                uint8_t c = in[pos++];
                out.push_back(c);
                window_[r_++] = c;
                r_ &= 4095;
            } else {
                // 引用
                int p = in[pos++];
                int l = in[pos++];
                p = p | ((l & 0xf0) << 4);
                l = (l & 0x0f) + 3;
                
                for (int i = 0; i < l; i++) {
                    uint8_t c = window_[p & 4095];
                    out.push_back(c);
                    window_[r_++ & 4095] = c;
                    p++;
                }
            }
        }
        return out;
    }
};
```

---

## 对照项目源码

| 本节概念 | 源码位置 | 说明 |
|---------|---------|------|
| TLG 入口分发 | `LoadTLG.cpp:446-599` | `TVPLoadTLG` 函数，版本检测与 SDS 解析 |
| TLG5 解码 | `LoadTLG.cpp:41-198` | `TVPLoadTLG5`，LZSS 解压 + 通道合成 |
| TLG6 解码 | `LoadTLG.cpp:204-441` | `TVPLoadTLG6`，Golomb + 色度滤波 |
| LZSS 解压 | `LoadTLG.cpp:41-90` | `TVPDecompressSlide` 函数 |
| Golomb 表初始化 | `tvpgl.cpp` | `TVPFillGolombTable` 函数 |
| 色度滤波宏 | `LoadTLG.cpp:250-320` | `TVP_TLG6_DO_CHROMA_DECODE` 宏 |
| Header 读取 | `LoadTLG.cpp:601-656` | `TVPLoadHeaderTLG` 函数 |
| 块大小常量 | `tvpgl.h` | `TVP_TLG6_H_BLOCK_SIZE = 8` |

---

## 常见错误与排查

### 错误1：TLG 文件头不匹配

**症状**：加载时抛出 "Not a TLG file" 异常。

**原因**：
- 文件不是 TLG 格式（可能是改了扩展名的 PNG/BMP）
- 文件损坏，前 11 字节被破坏
- 字节序问题（TLG 使用小端序）

**排查步骤**：
```bash
# 查看文件头（十六进制）
# Windows:
certutil -encodehex test.tlg head.txt 4 | head -1

# Linux/macOS:
xxd -l 16 test.tlg

# 预期输出（TLG6）:
# 54 4c 47 36 2e 30 00 72 61 77 1a ...
# T  L  G  6  .  0  \0 r  a  w  \x1a
```

### 错误2：解码后图像颜色异常（偏蓝/偏红）

**症状**：图像可以加载但颜色不对。

**原因**：TLG 内部使用 **BGRA** 字节序，如果按 RGBA 解释就会 B/R 通道互换。

**修复**：
```cpp
// 错误：假设 RGBA 顺序
pixel.r = data[0];  // 实际是 B！
pixel.g = data[1];
pixel.b = data[2];  // 实际是 R！

// 正确：使用 BGRA 顺序
pixel.b = data[0];
pixel.g = data[1];
pixel.r = data[2];
pixel.a = data[3];
```

### 错误3：TLG6 解码时 Golomb 表未初始化

**症状**：解码产生随机噪声或崩溃。

**原因**：`TVPTLG6GolombBitLengthTable` 未调用 `TVPFillGolombTable()` 初始化。

**修复**：确保在首次加载 TLG6 前调用初始化：
```cpp
// 引擎启动时自动初始化（tvpgl.cpp 中）
static bool initialized = false;
if (!initialized) {
    TVPFillGolombTable();
    initialized = true;
}
```

---

## 跨平台差异

| 方面 | Windows | Linux | macOS | Android |
|------|---------|-------|-------|---------|
| 字节序 | 小端序（x86） | 小端序（x86/ARM-LE） | 小端序（Apple Silicon/x86） | 小端序（ARM-LE） |
| TLG 文件路径 | `data\image.tlg` | `data/image.tlg` | `data/image.tlg` | assets 或内部存储 |
| SIMD 加速 | SSE2（x86） | SSE2/NEON | NEON（Apple Silicon） | NEON（ARM） |
| 内存对齐 | 4字节对齐 | 4字节对齐 | 16字节对齐推荐 | 4字节对齐 |

> **注意**：TLG 格式本身是平台无关的（固定小端序），但解码器中的 SIMD 优化路径因平台而异。KrKr2 在 ARM 平台上会自动退回到纯 C 实现。

---

## 本节小结

- TLG 格式有三层结构：**TLG0.0 SDS 容器**（可选的标签封装）→ **TLG5.0/6.0 原始数据**
- TLG5 使用**改进型 LZSS** 压缩，核心是 4096 字节滑动窗口 + 逐通道独立压缩 + Delta 行间滤波
- TLG6 使用**Golomb 熵编码**，配合 8×8 块色度滤波和 MED 预测器，压缩率更高
- SDS 标签使用 `key_len:key=value_len:value` 文本格式存储元数据
- Header-only 加载可以不解码像素就获取图像尺寸，用于资源预加载
- 内部始终使用 **BGRA** 字节序，注意不要与 RGBA 混淆

---

## 练习题与答案

### 练习1：LZSS 参数计算

**问题**：TLG5 的 LZSS 使用 12 位偏移和 4 位长度（加 3 的偏移）。请计算：
1. 滑动窗口最大大小是多少字节？
2. 最大匹配长度是多少字节？
3. 一个引用对占多少字节？

<details>
<summary>查看答案</summary>

1. 滑动窗口大小 = 2^12 = **4096 字节**
2. 最大匹配长度 = 2^4 - 1 + 3 = 15 + 3 = **18 字节**
3. 一个引用对占 **2 字节**（低8位偏移 + 高4位偏移与低4位长度共享第二字节）

压缩增益分析：引用对 2 字节可以替代 3-18 字节的重复数据。当匹配长度 ≥3 时，使用引用比字面量更划算。

</details>

### 练习2：色度滤波器选择

**问题**：如果一个 8×8 块中 B 和 R 通道的值与 G 通道高度相关（例如灰度渐变），应该选择哪种色度滤波器？为什么？

<details>
<summary>查看答案</summary>

应选择**滤波器1**（`B' = B - G, R' = R - G`）。

原因：当 B ≈ G ≈ R 时（灰度区域），B-G 和 R-G 的值接近 0。这意味着：
- 滤波后的 B' 和 R' 通道数据集中在 0 附近
- Golomb 编码对接近 0 的值编码最短
- 只需要精确编码 G 通道，B' 和 R' 几乎不占空间

如果使用滤波器0（无变换），则三个通道都需要完整编码，浪费空间。

</details>

### 练习3：SDS 标签解析

**问题**：给定标签字符串 `"4:LEFT=3:100,3:TOP=2:50,5:WIDTH=4:1024"`，请手动解析出所有键值对。

<details>
<summary>查看答案</summary>

解析过程：

1. `4:LEFT=3:100` → key_len=4, key="LEFT", val_len=3, value="100"
2. `3:TOP=2:50` → key_len=3, key="TOP", val_len=2, value="50"
3. `5:WIDTH=4:1024` → key_len=5, key="WIDTH", val_len=4, value="1024"

结果：
| 键 | 值 |
|----|----|
| LEFT | 100 |
| TOP | 50 |
| WIDTH | 1024 |

注意：值都是字符串类型，使用时需要转换为整数。

</details>

---

## 下一步

下一节我们将进入 **第四章：OGL 后端**，学习 KrKr2 的 OpenGL 渲染管理器架构，包括：
- `iTVPRenderManager` 接口设计与注册机制
- 软件/硬件渲染后端的切换策略
- 纹理管理与 GPU 资源生命周期
