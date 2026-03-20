# psdfile 与 psbfile 插件源码导读

> **所属模块：** M09-插件系统与开发
> **前置知识：** [01-scriptsEx与csvParser](./01-scriptsEx与csvParser.md)、[02-类注册与宏体系](../02-ncbind绑定框架/02-类注册与宏体系.md)
> **预计阅读时间：** 40 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 psdfile 插件的三层架构（psdparse 解析库 → PSD 桥接类 → PSDStorage 虚拟文件系统）
2. 理解 psbfile 插件如何解析 PSB/PIMG 二进制格式并注册为自定义存储媒体
3. 掌握 `iTVPStorageMedia` 虚拟文件系统接口的实现模式
4. 理解 `NCB_REGISTER_CLASS` 中 `Factory`、`RawCallback`、`Variant`、`Property` 的混合使用
5. 对比两个插件在注册策略、资源管理、缓存设计上的异同

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| PSD | Photoshop Document | Adobe Photoshop 的原生文件格式，保存图层、蒙版、混合模式等完整编辑信息 |
| PSB/PIMG | PSB Binary / Packaged Image | E-mote 动画引擎使用的二进制打包格式，内含多个图片资源和动画数据 |
| 存储媒体 | Storage Media | KiriKiri 引擎的虚拟文件系统抽象，通过 `psd://` 或 `psb://` 前缀访问非标准文件资源 |
| 类工厂 | Factory | ncbind 中自定义对象创建逻辑的机制，替代默认的 `new Class()` 构造 |
| 混合模式 | Blend Mode | 图层合成时像素颜色的混合算法，如正常、正片叠底、滤色、叠加等 |
| 内存映射文件 | Memory-Mapped File | 操作系统将文件直接映射到进程地址空间的技术，读取文件像读数组一样高效 |
| 弱引用 | Weak Reference | 不增加引用计数的对象引用方式，被引用对象被销毁时弱引用自动失效 |

---

## 一、psdfile 三层架构概览

psdfile 是 KrKr2 中最复杂的插件之一，它的职责是让 TJS 脚本能够直接读取 Adobe Photoshop 的 PSD 文件，访问其中的图层数据、蒙版信息、混合模式等完整编辑信息。这个插件的设计采用了**三层分离架构**，每一层只负责一件事，层与层之间通过清晰的接口通信。这种分层设计是大型插件的典型模式——如果把所有逻辑塞进一个类，维护和调试都将极为困难。

三层架构从底到顶分别是：

1. **psdparse 解析库**（底层）：纯 C++ 的 PSD 文件解析器，与 KiriKiri 引擎无关
2. **PSD 桥接类**（中间层）：继承 psdparse 的解析能力，添加 TJS 集成逻辑
3. **PSDStorage 虚拟文件系统**（顶层）：实现 `iTVPStorageMedia` 接口，让图层可以通过 `psd://` 协议访问

下面用架构图展示三层之间的关系：

```
┌─────────────────────────────────────────────────────────┐
│                    TJS 脚本层                            │
│  var psd = new PSD();                                   │
│  psd.load("image.psd");                                 │
│  var info = psd.getLayerInfo(0);                        │
│  // 或通过虚拟文件系统：                                  │
│  var bmp = "psd://image.psd/root/folder/layer.bmp";     │
├─────────────────────────────────────────────────────────┤
│  第三层：PSDStorage（虚拟文件系统）                       │
│  ┌─────────────────────────────────────────────────┐    │
│  │ class PSDStorage : public iTVPStorageMedia,     │    │
│  │                    public tTVPCompactEventCallba │    │
│  │ • GetName() → "psd"                             │    │
│  │ • CheckExistentStorage() → 委托给 PSD 对象       │    │
│  │ • Open() → 生成 BMP 内存流                       │    │
│  │ • GetListAt() → 列出图层名                       │    │
│  │ • OnCompact() → 内存不足时清理缓存               │    │
│  └───────────────┬─────────────────────────────────┘    │
│                  │ 持有 PSD 弱引用 (PSDMap)              │
├──────────────────┼──────────────────────────────────────┤
│  第二层：PSD 桥接类                                      │
│  ┌───────────────▼─────────────────────────────────┐    │
│  │ class PSD : public psd::PSDFile                 │    │
│  │ • factory() → ncbind Factory 入口                │    │
│  │ • load(filename) → 加载 PSD 文件                 │    │
│  │ • getLayerInfo(no) → 返回 TJS 辞书              │    │
│  │ • getLayerData(layer, no) → 写入 Layer 像素     │    │
│  │ • getSlices() / getGuides() / getBlend()        │    │
│  │ • CheckExistentStorage() → 虚拟路径解析          │    │
│  │ • openLayerImage() → 生成 BMP 内存流            │    │
│  └───────────────┬─────────────────────────────────┘    │
│                  │ 继承                                  │
├──────────────────┼──────────────────────────────────────┤
│  第一层：psdparse 解析库                                 │
│  ┌───────────────▼─────────────────────────────────┐    │
│  │ class psd::PSDFile                              │    │
│  │ • load(path) → boost 内存映射文件读取            │    │
│  │ • layerList → 图层信息列表                       │    │
│  │ • getLayerImage() → 解码像素数据                 │    │
│  │ • getMergedImage() → 获取合成结果                │    │
│  │ • isLoaded / get_width() / get_height()         │    │
│  └─────────────────────────────────────────────────┘    │
│  （纯 C++，不依赖 KiriKiri）                             │
└─────────────────────────────────────────────────────────┘
```

### 1.1 第一层：psdparse 解析库

psdparse 是一个独立的 C++ 库，存放在 `cpp/plugins/psdfile/psdparse/` 目录下。它的唯一职责是**解析 PSD 文件的二进制格式**，将文件头、图层列表、像素数据等信息解析为 C++ 结构体。这个库完全不依赖 KiriKiri 引擎，理论上可以在任何 C++ 项目中使用。

核心类定义在 `psdparse/psdfile.h`（第 1-47 行）：

```cpp
// 文件：cpp/plugins/psdfile/psdparse/psdfile.h
// 简化展示，保留核心成员

namespace psd {

// 图层信息结构体——保存单个图层的所有元数据
struct LayerInfo {
    int top, left, bottom, right;  // 图层在画布上的位置（像素坐标）
    int width, height;             // 图层尺寸
    int opacity;                   // 不透明度（0-255，255 为完全不透明）
    int fill_opacity;              // 填充不透明度（不影响图层效果）
    BlendMode blendMode;           // 混合模式枚举（Normal, Multiply, Screen 等）
    LayerType layerType;           // 图层类型（普通图层、文件夹、调整图层等）
    std::string layerName;         // 图层名称（ASCII 编码）
    std::wstring layerNameUnicode; // 图层名称（Unicode 编码，优先使用）
    LayerInfo *parent;             // 父图层指针（用于图层组嵌套）
    int layerId;                   // 图层唯一 ID
    std::vector<ChannelInfo> channels;    // 通道信息列表
    std::map<int, LayerCompInfo> layerComps; // 图层复合信息
    LayerExtraData extraData;      // 额外数据（蒙版、矢量数据等）
};

// PSD 文件解析器基类
class PSDFile {
protected:
    bool isLoaded;                     // 文件是否已成功加载
    std::vector<LayerInfo> layerList;  // 所有图层的列表
    unsigned char *imageData;          // 合成后的完整图像数据
    SliceResource slice;               // 切片信息
    GridGuideResource gridGuide;       // 参考线信息
    std::vector<LayerComp> layerComps; // 图层复合列表

public:
    void load(const std::string &path); // 加载 PSD 文件（使用 boost 内存映射）
    void clearData();                    // 释放所有解析数据
    int get_width();                     // 获取画布宽度
    int get_height();                    // 获取画布高度
    int get_channels();                  // 获取通道数
    int get_layer_count();               // 获取图层数量

    // 解码图层像素数据到指定缓冲区
    void getLayerImage(LayerInfo &lay, unsigned char *buffer,
                       ColorFormat format, int pitch,
                       ImageMode mode = IMAGE_MODE_MASKEDIMAGE);
    // 获取合成后的完整图像
    void getMergedImage(unsigned char *buffer, ColorFormat format, int pitch);
};

} // namespace psd
```

这个基类使用 boost 的内存映射文件（Memory-Mapped File）技术来读取 PSD 文件。内存映射文件是操作系统提供的一种高效文件访问方式——它不像传统 `fread` 那样将数据复制到应用程序缓冲区，而是直接将文件内容映射到进程的虚拟地址空间。读取文件数据就像访问数组一样简单，操作系统会按需将文件页面加载到物理内存，非常适合处理像 PSD 这样的大文件（一个包含几十个图层的 PSD 文件可能有几百 MB）。

### 1.2 第二层：PSD 桥接类

PSD 桥接类定义在 `psdclass.h`（233 行）和 `psdclass.cpp`（894 行）中。它继承自 `psd::PSDFile`，在底层解析能力之上添加了 KiriKiri/TJS 集成功能。这一层是"翻译官"——它把 psdparse 返回的 C++ 结构体翻译成 TJS 脚本能理解的辞书（Dictionary）和数组（Array）。

来看 `psdclass.h` 中的类声明（简化版）：

```cpp
// 文件：cpp/plugins/psdfile/psdclass.h
// 简化展示，突出与 psd::PSDFile 的继承关系和 TJS 集成方法

class PSD : public psd::PSDFile {
    // PSDStorage 和 PSDIterator 需要访问内部数据
    friend class PSDStorage;
    friend class PSDIterator;

private:
    iTJSDispatch2 *objthis;  // 指向自身的 TJS 对象引用（用于回调）

    // 虚拟文件系统支持的数据结构
    typedef std::map<ttstr, int> NameIdxMap;            // 图层名 → 索引号
    typedef std::map<ttstr, NameIdxMap> PathMap;         // 路径 → 名称映射
    PathMap pathMap;                                     // "root/folder/" → {"layer": 0}
    std::map<int, int> layerIdIdxMap;                   // 图层 ID → 索引号
    bool storageStarted;                                // 存储数据是否已初始化

public:
    // ncbind Factory 要求的静态工厂方法
    static tjs_error factory(PSD **result, tjs_int numparams,
                             tTJSVariant **params, iTJSDispatch2 *objthis);

    bool load(const tjs_char *filename);   // 加载 PSD 文件
    tTJSVariant getSelf();                 // 获取 TJS 自身对象

    // TJS 暴露的属性（只读）
    int get_width()       { return psd::PSDFile::get_width(); }
    int get_height()      { return psd::PSDFile::get_height(); }
    int get_channels()    { return psd::PSDFile::get_channels(); }
    int get_depth()       { return psd::PSDFile::get_depth(); }
    int get_color_mode()  { return (int)psd::PSDFile::get_color_mode(); }
    int get_layer_count() { return psd::PSDFile::get_layer_count(); }

    // 图层操作方法——返回 TJS 辞书/数组
    int getLayerType(int no);
    ttstr getLayerName(int no);
    tTJSVariant getLayerInfo(int no);      // 返回 TJS 辞书
    void getLayerData(tTJSVariant layer, int no);
    void getLayerDataRaw(tTJSVariant layer, int no);
    void getLayerDataMask(tTJSVariant layer, int no);
    tTJSVariant getSlices();               // 返回切片信息
    tTJSVariant getGuides();               // 返回参考线信息
    bool getBlend(tTJSVariant layer);      // 获取合成结果
    tTJSVariant getLayerComp();            // 返回图层复合信息

    // 虚拟文件系统接口（委托给 PSDStorage 调用）
    bool CheckExistentStorage(const ttstr &filename, int *layerIdxRet = 0);
    void GetListAt(const ttstr &pathname, iTVPStorageLister *lister);
    IStream *openLayerImage(const ttstr &name);
};
```

PSD 类的核心设计理念是**继承而非组合**：它直接继承 `psd::PSDFile`，因此可以直接访问 `layerList`、`isLoaded`、`imageData` 等受保护成员。这种继承关系意味着一个 PSD 对象既是 PSD 解析器，又是 TJS 绑定对象，同时还是虚拟文件系统的数据提供者。

### 1.3 第三层：PSDStorage 虚拟文件系统

PSDStorage 是 psdfile 插件最有特色的部分。它实现了 KiriKiri 引擎的 `iTVPStorageMedia` 接口（存储媒体接口，是 KiriKiri 虚拟文件系统的核心抽象），让 TJS 脚本可以像访问普通文件一样访问 PSD 文件中的图层。

举个具体例子：如果你有一个 PSD 文件 `character.psd`，其中有个名为 `smile` 的图层在 `face` 文件夹下，TJS 脚本可以这样访问它：

```javascript
// TJS 脚本示例：通过 psd:// 虚拟协议访问 PSD 图层
// 不需要手动创建 PSD 对象和调用 getLayerData

// 直接当作文件路径使用
var layer = new Layer(window, null);
layer.loadImages("psd://character.psd/root/face/smile.bmp");
// KiriKiri 引擎看到 "psd://" 前缀，自动委托给 PSDStorage 处理
// PSDStorage 自动加载 character.psd，找到 face/smile 图层，
// 将像素数据打包成 BMP 格式返回给引擎的图像加载器
```

来看 PSDStorage 的实现（`cpp/plugins/psdfile/main.cpp` 第 15-206 行）：

```cpp
// 文件：cpp/plugins/psdfile/main.cpp
// 简化展示，突出 iTVPStorageMedia 接口实现

// 弱引用映射：PSD 文件名 → PSD 对象指针
// 使用弱引用是因为 PSD 对象的生命周期由 TJS 垃圾回收管理，
// PSDStorage 不能持有强引用，否则会导致循环引用和内存泄漏
typedef std::map<ttstr, PSD*> PSDMap;

class PSDStorage : public iTVPStorageMedia,
                   public tTVPCompactEventCallbackIntf {
    PSDMap psdMap;       // 弱引用：文件名 → PSD 对象
    tTJSVariant cache;   // 强引用缓存：最近访问的 PSD 对象
    ttstr cacheName;     // 缓存的文件名

public:
    // 返回协议名称："psd"
    virtual void TJS_INTF_METHOD GetName(ttstr &name) override {
        name = TJS_W("psd");
    }

    // 检查 psd:// 路径下的文件（图层）是否存在
    virtual bool TJS_INTF_METHOD CheckExistentStorage(
        const ttstr &name) override {
        // 从 "psd://character.psd/root/face/smile.bmp" 中
        // 拆分出域名 "character.psd" 和路径 "root/face/smile.bmp"
        ttstr domain, path;
        splitPath(name, domain, path);

        // 在弱引用映射中查找已加载的 PSD 对象
        PSD *psd = findPSD(domain);
        if (!psd) {
            // 没有已加载的对象，自动创建并加载
            psd = autoLoadPSD(domain);
        }
        if (psd) {
            return psd->CheckExistentStorage(path);
        }
        return false;
    }

    // 打开 psd:// 路径下的文件，返回 IStream
    virtual IStream* TJS_INTF_METHOD Open(
        const ttstr &name, tjs_uint32 flags) override {
        ttstr domain, path;
        splitPath(name, domain, path);
        PSD *psd = findOrLoadPSD(domain);
        if (psd) {
            // PSD::openLayerImage 会将图层像素数据编码为
            // 内存中的 BMP 文件，返回 IStream 指针
            return psd->openLayerImage(path);
        }
        return nullptr;
    }

    // 列出 psd:// 路径下的"文件"（图层名列表）
    virtual void TJS_INTF_METHOD GetListAt(
        const ttstr &name, iTVPStorageLister *lister) override {
        ttstr domain, path;
        splitPath(name, domain, path);
        PSD *psd = findOrLoadPSD(domain);
        if (psd) {
            psd->GetListAt(path, lister);
        }
    }

    // tTVPCompactEventCallbackIntf 接口：内存不足时清理缓存
    virtual void TJS_INTF_METHOD OnCompact(tjs_int level) override {
        if (level >= TVP_CYCLIC_COMPACT_LEVEL) {
            cache.Clear();  // 释放强引用缓存
            cacheName.Clear();
        }
    }
};
```

PSDStorage 的缓存策略值得注意——它采用**两级缓存**：

| 缓存层级 | 类型 | 数据结构 | 生命周期 |
|---------|------|---------|---------|
| 第一级 | 强引用 | `tTJSVariant cache` | 最近访问的单个 PSD 对象，内存不足时清理 |
| 第二级 | 弱引用 | `PSDMap psdMap`（std::map） | 所有已加载的 PSD 对象，对象析构时自动移除 |

强引用缓存（`cache`）确保最近使用的 PSD 对象不会被 TJS 垃圾回收器回收。弱引用映射（`psdMap`）则记录所有已加载的 PSD 对象，但不阻止它们被回收——当 TJS 端的 PSD 对象被 `invalidate` 或垃圾回收时，PSD 析构函数会调用 `removeFromStorage()` 将自己从 `psdMap` 中移除。

---

## 二、PSD 的 ncbind 注册分析

PSD 类的 ncbind 注册代码位于 `cpp/plugins/psdfile/psdclass.cpp` 第 801-894 行，是一个**混合注册**的典型案例——在一个 `NCB_REGISTER_CLASS(PSD)` 块中同时使用了 `Factory`、`Variant` 常量、`Property` 属性和 `NCB_METHOD` 方法四种注册机制。

### 2.1 Factory 自定义工厂

```cpp
// 文件：cpp/plugins/psdfile/psdclass.cpp 第 801-803 行
NCB_REGISTER_CLASS(PSD) {
    Factory(&ClassT::factory);
    // ClassT 是 NCB_REGISTER_CLASS 宏内部 typedef，等价于 PSD
```

`Factory(&ClassT::factory)` 告诉 ncbind：不要使用默认的 `new PSD()` 构造方式，而是调用 `PSD::factory` 静态方法来创建对象。为什么需要自定义工厂？因为 PSD 构造函数需要接收 `iTJSDispatch2 *objthis` 参数（指向 TJS 端对象的指针），用于后续回调和自引用。默认构造器无法传递这个参数。

来看工厂方法的签名（第 108-112 行）：

```cpp
// 文件：cpp/plugins/psdfile/psdclass.cpp 第 108-112 行
// Factory 签名是 ncbind 规定的固定格式
tjs_error PSD::factory(
    PSD **result,           // [out] 创建好的对象指针
    tjs_int numparams,      // 传入的参数个数
    tTJSVariant **params,   // 参数数组
    iTJSDispatch2 *objthis  // TJS 端的对象引用
) {
    *result = new PSD(objthis);  // 将 objthis 传给构造函数
    return TJS_S_OK;             // 返回成功状态码
}
```

Factory 方法的签名必须严格遵循 `tjs_error (*)(T**, tjs_int, tTJSVariant**, iTJSDispatch2*)` 格式，这是 ncbind 内部的 `ncbNativeClassMethodBase::FactoryType` 约定。返回值 `TJS_S_OK`（值为 0）表示创建成功；如果构造失败，应返回 `TJS_E_FAIL` 并将 `*result` 设为 `nullptr`。

### 2.2 Variant 常量注册

PSD 类注册了约 50 个枚举常量，让 TJS 脚本可以用符号名访问 PSD 的颜色模式、混合模式和图层类型：

```cpp
// 文件：cpp/plugins/psdfile/psdclass.cpp 第 805-858 行（节选）
// NCB_TYPECONV_CAST_INTEGER 是一个宏，告诉 ncbind
// 如何将 psd:: 命名空间的枚举转换为 TJS 整数

// 颜色模式常量
Variant(TJS_W("COLOR_MODE_BITMAP"),       (tjs_int)psd::COLOR_MODE_BITMAP);
Variant(TJS_W("COLOR_MODE_GRAYSCALE"),    (tjs_int)psd::COLOR_MODE_GRAYSCALE);
Variant(TJS_W("COLOR_MODE_INDEXED"),      (tjs_int)psd::COLOR_MODE_INDEXED);
Variant(TJS_W("COLOR_MODE_RGB"),          (tjs_int)psd::COLOR_MODE_RGB);
Variant(TJS_W("COLOR_MODE_CMYK"),         (tjs_int)psd::COLOR_MODE_CMYK);
Variant(TJS_W("COLOR_MODE_MULTICHANNEL"), (tjs_int)psd::COLOR_MODE_MULTICHANNEL);
Variant(TJS_W("COLOR_MODE_DUOTONE"),      (tjs_int)psd::COLOR_MODE_DUOTONE);
Variant(TJS_W("COLOR_MODE_LAB"),          (tjs_int)psd::COLOR_MODE_LAB);

// 混合模式常量（约 25 个）
Variant(TJS_W("BLEND_MODE_NORMAL"),       (tjs_int)psd::BLEND_MODE_NORMAL);
Variant(TJS_W("BLEND_MODE_DISSOLVE"),     (tjs_int)psd::BLEND_MODE_DISSOLVE);
Variant(TJS_W("BLEND_MODE_DARKEN"),       (tjs_int)psd::BLEND_MODE_DARKEN);
Variant(TJS_W("BLEND_MODE_MULTIPLY"),     (tjs_int)psd::BLEND_MODE_MULTIPLY);
Variant(TJS_W("BLEND_MODE_COLOR_BURN"),   (tjs_int)psd::BLEND_MODE_COLOR_BURN);
Variant(TJS_W("BLEND_MODE_LINEAR_BURN"),  (tjs_int)psd::BLEND_MODE_LINEAR_BURN);
// ... 更多混合模式（Screen, Overlay, SoftLight, HardLight 等）

// 图层类型常量
Variant(TJS_W("LAYER_TYPE_NORMAL"),       (tjs_int)psd::LAYER_TYPE_NORMAL);
Variant(TJS_W("LAYER_TYPE_HIDDEN"),       (tjs_int)psd::LAYER_TYPE_HIDDEN);
Variant(TJS_W("LAYER_TYPE_FOLDER"),       (tjs_int)psd::LAYER_TYPE_FOLDER);
Variant(TJS_W("LAYER_TYPE_ADJUST"),       (tjs_int)psd::LAYER_TYPE_ADJUST);
```

在 TJS 脚本中使用这些常量：

```javascript
// TJS 脚本示例：使用 PSD 类的枚举常量
var psd = new PSD();
psd.load("character.psd");

// 检查颜色模式
if (psd.color_mode == PSD.COLOR_MODE_RGB) {
    System.inform("这是 RGB 模式的 PSD 文件");
}

// 遍历图层，过滤出普通图层
for (var i = 0; i < psd.layer_count; i++) {
    if (psd.getLayerType(i) == PSD.LAYER_TYPE_NORMAL) {
        var info = psd.getLayerInfo(i);
        System.inform("普通图层: " + info.name
            + "，混合模式=" + info.blend_mode);
    }
}
```

### 2.3 Property 属性注册——INTPROP 宏技巧

PSD 类使用了一个自定义的 `INTPROP` 宏来简化属性注册：

```cpp
// 文件：cpp/plugins/psdfile/psdclass.cpp 第 860-861 行
// 定义 INTPROP 宏——把属性名字符串化 + 拼接 getter 方法名
#define INTPROP(name) Property(TJS_W(#name), &Class::get_##name, 0)
// 展开示例：INTPROP(width) → Property(TJS_W("width"), &Class::get_width, 0)
//   TJS_W(#name) → 将宏参数字符串化为宽字符串 L"width"
//   &Class::get_##name → 拼接为 &Class::get_width（getter 方法指针）
//   0 → setter 为空指针，表示只读属性

// 使用 INTPROP 注册 6 个只读属性
INTPROP(width);        // → Property("width",      &PSD::get_width,       0)
INTPROP(height);       // → Property("height",     &PSD::get_height,      0)
INTPROP(channels);     // → Property("channels",   &PSD::get_channels,    0)
INTPROP(depth);        // → Property("depth",      &PSD::get_depth,       0)
INTPROP(color_mode);   // → Property("color_mode", &PSD::get_color_mode,  0)
INTPROP(layer_count);  // → Property("layer_count",&PSD::get_layer_count, 0)
#undef INTPROP  // 用完立即 #undef，避免污染全局宏命名空间
```

这里的 `#name` 是 C 预处理器的**字符串化操作符**（Stringification Operator），它将宏参数原样转换为字符串字面量。`##` 是**标记粘合操作符**（Token Pasting Operator），它将两个标记拼接成一个标识符。两者组合使用实现了"一个名字同时用作属性名和方法名"的效果。

`Property` 的第三个参数是 setter 方法指针，传 `0`（即空指针）表示该属性是只读的。如果 TJS 脚本尝试写入只读属性，ncbind 框架会抛出异常。

### 2.4 NCB_METHOD 方法注册

PSD 类注册了 10 个方法，涵盖图层数据读取、切片/参考线获取等功能：

```cpp
// 文件：cpp/plugins/psdfile/psdclass.cpp 第 869-893 行
NCB_METHOD(load);          // bool load(filename) — 加载 PSD 文件
NCB_METHOD(getLayerType);  // int getLayerType(no) — 获取图层类型枚举
NCB_METHOD(getLayerName);  // ttstr getLayerName(no) — 获取图层名称
NCB_METHOD(getLayerInfo);  // tTJSVariant getLayerInfo(no) — 获取图层详细信息辞书
NCB_METHOD(getLayerData);  // void getLayerData(layer, no) — 读取图层像素（含蒙版）
NCB_METHOD(getLayerDataRaw);  // void getLayerDataRaw(layer, no) — 读取原始像素
NCB_METHOD(getLayerDataMask); // void getLayerDataMask(layer, no) — 只读取蒙版
NCB_METHOD(getSlices);     // tTJSVariant getSlices() — 获取切片数据
NCB_METHOD(getGuides);     // tTJSVariant getGuides() — 获取参考线数据
NCB_METHOD(getBlend);      // bool getBlend(layer) — 获取合成结果
NCB_METHOD(getLayerComp);  // tTJSVariant getLayerComp() — 获取图层复合
```

其中 `getLayerInfo` 方法是最有教学价值的——它展示了如何用 `ncbDictionaryAccessor`（ncbind 提供的 TJS 辞书操作辅助类）将 C++ 结构体逐字段转换为 TJS 辞书：

```cpp
// 文件：cpp/plugins/psdfile/psdclass.cpp 第 216-301 行（简化）
tTJSVariant PSD::getLayerInfo(int no) {
    checkLayerNo(no);  // 检查图层号合法性
    psd::LayerInfo &lay = layerList[no];

    tTJSVariant result;
    ncbDictionaryAccessor dict;  // 创建一个 TJS 辞书对象
    if (dict.IsValid()) {
        // 宏 SETPROP 将 C++ 成员直接映射为辞书键值对
        // #prop 字符串化为辞书的 key，obj.prop 是 value
        #define SETPROP(dict, obj, prop) \
            dict.SetValue(TJS_W(#prop), obj.prop)

        SETPROP(dict, lay, top);      // dict["top"] = lay.top
        SETPROP(dict, lay, left);     // dict["left"] = lay.left
        SETPROP(dict, lay, bottom);   // dict["bottom"] = lay.bottom
        SETPROP(dict, lay, right);    // dict["right"] = lay.right
        SETPROP(dict, lay, width);    // dict["width"] = lay.width
        SETPROP(dict, lay, height);   // dict["height"] = lay.height
        SETPROP(dict, lay, opacity);  // dict["opacity"] = lay.opacity

        // 检查是否有蒙版通道
        bool mask = false;
        for (auto &channel : lay.channels) {
            if (channel.isMaskChannel()) { mask = true; break; }
        }
        dict.SetValue(TJS_W("mask"), mask);

        // 混合模式需要经过 convBlendMode 转换为 KiriKiri 内部的混合类型
        dict.SetValue(TJS_W("type"), convBlendMode(lay.blendMode));
        dict.SetValue(TJS_W("blend_mode"), lay.blendMode);
        dict.SetValue(TJS_W("visible"), lay.isVisible());
        dict.SetValue(TJS_W("name"), layname(lay));

        // 如果有父图层（图层组），记录父图层 ID
        if (lay.parent != nullptr)
            dict.SetValue(TJS_W("group_layer_id"), lay.parent->layerId);

        result = dict;  // 将辞书包装为 tTJSVariant 返回
    }
    return result;
}
```

`ncbDictionaryAccessor` 和 `ncbArrayAccessor` 是 ncbind 框架提供的两个辅助类，它们封装了 TJS 辞书和数组的创建与操作。使用它们比直接调用 `iTJSDispatch2` 接口简洁得多——无需手动管理引用计数、无需记忆 `FuncCall` / `PropSet` 等底层方法名。

---

## 三、PSDStorage——虚拟文件系统深度剖析

`iTVPStorageMedia` 是 KiriKiri 引擎的**存储媒体接口**，任何实现了该接口的类都可以注册为一种"文件系统协议"。KiriKiri 内置了 `file://`（本地文件）和 `arc://`（XP3 归档包）等协议；psdfile 插件通过注册 `psd://` 协议，让引擎能够"打开" PSD 文件中的图层，就像打开普通图片文件一样。

### 3.1 iTVPStorageMedia 接口规范

要实现一个自定义存储媒体，你需要实现以下方法（来自 `tp_stub.h`）：

```cpp
// KiriKiri 引擎定义的存储媒体接口
// 每个方法的 TJS_INTF_METHOD 表示使用引擎约定的调用约定
class iTVPStorageMedia {
public:
    // 返回协议名称（如 "psd"、"psb"）
    // 引擎据此将 "psd://xxx" 的请求路由到正确的处理器
    virtual void TJS_INTF_METHOD GetName(ttstr &name) = 0;

    // 规范化存储路径（将相对路径转为绝对路径）
    virtual bool TJS_INTF_METHOD NormalizeDomainName(ttstr &name) = 0;

    // 规范化路径名中的大小写和分隔符
    virtual bool TJS_INTF_METHOD NormalizePathName(ttstr &name) = 0;

    // 检查指定路径的"文件"是否存在
    virtual bool TJS_INTF_METHOD CheckExistentStorage(
        const ttstr &name) = 0;

    // 打开指定路径的"文件"，返回 IStream 流
    virtual IStream* TJS_INTF_METHOD Open(
        const ttstr &name, tjs_uint32 flags) = 0;

    // 列出指定路径下的所有"文件"
    virtual void TJS_INTF_METHOD GetListAt(
        const ttstr &name, iTVPStorageLister *lister) = 0;

    // 获取本地文件路径（如果不适用，返回空字符串）
    virtual void TJS_INTF_METHOD GetLocallyAccessibleName(
        ttstr &name) = 0;

    // AddRef/Release 引用计数管理
    virtual void TJS_INTF_METHOD AddRef() = 0;
    virtual void TJS_INTF_METHOD Release() = 0;
};
```

### 3.2 路径解析机制

PSD 虚拟文件系统支持两种路径格式：

**路径格式一：按图层路径访问**
```
psd://文件名/root/文件夹名/图层名.bmp
```

例如 `psd://character.psd/root/face/smile.bmp`：
- `character.psd` → 域名部分（PSD 文件名）
- `root/face/` → 路径部分（图层在 PSD 中的文件夹层级）
- `smile.bmp` → 文件名部分（图层名，`.bmp` 是固定后缀）

**路径格式二：按图层 ID 访问**
```
psd://文件名/id/图层ID.bmp
```

例如 `psd://character.psd/id/42.bmp`：
- `42` → 图层的唯一 ID（Photoshop 分配的数字标识符）

来看 `CheckExistentStorage` 中的路径解析逻辑（`psdclass.cpp` 第 640-705 行）：

```cpp
// 文件：cpp/plugins/psdfile/psdclass.cpp 第 640-705 行（简化）
bool PSD::CheckExistentStorage(const ttstr &filename, int *layerIdxRet) {
    startStorage();  // 延迟初始化路径映射数据

    const tjs_char *p = filename.c_str();

    // 路径格式二：id/42.bmp
    if (TJS_strncmp(p, TJS_W("id/"), 3) == 0) {
        p += 3;  // 跳过 "id/" 前缀
        const tjs_char *q;
        // 查找 ".bmp" 后缀，提取数字 ID
        if (!(q = TJS_strchr(p, '/')) &&
            ((q = TJS_strchr(p, '.')) &&
             (TJS_strcmp(q, BMPEXT) == 0))) {
            ttstr name = ttstr(p, q - p);  // 提取 "42"
            if (checkAllNum(name.c_str())) {  // 确保全是数字
                int id = TJS_atoi(name.c_str());
                auto n = layerIdIdxMap.find(id);  // 在 ID→索引映射中查找
                if (n != layerIdIdxMap.end()) {
                    if (layerIdxRet) *layerIdxRet = n->second;
                    return true;
                }
            }
        }
    }
    // 路径格式一：root/face/smile.bmp
    else {
        ttstr pname, fname;
        const tjs_char *q;
        if ((q = TJS_strchr(p, '/'))) {
            pname = ttstr(p, q - p + 1);  // "root/face/"
            fname = ttstr(q + 1);          // "smile.bmp"
        } else {
            return false;
        }

        // 提取 basename（去掉 .bmp 后缀）
        ttstr basename;
        p = fname.c_str();
        if ((q = TJS_strchr(p, '.')) &&
            (TJS_strcmp(q, BMPEXT) == 0)) {
            basename = ttstr(p, q - p);  // "smile"
        } else {
            return false;
        }

        // 在 pathMap 中查找
        auto n = pathMap.find(pname);  // 查找 "root/face/"
        if (n != pathMap.end()) {
            auto m = n->second.find(basename);  // 查找 "smile"
            if (m != n->second.end()) {
                if (layerIdxRet) *layerIdxRet = m->second;
                return true;
            }
        }
    }
    return false;
}
```

### 3.3 图层到 BMP 的转换

当引擎通过 `Open()` 方法请求某个图层的数据时，`openLayerImage` 方法会将图层像素编码为内存中的 BMP 文件。为什么选择 BMP 格式？因为 BMP 是无压缩格式，生成简单且速度快；而 KiriKiri 引擎的图像加载器本身就支持 BMP 解码，所以不需要额外的编解码开销。

```cpp
// 文件：cpp/plugins/psdfile/psdclass.cpp 第 744-793 行（简化）
IStream *PSD::openLayerImage(const ttstr &name) {
    int layerIdx;
    if (CheckExistentStorage(name, &layerIdx)) {
        psd::LayerInfo &lay = layerList[layerIdx];
        if (lay.layerType != psd::LAYER_TYPE_NORMAL ||
            lay.width <= 0 || lay.height <= 0)
            return nullptr;

        int width = lay.width;
        int height = lay.height;
        int pitch = width * 4;  // BGRA 每像素 4 字节

        // 计算 BMP 文件总大小
        int hsize = sizeof(TVP_WIN_BITMAPFILEHEADER);   // 14 字节
        int isize = hsize + sizeof(TVP_WIN_BITMAPINFOHEADER); // +40 字节
        int size = isize + pitch * height;  // + 像素数据

        // 在堆上分配 BMP 数据
        auto *handle = reinterpret_cast<uint8_t*>(malloc(size));
        if (handle) {
            // 填写 BMP 文件头
            TVP_WIN_BITMAPFILEHEADER bfh{};
            bfh.bfType = 'B' + ('M' << 8);  // "BM" 签名
            bfh.bfSize = size;
            bfh.bfOffBits = isize;  // 像素数据偏移
            memcpy(handle, &bfh, sizeof bfh);

            // 填写 BMP 信息头
            TVP_WIN_BITMAPINFOHEADER bih{};
            bih.biSize = sizeof(bih);
            bih.biWidth = width;
            bih.biHeight = height;    // 正数 = 自底向上存储
            bih.biPlanes = 1;
            bih.biBitCount = 32;      // 32 位 BGRA
            bih.biCompression = BI_RGB;  // 无压缩
            memcpy(handle + hsize, &bih, sizeof bih);

            // 解码图层像素到 BMP 数据区
            // 注意：BMP 的行顺序是自底向上的
            getLayerImage(lay,
                handle + isize + pitch * (height - 1),  // 从最后一行开始
                psd::BGRA_LE,   // 小端 BGRA 格式（Windows 标准）
                -pitch);        // 负 pitch = 向上逐行填充

            // 封装为 IStream 返回
            return SHCreateMemStream(handle, size);
        }
    }
    return nullptr;
}
```

### 3.4 自动加载与生命周期管理

PSDStorage 的 `main.cpp` 中实现了一套完整的 PSD 对象生命周期管理：

```cpp
// 文件：cpp/plugins/psdfile/main.cpp（简化）

// 全局弱引用映射
static PSDMap psdMap;

// PSD 对象加载后注册自己（由 PSD::load 调用）
void addToStorage(const ttstr &name, PSD *psd) {
    psdMap[name] = psd;  // 弱引用，不增加引用计数
}

// PSD 对象销毁前注销自己（由 PSD::clearData 调用）
void removeFromStorage(const ttstr &name) {
    psdMap.erase(name);
}

// 当 psd:// 请求引用了未加载的 PSD 文件时，自动创建并加载
// 这是 PSDStorage 最巧妙的设计之一
PSD* autoLoadPSD(const ttstr &domain) {
    // 执行 TJS 表达式创建 PSD 对象
    tTJSVariant val;
    TVPExecuteExpression(TJS_W("new PSD()"), &val);
    // 获取 PSD 对象并调用 load
    // val 被存入 cache（强引用），防止被垃圾回收
    cache = val;
    cacheName = domain;
    // ...
    return psd;
}
```

生命周期流程图：

```
TJS: var psd = new PSD()
  → ncbind Factory → PSD::factory() → new PSD(objthis)

TJS: psd.load("character.psd")
  → PSD::load() → psd::PSDFile::load() 解析文件
  → addToStorage("character.psd", this)  // 注册到弱引用映射

引擎: loadImages("psd://character.psd/root/face/smile.bmp")
  → PSDStorage::Open("character.psd", "root/face/smile.bmp")
  → psdMap["character.psd"]  // 通过弱引用找到 PSD 对象
  → PSD::openLayerImage("root/face/smile.bmp")
  → 返回 BMP 内存流

TJS: psd = void (释放引用)
  → TJS GC 回收 → PSD::~PSD() → PSD::clearData()
  → removeFromStorage("character.psd")  // 从映射中移除

内存不足时:
  → PSDStorage::OnCompact(level)
  → cache.Clear()  // 释放强引用缓存
```

### 3.5 注册与注销回调

PSDStorage 的注册使用 `NCB_PRE_REGIST_CALLBACK` 和 `NCB_POST_UNREGIST_CALLBACK`：

```cpp
// 文件：cpp/plugins/psdfile/main.cpp 第 242-260 行
static PSDStorage *psdStorage = nullptr;

// 插件加载时调用——注册 psd:// 存储媒体
static void initStorage() {
    psdStorage = new PSDStorage();
    TVPRegisterStorageMedia(psdStorage);  // 向引擎注册
}

// 插件卸载时调用——注销 psd:// 存储媒体
static void doneStorage() {
    TVPUnregisterStorageMedia(psdStorage);  // 从引擎注销
    psdStorage->Release();  // 释放存储媒体对象
}

NCB_PRE_REGIST_CALLBACK(initStorage);
NCB_POST_UNREGIST_CALLBACK(doneStorage);
```

这与上一节中 scriptsEx 使用 `NCB_PRE_REGIST_CALLBACK` 缓存 `Array.clear` 的模式完全一致——都是利用注册回调在插件初始化时执行全局设置。区别在于 psdfile 的回调做的事情更重：它创建了一个新的存储媒体对象并注册到引擎中。

---

## 四、psbfile 架构与注册

psbfile 是另一个提供虚拟文件系统的插件，但它处理的是 E-mote 动画引擎使用的 PSB/PIMG 格式。与 psdfile 的三层架构不同，psbfile 的结构更加扁平——只有两层：解析类（PSBFile）和存储媒体（PSBMedia）。

### 4.1 PSBFile 类

PSBFile 类定义在 `cpp/plugins/psbfile/PSBFile.h`（108 行），负责解析 PSB 二进制格式并提取其中的资源：

```cpp
// 文件：cpp/plugins/psbfile/PSBFile.h（简化）

class PSBFile {
    static const char SIGNATURE[];      // PSB 文件签名 "PSB\0"
    static const char SIGNATURE_MRES[]; // PIMG 文件签名 "mres"
    static const char SIGNATURE_EMOT[]; // E-mote 签名 "emot"

    // PSB 文件头结构（20 字节固定头）
    struct Header {
        char signature[4];    // 文件签名
        uint32_t version;     // 版本号
        uint32_t encrypt;     // 加密标志
        uint32_t treeOffset;  // 对象树偏移
        // ... 更多偏移量字段
    };

    uint8_t *base;           // 文件数据基地址
    uint32_t length;         // 文件数据长度
    Header header;           // 解析后的文件头

public:
    bool load(const ttstr &filename);  // 加载 PSB/PIMG 文件
    tTJSVariant getRoot();             // 获取对象树根节点
    int getResourceCount();            // 获取资源数量
    std::string getResourceName(int i); // 获取第 i 个资源的名称
    const uint8_t* getResourceData(int i, uint32_t &size);  // 获取资源数据
    std::string getType();             // 获取文件类型（"PSB"/"PIMG"/"EMOT"）
};
```

PSBFile 比 PSD 类简单得多：它不需要复杂的图层层级解析，只需从 PSB 二进制文件中提取**对象树**（一种类似 JSON 的嵌套数据结构）和**资源列表**（一组命名的二进制数据块，通常是图片）。

### 4.2 PSBFile 的 ncbind 注册——Factory + RawCallback + NCB_SET_CONVERTOR

psbfile 的注册代码位于 `cpp/plugins/psbfile/main.cpp`（151 行），虽然代码量不大，但使用了三种高级 ncbind 特性：

```cpp
// 文件：cpp/plugins/psbfile/main.cpp 第 128-148 行

// 自定义工厂——支持两种构造方式
struct PSBFileFactory {
    static tjs_error factory(
        PSBFile **result, tjs_int numparams,
        tTJSVariant **params, iTJSDispatch2 *objthis
    ) {
        if (numparams == 0) {
            // new PSBFile() — 空构造，稍后调用 load()
            *result = new PSBFile();
        } else {
            // new PSBFile("filename.pimg") — 直接加载
            *result = new PSBFile();
            ttstr filename = *params[0];
            (*result)->load(filename);
        }
        return TJS_S_OK;
    }
};

NCB_REGISTER_CLASS(PSBFile) {
    Factory(&PSBFileFactory::factory);  // 自定义工厂

    // RawCallback：直接操作 TJS 变量的低级方法
    // 当方法需要返回复杂的 TJS 对象树时，ncbind 的自动类型转换不够用
    RawCallback(TJS_W("root"),  &getRoot,  0);  // 获取 PSB 对象树
    RawCallback(TJS_W("load"),  &load,     0);  // 加载文件并注册资源

    NCB_METHOD(getResourceCount);  // 获取资源数量
    NCB_METHOD(getResourceName);   // 获取资源名称
    NCB_METHOD(getType);           // 获取文件类型
}
```

**RawCallback** 是 ncbind 中最底层的方法注册方式。与 `NCB_METHOD`（自动处理参数转换和返回值包装）不同，RawCallback 要求你直接操作 `tTJSVariant` 数组——自己解析参数、自己设置返回值。这在两种场景下必须使用：

1. 返回值是复杂的 TJS 对象树（无法通过 ncbind 自动转换）
2. 需要根据参数个数/类型做不同处理（重载模拟）

来看 `getRoot` 和 `load` 的 RawCallback 实现：

```cpp
// 文件：cpp/plugins/psbfile/main.cpp 第 38-89 行

// getRoot 的 RawCallback 实现
static tjs_error TJS_INTF_METHOD getRoot(
    tTJSVariant *result,     // [out] 返回值
    tjs_int numparams,       // 参数个数
    tTJSVariant **params,    // 参数数组
    iTJSDispatch2 *objthis   // TJS this 对象
) {
    // 从 TJS 对象获取 C++ 原生实例
    PSBFile *self = ncbInstanceAdaptor<PSBFile>::GetNativeInstance(objthis);
    if (!self) return TJS_E_NATIVECLASSCRASH;

    // 调用 C++ 方法获取对象树，直接赋值给 result
    if (result) {
        *result = self->getRoot();  // 返回 tTJSVariant 格式的对象树
    }
    return TJS_S_OK;
}

// load 的 RawCallback 实现
static tjs_error TJS_INTF_METHOD load(
    tTJSVariant *result, tjs_int numparams,
    tTJSVariant **params, iTJSDispatch2 *objthis
) {
    PSBFile *self = ncbInstanceAdaptor<PSBFile>::GetNativeInstance(objthis);
    if (!self) return TJS_E_NATIVECLASSCRASH;

    if (numparams < 1) return TJS_E_BADPARAMCOUNT;

    ttstr filename = *params[0];
    bool loaded = self->load(filename);

    if (loaded) {
        // 加载成功后，将资源注册到 PSBMedia 存储媒体
        int count = self->getResourceCount();
        for (int i = 0; i < count; i++) {
            std::string name = self->getResourceName(i);
            // 将资源注册为 psb://filename/resourceName 路径
            psbMedia->addResource(filename, name, self, i);
        }
    }

    if (result) *result = loaded;
    return TJS_S_OK;
}
```

### 4.3 NCB_SET_CONVERTOR——自定义类型转换器

psbfile 还使用了一个在前面章节中未涉及的高级特性：`NCB_SET_CONVERTOR`。这个宏用于注册**自定义类型转换器**（Type Convertor），让 ncbind 知道如何在 TJS `tTJSVariant` 和 C++ `PSBFile*` 之间互相转换。

```cpp
// 文件：cpp/plugins/psbfile/main.cpp 第 98-126 行

// 自定义转换器模板
template <class T>
struct PSBFileConvertor {
    // TJS → C++：从 tTJSVariant 中提取 C++ 对象指针
    static bool IsInstance(const tTJSVariant &src) {
        return ncbInstanceAdaptor<T>::GetNativeInstance(
            src.AsObjectNoAddRef()) != nullptr;
    }

    // TJS → C++：执行实际转换
    static bool GetData(const tTJSVariant &src, T **dest) {
        *dest = ncbInstanceAdaptor<T>::GetNativeInstance(
            src.AsObjectNoAddRef());
        return *dest != nullptr;
    }

    // C++ → TJS：将 C++ 对象包装为 tTJSVariant
    static bool SetData(tTJSVariant &dest, const T *src) {
        iTJSDispatch2 *adp =
            ncbInstanceAdaptor<T>::CreateAdaptor(const_cast<T*>(src));
        if (adp) {
            dest = tTJSVariant(adp, adp);
            adp->Release();  // CreateAdaptor 返回的引用计数为 1
            return true;
        }
        return false;
    }
};

// 注册转换器——让 ncbind 知道如何处理 PSBFile 类型
NCB_SET_CONVERTOR(PSBFile, PSBFileConvertor<PSBFile>);
```

为什么需要自定义转换器？默认情况下，ncbind 只知道如何处理基本类型（`int`、`double`、`ttstr` 等）和通过 `NCB_REGISTER_CLASS` 注册的类。但如果你的方法参数或返回值类型是另一个 ncbind 注册的类（例如一个方法接收 `PSBFile*` 参数），ncbind 需要知道如何从 `tTJSVariant` 中提取那个 C++ 对象指针。`NCB_SET_CONVERTOR` 就是告诉 ncbind 这个转换规则。

核心函数 `ncbInstanceAdaptor<T>::GetNativeInstance()` 会检查 TJS 对象是否确实是一个 `T` 类型的包装对象，如果是就返回内部的 C++ 指针；`CreateAdaptor()` 则反过来，将 C++ 对象包装为 TJS 可以识别的 `iTJSDispatch2` 对象。

### 4.4 PSBMedia 存储媒体

PSBMedia 定义在 `cpp/plugins/psbfile/PSBMedia.h`（46 行），实现与 PSDStorage 类似的 `iTVPStorageMedia` 接口：

```cpp
// 文件：cpp/plugins/psbfile/PSBMedia.h（简化）

class PSBMedia : public iTVPStorageMedia {
    // 资源映射：psb://文件名/资源名 → {PSBFile 指针, 资源索引}
    struct ResourceEntry {
        PSBFile *file;  // PSBFile 对象指针
        int index;       // 资源在 PSBFile 中的索引
    };
    std::map<ttstr, ResourceEntry> resources;

public:
    void GetName(ttstr &name) override {
        name = TJS_W("psb");  // 协议名 "psb"
    }

    bool CheckExistentStorage(const ttstr &name) override {
        // 在资源映射中查找
        return resources.find(name) != resources.end();
    }

    IStream* Open(const ttstr &name, tjs_uint32 flags) override {
        auto it = resources.find(name);
        if (it != resources.end()) {
            uint32_t size;
            const uint8_t *data = it->second.file->getResourceData(
                it->second.index, size);
            // 将资源数据封装为 IStream 返回
            return SHCreateMemStream(data, size);
        }
        return nullptr;
    }

    // 注册资源——由 PSBFile::load 的 RawCallback 调用
    void addResource(const ttstr &domain,
                     const std::string &name,
                     PSBFile *file, int index) {
        ttstr key = domain + "/" + ttstr(name.c_str());
        resources[key] = {file, index};
    }
};
```

PSBMedia 的注册同样使用回调模式：

```cpp
// 文件：cpp/plugins/psbfile/main.cpp 第 150-151 行
static PSBMedia *psbMedia = nullptr;

static void initPsbFile() {
    psbMedia = new PSBMedia();
    TVPRegisterStorageMedia(psbMedia);
}

static void deInitPsbFile() {
    TVPUnregisterStorageMedia(psbMedia);
    psbMedia->Release();
}

NCB_PRE_REGIST_CALLBACK(initPsbFile);
NCB_POST_UNREGIST_CALLBACK(deInitPsbFile);
```

---

## 五、两插件对比分析

psdfile 和 psbfile 都是"文件格式解析 + 虚拟文件系统"类型的插件，但在架构、注册策略和资源管理上有明显差异。下面从多个维度进行对比：

### 5.1 架构对比

| 对比维度 | psdfile | psbfile |
|---------|---------|---------|
| **代码规模** | ~1400 行（psdclass.cpp 894 + main.cpp 260 + psdparse 库） | ~410 行（main.cpp 151 + PSBFile.h 108 + PSBMedia.h 46 + PSBFile.cpp） |
| **层次结构** | 三层（psdparse → PSD → PSDStorage） | 两层（PSBFile → PSBMedia） |
| **继承关系** | PSD 继承 psd::PSDFile（底层解析库） | PSBFile 自包含（无外部解析库继承） |
| **虚拟协议** | `psd://` | `psb://` |
| **路径格式** | 支持按路径和按 ID 两种格式 | 仅按资源名 |
| **文件格式** | Adobe Photoshop PSD（开放格式，有公开规范） | E-mote PSB/PIMG（私有格式，需逆向） |

### 5.2 ncbind 注册策略对比

| 特性 | psdfile | psbfile |
|------|---------|---------|
| **Factory** | ✅ `Factory(&ClassT::factory)` — 需要 `objthis` | ✅ `Factory(&PSBFileFactory::factory)` — 支持双模构造 |
| **Variant 常量** | ✅ ~50 个（颜色模式、混合模式、图层类型） | ❌ 无常量注册 |
| **Property** | ✅ 6 个只读属性（INTPROP 宏） | ❌ 无属性注册 |
| **NCB_METHOD** | ✅ 10 个方法 | ✅ 3 个方法 |
| **RawCallback** | ❌ 不使用 | ✅ 2 个（root, load） |
| **NCB_SET_CONVERTOR** | ❌ 不使用 | ✅ `PSBFileConvertor<PSBFile>` |
| **TYPECONV** | ✅ `NCB_TYPECONV_CAST_INTEGER`（枚举→整数） | ❌ 不使用 |

从注册策略可以看出设计理念的不同：
- **psdfile** 走"丰富 API"路线——提供大量常量和属性，让 TJS 脚本可以精细操控 PSD 的每个细节
- **psbfile** 走"最小 API"路线——只暴露加载和资源访问的核心功能，大部分操作通过 `psb://` 虚拟文件系统完成

### 5.3 资源管理策略对比

```
psdfile 资源管理：                       psbfile 资源管理：
┌──────────────────────┐               ┌──────────────────────┐
│ PSD 对象被 TJS 持有   │               │ PSBFile 对象被 TJS 持有│
│ (强引用)              │               │ (强引用)              │
├──────────────────────┤               ├──────────────────────┤
│ PSDStorage 弱引用映射  │               │ PSBMedia 资源映射     │
│ + 强引用缓存(1个)     │               │ (直接引用资源数据)     │
├──────────────────────┤               ├──────────────────────┤
│ OnCompact 清理缓存    │               │ 无 OnCompact 支持     │
│ PSD 析构时自动注销    │               │ 需手动管理            │
├──────────────────────┤               ├──────────────────────┤
│ 自动加载：psd:// 访问  │               │ 手动加载：必须先调用   │
│ 未知 PSD 时自动创建   │               │ PSBFile.load()       │
└──────────────────────┘               └──────────────────────┘
```

关键区别：
- **psdfile** 支持**自动加载**——当通过 `psd://` 路径访问一个未加载的 PSD 文件时，PSDStorage 会自动执行 `new PSD()` 并调用 `load()`
- **psbfile** 要求**手动加载**——必须先在 TJS 脚本中创建 `PSBFile` 对象并调用 `load()` 方法，之后才能通过 `psb://` 路径访问资源
- **psdfile** 实现了 `tTVPCompactEventCallbackIntf` 接口，可以在内存不足时自动清理缓存
- **psbfile** 没有实现内存压缩回调，资源数据会一直保留在内存中直到 PSBFile 对象被销毁

### 5.4 使用场景对比

```javascript
// === psdfile 典型使用 ===

// 方式一：直接通过对象 API
var psd = new PSD();
psd.load("character.psd");
System.inform("宽度=" + psd.width + " 高度=" + psd.height);
System.inform("图层数=" + psd.layer_count);

// 遍历所有图层
for (var i = 0; i < psd.layer_count; i++) {
    var info = psd.getLayerInfo(i);
    if (info.layer_type == PSD.LAYER_TYPE_NORMAL) {
        System.inform("图层[" + i + "] " + info.name
            + " 位置=(" + info.left + "," + info.top + ")"
            + " 可见=" + info.visible);
    }
}

// 方式二：通过虚拟文件系统（更常用）
var layer = new Layer(window, null);
layer.loadImages("psd://character.psd/root/face/smile.bmp");
// 不需要手动创建 PSD 对象


// === psbfile 典型使用 ===

// 必须先手动加载
var psb = new PSBFile("animation.pimg");
System.inform("文件类型=" + psb.getType());
System.inform("资源数=" + psb.getResourceCount());

// 获取对象树（类似 JSON 的结构）
var root = psb.root;

// 通过虚拟文件系统访问内部资源
var tex = new Layer(window, null);
tex.loadImages("psb://animation.pimg/texture_00.png");
```

---

## 动手实践

### 练习一：跟踪 PSD 虚拟路径解析

在阅读源码时，跟踪以下路径的完整解析流程：

**输入路径**：`psd://portrait.psd/root/body/arm_left.bmp`

步骤：
1. 打开 `cpp/plugins/psdfile/main.cpp`，找到 `PSDStorage::CheckExistentStorage` 方法
2. 确认域名拆分逻辑：`portrait.psd` 是域名，`root/body/arm_left.bmp` 是路径
3. 打开 `cpp/plugins/psdfile/psdclass.cpp` 第 640 行的 `PSD::CheckExistentStorage`
4. 跟踪路径格式一的处理分支：
   - `pname` = `"root/body/"` — 路径部分
   - `fname` = `"arm_left.bmp"` — 文件名部分
   - `basename` = `"arm_left"` — 去掉 `.bmp` 后缀
5. 确认 `pathMap["root/body/"]` 中查找 `"arm_left"` 的逻辑

**输入路径**：`psd://portrait.psd/id/42.bmp`

步骤：
1. 确认 `id/` 前缀匹配
2. 提取数字 `42`
3. 在 `layerIdIdxMap` 中查找

### 练习二：对比两个插件的 CMakeLists.txt

```cmake
# 文件：cpp/plugins/psdfile/CMakeLists.txt（16 行）
add_library(psdfile STATIC)
target_sources(psdfile PRIVATE
    main.cpp
    psdclass.cpp
    psdparse/psdfile.cpp
    psdparse/psdlayer.cpp
    psdparse/psdparse.cpp
    psdparse/psdimage.cpp
    psdparse/psdresource.cpp
    psdparse/psdutil.cpp
    psdparse/psdbmp.cpp
    psdparse/psddesc.cpp
)
target_include_directories(psdfile PRIVATE ${KRKR2CORE_PATH})
target_link_libraries(psdfile Boost::boost)  # psdparse 依赖 boost 内存映射
```

```cmake
# 文件：cpp/plugins/psbfile/CMakeLists.txt（22 行）
add_library(psbfile STATIC)
target_sources(psbfile PRIVATE
    main.cpp
    PSBFile.cpp
    PSBMedia.cpp
    types/PSBArray.cpp
    types/PSBBool.cpp
    types/PSBCollection.cpp
    # ... 更多类型解析源文件
    resources/PSBResource.cpp
    resources/PSBResourceManager.cpp
)
target_include_directories(psbfile PRIVATE ${KRKR2CORE_PATH})
# psbfile 不需要外部依赖
```

观察要点：
- psdfile 依赖 `Boost::boost`（通过 vcpkg 管理），psbfile 无外部依赖
- psdfile 的源文件集中在 `psdparse/` 子目录，psbfile 按类型拆分为 `types/` 和 `resources/`
- 两者都是 `STATIC` 库，最终链接到 `krkr2plugin`

---

## 对照项目源码

相关文件路径和行号索引：

| 文件 | 行号范围 | 内容 |
|------|---------|------|
| `cpp/plugins/psdfile/psdparse/psdfile.h` | 1-47 | psd::PSDFile 基类定义 |
| `cpp/plugins/psdfile/psdclass.h` | 1-233 | PSD 桥接类声明 |
| `cpp/plugins/psdfile/psdclass.cpp` | 108-112 | PSD::factory 工厂方法 |
| `cpp/plugins/psdfile/psdclass.cpp` | 124-148 | PSD::load 加载逻辑 |
| `cpp/plugins/psdfile/psdclass.cpp` | 216-301 | getLayerInfo 辞书构建 |
| `cpp/plugins/psdfile/psdclass.cpp` | 309-384 | _getLayerData 像素提取 |
| `cpp/plugins/psdfile/psdclass.cpp` | 640-705 | CheckExistentStorage 路径解析 |
| `cpp/plugins/psdfile/psdclass.cpp` | 744-793 | openLayerImage BMP 生成 |
| `cpp/plugins/psdfile/psdclass.cpp` | 801-894 | NCB_REGISTER_CLASS(PSD) 完整注册块 |
| `cpp/plugins/psdfile/main.cpp` | 15-206 | PSDStorage 类完整实现 |
| `cpp/plugins/psdfile/main.cpp` | 242-260 | initStorage/doneStorage 回调 |
| `cpp/plugins/psbfile/main.cpp` | 38-89 | getRoot/load RawCallback 实现 |
| `cpp/plugins/psbfile/main.cpp` | 98-126 | PSBFileConvertor 类型转换器 |
| `cpp/plugins/psbfile/main.cpp` | 128-148 | NCB_REGISTER_CLASS(PSBFile) |
| `cpp/plugins/psbfile/PSBFile.h` | 1-108 | PSBFile 类定义 |
| `cpp/plugins/psbfile/PSBMedia.h` | 1-46 | PSBMedia 存储媒体定义 |

---

## 常见错误与排查

### 错误一：Factory 签名不匹配

```
编译错误：cannot convert 'tjs_error (*)(MyClass**, int, tTJSVariant**)'
         to 'ncbNativeClassMethodBase::FactoryType'
```

**原因**：Factory 方法的签名缺少 `iTJSDispatch2 *objthis` 参数。ncbind 要求 Factory 方法必须有完整的四参数签名。

**错误代码**：
```cpp
// 错误：缺少 objthis 参数
static tjs_error factory(MyClass **result, tjs_int numparams,
                         tTJSVariant **params) {
    *result = new MyClass();
    return TJS_S_OK;
}
```

**修正代码**：
```cpp
// 正确：包含完整的四参数签名
static tjs_error factory(MyClass **result, tjs_int numparams,
                         tTJSVariant **params,
                         iTJSDispatch2 *objthis) {
    *result = new MyClass();
    return TJS_S_OK;
}
```

### 错误二：虚拟文件系统路径后缀错误

```
TJS 运行时错误：psd://character.psd/root/face/smile.png — file not found
```

**原因**：PSD 虚拟文件系统只支持 `.bmp` 后缀。`CheckExistentStorage` 在验证后缀时会严格检查 `TJS_strcmp(q, BMPEXT) == 0`，其中 `BMPEXT` 定义为 `".bmp"`。使用 `.png`、`.jpg` 或其他后缀都会返回 `false`。

**修正**：将路径中的后缀改为 `.bmp`：
```javascript
// 错误
layer.loadImages("psd://character.psd/root/face/smile.png");
// 正确
layer.loadImages("psd://character.psd/root/face/smile.bmp");
```

### 错误三：RawCallback 中忘记获取 NativeInstance

```
TJS 运行时崩溃：TJS_E_NATIVECLASSCRASH
```

**原因**：在 RawCallback 函数中没有正确获取 C++ 原生实例。这通常发生在忘记调用 `ncbInstanceAdaptor<T>::GetNativeInstance()` 或者传入的 `objthis` 为空时。

**错误代码**：
```cpp
// 错误：直接将 objthis 转换为 C++ 指针（完全错误的做法）
static tjs_error myCallback(tTJSVariant *result, tjs_int numparams,
                            tTJSVariant **params, iTJSDispatch2 *objthis) {
    MyClass *self = (MyClass*)objthis;  // 错误！objthis 是 TJS 对象，不是 C++ 指针
    self->doSomething();
    return TJS_S_OK;
}
```

**修正代码**：
```cpp
// 正确：使用 ncbInstanceAdaptor 获取原生实例
static tjs_error myCallback(tTJSVariant *result, tjs_int numparams,
                            tTJSVariant **params, iTJSDispatch2 *objthis) {
    MyClass *self = ncbInstanceAdaptor<MyClass>::GetNativeInstance(objthis);
    if (!self) return TJS_E_NATIVECLASSCRASH;  // 安全检查
    self->doSomething();
    return TJS_S_OK;
}
```

---

## 本节小结

1. **psdfile 采用三层架构**：psdparse（纯 C++ 解析库）→ PSD（TJS 桥接类）→ PSDStorage（`psd://` 虚拟文件系统），每层职责清晰分离
2. **PSD 类使用 Factory 模式**的原因是需要在构造时获取 `objthis`（TJS 端对象引用），默认构造器无法传递此参数
3. **Variant 常量注册**让 TJS 脚本通过 `PSD.COLOR_MODE_RGB` 等符号名访问约 50 个枚举值
4. **INTPROP 宏**结合 `#`（字符串化）和 `##`（标记粘合）预处理操作符，用一行代码完成属性名和 getter 方法的映射
5. **PSDStorage 的两级缓存**：强引用缓存（最近使用的 1 个对象）+ 弱引用映射（所有已加载对象），并实现 `OnCompact` 内存压缩回调
6. **psbfile 使用 RawCallback** 处理需要直接操作 TJS 变量的方法，以及 **NCB_SET_CONVERTOR** 注册自定义类型转换器
7. **两个插件都使用 iTVPStorageMedia** 接口实现虚拟文件系统，但 psdfile 支持自动加载而 psbfile 要求手动加载
8. **openLayerImage 将图层编码为 BMP** 格式的内存流，利用了 BMP 无压缩的特性实现快速转换

---

## 练习题与答案

### 题目 1：PSD Factory 的必要性

如果 PSD 类不需要在构造函数中接收 `objthis` 参数（即构造函数改为无参 `PSD()`），是否还需要使用 `Factory` 模式？请说明理由。

<details>
<summary>查看答案</summary>

**不需要**。如果构造函数不需要 `objthis` 参数，可以直接使用 ncbind 的默认构造机制，不声明 `Factory`：

```cpp
NCB_REGISTER_CLASS(PSD) {
    // 不调用 Factory()，ncbind 将使用默认的 new PSD() 构造
    NCB_CONSTRUCTOR(());  // 声明无参构造

    INTPROP(width);
    // ... 其他注册
}
```

`Factory` 模式存在的唯一原因是 PSD 构造函数需要特殊参数（`objthis`），而 ncbind 的默认构造路径无法传递这些引擎内部参数。`Factory` 方法的参数列表中包含 `numparams`、`params` 和 `objthis`，让你可以完全控制对象创建过程。

常见需要 Factory 的场景：
1. 构造函数需要 `objthis`（PSD 的场景）
2. 需要根据参数个数选择不同构造方式（PSBFile 的双模构造场景）
3. 构造过程可能失败，需要返回错误码而非抛出异常

</details>

### 题目 2：实现一个简单的虚拟文件系统

参考 PSDStorage 和 PSBMedia 的实现，编写一个 `txtpack://` 虚拟文件系统，它将一个简单的文本打包文件（多个文本文件连接在一起，有索引头）暴露为可通过 `txtpack://package.dat/readme.txt` 访问的虚拟文件。

<details>
<summary>查看答案</summary>

```cpp
#include "ncbind.hpp"

// 简单的文本打包文件格式：
// [4字节: 文件数N] [N个索引项: 名称(64字节)+偏移(4字节)+大小(4字节)] [数据区]

struct TextPackEntry {
    std::string name;    // 文件名
    uint32_t offset;     // 数据在文件中的偏移
    uint32_t size;       // 数据大小
};

class TextPack {
    std::vector<TextPackEntry> entries; // 索引项列表
    std::vector<uint8_t> data;         // 完整文件数据

public:
    bool load(const ttstr &filename) {
        // 通过 KiriKiri 流读取文件
        IStream *stream = TVPCreateIStream(filename, TJS_BS_READ);
        if (!stream) return false;

        // 读取文件数量
        uint32_t count;
        stream->Read(&count, 4, nullptr);

        // 读取索引
        entries.resize(count);
        for (uint32_t i = 0; i < count; i++) {
            char nameBuf[64] = {};
            stream->Read(nameBuf, 64, nullptr);
            stream->Read(&entries[i].offset, 4, nullptr);
            stream->Read(&entries[i].size, 4, nullptr);
            entries[i].name = nameBuf;
        }

        // 读取全部数据到内存
        STATSTG stat;
        stream->Stat(&stat, STATFLAG_NONAME);
        data.resize(stat.cbSize.QuadPart);
        stream->Seek({0}, STREAM_SEEK_SET, nullptr);
        stream->Read(data.data(), data.size(), nullptr);
        stream->Release();
        return true;
    }

    bool hasEntry(const std::string &name) const {
        for (auto &e : entries) {
            if (e.name == name) return true;
        }
        return false;
    }

    IStream* openEntry(const std::string &name) const {
        for (auto &e : entries) {
            if (e.name == name) {
                // 将对应区域的数据封装为 IStream
                return SHCreateMemStream(
                    data.data() + e.offset, e.size);
            }
        }
        return nullptr;
    }
};

// 存储媒体实现
class TextPackStorage : public iTVPStorageMedia {
    std::map<ttstr, TextPack*> packs; // 已注册的包

public:
    void TJS_INTF_METHOD GetName(ttstr &name) override {
        name = TJS_W("txtpack");
    }

    bool TJS_INTF_METHOD NormalizeDomainName(ttstr &name) override {
        return true;
    }

    bool TJS_INTF_METHOD NormalizePathName(ttstr &name) override {
        return true;
    }

    bool TJS_INTF_METHOD CheckExistentStorage(
        const ttstr &name) override {
        ttstr domain, path;
        // 解析 "package.dat" 和 "readme.txt"
        const tjs_char *p = TJS_strchr(name.c_str(), '/');
        if (!p) return false;
        domain = ttstr(name.c_str(), p - name.c_str());
        path = ttstr(p + 1);

        auto it = packs.find(domain);
        if (it == packs.end()) return false;

        NarrowString narrow(path);
        return it->second->hasEntry(narrow);
    }

    IStream* TJS_INTF_METHOD Open(
        const ttstr &name, tjs_uint32 flags) override {
        // 同上解析 domain/path，调用 pack->openEntry()
        // 省略重复的路径解析逻辑
        return nullptr;
    }

    void TJS_INTF_METHOD GetListAt(
        const ttstr &name, iTVPStorageLister *lister) override {
        // 列出包中所有文件名
    }

    void TJS_INTF_METHOD GetLocallyAccessibleName(
        ttstr &name) override {
        name = TJS_W("");
    }

    void TJS_INTF_METHOD AddRef() override {}
    void TJS_INTF_METHOD Release() override {}

    // 注册包
    void addPack(const ttstr &name, TextPack *pack) {
        packs[name] = pack;
    }
};

static TextPackStorage *storage = nullptr;

static void initTextPack() {
    storage = new TextPackStorage();
    TVPRegisterStorageMedia(storage);
}

static void doneTextPack() {
    TVPUnregisterStorageMedia(storage);
    delete storage;
}

NCB_PRE_REGIST_CALLBACK(initTextPack);
NCB_POST_UNREGIST_CALLBACK(doneTextPack);
```

</details>

### 题目 3：RawCallback vs NCB_METHOD 的选择

以下两个方法，哪个应该用 `NCB_METHOD` 注册，哪个应该用 `RawCallback` 注册？请说明理由。

方法 A：`int getCount()` — 返回一个整数

方法 B：`tTJSVariant buildTree()` — 递归构建一个嵌套辞书/数组的 TJS 对象树

<details>
<summary>查看答案</summary>

- **方法 A 应该用 `NCB_METHOD`**：返回类型是 `int`，这是 ncbind 可以自动转换的基本类型。使用 `NCB_METHOD(getCount)` 即可，ncbind 会自动将 `int` 包装为 `tTJSVariant` 返回给 TJS。没有理由使用更复杂的 `RawCallback`。

- **方法 B 应该用 `RawCallback`**：虽然返回类型是 `tTJSVariant`（ncbind 也能处理），但如果 `buildTree()` 内部使用了 `ncbDictionaryAccessor` 和 `ncbArrayAccessor` 构建嵌套结构，那么函数内部实际上已经在直接操作 TJS 对象了。此时有两种选择：

  **选择一**（更简单）：如果方法直接返回 `tTJSVariant`，可以用 `NCB_METHOD`：
  ```cpp
  tTJSVariant buildTree() {
      ncbDictionaryAccessor dict;
      dict.SetValue(TJS_W("name"), ttstr("root"));
      // ...
      return tTJSVariant(dict);
  }
  NCB_METHOD(buildTree);  // 可以工作
  ```

  **选择二**（更灵活）：如果需要根据参数做不同处理、或者需要处理异常返回不同错误码，用 `RawCallback`：
  ```cpp
  static tjs_error TJS_INTF_METHOD buildTree(
      tTJSVariant *result, tjs_int numparams,
      tTJSVariant **params, iTJSDispatch2 *objthis) {
      // 可以根据 numparams 做不同处理
      // 可以返回 TJS_E_FAIL 等细粒度错误码
  }
  RawCallback(TJS_W("buildTree"), &buildTree, 0);
  ```

  psbfile 选择 `RawCallback` 的原因：`load` 方法在加载成功后还要将资源注册到 PSBMedia，这个副作用逻辑适合在 RawCallback 中完成，而不是混在纯 C++ 方法中。

</details>

---

## 下一步

[layerExDraw 与 fstat 插件源码导读](./03-layerExDraw与fstat.md) — 分析扩展图层绘制插件的平台适配策略（Windows GDI+ vs 通用实现）以及文件状态查询插件的简洁实现。

