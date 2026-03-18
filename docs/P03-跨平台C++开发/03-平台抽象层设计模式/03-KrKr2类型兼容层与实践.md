# KrKr2 类型兼容层与实践

> **所属模块：** P03-跨平台 C++ 开发
> **前置知识：** [01-PAL 概念与编译期文件分离](./01-PAL概念与编译期文件分离.md)、[02-PIMPL 与运行时多态](./02-PIMPL与运行时多态.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：
1. 理解"类型兼容层"（Type Compatibility Layer）在跨平台移植中的作用——为什么仅靠文件分离和 PIMPL 还不够
2. 分析 KrKr2 的四层类型兼容体系：`typedefine.h`、`tjsTypes.h`、`combase.h`、`vkdefine.h`
3. 自己编写一个类型兼容层，把 Windows 专有类型映射为跨平台等价物
4. 识别类型兼容层设计中的常见陷阱（大小不匹配、有符号/无符号混淆、对齐差异）
5. 在实际项目中为新的 PAL 函数编写完整的四平台实现

---

## 为什么需要类型兼容层

### 场景：当你继承了一座 Windows "遗产"

想象这样的情景：你接手了一个有十万行代码的游戏引擎，它最初是纯 Windows 程序。代码中到处都是 Windows 专有类型：

```cpp
// 某个图像加载模块（原版 KiriKiri 风格）
HRESULT LoadBitmap(HBITMAP* outBitmap, const tjs_char* filename) {
    DWORD fileSize = GetFileSize(hFile);
    BYTE* buffer = new BYTE[fileSize];
    WORD bmpMagic = *(WORD*)buffer;
    // ... 一万行类似代码
    return S_OK;
}
```

这段代码中，`HRESULT`、`HBITMAP`、`DWORD`、`BYTE`、`WORD`、`S_OK` 全部是 Windows SDK 定义的类型和常量。在 Linux、macOS、Android 上，这些符号根本不存在——编译器会报出成百上千个"undeclared identifier"错误。

你面临三个选择：

| 方案 | 工作量 | 风险 |
|------|--------|------|
| ① 把所有 Windows 类型替换为标准类型 | 极大（十万行逐行改） | 高（容易引入 bug） |
| ② 在每处使用 `#ifdef _WIN32` 包裹 | 极大（代码变得不可读） | 中（维护噩梦） |
| ③ **写一个兼容头文件，在非 Windows 平台用 typedef 模拟这些类型** | 小（一个文件搞定） | 低（改动最小） |

方案③就是**类型兼容层**——用最少的代码量，让原版 Windows 代码在四个平台都能编译通过，而无需修改业务逻辑。

这正是 KrKr2 的选择。KrKr2 的前身 KiriKiri 是一个纯 Windows 引擎，其代码大量使用 Windows 类型。KrKr2 项目要在 Linux、macOS、Android 上运行这些代码，类型兼容层是最务实的解决方案。

---

## KrKr2 的四层类型兼容体系

KrKr2 并不是只有一个类型兼容文件，而是有**四层**，各自解决不同层面的跨平台问题：

```
┌─────────────────────────────────────────────────────────┐
│                    应用代码（引擎各模块）                  │
│      使用 DWORD、HRESULT、tjs_int、VK_RETURN 等         │
├─────────────────┬──────────┬──────────┬─────────────────┤
│  typedefine.h   │tjsTypes.h│combase.h │   vkdefine.h    │
│  Windows 基础   │TJS 脚本  │COM 接口  │   虚拟键码      │
│  类型别名       │引擎类型  │模拟层    │   常量定义       │
├─────────────────┴──────────┴──────────┴─────────────────┤
│              <stdint.h> / <cstdint>                      │
│         C/C++ 标准固定宽度整数类型（跨平台保证）           │
└─────────────────────────────────────────────────────────┘
```

让我们逐层分析。

---

### 第一层：typedefine.h —— Windows 基础类型模拟

这是最核心的兼容层。它的工作原理用一句话概括：**在 Windows 上什么都不做（直接用原生类型），在其他平台上用 `<stdint.h>` 的固定宽度类型来模拟 Windows 类型。**

来看实际代码（`cpp/core/environ/typedefine.h`）：

```cpp
// cpp/core/environ/typedefine.h — KrKr2 实际源码
#pragma once
#include <stdint.h>

// 第一步：平台检测
#if defined(_WIN32) || defined(_WIN64)
#define TARGET_WINDOWS 1
#include <windows.h>    // Windows: 直接用原生 SDK 类型
#else
#define TARGET_WINDOWS 0
#include "tjsTypes.h"   // 非 Windows: 引入 TJS 类型定义
#include <sys/types.h>
#endif

// 第二步：仅在非 Windows 平台定义 Windows 类型的替代品
#if !TARGET_WINDOWS

/* ========== 整数类型别名 ========== */
typedef int32_t  LONG;       // Windows 的 LONG 是 32 位有符号
typedef uint32_t ULONG;      // 无符号 32 位
typedef uint32_t DWORD;      // "Double WORD" = 32 位无符号
typedef uint16_t WORD;       // 16 位无符号
typedef uint8_t  BYTE;       // 8 位无符号
typedef int16_t  SHORT;      // 16 位有符号
typedef int32_t  INT32;      // 32 位有符号
typedef uint32_t UINT32;     // 32 位无符号
typedef uint64_t ULONGLONG;  // 64 位无符号

/* ========== 指针类型别名 ========== */
typedef void*        LPVOID;   // "Long Pointer to VOID"
typedef uint32_t*    LPDWORD;  // "Long Pointer to DWORD"
typedef char*        PSTR;     // "Pointer to STRing"（ANSI）
typedef char*        LPSTR;    // 同上（历史遗留的重复命名）
typedef const char*  LPCSTR;   // "Long Pointer to Const STRing"

/* ========== 句柄类型模拟 ========== */
typedef void* HBITMAP;      // 位图句柄 → 模拟为不透明指针
typedef void* HDC;          // 设备上下文句柄
typedef void* HENHMETAFILE; // 增强型图元文件句柄
typedef void* HFONT;        // 字体句柄
typedef void* HICON;        // 图标句柄
typedef void* HINSTANCE;    // 模块实例句柄
typedef void* HMETAFILE;    // 图元文件句柄
typedef void* HPALETTE;     // 调色板句柄
typedef LONG  HRESULT;      // COM 返回值（32 位有符号整数）

/* ========== 结构体模拟 ========== */
typedef struct _FILETIME {
    DWORD dwLowDateTime;   // 低 32 位
    DWORD dwHighDateTime;  // 高 32 位
} FILETIME;

typedef union _ULARGE_INTEGER {
    struct {
        DWORD LowPart;
        DWORD HighPart;
    } u;
    ULONGLONG QuadPart;    // 64 位整体访问
} ULARGE_INTEGER;

typedef union _LARGE_INTEGER {
    struct {
        DWORD LowPart;
        LONG  HighPart;
    } u;
    int64_t QuadPart;
} LARGE_INTEGER;

/* ========== 兼容性空宏 ========== */
#define IN              // Windows SAL 注解 → 空宏
#define OUT             // Windows SAL 注解 → 空宏
#define CONST const

#endif // !TARGET_WINDOWS

/* ========== 所有平台通用的固定宽度类型 ========== */
// 这些在所有平台上都需要，不受 #if !TARGET_WINDOWS 保护
typedef uint8_t  UINT8;
typedef uint16_t UINT16;
typedef uint32_t UINT32;
typedef uint64_t UINT64;
typedef int8_t   INT8;
typedef int16_t  INT16;
typedef int32_t  INT32;
typedef int64_t  INT64;
typedef float    REAL;
```

#### 关键设计决策解读

**为什么句柄都用 `void*`？**

在 Windows 上，`HBITMAP`、`HDC` 等句柄本质上就是不透明指针（opaque pointer）——你不能解引用它们，只能传给 API 函数。在非 Windows 平台上，引擎不会调用 Windows GDI 函数，这些句柄类型只需要"能编译通过"就行，所以用 `void*` 模拟完全足够。

**为什么 `HRESULT` 用 `LONG` 而不是 `void*`？**

因为 `HRESULT` 不是句柄，而是一个**错误码**。代码中会做数值比较：

```cpp
HRESULT hr = SomeFunction();
if (SUCCEEDED(hr)) {   // 展开为 ((HRESULT)(hr)) >= 0
    // ...
}
```

如果把 `HRESULT` 定义为 `void*`，这个比较就编译不过。所以它必须是整数类型，而且必须是**有符号的** 32 位整数（负值表示错误）。

**为什么 `IN`、`OUT` 定义为空宏？**

这些是 Windows 的 SAL（Source Code Annotation Language）标记，用于静态分析。它们对编译没有影响，但出现在大量函数声明中：

```cpp
HRESULT STDMETHODCALLTYPE Read(
    /* [annotation][write_to] */
    IN void* pv,           // IN: 输入参数
    IN ULONG cb,           // IN: 输入参数
    OUT ULONG* pcbRead);   // OUT: 输出参数
```

在非 Windows 平台上，把它们定义为空宏就能让这些声明编译通过。

---

### 第二层：tjsTypes.h —— TJS 脚本引擎类型

TJS2 是 KiriKiri 的内嵌脚本语言（类似 JavaScript 的角色）。它定义了一套独立的类型系统，这套类型系统从一开始就是跨平台的：

```cpp
// cpp/core/tjs2/tjsTypes.h — KrKr2 实际源码
#pragma once
#include <cstdint>
#include <stddef.h>    // ptrdiff_t

/* ========== TJS 固定宽度整数 ========== */
typedef std::int8_t   tjs_int8;    // 8 位有符号
typedef std::uint8_t  tjs_uint8;   // 8 位无符号
typedef std::int16_t  tjs_int16;   // 16 位有符号
typedef std::uint16_t tjs_uint16;  // 16 位无符号
typedef std::int32_t  tjs_int32;   // 32 位有符号
typedef std::uint32_t tjs_uint32;  // 32 位无符号
typedef std::int64_t  tjs_int64;   // 64 位有符号
typedef std::uint64_t tjs_uint64;  // 64 位无符号

/* ========== TJS 基础类型 ========== */
typedef char16_t    tjs_char;      // TJS 字符 = UTF-16 码元
typedef char        tjs_nchar;     // 窄字符（ASCII/UTF-8）
typedef double      tjs_real;      // 浮点数（64 位双精度）
typedef int         tjs_int;       // 平台原生 int
typedef unsigned int tjs_uint;     // 平台原生 unsigned int

/* ========== 指针大小相关 ========== */
typedef intptr_t    tjs_intptr_t;  // 能存放指针的有符号整数
typedef uintptr_t   tjs_uintptr_t; // 能存放指针的无符号整数
typedef size_t      tjs_size;      // 大小类型
typedef ptrdiff_t   tjs_offset;    // 偏移量类型

/* ========== TJS 虚拟机类型 ========== */
typedef tjs_int32   tjs_error;     // 错误码
typedef tjs_int64   tTVInteger;    // TJS 整数值（64 位）
typedef tjs_real    tTVReal;       // TJS 实数值（double）

/* ========== 字符串字面量宏 ========== */
#define TJS_W(X) u##X    // TJS 宽字符串: TJS_W("hello") → u"hello"
#define TJS_N(X) X       // TJS 窄字符串: TJS_N("hello") → "hello"
```

#### 为什么 TJS 不直接用 `<cstdint>`？

你可能会问：既然 `tjs_int32` 就是 `std::int32_t`，为什么要多此一举加一层 typedef？三个原因：

1. **历史兼容性**：TJS2 诞生于 2000 年，当时 `<cstdint>` 还不是标准的一部分（C++11 才正式加入）。TJS 需要自己的类型别名来适配不同编译器。

2. **语义清晰**：`tjs_char` 明确告诉你"这是 TJS 脚本引擎的字符类型"，而 `char16_t` 只告诉你"这是 16 位字符"。当你看到 `tjs_char*` 时立刻知道这是 TJS 字符串，而不是随便哪个 UTF-16 字符串。

3. **可替换性**：如果将来需要把 TJS 的字符编码从 UTF-16 改为 UTF-32，只需修改一处 typedef，而不用全局替换 `char16_t`。

#### typedefine.h 与 tjsTypes.h 的依赖关系

注意 `typedefine.h` 在非 Windows 平台上会 `#include "tjsTypes.h"`。这是因为非 Windows 平台需要用 `tjs_char` 来定义 `LPWSTR`：

```cpp
// typedefine.h 末尾（非 Windows 平台）
typedef tjs_char*       LPWSTR;   // "Long Pointer to Wide STRing"
typedef const tjs_char* LPCWSTR;  // "Long Pointer to Const Wide STRing"
```

在 Windows 上，`LPWSTR` 是 `wchar_t*`（16 位），而在非 Windows 平台上，KrKr2 用 `tjs_char`（也是 16 位的 `char16_t`）来替代。这保证了宽字符串在所有平台上都是 16 位编码。

---

### 第三层：combase.h —— COM 接口模拟

COM（Component Object Model）是 Windows 的组件技术。原版 KiriKiri 大量使用 COM 接口（`IStream`、`IUnknown`），特别是在文件 I/O 和插件系统中。KrKr2 的 `combase.h` 在非 Windows 平台上完整模拟了这些接口：

```cpp
// cpp/core/environ/combase.h — KrKr2 实际源码（精简注释版）
#pragma once
#include "typedefine.h"    // 依赖第一层的类型定义

#ifdef _WIN32
#include <objidl.h>        // Windows: 直接用 SDK 的 COM 定义
#endif

// 非 Windows: 模拟 COM 错误码常量
#ifndef S_OK
#define S_OK           ((HRESULT)0L)          // 成功
#define S_FALSE        ((HRESULT)1L)          // 成功但无数据
#define E_FAIL         ((HRESULT)0x80004005L) // 通用失败
#define E_NOTIMPL      ((HRESULT)0x80004001L) // 未实现
#define E_INVALIDARG   ((HRESULT)0x80000003L) // 参数无效
#define E_OUTOFMEMORY  ((HRESULT)0x8007000EL) // 内存不足

// HRESULT 检查宏
#define FAILED(hr)     (((HRESULT)(hr)) < 0)
#define SUCCEEDED(hr)  (((HRESULT)(hr)) >= 0)

// IID 结构体（接口标识符）
struct IID {
    uint32_t x;
    uint16_t s1;
    uint16_t s2;
    uint8_t  c[8];
};
#define REFIID const IID&
#endif

// 调用约定宏
#ifndef STDMETHODCALLTYPE
#ifdef _WIN32
#define STDMETHODCALLTYPE __stdcall  // Windows: 使用 stdcall
#else
#define STDMETHODCALLTYPE            // 其他平台: 空（使用默认 cdecl）
#endif

// 模拟 COM 基础接口 IUnknown
class IUnknown {
public:
    virtual HRESULT STDMETHODCALLTYPE QueryInterface(
        REFIID riid, void** ppvObject) = 0;
    virtual ULONG STDMETHODCALLTYPE AddRef() = 0;
    virtual ULONG STDMETHODCALLTYPE Release() = 0;
};

// 模拟流接口 ISequentialStream → IStream
class ISequentialStream : public IUnknown {
public:
    virtual HRESULT STDMETHODCALLTYPE Read(
        void* pv, ULONG cb, ULONG* pcbRead) = 0;
    virtual HRESULT STDMETHODCALLTYPE Write(
        const void* pv, ULONG cb, ULONG* pcbWritten) = 0;
};

struct IStream : public ISequentialStream {
public:
    virtual HRESULT STDMETHODCALLTYPE Seek(
        LARGE_INTEGER dlibMove, DWORD dwOrigin,
        ULARGE_INTEGER* plibNewPosition) = 0;
    virtual HRESULT STDMETHODCALLTYPE SetSize(
        ULARGE_INTEGER libNewSize) = 0;
    // ... Commit, Revert, LockRegion, Stat, Clone 等方法
};
#endif
```

#### 为什么要模拟整个 COM 体系？

因为 KiriKiri 的文件系统完全基于 `IStream` 接口。比如 XP3 档案（KiriKiri 的自定义打包格式）的读取：

```cpp
// cpp/core/base/XP3Archive.h 中使用 IStream
class XP3Archive {
    IStream* OpenStream(const tjs_char* name);  // 返回 COM 流接口
};
```

如果不模拟 `IStream`，整个文件 I/O 子系统就无法编译。模拟它比重写文件系统代价小得多。

---

### 第四层：vkdefine.h —— 虚拟键码常量

Windows 定义了一套虚拟键码（Virtual Key Codes），用 `VK_` 前缀标识键盘按键。KiriKiri 的脚本系统（TJS）和输入处理都依赖这些常量：

```cpp
// cpp/core/environ/vkdefine.h — KrKr2 实际源码（节选）
#pragma once

// 字母键
#define VK_A 0x41
#define VK_B 0x42
// ... A-Z 对应 0x41-0x5A

// 功能键
#define VK_F1     0x70
#define VK_F2     0x71
// ... F1-F24

// 控制键
#define VK_RETURN  0x0D    // 回车
#define VK_ESCAPE  0x1B    // ESC
#define VK_SPACE   0x20    // 空格
#define VK_BACK    0x08    // 退格
#define VK_TAB     0x09    // Tab
#define VK_SHIFT   0x10    // Shift
#define VK_CONTROL 0x11    // Ctrl

// 方向键
#define VK_LEFT    0x25
#define VK_UP      0x26
#define VK_RIGHT   0x27
#define VK_DOWN    0x28

// 数字小键盘
#define VK_NUMPAD0 0x60
#define VK_NUMPAD1 0x61
// ... NUMPAD0-9

// OEM 键（标点符号，因键盘布局而异）
#define VK_OEM_1      0xBA   // ';:' (US 布局)
#define VK_OEM_PLUS   0xBB   // '+' (所有布局)
#define VK_OEM_COMMA  0xBC   // ',' (所有布局)
#define VK_OEM_MINUS  0xBD   // '-' (所有布局)
#define VK_OEM_PERIOD 0xBE   // '.' (所有布局)
```

#### 为什么不重新设计键码系统？

同样是务实考虑。TJS 脚本中用硬编码的数字引用这些键码：

```javascript
// TJS 脚本示例（游戏脚本可能这样写）
function onKeyDown(key) {
    if (key == 0x0D) {  // VK_RETURN
        advanceText();
    }
}
```

如果换一套键码系统，所有已有的 TJS 游戏脚本都会出错。保持 Windows 虚拟键码的数值不变，是兼容性的硬要求。

---

## 类型兼容层的设计原则

从 KrKr2 的实践中，我们可以提炼出五条设计原则：

### 原则一：大小必须精确匹配

```cpp
// ✅ 正确：Windows DWORD = 32 位无符号，用 uint32_t 精确匹配
typedef uint32_t DWORD;

// ❌ 错误：unsigned long 在 64 位 Linux 上是 64 位！
typedef unsigned long DWORD;  // 大小不匹配，会导致序列化/结构体布局错误
```

这一点在处理文件格式时尤其关键。XP3 档案头中有这样的结构：

```cpp
struct XP3Header {
    BYTE  magic[11];    // 必须是 11 字节
    DWORD indexOffset;  // 必须是 4 字节，不能是 8 字节
};
```

如果 `DWORD` 在某个平台变成了 8 字节，文件解析就会完全错乱。

### 原则二：有符号性必须匹配

```cpp
// ✅ 正确：HRESULT 在 Windows 上是有符号 32 位
typedef int32_t LONG;
typedef LONG    HRESULT;

// ❌ 错误：无符号类型永远 >= 0，FAILED() 宏永远返回 false
typedef uint32_t HRESULT;  // FAILED() 检查 < 0，永远为 false！
```

### 原则三：条件编译避免重定义

```cpp
// ✅ 正确：只在非 Windows 平台定义
#if !TARGET_WINDOWS
typedef uint32_t DWORD;
#endif

// ❌ 错误：无条件定义，Windows 上与 SDK 冲突
typedef uint32_t DWORD;  // error: 'DWORD': redefinition
```

### 原则四：句柄用不透明指针

```cpp
// ✅ 正确：不透明指针，调用者不能解引用
typedef void* HBITMAP;

// ❌ 错误：暴露了内部结构
typedef struct { int width; int height; } HBITMAP;
```

### 原则五：常量值必须与原始定义一致

```cpp
// ✅ 正确：与 Windows SDK 的 winerror.h 中的定义一致
#define S_OK    ((HRESULT)0L)
#define E_FAIL  ((HRESULT)0x80004005L)

// ❌ 错误：随便编一个错误码
#define E_FAIL  ((HRESULT)-1)  // 与 Windows 不兼容
```

---

## 常见错误与陷阱

### 错误一：unsigned long 在不同平台大小不同

```cpp
// ❌ 很多初学者会这样写
typedef unsigned long DWORD;
```

| 平台 | `unsigned long` 大小 | 期望 `DWORD` 大小 | 匹配？ |
|------|---------------------|-------------------|--------|
| Windows (MSVC) | 4 字节 | 4 字节 | ✅ |
| Linux x86_64 (GCC) | **8 字节** | 4 字节 | ❌ |
| macOS arm64 (Clang) | **8 字节** | 4 字节 | ❌ |
| Android arm64 (NDK) | **8 字节** | 4 字节 | ❌ |

**修复**：始终用 `<stdint.h>` 的固定宽度类型：

```cpp
// ✅ 所有平台上都是精确的 4 字节
typedef uint32_t DWORD;
```

### 错误二：忘记模拟 Windows 宏

Windows 代码经常使用 `MAX_PATH` 这样的宏：

```cpp
char path[MAX_PATH];  // Linux 上编译失败：MAX_PATH 未定义
```

KrKr2 的解决方案：

```cpp
// typedefine.h 中
#if !TARGET_WINDOWS || !defined(MAX_PATH)
#define MAX_PATH 260    // 与 Windows 的定义一致
#endif
```

注意条件是 `!TARGET_WINDOWS || !defined(MAX_PATH)`——即使在 Windows 上，如果某些头文件没有被包含导致 `MAX_PATH` 未定义，也要提供默认值。

### 错误三：忽略结构体对齐差异

```cpp
// 这个结构体在不同编译器上可能有不同的大小
typedef struct _FILETIME {
    DWORD dwLowDateTime;
    DWORD dwHighDateTime;
} FILETIME;
```

在大多数情况下，两个 `DWORD`（各 4 字节）紧密排列，结构体总共 8 字节。但某些编译器或优化选项可能插入填充字节。如果结构体用于读写文件，必须确保对齐一致：

```cpp
// ✅ 安全做法：显式指定无填充
#pragma pack(push, 1)
typedef struct _FILETIME {
    DWORD dwLowDateTime;
    DWORD dwHighDateTime;
} FILETIME;
#pragma pack(pop)
```

KrKr2 的 `FILETIME` 主要用于内存中的时间戳比较，不直接序列化到文件，所以没有使用 `#pragma pack`。但在 XP3 档案头等需要精确布局的场景中，必须注意这一点。

---

## 动手实践

### 实践 1：编写一个迷你类型兼容层

创建一个独立的类型兼容层，然后用它编写跨平台代码。

**步骤 1：** 创建 `compat_types.h`

```cpp
// compat_types.h — 迷你 Windows 类型兼容层
#pragma once
#include <cstdint>

// ===== 平台检测 =====
#if defined(_WIN32) || defined(_WIN64)
    #define COMPAT_WINDOWS 1
    #include <windows.h>
#else
    #define COMPAT_WINDOWS 0

    // 整数类型
    typedef int32_t   LONG;
    typedef uint32_t  ULONG;
    typedef uint32_t  DWORD;
    typedef uint16_t  WORD;
    typedef uint8_t   BYTE;
    typedef int32_t   BOOL;

    // 句柄类型
    typedef void*     HANDLE;
    typedef void*     HBITMAP;
    typedef LONG      HRESULT;

    // 指针类型
    typedef void*     LPVOID;
    typedef const char* LPCSTR;

    // 常量
    #define TRUE  1
    #define FALSE 0
    #define INVALID_HANDLE_VALUE ((HANDLE)(intptr_t)-1)
    #define MAX_PATH 260

    // HRESULT 错误码
    #define S_OK          ((HRESULT)0L)
    #define S_FALSE       ((HRESULT)1L)
    #define E_FAIL        ((HRESULT)0x80004005L)
    #define E_INVALIDARG  ((HRESULT)0x80000003L)

    // HRESULT 检查宏
    #define SUCCEEDED(hr) (((HRESULT)(hr)) >= 0)
    #define FAILED(hr)    (((HRESULT)(hr)) < 0)
#endif
```

**步骤 2：** 编写使用兼容层的跨平台代码

```cpp
// main.cpp — 使用兼容层的测试程序
#include "compat_types.h"
#include <cstdio>
#include <cstring>

// 模拟一个使用 Windows 类型的旧函数
HRESULT ProcessData(const BYTE* data, DWORD size, DWORD* outChecksum) {
    if (data == nullptr || size == 0) {
        return E_INVALIDARG;
    }

    DWORD checksum = 0;
    for (DWORD i = 0; i < size; i++) {
        checksum += data[i];           // 逐字节累加
        checksum = (checksum << 1) | (checksum >> 31);  // 循环左移 1 位
    }

    *outChecksum = checksum;
    return S_OK;
}

int main() {
    // 验证类型大小
    printf("sizeof(DWORD)  = %zu (expect 4)\n", sizeof(DWORD));
    printf("sizeof(WORD)   = %zu (expect 2)\n", sizeof(WORD));
    printf("sizeof(BYTE)   = %zu (expect 1)\n", sizeof(BYTE));
    printf("sizeof(LONG)   = %zu (expect 4)\n", sizeof(LONG));
    printf("sizeof(HRESULT)= %zu (expect 4)\n", sizeof(HRESULT));

    // 使用兼容层的函数
    const char* message = "Hello, cross-platform!";
    DWORD checksum = 0;

    HRESULT hr = ProcessData(
        (const BYTE*)message,
        (DWORD)strlen(message),
        &checksum
    );

    if (SUCCEEDED(hr)) {
        printf("Checksum: 0x%08X\n", checksum);
    } else {
        printf("Error: HRESULT = 0x%08X\n", (unsigned int)hr);
    }

    // 测试错误路径
    hr = ProcessData(nullptr, 0, &checksum);
    if (FAILED(hr)) {
        printf("Correctly rejected null input: 0x%08X\n",
               (unsigned int)hr);
    }

    return 0;
}
```

**步骤 3：** 四平台编译验证

```bash
# Windows (MSVC)
cl /EHsc /std:c++17 /Fe:test.exe main.cpp
test.exe

# Linux (GCC)
g++ -std=c++17 -o test main.cpp
./test

# macOS (Clang)
clang++ -std=c++17 -o test main.cpp
./test

# Android (NDK, 交叉编译)
$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android21-clang++ \
    -std=c++17 -o test main.cpp
# 需要 adb push 到设备运行
adb push test /data/local/tmp/
adb shell /data/local/tmp/test
```

**期望输出（所有平台一致）：**

```
sizeof(DWORD)  = 4 (expect 4)
sizeof(WORD)   = 2 (expect 2)
sizeof(BYTE)   = 1 (expect 1)
sizeof(LONG)   = 4 (expect 4)
sizeof(HRESULT)= 4 (expect 4)
Checksum: 0x5B2E8944
Correctly rejected null input: 0x80000003
```

### 实践 2：为 KrKr2 添加一个新的 PAL 函数

按照上一节学的文件分离模式，结合本节的类型兼容层知识，为 KrKr2 添加一个获取 CPU 型号的函数。

**步骤 1：** 在 `Platform.h` 中声明接口（平台无关）

```cpp
// 在 cpp/core/environ/Platform.h 末尾添加
#include <string>

// 获取 CPU 型号字符串
// 返回值示例：
//   Windows: "Intel(R) Core(TM) i7-12700H"
//   Linux:   "Intel(R) Core(TM) i7-12700H"
//   macOS:   "Apple M1 Pro"
//   Android: "Qualcomm Snapdragon 888"
std::string TVPGetCPUModelString();
```

**步骤 2：** Windows 实现 — 使用 CPUID 指令

```cpp
// cpp/core/environ/win32/Platform.cpp 中添加
#include <intrin.h>   // MSVC 内建函数
#include <string>
#include <cstring>

std::string TVPGetCPUModelString() {
    int cpuInfo[4] = {};
    char brand[48] = {};    // CPUID brand string 最长 48 字节

    // CPUID 0x80000002-0x80000004 返回品牌字符串
    // 每次调用填充 16 字节（4 个 int）
    __cpuid(cpuInfo, 0x80000002);
    std::memcpy(brand, cpuInfo, sizeof(cpuInfo));

    __cpuid(cpuInfo, 0x80000003);
    std::memcpy(brand + 16, cpuInfo, sizeof(cpuInfo));

    __cpuid(cpuInfo, 0x80000004);
    std::memcpy(brand + 32, cpuInfo, sizeof(cpuInfo));

    // 去除末尾空格
    std::string result(brand);
    while (!result.empty() && result.back() == ' ') {
        result.pop_back();
    }
    return result;
}
```

**步骤 3：** Linux 实现 — 读取 `/proc/cpuinfo`

```cpp
// cpp/core/environ/linux/Platform.cpp 中添加
#include <fstream>
#include <string>

std::string TVPGetCPUModelString() {
    std::ifstream cpuinfo("/proc/cpuinfo");
    if (!cpuinfo.is_open()) {
        return "Unknown CPU";
    }

    std::string line;
    while (std::getline(cpuinfo, line)) {
        // x86 Linux: 字段名是 "model name"
        if (line.find("model name") != std::string::npos) {
            auto pos = line.find(':');
            if (pos != std::string::npos && pos + 2 < line.size()) {
                return line.substr(pos + 2);  // 跳过 ": "
            }
        }
    }
    return "Unknown CPU";
}
```

**步骤 4：** macOS 实现 — 使用 sysctl

```cpp
// cpp/core/environ/apple/macos/platform.mm 中添加
#include <sys/sysctl.h>
#include <string>

std::string TVPGetCPUModelString() {
    char brand[256] = {};
    size_t size = sizeof(brand);

    // machdep.cpu.brand_string: Intel CPU 的品牌字符串
    // 如果失败（Apple Silicon），尝试 hw.model
    if (sysctlbyname("machdep.cpu.brand_string", brand, &size,
                     nullptr, 0) == 0) {
        return std::string(brand);
    }

    // Apple Silicon (M1/M2/...) 没有 brand_string
    size = sizeof(brand);
    if (sysctlbyname("hw.model", brand, &size, nullptr, 0) == 0) {
        return std::string(brand);
    }

    return "Unknown Apple CPU";
}
```

**步骤 5：** Android 实现 — 读取 `/proc/cpuinfo`（ARM 变体）

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 中添加
#include <fstream>
#include <string>

std::string TVPGetCPUModelString() {
    std::ifstream cpuinfo("/proc/cpuinfo");
    if (!cpuinfo.is_open()) {
        return "Unknown ARM CPU";
    }

    std::string line;
    while (std::getline(cpuinfo, line)) {
        // ARM 上的字段名与 x86 不同：
        //   ARM: "Hardware"（整个 SoC 名称）
        //   x86: "model name"（CPU 具体型号）
        if (line.find("Hardware") != std::string::npos ||
            line.find("model name") != std::string::npos) {
            auto pos = line.find(':');
            if (pos != std::string::npos && pos + 2 < line.size()) {
                return line.substr(pos + 2);
            }
        }
    }
    return "Unknown ARM CPU";
}
```

注意 Android 和 Linux 的代码很相似（都读 `/proc/cpuinfo`），但搜索的字段名不同。Android 设备的 `/proc/cpuinfo` 通常有 `Hardware` 字段而没有 `model name`（x86 Android 模拟器除外）。

**不需要修改 CMakeLists.txt**——因为这些 `.cpp` 文件已经被 CMake 按平台选择编译了（这就是文件分离的好处）。

---

## 对照项目源码

### 关键文件路径

| 文件 | 作用 | 行数 |
|------|------|------|
| `cpp/core/environ/typedefine.h` | Windows 基础类型兼容层 | 115 行 |
| `cpp/core/tjs2/tjsTypes.h` | TJS 脚本引擎类型定义 | 155 行 |
| `cpp/core/environ/combase.h` | COM 接口模拟层 | 159 行 |
| `cpp/core/environ/vkdefine.h` | 虚拟键码常量定义 | 156 行 |
| `cpp/core/environ/cpu_types.h` | CPU 特征检测标志 | 54 行 |

### 类型使用分布

通过搜索代码库，可以看到这些兼容类型被广泛使用：

- `DWORD` — 出现在 13 个头文件中（档案解析、线程、窗口、音频、图像加载）
- `HRESULT` — 出现在 8 个头文件中（COM 流接口、插件系统、渲染设备）
- `tjs_char` / `tjs_int` — 出现在整个 TJS2 引擎和所有脚本相关模块中
- `VK_*` — 出现在输入处理和 TJS 脚本系统中

这说明类型兼容层不是可有可无的，而是整个引擎赖以编译的基础设施。

---

## 本节小结

- **类型兼容层**是跨平台移植的务实策略——用 typedef 在非目标平台上模拟原始平台的类型，让现有代码无需大量修改就能编译
- KrKr2 有**四层**类型兼容体系：`typedefine.h`（Windows 基础类型）、`tjsTypes.h`（TJS 脚本类型）、`combase.h`（COM 接口模拟）、`vkdefine.h`（虚拟键码常量）
- 类型别名必须**大小精确匹配**——永远用 `<stdint.h>` 的固定宽度类型（`uint32_t`），不要用 `unsigned long`（平台间大小不同）
- `HRESULT` 必须是**有符号类型**，否则 `FAILED()` 宏无法正常工作
- 句柄类型用 **`void*`（不透明指针）** 模拟——只需编译通过，不需要真的操作 Windows 资源
- 常量值（错误码、虚拟键码）必须与 Windows SDK 的定义**数值一致**，否则会破坏脚本兼容性
- 在 Windows 上，兼容层**什么都不做**（直接用原生 SDK），条件编译确保不重定义

---

## 练习题与答案

### 题目 1：为什么 KrKr2 把 `HRESULT` 定义为 `LONG`（有符号 32 位）而不是 `DWORD`（无符号 32 位）？

<details>
<summary>查看答案</summary>

`HRESULT` 的错误检查机制依赖**符号位**：

```cpp
#define FAILED(hr)    (((HRESULT)(hr)) < 0)
#define SUCCEEDED(hr) (((HRESULT)(hr)) >= 0)
```

- 成功码（如 `S_OK = 0`、`S_FALSE = 1`）是非负数
- 错误码（如 `E_FAIL = 0x80004005`）最高位为 1，作为有符号数解释时是负数

如果把 `HRESULT` 定义为 `DWORD`（无符号），那么 `0x80004005` 就是一个很大的正数，`< 0` 永远为 `false`，`FAILED()` 宏永远不会触发，所有错误都会被当作成功处理。

```cpp
// 错误示范
typedef uint32_t HRESULT;  // 无符号
HRESULT hr = E_FAIL;       // hr = 0x80004005 = 2147500037
if (FAILED(hr)) {          // 2147500037 < 0 → false！
    // 永远不会进来
}
```

</details>

### 题目 2：下面的类型兼容层有三处错误，找出并修正

```cpp
#pragma once

typedef unsigned long DWORD;
typedef unsigned int  WORD;
typedef DWORD         HRESULT;

#define S_OK    0
#define E_FAIL  -1

#define SUCCEEDED(hr) ((hr) >= 0)
```

<details>
<summary>查看答案</summary>

**错误 1：** `unsigned long` 在 64 位 Linux/macOS 上是 8 字节，不是 4 字节。

```cpp
// ❌ typedef unsigned long DWORD;
// ✅
typedef uint32_t DWORD;
```

**错误 2：** `unsigned int` 是 4 字节，但 `WORD` 应该是 2 字节。

```cpp
// ❌ typedef unsigned int WORD;
// ✅
typedef uint16_t WORD;
```

**错误 3：** `HRESULT` 被定义为 `DWORD`（无符号），`SUCCEEDED()` 的 `>= 0` 检查对无符号类型永远为 true。而且 `E_FAIL` 的值不对——Windows 的 `E_FAIL` 是 `0x80004005`，不是 `-1`。

```cpp
// ❌ typedef DWORD HRESULT;
// ✅
typedef int32_t HRESULT;

// ❌ #define E_FAIL -1
// ✅
#define S_OK   ((HRESULT)0L)
#define E_FAIL ((HRESULT)0x80004005L)

// SUCCEEDED 也需要显式转型确保安全
#define SUCCEEDED(hr) (((HRESULT)(hr)) >= 0)
```

完整修正版：

```cpp
#pragma once
#include <cstdint>

typedef uint32_t DWORD;    // 精确 4 字节无符号
typedef uint16_t WORD;     // 精确 2 字节无符号
typedef int32_t  HRESULT;  // 有符号！错误码依赖符号位

#define S_OK   ((HRESULT)0L)
#define E_FAIL ((HRESULT)0x80004005L)

#define SUCCEEDED(hr) (((HRESULT)(hr)) >= 0)
#define FAILED(hr)    (((HRESULT)(hr)) < 0)
```

</details>

### 题目 3：为什么 `typedefine.h` 在文件末尾有 `#undef TARGET_WINDOWS`？这是好的实践还是坏的实践？

<details>
<summary>查看答案</summary>

`typedefine.h` 末尾确实有 `#undef TARGET_WINDOWS`（第 115 行）。这意味着 `TARGET_WINDOWS` 只在 `typedefine.h` 内部有效，包含该头文件的其他代码**不能**使用 `TARGET_WINDOWS`。

**这是一种防御性编程**——防止外部代码依赖 `typedefine.h` 的内部实现宏。外部代码应该使用 `Platform.h` 中的标准平台检测宏（如 `WINDOWS`、`LINUX`、`MACOS`、`ANDROID`），而不是 `TARGET_WINDOWS`。

但这也有争议：

- **支持**：符合"最小暴露原则"，宏只在需要的范围内存在
- **反对**：如果其他头文件意外地在 `typedefine.h` 之后检查 `TARGET_WINDOWS`，会得到未定义行为（宏不存在 ≠ 宏为 0）

在 KrKr2 中这是合理的，因为项目有专门的 `Platform.h` 提供标准化的平台宏，`typedefine.h` 的 `TARGET_WINDOWS` 纯属内部使用。

</details>

---

## 下一步

[04-常见错误与总结](./04-常见错误与总结.md) — 汇总 PAL 设计模式的常见错误，并对本章（文件分离、PIMPL、类型兼容层）进行综合回顾。
