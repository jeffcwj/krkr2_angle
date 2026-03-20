# TEST_FILES_PATH 与测试夹具

> **所属模块：** M10-测试与质量保证
> **前置知识：** [测试目录结构与 CMake 配置](./01-测试目录结构与CMake配置.md)、[自定义 main 与 spdlog 集成](./02-自定义main与spdlog集成.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：

1. 完整理解 `configure_file` 从模板到头文件的转换过程
2. 掌握 `TEST_FILES_PATH` 宏在测试代码中的各种拼接写法
3. 知道如何为新模块组织和添加测试夹具文件
4. 读懂项目中 `psbfile-dll.cpp` 和 `tjs.cpp` 是如何使用夹具的
5. 编写自己的夹具加载代码并处理常见错误

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 测试夹具 | Test Fixture | 测试运行前准备好的外部数据文件（脚本、二进制数据等），用于提供可重复的测试输入 |
| configure_file | configure_file | CMake 命令，将模板文件中的 `@变量名@` 占位符替换为实际值后输出到构建目录 |
| 字符串字面量拼接 | String Literal Concatenation | C/C++ 语言特性——相邻的两个字符串字面量自动合并为一个 |
| PSB 文件 | PSB File | E-mote 动画系统使用的二进制包格式，KrKr2 通过 psbfile 插件解析它 |
| TJS 脚本 | TJS Script | KiriKiri 引擎的脚本语言文件（扩展名 .tjs），用于实现游戏逻辑 |

---

## configure_file 工作原理

在上一章我们简单提到了 `configure_file`，现在深入讲解它的完整工作流程。

### 变量替换的两种语法

`configure_file` 支持两种占位符语法：

| 语法 | 示例 | 行为 |
|------|------|------|
| `@变量名@` | `@TEST_FILES_PATH@` | 替换为 CMake 变量的值；变量未定义则替换为空字符串 |
| `${变量名}` | `${TEST_FILES_PATH}` | 与 `@` 语法相同，但容易与 shell/Makefile 变量冲突 |

KrKr2 的 `test_config.h.in` 使用 `@` 语法，这是最佳实践——它只在 CMake 的 `configure_file` 中有效，不会与 C/C++ 预处理器或 shell 脚本中的 `${}` 混淆。

### 完整转换过程

让我们追踪从源文件到最终 C++ 代码的完整路径：

```
源码树                              构建目录
─────────                          ─────────
tests/test_config.h.in    ──CMake──►  out/windows/debug/tests/test_config.h
       │                                         │
       │  #cmakedefine TEST_FILES_PATH            │  #define TEST_FILES_PATH
       │  "@TEST_FILES_PATH@"                     │  "C:/Users/dev/krkr2/tests/test_files"
       │                                         │
       ▼                                         ▼
  模板（人写的）                            生成文件（CMake 写的）
```

### `#cmakedefine` 指令详解

你可能注意到模板中用的是 `#cmakedefine` 而不是普通的 `#define`。这两者有重要区别：

| 指令 | 行为 | 适用场景 |
|------|------|----------|
| `#define VAR "value"` | 无条件定义——不管 CMake 变量是否存在，都会写入 `#define` | 值一定存在的情况 |
| `#cmakedefine VAR "value"` | 条件定义——CMake 变量存在且不为假值时生成 `#define`，否则生成 `/* #undef VAR */` | 值可能不存在的可选配置 |

`#cmakedefine` 的条件判断规则：当 CMake 变量为以下值时，视为"假"，生成 `/* #undef */`：

- 空字符串 `""`
- `0`
- `OFF`、`NO`、`FALSE`
- 以 `-NOTFOUND` 结尾的字符串

来看一个完整示例，展示两种指令的对比：

```cmake
# CMakeLists.txt
set(TEST_FILES_PATH "${CMAKE_CURRENT_SOURCE_DIR}/test_files")  # 有值
set(OPTIONAL_FEATURE OFF)                                       # 假值
# MISSING_VAR 未定义

configure_file(config.h.in config.h)
```

```c
/* config.h.in — 模板 */
#cmakedefine TEST_FILES_PATH "@TEST_FILES_PATH@"
#cmakedefine OPTIONAL_FEATURE "@OPTIONAL_FEATURE@"
#cmakedefine MISSING_VAR "@MISSING_VAR@"
#define ALWAYS_HERE "hello"
```

```c
/* config.h — 生成结果 */
#define TEST_FILES_PATH "/home/dev/krkr2/tests/test_files"  /* 有值，正常定义 */
/* #undef OPTIONAL_FEATURE */                                /* OFF 视为假 */
/* #undef MISSING_VAR */                                     /* 未定义视为假 */
#define ALWAYS_HERE "hello"                                  /* 无条件定义不受影响 */
```

> **为什么 KrKr2 用 `#cmakedefine` 而不是 `#define`？**
> 安全性。如果开发者意外删除了 `set(TEST_FILES_PATH ...)` 那一行，`#cmakedefine` 会生成 `/* #undef TEST_FILES_PATH */`，编译器随后会在使用 `TEST_FILES_PATH` 的地方报"未定义标识符"错误——立刻暴露问题。如果用 `#define`，它会生成 `#define TEST_FILES_PATH ""`（空字符串），程序不会报编译错误，但测试运行时会尝试打开空路径的文件，产生难以排查的运行时错误。

---

## C/C++ 字符串字面量拼接

这是理解 `TEST_FILES_PATH` 用法的关键——C/C++ 的一个基础语言特性。

### 什么是字符串字面量拼接

字符串字面量拼接（String Literal Concatenation）是 C 语言从 C89 标准就支持的特性：**编译器会自动将相邻的字符串字面量合并为一个**。这里的"相邻"指两个字符串字面量之间只有空白字符（空格、换行、制表符），没有运算符或其他标记。

```cpp
// 示例 1：最基本的拼接
#include <iostream>

int main() {
    // 以下三行效果完全相同
    const char* s1 = "Hello" " " "World";    // 相邻，自动合并
    const char* s2 = "Hello World";           // 手写完整字符串
    const char* s3 = "Hello"
                     " "
                     "World";                 // 跨行也可以，只要没有运算符

    std::cout << s1 << std::endl;  // 输出: Hello World
    std::cout << s2 << std::endl;  // 输出: Hello World
    std::cout << s3 << std::endl;  // 输出: Hello World
    return 0;
}
```

> **编译器在哪个阶段做拼接？**
> 在"翻译阶段 6"（Translation Phase 6）——这是预处理之后、语法分析之前的阶段。也就是说，**宏展开先于字符串拼接**。这一点至关重要：`TEST_FILES_PATH "/emote/ezsave.pimg"` 中的 `TEST_FILES_PATH` 先被展开为 `"/home/dev/krkr2/tests/test_files"`，然后编译器看到两个相邻字符串字面量 `"/home/dev/krkr2/tests/test_files"` 和 `"/emote/ezsave.pimg"`，自动合并。

### 拼接的处理流程

用 KrKr2 的实际代码追踪完整过程：

```
源代码:
    f.loadPSBFile(TEST_FILES_PATH "/emote/ezsave.pimg");

                    ↓ 阶段 1-4: 预处理器展开宏

预处理后:
    f.loadPSBFile("/home/dev/krkr2/tests/test_files" "/emote/ezsave.pimg");

                    ↓ 阶段 6: 字符串字面量拼接

拼接后:
    f.loadPSBFile("/home/dev/krkr2/tests/test_files/emote/ezsave.pimg");

                    ↓ 后续阶段: 正常编译
```

### 拼接 vs 运行时拼接的对比

为什么不用 `std::string` 的 `+` 运算符？来看两种方式的对比：

```cpp
// 示例 2：编译时拼接 vs 运行时拼接
#include <iostream>
#include <string>

#define BASE_PATH "/data/test_files"

int main() {
    // 方式 A：编译时拼接（零开销）
    // 编译器直接生成完整字符串常量，存入只读数据段
    const char* path_a = BASE_PATH "/scripts/test.tjs";

    // 方式 B：运行时拼接（有开销）
    // 程序运行时分配内存、复制字符、再释放
    std::string path_b = std::string(BASE_PATH) + "/scripts/test.tjs";

    std::cout << "编译时: " << path_a << std::endl;
    std::cout << "运行时: " << path_b << std::endl;
    return 0;
}
```

| 对比维度 | 编译时拼接 (字面量相邻) | 运行时拼接 (std::string +) |
|----------|----------------------|--------------------------|
| 执行时间 | 零——编译期完成 | 需要分配内存 + 复制字节 |
| 内存分配 | 无——字符串在只读段（`.rodata`） | 堆内存分配（可能触发 `malloc`） |
| 结果类型 | `const char*`（C 风格字符串指针） | `std::string`（C++ 字符串对象） |
| 可用于 | C 和 C++ | 仅 C++ |
| 限制 | 只能拼接字面量和宏展开的字面量 | 可以拼接变量、表达式等任意字符串 |

---

## 项目中的夹具使用模式

KrKr2 的测试代码中有两种使用 `TEST_FILES_PATH` 加载夹具文件的写法。理解这两种模式，你就能为任何新模块编写夹具加载代码。

### 模式一：直接拼接（psbfile-dll.cpp）

`tests/unit-tests/plugins/psbfile-dll.cpp` 采用最直接的方式——在函数调用处直接拼接路径：

```cpp
// 来源：tests/unit-tests/plugins/psbfile-dll.cpp 第 10-14 行
#include "test_config.h"  // 引入 TEST_FILES_PATH 宏定义

TEST_CASE("read psbfile ezsave.pimg") {
    PSB::PSBFile f;
    // 直接拼接：TEST_FILES_PATH 展开为字符串字面量，
    // 与 "/emote/ezsave.pimg" 自动合并
    REQUIRE(f.loadPSBFile(TEST_FILES_PATH "/emote/ezsave.pimg"));
    // ...
}
```

这里的 `TEST_FILES_PATH "/emote/ezsave.pimg"` 就是字符串字面量拼接。预处理器先把 `TEST_FILES_PATH` 展开为 `"/home/dev/krkr2/tests/test_files"`，然后编译器把两个相邻字符串合并为完整路径。

**同一文件中的第二个 TEST_CASE 还展示了包含非 ASCII 字符的路径：**

```cpp
// 来源：tests/unit-tests/plugins/psbfile-dll.cpp 第 208-214 行
TEST_CASE("read psbfile e-mote3.0 psb") {
    int key = 742877301;       // PSB 文件的解密密钥
    PSB::PSBFile f;
    f.setSeed(key);            // 设置解密种子
    // 路径中包含日文字符 "バニラパジャマa"
    REQUIRE(
        f.loadPSBFile(TEST_FILES_PATH "/emote/e-mote3.0バニラパジャマa.psb"));
    REQUIRE(f.getType() == PSB::PSBType::Motion);
}
```

> **要点：** 夹具文件名可以包含 Unicode 字符（如日文）。只要源文件以 UTF-8 编码保存、编译器支持 UTF-8 字符串字面量（C++17 的主流编译器都支持），路径拼接就能正常工作。

**psbfile-dll.cpp 的夹具数据结构分析：**

该测试文件加载一个 `.pimg` 文件（E-mote 动画系统的图像包格式），然后用大量预定义数据验证解析结果：

```cpp
// 来源：tests/unit-tests/plugins/psbfile-dll.cpp 第 30-93 行（精简展示）
SECTION("get layers") {
    auto layers =
        std::dynamic_pointer_cast<PSB::PSBList>((*objs)["layers"]);
    REQUIRE(layers->size() == 32);  // 预期 32 个图层

    SECTION("check layer properties") {
        // 预定义的正确值数组——每个图层的宽度
        std::vector widths = {
            27, 27, 36, 34, 41, 41, 40, 36, 36, 36, 36,
            36, 36, 36, 36, 36, 36, 36, 36, 36, 40, 27,
            27, 36, 72, 80, 1280, 0, 0, 0, 0, 0
        };

        // 循环验证每个图层的每个属性
        for(int i = 0; i < layers->size(); i++) {
            auto layer = std::dynamic_pointer_cast<PSB::PSBDictionary>(
                (*layers)[i]);
            auto width = std::dynamic_pointer_cast<PSB::PSBNumber>(
                (*layer)["width"]);
            REQUIRE(static_cast<int>(*width) == widths[i]);  // 逐一比对
        }
    }
}
```

这种模式的特点是：**夹具文件提供复杂的二进制测试数据，测试代码中硬编码预期值数组**。这样做的好处是：如果解析逻辑出现回归（Regression，指修改代码后原本正确的功能变得错误），32 个图层中任何一个属性不匹配都会被 `REQUIRE` 捕获。

### 模式二：宏包装（tjs.cpp）

`tests/unit-tests/core/tjs2/tjs.cpp` 采用了更高层的抽象——用预处理器宏将路径拼接和文件加载逻辑封装起来：

```cpp
// 来源：tests/unit-tests/core/tjs2/tjs.cpp 第 46-54 行
#define SCRIPT(file_name)                                      \
    do {                                                       \
        ttstr tvPInitTJSScriptText{};                          \
        auto *stream = TVPCreateTextStreamForRead(             \
            TEST_FILES_PATH "/tjs2/" file_name, "");           \
        stream->Read(tvPInitTJSScriptText, 0);                 \
        tvPScriptEngine->ExecScript(tvPInitTJSScriptText);     \
        delete stream;                                         \
    } while(false)
```

使用时只需一行：

```cpp
// 来源：tests/unit-tests/core/tjs2/tjs.cpp 第 56-66 行
SECTION("exec test_class.tjs")    { SCRIPT("test_class.tjs");    }
SECTION("exec test_function.tjs") { SCRIPT("test_function.tjs"); }
SECTION("exec test_misc.tjs")     { SCRIPT("test_misc.tjs");     }
SECTION("exec test_string.tjs")   { SCRIPT("test_string.tjs");   }
SECTION("exec test_variant.tjs")  { SCRIPT("test_variant.tjs");  }
SECTION("exec test_with.tjs")     { SCRIPT("test_with.tjs");     }
```

**宏展开过程追踪：**

```
源码:    SCRIPT("test_class.tjs")

展开 SCRIPT 宏:
    do {
        ttstr tvPInitTJSScriptText{};
        auto *stream = TVPCreateTextStreamForRead(
            TEST_FILES_PATH "/tjs2/" "test_class.tjs", "");
            ↑ 此时 file_name 被替换为 "test_class.tjs"

展开 TEST_FILES_PATH:
    do {
        ttstr tvPInitTJSScriptText{};
        auto *stream = TVPCreateTextStreamForRead(
            "/home/dev/krkr2/tests/test_files" "/tjs2/" "test_class.tjs", "");
            ↑ 三个相邻字符串字面量

字符串拼接后:
    do {
        ttstr tvPInitTJSScriptText{};
        auto *stream = TVPCreateTextStreamForRead(
            "/home/dev/krkr2/tests/test_files/tjs2/test_class.tjs", "");
```

> **为什么用 `do { ... } while(false)` 包裹宏？**
> 这是 C/C++ 宏编程的经典技巧。如果不加 `do-while(false)`，宏在 `if-else` 语句中使用时会出现悬挂 `else` 问题。例如 `if(cond) SCRIPT("test.tjs"); else ...` 中，如果宏展开为多条语句（没有 `do-while` 包裹），`else` 会与宏中的最后一个 `if`（如果有的话）匹配，而不是外层的 `if`。加上 `do { ... } while(false)` 后，整个宏展开为一条完整语句，可以安全地放在任何需要语句的地方。

### 两种模式的对比与选择

| 对比维度 | 直接拼接 | 宏包装 |
|----------|---------|--------|
| 代码量 | 每次使用都写完整路径 | 定义一次宏，使用时只需文件名 |
| 可读性 | 路径意图一目了然 | 需要查看宏定义才知道完整行为 |
| 适用场景 | 不同路径结构、少量夹具 | 同一目录下大量同类夹具 |
| 封装程度 | 仅路径拼接 | 可以包含加载、解析、清理等完整流程 |
| KrKr2 实例 | psbfile-dll.cpp（2 个不同 PSB 文件） | tjs.cpp（6 个同类 TJS 脚本） |

**选择建议：** 当你需要加载同一目录下的多个同类文件时（如 tjs.cpp 中的 6 个脚本），宏包装能显著减少重复代码。当夹具文件类型、加载方式各不相同时（如不同格式的二进制文件），直接拼接更清晰。

---

## 夹具文件组织规范

KrKr2 的 `test_files/` 目录结构简单但有讲究：

```
tests/test_files/
├── tjs2/                    # TJS 脚本夹具（按模块名分目录）
│   ├── test_class.tjs       # 类系统测试脚本
│   ├── test_function.tjs    # 函数测试脚本
│   ├── test_misc.tjs        # 杂项测试脚本
│   ├── test_string.tjs      # 字符串操作测试脚本
│   ├── test_variant.tjs     # Variant 类型测试脚本
│   └── test_with.tjs        # with 语句测试脚本
└── emote/                   # E-mote 动画夹具（按功能分目录）
    ├── ezsave.pimg           # 图像包文件（32 个图层）
    └── e-mote3.0バニラパジャマa.psb  # 动作数据文件（加密）
```

**组织原则：**

1. **按模块分目录**：每个测试模块的夹具放在 `test_files/<模块名>/` 下，与 `unit-tests/<模块名>/` 对应
2. **文件名即用途**：夹具文件名应该能说明测试目的。例如 `test_class.tjs` 一看就知道是测试 TJS 类系统的脚本
3. **保留原始文件名**：对于从真实游戏中提取的测试数据（如 PSB 文件），保留原始文件名（包括日文字符），避免因重命名导致无法追溯来源
4. **最小化夹具大小**：只包含测试所需的最小数据集。不要把整个游戏的资源文件丢进 `test_files/`——Git 仓库不适合存储大二进制文件

---

## 动手实践

### 练习：为 core/utils 模块添加夹具支持

假设你要为 `core/utils` 模块添加一个测试，需要从文件中读取配置数据。以下是完整步骤：

**第 1 步：创建夹具文件**

```
tests/test_files/utils/sample_config.txt
```

文件内容（一个简单的键值对配置文件）：

```text
# 测试用配置文件
app_name=KrKr2
version=2.0
debug_mode=true
max_fps=60
```

**第 2 步：确认 CMake 配置**

检查 `tests/CMakeLists.txt`（第 11-12 行），确认 `TEST_FILES_PATH` 已设置并 `configure_file` 已调用：

```cmake
# 来源：tests/CMakeLists.txt 第 11-12 行
set(TEST_FILES_PATH ${CMAKE_CURRENT_SOURCE_DIR}/test_files)
configure_file(test_config.h.in test_config.h)
```

这已经由项目配好，不需要修改。`TEST_FILES_PATH` 指向 `tests/test_files/`，你新建的 `utils/sample_config.txt` 自然在其子目录中。

**第 3 步：编写使用夹具的测试代码**

```cpp
// 示例 3：完整的夹具加载测试
// 文件：tests/unit-tests/core/utils/config_reader_test.cpp
#include <catch2/catch_test_macros.hpp>
#include <fstream>
#include <string>
#include <map>
#include "test_config.h"  // 引入 TEST_FILES_PATH

// 简易配置文件解析函数（被测代码）
std::map<std::string, std::string> parseConfig(const char* path) {
    std::map<std::string, std::string> result;
    std::ifstream file(path);          // 打开文件
    std::string line;
    while(std::getline(file, line)) {  // 逐行读取
        if(line.empty() || line[0] == '#') continue;  // 跳过空行和注释
        auto pos = line.find('=');     // 查找分隔符
        if(pos != std::string::npos) {
            result[line.substr(0, pos)] = line.substr(pos + 1);
        }
    }
    return result;
}

TEST_CASE("parse config file from fixture") {
    // 使用 TEST_FILES_PATH 拼接夹具路径
    auto config = parseConfig(TEST_FILES_PATH "/utils/sample_config.txt");

    SECTION("check key count") {
        REQUIRE(config.size() == 4);  // 4 个有效键值对
    }

    SECTION("check values") {
        REQUIRE(config["app_name"] == "KrKr2");
        REQUIRE(config["version"] == "2.0");
        REQUIRE(config["debug_mode"] == "true");
        REQUIRE(config["max_fps"] == "60");
    }
}
```

**第 4 步：配置 CMakeLists.txt**

```cmake
# 文件：tests/unit-tests/core/utils/CMakeLists.txt
set(SOURCES
    config_reader_test.cpp
)

# 遵循项目的 foreach 模式：一源一可执行文件
string(REPLACE ".cpp" "" BASENAMES_SOURCES "${SOURCES}")
foreach(name ${BASENAMES_SOURCES})
    add_executable(${name} ${name}.cpp main.cpp)
    target_link_libraries(${name}
        PRIVATE Catch2::Catch2
        PUBLIC krkr2core)
    target_include_directories(${name}
        PRIVATE "${TEST_CONFIG_DIR}")  # 找到生成的 test_config.h
    catch_discover_tests(${name})
endforeach()
```

**第 5 步：编写自定义 main.cpp**

```cpp
// 文件：tests/unit-tests/core/utils/main.cpp
#define CATCH_CONFIG_RUNNER
#include <catch2/catch_session.hpp>
#include <spdlog/spdlog.h>

int main(int argc, char* argv[]) {
    // 创建 spdlog 日志器——与项目其他测试模块保持一致
    spdlog::stdout_color_mt("core");
    spdlog::stdout_color_mt("tjs2");

    return Catch::Session().run(argc, argv);
}
```

**第 6 步：注册子目录**

在 `tests/CMakeLists.txt` 中添加：

```cmake
add_subdirectory(unit-tests/core/utils)  # 新增
```

---

## 对照项目源码

以下是本节涉及的项目源文件汇总，建议对照阅读：

| 文件路径 | 行号 | 说明 |
|----------|------|------|
| `tests/CMakeLists.txt` | 第 11-12 行 | `set(TEST_FILES_PATH ...)` 和 `configure_file(...)` |
| `tests/test_config.h.in` | 第 1-6 行 | 模板文件，`#cmakedefine TEST_FILES_PATH "@TEST_FILES_PATH@"` |
| `tests/unit-tests/plugins/psbfile-dll.cpp` | 第 10 行 | `#include "test_config.h"` 引入生成头文件 |
| `tests/unit-tests/plugins/psbfile-dll.cpp` | 第 14 行 | 直接拼接模式：`TEST_FILES_PATH "/emote/ezsave.pimg"` |
| `tests/unit-tests/plugins/psbfile-dll.cpp` | 第 213 行 | Unicode 路径：`TEST_FILES_PATH "/emote/e-mote3.0バニラパジャマa.psb"` |
| `tests/unit-tests/core/tjs2/tjs.cpp` | 第 46-54 行 | `SCRIPT` 宏定义——包装 `TEST_FILES_PATH "/tjs2/"` 拼接 |
| `tests/unit-tests/core/tjs2/tjs.cpp` | 第 56-66 行 | 使用 `SCRIPT` 宏加载 6 个 TJS 脚本 |

**阅读顺序建议：**

1. 先看 `test_config.h.in`（6 行）——理解模板
2. 再看 `CMakeLists.txt` 第 11-12 行——理解变量设置和转换触发
3. 然后看 `psbfile-dll.cpp` 第 14 行——理解最简单的使用方式
4. 最后看 `tjs.cpp` 第 46-66 行——理解宏封装的高级用法

---

## 跨平台路径差异

`TEST_FILES_PATH` 在不同平台上展开为不同格式的路径：

| 平台 | 典型展开结果 | 路径分隔符 |
|------|------------|-----------|
| Windows | `"C:/Users/dev/krkr2/tests/test_files"` | 正斜杠 `/`（CMake 统一） |
| Linux | `"/home/dev/krkr2/tests/test_files"` | 正斜杠 `/` |
| macOS | `"/Users/dev/krkr2/tests/test_files"` | 正斜杠 `/` |
| Android（交叉编译） | `"/path/to/ndk-build/tests/test_files"` | 正斜杠 `/` |

> **Windows 上为什么是正斜杠？**
> CMake 内部统一使用正斜杠 `/` 作为路径分隔符，即使在 Windows 上。`CMAKE_CURRENT_SOURCE_DIR` 在 Windows 上返回的是 `C:/Users/...` 而不是 `C:\Users\...`。好消息是 Windows 的文件 API（`fopen`、`ifstream` 等）同时接受正斜杠和反斜杠，所以拼接出的路径 `C:/Users/dev/krkr2/tests/test_files/emote/ezsave.pimg` 在 Windows 上也能正常打开文件。

---

## 常见错误与解决方案

### 错误 1：忘记包含 test_config.h

```
error: 'TEST_FILES_PATH' was not declared in this scope
```

**原因：** 测试源文件中没有 `#include "test_config.h"`。

**解决：** 在文件顶部添加包含指令：

```cpp
#include "test_config.h"  // 提供 TEST_FILES_PATH 宏
```

**同时确认** CMakeLists.txt 中有 `target_include_directories(${name} PRIVATE "${TEST_CONFIG_DIR}")`，否则编译器找不到这个头文件。

### 错误 2：路径拼接中多余的斜杠

```cpp
// 错误：多了一个斜杠
f.loadPSBFile(TEST_FILES_PATH "/emote/" "/ezsave.pimg");
//                                    ↑ 多余的 "/"
// 拼接结果："/.../test_files/emote//ezsave.pimg"（双斜杠）
```

**影响：** 在 Linux/macOS 上双斜杠通常无害（操作系统会忽略多余的 `/`），但在 Windows 上某些 API 可能无法处理 `//`。

**解决：** 确保每段路径的分隔符只出现一次：

```cpp
// 正确：每段之间恰好一个 /
f.loadPSBFile(TEST_FILES_PATH "/emote/ezsave.pimg");
```

### 错误 3：尝试用变量做字面量拼接

```cpp
// 错误：变量不能参与字面量拼接
std::string subdir = "/emote";
f.loadPSBFile(TEST_FILES_PATH subdir "/ezsave.pimg");
// 编译错误：expected ')' before 'subdir'
```

**原因：** 字符串字面量拼接只对编译期的字面量有效，`subdir` 是运行时变量，编译器不认识"字符串字面量紧跟一个变量"这种语法。

**解决：** 要拼接变量，必须使用运行时拼接：

```cpp
// 方式 A：std::string 拼接
std::string subdir = "/emote";
std::string path = std::string(TEST_FILES_PATH) + subdir + "/ezsave.pimg";
f.loadPSBFile(path.c_str());

// 方式 B：如果子目录是固定的，直接用字面量
f.loadPSBFile(TEST_FILES_PATH "/emote/ezsave.pimg");  // 推荐
```

---

## 本节小结

- `configure_file` 是 CMake 的模板引擎——将 `.in` 文件中的 `@变量名@` 占位符替换为实际值后输出到构建目录
- `#cmakedefine` 比 `#define` 更安全：变量未定义时生成 `/* #undef */`，让编译器立刻报错
- C/C++ 字符串字面量拼接是**编译期**行为，零运行时开销，发生在预处理之后、语法分析之前
- KrKr2 通过 `TEST_FILES_PATH` 宏 + 字面量拼接，实现了与构建路径无关的夹具文件引用
- 项目中有两种夹具使用模式：**直接拼接**（适合少量不同类型文件）和**宏包装**（适合同目录大量同类文件）
- 夹具文件按模块分目录组织在 `tests/test_files/` 下，与 `unit-tests/` 的目录结构对应
- 跨平台上 CMake 统一使用正斜杠 `/`，Windows 文件 API 兼容正斜杠，所以路径拼接无需特殊处理
- 新增夹具需要：创建夹具文件 → 确认 CMake 配置 → 在测试代码中 `#include "test_config.h"` → 使用 `TEST_FILES_PATH "/子目录/文件名"` 引用

---

## 练习题与答案

### 题目 1：configure_file 输出预测

给定以下 CMake 代码和模板文件，预测生成的头文件内容：

```cmake
# CMakeLists.txt
set(PROJECT_NAME "KrKr2")
set(ENABLE_DEBUG OFF)
# FEATURE_X 未定义
configure_file(app_config.h.in app_config.h)
```

```c
/* app_config.h.in */
#cmakedefine PROJECT_NAME "@PROJECT_NAME@"
#cmakedefine ENABLE_DEBUG
#cmakedefine FEATURE_X "@FEATURE_X@"
#define BUILD_YEAR "@CMAKE_CURRENT_LIST_DIR@"
```

<details>
<summary>查看答案</summary>

```c
/* app_config.h — 生成结果 */
#define PROJECT_NAME "KrKr2"       /* PROJECT_NAME 有值且非假 → 正常定义 */
/* #undef ENABLE_DEBUG */          /* OFF 被 #cmakedefine 视为假 → undef */
/* #undef FEATURE_X */             /* 未定义 → undef */
#define BUILD_YEAR "/path/to/source"  /* #define 无条件生效，@...@ 被替换 */
```

**解析：**

1. `PROJECT_NAME` 的值是 `"KrKr2"`（非空、非假），所以 `#cmakedefine` 生成正常的 `#define`
2. `ENABLE_DEBUG` 的值是 `OFF`，这是 CMake 的假值之一，所以 `#cmakedefine` 生成 `/* #undef */`
3. `FEATURE_X` 从未 `set()`，视为未定义，`#cmakedefine` 生成 `/* #undef */`
4. `BUILD_YEAR` 使用的是 `#define`（不是 `#cmakedefine`），所以无条件定义。`@CMAKE_CURRENT_LIST_DIR@` 是 CMake 内置变量，会被替换为当前 CMakeLists.txt 所在目录的路径

</details>

### 题目 2：修复路径拼接错误

以下代码有编译错误，找出并修复：

```cpp
#include <catch2/catch_test_macros.hpp>
#include <fstream>
#include "test_config.h"

TEST_CASE("load test data") {
    std::string module = "visual";
    std::ifstream file(TEST_FILES_PATH "/" module "/test_image.png");
    REQUIRE(file.is_open());
}
```

<details>
<summary>查看答案</summary>

**错误：** `TEST_FILES_PATH "/" module "/test_image.png"` 中的 `module` 是 `std::string` 变量，不是字符串字面量。编译器无法将字面量与变量做字面量拼接，会报语法错误。

**修复方式 A**（运行时拼接）：

```cpp
#include <catch2/catch_test_macros.hpp>
#include <fstream>
#include <string>
#include "test_config.h"

TEST_CASE("load test data") {
    std::string module = "visual";
    // 使用 std::string 的 + 运算符拼接
    std::string path = std::string(TEST_FILES_PATH) + "/" + module + "/test_image.png";
    std::ifstream file(path);
    REQUIRE(file.is_open());
}
```

**修复方式 B**（如果模块名是编译期常量，用字面量直接拼接）：

```cpp
#include <catch2/catch_test_macros.hpp>
#include <fstream>
#include "test_config.h"

TEST_CASE("load test data") {
    // 直接用字面量拼接——编译期完成，零开销
    std::ifstream file(TEST_FILES_PATH "/visual/test_image.png");
    REQUIRE(file.is_open());
}
```

**推荐方式 B**——如果子目录名是固定的，没理由用运行时变量。

</details>

### 题目 3：设计夹具加载宏

参考 tjs.cpp 的 `SCRIPT` 宏，为一个假设的 JSON 配置文件测试设计一个类似的夹具加载宏。要求：

1. 宏名为 `LOAD_JSON`
2. 接受一个文件名参数
3. 从 `TEST_FILES_PATH "/json/"` 目录加载文件
4. 将文件内容读入 `std::string` 变量 `json_content`
5. 使用 `do { ... } while(false)` 包装

<details>
<summary>查看答案</summary>

```cpp
// 示例 5：JSON 夹具加载宏
#include <catch2/catch_test_macros.hpp>
#include <fstream>
#include <sstream>
#include <string>
#include "test_config.h"

// 宏定义：加载 JSON 夹具文件到 json_content 变量
#define LOAD_JSON(file_name)                                   \
    do {                                                       \
        std::ifstream ifs(                                     \
            TEST_FILES_PATH "/json/" file_name);               \
        REQUIRE(ifs.is_open());                                \
        std::ostringstream oss;                                \
        oss << ifs.rdbuf();    /* 一次性读取整个文件 */          \
        json_content = oss.str();                              \
    } while(false)

TEST_CASE("parse json config files") {
    std::string json_content;  // 宏内部会写入这个变量

    SECTION("load app config") {
        LOAD_JSON("app_config.json");
        // json_content 现在包含文件的完整内容
        REQUIRE(!json_content.empty());
        // 可以继续用 JSON 解析库解析 json_content
    }

    SECTION("load user preferences") {
        LOAD_JSON("user_prefs.json");
        REQUIRE(!json_content.empty());
    }
}
```

**关键设计点：**

1. `file_name` 参数必须是字符串字面量（如 `"app_config.json"`），这样才能与 `TEST_FILES_PATH "/json/"` 做编译期拼接
2. `do { ... } while(false)` 保证宏在任何语法上下文中都安全
3. 宏内部使用 `REQUIRE(ifs.is_open())` 确保文件存在——如果夹具文件缺失，测试立刻失败并给出明确错误，而不是后续得到空字符串的莫名错误
4. `json_content` 变量需要在宏外部声明，因为宏内部的 `do-while` 作用域结束后局部变量会被销毁

</details>

---

## 下一步

接下来进入实战环节！在 [选择模块与分析接口](../03-实战-为模块添加测试/01-选择模块与分析接口.md) 中，我们将选择一个真实的 KrKr2 模块（core/visual），分析它的公开接口，为编写测试用例做准备。


