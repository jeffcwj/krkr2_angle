# GenNodeCode 与操作码生成

> **所属模块：** M07-TJS2 脚本引擎
> **前置知识：** [AST 构建与节点类型](../03-语法分析器/02-AST构建与节点类型.md)、[错误恢复与预处理](../03-语法分析器/03-错误恢复与预处理.md)
> **预计阅读时间：** 60 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：
1. 理解 `GenNodeCode` 函数的整体架构——一个巨型 switch 如何将 AST 节点翻译为 VM 操作码
2. 掌握 TJS2 全部 VM 操作码（`tTJSVMCodes` 枚举）的分类与用途
3. 追踪一条 TJS2 表达式从 AST 到字节码序列的完整翻译过程
4. 理解 `PutCode` / `PutData` 两个核心发射函数的工作原理
5. 解释操作码的"属性访问变体"机制——`TJS_NORMAL_AND_PROPERTY_ACCESSER` 宏如何生成 4 个版本的操作码

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 操作码 | Opcode (Operation Code) | VM 能识别的最小指令单元，每个操作码对应一种操作（如加法、跳转） |
| 字节码 | Bytecode | 由操作码和操作数组成的指令序列，是 AST 编译后的中间产物，供 VM 执行 |
| 代码区 | Code Area | 存储编译后字节码的 `tjs_int32` 数组，每个元素占 4 字节 |
| 数据区 | Data Area | 存储常量值（字符串、数字等）的 `tTJSVariant` 指针数组 |
| 寄存器地址 | Register Address | VM 用字节偏移表示的寄存器位置，通过 `TJS_TO_VM_REG_ADDR` 宏从逻辑编号转换 |
| 属性访问变体 | Property Accessor Variant | 同一操作的 4 个版本（普通/直接属性/间接属性/属性对象），用于处理属性赋值时的操作 |
| 条件标志 | Condition Flag | VM 内部的布尔状态位，比较指令设置它，条件跳转指令读取它 |

---

## §1 从 AST 到字节码：编译流水线全景

### 1.1 编译流水线回顾

在前几节中，我们学习了 TJS2 的词法分析和语法分析。现在来到编译流水线的第三个阶段——**字节码生成**（Bytecode Generation）。整个流程如下：

```
TJS2 源码文本
    │
    ▼
┌──────────────┐
│  词法分析器   │  tjsLex.cpp — 将字符流切分为 Token 序列
│  (Lexer)     │
└──────┬───────┘
       │ Token 流
       ▼
┌──────────────┐
│  语法分析器   │  tjs.y (Bison) — 将 Token 序列组装为 AST
│  (Parser)    │
└──────┬───────┘
       │ AST (tTJSExprNode 树)
       ▼
┌──────────────────────┐
│  字节码编译器          │  tjsInterCodeGen.cpp — GenNodeCode()
│  (Bytecode Compiler) │  将 AST 节点递归翻译为 VM 操作码序列
└──────┬───────────────┘
       │ CodeArea (tjs_int32[]) + DataArea (tTJSVariant[])
       ▼
┌──────────────┐
│  虚拟机执行器  │  tjsInterCodeExec.cpp — switch 分发执行
│  (VM)        │
└──────────────┘
```

字节码编译器的核心是 `tTJSInterCodeContext` 类中的 `GenNodeCode` 函数。这个函数接收一棵 AST 子树，递归遍历每个节点，为每种节点类型生成对应的 VM 指令序列。生成的指令存储在两个线性数组中：

- **CodeArea**（代码区）：`tjs_int32[]` 数组，存储操作码和操作数，每个元素 4 字节
- **_DataArea → DataArea**（数据区）：`tTJSVariant*[]` 数组，存储常量值（字符串字面量、数字字面量等）

### 1.2 tTJSInterCodeContext 的角色

`tTJSInterCodeContext`（以下简称"编译上下文"）是 TJS2 编译器的核心类。每个函数、每个类的每个方法、每段顶层代码都会创建一个独立的编译上下文。编译上下文负责：

1. **管理代码区和数据区** —— 通过 `PutCode()` 和 `PutData()` 向两个区域追加数据
2. **管理寄存器帧** —— 跟踪临时寄存器的分配与释放（`frame` 变量）
3. **管理局部变量命名空间** —— 通过 `Namespace` 对象记录局部变量名到索引的映射
4. **管理跳转修补** —— 记录需要回填的跳转地址（`JumpList`）
5. **递归翻译 AST** —— `GenNodeCode()` 是翻译的主入口

编译上下文在 `tjsInterCodeGen.h` 中定义，关键成员变量如下：

```cpp
// 源码位置：cpp/core/tjs2/tjsInterCodeGen.h
class tTJSInterCodeContext : public tTJSInterCodeObject {
    // 代码区 —— 存储编译后的操作码序列
    tjs_int32 *CodeArea;        // 操作码数组指针
    tjs_int CodeAreaSize;       // 当前已使用的元素数
    tjs_int CodeAreaCapa;       // 已分配的容量

    // 数据区 —— 存储常量值
    tTJSVariant **_DataArea;    // 编译期使用的指针数组（可去重）
    tjs_int _DataAreaSize;      // 编译期数据区大小

    tTJSVariant *DataArea;      // 最终打包后的值数组（Commit 时从 _DataArea 拷贝）

    // 寄存器管理
    tjs_int FrameBase;          // 帧基址，初始化为 1（0 号寄存器预留给 this）

    // 源码位置追踪
    tSourcePos *SourcePosArray; // 操作码位置 → 源码位置的映射表
    tjs_int SourcePosArraySize;
    // ...
};
```

> **关键发现**：`FrameBase` 初始化为 1（构造函数第 324 行），这意味着 0 号寄存器被预留给 `this` 对象。所有临时值从 1 号寄存器开始分配。

### 1.3 编译触发点

编译不是直接调用 `GenNodeCode` 开始的。实际调用链是：

```
tTJSScriptBlock::SetText()          // 设置待编译的脚本文本
  → tTJSScriptBlock::Parse()        // 调用 Bison 解析器
    → parser 归约 action 中调用:
      tTJSInterCodeContext::CreateExprCode(node)   // 第 2913 行
        → GenNodeCode(frame, node, ...)            // 第 1110 行
```

`CreateExprCode` 是 AST 到字节码的桥梁函数（第 2913-2918 行），它的实现非常简洁：

```cpp
// 源码位置：cpp/core/tjs2/tjsInterCodeGen.cpp 第 2913-2918 行
void tTJSInterCodeContext::CreateExprCode(tTJSExprNode *node) {
    // 从 FrameBase 开始分配临时寄存器
    tjs_int frame = FrameBase;
    // 翻译整棵 AST 子树
    GenNodeCode(frame, node, 0, 0, tSubParam());
    // 更新 FrameBase（frame 在翻译过程中可能增长）
    if(FrameBase < frame) FrameBase = frame;
}
```

注意 `restype=0` 表示不要求返回值——顶层语句通常不需要结果。

---

## §2 PutCode 与 PutData：字节码发射双引擎

在深入 GenNodeCode 之前，必须先理解它依赖的两个底层函数——`PutCode` 和 `PutData`。它们是字节码发射的"双引擎"，GenNodeCode 的所有输出最终都通过这两个函数写入代码区和数据区。

### 2.1 PutCode —— 向代码区追加一个 32 位字

`PutCode` 将一个 `tjs_int32` 值追加到 `CodeArea` 数组末尾，并返回它在数组中的位置索引。

```cpp
// 源码位置：cpp/core/tjs2/tjsInterCodeGen.cpp 第 584-632 行
tjs_int tTJSInterCodeContext::PutCode(tjs_int32 num, tjs_int32 pos) {
    // num = 要写入的值（操作码或操作数）
    // pos = 该指令在源码中的字节偏移位置（-1 表示不记录）
    // 返回值：新写入的代码在 CodeArea 中的索引

    // 容量不足时以 256 为增量扩容
    if(CodeAreaSize >= CodeAreaCapa) {
        CodeArea = (tjs_int32 *)TJS_realloc(
            CodeArea, sizeof(tjs_int32) * (CodeAreaCapa + TJS_INC_SIZE));
        // TJS_INC_SIZE = 256
        if(!CodeArea)
            TJS_eTJSScriptError(TJSInsufficientMem, Block, pos);
        CodeAreaCapa += TJS_INC_SIZE;
    }

    // 记录源码位置映射（仅当 pos != -1 且位置变化时）
    if(pos != -1) {
        if(PrevSourcePos != pos) {
            PrevSourcePos = pos;
            SourcePosArraySorted = false;
            // ... 分配/扩容 SourcePosArray ...
            SourcePosArray[SourcePosArraySize].CodePos = CodeAreaSize;
            SourcePosArray[SourcePosArraySize].SourcePos = pos;
            SourcePosArraySize++;
        }
    }

    // 写入值并递增大小
    CodeArea[CodeAreaSize] = num;
    return CodeAreaSize++;
}
```

**关键设计点**：

1. **统一的 32 位编码**：操作码和操作数都是 `tjs_int32`，在 CodeArea 中不做区分。VM 根据操作码的定义知道后面跟几个操作数。
2. **增量扩容**：以 `TJS_INC_SIZE = 256` 个元素为增量扩容，而非倍增。对于脚本编译场景这个策略足够高效。
3. **源码位置追踪**：`SourcePosArray` 记录了"第 N 条指令对应源码第 M 字节"的映射关系。这个映射在运行时报错时用于定位源码行号。注意它只在 `pos` 发生变化时才记录一条新映射——连续来自同一源码位置的多条指令共享一条映射记录。

### 2.2 PutData —— 向数据区追加一个常量值

`PutData` 将一个 `tTJSVariant` 值存入 `_DataArea` 数组，并返回该值的索引。它有一个重要的优化——**去重**（deduplication）。

```cpp
// 源码位置：cpp/core/tjs2/tjsInterCodeGen.cpp 第 638-682 行
tjs_int tTJSInterCodeContext::PutData(const tTJSVariant &val) {
    // 容量不足时扩容（同样以 256 为增量）
    if(_DataAreaSize >= _DataAreaCapa) {
        _DataArea = (tTJSVariant **)TJS_realloc(
            _DataArea,
            sizeof(tTJSVariant *) * (_DataAreaCapa + TJS_INC_SIZE));
        if(!_DataArea)
            TJS_eTJSScriptError(TJSInsufficientMem, Block, LEX_POS);
        _DataAreaCapa += TJS_INC_SIZE;
    }

    // 在最近 20 个数据项中搜索相同值（去重优化）
    if(_DataAreaSize) {
        tTJSVariant **ptr = _DataArea + _DataAreaSize - 1;
        tjs_int count = 0;
        while(count < 20) {  // 只回溯最近 20 个条目
            if((*ptr)->DiscernCompareStrictReal(val)) {
                return (tjs_int)(ptr - _DataArea);  // 找到相同值，复用
            }
            count++;
            if(ptr == _DataArea) break;
            ptr--;
        }
    }

    // 未找到重复值，创建新条目
    tTJSVariant *v;
    if(val.Type() == tvtString) {
        // 字符串走全局字符串映射池（intern），共享内存
        v = new tTJSVariant(TJSMapGlobalStringMap(val));
    } else {
        v = new tTJSVariant(val);
    }
    _DataArea[_DataAreaSize] = v;
    return _DataAreaSize++;
}
```

**关键设计点**：

1. **窗口去重**：只搜索最近 20 个条目，而非全部数据区。这是时间/空间权衡——完整扫描太慢，20 个窗口已能覆盖大部分局部重复（如循环体中反复使用同一字符串属性名）。原始注释写道："is waste of time if it exceeds 20 limit?"，作者自己也在思考这个阈值是否合适。
2. **字符串 intern**：字符串类型的常量通过 `TJSMapGlobalStringMap` 走全局字符串池（Global String Map），避免相同字符串字面量在不同编译上下文中重复存储。
3. **严格比较**：`DiscernCompareStrictReal` 进行严格实值比较（区分 `1` 和 `1.0`），确保类型和值都匹配才算相同。

### 2.3 PutCode 与 PutData 的协作模式

来看一个典型的协作场景——编译常量值节点 `T_CONSTVAL`：

```cpp
// 将常量值 42 加载到寄存器
tjs_int dp = PutData(node->GetValue());  // ① 将 42 存入数据区，得到索引 dp
PutCode(VM_CONST, node_pos);             // ② 发射操作码 VM_CONST
PutCode(TJS_TO_VM_REG_ADDR(frame), node_pos);  // ③ 发射目标寄存器地址
PutCode(TJS_TO_VM_REG_ADDR(dp), node_pos);     // ④ 发射数据区索引
return frame++;  // ⑤ 分配下一个临时寄存器
```

生成的指令在 CodeArea 中的布局：

```
CodeArea[n]   = VM_CONST          // 操作码
CodeArea[n+1] = frame * 16       // 目标寄存器（字节偏移）
CodeArea[n+2] = dp * 16          // 数据区索引（字节偏移）
                                  // 16 = sizeof(tTJSVariant)
```

> **为什么要乘以 sizeof(tTJSVariant)?**  `TJS_TO_VM_REG_ADDR(x)` 宏将逻辑寄存器编号转换为字节偏移：`(x) * sizeof(tTJSVariant)`。因为 `tTJSVariant` 结构体大小为 16 字节，所以逻辑编号 1 对应字节偏移 16，编号 2 对应 32，以此类推。VM 执行时通过 `TJS_GET_VM_REG(base, offset)` 宏直接用字节偏移做指针运算，避免了运行时乘法。

---

## §3 VM 操作码全表：tTJSVMCodes 枚举

TJS2 VM 的全部操作码定义在 `tjsInterCodeGen.h` 第 44-131 行的 `tTJSVMCodes` 枚举中。总共约 **87 个**操作码（含属性访问变体）。我们按功能分组详细讲解。

### 3.1 基础操作码（无属性变体）

这类操作码只有一个版本，不涉及属性访问的特殊处理：

| 操作码 | 编号 | 操作数 | 含义 | 指令格式 |
|--------|------|--------|------|----------|
| `VM_NOP` | 0 | 无 | 空操作（No Operation），占位用 | `NOP` |
| `VM_CONST` | 1 | reg, data | 将数据区的常量值加载到寄存器 | `CONST %reg, *data` |
| `VM_CP` | 2 | dst, src | 复制寄存器值（Copy） | `CP %dst, %src` |
| `VM_CL` | 3 | reg | 清空寄存器（Clear），将其设为 void | `CL %reg` |
| `VM_CCL` | 4 | start, count | 批量清空连续寄存器 | `CCL %start, count` |
| `VM_TT` | 5 | reg | 测试真值（Test True），将寄存器值转为条件标志 | `TT %reg` |
| `VM_TF` | 6 | reg | 测试假值（Test False），反向转换 | `TF %reg` |
| `VM_CEQ` | 7 | reg1, reg2 | 普通相等比较，结果存条件标志 | `CEQ %r1, %r2` |
| `VM_CDEQ` | 8 | reg1, reg2 | 严格相等比较（Discerning Equal），区分类型 | `CDEQ %r1, %r2` |
| `VM_CLT` | 9 | reg1, reg2 | 小于比较 | `CLT %r1, %r2` |
| `VM_CGT` | 10 | reg1, reg2 | 大于比较 | `CGT %r1, %r2` |
| `VM_SETF` | 11 | reg | 将条件标志的值存入寄存器（Set from Flag） | `SETF %reg` |
| `VM_SETNF` | 12 | reg | 将条件标志的**反值**存入寄存器 | `SETNF %reg` |
| `VM_LNOT` | 13 | reg | 逻辑非（Logical NOT），原地修改寄存器 | `LNOT %reg` |
| `VM_NF` | 14 | 无 | 翻转条件标志（Negate Flag） | `NF` |
| `VM_JF` | 15 | offset | 条件标志为真时跳转（Jump if Flag） | `JF offset` |
| `VM_JNF` | 16 | offset | 条件标志为假时跳转（Jump if Not Flag） | `JNF offset` |
| `VM_JMP` | 17 | offset | 无条件跳转 | `JMP offset` |

> **条件标志系统**：TJS2 VM 使用独立的条件标志位（类似 CPU 的 FLAGS 寄存器）来传递比较结果。比较指令（`VM_CEQ`/`VM_CLT`/`VM_CGT`）设置条件标志，跳转指令（`VM_JF`/`VM_JNF`）读取它。这种设计避免了为每个比较操作分配临时寄存器来存储布尔结果。`VM_SETF`/`VM_SETNF` 用于在需要将条件标志"物化"为寄存器值时使用（例如 `var x = (a == b)`）。

### 3.2 属性访问变体操作码

这是 TJS2 操作码设计中最精巧的部分。通过 `TJS_NORMAL_AND_PROPERTY_ACCESSER(x)` 宏，每个算术/逻辑操作码自动展开为 **4 个版本**：

```cpp
// 源码位置：cpp/core/tjs2/tjsInterCodeGen.h 第 42 行
#define TJS_NORMAL_AND_PROPERTY_ACCESSER(x) x, x##PD, x##PI, x##P
```

以 `VM_ADD` 为例，宏展开为：

| 变体 | 名称 | 用途 | 操作数格式 |
|------|------|------|-----------|
| `VM_ADD` | 普通 | 两个寄存器相加 | `ADD %dst, %src` |
| `VM_ADDPD` | 直接属性（Property Direct） | 通过字面名访问属性并加 | `ADDPD %result, %obj, *name, %src` |
| `VM_ADDPI` | 间接属性（Property Indirect） | 通过寄存器值访问属性并加 | `ADDPI %result, %obj, %name, %src` |
| `VM_ADDP` | 属性对象（Property object） | 对属性对象本身执行加法 | `ADDP %result, %prop, %src` |

**为什么需要 4 个版本？** 考虑 TJS2 的 `+=` 运算符在不同场景下的行为：

```javascript
// 场景 1: 普通变量 — 使用 VM_ADD
var x = 10;
x += 5;
// 编译为: ADD %x, %5

// 场景 2: 对象的命名属性 — 使用 VM_ADDPD
obj.count += 1;
// 编译为: ADDPD %result, %obj, "count", %1
// 等价于: obj.count = obj.count + 1，但只访问一次属性

// 场景 3: 对象的动态属性 — 使用 VM_ADDPI
obj[key] += 1;
// 编译为: ADDPI %result, %obj, %key, %1

// 场景 4: 属性对象本身 — 使用 VM_ADDP
*prop += 1;
// 编译为: ADDP %result, %prop, %1
```

带属性变体的完整操作码列表（每行展开为 4 个）：

| 基础操作码 | 含义 | 展开后的变体 |
|-----------|------|-------------|
| `VM_INC` | 自增 | `VM_INC`, `VM_INCPD`, `VM_INCPI`, `VM_INCP` |
| `VM_DEC` | 自减 | `VM_DEC`, `VM_DECPD`, `VM_DECPI`, `VM_DECP` |
| `VM_LOR` | 逻辑或赋值 | `VM_LOR`, `VM_LORPD`, `VM_LORPI`, `VM_LORP` |
| `VM_LAND` | 逻辑与赋值 | `VM_LAND`, `VM_LANDPD`, `VM_LANDPI`, `VM_LANDP` |
| `VM_BOR` | 按位或 | `VM_BOR`, `VM_BORPD`, `VM_BORPI`, `VM_BORP` |
| `VM_BXOR` | 按位异或 | `VM_BXOR`, `VM_BXORPD`, `VM_BXORPI`, `VM_BXORP` |
| `VM_BAND` | 按位与 | `VM_BAND`, `VM_BANDPD`, `VM_BANDPI`, `VM_BANDP` |
| `VM_SAR` | 算术右移 | `VM_SAR`, `VM_SARPD`, `VM_SARPI`, `VM_SARP` |
| `VM_SAL` | 算术左移 | `VM_SAL`, `VM_SALPD`, `VM_SALPI`, `VM_SALP` |
| `VM_SR` | 逻辑右移 | `VM_SR`, `VM_SRPD`, `VM_SRPI`, `VM_SRP` |
| `VM_ADD` | 加法 | `VM_ADD`, `VM_ADDPD`, `VM_ADDPI`, `VM_ADDP` |
| `VM_SUB` | 减法 | `VM_SUB`, `VM_SUBPD`, `VM_SUBPI`, `VM_SUBP` |
| `VM_MOD` | 取模 | `VM_MOD`, `VM_MODPD`, `VM_MODPI`, `VM_MODP` |
| `VM_DIV` | 浮点除 | `VM_DIV`, `VM_DIVPD`, `VM_DIVPI`, `VM_DIVP` |
| `VM_IDIV` | 整数除 | `VM_IDIV`, `VM_IDIVPD`, `VM_IDIVPI`, `VM_IDIVP` |
| `VM_MUL` | 乘法 | `VM_MUL`, `VM_MULPD`, `VM_MULPI`, `VM_MULP` |

> **编码技巧**：4 个变体的枚举值是**连续的**。基础操作码 +1 = PD 变体，+2 = PI 变体，+3 = P 变体。GenNodeCode 利用这一点直接用算术计算变体编号：`PutCode((tjs_int32)param.SubType + (direct ? 1 : 2), ...)` ——其中 `direct` 表示是直接属性访问（`.`）还是间接属性访问（`[]`）。这个巧妙的位移设计见 `tjsInterCodeGen.cpp` 第 2081 行。

### 3.3 一元与类型操作码

```
VM_BNOT    — 按位取反 (~)
VM_TYPEOF  — 获取值类型 (typeof)
VM_TYPEOFD — 获取直接属性的类型 (typeof obj.prop)
VM_TYPEOFI — 获取间接属性的类型 (typeof obj[key])
VM_EVAL    — 求值运算符 (后置 !)，类似 JavaScript 的 eval
VM_EEXP    — 求值但不返回结果（表达式上下文的 eval）
VM_CHKINS  — instanceof 检查
VM_ASC     — 字符到 ASCII 码 (#)
VM_CHR     — ASCII 码到字符 ($)
VM_NUM     — 转为数字 (一元 +)
VM_CHS     — 取负 (一元 -)
VM_INV     — invalidate 运算符（释放对象）
VM_CHKINV  — isvalid 检查
VM_INT     — 强制转为整数类型
VM_REAL    — 强制转为实数类型
VM_STR     — 强制转为字符串类型
VM_OCTET   — 强制转为八位字节串类型
```

### 3.4 函数调用与对象操作码

| 操作码 | 操作数 | 含义 |
|--------|--------|------|
| `VM_CALL` | result, func, args... | 调用闭包/函数 |
| `VM_CALLD` | result, obj, name, args... | 调用对象的命名方法（Direct） |
| `VM_CALLI` | result, obj, key, args... | 调用对象的动态方法（Indirect） |
| `VM_NEW` | result, class, args... | 创建新对象（new 运算符） |
| `VM_GPD` | result, obj, name | 获取直接属性（Get Property Direct） |
| `VM_SPD` | obj, name, value | 设置直接属性（Set Property Direct），this 上下文 |
| `VM_SPDE` | obj, name, value | 设置直接属性，非 this 上下文 |
| `VM_SPDEH` | obj, name, value | 设置直接属性（带隐藏标志） |
| `VM_GPI` | result, obj, key | 获取间接属性（Get Property Indirect） |
| `VM_SPI` | obj, key, value | 设置间接属性 |
| `VM_SPIE` | obj, key, value | 设置间接属性，非 this 上下文 |
| `VM_GPDS` | result, obj, name | 获取直接属性（忽略 getter，直接取值） |
| `VM_SPDS` | obj, name, value | 设置直接属性（忽略 setter，直接赋值） |
| `VM_GPIS` | result, obj, key | 获取间接属性（忽略 getter） |
| `VM_SPIS` | obj, key, value | 设置间接属性（忽略 setter） |
| `VM_SETP` | prop, value | 通过属性对象写入值 |
| `VM_GETP` | result, prop | 通过属性对象读取值 |
| `VM_DELD` | result, obj, name | 删除直接属性 |
| `VM_DELI` | result, obj, key | 删除间接属性 |

> **SPD vs SPDE 的区别**：当赋值目标是 `this` 的属性时（如 `this.x = 1`），使用 `VM_SPD`；当目标是其他对象的属性时（如 `obj.x = 1`），使用 `VM_SPDE`。两者的区别在于 VM 执行时传递的 `objthis` 参数不同。见 `tjsInterCodeGen.cpp` 第 2054-2058 行的判断逻辑。

### 3.5 控制流与特殊操作码

| 操作码 | 操作数 | 含义 |
|--------|--------|------|
| `VM_SRV` | reg | 设置返回值（Set Return Value） |
| `VM_RET` | 无 | 函数返回 |
| `VM_ENTRY` | try_offset, catch_offset | 进入 try 块（设置异常处理点） |
| `VM_EXTRY` | 无 | 退出 try 块 |
| `VM_THROW` | reg | 抛出异常 |
| `VM_CHGTHIS` | closure, new_this | 更改闭包的 this 上下文（incontextof） |
| `VM_GLOBAL` | reg | 获取全局对象引用 |
| `VM_ADDCI` | reg, class_info | 添加类信息（继承链注册） |
| `VM_REGMEMBER` | 无 | 注册成员（类构造时批量注册属性/方法） |
| `VM_DEBUGGER` | 无 | 调试器断点 |

---

## §4 GenNodeCode 深度解析：switch 分发架构

### 4.1 函数签名与参数含义

`GenNodeCode` 是一个 **2600+ 行**的巨型函数（第 1110-3767 行），通过一个 switch 语句覆盖所有 AST 节点类型。先看它的签名：

```cpp
// 源码位置：cpp/core/tjs2/tjsInterCodeGen.cpp 第 1110 行
tjs_int tTJSInterCodeContext::GenNodeCode(
    tjs_int &frame,           // [in/out] 当前寄存器帧指针
    tTJSExprNode *node,       // [in]     要翻译的 AST 节点
    tjs_uint32 restype,       // [in]     期望的结果类型标志
    tjs_int reqresaddr,       // [in]     指定结果寄存器地址（目前未使用）
    const tSubParam &param    // [in]     附加参数（赋值子类型等）
)
// 返回值：包含结果的寄存器地址
//         特殊值 TJS_GNC_CFLAG 表示结果在条件标志中
//         特殊值 TJS_GNC_CFLAG_I 表示结果在条件标志的反值中
```

各参数的详细解释：

**`frame`（寄存器帧指针）**：这是一个引用参数，指向下一个可用的临时寄存器编号。每次 GenNodeCode 需要一个临时寄存器来存储中间结果时，它使用当前 frame 值并将 frame 加 1。由于是引用，递归调用结束后调用者能看到 frame 的新值。

**`restype`（结果类型标志）**：位标志，控制 GenNodeCode 的行为：
- `TJS_RT_NEEDED` (0x01)：调用者需要这个节点的值。如果不设置，GenNodeCode 只产生副作用（如赋值）而不保存结果。
- `TJS_RT_CFLAG` (0x02)：调用者希望结果以条件标志形式返回，而非存入寄存器。这在 if/while 的条件表达式中使用，避免不必要的"条件标志 → 寄存器 → 条件标志"往返转换。

**`param`（附加参数）**：`tSubParam` 结构体，包含：
- `SubType`：赋值子类型，如 `stEqual`（直接赋值）、`stAdd`（加法赋值）、`stPreInc`（前置自增）等
- `SubAddress`：赋值源的寄存器地址

这个参数实现了 TJS2 的一个巧妙设计：**赋值是"反向传播"的**。当遇到 `x = expr` 时，编译器先编译 `expr` 得到结果寄存器 `resaddr`，然后用 `param2.SubType = stEqual; param2.SubAddress = resaddr` 再次调用 GenNodeCode 处理 `x` 节点——这次 `x` 节点知道自己要被赋值，生成对应的写入指令。

### 4.2 辅助函数与辅助宏

在 GenNodeCode 上方定义了几个辅助函数和宏：

```cpp
// 第 1094 行 — _GenNodeCode 是 GenNodeCode 的包装，统计调用次数
// 在 DEBUG 模式下用于性能分析

// 第 1100 行 — 判断返回值是否为条件标志
static bool inline TJSIsCondFlagRetValue(tjs_int r) {
    return r == TJS_GNC_CFLAG || r == TJS_GNC_CFLAG_I;
}

// 第 1104 行 — 判断返回值是否为临时帧寄存器
static bool inline TJSIsFrame(tjs_int r) {
    return r > 0;  // 正数 = 帧寄存器，负数 = 局部变量，0 = this
}
```

`TJSIsFrame` 的返回值含义：
- **正数**（> 0）：值在临时帧寄存器中（由 frame 分配）
- **负数**（< 0）：值在局部变量寄存器中（编码为 `-n - VariableReserveCount - 1`）
- **零**（= 0）：值在 `this` 寄存器中

这个三分法是 TJS2 寄存器编码的核心设计，我们在下一节会详细讲解。

### 4.3 switch 分发一览

GenNodeCode 的 switch 覆盖了约 30 种 AST 节点类型。按功能分组如下：

```
switch(node->GetOpcode()) {
    // ── 常量与标识符 ──
    case T_CONSTVAL:      // 常量值（数字、字符串字面量）
    case T_SYMBOL:        // 变量名（标识符）
    case T_THIS:          // this 关键字
    case T_THIS_PROXY:    // this 代理（内部使用）
    case T_GLOBAL:        // global 关键字
    case T_VOID:          // void 表达式

    // ── 赋值运算 ──
    case T_EQUAL:         // = 赋值
    case T_AMPERSANDEQUAL ... T_RBITSHIFTEQUAL:  // 复合赋值 (&=, |=, +=, ...)

    // ── 二元运算 ──
    case T_VERTLINE ... T_ASTERISK:              // 算术/位运算 (|, ^, &, +, -, ...)
    case T_EQUALEQUAL ... T_GTOREQUAL:           // 比较运算 (==, !=, <, >, ...)
    case T_LOGICALOR:     // || 短路或
    case T_LOGICALAND:    // && 短路与
    case T_INSTANCEOF:    // instanceof 类型检查

    // ── 一元运算 ──
    case T_EXCRAMATION:   // ! 逻辑非
    case T_TILDE ... T_OCTET:   // ~, #, $, +x, -x, invalidate, isvalid, int, real, string, octet
    case T_TYPEOF:        // typeof
    case T_DELETE:        // delete
    case T_INCREMENT ... T_POSTDECREMENT:  // ++x, --x, x++, x--

    // ── 函数调用 ──
    case T_LPARENTHESIS:  // 函数调用 f(args)
    case T_NEW:           // new 构造
    case T_ARG:           // 函数参数列表节点
    case T_OMIT:          // 参数省略（...）

    // ── 属性访问 ──
    case T_DOT:           // obj.prop 直接属性访问
    case T_LBRACKET:      // obj[key] 间接属性访问
    case T_WITHDOT:       // with 语句的属性访问
    case T_IGNOREPROP:    // &prop 忽略属性 getter/setter
    case T_PROPACCESS:    // *prop 强制属性访问

    // ── 控制流 ──
    case T_IF:            // if 表达式运算符
    case T_QUESTION:      // ?: 三元运算符
    case T_COMMA:         // , 逗号运算符
    case T_SWAP:          // <-> 交换运算符

    // ── 字面量构造 ──
    case T_ARRAYARG:      // 数组字面量
    case T_DICTARG:       // 字典字面量

    // ── 内联函数/类 ──
    case T_FUNCTION:      // 匿名函数表达式
    case T_CLASS:         // 匿名类表达式
}
```

> **重要发现**：注意 `while`、`for`、`switch` 等控制流**语句**不在这个 switch 中。这些语句在 Bison 的归约 action 中直接调用 `EnterWhileCode()`、`EnterForCode()`、`EnterSwitchCode()` 等专用方法，**绕过了 AST**。只有**表达式**才走 GenNodeCode 路径。这是 TJS2 编译器的一个重要设计决策——语句级控制流在解析阶段就直接生成字节码。

---

## §5 常量值与变量访问的编译

### 5.1 T_CONSTVAL —— 常量值加载

这是最简单的节点类型，将一个字面量值加载到寄存器：

```cpp
// 源码位置：cpp/core/tjs2/tjsInterCodeGen.cpp 第 1133-1147 行
case parser::token_kind_type::T_CONSTVAL:
{
    // 如果这个常量被用在赋值左侧，报错
    if(TJSIsModifySubType(param.SubType))
        _yyerror(TJSCannotModifyLHS, Block);

    // 如果调用者不需要结果，直接返回（优化：不生成无用代码）
    if(!(restype & TJS_RT_NEEDED))
        return 0;

    // ① 将常量存入数据区（可能去重复用已有条目）
    tjs_int dp = PutData(node->GetValue());

    // ② 生成 CONST 指令
    PutCode(VM_CONST, node_pos);           // 操作码
    PutCode(TJS_TO_VM_REG_ADDR(frame), node_pos);  // 目标寄存器
    PutCode(TJS_TO_VM_REG_ADDR(dp), node_pos);     // 数据区索引

    // ③ 分配临时寄存器并返回
    return frame++;
}
```

**完整编译示例**：TJS2 表达式 `42` 被编译为：

```
数据区: DataArea[0] = 42 (tTJSVariant, 类型 tvtInteger)

代码区: CONST %r1, *d0
        // %r1 = 寄存器 1（frame=1, 编译后 frame 变为 2）
        // *d0 = 数据区第 0 项
```

### 5.2 T_SYMBOL —— 变量读写

变量访问是 GenNodeCode 中最复杂的 case 之一（第 2179-2346 行），因为它需要区分**局部变量**和**非局部变量**，并根据 `param.SubType` 决定是读取还是写入。

**局部变量查找**：

```cpp
// 第 2182-2190 行
case parser::token_kind_type::T_SYMBOL:
{
    tjs_int n;
    if(AsGlobalContextMode) {
        n = -1;  // 全局上下文模式不访问局部变量
    } else {
        // 在命名空间中查找变量名
        tTJSVariantString *str = node->GetValue().AsString();
        n = Namespace.Find(str->operator const tjs_char *());
        str->Release();
    }
    // n >= 0: 局部变量（在命名空间中找到了）
    // n == -1: 非局部变量（需要去 this/global 查找）
```

**局部变量读取**（`n >= 0, param.SubType == stNone`）：

```cpp
    // 第 2308-2317 行
    // 直接返回局部变量的寄存器地址
    tjs_int n = Namespace.Find(str->operator const tjs_char *());
    str->Release();
    return -n - VariableReserveCount - 1;
    // 这个负数编码在 VM 执行时会被 TJS_GET_VM_REG 宏正确解码
```

**局部变量写入**（`n >= 0, param.SubType == stEqual`）：

```cpp
    // 第 2198-2205 行 — 简单赋值
    case stEqual:
        PutCode(VM_CP, node_pos);
        PutCode(TJS_TO_VM_REG_ADDR(-n - VariableReserveCount - 1), node_pos);
        PutCode(TJS_TO_VM_REG_ADDR(param.SubAddress), node_pos);
        break;
    // 等价于: CP %localvar, %source
    // 即把源寄存器的值复制到局部变量寄存器
```

**非局部变量的处理**（`n == -1`）——这是 TJS2 编译器最巧妙的设计之一：

```cpp
    // 第 2319-2345 行
    } else {
        // 变量不在局部命名空间中
        // 构造一个合成的 DOT 访问节点：this.变量名
        tTJSExprNode nodep;
        nodep.SetOpcode(parser::token::T_DOT);
        // ...
        tTJSExprNode *node1 = new tTJSExprNode;
        node1->SetOpcode(
            AsGlobalContextMode
                ? parser::token_kind_type::T_GLOBAL      // global.变量名
                : parser::token_kind_type::T_THIS_PROXY); // this.变量名
        // ...
        tTJSExprNode *node2 = new tTJSExprNode;
        node2->SetOpcode(parser::token::T_SYMBOL);
        node2->SetValue(node->GetValue());  // 变量名
        // 递归调用 GenNodeCode 处理合成的 DOT 节点
        return _GenNodeCode(frame, &nodep, restype, reqresaddr, param);
    }
```

**这意味着**：当你写 `someVar` 而 `someVar` 不是局部变量时，TJS2 编译器自动将它转换为 `this.someVar`（或 `global.someVar`，取决于上下文模式）。这就是 TJS2 的**隐式 this 解析**机制——变量名首先在局部作用域查找，找不到就自动变成属性访问。

> **与 JavaScript 的对比**：JavaScript 中未声明的变量会沿着作用域链向上查找直到全局对象。TJS2 更激进——未声明的变量直接被当作 `this` 的属性。这在 KiriKiri 游戏脚本中很常见，因为 KAG 标签的属性经常被当作隐式变量使用。

---

## §6 二元运算符的编译

### 6.1 算术与位运算（T_PLUS, T_MINUS, T_ASTERISK, ...）

所有二元算术/位运算操作码共享同一个 case 分支（第 1512-1591 行），因为它们的编译模式完全一致：

```cpp
case parser::token_kind_type::T_VERTLINE:    // |
case parser::token_kind_type::T_CHEVRON:     // ^
case parser::token_kind_type::T_AMPERSAND:   // &
case parser::token_kind_type::T_RARITHSHIFT: // >>
case parser::token_kind_type::T_LARITHSHIFT: // <<
case parser::token_kind_type::T_RBITSHIFT:   // >>>
case parser::token_kind_type::T_PLUS:        // +
case parser::token_kind_type::T_MINUS:       // -
case parser::token_kind_type::T_PERCENT:     // %
case parser::token_kind_type::T_SLASH:       // /
case parser::token_kind_type::T_BACKSLASH:   // \ (整数除法)
case parser::token_kind_type::T_ASTERISK:    // *
{
    tjs_int resaddr1, resaddr2;
    // 赋值左侧检查
    if(TJSIsModifySubType(param.SubType))
        _yyerror(TJSCannotModifyLHS, Block);

    // ① 编译左操作数
    resaddr1 = _GenNodeCode(frame, (*node)[0], TJS_RT_NEEDED, 0, tSubParam());

    // ② 确保左操作数在帧寄存器中（结果会原地修改）
    if(!TJSIsFrame(resaddr1)) {
        PutCode(VM_CP, node_pos);
        PutCode(TJS_TO_VM_REG_ADDR(frame), node_pos);
        PutCode(TJS_TO_VM_REG_ADDR(resaddr1), node_pos);
        resaddr1 = frame;
        frame++;
    }

    // ③ 编译右操作数
    resaddr2 = _GenNodeCode(frame, (*node)[1], TJS_RT_NEEDED, 0, tSubParam());

    // ④ 根据节点类型选择操作码
    tjs_int32 code;
    switch(node->GetOpcode()) {
        case parser::token_kind_type::T_PLUS:      code = VM_ADD;  break;
        case parser::token_kind_type::T_MINUS:     code = VM_SUB;  break;
        case parser::token_kind_type::T_ASTERISK:  code = VM_MUL;  break;
        case parser::token_kind_type::T_SLASH:     code = VM_DIV;  break;
        case parser::token_kind_type::T_BACKSLASH: code = VM_IDIV; break;
        case parser::token_kind_type::T_PERCENT:   code = VM_MOD;  break;
        // ... 位运算类似
    }

    // ⑤ 发射二元操作指令
    PutCode(code, node_pos);
    PutCode(TJS_TO_VM_REG_ADDR(resaddr1), node_pos);  // dst/left
    PutCode(TJS_TO_VM_REG_ADDR(resaddr2), node_pos);  // right
    return resaddr1;  // 结果覆盖左操作数寄存器
}
```

**完整编译示例**：`a + b * 2` 的编译过程（假设 `a` 是局部变量 0，`b` 是局部变量 1）：

```
① AST 结构：
       T_PLUS
      /      \
   T_SYMBOL  T_ASTERISK
   "a"       /        \
          T_SYMBOL  T_CONSTVAL
          "b"       2

② 编译 T_PLUS 的左子节点 T_SYMBOL "a"：
   → 返回 resaddr1 = -0 - VariableReserveCount - 1 (局部变量地址)
   → 因为 TJSIsFrame(-...) == false，需要 CP 到帧寄存器
   → CP %r1, %local_a          // 复制到帧寄存器
   → resaddr1 = 1, frame = 2

③ 编译 T_PLUS 的右子节点 T_ASTERISK：
   ③a 编译左子节点 T_SYMBOL "b"：
      → 返回 resaddr1 = -1 - VRC - 1
      → CP %r2, %local_b
      → resaddr1 = 2, frame = 3

   ③b 编译右子节点 T_CONSTVAL 2：
      → CONST %r3, *d0          // d0 = 2
      → resaddr2 = 3, frame = 4

   ③c 发射 MUL：
      → MUL %r2, %r3
      → 返回 resaddr = 2

④ 回到 T_PLUS，发射 ADD：
   → ADD %r1, %r2
   → 返回 resaddr = 1

最终字节码序列：
   CP    %r1, %local_a     // 复制局部变量 a 到临时寄存器
   CP    %r2, %local_b     // 复制局部变量 b 到临时寄存器
   CONST %r3, *d0          // 加载常量 2
   MUL   %r2, %r3          // r2 = b * 2
   ADD   %r1, %r2          // r1 = a + (b * 2)
```

> **注意"结果覆盖左操作数"模式**：所有二元运算都将结果写回左操作数寄存器。这就是为什么步骤②要先把局部变量复制到帧寄存器——如果直接对局部变量寄存器做 ADD，会破坏局部变量的原始值。`TJSIsFrame()` 检查正是为此设计的：只有帧寄存器（正数地址）才允许被覆盖。

### 6.2 比较运算符

比较运算符（`==`, `!=`, `<`, `>`, `<=`, `>=`, `===`, `!==`）的编译更复杂，因为它们需要与条件标志系统协作：

```cpp
// 第 1593-1673 行（简化版）
case parser::token_kind_type::T_EQUALEQUAL:   // ==
case parser::token_kind_type::T_NOTEQUAL:     // !=
case parser::token_kind_type::T_LT:           // <
// ...
{
    // ① 编译两个操作数（同上）
    resaddr1 = _GenNodeCode(frame, (*node)[0], ...);
    resaddr2 = _GenNodeCode(frame, (*node)[1], ...);

    // ② 选择比较指令和标志设置方式
    tjs_int32 code1, code2;
    switch(node->GetOpcode()) {
        case T_EQUALEQUAL:    code1 = VM_CEQ;  code2 = VM_SETF;  break;
        case T_NOTEQUAL:      code1 = VM_CEQ;  code2 = VM_SETNF; break;
        case T_DISCEQUAL:     code1 = VM_CDEQ; code2 = VM_SETF;  break;
        case T_DISCNOTEQUAL:  code1 = VM_CDEQ; code2 = VM_SETNF; break;
        case T_LT:            code1 = VM_CLT;  code2 = VM_SETF;  break;
        case T_GT:            code1 = VM_CGT;  code2 = VM_SETF;  break;
        case T_LTOREQUAL:     code1 = VM_CGT;  code2 = VM_SETNF; break; // <= = !(>)
        case T_GTOREQUAL:     code1 = VM_CLT;  code2 = VM_SETNF; break; // >= = !(<)
    }

    // ③ 发射比较指令
    PutCode(code1, node_pos);
    PutCode(TJS_TO_VM_REG_ADDR(resaddr1), node_pos);
    PutCode(TJS_TO_VM_REG_ADDR(resaddr2), node_pos);

    // ④ 如果调用者不需要条件标志，物化为寄存器值
    if(!(restype & TJS_RT_CFLAG)) {
        PutCode(code2, node_pos);   // SETF 或 SETNF
        PutCode(TJS_TO_VM_REG_ADDR(resaddr1), node_pos);
    }

    // ⑤ 返回结果
    return (restype & TJS_RT_CFLAG)
        ? (code2 == VM_SETNF ? TJS_GNC_CFLAG_I : TJS_GNC_CFLAG)
        : resaddr1;
}
```

**关键设计**：`<=` 运算符没有专用的 VM 指令。编译器用 `CGT`（大于比较）+ `SETNF`（取反）来实现 `<=`。同理 `>=` 用 `CLT` + `SETNF` 实现。这样 VM 只需要 3 个比较指令（`CEQ`/`CLT`/`CGT`）就能支持 8 种比较运算符。

**`TJS_RT_CFLAG` 优化**：当比较表达式的结果直接用于 `if` 条件时（`restype & TJS_RT_CFLAG`），编译器跳过 SETF/SETNF 步骤，直接返回条件标志。这避免了不必要的"比较→物化为布尔值→再测试布尔值"的往返。

### 6.3 短路求值（T_LOGICALOR, T_LOGICALAND）

`||` 和 `&&` 的编译需要处理短路求值（Short-circuit Evaluation）——当左操作数已经确定结果时跳过右操作数的求值：

```
a || b 的字节码：
    [编译 a]              // 求值 a
    TT %resaddr_a         // 测试 a 是否为真
    JF skip               // 如果为真，跳过 b（短路）
    [编译 b]              // 求值 b
    TT %resaddr_b         // 测试 b
skip:
    SETF %result          // 将条件标志物化为寄存器值

a && b 的字节码：
    [编译 a]              // 求值 a
    TT %resaddr_a         // 测试 a 是否为真
    JNF skip              // 如果为假，跳过 b（短路）
    [编译 b]              // 求值 b
    TT %resaddr_b         // 测试 b
skip:
    SETF %result          // 将条件标志物化为寄存器值
```

注意 `||` 用 `JF`（为真时跳转），`&&` 用 `JNF`（为假时跳转）——跳转方向相反。跳转偏移量在发射时先填 0，随后用 `CodeArea[addr + 1] = CodeAreaSize - addr` 回填实际距离。

---

## §7 赋值运算符的编译

### 7.1 简单赋值（T_EQUAL）

`=` 赋值的编译采用"反向传播"策略：

```cpp
// 第 1246-1265 行
case parser::token_kind_type::T_EQUAL:
{
    if(param.SubType)
        _yyerror(TJSCannotModifyLHS, Block);

    // 在布尔上下文中使用 = 发出警告（可能误写 == 为 =）
    if(restype & TJS_RT_CFLAG) {
        OutputWarning(TJSSubstitutionInBooleanContext, node_pos);
    }

    // ① 先编译右侧表达式，得到结果寄存器
    resaddr = _GenNodeCode(frame, (*node)[1], TJS_RT_NEEDED, 0, param);

    // ② 将结果传递给左侧节点进行写入
    tSubParam param2;
    param2.SubType = stEqual;          // 告诉左侧"你被赋值了"
    param2.SubAddress = resaddr;       // 来源寄存器
    _GenNodeCode(frame, (*node)[0], 0, 0, param2);

    return resaddr;
}
```

如果左侧是局部变量 `x`，会进入 T_SYMBOL 的 `stEqual` 分支，生成 `CP %x, %resaddr`。如果左侧是属性 `obj.prop`，会进入 T_DOT 的 `stEqual` 分支，生成 `SPDE %obj, "prop", %resaddr`。

### 7.2 复合赋值（T_PLUSEQUAL, T_MINUSEQUAL, ...）

复合赋值运算符（`+=`, `-=`, `*=` 等）共 14 种，它们的编译策略统一：先编译右操作数，然后用 `param2.SubType = stXxx` 告诉左侧节点进行对应的修改赋值：

```cpp
// 第 1267-1344 行（简化）
case T_PLUSEQUAL:
case T_MINUSEQUAL:
// ... 其他 12 种 ...
{
    // ① 编译右操作数
    resaddr = _GenNodeCode(frame, (*node)[1], TJS_RT_NEEDED, 0, tSubParam());

    // ② 根据运算符类型设置 SubType
    tSubParam param2;
    switch(node->GetOpcode()) {
        case T_PLUSEQUAL:        param2.SubType = stAdd;    break;
        case T_MINUSEQUAL:      param2.SubType = stSub;    break;
        case T_ASTERISKEQUAL:   param2.SubType = stMul;    break;
        case T_SLASHEQUAL:      param2.SubType = stDiv;    break;
        case T_BACKSLASHEQUAL:  param2.SubType = stIDiv;   break;
        case T_PERCENTEQUAL:    param2.SubType = stMod;    break;
        case T_AMPERSANDEQUAL:  param2.SubType = stBitAND; break;
        case T_VERTLINEEQUAL:   param2.SubType = stBitOR;  break;
        // ...
    }
    param2.SubAddress = resaddr;

    // ③ 交给左侧节点处理
    return _GenNodeCode(frame, (*node)[0], restype, reqresaddr, param2);
}
```

当左侧是 `obj.count`（T_DOT 节点）且 `param.SubType = stAdd` 时，编译器利用属性变体操作码的**连续编号**特性：

```cpp
// 第 2081-2093 行
case stBitAND: case stBitOR: case stBitXOR:
case stSub: case stAdd: case stMod:
case stDiv: case stIDiv: case stMul:
// ...
    PutCode((tjs_int32)param.SubType + (direct ? 1 : 2), node_pos);
    // stAdd 的枚举值恰好等于 VM_ADD！
    // +1 = VM_ADDPD（直接属性）, +2 = VM_ADDPI（间接属性）
```

这就是 §3.2 中提到的"编码技巧"——`stAdd` 等枚举值与 `VM_ADD` 等操作码值对齐，加 1 得到 PD 变体，加 2 得到 PI 变体。

### 7.3 自增自减运算符

自增自减分为 4 种：`++x`（前置自增）、`--x`（前置自减）、`x++`（后置自增）、`x--`（后置自减）。它们的编译差异在于**何时保存旧值**：

**前置自增** `++x`（需要先增后取值）：

```
对局部变量：
    INC %local_x              // 原地自增
    返回 %local_x             // 返回新值

对属性 obj.prop：
    INCPD %result, %obj, "prop"  // 属性自增，结果是新值
```

**后置自增** `x++`（需要先取值后增）：

```
对局部变量：
    CP %temp, %local_x        // 先把旧值复制到临时寄存器
    INC %local_x              // 再自增
    返回 %temp                // 返回旧值

对属性 obj.prop：
    GPD %temp, %obj, "prop"   // 先获取旧值
    INCPD %0, %obj, "prop"    // 再自增（%0 = 不需要结果）
    返回 %temp
```

> **性能提示**：在不需要返回值的语句上下文中（`restype & TJS_RT_NEEDED == 0`），`x++` 和 `++x` 生成完全相同的代码——只有 `INC` 一条指令，没有多余的 CP。这是因为 GenNodeCode 在每种情况下都检查 `restype`，只在调用者确实需要旧值时才生成复制指令。

---

## §8 函数调用的编译

### 8.1 T_LPARENTHESIS / T_NEW —— 函数调用与对象创建

函数调用是 GenNodeCode 中最复杂的分支之一（第 1879-1983 行），因为它需要处理多种调用模式：

```cpp
case parser::token_kind_type::T_LPARENTHESIS:  // f(args)
case parser::token_kind_type::T_NEW:            // new C(args)
{
    // ① 判断调用目标是否有属性访问节点
    bool haspropnode, hasnonlocalsymbol;
    tTJSExprNode *cnode = (*node)[0];  // 被调用的表达式

    // 如果是 func() 且 func 是 obj.method 或 obj[key]
    if(node->GetOpcode() == T_LPARENTHESIS &&
       (cnode->GetOpcode() == T_DOT || cnode->GetOpcode() == T_LBRACKET))
        haspropnode = true;

    // 如果是 func() 且 func 是非局部符号
    if(node->GetOpcode() == T_LPARENTHESIS &&
       cnode->GetOpcode() == T_SYMBOL) {
        if(Namespace.Find(name) == -1)
            hasnonlocalsymbol = true;  // 不在局部作用域 → 需要通过 this/global 调用
    }

    bool do_direct_access = haspropnode || hasnonlocalsymbol;
```

这个判断分出两条路径：

**路径 A：直接访问调用**（`do_direct_access = true`）

当调用目标是属性访问（`obj.method(args)`）或非局部符号时，编译器使用 `VM_CALLD`/`VM_CALLI` 指令。这些指令接收对象引用和方法名，VM 负责解析方法：

```
obj.method(arg1, arg2) 编译为：
    [编译 arg1 → %r2]
    [编译 arg2 → %r3]
    CALLD %r1, %obj, "method"    // 直接属性调用
    arg_count: 2                  // 参数个数
    arg0: %r2                    // 第 1 个参数
    arg1: %r3                    // 第 2 个参数
```

**路径 B：闭包调用**（`do_direct_access = false`）

当调用目标是局部变量（如 `var f = function() {...}; f()`）时，编译器使用 `VM_CALL` 指令：

```
f(arg1, arg2) 编译为：
    [编译 arg1 → %r2]
    [编译 arg2 → %r3]
    CALL %r1, %f                  // 闭包调用
    arg_count: 2
    arg0: %r2
    arg1: %r3
```

**`VM_NEW` 与 `VM_CALL` 的切换**（第 1955-1958 行）：

```cpp
PutCode(node->GetOpcode() == parser::token_kind_type::T_NEW
    ? VM_NEW    // new 运算符
    : VM_CALL,  // 普通函数调用
    node_pos);
```

`new ClassName(args)` 使用与函数调用几乎相同的代码生成逻辑，只是操作码从 `VM_CALL` 换成 `VM_NEW`。VM 执行 `VM_NEW` 时会先创建一个新的空对象，再以该对象为 this 调用构造函数。

### 8.2 参数编码

函数参数通过 `StartFuncArg()` / `AddFuncArg()` / `EndFuncArg()` / `GenerateFuncCallArgCode()` 四个函数管理。参数列表在 AST 中是 `T_ARG` 节点的反向链表结构（见第二章发现），编译器先递归编译所有参数到连续的帧寄存器中，然后生成参数描述：

```cpp
// T_ARG 的编译（第 1985-2011 行）
case parser::token_kind_type::T_ARG:
    // 先编译链表中的后续参数（反向链表，所以先处理 [1]）
    if(node->GetSize() >= 2) {
        if((*node)[1])
            _GenNodeCode(frame, (*node)[1], TJS_RT_NEEDED, 0, tSubParam());
    }
    // 再编译当前参数
    if((*node)[0]) {
        tTJSExprNode *n = (*node)[0];
        if(n->GetOpcode() == T_EXPANDARG) {
            // 展开参数 func(args*)
            AddFuncArg(..., fatExpand);
        } else {
            // 普通参数
            AddFuncArg(..., fatNormal);
        }
    } else {
        // 空参数位（如 func(,b) 中跳过的第一个参数）
        AddFuncArg(0, fatNormal);
    }
    return 0;
```

`GenerateFuncCallArgCode()` 最终将参数信息编码到 CodeArea 中。参数个数和每个参数的寄存器地址紧跟在 CALL/CALLD/CALLI 指令之后。

---

## §9 属性访问的编译

### 9.1 T_DOT 与 T_LBRACKET —— 直接与间接属性访问

属性访问节点（第 2018-2177 行）是 GenNodeCode 中第二复杂的分支，因为它需要根据 `param.SubType` 的不同生成完全不同的指令。这个分支有一个大的 switch-on-SubType：

```
T_DOT / T_LBRACKET 的属性访问编译矩阵：

SubType          | 直接 (T_DOT)      | 间接 (T_LBRACKET)
─────────────────┼────────────────────┼──────────────────
stNone (读取)    | VM_GPD             | VM_GPI
stIgnorePropGet  | VM_GPDS            | VM_GPIS
stEqual (赋值)   | VM_SPDE / VM_SPD   | VM_SPIE / VM_SPI
stIgnorePropSet  | VM_SPDS            | VM_SPIS
stAdd            | VM_ADDPD           | VM_ADDPI
stSub            | VM_SUBPD           | VM_SUBPI
stMul            | VM_MULPD           | VM_MULPI
... (其他复合赋值同理)
stPreInc         | VM_INCPD           | VM_INCPI
stPreDec         | VM_DECPD           | VM_DECPI
stPostInc        | GPD+INCPD          | GPI+INCPI
stPostDec        | GPD+DECPD          | GPI+DECPI
stTypeOf         | VM_TYPEOFD         | VM_TYPEOFI
stDelete         | VM_DELD            | VM_DELI
stFuncCall       | VM_CALLD           | VM_CALLI
```

这个矩阵清晰地展示了为什么 TJS2 需要这么多操作码变体——属性访问与各种操作的组合产生了大量的指令。

**直接属性读取示例**（`obj.name`）：

```cpp
// SubType == stNone 分支（第 2039-2049 行）
PutCode(VM_GPD, node_pos);              // Get Property Direct
PutCode(TJS_TO_VM_REG_ADDR(frame), node_pos);    // result 寄存器
PutCode(TJS_TO_VM_REG_ADDR(resaddr), node_pos);  // obj 寄存器
PutCode(TJS_TO_VM_REG_ADDR(dp), node_pos);       // 属性名在数据区的索引
frame++;
return frame - 1;
```

**函数调用属性访问**（`obj.method(args)` 中的 `stFuncCall`）：

```cpp
// 第 2153-2171 行
case stFuncCall:
    PutCode(direct ? VM_CALLD : VM_CALLI, node_pos);
    PutCode(TJS_TO_VM_REG_ADDR(
        (restype & TJS_RT_NEEDED) ? frame : 0), node_pos);  // result
    PutCode(TJS_TO_VM_REG_ADDR(resaddr), node_pos);  // object
    PutCode(TJS_TO_VM_REG_ADDR(dp), node_pos);       // method name
    GenerateFuncCallArgCode();  // 参数
    // ...
```

> **PD vs PI 的操作数差异**：PD（Property Direct）的属性名是数据区索引（`*name`），PI（Property Indirect）的属性名是寄存器地址（`%key`）。PD 用于 `obj.prop`，属性名在编译时已知；PI 用于 `obj[key]`，属性名在运行时才确定。

---

## §10 一元运算符的编译

### 10.1 通用一元运算（T_TILDE, T_SHARP, T_DOLLAR, ...）

一元运算符的编译模式简洁统一（第 1709-1797 行）：

```cpp
case parser::token_kind_type::T_TILDE:      // ~ 按位取反
case parser::token_kind_type::T_SHARP:      // # 字符→ASCII码
case parser::token_kind_type::T_DOLLAR:     // $ ASCII码→字符
case parser::token_kind_type::T_UPLUS:      // +x 转数值
case parser::token_kind_type::T_UMINUS:     // -x 取负
case parser::token_kind_type::T_INVALIDATE: // invalidate 释放对象
case parser::token_kind_type::T_ISVALID:    // isvalid 检查有效性
case parser::token_kind_type::T_EVAL:       // x! 后置求值
case parser::token_kind_type::T_INT:        // int(x) 转整数
case parser::token_kind_type::T_REAL:       // real(x) 转实数
case parser::token_kind_type::T_STRING:     // string(x) 转字符串
case parser::token_kind_type::T_OCTET:      // octet(x) 转八位串
{
    // ① 编译操作数
    resaddr = _GenNodeCode(frame, (*node)[0], TJS_RT_NEEDED, 0, tSubParam());

    // ② 确保在帧寄存器中（结果会原地修改）
    if(!TJSIsFrame(resaddr)) {
        PutCode(VM_CP, node_pos);
        PutCode(TJS_TO_VM_REG_ADDR(frame), node_pos);
        PutCode(TJS_TO_VM_REG_ADDR(resaddr), node_pos);
        resaddr = frame;
        frame++;
    }

    // ③ 选择操作码
    tjs_int32 code;
    switch(node->GetOpcode()) {
        case T_TILDE:      code = VM_BNOT;   break;
        case T_SHARP:      code = VM_ASC;    break;
        case T_DOLLAR:     code = VM_CHR;    break;
        case T_UPLUS:      code = VM_NUM;    break;
        case T_UMINUS:     code = VM_CHS;    break;
        case T_INVALIDATE: code = VM_INV;    break;
        case T_ISVALID:    code = VM_CHKINV; break;
        case T_INT:        code = VM_INT;    break;
        case T_REAL:       code = VM_REAL;   break;
        case T_STRING:     code = VM_STR;    break;
        case T_OCTET:      code = VM_OCTET;  break;
        case T_EVAL:
            code = (restype & TJS_RT_NEEDED) ? VM_EVAL : VM_EEXP;
            break;
    }

    // ④ 发射指令
    PutCode(code, node_pos);
    PutCode(TJS_TO_VM_REG_ADDR(resaddr), node_pos);
    return resaddr;
}
```

**`T_EVAL`（后置 `!` 运算符）的特殊处理**：TJS2 的后置 `!` 运算符将字符串当作 TJS2 代码执行（类似 JavaScript 的 `eval`）。如果结果被需要（`TJS_RT_NEEDED`），使用 `VM_EVAL`（返回执行结果）；如果不需要结果，使用 `VM_EEXP`（执行但丢弃结果）。此外，如果在非顶层上下文使用 `!` 运算符，编译器会发出警告（第 1775-1777 行）。

### 10.2 逻辑非（T_EXCRAMATION）

`!` 运算符有独立的处理分支（第 1675-1707 行），因为它需要与条件标志系统协作：

```cpp
case parser::token_kind_type::T_EXCRAMATION:  // !
{
    resaddr = _GenNodeCode(frame, (*node)[0], restype, reqresaddr, tSubParam());

    if(!(restype & TJS_RT_CFLAG)) {
        // 调用者需要寄存器值 → 使用 VM_LNOT
        if(!TJSIsFrame(resaddr)) {
            // 复制到帧寄存器（避免修改原变量）
            PutCode(VM_CP, node_pos);
            PutCode(TJS_TO_VM_REG_ADDR(frame), node_pos);
            PutCode(TJS_TO_VM_REG_ADDR(resaddr), node_pos);
            resaddr = frame++;
        }
        PutCode(VM_LNOT, node_pos);
        PutCode(TJS_TO_VM_REG_ADDR(resaddr), node_pos);
        return resaddr;
    } else {
        // 调用者需要条件标志 → 只需翻转标志的语义
        if(!TJSIsCondFlagRetValue(resaddr)) {
            // 子表达式返回了寄存器值 → 用 TF（Test False）转为标志
            PutCode(VM_TF, node_pos);
            PutCode(TJS_TO_VM_REG_ADDR(resaddr));
            return TJS_GNC_CFLAG;
        }
        // 子表达式返回了条件标志 → 翻转 CFLAG 和 CFLAG_I
        return resaddr == TJS_GNC_CFLAG_I ? TJS_GNC_CFLAG : TJS_GNC_CFLAG_I;
    }
}
```

**零成本翻转**：当子表达式已经返回条件标志且调用者也需要条件标志时，`!` 运算符不生成任何指令——仅仅将 `TJS_GNC_CFLAG` 和 `TJS_GNC_CFLAG_I` 互换。这是一个编译期优化，在 `if(!condition)` 这样的常见模式中完全消除了取反指令。

### 10.3 typeof 运算符

`typeof` 有两种处理方式（第 1799-1835 行），取决于操作数是否为属性访问：

```
typeof x         → VM_TYPEOF %x        // 普通 typeof
typeof obj.prop  → VM_TYPEOFD %, %obj, "prop"  // 属性 typeof
typeof obj[key]  → VM_TYPEOFI %, %obj, %key    // 间接属性 typeof
```

为什么需要属性版本的 typeof？因为对不存在的变量使用 `typeof` 不应该抛异常——它应该返回 `"undefined"`。属性版本的 typeof 指令内部捕获"成员不存在"异常并返回 `"undefined"`。

---

## §11 条件表达式与控制流的编译

### 11.1 三元运算符（T_QUESTION）

`a ? b : c` 的编译（第 1346-1428 行）是条件分支的经典模式：

```
[编译条件 a]
TT %resaddr_a              // 测试条件
JNF else_branch             // 为假时跳转到 else

[编译 then 分支 b → %result]
JMP end                     // 跳过 else 分支

else_branch:
[编译 else 分支 c → %result']
// 如果 b 和 c 的结果不在同一个寄存器，用 CP 统一
CP %result, %result'

end:
// %result 包含最终值
```

**帧恢复机制**（第 1400 行）：编译 else 分支前，`frame` 被重置为条件编译开始时的值 `cur_frame`。这是因为 then 和 else 分支是互斥的——then 中使用的临时寄存器在 else 中可以复用。最后取两个分支中较大的 frame 值（第 1426 行）。

### 11.2 if 表达式运算符（T_IF）

TJS2 的 `if` 可以作为**表达式运算符**使用（虽然不能产生值）。这不是 `if` 语句——`if` 语句的编译在 `EnterIfCode()` 中。T_IF 节点的编译如下（第 1149-1176 行）：

```cpp
case parser::token_kind_type::T_IF:
{
    if(restype & TJS_RT_NEEDED)
        _yyerror(TJSCannotGetResult, Block);  // if 表达式不能作为值

    // 编译条件（右子节点 [1]）
    tjs_int resaddr = _GenNodeCode(frame, (*node)[1],
        TJS_RT_NEEDED | TJS_RT_CFLAG, 0, tSubParam());

    // 生成条件跳转
    bool inv = false;
    if(!TJSIsCondFlagRetValue(resaddr)) {
        PutCode(VM_TT, node_pos);
        PutCode(TJS_TO_VM_REG_ADDR(resaddr), node_pos);
    } else {
        if(resaddr == TJS_GNC_CFLAG_I) inv = true;
    }

    tjs_int addr = CodeAreaSize;
    AddJumpList();
    PutCode(inv ? VM_JF : VM_JNF, node_pos);
    PutCode(0, node_pos);  // 待回填的跳转偏移

    // 编译 then 体（左子节点 [0]）
    _GenNodeCode(frame, (*node)[0], 0, 0, param);

    // 回填跳转偏移
    CodeArea[addr + 1] = CodeAreaSize - addr;
    return 0;
}
```

### 11.3 incontextof 运算符（T_INCONTEXTOF）

TJS2 特有的 `incontextof` 运算符改变闭包的 `this` 上下文（第 1178-1203 行）：

```javascript
// TJS2 代码
var func = obj.method incontextof otherObj;
// func 现在绑定到 otherObj 作为 this
```

编译为：

```
[编译 closure → %r1]    // 闭包对象
[编译 new_this → %r2]   // 新的 this 对象
CHGTHIS %r1, %r2         // 修改闭包的上下文
// %r1 现在持有重新绑定的闭包
```

### 11.4 swap 运算符（T_SWAP）

TJS2 的 `<->` 交换运算符交换两个变量的值（第 1211-1244 行）：

```javascript
// TJS2 代码
a <-> b;  // 交换 a 和 b 的值
```

编译策略：先读取两个值，然后交叉赋值：

```
[编译 a → %r1]           // 保存 a 的值
[编译 b → %r2]           // 保存 b 的值
[将 %r2 赋给 a]          // a = 旧的 b
[将 %r1 赋给 b]          // b = 旧的 a
```

---

## §12 地址编码宏详解

### 12.1 六个核心宏

TJS2 的寄存器寻址通过 6 个宏实现（`tjsInterCodeGen.h` 第 34-41 行）：

```cpp
// 编译期宏（逻辑编号 → 字节偏移）
#define TJS_TO_VM_CODE_ADDR(x)  ((x) * (tjs_int)sizeof(tjs_uint32))
// 将代码区索引转为字节偏移。sizeof(tjs_uint32) = 4

#define TJS_TO_VM_REG_ADDR(x)   ((x) * (tjs_int)sizeof(tTJSVariant))
// 将寄存器逻辑编号转为字节偏移。sizeof(tTJSVariant) = 16

// 反向宏（字节偏移 → 逻辑编号）
#define TJS_FROM_VM_CODE_ADDR(x)  ((tjs_int)(x) / (tjs_int)sizeof(tjs_uint32))
#define TJS_FROM_VM_REG_ADDR(x)   ((tjs_int)(x) / (tjs_int)sizeof(tTJSVariant))

// 运行期宏（基址 + 偏移 → 指针/值）
#define TJS_ADD_VM_CODE_ADDR(dest, x)  ((*(char **)&(dest)) += (x))
// 将代码指针前进 x 字节

#define TJS_GET_VM_REG_ADDR(base, x)  \
    ((tTJSVariant *)((char *)(base) + (tjs_int)(x)))
// 从寄存器基址 base 偏移 x 字节，得到 tTJSVariant 指针

#define TJS_GET_VM_REG(base, x)  (*(TJS_GET_VM_REG_ADDR(base, x)))
// 同上，但直接解引用得到 tTJSVariant 值
```

### 12.2 编码转换示例

```
逻辑寄存器 2  →  TJS_TO_VM_REG_ADDR(2)  =  2 * 16  =  32 (字节偏移)
代码位置 10   →  TJS_TO_VM_CODE_ADDR(10) = 10 * 4   =  40 (字节偏移)

VM 执行时：
    code_ptr = CodeArea;
    reg_base = &registers[0];

    // 读取操作码
    opcode = *code_ptr;

    // 读取操作数中的寄存器偏移
    offset = *(code_ptr + 1);  // 例如 32

    // 获取寄存器值
    tTJSVariant &reg = TJS_GET_VM_REG(reg_base, offset);
    // = *(tTJSVariant *)((char *)reg_base + 32)
    // = registers[2]
```

> **为什么不直接用数组索引？**  因为 VM 执行时希望避免乘法运算。编译期将逻辑编号预乘为字节偏移，运行期只需做指针加法。在热循环中这个优化非常值得。

---

## §13 动手实践：构建微型字节码编译器

下面的实验帮助你亲手体验 GenNodeCode 的编译模式。

### 实验 1：表达式编译器

```cpp
// 文件：bytecode_compiler_demo.cpp
// 演示 TJS2 风格的 AST → 字节码编译过程
// 编译：g++ -std=c++17 -o compiler_demo bytecode_compiler_demo.cpp
// 运行：./compiler_demo

#include <cstdint>
#include <iostream>
#include <string>
#include <variant>
#include <vector>

// ─── 操作码定义（简化版 tTJSVMCodes）───
enum VMCode : int32_t {
    VM_NOP   = 0,
    VM_CONST = 1,   // CONST %dst, *data_index
    VM_CP    = 2,   // CP %dst, %src
    VM_ADD   = 3,   // ADD %dst, %src  (dst += src)
    VM_SUB   = 4,   // SUB %dst, %src
    VM_MUL   = 5,   // MUL %dst, %src
    VM_CEQ   = 6,   // CEQ %r1, %r2   (设置条件标志)
    VM_CLT   = 7,   // CLT %r1, %r2
    VM_SETF  = 8,   // SETF %dst      (条件标志→寄存器)
    VM_SETNF = 9,   // SETNF %dst
    VM_JF    = 10,  // JF offset
    VM_JNF   = 11,  // JNF offset
    VM_JMP   = 12,  // JMP offset
    VM_TT    = 13,  // TT %reg        (测试真值)
    VM_LNOT  = 14,  // LNOT %reg      (逻辑非)
};

// ─── AST 节点定义（简化版 tTJSExprNode）───
enum NodeType {
    ND_CONST,   // 常量
    ND_VAR,     // 局部变量
    ND_ADD,     // +
    ND_SUB,     // -
    ND_MUL,     // *
    ND_EQ,      // ==
    ND_LT,      // <
    ND_NOT,     // !
    ND_ASSIGN,  // =
};

struct ASTNode {
    NodeType type;
    double value;       // ND_CONST 时使用
    std::string name;   // ND_VAR 时使用
    ASTNode *left = nullptr;
    ASTNode *right = nullptr;
};

// ─── 编译上下文（简化版 tTJSInterCodeContext）───
struct CompileContext {
    std::vector<int32_t> code_area;     // 代码区
    std::vector<double> data_area;      // 数据区（简化为 double）
    std::vector<std::string> locals;    // 局部变量表
    int frame = 1;                      // 帧寄存器起点（0 号预留给 this）

    // 模拟 TJS2 的 PutCode
    int put_code(int32_t val) {
        int pos = (int)code_area.size();
        code_area.push_back(val);
        return pos;
    }

    // 模拟 TJS2 的 PutData（含去重）
    int put_data(double val) {
        // 搜索最近 20 项
        int search_limit = std::min((int)data_area.size(), 20);
        for (int i = (int)data_area.size() - 1;
             i >= (int)data_area.size() - search_limit; i--) {
            if (data_area[i] == val) return i;
        }
        int idx = (int)data_area.size();
        data_area.push_back(val);
        return idx;
    }

    // 模拟 Namespace.Find
    int find_local(const std::string &name) {
        for (int i = 0; i < (int)locals.size(); i++) {
            if (locals[i] == name) return i;
        }
        return -1;
    }

    // 寄存器编号 → 字节偏移（模拟 TJS_TO_VM_REG_ADDR）
    static int32_t reg_addr(int reg) { return reg * 16; }
};

// ─── 编译函数（简化版 GenNodeCode）───
// 返回结果寄存器编号
int gen_node_code(CompileContext &ctx, ASTNode *node, bool need_result) {
    switch (node->type) {

        case ND_CONST: {
            if (!need_result) return 0;
            int dp = ctx.put_data(node->value);
            ctx.put_code(VM_CONST);
            ctx.put_code(CompileContext::reg_addr(ctx.frame));
            ctx.put_code(CompileContext::reg_addr(dp));
            return ctx.frame++;
        }

        case ND_VAR: {
            int n = ctx.find_local(node->name);
            if (n == -1) {
                std::cerr << "未定义变量: " << node->name << std::endl;
                return 0;
            }
            // 局部变量编码：-n - 2（简化版，省略 VariableReserveCount）
            return -n - 2;
        }

        case ND_ADD: case ND_SUB: case ND_MUL: {
            // ① 编译左操作数
            int r1 = gen_node_code(ctx, node->left, true);
            // ② 确保在帧寄存器中
            if (r1 <= 0) {
                ctx.put_code(VM_CP);
                ctx.put_code(CompileContext::reg_addr(ctx.frame));
                ctx.put_code(CompileContext::reg_addr(r1));
                r1 = ctx.frame++;
            }
            // ③ 编译右操作数
            int r2 = gen_node_code(ctx, node->right, true);
            // ④ 发射运算指令
            VMCode op = (node->type == ND_ADD) ? VM_ADD :
                        (node->type == ND_SUB) ? VM_SUB : VM_MUL;
            ctx.put_code(op);
            ctx.put_code(CompileContext::reg_addr(r1));
            ctx.put_code(CompileContext::reg_addr(r2));
            return r1;  // 结果覆盖左操作数
        }

        case ND_ASSIGN: {
            // ① 编译右侧
            int src = gen_node_code(ctx, node->right, true);
            // ② 局部变量赋值
            int n = ctx.find_local(node->left->name);
            if (n == -1) {
                std::cerr << "赋值目标未定义: "
                          << node->left->name << std::endl;
                return 0;
            }
            ctx.put_code(VM_CP);
            ctx.put_code(CompileContext::reg_addr(-n - 2));
            ctx.put_code(CompileContext::reg_addr(src));
            return src;
        }

        case ND_NOT: {
            int r = gen_node_code(ctx, node->left, true);
            if (r <= 0) {
                ctx.put_code(VM_CP);
                ctx.put_code(CompileContext::reg_addr(ctx.frame));
                ctx.put_code(CompileContext::reg_addr(r));
                r = ctx.frame++;
            }
            ctx.put_code(VM_LNOT);
            ctx.put_code(CompileContext::reg_addr(r));
            return r;
        }

        default:
            return 0;
    }
}

// ─── 反汇编输出 ───
const char *opcode_name(int32_t op) {
    switch (op) {
        case VM_NOP:   return "NOP";
        case VM_CONST: return "CONST";
        case VM_CP:    return "CP";
        case VM_ADD:   return "ADD";
        case VM_SUB:   return "SUB";
        case VM_MUL:   return "MUL";
        case VM_CEQ:   return "CEQ";
        case VM_CLT:   return "CLT";
        case VM_SETF:  return "SETF";
        case VM_SETNF: return "SETNF";
        case VM_JF:    return "JF";
        case VM_JNF:   return "JNF";
        case VM_JMP:   return "JMP";
        case VM_TT:    return "TT";
        case VM_LNOT:  return "LNOT";
        default:       return "???";
    }
}

void disassemble(const CompileContext &ctx) {
    std::cout << "=== 数据区 ===" << std::endl;
    for (int i = 0; i < (int)ctx.data_area.size(); i++) {
        std::cout << "  *d" << i << " = " << ctx.data_area[i] << std::endl;
    }

    std::cout << "\n=== 代码区 ===" << std::endl;
    int pc = 0;
    while (pc < (int)ctx.code_area.size()) {
        int32_t op = ctx.code_area[pc];
        std::cout << "  [" << pc << "] " << opcode_name(op);

        // 根据操作码确定操作数个数
        int operands = 0;
        switch (op) {
            case VM_NOP:                    operands = 0; break;
            case VM_CONST:                  operands = 2; break;
            case VM_CP: case VM_ADD:
            case VM_SUB: case VM_MUL:
            case VM_CEQ: case VM_CLT:       operands = 2; break;
            case VM_SETF: case VM_SETNF:
            case VM_TT:  case VM_LNOT:      operands = 1; break;
            case VM_JF: case VM_JNF:
            case VM_JMP:                    operands = 1; break;
            default:                        operands = 0; break;
        }

        for (int i = 0; i < operands; i++) {
            pc++;
            int32_t val = ctx.code_area[pc];
            int reg = val / 16;  // 字节偏移 → 逻辑编号
            if (op == VM_CONST && i == 1) {
                std::cout << " *d" << reg;
            } else if (op == VM_JF || op == VM_JNF || op == VM_JMP) {
                std::cout << " +" << val;
            } else if (reg < 0) {
                std::cout << " %local" << (-reg - 2);
            } else {
                std::cout << " %r" << reg;
            }
        }
        std::cout << std::endl;
        pc++;
    }
}

int main() {
    CompileContext ctx;
    ctx.locals = {"a", "b"};  // 两个局部变量

    // 构造 AST：a = a + b * 2
    ASTNode const2 = {ND_CONST, 2.0, "", nullptr, nullptr};
    ASTNode varA1  = {ND_VAR, 0, "a", nullptr, nullptr};
    ASTNode varB   = {ND_VAR, 0, "b", nullptr, nullptr};
    ASTNode mul    = {ND_MUL, 0, "", &varB, &const2};
    ASTNode add    = {ND_ADD, 0, "", &varA1, &mul};
    ASTNode varA2  = {ND_VAR, 0, "a", nullptr, nullptr};
    ASTNode assign = {ND_ASSIGN, 0, "", &varA2, &add};

    std::cout << "编译表达式: a = a + b * 2\n" << std::endl;
    gen_node_code(ctx, &assign, false);
    disassemble(ctx);

    return 0;
}
```

**预期输出**：

```
编译表达式: a = a + b * 2

=== 数据区 ===
  *d0 = 2

=== 代码区 ===
  [0] CP %r1 %local0           // 复制局部变量 a 到帧寄存器
  [3] CP %r2 %local1           // 复制局部变量 b 到帧寄存器
  [6] CONST %r3 *d0            // 加载常量 2
  [9] MUL %r2 %r3              // r2 = b * 2
  [12] ADD %r1 %r2             // r1 = a + b*2
  [15] CP %local0 %r1          // a = r1（赋值）
```

### 实验 2：操作码统计器

```cpp
// 文件：opcode_stats.cpp
// 统计 TJS2 操作码枚举中各类操作码的数量
// 编译：g++ -std=c++17 -o opcode_stats opcode_stats.cpp
// 运行：./opcode_stats

#include <iostream>
#include <string>
#include <map>

// 模拟 TJS_NORMAL_AND_PROPERTY_ACCESSER 宏
#define EXPAND_4(base) base, base##_PD, base##_PI, base##_P

int main() {
    // 基础操作码（无属性变体）
    int basic_count = 18;  // VM_NOP 到 VM_JMP

    // 带属性变体的操作码组数
    int property_groups = 16;  // INC, DEC, LOR...MUL

    // 每组展开 4 个
    int property_opcodes = property_groups * 4;  // 64

    // 其他独立操作码
    int misc_count = 0;
    std::map<std::string, int> categories;
    categories["一元运算"] = 15;     // BNOT 到 OCTET
    categories["函数调用"] = 4;      // CALL, CALLD, CALLI, NEW
    categories["属性操作"] = 14;     // GPD 到 DELI
    categories["控制流"]   = 6;      // SRV, RET, ENTRY, EXTRY, THROW, CHGTHIS
    categories["特殊"]     = 4;      // GLOBAL, ADDCI, REGMEMBER, DEBUGGER

    for (auto &[cat, count] : categories) misc_count += count;

    int total = basic_count + property_opcodes + misc_count;

    std::cout << "=== TJS2 VM 操作码统计 ===" << std::endl;
    std::cout << "基础操作码（无变体）:  " << basic_count << std::endl;
    std::cout << "属性变体操作码组:      " << property_groups
              << " 组 × 4 = " << property_opcodes << std::endl;
    for (auto &[cat, count] : categories) {
        std::cout << cat << ":              " << count << std::endl;
    }
    std::cout << "────────────────────────────" << std::endl;
    std::cout << "总计:                  ~" << total << " 个操作码" << std::endl;
    std::cout << "\n枚举最后值 __VM_LAST 之前共约 "
              << total << " 个有效操作码" << std::endl;

    // 展示属性变体的编码规律
    std::cout << "\n=== 属性变体编码规律 ===" << std::endl;
    std::cout << "以 VM_ADD 为例（假设基础值 = N）:" << std::endl;
    std::cout << "  VM_ADD    = N     (普通: ADD %dst, %src)" << std::endl;
    std::cout << "  VM_ADDPD  = N+1   (直接属性: ADDPD %r, %obj, *name, %src)"
              << std::endl;
    std::cout << "  VM_ADDPI  = N+2   (间接属性: ADDPI %r, %obj, %key, %src)"
              << std::endl;
    std::cout << "  VM_ADDP   = N+3   (属性对象: ADDP %r, %prop, %src)"
              << std::endl;
    std::cout << "\n编译器利用: code = base_opcode + (direct ? 1 : 2)"
              << std::endl;
    std::cout << "或: code = base_opcode + 3  (属性对象模式)" << std::endl;

    return 0;
}
```

**预期输出**：

```
=== TJS2 VM 操作码统计 ===
基础操作码（无变体）:  18
属性变体操作码组:      16 组 × 4 = 64
一元运算:              15
函数调用:              4
属性操作:              14
控制流:                6
特殊:                  4
────────────────────────────
总计:                  ~121 个操作码

枚举最后值 __VM_LAST 之前共约 121 个有效操作码

=== 属性变体编码规律 ===
以 VM_ADD 为例（假设基础值 = N）:
  VM_ADD    = N     (普通: ADD %dst, %src)
  VM_ADDPD  = N+1   (直接属性: ADDPD %r, %obj, *name, %src)
  VM_ADDPI  = N+2   (间接属性: ADDPI %r, %obj, %key, %src)
  VM_ADDP   = N+3   (属性对象: ADDP %r, %prop, %src)

编译器利用: code = base_opcode + (direct ? 1 : 2)
或: code = base_opcode + 3  (属性对象模式)
```

### 实验 3：条件标志优化演示

```cpp
// 文件：cflag_optimization_demo.cpp
// 演示 TJS2 条件标志优化：比较不同编译策略的代码量
// 编译：g++ -std=c++17 -o cflag_demo cflag_optimization_demo.cpp
// 运行：./cflag_demo

#include <iostream>
#include <string>
#include <vector>

struct Instruction {
    std::string opcode;
    std::string operands;
};

void print_instructions(const std::string &title,
                        const std::vector<Instruction> &insts) {
    std::cout << title << " (" << insts.size() << " 条指令):" << std::endl;
    for (int i = 0; i < (int)insts.size(); i++) {
        std::cout << "  [" << i << "] " << insts[i].opcode
                  << " " << insts[i].operands << std::endl;
    }
    std::cout << std::endl;
}

int main() {
    std::cout << "=== 条件标志优化对比 ===" << std::endl;
    std::cout << "源码: if (a == b) { doSomething(); }\n" << std::endl;

    // 方案 A：朴素编译（无条件标志优化）
    // 比较结果先存入寄存器，再测试寄存器
    std::vector<Instruction> naive = {
        {"CEQ",    "%a, %b"},       // 比较 a 和 b
        {"SETF",   "%tmp"},         // 条件标志 → 寄存器
        {"TT",     "%tmp"},         // 测试寄存器 → 条件标志
        {"JNF",    "skip"},         // 条件跳转
        {"CALLD",  "%0, %this, \"doSomething\""},
        // skip:
    };

    // 方案 B：TJS2 实际编译（有条件标志优化）
    // restype 设置 TJS_RT_CFLAG，跳过 SETF/TT 往返
    std::vector<Instruction> optimized = {
        {"CEQ",    "%a, %b"},       // 比较 a 和 b → 条件标志
        {"JNF",    "skip"},         // 直接使用条件标志跳转
        {"CALLD",  "%0, %this, \"doSomething\""},
        // skip:
    };

    print_instructions("方案 A: 朴素编译", naive);
    print_instructions("方案 B: TJS2 条件标志优化", optimized);

    std::cout << "优化效果: 减少 " << naive.size() - optimized.size()
              << " 条指令 (" << (naive.size() - optimized.size()) * 100 /
                    naive.size()
              << "% 更紧凑)" << std::endl;

    std::cout << "\n--- 逻辑非的零成本翻转 ---" << std::endl;
    std::cout << "源码: if (!(a == b)) { ... }\n" << std::endl;

    // 朴素编译
    std::vector<Instruction> naive_not = {
        {"CEQ",    "%a, %b"},
        {"SETF",   "%tmp"},
        {"LNOT",   "%tmp"},     // 逻辑非
        {"TT",     "%tmp"},
        {"JNF",    "skip"},
    };

    // TJS2 实际编译：! 翻转 CFLAG ↔ CFLAG_I，无额外指令
    std::vector<Instruction> optimized_not = {
        {"CEQ",    "%a, %b"},   // 比较 → CFLAG
        {"JF",     "skip"},     // !(CFLAG) 变为 JF 而非 JNF
    };

    print_instructions("方案 A: 朴素编译 !(a==b)", naive_not);
    print_instructions("方案 B: TJS2 零成本翻转 !(a==b)", optimized_not);

    std::cout << "优化效果: !(a==b) 与 (a==b) 的指令数相同！" << std::endl;
    std::cout << "! 运算符仅翻转 JNF↔JF，不生成任何额外指令。" << std::endl;

    return 0;
}
```

**预期输出**：

```
=== 条件标志优化对比 ===
源码: if (a == b) { doSomething(); }

方案 A: 朴素编译 (5 条指令):
  [0] CEQ %a, %b
  [1] SETF %tmp
  [2] TT %tmp
  [3] JNF skip
  [4] CALLD %0, %this, "doSomething"

方案 B: TJS2 条件标志优化 (3 条指令):
  [0] CEQ %a, %b
  [1] JNF skip
  [2] CALLD %0, %this, "doSomething"

优化效果: 减少 2 条指令 (40% 更紧凑)

--- 逻辑非的零成本翻转 ---
源码: if (!(a == b)) { ... }

方案 A: 朴素编译 !(a==b) (5 条指令):
  [0] CEQ %a, %b
  [1] SETF %tmp
  [2] LNOT %tmp
  [3] TT %tmp
  [4] JNF skip

方案 B: TJS2 零成本翻转 !(a==b) (2 条指令):
  [0] CEQ %a, %b
  [1] JF skip

优化效果: !(a==b) 与 (a==b) 的指令数相同！
! 运算符仅翻转 JNF↔JF，不生成任何额外指令。
```

### 实验 4：赋值反向传播追踪器

```cpp
// 文件：assign_trace.cpp
// 可视化 TJS2 的赋值"反向传播"机制
// 编译：g++ -std=c++17 -o assign_trace assign_trace.cpp
// 运行：./assign_trace

#include <iostream>
#include <string>
#include <vector>

// 模拟 SubType 枚举
enum SubType {
    stNone = 0,
    stEqual,        // = 直接赋值
    stAdd,          // += 加法赋值
    stSub,          // -= 减法赋值
    stMul,          // *= 乘法赋值
    stPreInc,       // ++x
    stPreDec,       // --x
    stPostInc,      // x++
    stPostDec,      // x--
};

const char *subtype_name(SubType st) {
    switch (st) {
        case stNone:    return "stNone (读取)";
        case stEqual:   return "stEqual (= 赋值)";
        case stAdd:     return "stAdd (+= 赋值)";
        case stSub:     return "stSub (-= 赋值)";
        case stMul:     return "stMul (*= 赋值)";
        case stPreInc:  return "stPreInc (++x)";
        case stPreDec:  return "stPreDec (--x)";
        case stPostInc: return "stPostInc (x++)";
        case stPostDec: return "stPostDec (x--)";
    }
    return "unknown";
}

// 模拟 GenNodeCode 对 T_SYMBOL 的处理
void simulate_symbol_code_gen(
    const std::string &var_name,
    SubType sub_type,
    int sub_address,
    int depth = 0
) {
    std::string indent(depth * 2, ' ');

    std::cout << indent << "GenNodeCode(T_SYMBOL \"" << var_name
              << "\", " << subtype_name(sub_type) << ")" << std::endl;

    int local_index = 0;  // 假设变量在局部命名空间索引 0
    int var_reg = -local_index - 2;  // 简化编码

    switch (sub_type) {
        case stNone:
            std::cout << indent << "  → 返回寄存器地址 " << var_reg
                      << " (局部变量直接引用)" << std::endl;
            break;
        case stEqual:
            std::cout << indent << "  → 生成: CP %local"
                      << local_index << ", %r" << sub_address << std::endl;
            break;
        case stAdd:
            std::cout << indent << "  → 生成: ADD %local"
                      << local_index << ", %r" << sub_address << std::endl;
            break;
        case stPreInc:
            std::cout << indent << "  → 生成: INC %local"
                      << local_index << std::endl;
            break;
        case stPostInc:
            std::cout << indent << "  → 生成: CP %temp, %local"
                      << local_index << std::endl;
            std::cout << indent << "  → 生成: INC %local"
                      << local_index << std::endl;
            std::cout << indent << "  → 返回 %temp (旧值)" << std::endl;
            break;
        default:
            break;
    }
}

int main() {
    std::cout << "=== TJS2 赋值反向传播机制演示 ===\n" << std::endl;

    // 场景 1: x = 42
    std::cout << "--- 场景 1: x = 42 ---" << std::endl;
    std::cout << "步骤 1: GenNodeCode(T_EQUAL)" << std::endl;
    std::cout << "  → 先编译右侧 42 → 结果在 %r1" << std::endl;
    std::cout << "  → 构造 param2 = {stEqual, sub_addr=%r1}" << std::endl;
    std::cout << "  → 调用 GenNodeCode(左侧 T_SYMBOL \"x\", param2)"
              << std::endl;
    simulate_symbol_code_gen("x", stEqual, 1, 1);

    // 场景 2: x += 10
    std::cout << "\n--- 场景 2: x += 10 ---" << std::endl;
    std::cout << "步骤 1: GenNodeCode(T_PLUSEQUAL)" << std::endl;
    std::cout << "  → 先编译右侧 10 → 结果在 %r1" << std::endl;
    std::cout << "  → 构造 param2 = {stAdd, sub_addr=%r1}" << std::endl;
    std::cout << "  → 调用 GenNodeCode(左侧 T_SYMBOL \"x\", param2)"
              << std::endl;
    simulate_symbol_code_gen("x", stAdd, 1, 1);

    // 场景 3: ++x
    std::cout << "\n--- 场景 3: ++x ---" << std::endl;
    std::cout << "步骤 1: GenNodeCode(T_INCREMENT)" << std::endl;
    std::cout << "  → 构造 param2 = {stPreInc}" << std::endl;
    std::cout << "  → 调用 GenNodeCode(T_SYMBOL \"x\", param2)" << std::endl;
    simulate_symbol_code_gen("x", stPreInc, 0, 1);

    // 场景 4: x++
    std::cout << "\n--- 场景 4: x++ ---" << std::endl;
    std::cout << "步骤 1: GenNodeCode(T_POSTINCREMENT)" << std::endl;
    std::cout << "  → 构造 param2 = {stPostInc}" << std::endl;
    std::cout << "  → 调用 GenNodeCode(T_SYMBOL \"x\", param2)" << std::endl;
    simulate_symbol_code_gen("x", stPostInc, 0, 1);

    std::cout << "\n=== 总结 ===" << std::endl;
    std::cout << "赋值运算符的编译从不直接处理左侧节点。" << std::endl;
    std::cout << "而是通过 param.SubType 将\"请求\"传递给左侧节点，"
              << std::endl;
    std::cout << "让左侧节点自己决定如何生成正确的写入指令。" << std::endl;
    std::cout << "这种\"反向传播\"设计使同一个 T_SYMBOL case 能处理"
              << std::endl;
    std::cout << "所有赋值变体（=, +=, -=, ++, -- 等）。" << std::endl;

    return 0;
}
```

### 实验 5：跳转回填可视化

```cpp
// 文件：jump_patch_demo.cpp
// 可视化 TJS2 的跳转偏移回填机制
// 编译：g++ -std=c++17 -o jump_patch jump_patch_demo.cpp
// 运行：./jump_patch

#include <cstdint>
#include <iostream>
#include <string>
#include <vector>

struct CodeEmitter {
    std::vector<int32_t> code;
    std::vector<std::string> labels;  // 每个位置的标签

    int emit(int32_t val, const std::string &label = "") {
        int pos = (int)code.size();
        code.push_back(val);
        labels.push_back(label);
        return pos;
    }

    void patch(int pos, int32_t val) {
        code[pos] = val;
    }

    void dump() {
        for (int i = 0; i < (int)code.size(); i++) {
            std::cout << "  [" << i << "] ";
            if (!labels[i].empty()) {
                std::cout << labels[i];
            } else {
                std::cout << code[i];
            }
            std::cout << std::endl;
        }
    }
};

int main() {
    std::cout << "=== 跳转偏移回填机制演示 ===\n" << std::endl;
    std::cout << "源码: a ? b : c\n" << std::endl;

    CodeEmitter emitter;

    // 编译条件 a
    std::cout << "步骤 1: 编译条件表达式 a" << std::endl;
    emitter.emit(13, "TT %a");          // VM_TT

    // 发射条件跳转（偏移量暂填 0）
    int jnf_pos = emitter.emit(11, "JNF ???");  // VM_JNF
    int patch_pos1 = emitter.emit(0, "offset=0 (待回填)");

    std::cout << "步骤 2: 记录 JNF 在位置 " << jnf_pos
              << "，偏移量位置 " << patch_pos1 << " 待回填\n" << std::endl;

    // 编译 then 分支 b
    emitter.emit(1, "CONST %r1, *b_val");  // 编译 b
    emitter.emit(32, "%r1");
    emitter.emit(16, "*d0");

    // 发射无条件跳转到末尾
    int jmp_pos = emitter.emit(12, "JMP ???");  // VM_JMP
    int patch_pos2 = emitter.emit(0, "offset=0 (待回填)");

    // 回填 JNF 的跳转偏移
    int else_start = (int)emitter.code.size();
    int offset1 = else_start - jnf_pos;
    emitter.patch(patch_pos1, offset1);
    std::cout << "步骤 3: 回填 JNF 偏移 = " << else_start
              << " - " << jnf_pos << " = " << offset1 << std::endl;

    // 编译 else 分支 c
    emitter.emit(1, "CONST %r1, *c_val");  // 编译 c
    emitter.emit(32, "%r1");
    emitter.emit(48, "*d1");

    // 回填 JMP 的跳转偏移
    int end_pos = (int)emitter.code.size();
    int offset2 = end_pos - jmp_pos;
    emitter.patch(patch_pos2, offset2);
    std::cout << "步骤 4: 回填 JMP 偏移 = " << end_pos
              << " - " << jmp_pos << " = " << offset2 << std::endl;

    std::cout << "\n=== 最终代码（回填后）===" << std::endl;
    emitter.labels[patch_pos1] = "offset=" + std::to_string(offset1)
                                 + " (已回填)";
    emitter.labels[patch_pos2] = "offset=" + std::to_string(offset2)
                                 + " (已回填)";
    emitter.dump();

    std::cout << "\n=== 执行流 ===" << std::endl;
    std::cout << "如果 a 为真: [0]TT → [1]JNF(不跳) → [3]CONST b → [6]JMP"
              << " → 结束" << std::endl;
    std::cout << "如果 a 为假: [0]TT → [1]JNF(跳到 " << else_start
              << ") → [" << else_start << "]CONST c → 结束" << std::endl;

    return 0;
}
```

---

## §14 对照项目源码

以下是本节涉及的所有源码文件及关键行号：

| 文件路径 | 行号范围 | 内容 |
|---------|----------|------|
| `cpp/core/tjs2/tjsInterCodeGen.h` | 34-41 | 地址编码宏（TJS_TO_VM_REG_ADDR 等 6 个宏） |
| `cpp/core/tjs2/tjsInterCodeGen.h` | 42 | TJS_NORMAL_AND_PROPERTY_ACCESSER 宏定义 |
| `cpp/core/tjs2/tjsInterCodeGen.h` | 44-131 | tTJSVMCodes 枚举（全部 VM 操作码） |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 584-632 | PutCode 实现（代码区追加 + 源码位置追踪） |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 638-682 | PutData 实现（数据区追加 + 窗口去重） |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1100-1104 | TJSIsCondFlagRetValue / TJSIsFrame 辅助函数 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1110-1132 | GenNodeCode 函数签名及参数说明 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1133-1147 | T_CONSTVAL 常量值编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1149-1176 | T_IF 表达式运算符编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1178-1203 | T_INCONTEXTOF 编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1246-1265 | T_EQUAL 赋值编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1267-1344 | 复合赋值运算符编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1346-1428 | T_QUESTION 三元运算符编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1430-1488 | T_LOGICALOR/AND 短路求值编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1512-1591 | 二元算术/位运算编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1593-1673 | 比较运算符编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1675-1707 | T_EXCRAMATION 逻辑非编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1709-1797 | 通用一元运算符编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1837-1877 | 自增自减运算符编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1879-1983 | 函数调用 / new 运算符编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 2018-2177 | T_DOT / T_LBRACKET 属性访问编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 2179-2346 | T_SYMBOL 变量访问编译 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 2913-2918 | CreateExprCode（AST→字节码桥梁） |

**阅读建议**：
1. 先看 `PutCode`（584 行）和 `PutData`（638 行），理解底层发射机制
2. 再看 `T_CONSTVAL`（1133 行），这是最简单的 GenNodeCode case
3. 然后看 `T_SYMBOL`（2179 行），理解变量读写和隐式 this 解析
4. 最后看 `T_DOT`（2018 行），理解属性访问与属性变体操作码的关系

---

## §15 本节小结

1. **GenNodeCode 是 TJS2 编译器的核心**——一个 2600+ 行的巨型 switch 函数，将 AST 节点递归翻译为 VM 操作码序列
2. **PutCode 和 PutData 是字节码发射的双引擎**——PutCode 追加 32 位字到代码区，PutData 追加常量值到数据区（含窗口去重）
3. **TJS2 有约 121 个操作码**——其中 64 个是 16 组算术/逻辑运算的属性访问变体（每组 4 个：普通/直接属性/间接属性/属性对象）
4. **属性变体编号连续排列**——编译器通过 `base + (direct ? 1 : 2)` 算术直接计算变体编号
5. **条件标志系统避免了不必要的寄存器分配**——比较指令设置标志，条件跳转读取标志；`!` 运算符甚至能"零成本"翻转标志
6. **赋值使用"反向传播"策略**——先编译右侧表达式，再通过 `param.SubType` 告知左侧节点进行对应的写入
7. **未声明变量自动变成 `this.变量名`**——Namespace.Find 返回 -1 时，编译器合成 T_DOT 节点递归处理
8. **语句级控制流绕过 AST**——while/for/switch 等语句在 Bison action 中直接调用专用方法生成字节码
9. **PutData 的字符串 intern**——字符串常量通过全局字符串映射池共享，减少内存占用
10. **跳转偏移使用回填技术**——先填 0，编译完目标代码后用 `CodeArea[addr+1] = CodeAreaSize - addr` 回填

---

## 练习题与答案

### 题目 1：操作码变体推导

已知 `VM_SUB` 的枚举值为某个数 N。请回答：
1. `VM_SUBPD` 的值是多少？
2. `VM_SUBPI` 的值是多少？
3. `VM_SUBP` 的值是多少？
4. 对于表达式 `obj[key] -= value`，编译器会选择哪个操作码？它是通过什么计算得出的？

<details>
<summary>查看答案</summary>

1. `VM_SUBPD = N + 1`（直接属性变体）
2. `VM_SUBPI = N + 2`（间接属性变体）
3. `VM_SUBP = N + 3`（属性对象变体）
4. 选择 `VM_SUBPI`（间接属性减法）。计算方式：
   - `param.SubType = stSub`（来自 T_MINUSEQUAL 的复合赋值）
   - `stSub` 的枚举值恰好等于 `VM_SUB = N`
   - `direct = false`（因为是 `obj[key]` 间接访问）
   - 所以操作码 = `(tjs_int32)stSub + (direct ? 1 : 2) = N + 2 = VM_SUBPI`

这个巧妙的设计依赖于 `stXxx` 枚举值与对应的 `VM_XXX` 操作码值对齐。

</details>

### 题目 2：编译结果预测

预测以下 TJS2 代码编译后的字节码序列（假设 `x` 是局部变量 0，`y` 是局部变量 1）：

```javascript
var result = !(x < y);
```

<details>
<summary>查看答案</summary>

编译过程分析：

1. T_EQUAL 处理：先编译右侧 `!(x < y)`
2. T_EXCRAMATION 处理：编译 `x < y`，请求条件标志结果
3. T_LT 处理：编译 `x` 和 `y` 的比较

由于 `result` 需要存储值（`restype & TJS_RT_NEEDED`），但 `!` 在赋值上下文中不能使用条件标志快捷路径：

```
CP    %r1, %local0       // 复制 x 到帧寄存器（左操作数不能被修改）
CLT   %r1, %local1       // CLT %x, %y → 设置条件标志
SETF  %r1                // 条件标志 → 寄存器（x < y 的布尔值）
LNOT  %r1                // 逻辑非（取反）
CP    %local_result, %r1 // 赋值给 result
```

但实际上如果 `result` 的寄存器地址恰好是编号 2（第三个局部变量），最后一条 CP 的目标地址会是 `-2 - VariableReserveCount - 1`。

**注意**：如果这个表达式出现在 `if` 条件中而非赋值中，编译器会使用条件标志快捷路径：

```
CLT   %local0, %local1   // 比较 x < y → CFLAG
JF    skip                // !(CFLAG) → JF 而非 JNF（零成本翻转）
```

</details>

### 题目 3：手工实现 PutData 去重

请用 C++ 实现一个简化版的 PutData 函数，要求：
1. 维护一个 `std::vector<double>` 作为数据区
2. 新增数据时在最近 N 个条目中搜索重复值
3. 找到重复则返回已有索引，否则追加新条目
4. 测试用例：依次插入 `1.0, 2.0, 3.0, 1.0, 4.0, 2.0`，验证去重效果

<details>
<summary>查看答案</summary>

```cpp
// 文件：put_data_demo.cpp
// 编译：g++ -std=c++17 -o put_data_demo put_data_demo.cpp
// 运行：./put_data_demo

#include <cmath>
#include <iostream>
#include <vector>

class SimpleDataArea {
    std::vector<double> data_;
    int dedup_window_;  // 去重搜索窗口大小

public:
    explicit SimpleDataArea(int window = 20) : dedup_window_(window) {}

    // 模拟 TJS2 的 PutData
    int put_data(double val) {
        // 在最近 dedup_window_ 个条目中搜索
        if (!data_.empty()) {
            int start = (int)data_.size() - 1;
            int count = 0;
            for (int i = start; count < dedup_window_ && i >= 0;
                 i--, count++) {
                // 严格比较（模拟 DiscernCompareStrictReal）
                if (data_[i] == val) {
                    std::cout << "  PutData(" << val << ") → 复用索引 "
                              << i << std::endl;
                    return i;
                }
            }
        }
        // 未找到重复，追加新条目
        int idx = (int)data_.size();
        data_.push_back(val);
        std::cout << "  PutData(" << val << ") → 新建索引 "
                  << idx << std::endl;
        return idx;
    }

    void dump() const {
        std::cout << "数据区: [";
        for (int i = 0; i < (int)data_.size(); i++) {
            if (i > 0) std::cout << ", ";
            std::cout << "d" << i << "=" << data_[i];
        }
        std::cout << "]" << std::endl;
    }
};

int main() {
    SimpleDataArea area(20);

    std::cout << "=== PutData 去重演示 ===" << std::endl;

    double values[] = {1.0, 2.0, 3.0, 1.0, 4.0, 2.0};
    int indices[6];
    for (int i = 0; i < 6; i++) {
        indices[i] = area.put_data(values[i]);
    }

    std::cout << "\n最终状态:" << std::endl;
    area.dump();

    std::cout << "\n返回的索引: ";
    for (int i = 0; i < 6; i++) {
        std::cout << indices[i];
        if (i < 5) std::cout << ", ";
    }
    std::cout << std::endl;

    std::cout << "\n验证: 第 4 次插入 1.0 返回索引 "
              << indices[3] << " (应为 0)" << std::endl;
    std::cout << "验证: 第 6 次插入 2.0 返回索引 "
              << indices[5] << " (应为 1)" << std::endl;
    std::cout << "数据区大小: " << 4 << " (而非 6，节省了 2 个条目)"
              << std::endl;

    return 0;
}
```

预期输出：

```
=== PutData 去重演示 ===
  PutData(1) → 新建索引 0
  PutData(2) → 新建索引 1
  PutData(3) → 新建索引 2
  PutData(1) → 复用索引 0
  PutData(4) → 新建索引 3
  PutData(2) → 复用索引 1

最终状态:
数据区: [d0=1, d1=2, d2=3, d3=4]

返回的索引: 0, 1, 2, 0, 3, 1

验证: 第 4 次插入 1.0 返回索引 0 (应为 0)
验证: 第 6 次插入 2.0 返回索引 1 (应为 1)
数据区大小: 4 (而非 6，节省了 2 个条目)
```

</details>

---

## 下一步

[寄存器分配与帧管理](./02-寄存器分配与帧管理.md) —— 深入理解 FrameBase/frame 临时寄存器机制、局部变量的负地址编码、VariableReserveCount 的作用，以及 ClearFrame 如何回收寄存器。

