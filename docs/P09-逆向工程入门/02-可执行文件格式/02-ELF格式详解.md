# 02 - ELF 格式详解

## 本節目標

- 理解 Linux ELF（Executable and Linkable Format）格式的整体结构
- 掌握 ELF 头、程序头表、节头表的字段含义
- 能够使用 readelf/objdump 等工具分析 ELF 文件
- 理解 ELF 与 PE 格式的核心差异，建立跨平台逆向视角

---

## 1. ELF 格式总览

ELF 是 Linux/Unix/BSD/Android 等平台上可执行文件、共享库、目标文件的标准格式。

```
ELF 文件整体结构：
┌──────────────────────────────┐  偏移 0x0000
│       ELF Header (64字节)     │
│  e_ident = "\x7fELF"         │
│  e_type / e_machine          │
│  e_phoff / e_shoff           │
├──────────────────────────────┤
│    Program Header Table       │
│  (程序头表 - 运行时视图)       │
│  ├── LOAD segment (代码)      │
│  ├── LOAD segment (数据)      │
│  ├── DYNAMIC segment          │
│  └── ...                      │
├──────────────────────────────┤
│       .text section           │
│     (机器码指令)               │
├──────────────────────────────┤
│       .rodata section         │
│     (只读数据/字符串常量)      │
├──────────────────────────────┤
│       .data section           │
│     (已初始化全局变量)         │
├──────────────────────────────┤
│       .bss section            │
│     (未初始化数据, 文件不占)   │
├──────────────────────────────┤
│       .symtab / .strtab       │
│     (符号表 / 字符串表)        │
├──────────────────────────────┤
│    Section Header Table       │
│  (节头表 - 链接时视图)         │
└──────────────────────────────┘
```

### ELF 的两种视图

```
链接视图 (Linking View)          运行视图 (Execution View)
┌──────────────┐               ┌──────────────┐
│  ELF Header  │               │  ELF Header  │
├──────────────┤               ├──────────────┤
│              │               │ Program Hdr  │
│  Sections    │               │   Table      │
│  (.text,     │               ├──────────────┤
│   .data,     │               │              │
│   .rodata,   │               │  Segments    │
│   .symtab,   │               │  (LOAD,      │
│   .strtab,   │               │   DYNAMIC,   │
│   ...)       │               │   ...)       │
│              │               │              │
├──────────────┤               ├──────────────┤
│ Section Hdr  │               │  (可选)       │
│   Table      │               │ Section Hdr  │
└──────────────┘               └──────────────┘

  用于编译链接                    用于程序加载执行
  节(Section)是基本单位            段(Segment)是基本单位
  静态分析常用                     动态分析常用
```

---

## 2. ELF Header

```c
// elf.h 中的定义 (64-bit)
typedef struct {
    unsigned char e_ident[16]; // ELF 标识
    Elf64_Half  e_type;        // 文件类型
    Elf64_Half  e_machine;     // CPU 架构
    Elf64_Word  e_version;     // ELF 版本
    Elf64_Addr  e_entry;       // ★ 入口点虚拟地址
    Elf64_Off   e_phoff;       // ★ 程序头表文件偏移
    Elf64_Off   e_shoff;       // ★ 节头表文件偏移
    Elf64_Word  e_flags;       // 处理器标志
    Elf64_Half  e_ehsize;      // ELF Header 大小
    Elf64_Half  e_phentsize;   // 程序头表项大小
    Elf64_Half  e_phnum;       // 程序头表项数
    Elf64_Half  e_shentsize;   // 节头表项大小
    Elf64_Half  e_shnum;       // 节头表项数
    Elf64_Half  e_shstrndx;    // 节名字符串表索引
} Elf64_Ehdr;
```

### 2.1 e_ident 标识字段

```
e_ident[16] 各字节含义：
┌──────┬─────────────────────────────────────┐
│ 偏移  │ 含义                                 │
├──────┼─────────────────────────────────────┤
│ [0-3]│ Magic: 0x7f, 'E', 'L', 'F'          │
│ [4]  │ Class: 1=32位(ELFCLASS32)            │
│      │        2=64位(ELFCLASS64)            │
│ [5]  │ Data:  1=小端(LSB)                   │
│      │        2=大端(MSB)                   │
│ [6]  │ Version: 1=当前版本                   │
│ [7]  │ OS/ABI: 0=UNIX System V              │
│      │         3=Linux                      │
│ [8-15]│ 填充（全零）                          │
└──────┴─────────────────────────────────────┘
```

### 2.2 e_type 和 e_machine

```c
// e_type 文件类型
// ET_NONE   0  无类型
// ET_REL    1  可重定位文件 (.o)
// ET_EXEC   2  可执行文件
// ET_DYN    3  共享对象 (.so) ★ 也用于PIE可执行文件
// ET_CORE   4  Core dump

// e_machine CPU 架构
// EM_386     3   Intel 80386 (x86 32位)
// EM_ARM    40   ARM
// EM_X86_64 62   AMD x86-64 (x64)
// EM_AARCH64 183 ARM 64位
// EM_RISCV  243  RISC-V
```

---

## 3. 程序头表（Program Header Table）

程序头表描述了 ELF 文件加载到内存时的**段（Segment）**布局。

```c
typedef struct {
    Elf64_Word  p_type;    // 段类型
    Elf64_Word  p_flags;   // 段标志（权限）
    Elf64_Off   p_offset;  // 文件偏移
    Elf64_Addr  p_vaddr;   // ★ 虚拟地址
    Elf64_Addr  p_paddr;   // 物理地址（通常=vaddr）
    Elf64_Xword p_filesz;  // 文件中的大小
    Elf64_Xword p_memsz;   // 内存中的大小 (≥filesz)
    Elf64_Xword p_align;   // 对齐
} Elf64_Phdr;

// p_type 常见值：
// PT_NULL    0  未使用
// PT_LOAD    1  ★ 可加载段（代码段/数据段）
// PT_DYNAMIC 2  ★ 动态链接信息
// PT_INTERP  3  动态链接器路径
// PT_NOTE    4  辅助信息
// PT_PHDR    6  程序头表自身
// PT_GNU_EH_FRAME  0x6474e550  异常处理帧
// PT_GNU_STACK     0x6474e551  栈权限
// PT_GNU_RELRO     0x6474e552  只读重定位

// p_flags 权限标志：
// PF_X  0x1  可执行
// PF_W  0x2  可写
// PF_R  0x4  可读
```

**典型段布局：**

```
readelf -l 输出示例：

Type      Offset    VirtAddr          FileSiz   MemSiz    Flg Align
PHDR      0x000040  0x0000000000400040 0x0001f8  0x0001f8  R   0x8
INTERP    0x000238  0x0000000000400238 0x00001c  0x00001c  R   0x1
LOAD      0x000000  0x0000000000400000 0x000a48  0x000a48  R E 0x200000
LOAD      0x000db8  0x0000000000600db8 0x000260  0x000268  RW  0x200000
DYNAMIC   0x000dc8  0x0000000000600dc8 0x0001f0  0x0001f0  RW  0x8
NOTE      0x000254  0x0000000000400254 0x000044  0x000044  R   0x4

段到节的映射：
  LOAD (RE):  .text .rodata .eh_frame      ← 代码段（可读可执行）
  LOAD (RW):  .data .bss .dynamic .got     ← 数据段（可读可写）
  DYNAMIC:    .dynamic                      ← 动态链接信息
```

---

## 4. 节头表（Section Header Table）

节头表是 ELF 的链接时视图，每个节包含特定类型的数据。

```c
typedef struct {
    Elf64_Word  sh_name;       // 节名（字符串表偏移）
    Elf64_Word  sh_type;       // 节类型
    Elf64_Xword sh_flags;      // 节标志
    Elf64_Addr  sh_addr;       // 内存地址
    Elf64_Off   sh_offset;     // 文件偏移
    Elf64_Xword sh_size;       // 节大小
    Elf64_Word  sh_link;       // 关联节索引
    Elf64_Word  sh_info;       // 附加信息
    Elf64_Xword sh_addralign;  // 对齐
    Elf64_Xword sh_entsize;    // 表项大小（若节是表）
} Elf64_Shdr;

// sh_type 常见值：
// SHT_NULL      0   空节
// SHT_PROGBITS  1   ★ 程序数据（.text/.rodata/.data）
// SHT_SYMTAB    2   ★ 符号表
// SHT_STRTAB    3   ★ 字符串表
// SHT_RELA      4   重定位表（含加数）
// SHT_HASH      5   符号哈希表
// SHT_DYNAMIC   6   ★ 动态链接信息
// SHT_NOTE      7   注释
// SHT_NOBITS    8   .bss（文件不占空间）
// SHT_REL       9   重定位表
// SHT_DYNSYM   11   ★ 动态符号表
```

### 常见节及其用途

```
┌──────────────┬──────────┬────────────────────────────────┐
│ 节名          │ 类型      │ 内容与逆向用途                   │
├──────────────┼──────────┼────────────────────────────────┤
│ .text        │ PROGBITS │ 机器码指令 → 主要分析目标         │
│ .rodata      │ PROGBITS │ 只读数据(字符串常量) → 字符串搜索 │
│ .data        │ PROGBITS │ 已初始化全局变量 → 全局状态分析    │
│ .bss         │ NOBITS   │ 未初始化全局变量                  │
│ .plt         │ PROGBITS │ 过程链接表 → 外部函数调用跳板      │
│ .got         │ PROGBITS │ 全局偏移表 → 动态链接关键          │
│ .got.plt     │ PROGBITS │ PLT 的 GOT 条目                  │
│ .dynsym      │ DYNSYM   │ 动态符号表 → 导出/导入符号        │
│ .dynstr      │ STRTAB   │ 动态字符串表 → 符号名             │
│ .symtab      │ SYMTAB   │ 完整符号表（strip后消失）         │
│ .strtab      │ STRTAB   │ 符号名字符串表                   │
│ .dynamic     │ DYNAMIC  │ 动态链接标签数组                  │
│ .rel/.rela   │ REL/RELA │ 重定位信息                       │
│ .init/.fini  │ PROGBITS │ 初始化/终止代码                   │
│ .debug_*     │ PROGBITS │ DWARF 调试信息                   │
│ .comment     │ PROGBITS │ 编译器版本字符串                  │
└──────────────┴──────────┴────────────────────────────────┘
```

---

## 5. 用 readelf/objdump 分析

### 5.1 readelf（ELF 专用分析工具）

```bash
# 查看 ELF 头
readelf -h target
# 输出：
#   Magic:   7f 45 4c 46 02 01 01 00 ...
#   Class:                             ELF64
#   Data:                              2's complement, little endian
#   Type:                              DYN (Shared object file)
#   Machine:                           Advanced Micro Devices X86-64
#   Entry point address:               0x1060

# 查看程序头表
readelf -l target

# 查看节头表
readelf -S target
# 输出：
#   [Nr] Name        Type      Address          Off    Size   ES Flg
#   [ 1] .interp     PROGBITS  0000000000000318 000318 00001c 00   A
#   [14] .text       PROGBITS  0000000000001060 001060 0001a5 00  AX
#   [16] .rodata     PROGBITS  0000000000002000 002000 000035 00   A
#   [24] .data       PROGBITS  0000000000004000 003000 000010 00  WA
#   [25] .bss        NOBITS    0000000000004010 003010 000008 00  WA

# 查看符号表
readelf -s target
# 输出：
#   Num:    Value          Size Type    Bind   Vis      Ndx Name
#     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS target.c
#    45: 0000000000001149    38 FUNC    GLOBAL DEFAULT   14 main
#    50: 0000000000001060     0 FUNC    GLOBAL DEFAULT   14 _start

# 查看动态段
readelf -d target
# 输出：
#   Tag        Type         Name/Value
#   0x0000001  (NEEDED)     Shared library: [libc.so.6]
#   0x000000c  (INIT)       0x1000
#   0x000000d  (FINI)       0x1208

# 查看重定位表
readelf -r target
```

### 5.2 objdump（反汇编与结构分析）

```bash
# 反汇编代码段
objdump -d target
# 输出：
# 0000000000001149 <main>:
#     1149:  55                  push   %rbp
#     114a:  48 89 e5            mov    %rsp,%rbp
#     114d:  48 83 ec 10         sub    $0x10,%rsp
#     ...

# 反汇编并显示源代码（需要调试信息）
objdump -d -S target

# 查看所有节的内容
objdump -s target

# 查看动态重定位
objdump -R target

# 只查看特定节
objdump -j .rodata -s target
```

### 5.3 nm（符号表查看）

```bash
# 查看符号表
nm target
# 输出：
# 0000000000004010 B __bss_start
# 0000000000001149 T main           # T = .text 段的全局符号
# 0000000000001129 T encode         # 函数
#                  U printf@@GLIBC  # U = 未定义（外部导入）

# 只显示外部符号
nm -D target.so

# 按地址排序
nm -n target

# C++ 符号反修饰
nm -C target
```

---

## 6. 用 Python 解析 ELF

```python
import struct

def parse_elf(filepath):
    """手动解析 ELF 文件核心结构"""
    with open(filepath, 'rb') as f:
        data = f.read()

    # === ELF Header ===
    magic = data[0:4]
    assert magic == b'\x7fELF', f"不是 ELF 文件: {magic}"

    ei_class = data[4]   # 1=32位, 2=64位
    ei_data = data[5]    # 1=小端, 2=大端
    is_64 = (ei_class == 2)
    endian = '<' if ei_data == 1 else '>'

    print(f"ELF Header:")
    print(f"  Class:    {'64-bit' if is_64 else '32-bit'}")
    print(f"  Endian:   {'Little' if ei_data == 1 else 'Big'}")

    if is_64:
        e_type, e_machine = struct.unpack_from(f'{endian}HH', data, 16)
        e_entry = struct.unpack_from(f'{endian}Q', data, 24)[0]
        e_phoff = struct.unpack_from(f'{endian}Q', data, 32)[0]
        e_shoff = struct.unpack_from(f'{endian}Q', data, 40)[0]
        e_phentsize, e_phnum = struct.unpack_from(f'{endian}HH', data, 54)
        e_shentsize, e_shnum, e_shstrndx = struct.unpack_from(
            f'{endian}HHH', data, 58)
    else:
        e_type, e_machine = struct.unpack_from(f'{endian}HH', data, 16)
        e_entry = struct.unpack_from(f'{endian}I', data, 24)[0]
        e_phoff = struct.unpack_from(f'{endian}I', data, 28)[0]
        e_shoff = struct.unpack_from(f'{endian}I', data, 32)[0]
        e_phentsize, e_phnum = struct.unpack_from(f'{endian}HH', data, 42)
        e_shentsize, e_shnum, e_shstrndx = struct.unpack_from(
            f'{endian}HHH', data, 46)

    type_names = {0: 'NONE', 1: 'REL', 2: 'EXEC', 3: 'DYN', 4: 'CORE'}
    machine_names = {3: 'i386', 40: 'ARM', 62: 'x86_64', 183: 'AArch64'}

    print(f"  Type:     {type_names.get(e_type, hex(e_type))}")
    print(f"  Machine:  {machine_names.get(e_machine, hex(e_machine))}")
    print(f"  Entry:    {hex(e_entry)}")
    print(f"  PH off:   {hex(e_phoff)} ({e_phnum} entries)")
    print(f"  SH off:   {hex(e_shoff)} ({e_shnum} entries)")

    # === 读取节名字符串表 ===
    shstrtab_off = e_shoff + e_shstrndx * e_shentsize
    if is_64:
        strtab_offset = struct.unpack_from(f'{endian}Q', data,
                                            shstrtab_off + 24)[0]
        strtab_size = struct.unpack_from(f'{endian}Q', data,
                                          shstrtab_off + 32)[0]
    else:
        strtab_offset = struct.unpack_from(f'{endian}I', data,
                                            shstrtab_off + 16)[0]
        strtab_size = struct.unpack_from(f'{endian}I', data,
                                          shstrtab_off + 20)[0]
    strtab = data[strtab_offset:strtab_offset + strtab_size]

    def get_string(offset):
        end = strtab.index(b'\x00', offset)
        return strtab[offset:end].decode('ascii')

    # === Section Headers ===
    print(f"\nSection Headers:")
    print(f"  {'Nr':>3s} {'Name':12s} {'Type':>10s} {'Addr':>18s} "
          f"{'Size':>10s} {'Flags':>5s}")

    for i in range(e_shnum):
        off = e_shoff + i * e_shentsize
        if is_64:
            sh_name = struct.unpack_from(f'{endian}I', data, off)[0]
            sh_type = struct.unpack_from(f'{endian}I', data, off + 4)[0]
            sh_flags = struct.unpack_from(f'{endian}Q', data, off + 8)[0]
            sh_addr = struct.unpack_from(f'{endian}Q', data, off + 16)[0]
            sh_size = struct.unpack_from(f'{endian}Q', data, off + 32)[0]
        else:
            sh_name = struct.unpack_from(f'{endian}I', data, off)[0]
            sh_type = struct.unpack_from(f'{endian}I', data, off + 4)[0]
            sh_flags = struct.unpack_from(f'{endian}I', data, off + 8)[0]
            sh_addr = struct.unpack_from(f'{endian}I', data, off + 12)[0]
            sh_size = struct.unpack_from(f'{endian}I', data, off + 20)[0]

        name = get_string(sh_name) if sh_name < len(strtab) else '?'
        type_map = {0:'NULL', 1:'PROGBITS', 2:'SYMTAB', 3:'STRTAB',
                    4:'RELA', 6:'DYNAMIC', 8:'NOBITS', 11:'DYNSYM'}
        flags_str = ''
        if sh_flags & 0x1: flags_str += 'W'
        if sh_flags & 0x2: flags_str += 'A'
        if sh_flags & 0x4: flags_str += 'X'

        print(f"  [{i:2d}] {name:12s} {type_map.get(sh_type, hex(sh_type)):>10s} "
              f"{hex(sh_addr):>18s} {hex(sh_size):>10s} {flags_str:>5s}")

# parse_elf('/usr/bin/ls')
```

---

## 7. PE 与 ELF 对比

```
┌─────────────────┬──────────────────┬──────────────────┐
│ 特性              │ PE (Windows)      │ ELF (Linux)       │
├─────────────────┼──────────────────┼──────────────────┤
│ Magic           │ "MZ" (0x5A4D)    │ "\x7fELF"        │
│ 入口定位         │ e_lfanew → PE头   │ 直接在 ELF 头中   │
│ 加载单位         │ Section           │ Segment (LOAD)   │
│ 地址基址         │ ImageBase (固定)  │ PIE 随机/固定     │
│ 动态链接         │ Import Table      │ .dynamic + GOT   │
│ 导出符号         │ Export Table      │ .dynsym          │
│ 重定位           │ .reloc 节         │ .rel/.rela 节    │
│ 调试信息         │ PDB (独立文件)    │ DWARF (嵌入/独立) │
│ DLL/SO 标志      │ Characteristics   │ e_type = ET_DYN  │
│ 延迟加载         │ Delay Import      │ dlopen/dlsym     │
│ 节对齐           │ SectionAlignment  │ p_align          │
│ 分析工具         │ dumpbin/PE-bear   │ readelf/objdump  │
│ 主力调试器       │ x64dbg/WinDbg     │ GDB/LLDB         │
└─────────────────┴──────────────────┴──────────────────┘
```

---

## 動手実践

用 readelf 分析一个共享库并提取关键信息：

```bash
# 步骤1: 编译一个简单的共享库用于练习
cat > mylib.c << 'EOF'
#include <stdio.h>

__attribute__((visibility("default")))
int my_add(int a, int b) {
    return a + b;
}

__attribute__((visibility("default")))
const char* my_version(void) {
    return "1.0.0";
}

// 内部函数（不导出）
static int helper(int x) {
    return x * 2;
}
EOF

gcc -shared -fPIC -o libmylib.so mylib.c

# 步骤2: 分析 ELF 头
readelf -h libmylib.so

# 步骤3: 查看节表
readelf -S libmylib.so

# 步骤4: 查看动态符号（导出函数）
readelf -s --dyn-syms libmylib.so | grep FUNC

# 步骤5: 查看依赖库
readelf -d libmylib.so | grep NEEDED

# 步骤6: 反汇编导出函数
objdump -d -j .text libmylib.so
```

---

## 対照項目源码

KiriKiri2 虽然是 Windows 平台项目（PE 格式），但交叉编译或 Linux 移植版本会使用 ELF：

```
跨平台对照：
  Windows 构建:  kirikiri2.exe (PE)  + plugin.dll (PE)
  Linux 移植:    kirikiri2 (ELF)     + plugin.so (ELF)

KiriKiri2 的跨平台构建系统：
  CMakeLists.txt 中通过 add_library(SHARED) 生成：
    Windows → .dll (PE, 导出 V2Link/V2Unlink)
    Linux   → .so  (ELF, 导出 V2Link/V2Unlink)

  插件入口在两个平台上的差异：
    PE:  __declspec(dllexport) HRESULT __stdcall V2Link(...)
    ELF: __attribute__((visibility("default"))) int V2Link(...)
```

---

## 常见错误与解决

### 错误1：readelf 显示 "no sections" 但程序正常运行

```
原因：节头表被 strip 或故意删除（反分析技巧）
解决：
  1. ELF 运行只依赖程序头表，节头表可以完全删除
  2. 使用 readelf -l 查看程序头表（仍然存在）
  3. 使用工具重建节头表：
     - Ghidra 会自动根据段信息推断节
     - 手动根据 LOAD 段的权限划分代码段和数据段
```

### 错误2：strip 后符号信息全部丢失

```
原因：strip 命令删除了 .symtab 和 .strtab
解决：
  1. 动态符号(.dynsym)不受 strip 影响（共享库必需）
  2. readelf -s --dyn-syms 查看动态符号
  3. 静态链接的程序被 strip 后几乎无符号
     → 只能依赖反汇编器的函数签名识别（FLIRT等）
  4. 如有 .gnu_debuglink 或 .note.gnu.build-id
     → 可能有独立的调试符号文件
```

### 错误3：PIE 可执行文件地址每次不同

```
原因：Position Independent Executable 启用 ASLR
解决：
  1. GDB 中使用 "info proc mappings" 查看实际加载地址
  2. 逆向分析时使用文件偏移或相对偏移
  3. 临时关闭 ASLR 便于调试：
     echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
     调试完毕后恢复：
     echo 2 | sudo tee /proc/sys/kernel/randomize_va_space
```

---

## 本節小結

| ELF 组件 | 关键内容 | 逆向用途 |
|---------|---------|---------|
| ELF Header | e_type, e_machine, e_entry | 文件类型、架构、入口点 |
| Program Headers | LOAD, DYNAMIC segments | 内存布局、加载信息 |
| Section Headers | .text, .rodata, .data | 代码/数据区域定位 |
| .dynsym / .dynstr | 动态符号表 | 导出/导入函数 |
| .plt / .got | 间接调用表 | 动态链接调用分析 |

核心认知：
- ELF 有**两种视图**：运行时用段（Segment），分析时用节（Section）
- 与 PE 不同，ELF 的符号信息可被完全 strip
- .so 共享库的逆向与 .dll 类似，但导入/导出机制不同（GOT/PLT vs IAT）
- 现代 Linux 可执行文件几乎都是 PIE（ET_DYN），需要注意 ASLR

---

## 練習題与答案

### 练习1：ELF 文件类型判断

**题目**：用 readelf -h 看到以下信息，判断文件类型和特征：
- (A) Type: EXEC, Machine: Advanced Micro Devices X86-64
- (B) Type: DYN, Machine: Intel 80386
- (C) Type: REL, Machine: ARM

<details>
<summary>参考答案</summary>

```
(A) Type: EXEC, Machine: x86_64
    → 64位 x86 可执行文件（非PIE）
    → 固定加载地址，无ASLR
    → 较老的编译方式（现代默认PIE即DYN）

(B) Type: DYN, Machine: i386
    → 32位 x86 共享库(.so) 或 PIE 可执行文件
    → 需要检查是否有 INTERP 段来区分：
      有 INTERP → PIE 可执行文件
      无 INTERP → 共享库

(C) Type: REL, Machine: ARM
    → ARM 架构的可重定位目标文件 (.o)
    → 编译但未链接的中间产物
    → 不能直接执行，需要链接器处理
```

</details>

### 练习2：PE 与 ELF 导入机制对比

**题目**：说明 Windows DLL 和 Linux .so 在导入外部函数时的机制差异，以 `printf` 为例。

<details>
<summary>参考答案</summary>

```
=== Windows PE 导入 printf ===
1. 编译时：编译器生成 call [IAT_entry] 间接调用
2. PE 文件中：
   - Import Table 记录: 依赖 msvcrt.dll 的 printf
   - IAT (Import Address Table) 预留一个条目
3. 加载时：Windows loader 解析 printf 地址，写入 IAT
4. 运行时：call [0x00402000] → 读取 IAT → 跳转到 printf

=== Linux ELF 导入 printf ===
1. 编译时：编译器生成 call printf@plt
2. ELF 文件中：
   - .dynamic 段记录: NEEDED libc.so.6
   - .plt 中有 printf 的跳板代码
   - .got.plt 中预留一个条目
3. 首次调用（延迟绑定）：
   call printf@plt
   → .plt 跳板跳到 .got.plt 条目
   → 第一次: 跳到动态链接器 _dl_runtime_resolve
   → 动态链接器解析 printf 真实地址
   → 写入 .got.plt 条目
4. 后续调用：
   call printf@plt → .got.plt → 直接跳到 printf（已缓存）

关键差异：
  PE:  加载时一次性解析所有导入（eager binding）
  ELF: 默认延迟绑定（lazy binding），首次调用时才解析
       可用 LD_BIND_NOW=1 强制立即绑定
```

</details>

---

## 下一步

下一节 [03-导入导出表与动态链接](./03-导入导出表与动态链接.md) 将深入分析 PE 和 ELF 的导入/导出机制，这是逆向 DLL/SO 的核心知识。
