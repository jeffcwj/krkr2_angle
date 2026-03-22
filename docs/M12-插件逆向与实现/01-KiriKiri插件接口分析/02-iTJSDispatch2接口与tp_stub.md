# iTJSDispatch2 接口与 tp_stub

> **所属模块：** M12-插件逆向与实现
> **前置知识：** [01-V2Link 与 V2Unlink 入口机制](./01-V2Link与V2Unlink入口机制.md)、[M07-TJS2 脚本引擎](../../M07-TJS2脚本引擎/)
> **预计阅读时间：** 约 30 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 `iTJSDispatch2` 接口的设计理念——为什么 KiriKiri 选择用一个统一接口来表示所有 TJS 对象
2. 掌握 `iTJSDispatch2` 的核心方法（PropGet、PropSet、FuncCall、CreateNew 等），并能在逆向分析中识别这些调用
3. 理解原版 `tp_stub.h` 与 KrKr2 简化版的区别，以及这种差异对插件开发/逆向的影响
4. 在实际插件代码中正确使用 `iTJSDispatch2` 进行属性读写、方法调用和对象创建

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| iTJSDispatch2 | iTJSDispatch2 | TJS2 引擎中所有对象的统一接口，通过它可以对任何 TJS 对象执行属性读写、方法调用等操作 |
| Dispatch 接口 | Dispatch Interface | 一种允许在运行时通过名称调用方法/访问属性的接口设计，源自 COM 的 IDispatch |
| tp_stub | tp_stub | 原版 KiriKiri 提供给插件开发者的"桩文件"，包含引擎内部 API 的声明和实现 |
| tTJSVariant | tTJSVariant | TJS 的万能变量类型，可以存储整数、浮点数、字符串、对象引用等任何 TJS 值 |
| tTJSVariantClosure | tTJSVariantClosure | 将一个 iTJSDispatch2 对象指针和 this 上下文打包在一起的结构，用于闭包调用 |
| 引用计数 | Reference Counting | 一种内存管理技术，每个对象维护被引用的次数，计数归零时自动释放 |
| 成员提示 | Member Hint | KiriKiri 用于加速属性查找的缓存机制，第一次按名称查找后记录 hint 值，后续查找可直接定位 |

## 一、iTJSDispatch2 的设计理念

### 1.1 为什么需要统一接口

在 TJS2 脚本引擎中，"一切皆对象"：整数、字符串、数组、字典、函数、类实例——它们在引擎内部都通过同一个接口 `iTJSDispatch2` 来访问。这种设计的灵感来自 Microsoft COM（Component Object Model，组件对象模型）中的 `IDispatch` 接口。

为什么选择这种设计？考虑一个场景：TJS 脚本中写了 `var x = obj.name;`。引擎在执行这行代码时需要：
1. 找到 `obj` 对应的 C++ 对象
2. 在该对象上查找名为 `name` 的属性
3. 读取属性值并赋给变量 `x`

问题是：`obj` 可能是一个字典（Dictionary）、一个类实例、一个图层对象、一个插件注册的自定义类——这些类型在 C++ 层面完全不同。如果没有统一接口，引擎需要为每种类型写一套属性访问代码，这在插件可以注册任意新类型的情况下是不可能的。

`iTJSDispatch2` 就是解决这个问题的统一接口：不管底层对象是什么 C++ 类型，引擎和插件都通过 `iTJSDispatch2` 指针来操作它。

### 1.2 接口名称解读

`iTJSDispatch2` 这个名字包含了几层含义：

| 部分 | 含义 |
|------|------|
| `i` | Interface（接口），表示这是一个纯虚类 |
| `TJS` | 指 TJS2 脚本引擎 |
| `Dispatch` | 分发/调度，来自 COM 的 IDispatch 概念——通过名称在运行时动态分发方法调用 |
| `2` | 第二版接口，与 V2Link 中的 "2" 对应——TJS2 引擎的第二版对象接口 |

### 1.3 iTJSDispatch2 与 COM IDispatch 的对比

如果你熟悉 Windows COM 编程，会发现 `iTJSDispatch2` 与 `IDispatch` 有很多相似之处：

```
COM IDispatch                       TJS iTJSDispatch2
─────────────────────────────────  ─────────────────────────────────
GetTypeInfo()                       (无对应——TJS 无类型信息系统)
GetIDsOfNames("method")             PropGet(membername, hint)
Invoke(dispID, args)                FuncCall(membername, args)
引用计数: AddRef / Release          引用计数: AddRef / Release
类型查询: QueryInterface            类型查询: IsInstanceOf
```

关键区别在于：COM 使用数字 ID（dispID）来标识成员，需要先调用 `GetIDsOfNames` 获取 ID；TJS 直接使用字符串名称，配合 hint 缓存机制来加速后续查找。TJS 的方式更直观，但在频繁调用同一成员时性能略低——这就是 hint 参数存在的原因。

## 二、iTJSDispatch2 核心方法详解

### 2.1 方法概览

`iTJSDispatch2` 定义了约 20 个虚方法，覆盖了对象操作的所有场景。按功能分组如下：

```
┌─────────────────────────────────────────────────────┐
│              iTJSDispatch2 方法分组                    │
├─────────────────────────────────────────────────────┤
│  生命周期管理                                         │
│    AddRef()          — 增加引用计数                   │
│    Release()         — 减少引用计数（归零时销毁）       │
│                                                      │
│  属性读写（按名称）                                    │
│    PropGet()         — 读取属性值                     │
│    PropSet()         — 设置属性值                     │
│    PropGetByNum()    — 按数字索引读取（数组用）         │
│    PropSetByNum()    — 按数字索引设置（数组用）         │
│                                                      │
│  方法调用                                             │
│    FuncCall()        — 调用方法                       │
│    FuncCallByNum()   — 按索引调用（罕见）              │
│                                                      │
│  对象创建                                             │
│    CreateNew()       — 创建新实例（等价于 new）         │
│                                                      │
│  类型查询                                             │
│    IsInstanceOf()    — 判断是否为某类的实例             │
│    GetCount()        — 获取元素数量（数组/字典用）      │
│                                                      │
│  成员枚举                                             │
│    EnumMembers()     — 遍历所有成员（用回调函数）       │
│                                                      │
│  成员管理                                             │
│    DeleteMember()    — 删除成员                       │
│    IsValid()         — 检查对象是否有效                │
│    Invalidate()      — 使对象失效                     │
└─────────────────────────────────────────────────────┘
```

### 2.2 PropGet — 属性读取

`PropGet` 是最常用的方法之一，用于从对象上读取一个属性的值。函数签名如下：

```cpp
// iTJSDispatch2::PropGet — 读取对象属性
// 返回值：tjs_error，TJS_S_OK 表示成功
virtual tjs_error PropGet(
    tjs_uint32 flag,           // 操作标志（如 TJS_MEMBERMUSTEXIST）
    const tjs_char *membername,// 属性名（宽字符串）
    tjs_uint32 *hint,          // 成员提示（查找缓存，可为 nullptr）
    tTJSVariant *result,       // 输出：属性值
    iTJSDispatch2 *objthis     // this 上下文对象
) = 0;
```

参数详解：

| 参数 | 含义 | 典型值 |
|------|------|--------|
| `flag` | 控制行为的标志位 | `0`（默认）、`TJS_MEMBERMUSTEXIST`（属性必须存在，否则报错）、`TJS_IGNOREPROP`（跳过属性 getter，直接读原始值） |
| `membername` | 要读取的属性名，宽字符串 | `TJS_W("count")`、`TJS_W("visible")` |
| `hint` | 查找提示，用于加速重复查找。第一次查找时传入 `&someHint`（初始值 0），引擎填入缓存值，后续查找传入同一变量可跳过名称比较 | `&countHint`（static tjs_uint32 类型） |
| `result` | 输出参数，接收属性值。类型为 `tTJSVariant`，可以包含任何 TJS 类型 | `&myVariant` |
| `objthis` | 指定 this 上下文。对于普通属性访问通常传 `nullptr`；对于闭包调用需要传具体对象 | `nullptr` |

使用示例：

```cpp
// 示例：读取 TJS 数组的 count 属性
// 等价 TJS 脚本：var c = myArray.count;
void getArrayCount(iTJSDispatch2* arrayObj) {
    tTJSVariant result;        // 用于接收属性值
    static tjs_uint32 hint = 0; // hint 缓存（static 保证跨调用保持）

    // 调用 PropGet 读取 "count" 属性
    tjs_error err = arrayObj->PropGet(
        0,                     // flag: 默认行为
        TJS_W("count"),        // 属性名
        &hint,                 // hint 指针（首次填充，后续加速）
        &result,               // 输出：属性值
        arrayObj               // this 上下文
    );

    if (err == TJS_S_OK) {
        tjs_int count = result;  // tTJSVariant 可隐式转换为整数
        // count 现在就是数组的元素个数
    }
}
```

### 2.3 PropSet — 属性写入

`PropSet` 用于设置对象的属性值：

```cpp
// iTJSDispatch2::PropSet — 设置对象属性
virtual tjs_error PropSet(
    tjs_uint32 flag,              // 标志（TJS_MEMBERENSURE = 属性不存在则创建）
    const tjs_char *membername,   // 属性名
    tjs_uint32 *hint,             // 查找提示
    const tTJSVariant *param,     // 输入：要设置的值
    iTJSDispatch2 *objthis        // this 上下文
) = 0;
```

重要标志位：

| 标志 | 含义 |
|------|------|
| `TJS_MEMBERENSURE` | 如果属性不存在则自动创建（相当于 TJS 中 `obj.newProp = value`） |
| `TJS_MEMBERMUSTEXIST` | 属性必须已存在，否则返回错误 |
| `TJS_HIDDENMEMBER` | 设置为隐藏成员（不在 `EnumMembers` 中出现） |

```cpp
// 示例：向字典对象添加一个键值对
// 等价 TJS 脚本：myDict["name"] = "KrKr2";
void setDictValue(iTJSDispatch2* dictObj) {
    tTJSVariant value(TJS_W("KrKr2"));  // 创建字符串类型的 Variant

    tjs_error err = dictObj->PropSet(
        TJS_MEMBERENSURE,    // 不存在则创建
        TJS_W("name"),       // 属性名
        nullptr,             // 不使用 hint
        &value,              // 要设置的值
        dictObj              // this 上下文
    );
    // err == TJS_S_OK 表示设置成功
}
```

### 2.4 FuncCall — 方法调用

`FuncCall` 用于调用对象上的方法：

```cpp
// iTJSDispatch2::FuncCall — 调用对象方法
virtual tjs_error FuncCall(
    tjs_uint32 flag,              // 标志
    const tjs_char *membername,   // 方法名（nullptr = 调用默认方法/函数对象本身）
    tjs_uint32 *hint,             // 查找提示
    tTJSVariant *result,          // 输出：返回值（nullptr = 不关心返回值）
    tjs_int numparams,            // 参数个数
    tTJSVariant **param,          // 参数数组（每个元素是指向 tTJSVariant 的指针）
    iTJSDispatch2 *objthis        // this 上下文
) = 0;
```

```cpp
// 示例：调用 TJS 数组的 add 方法
// 等价 TJS 脚本：myArray.add(42);
void addToArray(iTJSDispatch2* arrayObj) {
    tTJSVariant value(42);       // 要添加的值
    tTJSVariant* params[] = { &value };  // 参数数组
    static tjs_uint32 addHint = 0;

    tjs_error err = arrayObj->FuncCall(
        0,                       // flag: 默认
        TJS_W("add"),            // 方法名
        &addHint,                // hint
        nullptr,                 // 不关心返回值
        1,                       // 1 个参数
        params,                  // 参数数组
        arrayObj                 // this
    );
}
```

### 2.5 引用计数：AddRef 与 Release

`iTJSDispatch2` 对象使用引用计数（Reference Counting）来管理生命周期。每个对象维护一个计数器，表示有多少处代码持有该对象的引用：

```cpp
// 引用计数的基本原则
iTJSDispatch2* obj = TJSCreateDictionaryObject();
// 此时 obj 引用计数 = 1（创建时初始化为 1）

obj->AddRef();
// 引用计数 = 2（手动增加引用——当你需要在多处持有这个对象时）

obj->Release();
// 引用计数 = 1（释放一个引用）

obj->Release();
// 引用计数 = 0 → 对象被自动销毁（析构函数被调用，内存被释放）
// 此后再访问 obj 就是未定义行为（悬垂指针）！
```

**安全使用模式**——利用 `tTJSVariant` 自动管理引用计数：

```cpp
// 推荐：通过 tTJSVariant 自动管理引用计数
// tTJSVariant 的构造函数会调用 AddRef，析构函数会调用 Release
void safeUsage() {
    iTJSDispatch2* rawObj = TJSCreateArrayObject();
    // rawObj 引用计数 = 1

    {
        tTJSVariant wrapper(rawObj, rawObj);
        // wrapper 构造时 AddRef → 引用计数 = 2
        // 现在可以安全使用 wrapper

    } // wrapper 析构时 Release → 引用计数 = 1

    rawObj->Release();
    // 引用计数 = 0 → 对象销毁
}
```

## 三、tTJSVariant 与 tTJSVariantClosure

### 3.1 tTJSVariant — TJS 的万能变量类型

`tTJSVariant` 是 TJS 引擎中最重要的辅助类型。它对应 TJS 脚本中的变量——可以存储任何类型的值。在 C++ 插件代码中，几乎所有与 TJS 的数据交换都通过 `tTJSVariant` 进行。

```
┌────────────────────────────────────────────┐
│           tTJSVariant 可存储的类型           │
├─────────────────┬──────────────────────────┤
│ TJS 类型        │ C++ 对应                  │
├─────────────────┼──────────────────────────┤
│ tvtVoid         │ 空值（类似 undefined）     │
│ tvtObject       │ iTJSDispatch2* 对象引用   │
│ tvtString       │ tjs_char* 宽字符串        │
│ tvtOctet        │ tTJSVariantOctet 字节序列 │
│ tvtInteger      │ tjs_int64 (64位整数)      │
│ tvtReal          │ tjs_real (double)         │
└─────────────────┴──────────────────────────┘
```

`tTJSVariant` 支持与 C++ 基本类型之间的隐式转换：

```cpp
// tTJSVariant 的多种构造方式
tTJSVariant v1;                    // tvtVoid（空值）
tTJSVariant v2(42);                // tvtInteger（整数 42）
tTJSVariant v3(3.14);              // tvtReal（浮点数 3.14）
tTJSVariant v4(TJS_W("hello"));    // tvtString（宽字符串）
tTJSVariant v5(obj, obj);          // tvtObject（对象引用，同时指定 this）

// 类型查询
if (v2.Type() == tvtInteger) {
    tjs_int64 num = v2;            // 隐式转换为整数
}

// 类型转换
ttstr str = v2;                    // 整数 42 → 字符串 "42"
tjs_int64 num = v4;                // 字符串可能抛异常
```

### 3.2 tTJSVariantClosure — 闭包封装

`tTJSVariantClosure` 是一个轻量结构体，将一个 `iTJSDispatch2` 对象指针和一个 `this` 上下文打包在一起。它是 TJS 闭包（Closure）在 C++ 层面的表示：

```cpp
// tTJSVariantClosure 的结构（简化）
struct tTJSVariantClosure {
    iTJSDispatch2 *Object;   // 被调用的对象（方法/属性的持有者）
    iTJSDispatch2 *ObjThis;  // this 上下文（方法执行时的 this 指向）

    // 便利方法：直接在闭包上调用 PropGet、FuncCall 等
    // 内部会将 ObjThis 作为 objthis 参数传给 Object 的对应方法
    tjs_error PropGet(tjs_uint32 flag, const tjs_char *membername,
                      tjs_uint32 *hint, tTJSVariant *result,
                      iTJSDispatch2 *objthis);

    tjs_error FuncCall(tjs_uint32 flag, const tjs_char *membername,
                       tjs_uint32 *hint, tTJSVariant *result,
                       tjs_int numparams, tTJSVariant **param,
                       iTJSDispatch2 *objthis);

    tjs_error IsInstanceOf(tjs_uint32 flag, const tjs_char *membername,
                           tjs_uint32 *hint, const tjs_char *classname,
                           iTJSDispatch2 *objthis);
    // ... 更多便利方法
};
```

在 KrKr2 的 `scriptsEx.cpp` 插件中可以看到大量 `tTJSVariantClosure` 的使用：

```cpp
// 来源：cpp/plugins/scriptsEx.cpp 第 529-531 行
// 从 tTJSVariant 获取闭包引用
tTJSVariantClosure &o1 = v1.AsObjectClosureNoAddRef();
tTJSVariantClosure &o2 = v2.AsObjectClosureNoAddRef();

// 通过闭包检查对象类型
// IsInstanceOf 返回 TJS_S_TRUE 表示是指定类的实例
if (o1.IsInstanceOf(0, nullptr, nullptr, TJS_W("Array"), nullptr)
    == TJS_S_TRUE) {
    // o1 是一个数组对象
}

// 通过闭包读取属性
tTJSVariant countVal;
static tjs_uint32 countHint = 0;
o1.PropGet(0, TJS_W("count"), &countHint, &countVal, nullptr);
tjs_int count = countVal;  // 获取数组元素数量
```

### 3.3 EnumMembers — 成员遍历

`EnumMembers` 是一个强大的方法，用于遍历对象的所有成员。它使用回调模式——你传入一个回调对象（继承自 `tTJSDispatch`），引擎对每个成员调用回调的 `FuncCall`：

```cpp
// 来源：cpp/plugins/scriptsEx.cpp 第 257-288 行
// DictMemberGetCaller — 用于获取字典所有键名的回调类
class DictMemberGetCaller : public tTJSDispatch {
public:
    DictMemberGetCaller(iTJSDispatch2 *array) : array(array) {};

    // 引擎对字典的每个成员调用此方法
    virtual tjs_error FuncCall(
        tjs_uint32 flag,
        const tjs_char *membername,
        tjs_uint32 *hint,
        tTJSVariant *result,
        tjs_int numparams,
        tTJSVariant **param,       // param[0]=键名, param[1]=标志, param[2]=值
        iTJSDispatch2 *objthis
    ) {
        if (numparams > 1) {
            tTVInteger flag = param[1]->AsInteger();
            // 跳过隐藏成员
            if (!(flag & TJS_HIDDENMEMBER)) {
                static tjs_uint addHint = 0;
                // 将键名添加到结果数组
                array->FuncCall(0, TJS_W("add"), &addHint, 0, 1,
                                &param[0], array);
            }
        }
        if (result) {
            *result = true;  // 返回 true 表示继续遍历
        }
        return TJS_S_OK;
    }

protected:
    iTJSDispatch2 *array;  // 用于收集键名的数组
};
```

## 四、tp_stub.h — 原版与 KrKr2 的差异

### 4.1 原版 tp_stub.h 的角色

在原版 KiriKiri 引擎中，`tp_stub.h`（TJS Plugin Stub Header）是一个巨大的头文件（通常超过 5000 行）。它的职责是：

1. **声明引擎内部类型**：`iTJSDispatch2`、`tTJSVariant`、`tTJSVariantClosure` 等接口和类型的完整定义
2. **实现函数转发**：通过 `iTVPFunctionExporter` 查询获取引擎函数指针，然后提供包装函数供插件代码直接调用
3. **提供便利工具**：字符串转换、内存分配、异常处理等工具函数

原版 `tp_stub.h` 本质上是引擎和插件之间的"翻译层"——它让插件开发者可以像直接调用引擎 API 一样编写代码，而实际调用经过函数指针间接转发：

```
原版 KiriKiri 插件 DLL 内部：
┌─────────────────────────────────────────────┐
│  plugin.cpp                                  │
│    #include "tp_stub.h"  // 包含 5000+ 行    │
│                                              │
│    // 插件代码直接调用"引擎函数"               │
│    TVPAddLog(TJS_W("hello"));               │
│         ↓                                    │
│    // tp_stub.h 内部转发：                    │
│    // static_cast<void(*)(...)>(             │
│    //   exporter_table["TVPAddLog"]          │
│    // )(TJS_W("hello"));                     │
│                                              │
│    // 同样，iTJSDispatch2 的定义也来自此文件    │
│    // tTJSVariant 的实现也在此文件中           │
└─────────────────────────────────────────────┘
```

### 4.2 KrKr2 简化版 tp_stub.h

在 KrKr2 项目中，`tp_stub.h` 被大幅简化为仅 14 行的 include 集合：

```cpp
// 文件：cpp/plugins/tp_stub.h（完整文件）
#pragma once
#include "tjsCommHead.h"    // TJS 公共头（类型定义、宏）
#include "tjsNative.h"      // TJS 原生类基类
#include "ScriptMgnIntf.h"  // 脚本管理器接口
#include "tjsArray.h"       // TJS 数组实现
#include "tjsDictionary.h"  // TJS 字典实现
#include "DebugIntf.h"      // 调试接口（TVPAddLog 等）
#include "StorageImpl.h"    // 存储/文件系统实现
#include "TextStream.h"     // 文本流
#include "MsgIntf.h"        // 消息接口（TVPGetAboutString 等）
#include "PluginImpl.h"     // 插件实现（TVPLoadPlugin 等）
#include "CharacterSet.h"   // 字符集转换
#include "TransIntf.h"      // 转场效果接口
```

为什么能这么简化？因为 KrKr2 的插件是**静态链接**的——插件代码和引擎代码编译在同一个二进制文件中，可以直接 `#include` 引擎头文件并调用引擎函数，不需要函数指针转发层。

### 4.3 对比表

| 对比维度 | 原版 tp_stub.h | KrKr2 tp_stub.h |
|----------|---------------|-----------------|
| **行数** | 5000+ 行 | 14 行 |
| **内容** | iTJSDispatch2 完整定义 + 函数指针转发 + 工具函数 | 仅 #include 引擎头文件 |
| **iTJSDispatch2 来源** | tp_stub.h 内部重新声明 | 来自 tjsNative.h（引擎内部头文件） |
| **引擎函数调用** | 通过函数指针表间接调用 | 直接链接调用 |
| **编译依赖** | 仅依赖 tp_stub.h | 依赖整个引擎头文件树 |
| **适用场景** | DLL 插件（与引擎分离编译） | 静态链接插件（与引擎一起编译） |

### 4.4 逆向分析的启示

理解 tp_stub.h 的差异对逆向分析无源码插件非常重要：

1. **原版 DLL 插件**中的 `iTJSDispatch2` 调用最终都通过函数指针转发。在 Ghidra 反编译中，你会看到间接调用（`call [eax+0x1C]`），其中偏移量对应 vtable 中的方法位置
2. **偏移量→方法映射**是逆向的关键：如果 vtable 偏移 0x1C 对应 PropGet，那么 `call [eax+0x1C]` 就是 PropGet 调用
3. **KrKr2 不存在这些间接调用**——所有 iTJSDispatch2 方法都是直接虚函数调用

```
Ghidra 中识别 iTJSDispatch2 虚函数调用的方法：

vtable 偏移量（32 位，每个指针 4 字节）：
  +0x00  析构函数
  +0x04  AddRef
  +0x08  Release
  +0x0C  FuncCall
  +0x10  FuncCallByNum
  +0x14  PropGet
  +0x18  PropGetByNum
  +0x1C  PropSet
  +0x20  PropSetByNum
  +0x24  GetCount
  +0x28  CreateNew
  +0x2C  IsInstanceOf
  +0x30  IsValid
  +0x34  Invalidate
  +0x38  DeleteMember
  +0x3C  DeleteMemberByNum
  +0x40  EnumMembers

注意：实际偏移量可能因编译器和继承层次不同而有差异
在逆向分析时应通过交叉引用来验证
```

## 五、实战：在插件中使用 iTJSDispatch2

### 5.1 创建和操作字典

以下是一个完整的示例，展示如何在 KrKr2 插件中创建字典、设置属性、读取属性：

```cpp
// 文件：my_dict_plugin.cpp
// 演示如何在 KrKr2 插件中操作 TJS 字典对象
#include "ncbind.hpp"

#define NCB_MODULE_NAME TJS_W("my_dict_plugin.dll")

// 创建一个包含系统信息的字典并返回
static tTJSVariant getSystemInfo() {
    // 第一步：创建空字典对象
    iTJSDispatch2 *dict = TJSCreateDictionaryObject();
    // dict 引用计数 = 1

    // 第二步：向字典中设置属性
    tTJSVariant vName(TJS_W("KrKr2 Emulator"));
    dict->PropSet(
        TJS_MEMBERENSURE,      // 属性不存在则创建
        TJS_W("appName"),      // 属性名
        nullptr,               // 不使用 hint
        &vName,                // 值
        dict                   // this
    );

    tTJSVariant vVersion(TJS_W("2.0.0"));
    dict->PropSet(TJS_MEMBERENSURE, TJS_W("version"),
                  nullptr, &vVersion, dict);

    tTJSVariant vPlatform(TJS_W(
#if defined(_WIN32)
        "Windows"
#elif defined(__ANDROID__)
        "Android"
#elif defined(__APPLE__)
        "macOS"
#elif defined(__linux__)
        "Linux"
#else
        "Unknown"
#endif
    ));
    dict->PropSet(TJS_MEMBERENSURE, TJS_W("platform"),
                  nullptr, &vPlatform, dict);

    // 第三步：用 tTJSVariant 包装返回值（自动管理引用计数）
    tTJSVariant result(dict, dict);
    // result 构造时 AddRef → 引用计数 = 2

    dict->Release();
    // 引用计数 = 1（result 持有）

    return result;
    // 返回后 result 的引用由调用者管理
}

// 将函数注册到 TJS 的 System 对象上
// TJS 脚本可以通过 System.getSystemInfo() 调用
NCB_ATTACH_FUNCTION(getSystemInfo, System, getSystemInfo);
```

### 5.2 遍历数组并处理每个元素

```cpp
// 示例：将 TJS 数组中的所有字符串转为大写
// 展示 PropGetByNum / PropSetByNum 的用法
static tjs_error processArray(
    tTJSVariant *result,
    tjs_int numparams,
    tTJSVariant **param,
    iTJSDispatch2 *objthis
) {
    if (numparams < 1) return TJS_E_BADPARAMCOUNT;

    // 获取数组对象的闭包
    tTJSVariantClosure &arrayClo = param[0]->AsObjectClosureNoAddRef();

    // 读取数组长度
    tTJSVariant countVar;
    static tjs_uint32 countHint = 0;
    arrayClo.PropGet(0, TJS_W("count"), &countHint, &countVar, nullptr);
    tjs_int count = countVar;

    // 遍历每个元素
    for (tjs_int i = 0; i < count; i++) {
        tTJSVariant elemVar;

        // 按索引读取元素（PropGetByNum 用数字索引代替字符串名称）
        arrayClo.PropGetByNum(
            TJS_IGNOREPROP,  // 跳过属性 getter
            i,               // 索引
            &elemVar,        // 输出
            nullptr          // this
        );

        // 检查是否为字符串类型
        if (elemVar.Type() == tvtString) {
            ttstr str = elemVar;
            // 转大写（ttstr 提供 ToUpper 方法）
            str.ToUpper();

            tTJSVariant newVal(str);
            // 按索引写回
            arrayClo.PropSetByNum(
                TJS_MEMBERENSURE,
                i,
                &newVal,
                nullptr
            );
        }
    }

    if (result) *result = (tjs_int)count;
    return TJS_S_OK;
}
```

## 动手实践

### 实践 1：阅读 scriptsEx.cpp 中的 iTJSDispatch2 用法

打开 `cpp/plugins/scriptsEx.cpp`，这是 KrKr2 项目中使用 iTJSDispatch2 最密集的插件文件。重点观察：

1. **第 19-42 行**：`t_iTJSDispatch2_PropGet` 结构体——为什么要把 PropGet 的参数封装成结构体？（提示：与异常安全有关，见 `TVPDoTryBlock` 调用）
2. **第 257-288 行**：`DictMemberGetCaller` 类——这是 `EnumMembers` 回调模式的经典范例
3. **第 968-991 行**：`NCB_ATTACH_CLASS(ScriptsAdd, Scripts)` 块——注意 `RawCallback` 与 `NCB_METHOD` 的区别

### 实践 2：追踪 tTJSVariant 的类型转换

在 `cpp/core/tjs2/` 目录下找到 `tTJSVariant` 的定义（通常在 `tjsVariant.h` 中），观察：
1. 它如何通过联合体（union）在同一内存区域存储不同类型的值
2. `Type()` 方法如何返回当前存储的类型枚举
3. 隐式类型转换运算符（`operator tjs_int64()`、`operator ttstr()` 等）的实现

## 对照项目源码

| 文件路径 | 行号范围 | 知识点 |
|----------|----------|--------|
| `cpp/plugins/tp_stub.h` | 全文（14 行） | KrKr2 简化版 tp_stub——仅 include 引擎头文件 |
| `cpp/plugins/scriptsEx.cpp` | 第 19-42 行 | iTJSDispatch2 方法的异常安全封装（Try_ 前缀函数） |
| `cpp/plugins/scriptsEx.cpp` | 第 257-288 行 | EnumMembers 回调模式（DictMemberGetCaller） |
| `cpp/plugins/scriptsEx.cpp` | 第 506-586 行 | 深度结构体比较——递归使用 PropGet/PropGetByNum |
| `cpp/plugins/scriptsEx.cpp` | 第 968-991 行 | NCB_ATTACH_CLASS 注册块——RawCallback 与 NCB_METHOD |
| `cpp/core/plugin/ncbind.hpp` | 第 45-55 行 | ncbTypeConvertor——tTJSVariant 类型自动转换机制 |

## 常见错误与排查

### 错误 1：忘记 Release 导致内存泄漏

**症状**：程序运行时间越来越慢，内存持续增长。

**原因**：创建了 iTJSDispatch2 对象但忘记调用 Release。

```cpp
// 错误：内存泄漏
void leakyCode() {
    iTJSDispatch2 *dict = TJSCreateDictionaryObject();
    // ... 使用 dict ...
    // 忘记 dict->Release() → 内存泄漏！
}

// 正确：通过 tTJSVariant 自动管理
void safeCode() {
    iTJSDispatch2 *rawDict = TJSCreateDictionaryObject();
    tTJSVariant dict(rawDict, rawDict);  // RAII 管理
    rawDict->Release();                   // 释放创建时的引用
    // dict 析构时自动 Release → 无泄漏
}
```

### 错误 2：PropGet 返回 TJS_E_MEMBERNOTFOUND

**症状**：属性读取失败，返回 `TJS_E_MEMBERNOTFOUND`。

**原因**：属性名拼写错误，或使用了 `TJS_MEMBERMUSTEXIST` 标志但属性不存在。

**解决**：检查属性名拼写；使用默认标志 `0` 而非 `TJS_MEMBERMUSTEXIST`，然后检查返回的 variant 是否为 void 类型。

### 错误 3：hint 变量未初始化为 0

**症状**：PropGet/FuncCall 有时能正常工作，有时返回错误结果。

**原因**：hint 变量必须初始化为 0。未初始化的随机值会被引擎当作有效缓存使用。

```cpp
// 错误：hint 未初始化
tjs_uint32 hint;  // 随机值！
obj->PropGet(0, TJS_W("count"), &hint, &result, obj);

// 正确：hint 初始化为 0（推荐用 static 保持缓存）
static tjs_uint32 hint = 0;
obj->PropGet(0, TJS_W("count"), &hint, &result, obj);
```

## 本节小结

- **iTJSDispatch2** 是 TJS2 引擎中所有对象的统一接口，灵感来自 COM IDispatch
- **核心方法**：PropGet（读属性）、PropSet（写属性）、FuncCall（调方法）、CreateNew（创建实例）、EnumMembers（遍历成员）
- **tTJSVariant** 是万能变量类型，支持存储整数、浮点、字符串、对象等，提供自动引用计数管理
- **tTJSVariantClosure** 将对象指针和 this 上下文打包，是闭包调用的 C++ 表示
- **原版 tp_stub.h** 是 5000+ 行的巨型头文件，包含函数指针转发层
- **KrKr2 tp_stub.h** 仅 14 行 include，因为静态链接不需要转发层
- **逆向关键**：识别 vtable 偏移量→iTJSDispatch2 方法的映射关系

## 练习题与答案

### 题目 1：iTJSDispatch2 方法识别

以下 Ghidra 反编译伪代码中，`call [eax+0x14]` 对应 `iTJSDispatch2` 的哪个方法？如果参数中包含字符串 `"visible"` 和一个 `tTJSVariant*` 输出指针，这行代码最可能在做什么？

<details>
<summary>查看答案</summary>

根据 vtable 偏移量表，`+0x14` 对应 `PropGet` 方法。

结合参数分析：字符串 `"visible"` 是属性名，`tTJSVariant*` 是输出参数。这行代码最可能在读取某个 TJS 对象的 `visible` 属性，等价于 TJS 脚本中的 `var v = obj.visible;`。

这在 KiriKiri 游戏中非常常见——检查图层或窗口的可见性状态。

</details>

### 题目 2：引用计数追踪

分析以下代码，标出每一步的引用计数变化，并指出是否存在内存泄漏：

```cpp
iTJSDispatch2* arr = TJSCreateArrayObject();
tTJSVariant v1(arr, arr);
tTJSVariant v2 = v1;
arr->Release();
// v1 和 v2 在此处离开作用域
```

<details>
<summary>查看答案</summary>

逐步分析引用计数：

1. `TJSCreateArrayObject()` → 引用计数 = **1**
2. `tTJSVariant v1(arr, arr)` → v1 的构造函数调用 AddRef → 引用计数 = **2**
3. `tTJSVariant v2 = v1` → v2 的拷贝构造函数调用 AddRef → 引用计数 = **3**
4. `arr->Release()` → 引用计数 = **2**
5. `v2` 离开作用域，析构函数调用 Release → 引用计数 = **1**
6. `v1` 离开作用域，析构函数调用 Release → 引用计数 = **0** → 对象被销毁

**结论：无内存泄漏。** 所有引用都被正确释放，最终引用计数归零。这是使用 `tTJSVariant` 管理 iTJSDispatch2 对象生命周期的标准模式。

</details>

### 题目 3：编写一个完整的 TJS 对象操作程序

编写一个独立的 C++ 程序，模拟 iTJSDispatch2 的 PropGet 和 PropSet 操作。要求：
- 定义一个简化版 `iTJSDispatch2` 接口（只需 PropGet 和 PropSet 两个方法）
- 实现一个基于 `std::unordered_map` 的字典类
- 在 main 中创建字典、设置属性、读取属性

<details>
<summary>查看答案</summary>

```cpp
// 文件：dispatch_simulation.cpp
// 编译：g++ -std=c++17 -o dispatch_sim dispatch_simulation.cpp
// 或：cl /std:c++17 /EHsc dispatch_simulation.cpp

#include <iostream>
#include <string>
#include <unordered_map>
#include <variant>

// 模拟 TJS 错误码
enum TJSError {
    TJS_S_OK = 0,
    TJS_E_MEMBERNOTFOUND = -1,
    TJS_E_BADPARAMCOUNT = -2
};

// 模拟 tTJSVariant — 万能变量类型
class SimpleVariant {
public:
    using Value = std::variant<std::monostate, int64_t, double, std::wstring>;

    SimpleVariant() : value_(std::monostate{}) {}
    SimpleVariant(int64_t v) : value_(v) {}
    SimpleVariant(double v) : value_(v) {}
    SimpleVariant(const wchar_t* v) : value_(std::wstring(v)) {}

    // 类型检查
    bool isVoid() const { return std::holds_alternative<std::monostate>(value_); }
    bool isInteger() const { return std::holds_alternative<int64_t>(value_); }
    bool isReal() const { return std::holds_alternative<double>(value_); }
    bool isString() const { return std::holds_alternative<std::wstring>(value_); }

    // 类型转换
    int64_t asInteger() const { return std::get<int64_t>(value_); }
    std::wstring asString() const { return std::get<std::wstring>(value_); }

    // 打印
    void print() const {
        if (isVoid()) std::wcout << L"(void)";
        else if (isInteger()) std::wcout << asInteger();
        else if (isReal()) std::wcout << std::get<double>(value_);
        else if (isString()) std::wcout << L"\"" << asString() << L"\"";
    }

private:
    Value value_;
};

// 模拟 iTJSDispatch2 接口（简化版，只有 PropGet 和 PropSet）
class SimpleDispatch {
public:
    virtual ~SimpleDispatch() = default;
    virtual TJSError PropGet(const std::wstring& name, SimpleVariant* result) = 0;
    virtual TJSError PropSet(const std::wstring& name, const SimpleVariant& value) = 0;
};

// 模拟 TJS 字典——基于 unordered_map 实现 SimpleDispatch
class SimpleDictionary : public SimpleDispatch {
public:
    TJSError PropGet(const std::wstring& name, SimpleVariant* result) override {
        auto it = members_.find(name);
        if (it == members_.end()) {
            return TJS_E_MEMBERNOTFOUND;
        }
        if (result) *result = it->second;
        return TJS_S_OK;
    }

    TJSError PropSet(const std::wstring& name, const SimpleVariant& value) override {
        members_[name] = value;
        return TJS_S_OK;
    }

private:
    std::unordered_map<std::wstring, SimpleVariant> members_;
};

int main() {
    std::wcout << L"=== iTJSDispatch2 模拟演示 ===" << std::endl;

    // 创建字典对象
    SimpleDictionary dict;
    SimpleDispatch* dispatch = &dict;  // 通过接口指针操作

    // PropSet：设置属性
    dispatch->PropSet(L"name", SimpleVariant(L"KrKr2 Emulator"));
    dispatch->PropSet(L"version", SimpleVariant(L"2.0.0"));
    dispatch->PropSet(L"buildNumber", SimpleVariant(int64_t(427)));

    std::wcout << L"[设置] 3 个属性已写入字典" << std::endl;

    // PropGet：读取属性
    SimpleVariant result;
    if (dispatch->PropGet(L"name", &result) == TJS_S_OK) {
        std::wcout << L"[读取] name = ";
        result.print();
        std::wcout << std::endl;
    }

    if (dispatch->PropGet(L"buildNumber", &result) == TJS_S_OK) {
        std::wcout << L"[读取] buildNumber = ";
        result.print();
        std::wcout << std::endl;
    }

    // 读取不存在的属性
    TJSError err = dispatch->PropGet(L"nonExistent", &result);
    if (err == TJS_E_MEMBERNOTFOUND) {
        std::wcout << L"[读取] nonExistent → TJS_E_MEMBERNOTFOUND（属性不存在）"
                   << std::endl;
    }

    return 0;
}
```

**预期输出：**
```
=== iTJSDispatch2 模拟演示 ===
[设置] 3 个属性已写入字典
[读取] name = "KrKr2 Emulator"
[读取] buildNumber = 427
[读取] nonExistent → TJS_E_MEMBERNOTFOUND（属性不存在）
```

</details>

## 下一步

下一节 [03-ncbind 注册框架详解](./03-ncbind注册框架详解.md) 将深入讲解 KrKr2 的 ncbind 宏系统——它如何将 C++ 类和函数自动注册为 TJS 可调用对象，以及 `NCB_REGISTER_CLASS`、`NCB_ATTACH_FUNCTION` 等宏的内部实现原理。

