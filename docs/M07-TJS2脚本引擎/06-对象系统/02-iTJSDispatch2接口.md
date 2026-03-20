# iTJSDispatch2 接口与对象实现

> **所属模块：** M07-TJS2脚本引擎
> **前置知识：** [tTJSVariant与类型系统](./01-tTJSVariant与类型系统.md)、[VM执行-ExecuteCode调度循环](../05-VM执行/01-ExecuteCode调度循环.md)
> **预计阅读时间：** 60 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 `iTJSDispatch2` 作为 TJS2 万物之源接口的设计思想和 14 对虚方法
2. 掌握错误码系统（`tjs_error`）和调用标志（`TJS_MEMBERENSURE` 等）的完整含义
3. 理解 `tTJSDispatch` 基类的默认实现和引用计数机制
4. 掌握 `tTJSCustomObject` 哈希表对象的内部结构——符号表、哈希桶、链式冲突、动态扩容
5. 理解 `missing` 机制和 `finalize` 生命周期钩子
6. 能够独立阅读和扩展 `tjsObject.h` / `tjsObject.cpp` 源码

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| Dispatch 接口 | Dispatch Interface | 一种"万能对象协议"——所有对象（函数、类、属性、字典等）都通过同一组方法来访问，调用方无需知道对象的具体类型 |
| 虚方法表 | Virtual Method Table (vtable) | C++ 编译器为每个有虚函数的类生成的函数指针数组，运行时通过它实现多态——调用派生类的真正实现 |
| 符号数据 | Symbol Data | 对象内部存储成员的基本单元，包含名字、哈希值、标志位和值（tTJSVariant），类似于 JavaScript 对象的属性描述符 |
| 哈希桶 | Hash Bucket | 哈希表中的一个槽位，通过 `hash & mask` 计算得到索引，多个符号可能映射到同一个桶（冲突），用链表串起来 |
| 成员确保 | Member Ensure | 一种调用标志——当访问不存在的成员时，不返回错误而是自动创建该成员（类似 JavaScript 给对象赋值新属性时自动创建） |
| 原生实例 | Native Instance | C++ 端创建的对象实例，通过 `NativeInstanceSupport` 与 TJS2 对象关联，实现脚本与原生代码的桥接 |
| 属性处理器 | Property Handler | 一种特殊对象——当读写它时，不是直接读写值，而是调用它的 PropGet/PropSet 方法（类似于 JavaScript 的 getter/setter） |

---

## §1 为什么需要统一的 Dispatch 接口

### 1.1 问题：TJS2 中"一切皆对象"

TJS2 脚本语言中，函数、类、普通对象、数组、字典、属性……这些看起来完全不同的东西，在 C++ 引擎内部需要统一管理。VM 执行字节码时，不知道（也不关心）操作数是函数还是字典——它只需要一组通用操作：

- 读取成员的值（PropGet）
- 设置成员的值（PropSet）
- 调用成员作为函数（FuncCall）
- 删除成员（DeleteMember）
- 枚举所有成员（EnumMembers）
- ……

这就是 **Dispatch 接口**（分派接口）的核心思想：用一个统一的 C++ 纯虚接口来定义所有对象必须支持的操作集合。任何 TJS2 对象——不管是脚本定义的类实例、原生 C++ 对象、还是内置的 Array——都必须实现这个接口。

### 1.2 类比：COM 的 IDispatch

如果你了解 Windows COM 编程，会发现 `iTJSDispatch2` 的设计与微软的 `IDispatch` 接口非常相似。这并非巧合——KiriKiri 最初是 Windows 平台的视觉小说引擎，其作者 W.Dee 深受 COM 设计影响：

| 概念 | COM IDispatch | TJS2 iTJSDispatch2 |
|------|--------------|-------------------|
| 获取属性 | `Invoke(DISPATCH_PROPERTYGET)` | `PropGet()` |
| 设置属性 | `Invoke(DISPATCH_PROPERTYPUT)` | `PropSet()` |
| 调用方法 | `Invoke(DISPATCH_METHOD)` | `FuncCall()` |
| 引用计数 | `AddRef()` / `Release()` | `AddRef()` / `Release()` |
| 按名称查找 | `GetIDsOfNames()` | 直接传 `membername` 字符串 |
| 错误码 | `HRESULT`（32位有符号整数） | `tjs_error`（32位有符号整数） |

但 `iTJSDispatch2` 比 COM 的 `IDispatch` 更丰富——它直接在接口层面提供了成员删除、对象无效化、类型检查、原生实例绑定等操作，不需要像 COM 那样通过额外的 `QueryInterface` 再获取其他接口。

### 1.3 继承体系总览

```
iTJSDispatch2               ← 纯虚接口（14 对方法 + 引用计数）
    └── tTJSDispatch        ← 默认基类（所有方法返回 E_NOTIMPL）
        ├── tTJSCustomObject     ← 哈希表对象（通用运行时对象）
        ├── tTJSInterCodeContext ← 编译后的函数/类
        ├── tTJSNativeFunction   ← 原生 C++ 函数
        ├── tTJSNativeClass      ← 原生 C++ 类
        ├── tTJSArrayObject      ← Array 内置对象
        ├── tTJSDictionaryObject ← Dictionary 内置对象
        └── ...其他内置对象
```

这个体系的核心设计原则是：**子类只重写自己需要的方法，其余全部继承基类的默认"不支持"行为**。例如 `tTJSNativeFunction` 只重写 `FuncCall()`，其他方法（`PropGet`、`DeleteMember` 等）直接返回 `TJS_E_NOTIMPL`。

---

## §2 错误码系统——tjs_error

在深入接口方法之前，必须先理解 TJS2 的错误码系统。每个 `iTJSDispatch2` 方法都返回 `tjs_error` 类型的值，这是一个 32 位有符号整数，定义在 `tjsErrorDefs.h` 中。

### 2.1 完整的错误码定义

```cpp
// 文件：cpp/core/tjs2/tjsErrorDefs.h

// ============ 成功码（≥ 0）============
#define TJS_S_OK    (0)    // 操作成功
#define TJS_S_TRUE  (1)    // 操作成功，逻辑结果为"真"
#define TJS_S_FALSE (2)    // 操作成功，逻辑结果为"假"
#define TJS_S_MAX   (2)    // 成功码的最大值（用于严格检查模式）

// ============ 失败码（< 0）============
#define TJS_E_MEMBERNOTFOUND  (-1001)  // 指定的成员不存在
#define TJS_E_NOTIMPL         (-1002)  // 该操作未实现（子类未重写）
#define TJS_E_INVALIDPARAM    (-1003)  // 传入参数无效
#define TJS_E_BADPARAMCOUNT   (-1004)  // 参数个数不正确
#define TJS_E_INVALIDTYPE     (-1005)  // 类型不匹配（如对非函数对象调用 FuncCall）
#define TJS_E_INVALIDOBJECT   (-1006)  // 对象已失效（被 Invalidate 过）
#define TJS_E_ACCESSDENYED    (-1007)  // 访问被拒绝（如写只读属性）
#define TJS_E_NATIVECLASSCRASH (-1008) // 原生类实例异常崩溃
#define TJS_E_FAIL            (-1)     // 通用失败（不属于以上任何类别）
```

### 2.2 成功/失败判断宏

```cpp
// 标准模式：负数 = 失败
#define TJS_FAILED(x)    ((x) < 0)
#define TJS_SUCCEEDED(x) (!TJS_FAILED(x))
```

注意 `TJS_S_TRUE`（1）和 `TJS_S_FALSE`（2）都是**成功码**。这是一个容易踩的坑——`TJS_S_FALSE` 表示"操作成功执行了，但逻辑结果为假"，而不是"操作失败了"。典型用法是 `IsValid()` 和 `IsInstanceOf()`：

```cpp
// 正确：用 TJS_S_TRUE/TJS_S_FALSE 表示布尔查询结果
tjs_error hr = obj->IsValid(0, nullptr, nullptr, objthis);
if (hr == TJS_S_TRUE) {
    // 对象有效
} else if (hr == TJS_S_FALSE) {
    // 对象已无效——但这不是"失败"，查询本身成功了
}
```

### 2.3 严格检查模式

定义 `TJS_STRICT_ERROR_CODE_CHECK` 宏后，`TJS_FAILED` 会变成函数版本，对大于 `TJS_S_MAX`（2）的返回值也视为失败。这个模式用于开发调试，帮助发现返回了非法错误码的 Bug：

```cpp
#ifdef TJS_STRICT_ERROR_CODE_CHECK
static inline bool TJS_FAILED(tjs_error hr) {
    if (hr < 0) return true;       // 负数 = 明确失败
    if (hr > TJS_S_MAX) return true; // 大于 2 = 非法的"成功"码
    return false;
}
#endif
```

### 2.4 TJSIsObjectValid 辅助函数

```cpp
// 判断对象是否有效（对 IsValid 返回值的包装）
inline bool TJSIsObjectValid(tjs_error hr) {
    if (hr == TJS_S_TRUE) return true;   // 对象明确有效
    if (hr == TJS_E_NOTIMPL) return true; // 未实现 IsValid 也视为有效
    return false;                         // 其他情况视为无效
}
```

这个函数有个微妙之处：**未实现 `IsValid()` 的对象被认为永远有效**。这保证了那些没有无效化概念的简单对象（如原生函数）不会被意外地判定为无效。

---

## §3 调用标志——控制方法行为的开关

每个 `iTJSDispatch2` 方法的第一个参数都是 `tjs_uint32 flag`，通过位运算组合多个标志来控制方法的行为。这些标志定义在 `tjsInterface.h` 中。

### 3.1 成员管理标志

```cpp
// 文件：cpp/core/tjs2/tjsInterface.h

#define TJS_MEMBERENSURE   0x00000200
// "成员确保"——如果访问的成员不存在，自动创建它
// 类似 JavaScript 中 obj.newProp = value 会自动创建 newProp

#define TJS_MEMBERMUSTEXIST 0x00000400
// "成员必须存在"——如果成员不存在，返回错误而不是尝试创建
// 与 TJS_MEMBERENSURE 相反

#define TJS_IGNOREPROP     0x00000800
// "忽略属性处理器"——直接读写存储的值，不触发 getter/setter
// 类似于绕过 JavaScript 的 Object.defineProperty

#define TJS_HIDDENMEMBER   0x00001000
// "隐藏成员"——设置后该成员在 EnumMembers 时默认不可见
// 类似于 JavaScript 的 Object.defineProperty({ enumerable: false })

#define TJS_STATICMEMBER   0x00002000
// "静态成员"——标记为类级别的成员，而非实例级别
```

### 3.2 枚举标志

```cpp
#define TJS_ENUM_NO_VALUE  0x00100000
// 枚举成员时不获取值，只获取名字和标志
// 用于 for-in 循环中只需要键名的场景，节省开销
```

### 3.3 标志的实际使用示例

```cpp
// 场景 1：VM 执行 obj.name = value 时
// 编译器生成的字节码带有 TJS_MEMBERENSURE 标志
obj->PropSet(TJS_MEMBERENSURE, "name", &hint, &value, objthis);
// 如果 "name" 不存在，就创建它；如果存在，就覆盖

// 场景 2：内部设置隐藏的类元数据
obj->PropSet(
    TJS_MEMBERENSURE | TJS_HIDDENMEMBER | TJS_IGNOREPROP,
    "__class_id__",
    nullptr,
    &classIdVariant,
    objthis
);
// 创建一个隐藏成员，直接赋值而不触发属性处理器

// 场景 3：高效枚举所有键名（不取值）
obj->EnumMembers(TJS_ENUM_NO_VALUE, &callback, objthis);
// callback 只接收 2 个参数（name, flags），不接收 value
```

### 3.4 符号标志与调用标志的映射

`tTJSCustomObject` 内部的符号数据（`tTJSSymbolData`）有自己的标志位，与外部的调用标志存在映射关系：

| 调用标志 | 值 | 内部符号标志 | 值 | 说明 |
|---------|-----|-------------|-----|------|
| `TJS_HIDDENMEMBER` | `0x1000` | `TJS_SYMBOL_HIDDEN` | `0x8` | 隐藏成员 |
| `TJS_STATICMEMBER` | `0x2000` | `TJS_SYMBOL_STATIC` | `0x10` | 静态成员 |
| — | — | `TJS_SYMBOL_USING` | `0x1` | 该槽位正在使用 |
| — | — | `TJS_SYMBOL_INIT` | `0x2` | 该槽位已初始化 |

在 `PropSet` 中，外部标志被转换为内部符号标志：

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 1385-1393 行
if (flag & TJS_HIDDENMEMBER)
    data->SymFlags |= TJS_SYMBOL_HIDDEN;   // 设置隐藏
else
    data->SymFlags &= ~TJS_SYMBOL_HIDDEN;  // 清除隐藏

if (flag & TJS_STATICMEMBER)
    data->SymFlags |= TJS_SYMBOL_STATIC;   // 设置静态
else
    data->SymFlags &= ~TJS_SYMBOL_STATIC;  // 清除静态
```

---

## §4 iTJSDispatch2 接口——14 对虚方法详解

`iTJSDispatch2` 定义在 `tjsInterface.h` 的第 105-323 行。它是一个纯虚类（C++ abstract class），要求子类实现所有方法。下面逐一解析每对方法。

### 4.1 引用计数：AddRef / Release

```cpp
virtual tjs_uint AddRef() = 0;   // 增加引用计数，返回新计数值
virtual tjs_uint Release() = 0;  // 减少引用计数，返回新计数值；减到 0 时删除对象
```

这是最基本的生命周期管理。TJS2 不使用垃圾回收（Garbage Collection，一种自动内存管理方式——程序运行时自动发现不再使用的对象并释放），而是使用**手动引用计数**。每当一个新的指针指向对象时调用 `AddRef()`，指针不再使用时调用 `Release()`。

`tTJSDispatch` 基类的 `Release()` 实现有一个精妙之处：

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 110-130 行
tjs_uint tTJSDispatch::Release() {
    if (RefCount == 1) {
        // 引用计数即将归零——先调用析构前钩子
        if (!BeforeDestructionCalled) {
            BeforeDestructionCalled = true;
            BeforeDestruction();  // ← 可能会再次 AddRef！
        }
        if (RefCount == 1) {  // ← 再次检查：BeforeDestruction 可能增加了引用
            delete this;
            return 0;
        }
    }
    return --RefCount;
}
```

为什么要**两次检查** `RefCount == 1`？因为 `BeforeDestruction()` 钩子中可能会执行脚本代码（如 `finalize` 方法），脚本可能会创建新的对该对象的引用（`AddRef`），使得对象"复活"。如果不做二次检查，就会在对象仍被引用时错误地 `delete`。

### 4.2 函数调用：FuncCall / FuncCallByNum

```cpp
virtual tjs_error FuncCall(
    tjs_uint32 flag,           // 调用标志
    const tjs_char *membername, // 成员名（nullptr 表示调用对象自身）
    tjs_uint32 *hint,          // 名称哈希缓存（加速后续查找）
    tTJSVariant *result,       // 返回值存放位置（nullptr 表示丢弃返回值）
    tjs_int numparams,         // 参数个数
    tTJSVariant **param,       // 参数数组
    iTJSDispatch2 *objthis     // this 对象
) = 0;

virtual tjs_error FuncCallByNum(
    tjs_uint32 flag,
    tjs_int num,               // 用整数索引代替字符串名称
    tTJSVariant *result,
    tjs_int numparams,
    tTJSVariant **param,
    iTJSDispatch2 *objthis
) = 0;
```

**membername 为 nullptr 的含义**：当 `membername` 传入 `nullptr` 时，表示"调用这个对象本身作为函数"。例如：

```javascript
// TJS2 脚本
var func = function() { return 42; };
func();  // ← 此时 membername == nullptr，调用对象自身
```

**membername 不为 nullptr 的含义**：表示"在这个对象中查找名为 membername 的成员，并调用它"：

```javascript
// TJS2 脚本
var obj = %[ greet: function() { return "hello"; } ];
obj.greet();  // ← membername == "greet"
```

**hint 参数**：这是一个性能优化设计。首次查找时，`hint` 指向的值为 0；`Find()` 函数会计算哈希值并写入 `*hint`；后续对同一名称的查找可以跳过哈希计算，直接用缓存的值。在热循环中这能显著减少字符串哈希的开销。

**ByNum 变体**：所有 `ByNum` 方法在基类 `tTJSDispatch` 中的默认实现都是将整数转为字符串再调用字符串版本：

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 133-141 行
tjs_error tTJSDispatch::FuncCallByNum(tjs_uint32 flag, tjs_int num,
                                      tTJSVariant *result, tjs_int numparams,
                                      tTJSVariant **param, iTJSDispatch2 *objthis) {
    tjs_char buf[34];
    TJS_int_to_str(num, buf);  // 把整数转成字符串："0", "1", "2", ...
    return FuncCall(flag, buf, nullptr, result, numparams, param, objthis);
}
```

这意味着 `Array[0]` 和 `Array["0"]` 在底层是等价的——这也是 TJS2 与 JavaScript 的相似之处。

### 4.3 属性读取：PropGet / PropGetByNum

```cpp
virtual tjs_error PropGet(
    tjs_uint32 flag,
    const tjs_char *membername,
    tjs_uint32 *hint,
    tTJSVariant *result,       // 读取结果存放位置
    iTJSDispatch2 *objthis
) = 0;
```

**属性处理器的透明拦截**：当读取的成员值是一个 `tvtObject` 类型、且未设置 `TJS_IGNOREPROP` 标志时，`tTJSCustomObject::PropGet` 会尝试调用该对象的 `PropGet(nullptr)` 方法（即默认成员调用）。如果成功，返回属性处理器的结果而不是对象本身：

```cpp
// cpp/core/tjs2/tjsObject.cpp TJSDefaultPropGet 简化逻辑
tjs_error TJSDefaultPropGet(tjs_uint32 flag, tTJSVariant &targ,
                            tTJSVariant *result, iTJSDispatch2 *objthis) {
    if (!(flag & TJS_IGNOREPROP) && targ.Type() == tvtObject) {
        // 尝试调用目标对象的 PropGet(nullptr)
        tjs_error hr = targ.AsObjectClosure().Object->PropGet(
            0, nullptr, nullptr, result, objthis);
        if (TJS_SUCCEEDED(hr)) return hr;  // 属性处理器成功拦截
        // 如果返回 E_NOTIMPL/E_INVALIDTYPE，则回退到直接返回值
    }
    result->CopyRef(targ);  // 直接返回存储的值
    return TJS_S_OK;
}
```

这就是 TJS2 属性（property）的实现原理——把一个带有 PropGet/PropSet 实现的对象存为成员值，读写该成员时自动触发拦截。

### 4.4 属性设置：PropSet / PropSetByNum / PropSetByVS

```cpp
virtual tjs_error PropSet(
    tjs_uint32 flag,
    const tjs_char *membername,
    tjs_uint32 *hint,
    const tTJSVariant *param,  // 要设置的新值
    iTJSDispatch2 *objthis
) = 0;

virtual tjs_error PropSetByVS(
    tjs_uint32 flag,
    tTJSVariantString *membername,  // 使用 VariantString 而不是裸字符串
    const tTJSVariant *param,
    iTJSDispatch2 *objthis
) = 0;
```

`PropSetByVS` 是一个特殊的优化版本——它接受 `tTJSVariantString` 指针而不是 `const tjs_char*`。`tTJSVariantString` 自带引用计数和预计算的哈希值，可以避免重复的字符串拷贝和哈希计算。字节码加载器在加载常量池中的字符串后，优先使用这个版本。

**循环引用保护**：`tTJSCustomObject::PropSet` 在赋值前后会调用 `CheckObjectClosureRemove` 和 `CheckObjectClosureAdd`：

```cpp
// cpp/core/tjs2/tjsObject.h 第 379-395 行
void CheckObjectClosureAdd(const tTJSVariant &val) {
    // 如果新值的闭包 ObjThis 指向自身，减少引用计数
    if (val.Type() == tvtObject) {
        iTJSDispatch2 *dsp = val.AsObjectClosureNoAddRef().ObjThis;
        if (dsp == (iTJSDispatch2*)this)
            this->Release();  // 防止循环引用导致内存泄漏
    }
}

void CheckObjectClosureRemove(const tTJSVariant &val) {
    // 移除旧值前，如果旧值的 ObjThis 指向自身，恢复引用计数
    if (val.Type() == tvtObject) {
        iTJSDispatch2 *dsp = val.AsObjectClosureNoAddRef().ObjThis;
        if (dsp == (iTJSDispatch2*)this)
            this->AddRef();   // 恢复之前被减掉的计数
    }
}
```

这是 TJS2 解决循环引用的一种特殊手段：当对象的成员闭包的 `ObjThis` 指向对象自身时（极其常见——方法的 `this` 就是所属对象），额外的引用计数会被"中和"掉。否则，每个方法都会持有一个指向所属对象的引用，对象永远无法释放。

### 4.5 成员计数：GetCount / GetCountByNum

```cpp
virtual tjs_error GetCount(
    tjs_int *result,           // 输出：成员数量
    const tjs_char *membername,
    tjs_uint32 *hint,
    iTJSDispatch2 *objthis
) = 0;
```

当 `membername` 为 `nullptr` 时，返回对象自身的成员总数。在 `tTJSCustomObject` 中，直接返回 `Count` 字段的值：

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 1431-1444 行
tjs_error tTJSCustomObject::GetCount(tjs_int *result,
                                     const tjs_char *membername,
                                     tjs_uint32 *hint,
                                     iTJSDispatch2 *objthis) {
    if (!GetValidity()) return TJS_E_INVALIDOBJECT;
    if (!result) return TJS_E_INVALIDPARAM;
    *result = Count;
    return TJS_S_OK;
}
```

### 4.6 成员枚举：EnumMembers

```cpp
virtual tjs_error EnumMembers(
    tjs_uint32 flag,
    tTJSVariantClosure *callback,  // 回调闭包，每个成员调用一次
    iTJSDispatch2 *objthis
) = 0;
```

回调函数接收 2 或 3 个参数（取决于是否设置了 `TJS_ENUM_NO_VALUE`）：

```
callback(name, flags)          // TJS_ENUM_NO_VALUE 模式
callback(name, flags, value)   // 正常模式
```

回调返回非零值表示继续枚举，返回零表示停止。`InternalEnumMembers` 遍历哈希表的每个桶及其链表，对每个 `TJS_SYMBOL_USING` 的符号调用回调。

### 4.7 成员删除：DeleteMember / DeleteMemberByNum

```cpp
virtual tjs_error DeleteMember(
    tjs_uint32 flag,
    const tjs_char *membername,
    tjs_uint32 *hint,
    iTJSDispatch2 *objthis
) = 0;
```

对应 TJS2 脚本的 `delete obj.member` 语法。`tTJSCustomObject::DeleteMember` 调用 `DeleteByName` 从哈希表中移除符号数据。删除一个链上的节点（而非桶头）时，需要将其从链表中断开（`prevd->Next = d->Next`）并 `delete d`；删除桶头节点时，只做 `PostClear()` 清除数据但保留槽位。

### 4.8 对象无效化：Invalidate / InvalidateByNum

```cpp
virtual tjs_error Invalidate(
    tjs_uint32 flag,
    const tjs_char *membername,
    tjs_uint32 *hint,
    iTJSDispatch2 *objthis
) = 0;
```

"无效化"（Invalidation）是 TJS2 对象生命周期中的一个重要概念——它不同于销毁。无效化标记对象为"已废弃"，后续对该对象的操作会返回 `TJS_E_INVALIDOBJECT`。

当 `membername` 为 `nullptr` 时，无效化对象自身：

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 1572-1584 行
tjs_error tTJSCustomObject::Invalidate(tjs_uint32 flag,
                                       const tjs_char *membername,
                                       tjs_uint32 *hint,
                                       iTJSDispatch2 *objthis) {
    if (!GetValidity()) return TJS_E_INVALIDOBJECT;
    if (membername == nullptr) {
        if (IsInvalidated) return TJS_S_FALSE;  // 已经无效化过了
        _Finalize();                            // 调用 finalize 逻辑
        return TJS_S_TRUE;                      // 首次无效化成功
    }
    // membername 不为 nullptr：无效化某个成员持有的对象
    // ...
}
```

`_Finalize()` 的执行流程：
1. 设置 `IsInvalidating = true`（防止重入）
2. 调用虚函数 `Finalize()`
3. `Finalize()` 调用对象的 `"finalize"` 方法（TJS2 脚本层的析构钩子）
4. 对所有已注册的 `iTJSNativeInstance` 调用 `Invalidate()`
5. 调用 `DeleteAllMembers()` 清除所有成员

### 4.9 有效性检查：IsValid / IsValidByNum

```cpp
virtual tjs_error IsValid(
    tjs_uint32 flag,
    const tjs_char *membername,
    tjs_uint32 *hint,
    iTJSDispatch2 *objthis
) = 0;
```

返回 `TJS_S_TRUE` 或 `TJS_S_FALSE`（注意不是返回 `bool`）。当 `membername` 不为 `nullptr` 时，检查的是该成员持有的对象是否有效。

### 4.10 对象创建：CreateNew / CreateNewByNum

```cpp
virtual tjs_error CreateNew(
    tjs_uint32 flag,
    const tjs_char *membername,
    tjs_uint32 *hint,
    iTJSDispatch2 **result,    // 输出：新创建的对象
    tjs_int numparams,
    tTJSVariant **param,
    iTJSDispatch2 *objthis
) = 0;
```

对应 TJS2 的 `new ClassName()` 语法。当 `membername` 为 `nullptr` 时，表示"用这个对象作为构造函数来创建新实例"。`tTJSCustomObject` 本身不是构造函数，所以对 `membername == nullptr` 返回 `TJS_E_INVALIDTYPE`。

### 4.11 类型检查：IsInstanceOf / IsInstanceOfByNum

```cpp
virtual tjs_error IsInstanceOf(
    tjs_uint32 flag,
    const tjs_char *membername,
    tjs_uint32 *hint,
    const tjs_char *classname,  // 要检查的类名
    iTJSDispatch2 *objthis
) = 0;
```

返回 `TJS_S_TRUE` 或 `TJS_S_FALSE`。`tTJSCustomObject::IsInstanceOf` 的实现：

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 1759-1781 行
tjs_error tTJSCustomObject::IsInstanceOf(tjs_uint32 flag,
                                         const tjs_char *membername,
                                         tjs_uint32 *hint,
                                         const tjs_char *classname,
                                         iTJSDispatch2 *objthis) {
    if (!GetValidity()) return TJS_E_INVALIDOBJECT;

    if (membername == nullptr) {
        // 检查对象自身是否是 classname 的实例
        if (!TJS_strcmp(classname, TJS_W("Object")))
            return TJS_S_TRUE;  // 一切皆 Object

        // 在 ClassNames 列表中查找
        for (tjs_uint i = 0; i < ClassNames.size(); i++) {
            if (!TJS_strcmp(ClassNames[i].c_str(), classname))
                return TJS_S_TRUE;
        }
        return TJS_S_FALSE;
    }
    // membername 不为 nullptr：检查成员持有的对象
    // ...
}
```

`ClassNames` 是一个 `std::vector<ttstr>`，在对象被构造时通过 `ClassInstanceInfo(TJS_CII_ADD, ...)` 注册类名。注册顺序是从子类到父类（descendant to ancestor），所以 `ClassNames[0]` 总是最具体的类名。

### 4.12 成员操作：Operation / OperationByNum

```cpp
virtual tjs_error Operation(
    tjs_uint32 flag,           // 低位存放操作类型（TJS_OP_ADD 等）
    const tjs_char *membername,
    tjs_uint32 *hint,
    tTJSVariant *result,
    const tTJSVariant *param,
    iTJSDispatch2 *objthis
) = 0;
```

`Operation` 对应 `+=`、`-=`、`++`、`--` 等就地修改操作。操作类型通过 `flag & TJS_OP_MASK` 提取：

```cpp
// 文件：cpp/core/tjs2/tjsInterface.h
#define TJS_OP_BAND  0x0001  // &=
#define TJS_OP_BOR   0x0002  // |=
#define TJS_OP_BXOR  0x0003  // ^=
#define TJS_OP_SUB   0x0004  // -=
#define TJS_OP_ADD   0x0005  // +=
#define TJS_OP_MOD   0x0006  // %=
#define TJS_OP_DIV   0x0007  // /=
#define TJS_OP_IDIV  0x0008  // \= （整数除法）
#define TJS_OP_MUL   0x0009  // *=
#define TJS_OP_LOR   0x000a  // ||=
#define TJS_OP_LAND  0x000b  // &&=
#define TJS_OP_SAR   0x000c  // >>=
#define TJS_OP_SAL   0x000d  // <<=
#define TJS_OP_SR    0x000e  // >>>=（无符号右移赋值）
#define TJS_OP_INC   0x000f  // ++
#define TJS_OP_DEC   0x0010  // --
```

`Operation` 的实现流程（在 `tTJSCustomObject` 中）：

1. 查找成员
2. 如果成员是属性处理器（`tvtObject`），通过 `PropGet` 取出当前值
3. 对取出的值执行运算（`TJSDoVariantOperation`）
4. 通过 `PropSet` 写回运算结果
5. 如果不是属性处理器，直接在值上就地修改

### 4.13 原生实例支持：NativeInstanceSupport

```cpp
virtual tjs_error NativeInstanceSupport(
    tjs_uint32 flag,          // TJS_NIS_REGISTER 或 TJS_NIS_GETINSTANCE
    tjs_int32 classid,        // 原生类的唯一 ID
    iTJSNativeInstance **pointer  // 原生实例指针
) = 0;
```

这是 TJS2 与 C++ 原生代码桥接的核心机制。每个原生类有一个全局唯一的 `classid`，通过这个 ID 可以在 TJS2 对象上注册和检索 C++ 对象实例。

**注册原生实例**（`TJS_NIS_REGISTER`）：

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 1938-1967 行
tjs_error tTJSCustomObject::NativeInstanceSupport(tjs_uint32 flag,
                                                  tjs_int32 classid,
                                                  iTJSNativeInstance **pointer) {
    if (flag == TJS_NIS_REGISTER) {
        // 在 ClassIDs/ClassInstances 数组中找到空位
        for (tjs_int i = 0; i < TJS_MAX_NATIVE_CLASS; i++) {
            if (ClassIDs[i] == -1) {
                ClassIDs[i] = classid;
                ClassInstances[i] = *pointer;
                return TJS_S_OK;
            }
        }
        return TJS_E_FAIL;  // 4 个槽位已满！
    }
    // TJS_NIS_GETINSTANCE: 按 classid 查找并返回实例指针
    // ...
}
```

**TJS_MAX_NATIVE_CLASS = 4**：每个 TJS2 对象最多关联 4 个不同原生类的实例。这个数字看起来很小，但实际上大多数对象只需要 1-2 个原生实例（自身的类 + 可能的一个父类）。如果 4 个槽位不够用，`NativeInstanceSupport` 返回 `TJS_E_FAIL`。

### 4.14 类实例信息：ClassInstanceInfo

```cpp
virtual tjs_error ClassInstanceInfo(
    tjs_uint32 flag,    // 操作类型
    tjs_uint num,       // 索引或无用
    tTJSVariant *value  // 输入/输出值
) = 0;
```

`flag` 参数的可能值：

```cpp
#define TJS_CII_ADD             0  // 添加类名到 ClassNames 列表
#define TJS_CII_GET             1  // 获取第 num 个类名
#define TJS_CII_SET_FINALIZE    2  // 设置 finalize 方法名
#define TJS_CII_SET_MISSING     3  // 设置 missing 方法名
#define TJS_CII_SET_SUPRECLASS  4  // 设置父类（预留，未实现）
#define TJS_CII_GET_SUPRECLASS  5  // 获取父类（预留，未实现）
```

**TJS_CII_ADD** 是最常用的——在构造函数中，脚本引擎会为新对象注册所有继承链上的类名：

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 1973-1984 行
case TJS_CII_ADD: {
    ttstr name = value->AsStringNoAddRef();
    if (TJSObjectHashMapEnabled() && ClassNames.size() == 0)
        TJSObjectHashSetType(this, TJS_W("instance of class ") + name);
    // 第一个类名 = 最具体的子类名（因为注册顺序从子到父）
    ClassNames.push_back(name);
    return TJS_S_OK;
}
```

**TJS_CII_SET_MISSING** 允许自定义"找不到成员时"的处理方法名：

```cpp
case TJS_CII_SET_MISSING: {
    missing_name = *value;
    CallMissing = !missing_name.IsEmpty();  // 非空时启用 missing 机制
    return TJS_S_OK;
}
```

### 4.15 保留方法：Reserved1 / Reserved2 / Reserved3

```cpp
virtual tjs_error Reserved1() { return TJS_E_NOTIMPL; }
virtual tjs_error Reserved2() { return TJS_E_NOTIMPL; }
virtual tjs_error Reserved3() { return TJS_E_NOTIMPL; }
```

这三个方法是虚表中预留的空位，用于未来扩展接口而不破坏二进制兼容性（ABI compatibility，即"应用二进制接口兼容性"——确保已编译的插件 DLL 不需要重新编译就能继续使用）。由于 `iTJSDispatch2` 的虚表布局必须在所有编译单元中一致，如果直接在中间插入新虚方法，所有已编译的插件都会崩溃。这些 Reserved 方法占住了虚表的位置，未来可以安全地赋予它们新的含义。

---

## §5 tTJSDispatch 基类——"什么都不做"的默认实现

`tTJSDispatch`（定义在 `tjsObject.h` 第 85-233 行）是 `iTJSDispatch2` 的第一个具体实现类。它的设计理念非常简单：**为所有 14 对方法提供"不支持"的默认实现**，子类只需要重写自己需要的方法。

### 5.1 核心字段

```cpp
// cpp/core/tjs2/tjsObject.h 第 88-93 行
class tTJSDispatch : public iTJSDispatch2 {
private:
    tjs_uint RefCount;              // 引用计数（初始为 1）
    bool BeforeDestructionCalled;   // BeforeDestruction 是否已调用（防止重复调用）
};
```

仅有两个字段——引用计数和一个布尔标志。`tTJSDispatch` 本身不存储任何成员数据，因为它只是一个"骨架"。

### 5.2 默认方法实现模式

每个方法的默认实现遵循统一的模式——根据 `membername` 是否为 `nullptr` 返回不同的错误码：

```cpp
// 以 FuncCall 为例（tjsObject.h 第 101-112 行）
tjs_error FuncCall(tjs_uint32 flag, const tjs_char *membername,
                   tjs_uint32 *hint, tTJSVariant *result,
                   tjs_int numparams, tTJSVariant **param,
                   iTJSDispatch2 *objthis) override {
    return TJS_E_NOTIMPL;  // "我不支持被当作函数调用"
}

// 以 IsInstanceOf 为例（tjsObject.h 第 201-205 行）
tjs_error IsInstanceOf(tjs_uint32 flag, const tjs_char *membername,
                       tjs_uint32 *hint, const tjs_char *classname,
                       iTJSDispatch2 *objthis) override {
    return membername ? TJS_E_MEMBERNOTFOUND : TJS_E_NOTIMPL;
    // membername 不为空 → "找不到这个成员"
    // membername 为空   → "我不支持类型检查"
}
```

### 5.3 ByNum 方法的统一转发

所有 10 个 `ByNum` 方法都在 `tjsObject.cpp` 中实现，采用完全相同的模式——把整数转成字符串，然后转发给字符串版本：

```cpp
// 通用模式（以 PropGetByNum 为例）
tjs_error tTJSDispatch::PropGetByNum(tjs_uint32 flag, tjs_int num,
                                     tTJSVariant *result, iTJSDispatch2 *objthis) {
    tjs_char buf[34];           // 足以容纳 64 位整数的十进制表示
    TJS_int_to_str(num, buf);   // 整数 → 字符串
    return PropGet(flag, buf, nullptr, result, objthis);
}
```

`buf[34]` 的大小看起来像一个魔术数字，但 64 位有符号整数的十进制表示最多 20 位（`-9223372036854775808`），加上负号和结尾的 `\0` 是 21 字节；34 字节（17 个宽字符）提供了足够的余量。

### 5.4 Operation 的基类实现

`tTJSDispatch::Operation` 与其他方法不同——它不是简单地返回 `TJS_E_NOTIMPL`，而是有实际的通用逻辑：

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 224-253 行
tjs_error tTJSDispatch::Operation(tjs_uint32 flag,
                                  const tjs_char *membername,
                                  tjs_uint32 *hint, tTJSVariant *result,
                                  const tTJSVariant *param,
                                  iTJSDispatch2 *objthis) {
    tjs_uint32 op = flag & TJS_OP_MASK;
    // 参数校验...

    tTJSVariant tmp;
    tjs_error hr;
    hr = PropGet(0, membername, hint, &tmp, objthis);  // 1. 先读当前值
    if (TJS_FAILED(hr)) return hr;

    TJSDoVariantOperation(op, tmp, param);             // 2. 执行运算

    hr = PropSet(0, membername, hint, &tmp, objthis);  // 3. 写回结果
    if (TJS_FAILED(hr)) return hr;

    if (result) result->CopyRef(tmp);                  // 4. 返回结果
    return TJS_S_OK;
}
```

这意味着只要子类实现了 `PropGet` 和 `PropSet`，`Operation` 就自动可用——不需要子类额外重写。这是一个优秀的"模板方法"设计模式（Template Method Pattern——基类定义算法骨架，子类填充具体步骤）的实例。

### 5.5 TJSDoVariantOperation——运算分派

`Operation` 中的核心运算由 `TJSDoVariantOperation` 完成，它是一个简单的 switch 分派：

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 26-78 行
void TJSDoVariantOperation(tjs_int op, tTJSVariant &target,
                           const tTJSVariant *param) {
    switch(op) {
        case TJS_OP_BAND: target &= (*param); return;   // 按位与
        case TJS_OP_BOR:  target |= (*param); return;   // 按位或
        case TJS_OP_BXOR: target ^= (*param); return;   // 按位异或
        case TJS_OP_SUB:  target -= (*param); return;   // 减法
        case TJS_OP_ADD:  target += (*param); return;   // 加法
        case TJS_OP_MOD:  target %= (*param); return;   // 取模
        case TJS_OP_DIV:  target /= (*param); return;   // 除法
        case TJS_OP_IDIV: target.idivequal(*param); return; // 整数除法
        case TJS_OP_MUL:  target *= (*param); return;   // 乘法
        case TJS_OP_LOR:  target.logicalorequal(*param); return;  // 逻辑或
        case TJS_OP_LAND: target.logicalandequal(*param); return; // 逻辑与
        case TJS_OP_SAR:  target >>= (*param); return;  // 算术右移
        case TJS_OP_SAL:  target <<= (*param); return;  // 算术左移
        case TJS_OP_SR:   target.rbitshiftequal(*param); return; // 无符号右移
        case TJS_OP_INC:  target.increment(); return;   // 自增
        case TJS_OP_DEC:  target.decrement(); return;   // 自减
    }
}
```

注意 `TJS_OP_INC` 和 `TJS_OP_DEC` 不需要 `param` 参数（一元运算），其他 14 种操作都是二元运算。

---

## §6 tTJSCustomObject——哈希表对象

`tTJSCustomObject` 是 TJS2 中最重要的运行时对象——脚本中创建的每个类实例、每个 `%[ ... ]` 字典字面量，在 C++ 层面都是一个 `tTJSCustomObject`。它用开放地址哈希表（open addressing hash table）+ 链式冲突解决（separate chaining）来存储成员。

### 6.1 内存布局

```
tTJSCustomObject
├── tTJSDispatch 基类 (RefCount, BeforeDestructionCalled)
├── Count: tjs_int                    — 当前成员总数
├── HashMask: tjs_int                 — 哈希掩码 (HashSize - 1)
├── HashSize: tjs_int                 — 桶数量 (2 的幂)
├── Symbols: tTJSSymbolData*          — 桶数组（堆分配）
├── RebuildHashMagic: tjs_uint        — 全局重建魔数（用于延迟重建）
├── IsInvalidated: bool               — 是否已无效化
├── IsInvalidating: bool              — 是否正在无效化中（防重入）
├── ClassInstances[4]: iTJSNativeInstance* — 原生实例槽位
├── ClassIDs[4]: tjs_int32            — 原生类 ID 槽位
├── CallFinalize: bool                — 是否需要调用 finalize
├── finalize_name: ttstr              — finalize 方法名
├── CallMissing: bool                 — 是否启用 missing 机制
├── ProsessingMissing: bool           — 是否正在处理 missing（防重入）
├── missing_name: ttstr               — missing 方法名
└── ClassNames: std::vector<ttstr>    — 类名列表（子类 → 父类顺序）
```

### 6.2 tTJSSymbolData——符号数据结构

每个成员用一个 `tTJSSymbolData` 结构体表示：

```cpp
// cpp/core/tjs2/tjsObject.h 第 264-339 行
struct tTJSSymbolData {
    tTJSVariantString *Name;   // 成员名（引用计数字符串）
    tjs_uint32 Hash;           // 名称的哈希值
    tjs_uint32 SymFlags;       // 符号标志（USING/INIT/HIDDEN/STATIC）
    tjs_uint32 Flags;          // 保留标志位
    tTJSVariant_S Value;       // 成员的值（20 字节，原始存储）
    tTJSSymbolData *Next;      // 链表指针（冲突链的下一个节点）
};
```

注意 `Value` 的类型是 `tTJSVariant_S`（原始联合体）而不是 `tTJSVariant`（完整类）。这避免了在 `memset` 清零初始化时触发 `tTJSVariant` 的构造函数。代码中通过强制类型转换来使用它：

```cpp
#define GetValue(x) (*((tTJSVariant*)(&(x->Value))))
```

### 6.3 哈希表结构

```
Symbols 数组（HashSize 个桶）
┌───────────┬───────────┬───────────┬───────────┐
│ Bucket[0] │ Bucket[1] │ Bucket[2] │ Bucket[3] │  ← 直接在数组中
│ Name="a"  │ (空)      │ Name="c"  │ Name="d"  │
│ Hash=...  │           │ Hash=...  │ Hash=...  │
│ Value=10  │           │ Value=30  │ Value=40  │
│ Next ──┐  │           │ Next=null │ Next ──┐  │
└────────┼──┴───────────┴───────────┴────────┼──┘
         │                                   │
         ▼                                   ▼
  ┌─────────────┐                    ┌─────────────┐
  │ Name="e"    │                    │ Name="f"    │
  │ Hash=...    │                    │ Hash=...    │
  │ Value=50    │                    │ Value=60    │
  │ Next=null   │                    │ Next=null   │
  └─────────────┘                    └─────────────┘
  （堆分配的冲突节点）                （堆分配的冲突节点）
```

关键设计：
- **桶头在数组中**（不是指针）——减少一次内存间接访问
- **冲突节点在堆上** `new tTJSSymbolData` 分配
- **插入到链表头部**（`data->Next = lv1->Next; lv1->Next = data`），O(1) 插入
- **访问频率优化**：当在链表中搜索时，如果走了超过 2 步才找到目标，会把该节点移到链表头部（move-to-front heuristic，一种"最近访问优先"的启发式策略）

### 6.4 成员查找：Find

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 983-1062 行（简化）
tTJSSymbolData* tTJSCustomObject::Find(const tjs_char *name, tjs_uint32 *hint) {
    if (!name) return nullptr;

    // 第一步：尝试用 hint（缓存的哈希值）快速查找
    if (hint && *hint) {
        tjs_uint32 hash = *hint;
        tTJSSymbolData *lv1 = Symbols + (hash & HashMask);
        // 先搜索链表（跳过桶头）
        tjs_int cnt = 0;
        for (auto *d = lv1->Next; d; d = d->Next, cnt++) {
            if (d->Hash == hash && (d->SymFlags & TJS_SYMBOL_USING)) {
                if (d->NameMatch(name)) {
                    if (cnt > 2) { /* 移到链表头部 */ }
                    return d;
                }
            }
        }
        // 再检查桶头
        if (lv1->Hash == hash && (lv1->SymFlags & TJS_SYMBOL_USING))
            if (lv1->NameMatch(name)) return lv1;
    }

    // 第二步：计算哈希值查找
    tjs_uint32 hash = tTJSHashFunc<tjs_char*>::Make(name);
    if (hint) {
        if (*hint == hash) return nullptr;  // 已经用这个哈希查过了
        *hint = hash;  // 缓存哈希值供下次使用
    }
    // 同样的链表搜索逻辑...
}
```

这个 `Find` 的双重查找策略保证了：
- **有 hint 时**：直接跳到目标桶，不需要计算哈希
- **hint 失效时**：重新计算哈希并更新 hint
- **hint 与新哈希相同时**：说明确实不存在，直接返回 `nullptr`

### 6.5 成员添加：Add

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 492-545 行（简化）
tTJSSymbolData* tTJSCustomObject::Add(const tjs_char *name, tjs_uint32 *hint) {
    // 先查找——如果已存在，直接返回
    tTJSSymbolData *data = Find(name, hint);
    if (data) return data;

    tjs_uint32 hash = /* 从 hint 或重新计算 */;
    tTJSSymbolData *lv1 = Symbols + (hash & HashMask);

    if (lv1->SymFlags & TJS_SYMBOL_USING) {
        // 桶头已被占用——分配新节点插入链表头
        data = new tTJSSymbolData;
        data->SelfClear();
        data->Next = lv1->Next;
        lv1->Next = data;
        data->SetName(name, hash);
        data->SymFlags |= TJS_SYMBOL_USING;
    } else {
        // 桶头空闲——直接使用
        lv1->SetName(name, hash);
        lv1->SymFlags |= TJS_SYMBOL_USING;
        data = lv1;
    }
    Count++;
    return data;
}
```

### 6.6 动态扩容：RebuildHash

当全局 `TJSGlobalRebuildHashMagic` 变化时（通过调用 `TJSDoRehash()` 触发），所有 `tTJSCustomObject` 在下次 `PropGet` 时会检测到自己的 `RebuildHashMagic` 与全局值不匹配，从而触发哈希表重建：

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 1276-1282 行
tjs_error tTJSCustomObject::PropGet(...) {
    if (RebuildHashMagic != TJSGlobalRebuildHashMagic) {
        RebuildHash();  // 延迟重建——只在真正访问时才做
    }
    // ...
}
```

`RebuildHash(requestcount)` 的新表大小计算使用了一个巧妙的位操作来找到 `requestcount` 的最高有效位，然后加 2 得到新的哈希位数。这确保哈希表大小总是 2 的幂，且比当前成员数大约大 4 倍（负载因子约 0.25），以保证查找性能。

重建过程：
1. 分配新的 `tTJSSymbolData` 数组
2. 遍历旧表的所有桶和链表
3. 用 `AddTo` 将每个成员插入新表
4. 释放旧表
5. 如果中途异常——恢复旧表，释放新表

### 6.7 成员删除

`DeleteByName` 分两种情况处理：

```cpp
// 情况 1：目标在桶头——不能真的删除（会破坏数组索引），只做 PostClear
if (lv1->NameMatch(name)) {
    CheckObjectClosureRemove(*(tTJSVariant*)(&(lv1->Value)));
    lv1->PostClear();   // 释放名字和值，清除 USING 标志
    Count--;
    return true;
}

// 情况 2：目标在链表中——从链表中断开并 delete
if (d->NameMatch(name)) {
    prevd->Next = d->Next;  // 从链表断开
    CheckObjectClosureRemove(*(tTJSVariant*)(&(d->Value)));
    d->Destory();   // 释放名字和值（注意原代码的拼写错误 Destory）
    delete d;       // 释放内存
    Count--;
    return true;
}
```

注意源码中有一个拼写错误：`Destory` 应该是 `Destroy`。这是一个有意思的历史遗留——修改拼写会破坏现有代码的所有调用点。

---

## §7 missing 机制——动态属性拦截

TJS2 支持一种类似 JavaScript `Proxy` 的特性：当访问不存在的成员时，可以调用一个名为 `missing` 的回调方法来动态处理。

### 7.1 启用方式

通过 `ClassInstanceInfo(TJS_CII_SET_MISSING, 0, &methodName)` 设置 `missing` 方法名。设置后 `CallMissing` 标志变为 `true`。

### 7.2 调用流程

以 `PropGet` 为例：

```
PropGet("someProperty")
  → Find("someProperty") == nullptr（成员不存在）
  → CallMissing == true？
    → 是：调用 CallGetMissing("someProperty", result)
      → 创建一个临时 tTJSSimpleGetSetProperty 对象
      → 调用 this->FuncCall("missing", args=[false, "someProperty", prop])
      → 如果 missing 返回非零值：使用 prop 中的值作为结果
      → 如果 missing 返回零或失败：继续走正常的"成员不存在"逻辑
    → 否：返回 TJS_E_MEMBERNOTFOUND
```

### 7.3 tTJSSimpleGetSetProperty——临时属性桥

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 260-295 行
class tTJSSimpleGetSetProperty : public tTJSDispatch {
private:
    tTJSVariant &Value;  // 引用外部变量
public:
    tTJSSimpleGetSetProperty(tTJSVariant &value) : Value(value) {};

    tjs_error PropGet(...) override {
        if (membername) return TJS_E_MEMBERNOTFOUND;
        if (result) *result = Value;  // 读取：返回引用的值
        return TJS_S_OK;
    }

    tjs_error PropSet(...) override {
        if (membername) return TJS_E_MEMBERNOTFOUND;
        Value = *param;  // 写入：修改引用的值
        return TJS_S_OK;
    }
};
```

这个小对象作为"桥梁"传递给 `missing` 方法——`missing` 方法通过 `PropSet` 设置返回值，调用方通过读取 `Value` 获取结果。

### 7.4 防重入保护

```cpp
bool tTJSCustomObject::CallGetMissing(const tjs_char *name, tTJSVariant &result) {
    if (ProsessingMissing) return false;  // 正在处理 missing，不递归
    ProsessingMissing = true;
    // ... 调用 missing 方法 ...
    ProsessingMissing = false;
    return res;
}
```

如果没有 `ProsessingMissing` 保护，`missing` 方法内部如果访问了另一个不存在的成员，就会无限递归直到栈溢出。

---

## §8 finalize 生命周期——对象的"临终遗言"

### 8.1 触发时机

`finalize` 在两种情况下被调用：
1. **显式无效化**：调用 `Invalidate(0, nullptr, nullptr, objthis)` 时
2. **引用计数归零**：`Release()` 检测到 `RefCount == 1` 时调用 `BeforeDestruction()`

### 8.2 执行流程

```
Release() → RefCount == 1
  → BeforeDestruction()
    → TJSSetObjectHashFlag(this, TJS_OHMF_DELETING)  // 标记正在删除
    → _Finalize()
      → IsInvalidating = true（防重入）
      → Finalize()
        → FuncCall(0, "finalize", ..., this)  // 调用脚本的 finalize 方法
        → 对每个 ClassInstances[i]→Invalidate()  // 无效化原生实例
        → DeleteAllMembers()  // 清除所有成员
      → IsInvalidated = true
      → IsInvalidating = false
  → if (RefCount == 1)  // 二次检查——finalize 可能复活了对象
      → delete this
```

### 8.3 DeleteAllMembers 的安全策略

删除所有成员时，如果直接逐个释放对象引用，可能触发连锁析构（成员 A 的析构导致成员 B 被释放，B 的析构又触发当前对象的方法……）。`tTJSCustomObject::DeleteAllMembers` 采用"先收集后释放"的安全策略：

```cpp
// cpp/core/tjs2/tjsObject.cpp 第 802-892 行（简化）
void tTJSCustomObject::DeleteAllMembers() {
    if (Count <= 10) return _DeleteAllMembers();  // 少量成员用栈数组

    std::vector<iTJSDispatch2*> vector;  // 收集所有对象引用

    // 第一遍：遍历所有成员，收集对象引用，清除值
    for (每个符号) {
        if (值是 tvtObject) {
            CheckObjectClosureRemove(值);
            AddRef 闭包中的 Object 和 ObjThis;
            将它们放入 vector;
            值.Clear();  // 先断开引用
        }
    }

    // 第二遍：删除所有符号数据结构
    for (每个符号) { Destory(); delete; }
    Count = 0;

    // 第三步：最后才释放收集的对象引用
    for (auto *dsp : vector) {
        dsp->Release();  // 这里可能触发其他对象的析构——但此时我们的成员已清空
    }
}
```

这个"先断开、再清理、最后释放"的三步策略确保了在对象引用被释放（可能触发连锁析构）时，当前对象的成员表已经是空的，不会有悬挂指针。

`_DeleteAllMembers` 是针对少量成员（≤10 个）的优化版本，使用栈上的固定大小数组 `iTJSDispatch2 *dsps[20]` 代替堆分配的 `std::vector`，避免了小对象场景下的内存分配开销。

---

## §9 iTJSNativeInstance——脚本与原生代码的桥梁

### 9.1 接口定义

```cpp
// cpp/core/tjs2/tjsInterface.h 第 326-337 行
class iTJSNativeInstance {
public:
    virtual tjs_error Construct(tjs_int numparams, tTJSVariant **param,
                                iTJSDispatch2 *tjs_obj) = 0;
    virtual void Invalidate() = 0;
    virtual void Destruct() = 0;
};
```

三个纯虚方法的职责：
- **Construct**：在 TJS2 的 `new` 操作中被调用，接收构造参数
- **Invalidate**：在 TJS2 对象被 `invalidate` 时调用，释放资源但不销毁
- **Destruct**：在 TJS2 对象被 `delete` 时调用，彻底销毁

### 9.2 生命周期对应

```
TJS2: var obj = new MyClass(arg1, arg2);
  C++: iTJSNativeInstance::Construct(2, [arg1, arg2], tjs_obj)

TJS2: invalidate obj;
  C++: iTJSNativeInstance::Invalidate()

TJS2: (引用计数归零)
  C++: iTJSNativeInstance::Destruct()
```

### 9.3 使用示例——在 C++ 中实现一个原生类

```cpp
#include "tjsNative.h"

// 原生实例类：存储 C++ 端的数据
class MyNativeInstance : public tTJSNativeInstance {
    // tTJSNativeInstance 是 iTJSNativeInstance 的便利基类
    int internalValue;

public:
    // 构造——接收 TJS2 脚本传入的参数
    tjs_error Construct(tjs_int numparams, tTJSVariant **param,
                        iTJSDispatch2 *tjs_obj) override {
        if (numparams >= 1) {
            internalValue = (tjs_int)*param[0];  // 第一个参数作为初始值
        }
        return TJS_S_OK;
    }

    // 无效化——释放资源但保留对象
    void Invalidate() override {
        internalValue = 0;
    }

    // 析构——彻底清理
    void Destruct() override {
        // C++ 端清理逻辑
    }

    int GetValue() const { return internalValue; }
    void SetValue(int v) { internalValue = v; }
};
```

---

## §10 动手实践

### 实践 1：手动实现一个最小的 iTJSDispatch2 子类

```cpp
#include <cstdio>
#include <cstring>

// 简化的类型定义（实际项目中使用 tjsTypes.h）
typedef int tjs_int;
typedef unsigned int tjs_uint;
typedef unsigned int tjs_uint32;
typedef int tjs_int32;
typedef int tjs_error;
typedef wchar_t tjs_char;

#define TJS_S_OK 0
#define TJS_S_TRUE 1
#define TJS_S_FALSE 2
#define TJS_E_NOTIMPL (-1002)
#define TJS_E_MEMBERNOTFOUND (-1001)
#define TJS_E_INVALIDPARAM (-1003)
#define TJS_FAILED(x) ((x) < 0)
#define TJS_SUCCEEDED(x) (!TJS_FAILED(x))

// 模拟 tTJSVariant（简化为 int）
struct SimpleVariant {
    int intValue;
    SimpleVariant() : intValue(0) {}
    SimpleVariant(int v) : intValue(v) {}
};

// 最简单的 Dispatch 实现——只支持 FuncCall
class SimpleFuncDispatch {
    tjs_uint refCount;
public:
    SimpleFuncDispatch() : refCount(1) {}
    virtual ~SimpleFuncDispatch() {}

    tjs_uint AddRef() { return ++refCount; }
    tjs_uint Release() {
        if (refCount == 1) { delete this; return 0; }
        return --refCount;
    }

    // 唯一有意义的方法：被调用时返回 42
    virtual tjs_error FuncCall(const tjs_char *membername,
                               SimpleVariant *result) {
        if (membername != nullptr) return TJS_E_MEMBERNOTFOUND;
        if (result) result->intValue = 42;
        printf("SimpleFuncDispatch::FuncCall called, returning 42\n");
        return TJS_S_OK;
    }

    // 所有其他方法返回 E_NOTIMPL
    virtual tjs_error PropGet(const tjs_char *membername, SimpleVariant *result) {
        return TJS_E_NOTIMPL;
    }
    virtual tjs_error PropSet(const tjs_char *membername, const SimpleVariant *param) {
        return TJS_E_NOTIMPL;
    }
};

int main() {
    SimpleFuncDispatch *func = new SimpleFuncDispatch();

    // 调用对象自身作为函数（membername == nullptr）
    SimpleVariant result;
    tjs_error hr = func->FuncCall(nullptr, &result);
    printf("FuncCall result: %s, value: %d\n",
           TJS_SUCCEEDED(hr) ? "OK" : "FAILED", result.intValue);
    // 输出：FuncCall result: OK, value: 42

    // 尝试不支持的操作
    hr = func->PropGet(nullptr, &result);
    printf("PropGet result: %s (expected: FAILED)\n",
           TJS_SUCCEEDED(hr) ? "OK" : "FAILED");
    // 输出：PropGet result: FAILED (expected: FAILED)

    func->Release();
    return 0;
}
```

### 实践 2：模拟哈希表对象的成员管理

```cpp
#include <cstdio>
#include <cstring>
#include <cstdlib>

// 简化的符号数据
struct SymbolData {
    char name[64];
    unsigned int hash;
    int value;
    bool inUse;
    SymbolData *next;

    void clear() {
        name[0] = '\0';
        hash = 0;
        value = 0;
        inUse = false;
        next = nullptr;
    }
};

// 简化的哈希函数
unsigned int simpleHash(const char *str) {
    unsigned int h = 0;
    while (*str) { h = h * 31 + (unsigned char)(*str++); }
    return h;
}

// 简化的哈希表对象
class SimpleHashObject {
    SymbolData *buckets;
    int hashSize;
    int hashMask;
    int count;

public:
    SimpleHashObject(int bits = 3) {
        hashSize = (1 << bits);  // 默认 8 个桶
        hashMask = hashSize - 1;
        buckets = new SymbolData[hashSize];
        for (int i = 0; i < hashSize; i++) buckets[i].clear();
        count = 0;
    }

    ~SimpleHashObject() {
        // 清理链表节点
        for (int i = 0; i < hashSize; i++) {
            SymbolData *d = buckets[i].next;
            while (d) {
                SymbolData *next = d->next;
                delete d;
                d = next;
            }
        }
        delete[] buckets;
    }

    // 添加或查找成员
    SymbolData* add(const char *name) {
        // 先查找是否已存在
        SymbolData *found = find(name);
        if (found) return found;

        unsigned int hash = simpleHash(name);
        SymbolData *bucket = &buckets[hash & hashMask];

        if (bucket->inUse) {
            // 桶头已占用——创建链表节点
            SymbolData *newNode = new SymbolData;
            newNode->clear();
            strncpy(newNode->name, name, 63);
            newNode->hash = hash;
            newNode->inUse = true;
            newNode->next = bucket->next;
            bucket->next = newNode;
            count++;
            return newNode;
        } else {
            // 桶头空闲——直接使用
            strncpy(bucket->name, name, 63);
            bucket->hash = hash;
            bucket->inUse = true;
            count++;
            return bucket;
        }
    }

    // 查找成员
    SymbolData* find(const char *name) {
        unsigned int hash = simpleHash(name);
        SymbolData *bucket = &buckets[hash & hashMask];

        // 检查桶头
        if (bucket->inUse && strcmp(bucket->name, name) == 0) return bucket;

        // 检查链表
        for (SymbolData *d = bucket->next; d; d = d->next) {
            if (d->inUse && d->hash == hash && strcmp(d->name, name) == 0)
                return d;
        }
        return nullptr;
    }

    int getCount() const { return count; }
};

int main() {
    SimpleHashObject obj;

    // 添加几个成员
    obj.add("name")->value = 100;
    obj.add("age")->value = 25;
    obj.add("score")->value = 95;
    printf("Member count: %d\n", obj.getCount());
    // 输出：Member count: 3

    // 查找
    SymbolData *d = obj.find("age");
    if (d) printf("age = %d\n", d->value);
    // 输出：age = 25

    // 重复添加——返回已有的
    SymbolData *existing = obj.add("name");
    printf("name (re-add) = %d, count still: %d\n",
           existing->value, obj.getCount());
    // 输出：name (re-add) = 100, count still: 3

    // 查找不存在的
    d = obj.find("nonexistent");
    printf("nonexistent: %s\n", d ? "found" : "not found");
    // 输出：nonexistent: not found

    return 0;
}
```

### 实践 3：模拟 missing 机制

```cpp
#include <cstdio>
#include <cstring>
#include <map>
#include <string>

// 模拟一个支持 missing 机制的对象
class MissingAwareObject {
    std::map<std::string, int> members;
    bool callMissing;
    bool processingMissing;  // 防重入

public:
    MissingAwareObject() : callMissing(true), processingMissing(false) {}

    // 模拟 PropGet
    bool propGet(const char *name, int &result) {
        auto it = members.find(name);
        if (it != members.end()) {
            result = it->second;
            return true;
        }

        // 成员不存在——尝试 missing
        if (callMissing && !processingMissing) {
            processingMissing = true;
            bool handled = onMissing(false, name, result);
            processingMissing = false;
            if (handled) return true;
        }

        printf("  PropGet('%s'): MEMBER_NOT_FOUND\n", name);
        return false;
    }

    // 模拟 PropSet
    bool propSet(const char *name, int value) {
        auto it = members.find(name);
        if (it != members.end()) {
            it->second = value;
            return true;
        }

        // 成员不存在——尝试 missing
        if (callMissing && !processingMissing) {
            processingMissing = true;
            bool handled = onMissing(true, name, value);
            processingMissing = false;
            if (handled) return true;
        }

        // 不存在且 missing 未处理——创建新成员
        members[name] = value;
        return true;
    }

    // missing 回调——可以拦截对不存在成员的访问
    bool onMissing(bool isSet, const char *name, int &value) {
        printf("  missing(%s, '%s')\n", isSet ? "SET" : "GET", name);

        // 示例：为以 "computed_" 开头的名字提供计算值
        if (strncmp(name, "computed_", 9) == 0) {
            if (!isSet) {
                value = 42;  // 返回计算的默认值
                printf("    -> handled: computed value = %d\n", value);
            } else {
                printf("    -> handled: ignored set for computed property\n");
            }
            return true;  // 已处理
        }
        return false;  // 未处理——走正常逻辑
    }
};

int main() {
    MissingAwareObject obj;
    int value;

    // 正常的属性访问
    obj.propSet("name", 100);
    obj.propGet("name", value);
    printf("name = %d\n\n", value);
    // 输出：name = 100

    // 访问计算属性——被 missing 拦截
    obj.propGet("computed_total", value);
    printf("computed_total = %d\n\n", value);
    // 输出：missing(GET, 'computed_total')
    //        -> handled: computed value = 42
    //       computed_total = 42

    // 访问普通不存在的属性——missing 不处理
    obj.propGet("unknown", value);
    // 输出：missing(GET, 'unknown')
    //        PropGet('unknown'): MEMBER_NOT_FOUND

    return 0;
}
```

### 实践 4：模拟 Operation 的模板方法模式

```cpp
#include <cstdio>
#include <cstring>
#include <map>
#include <string>

#define OP_ADD 5
#define OP_SUB 4
#define OP_MUL 9
#define OP_INC 15
#define OP_DEC 16

// 基类——只实现 Operation，依赖子类的 PropGet/PropSet
class BaseObject {
public:
    virtual ~BaseObject() {}

    virtual bool propGet(const char *name, int &result) = 0;
    virtual bool propSet(const char *name, int value) = 0;

    // Operation 使用模板方法模式
    bool operation(int op, const char *name, int param, int &result) {
        // 第一步：读取当前值
        int current;
        if (!propGet(name, current)) return false;

        // 第二步：执行运算
        switch (op) {
            case OP_ADD: current += param; break;
            case OP_SUB: current -= param; break;
            case OP_MUL: current *= param; break;
            case OP_INC: current++; break;
            case OP_DEC: current--; break;
            default: return false;
        }

        // 第三步：写回
        if (!propSet(name, current)) return false;

        result = current;
        return true;
    }
};

// 子类——实现具体的存储
class MapObject : public BaseObject {
    std::map<std::string, int> data;
public:
    bool propGet(const char *name, int &result) override {
        auto it = data.find(name);
        if (it == data.end()) return false;
        result = it->second;
        return true;
    }
    bool propSet(const char *name, int value) override {
        data[name] = value;
        return true;
    }
};

int main() {
    MapObject obj;
    int result;

    obj.propSet("counter", 10);

    // counter += 5
    obj.operation(OP_ADD, "counter", 5, result);
    printf("counter += 5 -> %d\n", result);  // 15

    // counter *= 3
    obj.operation(OP_MUL, "counter", 3, result);
    printf("counter *= 3 -> %d\n", result);  // 45

    // counter++
    obj.operation(OP_INC, "counter", 0, result);
    printf("counter++ -> %d\n", result);  // 46

    // counter -= 6
    obj.operation(OP_SUB, "counter", 6, result);
    printf("counter -= 6 -> %d\n", result);  // 40

    return 0;
}
```

### 实践 5：模拟 NativeInstanceSupport 的注册与检索

```cpp
#include <cstdio>

#define MAX_NATIVE_CLASS 4
#define NIS_REGISTER 0
#define NIS_GETINSTANCE 1

// 模拟原生实例接口
class INativeInstance {
public:
    virtual ~INativeInstance() {}
    virtual const char* getClassName() = 0;
    virtual void invalidate() = 0;
};

// 一个具体的原生实例
class SpriteInstance : public INativeInstance {
    int x, y, width, height;
public:
    SpriteInstance(int x, int y, int w, int h) : x(x), y(y), width(w), height(h) {}
    const char* getClassName() override { return "Sprite"; }
    void invalidate() override { printf("Sprite invalidated\n"); }
    void draw() { printf("Drawing Sprite at (%d,%d) size %dx%d\n", x, y, width, height); }
};

class AudioInstance : public INativeInstance {
    const char *filename;
public:
    AudioInstance(const char *fn) : filename(fn) {}
    const char* getClassName() override { return "Audio"; }
    void invalidate() override { printf("Audio invalidated\n"); }
    void play() { printf("Playing audio: %s\n", filename); }
};

// 模拟 tTJSCustomObject 的 NativeInstanceSupport
class ScriptObject {
    INativeInstance *instances[MAX_NATIVE_CLASS];
    int classIDs[MAX_NATIVE_CLASS];

public:
    ScriptObject() {
        for (int i = 0; i < MAX_NATIVE_CLASS; i++) {
            instances[i] = nullptr;
            classIDs[i] = -1;
        }
    }

    ~ScriptObject() {
        // 逆序销毁（子类先于父类）
        for (int i = MAX_NATIVE_CLASS - 1; i >= 0; i--) {
            if (classIDs[i] != -1 && instances[i]) {
                instances[i]->invalidate();
                delete instances[i];
            }
        }
    }

    bool nativeInstanceSupport(int flag, int classID, INativeInstance **pointer) {
        if (flag == NIS_REGISTER) {
            for (int i = 0; i < MAX_NATIVE_CLASS; i++) {
                if (classIDs[i] == -1) {
                    classIDs[i] = classID;
                    instances[i] = *pointer;
                    printf("Registered class ID %d at slot %d\n", classID, i);
                    return true;
                }
            }
            printf("ERROR: All %d slots full!\n", MAX_NATIVE_CLASS);
            return false;
        }

        if (flag == NIS_GETINSTANCE) {
            for (int i = 0; i < MAX_NATIVE_CLASS; i++) {
                if (classIDs[i] == classID) {
                    *pointer = instances[i];
                    return true;
                }
            }
            return false;
        }
        return false;
    }
};

int main() {
    ScriptObject obj;

    // 注册两个原生实例
    INativeInstance *sprite = new SpriteInstance(100, 200, 64, 64);
    obj.nativeInstanceSupport(NIS_REGISTER, 1001, &sprite);
    // 输出：Registered class ID 1001 at slot 0

    INativeInstance *audio = new AudioInstance("bgm.ogg");
    obj.nativeInstanceSupport(NIS_REGISTER, 1002, &audio);
    // 输出：Registered class ID 1002 at slot 1

    // 检索原生实例并使用
    INativeInstance *retrieved = nullptr;
    if (obj.nativeInstanceSupport(NIS_GETINSTANCE, 1001, &retrieved)) {
        printf("Found: %s\n", retrieved->getClassName());
        static_cast<SpriteInstance*>(retrieved)->draw();
    }
    // 输出：Found: Sprite
    //       Drawing Sprite at (100,200) size 64x64

    if (obj.nativeInstanceSupport(NIS_GETINSTANCE, 1002, &retrieved)) {
        printf("Found: %s\n", retrieved->getClassName());
        static_cast<AudioInstance*>(retrieved)->play();
    }
    // 输出：Found: Audio
    //       Playing audio: bgm.ogg

    // 查找不存在的
    if (!obj.nativeInstanceSupport(NIS_GETINSTANCE, 9999, &retrieved)) {
        printf("Class ID 9999 not found\n");
    }
    // 输出：Class ID 9999 not found

    return 0;
}
// 析构时输出：
// Audio invalidated
// Sprite invalidated
```

---

## §11 对照项目源码

| 文件路径 | 行范围 | 内容说明 |
|---------|--------|---------|
| `cpp/core/tjs2/tjsInterface.h` | 1-50 | 调用标志 `TJS_MEMBERENSURE` 等 |
| `cpp/core/tjs2/tjsInterface.h` | 51-104 | 操作标志 `TJS_OP_*`、NIS/CII 标志 |
| `cpp/core/tjs2/tjsInterface.h` | 105-323 | `iTJSDispatch2` 纯虚接口完整定义 |
| `cpp/core/tjs2/tjsInterface.h` | 326-337 | `iTJSNativeInstance` 接口 |
| `cpp/core/tjs2/tjsErrorDefs.h` | 1-69 | 错误码定义、`TJS_FAILED`/`TJS_SUCCEEDED` 宏 |
| `cpp/core/tjs2/tjsObject.h` | 85-233 | `tTJSDispatch` 基类——默认实现 |
| `cpp/core/tjs2/tjsObject.h` | 241-251 | 符号标志 `TJS_SYMBOL_*`、`TJS_MAX_NATIVE_CLASS` |
| `cpp/core/tjs2/tjsObject.h` | 258-339 | `tTJSCustomObject::tTJSSymbolData` 结构体 |
| `cpp/core/tjs2/tjsObject.h` | 342-374 | `tTJSCustomObject` 成员变量、finalize/missing 配置 |
| `cpp/core/tjs2/tjsObject.h` | 379-395 | `CheckObjectClosureAdd` / `Remove`——循环引用保护 |
| `cpp/core/tjs2/tjsObject.h` | 401-433 | `Add` / `Find` / `RebuildHash` / `DeleteByName` 声明 |
| `cpp/core/tjs2/tjsObject.h` | 448-501 | `tTJSCustomObject` 重写的所有虚方法声明 |
| `cpp/core/tjs2/tjsObject.cpp` | 26-78 | `TJSDoVariantOperation`——运算分派 |
| `cpp/core/tjs2/tjsObject.cpp` | 85-130 | `tTJSDispatch` 构造/析构/AddRef/Release |
| `cpp/core/tjs2/tjsObject.cpp` | 133-253 | `tTJSDispatch` 所有 ByNum 方法 + Operation |
| `cpp/core/tjs2/tjsObject.cpp` | 260-295 | `tTJSSimpleGetSetProperty`——missing 桥接对象 |
| `cpp/core/tjs2/tjsObject.cpp` | 328-356 | `tTJSCustomObject` 构造函数——初始化哈希表 |
| `cpp/core/tjs2/tjsObject.cpp` | 359-410 | 析构、`_Finalize`、`BeforeDestruction` |
| `cpp/core/tjs2/tjsObject.cpp` | 413-489 | `CallGetMissing` / `CallSetMissing`——missing 机制 |
| `cpp/core/tjs2/tjsObject.cpp` | 492-599 | `Add`（两个重载）——成员添加 |
| `cpp/core/tjs2/tjsObject.cpp` | 648-756 | `RebuildHash`——哈希表动态扩容 |
| `cpp/core/tjs2/tjsObject.cpp` | 759-799 | `DeleteByName`——成员删除 |
| `cpp/core/tjs2/tjsObject.cpp` | 802-980 | `DeleteAllMembers` / `_DeleteAllMembers`——安全批量删除 |
| `cpp/core/tjs2/tjsObject.cpp` | 983-1062 | `Find`——成员查找（hint 优化） |
| `cpp/core/tjs2/tjsObject.cpp` | 1065-1129 | `CallEnumCallbackForData` / `InternalEnumMembers` |
| `cpp/core/tjs2/tjsObject.cpp` | 1141-1230 | `TJSTryFuncCallViaPropGet` / `TJSDefaultFuncCall` / `FuncCall` |
| `cpp/core/tjs2/tjsObject.cpp` | 1233-1312 | `TJSDefaultPropGet` / `PropGet` |
| `cpp/core/tjs2/tjsObject.cpp` | 1315-1428 | `TJSDefaultPropSet` / `PropSet` |
| `cpp/core/tjs2/tjsObject.cpp` | 1525-1534 | `EnumMembers` |
| `cpp/core/tjs2/tjsObject.cpp` | 1555-1601 | `TJSDefaultInvalidate` / `Invalidate` |
| `cpp/core/tjs2/tjsObject.cpp` | 1604-1646 | `TJSDefaultIsValid` / `IsValid` |
| `cpp/core/tjs2/tjsObject.cpp` | 1649-1697 | `TJSDefaultCreateNew` / `CreateNew` |
| `cpp/core/tjs2/tjsObject.cpp` | 1712-1799 | `TJSDefaultIsInstanceOf` / `IsInstanceOf` |
| `cpp/core/tjs2/tjsObject.cpp` | 1802-1936 | `TJSDefaultOperation` / `Operation` |
| `cpp/core/tjs2/tjsObject.cpp` | 1938-1967 | `NativeInstanceSupport` |
| `cpp/core/tjs2/tjsObject.cpp` | 1970-2010 | `ClassInstanceInfo` |
| `cpp/core/tjs2/tjsObject.cpp` | 2016-2020 | `TJSCreateCustomObject` 工厂函数 |

---

## §12 本节小结

1. **iTJSDispatch2 是万物之源**——TJS2 中所有对象都实现这个接口，VM 通过它进行一切对象操作。接口定义了 14 对方法（by-name + by-number），覆盖了函数调用、属性读写、成员枚举/删除、对象无效化、类型检查、原生实例绑定等全部操作。

2. **错误码系统借鉴 COM HRESULT**——负数表示失败，0 和正数表示成功。`TJS_S_FALSE`（2）容易被误解为失败，实际上是"操作成功但结果为假"。

3. **调用标志控制方法行为**——`TJS_MEMBERENSURE` 自动创建不存在的成员，`TJS_IGNOREPROP` 绕过属性处理器，`TJS_HIDDENMEMBER` 设置不可枚举属性。

4. **tTJSDispatch 基类是"骨架"**——所有方法默认返回 `TJS_E_NOTIMPL`，子类只重写需要的方法。`ByNum` 方法统一转发给字符串版本。

5. **tTJSCustomObject 用哈希表存储成员**——桶头在数组中，冲突节点在堆上用链表串联。支持 hint 加速查找、move-to-front 热度优化、延迟扩容。

6. **循环引用通过 CheckObjectClosure 中和**——当成员闭包的 `ObjThis` 指向自身时，额外的引用会被抵消。

7. **missing 机制是 TJS2 的动态属性拦截**——类似 JavaScript `Proxy`，通过 `ProsessingMissing` 防止重入递归。

8. **finalize 是对象的临终钩子**——在无效化或引用计数归零时调用。`DeleteAllMembers` 采用"先断开后释放"的安全策略防止连锁析构导致的悬挂指针。

9. **NativeInstanceSupport 桥接脚本与原生代码**——每个对象最多关联 4 个 C++ 原生实例（`TJS_MAX_NATIVE_CLASS = 4`），通过 `classid` 注册和检索。

10. **Reserved 方法预留了 ABI 扩展空间**——3 个预留虚方法确保未来添加新功能时不会破坏已编译插件的二进制兼容性。

---

## §13 练习题与答案

### 题目 1：`TJS_S_FALSE` 是成功还是失败？

在以下代码中，`hr` 的值为 `TJS_S_FALSE`（即 2）。`TJS_FAILED(hr)` 和 `TJS_SUCCEEDED(hr)` 的返回值分别是什么？为什么 `TJS_S_FALSE` 不表示"失败"？

```cpp
tjs_error hr = obj->IsValid(0, nullptr, nullptr, objthis);
// 假设 hr == TJS_S_FALSE
bool failed = TJS_FAILED(hr);
bool succeeded = TJS_SUCCEEDED(hr);
```

<details>
<summary>查看答案</summary>

`TJS_FAILED(TJS_S_FALSE)` 返回 `false`（因为 `TJS_S_FALSE` 的值是 2，不小于 0）。
`TJS_SUCCEEDED(TJS_S_FALSE)` 返回 `true`。

`TJS_S_FALSE` 表示"操作成功执行了，但查询的逻辑结果为假"。在 `IsValid` 的语境中，它表示"我成功地检查了这个对象的有效性，结果是：无效"。

这与 "操作失败"（如 `TJS_E_INVALIDOBJECT`）不同——失败意味着操作本身无法完成（比如对象已被销毁导致无法执行检查），而 `TJS_S_FALSE` 意味着操作正常完成了。

正确的判断逻辑：
```cpp
tjs_error hr = obj->IsValid(0, nullptr, nullptr, objthis);
if (TJS_FAILED(hr)) {
    // 操作本身失败了——可能是严重错误
} else if (hr == TJS_S_TRUE) {
    // 对象有效
} else if (hr == TJS_S_FALSE) {
    // 对象已被无效化，但检查操作成功
}
```

</details>

### 题目 2：为什么 `Release()` 要两次检查 `RefCount == 1`？

阅读 `tTJSDispatch::Release()` 的代码，解释为什么在 `BeforeDestruction()` 调用后需要再次检查 `RefCount == 1`。给出一个会导致第二次检查不通过的具体场景。

<details>
<summary>查看答案</summary>

`BeforeDestruction()` 是一个虚方法，子类（如 `tTJSCustomObject`）的实现会调用 `_Finalize()` → `Finalize()` → `FuncCall("finalize", ...)`——即执行 TJS2 脚本中的 `finalize` 方法。

这个 `finalize` 脚本可能做出以下操作使对象"复活"：

```javascript
// TJS2 脚本
class MyClass {
    var globalRef;

    function finalize() {
        // 在 finalize 中把自己保存到全局变量——增加了引用计数
        globalRef = this;  // this 的 AddRef 被调用，RefCount 变为 2
    }
}
```

执行流程：
1. `Release()` 被调用，`RefCount` 从 2 变为... 等等，其实 `Release()` 先检查 `RefCount == 1`（不做 `--RefCount`）
2. `BeforeDestruction()` 调用 `finalize` 脚本
3. `finalize` 中 `globalRef = this` 导致 `AddRef()` 被调用，`RefCount` 变为 2
4. `Release()` 再次检查 `RefCount == 1`——不通过！
5. 所以不执行 `delete this`，而是执行 `return --RefCount`（变为 1）

这样对象就不会被错误地销毁。当后续 `globalRef` 被清除时，`Release()` 会再次被调用，此时如果 `RefCount` 真的是 1 且 `finalize` 不再复活对象，对象才会被真正删除。

注意 `BeforeDestructionCalled` 标志确保 `BeforeDestruction()` 只调用一次——第二次 `Release()` 不会再触发 `finalize`。

</details>

### 题目 3：手动实现一个支持 `PropGet`、`PropSet` 和 `Operation` 的对象

基于 `tTJSDispatch::Operation` 的模板方法模式，实现一个 `CounterObject` 类，支持：
- `PropGet("value")` 返回当前计数
- `PropSet("value", n)` 设置计数为 n
- `Operation(TJS_OP_INC, "value")` 自增
- `Operation(TJS_OP_ADD, "value", 5)` 加 5

<details>
<summary>查看答案</summary>

```cpp
#include <cstdio>
#include <cstring>

typedef int tjs_int;
typedef unsigned int tjs_uint32;
typedef int tjs_error;
typedef wchar_t tjs_char;

#define TJS_S_OK 0
#define TJS_E_MEMBERNOTFOUND (-1001)
#define TJS_E_NOTIMPL (-1002)
#define TJS_E_INVALIDPARAM (-1003)
#define TJS_FAILED(x) ((x) < 0)
#define TJS_OP_ADD 5
#define TJS_OP_SUB 4
#define TJS_OP_INC 15
#define TJS_OP_DEC 16
#define TJS_OP_MASK 0x001f
#define TJS_OP_MIN 1
#define TJS_OP_MAX 16

struct Variant {
    int intVal;
    Variant() : intVal(0) {}
    Variant(int v) : intVal(v) {}
    void operator+=(const Variant &o) { intVal += o.intVal; }
    void operator-=(const Variant &o) { intVal -= o.intVal; }
    void increment() { intVal++; }
    void decrement() { intVal--; }
};

void DoOperation(int op, Variant &target, const Variant *param) {
    switch(op) {
        case TJS_OP_ADD: target += *param; return;
        case TJS_OP_SUB: target -= *param; return;
        case TJS_OP_INC: target.increment(); return;
        case TJS_OP_DEC: target.decrement(); return;
    }
}

class CounterObject {
    Variant counter;  // 内部状态

public:
    CounterObject(int initial = 0) : counter(initial) {}

    tjs_error PropGet(const char *membername, Variant *result) {
        if (!membername) return TJS_E_NOTIMPL;
        if (strcmp(membername, "value") != 0) return TJS_E_MEMBERNOTFOUND;
        if (result) *result = counter;
        return TJS_S_OK;
    }

    tjs_error PropSet(const char *membername, const Variant *param) {
        if (!membername) return TJS_E_NOTIMPL;
        if (strcmp(membername, "value") != 0) return TJS_E_MEMBERNOTFOUND;
        if (!param) return TJS_E_INVALIDPARAM;
        counter = *param;
        return TJS_S_OK;
    }

    // 模板方法模式——复用 PropGet/PropSet
    tjs_error Operation(tjs_uint32 flag, const char *membername,
                        Variant *result, const Variant *param) {
        tjs_uint32 op = flag & TJS_OP_MASK;
        if (op != TJS_OP_INC && op != TJS_OP_DEC && param == nullptr)
            return TJS_E_INVALIDPARAM;
        if (op < TJS_OP_MIN || op > TJS_OP_MAX) return TJS_E_INVALIDPARAM;

        Variant tmp;
        tjs_error hr = PropGet(membername, &tmp);  // 读
        if (TJS_FAILED(hr)) return hr;

        DoOperation(op, tmp, param);               // 算

        hr = PropSet(membername, &tmp);             // 写
        if (TJS_FAILED(hr)) return hr;

        if (result) *result = tmp;                  // 返回
        return TJS_S_OK;
    }
};

int main() {
    CounterObject counter(10);
    Variant result;

    // PropGet
    counter.PropGet("value", &result);
    printf("Initial: %d\n", result.intVal);     // 10

    // Operation: value += 5
    Variant five(5);
    counter.Operation(TJS_OP_ADD, "value", &result, &five);
    printf("After += 5: %d\n", result.intVal);  // 15

    // Operation: value++
    counter.Operation(TJS_OP_INC, "value", &result, nullptr);
    printf("After ++: %d\n", result.intVal);    // 16

    // Operation: value -= 3
    Variant three(3);
    counter.Operation(TJS_OP_SUB, "value", &result, &three);
    printf("After -= 3: %d\n", result.intVal);  // 13

    // PropSet to 100
    Variant hundred(100);
    counter.PropSet("value", &hundred);
    counter.PropGet("value", &result);
    printf("After set 100: %d\n", result.intVal); // 100

    return 0;
}
```

这个实现展示了 `Operation` 的核心设计——**它不直接操作数据，而是通过 `PropGet` 和 `PropSet` 间接操作**。这意味着：
- 如果 `PropGet`/`PropSet` 有属性处理器（getter/setter），`Operation` 自动尊重它
- 子类只需要重写 `PropGet`/`PropSet`，`Operation` 就自动可用
- 这就是"模板方法模式"的威力——算法骨架在基类，具体步骤在子类

</details>

---

## 下一步

下一节 [内置对象实现](./03-内置对象实现.md) 将深入分析 TJS2 的 Array、Dictionary、RegExp 等内置对象如何继承 `tTJSCustomObject`，实现各自的特殊行为。

