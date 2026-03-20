# Token 类型与识别

## 术语预览

| 术语 | 含义 |
|------|------|
| **Token 类型码** | 每个 Token 都有一个整数类型码（如 `T_PLUS = 43`、`T_IF = 258`），语法分析器根据此类型码决定如何处理该 Token。类型码分为两类：单字符运算符直接使用 ASCII 码值，多字符 Token 使用 258 以上的编号 |
| **TJS_MATCH_S 宏** | 用于在 GetToken 的 switch-case 中匹配多字符运算符的辅助宏，按照"最长匹配优先"原则依次尝试从最长的候选开始匹配 |
| **TJSStringMatch** | 字符串匹配辅助函数，比较当前扫描位置开始的字符序列是否与给定模式匹配。TJS_MATCH_S 宏的底层实现依赖此函数 |
| **保留字哈希表** | 存储所有 TJS2 保留字（关键字）的查找表，使用 `tTJSCustomObject`（TJS2 自身的对象系统哈希表）实现，在词法分析器初始化时构建，包含 50+ 个条目 |
| **T_SYMBOL** | 标识符 Token 类型，表示用户定义的名称（变量名、函数名、属性名等）。当词法分析器扫描到一个标识符但在保留字哈希表中找不到匹配时，返回此类型 |
| **T_CONSTVAL** | 常量值 Token 类型，表示字面量值。数值（`42`、`3.14`）、布尔值（`true`/`false`）、`null`、`NaN`、`Infinity` 等在词法阶段直接被转换为此类型 |
| **最长匹配（Maximal Munch）** | 词法分析的基本原则：当多个 Token 模式都能匹配当前位置时，选择匹配最多字符的那个。例如 `>>>` 优先于 `>>` 优先于 `>` |
| **运算符优先级分派** | GetToken 的 switch 语句中，每个 case 分支负责识别以该字符开头的所有可能运算符。例如 `case '>'` 分支处理 `>`、`>=`、`>>`、`>>=`、`>>>`、`>>>=` 六种情况 |
| **T_SWAP** | TJS2 独有的交换运算符 `<->`，用于原子地交换两个变量的值，无需临时变量 |
| **T_COMMA 映射** | TJS2 将 `=>` 在词法层面映射为 `T_COMMA`（逗号），使其成为纯粹的语法糖——在语法分析器看来，字典中的 `key => value` 和 `key, value` 是完全相同的 |

---

## 1. Token 类型码的编号体系

TJS2 的 Token 类型码遵循一个与 Bison 兼容的编号体系。理解这个体系是读懂 GetToken 代码的前提。

### 1.1 编号分区

```
类型码范围          用途                     示例
─────────────────  ──────────────────────  ─────────────────────
0                  EOF（文件结束）          源代码扫描完毕
1 ~ 255            单字符 Token             '(' = 40, ')' = 41,
                   （直接使用 ASCII 码值）  '{' = 123, '}' = 125,
                                           '~' = 126, '?' = 63,
                                           ':' = 58,  ';' = 59
258 ~ 400+         多字符 Token             T_IF = 258,
                   （Bison 定义的枚举值）   T_PLUS = 280,
                                           T_SYMBOL = 300,
                                           T_CONSTVAL = 301
```

**为什么从 258 开始？** Bison 约定用户定义的 Token 编号从 258 开始（0-255 保留给单字符 Token，256 是 `error`，257 是 `$undefined`）。TJS2 严格遵循了这个约定。

### 1.2 单字符 Token 的处理

对于只有一种含义的单字符运算符，GetToken 直接返回其 ASCII 码值：

```cpp
// GetToken 中的单字符 Token 处理
switch(*Current) {
    // ... 多字符运算符的 case 分支 ...

    default:
        // 不属于任何多字符运算符的开头
        // 直接返回字符的 ASCII 值作为 Token 类型
        {
            tjs_char c = *Current;
            TJSNext(&Current);  // 前进扫描指针

            // 以下字符作为单字符 Token 返回：
            // (  )  {  }  [  ]  ~  ?  :  ;  ,  #
            return (tjs_int)c;
        }
}
```

这意味着在 Bison 文法文件中，这些 Token 可以直接用字符字面量来引用：

```yacc
/* tjs.y 中的文法规则 */
func_call_expr
    : expr '(' arg_list ')'   /* ( 和 ) 直接匹配 ASCII 40 和 41 */
    ;
```

### 1.3 多字符 Token 的 Bison 定义

多字符 Token 在 Bison 文法文件 `tjs.y` 的 `%token` 声明中定义：

```yacc
/* tjs.y 中的 Token 声明（部分） */
%token T_COMMA           /* , 和 => */
%token T_EQUAL           /* = */
%token T_AMPERSANDEQUAL  /* &= */
%token T_VERTLINEEQUAL   /* |= */
%token T_CHEVRONEQUAL    /* ^= */
%token T_PLUSEQUAL        /* += */
%token T_MINUSEQUAL       /* -= */
%token T_PERCENTEQUAL     /* %= */
%token T_SLASHEQUAL       /* /= */
%token T_ASTERISKEQUAL    /* *= */
%token T_LOGICALOREQUAL   /* ||= */
%token T_LOGICALANDEQUAL  /* &&= */
%token T_RARITHSHIFTEQUAL /* >>>= */
%token T_LARITHSHIFTEQUAL /* <<= */
%token T_RBITSHIFTEQUAL   /* >>= */

%token T_IF T_ELSE T_WHILE T_DO T_FOR
%token T_BREAK T_CONTINUE T_RETURN
%token T_SWITCH T_CASE T_DEFAULT
%token T_CLASS T_FUNCTION T_PROPERTY
%token T_GETTER T_SETTER
%token T_VAR T_CONST
%token T_NEW T_DELETE T_TYPEOF T_INSTANCEOF
%token T_IN T_INCONTEXTOF
%token T_VOID T_WITH T_DEBUGGER
%token T_TRY T_CATCH T_THROW T_FINALLY
%token T_SYNCHRONIZED

%token T_SYMBOL T_CONSTVAL T_REGEXP
/* ... 更多 Token 定义 ... */
```

Bison 从 258 开始自动为每个 `%token` 分配整数值。词法分析器返回这些整数值，Bison 的状态机根据它们来驱动语法分析。

---

## 2. GetToken 的运算符分派架构

GetToken 的核心是一个约 400 行的 `switch(*Current)` 分派器，按照当前字符决定如何识别 Token。我们将其按类别分析。

### 2.1 TJS_MATCH_S 宏与最长匹配

TJS2 使用 `TJS_MATCH_S` 系列宏来实现多字符运算符的最长匹配。其核心思想是：**对每个起始字符，按照候选运算符从长到短的顺序依次尝试匹配**。

```cpp
// TJSStringMatch：底层字符串比较函数
static inline bool TJSStringMatch(
    const tjs_char *sc,     // 当前扫描位置
    const tjs_char *pattern, // 要匹配的模式
    bool skipsc = false     // 是否跳过第一个字符
)
{
    if(skipsc) sc++;  // 已经在 switch 中匹配了第一个字符
    while(*pattern) {
        if(*sc != *pattern) return false;
        sc++;
        pattern++;
    }
    return true;
}

// TJS_MATCH_S 宏的原理（概念演示，非精确源码）
// 对于以 > 开头的运算符：
case TJS_W('>'):
    // 从最长匹配开始尝试
    if(TJSStringMatch(Current, TJS_W(">>>="), true)) {
        // 匹配 >>>=（无符号右移赋值）
        Current += 4;  // 跳过 4 个字符
        return T_RARITHSHIFTEQUAL;
    }
    if(TJSStringMatch(Current, TJS_W(">>>"), true)) {
        // 匹配 >>>（无符号右移）
        Current += 3;
        return T_RARITHSHIFT;
    }
    if(TJSStringMatch(Current, TJS_W(">>="), true)) {
        // 匹配 >>=（右移赋值）
        Current += 3;
        return T_RBITSHIFTEQUAL;
    }
    if(TJSStringMatch(Current, TJS_W(">>"), true)) {
        // 匹配 >>（右移）
        Current += 2;
        return T_RBITSHIFT;
    }
    if(TJSStringMatch(Current, TJS_W(">="), true)) {
        // 匹配 >=（大于等于）
        Current += 2;
        return T_GTOREQUAL;
    }
    // 最后：单个 >
    Current += 1;
    return T_GT;
```

### 2.2 完整的运算符分派表

下表列出了 GetToken 中所有通过 switch-case 处理的运算符，按起始字符分组：

```
起始字符    候选运算符（从长到短）         Token 类型
────────  ──────────────────────────  ──────────────────────
>         >>>=  >>>  >>=  >>  >=  >  T_RARITHSHIFTEQUAL,
                                     T_RARITHSHIFT,
                                     T_RBITSHIFTEQUAL,
                                     T_RBITSHIFT,
                                     T_GTOREQUAL, T_GT

<         <<=  <->  <%   <<  <=  <  T_LARITHSHIFTEQUAL,
                                     T_SWAP, (见 2.3),
                                     T_LBITSHIFT,
                                     T_LTOREQUAL, T_LT

=         ===  ==  =>  =            T_DISCEQUAL,
                                     T_EQUALEQUAL,
                                     T_COMMA(!), T_EQUAL

!         !==  !=  !                T_DISCNOTEQUAL,
                                     T_NOTEQUAL, T_EXCRAMATION

&         &&=  &&  &=  &            T_LOGICALANDEQUAL,
                                     T_LOGICALAND,
                                     T_AMPERSANDEQUAL,
                                     T_AMPERSAND

|         ||=  ||  |=  |            T_LOGICALOREQUAL,
                                     T_LOGICALOR,
                                     T_VERTLINEEQUAL,
                                     T_VERTLINE

^         ^=  ^                      T_CHEVRONEQUAL,
                                     T_CHEVRON

.         ...  .数字  .              T_OMIT, (数值解析),
                                     T_DOT

+         ++  +=  +                  T_INCREMENT,
                                     T_PLUSEQUAL, T_PLUS

-         --  -=  -                  T_DECREMENT,
                                     T_MINUSEQUAL, T_MINUS

*         *=  *                      T_ASTERISKEQUAL,
                                     T_ASTERISK

/         //  /*  /=  /              (注释), (注释),
                                     T_SLASHEQUAL, T_SLASH

%         %=  %                      T_PERCENTEQUAL,
                                     T_PERCENT

'  "      (字符串解析)               T_CONSTVAL

@         @"  @set/@if/@endif        (嵌入式字符串),
                                     (预处理指令)

0-9       (数值解析)                 T_CONSTVAL

a-z A-Z _ (标识符/保留字)           T_SYMBOL 或对应关键字

其他      直接返回 ASCII 值          ( ) { } [ ] ~ ? : ; , #
```

### 2.3 特殊运算符：<-> 交换

`<->` 是 TJS2 独有的交换运算符，用于原子地交换两个变量的值：

```javascript
// TJS2 交换运算符
var a = 1, b = 2;
a <-> b;  // 现在 a == 2, b == 1

// 等价于但更高效：
// var temp = a; a = b; b = temp;
```

在词法分析器中，`<->` 被识别为 `T_SWAP`：

```cpp
case TJS_W('<'):
    // ...其他 < 开头的运算符...
    if(TJSStringMatch(Current, TJS_W("<->"), true)) {
        Current += 3;
        return T_SWAP;
    }
    // ...
```

这个运算符在 VM 层面有专门的字节码指令（`VM_CHGS`），可以直接交换两个寄存器的内容，避免了创建临时变量的开销。

### 2.4 特殊映射：=> 作为逗号

`=>` 运算符在词法层面被直接映射为 `T_COMMA`：

```cpp
case TJS_W('='):
    if(TJSStringMatch(Current, TJS_W("==="), true)) {
        Current += 3;
        return T_DISCEQUAL;      // 严格相等
    }
    if(TJSStringMatch(Current, TJS_W("=="), true)) {
        Current += 2;
        return T_EQUALEQUAL;     // 相等
    }
    if(TJSStringMatch(Current, TJS_W("=>"), true)) {
        Current += 2;
        return T_COMMA;          // => 映射为逗号！
    }
    Current++;
    return T_EQUAL;              // 赋值
```

这意味着以下两种写法在 TJS2 中完全等价：

```javascript
// 使用 => 的 Perl 风格字典语法
var dict = %[
    name  => "TJS2",
    ver   => 2,
    debug => true
];

// 使用逗号的传统语法（完全等价）
var dict = %[
    name,  "TJS2",
    ver,   2,
    debug, true
];
```

`=>` 的存在纯粹是为了可读性——它让字典的 key-value 关系更加直观，但在语法分析器看来它就是一个逗号。

### 2.5 <% 字节序列字面量

`<` 开头的运算符中还有一个特殊情况——`<%` 标记字节序列（Octet）字面量的开始：

```cpp
case TJS_W('<'):
    // ...
    if(TJSStringMatch(Current, TJS_W("<%"), true)) {
        // 进入字节序列解析模式
        // 格式：<% 十六进制字节对 %>
        return TJSParseOctet(Current, val);
    }
    // ...
```

字节序列字面量的语法是 `<% hex_bytes %>`：

```javascript
// 字节序列字面量
var header = <% 89 50 4E 47 0D 0A 1A 0A %>;  // PNG 文件头

// 每两个十六进制字符表示一个字节
// 空格可选，仅用于可读性
var data = <%FF D8 FF E0%>;  // JPEG 文件头
```

---

## 3. 注释处理

`/` 字符的处理是 GetToken 中最复杂的分支之一，因为它必须区分四种情况：行注释、块注释、除法赋值运算符、和普通除法运算符。

### 3.1 行注释 //

```cpp
case TJS_W('/'):
    // 检查是否是行注释
    if(*(Current + 1) == TJS_W('/')) {
        // 跳过到行尾
        while(*Current && *Current != TJS_W('\n'))
            TJSNext(&Current);
        // 不返回 Token——继续扫描下一个 Token
        // （通过 goto 或循环回到 switch 开头）
        continue;  // 或 goto retry;
    }
```

### 3.2 块注释 /* */

TJS2 的块注释支持**嵌套**——这与 C/C++ 不同：

```cpp
    // 检查是否是块注释
    if(*(Current + 1) == TJS_W('*')) {
        Current += 2;  // 跳过 /*

        // TJS2 支持嵌套的块注释！
        tjs_int level = 1;
        while(*Current && level > 0) {
            if(*Current == TJS_W('/') &&
               *(Current + 1) == TJS_W('*')) {
                level++;        // 嵌套层级 +1
                Current += 2;
            } else if(*Current == TJS_W('*') &&
                      *(Current + 1) == TJS_W('/')) {
                level--;        // 嵌套层级 -1
                Current += 2;
            } else {
                TJSNext(&Current);
            }
        }
        // 如果 level > 0 到达文件末尾，说明注释未关闭
        // TJS2 在此不报错（静默处理）
        continue;  // 继续扫描下一个 Token
    }
```

嵌套块注释在实际开发中非常有用——可以注释掉已包含注释的代码块：

```javascript
/* 外层注释
   /* 内层注释 — 在 C/C++ 中会导致错误 */
   这段代码被安全地注释掉了
*/
```

### 3.3 /= 和 / 运算符

```cpp
    // 不是注释，检查 /= 赋值运算符
    if(*(Current + 1) == TJS_W('=')) {
        Current += 2;
        return T_SLASHEQUAL;    // /=
    }

    // 普通除法运算符
    Current++;
    return T_SLASH;             // /
```

### 3.4 正则表达式的特殊处理

注意上面的代码中**没有**正则表达式的处理——正则表达式不在 `/` 的 case 分支中处理。如前一节所述，正则表达式的识别通过语法分析器回调 `SetStartOfRegExp()` 触发，在 GetToken 的 switch 之前就被拦截：

```cpp
tjs_int tTJSLexicalAnalyzer::GetToken(tTJSVariant &val)
{
    // ★ 正则表达式检查在 switch 之前
    if(RegularExpression) {
        RegularExpression = false;
        return TJSParseRegExp(Current, val);
    }

    TJSSkipSpace(&Current);
    PrevPos = Current;
    if(*Current == 0) return 0;

    switch(*Current) {
        case TJS_W('/'):
            // 到达这里时，已确定 / 不是正则表达式开始
            // ...
    }
}
```

---

## 4. . 运算符的三重含义

点号 `.` 在 TJS2 中有三种可能的含义，GetToken 需要根据后续字符来区分：

```cpp
case TJS_W('.'):
    // 情况 1：以 . 开头的浮点数（如 .5、.123）
    if(TJS_iswdigit(*(Current + 1))) {
        // 进入数值解析
        return TJSParseNumber(Current, val);
    }

    // 情况 2：... 省略标记（用于函数的可变参数）
    if(*(Current + 1) == TJS_W('.') &&
       *(Current + 2) == TJS_W('.')) {
        Current += 3;
        return T_OMIT;  // ...
    }

    // 情况 3：普通的成员访问运算符
    Current++;
    return T_DOT;  // .
```

```javascript
// 三种用法示例
var x = .5;              // 情况 1：浮点数 0.5
function f(a, ...) {}    // 情况 2：可变参数
var name = obj.name;     // 情况 3：属性访问
```

注意 `.5` 这样的写法在很多语言中都合法（C、JavaScript、Python），TJS2 也支持。词法分析器通过检查 `.` 后面是否紧跟数字来判断——如果是，则将 `.` 连同后续数字一起交给 `TJSParseNumber` 处理。

---

## 5. 保留字哈希表

### 5.1 哈希表的构建

TJS2 使用自身的对象系统 `tTJSCustomObject` 来构建保留字哈希表。这是一个有趣的"自举"设计——词法分析器用 TJS2 自己的数据结构来加速自身的工作。

```cpp
// 保留字哈希表的构建（在第一次创建词法分析器时执行）
static tTJSCustomObject *TJSReservedWordHash = NULL;

static void TJSInitReservedWordHash()
{
    if(TJSReservedWordHash) return;  // 已初始化

    TJSReservedWordHash = new tTJSCustomObject();

    // 注册所有保留字
    // RegisterNCM 将字符串名映射到整数 Token 类型码
    struct { const tjs_char *name; tjs_int token; } words[] = {
        // 控制流
        { TJS_W("if"),          T_IF },
        { TJS_W("else"),        T_ELSE },
        { TJS_W("while"),       T_WHILE },
        { TJS_W("do"),          T_DO },
        { TJS_W("for"),         T_FOR },
        { TJS_W("break"),       T_BREAK },
        { TJS_W("continue"),    T_CONTINUE },
        { TJS_W("return"),      T_RETURN },
        { TJS_W("switch"),      T_SWITCH },
        { TJS_W("case"),        T_CASE },
        { TJS_W("default"),     T_DEFAULT },
        { TJS_W("with"),        T_WITH },

        // 异常处理
        { TJS_W("try"),         T_TRY },
        { TJS_W("catch"),       T_CATCH },
        { TJS_W("throw"),       T_THROW },
        { TJS_W("finally"),     T_FINALLY },

        // 声明
        { TJS_W("var"),         T_VAR },
        { TJS_W("const"),       T_CONST },
        { TJS_W("function"),    T_FUNCTION },
        { TJS_W("class"),       T_CLASS },
        { TJS_W("property"),    T_PROPERTY },
        { TJS_W("getter"),      T_GETTER },
        { TJS_W("setter"),      T_SETTER },

        // 运算符关键字
        { TJS_W("typeof"),      T_TYPEOF },
        { TJS_W("instanceof"),  T_INSTANCEOF },
        { TJS_W("new"),         T_NEW },
        { TJS_W("delete"),      T_DELETE },
        { TJS_W("void"),        T_VOID },
        { TJS_W("in"),          T_IN },
        { TJS_W("incontextof"), T_INCONTEXTOF },

        // TJS2 特有关键字
        { TJS_W("invalidate"),  T_INVALIDATE },
        { TJS_W("isvalid"),     T_ISVALID },
        { TJS_W("synchronized"),T_SYNCHRONIZED },
        { TJS_W("global"),      T_GLOBAL },
        { TJS_W("this"),        T_THIS },
        { TJS_W("super"),       T_SUPER },
        { TJS_W("debugger"),    T_DEBUGGER },

        // 字面量关键字（特殊处理，见 5.3）
        { TJS_W("true"),        /* 特殊 */ },
        { TJS_W("false"),       /* 特殊 */ },
        { TJS_W("null"),        /* 特殊 */ },
        { TJS_W("NaN"),         /* 特殊 */ },
        { TJS_W("Infinity"),    /* 特殊 */ },

        // ... 50+ 个条目
    };

    for(auto &w : words) {
        // 使用 tTJSCustomObject 的哈希表存储
        // key = 保留字字符串
        // value = Token 类型码
        tTJSVariant val((tjs_int)w.token);
        TJSReservedWordHash->PropSet(
            TJS_MEMBERENSURE, w.name, NULL, &val, TJSReservedWordHash);
    }
}
```

### 5.2 保留字查找

当词法分析器扫描到一个标识符后，用哈希表查找它是否是保留字：

```cpp
// 标识符扫描和保留字查找
const tjs_char *start = Current;

// 扫描标识符字符：字母、数字、下划线
while(TJS_iswalpha(*Current) || TJS_iswdigit(*Current) ||
      *Current == TJS_W('_')) {
    TJSNext(&Current);
}

tjs_int len = (tjs_int)(Current - start);

// 在保留字哈希表中查找
tTJSVariant val;
ttstr name(start, len);
if(TJSReservedWordHash->PropGet(0, name.c_str(), NULL,
                                 &val, TJSReservedWordHash)
   == TJS_S_OK) {
    // 找到了——是保留字
    tjs_int token = (tjs_int)val;
    return token;  // 返回对应的 Token 类型
} else {
    // 没找到——是普通标识符
    val = tTJSVariant(name);
    return T_SYMBOL;
}
```

### 5.3 字面量关键字的特殊处理

有五个"关键字"在词法阶段不仅要返回特定的 Token 类型，还需要携带语义值。它们是 `true`、`false`、`null`、`NaN`、`Infinity`：

```cpp
// 在标识符扫描中的特殊处理
if(/* 匹配到保留字 */) {
    tjs_int token = (tjs_int)val;

    // 字面量关键字的特殊分支
    switch(token) {
    case T_FALSE:
        // false → T_CONSTVAL，值为 tTJSVariant(false)
        val = tTJSVariant((tjs_int)0);
        return T_CONSTVAL;

    case T_NULL:
        // null → T_CONSTVAL，值为 void（空变量）
        val.Clear();  // tTJSVariant 的默认值即为 void/null
        return T_CONSTVAL;

    case T_TRUE:
        // true → T_CONSTVAL，值为 tTJSVariant(true)
        val = tTJSVariant((tjs_int)1);
        return T_CONSTVAL;

    case T_NAN:
        // NaN → T_CONSTVAL，值为 IEEE 754 NaN
        {
            tjs_real d;
            *(tjs_uint64*)&d = TJS_IEEE_D_P_NaN;
            val = tTJSVariant(d);
            return T_CONSTVAL;
        }

    case T_INFINITY:
        // Infinity → T_CONSTVAL，值为正无穷
        {
            tjs_real d;
            *(tjs_uint64*)&d = TJS_IEEE_D_P_INF;
            val = tTJSVariant(d);
            return T_CONSTVAL;
        }

    default:
        // 普通保留字，不携带值
        return token;
    }
}
```

注意 `true` 被转换为整数 `1`，`false` 被转换为整数 `0`——TJS2 中的布尔值在内部就是整数。`NaN` 和 `Infinity` 通过直接操作 IEEE 754 位模式来创建，避免了浮点运算可能引起的平台差异。

### 5.4 为什么用 tTJSCustomObject 而不是 std::unordered_map？

这是一个值得探讨的设计决策。使用 TJS2 自身的对象系统哈希表有以下考量：

**实际原因**：TJS2 的代码库在 2000 年前后编写，当时 C++ 标准库中还没有 `std::unordered_map`（它在 C++11 中才被标准化）。`tTJSCustomObject` 是 TJS2 已有的高效哈希表实现，自然被复用。

**技术特点**：`tTJSCustomObject` 的哈希表为 `tjs_char*` 键做了专门优化——使用宽字符的哈希函数，支持大小写不敏感查找（虽然保留字查找是大小写敏感的），并且内存布局对短字符串友好。

```cpp
// tTJSCustomObject 的哈希表结构（简化）
class tTJSCustomObject {
    struct tTJSSymbolData {
        tjs_uint32 Hash;        // 预计算的哈希值
        ttstr Name;             // 符号名
        tTJSVariant Value;      // 关联值
        tTJSSymbolData *Next;   // 链表指针（解决冲突）
    };

    tTJSSymbolData **HashTable; // 哈希桶数组
    tjs_int HashSize;           // 桶数量（2 的幂）
    tjs_int Count;              // 当前条目数
};
```

---

## 6. 标识符扫描的完整流程

标识符扫描是 GetToken 中最常执行的路径之一（变量名、函数名、类名等都是标识符）。下面是完整的处理流程：

```cpp
// GetToken 的 default 分支（简化，展示标识符处理的完整逻辑）
default:
{
    // 检查是否是标识符起始字符
    if(TJS_iswalpha(*Current) || *Current == TJS_W('_')) {

        // 1. 记录起始位置
        const tjs_char *start = Current;

        // 2. 扫描所有标识符字符
        //    标识符字符 = 字母 + 数字 + 下划线
        while(TJS_iswalpha(*Current) ||
              TJS_iswdigit(*Current) ||
              *Current == TJS_W('_')) {
            TJSNext(&Current);
        }

        tjs_int len = (tjs_int)(Current - start);

        // 3. BareWord 检查（最优先）
        if(BareWord) {
            BareWord = false;
            // 无论是否是保留字，都返回 T_SYMBOL
            val = tTJSVariant(ttstr(start, len));
            return T_SYMBOL;
        }

        // 4. 保留字哈希表查找
        ttstr name(start, len);
        tTJSVariant hashVal;
        if(TJSReservedWordHash->PropGet(
            0, name.c_str(), NULL,
            &hashVal, TJSReservedWordHash) == TJS_S_OK) {

            tjs_int token = (tjs_int)hashVal;

            // 5. 字面量关键字特殊处理
            if(token == T_FALSE) {
                val = tTJSVariant((tjs_int)0);
                return T_CONSTVAL;
            }
            if(token == T_TRUE) {
                val = tTJSVariant((tjs_int)1);
                return T_CONSTVAL;
            }
            if(token == T_NULL) {
                val.Clear();
                return T_CONSTVAL;
            }
            if(token == T_NAN) {
                tjs_real d;
                *(tjs_uint64*)&d = TJS_IEEE_D_P_NaN;
                val = tTJSVariant(d);
                return T_CONSTVAL;
            }
            if(token == T_INFINITY) {
                tjs_real d;
                *(tjs_uint64*)&d = TJS_IEEE_D_P_INF;
                val = tTJSVariant(d);
                return T_CONSTVAL;
            }

            // 6. 普通保留字
            return token;
        }

        // 7. 不是保留字，返回 T_SYMBOL
        val = tTJSVariant(name);
        return T_SYMBOL;
    }

    // 不是标识符——单字符 Token
    tjs_char c = *Current;
    TJSNext(&Current);
    return (tjs_int)c;
}
```

### 6.1 BareWord 优先级

注意 BareWord 检查在保留字查找**之前**执行。这确保了当语法分析器设置了 `BareWord = true` 后，即使扫描到的标识符是保留字，也会被强制返回为 `T_SYMBOL`。

```javascript
// BareWord 的实际效果
obj.if        // BareWord=true → "if" 作为 T_SYMBOL 返回
obj.class     // BareWord=true → "class" 作为 T_SYMBOL 返回
obj.return    // BareWord=true → "return" 作为 T_SYMBOL 返回
obj.myFunc    // BareWord=true → "myFunc" 作为 T_SYMBOL 返回（本来就是）
```

BareWord 标志是**一次性的**——在第 3 步中 `BareWord = false` 立即将其重置。因此它精确地只影响 `.` 后面的下一个标识符。

### 6.2 标识符扫描的边界条件

标识符扫描有一些值得注意的边界条件：

```javascript
// 数字不能作为标识符的起始字符
var 123abc;    // 语法错误：123 被解析为数值，abc 是另一个 Token

// 但数字可以出现在标识符中间
var abc123 = 1; // 合法

// 下划线可以是完整的标识符
var _ = 42;     // 合法
var __ = 43;    // 合法

// Unicode 字符可以作为标识符
var 変数 = 100;  // 合法（日文）
var α = 3.14;   // 合法（希腊字母）

// 但某些 Unicode 字符虽然被接受，实际不应该用
var ★ = "star"; // 被接受（★ 的码值 > 0x100）
                // 但这是 TJS2 宽泛策略的副作用
```

---

## 7. @ 字符的多重角色

`@` 字符在 TJS2 中承担了两种完全不同的角色，GetToken 需要根据 `@` 后面的字符来判断：

### 7.1 嵌入式表达式字符串

当 `@` 后面紧跟 `"` 或 `'` 时，它标记嵌入式表达式字符串的开始：

```cpp
case TJS_W('@'):
    if(*(Current + 1) == TJS_W('"') ||
       *(Current + 1) == TJS_W('\'')) {
        // 设置嵌入式表达式状态机
        EmbeddableExpressionState = evsNextIsStringLiteral;
        Current++;  // 跳过 @，停在引号上
        // 后续 GetNext 会进入嵌入式表达式处理逻辑
        // 第一步：注入 T_LPARENTHESIS
        // ...
    }
```

### 7.2 预处理指令

当 `@` 后面跟的是标识符（`set`、`if`、`endif`）时，它标记预处理指令：

```cpp
    else {
        // 检查预处理指令
        Current++;  // 跳过 @
        if(TJSStringMatch(Current, TJS_W("set"))) {
            // @set(expr) — 设置预处理变量
            return ProcessPPStatement(pp_set);
        }
        if(TJSStringMatch(Current, TJS_W("if"))) {
            // @if(expr) — 条件编译开始
            return ProcessPPStatement(pp_if);
        }
        if(TJSStringMatch(Current, TJS_W("endif"))) {
            // @endif — 条件编译结束
            return ProcessPPStatement(pp_endif);
        }
        // 不是已知的预处理指令
        // 报错或忽略
    }
```

预处理指令在词法阶段就被处理——它们不会产生 Token 传递给语法分析器。`@if` 控制的代码块如果条件为假，其中的代码会被 `SkipUntil_endif()` 完全跳过（不产生任何 Token）。

### 7.3 两种角色的判断逻辑

```
@ 的后续字符     判断结果
─────────────   ──────────────────────
@"              嵌入式表达式字符串
@'              嵌入式表达式字符串
@set            预处理指令：设置变量
@if             预处理指令：条件编译
@endif          预处理指令：结束条件编译
@其他           错误或特殊处理
```

---

## 8. Token 流的实际输出示例

### 8.1 简单赋值语句

```javascript
var x = 42;
```

```
Token          类型码    语义值
─────────────  ────────  ──────
T_VAR          258       (无)
T_SYMBOL       300       "x"
T_EQUAL        ...       (无)
T_CONSTVAL     301       42
';'            59        (无)
```

### 8.2 复合运算符

```javascript
x >>>= y;
```

```
Token                   类型码    语义值
─────────────────────  ────────  ──────
T_SYMBOL               300       "x"
T_RARITHSHIFTEQUAL     ...       (无)
T_SYMBOL               300       "y"
';'                    59        (无)
```

### 8.3 属性访问与保留字

```javascript
obj.class = "wizard";
```

```
Token          类型码    语义值     说明
─────────────  ────────  ────────  ──────────────────
T_SYMBOL       300       "obj"     普通标识符
T_DOT          ...       (无)      . 运算符
                                   → 语法分析器调用 SetNextIsBareWord()
T_SYMBOL       300       "class"   BareWord! 强制为标识符
T_EQUAL        ...       (无)      赋值
T_CONSTVAL     301       "wizard"  字符串字面量
';'            59        (无)      分号
```

### 8.4 嵌套注释

```javascript
/* 外层
   /* 内层 */
   仍在注释中
*/
var done = true;
```

```
Token          类型码    语义值     说明
─────────────  ────────  ────────  ──────────────────
T_VAR          258       (无)      /* ... */ 整体被跳过
T_SYMBOL       300       "done"
T_EQUAL        ...       (无)
T_CONSTVAL     301       1         true → 整数 1
';'            59        (无)
```

### 8.5 交换运算符

```javascript
a <-> b;
```

```
Token          类型码    语义值
─────────────  ────────  ──────
T_SYMBOL       300       "a"
T_SWAP         ...       (无)
T_SYMBOL       300       "b"
';'            59        (无)
```

---

## 9. 与 JavaScript 词法分析的差异总结

TJS2 虽然在语法上与 JavaScript 高度相似，但词法层面有多个显著差异。理解这些差异对于从 JavaScript 背景转向 TJS2 开发的程序员尤为重要。

### 9.1 独有 Token 对比表

| 特性 | TJS2 | JavaScript (ES2020+) |
|------|------|---------------------|
| 交换运算符 | `<->` (T_SWAP) | 无（需要临时变量或解构赋值） |
| 字典分隔符 | `=>` (映射为逗号) | `=>` 是箭头函数 |
| 严格等于 | `===` (T_DISCEQUAL) | `===` |
| 省略标记 | `...` (T_OMIT) | `...` 是展开运算符 |
| 字节序列 | `<% ... %>` | 无 |
| 嵌入式字符串 | `@"...&expr;..."` | `` `...${expr}...` `` |
| `incontextof` | 关键字 | 无 |
| `invalidate` | 关键字 | 无 |
| `isvalid` | 关键字 | 无 |
| `property` | 关键字 | 无（ES5 后通过 defineProperty） |
| `getter`/`setter` | 关键字 | `get`/`set`（语法不同） |
| `synchronized` | 关键字 | 无 |
| `global` | 关键字 | `globalThis`（ES2020） |
| 嵌套块注释 | `/* /* */ */` 合法 | 不支持 |
| 逻辑赋值 | `&&=` `||=` | ES2021 才引入 |
| 条件编译 | `@if`/`@set`/`@endif` | 无 |

### 9.2 语义差异

即使 Token 名称相同，某些运算符的语义也有差异：

```javascript
// TJS2 的 === 是"判别相等"（discriminant equal）
// 它比较的是值的类型标签和内容，与 JavaScript 的 === 类似但不完全相同
var a = 0;
var b = "0";
a === b;  // TJS2: false（类型不同）
a == b;   // TJS2: true（类型转换后比较）

// TJS2 的 ... 是"省略"标记，用于可变参数
function f(a, b, ...) { }  // 等价于 JS 的 ...args
// 但 TJS2 的 ... 不能用于展开操作
```

---

## 10. 性能考量

### 10.1 GetToken 的时间复杂度

GetToken 的每次调用的平均时间复杂度为 O(k)，其中 k 是 Token 的字符长度：

- **空白跳过**：O(空白字符数量)
- **Switch 分派**：O(1)（编译器会优化为跳转表）
- **多字符运算符匹配**：O(运算符长度)，最长为 4 (`>>>=`)
- **标识符扫描**：O(标识符长度)
- **保留字查找**：O(1) 平均（哈希表查找）

### 10.2 保留字查找的优化空间

当前的保留字查找使用通用的 `tTJSCustomObject` 哈希表，这并非最优选择。更高效的方案包括：

```cpp
// 方案 1：完美哈希（Perfect Hash）
// 预计算一个无冲突的哈希函数，保证 O(1) 最坏情况
// gperf 工具可以自动生成

// 方案 2：Trie 树（前缀树）
// 对于保留字查找特别高效，因为可以在扫描标识符的同时进行查找
// 不需要先完成扫描再查表

// 方案 3：手写的 switch-case 树
// 按首字母分组，然后按长度/后续字符分支
// 编译器会将其优化为高效的跳转表
// V8 的词法分析器就使用了这种方式
```

但在实际的 KR2 引擎中，词法分析不是性能瓶颈——TJS2 脚本通常很短（几千行），词法分析只在加载时执行一次，后续运行的是编译后的字节码。因此使用通用哈希表的性能完全可接受。

### 10.3 TJSStringMatch 的优化

当前的 `TJSStringMatch` 是一个简单的逐字符比较函数。由于比较的模式都很短（2-4 个字符），编译器通常会将其内联并展开循环：

```cpp
// 编译器可能将 TJSStringMatch(Current, ">>>", true)
// 优化为：
if(Current[1] == '>' && Current[2] == '>' && Current[3] == '=') {
    // 匹配 >>>=
}
```

---

## 练习题与答案

### 练习 1：最长匹配的重要性

**问题**：如果词法分析器对 `>` 开头的运算符不使用最长匹配（即改为最短匹配），以下代码会被如何错误解析？

```javascript
x >>>= y;
```

<details>
<summary>点击查看答案</summary>

如果使用最短匹配，`>` 会被单独匹配为 `T_GT`。结果 Token 流为：

```
T_SYMBOL("x")  T_GT  T_GT  T_GT  T_EQUAL  T_SYMBOL("y")  ';'
```

即 `x > > > = y;`，这会导致语法错误——连续三个 `>` 和一个 `=` 不构成合法的 TJS2 语法。

正确的最长匹配结果应该是：

```
T_SYMBOL("x")  T_RARITHSHIFTEQUAL  T_SYMBOL("y")  ';'
```

即 `x >>>= y;`，表示"将 x 无符号右移并赋值"。

这就是为什么 GetToken 必须从最长候选开始尝试匹配的原因——最长匹配原则（Maximal Munch）是词法分析器的基本原则。

</details>

### 练习 2：=> 映射为逗号的影响

**问题**：由于 `=>` 在词法层面被映射为 `T_COMMA`，以下代码是否合法？如果合法，会产生什么结果？

```javascript
var arr = [1 => 2 => 3];
```

<details>
<summary>点击查看答案</summary>

这段代码**合法**。由于 `=>` 被映射为逗号，语法分析器看到的实际上是：

```javascript
var arr = [1, 2, 3];
```

这会创建一个包含三个元素 `[1, 2, 3]` 的数组。`=>` 在这里的行为与逗号完全相同。

这是 TJS2 将 `=>` 无条件映射为逗号的一个副作用——在字典上下文中 `=>` 很直观（`key => value`），但在数组上下文中使用它会产生令人困惑但仍然合法的代码。

反过来，字典中也可以使用逗号代替 `=>`：

```javascript
// 以下两种写法完全等价
var d1 = %[name => "TJS2", ver => 2];
var d2 = %[name, "TJS2", ver, 2];
```

</details>

### 练习 3：嵌套块注释

**问题**：以下代码在 TJS2 和 C++ 中分别会产生什么结果？

```
/* level 1 /* level 2 */ still commenting? */ var x = 1;
```

<details>
<summary>点击查看答案</summary>

**TJS2**：整行都是注释。TJS2 支持嵌套块注释，处理过程为：
1. 遇到第一个 `/*`，层级变为 1
2. 遇到第二个 `/*`，层级变为 2
3. 遇到第一个 `*/`，层级变为 1（仍在注释中）
4. 遇到第二个 `*/`，层级变为 0（注释结束）
5. 但此时已到达行尾，`var x = 1;` 不存在

实际上如果 `var x = 1;` 在第二个 `*/` 之后，它会被正常解析。完整分析：

```
/* level 1 /* level 2 */ still commenting? */  var x = 1;
^                                           ^  ^^^^^^^^^^
|          嵌套计数：                        |  正常代码
开始 (1)       +1 (2)       -1 (1)     -1 (0)
```

所以 `var x = 1;` 会被正常解析为代码。

**C++**：注释在第一个 `*/` 处结束（C++ 不支持嵌套块注释），所以：
```
/* level 1 /* level 2 */  still commenting? */ var x = 1;
^                      ^  ^^^^^^^^^^^^^^^^^  ^
|______ 注释 __________|  | 正常代码（语法错误）| 运算符 * 后跟 /
```
`still commenting? */` 会被当作代码解析，导致语法错误。

</details>

### 练习 4：Unicode 标识符的边界

**问题**：判断以下标识符在 TJS2 中是否合法，并解释原因：

```javascript
var café = 1;     // (A)
var 123abc = 2;   // (B)
var _0xFF = 3;    // (C)
var ☆star = 4;    // (D)
```

<details>
<summary>点击查看答案</summary>

- **(A) `café`** — **合法**。`c`、`a`、`f` 是 ASCII 字母（`TJS_iswalpha` 返回 true），`é`（U+00E9）的码值 0xE9 小于 0x100，所以 `0xE9 & 0xFF00 == 0`。但等等——`é` 的码值是 0x00E9，`0x00E9 & 0xFF00 = 0x0000`，这意味着 `TJS_iswalpha('é')` 返回 **false**！

  所以 `café` 实际上**不合法**——`é` 会被视为非标识符字符，词法分析器会在 `caf` 处截断标识符，然后尝试将 `é` 作为下一个 Token。由于 `é` 不是任何合法 Token 的开始，会导致错误。

  修正：使用全角或码值 >= 0x100 的字符。

- **(B) `123abc`** — **不合法**。`1` 是数字，不满足 `TJS_iswalpha` 也不是 `_`，所以不会进入标识符扫描。`123` 会被解析为数值 Token，`abc` 会被解析为另一个标识符 Token。`var 123` 在语法上不合法。

- **(C) `_0xFF`** — **合法**。以 `_` 开头，满足标识符起始条件。后续的 `0`、`x`、`F`、`F` 都满足 `TJS_iswalpha` 或 `TJS_iswdigit`。

- **(D) `☆star`** — **合法**。`☆`（U+2606）的码值 0x2606，`0x2606 & 0xFF00 = 0x2600 ≠ 0`，所以 `TJS_iswalpha('☆')` 返回 true。后续的 `s`、`t`、`a`、`r` 都是 ASCII 字母。虽然 `☆` 在 Unicode 标准中是"杂项符号"而非字母，但 TJS2 的宽泛策略接受它。

</details>

### 练习 5：预处理指令与普通代码的交互

**问题**：以下代码在 `debug_mode` 为 false 时，词法分析器会产生哪些 Token？

```javascript
var x = 1;
@if(debug_mode)
var debug_info = "enabled";
@endif
var y = 2;
```

<details>
<summary>点击查看答案</summary>

当 `debug_mode` 为 false 时，`@if` 到 `@endif` 之间的代码会被 `SkipUntil_endif()` 完全跳过。词法分析器产生的 Token 流等价于：

```javascript
var x = 1;
var y = 2;
```

Token 序列：
```
T_VAR  T_SYMBOL("x")  T_EQUAL  T_CONSTVAL(1)  ';'
T_VAR  T_SYMBOL("y")  T_EQUAL  T_CONSTVAL(2)  ';'
```

`@if`、`@endif` 和中间的 `var debug_info = "enabled";` 都不会产生任何 Token。预处理指令在词法层面就被处理了——语法分析器完全不知道它们的存在。

注意：`SkipUntil_endif()` 的实现也会正确处理嵌套的 `@if`/`@endif`，确保跳过正确的代码范围。

</details>
