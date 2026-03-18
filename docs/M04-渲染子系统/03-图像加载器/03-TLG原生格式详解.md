# TLG 原生格式详解

> **所属模块：** M04-渲染子系统
> **前置知识：** [01-加载器注册机制](./01-加载器注册机制.md)、[02-主流格式加载器实现](./02-主流格式加载器实现.md)
> **预计阅读时间：** 45 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 TLG5 和 TLG6 格式的文件结构和 Magic Bytes
2. 掌握 LZSS 滑动窗口压缩算法在 TLG5 中的应用
3. 理解 Golomb-Rice 熵编码在 TLG6 中的实现
4. 分析 MED（Median Edge Detector）像素预测算法
5. 实现 TLG 格式的完整解码流程

---

## 1. TLG 格式概述

TLG（T Visual Graphics）是 KiriKiri2 引擎的原生图像格式，由 W.Dee 设计开发。该格式针对视觉小说场景优化，具有以下特点：

| 特性 | TLG5 | TLG6 |
|------|------|------|
| 设计目标 | 极速解码 | 高压缩率 + 快速解码 |
| 压缩算法 | 改进型 LZSS | Golomb-Rice + 色彩滤波 |
| 解码速度 | 最快 | 约为 TLG5 的 70-80% |
| 压缩率 | 中等 | 接近 JPEG2000 无损 |
| 色彩支持 | RGB/RGBA | 灰度/RGB/RGBA |
| 元数据支持 | 无 | TLG0.0 SDS 容器 |

### 1.1 文件结构层次

```
┌─────────────────────────────────────────────────┐
│ TLG0.0 SDS 容器 (可选)                          │
│  ├─ Magic: "TLG0.0\x00sds\x1a\x00" (11 bytes)   │
│  ├─ Raw Data Size (4 bytes, LE)                 │
│  ├─ Raw TLG Data (TLG5 或 TLG6)                 │
│  └─ Chunks (tags, etc.)                         │
├─────────────────────────────────────────────────┤
│ TLG5 Raw Data                                   │
│  └─ Magic: "TLG5.0\x00raw\x1a\x00" (11 bytes)   │
├─────────────────────────────────────────────────┤
│ TLG6 Raw Data                                   │
│  └─ Magic: "TLG6.0\x00raw\x1a\x00" (11 bytes)   │
└─────────────────────────────────────────────────┘
```

### 1.2 Magic Bytes 识别

```cpp
// 文件路径: cpp/core/visual/LoadTLG.cpp 第 457-467 行

// TLG5 Magic: 54 4C 47 35 2E 30 00 72 61 77 1A 00
// ASCII:      T  L  G  5  .  0  \0 r  a  w  \x1a \0
static const char TLG5_MAGIC[] = "TLG5.0\x00raw\x1a\x00";

// TLG6 Magic: 54 4C 47 36 2E 30 00 72 61 77 1A 00
// ASCII:      T  L  G  6  .  0  \0 r  a  w  \x1a \0
static const char TLG6_MAGIC[] = "TLG6.0\x00raw\x1a\x00";

// TLG0.0 SDS 容器 Magic
static const char TLG0_SDS_MAGIC[] = "TLG0.0\x00sds\x1a\x00";

bool IsTLGFormat(const unsigned char* data, size_t size) {
    if (size < 11) return false;
    
    // 检查三种可能的 Magic
    if (memcmp(data, TLG5_MAGIC, 11) == 0) return true;
    if (memcmp(data, TLG6_MAGIC, 11) == 0) return true;
    if (memcmp(data, TLG0_SDS_MAGIC, 11) == 0) return true;
    
    return false;
}
```

---

## 2. TLG5 格式解析

TLG5 针对解码速度优化，使用改进型 LZSS（Lempel-Ziv-Storer-Szymanski）压缩算法。

### 2.1 TLG5 文件头结构

```cpp
// TLG5 Header Structure
struct TLG5Header {
    char     magic[11];     // "TLG5.0\x00raw\x1a\x00"
    uint8_t  colors;        // 颜色通道数: 3=RGB, 4=RGBA
    uint32_t width;         // 图像宽度 (Little Endian)
    uint32_t height;        // 图像高度 (Little Endian)  
    uint32_t blockHeight;   // 块高度（默认 4）
    // 后跟 blockCount 个 uint32_t 的块大小数组
};

// 解析示例
void ParseTLG5Header(tTJSBinaryStream* src, TLG5Header& header) {
    // 读取 Magic（假设已验证）
    src->ReadBuffer(header.magic, 11);
    
    // 读取颜色通道数
    unsigned char colorByte;
    src->ReadBuffer(&colorByte, 1);
    header.colors = colorByte;
    
    // 读取维度信息（Little Endian）
    header.width = src->ReadI32LE();
    header.height = src->ReadI32LE();
    header.blockHeight = src->ReadI32LE();
    
    // 验证颜色通道
    if (header.colors != 3 && header.colors != 4) {
        throw std::runtime_error("TLG5: Unsupported color type");
    }
}
```

### 2.2 LZSS 滑动窗口压缩

TLG5 使用改进型 LZSS 算法，滑动窗口大小为 4096 字节，最大匹配长度为 18+255=273 字节。

```cpp
// 文件路径: cpp/core/visual/SaveTLG.h

#define SLIDE_N 4096      // 滑动窗口大小（4KB）
#define SLIDE_M (18 + 255) // 最大匹配长度（273 字节）

// LZSS 压缩器结构
class SlideCompressor {
    struct Chain {
        int Prev;  // 链表前驱
        int Next;  // 链表后继
    };

    unsigned char Text[SLIDE_N + SLIDE_M - 1];  // 滑动窗口文本缓冲区
    int Map[256 * 256];                          // 双字节哈希映射表
    Chain Chains[SLIDE_N];                       // 匹配链表
    int S;                                       // 当前窗口位置

public:
    SlideCompressor();
    
    // 编码函数：将 in 压缩到 out
    void Encode(const unsigned char* in, long inlen, 
                unsigned char* out, long& outlen);
    
    // 状态保存/恢复（用于最优压缩选择）
    void Store();
    void Restore();
};
```

### 2.3 LZSS 解码实现

```cpp
// 文件路径: cpp/core/visual/tvpgl.h 第 864-867 行
// LZSS 解压缩函数声明

TVP_GL_FUNC_PTR_EXTERN_DECL(tjs_int, TVPTLG5DecompressSlide,
    (tjs_uint8* out, const tjs_uint8* in, tjs_int insize, 
     tjs_uint8* text, tjs_int initialr));

// LZSS 解码核心算法
tjs_int TVPTLG5DecompressSlide_c(
    tjs_uint8* out,       // 输出缓冲区
    const tjs_uint8* in,  // 输入（压缩）数据
    tjs_int insize,       // 输入大小
    tjs_uint8* text,      // 4096 字节滑动窗口
    tjs_int initialr      // 初始写入位置
) {
    tjs_int r = initialr;
    const tjs_uint8* inlim = in + insize;
    
    while (in < inlim) {
        // 读取标志字节，每位表示后续 8 个 token 的类型
        tjs_uint flags = *in++;
        
        for (int bit = 0; bit < 8 && in < inlim; bit++) {
            if (flags & (1 << bit)) {
                // 标志位=1：匹配引用 (position, length)
                // 从压缩数据读取位置和长度
                tjs_int pos = in[0] | ((in[1] & 0x0F) << 8);
                tjs_int len = (in[1] >> 4) + 3;  // 最小长度 3
                
                if (len == 18) {
                    // 扩展长度编码
                    len = 18 + in[2];
                    in += 3;
                } else {
                    in += 2;
                }
                
                // 从滑动窗口复制匹配数据
                for (int i = 0; i < len; i++) {
                    tjs_uint8 c = text[pos];
                    pos = (pos + 1) & (SLIDE_N - 1);  // 环形缓冲
                    *out++ = c;
                    text[r] = c;
                    r = (r + 1) & (SLIDE_N - 1);
                }
            } else {
                // 标志位=0：字面量字节
                tjs_uint8 c = *in++;
                *out++ = c;
                text[r] = c;
                r = (r + 1) & (SLIDE_N - 1);
            }
        }
    }
    
    return r;  // 返回新的写入位置
}
```

### 2.4 TLG5 颜色重建

TLG5 使用差分编码存储像素值，需要在解码后重建实际颜色。

```cpp
// 文件路径: cpp/core/visual/LoadTLG.cpp 第 137-174 行
// TLG5 颜色合成（首行处理）

// 首行没有上行参考，使用累积差分
for (tjs_int pr = 0, pg = 0, pb = 0, x = 0; x < width; x++) {
    tjs_int r = outbufp[0][x];  // R 差分
    tjs_int g = outbufp[1][x];  // G 差分
    tjs_int b = outbufp[2][x];  // B 差分
    
    // 色彩相关性解码：B 和 R 基于 G 的差分
    b += g;  // B' = B_diff + G_diff
    r += g;  // R' = R_diff + G_diff
    
    // 累积重建
    0[current++] = pb += b;  // B 累积
    0[current++] = pg += g;  // G 累积
    0[current++] = pr += r;  // R 累积
    0[current++] = 0xff;     // Alpha = 255 (RGB 模式)
}

// 非首行处理（使用上行参考）
// 文件路径: cpp/core/visual/tvpgl.h 第 858-863 行
TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPTLG5ComposeColors3To4,
    (tjs_uint8* outp, const tjs_uint8* upper,
     tjs_uint8* const* buf, tjs_int width));

TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPTLG5ComposeColors4To4,
    (tjs_uint8* outp, const tjs_uint8* upper,
     tjs_uint8* const* buf, tjs_int width));
```

### 2.5 完整 TLG5 解码流程

```cpp
// 简化版 TLG5 解码器
void DecodeTLG5(tTJSBinaryStream* src, ImageBuffer& image) {
    // 1. 读取头部
    TLG5Header header;
    ParseTLG5Header(src, header);
    
    // 2. 计算块数量
    int blockCount = (header.height - 1) / header.blockHeight + 1;
    
    // 3. 跳过块大小表（blockCount * 4 字节）
    src->SetPosition(src->GetPosition() + blockCount * sizeof(uint32_t));
    
    // 4. 分配缓冲区
    image.Allocate(header.width, header.height, header.colors);
    
    // LZSS 滑动窗口（4KB + 16 字节对齐边界）
    uint8_t* text = (uint8_t*)TJSAlignedAlloc(4096 + 32, 4) + 16;
    memset(text, 0, 4096);
    
    // 各通道输出缓冲区
    uint8_t* outbuf[4];
    for (int c = 0; c < header.colors; c++) {
        outbuf[c] = new uint8_t[header.blockHeight * header.width];
    }
    
    int r = 0;  // LZSS 窗口位置
    uint8_t* prevline = nullptr;
    
    // 5. 按块解码
    for (int y_blk = 0; y_blk < header.height; y_blk += header.blockHeight) {
        // 解码各颜色通道
        for (int c = 0; c < header.colors; c++) {
            uint8_t mark;
            src->ReadBuffer(&mark, 1);
            int size = src->ReadI32LE();
            
            if (mark == 0) {
                // LZSS 压缩数据
                uint8_t* inbuf = new uint8_t[size];
                src->ReadBuffer(inbuf, size);
                r = TVPTLG5DecompressSlide(outbuf[c], inbuf, size, text, r);
                delete[] inbuf;
            } else {
                // 原始数据（压缩无效时回退）
                src->ReadBuffer(outbuf[c], size);
            }
        }
        
        // 6. 颜色合成
        int y_lim = std::min(y_blk + header.blockHeight, (int)header.height);
        uint8_t* outbufp[4];
        for (int c = 0; c < header.colors; c++) {
            outbufp[c] = outbuf[c];
        }
        
        for (int y = y_blk; y < y_lim; y++) {
            uint8_t* current = image.GetScanline(y);
            
            if (prevline) {
                // 使用优化的 SIMD 函数
                TVPTLG5ComposeColors4To4(current, prevline, outbufp, header.width);
            } else {
                // 首行特殊处理（无参考行）
                ComposeFirstLine(current, outbufp, header.width, header.colors);
            }
            
            prevline = current;
            for (int c = 0; c < header.colors; c++) {
                outbufp[c] += header.width;
            }
        }
    }
    
    // 7. 清理
    TJSAlignedDealloc(text - 16);
    for (int c = 0; c < header.colors; c++) {
        delete[] outbuf[c];
    }
}
```

---

## 3. TLG6 格式解析

TLG6 是 KiriKiri2 的高级图像格式，设计目标是在保持快速解码的同时实现接近 JPEG2000 无损模式的压缩率。

### 3.1 TLG6 压缩流程概览

```
原始图像
    ↓
┌───────────────────────────────────────┐
│ 1. MED/平均法 像素预测                │
│    - 预测每个像素值                   │
│    - 计算预测误差                     │
└───────────────────────────────────────┘
    ↓
┌───────────────────────────────────────┐
│ 2. 8×8 块重排序                       │
│    - 锯齿形扫描顺序                   │
│    - 提高连续性                       │
└───────────────────────────────────────┘
    ↓
┌───────────────────────────────────────┐
│ 3. 色彩相关滤波 (40 种滤波器)         │
│    - 选择最优滤波器                   │
│    - 去除 RGB 相关性                  │
└───────────────────────────────────────┘
    ↓
┌───────────────────────────────────────┐
│ 4. Golomb-Rice 熵编码                 │
│    - 零游程编码                       │
│    - 自适应比特长度                   │
└───────────────────────────────────────┘
    ↓
压缩数据
```

### 3.2 TLG6 文件头结构

```cpp
// TLG6 Header Structure
struct TLG6Header {
    char     magic[11];       // "TLG6.0\x00raw\x1a\x00"
    uint8_t  colors;          // 颜色通道: 1=灰度, 3=RGB, 4=RGBA
    uint8_t  dataFlags;       // 数据标志（当前必须为 0）
    uint8_t  colorType;       // 颜色类型（当前必须为 0）
    uint8_t  externalGolomb;  // 外部 Golomb 表（当前必须为 0）
    uint32_t width;           // 图像宽度
    uint32_t height;          // 图像高度
    uint32_t maxBitLength;    // 最大比特长度
};

// 解析示例
void ParseTLG6Header(tTJSBinaryStream* src, TLG6Header& header) {
    unsigned char buf[4];
    src->ReadBuffer(buf, 4);
    
    header.colors = buf[0];
    header.dataFlags = buf[1];
    header.colorType = buf[2];
    header.externalGolomb = buf[3];
    
    // 验证
    if (header.colors != 1 && header.colors != 3 && header.colors != 4) {
        throw std::runtime_error("TLG6: Unsupported color count");
    }
    if (header.dataFlags != 0 || header.colorType != 0 || header.externalGolomb != 0) {
        throw std::runtime_error("TLG6: Invalid header flags");
    }
    
    header.width = src->ReadI32LE();
    header.height = src->ReadI32LE();
    header.maxBitLength = src->ReadI32LE();
}
```

### 3.3 MED 像素预测算法

MED（Median Edge Detector）是一种自适应预测算法，能根据局部边缘方向选择最佳预测值。

```cpp
// MED 预测算法
// 位置参考：
//   +---+---+
//   | c | b |
//   +---+---+
//   | a | x |  ← x 是要预测的像素
//   +---+---+

uint8_t MEDPredict(uint8_t a, uint8_t b, uint8_t c) {
    // 计算 a 和 b 的最小/最大值
    uint8_t min_ab = (a < b) ? a : b;
    uint8_t max_ab = (a > b) ? a : b;
    
    // MED 预测逻辑
    if (c >= max_ab) {
        // c 在 a、b 之上 → 检测到水平边缘
        // 使用较小值（垂直方向）
        return min_ab;
    } else if (c < min_ab) {
        // c 在 a、b 之下 → 检测到垂直边缘
        // 使用较大值（水平方向）
        return max_ab;
    } else {
        // 平滑区域 → 使用线性预测
        return a + b - c;
    }
}

// 平均法预测（备选）
uint8_t AveragePredict(uint8_t a, uint8_t b) {
    return (a + b + 1) >> 1;  // 四舍五入平均
}

// TLG6 选择最优预测方法
// 文件路径: cpp/core/visual/SaveTLG6.cpp 第 1081-1121 行
for (int p = 0; p < 2; p++) {  // p=0: MED, p=1: Average
    for (int c = 0; c < colors; c++) {
        int wp = 0;
        for (int yy = y; yy < ylim; yy++) {
            const uint8_t* sl = scanline[yy];
            const uint8_t* usl = (yy > 0) ? scanline[yy-1] : nullptr;
            
            for (int xx = x; xx < xlim; xx++) {
                uint8_t pa = (xx > 0) ? sl[-stride] : 0;  // 左像素
                uint8_t pb = usl ? *usl : 0;               // 上像素
                uint8_t px = *sl;                          // 当前像素
                
                uint8_t py;
                if (p == 0) {
                    // MED 预测
                    uint8_t pc = (xx > 0 && usl) ? usl[-stride] : 0;
                    uint8_t min_ab = (pa > pb) ? pb : pa;
                    uint8_t max_ab = (pa < pb) ? pb : pa;
                    
                    if (pc >= max_ab)
                        py = min_ab;
                    else if (pc < min_ab)
                        py = max_ab;
                    else
                        py = pa + pb - pc;
                } else {
                    // 平均法预测
                    py = (pa + pb + 1) >> 1;
                }
                
                // 存储预测误差
                buf[c][wp++] = (uint8_t)(px - py);
                sl += stride;
                if (usl) usl += stride;
            }
        }
    }
}
```

### 3.4 8×8 块重排序（Zigzag Reordering）

为了提高熵编码效率，TLG6 将相邻像素重新排列成锯齿形顺序。

```cpp
// 文件路径: cpp/core/visual/SaveTLG6.cpp 第 292-294 行
#define W_BLOCK_SIZE 8
#define H_BLOCK_SIZE 8

// 块重排序逻辑
// 偶数列块（从上到下）：
//  1  2  3  4  5  6  7  8
// 16 15 14 13 12 11 10  9
// 17 18 19 20 21 22 23 24
// 32 31 30 29 28 27 26 25
// 33 34 35 36 37 38 39 40
// 48 47 46 45 44 43 42 41
// 49 50 51 52 53 54 55 56
// 64 63 62 61 60 59 58 57

// 奇数列块（从下到上）：
// 57 58 59 60 61 62 63 64
// 56 55 54 53 52 51 50 49
// 41 42 43 44 45 46 47 48
// 40 39 38 37 36 35 34 33
// 25 26 27 28 29 30 31 32
// 24 23 22 21 20 19 18 17
//  9 10 11 12 13 14 15 16
//  8  7  6  5  4  3  2  1

// 重排序实现
void ReorderBlock(const uint8_t* src, uint8_t* dst, 
                  int blockX, int blockY, int blockWidth, int blockHeight) {
    int wp = 0;
    bool isOddBlock = (blockX & 1);
    
    for (int yy = 0; yy < blockHeight; yy++) {
        int ofs;
        if (!isOddBlock) {
            // 偶数块：从上往下
            ofs = yy * blockWidth;
        } else {
            // 奇数块：从下往上
            ofs = (blockHeight - yy - 1) * blockWidth;
        }
        
        // 确定扫描方向
        bool reverse;
        if (!(blockHeight & 1)) {
            // 块高度为偶数
            reverse = ((yy & 1) ^ isOddBlock) ? true : false;
        } else {
            if (isOddBlock) {
                reverse = (yy & 1);
            } else {
                reverse = ((yy & 1) ^ isOddBlock) ? true : false;
            }
        }
        
        if (!reverse) {
            // 正向扫描
            for (int xx = 0; xx < blockWidth; xx++) {
                dst[wp++] = src[ofs + xx];
            }
        } else {
            // 反向扫描
            for (int xx = blockWidth - 1; xx >= 0; xx--) {
                dst[wp++] = src[ofs + xx];
            }
        }
    }
}
```

### 3.5 色彩相关滤波器

TLG6 使用 40 种色彩滤波器去除 RGB 通道间的相关性。

```cpp
// 文件路径: cpp/core/visual/SaveTLG6.cpp 第 721-908 行
// 16 种基本滤波器 + 扩展滤波器

void ApplyColorFilter(char* bufb, char* bufg, char* bufr, int len, int code) {
    switch (code) {
        case 0:  // 无滤波
            break;
            
        case 1:  // R -= G, B -= G
            for (int d = 0; d < len; d++) {
                bufr[d] -= bufg[d];
                bufb[d] -= bufg[d];
            }
            break;
            
        case 2:  // R -= G, G -= B
            for (int d = 0; d < len; d++) {
                bufr[d] -= bufg[d];
                bufg[d] -= bufb[d];
            }
            break;
            
        case 3:  // B -= G, G -= R
            for (int d = 0; d < len; d++) {
                bufb[d] -= bufg[d];
                bufg[d] -= bufr[d];
            }
            break;
            
        case 4:  // R -= G, G -= B, B -= R（循环依赖）
            for (int d = 0; d < len; d++) {
                bufr[d] -= bufg[d];
                bufg[d] -= bufb[d];
                bufb[d] -= bufr[d];
            }
            break;
            
        // ... 更多滤波器模式
        
        case 15: // R -= B<<1, G -= B<<1
            for (int d = 0; d < len; d++) {
                unsigned char t = bufb[d] << 1;
                bufr[d] -= t;
                bufg[d] -= t;
            }
            break;
            
        // case 16-39: 扩展滤波器
    }
}

// 选择最优滤波器
int DetectColorFilter(char* b, char* g, char* r, int size, int& outsize) {
    int minbits = -1;
    int mincode = -1;
    
    char bbuf[H_BLOCK_SIZE * W_BLOCK_SIZE];
    char gbuf[H_BLOCK_SIZE * W_BLOCK_SIZE];
    char rbuf[H_BLOCK_SIZE * W_BLOCK_SIZE];
    TryCompressGolomb bc, gc, rc;
    
    // 尝试前 16 种滤波器
    for (int code = 0; code < 16; code++) {
        // 复制到临时缓冲区
        memcpy(bbuf, b, sizeof(char) * size);
        memcpy(gbuf, g, sizeof(char) * size);
        memcpy(rbuf, r, sizeof(char) * size);
        
        // 重置压缩器
        bc.Reset();
        gc.Reset();
        rc.Reset();
        
        // 应用滤波器
        ApplyColorFilter(bbuf, gbuf, rbuf, size, code);
        
        // 估算压缩后比特数
        int bits;
        bits = bc.Try(bbuf, size), bc.Flush();
        if (minbits != -1 && minbits < bits) continue;
        
        bits += gc.Try(gbuf, size), gc.Flush();
        if (minbits != -1 && minbits < bits) continue;
        
        bits += rc.Try(rbuf, size), rc.Flush();
        
        if (minbits == -1 || minbits > bits) {
            minbits = bits;
            mincode = code;
        }
    }
    
    outsize = minbits;
    return mincode;
}
```

### 3.6 Golomb-Rice 熵编码

Golomb-Rice 编码是一种自适应熵编码方法，适合编码几何分布的整数。

```cpp
// Golomb-Rice 编码原理
// 对于非零值 e：
// 1. 转换为非负整数: m = ((e >= 0) ? 2*e : -2*e-1) - 1
// 2. 预测比特长度 k（从查找表获取）
// 3. 输出 (m >> k) 个 0
// 4. 输出 1（终止符）
// 5. 输出 m 的低 k 位

// 例: e=66, k=5
// m = 2*66 - 1 = 131
// m >> 5 = 4, 输出 4 个 0: 0000
// 输出终止符 1: 10000
// m 的低 5 位 = 131 & 31 = 3 (00011)
// 最终: 00011 1 0000 (共 10 位)

// Golomb 比特长度查找表
extern "C" char TVPTLG6GolombBitLengthTable[TVP_TLG6_GOLOMB_N_COUNT * 2 * 128]
                                           [TVP_TLG6_GOLOMB_N_COUNT];

// Golomb 编码器
class TLG6BitStream {
    int BufferBitPos;
    long BufferBytePos;
    tTJSBinaryStream* OutStream;
    unsigned char* Buffer;
    long BufferCapacity;
    
public:
    // 输出单个比特
    void Put1Bit(bool b) {
        if (BufferBytePos == BufferCapacity) {
            // 扩展缓冲区
            long org_cap = BufferCapacity;
            BufferCapacity += 0x1000;
            Buffer = (unsigned char*)TJS_realloc(Buffer, BufferCapacity);
            memset(Buffer + org_cap, 0, BufferCapacity - org_cap);
        }
        
        if (b) {
            Buffer[BufferBytePos] |= 1 << BufferBitPos;
        }
        
        BufferBitPos++;
        if (BufferBitPos == 8) {
            BufferBitPos = 0;
            BufferBytePos++;
        }
    }
    
    // 输出 Gamma 编码（用于游程长度）
    void PutGamma(int v) {
        // Gamma 编码: 值 v 编码为 ceil(log2(v)) 个 0 + 1 + v 的低位
        int t = v;
        t >>= 1;
        int cnt = 0;
        while (t) {
            Put1Bit(0);
            t >>= 1;
            cnt++;
        }
        Put1Bit(1);
        while (cnt--) {
            Put1Bit(v & 1);
            v >>= 1;
        }
    }
    
    // 输出固定长度值
    void PutValue(long v, int len) {
        while (len--) {
            Put1Bit(v & 1);
            v >>= 1;
        }
    }
    
    // 获取 Gamma 编码长度（静态方法）
    static int GetGammaBitLength(int v) {
        if (v <= 1) return 1;
        if (v <= 3) return 3;
        if (v <= 7) return 5;
        if (v <= 15) return 7;
        if (v <= 31) return 9;
        if (v <= 63) return 11;
        if (v <= 127) return 13;
        if (v <= 255) return 15;
        if (v <= 511) return 17;
        return GetGammaBitLengthGeneric(v);
    }
};
```

### 3.7 Golomb-Rice 解码实现

```cpp
// 文件路径: cpp/core/visual/tvpgl.h 第 868-873 行

TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPTLG6DecodeGolombValuesForFirst,
    (tjs_int8* pixelbuf, tjs_int pixel_count, tjs_uint8* bit_pool));

TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPTLG6DecodeGolombValues,
    (tjs_int8* pixelbuf, tjs_int pixel_count, tjs_uint8* bit_pool));

// Golomb 解码核心
void TVPTLG6DecodeGolombValues_c(
    tjs_int8* pixelbuf,      // 输出像素缓冲区
    tjs_int pixel_count,      // 像素数量
    tjs_uint8* bit_pool       // 比特流输入
) {
    int n = TVP_TLG6_GOLOMB_N_COUNT - 1;  // 计数器
    int a = 0;                             // 累积绝对值
    
    int bit_pos = 0;
    int byte_pos = 0;
    
    for (int i = 0; i < pixel_count; i++) {
        // 1. 读取 k 值（从查找表）
        int k = TVPTLG6GolombBitLengthTable[a][n];
        
        // 2. 计数前导零
        int zero_count = 0;
        while (!(bit_pool[byte_pos] & (1 << bit_pos))) {
            zero_count++;
            bit_pos++;
            if (bit_pos == 8) {
                bit_pos = 0;
                byte_pos++;
            }
        }
        // 跳过终止符 '1'
        bit_pos++;
        if (bit_pos == 8) {
            bit_pos = 0;
            byte_pos++;
        }
        
        // 3. 读取低 k 位
        int m = 0;
        for (int b = 0; b < k; b++) {
            if (bit_pool[byte_pos] & (1 << bit_pos)) {
                m |= (1 << b);
            }
            bit_pos++;
            if (bit_pos == 8) {
                bit_pos = 0;
                byte_pos++;
            }
        }
        m |= (zero_count << k);
        
        // 4. 反向转换: e = ((m & 1) ? (m+1)/2 : -(m/2+1)) + 1
        int e;
        if (m & 1) {
            e = ((m >> 1) + 1);
        } else {
            e = -((m >> 1) + 1);
        }
        
        pixelbuf[i] = (tjs_int8)e;
        
        // 5. 更新累积值
        a += (m >> 1);
        if (--n < 0) {
            a >>= 1;
            n = TVP_TLG6_GOLOMB_N_COUNT - 1;
        }
    }
}
```

### 3.8 TLG6 行解码与滤波器应用

```cpp
// 文件路径: cpp/core/visual/tvpgl.h 第 874-886 行

TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPTLG6DecodeLineGeneric,
    (tjs_uint32* prevline, tjs_uint32* curline, tjs_int width,
     tjs_int start_block, tjs_int block_limit, tjs_uint8* filtertypes,
     tjs_int skipblockbytes, tjs_uint32* in, tjs_uint32 initialp,
     tjs_int oddskip, tjs_int dir));

TVP_GL_FUNC_PTR_EXTERN_DECL(void, TVPTLG6DecodeLine,
    (tjs_uint32* prevline, tjs_uint32* curline, tjs_int width,
     tjs_int block_count, tjs_uint8* filtertypes, tjs_int skipblockbytes,
     tjs_uint32* in, tjs_uint32 initialp, tjs_int oddskip, tjs_int dir));

// 解码并应用滤波器
void TVPTLG6DecodeLine_c(
    tjs_uint32* prevline,     // 上一行像素
    tjs_uint32* curline,      // 当前行输出
    tjs_int width,            // 图像宽度
    tjs_int block_count,      // 块数量
    tjs_uint8* filtertypes,   // 滤波器类型数组
    tjs_int skipblockbytes,   // 跳过字节数
    tjs_uint32* in,           // 输入像素缓冲
    tjs_uint32 initialp,      // 初始像素值
    tjs_int oddskip,          // 奇数行跳过
    tjs_int dir               // 方向: 0=正向, 1=反向
) {
    tjs_uint32 p = initialp;
    
    for (tjs_int blk = 0; blk < block_count; blk++) {
        // 获取当前块的滤波器类型
        int ft = filtertypes[blk];
        int filter = ft >> 1;     // 滤波器索引 (0-15)
        int method = ft & 1;      // 预测方法: 0=MED, 1=Average
        
        // 处理块内的每个像素
        for (tjs_int i = 0; i < TVP_TLG6_W_BLOCK_SIZE; i++) {
            // 解包预测误差
            tjs_int32 r_diff = (tjs_int8)((*in) & 0xFF);
            tjs_int32 g_diff = (tjs_int8)(((*in) >> 8) & 0xFF);
            tjs_int32 b_diff = (tjs_int8)(((*in) >> 16) & 0xFF);
            tjs_int32 a_diff = (tjs_int8)(((*in) >> 24) & 0xFF);
            in++;
            
            // 逆向应用滤波器
            ApplyInverseColorFilter(&b_diff, &g_diff, &r_diff, filter);
            
            // 应用预测重建
            tjs_uint32 prev = (blk > 0 || i > 0) ? curline[-1] : initialp;
            tjs_uint32 upper = prevline ? *prevline : initialp;
            
            tjs_uint8 pr, pg, pb, pa;
            if (method == 0) {
                // MED 预测
                // ... MED 逆向计算
            } else {
                // 平均法预测
                pr = ((prev & 0xFF) + (upper & 0xFF) + 1) >> 1;
                pg = (((prev >> 8) & 0xFF) + ((upper >> 8) & 0xFF) + 1) >> 1;
                pb = (((prev >> 16) & 0xFF) + ((upper >> 16) & 0xFF) + 1) >> 1;
                pa = (((prev >> 24) & 0xFF) + ((upper >> 24) & 0xFF) + 1) >> 1;
            }
            
            // 重建像素
            *curline++ = ((pr + r_diff) & 0xFF) |
                        (((pg + g_diff) & 0xFF) << 8) |
                        (((pb + b_diff) & 0xFF) << 16) |
                        (((pa + a_diff) & 0xFF) << 24);
            
            if (prevline) prevline++;
        }
    }
}
```

---

## 4. TLG0.0 SDS 元数据容器

TLG0.0 SDS（Structured Data Stream）是一个容器格式，允许在 TLG5/TLG6 图像数据之外存储元数据。

### 4.1 SDS 容器结构

```cpp
// TLG0.0 SDS 容器
// ┌─────────────────────────────────────────────────┐
// │ Magic: "TLG0.0\x00sds\x1a\x00" (11 bytes)       │
// ├─────────────────────────────────────────────────┤
// │ Raw Data Size (4 bytes, LE)                     │
// ├─────────────────────────────────────────────────┤
// │ Raw TLG Data (TLG5 或 TLG6)                     │
// ├─────────────────────────────────────────────────┤
// │ Chunks...                                       │
// │  ├─ Chunk Name (4 bytes, e.g., "tags")         │
// │  ├─ Chunk Size (4 bytes, LE)                   │
// │  └─ Chunk Data (variable)                      │
// └─────────────────────────────────────────────────┘

// 解析 SDS 容器
void ParseTLG0SDS(tTJSBinaryStream* src, MetaInfoCallback callback) {
    // 读取原始数据大小
    tjs_uint rawlen = src->ReadI32LE();
    
    // 记录当前位置（用于跳转到元数据）
    tjs_uint64 dataStart = src->GetPosition();
    
    // 加载图像数据（调用内部加载函数）
    TVPInternalLoadTLG(...);
    
    // 定位到元数据区域
    src->Seek(dataStart + rawlen, TJS_BS_SEEK_SET);
    
    // 读取 chunks
    while (true) {
        char chunkName[4];
        if (src->Read(chunkName, 4) != 4) break;  // EOF
        
        tjs_uint chunkSize = src->ReadI32LE();
        
        if (memcmp(chunkName, "tags", 4) == 0) {
            // 解析标签数据
            char* tagData = new char[chunkSize + 1];
            src->ReadBuffer(tagData, chunkSize);
            tagData[chunkSize] = 0;
            
            // 标签格式: "4:name=5:value,"
            // 长度:名称=长度:值,
            ParseTags(tagData, callback);
            
            delete[] tagData;
        } else {
            // 跳过未知 chunk
            src->SetPosition(src->GetPosition() + chunkSize);
        }
    }
}

// 标签解析
void ParseTags(const char* tagData, MetaInfoCallback callback) {
    const char* p = tagData;
    const char* end = tagData + strlen(tagData);
    
    while (p < end) {
        // 读取名称长度
        int nameLen = 0;
        while (*p >= '0' && *p <= '9') {
            nameLen = nameLen * 10 + (*p - '0');
            p++;
        }
        if (*p++ != ':') throw std::runtime_error("Malformed tag");
        
        // 读取名称
        std::string name(p, nameLen);
        p += nameLen;
        
        if (*p++ != '=') throw std::runtime_error("Malformed tag");
        
        // 读取值长度
        int valueLen = 0;
        while (*p >= '0' && *p <= '9') {
            valueLen = valueLen * 10 + (*p - '0');
            p++;
        }
        if (*p++ != ':') throw std::runtime_error("Malformed tag");
        
        // 读取值
        std::string value(p, valueLen);
        p += valueLen;
        
        if (*p++ != ',') throw std::runtime_error("Malformed tag");
        
        // 回调通知
        callback(name, value);
    }
}
```

### 4.2 支持的元数据标签

```cpp
// TLG 保存时支持的标签
// 文件路径: cpp/core/visual/SaveTLG6.cpp 第 20-35 行

// 混合模式标签
static const tjs_char* const LAYER_BLEND_MODES[] = {
    TJS_W("opaque"),    TJS_W("alpha"),
    TJS_W("add"),       TJS_W("sub"),
    TJS_W("mul"),       TJS_W("dodge"),
    TJS_W("darken"),    TJS_W("lighten"),
    TJS_W("screen"),    TJS_W("addalpha"),
    TJS_W("psnormal"),  TJS_W("psadd"),
    // ... 更多 Photoshop 风格混合模式
    nullptr
};

// 常用标签:
// - mode: 混合模式 (opaque, alpha, add, ...)
// - offs_x: X 偏移量（像素）
// - offs_y: Y 偏移量（像素）
// - offs_unit: 偏移单位 (pixel)
```

---

## 5. 动手实践

### 5.1 实现简单的 TLG 格式检测器

```cpp
#include <cstdio>
#include <cstring>

enum TLGVersion {
    TLG_UNKNOWN,
    TLG_5,
    TLG_6,
    TLG_0_SDS  // 容器格式
};

struct TLGInfo {
    TLGVersion version;
    int width;
    int height;
    int colors;
    bool hasSDS;
};

TLGInfo DetectTLGFormat(const char* filename) {
    TLGInfo info = {};
    
    FILE* fp = fopen(filename, "rb");
    if (!fp) {
        info.version = TLG_UNKNOWN;
        return info;
    }
    
    // 读取头部
    unsigned char header[15];
    if (fread(header, 1, 15, fp) != 15) {
        fclose(fp);
        info.version = TLG_UNKNOWN;
        return info;
    }
    
    // 检测格式
    if (memcmp(header, "TLG0.0\x00sds\x1a\x00", 11) == 0) {
        info.version = TLG_0_SDS;
        info.hasSDS = true;
        
        // 跳过 raw size，读取内部格式
        fseek(fp, 15, SEEK_SET);  // 11 + 4 = 15
        if (fread(header, 1, 15, fp) != 15) {
            fclose(fp);
            return info;
        }
    }
    
    if (memcmp(header, "TLG5.0\x00raw\x1a\x00", 11) == 0) {
        info.version = TLG_5;
        info.colors = header[11];
        
        // 读取宽高（小端序）
        info.width = header[12] | (header[13] << 8);
        fread(&header[14], 1, 1, fp);  // 读取高度高字节
        unsigned char temp;
        fread(&temp, 1, 1, fp);
        // ... 完整解析需要更多字节
        
    } else if (memcmp(header, "TLG6.0\x00raw\x1a\x00", 11) == 0) {
        info.version = TLG_6;
        info.colors = header[11];
        // 跳过 dataFlags, colorType, externalGolomb
        info.width = header[14] | 
                    (header[15] << 8);  // 需要更多字节
    }
    
    fclose(fp);
    return info;
}

int main(int argc, char* argv[]) {
    if (argc < 2) {
        printf("Usage: %s <tlg_file>\n", argv[0]);
        return 1;
    }
    
    TLGInfo info = DetectTLGFormat(argv[1]);
    
    printf("File: %s\n", argv[1]);
    printf("Format: ");
    switch (info.version) {
        case TLG_5: printf("TLG5.0"); break;
        case TLG_6: printf("TLG6.0"); break;
        case TLG_0_SDS: printf("TLG0.0 SDS Container"); break;
        default: printf("Unknown"); break;
    }
    printf("\n");
    
    if (info.hasSDS) {
        printf("Contains metadata: Yes\n");
    }
    
    printf("Colors: %d (%s)\n", info.colors,
           info.colors == 1 ? "Grayscale" :
           info.colors == 3 ? "RGB" : "RGBA");
    
    return 0;
}
```

### 5.2 TLG5 简化解码示例

```cpp
// 简化的 TLG5 解码器（仅演示核心逻辑）
#include <vector>
#include <cstdint>

class SimpleTLG5Decoder {
public:
    struct Image {
        int width, height, colors;
        std::vector<uint8_t> pixels;  // RGBA 格式
    };
    
    static Image Decode(const uint8_t* data, size_t size) {
        Image img;
        const uint8_t* p = data;
        
        // 验证 Magic
        if (memcmp(p, "TLG5.0\x00raw\x1a\x00", 11) != 0) {
            throw std::runtime_error("Not a valid TLG5 file");
        }
        p += 11;
        
        // 读取头部
        img.colors = *p++;
        img.width = p[0] | (p[1] << 8) | (p[2] << 16) | (p[3] << 24); p += 4;
        img.height = p[0] | (p[1] << 8) | (p[2] << 16) | (p[3] << 24); p += 4;
        int blockHeight = p[0] | (p[1] << 8) | (p[2] << 16) | (p[3] << 24); p += 4;
        
        int blockCount = (img.height - 1) / blockHeight + 1;
        
        // 跳过块大小表
        p += blockCount * 4;
        
        // 分配输出缓冲
        img.pixels.resize(img.width * img.height * 4);
        
        // LZSS 滑动窗口
        std::vector<uint8_t> text(4096, 0);
        int r = 0;
        
        // 通道缓冲
        std::vector<std::vector<uint8_t>> outbuf(img.colors);
        for (int c = 0; c < img.colors; c++) {
            outbuf[c].resize(blockHeight * img.width);
        }
        
        // 按块解码
        for (int y_blk = 0; y_blk < img.height; y_blk += blockHeight) {
            // 解码各通道
            for (int c = 0; c < img.colors; c++) {
                uint8_t mark = *p++;
                int sz = p[0] | (p[1] << 8) | (p[2] << 16) | (p[3] << 24);
                p += 4;
                
                if (mark == 0) {
                    // LZSS 解压
                    r = DecompressLZSS(p, sz, outbuf[c].data(), 
                                       text.data(), r);
                } else {
                    // 原始数据
                    memcpy(outbuf[c].data(), p, sz);
                }
                p += sz;
            }
            
            // 颜色合成
            int y_lim = std::min(y_blk + blockHeight, img.height);
            int offset[4] = {0, 0, 0, 0};
            
            for (int y = y_blk; y < y_lim; y++) {
                uint8_t* row = img.pixels.data() + y * img.width * 4;
                
                // 首行或后续行处理
                ComposeRow(row, y == 0 ? nullptr : row - img.width * 4,
                           outbuf, offset, img.width, img.colors);
                
                for (int c = 0; c < img.colors; c++) {
                    offset[c] += img.width;
                }
            }
        }
        
        return img;
    }
    
private:
    static int DecompressLZSS(const uint8_t* in, int insize,
                              uint8_t* out, uint8_t* text, int r) {
        const uint8_t* inlim = in + insize;
        
        while (in < inlim) {
            uint8_t flags = *in++;
            
            for (int bit = 0; bit < 8 && in < inlim; bit++) {
                if (flags & (1 << bit)) {
                    // 匹配引用
                    int pos = in[0] | ((in[1] & 0x0F) << 8);
                    int len = (in[1] >> 4) + 3;
                    
                    if (len == 18 && in + 2 < inlim) {
                        len = 18 + in[2];
                        in += 3;
                    } else {
                        in += 2;
                    }
                    
                    for (int i = 0; i < len; i++) {
                        uint8_t c = text[pos];
                        pos = (pos + 1) & 0xFFF;
                        *out++ = c;
                        text[r] = c;
                        r = (r + 1) & 0xFFF;
                    }
                } else {
                    // 字面量
                    uint8_t c = *in++;
                    *out++ = c;
                    text[r] = c;
                    r = (r + 1) & 0xFFF;
                }
            }
        }
        return r;
    }
    
    static void ComposeRow(uint8_t* out, const uint8_t* upper,
                           std::vector<std::vector<uint8_t>>& buf,
                           int* offset, int width, int colors) {
        int pr = 0, pg = 0, pb = 0, pa = 0;
        
        for (int x = 0; x < width; x++) {
            int r = buf[0][offset[0] + x];
            int g = buf[1][offset[1] + x];
            int b = buf[2][offset[2] + x];
            int a = (colors == 4) ? buf[3][offset[3] + x] : 0xFF;
            
            // 色彩相关性解码
            b += g;
            r += g;
            
            if (upper) {
                // 使用上行参考
                pb = upper[x * 4 + 0];
                pg = upper[x * 4 + 1];
                pr = upper[x * 4 + 2];
                pa = upper[x * 4 + 3];
            }
            
            // 累积重建
            pb = (pb + b) & 0xFF;
            pg = (pg + g) & 0xFF;
            pr = (pr + r) & 0xFF;
            pa = (pa + a) & 0xFF;
            
            out[x * 4 + 0] = pb;
            out[x * 4 + 1] = pg;
            out[x * 4 + 2] = pr;
            out[x * 4 + 3] = pa;
        }
    }
};
```

---

## 6. 对照项目源码

### 6.1 TLG 加载器入口

```cpp
// 文件路径: cpp/core/visual/LoadTLG.cpp 第 470-599 行

void TVPLoadTLG(void* formatdata, void* callbackdata,
                tTVPGraphicSizeCallback sizecallback,
                tTVPGraphicScanLineCallback scanlinecallback,
                tTVPMetaInfoPushCallback metainfopushcallback,
                tTJSBinaryStream* src, tjs_int keyidx,
                tTVPGraphicLoadMode mode) {
    // 读取 Magic
    unsigned char mark[12];
    src->ReadBuffer(mark, 11);
    
    // 检测 TLG0.0 SDS 容器
    if (!memcmp("TLG0.0\x00sds\x1a\x00", mark, 11)) {
        // 读取原始数据大小
        tjs_uint rawlen = src->ReadI32LE();
        
        // 加载图像
        TVPInternalLoadTLG(...);
        
        // 定位到元数据
        src->Seek(rawlen + 11 + 4, TJS_BS_SEEK_SET);
        
        // 读取 chunks
        // ...
    } else {
        // 直接加载 TLG5/TLG6
        src->Seek(0, TJS_BS_SEEK_SET);
        TVPInternalLoadTLG(...);
    }
}
```

### 6.2 TLG 头部加载

```cpp
// 文件路径: cpp/core/visual/LoadTLG.cpp 第 601-656 行

void TVPLoadHeaderTLG(void* formatdata, tTJSBinaryStream* src,
                      iTJSDispatch2** dic) {
    if (dic == nullptr) return;
    
    unsigned char mark[12];
    src->ReadBuffer(mark, 11);
    
    tjs_int width = 0, height = 0, colors = 0;
    
    // 检测容器
    if (!memcmp("TLG0.0\x00sds\x1a\x00", mark, 11)) {
        tjs_uint rawlen = src->ReadI32LE();
        src->ReadBuffer(mark, 11);
    }
    
    // 解析 TLG5
    if (!memcmp("TLG5.0\x00raw\x1a\x00", mark, 11)) {
        src->ReadBuffer(mark, 1);
        colors = mark[0];
        width = src->ReadI32LE();
        height = src->ReadI32LE();
    }
    // 解析 TLG6
    else if (!memcmp("TLG6.0\x00raw\x1a\x00", mark, 11)) {
        src->ReadBuffer(mark, 4);
        colors = mark[0];
        width = src->ReadI32LE();
        height = src->ReadI32LE();
    }
    
    // 创建字典返回
    *dic = TJSCreateDictionaryObject();
    tTJSVariant val(width);
    (*dic)->PropSet(TJS_MEMBERENSURE, TJS_W("width"), nullptr, &val, *dic);
    val = tTJSVariant(height);
    (*dic)->PropSet(TJS_MEMBERENSURE, TJS_W("height"), nullptr, &val, *dic);
    val = tTJSVariant(colors * 8);  // bpp
    (*dic)->PropSet(TJS_MEMBERENSURE, TJS_W("bpp"), nullptr, &val, *dic);
}
```

### 6.3 TLG 保存入口

```cpp
// 文件路径: cpp/core/visual/SaveTLG6.cpp 第 1305-1421 行

void TVPSaveAsTLG(void* formatdata, tTJSBinaryStream* dst,
                  const iTVPBaseBitmap* image, const ttstr& mode,
                  iTJSDispatch2* meta) {
    // 收集元数据标签
    std::vector<std::string> tags;
    if (meta) {
        // 枚举字典成员
        // ...
    }
    
    // 选择格式
    bool isTLG6 = mode.StartsWith(TJS_W("tlg6"));
    
    // 检查图像有效性
    if (image->GetHeight() == 0 || image->GetWidth() == 0) {
        TVPThrowInternalError;
    }
    
    if (tags.size()) {
        // 写入 TLG0.0 SDS 容器
        dst->WriteBuffer("TLG0.0\x00sds\x1a\x00", 11);
        
        // 占位符：原始数据大小
        tjs_uint rawlenpos = dst->GetPosition();
        dst->WriteBuffer("0000", 4);
        
        // 写入图像数据
        if (isTLG6) {
            SaveTLG6(dst, image, mode == TJS_W("tlg624"));
        } else {
            SaveTLG5(dst, image, mode == TJS_W("tlg524"));
        }
        
        // 回填原始数据大小
        tjs_uint pos_save = dst->GetPosition();
        dst->SetPosition(rawlenpos);
        tjs_uint size = pos_save - rawlenpos - 4;
        WriteInt32(size, dst);
        dst->SetPosition(pos_save);
        
        // 写入 tags chunk
        dst->WriteBuffer("tags", 4);
        // ...
    } else {
        // 直接写入图像
        if (isTLG6) {
            SaveTLG6(dst, image, mode == TJS_W("tlg624"));
        } else {
            SaveTLG5(dst, image, mode == TJS_W("tlg524"));
        }
    }
}
```

---

## 7. 本节小结

- **TLG5** 是 KiriKiri2 的快速解码格式，使用改进型 LZSS 压缩，滑动窗口 4KB，最大匹配长度 273 字节
- **TLG6** 是高压缩率格式，采用 MED 像素预测 + 8×8 块重排序 + 40 种色彩滤波器 + Golomb-Rice 熵编码
- **TLG0.0 SDS** 是容器格式，支持在图像数据外附加元数据（如混合模式、偏移量）
- MED 算法根据局部边缘方向选择预测值，在水平/垂直边缘处表现优异
- Golomb-Rice 编码使用自适应比特长度表，结合零游程压缩实现高效熵编码
- TLG 格式的 SIMD 优化实现位于 `tvpgl.cpp` 中，提供 SSE/NEON 加速

---

## 8. 练习题与答案

### 题目 1：TLG Magic Bytes 识别

编写一个函数，读取文件前 20 字节，返回 TLG 版本号和是否包含 SDS 容器。

<details>
<summary>查看答案</summary>

```cpp
#include <cstdio>
#include <cstring>
#include <string>

struct TLGDetectResult {
    std::string version;      // "TLG5.0", "TLG6.0", "Unknown"
    bool hasSDS;              // 是否有 SDS 容器
    std::string innerVersion; // SDS 内部格式
};

TLGDetectResult DetectTLGVersion(const char* filename) {
    TLGDetectResult result = {"Unknown", false, ""};
    
    FILE* fp = fopen(filename, "rb");
    if (!fp) return result;
    
    unsigned char buf[26];  // 足够读取 SDS + 内部头
    size_t read = fread(buf, 1, 26, fp);
    fclose(fp);
    
    if (read < 11) return result;
    
    // 检查 TLG0.0 SDS
    if (memcmp(buf, "TLG0.0\x00sds\x1a\x00", 11) == 0) {
        result.hasSDS = true;
        
        // 跳过 raw size (4 bytes)
        if (read >= 26) {
            if (memcmp(buf + 15, "TLG5.0\x00raw\x1a\x00", 11) == 0) {
                result.version = "TLG0.0 SDS";
                result.innerVersion = "TLG5.0";
            } else if (memcmp(buf + 15, "TLG6.0\x00raw\x1a\x00", 11) == 0) {
                result.version = "TLG0.0 SDS";
                result.innerVersion = "TLG6.0";
            }
        }
    }
    // 直接 TLG5
    else if (memcmp(buf, "TLG5.0\x00raw\x1a\x00", 11) == 0) {
        result.version = "TLG5.0";
    }
    // 直接 TLG6
    else if (memcmp(buf, "TLG6.0\x00raw\x1a\x00", 11) == 0) {
        result.version = "TLG6.0";
    }
    
    return result;
}

// 使用示例
int main() {
    auto result = DetectTLGVersion("test.tlg");
    
    printf("Version: %s\n", result.version.c_str());
    printf("Has SDS Container: %s\n", result.hasSDS ? "Yes" : "No");
    if (result.hasSDS) {
        printf("Inner Format: %s\n", result.innerVersion.c_str());
    }
    
    return 0;
}
```

</details>

### 题目 2：实现 MED 预测器

实现完整的 MED 预测函数，并编写测试用例验证边缘检测行为。

<details>
<summary>查看答案</summary>

```cpp
#include <cstdint>
#include <cassert>
#include <cstdio>

// MED 预测器
uint8_t MEDPredict(uint8_t a, uint8_t b, uint8_t c) {
    // a: 左像素, b: 上像素, c: 左上像素
    //   +---+---+
    //   | c | b |
    //   +---+---+
    //   | a | x |
    //   +---+---+
    
    uint8_t min_ab = (a < b) ? a : b;
    uint8_t max_ab = (a > b) ? a : b;
    
    if (c >= max_ab) {
        // 水平边缘：c 亮于 a 和 b
        // 预测 x 接近较暗的那个
        return min_ab;
    } else if (c < min_ab) {
        // 垂直边缘：c 暗于 a 和 b
        // 预测 x 接近较亮的那个
        return max_ab;
    } else {
        // 平滑区域：线性预测
        return (uint8_t)((int)a + (int)b - (int)c);
    }
}

// 计算预测误差
int8_t ComputeError(uint8_t actual, uint8_t predicted) {
    return (int8_t)((int)actual - (int)predicted);
}

// 重建像素
uint8_t Reconstruct(int8_t error, uint8_t predicted) {
    return (uint8_t)((int)error + (int)predicted);
}

void TestMED() {
    printf("Testing MED Predictor...\n");
    
    // 测试 1: 水平边缘
    // c=200, b=200
    // a=100, x=?
    // c >= max(a,b) → 返回 min(a,b) = 100
    assert(MEDPredict(100, 200, 200) == 100);
    printf("  Test 1 (horizontal edge): PASS\n");
    
    // 测试 2: 垂直边缘
    // c=50, b=200
    // a=200, x=?
    // c < min(a,b) → 返回 max(a,b) = 200
    assert(MEDPredict(200, 200, 50) == 200);
    printf("  Test 2 (vertical edge): PASS\n");
    
    // 测试 3: 平滑区域
    // c=100, b=110
    // a=105, x=?
    // 线性预测: 105 + 110 - 100 = 115
    assert(MEDPredict(105, 110, 100) == 115);
    printf("  Test 3 (smooth region): PASS\n");
    
    // 测试 4: 完整编解码
    uint8_t actual = 120;
    uint8_t a = 100, b = 110, c = 105;
    uint8_t predicted = MEDPredict(a, b, c);
    int8_t error = ComputeError(actual, predicted);
    uint8_t reconstructed = Reconstruct(error, predicted);
    assert(reconstructed == actual);
    printf("  Test 4 (encode/decode): PASS\n");
    
    printf("All MED tests passed!\n");
}

int main() {
    TestMED();
    return 0;
}
```

</details>

### 题目 3：Golomb-Rice 编解码

实现简化版 Golomb-Rice 编解码器，支持编码 int8_t 范围的有符号整数。

<details>
<summary>查看答案</summary>

```cpp
#include <cstdint>
#include <vector>
#include <cassert>
#include <cstdio>

class GolombRiceCoder {
public:
    // 将有符号整数转换为非负整数
    // e=1 → m=1, e=-1 → m=0, e=2 → m=3, e=-2 → m=2, ...
    static int SignedToUnsigned(int e) {
        if (e > 0) {
            return 2 * e - 1;  // 正数映射到奇数
        } else {
            return -2 * e;     // 非正数映射到偶数
        }
    }
    
    // 逆变换
    static int UnsignedToSigned(int m) {
        if (m & 1) {
            return (m + 1) / 2;   // 奇数 → 正数
        } else {
            return -(m / 2);      // 偶数 → 非正数
        }
    }
    
    // 编码单个值
    static std::vector<bool> Encode(int e, int k) {
        std::vector<bool> bits;
        
        int m = SignedToUnsigned(e);
        
        // 输出 (m >> k) 个 0
        int q = m >> k;
        for (int i = 0; i < q; i++) {
            bits.push_back(false);
        }
        
        // 输出终止符 1
        bits.push_back(true);
        
        // 输出低 k 位（LSB 优先）
        for (int i = 0; i < k; i++) {
            bits.push_back((m >> i) & 1);
        }
        
        return bits;
    }
    
    // 解码
    static int Decode(const std::vector<bool>& bits, int k, int& pos) {
        // 计数前导零
        int q = 0;
        while (pos < bits.size() && !bits[pos]) {
            q++;
            pos++;
        }
        
        // 跳过终止符 1
        pos++;
        
        // 读取低 k 位
        int r = 0;
        for (int i = 0; i < k && pos < bits.size(); i++) {
            if (bits[pos++]) {
                r |= (1 << i);
            }
        }
        
        // 重建 m
        int m = (q << k) | r;
        
        // 转换回有符号整数
        return UnsignedToSigned(m);
    }
};

void TestGolombRice() {
    printf("Testing Golomb-Rice Coder...\n");
    
    // 测试有符号/无符号转换
    assert(GolombRiceCoder::SignedToUnsigned(1) == 1);
    assert(GolombRiceCoder::SignedToUnsigned(-1) == 2);
    assert(GolombRiceCoder::SignedToUnsigned(2) == 3);
    assert(GolombRiceCoder::SignedToUnsigned(-2) == 4);
    assert(GolombRiceCoder::SignedToUnsigned(0) == 0);
    printf("  SignedToUnsigned: PASS\n");
    
    // 测试逆变换
    assert(GolombRiceCoder::UnsignedToSigned(0) == 0);
    assert(GolombRiceCoder::UnsignedToSigned(1) == 1);
    assert(GolombRiceCoder::UnsignedToSigned(2) == -1);
    assert(GolombRiceCoder::UnsignedToSigned(3) == 2);
    assert(GolombRiceCoder::UnsignedToSigned(4) == -2);
    printf("  UnsignedToSigned: PASS\n");
    
    // 测试编解码往返
    for (int e = -10; e <= 10; e++) {
        for (int k = 0; k <= 4; k++) {
            auto bits = GolombRiceCoder::Encode(e, k);
            int pos = 0;
            int decoded = GolombRiceCoder::Decode(bits, k, pos);
            
            if (decoded != e) {
                printf("FAIL: e=%d, k=%d, decoded=%d\n", e, k, decoded);
                assert(false);
            }
        }
    }
    printf("  Round-trip encode/decode: PASS\n");
    
    // 测试特定案例 (e=66, k=5)
    // m = 2*66-1 = 131
    // q = 131 >> 5 = 4
    // r = 131 & 31 = 3 (00011)
    // bits: 0000 1 11000 (4个0, 1, 低5位=3)
    auto bits = GolombRiceCoder::Encode(66, 5);
    printf("  e=66, k=5: %zu bits: ", bits.size());
    for (bool b : bits) printf("%d", b ? 1 : 0);
    printf("\n");
    
    int pos = 0;
    int decoded = GolombRiceCoder::Decode(bits, 5, pos);
    assert(decoded == 66);
    printf("  Specific case (e=66, k=5): PASS\n");
    
    printf("All Golomb-Rice tests passed!\n");
}

int main() {
    TestGolombRice();
    return 0;
}
```

</details>

---

## 9. 下一步

完成本节学习后，你已经掌握了 TLG 原生图像格式的完整技术细节。接下来，我们将进入 OpenGL 渲染后端的学习：

**下一节：** [04-OGL后端/01-RenderManager架构](../04-OGL后端/01-RenderManager架构.md)

在下一节中，你将学习：
- KrKr2 的 OpenGL 渲染管线架构
- RenderManager 抽象类设计
- 纹理管理与批量渲染优化
