# V2Link 与 V2Unlink 入口机制

> **所属模块：** M12-插件逆向与实现
> **前置知识：** [M09-插件系统与开发](../../M09-插件系统与开发/)、[P09-逆向工程入门](../../P09-逆向工程入门/)
> **预计阅读时间：** 约 25 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 KiriKiri 原版插件 DLL 的加载流程——引擎如何发现、加载并初始化一个插件
2. 编写符合原版 KiriKiri 插件接口规范的 `V2Link` 和 `V2Unlink` 函数
3. 理解 KrKr2 中为何不使用动态加载而改用静态链接，以及两种方式的本质区别
4. 在 Ghidra 中识别一个未知 DLL 是否为 KiriKiri 插件

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| V2Link | V2Link | KiriKiri 插件的初始化入口函数，引擎加载 DLL 后首先调用它来建立引擎与插件的连接 |
| V2Unlink | V2Unlink | KiriKiri 插件的卸载函数，引擎卸载插件前调用它来释放资源 |
| 导出函数 | Exported Function | DLL 中标记为对外公开的函数，其他程序可以通过名称找到并调用它 |
| LoadLibrary | LoadLibrary | Windows API，用于在运行时将 DLL 文件加载到进程内存空间 |
| GetProcAddress | GetProcAddress | Windows API，用于从已加载的 DLL 中按名称查找函数地址 |
| 静态链接 | Static Linking | 编译时将库代码直接合并进最终二进制文件，运行时不需要额外的 DLL 文件 |
| 函数指针表 | Function Pointer Table | 一组函数指针的集合，引擎通过它向插件提供内部 API 的访问能力 |

## 一、KiriKiri 原版插件加载机制

### 1.1 原版插件的工作方式

在原版 KiriKiri/吉里吉里引擎中，插件以 Windows DLL（Dynamic Link Library，动态链接库）的形式存在。每个插件是一个独立的 `.dll` 文件，放在游戏目录的 `plugin/` 文件夹下。引擎启动时会扫描该目录，发现 DLL 文件后按照以下流程加载：

```
游戏目录
├── data.xp3          ← 游戏数据包
├── krkr.eXe           ← KiriKiri 引擎主程序
└── plugin/
    ├── csvParser.dll   ← CSV 解析插件
    ├── fstat.dll       ← 文件状态插件
    ├── PackinOne.dll   ← 归档压缩插件（无源码）
    └── TextRender.dll  ← 文字渲染插件（无源码）
```

引擎加载每个 DLL 的完整流程如下：

```
┌─────────────────────────────────────────────────┐
│              KiriKiri 引擎主程序                   │
│                                                   │
│  1. 扫描 plugin/ 目录，找到 *.dll 文件              │
│  2. 调用 LoadLibrary("plugin/xxx.dll")             │
│     → Windows 将 DLL 映射到进程内存                  │
│  3. 调用 GetProcAddress(dll, "V2Link")              │
│     → 查找 DLL 中名为 "V2Link" 的导出函数            │
│  4. 调用 V2Link(iTVPFunctionExporter*)              │
│     → 将引擎内部 API 的函数指针表传给插件              │
│  5. 插件通过函数指针表注册自己的类/函数               │
│                                                   │
│  卸载时：                                          │
│  6. 调用 GetProcAddress(dll, "V2Unlink")            │
│  7. 调用 V2Unlink() → 插件释放资源                  │
│  8. 调用 FreeLibrary(dll) → 卸载 DLL                │
└─────────────────────────────────────────────────┘
```

这个流程的关键点在于：**引擎和插件之间通过两个约定好的函数名（V2Link 和 V2Unlink）建立连接**。只要 DLL 导出了这两个函数，引擎就认为它是一个合法的 KiriKiri 插件。

### 1.2 为什么叫 "V2"

"V2" 中的 "V" 代表 "Version"（版本），"2" 代表第二版接口。KiriKiri 引擎在早期版本中使用过 V1 接口（`V1Link`/`V1Unlink`），后来因为接口不够灵活而升级为 V2。V2 接口的核心改进是引入了函数指针表机制（`iTVPFunctionExporter`），使得插件可以访问引擎内部的任意 API，而不是只能使用预定义的少量功能。

实际上，绝大多数现存的 KiriKiri 插件都使用 V2 接口。如果你在逆向分析一个 DLL 时发现它导出了 `V2Link` 函数，基本可以确定它是一个 KiriKiri 插件。

## 二、V2Link 函数签名详解

### 2.1 函数原型

在原版 KiriKiri 引擎的头文件中，V2Link 的函数原型定义如下：

```cpp
// 来源：原版 KiriKiri tp_stub.h（原版有数千行，KrKr2 已简化）
// V2Link 是插件的初始化入口，引擎加载 DLL 后调用此函数
typedef HRESULT (_stdcall * tTVPV2LinkProc)(iTVPFunctionExporter *);
```

逐一拆解这个声明中的每个部分：

| 组成部分 | 含义 |
|----------|------|
| `HRESULT` | 返回值类型，Windows COM 风格的错误码。`S_OK`（值为 0）表示成功，负值表示失败 |
| `_stdcall` | 调用约定（Calling Convention），指定函数参数的入栈顺序和栈清理方式。Windows API 标准使用 `_stdcall`，由被调用者清理栈 |
| `tTVPV2LinkProc` | 这是一个类型别名（typedef），代表"指向 V2Link 函数的指针类型"。引擎内部通过这个类型来存储从 DLL 获取的函数地址 |
| `iTVPFunctionExporter *` | 参数类型——一个指向函数导出器接口的指针。引擎通过这个接口向插件提供内部 API |

### 2.2 iTVPFunctionExporter 接口

`iTVPFunctionExporter` 是 V2Link 机制的核心。它是一个纯虚接口（Abstract Interface），定义了插件从引擎获取内部函数的方法。在 KrKr2 项目中，这个接口定义在 `cpp/core/plugin/PluginImpl.h`（第 28-34 行，位于 `#if 0` 条件编译块内，因为 KrKr2 不使用动态加载）：

```cpp
// 文件：cpp/core/plugin/PluginImpl.h 第 28-34 行
// iTVPFunctionExporter — 引擎向插件暴露内部 API 的桥梁
struct iTVPFunctionExporter {
    // 通过宽字符函数名数组查询函数指针
    // name: 函数名数组（如 {"TVPAddLog", "TVPGetVersion", NULL}）
    // function: 输出参数，接收对应的函数指针
    // count: 要查询的函数数量
    // 返回 true 表示所有函数都找到了
    virtual bool QueryFunctions(
        const tjs_char **name,    // tjs_char 是 KiriKiri 的宽字符类型（wchar_t）
        void **function,          // 输出：函数指针数组
        tjs_uint count            // 要查询的函数个数
    ) = 0;

    // 通过窄字符（char*）函数名查询——提供 ASCII 版本的便利接口
    virtual bool QueryFunctionsByNarrowString(
        const char **name,        // ASCII 函数名数组
        void **function,          // 输出：函数指针数组
        tjs_uint count            // 要查询的函数个数
    ) = 0;
};
```

这个接口的设计体现了一个经典的架构模式——**依赖注入**（Dependency Injection）。插件不直接 `#include` 引擎头文件来调用引擎函数，而是在运行时通过引擎传入的 `iTVPFunctionExporter` 指针来动态查询所需函数。这样做的好处是：

1. **版本解耦**：插件编译时不需要知道引擎的完整 API，只需要知道 `iTVPFunctionExporter` 接口
2. **二进制兼容**：即使引擎内部 API 地址变化，只要函数名不变，插件无需重新编译
3. **按需获取**：插件只查询自己需要的函数，不需要加载全部引擎 API

### 2.3 一个完整的 V2Link 实现示例

下面是一个原版 KiriKiri 插件中典型的 V2Link 实现。这段代码展示了插件如何在初始化时从引擎获取所需函数：

```cpp
// 示例：一个简单的原版 KiriKiri 插件的 V2Link 实现
// 编译后生成 myplugin.dll，放到游戏的 plugin/ 目录即可

#include <windows.h>

// 前向声明引擎接口（原版 tp_stub.h 提供完整定义）
struct iTVPFunctionExporter;
typedef wchar_t tjs_char;
typedef unsigned int tjs_uint;
typedef long HRESULT;
#define S_OK 0
#define E_FAIL ((HRESULT)0x80004005L)

// 引擎函数指针——V2Link 成功后这些变量就指向引擎内部函数
typedef void (*tTVPAddLogProc)(const tjs_char* msg);
static tTVPAddLogProc TVPAddLog = nullptr;  // 引擎的日志输出函数

typedef const tjs_char* (*tTVPGetVersionProc)();
static tTVPGetVersionProc TVPGetVersion = nullptr;  // 获取引擎版本号

// V2Link — 插件初始化入口
// 引擎加载此 DLL 后会调用这个函数
extern "C" __declspec(dllexport) HRESULT _stdcall V2Link(
    iTVPFunctionExporter *exporter  // 引擎传入的函数查询接口
) {
    // 第一步：通过 exporter 查询引擎内部函数
    const char* names[] = { "TVPAddLog", "TVPGetVersion" };
    void* functions[2] = {};

    // QueryFunctionsByNarrowString 按名称查找引擎函数
    // 如果某个函数找不到，整个调用返回 false
    bool ok = exporter->QueryFunctionsByNarrowString(
        names,      // 要查询的函数名
        functions,  // 输出：对应的函数指针
        2           // 查询 2 个函数
    );

    if (!ok) {
        return E_FAIL;  // 查询失败——可能引擎版本不匹配
    }

    // 第二步：保存函数指针到全局变量
    TVPAddLog = (tTVPAddLogProc)functions[0];
    TVPGetVersion = (tTVPGetVersionProc)functions[1];

    // 第三步：使用引擎函数进行初始化
    TVPAddLog(L"MyPlugin: V2Link 成功，插件已初始化");

    return S_OK;  // 返回成功
}

// V2Unlink — 插件卸载时调用
extern "C" __declspec(dllexport) HRESULT _stdcall V2Unlink() {
    // 清理插件资源
    TVPAddLog(L"MyPlugin: V2Unlink，插件正在卸载");
    TVPAddLog = nullptr;
    TVPGetVersion = nullptr;
    return S_OK;
}
```

上面的代码展示了原版 KiriKiri 插件开发的完整模式：通过 `V2Link` 获取引擎函数，通过 `V2Unlink` 清理资源。这是所有 KiriKiri 插件必须遵循的协议。

## 三、V2Unlink 函数详解

### 3.1 函数原型

V2Unlink 比 V2Link 简单得多，它不接收任何参数：

```cpp
// 来源：cpp/core/plugin/PluginImpl.h 第 55 行附近
// V2Unlink 是插件卸载时的清理函数
typedef HRESULT (_stdcall * tTVPV2UnlinkProc)();
```

| 组成部分 | 含义 |
|----------|------|
| `HRESULT` | 返回值，`S_OK` 表示卸载成功 |
| `_stdcall` | 调用约定，与 V2Link 一致 |
| `void`（无参数） | V2Unlink 不需要参数——插件在 V2Link 时已经保存了所需的引擎函数指针 |

### 3.2 V2Unlink 的职责

V2Unlink 被调用时，插件应当完成以下清理工作：

```
┌──────────────────────────────────────────┐
│           V2Unlink 清理流程               │
│                                          │
│  1. 注销已注册的 TJS 类和函数              │
│     → 从引擎的类注册表中移除自己的条目       │
│                                          │
│  2. 释放动态分配的资源                     │
│     → free/delete 在 V2Link 中分配的内存   │
│     → 关闭文件句柄、网络连接等             │
│                                          │
│  3. 清空函数指针                          │
│     → 将保存的引擎函数指针置为 nullptr      │
│     → 防止 DLL 卸载后使用悬垂指针          │
│                                          │
│  4. 返回 S_OK                            │
│     → 告知引擎可以安全调用 FreeLibrary     │
└──────────────────────────────────────────┘
```

### 3.3 V2Unlink 中的常见错误

在逆向分析无源码插件时，经常会发现一些插件的 V2Unlink 实现存在问题。了解这些常见错误有助于理解逆向时遇到的异常行为：

```cpp
// 错误示例 1：V2Unlink 为空实现
// 后果：插件注册的 TJS 类不会被移除，引擎可能在后续访问已卸载的代码
extern "C" __declspec(dllexport) HRESULT _stdcall V2Unlink() {
    return S_OK;  // 什么都不做——这是一个常见的偷懒写法
    // 问题：如果引擎尝试调用此插件注册的 TJS 函数，会访问已释放的内存
}

// 错误示例 2：V2Unlink 中使用已释放的引擎函数
// 后果：如果引擎先卸载了其他依赖插件，某些函数指针可能已失效
extern "C" __declspec(dllexport) HRESULT _stdcall V2Unlink() {
    TVPAddLog(L"正在卸载...");  // 危险：TVPAddLog 可能已经失效
    // 正确做法：先检查指针是否为 nullptr
    if (TVPAddLog) {
        TVPAddLog(L"正在卸载...");
    }
    TVPAddLog = nullptr;
    return S_OK;
}
```

## 四、KrKr2 的静态链接方案

### 4.1 为什么 KrKr2 不使用动态加载

KrKr2 是一个跨平台模拟器项目，目标平台包括 Windows、Linux、macOS 和 Android。原版 KiriKiri 的动态加载方案（`LoadLibrary` + `GetProcAddress`）存在以下跨平台问题：

| 问题 | 说明 |
|------|------|
| Windows 专属 API | `LoadLibrary`、`GetProcAddress` 是 Windows 独有的，Linux 用 `dlopen`/`dlsym`，macOS 类似但有细微差异 |
| DLL 格式不通用 | Windows 的 `.dll`、Linux 的 `.so`、macOS 的 `.dylib` 格式完全不同，无法通用 |
| Android 限制 | Android 上动态加载自定义 `.so` 有安全限制（SELinux、应用沙箱） |
| 调用约定差异 | `_stdcall` 是 Windows 专属概念，其他平台使用不同的 ABI（应用二进制接口） |
| 二进制分发复杂 | 每个平台都需要单独编译和分发插件二进制文件 |

因此，KrKr2 采用了完全不同的方案——**静态链接**：将所有插件的源代码直接编译进主程序。

### 4.2 KrKr2 的插件加载流程

在 KrKr2 中，插件不再是独立的 DLL 文件，而是通过 ncbind 宏系统在编译时注册到一个全局表中。运行时，引擎通过查表来"加载"插件。完整流程如下：

```
编译时（Static Registration）：
┌──────────────────────────────────────────────────────────┐
│  cpp/plugins/getabout.cpp:                                │
│    NCB_MODULE_NAME = TJS_W("getabout.dll")               │
│    NCB_ATTACH_FUNCTION(getAboutString, System, ...)      │
│                                                           │
│  → 编译器执行全局构造函数                                   │
│  → ncbAutoRegister 实例被创建                              │
│  → 实例自动注册到 _internal_plugins["getabout.dll"] 列表   │
└──────────────────────────────────────────────────────────┘

运行时（Plugin Loading）：
┌──────────────────────────────────────────────────────────┐
│  TJS 脚本: Plugins.link("getabout.dll")                   │
│       ↓                                                   │
│  TVPLoadPlugin("getabout.dll")                            │
│       ↓                                                   │
│  TVPLoadInternalPlugin("getabout.dll")                    │
│       ↓                                                   │
│  ncbAutoRegister::LoadModule("getabout.dll")              │
│       ↓                                                   │
│  遍历 _internal_plugins["getabout.dll"]                   │
│       ↓                                                   │
│  对每个注册项调用 Regist()                                 │
│       ↓                                                   │
│  插件的 TJS 类/函数被注册到引擎中                           │
│       ↓                                                   │
│  "getabout.dll" 加入 TVPRegisteredPlugins 集合             │
└──────────────────────────────────────────────────────────┘
```

让我们看 KrKr2 项目中的实际代码。首先是 `TVPLoadPlugin` 函数：

```cpp
// 文件：cpp/core/plugin/PluginImpl.cpp 第 40-65 行
// TVPLoadPlugin — TJS 脚本调用 Plugins.link() 时触发
void TVPLoadPlugin(const ttstr &name) {
    ttstr pluginName = name;

    // 特殊处理：emoteplayer.dll 实际上是 motionplayer.dll 的别名
    // 某些游戏使用旧名称 emoteplayer，需要映射到实际实现
    if (pluginName == TJS_W("emoteplayer.dll")) {
        pluginName = TJS_W("motionplayer.dll");
    }

    // 检查是否已经加载过此插件（避免重复注册）
    if (TVPRegisteredPlugins.find(pluginName) != TVPRegisteredPlugins.end()) {
        return;  // 已加载，直接返回
    }

    // 调用内部加载函数
    TVPLoadInternalPlugin(pluginName);
}
```

然后是 `TVPLoadInternalPlugin`：

```cpp
// 文件：cpp/core/plugin/PluginImpl.cpp 第 67-80 行
// TVPLoadInternalPlugin — 从编译时注册的静态表中加载插件
static void TVPLoadInternalPlugin(const ttstr &name) {
    // ncbAutoRegister::LoadModule 遍历 _internal_plugins[name]
    // 对每个注册项调用 Regist() 方法完成 TJS 类/函数注册
    ncbAutoRegister::LoadModule(name);

    // 将插件名加入已注册集合
    TVPRegisteredPlugins.insert(name);
}
```

### 4.3 动态加载 vs 静态链接对比

| 对比维度 | 原版 KiriKiri（动态加载） | KrKr2（静态链接） |
|----------|--------------------------|-------------------|
| **插件形式** | 独立 .dll 文件 | 源码编译进主程序 |
| **加载方式** | `LoadLibrary` + `GetProcAddress` | 全局构造函数 + 查表 |
| **初始化** | 调用 `V2Link(exporter)` | 调用 `ncbAutoRegister::Regist()` |
| **卸载** | 调用 `V2Unlink()` + `FreeLibrary` | 程序退出时全局析构 |
| **跨平台** | ❌ 仅 Windows | ✅ 全平台通用 |
| **热加载** | ✅ 可运行时替换 DLL | ❌ 需重新编译 |
| **调试难度** | 较高（需附加到 DLL） | 较低（统一调试） |
| **分发** | 需分发 DLL 文件 | 单一可执行文件 |
| **插件发现** | 扫描 plugin/ 目录 | 查询 `_internal_plugins` 表 |

### 4.4 KrKr2 中 V2Link/V2Unlink 的遗迹

虽然 KrKr2 不使用动态加载，但在代码中仍保留了 V2Link/V2Unlink 的类型定义，只是被放在了 `#if 0`（永不编译）的条件块中：

```cpp
// 文件：cpp/core/plugin/PluginImpl.h
// 这些定义被 #if 0 包裹——KrKr2 不使用动态加载
#if 0
typedef HRESULT (_stdcall * tTVPV2LinkProc)(iTVPFunctionExporter *);
typedef HRESULT (_stdcall * tTVPV2UnlinkProc)();
#endif
```

保留这些定义有两个目的：
1. **文档价值**：让后续开发者理解原版 KiriKiri 的插件接口设计
2. **潜在恢复**：如果将来需要支持加载原版 KiriKiri DLL 插件（例如在 Windows 上直接运行原版 DLL），可以重新启用这些代码

## 五、在 Ghidra 中识别 KiriKiri 插件

### 5.1 导出表特征

当你拿到一个未知的 DLL 文件需要判断是否为 KiriKiri 插件时，最快的方法是检查其导出函数表。Ghidra（一款免费的逆向工程工具，由美国国家安全局 NSA 开发并开源）可以直接查看 DLL 的导出表。

在 Ghidra 中打开一个 DLL 后，导航到 **Window → Symbol Table**，然后按 Type 列排序，查找 `Function` 类型的导出符号。如果看到以下导出函数，可以确认是 KiriKiri 插件：

```
导出函数名          意义
─────────────────────────────────────
V2Link              KiriKiri V2 插件初始化入口（必须存在）
V2Unlink            KiriKiri V2 插件卸载函数（必须存在）
```

某些插件还可能导出额外函数：

```
可选导出函数          意义
─────────────────────────────────────
V2LinkDispatched     增强版初始化（支持额外的 Dispatch 参数）
GetModuleStat        返回插件状态信息（版本、作者等）
```

### 5.2 Ghidra 操作步骤

以下是在 Ghidra 中分析一个 KiriKiri 插件 DLL 的完整步骤：

```
步骤 1：创建 Ghidra 项目
─────────────────────────────────────
  File → New Project → Non-Shared Project
  选择保存目录，输入项目名称（如 "KiriKiri_Plugins"）

步骤 2：导入 DLL 文件
─────────────────────────────────────
  File → Import File → 选择目标 DLL（如 PackinOne.dll）
  Ghidra 会自动识别为 PE (Portable Executable) 格式
  Language 选择：x86:LE:32:default（大多数 KiriKiri 插件是 32 位）
  点击 OK 开始分析

步骤 3：运行自动分析
─────────────────────────────────────
  导入后 Ghidra 会询问是否运行自动分析
  点击 Yes → 保持默认分析选项 → 点击 Analyze
  等待分析完成（通常几秒到几分钟）

步骤 4：查看导出函数
─────────────────────────────────────
  Window → Symbol Table
  在 Filter 中输入 "V2Link"
  如果找到 V2Link 和 V2Unlink → 确认为 KiriKiri 插件

步骤 5：分析 V2Link 函数体
─────────────────────────────────────
  双击 V2Link 符号 → 跳转到反汇编视图
  Ghidra 的反编译器会在右侧显示伪 C 代码
  查找 QueryFunctions / QueryFunctionsByNarrowString 调用
  → 这些调用的参数列表就是该插件使用的引擎 API 清单
```

### 5.3 从 V2Link 提取插件 API 依赖

逆向分析 V2Link 函数最重要的目标是找出插件依赖了哪些引擎 API。以下是一个 Ghidra 反编译输出的典型示例：

```cpp
// Ghidra 反编译伪代码示例（对应某个 KiriKiri 插件的 V2Link）
// 变量名已手动重命名以提高可读性
HRESULT __stdcall V2Link(iTVPFunctionExporter *exporter) {
    // 该插件查询了 5 个引擎函数
    const char *names[5];
    void *functions[5];

    names[0] = "TVPAddLog";           // 日志输出
    names[1] = "TVPGetVersion";       // 获取引擎版本
    names[2] = "TVPCreateStream";     // 创建文件流
    names[3] = "TVPRegisterClass";    // 注册 TJS 类
    names[4] = "TVPGetMemoryInfo";    // 获取内存信息

    // 批量查询引擎函数
    bool ok = exporter->QueryFunctionsByNarrowString(names, functions, 5);
    if (!ok) return E_FAIL;

    // 保存函数指针到全局变量
    g_TVPAddLog = (void*)functions[0];
    g_TVPGetVersion = (void*)functions[1];
    g_TVPCreateStream = (void*)functions[2];
    g_TVPRegisterClass = (void*)functions[3];
    g_TVPGetMemoryInfo = (void*)functions[4];

    // 使用 TVPRegisterClass 注册自己的类
    g_TVPRegisterClass(&MyPluginClass);

    return S_OK;
}
```

从这个反编译结果中，我们可以提取出以下关键信息：

1. **API 依赖清单**：TVPAddLog、TVPGetVersion、TVPCreateStream、TVPRegisterClass、TVPGetMemoryInfo
2. **插件功能推断**：使用了 CreateStream，说明涉及文件 I/O；使用了 RegisterClass，说明注册了新的 TJS 类
3. **重新实现参考**：当我们要在 KrKr2 中重新实现此插件时，需要确保这 5 个引擎函数在 KrKr2 中都有对应实现

## 动手实践

### 实践 1：分析 KrKr2 最简插件

打开 KrKr2 项目中最简单的插件——`getabout.cpp`，理解 ncbind 静态注册如何替代 V2Link：

```cpp
// 文件：cpp/plugins/getabout.cpp（完整文件，仅 5 行）
#include "ncbind.hpp"    // ncbind 框架头文件
#include "MsgIntf.h"     // 包含 TVPGetAboutString 函数声明

// 定义此插件对应的"虚拟 DLL 名"
// 当 TJS 脚本调用 Plugins.link("getabout.dll") 时
// 引擎通过这个名称在 _internal_plugins 表中查找
#define NCB_MODULE_NAME TJS_W("getabout.dll")

// 将引擎函数 TVPGetAboutString 绑定到 TJS 的 System 对象上
// 效果：TJS 脚本可以通过 System.getAboutString() 调用
// 等价于原版 V2Link 中手动注册函数的操作
NCB_ATTACH_FUNCTION(getAboutString, System, TVPGetAboutString);
```

**思考题：** 对比上面第 2.3 节中原版 V2Link 的完整实现（约 50 行），KrKr2 的 ncbind 方式只需 5 行就完成了同样的功能。ncbind 内部做了哪些工作来实现这种简化？（提示：全局构造函数 + 编译时注册）

### 实践 2：追踪插件加载链路

按照以下步骤在 KrKr2 源码中追踪一个插件从脚本调用到实际注册的完整链路：

1. 打开 `cpp/core/plugin/PluginImpl.cpp`，找到 `TVPLoadPlugin` 函数（约第 40 行）
2. 找到 `emoteplayer.dll` → `motionplayer.dll` 的别名映射（约第 51-52 行）
3. 跟踪到 `TVPLoadInternalPlugin`，再到 `ncbAutoRegister::LoadModule`
4. 打开 `cpp/core/plugin/ncbind.cpp`，找到 `_internal_plugins` 全局 map 的定义
5. 理解 `LoadModule` 如何遍历注册列表并调用 `Regist()`

### 实践 3：用 dumpbin 检查 DLL 导出表（Windows）

如果你手头有一个 KiriKiri 游戏的插件 DLL，可以不用 Ghidra，直接用 Windows SDK 自带的 `dumpbin` 命令行工具查看导出函数：

```bash
# 在 Visual Studio 开发者命令提示符中运行
# dumpbin 是 MSVC 工具链自带的 PE 文件分析工具

# 查看 DLL 导出函数列表
dumpbin /EXPORTS PackinOne.dll

# 典型输出：
#   Microsoft (R) COFF/PE Dumper Version 14.36.32535.0
#   ...
#   ordinal hint RVA      name
#         1    0 0001A340 V2Link
#         2    1 0001A3B0 V2Unlink
#
# 看到 V2Link 和 V2Unlink → 确认是 KiriKiri 插件
```

在 Linux/macOS 上，如果通过 Wine 运行 KiriKiri 游戏，可以用 `objdump` 或 `pe-parse` 来分析 DLL：

```bash
# Linux：使用 objdump 查看 PE DLL 的导出表
# 需要安装 binutils（大多数发行版自带）
objdump -p PackinOne.dll | grep -A 50 "Export Table"

# macOS：使用 Homebrew 安装 pe-parse
brew install pe-parse
dump-pe PackinOne.dll | grep "Name:"
```

在 Android 上通常不需要直接分析 DLL 文件，因为 KrKr2 的 Android 版本将所有插件静态链接进 `.so` 文件。但如果需要在 Android 设备上分析其他原生库，可以使用 `readelf`：

```bash
# Android NDK 自带 readelf
# 注意：readelf 用于 ELF 格式（.so），不能直接分析 Windows DLL
$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-readelf \
    --dyn-syms libkrkr2.so | grep TVPLoad
```

## 常见错误与排查

### 错误 1：在 KrKr2 代码中搜索 V2Link 实现但找不到

**症状**：在 KrKr2 源码中全局搜索 `V2Link`，只找到 `#if 0` 块中的 typedef，找不到任何实际调用。

**原因**：KrKr2 使用 ncbind 静态注册替代了 V2Link 动态加载。V2Link/V2Unlink 只在原版 KiriKiri 引擎中使用。

**解决**：如果你想理解 KrKr2 的插件加载机制，应该搜索 `ncbAutoRegister`、`NCB_MODULE_NAME`、`TVPLoadInternalPlugin` 这些关键词。

### 错误 2：混淆 V2Link 与 DllMain

**症状**：误认为 V2Link 等同于 Windows DLL 的 `DllMain` 入口点。

**原因**：`DllMain` 是 Windows 加载器在 DLL 映射到进程时**自动调用**的，由操作系统触发；`V2Link` 是 KiriKiri 引擎在 DLL 加载后**主动调用**的，由应用层触发。

**对比**：

```
Windows 加载 DLL 的完整顺序：
1. LoadLibrary("xxx.dll")
   → 操作系统将 DLL 映射到内存
   → 操作系统自动调用 DllMain(DLL_PROCESS_ATTACH)
   → DllMain 返回后，DLL 加载完成

2. GetProcAddress(dll, "V2Link")
   → 获取 V2Link 的函数地址

3. V2Link(exporter)
   → KiriKiri 引擎主动调用，传入函数指针表
   → 插件在此完成与引擎的集成
```

### 错误 3：逆向时未考虑 C++ 名称修饰

**症状**：在 DLL 导出表中看到类似 `?V2Link@@YAGJPAU_iTVPFunctionExporter@@@Z` 的乱码。

**原因**：C++ 编译器会对函数名进行**名称修饰**（Name Mangling），将参数类型信息编码进函数名。只有用 `extern "C"` 声明的函数才能保持原始名称。

**解决**：大多数 KiriKiri 插件使用 `extern "C" __declspec(dllexport)` 声明 V2Link，因此导出表中的名称是干净的。如果遇到名称修饰，可以使用 `undname`（MSVC 工具）或在线工具还原：

```bash
# Windows：使用 undname 工具还原修饰名
undname "?V2Link@@YAGJPAU_iTVPFunctionExporter@@@Z"
# 输出：long __stdcall V2Link(struct _iTVPFunctionExporter *)
```

## 对照项目源码

本节涉及的 KrKr2 项目源码文件及其对应知识点：

| 文件路径 | 行号范围 | 知识点 |
|----------|----------|--------|
| `cpp/core/plugin/PluginImpl.h` | 第 28-34 行 | `iTVPFunctionExporter` 接口定义（`#if 0` 块内） |
| `cpp/core/plugin/PluginImpl.h` | 第 50-55 行 | `tTVPV2LinkProc` / `tTVPV2UnlinkProc` 类型定义（`#if 0` 块内） |
| `cpp/core/plugin/PluginImpl.cpp` | 第 40-65 行 | `TVPLoadPlugin` — 入口函数，含 emoteplayer 别名处理 |
| `cpp/core/plugin/PluginImpl.cpp` | 第 67-80 行 | `TVPLoadInternalPlugin` — 调用 `ncbAutoRegister::LoadModule` |
| `cpp/core/plugin/ncbind.hpp` | 第 2346-2382 行 | `NCB_ATTACH_FUNCTION`、`NCB_MODULE_NAME` 等宏定义 |
| `cpp/plugins/getabout.cpp` | 全文（5 行） | 最简插件示例——用 ncbind 替代 V2Link |
| `cpp/plugins/tp_stub.h` | 全文（14 行） | KrKr2 简化版 tp_stub——对比原版的数千行 |

## 本节小结

- **V2Link** 是原版 KiriKiri 插件的初始化入口，接收 `iTVPFunctionExporter*` 参数，通过函数名查询获取引擎内部 API
- **V2Unlink** 是插件卸载时的清理函数，无参数，负责注销已注册的类/函数并释放资源
- **iTVPFunctionExporter** 是连接引擎和插件的桥梁，采用依赖注入模式实现版本解耦
- **KrKr2 不使用 V2Link/V2Unlink**，改用 ncbind 宏的静态链接方案，解决了跨平台问题
- **ncbAutoRegister** 通过编译时全局构造函数 + 运行时查表，实现了与 V2Link 等价的功能
- **逆向识别**：在 DLL 导出表中找到 `V2Link` 和 `V2Unlink` 即可确认为 KiriKiri 插件
- **V2Link 反编译分析**可以提取插件的引擎 API 依赖清单，这是重新实现插件的第一步

## 练习题与答案

### 题目 1：V2Link 参数分析

原版 KiriKiri 的 V2Link 函数接收一个 `iTVPFunctionExporter*` 参数。请回答：
1. 这个参数是谁创建的？（引擎还是插件？）
2. 为什么不直接让插件 `#include` 引擎的头文件来调用引擎函数？
3. 如果 `QueryFunctionsByNarrowString` 返回 `false`，最可能的原因是什么？

<details>
<summary>查看答案</summary>

1. **由引擎创建**。`iTVPFunctionExporter` 对象由 KiriKiri 引擎在启动时初始化，它内部维护了一个从函数名到函数地址的映射表。引擎在调用 `V2Link` 时将这个对象的指针作为参数传入。

2. **二进制兼容性**。如果插件直接 `#include` 引擎头文件，那么引擎每次升级（哪怕只是改了一个函数的参数类型），所有插件都必须重新编译。使用 `iTVPFunctionExporter` 的间接调用方式，只要函数名和签名不变，插件无需重新编译就能适配新版本引擎。这是一种经典的 **ABI 稳定性**设计。

3. **引擎版本不匹配**。`QueryFunctionsByNarrowString` 返回 `false` 意味着引擎中找不到插件请求的某个函数。最常见原因是插件使用了新版引擎才有的 API，但运行在旧版引擎上。另一种可能是函数名拼写错误。

</details>

### 题目 2：KrKr2 插件加载链路追踪

请按照正确顺序排列以下 KrKr2 插件加载过程中的步骤（从 TJS 脚本触发到插件函数可用）：

A. `ncbAutoRegister::Regist()` 被调用
B. `TVPRegisteredPlugins.insert(name)` 记录已加载
C. TJS 脚本执行 `Plugins.link("csvParser.dll")`
D. `ncbAutoRegister::LoadModule("csvParser.dll")` 查询 `_internal_plugins`
E. `TVPLoadPlugin("csvParser.dll")` 被触发
F. 编译时 `NCB_MODULE_NAME` 宏注册到 `_internal_plugins` 表

<details>
<summary>查看答案</summary>

正确顺序：**F → C → E → D → A → B**

详细解释：
1. **F（编译时）**：`csvParser.cpp` 中的 `#define NCB_MODULE_NAME TJS_W("csvParser.dll")` 和 `NCB_PRE_REGIST_CALLBACK` 宏在编译时生成全局 `ncbAutoRegister` 对象，其构造函数在程序启动时将自己注册到 `_internal_plugins["csvParser.dll"]` 列表中。

2. **C（运行时）**：TJS 脚本调用 `Plugins.link("csvParser.dll")`，这触发引擎的插件加载逻辑。

3. **E**：`TVPLoadPlugin("csvParser.dll")` 被调用，它首先检查 `TVPRegisteredPlugins` 中是否已有此插件（避免重复加载）。

4. **D**：`TVPLoadInternalPlugin` 调用 `ncbAutoRegister::LoadModule("csvParser.dll")`，后者在 `_internal_plugins` map 中查找键为 `"csvParser.dll"` 的注册列表。

5. **A**：对注册列表中的每个 `ncbAutoRegister` 实例调用 `Regist()` 方法，将 TJS 类/函数注册到引擎的脚本环境中。

6. **B**：`TVPRegisteredPlugins.insert("csvParser.dll")` 将插件标记为已加载，后续重复调用 `Plugins.link` 时会直接跳过。

</details>

### 题目 3：编写一个模拟 V2Link 的程序

请编写一个完整可编译的 C++ 程序，模拟 KiriKiri 引擎的 V2Link 调用机制。程序应包含：
- 一个模拟的 `iTVPFunctionExporter` 实现
- 一个模拟的 V2Link 函数
- main 函数中调用 V2Link 并验证函数注册成功

<details>
<summary>查看答案</summary>

```cpp
// 文件：v2link_simulation.cpp
// 编译：g++ -std=c++17 -o v2link_sim v2link_simulation.cpp
// 或：cl /std:c++17 /EHsc v2link_simulation.cpp

#include <iostream>
#include <string>
#include <unordered_map>
#include <cstring>

// ===== 模拟引擎侧 =====

// 模拟的引擎内部函数
void engine_AddLog(const char* msg) {
    std::cout << "[引擎日志] " << msg << std::endl;
}

const char* engine_GetVersion() {
    return "KiriKiri 2.32.2.427";
}

int engine_Add(int a, int b) {
    return a + b;
}

// 模拟 iTVPFunctionExporter
// 引擎维护一个从函数名到函数指针的映射表
class SimulatedExporter {
public:
    // 注册引擎函数到映射表
    void RegisterFunction(const char* name, void* func) {
        function_map_[name] = func;
    }

    // 模拟 QueryFunctionsByNarrowString
    // 按名称批量查询函数指针
    bool QueryFunctionsByNarrowString(
        const char** names,     // 函数名数组
        void** functions,       // 输出：函数指针数组
        unsigned int count      // 查询数量
    ) {
        for (unsigned int i = 0; i < count; ++i) {
            auto it = function_map_.find(names[i]);
            if (it == function_map_.end()) {
                std::cerr << "[错误] 找不到函数: " << names[i] << std::endl;
                return false;  // 有函数找不到，返回失败
            }
            functions[i] = it->second;
        }
        return true;  // 所有函数都找到了
    }

private:
    std::unordered_map<std::string, void*> function_map_;
};

// ===== 模拟插件侧 =====

// 插件保存的引擎函数指针（全局变量）
typedef void (*AddLogFunc)(const char*);
typedef const char* (*GetVersionFunc)();
typedef int (*AddFunc)(int, int);

static AddLogFunc g_AddLog = nullptr;
static GetVersionFunc g_GetVersion = nullptr;
static AddFunc g_Add = nullptr;

// 模拟 V2Link 实现
long V2Link(SimulatedExporter* exporter) {
    const char* names[] = { "AddLog", "GetVersion", "Add" };
    void* functions[3] = {};

    bool ok = exporter->QueryFunctionsByNarrowString(names, functions, 3);
    if (!ok) {
        return -1;  // 模拟 E_FAIL
    }

    // 保存函数指针
    g_AddLog = (AddLogFunc)functions[0];
    g_GetVersion = (GetVersionFunc)functions[1];
    g_Add = (AddFunc)functions[2];

    // 使用引擎函数
    g_AddLog("V2Link 成功：插件已完成初始化");
    return 0;  // 模拟 S_OK
}

// 模拟 V2Unlink 实现
long V2Unlink() {
    if (g_AddLog) {
        g_AddLog("V2Unlink：插件正在卸载");
    }
    g_AddLog = nullptr;
    g_GetVersion = nullptr;
    g_Add = nullptr;
    return 0;
}

// ===== 模拟引擎主程序 =====

int main() {
    std::cout << "=== KiriKiri V2Link 机制模拟 ===" << std::endl;

    // 第 1 步：引擎初始化 FunctionExporter
    SimulatedExporter exporter;
    exporter.RegisterFunction("AddLog", (void*)engine_AddLog);
    exporter.RegisterFunction("GetVersion", (void*)engine_GetVersion);
    exporter.RegisterFunction("Add", (void*)engine_Add);
    std::cout << "[引擎] FunctionExporter 已初始化，注册了 3 个函数" << std::endl;

    // 第 2 步：引擎调用插件的 V2Link
    std::cout << "[引擎] 正在调用 V2Link..." << std::endl;
    long result = V2Link(&exporter);
    if (result != 0) {
        std::cerr << "[引擎] V2Link 失败！" << std::endl;
        return 1;
    }
    std::cout << "[引擎] V2Link 成功" << std::endl;

    // 第 3 步：验证插件获取的函数是否可用
    std::cout << "\n[测试] 通过插件获取的函数指针调用引擎函数：" << std::endl;
    std::cout << "[测试] 引擎版本: " << g_GetVersion() << std::endl;
    std::cout << "[测试] 3 + 7 = " << g_Add(3, 7) << std::endl;

    // 第 4 步：引擎调用 V2Unlink 卸载插件
    std::cout << "\n[引擎] 正在调用 V2Unlink..." << std::endl;
    V2Unlink();
    std::cout << "[引擎] 插件已卸载" << std::endl;

    // 验证函数指针已清空
    std::cout << "[验证] g_AddLog = " << (g_AddLog ? "非空(错误)" : "nullptr(正确)") << std::endl;

    return 0;
}
```

**预期输出：**

```
=== KiriKiri V2Link 机制模拟 ===
[引擎] FunctionExporter 已初始化，注册了 3 个函数
[引擎] 正在调用 V2Link...
[引擎日志] V2Link 成功：插件已完成初始化
[引擎] V2Link 成功

[测试] 通过插件获取的函数指针调用引擎函数：
[测试] 引擎版本: KiriKiri 2.32.2.427
[测试] 3 + 7 = 10

[引擎] 正在调用 V2Unlink...
[引擎日志] V2Unlink：插件正在卸载
[引擎] 插件已卸载
[验证] g_AddLog = nullptr(正确)
```

</details>

## 下一步

下一节 [02-iTJSDispatch2 接口与 tp_stub](./02-iTJSDispatch2接口与tp_stub.md) 将深入讲解 KiriKiri 插件与 TJS 脚本引擎交互的核心接口 `iTJSDispatch2`，以及原版 `tp_stub.h` 如何将引擎内部 API 暴露给插件开发者。

