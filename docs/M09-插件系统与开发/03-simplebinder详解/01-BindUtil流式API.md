# BindUtil 流式 API

> **所属模块：** M09-插件系统与开发
> **前置知识：** [ncbind 类型系统与转换](../02-ncbind绑定框架/01-类型系统与转换.md)、[ncbind 类注册与宏体系](../02-ncbind绑定框架/02-类注册与宏体系.md)
> **预计阅读时间：** 25 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：
1. 理解 simplebinder 的设计动机——为什么在 ncbind 之外还需要另一套绑定框架
2. 掌握 `BindUtil` 流式 API（Fluent API）的完整用法：注册函数、属性、常量、类
3. 理解 simplebinder 内部的存储体系：`StoreUtil`、`FunctionStore`、`PropertyStore`、`ConstantStore`、`ClassStore`
4. 对比 simplebinder 与 ncbind 在注册流程上的核心差异
5. 使用 simplebinder 为一个完整的 C++ 类编写 TJS2 绑定

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 流式 API | Fluent API | 一种编程风格，每个方法都返回 `this` 引用，允许链式调用如 `obj.A().B().C()` |
| BindUtil | BindUtil | simplebinder 的核心入口类，提供 `Function()`、`Property()`、`Class()` 等链式注册方法 |
| StoreUtil | StoreUtil | simplebinder 的基础存储类，封装了 `iTJSDispatch2` 的注册/卸载操作 |
| FunctionStore | FunctionStore | 函数绑定的存储类，将 C++ 函数/成员函数包装为 TJS2 可调用的 `iTJSDispatch2` 对象 |
| PropertyStore | PropertyStore | 属性绑定的存储类，将 getter/setter 函数包装为 TJS2 属性 |
| ConstantStore | ConstantStore | 常量绑定的存储类，创建一个只读的 TJS2 属性，写操作返回 `TJS_E_ACCESSDENYED` |
| ClassStore | ClassStore | 类绑定的存储类，管理类的构造函数工厂、实例包装器和类对象的完整生命周期 |
| InstanceWrapper | InstanceWrapper | 将 C++ 原生实例包装为 `tTJSNativeInstance`，负责构造和析构的生命周期管理 |

---

## 一、simplebinder 的设计动机

### 1.1 为什么需要另一套绑定框架？

在前两节中，我们深入分析了 ncbind 框架。ncbind 功能强大，支持方法、属性、构造函数、RawCallback、Proxy、Bridge 等多种注册模式。但它有一个显著特点：**重度依赖宏**。

回忆一下 ncbind 的典型用法：

```cpp
// ncbind 风格：宏驱动
NCB_REGISTER_CLASS(MyClass) {
    NCB_CONSTRUCTOR((int, const ttstr&));
    NCB_METHOD(doWork);
    NCB_PROPERTY(name, getName, setName);
    NCB_PROPERTY_RO(count, getCount);
}
```

这种宏风格有几个问题：

1. **调试困难**：宏展开后的代码难以在 IDE 中追踪，编译错误信息指向宏内部而非用户代码
2. **灵活性受限**：要注册全局函数（不属于任何类的函数），ncbind 需要使用 `NCB_ATTACH_FUNCTION` 等专门宏
3. **运行时注册不便**：ncbind 的注册块是静态的（在 `ncbClassRegist` 构造函数中执行），不适合需要在运行时动态注册/卸载的场景

simplebinder 采用了完全不同的设计思路：

```cpp
// simplebinder 风格：流式 API
SimpleBinder::BindUtil bind(true);  // true = 注册模式
bind.Function(TJS_W("myGlobalFunc"), &myGlobalFunc)
    .Function(TJS_W("anotherFunc"), &anotherFunc)
    .Constant(TJS_W("VERSION"), 42)
    .Property(TJS_W("debugMode"), &getDebugMode, &setDebugMode);
```

### 1.2 两种设计哲学对比

| 维度 | ncbind | simplebinder |
|------|--------|--------------|
| 注册时机 | 静态（程序启动时自动执行） | 手动（由 `NCB_PRE_REGIST_CALLBACK` 或任意代码触发） |
| 风格 | 宏驱动（`NCB_METHOD`、`NCB_PROPERTY` 等） | 流式 API（`bind.Function().Property().Class()`） |
| 类型推导 | 通过模板宏展开（`TypeWrap<>`） | 通过函数模板参数推导 |
| 卸载支持 | 有（`NCB_POST_UNREGIST_CALLBACK`） | 原生支持（构造时传 `false` 即卸载模式） |
| 全局函数 | 需要 `NCB_ATTACH_FUNCTION` | 直接 `bind.Function()` |
| 复杂度 | 高（30+ 宏，2382 行头文件） | 低（1237 行头文件，主要是模板展开辅助） |
| 项目中使用情况 | **所有插件均使用** | **仅提供，无实际使用** |

> **重要发现：** 在 KrKr2 项目中，simplebinder 虽然作为头文件提供（`cpp/plugins/simplebinder/simplebinder.hpp`，1237 行），但**目前没有任何插件实际使用它**。所有插件（scriptsEx、csvParser、windowEx、psdfile 等）都使用 ncbind 进行绑定。simplebinder 是原始 KiriKiri 社区提供的一个替代方案，被保留在代码库中以备将来使用。

### 1.3 simplebinder 的核心架构

simplebinder 的整体架构比 ncbind 简洁得多，可以用以下层次图表示：

```
┌─────────────────────────────────────────────┐
│               BindUtil (入口)                 │
│  .Function() .Property() .Class()           │
│  .Constant() .Variant() .SetContext()       │
└────────┬──────────┬───────────┬─────────────┘
         │          │           │
    ┌────▼───┐ ┌───▼────┐ ┌───▼──────┐
    │Function│ │Property│ │  Class   │
    │ Store  │ │ Store  │ │  Store   │
    └────┬───┘ └───┬────┘ └───┬──────┘
         │          │           │
    ┌────▼───┐ ┌───▼────┐ ┌───▼──────┐
    │Function│ │Setter/ │ │ Instance │
    │Callback│ │Getter  │ │ Wrapper  │
    │        │ │Callback│ │          │
    └────┬───┘ └───┬────┘ └───┬──────┘
         │          │           │
         └────┬─────┘           │
              │                 │
    ┌─────────▼─────────────────▼─────────┐
    │         StoreUtil (基础层)            │
    │   Link() / Unlink() 到 TJS2 对象     │
    └─────────────────────────────────────┘
```

所有的 `*Store` 类都继承自 `StoreUtil`，而 `StoreUtil` 自身是 `tTJSDispatch` 的子类（即一个合法的 TJS2 对象）。这意味着 simplebinder 注册的每一个函数/属性/常量，本身就是一个 TJS2 分发器（Dispatch Object，TJS2 中所有值的底层抽象，负责接收 `FuncCall`、`PropGet`、`PropSet` 等操作）。

---

## 二、BindUtil 类详解

### 2.1 构造函数——注册模式与卸载模式

`BindUtil` 有四个构造函数重载（源码 `simplebinder.hpp` 第 1053-1070 行）：

```cpp
// 重载 1：全局模式——将函数/属性注册到 TJS2 全局对象
BindUtil(bool link);
// link=true  → 注册模式：调用 .Function()/.Property() 会注册到全局
// link=false → 卸载模式：调用 .Function()/.Property() 会从全局删除

// 重载 2：指定命名空间——注册到指定路径的对象上
BindUtil(const ttstr& base, bool link);
// base = "System" → 注册到 System 对象上
// base = "KAG.Utils" → 注册到 KAG.Utils 对象上（支持 "." 分隔的层级路径）

// 重载 3：指定命名空间 + 自定义根对象
BindUtil(const ttstr& base, iTJSDispatch2* root, bool link);
// 从 root 对象开始查找 base 路径

// 重载 4：直接指定目标对象
BindUtil(iTJSDispatch2* store, bool link);
// 直接操作指定的 TJS2 分发器对象
```

最常用的是重载 1 和重载 2：

```cpp
// 示例 1：注册全局函数
SimpleBinder::BindUtil bind(true);  // 注册到全局
bind.Function(TJS_W("myFunc"), &myFunc);

// 示例 2：注册到 System 命名空间
SimpleBinder::BindUtil bind(TJS_W("System"), true);
bind.Function(TJS_W("getVersion"), &getVersion);
// TJS2 中调用：System.getVersion()

// 示例 3：插件卸载时清理
SimpleBinder::BindUtil unbind(false);  // 卸载模式
unbind.Function(TJS_W("myFunc"), &myFunc);
// myFunc 从全局对象中移除
```

> **注册/卸载对称设计**：这是 simplebinder 的一个精巧之处。同样的链式调用代码，只需将构造函数的 `link` 参数从 `true` 改为 `false`，就能从"注册"切换到"卸载"。ncbind 需要单独的 `NCB_POST_UNREGIST_CALLBACK` 回调来实现卸载，而 simplebinder 用一个布尔参数就解决了。

### 2.2 Function() —— 函数注册

`Function()` 方法将一个 C++ 函数绑定为 TJS2 可调用的函数：

```cpp
// simplebinder.hpp 第 1074-1082 行
template <typename FUNC>
BindUtil& Function(const ttstr& key, const FUNC& func) {
    if (_store)
        _error |= !(_link 
            ? FunctionStore::Link(_store, key, func, _context)
            : StoreUtil::Unlink(_store, key));
    return *this;  // 返回自身引用，实现链式调用
}
```

`FUNC` 可以是以下任意类型：
- **普通函数指针**：`tjs_error (*)(tTJSVariant*, tjs_int, tTJSVariant**, iTJSDispatch2*)`
- **成员函数指针**：`ReturnType (Class::*)(Args...)`
- **自由函数**：`ReturnType (*)(Args...)`

simplebinder 通过 `MethodInvoker<FUNC>` 模板来推导参数数量和类型，自动生成参数提取和返回值转换的代码。

完整示例——注册全局函数：

```cpp
#include <simplebinder/simplebinder.hpp>

// 普通 C++ 函数
static int addNumbers(int a, int b) {
    return a + b;
}

// 带字符串参数的函数
static ttstr greet(const ttstr& name) {
    return ttstr(TJS_W("你好，")) + name + TJS_W("！");
}

// 原始 TJS 签名的函数（手动处理参数）
static tjs_error TJS_INTF_METHOD rawFunc(
    tTJSVariant* result, 
    tjs_int numparams,
    tTJSVariant** param, 
    iTJSDispatch2* objthis) 
{
    // 手动处理可变参数
    if (result) {
        *result = tTJSVariant(numparams);  // 返回参数个数
    }
    return TJS_S_OK;
}

// 在插件注册回调中使用
static void preRegistCallback() {
    SimpleBinder::BindUtil bind(true);
    
    bind.Function(TJS_W("addNumbers"), &addNumbers)
        .Function(TJS_W("greet"), &greet)
        .Function(TJS_W("rawFunc"), &rawFunc);
    
    // 检查是否全部注册成功
    if (!bind.IsValid()) {
        TVPAddLog(TJS_W("函数注册失败！"));
    }
}

// 在插件卸载回调中清理
static void postUnregistCallback() {
    SimpleBinder::BindUtil unbind(false);
    
    unbind.Function(TJS_W("addNumbers"), &addNumbers)
          .Function(TJS_W("greet"), &greet)
          .Function(TJS_W("rawFunc"), &rawFunc);
}
```

对应的 TJS2 脚本使用：

```javascript
// 调用注册的全局函数
var sum = addNumbers(3, 7);         // sum == 10
var msg = greet("开发者");           // msg == "你好，开发者！"
var count = rawFunc(1, 2, 3, 4, 5); // count == 5
```

### 2.3 Property() —— 属性注册

`Property()` 方法将 getter 和 setter 函数绑定为 TJS2 属性：

```cpp
// simplebinder.hpp 第 1090-1099 行
template <typename GET, typename SET>
BindUtil& Property(const ttstr& key, const GET& get, const SET& set) {
    if (_store)
        _error |= !(_link 
            ? PropertyStore::Link(_store, key, set, get, _context)
            : StoreUtil::Unlink(_store, key));
    return *this;
}
```

> **注意参数顺序**：`Property()` 的公开接口是 `(key, get, set)`，但内部调用 `PropertyStore::Link()` 时是 `(store, key, set, get)`。这是 simplebinder 源码中的一个容易混淆的细节——`PropertyStore` 构造函数的参数顺序是 setter 在前、getter 在后。

getter 和 setter 函数支持多种签名：

```cpp
// === getter 签名（以下任选其一）===

// 方式 1：全局 getter（不需要实例）
tjs_error myGetter(tTJSVariant* result);

// 方式 2：全局 getter（带 objthis）
tjs_error myGetter(iTJSDispatch2* objthis, tTJSVariant* result);

// 方式 3：成员函数 getter（const）
tjs_error MyClass::myGetter(tTJSVariant* result) const;

// 方式 4：成员函数 getter（非 const）
tjs_error MyClass::myGetter(tTJSVariant* result);

// 方式 5：自由函数 getter（接受 const 实例指针）
tjs_error myGetter(const MyClass* self, tTJSVariant* result);

// 方式 6：自由函数 getter（接受非 const 实例指针）
tjs_error myGetter(MyClass* self, tTJSVariant* result);


// === setter 签名（以下任选其一）===

// 方式 1：全局 setter
tjs_error mySetter(const tTJSVariant* param);

// 方式 2：全局 setter（带 objthis）
tjs_error mySetter(iTJSDispatch2* objthis, const tTJSVariant* param);

// 方式 3：成员函数 setter
tjs_error MyClass::mySetter(const tTJSVariant* param);

// 方式 4：自由函数 setter（接受实例指针）
tjs_error mySetter(MyClass* self, const tTJSVariant* param);
```

只读属性传 `0` 作为 setter，只写属性传 `0` 作为 getter：

```cpp
// 只读属性（setter 传 0）
bind.Property(TJS_W("readOnly"), &myGetter, 0);

// 只写属性（getter 传 0）
bind.Property(TJS_W("writeOnly"), 0, &mySetter);
```

### 2.4 Constant() —— 常量注册

`Constant()` 方法注册一个只读常量（底层是 `ConstantStore`，写操作返回 `TJS_E_ACCESSDENYED`）：

```cpp
// simplebinder.hpp 第 1106-1113 行
template <typename T>
BindUtil& Constant(const ttstr& key, const T& value) {
    if (_store)
        _error |= !(_link 
            ? ConstantStore::Link(_store, key, value)
            : StoreUtil::Unlink(_store, key));
    return *this;
}
```

`T` 可以是任何能转换为 `tTJSVariant` 的类型（`int`、`double`、`ttstr`、`bool` 等）：

```cpp
SimpleBinder::BindUtil bind(true);
bind.Constant(TJS_W("PI"), 3.14159265358979)     // double 常量
    .Constant(TJS_W("APP_NAME"), TJS_W("MyApp"))  // 字符串常量
    .Constant(TJS_W("MAX_LAYERS"), 256)            // int 常量
    .Constant(TJS_W("DEBUG"), true);               // bool 常量
```

### 2.5 Variant() —— 变量注册

`Variant()` 与 `Constant()` 不同，它直接通过 `PropSet` 将值设置到目标对象上，支持自定义标志位：

```cpp
// simplebinder.hpp 第 1120-1133 行
template <typename T>
BindUtil& Variant(const ttstr& key, const T& value,
                  tjs_uint32 flag = StoreUtil::DEFAULT_FLAG) {
    if (_store) {
        if (_link) {
            tTJSVariant v(value);
            _error |= TJS_FAILED(
                _store->PropSet(flag, key.c_str(), NULL, &v, _store));
        } else {
            _error |= !StoreUtil::Unlink(_store, key);
        }
    }
    return *this;
}
```

`Variant()` 和 `Constant()` 的区别：

| 特性 | Constant() | Variant() |
|------|-----------|-----------|
| 只读 | ✅ 写操作返回 `TJS_E_ACCESSDENYED` | ❌ 值可被 TJS2 脚本覆盖 |
| 底层实现 | `ConstantStore`（自定义分发器） | 直接 `PropSet` 到目标对象 |
| 标志位 | 固定 `DEFAULT_FLAG` | 可自定义（`TJS_HIDDENMEMBER` 等） |
| 适用场景 | 真正的常量（不应被修改） | 初始值（可被脚本修改） |

### 2.6 Class() —— 类注册

`Class()` 是 simplebinder 最强大的方法，用于将一个完整的 C++ 类绑定到 TJS2：

```cpp
// simplebinder.hpp 第 1139-1151 行
template <typename CTOR>
BindUtil& Class(const ttstr& key, const CTOR& ctor) {
    if (_store) {
        typedef typename FactoryInvoker<CTOR>::Class TargetClass;
        iTJSDispatch2* rebase = 0;
        _error |= !(_link 
            ? ClassStore<TargetClass>::Link(_store, key, ctor, &rebase)
            : StoreUtil::Unlink(_store, key));
        _store = rebase;  // 关键：切换到类对象，后续注册进入类内部
    }
    return *this;
}
```

> **`_store = rebase` 的精妙设计**：调用 `Class()` 后，`BindUtil` 的内部存储对象（`_store`）切换为新创建的 TJS2 类对象。这意味着后续的 `.Function()`、`.Property()` 调用会自动注册到该类内部，而非全局对象上。这就是 simplebinder 实现"类内成员注册"的方式——无需宏块，纯粹通过链式调用上下文切换。

完整的类注册示例：

```cpp
#include <simplebinder/simplebinder.hpp>

// C++ 类定义
class Calculator {
public:
    int value_;
    
    // 工厂函数（simplebinder 的构造方式）
    // 返回 Calculator*，参数从 TJS2 传入
    static Calculator* Create(int initial) {
        return new Calculator{initial};
    }
    
    int add(int x) { 
        value_ += x; 
        return value_; 
    }
    
    int getValue() const { return value_; }
    void setValue(int v) { value_ = v; }
};

// getter/setter 需要手动编写（simplebinder 不像 ncbind 那样直接绑定成员函数）
static tjs_error calcGetValue(const Calculator* self, tTJSVariant* result) {
    if (result) *result = self->getValue();
    return TJS_S_OK;
}

static tjs_error calcSetValue(Calculator* self, const tTJSVariant* param) {
    self->setValue(static_cast<int>(*param));
    return TJS_S_OK;
}

// 注册
static void preRegistCallback() {
    SimpleBinder::BindUtil bind(true);
    
    bind.Class(TJS_W("Calculator"), &Calculator::Create)
        // 以下注册自动进入 Calculator 类内部
        .Function(TJS_W("add"), &Calculator::add)
        .Property(TJS_W("value"), &calcGetValue, &calcSetValue);
    
    if (!bind.IsValid()) {
        TVPAddLog(TJS_W("Calculator 注册失败"));
    }
}
```

TJS2 脚本使用：

```javascript
var calc = new Calculator(10);      // 调用 Calculator::Create(10)
calc.add(5);                        // calc.value_ == 15
System.inform(calc.value);          // 输出 15
calc.value = 100;                   // 调用 setValue(100)
calc.add(1);                        // calc.value_ == 101
```

### 2.7 SetContext() —— 上下文切换

`SetContext()` 用于改变后续 `Function()` / `Property()` 注册的上下文对象：

```cpp
// simplebinder.hpp 第 1177-1180 行
BindUtil& SetContext(iTJSDispatch2* context = NULL) {
    _context = context;
    return *this;
}
```

上下文（Context）在 TJS2 中决定了方法调用时的 `this` 指向。传 `NULL` 使用默认上下文：

```cpp
// 示例：为某个特定对象设置上下文
SimpleBinder::BindUtil bind(true);
iTJSDispatch2* myObj = /* 某个 TJS2 对象 */;

bind.SetContext(myObj)
    .Function(TJS_W("method1"), &func1)  // 调用时 this 指向 myObj
    .SetContext(NULL)                      // 恢复默认上下文
    .Function(TJS_W("method2"), &func2); // 调用时 this 指向调用者
```

### 2.8 IsValid() —— 错误检查

`IsValid()` 返回一个布尔值，指示到目前为止的所有注册/卸载操作是否全部成功：

```cpp
bool IsValid() const { return !_error; }
```

`_error` 是一个位或（`|=`）累积标志。一旦任何操作失败，`_error` 变为 `true`，`IsValid()` 返回 `false`。这种设计允许你在一长串链式调用结束后只检查一次：

```cpp
SimpleBinder::BindUtil bind(true);
bind.Function(TJS_W("func1"), &func1)
    .Function(TJS_W("func2"), &func2)
    .Function(TJS_W("func3"), &func3)
    .Constant(TJS_W("VER"), 1);

if (!bind.IsValid()) {
    // 至少有一个注册失败
    TVPAddLog(TJS_W("注册过程中发生错误！"));
}
```

---

## 三、存储体系深度解析

### 3.1 StoreUtil —— 基础层

`StoreUtil` 是所有存储类的基类，继承自 `tTJSDispatch`（TJS2 分发器的基类）：

```cpp
// simplebinder.hpp 第 394-503 行（简化）
class StoreUtil : public tTJSDispatch {
protected:
    tjs_uint32 _flag;    // 注册标志位
    
public:
    static const tjs_uint32 DEFAULT_FLAG = 
        TJS_MEMBERENSURE;  // 如果成员不存在则创建
    
    // 将 store 对象链接到 obj 的指定 key 上
    static bool Link(iTJSDispatch2* obj, const ttstr& key, 
                     StoreUtil* store, 
                     iTJSDispatch2* context = NULL) {
        tTJSVariant val(store, store);  // 将 store 包装为 Variant
        store->Release();               // 释放多余引用
        // 设置上下文
        if (context) val.ChangeClosureObjThis(context);
        // 注册到目标对象
        return obj && TJS_SUCCEEDED(
            obj->PropSet(store->_flag, key.c_str(), 
                        NULL, &val, obj));
    }
    
    // 从 obj 中移除指定 key
    static bool Unlink(iTJSDispatch2* obj, const ttstr& key) {
        return obj && TJS_SUCCEEDED(
            obj->DeleteMember(0, key.c_str(), NULL, obj));
    }
    
    // 通过 "." 分隔的路径查找对象
    static iTJSDispatch2* GetObject(const ttstr& base, 
                                    iTJSDispatch2* root = 0);
};
```

`StoreUtil::Link()` 是整个 simplebinder 注册机制的核心。它将一个 `StoreUtil` 子类实例（即一个 TJS2 分发器对象）设置为目标对象的成员。当 TJS2 脚本访问该成员时，TJS2 引擎会调用分发器的 `FuncCall` / `PropGet` / `PropSet` 方法，从而触发 C++ 回调。

### 3.2 FunctionStore —— 函数封装

`FunctionStore`（源码第 538-578 行）将 C++ 函数包装为可被 TJS2 调用的分发器：

```cpp
class FunctionStore : public StoreUtil {
    FunctionInterface* _function;  // 持有函数回调接口
    
public:
    template <typename T>
    FunctionStore(const T& func, bool autoStatic = false,
                  tjs_uint32 flag = StoreUtil::DEFAULT_FLAG)
        : StoreUtil(flag), _function(0) 
    {
        // 如果函数是静态的（不需要实例），自动添加 TJS_STATICMEMBER 标志
        if (autoStatic && MethodInvoker<T>::Static)
            _flag |= TJS_STATICMEMBER;
        _function = new FunctionCallback<T>(func);
    }
    
    // TJS2 调用入口
    tjs_error FuncCall(tjs_uint32 flag, const tjs_char* membername,
                       tjs_uint32* hint, tTJSVariant* result,
                       tjs_int numparams, tTJSVariant** param,
                       iTJSDispatch2* objthis) {
        if (membername) return TJS_E_MEMBERNOTFOUND;
        return _function->invoke(flag, result, numparams, param, objthis);
    }
};
```

`FunctionCallback<T>` 是模板类，它通过 `MethodInvoker<T>` 来执行实际的函数调用。`MethodInvoker` 根据函数签名自动推导参数数量（`Count`），并在调用前检查参数数量是否足够（`numparams < _minparams` → `TJS_E_BADPARAMCOUNT`）。

### 3.3 PropertyStore —— 属性封装

`PropertyStore`（源码第 658-726 行）封装 getter 和 setter：

```cpp
class PropertyStore : public StoreUtil {
    SetterInterface* _setter;  // setter 回调（可为 nullptr）
    GetterInterface* _getter;  // getter 回调（可为 nullptr）
    
public:
    // 读写属性
    template <typename SET, typename GET>
    PropertyStore(const SET& set, const GET& get) 
        : _setter(new SetterCallback<SET>(set)),
          _getter(new GetterCallback<GET>(get)) {}
    
    // 只写属性（getter_only 参数仅用于重载区分）
    template <typename SET>
    PropertyStore(const SET& set, int setter_only = 0)
        : _setter(new SetterCallback<SET>(set)), _getter(0) {}
    
    // 只读属性
    template <typename GET>
    PropertyStore(int getter_only, const GET& get)
        : _setter(0), _getter(new GetterCallback<GET>(get)) {}
    
    // TJS2 写入入口
    tjs_error PropSet(...) {
        return _setter 
            ? _setter->invoke(flag, param, objthis)
            : TJS_E_ACCESSDENYED;  // 无 setter → 拒绝写入
    }
    
    // TJS2 读取入口
    tjs_error PropGet(...) {
        return _getter 
            ? _getter->invoke(flag, result, objthis)
            : TJS_E_ACCESSDENYED;  // 无 getter → 拒绝读取
    }
};
```

注意这里的重载区分方式：`PropertyStore(const SET& set, int)` 用一个 `int` 哑参数来区分"只写"和"读写"构造函数，这与 ncbind 用 `(int)0` 做哨兵值的思路异曲同工。

### 3.4 ConstantStore —— 常量封装

`ConstantStore`（源码第 731-771 行）是最简单的存储类：

```cpp
class ConstantStore : public StoreUtil {
    tTJSVariant _value;  // 存储常量值
    
public:
    template <typename T>
    ConstantStore(const T& value) : _value(value) {}
    
    // 写操作一律拒绝
    tjs_error PropSet(...) {
        return membername ? TJS_E_MEMBERNOTFOUND : TJS_E_ACCESSDENYED;
    }
    
    // 读操作返回存储的值
    tjs_error PropGet(...) {
        if (membername) return TJS_E_MEMBERNOTFOUND;
        if (result) *result = _value;
        return TJS_S_OK;
    }
};
```

### 3.5 ClassStore —— 类管理

`ClassStore<C>`（源码第 920-1033 行）是最复杂的存储类，负责：
1. 创建 TJS2 原生类对象（`tTJSNativeClassForPlugin`）
2. 注册类 ID
3. 管理 `InstanceWrapper<C>` 的构造和析构
4. 通过单例模式（`_Singleton`）确保每个 C++ 类只注册一次

核心流程：

```
ClassStore::Link()
    │
    ├─ CreateClassObject(name, entry)
    │       │
    │       ├─ TJSCreateNativeClassForPlugin()  // 创建 TJS2 类对象
    │       ├─ TJSRegisterNativeClass()         // 注册类 ID
    │       ├─ new ClassStore(id, entry)         // 创建单例
    │       ├─ Wrapper::SetupClassObject()       // 注册 finalize + 构造函数
    │       └─ entry->entryMembers(classobj)     // 回调：注册成员
    │
    └─ rebase = classobj  // 返回类对象，BindUtil 用它接收后续注册
```

---

## 四、MethodInvoker —— 参数自动提取

simplebinder 的参数提取机制与 ncbind 的 `paramsFunctor` 类似，但实现更简洁。它通过宏展开生成 0-8 个参数的特化版本：

```cpp
// simplebinder.hpp 第 120-200 行（简化示意）
// 对于参数数量 N = 0, 1, 2, ..., 8，
// 宏 UNROLL 生成对应的 MethodInvoker 特化

template <typename CB>
struct MethodInvoker;

// 0 参数特化（手动展示）
template <typename R>
struct MethodInvoker<R(*)()> {
    static const int Count = 0;      // 需要 0 个 TJS2 参数
    static const bool Static = true; // 这是静态函数
    
    static tjs_error Invoke(R(*func)(), tTJSVariant* result,
                           tjs_int numparams, tTJSVariant** param,
                           iTJSDispatch2* objthis) {
        if (result) *result = func();  // 调用并转换返回值
        return TJS_S_OK;
    }
};

// 2 参数特化示意
template <typename R, typename A0, typename A1>
struct MethodInvoker<R(*)(A0, A1)> {
    static const int Count = 2;
    static const bool Static = true;
    
    static tjs_error Invoke(R(*func)(A0, A1), tTJSVariant* result,
                           tjs_int numparams, tTJSVariant** param,
                           iTJSDispatch2* objthis) {
        // 从 param[0]、param[1] 提取参数并转换类型
        if (result) *result = func(
            static_cast<A0>(*param[0]),
            static_cast<A1>(*param[1])
        );
        return TJS_S_OK;
    }
};
```

实际实现中，这些特化通过 `UNROLL_FOREACH` 宏批量生成（源码第 72-83 行），避免手写 9 个版本的重复代码。

---

## 动手实践

### 实践：使用 simplebinder 注册一个完整的 StringHelper 工具类

以下示例展示如何用 simplebinder 的流式 API 注册一个字符串工具类：

```cpp
#include <simplebinder/simplebinder.hpp>
#include <algorithm>

// ========================
// C++ 类定义
// ========================
class StringHelper {
public:
    ttstr text_;
    
    // 工厂函数（simplebinder 要求）
    static StringHelper* Create(const ttstr& initial) {
        auto* obj = new StringHelper();
        obj->text_ = initial;
        return obj;
    }
    
    // 成员方法
    int length() const { 
        return static_cast<int>(text_.length()); 
    }
    
    ttstr toUpper() const {
        ttstr result = text_;
        // 逐字符转大写（简化，仅处理 ASCII）
        for (tjs_int i = 0; i < result.length(); i++) {
            tjs_char ch = result.c_str()[i];
            if (ch >= TJS_W('a') && ch <= TJS_W('z')) {
                // 注意：ttstr 不支持直接修改，需要用其他方式
                // 这里简化处理
            }
        }
        return result;
    }
    
    bool contains(const ttstr& sub) const {
        return text_.c_str() != nullptr && 
               wcsstr(text_.c_str(), sub.c_str()) != nullptr;
    }
    
    ttstr repeat(int count) const {
        ttstr result;
        for (int i = 0; i < count; i++) {
            result += text_;
        }
        return result;
    }
};

// ========================
// Getter/Setter 函数
// ========================
static tjs_error getText(const StringHelper* self, tTJSVariant* result) {
    if (result) *result = self->text_;
    return TJS_S_OK;
}

static tjs_error setText(StringHelper* self, const tTJSVariant* param) {
    self->text_ = param->AsStringNoAddRef();
    return TJS_S_OK;
}

static tjs_error getLength(const StringHelper* self, tTJSVariant* result) {
    if (result) *result = self->length();
    return TJS_S_OK;
}

// ========================
// 全局辅助函数
// ========================
static ttstr concatStrings(const ttstr& a, const ttstr& b) {
    return a + b;
}

// ========================
// 注册函数
// ========================
static void preRegistCallback() {
    SimpleBinder::BindUtil bind(true);
    
    // 1. 注册全局辅助函数
    bind.Function(TJS_W("concatStrings"), &concatStrings);
    
    // 2. 注册 StringHelper 类
    //    注意：调用 Class() 后，后续的注册自动进入类内部
    bind.Class(TJS_W("StringHelper"), &StringHelper::Create)
        .Function(TJS_W("contains"), &StringHelper::contains)
        .Function(TJS_W("repeat"), &StringHelper::repeat)
        .Property(TJS_W("text"), &getText, &setText)
        .Property(TJS_W("length"), &getLength, 0);  // 只读属性
    
    if (!bind.IsValid()) {
        TVPAddLog(TJS_W("StringHelper 注册失败！"));
    }
}

static void postUnregistCallback() {
    SimpleBinder::BindUtil unbind(false);
    unbind.Function(TJS_W("concatStrings"), &concatStrings)
          .Class(TJS_W("StringHelper"), &StringHelper::Create);
}

// 使用 ncbind 的回调宏来触发 simplebinder 的注册
NCB_PRE_REGIST_CALLBACK(preRegistCallback);
NCB_POST_UNREGIST_CALLBACK(postUnregistCallback);
```

TJS2 脚本测试：

```javascript
// 全局函数
var s = concatStrings("Hello, ", "World!");
System.inform(s);  // "Hello, World!"

// StringHelper 类
var helper = new StringHelper("KiriKiri");
System.inform(helper.text);     // "KiriKiri"
System.inform(helper.length);   // 8
System.inform(helper.contains("Kiri")); // true
System.inform(helper.repeat(3)); // "KiriKiriKiriKiriKiriKiri"

helper.text = "新文本";
System.inform(helper.text);     // "新文本"
// helper.length = 5;  // 报错：TJS_E_ACCESSDENYED（只读属性）
```

---

## 对照项目源码

### simplebinder 源文件位置

- **`cpp/plugins/simplebinder/simplebinder.hpp`** 第 1-1237 行 —— simplebinder 的完整实现（头文件唯一）

### 关键代码段

| 源码位置 | 内容 |
|----------|------|
| 第 1-83 行 | `UNROLL_*` 宏定义——用于批量生成 0-8 参数的模板特化 |
| 第 88-109 行 | `MethodInvokerBase`——基础工具类（`VarNum`、`VarArgs`、`GetSelf`） |
| 第 394-503 行 | `StoreUtil`——基础存储类，`Link()` / `Unlink()` 核心逻辑 |
| 第 508-578 行 | `FunctionStore`——函数绑定的存储和分发 |
| 第 583-726 行 | `SetterCallback` / `GetterCallback` / `PropertyStore`——属性绑定 |
| 第 731-771 行 | `ConstantStore`——常量绑定（只读属性） |
| 第 776-837 行 | `ClassEntryInterface` / `ClassEntryCallback`——类的构造/析构工厂 |
| 第 840-917 行 | `InstanceWrapper<C>`——原生实例的 TJS2 适配器 |
| 第 920-1033 行 | `ClassStore<C>`——类注册管理（单例模式） |
| 第 1048-1211 行 | `BindUtil`——公开的流式 API 入口类 |
| 第 1218 行 | `typedef Detail::BindUtil BindUtil`——公开命名空间导出 |
| 第 1227-1236 行 | `SIMPLEBINDER_CUSTOM_CONTEXT_RESOLV` 宏——自定义实例解析 |

### 项目使用情况

经过全面搜索，**KrKr2 项目中目前没有任何 `.cpp` 文件使用 `SimpleBinder` 命名空间或 `BindUtil` 类**。所有现有插件（scriptsEx、csvParser、windowEx、psdfile、psbfile、layerExDraw、fstat 等）都使用 ncbind 框架进行 TJS2 绑定。

simplebinder 作为一个备用方案保留在代码库中，可能在以下场景中有用：
- 需要在运行时动态注册/卸载 TJS2 函数的场景
- 编写不依赖 ncbind 宏体系的新插件
- 需要更细粒度控制注册过程的场景

---

## 常见错误与排查

### 错误 1：Class() 之后的注册进入了全局而非类内部

**症状：** 注册的方法在 TJS2 中是全局函数，而非类的成员方法。

**原因：** `Class()` 内部通过 `_store = rebase` 切换存储对象。如果 `ClassStore::Link()` 失败（例如类名冲突），`rebase` 为 `nullptr`，后续注册操作会被静默跳过（因为 `_store` 为空时所有方法都直接 `return *this`）。

```cpp
// ❌ 类注册失败时，后续注册会被跳过
bind.Class(TJS_W("MyClass"), &MyClass::Create)  // 如果失败
    .Function(TJS_W("doWork"), &MyClass::doWork)  // 静默跳过
    .Property(TJS_W("name"), &getName, &setName);  // 静默跳过

// ✅ 检查 IsValid()
bind.Class(TJS_W("MyClass"), &MyClass::Create);
if (!bind.IsValid()) {
    TVPAddLog(TJS_W("MyClass 注册失败，请检查类名是否冲突"));
    return;
}
bind.Function(TJS_W("doWork"), &MyClass::doWork);
```

### 错误 2：Property() 参数顺序搞反

**症状：** 读属性时触发 setter 逻辑，写属性时触发 getter 逻辑。

**原因：** `BindUtil::Property()` 的公开接口是 `(key, getter, setter)`，但内部调用 `PropertyStore::Link()` 时参数顺序是 `(store, key, setter, getter)`。如果你直接使用 `PropertyStore::Link()` 而非 `BindUtil::Property()`，需要注意这个顺序差异。

```cpp
// ✅ 使用 BindUtil::Property()——参数顺序是 (key, getter, setter)
bind.Property(TJS_W("name"), &getName, &setName);

// ⚠️ 直接使用 PropertyStore::Link()——参数顺序是 (store, key, setter, getter)
PropertyStore::Link(store, TJS_W("name"), &setName, &getName);
```

### 错误 3：忘记卸载注册

**症状：** 插件卸载后，TJS2 脚本仍然能调用已注册的函数，但调用时崩溃（因为 C++ 代码已被卸载）。

**原因：** simplebinder 注册的函数/属性/类在插件卸载后不会自动清理。必须在 `NCB_POST_UNREGIST_CALLBACK` 中使用 `BindUtil(false)` 手动卸载。

```cpp
// ❌ 只注册不卸载
NCB_PRE_REGIST_CALLBACK(preRegist);
// 缺少 NCB_POST_UNREGIST_CALLBACK

// ✅ 注册和卸载成对出现
NCB_PRE_REGIST_CALLBACK(preRegist);
NCB_POST_UNREGIST_CALLBACK(postUnregist);
```

---

## 本节小结

- **simplebinder 是 KrKr2 项目中备用的 TJS2 绑定框架**，提供流式 API 风格（`BindUtil`），作为 ncbind 宏驱动风格的替代方案。目前项目中所有插件都使用 ncbind，simplebinder 未被实际使用
- **BindUtil 的核心设计**：通过构造函数的 `link` 参数控制注册/卸载模式，所有方法返回 `*this` 实现链式调用
- **五大注册方法**：`Function()`（函数）、`Property()`（属性）、`Constant()`（只读常量）、`Variant()`（可修改变量）、`Class()`（类）
- **Class() 的上下文切换**：调用 `Class()` 后，`_store` 自动切换为新创建的 TJS2 类对象，后续的 `Function()` / `Property()` 注册自动进入类内部
- **存储体系**：所有绑定对象（`FunctionStore`、`PropertyStore`、`ConstantStore`）都继承自 `StoreUtil`（即 `tTJSDispatch` 子类），本身就是合法的 TJS2 分发器对象
- **参数自动提取**：通过 `MethodInvoker<FUNC>` 模板推导参数数量和类型，支持 0-8 个参数

---

## 练习题与答案

### 题目 1：使用 simplebinder 注册全局函数和常量

请使用 `SimpleBinder::BindUtil` 完成以下注册：
1. 全局函数 `clamp(int value, int min, int max)` —— 将值限制在 [min, max] 范围内
2. 常量 `INT_MAX_VALUE` = 2147483647
3. 常量 `INT_MIN_VALUE` = -2147483648
4. 编写对应的卸载函数

<details>
<summary>查看答案</summary>

```cpp
#include <simplebinder/simplebinder.hpp>

// C++ 实现
static int clamp(int value, int minVal, int maxVal) {
    if (value < minVal) return minVal;
    if (value > maxVal) return maxVal;
    return value;
}

// 注册
static void preRegist() {
    SimpleBinder::BindUtil bind(true);
    bind.Function(TJS_W("clamp"), &clamp)
        .Constant(TJS_W("INT_MAX_VALUE"), 2147483647)
        .Constant(TJS_W("INT_MIN_VALUE"), -2147483648);
    
    if (!bind.IsValid()) {
        TVPAddLog(TJS_W("clamp 注册失败"));
    }
}

// 卸载——同样的调用，link=false
static void postUnregist() {
    SimpleBinder::BindUtil unbind(false);
    unbind.Function(TJS_W("clamp"), &clamp)
          .Constant(TJS_W("INT_MAX_VALUE"), 2147483647)
          .Constant(TJS_W("INT_MIN_VALUE"), -2147483648);
}

NCB_PRE_REGIST_CALLBACK(preRegist);
NCB_POST_UNREGIST_CALLBACK(postUnregist);
```

TJS2 测试：

```javascript
System.inform(clamp(150, 0, 100));   // 输出 100
System.inform(clamp(-5, 0, 100));    // 输出 0
System.inform(clamp(50, 0, 100));    // 输出 50
System.inform(INT_MAX_VALUE);        // 输出 2147483647
// INT_MAX_VALUE = 0;  // 报错：TJS_E_ACCESSDENYED
```

</details>

### 题目 2：解释 Class() 之后为什么 Function() 注册到类内部

请详细说明以下代码中，`doWork` 方法是如何被注册到 `MyClass` 类内部（而非全局对象）的。追踪 `_store` 变量的变化。

```cpp
SimpleBinder::BindUtil bind(true);
bind.Class(TJS_W("MyClass"), &MyClass::Create)
    .Function(TJS_W("doWork"), &MyClass::doWork);
```

<details>
<summary>查看答案</summary>

**`_store` 变量的变化过程：**

**第 1 步：`BindUtil(true)` 构造**

```cpp
BindUtil(bool link) : _link(true), _error(false), _store(0), _context(0) {
    _store = TVPGetScriptDispatch();  // _store = TJS2 全局对象
}
```

此时 `_store` 指向 **TJS2 全局对象**。

**第 2 步：`.Class(TJS_W("MyClass"), &MyClass::Create)`**

```cpp
BindUtil& Class(const ttstr& key, const CTOR& ctor) {
    iTJSDispatch2* rebase = 0;
    _error |= !ClassStore<MyClass>::Link(
        _store,    // 全局对象
        key,       // "MyClass"  
        ctor,      // &MyClass::Create
        &rebase    // 输出参数：新创建的类对象
    );
    _store = rebase;  // ★ 关键：_store 从全局对象变成 MyClass 类对象
    return *this;
}
```

`ClassStore::Link()` 内部调用 `CreateClassObject()`，它：
1. 调用 `TJSCreateNativeClassForPlugin("MyClass")` 创建 TJS2 类对象
2. 将类对象注册到全局（原先的 `_store`）
3. 通过 `rebase` 输出参数返回类对象的指针

最后 `_store = rebase`，**`_store` 从全局对象切换为 MyClass 的类对象**。

**第 3 步：`.Function(TJS_W("doWork"), &MyClass::doWork)`**

```cpp
BindUtil& Function(const ttstr& key, const FUNC& func) {
    if (_store)  // _store 现在是 MyClass 类对象
        _error |= !FunctionStore::Link(
            _store,  // MyClass 类对象（不是全局！）
            key,     // "doWork"
            func     // &MyClass::doWork
        );
    return *this;
}
```

因为 `_store` 已经被 `Class()` 切换为 MyClass 的类对象，所以 `FunctionStore::Link()` 将 `doWork` 注册到 **MyClass 类内部**，而非全局对象。

**总结：** `Class()` 方法通过修改 `_store` 成员变量实现了隐式的上下文切换，使得后续的 `Function()` / `Property()` 调用自动注册到类内部。这是 simplebinder 流式 API 的核心设计技巧。

</details>

### 题目 3：Constant() 与 Variant() 的行为差异

以下两段代码注册了同名的 `VERSION`，但行为不同。请说明 TJS2 脚本中 `VERSION = 2` 操作的结果分别是什么，并解释原因。

```cpp
// 方案 A
bind.Constant(TJS_W("VERSION"), 1);

// 方案 B
bind.Variant(TJS_W("VERSION"), 1);
```

<details>
<summary>查看答案</summary>

**方案 A：`Constant()`**

```javascript
System.inform(VERSION);  // 输出 1
VERSION = 2;             // 报错：TJS_E_ACCESSDENYED
System.inform(VERSION);  // 仍然输出 1
```

原因：`Constant()` 创建一个 `ConstantStore` 对象。`ConstantStore` 的 `PropSet()` 始终返回 `TJS_E_ACCESSDENYED`，因此写操作被拒绝。值永远是初始值 `1`。

**方案 B：`Variant()`**

```javascript
System.inform(VERSION);  // 输出 1
VERSION = 2;             // 成功！
System.inform(VERSION);  // 输出 2
```

原因：`Variant()` 直接通过 `_store->PropSet()` 将 `tTJSVariant(1)` 设置到目标对象上。它不创建自定义分发器，而是使用目标对象的默认属性存储。因此 TJS2 脚本可以像修改普通变量一样修改这个值。

**核心区别：**
- `Constant()` → 创建 `ConstantStore`（自定义只读分发器）→ 写操作被拦截
- `Variant()` → 直接 `PropSet` 到目标对象 → 使用目标对象的默认可写属性机制

</details>

---

## 下一步

本节介绍了 simplebinder 的流式 API 设计和完整用法。下一节我们将对 simplebinder 和 ncbind 进行系统性的对比分析，帮助你在编写新插件时做出正确的框架选择：

→ [与 ncbind 对比分析](02-与ncbind对比分析.md)
