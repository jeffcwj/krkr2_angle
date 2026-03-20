# AST 构建与节点类型

> **所属模块：** M07-TJS2脚本引擎
> **前置知识：** [01-bison语法文件详解](./01-bison语法文件详解.md)、[P08-编译原理基础](../../P08-编译原理基础/)
> **预计阅读时间：** 60 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：
1. 理解 `tTJSExprNode` 类的四个核心字段（`Op`、`Position`、`Nodes`、`Val`）及其含义
2. 掌握 `MakeNP0`/`MakeNP1`/`MakeNP2`/`MakeNP3` 四个工厂函数的用法与常量折叠（Constant Folding）优化
3. 画出任意 TJS2 表达式的完整 AST 树形结构
4. 理解 `CurrentNodeVector` 栈在内联数组/字典构建中的作用
5. 理解 `NodeToDeleteVector` 的内存管理策略
6. 理解 `GenNodeCode` 如何将 AST 翻译为字节码（概念桥接，详见下一章）

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| AST | Abstract Syntax Tree | 抽象语法树——源代码经语法分析后生成的树状数据结构，每个节点代表一个语法构造（运算符、常量、变量等） |
| 表达式节点 | Expression Node | TJS2 中用 `tTJSExprNode` 类表示的 AST 节点，存储操作码（opcode）、源码位置、子节点指针和可选值 |
| 操作码 | Opcode | 节点类型标识符，如 `T_PLUS`（加法）、`T_CONSTVAL`（常量值）、`T_SYMBOL`（标识符），用 Token 枚举值表示 |
| 工厂函数 | Factory Function | 创建并返回对象实例的函数——`MakeNP0`~`MakeNP3` 按子节点数量分别创建 0~3 个子节点的 AST 节点 |
| 常量折叠 | Constant Folding | 编译期优化：当表达式的所有操作数都是常量时，直接计算结果，避免运行时计算。如 `1+2` 折叠为 `3` |
| 节点删除向量 | Node-To-Delete Vector | `NodeToDeleteVector`——语法分析期间分配的所有 AST 节点的集中注册表，编译完成后统一释放 |
| 当前节点栈 | Current Node Stack | `CurrentNodeVector`——用于内联数组/字典构建的栈结构，`PushCurrentNode`/`PopCurrentNode` 操作 |
| 常量池 | Data Area / Constant Pool | `_DataArea`——编译期存储常量值（字符串、数字等）的数组，运行时由 `DataArea` 持有最终版本 |

---

## 1. AST 在 TJS2 编译流水线中的位置

在前一节我们学习了 Bison 语法文件 `tjs.y` 的规则结构。每条语法规则的动作代码（`{ ... }` 部分）都在构建 AST 节点。AST 是语法分析器的输出、字节码编译器的输入，是整个编译流水线的核心中间表示（Intermediate Representation, IR——编译器内部使用的数据结构，将源代码转换为更适合优化和代码生成的形式）。

```
源代码文本
    │
    ▼
┌──────────┐
│ 词法分析 │  tjsLex.cpp
│ (Lexer)  │  → Token 流: T_SYMBOL, T_PLUS, T_CONSTVAL, ...
└──────────┘
    │
    ▼
┌──────────┐
│ 语法分析 │  tjs.y (Bison 生成)
│ (Parser) │  → AST 树 (tTJSExprNode 节点构成)   ← 本节重点
└──────────┘
    │
    ▼
┌──────────┐
│ 字节码   │  tjsInterCodeGen.cpp
│ 编译器   │  → VM 指令序列 (CodeArea) + 常量池 (DataArea)
└──────────┘
    │
    ▼
┌──────────┐
│ 虚拟机   │  tjsInterCodeExec.cpp
│ (VM)     │  → 执行结果
└──────────┘
```

> **关键理解**：TJS2 的 AST 并不是一个独立的数据结构。它在语法分析过程中被创建，在字节码编译过程中被遍历生成指令，然后被立即销毁。AST 节点不会被序列化保存，也不会被传递到 VM 执行阶段。这与一些现代语言编译器（如 Rust 的 HIR/MIR）的多趟编译不同——TJS2 采用的是单趟编译（One-pass Compilation）策略，语法分析和代码生成交织进行。

### 1.1 语句与表达式的不同处理路径

在 TJS2 中，并非所有语法构造都经过 AST。我们需要区分两种处理方式：

**（1）语句级构造——直接调用 Context 方法**

控制流语句（`while`、`if`、`for` 等）在 Bison 动作中直接调用 `tTJSInterCodeContext` 的 Enter/Exit 方法，绕过 AST 节点：

```cpp
// tjs.y 第 254-257 行 — while 语句
while
    : "while"                           { cc->EnterWhileCode(false); }
      "(" expr ")"                      { cc->CreateWhileExprCode($4, false); }
      block_or_statement                { cc->ExitWhileCode(false); }
;
```

这里 `EnterWhileCode` 和 `ExitWhileCode` 直接在 `NestVector`（嵌套栈——编译器内部用于跟踪当前嵌套的控制结构的栈，存储循环/条件的入口地址、跳转补丁点等信息）上操作，直接生成字节码。只有条件表达式 `$4` 是作为 AST 节点传入的。

**（2）表达式——构建 AST 树后遍历编译**

所有表达式（算术运算、赋值、函数调用等）在 Bison 动作中被构建成 AST 节点树，然后通过 `CreateExprCode` → `GenNodeCode` 递归遍历生成字节码：

```cpp
// tjs.y 第 225 行 — 表达式语句
| expr ";"          { cc->CreateExprCode($1); }
```

`CreateExprCode` 是表达式 AST 到字节码的入口：

```cpp
// tjsInterCodeGen.cpp 第 2913-2918 行
void tTJSInterCodeContext::CreateExprCode(tTJSExprNode *node) {
    tjs_int fr = FrameBase;           // 获取当前寄存器帧基址
    GenNodeCode(fr, node, 0, 0, tSubParam());  // 递归编译整棵 AST 树
    ClearFrame(fr);                   // 清理临时寄存器
}
```

> **设计取舍**：为什么语句不走 AST？因为控制流语句需要在特定时刻生成跳转指令并记录补丁地址（Patch Address——编译时暂时填入占位值的跳转目标地址，待目标确定后再回填正确值）。如果先构建完整 AST 再遍历，就需要额外记录这些位置信息，增加复杂度。直接在语法规则的动作中逐步生成字节码更简洁。表达式则不同，表达式可能需要常量折叠等优化，用 AST 树更方便处理。

---

## 2. tTJSExprNode 类——AST 的基本构建块

`tTJSExprNode` 是 TJS2 AST 的唯一节点类型。与许多编译器使用继承层次（如 `BinaryExpr`、`UnaryExpr`、`LiteralExpr` 等子类）不同，TJS2 用一个通用节点类配合不同的操作码来区分节点类型。这种"扁平"设计简化了内存管理，但需要在每个操作码的处理代码中自行检查子节点数量。

### 2.1 类定义（源码位置：`tjsInterCodeGen.h` 第 181-234 行）

```cpp
// cpp/core/tjs2/tjsInterCodeGen.h 第 181-234 行
class tTJSExprNode {
    tjs_int Op;                              // ① 操作码——标识这个节点代表什么操作
    tjs_int Position;                        // ② 源码位置——在源代码中的字符偏移量
    std::vector<tTJSExprNode *> *Nodes;      // ③ 子节点列表——指向子节点的指针向量
    tTJSVariant *Val;                        // ④ 值——常量值、符号名等

public:
    tTJSExprNode();                          // 构造函数：Op=0, Nodes=nullptr, Val=nullptr
    ~tTJSExprNode() { Clear(); }             // 析构函数：释放 Nodes 和 Val

    void Clear();                            // 释放 Nodes 向量和 Val，重置为 nullptr
    void SetOpcode(tjs_int op) { Op = op; }  // 设置操作码
    void SetPosition(tjs_int pos) { Position = pos; }  // 设置源码位置
    void SetValue(const tTJSVariant &val);   // 设置值（懒分配 Val 指针）

    void Add(tTJSExprNode *n);               // 添加一个子节点

    tjs_int GetOpcode() const { return Op; }
    tjs_int GetPosition() const { return Position; }
    tTJSVariant &GetValue();                 // 获取值（注意：Val 为 nullptr 时行为未定义）

    tTJSExprNode *operator[](tjs_int i) const;  // 获取第 i 个子节点
    unsigned int GetSize() const;               // 获取子节点数量

    // 以下两个方法仅用于编译时常量数组/字典
    void AddArrayElement(const tTJSVariant &val);
    void AddDictionaryElement(const tTJSString &name, const tTJSVariant &val);
};
```

### 2.2 四个核心字段详解

#### 字段 ① `Op`（操作码）

`Op` 字段存储的是 Token 枚举值，与 Bison 生成的 `parser::token` 枚举完全一致。不同的操作码含义截然不同：

| 操作码类别 | 示例 | 子节点数 | Val 含义 |
|-----------|------|---------|---------|
| 常量 | `T_CONSTVAL` | 0 | 常量值（整数/浮点/字符串/八进制序列） |
| 标识符 | `T_SYMBOL` | 0 | 符号名（`tTJSVariant` 包裹的字符串） |
| 关键字 | `T_THIS`、`T_SUPER`、`T_GLOBAL`、`T_VOID` | 0 | 无 |
| 一元运算 | `T_EXCRAMATION`（`!`）、`T_UMINUS`（`-`） | 1 | 无 |
| 二元运算 | `T_PLUS`、`T_MINUS`、`T_DOT` | 2 | 无 |
| 三元运算 | `T_QUESTION`（`?:`） | 3 | 无 |
| 赋值 | `T_EQUAL`、`T_PLUSEQUAL` | 2 | 无 |
| 函数调用 | `T_LPARENTHESIS` | 2 | 无（node[0]=函数，node[1]=参数列表） |
| 数组下标 | `T_LBRACKET` | 2 | 无（node[0]=对象，node[1]=下标） |
| 属性访问 | `T_DOT` | 2 | 无（node[0]=对象，node[1]=属性名常量） |
| 内联数组 | `T_INLINEARRAY` | N | 无（每个子节点是 `T_ARRAYARG`） |
| 内联字典 | `T_INLINEDIC` | N | 无（每个子节点是 `T_DICELM`） |
| 内部标记 | `T_ARG`、`T_EXPANDARG`、`T_ARRAYARG`、`T_DICELM` | 1-2 | 无 |

> **注意**：操作码复用了 Token 枚举值，但不是所有 Token 都能作为操作码。前一节提到的 17 个内部 Token（如 `T_UPLUS`、`T_POSTDECREMENT`、`T_INLINEARRAY` 等）就是只用作操作码、不会被词法分析器产生的值。

#### 字段 ② `Position`（源码位置）

`Position` 存储的是这个节点对应的源代码字符偏移量（从文件开头算起的字符索引）。这个值来自 `LEX_POS` 宏：

```cpp
// tjsInterCodeGen.cpp 第 25 行
#define LEX_POS (Block->GetLexicalAnalyzer()->GetCurrentPosition())
```

> **调试用途**：当运行时出现异常时，VM 通过 `CodePosToSrcPos` 方法将字节码地址映射回源码位置，再通过 `SrcPosToLine` 计算行号。`Position` 就是这条信息链的起点——在编译阶段从 AST 节点传递到 `SourcePosArray` 中。

#### 字段 ③ `Nodes`（子节点向量）

`Nodes` 是一个指向 `std::vector<tTJSExprNode *>` 的指针，采用懒分配策略：

```cpp
// tjsInterCodeGen.cpp 第 111-115 行
void tTJSExprNode::Add(tTJSExprNode *n) {
    if(!Nodes)
        Nodes = new std::vector<tTJSExprNode *>;  // 首次添加时才分配
    Nodes->push_back(n);
}
```

为什么用指针而不直接嵌入 `std::vector`？因为叶子节点（常量、标识符）不需要子节点向量，使用指针可以省下 `sizeof(std::vector)` 的内存开销（通常 24 字节）。在一棵典型的 AST 中，叶子节点占多数，这个优化是有意义的。

子节点的访问通过重载的 `operator[]` 完成：

```cpp
// tjsInterCodeGen.h 第 217-221 行
tTJSExprNode *operator[](tjs_int i) const {
    if(!Nodes)
        return nullptr;        // 没有子节点时安全返回 nullptr
    return (*Nodes)[i];        // 注意：不做越界检查
}
```

> **常见用法**：在 `GenNodeCode` 中，二元运算节点的左操作数用 `(*node)[0]` 访问，右操作数用 `(*node)[1]` 访问。三元 `?:` 运算符的条件、真值、假值分别是 `[0]`、`[1]`、`[2]`。

#### 字段 ④ `Val`（值）

`Val` 是一个指向 `tTJSVariant` 的指针，同样采用懒分配：

```cpp
// tjsInterCodeGen.h 第 198-203 行
void SetValue(const tTJSVariant &val) {
    if(!Val)
        Val = new tTJSVariant(val);  // 首次设置时分配
    else
        Val->CopyRef(val);           // 后续设置时复用已有对象
}
```

`tTJSVariant` 是 TJS2 的万能值类型（类似 JavaScript 中的变量——可以存储整数、浮点数、字符串、对象引用、八进制序列等任意类型的值），在这里它主要用于存储：

- **常量值**：`T_CONSTVAL` 节点的字面量值，如 `42`、`3.14`、`"hello"`
- **符号名**：`T_SYMBOL` 节点的标识符字符串
- **正则表达式**：`T_REGEXP` 节点的正则字面量
- **匿名函数引用**：`func_expr_def` 产生的 `T_CONSTVAL` 节点中存储 `tTJSVariant(cc)`——一个指向新创建的函数编译上下文的引用

### 2.3 内存管理——节点生命周期

AST 节点的生命周期由 `NodeToDeleteVector` 统一管理。这是一种简单但有效的内存管理策略——类似于区域分配器（Arena Allocator——一次性分配大量小对象，然后统一释放，避免逐个 `delete` 的开销和忘记释放的风险）。

```
创建节点               使用节点            销毁节点
──────                ──────             ──────
MakeNP0/1/2/3         Bison 动作拼装        Commit() 或
MakeConstValNode      GenNodeCode 遍历      ClearNodesToDelete()
    │                      │                    │
    ├──→ new tTJSExprNode  ├──→ node->GetOpcode()   ├──→ delete 每个节点
    ├──→ 加入 NodeToDeleteVector  ├──→ (*node)[0]     └──→ 清空 Vector
    └──→ 返回指针          └──→ node->GetValue()
```

具体实现：

```cpp
// tjsInterCodeGen.cpp 第 416-424 行
void tTJSInterCodeContext::ClearNodesToDelete() {
    if(NodeToDeleteVector.size()) {
        for(tjs_int i = (tjs_int)(NodeToDeleteVector.size() - 1); i >= 0;
            i--)
            delete NodeToDeleteVector[i];  // 逆序删除，后创建的先删除
    }
    NodeToDeleteVector.clear();
}
```

> **逆序删除的原因**：后创建的节点可能引用先创建的节点（通过 `Nodes` 向量）。虽然 `tTJSExprNode::Clear()` 只释放自己的 `Nodes` 向量和 `Val`，不递归释放子节点（子节点由 `NodeToDeleteVector` 统一管理），但逆序删除仍然是更安全的做法。

清理时机有两个：

1. **`Commit()` 调用时**（`tjsInterCodeGen.cpp` 第 998 行）：当一个编译上下文（函数/类/属性）完成编译时，其积累的所有 AST 节点被一次性释放。
2. **编译失败时**：析构函数中通过 `Finalize()` 间接调用清理。

这意味着 AST 节点的生命周期与编译上下文（`tTJSInterCodeContext`）绑定——每编译完一个函数或一个顶层代码块，对应的 AST 节点就被全部释放。

### 2.4 特殊方法：AddArrayElement 和 AddDictionaryElement

这两个方法仅用于编译时常量数组/字典（`(const)[...]` 和 `(const)%[...]` 语法），它们直接操作 `Val` 中存储的数组/字典对象：

```cpp
// tjsInterCodeGen.cpp 第 118-133 行
void tTJSExprNode::AddArrayElement(const tTJSVariant &val) {
    static tTJSString ss_add(TJS_W("add"));
    tTJSVariant arg(val);
    tTJSVariant *args[1] = { &arg };
    // 调用 Val 所引用的数组对象的 "add" 方法
    Val->AsObjectClosureNoAddRef().FuncCall(
        0, ss_add.c_str(), ss_add.GetHint(), nullptr, 1, args, nullptr);
}

void tTJSExprNode::AddDictionaryElement(const tTJSString &name,
                                        const tTJSVariant &val) {
    tTJSString membername(name);
    // 调用 Val 所引用的字典对象的 PropSet 方法
    Val->AsObjectClosureNoAddRef().PropSet(
        TJS_MEMBERENSURE, membername.c_str(), membername.GetHint(), &val,
        nullptr);
}
```

> **编译时执行**：注意这是在编译阶段就真实创建了数组/字典对象并填入数据。这与普通的内联数组 `[1, 2, 3]`（运行时构建）不同——`(const)[1, 2, 3]` 在编译时就完成了构建，运行时直接作为常量使用。这是一种编译时求值优化，类似 C++ 的 `constexpr`。

---

## 3. MakeNP 工厂函数族——AST 节点的创建

所有 AST 节点都通过四个工厂函数创建，它们是 Bison 动作代码中最频繁调用的方法。函数名中的数字表示子节点数量。

### 3.1 MakeNP0——零子节点（叶子节点）

`MakeNP0` 创建没有子节点的叶子节点，用于常量值、标识符、关键字等：

```cpp
// tjsInterCodeGen.cpp 第 3800-3808 行
tTJSExprNode *tTJSInterCodeContext::MakeNP0(tjs_int opecode) {
    tTJSExprNode *n = new tTJSExprNode;       // ① 分配新节点
    NodeToDeleteVector.push_back(n);           // ② 注册到删除列表
    n->SetOpcode(opecode);                     // ③ 设置操作码
    n->SetPosition(LEX_POS);                   // ④ 记录源码位置
    return n;                                  // ⑤ 返回节点指针
}
```

在 Bison 语法文件中的典型使用场景：

```cpp
// tjs.y 第 696-704 行 — factor_expr 规则
factor_expr
    : T_CONSTVAL    { $$ = cc->MakeNP0(token::T_CONSTVAL);   // 常量字面量节点
                      $$->SetValue(lx->GetValue($1)); }       // 设置常量值
    | T_SYMBOL      { $$ = cc->MakeNP0(token::T_SYMBOL);      // 标识符节点
                      $$->SetValue(tTJSVariant(
                          lx->GetString($1))); }                // 设置符号名
    | "this"        { $$ = cc->MakeNP0(token::T_THIS); }       // this 关键字
    | "super"       { $$ = cc->MakeNP0(token::T_SUPER); }      // super 关键字
    | "global"      { $$ = cc->MakeNP0(token::T_GLOBAL); }     // global 关键字
    | "void"        { $$ = cc->MakeNP0(token::T_VOID); }       // void 关键字
;
```

> **关键细节**：`T_CONSTVAL` 和 `T_SYMBOL` 节点创建后还需要额外调用 `SetValue`。`$1` 是词法分析器通过 `%union` 的 `num` 成员传递的值索引——调用 `lx->GetValue($1)` 或 `lx->GetString($1)` 从词法分析器的 `Values` 数组中取出实际值。

### 3.2 MakeNP1——一元运算节点

`MakeNP1` 创建只有一个子节点的节点，用于一元运算符：

```cpp
// tjsInterCodeGen.cpp 第 3812-3884 行（简化）
tTJSExprNode *tTJSInterCodeContext::MakeNP1(tjs_int opecode,
                                            tTJSExprNode *node1) {
    // ★ 常量折叠优化（编译期计算）
    if(node1 && node1->GetOpcode() == parser::token_kind_type::T_CONSTVAL) {
        tTJSExprNode *ret = nullptr;
        switch(opecode) {
            case T_EXCRAMATION:  // !常量
                ret = MakeConstValNode(!node1->GetValue());
                break;
            case T_TILDE:        // ~常量
                ret = MakeConstValNode(~node1->GetValue());
                break;
            case T_UPLUS:        // +常量（转数字）
                { tTJSVariant val(node1->GetValue());
                  val.tonumber();
                  ret = MakeConstValNode(val); }
                break;
            case T_UMINUS:       // -常量（取反）
                { tTJSVariant val(node1->GetValue());
                  val.changesign();
                  ret = MakeConstValNode(val); }
                break;
            case T_INT:          // (int)常量
                { tTJSVariant val(node1->GetValue());
                  val.ToInteger();
                  ret = MakeConstValNode(val); }
                break;
            case T_REAL:         // (real)常量
                { tTJSVariant val(node1->GetValue());
                  val.ToReal();
                  ret = MakeConstValNode(val); }
                break;
            case T_STRING:       // (string)常量
                { tTJSVariant val(node1->GetValue());
                  val.ToString();
                  ret = MakeConstValNode(val); }
                break;
            // ... T_SHARP(#)、T_DOLLAR($)、T_OCTET 同理
        }
        if(ret) {
            node1->Clear();    // 清理原节点的 Val，避免重复引用
            return ret;        // 返回折叠后的常量节点
        }
    }

    // 无法折叠时，正常创建一元节点
    tTJSExprNode *n = new tTJSExprNode;
    NodeToDeleteVector.push_back(n);
    n->SetOpcode(opecode);
    n->SetPosition(LEX_POS);
    n->Add(node1);              // 添加唯一的子节点
    return n;
}
```

在语法文件中的使用示例：

```cpp
// tjs.y 第 642-668 行 — unary_expr 规则（节选）
unary_expr
    : "!" unary_expr            { $$ = cc->MakeNP1(token::T_EXCRAMATION, $2); }
    | "~" unary_expr            { $$ = cc->MakeNP1(token::T_TILDE, $2); }
    | "--" unary_expr           { $$ = cc->MakeNP1(token::T_DECREMENT, $2); }
    | "++" unary_expr           { $$ = cc->MakeNP1(token::T_INCREMENT, $2); }
    | "invalidate" unary_expr   { $$ = cc->MakeNP1(token::T_INVALIDATE, $2); }
    | "delete" unary_expr       { $$ = cc->MakeNP1(token::T_DELETE, $2); }
    | "typeof" unary_expr       { $$ = cc->MakeNP1(token::T_TYPEOF, $2); }
    | "+" unary_expr            { $$ = cc->MakeNP1(token::T_UPLUS, $2); }
    | "-" unary_expr            { $$ = cc->MakeNP1(token::T_UMINUS, $2); }
    | "&" unary_expr            { $$ = cc->MakeNP1(token::T_IGNOREPROP, $2); }
    | "*" unary_expr            { $$ = cc->MakeNP1(token::T_PROPACCESS, $2); }
;
```

### 3.3 MakeNP2——二元运算节点

`MakeNP2` 是使用频率最高的工厂函数，创建包含两个子节点的节点：

```cpp
// tjsInterCodeGen.cpp 第 3888-3979 行（简化）
tTJSExprNode *tTJSInterCodeContext::MakeNP2(tjs_int opecode,
                                            tTJSExprNode *node1,
                                            tTJSExprNode *node2) {
    // ★ 二元常量折叠：两个操作数都是常量时直接计算
    if(node1 && node1->GetOpcode() == T_CONSTVAL &&
       node2 && node2->GetOpcode() == T_CONSTVAL) {
        switch(opecode) {
            case T_PLUS:      return MakeConstValNode(node1->GetValue() + node2->GetValue());
            case T_MINUS:     return MakeConstValNode(node1->GetValue() - node2->GetValue());
            case T_ASTERISK:  return MakeConstValNode(node1->GetValue() * node2->GetValue());
            case T_SLASH:     return MakeConstValNode(node1->GetValue() / node2->GetValue());
            case T_BACKSLASH: return MakeConstValNode(node1->GetValue().idiv(node2->GetValue()));
            case T_PERCENT:   return MakeConstValNode(node1->GetValue() % node2->GetValue());
            case T_LOGICALOR: return MakeConstValNode(node1->GetValue() || node2->GetValue());
            case T_LOGICALAND:return MakeConstValNode(node1->GetValue() && node2->GetValue());
            case T_EQUALEQUAL:return MakeConstValNode(node1->GetValue() == node2->GetValue());
            case T_NOTEQUAL:  return MakeConstValNode(node1->GetValue() != node2->GetValue());
            case T_LT:        return MakeConstValNode(node1->GetValue() < node2->GetValue());
            case T_GT:        return MakeConstValNode(node1->GetValue() > node2->GetValue());
            // ... 共 21 种二元操作都支持常量折叠
            case T_COMMA:     return MakeConstValNode(node2->GetValue());  // 逗号表达式取右值
        }
    }

    // 无法折叠时，正常创建二元节点
    tTJSExprNode *n = new tTJSExprNode;
    NodeToDeleteVector.push_back(n);
    n->SetOpcode(opecode);
    n->SetPosition(LEX_POS);
    n->Add(node1);     // 子节点 [0]：左操作数
    n->Add(node2);     // 子节点 [1]：右操作数
    return n;
}
```

### 3.4 MakeNP3——三元运算节点

`MakeNP3` 仅用于三元条件运算符 `? :`：

```cpp
// tjsInterCodeGen.cpp 第 3983-3997 行
tTJSExprNode *tTJSInterCodeContext::MakeNP3(tjs_int opecode,
                                            tTJSExprNode *node1,
                                            tTJSExprNode *node2,
                                            tTJSExprNode *node3) {
    tTJSExprNode *n = new tTJSExprNode;
    NodeToDeleteVector.push_back(n);
    n->SetOpcode(opecode);
    n->SetPosition(LEX_POS);
    n->Add(node1);     // 子节点 [0]：条件表达式
    n->Add(node2);     // 子节点 [1]：条件为真时的表达式
    n->Add(node3);     // 子节点 [2]：条件为假时的表达式
    return n;
}
```

语法文件中的使用：

```cpp
// tjs.y 第 565-570 行
cond_expr
    : logical_or_expr                       { $$ = $1; }
    | logical_or_expr "?"
        cond_expr ":"
        cond_expr                           { $$ = cc->MakeNP3(token::T_QUESTION, $1, $3, $5); }
;
```

> **注意**：`MakeNP3` 没有常量折叠优化。理论上可以在三个操作数都是常量时折叠（如 `true ? 1 : 2` → `1`），但 TJS2 选择不实现这个优化。

### 3.5 MakeConstValNode——辅助工厂函数

`MakeConstValNode` 是常量折叠的专用辅助函数，创建一个 `T_CONSTVAL` 节点并设置值：

```cpp
// tjsInterCodeGen.cpp 第 3787-3793 行
tTJSExprNode *tTJSInterCodeContext::MakeConstValNode(const tTJSVariant &val) {
    tTJSExprNode *n = new tTJSExprNode;
    NodeToDeleteVector.push_back(n);
    n->SetOpcode(parser::token::T_CONSTVAL);  // 操作码固定为 T_CONSTVAL
    n->SetValue(val);                          // 设置折叠后的常量值
    n->SetPosition(LEX_POS);
    return n;
}
```

### 3.6 常量折叠的传递效应

常量折叠具有传递效应——当子表达式折叠为常量后，父表达式可能也变成常量可折叠的。例如：

```javascript
// TJS2 源代码
var x = 1 + 2 * 3;
```

编译过程中的折叠链：

```
步骤 1: MakeNP0(T_CONSTVAL) → 值 2
步骤 2: MakeNP0(T_CONSTVAL) → 值 3
步骤 3: MakeNP2(T_ASTERISK, [2], [3])
        → 两个操作数都是 T_CONSTVAL
        → 常量折叠: MakeConstValNode(6)     ← 2*3=6
步骤 4: MakeNP0(T_CONSTVAL) → 值 1
步骤 5: MakeNP2(T_PLUS, [1], [6])
        → 两个操作数都是 T_CONSTVAL
        → 常量折叠: MakeConstValNode(7)     ← 1+6=7
```

最终 `1 + 2 * 3` 在编译时就被计算为 `7`，运行时只需执行一条 `VM_CONST` 指令把 `7` 加载到寄存器。

**不会被折叠的情况**：

```javascript
var a = 10;
var b = a + 5;       // a 是 T_SYMBOL，不是 T_CONSTVAL → 不折叠
var c = foo() + 1;   // foo() 是函数调用 → 不折叠
var d = [1, 2, 3];   // 内联数组 → 不折叠（但 (const)[1,2,3] 在编译时构建）
```

> **TJS2 常量折叠的完整覆盖范围**：10 种一元操作（`!`、`~`、`#`、`$`、`+`、`-`、`(int)`、`(real)`、`(string)`、`(octet)`）+ 21 种二元操作（四则运算、位运算、逻辑运算、比较运算、移位运算、逗号表达式）。不包含三元运算 `?:`。这种覆盖范围在脚本语言中算是相当全面的。

---

## 4. AST 树形结构实例分析

理解 AST 最好的方法是画出具体代码的树形结构。本节通过多个由简到繁的例子展示 TJS2 的 AST 构建过程。

### 4.1 简单算术表达式

**源代码**：`a + b * 2`

Bison 按优先级规则先规约 `b * 2`（乘法优先），再规约加法。构建过程：

```
步骤 1: 规约 T_SYMBOL "a"     → MakeNP0(T_SYMBOL), SetValue("a")
步骤 2: 规约 T_SYMBOL "b"     → MakeNP0(T_SYMBOL), SetValue("b")
步骤 3: 规约 T_CONSTVAL 2     → MakeNP0(T_CONSTVAL), SetValue(2)
步骤 4: 规约 b * 2             → MakeNP2(T_ASTERISK, [b节点], [2节点])
步骤 5: 规约 a + (b*2)         → MakeNP2(T_PLUS, [a节点], [*节点])
```

生成的 AST 树：

```
            T_PLUS
           ╱      ╲
     T_SYMBOL    T_ASTERISK
     Val="a"     ╱       ╲
            T_SYMBOL   T_CONSTVAL
            Val="b"    Val=2
```

### 4.2 赋值表达式

**源代码**：`x = y + 1`

赋值运算符是右结合的（从右向左求值），但在语法规则中 `assign_expr` 采用了递归写法 `cond_expr "=" assign_expr`，自然实现了右结合：

```
            T_EQUAL
           ╱      ╲
     T_SYMBOL    T_PLUS
     Val="x"    ╱      ╲
          T_SYMBOL   T_CONSTVAL
          Val="y"    Val=1
```

### 4.3 属性访问与函数调用

**源代码**：`obj.method(arg1, arg2)`

这是一个典型的方法调用。首先构建属性访问 `obj.method`，然后将其作为函数调用的被调函数：

```
                T_LPARENTHESIS                    ← 函数调用
               ╱              ╲
          T_DOT              T_ARG                ← 参数列表（链表形式）
         ╱     ╲            ╱      ╲
   T_SYMBOL  T_CONSTVAL  T_SYMBOL   T_ARG
   Val="obj" Val="method" Val="arg2" ╱  ╲
                                T_SYMBOL  (null)
                                Val="arg1"
```

> **注意参数列表的构建**：参数列表用 `T_ARG` 节点构成反向链表。第一个参数创建 `MakeNP1(T_ARG, $1)`，后续参数创建 `MakeNP2(T_ARG, $3, $1)`，其中 `$1` 是前面的参数列表、`$3` 是新参数。所以参数链的根节点指向最后一个参数，末尾节点指向第一个参数。

来看语法规则如何构建：

```cpp
// tjs.y 第 720-735 行
func_call_expr
    : priority_expr "(" call_arg_list ")"  { $$ = cc->MakeNP2(token::T_LPARENTHESIS, $1, $3); }
;

call_arg_list
    : "..."                    { $$ = cc->MakeNP0(token::T_OMIT); }       // 省略参数
    | call_arg                 { $$ = cc->MakeNP1(token::T_ARG, $1); }    // 单参数
    | call_arg_list "," call_arg { $$ = cc->MakeNP2(token::T_ARG, $3, $1); } // 多参数
;

call_arg
    : /* empty */              { $$ = NULL; }                    // 空参数（占位）
    | "*"                      { $$ = cc->MakeNP1(token::T_EXPANDARG, NULL); }  // * 展开
    | mul_div_expr_and_asterisk { $$ = cc->MakeNP1(token::T_EXPANDARG, $1); }   // expr* 展开
    | expr_no_comma            { $$ = $1; }                                      // 普通参数
;
```

### 4.4 属性访问中的 DOT 语法细节

属性访问 `obj.prop` 的处理值得特别关注。语法规则中有一个关键的词法分析器交互：

```cpp
// tjs.y 第 681-684 行
priority_expr
    : priority_expr "."         { lx->SetNextIsBareWord(); }     // ★ 关键
      T_SYMBOL                  { tTJSExprNode *node = cc->MakeNP0(token::T_CONSTVAL);
                                  node->SetValue(lx->GetValue($4));
                                  $$ = cc->MakeNP2(token::T_DOT, $1, node); }
;
```

`SetNextIsBareWord()` 告诉词法分析器：下一个标识符是"裸词"（Bare Word——不作为关键字解析的标识符），即使它恰好与关键字同名也按普通标识符处理。例如 `obj.class` 中的 `class` 不会被解析为 `T_CLASS` 关键字，而是作为 `T_SYMBOL` 返回。

属性名被创建为 `T_CONSTVAL` 节点（而不是 `T_SYMBOL`），因为属性名在这个上下文中是一个字符串常量，不是一个需要查找的变量。

### 4.5 三元条件表达式

**源代码**：`a > 0 ? a : -a`

```
              T_QUESTION
            ╱     │     ╲
      T_GT       T_SYMBOL   T_UMINUS
     ╱    ╲      Val="a"       │
T_SYMBOL  T_CONSTVAL         T_SYMBOL
Val="a"   Val=0              Val="a"
```

三元节点 `T_QUESTION` 的三个子节点：
- `[0]` = 条件（`a > 0`）
- `[1]` = 真值（`a`）
- `[2]` = 假值（`-a`）

### 4.6 后缀 if 表达式

**源代码**：`foo() if bar`

TJS2 支持 Ruby 风格的后缀 `if`，在表达式之后附加条件：

```cpp
// tjs.y 第 532-535 行
expr
    : comma_expr                           { $$ = $1; }
    | comma_expr "if" expr                 { $$ = cc->MakeNP2(token::T_IF, $1, $3); }
;
```

生成的 AST：

```
              T_IF
             ╱    ╲
    T_LPARENTHESIS  T_SYMBOL
       ╱    ╲       Val="bar"
  T_SYMBOL  (null参数列表)
  Val="foo"
```

节点 `[0]` 是被有条件执行的表达式（函数调用），`[1]` 是条件。注意这与前缀 `if` 语句不同——前缀 `if` 直接调用 `cc->EnterIfCode()`/`cc->CreateIfExprCode()`，不创建 AST 节点。

### 4.7 复合赋值表达式

**源代码**：`a += b * c`

```
          T_PLUSEQUAL
          ╱         ╲
    T_SYMBOL       T_ASTERISK
    Val="a"       ╱         ╲
             T_SYMBOL     T_SYMBOL
             Val="b"      Val="c"
```

所有 16 种复合赋值运算符（`+=`、`-=`、`*=`、`/=`、`\\=`、`%=`、`&=`、`|=`、`^=`、`||=`、`&&=`、`>>=`、`<<=`、`>>>=`、`<->`）都生成相同结构的二元节点，只是操作码不同。`GenNodeCode` 在处理时会将其拆解为"先求值右侧，再与左侧执行运算，最后写回左侧"的指令序列。

### 4.8 new 运算符

**源代码**：`new ClassName(arg)`

`new` 的语法处理非常巧妙——它复用了函数调用的 AST 结构：

```cpp
// tjs.y 第 648 行
| "new" func_call_expr   { $$ = $2; $$->SetOpcode(token::T_NEW); }
```

先按正常函数调用 `ClassName(arg)` 构建 `T_LPARENTHESIS` 节点，然后把操作码改写为 `T_NEW`。所以 `new ClassName(arg)` 的 AST 与 `ClassName(arg)` 结构完全一样，只是根节点操作码从 `T_LPARENTHESIS` 变成了 `T_NEW`：

```
              T_NEW
             ╱     ╲
       T_SYMBOL    T_ARG
       Val="ClassName"  │
                    T_SYMBOL
                    Val="arg"
```

### 4.9 后缀运算符

**源代码**：`i++`

```cpp
// tjs.y 第 685-686 行
| priority_expr "++"    { $$ = cc->MakeNP1(token::T_POSTINCREMENT, $1); }
| priority_expr "--"    { $$ = cc->MakeNP1(token::T_POSTDECREMENT, $1); }
```

后缀 `++`/`--` 使用内部 Token `T_POSTINCREMENT`/`T_POSTDECREMENT`（不同于前缀 `++`/`--` 的 `T_INCREMENT`/`T_DECREMENT`），以便 `GenNodeCode` 能区分前后缀的行为差异。

### 4.10 后缀求值运算符 `!`

**源代码**：`dict.value!`

```cpp
// tjs.y 第 687 行
| priority_expr "!"    { $$ = cc->MakeNP1(token::T_EVAL, $1); }
```

后缀 `!` 是 TJS2 特有的运算符，强制对属性值进行求值（evaluate）。当一个属性本身也是一个可调用对象时，不加 `!` 只取属性引用，加了 `!` 则执行该属性作为函数并返回结果。生成的 AST 使用内部 Token `T_EVAL`。

---

## 5. CurrentNodeVector 栈式管理

在第 2 节中，我们看到 `tTJSExprNode` 只记录父→子的单向引用。但在解析**内联数组**（Inline Array，如 `[1, 2, 3]`）和**内联字典**（Inline Dictionary，如 `%["key" => "val"]`）时，语法分析器面临一个问题：元素是逐个归约（reduce）的，每归约一个元素就要把它添加到容器节点上，但容器节点本身在中间归约步骤中并不在语法栈的 `$$` 位置。

TJS2 的解决方案是 `CurrentNodeVector`——一个显式的节点栈（Node Stack），用于在归约过程中临时持有"当前正在构建的容器节点"。

### 5.1 三个关键方法

```cpp
// tjsInterCodeGen.cpp 第 3768-3782 行

void tTJSInterCodeContext::PushCurrentNode(tTJSExprNode *node) {
    CurrentNodeVector.push_back(node);   // 将容器节点压入栈顶
}

tTJSExprNode * tTJSInterCodeContext::GetCurrentNode() {
    if(CurrentNodeVector.size() == 0)
        return nullptr;
    return CurrentNodeVector.back();      // 返回栈顶，不弹出
}

void tTJSInterCodeContext::PopCurrentNode() {
    CurrentNodeVector.pop_back();         // 弹出栈顶
}
```

这三个方法构成了经典的栈操作三件套（push/peek/pop）。在语法文件 `tjs.y` 中，它们通过 `cc->` 前缀调用（`cc` 是当前的 `tTJSInterCodeContext` 指针），并配合一个方便宏 `cn`：

```cpp
// tjs.y 第 19 行
#define cn cc->GetCurrentNode()    // cn = Current Node = 栈顶容器
```

### 5.2 内联数组的构建过程

以 `var a = [10, 20, 30];` 为例，看 `tjs.y` 中的语法规则如何配合 `CurrentNodeVector` 完成构建：

```cpp
// tjs.y 第 740-758 行

inline_array
    : "[" { $<np>$ = cc->MakeNP0(token::T_INLINEARRAY);    // ① 创建空的数组容器节点
            cc->PushCurrentNode($<np>$); }                   // ② 压入栈
      inline_array_elm "]"
           { $$ = $<np>2;                                    // ⑤ 取回容器节点作为结果
             cc->PopCurrentNode(); }                         // ⑥ 弹出栈
;

inline_array_elm
    : /* 空 */
    | expr_no_comma          { cn->AddArrayElement($1); }    // ③ 第一个元素
    | inline_array_elm "," expr_no_comma
                             { cn->AddArrayElement($3); }    // ④ 后续元素
;
```

> **`$<np>$` 和 `$<np>2`**：Bison 语法中，`$<np>$` 表示"当前产生式中间动作的值，按 `np`（即 `tTJSExprNode*`）类型解释"。`$<np>2` 表示"产生式第 2 个符号（即中间动作）的值"。这是 Bison 的类型标注语法，解决 union 类型的歧义问题。

完整的执行时序：

```
源代码: [10, 20, 30]

Token 流:  "["  10  ","  20  ","  30  "]"
                                         
Step 1: 归约 "[" → 执行中间动作:
        node = MakeNP0(T_INLINEARRAY)   // 创建 {Op=T_INLINEARRAY, Nodes=null}
        PushCurrentNode(node)            // 栈: [node]
        cn == node                       // cn 宏返回栈顶

Step 2: 归约 expr_no_comma (值 10) → inline_array_elm:
        cn->AddArrayElement(MakeNP0/val(10))
        // node.Nodes = [T_ARRAYARG{val=10}]

Step 3: 归约 inline_array_elm "," expr_no_comma (值 20):
        cn->AddArrayElement(MakeNP0/val(20))
        // node.Nodes = [T_ARRAYARG{val=10}, T_ARRAYARG{val=20}]

Step 4: 归约 inline_array_elm "," expr_no_comma (值 30):
        cn->AddArrayElement(MakeNP0/val(30))
        // node.Nodes = [T_ARRAYARG{val=10}, T_ARRAYARG{val=20}, T_ARRAYARG{val=30}]

Step 5: 归约 "[" {mid} inline_array_elm "]" → inline_array:
        $$ = $<np>2 = node               // 取回容器节点
        PopCurrentNode()                  // 栈: []
```

最终生成的 AST：

```
          T_INLINEARRAY
         ╱      │      ╲
   T_ARRAYARG T_ARRAYARG T_ARRAYARG
     Val=10     Val=20     Val=30
```

`AddArrayElement` 方法（定义在 `tTJSExprNode` 类中）做了两件事：创建一个 `T_ARRAYARG` 包装节点，然后将其追加到容器的 `Nodes` 数组中。

```cpp
// tjsInterCodeGen.h 第 216-222 行（简化）
void AddArrayElement(tTJSExprNode *node) {
    tTJSExprNode *elem = new tTJSExprNode();
    elem->SetOpcode(T_ARRAYARG);
    elem->SetPosition(node ? node->GetPosition() : 0);
    elem->Add(node);                    // 将实际值挂为 T_ARRAYARG 的子节点
    Add(elem);                          // 将 T_ARRAYARG 追加到容器的 Nodes
}
```

### 5.3 内联字典的构建过程

内联字典（Inline Dictionary）用 `%[ ]` 语法表示，例如 `%["name" => "Alice", "age" => 25]`。构建机制与数组完全对称：

```cpp
// tjs.y 第 760-785 行

inline_dic
    : "%[" { $<np>$ = cc->MakeNP0(token::T_INLINEDIC);     // 创建字典容器
             cc->PushCurrentNode($<np>$); }                  // 压栈
      inline_dic_elm "]"
            { $$ = $<np>2;
              cc->PopCurrentNode(); }                        // 弹栈
;

inline_dic_elm
    : /* 空 */
    | inline_dic_elm_pair
    | inline_dic_elm "," inline_dic_elm_pair
;

inline_dic_elm_pair
    : expr_no_comma "=>" expr_no_comma
        { cn->AddDictionaryElement($1, $3); }                // 添加键值对
;
```

`AddDictionaryElement` 与 `AddArrayElement` 类似，但创建的包装节点类型是 `T_DICELM`，且包含两个子节点（键和值）：

```cpp
// tjsInterCodeGen.h（简化）
void AddDictionaryElement(tTJSExprNode *name, tTJSExprNode *value) {
    tTJSExprNode *elem = new tTJSExprNode();
    elem->SetOpcode(T_DICELM);
    elem->Add(name);          // Nodes[0] = 键
    elem->Add(value);         // Nodes[1] = 值
    Add(elem);                // 追加到字典容器的 Nodes
}
```

`%["name" => "Alice", "age" => 25]` 生成的 AST：

```
              T_INLINEDIC
             ╱            ╲
       T_DICELM          T_DICELM
       ╱      ╲          ╱      ╲
  T_CONSTVAL T_CONSTVAL T_CONSTVAL T_CONSTVAL
  Val="name" Val="Alice" Val="age"  Val=25
```

### 5.4 嵌套支持——栈的真正价值

`CurrentNodeVector` 是栈（而非单个指针），是因为 TJS2 允许容器嵌套。例如：

```javascript
var data = [1, [2, 3], %["k" => [4, 5]]];
```

解析过程中栈的变化：

```
时间点                          CurrentNodeVector 栈状态
──────────────────────────────  ─────────────────────────
归约 "["                        [外层数组]
归约 expr(1)                    [外层数组]           → cn=外层数组, AddArrayElement(1)
归约 ","                        [外层数组]
归约内层 "["                    [外层数组, 内层数组]  → 压入内层
归约 expr(2)                    [外层数组, 内层数组]  → cn=内层数组, AddArrayElement(2)
归约 "," expr(3)                [外层数组, 内层数组]  → cn=内层数组, AddArrayElement(3)
归约内层 "]"                    [外层数组]           → 弹出内层，返回内层数组节点
外层 AddArrayElement(内层数组)  [外层数组]           → cn=外层数组
归约 ","                        [外层数组]
归约 "%["                       [外层数组, 字典]     → 压入字典
归约 "k"=>[...]                 [外层数组, 字典, 内层数组2] → 值是嵌套数组
...                             ...
归约最外层 "]"                  []                   → 弹出外层
```

如果没有栈，只用一个全局指针，则内层容器解析时会覆盖外层容器的引用，导致外层容器"丢失"。栈的后进先出（LIFO，Last In First Out——最后压入的元素最先弹出）特性恰好匹配了语法嵌套的进出顺序。

### 5.5 小结——为什么不直接用语法栈

读者可能会问：Bison 自身有一个语法栈（`yyval` 栈），为什么不利用它来传递容器节点，而要额外维护 `CurrentNodeVector`？

原因在于 Bison 语法栈只在产生式归约时提供 `$1`、`$2` 等位置引用。而 `inline_array_elm` 的递归产生式中，容器节点不在任何 `$N` 位置——它在外层产生式 `inline_array` 的中间动作里创建，对内层产生式不可见。要让内层规则访问外层的节点，有两条路：

1. **传参**：修改 Bison 的 `%param` 机制将容器传入——但这会改变所有产生式的签名，非常侵入性
2. **全局/上下文栈**：用一个显式栈存放——简单、非侵入、支持嵌套

TJS2 选择了方案 2，这是一种在手写和自动生成解析器中都常见的惯用模式。

---

## 6. GenNodeCode 入口与 AST→字节码桥接

AST 构建完成后，如何将其转化为字节码？这是第 4 章（字节码编译）的核心内容，但在本节最后，我们先建立整体认知。

### 6.1 表达式语句的处理入口

在 TJS2 的编译流水线中，语句（Statement）和表达式（Expression）走不同的路径：

- **控制流语句**（`if`/`while`/`for`/`switch`/`try`）：在语法动作中**直接调用** `tTJSInterCodeContext` 的 Enter/Exit 方法（如 `EnterWhileCode()`、`ExitWhileCode()`），**不经过 AST**
- **表达式语句**（赋值、函数调用、运算等）：先构建 AST 子树，然后通过 `CreateExprCode()` 桥接到字节码生成

```cpp
// tjs.y 第 520-524 行
expr_stmt
    : expr ";"  { cc->CreateExprCode($1); }    // 表达式语句 → 构建AST → 生成字节码
;
```

### 6.2 CreateExprCode 与 GenNodeCode

`CreateExprCode` 是桥接函数，它将 AST 根节点传给 `GenNodeCode` 进行递归编译：

```cpp
// tjsInterCodeGen.cpp 第 2913-2918 行
void tTJSInterCodeContext::CreateExprCode(tTJSExprNode *node) {
    tjs_int frame = FrameBase;              // 当前寄存器帧基址
    GNC_GenNodeCode(frame, node,            // 递归生成字节码
        TJS_RT_NEEDED, 0, tSubType());
    ClearFrame(frame);                      // 清理临时寄存器
}
```

> **`GNC_GenNodeCode`** 是一个宏，展开为 `GenNodeCode`。之所以用宏是为了在调试构建中插入额外的栈溢出检查。

### 6.3 GenNodeCode 的调度机制

`GenNodeCode` 是一个巨大的 switch-case 函数（约 2500 行），根据节点的操作码分派处理逻辑：

```cpp
// tjsInterCodeGen.cpp 第 1110-1199 行（简化）
tjs_int tTJSInterCodeContext::GenNodeCode(
    tjs_int &frame,          // 寄存器栈指针（引用传递，会被修改）
    tTJSExprNode *node,      // 要编译的 AST 节点
    tjs_uint32 restype,      // 结果类型：TJS_RT_NEEDED（需要值）或 TJS_RT_CFLAG（需要条件标志）
    tjs_int reqresaddr,      // 请求的结果寄存器地址（0 = 自动分配）
    const tSubType &param)   // 子类型参数（用于复合赋值等）
{
    tjs_int resaddr = 0;
    tjs_int op = node->GetOpcode();

    switch(op) {
    case T_CONSTVAL:
        // 常量值 → 生成 VM_CONST 指令，加载到寄存器
        break;
    case T_SYMBOL:
        // 变量引用 → 查找变量位置，生成 VM_CP/VM_CL 等指令
        break;
    case T_PLUS:
        // 加法 → 递归编译左右子节点，生成 VM_ADD 指令
        break;
    case T_LPARENTHESIS:
        // 函数调用 → 编译被调函数和参数列表，生成 VM_CALL 指令
        break;
    case T_DOT:
        // 属性访问 → 编译对象和属性名，生成 VM_GPDS 指令
        break;
    case T_INLINEARRAY:
        // 内联数组 → 遍历 Nodes，逐元素生成代码
        break;
    case T_INLINEDIC:
        // 内联字典 → 遍历 Nodes，逐键值对生成代码
        break;
    // ... 还有约 80 种 case
    }
    return resaddr;
}
```

核心思想是**递归下降编译**（Recursive Descent Compilation）：每个节点的处理逻辑中，会递归调用 `GenNodeCode` 来编译子节点，将子节点的结果放入寄存器，然后生成当前节点对应的 VM 指令，将结果写入另一个寄存器。这个过程从 AST 根节点开始，一直递归到叶节点（常量或变量），形成自底向上的指令发射顺序。

### 6.4 AST 生命周期总结

```
┌──────────────────────────────────────────────────────────────┐
│                    AST 完整生命周期                           │
├──────────────────────────────────────────────────────────────┤
│  1. 创建                                                     │
│     语法动作调用 MakeNP0/1/2/3 → new tTJSExprNode           │
│     节点自动注册到 NodeToDeleteVector                         │
│                                                              │
│  2. 组装                                                     │
│     父节点的 Nodes 数组指向子节点                              │
│     CurrentNodeVector 辅助内联容器构建                         │
│                                                              │
│  3. 优化                                                     │
│     MakeNP1/MakeNP2 中的常量折叠                              │
│     编译期可计算的表达式直接归约为 T_CONSTVAL                   │
│                                                              │
│  4. 编译                                                     │
│     CreateExprCode → GenNodeCode 递归遍历                    │
│     每个节点生成对应的 VM 指令                                 │
│                                                              │
│  5. 销毁                                                     │
│     Commit() 调用 ClearNodesToDelete()                       │
│     遍历 NodeToDeleteVector，delete 所有节点                  │
│     一次性批量释放，无需 AST 遍历                              │
└──────────────────────────────────────────────────────────────┘
```

> 详细的字节码生成过程（`GenNodeCode` 的每种 case 如何发射指令）将在第 4 章"字节码编译"中深入讲解。

---

## 7. 动手实践

### 实验 1：手动构建 AST 并验证常量折叠

编写一个独立的 C++ 程序，模拟 TJS2 的 AST 节点结构和 MakeNP 工厂函数，验证常量折叠的效果。

```cpp
// ast_demo.cpp — 模拟 TJS2 AST 节点与常量折叠
#include <iostream>
#include <vector>
#include <string>
#include <variant>

// 简化的操作码枚举
enum Opcode {
    T_CONSTVAL,     // 常量值
    T_SYMBOL,       // 变量引用
    T_PLUS,         // 加法
    T_MINUS,        // 减法
    T_ASTERISK,     // 乘法
    T_UMINUS,       // 一元取负
    T_INLINEARRAY,  // 内联数组
    T_ARRAYARG,     // 数组元素包装
};

// 操作码→字符串的映射（用于打印）
const char* OpcodeStr(Opcode op) {
    switch(op) {
        case T_CONSTVAL:    return "T_CONSTVAL";
        case T_SYMBOL:      return "T_SYMBOL";
        case T_PLUS:        return "T_PLUS";
        case T_MINUS:       return "T_MINUS";
        case T_ASTERISK:    return "T_ASTERISK";
        case T_UMINUS:      return "T_UMINUS";
        case T_INLINEARRAY: return "T_INLINEARRAY";
        case T_ARRAYARG:    return "T_ARRAYARG";
        default:            return "UNKNOWN";
    }
}

// 简化的值类型（只支持整数和字符串）
using Value = std::variant<int, std::string>;

// 简化的 AST 节点——对标 tTJSExprNode
struct ExprNode {
    Opcode op;                           // 操作码
    Value val;                           // 值（仅 T_CONSTVAL/T_SYMBOL 使用）
    std::vector<ExprNode*> nodes;        // 子节点数组
    
    ExprNode(Opcode o) : op(o), val(0) {}
    
    void Add(ExprNode* child) {
        nodes.push_back(child);          // 追加子节点
    }
    
    // 打印 AST 树（带缩进的递归遍历）
    void Print(int indent = 0) const {
        for (int i = 0; i < indent; i++) std::cout << "  ";
        std::cout << OpcodeStr(op);
        if (op == T_CONSTVAL) {
            if (auto* p = std::get_if<int>(&val))
                std::cout << "(" << *p << ")";
            else if (auto* p = std::get_if<std::string>(&val))
                std::cout << "(\"" << *p << "\")";
        } else if (op == T_SYMBOL) {
            if (auto* p = std::get_if<std::string>(&val))
                std::cout << "(\"" << *p << "\")";
        }
        std::cout << "\n";
        for (auto* child : nodes) {
            if (child) child->Print(indent + 1);
            else {
                for (int i = 0; i < indent + 1; i++) std::cout << "  ";
                std::cout << "(null)\n";
            }
        }
    }
};

// 全局节点池（模拟 NodeToDeleteVector）
std::vector<ExprNode*> g_nodePool;

// 节点工厂——分配节点并注册到池中
ExprNode* AllocNode(Opcode op) {
    auto* node = new ExprNode(op);
    g_nodePool.push_back(node);          // 注册，Commit 时批量释放
    return node;
}

// MakeNP0 — 叶节点
ExprNode* MakeNP0(Opcode op) {
    return AllocNode(op);
}

// MakeConstValNode — 创建整数常量节点
ExprNode* MakeConstValNode(int value) {
    auto* node = AllocNode(T_CONSTVAL);
    node->val = value;
    return node;
}

// MakeSymbolNode — 创建变量引用节点
ExprNode* MakeSymbolNode(const std::string& name) {
    auto* node = AllocNode(T_SYMBOL);
    node->val = name;
    return node;
}

// MakeNP1 — 一元节点（带常量折叠）
ExprNode* MakeNP1(Opcode op, ExprNode* child) {
    // 常量折叠：如果子节点是常量，直接计算结果
    if (child && child->op == T_CONSTVAL) {
        if (auto* p = std::get_if<int>(&child->val)) {
            switch (op) {
                case T_UMINUS: {
                    std::cout << "[常量折叠] -(" << *p << ") => " << -(*p) << "\n";
                    child->val = -(*p);  // 直接修改值
                    return child;        // 返回常量节点本身
                }
                default: break;
            }
        }
    }
    auto* node = AllocNode(op);
    node->Add(child);
    return node;
}

// MakeNP2 — 二元节点（带常量折叠）
ExprNode* MakeNP2(Opcode op, ExprNode* left, ExprNode* right) {
    // 常量折叠：如果两个子节点都是常量，直接计算结果
    if (left && left->op == T_CONSTVAL &&
        right && right->op == T_CONSTVAL) {
        auto* lp = std::get_if<int>(&left->val);
        auto* rp = std::get_if<int>(&right->val);
        if (lp && rp) {
            int result = 0;
            bool folded = true;
            switch (op) {
                case T_PLUS:     result = *lp + *rp; break;
                case T_MINUS:    result = *lp - *rp; break;
                case T_ASTERISK: result = *lp * *rp; break;
                default: folded = false; break;
            }
            if (folded) {
                std::cout << "[常量折叠] " << *lp << " " 
                          << OpcodeStr(op) << " " << *rp 
                          << " => " << result << "\n";
                left->val = result;
                return left;             // 复用左子节点
            }
        }
    }
    auto* node = AllocNode(op);
    node->Add(left);
    node->Add(right);
    return node;
}

// 释放所有节点（模拟 ClearNodesToDelete）
void ClearAllNodes() {
    for (auto* node : g_nodePool) delete node;
    g_nodePool.clear();
    std::cout << "[内存] 所有 AST 节点已释放\n";
}

int main() {
    std::cout << "=== 实验 1: 表达式 1 + 2 * 3 ===\n\n";
    
    // 模拟解析 "1 + 2 * 3"
    // 按照运算优先级，先构建 2*3，再构建 1+(2*3)
    auto* two   = MakeConstValNode(2);     // 常量 2
    auto* three = MakeConstValNode(3);     // 常量 3
    auto* mul   = MakeNP2(T_ASTERISK, two, three);  // 2 * 3 → 折叠为 6
    
    auto* one   = MakeConstValNode(1);     // 常量 1
    auto* add   = MakeNP2(T_PLUS, one, mul);        // 1 + 6 → 折叠为 7
    
    std::cout << "\n最终 AST:\n";
    add->Print();
    
    std::cout << "\n=== 实验 2: 表达式 a + 2 * 3（部分折叠）===\n\n";
    
    auto* a2    = MakeConstValNode(2);
    auto* a3    = MakeConstValNode(3);
    auto* amul  = MakeNP2(T_ASTERISK, a2, a3);  // 2 * 3 → 折叠为 6
    
    auto* aVar  = MakeSymbolNode("a");     // 变量 a — 不是常量
    auto* aadd  = MakeNP2(T_PLUS, aVar, amul);  // a + 6 — 无法折叠（a 不是常量）
    
    std::cout << "\n最终 AST:\n";
    aadd->Print();
    
    std::cout << "\n=== 实验 3: 常量折叠级联 -(2 + 3) ===\n\n";
    
    auto* b2    = MakeConstValNode(2);
    auto* b3    = MakeConstValNode(3);
    auto* badd  = MakeNP2(T_PLUS, b2, b3);      // 2 + 3 → 折叠为 5
    auto* bneg  = MakeNP1(T_UMINUS, badd);       // -(5)  → 折叠为 -5
    
    std::cout << "\n最终 AST:\n";
    bneg->Print();
    
    // 批量释放
    std::cout << "\n";
    ClearAllNodes();
    
    return 0;
}
```

**编译与运行**：

```bash
# Linux / macOS
g++ -std=c++17 -o ast_demo ast_demo.cpp && ./ast_demo

# Windows (MSVC)
cl /std:c++17 /EHsc ast_demo.cpp && ast_demo.exe
```

**预期输出**：

```
=== 实验 1: 表达式 1 + 2 * 3 ===

[常量折叠] 2 T_ASTERISK 3 => 6
[常量折叠] 1 T_PLUS 6 => 7

最终 AST:
T_CONSTVAL(7)

=== 实验 2: 表达式 a + 2 * 3（部分折叠）===

[常量折叠] 2 T_ASTERISK 3 => 6

最终 AST:
T_PLUS
  T_SYMBOL("a")
  T_CONSTVAL(6)

=== 实验 3: 常量折叠级联 -(2 + 3) ===

[常量折叠] 2 T_PLUS 3 => 5
[常量折叠] -(5) => -5

最终 AST:
T_CONSTVAL(-5)

[内存] 所有 AST 节点已释放
```

**观察要点**：
1. 实验 1 中 `1 + 2 * 3` 被完全折叠为单个常量节点 `T_CONSTVAL(7)`
2. 实验 2 中只有 `2 * 3` 被折叠，`a + 6` 因为 `a` 是变量而保留为二元节点
3. 实验 3 演示了级联折叠：加法先折叠为 5，然后一元取负再折叠为 -5

### 实验 2：模拟 CurrentNodeVector 构建嵌套数组

在实验 1 的基础上，添加内联数组支持，验证 CurrentNodeVector 栈在嵌套场景下的正确性。

```cpp
// array_demo.cpp — 模拟 CurrentNodeVector 构建嵌套数组
#include <iostream>
#include <vector>
#include <string>
#include <variant>
#include <cassert>

enum Opcode { T_CONSTVAL, T_INLINEARRAY, T_ARRAYARG };

using Value = std::variant<int, std::string>;

struct ExprNode {
    Opcode op;
    Value val;
    std::vector<ExprNode*> nodes;
    
    ExprNode(Opcode o) : op(o), val(0) {}
    
    void Add(ExprNode* child) { nodes.push_back(child); }
    
    // 模拟 AddArrayElement
    void AddArrayElement(ExprNode* elem) {
        auto* wrapper = new ExprNode(T_ARRAYARG);   // 包装为 T_ARRAYARG
        wrapper->Add(elem);
        Add(wrapper);
    }
    
    void Print(int indent = 0) const {
        for (int i = 0; i < indent; i++) std::cout << "  ";
        if (op == T_CONSTVAL) {
            if (auto* p = std::get_if<int>(&val))
                std::cout << "T_CONSTVAL(" << *p << ")\n";
        } else if (op == T_INLINEARRAY) {
            std::cout << "T_INLINEARRAY [" << nodes.size() << " 个元素]\n";
        } else if (op == T_ARRAYARG) {
            std::cout << "T_ARRAYARG\n";
        }
        for (auto* child : nodes)
            if (child) child->Print(indent + 1);
    }
};

// 模拟 CurrentNodeVector
std::vector<ExprNode*> CurrentNodeVector;

void PushCurrentNode(ExprNode* node) {
    CurrentNodeVector.push_back(node);
    std::cout << "[栈] Push → 深度 " << CurrentNodeVector.size() << "\n";
}

ExprNode* GetCurrentNode() {
    if (CurrentNodeVector.empty()) return nullptr;
    return CurrentNodeVector.back();
}

void PopCurrentNode() {
    CurrentNodeVector.pop_back();
    std::cout << "[栈] Pop  → 深度 " << CurrentNodeVector.size() << "\n";
}

int main() {
    std::cout << "=== 构建嵌套数组 [1, [2, 3], 4] ===\n\n";
    
    // 外层数组 "["
    auto* outer = new ExprNode(T_INLINEARRAY);
    PushCurrentNode(outer);
    
    // 第一个元素: 1
    auto* val1 = new ExprNode(T_CONSTVAL); val1->val = 1;
    GetCurrentNode()->AddArrayElement(val1);
    std::cout << "  添加元素 1 到外层\n";
    
    // 第二个元素: [2, 3] — 嵌套数组
    auto* inner = new ExprNode(T_INLINEARRAY);
    PushCurrentNode(inner);                       // 压入内层
    
    auto* val2 = new ExprNode(T_CONSTVAL); val2->val = 2;
    GetCurrentNode()->AddArrayElement(val2);
    std::cout << "  添加元素 2 到内层\n";
    
    auto* val3 = new ExprNode(T_CONSTVAL); val3->val = 3;
    GetCurrentNode()->AddArrayElement(val3);
    std::cout << "  添加元素 3 到内层\n";
    
    PopCurrentNode();                             // 弹出内层
    GetCurrentNode()->AddArrayElement(inner);     // 将内层数组添加到外层
    std::cout << "  添加内层数组到外层\n";
    
    // 第三个元素: 4
    auto* val4 = new ExprNode(T_CONSTVAL); val4->val = 4;
    GetCurrentNode()->AddArrayElement(val4);
    std::cout << "  添加元素 4 到外层\n";
    
    PopCurrentNode();                             // 弹出外层
    
    std::cout << "\n最终 AST:\n";
    outer->Print();
    
    // 验证结构
    assert(outer->nodes.size() == 3);             // 3 个元素
    assert(CurrentNodeVector.empty());            // 栈已清空
    std::cout << "\n[验证] 所有断言通过 ✓\n";
    
    // 清理（简化，实际应用 NodeToDeleteVector 批量释放）
    delete val1; delete val2; delete val3; delete val4;
    delete inner; delete outer;
    
    return 0;
}
```

**编译与运行**：

```bash
g++ -std=c++17 -o array_demo array_demo.cpp && ./array_demo
```

**预期输出**：

```
=== 构建嵌套数组 [1, [2, 3], 4] ===

[栈] Push → 深度 1
  添加元素 1 到外层
[栈] Push → 深度 2
  添加元素 2 到内层
  添加元素 3 到内层
[栈] Pop  → 深度 1
  添加内层数组到外层
  添加元素 4 到外层
[栈] Pop  → 深度 0

最终 AST:
T_INLINEARRAY [3 个元素]
  T_ARRAYARG
    T_CONSTVAL(1)
  T_ARRAYARG
    T_INLINEARRAY [2 个元素]
      T_ARRAYARG
        T_CONSTVAL(2)
      T_ARRAYARG
        T_CONSTVAL(3)
  T_ARRAYARG
    T_CONSTVAL(4)

[验证] 所有断言通过 ✓
```

### 实验 3：手画 AST 练习

不运行代码，纯手工在纸上画出以下 TJS2 表达式的 AST 树形结构。完成后对照下方答案。

**表达式 A**：`obj.name = "hello"`

提示：赋值 (`T_EQUAL`) 是根节点，左侧是属性访问 (`T_DOT`)，右侧是字符串常量。属性名是 `T_CONSTVAL` 而非 `T_SYMBOL`（参考 §4.4）。

<details>
<summary>查看答案</summary>

```
            T_EQUAL
           ╱       ╲
      T_DOT        T_CONSTVAL
     ╱     ╲       Val="hello"
T_SYMBOL  T_CONSTVAL
Val="obj" Val="name"
```

</details>

**表达式 B**：`arr[i] = arr[i] + 1`

提示：`arr[i]` 用 `T_LBRACKET` 表示（方括号索引访问），等号两侧的 `arr[i]` 在 AST 中是两棵**独立的子树**（不共享节点）。

<details>
<summary>查看答案</summary>

```
                   T_EQUAL
                  ╱        ╲
          T_LBRACKET       T_PLUS
          ╱       ╲       ╱       ╲
     T_SYMBOL  T_SYMBOL T_LBRACKET T_CONSTVAL
     Val="arr" Val="i"  ╱       ╲   Val=1
                   T_SYMBOL  T_SYMBOL
                   Val="arr" Val="i"
```

</details>

**表达式 C**：`new Sprite(100, 200) if visible`

提示：后缀 `if` 的 `T_IF` 是最外层节点（§4.6），`new` 复用函数调用结构但操作码改为 `T_NEW`（§4.8），参数列表用 `T_ARG` 链表（§4.3）。

<details>
<summary>查看答案</summary>

```
                 T_IF
                ╱    ╲
           T_NEW    T_SYMBOL
          ╱     ╲    Val="visible"
    T_SYMBOL   T_ARG
    Val="Sprite" ╱    ╲
           T_CONSTVAL T_ARG
           Val=200    ╱   ╲
                 T_CONSTVAL (null)
                 Val=100
```

注意参数列表是**反向链表**：根 `T_ARG` 的 `[0]` 是最后一个参数 200，`[1]` 指向前一个 `T_ARG`，其 `[0]` 是第一个参数 100。

</details>

---

## 8. 对照项目源码

本节涉及的所有代码均来自 KrKr2 项目的 TJS2 引擎。以下是关键文件和行号的完整索引：

### 核心文件

| 文件 | 行号范围 | 内容 |
|------|----------|------|
| `cpp/core/tjs2/tjsInterCodeGen.h` | 181-240 行 | `tTJSExprNode` 类定义：Op/Position/Nodes/Val 四字段、Add()/AddArrayElement()/AddDictionaryElement() 方法 |
| `cpp/core/tjs2/tjsInterCodeGen.h` | 245-872 行 | `tTJSInterCodeContext` 类定义：MakeNP0/1/2/3 声明、CurrentNodeVector/NodeToDeleteVector 成员 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 93-133 行 | `tTJSExprNode` 的构造函数和 `Add()` 方法实现 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 416-424 行 | `ClearNodesToDelete()`——遍历 NodeToDeleteVector 执行 delete |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 952-998 行 | `Commit()`——编译上下文的终结函数，调用 ClearNodesToDelete |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 1110-1199 行 | `GenNodeCode()` 入口——AST 节点到字节码的总调度 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 2913-2918 行 | `CreateExprCode()`——表达式语句的编译桥接函数 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 3768-3782 行 | `PushCurrentNode()`/`GetCurrentNode()`/`PopCurrentNode()` |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 3787-3793 行 | `MakeConstValNode()`——创建常量节点的辅助函数 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 3800-3850 行 | `MakeNP0()`——叶节点工厂 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 3852-3920 行 | `MakeNP1()`——一元节点工厂，含 10 种运算的常量折叠 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 3922-3980 行 | `MakeNP2()`——二元节点工厂，含 21 种运算的常量折叠 |
| `cpp/core/tjs2/tjsInterCodeGen.cpp` | 3982-3997 行 | `MakeNP3()`——三元节点工厂（无常量折叠） |

### 语法文件

| 文件 | 行号范围 | 内容 |
|------|----------|------|
| `cpp/core/tjs2/bison/tjs.y` | 1-30 行 | `%code` 段：`cc`/`lx` 宏定义、`cn` 宏、yylex 桥接 |
| `cpp/core/tjs2/bison/tjs.y` | 520-535 行 | `expr_stmt`/`expr` 产生式：`CreateExprCode` 调用和后缀 `if` |
| `cpp/core/tjs2/bison/tjs.y` | 640-720 行 | 运算符优先级表达式：一元/二元运算、属性访问、函数调用 |
| `cpp/core/tjs2/bison/tjs.y` | 648 行 | `new` 运算符——复用函数调用后改写 opcode |
| `cpp/core/tjs2/bison/tjs.y` | 681-684 行 | DOT 属性访问——`SetNextIsBareWord()` 词法分析器交互 |
| `cpp/core/tjs2/bison/tjs.y` | 685-687 行 | 后缀运算符：`++`/`--`/`!` |
| `cpp/core/tjs2/bison/tjs.y` | 720-735 行 | 函数调用与参数列表（T_ARG 链表） |
| `cpp/core/tjs2/bison/tjs.y` | 740-785 行 | 内联数组与内联字典（CurrentNodeVector 使用） |

### 阅读建议

1. **入门路径**：先读 `tTJSExprNode` 类定义（h 文件 181-240 行），理解节点的四个核心字段
2. **工厂函数**：再读 MakeNP0/1/2/3（cpp 文件 3800-3997 行），理解节点创建和常量折叠
3. **语法驱动**：然后读 `tjs.y` 的表达式产生式（640-785 行），看语法规则如何调用工厂函数
4. **桥接**：最后读 `CreateExprCode`（cpp 文件 2913-2918 行）和 `GenNodeCode` 入口（1110-1199 行）

---

## 9. 本节小结

- **tTJSExprNode 是统一的 AST 节点类**，用四个字段（Op 操作码、Position 源码位置、Nodes 子节点数组、Val 值）表示所有表达式结构。没有继承层次，所有节点都是同一个类的实例
- **MakeNP0/1/2/3 是节点工厂函数**，分别创建 0/1/2/3 个子节点的 AST 节点。MakeNP1 对 10 种一元运算做常量折叠，MakeNP2 对 21 种二元运算做常量折叠。常量折叠是编译期优化（Compile-time Optimization），将常量表达式在编译阶段直接计算出结果
- **NodeToDeleteVector 实现 Arena 式内存管理**。所有 AST 节点在创建时注册到这个向量中，在 `Commit()` 时通过 `ClearNodesToDelete()` 一次性批量释放，无需遍历 AST 树结构
- **CurrentNodeVector 是一个节点栈**，用于在语法归约过程中临时持有"当前正在构建的容器节点"（内联数组或内联字典）。栈的后进先出特性天然支持容器嵌套
- **控制流语句和表达式语句走不同路径**。控制流（if/while/for 等）在语法动作中直接调用 Enter/Exit 方法，不经过 AST；表达式语句先构建 AST 子树，再通过 `CreateExprCode()` → `GenNodeCode()` 桥接到字节码生成
- **参数列表用 T_ARG 构成反向链表**，根节点指向最后一个参数，末尾节点指向第一个参数
- **DOT 属性访问会与词法分析器交互**（`SetNextIsBareWord()`），确保属性名即使是关键字也按标识符处理
- **new 运算符复用函数调用的 AST 结构**，只是将根节点操作码从 `T_LPARENTHESIS` 改写为 `T_NEW`

---

## 10. 练习题与答案

### 题目 1：常量折叠的边界

TJS2 的 `MakeNP2` 对 21 种二元运算做常量折叠。请回答以下问题：

1. 表达式 `"hello" + " world"` 会被常量折叠吗？为什么？
2. 表达式 `10 / 0` 在常量折叠阶段会发生什么？
3. 如果要让 `MakeNP3`（三元运算符 `? :`）也支持常量折叠，应该怎么做？需要考虑什么边界条件？

<details>
<summary>查看答案</summary>

**1. `"hello" + " world"` 的常量折叠**

会被折叠。TJS2 的 `MakeNP2` 中对 `T_PLUS` 的常量折叠逻辑使用 `tTJSVariant` 的运算符重载。`tTJSVariant` 支持字符串类型，字符串加法定义为拼接。所以当两个操作数都是字符串常量时，折叠结果为 `"hello world"`。

```cpp
// tjsInterCodeGen.cpp 中常量折叠的核心代码（简化）
case T_PLUS:
    *val1 += *val2;   // tTJSVariant::operator+=
    // 对字符串类型，+= 执行字符串拼接
    break;
```

**2. `10 / 0` 的常量折叠**

常量折叠阶段会调用 `tTJSVariant::operator/=()`。TJS2 的除法对整数除以零的处理是：

- 整数除零：不抛异常，而是返回 0（TJS2 的设计选择）
- 浮点除零：返回 `Infinity` 或 `NaN`

所以 `10 / 0` 会被折叠为常量 `0`，不会产生运行时错误。这与 JavaScript 的行为不同（JavaScript 整数除零返回 `Infinity`）。

**3. MakeNP3 的常量折叠**

理论上可以实现：当条件节点是常量时，直接选择真值分支或假值分支的节点返回。

```cpp
ExprNode* MakeNP3(Opcode op, ExprNode* cond, ExprNode* t, ExprNode* f) {
    if (cond && cond->op == T_CONSTVAL) {
        // 条件是常量 → 直接选择分支
        bool condValue = IsTrue(cond->val);   // 转换为布尔值
        return condValue ? t : f;              // 返回对应分支
    }
    // 否则正常构建三元节点
    auto* node = AllocNode(op);
    node->Add(cond); node->Add(t); node->Add(f);
    return node;
}
```

边界条件：
- 未选中的分支可能包含副作用（如函数调用）。编译期删除该分支是安全的，因为在源码语义上条件为常量时，该分支本就不会执行
- 条件的"真假"判定需要与 TJS2 运行时的 truthiness 规则一致（0、空字符串、void 为假）
- TJS2 实际上**没有实现**三元运算符的常量折叠，可能是因为条件为常量的场景极少，优化收益不大

</details>

### 题目 2：AST 节点生命周期

阅读以下代码，回答问题：

```tjs
function calc(x) {
    var a = x + 1;
    var b = a * 2;
    return a + b;
}
```

1. 这段代码中一共会创建多少个 AST 节点（假设没有常量折叠）？列出每条表达式语句创建的节点。
2. 这些节点何时被释放？是逐条语句释放还是一次性释放？
3. `function calc(x) { ... }` 整体是控制流语句还是表达式语句？它走 AST 路径还是 Enter/Exit 路径？

<details>
<summary>查看答案</summary>

**1. AST 节点数量**

每条**表达式语句**创建的 AST 节点：

- `var a = x + 1;`：
  - `T_SYMBOL("a")`（左值）、`T_SYMBOL("x")`、`T_CONSTVAL(1)`、`T_PLUS`（加法）、`T_EQUAL`（赋值）、`T_VAR`（变量声明）
  - **6 个节点**

- `var b = a * 2;`：
  - `T_SYMBOL("b")`、`T_SYMBOL("a")`、`T_CONSTVAL(2)`、`T_ASTERISK`、`T_EQUAL`、`T_VAR`
  - **6 个节点**

- `return a + b;`：
  - `return` 是控制流关键字，直接调用 `cc->EnterReturnCode()`，但 `a + b` 仍构建 AST
  - `T_SYMBOL("a")`、`T_SYMBOL("b")`、`T_PLUS`
  - **3 个节点**

总计约 **15 个** AST 节点。

注意：实际实现中，`var` 声明可能通过特殊语法处理（`cc->CreateExprCode()` 或 `cc->AddLocalVariable()`），节点数量可能有所不同。此处是概念性分析。

**2. 节点释放时机**

所有节点在 `Commit()` 时通过 `ClearNodesToDelete()` **一次性批量释放**。`Commit()` 在函数体编译结束时调用。

不是逐条语句释放，因为 `ClearNodesToDelete()` 只在 `Commit()` 中调用，而 `Commit()` 是在一个完整的函数/全局上下文编译结束时才执行。

**3. function 语句的路径**

`function calc(x) { ... }` 走 **Enter/Exit 路径**：

```cpp
// tjs.y 中的 function 产生式（简化）
"function" T_SYMBOL "(" ... ")" { cc->EnterFuncBody(); }
  block
{ cc->ExitFuncBody(); }
```

`EnterFuncBody()` 创建新的 `tTJSInterCodeContext`（编译上下文），函数体在新上下文中编译。`ExitFuncBody()` 调用 `Commit()` 完成编译并释放该上下文中的所有 AST 节点。

函数定义本身不是表达式语句，但函数体内的每条表达式语句会各自构建 AST 子树。

</details>

### 题目 3：设计一个 AST 节点可视化工具

假设你需要编写一个调试工具，在 TJS2 编译过程中将 AST 以可读格式打印出来。请回答：

1. 你会在 `CreateExprCode()` 的哪个位置插入打印代码？为什么？
2. 编写一个递归函数 `void PrintAST(tTJSExprNode* node, int depth)`，用缩进表示层级，打印每个节点的操作码和值。给出完整的 C++ 代码。
3. 如何确保打印不影响正常编译流程？

<details>
<summary>查看答案</summary>

**1. 插入位置**

在 `CreateExprCode()` 中，`GenNodeCode()` 调用**之前**插入打印代码：

```cpp
void tTJSInterCodeContext::CreateExprCode(tTJSExprNode *node) {
    // ★ 在这里插入 AST 打印
    #ifdef TJS_DEBUG_AST
    PrintAST(node, 0);
    #endif
    
    tjs_int frame = FrameBase;
    GNC_GenNodeCode(frame, node, TJS_RT_NEEDED, 0, tSubType());
    ClearFrame(frame);
}
```

在 `GenNodeCode` 之前打印，因为：
- 此时 AST 已完全构建，包括常量折叠已完成
- `GenNodeCode` 不会修改 AST 结构（只读遍历），但在之后节点关系更难追踪
- 如果 `GenNodeCode` 中出现错误，至少打印已经完成，可以辅助调试

**2. PrintAST 完整代码**

```cpp
#include <cstdio>
#include "tjsInterCodeGen.h"   // tTJSExprNode 定义
#include "tjsLex.h"            // Token 枚举

// Token 操作码→可读字符串
static const char* TokenToStr(tjs_int op) {
    switch(op) {
        case token::T_CONSTVAL:       return "T_CONSTVAL";
        case token::T_SYMBOL:         return "T_SYMBOL";
        case token::T_PLUS:           return "T_PLUS";
        case token::T_MINUS:          return "T_MINUS";
        case token::T_ASTERISK:       return "T_ASTERISK";
        case token::T_SLASH:          return "T_SLASH";
        case token::T_EQUAL:          return "T_EQUAL";
        case token::T_DOT:            return "T_DOT";
        case token::T_LPARENTHESIS:   return "T_LPARENTHESIS";
        case token::T_NEW:            return "T_NEW";
        case token::T_IF:             return "T_IF";
        case token::T_ARG:            return "T_ARG";
        case token::T_INLINEARRAY:    return "T_INLINEARRAY";
        case token::T_INLINEDIC:      return "T_INLINEDIC";
        case token::T_ARRAYARG:       return "T_ARRAYARG";
        case token::T_DICELM:         return "T_DICELM";
        case token::T_QUESTION:       return "T_QUESTION";
        case token::T_LBRACKET:       return "T_LBRACKET";
        case token::T_POSTINCREMENT:  return "T_POSTINCREMENT";
        case token::T_EVAL:           return "T_EVAL";
        default: {
            static char buf[32];
            snprintf(buf, sizeof(buf), "OP_%d", op);
            return buf;
        }
    }
}

void PrintAST(tTJSExprNode* node, int depth) {
    if (!node) {
        // 空节点也要打印，便于理解结构
        for (int i = 0; i < depth; i++) printf("  ");
        printf("(null)\n");
        return;
    }
    
    // 打印缩进
    for (int i = 0; i < depth; i++) printf("  ");
    
    // 打印操作码
    printf("%s", TokenToStr(node->GetOpcode()));
    
    // 如果是常量或变量，打印值
    if (node->GetOpcode() == token::T_CONSTVAL ||
        node->GetOpcode() == token::T_SYMBOL) {
        const tTJSVariant& val = node->GetValue();
        tjs_int type = val.Type();
        if (type == tvtInteger) {
            printf("(%lld)", (long long)val.AsInteger());
        } else if (type == tvtReal) {
            printf("(%.6f)", val.AsReal());
        } else if (type == tvtString) {
            // 字符串值（TJS2 使用宽字符）
            tTJSString str(val);
            printf("(\"%ls\")", str.c_str());
        }
    }
    
    printf("\n");
    
    // 递归打印子节点
    if (node->GetNodes()) {
        for (tjs_int i = 0; i < node->GetNodes()->GetCount(); i++) {
            PrintAST(node->GetNodes()->At(i), depth + 1);
        }
    }
}
```

**3. 确保不影响正常编译**

- 使用 `#ifdef TJS_DEBUG_AST` 条件编译，Release 构建中完全不包含打印代码
- 打印函数只**读取** AST 节点，不修改任何字段
- 使用 `printf` 输出到 stderr 而非 stdout，避免干扰正常的脚本输出：将 `printf` 替换为 `fprintf(stderr, ...)`
- 在 CMakeLists.txt 中仅在 Debug 构建启用：

```cmake
target_compile_definitions(tjs2 PRIVATE
    $<$<CONFIG:Debug>:TJS_DEBUG_AST>
)
```

</details>

---

## 下一步

下一节 [03-错误恢复与预处理](./03-错误恢复与预处理.md) 将讲解 TJS2 语法分析器的错误恢复机制（Bison 的 `error` 令牌和 `def_list error ";"` 规则）以及 TJS2 预处理器（`tjspp.y` 中的 `@if`/`@set`/`@endif` 指令）。

