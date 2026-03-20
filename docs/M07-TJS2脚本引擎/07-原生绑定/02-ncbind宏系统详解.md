# ncbind 宏系统详解

> **所属模块：** M07-TJS2脚本引擎
> **前置知识：** [tjsNative与原生类基础](./01-tjsNative与原生类基础.md)
> **预计阅读时间：** 50 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 ncbind 相对于底层 TJS_BEGIN_NATIVE 宏的设计优势和适用场景
2. 掌握 `NCB_REGISTER_CLASS` 宏展开后的完整注册链路
3. 理解 `ncbInstanceAdaptor<T>` 如何桥接 C++ 对象与 TJS2 对象
4. 掌握 `ncbTypeConvertor` 的自动类型转换机制
5. 熟练使用 `NCB_METHOD`、`NCB_PROPERTY`、`NCB_CONSTRUCTOR` 等注册宏

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| ncbind | Native Class Binding | KiriKiri 的高级原生绑定框架，用 C++ 模板元编程自动生成类型转换和方法包装代码 |
| 自动注册 | Auto Registration | 利用 C++ 静态变量的构造函数在程序启动时自动完成类注册，无需手动调用 |
| 实例适配器 | Instance Adaptor | `ncbInstanceAdaptor<T>` 模板类，将 C++ 对象指针包装为 `tTJSNativeInstance` |
| 类型转换器 | Type Convertor | `ncbTypeConvertor` 系统，在编译期确定 C++ 类型与 `tTJSVariant` 之间的转换方法 |
| 粘性标志 | Sticky Flag | `ncbInstanceAdaptor` 的 `_sticky` 字段，控制适配器销毁时是否删除底层 C++ 对象 |
| 参数函子 | Params Functor | `paramsFunctor<METHOD>` 模板，自动从 `tTJSVariant**` 提取参数并转换为 C++ 类型 |
| 装箱/拆箱 | Boxing/Unboxing | 将 C++ 对象包装为 TJS2 对象（装箱）或从 TJS2 对象提取 C++ 指针（拆箱） |

## 1. ncbind 的设计动机

### 1.1 底层宏的痛点

上一节介绍的 `TJS_BEGIN_NATIVE_METHOD_DECL` 宏虽然比纯手写 `RegisterNCM` 简单，但仍然有几个问题：

```cpp
// 使用底层宏注册方法——需要手动做的事情太多
TJS_BEGIN_NATIVE_METHOD_DECL(setPosition)
{
    // 1. 手动获取原生实例
    TJS_GET_NATIVE_INSTANCE(ni, tMyClassNI);

    // 2. 手动检查参数数量
    if (numparams < 2)
        return TJS_E_BADPARAMCOUNT;

    // 3. 手动从 tTJSVariant 转换每个参数类型
    tjs_int x = (tjs_int)*param[0];
    tjs_int y = (tjs_int)*param[1];

    // 4. 手动调用 C++ 方法
    ni->SetPosition(x, y);

    // 5. 手动设置返回值
    if (result) *result = tTJSVariant();
    return TJS_S_OK;
}
TJS_END_NATIVE_METHOD_DECL(setPosition)
```

每个方法都要重复这 5 步"样板代码"——获取实例、检查参数、转换类型、调用方法、设置返回值。如果一个类有 30 个方法，每个方法都要写同样的模式，既繁琐又容易出错。

### 1.2 ncbind 的解决方案

ncbind 用 C++ 模板元编程（Template Metaprogramming）在**编译期**自动生成所有样板代码：

```cpp
// 使用 ncbind 注册同一个方法——一行搞定
NCB_REGISTER_CLASS(MyClass) {
    NCB_METHOD(SetPosition);   // 自动推导参数类型、自动转换、自动调用
}
```

编译器会在编译期分析 `MyClass::SetPosition` 的签名（参数类型、返回类型、是否是 const 方法等），自动生成完整的包装代码。运行时零额外开销——所有类型转换逻辑在编译期就已经确定了。

### 1.3 两套系统的关系

```
┌──────────────────────────────────────────────────────┐
│                     TJS2 脚本层                       │
│  var obj = new MyClass();  obj.SetPosition(10, 20);  │
└──────────────────────┬───────────────────────────────┘
                       │
┌──────────────────────┼───────────────────────────────┐
│     ncbind 高级封装层 │ (ncbind.hpp)                   │
│                       │                               │
│  NCB_REGISTER_CLASS  │                               │
│  NCB_METHOD          │    ← 自动生成包装代码           │
│  NCB_PROPERTY        │    ← 编译期类型推导             │
│  NCB_CONSTRUCTOR     │                               │
└──────────────────────┼───────────────────────────────┘
                       │ 内部使用
┌──────────────────────┼───────────────────────────────┐
│     底层原生绑定层     │ (tjsNative.h/cpp)             │
│                       │                               │
│  tTJSNativeClass     │                               │
│  RegisterNCM         │    ← 实际注册机制               │
│  tTJSNativeClassMethod    ← 实际包装对象               │
│  tTJSNativeClassForPlugin ← 跨编译器兼容              │
└──────────────────────┴───────────────────────────────┘
```

ncbind 是底层宏的**上层封装**，最终还是通过 `RegisterNCM` 和 `tTJSNativeClassForPlugin` 来完成注册。选择哪套系统取决于需求：

| 特性 | 底层宏 (TJS_BEGIN_NATIVE_*) | ncbind (NCB_*) |
|------|---------------------------|----------------|
| 代码量 | 多（每个方法约 10-20 行样板） | 少（每个方法 1 行） |
| 类型安全 | 手动转换，易出错 | 编译期检查，类型安全 |
| 灵活性 | 完全控制每个细节 | 自动化程度高，自定义需特殊处理 |
| 适用场景 | 引擎内置类（Array、Math 等） | 插件开发、快速绑定 |
| 编译时间 | 快 | 慢（大量模板实例化） |

## 2. NCB_REGISTER_CLASS —— 自动注册入口

### 2.1 宏展开分析

`NCB_REGISTER_CLASS` 是 ncbind 最核心的宏。让我们看看它展开后到底生成了什么：

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 2157-2175 行

// 用户写的代码:
NCB_REGISTER_CLASS(MyClass) {
    NCB_METHOD(DoSomething);
    NCB_PROPERTY(name, GetName, SetName);
}

// 宏展开后等价于:
// 1. 设置 ncbClassInfo<MyClass> 的类名
template<>
struct ncbClassInfo<MyClass> {
    static const tjs_char* ClassName;  // = TJS_W("MyClass")
    static tjs_int32 ClassID;
    static ncbTypedefs::ClassObjectT *ClassObject;
};

// 2. 创建自动注册器（静态变量，程序启动时自动构造）
static ncbNativeClassAutoRegister<MyClass>
    _ncb_auto_register_MyClass(TJS_W("MyClass"));

// 3. 在注册器的 Regist() 方法中，用户的代码块变成方法体
void ncbNativeClassAutoRegister<MyClass>::Regist() {
    // --- 用户代码块 ---
    NCB_METHOD(DoSomething);
    NCB_PROPERTY(name, GetName, SetName);
    // --- 用户代码块结束 ---
}
```

### 2.2 自动注册链

ncbind 使用 C++ 静态变量的一个特性：**静态变量在 `main()` 之前构造**。通过在每个 `NCB_REGISTER_CLASS` 宏中创建一个静态的 `ncbNativeClassAutoRegister` 对象，所有原生类的注册可以在引擎启动前自动完成。

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 2096-2141 行

// 自动注册器基类
class ncbAutoRegister {
    enum { LINE_COUNT = 3 };  // 三个注册阶段
    static ThisClassT const* _top[LINE_COUNT];

public:
    // 三阶段注册
    enum {
        PreRegist   = 0,  // 预注册（设置回调等）
        ClassRegist = 1,  // 类注册（注册 TJS2 类）
        PostRegist  = 2   // 后注册（注册完成后的回调）
    };

    // 注册所有自动注册器
    static void AllRegist() {
        // 阶段 0: 预注册
        for (auto* p = _top[PreRegist]; p; p = p->_next)
            p->Regist();
        // 阶段 1: 类注册
        for (auto* p = _top[ClassRegist]; p; p = p->_next)
            p->Regist();
        // 阶段 2: 后注册
        for (auto* p = _top[PostRegist]; p; p = p->_next)
            p->Regist();
    }

    // 链表指针——所有注册器串成链表
    ThisClassT const* _next;
};
```

当引擎启动并初始化 TJS2 时，调用 `ncbAutoRegister::AllRegist()` 即可注册所有原生类。

### 2.3 注册的三阶段流程

```
程序启动 → main() 之前
│
├─ 静态变量构造
│   ├─ ncbNativeClassAutoRegister<MyClass1> 加入链表
│   ├─ ncbNativeClassAutoRegister<MyClass2> 加入链表
│   └─ ... 所有 NCB_REGISTER_CLASS 的注册器
│
▼
TJS2 引擎初始化 → AllRegist()
│
├─ Phase 0: PreRegist
│   └─ 执行 NCB_PRE_REGIST_CALLBACK 注册的回调
│
├─ Phase 1: ClassRegist
│   ├─ MyClass1.Regist()
│   │   ├─ RegistBegin() → 创建 tTJSNativeClassForPlugin
│   │   ├─ RegistItem()  → 注册每个方法/属性
│   │   └─ RegistEnd()   → 注册到 TJS2 全局作用域
│   └─ MyClass2.Regist()
│       └─ ... 同上
│
└─ Phase 2: PostRegist
    └─ 执行 NCB_POST_REGIST_CALLBACK 注册的回调
```

### 2.4 RegistBegin / RegistItem / RegistEnd

`ncbRegistNativeClass<CLASS>` 负责 TJS2 侧的注册：

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 1843-1959 行

template <class CLASS>
struct ncbRegistNativeClass {

    // 开始注册——创建类对象
    void RegistBegin() {
        // 1. 创建 tTJSNativeClassForPlugin 对象
        //    传入 CreateNativeInstance 函数指针
        ClassObject = TJSCreateNativeClassForPlugin(
            ClassName,
            &ncbInstanceAdaptor<CLASS>::CreateNativeInstance);

        // 2. 注册 ClassID
        ClassID = TJSRegisterNativeClass(ClassName);

        // 3. 设置 ncbClassInfo 的静态字段
        ncbClassInfo<CLASS>::Set(
            ClassName, ClassID, ClassObject);

        // 4. 注册空的 finalize 方法
        // （避免 TJS2 在销毁时报错）
    }

    // 注册一个成员
    void RegistItem(const tjs_char *name,
                    iTJSDispatch2 *dsp,
                    tjs_uint32 flags) {
        // 调用底层的 TJSNativeClassRegisterNCM
        TJSNativeClassRegisterNCM(
            ClassObject, name, dsp,
            ClassName, flags, 0);
    }

    // 完成注册——添加到全局作用域
    void RegistEnd() {
        // 获取 TJS2 全局对象
        iTJSDispatch2 *global = TJSGetGlobal();
        tTJSVariant val(ClassObject, ClassObject);

        // 注册到全局: global.MyClass = ClassObject
        global->PropSet(
            TJS_MEMBERENSURE,
            ClassName,
            NULL, &val, global);

        ClassObject->Release();
        global->Release();
    }

    // 取消注册（卸载插件时调用）
    void UnregistEnd() {
        iTJSDispatch2 *global = TJSGetGlobal();
        // 从全局删除: delete global.MyClass
        global->DeleteMember(0, ClassName, NULL, global);
        global->Release();
    }
};
```

## 3. ncbInstanceAdaptor —— C++ 对象桥接

### 3.1 设计目的

底层的 `tTJSNativeInstance` 是一个纯虚基类，每个原生类都需要手写子类（如 `tTJSArrayNI`）。ncbind 用 `ncbInstanceAdaptor<T>` 模板自动生成这个子类，避免重复代码：

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 119-232 行

template <class T>
class ncbInstanceAdaptor : public tTJSNativeInstance {
    T *_instance;    // 指向真正的 C++ 对象
    bool _sticky;    // 是否"粘性"——true 表示不拥有 _instance

public:
    // 构造：接收 C++ 对象指针和粘性标志
    ncbInstanceAdaptor(T *inst, bool sticky = false)
        : _instance(inst), _sticky(sticky) {}

    // 析构：如果不是粘性的，删除 C++ 对象
    ~ncbInstanceAdaptor() {
        if (!_sticky && _instance) {
            delete _instance;
            _instance = nullptr;
        }
    }

    // Invalidate: 如果不是粘性的，删除并清空
    void Invalidate() override {
        if (!_sticky && _instance) {
            delete _instance;
            _instance = nullptr;
        }
    }

    // 获取底层 C++ 对象指针
    T *GetNativeInstance() { return _instance; }
};
```

### 3.2 粘性标志（Sticky Flag）

粘性标志控制适配器的所有权语义：

```
┌─────────────────────────────────────────────────┐
│ sticky = false（默认）                            │
│ 适配器拥有 C++ 对象                               │
│                                                 │
│ TJS2 对象销毁 → Invalidate() → delete _instance  │
│ → C++ 对象被释放                                  │
│                                                 │
│ 适用: TJS2 脚本通过 new 创建的对象                 │
│ 例: var obj = new MyClass();                     │
│     // obj 销毁时自动 delete C++ 对象              │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ sticky = true                                    │
│ 适配器不拥有 C++ 对象                              │
│                                                 │
│ TJS2 对象销毁 → Invalidate() → 什么都不做         │
│ → C++ 对象不被释放（由外部代码管理生命周期）         │
│                                                 │
│ 适用: C++ 侧已有对象，只是暴露给 TJS2 访问         │
│ 例: 引擎把自己的 Renderer 对象暴露给脚本            │
│     // 脚本不应该 delete 引擎的核心对象              │
└─────────────────────────────────────────────────┘
```

### 3.3 静态辅助方法

`ncbInstanceAdaptor` 提供了多个静态方法用于 C++ 对象和 TJS2 对象之间的转换：

```cpp
// 从 TJS2 对象获取适配器
static ncbInstanceAdaptor<T> *GetAdaptor(
    iTJSDispatch2 *obj) {
    tTJSNativeInstance *ni = nullptr;
    obj->NativeInstanceSupport(
        TJS_NIS_GETINSTANCE,
        ncbClassInfo<T>::ClassID,
        &ni);
    return static_cast<ncbInstanceAdaptor<T>*>(ni);
}

// 从 TJS2 对象获取 C++ 指针
static T *GetNativeInstance(iTJSDispatch2 *obj) {
    ncbInstanceAdaptor<T> *adp = GetAdaptor(obj);
    return adp ? adp->GetNativeInstance() : nullptr;
}

// 给 TJS2 对象设置 C++ 指针
static void SetNativeInstance(
    iTJSDispatch2 *obj, T *instance) {
    ncbInstanceAdaptor<T> *adp = GetAdaptor(obj);
    if (adp) adp->_instance = instance;
}

// 创建一个包装 C++ 对象的 TJS2 对象
static iTJSDispatch2 *CreateAdaptor(
    T *instance, bool sticky = false) {
    // 1. 通过类对象的 CreateNew 创建空 TJS2 对象
    //    （传入 void 参数跳过构造函数）
    iTJSDispatch2 *obj = nullptr;
    tTJSVariant voidParam;
    ncbClassInfo<T>::ClassObject->CreateNew(
        0, NULL, NULL, &obj, 1, &voidParam, NULL);

    // 2. 创建适配器并注册
    auto *adp = new ncbInstanceAdaptor<T>(
        instance, sticky);
    obj->NativeInstanceSupport(
        TJS_NIS_REGISTER,
        ncbClassInfo<T>::ClassID,
        reinterpret_cast<tTJSNativeInstance**>(&adp));

    return obj;  // 返回 TJS2 对象
}
```

`CreateAdaptor` 使用了"void 参数技巧"——当 `ncbNativeClassConstructor` 收到一个 `void` 类型的参数时，它跳过实例创建步骤，返回一个没有 C++ 对象的空壳。然后由调用者手动设置 C++ 对象指针。

## 4. ncbTypeConvertor —— 编译期类型转换

### 4.1 问题定义

TJS2 的变量类型统一为 `tTJSVariant`，而 C++ 有丰富的类型系统。ncbind 需要在编译期确定如何在两种类型系统之间转换。

### 4.2 基本转换规则

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 236-358 行

struct ncbTypeConvertor {
    // Stripper<T>: 去除 const/volatile/引用/指针修饰符
    // Stripper<const int&>::Type == int
    // Stripper<std::string*>::Type == std::string
    template <typename T>
    struct Stripper {
        typedef typename std::remove_cv<
            typename std::remove_reference<
                typename std::remove_pointer<T>::type
            >::type
        >::type Type;
    };

    // Conversion<FROM, TO>: 类型转换模板
    template <typename FROM, typename TO>
    struct Conversion {
        // 默认实现: 直接 static_cast
        static TO Convert(FROM val) {
            return static_cast<TO>(val);
        }
    };
};
```

### 4.3 数值类型转换

ncbind 为所有数值类型预注册了转换规则：

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 394-404 行

// 整数类型 ↔ tTJSVariant
NCB_TYPECONV_CAST_INTEGER(int);
NCB_TYPECONV_CAST_INTEGER(unsigned int);
NCB_TYPECONV_CAST_INTEGER(long);
NCB_TYPECONV_CAST_INTEGER(unsigned long);
NCB_TYPECONV_CAST_INTEGER(short);
NCB_TYPECONV_CAST_INTEGER(char);
NCB_TYPECONV_CAST_INTEGER(tjs_int);
NCB_TYPECONV_CAST_INTEGER(tjs_int32);

// 浮点类型 ↔ tTJSVariant
NCB_TYPECONV_CAST_REAL(float);
NCB_TYPECONV_CAST_REAL(double);
NCB_TYPECONV_CAST_REAL(tjs_real);
```

这些宏展开后生成特化的 `Conversion` 模板：

```cpp
// NCB_TYPECONV_CAST_INTEGER(int) 展开为:
template<>
struct Conversion<tTJSVariant, int> {
    static int Convert(const tTJSVariant &v) {
        return static_cast<int>((tTVInteger)v);
    }
};
template<>
struct Conversion<int, tTJSVariant> {
    static tTJSVariant Convert(int v) {
        return tTJSVariant((tTVInteger)v);
    }
};
```

### 4.4 字符串类型转换

字符串转换稍复杂，需要处理宽字符和窄字符：

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 407-463 行

// 窄字符 (char*) ↔ tTJSVariant
struct ncbNarrowCharConvertor {
    // C++ → TJS2: 窄字符串转宽字符串再转 Variant
    static tTJSVariant Convert(const char *str) {
        // 使用平台 API 将 char* 转为 wchar_t*
        ttstr wide(str);
        return tTJSVariant(wide);
    }

    // TJS2 → C++: Variant 转宽字符串再转窄字符串
    static std::string ConvertBack(const tTJSVariant &v) {
        ttstr wide((tjs_nchar)v);
        // 使用平台 API 将 wchar_t* 转为 char*
        return std::string(wide.AsNarrowStdString());
    }
};

// 宽字符 (wchar_t*) ↔ tTJSVariant
struct ncbWideCharConvertor {
    static tTJSVariant Convert(const tjs_char *str) {
        return tTJSVariant(ttstr(str));
    }
    static ttstr ConvertBack(const tTJSVariant &v) {
        return ttstr(v);
    }
};
```

### 4.5 自定义类型转换——装箱/拆箱

对于用户自定义的 C++ 类，ncbind 支持"装箱（Boxing）"和"拆箱（Unboxing）"：

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 504-553 行

// NCB_TYPECONV_BOXING(MyStruct) 宏生成:

// 装箱: MyStruct → tTJSVariant（创建 TJS2 对象包装 C++ 对象）
template<>
struct Conversion<MyStruct, tTJSVariant> {
    static tTJSVariant Convert(const MyStruct &obj) {
        // 复制一份 MyStruct，用 CreateAdaptor 包装
        MyStruct *copy = new MyStruct(obj);
        iTJSDispatch2 *dsp =
            ncbInstanceAdaptor<MyStruct>::CreateAdaptor(copy);
        tTJSVariant result(dsp, dsp);
        dsp->Release();
        return result;
    }
};

// 拆箱: tTJSVariant → MyStruct*（从 TJS2 对象提取 C++ 指针）
template<>
struct Conversion<tTJSVariant, MyStruct*> {
    static MyStruct* Convert(const tTJSVariant &v) {
        iTJSDispatch2 *obj = v.AsObjectNoAddRef();
        return ncbInstanceAdaptor<MyStruct>::GetNativeInstance(obj);
    }
};
```

使用示例：

```cpp
// 定义一个 C++ 结构体
struct Vector2D {
    float x, y;
    Vector2D(float x, float y) : x(x), y(y) {}
};

// 注册为 TJS2 类
NCB_REGISTER_CLASS(Vector2D) {
    NCB_CONSTRUCTOR((float, float));
    NCB_PROPERTY_RO(x, GetX);
    NCB_PROPERTY_RO(y, GetY);
}

// 启用装箱/拆箱
NCB_TYPECONV_BOXING(Vector2D);

// 现在 Vector2D 可以作为参数/返回值自动转换
class Player {
public:
    Vector2D GetPosition() { return {x, y}; }  // 返回值自动装箱
    void SetPosition(Vector2D pos) { ... }       // 参数自动拆箱
};
```

## 5. 方法包装——ncbNativeClassMethodBase

### 5.1 参数函子（paramsFunctor）

ncbind 最核心的模板技术是 `paramsFunctor`——它在编译期分析 C++ 方法的签名，自动生成参数提取和类型转换代码：

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 849-1317 行（高度简化）

template <typename METHOD>
struct paramsFunctor {
    // 假设 METHOD = void (MyClass::*)(int, float)
    // 编译器会推导出:
    //   返回类型 = void
    //   参数 0 类型 = int
    //   参数 1 类型 = float

    static tjs_error Invoke(
        METHOD method,          // 方法指针
        MyClass *instance,      // 实例指针
        tTJSVariant *result,    // 返回值
        tjs_int numparams,      // 参数数量
        tTJSVariant **param)    // 参数数组
    {
        // 编译期生成的代码（等价于）:

        // 1. 检查参数数量
        if (numparams < 2)
            return TJS_E_BADPARAMCOUNT;

        // 2. 自动转换每个参数
        int arg0 = ncbTypeConvertor::Convert<
            tTJSVariant, int>(*param[0]);
        float arg1 = ncbTypeConvertor::Convert<
            tTJSVariant, float>(*param[1]);

        // 3. 调用方法
        (instance->*method)(arg0, arg1);

        // 4. 返回值处理（void 方法无返回值）
        return TJS_S_OK;
    }
};
```

实际实现使用了递归模板展开（C++11 参数包展开），但概念上就是上面的逻辑。对于有返回值的方法：

```cpp
// 假设 METHOD = int (MyClass::*)(const char*)
// 生成:
static tjs_error Invoke(...) {
    std::string arg0 = ncbTypeConvertor::Convert<
        tTJSVariant, std::string>(*param[0]);
    int ret = (instance->*method)(arg0.c_str());
    if (result)
        *result = ncbTypeConvertor::Convert<
            int, tTJSVariant>(ret);
    return TJS_S_OK;
}
```

### 5.2 原生实例获取器

`paramsFunctor` 还需要从 `objthis` 获取 C++ 实例指针。这通过 `nativeInstanceGetter` 实现：

```cpp
template <class T>
struct nativeInstanceGetter {
    static T *Get(iTJSDispatch2 *objthis) {
        return ncbInstanceAdaptor<T>::GetNativeInstance(objthis);
    }
};

// 对于静态方法（ClassT == void），不需要实例:
template <>
struct nativeInstanceGetter<void> {
    static void *Get(iTJSDispatch2 *) {
        return nullptr;
    }
};
```

### 5.3 完整的调用链

一个 ncbind 注册的方法，从 TJS2 脚本调用到 C++ 执行的完整路径：

```
TJS2: obj.DoSomething(42, "hello")
│
├─ 1. PropGet("DoSomething") → 获取 ncbNativeClassMethod 对象
│
├─ 2. FuncCall(NULL, result, 2, [42, "hello"], obj)
│     └─ ncbNativeClassMethod::FuncCall
│        └─ doInvoke<SELECTOR>::Exec(method, result, numparams, param, objthis)
│
├─ 3. nativeInstanceGetter<MyClass>::Get(objthis)
│     └─ ncbInstanceAdaptor<MyClass>::GetNativeInstance(objthis)
│     └─ → MyClass* instance
│
├─ 4. paramsFunctor::Invoke(method, instance, result, numparams, param)
│     ├─ Convert<tTJSVariant, int>(param[0])         → int arg0 = 42
│     ├─ Convert<tTJSVariant, std::string>(param[1])  → string arg1 = "hello"
│     └─ instance->DoSomething(arg0, arg1)
│
└─ 5. Convert<ReturnType, tTJSVariant>(ret) → 设置 result
```

## 6. 注册宏详解

### 6.1 NCB_METHOD —— 注册方法

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 2206-2210 行

#define NCB_METHOD(method) \
    Method(TJS_W(#method), &Class::method)

// 内部实现:
template <typename M>
void Method(const tjs_char *name, M method) {
    // 创建 ncbNativeClassMethod<M> 包装
    // 注册到类对象
    RegistItem(name,
        new ncbNativeClassMethod<M>(method),
        0);  // flags = 0，普通成员
}
```

使用示例：

```cpp
class Calculator {
public:
    int Add(int a, int b) { return a + b; }
    void Reset() { value = 0; }
    static double Pi() { return 3.14159265; }

private:
    int value = 0;
};

NCB_REGISTER_CLASS(Calculator) {
    NCB_METHOD(Add);     // 自动检测: int(int,int)，实例方法
    NCB_METHOD(Reset);   // 自动检测: void()，实例方法
    NCB_METHOD(Pi);      // 自动检测: double()，静态方法
}
```

ncbind 通过模板参数推导自动判断方法是否为静态——如果方法指针类型不包含类名（即普通函数指针而非成员函数指针），则标记为 `TJS_STATICMEMBER`。

### 6.2 NCB_METHOD_DIFFER —— 用不同名称注册

当 C++ 方法名与期望的 TJS2 方法名不同时使用：

```cpp
#define NCB_METHOD_DIFFER(name, method) \
    Method(TJS_W(#name), &Class::method)

// 使用示例:
class Player {
public:
    void SetPositionXY(int x, int y) { ... }
};

NCB_REGISTER_CLASS(Player) {
    // TJS2 中叫 "setPos"，C++ 中叫 "SetPositionXY"
    NCB_METHOD_DIFFER(setPos, SetPositionXY);
}

// TJS2 脚本:
// player.setPos(100, 200);  // OK
// player.SetPositionXY(100, 200);  // 报错: member not found
```

### 6.3 NCB_PROPERTY / NCB_PROPERTY_RO / NCB_PROPERTY_WO

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 2228-2242 行

// 读写属性
#define NCB_PROPERTY(name, getter, setter) \
    Property(TJS_W(#name), &Class::getter, &Class::setter)

// 只读属性
#define NCB_PROPERTY_RO(name, getter) \
    Property(TJS_W(#name), &Class::getter, (int)0)

// 只写属性
#define NCB_PROPERTY_WO(name, setter) \
    Property(TJS_W(#name), (int)0, &Class::setter)
```

使用示例：

```cpp
class Window {
    int width_, height_;
    std::string title_;

public:
    int GetWidth() const { return width_; }
    void SetWidth(int w) { width_ = w; }

    int GetHeight() const { return height_; }
    // 没有 SetHeight —— 高度只读

    const std::string &GetTitle() const { return title_; }
    void SetTitle(const std::string &t) { title_ = t; }
};

NCB_REGISTER_CLASS(Window) {
    NCB_PROPERTY(width, GetWidth, SetWidth);    // 读写
    NCB_PROPERTY_RO(height, GetHeight);          // 只读
    NCB_PROPERTY(title, GetTitle, SetTitle);     // 读写
}

// TJS2 脚本:
// var w = new Window();
// w.width = 800;         // 调用 SetWidth(800)
// var h = w.height;      // 调用 GetHeight()
// w.height = 600;        // 报错: TJS_E_ACCESSDENYED
```

### 6.4 NCB_CONSTRUCTOR —— 注册构造函数

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 2196-2200 行

#define NCB_CONSTRUCTOR(cargs) \
    Constructor(TypeWrap<void (Class::*)cargs>())

// 使用示例:
class Sprite {
public:
    Sprite() {}                              // 默认构造
    Sprite(int x, int y) : x_(x), y_(y) {}  // 带参构造
};

NCB_REGISTER_CLASS(Sprite) {
    NCB_CONSTRUCTOR((int, int));  // 注册带参构造函数
    // 注意: 参数类型用括号包裹
}

// TJS2 脚本:
// var s = new Sprite(100, 200);
```

如果不调用 `NCB_CONSTRUCTOR`，则使用默认构造函数（无参数）。`Constructor()` 不带参数时注册默认构造。

### 6.5 NCB_ATTACH_CLASS —— 扩展已有类

有时需要给已经注册的 TJS2 类（如 Layer、Window）添加新方法：

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 2184-2188 行

#define NCB_ATTACH_CLASS(cls, attach) \
    /* 不创建新的 TJS2 类，而是将方法添加到已有类 */

// 使用示例:
class LayerExtension {
public:
    void DrawCircle(int x, int y, int r) { ... }
    void DrawLine(int x1, int y1, int x2, int y2) { ... }
};

// 将 LayerExtension 的方法附加到已有的 "Layer" 类
NCB_ATTACH_CLASS(LayerExtension, Layer) {
    NCB_METHOD(DrawCircle);
    NCB_METHOD(DrawLine);
}

// TJS2 脚本:
// var layer = new Layer(window, null);
// layer.DrawCircle(100, 100, 50);  // 使用扩展方法
```

### 6.6 NCB_REGISTER_FUNCTION —— 注册独立函数

注册全局函数（不属于任何类）：

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 2275-2280 行

#define NCB_REGISTER_FUNCTION(name, function) \
    /* 创建包装并注册到 TJS2 全局作用域 */

// 使用示例:
int MathHelper_GCD(int a, int b) {
    while (b) { int t = b; b = a % b; a = t; }
    return a;
}

NCB_REGISTER_FUNCTION(GCD, MathHelper_GCD);

// TJS2 脚本:
// var result = GCD(12, 8);  // result = 4
```

### 6.7 NCB_METHOD_RAW_CALLBACK —— 原始回调

当需要完全控制参数处理时，可以注册原始回调函数：

```cpp
#define NCB_METHOD_RAW_CALLBACK(name, cb, flag) \
    RawCallback(TJS_W(#name), cb, flag)

// 回调函数签名
static tjs_error MyRawCallback(
    tTJSVariant *result,
    tjs_int numparams,
    tTJSVariant **param,
    iTJSDispatch2 *objthis)
{
    // 完全手动处理参数
    // 这跟底层宏的 Process 函数签名完全一样
    return TJS_S_OK;
}

NCB_REGISTER_CLASS(MyClass) {
    NCB_METHOD_RAW_CALLBACK(rawMethod, MyRawCallback, 0);
}
```

### 6.8 NCB_SUBCLASS —— 注册子类

```cpp
#define NCB_SUBCLASS(name, cls) \
    SubClass(TJS_W(#name), TypeWrap<cls>())

// 使用示例:
class OuterClass {
public:
    class InnerClass {
        int value;
    public:
        int GetValue() { return value; }
    };
};

NCB_REGISTER_CLASS(OuterClass) {
    NCB_SUBCLASS(Inner, OuterClass::InnerClass);
}

// TJS2 脚本:
// var inner = new OuterClass.Inner();
```

## 7. ncbPropAccessor —— 属性访问工具

### 7.1 设计目的

当 C++ 代码需要反向操作 TJS2 对象（读取/设置属性、调用方法）时，直接使用 `iTJSDispatch2` 的接口太繁琐。`ncbPropAccessor` 提供了便捷的封装：

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 602-821 行

class ncbPropAccessor {
    iTJSDispatch2 *_obj;  // 被访问的 TJS2 对象

public:
    ncbPropAccessor(iTJSDispatch2 *obj) : _obj(obj) {}
    // 也可以从 tTJSVariant 构造
    ncbPropAccessor(const tTJSVariant &v)
        : _obj(v.AsObjectNoAddRef()) {}

    // 获取整数属性
    tjs_int GetValue(const tjs_char *name,
                     tjs_int defval = 0) {
        tTJSVariant val;
        if (TJS_SUCCEEDED(_obj->PropGet(
                0, name, NULL, &val, _obj)))
            return (tjs_int)val;
        return defval;  // 属性不存在时返回默认值
    }

    // 获取字符串属性
    ttstr GetValue(const tjs_char *name,
                   const ttstr &defval = ttstr()) {
        tTJSVariant val;
        if (TJS_SUCCEEDED(_obj->PropGet(
                0, name, NULL, &val, _obj)))
            return ttstr(val);
        return defval;
    }

    // 设置属性
    template <typename T>
    void SetValue(const tjs_char *name, T value) {
        tTJSVariant val(value);
        _obj->PropSet(
            TJS_MEMBERENSURE, name, NULL, &val, _obj);
    }

    // 检查属性是否存在
    bool HasValue(const tjs_char *name) {
        tTJSVariant val;
        return TJS_SUCCEEDED(_obj->PropGet(
            0, name, NULL, &val, _obj));
    }

    // 调用方法
    tTJSVariant FuncCall(const tjs_char *name,
                         tjs_int numparams = 0,
                         tTJSVariant **param = NULL) {
        tTJSVariant result;
        _obj->FuncCall(0, name, NULL, &result,
                       numparams, param, _obj);
        return result;
    }

    // 获取数组长度（通过 count 属性）
    tjs_int GetArrayCount() {
        return GetValue(TJS_W("count"), 0);
    }
};
```

### 7.2 使用示例

```cpp
// 在 C++ 原生方法中操作传入的 TJS2 字典对象
static tjs_error ProcessConfig(
    tTJSVariant *result, tjs_int numparams,
    tTJSVariant **param, iTJSDispatch2 *objthis)
{
    if (numparams < 1) return TJS_E_BADPARAMCOUNT;

    // 用 ncbPropAccessor 方便地访问字典
    ncbPropAccessor config(*param[0]);

    // 读取属性
    int width = config.GetValue(TJS_W("width"), 800);
    int height = config.GetValue(TJS_W("height"), 600);
    ttstr title = config.GetValue(
        TJS_W("title"), ttstr(TJS_W("Untitled")));
    bool fullscreen = config.GetValue(
        TJS_W("fullscreen"), 0) != 0;

    // 检查可选属性
    if (config.HasValue(TJS_W("vsync"))) {
        int vsync = config.GetValue(TJS_W("vsync"), 1);
        // 设置垂直同步...
    }

    return TJS_S_OK;
}
```

ncbind 还提供了 `ncbArrayAccessor` 和 `ncbDictionaryAccessor` 作为特化的便捷包装。

## 8. 自动注册回调

### 8.1 NCB_PRE_REGIST_CALLBACK / NCB_POST_REGIST_CALLBACK

```cpp
// 源码位置: cpp/core/plugin/ncbind.hpp 第 2265-2270 行

// 在所有类注册之前执行
#define NCB_PRE_REGIST_CALLBACK(cb) \
    static ncbAutoRegisterCallback _ncb_pre_##cb(\
        cb, ncbAutoRegister::PreRegist)

// 在所有类注册之后执行
#define NCB_POST_REGIST_CALLBACK(cb) \
    static ncbAutoRegisterCallback _ncb_post_##cb(\
        cb, ncbAutoRegister::PostRegist)
```

使用示例：

```cpp
// 预注册回调——在类注册前初始化全局资源
static void InitResources() {
    // 初始化图形资源管理器
    ResourceManager::Initialize();
}
NCB_PRE_REGIST_CALLBACK(InitResources);

// 后注册回调——在类注册后执行额外设置
static void PostSetup() {
    // 向 TJS2 全局注入一些常量
    iTJSDispatch2 *global = TJSGetGlobal();
    tTJSVariant ver(TJS_W("1.0.0"));
    global->PropSet(TJS_MEMBERENSURE,
        TJS_W("PLUGIN_VERSION"), NULL, &ver, global);
    global->Release();
}
NCB_POST_REGIST_CALLBACK(PostSetup);
```

### 8.2 NCB_MODULE_NAME —— 插件模块名

```cpp
// 定义模块名（用于内部插件系统的按需加载）
#define NCB_MODULE_NAME TJS_W("steam_api.dll")

// 引擎可以通过 ncbAutoRegister::LoadModule("steam_api.dll")
// 按需加载特定模块的所有注册
```

在 KrKr2 中，所有插件都静态链接，但 `NCB_MODULE_NAME` 允许引擎按模块名分组管理注册，实现按需加载的效果。

## 动手实践

### 实践 1：分析 steam_api 插件的注册过程

KrKr2 中唯一使用 ncbind 的插件 `steam_api.cpp` 只有 12 行代码：

```cpp
// 源码位置: cpp/plugins/steam/steam_api.cpp

#include "ncbind.hpp"
#include <string>

#define NCB_MODULE_NAME TJS_W("steam_api.dll")

class steam_api {
public:
    std::string vserion = "1.0.0";
};

NCB_REGISTER_CLASS(steam_api) { Constructor(); }
```

请回答：
1. `Constructor()` 不带参数意味着什么？
2. `vserion` 字段能否从 TJS2 脚本访问？为什么？
3. `NCB_MODULE_NAME` 的作用是什么？

答案：
1. 注册默认无参构造函数 `steam_api()`。TJS2 脚本可以用 `new steam_api()` 创建实例。
2. 不能。`vserion` 是 C++ 成员变量，但没有通过 `NCB_PROPERTY` 注册为 TJS2 属性，TJS2 脚本无法访问它。
3. 将这个插件的注册归入 `"steam_api.dll"` 模块名下，引擎可以通过 `ncbAutoRegister::LoadModule(TJS_W("steam_api.dll"))` 按需触发注册。

### 实践 2：编写完整的 ncbind 注册

为以下 C++ 类编写完整的 ncbind 注册代码：

```cpp
class StringBuffer {
    std::string buffer_;

public:
    StringBuffer() : buffer_("") {}
    StringBuffer(const char *initial)
        : buffer_(initial) {}

    void Append(const char *text) {
        buffer_ += text;
    }

    void Clear() {
        buffer_.clear();
    }

    const char *GetContent() const {
        return buffer_.c_str();
    }

    int GetLength() const {
        return (int)buffer_.size();
    }

    bool IsEmpty() const {
        return buffer_.empty();
    }
};

// 注册代码:
NCB_REGISTER_CLASS(StringBuffer) {
    // 注册带参构造函数
    NCB_CONSTRUCTOR((const char*));

    // 注册方法
    NCB_METHOD(Append);
    NCB_METHOD(Clear);
    NCB_METHOD(IsEmpty);

    // 注册属性
    NCB_PROPERTY_RO(content, GetContent);  // 只读
    NCB_PROPERTY_RO(length, GetLength);    // 只读
}
```

在 TJS2 脚本中使用：

```javascript
var buf = new StringBuffer("Hello");
buf.Append(", World!");
Debug.message(buf.content);   // "Hello, World!"
Debug.message(buf.length);    // 13
Debug.message(buf.IsEmpty()); // false
buf.Clear();
Debug.message(buf.IsEmpty()); // true
```

## 对照项目源码

相关文件：
- `cpp/core/plugin/ncbind.hpp` 第 73-114 行 —— `ncbClassInfo<T>` 类信息存储
- `cpp/core/plugin/ncbind.hpp` 第 119-232 行 —— `ncbInstanceAdaptor<T>` 实例适配器
- `cpp/core/plugin/ncbind.hpp` 第 236-553 行 —— `ncbTypeConvertor` 类型转换系统
- `cpp/core/plugin/ncbind.hpp` 第 602-821 行 —— `ncbPropAccessor` 属性访问工具
- `cpp/core/plugin/ncbind.hpp` 第 849-1317 行 —— `ncbNativeClassMethodBase` 方法包装基类
- `cpp/core/plugin/ncbind.hpp` 第 1664-1815 行 —— `ncbRegistClass<IMPL>` 注册 DSL
- `cpp/core/plugin/ncbind.hpp` 第 1843-1959 行 —— `ncbRegistNativeClass` TJS2 注册
- `cpp/core/plugin/ncbind.hpp` 第 2096-2141 行 —— `ncbAutoRegister` 自动注册链
- `cpp/core/plugin/ncbind.hpp` 第 2157-2280 行 —— 所有高级宏定义
- `cpp/core/plugin/ncbind.cpp` 第 1-29 行 —— 自动注册器静态变量和 LoadModule 实现
- `cpp/plugins/steam/steam_api.cpp` —— 唯一的 ncbind 插件使用示例

## 常见错误与排查

### 错误 1：编译错误 "no matching function for call to Method"

```
症状: NCB_METHOD(MyFunc) 编译报错
原因: MyFunc 的参数类型没有对应的 ncbTypeConvertor 转换
排查:
  1. 检查 MyFunc 的参数是否都是支持的类型
     （int, float, double, const char*, ttstr 等）
  2. 对于自定义类型，需要先注册 NCB_TYPECONV_BOXING
  3. 重载函数需要用 NCB_METHOD_DIFFER 并指定正确的签名
```

### 错误 2：运行时崩溃 "native class crash"

```
症状: TJS2 脚本调用 ncbind 注册的方法时崩溃
原因: ncbInstanceAdaptor 无法获取 C++ 实例
排查:
  1. 确认类已经通过 NCB_REGISTER_CLASS 注册
  2. 确认对象是通过 TJS2 的 new 创建的
  3. 检查是否在 invalidate 之后继续使用
  4. 检查 ClassID 是否冲突（不同类用了相同名称）
```

### 错误 3：属性读取返回 undefined

```
症状: TJS2 脚本读取属性得到 void/undefined
原因: getter 方法签名不匹配
排查:
  1. getter 必须是 const 方法（推荐但非强制）
  2. getter 返回类型必须可转换为 tTJSVariant
  3. getter 不能有参数
  4. 确认 NCB_PROPERTY_RO 的第二个参数是 getter 不是 setter
```

## 本节小结

- **ncbind vs 底层宏**：ncbind 用模板元编程自动生成类型转换和参数处理代码，一行 `NCB_METHOD` 替代 10-20 行样板代码
- **自动注册**：`NCB_REGISTER_CLASS` 创建静态变量，利用 C++ 静态初始化在 `main()` 前自动注册所有类
- **三阶段注册**：PreRegist → ClassRegist → PostRegist，通过链表串连所有注册器
- **ncbInstanceAdaptor**：将 C++ 对象包装为 `tTJSNativeInstance`，支持粘性/非粘性两种所有权模式
- **ncbTypeConvertor**：编译期类型转换系统，数值/字符串/自定义类型都有对应的转换路径
- **paramsFunctor**：编译期分析方法签名，自动生成参数提取、类型转换、方法调用的完整代码
- **ncbPropAccessor**：反向操作 TJS2 对象的便捷工具，提供类型安全的属性读写和方法调用

## 练习题与答案

### 题目 1：理解自动注册时序

以下代码中，三个回调/注册的执行顺序是什么？

```cpp
NCB_POST_REGIST_CALLBACK(DoPost);
NCB_REGISTER_CLASS(MyClass) { NCB_METHOD(Func); }
NCB_PRE_REGIST_CALLBACK(DoPre);
```

<details>
<summary>查看答案</summary>

执行顺序为：
1. `DoPre()` —— PreRegist 阶段
2. `MyClass` 的类注册 —— ClassRegist 阶段
3. `DoPost()` —— PostRegist 阶段

与代码中的书写顺序无关。ncbAutoRegister 按三个阶段分别遍历链表：先执行所有 PreRegist 阶段的回调，再执行所有 ClassRegist 阶段的注册，最后执行所有 PostRegist 阶段的回调。

注意：在同一阶段内，多个注册的执行顺序取决于静态变量的初始化顺序（C++ 标准不保证跨编译单元的顺序），所以不应依赖同阶段内的执行顺序。

</details>

### 题目 2：sticky 标志的使用场景

解释以下代码中 `CreateAdaptor` 的 `sticky` 参数为何设为 `true`：

```cpp
class GameEngine {
    Renderer renderer_;  // 引擎拥有的渲染器

public:
    iTJSDispatch2 *GetRendererForScript() {
        return ncbInstanceAdaptor<Renderer>::CreateAdaptor(
            &renderer_, true);  // sticky = true
    }
};
```

<details>
<summary>查看答案</summary>

`sticky = true` 的原因是**所有权**问题：

- `renderer_` 是 `GameEngine` 的成员变量，由 `GameEngine` 负责生命周期管理
- 如果 `sticky = false`（默认值），当 TJS2 脚本中引用的 Renderer 对象被垃圾回收时，`ncbInstanceAdaptor` 的 `Invalidate()` 会 `delete _instance`
- 这会导致 `GameEngine::renderer_` 被删除，但 `GameEngine` 不知道这件事，后续访问 `renderer_` 会导致 use-after-free 崩溃

设为 `sticky = true` 后：
- TJS2 对象被垃圾回收时，`Invalidate()` 不会删除 `renderer_`
- `renderer_` 的生命周期仍然由 `GameEngine` 管理
- TJS2 脚本只是"借用"这个对象，不拥有它

**规则总结**：
- 如果 C++ 对象由 TJS2 脚本通过 `new` 创建 → `sticky = false`（适配器拥有对象）
- 如果 C++ 对象由其他代码管理，只是暴露给脚本访问 → `sticky = true`（适配器不拥有对象）

</details>

### 题目 3：补全 ncbind 注册代码

以下 C++ 类需要暴露给 TJS2 脚本。请补全 `NCB_REGISTER_CLASS` 代码块：

```cpp
class Timer {
    double startTime_;
    bool running_;

public:
    Timer() : startTime_(0), running_(false) {}

    void Start() {
        startTime_ = GetCurrentTime();
        running_ = true;
    }

    void Stop() { running_ = false; }

    double GetElapsed() const {
        if (!running_) return 0.0;
        return GetCurrentTime() - startTime_;
    }

    bool IsRunning() const { return running_; }

    static double GetCurrentTime() {
        // 返回当前时间（秒）
        return /* ... */;
    }
};

// 补全以下注册代码:
NCB_REGISTER_CLASS(Timer) {
    // 你的代码
}
```

<details>
<summary>查看答案</summary>

```cpp
NCB_REGISTER_CLASS(Timer) {
    // 注册默认构造函数
    Constructor();

    // 注册实例方法
    NCB_METHOD(Start);
    NCB_METHOD(Stop);
    NCB_METHOD(IsRunning);

    // 注册只读属性（elapsed 更适合作为属性而非方法）
    NCB_PROPERTY_RO(elapsed, GetElapsed);

    // 注册静态方法
    // ncbind 自动检测 GetCurrentTime 为 static 方法
    // 并设置 TJS_STATICMEMBER 标志
    NCB_METHOD(GetCurrentTime);
}
```

TJS2 脚本使用：
```javascript
var timer = new Timer();
timer.Start();
// ... 等待一段时间 ...
Debug.message("Elapsed: " + timer.elapsed + " seconds");
Debug.message("Running: " + timer.IsRunning());
timer.Stop();

// 静态方法调用
var now = Timer.GetCurrentTime();
```

</details>

## 下一步

[C++类暴露给TJS2](./03-C++类暴露给TJS2.md) —— 通过完整的实战案例，对比两种绑定方式（底层宏 vs ncbind）的优劣，学习真实项目中如何选择和使用原生绑定方案。

