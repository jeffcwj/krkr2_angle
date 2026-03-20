# clang-format 代码格式化

> **所属模块：** M11-CI/CD与Docker
> **前置知识：** [GitHub Actions 基础与 YAML 语法](../01-GitHub-Actions工作流/01-GitHub-Actions基础与YAML语法.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 clang-format 的工作原理和配置文件语法
2. 逐项解读 KrKr2 项目的 `.clang-format` 配置文件中每一条规则
3. 理解 CI 中 `code-format-check.yml` 的格式门禁工作机制
4. 在 VS Code、CLion、Visual Studio 等编辑器中配置自动格式化
5. 手动运行 clang-format 批量格式化项目代码
6. 排查格式检查失败的常见原因

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| clang-format | clang-format | LLVM 项目提供的 C/C++ 代码格式化工具，根据配置文件自动调整代码的缩进、空格、换行等格式 |
| 基础样式 | BasedOnStyle | `.clang-format` 文件中的基准样式，在此基础上覆盖个别选项——类似 CSS 中的"继承" |
| 列宽限制 | ColumnLimit | 单行最大字符数，超过此限制会自动换行 |
| 大括号换行 | BraceWrapping | 控制 `{` 是否另起一行——不同风格（K&R、Allman、LLVM）对此有不同偏好 |
| 格式禁用 | DisableFormat | 在 `.clang-format` 文件中设置此项为 true，可以让整个目录跳过格式化 |
| 格式门禁 | Format Gate | CI 中的格式检查步骤——如果代码格式不符合规范，PR 无法合并 |

## clang-format 基础

### 什么是 clang-format？

clang-format 是 LLVM 编译器项目的一部分，是一个专门用于 C、C++、Java、JavaScript、Objective-C、Protobuf、C# 等语言的源代码格式化工具。它的核心作用是：**根据预定义的规则自动调整代码的排版格式**，包括缩进宽度、空格位置、换行策略、对齐方式等。

与手动排版相比，clang-format 的优势在于：
1. **一致性**：所有开发者的代码风格完全统一，不会出现"A 喜欢 tab，B 喜欢空格"的问题
2. **自动化**：配合编辑器的"保存时格式化"功能，开发者无需手动调整格式
3. **可 CI 集成**：在持续集成中加入格式检查，不符合规范的代码无法合并

### 安装 clang-format

```bash
# Ubuntu/Debian
sudo apt-get install clang-format
# 或安装特定版本（KrKr2 CI 使用 clang-format 20）
sudo apt-get install clang-format-20

# macOS（Homebrew）
brew install clang-format
# 或通过 llvm 包
brew install llvm
# clang-format 在 /opt/homebrew/opt/llvm/bin/clang-format

# Windows（Chocolatey）
choco install llvm
# clang-format 在 C:\Program Files\LLVM\bin\clang-format.exe

# Windows（Visual Studio）
# Visual Studio 2019+ 自带 clang-format
# 位置：C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\Llvm\bin\clang-format.exe

# 验证安装
clang-format --version
# clang-format version 20.0.0 (https://...)
```

### 基本用法

```bash
# 格式化单个文件（直接修改文件）
clang-format -i src/main.cpp

# 格式化单个文件（输出到标准输出，不修改文件）
clang-format src/main.cpp

# 查看格式化前后的差异（不修改文件）
clang-format src/main.cpp | diff src/main.cpp -

# 批量格式化目录下所有 C++ 文件
# Linux/macOS:
find ./cpp ./platforms ./tests ./tools -regex ".+\.\(cpp\|cc\|h\|hpp\|inc\)" -exec clang-format -i {} +

# Windows (PowerShell):
Get-ChildItem -Path cpp,platforms,tests,tools -Recurse -Include *.cpp,*.h,*.hpp,*.cc,*.inc |
    ForEach-Object { clang-format -i $_.FullName }

# 检查文件是否已格式化（不修改，只返回状态码）
clang-format --dry-run --Werror src/main.cpp
# 退出码 0 = 已格式化
# 退出码 1 = 需要格式化
```

## KrKr2 项目的 .clang-format 配置详解

KrKr2 的 `.clang-format` 文件位于项目根目录（55 行），基于 LLVM 样式定制。以下逐项解读每个配置选项。

### 基础样式与缩进

```yaml
# .clang-format 第 1-8 行

# 基于 LLVM 官方样式，在此基础上覆盖个别选项
# LLVM 样式的默认值：缩进 2 空格、列宽 80、K&R 大括号等
# 其他可选基础：Google、Chromium、Mozilla、WebKit、Microsoft
BasedOnStyle: LLVM

# 缩进宽度 = 4 个空格（覆盖 LLVM 默认的 2 个空格）
IndentWidth: 4

# 制表符宽度 = 4（定义一个 Tab 字符等同于多少个空格的显示宽度）
TabWidth: 4

# 使用空格缩进，不使用 Tab 字符
# Never = 永远用空格
# Always = 永远用 Tab
# ForIndentation = 缩进用 Tab，对齐用空格
UseTab: Never
```

**为什么选 LLVM 而不是 Google？** LLVM 样式更接近传统 C++ 风格（4 空格缩进），而 Google 样式使用 2 空格缩进，在深层嵌套时可读性较差。KrKr2 作为游戏引擎，代码中有不少深层回调和嵌套，4 空格更清晰。

### 列宽与换行

```yaml
# .clang-format 第 10-15 行

# 单行最大字符数 = 80
# 超过 80 字符的行会被自动换行
# 80 是传统的终端宽度，也是大多数编辑器的默认标尺位置
ColumnLimit: 80

# 允许短函数放在一行上
# Empty = 只有空函数体可以放一行：void foo() {}
# 其他选项：None（不允许）、InlineOnly（内联函数可以）、All（所有都可以）
AllowShortFunctionsOnASingleLine: Empty

# 允许短 if 语句放在一行上
# Never = 不允许，即使 if(x) return; 也要拆成两行
AllowShortIfStatementsOnASingleLine: Never

# 允许短循环放在一行上
# false = 不允许，while(x) do_something(); 必须拆成两行
AllowShortLoopsOnASingleLine: false
```

`ColumnLimit: 80` 的效果示例：

```cpp
// 超过 80 字符，clang-format 自动换行
void SomeLongFunctionName(int parameterOne,
                          const std::string& parameterTwo,
                          float parameterThree) {
    // 函数体
}

// 短函数允许单行（AllowShortFunctionsOnASingleLine: Empty）
void Reset() {}           // ✓ 空函数体可以一行
void Reset() { x = 0; }  // ✗ 非空函数体必须展开
```

### 空格规则

```yaml
# .clang-format 第 17-27 行

# 函数名与括号之间不加空格
# Never:  foo(x)    ← KrKr2 选择
# Always: foo (x)
# ControlStatements: if (x) 但 foo(x)
SpaceBeforeParens: Never

# 赋值运算符两边加空格
# true:  x = 1;
# false: x=1;
SpaceBeforeAssignmentOperators: true

# C 风格强制类型转换后不加空格
# false: (int)x      ← KrKr2 选择
# true:  (int) x
SpaceAfterCStyleCast: false

# 模板尖括号内不加空格
# false: vector<int>   ← KrKr2 选择
# true:  vector< int >
SpacesInAngles: false

# 圆括号内不加空格
# false: foo(x, y)     ← KrKr2 选择
# true:  foo( x, y )
SpacesInParentheses: false
```

### 大括号换行策略（重点）

```yaml
# .clang-format 第 29-48 行

# 大括号换行模式 = Custom（自定义每种情况）
# 其他选项：
# Attach = 所有大括号跟在语句后面（K&R 风格）
# Linux = 函数定义另起一行，其他跟在后面
# Allman = 所有大括号都另起一行
# GNU = GNU 风格
BreakBeforeBraces: Custom

# 自定义大括号换行规则
BraceWrapping:
  # 类定义的 { 不换行
  AfterClass: false          # class Foo {   （不是 class Foo\n{）

  # 控制语句（if/for/while/switch）的 { 不换行
  AfterControlStatement: false  # if(x) {   （不是 if(x)\n{）

  # 枚举的 { 不换行
  AfterEnum: false           # enum Color {

  # 函数定义的 { 不换行
  AfterFunction: false       # void foo() {

  # 命名空间的 { 不换行
  AfterNamespace: false      # namespace ns {

  # 结构体的 { 不换行
  AfterStruct: false         # struct Point {

  # 联合体的 { 不换行
  AfterUnion: false          # union Data {

  # catch 的 { 不换行
  BeforeCatch: false         # } catch(e) {

  # else 的 { 不换行
  BeforeElse: false          # } else {
```

所有 `false` 意味着 KrKr2 采用 **K&R 风格**（Kernighan & Ritchie，C 语言发明者在《The C Programming Language》一书中使用的大括号样式）——大括号跟在语句后面，不另起一行。这也是 LLVM 的默认风格。

```cpp
// KrKr2 的大括号风格（K&R / LLVM）
if(condition) {
    doSomething();
} else {
    doOther();
}

// 对比 Allman 风格（如果全部设为 true）
if(condition)
{
    doSomething();
}
else
{
    doOther();
}
```

### 其他格式规则

```yaml
# .clang-format 第 49-55 行

# 命名空间内的代码缩进
# All = 所有内容都缩进（KrKr2 选择）
# None = 命名空间内容不缩进（LLVM 默认）
# Inner = 只有嵌套命名空间的内容缩进
NamespaceIndentation: All

# 不排序 #include 头文件
# Never = 保持头文件的原始顺序
# CaseSensitive = 区分大小写排序
# CaseInsensitive = 不区分大小写排序
SortIncludes: Never

# 访问修饰符（public/protected/private）的缩进偏移
# -4 表示访问修饰符向左退回 4 格，与类定义的 { 对齐
AccessModifierOffset: -4
```

`NamespaceIndentation: All` 和 `AccessModifierOffset: -4` 的组合效果：

```cpp
namespace krkr2 {
    namespace core {
        class Player {
        public:              // ← AccessModifierOffset: -4，退回与 class 的 { 对齐
            void play();
        private:
            int volume;
        };
    }  // namespace core
}  // namespace krkr2
```

## DisableFormat 目录

KrKr2 中有两个目录设置了格式禁用：

```yaml
# cpp/external/.clang-format
DisableFormat: true

# cpp/core/plugin/.clang-format
DisableFormat: true
```

**为什么禁用？**

- **`cpp/external/`**：包含第三方库代码（libbpg、minizip）。这些代码不是 KrKr2 团队维护的，修改它们的格式会导致与上游代码的差异变大，不利于将来合并上游更新
- **`cpp/core/plugin/`**：包含 ncbind 绑定框架代码。这部分代码大量使用宏和模板元编程，clang-format 对这类代码的格式化效果不佳，强制格式化可能降低可读性

### clang-format 的配置文件查找规则

clang-format 从被格式化文件所在目录开始，**向上逐级查找** `.clang-format` 文件，使用最近的那个。这就是为什么子目录可以覆盖根目录的配置：

```
krkr2/
├── .clang-format              ← 根配置（55 行规则）
├── cpp/
│   ├── core/
│   │   ├── base/
│   │   │   └── XP3Archive.cpp    ← 使用根目录的 .clang-format
│   │   └── plugin/
│   │       ├── .clang-format      ← DisableFormat: true
│   │       └── ncbind.cpp         ← 跳过格式化
│   └── external/
│       ├── .clang-format          ← DisableFormat: true
│       └── libbpg/                ← 跳过格式化
```

## CI 格式门禁解析

KrKr2 的 CI 使用 `.github/workflows/code-format-check.yml`（40 行）在每次 push 和 PR 时检查代码格式。

### 工作流文件逐行解读

```yaml
# .github/workflows/code-format-check.yml

name: Code Format Check

# 触发条件：push 到 main 分支，或 PR 到 main 分支
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  format-check:
    runs-on: ubuntu-latest

    steps:
      # 检出代码
      - uses: actions/checkout@v4

      # 安装 clang-format 20
      - name: Install clang-format
        run: |
          wget https://apt.llvm.org/llvm.sh
          chmod +x llvm.sh
          sudo ./llvm.sh 20
          sudo apt-get install -y clang-format-20

      # 执行格式检查
      - name: Check format
        run: |
          # 查找所有需要检查的文件
          FILES=$(find ./cpp ./platforms -regex ".+\.\(cpp\|cc\|h\|hpp\|inc\)")
          # 用 clang-format-20 检查每个文件
          # --dry-run 不修改文件，--Werror 将格式差异视为错误
          clang-format-20 --dry-run --Werror $FILES
```

### 关键设计决策分析

**1. 只检查 `cpp/` 和 `platforms/` 目录：**

```bash
find ./cpp ./platforms -regex ".+\.\(cpp\|cc\|h\|hpp\|inc\)"
```

注意这里**没有**包含 `tests/` 和 `tools/` 目录。这意味着测试代码和工具代码不受格式门禁约束。这是一个有意的设计选择——测试代码通常包含特殊的格式（如长字符串字面量、表格对齐的测试数据），强制格式化可能降低可读性。

**2. 使用 `--dry-run --Werror` 而非直接修改：**

CI 不应该修改代码，只应该检查。`--dry-run` 让 clang-format 只输出差异而不修改文件，`--Werror` 让格式差异导致非零退出码，从而让 CI 步骤失败。

**3. 门禁机制：所有构建依赖格式检查通过**

```yaml
# .github/workflows/build-linux.yml 第 3-7 行
on:
  workflow_run:
    workflows: ["Code Format Check"]
    types: [completed]
```

三个构建工作流（Linux、Windows、Android）都使用 `workflow_run` 触发，并在内部检查格式检查的结论（conclusion）：

```yaml
# 只有格式检查成功时才继续构建
if: ${{ github.event.workflow_run.conclusion == 'success' }}
```

这形成了一个**门禁链**（Gate Chain）：

```
Code Format Check ──成功──→ Build Linux
                  ──成功──→ Build Windows
                  ──成功──→ Build Android
                  ──失败──→ 所有构建被跳过
```

## 编辑器集成

### VS Code

```json
// .vscode/settings.json
{
    // 使用 clang-format 作为 C++ 格式化器
    "C_Cpp.clang_format_style": "file",
    // "file" 表示使用项目根目录的 .clang-format 文件

    // 保存时自动格式化
    "editor.formatOnSave": true,

    // 指定 clang-format 可执行文件路径（如果不在 PATH 中）
    // Windows:
    // "C_Cpp.clang_format_path": "C:\\Program Files\\LLVM\\bin\\clang-format.exe"
    // macOS:
    // "C_Cpp.clang_format_path": "/opt/homebrew/opt/llvm/bin/clang-format"

    // 可选：设置默认格式化器
    "[cpp]": {
        "editor.defaultFormatter": "ms-vscode.cpptools"
    }
}
```

安装 Microsoft C/C++ 扩展后，VS Code 会自动识别 `.clang-format` 文件。按 `Shift+Alt+F`（Windows/Linux）或 `Shift+Option+F`（macOS）可以手动格式化当前文件。

### CLion

CLion 内置 clang-format 支持：

```
设置路径：
File → Settings → Editor → Code Style → C/C++

1. 勾选 "Enable ClangFormat (only for C/C++/Objective-C)"
2. CLion 会自动检测项目根目录的 .clang-format 文件
3. 快捷键 Ctrl+Alt+L（Windows/Linux）或 Cmd+Alt+L（macOS）格式化当前文件

验证是否生效：
- 在格式化后查看底部状态栏，应显示 "Formatted with ClangFormat"
```

### Visual Studio

```
Visual Studio 2019+：

1. 将 .clang-format 文件放在项目根目录（已经有了）
2. Tools → Options → Text Editor → C/C++ → Formatting
3. 勾选 "Enable ClangFormat support"
4. 选择 "Use custom clang-format.exe" 并指定路径（可选）
5. 快捷键 Ctrl+K, Ctrl+D 格式化当前文档

Visual Studio 会自动读取 .clang-format 文件，
如果未找到则使用内置的默认样式。
```

## 动手实践

### 实践 1：查看格式化前后差异

```bash
# 创建一个故意不符合格式的测试文件
cat > /tmp/test_format.cpp << 'EOF'
#include <iostream>
#include <string>

namespace test{
class Foo{
public:
void bar( int x,const std::string&y ){
if(x>0){
std::cout<<"hello "<<y<<std::endl;
}else{
std::cout<<"bye"<<std::endl;
}
}
};
}
EOF

# 用 clang-format 格式化（使用 KrKr2 的配置）
# 先进入项目根目录（clang-format 会查找 .clang-format 文件）
cd /path/to/krkr2
clang-format /tmp/test_format.cpp
```

格式化后的输出：

```cpp
#include <iostream>
#include <string>

namespace test {
    class Foo {
    public:
        void bar(int x, const std::string& y) {
            if(x > 0) {
                std::cout << "hello " << y << std::endl;
            } else {
                std::cout << "bye" << std::endl;
            }
        }
    };
}  // namespace test
```

观察变化：
- 命名空间 `{` 前加了空格
- 类内容缩进了 4 格（`NamespaceIndentation: All`）
- 函数参数括号内无空格（`SpacesInParentheses: false`）
- `const` 前加了空格，`&` 后加了空格
- 运算符两边加了空格
- `else` 跟在 `}` 后面（`BeforeElse: false`）

### 实践 2：批量检查项目格式

```bash
# 在项目根目录执行
cd /path/to/krkr2

# 检查所有源文件是否符合格式规范（不修改文件）
# Linux/macOS:
find ./cpp ./platforms -regex ".+\.\(cpp\|cc\|h\|hpp\|inc\)" \
    -exec clang-format --dry-run --Werror {} + 2>&1 | head -20

# 如果所有文件都符合规范，没有任何输出
# 如果有不符合的文件，输出类似：
# /path/to/file.cpp:42:10: warning: code should be clang-formatted [-Wclang-format-violations]

# Windows (PowerShell):
Get-ChildItem -Path cpp,platforms -Recurse -Include *.cpp,*.h,*.hpp,*.cc,*.inc |
    ForEach-Object {
        $result = clang-format --dry-run --Werror $_.FullName 2>&1
        if($LASTEXITCODE -ne 0) {
            Write-Host "NOT FORMATTED: $($_.FullName)"
            $result
        }
    }
```

### 实践 3：创建预提交钩子

```bash
# 创建 Git pre-commit hook，在提交前自动检查格式
cat > .git/hooks/pre-commit << 'HOOK'
#!/bin/bash
# Pre-commit hook: 检查 C++ 代码格式

# 获取暂存区中的 C++ 文件
FILES=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(cpp|cc|h|hpp|inc)$')

if [ -z "$FILES" ]; then
    exit 0  # 没有 C++ 文件，跳过检查
fi

# 检查格式
ERRORS=0
for FILE in $FILES; do
    # 只检查暂存的内容（不是工作区的内容）
    git show ":$FILE" | clang-format --assume-filename="$FILE" --dry-run --Werror 2>/dev/null
    if [ $? -ne 0 ]; then
        echo "格式错误: $FILE"
        ERRORS=$((ERRORS + 1))
    fi
done

if [ $ERRORS -ne 0 ]; then
    echo ""
    echo "发现 $ERRORS 个文件格式不符合规范。"
    echo "请运行以下命令格式化代码后重新提交："
    echo "  clang-format -i <文件路径>"
    echo ""
    echo "或一键格式化所有文件："
    echo '  find ./cpp ./platforms -regex ".+\.\(cpp\|cc\|h\|hpp\|inc\)" -exec clang-format -i {} +'
    exit 1
fi

exit 0
HOOK

chmod +x .git/hooks/pre-commit
echo "Pre-commit hook 安装成功！"
```

### 实践 4：使用 clang-format 注释禁用区域格式化

有时某段代码不适合自动格式化（如手动对齐的查找表）：

```cpp
// 使用 clang-format off/on 注释包裹不需要格式化的区域

// clang-format off
// 这个查找表经过精心对齐，不应该被格式化打乱
static const int lookup_table[] = {
    0,   1,   2,   3,   4,   5,   6,   7,
    8,   9,  10,  11,  12,  13,  14,  15,
   16,  17,  18,  19,  20,  21,  22,  23,
   24,  25,  26,  27,  28,  29,  30,  31,
};
// clang-format on

// 此后的代码继续被格式化
void processData(int index) {
    if(index >= 0 && index < 32) {
        return lookup_table[index];
    }
}
```

**注意：** `// clang-format off` 和 `// clang-format on` 必须成对出现。如果忘记写 `on`，该文件剩余部分都不会被格式化。

## 对照项目源码

本节涉及的项目配置文件：

```
项目文件与 clang-format 的关联：

┌──────────────────────────────────────┬────────┬─────────────────────────────┐
│ 文件路径                              │ 行数   │ 作用                         │
├──────────────────────────────────────┼────────┼─────────────────────────────┤
│ .clang-format                         │ 55 行  │ 项目根目录的格式化规则        │
│ cpp/external/.clang-format            │ 1 行   │ DisableFormat: true          │
│ cpp/core/plugin/.clang-format         │ 1 行   │ DisableFormat: true          │
│ .github/workflows/code-format-check.yml│ 40 行 │ CI 格式检查门禁工作流        │
│ .github/workflows/build-linux.yml     │ 100 行 │ workflow_run 依赖格式检查    │
│ .github/workflows/build-windows.yml   │ 84 行  │ workflow_run 依赖格式检查    │
│ .github/workflows/build-android.yml   │ 169 行 │ workflow_run 依赖格式检查    │
└──────────────────────────────────────┴────────┴─────────────────────────────┘
```

## 常见错误与排查

### 错误 1：CI 格式检查失败但本地看起来正常

```
错误信息（CI 日志）：
cpp/core/visual/LoadPNG.cpp:42:10: warning: code should be
clang-formatted [-Wclang-format-violations]

原因：本地 clang-format 版本与 CI 不一致。
CI 使用 clang-format 20，本地可能是 14 或 17。
不同版本的格式化结果可能有细微差异。
```

```bash
# 解决方案：安装与 CI 一致的 clang-format 版本
# Ubuntu:
wget https://apt.llvm.org/llvm.sh
chmod +x llvm.sh
sudo ./llvm.sh 20
sudo apt-get install -y clang-format-20

# 使用指定版本格式化
clang-format-20 -i cpp/core/visual/LoadPNG.cpp
```

### 错误 2：格式化后出现意外的命名空间缩进

```
现象：格式化后命名空间内的所有代码都缩进了 4 格，
看起来很怪。

原因：NamespaceIndentation: All 会缩进命名空间内的所有内容。
这是 KrKr2 的有意选择，不是 bug。
```

```cpp
// KrKr2 的风格（NamespaceIndentation: All）
namespace krkr2 {
    void foo() {       // ← 缩进 4 格
        // ...
    }
}

// 大多数其他项目（NamespaceIndentation: None）
namespace krkr2 {
void foo() {           // ← 不缩进
    // ...
}
}
```

如果你不习惯这种风格，**不要修改 `.clang-format`**——保持与项目一致是最重要的。

### 错误 3：头文件顺序被打乱

```
现象：格式化后 #include 的顺序变了，导致编译错误。

原因：某些项目的代码依赖特定的头文件包含顺序
（如先 include 预编译头文件）。
```

KrKr2 通过 `SortIncludes: Never` 避免了这个问题——clang-format 不会重新排序 `#include` 指令。这是一个安全的选择，因为 KrKr2 的很多源文件依赖特定的包含顺序（如 `tjsCommHead.h` 必须先于其他 TJS2 头文件）。

---

## 本节小结

- **clang-format** 是 LLVM 项目提供的 C/C++ 代码格式化工具，能自动将源代码调整为统一风格，消除团队协作中的格式争论
- KrKr2 使用 **LLVM 基础样式**并进行了大量自定义：`IndentWidth: 4`（缩进 4 格）、`ColumnLimit: 80`（每行最多 80 字符）、`SpaceBeforeParens: Never`（函数名与括号之间无空格）
- **K&R 大括号风格**：通过将所有 `BraceWrapping` 选项设为 `false`，开括号 `{` 跟在语句末尾而不另起一行
- **NamespaceIndentation: All** 是 KrKr2 的特殊选择——命名空间内部所有内容都缩进，与多数开源项目不同
- **SortIncludes: Never** 防止 clang-format 重排 `#include` 顺序，避免因头文件依赖顺序导致编译失败
- **DisableFormat 机制**：`cpp/external/` 和 `cpp/core/plugin/` 通过子目录 `.clang-format` 文件禁用格式化，分别保护第三方代码和模板元编程代码
- **CI 门禁**（CI Gate，持续集成流水线中的自动检查点）通过 `code-format-check.yml` 工作流在每次 push / PR 时检查格式，格式不通过则阻止所有后续构建
- **编辑器集成**是最佳实践——在 VS Code、CLion、Visual Studio 中配置"保存时格式化"，可以在编码过程中自动保持格式一致，避免提交时才发现格式问题
- **预提交钩子**（Pre-commit Hook，Git 在执行 `commit` 操作前自动运行的脚本）可在本地拦截格式问题，比等 CI 反馈更快

---

## 练习题与答案

### 题目 1：分析 KrKr2 格式化配置的设计意图

阅读以下 `.clang-format` 配置片段，回答问题：

```yaml
BasedOnStyle: LLVM
IndentWidth: 4
ColumnLimit: 80
SpaceBeforeParens: Never
NamespaceIndentation: All
SortIncludes: Never
BraceWrapping:
  AfterFunction: false
  AfterClass: false
  AfterControlStatement: false
AccessModifierOffset: -4
```

问题：
1. `AccessModifierOffset: -4` 与 `IndentWidth: 4` 搭配使用会产生什么效果？请用代码示例说明。
2. 为什么 `SortIncludes` 设为 `Never` 而不是更常见的 `CaseInsensitive`？
3. 如果将 `NamespaceIndentation` 从 `All` 改为 `None`，对项目中已有的命名空间代码会产生什么影响？

<details>
<summary>查看答案</summary>

**1. `AccessModifierOffset: -4` 的效果：**

`AccessModifierOffset` 控制 `public:`、`protected:`、`private:` 等访问修饰符（Access Modifier，C++ 类中控制成员可见性的关键字）相对于类体缩进的偏移量。`IndentWidth: 4` 意味着类体内容缩进 4 格，而 `AccessModifierOffset: -4` 将访问修饰符"退回" 4 格，使其与 `class` 关键字对齐：

```cpp
class Player {
public:                    // ← 偏移 -4，与 class 关键字对齐（0 格缩进）
    int health;            // ← 正常缩进 4 格
    void takeDamage(int d); // ← 正常缩进 4 格

private:                   // ← 偏移 -4，同样与 class 对齐
    std::string name;      // ← 正常缩进 4 格
};
```

如果 `AccessModifierOffset` 为 `0`（默认），访问修饰符会和成员变量一样缩进 4 格：

```cpp
class Player {
    public:                // ← 缩进 4 格，视觉上不突出
        int health;        // ← 缩进 8 格（4 + 4）
};
```

KrKr2 选择 `-4` 是为了让访问修饰符在视觉上更突出，方便快速定位类的 public/private 分区。

**2. `SortIncludes: Never` 的原因：**

KrKr2 的源代码中很多文件依赖特定的头文件包含顺序。例如 TJS2 引擎的源文件需要先包含 `tjsCommHead.h`（通用头文件，定义了全局宏和类型别名），如果 clang-format 按字母顺序重排 `#include`，可能把其他头文件排到 `tjsCommHead.h` 前面，导致编译失败。`CaseInsensitive` 或 `CaseSensitive` 都会触发排序，只有 `Never` 才能完全避免重排。

**3. 将 `NamespaceIndentation` 改为 `None` 的影响：**

项目中所有命名空间内的代码都会"退缩进"——从缩进 4 格变为 0 格。这意味着一次 `clang-format` 运行会修改大量文件，产生巨大的 diff（代码差异），几乎每个 `.cpp` 和 `.h` 文件都会被改动。这会：
- 让 `git blame`（查看每行代码最后修改者的 Git 命令）变得无用，因为所有行都显示为"格式化"提交修改的
- 产生大量合并冲突（如果有其他未合并的分支）
- 破坏代码审查历史

因此这类全局格式变更需要团队共识，且应在专门的"格式化"提交中一次性完成。

</details>

### 题目 2：编写预提交钩子脚本

编写一个 Git pre-commit 钩子脚本，满足以下要求：
1. 只检查本次 commit 中被修改的 `.cpp`、`.h`、`.hpp` 文件
2. 使用 `clang-format --dry-run -Werror` 检查格式
3. 如果有格式不合规的文件，打印文件列表并阻止提交
4. 脚本需要兼容 Linux 和 macOS（使用 bash）

<details>
<summary>查看答案</summary>

```bash
#!/usr/bin/env bash
# 文件路径：.git/hooks/pre-commit
# 功能：在 git commit 前检查 C++ 文件格式是否符合 clang-format 规则
# 使用方法：将此文件保存到 .git/hooks/pre-commit 并添加执行权限
#           chmod +x .git/hooks/pre-commit

set -euo pipefail  # 严格模式：任何命令失败立即退出

# ===== 配置 =====
# clang-format 可执行文件路径（可通过环境变量覆盖）
CLANG_FORMAT="${CLANG_FORMAT:-clang-format}"

# 需要检查的文件扩展名
FILE_EXTENSIONS="\.cpp$|\.h$|\.hpp$|\.cc$|\.inc$"

# ===== 检查 clang-format 是否安装 =====
if ! command -v "$CLANG_FORMAT" &> /dev/null; then
    echo "错误：未找到 clang-format，请先安装"
    echo "  Ubuntu/Debian: sudo apt install clang-format"
    echo "  macOS:         brew install clang-format"
    exit 1
fi

# ===== 获取本次提交中修改的 C++ 文件 =====
# git diff --cached：只看暂存区（staged）的文件
# --name-only：只输出文件名
# --diff-filter=ACMR：只看 Added/Copied/Modified/Renamed 的文件（排除 Deleted）
CHANGED_FILES=$(git diff --cached --name-only --diff-filter=ACMR \
    | grep -E "$FILE_EXTENSIONS" || true)

# 如果没有修改的 C++ 文件，直接通过
if [ -z "$CHANGED_FILES" ]; then
    echo "pre-commit: 没有 C++ 文件被修改，跳过格式检查"
    exit 0
fi

# ===== 逐文件检查格式 =====
FAILED_FILES=()  # 存储格式不合规的文件名

for file in $CHANGED_FILES; do
    # 跳过不存在的文件（可能被删除但 diff 还在）
    if [ ! -f "$file" ]; then
        continue
    fi

    # --dry-run：不实际修改文件，只检查
    # -Werror：将格式警告视为错误（返回非零退出码）
    if ! "$CLANG_FORMAT" --dry-run -Werror "$file" 2>/dev/null; then
        FAILED_FILES+=("$file")
    fi
done

# ===== 输出结果 =====
if [ ${#FAILED_FILES[@]} -gt 0 ]; then
    echo "=========================================="
    echo " clang-format 检查未通过"
    echo "=========================================="
    echo ""
    echo "以下文件格式不符合 .clang-format 规则："
    for f in "${FAILED_FILES[@]}"; do
        echo "  ❌ $f"
    done
    echo ""
    echo "请运行以下命令修复："
    echo "  $CLANG_FORMAT -i ${FAILED_FILES[*]}"
    echo ""
    echo "或一键修复所有暂存文件："
    echo "  git diff --cached --name-only --diff-filter=ACMR \\"
    echo "    | grep -E '$FILE_EXTENSIONS' \\"
    echo "    | xargs $CLANG_FORMAT -i"
    echo ""
    echo "修复后重新 git add 并提交。"
    exit 1  # 非零退出码阻止提交
fi

echo "pre-commit: ✅ 所有 C++ 文件格式检查通过"
exit 0
```

**安装步骤：**

```bash
# 将脚本保存到 .git/hooks/ 目录
cp pre-commit-format.sh .git/hooks/pre-commit

# 添加执行权限（Linux/macOS 必须）
chmod +x .git/hooks/pre-commit

# 测试：故意写一段格式不对的代码然后提交
echo "int main(){return 0;}" > test_format.cpp
git add test_format.cpp
git commit -m "test"  # 应该被钩子拦截
```

</details>

### 题目 3：排查 CI 格式检查失败

你在 KrKr2 项目中提交了一个 PR，CI 的 `Code Format Check` 工作流报错：

```
Error: code formatting check failed.
Please run clang-format 20 locally and commit the changes.
Files checked: ./cpp/ ./platforms/
```

但你在本地运行 `clang-format -i` 后再 `git diff` 发现没有任何变化。请分析可能的原因，并给出排查步骤。

<details>
<summary>查看答案</summary>

**最可能的原因及排查步骤：**

**原因 1：本地 clang-format 版本与 CI 不一致（最常见）**

CI 使用 clang-format **20**（通过 `apt install clang-format-20` 安装），而本地可能是旧版本。不同版本对同一配置的格式化结果可能不同。

```bash
# 排查步骤：检查本地版本
clang-format --version
# 如果输出不是 20.x.x，就是版本问题

# 修复方法（Ubuntu）：
sudo apt install clang-format-20
# 使用指定版本运行
clang-format-20 -i your_file.cpp
```

```bash
# 修复方法（macOS）：
brew install llvm@20
# LLVM 安装在 /opt/homebrew/opt/llvm@20/bin/
/opt/homebrew/opt/llvm@20/bin/clang-format -i your_file.cpp
```

**原因 2：文件不在 CI 检查范围内**

CI 的 `code-format-check.yml` 只检查 `./cpp/` 和 `./platforms/` 目录。如果你修改的文件在 `tests/` 或 `tools/` 中，CI 不会检查这些文件——但 CI 的错误信息可能让你误以为是这些目录的文件有问题。

```bash
# 排查步骤：确认你修改了哪些目录的文件
git diff --name-only origin/main...HEAD

# 逐目录检查
clang-format-20 --dry-run -Werror cpp/**/*.cpp cpp/**/*.h
clang-format-20 --dry-run -Werror platforms/**/*.cpp platforms/**/*.h
```

**原因 3：行尾符差异（Windows 开发者常见）**

Windows 使用 `CRLF`（`\r\n`）作为行尾符（Line Ending，文本文件中表示换行的控制字符），而 CI（Linux 环境）使用 `LF`（`\n`）。如果 Git 的 `core.autocrlf` 配置不当，可能导致 CI 看到的文件内容和本地不同。

```bash
# 排查步骤：检查 Git 行尾符配置
git config core.autocrlf

# 查看文件实际的行尾符
file your_file.cpp
# 如果输出包含 "CRLF"，可能是问题所在

# 修复：确保 .gitattributes 正确配置
echo "*.cpp text eol=lf" >> .gitattributes
echo "*.h text eol=lf" >> .gitattributes

# 重新规范化所有文件的行尾符
git add --renormalize .
```

**原因 4：`.clang-format` 文件没有被提交**

如果你在本地修改了 `.clang-format` 但没有提交，CI 使用的是仓库中旧版本的配置。

```bash
# 排查步骤：检查 .clang-format 是否有未提交的修改
git status .clang-format
git diff .clang-format
```

**系统化排查流程：**

```
1. clang-format --version          → 确认版本 = 20
2. git diff --name-only            → 确认修改文件在 cpp/ 或 platforms/ 中
3. file <文件名>                   → 确认行尾符是 LF 而非 CRLF
4. git status .clang-format        → 确认配置文件已提交
5. clang-format-20 --dry-run -Werror <文件> → 本地模拟 CI 检查
```

</details>

---

## 下一步

下一节我们将学习 **clang-tidy 静态分析工具**——与 clang-format 关注"代码长什么样"不同，clang-tidy 关注"代码写得对不对"。我们将深入解读 KrKr2 的 `.clang-tidy` 配置文件中的 8 大类检查规则，学习如何在本地运行静态分析并修复常见警告。

👉 [02-clang-tidy 静态分析](./02-clang-tidy静态分析.md)




