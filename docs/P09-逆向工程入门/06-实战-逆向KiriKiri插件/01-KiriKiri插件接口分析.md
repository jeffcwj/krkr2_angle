# KiriKiri 插件接口分析

## 本節目標

- 理解 KiriKiri2 插件系统的整体架构和加载流程
- 掌握 V2Link/V2Unlink 导出函数的作用和调用时机
- 深入理解 iTVPFunctionExporter 接口的设计思想
- 掌握 tp_stub.h 的核心机制——如何在不链接 krkr.exe 的情况下调用引擎函数
- 理解 iTJSDispatch2 接口的虚函数表布局
- 学会使用 ncbind 宏系统注册 TJS 类和函数

## 6.1.1 插件系统架构总览

### KiriKiri2 插件加载流程

```
krkr.exe 启动
    │
    ├─ 1. 初始化 TJS2 脚本引擎
    │
    ├─ 2. 扫描 plugin/ 目录下的 DLL 文件
    │
    ├─ 3. 对每个 DLL 执行:
    │     │
    │     ├─ LoadLibrary(dll_path)          ← 加载 DLL 到进程空间
    │     │
    │     ├─ GetProcAddress("V2Link")       ← 获取入口函数
    │     │
    │     ├─ V2Link(exporter)               ← 调用插件初始化
    │     │   │                                参数: iTVPFunctionExporter*
    │     │   │
    │     │   ├─ TVPInitImportStub(exporter) ← 插件通过 exporter 获取引擎函数
    │     │   │
    │     │   └─ 注册 TJS 类/函数            ← 通过 ncbind 或手动注册
    │     │
    │     └─ 记录插件信息（句柄、V2Unlink 地址）
    │
    └─ 4. 执行 TJS 脚本（脚本中可使用插件注册的类/函数）

krkr.exe 退出
    │
    └─ 对每个已加载的插件:
          │
          ├─ V2Unlink()                    ← 调用插件清理
          │
          └─ FreeLibrary(dll_handle)       ← 卸载 DLL
```

### 关键组件关系

```
┌──────────────────────────────────────────────────────────┐
│                     krkr.exe (主程序)                     │
│                                                          │
│  ┌──────────────────┐    ┌──────────────────────────┐   │
│  │   TJS2 引擎       │    │   iTVPFunctionExporter    │   │
│  │  ┌──────────────┐ │    │   (函数导出器)             │   │
│  │  │ iTJSDispatch2│ │    │                          │   │
│  │  │  ├─ PropGet  │ │    │  QueryFunctionsByNarrow  │   │
│  │  │  ├─ PropSet  │ │    │  QueryFunctions          │   │
│  │  │  ├─ FuncCall │ │    └──────────┬───────────────┘   │
│  │  │  └─ ...      │ │               │                   │
│  │  └──────────────┘ │               │                   │
│  └──────────────────┘               │                   │
│                                      │                   │
└──────────────────────────────────────┼───────────────────┘
                                       │ V2Link(exporter)
                                       ▼
┌──────────────────────────────────────────────────────────┐
│                    plugin.dll (插件)                      │
│                                                          │
│  ┌──────────────────────────────────────┐               │
│  │  tp_stub.h / tp_stub.cpp             │               │
│  │  (引擎函数的本地代理)                  │               │
│  │                                      │               │
│  │  TVPInitImportStub(exporter)         │               │
│  │  → 通过 exporter 获取所有引擎函数指针   │               │
│  │  → 填充全局函数指针表                  │               │
│  │  → 之后插件可以像直接链接一样调用引擎函数 │               │
│  └──────────────────────────────────────┘               │
│                                                          │
│  ┌──────────────────────────────────────┐               │
│  │  插件业务代码                          │               │
│  │  使用 ncbind 注册 TJS 类/函数          │               │
│  │  调用 TJS API 操作脚本对象             │               │
│  └──────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────┘
```

## 6.1.2 V2Link 与 V2Unlink

### V2Link 函数

V2Link 是插件的入口函数，在 DLL 被加载后由 krkr.exe 调用：

```cpp
// tp_stub.h 中的声明
extern "C" {
    HRESULT __stdcall V2Link(iTVPFunctionExporter* exporter);
}

// 典型实现
HRESULT __stdcall V2Link(iTVPFunctionExporter* exporter) {
    // 步骤 1: 初始化导入存根
    // 这一步至关重要——它让插件能够调用引擎内部函数
    TVPInitImportStub(exporter);
    
    // 步骤 2: 注册插件功能
    // 方式 A: 使用 ncbind 宏（推荐）
    // NCB_PRE_REGIST_CALLBACK 和 NCB_REGISTER_CLASS 自动处理
    
    // 方式 B: 手动注册 TJS 类
    // TVPRegisterPlugin(&MyTJSClass);
    
    return S_OK;  // 返回 S_OK (0) 表示成功
}
```

### V2Unlink 函数

```cpp
extern "C" {
    HRESULT __stdcall V2Unlink();
}

HRESULT __stdcall V2Unlink() {
    // 清理插件资源
    // 取消注册 TJS 类
    // 释放分配的内存
    
    return S_OK;
}
```

### DEF 文件导出

插件通常使用 `.def` 文件导出这两个函数：

```def
; plugin.def
LIBRARY plugin
EXPORTS
    V2Link    @1
    V2Unlink  @2
```

或者使用 `__declspec(dllexport)`：

```cpp
#ifdef _WIN32
#define PLUGIN_EXPORT __declspec(dllexport)
#else
#define PLUGIN_EXPORT __attribute__((visibility("default")))
#endif

extern "C" {
    PLUGIN_EXPORT HRESULT __stdcall V2Link(iTVPFunctionExporter* exporter);
    PLUGIN_EXPORT HRESULT __stdcall V2Unlink();
}
```

## 6.1.3 iTVPFunctionExporter 详解

### 接口设计

iTVPFunctionExporter 是 krkr.exe 传递给插件的接口，允许插件动态获取引擎内部函数的地址：

```cpp
// tp_stub.h 中的定义
class iTVPFunctionExporter {
public:
    // 通过窄字符函数名查询
    // name: 函数名（如 "void TVPAddLog(const ttstr &)"）
    // ptr:  输出参数，接收函数指针
    // 返回: true = 找到，false = 未找到
    virtual bool TJS_INTF_METHOD QueryFunctions(
        const tjs_char** name,
        void** ptr,
        tjs_uint count
    ) = 0;
    
    virtual bool TJS_INTF_METHOD QueryFunctionsByNarrow(
        const char** name,
        void** ptr,
        tjs_uint count
    ) = 0;
};
```

### 工作原理

krkr.exe 内部维护了一个函数注册表，将函数签名字符串映射到函数指针：

```cpp
// krkr.exe 内部（简化）
struct FunctionEntry {
    const char* signature;   // "void TVPAddLog(const ttstr &)"
    void*       address;     // 函数的实际地址
};

// 注册表
static std::vector<FunctionEntry> g_exportedFunctions = {
    {"void TVPAddLog(const ttstr &)",           (void*)&TVPAddLog},
    {"iTJSDispatch2 * TVPGetScriptDispatch()", (void*)&TVPGetScriptDispatch},
    {"void TVPExecuteExpression(const ttstr &)",(void*)&TVPExecuteExpression},
    // ... 数百个引擎函数
};
```

### tp_stub.cpp 的导入过程

```cpp
// tp_stub.cpp 中的核心逻辑（简化）
static iTVPFunctionExporter* TVPFunctionExporter = NULL;

// 函数指针声明（全局变量）
static void (*proc_TVPAddLog)(const ttstr&) = NULL;
static iTJSDispatch2* (*proc_TVPGetScriptDispatch)() = NULL;
// ... 更多函数指针

void TVPInitImportStub(iTVPFunctionExporter* exporter) {
    TVPFunctionExporter = exporter;
    
    // 批量查询函数地址
    const char* names[] = {
        "void TVPAddLog(const ttstr &)",
        "iTJSDispatch2 * TVPGetScriptDispatch()",
        // ...
    };
    void* ptrs[] = {
        (void**)&proc_TVPAddLog,
        (void**)&proc_TVPGetScriptDispatch,
        // ...
    };
    
    exporter->QueryFunctionsByNarrow(names, ptrs, sizeof(names)/sizeof(names[0]));
    
    // 之后插件代码就可以这样调用:
    // proc_TVPAddLog(someString);  等价于直接调用 krkr.exe 的 TVPAddLog
}

// 包装函数（让调用方更自然）
void TVPAddLog(const ttstr& msg) {
    if (proc_TVPAddLog) proc_TVPAddLog(msg);
}
```

### 这种设计的意义

```
问题：插件是独立的 DLL，如何调用 krkr.exe 内部的函数？

方案 1（不可行）：直接链接
  插件不能链接 krkr.exe 的 .lib，因为：
  - krkr.exe 是 EXE 不是 DLL，没有导出表
  - 不同版本的 krkr.exe 函数地址不同

方案 2（KiriKiri 采用）：运行时动态获取
  krkr.exe 在调用 V2Link 时传入 exporter 接口
  插件通过函数签名字符串查询函数指针
  优点：
  - 插件与 krkr.exe 版本解耦
  - 插件只依赖 tp_stub.h/cpp，不依赖 krkr.exe 的任何 .lib
  - 允许渐进式 API 扩展（新版本添加新函数，旧插件不受影响）
```

## 6.1.4 iTJSDispatch2 虚函数表

### 接口定义

iTJSDispatch2 是 TJS2 脚本引擎中所有对象的基础接口。理解它的虚函数表是逆向 KiriKiri2 插件的关键：

```cpp
// tjsInterfaces.h 中的定义（简化）
class iTJSDispatch2 {
public:
    // 引用计数
    virtual tjs_uint TJS_INTF_METHOD AddRef() = 0;           // vtable[0]
    virtual tjs_uint TJS_INTF_METHOD Release() = 0;          // vtable[1]
    
    // 函数调用
    virtual tjs_error TJS_INTF_METHOD FuncCall(              // vtable[2]
        tjs_uint32 flag,
        const tjs_char* membername,
        tjs_uint32* hint,
        tTJSVariant* result,
        tjs_int numparams,
        tTJSVariant** param,
        iTJSDispatch2* objthis
    ) = 0;
    
    virtual tjs_error TJS_INTF_METHOD FuncCallByNum(         // vtable[3]
        tjs_uint32 flag,
        tjs_int num,
        tTJSVariant* result,
        tjs_int numparams,
        tTJSVariant** param,
        iTJSDispatch2* objthis
    ) = 0;
    
    // 属性访问
    virtual tjs_error TJS_INTF_METHOD PropGet(               // vtable[4]
        tjs_uint32 flag,
        const tjs_char* membername,
        tjs_uint32* hint,
        tTJSVariant* result,
        iTJSDispatch2* objthis
    ) = 0;
    
    virtual tjs_error TJS_INTF_METHOD PropGetByNum(          // vtable[5]
        tjs_uint32 flag,
        tjs_int num,
        tTJSVariant* result,
        iTJSDispatch2* objthis
    ) = 0;
    
    virtual tjs_error TJS_INTF_METHOD PropSet(               // vtable[6]
        tjs_uint32 flag,
        const tjs_char* membername,
        tjs_uint32* hint,
        const tTJSVariant* param,
        iTJSDispatch2* objthis
    ) = 0;
    
    virtual tjs_error TJS_INTF_METHOD PropSetByNum(          // vtable[7]
        tjs_uint32 flag,
        tjs_int num,
        const tTJSVariant* param,
        iTJSDispatch2* objthis
    ) = 0;
    
    // 对象创建
    virtual tjs_error TJS_INTF_METHOD CreateNew(             // vtable[8]
        tjs_uint32 flag,
        const tjs_char* membername,
        tjs_uint32* hint,
        iTJSDispatch2** result,
        tjs_int numparams,
        tTJSVariant** param,
        iTJSDispatch2* objthis
    ) = 0;
    
    // 更多虚函数...
    // IsInstanceOf, Operation, NativeInstanceSupport, ClassInstanceInfo
    // GetCount, DeleteMemberByNum, EnumMembers, Invalidate, IsValid
};
```

### 虚函数表内存布局

```
iTJSDispatch2 对象在内存中的布局:

地址          内容
──────────    ────────────────────────────
obj+0x00      vtable 指针 ──→ ┌─ vtable ─────────────┐
obj+0x04      (成员变量...)    │ [0]  AddRef          │
obj+0x08      ...              │ [1]  Release         │
                               │ [2]  FuncCall        │
                               │ [3]  FuncCallByNum   │
                               │ [4]  PropGet         │
                               │ [5]  PropGetByNum    │
                               │ [6]  PropSet         │
                               │ [7]  PropSetByNum    │
                               │ [8]  CreateNew       │
                               │ [9]  CreateNewByNum  │
                               │ [10] GetCount        │
                               │ [11] ...             │
                               └──────────────────────┘

在 32 位程序中，每个 vtable 条目 4 字节:
  vtable[0] = [vtable_ptr + 0x00] = AddRef 地址
  vtable[1] = [vtable_ptr + 0x04] = Release 地址
  vtable[2] = [vtable_ptr + 0x08] = FuncCall 地址
  ...
```

### 在反汇编中识别 iTJSDispatch2 调用

```nasm
; 调用 obj->FuncCall(flag, name, hint, result, nparams, params, objthis)
mov     ecx, [ebp-10h]        ; ECX = iTJSDispatch2* obj (this)
mov     eax, [ecx]            ; EAX = vtable 指针
push    ecx                   ; objthis
push    esi                   ; params (tTJSVariant**)
push    2                     ; numparams = 2
lea     edx, [ebp-20h]
push    edx                   ; result
push    0                     ; hint = NULL
push    offset aMethodName    ; membername = L"methodName"
push    0                     ; flag = 0
call    dword ptr [eax+08h]   ; vtable[2] = FuncCall
; 注意: +08h / 4 = 索引 2 = FuncCall
```

## 6.1.5 ncbind 注册系统

### ncbind 概述

ncbind 是 KiriKiri2 的插件绑定框架，通过 C++ 宏自动将 C++ 类注册为 TJS 类：

```cpp
// 典型的 ncbind 使用示例
// 将 C++ 类 NativeMyClass 注册为 TJS 中的 "MyClass"

class NativeMyClass {
public:
    NativeMyClass() : value_(0) {}
    
    int GetValue() const { return value_; }
    void SetValue(int v) { value_ = v; }
    int Calculate(int a, int b) { return a + b + value_; }
    
private:
    int value_;
};

// ncbind 注册宏
NCB_REGISTER_CLASS(NativeMyClass) {
    // 注册类名（TJS 中的名称）
    NCB_CONSTRUCTOR(());  // 无参构造函数
    
    // 注册属性
    NCB_PROPERTY_RO(value, GetValue);         // 只读属性
    NCB_PROPERTY(value, GetValue, SetValue);  // 读写属性
    
    // 注册方法
    NCB_METHOD(Calculate);
}
```

注册后，TJS 脚本中可以这样使用：

```javascript
// TJS 脚本
var obj = new MyClass();
obj.value = 42;
var result = obj.Calculate(10, 20);  // result = 72
```

### ncbind 核心宏展开

```cpp
// NCB_REGISTER_CLASS 展开后（简化）:
// 1. 创建一个静态注册器对象
// 2. 在 V2Link 时自动执行注册
// 3. 在 V2Unlink 时自动执行注销

// NCB_REGISTER_CLASS(NativeMyClass) 大致展开为:
static struct NCB_Registerer_NativeMyClass {
    NCB_Registerer_NativeMyClass() {
        // 注册回调
        TVPRegisterAutoRegistCallback([]() {
            // 创建 TJS 类对象
            iTJSDispatch2* cls = TJSCreateNativeClass(L"MyClass");
            
            // 注册构造函数
            TJSNativeClassRegisterNCM(cls, L"MyClass", 
                &NativeMyClass_Constructor, ...);
            
            // 注册方法
            TJSNativeClassRegisterNCM(cls, L"Calculate",
                &NativeMyClass_Calculate_Wrapper, ...);
            
            // 注册到全局
            iTJSDispatch2* global = TVPGetScriptDispatch();
            tTJSVariant val(cls);
            global->PropSet(TJS_MEMBERENSURE, L"MyClass", NULL, &val, global);
            global->Release();
            cls->Release();
        });
    }
} ncb_registerer_NativeMyClass;
```

### NCB_PRE_REGIST_CALLBACK

对于不使用 ncbind 类注册的插件，可以用回调直接操作 TJS：

```cpp
// 在 V2Link 之前执行的注册回调
static void PreRegistCallback() {
    // 直接操作 TJS 全局对象
    iTJSDispatch2* global = TVPGetScriptDispatch();
    if (!global) return;
    
    // 注册一个全局函数
    tTJSVariant func(&MyFunction, NULL);
    global->PropSet(TJS_MEMBERENSURE, L"myGlobalFunc", NULL, &func, global);
    
    global->Release();
}
NCB_PRE_REGIST_CALLBACK(PreRegistCallback);
```

## 6.1.6 项目中的实际插件分析

### extrans 插件（转场效果）

```
文件: cpp/plugins/ 下的转场相关代码
导出: V2Link, V2Unlink
功能: 提供页面转场视觉效果

V2Link 流程:
1. TVPInitImportStub(exporter)  — 获取引擎函数
2. 注册多种转场效果类:
   - CrossFade（交叉淡入淡出）
   - Scroll（滚动）
   - Wave（波浪）
   - Mosaic（马赛克）
   - ...
```

### csvParser 插件

```cpp
// cpp/plugins/csvParser.cpp（项目实际代码）
// 将 CSV 解析功能暴露给 TJS 脚本

// 使用 ncbind 注册
NCB_REGISTER_CLASS(NativeCSVParser) {
    NCB_CONSTRUCTOR(());
    NCB_METHOD(Parse);
    NCB_METHOD(GetCell);
    NCB_PROPERTY_RO(rowCount, GetRowCount);
    NCB_PROPERTY_RO(colCount, GetColCount);
}

// TJS 脚本中的使用:
// var csv = new CSVParser();
// csv.Parse("data.csv");
// var name = csv.GetCell(0, 1);  // 第0行第1列
```

### scriptsEx 插件

```cpp
// cpp/plugins/scriptsEx.cpp
// 扩展 TJS 脚本功能

// 注册全局函数
static void PreRegistCallback() {
    iTJSDispatch2* global = TVPGetScriptDispatch();
    
    // 添加 readFile 全局函数
    // 添加 writeFile 全局函数
    // 添加 getEnvironmentVariable 全局函数
    
    global->Release();
}
NCB_PRE_REGIST_CALLBACK(PreRegistCallback);
```

## 6.1.7 常见错误与解决方案

### 错误 1：V2Link 返回非零值但插件不工作

```
症状: V2Link 被调用，返回 S_OK，但 TJS 中找不到注册的类
原因: TVPInitImportStub 调用失败或 ncbind 注册回调未执行

诊断步骤:
1. 在 V2Link 入口设断点，确认 exporter 参数非 NULL
2. 步入 TVPInitImportStub，检查函数指针是否成功填充
3. 检查 ncbind 注册器的静态构造函数是否执行

解决方案:
- 确保 tp_stub.cpp 被编译链接到插件中
- 确保 ncbind.cpp 也被包含（如果使用 ncbind）
- 检查 DLL 的链接顺序——静态初始化器可能因链接顺序问题未执行
```

### 错误 2：逆向时看到大量虚函数调用，无法辨别

```
症状: 反汇编中到处是 call [eax+XX]，不知道调用的是什么
原因: iTJSDispatch2 是纯虚接口，所有操作都通过虚函数

解决方案:
1. 记住 iTJSDispatch2 vtable 布局:
   [+00] AddRef      [+04] Release
   [+08] FuncCall    [+0C] FuncCallByNum
   [+10] PropGet     [+14] PropGetByNum
   [+18] PropSet     [+1C] PropSetByNum
   [+20] CreateNew   ...

2. 在 IDA/Ghidra 中导入 tp_stub.h 的结构体定义
3. 将对象类型标注为 iTJSDispatch2*
4. 反编译器会自动显示虚函数名称
```

### 错误 3：tp_stub 版本不匹配

```
症状: 插件加载后崩溃（访问违例）
原因: 插件编译使用的 tp_stub.h 版本与 krkr.exe 不匹配
      导致虚函数表索引错位

解决方案:
1. 确认 krkr.exe 的版本（在标题栏或 About 对话框中查看）
2. 使用对应版本的 tp_stub.h
3. KiriKiri2 和 KiriKiriZ 的接口有细微差别，不能混用
4. 项目仓库中的 tp_stub.h 位于: cpp/core/plugin/tp_stub.h
```

## 動手実践

### 练习：追踪插件加载流程

```
目标: 使用 x64dbg 追踪 KiriKiri2 加载 extrans.dll 的完整过程

步骤:
1. 用 x32dbg 打开 krkr.exe
2. 设置断点:
   bp LoadLibraryW
   bp GetProcAddress
3. F9 运行
4. 在 LoadLibraryW 断下时，查看参数:
   [ESP+4] = 文件路径 → 确认是 extrans.dll
5. F8 步过 → EAX = DLL 基址
6. 继续 F9 → 在 GetProcAddress 断下:
   [ESP+4] = DLL 基址
   [ESP+8] → "V2Link"
7. F8 步过 → EAX = V2Link 函数地址
8. 在 V2Link 地址设断点: bp EAX
9. F9 → 断在 V2Link 入口
10. 查看 [ESP+4] = iTVPFunctionExporter* (exporter 对象)
11. 记录 exporter 的 vtable 地址
12. F7 步入 V2Link，观察 TVPInitImportStub 的调用
```

## 対照項目源码

| 概念 | 源码文件 | 关键内容 |
|------|---------|---------|
| V2Link/V2Unlink | `cpp/core/plugin/tp_stub.h` | 导出函数声明和 TVPInitImportStub |
| iTVPFunctionExporter | `cpp/core/plugin/tp_stub.h` | 函数查询接口定义 |
| iTJSDispatch2 | `cpp/core/tjs2/tjsInterfaces.h` | 完整虚函数表定义 |
| ncbind 系统 | `cpp/core/plugin/ncbind.hpp` | NCB_REGISTER_CLASS 等宏定义 |
| ncbind 实现 | `cpp/core/plugin/ncbind.cpp` | 注册回调管理 |
| 插件加载逻辑 | `cpp/core/plugin/PluginImpl.cpp` | LoadLibrary + GetProcAddress 流程 |
| 实际插件示例 | `cpp/plugins/csvParser.cpp` | 使用 ncbind 的完整示例 |
| 实际插件示例 | `cpp/plugins/scriptsEx.cpp` | 使用 NCB_PRE_REGIST_CALLBACK 的示例 |

## 本節小結

1. **V2Link/V2Unlink** 是 KiriKiri2 插件的标准接口，V2Link 接收 exporter 参数用于获取引擎函数
2. **iTVPFunctionExporter** 实现了插件与主程序的解耦——插件通过函数签名字符串动态获取函数指针
3. **tp_stub.h/cpp** 是桥梁层，TVPInitImportStub 填充全局函数指针表，让插件代码可以自然地调用引擎函数
4. **iTJSDispatch2** 的虚函数表布局是逆向分析的核心——所有 TJS 对象操作都通过它的虚函数完成
5. **ncbind** 通过 C++ 宏和模板自动化了类注册过程，是大多数插件的首选绑定方式

## 練習題与答案

### 题目 1：V2Link 参数分析

**问题**：在逆向一个未知的 KiriKiri2 插件时，你在 V2Link 函数中看到以下反汇编代码。解释每一步在做什么。

```nasm
_V2Link@4:
    push    ebp
    mov     ebp, esp
    push    [ebp+8]              ; (A)
    call    _TVPInitImportStub@4 ; (B)
    call    _RegisterMyClass     ; (C)
    xor     eax, eax             ; (D)
    pop     ebp
    ret     4                    ; (E)
```

<details>
<summary>查看答案</summary>

```
(A) push [ebp+8]
    将 V2Link 的第一个参数（iTVPFunctionExporter* exporter）压栈
    这个参数由 krkr.exe 传入，是引擎函数的导出器接口

(B) call _TVPInitImportStub@4
    调用 tp_stub.cpp 中的 TVPInitImportStub
    功能: 通过 exporter 接口查询并缓存所有引擎函数的地址
    @4 表示 stdcall，清理 4 字节参数（即 exporter 指针）

(C) call _RegisterMyClass
    调用插件自定义的注册函数
    功能: 将 C++ 类注册为 TJS 类，或添加全局函数
    这通常涉及 ncbind 宏或手动的 iTJSDispatch2 操作

(D) xor eax, eax
    将 EAX 设为 0
    等价于 return S_OK（S_OK = 0）
    表示插件初始化成功

(E) ret 4
    返回，并清理 4 字节的栈参数
    这确认了 V2Link 是 __stdcall 调用约定
    参数: 1 个指针（4 字节）
```

</details>

### 题目 2：iTJSDispatch2 虚函数识别

**问题**：以下反汇编代码调用了 iTJSDispatch2 的哪个虚函数？推断调用的 TJS 操作是什么。

```nasm
mov     ecx, [ebp-8]           ; iTJSDispatch2* obj
mov     eax, [ecx]             ; vtable
push    ecx                    ; objthis
push    0                      ; hint = NULL
lea     edx, [ebp-20h]
push    edx                    ; param (tTJSVariant*)
push    0                      ; hint = NULL
push    offset aWidth          ; membername = L"width"
push    0                      ; flag = 0
call    dword ptr [eax+18h]    ; vtable[?]
```

<details>
<summary>查看答案</summary>

```
vtable 偏移 = 0x18 = 24
索引 = 24 / 4 = 6

根据 iTJSDispatch2 vtable 布局:
  [0] AddRef       (+00)
  [1] Release      (+04)
  [2] FuncCall     (+08)
  [3] FuncCallByNum(+0C)
  [4] PropGet      (+10)
  [5] PropGetByNum (+14)
  [6] PropSet      (+18) ★
  [7] PropSetByNum (+1C)

vtable[6] = PropSet

参数分析:
  flag = 0
  membername = L"width"
  hint = NULL
  param = [ebp-20h] 指向的 tTJSVariant（要设置的值）
  objthis = ecx（对象自身）

TJS 等价操作:
  obj.width = <param 中的值>;

这是在设置对象的 "width" 属性。
```

</details>

### 题目 3：ncbind 注册分析

**问题**：给定以下 ncbind 注册代码，描述注册后 TJS 脚本中可以执行哪些操作。

```cpp
NCB_REGISTER_CLASS(NativeFileReader) {
    NCB_CONSTRUCTOR((const tjs_char*));
    NCB_METHOD(ReadLine);
    NCB_METHOD(ReadAll);
    NCB_PROPERTY_RO(lineCount, GetLineCount);
    NCB_PROPERTY_RO(eof, IsEOF);
    NCB_METHOD(Close);
}
```

<details>
<summary>查看答案</summary>

注册后，TJS 脚本中可以执行以下操作：

```javascript
// 1. 创建对象（构造函数接受一个字符串参数）
var reader = new FileReader("data.txt");

// 2. 调用 ReadLine 方法
var line = reader.ReadLine();

// 3. 调用 ReadAll 方法
var content = reader.ReadAll();

// 4. 读取 lineCount 属性（只读）
var count = reader.lineCount;
// reader.lineCount = 10;  // 错误！只读属性

// 5. 读取 eof 属性（只读）
while (!reader.eof) {
    var line = reader.ReadLine();
    // 处理每一行
}

// 6. 调用 Close 方法
reader.Close();
```

分析：
- `NCB_CONSTRUCTOR((const tjs_char*))` → 构造函数接受宽字符串参数（文件路径）
- `NCB_METHOD(ReadLine)` → 注册 ReadLine 为可调用方法
- `NCB_METHOD(ReadAll)` → 注册 ReadAll 为可调用方法
- `NCB_PROPERTY_RO(lineCount, GetLineCount)` → 注册只读属性 lineCount，映射到 GetLineCount()
- `NCB_PROPERTY_RO(eof, IsEOF)` → 注册只读属性 eof，映射到 IsEOF()
- `NCB_METHOD(Close)` → 注册 Close 为可调用方法

</details>

## 下一步

下一节我们将进入**逆向分析过程**——选取一个实际的 KiriKiri2 插件 DLL（无源码），使用 IDA/Ghidra 进行静态分析，配合 x64dbg 动态调试，完成从二进制到功能理解的完整逆向流程。
