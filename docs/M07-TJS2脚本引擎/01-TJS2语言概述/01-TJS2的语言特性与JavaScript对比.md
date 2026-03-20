# TJS2 的语言特性与 JavaScript 对比

> **所属模块：** M07-TJS2脚本引擎
> **前置知识：** [P08-编译原理基础](../../P08-编译原理基础/README.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 TJS2 的设计定位——为什么 KiriKiri 引擎选择自研脚本语言而不直接用 Lua 或 JavaScript
2. 掌握 TJS2 与 JavaScript 的语法异同，从而快速上手 TJS2 代码的阅读
3. 了解 TJS2 特有的语言特性（property 声明、八进制打包字符串、with 语句增强等）
4. 理解 TJS2 在 KrKr2 引擎中的运行时环境

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| KiriKiri | 吉里吉里 | 日本开发者 W.Dee 创建的视觉小说引擎，TJS2 是它的内嵌脚本语言 |
| 动态类型 | Dynamic Typing | 变量在运行时才确定类型，不需要在代码中声明类型，如 `var x = 42;`，x 之后可以被赋值为字符串 |
| 原型继承 | Prototype Inheritance | 对象通过原型链而非类层次来继承属性和方法的机制——TJS2 和 JavaScript 都采用此方式（TJS2 同时也支持 class 关键字） |
| 闭包 | Closure | 函数能"捕获"定义时所在作用域的变量，即使该作用域已经退出——这是实现回调、延迟执行的关键机制 |
| property | Property（TJS2 特有） | TJS2 的一种特殊成员声明，允许你为"读取"和"写入"操作分别定义 getter 和 setter 函数，对外表现为普通变量 |
| KAG | KiriKiri Adventure Game | 基于 TJS2 的上层标记语言，使用 `[tag]` 格式编写对话和演出指令 |

## 一、TJS2 的历史与设计定位

### 1.1 为什么要自研脚本语言？

KiriKiri 引擎诞生于 1998 年，其作者 W.Dee（渡邊裕太）需要一种能嵌入到 C++ 引擎中的脚本语言。那时候可选的嵌入式脚本语言有哪些？

| 语言 | 首次发布 | 1998 年状态 | 不适合 KiriKiri 的原因 |
|------|---------|------------|----------------------|
| Lua | 1993 | Lua 3.2 | 语法简陋，没有类系统，字符串处理能力弱 |
| Python | 1991 | Python 1.5 | 嵌入式 API 复杂，运行时体积大，GIL（全局解释器锁）限制 |
| JavaScript | 1995 | 无标准嵌入库 | 当时没有独立的嵌入式 JS 引擎（SpiderMonkey 紧耦合 Mozilla） |
| Tcl | 1988 | Tcl 8.0 | 语法奇特（一切皆字符串），日文处理困难 |

W.Dee 选择自研 TJS（后升级为 TJS2），核心考量是：

1. **日文优先**：TJS2 原生支持 Unicode（使用 UTF-16 编码的宽字符），日文字符串处理无障碍
2. **JavaScript 式语法**：对 Web 开发者友好，学习曲线低
3. **高度可嵌入**：编译为 C++ 库，与引擎紧密集成
4. **游戏特化**：支持 property、octet 字符串等游戏开发常用特性

### 1.2 TJS2 vs JavaScript：一句话总结

> TJS2 ≈ JavaScript ES3 + property 声明 + 八进制打包 + 宽字符原生支持 - 原型链细节差异

如果你会 JavaScript，你已经会 70% 的 TJS2 了。

## 二、语法对比：从 JavaScript 到 TJS2

### 2.1 变量声明

```javascript
// JavaScript
var x = 42;
let y = "hello";  // ES6+
const z = 3.14;   // ES6+

// TJS2
var x = 42;       // 唯一的变量声明方式
var y = "hello";  // 没有 let/const
var z = 3.14;
// TJS2 不区分 let/const/var
// 所有变量都用 var 声明
```

**关键差异**：TJS2 只有 `var`，没有 `let` 和 `const`。所有变量的作用域遵循函数作用域（function scope），而不是 JavaScript ES6+ 的块作用域。

### 2.2 函数定义

```javascript
// JavaScript — 三种方式
function greet(name) { return "Hello, " + name; }
var greet = function(name) { return "Hello, " + name; };
var greet = (name) => "Hello, " + name;  // ES6+

// TJS2 — 两种方式
function greet(name) { return "Hello, " + name; }
var greet = function(name) { return "Hello, " + name; };
// TJS2 没有箭头函数
```

**关键差异**：没有箭头函数（`=>`），没有默认参数，没有剩余参数（`...args`）。

### 2.3 类与继承

这是 TJS2 与 JavaScript（ES3-5）**差异最大**的地方。TJS2 在 1998 年就内置了 `class` 关键字：

```javascript
// TJS2 的 class 语法
class Animal {
    var name;        // 成员变量
    var _age = 0;    // 带默认值的成员变量

    // 构造函数：与类同名的函数
    function Animal(name, age) {
        this.name = name;
        this._age = age;
    }

    // 普通方法
    function speak() {
        return this.name + " makes a sound";
    }

    // property 声明（TJS2 特有！）
    property age {
        getter() {
            return this._age;
        }
        setter(value) {
            if (value < 0) throw new Exception("年龄不能为负");
            this._age = value;
        }
    }
}

// 继承用 extends 关键字
class Dog extends Animal {
    function Dog(name, age) {
        // 调用父类构造函数
        super.Animal(name, age);
    }

    function speak() {
        return this.name + " barks";
    }
}

// 使用
var dog = new Dog("旺财", 3);
dog.age = 5;           // 调用 setter
var a = dog.age;       // 调用 getter → 5
var s = dog.speak();   // "旺财 barks"
```

```javascript
// 对比 JavaScript ES6+ 的 class（2015 年才标准化）
class Animal {
    constructor(name, age) {
        this.name = name;
        this._age = age;
    }

    speak() {
        return `${this.name} makes a sound`;
    }

    get age() { return this._age; }
    set age(value) {
        if (value < 0) throw new Error("年龄不能为负");
        this._age = value;
    }
}
```

**关键差异总结：**

| 特性 | TJS2 | JavaScript (ES3-5) | JavaScript (ES6+) |
|------|------|--------------------|--------------------|
| class 关键字 | ✅ 原生支持 | ❌ 不支持 | ✅ 语法糖 |
| 构造函数 | 与类同名 | `function Foo()` | `constructor()` |
| property getter/setter | `property x { getter() {} setter(v) {} }` | `__defineGetter__` (非标准) | `get x() {} set x(v) {}` |
| 继承 | `extends` | 原型链手动设置 | `extends` |
| 调用父类 | `super.ClassName()` | 手动调用 | `super()` |
| 成员变量声明 | `var name;` 在 class 体内 | 无法声明 | `name;` 或 `#name;` |

### 2.4 运算符差异

TJS2 有几个独特的运算符：

```javascript
// === 身份比较（TJS2）vs 严格相等（JavaScript）
// 在 JavaScript 中，=== 比较值和类型
// 在 TJS2 中，=== 比较对象身份（是否是同一个对象）

// TJS2
var a = new Object();
var b = a;
var c = new Object();
a === b;    // true（同一个对象）
a === c;    // false（不同对象，即使内容相同）

// 特有运算符
var x = 42;
x instanceof "Integer";  // true
x instanceof "String";   // false

// 字符串中的内嵌表达式
var name = "世界";
var msg = "你好，%{name}！";  // TJS2 特有：%{} 插值
// JavaScript 用模板字符串：`你好，${name}！`
```

### 2.5 异常处理

```javascript
// TJS2 的异常处理与 JavaScript 几乎一致
try {
    var result = riskyOperation();
} catch (e) {
    // e 是异常对象
    System.inform("错误: " + e.message);
} finally {
    // 可选的 finally 块
    cleanup();
}

// 抛出异常
throw new Exception("something went wrong");
```

### 2.6 八进制打包字符串（Octet String）

这是 TJS2 独有的特性，JavaScript 中完全没有：

```javascript
// TJS2 的八进制（字节）字符串
// 用 <% %> 包围，内容是十六进制字节
var data = <% 48 65 6C 6C 6F %>;  // 等价于 ASCII "Hello"

// 这实际上是一个字节数组（octet string）
// 类似于 JavaScript 的 Uint8Array 或 Python 的 bytes

// 常见用途：
// 1. 内嵌二进制数据
// 2. 加密密钥
// 3. 文件格式魔术数字
var xp3_magic = <% 58 50 33 0D 0A 20 0A 1A 8B 67 01 %>;

// 可以与普通字符串互转
var str = data;  // 从 octet 转为字符串（按平台编码解释）
```

### 2.7 with 语句增强

TJS2 的 `with` 语句比 JavaScript 更强大，在 KiriKiri 游戏脚本中被大量使用：

```javascript
// TJS2 的 with 语句
with (kag.fore.layers[0]) {
    .loadImages("bg_school");  // 注意前面的点号！
    .visible = true;           // .property 等价于 this.property
    .opacity = 255;
}

// 等价于
kag.fore.layers[0].loadImages("bg_school");
kag.fore.layers[0].visible = true;
kag.fore.layers[0].opacity = 255;

// JavaScript 也有 with，但：
// 1. 不支持前导点号语法
// 2. 严格模式下被禁用
// 3. 社区强烈不推荐使用
```

**前导点号（`.property`）** 是 TJS2 `with` 语句的独特语法。当你在 `with` 块内写 `.xxx` 时，编译器知道这是对 `with` 对象的成员访问。这在视觉小说脚本中极其实用——操作一个图层的多个属性时，代码简洁了很多。

## 三、TJS2 特有特性详解

### 3.1 property 声明

property 是 TJS2 最重要的特有特性。它允许你把看起来像变量访问的操作转换为函数调用：

```javascript
class Sprite {
    var _x = 0;
    var _y = 0;
    var _needsUpdate = false;

    // property 声明
    property x {
        getter() {
            return _x;
        }
        setter(value) {
            if (_x != value) {
                _x = value;
                _needsUpdate = true;  // 标记需要重绘
            }
        }
    }

    property y {
        getter() {
            return _y;
        }
        setter(value) {
            if (_y != value) {
                _y = value;
                _needsUpdate = true;
            }
        }
    }

    // 只读 property（省略 setter）
    property needsUpdate {
        getter() {
            return _needsUpdate;
        }
        // 没有 setter → 尝试赋值会报错
    }
}

var sprite = new Sprite();
sprite.x = 100;  // 看起来像赋值，实际调用了 setter
sprite.y = 200;  // setter 中自动标记了需要重绘
// sprite.needsUpdate == true
```

为什么这在视觉小说引擎中特别重要？因为图层的每个属性变化（位置、透明度、可见性）都可能触发重绘。property 让引擎能在属性变化时自动执行额外逻辑，而游戏脚本开发者完全不需要知道底层细节。

### 3.2 正则表达式

TJS2 的正则表达式语法与 JavaScript 基本一致：

```javascript
// 正则表达式字面量
var pattern = /^Hello,\s+(.+)!/;

// 匹配
var result = pattern.exec("Hello, World!");
// result[0] = "Hello, World!"
// result[1] = "World"

// 替换
var text = "The quick brown fox";
var newText = text.replace(/quick/, "slow");
// newText = "The slow brown fox"

// 标志位
var caseInsensitive = /hello/i;  // i = 不区分大小写
var global = /\d+/g;             // g = 全局匹配
```

KrKr2 中使用 Oniguruma 正则表达式库作为底层实现（文件 `tjsRegExp.cpp`），这意味着 TJS2 的正则功能比 JavaScript 的还要强大——支持命名捕获组、回顾断言等高级特性。

### 3.3 typeof 与 instanceof

```javascript
// typeof 返回字符串类型名
typeof 42;           // "Integer"
typeof 3.14;         // "Real"
typeof "hello";      // "String"
typeof void;         // "Void"
typeof new Object(); // "Object"

// instanceof 检查类型
42 instanceof "Integer";     // true
"hello" instanceof "String"; // true

// 注意与 JavaScript 的差异：
// JavaScript:  typeof 42 === "number"
// TJS2:        typeof 42 === "Integer"
//
// JavaScript:  x instanceof Array
// TJS2:        x instanceof "Array"  （用字符串比较）
```

### 3.4 void 类型

TJS2 有一个显式的 `void` 值，类似 JavaScript 的 `undefined`：

```javascript
// void 是一个值，也是一个类型
var x;            // x 的值为 void
x = void;         // 显式赋值为 void
if (x === void) { // 检查是否为 void
    System.inform("x 是 void");
}

// void 不同于 null
// null = 空对象引用（"这里应该有个对象，但没有"）
// void = 未初始化/无值（"这里从来就没有值"）
```

## 四、TJS2 在 KrKr2 中的运行时环境

### 4.1 TJS2 引擎的生命周期

在 KrKr2 中，TJS2 引擎的生命周期如下：

```
程序启动
    │
    ├─ 1. 创建 tTJS 实例（TJS2 引擎的顶层对象）
    │      位于 cpp/core/tjs2/tjs.cpp
    │
    ├─ 2. 注册全局对象
    │      System, Storages, Debug, Scripts 等
    │      这些对象由 C++ 实现，暴露给 TJS2 脚本使用
    │
    ├─ 3. 加载启动脚本
    │      通常是 startup.tjs
    │      由 ScriptBlock 编译并执行
    │
    ├─ 4. 脚本执行循环
    │      TJS2 脚本调用引擎 API
    │      引擎通过事件回调通知 TJS2 脚本
    │      └─ 游戏逻辑全部由 TJS2 驱动
    │
    └─ 5. 关闭
           释放所有 TJS2 对象（引用计数归零）
           销毁 tTJS 实例
```

### 4.2 全局对象

TJS2 脚本中可以直接使用的全局对象，这些都在 C++ 端通过 `tjsNative` 注册：

```javascript
// System — 系统操作
System.inform("提示消息");       // 弹出消息框
System.title = "我的游戏";       // 设置窗口标题
System.terminate();              // 退出程序

// Storages — 文件操作
var stream = Storages.open("data.tjs");  // 打开文件
Storages.isExistentStorage("bg.png");    // 检查文件存在

// Scripts — 脚本加载
Scripts.execStorage("another.tjs");  // 加载并执行另一个脚本

// Debug — 调试
Debug.message("调试信息");          // 输出调试日志
```

### 4.3 TJS2 与 KAG 的关系

KAG（KiriKiri Adventure Game）是建立在 TJS2 之上的一层标记语言。视觉小说的剧本通常用 KAG 编写，而 KAG 标签最终被翻译为 TJS2 函数调用：

```
; KAG 脚本示例（.ks 文件）
*start|开场
[cm]
[bg storage="bg_school" time=1000]
[l][r]
这里是学校的教室。[l]
[r]
窗外的樱花正在盛开。[l]
[jump target=*next]

*next|下一段
[bg storage="bg_corridor"]
走廊里很安静。
```

```javascript
// 上面的 KAG 标签实际上调用了 TJS2 函数：
// [bg storage="bg_school" time=1000]
// 等价于：
kag.tagHandlers.bg(%[
    storage: "bg_school",
    time: 1000
]);

// %[ ] 是 TJS2 的字典字面量语法
// 类似 JavaScript 的 { }
var dict = %[ key: "value", count: 42 ];
```

### 4.4 对照项目源码

TJS2 引擎的入口在 `cpp/core/tjs2/tjs.cpp` 和 `tjs.h` 中：

**相关文件：**
- `cpp/core/tjs2/tjs.h` — `tTJS` 类定义（TJS2 引擎的顶层类）
- `cpp/core/tjs2/tjs.cpp` — `tTJS` 类实现（初始化、脚本执行、全局对象注册）
- `cpp/core/tjs2/tjsScriptBlock.h` — `tTJSScriptBlock` 类（一个编译单元）
- `cpp/core/tjs2/tjsVariant.h` — `tTJSVariant` 类（TJS2 的动态类型实现）

引擎的核心执行流程：

```
tTJS::ExecScript(script_text)
    │
    ├─ tTJSScriptBlock::Compile()     ← 编译 TJS2 源码
    │   ├─ tTJSLexicalAnalyzer::...  ← 词法分析
    │   └─ yyparse() [bison生成]      ← 语法分析 + 字节码生成
    │
    └─ tTJSScriptBlock::Execute()     ← 执行编译后的字节码
        └─ tjsInterCodeExec::...      ← VM 指令分发循环
```

## 动手实践

### 实践 1：编写你的第一个 TJS2 脚本

虽然 KrKr2 目前没有独立的 TJS2 REPL（交互式解释器），但你可以在游戏启动脚本中编写 TJS2 代码来测试。创建一个简单的 `startup.tjs` 文件：

```javascript
// startup.tjs — TJS2 入门练习
// 这个文件会在 KrKr2 启动时自动执行

// 1. 变量和类型
var greeting = "你好，TJS2！";
var number = 42;
var pi = 3.14159;

System.inform(greeting);
Debug.message("数字: " + number);
Debug.message("类型: " + typeof number);

// 2. 函数定义
function factorial(n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}

Debug.message("10! = " + factorial(10));

// 3. 类定义
class Counter {
    var _count = 0;

    function Counter(initial) {
        _count = initial;
    }

    function increment() {
        _count++;
    }

    property count {
        getter() { return _count; }
    }
}

var counter = new Counter(0);
counter.increment();
counter.increment();
counter.increment();
Debug.message("计数器: " + counter.count);  // 3
```

### 实践 2：对比特性清单

下面是一个 TJS2/JavaScript 特性对照表，请尝试为每个 "?" 填写正确答案：

| 特性 | TJS2 | JavaScript |
|------|------|------------|
| 变量声明 | var | var/let/const |
| 模板字符串 | `%{expr}` | `` `${expr}` `` |
| 字典字面量 | `%[ ]` | `{ }` |
| 空值 | ? | null/undefined |
| 类型检查运算符 | ? | typeof/instanceof |
| 箭头函数 | ? | `() => {}` |
| property 声明 | ✅ | ? |

（答案在练习题部分）

## 本节小结

- TJS2 是 KiriKiri 引擎的内嵌脚本语言，语法高度接近 JavaScript ES3，但在 1998 年就内置了 class、property、extends 等现代特性
- 与 JavaScript 最大的差异是：property 声明语法、八进制打包字符串（`<% %>`）、字典字面量（`%[ ]`）、字符串插值（`%{}`）、前导点号 with 语句
- TJS2 只有 `var` 声明，没有 `let`/`const`，没有箭头函数，没有解构赋值
- 在 KrKr2 中，TJS2 通过 `tTJS` 类实例化运行，脚本可以访问 System、Storages 等 C++ 注册的全局对象
- KAG 是建立在 TJS2 之上的标记语言，`[tag]` 标签最终被翻译为 TJS2 函数调用

## 练习题与答案

### 题目 1：语法改写

请将以下 JavaScript 代码改写为等价的 TJS2 代码：

```javascript
// JavaScript
class Rectangle {
    #width;  // 私有字段
    #height;

    constructor(w, h) {
        this.#width = w;
        this.#height = h;
    }

    get area() {
        return this.#width * this.#height;
    }

    set width(value) {
        if (value <= 0) throw new Error("宽度必须为正");
        this.#width = value;
    }

    toString() {
        return `Rectangle(${this.#width}, ${this.#height})`;
    }
}
```

<details>
<summary>查看答案</summary>

```javascript
// TJS2
class Rectangle {
    var _width;    // TJS2 没有真正的私有字段
    var _height;   // 用下划线前缀约定表示"不应外部访问"

    // 构造函数与类同名
    function Rectangle(w, h) {
        _width = w;
        _height = h;
    }

    // 只读 property
    property area {
        getter() {
            return _width * _height;
        }
    }

    // 可写 property
    property width {
        getter() {
            return _width;
        }
        setter(value) {
            if (value <= 0)
                throw new Exception("宽度必须为正");
            _width = value;
        }
    }

    // 普通方法
    function toString() {
        // TJS2 的字符串插值用 %{}
        return "Rectangle(%{_width}, %{_height})";
    }
}
```

关键差异：
1. 构造函数从 `constructor()` 变为与类同名的 `function Rectangle()`
2. JavaScript 的 `#` 私有字段在 TJS2 中用 `var _xxx` 约定替代
3. `get`/`set` 关键字变为 `property { getter() setter() }` 块
4. 模板字符串从 `` `${}` `` 变为普通字符串中的 `%{}`
5. `Error` 变为 `Exception`

</details>

### 题目 2：特性对照表答案

请填写实践 2 中对照表的空缺部分。

<details>
<summary>查看答案</summary>

| 特性 | TJS2 | JavaScript |
|------|------|------------|
| 变量声明 | var | var/let/const |
| 模板字符串 | `%{expr}` | `` `${expr}` `` |
| 字典字面量 | `%[ ]` | `{ }` |
| 空值 | **void 和 null** | null/undefined |
| 类型检查运算符 | **typeof 和 instanceof（用字符串参数）** | typeof/instanceof |
| 箭头函数 | **不支持** | `() => {}` |
| property 声明 | ✅ | **ES6+ 的 get/set（不同语法）** |

</details>

### 题目 3：阅读 TJS2 代码

以下是一段真实的 KiriKiri 游戏脚本片段。请解释每一行的作用：

```javascript
class MainWindow extends Window {
    var fore;    // 前景图层管理器
    var back;    // 背景图层管理器

    function MainWindow() {
        super.Window();
        fore = new LayerManager(this);
        back = new LayerManager(this);
    }

    property caption {
        getter() {
            return super.caption;
        }
        setter(value) {
            super.caption = "KiriKiri - " + value;
        }
    }

    function onCloseQuery() {
        if (System.inform("确定退出？", "是", "否") == "是") {
            super.close();
        }
    }
}
```

<details>
<summary>查看答案</summary>

逐行解析：

```javascript
class MainWindow extends Window {
// 定义 MainWindow 类，继承自 Window（引擎内置类）

    var fore;    // 声明成员变量 fore（前景图层管理器）
    var back;    // 声明成员变量 back（背景图层管理器）

    function MainWindow() {
    // 构造函数（与类同名）

        super.Window();
        // 调用父类 Window 的构造函数
        // TJS2 用 super.ClassName() 调用父类构造函数
        // 而不是 JavaScript 的 super()

        fore = new LayerManager(this);
        // 创建前景图层管理器，传入当前窗口作为参数

        back = new LayerManager(this);
        // 创建背景图层管理器
    }

    property caption {
    // 声明 property：窗口标题

        getter() {
            return super.caption;
            // 读取时返回父类的 caption 值
        }
        setter(value) {
            super.caption = "KiriKiri - " + value;
            // 写入时在标题前加上 "KiriKiri - " 前缀
            // 用户设置 mainWindow.caption = "我的游戏"
            // 实际显示 "KiriKiri - 我的游戏"
        }
    }

    function onCloseQuery() {
    // 窗口关闭确认回调

        if (System.inform("确定退出？", "是", "否") == "是") {
            // System.inform 弹出对话框
            // 参数：消息, 按钮1文本, 按钮2文本
            // 返回值：用户点击的按钮文本

            super.close();
            // 确认后调用父类的 close() 关闭窗口
        }
    }
}
```

这段代码展示了 TJS2 在实际游戏中的典型用法：继承引擎内置类、使用 property 自定义窗口标题行为、响应用户事件。

</details>

## 下一步

[02-类型系统与变量](./02-类型系统与变量.md) — 深入 TJS2 的类型系统，包括整数、浮点数、字符串、void、null、对象引用的内存表示和类型转换规则。
