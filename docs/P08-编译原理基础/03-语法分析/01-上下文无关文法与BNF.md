# 上下文无关文法与 BNF

> **所属模块：** P08-编译原理基础
> **前置知识：** [手写词法分析器](../02-词法分析/02-手写词法分析器.md)、[编译器五阶段概览](../01-编译器总览/01-从源码到执行.md)
> **预计阅读时间：** 30 分钟
> **关键术语：** 上下文无关文法（Context-Free Grammar, CFG）、巴科斯-诺尔范式（Backus-Naur Form, BNF）、扩展 BNF（EBNF）、终结符（Terminal）、非终结符（Non-terminal）、产生式（Production）、推导（Derivation）、语法分析树（Parse Tree）

## 本节目标

读完本节后，你将能够：
1. 用形式化定义写出一个上下文无关文法的四元组 `G = (V, Σ, R, S)`
2. 区分终结符与非终结符，解释它们与词法器的关系
3. 将自然语言描述的语法规则转换为 BNF 和 EBNF 表示
4. 对给定的输入串进行最左推导和最右推导
5. 画出语法分析树，识别文法的二义性问题

## 从正则到上下文无关

上一章我们学习了正则语言——用正则表达式和有限自动机描述的语言。正则语言能处理标识符、数字、运算符等 Token 的识别，但它有一个根本性的限制：

> **正则语言无法描述"配对嵌套"结构。**

例如，匹配配对的括号 `((()))` 需要"记住"已经看到了多少个左括号。有限自动机只有固定数量的状态，无法计数到任意深度。用泵引理（Pumping Lemma）可以严格证明 `{aⁿbⁿ | n ≥ 0}` 不是正则语言。

但编程语言处处是嵌套结构：

```
if (a > (b + c)) {         // 括号嵌套
    while (x) {             // 语句块嵌套
        f(g(h(x)));         // 函数调用嵌套
    }
}
```

为了描述这类语言，我们需要更强的形式化工具——**上下文无关文法**（Context-Free Grammar, CFG）。

### 乔姆斯基层次

语言学家 Noam Chomsky 在 1956 年提出了形式语言的四级层次：

```
Type 0: 无限制文法（递归可枚举语言）  ← 图灵机
  ⊃ Type 1: 上下文相关文法             ← 线性有界自动机
    ⊃ Type 2: 上下文无关文法 (CFG)     ← 下推自动机 ← 编程语言语法！
      ⊃ Type 3: 正则文法              ← 有限自动机 ← 词法分析！
```

编程语言的语法（Syntax）几乎完全落在 Type 2 层，这就是为什么语法分析器基于 CFG 理论。词法分析用 Type 3（正则），语法分析用 Type 2（上下文无关），两者恰好对应编译器前端的两个阶段。

## 上下文无关文法的形式定义

一个上下文无关文法是一个四元组 **G = (V, Σ, R, S)**：

| 符号 | 名称 | 含义 | 编译器中的对应 |
|------|------|------|----------------|
| V | 非终结符集合 | 语法变量，代表语言的抽象结构 | `Expr`、`Stmt`、`FuncDecl` 等语法规则名 |
| Σ | 终结符集合 | 不可再分解的原子符号 | 词法器产出的 Token：`T_IF`、`T_PLUS`、`T_NUMBER` |
| R | 产生式集合 | 形如 A → α 的替换规则 | BNF 中的每条规则 |
| S | 起始符号 | S ∈ V，推导的起点 | 通常是 `Program` 或 `TranslationUnit` |

**关键约束：** 每条产生式的左侧必须是**单个非终结符**。即 `A → α`，A 是一个非终结符，α 是终结符和非终结符的任意串（含空串 ε）。这就是"上下文无关"的含义——A 无论出现在什么"上下文"中，都可以被替换为 α。

### 示例：简单算术表达式

让我们定义一个支持加法和乘法的表达式文法：

```
G = (V, Σ, R, S) 其中：
  V = { Expr, Term, Factor }
  Σ = { number, +, *, (, ) }
  S = Expr
  R = {
    Expr   → Expr + Term | Term
    Term   → Term * Factor | Factor
    Factor → ( Expr ) | number
  }
```

这个文法能生成 `1 + 2 * 3`、`(1 + 2) * 3`、`42` 等算术表达式。注意 `|` 表示"或"——`Expr → Expr + Term | Term` 是两条产生式的简写：`Expr → Expr + Term` 和 `Expr → Term`。

> **常见错误：** 初学者常把 `+`、`*` 当成非终结符。记住：**终结符就是 Token**——词法器输出什么，语法文法中就用什么作为终结符。`+` 对应 `T_PLUS`，`number` 对应 `T_NUMBER`。

## 推导与语法分析树

### 推导过程

**推导（Derivation）** 是从起始符号 S 出发，反复应用产生式替换非终结符，直到得到纯终结符串的过程。

以输入 `1 + 2 * 3` 为例，进行**最左推导**（每步替换最左边的非终结符）：

```
Expr
⇒ Expr + Term                    [Expr → Expr + Term]
⇒ Term + Term                    [Expr → Term]
⇒ Factor + Term                  [Term → Factor]
⇒ number + Term                  [Factor → number]（number=1）
⇒ number + Term * Factor         [Term → Term * Factor]
⇒ number + Factor * Factor       [Term → Factor]
⇒ number + number * Factor       [Factor → number]（number=2）
⇒ number + number * number       [Factor → number]（number=3）
```

共 8 步推导，得到 `1 + 2 * 3`。

**最右推导**（每步替换最右边的非终结符）也能得到相同结果，但从右侧开始替换：`Expr ⇒ Expr + Term ⇒ Expr + Term * Factor ⇒ Expr + Term * number(3) ⇒ ... ⇒ number(1) + number(2) * number(3)`。最左推导对应自顶向下解析（递归下降），最右推导的逆过程对应自底向上解析（LR/LALR）。TJS2 的 Bison 生成的解析器使用的就是**最右推导的逆过程**——在下一节会详细讨论。

### 语法分析树

推导过程可以用**语法分析树（Parse Tree）** 可视化。上面的 `1 + 2 * 3` 对应：

```
           Expr
          / | \
       Expr + Term
        |    / | \
      Term  Term * Factor
        |     |      |
     Factor Factor  number(3)
        |     |
    number(1) number(2)
```

从根到叶的路径反映了推导步骤；叶节点从左到右读出就是输入串。

**关键观察：** `*` 比 `+` 更靠近叶节点（更先被归约），这意味着乘法的优先级高于加法。文法的**层次结构**天然编码了运算符优先级——不需要额外机制。

## 二义性文法

如果同一个输入串对应**两棵不同的语法分析树**，文法就是**二义的（Ambiguous）**。

### 经典例子：悬挂 else

考虑文法 `Stmt → if Expr then Stmt | if Expr then Stmt else Stmt | other`。对于输入 `if E1 then if E2 then S1 else S2`，`else` 可以配近（属于内层 `if E2`）或配远（属于外层 `if E1`）。几乎所有语言（C/C++/Java/TJS2）都选择**配近**——`else` 与最近的未匹配 `if` 配对。

### 消除二义性的方法

**方法 1：改写文法**——将非终结符拆分为"匹配的"和"未匹配的"两类：

```
Stmt        → MatchedStmt | UnmatchedStmt
MatchedStmt → if Expr then MatchedStmt else MatchedStmt | other
UnmatchedStmt → if Expr then Stmt
              | if Expr then MatchedStmt else UnmatchedStmt
```

**方法 2：在解析器中设置优先级/结合性规则**——Bison/Yacc 的 `%left`、`%right`、`%nonassoc` 声明就是这个机制。TJS2 的 `tjs.y` 中大量使用：

```yacc
// tjs.y 中的优先级声明（摘录）
%left  T_COMMA
%right T_EQUAL T_AMPERSANDEQUAL T_VERTLINEEQUAL ...
%left  T_LOGICALOR
%left  T_LOGICALAND
%left  T_VERTLINE
%left  T_CHEVRON
%left  T_AMPERSAND
%left  T_NOTEQUAL T_EQUALEQUAL T_DISCNOTEQUAL T_DISCEQUAL
%left  T_LT T_GT T_LTOREQUAL T_GTOREQUAL
```

优先级从低到高排列——`T_COMMA` 优先级最低，`T_LT`/`T_GT` 优先级较高。

## BNF：巴科斯-诺尔范式

BNF（Backus-Naur Form）是 1960 年由 John Backus 和 Peter Naur 为 ALGOL 60 语言设计的语法描述格式。它是 CFG 产生式的标准文本表示法。

### BNF 语法

```
<非终结符> ::= 替换体 | 替换体 | ...
```

- `<...>` 包围的是非终结符
- `::=` 读作"定义为"
- `|` 分隔多个替换选项
- 不在 `<>` 中的符号是终结符

### 示例：BNF 描述算术表达式

```bnf
<expr>   ::= <expr> "+" <term> | <term>
<term>   ::= <term> "*" <factor> | <factor>
<factor> ::= "(" <expr> ")" | <number>
<number> ::= <digit> | <number> <digit>
<digit>  ::= "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9"
```

这个 BNF 与前面的 CFG 四元组完全等价——只是表示形式不同。

### BNF 的局限

纯 BNF 缺少三个常用的表达能力：
1. **可选项**（出现 0 或 1 次）——只能用两条产生式：`<A> ::= <B> <C> | <C>`
2. **重复**（出现 0 次或多次）——只能用递归：`<list> ::= <item> | <list> <item>`
3. **分组**——没有括号来组织子表达式

这导致复杂文法的 BNF 描述非常冗长。于是有了扩展版本。

## EBNF：扩展巴科斯-诺尔范式

EBNF（Extended BNF）在 BNF 基础上增加了三个元操作符：

| 符号 | 含义 | BNF 等价形式 |
|------|------|-------------|
| `[...]` 或 `?` | 可选（0 或 1 次） | `A → B C \| C` |
| `{...}` 或 `*` | 重复（0 次或多次） | `A → ε \| A B`（左递归） |
| `(...)` | 分组 | 展开为多条产生式 |

### 示例：EBNF 描述算术表达式

```ebnf
expr   = term { ("+" | "-") term } ;
term   = factor { ("*" | "/") factor } ;
factor = "(" expr ")" | number ;
number = digit { digit } ;
digit  = "0" | "1" | ... | "9" ;
```

对比 BNF 版本：EBNF 用 `{ }` 消除了递归，用 `( )` 分组了 `+`/`-` 和 `*`/`/`，行数从 5 行减少到 5 行（但每行更紧凑，且消除了左递归）。

### EBNF 变体

不幸的是，EBNF 没有统一标准。常见变体包括：

| 记法 | 可选 | 重复 | 分组 | 使用者 |
|------|------|------|------|--------|
| ISO 14977 | `[ ]` | `{ }` | `( )` | 官方标准 |
| W3C (XML) | `?` | `*` / `+` | `( )` | XML Schema, XHTML |
| ANTLR | `?` | `*` / `+` | `( )` | ANTLR 4 |
| Yacc/Bison | 无内置 | 无内置 | 无内置 | 手动展开为 BNF |

Yacc/Bison 使用**纯 BNF**——没有 `?`、`*`、`+` 操作符。所有可选和重复都必须手动用递归产生式表达。TJS2 的 `tjs.y` 就是这种风格：

```yacc
// tjs.y 中的参数列表（纯 BNF 递归）
func_expr_def_list:               /* 空——零个参数 */
    | T_SYMBOL                    /* 一个参数 */
    | func_expr_def_list T_COMMA T_SYMBOL  /* 多个参数（左递归） */
    ;
```

等价的 EBNF 只需一行：`func_expr_def_list = [ T_SYMBOL { "," T_SYMBOL } ] ;`

## 动手实践：从自然语言到 CFG

### 练习：设计一个 JSON 子集的文法

目标：描述以下 JSON 子集的语法：
- 值可以是：字符串（用 `"` 包围）、数字、`true`、`false`、`null`
- 数组：`[值, 值, ...]`
- 对象：`{字符串: 值, 字符串: 值, ...}`

**第 1 步：确定终结符（Token）**

```
Σ = { string, number, true, false, null, [, ], {, }, :, , }
```

这些是词法器输出的 Token。

**第 2 步：确定非终结符**

```
V = { Value, Array, Object, Elements, Members, Pair }
```

**第 3 步：写产生式**

```bnf
<Value>    ::= string | number | "true" | "false" | "null"
             | <Array> | <Object>

<Array>    ::= "[" "]"
             | "[" <Elements> "]"

<Elements> ::= <Value>
             | <Elements> "," <Value>

<Object>   ::= "{" "}"
             | "{" <Members> "}"

<Members>  ::= <Pair>
             | <Members> "," <Pair>

<Pair>     ::= string ":" <Value>
```

**第 4 步：验证——推导 `{"name": "TJS2", "version": 2}`**

```
Value
⇒ Object
⇒ { Members }
⇒ { Members , Pair }
⇒ { Pair , Pair }
⇒ { string : Value , Pair }
⇒ { string : string , Pair }           ("name": "TJS2")
⇒ { string : string , string : Value }
⇒ { string : string , string : number } ("version": 2)
```

推导成功，文法正确描述了这个 JSON 子集。

### 练习：识别二义性

考虑这个"简化"的表达式文法：

```
Expr → Expr + Expr | Expr * Expr | number
```

输入 `1 + 2 * 3`，写出两棵不同的语法分析树。

**树 1（* 优先）：**
```
    Expr
   / | \
 Expr + Expr
  |    / | \
 1  Expr * Expr
      |      |
      2      3
```

**树 2（+ 优先）：**
```
      Expr
     / | \
  Expr * Expr
  / | \    |
Expr + Expr 3
 |      |
 1      2
```

树 1 计算为 `1 + (2 * 3) = 7`，树 2 计算为 `(1 + 2) * 3 = 9`。**同一输入，两种结果**——这就是二义性的危害。解决方案就是前面展示的分层文法（Expr/Term/Factor）。

## 对照项目源码

TJS2 的语法文法定义在 `cpp/core/tjs2/bison/tjs.y` 中，使用 Yacc/Bison 格式（纯 BNF + 优先级声明）：

```yacc
// tjs.y 第 1-20 行（摘录）
%{
#include "tjsInterCodeGen.h"
#define YYMALLOC  ::malloc
#define YYREALLOC ::realloc
#define YYFREE    ::free
%}

// 第 44-181 行：Token 声明（终结符集合 Σ）
%token T_COMMA T_EQUAL T_AMPERSANDEQUAL ...
%token T_IF T_ELSE T_WHILE T_FOR ...
%token T_CONSTVAL T_SYMBOL T_REGEXP ...
```

几个值得注意的设计：

**1. 表达式的优先级分层：** `tjs.y` 中的表达式规则完全用本节讲的分层 BNF 实现：

```yacc
// 低优先级 → 高优先级
expr           : assign_expr ;                     // 最顶层
assign_expr    : cond_expr | expr T_EQUAL expr ... ;
cond_expr      : logical_or_expr | logical_or_expr '?' expr ':' expr ;
logical_or_expr: logical_and_expr | logical_or_expr T_LOGICALOR logical_and_expr ;
// ... 逐层向下 ...
unary_expr     : postfix_expr | T_EXCRAMATION unary_expr | T_MINUS unary_expr ... ;
postfix_expr   : prim_expr | postfix_expr '[' expr ']' | postfix_expr '(' args ')' ... ;
prim_expr      : T_CONSTVAL | T_SYMBOL | '(' expr ')' ... ;
```

从 `expr` 到 `prim_expr` 共约 15 层——每多一层，运算符优先级高一级。

**2. 列表的左递归模式：** Bison（LALR 解析器）天然支持左递归，TJS2 大量使用：

```yacc
// 参数列表：空 | 单个 | 多个
func_call_expr_list:              /* empty */
    | expr_no_comma               /* 一个参数 */
    | func_call_expr_list T_COMMA expr_no_comma  /* 递归：追加参数 */
    ;
```

相关文件：
- `cpp/core/tjs2/bison/tjs.y` 第 44-181 行 — Token 声明（终结符集合）
- `cpp/core/tjs2/bison/tjs.y` 第 183-230 行 — 优先级与结合性声明
- `cpp/core/tjs2/bison/tjs.y` 第 232-862 行 — 产生式规则（非终结符展开）

## 本节小结

- **上下文无关文法（CFG）** 是描述编程语言语法的标准工具，用四元组 `G = (V, Σ, R, S)` 定义
- **终结符 = Token**（词法器输出），**非终结符 = 语法规则**（文法中的抽象结构）
- **推导**是从起始符号到终结符串的替换过程；最左推导对应自顶向下解析，最右推导的逆对应 LR 解析
- **语法分析树**可视化推导过程，其层次结构天然编码运算符优先级
- **二义性**指同一输入有多棵分析树；通过**分层文法**或**优先级声明**消除
- **BNF** 是 CFG 的标准文本格式；**EBNF** 增加了 `?`/`*`/`+` 简化重复和可选
- Yacc/Bison 使用纯 BNF + 优先级声明，TJS2 的 `tjs.y` 是这种风格的典型示例

## 练习题与答案

### 题目 1：写出 CFG 四元组

为以下语言写出完整的 CFG 四元组 `G = (V, Σ, R, S)`：
- 语言 L = `{aⁿbⁿ | n ≥ 1}`（相同数量的 a 后跟相同数量的 b）

<details>
<summary>查看答案</summary>

```
G = (V, Σ, R, S) 其中：
  V = { S }
  Σ = { a, b }
  R = {
    S → a S b      （递归：每次加一对 ab）
    S → a b        （基础情况：最内层的 ab）
  }
  S = S
```

验证 n=3：`S ⇒ aSb ⇒ aaSbb ⇒ aaabbb` ✓

也可以合并为 `S → a S b | a b`。注意 `S → a S b | ε` 生成的是 `{aⁿbⁿ | n ≥ 0}`（含空串），与题目要求的 `n ≥ 1` 不同。

</details>

### 题目 2：消除二义性

下面的文法是二义的，请改写为无二义性的等价文法，使得 `-` 是左结合的（即 `1 - 2 - 3` = `(1 - 2) - 3`）：

```
E → E - E | number
```

<details>
<summary>查看答案</summary>

使用左递归确保左结合性：

```
E → E - number | number
```

推导 `1 - 2 - 3`：
```
E ⇒ E - number(3)
  ⇒ E - number(2) - number(3)
  ⇒ number(1) - number(2) - number(3)
```

语法分析树：
```
      E
    / | \
   E  -  3
 / | \
E  -  2
|
1
```

只有一棵树，且 `(1-2)` 先被归约——符合左结合语义。

如果要支持多种运算符且保持优先级，就需要前面讲的分层方案（Expr/Term/Factor）。

</details>

### 题目 3：BNF 到 EBNF 转换

将以下 BNF 改写为等价的 EBNF：

```bnf
<stmts>  ::= <stmt> | <stmts> <stmt>
<stmt>   ::= <var_decl> | <assign> | <if_stmt>
<if_stmt>::= "if" "(" <expr> ")" <stmt>
           | "if" "(" <expr> ")" <stmt> "else" <stmt>
```

<details>
<summary>查看答案</summary>

```ebnf
stmts   = stmt { stmt } ;                    (* 1个或多个语句 *)
stmt    = var_decl | assign | if_stmt ;
if_stmt = "if" "(" expr ")" stmt [ "else" stmt ] ;  (* else 可选 *)
```

关键转换：
1. `<stmts> ::= <stmt> | <stmts> <stmt>` 的递归表示"一个或多个 stmt"→ EBNF 的 `stmt { stmt }`
2. `<if_stmt>` 的两条产生式唯一差别是有无 `else` 子句 → EBNF 用 `[ else stmt ]` 表示可选

注意：EBNF 版本没有消除悬挂 else 的二义性——它只是更紧凑地表达了同样的文法。解析器仍需要额外规则（如"else 配最近 if"）来消解。

</details>

## 下一步

[递归下降解析器](02-递归下降解析器.md) — 学习如何将 EBNF 文法直接翻译为代码，用纯手写的递归函数实现自顶向下语法分析。
