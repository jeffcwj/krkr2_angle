# PackinOne 功能分析与导出表

> **所属模块：** M12-插件逆向与实现
> **前置知识：** [DLL 导出表与接口识别](../03-逆向方法论/01-DLL导出表与接口识别.md)、[逆向验证与测试策略](../03-逆向方法论/03-逆向验证与测试策略.md)
> **预计阅读时间：** 40 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：
1. 理解 PackinOne 插件在 KiriKiri 游戏引擎中的作用和位置
2. 使用 dumpbin（Windows）和 objdump（Linux/macOS）分析 PackinOne.dll 的导出表
3. 在 Ghidra 中加载 PackinOne.dll 并定位关键函数
4. 通过导入表推断 PackinOne 使用的 Windows API 和算法特征
5. 建立 PackinOne 的功能模型——哪些函数做什么、数据如何流动

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| PackinOne | PackinOne | KiriKiri 游戏引擎的一个压缩/归档插件，将多个文件打包成单一归档文件以便分发 |
| 导出表 | Export Table | DLL 文件中列出"本 DLL 对外提供哪些函数"的数据结构 |
| 导入表 | Import Table | DLL 文件中列出"本 DLL 需要调用哪些外部函数"的数据结构 |
| V2Link | V2Link | KiriKiri 插件的入口函数——引擎加载插件时第一个调用的函数 |
| V2Unlink | V2Unlink | KiriKiri 插件的清理函数——引擎卸载插件时调用 |
| Ghidra | Ghidra | NSA（美国国家安全局）开发的开源逆向工程框架，支持反汇编和反编译 |
| PE 文件 | Portable Executable | Windows 可执行文件格式（.exe、.dll 都是 PE 格式） |
| 交叉引用 | Cross-reference (XREF) | 反汇编器中标记"哪些地方调用了这个函数"或"哪些地方引用了这个数据"的信息 |

---

## 一、PackinOne 在 KiriKiri 生态中的定位

### 1.1 PackinOne 是什么

PackinOne 是 KiriKiri/吉里吉里 游戏引擎的一个第三方插件（Third-party Plugin），主要功能是**数据压缩和归档**。在 KiriKiri 游戏的发布流程中，开发者会使用 PackinOne 将游戏资源（脚本、图像、音频等）打包成自定义格式的归档文件，起到两个作用：

1. **减小分发体积：** 对资源文件进行压缩，减少最终安装包的大小
2. **简单的防篡改：** 将文件打包后，玩家不容易直接修改游戏资源（虽然对真正的逆向人员来说只是多了一步解包操作）

### 1.2 为什么需要逆向 PackinOne

在我们的 KrKr2 模拟器项目中，PackinOne 被列为 **B 类插件（有外部源码参考）**（参见 [无源码插件功能分析与优先级](../02-无源码插件清单/02-无源码插件功能分析与优先级.md)），优先级为"Phase 2 — 第一批需要逆向的插件"。逆向 PackinOne 的原因：

1. **部分游戏依赖它：** 一些 KiriKiri 游戏使用 PackinOne 格式的归档文件存放资源。如果模拟器不能解压这种格式，这些游戏就无法运行
2. **没有官方源码：** PackinOne 的作者没有公开源码，只提供了编译好的 Windows DLL
3. **无法直接使用原版 DLL：** 我们的模拟器需要在 Android/Linux/macOS 上运行，Windows DLL 无法直接加载
4. **有参考实现可对照：** 社区中存在一些 PackinOne 格式的部分分析，可以辅助逆向（这也是它被归为 B 类而非 C 类的原因）

### 1.3 PackinOne 在插件清单中的位置

回顾 `doc/krkr2_plugins.md` 中与 PackinOne 相关的条目：

```
# 项目插件清单（摘录）

| 插件名称        | 类别 | 源码状态 | 功能         |
|----------------|------|---------|--------------|
| PackinOne.dll  | B    | 外部参考 | 数据压缩/归档 |
```

在我们的 4 阶段逆向路线图中，PackinOne 属于 **Phase 2（需要逆向，预计 1-2 周）**，与 `util_graph.dll` 并列为第一批逆向目标。

---

## 二、使用 dumpbin 分析导出表（Windows）

### 2.1 获取 PackinOne.dll

首先需要从一款使用 PackinOne 的 KiriKiri 游戏中提取 DLL 文件。典型的位置：

```
游戏安装目录/
├── game.exe          # KiriKiri 主程序
├── data.xp3          # XP3 格式资源包
├── plugin/
│   ├── PackinOne.dll  # ← 我们需要的文件
│   ├── wuvorbis.dll
│   └── ...
```

> **法律提示：** 逆向商业软件的合法性因国家/地区而异。在中国，《计算机软件保护条例》第十七条允许为了学习和研究软件内含的设计思想而进行必要的复制。本教程的逆向分析仅用于开源模拟器的兼容性实现。

### 2.2 dumpbin 导出表分析

`dumpbin` 是 Visual Studio 自带的命令行工具，用于分析 PE（Portable Executable——Windows 的可执行文件格式，.exe 和 .dll 都属于 PE 格式）文件。在 "x64 Native Tools Command Prompt" 或 "Developer Command Prompt" 中运行：

```batch
REM 查看导出表（PackinOne 对外提供的函数）
dumpbin /exports PackinOne.dll

REM 典型输出：
REM Microsoft (R) COFF/PE Dumper Version 14.38.33130.0
REM
REM Dump of file PackinOne.dll
REM
REM File Type: DLL
REM
REM   Section contains the following exports for PackinOne.dll
REM
REM     00000000 characteristics
REM     FFFFFFFF time date stamp
REM            0 major version
REM            0 minor version
REM            1 ordinal base
REM            2 number of functions
REM            2 number of names
REM
REM     ordinal hint RVA      name
REM
REM           1    0 00001000 V2Link
REM           2    1 00001050 V2Unlink
```

**解读：** PackinOne 只导出两个函数——`V2Link` 和 `V2Unlink`。这是 KiriKiri 插件的标准模式（参见 [V2Link 与 V2Unlink 入口机制](../01-KiriKiri插件接口分析/01-V2Link与V2Unlink入口机制.md)），意味着：

- 所有压缩/解压功能都通过 TJS（T Visual Presenter Script——KiriKiri 的内置脚本语言）对象注册，**不通过导出函数暴露**
- V2Link 是入口，执行初始化和 TJS 注册
- V2Unlink 是清理，解除 TJS 注册

### 2.3 dumpbin 导入表分析

导入表告诉我们 PackinOne 依赖了哪些外部函数——这些线索对推断功能至关重要：

```batch
REM 查看导入表（PackinOne 需要调用的外部函数）
dumpbin /imports PackinOne.dll

REM 典型输出（关键部分）：
REM
REM   Section contains the following imports:
REM
REM   KERNEL32.dll
REM     VirtualAlloc
REM     VirtualFree
REM     GetProcessHeap
REM     HeapAlloc
REM     HeapFree
REM     GetTickCount
REM     CreateFileW
REM     ReadFile
REM     WriteFile
REM     CloseHandle
REM     GetFileSize
REM
REM   MSVCRT.dll
REM     malloc
REM     free
REM     memcpy
REM     memset
REM     _CxxThrowException
```

**从导入表推断功能：**

```
导入表功能推断矩阵：

┌─────────────────────┬────────────────────────────────────┐
│ 导入的函数          │ 推断                               │
├─────────────────────┼────────────────────────────────────┤
│ VirtualAlloc        │ 需要大块连续内存——可能用于         │
│ VirtualFree         │ 解压缓冲区（压缩数据可能很大）     │
├─────────────────────┼────────────────────────────────────┤
│ CreateFileW         │ 直接操作文件（而不是通过 KiriKiri   │
│ ReadFile/WriteFile  │ 的 Stream 接口）——可能是工具模式   │
│ GetFileSize         │ 下的独立文件处理                   │
├─────────────────────┼────────────────────────────────────┤
│ memcpy/memset       │ 大量内存操作——LZ 类算法的典型特征  │
│                     │ （回引复制用 memcpy，初始化用       │
│                     │ memset）                           │
├─────────────────────┼────────────────────────────────────┤
│ _CxxThrowException  │ 使用 C++ 异常——错误处理用 throw   │
│                     │ 而不是返回错误码                   │
├─────────────────────┼────────────────────────────────────┤
│ 无 GDI32.dll        │ 不涉及图形/字体——纯数据处理插件   │
│ 无 WS2_32.dll       │ 不涉及网络——纯本地操作             │
└─────────────────────┴────────────────────────────────────┘
```

**关键发现：** 导入表中没有 `zlib`、`lz4`、`lzma` 等标准压缩库的函数——说明 PackinOne **使用自研或内联的压缩算法**，这是逆向的核心挑战。

---

## 三、使用 objdump 分析导出表（Linux/macOS）

### 3.1 在 Linux 上分析 Windows DLL

虽然 PackinOne.dll 是 Windows PE 文件，但 Linux 上的 `objdump`（来自 GNU binutils）和 `readpe`（来自 pev 工具集）也能分析它：

```bash
# 方法 1：objdump（大多数 Linux 发行版预装）
# 需要安装 mingw 版本的 binutils 来支持 PE 格式
# Ubuntu/Debian:
sudo apt install binutils-mingw-w64-i686
i686-w64-mingw32-objdump -p PackinOne.dll | grep -A 100 "Export Table"

# 方法 2：readpe（来自 pev 工具集）
# Ubuntu/Debian:
sudo apt install pev
readpe -e PackinOne.dll    # 显示导出表
readpe -i PackinOne.dll    # 显示导入表

# 方法 3：使用 Python 的 pefile 库（跨平台最佳选择）
pip install pefile
python3 -c "
import pefile
pe = pefile.PE('PackinOne.dll')

print('=== 导出表 ===')
if hasattr(pe, 'DIRECTORY_ENTRY_EXPORT'):
    for exp in pe.DIRECTORY_ENTRY_EXPORT.symbols:
        name = exp.name.decode() if exp.name else '(ordinal only)'
        print(f'  {exp.ordinal:4d}  0x{exp.address:08X}  {name}')

print()
print('=== 导入表 ===')
if hasattr(pe, 'DIRECTORY_ENTRY_IMPORT'):
    for entry in pe.DIRECTORY_ENTRY_IMPORT:
        dll_name = entry.dll.decode()
        print(f'  {dll_name}:')
        for imp in entry.imports:
            name = imp.name.decode() if imp.name else f'ordinal({imp.ordinal})'
            print(f'    {name}')
"
```

### 3.2 macOS 上的分析

macOS 没有预装 PE 分析工具，推荐使用 Python pefile 或 Homebrew 安装的工具：

```bash
# 安装 pefile（最简单的方案）
pip3 install pefile

# 或者安装 binutils（包含 objdump）
brew install binutils
gobjdump -p PackinOne.dll | grep -A 50 "Export"

# 或者使用 radare2（强大的跨平台逆向工具）
brew install radare2
r2 -q -c "iE" PackinOne.dll    # 显示导出表
r2 -q -c "ii" PackinOne.dll    # 显示导入表
```

### 3.3 PE 文件头部信息

除了导出/导入表，PE 头部还包含有用的元数据：

```bash
# 使用 pefile 提取 PE 头部信息
python3 -c "
import pefile
pe = pefile.PE('PackinOne.dll')

print('=== PE 头部信息 ===')
print(f'机器类型: {hex(pe.FILE_HEADER.Machine)}')
# 0x14c = i386 (32位), 0x8664 = AMD64 (64位)

print(f'时间戳:   {pe.FILE_HEADER.TimeDateStamp}')
print(f'节数量:   {pe.FILE_HEADER.NumberOfSections}')

print()
print('=== 节表 ===')
for section in pe.sections:
    name = section.Name.decode().rstrip('\\x00')
    print(f'  {name:8s}  VA=0x{section.VirtualAddress:08X}  '
          f'Size=0x{section.Misc_VirtualSize:08X}  '
          f'Raw=0x{section.SizeOfRawData:08X}')
"

# 典型输出：
# === PE 头部信息 ===
# 机器类型: 0x14c         ← 32位 x86
# 时间戳:   1234567890
# 节数量:   4
#
# === 节表 ===
#   .text     VA=0x00001000  Size=0x00008000  Raw=0x00008000   ← 代码段
#   .rdata    VA=0x00009000  Size=0x00002000  Raw=0x00002000   ← 只读数据
#   .data     VA=0x0000B000  Size=0x00001000  Raw=0x00000800   ← 可写数据
#   .reloc    VA=0x0000C000  Size=0x00000800  Raw=0x00000800   ← 重定位表
```

**关键信息：** 
- **机器类型 0x14c（i386）** 说明这是 32 位 DLL，Ghidra 分析时需要选择 `x86:LE:32:default` 语言
- **.text 段大小**约 32KB，说明代码量不大——压缩算法不会太复杂

---

## 四、在 Ghidra 中加载和初步分析

### 4.1 创建 Ghidra 项目

打开 Ghidra，按以下步骤加载 PackinOne.dll：

```
步骤 1：File → New Project → Non-Shared Project
        项目名：PackinOne_RE
        路径：选择你的逆向工作目录

步骤 2：File → Import File → 选择 PackinOne.dll
        Ghidra 自动检测格式：
          Format:    Portable Executable (PE)
          Language:  x86:LE:32:default     ← 32位小端序
          Compiler:  Visual Studio

步骤 3：双击文件打开 CodeBrowser
        Ghidra 提示 "Analyze?"：点击 Yes
        分析选项：保持默认，确保以下勾选：
          ✅ Decompiler Parameter ID
          ✅ Windows x86 PE Exception Handling
          ✅ Stack
          ✅ Reference
        点击 Analyze，等待分析完成（约 10-30 秒）
```

### 4.2 定位 V2Link 函数

分析完成后，在 Symbol Tree 中找到导出函数：

```
Symbol Tree → Exports:
  📁 V2Link    → 地址 0x10001000
  📁 V2Unlink  → 地址 0x10001050
```

双击 `V2Link` 查看反编译结果。典型的 KiriKiri 插件 V2Link 模式如下：

```cpp
// Ghidra 反编译输出（已手动重命名变量和类型）
// 地址：0x10001000
HRESULT __stdcall V2Link(iTVPFunctionExporter* exporter) {

    // 步骤 1：初始化 tp_stub（导入 KiriKiri 引擎函数）
    // TVPInitImportStub 是 tp_stub.h 中的宏展开
    bool success = TVPInitImportStub(exporter);
    if (!success) {
        return E_FAIL;  // 0x80004005
    }

    // 步骤 2：获取 TJS 全局脚本对象
    iTJSDispatch2* global = nullptr;
    TVPGetScriptDispatch(&global);

    if (global != nullptr) {
        // 步骤 3：注册 PackinOne 的 TJS 类/函数
        // 这里是关键——我们需要找出注册了什么
        register_pkn_class(global);

        global->Release();
    }

    return S_OK;  // 0x00000000
}
```

> **注意：** Ghidra 的反编译输出不会这么"干净"——变量名会是 `param_1`、`local_4` 等自动生成的名字，类型也可能不准确。上面的代码是经过人工重命名后的版本。原始输出更接近：
>
> ```c
> undefined4 __stdcall V2Link(int param_1) {
>     int iVar1;
>     int local_4;
>     iVar1 = FUN_10002000(param_1);
>     if (iVar1 == 0) return 0x80004005;
>     FUN_10002100(&local_4);
>     if (local_4 != 0) {
>         FUN_10003000(local_4);
>         (**(code **)(*(int *)local_4 + 8))(local_4);  // Release()
>     }
>     return 0;
> }
> ```

### 4.3 追踪 TJS 注册调用

关键是 `register_pkn_class`（即 `FUN_10003000`）做了什么。在 Ghidra 中双击进入这个函数：

```cpp
// FUN_10003000 的反编译输出（经人工分析重命名）
void register_pkn_class(iTJSDispatch2* global) {

    // 创建一个 TJS 原生类
    iTJSDispatch2* pkn_class = create_native_class("PackinOne");

    // 注册方法到类上
    // 方法 1：compress（压缩）
    register_method(pkn_class, L"compress", &pkn_compress_wrapper);

    // 方法 2：decompress（解压）
    register_method(pkn_class, L"decompress", &pkn_decompress_wrapper);

    // 方法 3：getArchiveInfo（获取归档信息）
    register_method(pkn_class, L"getArchiveInfo", &pkn_getinfo_wrapper);

    // 将类注册到全局作用域
    tTJSVariant class_var(pkn_class);
    global->PropSet(
        TJS_MEMBERENSURE,    // 如果不存在就创建
        L"PackinOne",        // 全局名称
        nullptr,             // hint
        &class_var,          // 值
        global               // 对象自身
    );

    pkn_class->Release();
}
```

**这告诉我们 PackinOne 注册了 3 个关键方法：**

| TJS 方法名 | 底层实现函数 | 功能 |
|-----------|-------------|------|
| `compress` | `pkn_compress_wrapper` → `FUN_10004000` | 将数据压缩为 PackinOne 格式 |
| `decompress` | `pkn_decompress_wrapper` → `FUN_10005000` | 将 PackinOne 格式数据解压 |
| `getArchiveInfo` | `pkn_getinfo_wrapper` → `FUN_10006000` | 读取归档文件的元数据（文件列表、大小等） |

### 4.4 交叉引用分析——确认核心函数

使用 Ghidra 的交叉引用功能确认哪些函数是核心实现，哪些是包装器：

```
操作步骤：
1. 在 Listing 视图中选中 FUN_10004000 的地址
2. 右键 → References → Show References to Address
3. 查看引用列表：

XREF[1]:
  FUN_10003000:10003042  ← 被注册函数调用（TJS 包装器）

对 FUN_10004000 内部的调用再做交叉引用：
  FUN_10004000 调用了：
    → FUN_10004200  ← 实际的压缩算法实现
    → FUN_10004300  ← 缓冲区管理
    → HeapAlloc      ← 内存分配
    → memcpy         ← 数据复制
```

**函数调用图（由交叉引用推断）：**

```
V2Link
  └→ register_pkn_class
       ├→ pkn_compress_wrapper
       │    └→ pkn_compress_impl       ← 核心：压缩算法
       │         ├→ init_compress_ctx   ← 初始化压缩上下文
       │         ├→ compress_block      ← 按块压缩
       │         └→ write_header        ← 写入文件头
       │
       ├→ pkn_decompress_wrapper
       │    └→ pkn_decompress_impl     ← 核心：解压算法
       │         ├→ read_header         ← 读取并验证文件头
       │         ├→ decompress_block    ← 按块解压
       │         └→ verify_checksum     ← 校验和验证
       │
       └→ pkn_getinfo_wrapper
            └→ pkn_getinfo_impl        ← 解析归档元数据
                 └→ read_header
```

---

## 五、识别压缩算法特征

### 5.1 通过反编译代码识别算法类型

进入 `compress_block`（FUN_10004200）函数，观察其结构特征：

```cpp
// FUN_10004200 反编译输出（关键部分，已重命名）
void compress_block(
    uint8_t* output, size_t* out_size,
    const uint8_t* input, size_t in_size
) {
    // 特征 1：滑动窗口搜索
    // 在已处理的数据中搜索与当前位置相同的最长匹配
    size_t pos = 0;
    while (pos < in_size) {
        size_t best_offset = 0;
        size_t best_length = 0;

        // 向前搜索匹配（回引窗口大小 = 0x1000 = 4096 字节）
        size_t search_start = (pos > 0x1000) ? (pos - 0x1000) : 0;
        for (size_t s = search_start; s < pos; ++s) {
            size_t match_len = 0;
            while (pos + match_len < in_size
                   && input[s + match_len] == input[pos + match_len]
                   && match_len < 0x12) {  // 最大匹配长度 = 18
                ++match_len;
            }
            if (match_len > best_length) {
                best_length = match_len;
                best_offset = pos - s;
            }
        }

        // 特征 2：标志位编码
        if (best_length >= 3) {
            // 回引：用 (offset, length) 对编码
            // 编码格式：标志位 1 + 12位偏移 + 4位长度
            encode_backref(output, out_size, best_offset, best_length);
            pos += best_length;
        } else {
            // 字面量：直接写入原始字节
            // 编码格式：标志位 0 + 8位字面值
            encode_literal(output, out_size, input[pos]);
            ++pos;
        }
    }
}
```

### 5.2 算法特征总结

通过以上反编译分析，我们可以确认 PackinOne 使用的是 **LZSS 变体**（Lempel-Ziv-Storer-Szymanski——一种基于滑动窗口的无损压缩算法，是 LZ77 的改进版本）。具体参数：

| 参数 | 值 | 说明 |
|------|-----|------|
| 滑动窗口大小 | 4096 字节（12 位） | 回引偏移的最大范围 |
| 最大匹配长度 | 18 字节（4 位 + 2） | 单次回引能复制的最大字节数 |
| 最小匹配长度 | 3 字节 | 短于 3 字节的匹配不值得编码（回引本身占 2 字节） |
| 标志位方式 | 8 位标志字节 | 每 8 个编码单元共享一个标志字节，每位指示后续是字面量(0)还是回引(1) |
| 回引编码 | 2 字节 | 高 12 位 = 偏移，低 4 位 = 长度 - 2 |
| 字面量编码 | 1 字节 | 直接存储原始字节 |

这个参数组合与经典的 LZSS 非常接近，只是标志位的排列方式稍有不同。

### 5.3 文件格式头部结构

通过分析 `read_header` 和 `write_header` 函数，推断出 PackinOne 归档的文件格式：

```cpp
// PackinOne 归档文件头部结构（从反编译代码推断）
// 注意：所有多字节字段为小端序（Little Endian）

#pragma pack(push, 1)
struct PknFileHeader {
    uint8_t  magic[4];      // 0x00: 魔数 "PKN\0" (0x50, 0x4B, 0x4E, 0x00)
    uint32_t version;       // 0x04: 版本号（通常为 1 或 2）
    uint32_t num_entries;   // 0x08: 归档中的文件条目数
    uint32_t data_offset;   // 0x0C: 第一个数据块的偏移量（相对于文件开头）
    uint32_t total_size;    // 0x10: 所有文件解压后的总大小
    uint32_t compressed_size; // 0x14: 所有文件压缩后的总大小
    uint32_t checksum;      // 0x18: 头部 CRC32 校验和
};

struct PknEntryHeader {
    uint32_t name_length;       // 0x00: 文件名长度（字节数，包含 null 终止符）
    // char  name[name_length]; // 0x04: 文件名（UTF-8 或 Shift-JIS 编码）
    uint32_t original_size;     // 变长: 文件原始大小
    uint32_t compressed_size;   // 变长: 文件压缩后大小
    uint32_t offset;            // 变长: 数据在归档中的偏移
    uint32_t checksum;          // 变长: 文件数据的 CRC32 校验和
};
#pragma pack(pop)
```

### 5.4 用十六进制编辑器验证格式

用实际的 PackinOne 归档文件验证推断的格式：

```bash
# 使用 xxd 查看文件开头
xxd -l 64 sample.pkn
# 输出示例：
# 00000000: 504b 4e00 0100 0000 0300 0000 1c00 0000  PKN.............
#           ^^^^^^^^ ^^^^^^^^ ^^^^^^^^ ^^^^^^^^
#           magic    version  entries  data_off
#           "PKN\0"  1        3个文件   0x1C
#
# 00000010: 0050 0000 2038 0000 a1b2 c3d4           .P.. 8......
#           ^^^^^^^^ ^^^^^^^^ ^^^^^^^^
#           total_sz comp_sz  checksum
#           20480    14368    CRC32

# 使用 Python 验证 CRC32
python3 -c "
import struct, binascii

with open('sample.pkn', 'rb') as f:
    header = f.read(0x1C)

# 校验和是头部最后 4 字节
stored_crc = struct.unpack_from('<I', header, 0x18)[0]

# 计算前 0x18 字节的 CRC32
computed_crc = binascii.crc32(header[:0x18]) & 0xFFFFFFFF

print(f'存储的 CRC32: 0x{stored_crc:08X}')
print(f'计算的 CRC32: 0x{computed_crc:08X}')
print(f'匹配: {stored_crc == computed_crc}')
"
```

---

## 动手实践

### 实践 1：用 Python pefile 编写自动化 DLL 分析脚本

**目标：** 编写一个 Python 脚本，自动分析任意 KiriKiri 插件 DLL，输出导出表、导入表摘要和功能推断。

```python
#!/usr/bin/env python3
# analyze_krkr_plugin.py — KiriKiri 插件 DLL 自动分析工具
import sys
import pefile

def analyze_plugin(dll_path: str) -> None:
    """分析 KiriKiri 插件 DLL 的导出表和导入表"""
    pe = pefile.PE(dll_path)

    # === 基本信息 ===
    machine = pe.FILE_HEADER.Machine
    machine_str = {0x14c: "x86 (32位)", 0x8664: "AMD64 (64位)"}.get(
        machine, f"未知 (0x{machine:x})"
    )
    print(f"文件: {dll_path}")
    print(f"架构: {machine_str}")
    print(f"节数: {pe.FILE_HEADER.NumberOfSections}")
    print()

    # === 导出表 ===
    print("=== 导出表 ===")
    is_krkr_plugin = False
    if hasattr(pe, "DIRECTORY_ENTRY_EXPORT"):
        for exp in pe.DIRECTORY_ENTRY_EXPORT.symbols:
            name = exp.name.decode() if exp.name else f"ordinal({exp.ordinal})"
            print(f"  {exp.ordinal:4d}  0x{exp.address:08X}  {name}")
            if name in ("V2Link", "V2Unlink"):
                is_krkr_plugin = True
    else:
        print("  (无导出表)")

    if is_krkr_plugin:
        print("\n  ✅ 确认为 KiriKiri 插件（导出 V2Link/V2Unlink）")
    else:
        print("\n  ⚠️ 未检测到 V2Link/V2Unlink，可能不是 KiriKiri 插件")

    # === 导入表分析 ===
    print("\n=== 导入表（功能推断）===")
    gdi_imports = []
    file_imports = []
    net_imports = []
    memory_imports = []

    if hasattr(pe, "DIRECTORY_ENTRY_IMPORT"):
        for entry in pe.DIRECTORY_ENTRY_IMPORT:
            dll_name = entry.dll.decode()
            funcs = []
            for imp in entry.imports:
                name = imp.name.decode() if imp.name else f"ord({imp.ordinal})"
                funcs.append(name)

                # 分类
                if "GDI" in dll_name.upper():
                    gdi_imports.append(name)
                if name in ("CreateFileW", "ReadFile", "WriteFile", "GetFileSize"):
                    file_imports.append(name)
                if "WS2" in dll_name.upper() or "WINHTTP" in dll_name.upper():
                    net_imports.append(name)
                if name in ("VirtualAlloc", "HeapAlloc", "malloc"):
                    memory_imports.append(name)

            print(f"  {dll_name} ({len(funcs)} 函数)")

    # === 功能推断 ===
    print("\n=== 功能推断 ===")
    if gdi_imports:
        print(f"  📝 涉及图形/字体处理（导入 GDI: {', '.join(gdi_imports[:5])}）")
    if file_imports:
        print(f"  📁 涉及文件 I/O（{', '.join(file_imports)}）")
    if net_imports:
        print(f"  🌐 涉及网络通信（{', '.join(net_imports[:3])}）")
    if memory_imports:
        print(f"  💾 大量内存操作（{', '.join(memory_imports)}）→ 可能是压缩/解压")
    if not (gdi_imports or net_imports):
        print("  ℹ️ 纯数据处理插件（无图形/网络导入）")


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print(f"用法: {sys.argv[0]} <plugin.dll>")
        sys.exit(1)
    analyze_plugin(sys.argv[1])
```

**运行：**

```bash
python3 analyze_krkr_plugin.py PackinOne.dll
```

### 实践 2：在 Ghidra 中手动追踪 V2Link 的注册流程

**步骤：**

1. 在 Ghidra 中打开任意 KiriKiri 插件 DLL
2. 找到 `V2Link` 导出函数
3. 识别 `TVPInitImportStub` 调用（通常是 V2Link 中的第一个函数调用）
4. 找到 `TVPGetScriptDispatch` 调用
5. 追踪后续的注册调用，记录注册的 TJS 类名和方法名
6. 绘制函数调用图

**预期结果：** 能够列出插件注册的所有 TJS 接口（类名、方法名、属性名）。

---

## 对照项目源码

以下是 KrKr2 项目中与插件加载和归档处理相关的文件：

| 文件路径 | 说明 |
|----------|------|
| `cpp/core/plugin/ncbind.hpp` 第 1-50 行 | ncbind 框架的核心宏定义——PackinOne 可能使用类似的注册模式 |
| `cpp/core/plugin/ncbind.cpp` 第 1-29 行 | LoadModule 函数——引擎如何加载插件的入口 |
| `cpp/core/base/XP3Archive.cpp` 全文 | XP3 归档格式的实现——PackinOne 是类似的归档格式，可以参考其结构 |
| `cpp/plugins/xp3filter.cpp` 第 1-100 行 | XP3 过滤器插件——展示了另一种数据处理插件的实现模式 |
| `doc/krkr2_plugins.md` 全文 | 完整插件清单——PackinOne 的分类和状态 |

**关键对比：** XP3Archive 的文件头解析逻辑（`XP3Archive.cpp`）和 PackinOne 的头部解析高度类似——都使用魔数验证 + 固定偏移读取字段 + CRC32 校验。如果你已经读懂了 XP3 的实现，PackinOne 的头部解析就很容易理解。

---

## 常见错误与排查

### 错误 1：Ghidra 分析 32 位 DLL 时选错了语言

**症状：** 反编译输出完全是乱码，函数调用关系不合理，到处是 `undefined` 类型。

**原因：** 导入文件时 Language 选成了 `x86:LE:64:default`（64 位），但 DLL 实际是 32 位的。32 位和 64 位的调用约定完全不同（32 位用栈传参，64 位用寄存器传参），导致参数解析全错。

**解决：** 删除当前分析，重新导入，Language 选择 `x86:LE:32:default`。可以用 `dumpbin /headers` 或 `pefile` 先确认 PE 的 Machine 字段。

### 错误 2：反编译输出中的虚函数调用看不懂

**症状：** 看到类似 `(**(code **)(*(int *)param_1 + 0x18))(param_1, ...)` 的代码，不知道在调用什么。

**原因：** 这是 C++ 虚函数调用的反编译模式。`param_1` 是对象指针，`*(int *)param_1` 是虚函数表（vtable）指针，`+ 0x18` 是 vtable 中第 6 个函数（每个指针 4 字节，0x18/4 = 6）。

**解决：** 参考 [iTJSDispatch2 接口与 tp_stub](../01-KiriKiri插件接口分析/02-iTJSDispatch2接口与tp_stub.md) 中的 vtable 布局。偏移 0x18 对应 `PropSet` 方法。在 Ghidra 中可以创建结构体类型来辅助分析：

```
Ghidra 操作：
1. Data Type Manager → 右键 → New → Structure
2. 命名为 iTJSDispatch2_vtable
3. 按 vtable 布局添加函数指针字段
4. 在反编译器中将 param_1 的类型设为 iTJSDispatch2*
```

### 错误 3：Python pefile 无法解析某些 DLL

**症状：** `pefile.PE('xxx.dll')` 抛出 `PEFormatError: Invalid DOS header`。

**原因：** 文件不是标准 PE 格式，可能是 UPX 加壳后的 DLL，或者是 .NET 程序集。

**解决：**

```bash
# 检查是否 UPX 加壳
upx -t PackinOne.dll
# 如果显示 "[OK]" 说明是 UPX 压缩的

# 脱壳
upx -d PackinOne.dll -o PackinOne_unpacked.dll

# 然后分析脱壳后的文件
python3 analyze_krkr_plugin.py PackinOne_unpacked.dll
```

---

## 本节小结

- **PackinOne 是 KiriKiri 的数据压缩/归档插件**，在模拟器项目中属于 B 类插件（有外部参考），优先级为 Phase 2
- **导出表只有 V2Link 和 V2Unlink** 两个函数——这是 KiriKiri 插件的标准模式，所有业务功能通过 TJS 注册暴露
- **导入表分析**能快速推断插件功能：无 GDI32 = 非图形类；有 VirtualAlloc + memcpy = 大块数据处理（压缩/解压）；无第三方压缩库 = 自研算法
- **三种跨平台 PE 分析方法**：Windows 用 dumpbin、Linux 用 objdump/readpe、跨平台用 Python pefile
- **在 Ghidra 中追踪 V2Link 的注册流程**，能找到插件注册的所有 TJS 类和方法——这定义了插件的对外接口
- **PackinOne 使用 LZSS 变体压缩算法**：12 位偏移（4096 字节窗口）、4 位长度（最大 18 字节匹配）、8 位标志字节
- **归档文件格式**：PKN\0 魔数 + 版本号 + 条目数 + 数据偏移 + CRC32 校验——结构类似 XP3 但更简单
- **函数调用图**是逆向分析的重要产出——从 V2Link 出发，梳理注册流程和核心算法入口

---

## 练习题与答案

### 题目 1：分析一个未知 KiriKiri 插件的导入表

**场景：** 你发现一个名为 `UnknownPlugin.dll` 的 KiriKiri 插件，它的导入表中包含以下 DLL 和函数：

- `GDI32.dll`: `CreateFontW`, `GetTextMetricsW`, `GetGlyphOutlineW`, `SelectObject`, `CreateCompatibleDC`
- `KERNEL32.dll`: `VirtualAlloc`, `VirtualFree`, `CreateFileW`, `WriteFile`
- `USER32.dll`: `GetDC`, `ReleaseDC`

**问题：** (a) 这个插件最可能是什么功能？(b) 它与 PackinOne 的导入表有什么本质区别？(c) 在 KrKr2 的逆向路线图中，它可能对应哪个插件？

<details>
<summary>查看答案</summary>

**(a)** 这个插件最可能是**文字渲染/字体处理**插件。理由：
- `CreateFontW`：创建 Windows 字体对象
- `GetTextMetricsW`：获取字体度量信息（字高、字宽等）
- `GetGlyphOutlineW`：获取单个字形的轮廓数据——这是自定义文字渲染的核心 API
- `CreateCompatibleDC` + `GetDC`：创建设备上下文用于离屏渲染

**(b)** 本质区别在于 **GDI32.dll 导入**：
- PackinOne 导入表中**没有 GDI32.dll**，是纯数据处理插件
- UnknownPlugin 大量导入 GDI32 字体/绘图 API，明确涉及图形操作
- 这意味着跨平台移植策略完全不同——PackinOne 只需移植压缩算法，而这个插件需要用 FreeType 等库替代 Windows GDI 的字体功能

**(c)** 在逆向路线图中，它最可能对应 **TextRender.dll**——Phase 3 的逆向目标，是最复杂的逆向任务（预计 2-4 周）。

</details>

### 题目 2：解读 Ghidra 反编译输出中的虚函数调用

**问题：** 以下 Ghidra 反编译代码中，`FUN_10001234` 被调用时实际执行的是 `iTJSDispatch2` 的哪个方法？请解释你的推理过程。

```c
void FUN_10001234(int param_1, wchar_t* param_2, int param_3) {
    int vtable = *(int *)param_1;            // 读取 vtable 指针
    (*(code *)(vtable + 0x10))(              // 通过 vtable 调用
        param_1,        // this 指针
        0,              // flag
        param_2,        // membername
        0,              // hint
        param_3,        // result
        0,              // numparams
        0,              // param
        param_1         // objthis
    );
}
```

<details>
<summary>查看答案</summary>

**推理过程：**

1. `vtable + 0x10` 表示 vtable 中偏移 0x10 的函数指针
2. 在 32 位系统中，每个指针占 4 字节，所以 0x10 / 4 = 第 4 个虚函数
3. 参照 iTJSDispatch2 的 vtable 布局（来自 [iTJSDispatch2 接口与 tp_stub](../01-KiriKiri插件接口分析/02-iTJSDispatch2接口与tp_stub.md)）：

| 偏移 | 方法 |
|------|------|
| 0x00 | AddRef |
| 0x04 | Release |
| 0x08 | FuncCall |
| 0x0C | FuncCallByNum |
| **0x10** | **PropGet** |
| 0x14 | PropGetByNum |
| 0x18 | PropSet |

4. 偏移 0x10 对应 **`PropGet`** 方法
5. 验证参数：`PropGet(flag=0, membername=param_2, hint=0, result=param_3, objthis=param_1)` 与 `PropGet` 的签名一致

**结论：** 调用的是 `iTJSDispatch2::PropGet()`——读取对象的属性值。

</details>

### 题目 3：编写一个 PackinOne 头部解析器

**问题：** 根据本节推断的 PknFileHeader 结构，编写一个 C++ 程序，读取 .pkn 文件的头部并验证其合法性。要求：(a) 检查魔数、(b) 验证 CRC32、(c) 打印所有字段的值。

<details>
<summary>查看答案</summary>

```cpp
// pkn_header_parser.cpp — PackinOne 头部解析器
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <iostream>
#include <vector>

// CRC32 查找表（标准 CRC-32/ISO-HDLC）
static uint32_t crc32_table[256];
static bool crc32_initialized = false;

static void init_crc32_table() {
    for (uint32_t i = 0; i < 256; ++i) {
        uint32_t crc = i;
        for (int j = 0; j < 8; ++j) {
            crc = (crc >> 1) ^ ((crc & 1) ? 0xEDB88320u : 0);
        }
        crc32_table[i] = crc;
    }
    crc32_initialized = true;
}

static uint32_t compute_crc32(const uint8_t* data, size_t size) {
    if (!crc32_initialized) init_crc32_table();
    uint32_t crc = 0xFFFFFFFF;
    for (size_t i = 0; i < size; ++i) {
        crc = crc32_table[(crc ^ data[i]) & 0xFF] ^ (crc >> 8);
    }
    return crc ^ 0xFFFFFFFF;
}

// 从小端序字节读取 uint32_t
static uint32_t read_u32_le(const uint8_t* p) {
    return p[0] | (p[1] << 8) | (p[2] << 16) | (p[3] << 24);
}

int main(int argc, char* argv[]) {
    if (argc < 2) {
        std::cerr << "用法: " << argv[0] << " <file.pkn>\n";
        return 1;
    }

    // 读取文件头部（至少 0x1C = 28 字节）
    std::ifstream ifs(argv[1], std::ios::binary);
    if (!ifs) {
        std::cerr << "无法打开文件: " << argv[1] << "\n";
        return 1;
    }

    uint8_t header[28];
    ifs.read(reinterpret_cast<char*>(header), sizeof(header));
    if (ifs.gcount() < 28) {
        std::cerr << "文件太小，不是有效的 PKN 归档\n";
        return 1;
    }

    // (a) 检查魔数
    const uint8_t expected_magic[] = {0x50, 0x4B, 0x4E, 0x00};  // "PKN\0"
    if (memcmp(header, expected_magic, 4) != 0) {
        std::cerr << "魔数不匹配！预期 PKN\\0，实际: "
                  << std::hex
                  << (int)header[0] << " " << (int)header[1] << " "
                  << (int)header[2] << " " << (int)header[3] << "\n";
        return 1;
    }
    std::cout << "✅ 魔数验证通过: PKN\\0\n";

    // (c) 打印所有字段
    uint32_t version    = read_u32_le(header + 0x04);
    uint32_t num_entries = read_u32_le(header + 0x08);
    uint32_t data_offset = read_u32_le(header + 0x0C);
    uint32_t total_size  = read_u32_le(header + 0x10);
    uint32_t comp_size   = read_u32_le(header + 0x14);
    uint32_t stored_crc  = read_u32_le(header + 0x18);

    std::cout << "\n=== PKN 头部信息 ===\n";
    std::cout << "版本号:       " << version << "\n";
    std::cout << "文件条目数:   " << num_entries << "\n";
    std::cout << "数据偏移:     0x" << std::hex << data_offset << "\n";
    std::cout << "解压总大小:   " << std::dec << total_size << " 字节\n";
    std::cout << "压缩总大小:   " << comp_size << " 字节\n";

    if (total_size > 0) {
        double ratio = 100.0 * comp_size / total_size;
        std::cout << "压缩率:       " << ratio << "%\n";
    }

    // (b) 验证 CRC32
    uint32_t computed_crc = compute_crc32(header, 0x18);  // 前 24 字节
    std::cout << "\n存储的 CRC32: 0x" << std::hex << stored_crc << "\n";
    std::cout << "计算的 CRC32: 0x" << computed_crc << "\n";

    if (stored_crc == computed_crc) {
        std::cout << "✅ CRC32 验证通过\n";
    } else {
        std::cout << "❌ CRC32 不匹配！文件可能已损坏\n";
    }

    return 0;
}
```

编译和运行：

```bash
# Linux/macOS
g++ -std=c++17 -o pkn_parser pkn_header_parser.cpp
./pkn_parser sample.pkn

# Windows
cl /std:c++17 /EHsc pkn_header_parser.cpp /Fe:pkn_parser.exe
pkn_parser.exe sample.pkn
```

</details>

---

## 下一步

现在你已经掌握了 PackinOne 的整体架构和文件格式，下一节我们将深入压缩算法的核心：

→ [压缩算法还原与 C++ 实现](./02-压缩算法还原与C++实现.md)

你将学到：
- 从 Ghidra 反编译输出逐步还原 LZSS 压缩和解压的 C++ 代码
- 处理逆向过程中的典型难点（位操作、标志字节解析、滑动窗口管理）
- 建立差分测试环境，验证还原代码与原版 DLL 的输出一致性
- 将还原代码封装为项目可用的静态库

