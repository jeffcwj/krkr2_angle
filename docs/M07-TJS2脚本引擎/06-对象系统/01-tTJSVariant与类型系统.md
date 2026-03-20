# tTJSVariant 与类型系统

> **所属模块：** M07-TJS2脚本引擎
> **前置知识：** [VM执行-ExecuteCode调度循环](../05-VM执行/01-ExecuteCode调度循环.md)、[字节码编译-常量池与代码优化](../04-字节码编译/03-常量池与代码优化.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 `tTJSVariant` 的内存布局和 6 种类型标签
2. 掌握 `tTJSVariantClosure` 的 Object/ObjThis 双指针设计及其 this 绑定逻辑
3. 理解 TJS2 动态类型系统的类型转换规则和引用计数管理
4. 阅读和修改涉及 `tTJSVariant` 的 C++ 源码

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| Variant | Variant Type | 一种可以存储多种类型值的"万能容器"，运行时通过类型标签区分当前存储的值类型 |
| 类型标签 | Type Tag / Discriminant | 一个枚举值，标识 Variant 当前存储的是哪种类型（void/object/string/integer/real/octet） |
| 闭包 | Closure | 一个函数对象加上它的执行上下文（this 指针），两者绑定在一起形成一个可调用单元 |
| 引用计数 | Reference Counting | 一种内存管理策略：每个对象维护一个计数器，被引用时加 1，不再被引用时减 1，减到 0 时自动释放 |
| Octet | Octet (Binary Data) | 原始二进制数据（字节序列），类似于 JavaScript 的 `Uint8Array`，用于处理文件、网络数据等 |
| ObjThis | Object This Pointer | 闭包中绑定的 this 对象指针，决定了函数执行时的 `this` 是谁 |

---

## §1 TJS2 的类型系统概览

TJS2 是一门**动态类型语言**（dynamically typed language，变量的类型在运行时确定，而不是编译时）——一个变量可以在不同时刻存储不同类型的值。为了实现这个特性，TJS2 使用 `tTJSVariant` 作为统一的值容器：VM 的每个寄存器、每个函数参数、每个返回值都是一个 `tTJSVariant`。

```
┌─────────────────────────────────────────────────┐
│                TJS2 类型系统                      │
├─────────────────────────────────────────────────┤
│                                                  │
│  值类型（直接存储在 Variant 内部）                 │
│  ├── tvtVoid     —— 空值（类似 null/undefined）   │
│  ├── tvtInteger  —— 64 位整数                     │
│  └── tvtReal     —— 64 位浮点数                   │
│                                                  │
│  引用类型（通过指针引用，使用引用计数）             │
│  ├── tvtObject   —— 对象闭包（Object + ObjThis） │
│  ├── tvtString   —— 字符串（tTJSVariantString*）  │
│  └── tvtOctet    —— 二进制数据（tTJSVariantOctet*）│
│                                                  │
└─────────────────────────────────────────────────┘
```

与 JavaScript 的类型系统对比：

| TJS2 类型 | JavaScript 等价 | 区别 |
|-----------|-----------------|------|
| `tvtVoid` | `undefined` / `null` | TJS2 不区分 undefined 和 null |
| `tvtInteger` | `Number`（整数部分） | TJS2 有独立的 64 位整数类型 |
| `tvtReal` | `Number`（浮点部分） | TJS2 将整数和浮点分开存储 |
| `tvtString` | `string` | 两者都是不可变引用类型 |
| `tvtObject` | `object` / `function` | TJS2 的 Object 同时携带 this 绑定 |
| `tvtOctet` | `Uint8Array` | TJS2 有原生的二进制数据类型 |

---

## §2 tTJSVariant_S — 底层内存布局

`tTJSVariant` 继承自 `tTJSVariant_S`，后者定义了实际的内存布局：

```cpp
// 源码：cpp/core/tjs2/tjsVariant.h 第 103-153 行

// 类型枚举
enum tTJSVariantType {
    tvtVoid,      // 0 — 空值
    tvtObject,    // 1 — 对象（闭包）
    tvtString,    // 2 — 字符串
    tvtOctet,     // 3 — 二进制数据
    tvtInteger,   // 4 — 64 位整数
    tvtReal       // 5 — 64 位浮点数
};

// 闭包结构体
struct tTJSVariantClosure_S {
    iTJSDispatch2 *Object;    // 对象/函数指针
    iTJSDispatch2 *ObjThis;   // this 绑定指针
};

// Variant 的底层存储结构
#pragma pack(push, 4)
struct tTJSVariant_S {
    union {
        tTJSVariantClosure_S Object;   // tvtObject 时使用
        tTVInteger Integer;             // tvtInteger 时使用（tjs_int64）
        tTVReal Real;                   // tvtReal 时使用（double）
        tTJSVariantString *String;      // tvtString 时使用
        tTJSVariantOctet *Octet;        // tvtOctet 时使用
    };
    tTJSVariantType vt;  // 类型标签
};
#pragma pack(pop)
```

### 2.1 内存布局图

```
tTJSVariant_S 内存布局（16 字节 + 4 字节 = 20 字节，4 字节对齐）

偏移  大小    tvtObject     tvtInteger    tvtReal       tvtString     tvtOctet
────  ────    ──────────    ──────────    ──────────    ──────────    ──────────
 0    8字节   Object*       Integer       Real          String*       Octet*
                            (tjs_int64)   (double)      (指针)        (指针)
 8    8字节   ObjThis*      (填充)        (填充)        (填充)        (填充)
16    4字节   vt=tvtObject  vt=tvtInteger vt=tvtReal    vt=tvtString  vt=tvtOctet
```

**关键设计选择：**

1. **union 联合体**：所有类型共享同一块 16 字节内存。同一时刻只有一种类型有效，由 `vt` 字段（类型标签）标识。这节省了大量内存——如果为每种类型都分配独立的存储，`tTJSVariant` 将大 3 倍以上。

2. **`#pragma pack(push, 4)`**：强制 4 字节对齐。默认情况下编译器可能将结构体按 8 字节对齐（因为包含 `tjs_int64` 和 `double`），这里压缩到 4 字节以减少内存占用。

3. **`tvtObject` 占用 16 字节**：Object 类型需要两个指针（Object + ObjThis），是所有类型中占用最大的，因此 union 的大小由它决定。

### 2.2 tTJSVariant_BITCOPY 宏

```cpp
#define tTJSVariant_BITCOPY(a, b) \
    { *(tTJSVariant_S *)&(a) = *(tTJSVariant_S *)&(b); }
```

这个宏执行**逐位复制**（bitwise copy），直接拷贝整个 `tTJSVariant_S` 结构体的 20 字节。它比逐字段复制更快，但**不进行任何引用计数操作**。这意味着：
- 复制后，如果内容是引用类型（Object/String/Octet），引用计数不会被增加
- 调用者必须自己处理引用计数（或确保旧值不再被使用）
- 在 VM 内部的某些"移动"操作中使用，避免了 AddRef/Release 的开销

---

## §3 tTJSVariant 类 — 完整的值容器

`tTJSVariant` 继承 `tTJSVariant_S` 并添加了类型安全的构造、析构、赋值和类型转换方法：

### 3.1 构造函数族

```cpp
// 源码：cpp/core/tjs2/tjsVariant.h 第 546-643 行

// 默认构造：创建 void 值
tTJSVariant() { vt = tvtVoid; }

// 从另一个 Variant 复制
tTJSVariant(const tTJSVariant &ref);  // 深复制，增加引用计数

// 从对象指针构造（Object 类型）
tTJSVariant(iTJSDispatch2 *ref) {
    if(ref) ref->AddRef();    // 增加引用计数
    vt = tvtObject;
    Object.Object = ref;
    Object.ObjThis = nullptr; // 无 this 绑定
}

// 从闭包构造（Object + ObjThis）
tTJSVariant(iTJSDispatch2 *obj, iTJSDispatch2 *objthis) {
    if(obj) obj->AddRef();
    if(objthis) objthis->AddRef();
    vt = tvtObject;
    Object.Object = obj;
    Object.ObjThis = objthis;
}

// 从 C 字符串构造（String 类型）
tTJSVariant(const tjs_char *ref) {
    vt = tvtString;
    if(ref) {
        String = TJSAllocVariantString(ref);  // 分配并复制字符串
    } else {
        String = nullptr;
    }
}

// 从 tTJSString 构造
tTJSVariant(const tTJSString &ref) {
    vt = tvtString;
    String = ref.AsVariantStringNoAddRef();
    if(String) String->AddRef();
}

// 从二进制数据构造（Octet 类型）
tTJSVariant(const tjs_uint8 *ref, tjs_uint len) {
    vt = tvtOctet;
    if(ref) {
        Octet = TJSAllocVariantOctet(ref, len);
    } else {
        Octet = nullptr;
    }
}

// 从 bool 构造（Integer 类型）
tTJSVariant(bool ref) {
    vt = tvtInteger;
    Integer = (tjs_int64)(tjs_int)ref;  // true→1, false→0
}

// 从 32 位整数构造
tTJSVariant(tjs_int32 ref) {
    vt = tvtInteger;
    Integer = (tjs_int64)ref;  // 扩展为 64 位
}

// 从 64 位整数构造
tTJSVariant(tjs_int64 ref) {
    vt = tvtInteger;
    Integer = ref;
}

// 从浮点数构造
tTJSVariant(tjs_real ref) {
    vt = tvtReal;
    TJSSetFPUE();  // 设置浮点异常标志
    Real = ref;
}
```

**设计要点：**
- 值类型（Integer/Real/Bool）直接赋值，无内存分配
- 引用类型（Object/String/Octet）调用 AddRef 或分配新内存
- Bool 被存储为 Integer（0 或 1），TJS2 没有独立的布尔类型
- `TJSSetFPUE()` 在浮点数操作前调用，配置 FPU 异常处理模式

### 3.2 引用计数管理

```cpp
// 源码：cpp/core/tjs2/tjsVariant.h 第 480-527 行

// 对象引用管理（只处理 Object 类型）
void AddRefObject() {
    if(Object.Object) Object.Object->AddRef();
    if(Object.ObjThis) Object.ObjThis->AddRef();
}

void ReleaseObject() {
    iTJSDispatch2 *object = Object.Object;
    iTJSDispatch2 *objthis = Object.ObjThis;
    if(object) Object.Object = nullptr, object->Release();
    if(objthis) Object.ObjThis = nullptr, objthis->Release();
}

// 通用内容引用管理（处理所有引用类型）
void AddRefContent() {
    if(vt == tvtObject) {
        if(Object.Object) Object.Object->AddRef();
        if(Object.ObjThis) Object.ObjThis->AddRef();
    } else if(vt == tvtString) {
        if(String) String->AddRef();
    } else if(vt == tvtOctet) {
        if(Octet) Octet->AddRef();
    }
}

void ReleaseContent() {
    if(vt == tvtObject) {
        ReleaseObject();
    } else if(vt == tvtString) {
        if(String) String->Release();
    } else if(vt == tvtOctet) {
        if(Octet) Octet->Release();
    }
}
```

**`ReleaseObject` 的 null-then-release 模式**：

```cpp
if(object) Object.Object = nullptr, object->Release();
```

注意这里先将成员指针设为 nullptr，**然后**调用 Release。这是为了防止**释放过程中的重入**——如果 Release 触发了对象的析构，析构过程可能回调到当前对象并尝试读取 Object.Object，此时它已经是 nullptr 而不是一个悬空指针。

### 3.3 Clear 操作

```cpp
// 源码：cpp/core/tjs2/tjsVariant.h 第 677 行
void Clear() {
    ReleaseContent();  // 释放当前内容的引用
    vt = tvtVoid;      // 设为 void
}
```

`Clear()` 是一个安全的"重置"操作——先释放引用，再设置为 void。这与直接赋值 `vt = tvtVoid` 不同：直接赋值会跳过引用计数释放，导致内存泄漏。

---

## §4 tTJSVariantClosure — 对象闭包

TJS2 的对象系统有一个独特的设计：每个 Object 类型的 Variant 不只存储一个指针，而是存储一个**闭包**——由 `Object`（对象/函数本身）和 `ObjThis`（this 绑定）组成的配对。

### 4.1 为什么需要 ObjThis？

考虑这段 TJS2 脚本：

```javascript
class Foo {
    var name = "Foo";
    function greet() {
        return "Hello from " + this.name;
    }
}

var foo = new Foo();
var func = foo.greet;  // 从对象上取出方法

// 如果不绑定 this，func() 调用时 this 是谁？
func();  // 需要知道 this 是 foo
```

在 JavaScript 中，`var func = foo.greet` 会丢失 `this` 绑定（除非使用 `.bind()`）。TJS2 通过 `ObjThis` 机制解决这个问题——当从对象上读取方法时，返回的 Variant 中 `Object` 指向方法，`ObjThis` 指向对象本身。

### 4.2 ObjThis 的优先级链

```cpp
// 源码：cpp/core/tjs2/tjsVariant.h 第 222-228 行
tjs_error FuncCall(..., iTJSDispatch2 *objthis) const {
    if(!Object) TJSThrowNullAccess();
    return Object->FuncCall(
        flag, membername, hint, result, numparams, param,
        ObjThis ? ObjThis : (objthis ? objthis : Object));
        //  ↑ 优先级 1    ↑ 优先级 2    ↑ 优先级 3
}
```

**this 的三级优先级：**

| 优先级 | 来源 | 含义 |
|--------|------|------|
| 1（最高） | `ObjThis` | 闭包中绑定的 this（创建闭包时确定） |
| 2 | `objthis` 参数 | 调用方显式传入的 this（如 `foo.method()` 中 foo 作为 this） |
| 3（最低） | `Object` | 方法对象自身作为 this（兜底方案） |

这意味着一旦闭包绑定了 ObjThis，无论调用方传什么 this 都不会覆盖它——这类似于 JavaScript 的 `Function.prototype.bind()`。

### 4.3 SelectObjectNoAddRef

```cpp
iTJSDispatch2 *SelectObjectNoAddRef() {
    return ObjThis ? ObjThis : Object;
}
```

这个方法返回"实际的对象"——如果有 ObjThis（意味着 Object 是方法/属性），返回 ObjThis（方法所属的对象）；否则返回 Object 本身。用于需要获取"真正的对象实例"（而非方法引用）的场景。

### 4.4 tTJSVariantClosure 的方法代理

`tTJSVariantClosure` 继承 `tTJSVariantClosure_S`，为 `iTJSDispatch2` 的所有方法提供了代理包装：

```cpp
// 源码：cpp/core/tjs2/tjsVariant.h 第 179-445 行

class tTJSVariantClosure : public tTJSVariantClosure_S {
public:
    // FuncCall、PropGet、PropSet、CreateNew、DeleteMember、
    // Invalidate、IsValid、IsInstanceOf、Operation、EnumMembers
    // 等方法都遵循相同的模式：

    tjs_error SomeMethod(..., iTJSDispatch2 *objthis) const {
        if(!Object) TJSThrowNullAccess();  // null 检查
        return Object->SomeMethod(
            ...,
            ObjThis ? ObjThis : (objthis ? objthis : Object)
            // ObjThis 优先级最高
        );
    }
};
```

每个方法做三件事：
1. **空指针检查**：Object 为 null 时抛出 `TJSThrowNullAccess()`
2. **方法委托**：调用实际对象的方法
3. **this 绑定**：按优先级选择合适的 this

---

## §5 类型转换方法

TJS2 的类型转换分为两类：**隐式转换**（自动发生）和**显式转换**（通过 `AsXxx` 方法调用）。

### 5.1 AsObject — 转为对象

```cpp
// 源码：cpp/core/tjs2/tjsVariant.h 第 682-698 行

// 带引用计数的版本（调用者必须 Release）
iTJSDispatch2 *AsObject() const {
    if(vt == tvtObject) {
        if(Object.Object) Object.Object->AddRef();
        return Object.Object;
    }
    TJSThrowVariantConvertError(*this, tvtObject);
    return nullptr;
}

// 不增加引用计数的版本（调用者不能 Release）
iTJSDispatch2 *AsObjectNoAddRef() const {
    if(vt == tvtObject) return Object.Object;
    TJSThrowVariantConvertError(*this, tvtObject);
    return nullptr;
}
```

**`NoAddRef` 变体的使用场景**：当你只需要临时使用对象指针、且确保在使用期间 Variant 不会被销毁时，使用 `NoAddRef` 版本可以避免一次 AddRef/Release 配对的开销。这在 VM 内部的热路径上非常重要。

### 5.2 AsString — 转为字符串

```cpp
// 源码：cpp/core/tjs2/tjsVariant.h 第 755-775 行
tTJSVariantString *AsString() const {
    switch(vt) {
        case tvtVoid:     return nullptr;
        case tvtObject:   return TJSObjectToString(*(tTJSVariantClosure *)&Object);
        case tvtString:   if(String) String->AddRef(); return String;
        case tvtInteger:  return TJSIntegerToString(Integer);
        case tvtReal:     return TJSRealToString(Real);
        case tvtOctet:    TJSThrowVariantConvertError(*this, tvtString);
    }
    return nullptr;
}
```

**转换规则：**

| 源类型 | → 字符串 | 方法 |
|--------|----------|------|
| `tvtVoid` | `nullptr`（空） | 直接返回 |
| `tvtObject` | 对象的字符串表示 | `TJSObjectToString()` |
| `tvtString` | 自身 | 增加引用计数后返回 |
| `tvtInteger` | `"123"` | `TJSIntegerToString()` |
| `tvtReal` | `"3.14"` | `TJSRealToString()` |
| `tvtOctet` | **错误** | 抛出类型转换异常 |

注意 Octet（二进制数据）**不能**转为字符串——这是设计决策，防止二进制数据被意外当作文本处理。

### 5.3 AsInteger — 转为整数

```cpp
// 源码：cpp/core/tjs2/tjsVariant.h 第 862-879 行
tTVInteger AsInteger() const {
    switch(vt) {
        case tvtVoid:     return 0;           // void → 0
        case tvtObject:   TJSThrowVariantConvertError(...);  // 对象不能转整数
        case tvtString:   return String->ToInteger();  // 字符串解析为整数
        case tvtInteger:  return Integer;     // 直接返回
        case tvtReal:     return (tTVInteger)Real;  // 截断小数部分
        case tvtOctet:    TJSThrowVariantConvertError(...);  // 二进制不能转整数
    }
    return 0;
}
```

**转换规则：**

| 源类型 | → 整数 | 行为 |
|--------|--------|------|
| `tvtVoid` | `0` | 空值视为零 |
| `tvtObject` | **错误** | 对象无法自动转为数字 |
| `tvtString` | 解析结果 | `"123"` → `123`，`"abc"` → `0` |
| `tvtInteger` | 自身 | 无转换 |
| `tvtReal` | 截断 | `3.7` → `3` |
| `tvtOctet` | **错误** | 二进制数据无法转为数字 |

### 5.4 完整的类型转换矩阵

```
         → Void  Object  String  Integer  Real  Octet
Void       ✓      ✗       null    0        0     ✗
Object     -      ✓       toString ✗       ✗     ✗
String     -      ✗       ✓       parse    parse ✗
Integer    -      ✗       format  ✓        cast  ✗
Real       -      ✗       format  trunc    ✓     ✗
Octet      -      ✗       ✗       ✗        ✗     ✓

✓ = 直接/自身   ✗ = 抛异常   其他 = 转换方法
```

---

## §6 tTJSVariantOctet — 二进制数据类型

Octet 是 TJS2 的原生二进制数据类型，用于处理文件内容、网络数据等字节序列：

```cpp
// 源码：cpp/core/tjs2/tjsVariant.h 第 34-75 行

#pragma pack(push, 4)
struct tTJSVariantOctet_S {
    tjs_uint Length;      // 数据长度（字节数）
    tjs_int RefCount;     // 引用计数
    tjs_uint8 *Data;      // 数据指针
};
#pragma pack(pop)

class tTJSVariantOctet : protected tTJSVariantOctet_S {
public:
    // 从字节数组构造
    tTJSVariantOctet(const tjs_uint8 *data, tjs_uint length);

    // 拼接两段数据构造
    tTJSVariantOctet(const tjs_uint8 *data1, tjs_uint len1,
                     const tjs_uint8 *data2, tjs_uint len2);

    // 拼接两个 Octet 构造
    tTJSVariantOctet(const tTJSVariantOctet *o1, const tTJSVariantOctet *o2);

    void AddRef() { RefCount++; }
    void Release();  // RefCount-- 到 0 时释放

    tjs_uint GetLength() const { return Length; }
    const tjs_uint8 *GetData() const { return Data; }

    // 持久化支持
    tjs_int QueryPersistSize() { return sizeof(tjs_uint) + Length; }
    void Persist(tjs_uint8 *dest) {
        *(tjs_uint *)dest = Length;
        if(Data) TJS_octetcpy(dest + sizeof(tjs_uint), Data, Length);
    }
};
```

**设计特点：**
- **不可变**（immutable）：创建后数据内容不能修改，拼接操作产生新对象
- **引用计数**：多个 Variant 可以共享同一个 Octet 对象
- **持久化支持**：`QueryPersistSize` 和 `Persist` 方法支持将 Octet 序列化到字节码文件中

Octet 在 KrKr2 中的主要用途：
- 读取和处理二进制文件（图像、音频、归档）
- 网络通信数据
- 加密/解密操作的数据载体

---

## §7 赋值操作符

`tTJSVariant` 提供了丰富的赋值操作符，每个都遵循"释放旧值、设置新值"的模式：

```cpp
// 源码：cpp/core/tjs2/tjsVariant.h（赋值操作符部分）

// 从另一个 Variant 赋值
tTJSVariant &operator=(const tTJSVariant &ref) {
    // 1. 先增加新值的引用
    // 2. 释放旧值的引用
    // 3. 复制数据
    // （源码中实际实现在 .cpp 文件中，考虑了自赋值安全性）
}

// 从对象指针赋值
tTJSVariant &operator=(iTJSDispatch2 *ref) {
    if(ref) ref->AddRef();      // 增加新值引用
    ReleaseContent();            // 释放旧值
    vt = tvtObject;
    Object.Object = ref;
    Object.ObjThis = nullptr;
}

// 从字符串赋值
tTJSVariant &operator=(const tjs_char *ref) {
    ReleaseContent();
    vt = tvtString;
    if(ref) String = TJSAllocVariantString(ref);
    else String = nullptr;
}

// 从整数赋值
tTJSVariant &operator=(tjs_int64 ref) {
    ReleaseContent();
    vt = tvtInteger;
    Integer = ref;
}

// 从浮点数赋值
tTJSVariant &operator=(tjs_real ref) {
    ReleaseContent();
    vt = tvtReal;
    TJSSetFPUE();
    Real = ref;
}
```

**自赋值安全**：当 `a = a` 时（自赋值），必须先增加引用计数再释放，否则对象可能被提前释放：

```cpp
// 安全的自赋值模式
void assign(const tTJSVariant &ref) {
    ref.AddRefContent();    // 先加引用（如果 ref 和 *this 是同一对象，此时引用计数 ≥ 2）
    this->ReleaseContent(); // 再减引用（引用计数回到 ≥ 1，对象不被释放）
    BITCOPY(*this, ref);    // 复制数据
}

// 不安全的模式（BUG！）
void assign_unsafe(const tTJSVariant &ref) {
    this->ReleaseContent(); // 先减引用（如果自赋值，对象可能被释放！）
    ref.AddRefContent();    // 访问已释放的对象 → 未定义行为！
    BITCOPY(*this, ref);
}
```

---

## §8 比较操作

TJS2 有两种比较模式——**普通比较**和**严格比较**，类似于 JavaScript 的 `==` 和 `===`：

### 8.1 NormalCompare（普通比较）

```cpp
// 源码：cpp/core/tjs2/tjsVariant.h 第 661 行
bool NormalCompare(const tTJSVariant &val2) const;
```

普通比较会进行类型转换再比较：
- 整数和浮点数可以互相比较（`3 == 3.0` 为 true）
- 字符串和数字可以互相比较（`"123" == 123` 可能为 true）
- 对象通过指针比较

### 8.2 DiscernCompare（严格比较）

```cpp
// 源码：cpp/core/tjs2/tjsVariant.h 第 663 行
bool DiscernCompare(const tTJSVariant &val2) const;
```

严格比较**不进行类型转换**——类型不同直接返回 false。这对应 TJS2 脚本中的 `===` 运算符。

### 8.3 大小比较

```cpp
bool GreaterThan(const tTJSVariant &val2) const;  // >
bool LittlerThan(const tTJSVariant &val2) const;  // <
```

这两个函数在 VM 指令中的使用有一个有趣的细节（在 §02-VM指令集详解 中提到）：`VM_CLT`（compare less than）实际调用 `GreaterThan`，`VM_CGT`（compare greater than）实际调用 `LittlerThan`——操作数顺序是反转的。

---

## §9 动手实践

### 练习 1：模拟 tTJSVariant 的类型系统

```cpp
#include <iostream>
#include <string>
#include <cstring>
#include <variant>
#include <cassert>

// 模拟 TJS2 的 Variant 类型枚举
enum VariantType {
    vtVoid = 0,
    vtObject = 1,
    vtString = 2,
    vtOctet = 3,
    vtInteger = 4,
    vtReal = 5
};

// 模拟 tTJSVariant（简化版）
class SimpleVariant {
    union {
        int64_t integer_;
        double real_;
        std::string *string_;  // 拥有所有权
        void *object_;         // 简化的对象指针
    };
    VariantType type_;

public:
    SimpleVariant() : type_(vtVoid), integer_(0) {}

    // 从整数构造
    explicit SimpleVariant(int64_t val) : type_(vtInteger), integer_(val) {}

    // 从浮点数构造
    explicit SimpleVariant(double val) : type_(vtReal), real_(val) {}

    // 从字符串构造
    explicit SimpleVariant(const std::string &val) : type_(vtString) {
        string_ = new std::string(val);
    }

    // 复制构造
    SimpleVariant(const SimpleVariant &other) : type_(other.type_) {
        switch(type_) {
            case vtString:
                string_ = other.string_ ? new std::string(*other.string_) : nullptr;
                break;
            case vtInteger: integer_ = other.integer_; break;
            case vtReal:    real_ = other.real_; break;
            default:        integer_ = other.integer_; break;
        }
    }

    ~SimpleVariant() { clear(); }

    void clear() {
        if(type_ == vtString) delete string_;
        type_ = vtVoid;
        integer_ = 0;
    }

    VariantType type() const { return type_; }

    // 类型转换方法
    int64_t asInteger() const {
        switch(type_) {
            case vtVoid:    return 0;
            case vtInteger: return integer_;
            case vtReal:    return static_cast<int64_t>(real_);
            case vtString:  return string_ ? std::stoll(*string_) : 0;
            default:
                throw std::runtime_error("Cannot convert to integer");
        }
    }

    double asReal() const {
        switch(type_) {
            case vtVoid:    return 0.0;
            case vtInteger: return static_cast<double>(integer_);
            case vtReal:    return real_;
            case vtString:  return string_ ? std::stod(*string_) : 0.0;
            default:
                throw std::runtime_error("Cannot convert to real");
        }
    }

    std::string asString() const {
        switch(type_) {
            case vtVoid:    return "";
            case vtInteger: return std::to_string(integer_);
            case vtReal:    return std::to_string(real_);
            case vtString:  return string_ ? *string_ : "";
            default:
                throw std::runtime_error("Cannot convert to string");
        }
    }

    // 类型名称
    const char *typeName() const {
        static const char *names[] = {
            "void", "object", "string", "octet", "integer", "real"
        };
        return names[type_];
    }

    // 赋值
    SimpleVariant &operator=(const SimpleVariant &other) {
        if(this != &other) {
            clear();
            type_ = other.type_;
            switch(type_) {
                case vtString:
                    string_ = other.string_ ? new std::string(*other.string_) : nullptr;
                    break;
                case vtInteger: integer_ = other.integer_; break;
                case vtReal:    real_ = other.real_; break;
                default:        integer_ = other.integer_; break;
            }
        }
        return *this;
    }
};

int main() {
    std::cout << "=== TJS2 Variant 类型系统模拟 ===\n\n";

    // 1. 创建各种类型的值
    SimpleVariant v_void;
    SimpleVariant v_int(42LL);
    SimpleVariant v_real(3.14);
    SimpleVariant v_str(std::string("Hello TJS2"));

    std::cout << "v_void: type=" << v_void.typeName() << "\n";
    std::cout << "v_int:  type=" << v_int.typeName() << ", value=" << v_int.asInteger() << "\n";
    std::cout << "v_real: type=" << v_real.typeName() << ", value=" << v_real.asReal() << "\n";
    std::cout << "v_str:  type=" << v_str.typeName() << ", value=" << v_str.asString() << "\n";

    // 2. 类型转换
    std::cout << "\n--- 类型转换 ---\n";
    std::cout << "void → integer: " << v_void.asInteger() << "\n";    // 0
    std::cout << "int → string:   " << v_int.asString() << "\n";      // "42"
    std::cout << "real → integer: " << v_real.asInteger() << "\n";     // 3（截断）
    std::cout << "string → int:   " << SimpleVariant(std::string("123")).asInteger() << "\n";  // 123

    // 3. 复制语义
    std::cout << "\n--- 复制语义 ---\n";
    SimpleVariant v_copy = v_str;
    std::cout << "v_str:  " << v_str.asString() << "\n";
    std::cout << "v_copy: " << v_copy.asString() << "\n";

    // 修改原值不影响副本
    v_str = SimpleVariant(std::string("Modified"));
    std::cout << "修改后 v_str:  " << v_str.asString() << "\n";
    std::cout << "v_copy 不变:   " << v_copy.asString() << "\n";

    return 0;
}
// 编译: g++ -std=c++17 -o variant_test variant_test.cpp
```

### 练习 2：模拟 ObjThis 优先级链

```cpp
#include <iostream>
#include <string>
#include <functional>

// 简化的 iTJSDispatch2 模拟
class SimpleObject {
    std::string name_;
    int refCount_ = 1;
public:
    explicit SimpleObject(const std::string &name) : name_(name) {}
    std::string name() const { return name_; }
    void addRef() { refCount_++; }
    void release() {
        if(--refCount_ == 0) {
            std::cout << "  [释放 " << name_ << "]\n";
            delete this;
        }
    }
};

// 简化的 tTJSVariantClosure
struct Closure {
    SimpleObject *object = nullptr;   // 函数/方法对象
    SimpleObject *objThis = nullptr;  // 绑定的 this

    // 模拟 FuncCall 的 this 选择逻辑
    SimpleObject *selectThis(SimpleObject *callerThis) {
        // ObjThis 优先级最高
        if(objThis) {
            std::cout << "  [this 来源: ObjThis 绑定 (" << objThis->name() << ")]\n";
            return objThis;
        }
        // 然后是调用方传入的 this
        if(callerThis) {
            std::cout << "  [this 来源: 调用方传入 (" << callerThis->name() << ")]\n";
            return callerThis;
        }
        // 最后是 Object 自身
        if(object) {
            std::cout << "  [this 来源: Object 自身 (" << object->name() << ")]\n";
            return object;
        }
        std::cout << "  [this 来源: null]\n";
        return nullptr;
    }
};

int main() {
    std::cout << "=== ObjThis 优先级链演示 ===\n\n";

    auto method = new SimpleObject("greet方法");
    auto foo = new SimpleObject("Foo实例");
    auto bar = new SimpleObject("Bar实例");

    // 场景 1：无 ObjThis 绑定，调用方传入 this
    {
        std::cout << "场景 1: closure={method, null}, callerThis=foo\n";
        Closure c{method, nullptr};
        c.selectThis(foo);
        // 预期: callerThis（foo）
    }

    // 场景 2：有 ObjThis 绑定，调用方也传入 this
    {
        std::cout << "\n场景 2: closure={method, foo}, callerThis=bar\n";
        Closure c{method, foo};
        c.selectThis(bar);
        // 预期: ObjThis（foo），忽略 bar
    }

    // 场景 3：无 ObjThis，无 callerThis
    {
        std::cout << "\n场景 3: closure={method, null}, callerThis=null\n";
        Closure c{method, nullptr};
        c.selectThis(nullptr);
        // 预期: Object 自身（method）
    }

    // 场景 4：有 ObjThis，无 callerThis
    {
        std::cout << "\n场景 4: closure={method, foo}, callerThis=null\n";
        Closure c{method, foo};
        c.selectThis(nullptr);
        // 预期: ObjThis（foo）
    }

    std::cout << "\n=== 结论 ===\n";
    std::cout << "ObjThis > callerThis > Object\n";
    std::cout << "一旦闭包绑定了 ObjThis，调用方无法覆盖 this 绑定\n";

    // 手动释放（实际项目中通过引用计数自动管理）
    delete method;
    delete foo;
    delete bar;

    return 0;
}
// 编译: g++ -std=c++17 -o closure_test closure_test.cpp
```

### 练习 3：引用计数的 null-then-release 模式

```cpp
#include <iostream>
#include <string>

// 模拟引用计数对象
class RefCounted {
    std::string name_;
    int refCount_ = 1;
    bool destroying_ = false;

public:
    explicit RefCounted(const std::string &name) : name_(name) {
        std::cout << "[构造] " << name_ << " (refCount=1)\n";
    }

    void addRef() {
        refCount_++;
        std::cout << "[AddRef] " << name_ << " (refCount=" << refCount_ << ")\n";
    }

    void release() {
        refCount_--;
        std::cout << "[Release] " << name_ << " (refCount=" << refCount_ << ")\n";
        if(refCount_ == 0) {
            destroying_ = true;
            std::cout << "[析构] " << name_ << " 开始\n";
            // 析构过程中可能回调——模拟 TJS2 的 finalize
            onDestroy();
            std::cout << "[析构] " << name_ << " 完成\n";
            delete this;
        }
    }

    virtual void onDestroy() {
        // 子类可以在析构时执行回调
    }

    const std::string &name() const { return name_; }
    bool isDestroying() const { return destroying_; }
};

// 模拟 Variant 持有对象的安全释放
class VariantHolder {
    RefCounted *obj_ = nullptr;

public:
    void set(RefCounted *obj) {
        if(obj) obj->addRef();

        // 安全释放模式：先 null 再 release
        RefCounted *old = obj_;
        obj_ = nullptr;         // 先设为 null
        if(old) old->release(); // 再释放旧值

        obj_ = obj;
    }

    // 不安全的释放模式（用于对比）
    void setUnsafe(RefCounted *obj) {
        if(obj) obj->addRef();
        if(obj_) obj_->release();  // 如果 release 触发回调访问 obj_，会得到悬空指针！
        obj_ = obj;
    }

    RefCounted *get() const { return obj_; }

    ~VariantHolder() {
        if(obj_) obj_->release();
    }
};

int main() {
    std::cout << "=== 引用计数 null-then-release 模式 ===\n\n";

    auto obj = new RefCounted("MyObject");
    // refCount=1

    {
        VariantHolder holder;

        std::cout << "--- 设置对象 ---\n";
        holder.set(obj);
        // addRef → refCount=2
        // 旧值是 null → 不 release

        std::cout << "\n--- 替换为新对象 ---\n";
        auto obj2 = new RefCounted("NewObject");
        holder.set(obj2);
        // addRef obj2 → refCount=2
        // null旧指针 → release obj → refCount=1

        std::cout << "\n--- holder 析构 ---\n";
        // holder 析构 → release obj2 → refCount=1
    }

    std::cout << "\n--- 释放最后的引用 ---\n";
    obj->release();  // refCount=0 → 析构

    return 0;
}
// 编译: g++ -std=c++17 -o refcount_test refcount_test.cpp
```

### 练习 4：类型转换矩阵验证

```cpp
#include <iostream>
#include <string>
#include <functional>

enum VType { Void, Integer, Real, String, Object, Octet };

const char *typeName(VType t) {
    static const char *names[] = {"Void", "Integer", "Real", "String", "Object", "Octet"};
    return names[t];
}

// 模拟类型转换是否允许
bool canConvert(VType from, VType to) {
    // 转换矩阵（基于 TJS2 实际行为）
    static bool matrix[6][6] = {
        //       Void   Int    Real   Str    Obj    Oct
        /*Void*/ {true,  true,  true,  true,  false, false},
        /*Int*/  {false, true,  true,  true,  false, false},
        /*Real*/ {false, true,  true,  true,  false, false},
        /*Str*/  {false, true,  true,  true,  false, false},
        /*Obj*/  {false, false, false, true,  true,  false},
        /*Oct*/  {false, false, false, false, false, true},
    };
    return matrix[from][to];
}

int main() {
    std::cout << "=== TJS2 类型转换矩阵 ===\n\n";
    std::cout << "         ";
    for(int to = 0; to < 6; to++)
        printf("%-8s", typeName((VType)to));
    std::cout << "\n";

    for(int from = 0; from < 6; from++) {
        printf("%-8s ", typeName((VType)from));
        for(int to = 0; to < 6; to++) {
            if(from == to)
                printf("  ✓     ");
            else if(canConvert((VType)from, (VType)to))
                printf("  →     ");
            else
                printf("  ✗     ");
        }
        std::cout << "\n";
    }

    std::cout << "\n图例: ✓=自身  →=可转换  ✗=抛异常\n";

    // 验证特定转换
    std::cout << "\n--- 特定转换规则 ---\n";
    std::cout << "Void → Integer:  0（空值视为零）\n";
    std::cout << "Real → Integer:  截断小数部分（3.7 → 3）\n";
    std::cout << "String → Integer: 解析（\"123\" → 123）\n";
    std::cout << "Object → String:  调用 toString\n";
    std::cout << "Octet → String:   ✗ 禁止（防止二进制被当文本）\n";
    std::cout << "Object → Integer: ✗ 禁止（对象不能自动转数字）\n";

    return 0;
}
// 编译: g++ -std=c++17 -o type_matrix type_matrix.cpp
```

### 练习 5：BITCOPY 与正常赋值的区别

```cpp
#include <iostream>
#include <cstring>

// 模拟带引用计数的字符串
struct RefString {
    char data[32];
    int refCount;

    RefString(const char *s) : refCount(1) {
        strncpy(data, s, 31);
        data[31] = '\0';
        std::cout << "[String 构造] \"" << data << "\" refCount=1\n";
    }

    void addRef() {
        refCount++;
        std::cout << "[String AddRef] \"" << data << "\" refCount=" << refCount << "\n";
    }

    void release() {
        refCount--;
        std::cout << "[String Release] \"" << data << "\" refCount=" << refCount << "\n";
        if(refCount == 0) {
            std::cout << "[String 析构] \"" << data << "\"\n";
            // 在实际代码中这里会 delete this
        }
    }
};

// 简化的 Variant
struct SimpleVar {
    RefString *str;
    int type;  // 0=void, 2=string

    SimpleVar() : str(nullptr), type(0) {}
};

int main() {
    std::cout << "=== BITCOPY vs 正常赋值 ===\n\n";

    // 创建一个字符串 Variant
    RefString *s = new RefString("Hello");
    SimpleVar a;
    a.str = s;
    a.type = 2;
    // s->refCount = 1, a 持有引用

    // BITCOPY：直接内存复制，不调整引用计数
    std::cout << "--- BITCOPY（危险！）---\n";
    SimpleVar b;
    std::memcpy(&b, &a, sizeof(SimpleVar));
    // 此时 a 和 b 都指向同一个 RefString
    // 但 refCount 仍然是 1 → 如果 a 释放，b 变成悬空指针！
    std::cout << "BITCOPY 后: a.str=\"" << a.str->data << "\", b.str=\"" << b.str->data << "\"\n";
    std::cout << "refCount=" << s->refCount << "（应该是2，但BITCOPY不增加！）\n";

    // 正常赋值：调整引用计数
    std::cout << "\n--- 正常赋值（安全）---\n";
    SimpleVar c;
    c.str = s;
    c.type = 2;
    s->addRef();  // 引用计数 +1
    std::cout << "正常赋值后: refCount=" << s->refCount << "（正确：2）\n";

    // 清理
    std::cout << "\n--- 释放 ---\n";
    s->release();  // a 的释放
    s->release();  // c 的释放（b 是 BITCOPY 的，不应该再 release）

    return 0;
}
// 编译: g++ -std=c++17 -o bitcopy_test bitcopy_test.cpp
// 注意: 这个示例故意展示了 BITCOPY 的危险性
// 在实际 TJS2 代码中，BITCOPY 只在特定的"移动"场景中使用
// 比如将值从一个寄存器"移动"到另一个（旧位置不再使用）
```

---

## §10 对照项目源码

| 文件路径 | 行号范围 | 内容 |
|----------|----------|------|
| `cpp/core/tjs2/tjsVariant.h` | 103-113 | `tTJSVariantType` 枚举定义（6 种类型） |
| `cpp/core/tjs2/tjsVariant.h` | 125-128 | `tTJSVariantClosure_S` 结构（Object + ObjThis） |
| `cpp/core/tjs2/tjsVariant.h` | 136-153 | `tTJSVariant_S` 结构（union + type tag） |
| `cpp/core/tjs2/tjsVariant.h` | 140-143 | `tTJSVariant_BITCOPY` 宏 |
| `cpp/core/tjs2/tjsVariant.h` | 179-445 | `tTJSVariantClosure` 完整方法代理 |
| `cpp/core/tjs2/tjsVariant.h` | 480-527 | 引用计数管理（AddRefObject/ReleaseObject/AddRefContent/ReleaseContent） |
| `cpp/core/tjs2/tjsVariant.h` | 546-643 | 构造函数族（从各种类型构造） |
| `cpp/core/tjs2/tjsVariant.h` | 660-670 | 比较方法（NormalCompare/DiscernCompare/GreaterThan/LittlerThan） |
| `cpp/core/tjs2/tjsVariant.h` | 682-735 | AsObject 系列方法（4 个变体） |
| `cpp/core/tjs2/tjsVariant.h` | 755-899 | AsString/AsInteger/AsReal/AsOctet 类型转换方法 |
| `cpp/core/tjs2/tjsVariant.h` | 34-75 | `tTJSVariantOctet` 二进制数据类型 |
| `cpp/core/tjs2/tjsInterface.h` | 24-31 | 调用标志常量（TJS_MEMBERENSURE 等） |

---

## §11 本节小结

- **tTJSVariant** 是 TJS2 的统一值容器，使用 union + 类型标签实现 6 种动态类型
- **内存布局**：20 字节（16 字节 union + 4 字节 type tag），4 字节对齐
- **引用类型**（Object/String/Octet）使用引用计数管理生命周期，值类型（Void/Integer/Real）直接存储
- **tTJSVariantClosure** 是 Object 类型的核心——双指针设计（Object + ObjThis）实现自动 this 绑定
- **ObjThis 优先级**：闭包绑定 > 调用方传入 > 对象自身，一旦绑定不可覆盖
- **BITCOPY** 是不安全的逐位复制（跳过引用计数），只在 VM 内部的"移动"语义中使用
- **类型转换**：Void 可以转为数值类型（视为 0），Object/Octet 转为数值类型会抛异常，Octet 转字符串也会抛异常
- **null-then-release** 模式防止析构过程中的重入问题

---

## §12 练习题与答案

### 题目 1：tTJSVariant 的内存布局

以下 `tTJSVariant` 值各占多少内存？union 中哪些字段被使用？

```cpp
tTJSVariant a;                      // void
tTJSVariant b(42LL);                // integer
tTJSVariant c(3.14);                // real
tTJSVariant d(TJS_W("hello"));      // string
tTJSVariant e(someObject, nullptr); // object with no ObjThis
tTJSVariant f(someObject, fooObj);  // object with ObjThis
```

<details>
<summary>查看答案</summary>

所有 `tTJSVariant` 实例占用**相同的内存大小**——每个都是 `sizeof(tTJSVariant_S)` = 20 字节（在 64 位平台上，`#pragma pack(push, 4)` 下）。

Union 字段使用情况：

| 变量 | `vt` | union 使用的字段 | 内容 |
|------|------|------------------|------|
| `a` | `tvtVoid` | 无（union 内容未定义） | — |
| `b` | `tvtInteger` | `Integer` (8 bytes) | `42` |
| `c` | `tvtReal` | `Real` (8 bytes) | `3.14` |
| `d` | `tvtString` | `String` (8 bytes, 指针) | 指向 `tTJSVariantString` |
| `e` | `tvtObject` | `Object.Object` (8 bytes) + `Object.ObjThis` (8 bytes) | `{someObject, nullptr}` |
| `f` | `tvtObject` | `Object.Object` (8 bytes) + `Object.ObjThis` (8 bytes) | `{someObject, fooObj}` |

关键点：
- Union 大小由最大成员决定 = `tTJSVariantClosure_S`（16 字节，两个指针）
- `tvtInteger` 和 `tvtReal` 只使用 union 的前 8 字节，后 8 字节是垃圾数据
- `tvtString` 和 `tvtOctet` 也只使用 8 字节（一个指针）
- 只有 `tvtObject` 使用完整的 16 字节

</details>

### 题目 2：引用计数安全

以下代码有内存泄漏。找出并修复：

```cpp
tTJSVariant func(iTJSDispatch2 *obj) {
    tTJSVariant result;
    result = tTJSVariant(obj);  // 赋值

    // 做一些操作...

    result = tTJSVariant(42);   // 替换为整数
    return result;
}
```

<details>
<summary>查看答案</summary>

这段代码**没有内存泄漏**。让我们逐步分析引用计数：

1. `result = tTJSVariant(obj)`：
   - 临时 `tTJSVariant(obj)` 构造时 `obj->AddRef()`（引用计数 +1）
   - 赋值操作符 `operator=` 内部：先 `AddRefContent()` 新值（临时值的引用不需要再加），再 `ReleaseContent()` 旧值（void，不做任何事），然后复制
   - 临时对象析构时 `ReleaseContent()`，`obj->Release()`（引用计数 -1）
   - 净效果：`result` 持有 `obj` 的一次引用

2. `result = tTJSVariant(42)`：
   - 赋值操作符：`ReleaseContent()` 旧值 → `obj->Release()`（引用计数 -1）
   - 设置 `vt = tvtInteger, Integer = 42`
   - 净效果：`result` 不再持有 `obj`

但如果代码写成这样就**有泄漏**：

```cpp
tTJSVariant func(iTJSDispatch2 *obj) {
    tTJSVariant result;
    result.vt = tvtObject;        // 直接设置类型标签
    result.Object.Object = obj;   // 直接设置指针
    obj->AddRef();                // 增加引用
    
    // BUG：直接设置 vt 和 Integer 不会释放 obj 的引用
    result.vt = tvtInteger;
    result.Integer = 42;
    // obj 的引用没有被释放 → 泄漏！
    
    return result;
}
```

修复方式：使用 `Clear()` 或赋值操作符而不是直接操作内部字段：

```cpp
result.Clear();               // 先释放旧值
result = tTJSVariant(42);    // 安全赋值
```

</details>

### 题目 3：ObjThis 绑定行为

预测以下 TJS2 代码中每次 `greet()` 调用时的 `this` 对象：

```javascript
class Animal {
    var name;
    function Animal(n) { this.name = n; }
    function greet() { return this.name + " says hello"; }
}

var cat = new Animal("Cat");
var dog = new Animal("Dog");

// 调用 1：直接调用
cat.greet();

// 调用 2：取出方法再调用
var func = cat.greet;
func();

// 调用 3：用 dog 的 this 调用 cat 的方法
// （伪代码，实际 TJS2 没有直接的 call/apply）
```

<details>
<summary>查看答案</summary>

**调用 1：`cat.greet()`**

`this` = `cat` 对象。`cat.greet` 通过 `PropGet` 获取方法时，返回的 Variant 的 `Object` 指向 `greet` 方法，`ObjThis` 指向 `cat`。调用时使用 ObjThis 优先级最高的规则。
- 输出：`"Cat says hello"`

**调用 2：`var func = cat.greet; func();`**

`this` = `cat` 对象。当 `cat.greet` 被赋值给 `func` 时，`func` 这个 Variant 仍然保持 `ObjThis = cat` 的绑定。即使后续独立调用 `func()`，ObjThis 绑定不会丢失。
- 输出：`"Cat says hello"`

这与 JavaScript 不同！在 JavaScript 中 `var func = cat.greet; func()` 会丢失 this 绑定（this 变成 undefined 或 window）。TJS2 的 ObjThis 机制自动保留了绑定。

**调用 3：** 在 TJS2 中没有 `Function.call()` 或 `Function.apply()` 等方法来动态改变 this。一旦闭包绑定了 ObjThis，它就不能被外部覆盖。这是 TJS2 和 JavaScript 的一个重要差异。

</details>

---

## 下一步

下一节我们将深入 [iTJSDispatch2 接口](./02-iTJSDispatch2接口.md)，了解 TJS2 对象系统的核心接口设计——所有 TJS2 对象都通过这个接口与外部交互。
