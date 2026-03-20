# layerExDraw 与 fstat 插件源码导读

> **所属模块：** M09-插件系统与开发
> **前置知识：** [ncbind 类型系统与转换](../02-ncbind绑定框架/01-类型系统与转换.md)、[类注册与宏体系](../02-ncbind绑定框架/02-类注册与宏体系.md)、[属性方法与回调绑定](../02-ncbind绑定框架/03-属性方法与回调绑定.md)、[scriptsEx 与 csvParser](./01-scriptsEx与csvParser.md)
> **预计阅读时间：** 45 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 layerExDraw 插件如何通过 `NCB_ATTACH_CLASS_WITH_HOOK` 将绘图能力注入到已有 Layer 类
2. 掌握 `NCB_GET_INSTANCE_HOOK` 自动创建模式——访问时按需创建 native 实例的惰性初始化技巧
3. 分析 GdipWrapper 模板如何包装无默认构造函数的 GDI+（Graphics Device Interface Plus，微软提供的 2D 图形库）对象
4. 理解 fstat 插件如何通过 `NCB_ATTACH_CLASS` 向已有 Storages 类批量注入文件操作方法
5. 掌握 RawCallback 与 NCB_METHOD 混合注册的设计思路和适用场景
6. 理解 TJS 预注入脚本（Pre-script）机制——在 C++ 插件中向 TJS 全局作用域注入常量

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| GDI+ | Graphics Device Interface Plus | 微软提供的 2D 图形 API，支持矢量绘图、图像处理、文字渲染等 |
| libgdiplus | libgdiplus (Mono) | Mono 项目的开源 GDI+ 实现，让 Linux/macOS 也能使用 GDI+ API |
| Instance Hook | NCB_GET_INSTANCE_HOOK | ncbind 的实例获取钩子，可在首次访问时自动创建 native 对象 |
| Attach Class | NCB_ATTACH_CLASS | 将 C++ 方法"附加"到已有 TJS 类上，不创建新类而是扩展现有类 |
| RawCallback | TJS Raw Callback | 直接操作 TJS 调用栈的底层回调方式，可实现灵活的参数处理 |
| Pre-script | NCB 预注入脚本 | 在插件 C++ 注册完成前先执行的一段 TJS 脚本，用于预定义常量等 |
| std::filesystem | C++17 Filesystem | C++17 标准库的文件系统模块，提供跨平台的路径操作和文件管理 |
| GpBitmap / GpGraphics | GDI+ Flat API 句柄 | GDI+ C 风格 API 中代表位图和绘图表面的不透明指针类型 |

---

## 一、layerExDraw 平台条件编译架构

### 1.1 为什么需要条件编译？

layerExDraw 插件的核心功能是在 KiriKiri 的 Layer（图层）对象上提供矢量绘图能力——画线、填充、文字渲染等。这些功能底层依赖 GDI+（Graphics Device Interface Plus），这是微软 Windows 平台专有的 2D 图形库。问题来了：KrKr2 是一个跨平台项目，Linux 和 macOS 上根本没有 GDI+。

解决方案是**双实现架构**：Windows 上直接调用原生 GDI+，非 Windows 平台上使用 libgdiplus——这是 Mono 项目开发的开源 GDI+ 重新实现，提供了与 Windows GDI+ 基本兼容的 C API。

### 1.2 CMakeLists.txt 条件编译结构

来看 `cpp/plugins/layerex_draw/CMakeLists.txt`（30 行）的完整内容：

```cmake
# 源码路径：cpp/plugins/layerex_draw/CMakeLists.txt
cmake_minimum_required(VERSION 3.28)

project(layerExDraw)

# 条件编译的核心：根据平台选择不同的源码目录
if(WINDOWS)
    # Windows 平台：使用 windows/ 子目录，直接调用系统 GDI+
    file(GLOB_RECURSE LAYEREX_DRAW_SRC
        "windows/*.cpp"    # Windows 专用实现
        "windows/*.h"
    )
else()
    # 非 Windows 平台：使用 general/ 子目录，依赖 libgdiplus
    file(GLOB_RECURSE LAYEREX_DRAW_SRC
        "general/*.cpp"    # 跨平台通用实现
        "general/*.h"
        "general/*.hpp"
    )
endif()

add_library(${PROJECT_NAME} STATIC ${LAYEREX_DRAW_SRC})

if(WINDOWS)
    # Windows 上链接系统自带的 gdiplus.lib
    target_link_libraries(${PROJECT_NAME} PRIVATE gdiplus)
else()
    # 非 Windows 上通过 vcpkg 获取 libgdiplus
    find_package(unofficial-libgdiplus CONFIG REQUIRED)
    target_link_libraries(${PROJECT_NAME} PRIVATE
        unofficial::libgdiplus::libgdiplus
    )
endif()
```

关键设计决策：

| 方面 | Windows | 非 Windows (Linux/macOS/Android) |
|------|---------|----------------------------------|
| 源码目录 | `windows/` | `general/` |
| GDI+ 实现 | 系统原生 `gdiplus.dll` | vcpkg 安装的 `libgdiplus` |
| 链接方式 | `target_link_libraries(gdiplus)` | `find_package` + unofficial 配置 |
| API 风格 | 可用 C++ COM 风格和 C 风格 | 仅 C 风格（Flat API） |

> **注意**：`WINDOWS` 不是 CMake 内置变量，而是 KrKr2 项目自定义的平台标志，在根 CMakeLists.txt 中根据目标平台设置。这一点在 [M02-项目构建系统深度解析](../../M02-项目构建系统深度解析/README.md) 中有详细说明。

### 1.3 目录结构对比

```
cpp/plugins/layerex_draw/
├── CMakeLists.txt            # 平台条件编译入口
├── windows/                  # Windows 专用实现
│   ├── main.cpp              # ncbind 注册（使用 C++ GDI+ 类）
│   ├── LayerExDraw.cpp       # 绘图方法实现（原生 GDI+）
│   └── LayerExDraw.hpp       # 类声明
└── general/                  # 跨平台通用实现
    ├── main.cpp              # ncbind 注册 + 类型转换器（使用 C Flat API）
    ├── LayerExDraw.cpp       # 绘图方法实现（libgdiplus C API）
    ├── LayerExDraw.hpp       # 类声明 + GpBitmap/GpGraphics 管理
    ├── DrawPath.cpp          # 路径绘制实现
    ├── gdip_cxx.h            # GDI+ C API 的 C++ 封装
    ├── gdip_cxx_brush.h      # 画刷封装
    └── gdip_cxx_pen.h        # 画笔封装
```

Windows 版直接使用 `Gdiplus::Graphics`、`Gdiplus::Bitmap` 等 C++ 类；general 版则需要手动封装 `GpGraphics*`、`GpBitmap*` 等 C 风格句柄。这种差异导致 general 版的代码量显著大于 Windows 版。

---

## 二、GdipWrapper 模板——包装无默认构造函数的 GDI+ 对象

### 2.1 问题：GDI+ 对象不能直接用于 ncbind

ncbind 注册的 C++ 类必须满足一个基本要求：**可以被默认构造**（或通过 Factory 构造）。但 GDI+ 的很多类型（如 `Pen`、`Brush`、`Font`、`Image`）在 C Flat API 中是不透明指针，没有"默认构造"的概念——你必须通过专门的创建函数（如 `GdipCreatePen1`）来获得实例。

此外，ncbind 的类型转换器（Convertor）在进行 TJS ↔ C++ 转换时，可能需要**拷贝**对象。但 GDI+ 对象也不支持拷贝构造——你只能通过 `Clone` 函数复制。

### 2.2 GdipWrapper 模板的设计

`general/main.cpp` 中定义了一个巧妙的包装模板来解决这两个问题：

```cpp
// 源码路径：cpp/plugins/layerex_draw/general/main.cpp（简化）

// GdipWrapper<T> 包装没有默认构造函数的 GDI+ 对象
// T 是 GDI+ 句柄类型，例如 GpPen*、GpBrush*、GpFont*
template <typename T>
struct GdipWrapper {
    T *instance;  // 持有的 GDI+ 句柄

    // 默认构造：instance 初始化为 nullptr
    GdipWrapper() : instance(nullptr) {}

    // 从已有句柄构造（获取所有权）
    GdipWrapper(T *p) : instance(p) {}

    // 拷贝构造：通过 Clone 复制 GDI+ 对象
    GdipWrapper(const GdipWrapper &other) : instance(nullptr) {
        if (other.instance) {
            // 调用 GDI+ 的 Clone 函数复制对象
            GdipClonePen(other.instance, &instance);  // 示意，实际根据 T 不同调用不同函数
        }
    }

    // 析构：释放 GDI+ 句柄
    ~GdipWrapper() {
        if (instance) {
            // 调用 GDI+ 的 Delete 函数释放
            GdipDeletePen(instance);  // 示意
        }
    }

    // 获取原始句柄
    T* get() const { return instance; }
    operator T*() const { return instance; }
};
```

这个模板的核心思想是**代理模式**（Proxy Pattern）：

1. **默认构造**：`instance = nullptr`，让 ncbind 可以先创建空壳对象
2. **克隆语义**：拷贝构造时调用 GDI+ 的 Clone API，而非内存复制
3. **RAII 生命周期**：析构时自动调用 GDI+ 的 Delete API 释放资源
4. **隐式转换**：`operator T*()` 让包装对象可以直接传给 GDI+ C API 函数

### 2.3 自定义类型转换器群

layerExDraw 的 `general/main.cpp` 定义了一组精心设计的类型转换器，让 TJS 脚本中的数组、字典、Layer 对象能无缝转换为 GDI+ 的各种类型。以下逐一分析：

#### PointFConvertor——数组转坐标点

```cpp
// 源码路径：cpp/plugins/layerex_draw/general/main.cpp
// 将 TJS 数组 [x, y] 转换为 GDI+ 的 PointF 结构
struct PointFConvertor {
    // TJS → C++：从 TJS 数组提取两个浮点数
    static void ToTarget(PointF &dst, const tTJSVariant &src) {
        ncbPropAccessor arr(src);   // 把 tTJSVariant 当数组访问
        dst.X = (float)arr.GetValue(0, 0.0);  // 第 0 个元素 → X
        dst.Y = (float)arr.GetValue(1, 0.0);  // 第 1 个元素 → Y
    }
    // C++ → TJS：构造包含两个元素的 TJS 数组
    static void ToSource(tTJSVariant &dst, const PointF &src) {
        // 创建 TJS Array，填入 X 和 Y
        ncbDictionaryAccessor dict;
        dict.SetValue(0, (double)src.X);
        dict.SetValue(1, (double)src.Y);
        dst = dict.GetDispatch();
    }
};
NCB_SET_CONVERTOR(PointF, PointFConvertor);  // 注册到 ncbind 类型系统
```

TJS 脚本中可以这样使用：

```javascript
// TJS 脚本示例
var layer = new Layer(window, window.primaryLayer);
// 传入数组 [100.0, 200.0]，自动转换为 PointF
layer.drawLine([10.0, 20.0], [100.0, 200.0], pen);
```

#### RectFConvertor——数组转矩形

```cpp
// 将 TJS 数组 [x, y, width, height] 转换为 GDI+ 的 RectF
struct RectFConvertor {
    static void ToTarget(RectF &dst, const tTJSVariant &src) {
        ncbPropAccessor arr(src);
        dst.X      = (float)arr.GetValue(0, 0.0);  // 左上角 X
        dst.Y      = (float)arr.GetValue(1, 0.0);  // 左上角 Y
        dst.Width  = (float)arr.GetValue(2, 0.0);  // 宽度
        dst.Height = (float)arr.GetValue(3, 0.0);  // 高度
    }
    static void ToSource(tTJSVariant &dst, const RectF &src) {
        ncbDictionaryAccessor dict;
        dict.SetValue(0, (double)src.X);
        dict.SetValue(1, (double)src.Y);
        dict.SetValue(2, (double)src.Width);
        dict.SetValue(3, (double)src.Height);
        dst = dict.GetDispatch();
    }
};
NCB_SET_CONVERTOR(RectF, RectFConvertor);
```

#### MatrixConvertor——数组转变换矩阵

```cpp
// 将 TJS 6 元素数组转换为 GDI+ 的 2x3 仿射变换矩阵
struct MatrixConvertor {
    static void ToTarget(GdipWrapper<GpMatrix> &dst, const tTJSVariant &src) {
        ncbPropAccessor arr(src);
        float m11 = (float)arr.GetValue(0, 1.0);  // 缩放 X
        float m12 = (float)arr.GetValue(1, 0.0);  // 剪切 Y
        float m21 = (float)arr.GetValue(2, 0.0);  // 剪切 X
        float m22 = (float)arr.GetValue(3, 1.0);  // 缩放 Y
        float dx  = (float)arr.GetValue(4, 0.0);  // 平移 X
        float dy  = (float)arr.GetValue(5, 0.0);  // 平移 Y
        // 创建 GDI+ Matrix 对象
        GdipCreateMatrix2(m11, m12, m21, m22, dx, dy, &dst.instance);
    }
};
NCB_SET_CONVERTOR(GdipWrapper<GpMatrix>, MatrixConvertor);
```

这个矩阵转换器展示了一个**重要特点**：它只实现了 `ToTarget`（TJS → C++）方向，没有实现 `ToSource`（C++ → TJS）。这是因为矩阵只作为输入参数传递给绘图函数，不需要从 C++ 返回给 TJS。ncbind 允许这种单向转换器。

#### ImageConvertor——三源输入的智能转换

这是最复杂的转换器，支持从三种不同的 TJS 值创建 GDI+ Image：

```cpp
// 源码路径：cpp/plugins/layerex_draw/general/main.cpp（简化）
struct ImageConvertor {
    static void ToTarget(GdipWrapper<GpImage> &dst, const tTJSVariant &src) {
        tTJSVariantType type = src.Type();

        if (type == tvtString) {
            // 情况 1：TJS 传入文件名字符串 → 从文件加载图像
            ttstr filename = src.GetString();
            // 通过 TVP 存储系统解析路径，然后创建 GDI+ Image
            GdipLoadImageFromFile(filename.c_str(), &dst.instance);
        }
        else if (type == tvtObject) {
            iTJSDispatch2 *obj = src.AsObjectNoAddRef();

            // 情况 2：传入的是 LayerExDraw 对象 → 从其内部位图克隆
            LayerExDraw *draw = ncbInstanceAdaptor<LayerExDraw>
                ::GetNativeInstance(obj);
            if (draw) {
                GdipCloneImage((GpImage*)draw->getBitmap(), &dst.instance);
                return;
            }

            // 情况 3：传入的是已包装的 Image 对象 → 直接克隆
            GdipWrapper<GpImage> *wrapped = ncbInstanceAdaptor<
                GdipWrapper<GpImage>>::GetNativeInstance(obj);
            if (wrapped && wrapped->instance) {
                GdipCloneImage(wrapped->instance, &dst.instance);
            }
        }
    }
};
NCB_SET_CONVERTOR(GdipWrapper<GpImage>, ImageConvertor);
```

这种"多源转换"设计非常实用——TJS 脚本可以传字符串、Layer 对象或 Image 对象，C++ 端都能正确处理：

```javascript
// TJS 脚本示例：三种方式创建图像源
layer.drawImage("data/background.png", 0, 0);      // 方式 1：文件名
layer.drawImage(anotherLayer, 0, 0);                // 方式 2：Layer 对象
layer.drawImage(existingImage, 0, 0);               // 方式 3：Image 对象
```

---

## 三、NCB_GET_INSTANCE_HOOK——惰性自动创建模式

### 3.1 问题：如何给已有 Layer 对象附加 native 实例？

layerExDraw 的设计是**扩展**已有的 TJS `Layer` 类，而不是创建新类。TJS 脚本中的 Layer 对象是引擎自己创建的，不经过插件的构造函数。那么问题来了：当脚本调用 `layer.drawLine(...)` 时，C++ 端的 `LayerExDraw` 实例还不存在！

`NCB_GET_INSTANCE_HOOK` 正是为这种场景设计的——它在**首次获取** native 实例时自动创建，之后复用。

### 3.2 源码分析

```cpp
// 源码路径：cpp/plugins/layerex_draw/general/main.cpp
NCB_GET_INSTANCE_HOOK(LayerExDraw) {
    // 这个宏展开为一个包含 GetInstance 方法的结构体

    // 自动创建逻辑：
    NCB_INSTANCE_GETTER(objthis) {
        // 尝试获取已有的 ClassT（即 LayerExDraw）实例
        ClassT *obj = GetNativeInstance(objthis);
        if (!obj) {
            // 首次访问：还没有 native 实例，自动创建一个
            obj = new ClassT(objthis);  // 传入 TJS 的 objthis
            SetNativeInstance(objthis, obj);  // 绑定到 TJS 对象
        }
        return obj;  // 返回（可能是刚创建的）实例
    }

    // 析构钩子：当 TJS 对象被销毁时清理
    NCB_INSTANCE_DESTRUCTOR(release) {
        // release 自动调用 delete obj
    }
};
```

这种模式可以称为"**惰性单例**"（Lazy Singleton per Object）——每个 TJS Layer 对象最多关联一个 LayerExDraw 实例，在第一次调用绘图方法时按需创建。

### 3.3 配合 NCB_ATTACH_CLASS_WITH_HOOK

有了 Instance Hook，就可以用 `NCB_ATTACH_CLASS_WITH_HOOK` 将 LayerExDraw 的方法附加到已有的 Layer 类：

```cpp
// 源码路径：cpp/plugins/layerex_draw/general/main.cpp
NCB_ATTACH_CLASS_WITH_HOOK(LayerExDraw, Layer) {
    // LayerExDraw 的所有方法都被"注入"到 TJS 的 Layer 类

    // 绘图方法
    NCB_METHOD(drawEllipse);
    NCB_METHOD(fillEllipse);
    NCB_METHOD(drawArc);
    NCB_METHOD(fillPie);
    NCB_METHOD(drawRectangle);
    NCB_METHOD(fillRectangle);
    NCB_METHOD(drawRoundRectangle);
    NCB_METHOD(fillRoundRectangle);
    NCB_METHOD(drawLine);
    NCB_METHOD(drawLines);
    NCB_METHOD(drawBezier);
    NCB_METHOD(drawBeziers);
    NCB_METHOD(drawString);
    NCB_METHOD(drawImage);
    NCB_METHOD(drawPolygon);
    NCB_METHOD(fillPolygon);
    NCB_METHOD(drawCurve);
    NCB_METHOD(fillClosedCurve);

    // 绘图路径相关
    NCB_METHOD(drawPath);
    NCB_METHOD(fillPath);

    // 变换操作
    NCB_METHOD(setTransform);
    NCB_METHOD(resetTransform);
    NCB_METHOD(rotateTransform);
    NCB_METHOD(scaleTransform);
    NCB_METHOD(translateTransform);

    // 图形状态
    NCB_METHOD(setSmoothing);
    NCB_METHOD(setClip);
    NCB_METHOD(resetClip);

    // 元文件（Metafile，一种记录绘图命令的格式）录制
    NCB_METHOD(recordMetafile);
    NCB_METHOD(stopRecord);
    NCB_METHOD(drawMetafile);

    // 更新显示
    NCB_METHOD(updateRect);
};
```

`NCB_ATTACH_CLASS_WITH_HOOK` 与普通 `NCB_ATTACH_CLASS` 的区别：

| 宏 | 实例获取方式 | 适用场景 |
|----|-------------|---------|
| `NCB_ATTACH_CLASS` | 直接从 TJS 对象取 native 实例，如果不存在就报错 | native 实例确保已存在（如 fstat） |
| `NCB_ATTACH_CLASS_WITH_HOOK` | 通过 Instance Hook 获取，不存在时可自动创建 | native 实例需要惰性创建（如 layerExDraw） |

### 3.4 自动创建的完整生命周期

```
TJS 脚本                          C++ 端
──────                          ──────
var layer = new Layer(...)       → Layer 对象创建（引擎内部）
                                  此时没有 LayerExDraw 实例

layer.drawLine(...)              → NCB_GET_INSTANCE_HOOK 触发
                                → GetNativeInstance(objthis) == null
                                → new LayerExDraw(objthis)
                                → SetNativeInstance(objthis, obj)
                                → obj->drawLine(...)  执行

layer.fillRect(...)              → NCB_GET_INSTANCE_HOOK 触发
                                → GetNativeInstance(objthis) != null
                                → 直接使用已有实例
                                → obj->fillRect(...)  执行

invalidate layer                 → TJS 对象析构
                                → NCB_INSTANCE_DESTRUCTOR 触发
                                → delete obj（释放 LayerExDraw）
```

---

## 四、NCB_REGISTER_GDIP_SUBCLASS——GDI+ 子类批量注册

### 4.1 问题：大量 GDI+ 类型如何高效注册？

layerExDraw 需要把 GDI+ 的多种类型暴露给 TJS：Pen（画笔）、SolidBrush（实心画刷）、HatchBrush（阴影画刷）、LinearGradientBrush（线性渐变画刷）、Font（字体）、StringFormat（文字格式）、GraphicsPath（绘图路径）等。每种类型都需要：

1. 注册类型转换器（`GdipTypeConvertor`）
2. 注册包装类（`GdipWrapper<GpXxx>`）
3. 注册该类的属性和方法

手动写每一个太过冗长，所以源码定义了**宏家族**来批量处理。

### 4.2 GdipTypeConvertor 模板

```cpp
// 源码路径：cpp/plugins/layerex_draw/general/main.cpp（简化）
// 通用的 GDI+ 包装类型转换器
template <typename T>
struct GdipTypeConvertor {
    // TJS 对象 → GdipWrapper<T>
    static void ToTarget(GdipWrapper<T> &dst, const tTJSVariant &src) {
        if (src.Type() == tvtObject) {
            // 从 TJS 对象中获取 native 的 GdipWrapper<T> 实例
            GdipWrapper<T> *p = ncbInstanceAdaptor<GdipWrapper<T>>
                ::GetNativeInstance(src.AsObjectNoAddRef());
            if (p) {
                // 克隆一份（深拷贝），避免所有权问题
                dst = *p;  // 触发 GdipWrapper 的拷贝构造 → Clone
            }
        }
    }
    // GdipWrapper<T> → TJS 对象
    static void ToSource(tTJSVariant &dst, const GdipWrapper<T> &src) {
        // 创建新的 ncbInstanceAdaptor 包装并返回给 TJS
        iTJSDispatch2 *adaptor = ncbInstanceAdaptor<GdipWrapper<T>>
            ::CreateAdaptor(new GdipWrapper<T>(src));
        dst = tTJSVariant(adaptor, adaptor);
        adaptor->Release();
    }
};
```

### 4.3 NCB_REGISTER_GDIP_SUBCLASS 宏展开

```cpp
// 宏定义（简化）
#define NCB_REGISTER_GDIP_SUBCLASS(name) \
    NCB_SET_CONVERTOR(GdipWrapper<Gp##name>, GdipTypeConvertor<Gp##name>); \
    NCB_REGISTER_SUBCLASS(GdipWrapper<Gp##name>)

// 使用示例——注册 Pen 类
NCB_REGISTER_GDIP_SUBCLASS(Pen) {
    // Gp##Pen → GpPen，所以注册的是 GdipWrapper<GpPen>
    Factory(&Class::Factory);  // 自定义工厂方法

    // 方法注册
    NCB_METHOD(setWidth);
    NCB_METHOD(getWidth);
    NCB_METHOD(setColor);
    NCB_METHOD(getColor);
    NCB_METHOD(setDashStyle);
    NCB_METHOD(getDashStyle);
};

// 注册 SolidBrush
NCB_REGISTER_GDIP_SUBCLASS(SolidBrush) {
    Factory(&Class::Factory);
    NCB_METHOD(setColor);
    NCB_METHOD(getColor);
};

// 注册 Font
NCB_REGISTER_GDIP_SUBCLASS(Font) {
    Factory(&Class::Factory);
    NCB_METHOD(getSize);
    NCB_METHOD(getStyle);
    NCB_METHOD(getName);
};
```

宏展开后等价于：

```cpp
// NCB_REGISTER_GDIP_SUBCLASS(Pen) 展开为：

// 步骤 1：注册类型转换器——让 tTJSVariant ↔ GdipWrapper<GpPen> 互转
NCB_SET_CONVERTOR(GdipWrapper<GpPen>, GdipTypeConvertor<GpPen>);

// 步骤 2：注册子类——在 TJS 中创建 "Pen" 类
NCB_REGISTER_SUBCLASS(GdipWrapper<GpPen>) {
    Factory(&Class::Factory);
    NCB_METHOD(setWidth);
    NCB_METHOD(getWidth);
    // ...
};
```

### 4.4 GdiPlus 常量类

为了让 TJS 脚本能使用 GDI+ 的各种枚举值，源码注册了一个纯常量类：

```cpp
// 源码路径：cpp/plugins/layerex_draw/general/main.cpp
NCB_REGISTER_CLASS(GdiPlus) {
    // 平滑模式（Smoothing Mode，控制抗锯齿质量）
    Variant(TJS_W("SmoothingModeDefault"),         (tjs_int)0);
    Variant(TJS_W("SmoothingModeHighSpeed"),       (tjs_int)1);
    Variant(TJS_W("SmoothingModeHighQuality"),     (tjs_int)2);
    Variant(TJS_W("SmoothingModeAntiAlias"),       (tjs_int)4);

    // 线帽样式（Line Cap，线段端点的形状）
    Variant(TJS_W("LineCapFlat"),                  (tjs_int)0);
    Variant(TJS_W("LineCapSquare"),                (tjs_int)1);
    Variant(TJS_W("LineCapRound"),                 (tjs_int)2);
    Variant(TJS_W("LineCapTriangle"),              (tjs_int)3);

    // 虚线样式（Dash Style，虚线的图案）
    Variant(TJS_W("DashStyleSolid"),               (tjs_int)0);
    Variant(TJS_W("DashStyleDash"),                (tjs_int)1);
    Variant(TJS_W("DashStyleDot"),                 (tjs_int)2);
    Variant(TJS_W("DashStyleDashDot"),             (tjs_int)3);
    Variant(TJS_W("DashStyleDashDotDot"),          (tjs_int)4);

    // 文字对齐（String Alignment）
    Variant(TJS_W("StringAlignmentNear"),          (tjs_int)0);
    Variant(TJS_W("StringAlignmentCenter"),        (tjs_int)1);
    Variant(TJS_W("StringAlignmentFar"),           (tjs_int)2);

    // 填充模式（Fill Mode，多边形重叠区域的填充规则）
    Variant(TJS_W("FillModeAlternate"),            (tjs_int)0);
    Variant(TJS_W("FillModeWinding"),              (tjs_int)1);

    // ... 实际源码中约 120 个枚举常量
};
```

TJS 脚本中通过 `GdiPlus.SmoothingModeAntiAlias` 访问这些常量：

```javascript
// TJS 脚本示例
var layer = new Layer(window, window.primaryLayer);
layer.setSmoothing(GdiPlus.SmoothingModeAntiAlias);  // 开启抗锯齿

var pen = new Pen(0xFFFF0000, 2.0);  // 红色、2像素宽
pen.setDashStyle(GdiPlus.DashStyleDash);  // 设为虚线

layer.drawRectangle(10, 10, 200, 100, pen);
```

---

## 五、LayerExDraw 核心类——GpBitmap/GpGraphics 管理与绘图 API

### 5.1 类架构概览

`LayerExDraw` 继承自 `layerExBase`（图层扩展基类），其核心职责是管理 GDI+ 的 `GpBitmap`（位图画布）和 `GpGraphics`（绘图上下文），并提供 20+ 绘图方法。

```cpp
// 源码路径：cpp/plugins/layerex_draw/general/LayerExDraw.hpp（简化）
class LayerExDraw : public layerExBase {
private:
    // GDI+ 核心对象
    GpBitmap   *bitmap;     // 位图画布——像素数据存储在这里
    GpGraphics *graphics;   // 绘图上下文——所有绘图操作通过它执行

    // 变换系统
    GpMatrix   *world;      // 世界变换矩阵（仿射变换：平移、旋转、缩放）

    // 图层像素数据指针
    unsigned char *buffer;  // 指向 TJS Layer 的像素缓冲区
    int width, height;      // 图层尺寸

public:
    // 构造函数：从 TJS Layer 获取像素缓冲区，创建 GDI+ 画布
    LayerExDraw(iTJSDispatch2 *objthis);
    ~LayerExDraw();

    // ── 基础图形 ──
    void drawLine(PointF p1, PointF p2, GdipWrapper<GpPen> pen);
    void drawLines(tTJSVariant points, GdipWrapper<GpPen> pen);
    void drawRectangle(float x, float y, float w, float h,
                       GdipWrapper<GpPen> pen);
    void fillRectangle(float x, float y, float w, float h,
                       GdipWrapper<GpBrush> brush);
    void drawEllipse(float x, float y, float w, float h,
                     GdipWrapper<GpPen> pen);
    void fillEllipse(float x, float y, float w, float h,
                     GdipWrapper<GpBrush> brush);
    void drawArc(float x, float y, float w, float h,
                 float startAngle, float sweepAngle,
                 GdipWrapper<GpPen> pen);
    void fillPie(float x, float y, float w, float h,
                 float startAngle, float sweepAngle,
                 GdipWrapper<GpBrush> brush);

    // ── 高级图形 ──
    void drawBezier(PointF p1, PointF p2, PointF p3, PointF p4,
                    GdipWrapper<GpPen> pen);
    void drawPolygon(tTJSVariant points, GdipWrapper<GpPen> pen);
    void fillPolygon(tTJSVariant points, GdipWrapper<GpBrush> brush);
    void drawCurve(tTJSVariant points, GdipWrapper<GpPen> pen);
    void fillClosedCurve(tTJSVariant points, GdipWrapper<GpBrush> brush);

    // ── 路径绘制 ──
    void drawPath(GdipWrapper<GpPath> path, GdipWrapper<GpPen> pen);
    void fillPath(GdipWrapper<GpPath> path, GdipWrapper<GpBrush> brush);

    // ── 文字与图像 ──
    void drawString(ttstr text, GdipWrapper<GpFont> font,
                    PointF origin, GdipWrapper<GpBrush> brush);
    void drawImage(GdipWrapper<GpImage> image, float x, float y);

    // ── 变换操作 ──
    void setTransform(GdipWrapper<GpMatrix> matrix);
    void resetTransform();
    void rotateTransform(float angle);
    void scaleTransform(float sx, float sy);
    void translateTransform(float dx, float dy);

    // ── 元文件录制（Metafile，记录绘图命令的矢量格式） ──
    void recordMetafile(ttstr filename, int width, int height);
    void stopRecord();
    void drawMetafile(ttstr filename, float x, float y);

    // ── 显示更新 ──
    void updateRect(int x, int y, int w, int h);

    // 获取内部位图（供 ImageConvertor 使用）
    GpBitmap* getBitmap() const { return bitmap; }
};
```

### 5.2 像素缓冲区共享机制

LayerExDraw 的关键设计是**共享 TJS Layer 的像素缓冲区**，而不是维护独立的画布：

```cpp
// LayerExDraw 构造函数核心逻辑（简化）
LayerExDraw::LayerExDraw(iTJSDispatch2 *objthis) : layerExBase(objthis) {
    // 从 TJS Layer 对象获取像素缓冲区信息
    buffer = getBufferStart();  // 继承自 layerExBase
    width  = getWidth();
    height = getHeight();

    // 在 Layer 的像素缓冲区上创建 GDI+ Bitmap（零拷贝！）
    GdipCreateBitmapFromScan0(
        width, height,         // 尺寸
        width * 4,             // stride（每行字节数，BGRA 格式）
        PixelFormat32bppARGB,  // 像素格式
        buffer,                // 直接使用 Layer 的缓冲区
        &bitmap                // 输出 GpBitmap*
    );

    // 在 Bitmap 上创建 Graphics 绘图上下文
    GdipGetImageGraphicsContext((GpImage*)bitmap, &graphics);

    // 初始化世界变换矩阵为单位矩阵
    GdipCreateMatrix(&world);
}
```

这种设计的好处：
- **零拷贝**：GDI+ 直接在 Layer 的像素上绘图，不需要额外的 blit 操作
- **即时可见**：绘图完成后调用 `updateRect()` 通知引擎刷新，画面立刻更新
- **内存高效**：不需要额外的帧缓冲区

### 5.3 GDI+ 生命周期管理

GDI+ 库本身需要全局初始化和反初始化。layerExDraw 使用 ncbind 的注册回调来管理：

```cpp
// 源码路径：cpp/plugins/layerex_draw/general/main.cpp
static ULONG_PTR gdiplusToken;  // GDI+ 初始化令牌

// 插件注册前：初始化 GDI+ 库
NCB_PRE_REGIST_CALLBACK(initGdiPlus) {
    GdiplusStartupInput input;
    GdiplusStartup(&gdiplusToken, &input, nullptr);
    // 此时 GDI+ 全局状态已就绪
}

// 插件注销后：关闭 GDI+ 库
NCB_POST_UNREGIST_CALLBACK(deInitGdiPlus) {
    GdiplusShutdown(gdiplusToken);
    // 释放 GDI+ 全局资源
}
```

执行顺序：

```
引擎启动
  → NCB_PRE_REGIST_CALLBACK(initGdiPlus)    // GDI+ 初始化
  → NCB_REGISTER_CLASS(GdiPlus)             // 注册常量类
  → NCB_REGISTER_GDIP_SUBCLASS(Pen)         // 注册 Pen 等类型
  → NCB_REGISTER_GDIP_SUBCLASS(SolidBrush)  // 注册 SolidBrush
  → ... 其他类型注册 ...
  → NCB_ATTACH_CLASS_WITH_HOOK(LayerExDraw)  // 注册绘图方法

引擎关闭
  → NCB_POST_UNREGIST_CALLBACK(deInitGdiPlus)  // GDI+ 关闭
```

> **要点**：`NCB_PRE_REGIST_CALLBACK` 确保 GDI+ 在任何类型注册之前初始化，因为 Factory 函数可能在注册时就创建 GDI+ 对象。`NCB_POST_UNREGIST_CALLBACK` 确保所有 GDI+ 对象释放后才关闭库。这是对称的资源管理模式。

---

## 六、fstat 插件架构——向 Storages 类注入文件操作

### 6.1 设计目标

fstat 插件的目标是**扩展 TJS 内置的 `Storages` 类**，添加文件状态查询（fstat）、目录遍历、文件操作等功能。原版 KiriKiri 的 Storages 类功能有限，fstat 插件弥补了这些不足。

### 6.2 NCB_ATTACH_CLASS 基本注册

与 layerExDraw 不同，fstat 不需要 Instance Hook（因为 `StoragesFstat` 是无状态的工具类，所有方法都是静态的）：

```cpp
// 源码路径：cpp/plugins/fstat/main.cpp
NCB_ATTACH_CLASS(StoragesFstat, Storages) {
    // 将 StoragesFstat 的方法附加到 TJS 的 Storages 类

    // ─── RawCallback 方法（需要直接操作 TJS 调用栈） ───
    RawCallback(TJS_W("fstat"),           &Class::fstat,           0);
    RawCallback(TJS_W("getTime"),         &Class::getTime,         0);
    RawCallback(TJS_W("setTime"),         &Class::setTime,         0);
    RawCallback(TJS_W("dirtree"),         &Class::dirtree,         0);
    RawCallback(TJS_W("selectDirectory"), &Class::selectDirectory, 0);
    RawCallback(TJS_W("getMD5HashString"),&Class::getMD5HashString,0);
    RawCallback(TJS_W("searchPath"),      &Class::searchPath,      0);

    // ─── NCB_METHOD 方法（自动参数转换） ───
    NCB_METHOD(dirlist);
    NCB_METHOD(dirlistEx);
    NCB_METHOD(exportFile);
    NCB_METHOD(deleteFile);
    NCB_METHOD(moveFile);
    NCB_METHOD(copyFile);
    NCB_METHOD(createDirectory);
    NCB_METHOD(removeDirectory);
    NCB_METHOD(isExistentDirectory);
    NCB_METHOD(getFileSize);
    NCB_METHOD(getFileSizeStr);
    NCB_METHOD(getFileAttribute);
    NCB_METHOD(isExistent);
    NCB_METHOD(isWritable);
    NCB_METHOD(isArchive);
    NCB_METHOD(isDirectory);
    NCB_METHOD(getFullPath);
    NCB_METHOD(getShortPath);
    NCB_METHOD(getFileExt);
    NCB_METHOD(getFileName);
    NCB_METHOD(getFileDir);
    NCB_METHOD(getCanonicalPath);
    NCB_METHOD(truncateFile);
    NCB_METHOD(setFileAttributes);
    NCB_METHOD(resetFileAttributes);
    NCB_METHOD(getFileAttributes);
    NCB_METHOD(changeDirectory);
};
```

### 6.3 RawCallback 与 NCB_METHOD 混合注册——设计思路

为什么同一个插件要混合使用两种注册方式？各有不同的适用场景：

| 方面 | RawCallback | NCB_METHOD |
|------|------------|------------|
| 参数处理 | 手动从 `tTJSVariant *param[]` 提取 | 自动类型转换 |
| 返回值 | 手动设置 `*result = ...` | 自动转换返回类型 |
| 灵活性 | 完全控制参数个数和类型 | 固定签名 |
| 代码量 | 多（需要手动解包） | 少（一行注册） |
| 适用场景 | 变参函数、复杂返回值、需要 objthis | 固定签名的简单方法 |

以 `fstat` 方法为例，它必须用 RawCallback 因为返回值是复杂的 TJS 字典：

```cpp
// 源码路径：cpp/plugins/fstat/main.cpp（简化）
static tjs_error TJS_INTF_METHOD fstat(
    tTJSVariant *result,      // 返回值
    tjs_int numparams,        // 参数个数
    tTJSVariant **param,      // 参数数组
    iTJSDispatch2 *objthis    // 调用者对象
) {
    if (numparams < 1) return TJS_E_BADPARAMCOUNT;

    // 获取文件路径参数
    ttstr filename = *param[0];
    ttstr native = TVPGetLocallyAccessibleName(filename);

    // 使用 std::filesystem 获取文件状态
    std::error_code ec;
    auto status = std::filesystem::status(
        native.AsStdString(), ec
    );

    if (result) {
        // 构建 TJS 字典作为返回值
        ncbDictionaryAccessor dict;

        // 文件大小
        auto fsize = std::filesystem::file_size(
            native.AsStdString(), ec
        );
        dict.SetValue(TJS_W("size"), (tjs_int64)fsize);

        // 文件类型
        dict.SetValue(TJS_W("isDirectory"),
            std::filesystem::is_directory(status));
        dict.SetValue(TJS_W("isRegularFile"),
            std::filesystem::is_regular_file(status));
        dict.SetValue(TJS_W("isSymlink"),
            std::filesystem::is_symlink(status));

        // 最后修改时间
        auto mtime = std::filesystem::last_write_time(
            native.AsStdString(), ec
        );
        // 转换为 TJS Date 对象... （使用缓存的 Date 方法）

        *result = dict.GetDispatch();
    }
    return TJS_S_OK;
}
```

而 `deleteFile` 这样的简单方法就可以用 NCB_METHOD：

```cpp
// 简单方法：固定签名，自动转换
static bool deleteFile(ttstr filename) {
    ttstr native = TVPGetLocallyAccessibleName(filename);
    std::error_code ec;
    return std::filesystem::remove(native.AsStdString(), ec);
}
// NCB_METHOD(deleteFile); 一行注册即可
```

---

## 七、Date 类缓存与 TJS 预注入脚本

### 7.1 NCB_POST_REGIST_CALLBACK 缓存 Date 方法

fstat 的 `getTime` 和 `setTime` 方法需要将 C++ 的时间戳转换为 TJS 的 `Date` 对象。直接在每次调用时通过 TJS 反射查找 `Date` 类和它的方法会非常慢（每次 `TVPExecuteExpression` + 方法查找）。解决方案是**在插件注册完成后缓存**这些方法引用：

```cpp
// 源码路径：cpp/plugins/fstat/main.cpp（简化）

// 全局缓存：Date 类的 setTime/getTime 方法引用
static iTJSDispatch2 *dateClass = nullptr;     // Date 构造函数
static iTJSDispatch2 *dateSetTime = nullptr;   // Date.prototype.setTime
static iTJSDispatch2 *dateGetTime = nullptr;   // Date.prototype.getTime

NCB_POST_REGIST_CALLBACK(cacheDateMethods) {
    // 通过 TJS 表达式获取 Date 类
    tTJSVariant val;
    TVPExecuteExpression(TJS_W("Date"), &val);
    dateClass = val.AsObject();
    dateClass->AddRef();  // 增加引用计数，防止被 GC

    // 获取 Date.prototype.setTime 方法
    tTJSVariant method;
    dateClass->PropGet(
        0,
        TJS_W("setTime"),
        nullptr,
        &method,
        dateClass
    );
    dateSetTime = method.AsObject();
    dateSetTime->AddRef();

    // 同理获取 getTime...
    dateClass->PropGet(
        0,
        TJS_W("getTime"),
        nullptr,
        &method,
        dateClass
    );
    dateGetTime = method.AsObject();
    dateGetTime->AddRef();
}
```

然后在 `getTime` RawCallback 中就可以直接使用缓存的方法：

```cpp
static tjs_error TJS_INTF_METHOD getTime(
    tTJSVariant *result, tjs_int numparams,
    tTJSVariant **param, iTJSDispatch2 *objthis
) {
    ttstr filename = *param[0];
    // 获取文件修改时间（std::filesystem）
    auto mtime = std::filesystem::last_write_time(
        TVPGetLocallyAccessibleName(filename).AsStdString()
    );

    // 创建 Date 对象并设置时间
    tTJSVariant dateObj;
    dateClass->CreateNew(0, nullptr, nullptr, &dateObj, 0, nullptr, dateClass);

    // 用缓存的 setTime 方法设置时间戳
    tTJSVariant timeVal((tjs_int64)mtime_to_epoch_ms(mtime));
    tTJSVariant *args[] = { &timeVal };
    dateSetTime->FuncCall(0, nullptr, nullptr, nullptr, 1, args,
                          dateObj.AsObjectNoAddRef());

    if (result) *result = dateObj;
    return TJS_S_OK;
}
```

### 7.2 TJS 预注入脚本——StoragesFstatPreScript

fstat 还展示了一个独特的技巧——在 C++ 注册完成**之前**，先向 TJS 全局作用域注入一段脚本来定义文件属性常量：

```cpp
// 源码路径：cpp/plugins/fstat/main.cpp
// 定义预注入脚本内容
static const tjs_char *StoragesFstatPreScript =
    TJS_W("global.FILE_ATTRIBUTE_READONLY  = 0x00000001;\n")
    TJS_W("global.FILE_ATTRIBUTE_HIDDEN    = 0x00000002;\n")
    TJS_W("global.FILE_ATTRIBUTE_SYSTEM    = 0x00000004;\n")
    TJS_W("global.FILE_ATTRIBUTE_DIRECTORY = 0x00000010;\n")
    TJS_W("global.FILE_ATTRIBUTE_ARCHIVE   = 0x00000020;\n")
    TJS_W("global.FILE_ATTRIBUTE_DEVICE    = 0x00000040;\n")
    TJS_W("global.FILE_ATTRIBUTE_NORMAL    = 0x00000080;\n")
    TJS_W("global.FILE_ATTRIBUTE_TEMPORARY = 0x00000100;\n");

// 使用 NCB_PRE_REGIST_CALLBACK 在注册前执行脚本
NCB_PRE_REGIST_CALLBACK(injectFstatConstants) {
    TVPExecuteScript(StoragesFstatPreScript);
}
```

执行后，TJS 脚本就可以直接使用这些常量：

```javascript
// TJS 脚本示例
var attr = Storages.getFileAttributes("save/game01.dat");
if (attr & FILE_ATTRIBUTE_READONLY) {
    System.inform("文件是只读的！");
}
if (attr & FILE_ATTRIBUTE_HIDDEN) {
    System.inform("文件是隐藏的！");
}
```

**为什么用预注入脚本而不是 `Variant()` 注册常量？**

| 方式 | 作用域 | 访问方式 | 适用场景 |
|------|--------|---------|---------|
| `Variant()` 在 `NCB_REGISTER_CLASS` 中 | 类作用域 | `ClassName.CONST_NAME` | 逻辑上属于某个类的常量 |
| 预注入脚本 `global.XXX` | 全局作用域 | `CONST_NAME`（直接使用） | 与 Windows API 对齐的全局常量 |

fstat 选择全局注入是为了**与 Windows API 风格保持一致**——在 Win32 编程中，`FILE_ATTRIBUTE_READONLY` 等是全局宏定义，开发者习惯直接使用而不加类前缀。

---

## 八、std::filesystem 跨平台与 stub 方法模式

### 8.1 C++17 std::filesystem 的使用

fstat 大量使用 C++17 的 `std::filesystem`（文件系统库）来实现跨平台文件操作。这比直接调用平台 API（如 Windows 的 `GetFileAttributes` 或 POSIX 的 `stat`）要简洁得多：

```cpp
// 源码路径：cpp/plugins/fstat/main.cpp

// 目录列表——使用 std::filesystem::directory_iterator
static tTJSVariant dirlist(ttstr path) {
    ttstr native = TVPGetLocallyAccessibleName(path);
    ncbDictionaryAccessor arr;  // 创建 TJS 数组
    int idx = 0;

    std::error_code ec;
    for (auto &entry : std::filesystem::directory_iterator(
             native.AsStdString(), ec)) {
        ttstr name(entry.path().filename().u16string());
        arr.SetValue(idx++, name);
    }
    return arr.GetDispatch();
}

// 递归目录树——使用 recursive_directory_iterator
static tjs_error TJS_INTF_METHOD dirtree(
    tTJSVariant *result, tjs_int numparams,
    tTJSVariant **param, iTJSDispatch2 *objthis
) {
    ttstr path = *param[0];
    ttstr native = TVPGetLocallyAccessibleName(path);

    ncbDictionaryAccessor arr;
    int idx = 0;

    std::error_code ec;
    for (auto &entry : std::filesystem::recursive_directory_iterator(
             native.AsStdString(), ec)) {
        ncbDictionaryAccessor item;
        item.SetValue(TJS_W("path"),
            ttstr(entry.path().u16string()));
        item.SetValue(TJS_W("isDir"),
            entry.is_directory());
        item.SetValue(TJS_W("size"),
            (tjs_int64)entry.file_size(ec));
        arr.SetValue(idx++, item.GetDispatch());
    }

    if (result) *result = arr.GetDispatch();
    return TJS_S_OK;
}

// 文件复制——一行搞定
static bool copyFile(ttstr src, ttstr dst) {
    std::error_code ec;
    return std::filesystem::copy_file(
        TVPGetLocallyAccessibleName(src).AsStdString(),
        TVPGetLocallyAccessibleName(dst).AsStdString(),
        std::filesystem::copy_options::overwrite_existing,
        ec
    );
}

// 创建目录（递归创建父目录）
static bool createDirectory(ttstr path) {
    std::error_code ec;
    return std::filesystem::create_directories(
        TVPGetLocallyAccessibleName(path).AsStdString(), ec
    );
}
```

**跨平台表现**：

| 操作 | Windows | Linux | macOS | Android |
|------|---------|-------|-------|---------|
| `directory_iterator` | Win32 FindFirstFile | POSIX opendir/readdir | POSIX | Bionic libc |
| `file_size` | GetFileSizeEx | stat.st_size | stat.st_size | stat.st_size |
| `last_write_time` | GetFileTime | stat.st_mtime | stat.st_mtime | stat.st_mtime |
| `create_directories` | CreateDirectory 递归 | mkdir -p 等价 | mkdir -p 等价 | mkdir -p 等价 |
| `copy_file` | CopyFile2 | sendfile/read+write | copyfile | read+write |

> **注意**：`std::filesystem` 在 Android NDK 中的支持从 API 29（Android 10）开始完整。KrKr2 的 Android 最低版本要求可能需要检查此兼容性。

### 8.2 Stub 方法模式——未实现功能的优雅降级

fstat 中有几个方法因为跨平台实现困难或暂时不需要而**留为 stub**（桩，即只有接口没有实现的占位方法）：

```cpp
// 源码路径：cpp/plugins/fstat/main.cpp

// 截断文件——未实现
static bool truncateFile(ttstr filename, tjs_int64 size) {
    spdlog::warn("truncateFile: not implemented");
    return false;  // 返回失败
}

// 设置文件属性——未实现
static bool setFileAttributes(ttstr filename, tjs_int attrs) {
    spdlog::warn("setFileAttributes: not implemented");
    return false;
}

// 重置文件属性——未实现
static bool resetFileAttributes(ttstr filename, tjs_int attrs) {
    spdlog::warn("resetFileAttributes: not implemented");
    return false;
}

// 获取文件属性——未实现
static tjs_int getFileAttributes(ttstr filename) {
    spdlog::warn("getFileAttributes: not implemented");
    return 0;
}

// 更改工作目录——未实现
static bool changeDirectory(ttstr path) {
    spdlog::warn("changeDirectory: not implemented");
    return false;
}

// 选择目录对话框——未实现（跨平台 UI 差异太大）
static tjs_error TJS_INTF_METHOD selectDirectory(
    tTJSVariant *result, tjs_int numparams,
    tTJSVariant **param, iTJSDispatch2 *objthis
) {
    spdlog::warn("selectDirectory: not implemented");
    if (result) *result = tTJSVariant();  // 返回 void
    return TJS_S_OK;  // 不报错，静默降级
}
```

这种 stub 模式的设计要点：

1. **不抛异常**：返回 `false`/`0`/`void` 而非 `TJS_E_NOTIMPL`，避免游戏崩溃
2. **有日志告警**：使用 `spdlog::warn()` 记录调用，方便开发者发现未实现的功能
3. **接口完整**：保持与原版 KiriKiri fstat 插件相同的方法签名，确保 API 兼容性
4. **渐进实现**：随着需求出现，可以逐个替换 stub 为真实实现

### 8.3 TemporaryFiles 类——独立注册的辅助类

fstat 还注册了一个独立的 `TemporaryFiles` 类，用于管理临时文件：

```cpp
// 源码路径：cpp/plugins/fstat/main.cpp（简化）
class TemporaryFiles {
    std::vector<ttstr> files;  // 跟踪创建的临时文件

public:
    // 创建临时文件并返回路径
    ttstr create(ttstr prefix) {
        auto path = std::filesystem::temp_directory_path()
            / (prefix.AsStdString() + "_XXXXXX");
        // 生成唯一文件名并创建文件
        ttstr result(path.u16string());
        files.push_back(result);
        return result;
    }

    // 析构时自动删除所有临时文件
    ~TemporaryFiles() {
        for (auto &f : files) {
            std::error_code ec;
            std::filesystem::remove(f.AsStdString(), ec);
        }
    }
};

// 注意：这里用的是 NCB_REGISTER_CLASS，不是 ATTACH
NCB_REGISTER_CLASS(TemporaryFiles) {
    Constructor();  // 默认构造函数
    NCB_METHOD(create);
};
```

与 `StoragesFstat` 使用 `NCB_ATTACH_CLASS`（附加到 Storages）不同，`TemporaryFiles` 用 `NCB_REGISTER_CLASS` 创建了**独立的新类**。这是因为临时文件管理器有自己的实例状态（`files` 列表），需要通过 `new TemporaryFiles()` 独立创建。

---

## 九、两插件注册模式对比分析

### 9.1 综合对比表

| 对比维度 | layerExDraw | fstat |
|---------|-------------|-------|
| **注册方式** | `NCB_ATTACH_CLASS_WITH_HOOK` | `NCB_ATTACH_CLASS` |
| **目标类** | Layer（图层） | Storages（存储管理） |
| **Instance Hook** | ✅ 有（惰性自动创建） | ❌ 无（无状态工具类） |
| **native 实例** | 有状态（GpBitmap、GpGraphics） | 无状态（所有方法都是静态的） |
| **方法注册** | 纯 NCB_METHOD | RawCallback + NCB_METHOD 混合 |
| **类型转换器** | 5 种自定义转换器 | 无自定义转换器 |
| **全局初始化** | GDI+ Startup/Shutdown | Date 方法缓存 |
| **预注入脚本** | 无 | FILE_ATTRIBUTE_* 常量 |
| **Stub 方法** | 无 | 6 个未实现方法 |
| **辅助类注册** | GdiPlus（常量）+ 多个 GDI+ 子类 | TemporaryFiles（独立新类） |
| **跨平台策略** | 双目录条件编译 | std::filesystem 统一 API |
| **外部依赖** | GDI+/libgdiplus | 仅 C++17 标准库 |

### 9.2 选型指导——何时用哪种模式

```
你要扩展已有 TJS 类？
├─ 是 → 你的方法需要自己的实例状态？
│       ├─ 是 → NCB_ATTACH_CLASS_WITH_HOOK（像 layerExDraw）
│       │       每个宿主对象按需创建一个 native 实例
│       └─ 否 → NCB_ATTACH_CLASS（像 fstat）
│               纯静态方法，不需要 native 实例
└─ 否 → NCB_REGISTER_CLASS
        创建全新的 TJS 类（像 TemporaryFiles）
```

### 9.3 方法注册方式选型

```
你的方法签名是固定的？
├─ 是 → 参数类型都能自动转换？
│       ├─ 是 → NCB_METHOD（最简洁）
│       └─ 否 → 定义自定义 Convertor，然后用 NCB_METHOD
└─ 否 → 需要变参？需要 objthis？返回复杂字典？
        └─ RawCallback（完全控制）
```

---

## 动手实践

### 练习 1：追踪 layerExDraw 的绘图调用链

打开以下源码文件，跟踪一次 `layer.drawLine([0,0], [100,100], pen)` 的完整调用链：

1. **TJS 调用入口**：`general/main.cpp` 中 `NCB_ATTACH_CLASS_WITH_HOOK(LayerExDraw, Layer)` 的 `NCB_METHOD(drawLine)` 注册
2. **Instance Hook**：`NCB_GET_INSTANCE_HOOK(LayerExDraw)` 的 `NCB_INSTANCE_GETTER` 检查并可能创建实例
3. **参数转换**：`[0,0]` → `PointFConvertor::ToTarget` → `PointF`；`pen` → `GdipTypeConvertor<GpPen>::ToTarget` → `GdipWrapper<GpPen>`
4. **方法执行**：`LayerExDraw::drawLine(PointF, PointF, GdipWrapper<GpPen>)` → `GdipDrawLine(graphics, pen.get(), ...)`
5. **显示更新**：调用 `updateRect()` 通知引擎刷新图层

记录每一步涉及的源码文件和行号范围。

### 练习 2：对比 fstat 与原版 KiriKiri 的 Storages API

查看 `cpp/plugins/fstat/main.cpp` 中所有注册的方法名，对照 KiriKiri 官方文档的 `Storages` 类 API。思考：

1. 哪些方法是原版 Storages 就有的（fstat 只是重新实现）？
2. 哪些方法是 fstat 新增的扩展？
3. stub 方法为什么不直接省略不注册？（提示：考虑游戏兼容性）

---

## 对照项目源码

| 源码文件 | 行号范围 | 内容说明 |
|---------|---------|---------|
| `cpp/plugins/layerex_draw/CMakeLists.txt` | 1-30 | 平台条件编译：`if(WINDOWS)` 选择 `windows/` 或 `general/` |
| `cpp/plugins/layerex_draw/general/main.cpp` | 1-50 | GdipWrapper 模板定义，包装无默认构造函数的 GDI+ 对象 |
| `cpp/plugins/layerex_draw/general/main.cpp` | 51-150 | PointFConvertor、RectFConvertor、MatrixConvertor、ImageConvertor |
| `cpp/plugins/layerex_draw/general/main.cpp` | 151-250 | NCB_REGISTER_GDIP_SUBCLASS 宏使用：Pen、SolidBrush、Font 等 |
| `cpp/plugins/layerex_draw/general/main.cpp` | 251-400 | NCB_REGISTER_CLASS(GdiPlus) 枚举常量（约 120 个） |
| `cpp/plugins/layerex_draw/general/main.cpp` | 401-500 | NCB_GET_INSTANCE_HOOK(LayerExDraw) 自动创建逻辑 |
| `cpp/plugins/layerex_draw/general/main.cpp` | 501-700 | NCB_ATTACH_CLASS_WITH_HOOK(LayerExDraw, Layer) 方法注册 |
| `cpp/plugins/layerex_draw/general/main.cpp` | 900-933 | NCB_PRE_REGIST_CALLBACK / NCB_POST_UNREGIST_CALLBACK |
| `cpp/plugins/layerex_draw/general/LayerExDraw.hpp` | 1-50 | LayerExDraw 类声明，GpBitmap/GpGraphics 成员 |
| `cpp/plugins/layerex_draw/general/LayerExDraw.hpp` | 51-200 | 像素缓冲区共享构造逻辑 |
| `cpp/plugins/layerex_draw/general/LayerExDraw.hpp` | 200-821 | 20+ 绘图方法、变换操作、元文件录制实现 |
| `cpp/plugins/fstat/main.cpp` | 1-50 | 头文件包含、全局 Date 缓存变量声明 |
| `cpp/plugins/fstat/main.cpp` | 51-200 | NCB_ATTACH_CLASS(StoragesFstat, Storages) 方法注册 |
| `cpp/plugins/fstat/main.cpp` | 200-500 | fstat、getTime、setTime 等 RawCallback 实现 |
| `cpp/plugins/fstat/main.cpp` | 500-750 | dirlist、dirlistEx、dirtree 等目录操作 |
| `cpp/plugins/fstat/main.cpp` | 750-900 | deleteFile、copyFile、moveFile 等文件操作 |
| `cpp/plugins/fstat/main.cpp` | 900-1000 | stub 方法 + TemporaryFiles 类 |
| `cpp/plugins/fstat/main.cpp` | 1000-1023 | NCB_POST_REGIST_CALLBACK + StoragesFstatPreScript |

---

## 常见错误与排查

### 错误 1：layerExDraw 在非 Windows 平台编译失败

**现象**：`undefined reference to GdipCreateBitmapFromScan0`

**原因**：vcpkg 没有安装 libgdiplus，或者 `find_package(unofficial-libgdiplus)` 失败。

**解决**：

```bash
# 确保 vcpkg.json 中有 libgdiplus 依赖
# 检查 vcpkg 安装状态
vcpkg list | grep gdiplus

# 手动安装（如果缺失）
vcpkg install libgdiplus:x64-linux
```

### 错误 2：fstat 的 getTime 返回错误的时间

**现象**：`Storages.getTime("file.txt")` 返回的 Date 对象时间不对。

**原因**：`std::filesystem::last_write_time` 返回的是 `file_time_type`，其 epoch（纪元，即时间起点）在不同平台上可能不同（C++17 未规定必须是 Unix epoch）。需要正确转换到 JavaScript/TJS 的毫秒时间戳。

**排查步骤**：

```cpp
// 确认 file_time_type 的 epoch
auto now_fs = std::filesystem::file_time_type::clock::now();
auto now_sys = std::chrono::system_clock::now();
// 计算两个时钟的偏差...
```

### 错误 3：NCB_ATTACH_CLASS_WITH_HOOK 忘记 Instance Hook

**现象**：使用 `NCB_ATTACH_CLASS_WITH_HOOK` 但没有定义 `NCB_GET_INSTANCE_HOOK`，编译报错。

**原因**：`_WITH_HOOK` 变体要求必须有配套的 Instance Hook 定义。

**解决**：如果不需要自动创建，改用 `NCB_ATTACH_CLASS`。如果需要，确保定义了完整的 Hook：

```cpp
// 必须定义这个 Hook
NCB_GET_INSTANCE_HOOK(YourClass) {
    NCB_INSTANCE_GETTER(objthis) {
        ClassT *obj = GetNativeInstance(objthis);
        if (!obj) {
            obj = new ClassT(objthis);
            SetNativeInstance(objthis, obj);
        }
        return obj;
    }
    NCB_INSTANCE_DESTRUCTOR(release) {}
};

// 然后才能用 WITH_HOOK 版本
NCB_ATTACH_CLASS_WITH_HOOK(YourClass, TargetTJSClass) {
    NCB_METHOD(yourMethod);
};
```

---

## 本节小结

- **layerExDraw 使用双目录条件编译**：Windows 调用原生 GDI+，非 Windows 通过 vcpkg 使用 libgdiplus，共享统一的 API 语义
- **GdipWrapper 模板**解决了 GDI+ 对象无默认构造、不可拷贝的问题，通过 nullptr 初始化 + Clone 语义 + RAII 析构实现 ncbind 兼容
- **5 种自定义类型转换器**（PointF、RectF、Matrix、Image、GdipType）让 TJS 数组/对象与 GDI+ 类型无缝互转
- **NCB_GET_INSTANCE_HOOK 实现惰性创建**：首次调用绘图方法时自动创建 LayerExDraw 实例并绑定到 TJS Layer 对象
- **NCB_ATTACH_CLASS_WITH_HOOK 与 NCB_ATTACH_CLASS 的选择**取决于是否需要 native 实例状态
- **fstat 的 RawCallback + NCB_METHOD 混合注册**：变参/复杂返回值用 RawCallback，简单方法用 NCB_METHOD
- **NCB_POST_REGIST_CALLBACK 缓存 TJS 类方法**：避免每次调用都做反射查找，显著提升性能
- **TJS 预注入脚本**机制让 C++ 插件能向 TJS 全局作用域注入常量，适合与原生 API 风格对齐的场景
- **Stub 方法模式**通过返回默认值 + spdlog 告警实现优雅降级，保持 API 兼容而不崩溃
- **std::filesystem 提供了跨平台文件操作的统一接口**，大幅减少平台条件编译的需求

---

## 练习题与答案

### 题目 1：为 layerExDraw 添加 ColorConvertor

假设你需要让 TJS 脚本传入 `[255, 128, 0, 255]`（ARGB 数组）自动转换为 GDI+ 的 `ARGB` 类型（一个 32 位整数，格式为 `0xAARRGGBB`）。请编写完整的类型转换器并注册。

<details>
<summary>查看答案</summary>

```cpp
// ColorConvertor——TJS ARGB 数组与 ARGB 整数互转
struct ColorConvertor {
    // TJS 数组 [A, R, G, B] → ARGB 整数
    static void ToTarget(ARGB &dst, const tTJSVariant &src) {
        if (src.Type() == tvtInteger) {
            // 如果直接传入整数，原样使用
            dst = (ARGB)(tjs_int)src;
            return;
        }
        // 如果传入数组 [A, R, G, B]
        ncbPropAccessor arr(src);
        int a = (int)arr.GetValue(0, 255);   // Alpha，默认不透明
        int r = (int)arr.GetValue(1, 0);     // Red
        int g = (int)arr.GetValue(2, 0);     // Green
        int b = (int)arr.GetValue(3, 0);     // Blue
        dst = (ARGB)((a << 24) | (r << 16) | (g << 8) | b);
    }

    // ARGB 整数 → TJS 数组 [A, R, G, B]
    static void ToSource(tTJSVariant &dst, const ARGB &src) {
        ncbDictionaryAccessor arr;
        arr.SetValue(0, (tjs_int)((src >> 24) & 0xFF));  // A
        arr.SetValue(1, (tjs_int)((src >> 16) & 0xFF));  // R
        arr.SetValue(2, (tjs_int)((src >>  8) & 0xFF));  // G
        arr.SetValue(3, (tjs_int)( src        & 0xFF));  // B
        dst = arr.GetDispatch();
    }
};

// 注册到 ncbind 类型系统
NCB_SET_CONVERTOR(ARGB, ColorConvertor);
```

关键设计决策：
- `ToTarget` 支持两种输入：直接整数（`0xFFFF0000`）和数组（`[255, 255, 0, 0]`），提高 API 灵活性
- `ToSource` 始终返回数组格式，便于 TJS 脚本逐分量操作颜色
- 默认 Alpha 为 255（完全不透明），符合直觉

</details>

### 题目 2：为 fstat 实现 truncateFile

fstat 中的 `truncateFile` 目前是 stub。请使用 `std::filesystem` 实现跨平台的文件截断功能。

<details>
<summary>查看答案</summary>

```cpp
#include <filesystem>
#include <fstream>

// 截断文件到指定大小
static bool truncateFile(ttstr filename, tjs_int64 size) {
    ttstr native = TVPGetLocallyAccessibleName(filename);
    std::string path = native.AsStdString();

    // 检查文件是否存在
    std::error_code ec;
    if (!std::filesystem::exists(path, ec)) {
        spdlog::warn("truncateFile: file not found: {}", path);
        return false;
    }

    // std::filesystem 没有直接的 truncate 函数
    // 使用 std::filesystem::resize_file（C++17）
    std::filesystem::resize_file(path, (uintmax_t)size, ec);
    if (ec) {
        spdlog::warn("truncateFile failed: {} ({})", 
                     ec.message(), path);
        return false;
    }

    spdlog::info("truncateFile: {} truncated to {} bytes", 
                 path, size);
    return true;
}
```

关键点：
- `std::filesystem::resize_file` 是 C++17 提供的跨平台文件截断函数
- 如果目标 `size` 小于当前文件大小，文件被截断；如果更大，文件被扩展（填充零字节）
- 使用 `std::error_code` 重载避免抛异常，与 fstat 其他方法保持一致的错误处理风格
- 跨平台实现：Windows 上调用 `SetEndOfFile`，POSIX 上调用 `ftruncate`

</details>

### 题目 3：分析 NCB_GET_INSTANCE_HOOK 的线程安全性

阅读下面的 Instance Hook 代码，分析在多线程环境下是否存在竞态条件（Race Condition，多个线程同时访问共享资源导致的不确定行为），如果存在，请提出修复方案。

```cpp
NCB_GET_INSTANCE_HOOK(LayerExDraw) {
    NCB_INSTANCE_GETTER(objthis) {
        ClassT *obj = GetNativeInstance(objthis);
        if (!obj) {
            obj = new ClassT(objthis);
            SetNativeInstance(objthis, obj);
        }
        return obj;
    }
};
```

<details>
<summary>查看答案</summary>

**竞态条件分析**：

是的，这段代码在多线程环境下存在经典的**检查-然后-执行**（Check-Then-Act）竞态：

```
线程 A                          线程 B
────────                      ────────
GetNativeInstance → null
                               GetNativeInstance → null
new ClassT(objthis)
                               new ClassT(objthis)  // 重复创建！
SetNativeInstance(obj_A)
                               SetNativeInstance(obj_B)  // 覆盖 A！
                               // obj_A 泄漏，永远不会被释放
```

**修复方案——使用互斥锁**：

```cpp
#include <mutex>

NCB_GET_INSTANCE_HOOK(LayerExDraw) {
    static std::mutex instanceMutex;  // 保护实例创建

    NCB_INSTANCE_GETTER(objthis) {
        // 双重检查锁（Double-Checked Locking）
        ClassT *obj = GetNativeInstance(objthis);
        if (!obj) {
            std::lock_guard<std::mutex> lock(instanceMutex);
            obj = GetNativeInstance(objthis);  // 再次检查
            if (!obj) {
                obj = new ClassT(objthis);
                SetNativeInstance(objthis, obj);
            }
        }
        return obj;
    }
};
```

**实际影响**：在 KrKr2 中，TJS 脚本通常在单线程中执行，所以这个竞态条件在实践中不太可能触发。但如果未来引入多线程 TJS 执行或从 C++ 工作线程访问 Layer，这就会成为真正的问题。

</details>

---

## 下一步

[需求分析与设计](../05-实战-从零开发插件/01-需求分析与设计.md) — 进入插件开发实战环节，从零开始设计并实现一个完整的 KrKr2 插件。

