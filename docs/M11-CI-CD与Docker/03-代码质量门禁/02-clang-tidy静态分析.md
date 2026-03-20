# clang-tidy 静态分析

> **所属模块：** M11-CI/CD 与 Docker
> **前置知识：** [01-clang-format 代码格式化](./01-clang-format代码格式化.md)
> **预计阅读时间：** 45 分钟（约 9000 字）

## 本节目标

读完本节后，你将能够：
1. 理解 clang-tidy 与 clang-format 的本质区别——"格式" vs "语义"
2. 逐类解读 KrKr2 `.clang-tidy` 配置文件中的 8 大检查类别（共 100+ 条规则）
3. 在本地运行 clang-tidy 对 KrKr2 源码进行静态分析并理解输出
4. 在 VS Code、CLion、Visual Studio 中集成 clang-tidy 实时检查
5. 识别并修复 clang-tidy 报告的常见警告

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 静态分析 | Static Analysis | 不运行程序，仅通过分析源代码文本来发现潜在 bug 和代码异味的技术 |
| clang-tidy | clang-tidy | LLVM 项目提供的 C++ 静态分析和代码转换工具，基于 Clang 编译器前端 |
| 检查规则 | Check | clang-tidy 中的一条具体检查逻辑，如 `bugprone-use-after-move` 检查移动后使用 |
| 编译数据库 | Compilation Database | 一个 JSON 文件（`compile_commands.json`），记录每个源文件的编译命令，供分析工具使用 |
| AST | Abstract Syntax Tree | 抽象语法树，编译器将源代码解析后生成的树形结构，clang-tidy 在 AST 上做模式匹配 |
| CERT | CERT C++ Coding Standard | 美国卡内基梅隆大学 SEI 发布的 C/C++ 安全编码标准 |
| C++ Core Guidelines | C++ Core Guidelines | 由 Bjarne Stroustrup 和 Herb Sutter 主导的 C++ 最佳实践指南 |

---

## clang-format vs clang-tidy：格式与语义

在上一节中我们学习了 clang-format——它关注代码的**外观**：缩进、空格、换行、大括号位置。clang-format 不理解代码的含义，只做"排版"。

clang-tidy 则完全不同，它关注代码的**语义正确性和质量**：

| 维度 | clang-format | clang-tidy |
|------|-------------|------------|
| 关注点 | 代码"长什么样" | 代码"写得对不对" |
| 分析层次 | 纯文本（Token 流，即词法单元序列） | AST（抽象语法树，编译器对代码结构的完整理解） |
| 典型问题 | 缩进不一致、行太长 | 使用已移动的变量、遗漏 `override`、性能问题 |
| 是否修改代码 | 总是修改（格式化） | 默认只报告，`--fix` 可自动修复部分问题 |
| 运行速度 | 极快（毫秒级） | 较慢（需要解析编译，秒到分钟级） |
| 需要编译信息 | 不需要 | 需要（`compile_commands.json` 编译数据库） |

**一个直观的比喻：** clang-format 是"排版编辑"，让文章段落整齐美观；clang-tidy 是"审稿编辑"，检查文章有没有逻辑错误、用词不当、前后矛盾。

---

## KrKr2 `.clang-tidy` 配置文件解析

KrKr2 项目根目录下的 `.clang-tidy` 文件共 142 行，由 CLion IDE（JetBrains 出品的 C++ 集成开发环境）自动生成。文件的第一行注释说明了这一点：

```yaml
# Generated from CLion Inspection settings
```

### 配置结构概览

整个文件只有一个关键字段 `Checks`，使用 YAML 格式的多行字符串：

```yaml
---
Checks: '-*,
bugprone-argument-comment,
bugprone-assert-side-effect,
...
readability-use-anyofallof'
```

**`-*`** 是关键——它的含义是"先禁用所有内置检查规则"。clang-tidy 内置了数百条检查规则，如果全部启用会产生大量噪音（与当前项目无关的警告）。`-*` 先清空，然后逐条添加需要的检查，形成**白名单模式**（Whitelist，只启用明确列出的规则，其他一律禁用）。

### 检查规则分类统计

KrKr2 启用了以下 8 大类检查（加上 3 条特殊类别），共计 **100+ 条**规则：

| 类别 | 英文前缀 | 启用数量 | 关注领域 |
|------|----------|----------|----------|
| 缺陷检测 | `bugprone-` | 49 | 常见编程错误、可疑代码模式 |
| 现代化 | `modernize-` | 22 | 将旧式 C++ 代码升级到 C++11/14/17 风格 |
| 可读性 | `readability-` | 17 | 代码清晰度、冗余代码消除 |
| 性能 | `performance-` | 13 | 不必要的拷贝、低效算法使用 |
| 安全编码 | `cert-` | 9 | CERT C++ 安全编码标准合规 |
| 核心指南 | `cppcoreguidelines-` | 5 | C++ Core Guidelines 合规 |
| 杂项 | `misc-` | 6 | 其他通用最佳实践 |
| Google 规范 | `google-` | 2 | Google C++ 风格指南中的特定规则 |
| HIC++ | `hicpp-` | 2 | 高完整性 C++（High Integrity C++）标准 |
| 并行 | `mpi-` / `openmp-` | 3 | MPI 和 OpenMP 并行编程检查 |
| 可移植性 | `portability-` | 1 | SIMD 内联函数可移植性 |

> **注意：** MPI（Message Passing Interface，消息传递接口，用于分布式并行计算的通信标准）和 OpenMP（Open Multi-Processing，一种基于编译器指令的共享内存并行编程 API）的检查虽然被启用，但 KrKr2 项目中实际上不使用 MPI，OpenMP 也仅在非 Apple 平台启用。这些规则的存在是因为配置从 CLion 模板生成，包含了一些"无害但不适用"的规则。

---

## 八大检查类别详解

### 1. bugprone — 缺陷检测（49 条规则）

`bugprone` 是最大的检查类别，专门捕获常见的编程错误。这些规则的核心思想是：**某些代码虽然能编译通过，但几乎总是 bug**。

#### 代表性规则详解

**`bugprone-use-after-move`** — 检测"移动后使用"

C++ 的移动语义（Move Semantics，C++11 引入的特性，允许将资源"转移"而非复制，避免不必要的内存分配）允许你把一个对象的资源"偷走"给另一个对象。但移动之后，原对象处于"有效但未指定"状态——可以安全销毁，但不应该再读取其值。

```cpp
#include <iostream>
#include <string>
#include <vector>

int main() {
    std::string name = "KrKr2 Emulator";

    // 移动 name 的内容到 other
    std::string other = std::move(name);

    // ❌ bugprone-use-after-move: name 已被移动，值不可预测
    std::cout << "Name: " << name << std::endl;  // 可能输出空字符串

    // ✅ 正确做法：移动后不再使用 name，或重新赋值后再使用
    name = "New Name";  // 重新赋值后安全
    std::cout << "Name: " << name << std::endl;  // 输出 "New Name"

    return 0;
}
```

**`bugprone-sizeof-expression`** — 检测 `sizeof` 的可疑用法

```cpp
#include <cstring>
#include <iostream>

int main() {
    char buffer[256];
    const char* src = "Hello";

    // ❌ bugprone-sizeof-expression: sizeof(src) 是指针大小(8字节)，不是字符串长度
    std::memcpy(buffer, src, sizeof(src));  // 只拷贝了 8 字节

    // ✅ 正确做法：使用 strlen + 1 获取实际长度（含终止符）
    std::memcpy(buffer, src, std::strlen(src) + 1);

    std::cout << buffer << std::endl;  // 输出 "Hello"
    return 0;
}
```

**`bugprone-suspicious-semicolon`** — 检测可疑的空语句

```cpp
#include <iostream>

int main() {
    int x = 10;

    // ❌ bugprone-suspicious-semicolon: if 后面多了分号，循环体总是执行
    if(x > 100);  // ← 这个分号让 if 的"身体"变成空语句
    {
        std::cout << "这一行总是执行，不管 x 的值" << std::endl;
    }

    // ✅ 正确写法
    if(x > 100) {
        std::cout << "只有 x > 100 时才执行" << std::endl;
    }

    return 0;
}
```

**`bugprone-macro-parentheses`** — 检测宏定义缺少括号

```cpp
#include <iostream>

// ❌ bugprone-macro-parentheses: 参数和整体都缺少括号
#define SQUARE(x) x * x

// ✅ 正确写法：参数和整体都加括号
#define SQUARE_SAFE(x) ((x) * (x))

int main() {
    // SQUARE(1 + 2) 展开为 1 + 2 * 1 + 2 = 5（而非预期的 9）
    std::cout << SQUARE(1 + 2) << std::endl;      // 输出 5 ❌
    std::cout << SQUARE_SAFE(1 + 2) << std::endl;  // 输出 9 ✅
    return 0;
}
```

#### 其他重要 bugprone 规则一览

| 规则 | 检测场景 |
|------|----------|
| `bugprone-branch-clone` | if/else 的两个分支代码完全相同（通常是复制粘贴 bug） |
| `bugprone-copy-constructor-init` | 拷贝构造函数没有正确初始化基类 |
| `bugprone-dangling-handle` | 字符串视图（`string_view`）指向已销毁的临时字符串 |
| `bugprone-integer-division` | 整数除法赋值给浮点数（精度丢失） |
| `bugprone-move-forwarding-reference` | 对转发引用使用 `std::move` 而非 `std::forward` |
| `bugprone-virtual-near-miss` | 虚函数名拼写与基类几乎相同但不完全匹配（想 override 但没生效） |
| `bugprone-unused-raii` | RAII 对象创建后立即销毁（如 `std::lock_guard<std::mutex>(mtx);`） |

### 2. modernize — 现代化（22 条规则）

`modernize` 类别帮助将 C++98/03 风格的代码升级到 C++11/14/17 风格。这些规则不是修 bug，而是利用新语言特性让代码更简洁、更安全。

#### 代表性规则详解

**`modernize-use-override`** — 添加 `override` 关键字

`override`（C++11 引入的说明符，明确标记一个成员函数是对基类虚函数的重写，如果签名不匹配则编译报错）是防止虚函数重写错误的重要安全网：

```cpp
#include <iostream>
#include <string>

class BaseRenderer {
public:
    virtual void render(int width, int height) {
        std::cout << "BaseRenderer::render " << width << "x" << height << std::endl;
    }
    virtual ~BaseRenderer() = default;
};

// ❌ modernize-use-override: 缺少 override 标记
class OpenGLRenderer : public BaseRenderer {
public:
    virtual void render(int width, int height) {  // 应加 override
        std::cout << "OpenGLRenderer::render " << width << "x" << height << std::endl;
    }
};

// ✅ 正确写法
class VulkanRenderer : public BaseRenderer {
public:
    void render(int width, int height) override {  // override 明确标记重写
        std::cout << "VulkanRenderer::render " << width << "x" << height << std::endl;
    }
};

int main() {
    VulkanRenderer vr;
    BaseRenderer* base = &vr;
    base->render(1920, 1080);  // 输出 "VulkanRenderer::render 1920x1080"
    return 0;
}
```

**`modernize-use-nullptr`** — 用 `nullptr` 替代 `NULL` 和 `0`

```cpp
#include <iostream>

void process(int value) {
    std::cout << "process(int): " << value << std::endl;
}

void process(const char* str) {
    std::cout << "process(const char*): " << (str ? str : "null") << std::endl;
}

int main() {
    // ❌ modernize-use-nullptr: NULL 可能被定义为 0，导致调用 process(int)
    process(NULL);     // 可能调用 process(int) 而非 process(const char*)

    // ✅ nullptr 明确表示空指针，调用 process(const char*)
    process(nullptr);  // 确保调用 process(const char*)

    return 0;
}
```

**`modernize-use-auto`** — 在类型明显时使用 `auto`

```cpp
#include <map>
#include <string>
#include <iostream>

int main() {
    std::map<std::string, int> scores;
    scores["Alice"] = 95;
    scores["Bob"] = 87;

    // ❌ modernize-use-auto: 类型可以从右侧推断
    std::map<std::string, int>::iterator it = scores.begin();

    // ✅ 使用 auto 简化（类型从 scores.begin() 返回值推断）
    auto it2 = scores.begin();

    std::cout << it2->first << ": " << it2->second << std::endl;
    return 0;
}
```

#### 其他重要 modernize 规则一览

| 规则 | 自动升级内容 |
|------|-------------|
| `modernize-make-unique` | `new T(args)` → `std::make_unique<T>(args)` |
| `modernize-make-shared` | `new T(args)` → `std::make_shared<T>(args)` |
| `modernize-loop-convert` | C 风格 `for(int i=0; i<n; ++i)` → 范围 for `for(auto& x : container)` |
| `modernize-use-emplace` | `vec.push_back(T(args))` → `vec.emplace_back(args)` |
| `modernize-deprecated-headers` | `<stdio.h>` → `<cstdio>`, `<stdlib.h>` → `<cstdlib>` |
| `modernize-use-equals-default` | 空的构造/析构函数体 → `= default` |
| `modernize-use-equals-delete` | 私有未实现的拷贝构造 → `= delete` |
| `modernize-use-noexcept` | `throw()` → `noexcept` |
| `modernize-raw-string-literal` | 转义字符串 → R"(...)" 原始字符串 |
| `modernize-pass-by-value` | 构造函数中 `const T&` 参数 + 拷贝 → 按值传递 + `std::move` |

### 3. performance — 性能优化（13 条规则）

`performance` 类别检测会导致不必要的性能开销的代码模式——如多余的拷贝、低效的容器操作。

#### 代表性规则详解

**`performance-unnecessary-copy-initialization`** — 不必要的拷贝初始化

```cpp
#include <string>
#include <vector>
#include <iostream>

std::string getConfig() {
    return "window_width=1920";
}

int main() {
    std::vector<std::string> names = {"Alice", "Bob", "Charlie"};

    // ❌ performance-unnecessary-copy-initialization: 拷贝了整个字符串
    // name 只用来读取，没有修改，应该用 const 引用
    for(int i = 0; i < static_cast<int>(names.size()); ++i) {
        const std::string name = names[i];  // 每次循环都拷贝一个字符串
        std::cout << name << std::endl;
    }

    // ✅ 使用 const 引用避免拷贝
    for(int i = 0; i < static_cast<int>(names.size()); ++i) {
        const std::string& name = names[i];  // 零拷贝
        std::cout << name << std::endl;
    }

    // ✅✅ 更好：使用范围 for + auto
    for(const auto& name : names) {
        std::cout << name << std::endl;
    }

    return 0;
}
```

**`performance-inefficient-string-concatenation`** — 低效的字符串拼接

```cpp
#include <string>
#include <iostream>

int main() {
    std::string base = "/home/user";
    std::string dir = "/games";
    std::string file = "/krkr2.exe";

    // ❌ performance-inefficient-string-concatenation:
    // 每个 + 都创建临时 string 对象
    std::string path = base + dir + file;

    // ✅ 使用 append 或 reserve + append 减少临时对象
    std::string path2;
    path2.reserve(base.size() + dir.size() + file.size());  // 预分配内存
    path2.append(base).append(dir).append(file);

    std::cout << path << std::endl;   // /home/user/games/krkr2.exe
    std::cout << path2 << std::endl;  // /home/user/games/krkr2.exe

    return 0;
}
```

#### 其他重要 performance 规则

| 规则 | 优化场景 |
|------|----------|
| `performance-for-range-copy` | 范围 for 中按值捕获导致拷贝 → 改用 `const auto&` |
| `performance-move-const-arg` | 对 const 对象或基本类型调用 `std::move`（无意义的移动） |
| `performance-noexcept-move-constructor` | 移动构造函数缺少 `noexcept`（阻止 `std::vector` 优化） |
| `performance-inefficient-vector-operation` | 循环中 `push_back` 不预先 `reserve`（反复扩容） |
| `performance-unnecessary-value-param` | 函数参数按值传递大对象但只读取（应改为 `const&`） |

### 4. cert — 安全编码标准（9 条规则）

`cert` 规则来自 CERT C++ 安全编码标准，关注可能导致安全漏洞或未定义行为的代码模式。

#### 代表性规则详解

**`cert-err34-c`** — 检查 `atoi`/`atof` 等函数的返回值

`atoi`（ASCII to Integer，C 标准库中将字符串转换为整数的函数）在转换失败时返回 0，与输入 "0" 无法区分——这是一个严重的安全隐患。

```cpp
#include <cstdlib>
#include <iostream>
#include <string>

int main() {
    const char* input = "not_a_number";

    // ❌ cert-err34-c: atoi 无法区分 "0" 和转换失败
    int value = std::atoi(input);  // 返回 0，但不知道是否转换成功
    std::cout << "atoi result: " << value << std::endl;

    // ✅ 使用 std::stoi（C++11），转换失败抛出异常
    try {
        int safe_value = std::stoi(input);
        std::cout << "stoi result: " << safe_value << std::endl;
    } catch(const std::invalid_argument& e) {
        std::cerr << "转换失败: " << e.what() << std::endl;
    } catch(const std::out_of_range& e) {
        std::cerr << "值超出范围: " << e.what() << std::endl;
    }

    return 0;
}
```

**`cert-dcl58-cpp`** — 禁止在 `std` 命名空间中添加定义

```cpp
// ❌ cert-dcl58-cpp: 不允许在 std 命名空间中添加内容
namespace std {
    template<>
    struct hash<MyClass> {  // 未定义行为！
        size_t operator()(const MyClass& obj) const;
    };
}

// ✅ C++11 允许的例外：模板特化
// 但更推荐的做法是自定义 hash 函数对象，在容器声明时指定
struct MyClassHash {
    size_t operator()(const MyClass& obj) const {
        return std::hash<int>()(obj.id);
    }
};
// 使用：std::unordered_set<MyClass, MyClassHash> mySet;
```

### 5. readability — 可读性（17 条规则）

`readability` 类别消除冗余代码、统一命名风格，让代码更容易阅读和理解。

#### 代表性规则详解

**`readability-container-size-empty`** — 用 `empty()` 替代 `size() == 0`

```cpp
#include <vector>
#include <string>
#include <iostream>

int main() {
    std::vector<int> data;

    // ❌ readability-container-size-empty: size() == 0 不如 empty() 直观
    // 且某些容器的 size() 可能是 O(n) 复杂度
    if(data.size() == 0) {
        std::cout << "容器为空（size 写法）" << std::endl;
    }

    // ✅ empty() 更清晰，且保证 O(1) 复杂度
    if(data.empty()) {
        std::cout << "容器为空（empty 写法）" << std::endl;
    }

    return 0;
}
```

**`readability-redundant-string-cstr`** — 检测多余的 `.c_str()` 调用

```cpp
#include <string>
#include <iostream>

int main() {
    std::string filename = "game.xp3";

    // ❌ readability-redundant-string-cstr:
    // std::cout 直接支持 std::string，不需要 .c_str()
    std::cout << filename.c_str() << std::endl;

    // ✅ 直接使用 std::string
    std::cout << filename << std::endl;

    // 注意：以下场景 .c_str() 是必要的（C API 需要 const char*）
    // FILE* f = fopen(filename.c_str(), "r");  // C 函数需要 const char*
    return 0;
}
```

### 6. cppcoreguidelines — C++ Core Guidelines（5 条规则）

这些规则来自 C++ 语言之父 Bjarne Stroustrup 和 C++ 标准委员会成员 Herb Sutter 主导编写的 **C++ Core Guidelines**——一份旨在让 C++ 代码更安全、更高效的最佳实践文档。

| 规则 | 含义 |
|------|------|
| `cppcoreguidelines-pro-type-member-init` | 类成员变量必须在构造函数中初始化，防止读取未初始化内存 |
| `cppcoreguidelines-narrowing-conversions` | 检测隐式窄化转换（如 `double` → `int`），可能丢失数据 |
| `cppcoreguidelines-slicing` | 将派生类对象赋值给基类对象时发生"切片"——派生类独有的数据被丢弃 |
| `cppcoreguidelines-pro-type-static-cast-downcast` | 用 `static_cast` 向下转型不安全，应使用 `dynamic_cast` |
| `cppcoreguidelines-interfaces-global-init` | 全局变量的初始化顺序在不同编译单元之间未定义——避免全局变量相互依赖 |

```cpp
#include <iostream>

class Entity {
public:
    int health;     // ❌ cppcoreguidelines-pro-type-member-init: 未初始化
    int damage;     // ❌ 同上

    // ✅ 使用成员初始化列表（或类内默认值）
    // Entity() : health(100), damage(10) {}
};

class Player : public Entity {
public:
    std::string name;
};

int main() {
    Player p;
    p.name = "Hero";
    p.health = 100;
    p.damage = 15;

    // ❌ cppcoreguidelines-slicing: Player 被切片为 Entity
    Entity e = p;  // name 字段丢失！
    // e 只有 health 和 damage，没有 name

    // ✅ 使用指针或引用避免切片
    Entity& ref = p;  // 引用保留多态性
    std::cout << ref.health << std::endl;

    return 0;
}
```

### 7. misc — 杂项检查（6 条规则）

| 规则 | 含义 |
|------|------|
| `misc-throw-by-value-catch-by-reference` | 抛出异常时按值抛出，捕获时按引用捕获——防止切片和不必要拷贝 |
| `misc-new-delete-overloads` | 重载 `operator new` 时必须同时重载 `operator delete` |
| `misc-non-copyable-objects` | 某些系统资源（如文件句柄）不应被拷贝 |
| `misc-uniqueptr-reset-release` | `a.reset(b.release())` 应改为 `a = std::move(b)` |
| `misc-unconventional-assign-operator` | 赋值运算符应返回 `T&` 类型 |
| `misc-misplaced-const` | `const` 位置不当，如 `char const*` vs `const char*` |

### 8. google / hicpp — 特定标准（4 条规则）

```cpp
#include <iostream>

// ❌ google-default-arguments: 虚函数不应有默认参数
// 默认参数在编译时绑定基类版本，运行时调用派生类版本——两者可能不一致
class Base {
public:
    virtual void draw(int x, int y = 0) {  // 默认参数 y=0
        std::cout << "Base::draw " << x << ", " << y << std::endl;
    }
    virtual ~Base() = default;
};

class Derived : public Base {
public:
    void draw(int x, int y = 100) override {  // 默认参数 y=100
        std::cout << "Derived::draw " << x << ", " << y << std::endl;
    }
};

int main() {
    Derived d;
    Base* ptr = &d;

    // 通过基类指针调用：使用基类的默认参数 y=0，但调用派生类的函数体
    ptr->draw(10);  // 输出 "Derived::draw 10, 0" (不是 100！)

    // 直接调用：使用派生类的默认参数 y=100
    d.draw(10);     // 输出 "Derived::draw 10, 100"

    return 0;
}
```

> `hicpp-exception-baseclass` 要求只抛出 `std::exception` 的派生类（不要抛出 `int`、`const char*` 等非异常类型）。`hicpp-multiway-paths-covered` 要求 `switch` 语句覆盖所有枚举值或有 `default` 分支。

---

## 在本地运行 clang-tidy

### 前提：生成编译数据库

clang-tidy 需要知道每个源文件的编译命令（包括头文件搜索路径、宏定义、编译标志等），这些信息存储在 `compile_commands.json` 文件中——即**编译数据库**（Compilation Database）。

对于 CMake 项目，生成方法非常简单：

```bash
# 方法 1：在 CMake 配置时生成（推荐）
cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

# 生成的文件位于：build/compile_commands.json
```

```bash
# 方法 2：使用 KrKr2 的 CMake Preset（自动传递参数）
cmake --preset="Linux Debug Config"
# 需要手动添加 -DCMAKE_EXPORT_COMPILE_COMMANDS=ON 或在 CMakePresets.json 中配置
```

生成的 `compile_commands.json` 内容示例：

```json
[
  {
    "directory": "/home/user/krkr2/build",
    "command": "/usr/bin/c++ -DTJS_TEXT_OUT_CRLF -I/home/user/krkr2/cpp/core/tjs2 -std=c++17 -o CMakeFiles/tjs2.dir/tjsLex.cpp.o -c /home/user/krkr2/cpp/core/tjs2/tjsLex.cpp",
    "file": "/home/user/krkr2/cpp/core/tjs2/tjsLex.cpp"
  }
]
```

### 运行 clang-tidy

#### 基本用法

```bash
# 检查单个文件
clang-tidy -p build cpp/core/tjs2/tjsLex.cpp

# -p build：指定编译数据库所在目录
# 输出示例：
# /home/user/krkr2/cpp/core/tjs2/tjsLex.cpp:142:5: warning:
#   use 'nullptr' instead of 'NULL' [modernize-use-nullptr]
```

#### 检查整个目录

```bash
# 使用 run-clang-tidy（clang-tidy 的批量运行脚本，随 LLVM 安装）
# 自动并行处理所有文件
run-clang-tidy -p build cpp/core/

# 或手动查找文件并逐个检查
find cpp/ -name "*.cpp" | xargs -P4 -I{} clang-tidy -p build {}
```

> **Windows 用户注意：** Windows 上 `find` 是系统命令而非 GNU find。请使用 Git Bash 或 WSL 运行上述命令，或使用 PowerShell 等效写法：
> ```powershell
> Get-ChildItem -Recurse -Filter *.cpp cpp\ |
>     ForEach-Object { clang-tidy -p build $_.FullName }
> ```

#### 自动修复

```bash
# --fix 标志让 clang-tidy 自动修复它能修复的问题
clang-tidy -p build --fix cpp/core/visual/LoadPNG.cpp

# --fix-errors 更激进：即使修复可能改变语义也会应用
# （慎用，修复后务必检查代码）
clang-tidy -p build --fix-errors cpp/core/visual/LoadPNG.cpp
```

#### 只运行特定类别的检查

```bash
# 只运行 modernize 类检查
clang-tidy -p build -checks='-*,modernize-*' cpp/core/tjs2/tjsLex.cpp

# 只运行 performance 和 bugprone 类检查
clang-tidy -p build -checks='-*,performance-*,bugprone-*' cpp/core/tjs2/tjsLex.cpp

# 使用项目 .clang-tidy 配置但排除某个规则
clang-tidy -p build -checks='-bugprone-macro-parentheses' cpp/core/tjs2/tjsLex.cpp
```

### 四平台安装 clang-tidy

| 平台 | 安装命令 | 备注 |
|------|----------|------|
| **Ubuntu/Debian** | `sudo apt install clang-tidy` | 或 `clang-tidy-20` 指定版本 |
| **macOS** | `brew install llvm` | 安装在 `/opt/homebrew/opt/llvm/bin/clang-tidy` |
| **Windows** | 安装 Visual Studio 的"C++ Clang 工具" | 或从 [LLVM 官网](https://releases.llvm.org/) 下载安装包 |
| **Android** | NDK 自带 | `$ANDROID_NDK/toolchains/llvm/prebuilt/*/bin/clang-tidy` |

---

## 编辑器集成

### VS Code

安装 **clangd** 扩展（由 LLVM 官方维护，提供代码补全、诊断、格式化一体化支持）：

1. 安装扩展：`Extensions` → 搜索 `clangd` → 安装
2. 确保 `compile_commands.json` 存在于项目 `build/` 目录中
3. 在 `.vscode/settings.json` 中配置：

```json
{
    "clangd.arguments": [
        "--compile-commands-dir=build",
        "--clang-tidy",
        "--header-insertion=never",
        "--background-index"
    ],
    "clangd.path": "clangd"
}
```

clangd 内置了 clang-tidy 集成——它会自动读取项目根目录的 `.clang-tidy` 文件，在编辑时实时显示 clang-tidy 警告（黄色波浪线），并提供"快速修复"（Quick Fix）按钮。

### CLion

CLion 原生支持 clang-tidy，无需任何额外配置：

1. `Settings` → `Editor` → `Inspections` → `C/C++` → `General` → `Clang-Tidy`
2. CLion 会自动检测项目根目录的 `.clang-tidy` 文件
3. KrKr2 的 `.clang-tidy` 就是从 CLion 的 Inspection 设置导出的
4. 警告直接显示在编辑器中，可通过 `Alt+Enter` 快速修复

### Visual Studio

Visual Studio 2019+ 支持 clang-tidy 集成：

1. `工具` → `选项` → `文本编辑器` → `C/C++` → `代码样式` → `Clang-Tidy`
2. 勾选"启用 Clang-Tidy"
3. 选择"使用 .clang-tidy 文件中的检查"
4. 确保项目已生成 `compile_commands.json`（或使用 Visual Studio 的内置分析）

---

## 动手实践

### 实践 1：生成编译数据库并运行首次分析

```bash
# 1. 进入 KrKr2 项目根目录
cd /path/to/krkr2

# 2. 使用 CMake 生成编译数据库
cmake -S . -B build-tidy \
    -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
    -DCMAKE_CXX_STANDARD=17

# 3. 创建符号链接让 clang-tidy 在项目根目录找到编译数据库
# Linux/macOS:
ln -sf build-tidy/compile_commands.json compile_commands.json
# Windows (管理员 PowerShell):
# New-Item -ItemType SymbolicLink -Path compile_commands.json -Target build-tidy\compile_commands.json

# 4. 对单个文件运行 clang-tidy
clang-tidy -p build-tidy cpp/core/utils/TimerThread.cpp

# 5. 观察输出：记录有多少 warning，分别属于哪些类别
```

### 实践 2：对比"修复前后"的代码

```bash
# 1. 选择一个文件（先备份）
cp cpp/core/utils/ClipboardImpl.cpp cpp/core/utils/ClipboardImpl.cpp.bak

# 2. 运行自动修复
clang-tidy -p build-tidy --fix cpp/core/utils/ClipboardImpl.cpp

# 3. 查看修改了什么
diff cpp/core/utils/ClipboardImpl.cpp.bak cpp/core/utils/ClipboardImpl.cpp

# 4. 恢复原文件（不要提交自动修复的结果，除非团队同意）
mv cpp/core/utils/ClipboardImpl.cpp.bak cpp/core/utils/ClipboardImpl.cpp
```

### 实践 3：只运行单一类别检查

```bash
# 只看 modernize 类建议（代码现代化）
clang-tidy -p build-tidy \
    -checks='-*,modernize-*' \
    cpp/core/tjs2/tjsLex.cpp 2>&1 | head -50

# 只看 performance 类建议（性能优化）
clang-tidy -p build-tidy \
    -checks='-*,performance-*' \
    cpp/core/visual/LayerBitmapImpl.cpp 2>&1 | head -50

# 统计各类别的 warning 数量
clang-tidy -p build-tidy cpp/core/tjs2/tjsLex.cpp 2>&1 \
    | grep "warning:" \
    | grep -oP '\[\K[^\]]+' \
    | sort | uniq -c | sort -rn
```

> **Windows 用户：** 上面的 `grep -oP` 在 Windows 不可用。在 Git Bash 中可用 `grep -o '\[[^]]*\]'` 替代；或用 PowerShell：
> ```powershell
> clang-tidy -p build-tidy cpp\core\tjs2\tjsLex.cpp 2>&1 |
>     Select-String -Pattern '\[([^\]]+)\]' -AllMatches |
>     ForEach-Object { $_.Matches.Groups[1].Value } |
>     Group-Object | Sort-Object Count -Descending
> ```

### 实践 4：编写自定义 .clang-tidy 配置

创建一个更严格的本地配置，体验不同检查规则的效果：

```yaml
# 保存为 my-strict.clang-tidy（放在任意目录）
---
Checks: '-*,
  bugprone-*,
  modernize-*,
  performance-*,
  readability-*,
  cppcoreguidelines-*'
WarningsAsErrors: 'bugprone-use-after-move,bugprone-dangling-handle'
HeaderFilterRegex: '.*'
CheckOptions:
  - key: modernize-use-nullptr.NullMacros
    value: 'NULL'
  - key: readability-identifier-naming.ClassCase
    value: CamelCase
```

```bash
# 使用自定义配置运行（--config-file 指定配置文件路径）
clang-tidy -p build-tidy \
    --config-file=my-strict.clang-tidy \
    cpp/core/base/XP3Archive.cpp
```

---

## 对照项目源码

KrKr2 项目中与 clang-tidy 相关的文件：

| 文件路径 | 行数 | 说明 |
|----------|------|------|
| `.clang-tidy` | 142 行 | 项目 clang-tidy 配置，白名单模式启用 100+ 条规则 |
| `.clang-format` | 55 行 | 与 clang-tidy 互补——format 管外观，tidy 管语义 |
| `.github/workflows/code-format-check.yml` | 40 行 | CI 目前只检查 clang-format，**未启用 clang-tidy 门禁** |

**关键观察：** 当前 KrKr2 的 CI 流水线**只做格式检查，不做静态分析**。`.clang-tidy` 文件的存在说明开发者在本地使用 CLion 进行静态分析（配置是从 CLion 导出的），但这些检查没有集成到 CI 中。这意味着：

1. 只有使用 CLion 的开发者会看到 clang-tidy 警告
2. 使用其他编辑器的开发者可能无意中引入 clang-tidy 能检测到的问题
3. 将 clang-tidy 集成到 CI 是一个有价值的改进方向（但需要先清理现有代码中的所有 warning）

---

## 常见错误与排查

### 错误 1：找不到编译数据库

```
Error while trying to load a compilation database:
Could not auto-detect compilation database for file "xxx.cpp"
```

**原因：** clang-tidy 在当前目录和父目录中找不到 `compile_commands.json`。

**解决：**
```bash
# 确认文件存在
ls build/compile_commands.json

# 方法 1：用 -p 参数指定路径
clang-tidy -p build xxx.cpp

# 方法 2：在项目根目录创建符号链接
ln -sf build/compile_commands.json .
```

### 错误 2：大量无关的头文件警告

```
/usr/include/c++/12/bits/stl_vector.h:389:7: warning: ...
```

**原因：** clang-tidy 默认也分析系统头文件和第三方库头文件。

**解决：** 在 `.clang-tidy` 中添加 `HeaderFilterRegex` 限制只分析项目头文件：

```yaml
---
Checks: '-*,bugprone-*,...'
HeaderFilterRegex: 'cpp/core/|cpp/plugins/|platforms/'
```

### 错误 3：clang-tidy 运行极慢

clang-tidy 对每个文件都执行完整的编译前端解析，项目越大越慢。

**缓解措施：**
```bash
# 1. 只分析修改的文件（结合 Git）
git diff --name-only HEAD~1 -- '*.cpp' | \
    xargs -P$(nproc) -I{} clang-tidy -p build {}

# 2. 使用 run-clang-tidy 的并行功能
run-clang-tidy -p build -j$(nproc) cpp/core/

# 3. 减少检查规则（越少越快）
clang-tidy -p build -checks='-*,bugprone-use-after-move' file.cpp
```

---

## 本节小结

- **clang-tidy** 是基于 Clang 编译器前端的 C++ 静态分析工具，在 AST（抽象语法树）层面检测代码缺陷、安全隐患和风格问题
- 与 clang-format（关注代码外观）不同，clang-tidy 关注代码的**语义正确性**——它能发现"编译通过但几乎肯定是 bug"的代码模式
- KrKr2 的 `.clang-tidy` 使用**白名单模式**（`-*` 先禁用全部，再逐条启用），从 CLion 的 Inspection 设置导出，共启用 100+ 条规则
- **八大检查类别：** bugprone（缺陷检测，49 条最多）、modernize（C++11/14/17 升级，22 条）、readability（可读性，17 条）、performance（性能，13 条）、cert（安全编码，9 条）、cppcoreguidelines（核心指南，5 条）、misc（杂项，6 条）、google/hicpp（特定标准，4 条）
- 运行 clang-tidy **需要编译数据库**（`compile_commands.json`），通过 CMake 的 `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` 生成
- 当前 KrKr2 的 CI 流水线**未集成 clang-tidy 检查**——`.clang-tidy` 仅供本地 IDE 使用
- 三大编辑器（VS Code + clangd、CLion 原生支持、Visual Studio 内置）都支持 clang-tidy 实时检查

---

## 练习题与答案

### 题目 1：判断 clang-tidy 会报告哪些警告

阅读以下代码，判断 KrKr2 的 `.clang-tidy` 配置会报告哪些警告，并说明每条警告属于哪个类别：

```cpp
#include <stdlib.h>
#include <vector>
#include <string>

class Animal {
public:
    int age;
    virtual void speak() { }
    virtual ~Animal() {}
};

class Dog : public Animal {
public:
    virtual void speak() { }
};

int main() {
    const char* str = "42";
    int value = atoi(str);

    std::vector<int> data;
    for(int i = 0; i < 100; ++i) {
        data.push_back(i);
    }

    Dog dog;
    Animal a = dog;

    if(data.size() == 0) {
        return 1;
    }
    return 0;
}
```

<details>
<summary>查看答案</summary>

这段代码会触发以下 clang-tidy 警告：

1. **`modernize-deprecated-headers`** — `#include <stdlib.h>` 应改为 `#include <cstdlib>`。C++ 推荐使用 C 标准库的 `<c*>` 版本，它将所有符号放入 `std` 命名空间。

2. **`cppcoreguidelines-pro-type-member-init`** — `Animal` 类的 `age` 成员变量未初始化。应在类内声明时给默认值（`int age = 0;`）或在构造函数中初始化。

3. **`modernize-use-override`** — `Dog::speak()` 重写了基类虚函数但没有标记 `override`。应改为 `void speak() override { }`。

4. **`cert-err34-c`** — `atoi(str)` 无法检测转换失败。应使用 `std::stoi(str)`。

5. **`modernize-use-nullptr`** — 虽然这段代码没有直接使用 `NULL`，但如果有其他地方使用了 `NULL` 或 `0` 作为指针，也会触发。

6. **`performance-inefficient-vector-operation`** — `push_back` 在循环中反复调用，每次可能触发内存重新分配。应在循环前调用 `data.reserve(100)` 预分配内存。

7. **`cppcoreguidelines-slicing`** — `Animal a = dog;` 将 `Dog` 对象"切片"为 `Animal` 对象，`Dog` 独有的数据被丢弃。

8. **`readability-container-size-empty`** — `data.size() == 0` 应改为 `data.empty()`。

修复后的代码：

```cpp
#include <cstdlib>    // modernize-deprecated-headers: <stdlib.h> → <cstdlib>
#include <vector>
#include <string>
#include <iostream>
#include <stdexcept>

class Animal {
public:
    int age = 0;      // cppcoreguidelines-pro-type-member-init: 初始化成员
    virtual void speak() { }
    virtual ~Animal() = default;  // modernize-use-equals-default
};

class Dog : public Animal {
public:
    void speak() override { }  // modernize-use-override: 添加 override
};

int main() {
    const char* str = "42";
    int value = 0;
    try {
        value = std::stoi(str);  // cert-err34-c: atoi → stoi
    } catch(const std::exception& e) {
        std::cerr << "转换失败: " << e.what() << std::endl;
        return 1;
    }

    std::vector<int> data;
    data.reserve(100);  // performance-inefficient-vector-operation: 预分配
    for(int i = 0; i < 100; ++i) {
        data.push_back(i);
    }

    Dog dog;
    const Animal& a = dog;  // cppcoreguidelines-slicing: 用引用避免切片

    if(data.empty()) {  // readability-container-size-empty: size()==0 → empty()
        return 1;
    }

    std::cout << "value=" << value << ", age=" << a.age << std::endl;
    return 0;
}
```

</details>

### 题目 2：为 KrKr2 编写 clang-tidy CI 工作流

KrKr2 目前的 CI 只有 clang-format 检查，没有 clang-tidy 检查。请编写一个 GitHub Actions 工作流文件 `clang-tidy-check.yml`，满足：
1. 在 push 到 main 和 PR 时触发
2. 使用 `compile_commands.json` 运行 clang-tidy
3. 只检查本次 PR/push 中修改的 `.cpp` 和 `.h` 文件
4. 使用项目根目录的 `.clang-tidy` 配置

<details>
<summary>查看答案</summary>

```yaml
# .github/workflows/clang-tidy-check.yml
# 功能：在 PR 和 push 时对修改的 C++ 文件运行 clang-tidy 静态分析
name: Clang-Tidy Check

on:
  push:
    branches: [main]
    paths:
      - 'cpp/**'
      - 'platforms/**'
  pull_request:
    branches: [main]
    paths:
      - 'cpp/**'
      - 'platforms/**'

jobs:
  clang-tidy:
    runs-on: ubuntu-latest
    steps:
      # 1. 检出代码（fetch-depth: 0 获取完整历史，用于 diff）
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # 2. 安装 clang-tidy 和项目依赖
      - name: Install clang-tidy
        run: |
          sudo apt-get update
          sudo apt-get install -y clang-tidy cmake ninja-build

      # 3. 配置 CMake 并生成编译数据库
      - name: Generate compile_commands.json
        run: |
          cmake -S . -B build \
            -G Ninja \
            -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
            -DCMAKE_CXX_STANDARD=17 \
            -DENABLE_TESTS=OFF

      # 4. 获取本次变更的 C++ 文件列表
      - name: Get changed files
        id: changed
        run: |
          if [ "${{ github.event_name }}" = "pull_request" ]; then
            # PR：对比 PR 基础分支和当前分支
            BASE=${{ github.event.pull_request.base.sha }}
          else
            # Push：对比上一次提交
            BASE=${{ github.event.before }}
          fi

          CHANGED=$(git diff --name-only --diff-filter=ACMR "$BASE" HEAD \
            | grep -E '\.(cpp|h|hpp|cc)$' || true)

          if [ -z "$CHANGED" ]; then
            echo "files=" >> $GITHUB_OUTPUT
            echo "has_files=false" >> $GITHUB_OUTPUT
          else
            # 将文件列表存为单行（空格分隔）
            echo "files=$(echo $CHANGED | tr '\n' ' ')" >> $GITHUB_OUTPUT
            echo "has_files=true" >> $GITHUB_OUTPUT
          fi

      # 5. 运行 clang-tidy（仅当有变更文件时）
      - name: Run clang-tidy
        if: steps.changed.outputs.has_files == 'true'
        run: |
          echo "检查以下文件："
          echo "${{ steps.changed.outputs.files }}" | tr ' ' '\n'
          echo "---"

          ERRORS=0
          for file in ${{ steps.changed.outputs.files }}; do
            if [ -f "$file" ]; then
              echo "::group::Checking $file"
              if ! clang-tidy -p build "$file" 2>&1; then
                ERRORS=$((ERRORS + 1))
              fi
              echo "::endgroup::"
            fi
          done

          if [ $ERRORS -gt 0 ]; then
            echo "::error::clang-tidy 发现 $ERRORS 个文件有问题"
            exit 1
          fi

          echo "✅ 所有文件通过 clang-tidy 检查"

      - name: Skip (no C++ files changed)
        if: steps.changed.outputs.has_files == 'false'
        run: echo "没有 C++ 文件被修改，跳过 clang-tidy 检查"
```

**设计说明：**

1. **增量检查**：只分析本次变更的文件，避免对整个项目运行 clang-tidy（太慢）
2. **路径过滤**：`paths` 条件确保只有 C++ 代码变更才触发此工作流
3. **编译数据库**：必须先运行 CMake 生成 `compile_commands.json`，否则 clang-tidy 无法工作
4. **与 format check 独立**：这是单独的工作流，不阻塞格式检查和构建

</details>

### 题目 3：解释 `-*` 白名单模式 vs 黑名单模式的权衡

KrKr2 的 `.clang-tidy` 使用 `-*` 白名单模式（先禁用所有，再逐条启用）。有些项目使用黑名单模式（默认启用所有，再逐条禁用不需要的）。请比较两种模式的优缺点，并解释为什么 KrKr2 选择了白名单模式。

<details>
<summary>查看答案</summary>

**白名单模式（KrKr2 使用）：**
```yaml
Checks: '-*,bugprone-use-after-move,modernize-use-override,...'
```

| 方面 | 优点 | 缺点 |
|------|------|------|
| 噪音控制 | ✅ 只看到自己明确启用的规则，zero noise | ❌ 新版 clang-tidy 添加的有用规则不会自动启用 |
| 可预测性 | ✅ 升级 clang-tidy 版本不会突然出现新的 warning | ❌ 需要定期审查新规则并手动添加 |
| 维护成本 | ❌ 需要列出所有想要的规则（KrKr2 列了 100+ 条） | — |
| 适用场景 | ✅ 大型项目、遗留代码（无法一次性修复所有 warning） | — |

**黑名单模式：**
```yaml
Checks: '*,-google-readability-todo,-fuchsia-*,...'
```

| 方面 | 优点 | 缺点 |
|------|------|------|
| 覆盖面 | ✅ 自动获得新规则的检查 | ❌ 升级时可能突然出现大量新 warning |
| 维护成本 | ✅ 只需列出不想要的规则，通常比白名单少 | ❌ 需要频繁更新禁用列表 |
| 噪音 | ❌ 可能包含与项目无关的规则（如 `mpi-*` 对不使用 MPI 的项目） | — |
| 适用场景 | ✅ 新项目、严格要求代码质量的项目 | — |

**KrKr2 选择白名单模式的原因：**

1. **遗留代码库：** KrKr2 基于 KiriKiri 引擎的 C++ 代码，历史悠久，无法一次性修复所有潜在问题。白名单模式让团队可以逐步增加检查规则。

2. **CLion 生成：** 配置是从 CLion 的 Inspection 设置导出的——CLion 的 UI 天然是"勾选想要的检查"（白名单思维），导出后就是 `-*` + 逐条启用的格式。

3. **CI 尚未集成：** 因为 clang-tidy 没有集成到 CI 中（只在本地使用），白名单模式的"不自动获得新规则"缺点影响不大——开发者通过 CLion 更新 Inspection 设置并重新导出即可。

4. **精确控制：** 100+ 条规则都是经过筛选的有价值检查，每条都有明确的启用理由。这比"启用全部然后禁掉不需要的"更有意识（intentional）。

</details>

---

## 下一步

下一节我们将学习 **vcpkg 二进制缓存与依赖管理**——了解 KrKr2 如何使用 vcpkg 的清单模式管理 C++ 依赖、如何通过 NuGet 二进制缓存加速 CI 构建、以及 overlay ports 自定义端口的使用方法。

👉 [03-vcpkg 二进制缓存与依赖管理](./03-vcpkg二进制缓存与依赖管理.md)
```
```

