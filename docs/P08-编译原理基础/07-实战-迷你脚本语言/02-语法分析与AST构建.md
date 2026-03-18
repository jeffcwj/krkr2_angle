# 语法分析与 AST 构建

> **所属模块：** P08-编译原理基础
> **前置知识：** [语言设计与词法器实现](01-语言设计与词法器实现.md)、[递归下降 Parser](../03-语法分析/01-递归下降Parser.md)
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：

1. 为 MiniScript 设计完整的 AST 节点体系——用 `std::variant` + `std::unique_ptr` 表示
2. 实现递归下降 Parser，每条 EBNF 产生式对应一个解析函数
3. 正确处理运算符优先级和结合性
4. 实现精确到行列号的 Parser 错误报告

---

## AST 节点设计

AST（抽象语法树）是源代码的树形表示，去除括号、分号等语法噪声，只保留**语义结构**：

```
源码: var x = 1 + 2 * 3;

AST:  VarDecl("x")
       └── BinaryExpr("+")
            ├── NumberLit(1)
            └── BinaryExpr("*")
                 ├── NumberLit(2)
                 └── NumberLit(3)
```

### 表达式节点

```cpp
// mini_ast.h
#pragma once
#include <string>
#include <vector>
#include <memory>
#include <variant>

struct Expr;
struct Stmt;
using ExprPtr = std::unique_ptr<Expr>;
using StmtPtr = std::unique_ptr<Stmt>;

// ---- 表达式 ----
struct NumberLit  { double value; };
struct StringLit  { std::string value; };
struct BoolLit    { bool value; };
struct Identifier { std::string name; };
struct UnaryExpr  { std::string op; ExprPtr operand; };
struct BinaryExpr { std::string op; ExprPtr left, right; };
struct AssignExpr { std::string name; ExprPtr value; };
struct CallExpr   { ExprPtr callee; std::vector<ExprPtr> args; };

struct Expr {
    std::variant<NumberLit, StringLit, BoolLit, Identifier,
                 UnaryExpr, BinaryExpr, AssignExpr, CallExpr> node;
};

// 辅助构造——避免繁琐的 make_unique + variant 初始化
inline ExprPtr makeExpr(auto&& val) {
    auto e = std::make_unique<Expr>();
    e->node = std::forward<decltype(val)>(val);
    return e;
}
```

### 语句节点

```cpp
// ---- 语句 ----
struct ExprStmt   { ExprPtr expr; };
struct PrintStmt  { ExprPtr expr; };
struct VarDecl    { std::string name; ExprPtr init; };  // init 可为 nullptr
struct ReturnStmt { ExprPtr value; };
struct Block      { std::vector<StmtPtr> stmts; };
struct IfStmt     { ExprPtr condition; StmtPtr thenBranch; StmtPtr elseBranch; };
struct WhileStmt  { ExprPtr condition; StmtPtr body; };
struct FnDecl     { std::string name; std::vector<std::string> params; StmtPtr body; };

struct Stmt {
    std::variant<ExprStmt, PrintStmt, VarDecl, ReturnStmt,
                 Block, IfStmt, WhileStmt, FnDecl> node;
};

inline StmtPtr makeStmt(auto&& val) {
    auto s = std::make_unique<Stmt>();
    s->node = std::forward<decltype(val)>(val);
    return s;
}
```

**设计决策：** `std::variant` 不需要虚函数表，编译期类型安全。TJS2 没有独立 AST——Bison 的语法动作直接在 `tjs.y` 中生成字节码（单遍编译）。MiniScript 分两步（AST → 字节码），更易理解。

---

## Parser 类结构

```cpp
// mini_parser.h
#pragma once
#include "mini_token.h"
#include "mini_ast.h"
#include <vector>
#include <stdexcept>

class Parser {
    std::vector<Token> tokens;
    int pos = 0;
public:
    explicit Parser(std::vector<Token> toks) : tokens(std::move(toks)) {}
    std::vector<StmtPtr> parseProgram();
private:
    const Token& peek() const { return tokens[pos]; }
    const Token& previous() const { return tokens[pos - 1]; }
    bool check(TokenType t) const { return peek().type == t; }
    Token advance() {
        Token tok = tokens[pos];
        if (tok.type != TokenType::Eof) pos++;
        return tok;
    }
    bool match(TokenType t) {
        if (!check(t)) return false;
        advance(); return true;
    }
    Token expect(TokenType t, const std::string& msg) {
        if (check(t)) return advance();
        auto& tok = peek();
        throw std::runtime_error(
            "L" + std::to_string(tok.loc.line) + ":" +
            std::to_string(tok.loc.column) + " " + msg +
            ", got '" + tok.lexeme + "'");
    }

    // 声明 / 语句
    StmtPtr declaration();
    StmtPtr varDeclaration();
    StmtPtr fnDeclaration();
    StmtPtr statement();
    StmtPtr ifStatement();
    StmtPtr whileStatement();
    StmtPtr printStatement();
    StmtPtr returnStatement();
    StmtPtr blockStatement();

    // 表达式——每个优先级一个函数
    ExprPtr expression();   // → assignment
    ExprPtr assignment();   // → logicOr （右结合）
    ExprPtr logicOr();      // → logicAnd { "or" logicAnd }
    ExprPtr logicAnd();     // → equality { "and" equality }
    ExprPtr equality();     // → comparison { ("=="|"!=") comparison }
    ExprPtr comparison();   // → addition { ("<"|">"|"<="|">=") addition }
    ExprPtr addition();     // → multiplication { ("+"|"-") multiplication }
    ExprPtr multiplication(); // → unary { ("*"|"/"|"%") unary }
    ExprPtr unary();        // → ("-"|"not") unary | call
    ExprPtr call();         // → primary { "(" argList ")" }
    ExprPtr primary();      // → NUMBER | STRING | IDENT | "(" expr ")"
};
```

---

## 表达式解析——优先级递归

核心思想：**优先级从低到高，每个优先级是一个函数，函数内调用下一级**。函数调用栈自然实现优先级：

```cpp
ExprPtr Parser::expression() { return assignment(); }

ExprPtr Parser::assignment() {
    ExprPtr expr = logicOr();
    if (match(TokenType::Equal)) {
        ExprPtr val = assignment();  // 右结合——递归调用自己
        if (auto* id = std::get_if<Identifier>(&expr->node))
            return makeExpr(AssignExpr{id->name, std::move(val)});
        throw std::runtime_error("invalid assignment target");
    }
    return expr;
}

// logicOr / logicAnd / equality / comparison / addition / multiplication
// 结构完全相同——只是运算符和下一级函数不同
ExprPtr Parser::logicOr() {
    ExprPtr left = logicAnd();
    while (match(TokenType::Or)) {
        ExprPtr right = logicAnd();
        left = makeExpr(BinaryExpr{"or", std::move(left), std::move(right)});
    }
    return left;
}

ExprPtr Parser::logicAnd() {
    ExprPtr left = equality();
    while (match(TokenType::And)) {
        ExprPtr right = equality();
        left = makeExpr(BinaryExpr{"and", std::move(left), std::move(right)});
    }
    return left;
}

ExprPtr Parser::equality() {
    ExprPtr left = comparison();
    while (match(TokenType::EqualEqual) || match(TokenType::BangEqual)) {
        std::string op = previous().lexeme;
        left = makeExpr(BinaryExpr{op, std::move(left), comparison()});
    }
    return left;
}

ExprPtr Parser::comparison() {
    ExprPtr left = addition();
    while (match(TokenType::Less) || match(TokenType::LessEqual) ||
           match(TokenType::Greater) || match(TokenType::GreaterEqual)) {
        std::string op = previous().lexeme;
        left = makeExpr(BinaryExpr{op, std::move(left), addition()});
    }
    return left;
}

ExprPtr Parser::addition() {
    ExprPtr left = multiplication();
    while (match(TokenType::Plus) || match(TokenType::Minus)) {
        std::string op = previous().lexeme;
        left = makeExpr(BinaryExpr{op, std::move(left), multiplication()});
    }
    return left;
}

ExprPtr Parser::multiplication() {
    ExprPtr left = unary();
    while (match(TokenType::Star) || match(TokenType::Slash) ||
           match(TokenType::Percent)) {
        std::string op = previous().lexeme;
        left = makeExpr(BinaryExpr{op, std::move(left), unary()});
    }
    return left;
}
```

### 一元表达式和函数调用

```cpp
ExprPtr Parser::unary() {
    if (match(TokenType::Minus))
        return makeExpr(UnaryExpr{"-", unary()});
    if (match(TokenType::Not))
        return makeExpr(UnaryExpr{"not", unary()});
    return call();
}

ExprPtr Parser::call() {
    ExprPtr expr = primary();
    while (match(TokenType::LeftParen)) {  // 支持 f(1)(2) 连续调用
        std::vector<ExprPtr> args;
        if (!check(TokenType::RightParen)) {
            do { args.push_back(expression()); } while (match(TokenType::Comma));
        }
        expect(TokenType::RightParen, "expected ')'");
        expr = makeExpr(CallExpr{std::move(expr), std::move(args)});
    }
    return expr;
}

ExprPtr Parser::primary() {
    if (match(TokenType::Number))
        return makeExpr(NumberLit{std::stod(previous().lexeme)});
    if (match(TokenType::String))
        return makeExpr(StringLit{previous().lexeme});
    if (match(TokenType::True))  return makeExpr(BoolLit{true});
    if (match(TokenType::False)) return makeExpr(BoolLit{false});
    if (match(TokenType::Identifier))
        return makeExpr(Identifier{previous().lexeme});
    if (match(TokenType::LeftParen)) {
        ExprPtr e = expression();
        expect(TokenType::RightParen, "expected ')'");
        return e;
    }
    throw std::runtime_error("L" + std::to_string(peek().loc.line) +
        " unexpected '" + peek().lexeme + "'");
}
```

---

## 语句解析

```cpp
std::vector<StmtPtr> Parser::parseProgram() {
    std::vector<StmtPtr> prog;
    while (!check(TokenType::Eof)) prog.push_back(declaration());
    return prog;
}

StmtPtr Parser::declaration() {
    if (match(TokenType::Var)) return varDeclaration();
    if (match(TokenType::Fn))  return fnDeclaration();
    return statement();
}

StmtPtr Parser::varDeclaration() {
    Token name = expect(TokenType::Identifier, "expected variable name");
    ExprPtr init = match(TokenType::Equal) ? expression() : nullptr;
    expect(TokenType::Semicolon, "expected ';'");
    return makeStmt(VarDecl{name.lexeme, std::move(init)});
}

StmtPtr Parser::fnDeclaration() {
    Token name = expect(TokenType::Identifier, "expected function name");
    expect(TokenType::LeftParen, "expected '('");
    std::vector<std::string> params;
    if (!check(TokenType::RightParen)) {
        do {
            params.push_back(
                expect(TokenType::Identifier, "expected param").lexeme);
        } while (match(TokenType::Comma));
    }
    expect(TokenType::RightParen, "expected ')'");
    return makeStmt(FnDecl{name.lexeme, std::move(params), blockStatement()});
}

StmtPtr Parser::statement() {
    if (match(TokenType::If))     return ifStatement();
    if (match(TokenType::While))  return whileStatement();
    if (match(TokenType::Print))  return printStatement();
    if (match(TokenType::Return)) return returnStatement();
    if (check(TokenType::LeftBrace)) return blockStatement();
    ExprPtr e = expression();
    expect(TokenType::Semicolon, "expected ';'");
    return makeStmt(ExprStmt{std::move(e)});
}

StmtPtr Parser::ifStatement() {
    expect(TokenType::LeftParen, "expected '('");
    ExprPtr cond = expression();
    expect(TokenType::RightParen, "expected ')'");
    StmtPtr then = blockStatement();
    StmtPtr els = match(TokenType::Else) ? blockStatement() : nullptr;
    return makeStmt(IfStmt{std::move(cond), std::move(then), std::move(els)});
}

StmtPtr Parser::whileStatement() {
    expect(TokenType::LeftParen, "expected '('");
    ExprPtr cond = expression();
    expect(TokenType::RightParen, "expected ')'");
    return makeStmt(WhileStmt{std::move(cond), blockStatement()});
}

StmtPtr Parser::printStatement() {
    expect(TokenType::LeftParen, "expected '('");
    ExprPtr e = expression();
    expect(TokenType::RightParen, "expected ')'");
    expect(TokenType::Semicolon, "expected ';'");
    return makeStmt(PrintStmt{std::move(e)});
}

StmtPtr Parser::returnStatement() {
    ExprPtr val = check(TokenType::Semicolon) ? nullptr : expression();
    expect(TokenType::Semicolon, "expected ';'");
    return makeStmt(ReturnStmt{std::move(val)});
}

StmtPtr Parser::blockStatement() {
    expect(TokenType::LeftBrace, "expected '{'");
    std::vector<StmtPtr> stmts;
    while (!check(TokenType::RightBrace) && !check(TokenType::Eof))
        stmts.push_back(declaration());
    expect(TokenType::RightBrace, "expected '}'");
    return makeStmt(Block{std::move(stmts)});
}
```

### 常见错误与解决方案

**错误 1：左结合写成右递归**

```cpp
// ❌ 1+2+3 变成 1+(2+3)
ExprPtr addition() {
    ExprPtr left = multiplication();
    if (match(Plus)) return makeExpr(BinaryExpr{"+", move(left), addition()});
    return left;
}
// ✅ 用 while 循环实现左结合
while (match(Plus) || match(Minus)) { left = makeExpr(...); }
```

**错误 2：赋值左值检查遗漏**

```cpp
// ❌ 允许 3 = 5
if (match(Equal)) return makeExpr(AssignExpr{???, assignment()});
// ✅ 检查左侧是 Identifier
if (auto* id = std::get_if<Identifier>(&expr->node)) { ... }
```

**错误 3：`match()` 的短路求值陷阱**

```cpp
// match(Less) 和 match(LessEqual) 的顺序不影响——
// 因为 Lexer 已把 "<=" 合并为一个 LessEqual Token。
// 但如果 Lexer 没有合并（如某些简单实现），就必须先检查 LessEqual
```

---

## 动手实践

### AST 打印器验证

```cpp
void printExpr(const Expr& e, int depth = 0) {
    std::string pad(depth * 2, ' ');
    std::visit([&](auto&& n) {
        using T = std::decay_t<decltype(n)>;
        if constexpr (std::is_same_v<T, NumberLit>)
            std::cout << pad << "Num(" << n.value << ")\n";
        else if constexpr (std::is_same_v<T, Identifier>)
            std::cout << pad << "Id(" << n.name << ")\n";
        else if constexpr (std::is_same_v<T, BinaryExpr>) {
            std::cout << pad << "Bin(" << n.op << ")\n";
            printExpr(*n.left, depth+1);
            printExpr(*n.right, depth+1);
        } else if constexpr (std::is_same_v<T, CallExpr>) {
            std::cout << pad << "Call\n";
            printExpr(*n.callee, depth+1);
            for (auto& a : n.args) printExpr(*a, depth+1);
        }
        // StringLit, BoolLit, UnaryExpr, AssignExpr 类似
    }, e.node);
}

int main() {
    Lexer lexer("var x = 1 + 2 * 3; print(x);");
    Parser parser(lexer.scanAll());
    auto prog = parser.parseProgram();
    // 遍历打印每条语句的 AST...
}
```

### 预期输出

```
VarDecl(x):
  Bin(+)
    Num(1)
    Bin(*)
      Num(2)
      Num(3)
Print:
  Id(x)
```

`1 + 2 * 3` 被正确解析为 `1 + (2 * 3)`——`*` 优先级高于 `+`，在 AST 中 `*` 是 `+` 的右子树。

### 更复杂的测试

```cpp
// 测试 if-else
Parser p2(Lexer("if (x > 0) { print(x); } else { print(0); }").scanAll());
// AST: IfStmt { cond: Bin(">", Id(x), Num(0)), then: Block[Print(Id(x))], else: Block[Print(Num(0))] }

// 测试嵌套函数调用
Parser p3(Lexer("print(fib(n - 1) + fib(n - 2));").scanAll());
// AST: Print { Bin("+", Call(Id(fib), [Bin("-",Id(n),Num(1))]), Call(Id(fib), [Bin("-",Id(n),Num(2))])) }
```

---

## 对照项目源码

| 环节 | MiniScript | TJS2 (`tjs.y`) |
|------|-----------|----------------|
| Parser 类型 | 手写递归下降 | Bison LALR(1) |
| 优先级实现 | 函数调用栈层级 | `%left`/`%right` 声明 |
| AST | 显式 Expr/Stmt 节点 | 无 AST——语法动作直接生成字节码 |
| 错误报告 | `expect()` 抛异常 | `yyerror()` + `TJSThrowSyntaxError` |

相关文件：
- `cpp/core/tjs2/bison/tjs.y` 第 1-100 行 — `%token` 和优先级声明
- `cpp/core/tjs2/bison/tjs.y` 第 200-500 行 — 表达式产生式

---

## 本节小结

- AST 用 `std::variant` + `unique_ptr` 表达，类型安全、自动内存管理
- 递归下降核心：**每个优先级一个函数，while 循环=左结合，递归调用=右结合**
- `expect()` 是 Parser 的错误守卫，配合 `SourceLoc` 给出精确诊断
- 六种二元运算优先级的解析函数结构完全相同——只是运算符和下一级函数不同

---

## 练习题与答案

### 题目 1：实现 panic-mode 错误恢复

当前 Parser 遇错即停。实现 `synchronize()` 函数：跳到下一个 `;` 或语句关键字后继续解析。

<details>
<summary>查看答案</summary>

```cpp
void Parser::synchronize() {
    advance();
    while (!check(TokenType::Eof)) {
        if (previous().type == TokenType::Semicolon) return;
        switch (peek().type) {
            case TokenType::Var: case TokenType::Fn:
            case TokenType::If:  case TokenType::While:
            case TokenType::Print: case TokenType::Return:
                return;
            default: advance();
        }
    }
}

// 在 parseProgram 中包裹 try-catch：
std::vector<StmtPtr> Parser::parseProgram() {
    std::vector<StmtPtr> prog;
    while (!check(TokenType::Eof)) {
        try { prog.push_back(declaration()); }
        catch (const std::runtime_error& e) {
            std::cerr << "Error: " << e.what() << "\n";
            synchronize();
        }
    }
    return prog;
}
```

</details>

### 题目 2：添加 `!=` 的去语法糖

在 Parser 层将 `a != b` 转换为 `not (a == b)` 的 AST，让后端只需处理 `==` 和 `not`。

<details>
<summary>查看答案</summary>

```cpp
ExprPtr Parser::equality() {
    ExprPtr left = comparison();
    while (match(TokenType::EqualEqual) || match(TokenType::BangEqual)) {
        bool notEq = (previous().type == TokenType::BangEqual);
        ExprPtr eq = makeExpr(BinaryExpr{"==", std::move(left), comparison()});
        left = notEq ? makeExpr(UnaryExpr{"not", std::move(eq)}) : std::move(eq);
    }
    return left;
}
```

好处：Compiler 不需要单独的 `NEQ` 字节码——`NOT` + `EQ` 组合即可。

</details>

---

## 下一步

[字节码编译器](03-字节码编译器.md) —— 遍历 AST，将 MiniScript 程序编译为 `BytecodeChunk` 序列，生成可被 VM 执行的指令流。
