# scriptsEx 与 csvParser 源码导读

> **所属模块：** M09-插件系统与开发
> **前置知识：** [ncbind 类注册与宏体系](../02-ncbind绑定框架/02-类注册与宏体系.md)、[属性方法与回调绑定](../02-ncbind绑定框架/03-属性方法与回调绑定.md)
> **预计阅读时间：** 45 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：

1. 理解 scriptsEx 插件如何通过 `NCB_ATTACH_CLASS` 向已有 TJS2 类（`Scripts`）注入新方法
2. 掌握 `Try_` 安全包装器（Safe Wrapper）模式，学会用 `TVPDoTryBlock` 在 C++ 中安全调用 TJS2 API
3. 理解 `RawCallback`、`NCB_METHOD`、`Variant` 三种注册方式的选择依据
4. 理解 csvParser 插件的手动类创建模式（`TJSCreateNativeClassForPlugin` + `TJS_BEGIN_NATIVE_MEMBERS`）
5. 对比 ncbind 自动注册与手动 TJS 宏注册的优劣，能够为不同场景选择合适的绑定方案

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| NCB_ATTACH_CLASS | N/A (宏名) | ncbind 提供的宏，用于向一个**已存在**的 TJS2 类追加 C++ 方法，而非创建新类 |
| RawCallback | Raw Callback | ncbind 中注册"原始回调"函数的方式——函数签名必须是 `tjs_error(tTJSVariant*, tjs_int, tTJSVariant**, iTJSDispatch2*)` |
| TVPDoTryBlock | TVP Do Try Block | KiriKiri 引擎提供的 C 风格异常捕获机制，用于安全执行可能抛出 TJS2 异常的代码 |
| tTJSNativeInstance | TJS Native Instance | TJS2 原生实例基类——手动创建 TJS 类时，C++ 侧实例必须继承此类 |
| TJS_BEGIN_NATIVE_MEMBERS | N/A (宏名) | KiriKiri 原始 TJS 类注册宏，在 ncbind 出现之前的"老式"注册方式 |
| NCB_PRE_REGIST_CALLBACK | N/A (宏名) | ncbind 的预注册回调宏——让手动注册代码也能搭上 ncbind 的自动初始化时机 |
| EnumMembers | Enum Members | iTJSDispatch2 的成员枚举方法——遍历 Dictionary/Object 的所有键值对 |

---

## 一、scriptsEx 插件概览

### 1.1 scriptsEx 是什么

`scriptsEx`（全称 Scripts Extension，脚本扩展）是一个**功能增强型**插件。它不创建新的 TJS2 类，而是向 TJS2 引擎中**已有的** `Scripts` 全局对象追加一组实用方法。这些方法在原版 KiriKiri2 引擎中不存在，由社区开发者（wamsoft）开发后作为 DLL 插件分发。

在 KrKr2 模拟器中，scriptsEx 被静态编译进主程序，源码位于：

```
cpp/plugins/scriptsEx.cpp    （993 行，单文件插件）
```

scriptsEx 追加的方法涵盖以下功能领域：

| 方法名 | 功能 | 注册方式 |
|--------|------|----------|
| `getObjectKeys` | 获取 Dictionary 的所有键名（排序后返回 Array） | RawCallback |
| `getObjectCount` | 获取 Dictionary/Array 的成员数量 | RawCallback |
| `getObjectContext` | 获取对象的 `this` 上下文 | NCB_METHOD |
| `isNullContext` | 判断对象上下文是否为 nullptr | NCB_METHOD |
| `equalStruct` | 深度结构体比较（严格类型匹配） | NCB_METHOD |
| `equalStructNumericLoose` | 深度结构体比较（数字类型宽松匹配） | NCB_METHOD |
| `foreach` | 遍历 Array/Dictionary 的所有元素 | RawCallback |
| `getMD5HashString` | 计算 octet 数据的 MD5 哈希值 | RawCallback |
| `clone` | 深拷贝对象（递归复制 Array/Dictionary） | NCB_METHOD |
| `propSet` | 带标志位的属性设置 | RawCallback |
| `propGet` | 带标志位的属性获取 | RawCallback |
| `safeEvalStorage` | 安全评估（eval）存储文件中的常量表达式 | RawCallback |
| `rehash` | 重新计算 TJS2 的全局哈希表 | NCB_ATTACH_FUNCTION |

### 1.2 整体架构

scriptsEx 的代码结构可以分为四个层次：

```
┌─────────────────────────────────────────────────────────┐
│              TJS2 脚本层 (Scripts 全局对象)                │
│  Scripts.getObjectKeys(dict)                             │
│  Scripts.foreach(obj, func)                              │
│  Scripts.clone(obj)                                      │
└──────────────────────────┬──────────────────────────────┘
                           │ NCB_ATTACH_CLASS(ScriptsAdd, Scripts)
┌──────────────────────────▼──────────────────────────────┐
│              ScriptsAdd 类 (C++ 静态方法集合)              │
│  getKeys(), getCount(), foreach(), clone(), ...          │
└──────────────────────────┬──────────────────────────────┘
                           │ 调用
┌──────────────────────────▼──────────────────────────────┐
│              辅助类层                                      │
│  DictMemberGetCaller      — 键名枚举回调                  │
│  DictMemberCompareCaller  — 结构体比较回调                 │
│  DictIterateCaller        — 遍历回调                      │
│  DictMemberCloneCaller    — 深拷贝回调                    │
└──────────────────────────┬──────────────────────────────┘
                           │ 调用
┌──────────────────────────▼──────────────────────────────┐
│              Try_ 安全包装器层                              │
│  Try_iTJSDispatch2_PropGet()                              │
│  Try_iTJSDispatch2_PropSet()                              │
│  Try_iTJSDispatch2_PropGetByNum()                         │
│  Try_iTJSDispatch2_PropSetByNum()                         │
│  Try_iTJSDispatch2_AddRef()                               │
└─────────────────────────────────────────────────────────┘
```

这个分层设计的核心思想是：**TJS2 的异常系统和 C++ 的异常系统不兼容**。TJS2 使用自己的异常传播机制（基于 `longjmp`/setjmp 风格），不能直接在 C++ 的 `try/catch` 中捕获。因此 scriptsEx 在最底层封装了一组 `Try_` 包装器，通过 KiriKiri 引擎提供的 `TVPDoTryBlock` 函数安全地调用 `iTJSDispatch2` 接口。

## 二、Try_ 安全包装器详解

### 2.1 为什么需要安全包装器

在普通 C++ 代码中，我们调用 `iTJSDispatch2` 的方法（如 `PropGet`、`PropSet`）时，如果 TJS2 脚本层的属性访问器（getter/setter）内部抛出了 TJS2 异常，这个异常不会被 C++ 的 `try/catch` 捕获。原因是 TJS2 的异常传播使用了引擎内部的 `longjmp` 机制，会直接跳过 C++ 的栈展开（Stack Unwinding，即 C++ 在异常传播时逐层调用析构函数清理局部对象的过程），导致 C++ 侧的局部对象不被析构，造成资源泄漏甚至程序崩溃。

`TVPDoTryBlock` 是 KiriKiri 引擎提供的解决方案。它的签名如下：

```cpp
// KiriKiri 引擎内部 API
// tryFunc: 要执行的函数指针（可能抛出 TJS2 异常）
// catchFunc: 异常捕获回调（nullptr 表示忽略异常）
// finallyFunc: 最终回调（类似 finally 块，总是执行）
// data: 传给上述三个函数的用户数据指针
void TVPDoTryBlock(
    void (TJS_USERENTRY *tryFunc)(void *data),     // "try" 块
    bool (TJS_USERENTRY *catchFunc)(void *data,     // "catch" 块
                                    const tTVPExceptionDesc &desc),
    void (TJS_USERENTRY *finallyFunc)(void *data),  // "finally" 块
    void *data                                       // 用户数据
);
```

这里 `TJS_USERENTRY` 是一个调用约定宏（Calling Convention Macro，指定函数参数如何传递的编译器约定），在不同平台上可能展开为 `__stdcall`（Windows）或空（其他平台）。`tTVPExceptionDesc` 是 TJS2 异常描述结构体，包含异常消息字符串。

### 2.2 scriptsEx 中的包装器实现模式

scriptsEx 为 `iTJSDispatch2` 的 5 个常用方法各自实现了一个 `Try_` 包装器。每个包装器遵循相同的三步模式：

**第一步：定义参数结构体**——把所有函数参数打包到一个结构体中，因为 `TVPDoTryBlock` 只接受一个 `void*` 数据指针。

**第二步：定义执行函数**——从 `void*` 恢复结构体，调用真正的 `iTJSDispatch2` 方法。

**第三步：包装函数**——组装结构体，调用 `TVPDoTryBlock`，返回结果。

以 `PropGet` 的包装器为例（源码位于 `cpp/plugins/scriptsEx.cpp` 第 26-59 行）：

```cpp
// 第一步：参数结构体 —— 把 PropGet 的全部参数装进一个 struct
// 源码：cpp/plugins/scriptsEx.cpp 第 26-42 行
struct t_iTJSDispatch2_PropGet {
    tjs_error _ret;              // 存放返回值
    iTJSDispatch2 *_this;        // 调用目标对象
    tjs_uint32 flag;             // 访问标志（如 TJS_MEMBERENSURE）
    const tjs_char *membername;  // 属性名
    tjs_uint32 *hint;            // 哈希提示（加速查找，可为 nullptr）
    tTJSVariant *result;         // 输出结果
    iTJSDispatch2 *objthis;      // this 上下文

    // 构造函数：把所有参数存入结构体
    t_iTJSDispatch2_PropGet(iTJSDispatch2 *_this_, tjs_uint32 flag_,
                            const tjs_char *membername_, tjs_uint32 *hint_,
                            tTJSVariant *result_, iTJSDispatch2 *objthis_)
        : _this(_this_), flag(flag_), membername(membername_),
          hint(hint_), result(result_), objthis(objthis_) {}
};

// 第二步：执行函数 —— 从 void* 恢复结构体，调用真正的 PropGet
// 源码：cpp/plugins/scriptsEx.cpp 第 44-50 行
static void TJS_USERENTRY _Try_iTJSDispatch2_PropGet(void *data) {
    t_iTJSDispatch2_PropGet *arg = (t_iTJSDispatch2_PropGet *)data;
    arg->_ret = arg->_this->PropGet(
        arg->flag, arg->membername, arg->hint,
        arg->result, arg->objthis
    );
}

// 第三步：包装函数 —— 组装结构体 + TVPDoTryBlock + 返回结果
// 源码：cpp/plugins/scriptsEx.cpp 第 52-59 行
tjs_error Try_iTJSDispatch2_PropGet(
    iTJSDispatch2 *_this, tjs_uint32 flag,
    const tjs_char *membername, tjs_uint32 *hint,
    tTJSVariant *result, iTJSDispatch2 *objthis)
{
    // 把所有参数打包成结构体
    t_iTJSDispatch2_PropGet arg(_this, flag, membername, hint, result, objthis);
    // 通过 TVPDoTryBlock 安全执行
    // _CatchFuncCall 负责将 TJS2 异常转换为 C++ 异常重新抛出
    TVPDoTryBlock(_Try_iTJSDispatch2_PropGet, _CatchFuncCall, nullptr, &arg);
    return arg._ret;  // 从结构体中取出返回值
}
```

### 2.3 异常捕获回调

scriptsEx 定义了一个通用的异常捕获回调 `_CatchFuncCall`（第 13-17 行）：

```cpp
// 源码：cpp/plugins/scriptsEx.cpp 第 13-17 行
static bool TJS_USERENTRY
_CatchFuncCall(void *data, const tTVPExceptionDesc &desc) {
    throw desc;  // 将 TJS2 异常转换为 C++ 异常重新抛出
}
```

这里的设计意图是：把 TJS2 的内部异常"桥接"到 C++ 的异常系统中。`TVPDoTryBlock` 的 try 块执行过程中如果发生 TJS2 异常，引擎会调用 catchFunc（即 `_CatchFuncCall`），此回调通过 `throw desc` 将异常转换为 C++ 异常。这样调用 `Try_` 包装器的上层代码就可以用标准 C++ 的 `try/catch` 来处理了。

### 2.4 完整的包装器清单

scriptsEx 一共实现了 5 个 `Try_` 包装器：

| 包装器函数 | 对应的 iTJSDispatch2 方法 | 源码行号 | 用途 |
|-----------|------------------------|---------|------|
| `Try_iTJSDispatch2_PropGet` | `PropGet(flag, name, hint, result, objthis)` | 52-59 | 按名称获取属性值 |
| `Try_iTJSDispatch2_PropSet` | `PropSet(flag, name, hint, param, objthis)` | 137-144 | 按名称设置属性值 |
| `Try_iTJSDispatch2_PropGetByNum` | `PropGetByNum(flag, num, result, objthis)` | 120-127 | 按数字索引获取属性值 |
| `Try_iTJSDispatch2_PropSetByNum` | `PropSetByNum(flag, num, param, objthis)` | 171-178 | 按数字索引设置属性值 |
| `Try_iTJSDispatch2_AddRef` | `AddRef()` | 70-74 | 安全增加引用计数 |

这 5 个包装器仅在 `ScriptsAdd::propGet` 和 `ScriptsAdd::propSet` 方法中使用——因为这两个方法允许 TJS2 脚本传入任意标志位（flag）来操作属性，执行过程中更容易触发 TJS2 异常。其余 `ScriptsAdd` 方法直接调用 `iTJSDispatch2` 的方法而不通过包装器，因为它们使用的标志位是确定的、安全的。

## 三、ScriptsAdd 类 —— 方法实现详解

`ScriptsAdd` 类（第 183-252 行声明，第 438-966 行实现）是 scriptsEx 的核心。它包含的全部方法都是 `static` 的——这是因为 scriptsEx 不需要实例化，所有方法都是工具函数，通过 `Scripts.xxx()` 直接调用。

### 3.1 辅助回调类（Callback Helper Classes）

在理解 ScriptsAdd 的方法之前，需要先了解它使用的 4 个辅助回调类。这些类都继承自 `tTJSDispatch`（TJS2 的调度接口基类，所有 TJS2 对象的底层接口），通过重写 `FuncCall` 虚函数来实现自定义逻辑。它们被用于 `iTJSDispatch2::EnumMembers` 方法——该方法遍历 Dictionary 的每个成员时，对每个成员调用一次回调的 `FuncCall`。

```
iTJSDispatch2::EnumMembers 的回调协议：
  param[0] = 成员名（tjs_char* 字符串）
  param[1] = 成员标志（TJS_HIDDENMEMBER 等）
  param[2] = 成员值（tTJSVariant）
  返回 true 继续枚举，false 停止
```

**DictMemberGetCaller**（第 257-288 行）—— 获取所有键名：

```cpp
// 源码：cpp/plugins/scriptsEx.cpp 第 257-288 行
class DictMemberGetCaller : public tTJSDispatch {
public:
    // array: 用于存放所有键名的 TJS2 数组对象
    DictMemberGetCaller(iTJSDispatch2 *array) : array(array) {};

    virtual tjs_error FuncCall(
        tjs_uint32 flag, const tjs_char *membername,
        tjs_uint32 *hint, tTJSVariant *result,
        tjs_int numparams, tTJSVariant **param,
        iTJSDispatch2 *objthis)
    {
        if (numparams > 1) {
            tTVInteger flag = param[1]->AsInteger();
            static tjs_uint addHint = 0;
            // 跳过隐藏成员（TJS2 内部属性）
            if (!(flag & TJS_HIDDENMEMBER)) {
                // 将键名添加到数组中
                array->FuncCall(0, TJS_W("add"), &addHint, 0,
                                1, &param[0], array);
            }
        }
        if (result) { *result = true; }  // 返回 true 继续枚举
        return TJS_S_OK;
    }
protected:
    iTJSDispatch2 *array;  // 收集键名的目标数组
};
```

**DictMemberCompareCaller**（第 310-351 行）—— 递归深度比较 Dictionary 的每个成员：

```cpp
// 源码：cpp/plugins/scriptsEx.cpp 第 310-351 行
// 关键逻辑：遍历 Dictionary A 的每个成员，与 Dictionary B 中同名成员比较
class DictMemberCompareCaller : public tTJSDispatch {
public:
    tTJSVariantClosure &target;  // 比较目标（Dictionary B）
    bool match;                   // 当前是否仍然匹配

    DictMemberCompareCaller(tTJSVariantClosure &_target)
        : target(_target), match(true) {}

    virtual tjs_error FuncCall(...) {
        if (numparams > 1) {
            if ((int)*param[1] != TJS_HIDDENMEMBER) {
                const tjs_char *key = param[0]->GetString();
                tTJSVariant value = *param[2];
                tTJSVariant targetValue;
                // 从 B 中获取同名成员
                if (target.PropGet(TJS_MEMBERMUSTEXIST, key,
                                   nullptr, &targetValue, nullptr)
                    == TJS_S_OK) {
                    // 递归深度比较
                    match = match &&
                            ScriptsAdd::equalStruct(value, targetValue);
                } else {
                    match = false;  // B 中不存在此键 → 不匹配
                }
            }
        }
        return TJS_S_OK;
    }
};
```

**DictIterateCaller**（第 395-432 行）—— `foreach` 方法的 Dictionary 遍历回调，支持提前中断（返回非 void 值即中断）。

**DictMemberCloneCaller**（第 785-809 行）—— `clone` 方法的深拷贝回调，遍历源 Dictionary 的每个成员，递归克隆后写入新 Dictionary。

### 3.2 键枚举与计数方法

**getKeys**（第 457-463 行）—— 获取 Dictionary 的所有键名并排序返回：

```cpp
// 源码：cpp/plugins/scriptsEx.cpp 第 438-452 行（内部实现）
void ScriptsAdd::_getKeys(tTJSVariant *result, tTJSVariant &obj) {
    if (result) {
        iTJSDispatch2 *array = TJSCreateArrayObject();  // 创建空数组
        DictMemberGetCaller *caller = new DictMemberGetCaller(array);
        tTJSVariantClosure closure(caller);
        // 枚举所有成员，TJS_ENUM_NO_VALUE 表示不需要值（只要键名）
        obj.AsObjectClosureNoAddRef().EnumMembers(
            TJS_IGNOREPROP | TJS_ENUM_NO_VALUE, &closure, nullptr);
        caller->Release();  // 释放回调对象
        static tjs_uint sortHint = 0;
        // 对键名数组排序（保证结果确定性）
        array->FuncCall(0, TJS_W("sort"), &sortHint, 0, 0, 0, array);
        *result = tTJSVariant(array, array);
        array->Release();
    }
}
```

TJS2 脚本中的使用方式：

```javascript
// TJS2 脚本
var dict = %["name" => "Alice", "age" => 25, "city" => "Tokyo"];
var keys = Scripts.getObjectKeys(dict);
// keys == ["age", "city", "name"] （按字母排序）
```

**getCount**（第 468-483 行）—— 通过 `GetCount` 方法获取成员数量，返回整数。

### 3.3 深度结构体比较

`equalStruct`（第 506-586 行）是 scriptsEx 中最复杂的方法之一。它实现了 TJS2 对象的**递归深度比较**——不仅比较引用是否相同，还比较内容是否逻辑等价。

比较逻辑按对象类型分层处理：

```
equalStruct(v1, v2) 判定流程：
┌─ v1, v2 都是 Object 类型？
│  ├─ 引用相同（同一对象）? → true
│  ├─ 都是 Function? → DiscernCompare（严格比较）
│  ├─ 都是 Array?
│  │   ├─ count 不同? → false
│  │   └─ 逐元素递归 equalStruct → 全部匹配则 true
│  ├─ 都是 Dictionary?
│  │   ├─ 键列表不同? → false
│  │   └─ EnumMembers + DictMemberCompareCaller 逐项递归比较
│  └─ 其他对象类型 → DiscernCompare
└─ 非 Object 类型 → DiscernCompare（基本类型严格比较）
```

`equalStructNumericLoose`（第 590-677 行）的逻辑几乎相同，唯一区别在于对数字类型使用 `NormalCompare` 而非 `DiscernCompare`——前者会进行数值转换（如 `1` == `1.0`），后者严格区分整数和浮点数。

```javascript
// TJS2 脚本中的使用差异
var a = %["x" => 1];
var b = %["x" => 1.0];

Scripts.equalStruct(a, b);             // false — 1 (Integer) ≠ 1.0 (Real)
Scripts.equalStructNumericLoose(a, b); // true  — 1 == 1.0（数值相等）
```

### 3.4 深拷贝（clone）

`clone`（第 813-865 行）实现了 TJS2 对象的递归深拷贝：

```cpp
// 源码：cpp/plugins/scriptsEx.cpp 第 813-865 行（简化展示）
tTJSVariant ScriptsAdd::clone(tTJSVariant obj) {
    if (obj.Type() == tvtObject) {
        tTJSVariantClosure &o1 = obj.AsObjectClosureNoAddRef();

        // Array 的深拷贝：创建新数组，逐元素递归 clone
        if (o1.IsInstanceOf(..., TJS_W("Array"), ...) == TJS_S_TRUE) {
            iTJSDispatch2 *array = TJSCreateArrayObject();
            tjs_int count = /* 获取 count 属性 */;
            for (tjs_int i = 0; i < count; i++) {
                tTJSVariant val;
                o1.PropGetByNum(TJS_IGNOREPROP, i, &val, nullptr);
                val = ScriptsAdd::clone(val);  // 递归深拷贝
                array->FuncCall(0, TJS_W("add"), ...);
            }
            return tTJSVariant(array, array);
        }

        // Dictionary 的深拷贝：使用 DictMemberCloneCaller 遍历
        if (o1.IsInstanceOf(..., TJS_W("Dictionary"), ...) == TJS_S_TRUE) {
            iTJSDispatch2 *dict = TJSCreateDictionaryObject();
            DictMemberCloneCaller *caller = new DictMemberCloneCaller(dict);
            tTJSVariantClosure closure(caller);
            o1.EnumMembers(TJS_IGNOREPROP, &closure, nullptr);
            caller->Release();
            return tTJSVariant(dict, dict);
        }

        // 其他对象：尝试调用对象自身的 clone 方法
        tTJSVariant result;
        if (o1.FuncCall(0, TJS_W("clone"), ..., &result, ...) == TJS_S_TRUE) {
            return result;
        }
    }
    return obj;  // 基本类型直接返回（值拷贝）
}
```

### 3.5 foreach 遍历

`foreach`（第 681-749 行）是一个统一的 Array/Dictionary 遍历方法：

```javascript
// TJS2 脚本用法
// 遍历 Array —— callback(index, value, ...extraArgs)
Scripts.foreach([10, 20, 30], function(key, value) {
    Debug.message("index=" + key + " value=" + value);
});
// 输出: index=0 value=10, index=1 value=20, index=2 value=30

// 遍历 Dictionary —— callback(key, value, ...extraArgs)
Scripts.foreach(%["a" => 1, "b" => 2], function(key, value) {
    Debug.message(key + "=" + value);
});
// 输出: a=1, b=2

// 提前中断：回调返回非 void 值即停止遍历
Scripts.foreach([1, 2, 3, 4, 5], function(key, value) {
    if (value > 3) return "stopped";  // 返回非 void → 中断
    Debug.message(value);
});
// 输出: 1, 2, 3 （4 和 5 不会被访问）
```

Array 的遍历直接用 `PropGetByNum` 逐索引访问；Dictionary 的遍历通过 `DictIterateCaller` 配合 `EnumMembers` 实现。两种路径都支持传递额外参数给回调函数。

## 四、NCB_ATTACH_CLASS 注册机制

### 4.1 NCB_ATTACH_CLASS vs NCB_REGISTER_CLASS

在前面章节中我们学过 `NCB_REGISTER_CLASS` 用于创建新的 TJS2 类。scriptsEx 使用的是另一个宏 `NCB_ATTACH_CLASS`——它不创建新类，而是向**已存在**的 TJS2 类追加方法：

| 特性 | NCB_REGISTER_CLASS | NCB_ATTACH_CLASS |
|------|-------------------|-----------------|
| 功能 | 创建全新的 TJS2 类 | 向已有类追加成员 |
| 语法 | `NCB_REGISTER_CLASS(CppClass)` | `NCB_ATTACH_CLASS(CppClass, TJSClassName)` |
| 实例化 | 支持（`new TJSClassName()`） | 不支持（无构造函数） |
| 典型场景 | 创建 CSVParser、LayerEx 等新类 | 向 Scripts、Window 追加工具方法 |
| C++ 类要求 | 需要构造/析构对应 TJS 生命周期 | 通常只包含 static 方法 |

### 4.2 scriptsEx 的注册代码详解

scriptsEx 的注册代码位于文件末尾（第 968-993 行）：

```cpp
// 源码：cpp/plugins/scriptsEx.cpp 第 968-993 行
NCB_ATTACH_CLASS(ScriptsAdd, Scripts) {
    // === RawCallback 注册 ===
    // 参数1：TJS2 中的方法名
    // 参数2：C++ 函数指针
    // 参数3：TJS_STATICMEMBER 表示注册为静态方法
    RawCallback(TJS_W("getObjectKeys"),
                &ScriptsAdd::getKeys, TJS_STATICMEMBER);
    RawCallback(TJS_W("getObjectCount"),
                &ScriptsAdd::getCount, TJS_STATICMEMBER);

    // === NCB_METHOD 注册 ===
    // ncbind 自动推导参数类型并生成包装器
    NCB_METHOD(getObjectContext);
    NCB_METHOD(isNullContext);
    NCB_METHOD(equalStruct);
    NCB_METHOD(equalStructNumericLoose);

    // === 更多 RawCallback ===
    RawCallback(TJS_W("foreach"),
                &ScriptsAdd::foreach, TJS_STATICMEMBER);
    RawCallback(TJS_W("getMD5HashString"),
                &ScriptsAdd::getMD5HashString, TJS_STATICMEMBER);

    NCB_METHOD(clone);

    RawCallback("propSet", &ScriptsAdd::propSet, TJS_STATICMEMBER);
    RawCallback("propGet", &ScriptsAdd::propGet, TJS_STATICMEMBER);

    // === Variant 常量注册 ===
    // 将 C++ 宏常量注册为 TJS2 中 Scripts 对象的属性
    Variant(TJS_W("pfMemberEnsure"), TJS_MEMBERENSURE);
    Variant(TJS_W("pfMemberMustExist"), TJS_MEMBERMUSTEXIST);
    Variant(TJS_W("pfIgnoreProp"), TJS_IGNOREPROP);
    Variant(TJS_W("pfHiddenMember"), TJS_HIDDENMEMBER);
    Variant(TJS_W("pfStaticMember"), TJS_STATICMEMBER);

    RawCallback(TJS_W("safeEvalStorage"),
                &ScriptsAdd::safeEvalStorage, TJS_STATICMEMBER);
};

// 独立函数附加——将全局函数 TJSDoRehash 作为 Scripts.rehash 注册
NCB_ATTACH_FUNCTION(rehash, Scripts, TJSDoRehash);
```

### 4.3 三种注册方式的选择依据

从上面的代码可以看出，scriptsEx 混用了三种注册方式。选择依据如下：

| 注册方式 | 使用条件 | 示例 |
|---------|---------|------|
| **NCB_METHOD** | 函数签名符合 ncbind 自动转换规则（参数/返回值都是 ncbind 支持的类型） | `getObjectContext(tTJSVariant)` → 返回 `tTJSVariant`，ncbind 可以自动处理 |
| **RawCallback** | 函数需要直接访问原始 TJS2 参数（`tTJSVariant**`、参数个数 `numparams`、`objthis`） | `getKeys` 需要检查 `numparams`、直接操作 `tTJSVariant*`，ncbind 的自动转换不够灵活 |
| **Variant** | 注册常量值（整数、字符串等） | `pfMemberEnsure` = `TJS_MEMBERENSURE`（整数常量） |

关键判断规则：**如果函数的参数个数是可变的，或者需要直接操作 `tTJSVariant` 指针（而非自动转换后的 C++ 类型），就必须使用 RawCallback**。例如 `getKeys` 需要校验 `numparams < 1` 然后手动取参数，这超出了 `NCB_METHOD` 的自动转换能力。

---

## 五、csvParser 插件概览

### 5.1 csvParser 是什么

`csvParser`（CSV Parser，CSV 解析器）是一个用于解析 CSV（Comma-Separated Values，逗号分隔值）格式文本的插件。CSV 是一种简单的表格数据格式，每行代表一条记录，字段之间用逗号分隔。在视觉小说游戏中，CSV 常被用来存储角色数据表、对话选项列表、配置参数等结构化数据。

csvParser 的源码位于：

```
cpp/plugins/csvParser.cpp    （466 行，单文件插件）
```

与 scriptsEx 不同，csvParser **创建了一个全新的 TJS2 类** `CSVParser`，并且使用了**完全不同的注册方式**——手动调用 KiriKiri 原始 TJS API（`TJSCreateNativeClassForPlugin` + `TJS_BEGIN_NATIVE_MEMBERS` 宏），再通过 `NCB_PRE_REGIST_CALLBACK` 搭上 ncbind 的自动初始化时机。

### 5.2 整体架构

```
┌─────────────────────────────────────────────────────────┐
│              TJS2 脚本层                                   │
│  var parser = new CSVParser(callback);                   │
│  parser.initStorage("data.csv");                         │
│  parser.parse();                                         │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│     TJS 原生类层（TJS_BEGIN_NATIVE_MEMBERS 宏生成）        │
│  Create_NC_CSVParser() → TJSCreateNativeClassForPlugin   │
│  ├─ TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL(CSVParser)         │
│  ├─ TJS_BEGIN_NATIVE_METHOD_DECL(init)                   │
│  ├─ TJS_BEGIN_NATIVE_METHOD_DECL(initStorage)            │
│  ├─ TJS_BEGIN_NATIVE_METHOD_DECL(getNextLine)            │
│  ├─ TJS_BEGIN_NATIVE_METHOD_DECL(parse)                  │
│  ├─ TJS_BEGIN_NATIVE_METHOD_DECL(parseStorage)           │
│  └─ TJS_BEGIN_NATIVE_PROP_DECL(currentLineNumber)        │
└──────────────────────────┬──────────────────────────────┘
                           │ TJS_GET_NATIVE_INSTANCE
┌──────────────────────────▼──────────────────────────────┐
│     NI_CSVParser : public tTJSNativeInstance              │
│  ├─ Construct() / Invalidate()  — TJS 生命周期           │
│  ├─ init() / initStorage()      — 数据源初始化           │
│  ├─ getNextLine()               — 逐行读取 + 解析       │
│  ├─ parse()                     — 批量解析（回调模式）   │
│  └─ split()                     — CSV 字段分割核心算法   │
└──────────────────────────┬──────────────────────────────┘
                           │ 使用
┌──────────────────────────▼──────────────────────────────┐
│     IFile 接口层                                          │
│  IFile (抽象基类) ← IFileStr (字符串数据源)               │
│  提供 addNextLine() 逐行读取接口                          │
└─────────────────────────────────────────────────────────┘
```

### 5.3 IFile 抽象接口

csvParser 定义了一个简单的文件抽象接口 `IFile`（第 15-19 行）：

```cpp
// 源码：cpp/plugins/csvParser.cpp 第 15-19 行
class IFile {
public:
    virtual ~IFile() {};
    // 从数据源读取下一行，追加到 str 尾部
    // 返回 true 表示成功读取，false 表示到达末尾
    virtual bool addNextLine(ttstr &str) = 0;
};
```

目前只有一个实现 `IFileStr`（第 21-69 行），它封装了一个 `ttstr` 字符串作为数据源，模拟逐字符读取：

```cpp
// 源码：cpp/plugins/csvParser.cpp 第 21-69 行
class IFileStr : public IFile {
    ttstr dat;       // 原始文本数据
    uint32_t pos;    // 当前读取位置

public:
    IFileStr(const ttstr &str) { dat = str; pos = 0; }

    int getc() {
        // 逐字符读取，到末尾返回 EOF
        return pos < (uint32_t)dat.length() ? dat[pos++] : EOF;
    }

    void ungetc() {    // 回退一个字符
        if (pos > 0) { pos--; }
    }

    bool eof() { return pos >= (uint32_t)dat.length(); }

    // 换行检测：处理 \r、\n、\r\n 三种换行格式
    bool endOfLine(tjs_char c) {
        bool eol = (c == '\r' || c == '\n');
        if (c == '\r') {
            c = getc();
            // \r\n 组合只算一次换行
            if (!eof() && c != '\n') { ungetc(); }
        }
        return eol;
    }

    bool addNextLine(ttstr &str) {
        int l = 0;
        int c;
        // 读取字符直到换行或 EOF
        while ((c = getc()) != EOF && !endOfLine(c)) {
            str += c;
            l++;
        }
        // 只要读到了字符或者还没到 EOF，就算成功
        return (l > 0 || c != EOF);
    }
};
```

这个设计预留了扩展空间——如果未来需要支持从文件流直接读取（而非先加载整个文件到内存），只需新增一个 `IFileStream` 实现即可。

### 5.4 NI_CSVParser 核心实现

`NI_CSVParser`（第 125-337 行）继承自 `tTJSNativeInstance`，这是手动创建 TJS 类时 C++ 侧实例必须继承的基类。它提供了两个关键虚函数：

- **`Construct(numparams, param, tjs_obj)`**——TJS2 的 `new CSVParser(...)` 时调用，对应 TJS 构造函数
- **`Invalidate()`**——TJS2 对象被垃圾回收或显式 invalidate 时调用，对应析构

```cpp
// 源码：cpp/plugins/csvParser.cpp 第 222-255 行
tjs_error NI_CSVParser::Construct(tjs_int numparams, tTJSVariant **param,
                                  iTJSDispatch2 *tjs_obj) {
    if (numparams > 0) {
        target = param[0]->AsObject();    // 第1个参数：回调目标对象
        if (numparams > 1) {
            separator = (tjs_int)*param[1]; // 第2个参数：分隔符（默认','）
            if (numparams > 2) {
                newline = *param[2];        // 第3个参数：换行符（默认"\r\n"）
            }
        }
    }
    return TJS_S_OK;
}

void NI_CSVParser::Invalidate() {
    clear();             // 关闭文件
    if (target) {
        target->Release(); // 释放回调目标的引用
        target = nullptr;
    }
}
```

CSV 分割的核心算法在 `split` 方法（第 155-200 行）中。它实现了完整的 RFC 4180 兼容 CSV 解析：支持双引号包裹的字段（内部可包含逗号和换行）、双引号转义（`""` 表示一个 `"`）、跨行字段等。

## 六、csvParser 的手动 TJS 类工厂

与 scriptsEx 使用 ncbind 宏自动注册不同，csvParser 采用**手动 TJS 原生类注册**（Manual TJS Native Class Registration）模式——这是 KiriKiri 原版 SDK 提供的底层注册方式，开发者需要手写每个方法/属性的包装函数，通过一组 `TJS_BEGIN_NATIVE_*` 宏把 C++ 类"翻译"给 TJS2 引擎。这种方式比 ncbind 繁琐，但对注册流程有完全控制权。

### 6.1 ClassID 手动设置

在使用 TJS 宏之前，必须为类配置一个全局唯一的 **ClassID**（类标识符，TJS2 引擎用来区分不同原生类的整数标识）。csvParser 用 `#undef` / `#define` 重定义宏来实现：

```cpp
// 源码：cpp/plugins/csvParser.cpp 第 112-120 行
#undef TJS_NATIVE_CLASSID_NAME
// TJS_NATIVE_CLASSID_NAME 宏定义了一个静态 tjs_int32 变量名，
// TJS 引擎通过这个变量存储/读取类的唯一 ID
#define TJS_NATIVE_CLASSID_NAME CSVParser_classid

// 声明全局静态 ClassID 变量
static tjs_int32 CSVParser_classid = -1;
// -1 表示"尚未分配"，TJS 引擎在第一次注册时会写入真实 ID
```

这段代码做了三件事：
1. **取消旧定义**：`#undef TJS_NATIVE_CLASSID_NAME` 移除可能的残留定义
2. **绑定新名字**：`#define TJS_NATIVE_CLASSID_NAME CSVParser_classid` 告诉后续所有 TJS 宏"本类的 ClassID 存在 `CSVParser_classid` 这个变量里"
3. **声明并初始化**：`static tjs_int32 CSVParser_classid = -1` 提供实际存储

> **为什么 ncbind 不需要这一步？** 因为 `NCB_REGISTER_CLASS` / `NCB_ATTACH_CLASS` 宏内部自动生成了 ClassID 变量。手动模式把这个自动化步骤暴露给了开发者。

### 6.2 Create_NC_CSVParser —— 类工厂函数

`Create_NC_CSVParser()` 是整个手动注册的核心，它创建一个 `tTJSNativeClassForPlugin`（TJS 原生类容器，专为插件设计的类对象模板）实例，然后逐一注册构造函数、方法、属性：

```cpp
// 源码：cpp/plugins/csvParser.cpp 第 341-343 行
static iTJSDispatch2 *Create_NC_CSVParser() {
    // 创建类对象：参数1=TJS类名, 参数2=NativeInstance工厂函数
    tTJSNativeClassForPlugin *classobj =
        TJSCreateNativeClassForPlugin(
            TJS_W("CSVParser"),       // TJS2 脚本中可见的类名
            Create_NI_CSVParser       // 返回 new NI_CSVParser() 的工厂
        );
```

`TJSCreateNativeClassForPlugin` 是 KiriKiri SDK 提供的工厂函数（Factory Function，一种创建对象的设计模式），返回一个 `tTJSNativeClassForPlugin` 指针，后续通过 `TJS_BEGIN_NATIVE_MEMBERS` 宏向其添加成员。

### 6.3 TJS_BEGIN_NATIVE_MEMBERS —— 成员注册块

`TJS_BEGIN_NATIVE_MEMBERS(CSVParser)` 宏展开后生成一段初始化代码，设置当前注册上下文为 `classobj`。之后在这个块内，每个 `TJS_BEGIN_NATIVE_METHOD_DECL` / `TJS_BEGIN_NATIVE_PROP_DECL` 都会向 `classobj` 添加成员。块最终以 `TJS_END_NATIVE_MEMBERS` 结束。

**析构函数（Finalize Method）：**

```cpp
// 源码：cpp/plugins/csvParser.cpp 第 347 行
TJS_DECL_EMPTY_FINALIZE_METHOD
// 展开为：注册一个名为"finalize"的空方法
// TJS2 在销毁对象时调用 finalize，csvParser 不需要脚本侧清理
// 实际的 C++ 资源释放在 NI_CSVParser::Invalidate() 中完成
```

> **finalize vs Invalidate**：TJS2 对象销毁分两步。首先调用脚本侧 `finalize`（可被 TJS 脚本重写），然后调用 C++ 侧 `Invalidate()`（释放原生资源）。csvParser 的脚本侧不需要做任何事，所以用空 finalize。

**构造函数包装：**

```cpp
// 源码：cpp/plugins/csvParser.cpp 第 349-355 行
TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL(
    /*var.name*/ _this,          // C++ 局部变量名，指向 NI_CSVParser 实例
    /*var.type*/ NI_CSVParser,   // NativeInstance 的 C++ 类型
    /*TJS class name*/ CSVParser // TJS 类名（用于错误消息）
) {
    return TJS_S_OK;
    // 构造函数体为空——实际初始化在 NI_CSVParser::Construct() 中
    // TJS 引擎会先调用这里，再调用 Construct(numparams, param, tjs_obj)
}
TJS_END_NATIVE_CONSTRUCTOR_DECL(/*TJS class name*/ CSVParser)
```

这个宏展开后生成一个注册到 `classobj` 的"构造函数"回调。当 TJS 脚本执行 `new CSVParser(target, ',')` 时，引擎内部流程为：

```
TJS 脚本: new CSVParser(target, ',')
  → 引擎调用 Create_NI_CSVParser() 创建 C++ 实例
  → 引擎调用构造函数包装（这里 return TJS_S_OK）
  → 引擎调用 NI_CSVParser::Construct(numparams, param, tjs_obj)
  → C++ 实例保存 target 引用、设置 separator
```

### 6.4 方法注册 —— TJS_BEGIN_NATIVE_METHOD_DECL

csvParser 注册了 5 个方法。以 `init` 为例：

```cpp
// 源码：cpp/plugins/csvParser.cpp 第 357-365 行
TJS_BEGIN_NATIVE_METHOD_DECL(/*func. name*/ init) {
    // TJS_GET_NATIVE_INSTANCE 从 TJS 对象中提取绑定的 C++ 实例
    // 参数：局部变量名、C++ 类型
    // 如果对象没有绑定的 NI_CSVParser 实例，会抛出异常
    TJS_GET_NATIVE_INSTANCE(/*var. name*/ _this,
                            /*var. type*/ NI_CSVParser);

    if (numparams < 1)
        return TJS_E_BADPARAMCOUNT; // 参数不足时返回错误码

    // 调用 C++ 实例的 init 方法，传入第一个参数作为字符串
    _this->init(param[0]->AsStringNoAddRef());
    return TJS_S_OK;               // 成功
}
TJS_END_NATIVE_METHOD_DECL(/*func. name*/ init)
```

**宏展开后的实际签名**：每个 `TJS_BEGIN_NATIVE_METHOD_DECL(name)` 展开为一个符合 TJS2 回调签名的静态函数，其隐含参数为：

| 隐含参数 | 类型 | 含义 |
|----------|------|------|
| `result` | `tTJSVariant *` | 返回值写入位置（可为 nullptr 表示调用者不关心返回值） |
| `numparams` | `tjs_int` | 实参个数 |
| `param` | `tTJSVariant **` | 实参数组指针 |
| `objthis` | `iTJSDispatch2 *` | 调用该方法的 TJS 对象（即 `this`） |

所有 5 个方法的注册模式完全一致，区别仅在于调用的 C++ 方法不同：

| TJS 方法名 | 对应 C++ 调用 | 参数要求 | 功能 |
|-----------|--------------|---------|------|
| `init` | `_this->init(str)` | 1 个字符串 | 从字符串初始化 CSV 数据 |
| `initStorage` | `_this->initStorage(str, flag)` | 1-2 个参数 | 从文件路径加载 CSV（flag 控制是否跳过 BOM） |
| `getNextLine` | `_this->getNextLine(result)` | 无 | 读取下一行，结果写入 `result` |
| `parse` | `_this->parse(objthis)` | 0-1 个参数 | 解析全部行，回调到 TJS 对象 |
| `parseStorage` | `_this->parse(objthis)` | 0-1 个参数 | 从文件加载并解析全部行 |

注意 `parse` 和 `parseStorage` 的巧妙设计：如果传入参数，会先调用 `init` / `initStorage` 设置数据源，然后再调用 `parse` 执行解析——一个方法兼顾了"先初始化再解析"和"直接解析已初始化数据"两种用法。

### 6.5 属性注册 —— TJS_BEGIN_NATIVE_PROP_DECL

csvParser 注册了一个只读属性 `currentLineNumber`：

```cpp
// 源码：cpp/plugins/csvParser.cpp 第 409-418 行
TJS_BEGIN_NATIVE_PROP_DECL(currentLineNumber) {
    // Getter：读取属性值
    TJS_BEGIN_NATIVE_PROP_GETTER {
        TJS_GET_NATIVE_INSTANCE(
            /*var. name*/ _this,
            /*var. type*/ NI_CSVParser);
        *result = _this->getLineNumber(); // 将当前行号写入返回值
        return TJS_S_OK;
    }
    TJS_END_NATIVE_PROP_GETTER

    // 拒绝写入——这是只读属性
    TJS_DENY_NATIVE_PROP_SETTER
}
TJS_END_NATIVE_PROP_DECL(currentLineNumber)
```

TJS2 属性注册的三种模式：

| 模式 | Getter 宏 | Setter 宏 | 效果 |
|------|-----------|-----------|------|
| 只读 | `TJS_BEGIN_NATIVE_PROP_GETTER` | `TJS_DENY_NATIVE_PROP_SETTER` | 读取正常，写入抛异常 |
| 读写 | `TJS_BEGIN_NATIVE_PROP_GETTER` | `TJS_BEGIN_NATIVE_PROP_SETTER` | 读写均正常 |
| 只写 | `TJS_DENY_NATIVE_PROP_GETTER` | `TJS_BEGIN_NATIVE_PROP_SETTER` | 读取抛异常，写入正常（罕见） |

最后 `TJS_END_NATIVE_MEMBERS` 关闭成员注册块，函数返回 `classobj`：

```cpp
// 源码：cpp/plugins/csvParser.cpp 第 420-428 行
    TJS_END_NATIVE_MEMBERS
    // 成员注册完毕，classobj 现在包含：
    // - 构造函数 + finalize
    // - 5 个方法（init, initStorage, getNextLine, parse, parseStorage）
    // - 1 个只读属性（currentLineNumber）

    return classobj; // 返回完整的 TJS 类对象
}
```

### 6.6 手动注册模式总结

整个 `Create_NC_CSVParser()` 函数的执行流程可以用这张图概括：

```
Create_NC_CSVParser()
  │
  ├─ TJSCreateNativeClassForPlugin("CSVParser", factory)
  │    → 创建空的 TJS 类容器 classobj
  │
  ├─ TJS_BEGIN_NATIVE_MEMBERS(CSVParser)
  │    → 设置注册上下文
  │
  ├─ TJS_DECL_EMPTY_FINALIZE_METHOD
  │    → 注册空的 finalize 方法
  │
  ├─ TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL
  │    → 注册构造函数包装（委托给 Construct）
  │
  ├─ TJS_BEGIN_NATIVE_METHOD_DECL × 5
  │    → 注册 init / initStorage / getNextLine / parse / parseStorage
  │
  ├─ TJS_BEGIN_NATIVE_PROP_DECL × 1
  │    → 注册只读属性 currentLineNumber
  │
  ├─ TJS_END_NATIVE_MEMBERS
  │    → 关闭注册块
  │
  └─ return classobj
       → 返回可注册到全局的类对象
```

## 七、混合注册模式 —— ncbind 回调 + 手动 TJS 宏

csvParser 最巧妙的设计在文件末尾。它用手动 TJS 宏创建类，但用 ncbind 的回调机制触发注册——两种风格的"混血"：

### 7.1 InitPlugin_CSVParser —— 手动全局注册

```cpp
// 源码：cpp/plugins/csvParser.cpp 第 430-448 行
void InitPlugin_CSVParser() {
    // 获取 TJS2 全局对象（相当于 JavaScript 的 window/globalThis）
    iTJSDispatch2 *global = TVPGetScriptDispatch();

    if (global) {
        // 获取 Array 类的 clear 方法引用
        // csvParser 在解析时需要清空数组，提前缓存避免重复查找
        {
            tTJSVariant varScripts;
            TVPExecuteExpression(TJS_W("Array"), &varScripts);
            iTJSDispatch2 *dispatch = varScripts.AsObjectNoAddRef();
            ArrayClearMethod = getMember(dispatch, TJS_W("clear"));
        }

        // 将 CSVParser 类注册到全局作用域
        // Create_NC_CSVParser() 返回完整的类对象
        addMember(global, TJS_W("CSVParser"), Create_NC_CSVParser());

        global->Release(); // 释放全局对象引用
    }
}
```

这个函数做了两件事：
1. **缓存 Array.clear**：解析 CSV 时需要反复清空数组行缓冲，预先获取 `clear` 方法的 `iTJSDispatch2` 指针，避免每次解析都做 `getMember` 查找
2. **注册 CSVParser 到全局**：调用 `addMember` 把 `Create_NC_CSVParser()` 返回的类对象挂到 TJS2 全局命名空间

### 7.2 NCB_PRE_REGIST_CALLBACK —— 混合桥接

```cpp
// 源码：cpp/plugins/csvParser.cpp 第 466 行
NCB_PRE_REGIST_CALLBACK(InitPlugin_CSVParser);
```

这一行是整个混合模式的关键。`NCB_PRE_REGIST_CALLBACK` 是 ncbind 框架提供的宏，它把 `InitPlugin_CSVParser` 注册为"ncbind 注册阶段之前的回调"（Pre-Registration Callback）。当引擎启动时，ncbind 的 `AutoRegisterFactory` 机制（详见 [02-类注册与宏体系](../../02-ncbind绑定框架/02-类注册与宏体系.md)）会按顺序：

1. 执行所有 `NCB_PRE_REGIST_CALLBACK` 注册的回调
2. 执行所有 `NCB_REGISTER_CLASS` / `NCB_ATTACH_CLASS` 的自动注册

这样 csvParser 就"搭便车"利用了 ncbind 的自动注册时机，而类本身的创建仍然用手动 TJS 宏。

### 7.3 三种注册模式对比

KrKr2 项目中存在三种插件注册风格，它们的复杂度和灵活性不同：

| 对比维度 | ncbind 自动注册 | 手动 TJS 宏 | 混合模式（csvParser） |
|----------|----------------|------------|---------------------|
| **代表插件** | scriptsEx, dirlist, saveStruct | 纯原版 KiriKiri 插件 | csvParser |
| **类创建** | `NCB_REGISTER_CLASS` / `NCB_ATTACH_CLASS` 自动 | `TJS_BEGIN_NATIVE_MEMBERS` 手动 | `TJS_BEGIN_NATIVE_MEMBERS` 手动 |
| **注册时机** | `AutoRegisterFactory` 自动触发 | 需手动调用注册函数 | `NCB_PRE_REGIST_CALLBACK` 触发 |
| **ClassID** | 宏自动生成 | 手动 `#undef`/`#define` | 手动 `#undef`/`#define` |
| **类型转换** | 自动（ncbind 类型系统） | 手动（`param[i]->AsStringNoAddRef()` 等） | 手动 |
| **代码量** | 最少（1-3 行宏） | 最多（每个方法 5-10 行） | 中等（类手动 + 注册自动） |
| **灵活性** | 受限于 ncbind 支持的签名 | 完全控制 | 完全控制 |
| **适用场景** | C++ 方法签名规则、参数类型简单 | 复杂参数处理、遗留代码移植 | 遗留代码 + 需要自动注册时机 |

> **选择建议**：新写插件优先用 ncbind 自动注册（代码最少、维护最简单）。只有当方法签名不兼容 ncbind 的类型转换系统（如需要直接操作 `tTJSVariant **param`）或需要在注册阶段做额外初始化（如缓存 Array.clear）时，才使用手动或混合模式。

## 八、动手实践

### 练习 1：用 TJS 宏给 csvParser 添加一个只读属性

假设我们要为 `CSVParser` 添加一个 `separator` 属性，让 TJS 脚本可以读取当前使用的分隔符：

```cpp
// 在 TJS_BEGIN_NATIVE_MEMBERS(CSVParser) 块内，
// TJS_END_NATIVE_MEMBERS 之前添加：

TJS_BEGIN_NATIVE_PROP_DECL(separator) {
    TJS_BEGIN_NATIVE_PROP_GETTER {
        TJS_GET_NATIVE_INSTANCE(
            /*var. name*/ _this,
            /*var. type*/ NI_CSVParser);
        // NI_CSVParser::separator 是 tjs_char 类型（单个字符）
        // 将其转为 tjs_int 返回（字符的 Unicode 码点）
        *result = (tjs_int)_this->separator;
        return TJS_S_OK;
    }
    TJS_END_NATIVE_PROP_GETTER

    TJS_DENY_NATIVE_PROP_SETTER // 只读
}
TJS_END_NATIVE_PROP_DECL(separator)
```

TJS 脚本使用方式：

```javascript
// TJS2 脚本
var parser = new CSVParser(this);
System.inform("分隔符码点: " + parser.separator);
// 默认输出: 分隔符码点: 44  （逗号的 Unicode 码点）
```

### 练习 2：用 ncbind 实现等价的方法注册

对比手动 TJS 宏和 ncbind 的代码量差异。假设 `NI_CSVParser` 的 `init` 方法签名改为接受 `const ttstr &`，ncbind 版本只需要：

```cpp
// ncbind 方式（假设签名兼容）
NCB_REGISTER_CLASS(CSVParser) {
    NCB_CONSTRUCTOR((iTJSDispatch2 *, tjs_char, ttstr));
    NCB_METHOD(init);           // 1 行 = 手动模式的 8 行
    NCB_METHOD(initStorage);
    NCB_METHOD(getNextLine);
    NCB_METHOD(parse);
    NCB_METHOD(parseStorage);
    NCB_PROPERTY_RO(currentLineNumber, getLineNumber);
}
// 总计 ~10 行 vs 手动模式的 ~90 行
```

实际上 csvParser 不能直接用 ncbind，原因有两个：
1. 方法参数需要直接操作 `tTJSVariant **param`（例如 `initStorage` 的第二个参数是可选的，且需要转为 bool）
2. `InitPlugin_CSVParser` 需要在注册时缓存 `Array.clear`，ncbind 的标准注册流程没有提供这个钩子

## 对照项目源码

本节所有代码均来自 KrKr2 项目实际源码，以下是完整索引：

| 源码文件 | 行号范围 | 本节对应章节 | 关键内容 |
|----------|---------|-------------|---------|
| `cpp/plugins/scriptsEx.cpp` | 1-178 | 二、Try_ 安全包装 | TVPDoTryBlock 机制、PropGet/PropSet/FuncCall 等 8 个包装函数 |
| `cpp/plugins/scriptsEx.cpp` | 183-252 | 三、ScriptsAdd 扩展方法 | 类声明、12+ 静态方法、4 个回调辅助类 |
| `cpp/plugins/scriptsEx.cpp` | 257-432 | 三、ScriptsAdd 扩展方法 | DictMemberGetCaller、CompareHelper、IterateHelper、CloneHelper |
| `cpp/plugins/scriptsEx.cpp` | 438-966 | 三、ScriptsAdd 扩展方法 | getKeys/getCount/equalStruct/clone/foreach 方法实现 |
| `cpp/plugins/scriptsEx.cpp` | 968-993 | 四、scriptsEx 注册分析 | NCB_ATTACH_CLASS + NCB_ATTACH_FUNCTION 注册 |
| `cpp/plugins/csvParser.cpp` | 15-69 | 五、csvParser 架构 | IFile/IFileStr 抽象接口 |
| `cpp/plugins/csvParser.cpp` | 73-120 | 五、csvParser 架构 | 辅助函数 + ClassID 手动设置 |
| `cpp/plugins/csvParser.cpp` | 125-337 | 五、csvParser 架构 | NI_CSVParser 完整实现 |
| `cpp/plugins/csvParser.cpp` | 339-428 | 六、手动 TJS 类工厂 | Create_NC_CSVParser() 宏注册 |
| `cpp/plugins/csvParser.cpp` | 430-466 | 七、混合注册模式 | InitPlugin_CSVParser + NCB_PRE_REGIST_CALLBACK |
| `cpp/core/plugin/ncbind.hpp` | 全文 | 四、注册分析（对比） | ncbind 宏定义、AutoRegisterFactory |

## 常见错误与排查

### 错误 1：TJS_GET_NATIVE_INSTANCE 崩溃

**症状**：在手动注册的方法中调用 `TJS_GET_NATIVE_INSTANCE` 时，程序崩溃或返回 `TJS_E_NATIVECLASSCRASH`。

**原因**：`TJS_NATIVE_CLASSID_NAME` 宏没有正确定义，导致 ClassID 不匹配。

**解决方案**：确保在 `TJS_BEGIN_NATIVE_MEMBERS` 之前执行了 `#undef TJS_NATIVE_CLASSID_NAME` 和 `#define TJS_NATIVE_CLASSID_NAME YourClassId`，并声明了 `static tjs_int32 YourClassId = -1`：

```cpp
// 正确写法
#undef TJS_NATIVE_CLASSID_NAME
#define TJS_NATIVE_CLASSID_NAME MyPlugin_classid
static tjs_int32 MyPlugin_classid = -1;

// 然后才能使用 TJS_BEGIN_NATIVE_MEMBERS
```

### 错误 2：NCB_ATTACH_CLASS 目标类不存在

**症状**：使用 `NCB_ATTACH_CLASS(MyClass, TargetClass)` 时，TJS2 脚本调用方法报"member not found"。

**原因**：`TargetClass` 在 ncbind 注册阶段尚未被 TJS2 引擎创建（例如 `Scripts` 类是引擎内置类，在 TJS2 初始化时创建——如果你的插件在 TJS2 初始化之前注册就会失败）。

**解决方案**：确认目标类是 TJS2 内置类（`Scripts`、`System`、`Array` 等），它们在引擎启动早期就存在。如果目标是其他插件创建的类，使用 `NCB_PRE_REGIST_CALLBACK` 控制注册顺序。

### 错误 3：手动注册的方法参数数量检查遗漏

**症状**：TJS 脚本少传参数时，C++ 侧访问 `param[i]` 越界导致内存错误。

**原因**：手动 `TJS_BEGIN_NATIVE_METHOD_DECL` 不会自动检查参数个数，需要开发者手动校验。

**解决方案**：每个方法开头必须检查 `numparams`：

```cpp
TJS_BEGIN_NATIVE_METHOD_DECL(myMethod) {
    TJS_GET_NATIVE_INSTANCE(_this, NI_MyPlugin);
    // 必须检查！否则 param[0] 可能越界
    if (numparams < 2)
        return TJS_E_BADPARAMCOUNT;
    // 安全地使用 param[0] 和 param[1]
    _this->doSomething(
        param[0]->AsStringNoAddRef(),
        (tjs_int)*param[1]
    );
    return TJS_S_OK;
}
TJS_END_NATIVE_METHOD_DECL(myMethod)
```

## 本节小结

- **scriptsEx** 通过 `NCB_ATTACH_CLASS(ScriptsAdd, Scripts)` 将 12+ 个扩展方法挂载到 TJS2 内置 `Scripts` 类上，使用 ncbind 自动注册，混合了 `RawCallback`（灵活签名）、`NCB_METHOD`（自动类型转换）和 `Variant`（常量）三种注册方式
- **Try_ 安全包装**是 scriptsEx 的核心设计模式：用 `TVPDoTryBlock` + `_CatchFuncCall` 将 TJS2 的 longjmp 异常桥接到 C++ 异常，保护 `iTJSDispatch2` 的每个方法调用
- **ScriptsAdd 的实用方法**（`getKeys`、`equalStruct`、`clone`、`foreach`）大量使用 `EnumMembers` + 回调辅助类模式，这是操作 TJS2 字典/数组的惯用手法
- **csvParser** 采用手动 TJS 宏注册，通过 `TJS_BEGIN_NATIVE_MEMBERS` 逐一注册构造函数、5 个方法和 1 个只读属性，对每个参数做手动类型转换和数量检查
- **混合注册模式**：csvParser 用 `NCB_PRE_REGIST_CALLBACK` "搭便车"利用 ncbind 的自动注册时机，同时保持手动 TJS 宏对注册细节的完全控制
- **ClassID 手动设置**需要 `#undef TJS_NATIVE_CLASSID_NAME` + `#define` + `static tjs_int32` 三步，而 ncbind 宏自动完成这些
- 新写插件优先使用 ncbind 自动注册（代码量最少），仅在需要直接操作 `tTJSVariant **param` 或需要注册时副作用时使用手动/混合模式

## 练习题与答案

### 题目 1：scriptsEx 的 Try_ 包装模式

`Try_iTJSDispatch2_PropGet` 的三步流程是什么？如果去掉 `TVPDoTryBlock` 包装，直接调用 `dispatch->PropGet(0, name, nullptr, result, dispatch)`，会有什么后果？

<details>
<summary>查看答案</summary>

三步流程：
1. **构造参数包**：创建 `TryFuncCallParam` 结构体，存储 `dispatch`、`name`、`result` 等参数
2. **安全执行**：调用 `TVPDoTryBlock(&_PropGet, &_CatchFuncCall, &params)`，其中 `_PropGet` 是实际调用 `PropGet` 的函数指针
3. **异常桥接**：如果 `PropGet` 在 TJS2 虚拟机内部抛出异常（longjmp），`TVPDoTryBlock` 捕获后调用 `_CatchFuncCall`，将异常描述转为 C++ `throw`

如果去掉 `TVPDoTryBlock` 直接调用：
- TJS2 的异常机制基于 `setjmp`/`longjmp`，会直接跳过 C++ 的栈展开（Stack Unwinding）
- 任何 RAII 对象（智能指针、`std::string`、`std::vector` 等）的析构函数都不会被调用
- 导致内存泄漏、资源未释放、程序状态不一致
- 在某些平台上（特别是 Windows SEH）可能直接导致程序崩溃

</details>

### 题目 2：csvParser 的手动注册 vs ncbind 对比

csvParser 为什么不能直接用 `NCB_REGISTER_CLASS` 注册？如果强行用 ncbind，需要对 `NI_CSVParser` 做哪些修改才能兼容？

<details>
<summary>查看答案</summary>

csvParser 不能直接用 ncbind 的两个原因：

**原因 1：方法签名不兼容**
csvParser 的方法（如 `initStorage`）需要直接操作 `tTJSVariant **param` 来处理可选参数：
```cpp
// initStorage 的第二个参数是可选的 bool
_this->initStorage(param[0]->AsStringNoAddRef(),
                   numparams > 1 && (tjs_int)*param[1] != 0);
```
ncbind 的 `NCB_METHOD` 要求方法参数与 TJS 参数一一对应，不支持"可选参数用 numparams 判断"的模式。

**原因 2：注册时副作用**
`InitPlugin_CSVParser` 在注册 CSVParser 类的同时，还需要获取并缓存 `Array.clear` 方法的引用：
```cpp
ArrayClearMethod = getMember(dispatch, TJS_W("clear"));
```
ncbind 的 `NCB_REGISTER_CLASS` 没有提供"注册时执行额外代码"的钩子。

**修改方案**（使之兼容 ncbind）：
1. 将 `init(const tjs_char *)` 改为 `init(const ttstr &)`
2. 将 `initStorage` 拆为两个重载：`initStorage(const ttstr &)` 和 `initStorage(const ttstr &, bool)`
3. 用 `NCB_PRE_REGIST_CALLBACK` 执行 `Array.clear` 缓存，与 `NCB_REGISTER_CLASS` 组合使用
4. `parse` 和 `parseStorage` 需要用 `RawCallback` 注册（因为需要 `objthis` 参数）

修改后的代码：
```cpp
// 保留 PRE_REGIST_CALLBACK 用于缓存 Array.clear
void InitCSVParserDeps() {
    tTJSVariant varScripts;
    TVPExecuteExpression(TJS_W("Array"), &varScripts);
    iTJSDispatch2 *dispatch = varScripts.AsObjectNoAddRef();
    ArrayClearMethod = getMember(dispatch, TJS_W("clear"));
}
NCB_PRE_REGIST_CALLBACK(InitCSVParserDeps);

// ncbind 注册（parse/parseStorage 仍需 RawCallback）
NCB_REGISTER_CLASS(CSVParser) {
    NCB_CONSTRUCTOR((iTJSDispatch2 *));
    NCB_METHOD(init);
    NCB_METHOD(initStorage);  // 需要重载版本
    NCB_METHOD(getNextLine);
    RawCallback("parse", &CSVParser_parse_raw, 0);
    RawCallback("parseStorage", &CSVParser_parseStorage_raw, 0);
    NCB_PROPERTY_RO(currentLineNumber, getLineNumber);
}
```

</details>

### 题目 3：EnumMembers 回调模式

scriptsEx 的 `getKeys` 方法使用 `DictMemberGetCaller` 回调类来收集字典的所有键名。请解释 `EnumMembers` 的回调签名要求，以及 `DictMemberGetCaller::FuncCall` 的参数各代表什么。

<details>
<summary>查看答案</summary>

`EnumMembers` 的回调要求：传入一个实现了 `iTJSDispatch2` 接口的对象，引擎对字典/数组的每个成员调用该对象的 `FuncCall` 方法。

`FuncCall` 在 `EnumMembers` 上下文中的参数含义：

```cpp
tjs_error FuncCall(
    tjs_uint32 flag,         // 调用标志（通常为 0）
    const tjs_char *membername, // 当前枚举的成员名（字典键名）
    tjs_uint32 *hint,        // 名称 hash 提示（可忽略）
    tTJSVariant *result,     // 返回值（EnumMembers 不使用）
    tjs_int numparams,       // 参数个数（固定为 3）
    tTJSVariant **param,     // 参数数组：
                             //   param[0] = 成员名（tTJSVariant 字符串）
                             //   param[1] = 成员值（tTJSVariant）
                             //   param[2] = 标志位（0=普通成员, 1=隐藏成员等）
    iTJSDispatch2 *objthis   // 被枚举的对象
);
```

`DictMemberGetCaller` 的实现（简化）：
```cpp
class DictMemberGetCaller : public tTJSDispatch {
    iTJSDispatch2 *array;    // 收集结果的数组
    tjs_int idx;             // 当前写入位置

    tjs_error FuncCall(...) override {
        tTJSVariant key = param[0]; // 取成员名
        array->PropSetByNum(0, idx++, &key, array);
        // 将键名写入结果数组的第 idx 个位置
        return TJS_S_OK;
    }
};
```

使用方式：
```cpp
// 在 getKeys 方法中
DictMemberGetCaller caller(resultArray);
dict->EnumMembers(
    TJS_ENUM_NO_VALUE,  // 标志：不需要值，只要键名
    &caller,            // 回调对象
    dict                // 被枚举的字典
);
```

返回值 `TJS_S_OK` 表示继续枚举，返回 `TJS_S_FALSE` 可提前终止。

</details>

## 下一步

下一节 [02-psdfile与psbfile](./02-psdfile与psbfile.md) 将深入分析两个最复杂的插件——PSD 文件解析器和 PSB/E-mote 格式解析器，它们展示了如何用 ncbind 注册多层嵌套的对象模型以及处理二进制文件格式。
