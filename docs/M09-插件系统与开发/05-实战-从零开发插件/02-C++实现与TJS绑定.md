# C++ 实现与 TJS 绑定

> **所属模块：** M09-插件系统与开发
> **前置知识：** [需求分析与设计](./01-需求分析与设计.md)、[类型系统与转换](../02-ncbind绑定框架/01-类型系统与转换.md)、[类注册与宏体系](../02-ncbind绑定框架/02-类注册与宏体系.md)
> **预计阅读时间：** 40 分钟

## 本节目标

读完本节后，你将能够：
1. 编写完整的 TextMetrics C++ 类，包含平台抽象层接口和 Windows/FreeType/CoreText 三套实现
2. 使用 ncbind 宏完成类注册、构造函数绑定、方法绑定和属性绑定
3. 正确使用 `ncbDictionaryAccessor` 构建 TJS 字典返回值
4. 正确使用 `ncbPropAccessor` 构建 TJS 数组返回值
5. 处理跨平台字体查找、Unicode 文本度量等实际问题

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| FreeType | FreeType Library | 开源字体渲染引擎，可加载 TTF/OTF 字体并计算字形度量 |
| Core Text | Apple Core Text | macOS/iOS 的底层文字排版引擎，提供字体度量和文字布局 API |
| HFONT | GDI Font Handle | Windows GDI 中代表字体的不透明句柄，通过 `CreateFont` 创建 |
| FT_Face | FreeType Face | FreeType 中代表一个字体文件的数据结构，包含所有字形信息 |
| CTFont | Core Text Font | Core Text 中的字体对象，封装了字体度量和字形数据 |
| advance | Glyph Advance | 字形推进宽度——渲染一个字符后光标应移动的水平距离 |
| baseline | Text Baseline | 文字基线——字母 "a" 底部所在的水平线，是排版的基准参考线 |

---

## 一、TextMetrics 核心类实现

### 1.1 头文件声明

```cpp
// 源码路径：cpp/plugins/textMetrics/TextMetrics.h
#pragma once

#include <memory>
#include <string>
#include <vector>
#include "TextMetricsImpl.h"  // 平台抽象接口

// 前向声明 TJS 类型（避免 include 整个 TJS 头文件）
class tTJSVariant;
class ttstr;
struct iTJSDispatch2;

// 字体样式常量
enum TextMetricsStyle {
    STYLE_NORMAL    = 0,
    STYLE_BOLD      = 1,
    STYLE_ITALIC    = 2,
    STYLE_UNDERLINE = 4  // 可以位或组合：STYLE_BOLD | STYLE_ITALIC
};

class TextMetrics {
private:
    // 平台实现（通过抽象接口隔离平台差异）
    std::unique_ptr<TextMetricsImpl> impl_;

    // 当前字体参数（缓存，用于属性访问）
    std::u16string fontName_;
    int fontSize_;
    int fontStyle_;

    // 字间距（像素）
    float letterSpacing_ = 0.0f;

public:
    // 构造函数——ncbind 通过 Constructor<> 调用
    TextMetrics(const ttstr &fontName, int fontSize, int fontStyle);
    ~TextMetrics();

    // ── 度量方法 ──
    float measureWidth(const ttstr &text);
    float measureHeight(const ttstr &text);
    tTJSVariant measureText(const ttstr &text);
    tTJSVariant wrapText(const ttstr &text, int maxWidth);

    // ── 字体管理 ──
    void setFont(const ttstr &name, int size, int style);

    // ── 属性 getter ──
    ttstr getFontName() const;
    int getFontSize() const;
};
```

### 1.2 构造函数与析构函数

```cpp
// 源码路径：cpp/plugins/textMetrics/TextMetrics.cpp
#include "TextMetrics.h"
#include "TextMetricsImpl.h"
#include <spdlog/spdlog.h>

// ncbind 头文件——提供 ttstr、tTJSVariant 等类型
#include "ncbind.hpp"

TextMetrics::TextMetrics(
    const ttstr &fontName, int fontSize, int fontStyle
) : fontSize_(fontSize), fontStyle_(fontStyle) {
    // 保存字体名（ttstr 转 std::u16string）
    fontName_ = fontName.IsEmpty()
        ? u""
        : std::u16string(fontName.c_str(), fontName.GetLen());

    // 创建平台实现（工厂方法根据编译平台返回具体实现）
    impl_ = TextMetricsImpl::create();

    // 尝试设置指定字体
    if (!impl_->setFont(fontName_, fontSize_, fontStyle_)) {
        // 字体不存在→回退到平台默认字体
        spdlog::warn(
            "TextMetrics: font '{}' not found, using default",
            std::string(fontName_.begin(), fontName_.end())
        );
        fontName_ = impl_->getDefaultFontName();
        impl_->setFont(fontName_, fontSize_, fontStyle_);
    }

    spdlog::info(
        "TextMetrics: initialized with font '{}' size {}",
        std::string(fontName_.begin(), fontName_.end()),
        fontSize_
    );
}

TextMetrics::~TextMetrics() {
    // impl_ 的 unique_ptr 自动析构平台资源
    spdlog::debug("TextMetrics: destroyed");
}
```

### 1.3 度量方法实现

```cpp
// ── measureWidth：最常用的方法，性能优先 ──
float TextMetrics::measureWidth(const ttstr &text) {
    if (text.IsEmpty()) return 0.0f;

    std::u16string str(text.c_str(), text.GetLen());
    float width = impl_->measureWidth(str);

    // 加上字间距
    if (letterSpacing_ != 0.0f && str.size() > 1) {
        width += letterSpacing_ * (float)(str.size() - 1);
    }

    return width;
}

// ── measureHeight：返回单行文字高度 ──
float TextMetrics::measureHeight(const ttstr &text) {
    if (text.IsEmpty()) return 0.0f;

    std::u16string str(text.c_str(), text.GetLen());
    FontMetrics m = impl_->measure(str);
    return m.height;
}

// ── measureText：返回完整度量信息的 TJS 字典 ──
tTJSVariant TextMetrics::measureText(const ttstr &text) {
    // 获取平台度量
    std::u16string str = text.IsEmpty()
        ? u" "  // 空字符串也测量一个空格，获取行高信息
        : std::u16string(text.c_str(), text.GetLen());

    FontMetrics m = impl_->measure(str);

    // 如果原文是空的，宽度应为 0
    if (text.IsEmpty()) {
        m.width = 0.0f;
    }

    // 加上字间距
    if (letterSpacing_ != 0.0f && str.size() > 1 && !text.IsEmpty()) {
        m.width += letterSpacing_ * (float)(str.size() - 1);
    }

    // 构建 TJS 字典
    ncbDictionaryAccessor dict;
    dict.SetValue(TJS_W("width"),   (double)m.width);
    dict.SetValue(TJS_W("height"),  (double)m.height);
    dict.SetValue(TJS_W("ascent"),  (double)m.ascent);
    dict.SetValue(TJS_W("descent"), (double)m.descent);
    dict.SetValue(TJS_W("leading"), (double)m.leading);

    // 返回 TJS Variant（包装字典对象）
    tTJSVariant result;
    result = dict.GetDispatch();
    return result;
}
```

### 1.4 wrapText 自动换行实现

```cpp
// ── wrapText：文字自动换行 ──
tTJSVariant TextMetrics::wrapText(const ttstr &text, int maxWidth) {
    std::u16string str(text.c_str(), text.GetLen());
    std::vector<std::u16string> lines;

    std::u16string currentLine;
    float currentWidth = 0.0f;
    float maxW = (float)maxWidth;

    size_t i = 0;
    while (i < str.size()) {
        // 处理手动换行符
        if (str[i] == u'\n') {
            lines.push_back(currentLine);
            currentLine.clear();
            currentWidth = 0.0f;
            i++;
            continue;
        }

        // 判断是否 CJK 字符（可在任意位置断行）
        char16_t ch = str[i];
        bool isCJK = (ch >= 0x4E00 && ch <= 0x9FFF)     // CJK 统一表意文字
                  || (ch >= 0x3000 && ch <= 0x303F)      // CJK 标点
                  || (ch >= 0x3040 && ch <= 0x309F)      // 平假名
                  || (ch >= 0x30A0 && ch <= 0x30FF)      // 片假名
                  || (ch >= 0xFF00 && ch <= 0xFFEF)      // 全角字符
                  || (ch >= 0xAC00 && ch <= 0xD7AF);     // 韩文音节

        if (isCJK) {
            // CJK：逐字符处理
            std::u16string charStr(1, ch);
            float charW = impl_->measureWidth(charStr);
            if (letterSpacing_ != 0.0f && !currentLine.empty()) {
                charW += letterSpacing_;
            }
            if (currentWidth + charW > maxW && !currentLine.empty()) {
                lines.push_back(currentLine);
                currentLine.clear();
                currentWidth = 0.0f;
            }
            currentLine += ch;
            currentWidth += charW;
            i++;
        } else {
            // 非 CJK：提取整个单词
            std::u16string word;
            size_t wordStart = i;
            while (i < str.size() && str[i] != u'\n'
                   && str[i] != u' '
                   && !((str[i] >= 0x4E00 && str[i] <= 0x9FFF)
                     || (str[i] >= 0x3000 && str[i] <= 0x30FF))) {
                word += str[i++];
            }
            // 包含尾随空格
            while (i < str.size() && str[i] == u' ') {
                word += str[i++];
            }

            float wordW = impl_->measureWidth(word);
            if (letterSpacing_ != 0.0f && !word.empty()) {
                wordW += letterSpacing_ * (float)(word.size() - 1);
                if (!currentLine.empty()) {
                    wordW += letterSpacing_;
                }
            }
            if (currentWidth + wordW > maxW && !currentLine.empty()) {
                lines.push_back(currentLine);
                currentLine.clear();
                currentWidth = 0.0f;
            }
            currentLine += word;
            currentWidth += wordW;
        }
    }
    // 最后一行
    if (!currentLine.empty()) {
        lines.push_back(currentLine);
    }

    // 构建 TJS 数组返回值
    ncbPropAccessor arr;  // 作为 Array 使用
    for (size_t idx = 0; idx < lines.size(); idx++) {
        ttstr line(lines[idx].c_str());
        arr.SetValue((tjs_int)idx, tTJSVariant(line));
    }

    tTJSVariant result;
    result = arr.GetDispatch();
    return result;
}
```

### 1.5 字体管理与属性

```cpp
// ── setFont：切换字体 ──
void TextMetrics::setFont(
    const ttstr &name, int size, int style
) {
    fontName_ = std::u16string(name.c_str(), name.GetLen());
    fontSize_ = size;
    fontStyle_ = style;

    if (!impl_->setFont(fontName_, fontSize_, fontStyle_)) {
        spdlog::warn(
            "TextMetrics: font '{}' not found, keeping current",
            std::string(fontName_.begin(), fontName_.end())
        );
        // 保持当前字体不变
        return;
    }
}

// ── 属性 getter ──
ttstr TextMetrics::getFontName() const {
    return ttstr(fontName_.c_str());
}

int TextMetrics::getFontSize() const {
    return fontSize_;
}
```

---

## 二、平台抽象接口

### 2.1 接口定义

```cpp
// 源码路径：cpp/plugins/textMetrics/TextMetricsImpl.h
#pragma once

#include <memory>
#include <string>

// 文字度量结果结构
struct FontMetrics {
    float width   = 0.0f;  // 文字总宽度（像素）
    float height  = 0.0f;  // 文字总高度：ascent + descent + leading
    float ascent  = 0.0f;  // 基线以上高度
    float descent = 0.0f;  // 基线以下高度
    float leading = 0.0f;  // 行间距额外空间
};

// 平台抽象接口
class TextMetricsImpl {
public:
    virtual ~TextMetricsImpl() = default;

    // 设置字体，返回是否成功
    virtual bool setFont(
        const std::u16string &name,
        int size, int style) = 0;

    // 获取完整度量信息
    virtual FontMetrics measure(
        const std::u16string &text) = 0;

    // 获取文字宽度（性能优化版）
    virtual float measureWidth(
        const std::u16string &text) = 0;

    // 获取平台默认字体名
    virtual std::u16string getDefaultFontName() = 0;

    // 工厂方法
    static std::unique_ptr<TextMetricsImpl> create();
};
```

### 2.2 工厂方法——条件编译入口

```cpp
// 源码路径：cpp/plugins/textMetrics/TextMetricsImpl.cpp
// 这个文件只有工厂方法，使用条件编译选择具体实现

#include "TextMetricsImpl.h"

#if defined(_WIN32)
  #include "TextMetricsImpl_Win.h"
#elif defined(__APPLE__)
  #include "TextMetricsImpl_CT.h"
#else
  #include "TextMetricsImpl_FT.h"
#endif

std::unique_ptr<TextMetricsImpl> TextMetricsImpl::create() {
#if defined(_WIN32)
    return std::make_unique<TextMetricsImplWin>();
#elif defined(__APPLE__)
    return std::make_unique<TextMetricsImplCT>();
#else
    return std::make_unique<TextMetricsImplFT>();
#endif
}
```

---

## 三、Windows 平台实现（GDI）

### 3.1 GDI 文字度量 API

Windows 的 GDI（Graphics Device Interface，图形设备接口）提供了一套简单直接的文字度量 API：

| API | 功能 | 返回值 |
|-----|------|--------|
| `CreateFontW` | 创建字体对象 | `HFONT`（字体句柄） |
| `GetTextExtentPoint32W` | 计算文字像素尺寸 | `SIZE`（宽度 + 高度） |
| `GetTextMetricsW` | 获取字体度量信息 | `TEXTMETRICW`（ascent、descent 等） |
| `SelectObject` | 将字体选入设备上下文 | 旧对象句柄 |
| `DeleteObject` | 释放字体句柄 | `BOOL` |

### 3.2 完整实现

```cpp
// 源码路径：cpp/plugins/textMetrics/TextMetricsImpl_Win.cpp
#include "TextMetricsImpl.h"
#include <windows.h>     // GDI 函数
#include <spdlog/spdlog.h>

class TextMetricsImplWin : public TextMetricsImpl {
private:
    HDC hdc_;       // 设备上下文——GDI 绘图操作的"画布"
    HFONT hfont_;   // 当前字体句柄
    HFONT oldFont_; // 保存的旧字体（用于恢复）

public:
    TextMetricsImplWin() : hfont_(nullptr), oldFont_(nullptr) {
        // 创建内存 DC（Device Context，设备上下文）
        // 不关联任何窗口，纯粹用于度量计算
        hdc_ = CreateCompatibleDC(nullptr);
        if (!hdc_) {
            spdlog::error("TextMetricsImplWin: CreateCompatibleDC failed");
        }
    }

    ~TextMetricsImplWin() override {
        // 恢复旧字体并释放资源
        if (oldFont_ && hdc_) {
            SelectObject(hdc_, oldFont_);
        }
        if (hfont_) {
            DeleteObject(hfont_);
        }
        if (hdc_) {
            DeleteDC(hdc_);
        }
    }

    bool setFont(const std::u16string &name,
                 int size, int style) override {
        // 释放旧字体
        if (oldFont_ && hdc_) {
            SelectObject(hdc_, oldFont_);
            oldFont_ = nullptr;
        }
        if (hfont_) {
            DeleteObject(hfont_);
            hfont_ = nullptr;
        }

        // 将字号（磅）转换为逻辑单位
        // GDI 使用负值表示"按字符高度"而非"按单元格高度"
        int logHeight = -MulDiv(
            size,
            GetDeviceCaps(hdc_, LOGPIXELSY),  // 屏幕 DPI
            72  // 72 磅 = 1 英寸
        );

        // 创建字体
        hfont_ = CreateFontW(
            logHeight,                          // 高度
            0,                                  // 宽度（0=自动）
            0, 0,                               // 转义角度、方向角度
            (style & 1) ? FW_BOLD : FW_NORMAL,  // 粗细
            (style & 2) ? TRUE : FALSE,          // 斜体
            (style & 4) ? TRUE : FALSE,          // 下划线
            FALSE,                               // 删除线
            DEFAULT_CHARSET,                     // 字符集
            OUT_DEFAULT_PRECIS,                  // 输出精度
            CLIP_DEFAULT_PRECIS,                 // 裁剪精度
            CLEARTYPE_QUALITY,                   // 渲染质量
            DEFAULT_PITCH | FF_DONTCARE,         // 字距和字族
            reinterpret_cast<LPCWSTR>(name.c_str())  // 字体名
        );

        if (!hfont_) {
            spdlog::warn("TextMetricsImplWin: CreateFont failed for '{}'",
                         std::string(name.begin(), name.end()));
            return false;
        }

        // 将字体选入 DC
        oldFont_ = (HFONT)SelectObject(hdc_, hfont_);
        return true;
    }

    FontMetrics measure(const std::u16string &text) override {
        FontMetrics result;
        if (!hfont_ || !hdc_) return result;

        // 获取文字尺寸
        SIZE sz;
        GetTextExtentPoint32W(
            hdc_,
            reinterpret_cast<LPCWSTR>(text.c_str()),
            (int)text.size(),
            &sz
        );
        result.width  = (float)sz.cx;
        result.height = (float)sz.cy;

        // 获取字体度量（ascent、descent、leading）
        TEXTMETRICW tm;
        GetTextMetricsW(hdc_, &tm);
        result.ascent  = (float)tm.tmAscent;
        result.descent = (float)tm.tmDescent;
        result.leading = (float)tm.tmExternalLeading;

        return result;
    }

    float measureWidth(const std::u16string &text) override {
        if (!hfont_ || !hdc_) return 0.0f;

        SIZE sz;
        GetTextExtentPoint32W(
            hdc_,
            reinterpret_cast<LPCWSTR>(text.c_str()),
            (int)text.size(),
            &sz
        );
        return (float)sz.cx;
    }

    std::u16string getDefaultFontName() override {
        // Windows 默认字体
        return u"MS Gothic";
    }
};
```

---

## 四、FreeType 平台实现（Linux/Android）

### 4.1 FreeType 核心概念

FreeType 是一个开源字体渲染库，它的核心 API 围绕以下概念：

- **FT_Library**：FreeType 库全局实例，整个程序生命周期只需一个
- **FT_Face**：代表一个字体文件（如 `.ttf`），包含所有字形（Glyph，即单个字符的视觉形状）数据
- **FT_Load_Char**：加载指定字符的字形数据，包括宽度（advance）和位图
- **字形度量**：`face->glyph->advance.x` 是 advance 宽度（以 1/64 像素为单位）

### 4.2 完整实现

```cpp
// 源码路径：cpp/plugins/textMetrics/TextMetricsImpl_FT.cpp
#include "TextMetricsImpl.h"
#include <ft2build.h>
#include FT_FREETYPE_H    // FreeType 核心 API
#include <spdlog/spdlog.h>
#include <filesystem>

#ifdef __linux__
  #include <fontconfig/fontconfig.h>  // 字体查找
#endif

class TextMetricsImplFT : public TextMetricsImpl {
private:
    FT_Library ftLib_;   // FreeType 库实例
    FT_Face    ftFace_;  // 当前字体
    bool       initialized_ = false;

    // 通过字体名查找字体文件路径
    std::string findFontFile(const std::u16string &name) {
        std::string nameUtf8(name.begin(), name.end());

#ifdef __linux__
        // Linux：使用 Fontconfig 查找字体
        FcConfig *config = FcInitLoadConfigAndFonts();
        FcPattern *pattern = FcNameParse(
            (const FcChar8 *)nameUtf8.c_str()
        );
        FcConfigSubstitute(config, pattern, FcMatchPattern);
        FcDefaultSubstitute(pattern);

        FcResult result;
        FcPattern *match = FcFontMatch(config, pattern, &result);

        std::string fontPath;
        if (match) {
            FcChar8 *file = nullptr;
            if (FcPatternGetString(match, FC_FILE, 0, &file)
                == FcResultMatch) {
                fontPath = (const char *)file;
            }
            FcPatternDestroy(match);
        }
        FcPatternDestroy(pattern);
        FcConfigDestroy(config);

        return fontPath;
#else
        // Android：在系统字体目录中查找
        std::vector<std::string> searchPaths = {
            "/system/fonts/",
            "/system/font/",
            "/data/fonts/"
        };
        // 尝试常见的字体文件名模式
        std::vector<std::string> candidates = {
            nameUtf8 + ".ttf",
            nameUtf8 + ".otf",
            nameUtf8 + ".ttc"
        };
        for (auto &dir : searchPaths) {
            for (auto &file : candidates) {
                std::string path = dir + file;
                if (std::filesystem::exists(path)) {
                    return path;
                }
            }
        }
        return "";
#endif
    }

public:
    TextMetricsImplFT() : ftLib_(nullptr), ftFace_(nullptr) {
        FT_Error err = FT_Init_FreeType(&ftLib_);
        if (err) {
            spdlog::error("FreeType: init failed (error {})", err);
        } else {
            initialized_ = true;
        }
    }

    ~TextMetricsImplFT() override {
        if (ftFace_) FT_Done_Face(ftFace_);
        if (ftLib_)  FT_Done_FreeType(ftLib_);
    }

    bool setFont(const std::u16string &name,
                 int size, int style) override {
        if (!initialized_) return false;

        // 释放旧字体
        if (ftFace_) {
            FT_Done_Face(ftFace_);
            ftFace_ = nullptr;
        }

        // 查找字体文件
        std::string fontPath = findFontFile(name);
        if (fontPath.empty()) {
            spdlog::warn("FreeType: font '{}' not found",
                         std::string(name.begin(), name.end()));
            return false;
        }

        // 加载字体
        FT_Error err = FT_New_Face(ftLib_, fontPath.c_str(),
                                    0, &ftFace_);
        if (err) {
            spdlog::warn("FreeType: failed to load '{}'", fontPath);
            return false;
        }

        // 设置字号（以 1/64 像素为单位）
        // size 是磅值，假设 96 DPI
        FT_Set_Char_Size(ftFace_,
            0,             // 宽度（0=与高度相同）
            size * 64,     // 高度（磅 × 64）
            96, 96         // 水平和垂直 DPI
        );

        return true;
    }

    FontMetrics measure(const std::u16string &text) override {
        FontMetrics result;
        if (!ftFace_) return result;

        float totalWidth = 0.0f;
        for (char16_t ch : text) {
            // 加载字符的字形数据
            FT_Error err = FT_Load_Char(
                ftFace_, (FT_ULong)ch,
                FT_LOAD_DEFAULT  // 只加载度量，不渲染位图
            );
            if (err) continue;

            // advance.x 以 1/64 像素为单位，右移 6 位转换
            totalWidth += (float)(ftFace_->glyph->advance.x >> 6);
        }

        // 字体全局度量（所有字符共享的行高信息）
        result.width   = totalWidth;
        result.ascent  = (float)(ftFace_->size->metrics.ascender >> 6);
        result.descent = (float)(-ftFace_->size->metrics.descender >> 6);
        result.leading = (float)(
            (ftFace_->size->metrics.height >> 6)
            - (ftFace_->size->metrics.ascender >> 6)
            + (ftFace_->size->metrics.descender >> 6)
        );
        result.height  = result.ascent + result.descent + result.leading;

        return result;
    }

    float measureWidth(const std::u16string &text) override {
        if (!ftFace_) return 0.0f;

        float width = 0.0f;
        for (char16_t ch : text) {
            if (FT_Load_Char(ftFace_, (FT_ULong)ch,
                             FT_LOAD_DEFAULT) == 0) {
                width += (float)(ftFace_->glyph->advance.x >> 6);
            }
        }
        return width;
    }

    std::u16string getDefaultFontName() override {
#ifdef __linux__
        return u"sans-serif";  // Fontconfig 会解析为实际字体
#else
        return u"DroidSansFallback";  // Android 默认中日韩字体
#endif
    }
};
```

---

## 五、macOS 平台实现（Core Text）

### 5.1 Core Text 核心概念

Core Text 是 macOS/iOS 的底层文字排版引擎。与 FreeType 逐字符度量不同，Core Text 面向**整行文字**进行排版计算：

- **CTFontRef**：字体对象，类似 FreeType 的 FT_Face
- **CFAttributedStringRef**：带属性的字符串（字体、颜色等属性附加到文本上）
- **CTLineRef**：排版后的一行文字，包含完整的度量信息
- **CTLineGetTypographicBounds**：获取一行文字的 ascent、descent、leading

### 5.2 完整实现

```cpp
// 源码路径：cpp/plugins/textMetrics/TextMetricsImpl_CT.cpp
#include "TextMetricsImpl.h"
#include <CoreText/CoreText.h>
#include <CoreFoundation/CoreFoundation.h>
#include <spdlog/spdlog.h>

class TextMetricsImplCT : public TextMetricsImpl {
private:
    CTFontRef ctFont_ = nullptr;  // Core Text 字体

public:
    ~TextMetricsImplCT() override {
        if (ctFont_) CFRelease(ctFont_);
    }

    bool setFont(const std::u16string &name,
                 int size, int style) override {
        if (ctFont_) {
            CFRelease(ctFont_);
            ctFont_ = nullptr;
        }

        // 创建字体名 CFString
        CFStringRef cfName = CFStringCreateWithCharacters(
            kCFAllocatorDefault,
            reinterpret_cast<const UniChar *>(name.c_str()),
            (CFIndex)name.size()
        );

        // 创建 Core Text 字体
        ctFont_ = CTFontCreateWithName(
            cfName,
            (CGFloat)size,    // 字号
            nullptr           // 变换矩阵（nullptr = 不变换）
        );
        CFRelease(cfName);

        if (!ctFont_) {
            spdlog::warn("CoreText: font creation failed");
            return false;
        }

        // 处理粗体和斜体
        CTFontSymbolicTraits desiredTraits = 0;
        if (style & 1) desiredTraits |= kCTFontBoldTrait;
        if (style & 2) desiredTraits |= kCTFontItalicTrait;

        if (desiredTraits != 0) {
            CTFontRef styledFont = CTFontCreateCopyWithSymbolicTraits(
                ctFont_, 0.0, nullptr,
                desiredTraits, desiredTraits
            );
            if (styledFont) {
                CFRelease(ctFont_);
                ctFont_ = styledFont;
            }
        }

        return true;
    }

    FontMetrics measure(const std::u16string &text) override {
        FontMetrics result;
        if (!ctFont_) return result;

        // 创建带属性的字符串
        CFStringRef cfStr = CFStringCreateWithCharacters(
            kCFAllocatorDefault,
            reinterpret_cast<const UniChar *>(text.c_str()),
            (CFIndex)text.size()
        );

        // 附加字体属性
        CFStringRef keys[] = { kCTFontAttributeName };
        CFTypeRef values[] = { ctFont_ };
        CFDictionaryRef attrs = CFDictionaryCreate(
            kCFAllocatorDefault,
            (const void **)keys,
            (const void **)values,
            1,
            &kCFTypeDictionaryKeyCallBacks,
            &kCFTypeDictionaryValueCallBacks
        );

        CFAttributedStringRef attrStr =
            CFAttributedStringCreate(
                kCFAllocatorDefault, cfStr, attrs
            );
        CFRelease(cfStr);
        CFRelease(attrs);

        // 创建排版行
        CTLineRef line = CTLineCreateWithAttributedString(attrStr);
        CFRelease(attrStr);

        // 获取排版度量
        CGFloat ascent, descent, leading;
        double width = CTLineGetTypographicBounds(
            line, &ascent, &descent, &leading
        );

        result.width   = (float)width;
        result.ascent  = (float)ascent;
        result.descent = (float)descent;
        result.leading = (float)leading;
        result.height  = (float)(ascent + descent + leading);

        CFRelease(line);
        return result;
    }

    float measureWidth(const std::u16string &text) override {
        FontMetrics m = measure(text);  // Core Text 没有单独的宽度 API
        return m.width;
    }

    std::u16string getDefaultFontName() override {
        return u"Hiragino Sans";  // macOS 默认日文字体
    }
};
```

---

## 六、ncbind 注册入口

### 6.1 main.cpp 完整实现

```cpp
// 源码路径：cpp/plugins/textMetrics/main.cpp
// ncbind 注册入口——将 TextMetrics 暴露给 TJS 脚本

#include "ncbind.hpp"      // ncbind 核心头文件
#include "TextMetrics.h"   // TextMetrics 类声明

// 注册 TextMetrics 类
NCB_REGISTER_CLASS(TextMetrics) {
    // ── 样式常量 ──
    // 注册为类级常量，TJS 中通过 TextMetrics.STYLE_BOLD 访问
    Variant(TJS_W("STYLE_NORMAL"),    (tjs_int)0);
    Variant(TJS_W("STYLE_BOLD"),      (tjs_int)1);
    Variant(TJS_W("STYLE_ITALIC"),    (tjs_int)2);
    Variant(TJS_W("STYLE_UNDERLINE"), (tjs_int)4);

    // ── 构造函数 ──
    // TJS: new TextMetrics("MS Gothic", 24, TextMetrics.STYLE_BOLD)
    // 模板参数必须与 C++ 构造函数签名精确匹配
    Constructor<const ttstr&, int, int>(0);

    // ── 度量方法 ──
    NCB_METHOD(measureWidth);    // float measureWidth(ttstr)
    NCB_METHOD(measureHeight);   // float measureHeight(ttstr)
    NCB_METHOD(measureText);     // tTJSVariant measureText(ttstr)
    NCB_METHOD(wrapText);        // tTJSVariant wrapText(ttstr, int)

    // ── 字体管理 ──
    NCB_METHOD(setFont);         // void setFont(ttstr, int, int)

    // ── 只读属性 ──
    // NCB_PROPERTY_RO(name, getter) 等价于
    // Property(TJS_W("name"), &Class::getter, nullptr)
    NCB_PROPERTY_RO(fontName, getFontName);
    NCB_PROPERTY_RO(fontSize, getFontSize);
};
```

### 6.2 注册代码详解

这段注册代码虽然只有 20 行，但包含了 ncbind 的所有核心模式：

| 注册元素 | 宏/方法 | TJS 访问方式 | 说明 |
|---------|---------|-------------|------|
| 常量 | `Variant(name, value)` | `TextMetrics.STYLE_BOLD` | 类级常量 |
| 构造 | `Constructor<types>(0)` | `new TextMetrics(...)` | 自动参数转换 |
| 方法 | `NCB_METHOD(name)` | `obj.measureWidth(...)` | 自动参数/返回值转换 |
| 只读属性 | `NCB_PROPERTY_RO(name, getter)` | `obj.fontName` | getter 返回值自动转换 |

> **注意**：`Constructor<const ttstr&, int, int>(0)` 的 `(0)` 是必需参数个数的**最小值**。传 `0` 表示所有参数都有默认值（ncbind 会使用 C++ 默认参数）。如果传 `3`，则 TJS 调用时必须提供全部 3 个参数。

---

## 动手实践

### 练习 1：添加 letterSpacing 属性

在注册代码中添加可读写的 `letterSpacing` 属性。需要：
1. 在 `TextMetrics` 类中添加 `getLetterSpacing()` 和 `setLetterSpacing()` 方法
2. 在 `NCB_REGISTER_CLASS` 中注册读写属性

### 练习 2：验证 FreeType 的 1/64 像素单位

编写一个测试程序，加载一个 TTF 字体，使用 `FT_Load_Char` 获取字符 'A' 的 advance.x，验证它确实是以 1/64 像素为单位的（即 `advance.x / 64.0` 才是实际像素值）。

---

## 对照项目源码

| 参考文件 | 行号范围 | 关注点 |
|---------|---------|--------|
| `cpp/plugins/psdfile/main.cpp` | 1-50 | NCB_REGISTER_CLASS + Variant 常量——与我们的注册代码结构对比 |
| `cpp/plugins/layerex_draw/general/main.cpp` | 401-500 | NCB_GET_INSTANCE_HOOK——对比我们不需要 Hook 的理由 |
| `cpp/plugins/fstat/main.cpp` | 200-500 | RawCallback 返回 TJS 字典——对比我们用 NCB_METHOD + ncbDictionaryAccessor |
| `cpp/plugins/layerex_draw/general/LayerExDraw.hpp` | 51-200 | GpBitmap 像素共享——对比我们的 TextMetrics 不需要像素操作 |

---

## 常见错误与排查

### 错误 1：FreeType 度量值出现"巨大"数字

**现象**：`measureWidth("A")` 返回 1536 而非预期的 24

**原因**：忘记将 FreeType 的 1/64 像素单位转换。`advance.x` 需要右移 6 位（`>> 6`）或除以 64。

```cpp
// 错误
width += (float)face->glyph->advance.x;  // 1536（1/64 像素）

// 正确
width += (float)(face->glyph->advance.x >> 6);  // 24（像素）
```

### 错误 2：Windows HFONT 创建时字号异常

**现象**：字体看起来比预期大很多或小很多

**原因**：`CreateFontW` 的第一个参数 `nHeight` 的正负号含义不同：
- **正值**：指定字符单元格高度（包含行间距）
- **负值**：指定字符高度（不含行间距，通常是你想要的）

```cpp
// 错误：正值，包含行间距
CreateFontW(24, ...);  // 实际字符高度可能只有 20

// 正确：负值，精确控制字符高度
int logHeight = -MulDiv(size, GetDeviceCaps(hdc, LOGPIXELSY), 72);
CreateFontW(logHeight, ...);
```

### 错误 3：Core Text CFRelease 顺序错误

**现象**：macOS 上随机崩溃，`EXC_BAD_ACCESS`

**原因**：Core Text 对象之间有引用关系，释放顺序不对会导致 use-after-free（使用已释放的内存）。

```cpp
// 错误：先释放 attrStr，但 line 还在引用它
CFRelease(attrStr);
CTLineGetTypographicBounds(line, ...);  // 💥 崩溃
CFRelease(line);

// 正确：先使用 line，再按创建的逆序释放
CTLineGetTypographicBounds(line, ...);
CFRelease(line);       // 最后创建的最先释放
CFRelease(attrStr);    // 先创建的后释放
```

---

## 本节小结

- **TextMetrics 类**包含构造函数、4 个度量方法、1 个字体管理方法、2 个属性 getter
- **平台抽象层**通过纯虚接口 + 工厂方法隔离了三套实现（GDI/FreeType/CoreText）
- **Windows 实现**使用 `CreateFontW` + `GetTextExtentPoint32W` + `GetTextMetricsW`
- **FreeType 实现**逐字符 `FT_Load_Char` 累加 `advance.x >> 6`，Linux 用 Fontconfig 查找字体
- **Core Text 实现**面向整行排版，`CTLineGetTypographicBounds` 一次返回所有度量
- **ncbind 注册**只需 20 行，覆盖常量、构造、方法、属性四种注册类型
- **wrapText** 实现了 CJK/英文混排的智能换行，处理了手动换行符、单字超宽等边界情况

---

## 练习题与答案

### 题目 1：添加 kerning（字偶距）支持

kerning（字偶距）是指特定字符对之间的间距调整。例如 "AV" 之间通常比 "AA" 之间更紧凑。请在 FreeType 实现中添加 kerning 支持。

<details>
<summary>查看答案</summary>

```cpp
// 在 TextMetricsImplFT::measure 中添加 kerning 支持
FontMetrics TextMetricsImplFT::measure(
    const std::u16string &text
) {
    FontMetrics result;
    if (!ftFace_) return result;

    // 检查字体是否有 kerning 表
    bool hasKerning = FT_HAS_KERNING(ftFace_);

    float totalWidth = 0.0f;
    FT_UInt prevIndex = 0;  // 前一个字符的字形索引

    for (char16_t ch : text) {
        // 获取当前字符的字形索引
        FT_UInt glyphIndex = FT_Get_Char_Index(ftFace_, (FT_ULong)ch);

        // 如果有 kerning 且不是第一个字符，计算字偶距
        if (hasKerning && prevIndex && glyphIndex) {
            FT_Vector delta;
            FT_Get_Kerning(
                ftFace_,
                prevIndex,           // 前一个字符
                glyphIndex,          // 当前字符
                FT_KERNING_DEFAULT,  // 以 1/64 像素为单位
                &delta
            );
            totalWidth += (float)(delta.x >> 6);  // 加上 kerning 偏移
        }

        // 加载字形并累加 advance
        if (FT_Load_Glyph(ftFace_, glyphIndex, FT_LOAD_DEFAULT) == 0) {
            totalWidth += (float)(ftFace_->glyph->advance.x >> 6);
        }

        prevIndex = glyphIndex;
    }

    result.width = totalWidth;
    // ascent、descent、leading 不变...
    result.ascent  = (float)(ftFace_->size->metrics.ascender >> 6);
    result.descent = (float)(-ftFace_->size->metrics.descender >> 6);
    result.leading = (float)(
        (ftFace_->size->metrics.height >> 6)
        - result.ascent - result.descent
    );
    result.height = result.ascent + result.descent + result.leading;

    return result;
}
```

关键变化：
- 使用 `FT_Get_Char_Index` 获取字形索引（而非直接用 `FT_Load_Char`）
- `FT_Get_Kerning` 获取两个相邻字符间的偏移量
- `FT_HAS_KERNING` 提前检查，避免对没有 kerning 表的字体做无用计算
- `delta.x` 可以是**负数**（表示字符需要更紧凑），所以用 `+=` 即可

</details>

### 题目 2：为 measureText 添加 charWidths 数组

扩展 `measureText` 的返回值，增加一个 `charWidths` 数组，包含每个字符的独立宽度。这对实现光标定位和文字选择非常有用。

<details>
<summary>查看答案</summary>

```cpp
tTJSVariant TextMetrics::measureText(const ttstr &text) {
    std::u16string str = text.IsEmpty()
        ? u" "
        : std::u16string(text.c_str(), text.GetLen());

    FontMetrics m = impl_->measure(str);

    // 构建字典
    ncbDictionaryAccessor dict;
    dict.SetValue(TJS_W("width"),   (double)m.width);
    dict.SetValue(TJS_W("height"),  (double)m.height);
    dict.SetValue(TJS_W("ascent"),  (double)m.ascent);
    dict.SetValue(TJS_W("descent"), (double)m.descent);
    dict.SetValue(TJS_W("leading"), (double)m.leading);

    // 新增：每个字符的独立宽度数组
    if (!text.IsEmpty()) {
        ncbPropAccessor widthArr;
        for (size_t i = 0; i < str.size(); i++) {
            std::u16string singleChar(1, str[i]);
            float charW = impl_->measureWidth(singleChar);
            widthArr.SetValue((tjs_int)i, (double)charW);
        }
        tTJSVariant arrVar;
        arrVar = widthArr.GetDispatch();
        dict.SetValue(TJS_W("charWidths"), arrVar);
    }

    tTJSVariant result;
    result = dict.GetDispatch();
    return result;
}
```

TJS 使用示例：

```javascript
var tm = new TextMetrics("MS Gothic", 24, 0);
var info = tm.measureText("Hello");

// 输出每个字符的宽度
for (var i = 0; i < info.charWidths.count; i++) {
    System.inform("字符 " + i + " 宽度: " + info.charWidths[i]);
}

// 计算光标位置（在第 3 个字符后）
var cursorX = 0;
for (var i = 0; i < 3; i++) {
    cursorX += info.charWidths[i];
}
System.inform("光标 X 位置: " + cursorX);
```

**性能注意**：逐字符调用 `measureWidth` 比整体度量慢得多。如果只需要总宽度，使用 `measureWidth` 而非 `measureText`。`charWidths` 是可选的高级功能。

</details>

---

## 下一步

[CMake 注册与测试](./03-CMake注册与测试.md) — 将 TextMetrics 插件集成到 KrKr2 构建系统，编写单元测试验证功能。