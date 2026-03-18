# IDA Free 与 IDA Pro

## 本节目标

- 了解 IDA 的版本差异（Free / Home / Pro）及选择建议
- 掌握 IDA 界面布局和核心导航操作
- 熟练使用反汇编视图、交叉引用（Xref）、函数列表
- 学会重命名、添加注释、修改函数类型等分析操作
- 入门 IDAPython 脚本自动化分析
- 在 KiriKiri2 DLL 上实际演练 IDA 分析流程

---

## 1. IDA 版本对比与获取

### 1.1 版本差异

```
┌─────────────────┬──────────────┬──────────────┬──────────────┐
│ 特性            │ IDA Free     │ IDA Home     │ IDA Pro      │
├─────────────────┼──────────────┼──────────────┼──────────────┤
│ 价格            │ 免费         │ ~$365/年     │ ~$1,400+/年  │
│ 反汇编器        │ x86/x64/ARM  │ 全架构       │ 全架构       │
│ 反编译器        │ x86/x64 云端 │ 本地 1 架构   │ 本地全架构   │
│ IDAPython       │ ✓           │ ✓           │ ✓           │
│ 插件/SDK        │ 有限         │ 完整         │ 完整         │
│ 调试器          │ 本地         │ 本地+远程    │ 本地+远程    │
│ 商业使用        │ ✗           │ ✗           │ ✓           │
│ 批处理模式      │ ✗           │ ✗           │ ✓           │
│ FLIRT 签名库    │ 基本         │ 完整         │ 完整         │
└─────────────────┴──────────────┴──────────────┴──────────────┘
```

### 1.2 下载与安装

```bash
# IDA Free 下载地址
# https://hex-rays.com/ida-free/

# 安装后默认路径 (Windows):
# C:\Program Files\IDA Free 8.x\

# 首次启动配置建议:
# Options → General → Disassembly
#   - Line prefixes: 勾选 (显示地址)
#   - Stack pointer: 勾选 (显示 ESP 变化)
#   - Auto comments: 勾选 (自动添加指令注释)
```

> **推荐**：学习阶段使用 IDA Free 完全够用。它支持 x86/x64 反汇编和云端反编译，足以分析 KiriKiri2 的 32 位 DLL 和 EXE。

---

## 2. 界面布局

### 2.1 主要窗口

IDA 启动并加载二进制文件后，默认界面布局：

```
┌──────────────────────────────────────────────────────────┐
│                     菜单栏 / 工具栏                       │
├──────────┬────────────────────────────────┬───────────────┤
│          │                                │               │
│ 函数列表  │     IDA View-A (反汇编视图)     │  交叉引用     │
│ Functions │     ← 核心工作区域              │  Xrefs        │
│          │                                │               │
│          │                                │               │
├──────────┤                                ├───────────────┤
│          │                                │               │
│ 结构体    │                                │  Hex View     │
│ Structs  │                                │  (十六进制)    │
│          │                                │               │
├──────────┴────────────────────────────────┴───────────────┤
│                    Output Window (日志)                     │
└──────────────────────────────────────────────────────────┘
```

### 2.2 核心快捷键

```
┌──────────┬──────────────────────────────────────────────┐
│ 快捷键   │ 功能                                         │
├──────────┼──────────────────────────────────────────────┤
│ G        │ Go to address — 跳转到指定地址               │
│ N        │ Rename — 重命名当前符号                      │
│ Y        │ Set type — 修改函数/变量类型                  │
│ ;        │ Add comment — 添加行尾注释                   │
│ Ins/:    │ Add anterior/posterior comment                │
│ X        │ Xrefs to — 查看谁引用了当前地址              │
│ Ctrl+X   │ Xrefs from — 查看当前地址引用了什么          │
│ Esc      │ Back — 返回上一个位置                        │
│ Ctrl+Enter│ Forward — 前进到下一个位置                  │
│ Space    │ 切换 Graph/Text 视图                         │
│ F5       │ Decompile — 反编译 (需要反编译器)            │
│ Tab      │ 在反汇编和反编译窗口间切换                    │
│ Ctrl+1   │ Quick view — 快速窗口选择                    │
│ Alt+T    │ Text search — 文本搜索                       │
│ Ctrl+F   │ Search in current view                       │
│ P        │ Create function — 创建函数                   │
│ D        │ Data — 将当前位置标记为数据                   │
│ C        │ Code — 将当前位置标记为代码                   │
│ U        │ Undefine — 取消定义                          │
└──────────┴──────────────────────────────────────────────┘
```

---

## 3. 加载二进制文件

### 3.1 打开 PE 文件

以分析 KiriKiri2 插件 DLL 为例：

```
1. File → Open → 选择 DLL 文件
2. 加载选项对话框:
   ┌────────────────────────────────────────────┐
   │ Processor type: MetaPC (x86/x64)          │
   │ File format: Portable Executable (PE)     │
   │ Loading segment: ☑ .text ☑ .data ☑ .rdata │
   │ Manual load: ☐ (通常不需要)                │
   │ Load resources: ☑                          │
   │ Make imports section: ☑                    │
   └────────────────────────────────────────────┘
3. 点击 OK → IDA 开始自动分析
4. 等待左下角状态栏显示 "AU: idle" (自动分析完成)
```

### 3.2 加载 PDB 调试符号

如果有对应的 PDB 文件（调试构建产物）：

```
方法 1: 自动加载
- 将 .pdb 文件放在与 .dll/.exe 同目录
- IDA 自动检测并加载符号

方法 2: 手动指定
- File → Load file → PDB file...
- 选择 .pdb 文件路径

方法 3: 从 Microsoft 符号服务器下载
- Options → General → PDB
- Symbol server: https://mssymbols.l1e.net
- (系统 DLL 如 kernel32.dll, ntdll.dll 的符号)
```

> **KiriKiri2 提示**：如果你从源码编译 KiriKiri2（Debug 配置），会生成 PDB 文件。加载 PDB 后 IDA 会显示完整的函数名、变量名、类型信息，极大降低分析难度。建议先用带符号的版本熟悉结构，再挑战无符号版本。

---

## 4. 反汇编导航

### 4.1 函数列表（Functions Window）

```
View → Open subviews → Functions (Shift+F3)

函数列表显示:
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ Function │ Segment  │ Start    │ Length   │ Flags    │
├──────────┼──────────┼──────────┼──────────┼──────────┤
│ _V2Link@4│ .text    │ 10001000 │ 0x45     │ S        │
│ _V2Unlink│ .text    │ 10001050 │ 0x20     │ S        │
│ sub_10001 │ .text   │ 10001080 │ 0x120    │          │
│ ...      │          │          │          │          │
└──────────┴──────────┴──────────┴──────────┴──────────┘

Flags 含义:
- S = stdcall  - T = thiscall  - (空) = cdecl
- R = 返回值  - L = library function  - B = BP-based frame

过滤: 在函数列表窗口按 Ctrl+F 搜索函数名
```

### 4.2 地址跳转

```
G 键 → 输入地址:
- 绝对地址: 0x10001234
- 函数名:   V2Link
- 标签:     loc_10001234
- 相对表达式: .text:10001000 + 0x50

; 常用跳转方式:
; 双击函数名 → 跳转到函数定义
; 双击 call 的目标 → 跳转到被调函数
; 双击 jmp 的目标 → 跳转到跳转目标
; Enter → 进入当前光标处的引用目标
; Esc → 返回上一个位置
```

### 4.3 图形视图与文本视图

```
Space 键切换:

文本视图 (Text View):
.text:10001000    push    ebp
.text:10001001    mov     ebp, esp
.text:10001003    sub     esp, 10h
.text:10001006    mov     eax, [ebp+8]
; 优点: 看到完整地址、字节编码、线性布局

图形视图 (Graph View):
┌─────────────┐
│ Block A     │ ──→ ┌─────────────┐
│ cmp / jz    │     │ Block B     │
└──────┬──────┘     │ (else 分支)  │
       │            └─────────────┘
       ↓
┌─────────────┐
│ Block C     │
│ (then 分支)  │
└─────────────┘
; 优点: 清晰展示控制流，适合理解函数逻辑
; 右键 → Graph options 调整布局
```

---

## 5. 交叉引用（Xrefs）

### 5.1 查看引用

交叉引用是逆向分析最重要的功能之一：

```
X 键 — 谁引用了当前地址 (Xrefs To):
  场景: 光标在某个函数名上，按 X
  结果: 列出所有调用该函数的位置

  例: 在 V2Link 上按 X:
  ┌──────────────────────────────────────────────┐
  │ Direction  │ Type  │ Address    │ Function    │
  ├────────────┼───────┼────────────┼─────────────┤
  │ Up         │ p     │ 10005678   │ DllMain     │
  │ Up         │ p     │ 10005A00   │ sub_10005A00│
  └────────────┴───────┴────────────┴─────────────┘
  Type: p=调用(procedure), o=偏移(offset), r=读(read), w=写(write)

Ctrl+X — 当前地址引用了什么 (Xrefs From):
  场景: 光标在 call 指令上
  结果: 列出目标函数
```

### 5.2 字符串引用

```
View → Open subviews → Strings (Shift+F12)

字符串窗口:
┌─────────────┬──────────┬──────────────────────────┐
│ Address     │ Length   │ String                    │
├─────────────┼──────────┼──────────────────────────┤
│ .rdata:1000A│ 10       │ "onCreate"               │
│ .rdata:1000B│ 12       │ "onDestroy"              │
│ .rdata:1000C│ 8        │ "V2Link"                 │
│ .rdata:1000D│ 22       │ "KiriKiri2 Plugin v1.0"  │
└─────────────┴──────────┴──────────────────────────┘

; 在字符串上按 X 查看引用 → 快速定位使用该字符串的代码
; 这是定位关键功能代码的最有效方法之一
```

**实战技巧**：分析 KiriKiri2 插件时，搜索字符串 `"V2Link"`、`"V2Unlink"`、TJS 方法名（如 `"open"`、`"close"`）可以快速定位插件的入口和功能注册代码。

### 5.3 导入/导出表

```
View → Open subviews → Imports / Exports

Exports（对 DLL 分析尤其重要）:
┌─────────────┬──────────┬──────────────────────┐
│ Address     │ Ordinal  │ Name                  │
├─────────────┼──────────┼──────────────────────┤
│ 10001000    │ 1        │ V2Link                │
│ 10001050    │ 2        │ V2Unlink              │
└─────────────┴──────────┴──────────────────────┘

Imports:
┌─────────────┬──────────────────────┬──────────────┐
│ Address     │ Name                  │ Module        │
├─────────────┼──────────────────────┼──────────────┤
│ .idata:1000 │ MessageBoxW           │ user32.dll   │
│ .idata:1000 │ CreateFileW           │ kernel32.dll │
│ .idata:1000 │ ??2@YAPAXI@Z         │ msvcrt.dll   │
│ (= operator new)                                   │
└─────────────┴──────────────────────┴──────────────┘

; 通过 Imports 了解程序使用了哪些 API
; 双击导入名 → 跳转到 IAT 条目 → X 查看谁调用了这个 API
```

---

## 6. 分析操作

### 6.1 重命名（N 键）

```
; 原始反汇编:
call    sub_10002000

; 分析后知道这是 InitPlugin 函数，按 N 重命名:
call    InitPlugin

; 重命名局部变量:
mov     eax, [ebp+var_4]     ; 按 N → 改名为 'count'
mov     eax, [ebp+count]     ; 更易读

; 命名建议:
; - 函数: 动词+名词 (InitPlugin, ParseScript, GetTextWidth)
; - 变量: 描述性名称 (line_count, file_handle, vtable_ptr)
; - 加前缀区分类型: p_buffer (指针), n_size (整数), sz_name (字符串)
```

### 6.2 修改类型（Y 键）

```
; 修改函数签名:
; 光标在函数开头，按 Y:
; 输入: int __stdcall V2Link(void *exporter)
; IDA 会据此调整反汇编和反编译输出

; 修改变量类型:
; 光标在变量上，按 Y:
; 输入: iTJSDispatch2 *
; IDA 会用结构体偏移替代数字偏移

; 修改前:
mov     eax, [ecx]
call    dword ptr [eax+44h]

; 添加类型信息后:
mov     eax, [ecx]              ; vtable
call    [eax+iTJSDispatch2.FuncCall]  ; 更有意义
```

### 6.3 创建结构体

```
View → Open subviews → Structures (Shift+F9)

; 按 Ins 创建新结构体:
; 例: 根据反汇编分析创建 KiriKiri2 的 tTJSVariant 结构

struct tTJSVariant {        ; size = 0x18
    void*   vtable;         ; +00h
    int     vt;             ; +04h (variant type)
    int     padding;        ; +08h
    union {                 ; +0Ch
        tjs_int64   Integer;
        double      Real;
        void*       Object;
        wchar_t*    String;
    };
};

; 创建后在反汇编中应用:
; 光标在 [esi+0Ch] 上 → 按 T → 选择 tTJSVariant → 选择成员
; [esi+0Ch] 变为 [esi+tTJSVariant.Integer]
```

### 6.4 添加注释

```
; 行尾注释 (;)
mov     ecx, [ebp+8]     ; this pointer from param1

; 函数注释 (在函数开头按 Ins 添加)
; ╔══════════════════════════════════════╗
; ║ InitPlugin - 初始化插件              ║
; ║ 参数: exporter - 函数导出接口        ║
; ║ 返回: HRESULT (S_OK=成功)           ║
; ╚══════════════════════════════════════╝

; 可重复注释 (在地址上右键 → Repeatable comment)
; 这类注释会在所有引用该地址的位置显示
```

---

## 7. 反编译器（Hex-Rays Decompiler）

### 7.1 基本使用

```
; 在函数上按 F5 → 打开反编译窗口

; 原始反汇编:
push    ebp
mov     ebp, esp
sub     esp, 8
mov     eax, [ebp+8]
mov     ecx, [ebp+0Ch]
add     eax, ecx
mov     [ebp-4], eax
...

; F5 反编译结果:
int __stdcall sub_10001000(int a1, int a2) {
    int v2 = a1 + a2;
    // ...
    return v2;
}
```

### 7.2 改进反编译输出

```
; 技巧 1: 先设置函数类型再反编译
; Y 键设置: int __thiscall CMyClass::Init(CMyClass *this, const wchar_t *name)
; 反编译结果会用正确的类型和参数名

; 技巧 2: 在反编译窗口中重命名变量
; 双击变量名 → 输入新名称
; 或 N 键重命名

; 技巧 3: 修改变量类型
; 右键变量 → Set type → 输入类型
; 例: 将 int 改为 iTJSDispatch2*

; 技巧 4: 在反编译和反汇编间同步
; Tab 键在两个窗口间切换，光标位置同步
; 这对理解反编译器的推断很有帮助

; 技巧 5: 内联和折叠
; 右键 → Hide/Show cast → 隐藏不必要的类型转换
; 右键 → Collapse → 折叠已分析完的代码块
```

### 7.3 反编译器的局限

```
反编译器不是万能的，常见问题:

1. 类型推断错误:
   // 反编译显示: v3 = *(_DWORD *)(a1 + 16);
   // 实际含义: v3 = this->member_at_0x10;
   // 解决: 创建结构体并应用

2. 控制流过于复杂:
   // goto 语句过多，难以阅读
   // 解决: 在关键位置添加注释，手动理解逻辑

3. 虚函数调用无法解析:
   // 显示: (*(void (__thiscall **)(int, int))(*(_DWORD *)v1 + 68))(v1, 0);
   // 解决: 设置 vtable 结构体类型

4. 内联汇编/SIMD 指令:
   // 反编译器可能显示 __asm 块或 intrinsic
   // 解决: 需要手动分析汇编

5. 异常处理代码:
   // SEH/C++ EH 的反编译可能不准确
   // 解决: 对比反汇编确认
```

---

## 8. IDAPython 脚本入门

### 8.1 脚本控制台

```python
# File → Script command (Shift+F2) 打开脚本输入窗口
# 或在 Output Window 底部的 Python 命令行直接输入

# 获取当前地址
import ida_kernwin
ea = ida_kernwin.get_screen_ea()
print(f"当前地址: {hex(ea)}")

# 获取函数名
import ida_funcs
func = ida_funcs.get_func(ea)
if func:
    print(f"函数起始: {hex(func.start_ea)}")
    print(f"函数结束: {hex(func.end_ea)}")
    print(f"函数大小: {func.end_ea - func.start_ea}")
```

### 8.2 常用 IDAPython 操作

```python
import idautils
import idc
import ida_name

# 1. 枚举所有函数
for func_ea in idautils.Functions():
    name = idc.get_func_name(func_ea)
    size = idc.get_func_attr(func_ea, idc.FUNCATTR_END) - func_ea
    print(f"{hex(func_ea)}: {name} ({size} bytes)")

# 2. 搜索特定模式（查找所有 V2Link 调用）
for func_ea in idautils.Functions():
    name = idc.get_func_name(func_ea)
    if "V2Link" in name:
        print(f"Found: {hex(func_ea)} - {name}")
        # 列出所有引用
        for ref in idautils.CodeRefsTo(func_ea, True):
            print(f"  Called from: {hex(ref)}")

# 3. 查找字符串引用
for s in idautils.Strings():
    if "KiriKiri" in str(s):
        print(f"String at {hex(s.ea)}: {str(s)}")
        for ref in idautils.DataRefsTo(s.ea):
            print(f"  Referenced at: {hex(ref)}")
```

### 8.3 批量分析脚本

```python
# 批量导出函数信息到文件
import json

def export_functions(output_path):
    """导出所有函数信息为 JSON"""
    functions = []
    for func_ea in idautils.Functions():
        func = ida_funcs.get_func(func_ea)
        if not func:
            continue
        info = {
            "address": hex(func_ea),
            "name": idc.get_func_name(func_ea),
            "size": func.end_ea - func_ea,
            "flags": func.flags,
            "xrefs_to": [hex(x) for x in idautils.CodeRefsTo(func_ea, True)],
        }
        functions.append(info)

    with open(output_path, 'w', encoding='utf-8') as f:
        json.dump(functions, f, indent=2, ensure_ascii=False)
    print(f"Exported {len(functions)} functions to {output_path}")

# 执行:
export_functions("C:/analysis/krkr2_functions.json")
```

```python
# 自动识别 KiriKiri2 TJS 方法注册模式
def find_tjs_method_registrations():
    """搜索 TJS 原生方法注册调用"""
    results = []
    # 搜索字符串中的 TJS 方法名
    for s in idautils.Strings():
        s_str = str(s)
        # TJS 方法通常是小写字母开头的标识符
        if s_str and s_str[0].islower() and len(s_str) < 50:
            refs = list(idautils.DataRefsTo(s.ea))
            if len(refs) > 0:
                # 检查引用位置附近是否有 ncbind 注册模式
                for ref in refs:
                    func = ida_funcs.get_func(ref)
                    if func:
                        func_name = idc.get_func_name(func.start_ea)
                        results.append({
                            "method": s_str,
                            "ref_addr": hex(ref),
                            "in_function": func_name
                        })
    return results

# 执行并输出
for r in find_tjs_method_registrations()[:20]:
    print(f"TJS Method '{r['method']}' referenced at "
          f"{r['ref_addr']} in {r['in_function']}")
```

---

## 9. 动手实践

### 练习 1：加载并导航 KiriKiri2 DLL

操作步骤：

```
1. 从 KiriKiri2 项目编译一个插件 DLL (Debug 模式)
   - 或使用任意 PE DLL 文件练习
2. 用 IDA Free 打开该 DLL
3. 等待自动分析完成 (AU: idle)
4. 完成以下操作:
   a. 打开 Exports 窗口，找到 V2Link 和 V2Unlink
   b. 双击 V2Link 跳转到代码
   c. 按 Space 切换图形视图
   d. 按 X 查看 V2Link 的交叉引用
   e. 按 F5 尝试反编译
   f. 打开 Strings 窗口 (Shift+F12) 搜索关键字符串
```

<details>
<summary>预期结果</summary>

- V2Link 应该是 stdcall 函数，1 个参数（iTVPFunctionExporter*）
- 函数内部调用 TVPImportFuncPtr 系列函数注册回调
- 字符串窗口中能找到插件名称和 TJS 方法名
- 反编译结果应该基本可读（可能需要手动修正类型）

</details>

### 练习 2：交叉引用追踪

```
目标: 从一个字符串出发，追踪到功能代码

1. Strings 窗口找到 "onCreate" 字符串
2. 按 X 查看引用 → 跳转到引用代码
3. 确定该代码所在的函数
4. 分析函数的调用约定和参数
5. 继续按 X 查看该函数被谁调用
6. 画出调用链:
   入口函数 → 注册函数 → 使用 "onCreate" 的具体逻辑
```

<details>
<summary>追踪思路</summary>

典型的 KiriKiri2 调用链：

```
V2Link()
  └→ RegisterTJSClass()
       └→ ncbind::DefineMethod("onCreate", ...)
            └→ handler 函数中引用 "onCreate" 字符串
```

通过交叉引用链，你可以从一个字符串逆向追踪到完整的注册和调用流程。

</details>

### 练习 3：函数类型修正

```
目标: 改善 IDA 的自动分析结果

1. 找到 V2Link 函数
2. 按 Y 修改类型为:
   HRESULT __stdcall V2Link(ITVPFunctionExporter *exporter)
3. 找到一个虚函数调用 (call [eax+N] 模式)
4. 尝试创建 vtable 结构体
5. 在虚函数调用处应用结构体偏移
6. 观察反编译输出的改善
```

### 练习 4：IDAPython 小脚本

```python
# 目标: 统计函数大小分布
# 在 IDA 的 Script command (Shift+F2) 中执行:

import idautils
import idc

# 按大小分类统计
size_buckets = {
    "tiny (<32B)": 0,
    "small (32-128B)": 0,
    "medium (128-512B)": 0,
    "large (512B-2KB)": 0,
    "huge (>2KB)": 0
}

for func_ea in idautils.Functions():
    end = idc.get_func_attr(func_ea, idc.FUNCATTR_END)
    size = end - func_ea
    if size < 32:
        size_buckets["tiny (<32B)"] += 1
    elif size < 128:
        size_buckets["small (32-128B)"] += 1
    elif size < 512:
        size_buckets["medium (128-512B)"] += 1
    elif size < 2048:
        size_buckets["large (512B-2KB)"] += 1
    else:
        size_buckets["huge (>2KB)"] += 1

print("=== Function Size Distribution ===")
total = sum(size_buckets.values())
for name, count in size_buckets.items():
    pct = count / total * 100 if total > 0 else 0
    bar = "█" * int(pct / 2)
    print(f"  {name:20s}: {count:4d} ({pct:5.1f}%) {bar}")
print(f"  Total functions: {total}")
```

<details>
<summary>预期输出示例</summary>

```
=== Function Size Distribution ===
  tiny (<32B)         :  145 (28.7%) ██████████████
  small (32-128B)     :  198 (39.2%) ███████████████████
  medium (128-512B)   :  112 (22.2%) ███████████
  large (512B-2KB)    :   38 ( 7.5%) ███
  huge (>2KB)         :   12 ( 2.4%) █
  Total functions: 505
```

大量 tiny 函数通常是编译器生成的存根（thunk）、简单 getter/setter 或引用计数操作。

</details>

---

## 10. 常见错误与解决方案

### 错误 1：IDA 未正确识别函数

```
症状: 代码显示为 "db 55h, 8Bh, ECh" 而非 "push ebp; mov ebp, esp"
原因: IDA 未将该区域识别为代码

解决方案:
1. 光标放在地址上
2. 按 C (Code) 将其标记为代码
3. 按 P (Create function) 让 IDA 创建函数

; 如果 P 失败（无法确定函数边界）:
; Edit → Functions → Create function... → 手动指定结束地址
```

### 错误 2：反编译结果类型错误

```c
// 反编译显示 (类型全是 int):
int __cdecl sub_401000(int a1, int a2) {
    int v2 = *(int *)(a1 + 4);
    ...
}

// 实际应该是:
tjs_error __thiscall CMyObj::Method(CMyObj *this, const wchar_t *name) {
    int refCount = this->refCount;
    ...
}

// 解决: Y 键设置正确的函数签名
// 创建结构体并应用到参数
```

### 错误 3：快捷键不生效

```
常见原因:
1. 焦点在错误的窗口 → 点击目标窗口获取焦点
2. 光标在数据区域 → F5 反编译需要在代码区域
3. IDA 还在分析中 → 等待 "AU: idle"
4. Free 版限制 → 某些功能仅 Pro 版可用
```

---

## 11. 对照项目源码

```
IDA 分析 KiriKiri2 时的重要参考:

文件: cpp/core/plugin/tp_stub.h
- iTVPFunctionExporter 接口定义
- V2Link/V2Unlink 的参数类型
- 在 IDA 中创建对应结构体可大幅改善分析

文件: cpp/core/tjs2/tjsTypes.h
- tjs_int, tjs_uint, tjs_char 等类型定义
- 用于在 IDA 中设置正确的类型别名

文件: cpp/core/plugin/ncbind/
- ncbind 绑定系统的模板展开后的代码
- IDA 中看到的复杂模板实例化对应这里的宏
```

---

## 本节小结

| 概念 | 要点 |
|------|------|
| IDA Free | 免费，支持 x86/x64，云端反编译，足够学习使用 |
| 核心快捷键 | G(跳转), N(重命名), X(引用), F5(反编译), Y(类型) |
| 交叉引用 | 最重要的分析功能，从字符串/API追踪到功能代码 |
| 反编译 | 需要手动修正类型以获得更好的输出 |
| IDAPython | 自动化批量分析，脚本化重复操作 |
| 结构体 | 创建结构体并应用到变量上，替代数字偏移 |

---

## 练习题与答案

### 题目 1

在 IDA 中如何快速定位 KiriKiri2 插件的入口函数？说出至少两种方法。

<details>
<summary>答案</summary>

1. **Exports 窗口**：View → Open subviews → Exports，直接找到 V2Link 和 V2Unlink 导出函数
2. **Strings 搜索**：Shift+F12 打开 Strings 窗口，搜索 "V2Link" 字符串
3. **Functions 搜索**：Shift+F3 打开函数列表，Ctrl+F 搜索 "V2Link"
4. **地址跳转**：如果知道导出表中的 RVA，按 G 直接输入地址

</details>

### 题目 2

为什么在 IDA 反编译窗口看到 `(*(void (__thiscall **)(int, int))(*(_DWORD *)a1 + 68))(a1, 0)` 这样的代码？如何改善？

<details>
<summary>答案</summary>

这是一个虚函数调用，IDA 无法自动解析 vtable 结构。

改善步骤：
1. 分析 `a1` 的类型（从调用上下文推断是哪个类的对象）
2. 创建 vtable 结构体，填入各虚函数偏移
3. 对 `a1` 参数应用正确的类型（如 `iTJSDispatch2*`）
4. IDA 会将 `+68` 替换为结构体成员名

改善后：`a1->vtable->FuncCall(a1, 0)`

</details>

### 题目 3

编写一个 IDAPython 脚本，找出所有调用了 `MessageBoxW` 的函数。

<details>
<summary>答案</summary>

```python
import idautils
import idc

# 查找 MessageBoxW 的导入地址
msgbox_ea = idc.get_name_ea_simple("MessageBoxW")
if msgbox_ea == idc.BADADDR:
    # 尝试带前缀
    msgbox_ea = idc.get_name_ea_simple("_MessageBoxW@16")

if msgbox_ea != idc.BADADDR:
    print(f"MessageBoxW at {hex(msgbox_ea)}")
    for ref in idautils.CodeRefsTo(msgbox_ea, True):
        func = idautils.Functions(ref).__next__() if ref else None
        func_name = idc.get_func_name(ref)
        print(f"  Called from {hex(ref)} in {func_name}")
else:
    print("MessageBoxW not found in imports")
```

</details>

---

## 下一步

掌握了 IDA 的基本使用后，下一节 [Ghidra 详解](./02-Ghidra详解.md) 将介绍免费开源的 Ghidra 反编译工具——作为 IDA 的强力补充，Ghidra 提供了完全免费的本地反编译器和灵活的脚本系统。
