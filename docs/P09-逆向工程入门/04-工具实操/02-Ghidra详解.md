# Ghidra 详解

## 本节目标

- 了解 Ghidra 的架构、安装配置和项目管理
- 掌握 CodeBrowser 主界面的导航和分析操作
- 熟练使用 Ghidra 反编译器阅读和改善反编译输出
- 入门 Ghidra 脚本（Java / Python）自动化分析
- 对比 Ghidra 与 IDA 的优劣，掌握互补使用策略
- 在 KiriKiri2 DLL 上实际演练 Ghidra 分析流程

---

## 1. Ghidra 概述与安装

### 1.1 什么是 Ghidra

Ghidra 是美国 NSA（国家安全局）开发并开源的逆向工程框架：

```
┌──────────────────────────────────────────────────────────┐
│                    Ghidra 核心优势                        │
├──────────────────────────────────────────────────────────┤
│ ✓ 完全免费开源 (Apache 2.0)                              │
│ ✓ 内置反编译器（本地运行，无需云端）                       │
│ ✓ 支持 30+ 处理器架构                                    │
│ ✓ 强大的脚本系统 (Java + Python)                         │
│ ✓ 多人协作分析 (Ghidra Server)                           │
│ ✓ 版本控制集成                                           │
│ ✓ 活跃的社区和插件生态                                    │
├──────────────────────────────────────────────────────────┤
│ △ 主要劣势                                               │
│ - 基于 Java，启动较慢，内存占用较高                        │
│ - UI 不如 IDA 流畅                                       │
│ - 反编译质量在某些场景不如 Hex-Rays                       │
│ - 调试器功能相对简单                                      │
└──────────────────────────────────────────────────────────┘
```

### 1.2 安装配置

```bash
# 前置依赖: JDK 17+ (推荐 OpenJDK 或 Adoptium)
# 下载 JDK:
# https://adoptium.net/temurin/releases/

# 设置 JAVA_HOME 环境变量 (Windows):
setx JAVA_HOME "C:\Program Files\Eclipse Adoptium\jdk-17"

# 下载 Ghidra:
# https://github.com/NationalSecurityAgency/ghidra/releases

# 解压到目标目录 (建议路径不含中文和空格):
# C:\Tools\ghidra_11.x

# 启动 Ghidra:
# Windows: 双击 ghidraRun.bat
# Linux/macOS: ./ghidraRun
```

**首次启动配置**：

```
1. 首次启动会要求确认 JDK 路径
2. 打开 Project 窗口 → File → New Project
3. 选择 "Non-Shared Project" (本地项目)
4. 设置项目路径和名称:
   目录: C:\GhidraProjects
   名称: KiriKiri2_Analysis
5. 点击 Finish
```

---

## 2. 项目管理与文件导入

### 2.1 导入二进制文件

```
File → Import File → 选择 PE 文件

导入选项对话框:
┌────────────────────────────────────────────────┐
│ Format:    Portable Executable (PE)            │
│ Language:  x86:LE:32:default (32位小端)         │
│            或 x86:LE:64:default (64位)          │
│ Options:                                       │
│   ☑ Load External Libraries (加载依赖)         │
│   ☑ Create Bookmarks (创建书签)                │
└────────────────────────────────────────────────┘

导入后:
- 文件出现在 Project 窗口的文件列表中
- 双击打开 → 进入 CodeBrowser (主分析界面)
- 首次打开会提示 "Analyze?" → 点击 "Yes"
```

### 2.2 自动分析选项

```
首次分析时的推荐配置:
┌────────────────────────────────────────────────┐
│ ☑ ASCII Strings                                │
│ ☑ Apply Data Archives                          │
│ ☑ Call Convention Identification                │
│ ☑ Call-Fixup Installer                         │
│ ☑ Create Address Tables                        │
│ ☑ Data Reference                               │
│ ☑ Decompiler Parameter ID                      │
│ ☑ Decompiler Switch Analysis                   │
│ ☑ Demangler (MSVC / GNU)                       │
│ ☑ Disassemble                                  │
│ ☑ Function Start Search                        │
│ ☑ Non-Returning Functions - Discovered          │
│ ☑ Reference                                    │
│ ☑ Stack                                        │
│ ☑ Windows x86 PE Exception Handling             │
│ ☑ WindowsPE x86 Propagate External Parameters   │
└────────────────────────────────────────────────┘

; 点击 "Analyze" 开始
; 右下角进度条显示分析进度
; 分析完成后自动保存
```

---

## 3. CodeBrowser 界面

### 3.1 主要窗口布局

```
┌──────────────────────────────────────────────────────────┐
│                     菜单栏 / 工具栏                       │
├──────────────────┬───────────────────┬────────────────────┤
│                  │                   │                    │
│ Program Trees    │   Listing         │   Decompile        │
│ (程序结构树)      │   (反汇编视图)     │   (反编译窗口)     │
│                  │                   │                    │
│ Symbol Tree      │                   │                    │
│ (符号树)         │                   │                    │
│ - Imports        │                   │                    │
│ - Exports        │                   │                    │
│ - Functions      │                   │                    │
│ - Labels         │                   │                    │
│ - Classes        │                   │                    │
│ - Namespaces     │                   │                    │
├──────────────────┴───────────────────┴────────────────────┤
│              Console / Data Type Manager                   │
└──────────────────────────────────────────────────────────┘
```

### 3.2 核心快捷键

```
┌──────────┬──────────────────────────────────────────────┐
│ 快捷键   │ 功能                                         │
├──────────┼──────────────────────────────────────────────┤
│ G        │ Go to address — 跳转到地址                   │
│ L        │ Label — 重命名/添加标签                       │
│ ;        │ Set comment — 添加注释                       │
│ Ctrl+Shift+F │ Search → For Strings — 搜索字符串       │
│ Ctrl+Shift+E │ Search → For Direct References          │
│ Ctrl+E   │ Show references to — 交叉引用                │
│ Alt+←    │ Back — 导航返回                              │
│ Alt+→    │ Forward — 导航前进                           │
│ F        │ Edit function — 编辑函数属性                  │
│ T        │ Choose data type — 选择数据类型               │
│ Ctrl+L   │ Retype variable — 修改变量类型               │
│ Ctrl+Shift+B │ Function Call Trees — 函数调用树         │
│ Space    │ 在 Listing 中切换展开/折叠                    │
└──────────┴──────────────────────────────────────────────┘
```

---

## 4. 反编译器（Decompiler）

### 4.1 Ghidra 反编译器特点

Ghidra 最大的优势之一是内置免费的反编译器：

```
┌──────────────────────────────────────────────────────────┐
│              Ghidra Decompiler vs Hex-Rays                │
├──────────────────┬──────────────────┬────────────────────┤
│                  │ Ghidra           │ Hex-Rays (IDA)     │
├──────────────────┼──────────────────┼────────────────────┤
│ 价格             │ 免费             │ $1400+/年          │
│ 反编译质量       │ 优秀 (90%)       │ 顶级 (95%+)        │
│ 运行方式         │ 本地             │ 本地 (Pro) / 云端  │
│ 架构支持         │ 30+              │ ~10 (需单独购买)   │
│ 类型恢复         │ 良好             │ 优秀              │
│ C++ 支持        │ 良好             │ 优秀              │
│ 交互修改         │ 支持             │ 支持              │
│ 脚本访问         │ 完整 API         │ 完整 API          │
└──────────────────┴──────────────────┴────────────────────┘
```

### 4.2 使用反编译器

```
1. 在 Listing 窗口双击函数进入
2. Decompile 窗口自动显示反编译结果
3. 两个窗口同步高亮（点击一处，另一处对应位置高亮）

反编译窗口示例:

undefined4 FUN_10001000(int param_1)
{
    undefined4 uVar1;
    int iVar2;

    if (param_1 == 0) {
        uVar1 = 0xffffffff;
    }
    else {
        iVar2 = 0;
        while (*(short *)(param_1 + iVar2 * 2) != 0) {
            iVar2 = iVar2 + 1;
        }
        uVar1 = iVar2;
    }
    return uVar1;
}
```

### 4.3 改善反编译输出

```
; 技巧 1: 修改函数签名
; 右键函数名 → Edit Function Signature
; 输入: int __stdcall wcslen_custom(wchar_t * str)
; Ghidra 重新分析，输出更清晰

; 技巧 2: 修改变量类型
; 右键变量 → Retype Variable (Ctrl+L)
; 将 param_1 改为 wchar_t *
; 将 iVar2 改为 int

; 技巧 3: 重命名变量
; 右键变量 → Rename Variable
; param_1 → str, iVar2 → len

; 修改后的反编译输出:
int wcslen_custom(wchar_t *str)
{
    int len;
    if (str == NULL) {
        return -1;
    }
    len = 0;
    while (str[len] != 0) {
        len++;
    }
    return len;
}
; 比原始输出清晰得多！
```

### 4.4 数据类型管理器

```
Window → Data Type Manager

Ghidra 的类型系统:
├── BuiltInTypes (内置类型)
│   ├── byte, word, dword, qword
│   ├── char, wchar_t, int, long
│   └── float, double
├── windows_vs12_32 (Windows SDK 类型)
│   ├── HANDLE, HWND, HINSTANCE
│   ├── DWORD, WORD, BYTE
│   └── LPCWSTR, LPVOID
└── [项目自定义类型]
    ├── tTJSVariant (手动创建)
    ├── iTJSDispatch2 (手动创建)
    └── ...

; 创建自定义结构体:
; 右键自定义类型分类 → New → Structure
; 或 Data Type Manager 工具栏 → Create Structure
```

创建 KiriKiri2 类型的示例：

```
Structure: tTJSVariant (size: 0x18)
┌────────┬────────────┬──────────────────┐
│ Offset │ Type       │ Name             │
├────────┼────────────┼──────────────────┤
│ 0x00   │ pointer    │ vtable           │
│ 0x04   │ int        │ vt               │
│ 0x08   │ int        │ padding          │
│ 0x0C   │ longlong   │ IntValue         │
│ 0x14   │ pointer    │ padding2         │
└────────┴────────────┴──────────────────┘

; 在反编译窗口中应用:
; 右键变量 → Retype Variable → 选择 tTJSVariant *
```

---

## 5. 高级分析功能

### 5.1 函数调用图（Call Graph）

```
Window → Function Call Graph

; 或在函数上右键 → References → Show Call Trees

; 调用图显示:
; - Incoming calls: 谁调用了这个函数
; - Outgoing calls: 这个函数调用了谁
; - 可视化展开调用链

; KiriKiri2 分析示例:
; V2Link 的调用图:
; V2Link → TVPImportFuncPtr → (多个 API 获取)
;        → RegisterPlugin → ncbRegisterClass
;                         → ncbRegisterMethod
```

### 5.2 Function ID（签名匹配）

```
Ghidra 的 Function ID 类似 IDA 的 FLIRT:
- 自动识别已知库函数（CRT、STL 等）
- 通过函数字节模式匹配签名数据库

; 应用 Function ID:
; Analysis → One Shot → Function ID
; 或在初始分析时确保勾选了 "Apply Data Archives"

; 自定义签名库:
; Tools → Function ID → Create new FidDb
; 可以导入自己的 PDB 创建签名数据库
; 适用于分析多个使用相同库的 KiriKiri2 插件
```

### 5.3 补丁与标注

```
; Ghidra 允许直接修改二进制（用于测试/补丁）:

; 修改指令:
; 右键指令 → Patch Instruction
; 例: 将 jz 改为 jmp (跳过检查)
; 原始: 74 0A (jz +0xA)
; 修改: EB 0A (jmp +0xA)

; 修改字节:
; 右键 → Patch Data
; 直接修改十六进制值

; 导出补丁后的文件:
; File → Export Program → Binary
; 选择输出格式和路径
```

---

## 6. Ghidra 脚本

### 6.1 脚本管理器

```
Window → Script Manager

; 脚本管理器界面:
┌────────────────────────────────────────────────┐
│ 分类:                                          │
│ ├── Analysis    (分析相关脚本)                  │
│ ├── Data        (数据处理脚本)                  │
│ ├── Functions   (函数分析脚本)                  │
│ ├── Search      (搜索脚本)                     │
│ └── [用户自定义]                                │
│                                                │
│ 内置脚本数量: 100+                              │
│ 支持语言: Java, Python (Jython/Pyhidra)         │
└────────────────────────────────────────────────┘

; 运行脚本: 选中 → 点击绿色运行按钮
; 新建脚本: 右上角 Create New Script
```

### 6.2 Java 脚本示例

```java
// ListExports.java — 列出所有导出函数
// @category Analysis
// @description 列出 PE 文件的所有导出函数及其地址

import ghidra.app.script.GhidraScript;
import ghidra.program.model.symbol.*;

public class ListExports extends GhidraScript {
    @Override
    public void run() throws Exception {
        SymbolTable st = currentProgram.getSymbolTable();
        SymbolIterator iter = st.getExternalEntryPointIterator();

        println("=== Exported Functions ===");
        int count = 0;
        while (iter.hasNext()) {
            Symbol sym = iter.next();
            printf("  %s at %s\n", sym.getName(), sym.getAddress());
            count++;
        }
        println("Total exports: " + count);
    }
}
```

```java
// FindVirtualCalls.java — 查找所有虚函数调用
// @category Analysis
// @description 识别 call [reg+offset] 模式的虚函数调用

import ghidra.app.script.GhidraScript;
import ghidra.program.model.listing.*;
import ghidra.program.model.lang.*;

public class FindVirtualCalls extends GhidraScript {
    @Override
    public void run() throws Exception {
        Listing listing = currentProgram.getListing();
        InstructionIterator instIter =
            listing.getInstructions(currentProgram.getMemory(), true);

        int vcallCount = 0;
        while (instIter.hasNext()) {
            Instruction inst = instIter.next();
            if (inst.getMnemonicString().equals("CALL")) {
                String rep = inst.getDefaultOperandRepresentation(0);
                // 匹配间接调用模式: [EAX+0x...] 或 [reg+offset]
                if (rep.contains("[") && rep.contains("+")) {
                    Function func = getFunctionContaining(inst.getAddress());
                    String funcName = func != null ?
                        func.getName() : "unknown";
                    printf("  VCall: %s in %s: %s\n",
                        inst.getAddress(), funcName, rep);
                    vcallCount++;
                }
            }
        }
        println("Total virtual calls found: " + vcallCount);
    }
}
```

### 6.3 Python (Jython) 脚本示例

```python
# find_tjs_strings.py — 查找 TJS 相关字符串
# @category Search
# @description 搜索包含 TJS 关键字的字符串

from ghidra.program.model.data import StringDataType

# 获取所有已定义的字符串
data_iter = currentProgram.getListing().getDefinedData(True)
tjs_strings = []

for data in data_iter:
    if data.hasStringValue():
        value = data.getValue()
        if value and any(kw in str(value).lower()
                        for kw in ['tjs', 'oncreate', 'ondestroy',
                                   'v2link', 'dispatch']):
            tjs_strings.append((data.getAddress(), str(value)))

print("=== TJS-Related Strings ===")
for addr, s in sorted(tjs_strings, key=lambda x: str(x[0])):
    # 查找引用
    refs = getReferencesTo(addr)
    ref_count = sum(1 for _ in refs)
    print("  {} : \"{}\" ({} refs)".format(addr, s, ref_count))

print("Total: {} TJS-related strings".format(len(tjs_strings)))
```

```python
# analyze_calling_conventions.py — 统计调用约定分布
# @category Analysis

from ghidra.program.model.listing import Function

fm = currentProgram.getFunctionManager()
functions = fm.getFunctions(True)

conventions = {}
for func in functions:
    cc = func.getCallingConventionName()
    conventions[cc] = conventions.get(cc, 0) + 1

print("=== Calling Convention Distribution ===")
total = sum(conventions.values())
for cc, count in sorted(conventions.items(),
                       key=lambda x: -x[1]):
    pct = count * 100.0 / total
    print("  {:20s}: {:4d} ({:5.1f}%)".format(cc, count, pct))
print("Total functions: {}".format(total))
```

---

## 7. Ghidra vs IDA 互补策略

### 7.1 何时用 Ghidra

```
✓ 需要免费反编译器（预算有限或学习阶段）
✓ 分析非 x86 架构（ARM、MIPS、PowerPC 等）
✓ 需要多人协作分析（Ghidra Server）
✓ 需要深度定制的脚本自动化
✓ 需要修改/补丁二进制文件
✓ 学习和教学场景
```

### 7.2 何时用 IDA

```
✓ 追求最高反编译质量（复杂 C++ 代码）
✓ 需要成熟的调试器集成
✓ 需要 FLIRT 库函数识别（更成熟）
✓ 需要处理混淆/保护的二进制
✓ 团队已有 IDA 工作流和插件生态
✓ 需要最流畅的 UI 体验
```

### 7.3 推荐工作流

```
┌──────────────────────────────────────────────────────────┐
│              推荐的双工具分析流程                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ 1. IDA Free 初始加载                                     │
│    → 快速浏览导出表、导入表、字符串                       │
│    → 识别关键函数入口                                     │
│                                                          │
│ 2. Ghidra 深度分析                                       │
│    → 利用免费反编译器分析函数逻辑                         │
│    → 创建结构体、修改类型                                 │
│    → 运行自定义脚本批量分析                               │
│                                                          │
│ 3. IDA 交叉验证                                          │
│    → 对复杂函数用 IDA 云端反编译器二次确认                │
│    → 利用 IDA 更成熟的控制流分析                          │
│                                                          │
│ 4. 两者结果对比                                          │
│    → 反编译结果不一致处重点分析                           │
│    → 通常真相在两者之间                                   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 8. 动手实践

### 练习 1：Ghidra 初始化分析

```
1. 安装 Ghidra + JDK 17
2. 创建新项目 "KiriKiri2_RE"
3. 导入一个 KiriKiri2 插件 DLL (或任意 PE DLL)
4. 运行自动分析
5. 完成以下操作:
   a. 在 Symbol Tree 中找到 Exports
   b. 双击导出函数查看反编译
   c. 在 Defined Strings 中搜索关键字符串
   d. 使用 Ctrl+E 查看字符串的交叉引用
   e. 修改一个函数的签名和参数名
   f. 观察反编译输出的变化
```

<details>
<summary>操作要点</summary>

- Symbol Tree → Exports 可直接看到 V2Link/V2Unlink
- Window → Defined Strings 列出所有识别的字符串
- 修改函数签名：右键函数名 → Edit Function Signature
- 修改后反编译器自动重新生成输出
- 保存：Ghidra 自动保存分析结果到项目数据库

</details>

### 练习 2：创建并应用自定义结构体

```
目标: 创建 KiriKiri2 的 iTJSDispatch2 vtable 结构体

1. Window → Data Type Manager
2. 右键项目数据类型 → New → Structure
3. 名称: iTJSDispatch2_vtable
4. 添加成员 (基于 tp_stub.h 的虚函数表):
   +0x00: FuncPtr   AddRef
   +0x04: FuncPtr   Release
   +0x08: FuncPtr   FuncCall
   +0x0C: FuncPtr   FuncCallByNum
   +0x10: FuncPtr   PropGet
   +0x14: FuncPtr   PropGetByNum
   +0x18: FuncPtr   PropSet
   +0x1C: FuncPtr   PropSetByNum
   ... (继续添加其他虚函数)
5. 在反汇编中找到虚函数调用
6. 将 vtable 指针变量类型设为此结构体
7. 观察反编译输出中数字偏移变为方法名
```

### 练习 3：编写分析脚本

```python
# 练习: 编写脚本统计函数大小分布
# 在 Script Manager 中新建 Python 脚本

# @category Analysis
# @description 统计函数大小分布并找出最大的 10 个函数

fm = currentProgram.getFunctionManager()
functions = []

for func in fm.getFunctions(True):
    body = func.getBody()
    size = body.getNumAddresses()
    functions.append((func.getName(), func.getEntryPoint(), size))

# 按大小排序
functions.sort(key=lambda x: -x[2])

# 输出最大的 10 个函数
print("=== Top 10 Largest Functions ===")
for name, addr, size in functions[:10]:
    print("  {:40s} at {} ({} bytes)".format(name, addr, size))

# 大小分布统计
buckets = {"<32": 0, "32-128": 0, "128-512": 0,
           "512-2K": 0, ">2K": 0}
for _, _, size in functions:
    if size < 32: buckets["<32"] += 1
    elif size < 128: buckets["32-128"] += 1
    elif size < 512: buckets["128-512"] += 1
    elif size < 2048: buckets["512-2K"] += 1
    else: buckets[">2K"] += 1

print("\n=== Size Distribution ===")
for bucket, count in buckets.items():
    print("  {:10s}: {}".format(bucket, count))
```

<details>
<summary>预期输出</summary>

```
=== Top 10 Largest Functions ===
  FUN_10008000                             at 10008000 (4523 bytes)
  FUN_1000a200                             at 1000a200 (2156 bytes)
  ...

=== Size Distribution ===
  <32       : 89
  32-128    : 156
  128-512   : 98
  512-2K    : 34
  >2K       : 8
```

最大的函数通常是 TJS2 解释器循环或复杂的初始化逻辑。

</details>

### 练习 4：反编译结果对比

```
目标: 同一个函数用 IDA Free 和 Ghidra 分别反编译，对比差异

1. 选择一个中等复杂度的函数 (100-300 字节)
2. 在 IDA Free 中 F5 反编译
3. 在 Ghidra 中查看 Decompile 窗口
4. 对比以下方面:
   - 变量命名风格
   - 类型推断准确度
   - 控制流结构 (if/switch/loop)
   - 函数调用参数识别
5. 记录各自的优势和不足
```

<details>
<summary>对比要点</summary>

| 方面 | IDA Free (Hex-Rays 云端) | Ghidra |
|------|-------------------------|--------|
| 变量命名 | v1, v2, a1, a2 | uVar1, iVar2, param_1 |
| 类型推断 | 通常更准确 | 偏保守 (undefined4) |
| 控制流 | 更简洁 | 可能有更多 goto |
| 虚函数 | 直接显示间接调用 | 类似，可能多一层转换 |
| 注释 | 较少自动注释 | 更多自动注释 |

两者各有所长，建议都掌握。

</details>

---

## 9. 常见错误与解决方案

### 错误 1：Ghidra 启动时找不到 JDK

```
错误: "Could not find a supported JDK"

解决方案:
1. 确认 JDK 17+ 已安装（不是 JRE）
2. 设置环境变量:
   Windows: setx JAVA_HOME "C:\Program Files\Eclipse Adoptium\jdk-17"
   Linux: export JAVA_HOME=/usr/lib/jvm/java-17
3. 或编辑 <ghidra>/support/launch.properties:
   JAVA_HOME_OVERRIDE=C:/Program Files/Eclipse Adoptium/jdk-17
```

### 错误 2：反编译显示大量 undefined

```
反编译输出全是 undefined4、undefined8:

原因: Ghidra 无法推断变量类型

解决方案:
1. 确保运行了完整的自动分析 (Analysis → Auto Analyze)
2. 手动设置函数签名 (Edit Function Signature)
3. 导入 Windows 数据类型归档:
   File → Parse C Source → 选择 windows_vs12_32.h
4. 创建自定义结构体并应用
```

### 错误 3：分析大型文件时 OutOfMemoryError

```
错误: java.lang.OutOfMemoryError: Java heap space

解决方案:
编辑 <ghidra>/support/launch.properties:
# 增加最大堆内存 (默认 768M，建议 2-4G)
MAXMEM=4G

# 或启动时指定:
ghidraRun -Xmx4g
```

---

## 10. 对照项目源码

```
Ghidra 分析 KiriKiri2 时的关键参考:

文件: cpp/core/plugin/tp_stub.h
- 定义了所有 TJS 接口的虚函数表布局
- 用于在 Ghidra 中创建 vtable 结构体
- iTJSDispatch2 有 ~30 个虚函数

文件: cpp/core/plugin/ncbind/ncbind.hpp
- ncbind 模板绑定系统
- Ghidra 反编译的模板实例化对应这里的代码
- 理解 ncbind 模式有助于识别 TJS 方法注册

文件: cpp/core/tjs2/tjsVariant.h
- tTJSVariant 结构体定义
- 在 Ghidra Data Type Manager 中创建对应结构体
- 对分析 TJS 变量操作代码至关重要
```

---

## 本节小结

| 概念 | 要点 |
|------|------|
| Ghidra | 免费开源，NSA 开发，内置反编译器 |
| 安装 | 需要 JDK 17+，解压即用 |
| CodeBrowser | 主分析界面，左栏符号树 + 中间反汇编 + 右侧反编译 |
| 反编译 | 免费本地运行，质量约 IDA 的 90% |
| 类型修改 | 修改函数签名和变量类型可大幅改善输出 |
| 脚本 | Java / Python (Jython)，Script Manager 管理 |
| 与 IDA 互补 | IDA 做初步识别，Ghidra 做深度反编译分析 |

---

## 练习题与答案

### 题目 1

Ghidra 的反编译器相比 IDA Free 的云端反编译器有什么优势？

<details>
<summary>答案</summary>

1. **本地运行**：不需要网络连接，适合离线环境和敏感项目
2. **无使用限制**：不受云端反编译的频率和文件大小限制
3. **完全免费**：IDA Pro 的本地反编译器价格昂贵
4. **多架构支持**：Ghidra 支持 30+ 架构的反编译，IDA Free 仅支持 x86/x64
5. **脚本可访问**：可以在脚本中直接调用反编译 API 进行自动化分析

</details>

### 题目 2

在 Ghidra 中如何创建一个 C 结构体并应用到反编译窗口？

<details>
<summary>答案</summary>

1. 打开 Data Type Manager（Window → Data Type Manager）
2. 右键项目类型分类 → New → Structure
3. 输入结构体名称和大小
4. 逐个添加成员（指定偏移、类型、名称）
5. 保存结构体定义
6. 在 Decompile 窗口中，右键目标变量 → Retype Variable → 选择刚创建的结构体指针类型
7. Ghidra 自动用结构体成员名替代数字偏移

</details>

### 题目 3

编写一个 Ghidra Python 脚本，找出所有名称包含 "TJS" 的函数并列出它们的地址和大小。

<details>
<summary>答案</summary>

```python
# @category Search
# @description Find all functions with 'TJS' in name

fm = currentProgram.getFunctionManager()
tjs_funcs = []

for func in fm.getFunctions(True):
    name = func.getName()
    if 'TJS' in name or 'tjs' in name:
        size = func.getBody().getNumAddresses()
        tjs_funcs.append((name, func.getEntryPoint(), size))

tjs_funcs.sort(key=lambda x: str(x[1]))

print("=== TJS Functions ({} found) ===".format(len(tjs_funcs)))
for name, addr, size in tjs_funcs:
    print("  {} at {} ({} bytes)".format(name, addr, size))
```

</details>

---

## 下一步

掌握了 IDA 和 Ghidra 两大静态分析工具后，下一节 [x64dbg 动态调试](./03-x64dbg动态调试.md) 将介绍动态调试技术——在程序运行时设置断点、观察寄存器和内存变化、追踪函数调用，这是验证静态分析结论的关键手段。
