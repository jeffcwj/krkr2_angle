# 与 ncbind 对比分析

> **所属模块：** M09-插件系统与开发
> **前置知识：** [ncbind 类型系统与转换](../02-ncbind绑定框架/01-类型系统与转换.md)、[类注册与宏体系](../02-ncbind绑定框架/02-类注册与宏体系.md)、[BindUtil 流式 API](./01-BindUtil流式API.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：
1. 系统对比 ncbind 和 simplebinder 在设计哲学、注册方式、类型转换、性能等维度的差异
2. 根据具体插件需求选择最合适的绑定框架
3. 理解为什么 KrKr2 项目中所有插件都使用 ncbind 而非 simplebinder
4. 掌握两个框架之间的迁移方法
5. 能够分析混合使用两个框架的场景和注意事项

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 宏注册 | Macro-based Registration | ncbind 的核心方式——通过 C++ 预处理器宏在全局作用域声明类/方法绑定 |
| 流式注册 | Fluent Registration | simplebinder 的核心方式——通过链式方法调用（`.Function().Property()`）声明绑定 |
| 自动注册 | Auto Registration | ncbind 的 `NCB_REGISTER_CLASS` 宏利用全局静态变量的构造函数在 `main()` 前自动执行注册 |
| 手动注册 | Manual Registration | simplebinder 需要在回调函数中显式创建 `BindUtil` 实例并调用注册方法 |
| 类型萃取 | Type Traits | C++ 模板元编程技术，在编译期推断类型信息（如函数参数类型、返回值类型） |
| SFINAE | Substitution Failure Is Not An Error | C++ 模板特化的核心规则——当模板参数替换失败时不报错而是尝试下一个重载 |
| 哨兵值 | Sentinel Value | 特殊占位值，ncbind 中 `(int)0` 用来表示"此属性为只读/只写" |

---

## 一、设计哲学对比

### 1.1 ncbind：声明式 + 宏驱动

ncbind 的核心设计理念是**声明式绑定**——开发者只需要用宏"声明"哪些 C++ 成员要暴露给 TJS2，框架自动生成所有胶水代码（Glue Code，即连接两个不同系统的中间代码）。这种方式深受 Boost.Python 和 SWIG（Simplified Wrapper and Interface Generator，一个自动生成多语言绑定的工具）的影响。

```cpp
// ncbind 的声明式风格 —— 看起来像"填表"
// 文件：cpp/plugins/scriptsEx.cpp（典型示例）

// 第一步：声明要绑定的类
NCB_REGISTER_CLASS(KAGParser) {
    // 第二步：逐个声明要暴露的方法
    NCB_METHOD(loadScenario);      // 加载剧本文件
    NCB_METHOD(goToLabel);         // 跳转到标签
    NCB_METHOD(getNextTag);        // 获取下一个标签
    
    // 第三步：声明属性（Property，TJS2 中可像字段一样访问的 getter/setter 对）
    NCB_PROPERTY_RO(curLine, getCurLine);      // 只读：当前行号
    NCB_PROPERTY_RO(curPos, getCurPos);        // 只读：当前位置
    NCB_PROPERTY(debugLevel, getDebugLevel,    // 读写：调试级别
                 setDebugLevel);
}
```

ncbind 的设计核心是**宏展开**——`NCB_REGISTER_CLASS` 会展开为一个继承自 `ncbRegistClass` 的匿名类，其构造函数在全局静态初始化阶段自动执行。开发者不需要关心注册时机、不需要手动管理生命周期、不需要编写任何 TJS2 的 `Dispatch`（分发器，TJS2 中负责方法查找和调用的核心对象）调用代码。

**优势：**
- 声明即绑定——代码即文档，一目了然
- 零样板代码——宏生成了所有胶水逻辑
- 编译期类型安全——参数类型不匹配直接编译报错
- 自动注册——无需在任何初始化函数中手动调用

**劣势：**
- 宏展开后代码难以调试（预处理后可能有数千行展开代码）
- 宏命名冲突风险（全局宏污染命名空间）
- 错误信息晦涩——模板错误消息经常长达数十行
- 灵活性不足——某些动态绑定场景难以实现

### 1.2 simplebinder：命令式 + 流式调用

simplebinder 的设计理念截然不同——它采用**命令式绑定**，通过链式方法调用（也叫流式 API 或 Fluent API，即每个方法返回 `*this` 使得调用可以一路"流"下去）显式描述绑定关系。这种风格更接近 LuaBridge（一个流行的 Lua/C++ 绑定库）和 pybind11（Python/C++ 绑定库）。

```cpp
// simplebinder 的命令式风格 —— 看起来像"写代码"
// 注意：这是基于 simplebinder.hpp 源码构造的示例

static void registCallback(bool isRegist) {
    if (isRegist) {
        // 在回调函数中显式创建 BindUtil 实例
        SimpleBinder::BindUtil bind;  // 初始化，获取全局 TJS2 对象

        // 逐个调用方法注册函数/属性/常量
        bind.Function("loadScenario", &KAGParser::loadScenario)
            .Function("goToLabel",    &KAGParser::goToLabel)
            .Function("getNextTag",   &KAGParser::getNextTag)
            .Property("curLine",  &KAGParser::getCurLine, nullptr)
            .Property("curPos",   &KAGParser::getCurPos,  nullptr)
            .Property("debugLevel",
                      &KAGParser::getDebugLevel,
                      &KAGParser::setDebugLevel);
    }
}

// 仍然需要用 ncbind 的宏来注册回调函数
NCB_PRE_REGIST_CALLBACK(registCallback);
```

**优势：**
- 代码可读性高——每一步操作都是显式的方法调用
- 调试友好——可以单步跟踪每个注册调用
- 灵活性强——可以在运行时动态决定注册内容
- IDE 支持好——自动补全和跳转都正常工作

**劣势：**
- 手动注册——需要在回调中显式调用
- 无类绑定——只能注册全局函数和属性，不支持 `new ClassName()` 构造
- 代码量略多——每个绑定都是完整的方法调用
- 仍依赖 ncbind——注册回调机制本身就是 ncbind 的 `NCB_PRE_REGIST_CALLBACK`

### 1.3 哲学总结

```
┌─────────────────────────────────────────────────────────┐
│              绑定框架设计哲学对比                          │
├─────────────────────────┬───────────────────────────────┤
│       ncbind             │      simplebinder             │
├─────────────────────────┼───────────────────────────────┤
│  "告诉我你要绑定什么"     │  "告诉我你要怎么绑定"          │
│  声明式 (Declarative)    │  命令式 (Imperative)          │
│  宏驱动                  │  方法调用驱动                  │
│  全自动注册              │  半自动注册（仍需宏触发）       │
│  类型安全优先            │  灵活性优先                    │
│  重度模板元编程          │  轻度模板 + 手动分发           │
│  编译期确定一切          │  运行时可动态调整              │
└─────────────────────────┴───────────────────────────────┘
```

---

## 二、注册方式对比

### 2.1 类注册

这是两个框架最大的差异点。ncbind 提供了完整的类注册机制，而 simplebinder 完全没有。

**ncbind 的类注册：**

```cpp
// 源码位置：cpp/core/plugin/ncbind.hpp 第 2281-2285 行
// NCB_REGISTER_CLASS 宏展开后的核心结构

// 开发者写的代码：
NCB_REGISTER_CLASS(MyPlugin) {
    NCB_CONSTRUCTOR((int, const tjs_char*));  // 构造函数签名
    NCB_METHOD(doWork);                        // 实例方法
    NCB_PROPERTY_RO(name, getName);            // 只读属性
}

// 宏展开后（简化）实际生成的代码：
struct ncbRegistClass_MyPlugin 
    : public ncbRegistClass<MyPlugin> 
{
    ncbRegistClass_MyPlugin() 
        : ncbRegistClass<MyPlugin>(TJS_W("MyPlugin")) 
    {
        // 以下是 {} 中的代码展开结果
        Constructor<int, const tjs_char*>();  // 注册构造函数
        Method(TJS_W("doWork"), &MyPlugin::doWork);
        Property(TJS_W("name"), &MyPlugin::getName, (int)0);
    }
};

// 全局静态实例——程序启动时自动构造，触发注册
static ncbRegistClass_MyPlugin ncbRegistClass_MyPlugin_instance;
```

注册完成后，TJS2 脚本可以：

```javascript
// TJS2 脚本中使用注册的类
var plugin = new MyPlugin(42, "hello");  // 调用 C++ 构造函数
plugin.doWork();                          // 调用 C++ 方法
var n = plugin.name;                      // 读取 C++ 属性
```

**simplebinder 无法做到这一点。** simplebinder 的 `Class()` 方法只是在 TJS2 全局空间中创建一个新的 Object（对象）作为命名空间容器，它**不是**一个可以 `new` 的类：

```cpp
// simplebinder 的 "Class" —— 实际上只是命名空间
// 源码位置：cpp/plugins/simplebinder/simplebinder.hpp 第 1150-1161 行

SimpleBinder::BindUtil bind;
bind.Class("MyNamespace")            // 创建一个 TJS2 Object，不是 Class
    .Function("doWork", &doWork)     // 往这个 Object 上挂函数
    .Property("name", &getName, nullptr);

// TJS2 中只能这样用：
// MyNamespace.doWork();   // OK —— 静态函数调用
// MyNamespace.name;       // OK —— 静态属性访问
// new MyNamespace();      // 错误！—— 这不是一个类，不能 new
```

### 2.2 构造函数绑定

| 特性 | ncbind | simplebinder |
|------|--------|-------------|
| 支持构造函数 | ✅ `NCB_CONSTRUCTOR((Types...))` | ❌ 不支持 |
| 支持工厂函数 | ✅ `NCB_CONSTRUCTOR_FACTORY(func)` | ❌ 不支持 |
| TJS2 中 `new` | ✅ `new ClassName()` | ❌ 无法使用 |
| 析构函数 | ✅ 自动调用 C++ 析构 | ❌ 不涉及 |
| 自定义销毁 | ✅ `NCB_REGISTER_SUBCLASS` | ❌ 不涉及 |

ncbind 的构造函数机制是其最强大的特性之一。它通过 `ncbNativeClassConstructor` 模板类，在 TJS2 调用 `new` 时自动提取参数、转换类型、调用 C++ 构造函数、包装为 TJS2 NativeInstance（原生实例，TJS2 中用于持有 C++ 对象指针的包装器）：

```cpp
// ncbind 构造函数注册 —— 支持多参数
// 源码参考：cpp/core/plugin/ncbind.hpp 第 1673-1690 行

NCB_REGISTER_CLASS(AudioPlayer) {
    // 单参数构造
    NCB_CONSTRUCTOR((const tjs_char*));  // AudioPlayer(filename)
}

// 或者工厂函数 —— 当构造逻辑复杂时
static AudioPlayer* createPlayer(
    iTJSDispatch2* objthis,  // TJS2 传入的 this 对象
    tjs_int numparams,       // 参数数量
    tTJSVariant** params     // 参数数组
) {
    if (numparams < 1) return nullptr;
    ttstr filename = *params[0];
    return new AudioPlayer(filename);
}

NCB_REGISTER_CLASS(AudioPlayer) {
    NCB_CONSTRUCTOR_FACTORY(createPlayer);  // 使用工厂函数
    NCB_METHOD(play);
    NCB_METHOD(stop);
}
```

simplebinder 由于架构限制，完全无法实现类注册。它的 `FunctionStore`、`PropertyStore`、`ConstantStore` 都是直接操作 `tTJSDispatch2` 对象（TJS2 的基础分发接口），而不是创建 `tTJSNativeClass`（TJS2 中用于注册原生 C++ 类的特殊类型）。

### 2.3 方法注册

两个框架都支持将 C++ 函数注册为 TJS2 可调用的方法，但实现方式差异显著：

**ncbind 的方法注册——宏 + 模板特化：**

```cpp
// ncbind 方式（三种常用宏）
NCB_REGISTER_CLASS(FileManager) {
    // 1. NCB_METHOD —— 自动推断签名，名称与 C++ 方法名相同
    NCB_METHOD(open);       // 展开为 Method(TJS_W("open"), &FileManager::open)
    
    // 2. NCB_METHOD_RAW —— 原始回调，手动处理参数
    NCB_METHOD_RAW(rawMethod, RawCallback, TJS_STATICMEMBER);
    
    // 3. NCB_STATIC_METHOD —— 静态方法
    NCB_STATIC_METHOD(getVersion);
    
    // 4. 方法别名 —— TJS2 中用不同名称调用
    Method(TJS_W("openFile"), &FileManager::open);  // TJS2 名 != C++ 名
}
```

ncbind 的方法注册内部经过复杂的模板推导。以 `NCB_METHOD(open)` 为例，展开路径是：

```
NCB_METHOD(open)
→ Method(TJS_W("open"), &FileManager::open)
→ ncbRegistClass<FileManager>::Method(name, method_ptr)
→ 创建 ncbNativeClassMethod<ClassT, MethodT> 实例
→ 其中 MethodT 通过模板特化自动推断出参数类型列表
→ 运行时调用时，paramsFunctor 逐个从 tTJSVariant 提取参数
→ 调用实际 C++ 方法，将返回值写回 tTJSVariant
```

**simplebinder 的方法注册——显式方法调用：**

```cpp
// simplebinder 方式
SimpleBinder::BindUtil bind;
bind.Function("open",       &FileManager::open)       // 实例方法
    .Function("getVersion", &FileManager::getVersion); // 静态方法

// 内部流程（简化）：
// Function("open", method_ptr)
// → 创建 FunctionStore 实例
// → FunctionStore::Link(store, "open", method_ptr) 
// → MethodInvoker<decltype(method_ptr)> 推断参数数量
// → 调用 PropSet(store, "open", funcObj, TJS_MEMBERENSURE)
// → 如果 MethodInvoker::Static 为 true，自动加 TJS_STATICMEMBER 标志
```

**关键差异对比表：**

| 维度 | ncbind | simplebinder |
|------|--------|-------------|
| 语法 | 宏：`NCB_METHOD(name)` | 方法：`.Function("name", &ptr)` |
| 名称映射 | 默认同名，可用 `Method()` 自定义 | 始终显式指定字符串名 |
| 静态方法 | 需要 `NCB_STATIC_METHOD` 或 `TJS_STATICMEMBER` | 自动检测（`MethodInvoker::Static`） |
| 参数提取 | `paramsFunctor` 模板递归 | `UNROLL_FOREACH` 宏展开 |
| 最大参数 | 理论无限（模板递归） | 最多 8 个（硬编码特化） |
| 重载支持 | 通过 `Method<Signature>()` 指定签名 | 不支持（需要手动包装） |
| Raw 回调 | `NCB_METHOD_RAW` 直接操作参数数组 | 不支持 |

### 2.4 属性注册

属性绑定（Property Binding）是将 C++ 的 getter/setter 函数对映射为 TJS2 的属性访问语法。两个框架在这方面的差异同样显著：

**ncbind 的属性注册：**

```cpp
NCB_REGISTER_CLASS(Config) {
    // 读写属性 —— getter + setter
    NCB_PROPERTY(volume, getVolume, setVolume);
    
    // 只读属性 —— 用 (int)0 哨兵值表示"无 setter"
    NCB_PROPERTY_RO(version, getVersion);
    // 展开为：Property(TJS_W("version"), &Config::getVersion, (int)0)
    // (int)0 触发特殊重载，setter 被替换为 ncbRawCallbackPropertySelector<int>
    // 该 selector 的 FuncCall 固定返回 TJS_E_ACCESSDENYED（注意原版拼写）
    
    // 只写属性 —— 用 (int)0 哨兵值表示"无 getter"
    NCB_PROPERTY_WO(password, setPassword);
    // 展开为：Property(TJS_W("password"), (int)0, &Config::setPassword)
    // getter 被替换为 ncbRawCallbackPropertySelector<int>
}
```

ncbind 的属性机制利用了 C++ 函数重载（Function Overloading）和模板特化（Template Specialization）的组合。当传入 `(int)0` 时，编译器选择 `Property` 的特殊重载版本，该版本将对应的 getter 或 setter 替换为一个固定返回"访问拒绝"错误的哑对象（Dummy Object）。

**simplebinder 的属性注册：**

```cpp
SimpleBinder::BindUtil bind;
bind.Property("volume",  &Config::getVolume, &Config::setVolume)  // 读写
    .Property("version", &Config::getVersion, nullptr)             // 只读（nullptr = 无 setter）
    .Property("password", nullptr, &Config::setPassword);          // 只写（nullptr = 无 getter）
```

simplebinder 的属性注册更直观——用 `nullptr` 表示缺失的 getter 或 setter，不需要哨兵值技巧。内部通过 `PropertyStore::Link()` 分别创建 getter 和 setter 的 `tTJSDispatch2` 对象，然后通过 `PropSetByVS` 设置到 TJS2 对象上。

**注意 simplebinder 的参数顺序陷阱：**

```cpp
// 公开 API：Property(key, getter, setter) —— getter 在前
bind.Property("volume", &getVolume, &setVolume);

// 但内部 PropertyStore::Link() 的参数顺序是：
// Link(store, key, setter, getter) —— setter 在前！
// 源码位置：cpp/plugins/simplebinder/simplebinder.hpp 第 960-1012 行

// 这意味着如果有人直接使用 PropertyStore（而非 BindUtil），
// 参数顺序容易搞反，导致读取属性时执行了写入逻辑
```

### 2.5 常量注册

| 特性 | ncbind | simplebinder |
|------|--------|-------------|
| 整型常量 | `NCB_CONSTANT(name)` | `.Constant("name", value)` |
| 字符串常量 | `NCB_CONSTANT(name)` | `.Constant("name", value)` |
| 可变全局量 | 不直接支持 | `.Variant("name", value)` |
| 注册位置 | 类内部或全局 | 全局或 Class 命名空间 |

ncbind 的常量注册通过 `NCB_CONSTANT` 宏，仅支持在 `NCB_REGISTER_CLASS` 块内部声明。常量值在编译期确定，运行时不可修改：

```cpp
NCB_REGISTER_CLASS(AudioFormat) {
    NCB_CONSTANT(FORMAT_WAV);    // 展开为 Variant("FORMAT_WAV", (tjs_int)AudioFormat::FORMAT_WAV)
    NCB_CONSTANT(FORMAT_OGG);
    NCB_CONSTANT(FORMAT_MP3);
}
```

simplebinder 提供了更灵活的选择——`Constant()` 创建不可修改的常量，`Variant()` 创建可读写的全局变量：

```cpp
SimpleBinder::BindUtil bind;
bind.Constant("FORMAT_WAV", 1)      // 不可修改 —— PropSet 带 TJS_HIDDENMEMBER 标志
    .Constant("FORMAT_OGG", 2)
    .Variant("currentFormat", 1);    // 可修改 —— PropSet 不带隐藏标志

// Constant 和 Variant 的区别在于存储方式：
// Constant: ConstantStore::Link() → tTJSVariant 直接存储值
// Variant:  直接调用 PropSet 将值设置到 TJS2 对象上
```

---

## 三、类型系统对比

### 3.1 类型转换机制

类型转换是绑定框架的核心——它负责在 C++ 类型和 TJS2 的 `tTJSVariant`（TJS2 的通用值类型，类似 JavaScript 的动态类型变量）之间自动转换。两个框架的实现方式有根本差异。

**ncbind 的类型转换——深度模板特化：**

ncbind 通过 `ncbTypeConvertor` 模板类提供了一套完整的类型转换系统。每种 C++ 类型都有对应的模板特化（Template Specialization，编译器根据类型自动选择的特定实现版本）：

```cpp
// ncbind 类型转换示例
// 源码位置：cpp/core/plugin/ncbind.hpp 第 44-115 行

// 基础类型转换 —— 通过 tTJSVariant 的隐式构造/转换
template<> struct ncbTypeConvertor::SelectConvertorType<tjs_int> {
    // tTJSVariant → tjs_int：调用 variant.AsInteger()
    // tjs_int → tTJSVariant：调用 tTJSVariant(intValue)
};

template<> struct ncbTypeConvertor::SelectConvertorType<ttstr> {
    // tTJSVariant → ttstr：调用 variant.GetString()
    // ttstr → tTJSVariant：调用 tTJSVariant(strValue)
};

// 自定义类型转换 —— 开发者可以为自己的类型添加特化
template<> struct ncbTypeConvertor::SelectConvertorType<MyColor> {
    typedef MyColor dst_type;
    static bool IsAccept(const tTJSVariant& v) {
        return v.Type() == tvtObject;  // 检查是否为对象类型
    }
    static dst_type GetData(const tTJSVariant& v) {
        // 从 TJS2 对象中提取 r, g, b 字段
        iTJSDispatch2* obj = v.AsObjectNoAddRef();
        tTJSVariant r, g, b;
        obj->PropGet(0, TJS_W("r"), nullptr, &r, obj);
        obj->PropGet(0, TJS_W("g"), nullptr, &g, obj);
        obj->PropGet(0, TJS_W("b"), nullptr, &b, obj);
        return MyColor((int)r, (int)g, (int)b);
    }
};
```

ncbind 支持的类型转换非常丰富：

| C++ 类型 | TJS2 类型 | 转换方式 |
|----------|----------|---------|
| `tjs_int`, `tjs_int32` | Integer | `AsInteger()` / `tTJSVariant(int)` |
| `tjs_real` | Real | `AsReal()` / `tTJSVariant(double)` |
| `ttstr`, `const tjs_char*` | String | `GetString()` / `tTJSVariant(str)` |
| `iTJSDispatch2*` | Object | `AsObjectNoAddRef()` |
| `tTJSVariant` | Any | 直接传递（零转换） |
| `bool` | Integer (0/1) | `IsTrue()` 方法（返回 `tjs_int`） |
| `void` | Void | 不写入返回值 |
| 自定义类型 | Object | 需要手动特化 `SelectConvertorType` |

**simplebinder 的类型转换——MethodInvoker 内联：**

simplebinder 没有独立的类型转换系统。它的 `MethodInvoker` 模板类在调用函数时直接将 `tTJSVariant*` 转换为目标类型：

```cpp
// simplebinder 类型转换
// 源码位置：cpp/plugins/simplebinder/simplebinder.hpp 第 56-68 行

// 参数提取模板 —— 从 tTJSVariant 提取特定类型
template<typename T> struct ParamExtract {
    // 默认：直接强转（依赖 tTJSVariant 的转换运算符）
    static T Get(tTJSVariant* param) {
        return static_cast<T>(*param);
    }
};

// 特殊处理：tTJSVariant 本身不需要转换
template<> struct ParamExtract<tTJSVariant> {
    static tTJSVariant Get(tTJSVariant* param) {
        return *param;  // 直接复制
    }
};

// 特殊处理：tTJSVariant 指针
template<> struct ParamExtract<tTJSVariant*> {
    static tTJSVariant* Get(tTJSVariant* param) {
        return param;   // 直接传指针
    }
};
```

### 3.2 类型安全对比

```
┌───────────────────────────────────────────────────────────────┐
│                    类型安全层次对比                              │
├───────────────────────────┬───────────────────────────────────┤
│         ncbind             │         simplebinder              │
├───────────────────────────┼───────────────────────────────────┤
│ 编译期：                   │ 编译期：                           │
│ ✅ 参数数量检查             │ ✅ 参数数量检查                    │
│ ✅ 参数类型检查             │ ⚠️ 依赖隐式转换                   │
│ ✅ 返回值类型检查           │ ✅ 返回值类型检查                  │
│ ✅ 方法签名完整匹配        │ ⚠️ 只检查参数数量                  │
│                            │                                   │
│ 运行时：                   │ 运行时：                           │
│ ✅ tTJSVariant 类型检查    │ ⚠️ 强制 static_cast               │
│ ✅ 参数不足返回错误        │ ⚠️ 参数不足时行为未定义            │
│ ✅ 自定义类型验证          │ ❌ 无自定义类型支持                │
└───────────────────────────┴───────────────────────────────────┘
```

ncbind 在类型安全方面远强于 simplebinder。ncbind 的 `paramsFunctor` 在运行时会逐个检查每个参数的类型是否与 C++ 函数签名匹配，不匹配则返回 `TJS_E_INVALIDPARAM`（参数无效错误）。而 simplebinder 的 `ParamExtract` 直接使用 `static_cast`，如果 TJS2 传入的值类型与期望的 C++ 类型不兼容，可能导致未定义行为（Undefined Behavior，C++ 中最危险的错误类型——程序可能崩溃、数据损坏或看似正常运行但结果错误）。

```cpp
// ncbind —— 类型不匹配时的安全处理
// 如果 TJS2 传入字符串，但 C++ 期望 int：
// paramsFunctor 调用 ncbTypeConvertor::IsAccept() 检查类型
// → 返回 false → 报告 TJS_E_INVALIDPARAM
// → TJS2 抛出异常："パラメータの型が不正です"（参数类型不正确）

// simplebinder —— 类型不匹配时的危险处理
// 如果 TJS2 传入字符串，但 C++ 期望 int：
// ParamExtract<int>::Get(param) 
// → static_cast<int>(*param)
// → tTJSVariant 的 operator int() 被调用
// → 字符串 "hello" 被转换为 0（静默失败！）
```

---

## 四、性能特征对比

### 4.1 编译期开销

两个框架在编译期的开销差异很大，主要体现在模板实例化（Template Instantiation，编译器为每个具体类型生成模板代码副本的过程）的深度和广度上。

**ncbind 的编译开销：**

ncbind 的宏展开会触发深层模板实例化链。以注册一个包含 5 个方法的类为例：

```
NCB_REGISTER_CLASS(MyClass) {
    NCB_METHOD(method1);  // → ncbNativeClassMethod<MyClass, void(MyClass::*)(int)>
    NCB_METHOD(method2);  // → ncbNativeClassMethod<MyClass, int(MyClass::*)(const tjs_char*)>
    NCB_METHOD(method3);  // → ncbNativeClassMethod<MyClass, void(MyClass::*)()>
    NCB_METHOD(method4);  // → ncbNativeClassMethod<MyClass, tTJSVariant(MyClass::*)(tTJSVariant)>
    NCB_METHOD(method5);  // → ncbNativeClassMethod<MyClass, bool(MyClass::*)(int, int)>
}

// 每个 NCB_METHOD 触发的模板实例化链：
// 1. ncbNativeClassMethod<ClassT, MethodT>
// 2.   → ncbNativeClassMethodBase<ClassT, MethodT>
// 3.     → ncbMethodProxy<ClassT, ReturnT, ArgTypes...>
// 4.       → paramsFunctor<N, ArgTypes...>（递归展开每个参数）
// 5.         → ncbTypeConvertor::SelectConvertorType<ArgT>（每个参数类型）
// 6.           → ncbTypeConvertor::Convertor<ArgT>
// 7. ncbNativeClassProperty（如果有属性）
// 8. ncbNativeClassConstructor<ClassT, CtorArgTypes...>（如果有构造函数）

// 总计：5 个方法 × 6-8 层模板 = 30-40 个模板实例化
```

**simplebinder 的编译开销：**

```
SimpleBinder::BindUtil bind;
bind.Function("method1", &MyClass::method1)  // → MethodInvoker<void(MyClass::*)(int)>
    .Function("method2", &MyClass::method2)  // → MethodInvoker<int(MyClass::*)(const tjs_char*)>
    ...

// 每个 Function 触发的模板实例化链：
// 1. MethodInvoker<MethodT>（单层特化）
// 2.   → ParamExtract<ArgT>（每个参数，但只是简单特化）
// 3. FunctionStore（继承 StoreUtil : tTJSDispatch）

// 总计：5 个方法 × 2-3 层模板 = 10-15 个模板实例化
```

simplebinder 的模板深度仅为 ncbind 的 1/3 到 1/2，编译速度更快。但在现代编译器（GCC 12+、Clang 15+、MSVC 19.30+）下，这个差异通常在秒级以内，对开发体验影响不大。

### 4.2 运行时开销

运行时性能差异主要在方法调用路径上。两个框架的调用链长度不同：

```
ncbind 方法调用路径（约 5-7 层函数调用）：
TJS2 调用 → tTJSDispatch2::FuncCall
→ ncbNativeClassMethod::FuncCall
→ ncbMethodProxy::Call
→ doInvokeBase
→ paramsFunctor::Exec（递归提取 N 个参数）
→ ncbTypeConvertor::GetData（每个参数转换）
→ 实际 C++ 方法

simplebinder 方法调用路径（约 3-4 层函数调用）：
TJS2 调用 → tTJSDispatch2::FuncCall
→ FunctionStore::FuncCall
→ MethodInvoker::Invoke
→ ParamExtract::Get（每个参数提取）
→ 实际 C++ 方法
```

**调用路径对比表：**

| 维度 | ncbind | simplebinder |
|------|--------|-------------|
| 调用深度 | 5-7 层 | 3-4 层 |
| 参数提取 | 模板递归 `paramsFunctor` | 宏展开 `UNROLL_FOREACH` |
| 类型检查 | 每个参数都检查 | 依赖隐式转换 |
| 虚函数调用 | 1 次（FuncCall） | 1 次（FuncCall） |
| 额外分支 | 4 种 invoke 模式选择 | 无 |
| 实例获取 | `GetNativeInstance` 查找 | 直接 `static_cast` |

在实际性能测试中，对于简单的方法调用（0-2 个参数），两者的性能差异在 10% 以内（纳秒级别）。这是因为瓶颈通常在 TJS2 的分发机制本身，而非绑定框架的转换逻辑。对于参数较多（4 个以上）的调用，ncbind 的模板递归可能略慢于 simplebinder 的宏展开，但差异仍然很小。

### 4.3 内存占用

```
ncbind 的内存模型：
┌────────────────────────────┐
│ 每个注册类：                │
│   tTJSNativeClass 实例     │  ~200-500 bytes
│   + 方法 Dispatch 对象     │  ~100 bytes × 方法数
│   + 属性 Dispatch 对象     │  ~150 bytes × 属性数
│   + 构造函数 Dispatch      │  ~100 bytes
│   + ncbRegistClass 静态链  │  ~50 bytes
├────────────────────────────┤
│ 每个实例：                  │
│   C++ 对象                 │  sizeof(Class)
│   + tTJSNativeInstance     │  ~30 bytes（指针 + 引用计数）
└────────────────────────────┘

simplebinder 的内存模型：
┌────────────────────────────┐
│ 每个注册命名空间：          │
│   tTJSDispatch2 对象       │  ~100-200 bytes
│   + FunctionStore 对象     │  ~80 bytes × 函数数
│   + PropertyStore 对象     │  ~120 bytes × 属性数
│   + ConstantStore 对象     │  ~60 bytes × 常量数
├────────────────────────────┤
│ 无实例管理                  │
│  （不支持 new，无实例开销） │
└────────────────────────────┘
```

ncbind 因为支持类实例化，每个 TJS2 创建的对象都需要额外的内存来保存 `tTJSNativeInstance` 包装器和引用计数。simplebinder 由于不支持实例化，没有这部分开销。但如果考虑到"功能等价"（即 ncbind 可以做而 simplebinder 做不到的事），这种比较意义不大。

---

## 五、实际使用场景分析

### 5.1 KrKr2 项目中的选择

通过搜索整个 KrKr2 代码库，我们发现一个重要事实：

> **simplebinder 在 KrKr2 项目中完全未被使用。所有插件（无一例外）均使用 ncbind。**

这个搜索结果值得深入分析。让我们看看各个插件的绑定方式：

```cpp
// scriptsEx.cpp —— 使用 NCB_REGISTER_CLASS + NCB_REGISTER_SUBCLASS
NCB_REGISTER_SUBCLASS(KAGParser) { ... }  // 附加到已有类
NCB_PRE_REGIST_CALLBACK(PreRegistCallback);  // 回调注册全局函数

// csvParser.cpp —— 使用 NCB_REGISTER_CLASS
NCB_REGISTER_CLASS(CSVParser) {
    NCB_CONSTRUCTOR(());  // 无参构造
    NCB_METHOD(open);
    NCB_METHOD(close);
    NCB_PROPERTY_RO(numFields, getNumFields);
    ...
}

// windowEx.cpp —— 使用 NCB_ATTACH_FUNCTION 大量绑定
NCB_ATTACH_FUNCTION(resetWindowIcon, Window, resetWindowIcon);
NCB_ATTACH_FUNCTION(setWindowIcon, Window, setWindowIcon);
// ... 50+ 个 NCB_ATTACH_FUNCTION 调用

// psdfile/ —— 使用 NCB_REGISTER_CLASS
NCB_REGISTER_CLASS(PSD) {
    NCB_CONSTRUCTOR(());
    NCB_METHOD(load);
    NCB_METHOD(getLayerType);
    ...
}

// layerExDraw/ —— 使用 NCB_REGISTER_SUBCLASS 附加到 Layer 类
NCB_REGISTER_SUBCLASS(LayerExDraw) {
    NCB_METHOD(drawText);
    NCB_METHOD(drawLine);
    ...
}

// fstat/ —— 使用 NCB_REGISTER_CLASS
NCB_REGISTER_CLASS(FileStat) {
    NCB_CONSTRUCTOR((const tjs_char*));
    NCB_PROPERTY_RO(size, getSize);
    ...
}
```

### 5.2 为什么所有插件选择 ncbind？

分析 KrKr2 中插件的绑定需求，我们可以总结出以下原因：

| 需求 | ncbind | simplebinder | 影响 |
|------|--------|-------------|------|
| `new ClassName()` 构造 | ✅ | ❌ | **致命缺陷**——大部分插件需要实例化 |
| 附加方法到已有类 | ✅ `NCB_REGISTER_SUBCLASS` | ❌ | windowEx、layerExDraw 的核心需求 |
| 工厂模式构造 | ✅ `NCB_CONSTRUCTOR_FACTORY` | ❌ | 复杂初始化场景 |
| Raw 回调 | ✅ `NCB_METHOD_RAW` | ❌ | scriptsEx 的变参方法 |
| 批量附加函数 | ✅ `NCB_ATTACH_FUNCTION` | ❌ | windowEx 的 50+ 全局函数 |
| 生命周期管理 | ✅ 自动析构 | ❌ | 有状态插件必须管理对象销毁 |
| 注册/注销回调 | ✅ `NCB_PRE_REGIST_CALLBACK` | 需借用 ncbind | 动态资源管理 |

simplebinder 的**根本限制**在于它不支持类注册。KrKr2 中几乎所有插件都需要通过 `new` 在 TJS2 中创建 C++ 对象实例（CSVParser、PSD、FileStat 等），这一点 simplebinder 无法满足。即使是那些主要注册全局函数的插件（如 windowEx），也需要 `NCB_ATTACH_FUNCTION`（将函数附加到已存在的 TJS2 类上），这同样是 simplebinder 不具备的能力。

### 5.3 simplebinder 的适用场景

虽然在 KrKr2 中未被使用，但 simplebinder 在以下场景中仍有存在价值：

**场景一：纯工具函数库**

```cpp
// 如果你只需要注册一组全局工具函数，不需要类实例化
// simplebinder 的代码更简洁

static int add(int a, int b) { return a + b; }
static int multiply(int a, int b) { return a * b; }
static double sqrt_safe(double x) { return x >= 0 ? std::sqrt(x) : 0; }

static void registMathUtils(bool isRegist) {
    if (isRegist) {
        SimpleBinder::BindUtil bind;
        bind.Function("Math_add",      &add)
            .Function("Math_multiply", &multiply)
            .Function("Math_sqrt",     &sqrt_safe)
            .Constant("Math_PI",       3.14159265358979)
            .Constant("Math_E",        2.71828182845905);
    }
}

NCB_PRE_REGIST_CALLBACK(registMathUtils);
```

**场景二：动态注册（根据运行时条件决定）**

```cpp
// ncbind 的宏注册是编译期确定的，无法根据运行时条件跳过
// simplebinder 可以在回调中做条件判断

static void registPlatformSpecific(bool isRegist) {
    if (isRegist) {
        SimpleBinder::BindUtil bind;
        
        // 根据平台注册不同函数
        #ifdef _WIN32
        bind.Function("getPlatform", &getWin32Platform)
            .Function("getRegistry", &readRegistry);
        #elif defined(__ANDROID__)
        bind.Function("getPlatform", &getAndroidPlatform)
            .Function("getJNIVersion", &getJNIVersion);
        #else
        bind.Function("getPlatform", &getGenericPlatform);
        #endif
        
        // 根据配置动态决定是否注册调试函数
        if (isDebugBuild()) {
            bind.Function("dumpMemory", &dumpMemoryStats)
                .Function("traceCall",  &traceCallStack);
        }
    }
}
```

**场景三：快速原型开发**

```cpp
// 开发初期快速验证 API 设计
// simplebinder 的流式写法更适合频繁修改

static void registPrototype(bool isRegist) {
    if (isRegist) {
        SimpleBinder::BindUtil bind;
        bind.Class("Experimental")
            .Function("feature1", &experimental_feature1)
            .Function("feature2", &experimental_feature2)
            .Variant("config", 42);  // 可变全局量，方便调试时修改
    }
}
```

---

## 六、框架迁移指南

### 6.1 从 simplebinder 迁移到 ncbind

如果你有一个使用 simplebinder 编写的插件，想要迁移到 ncbind（例如需要添加类实例化支持），以下是系统的迁移步骤：

**迁移前（simplebinder）：**

```cpp
#include "simplebinder.hpp"

// C++ 实现类
class TextProcessor {
public:
    TextProcessor() : lineCount_(0) {}
    
    void loadFile(const tjs_char* path) {
        // 加载文件逻辑
        lineCount_ = countLines(path);
    }
    
    int getLineCount() const { return lineCount_; }
    
    static const char* getVersion() { return "1.0.0"; }
    
private:
    int lineCount_;
};

// simplebinder 注册
static void registTextProcessor(bool isRegist) {
    if (isRegist) {
        SimpleBinder::BindUtil bind;
        bind.Class("TextProcessor")
            .Function("loadFile",     &TextProcessor::loadFile)
            .Function("getVersion",   &TextProcessor::getVersion)
            .Property("lineCount",    &TextProcessor::getLineCount, nullptr)
            .Constant("MAX_LINES",    100000);
    }
}

NCB_PRE_REGIST_CALLBACK(registTextProcessor);
```

**迁移后（ncbind）：**

```cpp
#include "ncbind.hpp"

// C++ 实现类（不需要修改）
class TextProcessor {
public:
    TextProcessor() : lineCount_(0) {}
    
    void loadFile(const tjs_char* path) {
        lineCount_ = countLines(path);
    }
    
    int getLineCount() const { return lineCount_; }
    
    static const char* getVersion() { return "1.0.0"; }
    
private:
    int lineCount_;
};

// ncbind 注册（替代 simplebinder）
NCB_REGISTER_CLASS(TextProcessor) {
    // 添加构造函数支持 —— simplebinder 做不到的！
    NCB_CONSTRUCTOR(());                          // 无参构造
    
    // 方法注册 —— NCB_METHOD 自动推断签名
    NCB_METHOD(loadFile);                         // 实例方法
    NCB_STATIC_METHOD(getVersion);                // 静态方法
    
    // 属性注册 —— NCB_PROPERTY_RO 对应只读属性
    NCB_PROPERTY_RO(lineCount, getLineCount);     // 只读属性
    
    // 常量 —— 在类作用域内注册
    NCB_CONSTANT(MAX_LINES);                      // 需要在类中定义 static const int MAX_LINES = 100000
}

// 不再需要 NCB_PRE_REGIST_CALLBACK —— NCB_REGISTER_CLASS 自动注册
```

**迁移后 TJS2 脚本的变化：**

```javascript
// 迁移前（simplebinder）—— 只能当命名空间用
TextProcessor.loadFile("test.txt");   // 静态调用
var count = TextProcessor.lineCount;  // 静态属性

// 迁移后（ncbind）—— 可以创建实例了！
var tp = new TextProcessor();         // 现在可以 new 了
tp.loadFile("test.txt");              // 实例方法调用
var count = tp.lineCount;             // 实例属性访问
var ver = TextProcessor.getVersion(); // 静态方法仍然可用
```

### 6.2 迁移映射表

| simplebinder 写法 | ncbind 等价写法 | 注意事项 |
|-------------------|----------------|---------|
| `bind.Function("name", &Class::method)` | `NCB_METHOD(method)` | 名称自动取 C++ 方法名 |
| `bind.Function("alias", &Class::method)` | `Method(TJS_W("alias"), &Class::method)` | 需要别名时用 `Method()` |
| `bind.Function("name", &freeFunc)` | `NCB_ATTACH_FUNCTION(name, TargetClass, freeFunc)` | 自由函数需指定目标类 |
| `bind.Property("name", &getter, &setter)` | `NCB_PROPERTY(name, getter, setter)` | 宏参数不带 `&` |
| `bind.Property("name", &getter, nullptr)` | `NCB_PROPERTY_RO(name, getter)` | 只读属性专用宏 |
| `bind.Property("name", nullptr, &setter)` | `NCB_PROPERTY_WO(name, setter)` | 只写属性专用宏 |
| `bind.Constant("name", value)` | `NCB_CONSTANT(name)` | ncbind 需要类内有同名常量 |
| `bind.Variant("name", value)` | 无直接等价 | 需要用 Raw 回调实现 |
| `bind.Class("NS").Function(...)` | 无直接等价 | ncbind 不支持命名空间式注册 |

### 6.3 从 ncbind 迁移到 simplebinder（不推荐）

理论上可以将 ncbind 的全局函数部分迁移到 simplebinder，但由于 simplebinder 功能子集更小，很多 ncbind 特性无法迁移。**一般不建议这个方向的迁移。**

不可迁移的 ncbind 特性：

```cpp
// ❌ 以下特性在 simplebinder 中无法实现

// 1. 类注册和构造函数
NCB_REGISTER_CLASS(MyClass) {
    NCB_CONSTRUCTOR((int));           // ❌ 无法迁移
}

// 2. 子类附加
NCB_REGISTER_SUBCLASS(MySubclass) {   // ❌ 无法迁移
    NCB_METHOD(newMethod);
}

// 3. 附加函数到已有类
NCB_ATTACH_FUNCTION(func, ExistingClass, impl);  // ❌ 无法迁移

// 4. Raw 回调方法
NCB_METHOD_RAW(rawMethod, RawCallback, 0);        // ❌ 无法迁移

// 5. Proxy/Bridge 调用模式
// ncbind 的 ivsProxy、ivsBridge 等                // ❌ 无法迁移

// 6. 自定义类型转换
// ncbTypeConvertor 特化                            // ❌ 无法迁移
```

---

## 七、混合使用方案

### 7.1 在同一插件中混合使用

在实际开发中，你可能需要在一个插件中同时使用两个框架。由于 simplebinder 本身就依赖 ncbind 的 `NCB_PRE_REGIST_CALLBACK` 来触发注册，两者天然可以共存：

```cpp
#include "ncbind.hpp"
#include "simplebinder.hpp"

// 用 ncbind 注册需要实例化的类
class ImageFilter {
public:
    ImageFilter(const tjs_char* name) : name_(name) {}
    void apply(iTJSDispatch2* layer) { /* 应用滤镜 */ }
    const tjs_char* getName() const { return name_.c_str(); }
private:
    ttstr name_;
};

NCB_REGISTER_CLASS(ImageFilter) {
    NCB_CONSTRUCTOR((const tjs_char*));
    NCB_METHOD(apply);
    NCB_PROPERTY_RO(name, getName);
}

// 用 simplebinder 注册工具函数和常量
static int blurRadius(int input, float sigma) {
    return static_cast<int>(std::ceil(sigma * 3.0f));
}

static void registFilterUtils(bool isRegist) {
    if (isRegist) {
        SimpleBinder::BindUtil bind;
        bind.Function("Filter_blurRadius", &blurRadius)
            .Constant("Filter_GAUSSIAN", 1)
            .Constant("Filter_BOX",      2)
            .Constant("Filter_MEDIAN",   3);
    }
}

NCB_PRE_REGIST_CALLBACK(registFilterUtils);
```

### 7.2 注册顺序和依赖

当混合使用两个框架时，需要注意注册顺序：

```
程序启动
│
├─ 全局静态初始化阶段（main() 之前）
│  │
│  ├─ ncbind 的 NCB_REGISTER_CLASS 相关静态实例构造
│  │  → 将注册信息添加到 ncbClassInfo 链表中
│  │
│  └─ ncbind 的 NCB_PRE_REGIST_CALLBACK 静态实例构造
│     → 将回调函数指针添加到回调链表中
│
├─ PluginInit() 被调用（TJS2 引擎初始化插件时）
│  │
│  ├─ 1. 执行所有 PRE_REGIST_CALLBACK（包括 simplebinder 的注册回调）
│  │     → simplebinder 的 BindUtil 在此时创建并注册函数/属性
│  │
│  ├─ 2. 执行所有 NCB_REGISTER_CLASS 的实际注册
│  │     → ncbind 的类在此时注册到 TJS2 全局空间
│  │
│  └─ 3. 执行所有 POST_REGIST_CALLBACK
│        → 注册后的清理/初始化工作
│
└─ 插件可用
```

**关键点：** `PRE_REGIST_CALLBACK` 在 `NCB_REGISTER_CLASS` **之前**执行。这意味着如果 simplebinder 注册的函数依赖 ncbind 注册的类（例如需要引用 ncbind 创建的 TJS2 类对象），在注册回调执行时那些类可能还不存在。

```cpp
// ⚠️ 可能出问题的写法
static void registDependent(bool isRegist) {
    if (isRegist) {
        SimpleBinder::BindUtil bind;
        // 试图引用 ncbind 注册的 ImageFilter 类
        // 但此时 NCB_REGISTER_CLASS(ImageFilter) 可能还没执行！
        // bind.Function("createDefaultFilter", &createDefaultImageFilter);
        // → 如果 createDefaultImageFilter 内部引用 ImageFilter 的 TJS2 类对象
        //   则可能崩溃或返回 null
    }
}

// ✅ 安全的写法——使用 POST 回调
static void registDependent_safe(bool isRegist) {
    // POST 回调在所有 NCB_REGISTER_CLASS 之后执行
    if (isRegist) {
        SimpleBinder::BindUtil bind;
        bind.Function("createDefaultFilter", &createDefaultImageFilter);
        // 此时 ImageFilter 类已经注册完成，可以安全引用
    }
}

NCB_POST_REGIST_CALLBACK(registDependent_safe);  // 注意用 POST 而非 PRE
```

---

## 八、综合对比总表

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│      维度         │       ncbind          │    simplebinder       │
├──────────────────┼──────────────────────┼──────────────────────┤
│ 设计哲学          │ 声明式，宏驱动        │ 命令式，流式调用      │
│ 源码行数          │ ~2400 行（hpp+cpp）   │ ~1240 行（仅 hpp）   │
│ 类注册            │ ✅ 完整支持           │ ❌ 不支持             │
│ 构造函数          │ ✅ 多种模式           │ ❌ 不支持             │
│ 方法绑定          │ ✅ 含别名/Raw/Proxy   │ ✅ 基础支持           │
│ 属性绑定          │ ✅ RO/WO/RW + 哨兵   │ ✅ 用 nullptr 控制    │
│ 常量              │ ✅ NCB_CONSTANT       │ ✅ Constant+Variant   │
│ 子类附加          │ ✅ REGISTER_SUBCLASS  │ ❌ 不支持             │
│ 函数附加          │ ✅ ATTACH_FUNCTION    │ ❌ 不支持             │
│ 类型安全          │ 强（编译期+运行时）   │ 弱（依赖隐式转换）   │
│ 自定义类型        │ ✅ 模板特化           │ ❌ 不支持             │
│ 最大参数数        │ 理论无限              │ 8 个                  │
│ 自动注册          │ ✅ 全自动             │ ❌ 需手动触发         │
│ 调试友好度        │ 低（宏+深层模板）     │ 高（显式调用链）      │
│ IDE 支持          │ 低（宏遮蔽类型）      │ 高（正常 C++ 代码）   │
│ 编译速度          │ 较慢（深层模板）      │ 较快（浅层模板）      │
│ 运行时性能        │ 稍慢（多层调用）      │ 稍快（少层调用）      │
│ 灵活性            │ 低（编译期确定）      │ 高（运行时可调）      │
│ 项目中使用        │ ✅ 所有插件           │ ❌ 从未使用           │
│ 学习曲线          │ 陡（需理解宏+模板）   │ 缓（链式调用直觉）   │
│ 独立性            │ ✅ 自包含             │ ❌ 依赖 ncbind 触发   │
└──────────────────┴──────────────────────┴──────────────────────┘
```

---

## 动手实践

### 实践一：用两种框架实现同一个工具库

我们来用 ncbind 和 simplebinder 分别实现一个字符串工具库，体验两种框架的开发手感。

**需求：** 注册三个全局函数——`StringUtil_reverse`（反转字符串）、`StringUtil_repeat`（重复字符串 N 次）、`StringUtil_contains`（检查是否包含子串）。

**ncbind 实现：**

```cpp
// string_util_ncbind.cpp
#include "ncbind.hpp"
#include <string>
#include <algorithm>

// 纯 C++ 实现（两种方式共用）
static ttstr reverseString(const tjs_char* input) {
    ttstr s(input);
    std::reverse(s.c_str(), s.c_str() + s.GetLen());  // 原地反转
    return s;
}

static ttstr repeatString(const tjs_char* input, int count) {
    ttstr result;
    for (int i = 0; i < count; ++i) {
        result += input;  // 逐次追加
    }
    return result;
}

static bool containsString(const tjs_char* haystack, const tjs_char* needle) {
    ttstr h(haystack);
    ttstr n(needle);
    return h.AsStdString().find(n.AsStdString()) != std::string::npos;
}

// ncbind 方式一：用 NCB_ATTACH_FUNCTION 附加到全局
// 需要一个"宿主类"来挂载函数
class StringUtilDummy {};  // 占位类

NCB_REGISTER_CLASS(StringUtilDummy) {}  // 注册空类

NCB_ATTACH_FUNCTION(StringUtil_reverse,  StringUtilDummy, reverseString);
NCB_ATTACH_FUNCTION(StringUtil_repeat,   StringUtilDummy, repeatString);
NCB_ATTACH_FUNCTION(StringUtil_contains, StringUtilDummy, containsString);

// ncbind 方式二：用 PRE_REGIST_CALLBACK + RawCallback
// 这种方式更灵活但代码量更大
static tjs_error TJS_INTF_METHOD
rawReverse(tTJSVariant* result, tjs_int numparams,
           tTJSVariant** params, iTJSDispatch2* objthis)
{
    if (numparams < 1) return TJS_E_BADPARAMCOUNT;
    ttstr input = *params[0];
    *result = reverseString(input.c_str());
    return TJS_S_OK;
}

static void registRawMethod(bool isRegist) {
    if (isRegist) {
        // 获取全局 TJS2 对象
        iTJSDispatch2* global = TVPGetScriptDispatch();
        if (!global) return;
        
        // 创建函数对象
        tTJSVariant func(&rawReverse, nullptr);
        
        // 注册到全局
        global->PropSet(
            TJS_MEMBERENSURE,           // 确保创建成员
            TJS_W("StringUtil_rawReverse"),
            nullptr,
            &func,
            global
        );
        
        global->Release();  // 释放全局对象引用
    }
}

NCB_PRE_REGIST_CALLBACK(registRawMethod);
```

**simplebinder 实现：**

```cpp
// string_util_simplebinder.cpp
#include "simplebinder.hpp"
#include <string>
#include <algorithm>

// 纯 C++ 实现（与上面完全相同）
static ttstr reverseString(const tjs_char* input) { /* ... */ }
static ttstr repeatString(const tjs_char* input, int count) { /* ... */ }
static bool containsString(const tjs_char* haystack, const tjs_char* needle) { /* ... */ }

// simplebinder 实现 —— 简洁明了
static void registStringUtils(bool isRegist) {
    if (isRegist) {
        SimpleBinder::BindUtil bind;
        bind.Function("StringUtil_reverse",  &reverseString)
            .Function("StringUtil_repeat",   &repeatString)
            .Function("StringUtil_contains", &containsString)
            .Constant("StringUtil_VERSION",  1);
    }
}

NCB_PRE_REGIST_CALLBACK(registStringUtils);
// 完成！simplebinder 版本只需 7 行注册代码
// 对比 ncbind 的 ATTACH_FUNCTION 需要创建占位类 + 3 行宏
// 对比 ncbind 的 RawCallback 需要 20+ 行手动参数处理
```

**对比体会：**

对于纯全局函数注册，simplebinder 代码量约为 ncbind 的 1/3。但如果需要添加类实例化功能（比如 `new StringUtil()`），simplebinder 就无能为力了。

### 实践二：选择框架的决策流程

面对一个新插件需求，用以下决策树选择框架：

```
开始
│
├─ 需要在 TJS2 中 new 对象？
│  ├─ 是 → 使用 ncbind（NCB_REGISTER_CLASS）
│  └─ 否 ↓
│
├─ 需要附加方法到已有 TJS2 类？
│  ├─ 是 → 使用 ncbind（NCB_REGISTER_SUBCLASS / NCB_ATTACH_FUNCTION）
│  └─ 否 ↓
│
├─ 需要 Raw 回调（手动处理变参）？
│  ├─ 是 → 使用 ncbind（NCB_METHOD_RAW）
│  └─ 否 ↓
│
├─ 需要超过 8 个参数的函数？
│  ├─ 是 → 使用 ncbind（模板递归无限制）
│  └─ 否 ↓
│
├─ 纯全局函数 + 常量？
│  ├─ 是 → 两者皆可，simplebinder 更简洁
│  └─ 否 ↓
│
└─ 需要运行时动态注册？
   ├─ 是 → simplebinder 更灵活
   └─ 否 → 默认使用 ncbind（项目标准）
```

---

## 对照项目源码

以下是两个框架在 KrKr2 项目中的源码位置和关键代码段：

| 文件 | 行号范围 | 内容说明 |
|------|---------|---------|
| `cpp/core/plugin/ncbind.hpp` | 1-30 | ncbind 头文件和类型定义 |
| `cpp/core/plugin/ncbind.hpp` | 44-115 | `ncbTypeConvertor` 类型转换系统 |
| `cpp/core/plugin/ncbind.hpp` | 120-380 | `ncbRegistClass` 注册基类 |
| `cpp/core/plugin/ncbind.hpp` | 1500-1730 | `Property()` 重载和哨兵值处理 |
| `cpp/core/plugin/ncbind.hpp` | 2247-2382 | 30+ 个注册宏定义 |
| `cpp/plugins/simplebinder/simplebinder.hpp` | 1-83 | `MethodInvoker` 和 `UNROLL_FOREACH` |
| `cpp/plugins/simplebinder/simplebinder.hpp` | 850-1012 | `PropertyStore` 实现 |
| `cpp/plugins/simplebinder/simplebinder.hpp` | 1048-1211 | `BindUtil` 公开 API |
| `cpp/plugins/scriptsEx.cpp` | 968-991 | ncbind 实际使用：RawCallback + NCB_METHOD |
| `cpp/plugins/csvParser.cpp` | 420-466 | ncbind 实际使用：完整类注册 |
| `cpp/plugins/windowEx.cpp` | 1513-1591 | ncbind 实际使用：大量 NCB_ATTACH_FUNCTION |
| `cpp/plugins/psdfile/PSD.cpp` | 末尾 | ncbind 实际使用：PSD 类注册 |

---

## 常见错误与排查

### 错误一：在 simplebinder 回调中使用尚未注册的 ncbind 类

**症状：** TJS2 脚本调用函数时崩溃或返回 `void`。

```cpp
// ❌ 错误写法
static void registEarly(bool isRegist) {
    if (isRegist) {
        SimpleBinder::BindUtil bind;
        // createParser 内部尝试创建 ncbind 注册的 CSVParser 实例
        // 但 PRE_REGIST_CALLBACK 在 NCB_REGISTER_CLASS 之前执行！
        bind.Function("createDefaultParser", &createParser);
    }
}
NCB_PRE_REGIST_CALLBACK(registEarly);  // ← PRE = 太早了

// ✅ 修复：改用 POST_REGIST_CALLBACK
NCB_POST_REGIST_CALLBACK(registEarly);  // ← POST = 在类注册之后
```

### 错误二：simplebinder 的 Property 参数顺序写反

**症状：** 读取属性时写入了数据，或写入属性时读取了旧值。

```cpp
// ❌ 容易犯的错误（直接使用 PropertyStore）
// PropertyStore::Link 的参数顺序是 (store, key, SETTER, GETTER)
// 注意：setter 在前，getter 在后！
propertyStore->Link(store, "name", &setName, &getName);  // 正确顺序
propertyStore->Link(store, "name", &getName, &setName);  // 写反了！

// ✅ 推荐：使用 BindUtil 封装（自动处理顺序）
bind.Property("name", &getName, &setName);  // BindUtil 参数顺序是 getter, setter
// 内部自动翻转为 PropertyStore 的 setter, getter 顺序
```

### 错误三：混淆 simplebinder 的 Class() 和 ncbind 的 NCB_REGISTER_CLASS

**症状：** TJS2 中执行 `new MyNamespace()` 报错。

```cpp
// ❌ 误解：以为 simplebinder 的 Class() 创建了可 new 的类
SimpleBinder::BindUtil bind;
bind.Class("MyPlugin")
    .Function("doWork", &doWork);
// TJS2: new MyPlugin()  → 错误！MyPlugin 不是一个类

// ✅ 正确理解：Class() 只创建命名空间对象
// TJS2: MyPlugin.doWork()  → 正确

// 如果需要 new，必须用 ncbind
NCB_REGISTER_CLASS(MyPlugin) {
    NCB_CONSTRUCTOR(());
    NCB_METHOD(doWork);
}
// TJS2: var p = new MyPlugin()  → 正确
//        p.doWork()              → 正确
```

---

## 本节小结

- ncbind 和 simplebinder 代表两种截然不同的绑定哲学：**声明式宏驱动** vs **命令式流式调用**。前者适合完整的类绑定场景，后者适合轻量的全局函数注册
- simplebinder 的**致命缺陷**是不支持类注册（`new`）、子类附加（`NCB_REGISTER_SUBCLASS`）和函数附加（`NCB_ATTACH_FUNCTION`），这使得它无法满足 KrKr2 中绝大多数插件的需求
- KrKr2 项目中**所有插件**均使用 ncbind，simplebinder 从未被实际使用。这不是偶然——而是因为 ncbind 的功能覆盖面远大于 simplebinder
- 类型安全方面 ncbind 远强于 simplebinder——ncbind 在编译期和运行时都有完整的类型检查，而 simplebinder 依赖 `static_cast` 的隐式转换，可能导致静默失败
- 两个框架可以**混合使用**，但需注意注册顺序：`PRE_REGIST_CALLBACK`（simplebinder 通常在这里注册）在 `NCB_REGISTER_CLASS` 之前执行，如果有依赖关系应改用 `POST_REGIST_CALLBACK`
- 选择框架的核心判据是：**是否需要类实例化**。需要 → ncbind，不需要且只注册全局函数 → simplebinder 可选（但 ncbind 同样可以做到）

---

## 练习题与答案

### 题目 1：判断以下插件需求应该使用哪个框架

给定以下插件需求，分析每个需求应该选择 ncbind 还是 simplebinder，并说明理由：

1. 注册一个 `JSONParser` 类，支持 `new JSONParser()`、`parse(string)`、`stringify(object)` 方法
2. 注册 3 个数学工具函数 `Math_clamp`、`Math_lerp`、`Math_smoothstep` 到全局
3. 为已有的 `Layer` 类添加一个 `drawGradient` 方法
4. 注册一组配置常量 `CONFIG_LOW=0`、`CONFIG_MEDIUM=1`、`CONFIG_HIGH=2`，并提供一个可在运行时修改的 `currentConfig` 全局变量

<details>
<summary>查看答案</summary>

**需求 1：使用 ncbind**

```cpp
// 必须用 ncbind —— 需要 new 构造实例
NCB_REGISTER_CLASS(JSONParser) {
    NCB_CONSTRUCTOR(());                    // 支持 new JSONParser()
    NCB_METHOD(parse);                      // 实例方法
    NCB_STATIC_METHOD(stringify);           // 静态方法也可以
}
// simplebinder 无法实现 new 构造，此需求只能用 ncbind
```

**需求 2：两者皆可，simplebinder 更简洁**

```cpp
// simplebinder 版（推荐——纯全局函数，无需类）
static double clamp(double x, double lo, double hi) {
    return std::max(lo, std::min(hi, x));
}
static double lerp(double a, double b, double t) {
    return a + (b - a) * t;
}
static double smoothstep(double edge0, double edge1, double x) {
    double t = clamp((x - edge0) / (edge1 - edge0), 0.0, 1.0);
    return t * t * (3.0 - 2.0 * t);
}

static void registMath(bool isRegist) {
    if (isRegist) {
        SimpleBinder::BindUtil bind;
        bind.Function("Math_clamp",      &clamp)
            .Function("Math_lerp",       &lerp)
            .Function("Math_smoothstep", &smoothstep);
    }
}
NCB_PRE_REGIST_CALLBACK(registMath);
```

**需求 3：使用 ncbind**

```cpp
// 必须用 ncbind —— 需要附加方法到已有类
// simplebinder 无法向已存在的 TJS2 类添加方法
NCB_REGISTER_SUBCLASS(LayerExGradient) {
    NCB_METHOD(drawGradient);
}
// 或
NCB_ATTACH_FUNCTION(drawGradient, Layer, drawGradientImpl);
```

**需求 4：simplebinder 最佳**

```cpp
// simplebinder 是唯一支持 Variant（可修改全局量）的框架
static void registConfig(bool isRegist) {
    if (isRegist) {
        SimpleBinder::BindUtil bind;
        bind.Constant("CONFIG_LOW",    0)
            .Constant("CONFIG_MEDIUM", 1)
            .Constant("CONFIG_HIGH",   2)
            .Variant("currentConfig",  1);  // 运行时可修改！
    }
}
NCB_PRE_REGIST_CALLBACK(registConfig);
// ncbind 的 NCB_CONSTANT 不支持可修改的全局变量
// 要用 ncbind 实现可修改全局量，需要 RawCallback 手写 getter/setter
```

</details>

### 题目 2：分析以下混合使用代码的 Bug

```cpp
#include "ncbind.hpp"
#include "simplebinder.hpp"

class Database {
public:
    Database(const tjs_char* path) : path_(path) {}
    bool query(const tjs_char* sql) { /* ... */ return true; }
    const tjs_char* getPath() const { return path_.c_str(); }
private:
    ttstr path_;
};

NCB_REGISTER_CLASS(Database) {
    NCB_CONSTRUCTOR((const tjs_char*));
    NCB_METHOD(query);
    NCB_PROPERTY_RO(path, getPath);
}

static Database* defaultDB = nullptr;

static void registHelpers(bool isRegist) {
    if (isRegist) {
        defaultDB = new Database(TJS_W("default.db"));
        SimpleBinder::BindUtil bind;
        bind.Function("DB_getDefaultPath", &Database::getPath);
    }
}

NCB_PRE_REGIST_CALLBACK(registHelpers);
```

找出这段代码中的 Bug 并修复。

<details>
<summary>查看答案</summary>

这段代码有**两个 Bug**：

**Bug 1：成员函数指针无法直接注册为全局函数**

`&Database::getPath` 是一个成员函数指针（类型为 `const tjs_char* (Database::*)() const`），它需要一个 `Database*` 实例才能调用。但 simplebinder 的 `Function()` 在注册非静态成员函数时，需要通过 `MethodInvoker` 从 TJS2 参数中获取实例指针。由于这里没有关联任何 TJS2 对象实例，调用时 `this` 指针将是垃圾值，导致崩溃。

**Bug 2：在 PRE_REGIST_CALLBACK 中创建 Database 实例可能过早**

`new Database(...)` 如果内部依赖 TJS2 引擎状态（比如使用 `TVPGetScriptDispatch` 或访问 TJS2 全局变量），在 PRE 阶段这些可能还未初始化。

**修复：**

```cpp
// 方法一：包装为自由函数（推荐）
static ttstr getDefaultDBPath() {
    if (defaultDB) return defaultDB->getPath();
    return TJS_W("");
}

static void registHelpers(bool isRegist) {
    if (isRegist) {
        defaultDB = new Database(TJS_W("default.db"));
        SimpleBinder::BindUtil bind;
        // 注册自由函数而非成员函数
        bind.Function("DB_getDefaultPath", &getDefaultDBPath);
    } else {
        delete defaultDB;  // 别忘了在注销时释放内存
        defaultDB = nullptr;
    }
}

NCB_PRE_REGIST_CALLBACK(registHelpers);

// 方法二：使用 ncbind 的 NCB_ATTACH_FUNCTION（更安全）
// 将函数直接附加到 Database 类上
static ttstr getDefaultPath() {
    if (defaultDB) return defaultDB->getPath();
    return TJS_W("");
}
NCB_ATTACH_FUNCTION(DB_getDefaultPath, Database, getDefaultPath);
```

</details>

### 题目 3：将以下 ncbind 代码改写为 simplebinder 版本

```cpp
NCB_REGISTER_CLASS(Timer) {
    NCB_CONSTRUCTOR(());
    NCB_METHOD(start);
    NCB_METHOD(stop);
    NCB_METHOD(reset);
    NCB_PROPERTY_RO(elapsed, getElapsed);
    NCB_PROPERTY(interval, getInterval, setInterval);
    NCB_CONSTANT(MAX_TIMERS);
}
```

请改写为 simplebinder 版本，并说明哪些功能无法迁移。

<details>
<summary>查看答案</summary>

```cpp
// simplebinder 版本
static void registTimer(bool isRegist) {
    if (isRegist) {
        SimpleBinder::BindUtil bind;
        bind.Class("Timer")  // 注意：这只是命名空间，不是类！
            .Function("start", &Timer::start)
            .Function("stop",  &Timer::stop)
            .Function("reset", &Timer::reset)
            .Property("elapsed",  &Timer::getElapsed, nullptr)   // 只读
            .Property("interval", &Timer::getInterval, &Timer::setInterval)
            .Constant("MAX_TIMERS", Timer::MAX_TIMERS);
    }
}

NCB_PRE_REGIST_CALLBACK(registTimer);
```

**无法迁移的功能：**

1. **`NCB_CONSTRUCTOR(())`** —— simplebinder 不支持构造函数注册，TJS2 中无法使用 `new Timer()` 创建实例。所有方法都变成了"静态"调用（`Timer.start()` 而非 `timer.start()`）
2. **实例方法语义** —— ncbind 的方法是实例方法，每个 TJS2 对象关联一个 C++ Timer 实例。simplebinder 的 `Function` 注册的成员方法没有关联实例，调用时 `this` 为空
3. **实例属性** —— ncbind 的 `elapsed` 和 `interval` 是每个实例独立的属性。simplebinder 的 Property 是全局的，所有访问共享同一个值

**结论：** 这个 Timer 类**不适合用 simplebinder 实现**，因为它的核心价值在于多实例——你可能同时需要多个 Timer（`var t1 = new Timer(); var t2 = new Timer();`），每个有独立状态。simplebinder 的"命名空间"式注册无法满足这个需求。

正确做法是保持 ncbind 实现，如果只需要一个全局 Timer 单例，可以用 simplebinder 包装：

```cpp
// 全局单例 Timer 的 simplebinder 包装
static Timer globalTimer;

static void timerStart()    { globalTimer.start(); }
static void timerStop()     { globalTimer.stop(); }
static void timerReset()    { globalTimer.reset(); }
static double timerElapsed() { return globalTimer.getElapsed(); }
static int timerGetInterval() { return globalTimer.getInterval(); }
static void timerSetInterval(int v) { globalTimer.setInterval(v); }

static void registTimer(bool isRegist) {
    if (isRegist) {
        SimpleBinder::BindUtil bind;
        bind.Class("Timer")
            .Function("start",   &timerStart)
            .Function("stop",    &timerStop)
            .Function("reset",   &timerReset)
            .Property("elapsed",  &timerElapsed, nullptr)
            .Property("interval", &timerGetInterval, &timerSetInterval)
            .Constant("MAX_TIMERS", Timer::MAX_TIMERS);
    }
}
```

这种包装可行但丧失了多实例能力，且代码量反而更多。所以在需要实例化的场景下，ncbind 是唯一正确的选择。

</details>

---

## 下一步

下一节 [高级用法与扩展](./03-高级用法与扩展.md) 将深入探讨 simplebinder 的内部实现细节和扩展可能性，包括自定义 Store 类型、`UNROLL_FOREACH` 宏展开机制、以及如何在 simplebinder 之上构建更高级的绑定抽象。










