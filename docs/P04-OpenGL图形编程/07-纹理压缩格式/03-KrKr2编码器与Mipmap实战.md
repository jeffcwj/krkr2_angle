# KrKr2 编码器与 Mipmap 实战

> **所属模块：** P04-OpenGL 图形编程  
> **前置知识：** [01-GPU纹理压缩全景](./01-GPU纹理压缩全景.md)、[02-压缩纹理加载实战](./02-压缩纹理加载实战.md)  
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 说明 KrKr2 内置的三个实时纹理编码器（PVRTC / ETC / ASTC）各自的输入格式、内存管理和使用限制。
2. 正确上传包含多层 Mipmap 的压缩纹理，每层独立计算 `imageSize`。
3. 编写"运行时编码 + 上传 + 回退"的完整流程代码。
4. 识别 BGRA / RGBA 通道差异导致的颜色反转问题并修复。

---

## 1. KrKr2 实时编码器 API 分析

KrKr2 在 `cpp/core/visual/ogl/` 下内置了三个实时纹理压缩器，用于将 CPU 端像素数据在运行时压缩后再上传 GPU。这与"离线预压缩"是互补的两条路：

### 1.1 PVRTC 编码器

**文件：** `pvrtc.h` / `pvrtc.cpp`（来源：bitbucket jthlim/PvrTcCompressor）

```cpp
// pvrtc.h 公开接口
namespace Javelin {
    class PvrTcEncoder {
    public:
        // 将 RGBA 像素压缩为 PVRTC 4bpp
        // inBuf:     输入 RGBA8 像素数据（宽×高×4 字节）
        // outBuffer: 输出压缩数据缓冲区
        // width, height: 纹理尺寸（必须是 POT，即 2 的幂）
        // isOpaque:  true = 不透明优化路径
        static void EncodeRgba4Bpp(
            const void *inBuf,
            void *outBuffer,
            int width,
            int height,
            bool isOpaque
        );
    };
}
```

**使用要点：**
- 输入必须是 RGBA8 格式的像素数组。
- 宽高必须是 2 的幂（POT），否则 PVRTC 编码器行为未定义。
- 输出缓冲区大小 = `width × height / 2` 字节（4bpp = 每像素 0.5 字节）。
- `isOpaque` 为 true 时编码器跳过 Alpha 处理，质量更高。

### 1.2 ETC 编码器

**文件：** `etcpak.h` / `etcpak.cpp`（来源：wolfpld/etcpak）

```cpp
// etcpak.h 公开接口
class ETCPacker {
public:
    // ETC1 压缩（不带 Alpha）
    // pixel:   输入 RGBA 像素
    // w, h:    尺寸
    // pitch:   每行字节数（通常 w × 4）
    // etc2:    false = ETC1, true = ETC2
    // datalen: [输出] 压缩数据长度
    static void *convert(const void *pixel, int w, int h,
                         int pitch, bool etc2, int &datalen);

    // ETC2 RGBA 压缩（带 Alpha）
    static void *convertWithAlpha(const void *pixel, int w, int h,
                                  int pitch, int &datalen);

    // 解压缩（GPU 不支持时用 CPU 解码回 RGBA）
    static void decode(const void *data, void *pixel,
                       int w, int h);
    static void decodeWithAlpha(const void *data, void *pixel,
                                int w, int h);
};
```

**使用要点：**
- `convert` 和 `convertWithAlpha` 内部分配内存并返回指针，调用方需要 `free()`。
- `pitch` 参数支持行对齐不等于 `w × 4` 的情况（某些图像库的输出带行填充，pitch 即"行跨度"——pitch（行跨度）是指图像数据中一行像素在内存中实际占用的字节数，有时会因为内存对齐而大于 `宽度 × 每像素字节数`）。
- `etc2` 参数控制使用 ETC1 还是 ETC2 模式——同一个函数两种路径。
- `decode` / `decodeWithAlpha` 是软件解码器，当 GPU 不支持 ETC 格式时可以回退为 CPU 解码后用 `glTexImage2D` 上传 RGBA8。

### 1.3 ASTC 编码器

**文件：** `astcrt.h` / `astcrt.cpp`（来源：daoo/astcrt，实时 ASTC 压缩）

```cpp
// astcrt.h 公开接口
class ASTCRealTimeCodec {
public:
    // ASTC 4×4 压缩
    // src:    输入像素数据（注意：BGRA 格式，不是 RGBA！）
    // dst:    输出压缩数据缓冲区
    // width, height: 纹理尺寸
    static void compress_texture_4x4(
        const unsigned char *src,
        unsigned char *dst,
        int width,
        int height
    );
};
```

**使用要点：**
- **输入是 BGRA（Blue-Green-Red-Alpha）而非 RGBA**——这是与其他两个编码器最大的不同。BGRA 是指每个像素的四个字节按 B、G、R、A 顺序排列，而 RGBA 按 R、G、R、A 顺序。如果像素源是 RGBA，必须先做通道交换。
- 只支持 4×4 块（不支持 6×6、8×8 等其他 ASTC 块大小）。
- 输出缓冲区大小 = `ceil(width/4) × ceil(height/4) × 16` 字节。
- "RealTime"意味着它是为运行时设计的轻量级编码器，质量比离线工具（如 `astcenc`）低，但速度更快。

### 1.4 三编码器对比

| 编码器 | 格式 | 输入格式 | 内存管理 | Alpha | POT 要求 |
|--------|------|----------|----------|-------|----------|
| PvrTcEncoder | PVRTC 4bpp | RGBA8 | 调用方分配 | 可选 | 是 |
| ETCPacker | ETC1/ETC2 | RGBA8 | 内部 malloc | convert 无 / convertWithAlpha 有 | 否 |
| ASTCRealTimeCodec | ASTC 4×4 | **BGRA** | 调用方分配 | 自动 | 否 |

---

## 2. Mipmap 链上传

压缩纹理的 Mipmap（多级渐远纹理——GPU 为远处物体准备的缩小版本，使远处物体不会因为采样不足而出现闪烁和锯齿）链上传与普通纹理类似，但每一层都要独立计算 `imageSize`：

```cpp
// 示例 1：上传完整 mipmap 链
#include <vector>
#include <algorithm>
#include <cstdio>

#ifdef __ANDROID__
#include <GLES3/gl3.h>
#else
#include <GL/glew.h>
#endif

struct MipLevel {
    int width;
    int height;
    std::vector<unsigned char> data;  // 该层的压缩数据
};

void UploadCompressedMipChain(
    GLuint texId,
    GLenum internalFormat,
    int blockW, int blockH, int blockBytes,
    const std::vector<MipLevel> &mips)
{
    glBindTexture(GL_TEXTURE_2D, texId);

    for (size_t level = 0; level < mips.size(); ++level) {
        const auto &m = mips[level];

        // 每层独立计算 imageSize
        int bx = std::max(1, (m.width  + blockW - 1) / blockW);
        int by = std::max(1, (m.height + blockH - 1) / blockH);
        GLsizei imageSize = static_cast<GLsizei>(bx * by * blockBytes);

        // 校验数据长度
        if (static_cast<GLsizei>(m.data.size()) != imageSize) {
            std::printf("mip %zu: 数据 %zu 字节 != 预期 %d 字节！\n",
                        level, m.data.size(), imageSize);
            return;  // 数据异常，中止上传
        }

        glCompressedTexImage2D(
            GL_TEXTURE_2D,
            static_cast<GLint>(level),   // mip 层级
            internalFormat,              // 压缩格式
            m.width, m.height,
            0,                           // border
            imageSize,                   // 该层压缩字节数
            m.data.data()
        );

        GLenum err = glGetError();
        if (err != GL_NO_ERROR) {
            std::printf("mip %zu 上传失败: GL error 0x%04X\n", level, err);
            return;
        }
    }

    // 压缩纹理有 mip 链时使用三线性过滤
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER,
                    GL_LINEAR_MIPMAP_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

    std::printf("mip 链上传完成: %zu 层\n", mips.size());
}
```

**常见陷阱：** Mip 层尺寸递减时，最小不能低于 1×1。计算公式：`mipWidth = max(1, baseWidth >> level)`。每层都要重新算 `imageSize`——不能复用上一层的值。

```cpp
// 示例 2：生成 mip 尺寸序列（用于校验离线生成的 mip 数据）
#include <cstdio>
#include <algorithm>

void PrintMipSizes(int baseW, int baseH) {
    int level = 0;
    int w = baseW, h = baseH;
    while (w >= 1 || h >= 1) {
        w = std::max(1, w);
        h = std::max(1, h);
        std::printf("mip %d: %d × %d\n", level, w, h);
        if (w == 1 && h == 1) break;  // 最后一层
        w >>= 1;  // 宽度减半
        h >>= 1;  // 高度减半
        ++level;
    }
}

int main() {
    // 示例：1024×512 的 mip 链
    PrintMipSizes(1024, 512);
    // 输出：
    // mip 0: 1024 × 512
    // mip 1: 512 × 256
    // mip 2: 256 × 128
    // mip 3: 128 × 64
    // mip 4: 64 × 32
    // mip 5: 32 × 16
    // mip 6: 16 × 8
    // mip 7: 8 × 4
    // mip 8: 4 × 2
    // mip 9: 2 × 1
    // mip 10: 1 × 1
    return 0;
}
```

---

## 3. 运行时编码 + 上传完整流程

将 KrKr2 的编码器与上传逻辑串联：

```cpp
// 示例 3：运行时 ETC2 编码并上传
#include <cstdio>
#include <cstdlib>

#ifdef __ANDROID__
#include <GLES3/gl3.h>
#else
#include <GL/glew.h>
#endif

// 假设 etcpak.h 已包含
// class ETCPacker { ... };

// 假设格式检测函数已实现（见上一节）
// bool IsFormatSupported(GLenum fmt);

GLuint CreateCompressedTextureFromRGBA(
    const void *rgbaPixels,    // 输入 RGBA8 像素
    int width, int height,
    bool hasAlpha)
{
    GLuint texId = 0;
    glGenTextures(1, &texId);
    glBindTexture(GL_TEXTURE_2D, texId);

    // 检查 ETC2 支持
    bool supportsETC2 = IsFormatSupported(0x9278);  // ETC2_RGBA8_EAC

    if (supportsETC2 && hasAlpha) {
        // 路径 A：ETC2 RGBA 压缩上传
        int datalen = 0;
        void *compressed = ETCPacker::convertWithAlpha(
            rgbaPixels, width, height,
            width * 4,    // pitch = 每行字节数
            datalen
        );

        glCompressedTexImage2D(
            GL_TEXTURE_2D, 0,
            0x9278,               // GL_COMPRESSED_RGBA8_ETC2_EAC
            width, height, 0,
            datalen,
            compressed
        );

        free(compressed);  // ETCPacker 内部 malloc，调用方需 free

        GLenum err = glGetError();
        if (err != GL_NO_ERROR) {
            // 压缩上传失败，回退到 RGBA8
            std::printf("[TexLoader] ETC2 上传失败 (0x%04X)，回退 RGBA8\n", err);
            glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width, height,
                         0, GL_RGBA, GL_UNSIGNED_BYTE, rgbaPixels);
        }
    } else {
        // 路径 B：直接 RGBA8 上传
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width, height,
                     0, GL_RGBA, GL_UNSIGNED_BYTE, rgbaPixels);
    }

    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

    return texId;
}
```

### 3.1 RGBA ↔ BGRA 通道交换

使用 ASTC 编码器时，必须先把 RGBA 像素转为 BGRA：

```cpp
// 示例 4：RGBA 转 BGRA（原地交换）
#include <cstddef>

void ConvertRGBAtoBGRA(unsigned char *pixels, size_t pixelCount) {
    for (size_t i = 0; i < pixelCount; ++i) {
        unsigned char *p = pixels + i * 4;
        unsigned char tmp = p[0];  // 保存 R
        p[0] = p[2];              // R 位置写入 B
        p[2] = tmp;               // B 位置写入 R
        // p[1] (G) 和 p[3] (A) 不变
    }
}

// 使用示例：
// unsigned char *rgba = LoadImage("texture.png", &w, &h);
// ConvertRGBAtoBGRA(rgba, w * h);
// ASTCRealTimeCodec::compress_texture_4x4(rgba, dst, w, h);
```

> **提示：** Windows 平台上，GDI/GDI+ 的位图数据本身就是 BGRA 格式，可以直接传给 ASTC 编码器而无需转换。这也是 KrKr2 在 Windows 上使用 ASTC 编码器时不做额外转换的原因之一。

---

## 动手实践

**实践目标：** 构建一个跨平台压缩纹理加载模块。

**步骤：**

1. 在引擎初始化阶段（GL 上下文创建后），调用格式检测函数，将结果存入全局集合。
2. 准备四套纹理资源：ASTC 4×4、ETC2 RGBA、BC3、RGBA8（原始回退）。你可以用 `astcenc`、`PVRTexToolCLI`、`texconv` 等离线工具从同一张 PNG 生成。
3. 编写格式选择函数：按 `ASTC → ETC2 → BC3 → RGBA8` 优先级选择。
4. 使用 `glCompressedTexImage2D` 上传压缩数据，每层 mip 独立计算 `imageSize`。
5. 上传后检查 `glGetError()`。若返回非 `GL_NO_ERROR`，回退到 RGBA8 路径。
6. 在控制台输出：选择的格式名称、每层 imageSize、是否触发了回退。
7. 在 Windows、Linux、macOS、Android 分别验证，对比显存占用（可用 GPU 调试工具如 RenderDoc 或 Android GPU Inspector）。

**验收标准：**
- 透明 UI 元素无黑边。
- 远景采样无闪烁（mip 链完整）。
- 显存占用相比纯 RGBA8 明显降低（ASTC 4×4 约 4:1，ETC2 约 4:1）。

---

## 对照项目源码

请重点阅读以下文件，对照本节所学：

| 文件 | 行号范围 | 关注内容 |
|------|----------|----------|
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 193-208 | `TVPInitTextureFormatList()` — 格式集合构建 |
| `cpp/core/visual/ogl/RenderManager_ogl.cpp` | 277-434 | `TVPGetOpenGLInfo()` — 50+ 格式枚举 → 名称映射表 |
| `cpp/core/visual/ogl/pvrtc.h` | 全文 | PVRTC 编码器接口 |
| `cpp/core/visual/ogl/etcpak.h` | 全文 | ETC1/ETC2 编码器接口（含 encode + decode） |
| `cpp/core/visual/ogl/astcrt.h` | 全文 | ASTC 4×4 实时编码器接口（注意：输入 BGRA） |

---

## 常见错误及解决方案

### 错误 1：ASTC 编码器传入 RGBA 数据导致红蓝反转

**症状：** 纹理颜色明显偏蓝或偏红，人物皮肤发青。

**原因：** `ASTCRealTimeCodec::compress_texture_4x4` 期望 BGRA 输入，如果传入 RGBA，R 和 B 通道互换。

**解决：** 在编码前调用 `ConvertRGBAtoBGRA()`（见示例 4）。如果像素源本身就是 BGRA（如 Windows GDI 输出），则无需转换。

### 错误 2：ETCPacker 返回的内存忘记 free

**症状：** 内存持续增长，长时间运行后 OOM（Out of Memory，内存耗尽）崩溃。

**原因：** `ETCPacker::convert()` 和 `convertWithAlpha()` 内部使用 `malloc` 分配内存并返回指针。调用方必须在上传完成后 `free()` 该指针。

**解决：** 上传后立即 `free(compressed)`。建议用 RAII 包装（如 `std::unique_ptr<void, decltype(&free)>`）防止异常路径遗漏。

### 错误 3：Mipmap 层的 imageSize 复用上一层的值

**症状：** 第 2 层及以后的 mip 上传时 `GL_INVALID_VALUE`。

**原因：** 每层的宽高都是上一层的一半，块数和 imageSize 也随之变化，不能复用。

**解决：** 在循环中每层重新计算 `imageSize = ceil(w/blockW) × ceil(h/blockH) × bytesPerBlock`。

---

## 本节小结

1. KrKr2 内置三个实时编码器（PVRTC / ETC / ASTC），各有不同的输入格式要求和内存管理方式——ASTC 编码器要求 BGRA 输入，ETC 编码器内部 malloc 需要调用方 free，PVRTC 要求 POT 尺寸。
2. Mipmap 链上传时，每层的 `imageSize` 必须独立计算——宽高减半后块数也变，不能复用上一层的值。
3. 运行时编码流程：检测格式支持 → 选择编码器 → 编码 → 上传 → 错误时回退 RGBA8。
4. RGBA / BGRA 通道差异是 ASTC 编码最常见的颜色 Bug 来源——Windows GDI 输出天然 BGRA，其他平台通常需要手动交换 R 和 B。

---

## 练习题与答案

### 题目 1：KrKr2 的 ASTC 编码器输入像素格式与另外两个编码器有何不同？这在实际使用中意味着什么？

<details>
<summary>查看答案</summary>

KrKr2 的三个编码器的输入格式：
- `PvrTcEncoder::EncodeRgba4Bpp` — 输入 **RGBA**
- `ETCPacker::convert` / `convertWithAlpha` — 输入 **RGBA**
- `ASTCRealTimeCodec::compress_texture_4x4` — 输入 **BGRA**

ASTC 编码器要求 BGRA 格式，与另外两个不同。

实际使用中的影响：
1. 如果像素源是 RGBA（大多数图像加载库的默认输出），在调用 ASTC 编码器前必须做通道交换（R ↔ B），否则红蓝通道会反转。
2. 交换可以用简单循环实现（见本节示例 4）。
3. Windows 上的位图数据（如 GDI/GDI+ 输出）通常就是 BGRA 格式，这种情况下可以直接传给 ASTC 编码器而不需要转换。

</details>

### 题目 2：为什么 ETCPacker 的 `convert` 返回 `void*` 而不是 `std::vector<uint8_t>`？使用时需要注意什么？

<details>
<summary>查看答案</summary>

ETCPacker 来源于 C 风格的开源库 etcpak，其接口设计遵循 C 惯例：内部使用 `malloc` 分配内存并返回裸指针，数据长度通过引用参数 `datalen` 输出。

使用时需要注意：
1. **必须手动 `free()`**——`convert` 和 `convertWithAlpha` 返回的指针是 `malloc` 分配的，不 free 会内存泄漏。
2. **不要用 `delete`**——`malloc` 分配的内存必须用 `free` 释放，不能用 `delete`（二者来自不同的内存管理体系）。
3. **异常安全**——如果在上传过程中抛出异常，`free` 可能被跳过。建议用 RAII 包装：

```cpp
#include <memory>
// 用 unique_ptr + 自定义删除器确保释放
std::unique_ptr<void, decltype(&free)> compressed(
    ETCPacker::convertWithAlpha(pixels, w, h, w * 4, datalen),
    free  // 释放器
);
// compressed.get() 获取指针
// 离开作用域自动 free
```

</details>

### 题目 3：一张 512×256 的纹理有多少层 mipmap？最小一层的尺寸是多少？

<details>
<summary>查看答案</summary>

从 mip 0（512×256）开始，每层宽高各减半：

| 层级 | 宽 | 高 |
|------|----|----|
| 0 | 512 | 256 |
| 1 | 256 | 128 |
| 2 | 128 | 64 |
| 3 | 64 | 32 |
| 4 | 32 | 16 |
| 5 | 16 | 8 |
| 6 | 8 | 4 |
| 7 | 4 | 2 |
| 8 | 2 | 1 |
| 9 | 1 | 1 |

共 **10 层**（mip 0 到 mip 9）。最小一层是 **1×1**。

注意：当宽高不等时，较大的一边先到 1，较小的一边已经是 1 后保持不变。例如 mip 8 是 2×1，再减半时 `max(1, 2>>1) = 1`，`max(1, 1>>1) = 1`，所以 mip 9 = 1×1。

</details>

---

## 下一步

下一章：[08-实战-实现一个2D渲染器](../08-实战-实现一个2D渲染器/)

在掌握了纹理压缩的格式选择与加载后，下一章我们将综合运用前面所学的全部 OpenGL 知识（缓冲对象、着色器、纹理、帧缓冲、压缩格式），动手搭建一个完整的 2D 批量渲染器，对照 KrKr2 的 `RenderManager_ogl.cpp` 实现。
