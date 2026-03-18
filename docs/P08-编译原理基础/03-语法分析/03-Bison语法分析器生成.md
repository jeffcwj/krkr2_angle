# Bison 语法分析器生成

> **所属模块：** P08-编译原理基础
> **前置知识：** [上下文无关文法与 BNF](01-上下文无关文法与BNF.md)、[递归下降解析器](02-递归下降解析器.md)
> **预计阅读时间：** 30 分钟
> **关键术语：** Bison/Yacc、LALR(1)、移进-归约（Shift-Reduce）、动作（Action）、语义值（Semantic Value）、`%union`、`%token`、`%left`/`%right`、`$$`/`$1`

## 本节目标

读完本节后，你将能够：
1. 解释移进-归约解析的工作原理（栈操作 + 状态转换）
2. 阅读 Bison `.y` 文法文件的四个组成部分
3. 使用 `%left`、`%right`、`%nonassoc` 声明运算符优先级与结合性
4. 在产生式中嵌入语义动作，通过 `$$`、`$1`、`$2` 传递值
5. 理解 LALR(1) 与 LL(1) 的核心差异

## 自底向上解析：移进与归约

递归下降是**自顶向下**——从起始符号展开到终结符。Bison 生成的解析器恰好相反：**自底向上**——从终结符归约到起始符号。

核心机制是两个操作：

| 操作 | 含义 | 类比 |
|------|------|------|
| **移进（Shift）** | 从输入读一个 Token 压入栈 | "吃一个 Token" |
| **归约（Reduce）** | 栈顶若干元素匹配某产生式右侧，弹出并替换为左侧非终结符 | "识别一个结构" |

### 示例：解析 `1 + 2 * 3`

文法：`E → E + T | T`，`T → T * F | F`，`F → number`

```
栈                    输入              动作
------               ------            ------
                     1 + 2 * 3 $       移进
1                    + 2 * 3 $         归约 F → number
F                    + 2 * 3 $         归约 T → F
T                    + 2 * 3 $         归约 E → T
E                    + 2 * 3 $         移进
E +                  2 * 3 $           移进
E + 2                * 3 $             归约 F → number
E + F                * 3 $             归约 T → F
E + T                * 3 $             移进（* 优先级高于 +，不归约 E + T）
E + T *              3 $               移进
E + T * 3            $                 归约 F → number
E + T * F            $                 归约 T → T * F
E + T                $                 归约 E → E + T
E                    $                 接受！
```

关键决策在第 9 行：栈顶是 `E + T`，下一个输入是 `*`。解析器选择**移进**（因为 `*` 优先级高于 `+`），而不是归约 `E → E + T`——这正确实现了乘法优先于加法。

> **常见错误：** 初学者常问"解析器怎么知道该移进还是归约？"答案是：Bison 在编译时生成了一张**解析表**，表中对每个（状态, Token）组合预先计算好了动作。运行时只需查表。

## Bison 文件结构

一个 `.y` 文件由四个部分组成，用 `%%` 分隔：

```yacc
%{
/* 第 1 部分：C/C++ 声明（头文件、宏定义） */
#include <stdio.h>
%}

/* 第 2 部分：Bison 声明（Token、类型、优先级） */
%token NUMBER PLUS TIMES
%left PLUS
%left TIMES

%%

/* 第 3 部分：语法规则 + 语义动作 */
expr : expr PLUS term   { $$ = $1 + $3; }
     | term             { $$ = $1; }
     ;

%%

/* 第 4 部分：辅助 C/C++ 代码（main、yyerror 等） */
void yyerror(const char *s) { fprintf(stderr, "Error: %s\n", s); }
```

### 第 2 部分详解：声明

**Token 声明（终结符）：**

```yacc
%token T_IF T_ELSE T_WHILE        // 无值的 Token
%token <num> T_NUMBER T_INTEGER    // 有值的 Token（类型为 num）
%token <np>  T_SYMBOL              // 有值的 Token（类型为 np，即指针）
```

**语义值类型（`%union`）：**

```yacc
%union {
    int    num;         // 整数值
    double flt;         // 浮点值
    char*  str;         // 字符串指针
    Node*  np;          // AST 节点指针
}
%type <num> expr term factor       // 非终结符的语义值类型
```

`%union` 定义了一个 C union——所有 Token 和非终结符的语义值共享这块内存。`%type` 指定非终结符对应 union 中的哪个字段。

**优先级与结合性：**

```yacc
%left  T_PLUS T_MINUS              // 左结合，最低优先级
%left  T_STAR T_SLASH              // 左结合，高一级
%right T_POWER                     // 右结合（2**3**2 = 2**(3**2)）
%nonassoc T_LT T_GT               // 不可结合（a < b < c 是语法错误）
```

声明**从上到下优先级递增**。同一行的 Token 优先级相同。

## 语义动作详解

产生式右侧花括号中的代码在**归约时**执行：

```yacc
expr : expr PLUS term   { $$ = $1 + $3; }
//     $1   $2   $3       $$
```

| 符号 | 含义 |
|------|------|
| `$$` | 归约后的非终结符（左侧）的语义值 |
| `$1` | 产生式右侧第 1 个符号的语义值 |
| `$2` | 第 2 个符号（此处是 `PLUS`，通常不用） |
| `$3` | 第 3 个符号的语义值 |

### 完整示例：四则运算计算器

```yacc
%{
#include <stdio.h>
int yylex(void);
void yyerror(const char *s);
%}

%union { int val; }
%token <val> NUMBER
%type  <val> expr term factor
%left  '+' '-'
%left  '*' '/'

%%

input : expr '\n'  { printf("结果: %d\n", $1); }
      ;

expr : expr '+' term  { $$ = $1 + $3; }
     | expr '-' term  { $$ = $1 - $3; }
     | term           { $$ = $1; }
     ;

term : term '*' factor { $$ = $1 * $3; }
     | term '/' factor { $$ = $1 / $3; }
     | factor          { $$ = $1; }
     ;

factor : '(' expr ')' { $$ = $2; }    // 注意：$2 是 expr，$1 是 '('
       | NUMBER        { $$ = $1; }
       ;

%%

void yyerror(const char *s) { fprintf(stderr, "%s\n", s); }
int main() { return yyparse(); }
```

编译运行：

```bash
bison -d calc.y            # 生成 calc.tab.c + calc.tab.h
gcc calc.tab.c lex.yy.c -o calc
echo "2 + 3 * 4" | ./calc  # 输出：结果: 14
```

### 构建 AST 而非直接求值

前面的计算器在语义动作中直接计算结果。实际编译器通常构建 AST，后续再遍历处理：

```yacc
%union { int val; struct Node *np; }
%token <val> NUMBER
%type  <np>  expr term factor
%%
expr : expr '+' term  { $$ = make_binop('+', $1, $3); }
     | term           { $$ = $1; }
     ;
/* make_binop 创建一个二元运算节点（操作符 + 左子树 + 右子树） */
```

TJS2 正是这种模式——`MakeNP2(op, left, right)` 创建二元节点，`MakeNP1(op, child)` 创建一元节点，`MakeNP0(op)` 创建叶节点（常量、符号等）。这三个工厂方法构成了整个 AST 构建 API。

## Bison 的错误恢复机制

当输入包含语法错误时，解析器需要一种机制来恢复并继续解析后续内容，而非直接终止。Bison 提供了 `error` 伪 Token：

```yacc
def_list
    :                                     /* 空 */
    | def_list block_or_statement
    | def_list error ";"                  { yyerrok; }
    ;
```

工作原理：

1. 解析器遇到无法匹配的 Token 时，触发错误
2. Bison 开始**弹出栈**，直到找到一个状态能接受 `error` Token
3. 然后跳过输入中的 Token，直到找到 `error` 后面指定的同步 Token（此处为 `;`）
4. `yyerrok` 宏告诉解析器"已从错误中恢复，停止报告后续错误"

TJS2 的 `tjs.y` 第 211-213 行使用了完全相同的模式——遇到语法错误时跳到下一个分号，然后继续解析后续语句。还加了一个安全阀：连续错误超过 20 次就 `YYABORT`（彻底终止解析）。

常见的同步 Token 选择：

| 同步 Token | 适用场景 |
|-----------|---------|
| `;` | 语句级恢复（最常见） |
| `}` | 块级恢复 |
| `)` | 表达式括号恢复 |
| `EOF` | 最后手段，跳过所有剩余输入 |

> **常见错误：** 不放 `error` 规则时，第一个语法错误就导致整个解析失败。生产级解析器**必须**有错误恢复——用户代码中一个拼写错误不应该让编译器放弃所有后续内容的分析。

## LALR(1) vs LL(1)

| 特性 | LL(1)（递归下降） | LALR(1)（Bison） |
|------|-------------------|-------------------|
| 方向 | 自顶向下 | 自底向上 |
| 左递归 | ❌ 必须消除 | ✅ 天然支持 |
| 优先级 | 用文法分层（Expr/Term/Factor） | `%left`/`%right` 声明 |
| 前看 Token | 1 个 | 1 个（但利用方式不同） |
| 错误恢复 | 手写，灵活 | `error` Token，相对固定 |
| 实现方式 | 手写代码 | 生成状态机表 |
| 代表工具 | 手写、ANTLR | Bison/Yacc |

LALR(1) 能识别的文法严格大于 LL(1)——所有 LL(1) 文法都是 LALR(1)，反之不然。

## 移进-归约冲突

当解析器在某状态下，既可以移进又可以归约时，就产生**移进-归约冲突（Shift-Reduce Conflict）**。

经典案例：悬挂 else

```yacc
stmt : IF expr THEN stmt          // 产生式 1
     | IF expr THEN stmt ELSE stmt // 产生式 2
     ;
```

当栈上有 `IF expr THEN stmt`，下一个 Token 是 `ELSE` 时：
- **移进** `ELSE` → 走产生式 2（else 配近 if）
- **归约**产生式 1 → else 配远 if

Bison 默认选择**移进**——恰好是我们想要的行为。这就是 TJS2 的 `tjs.y` 文件头部 `%expect 1` 的含义——告诉 Bison"我知道有 1 个移进-归约冲突，这是故意的"。

### 归约-归约冲突

当栈顶同时匹配两条产生式的右侧时，产生**归约-归约冲突**——这通常意味着文法设计有问题，需要修改。Bison 会报错并选择先出现的产生式。

## Bison C++ 模式

TJS2 使用 Bison 的 C++ 模式（`%language "C++"`），与 C 模式有几个重要差异：

```yacc
// tjs.y 第 1-7 行
%require "3.8.2"              // 要求 Bison 3.8.2+
%language "C++"               // 生成 C++ 代码
%expect 1                     // 允许 1 个 S/R 冲突（悬挂 else）
%header "tjs.tab.hpp"         // 生成头文件名
%output "tjs.tab.cpp"         // 生成源文件名
%define api.namespace {TJS}   // 解析器类放在 TJS 命名空间

%parse-param { tTJSScriptBlock *ptr }  // 解析器构造参数
%lex-param   { tTJSScriptBlock *ptr }  // 词法器调用参数
```

C++ 模式下，Bison 生成一个 `TJS::parser` 类（而非全局函数 `yyparse()`）。构造时传入 `tTJSScriptBlock*`，它同时作为词法器接口和语义动作的上下文。

三个关键宏简化了代码：

```cpp
#define cc (ptr->GetCurrentContext())   // 当前编译上下文
#define cn (cc->GetCurrentNode())       // 当前 AST 节点
#define lx (ptr->GetLexicalAnalyzer())  // 词法器实例
```

## 常见错误与解决方案

### 错误 1：语义值类型不匹配

```
bison: warning: type clash on default action: <np> != <num>
```

产生式左侧的 `%type` 声明与右侧某个符号的类型不一致，或忘记写 `%type` 声明。解决：确保每个非终结符都有 `%type <字段名>` 声明，且 `$$` 的赋值类型正确。

### 错误 2：未声明的 Token

```
bison: error: symbol T_PLUS is used, but is not defined as a token
```

文法规则中使用了 `T_PLUS`，但忘记在声明部分写 `%token T_PLUS`。使用字符字面量（如 `'+'`）则不需要声明。

### 错误 3：不可达的非终结符

```
bison: warning: 2 nonterminals useless in grammar
```

某些非终结符没有被任何其他规则引用，无法从起始符号推导到达。检查是否拼错了名称，或遗漏了引用该规则的上层产生式。

## 动手实践：读懂 Bison 输出

运行 `bison -v calc.y` 会生成 `calc.output`，包含完整的解析表。关键段落示例：

```
State 7

    expr  ->  expr . '+' term    (rule 1)
    expr  ->  expr . '-' term    (rule 2)

    '+'  shift, and go to state 8
    '-'  shift, and go to state 9

    $default  reduce using rule 3 (expr)
```

这段表示：在状态 7 中，如果看到 `+` 或 `-` 则**移进**；否则（其他 Token 或 EOF）**归约**。点 `.` 标记了解析器在产生式中的当前位置——"已看到 `expr`，等待 `+` 或 `-`"。

> **调试技巧：** 遇到冲突时，用 `bison -v` 查看 `.output` 文件中的冲突状态，找到具体是哪两条产生式在争抢。

### 从 `.y` 到可执行文件的完整流程

```
calc.y ──bison -d──> calc.tab.c + calc.tab.h
                            │           │
                            │    Token 枚举（NUMBER, PLUS 等）
                            ▼
calc.l ──flex──────> lex.yy.c ──include──> calc.tab.h
                            │
                            ▼
gcc calc.tab.c lex.yy.c ─────────────────> calc（可执行文件）
```

Bison 生成解析器（`calc.tab.c`），Flex 生成词法器（`lex.yy.c`），两者通过 `calc.tab.h` 中的 Token 枚举衔接。GCC 编译链接两个 `.c` 文件得到最终可执行文件。

如果不使用 Flex（如 TJS2 的手写词法器），只需自己实现 `yylex()` 函数，返回 Token 类型即可。

## 对照项目源码

TJS2 的 `tjs.y` 是本节理论的完美实例：

**1. `%union` 与语义值类型：**

```yacc
// tjs.y 第 32-35 行
%union {
    tjs_int        num;    // 整数（Token 值、操作符代码）
    tTJSExprNode * np;     // AST 节点指针
};
```

只有两个字段——整数和 AST 节点指针。所有表达式的语义值都是 `tTJSExprNode*`。

**2. 语义动作中的 AST 构建：**

```yacc
// tjs.y 中的加法表达式（简化）
expr : expr "+" expr   { $$ = cn->MakeNP2(T_PLUS, $1, $3); }
     | expr "-" expr   { $$ = cn->MakeNP2(T_MINUS, $1, $3); }
     ;
```

`MakeNP2` 创建一个二元运算 AST 节点——操作符 + 左子树 + 右子树。归约时立即构建 AST，不需要后续遍历。

**3. 优先级声明层次：**

TJS2 在 `tjs.y` 第 183-191 行没有用大量 `%left`/`%right` 声明，而是**主要通过文法分层**实现优先级（`assign_expr → cond_expr → logical_or_expr → ... → add_sub_expr → mul_div_expr → unary_expr → factor_expr`）。只在一个地方使用了优先级声明——悬挂 else 的处理：

```yacc
// tjs.y 第 190-191 行
%nonassoc LOWER_ELSE     // 虚拟 Token，优先级低于 ELSE
%nonassoc T_ELSE         // else 关键字
```

配合 `%prec LOWER_ELSE`（第 226 行），这让 `if` 语句在没有 `else` 时的归约优先级低于 `T_ELSE` 的移进优先级，从而正确解决悬挂 else 问题。

**4. `%expect 1`：** 正如前面分析的，TJS2 文法中恰好有一个悬挂 else 的移进-归约冲突，用 `%expect 1` 声明为已知。

**5. 错误恢复：**

```yacc
// tjs.y 第 211-213 行
| def_list error ";"    { if(ptr->CompileErrorCount>20)
                              YYABORT;
                            else yyerrok; }
```

遇到语法错误时跳到最近的分号，然后继续解析。累计错误超过 20 个则放弃整个文件——避免产生大量无意义的级联错误消息。

相关文件：
- `cpp/core/tjs2/bison/tjs.y` — 完整文法定义（862 行）
- `cpp/core/tjs2/bison/tjs.tab.hpp` — 生成的头文件（Token 枚举）
- `cpp/core/tjs2/bison/tjs.tab.cpp` — 生成的解析器代码
- `cpp/core/tjs2/tjsScriptBlock.cpp` — 调用 `parser.parse()` 的入口

## 本节小结

- Bison 生成**自底向上**的 LALR(1) 解析器，通过**移进**和**归约**两个操作解析输入
- `.y` 文件分四部分：C 声明、Bison 声明（Token/优先级/类型）、语法规则+动作、辅助代码
- `%left`/`%right`/`%nonassoc` 声明优先级和结合性，**从上到下优先级递增**
- 语义动作在归约时执行，用 `$$`、`$1`、`$2`... 传递和计算语义值
- 移进-归约冲突通常可通过优先级声明解决；`%expect N` 声明已知的冲突数量
- TJS2 使用 Bison C++ 模式，生成 `TJS::parser` 类，语义动作中构建 AST 节点

## 练习题与答案

### 题目 1：移进-归约追踪

用前面的文法（`E → E+T | T`，`T → T*F | F`，`F → number`），手动追踪 `(2 + 3) * 4` 的移进-归约过程。

<details>
<summary>查看答案</summary>

```
栈               输入                动作
                 ( 2 + 3 ) * 4 $    移进
(                2 + 3 ) * 4 $      移进
( 2              + 3 ) * 4 $        归约 F→number, T→F, E→T
( E              + 3 ) * 4 $        移进
( E +            3 ) * 4 $          移进
( E + 3          ) * 4 $            归约 F→number, T→F
( E + T          ) * 4 $            归约 E→E+T
( E              ) * 4 $            移进
( E )            * 4 $              归约 F→(E)
F                * 4 $              归约 T→F
T                * 4 $              移进
T *              4 $                移进
T * 4            $                  归约 F→number
T * F            $                  归约 T→T*F
T                $                  归约 E→T
E                $                  接受
```

结果：先计算 `2+3=5`（括号内），再计算 `5*4=20`。括号改变了优先级。

</details>

### 题目 2：优先级声明

写出 Bison 优先级声明，使得以下运算符按正确优先级和结合性工作：
- `=`（赋值，右结合，最低）
- `||`（逻辑或，左结合）
- `&&`（逻辑与，左结合）
- `==`、`!=`（比较，左结合）
- `+`、`-`（左结合）
- `*`、`/`（左结合）
- `!`（一元非，右结合，最高）

<details>
<summary>查看答案</summary>

```yacc
%right T_EQUAL              // = 右结合，最低优先级
%left  T_LOGICALOR          // ||
%left  T_LOGICALAND         // &&
%left  T_EQUALEQUAL T_NOTEQUAL  // == !=
%left  T_PLUS T_MINUS       // + -
%left  T_STAR T_SLASH       // * /
%right T_EXCLAMATION        // ! 右结合，最高优先级
```

从上到下优先级递增。`=` 右结合确保 `a = b = c` 解析为 `a = (b = c)`。`!` 右结合确保 `!!x` 解析为 `!(!x)`。

</details>

### 题目 3：`%expect` 的含义

TJS2 的 `tjs.y` 声明了 `%expect 1`。如果去掉这行，Bison 会怎样？如果实际冲突数变为 2，Bison 会怎样？

<details>
<summary>查看答案</summary>

1. **去掉 `%expect 1`：** Bison 会生成一个**警告**（不是错误），报告有 1 个移进-归约冲突。解析器仍然会生成，Bison 默认选择移进。但警告在 CI 中可能被当作错误。

2. **实际冲突变为 2：** Bison 会报**错误**——`%expect 1` 明确声明只应该有 1 个冲突，实际有 2 个意味着引入了新的文法问题。这是 `%expect` 的安全网价值——任何意外的冲突变化都会被捕获。

`%expect` 不是"忽略冲突"，而是"精确断言冲突数量"——类似单元测试中的 `EXPECT_EQ`。

</details>

## 下一步

[TJS2 语法分析器源码分析](04-TJS2语法分析器源码分析.md) — 深入阅读 TJS2 的 `tjs.y` 文法文件，分析表达式层次、语句结构、函数/类声明的完整语法规则。
