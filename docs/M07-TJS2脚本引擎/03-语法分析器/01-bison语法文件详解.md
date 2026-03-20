# Bison 语法文件详解

> **所属模块：** M07-TJS2脚本引擎
> **前置知识：** [P08-编译原理基础](../../P08-编译原理基础/README.md)、[02-词法分析器](../02-词法分析器/01-tjsLex整体架构.md)
> **预计阅读时间：** 60 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 tjs.y 文件的整体结构和 Bison 3.8 C++ API 的配置方式
2. 掌握 TJS2 语法中 120+ Token 声明的分类体系和编号规则
3. 理解 `cc`/`cn`/`lx` 三大宏如何将 Bison 语义动作与编译器后端连接
4. 读懂从 `program` 到 `factor_expr` 的完整文法规则层次
5. 分析 TJS2 独有的语法特性（swap 运算符、postfix if、incontextof 等）

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| Bison | GNU Bison | 一种 LALR(1) 语法分析器生成器，读取 `.y` 文件生成 C/C++ 解析代码 |
| 产生式 | Production Rule | 文法中描述"一个非终结符由哪些符号序列组成"的规则，如 `expr : expr "+" expr` |
| 语义动作 | Semantic Action | 产生式匹配成功时执行的 C/C++ 代码，写在 `{ }` 中 |
| 终结符 | Terminal Symbol | 不可再分解的最小语法单元，对应词法分析器输出的 Token |
| 非终结符 | Non-terminal Symbol | 由其他符号组合而成的语法单元，如 `expr`、`statement` |
| 移进-归约冲突 | Shift-Reduce Conflict | 解析器同时可以"读入下一个 Token"或"应用某条规则"时产生的歧义 |
| 悬挂 else | Dangling Else | `if-if-else` 结构中 `else` 应该与哪个 `if` 配对的经典歧义问题 |
| AST | Abstract Syntax Tree | 抽象语法树，源代码的树形结构表示，每个节点代表一个语法结构 |
| 上下文栈 | Context Stack | TJS2 编译器维护的嵌套作用域栈，每进入函数/类/属性时压入新上下文 |
| 常量池 | Constant Pool / Data Area | 编译器收集的常量值数组，Token 通过索引引用池中的值 |

## 1. tjs.y 文件概览

### 1.1 文件在项目中的位置

tjs.y 是 TJS2 脚本引擎的核心语法定义文件，位于：

```
krkr2/
└── cpp/core/tjs2/
    └── bison/
        ├── tjs.y          ← 主语法文件（862 行）——本节重点
        ├── tjspp.y        ← 预处理器语法（@if/@set/@endif）
        ├── tjsdate.y      ← Date 对象的日期字符串解析语法
        ├── tjs.tab.hpp    ← Bison 生成的头文件（Token 枚举）
        └── tjs.tab.cpp    ← Bison 生成的解析器实现
```

**关键事实**：tjs.y 只有 862 行，却定义了一门完整编程语言的全部语法——包括变量声明、函数定义、类继承、异常处理、正则表达式、内联数组/字典等。这种紧凑性得益于 Bison 的声明式文法描述能力。

### 1.2 文件的四大组成部分

一个 Bison `.y` 文件的标准结构如下：

```
┌─────────────────────────────────┐
│  第一部分：序言（Prologue）       │  ← %require, %language, %code, %union
│  - Bison 配置指令                │
│  - C++ 头文件包含                │
│  - 宏定义                       │
├─────────────────────────────────┤
│  第二部分：声明（Declarations）   │  ← %token, %type, %nonassoc
│  - Token 声明（终结符）           │
│  - 类型声明（非终结符）           │
│  - 优先级/结合性声明              │
├──────── %% ─────────────────────┤  ← 分隔符
│  第三部分：规则（Grammar Rules）  │  ← 产生式 + 语义动作
│  - program → global_list         │
│  - statement 的 18 种形式         │
│  - 表达式优先级阶梯               │
│  - 内联数组/字典字面量             │
├──────── %% ─────────────────────┤  ← 分隔符
│  第四部分：尾声（Epilogue）       │  ← 用户 C++ 代码（tjs.y 中为空）
└─────────────────────────────────┘
```

tjs.y 的实际分布：

| 部分 | 行号范围 | 行数 | 内容 |
|------|----------|------|------|
| 序言 | 1-41 | 41 | Bison 配置 + C++ 头文件 + 宏 + %union + 前置声明 |
| Token 声明 | 44-191 | 148 | 120+ Token 定义 + 类型声明 + 优先级 |
| 文法规则 | 192-862 | 670 | 全部产生式和语义动作 |
| 尾声 | 862 | 1 | 仅一个 `%%` 标记（无额外代码） |

## 2. 序言部分：Bison 配置详解

### 2.1 Bison 版本与语言配置

tjs.y 的前 7 行是 Bison 的全局配置指令：

```yacc
%require "3.8.2"           // 要求 Bison 3.8.2 或更高版本
%language "C++"            // 生成 C++ 代码（而非默认的 C 代码）
%expect 1                  // 预期恰好 1 个移进-归约冲突（悬挂 else）
%header "tjs.tab.hpp"      // 生成的头文件名
%output "tjs.tab.cpp"      // 生成的源文件名
%define api.namespace {TJS} // 所有生成代码放入 TJS 命名空间
```

逐行解析：

**`%require "3.8.2"`** —— 这是一条版本守卫指令。如果你用低于 3.8.2 的 Bison 来编译这个 `.y` 文件，Bison 会直接报错退出。为什么需要 3.8.2？因为 tjs.y 使用了较新的 C++ API 特性，包括 `%define api.namespace`（在旧版本中语法不同）和改进的错误消息支持。在 Windows 上，需要安装 winflexbison 包来获取兼容版本。

**`%language "C++"`** —— 告诉 Bison 生成 C++ 类而非 C 函数。生成的解析器是一个 `TJS::parser` 类，主入口是 `parser::parse()` 方法。C++ 模式相比 C 模式的主要优势是：类型安全更好、命名空间隔离、析构函数自动清理。

**`%expect 1`** —— 声明文法中预期有恰好 1 个移进-归约冲突。如果 Bison 发现冲突数不是 1，会报警告。这个唯一的冲突就是经典的"悬挂 else"问题，稍后会详细解释。这条指令的作用是告诉 Bison（和代码审查者）："这个冲突是故意的，不是 bug。"

**`%define api.namespace {TJS}`** —— 所有生成的解析器代码放入 `TJS` 命名空间。这意味着解析器类的完整名称是 `TJS::parser`，Token 枚举是 `TJS::parser::token::T_PLUS` 等。与项目中其他 TJS2 代码（都在 `TJS` 命名空间下）保持一致。

### 2.2 解析器参数与词法接口

```yacc
%parse-param { tTJSScriptBlock *ptr }   // 解析器构造函数参数
%lex-param   { tTJSScriptBlock *ptr }   // 词法分析器调用参数
```

这两行定义了解析器与外部代码的连接方式：

- **`%parse-param`**：给生成的 `parser` 类构造函数添加一个参数 `ptr`。调用方式变成 `TJS::parser parser_obj(script_block_ptr); parser_obj.parse();`
- **`%lex-param`**：每次解析器调用 `yylex()` 获取下一个 Token 时，会把 `ptr` 传给 `yylex()`

`tTJSScriptBlock` 是一个编译单元对象，它同时持有词法分析器（`tTJSLexicalAnalyzer`）和编译器上下文栈（`tTJSInterCodeContext` 栈）。通过这个 `ptr` 参数，语义动作中的代码可以访问编译器的所有功能。

### 2.3 三大宏：cc、cn、lx

序言中最关键的是三个宏定义：

```cpp
%code top
{
#include <cstdlib>
#include "tjsTypes.h"
#include "tjsError.h"
#include "tjsArray.h"
#include "tjsDictionary.h"
#include "tjsInterCodeGen.h"
#include "tjsScriptBlock.h"

/* current context */
#define cc (ptr->GetCurrentContext())

/* current node */
#define cn (cc->GetCurrentNode())

/* lexical analyzer */
#define lx (ptr->GetLexicalAnalyzer())
}
```

这三个宏是理解 tjs.y 中所有语义动作的关键：

```
                    tTJSScriptBlock (ptr)
                   ┌────────────────────────┐
                   │                        │
    lx 宏 ────►    │  GetLexicalAnalyzer()   │──► tTJSLexicalAnalyzer
                   │                        │     - GetString(idx)  获取符号名
                   │                        │     - GetValue(idx)   获取常量值
                   │                        │     - SetNextIsBareWord()
                   │                        │     - SetStartOfRegExp()
                   │                        │
    cc 宏 ────►    │  GetCurrentContext()     │──► tTJSInterCodeContext (编译器)
                   │                        │     - MakeNP0/1/2/3()  构建 AST 节点
                   │                        │     - CreateExprCode() 生成表达式代码
                   │                        │     - EnterBlock() / ExitBlock()
                   │                        │     - EnterWhileCode() / ExitWhileCode()
                   │                        │     - AddLocalVariable()
                   │                        │     - PushCurrentNode() / PopCurrentNode()
                   │                        │
    cn 宏 ────►    │  cc->GetCurrentNode()   │──► tTJSExprNode (当前 AST 节点栈顶)
                   │                        │     - Add(node)  添加子节点
                   │                        │     - SetValue() 设置节点值
                   └────────────────────────┘
```

**`cc` 宏（current context）**—— 获取当前编译上下文。`tTJSInterCodeContext` 是编译器的核心类，既负责构建 AST 节点（通过 `MakeNP0`/`MakeNP1`/`MakeNP2`/`MakeNP3`），也负责生成字节码（通过 `CreateExprCode`、`EnterWhileCode` 等方法）。编译器维护一个上下文栈——每进入一个函数、类、属性定义时，会 `PushContextStack()` 压入新上下文；离开时 `PopContextStack()` 弹出。这确保了每个函数/类都有独立的代码区和常量池。

**`cn` 宏（current node）**—— 获取当前节点栈顶。这个宏专门用于构建内联数组 `[...]` 和内联字典 `%[...]` 这样的嵌套结构。解析器通过 `PushCurrentNode()` 把数组/字典的根节点压栈，然后在解析每个元素时用 `cn->Add(element)` 把元素挂到根节点上，最后 `PopCurrentNode()` 弹出。

**`lx` 宏（lexical analyzer）**—— 获取词法分析器。语义动作中频繁调用 `lx->GetString(idx)` 获取标识符名称、`lx->GetValue(idx)` 获取常量值。这里的 `idx` 是 Token 的语义值（`$1`、`$2` 等），它是一个整数索引，指向词法分析器内部的 `Values[]` 数组。

### 2.4 %union 与语义值类型

```yacc
%union{
    tjs_int         num;      // Token 的索引值（指向 Values[] 数组）
    tTJSExprNode *  np;       // AST 节点指针
}
```

Bison 的 `%union` 定义了语义值栈中每个元素可能的类型。tjs.y 只有两种：

1. **`num`（`tjs_int` 类型）**—— 用于终结符（Token）。当词法分析器返回 `T_SYMBOL`、`T_CONSTVAL`、`T_REGEXP` 时，语义值是一个整数索引。例如 `$1` 的值是 `42`，表示"去 `lx->Values[42]` 取实际的字符串或数值"。这种间接寻址设计避免了在语义值栈中存储大型对象。

2. **`np`（`tTJSExprNode*` 类型）**—— 用于非终结符（表达式）。文法规则匹配后，语义动作会调用 `cc->MakeNP0/1/2/3()` 创建 AST 节点，把节点指针赋给 `$$`。

对应的类型声明：

```yacc
// 终结符类型声明——这四个 Token 携带 <num> 类型的语义值
%token <num>    T_CONSTVAL    // 常量值（数字、字符串字面量）
%token <num>    T_SYMBOL      // 标识符（变量名、函数名）
%token <num>    T_REGEXP      // 正则表达式字面量
%token <num>    T_VARIANT     // 变体类型（内部使用）

// 非终结符类型声明——所有表达式非终结符都产生 <np> 类型
%type <np>
    expr comma_expr assign_expr cond_expr logical_or_expr
    logical_and_expr inclusive_or_expr exclusive_or_expr and_expr
    identical_expr compare_expr shift_expr add_sub_expr
    mul_div_expr mul_div_expr_and_asterisk
    unary_expr incontextof_expr priority_expr factor_expr
    call_arg call_arg_list func_expr_def func_call_expr
    expr_no_comma inline_array array_elm inline_dic dic_elm
    const_inline_array const_inline_dic
```

注意：大量非终结符（如 `statement`、`block`、`def_list`、`func_def`）**没有**类型声明。这意味着它们不产生语义值——它们的语义动作直接调用 `cc->EnterXxx()` / `cc->ExitXxx()` 等方法来生成字节码，而不是构建 AST 节点后返回。

### 2.5 悬挂 else 的解决

```yacc
%nonassoc LOWER_ELSE    // 虚拟 Token，优先级低于 T_ELSE
%nonassoc T_ELSE        // else 关键字的优先级
```

这两行解决了经典的"悬挂 else"（Dangling Else）问题。考虑以下代码：

```javascript
if (a) if (b) x = 1; else x = 2;
```

这段代码有两种解读：
- 解读 A：`else` 属于内层 `if (b)` —— 即 `if(a) { if(b) x=1; else x=2; }`
- 解读 B：`else` 属于外层 `if (a)` —— 即 `if(a) { if(b) x=1; } else { x=2; }`

几乎所有语言都选择解读 A（else 与最近的 if 配对）。Bison 的 LALR(1) 解析器遇到这种情况时会产生一个移进-归约冲突：看到 `else` 时，可以"归约"（把内层 if 语句合并），也可以"移进"（继续读入 else）。

tjs.y 的解决方案：

```yacc
// statement 中，没有 else 的 if 用 %prec LOWER_ELSE 标记
statement
    : ...
    | if %prec LOWER_ELSE    // 给 "if" 规则赋予 LOWER_ELSE 优先级
    | if_else                // if-else 规则使用 T_ELSE 的默认优先级
    ;
```

由于 `LOWER_ELSE` 在 `T_ELSE` 之前声明，它的优先级更低。当解析器面临"归约为不带 else 的 if"还是"移进 else"的选择时，移进的优先级（`T_ELSE`）更高，所以解析器选择移进——即 `else` 与最近的 `if` 配对。这就是 `%expect 1` 中那唯一的冲突。

## 3. Token 声明体系

### 3.1 Token 的四大分类

tjs.y 中声明了 120+ 个 Token，可以分为四大类别：

```
┌──────────────────────────────────────────────────────────┐
│                    TJS2 Token 分类体系                     │
├──────────────┬───────────────────────────────────────────┤
│ 第一类：运算符 │ 45+ 个，含赋值、比较、位运算、特殊运算符     │
│ (带字符串别名) │ T_PLUS("+"), T_SWAP("<->"), T_DISCEQUAL("===") │
├──────────────┼───────────────────────────────────────────┤
│ 第二类：关键字 │ 50+ 个，含控制流、声明、类型、字面量关键字   │
│ (带字符串别名) │ T_IF("if"), T_CLASS("class"), T_VOID("void")  │
├──────────────┼───────────────────────────────────────────┤
│ 第三类：内部 Token │ 17 个，词法分析器永远不会产生             │
│ (无字符串别名)   │ 仅在 AST 节点中作为操作码使用              │
│                 │ T_UPLUS, T_UMINUS, T_EVAL, T_WITHDOT 等  │
├──────────────┼───────────────────────────────────────────┤
│ 第四类：带值 Token │ 4 个，携带 <num> 类型语义值              │
│ (%token <num>)   │ T_CONSTVAL, T_SYMBOL, T_REGEXP, T_VARIANT │
└──────────────┴───────────────────────────────────────────┘
```

### 3.2 运算符 Token 详解

运算符 Token 使用 `T_名称 "符号"` 的格式声明。字符串别名让文法规则更易读——你可以在产生式中写 `"+"` 而不是 `T_PLUS`：

```yacc
// 赋值运算符家族（16 个）
T_EQUAL             "="       // 普通赋值
T_AMPERSANDEQUAL    "&="      // 位与赋值
T_VERTLINEEQUAL     "|="      // 位或赋值
T_CHEVRONEQUAL      "^="      // 位异或赋值
T_MINUSEQUAL        "-="      // 减法赋值
T_PLUSEQUAL         "+="      // 加法赋值
T_PERCENTEQUAL      "%="      // 取模赋值
T_SLASHEQUAL        "/="      // 除法赋值
T_BACKSLASHEQUAL    "\\="     // 整数除法赋值（TJS2 独有）
T_ASTERISKEQUAL     "*="      // 乘法赋值
T_LOGICALOREQUAL    "||="     // 逻辑或赋值（TJS2 独有）
T_LOGICALANDEQUAL   "&&="     // 逻辑与赋值（TJS2 独有）
T_RBITSHIFTEQUAL    ">>>="    // 无符号右移赋值
T_LARITHSHIFTEQUAL  "<<="     // 左移赋值
T_RARITHSHIFTEQUAL  ">>="     // 算术右移赋值
T_SWAP              "<->"     // 交换运算符（TJS2 独有）
```

**TJS2 独有运算符解析**：

- **`<->`（交换运算符）**：`a <-> b` 等价于 `var tmp = a; a = b; b = tmp;`，但在 VM 层面有专门的 `VM_CHGTHIS` 优化实现，不需要临时变量
- **`\\`（整数除法）**：`7 \\ 2` 结果为 `3`，等价于 JavaScript 的 `Math.floor(7/2)`。这是 TJS2 独有的运算符，JavaScript 中不存在
- **`||=` 和 `&&=`**：逻辑赋值运算符。`a ||= b` 等价于 `a = a || b`。JavaScript 在 ES2021 才引入这两个运算符，而 TJS2 在 2000 年就已支持
- **`===` 和 `!==`（严格相等）**：与 JavaScript 语义相同，不进行类型转换。在 tjs.y 中命名为 `T_DISCEQUAL`（discriminating equal）和 `T_DISCNOTEQUAL`

```yacc
// 比较运算符（8 个）
T_NOTEQUAL      "!="      // 不等于（会类型转换）
T_EQUALEQUAL    "=="      // 等于（会类型转换）
T_DISCNOTEQUAL  "!=="     // 严格不等于
T_DISCEQUAL     "==="     // 严格等于
T_LT            "<"       // 小于
T_GT            ">"       // 大于
T_LTOREQUAL     "<="      // 小于等于
T_GTOREQUAL     ">="      // 大于等于

// 位运算和移位（6 个）
T_VERTLINE      "|"       // 位或
T_CHEVRON       "^"       // 位异或
T_AMPERSAND     "&"       // 位与（也用作 & 单目运算符）
T_RARITHSHIFT   ">>"      // 算术右移
T_LARITHSHIFT   "<<"      // 左移
T_RBITSHIFT     ">>>"     // 无符号右移

// 算术运算符（6 个）
T_PERCENT       "%"       // 取模
T_SLASH         "/"       // 除法
T_BACKSLASH     "\\"      // 整数除法（TJS2 独有）
T_ASTERISK      "*"       // 乘法（也用作 * 单目运算符）
T_PLUS          "+"       // 加法
T_MINUS         "-"       // 减法

// 一元和后缀运算符（5 个）
T_EXCRAMATION   "!"       // 逻辑非（也用作后缀求值运算符）
T_TILDE         "~"       // 位取反
T_DECREMENT     "--"      // 自减
T_INCREMENT     "++"      // 自增
T_SHARP         "#"       // 字符编码（chr/asc）
T_DOLLAR        "$"       // 字符编码反向

// 其他运算符
T_QUESTION      "?"       // 三目条件运算符
T_LOGICALOR     "||"      // 逻辑或
T_LOGICALAND    "&&"      // 逻辑与
T_COMMA         ","       // 逗号运算符
T_DOT           "."       // 成员访问
T_LBRACKET      "["       // 索引访问
T_LPARENTHESIS  "("       // 函数调用
```

### 3.3 关键字 Token 详解

关键字 Token 也使用字符串别名，让产生式可以写成 `"if"` 而不是 `T_IF`：

```yacc
// 控制流关键字（14 个）
T_IF        "if"          T_ELSE      "else"
T_FOR       "for"         T_WHILE     "while"
T_DO        "do"          T_SWITCH    "switch"
T_CASE      "case"        T_DEFAULT   "default"
T_BREAK     "break"       T_CONTINUE  "continue"
T_RETURN    "return"      T_THROW     "throw"
T_TRY       "try"         T_CATCH     "catch"

// 声明关键字（10 个）
T_VAR       "var"         T_CONST     "const"
T_FUNCTION  "function"    T_CLASS     "class"
T_EXTENDS   "extends"     T_PROPERTY  "property"
T_SETTER    "setter"      T_GETTER    "getter"
T_EXPORT    "export"      T_IMPORT    "import"

// 类型关键字（4 个）——用于类型转换运算符
T_INT       "int"         T_REAL      "real"
T_STRING    "string"      T_OCTET     "octet"

// 面向对象关键字（9 个）
T_THIS          "this"          T_SUPER         "super"
T_GLOBAL        "global"        T_NEW           "new"
T_DELETE        "delete"        T_TYPEOF        "typeof"
T_INSTANCEOF    "instanceof"    T_INVALIDATE    "invalidate"
T_ISVALID       "isvalid"

// TJS2 独有关键字（4 个）
T_INCONTEXTOF   "incontextof"   // 改变执行上下文
T_WITH          "with"          // with 语句
T_DEBUGGER      "debugger"      // 触发调试器断点
T_SYNCHRONIZED  "synchronized"  // 同步块（保留，未实现）

// 字面量关键字（6 个）
T_TRUE      "true"        T_FALSE     "false"
T_NULL      "null"        T_VOID      "void"
T_NAN       "NaN"         T_INFINITY  "Infinity"

// 访问修饰符（3 个，保留但未完全实现）
T_PRIVATE   "private"     T_PUBLIC    "public"
T_PROTECTED "protected"   T_STATIC    "static"

// 其他保留字
T_ENUM      "enum"        T_GOTO      "goto"
T_FINALLY   "finally"     T_IN        "in"
T_OMIT      "..."         // 省略参数标记
```

**值得注意的点**：

- `T_OMIT` 的字符串别名是 `"..."`（三个点），用于函数调用中表示"传递所有参数"：`func(...)` 
- `enum`、`goto`、`finally`、`synchronized` 是保留字但未实现——如果在代码中使用会被识别为关键字，但语法分析器不接受包含它们的产生式
- `T_IN` 用于 `expr in expr`（检查成员是否存在），与 JavaScript 的 `in` 运算符语义相同

### 3.4 内部 Token（AST 专用操作码）

这 17 个 Token 是 tjs.y 最独特的设计之一——它们**永远不会被词法分析器产生**，只在语义动作中用作 AST 节点的操作码：

```yacc
// 无字符串别名 = 词法分析器不产生
T_UPLUS            // 一元正号 (+expr)，区别于二元加法
T_UMINUS           // 一元负号 (-expr)，区别于二元减法
T_EVAL             // 后缀求值运算符 (expr!)
T_POSTDECREMENT    // 后缀自减 (expr--)
T_POSTINCREMENT    // 后缀自增 (expr++)
T_IGNOREPROP       // 忽略属性访问器 (&expr)
T_PROPACCESS       // 强制属性访问 (*expr)
T_ARG              // 函数调用参数节点
T_EXPANDARG        // 展开参数 (expr* 或 *)
T_INLINEARRAY      // 内联数组根节点 [...]
T_ARRAYARG         // 数组元素节点
T_INLINEDIC        // 内联字典根节点 %[...]
T_DICELM           // 字典元素节点（键值对）
T_WITHDOT          // with 上下文属性访问 (.prop)
T_THIS_PROXY       // this 代理（内部优化）
T_WITHDOT_PROXY    // with-dot 代理（内部优化）
```

**为什么需要内部 Token？** 以 `+` 运算符为例，它在 TJS2 中有两种含义：

```javascript
a + b      // 二元加法：MakeNP2(T_PLUS, a, b)
+a         // 一元正号：MakeNP1(T_UPLUS, a)
```

词法分析器对两种情况都返回 `T_PLUS`，但语法分析器通过产生式上下文区分它们，并在构建 AST 时使用不同的操作码。这样字节码编译器只需检查节点的 `Op` 字段就能知道具体是哪种运算。

类似地，`!` 运算符有两种角色：
- 前缀 `!expr`：逻辑非，使用 `T_EXCRAMATION`
- 后缀 `expr!`：求值/调用，使用 `T_EVAL`（内部 Token）

## 4. 文法规则总览

### 4.1 顶层结构：program → def_list

tjs.y 的文法从 `program` 非终结符开始，经过三层到达具体的语句和表达式：

```
program
  └── global_list
        ├── PushContextStack("global", ctTopLevel)  ← 创建顶层编译上下文
        ├── def_list                                 ← 递归地解析定义/语句列表
        └── PopContextStack()                        ← 弹出顶层上下文
```

对应的 Bison 规则（源文件第 196-213 行）：

```yacc
program
    : global_list
;

global_list
    :                               { ptr->PushContextStack(
                                        TJS_W("global"), ctTopLevel); }
      def_list                      { ptr->PopContextStack(); }
;

def_list
    :                               // 空——递归基
    | def_list block_or_statement   // 递归：先解析前面的，再解析一条
    | def_list error ";"            // 错误恢复（第三节详讲）
                                    { if(ptr->CompileErrorCount>20) YYABORT;
                                      else yyerrok; }
;
```

**解析流程**：

1. 解析器启动时，首先匹配 `program → global_list`
2. `global_list` 的语义动作立即执行 `PushContextStack("global", ctTopLevel)`，创建一个类型为 `ctTopLevel`（顶层代码）的编译上下文
3. 然后进入 `def_list`，它是一个左递归规则，不断地解析 `block_or_statement` 直到输入结束
4. 最后 `PopContextStack()` 弹出顶层上下文，触发 `Commit()` 将生成的字节码固化

`def_list` 的第三条规则是错误恢复——如果某条语句解析失败，Bison 会跳过直到遇到 `;`，然后恢复解析。如果累计错误超过 20 个，则调用 `YYABORT` 终止解析。

### 4.2 语句的 18 种形式

`statement` 非终结符是 TJS2 语法的核心枢纽，定义了所有可能的语句形式（源文件第 223-244 行）：

```yacc
statement
    : ";"                           // 空语句
    | expr ";"                      // 表达式语句
    | if %prec LOWER_ELSE           // if（无 else）
    | if_else                       // if-else
    | while                         // while 循环
    | do_while                      // do-while 循环
    | for                           // for 循环
    | "break" ";"                   // break
    | "continue" ";"                // continue
    | "debugger" ";"                // 调试器断点
    | variable_def                  // 变量定义（var/const）
    | func_def                      // 函数定义
    | property_def                  // 属性定义（TJS2 独有）
    | class_def                     // 类定义
    | return                        // 返回语句
    | switch                        // switch-case
    | with                          // with 语句
    | case                          // case/default 标签
    | try                           // try-catch 异常处理
    | throw                         // 抛出异常
;
```

注意 `block`（由 `{` `}` 包围的代码块）**不在** `statement` 中——它通过 `block_or_statement` 非终结符处理：

```yacc
block_or_statement
    : statement
    | block
;

block
    : "{"                   { cc->EnterBlock(); }
      def_list
      "}"                   { cc->ExitBlock(); }
;
```

`EnterBlock()` 和 `ExitBlock()` 管理局部变量的作用域——每个 `{}` 块创建一个新的变量作用域层。

### 4.3 控制流语句

**while 循环**（源文件第 254-257 行）：

```yacc
while
    : "while"                       { cc->EnterWhileCode(false); }
      "(" expr ")"                  { cc->CreateWhileExprCode($4, false); }
      block_or_statement            { cc->ExitWhileCode(false); }
;
```

语义动作的执行顺序和生成的字节码结构：

```
EnterWhileCode(false)         →  记录循环起始地址
                                 ┌──────────────────┐
CreateWhileExprCode($4,false) →  │ [条件表达式代码]    │
                                 │ JNF exit_addr     │ ← 条件为假时跳出
                                 ├──────────────────┤
ExitWhileCode(false)          →  │ [循环体代码]       │
                                 │ JMP loop_start    │ ← 跳回循环开始
                                 │ exit_addr:        │ ← 跳出目标
                                 └──────────────────┘
```

**do-while 循环**（源文件第 262-267 行）：

```yacc
do_while
    : "do"                          { cc->EnterWhileCode(true); }
      block_or_statement
      "while"
      "(" expr ")"                  { cc->CreateWhileExprCode($6, true); }
      ";"                           { cc->ExitWhileCode(true); }
;
```

与 `while` 的区别在于 `EnterWhileCode(true)` 的参数为 `true`，表示循环体在条件判断之前执行。

**if / if-else**（源文件第 270-280 行）：

```yacc
if
    : "if" "("                      { cc->EnterIfCode(); }
      expr                          { cc->CreateIfExprCode($4); }
      ")" block_or_statement        { cc->ExitIfCode(); }
;

if_else
    : if "else"                     { cc->EnterElseCode(); }
      block_or_statement            { cc->ExitElseCode(); }
;
```

`if_else` 的巧妙之处：它不是从零开始定义，而是**复用** `if` 非终结符作为前缀。这意味着 `if_else` 规则首先完整匹配一个 `if`（执行 `EnterIfCode` → `CreateIfExprCode` → `ExitIfCode`），然后看到 `else` 时执行 `EnterElseCode()`（回填 if 的跳转地址并设置 else 的跳转）。

**for 循环**（源文件第 283-311 行）：

```yacc
for
    : "for" "("
      for_first_clause ";"         // 初始化子句
      for_second_clause ";"        // 条件子句
      for_third_clause ")"         // 更新子句
      block_or_statement           { cc->ExitForCode(); }
;

for_first_clause
    : /* 空 */                     { cc->EnterForCode(); }
    |                              { cc->EnterForCode(); }
      variable_def_inner           // 支持 for(var i=0; ...)
    | expr                         { cc->EnterForCode();
                                     cc->CreateExprCode($1); }
;

for_second_clause
    : /* 空 */                     { cc->CreateForExprCode(NULL); }
    | expr                         { cc->CreateForExprCode($1); }
;

for_third_clause
    : /* 空 */                     { cc->SetForThirdExprCode(NULL); }
    | expr                         { cc->SetForThirdExprCode($1); }
;
```

for 循环的三个子句各自独立处理。`for_first_clause` 支持三种形式：空、变量声明（`for(var i=0; ...)`）、或普通表达式（`for(i=0; ...)`）。第三子句通过 `SetForThirdExprCode()` 保存表达式节点但不立即生成代码——实际代码在循环体之后生成。

### 4.4 声明语句

**变量定义**（源文件第 314-348 行）：

```yacc
variable_def
    : variable_def_inner ";"
;

variable_def_inner
    : "var" variable_id_list
    | "const" variable_id_list
      /* const: note that current version does not
         actually disallow re-assigning new value */
;

variable_id
    : T_SYMBOL variable_type                    { cc->AddLocalVariable(
                                                    lx->GetString($1)); }
    | T_SYMBOL variable_type "=" expr_no_comma  { cc->InitLocalVariable(
                                                    lx->GetString($1), $4); }
;

variable_type
    : /* 空 */          // 不指定类型（最常见）
    | ":" T_SYMBOL      // var x : SomeType（类型注解，但不强制）
    | ":" T_VOID        // var x : void
    | ":" T_INT         // var x : int
    | ":" T_REAL        // var x : real
    | ":" T_STRING      // var x : string
    | ":" T_OCTET       // var x : octet
;
```

**关键发现**：

1. **`const` 不强制只读**——源码注释明确说"当前版本不会实际阻止重新赋值"。`var` 和 `const` 在编译层面完全等价，`const` 只是一个"程序员意图"标记
2. **类型注解被忽略**——`variable_type` 支持 `: int`、`: string` 等语法，但编译器不做类型检查。TJS2 是动态类型语言，这些注解纯粹是文档用途
3. **`variable_id_list`** 支持逗号分隔的多变量声明：`var a, b = 1, c`

**函数定义**（源文件第 350-410 行）：

```yacc
// 具名函数定义（语句形式）
func_def
    : "function" T_SYMBOL              { ptr->PushContextStack(
                                            lx->GetString($2), ctFunction);
                                         cc->EnterBlock(); }
      func_decl_arg_opt variable_type
      block                            { cc->ExitBlock();
                                         ptr->PopContextStack(); }
;

// 匿名函数表达式
func_expr_def
    : "function"                       { ptr->PushContextStack(
                                            TJS_W("(anonymous)"),
                                            ctExprFunction);
                                         cc->EnterBlock(); }
      func_decl_arg_opt variable_type
      block                            { cc->ExitBlock();
                                         tTJSVariant v(cc);
                                         ptr->PopContextStack();
                                         $$ = cc->MakeNP0(token::T_CONSTVAL);
                                         $$->SetValue(v); }
;
```

函数定义与函数表达式的关键区别：

| 特性 | func_def（语句） | func_expr_def（表达式） |
|------|-----------------|----------------------|
| 名称 | 必须有名字 `T_SYMBOL` | 匿名 `"(anonymous)"` |
| 上下文类型 | `ctFunction` | `ctExprFunction` |
| 返回值 | 无（语句不产生值） | 返回 `tTJSVariant(cc)` 包装的上下文对象 |
| 用途 | `function foo() { }` | `var f = function() { };` |

`func_expr_def` 的返回值处理很精妙：`tTJSVariant v(cc)` 将当前编译上下文（`tTJSInterCodeContext`）包装成一个变体值，然后创建 `T_CONSTVAL` 节点存储这个值。这意味着函数体编译后的字节码**本身就是一个常量值**。

**函数参数定义**（支持默认值和收集参数）：

```yacc
func_decl_arg_opt
    : /* 空 */                                          // 无参数
    | "(" func_decl_arg_collapse ")"                    // 仅收集参数
    | "(" func_decl_arg_list ")"                        // 普通参数列表
    | "(" func_decl_arg_at_least_one ","
          func_decl_arg_collapse ")"                    // 普通 + 收集
;

func_decl_arg
    : T_SYMBOL variable_type                            // 普通参数
        { cc->AddFunctionDeclArg(lx->GetString($1), NULL); }
    | T_SYMBOL variable_type "=" expr_no_comma          // 带默认值参数
        { cc->AddFunctionDeclArg(lx->GetString($1), $4); }
;

func_decl_arg_collapse
    : "*"                       { cc->AddFunctionDeclArgCollapse(NULL); }
    | T_SYMBOL variable_type "*" { cc->AddFunctionDeclArgCollapse(
                                      lx->GetString($1)); }
;
```

收集参数（collapse argument）是 TJS2 版的 rest parameter：

```javascript
// TJS2 的收集参数语法
function foo(a, b, rest*) {
    // rest 是一个数组，包含 a 和 b 之后的所有参数
}

// 等价于 JavaScript 的：
function foo(a, b, ...rest) { }
```

**类定义**（源文件第 455-478 行）：

```yacc
class_def
    : "class" T_SYMBOL                 { ptr->PushContextStack(
                                            lx->GetString($2), ctClass); }
      class_extender
      block                            { ptr->PopContextStack(); }
;

class_extender
    :                                   // 无继承
    | "extends" expr_no_comma           // 单继承
        { cc->CreateExtendsExprCode($2, true); }
    | "extends" expr_no_comma ","       // 多继承起始
        { cc->CreateExtendsExprCode($2, false); }
      extends_list                      // 多继承列表
;
```

**TJS2 支持多继承**——`class Foo extends Bar, Baz, Qux { }` 是合法的。`CreateExtendsExprCode` 的第二个参数 `hold` 表示是否是最后一个基类。多继承时，每个基类的构造函数都会被调用，属性按声明顺序叠加。

**属性定义**（源文件第 412-452 行）：

```yacc
property_def
    : "property" T_SYMBOL
      "{"                              { ptr->PushContextStack(
                                            lx->GetString($2), ctProperty); }
      property_handler_def_list
      "}"                              { ptr->PopContextStack(); }
;

property_handler_def_list
    : property_handler_setter                          // 只有 setter
    | property_handler_getter                          // 只有 getter
    | property_handler_setter property_handler_getter  // setter + getter
    | property_handler_getter property_handler_setter  // getter + setter（顺序无关）
;

property_handler_setter
    : "setter" "(" T_SYMBOL variable_type ")"
        { ptr->PushContextStack(TJS_W("(setter)"), ctPropertySetter);
          cc->EnterBlock();
          cc->SetPropertyDeclArg(lx->GetString($3)); }
      block
        { cc->ExitBlock(); ptr->PopContextStack(); }
;

property_handler_getter
    : property_getter_handler_head
        { ptr->PushContextStack(TJS_W("(getter)"), ctPropertyGetter);
          cc->EnterBlock(); }
      block
        { cc->ExitBlock(); ptr->PopContextStack(); }
;

property_getter_handler_head
    : "getter" "(" ")" variable_type   // getter() 形式
    | "getter" variable_type           // getter 形式（省略括号）
;
```

属性是 TJS2 独有的语法特性（JavaScript 在 ES5 才通过 `Object.defineProperty` 支持类似功能）：

```javascript
// TJS2 的属性定义
class MyClass {
    var _value = 0;
    
    property value {
        getter() {
            return _value;
        }
        setter(v) {
            if (v >= 0) _value = v;
        }
    }
}

var obj = new MyClass();
obj.value = 42;       // 调用 setter
var x = obj.value;    // 调用 getter
```

### 4.5 switch / with / try-catch

**switch 语句**（源文件第 488-505 行）：

```yacc
switch
    : "switch" "("
      expr ")"                     { cc->EnterSwitchCode($3); }
      block                        { cc->ExitSwitchCode(); }
;

case
    : "case" expr ":"              { cc->ProcessCaseCode($2); }
    | "default" ":"                { cc->ProcessCaseCode(NULL); }
;
```

注意 `case` 是作为独立的 `statement` 处理的，而不是嵌套在 `switch` 内部——这是因为 `switch` 的 `block` 展开后就是 `def_list`，而 `case` 是 `statement` 的一种。`EnterSwitchCode` 保存 switch 表达式的求值结果，`ProcessCaseCode` 生成比较代码。

**with 语句**（源文件第 494-499 行）：

```yacc
with
    : "with" "("
      expr ")"                     { cc->EnterWithCode($3); }
      block_or_statement           { cc->ExitWithCode(); }
;
```

`with` 语句将一个对象设为当前上下文，使得可以直接用 `.property` 访问其成员：

```javascript
with (obj) {
    .x = 10;        // 等价于 obj.x = 10
    .y = .x + 20;   // 等价于 obj.y = obj.x + 20
}
```

这里的 `.x` 在语法分析器中被识别为 `T_WITHDOT` 节点（见 priority_expr 规则中的 `"." T_SYMBOL` 产生式）。

**try-catch 异常处理**（源文件第 507-525 行）：

```yacc
try
    : "try"                        { cc->EnterTryCode(); }
      block_or_statement
      catch
      block_or_statement           { cc->ExitTryCode(); }
;

catch
    : "catch"                      { cc->EnterCatchCode(NULL); }
    | "catch" "(" ")"              { cc->EnterCatchCode(NULL); }
    | "catch" "(" T_SYMBOL ")"     { cc->EnterCatchCode(
                                        lx->GetString($3)); }
;
```

TJS2 的 try-catch 语法与 JavaScript 基本相同，但注意：
- **没有 `finally`**——虽然 `T_FINALLY` 是保留字，但语法中没有使用它
- `catch` 的参数是可选的：`catch { }` 和 `catch() { }` 都是合法的（等价于不捕获异常对象）

## 5. 表达式规则与优先级阶梯

### 5.1 Postfix if 表达式

TJS2 支持一种类似 Ruby 的后缀 if 语法（源文件第 528-535 行）：

```yacc
expr
    : comma_expr                        { $$ = $1; }
    | comma_expr "if" expr              { $$ = cc->MakeNP2(token::T_IF, $1, $3); }
;
```

这意味着你可以这样写：

```javascript
x = 42 if (condition);   // 等价于 if (condition) x = 42;
obj.update() if (dirty); // 等价于 if (dirty) obj.update();
```

后缀 if 在视觉小说引擎的脚本中很常用——脚本作者可以把主要操作放在前面，条件放在后面，更接近自然语言的表达顺序。

### 5.2 赋值表达式（右结合）

赋值运算符是**右结合**的——`a = b = c` 解析为 `a = (b = c)`。在 Bison 中通过右递归实现（源文件第 544-562 行）：

```yacc
assign_expr
    : cond_expr                            { $$ = $1; }
    | cond_expr "<->" assign_expr          { $$ = cc->MakeNP2(T_SWAP, $1, $3); }
    | cond_expr "="   assign_expr          { $$ = cc->MakeNP2(T_EQUAL, $1, $3); }
    | cond_expr "&="  assign_expr          { $$ = cc->MakeNP2(T_AMPERSANDEQUAL, $1, $3); }
    | cond_expr "|="  assign_expr          { $$ = cc->MakeNP2(T_VERTLINEEQUAL, $1, $3); }
    // ... 共 16 种赋值运算符
;
```

注意产生式的形式是 `cond_expr OP assign_expr`——右侧递归地引用自身（`assign_expr`），这自然实现了右结合性。左侧是 `cond_expr`（更高优先级），确保赋值运算符的优先级最低（在逗号之后）。

### 5.3 完整的优先级阶梯

TJS2 的表达式优先级从低到高排列如下（每一层是一个非终结符）：

```
优先级   非终结符                    运算符                        结合性
───────────────────────────────────────────────────────────────────────
最低  │ expr                    │ postfix if                   │ ─
      │ comma_expr              │ ,                            │ 左
      │ assign_expr             │ = <-> &= |= ^= -= += ...    │ 右
      │ cond_expr               │ ? :                          │ 右
      │ logical_or_expr         │ ||                           │ 左
      │ logical_and_expr        │ &&                           │ 左
      │ inclusive_or_expr       │ | (位或)                     │ 左
      │ exclusive_or_expr       │ ^ (位异或)                   │ 左
      │ and_expr                │ & (位与)                     │ 左
      │ identical_expr          │ == != === !==                │ 左
      │ compare_expr            │ < > <= >=                    │ 左
      │ shift_expr              │ >> << >>>                    │ 左
      │ add_sub_expr            │ + -                          │ 左
      │ mul_div_expr            │ % / \ *                      │ 左
      │ unary_expr              │ ! ~ ++ -- new delete typeof  │ 右
      │                         │ # $ + - & * int real string  │
      │                         │ instanceof in (int)(real)    │
      │ incontextof_expr        │ incontextof                  │ 左
      │ priority_expr           │ . [] () ++ -- ! (后缀)       │ 左
最高  │ factor_expr             │ 常量 标识符 this super ...    │ ─
```

这个阶梯与 JavaScript 几乎完全一致，额外增加了：
- `incontextof`（在 `unary_expr` 和 `priority_expr` 之间）
- `<->`（swap，与 `=` 同级）
- `\\`（整数除法，与 `*` `/` `%` 同级）
- 后缀 `!`（求值运算符，与 `.` `[]` `()` 同级）

### 5.4 乘法的特殊处理：mul_div_expr_and_asterisk

乘法运算符 `*` 在 TJS2 中有歧义——它既是二元乘法，也是一元的"属性访问"前缀运算符。tjs.y 用一个特殊的非终结符来消歧义（源文件第 630-640 行）：

```yacc
mul_div_expr
    : unary_expr                              { $$ = $1; }
    | mul_div_expr "%" unary_expr             { $$ = cc->MakeNP2(T_PERCENT, $1, $3); }
    | mul_div_expr "/" unary_expr             { $$ = cc->MakeNP2(T_SLASH, $1, $3); }
    | mul_div_expr "\\" unary_expr            { $$ = cc->MakeNP2(T_BACKSLASH, $1, $3); }
    | mul_div_expr_and_asterisk unary_expr    { $$ = cc->MakeNP2(T_ASTERISK, $1, $2); }
;

mul_div_expr_and_asterisk
    : mul_div_expr "*"                        { $$ = $1; }
;
```

技巧：`mul_div_expr_and_asterisk` 将 `*` "吞掉"并返回左操作数——这样当解析器看到 `expr * expr` 时，`*` 被左操作数"贪婪地"消耗，右操作数直接跟在后面。如果 `*` 出现在一元位置（如 `*prop`），它会被 `unary_expr` 中的 `"*" unary_expr` 规则捕获，不会与二元乘法混淆。

### 5.5 一元运算符的丰富性

TJS2 的一元运算符极为丰富（源文件第 642-668 行），远超 JavaScript：

```yacc
unary_expr
    : incontextof_expr                          // 基准
    | "!" unary_expr           // 逻辑非
    | "~" unary_expr           // 位取反
    | "--" unary_expr          // 前缀自减
    | "++" unary_expr          // 前缀自增
    | "new" func_call_expr     // 对象创建（注意：修改 opcode 为 T_NEW）
    | "invalidate" unary_expr  // 使对象无效（释放资源）
    | "isvalid" unary_expr     // 检查对象是否有效（前缀形式）
    | incontextof_expr "isvalid"  // isvalid 的后缀形式
    | "delete" unary_expr      // 删除成员
    | "typeof" unary_expr      // 获取类型
    | "#" unary_expr           // 字符编码（character → code）
    | "$" unary_expr           // 字符编码反向（code → character）
    | "+" unary_expr           // 一元正号 → T_UPLUS
    | "-" unary_expr           // 一元负号 → T_UMINUS
    | "&" unary_expr           // 忽略属性访问器（直接访问底层值）
    | "*" unary_expr           // 强制属性访问（即使没有属性访问器也尝试）
    | incontextof_expr "instanceof" unary_expr  // 类型检查
    | incontextof_expr "in" unary_expr          // 成员检查
    | "(" "int" ")" unary_expr    // C 风格类型转换 (int)expr
    | "int" unary_expr            // 函数风格类型转换 int expr
    | "(" "real" ")" unary_expr   // (real)expr
    | "real" unary_expr           // real expr
    | "(" "string" ")" unary_expr // (string)expr
    | "string" unary_expr         // string expr
;
```

**TJS2 独有运算符详解**：

- **`&expr`（IGNOREPROP）**：绕过属性的 getter，直接获取底层存储值。如果 `obj.x` 有 getter，`&obj.x` 跳过 getter 直接读取
- **`*expr`（PROPACCESS）**：强制通过属性访问器。即使 `obj.x` 没有定义 getter/setter，`*obj.x` 也会尝试通过属性接口访问
- **`#expr`**：将字符转为 ASCII/Unicode 编码值，如 `#"A"` 返回 `65`
- **`$expr`**：将编码值转为字符，如 `$65` 返回 `"A"`
- **`invalidate`**：TJS2 的资源释放机制。由于 TJS2 使用引用计数而非垃圾回收，`invalidate obj` 显式标记对象为无效，释放其持有的资源
- **`isvalid`**：检查对象是否已被 invalidate。支持前缀 `isvalid obj` 和后缀 `obj isvalid` 两种语法

**`new` 的特殊处理**：

```yacc
| "new" func_call_expr     { $$ = $2; $$->SetOpcode(token::T_NEW); }
```

`new` 不是创建新节点，而是复用 `func_call_expr`（函数调用表达式）的结果，然后**修改其操作码**为 `T_NEW`。这意味着 `new Foo(args)` 在 AST 层面和 `Foo(args)` 的结构相同，只是根节点的 Op 从 `T_LPARENTHESIS` 变成了 `T_NEW`。

## 6. 内联数组与字典字面量

### 6.1 内联数组 [...]

TJS2 的数组字面量使用 `[elem1, elem2, ...]` 语法（源文件第 740-758 行）：

```yacc
inline_array
    : "["                           { tTJSExprNode *node =
                                      cc->MakeNP0(token::T_INLINEARRAY);
                                      cc->PushCurrentNode(node); }
      array_elm_list
      "]"                           { $$ = cn; cc->PopCurrentNode(); }
;

array_elm_list
    : array_elm                     { cn->Add($1); }
    | array_elm_list "," array_elm  { cn->Add($3); }
;

array_elm
    : /* 空 */                      { $$ = cc->MakeNP1(token::T_ARRAYARG, NULL); }
    | expr_no_comma                 { $$ = cc->MakeNP1(token::T_ARRAYARG, $1); }
;
```

**PushCurrentNode / PopCurrentNode 模式**：这是一个栈式节点构建模式：

1. 创建 `T_INLINEARRAY` 根节点，压入当前节点栈
2. 每解析一个数组元素，通过 `cn->Add()` 将元素节点添加为根节点的子节点
3. 解析完成后，`cn` 就是构建好的完整数组节点，`PopCurrentNode()` 弹出

**空元素**：`array_elm` 可以为空，这意味着 `[,,,]` 是合法的——创建一个有 4 个 `void` 元素的数组。

### 6.2 内联字典 %[...]

TJS2 的字典字面量使用 `%[key, value, key, value]` 或 `%[key: value, key: value]` 语法（源文件第 760-791 行）：

```yacc
inline_dic
    : "%" "["                       { tTJSExprNode *node =
                                      cc->MakeNP0(token::T_INLINEDIC);
                                      cc->PushCurrentNode(node); }
      dic_elm_list
      dic_dummy_elm_opt
      "]"                           { $$ = cn; cc->PopCurrentNode(); }
;

dic_elm
    : expr_no_comma "," expr_no_comma
        { $$ = cc->MakeNP2(token::T_DICELM, $1, $3); }
    | T_SYMBOL ":" expr_no_comma
        { tTJSVariant val(lx->GetString($1));
          tTJSExprNode *node0 = cc->MakeNP0(token::T_CONSTVAL);
          node0->SetValue(val);
          $$ = cc->MakeNP2(token::T_DICELM, node0, $3); }
;
```

字典元素有两种写法：

```javascript
// 逗号分隔写法（传统 TJS2 风格）
var d1 = %["name", "Alice", "age", 25];

// 冒号写法（类 JSON 风格）
var d2 = %[name: "Alice", age: 25];
```

冒号写法会把 `T_SYMBOL` 自动转为字符串常量（`T_CONSTVAL`），所以 `name: "Alice"` 和 `"name", "Alice"` 效果完全相同。

`dic_dummy_elm_opt` 允许在最后一个元素后跟一个逗号：`%[a: 1, b: 2,]`——这是一个实用的语法糖，方便代码生成和版本控制的 diff。

### 6.3 编译时常量数组与字典

TJS2 还支持编译时常量的数组和字典（源文件第 795-858 行）：

```yacc
const_inline_array
    : "(" "const" ")" "["          { tTJSExprNode *node =
                                     cc->MakeNP0(token::T_CONSTVAL);
                                     iTJSDispatch2 * dsp = TJSCreateArrayObject();
                                     node->SetValue(tTJSVariant(dsp, dsp));
                                     dsp->Release();
                                     cc->PushCurrentNode(node); }
      const_array_elm_list_opt
      "]"                          { $$ = cn; cc->PopCurrentNode(); }
;
```

语法是 `(const)[elem1, elem2]` 和 `(const)%[key, val]`。与普通数组/字典的关键区别：

| 特性 | 普通 `[...]` | 常量 `(const)[...]` |
|------|-------------|---------------------|
| 创建时机 | 运行时 | 编译时 |
| 元素类型 | 任意表达式 | 仅常量值、void、嵌套常量容器 |
| AST 节点 | T_INLINEARRAY + 元素节点 | T_CONSTVAL（直接存储对象） |
| 性能 | 每次执行都创建新数组 | 编译一次，运行时直接引用 |

常量数组的元素直接通过 `cn->AddArrayElement()` / `cn->AddDictionaryElement()` 添加到已创建的数组/字典对象中，不生成任何运行时代码。

## 7. 函数调用与参数传递

### 7.1 函数调用表达式

```yacc
func_call_expr
    : priority_expr "(" call_arg_list ")"
        { $$ = cc->MakeNP2(token::T_LPARENTHESIS, $1, $3); }
;
```

函数调用在 AST 中用 `T_LPARENTHESIS` 节点表示——左子节点是被调用的函数，右子节点是参数列表。

### 7.2 参数列表的三种形式

```yacc
call_arg_list
    : "..."                     { $$ = cc->MakeNP0(token::T_OMIT); }
    | call_arg                  { $$ = cc->MakeNP1(token::T_ARG, $1); }
    | call_arg_list "," call_arg { $$ = cc->MakeNP2(token::T_ARG, $3, $1); }
;

call_arg
    : /* 空 */                  { $$ = NULL; }
    | "*"                       { $$ = cc->MakeNP1(token::T_EXPANDARG, NULL); }
    | mul_div_expr_and_asterisk { $$ = cc->MakeNP1(token::T_EXPANDARG, $1); }
    | expr_no_comma             { $$ = $1; }
;
```

**三种特殊参数**：

1. **`...`（省略）**：`func(...)` 表示"把当前函数的所有参数原封不动地传给 func"。在 KAG 脚本的事件处理中非常常用
2. **`*`（展开空数组）**：`func(*)` 传入一个空的展开参数
3. **`expr*`（展开表达式）**：`func(array*)` 将数组展开为多个参数，类似 JavaScript 的 `func(...array)`

注意参数可以为空——`func(a,,b)` 是合法的，中间的空参数传入 `void`。

## 动手实践

### 实验 1：阅读 Bison 生成的代码

运行以下命令，使用 Bison 生成解析器代码（需要安装 Bison 3.8.2+）：

```bash
# Windows（安装 winflexbison 后）
cd krkr2/cpp/core/tjs2/bison
win_bison --version    # 确认版本 >= 3.8.2
win_bison tjs.y        # 生成 tjs.tab.cpp 和 tjs.tab.hpp

# Linux / macOS
cd krkr2/cpp/core/tjs2/bison
bison --version
bison tjs.y
```

生成后，查看 `tjs.tab.hpp` 中的 Token 枚举：

```cpp
// tjs.tab.hpp（生成的）
namespace TJS {
    class parser {
    public:
        enum token {
            T_COMMA = 258,      // 注意：从 258 开始
            T_EQUAL = 259,
            T_AMPERSANDEQUAL = 260,
            // ... 120+ 个 Token
        };
    };
}
```

所有自定义 Token 的编号从 258 开始（0-255 保留给 ASCII 单字符 Token，256-257 被 Bison 内部使用）。

### 实验 2：手动追踪一段代码的解析过程

以下面的 TJS2 代码为例，追踪解析器的规则匹配顺序：

```javascript
var x = 1 + 2 * 3;
```

解析过程：

```
1. def_list → def_list block_or_statement
2. block_or_statement → statement
3. statement → variable_def
4. variable_def → variable_def_inner ";"
5. variable_def_inner → "var" variable_id_list
6. variable_id_list → variable_id
7. variable_id → T_SYMBOL variable_type "=" expr_no_comma
8. expr_no_comma → assign_expr → cond_expr → ... → add_sub_expr
9. add_sub_expr → add_sub_expr "+" mul_div_expr
10. add_sub_expr 左边 → mul_div_expr → unary_expr → ... → factor_expr → T_CONSTVAL (1)
11. mul_div_expr → mul_div_expr_and_asterisk unary_expr
12. mul_div_expr_and_asterisk → mul_div_expr "*" → factor_expr (2) "*"
13. unary_expr → factor_expr → T_CONSTVAL (3)

AST 结果：
    InitLocalVariable("x",
        MakeNP2(T_PLUS,
            MakeNP0(T_CONSTVAL, 1),
            MakeNP2(T_ASTERISK,
                MakeNP0(T_CONSTVAL, 2),
                MakeNP0(T_CONSTVAL, 3))))
```

### 实验 3：理解上下文栈变化

追踪以下代码的上下文栈操作：

```javascript
class Animal {
    var name;
    function speak() {
        return "...";
    }
    property displayName {
        getter() { return name; }
    }
}
```

上下文栈变化：

```
[global]                           ← PushContextStack("global", ctTopLevel)
[global, Animal]                   ← PushContextStack("Animal", ctClass)
[global, Animal]                   ← AddLocalVariable("name") — 在 Animal 上下文中
[global, Animal, speak]            ← PushContextStack("speak", ctFunction)
[global, Animal]                   ← PopContextStack() — speak 编译完成
[global, Animal, displayName]      ← PushContextStack("displayName", ctProperty)
[global, Animal, displayName, (getter)] ← PushContextStack("(getter)", ctPropertyGetter)
[global, Animal, displayName]      ← PopContextStack() — getter 编译完成
[global, Animal]                   ← PopContextStack() — displayName 编译完成
[global]                           ← PopContextStack() — Animal 编译完成
```

## 对照项目源码

tjs.y 是 TJS2 编译器前端的核心文件，与以下源文件紧密配合：

相关文件：
- `cpp/core/tjs2/bison/tjs.y` 第 1-862 行 —— 完整的 Bison 语法定义文件，本节重点
- `cpp/core/tjs2/tjsInterCodeGen.h` 第 181-234 行 —— `tTJSExprNode` 类定义，AST 节点结构
- `cpp/core/tjs2/tjsInterCodeGen.h` 第 246-872 行 —— `tTJSInterCodeContext` 类定义，语义动作中 `cc` 宏指向的编译器类
- `cpp/core/tjs2/tjsInterCodeGen.h` 第 630-638 行 —— `MakeNP0`/`MakeNP1`/`MakeNP2`/`MakeNP3` 方法声明
- `cpp/core/tjs2/tjsInterCodeGen.h` 第 546-600 行 —— `EnterWhileCode`、`CreateIfExprCode`、`EnterBlock` 等语义动作方法声明
- `cpp/core/tjs2/tjsScriptBlock.cpp` —— `SetText()` 方法创建词法分析器并调用 `parser{this}.parse()` 启动解析
- `cpp/core/tjs2/tjsLex.cpp` 第 875-945 行 —— `yylex()` 函数实现，将词法分析器与 Bison 连接
- `cpp/core/tjs2/bison/tjs.tab.hpp` —— Bison 生成的头文件，包含 `TJS::parser::token` 枚举

解析器的调用链路：

```
tTJSScriptBlock::SetText(code)
    ├── new tTJSLexicalAnalyzer(...)   // 创建词法分析器
    ├── TJS::parser parser(this)        // 创建解析器，this = tTJSScriptBlock
    └── parser.parse()                  // 启动 LALR(1) 解析
         ├── yylex(&yylval, ptr)        // 解析器反复调用 yylex 获取 Token
         │   └── lx->GetNext()          // 词法分析器返回下一个 Token
         └── 语义动作中调用 cc->XXX()    // 构建 AST 并生成字节码
```

## 本节小结

- tjs.y 是一个 862 行的 Bison 3.8 C++ 语法文件，定义了 TJS2 的完整语法
- 文件使用 `%language "C++"` 生成 C++ 解析器类 `TJS::parser`
- 三大宏 `cc`（编译器）、`cn`（当前节点）、`lx`（词法分析器）是所有语义动作的入口
- `%union` 只有两种类型：`num`（Token 索引值）和 `np`（AST 节点指针）
- 120+ 个 Token 分为四类：运算符、关键字、内部 Token（AST 操作码）、带值 Token
- 17 个内部 Token 永远不被词法分析器产生，仅在 AST 中使用
- 悬挂 else 通过 `%nonassoc LOWER_ELSE` / `%nonassoc T_ELSE` 解决
- 表达式优先级通过 13 层非终结符阶梯实现，与 JavaScript 基本一致
- TJS2 独有语法：`<->` swap、postfix `if`、`incontextof`、`property` 定义、`&`/`*` 属性运算符、`#`/`$` 字符编码、`(const)[...]` 编译时常量容器
- 内联数组/字典使用 PushCurrentNode/PopCurrentNode 栈式模式构建

## 练习题与答案

### 题目 1：Token 分类

以下 Token 中，哪些是"内部 Token"（词法分析器不会产生，仅在 AST 中使用）？

A. `T_PLUS`  
B. `T_UPLUS`  
C. `T_EVAL`  
D. `T_EXCRAMATION`  
E. `T_POSTINCREMENT`  
F. `T_INCREMENT`  
G. `T_WITHDOT`  
H. `T_DOT`

<details>
<summary>查看答案</summary>

**内部 Token 是 B、C、E、G。**

- `T_PLUS`（A）—— 有字符串别名 `"+"`，词法分析器产生，用于加法和一元正号
- `T_UPLUS`（B）—— **内部 Token**，无字符串别名。语法分析器在识别一元正号 `+expr` 时，用 `MakeNP1(T_UPLUS, expr)` 创建节点，以区别于二元加法 `T_PLUS`
- `T_EVAL`（C）—— **内部 Token**，用于后缀 `expr!` 求值运算符
- `T_EXCRAMATION`（D）—— 有字符串别名 `"!"`，词法分析器产生，用于前缀逻辑非
- `T_POSTINCREMENT`（E）—— **内部 Token**，用于后缀 `expr++`。前缀 `++expr` 使用 `T_INCREMENT`
- `T_INCREMENT`（F）—— 有字符串别名 `"++"`，词法分析器产生
- `T_WITHDOT`（G）—— **内部 Token**，用于 `with` 块中的 `.property` 访问
- `T_DOT`（H）—— 有字符串别名 `"."`，词法分析器产生，用于成员访问

判断方法：在 tjs.y 中，内部 Token 的声明没有字符串别名（`%token` 后面只有 Token 名称，没有 `"..."` 部分）。

</details>

### 题目 2：优先级判断

以下 TJS2 表达式如何被解析？写出等价的带括号表达式：

```javascript
a = b + c * d if e || f
```

<details>
<summary>查看答案</summary>

根据 tjs.y 的优先级阶梯（从低到高：postfix if → comma → assign → cond → logical_or → ... → add_sub → mul_div → unary → priority → factor）：

```javascript
// 解析步骤：
// 1. 最外层是 expr → comma_expr "if" expr
//    左边是 comma_expr (a = b + c * d)
//    右边是 expr (e || f)

// 2. 左边 comma_expr → assign_expr → cond_expr "=" assign_expr
//    a = (b + c * d)

// 3. b + c * d：* 优先级高于 +
//    b + (c * d)

// 4. 右边 e || f：logical_or_expr

// 最终带括号结果：
(a = (b + (c * d))) if (e || f)

// 等价于：
if (e || f) {
    a = b + c * d;
}
```

关键点：postfix `if` 的优先级最低，所以整个赋值表达式都在 `if` 的左边。

</details>

### 题目 3：编写 Bison 规则

假设你要为 TJS2 添加一个 `foreach` 语法：`foreach (var item in collection) { ... }`。请写出对应的 Bison 产生式和语义动作（假设编译器已有 `EnterForeachCode`、`CreateForeachExprCode`、`ExitForeachCode` 方法）。

<details>
<summary>查看答案</summary>

```yacc
/* 首先，在 %token 部分添加新的关键字 */
%token T_FOREACH "foreach"

/* 在 statement 非终结符中添加 foreach */
statement
    : ...（原有规则）
    | foreach
;

/* foreach 产生式 */
foreach
    : "foreach" "("                     { cc->EnterForeachCode(); }
      "var" T_SYMBOL                    { cc->AddLocalVariable(
                                            lx->GetString($5)); }
      "in" expr ")"                     { cc->CreateForeachExprCode(
                                            lx->GetString($5), $8); }
      block_or_statement                { cc->ExitForeachCode(); }
;
```

设计思路：

1. `EnterForeachCode()` —— 记录循环起始位置，设置嵌套数据
2. `AddLocalVariable` —— 在当前块中注册迭代变量
3. `CreateForeachExprCode` —— 生成获取迭代器和条件判断的代码
4. `ExitForeachCode()` —— 生成循环体后的跳转代码和退出标签

还需要在词法分析器的保留字表中注册 `foreach` 关键字，确保它被识别为 `T_FOREACH` 而不是 `T_SYMBOL`。

</details>

## 下一步

[02-AST构建与节点类型](./02-AST构建与节点类型.md) —— 深入分析 `tTJSExprNode` 类的结构、`MakeNP0/1/2/3` 的工作原理、以及完整的 AST 节点类型体系和表达式编译过程。


