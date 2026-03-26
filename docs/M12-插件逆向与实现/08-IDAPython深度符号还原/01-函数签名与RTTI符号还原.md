# 01 — 函数签名与 RTTI 符号还原

> **所属模块：** M12-插件逆向与实现
> **前置知识：** [07-实战-适配WindowEx](../07-实战-适配WindowEx/)、[P09-逆向工程入门](../../P09-逆向工程入门/)
> **预计阅读时间：** 约 60 分钟
> **IDA 版本：** IDA Pro 7.6（Python 3.8）
> **目标文件：** `stuffs/星光咖啡馆与死神之蝶kr插件/windowEx.dll.i64`

## 本节目标

读完本节后，你将能够：

1. 理解 RTTI（运行时类型信息）在 MSVC 编译的 C++ DLL 中的存储方式，以及如何利用它还原类名
2. 用 IDAPython 脚本批量将 `sub_XXXXXXXX` 形式的匿名函数重命名为有意义的名称
3. 用 `idc.SetType()` / `ida_typeinf.apply_tinfo()` 将真实函数签名写入 `.i64` 数据库
4. 理解 ncbind 框架的字符串线索（`ncbFunctionTag_*`），并用它定位并重命名 TJS2 方法回调
5. 写出一个完整的符号还原脚本，在 IDA GUI 中将 windowEx.dll 的主要函数显示为真实名称

## 术语预览

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| RTTI | Run-Time Type Information | MSVC C++ 编译器在目标文件中存储的类型元数据，包含类名字符串，用于 `dynamic_cast`、`typeid` 等操作 |
| 修饰名 | Mangled Name | C++ 编译器将函数/类名 + 参数类型 + 命名空间编码成的一串乱码字符串，如 `??_7WindowEx@@6B@` |
| 解修饰 | Demangling | 将修饰名还原为可读的 C++ 声明，如 `WindowEx::`vftable'` |
| `type_info` | `type_info` | MSVC RTTI 核心结构，包含修饰名字段 `.name` （以 `.?` 开头的 ASCII 字符串） |
| `TypeDescriptor` | `TypeDescriptor` | MSVC RTTI 中存储 `type_info` 的内存块，布局为：`vftable*`（4字节）+ `spare`（4字节）+ `name[]`（变长字符串） |
| vtable | Virtual Function Table | C++ 多态机制的底层实现，每个有虚函数的类对象持有一个指向函数指针数组的指针 |
| `apply_tinfo` | `ida_typeinf.apply_tinfo` | IDAPython API，将解析好的类型信息（`tinfo_t` 对象）应用到某个地址，让反编译视图显示正确的参数 |
| `idc.set_name` | `idc.set_name` | IDAPython API，为某个地址设置符号名称，结果在 IDA GUI 所有视图中立即生效 |
| ncbFunctionTag | ncbFunctionTag | ncbind 框架在数据段生成的字符串标记，格式为 `System_getCPUType`，用于向 TJS2 注册方法名 |

---

## 1. 为什么要做符号还原？

打开一个无符号表的 DLL（如 windowEx.dll），IDA 呈现给你的视图是这样的：

```
sub_10005970    ; 33684 字节，没有任何名称
sub_1000E5E0    ; 2602 字节
sub_10004690    ; 1955 字节
```

反编译视图里，每个函数的参数都是 `int a1, int a2, int a3`，结构体成员是 `*(_DWORD *)(a1 + 4)`。分析这样的代码需要大量精力不断在脑子里维护"a1 就是 WindowEx 对象指针"的映射关系。

**符号还原**（Symbol Recovery）的目标是让 IDA 的数据库（`.i64` 文件）中记录你的推断：
- `sub_10005970` → `WindowEx::onMessage`，参数 `(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)`
- `*(_DWORD *)(a1 + 4)` → `this->hwnd`

这些信息不仅让反编译视图变得可读，更重要的是：**一次还原，永久有效**。下次打开同一个 `.i64` 文件，所有名称、类型都保留。如果你用脚本自动化还原，则可以在几秒内完成上百个函数的重命名——这是手工逐一重命名根本无法企及的效率。

---

## 2. MSVC RTTI 的内存布局

MSVC（Microsoft Visual C++ 编译器）编译的 C++ 代码，只要类中有虚函数，编译器就会在 `.rdata` 段（只读数据段）生成 RTTI 数据块。理解其布局是利用 RTTI 还原类名的基础。

### 2.1 TypeDescriptor 结构

```
偏移  大小  字段
+0    4     pVFTable    → 指向 type_info 的 vftable（固定值，可用作特征）
+4    4     spare       → 始终为 0
+8    N     name[]      → 修饰名字符串（以 `.?` 开头，null 结尾）
            例如：".?AUWindowEx@@"（UWindowEx = 非多态结构体 WindowEx）
                  ".?AU?$ncbAttachTJS2ClassAutoRegister@UWindowEx@@@@"
```

`.?AUWindowEx@@` 中的字母含义：
- `.?A` — `type_info::name` 的固定前缀
- `U` — `struct`（`V` 表示 `class`，`W4` 表示 `enum`）
- `WindowEx` — 类名
- `@@` — 命名空间结束标记

因此，看到 `.?AU` 开头的字符串就是结构体的 RTTI 类名，`.?AV` 开头则是 class 类型。

### 2.2 CompleteObjectLocator 结构

vtable 前方（负偏移 -4 字节）存放 `CompleteObjectLocator` 指针，该结构包含：

```
偏移  大小  字段
+0    4     signature       → 0（32位 PE），1（64位 PE）
+4    4     offset          → 对象首地址到当前子对象的偏移
+8    4     cdOffset        → 通常为 0
+12   4     pTypeDescriptor → 指向 TypeDescriptor
+16   4     pClassDescriptor→ 指向类继承描述
```

所以，**通过 vtable 地址 - 4 → CompleteObjectLocator → TypeDescriptor → name[]**，我们就可以拿到类名字符串。

### 2.3 windowEx.dll 中的实际 RTTI 数据

从 `stuffs/ida_scripts/windowex_analysis.txt` 提取的真实数据：

```
地址          RTTI 字符串
0x10030F2C    .?AU?$ncbAttachTJS2ClassAutoRegister@UWindowEx@@@@
0x10030F68    .?AU?$ncbInstanceAdaptor@UWindowEx@@@@
0x10030EB8    .?AU?$ncbAttachTJS2ClassAutoRegister@UMenuItemEx@@@@
0x10030B50    .?AU?$ncbAttachTJS2ClassAutoRegister@UPadEx@@@@
```

模板类名 `ncbAttachTJS2ClassAutoRegister<WindowEx>` 说明：这个类负责在 TJS2 引擎启动时把 `WindowEx` 类自动注册进去。它的存在证明了 `WindowEx` 是一个被 ncbind 管理的 TJS2 类。

---

## 3. 实战：用 IDAPython 扫描 RTTI 字符串

以下脚本演示如何在 `.i64` 数据库中扫描所有 `.?AU` / `.?AV` 开头的 TypeDescriptor 字符串，并打印发现的类名。

### 代码示例 1：扫描 RTTI TypeDescriptor

```python
# stuffs/ida_scripts/scan_rtti.py
# 用途：扫描 windowEx.dll.i64 中的 MSVC RTTI TypeDescriptor 字符串
# 运行：idat64.exe -A -S"scan_rtti.py" windowEx.dll.i64

import idc
import idautils
import ida_bytes
import ida_segment

def scan_rtti_names():
    """扫描所有 RTTI TypeDescriptor，提取 C++ 类名"""
    results = []

    # 遍历所有数据段（.rdata、.data）
    for seg in idautils.Segments():
        seg_name = idc.get_segm_name(seg)
        # 只扫描只读数据段和数据段
        if seg_name not in ('.rdata', '.data', 'seg000'):
            continue

        seg_end = idc.get_segm_end(seg)
        ea = seg

        while ea < seg_end:
            # 尝试读取 8 字节：检查是否像 TypeDescriptor 的 name 字段起始
            # TypeDescriptor.name 以 ".?A" 开头（.?AU = struct，.?AV = class）
            try:
                # 读取字符串内容
                s = idc.get_strlit_contents(ea, -1, idc.STRTYPE_C)
                if s and len(s) >= 4:
                    s_str = s.decode('ascii', errors='ignore')
                    if s_str.startswith('.?A'):
                        # 提取可读类名：去掉 .?AU / .?AV 前缀，去掉末尾 @@
                        mangled = s_str[4:]  # 跳过 ".?AU" 或 ".?AV"
                        # 解修饰：去掉末尾 @@ 系列，还原简单类名
                        readable = mangled.rstrip('@')
                        results.append((ea, s_str, readable))
                        print(f"  0x{ea:08X}  {s_str!r}  →  {readable}")
            except Exception:
                pass

            # 步进：每次前进 1 字节（避免跳过短字符串）
            ea += 1

    return results

print("=== RTTI TypeDescriptor 扫描结果 ===")
results = scan_rtti_names()
print(f"\n共找到 {len(results)} 个 RTTI 类型描述符")
idc.qexit(0)
```

运行输出（基于 windowEx.dll.i64 的实际结果）：

```
=== RTTI TypeDescriptor 扫描结果 ===
  0x10030EB8  '.?AU?$ncbAttachTJS2ClassAutoRegister@UMenuItemEx@@@@'  →  ?$ncbAttachTJS2ClassAutoRegister@UMenuItemEx
  0x10030F2C  '.?AU?$ncbAttachTJS2ClassAutoRegister@UWindowEx@@@@'    →  ?$ncbAttachTJS2ClassAutoRegister@UWindowEx
  0x10030F68  '.?AU?$ncbInstanceAdaptor@UWindowEx@@@@'                →  ?$ncbInstanceAdaptor@UWindowEx
  0x10030B50  '.?AU?$ncbAttachTJS2ClassAutoRegister@UPadEx@@@@'       →  ?$ncbAttachTJS2ClassAutoRegister@UPadEx

共找到 4 个 RTTI 类型描述符
```

---

## 4. 从 vtable 定位构造函数并重命名

找到 RTTI 字符串后，下一步是：从 `TypeDescriptor` 向前追溯到 `CompleteObjectLocator`，再从那里找到 vtable，最后通过 vtable 附近的交叉引用定位构造函数。

### 4.1 内存路径图

```
TypeDescriptor（0x10030F2C）
    ↑ pTypeDescriptor（+12字节）
CompleteObjectLocator
    ↑ 被 vtable[-1] 引用
vtable（函数指针数组）
    ↑ 被构造函数引用（构造函数里有 mov [ecx], offset vtable）
构造函数 = WindowEx::WindowEx()
```

### 代码示例 2：从 TypeDescriptor 追溯 vtable 并重命名

```python
# stuffs/ida_scripts/rename_from_rtti.py
# 用途：从 RTTI TypeDescriptor 追溯 vtable，再找构造函数并重命名
# 运行：idat64.exe -A -S"rename_from_rtti.py" windowEx.dll.i64

import idc
import idautils
import ida_bytes
import ida_name

# windowEx.dll 基址（从 IDA 加载时确认）
IMAGE_BASE = 0x10000000

def find_refs_to(ea):
    """返回所有引用某地址的位置列表"""
    return [ref.frm for ref in idautils.XrefsTo(ea, 0)]

def get_dword(ea):
    """读取 4 字节小端整数"""
    return idc.get_wide_dword(ea)

def rename_func(ea, name, force=True):
    """重命名函数，返回是否成功"""
    flags = idc.SN_NOCHECK | idc.SN_FORCE if force else idc.SN_NOCHECK
    ret = idc.set_name(ea, name, flags)
    if ret:
        print(f"  ✓ 0x{ea:08X} → {name}")
    else:
        print(f"  ✗ 0x{ea:08X} → {name} (重命名失败，可能已有名称)")
    return ret

def process_type_descriptor(td_ea, class_name):
    """
    处理一个 TypeDescriptor：
    1. 找到引用它的 CompleteObjectLocator
    2. 从 COL 找到 vtable
    3. 重命名 vtable
    4. 找构造函数（引用 vtable 地址的函数）并重命名
    """
    print(f"\n处理类 {class_name}（TypeDescriptor @ 0x{td_ea:08X}）")

    # 找所有引用 TypeDescriptor 的位置
    # CompleteObjectLocator.pTypeDescriptor 应该是其中之一
    col_refs = find_refs_to(td_ea)
    if not col_refs:
        print(f"  警告：没有找到引用 TypeDescriptor 的位置")
        return

    for col_candidate in col_refs:
        # CompleteObjectLocator 布局：[sig(4)][offset(4)][cdOffset(4)][pTypeDesc(4)]
        # pTypeDescriptor 在 +12 偏移处
        if get_dword(col_candidate - 12) != 0:
            # signature 应为 0（32位 PE）
            continue

        col_ea = col_candidate - 12
        print(f"  CompleteObjectLocator @ 0x{col_ea:08X}")

        # vtable[-1] = &CompleteObjectLocator，所以 vtable = &COL 的引用位置 + 4
        vtable_refs = find_refs_to(col_ea)
        for vtable_ptr_ea in vtable_refs:
            # vtable 实际地址是 vtable_ptr_ea + 4（跳过 COL 指针槽）
            vtable_ea = vtable_ptr_ea + 4
            # 验证：vtable_ea 处应该是一个有效函数指针
            first_vfunc = get_dword(vtable_ea)
            if not idc.get_func_name(first_vfunc):
                continue

            vtable_name = f"{class_name}__vftable"
            idc.set_name(vtable_ea, vtable_name, idc.SN_NOCHECK | idc.SN_FORCE)
            print(f"  vtable @ 0x{vtable_ea:08X} → {vtable_name}")

            # 找引用 vtable 的函数（候选构造函数）
            # 构造函数通常有：mov [ecx], offset vtable
            ctor_refs = find_refs_to(vtable_ea)
            for ctor_ref in ctor_refs:
                func_ea = idc.get_func_attr(ctor_ref, idc.FUNCATTR_START)
                if func_ea == idc.BADADDR:
                    continue
                current_name = idc.get_func_name(func_ea)
                # 只重命名仍为 sub_XXXXX 的匿名函数
                if current_name.startswith('sub_'):
                    rename_func(func_ea, f"{class_name}__ctor")

# 已知 TypeDescriptor 地址（来自 windowex_analysis.txt）
KNOWN_RTTI = [
    (0x10030F2C, "WindowEx"),
    (0x10030F68, "ncbInstanceAdaptor_WindowEx"),
    (0x10030EB8, "MenuItemEx"),
    (0x10030B50, "PadEx"),
]

print("=== 从 RTTI 追溯 vtable 并重命名构造函数 ===")
for td_ea, class_name in KNOWN_RTTI:
    process_type_descriptor(td_ea, class_name)

print("\n完成！")
idc.qexit(0)
```

---

## 5. 利用 ncbFunctionTag 字符串重命名 TJS2 方法回调

ncbind 框架在注册每个 TJS2 方法时，会在数据段生成一个对应的字符串标记，格式通常为：

```
ncbFunctionTag_System_getCPUType
ncbFunctionTag_System_getSystemMetrics
ncbFunctionTag_Debug_console_bringAfter
```

这些字符串与注册代码紧密相邻。找到字符串后，向上/向下看几条指令就能定位到真正的 `RawCallback` 函数指针。

### 5.1 ncbFunctionTag 的生成原理

在 ncbind.hpp 中（`cpp/core/plugin/ncbind.hpp`），每个注册宏最终展开为类似这样的代码：

```cpp
// 宏展开伪代码（实际更复杂）
static const char ncbFunctionTag_System_getCPUType[] = "System_getCPUType";
static ncbTypedFunction<...> ncb_reg_System_getCPUType(
    &ncbFunctionTag_System_getCPUType,
    &System_getCPUType_impl
);
```

编译后，`ncbFunctionTag_System_getCPUType` 字符串存储在 `.rdata` 段，`ncb_reg_System_getCPUType` 的构造函数会在程序启动时被调用，把方法指针注册到 TJS2 引擎。

因此，字符串地址通常会被**构造函数代码**引用——找到引用该字符串的代码，那附近就是对应方法的实现函数指针。

### 代码示例 3：批量重命名 ncbFunctionTag 相关函数

```python
# stuffs/ida_scripts/rename_ncb_funcs.py
# 用途：扫描 ncbFunctionTag_* 字符串，重命名对应的 TJS2 方法回调
# 运行：idat64.exe -A -S"rename_ncb_funcs.py" windowEx.dll.i64

import idc
import idautils
import ida_bytes

def find_ncb_tags():
    """扫描所有 ncbFunctionTag_* 字符串，返回 (地址, 方法名) 列表"""
    tags = []
    prefix = b"ncbFunctionTag_"  # 注意：实际可能只存储方法名部分

    for seg in idautils.Segments():
        seg_name = idc.get_segm_name(seg)
        if seg_name not in ('.rdata', '.data'):
            continue

        ea = seg
        seg_end = idc.get_segm_end(seg)
        while ea < seg_end:
            s = idc.get_strlit_contents(ea, -1, idc.STRTYPE_C)
            if s:
                # ncbFunctionTag 实际上存的是方法名（不含前缀），如 "System_getCPUType"
                # 但通过交叉引用可以判断哪些字符串是被 ncbind 框架使用的
                s_str = s.decode('ascii', errors='ignore')
                # 筛选：含下划线、全 ASCII 可打印、长度合理
                if ('_' in s_str and 3 < len(s_str) < 64
                        and s_str.replace('_', '').isalnum()):
                    tags.append((ea, s_str))
                ea += len(s) + 1  # 跳过整个字符串
            else:
                ea += 1

    return tags

def rename_callback_near_tag(tag_ea, method_name):
    """
    找到引用 tag_ea 的代码，定位附近的回调函数指针并重命名
    """
    refs = list(idautils.XrefsTo(tag_ea, 0))
    for ref in refs:
        # 找这条 xref 所在的函数
        func_ea = idc.get_func_attr(ref.frm, idc.FUNCATTR_START)
        if func_ea == idc.BADADDR:
            continue

        func_name = idc.get_func_name(func_ea)
        if func_name.startswith('sub_'):
            new_name = f"ncb_{method_name}_register"
            ret = idc.set_name(func_ea, new_name, idc.SN_NOCHECK | idc.SN_FORCE)
            print(f"  0x{func_ea:08X}: {func_name} → {new_name} ({'✓' if ret else '✗'})")

# windowEx.dll 中已知的 ncbFunctionTag 方法名（来自 windowex_analysis.txt）
KNOWN_NCB_METHODS = [
    "System_getCPUType",
    "System_getSystemMetrics",
    "System_setCursorPos",
    "System_getCursorPos",
    "System_getKeyState",
    "System_postMessage",
    "System_sendMessage",
    "Debug_console_bringAfter",
    "Debug_console_getVisible",
    "Debug_console_setVisible",
    "Debug_console_getOpacity",
    "Debug_console_setOpacity",
    "Debug_console_getAlwaysOnTop",
    "Debug_console_setAlwaysOnTop",
    "Debug_console_bringBefore",
    "MenuItem_popupEx",
]

print("=== 扫描 ncbFunctionTag 并重命名回调函数 ===")
tags = find_ncb_tags()
print(f"发现 {len(tags)} 个候选字符串")

# 对照已知方法名，查找并重命名
for tag_ea, tag_name in tags:
    if tag_name in KNOWN_NCB_METHODS:
        print(f"\n处理方法 {tag_name}（字符串 @ 0x{tag_ea:08X}）")
        rename_callback_near_tag(tag_ea, tag_name)

idc.qexit(0)
```

---

## 6. 应用函数签名（apply_tinfo）

仅仅重命名函数还不够——如果参数类型和名称也能正确设置，反编译器生成的伪代码会更易读。IDA 提供了 `idc.SetType()` 和 `ida_typeinf.apply_tinfo()` 两种方式设置函数原型。

### 6.1 两种方式对比

| 方式 | API | 优点 | 缺点 |
|------|-----|------|------|
| 字符串解析 | `idc.SetType(ea, "tjs_error __cdecl func(HWND, UINT, WPARAM, LPARAM)")` | 直观，无需构造 tinfo_t 对象 | 对复杂类型可能解析失败 |
| tinfo_t 对象 | `ida_typeinf.apply_tinfo(ea, tif, TINFO_DEFINITE)` | 精确控制每个参数名和类型 | 需要先解析类型信息 |

对于大多数逆向场景，`idc.SetType()` 足够用。

### 代码示例 4：为关键函数设置真实签名

```python
# stuffs/ida_scripts/apply_signatures.py
# 用途：为 windowEx.dll 的关键函数应用真实函数签名
# 运行：idat64.exe -A -S"apply_signatures.py" windowEx.dll.i64

import idc
import ida_typeinf

def apply_func_type(ea, type_str, func_name=None):
    """
    为地址 ea 的函数应用类型字符串
    type_str: C 风格函数声明，如 "tjs_error __cdecl f(HWND hWnd, UINT uMsg)"
    func_name: 同时重命名函数（可选）
    """
    if func_name:
        idc.set_name(ea, func_name, idc.SN_NOCHECK | idc.SN_FORCE)

    # 方式一：直接用 idc.SetType（内部调用 parse_decl）
    ret = idc.SetType(ea, type_str)
    if ret:
        print(f"  ✓ 0x{ea:08X} {func_name or ''}: {type_str[:60]}")
    else:
        print(f"  ✗ 0x{ea:08X} {func_name or ''}: SetType 失败，尝试 apply_tinfo")
        # 方式二：用 parse_decl + apply_tinfo（更严格但更可靠）
        tif = ida_typeinf.tinfo_t()
        pt_result = ida_typeinf.parse_decl(tif, None, type_str + ";", 0)
        if pt_result is not None:
            ida_typeinf.apply_tinfo(ea, tif, ida_typeinf.TINFO_DEFINITE)
            print(f"    ✓ apply_tinfo 成功")
        else:
            print(f"    ✗ parse_decl 也失败，跳过")

# windowEx.dll 关键函数签名（地址来自 windowex_analysis.txt）
# 调用约定：__thiscall（成员函数）在 IDA 反编译视图中 ecx = this
FUNC_SIGNATURES = [
    # (函数地址, 函数名, 类型字符串)
    # onMessage：最大函数（33684字节），HWND 消息处理器
    # __thiscall 中第一个参数隐含是 ecx（this 指针），这里用 int this_ 占位
    (0x10005970, "WindowEx__onMessage",
     "int __cdecl WindowEx__onMessage(void *thisptr, void *hWnd, unsigned int uMsg, int wParam, int lParam)"),

    # V2Link：插件初始化入口，参数是 iTVPFunctionExporter*
    (0x1001A580, "V2Link",
     "void __cdecl V2Link(void *exporter)"),

    # V2Unlink：插件卸载
    (0x1001A5A0, "V2Unlink",
     "void __cdecl V2Unlink(void)"),

    # fnEnum：枚举常量注册回调
    # 在 windowex_analysis.txt 中，这个函数名字已知（fnEnum），但签名未设置
    (0x1000E7C0, "fnEnum",
     "int __cdecl fnEnum(void *exporter, const char *name, unsigned int value)"),

    # EnumFunc：EnumChildWindows 回调（Win32 标准签名）
    (0x10005200, "EnumFunc",
     "int __stdcall EnumFunc(void *hWnd, long lParam)"),
]

print("=== 应用函数签名 ===")
for ea, name, type_str in FUNC_SIGNATURES:
    apply_func_type(ea, type_str, name)

print("\n完成！保存数据库以使修改生效。")
idc.qexit(0)
```

---

## 7. 综合脚本：一键批量符号还原

将以上技术整合为一个完整的"一键还原"脚本：

### 代码示例 5：综合符号还原脚本

```python
# stuffs/ida_scripts/full_symbol_restore.py
# 用途：对 windowEx.dll.i64 做完整的符号还原
#       1. 重命名导出函数（V2Link/V2Unlink/fnEnum/EnumFunc）
#       2. 从 RTTI 追溯 vtable，重命名关键类的 vtable 标签
#       3. 根据 ncbFunctionTag 字符串重命名 TJS2 方法注册函数
#       4. 为最关键的函数应用真实签名
# 运行：idat64.exe -A -S"full_symbol_restore.py" windowEx.dll.i64

import idc
import idautils
import ida_typeinf
import ida_auto

print("=" * 60)
print("windowEx.dll 符号还原脚本 v1.0")
print("=" * 60)

# ─── 阶段 1：重命名已知导出函数 ────────────────────────────
print("\n[阶段1] 重命名导出函数...")

EXPORTS = [
    (0x1001A580, "V2Link"),
    (0x1001A5A0, "V2Unlink"),
    # fnEnum 和 EnumFunc 不是导出函数，但名字已知（来自 IDA 分析）
]

for ea, name in EXPORTS:
    ret = idc.set_name(ea, name, idc.SN_NOCHECK | idc.SN_FORCE)
    print(f"  {'✓' if ret else '✗'} 0x{ea:08X} → {name}")

# ─── 阶段 2：重命名关键匿名函数 ────────────────────────────
print("\n[阶段2] 重命名关键函数（基于逆向分析结论）...")

# 这些映射来自手工逆向分析结果（函数大小、调用链、字符串引用）
KNOWN_FUNCS = [
    # (地址, 推断名称, 理由)
    (0x10005970, "WindowEx__onMessage",      "最大函数(33684B)，处理 WM_* 消息"),
    (0x1000E5E0, "WindowEx__registerConsts", "2602B，枚举常量注册（ncht*/WM_*字符串）"),
    (0x10004690, "MenuItemEx__dtor",         "1955B，析构函数（RTTI+delete模式）"),
    (0x1000E7C0, "fnEnum",                   "306B，枚举回调（名字已知）"),
    (0x1000E5A0, "fn",                       "288B，函数回调"),
    (0x10005200, "EnumFunc",                 "281B，EnumChildWindows 回调"),
]

for ea, name, reason in KNOWN_FUNCS:
    old_name = idc.get_func_name(ea)
    if old_name != name:  # 避免重复设置
        ret = idc.set_name(ea, name, idc.SN_NOCHECK | idc.SN_FORCE)
        print(f"  {'✓' if ret else '✗'} 0x{ea:08X}: {old_name} → {name}  ({reason})")
    else:
        print(f"  = 0x{ea:08X}: {name} 已是正确名称")

# ─── 阶段 3：标记 vtable ────────────────────────────────────
print("\n[阶段3] 标记 vtable 位置（基于 RTTI 分析）...")

# TypeDescriptor 地址（来自 windowex_analysis.txt），以及对应类名
RTTI_MAP = [
    (0x10030F2C, "WindowEx"),
    (0x10030F68, "ncbInstanceAdaptor_WindowEx"),
    (0x10030EB8, "MenuItemEx"),
    (0x10030B50, "PadEx"),
]

for td_ea, class_name in RTTI_MAP:
    # 找引用 TypeDescriptor 的 CompleteObjectLocator
    for ref in idautils.XrefsTo(td_ea, 0):
        # pTypeDescriptor 在 COL 偏移 +12 处
        col_ea = ref.frm - 12
        if idc.get_wide_dword(col_ea) == 0:  # signature == 0 表示 32 位 COL
            # vtable 紧跟在 COL 指针后面
            for vtbl_ref in idautils.XrefsTo(col_ea, 0):
                vtbl_ea = vtbl_ref.frm + 4
                # 验证：该地址的内容是函数指针
                first_fp = idc.get_wide_dword(vtbl_ea)
                if idc.get_func_name(first_fp):
                    vtbl_label = f"{class_name}__vftable"
                    idc.set_name(vtbl_ea, vtbl_label, idc.SN_NOCHECK | idc.SN_FORCE)
                    print(f"  ✓ vtable @ 0x{vtbl_ea:08X} → {vtbl_label}")

# ─── 阶段 4：应用关键函数签名 ──────────────────────────────
print("\n[阶段4] 应用函数签名...")

SIGNATURES = [
    (0x1001A580, "void __cdecl V2Link(void *exporter)"),
    (0x1001A5A0, "void __cdecl V2Unlink(void)"),
    (0x10005200, "int __stdcall EnumFunc(void *hWnd, long lParam)"),
    (0x1000E7C0, "int __cdecl fnEnum(void *ctx, const char *name, unsigned int val)"),
]

for ea, type_str in SIGNATURES:
    ret = idc.SetType(ea, type_str)
    print(f"  {'✓' if ret else '✗'} 0x{ea:08X}: {type_str[:55]}")

# ─── 完成 ──────────────────────────────────────────────────
print("\n" + "=" * 60)
print("符号还原完成！")
print("建议：按 Ctrl+S 保存 .i64 数据库，或用 File → Save 菜单保存")
print("=" * 60)
idc.qexit(0)
```

---

## 8. 常见错误与排查

### 错误 1：`idc.set_name` 返回 0（失败）

**原因**：目标地址已有 IDA 自动分配的名称（如 `dword_XXXXXXXX`），某些情况下 `SN_FORCE` 仍无法覆盖。

**解决方案**：

```python
import ida_name

# 先清除现有名称，再设置新名称
ida_name.del_global_name(ea)  # 清除全局名
idc.set_name(ea, new_name, idc.SN_NOCHECK | idc.SN_FORCE | idc.SN_PUBLIC)
```

### 错误 2：`idc.SetType` 返回 0（类型解析失败）

**原因**：类型字符串中含有 IDA 类型库不认识的类型（如 `HWND`、`tjs_error`）。

**解决方案**：

```python
# 方案 A：先导入 Windows 类型库
idc.import_type(-1, "HWND")  # 从 mssdk.til 导入

# 方案 B：用等价的基础类型替换自定义类型
# HWND → void*
# tjs_error → int
# WPARAM/LPARAM → int（32位）
```

### 错误 3：`XrefsTo` 返回空列表

**原因**：IDA 还没有完成自动分析，RTTI 字符串的交叉引用未建立。

**解决方案**：

```python
import ida_auto

# 等待 IDA 自动分析完成
ida_auto.auto_wait()

# 然后再查询交叉引用
refs = list(idautils.XrefsTo(ea, 0))
```

### 错误 4：脚本中 `idc.qexit(0)` 导致 IDA 立即退出，未保存数据库

**原因**：`-A` 模式（无界面批处理）下，`qexit()` 会直接退出。如果只是重命名（修改 `.i64` 数据库），需要在 `qexit()` 前触发保存。

**解决方案**：

```python
import idc
import ida_loader

# 在 qexit() 前保存数据库
ida_loader.save_database("", 0)  # 第一个参数为空时保存到原路径
idc.qexit(0)
```

---

## 9. 验证：打开 IDA GUI 检查效果

完成脚本运行后，用以下步骤验证效果：

1. **重新打开 `.i64` 文件**（或在同一 IDA 会话中直接查看）
2. 在 Functions 窗口（View → Open Subviews → Functions）中，确认 `sub_10005970` 已变为 `WindowEx__onMessage`
3. 双击 `WindowEx__onMessage`，反编译视图中查看参数列表是否正确
4. 在 Names 窗口（View → Open Subviews → Names）中搜索 `vftable`，确认 vtable 标签已出现

**预期效果对比**：

| 还原前 | 还原后 |
|--------|--------|
| `sub_10005970(int a1, int a2, int a3, int a4)` | `WindowEx__onMessage(void *thisptr, void *hWnd, unsigned int uMsg, int wParam, int lParam)` |
| `sub_1001A580(int a1)` | `V2Link(void *exporter)` |
| `unk_10030F2C` | `WindowEx__TypeDescriptor` |
| `dword_100309E0` | `WindowEx__vftable` |

---

## 对照项目源码

本节技术与以下项目文件直接相关：

- `stuffs/ida_scripts/common.py` — 通用 IDAPython 工具（`idc.qexit(0)` 的封装、`ida_auto.auto_wait()` 调用）
- `stuffs/ida_scripts/extract_windowex.py` — windowEx 专用分析脚本，已提取了大量 RTTI 和函数信息
- `stuffs/ida_scripts/windowex_analysis.txt` — 真实 IDA 分析结果（4035 行），所有地址均来自这里
- `cpp/core/plugin/ncbind.hpp` — ncbind 框架源码，理解 `ncbFunctionTag_*` 字符串的生成逻辑

相关文件：
- `stuffs/ida_scripts/` — 第 5-6 节代码示例的存放位置
- `cpp/core/plugin/ncbind.hpp` 第 1-120 行 — `ncbAttachTJS2ClassAutoRegister<T>`、`ncbInstanceAdaptor<T>` 定义

---

## 本节小结

- MSVC RTTI 的 `TypeDescriptor` 结构存储类名字符串（以 `.?AU`/`.?AV` 开头），通过交叉引用可追溯到 vtable 和构造函数
- `idc.set_name(ea, name, SN_NOCHECK|SN_FORCE)` 是批量重命名的核心 API，失败时可先 `del_global_name()` 再重试
- `idc.SetType()` 用字符串直接解析函数签名，遇到未知类型时替换为等价基础类型
- ncbind 框架的 `ncbFunctionTag_*` 字符串是定位 TJS2 方法回调的有效线索
- 所有修改都写入 `.i64` 数据库，需要在脚本末尾调用 `ida_loader.save_database("", 0)` 持久化

---

## 练习题与答案

### 题目 1：修改扫描脚本，同时输出每个 RTTI 类的完整修饰名和解修饰后的可读名

<details>
<summary>查看答案</summary>

```python
import idc
import idautils

def demangle_simple(mangled):
    """
    简单解修饰：处理 MSVC RTTI TypeDescriptor 中的类名格式
    输入示例：".?AU?$ncbAttachTJS2ClassAutoRegister@UWindowEx@@@@"
    输出示例："ncbAttachTJS2ClassAutoRegister<WindowEx>"
    """
    # 去掉 .?A 前缀
    s = mangled[3:]  # 跳过 ".?A"
    # 识别类型标记
    if s.startswith('U'):
        kind = "struct"
        s = s[1:]
    elif s.startswith('V'):
        kind = "class"
        s = s[1:]
    else:
        kind = "unknown"

    # 去掉末尾 @@
    s = s.rstrip('@')

    # 简单替换模板语法（@UWindowEx → <WindowEx>）
    import re
    s = re.sub(r'@U(\w+)', r'<\1>', s)
    s = re.sub(r'@V(\w+)', r'<\1>', s)
    s = re.sub(r'\?\$', '', s)  # ?$ 是模板前缀

    return f"{kind} {s}"

for seg in idautils.Segments():
    if idc.get_segm_name(seg) not in ('.rdata', '.data'):
        continue
    ea = seg
    while ea < idc.get_segm_end(seg):
        s = idc.get_strlit_contents(ea, -1, idc.STRTYPE_C)
        if s:
            try:
                s_str = s.decode('ascii')
                if s_str.startswith('.?A'):
                    readable = demangle_simple(s_str)
                    print(f"0x{ea:08X}  修饰名: {s_str!r}")
                    print(f"           可读名: {readable}")
                ea += len(s) + 1
            except Exception:
                ea += 1
        else:
            ea += 1

idc.qexit(0)
```

</details>

### 题目 2：编写一个函数，检查某个地址的函数是否已有合理名称（非 `sub_*`），并输出统计结果

<details>
<summary>查看答案</summary>

```python
import idc
import idautils

def count_named_vs_anonymous():
    """统计数据库中具名函数与匿名函数的数量"""
    named = []
    anonymous = []
    library_funcs = []

    for func_ea in idautils.Functions():
        name = idc.get_func_name(func_ea)
        flags = idc.get_func_attr(func_ea, idc.FUNCATTR_FLAGS)

        # 库函数（来自 FLIRT 签名匹配）
        if flags & idc.FUNC_LIB:
            library_funcs.append((func_ea, name))
        # 匿名函数（以 sub_ 开头）
        elif name.startswith('sub_'):
            anonymous.append((func_ea, name))
        # 已命名函数
        else:
            named.append((func_ea, name))

    print(f"函数统计：")
    print(f"  已命名函数：{len(named)} 个")
    print(f"  匿名函数（sub_*）：{len(anonymous)} 个")
    print(f"  库函数（FLIRT 匹配）：{len(library_funcs)} 个")
    print(f"  总计：{len(named) + len(anonymous) + len(library_funcs)} 个")
    print(f"\n命名覆盖率：{len(named) / (len(named) + len(anonymous)) * 100:.1f}%")

    print(f"\n前10个已命名函数：")
    for ea, name in named[:10]:
        print(f"  0x{ea:08X}  {name}")

    return named, anonymous, library_funcs

named, anon, lib = count_named_vs_anonymous()
idc.qexit(0)
```

</details>

### 题目 3：解释为什么 `TypeDescriptor.name` 以 `.?A` 开头而不是直接存类名，这样设计的目的是什么？

<details>
<summary>查看答案</summary>

`.?A` 前缀是 MSVC `type_info::name()` 返回值的内部编码格式，具体原因如下：

1. **`.`**：MSVC `type_info::name()` 的修饰名以 `.` 开头（与函数修饰名以 `?` 开头区分，防止命名冲突）
2. **`?A`**：代表"没有访问修饰符的 class/struct 类型"（MSVC 内部的 type encoding）
3. **`U` vs `V`**：`U` = `struct`（在 C++ 类型系统中是没有隐式 `private` 的结构体），`V` = `class`（有隐式 `private` 的类）

这样设计的目的：
- **唯一性**：加前缀可以防止 `type_info::name()` 返回的字符串与普通字符串或其他符号冲突
- **快速识别**：IDA、调试器、反汇编工具可以快速扫描 `.?AU`/`.?AV` 前缀来定位所有 RTTI 记录
- **MSVC 兼容性**：`dynamic_cast` 的实现依赖这套固定格式的字符串来匹配类型

在 64 位 PE 中，TypeDescriptor 的布局相同，但所有指针宽度变为 8 字节，且 CompleteObjectLocator 中的指针改为相对偏移（RVA），而非绝对地址。

</details>

---

## 下一步

[02 — 结构体布局还原与内存对齐](./02-结构体布局还原与内存对齐.md)

掌握函数符号还原后，下一节深入讲解如何用 IDAPython 的 `add_struc` / `add_struc_member` API 在 IDA 数据库中创建 `WindowEx`、`MenuItemEx` 等结构体定义，让所有 `[ecx+0x4]` 风格的成员偏移访问自动变为 `this->hwnd` 形式，大幅提升反编译视图的可读性。
