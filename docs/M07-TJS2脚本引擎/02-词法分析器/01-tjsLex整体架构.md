# tjsLex 整体架构

## 术语预览

| 术语 | 含义 |
|------|------|
| **词法分析器（Lexer）** | 编译器前端的第一阶段，将源代码字符流分解为有意义的记号（Token）序列，供语法分析器消费 |
| **Token** | 词法分析的最小输出单元，包含类型标识（如 `T_PLUS`、`T_IF`、`T_SYMBOL`）和可选的语义值（如整数 42、字符串 "hello"） |
| **tTJSLexicalAnalyzer** | KR2 中 TJS2 词法分析器的核心类，定义在 `tjsLex.h`，实现在 `tjsLex.cpp`，负责将 TJS2 源代码转换为 Token 流 |
| **GetNext** | 词法分析器的公共入口方法，语法分析器通过调用此方法获取下一个 Token。GetNext 内部调用 GetToken，并在此基础上实现预处理注入、DicFunc 快捷语法、嵌入式表达式等高层逻辑 |
| **GetToken** | 底层的 Token 扫描方法，负责从字符流中直接识别并返回一个原始 Token，不涉及高层逻辑转换 |
| **PreProcess** | 预处理阶段，在第一次调用 GetNext 时触发，将源代码中的 CRLF 行尾统一为 LF，并用 `TJS_SKIP_CODE` 哨兵值标记被消除的字符位置 |
| **TJS_SKIP_CODE** | 值为 `(tjs_char)~0`（即 0xFFFF）的特殊哨兵字符，用于标记预处理阶段删除的字符位置。所有扫描函数在遇到此值时跳过，不产生 Token |
| **嵌入式表达式（Embeddable Expression）** | TJS2 的特殊字符串语法 `@"text &expr; more ${expr} end"`，允许在字符串字面量中嵌入任意表达式。词法分析器通过状态机将其分解为字符串拼接的 Token 序列 |
| **DicFunc 快捷语法** | 一种语法糖，允许 `%[key => val]` 和 `[1,2,3]` 等字面量出现在表达式位置。词法分析器将其包装为 `(function(){return ...;})()` 的 Token 序列 |
| **RetValDeque** | `std::deque<tTJSTokenValue>` 类型的 Token 注入队列，用于 DicFunc 和嵌入式表达式功能向 Token 流中插入合成 Token |
| **PutValue** | 向 Values 容器追加语义值的方法，返回该值在容器中的索引。语法分析器通过此索引在解析完成后取回 Token 携带的值 |
| **tTJSScriptBlock** | 表示一个编译单元（一段脚本源码）的类，是词法分析器的宿主。词法分析器作为 tTJSScriptBlock 的成员被创建和使用 |

---

## 1. 词法分析器在 TJS2 编译流水线中的位置

TJS2 编译器采用经典的三阶段流水线架构：词法分析 → 语法分析 → 字节码生成。词法分析器（Lexer）是这条流水线的第一个环节，它接收原始的源代码字符串，输出结构化的 Token 序列。

```
源代码字符串 (tjs_char*)
        │
        ▼
┌─────────────────────────────┐
│   tTJSLexicalAnalyzer       │
│                             │
│  PreProcess()               │  ← 第一步：CRLF 统一、TJS_SKIP_CODE 标记
│       │                     │
│       ▼                     │
│  GetNext()                  │  ← 公共入口（语法分析器调用）
│    ├─ DicFunc 注入          │     高层逻辑：快捷语法包装
│    ├─ 嵌入式表达式状态机    │     高层逻辑：@"..." 分解
│    └─ GetToken()            │  ← 底层扫描（字符→Token）
│         ├─ 运算符匹配       │
│         ├─ 字符串解析       │
│         ├─ 数值解析         │
│         ├─ 标识符/关键字    │
│         └─ 预处理指令       │
│                             │
│  PutValue() → Values[]     │  ← 语义值存储
└─────────────────────────────┘
        │
        ▼ Token 流 (token_type, value_index)
┌─────────────────────────────┐
│   Bison 语法分析器 (tjs.y)  │
│   cc->MakeNP0/1/2/3()      │  ← 构建 AST
└─────────────────────────────┘
        │
        ▼ AST 树
┌─────────────────────────────┐
│   tjsInterCodeGen           │
│   字节码编译器              │  ← 生成 VM 指令
└─────────────────────────────┘
```

这个架构中的关键设计决策是：**词法分析器和语法分析器之间存在双向通信**。通常编译器中词法分析器是被动的——语法分析器调用 `GetNext()` 拉取下一个 Token。但 TJS2 中，语法分析器还会反过来通知词法分析器两种特殊情况：

1. **`SetStartOfRegExp()`**：当语法分析器判断当前 `/` 应该是正则表达式的开始（而不是除法运算符）时，调用此方法让词法分析器回退并重新解析为正则表达式字面量。

2. **`SetNextIsBareWord()`**：当语法分析器在 `.` 运算符之后时，调用此方法告诉词法分析器下一个标识符应当被当作裸词处理——即使它是保留字（如 `if`、`for`），也返回 `T_SYMBOL` 而不是对应的关键字 Token。

这种双向通信是 TJS2 词法分析器区别于教科书实现的重要特征，我们会在后续小节中详细分析。

---

## 2. 核心类 tTJSLexicalAnalyzer 的结构

`tTJSLexicalAnalyzer` 类定义在 `tjsLex.h` 中，约 145 行。下面是完整的类声明及每个成员的详细解释：

```cpp
// 文件：tjsLex.h
// TJS2 词法分析器核心类

class tTJSLexicalAnalyzer
{
    // ===== 源代码管理 =====
    tjs_char *Script;       // 源代码副本（构造时复制，析构时释放）
    tjs_char *Current;      // 当前扫描位置指针
    tjs_char *PrevPos;      // 上一个 Token 的起始位置（用于 regex 回退）

    // ===== 宿主引用 =====
    tTJSScriptBlock *Block; // 所属的编译单元
    tTJSPPExprParser *PPExprParser; // 预处理表达式解析器（@if/@set 用）

    // ===== 行号追踪 =====
    tjs_int PrevToken;      // 上一个返回的 Token 类型

    // ===== 语义值管理 =====
    std::vector<tTJSVariant> Values; // Token 携带的值容器

    // ===== 控制标志 =====
    bool RegularExpression; // true 时下次 GetToken 解析正则表达式
    bool BareWord;          // true 时下次标识符强制返回 T_SYMBOL
    bool ExprMode;          // true 表示以"表达式模式"编译（非完整脚本）
    bool ResultNeeded;      // 表达式模式下是否需要返回值
    bool DicFunc;           // true 表示启用 DicFunc 快捷语法

    // ===== 预处理状态 =====
    bool IfLevel;           // @if 嵌套层级（注意：源码中类型为 bool，
                            // 但实际作为计数器使用）
    bool ProcessPPExpression; // 是否正在处理预处理表达式

    // ===== Token 注入队列 =====
    std::deque<tTJSTokenValue> RetValDeque;
    // DicFunc 和嵌入式表达式产生的合成 Token 队列
    // GetNext 优先从此队列取 Token，队列为空时才调用 GetToken

    // ===== 嵌入式表达式状态机 =====
    enum tEmbeddableExpressionState {
        evsStart,                  // 初始状态
        evsNextIsStringLiteral,    // 下一段是字符串字面量
        evsNextIsExpression        // 下一段是嵌入的表达式
    };
    tEmbeddableExpressionState EmbeddableExpressionState;
    // 当前状态机状态，控制 @"..." 字符串的分解过程

public:
    // ===== 公共接口 =====
    tTJSLexicalAnalyzer(
        tTJSScriptBlock *block,     // 所属编译单元
        const tjs_char *script,     // 源代码
        bool exprmode,              // 是否为表达式模式
        bool resultneeded           // 是否需要返回值
    );
    ~tTJSLexicalAnalyzer();

    void PreProcess();      // 预处理源代码
    tjs_int GetNext(tTJSVariant &val); // 获取下一个 Token
    tjs_int GetToken(tTJSVariant &val); // 底层 Token 扫描

    void SetStartOfRegExp();   // 语法分析器通知：/ 是正则开始
    void SetNextIsBareWord();  // 语法分析器通知：下一词是裸词

    tjs_int PutValue(const tTJSVariant &val); // 存储语义值
    void Free();               // 释放 Values 容器内存
};
```

### 2.1 内存管理策略

词法分析器的内存管理遵循一个简洁的原则：**构造时复制，析构时释放**。

```cpp
// 构造函数（简化版，展示核心逻辑）
tTJSLexicalAnalyzer::tTJSLexicalAnalyzer(
    tTJSScriptBlock *block,
    const tjs_char *script,
    bool exprmode,
    bool resultneeded)
    : Block(block)
    , ExprMode(exprmode)
    , ResultNeeded(resultneeded)
{
    // 1. 复制源代码（加 2 个字符余量用于 sentinel）
    tjs_int len = TJS_strlen(script);
    Script = new tjs_char[len + 2];
    TJS_strcpy(Script, script);

    // 2. 表达式模式：追加分号确保语法完整
    if(ExprMode) {
        Script[len] = TJS_W(';');
        Script[len + 1] = 0;
    }

    // 3. 初始化扫描指针
    Current = Script;
    PrevPos = Script;

    // 4. 处理 shebang 行（#! → //）
    if(Script[0] == TJS_W('#') && Script[1] == TJS_W('!')) {
        Script[0] = TJS_W('/');
        Script[1] = TJS_W('/');
    }

    // 5. DicFunc 检测
    //    源码中使用了一个巧妙的"字符标记"技巧
    //    检查全局标记 tjsEnableDicFuncQuickHack_mark
    //    如果第 4 个字节被设置，则启用 DicFunc 模式
    DicFunc = /* dicfunc quick-hack detection */;

    // 6. 初始化其他状态
    RegularExpression = false;
    BareWord = false;
    PrevToken = -1;
    IfLevel = 0;
    PPExprParser = NULL;
    EmbeddableExpressionState = evsStart;
}
```

几个值得注意的细节：

**表达式模式追加分号**：当 TJS2 需要求值一个表达式（而非执行一段完整脚本）时，`ExprMode` 为 true。构造函数在源代码末尾追加一个 `;`，这样后续的 GetNext 可以注入 `T_RETURN` 语句，将表达式结果作为脚本的返回值。这个机制用于 `Eval()` 函数和调试器的表达式求值。

**Shebang 处理**：如果脚本以 `#!` 开头（Unix shell 脚本风格），词法分析器将其就地替换为 `//`，使其变成 C 风格行注释。这使得 TJS2 脚本可以直接作为 Unix 可执行脚本运行（理论上）。实际在 KR2 引擎中很少用到，但这体现了 TJS2 设计时考虑了跨平台脚本执行的场景。

**DicFunc 检测**：这是一个值得单独讨论的"快捷语法"（quick-hack）机制。TJS2 允许在表达式位置直接使用 `%[key => val]`（字典字面量）和 `[1,2,3]`（数组字面量），但语法分析器的 LALR(1) 文法无法直接处理这种语法（会导致 reduce/reduce 冲突）。解决方案是在词法层面将它们包装为立即调用的匿名函数：

```javascript
// 源代码：
var d = %[name => "TJS2", version => 2];

// 词法分析器注入后的等效 Token 流：
var d = (function() { return %[name => "TJS2", version => 2]; })();
```

这个包装通过 `RetValDeque` 队列注入额外的 Token 来实现，我们在第 5 节会详细分析。

---

## 3. PreProcess 预处理阶段

`PreProcess()` 是词法分析的第零步——在任何 Token 扫描之前对源代码字符串进行就地修改。它在 GetNext 的第一次调用时被自动触发（通过检查 `PrevToken == -1`）。

### 3.1 CRLF 统一

预处理的主要任务是将 Windows 风格的 `\r\n` 行尾统一为 `\n`：

```cpp
void tTJSLexicalAnalyzer::PreProcess()
{
    // 遍历整个源代码
    tjs_char *p = Script;
    while(*p) {
        if(*p == TJS_W('\r')) {
            // 情况 1：\r\n → 保留 \n，用 TJS_SKIP_CODE 替换 \r
            if(*(p + 1) == TJS_W('\n')) {
                *p = (tjs_char)TJS_SKIP_CODE;  // 0xFFFF 哨兵
                p++;
                // \n 保持不变
            }
            // 情况 2：单独的 \r → 替换为 \n
            else {
                *p = TJS_W('\n');
            }
        }
        p++;
    }
}
```

### 3.2 TJS_SKIP_CODE 哨兵机制

`TJS_SKIP_CODE` 的值是 `(tjs_char)~0`，即 `0xFFFF`。这是一个巧妙的设计——在 Unicode 的 BMP（基本多文种平面）中，`0xFFFF` 是一个永久保留的非字符（noncharacter），不会出现在合法的源代码中。因此用它作为"已删除"标记是安全的。

所有的字符扫描函数都必须跳过 `TJS_SKIP_CODE`：

```cpp
// TJSNext：前进到下一个有效字符
static inline void TJSNext(const tjs_char **ptr)
{
    (*ptr)++;
    // 跳过所有连续的 TJS_SKIP_CODE
    while(**ptr == TJS_SKIP_CODE) (*ptr)++;
}

// TJSSkipSpace：跳过空白字符（空格、制表符、TJS_SKIP_CODE）
static inline void TJSSkipSpace(const tjs_char **ptr)
{
    // TJS_iswspace 的实现：
    //   ch == TJS_W(' ')  || ch == TJS_W('\t') ||
    //   ch == TJS_SKIP_CODE
    while(TJS_iswspace(**ptr)) (*ptr)++;
}
```

**为什么不直接删除字符？** 因为就地删除（移动后续字符）的时间复杂度是 O(n²)，而标记法只需 O(n)。同时，保持字符串长度不变意味着所有行号计算（基于 `\n` 计数）仍然正确——如果删除了字符，行号偏移就会出错。

### 3.3 预处理与 Token 扫描的时序关系

```
构造函数
    │
    ▼ PrevToken = -1
第一次 GetNext() 调用
    │
    ├─ 检测到 PrevToken == -1
    │
    ├─ 调用 PreProcess()
    │   └─ CRLF → TJS_SKIP_CODE + \n
    │
    ├─ 如果 ExprMode：
    │   └─ 向 RetValDeque 注入 T_RETURN Token
    │       （使表达式结果成为脚本返回值）
    │
    ├─ PrevToken = 0（标记预处理已完成）
    │
    └─ 继续正常的 GetToken() 流程
        │
        ▼
后续 GetNext() 调用
    │
    └─ PrevToken != -1，跳过预处理，直接进入 Token 扫描
```

---

## 4. GetNext 与 GetToken 的双层架构

TJS2 词法分析器最核心的设计是 **GetNext/GetToken 双层架构**。GetToken 是底层的字符级扫描引擎，负责从字符流中识别单个 Token；GetNext 是高层的调度器，在 GetToken 之上叠加了三个重要的增强功能。

### 4.1 GetToken：底层扫描引擎

GetToken 约 580 行（tjsLex.cpp 的 1040-1620 行），是一个巨大的 switch-case 分派器。它根据当前字符决定如何识别 Token：

```cpp
tjs_int tTJSLexicalAnalyzer::GetToken(tTJSVariant &val)
{
    // 跳过空白
    TJSSkipSpace(&Current);

    // 记录当前位置（用于 regex 回退）
    PrevPos = Current;

    // 检查是否到达源代码末尾
    if(*Current == 0) return 0;  // EOF

    // 主分派 switch
    switch(*Current) {
        case TJS_W('>'):
            // >>>, >>=, >>, >=, > 的多字符匹配
            // ...
        case TJS_W('<'):
            // <<=, <=, <->, <%, <<, < 的多字符匹配
            // ...
        case TJS_W('='):
            // ===, ==, => (逗号), = 的多字符匹配
            // ...
        case TJS_W('!'):
            // !==, !=, ! 的多字符匹配
            // ...
        case TJS_W('&'):
            // &&=, &&, &=, & 的多字符匹配
            // ...
        case TJS_W('|'):
            // ||=, ||, |=, | 的多字符匹配
            // ...
        // ... 更多运算符
        case TJS_W('\''):
        case TJS_W('"'):
            // 字符串字面量
            return TJSParseString(Current, val, ...);
        case TJS_W('@'):
            // 嵌入式表达式字符串或预处理指令
            // ...
        case TJS_W('0'): case TJS_W('1'): /* ... */ case TJS_W('9'):
            // 数值字面量
            return TJSParseNumber(Current, val, ...);
        case TJS_W('.'):
            // . 运算符，或 .5 这样的浮点数，或 ... 省略标记
            // ...
        case TJS_W('/'):
            // 注释（// 和 /* */）、/= 运算符、或 / 除法
            // ...
        default:
            // 标识符或保留字
            if(TJS_iswalpha(*Current) || *Current == TJS_W('_')) {
                // 扫描标识符
                // 查保留字哈希表
                // BareWord 标志检查
                // ...
            }
            // 单字符运算符：( ) { } [ ] ~ ? : ; ,
            // ...
    }
}
```

GetToken 有两个重要的特性：

1. **它不处理上下文相关的语法**——`/` 是除法还是正则的判断、`.` 后的关键字是否应当作标识符等问题，都由语法分析器通过回调通知。

2. **它不注入额外的 Token**——DicFunc 包装和嵌入式表达式分解等需要生成多个 Token 的工作，都由 GetNext 完成。

### 4.2 GetNext：高层调度器

GetNext 约 280 行（tjsLex.cpp 的 1660-1895 行），是语法分析器唯一直接调用的接口。它在 GetToken 之上实现了三个增强层：

```
GetNext() 调用流程
    │
    ├─ [1] 首次调用检查
    │   └─ PrevToken == -1?
    │       ├─ PreProcess()
    │       └─ ExprMode? → 注入 T_RETURN
    │
    ├─ [2] RetValDeque 优先队列
    │   └─ 队列非空？
    │       ├─ 是 → 直接取队首 Token 返回
    │       └─ 否 → 继续
    │
    ├─ [3] DicFunc 快捷语法检测
    │   └─ DicFunc 模式 && 当前是 %[ 或 [ ？
    │       ├─ 是 → 注入 function/return 包装 Token
    │       └─ 否 → 继续
    │
    ├─ [4] 嵌入式表达式状态机
    │   └─ EmbeddableExpressionState != evsStart?
    │       ├─ 是 → 按状态机逻辑处理
    │       └─ 否 → 继续
    │
    └─ [5] 正常路径：调用 GetToken()
        │
        └─ 记录 PrevToken，返回结果
```

下面我们依次分析三个增强层。

---

## 5. 增强层一：DicFunc 快捷语法

### 5.1 问题背景

TJS2 允许在表达式位置直接写字典和数组字面量：

```javascript
// 字典字面量
var config = %[
    width  => 800,
    height => 600,
    title  => "Game"
];

// 数组字面量
var colors = ["red", "green", "blue"];
```

但是，Bison 生成的 LALR(1) 语法分析器在处理 `[` 时会产生歧义——它可能是数组字面量的开始，也可能是属性访问运算符（`obj[key]`）。这个歧义无法在语法层面简单解决。

### 5.2 解决方案：词法层包装

TJS2 的解决方案是在词法层面将字典/数组字面量包装为立即调用函数表达式（IIFE）：

```javascript
// 原始代码：
var config = %[width => 800];

// 词法分析器输出的等效 Token 流：
var config = (function() { return %[width => 800]; })();

// 原始代码：
var colors = [1, 2, 3];

// 词法分析器输出的等效 Token 流：
var colors = (function() { return [1, 2, 3]; })();
```

通过将字面量放进函数体中，消除了语法层面的歧义——`return` 之后的 `%[` 或 `[` 只能是字面量，不可能是属性访问。

### 5.3 实现细节

```cpp
// GetNext 中的 DicFunc 处理逻辑（简化版）
tjs_int tTJSLexicalAnalyzer::GetNext(tTJSVariant &val)
{
    // ... 首次调用检查、RetValDeque 检查 ...

    if(DicFunc) {
        // 跳过空白，检查当前字符
        TJSSkipSpace(&Current);

        bool is_dict = (*Current == TJS_W('%') &&
                        *(Current+1) == TJS_W('['));
        bool is_array = (*Current == TJS_W('['));

        if(is_dict || is_array) {
            // 向 RetValDeque 注入以下 Token 序列：
            // (function { return
            //   ^^^^^^^^^^^^^^^^^^^ 前缀 Token

            tTJSTokenValue v;

            // Token: T_LPARENTHESIS  → (
            v.token = T_LPARENTHESIS;
            RetValDeque.push_back(v);

            // Token: T_FUNCTION      → function
            v.token = T_FUNCTION;
            RetValDeque.push_back(v);

            // Token: T_LBRACE        → {
            v.token = T_LBRACE;
            RetValDeque.push_back(v);

            // Token: T_RETURN        → return
            v.token = T_RETURN;
            RetValDeque.push_back(v);

            // 此时不注入 %[ 或 [ 本身——
            // 它们会在后续 GetToken 调用中自然被扫描

            // 但需要在扫描到 ] 之后注入后缀：
            // ; } ) ()
            // 这通过在 GetToken 返回 ] 时检测来实现

            // 从队列取第一个 Token 返回
            val = RetValDeque.front().value;
            tjs_int token = RetValDeque.front().token;
            RetValDeque.pop_front();
            return token;
        }
    }

    // 正常路径
    return GetToken(val);
}
```

实际实现中，后缀 Token（`; } ) ( )`）的注入时机更加复杂——需要跟踪 `[` 和 `]` 的嵌套层级，只在最外层的 `]` 之后才注入后缀。这通过一个嵌套计数器实现。

### 5.4 DicFunc 启用条件

DicFunc 模式不是默认启用的。它通过一个全局标记 `tjsEnableDicFuncQuickHack_mark` 控制：

```cpp
// 全局标记：4 字节的字符数组
// 第 4 字节（mark[3]）用于标记是否启用 DicFunc
extern tjs_char tjsEnableDicFuncQuickHack_mark[4];

// 在构造函数中检测：
DicFunc = (tjsEnableDicFuncQuickHack_mark[3] != 0);
```

这个设计使用了一个有趣的"字符标记"技巧——通过修改一个全局字符数组的特定位置来开关功能，避免了引入额外的全局变量或配置机制。KAG 在初始化时会设置这个标记，使得 TJS2 脚本中可以直接使用 `%[...]` 和 `[...]` 字面量。

---

## 6. 增强层二：嵌入式表达式状态机

### 6.1 嵌入式表达式语法

TJS2 支持一种强大的字符串模板语法，称为"嵌入式表达式"（embeddable expression）：

```javascript
// 使用 @ 前缀开启嵌入式表达式字符串
var msg = @"Player ${name} has &score; points";

// 等价于：
var msg = "Player " + string(name) + " has " + string(score) + " points";
```

嵌入语法有两种形式：
- `&expr;`：以 `&` 开始，以 `;` 结束，中间是一个表达式
- `${expr}`：以 `${` 开始，以 `}` 结束，中间是一个表达式（类似 ES6 模板字面量）

### 6.2 状态机设计

词法分析器用一个三状态的有限状态机来分解嵌入式表达式字符串：

```
                    ┌──────────────┐
                    │   evsStart   │ ← 初始状态
                    └──────┬───────┘
                           │ 遇到 @" 
                           ▼
              ┌────────────────────────┐
         ┌───▶│ evsNextIsStringLiteral │◀───┐
         │    └────────────┬───────────┘    │
         │                 │                │
         │    遇到 & 或 ${  │    遇到 ; 或 } │
         │                 ▼                │
         │    ┌────────────────────────┐    │
         │    │  evsNextIsExpression   │────┘
         │    └────────────────────────┘
         │                 │
         │    遇到结束 "    │
         └─────────────────┘
```

### 6.3 Token 流分解过程

以 `@"Hello &name; world"` 为例，词法分析器将其分解为以下 Token 序列：

```
原始字符串：@"Hello &name; world"

分解后的 Token 流：
  T_LPARENTHESIS    →  (
  T_CONSTVAL        →  "Hello "      ← 第一段纯文本
  T_PLUS            →  +
  T_SYMBOL          →  string        ← string() 类型转换
  T_LPARENTHESIS    →  (
  T_SYMBOL          →  name          ← 嵌入的表达式
  T_RPARENTHESIS    →  )
  T_PLUS            →  +
  T_CONSTVAL        →  " world"      ← 第二段纯文本
  T_RPARENTHESIS    →  )
```

整个表达式被包装在括号中，每段嵌入的表达式都用 `string()` 包裹以确保类型安全。

### 6.4 状态机实现

```cpp
// GetNext 中的嵌入式表达式处理（简化版）
tjs_int tTJSLexicalAnalyzer::GetNext(tTJSVariant &val)
{
    // ... 前置检查 ...

    switch(EmbeddableExpressionState) {
    case evsStart:
        // 正常模式，不做特殊处理
        break;

    case evsNextIsStringLiteral:
    {
        // 扫描到下一个 & 或 ${ 或结束 " 为止
        // 将这段纯文本作为字符串常量输出

        tjs_char *start = Current;
        // 扫描直到遇到 & 或 ${ 或 "
        while(*Current != 0) {
            if(*Current == TJS_W('&')) {
                // 切换到表达式模式
                EmbeddableExpressionState = evsNextIsExpression;
                break;
            }
            if(*Current == TJS_W('$') &&
               *(Current+1) == TJS_W('{')) {
                EmbeddableExpressionState = evsNextIsExpression;
                break;
            }
            if(*Current == TJS_W('"')) {
                // 字符串结束
                EmbeddableExpressionState = evsStart;
                break;
            }
            Current++;
        }

        // 将 start..Current 之间的文本作为字符串 Token
        // 同时注入 + string( 的 Token 前缀
        // ...

        break;
    }

    case evsNextIsExpression:
    {
        // 跳过 & 或 ${
        // 正常调用 GetToken 获取表达式 Token
        // 遇到 ; 或 } 时切回 evsNextIsStringLiteral

        tjs_int token = GetToken(val);

        if(token == T_SEMICOLON ||   // & 模式的结束符
           token == T_RBRACE) {       // ${ 模式的结束符
            // 注入 ) + 的 Token（关闭 string() 调用）
            EmbeddableExpressionState = evsNextIsStringLiteral;
            // ...
        }

        return token;
    }
    }

    // 正常 GetToken 路径
    return GetToken(val);
}
```

### 6.5 复杂嵌入式表达式示例

嵌入式表达式可以嵌套任意复杂的表达式：

```javascript
// 复杂嵌套示例
var report = @"Score: ${player.getScore() * 2} / ${maxScore}";

// 分解后等价于：
var report = ("Score: " + string(player.getScore() * 2) +
              " / " + string(maxScore));
```

```javascript
// 嵌套的字符串字面量
var html = @"<div class='${getClass()}'>&content;</div>";

// 分解后等价于：
var html = ("<div class='" + string(getClass()) +
            "'>" + string(content) + "</div>");
```

注意内层的普通字符串（如 `getClass()` 返回的值）不会被再次嵌入式解析——嵌入式表达式只在 `@"..."` 字符串的最外层生效。

---

## 7. 增强层三：语法分析器回调

### 7.1 正则表达式 vs 除法的歧义

在 JavaScript/TJS2 中，`/` 可以是除法运算符或正则表达式字面量的开始。这个歧义在词法层面无法完全解决——需要语法上下文来判断。

```javascript
// 这是除法：
var result = a / b;

// 这是正则表达式：
var pattern = /hello/gi;

// 歧义情况——词法分析器无法仅从 / 判断：
if(x) /pattern/.test(s);  // 正则
x = y / z / w;            // 两个除法
```

### 7.2 SetStartOfRegExp 机制

TJS2 的解决方案是让语法分析器在适当的时机通知词法分析器：

```cpp
void tTJSLexicalAnalyzer::SetStartOfRegExp()
{
    // 1. 回退扫描指针到上一个 Token 的位置
    //    （上一个 Token 就是 /，已经被当作 T_SLASH 返回了）
    Current = PrevPos;

    // 2. 设置正则表达式标志
    RegularExpression = true;

    // 说明：
    // PrevPos 在每次 GetToken 开始时被记录为当前位置
    // 所以回退到 PrevPos 就是回到 / 字符的位置
    // 下次 GetToken 调用时会检查 RegularExpression 标志
    // 如果为 true，调用 TJSParseRegExp 将 / 解析为正则表达式
}

// GetToken 中的处理：
tjs_int tTJSLexicalAnalyzer::GetToken(tTJSVariant &val)
{
    // 正则表达式检查（在 switch 之前）
    if(RegularExpression) {
        RegularExpression = false;
        // 调用正则表达式解析器
        return TJSParseRegExp(Current, val);
        // 返回 T_REGEXP Token
    }

    // ... 正常的 switch 分派 ...
}
```

这个机制的核心是**回退**——当语法分析器发现 `T_SLASH` 在语法上应该是正则表达式的开始时，它调用 `SetStartOfRegExp()`，词法分析器回退到 `/` 的位置，下次被调用时重新将其解析为正则表达式。

### 7.3 SetNextIsBareWord 机制

另一个语法分析器回调是 `SetNextIsBareWord()`，用于处理属性访问中使用保留字作为属性名的情况：

```javascript
// TJS2 允许保留字作为属性名：
obj.if       // .if 应被视为属性名 "if"，不是关键字
obj.class    // .class 应被视为属性名 "class"
obj.return   // .return 同理
```

```cpp
void tTJSLexicalAnalyzer::SetNextIsBareWord()
{
    BareWord = true;
}

// GetToken 中标识符扫描部分的处理：
if(TJS_iswalpha(*Current) || *Current == TJS_W('_')) {
    // 扫描标识符字符
    const tjs_char *start = Current;
    while(TJS_iswalpha(*Current) || TJS_iswdigit(*Current) ||
          *Current == TJS_W('_'))
        TJSNext(&Current);

    // 关键判断：BareWord 标志
    if(BareWord) {
        BareWord = false;
        // 强制返回 T_SYMBOL，即使是保留字
        val = tTJSVariant(ttstr(start, Current - start));
        return T_SYMBOL;
    }

    // 正常路径：查保留字哈希表
    tjs_int token = LookupReservedWord(start, Current - start);
    if(token != -1) {
        // 是保留字，返回对应的 Token 类型
        return token;
    } else {
        // 普通标识符
        val = tTJSVariant(ttstr(start, Current - start));
        return T_SYMBOL;
    }
}
```

语法分析器在 Bison 文法中处理 `.` 运算符的规则里调用 `SetNextIsBareWord()`：

```yacc
/* tjs.y 中的相关规则（简化） */
postfix_expr
    : postfix_expr '.' { lexer->SetNextIsBareWord(); } T_SYMBOL
        { /* 属性访问 */ }
    ;
```

---

## 8. 字符编码与 Unicode 支持

### 8.1 tjs_char 类型

TJS2 使用 `tjs_char`（通常定义为 `wchar_t` 或 `tjs_uint16`）作为源代码的字符类型。在 Windows 平台上，`wchar_t` 是 16 位的 UTF-16 编码单元，这意味着：

- BMP（基本多文种平面，U+0000 到 U+FFFF）中的字符可以直接表示
- 补充平面字符（U+10000 以上）需要代理对（surrogate pair），但 TJS2 的词法分析器**不特别处理代理对**——它将每个 16 位单元视为独立字符

### 8.2 字符分类函数

TJS2 使用自定义的字符分类函数，而不是标准库的 `iswalpha()`/`iswdigit()`：

```cpp
// 数字判断：严格限定为 ASCII 数字
static inline bool TJS_iswdigit(tjs_char ch)
{
    return ch >= TJS_W('0') && ch <= TJS_W('9');
}

// 十六进制数字判断
static inline bool TJS_iswxdigit(tjs_char ch)
{
    return (ch >= TJS_W('0') && ch <= TJS_W('9')) ||
           (ch >= TJS_W('a') && ch <= TJS_W('f')) ||
           (ch >= TJS_W('A') && ch <= TJS_W('F'));
}

// 字母判断：ASCII 字母 + 所有非 ASCII 字符
static inline bool TJS_iswalpha(tjs_char ch)
{
    // ASCII 范围：标准字母判断
    if(ch >= TJS_W('a') && ch <= TJS_W('z')) return true;
    if(ch >= TJS_W('A') && ch <= TJS_W('Z')) return true;

    // 非 ASCII 范围：一律视为字母
    if(ch & 0xFF00) return true;
    //  ^^^^^^^^^^
    //  如果高 8 位非零，说明字符码 >= 0x100
    //  这包含了所有 CJK 字符、西里尔字母、阿拉伯字母等

    return false;
}

// 空白字符判断
static inline bool TJS_iswspace(tjs_char ch)
{
    return ch == TJS_W(' ') ||
           ch == TJS_W('\t') ||
           ch == (tjs_char)TJS_SKIP_CODE;
    // 注意：TJS_SKIP_CODE 也被视为空白，会被跳过
    // 注意：\n 不是空白——换行在 Token 分析中有语义意义
}
```

### 8.3 Unicode 标识符

由于 `TJS_iswalpha` 将所有 `>= 0x100` 的字符视为字母，TJS2 天然支持 Unicode 标识符：

```javascript
// 以下都是合法的 TJS2 标识符：
var 名前 = "太郎";
var データ = %[];
var 播放速度 = 1.0;
var αβγ = 3.14;
```

这个设计是故意的宽泛——它不严格遵循 Unicode 标准中的 `ID_Start`/`ID_Continue` 属性，而是简单地允许所有非 ASCII 字符出现在标识符中。这种做法的优点是实现简单且对日文脚本友好；缺点是某些在 Unicode 中不适合作为标识符的字符（如标点符号、表情符号等）也会被接受。

```cpp
// 标识符扫描的核心循环
const tjs_char *ptr = Current;
while(TJS_iswalpha(*ptr) || TJS_iswdigit(*ptr) ||
      *ptr == TJS_W('_')) {
    TJSNext(&ptr);
}
// ptr 现在指向标识符之后的第一个非标识符字符
// start..ptr 之间就是完整的标识符
```

### 8.4 Unicode 相关的陷阱

在处理 Unicode 时有一个需要注意的细节：`TJS_SKIP_CODE` 的值 `0xFFFF` 恰好在 `ch & 0xFF00` 的判定范围内，意味着 `TJS_iswalpha(TJS_SKIP_CODE)` 返回 `true`。这看起来会导致预处理标记被误认为标识符字符，但实际上不会出问题，因为：

1. `TJS_iswspace` 优先匹配 `TJS_SKIP_CODE`
2. `TJSSkipSpace` 在 Token 扫描开始前就跳过了所有 `TJS_SKIP_CODE`
3. `TJSNext` 在指针前进时也跳过 `TJS_SKIP_CODE`

因此标识符扫描循环永远不会"看到" `TJS_SKIP_CODE`——它在进入扫描之前就已经被跳过了。

---

## 9. PutValue 与 Values 容器

### 9.1 语义值传递机制

词法分析器不仅要识别 Token 的类型，还要传递 Token 携带的语义值。例如：

- 数值字面量 `42` → Token 类型 `T_CONSTVAL`，语义值 `tTJSVariant(42)`
- 字符串字面量 `"hello"` → Token 类型 `T_CONSTVAL`，语义值 `tTJSVariant("hello")`
- 标识符 `myVar` → Token 类型 `T_SYMBOL`，语义值 `tTJSVariant("myVar")`

语义值通过 `PutValue` 方法存入 `Values` 容器：

```cpp
tjs_int tTJSLexicalAnalyzer::PutValue(const tTJSVariant &val)
{
    // 将值追加到 Values 向量
    tjs_int index = (tjs_int)Values.size();
    Values.push_back(val);
    return index;
    // 返回索引，语法分析器通过索引取回值
}
```

### 9.2 GetNext 的返回值协议

GetNext 的返回值是一个 `tjs_int`（Token 类型码），语义值通过引用参数 `val` 传出：

```cpp
// 语法分析器调用词法分析器的方式
tTJSVariant val;
tjs_int token = lexer->GetNext(val);

// token 是 Token 类型，如：
//   T_CONSTVAL  (常量值)
//   T_SYMBOL    (标识符)
//   T_PLUS      (+)
//   T_IF        (if)
//   0           (EOF)

// val 携带语义值（仅对 T_CONSTVAL 和 T_SYMBOL 有意义）
```

### 9.3 Free 方法

`Free` 方法使用 swap 技巧释放 `Values` 容器占用的内存：

```cpp
void tTJSLexicalAnalyzer::Free()
{
    // swap with empty vector 技巧
    // 普通的 clear() 不会释放 vector 的内部缓冲区
    // swap 后原缓冲区被临时 vector 析构时释放
    std::vector<tTJSVariant>().swap(Values);
}
```

这个技巧在 C++11 之前的代码中很常见（C++11 引入了 `shrink_to_fit()`）。TJS2 的代码库使用 C++03 标准，所以采用了这种经典的内存释放方式。

---

## 10. 完整的 Token 扫描流程示例

为了将前面的所有概念串联起来，让我们跟踪一段完整的 TJS2 代码通过词法分析器的全过程。

### 10.1 源代码

```javascript
var msg = @"Score: ${score * 2}";
```

### 10.2 预处理阶段

假设源文件使用 Windows 行尾（CRLF），预处理将 `\r\n` 中的 `\r` 替换为 `TJS_SKIP_CODE`（0xFFFF）：

```
原始：  v a r   m s g   =   @ " . . . " ; \r \n
预处理：v a r   m s g   =   @ " . . . " ; ■  \n
                                            ^^ TJS_SKIP_CODE
```

### 10.3 Token 扫描过程

```
步骤  GetNext 调用      返回 Token        语义值
────  ────────────────  ────────────────  ──────────────
 1    首次调用          ─                 (触发 PreProcess)
      GetToken()        T_VAR             ─
 2    GetToken()        T_SYMBOL          "msg"
 3    GetToken()        T_EQUAL           ─
 4    GetToken()        ─                 (遇到 @"，进入嵌入式表达式模式)
      状态→evsNextIsStringLiteral
      注入 T_LPARENTHESIS               ─
      返回 T_LPARENTHESIS               ─
 5    从状态机          T_CONSTVAL        "Score: "
 6    注入              T_PLUS            ─
 7    注入              T_SYMBOL          "string"  (类型转换函数)
 8    注入              T_LPARENTHESIS    ─
      状态→evsNextIsExpression
 9    GetToken()        T_SYMBOL          "score"
10    GetToken()        T_ASTERISK        ─
11    GetToken()        T_CONSTVAL        2
12    GetToken()        T_RBRACE          ─  (} 触发状态切换)
      状态→evsNextIsStringLiteral
      注入 T_RPARENTHESIS               ─  (关闭 string())
      注入 T_PLUS                       ─
13    从状态机          T_CONSTVAL        ""  (结束引号前的空字符串)
      状态→evsStart
      注入 T_RPARENTHESIS               ─  (关闭外层括号)
14    RetValDeque       T_RPARENTHESIS    ─
15    GetToken()        T_SEMICOLON       ─
16    GetToken()        0                 ─  (EOF)
```

### 10.4 最终 Token 流

语法分析器接收到的 Token 序列等价于：

```javascript
var msg = ("Score: " + string(score * 2) + "");
```

注意末尾的空字符串 `""` 是因为 `${...}` 之后直到结束引号之间没有更多文本。语法分析器/编译器可以在后续阶段优化掉这个无意义的空字符串拼接。

---

## 11. 源码文件组织

TJS2 词法分析器的实现分布在以下文件中：

```
cpp/core/tjs2/
├── tjsLex.h           # 类声明（~145 行）
├── tjsLex.cpp         # 核心实现（~1895 行）
│   ├── 辅助函数       #   L1-200: TJSNext, TJSSkipSpace, 字符转换
│   ├── 字符串解析     #   L200-380: TJSParseString, 转义序列
│   ├── 数值解析       #   L380-730: TJSParseNumber, 十六进制浮点
│   ├── 字节/正则解析  #   L730-880: TJSParseOctet, TJSParseRegExp
│   ├── 保留字表       #   L880-990: 哈希表初始化，50+ 关键字
│   ├── 构造函数       #   L990-1040: 初始化，shebang，DicFunc
│   ├── 预处理指令     #   L1040-1230: @set/@if/@endif
│   ├── GetToken       #   L1230-1620: 主分派 switch
│   └── GetNext        #   L1660-1895: 高层调度器
├── tjspp.y            # 预处理表达式的 Bison 文法
├── tjsScriptBlock.h   # 编译单元（词法分析器的宿主）
└── tjsScriptBlock.cpp # SetText() 中创建词法分析器
```

### 11.1 关键依赖关系

```
tTJSScriptBlock::SetText()
    │
    ├─ new tTJSLexicalAnalyzer(this, script, ...)
    │
    ├─ parser.parse()  ← Bison 生成的语法分析器
    │   │
    │   └─ 反复调用 lexer->GetNext(val)
    │       │
    │       ├─ 可能调用 lexer->SetStartOfRegExp()
    │       └─ 可能调用 lexer->SetNextIsBareWord()
    │
    └─ lexer->Free()   ← 释放 Values 容器
```

---

## 12. 与其他脚本语言词法分析器的对比

### 12.1 设计哲学比较

| 特性 | TJS2 | JavaScript (V8) | Lua | Python |
|------|------|-----------------|-----|--------|
| 实现方式 | 手写递归 | 手写递归 | 手写递归 | 手写递归 |
| CRLF 处理 | TJS_SKIP_CODE 哨兵 | 直接处理 | 直接处理 | 直接处理 |
| 正则歧义 | 语法分析器回调 | 语法分析器回调 | 无正则字面量 | 无正则字面量 |
| Unicode 标识符 | `ch >= 0x100` 全允许 | Unicode 属性表 | ASCII only | Unicode 属性表 |
| 模板字符串 | @"...&expr;..." | \`...${expr}...\` | 无 | f"...{expr}..." |
| 预处理器 | @if/@set/@endif | 无 | 无 | 无 |

### 12.2 TJS2 词法分析器的独特之处

**手写 vs 生成器**：TJS2 没有使用 Flex/Lex 等词法分析器生成器，而是完全手写。这在现代脚本语言中是主流做法——V8 (JavaScript)、CPython、Lua 的词法分析器都是手写的，因为手写词法分析器可以更灵活地处理上下文相关的语法（如正则/除法歧义）和嵌入式表达式。

**内置预处理器**：TJS2 在词法层面内置了类似 C 预处理器的条件编译功能（`@if`/`@set`/`@endif`），这在脚本语言中很少见。这个功能让游戏开发者可以根据编译时条件选择性地包含代码。

**DicFunc 快捷语法**：通过词法层面的 Token 注入来解决语法层面的歧义，这种"quick-hack"方式在工业代码中常见但在教科书中很少提及。它展示了编译器工程中的实用主义——有时最简单的解决方案不是修改文法，而是在词法层面做转换。

---

## 练习题与答案

### 练习 1：TJS_SKIP_CODE 的值选择

**问题**：为什么 `TJS_SKIP_CODE` 选择 `0xFFFF` 而不是 `0x0000`（空字符）作为哨兵值？

<details>
<summary>点击查看答案</summary>

`0x0000` 不能用作哨兵值，因为它是 C 字符串的终止符。如果将 CRLF 中的 `\r` 替换为 `0x0000`，扫描函数在遇到它时会认为到达了字符串末尾（EOF），导致后续代码被截断。

`0xFFFF` 是 Unicode BMP 中的永久保留非字符（noncharacter），根据 Unicode 标准它"不应该在文本交换中出现"。因此在合法的 TJS2 源代码中不会自然出现 `0xFFFF`，使其成为安全的哨兵值。

此外，`0xFFFF` 的高 8 位非零（`0xFF00 & 0xFFFF = 0xFF00`），意味着 `TJS_iswalpha(0xFFFF)` 返回 `true`。但这不会引起问题，因为 `TJS_iswspace` 会优先匹配 `TJS_SKIP_CODE`，而 `TJSSkipSpace` 和 `TJSNext` 都会跳过它。

</details>

### 练习 2：嵌入式表达式的嵌套限制

**问题**：以下 TJS2 代码是否合法？如果合法，词法分析器如何处理？

```javascript
var s = @"outer ${@"inner &x;"} end";
```

<details>
<summary>点击查看答案</summary>

这段代码在 TJS2 中**不合法**。嵌入式表达式状态机只有一层——没有状态栈来支持嵌套的 `@"..."` 字符串。当外层 `@"..."` 的状态机处于 `evsNextIsExpression` 状态时遇到内层的 `@"`，不会启动一个新的嵌入式表达式解析，而是会导致解析错误。

如果需要在嵌入表达式中构造字符串，应该使用普通字符串拼接：

```javascript
var inner = @"inner &x;";
var s = @"outer ${inner} end";
```

或使用普通的 `+` 运算符：

```javascript
var s = @"outer ${'' + x} end";
```

这是 TJS2 嵌入式表达式语法的一个已知限制，与 ES6 模板字面量可以任意嵌套的设计不同。

</details>

### 练习 3：BareWord 的作用范围

**问题**：在以下代码中，哪些 `class` 会被视为裸词（返回 `T_SYMBOL`），哪些会被视为关键字（返回 `T_CLASS`）？

```javascript
class MyObj {
    var class = "wizard";        // (A)
    function getClass() {
        return this.class;       // (B)
    }
}
var c = obj.class;               // (C)
var d = class;                   // (D)
```

<details>
<summary>点击查看答案</summary>

- **(A)** `T_CLASS`（关键字）— 这里的 `class` 不在 `.` 之后，会被正常识别为保留字。这行代码实际上是语法错误，因为 `class` 是保留字，不能用作变量名。
- **(B)** `T_SYMBOL`（裸词）— `this.class` 中的 `class` 在 `.` 之后，语法分析器会调用 `SetNextIsBareWord()`，使其被当作属性名处理。
- **(C)** `T_SYMBOL`（裸词）— 同理，`obj.class` 中的 `class` 在 `.` 之后。
- **(D)** `T_CLASS`（关键字）— 独立出现的 `class` 不在 `.` 之后，被识别为保留字。这行代码同样是语法错误。

`BareWord` 标志的作用范围是**仅下一个标识符**——`SetNextIsBareWord()` 设置 `BareWord = true` 后，GetToken 在处理完一个标识符后立即将其重置为 `false`。因此它精确地只影响 `.` 后面的第一个词。

</details>

### 练习 4：DicFunc 包装的完整 Token 序列

**问题**：给定以下 TJS2 源代码，写出词法分析器实际输出的完整 Token 序列：

```javascript
var a = %[x => 1];
```

<details>
<summary>点击查看答案</summary>

假设 DicFunc 模式已启用，词法分析器输出的 Token 序列为：

```
T_VAR                    → var
T_SYMBOL ("a")           → a
T_EQUAL                  → =
T_LPARENTHESIS           → (      ← DicFunc 注入
T_FUNCTION               → function ← DicFunc 注入
T_LBRACE                 → {      ← DicFunc 注入
T_RETURN                 → return ← DicFunc 注入
T_PERCENT                → %      ← 正常扫描
T_LBRACKET               → [      ← 正常扫描
T_SYMBOL ("x")           → x
T_COMMA                  → ,      ← => 被词法分析器映射为逗号
T_CONSTVAL (1)           → 1
T_RBRACKET               → ]      ← 正常扫描
T_SEMICOLON              → ;      ← DicFunc 注入
T_RBRACE                 → }      ← DicFunc 注入
T_RPARENTHESIS           → )      ← DicFunc 注入
T_LPARENTHESIS           → (      ← DicFunc 注入
T_RPARENTHESIS           → )      ← DicFunc 注入
T_SEMICOLON              → ;      ← 正常扫描
```

等价于解析的代码是：

```javascript
var a = (function { return %[x , 1]; })();
```

注意 `=>` 被映射为 `T_COMMA`——这是 TJS2 的设计决策，`=>` 只是逗号的语法糖，用于字典的 key-value 分隔，使语法更加直观。

</details>

### 练习 5：PreProcess 的行号保持

**问题**：假设以下源代码使用 Windows 行尾（CRLF），每行分别是：

```
第1行: var x = 1;\r\n
第2行: var y = 2;\r\n
第3行: var z = x + y;\r\n
```

预处理后，如何确保第 3 行的 `z` 仍然能被正确报告为"第 3 行"？请解释 TJS_SKIP_CODE 哨兵在行号计算中的作用。

<details>
<summary>点击查看答案</summary>

预处理后的字符串变为（■ 代表 TJS_SKIP_CODE）：

```
var x = 1;■\nvar y = 2;■\nvar z = x + y;■\n
```

行号的正确性取决于以下两个事实：

1. **`\n` 字符被保留**：预处理只替换了 CRLF 中的 `\r` 为 `TJS_SKIP_CODE`，`\n` 仍然保留在原位。因此通过计算 `\n` 的数量来确定行号仍然正确——从字符串开头到 `z` 变量位置，仍然有 2 个 `\n`，所以 `z` 在第 3 行。

2. **`TJS_SKIP_CODE` 被跳过但不被删除**：哨兵字符占据了原来 `\r` 的位置，保持了字符串的总长度不变。如果直接删除 `\r`（将后续字符前移），所有字符的绝对位置都会改变，这会使行号计算（基于字符偏移到行号的映射表）失效。

因此 TJS_SKIP_CODE 设计的核心优势是：**O(1) 就地修改 + 行号不变 + 字符偏移不变**。

</details>
