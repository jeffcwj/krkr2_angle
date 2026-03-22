# PackinOne 功能分析与导出表

> **所属模块：** M12-插件逆向与实现
> **前置知识：** [IDA Pro 静态分析基础](../../M12-插件逆向与实现/01-逆向工程环境搭建/01-IDA-Pro-安装与配置.md)、[KiriKiri 插件接口](../../M09-插件系统与开发/01-插件系统架构/01-插件加载机制.md)
> **预计阅读时间：** 60 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 PackinOne.dll 的真实功能——一个集成了 LZ4 压缩、CxImage 图片处理、文件系统操作、进程管理、图层扩展的综合性 KiriKiri 插件
2. 解读 IDA Pro 分析报告，从导出表、导入表、字符串三个角度推断 DLL 功能
3. 追踪 V2Link/V2Unlink 函数流程，理解 KiriKiri 插件初始化的完整机制
4. 识别 ncbind 自动注册模式（`ncbNativeClassAutoRegister`/`ncbNativeFunctionAutoRegister`）在二进制中的特征
5. 区分一个"多合一"插件与单功能插件的结构差异，并据此规划逆向优先级

## 术语预览

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 导出表 | Export Table | DLL 对外公开的函数列表，宿主进程通过它找到插件入口 |
| 导入表 | Import Table | DLL 自己依赖的外部函数列表，揭示插件调用了哪些系统 API |
| V2Link | — | KiriKiri 插件标准入口：插件加载时被调用，完成 TJS 类注册 |
| ncbind | — | KiriKiri 的"native class binding"——通过 C++ 模板自动将 native 类暴露给 TJS |
| iTJSDispatch2 | — | TJS 虚机的核心调度接口，所有 TJS 对象通过它互相调用 |
| tTJSVariant | — | TJS 的通用值类型，可以持有 int/real/string/object/octet |
| LZ4 | — | 一种极高速的无损压缩算法，以牺牲少量压缩率换取最快速度 |
| CxImage | — | 开源 C++ 图像处理库，支持 PNG/JPEG/BMP/GIF 等多种格式 |
| SHA-256 | — | 一种密码学哈希算法，输出 256 位摘要，常用于数据校验 |

---

## 1. PackinOne.dll 概览

### 1.1 基本信息

PackinOne.dll 是一款用于"星光咖啡馆与死神之蝶"等 KiriKiri 游戏的**综合功能扩展插件**。与名称暗示的"打包/单一化"不同，这个 DLL 实际上是一个**多合一工具集**，将多个独立功能打包进一个二进制文件，以减少游戏发行包中 DLL 数量。

**文件基本信息：**

| 属性 | 值 |
|------|-----|
| 文件大小 | 648 KB |
| 函数数量 | 1999 个 |
| 目标架构 | x86（32 位），MSVC 编译 |
| 导出函数 | 4 个（V2Link、V2Unlink 及各自的 stdcall 装饰名） |
| 第三方库 | LZ4（压缩）、CxImage 7.0.2（图像处理） |

**真实功能模块（由 IDA 分析确认）：**

```
PackinOne.dll
├── 压缩/解压    → LZ4 压缩流（LZ4StreamBase、LZ4CompressStream、LZ4DecompressStream）
├── 图像处理    → CxImage 7.0.2（PNG、JPEG 读写，含 iPhone CgBI 格式）
├── 文件系统    → StoragesFstat（文件属性查询）、TemporaryFiles（临时文件管理）
├── 进程管理    → Process 类（CreateProcess、管道通信、退出码获取）
├── 图层扩展    → layerExImage、layerExRaster、FontEx
├── 脚本扩展    → ScriptsAdd、ArrayAdd、DictAdd、SaveStruct
├── 系统工具    → System 命名空间扩展（DPI 感知、OS 版本、环境变量等）
├── 存储扩展    → Storages.setCurrentDirectory、Plugins.link/unlink
└── 图层特效    → Layer.shrinkCopy、Layer.fillAlpha、Layer.copyAlphaToProvince 等
```

### 1.2 为什么叫 PackinOne？

"Pack in one"的命名含义：**把多个本来应该分开的插件打包进一个 DLL**。这是 KiriKiri 游戏开发中常见的实践——游戏制作者为了方便分发，把常用的实用功能合并进一个"大杂烩"插件，避免玩家游戏目录里堆积几十个 DLL。

---

## 2. 导出表分析

### 2.1 真实导出表

运行 IDA Pro 提取脚本后得到的完整导出表：

```
[1] 0x10001060  V2Link          ← KiriKiri 标准插件入口
[2] 0x10001090  V2Unlink        ← KiriKiri 标准插件卸载
[3] 0x10001060  _V2Link@4       ← V2Link 的 stdcall 装饰名（与 [1] 同地址）
[4] 0x10001090  _V2Unlink@0     ← V2Unlink 的 stdcall 装饰名（与 [2] 同地址）
[268798463] 0x100589FF DllEntryPoint  ← DLL CRT 入口点（ordinal 极大，通常不直接调用）
```

**只有 2 个真正不同的函数地址**。`V2Link` 与 `_V2Link@4` 是同一函数的两种导出名称——前者是无修饰名（供脚本/工具使用），后者是 stdcall 修饰名（`@4` 表示参数为 4 字节，即 1 个 DWORD）。

> **关键推断**：PackinOne 不导出任何压缩/解压的 C 接口函数（如 `compress`、`decompress`）。外部代码无法通过 `GetProcAddress` 直接调用其压缩功能。所有功能都通过 TJS 脚本层访问。

### 2.2 V2Link 反汇编分析

```asm
; V2Link(iTJSDispatch2* tjs) at 0x10001060
0x10001060: mov     eax, [esp+arg_0]  ; 取第一个参数：iTJSDispatch2*
0x10001064: push    eax               ; 压栈传给 sub_1000D7A0
0x10001065: call    sub_1000D7A0      ; ← 设置全局 TJS 函数指针表（ncbind 延迟绑定）
0x1000106A: add     esp, 4
0x1000106D: call    sub_10001000      ; ← 注册所有 TJS 类/函数（触发所有 AutoRegister）
0x10001072: mov     ecx, dword_1009403C  ; 读取当前插件版本号
0x10001078: mov     dword_10094000, ecx  ; 保存到"已加载版本"槽
0x1000107E: xor     eax, eax          ; 返回值 = 0（S_OK）
0x10001080: retn    4                 ; stdcall：清理 4 字节参数
```

**流程说明：**

1. `sub_1000D7A0`：**初始化 ncbind 函数指针表**。ncbind 系统需要从主程序（krkr2.exe）解析 TJS API 函数地址，如 `tTJSVariant::tTJSVariant()`、`iTJSDispatch2::PropSet()` 等。这个函数通过传入的 `iTJSDispatch2*` 对象，用字符串匹配的方式获取函数指针并缓存到全局变量表中（`dword_10094048`、`dword_1009407C` 等）。

2. `sub_10001000`：**触发所有静态注册器**。MSVC 在 `.CRT$XCU` 段存放全局构造器列表，每个 `ncbNativeClassAutoRegister<T>` 和 `ncbNativeFunctionAutoRegister<F>` 都向这个列表注册了自己。`sub_10001000` 遍历这个列表，依次调用各注册器，将 C++ 类/函数绑定到 TJS 全局命名空间。

3. 版本号记录：`dword_1009403C` 是链接时写死的版本常量，`dword_10094000` 是运行时保存的"已安装版本"。V2Unlink 会通过比较二者来验证卸载合法性。

### 2.3 V2Unlink 反汇编分析

```asm
; V2Unlink() at 0x10001090
0x10001090: mov     eax, dword_1009403C  ; 读取插件版本常量
0x10001095: cmp     eax, dword_10094000  ; 与安装时保存的版本比较
0x1000109B: jle     short loc_100010A3   ; 若版本 <= 保存值则跳过错误
0x1000109D: mov     eax, 80004005h       ; E_FAIL：版本不匹配，拒绝卸载
0x100010A2: retn
0x100010A3: call    sub_10001030         ; ← 注销所有 TJS 类/函数
0x100010A8: call    nullsub_1            ; 空操作（可能是 CRT 清理桩）
0x100010AD: xor     eax, eax             ; 返回 S_OK
0x100010AF: retn
```

> **注意** `jle`（jump if less or equal）而不是 `jne`（jump if not equal）。这意味着允许"低版本插件卸载高版本安装"的情况，但不允许"高版本插件卸载低版本安装"。这是 KiriKiri 插件热更新场景的保护逻辑。

---

## 3. 导入表分析

导入表是了解 DLL"能做什么"的第二把钥匙。PackinOne 导入了以下模块：

### 3.1 KERNEL32（文件/进程/内存核心 API）

关键导入函数（按功能分组）：

```
文件操作：
  CreateFileW、CreateFileA、ReadFile、WriteFile、CloseHandle
  GetFileSizeEx、GetFileTime、SetFileTime、SetFilePointer、SetFilePointerEx
  GetFileAttributesW、SetFileAttributesW、DeleteFileW、MoveFileW
  FindFirstFileW、FindNextFileW、FindClose
  CopyFileW、SetEndOfFile

进程/线程：
  CreateProcessW、TerminateProcess、GetExitCodeProcess
  CreateThread、ResumeThread、ExitThread
  WaitForSingleObject

管道（IPC 通信）：
  CreatePipe、ReadFile（管道读）、WriteFile（管道写）
  PeekNamedPipe

内存管理：
  HeapAlloc、HeapFree、HeapReAlloc、HeapCreate、HeapDestroy
  VirtualAlloc、VirtualFree、GlobalAlloc、GlobalFree、GlobalLock、GlobalUnlock

环境/路径：
  GetEnvironmentVariableW、SetEnvironmentVariableW、ExpandEnvironmentStringsW
  GetCurrentDirectoryW、SetCurrentDirectoryW、SearchPathW
```

**推断**：KERNEL32 导入表强烈暗示 `Process` 类是真实实现的，包含管道通信（CreatePipe/PeekNamedPipe/ReadFile/WriteFile 组合用于进程间通信）。

### 3.2 USER32（窗口/UI API）

```
SendMessageW、PostMessageW、MessageBoxW
RegisterClassExW、CreateWindowExW、DestroyWindow、UnregisterClassW
SetWindowPos、SetWindowLongW、GetWindowLongW、DefWindowProcW
GetWindowRect、GetSystemMetrics、GetParent
MonitorFromWindow、GetMonitorInfoW
```

**推断**：UI 相关 API 暗示有某些需要弹出窗口或对话框的功能（如 `System.confirm()` 消息框、文件选择对话框）。

### 3.3 COMDLG32（通用对话框）

```
GetSaveFileNameW、GetOpenFileNameW、CommDlgExtendedError
```

**推断**：存在文件选择对话框功能（对应 TJS 中的 `System.openFileNameDialog()` 或类似 API）。

### 3.4 GDI32（图形设备）

```
AddFontMemResourceEx、RemoveFontMemResourceEx
AddFontResourceExW、RemoveFontResourceExW
```

**推断**：`FontEx` 类通过 GDI32 实现从内存加载字体资源——这对视觉小说游戏至关重要，允许从 XP3 归档中动态加载字体而无需安装。

### 3.5 ADVAPI32（注册表/安全）

```
RegCreateKeyExW、RegSetValueExW、RegCloseKey
InitializeSecurityDescriptor、SetSecurityDescriptorDacl
```

**推断**：`System.writeRegValue()` 功能通过 ADVAPI32 实现，用于在注册表中写入游戏配置（如语言设置、分辨率等）。

### 3.6 SHELL32（Shell 集成）

```
SHGetFileInfoW、SHGetMalloc、SHGetPathFromIDListW
SHBrowseForFolderW、ShellExecuteExW、SHGetDesktopFolder
```

**推断**：`SHBrowseForFolderW` 是 Windows 的文件夹选择对话框。`ShellExecuteExW` 用于打开外部程序/URL（游戏内的"打开网页"功能）。

### 3.7 ole32（COM 对象）

```
CreateStreamOnHGlobal、GetHGlobalFromStream
CoTaskMemAlloc、CoTaskMemFree
```

**推断**：CxImage 在处理图像时使用 COM IStream 接口，这是标准的图像读写抽象层。

---

## 4. 字符串分析：功能的"路标"

IDA 提取的字符串是最直接反映插件功能的证据。以下是关键字符串及其解读：

### 4.1 第三方库版权字符串（已确认集成的库）

```
"LZ4 Library\nCopyright (c) 2011-2016, Yann Collet"
→ 集成了 LZ4 压缩库（v2011-2016，对应 r130 前后版本）

"----- CxImage Copyright START -----\nCxImage version 7.0.2 07/Feb/2011"
→ 集成了 CxImage 7.0.2（2011年2月7日版本）
```

这两个字符串是**铁证**：不可能是偶然包含的，必然对应真实的库集成。

### 4.2 TJS 类/方法注册字符串

```
"fstat"          → 注册为 TJS 全局函数或类名
"getTime"        → 文件时间获取方法
"setTime"        → 文件时间设置方法
"clearStorageCaches"  → 清除存储缓存
"window"         → 访问窗口对象
"owner"          → 访问 owner 属性
```

### 4.3 错误/状态字符串

```
"Out of memory"         → 内存分配失败
"Read error"            → 读取失败
"Unsupported color count : "  → CxImage 不支持的颜色深度
"unknown compress method"     → 未知压缩方法（LZ4 流解压时）
```

### 4.4 RTTI（运行时类型信息）字符串——最有价值的线索

MSVC 生成的 RTTI 类型名称字符串直接告诉我们代码中存在哪些 C++ 类：

```
.?AVLZ4StreamBase@@         → LZ4StreamBase 类（基类）
.?AVLZ4DecompressStream@@   → LZ4DecompressStream 类（解压流）
.?AVLZ4CompressStream@@     → LZ4CompressStream 类（压缩流）
.?AUArchiveStreamBuf@@      → ArchiveStreamBuf 类（归档流缓冲）
.?AUBufferedStreamCryptFilter@@  → BufferedStreamCryptFilter 类（加密过滤器）
.?AUStreamCryptReaderWriter@@    → StreamCryptReaderWriter 类
.?AURangeLimitReader@@           → RangeLimitReader 类（范围限制读取器）
.?AVIFile@@                      → IFile 接口
.?AVIFileStr@@                   → IFileStr（基于字符串路径的文件接口）
.?AVIFileStorage@@               → IFileStorage（基于存储的文件接口）
.?AVIStreamReader@@              → IStreamReader（流读取接口）
.?AVIStreamWriter@@              → IStreamWriter（流写入接口）
.?AVStorageReader@@              → StorageReader（从 TVP 存储读取）
.?AVStorageWriter@@              → StorageWriter（向 TVP 存储写入）
.?AUPNGChunkStreamReader@@       → PNG chunk 流读取器
.?AUPNGChunkStreamWriter@PNGChunkWriter@@  → PNG chunk 流写入器
.?AUJPGChunkStreamWriter@JPGChunkWriter@@ → JPEG chunk 流写入器
.?AUJPGChunkStreamReader@@       → JPEG chunk 流读取器
.?AUImageChunkWriter@@           → 图像块写入器
.?AVStringReader@@               → 字符串读取器
.?AVStringWriter@@               → 字符串写入器
```

### 4.5 ncbind 注册器字符串（直接告诉我们 TJS 类名）

这类字符串格式为 `.?AU?$ncbNativeClassAutoRegister@V{类名}@@@@`：

```
ncbNativeClassAutoRegister<StoragesFstat>    → "StoragesFstat" 类注册到 TJS
ncbNativeClassAutoRegister<TemporaryFiles>   → "TemporaryFiles" 类注册到 TJS
ncbNativeClassAutoRegister<Process>          → "Process" 类注册到 TJS

ncbAttachTJS2ClassAutoRegister<ArrayAdd>          → 附加到 TJS Array 类
ncbAttachTJS2ClassAutoRegister<DictAdd>           → 附加到 TJS Dictionary 类
ncbAttachTJS2ClassAutoRegister<ScriptsAdd>        → 附加到 TJS Scripts 类
ncbAttachTJS2ClassAutoRegister<ScriptsAddForSaveStruct>  → SaveStruct 功能
ncbAttachTJS2ClassAutoRegister<FontEx>            → 附加到 TJS Font 类
ncbAttachTJS2ClassAutoRegister<layerExImage>      → 附加到 TJS Layer 类
ncbAttachTJS2ClassAutoRegister<layerExRaster>     → 附加到 TJS Layer 类（栅格操作）
```

ncbind 中 `Register` 和 `Attach` 的区别：
- `ncbNativeClassAutoRegister`：在 TJS 全局命名空间创建一个**新类**（如 `new Process()`）
- `ncbAttachTJS2ClassAutoRegister`：**附加方法到现有 TJS 类**（如给 `Array` 类加新方法）

### 4.6 ncbind 函数注册器字符串（TJS 全局函数）

格式为 `.?AU?$ncbNativeFunctionAutoRegisterTempl@UncbFunctionTag_{函数名}@@@@`：

```
System_writeRegValue      → System.writeRegValue() — 写注册表
System_readEnvValue       → System.readEnvValue() — 读环境变量
System_writeEnvValue      → System.writeEnvValue() — 写环境变量
System_expandEnvString    → System.expandEnvString() — 展开环境变量字符串
System_urlencode          → System.urlencode() — URL 编码
System_urldecode          → System.urldecode() — URL 解码
System_getAboutString     → System.getAboutString() — 获取关于信息
System_confirm            → System.confirm() — 确认对话框
System_waitForAppLock     → System.waitForAppLock() — 等待应用锁
System_setDpiAwareness    → System.setDpiAwareness() — 设置 DPI 感知模式
System_getOSVersion       → System.getOSVersion() — 获取 OS 版本
System_getKnownFolderPath → System.getKnownFolderPath() — 获取已知文件夹路径
System_commandExecute     → System.commandExecute() — 执行命令
Layer_shrinkCopy          → Layer.shrinkCopy() — 缩放复制
Layer_shrinkCopyFast      → Layer.shrinkCopyFast() — 快速缩放复制
Layer_copyRightBlueToLeftAlpha  → Layer.copyRightBlueToLeftAlpha()
Layer_copyBottomBlueToTopAlpha  → Layer.copyBottomBlueToTopAlpha()
Layer_fillAlpha           → Layer.fillAlpha() — 填充 Alpha 通道
Layer_copyAlphaToProvince → Layer.copyAlphaToProvince() — 复制 Alpha 到省份
Layer_clipAlphaRect       → Layer.clipAlphaRect() — 裁剪 Alpha 矩形
Layer_fillByProvince      → Layer.fillByProvince() — 按省份填充
Layer_fillToProvince      → Layer.fillToProvince() — 填充到省份
Plugins_link              → Plugins.link() — 动态加载插件
Plugins_unlink            → Plugins.unlink() — 卸载插件
Storages_setCurrentDirectory  → Storages.setCurrentDirectory() — 设置当前目录
Scripts_rehash            → Scripts.rehash() — 重新哈希脚本
```

---

## 5. TJS 类详细分析

### 5.1 StoragesFstat 类

由 IDA 中 ncbind 方法注册器的模板参数可以完整还原该类的 TJS 接口：

```tjs2
// StoragesFstat — 文件属性查询类
class StoragesFstat {
    // 构造函数（无参）
    function StoragesFstat();

    // 方法（从 ncbNativeClassMethod 类型中提取）
    function getFileSize(path);          // 返回 uint64 文件大小
    function isExistent(path);           // 检查文件是否存在 → bool
    function isDirectory(path);          // 检查是否是目录 → bool
    function isReadOnly(path, flag);     // 检查/设置只读属性 → bool
    function rename(from, to);           // 重命名文件
    function remove(path);              // 删除文件
    function getCreationTime(path);     // 获取创建时间 → tTJSVariant
    function setCreationTime(path, t);  // 设置创建时间
    function getLastAccessTime(path);   // 获取访问时间 → uint
    function copy(from, to, overwrite); // 复制文件 → bool

    // 属性
    property currentDirectory;          // getter/setter
}
```

从 ncbNativeClassMethod 的函数签名模板参数中还原（IDA 字符串中包含完整 C++ 类型信息）：

```
P6A_KVtTJSString@@@Z  → uint64 (tTJSString)  — 文件大小返回 uint64
P6A_NVtTJSString@@@Z  → bool (tTJSString)    — 路径检查返回 bool
P6AXVtTJSString@@0@Z  → void (tTJSString, tTJSString)  — 接收两个字符串参数
```

### 5.2 Process 类

```tjs2
// Process — 进程创建与管理
class Process {
    // 构造函数
    function Process();

    // 方法
    function run(cmdline);          // 运行进程（不等待）
    function wait(timeout);         // 等待进程结束 → bool（是否超时）

    // 属性
    property exitCode;              // int，进程退出码
}
```

从 KERNEL32 导入的 `CreateProcessW`、`WaitForSingleObject`、`GetExitCodeProcess`、`CreatePipe`、`PeekNamedPipe` 可以确认：Process 类支持带管道的子进程通信，可以读取子进程的标准输出。

### 5.3 TemporaryFiles 类

```tjs2
// TemporaryFiles — 临时文件管理
class TemporaryFiles {
    // 实例方法
    function add(path);             // 添加要追踪的临时文件路径
    function remove(path);          // 从追踪列表移除
    function clear();               // 清理所有追踪的临时文件

    // 析构时自动删除注册的文件
}
```

### 5.4 LayerExImage（Layer 扩展）

这是最复杂的一部分，附加到现有的 TJS `Layer` 类：

```tjs2
// 这些方法被注入到所有 Layer 实例中
// Layer 扩展方法（来自 layerExImage）
layer.loadCxImage(storage, left, top);     // 用 CxImage 加载图片
layer.saveCxImage(storage, format);        // 用 CxImage 保存图片（PNG/JPEG）
layer.resize(width, height, quality);      // 缩放图层
layer.flip(horizontal);                    // 翻转

// Layer 扩展方法（来自 layerExRaster）— 高级栅格操作
layer.raster(src, dx, dy, sx, sy, w, h, type);
```

---

## 6. 完整功能架构图

```
PackinOne.dll
│
├── KiriKiri 插件接口
│   ├── V2Link(iTJSDispatch2*)      ← 插件加载入口
│   └── V2Unlink()                  ← 插件卸载入口
│
├── 压缩子系统
│   ├── LZ4StreamBase               ← LZ4 流抽象基类
│   ├── LZ4CompressStream           ← LZ4 压缩流
│   ├── LZ4DecompressStream         ← LZ4 解压流
│   ├── ArchiveStreamBuf            ← 归档流缓冲
│   └── BufferedStreamCryptFilter   ← （可选）流加密过滤器
│
├── 图像处理子系统（CxImage 7.0.2）
│   ├── PNG 读写（含 CgBI/iPhone 格式）
│   │   ├── PNGChunkStreamReader    ← 按 chunk 读取 PNG
│   │   └── PNGChunkStreamWriter    ← 按 chunk 写入 PNG
│   └── JPEG 读写
│       ├── JPGChunkStreamReader
│       └── JPGChunkStreamWriter
│
├── 文件系统子系统
│   ├── StoragesFstat（TJS 类）     ← 文件属性查询
│   └── TemporaryFiles（TJS 类）    ← 临时文件生命周期管理
│
├── 进程管理子系统
│   └── Process（TJS 类）           ← 进程创建/等待/管道通信
│
├── 图层扩展子系统
│   ├── layerExImage（附加到 Layer）← CxImage 集成到 Layer
│   ├── layerExRaster（附加到 Layer）← 栅格操作
│   └── FontEx（附加到 Font）       ← 字体扩展（内存字体加载）
│
├── 脚本/集合扩展子系统
│   ├── ScriptsAdd（附加到 Scripts）
│   ├── ScriptsAddForSaveStruct     ← SaveStruct 序列化扩展
│   ├── ArrayAdd（附加到 Array）    ← 数组方法扩展
│   └── DictAdd（附加到 Dictionary）← 字典方法扩展
│
└── System 命名空间扩展
    ├── writeRegValue / readEnvValue / writeEnvValue
    ├── expandEnvString / urlencode / urldecode
    ├── confirm / getAboutString
    ├── waitForAppLock / commandExecute
    ├── setDpiAwareness / getOSVersion / getKnownFolderPath
    └── Layer 命名空间扩展（shrinkCopy、fillAlpha 等）
```

---

## 7. 关键函数列表（按大小排序 Top 10）

以下是 PackinOne 中最大的 10 个非库函数（IDA 分析确认）：

| 地址 | 大小（字节） | 推断功能 |
|------|-------------|---------|
| `0x10041B50` | 7519 | **SHA-256 压缩函数**（初始化常量 `0x510E527F` = SHA-256 H4） |
| `0x10053F50` | 5142 | **LZ4 压缩核心**（含哈希常量 `9E3779B1h`，窗口 `0x10000`） |
| `0x100457D0` | 4196 | **图像量化/抖动处理**（查找表 + 误差扩散算法） |
| `0x10037AF0` | 4179 | **SIMD 图像处理**（大量 SSE `punpcklwd`/`movdqa` 指令） |
| `0x1002EE90` | 3653 | **文件选择对话框**（`OPENFILENAMEW` 结构初始化 + `GetOpenFileNameW`） |
| `0x100159D0` | 3325 | **属性访问器框架**（`ncbPropAccessor vftable` + TJS PropGet/Set） |
| `0x1003C500` | 2875 | **图像内存分配器**（`malloc` + 行步长/位深度计算） |
| `0x100348C0` | 2860 | **PNG 块解析器**（`IHDR/IDAT/IEND/CgBI` 魔数判断） |
| `0x1003E4B0` | 2647 | **JPEG 解码器**（CxImage 封装） |
| `0x10018410` | 2439 | **图像重采样**（浮点 FPU 插值，SIMD 展开） |

> **逆向优先级建议**：如果目标是实现 LZ4 相关功能，优先分析 `0x10053F50`（LZ4 压缩）和 `0x100553A0`（LZ4 解压，2052 字节）。如果目标是理解文件格式，重点看 PNG/JPEG 相关函数。SHA-256（`0x10041B50`）可能用于文件完整性校验或加密，但不是核心功能。

---

## 8. 动手实践：用 Python 复现导出表提取

以下是一个独立可运行的 Python 脚本，模拟我们的 IDA 分析中"导出表提取"步骤：

```python
#!/usr/bin/env python3
"""
示例：使用 pefile 库读取 PackinOne.dll 导出表
等效于 IDA 分析报告中的 ## EXPORTS 节
"""

import pefile  # 安装：pip install pefile
import sys

def analyze_exports(dll_path: str) -> None:
    """读取并打印 DLL 导出表"""
    pe = pefile.PE(dll_path)

    if not hasattr(pe, 'DIRECTORY_ENTRY_EXPORT'):
        print("该 DLL 无导出表")
        return

    exports = pe.DIRECTORY_ENTRY_EXPORT
    print(f"DLL 名称: {exports.name.decode()}")
    print(f"导出函数数量: {len(exports.symbols)}")
    print()

    for sym in exports.symbols:
        name = sym.name.decode() if sym.name else f"(ordinal-only)"
        addr = pe.OPTIONAL_HEADER.ImageBase + sym.address
        print(f"  [{sym.ordinal:4d}] 0x{addr:08X}  {name}")

if __name__ == "__main__":
    path = sys.argv[1] if len(sys.argv) > 1 else "PackinOne.dll"
    analyze_exports(path)

# 预期输出（PackinOne.dll）：
# DLL 名称: PackinOne.dll
# 导出函数数量: 5
#   [   1] 0x10001060  V2Link
#   [   2] 0x10001090  V2Unlink
#   [   3] 0x10001060  _V2Link@4
#   [   4] 0x10001090  _V2Unlink@0
#   [268798463] 0x100589FF  DllEntryPoint
```

### 用 dumpbin 复现（无需 Python）

```cmd
# Windows 内置工具，Visual Studio 安装后可用
dumpbin /EXPORTS PackinOne.dll

# 预期输出关键部分：
#     ordinal hint RVA      name
#           1    0 00001060 V2Link
#           2    1 00001090 V2Unlink
#           3    2 00001060 _V2Link@4
#           4    3 00001090 _V2Unlink@0
```

### 用 IDA Python 脚本提取（在 IDA 内执行）

```python
# IDA Python 脚本：列出所有导出函数
import idautils
import idc

print("=== PackinOne.dll 导出表 ===")
for i, exp in enumerate(idautils.Entries()):
    ordinal, ea, name = exp
    size = 0
    # 计算函数大小
    func = idaapi.get_func(ea)
    if func:
        size = func.size()
    print(f"  [{ordinal}] 0x{ea:08X}  {name}  ({size} bytes)")
```

---

## 9. 对照项目源码

PackinOne 的 TJS 注册机制与 krkr2 项目中已实现插件的模式完全一致：

**相关文件：**
- `cpp/plugins/fstat/` — StoragesFstat 的开源等价实现（参考对比）
- `cpp/core/plugin/` — ncbind 系统核心（`ncbind.h`、`simplebinder.h`）
- `cpp/plugins/*.cpp` — 其他已实现插件（Process、SaveStruct 等）

**ncbind 注册模式（来自 `cpp/core/plugin/ncbind.h`）：**

```cpp
// 这就是 IDA 中看到的 ncbNativeClassAutoRegister<T> 对应的宏
NCB_REGISTER_CLASS(Process) {
    NCB_CONSTRUCTOR(());                    // 构造函数
    NCB_METHOD(run);                        // 注册 run 方法
    NCB_METHOD(wait);                       // 注册 wait 方法
    NCB_PROPERTY(exitCode, getExitCode, 0); // 只读属性
}
```

与 IDA 字符串 `.?AU?$ncbNativeClassAutoRegister@VProcess@@@@` 完全吻合。

---

## 本节小结

- PackinOne.dll 是一个**综合功能插件**，集成了 LZ4 压缩、CxImage 图像处理、文件系统操作、进程管理等多个功能模块
- 导出表**只有 2 个真实函数**（V2Link/V2Unlink），所有功能通过 TJS 类/方法访问
- **导入表**是判断 DLL 能力的第二线索：KERNEL32 + CreatePipe → 进程管道通信；COMDLG32 → 文件对话框；GDI32 (Font*) → 内存字体加载
- **RTTI 字符串**（`.?AV...@@`）直接泄露 C++ 类名，免去大量手工反汇编
- **ncbind 注册器字符串**（`.?AU?$ncbNativeClassAutoRegister@V{类名}@@@@`）直接告知 TJS 中会出现哪些类
- V2Link 的核心流程：初始化 ncbind 函数指针表 → 触发所有静态注册器 → 返回 S_OK

---

## 练习题与答案

### 题目 1：V2Link 参数类型

PackinOne 的 V2Link 在汇编层接收什么类型的参数？它用这个参数做了什么？

<details>
<summary>查看答案</summary>

```asm
; V2Link 汇编：
0x10001060: mov  eax, [esp+arg_0]   ; 取参数（4 字节，即一个指针）
0x10001064: push eax
0x10001065: call sub_1000D7A0       ; 传给函数指针初始化器
```

**参数类型：`iTJSDispatch2*`**（TJS 虚机的全局调度接口指针）

**用途**：V2Link 接收 TJS 主程序的调度对象指针，然后把它传给 `sub_1000D7A0`，后者通过该对象用字符串匹配的方式查找 TJS 内部函数地址（如 `tTJSVariant::tTJSVariant()`、`tTJSVariant::~tTJSVariant()` 等），缓存到全局函数指针表中。之后所有 ncbind 操作都通过这些缓存的函数指针来创建/销毁 TJS 值。

这就是 KiriKiri 插件 ABI 的精髓：**不硬链接 TJS 内部符号，而是通过运行时字符串查找获取函数地址**，从而实现插件对主程序的版本容忍。
</details>

---

### 题目 2：识别 ncbind 类

给定以下 IDA RTTI 字符串，哪些会在 TJS 中作为**独立的新类**出现（可以 `new ClassName()`）？哪些是**附加到现有类**的？

```
.?AU?$ncbNativeClassAutoRegister@VProcess@@@@
.?AU?$ncbAttachTJS2ClassAutoRegister@VArrayAdd@@@@
.?AU?$ncbNativeClassAutoRegister@VStoragesFstat@@@@
.?AU?$ncbAttachTJS2ClassAutoRegister@VlayerExImage@@@@
```

<details>
<summary>查看答案</summary>

**独立新类（`ncbNativeClassAutoRegister`）**：
- `Process` → TJS 中可以 `var p = new Process()`
- `StoragesFstat` → TJS 中可以 `var s = new StoragesFstat()`

**附加到现有类（`ncbAttachTJS2ClassAutoRegister`）**：
- `ArrayAdd` → 给 TJS 内置 `Array` 类增加新方法，不能 `new ArrayAdd()`，而是直接在数组对象上调用新方法
- `layerExImage` → 给 TJS 内置 `Layer` 类增加新方法（如 `layer.loadCxImage()`）

区分规则很简单：模板参数是 `ncbNativeClassAutoRegister` → 独立类；是 `ncbAttachTJS2ClassAutoRegister` → 扩展现有类。
</details>

---

### 题目 3：分析 V2Unlink 的保护逻辑

为什么 V2Unlink 用的是 `jle`（小于等于跳转）而不是 `jne`（不等于跳转）？设计意图是什么？

<details>
<summary>查看答案</summary>

```asm
0x10001095: cmp   eax, dword_10094000   ; 比较当前版本 vs 安装时版本
0x1000109B: jle   short loc_100010A3    ; 如果当前版本 <= 安装时版本，才允许卸载
0x1000109D: mov   eax, 80004005h        ; 否则返回 E_FAIL（拒绝卸载）
```

**`eax` = `dword_1009403C` = 当前插件版本号**  
**`dword_10094000`** = 加载时记录的版本号（V2Link 中写入的）

`jle` 意味着：**当前版本号 <= 安装时版本号时才允许卸载**。

这处理了一个热更新场景：如果加载了版本 N 的插件（写入 `dword_10094000 = N`），然后游戏在不重启的情况下尝试加载版本 N+1（`dword_1009403C = N+1`），再调用旧版 V2Unlink（`dword_1009403C` 仍是 N+1），这时 `N+1 > N`，卸载被拒绝。

本质上是防止**版本混淆攻击**：更新的插件不能卸载旧版安装的状态。只有当前版本 <= 安装版本时（即"我就是当初安装那个版本，或者更旧的版本"），才允许执行卸载清理。
</details>

---

## 下一步

→ [02-压缩算法还原与C++实现](./02-压缩算法还原与C++实现.md)：基于本节发现的 LZ4 流架构，深入分析 LZ4 压缩/解压核心函数，并在 C++ 中实现兼容 PackinOne 流格式的完整压缩接口。
