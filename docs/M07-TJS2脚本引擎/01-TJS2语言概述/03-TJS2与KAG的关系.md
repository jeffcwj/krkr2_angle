# TJS2 与 KAG 的关系

> **所属模块：** M07-TJS2脚本引擎
> **前置知识：** [TJS2的语言特性与JavaScript对比](./01-TJS2的语言特性与JavaScript对比.md)、[类型系统与变量](./02-类型系统与变量.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 KAG（KiriKiri Adventure Game，KiriKiri 引擎的冒险游戏脚本标记语言）与 TJS2 的分层关系
2. 掌握 KAG 标签语法（Tag Syntax）的完整规则，包括 `[tag]` 和 `@tag` 两种写法
3. 熟悉 KAG 的宏系统（Macro System）、条件分支（Conditional Branch）、标签跳转（Label Jump）等核心机制
4. 理解 `[iscript]...[endscript]` 如何让 TJS2 代码嵌入 KAG 场景中执行
5. 读懂 KrKr2 引擎中 `KAGParser` 的 C++ 实现，掌握场景解析的底层流程

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| KAG | KiriKiri Adventure Game | KiriKiri 引擎的上层标记语言，用于编写视觉小说的剧情脚本 |
| 场景文件 | Scenario File (.ks) | 存放 KAG 标记内容的文本文件，扩展名为 `.ks` |
| 标签 | Tag | KAG 的基本指令单元，格式为 `[tagname attr=value]` |
| 标号 | Label | 场景文件中的定位锚点，以 `*` 开头，用于跳转和调用 |
| 宏 | Macro | 用户自定义的标签组合，可封装多条指令为一个新标签 |
| 实体表达式 | Entity Expression | 以 `&` 开头的 TJS2 表达式，运行时求值后替换为结果字符串 |
| KAGParser | KAG Parser (C++ Class) | KrKr2 引擎中解析 KAG 场景文件的 C++ 核心类 |

## 两层脚本架构：TJS2 与 KAG 的分工

KiriKiri 引擎采用**双层脚本架构**——底层逻辑层使用 TJS2，上层演出层使用 KAG。理解这两层的分工关系是掌握整个引擎的关键：

```
┌─────────────────────────────────────────┐
│          KAG 层（演出脚本）              │
│  .ks 场景文件 — 标记语言，描述"演出"    │
│  [bg storage=bg01.png]                  │
│  [l][r]                                 │
│  这是对话文本。[p]                       │
├─────────────────────────────────────────┤
│          TJS2 层（逻辑引擎）             │
│  .tjs 脚本文件 — 编程语言，描述"行为"    │
│  class MyPlugin { ... }                 │
│  kag.loadScenario("scene01.ks");        │
├─────────────────────────────────────────┤
│          C++ 层（底层引擎）              │
│  KAGParser.cpp — 解析 KAG 标记          │
│  tjs2/ — TJS2 虚拟机                    │
│  visual/ — 渲染层                       │
└─────────────────────────────────────────┘
```

### KAG 的定位：领域特定语言

KAG 并不是一门通用编程语言（General-purpose Language），它是一门**领域特定语言**（Domain-Specific Language, DSL）。DSL 是专门为某个应用领域设计的语言——KAG 的领域就是视觉小说（Visual Novel）的演出控制。KAG 不需要变量声明、循环结构或函数定义，它只关心：

- **显示什么**：背景图、立绘、文本
- **播放什么**：BGM、音效、语音
- **怎么过渡**：场景切换、特效、等待用户操作
- **流程控制**：跳转到哪个标号、调用哪段宏

当 KAG 需要执行"逻辑"（判断变量、计算表达式、操作数据）时，它会**委托给 TJS2**——通过 `[iscript]` 块或实体表达式 `&expr`。

### TJS2 的定位：逻辑引擎

TJS2 则是一门完整的编程语言，拥有变量、函数、类、闭包、异常处理等一切编程要素。在 KiriKiri 引擎中，TJS2 承担的角色包括：

- **初始化引擎**：`startup.tjs` 是引擎启动时执行的第一个 TJS2 脚本
- **管理系统对象**：窗口（Window）、图层（Layer）、KAGParser 等都是 TJS2 对象
- **实现游戏逻辑**：存档读档、变量管理、条件分支的求值
- **扩展 KAG**：通过 TJS2 类为 KAG 注册新的标签处理函数

```tjs
// startup.tjs — TJS2 启动脚本示例
// 创建主窗口对象
var win = new Window();
win.caption = "我的视觉小说";
win.visible = true;

// 加载 KAG 场景
// KAGParser 是一个注册在 TJS2 全局作用域中的 C++ 原生类
var parser = new KAGParser();
parser.loadScenario("first.ks");

// 解析并执行标签
var tag;
while ((tag = parser.getNextTag()) !== void) {
    // tag 是一个 Dictionary 对象，形如 %["tagname" => "bg", "storage" => "bg01.png"]
    switch (tag.tagname) {
        case "bg":
            // 调用渲染层切换背景
            loadBackground(tag.storage);
            break;
        case "p":
            // 等待用户点击
            waitClick();
            break;
        // ... 其他标签处理
    }
}
```

这段代码展示了 TJS2 与 KAG 的典型交互模式：TJS2 创建 `KAGParser` 对象，加载 `.ks` 场景文件，然后循环调用 `getNextTag()` 获取解析后的标签字典，根据标签名分发到相应的处理函数。

## KAG 标签语法详解

### 基本标签格式

KAG 支持两种等价的标签写法：

**方括号格式**（最常用）：
```
[tagname attr1=value1 attr2=value2]
```

**行命令格式**（以 `@` 开头，标签独占一行）：
```
@tagname attr1=value1 attr2=value2
```

两种格式在语义上完全等价，解析器在 `_GetNextTag()` 函数中对它们做统一处理。方括号格式的优势是可以在同一行中混合文本和标签，而行命令格式则更清晰易读。

```
; 方括号格式：标签和文字可以混在一行
今天天气真好。[l]你觉得呢？[p]

; 行命令格式：每个标签独占一行
@bg storage=bg_park.png
@l
今天天气真好。
```

### 属性的值语法

标签的属性值支持多种引用方式：

```
; 无引号 — 值不能包含空格、引号或方括号
[bg storage=bg01.png]

; 双引号 — 值可以包含空格
[text value="hello world"]

; 单引号 — 同双引号
[text value='hello world']

; 反引号转义 — 值中包含引号时使用
; ` 后的下一个字符被视为字面量
[eval exp=`"she said `"hi`""`]
```

对应的 C++ 解析逻辑在 `KAGParser.cpp` 的 `_GetNextTag()` 中（约第 1500-1600 行）：

```cpp
// cpp/core/base/KAGParser.cpp 第 1550-1590 行（简化）
// 属性值解析逻辑
if (*lp == TJS_W('"') || *lp == TJS_W('\'')) {
    // 引号包裹的值
    tjs_char quotechar = *lp;
    lp++;  // 跳过开头引号
    while (*lp != quotechar) {
        if (*lp == TJS_W('`')) {
            // 反引号转义：取下一个字符的字面值
            lp++;  // 跳过反引号
        }
        value += *lp;
        lp++;
    }
    lp++;  // 跳过结尾引号
} else {
    // 无引号的值：读到空格、]、换行为止
    while (*lp != 0 && *lp != TJS_W(']') 
           && *lp != TJS_W(' ') && *lp != TJS_W('\t')) {
        value += *lp;
        lp++;
    }
}
```

### 特殊字符处理

在 KAG 场景文件中，以下字符有特殊含义：

| 字符 | 含义 | 说明 |
|------|------|------|
| `[` `]` | 标签定界符 | 包裹标签指令 |
| `@` | 行命令前缀 | 行首出现时，该行被解析为标签 |
| `*` | 标号定义 | 行首出现时，定义一个标号（Label） |
| `;` | 注释 | 行首出现时，整行被忽略 |
| `\` | 行继续符 | 行末出现时，下一行文本与当前行连接（不产生换行） |
| `&` | 实体表达式 | 引入一个 TJS2 表达式，运行时求值替换 |
| `%` | 宏参数引用 | 在宏内部引用调用者传入的参数 |

普通文本字符会被自动转换为 `ch` 标签输出。例如文本 `你好` 会产生两个标签：`{tagname:"ch", text:"你"}` 和 `{tagname:"ch", text:"好"}`。换行符（CR）则产生 `{tagname:"r"}` 标签。

对应的 C++ 代码（`KAGParser.cpp` 第 1367-1420 行）：

```cpp
// cpp/core/base/KAGParser.cpp 第 1380-1410 行（简化）
// 普通字符 → ch 标签
tjs_char ch = CurLineStr[CurPos];
if (ch != TJS_W('[') && ch != TJS_W(0)) {
    // 不是标签开始，也不是行尾 → 当作普通文字
    CurPos++;
    
    // 构造 ch 标签字典
    DicClear->FuncCall(0, NULL, NULL, 0, NULL, DicObj);
    
    // 设置 tagname = "ch"
    tTJSVariant tagname_val(TJS_W("ch"));
    DicObj->PropSet(TJS_MEMBERENSURE, TJS_W("tagname"), NULL, &tagname_val, DicObj);
    
    // 设置 text = 当前字符
    tjs_char text_buf[2] = { ch, 0 };
    tTJSVariant text_val(text_buf);
    DicObj->PropSet(TJS_MEMBERENSURE, TJS_W("text"), NULL, &text_val, DicObj);
    
    return true;  // 返回这个标签
}

// 换行符 → r 标签
if (/* 检测到换行 */) {
    DicClear->FuncCall(0, NULL, NULL, 0, NULL, DicObj);
    tTJSVariant tagname_val(TJS_W("r"));
    DicObj->PropSet(TJS_MEMBERENSURE, TJS_W("tagname"), NULL, &tagname_val, DicObj);
    return true;
}
```

## 标号系统：场景内的导航锚点

### 标号的定义与格式

标号（Label）是场景文件中的**定位锚点**，以 `*` 开头，可选附带页面名（Page Name）：

```
*opening|序章
这是序章的开始。[p]

*chapter1|第一章 邂逅
春天的校园里樱花正开。[p]

*ending
故事结束了。[p]
```

标号的完整格式为 `*label_name|page_name`：
- `label_name`：标号名称，用于跳转（`[jump]`）和调用（`[call]`）
- `page_name`：可选的页面显示名，仅用于 UI 展示（如存档画面），不影响逻辑

### 标号缓存机制

KAGParser 在首次访问场景文件的标号时，会执行**懒加载的标号缓存**（Lazy Label Cache）：遍历所有行，找出以 `*` 开头的行，将标号名映射到行号，存储在哈希表中。

```cpp
// cpp/core/base/KAGParser.cpp — EnsureLabelCache() 简化
// 标号缓存的构建过程
void tTVPScenarioCacheItem::EnsureLabelCache() {
    if (LabelCached) return;  // 已缓存则跳过
    LabelCached = true;
    
    for (tjs_uint i = 0; i < LineCount; i++) {
        const tjs_char *line = Lines[i].Start;
        if (line[0] == TJS_W('*')) {
            // 找到标号行
            ttstr label_name;
            ttstr page_name;
            
            // 解析标号名和页面名（以 | 分隔）
            const tjs_char *p = line + 1;
            while (*p && *p != TJS_W('|')) {
                label_name += *p++;
            }
            if (*p == TJS_W('|')) {
                p++;
                while (*p) page_name += *p++;
            }
            
            // 去除首尾空格
            label_name.Trim();
            
            // 检查重复标号 — 重复时自动追加 ":序号"
            if (LabelCache.find(label_name) != LabelCache.end()) {
                // 已存在同名标号，追加计数后缀
                int count = 2;
                ttstr new_name;
                do {
                    new_name = label_name + TJS_W(":") + ttstr(count);
                    count++;
                } while (LabelCache.find(new_name) != LabelCache.end());
                label_name = new_name;
            }
            
            // 存入缓存：标号名 → 行号
            LabelCache[label_name] = i;
            LabelAliases.push_back(label_name);
        }
    }
}
```

重复标号的处理策略很有特色：如果同名标号出现多次，后续的标号会自动追加 `:2`、`:3` 等后缀。例如两个 `*start` 标号，第二个实际存储为 `start:2`。

### 跳转与调用

KAG 提供三种标号导航指令：

```
; jump — 无条件跳转（不保存返回位置）
[jump storage=scene02.ks target=*chapter1]

; call — 调用（保存当前位置到调用栈，以便 return 返回）
[call storage=common.ks target=*show_title]

; return — 从 call 调用返回到之前的位置
[return]
```

`jump` 和 `call` 的区别类似于汇编语言中的 `jmp` 和 `call`：

```
┌──────────────────────────────┐
│ scene01.ks                   │
│ ...                          │
│ [call target=*greet]  ──────►│──┐  call: 压入返回地址
│ ◄── 返回到这里继续执行       │  │
│ ...                          │  │
│                              │  │
│ *greet                       │  │
│ 你好！[p]                    │◄─┘
│ [return]  ─── 弹出返回地址 ──│──► 回到 call 的下一行
└──────────────────────────────┘
```

KAGParser 内部使用 `CallStack`（一个 `std::vector<tCallStackData>`）来管理调用栈。每次 `call` 会压入当前解析状态（文件名、标号、行号、行缓冲区、解析位置等），每次 `return` 则弹出恢复：

```cpp
// cpp/core/base/KAGParser.cpp — call 和 return 的核心逻辑（简化）

// [call] 标签处理
void tTJSNI_KAGParser::CallLabel(const ttstr &label) {
    // 1. 保存当前状态到调用栈
    tCallStackData data;
    data.Storage = CurStorage;       // 当前文件
    data.Label = CurLabel;           // 当前标号
    data.Offset = CurLine;           // 当前行号
    data.OrgLineStr = CurLineStr;    // 当前行内容
    data.LineBuffer = LineBuffer;    // 行缓冲区（可能含宏展开内容）
    data.Pos = CurPos;               // 当前解析位置
    data.ExcludeLevelStack = ExcludeLevel;  // 条件栈状态
    data.IfLevelExecutedStack = IfLevelExecuted;
    CallStack.push_back(data);
    
    // 2. 跳转到目标标号
    GoToLabel(label);
}

// [return] 标签处理
void tTJSNI_KAGParser::PopCallStack() {
    if (CallStack.empty()) {
        TVPThrowExceptionMessage(TJS_W("KAGParser: call stack underflow"));
        return;
    }
    
    // 恢复之前保存的状态
    tCallStackData &data = CallStack.back();
    CurStorage = data.Storage;
    CurLabel = data.Label;
    CurLine = data.Offset;
    CurLineStr = data.OrgLineStr;
    LineBuffer = data.LineBuffer;
    CurPos = data.Pos;
    ExcludeLevel = data.ExcludeLevelStack;
    IfLevelExecuted = data.IfLevelExecutedStack;
    CallStack.pop_back();
}
```

### 场景缓存：LRU 策略

KAGParser 不会每次加载场景都重新解析文件。它维护一个**LRU（Least Recently Used，最近最少使用）缓存**，最多保留 8 个已解析的场景文件：

```cpp
// cpp/core/base/KAGParser.cpp — 场景缓存
#define TVP_SCENARIO_MAX_CACHE_SIZE 8

// LRU 缓存哈希表
static tTJSHashTable<ttstr, tTVPScenarioCacheItem*> TVPScenarioCache;

tTVPScenarioCacheItem* TVPGetScenario(const ttstr &storagename) {
    // 1. 先查缓存
    tTVPScenarioCacheItem **item = TVPScenarioCache.Find(storagename);
    if (item) {
        (*item)->AddRef();  // 引用计数加一
        return *item;       // 命中缓存，直接返回
    }
    
    // 2. 缓存未命中，创建新项
    tTVPScenarioCacheItem *newitem = new tTVPScenarioCacheItem(storagename);
    
    // 3. 缓存满了？淘汰最旧的
    while (TVPScenarioCache.GetCount() >= TVP_SCENARIO_MAX_CACHE_SIZE) {
        // 找到引用计数为 1（只有缓存本身持有）的最旧项，删除
        // ... 淘汰逻辑 ...
    }
    
    // 4. 加入缓存
    TVPScenarioCache.Add(storagename, newitem);
    newitem->AddRef();
    return newitem;
}
```

LRU 缓存的好处是：当游戏在几个场景之间频繁切换时（比如从主场景跳到通用对话场景再跳回来），不需要反复解析同一个文件。最多 8 个缓存项在实际游戏中足够覆盖大多数场景切换模式。

## 条件分支：if/else/elsif/endif

KAG 提供了完整的条件分支语法，但条件表达式本身由 TJS2 求值：

```
[if exp="f.visited_park == true"]
你之前来过这个公园。[p]
[elsif exp="f.know_park == true"]
你听说过这个公园。[p]
[else]
这是一个陌生的公园。[p]
[endif]
```

`exp` 属性的值是一个 TJS2 表达式，在运行时通过 `TVPExecuteExpression()` 求值。返回非零值（truthy）则执行该分支，否则跳过。

### 条件栈的实现

KAGParser 使用两个整数变量来跟踪条件嵌套状态：

```
ExcludeLevel：排除层级（Exclude Level）
  = 0  → 正常执行模式，标签正常返回
  > 0  → 跳过模式，标签被吞掉不返回

IfLevel：条件嵌套深度（If Nesting Level）
  记录当前嵌套了多少层 if

IfLevelExecuted：已执行标记（If Level Executed Flag）
  记录当前 if 层级是否已有分支被执行过
```

工作原理图解：

```
ExcludeLevel = 0（正常执行）

[if exp="true"]          → ExcludeLevel 保持 0, IfLevel++
  这段文字会显示。       → 正常输出（ExcludeLevel == 0）
[else]                   → 已执行过 → ExcludeLevel 变为 1
  这段文字被跳过。       → 被跳过（ExcludeLevel > 0）
[endif]                  → ExcludeLevel 恢复 0, IfLevel--

[if exp="false"]         → 条件为假 → ExcludeLevel 变为 1
  这段被跳过。           → 被跳过
[elsif exp="true"]       → 条件为真 → ExcludeLevel 恢复 0
  这段会显示。           → 正常输出
[else]                   → 已执行过 → ExcludeLevel 变为 1
  这段也被跳过。         → 被跳过
[endif]                  → ExcludeLevel 恢复 0
```

嵌套条件时，ExcludeLevel 可以大于 1，确保内层条件在外层为假时整体被跳过：

```
[if exp="false"]         → ExcludeLevel = 1
  [if exp="true"]        → ExcludeLevel = 2（不会因为内层为 true 而恢复到 0）
    这段被跳过。
  [endif]                → ExcludeLevel = 1（回到外层的排除状态）
[endif]                  → ExcludeLevel = 0
```

对应的 C++ 实现（`KAGParser.cpp` 第 1700-1780 行，简化）：

```cpp
// cpp/core/base/KAGParser.cpp — if/else/elsif/endif 处理逻辑
// [if] 标签
if (tagname == TJS_W("if")) {
    IfLevel++;
    if (ExcludeLevel == 0) {
        // 当前处于执行模式，求值条件表达式
        tTJSVariant val;
        ttstr exp = tag_attr["exp"];
        TVPExecuteExpression(exp, &val);
        
        if (val.IsTrue()) {
            // 条件为真 → 继续执行
            IfLevelExecuted = true;
        } else {
            // 条件为假 → 进入排除模式
            ExcludeLevel = 1;
            IfLevelExecuted = false;
        }
    } else {
        // 已在排除模式 → 嵌套深度加一
        ExcludeLevel++;
    }
}

// [elsif] 标签
if (tagname == TJS_W("elsif")) {
    if (ExcludeLevel == 0) {
        // 当前分支在执行 → 说明前面有分支已执行
        // 进入排除模式，跳过后续所有分支
        ExcludeLevel = 1;
    } else if (ExcludeLevel == 1 && !IfLevelExecuted) {
        // 排除深度为 1（同层）且未执行过 → 尝试这个分支
        tTJSVariant val;
        ttstr exp = tag_attr["exp"];
        TVPExecuteExpression(exp, &val);
        
        if (val.IsTrue()) {
            ExcludeLevel = 0;
            IfLevelExecuted = true;
        }
    }
}

// [else] 标签
if (tagname == TJS_W("else")) {
    if (ExcludeLevel == 0) {
        // 前面已有分支执行过 → 跳过 else
        ExcludeLevel = 1;
    } else if (ExcludeLevel == 1 && !IfLevelExecuted) {
        // 没有分支执行过 → else 兜底执行
        ExcludeLevel = 0;
        IfLevelExecuted = true;
    }
}

// [endif] 标签
if (tagname == TJS_W("endif")) {
    if (ExcludeLevel > 0) {
        ExcludeLevel--;
    }
    IfLevel--;
}
```

## 宏系统：用户自定义标签

KAG 的宏系统允许用户将多个标签组合封装为一个新的"自定义标签"。这是 KAG 最强大的扩展机制之一。

### 宏的定义与使用

```
; 定义一个宏：显示人物对话
[macro name=dialogue]
[font color=0x3366FF][emb exp="mp.name"][resetfont]
[r]
[endmacro]

; 使用宏（就像使用普通标签一样）
[dialogue name="小明"]
今天天气真好。[p]
```

宏定义时，`[macro name=xxx]` 和 `[endmacro]` 之间的内容被记录为字符串模板，存储在 `Macros` 字典中。当解析器遇到宏名对应的标签时，它将宏体字符串**插入到当前行缓冲区**（LineBuffer）中，实现"文本替换"式的展开。

### 宏参数：% 引用语法

宏内部通过 `%argname` 或 `%argname|default` 引用调用者传入的参数：

```
[macro name=chara_say]
; %name 引用调用者传入的 name 参数
; %color|0xFFFFFF 如果调用者没传 color，则默认为白色
[font color=%color|0xFFFFFF]【%name】[resetfont][r]
[endmacro]

; 调用时传参
[chara_say name="小红" color=0xFF6666]
你好啊！[p]

; 也可以省略有默认值的参数
[chara_say name="旁白"]
远处传来钟声。[p]
```

参数查找的 C++ 实现使用 `MacroArgs` 栈——一个 `std::vector` 存储每层宏调用的参数字典。查找时从栈顶开始，如果当前宏没有该参数则查找上层宏（支持宏嵌套调用时的参数透传）：

```cpp
// cpp/core/base/KAGParser.cpp — 宏参数解析（简化）
// 遇到 %name 时的处理
if (*lp == TJS_W('%')) {
    lp++;  // 跳过 %
    
    // 读取参数名
    ttstr param_name;
    while (*lp && *lp != TJS_W('|') && *lp != TJS_W(' ') 
           && *lp != TJS_W(']') && *lp != TJS_W('\t')) {
        param_name += *lp++;
    }
    
    // 读取默认值（如果有 | 分隔符）
    ttstr default_value;
    bool has_default = false;
    if (*lp == TJS_W('|')) {
        lp++;
        has_default = true;
        while (*lp && *lp != TJS_W(' ') && *lp != TJS_W(']')) {
            default_value += *lp++;
        }
    }
    
    // 从 MacroArgs 栈中查找参数值（从栈顶往下搜索）
    ttstr result;
    bool found = false;
    for (int i = MacroArgs.size() - 1; i >= 0; i--) {
        if (MacroArgs[i].find(param_name) != MacroArgs[i].end()) {
            result = MacroArgs[i][param_name];
            found = true;
            break;
        }
    }
    
    if (!found && has_default) {
        result = default_value;
    }
    
    // 将结果插入行缓冲区
    // ...
}
```

### 宏展开的内部机制

宏展开的核心思想是**字符串插入**：当解析器识别到一个宏标签时，它将宏体文本插入到当前行的解析位置，然后继续从插入点开始解析。这种方式类似 C 语言预处理器的宏展开，但发生在运行时。

```
原始行：[dialogue name="小明"]今天天气真好。
                  ↓ 识别到 dialogue 是宏
展开后的 LineBuffer：
[font color=0x3366FF][emb exp="mp.name"][resetfont][r][macropop]今天天气真好。
↑ 宏体内容被插入                                    ↑ 自动追加 macropop
                                                     ↑ 原始内容保留在后面
```

注意 `[macropop]` 是自动追加的——它在宏体执行完毕后弹出参数栈（MacroArgs），确保宏参数的作用域正确结束。

### emb 标签：内嵌表达式

`[emb exp=expression]` 标签用于在 KAG 文本中嵌入 TJS2 表达式的求值结果：

```
; 显示玩家姓名
你好，[emb exp="f.player_name"]！

; 显示计算结果
你已经收集了 [emb exp="f.items.count"] 个道具。

; mp 对象引用当前宏的参数
[macro name=showname]
【[emb exp="mp.name"]】
[endmacro]
```

`emb` 标签的处理过程：解析器调用 `TVPExecuteExpression()` 求值 `exp` 属性的 TJS2 表达式，将结果转换为字符串，然后**插入到当前行缓冲区**中继续解析。这意味着 `emb` 的结果还可以包含 KAG 标签，形成动态生成标签的强大能力。

## [iscript] 与 [endscript]：嵌入 TJS2 代码块

当 KAG 的标记语法无法满足逻辑需求时，可以直接嵌入 TJS2 代码块：

```
[iscript]
// 这里是完整的 TJS2 代码
var greeting;
if (System.getTickCount() % 2 == 0) {
    greeting = "偶数时刻，你好！";
} else {
    greeting = "奇数时刻，你好！";
}
f.greeting = greeting;  // 存到游戏变量中
[endscript]

; 回到 KAG 模式，使用刚才设置的变量
[emb exp="f.greeting"][p]
```

### iscript 的解析流程

解析器在遇到 `[iscript]` 或 `@iscript` 时，会切换到**脚本收集模式**：逐行读取后续内容，直到遇到 `[endscript]` 或 `@endscript`，然后将收集到的所有行拼接成一个字符串，通过 `onScript` 回调发送给 TJS2 引擎执行。

```cpp
// cpp/core/base/KAGParser.cpp — iscript 处理（SkipCommentOrLabel 函数内）
// 约第 1120-1160 行（简化）
if (/* 当前行匹配 [iscript] 或 @iscript */) {
    ttstr script;
    tjs_int script_start_line = CurLine;
    
    CurLine++;  // 跳过 [iscript] 所在行
    
    // 逐行收集脚本内容
    while (CurLine < LineCount) {
        ttstr line = GetCurrentLine();
        
        // 检查是否遇到 [endscript] 或 @endscript
        if (/* line 匹配 endscript */) {
            CurLine++;  // 跳过 [endscript] 行
            break;
        }
        
        // 将这一行追加到脚本字符串
        script += line;
        script += TJS_W("\r\n");
        CurLine++;
    }
    
    // 通过 onScript 回调将脚本发送给 Owner（通常是 TJS2 的 KAG 系统对象）
    // Owner 会调用 TVPExecuteScript() 执行这段 TJS2 代码
    tTJSVariant args[3];
    args[0] = script;              // 脚本内容
    args[1] = CurStorage;          // 所在的 .ks 文件名
    args[2] = script_start_line;   // 起始行号（用于错误报告）
    
    Owner->FuncCall(0, TJS_W("onScript"), NULL, NULL, 3, args, Owner);
}
```

`onScript` 回调的三个参数特别值得注意：除了脚本内容本身，还传递了文件名和起始行号。这使得 TJS2 在执行出错时，能够报告错误发生在哪个 `.ks` 文件的第几行——对调试非常重要。

### 实体表达式：& 前缀

除了 `[iscript]` 块，KAG 还支持在属性值中使用 `&` 前缀引入 TJS2 表达式：

```
; 用实体表达式动态设置属性值
[bg storage=&f.current_bg]

; 等价于（假设 f.current_bg == "bg_park.png"）
[bg storage=bg_park.png]
```

实体表达式在**属性值解析阶段**就会被求值替换。解析器在解析属性值时，如果发现 `&` 前缀，就调用 `TVPExecuteExpression()` 对后续的 TJS2 表达式求值，将结果替换为字符串：

```cpp
// cpp/core/base/KAGParser.cpp — 实体表达式处理（简化）
if (*lp == TJS_W('&')) {
    lp++;  // 跳过 &
    
    // 收集表达式文本（到空格、] 或行尾为止）
    ttstr expression;
    int paren_depth = 0;
    while (*lp) {
        if (*lp == TJS_W('(')) paren_depth++;
        else if (*lp == TJS_W(')')) {
            if (paren_depth == 0) break;
            paren_depth--;
        }
        else if (paren_depth == 0 && 
                 (*lp == TJS_W(' ') || *lp == TJS_W(']'))) break;
        expression += *lp++;
    }
    
    // 调用 TJS2 求值
    tTJSVariant result;
    TVPExecuteExpression(expression, &result);
    
    // 将结果转为字符串，替换原表达式
    ttstr result_str = result;
    // 将 result_str 插入属性值中
}
```

## KAGParser 的完整生命周期

### 作为 TJS2 原生类注册

KAGParser 并不是一个纯 C++ 内部类——它被注册为 TJS2 的原生类（Native Class），可以在 TJS2 脚本中直接使用 `new KAGParser()` 创建实例：

```cpp
// cpp/core/base/ScriptMgnIntf.cpp — KAGParser 注册
// 在引擎初始化时，注册 KAGParser 类到 TJS2 全局作用域
void TVPInitScriptEngine() {
    // ... 创建 TJS2 引擎实例 ...
    
    // 注册原生类（约 20 个）
    engine->registerObject(TJS_W("KAGParser"), 
                           TVPCreateNativeClass_KAGParser());
    // ... 其他类注册 ...
}
```

注册后，TJS2 脚本可以直接使用：

```tjs
// TJS2 中使用 KAGParser
var parser = new KAGParser();

// 方法：加载场景文件
parser.loadScenario("scene01.ks");

// 方法：跳转到标号
parser.goToLabel("*chapter1");

// 方法：调用标号（压入调用栈）
parser.callLabel("*subroutine");

// 方法：获取下一个标签
var tag = parser.getNextTag();

// 属性：当前解析位置
var line = parser.curLine;        // 当前行号
var pos = parser.curPos;          // 当前列位置
var str = parser.curLineStr;      // 当前行内容
var file = parser.curStorage;     // 当前文件名
var label = parser.curLabel;      // 当前标号

// 属性：运行时配置
parser.processSpecialTags = true; // 是否处理特殊标签
parser.ignoreCR = false;          // 是否忽略换行符
parser.debugLevel = 0;            // 调试级别

// 属性：宏系统
var macros = parser.macros;       // 宏定义字典
var mp = parser.mp;               // 当前宏参数
var depth = parser.callStackDepth; // 调用栈深度

// 方法：状态保存与恢复（用于存档/读档）
var state = parser.store();       // 保存当前状态
parser.restore(state);            // 恢复状态
```

### KAGParser 的回调机制

KAGParser 通过 `Owner` 对象（通常是 TJS2 中的 KAG 系统类）的方法回调来通知各种事件：

| 回调方法 | 触发时机 | 参数 |
|----------|----------|------|
| `onScenarioLoad` | 加载场景文件前 | `(storageName)` — 返回字符串则使用该字符串作为场景内容 |
| `onScenarioLoaded` | 场景文件加载完成后 | `(storageName)` |
| `onLabel` | 遇到标号行时 | `(labelName, pageName)` |
| `onScript` | 遇到 `[iscript]...[endscript]` 块时 | `(scriptContent, storageName, startLineNo)` |
| `onJump` | 执行 `[jump]` 前 | `(targetStorage, targetLabel)` |
| `onCall` | 执行 `[call]` 前 | `(targetStorage, targetLabel)` |
| `onReturn` | 执行 `[return]` 前 | 无 |
| `onAfterReturn` | `[return]` 执行完成后 | 无 |

特别值得注意的是 `onScenarioLoad` 回调：它在场景文件实际加载之前触发。如果回调函数返回一个字符串，KAGParser 会使用这个字符串作为场景内容，而不是从文件读取。这提供了一个强大的钩子（Hook），允许 TJS2 层在加载场景时进行预处理——比如加密场景文件的解密、动态生成场景内容等。

```tjs
// TJS2 中实现 onScenarioLoad 钩子
class MyKAGSystem {
    var parser;
    
    function MyKAGSystem() {
        parser = new KAGParser();
        // KAGParser 的 Owner 就是 this
    }
    
    // 当 KAGParser 要加载场景文件时，这个方法被回调
    function onScenarioLoad(storageName) {
        // 可以在这里做场景预处理
        if (storageName == "encrypted.ks") {
            // 解密场景文件
            var encrypted = Scripts.loadStorageToString(storageName);
            var decrypted = decrypt(encrypted, key);
            return decrypted;  // 返回解密后的内容给解析器使用
        }
        // 返回 void 则由 KAGParser 自己从文件加载
    }
    
    // [iscript] 块的回调
    function onScript(script, storage, lineNo) {
        // 执行嵌入的 TJS2 代码
        Scripts.exec(script, storage + ":" + lineNo);
    }
}
```

## 动手实践

### 实践 1：编写一个简单的 KAG 场景

创建一个 `test_scene.ks` 文件，体验 KAG 的基本语法：

```
; test_scene.ks — 一个完整的 KAG 场景示例
; 注释行以分号开头

*start|开始
; 设置背景
@bg storage=bg_room.png

; 显示对话文本
这是一个测试场景。[l]
按下鼠标继续。[p]

; 使用 iscript 嵌入 TJS2 代码
[iscript]
f.visit_count = 0 if f.visit_count == void;
f.visit_count++;
[endscript]

; 用实体表达式显示变量
你已经访问了 [emb exp="f.visit_count"] 次。[p]

; 条件分支
[if exp="f.visit_count >= 3"]
你来了很多次了。[p]
[else]
欢迎你的到来。[p]
[endif]

; 跳转到另一个标号
[jump target=*ending]

*ending|结局
故事到此结束。[p]
```

### 实践 2：定义和使用宏

创建一个包含宏定义的通用场景文件：

```
; macro.ks — 宏定义文件

; 定义一个角色对话宏
[macro name=say]
[font color=%color|0xFFFFFF bold=true]【%name】[resetfont][r]
[endmacro]

; 定义一个带立绘切换的完整对话宏
[macro name=talk]
[chgfg storage=%fg|none layer=0]
[say name=%name color=%color|0xFFFF00]
[endmacro]

; 使用宏
[talk name="小明" fg=chara_xiaoming_smile.png color=0x66CCFF]
今天的天空真蓝。[p]

[talk name="小红" fg=chara_xiaohong_normal.png color=0xFF6699]
是啊，适合出去散步。[p]
```

### 实践 3：跟踪 KAGParser 的解析流程

使用调试思维跟踪以下场景的解析过程，写出每次 `getNextTag()` 返回的标签字典：

```
*start
你好[l]世界[p]
```

预期解析结果序列：

```
1. 跳过 *start 标号行（触发 onLabel 回调）
2. getNextTag() → %["tagname" => "ch", "text" => "你"]
3. getNextTag() → %["tagname" => "ch", "text" => "好"]
4. getNextTag() → %["tagname" => "l"]
5. getNextTag() → %["tagname" => "ch", "text" => "世"]
6. getNextTag() → %["tagname" => "ch", "text" => "界"]
7. getNextTag() → %["tagname" => "p"]
8. getNextTag() → void（场景结束）
```

## 对照项目源码

> **M 系列必须包含此节**

以下是 KAGParser 相关的核心源码文件及其职责：

### 核心文件列表

- `cpp/core/base/KAGParser.h` 第 1-346 行 — KAGParser 的完整头文件，定义了 `tTVPScenarioCacheItem`（场景缓存项，包含行数组、标号缓存哈希表）和 `tTJSNI_KAGParser`（解析器本体，包含 Owner、Macros 字典、MacroArgs 参数栈、CallStack 调用栈、条件状态变量）
- `cpp/core/base/KAGParser.cpp` 第 1-2560 行 — 完整实现，包含：
  - 第 1-150 行：`tTVPScenarioCacheItem` 的场景加载和行分割
  - 第 151-350 行：标号缓存构建（`EnsureLabelCache`）和场景 LRU 缓存管理（`TVPGetScenario`）
  - 第 350-700 行：`tTJSNI_KAGParser` 的初始化、`LoadScenario`、`GoToLabel`
  - 第 700-1070 行：`Store`/`Restore`（状态序列化，用于存档）
  - 第 1071-1187 行：`SkipCommentOrLabel`（跳过注释、标号行，处理 `[iscript]`）
  - 第 1187-1366 行：辅助函数
  - 第 1367-2185 行：`_GetNextTag()`（核心解析循环——普通字符→ch 标签、标签解析、特殊标签处理、宏展开、实体求值）
  - 第 2206-2557 行：TJS2 原生类注册（方法和属性绑定）
- `cpp/core/base/ScriptMgnIntf.cpp` 第 1-1460 行 — 脚本引擎管理，其中第 800-850 行将 KAGParser 注册为 TJS2 全局类

### 关键数据结构

```
tCallStackData（调用栈帧）
├── Storage: ttstr          — 场景文件名
├── Label: ttstr            — 当前标号名
├── Offset: tjs_int         — 当前行号
├── OrgLineStr: ttstr       — 原始行字符串
├── LineBuffer: ttstr       — 行缓冲区（含宏展开内容）
├── Pos: tjs_int            — 行内解析位置
├── ExcludeLevelStack       — 条件排除层级
└── IfLevelExecutedStack    — 条件执行标记

tTVPScenarioCacheItem（场景缓存项）
├── Buffer: tTVPCharHolder  — 文件内容字符缓冲区
├── Lines[]: {Start, Length} — 行数组（指针+长度）
├── LineCount: tjs_uint     — 总行数
├── LabelCache: HashMap     — 标号名→行号映射
├── LabelAliases: vector    — 标号别名列表
└── RefCount: int           — 引用计数
```

## 本节小结

- KiriKiri 引擎采用**TJS2 + KAG 双层架构**：KAG 是面向演出的领域特定语言，TJS2 是面向逻辑的通用编程语言
- KAG 标签有两种写法：`[tag attr=val]`（方括号格式）和 `@tag attr=val`（行命令格式），语义完全等价
- KAG 的**标号系统**（`*label|page`）提供场景内导航，`[jump]` 直接跳转，`[call]`/`[return]` 支持子程序调用
- KAG 的**条件分支**（`[if]`/`[elsif]`/`[else]`/`[endif]`）使用 TJS2 表达式求值，通过 `ExcludeLevel` 计数器实现嵌套跳过
- KAG 的**宏系统**通过字符串插入实现展开，参数通过 `%name|default` 引用，支持嵌套和参数透传
- **`[iscript]...[endscript]`** 允许在 KAG 中嵌入完整的 TJS2 代码块，通过 `onScript` 回调执行
- **实体表达式** `&expr` 在属性值中嵌入 TJS2 表达式，运行时求值替换
- `KAGParser` 作为 TJS2 原生类注册在全局作用域，TJS2 脚本可直接创建和操作解析器实例
- 场景文件使用 **LRU 缓存**（最多 8 项）避免重复解析
- `KAGParser` 通过**回调机制**（onScenarioLoad、onScript、onJump 等）与 TJS2 层交互

## 练习题与答案

### 题目 1：KAG 标签解析输出

给定以下 KAG 场景内容：

```
*scene1|测试场景
; 这是注释
你好[bg storage=bg01.png]世界[p]
```

写出 `getNextTag()` 依次返回的所有标签字典（包括 ch 标签和普通标签）。

<details>
<summary>查看答案</summary>

解析流程：
1. `*scene1|测试场景` — 标号行，被 `SkipCommentOrLabel` 跳过，触发 `onLabel("scene1", "测试场景")` 回调
2. `; 这是注释` — 注释行，被跳过
3. 开始解析第三行 `你好[bg storage=bg01.png]世界[p]`

`getNextTag()` 返回序列：

```
调用 1: %["tagname" => "ch", "text" => "你"]
调用 2: %["tagname" => "ch", "text" => "好"]
调用 3: %["tagname" => "bg", "storage" => "bg01.png"]
调用 4: %["tagname" => "ch", "text" => "世"]
调用 5: %["tagname" => "ch", "text" => "界"]
调用 6: %["tagname" => "p"]
调用 7: void（场景结束）
```

注意：
- 标号行和注释行不会产生任何标签输出
- 每个汉字字符都会产生独立的 `ch` 标签
- `[bg storage=bg01.png]` 的属性值没有引号，因为不含空格
- 最后 `getNextTag()` 返回 `void` 表示场景解析结束

</details>

### 题目 2：条件分支嵌套追踪

追踪以下 KAG 代码的执行，假设 `f.x = 5, f.y = 10`，写出最终显示的文本：

```
[if exp="f.x > 3"]
条件A成立。
  [if exp="f.y < 5"]
  条件B成立。
  [else]
  条件B不成立。
  [endif]
[else]
条件A不成立。
[endif]
```

<details>
<summary>查看答案</summary>

追踪 ExcludeLevel 状态变化：

```
初始状态: ExcludeLevel=0, IfLevel=0

[if exp="f.x > 3"]     → f.x=5 > 3 为 true
                          ExcludeLevel=0（保持）, IfLevel=1, IfLevelExecuted=true
"条件A成立。"           → 输出（ExcludeLevel==0）    ✅ 显示

  [if exp="f.y < 5"]   → f.y=10 < 5 为 false
                          ExcludeLevel=1, IfLevel=2, IfLevelExecuted=false
  "条件B成立。"         → 跳过（ExcludeLevel>0）      ❌ 不显示

  [else]                → ExcludeLevel==1 且未执行过 → ExcludeLevel=0
                          IfLevelExecuted=true
  "条件B不成立。"       → 输出（ExcludeLevel==0）    ✅ 显示

  [endif]               → ExcludeLevel 保持 0, IfLevel=1

[else]                  → 已执行过（IfLevelExecuted==true）→ ExcludeLevel=1
"条件A不成立。"         → 跳过（ExcludeLevel>0）      ❌ 不显示

[endif]                 → ExcludeLevel=0, IfLevel=0
```

最终显示的文本：

```
条件A成立。
条件B不成立。
```

</details>

### 题目 3：编写一个 KAG 宏

编写一个名为 `choice` 的 KAG 宏，实现如下功能：
- 显示一段提示文本（参数 `prompt`）
- 根据参数 `option1` 和 `option2` 显示两个选项
- 使用 `[if]`/`[else]`/`[endif]` 根据参数 `selected`（值为 1 或 2）显示选中结果

<details>
<summary>查看答案</summary>

```
; 宏定义
[macro name=choice]
; 显示提示文本
[font size=28 bold=true]%prompt[resetfont][r]
[r]
; 显示选项
1. %option1[r]
2. %option2[r]
[r]

; 根据 selected 参数显示结果
[if exp="mp.selected == '1'"]
你选择了：%option1[p]
[else]
你选择了：%option2[p]
[endif]
[endmacro]

; 使用示例
[choice prompt="你想去哪里？" option1="公园" option2="图书馆" selected=1]
```

说明：
- `%prompt`、`%option1`、`%option2`、`%selected` 都是宏参数引用
- `mp.selected` 在 `[if]` 的 TJS2 表达式中引用当前宏参数——`mp` 是 KAGParser 提供的特殊对象，等价于当前宏的参数字典
- 宏展开时，所有 `%xxx` 会被替换为实际参数值，然后 `[if]` 中的 TJS2 表达式被求值

注意：在实际游戏中，选项选择通常不是通过预设参数 `selected` 实现的，而是通过 TJS2 层的 UI 交互来设置变量。这里简化为参数传入以演示宏语法。

</details>

## 下一步

[词法分析器整体架构](../02-词法分析器/01-tjsLex整体架构.md) — 深入 TJS2 引擎内部，从词法分析器开始，了解 TJS2 源代码如何被分解为 Token 流。
