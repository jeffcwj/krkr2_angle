# ncbind 注册框架详解

> **所属模块：** M12-插件逆向与实现
> **前置知识：** [01-V2Link与V2Unlink入口机制](./01-V2Link与V2Unlink入口机制.md)、[02-iTJSDispatch2接口与tp_stub](./02-iTJSDispatch2接口与tp_stub.md)
> **预计阅读时间：** 45 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：
1. 理解 ncbind 框架的设计思想——为何它能替代手写 V2Link/V2Unlink
2. 掌握 `NCB_MODULE_NAME`、`NCB_REGISTER_CLASS`、`NCB_ATTACH_CLASS`、`NCB_ATTACH_FUNCTION` 四大核心宏的用法与区别
3. 掌握 `NCB_PRE_REGIST_CALLBACK` 和 `NCB_POST_REGIST_CALLBACK` 的时机与使用场景
4. 理解 `ncbAutoRegister` 静态链表的注册链原理与 KrKr2 的 `_internal_plugins` 映射机制
5. 能够从零编写一个使用 ncbind 宏的完整插件

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| ncbind | Native Class Bind | KiriKiri 的 C++ 到 TJS2 绑定框架，用宏自动生成注册代码 |
| NCB_MODULE_NAME | Module Name Macro | 定义插件的 DLL 名称字符串，ncbind 所有宏都依赖此名 |
| NCB_REGISTER_CLASS | Register Class Macro | 将一个 C++ 类注册为 TJS2 中的新原生类 |
| NCB_ATTACH_CLASS | Attach Class Macro | 将 C++ 方法附加到 TJS2 中已存在的类上（不创建新类） |
| NCB_ATTACH_FUNCTION | Attach Function Macro | 将一个 C++ 自由函数附加到 TJS2 对象上 |
| ncbAutoRegister | Auto Register Base | ncbind 的自注册基类，通过静态构造函数自动收集插件注册信息 |
| 静态链表 | Static Linked List | 利用 C++ 全局对象构造顺序，在 `main()` 之前自动构建的链表结构 |

---

## 一、ncbind 的设计思想：告别手写 V2Link

### 1.1 手写注册的痛点

在前两节中，我们学习了原版 KiriKiri 的插件注册流程：`V2Link` 手动获取 `iTVPFunctionExporter`，通过 `TVPGetScriptDispatch()` 拿到全局对象，再用 `iTJSDispatch2::PropSet` 逐一注册方法和属性。这种方式存在三个严重问题：

1. **样板代码爆炸**：每注册一个方法，需要写 `TJS_BEGIN_NATIVE_METHOD_DECL` / `TJS_END_NATIVE_METHOD_DECL` 宏对，参数解包、类型转换全靠手写
2. **资源泄漏风险**：`AddRef` / `Release` 配对错误极易导致内存泄漏或悬垂指针
3. **注销遗漏**：`V2Unlink` 中必须手动反注册每个类和函数，少删一个就是资源泄漏

来看 `csvParser.cpp` 中手写注册的代码量（466 行源码中，真正的 CSV 解析逻辑只有约 200 行，其余全是 TJS 注册样板代码）：

```cpp
// csvParser.cpp — 手写注册示例（简化）
// 来源：cpp/plugins/csvParser.cpp 第 341-428 行

static iTJSDispatch2 *Create_NC_CSVParser() {
    // 1. 创建类对象
    tTJSNativeClassForPlugin *classobj =
        TJSCreateNativeClassForPlugin(
            TJS_W("CSVParser"),  // TJS 类名
            Create_NI_CSVParser  // 实例工厂函数
        );

    // 2. 开始注册成员
    TJS_BEGIN_NATIVE_MEMBERS(CSVParser)
    TJS_DECL_EMPTY_FINALIZE_METHOD

    // 3. 构造函数（每个方法都需要这种宏对）
    TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL(_this, NI_CSVParser, CSVParser)
    {
        return TJS_S_OK;
    }
    TJS_END_NATIVE_CONSTRUCTOR_DECL(CSVParser)

    // 4. 每个方法都要写一大段
    TJS_BEGIN_NATIVE_METHOD_DECL(init)
    {
        TJS_GET_NATIVE_INSTANCE(_this, NI_CSVParser);
        if (numparams < 1) return TJS_E_BADPARAMCOUNT;
        _this->init(param[0]->AsStringNoAddRef());
        return TJS_S_OK;
    }
    TJS_END_NATIVE_METHOD_DECL(init)

    // ... 每个方法重复以上模式 ...
    TJS_END_NATIVE_MEMBERS
    return classobj;
}
```

### 1.2 ncbind 的解决方案

ncbind（Native Class Bind 的缩写，一个用宏实现的 C++ 到 TJS2 的自动绑定框架）通过三个核心设计解决了上述问题：

| 设计要素 | 手写方式 | ncbind 方式 |
|----------|----------|-------------|
| 方法注册 | 每个方法 10+ 行宏对 | `NCB_METHOD(methodName)` 一行搞定 |
| 类注册 | 手写工厂函数 + TJS_BEGIN_NATIVE_MEMBERS | `NCB_REGISTER_CLASS(ClassName) { ... }` |
| 资源管理 | 手动 AddRef/Release | 框架自动管理生命周期 |
| 注销清理 | 手写 V2Unlink 逐一删除 | 框架自动生成反注册代码 |
| 插件入口 | 手写 V2Link 函数 | `NCB_MODULE_NAME` 宏声明即可 |

来看同样功能用 ncbind 的实现——最简插件 `getabout.cpp`（仅 5 行）：

```cpp
// getabout.cpp — ncbind 极简示例
// 来源：cpp/plugins/getabout.cpp（完整文件，共 5 行）

#include "ncbind.hpp"   // 引入 ncbind 框架头文件
#include "MsgIntf.h"    // TVPGetAboutString 的声明

// 定义插件模块名（对应原版 DLL 名）
#define NCB_MODULE_NAME TJS_W("getabout.dll")

// 将 C++ 函数 TVPGetAboutString 挂到 TJS 的 System 对象上，
// TJS 中调用 System.getAboutString() 即可
NCB_ATTACH_FUNCTION(getAboutString, System, TVPGetAboutString);
```

5 行代码完成了手写方式需要 50+ 行才能实现的功能。这就是 ncbind 的威力。

---

## 二、NCB_MODULE_NAME：插件的身份标识

### 2.1 宏的定义与作用

`NCB_MODULE_NAME` 是一个必须在每个插件源文件顶部定义的宏，它指定了该插件对应的原版 KiriKiri DLL 名称。ncbind 的所有注册宏内部都会引用这个名称来确定"这些注册项属于哪个插件模块"。

```cpp
// 标准用法：在 #include "ncbind.hpp" 之后立即定义
#define NCB_MODULE_NAME TJS_W("myPlugin.dll")
```

这里 `TJS_W()` 是 TJS2 的宽字符字符串宏（类似标准 C++ 的 `L""`，但在 TJS2 内部统一使用 `tjs_char` 类型），它将字符串字面量转换为 TJS2 内部使用的宽字符格式。

### 2.2 项目中的使用模式

KrKr2 项目中有 26 个插件文件定义了 `NCB_MODULE_NAME`，以下是部分示例：

```cpp
// cpp/plugins/getabout.cpp
#define NCB_MODULE_NAME TJS_W("getabout.dll")

// cpp/plugins/scriptsEx.cpp
#define NCB_MODULE_NAME TJS_W("scriptsEx.dll")

// cpp/plugins/csvParser.cpp
#define NCB_MODULE_NAME TJS_W("csvParser.dll")

// cpp/plugins/xp3filter.cpp
#define NCB_MODULE_NAME TJS_W("xp3filter.dll")

// cpp/plugins/windowEx.cpp
#define NCB_MODULE_NAME TJS_W("windowEx.dll")

// cpp/plugins/motionplayer/main.cpp
#define NCB_MODULE_NAME TJS_W("emoteplayer.dll")
```

注意 `motionplayer` 的模块名是 `emoteplayer.dll` 而非 `motionplayer.dll`——这是因为在 `PluginImpl.cpp` 中有映射逻辑：当游戏脚本请求加载 `emoteplayer.dll` 时，引擎会将其映射到 `motionplayer.dll` 内部实现（来源：`cpp/core/plugin/PluginImpl.cpp` 第 36 行的 `TVPLoadPlugin` 函数）。

### 2.3 NCB_MODULE_NAME 与插件加载流程的关系

当 TJS 脚本执行 `Plugins.link("csvParser.dll")` 时，引擎调用链如下：

```
TJS: Plugins.link("csvParser.dll")
  → TVPLoadPlugin("csvParser.dll")
    → TVPLoadInternalPlugin("csvParser.dll")
      → ncbAutoRegister::LoadModule("csvparser.dll")  // 注意：转为小写
        → 在 _internal_plugins 映射表中查找 "csvparser.dll"
        → 找到该模块的所有注册项
        → 依次调用每个注册项的 Regist() 方法
```

`NCB_MODULE_NAME` 的值就是插入到 `_internal_plugins` 映射表中的键。如果你定义了错误的模块名，插件虽然会编译通过，但在运行时 TJS 脚本无法加载它。

---

## 三、ncbAutoRegister：静态链表的自注册魔法

### 3.1 核心问题：如何在 main() 之前收集所有注册项？

ncbind 面临一个经典 C++ 问题：插件分散在数十个 `.cpp` 文件中，框架需要在运行时知道"有哪些插件、每个插件注册了哪些类和函数"，但又不能有一个中央注册文件逐一列举每个插件。

ncbind 的解决方案是利用 **C++ 全局对象的静态构造**（Static Initialization，指 C++ 程序在进入 `main()` 函数之前，自动调用所有全局/静态对象的构造函数这一语言特性）。每个 NCB 宏都会声明一个全局静态对象，该对象的构造函数会将自身插入到一个全局链表中。

### 3.2 ncbAutoRegister 类结构

来看 `ncbAutoRegister` 的核心代码（来源：`cpp/core/plugin/ncbind.hpp` 第 2097-2141 行）：

```cpp
// ncbAutoRegister — 所有 NCB 注册项的基类
// 来源：cpp/core/plugin/ncbind.hpp 第 2097-2141 行

struct ncbAutoRegister : public ncbTypedefs {
    typedef ncbAutoRegister ThisClassT;
    typedef DefsT::NameT NameT;

    // 三个注册阶段（Phase），按顺序执行
    enum LineT {
        PreRegist = 0,   // 预注册回调（NCB_PRE_REGIST_CALLBACK）
        ClassRegist,     // 类/函数注册（NCB_REGISTER_CLASS 等）
        PostRegist,      // 后注册回调（NCB_POST_REGIST_CALLBACK）
        LINE_COUNT       // 总数 = 3
    };

    NameT modulename;  // 模块名，即 NCB_MODULE_NAME 的值

    // 构造函数：将自身插入到对应阶段的链表头部
    ncbAutoRegister(NameT name, LineT line)
        : modulename(name), _next(_top[line])
    {
        _top[line] = this;
        // 经典的"头插法"链表构建
    }

    // 收集所有注册项到 _internal_plugins 映射表
    static void AllRegist(LineT line) {
        for (ThisClassT const* p = _top[line]; p; p = p->_next) {
            ttstr name = p->modulename;
            name.ToLowerCase();  // 模块名统一转小写
            _internal_plugins[name].lists[line].push_back(p);
        }
    }

    // 加载指定模块的所有注册项
    static bool LoadModule(const ttstr &_name);

protected:
    virtual void Regist()   const = 0;  // 纯虚：子类实现注册逻辑
    virtual void Unregist() const = 0;  // 纯虚：子类实现反注册逻辑

private:
    ThisClassT const* _next;                    // 链表的下一个节点
    static ThisClassT const* _top[LINE_COUNT];  // 三条链表的头指针

    // 模块名 → 注册项列表的映射
    struct INTERNAL_PLUGIN_LISTS {
        std::list<ThisClassT const*> lists[LINE_COUNT];
    };
    static std::map<ttstr, INTERNAL_PLUGIN_LISTS> _internal_plugins;
};
```

### 3.3 静态链表的构建过程（ASCII 图解）

假设有三个插件文件，每个文件使用了一个 NCB 宏：

```
编译时：每个 NCB 宏生成一个全局静态对象

getabout.cpp:  NCB_ATTACH_FUNCTION(...)  → 生成 static ncbFunctionAutoRegister_A
scriptsEx.cpp: NCB_ATTACH_CLASS(...)     → 生成 static ncbAttachTJS2ClassAutoRegister_B
xp3filter.cpp: NCB_POST_REGIST_CALLBACK(...) → 生成 static ncbCallbackAutoRegister_C

程序启动（main() 之前）：静态构造函数依次执行

┌─────────────────────────────────────────────────────┐
│              _top[PreRegist]  = nullptr              │
│              _top[ClassRegist] = nullptr             │
│              _top[PostRegist] = nullptr              │
└─────────────────────────────────────────────────────┘
               ↓ A 的构造函数执行（ClassRegist 阶段）
┌─────────────────────────────────────────────────────┐
│              _top[ClassRegist] → [A] → nullptr      │
└─────────────────────────────────────────────────────┘
               ↓ B 的构造函数执行（ClassRegist 阶段）
┌─────────────────────────────────────────────────────┐
│              _top[ClassRegist] → [B] → [A] → nullptr│
└─────────────────────────────────────────────────────┘
               ↓ C 的构造函数执行（PostRegist 阶段）
┌─────────────────────────────────────────────────────┐
│              _top[ClassRegist] → [B] → [A] → nullptr│
│              _top[PostRegist]  → [C] → nullptr      │
└─────────────────────────────────────────────────────┘

程序初始化时：AllRegist() 收集到 _internal_plugins

_internal_plugins = {
  "getabout.dll"   → { lists[ClassRegist] = [A] },
  "scriptsex.dll"  → { lists[ClassRegist] = [B] },
  "xp3filter.dll"  → { lists[PostRegist]  = [C] }
}
```

### 3.4 LoadModule 的实现

当 TJS 脚本加载插件时，`LoadModule` 从映射表中找到对应的注册项并依次执行（来源：`cpp/core/plugin/ncbind.cpp` 第 12-29 行）：

```cpp
// ncbAutoRegister::LoadModule — 加载一个内部插件
// 来源：cpp/core/plugin/ncbind.cpp 第 12-29 行

bool ncbAutoRegister::LoadModule(const ttstr &_name) {
    ttstr name = _name.AsLowerCase();  // 模块名转小写

    // 检查是否已加载（避免重复注册）
    if (TVPRegisteredPlugins.find(name) !=
        TVPRegisteredPlugins.end())
        return true;

    // 在映射表中查找
    auto it = _internal_plugins.find(name);
    if (it != _internal_plugins.end()) {
        // 按 PreRegist → ClassRegist → PostRegist 顺序执行
        for (const auto & plugin_list : it->second.lists) {
            for (auto i : plugin_list) {
                i->Regist();  // 调用每个注册项的虚函数
            }
        }
        TVPRegisteredPlugins.insert(name);  // 标记为已加载
        return true;
    }
    return false;  // 未找到该模块
}
```

注意 `TVPRegisteredPlugins` 是一个 `std::set<ttstr>` 全局变量（定义在 `ncbind.hpp` 第 18 行），用于追踪已加载的插件，防止同一插件被重复注册。

---

## 四、NCB_REGISTER_CLASS：注册新的 TJS 原生类

### 4.1 宏展开分析

`NCB_REGISTER_CLASS` 是最常用的类注册宏。它的作用是将一个 C++ 类作为全新的 TJS2 原生类注册到脚本引擎中（TJS 脚本可以用 `new ClassName()` 创建实例）。

宏定义（来源：`cpp/core/plugin/ncbind.hpp` 第 2166-2169 行）：

```cpp
// NCB_REGISTER_CLASS 宏的定义
#define NCB_REGISTER_CLASS(cls) \
    NCB_REGISTER_CLASS_DIFFER(cls, cls)

#define NCB_REGISTER_CLASS_DIFFER(name, cls) \
    NCB_TYPECONV_BOXING(cls); \
    NCB_REGISTER_CLASS_COMMON(cls, \
        ncbNativeClassAutoRegister, \
        ncbRegistNativeClass, \
        (NCB_MODULE_NAME, TJS_W(# name)))
```

展开后，这个宏实际上做了三件事：

1. **`NCB_TYPECONV_BOXING(cls)`**：为 `cls` 类型生成 TJS 变量（tTJSVariant）的自动装箱/拆箱代码，使 C++ 对象能与 TJS 变量互转
2. **声明一个 `ncbNativeClassAutoRegister<cls>` 全局静态对象**：自动插入到 `ClassRegist` 阶段的链表中
3. **定义 `ncbRegistClass<ncbRegistNativeClass<cls>>::Regist()` 方法的函数体**：花括号 `{ ... }` 中的内容就是注册体，包含 `NCB_METHOD`、`NCB_PROPERTY` 等方法注册宏

### 4.2 完整使用示例：steam_api 插件

来看项目中唯一使用 `NCB_REGISTER_CLASS` 的插件（来源：`cpp/plugins/steam/steam_api.cpp` 第 12 行）：

```cpp
// steam_api.cpp — NCB_REGISTER_CLASS 示例
// 来源：cpp/plugins/steam/steam_api.cpp 第 12 行

NCB_REGISTER_CLASS(steam_api) {
    Constructor();  // 注册无参构造函数
    // 可以在这里用 NCB_METHOD、NCB_PROPERTY 等注册方法
}
```

这一行代码完成了以下工作：
- 在 TJS 中创建名为 `steam_api` 的新类
- TJS 脚本可以用 `var obj = new steam_api();` 创建实例
- C++ 类 `steam_api` 的实例会被自动关联到 TJS 对象

### 4.3 注册体中可用的宏

在 `NCB_REGISTER_CLASS(ClassName) { ... }` 的花括号中，可以使用以下宏来注册类的成员（来源：`cpp/core/plugin/ncbind.hpp` 第 2248-2280 行）：

| 宏 | 作用 | 示例 |
|----|------|------|
| `Constructor()` | 注册无参构造函数 | `Constructor()` |
| `Constructor(args)` | 注册带参构造函数 | `NCB_CONSTRUCTOR((int, tjs_char const*))` |
| `NCB_METHOD(name)` | 注册同名成员方法 | `NCB_METHOD(doSomething)` |
| `NCB_METHOD_DIFFER(tjsName, cppMethod)` | TJS 名与 C++ 方法名不同时使用 | `NCB_METHOD_DIFFER(run, execute)` |
| `NCB_PROPERTY(name, get, set)` | 注册读写属性 | `NCB_PROPERTY(width, getWidth, setWidth)` |
| `NCB_PROPERTY_RO(name, get)` | 注册只读属性 | `NCB_PROPERTY_RO(count, getCount)` |
| `NCB_PROPERTY_WO(name, set)` | 注册只写属性 | `NCB_PROPERTY_WO(value, setValue)` |
| `RawCallback(name, cb, flag)` | 注册原始回调函数 | `RawCallback(TJS_W("exec"), &MyClass::rawExec, 0)` |
| `Variant(name, value)` | 注册常量值 | `Variant(TJS_W("VERSION"), 1)` |

### 4.4 自己动手写一个完整的 NCB_REGISTER_CLASS 插件

以下是一个完整的、可在 KrKr2 项目中编译的计数器插件示例：

```cpp
// counter_plugin.cpp — 完整的 NCB_REGISTER_CLASS 示例
// 将 C++ 的 Counter 类暴露给 TJS 脚本

#include "ncbind.hpp"

// 插件模块名
#define NCB_MODULE_NAME TJS_W("counter.dll")

// C++ 原生类
class Counter {
private:
    int value_;      // 当前计数值
    int step_;       // 步长
    ttstr name_;     // 计数器名称

public:
    // 构造函数：初始值和步长
    Counter(int initial, int step)
        : value_(initial), step_(step), name_(TJS_W("default")) {}

    // 方法：递增计数
    void increment() { value_ += step_; }

    // 方法：递减计数
    void decrement() { value_ -= step_; }

    // 方法：重置到指定值
    void reset(int newValue) { value_ = newValue; }

    // 属性 getter/setter
    int getValue() const { return value_; }
    void setStep(int s) { step_ = s; }
    int getStep() const { return step_; }
    void setName(ttstr n) { name_ = n; }
    ttstr getName() const { return name_; }
};

// 注册类到 TJS
NCB_REGISTER_CLASS(Counter) {
    // 注册带两个参数的构造函数：new Counter(0, 1)
    NCB_CONSTRUCTOR((int, int));

    // 注册方法（TJS 名与 C++ 方法名相同）
    NCB_METHOD(increment);
    NCB_METHOD(decrement);
    NCB_METHOD(reset);

    // 注册只读属性：counter.value（只能读，不能赋值）
    NCB_PROPERTY_RO(value, getValue);

    // 注册读写属性：counter.step 和 counter.name
    NCB_PROPERTY(step, getStep, setStep);
    NCB_PROPERTY(name, getName, setName);
}
```

对应的 TJS 脚本用法：

```javascript
// TJS 脚本中使用 Counter 类
Plugins.link("counter.dll");  // 加载插件

var c = new Counter(0, 1);    // 创建实例，初始值 0，步长 1
c.increment();                // value = 1
c.increment();                // value = 2
c.step = 5;                   // 修改步长为 5
c.increment();                // value = 7
c.name = "主计数器";          // 设置名称
System.inform(c.name + ": " + c.value);  // 显示 "主计数器: 7"
c.reset(0);                   // 重置为 0
```

---

## 五、NCB_ATTACH_CLASS——为已有 TJS 类追加方法

### 5.1 REGISTER 与 ATTACH 的本质区别

前文的 `NCB_REGISTER_CLASS` 创建一个**全新**的 TJS 类。但在 KrKr2 项目中，更常见的需求是：TJS 引擎已经内置了 `Scripts`、`System`、`Layer` 等核心类，插件想给这些**现成的类**追加新方法，而不是另起炉灶。`NCB_ATTACH_CLASS`（附加类注册宏）正是为此设计的。

| 特性 | NCB_REGISTER_CLASS | NCB_ATTACH_CLASS |
|------|-------------------|------------------|
| 用途 | 创建全新 TJS 类 | 给已有 TJS 类追加方法 |
| 底层注册器 | `ncbNativeClassAutoRegister` | `ncbAttachTJS2ClassAutoRegister` |
| 底层类 | `ncbRegistNativeClass` | `ncbAttachTJS2Class` |
| 实例管理 | 插件自己 new/delete | 延迟创建（Delay Create）——首次访问时才 new |
| 项目中使用次数 | 仅 1 处（steam_api.cpp） | 大量——scriptsEx、windowEx、layerex 等 |

### 5.2 宏展开解析

`NCB_ATTACH_CLASS` 定义在 `ncbind.hpp` 第 2239-2241 行：

```cpp
// 源码：cpp/core/plugin/ncbind.hpp 第 2239-2241 行
#define NCB_ATTACH_CLASS(cls, attach) \
    NCB_ATTACHED_INSTANCE_DELAY_CREATE(cls); \
    NCB_ATTACH_CLASS_WITH_HOOK(cls, attach)
```

第一部分 `NCB_ATTACHED_INSTANCE_DELAY_CREATE`（第 2230-2233 行）生成**延迟创建钩子**：

```cpp
// 源码：cpp/core/plugin/ncbind.hpp 第 2230-2233 行
// 展开后的伪代码：
NCB_GET_INSTANCE_HOOK(cls) {
    NCB_INSTANCE_GETTER(objthis) {
        ClassT* obj = GetNativeInstance(objthis);
        // 如果还没有 native 实例，就当场创建一个
        if (!obj) {
            obj = new ClassT();
            SetNativeInstance(objthis, obj);
        }
        return obj;
    }
}
```

这段代码的关键在于"延迟"（Delay）二字：因为 `Scripts` 类的 TJS 对象在插件加载时已经存在，但里面并没有插件的 C++ native 实例。当 TJS 脚本第一次调用插件追加的方法时，ncbind 会自动 `new ClassT()` 并绑定到该 TJS 对象上。后续再次调用时直接取出已创建的实例，不再重复创建。

第二部分 `NCB_ATTACH_CLASS_WITH_HOOK`（第 2235-2237 行）：

```cpp
// 源码：cpp/core/plugin/ncbind.hpp 第 2235-2237 行
#define NCB_ATTACH_CLASS_WITH_HOOK(cls, attach) \
    template <> struct ncbNativeClassMethodBase::nativeInstanceGetter<cls>; \
    NCB_REGISTER_CLASS_COMMON(cls, ncbAttachTJS2ClassAutoRegister, \
        ncbAttachTJS2Class, \
        (NCB_MODULE_NAME, TJS_W(# cls), TJS_W(# attach)))
```

注意最后一行的参数：`TJS_W(# cls)` 是 C++ 类名的宽字符串，`TJS_W(# attach)` 是要附加到的 **TJS 类名**。以 `NCB_ATTACH_CLASS(ScriptsAdd, Scripts)` 为例，`cls = ScriptsAdd`（C++ 类），`attach = Scripts`（TJS 类）。

### 5.3 实战：scriptsEx.cpp 的完整注册

`scriptsEx.cpp` 是项目中使用 `NCB_ATTACH_CLASS` 最典型的插件，它给 TJS 内置的 `Scripts` 全局对象追加了大量实用方法。完整注册代码如下：

```cpp
// 源码：cpp/plugins/scriptsEx.cpp 第 968-993 行
NCB_ATTACH_CLASS(ScriptsAdd, Scripts) {
    // === RawCallback：直接使用 TJS 调用约定的方法 ===

    // getObjectKeys —— 获取对象的所有属性名
    // TJS_STATICMEMBER 表示这是类级方法（静态成员），不需要实例
    RawCallback(TJS_W("getObjectKeys"),
        &ScriptsAdd::getKeys, TJS_STATICMEMBER);

    // getObjectCount —— 获取对象属性个数
    RawCallback(TJS_W("getObjectCount"),
        &ScriptsAdd::getCount, TJS_STATICMEMBER);

    // === NCB_METHOD：自动类型转换的方法 ===
    NCB_METHOD(getObjectContext);  // 获取对象上下文
    NCB_METHOD(isNullContext);     // 检查上下文是否为空
    NCB_METHOD(equalStruct);       // 结构体深度比较
    NCB_METHOD(equalStructNumericLoose);  // 数值宽松比较

    // foreach —— 遍历对象属性
    RawCallback(TJS_W("foreach"),
        &ScriptsAdd::foreach, TJS_STATICMEMBER);

    // getMD5HashString —— 计算 MD5 哈希
    RawCallback(TJS_W("getMD5HashString"),
        &ScriptsAdd::getMD5HashString, TJS_STATICMEMBER);

    NCB_METHOD(clone);  // 深拷贝对象

    // propSet / propGet —— 属性存取辅助
    RawCallback("propSet", &ScriptsAdd::propSet, TJS_STATICMEMBER);
    RawCallback("propGet", &ScriptsAdd::propGet, TJS_STATICMEMBER);

    // === Variant：注册常量值 ===
    Variant(TJS_W("pfMemberEnsure"),    TJS_MEMBERENSURE);
    Variant(TJS_W("pfMemberMustExist"), TJS_MEMBERMUSTEXIST);
    Variant(TJS_W("pfIgnoreProp"),      TJS_IGNOREPROP);
    Variant(TJS_W("pfHiddenMember"),    TJS_HIDDENMEMBER);
    Variant(TJS_W("pfStaticMember"),    TJS_STATICMEMBER);

    // safeEvalStorage —— 安全执行存储中的 TJS 脚本
    RawCallback(TJS_W("safeEvalStorage"),
        &ScriptsAdd::safeEvalStorage, TJS_STATICMEMBER);
};

// 追加独立函数到 Scripts 对象
NCB_ATTACH_FUNCTION(rehash, Scripts, TJSDoRehash);
```

这段代码展示了注册体（Registration Body）内三种注册方式的混合使用：

| 注册方式 | 功能 | 示例 |
|----------|------|------|
| `RawCallback(name, func, flag)` | 注册原始 TJS 回调函数（Raw TJS Callback）——参数使用 `tTJSVariant` 数组 | `getObjectKeys`、`foreach` |
| `NCB_METHOD(method)` | 注册自动类型转换方法——ncbind 自动将 `tTJSVariant` 转为 C++ 类型 | `clone`、`equalStruct` |
| `Variant(name, value)` | 注册常量值——将 C++ 整数/字符串作为 TJS 类的只读属性暴露 | `pfMemberEnsure` 等标志位 |

### 5.4 TJS_STATICMEMBER 标志详解

注册时传入的 `TJS_STATICMEMBER` 标志（Static Member Flag，静态成员标志）告诉 TJS 引擎：这个方法属于**类本身**，而非某个实例。在 TJS 脚本中调用时不需要 `new` 出一个对象：

```javascript
// TJS 脚本中直接通过类名调用静态方法
var keys = Scripts.getObjectKeys(someObj);  // 无需 new Scripts()
var md5 = Scripts.getMD5HashString("hello");

// 访问常量属性
var flag = Scripts.pfMemberEnsure;  // 值为 TJS_MEMBERENSURE (0x200)
```

如果去掉 `TJS_STATICMEMBER`，方法就会注册为实例方法，TJS 脚本必须先有一个实例才能调用——对于 `Scripts` 这种全局工具类来说显然不合理。

### 5.5 RawCallback 与 NCB_METHOD 的选择

两者都能注册方法，但签名完全不同：

```cpp
// RawCallback 的 C++ 函数签名——手动处理参数
static tjs_error TJS_INTF_METHOD getKeys(
    tTJSVariant *result,     // 返回值
    tjs_int numparams,       // 参数个数
    tTJSVariant **param,     // 参数数组
    iTJSDispatch2 *objthis)  // this 对象
{
    // 手动检查参数个数
    if (numparams < 1)
        return TJS_E_BADPARAMCOUNT;
    // 手动从 tTJSVariant 取值
    iTJSDispatch2 *obj = param[0]->AsObjectNoAddRef();
    // ... 具体逻辑 ...
    return TJS_S_OK;
}

// NCB_METHOD 的 C++ 函数签名——自动类型转换
ttstr clone(iTJSDispatch2 *obj) {
    // ncbind 自动把 tTJSVariant 转为 iTJSDispatch2*
    // 返回值自动转为 tTJSVariant
    // ... 具体逻辑 ...
    return result;
}
```

**选择指南**：当你需要可变参数、直接操作 `tTJSVariant`、或返回多个值时用 `RawCallback`；当函数签名简单且参数类型固定时用 `NCB_METHOD`，让 ncbind 代劳类型转换。

---

## 六、NCB_ATTACH_FUNCTION——注册独立函数

### 6.1 适用场景

有时你只想往某个 TJS 对象上**挂一个函数**，而不需要创建类、也不需要注册体大括号。`NCB_ATTACH_FUNCTION`（附加函数注册宏）就是为这种"一行搞定"的场景设计的。它把一个 C++ 自由函数（Free Function，即不属于任何类的普通函数）直接绑定为 TJS 对象的方法。

### 6.2 极简实战：getabout.cpp

整个项目中最短的插件文件就是 `getabout.cpp`，只有 5 行：

```cpp
// 源码：cpp/plugins/getabout.cpp（完整文件，共 5 行）
#include "ncbind.hpp"       // 包含 ncbind 注册框架头文件
#include "MsgIntf.h"        // 包含 TVPGetAboutString 函数声明

// 声明模块名
#define NCB_MODULE_NAME TJS_W("getabout.dll")

// 将 TVPGetAboutString 函数注册到 System 对象上，方法名为 getAboutString
NCB_ATTACH_FUNCTION(getAboutString, System, TVPGetAboutString);
```

执行效果：TJS 脚本可以通过 `System.getAboutString()` 调用 C++ 引擎中的 `TVPGetAboutString` 函数，获取"关于"信息字符串。

### 6.3 宏展开解析

`NCB_ATTACH_FUNCTION` 定义在 `ncbind.hpp` 第 2355 行：

```cpp
// 源码：cpp/core/plugin/ncbind.hpp 第 2354-2355 行
#define NCB_REGISTER_FUNCTION(name, function) \
    NCB_REGISTER_FUNCTION_COMMON(name, name, 0, function)
#define NCB_ATTACH_FUNCTION(name, attach, function) \
    NCB_REGISTER_FUNCTION_COMMON(name, attach ## _ ## name, \
        TJS_W(# attach), function)
```

`NCB_REGISTER_FUNCTION_COMMON`（第 2346-2352 行）的展开更有料：

```cpp
// 源码：cpp/core/plugin/ncbind.hpp 第 2346-2352 行
#define NCB_REGISTER_FUNCTION_COMMON(name, tag, attach, function) \
    // 1. 定义一个空结构体作为唯一标签
    struct ncbFunctionTag_ ## tag {}; \
    // 2. 对这个标签做模板特化
    template <> struct ncbNativeFunctionAutoRegisterTempl \
        <ncbFunctionTag_ ## tag> \
        : public ncbNativeFunctionAutoRegister { \
        // 构造函数：传入模块名
        ncbNativeFunctionAutoRegisterTempl() \
            : ncbNativeFunctionAutoRegister(NCB_MODULE_NAME){} \
        // 注册：找到目标对象 → 设置函数
        void Regist() const { \
            RegistFunction(TJS_W(# name), attach, &function); \
        } \
        // 反注册：从目标对象删除
        void Unregist() const { \
            UnregistFunction(TJS_W(# name), attach); \
        } \
    }; \
    // 3. 声明静态实例，触发全局构造
    static ncbNativeFunctionAutoRegisterTempl \
        <ncbFunctionTag_ ## tag> ncbFunctionAutoRegister_ ## tag
```

核心流程：`Regist()` 调用 `RegistFunction(name, attach, &function)`，而 `RegistFunction` 内部先通过 `GetDispatch(attach)` 找到目标 TJS 对象，再用 `PropSet` 把包装好的函数对象设置上去。

### 6.4 GetDispatch 的点号路径解析

`GetDispatch`（目标对象查找函数）是 `NCB_ATTACH_FUNCTION` 的核心，定义在 `ncbind.hpp` 第 2320-2341 行。它负责根据 `attach` 参数名找到对应的 TJS 对象：

```cpp
// 源码：cpp/core/plugin/ncbind.hpp 第 2320-2341 行
static iTJSDispatch2* GetDispatch(NameT attach) {
    iTJSDispatch2 *ret, *global = TVPGetScriptDispatch();
    if (!global) return nullptr;  // 全局对象不可用

    if (!attach) {
        ret = global;  // attach 为空 → 直接用全局对象
    } else {
        tTJSVariant val;
        NameT p = attach;
        // 检查是否包含点号
        while (*p) if (*p++ == TJS_W('.')) break;
        if (!*p) {
            // 无点号 → 直接从全局对象取属性
            // 例如 "System" → global.PropGet("System")
            global->PropGet(0, attach, nullptr, &val, global);
        } else {
            // 有点号 → 路径包含多级嵌套
            // 例如 "System.UI" → 用 eval 表达式求值
            TVPExecuteExpression(ttstr(attach), &val);
        }
        ret = val.AsObject();
        global->Release();
    }
    return ret;
}
```

这段代码处理了三种情况：

```
┌─────────────────────────────────────────────────────┐
│  GetDispatch(attach) 的三种路径                      │
├─────────────────────────────────────────────────────┤
│  attach = nullptr  → 返回全局对象                    │
│  attach = "System" → global.PropGet("System")        │
│  attach = "System.UI" → TVPExecuteExpression 求值    │
└─────────────────────────────────────────────────────┘
```

为什么有点号时要用 `TVPExecuteExpression`（TJS 表达式求值函数）？因为 `PropGet` 只能获取对象的直接成员，无法处理 `a.b.c` 这样的嵌套路径。`TVPExecuteExpression` 则相当于在 TJS 引擎中执行一行脚本来获取值，自动处理多级属性访问。

### 6.5 REGISTER_FUNCTION 与 ATTACH_FUNCTION 的区别

两个宏的区别很小，仅在于 `attach` 参数：

```cpp
// 注册到全局对象——TJS 中直接 myFunc() 即可调用
NCB_REGISTER_FUNCTION(myFunc, MyFunction);

// 注册到指定对象——TJS 中 System.getAboutString() 调用
NCB_ATTACH_FUNCTION(getAboutString, System, TVPGetAboutString);

// 注册到嵌套对象——TJS 中 Debug.Console.log() 调用
NCB_ATTACH_FUNCTION(log, Debug.Console, DebugLog);
```

---

## 七、NCB_PRE/POST_REGIST_CALLBACK——注册前后回调

### 7.1 三阶段注册模型回顾

回忆第三节中 `ncbAutoRegister` 的三个注册阶段（Phase）：

```
LoadModule("xxx.dll")
    │
    ├── Phase 0: PreRegist  ← NCB_PRE_REGIST_CALLBACK
    │   （在类注册之前执行，用于初始化环境、获取全局对象）
    │
    ├── Phase 1: ClassRegist ← NCB_REGISTER_CLASS / ATTACH_CLASS / ATTACH_FUNCTION
    │   （注册类和函数）
    │
    └── Phase 2: PostRegist  ← NCB_POST_REGIST_CALLBACK
        （在类注册之后执行，用于读取配置、注册过滤器）
```

`NCB_PRE_REGIST_CALLBACK` 和 `NCB_POST_REGIST_CALLBACK` 利用的就是 Phase 0 和 Phase 2。它们允许你在类注册的前后插入自定义的初始化/清理代码。

### 7.2 宏定义

两个宏定义在 `ncbind.hpp` 第 2375-2379 行：

```cpp
// 源码：cpp/core/plugin/ncbind.hpp 第 2375-2379 行
#define NCB_REGISTER_CALLBACK(pos, init, term, tag) \
    static ncbCallbackAutoRegister \
        ncbCallbackAutoRegister_ ## pos ## _ ## tag \
        (NCB_MODULE_NAME, ncbAutoRegister::pos, init, term)

// PRE：init = &cb, term = 0（只在加载时执行，卸载时不做事）
#define NCB_PRE_REGIST_CALLBACK(cb) \
    NCB_REGISTER_CALLBACK(PreRegist, &cb, 0, cb ## _0)

// POST：init = &cb, term = 0
#define NCB_POST_REGIST_CALLBACK(cb) \
    NCB_REGISTER_CALLBACK(PostRegist, &cb, 0, cb ## _0)

// 反向版本：只在卸载时执行
#define NCB_PRE_UNREGIST_CALLBACK(cb) \
    NCB_REGISTER_CALLBACK(PreRegist, 0, &cb, 0_ ## cb)
#define NCB_POST_UNREGIST_CALLBACK(cb) \
    NCB_REGISTER_CALLBACK(PostRegist, 0, &cb, 0_ ## cb)
```

底层的 `ncbCallbackAutoRegister`（第 2363-2372 行）非常简单：

```cpp
// 源码：cpp/core/plugin/ncbind.hpp 第 2363-2372 行
struct ncbCallbackAutoRegister : public ncbAutoRegister {
    typedef void (*CallbackT)();  // 回调函数类型：无参数无返回值

    ncbCallbackAutoRegister(NameT name, LineT line,
        CallbackT init, CallbackT term)
        : ncbAutoRegister(name, line),
          _init(init), _term(term) {}

protected:
    void Regist()   const override { if (_init) _init(); }  // 加载时调用 init
    void Unregist() const override { if (_term) _term(); }  // 卸载时调用 term
private:
    CallbackT _init, _term;  // 保存两个函数指针
};
```

### 7.3 实战：csvParser 的 PRE 回调

`csvParser.cpp` 使用 `NCB_PRE_REGIST_CALLBACK` 在类注册之前手动创建 `CSVParser` TJS 类。这是一种"混合模式"（Hybrid Pattern）——用回调执行传统的手写注册逻辑，同时利用 ncbind 的自动注册基础设施：

```cpp
// 源码：cpp/plugins/csvParser.cpp 第 430-448 行
void InitPlugin_CSVParser() {
    // 获取 TJS 全局对象
    iTJSDispatch2 *global = TVPGetScriptDispatch();
    if (global) {
        // 获取 Array 类的 clear 方法引用（后续解析 CSV 时需要）
        {
            tTJSVariant varScripts;
            TVPExecuteExpression(TJS_W("Array"), &varScripts);
            iTJSDispatch2 *dispatch = varScripts.AsObjectNoAddRef();
            ArrayClearMethod = getMember(dispatch, TJS_W("clear"));
        }
        // 用手写方式创建并注册 CSVParser 类
        // Create_NC_CSVParser() 内部使用 TJS_BEGIN_NATIVE_MEMBERS 宏
        addMember(global, TJS_W("CSVParser"), Create_NC_CSVParser());
        global->Release();
    }
}

// 在 Phase 0 (PreRegist) 执行 InitPlugin_CSVParser
// 源码：cpp/plugins/csvParser.cpp 第 466 行
NCB_PRE_REGIST_CALLBACK(InitPlugin_CSVParser);
```

为什么用 `PRE` 而不是 `POST`？因为 `csvParser` 使用的是旧式手写注册（TJS_BEGIN_NATIVE_MEMBERS 系列宏），不依赖 ncbind 的类注册阶段。放在 Phase 0 可以确保 CSVParser 类在其他插件的 ClassRegist 阶段之前就已经可用。

### 7.4 实战：xp3filter 的 POST 回调

`xp3filter.cpp` 使用 `NCB_POST_REGIST_CALLBACK` 在所有类注册完成后读取 XP3 过滤脚本（XP3 Filter Script，一种控制归档文件解密/解压的 TJS 脚本）并设置过滤器：

```cpp
// 源码：cpp/plugins/xp3filter.cpp 第 550-568 行
static void PostRegistCallback() {
    // 构造过滤脚本路径：应用目录 + "xp3filter.tjs"
    ttstr path = TVPGetAppPath() + TJS_W("xp3filter.tjs");

    // 检查脚本文件是否存在
    if (TVPIsExistentStorageNoSearch(path)) {
        // 创建文本读取流
        iTJSTextReadStream *stream =
            TVPCreateTextStreamForRead(path, "");
        try {
            // 读取整个脚本内容到 sXP3FilterScript
            stream->Read(sXP3FilterScript, 0);
        } catch(...) {
            stream->Destruct();
            throw;  // 异常继续向上传播
        }
        stream->Destruct();

        // 设置 XP3 归档解压过滤器
        TVPSetXP3ArchiveExtractionFilter(
            TVPXP3ArchiveExtractionFilterWrapper);
        // 设置 XP3 归档内容过滤器
        TVPSetXP3ArchiveContentFilter(
            TVPXP3ArchiveContentFilterWrapper);
    }
}

// 在 Phase 2 (PostRegist) 执行
// 源码：cpp/plugins/xp3filter.cpp 第 568 行
NCB_POST_REGIST_CALLBACK(PostRegistCallback);
```

为什么用 `POST`？因为过滤器的设置可能依赖于其他插件注册的类和函数。放在 Phase 2 确保所有类都已就绪，过滤器能安全地引用它们。

### 7.5 四种回调宏对比

| 宏 | 执行阶段 | 执行时机 | 典型用途 |
|---|---|---|---|
| `NCB_PRE_REGIST_CALLBACK` | Phase 0 | 模块加载时，类注册前 | 初始化环境、手写注册 |
| `NCB_POST_REGIST_CALLBACK` | Phase 2 | 模块加载时，类注册后 | 读取配置、设置过滤器 |
| `NCB_PRE_UNREGIST_CALLBACK` | Phase 0 | 模块卸载时 | 早期清理 |
| `NCB_POST_UNREGIST_CALLBACK` | Phase 2 | 模块卸载时 | 最终清理 |

---

## 动手实践

### 练习 1：用 NCB_ATTACH_FUNCTION 添加时间戳函数

给 TJS 的 `System` 对象添加一个 `getTimestamp` 方法，返回当前 Unix 时间戳：

```cpp
// 文件：timestamp_plugin.cpp
#include "ncbind.hpp"
#include <ctime>

#define NCB_MODULE_NAME TJS_W("timestamp.dll")

// 返回当前 Unix 时间戳（秒）
static tjs_int64 GetTimestamp() {
    return static_cast<tjs_int64>(std::time(nullptr));
}

// 注册到 System 对象
NCB_ATTACH_FUNCTION(getTimestamp, System, GetTimestamp);
```

验证方式：在 TJS 脚本中调用 `System.inform(System.getTimestamp());`，应弹出当前时间戳数字。

### 练习 2：用 NCB_ATTACH_CLASS 为 Array 添加 shuffle 方法

给 TJS 的 `Array` 类追加一个随机打乱方法：

```cpp
// 文件：array_shuffle.cpp
#include "ncbind.hpp"
#include <algorithm>
#include <random>

#define NCB_MODULE_NAME TJS_W("arrayshuffle.dll")

class ArrayShuffle {
public:
    // RawCallback 签名：直接操作 TJS 变量
    static tjs_error TJS_INTF_METHOD shuffle(
        tTJSVariant *result,
        tjs_int numparams,
        tTJSVariant **param,
        iTJSDispatch2 *objthis)
    {
        // 获取数组长度
        tTJSVariant countVar;
        objthis->PropGet(0, TJS_W("count"), nullptr,
                         &countVar, objthis);
        tjs_int count = (tjs_int)countVar;

        // Fisher-Yates 洗牌算法
        std::random_device rd;
        std::mt19937 gen(rd());
        for (tjs_int i = count - 1; i > 0; --i) {
            std::uniform_int_distribution<tjs_int> dist(0, i);
            tjs_int j = dist(gen);

            // 交换 array[i] 和 array[j]
            tTJSVariant vi, vj;
            objthis->PropGetByNum(0, i, &vi, objthis);
            objthis->PropGetByNum(0, j, &vj, objthis);
            objthis->PropSetByNum(0, i, &vj, objthis);
            objthis->PropSetByNum(0, j, &vi, objthis);
        }
        return TJS_S_OK;
    }
};

NCB_ATTACH_CLASS(ArrayShuffle, Array) {
    // 注册为实例方法（不加 TJS_STATICMEMBER）
    RawCallback(TJS_W("shuffle"), &ArrayShuffle::shuffle, 0);
};
```

验证方式：

```javascript
var a = [1, 2, 3, 4, 5];
a.shuffle();
System.inform(a);  // 应显示乱序数组
```

### 练习 3：用 NCB_POST_REGIST_CALLBACK 读取配置文件

在所有插件加载完成后读取 `config.tjs` 配置文件：

```cpp
// 文件：config_loader.cpp
#include "ncbind.hpp"

#define NCB_MODULE_NAME TJS_W("configloader.dll")

static void LoadConfigCallback() {
    ttstr path = TVPGetAppPath() + TJS_W("config.tjs");
    if (TVPIsExistentStorageNoSearch(path)) {
        // 执行配置脚本
        TVPExecuteStorage(path, nullptr, false, TJS_W(""));
        // 此时配置脚本中定义的全局变量已可用
    }
}

NCB_POST_REGIST_CALLBACK(LoadConfigCallback);
```

---

## 常见错误与排查

### 错误 1：忘记定义 NCB_MODULE_NAME

**症状**：编译报错 `NCB_MODULE_NAME was not declared in this scope`。

**原因**：所有 ncbind 宏都依赖 `NCB_MODULE_NAME` 来确定模块名，如果没有 `#define NCB_MODULE_NAME` 就使用任何 NCB_ 宏，预处理展开后会引用一个不存在的标识符。

**修复**：在文件顶部（`#include "ncbind.hpp"` 之后）添加：

```cpp
#define NCB_MODULE_NAME TJS_W("your_plugin.dll")
```

### 错误 2：NCB_ATTACH_CLASS 的 C++ 类没有默认构造函数

**症状**：编译报错 `no matching function for call to 'ClassName::ClassName()'`。

**原因**：`NCB_ATTACHED_INSTANCE_DELAY_CREATE` 展开后会尝试 `new ClassT()`（无参构造）。如果你的 C++ 类只有带参数的构造函数，延迟创建就会失败。

**修复**：为类添加无参默认构造函数，或改用 `NCB_ATTACH_CLASS_WITH_HOOK` 并提供自定义的 `NCB_GET_INSTANCE_HOOK` 实现：

```cpp
NCB_GET_INSTANCE_HOOK(MyClass) {
    NCB_INSTANCE_GETTER(objthis) {
        ClassT* obj = GetNativeInstance(objthis);
        if (!obj) {
            // 自定义构造逻辑
            obj = new ClassT("default_param");
            SetNativeInstance(objthis, obj);
        }
        return obj;
    }
};
NCB_ATTACH_CLASS_WITH_HOOK(MyClass, TargetTJSClass);
```

### 错误 3：NCB_ATTACH_FUNCTION 的目标对象在注册时不存在

**症状**：运行时插件加载后，TJS 脚本调用该函数报 `member "xxx" does not exist`。

**原因**：`GetDispatch` 在 Phase 1（ClassRegist）执行时去查找 TJS 对象，但该对象可能还未被创建。例如，如果你试图 `NCB_ATTACH_FUNCTION(myFunc, CustomClass, ...)` 但 `CustomClass` 是另一个插件在 Phase 1 才注册的——两者的注册顺序不确定。

**修复**：改用 `NCB_POST_REGIST_CALLBACK`，在 Phase 2 手动注册函数，确保所有类都已就绪：

```cpp
static void PostRegist() {
    iTJSDispatch2 *global = TVPGetScriptDispatch();
    if (global) {
        tTJSVariant val;
        global->PropGet(0, TJS_W("CustomClass"),
                        nullptr, &val, global);
        iTJSDispatch2 *cls = val.AsObject();
        // 在这里手动注册函数到 cls
        // ...
        cls->Release();
        global->Release();
    }
}
NCB_POST_REGIST_CALLBACK(PostRegist);
```

---

## 对照项目源码

以下是本节涉及的所有项目源码文件及关键行号，建议对照阅读：

| 文件路径 | 行号范围 | 内容说明 |
|----------|---------|---------|
| `cpp/core/plugin/ncbind.hpp` | 2100-2141 | `ncbAutoRegister` 基类——三阶段注册模型、静态链表、`_internal_plugins` 映射 |
| `cpp/core/plugin/ncbind.hpp` | 2143-2200 | `ncbNativeClassAutoRegister`——NCB_REGISTER_CLASS 的底层注册器 |
| `cpp/core/plugin/ncbind.hpp` | 2230-2241 | `NCB_ATTACHED_INSTANCE_DELAY_CREATE` 和 `NCB_ATTACH_CLASS` 宏定义 |
| `cpp/core/plugin/ncbind.hpp` | 2248-2280 | 注册体内部宏：`NCB_METHOD`、`NCB_PROPERTY`、`RawCallback`、`Variant` 等 |
| `cpp/core/plugin/ncbind.hpp` | 2286-2356 | `ncbNativeFunctionAutoRegister`——NCB_ATTACH_FUNCTION 底层实现和 `GetDispatch` |
| `cpp/core/plugin/ncbind.hpp` | 2363-2382 | `ncbCallbackAutoRegister`——回调注册类和四个回调宏 |
| `cpp/core/plugin/ncbind.cpp` | 1-29 | `LoadModule` 实现——三阶段顺序调用 |
| `cpp/plugins/getabout.cpp` | 1-5 | 最简 NCB_ATTACH_FUNCTION 示例（5 行完成） |
| `cpp/plugins/scriptsEx.cpp` | 968-993 | NCB_ATTACH_CLASS 完整示例（RawCallback + NCB_METHOD + Variant 混合） |
| `cpp/plugins/csvParser.cpp` | 430-466 | NCB_PRE_REGIST_CALLBACK 示例（手写注册 + ncbind 回调混合） |
| `cpp/plugins/xp3filter.cpp` | 550-568 | NCB_POST_REGIST_CALLBACK 示例（读取过滤脚本 + 设置归档过滤器） |
| `cpp/plugins/steam/steam_api.cpp` | 1-12 | NCB_REGISTER_CLASS 示例（项目中唯一的新类注册） |

---

## 本节小结

- **ncbind 是 KrKr2 的插件注册框架**，通过 C++ 宏和模板在编译期生成注册代码，利用全局静态变量的构造函数在 `main()` 之前完成注册
- **三阶段注册模型**：PreRegist（Phase 0）→ ClassRegist（Phase 1）→ PostRegist（Phase 2），对应不同宏
- **NCB_REGISTER_CLASS** 创建全新 TJS 类，项目中仅 `steam_api.cpp` 使用
- **NCB_ATTACH_CLASS** 给已有 TJS 类追加方法，是项目中最常用的注册方式。内含延迟创建机制
- **NCB_ATTACH_FUNCTION** 注册独立函数到 TJS 对象，支持点号路径解析
- **NCB_PRE/POST_REGIST_CALLBACK** 在类注册前后插入回调，用于初始化和配置
- **注册体内可混合使用** `RawCallback`（原始回调）、`NCB_METHOD`（自动转换）、`NCB_PROPERTY`（属性）和 `Variant`（常量）
- **NCB_MODULE_NAME** 是所有 ncbind 宏的前置依赖，必须在使用前定义

---

## 练习题与答案

### 题目 1：ncbind 注册宏选型

你需要实现以下三个功能，分别应该用哪个 ncbind 宏？请说明理由。

1. 创建一个全新的 `HTTPClient` TJS 类，支持 `new HTTPClient()` 语法
2. 给已有的 `Layer` 类添加一个 `blur()` 模糊方法
3. 在所有插件注册完成后，从文件读取渲染参数配置

<details>
<summary>查看答案</summary>

1. **`NCB_REGISTER_CLASS(HTTPClient)`**——需要创建全新的 TJS 类，只有 `NCB_REGISTER_CLASS` 支持。在注册体中用 `NCB_CONSTRUCTOR` 定义构造函数参数，用 `NCB_METHOD` 注册方法。

2. **`NCB_ATTACH_CLASS(LayerBlur, Layer)`**——`Layer` 是 TJS 引擎已有的类，不能重新创建，只能追加。`NCB_ATTACH_CLASS` 的第二个参数 `Layer` 指定了附加目标。

3. **`NCB_POST_REGIST_CALLBACK(LoadRenderConfig)`**——需要在所有类注册（Phase 1）完成后执行，这正是 Phase 2 (PostRegist) 的用途。定义一个 `void LoadRenderConfig()` 函数即可。

</details>

### 题目 2：分析宏展开结果

以下代码：

```cpp
#define NCB_MODULE_NAME TJS_W("test.dll")
NCB_ATTACH_FUNCTION(hello, System, SayHello);
```

请回答：
1. `GetDispatch` 会收到什么参数？
2. `RegistFunction` 注册的 TJS 方法名是什么？
3. 全局静态变量的类型名是什么？

<details>
<summary>查看答案</summary>

1. `GetDispatch` 收到 `TJS_W("System")`（宽字符串 `L"System"`）。宏展开 `TJS_W(# attach)` 即 `TJS_W("System")`。

2. 注册的 TJS 方法名是 `"hello"`。宏展开 `TJS_W(# name)` 即 `TJS_W("hello")`。

3. 全局静态变量类型名为 `ncbNativeFunctionAutoRegisterTempl<ncbFunctionTag_System_hello>`。宏展开 `attach ## _ ## name` 即 `System_hello`，然后用它作为 `ncbFunctionTag_` 后缀生成唯一标签。

完整展开（简化版）：

```cpp
struct ncbFunctionTag_System_hello {};
template <> struct ncbNativeFunctionAutoRegisterTempl
    <ncbFunctionTag_System_hello>
    : public ncbNativeFunctionAutoRegister {
    ncbNativeFunctionAutoRegisterTempl()
        : ncbNativeFunctionAutoRegister(TJS_W("test.dll")) {}
    void Regist() const {
        RegistFunction(TJS_W("hello"), TJS_W("System"), &SayHello);
    }
    void Unregist() const {
        UnregistFunction(TJS_W("hello"), TJS_W("System"));
    }
};
static ncbNativeFunctionAutoRegisterTempl<ncbFunctionTag_System_hello>
    ncbFunctionAutoRegister_System_hello;
```

</details>

### 题目 3：修复注册错误

以下插件代码编译通过但运行时 TJS 调用 `MyObj.calculate(10)` 时崩溃。请找出 bug 并修复：

```cpp
#include "ncbind.hpp"
#define NCB_MODULE_NAME TJS_W("calc.dll")

class Calculator {
    int base_;
public:
    Calculator(int base) : base_(base) {}
    int calculate(int x) { return base_ + x; }
};

NCB_ATTACH_CLASS(Calculator, MyObj) {
    NCB_METHOD(calculate);
};
```

<details>
<summary>查看答案</summary>

**Bug**：`Calculator` 没有默认构造函数。`NCB_ATTACH_CLASS` 内部展开 `NCB_ATTACHED_INSTANCE_DELAY_CREATE` 会尝试 `new Calculator()`（无参构造），但 `Calculator` 只有 `Calculator(int)` 构造函数。

虽然编译可能通过（取决于编译器的模板实例化时机），但运行时延迟创建失败，`GetNativeInstance` 返回的指针无效或为空，导致崩溃。

**修复方案 A**——添加默认构造函数：

```cpp
class Calculator {
    int base_;
public:
    Calculator() : base_(0) {}  // 添加默认构造
    Calculator(int base) : base_(base) {}
    int calculate(int x) { return base_ + x; }
};
```

**修复方案 B**——使用自定义实例钩子：

```cpp
NCB_GET_INSTANCE_HOOK(Calculator) {
    NCB_INSTANCE_GETTER(objthis) {
        ClassT* obj = GetNativeInstance(objthis);
        if (!obj) {
            obj = new ClassT(0);  // 默认 base = 0
            SetNativeInstance(objthis, obj);
        }
        return obj;
    }
};
NCB_ATTACH_CLASS_WITH_HOOK(Calculator, MyObj) {
    NCB_METHOD(calculate);
};
```

</details>

---

## 下一步

下一章 [02-无源码插件清单](../02-无源码插件清单/01-KrKr2插件全景图.md) 将对项目中所有插件进行全景梳理，识别哪些插件有源码可以直接编译，哪些需要通过逆向工程来还原实现。

