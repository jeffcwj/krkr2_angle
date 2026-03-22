# DLL 导出表与接口识别

> **所属模块：** M12-插件逆向与实现
> **前置知识：** [无源码插件功能分析与优先级](../02-无源码插件清单/02-无源码插件功能分析与优先级.md)、[P09-逆向工程入门](../../P09-逆向工程入门/README.md)
> **预计阅读时间：** 35 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：
1. 使用 dumpbin、PE Explorer、CFF Explorer 等工具查看 DLL 的导出表
2. 解释 PE 文件格式中导出表的结构（导出目录、名称表、序号表、地址表）
3. 从导出函数名推断 KiriKiri 插件的 TJS 接口结构
4. 识别 V2Link 入口中 iTJSDispatch2 的注册模式
5. 为目标 DLL 建立完整的接口文档（函数签名、参数类型、返回值）

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| PE 格式 | Portable Executable | Windows 可执行文件（.exe/.dll/.sys）的标准二进制格式，定义了文件头、节表、导入导出表等结构 |
| 导出表 | Export Table | PE 文件中记录对外公开函数/变量的数据结构，包含函数名、序号和内存地址 |
| RVA | Relative Virtual Address | 相对虚拟地址——文件加载到内存后，某个数据相对于模块基址的偏移量 |
| 名称修饰 | Name Mangling | C++ 编译器为支持函数重载，将函数名编码为包含参数类型信息的特殊格式（如 `?func@@YAHH@Z`） |
| 反编译器 | Decompiler | 将机器码转换回高级语言（如 C/C++）伪代码的工具，如 IDA Pro 的 Hex-Rays、Ghidra |
| 交叉引用 | Cross Reference (Xref) | 反编译工具中显示"谁调用了这个函数"或"这个函数调用了谁"的导航功能 |
| 虚函数表 | Virtual Function Table (vtable) | C++ 实现多态的机制——每个含虚函数的类都有一张函数指针表，运行时通过表查找要调用的函数 |
| 序号导出 | Ordinal Export | 不使用函数名而使用数字编号来标识导出函数的方式，常见于 COM 组件和系统 DLL |

---

## 一、PE 文件格式与导出表基础

### 1.1 什么是 PE 格式

PE（Portable Executable，可移植可执行文件）是 Windows 操作系统的标准二进制文件格式。所有 `.exe`、`.dll`、`.sys`、`.ocx` 文件都遵循这个格式。对于 KiriKiri 插件逆向来说，我们关注的是 `.dll` 文件——每个 KiriKiri 插件都是一个标准的 Windows DLL。

PE 格式的整体结构可以用以下 ASCII 图表示：

```
┌──────────────────────────────────────────────────────┐
│                    PE 文件总览                         │
├──────────────────────────────────────────────────────┤
│  DOS Header          (64 字节)                       │
│  ├── e_magic = "MZ"  (DOS 时代的标志)                 │
│  └── e_lfanew        (指向 PE 签名的偏移)             │
├──────────────────────────────────────────────────────┤
│  PE Signature        (4 字节 "PE\0\0")               │
├──────────────────────────────────────────────────────┤
│  COFF Header         (20 字节)                       │
│  ├── Machine         (CPU 架构: x86/x64/ARM)         │
│  ├── NumberOfSections (节的数量)                      │
│  └── TimeDateStamp   (编译时间戳)                    │
├──────────────────────────────────────────────────────┤
│  Optional Header     (32位: 224字节 / 64位: 240字节)  │
│  ├── AddressOfEntryPoint  (程序入口 RVA)              │
│  ├── ImageBase            (默认加载基址)              │
│  ├── SizeOfImage          (内存中的总大小)            │
│  └── DataDirectory[16]    (数据目录表)                │
│      ├── [0] Export Table ← 我们的重点！              │
│      ├── [1] Import Table                            │
│      ├── [5] Base Relocation                         │
│      └── ...其他目录                                  │
├──────────────────────────────────────────────────────┤
│  Section Headers     (每节 40 字节)                   │
│  ├── .text   (代码段——可执行指令)                     │
│  ├── .rdata  (只读数据——导出表、字符串常量)           │
│  ├── .data   (可读写数据——全局变量)                   │
│  └── .rsrc   (资源——图标、版本信息)                   │
├──────────────────────────────────────────────────────┤
│  Section Data        (各节的实际内容)                  │
└──────────────────────────────────────────────────────┘
```

### 1.2 导出表的内部结构

导出表（Export Table）位于 PE 文件的 `.rdata` 节中，通过 DataDirectory[0] 的 RVA（Relative Virtual Address，相对虚拟地址——即相对于模块加载基址的偏移）定位。导出表由一个目录结构和三个并行数组组成：

```
┌─────────────────────────────────────────────────────────┐
│              Export Directory Table                       │
├─────────────────────────────────────────────────────────┤
│  Characteristics      (保留，通常为 0)                    │
│  TimeDateStamp        (导出表创建时间)                    │
│  Name RVA             (DLL 名称字符串的 RVA)              │
│  OrdinalBase          (序号基数，通常为 1)                │
│  NumberOfFunctions    (导出函数总数)                      │
│  NumberOfNames        (按名称导出的函数数)                │
│  AddressOfFunctions   → [函数地址表 RVA]                  │
│  AddressOfNames       → [函数名称表 RVA]                  │
│  AddressOfNameOrdinals→ [序号表 RVA]                      │
└─────────────┬───────────┬────────────┬──────────────────┘
              │           │            │
              ▼           ▼            ▼
┌──────────────┐ ┌─────────────┐ ┌──────────────┐
│ 地址表 (EAT) │ │ 名称表(ENT) │ │  序号表(EOT)  │
├──────────────┤ ├─────────────┤ ├──────────────┤
│ [0] 0x12340  │ │ [0]→"V2Link"│ │ [0] → 0     │
│ [1] 0x123A0  │ │ [1]→"V2Unl" │ │ [1] → 1     │
│ [2] 0x12400  │ │             │ │              │
└──────────────┘ └─────────────┘ └──────────────┘
```

**三个数组的关系：**
- **名称表**（Export Name Table, ENT）：按字母顺序排列的函数名字符串指针
- **序号表**（Export Ordinal Table, EOT）：与名称表一一对应，记录每个名称的序号
- **地址表**（Export Address Table, EAT）：按序号索引，存储函数的 RVA

要通过名称查找函数地址，过程是：名称表 → 序号表 → 地址表。

### 1.3 KiriKiri 插件的导出表特征

KiriKiri 插件的导出表有一个非常明显的特征：**只导出 V2Link 和 V2Unlink 两个函数**。这是因为 KiriKiri 的插件架构不通过导出函数来暴露功能——所有 TJS 绑定都在 V2Link 内部完成，通过 `iTJSDispatch2` 接口注册到脚本引擎。

```
KiriKiri 插件的典型导出表：

  序号  名称         RVA
  ────  ──────────  ──────────
  1     V2Link      0x00012340
  2     V2Unlink    0x000123A0

  只有 2 个导出函数！
  所有 TJS 接口都在 V2Link 内部通过引擎 API 注册。
```

这意味着**单纯看导出表无法得知插件提供了哪些 TJS 功能**。要了解完整接口，必须深入分析 V2Link 的反汇编代码。

---

## 二、导出表分析工具实战

### 2.1 dumpbin（Windows 命令行工具）

`dumpbin` 是 Visual Studio 附带的 PE 分析工具，安装 Visual Studio 后即可使用。它是分析 DLL 导出表最快的方式。

```bash
# 查看导出表
# /EXPORTS 参数只显示导出函数列表
dumpbin /EXPORTS TextRender.dll

# 输出示例：
# Microsoft (R) COFF/PE Dumper Version 14.40.xxxxx
# Copyright (C) Microsoft Corporation.  All rights reserved.
#
# Dump of file TextRender.dll
#
# File Type: DLL
#
#   Section contains the following exports for TextRender.dll
#
#     00000000 characteristics
#     XXXXXXXX time date stamp
#     0.00 version
#     1 ordinal base
#     2 number of functions
#     2 number of names
#
#     ordinal  hint  RVA      name
#           1     0  00012340 V2Link
#           2     1  000123A0 V2Unlink
#
#   Summary
#     ...

# 查看所有头信息（更详细）
dumpbin /HEADERS TextRender.dll

# 查看导入表（该 DLL 依赖哪些外部 DLL）
# 这能帮助判断插件使用了哪些系统 API
dumpbin /IMPORTS TextRender.dll

# 查看所有信息
dumpbin /ALL TextRender.dll > analysis.txt
```

**在 Linux/macOS 上的替代方案：**

Linux 和 macOS 无法直接运行 dumpbin，但可以使用以下工具分析 Windows DLL：

```bash
# Linux：使用 objdump（binutils 套件的一部分）
# 需要安装 mingw 的 binutils
# Ubuntu/Debian:
sudo apt install binutils-mingw-w64-x86-64
x86_64-w64-mingw32-objdump -p TextRender.dll | grep -A 100 "Export Table"

# 或使用 wine + dumpbin
wine dumpbin.exe /EXPORTS TextRender.dll

# macOS：使用 Homebrew 安装 mingw 工具链
brew install mingw-w64
x86_64-w64-mingw32-objdump -p TextRender.dll

# 跨平台方案：使用 Python + pefile（推荐）
pip install pefile
python3 -c "
import pefile
pe = pefile.PE('TextRender.dll')
for exp in pe.DIRECTORY_ENTRY_EXPORT.symbols:
    name = exp.name.decode() if exp.name else f'Ordinal_{exp.ordinal}'
    print(f'{exp.ordinal:5d} {name:40s} RVA=0x{exp.address:08X}')
"
```

### 2.2 CFF Explorer（免费 GUI 工具）

CFF Explorer 是 NTCore 开发的免费 PE 文件编辑器，提供友好的图形界面来浏览 PE 结构。它是分析 DLL 的首选 GUI 工具。

**安装：** 从 https://ntcore.com/explorer-suite/ 下载 Explorer Suite（包含 CFF Explorer）。

**使用步骤：**

1. 打开 CFF Explorer，将目标 DLL 拖入窗口
2. 左侧导航树中点击 **Export Directory** 查看导出表
3. 可以看到每个导出函数的：
   - **Ordinal**（序号）
   - **Name**（函数名）
   - **Function RVA**（函数在内存中的相对地址）
4. 点击 **Import Directory** 查看导入表——了解该 DLL 依赖哪些系统 API
5. 点击 **Optional Header** 查看 PE 头信息——文件大小、入口点、目标平台

**CFF Explorer 的优势：**
- 免费且轻量（<10MB）
- 可以直接编辑 PE 头字段（但我们只需要查看，不需要修改）
- 支持 32 位和 64 位 PE 文件
- 内置 hex 编辑器，可以查看导出表的原始字节

### 2.3 PE-bear（开源跨平台工具）

PE-bear 是一个开源的 PE 文件查看器，支持 Windows、Linux 和 macOS。对于非 Windows 平台的开发者来说，这是分析 Windows DLL 的好选择。

**安装：**

```bash
# Windows：从 GitHub Releases 下载
# https://github.com/hasherezade/pe-bear/releases

# Linux（AppImage 方式）：
wget https://github.com/hasherezade/pe-bear/releases/latest/download/PE-bear_linux64.AppImage
chmod +x PE-bear_linux64.AppImage
./PE-bear_linux64.AppImage

# macOS：
brew install --cask pe-bear
```

### 2.4 完整的 PE 分析脚本（跨平台）

以下是一个综合性的 Python 脚本，可以在所有平台上运行，提取 KiriKiri 插件的关键信息：

```python
#!/usr/bin/env python3
"""
KiriKiri 插件 PE 分析工具
用法: python krkr_pe_analyzer.py <dll_path>
依赖: pip install pefile
"""
import sys
import os
import struct
import pefile

def analyze_krkr_plugin(dll_path: str) -> dict:
    """全面分析一个 KiriKiri 插件 DLL"""
    pe = pefile.PE(dll_path)
    result = {
        "file_name": os.path.basename(dll_path),
        "file_size": os.path.getsize(dll_path),
        "is_64bit": pe.OPTIONAL_HEADER.Magic == 0x20B,
        "exports": [],
        "imports": {},
        "is_krkr_plugin": False,
        "sections": [],
    }

    # 分析节表——了解 DLL 的内部布局
    for section in pe.sections:
        name = section.Name.decode().rstrip('\x00')
        result["sections"].append({
            "name": name,
            "virtual_size": section.Misc_VirtualSize,
            "raw_size": section.SizeOfRawData,
            # 节的属性标志
            "executable": bool(
                section.Characteristics & 0x20000000),
            "readable": bool(
                section.Characteristics & 0x40000000),
            "writable": bool(
                section.Characteristics & 0x80000000),
        })

    # 分析导出表
    if hasattr(pe, 'DIRECTORY_ENTRY_EXPORT'):
        for exp in pe.DIRECTORY_ENTRY_EXPORT.symbols:
            name = exp.name.decode() if exp.name else None
            result["exports"].append({
                "ordinal": exp.ordinal,
                "name": name,
                "rva": exp.address,
            })
            # 检查是否为 KiriKiri 插件
            if name == "V2Link":
                result["is_krkr_plugin"] = True

    # 分析导入表——了解该 DLL 使用了哪些系统 API
    if hasattr(pe, 'DIRECTORY_ENTRY_IMPORT'):
        for entry in pe.DIRECTORY_ENTRY_IMPORT:
            dll_name = entry.dll.decode()
            funcs = []
            for imp in entry.imports:
                if imp.name:
                    funcs.append(imp.name.decode())
            result["imports"][dll_name] = funcs

    return result

def print_report(info: dict) -> None:
    """打印分析报告"""
    print(f"{'=' * 60}")
    print(f"KiriKiri 插件分析报告")
    print(f"{'=' * 60}")
    print(f"文件名:     {info['file_name']}")
    print(f"文件大小:   {info['file_size']:,} 字节")
    print(f"架构:       {'64-bit (x64)' if info['is_64bit'] else '32-bit (x86)'}")
    print(f"KiriKiri:   {'✅ 是 KiriKiri 插件' if info['is_krkr_plugin'] else '❌ 不是 KiriKiri 插件'}")
    print()

    # 导出函数
    print(f"── 导出函数 ({len(info['exports'])} 个) ──")
    for exp in info["exports"]:
        name = exp["name"] or f"(序号 {exp['ordinal']})"
        print(f"  [{exp['ordinal']:3d}] {name:30s} RVA=0x{exp['rva']:08X}")
    print()

    # 节信息
    print(f"── 节表 ({len(info['sections'])} 个节) ──")
    for sec in info["sections"]:
        flags = ""
        if sec["executable"]: flags += "X"  # 可执行
        if sec["readable"]:   flags += "R"  # 可读
        if sec["writable"]:   flags += "W"  # 可写
        print(f"  {sec['name']:8s}  大小={sec['virtual_size']:8,}  属性={flags}")
    print()

    # 导入 DLL 统计
    print(f"── 导入依赖 ({len(info['imports'])} 个 DLL) ──")
    for dll, funcs in info["imports"].items():
        print(f"  {dll}: {len(funcs)} 个函数")
        # 只显示前 5 个
        for f in funcs[:5]:
            print(f"    - {f}")
        if len(funcs) > 5:
            print(f"    ... 和其他 {len(funcs)-5} 个")

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print(f"用法: {sys.argv[0]} <DLL文件路径>")
        sys.exit(1)
    info = analyze_krkr_plugin(sys.argv[1])
    print_report(info)
```

**运行效果示例：**

```
============================================================
KiriKiri 插件分析报告
============================================================
文件名:     TextRender.dll
文件大小:   389,120 字节
架构:       32-bit (x86)
KiriKiri:   ✅ 是 KiriKiri 插件

── 导出函数 (2 个) ──
  [  1] V2Link                         RVA=0x00012340
  [  2] V2Unlink                       RVA=0x000123A0

── 节表 (4 个节) ──
  .text     大小= 245,760  属性=XR
  .rdata    大小=  53,248  属性=R
  .data     大小=  12,288  属性=RW
  .rsrc     大小=   4,096  属性=R

── 导入依赖 (3 个 DLL) ──
  KERNEL32.dll: 28 个函数
    - VirtualAlloc
    - VirtualFree
    - HeapAlloc
    - GetProcessHeap
    - LoadLibraryA
    ... 和其他 23 个
  GDI32.dll: 15 个函数
    - CreateFontIndirectW
    - GetGlyphOutlineW
    - SelectObject
    - DeleteObject
    - GetTextMetricsW
    ... 和其他 10 个
  USER32.dll: 3 个函数
    - GetDC
    - ReleaseDC
    - GetSystemMetrics
```

---

## 三、从导出信息推断 TJS 接口

### 3.1 V2Link 反汇编分析方法

对于只导出 V2Link/V2Unlink 的 KiriKiri 插件，真正的接口信息隐藏在 V2Link 的函数体中。我们需要用反编译工具（如 IDA Pro 或 Ghidra）分析 V2Link 的代码，找到 TJS 注册调用。

**V2Link 的标准签名：**

```cpp
// V2Link 是 KiriKiri 引擎调用的插件初始化入口
// 参数 ppval 指向一个函数指针数组——引擎的 API 表
// 返回 S_OK (0) 表示成功
extern "C" __declspec(dllexport)
HRESULT __stdcall V2Link(iTVPFunctionExporter* exporter) {
    // 1. 获取引擎接口
    TVPInitImportStub(exporter);

    // 2. 注册 TJS 类和函数
    // 这里是我们需要逆向分析的关键部分
    // ...

    return S_OK;
}
```

在 IDA Pro 或 Ghidra 中打开 DLL 后，定位到 V2Link 函数（通常是第一个导出函数），其反编译结果的典型模式如下：

```
V2Link 反编译代码中的关键模式：

┌─────────────────────────────────────────────────────────┐
│  V2Link(exporter)                                       │
│  ├── TVPInitImportStub(exporter)  ← 初始化引擎 API     │
│  │                                                      │
│  ├── 注册模式 A：全局函数                                │
│  │   iTJSDispatch2* global = TVPGetScriptDispatch()     │
│  │   tTJSVariant val(createFunc, NULL)                  │
│  │   global->PropSet("funcName", &val, global)          │
│  │                                                      │
│  ├── 注册模式 B：TJS 类                                  │
│  │   tTJSNativeClassForPlugin* cls = ...                │
│  │   cls->RegisterNCM("methodName", callback, ...)      │
│  │   global->PropSet("ClassName", &cls_variant, ...)    │
│  │                                                      │
│  └── return S_OK                                        │
└─────────────────────────────────────────────────────────┘
```

### 3.2 识别 TJS 函数注册

在反编译代码中，TJS 函数注册有几种可辨识的特征：

**特征 1：字符串常量。** 编译器会将 TJS 函数名存储为宽字符串常量（UTF-16），在反编译结果中显示为 `L"functionName"` 或 Unicode 字符串。搜索这些字符串就能找到所有注册的函数名。

```cpp
// 反编译中可能看到的模式（IDA Pro 伪代码）
// 注意：实际的反编译输出不会这么整洁
// 但模式是一样的
wchar_t* name = L"drawText";        // TJS 方法名
iTJSDispatch2* method = CreateTJSFunction(drawText_impl);
global->PropSet(TJS_MEMBERENSURE, name, method);
```

**特征 2：PropSet 调用模式。** `iTJSDispatch2::PropSet` 是虚函数表中的第 6 个函数（偏移 0x28/0x14），在反汇编中表现为通过 vtable（虚函数表，C++ 实现多态的机制——每个含虚函数的类在内存中都有一张函数指针表）的间接调用：

```asm
; x86 反汇编中的 PropSet 调用
; this 指针在 ECX 中（__thiscall 调用约定）
mov ecx, [global_dispatch]     ; this = global
push offset L"drawText"        ; 参数: 属性名（宽字符串）
push eax                       ; 参数: tTJSVariant* 值
push 0                         ; 参数: hint (通常为 0)
push 1                         ; 参数: TJS_MEMBERENSURE (确保创建)
mov eax, [ecx]                 ; eax = vtable 指针
call [eax + 0x28]              ; 调用 vtable[10] = PropSet
```

**特征 3：ncbind 模板展开。** 如果原始插件使用了 ncbind 框架（与 KrKr2 相同的注册框架），反编译代码会呈现模板实例化的特征——大量的类型转换和包装函数。这种情况下可以直接参考 KrKr2 的 `ncbind.hpp` 来理解代码结构。

### 3.3 导入表辅助分析

虽然导出表通常只有 V2Link/V2Unlink，但**导入表**能提供大量有用信息。通过分析 DLL 导入了哪些系统 API，可以推断插件的功能领域：

| 导入的 DLL | 常见函数 | 推断的功能 |
|-----------|---------|-----------|
| GDI32.dll | CreateFontIndirectW, GetGlyphOutlineW | 字体处理、文字渲染 |
| GDI32.dll | CreateCompatibleDC, BitBlt, StretchBlt | 位图操作、图像处理 |
| KERNEL32.dll | CreateFileW, ReadFile, WriteFile | 文件 I/O |
| KERNEL32.dll | VirtualAlloc, HeapAlloc | 内存管理（可能有大数据处理） |
| ADVAPI32.dll | RegOpenKeyExW, RegQueryValueExW | 注册表操作（系统工具类） |
| WINMM.dll | waveOutOpen, midiOutOpen | 音频播放 |
| WS2_32.dll | socket, connect, send | 网络通信 |
| OPENGL32.dll | glTexImage2D, glDrawArrays | OpenGL 渲染 |

**示例：通过导入表推断 TextRender 的功能**

```
TextRender.dll 的导入分析：
  GDI32.dll:
    CreateFontIndirectW     → 创建字体对象
    GetGlyphOutlineW        → 获取字形轮廓（矢量数据）
    GetTextMetricsW         → 获取字体度量信息
    SelectObject            → 选择字体到设备上下文
    DeleteObject            → 释放 GDI 对象
    CreateCompatibleDC      → 创建设备上下文

  推断：TextRender 使用 Windows GDI 进行字体光栅化。
  在 KrKr2 跨平台实现中，需要用 FreeType 替代 GDI 的字体功能。
```

---

## 四、V2Link 入口分析实战

### 4.1 使用 Ghidra 分析 V2Link

Ghidra 是 NSA（美国国家安全局）开发的免费开源反编译工具，功能不输商业工具 IDA Pro。以下是用 Ghidra 分析 KiriKiri 插件 V2Link 的完整流程。

**安装 Ghidra（四平台）：**

```bash
# 前置依赖：Java 17+
# Windows:
# 从 https://ghidra-sre.org/ 下载 ZIP，解压后运行 ghidraRun.bat

# Linux:
sudo apt install openjdk-17-jdk  # Ubuntu/Debian
# 从 https://ghidra-sre.org/ 下载并解压
./ghidra_11.x/ghidraRun

# macOS:
brew install openjdk@17
# 从 https://ghidra-sre.org/ 下载并解压
./ghidra_11.x/ghidraRun

# Android（通过 Termux）：
# Ghidra 本身无法在 Android 上运行
# 推荐在 PC 上分析完成后，将结果导出
```

**分析步骤：**

1. **创建项目并导入 DLL：** 新建 Ghidra 项目 → File → Import File → 选择目标 DLL → 确认分析选项（默认即可） → 等待自动分析完成

2. **定位 V2Link 函数：** 在 Symbol Tree 窗口中展开 Exports → 双击 `V2Link` → 代码浏览器会跳转到该函数

3. **查看反编译结果：** Window → Decompile → 右侧面板会显示 C 伪代码

4. **搜索字符串常量：** Window → Defined Strings → 搜索 Unicode 字符串 → 查找类似 `drawText`、`setFont`、`TextRender` 等与插件功能相关的名称

5. **追踪交叉引用（Xref）：** 右键函数名 → References → Find References to → 找到谁调用了这个函数，或这个函数调用了谁

### 4.2 V2Link 反编译代码解读

以下是一个典型的 KiriKiri 插件 V2Link 函数的 Ghidra 反编译结果（经过整理和注释）：

```cpp
// Ghidra 反编译输出（已添加注释）
// 这不是原始代码，而是反编译器从机器码推导出的 C 伪代码

// V2Link 的函数签名
// param_1 是 iTVPFunctionExporter* 类型
int __stdcall V2Link(int* param_1) {
    int local_result;

    // 步骤 1：初始化引擎 API 导入
    // 这个调用将引擎的函数指针表复制到本 DLL 的全局变量中
    // 之后才能调用 TVPGetScriptDispatch 等引擎函数
    FUN_10001000(param_1);  // = TVPInitImportStub(exporter)

    // 步骤 2：获取 TJS 全局对象
    // iTJSDispatch2* global = TVPGetScriptDispatch()
    // 全局对象是注册 TJS 类和函数的根容器
    int* global_obj = (int*)FUN_10001050();

    if (global_obj != NULL) {
        // 步骤 3：创建 TJS 类对象
        // 这里创建了一个名为 "TextRender" 的 TJS 类
        int* class_obj = FUN_10002000();  // 创建类描述对象

        // 步骤 4：注册类方法
        // 每个 RegisterNCM 调用注册一个方法
        FUN_10002100(class_obj, L"drawText",  FUN_10003000, ...);
        FUN_10002100(class_obj, L"setFont",   FUN_10003100, ...);
        FUN_10002100(class_obj, L"setColor",  FUN_10003200, ...);
        FUN_10002100(class_obj, L"setOutline",FUN_10003300, ...);
        FUN_10002100(class_obj, L"setShadow", FUN_10003400, ...);
        FUN_10002100(class_obj, L"setRuby",   FUN_10003500, ...);
        // ↑ 每个宽字符串就是一个 TJS 方法名！

        // 步骤 5：将类注册到全局命名空间
        // global.TextRender = class_obj
        local_var = create_variant(class_obj);
        (**(code**)(*(int*)global_obj + 0x28))(  // PropSet 虚函数调用
            global_obj, 1, L"TextRender", 0, &local_var, global_obj);

        // 释放引用
        (**(code**)(*(int*)global_obj + 0x10))(global_obj);  // Release
        local_result = 0;  // S_OK
    } else {
        local_result = -1;  // E_FAIL
    }

    return local_result;
}
```

**从上述反编译代码中，我们可以提取出 TextRender 的 TJS 接口：**

```
TextRender 类的方法列表（从 V2Link 反编译中提取）：

  类名: TextRender
  方法:
    - drawText    (RVA: 0x10003000) — 核心渲染方法
    - setFont     (RVA: 0x10003100) — 设置字体
    - setColor    (RVA: 0x10003200) — 设置颜色
    - setOutline  (RVA: 0x10003300) — 设置描边
    - setShadow   (RVA: 0x10003400) — 设置阴影
    - setRuby     (RVA: 0x10003500) — 设置 Ruby 注音

  接下来需要逐个分析每个方法的参数和返回值。
```

### 4.3 名称修饰（Name Mangling）处理

部分插件的 V2Link 之外可能还有其他导出函数，且使用了 C++ 名称修饰。名称修饰（Name Mangling）是 C++ 编译器为支持函数重载而对函数名进行的编码——将参数类型信息嵌入函数名中。

```
# MSVC 名称修饰示例
# 格式: ?<函数名>@@<调用约定><返回类型><参数类型>@Z

?drawText@@YGHPAUHWND__@@PBGH@Z
│ │        │ │  │              │
│ │        │ │  │              └── 参数结束标记
│ │        │ │  └── 参数: PBGH = const unsigned short*, int
│ │        │ └── 返回: G = unsigned short
│ │        └── 调用约定: Y = __cdecl
│ └── 函数名: drawText
└── 修饰名开始标记

# 使用 undname 工具还原（Visual Studio 附带）
undname "?drawText@@YGHPAUHWND__@@PBGH@Z"
# 输出: unsigned short __cdecl drawText(struct HWND__*, unsigned short const*, int)
```

**在 Linux/macOS 上还原 MSVC 修饰名：**

```python
#!/usr/bin/env python3
"""MSVC 名称修饰还原工具（简化版）"""
# 完整的 MSVC demangling 比较复杂，推荐使用在线工具
# https://demangler.com/ 支持 MSVC 和 GCC/Clang 两种格式

import subprocess
import sys

def demangle_msvc(mangled: str) -> str:
    """尝试使用 wine + undname 还原 MSVC 修饰名"""
    try:
        result = subprocess.run(
            ["wine", "undname.exe", mangled],
            capture_output=True, text=True, timeout=5)
        return result.stdout.strip()
    except (FileNotFoundError, subprocess.TimeoutExpired):
        return f"(无法还原: {mangled})"

if __name__ == "__main__":
    for name in sys.argv[1:]:
        print(f"{name}")
        print(f"  → {demangle_msvc(name)}")
```

---

## 五、建立接口文档

### 5.1 接口文档模板

当我们从导出表和 V2Link 分析中收集到足够信息后，需要将其整理成结构化的接口文档。这份文档是后续实现替代插件的蓝图。

```markdown
# [插件名] 接口文档

## 基本信息
- DLL 名称: [name.dll]
- 文件大小: [xxx KB]
- 架构: [x86/x64]
- 导出函数数: [N]
- 依赖 DLL: [列表]

## TJS 接口

### 类: [ClassName]

| 方法名 | 参数 | 返回值 | 功能说明 | 实现 RVA |
|--------|------|--------|---------|----------|
| method1 | (int a, str b) | void | 说明 | 0x1234 |
| method2 | (Layer layer) | bool | 说明 | 0x1238 |

### 全局函数

| 函数名 | 参数 | 返回值 | 功能说明 | 实现 RVA |
|--------|------|--------|---------|----------|
| func1 | (str path) | int | 说明 | 0x1300 |

## 系统 API 依赖
- GDI32.dll: CreateFontIndirectW, GetGlyphOutlineW, ...
- KERNEL32.dll: VirtualAlloc, HeapAlloc, ...

## 逆向难点
- [列出已识别的复杂算法或不明确的逻辑]

## 替代方案
- [可用的开源库或标准实现]
```

### 5.2 参数类型推断技巧

V2Link 反编译代码中的注册函数通常不直接显示参数类型。需要通过以下技巧推断：

**技巧 1：从方法实现函数的反编译代码推断**

```cpp
// 假设 Ghidra 反编译 drawText 的实现函数为：
void FUN_10003000(int* this_ptr, int* result, int numParams,
                  int** params) {
    // params[0] 是第一个参数
    int* layer = *(int**)(params[0] + 0x10);  // iTJSDispatch2*
    int x = *(int*)(params[1]);                // tjs_int
    int y = *(int*)(params[2]);                // tjs_int
    wchar_t* text = *(wchar_t**)(params[3] + 0x08);  // tjs_string
    // ...
}

// 推断结果:
// drawText(Layer layer, int x, int y, string text)
```

**技巧 2：从游戏脚本中的调用反推**

```javascript
// 在游戏的 .tjs/.ks 脚本中搜索插件函数的调用
// 调用方式直接揭示了参数类型和数量

tr.drawText(layer, 10, 20, "Hello");
// → 4 个参数: Layer, int, int, string

tr.setFont("MS Gothic", 24);
// → 2 个参数: string, int

tr.setOutline(0x000000, 2, 0x333333, 4);
// → 4 个参数: int(颜色), int(宽度), int(颜色), int(宽度)
```

**技巧 3：从已知的 A 类插件实现对比**

KrKr2 项目中有很多已实现的 A 类插件，它们的注册模式可以作为参考。例如，`scriptsEx.cpp` 中的 TJS 类注册方式可以帮助理解 C 类插件的反编译代码。

---

## 动手实践

> 以下是分步骤的实操练习，帮助读者掌握 DLL 分析技能。

### 实践 1：分析 KrKr2 已有插件的导出表

**目标：** 用 Python 分析 KrKr2 项目中编译出的 krkr2.exe 或 krkr2plugin 库文件，了解其导出和导入特征。

**步骤：**

1. 安装 pefile 库：

```bash
pip install pefile
```

2. 编译 KrKr2 项目（Windows Debug 配置）：

```bash
cmake --preset="Windows Debug Config"
cmake --build --preset="Windows Debug Build"
```

3. 分析编译产物：

```python
#!/usr/bin/env python3
"""分析 KrKr2 编译产物"""
import pefile
import sys

def quick_analysis(path: str) -> None:
    pe = pefile.PE(path)

    # 打印基本信息
    print(f"文件: {path}")
    print(f"类型: {'DLL' if pe.is_dll() else 'EXE'}")
    print(f"架构: {'x64' if pe.OPTIONAL_HEADER.Magic == 0x20B else 'x86'}")

    # 导出计数
    export_count = 0
    if hasattr(pe, 'DIRECTORY_ENTRY_EXPORT'):
        export_count = len(pe.DIRECTORY_ENTRY_EXPORT.symbols)
    print(f"导出函数: {export_count} 个")

    # 导入计数
    import_count = 0
    if hasattr(pe, 'DIRECTORY_ENTRY_IMPORT'):
        for entry in pe.DIRECTORY_ENTRY_IMPORT:
            import_count += len(entry.imports)
    print(f"导入函数: {import_count} 个")

    # 节表
    print(f"节数: {len(pe.sections)}")
    for section in pe.sections:
        name = section.Name.decode().rstrip('\x00')
        print(f"  {name:8s} {section.Misc_VirtualSize:>10,} bytes")

if __name__ == "__main__":
    quick_analysis(sys.argv[1])
```

### 实践 2：在 Ghidra 中追踪 V2Link 注册流程

**目标：** 用 Ghidra 打开一个 KiriKiri 插件 DLL，找到所有注册的 TJS 方法名。

**步骤：**

1. 打开 Ghidra → 创建新项目 → 导入 DLL 文件
2. 等待自动分析完成（Analysis → Auto Analyze）
3. 在 Symbol Tree → Exports 中找到 `V2Link`
4. 双击跳转到 V2Link 函数
5. 在 Decompile 窗口中阅读反编译代码
6. 查找所有宽字符串引用——这些就是 TJS 方法名
7. 使用 Window → Defined Strings → 过滤 Unicode 字符串 → 搜索可能的方法名
8. 对每个找到的宽字符串，右键 → References → Find References to，追踪谁引用了这个字符串
9. 记录所有找到的 TJS 方法名和对应的实现函数地址（RVA）
10. 将结果整理为接口文档表格

**预期结果（以 TextRender 为例）：**

```
你在 Ghidra 中应该能找到类似以下内容：

Defined Strings 中的 Unicode 字符串：
  0x1000A100  u"TextRender"     ← 类名
  0x1000A120  u"drawText"       ← 方法名
  0x1000A140  u"setFont"        ← 方法名
  0x1000A160  u"setColor"       ← 方法名
  0x1000A180  u"setOutline"     ← 方法名
  0x1000A1A0  u"setShadow"      ← 方法名
  0x1000A1C0  u"setRuby"        ← 方法名

交叉引用追踪结果：
  u"drawText" → 被 FUN_10002100 引用 → 在 V2Link 中被调用
  → FUN_10002100 的第三个参数是 FUN_10003000（drawText 的实现函数）

最终整理的接口列表：
  类: TextRender
  方法: drawText, setFont, setColor, setOutline, setShadow, setRuby
  每个方法的实现函数 RVA 已知，可以进一步分析参数。
```

---

## 对照项目源码

> 以下列出 KrKr2 项目中与本节内容直接相关的源码文件，帮助读者将理论知识与实际代码对应起来。

### 相关文件

- **`cpp/core/plugin/ncbind.hpp`** 第 1-50 行 — ncbind 框架的核心头文件。其中定义了 `NCB_MODULE_NAME` 宏和 `ncBindClass` 模板类。本节讲解的 V2Link 注册模式（PropSet、RegisterNCM）在 ncbind 框架中被封装为 `NCB_REGISTER_CLASS`、`NCB_ATTACH_CLASS` 等宏。阅读这个文件可以理解为什么反编译代码中会看到大量模板展开。

- **`cpp/core/plugin/ncbind.cpp`** 第 1-29 行 — `LoadModule` 函数的实现。这个函数是引擎加载插件的入口：它调用 `GetProcAddress` 获取 V2Link 的函数指针，然后调用 `V2Link(TVPGetFunctionExporter())`。这正是本节"V2Link 入口分析实战"中分析的调用链的起点。

- **`cpp/plugins/getabout.cpp`** 全文（5 行）— 最简单的 KiriKiri 插件实现。只有一个 `NCB_PRE_REGIST_CALLBACK` 宏调用，展开后生成 V2Link/V2Unlink。用这个文件对照本节的导出表分析：编译后它只会有 V2Link 和 V2Unlink 两个导出函数。

- **`cpp/plugins/scriptsEx.cpp`** 第 1-50 行 — 使用 `NCB_ATTACH_CLASS` 注册 TJS 类的示例。对照本节"从导出信息推断 TJS 接口"部分：ncbind 宏展开后，V2Link 内部会调用 `TVPGetScriptDispatch()` 获取全局对象，然后通过 `PropSet` 注册类——与本节分析的反编译模式完全一致。

- **`doc/krkr2_plugins.md`** 全文（54 行）— KrKr2 项目的完整插件清单。其中 `nan`（无源码）标记的插件正是需要用本节方法进行逆向分析的目标。对照本节"建立接口文档"部分，每个 C 类插件都需要生成一份接口文档。

---

## 常见错误与排查

### 错误 1：pefile 报错 "not a valid PE file"

**症状：** 使用 Python pefile 库打开 DLL 时抛出异常 `pefile.PEFormatError: 'DOS Header magic not found.'`

**原因分析：** 文件不是有效的 PE 格式。常见原因包括：
- 文件损坏（下载不完整）
- 文件是 Unix/Linux 的 ELF 格式，不是 Windows 的 PE 格式
- 文件是 NE（16 位 Windows）格式的古老 DLL，pefile 不支持
- 文件名正确但实际内容是文本文件（如 `.dll.txt` 被重命名为 `.dll`）

**排查步骤：**

```python
#!/usr/bin/env python3
"""验证文件是否为有效 PE 格式"""
import sys

def check_pe_magic(path: str) -> None:
    with open(path, 'rb') as f:
        # 读取前 2 字节——PE 文件必须以 "MZ" 开头
        magic = f.read(2)
        if magic == b'MZ':
            print(f"✅ {path}: DOS 头有效 (MZ)")
            # 读取 e_lfanew（偏移 0x3C 处的 4 字节）
            f.seek(0x3C)
            pe_offset = int.from_bytes(f.read(4), 'little')
            f.seek(pe_offset)
            pe_sig = f.read(4)
            if pe_sig == b'PE\x00\x00':
                print(f"✅ PE 签名有效 (偏移 0x{pe_offset:X})")
            else:
                print(f"❌ PE 签名无效: {pe_sig!r}")
        elif magic == b'\x7fE':
            print(f"❌ {path}: 这是 ELF 格式（Linux），不是 PE")
        else:
            print(f"❌ {path}: 未知格式，魔数 = {magic!r}")

if __name__ == "__main__":
    check_pe_magic(sys.argv[1])
```

**解决方案：** 确认文件来源正确；如果是从游戏目录提取的 DLL，确保提取工具没有损坏文件；对于 XP3 归档中的 DLL，使用 KrKr2 的 XP3 提取工具（`tools/xp3/`）正确解包。

### 错误 2：Ghidra 反编译结果中看不到字符串

**症状：** 在 Ghidra 的 Decompile 窗口中，V2Link 函数的反编译代码只显示十六进制地址（如 `0x1000A100`），没有显示对应的字符串内容（如 `L"drawText"`）。

**原因分析：** Ghidra 的自动分析没有识别出这些地址处的字符串数据。这在以下情况下发生：
- DLL 使用了加壳工具（如 UPX、Themida），字符串在运行时才解密
- 字符串存储在 `.data` 节而非 `.rdata` 节，Ghidra 默认不扫描可写节中的字符串
- 字符串使用了非标准编码（如 Shift-JIS 而非 UTF-16）

**排查步骤：**

1. **手动定义字符串：** 在 Listing 窗口中跳转到地址 → 右键 → Data → TerminatedUnicode（对于宽字符串）或 TerminatedCString（对于窄字符串）

2. **重新运行字符串分析：** Analysis → One Shot → ASCII Strings → 勾选 "Search All" → 设置最小长度为 3

3. **检查是否加壳：**

```python
#!/usr/bin/env python3
"""检测 DLL 是否可能被加壳"""
import pefile
import sys

def detect_packing(path: str) -> None:
    pe = pefile.PE(path)

    # 检查节名——加壳工具通常修改节名
    suspicious_names = {'UPX0', 'UPX1', '.themida', '.vmp0',
                        '.aspack', '.nsp0', '.enigma'}
    for section in pe.sections:
        name = section.Name.decode().rstrip('\x00')
        if name in suspicious_names:
            print(f"⚠️  发现可疑节名: {name} — 可能已加壳")
            return

    # 检查入口点是否在第一个节中
    ep = pe.OPTIONAL_HEADER.AddressOfEntryPoint
    first = pe.sections[0]
    if not (first.VirtualAddress <= ep
            < first.VirtualAddress + first.Misc_VirtualSize):
        print("⚠️  入口点不在第一个节中 — 可能已加壳")
        return

    # 检查节的原始大小与虚拟大小差异
    for section in pe.sections:
        name = section.Name.decode().rstrip('\x00')
        ratio = (section.SizeOfRawData /
                 max(section.Misc_VirtualSize, 1))
        if ratio < 0.1 and section.Misc_VirtualSize > 4096:
            print(f"⚠️  节 {name}: 原始大小远小于虚拟大小"
                  f" ({section.SizeOfRawData} vs "
                  f"{section.Misc_VirtualSize}) — 可能已压缩")
            return

    print("✅ 未检测到明显的加壳特征")

if __name__ == "__main__":
    detect_packing(sys.argv[1])
```

**解决方案：** 如果 DLL 确实被加壳，需要先脱壳再分析。UPX 加壳可以用 `upx -d target.dll` 直接脱壳。对于商业壳（Themida、VMProtect），需要更高级的技术（运行时 dump），超出本节范围。

### 错误 3：dumpbin 报错 "is not recognized"

**症状：** 在 Windows 命令行中运行 `dumpbin /EXPORTS xxx.dll` 时报错 `'dumpbin' is not recognized as an internal or external command`。

**原因分析：** `dumpbin` 是 Visual Studio 的组件，需要在 **Developer Command Prompt**（开发者命令提示符）中运行，而不是普通的 `cmd.exe` 或 PowerShell。普通命令行中 PATH 环境变量不包含 dumpbin 所在的目录。

**解决方案（四平台）：**

```bash
# Windows 方案 1：使用 Developer Command Prompt
# 开始菜单 → 搜索 "Developer Command Prompt for VS 2022"
# 在打开的窗口中直接运行 dumpbin

# Windows 方案 2：手动设置环境变量
# dumpbin 通常位于以下路径（根据 VS 版本和安装目录调整）：
# C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.40.xxxxx\bin\Hostx64\x64\
# 将此路径添加到 PATH 中

# Windows 方案 3：使用 vcvarsall.bat 初始化环境
"C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsall.bat" x64
dumpbin /EXPORTS target.dll

# Linux：dumpbin 不可用，使用替代工具
# 方案 A：objdump（MinGW 版）
sudo apt install binutils-mingw-w64-x86-64
x86_64-w64-mingw32-objdump -p target.dll

# 方案 B：Python pefile（推荐，跨平台）
pip install pefile
python3 -c "import pefile; pe=pefile.PE('target.dll'); [print(e.name) for e in pe.DIRECTORY_ENTRY_EXPORT.symbols]"

# macOS：同样使用 Python pefile
pip3 install pefile
python3 -c "import pefile; pe=pefile.PE('target.dll'); [print(e.name) for e in pe.DIRECTORY_ENTRY_EXPORT.symbols]"

# Android (Termux)：使用 Python pefile
pkg install python
pip install pefile
python -c "import pefile; pe=pefile.PE('target.dll'); [print(e.name) for e in pe.DIRECTORY_ENTRY_EXPORT.symbols]"
```

---

## 本节小结

- **PE 格式**是 Windows 可执行文件（.exe/.dll）的标准二进制格式，包含 DOS 头、PE 签名、COFF 头、Optional 头、节表等结构
- **导出表**由导出目录、名称表（ENT）、序号表（EOT）、地址表（EAT）三个并行数组组成，通过"名称 → 序号 → 地址"的查找链定位函数
- **KiriKiri 插件只导出 V2Link 和 V2Unlink 两个函数**，所有 TJS 接口都在 V2Link 内部通过 `iTJSDispatch2` 注册——因此单看导出表无法得知插件功能
- **dumpbin**（Windows 命令行）、**CFF Explorer**（免费 GUI）、**PE-bear**（开源跨平台）、**Python pefile**（脚本化分析）是四种常用的 PE 分析工具，各有优劣
- **V2Link 反编译**是获取 TJS 接口的核心手段：定位 `TVPInitImportStub` → `TVPGetScriptDispatch` → `RegisterNCM`/`PropSet` 调用链，提取宽字符串常量即为方法名
- **导入表分析**能辅助推断插件功能领域：GDI32.dll → 字体/图形，KERNEL32.dll VirtualAlloc → 大数据处理，WS2_32.dll → 网络通信
- **名称修饰（Name Mangling）**是 C++ 编译器的函数名编码机制，可通过 `undname`（Windows）或在线工具（demangler.com）还原为可读的函数签名
- **接口文档**是逆向成果的结构化输出，应包含类名、方法名、参数类型、返回值、实现 RVA、系统 API 依赖和替代方案
- **参数类型推断**有三种途径：反编译方法实现函数、搜索游戏脚本中的调用、对比已知的 A 类插件实现
- **加壳检测**是 PE 分析的前置步骤：如果 DLL 被加壳（UPX、Themida 等），需要先脱壳才能正确分析导出表和反编译代码

---

## 练习题与答案

### 题目 1：编写 PE 导出表批量扫描工具

编写一个 Python 脚本，接受一个目录路径作为参数，扫描该目录下所有 `.dll` 文件，对每个文件判断是否为 KiriKiri 插件（是否导出 V2Link），并输出统计报告。

要求：
1. 使用 pefile 库
2. 输出格式为表格（文件名、是否 KiriKiri 插件、导出函数数、导入 DLL 数）
3. 处理无法解析的文件（加 try-except）

<details>
<summary>查看答案</summary>

```python
#!/usr/bin/env python3
"""KiriKiri 插件批量扫描工具"""
import os
import sys
import pefile

def scan_directory(dir_path: str) -> None:
    """扫描目录下所有 DLL 文件"""
    results = []

    # 遍历目录，找到所有 .dll 文件
    for fname in sorted(os.listdir(dir_path)):
        if not fname.lower().endswith('.dll'):
            continue
        fpath = os.path.join(dir_path, fname)

        try:
            pe = pefile.PE(fpath)

            # 统计导出函数
            export_count = 0
            is_krkr = False
            if hasattr(pe, 'DIRECTORY_ENTRY_EXPORT'):
                export_count = len(
                    pe.DIRECTORY_ENTRY_EXPORT.symbols)
                for exp in pe.DIRECTORY_ENTRY_EXPORT.symbols:
                    if exp.name and exp.name.decode() == 'V2Link':
                        is_krkr = True
                        break

            # 统计导入 DLL 数量
            import_count = 0
            if hasattr(pe, 'DIRECTORY_ENTRY_IMPORT'):
                import_count = len(pe.DIRECTORY_ENTRY_IMPORT)

            results.append({
                'name': fname,
                'krkr': is_krkr,
                'exports': export_count,
                'imports': import_count,
            })
            pe.close()

        except pefile.PEFormatError:
            results.append({
                'name': fname,
                'krkr': False,
                'exports': -1,  # -1 表示解析失败
                'imports': -1,
            })

    # 打印表格
    print(f"{'文件名':30s} {'KiriKiri?':10s} "
          f"{'导出数':8s} {'导入DLL数':10s}")
    print('-' * 62)
    krkr_count = 0
    for r in results:
        krkr_str = '✅ 是' if r['krkr'] else '❌ 否'
        if r['exports'] == -1:
            krkr_str = '⚠️ 解析失败'
        print(f"{r['name']:30s} {krkr_str:10s} "
              f"{r['exports']:8d} {r['imports']:10d}")
        if r['krkr']:
            krkr_count += 1

    print(f"\n共扫描 {len(results)} 个 DLL，"
          f"其中 {krkr_count} 个为 KiriKiri 插件")

if __name__ == '__main__':
    if len(sys.argv) != 2:
        print(f"用法: {sys.argv[0]} <目录路径>")
        sys.exit(1)
    scan_directory(sys.argv[1])
```

**运行效果：**

```
文件名                         KiriKiri?  导出数   导入DLL数
--------------------------------------------------------------
PackinOne.dll                  ✅ 是            2          3
TextRender.dll                 ✅ 是            2          3
lzfs.dll                       ✅ 是            2          2
msvcr100.dll                   ❌ 否         1478          4

共扫描 4 个 DLL，其中 3 个为 KiriKiri 插件
```

</details>

### 题目 3：通过导入表推断插件功能

一个未知的 KiriKiri 插件 `mystery.dll` 的导入表分析结果如下：

```
KERNEL32.dll: CreateFileW, ReadFile, WriteFile, GetFileSize,
              SetFilePointer, CloseHandle, VirtualAlloc, VirtualFree,
              GetTickCount, QueryPerformanceCounter
GDI32.dll:    (无)
USER32.dll:   (无)
ADVAPI32.dll: (无)
MSVCRT.dll:   malloc, free, memcpy, memset, sprintf
```

根据导入表信息，回答以下问题：
1. 这个插件最可能的功能领域是什么？（字体渲染 / 文件处理 / 网络通信 / 音频播放）
2. 为什么 `VirtualAlloc` 和 `VirtualFree` 的存在值得注意？
3. `GetTickCount` 和 `QueryPerformanceCounter` 暗示了什么？
4. 如果要为这个插件编写跨平台替代实现，哪些 Windows API 需要被替换为 POSIX/标准库函数？

<details>
<summary>查看答案</summary>

**答案：**

1. **最可能的功能领域是"文件处理"。** 理由：
   - 大量文件 I/O API（`CreateFileW`、`ReadFile`、`WriteFile`、`GetFileSize`、`SetFilePointer`、`CloseHandle`）表明核心功能是文件读写
   - 没有 GDI32.dll 导入，排除字体渲染
   - 没有 WS2_32.dll 导入，排除网络通信
   - 没有 WINMM.dll 或音频相关 DLL，排除音频播放
   - 这个导入模式与 PackinOne（归档/压缩插件）非常相似

2. **`VirtualAlloc`/`VirtualFree` 的存在值得注意**，因为它们是 Windows 的虚拟内存管理 API，直接操作操作系统的页表，比 `malloc`/`free` 更底层。通常在以下场景使用：
   - 需要分配大块连续内存（>1MB）——暗示插件可能处理大文件或大数据块
   - 需要设置内存保护属性（可读/可写/可执行）——与解压缩或解密操作有关
   - 自定义内存分配器——性能关键路径的优化

3. **`GetTickCount` 和 `QueryPerformanceCounter` 暗示了性能计时功能：**
   - `GetTickCount` 提供毫秒级精度的系统时间
   - `QueryPerformanceCounter` 提供微秒/纳秒级精度的高分辨率计时器
   - 两者同时存在说明插件可能有性能基准测试功能，或者对处理时间有严格要求（如实时压缩/解压需要控制延迟）

4. **跨平台替换对照表：**

| Windows API | POSIX/标准库替代 | 说明 |
|------------|-----------------|------|
| `CreateFileW` | `fopen` / `open` | C 标准库 / POSIX 文件打开 |
| `ReadFile` | `fread` / `read` | 文件读取 |
| `WriteFile` | `fwrite` / `write` | 文件写入 |
| `GetFileSize` | `fseek`+`ftell` / `stat` | 获取文件大小 |
| `SetFilePointer` | `fseek` / `lseek` | 移动文件指针 |
| `CloseHandle` | `fclose` / `close` | 关闭文件 |
| `VirtualAlloc` | `mmap` / `aligned_alloc` | 内存映射或对齐分配 |
| `VirtualFree` | `munmap` / `free` | 释放内存 |
| `GetTickCount` | `clock_gettime(CLOCK_MONOTONIC)` | Linux/macOS 单调时钟 |
| `QueryPerformanceCounter` | `std::chrono::high_resolution_clock` | C++11 标准高精度计时 |

</details>

---

## 下一步

下一节 [数据结构推断与行为还原](./02-数据结构推断与行为还原.md) 将讲解如何从反编译代码中识别 C++ 数据结构（结构体、类、枚举），推断算法逻辑，并将其还原为可编译的 C++ 代码——这是从"知道接口"到"实现替代"的关键一步。


阅读以下 Ghidra 反编译伪代码片段（简化版），回答问题：

```cpp
int __stdcall V2Link(int* param_1) {
    FUN_10001000(param_1);
    int* g = (int*)FUN_10001050();
    if (g != NULL) {
        int* c1 = FUN_10002000();
        FUN_10002100(c1, L"compress", FUN_10003000, 0, 0);
        FUN_10002100(c1, L"decompress", FUN_10003100, 0, 0);
        FUN_10002100(c1, L"getVersion", FUN_10003200, 0, 0);
        local_v1 = create_variant(c1);
        (**(code**)(*(int*)g + 0x28))(g, 1, L"PackinOne", 0, &local_v1, g);

        int* c2 = FUN_10002000();
        FUN_10002100(c2, L"open", FUN_10004000, 0, 0);
        FUN_10002100(c2, L"close", FUN_10004100, 0, 0);
        FUN_10002100(c2, L"read", FUN_10004200, 0, 0);
        FUN_10002100(c2, L"write", FUN_10004300, 0, 0);
        FUN_10002100(c2, L"seek", FUN_10004400, 0, 0);
        local_v2 = create_variant(c2);
        (**(code**)(*(int*)g + 0x28))(g, 1, L"PackinStream", 0, &local_v2, g);

        (**(code**)(*(int*)g + 0x10))(g);
        return 0;
    }
    return -1;
}
```

问题：
1. 这个插件注册了几个 TJS 类？类名分别是什么？
2. 每个类各有哪些方法？
3. `FUN_10001000` 和 `FUN_10001050` 分别对应什么引擎 API？
4. `(*(int*)g + 0x28)` 的间接调用对应 iTJSDispatch2 的哪个方法？

<details>
<summary>查看答案</summary>

**答案：**

1. **注册了 2 个 TJS 类：**
   - `PackinOne`（压缩/解压功能类）
   - `PackinStream`（流式读写类）

2. **各类的方法列表：**
   - PackinOne 类：
     - `compress`（RVA: 0x10003000）— 压缩数据
     - `decompress`（RVA: 0x10003100）— 解压数据
     - `getVersion`（RVA: 0x10003200）— 获取版本号
   - PackinStream 类：
     - `open`（RVA: 0x10004000）— 打开流
     - `close`（RVA: 0x10004100）— 关闭流
     - `read`（RVA: 0x10004200）— 读取数据
     - `write`（RVA: 0x10004300）— 写入数据
     - `seek`（RVA: 0x10004400）— 移动读写位置

3. **引擎 API 对应关系：**
   - `FUN_10001000(param_1)` → `TVPInitImportStub(exporter)` — 初始化引擎函数导入表
   - `FUN_10001050()` → `TVPGetScriptDispatch()` — 获取 TJS 全局对象

4. **`(*(int*)g + 0x28)` 对应 `iTJSDispatch2::PropSet`** — 这是 vtable 中偏移 0x28 处的虚函数，用于将类对象设置为全局对象的属性。`PropSet` 的第二个参数 `1` 是 `TJS_MEMBERENSURE` 标志（确保属性不存在时自动创建），第三个参数（`L"PackinOne"` 或 `L"PackinStream"`）是属性名。

</details>


