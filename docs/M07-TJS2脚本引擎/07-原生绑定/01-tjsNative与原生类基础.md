# tjsNative 与原生类基础

> **所属模块：** M07-TJS2脚本引擎
> **前置知识：** [iTJSDispatch2接口](../06-对象系统/02-iTJSDispatch2接口.md)、[内置对象实现](../06-对象系统/03-内置对象实现.md)
> **预计阅读时间：** 45 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 TJS2 原生类（Native Class）的完整注册与实例化机制
2. 掌握 `TJS_BEGIN_NATIVE_MEMBERS` 系列宏的展开逻辑与使用方法
3. 追踪 `tTJSNativeClass::CreateNew` 从类调用到实例创建的全链路流程
4. 理解 ClassID 注册系统的设计原理与实现细节
5. 区分 `tTJSNativeClassMethod`、`tTJSNativeClassConstructor`、`tTJSNativeClassProperty` 三种成员包装器

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 原生类 | Native Class | 用 C++ 实现的 TJS2 类，通过宏系统注册后可在脚本中像普通类一样使用 |
| ClassID | Class Identifier | 每个原生类的唯一整数标识符，用于快速判断对象是否是某个原生类的实例 |
| 原生实例 | Native Instance | `tTJSNativeInstance` 的子类对象，保存 C++ 侧的状态数据，附着在 TJS2 对象上 |
| 成员注册 | Member Registration | 通过宏将 C++ 函数/属性注册为 TJS2 类的方法/属性的过程 |
| 闭包修复 | Closure Fixup | 创建实例时将类模板中的方法对象的 `objthis` 指针重定向到新实例的过程 |
| 成员复制 | Member Copy | `FuncCall` 中将类对象的所有非静态成员拷贝到新实例的过程 |
| 插件兼容类 | Native Class For Plugin | `tTJSNativeClassForPlugin`，使用函数指针代替虚函数以解决跨编译器兼容问题 |

## 1. 原生类在 TJS2 中的角色

TJS2 引擎中有两种类的来源：

1. **脚本定义类** —— 用户在 `.tjs` 脚本中用 `class` 关键字定义的类，完全由 TJS2 解释器管理
2. **原生类（Native Class）** —— 用 C++ 代码实现的类，通过注册宏暴露给 TJS2 脚本使用

原生类是 TJS2 引擎与 C++ 宿主程序之间的桥梁。Array、Dictionary、RegExp、Math、Date 等所有内置类型都是原生类。当你在 TJS2 脚本中写下：

```javascript
// TJS2 脚本
var arr = new Array();     // Array 是 C++ 实现的原生类
arr.add("hello");          // add() 是 C++ 函数，通过宏注册为 TJS2 方法
var len = arr.count;       // count 是 C++ 属性，通过宏注册为 TJS2 属性
```

背后发生的一切都依赖于 `tjsNative.h` / `tjsNative.cpp` 中定义的原生绑定基础设施。

### 1.1 原生类的核心设计思想

原生类系统需要解决三个核心问题：

```
问题 1: 如何让 C++ 函数看起来像 TJS2 方法？
  → 答案: 用宏生成包装结构体，统一为 tTJSNativeClassMethod 接口

问题 2: 如何让 C++ 对象的数据附着在 TJS2 对象上？
  → 答案: tTJSNativeInstance 机制，通过 NativeInstanceSupport 存取

问题 3: 如何判断一个 TJS2 对象是某个原生类的实例？
  → 答案: ClassID + ClassInstanceInfo 系统
```

下面我们逐一深入每个组件。

## 2. ClassID 注册系统

### 2.1 设计原理

TJS2 需要快速回答一个问题："这个对象是不是 Array 的实例？"。如果每次都用字符串比较类名，效率太低。因此引擎设计了一套 **ClassID** 系统——给每个原生类分配一个唯一的整数 ID，用整数比较代替字符串比较。

ClassID 系统的实现出人意料地简单，就是一个全局静态 `vector` 加线性搜索：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.cpp 第 22-34 行

// 全局原生类名称注册表——一个简单的字符串向量
static std::vector<ttstr> NativeClassNames;

// 注册原生类，返回 ClassID（即在向量中的索引）
tjs_int32 TJSRegisterNativeClass(const tjs_char *name) {
    // 先线性搜索，看是否已经注册过
    for (tjs_uint i = 0; i < NativeClassNames.size(); i++) {
        if (NativeClassNames[i] == name)
            return i;  // 已存在，直接返回索引
    }
    // 没找到，追加到末尾，索引就是新 ClassID
    NativeClassNames.push_back(ttstr(name));
    return (tjs_int32)(NativeClassNames.size() - 1);
}
```

为什么用线性搜索而不用哈希表？因为原生类的数量非常少——整个 KrKr2 引擎也就几十个原生类，线性搜索的开销可以忽略不计。而且注册只在引擎启动时发生一次，运行时不会再调用。

### 2.2 ClassID 查找

除了注册，还有一个查找函数：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.h 第 24-33 行

tjs_int32 TJSFindNativeClassID(const tjs_char *name) {
    // 同样是线性搜索
    for (tjs_uint i = 0; i < NativeClassNames.size(); i++) {
        if (NativeClassNames[i] == name)
            return i;  // 找到了
    }
    return -1;  // 未找到，返回 -1
}
```

### 2.3 ClassID 的使用场景

ClassID 主要在两个地方使用：

```
场景 1: instanceof 判断
  TJS2 脚本: if (obj instanceof "Array") { ... }
  C++ 实现: tTJSNativeClass::IsInstanceOf() 比较 ClassID

场景 2: 获取原生实例
  C++ 代码需要从 TJS2 对象中取出 C++ 原生数据时，
  用 ClassID 通过 NativeInstanceSupport(TJS_NIS_GETINSTANCE) 获取
```

以下是 `IsInstanceOf` 的实现：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.cpp 第 442-456 行

tjs_error tTJSNativeClass::IsInstanceOf(
    tjs_uint32 flag, const tjs_char *membername,
    tjs_uint32 *hint, const tjs_char *classname,
    iTJSDispatch2 *objthis)
{
    if (membername == NULL) {
        // 判断对象本身是否是某个类的实例
        // "Class" 是所有原生类的基类名
        if (!TJS_strcmp(classname, TJS_W("Class")))
            return TJS_S_TRUE;
        // 比较 ClassName 是否匹配
        if (!TJS_strcmp(classname, ClassName.c_str()))
            return TJS_S_TRUE;
    }
    // 委托给父类继续判断
    return inherited::IsInstanceOf(flag, membername, hint,
                                   classname, objthis);
}
```

## 3. tTJSNativeInstance —— 原生实例基类

### 3.1 设计目的

当 TJS2 脚本创建一个 `new Array()` 时，引擎需要一个地方存放 C++ 侧的 Array 数据（比如内部的 `std::vector`）。这个"存放 C++ 数据的容器"就是 `tTJSNativeInstance`。

每个原生类都有对应的 NativeInstance 子类。比如 Array 有 `tTJSArrayNI`，Date 有 `tTJSDateNI`。它们继承 `tTJSNativeInstance` 并添加自己的数据字段。

### 3.2 接口定义

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.h 第 42-54 行

class tTJSNativeInstance {
public:
    // 构造函数和析构函数
    tTJSNativeInstance() {}
    virtual ~tTJSNativeInstance() {}

    // 在 TJS2 对象构造时调用（接收构造参数）
    virtual tjs_error TJS_INTF_METHOD
        Construct(tjs_int numparams, tTJSVariant **param,
                  iTJSDispatch2 *tjs_obj) {
        return TJS_S_OK;  // 默认实现什么都不做
    }

    // 在 TJS2 对象失效时调用（释放资源）
    virtual void TJS_INTF_METHOD
        Invalidate() {}

    // 在 TJS2 对象销毁时调用
    virtual void TJS_INTF_METHOD
        Destruct() { delete this; }
};
```

三个生命周期方法的调用时机：

```
new Array(arg1, arg2)
  │
  ├─ 1. CreateNativeInstance() → new tTJSArrayNI()
  │     ↳ 调用默认构造函数，此时只分配内存
  │
  ├─ 2. NativeInstanceSupport(TJS_NIS_REGISTER) → 注册到 TJS2 对象
  │     ↳ 将 NI 指针存入 TJS2 对象的内部存储
  │
  ├─ 3. Construct(2, [arg1, arg2], objthis)
  │     ↳ 初始化 C++ 数据，处理构造参数
  │
  │  ... 对象正常使用 ...
  │
  ├─ 4. Invalidate()
  │     ↳ 释放外部资源（如文件句柄、网络连接等）
  │     ↳ invalidate 可以被脚本显式调用
  │
  └─ 5. Destruct()
        ↳ 释放 NI 对象本身，默认执行 delete this
        ↳ 由垃圾回收器在 TJS2 对象释放时调用
```

### 3.3 实际示例：tTJSArrayNI

Array 的原生实例存放了真正的数据容器：

```cpp
// 简化自 cpp/core/tjs2/tjsArray.h

class tTJSArrayNI : public tTJSNativeInstance {
    // 内部数组存储（vector<tTJSVariant>）
    std::vector<tTJSVariant> Items;

public:
    // 构造时可以接收初始大小
    tjs_error Construct(tjs_int numparams,
                        tTJSVariant **param,
                        iTJSDispatch2 *tjs_obj) override {
        if (numparams >= 1) {
            // 如果传了参数，预分配空间
            tjs_int count = (tjs_int)*param[0];
            Items.resize(count);
        }
        return TJS_S_OK;
    }

    // 失效时清空数据
    void Invalidate() override {
        Items.clear();
    }

    // 获取元素个数
    tjs_int GetCount() const { return (tjs_int)Items.size(); }

    // ... 其他数据操作方法 ...
};
```

## 4. 成员包装器——Method、Constructor、Property

原生类的每个成员（方法、构造函数、属性）都需要一个 C++ 包装对象来承载。TJS2 定义了三种专用包装器。

### 4.1 回调函数类型

所有包装器的核心是一个统一的回调函数签名：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.h 第 62-65 行

// 原生类方法的回调函数指针类型
typedef tjs_error (*tTJSNativeClassMethodCallback)(
    tTJSVariant *result,       // 返回值（输出参数）
    tjs_int numparams,         // 参数个数
    tTJSVariant **param,       // 参数数组
    iTJSDispatch2 *objthis     // 调用者对象（this）
);
```

这个签名是原生绑定的"通用语言"——无论是方法、构造函数还是属性的 getter/setter，最终都要转换为这种形式。

### 4.2 tTJSNativeClassMethod —— 方法包装器

方法包装器将一个 C++ 回调函数包装成 TJS2 可调用的方法对象：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.h 第 83-102 行

class tTJSNativeClassMethod : public tTJSDispatch {
    tTJSNativeClassMethodCallback Process;  // 指向实际处理函数

public:
    tTJSNativeClassMethod(tTJSNativeClassMethodCallback process)
        : Process(process) {}

    // FuncCall 是 iTJSDispatch2 接口方法
    tjs_error FuncCall(tjs_uint32 flag,
        const tjs_char *membername,
        tjs_uint32 *hint,
        tTJSVariant *result,
        tjs_int numparams,
        tTJSVariant **param,
        iTJSDispatch2 *objthis) override;
};
```

`FuncCall` 的实现有一个关键的分派逻辑：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.cpp 第 86-106 行

tjs_error tTJSNativeClassMethod::FuncCall(
    tjs_uint32 flag, const tjs_char *membername,
    tjs_uint32 *hint, tTJSVariant *result,
    tjs_int numparams, tTJSVariant **param,
    iTJSDispatch2 *objthis)
{
    if (membername != NULL) {
        // 如果指定了成员名称，说明是在访问方法对象的成员
        // 委托给父类 tTJSDispatch 处理
        return tTJSDispatch::FuncCall(flag, membername, hint,
                                       result, numparams, param,
                                       objthis);
    }

    // membername 为 NULL，表示"调用这个方法本身"
    if (result) result->Clear();  // 清空返回值

    // 调用实际的 C++ 回调函数
    return Process(result, numparams, param, objthis);
}
```

**关键设计点**：`membername == NULL` 表示"调用这个对象本身"，是 TJS2 调用约定的核心。当 TJS2 脚本调用 `arr.add("hello")` 时，引擎先通过 `PropGet(0, "add", ...)` 获取 `add` 对应的 `tTJSNativeClassMethod` 对象，然后对这个对象调用 `FuncCall(0, NULL, ...)` 来执行它。

### 4.3 tTJSNativeClassConstructor —— 构造函数包装器

构造函数包装器与普通方法类似，但有一个微妙的区别：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.h 第 112-125 行

class tTJSNativeClassConstructor : public tTJSNativeClassMethod {
public:
    tTJSNativeClassConstructor(tTJSNativeClassMethodCallback process)
        : tTJSNativeClassMethod(process) {}

    tjs_error FuncCall(tjs_uint32 flag,
        const tjs_char *membername,
        tjs_uint32 *hint,
        tTJSVariant *result,
        tjs_int numparams,
        tTJSVariant **param,
        iTJSDispatch2 *objthis) override;
};
```

关键区别在 `FuncCall` 的实现中：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.cpp 第 118-134 行

tjs_error tTJSNativeClassConstructor::FuncCall(
    tjs_uint32 flag, const tjs_char *membername,
    tjs_uint32 *hint, tTJSVariant *result,
    tjs_int numparams, tTJSVariant **param,
    iTJSDispatch2 *objthis)
{
    if (membername != NULL) {
        // 构造函数的成员访问委托给 tTJSDispatch::FuncCall
        // 注意：这里跳过了 tTJSNativeClassMethod 层！
        return tTJSDispatch::FuncCall(flag, membername, hint,
                                      result, numparams, param,
                                      objthis);
    }

    // membername 为 NULL，调用构造函数本身
    return Process(result, numparams, param, objthis);
}
```

为什么构造函数要跳过 `tTJSNativeClassMethod::FuncCall` 直接回到 `tTJSDispatch::FuncCall`？因为构造函数是通过类名作为成员名来调用的（`FuncCall(0, "Array", ...)`)。如果走 `tTJSNativeClassMethod` 的路径，当 `membername != NULL` 时会调用 `tTJSDispatch::FuncCall` 的普通成员查找逻辑，而构造函数需要的是 `tTJSDispatch` 基类的直接分派。

### 4.4 tTJSNativeClassProperty —— 属性包装器

属性包装器存放 getter 和 setter 两个回调函数：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.h 第 155-179 行

class tTJSNativeClassProperty : public tTJSDispatch {
    // getter 和 setter 回调函数指针
    tTJSNativeClassMethodCallback Get;
    tTJSNativeClassMethodCallback Set;

public:
    tTJSNativeClassProperty(
        tTJSNativeClassMethodCallback get,
        tTJSNativeClassMethodCallback set)
        : Get(get), Set(set) {}

    // 属性读取
    tjs_error PropGet(tjs_uint32 flag,
        const tjs_char *membername,
        tjs_uint32 *hint,
        tTJSVariant *result,
        iTJSDispatch2 *objthis) override {
        if (membername) return tTJSDispatch::PropGet(flag,
            membername, hint, result, objthis);
        if (!objthis) return TJS_E_NATIVECLASSCRASH;
        return Get(result, 0, NULL, objthis);
    }

    // 属性写入
    tjs_error PropSet(tjs_uint32 flag,
        const tjs_char *membername,
        tjs_uint32 *hint,
        const tTJSVariant *param,
        iTJSDispatch2 *objthis) override {
        if (membername) return tTJSDispatch::PropSet(flag,
            membername, hint, param, objthis);
        if (!objthis) return TJS_E_NATIVECLASSCRASH;
        return Set(const_cast<tTJSVariant*>(param),
                   0, NULL, objthis);
    }
};
```

注意属性包装器有一个保护机制：当 `objthis` 为 `NULL` 时返回 `TJS_E_NATIVECLASSCRASH`。这是因为属性 getter/setter 几乎总是需要访问对象实例数据，如果没有 `objthis`，说明出了严重错误。

## 5. tTJSNativeClass —— 原生类核心

`tTJSNativeClass` 是所有原生类的 C++ 基类。它继承自 `tTJSExtendableObject`（可扩展对象，支持动态添加成员），提供了成员注册、实例创建、实例初始化等核心功能。

### 5.1 类定义

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.h 第 185-225 行

class tTJSNativeClass : public tTJSExtendableObject {
    typedef tTJSExtendableObject inherited;

protected:
    ttstr ClassName;         // 类名，如 "Array"
    tjs_int32 _ClassID;      // 通过 TJSRegisterNativeClass 获得的 ID

public:
    tTJSNativeClass(const ttstr &name);
    ~tTJSNativeClass();

    // 注册一个成员（方法、属性、构造函数等）到类对象
    void RegisterNCM(
        const tjs_char *name,
        iTJSDispatch2 *dsp,
        const tjs_char *classname,
        tjs_uint32 Type,       // TJS_HIDDENMEMBER / TJS_STATICMEMBER 等标志
        tjs_uint32 Flags
    );

    // 创建原生实例——子类必须覆盖此方法
    virtual tTJSNativeInstance *CreateNativeInstance() {
        return nullptr;  // 默认不创建（如 Math 这种纯静态类）
    }

    // 创建 TJS2 侧的基础对象
    iTJSDispatch2 *CreateBaseTJSObject();

    // FuncCall 用于实例初始化（复制成员）
    tjs_error FuncCall(tjs_uint32 flag,
        const tjs_char *membername,
        tjs_uint32 *hint,
        tTJSVariant *result,
        tjs_int numparams,
        tTJSVariant **param,
        iTJSDispatch2 *objthis) override;

    // CreateNew 是 "new ClassName()" 的入口
    tjs_error CreateNew(tjs_uint32 flag,
        const tjs_char *membername,
        tjs_uint32 *hint,
        iTJSDispatch2 **result,
        tjs_int numparams,
        tTJSVariant **param,
        iTJSDispatch2 *objthis) override;

    // 实例判断
    tjs_error IsInstanceOf(tjs_uint32 flag,
        const tjs_char *membername,
        tjs_uint32 *hint,
        const tjs_char *classname,
        iTJSDispatch2 *objthis) override;

    // 获取 ClassID
    tjs_int32 GetClassID() const { return _ClassID; }
};
```

### 5.2 RegisterNCM —— 成员注册

`RegisterNCM` 是将 C++ 包装对象注册到类对象中的核心方法：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.cpp 第 248-290 行

void tTJSNativeClass::RegisterNCM(
    const tjs_char *name,    // 成员名，如 "add"、"count"
    iTJSDispatch2 *dsp,      // 包装对象（Method/Property/Constructor）
    const tjs_char *classname,
    tjs_uint32 Type,         // 可见性标志
    tjs_uint32 Flags)
{
    // 1. 通过全局字符串映射优化名称查找
    tjs_char *narrow_name = TJSMapGlobalStringMap(name);

    // 2. 设置调试类型信息（便于调试时查看）
    // dsp->SetDebugTypeInfo(...);

    // 3. 将成员添加到类对象中
    //    TJS_MEMBERENSURE: 如果不存在就创建
    //    TJS_IGNOREPROP: 直接设置值，不触发属性 setter
    tTJSVariant val(dsp, NULL);
    tjs_error hr;

    // 首先尝试用优化的 PropSetByVS 方法
    tTJSVariantString *vs = TJSAllocVariantString(name);
    try {
        hr = PropSetByVS(
            TJS_MEMBERENSURE | TJS_IGNOREPROP | Type,
            vs, &val, this);
    } catch (...) {
        vs->Release();
        throw;
    }
    vs->Release();

    if (TJS_FAILED(hr)) {
        // 降级：使用普通 PropSet
        hr = PropSet(
            TJS_MEMBERENSURE | TJS_IGNOREPROP | Type,
            name, NULL, &val, this);
    }

    // 4. 释放 dsp 的引用计数
    //    （RegisterNCM 的调用者不再持有，类对象通过 PropSet 持有）
    dsp->Release();
}
```

注意最后的 `dsp->Release()`——这里有一个隐含的所有权转移。调用 `RegisterNCM` 时，`dsp` 的引用计数为 1（由工厂函数创建时设置）。`PropSet` 内部会 `AddRef`，使计数变为 2。然后 `Release` 使计数回到 1，此时只有类对象持有这个成员。

### 5.3 FuncCall —— 实例初始化（成员复制）

当 `tTJSNativeClass` 被当作函数调用（`membername == NULL`）时，它执行的是**实例初始化**——将类模板中的所有非静态成员复制到新实例上。这是 TJS2 原生类对象创建的核心步骤：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.cpp 第 305-378 行
// 以下为简化注释版

tjs_error tTJSNativeClass::FuncCall(
    tjs_uint32 flag, const tjs_char *membername,
    tjs_uint32 *hint, tTJSVariant *result,
    tjs_int numparams, tTJSVariant **param,
    iTJSDispatch2 *objthis)
{
    if (membername != NULL) {
        // 有成员名——这是在调用类的某个成员方法
        // 比如 Array.add(...)，委托给父类
        return inherited::FuncCall(flag, membername, hint,
                                   result, numparams, param,
                                   objthis);
    }

    // ====== membername == NULL: 实例初始化 ======

    // 步骤 1: 将当前类名添加到目标对象的类信息中
    objthis->ClassInstanceInfo(TJS_CII_ADD, 0, &ClassName);

    // 步骤 2: 创建原生实例（C++ 侧数据对象）
    tTJSNativeInstance *ni = CreateNativeInstance();
    if (ni) {
        // 步骤 3: 将原生实例注册到 TJS2 对象
        tjs_error hr = objthis->NativeInstanceSupport(
            TJS_NIS_REGISTER, _ClassID, &ni);
        if (TJS_FAILED(hr)) {
            ni->Destruct();
            return hr;
        }
    }

    // 步骤 4: 遍历类模板的所有成员，复制到新实例
    // 这是最关键的一步——让新对象拥有类的所有方法和属性
    tTJSVariant val;
    tjs_int count = 0;
    EnumMembers(TJS_IGNOREPROP, &count, NULL, NULL);
    // count 现在是成员总数

    for (tjs_int i = 0; i < count; i++) {
        tTJSVariant name_var;
        // 获取第 i 个成员的名字和值
        if (TJS_FAILED(EnumMembers(TJS_IGNOREPROP,
                                    &i, &val, &name_var)))
            continue;

        // 检查成员标志
        tjs_uint32 flags = (tjs_uint32)(tjs_int)name_var;
        if (flags & TJS_STATICMEMBER)
            continue;  // 跳过静态成员——它们属于类，不属于实例

        // 步骤 5: 闭包修复（Closure Fixup）
        // 如果成员是一个对象（如方法），检查它的闭包 objthis
        // 如果闭包的 objthis 为 NULL，将其重定向到新实例
        if (val.Type() == tvtObject) {
            iTJSDispatch2 *valobj = val.AsObjectNoAddRef();
            if (valobj) {
                iTJSDispatch2 *closure_objthis =
                    val.AsObjectThisNoAddRef();
                if (!closure_objthis) {
                    // objthis 为空，修复为指向新实例
                    valobj->ChangeClosureObjThis(objthis);
                }
            }
        }

        // 步骤 6: 设置到新实例
        ttstr name_str(name_var);
        objthis->PropSetByVS(
            TJS_MEMBERENSURE | TJS_IGNOREPROP,
            name_str.AsVariantStringNoAddRef(),
            &val, objthis);
    }

    return TJS_S_OK;
}
```

闭包修复是一个精妙的设计。类模板中定义的方法对象（如 `add` 方法的 `tTJSNativeClassMethod`）最初不知道自己会被哪个实例调用，所以它们的 `objthis` 为 NULL。当这些方法被复制到实例时，`ChangeClosureObjThis` 将方法绑定到具体的实例对象，这样方法内部通过 `objthis` 就能正确访问实例数据。

用流程图表示这个过程：

```
┌────────────────────────────────────────────┐
│         类模板对象 (tTJSNativeClass)          │
│                                            │
│  成员: add(objthis=NULL)                    │
│  成员: remove(objthis=NULL)                 │
│  成员: count(getter, objthis=NULL)          │
│  成员: length(static, 不复制)               │
└────────────────┬───────────────────────────┘
                 │ FuncCall(NULL, ...)
                 │ 成员复制 + 闭包修复
                 ▼
┌────────────────────────────────────────────┐
│           新实例 (objthis)                   │
│                                            │
│  成员: add(objthis=新实例)      ← 已修复     │
│  成员: remove(objthis=新实例)   ← 已修复     │
│  成员: count(getter, objthis=新实例)← 已修复 │
│  原生实例: tTJSArrayNI* ← 通过 NIS 注册     │
└────────────────────────────────────────────┘
```

### 5.4 CreateNew —— 完整的对象创建流程

`CreateNew` 是 TJS2 脚本中 `new ClassName()` 的最终入口。它协调整个对象创建过程：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.cpp 第 381-439 行
// 以下为简化注释版

tjs_error tTJSNativeClass::CreateNew(
    tjs_uint32 flag, const tjs_char *membername,
    tjs_uint32 *hint, iTJSDispatch2 **result,
    tjs_int numparams, tTJSVariant **param,
    iTJSDispatch2 *objthis)
{
    if (membername != NULL) {
        // 有成员名，委托给父类
        return inherited::CreateNew(flag, membername, hint,
                                    result, numparams, param,
                                    objthis);
    }

    // ====== 5 步创建流程 ======

    // 步骤 1: 如果有父类（SuperClass），先创建父类实例
    iTJSDispatch2 *superinst = NULL;
    if (SuperClass) {
        tjs_error hr = SuperClass->CreateNew(
            0, NULL, NULL, &superinst,
            0, NULL, NULL);
        if (TJS_FAILED(hr)) return hr;
    }

    // 步骤 2: 创建 TJS2 侧的基础对象
    //   如果有 SuperClass → tTJSExtendableObject（可继承）
    //   如果没有 SuperClass → tTJSCustomObject（普通对象）
    iTJSDispatch2 *dsp = CreateBaseTJSObject();

    // 步骤 3: 调用 FuncCall(NULL) 初始化实例
    //   这一步执行上面 5.3 节描述的成员复制和闭包修复
    tjs_error hr = FuncCall(0, NULL, NULL, NULL,
                             0, NULL, dsp);
    if (TJS_FAILED(hr)) {
        dsp->Release();
        if (superinst) superinst->Release();
        return hr;
    }

    // 步骤 4: 如果有父类实例，将父类实例信息注册到新对象
    if (superinst) {
        tTJSVariant val(superinst, superinst);
        dsp->ClassInstanceInfo(TJS_CII_SET_SUPRECLASS,
                               0, &val);
        superinst->Release();
    }

    // 步骤 5: 调用构造函数
    //   通过类名作为成员名调用 FuncCall
    hr = FuncCall(0, ClassName.c_str(), NULL, NULL,
                   numparams, param, dsp);
    if (hr == TJS_E_MEMBERNOTFOUND) {
        // 没有构造函数是合法的——不是错误
        hr = TJS_S_OK;
    }
    if (TJS_FAILED(hr)) {
        dsp->Release();
        return hr;
    }

    // 设置返回值
    *result = dsp;
    return TJS_S_OK;
}
```

将整个过程用流程图展示：

```
TJS2 脚本: var obj = new MyClass(arg1, arg2);
│
▼
tTJSNativeClass::CreateNew(NULL, ..., [arg1, arg2])
│
├─ Step 1: SuperClass?.CreateNew()
│           → 创建父类实例（递归调用）
│
├─ Step 2: CreateBaseTJSObject()
│           → 创建空的 TJS2 对象容器
│           → 有父类: tTJSExtendableObject
│           → 无父类: tTJSCustomObject
│
├─ Step 3: FuncCall(NULL, ..., dsp)
│           → 实例初始化（成员复制 + 闭包修复）
│           → CreateNativeInstance() → NI 注册
│
├─ Step 4: ClassInstanceInfo(TJS_CII_SET_SUPRECLASS)
│           → 关联父类实例
│
└─ Step 5: FuncCall("MyClass", ..., dsp)
            → 调用构造函数（可以不存在）
            → 构造函数内部调用 NI::Construct()
            ↓
            *result = dsp  ← 返回新创建的对象
```

### 5.5 CreateBaseTJSObject —— 选择基础对象类型

```cpp
// 简化自 cpp/core/tjs2/tjsNative.h

iTJSDispatch2 *tTJSNativeClass::CreateBaseTJSObject() {
    if (SuperClass) {
        // 有父类 → 使用可扩展对象（支持继承链）
        return new tTJSExtendableObject();
    } else {
        // 无父类 → 使用自定义对象（更轻量）
        return new tTJSCustomObject();
    }
}
```

为什么要区分？`tTJSExtendableObject` 比 `tTJSCustomObject` 多了继承相关的支持（如 `ClassInstanceInfo` 的完整实现），但也更重。大多数原生类没有父类，所以使用更轻量的 `tTJSCustomObject` 即可。

## 6. TJS_BEGIN_NATIVE_MEMBERS 宏系统

手写 `RegisterNCM` 调用非常繁琐——需要为每个方法手动创建回调函数、构造包装对象、注册到类。TJS2 提供了一套宏来简化这个过程。这些宏定义在 `tjsNative.h` 第 293-480 行。

### 6.1 宏系统总览

```
宏系统的分层结构：

┌─────────────────────────────────────────────────────────┐
│  最外层: TJS_BEGIN_NATIVE_MEMBERS / TJS_END_NATIVE_MEMBERS │
│  ↳ 打开/关闭成员注册块，设置 ClassID                      │
│                                                         │
│  ├─ 方法: TJS_BEGIN_NATIVE_METHOD_DECL                   │
│  │        TJS_END_NATIVE_METHOD_DECL                     │
│  │        TJS_END_NATIVE_STATIC_METHOD_DECL              │
│  │        TJS_END_NATIVE_HIDDEN_METHOD_DECL              │
│  │                                                       │
│  ├─ 构造: TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL               │
│  │        TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL_NO_INSTANCE   │
│  │                                                       │
│  ├─ 属性: TJS_BEGIN_NATIVE_PROP_DECL                     │
│  │        TJS_END_NATIVE_PROP_DECL                       │
│  │        TJS_BEGIN_NATIVE_PROP_GETTER                    │
│  │        TJS_END_NATIVE_PROP_GETTER                     │
│  │        TJS_BEGIN_NATIVE_PROP_SETTER                    │
│  │        TJS_END_NATIVE_PROP_SETTER                     │
│  │        TJS_DENY_NATIVE_PROP_GETTER                    │
│  │        TJS_DENY_NATIVE_PROP_SETTER                    │
│  │                                                       │
│  └─ 辅助: TJS_DECL_EMPTY_FINALIZE_METHOD                 │
│           TJS_GET_NATIVE_INSTANCE                        │
│           TJS_PARAM_EXIST                                │
└─────────────────────────────────────────────────────────┘
```

### 6.2 TJS_BEGIN_NATIVE_MEMBERS —— 开始成员注册

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.h 第 319-326 行

#define TJS_BEGIN_NATIVE_MEMBERS(classname)                \
    /* 注册原生类名称，获取 ClassID */                       \
    _ClassID = TJSRegisterNativeClass(TJS_W(#classname));  \
    /* 以下代码块中声明成员 */                               \
    {                                                       \
        /* classname 会被后续宏使用 */
```

这个宏做的事情很简单：调用 `TJSRegisterNativeClass` 将类名（如 `"Array"`）注册到全局表中，获得 `_ClassID`。然后打开一个代码块 `{`，在这个块内可以使用其他宏声明方法和属性。

### 6.3 TJS_BEGIN_NATIVE_METHOD_DECL —— 声明方法

方法声明宏是最常用的宏，它生成一个包含静态 `Process` 方法的匿名结构体：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.h 第 332-342 行

#define TJS_BEGIN_NATIVE_METHOD_DECL(name)                \
    {                                                      \
        /* 生成一个名为 NCM_##name 的结构体 */              \
        struct NCM_##name {                                \
            static tjs_error TJS_INTF_METHOD Process(      \
                tTJSVariant *result,                       \
                tjs_int numparams,                         \
                tTJSVariant **param,                       \
                iTJSDispatch2 *objthis)                    \
            {
// ---- 用户在这里写方法实现代码 ----

#define TJS_END_NATIVE_METHOD_DECL(name)                  \
            }  /* Process 函数结束 */                       \
        };     /* NCM_##name 结构体结束 */                  \
        /* 创建包装对象并注册 */                              \
        RegisterNCM(                                       \
            TJS_W(#name),                                  \
            TJSCreateNativeClassMethod(NCM_##name::Process),\
            ClassName.c_str(),                             \
            0,  /* Type: 普通成员 */                        \
            0   /* Flags */                                \
        );                                                 \
    }
```

宏展开后的完整代码如下：

```cpp
// 原始宏调用（来自 tjsArray.cpp 的 add 方法）:
TJS_BEGIN_NATIVE_METHOD_DECL(add)
{
    TJS_GET_NATIVE_INSTANCE(ni, tTJSArrayNI);
    if (numparams < 1)
        return TJS_E_BADPARAMCOUNT;
    ni->Items.push_back(*param[0]);
    return TJS_S_OK;
}
TJS_END_NATIVE_METHOD_DECL(add)

// 展开后等价于:
{
    struct NCM_add {
        static tjs_error TJS_INTF_METHOD Process(
            tTJSVariant *result,
            tjs_int numparams,
            tTJSVariant **param,
            iTJSDispatch2 *objthis)
        {
            // --- 用户代码开始 ---
            TJS_GET_NATIVE_INSTANCE(ni, tTJSArrayNI);
            if (numparams < 1)
                return TJS_E_BADPARAMCOUNT;
            ni->Items.push_back(*param[0]);
            return TJS_S_OK;
            // --- 用户代码结束 ---
        }
    };
    RegisterNCM(
        TJS_W("add"),
        TJSCreateNativeClassMethod(NCM_add::Process),
        ClassName.c_str(), 0, 0);
}
```

`TJSCreateNativeClassMethod` 是一个工厂函数，创建 `tTJSNativeClassMethod` 对象并传入 `Process` 函数指针。

### 6.4 TJS_GET_NATIVE_INSTANCE —— 获取原生实例

在方法体内，几乎总是需要访问 C++ 侧的数据对象。这个宏负责从 `objthis` 中提取：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.h 第 293-309 行

#define TJS_GET_NATIVE_INSTANCE(varname, typename)         \
    typename *varname;                                      \
    {                                                       \
        tTJSNativeInstance *_ni;                            \
        tjs_error hr = objthis->NativeInstanceSupport(     \
            TJS_NIS_GETINSTANCE,                           \
            _ClassID,   /* 用 ClassID 定位正确的 NI */      \
            &_ni);                                         \
        if (TJS_FAILED(hr))                                \
            return TJS_E_NATIVECLASSCRASH;                 \
        varname = static_cast<typename*>(_ni);             \
    }
```

使用示例：

```cpp
TJS_BEGIN_NATIVE_METHOD_DECL(getCount)
{
    // 从 objthis 获取 Array 的原生实例
    TJS_GET_NATIVE_INSTANCE(ni, tTJSArrayNI);

    // 现在可以访问 C++ 数据了
    if (result) *result = (tjs_int)ni->Items.size();
    return TJS_S_OK;
}
TJS_END_NATIVE_METHOD_DECL(getCount)
```

### 6.5 静态方法与隐藏方法

除了普通方法，还有两种变体：

```cpp
// 静态方法——使用 TJS_STATICMEMBER 标志
#define TJS_END_NATIVE_STATIC_METHOD_DECL(name)           \
            }                                              \
        };                                                 \
        RegisterNCM(                                       \
            TJS_W(#name),                                  \
            TJSCreateNativeClassMethod(NCM_##name::Process),\
            ClassName.c_str(),                             \
            TJS_STATICMEMBER,  /* 静态成员标志 */           \
            0                                              \
        );                                                 \
    }

// 隐藏方法——使用 TJS_HIDDENMEMBER 标志
// 隐藏成员不会被 EnumMembers 枚举，也不会被复制到实例
#define TJS_END_NATIVE_HIDDEN_METHOD_DECL(name)           \
            }                                              \
        };                                                 \
        RegisterNCM(                                       \
            TJS_W(#name),                                  \
            TJSCreateNativeClassMethod(NCM_##name::Process),\
            ClassName.c_str(),                             \
            TJS_HIDDENMEMBER,  /* 隐藏成员标志 */           \
            0                                              \
        );                                                 \
    }
```

静态方法在 `FuncCall` 的成员复制阶段会被跳过（`flags & TJS_STATICMEMBER` 检查），因为静态成员属于类而不是实例。隐藏成员则完全不出现在枚举中。

### 6.6 构造函数声明

构造函数宏将两件事合并：声明构造函数回调和获取原生实例：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.h 第 384-411 行

#define TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL(varname, typename, classname) \
    TJS_BEGIN_NATIVE_METHOD_DECL(classname)  /* 构造函数以类名为方法名 */ \
    {                                                                    \
        /* 获取已注册的原生实例 */                                        \
        typename *varname;                                               \
        {                                                                \
            tTJSNativeInstance *_ni;                                     \
            tjs_error hr = objthis->NativeInstanceSupport(              \
                TJS_NIS_GETINSTANCE, _ClassID, &_ni);                   \
            if (TJS_FAILED(hr)) return TJS_E_NATIVECLASSCRASH;         \
            varname = static_cast<typename*>(_ni);                      \
        }                                                                \
        /* 调用原生实例的 Construct 方法 */                               \
        tjs_error hr = varname->Construct(numparams, param, objthis);  \
        if (TJS_FAILED(hr)) return hr;
// ---- 用户在这里写额外的构造逻辑 ----

// 使用示例（来自 tjsDate.cpp）:
TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL(ni, tTJSDateNI, Date)
{
    // ni->Construct() 已经被调用过了
    // 这里可以做额外的初始化
}
TJS_END_NATIVE_METHOD_DECL(Date)
```

注意构造函数的方法名就是类名（如 `"Date"`）。这是因为 `CreateNew` 在步骤 5 中用 `FuncCall(0, ClassName.c_str(), ...)` 来查找和调用构造函数。

对于没有原生实例的纯静态类（如 Math），使用无实例版本：

```cpp
#define TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL_NO_INSTANCE(classname) \
    TJS_BEGIN_NATIVE_METHOD_DECL(classname)
// 不获取 NI，不调用 Construct

// Math 的构造函数声明:
TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL_NO_INSTANCE(Math)
{
    // Math 没有原生实例，构造函数什么都不做
    return TJS_S_OK;
}
TJS_END_NATIVE_METHOD_DECL(Math)
```

### 6.7 属性声明

属性宏比方法宏复杂一些，因为需要分别声明 getter 和 setter：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.h 第 435-480 行

#define TJS_BEGIN_NATIVE_PROP_DECL(name)                   \
    {                                                       \
        struct NCM_##name {

#define TJS_BEGIN_NATIVE_PROP_GETTER                        \
            static tjs_error TJS_INTF_METHOD Get(           \
                tTJSVariant *result,                        \
                iTJSDispatch2 *objthis) {

#define TJS_END_NATIVE_PROP_GETTER                          \
            }  /* Get 结束 */

#define TJS_BEGIN_NATIVE_PROP_SETTER                        \
            static tjs_error TJS_INTF_METHOD Set(           \
                const tTJSVariant *param,                   \
                iTJSDispatch2 *objthis) {

#define TJS_END_NATIVE_PROP_SETTER                          \
            }  /* Set 结束 */

#define TJS_END_NATIVE_PROP_DECL(name)                     \
        };  /* NCM_##name 结束 */                           \
        RegisterNCM(                                        \
            TJS_W(#name),                                   \
            TJSCreateNativeClassProperty(                   \
                NCM_##name::Get, NCM_##name::Set),          \
            ClassName.c_str(), 0, 0);                       \
    }
```

完整的属性使用示例：

```cpp
// Array 的 count 属性（只读）
TJS_BEGIN_NATIVE_PROP_DECL(count)
{
    TJS_BEGIN_NATIVE_PROP_GETTER
    {
        TJS_GET_NATIVE_INSTANCE(ni, tTJSArrayNI);
        if (result) *result = (tjs_int)ni->Items.size();
        return TJS_S_OK;
    }
    TJS_END_NATIVE_PROP_GETTER

    TJS_DENY_NATIVE_PROP_SETTER  // 只读属性，拒绝写入
}
TJS_END_NATIVE_PROP_DECL(count)
```

`TJS_DENY_NATIVE_PROP_GETTER` 和 `TJS_DENY_NATIVE_PROP_SETTER` 直接返回 `TJS_E_ACCESSDENYED`：

```cpp
#define TJS_DENY_NATIVE_PROP_GETTER                        \
    static tjs_error TJS_INTF_METHOD Get(                  \
        tTJSVariant *result,                               \
        iTJSDispatch2 *objthis) {                          \
        return TJS_E_ACCESSDENYED;                         \
    }

#define TJS_DENY_NATIVE_PROP_SETTER                        \
    static tjs_error TJS_INTF_METHOD Set(                  \
        const tTJSVariant *param,                          \
        iTJSDispatch2 *objthis) {                          \
        return TJS_E_ACCESSDENYED;                         \
    }
```

### 6.8 辅助宏

```cpp
// 检查参数是否存在（非 void）
#define TJS_PARAM_EXIST(num)                               \
    (numparams > (num) &&                                  \
     param[num]->Type() != tvtVoid)

// 声明空的 finalize 方法
// finalize 在对象销毁时调用，很多原生类不需要特殊的 finalize 逻辑
#define TJS_DECL_EMPTY_FINALIZE_METHOD                     \
    TJS_BEGIN_NATIVE_METHOD_DECL(finalize)                 \
    {                                                       \
        return TJS_S_OK;                                   \
    }                                                       \
    TJS_END_NATIVE_METHOD_DECL(finalize)
```

## 7. tTJSNativeClassForPlugin —— 插件兼容类

### 7.1 跨编译器兼容问题

当插件以 DLL（动态链接库）形式加载时，插件可能用不同的编译器编译。C++ 虚函数表（vtable）的布局在不同编译器之间可能不同——MSVC 和 GCC 可能用不同的方式组织虚函数指针。这意味着如果引擎用 MSVC 编译，插件用 GCC 编译，直接调用虚函数可能导致崩溃。

`tTJSNativeClassForPlugin` 通过用**函数指针**代替虚函数来解决这个问题：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.h 第 265-277 行

class tTJSNativeClassForPlugin : public tTJSNativeClass {
    typedef tTJSNativeClass inherited;

    // 用函数指针代替虚函数 CreateNativeInstance
    tTJSNativeInstance* (*procCreateNativeInstance)();

public:
    tTJSNativeClassForPlugin(
        const ttstr &name,
        tTJSNativeInstance* (*createNativeInstance)())
        : tTJSNativeClass(name),
          procCreateNativeInstance(createNativeInstance) {}

    // 覆盖虚函数，转发到函数指针
    tTJSNativeInstance *CreateNativeInstance() override {
        return procCreateNativeInstance
            ? procCreateNativeInstance()
            : nullptr;
    }
};
```

### 7.2 函数指针 vs 虚函数

```
                    引擎 (MSVC 编译)           插件 (GCC 编译)
               ┌──────────────────────┐   ┌──────────────────┐
 虚函数方案     │  vtable 布局: MSVC    │ ≠ │ vtable 布局: GCC  │
 (有风险)      │  CreateNativeInstance │   │ CreateNativeInstance│
               │  at offset 0x10      │   │ at offset 0x18    │ ← 偏移不同！
               └──────────────────────┘   └──────────────────┘

               ┌──────────────────────┐   ┌──────────────────┐
 函数指针方案   │  成员变量:            │   │ 传入函数指针:      │
 (安全)        │  procCreateNativeInst│ ← │ &MyClass::Create  │
               │  固定偏移，ABI 兼容    │   │ 普通函数，无 vtable│
               └──────────────────────┘   └──────────────────┘
```

这在 KrKr2 项目中实际上不是问题（所有插件都静态链接），但原始 KiriKiri 引擎支持动态加载 DLL 插件，所以保留了这个兼容设计。ncbind 系统默认使用 `tTJSNativeClassForPlugin`。

## 8. tTJSNativeFunction —— 独立原生函数

除了原生类的方法，TJS2 还支持注册独立的原生函数（不属于任何类）：

```cpp
// 源码位置: cpp/core/tjs2/tjsNative.h 第 486-509 行

class tTJSNativeFunction : public tTJSDispatch {
public:
    tTJSNativeFunction() {}
    virtual ~tTJSNativeFunction() {}

    // FuncCall 的实现
    tjs_error FuncCall(
        tjs_uint32 flag, const tjs_char *membername,
        tjs_uint32 *hint, tTJSVariant *result,
        tjs_int numparams, tTJSVariant **param,
        iTJSDispatch2 *objthis) override
    {
        if (membername != NULL)
            return tTJSDispatch::FuncCall(flag, membername,
                hint, result, numparams, param, objthis);
        // 调用纯虚方法——子类实现具体逻辑
        return Process(result, numparams, param, objthis);
    }

    // 子类必须实现
    virtual tjs_error Process(
        tTJSVariant *result, tjs_int numparams,
        tTJSVariant **param, iTJSDispatch2 *objthis) = 0;
};
```

独立函数通常用于工具性质的全局函数，如 `Debug.message()`。

## 动手实践

### 实践 1：追踪 Array 的创建流程

阅读以下代码，写出执行 `new Array(10)` 时各函数的调用顺序和关键参数：

```javascript
// TJS2 脚本
var arr = new Array(10);
```

完整调用链：

```
1. VM 执行 CreateNew 指令
   → 在全局作用域查找 "Array"
   → 获取 tTJSArrayClass* (tTJSNativeClass 的子类)

2. tTJSNativeClass::CreateNew(NULL, ..., 1, [10], NULL)
   ├─ SuperClass == NULL, 跳过 Step 1
   ├─ CreateBaseTJSObject() → new tTJSCustomObject()
   ├─ FuncCall(NULL, ..., dsp)
   │   ├─ ClassInstanceInfo(TJS_CII_ADD, "Array")
   │   ├─ CreateNativeInstance() → new tTJSArrayNI()
   │   ├─ NativeInstanceSupport(TJS_NIS_REGISTER, ClassID, &ni)
   │   └─ 枚举成员，复制 add/remove/count/... 到 dsp
   │       └─ 每个方法: ChangeClosureObjThis(dsp)
   ├─ FuncCall("Array", ..., 1, [10], dsp)
   │   ├─ 查找成员 "Array" → tTJSNativeClassConstructor
   │   └─ NCM_Array::Process(...)
   │       ├─ TJS_GET_NATIVE_INSTANCE(ni, tTJSArrayNI)
   │       └─ ni->Construct(1, [10], dsp)
   │           └─ Items.resize(10)
   └─ *result = dsp

3. VM 将 dsp 存入变量 arr
```

### 实践 2：用宏系统注册一个简单方法

仿照 Array 的 `add` 方法，为一个假想的 `Counter` 类注册 `increment` 方法：

```cpp
// 假设有以下原生实例类
class tTJSCounterNI : public tTJSNativeInstance {
public:
    tjs_int value;

    tjs_error Construct(tjs_int numparams,
                        tTJSVariant **param,
                        iTJSDispatch2 *tjs_obj) override {
        value = 0;
        if (numparams >= 1)
            value = (tjs_int)*param[0];
        return TJS_S_OK;
    }

    void Invalidate() override { value = 0; }
};

// 在某个 tTJSNativeClass 子类的成员注册中：
TJS_BEGIN_NATIVE_MEMBERS(Counter)

    // 空的 finalize
    TJS_DECL_EMPTY_FINALIZE_METHOD

    // 构造函数
    TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL(ni, tTJSCounterNI, Counter)
    {
        // ni->Construct() 已被自动调用
        // 可以在此做额外初始化
    }
    TJS_END_NATIVE_METHOD_DECL(Counter)

    // increment 方法
    TJS_BEGIN_NATIVE_METHOD_DECL(increment)
    {
        TJS_GET_NATIVE_INSTANCE(ni, tTJSCounterNI);
        tjs_int step = 1;
        if (TJS_PARAM_EXIST(0))
            step = (tjs_int)*param[0];
        ni->value += step;
        if (result) *result = ni->value;
        return TJS_S_OK;
    }
    TJS_END_NATIVE_METHOD_DECL(increment)

    // value 属性（读写）
    TJS_BEGIN_NATIVE_PROP_DECL(value)
    {
        TJS_BEGIN_NATIVE_PROP_GETTER
        {
            TJS_GET_NATIVE_INSTANCE(ni, tTJSCounterNI);
            if (result) *result = ni->value;
            return TJS_S_OK;
        }
        TJS_END_NATIVE_PROP_GETTER

        TJS_BEGIN_NATIVE_PROP_SETTER
        {
            TJS_GET_NATIVE_INSTANCE(ni, tTJSCounterNI);
            ni->value = (tjs_int)*param;
            return TJS_S_OK;
        }
        TJS_END_NATIVE_PROP_SETTER
    }
    TJS_END_NATIVE_PROP_DECL(value)

TJS_END_NATIVE_MEMBERS
```

## 对照项目源码

以下是本节涉及的关键源码文件和行号：

相关文件：
- `cpp/core/tjs2/tjsNative.h` 第 42-54 行 —— `tTJSNativeInstance` 基类定义
- `cpp/core/tjs2/tjsNative.h` 第 62-179 行 —— 三种成员包装器的声明
- `cpp/core/tjs2/tjsNative.h` 第 185-225 行 —— `tTJSNativeClass` 核心定义
- `cpp/core/tjs2/tjsNative.h` 第 265-277 行 —— `tTJSNativeClassForPlugin` 插件兼容
- `cpp/core/tjs2/tjsNative.h` 第 293-480 行 —— 宏系统完整定义
- `cpp/core/tjs2/tjsNative.h` 第 486-509 行 —— `tTJSNativeFunction` 独立函数
- `cpp/core/tjs2/tjsNative.cpp` 第 22-34 行 —— ClassID 注册实现
- `cpp/core/tjs2/tjsNative.cpp` 第 86-134 行 —— Method/Constructor FuncCall 实现
- `cpp/core/tjs2/tjsNative.cpp` 第 177-222 行 —— Property PropGet/PropSet 实现
- `cpp/core/tjs2/tjsNative.cpp` 第 248-290 行 —— `RegisterNCM` 成员注册
- `cpp/core/tjs2/tjsNative.cpp` 第 305-378 行 —— `FuncCall` 实例初始化
- `cpp/core/tjs2/tjsNative.cpp` 第 381-439 行 —— `CreateNew` 完整创建流程
- `cpp/core/tjs2/tjsArray.cpp` —— Array 类的完整原生成员注册示例
- `cpp/core/tjs2/tjsMath.cpp` —— Math 类（无实例的纯静态类）注册示例
- `cpp/core/tjs2/tjsDate.cpp` —— Date 类注册示例

## 常见错误与排查

### 错误 1：TJS_E_NATIVECLASSCRASH

```
症状: 脚本调用原生类方法时崩溃
原因: TJS_GET_NATIVE_INSTANCE 获取失败
排查:
  1. 检查 ClassID 是否正确（是否调用了 TJS_BEGIN_NATIVE_MEMBERS）
  2. 检查 CreateNativeInstance() 是否返回了正确类型的 NI
  3. 检查是否在 finalize 之后又调用了方法
```

### 错误 2：方法注册后无法从脚本调用

```
症状: TJS2 脚本调用方法报 "member not found"
原因: RegisterNCM 没有正确执行
排查:
  1. 检查宏的开始和结束是否匹配
     （TJS_BEGIN_NATIVE_METHOD_DECL 必须有对应的 TJS_END_NATIVE_METHOD_DECL）
  2. 检查方法名是否拼写正确——宏参数直接变成字符串
  3. 检查类是否注册到了全局作用域
```

### 错误 3：属性 getter 返回随机值

```
症状: 读取属性得到无意义的值
原因: result 参数未被正确设置
排查:
  1. getter 中必须写 if (result) *result = ...; 
     不能假设 result 非空
  2. 检查 NI 指针是否有效
  3. 确认 NI 中的数据是否被正确初始化
```

## 本节小结

- **ClassID 系统**：用全局 `vector<ttstr>` 存储类名，索引作为 ClassID，支持 O(1) 的实例类型判断
- **tTJSNativeInstance**：C++ 数据容器，通过 NativeInstanceSupport 附着在 TJS2 对象上，有 Construct/Invalidate/Destruct 三阶段生命周期
- **三种成员包装器**：Method（方法）、Constructor（构造函数）、Property（属性），都实现 iTJSDispatch2 接口
- **宏系统**：`TJS_BEGIN_NATIVE_MEMBERS` 系列宏自动生成包装结构体和注册调用，大幅简化原生类开发
- **CreateNew 五步流程**：创建父类实例 → 创建基础对象 → 成员复制（含闭包修复）→ 关联父类 → 调用构造函数
- **闭包修复**：将类模板中的方法对象的 `objthis` 重定向到新实例，确保方法能正确访问实例数据
- **tTJSNativeClassForPlugin**：用函数指针代替虚函数，解决跨编译器的 vtable 兼容问题

## 练习题与答案

### 题目 1：ClassID 的设计权衡

`TJSRegisterNativeClass` 使用线性搜索 `vector` 来查找已注册的类名。请分析：
1. 这种设计的时间复杂度是多少？
2. 为什么不用 `std::unordered_map` 获得 O(1) 查找？
3. 如果原生类数量从几十个增长到几万个，需要做什么改动？

<details>
<summary>查看答案</summary>

1. **时间复杂度**：注册和查找都是 O(n)，n 为已注册的原生类数量。

2. **不用哈希表的原因**：
   - 原生类数量极少（KrKr2 整个引擎约 10-30 个原生类），O(n) 和 O(1) 的差异在微秒级
   - 注册只在引擎启动时执行一次，不是热路径
   - `vector` 的内存局部性更好（顺序存储），cache 友好
   - 索引直接作为 ClassID，不需要额外的映射关系
   - 代码极简，减少维护成本

3. **如果需要扩展**：
```cpp
// 改用哈希表 + 自增 ID
static std::unordered_map<ttstr, tjs_int32> ClassNameToID;
static tjs_int32 NextClassID = 0;

tjs_int32 TJSRegisterNativeClass(const tjs_char *name) {
    auto it = ClassNameToID.find(ttstr(name));
    if (it != ClassNameToID.end())
        return it->second;
    tjs_int32 id = NextClassID++;
    ClassNameToID[ttstr(name)] = id;
    return id;
}
```
但实际上几万个原生类是不现实的场景。

</details>

### 题目 2：理解闭包修复

以下代码创建了一个 Array 实例。请解释为什么方法需要闭包修复，以及如果不修复会发生什么。

```javascript
var arr1 = new Array();
var arr2 = new Array();
arr1.add("a");
arr2.add("b");
```

<details>
<summary>查看答案</summary>

**为什么需要闭包修复**：

类模板中的 `add` 方法最初的 `objthis` 为 NULL：
```
Array 类模板
  └─ add 方法对象 (objthis = NULL)
```

当创建 `arr1` 和 `arr2` 时，`add` 方法被复制到两个实例：
```
arr1
  └─ add 方法对象 (objthis = arr1)  ← 修复后指向 arr1

arr2
  └─ add 方法对象 (objthis = arr2)  ← 修复后指向 arr2
```

**如果不修复**：

如果不修复闭包的 `objthis`，方法内部的 `TJS_GET_NATIVE_INSTANCE` 会收到 NULL 作为 `objthis`，调用 `NativeInstanceSupport(TJS_NIS_GETINSTANCE, ...)` 时因为 `objthis` 为 NULL 而返回错误码 `TJS_E_NATIVECLASSCRASH`，方法调用会失败。

即使某种情况下 `objthis` 不为 NULL（由调用者传入），两个实例的 `add` 方法可能指向同一个闭包对象，导致状态混乱——`arr1.add("a")` 可能修改 `arr2` 的数据。

闭包修复确保每个实例的方法都绑定到正确的 `this` 对象，这与 JavaScript 中 `Function.prototype.bind()` 的作用类似。

</details>

### 题目 3：编写一个完整的只读属性

为一个假想的 `Timer` 原生类编写 `elapsed` 只读属性。该属性返回自构造以来经过的毫秒数。要求使用 TJS_BEGIN_NATIVE_PROP_DECL 宏，包含完整的 getter 实现和 setter 拒绝。

<details>
<summary>查看答案</summary>

```cpp
#include <chrono>

// 原生实例类
class tTJSTimerNI : public tTJSNativeInstance {
public:
    // 使用 C++ chrono 记录启动时间
    std::chrono::steady_clock::time_point startTime;

    tjs_error Construct(tjs_int numparams,
                        tTJSVariant **param,
                        iTJSDispatch2 *tjs_obj) override {
        // 构造时记录当前时间
        startTime = std::chrono::steady_clock::now();
        return TJS_S_OK;
    }

    void Invalidate() override {}

    // 获取经过的毫秒数
    tjs_int GetElapsed() const {
        auto now = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<
            std::chrono::milliseconds>(now - startTime);
        return (tjs_int)ms.count();
    }
};

// 在 TJS_BEGIN_NATIVE_MEMBERS(Timer) 块中：

TJS_BEGIN_NATIVE_PROP_DECL(elapsed)
{
    TJS_BEGIN_NATIVE_PROP_GETTER
    {
        // 获取原生实例
        TJS_GET_NATIVE_INSTANCE(ni, tTJSTimerNI);
        // 返回经过的毫秒数
        if (result)
            *result = (tjs_int)ni->GetElapsed();
        return TJS_S_OK;
    }
    TJS_END_NATIVE_PROP_GETTER

    // 只读——拒绝写入
    TJS_DENY_NATIVE_PROP_SETTER
}
TJS_END_NATIVE_PROP_DECL(elapsed)
```

TJS2 脚本中的使用：
```javascript
var timer = new Timer();
// ... 做一些事情 ...
Debug.message("经过了 " + timer.elapsed + " 毫秒");

// 尝试写入会报错:
// timer.elapsed = 0;  // → TJS_E_ACCESSDENYED
```

</details>

## 下一步

[ncbind 宏系统详解](./02-ncbind宏系统详解.md) —— 学习更高层次的 ncbind 封装，它用 C++ 模板元编程大幅简化了原生类的注册过程，让你只需几行代码就能将 C++ 类暴露给 TJS2。

