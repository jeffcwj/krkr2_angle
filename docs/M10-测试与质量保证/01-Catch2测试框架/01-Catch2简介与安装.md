# Catch2 简介与安装

> **所属模块：** M10-测试与质量保证
> **前置知识：** [M01 — 项目导览与环境搭建](../../M01-项目导览与环境搭建/)、C++ 基础（模板、异常）
> **预计阅读时间：** 20 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 Catch2 v3 与其他 C++ 测试框架（Google Test、Boost.Test、doctest）的区别和优势
2. 通过 vcpkg 安装 Catch2 v3 并配置 CMake 集成
3. 编写并运行你的第一个 Catch2 测试程序
4. 理解 KrKr2 项目选择 Catch2 的技术原因

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 单元测试框架 | Unit Testing Framework | 提供编写、组织和运行测试用例的基础设施的库 |
| 头文件模式 | Header-Only | 整个库只由头文件组成，不需要单独编译链接——Catch2 v2 的模式 |
| 静态库模式 | Static Library | 库被预编译为 `.a`/`.lib` 文件，链接时合并到可执行文件中——Catch2 v3 的模式 |
| 测试发现 | Test Discovery | CMake/CTest 自动扫描测试可执行文件，找出所有注册的测试用例名称 |
| vcpkg | vcpkg | 微软开发的 C++ 包管理器，KrKr2 项目用它管理所有第三方依赖 |

## 为什么需要单元测试框架

### 手写测试的痛点

假设你写了一个字符串处理函数，想验证它是否正确。最原始的方法是在 `main()` 里手动调用并检查结果：

```cpp
#include <iostream>
#include <string>

// 被测函数：将字符串转为大写
std::string toUpper(const std::string& input) {
    std::string result = input;
    for (auto& c : result) {
        c = static_cast<char>(std::toupper(static_cast<unsigned char>(c)));
    }
    return result;
}

int main() {
    // 手动测试
    if (toUpper("hello") != "HELLO") {
        std::cerr << "测试失败: toUpper(\"hello\")" << std::endl;
        return 1;  // 返回非零表示失败
    }
    if (toUpper("") != "") {
        std::cerr << "测试失败: toUpper(\"\")" << std::endl;
        return 1;
    }
    if (toUpper("ABC") != "ABC") {
        std::cerr << "测试失败: toUpper(\"ABC\")" << std::endl;
        return 1;
    }
    std::cout << "所有测试通过！" << std::endl;
    return 0;
}
```

这种方式有几个严重的问题：

1. **第一个失败就停止**：如果第一个测试失败，后面的测试不会执行，你无法一次看到所有失败
2. **没有结构化输出**：失败信息完全靠手写 `cerr`，格式不统一，也无法生成测试报告
3. **代码重复**：每个检查都需要写 if/cerr/return，大量样板代码（Boilerplate Code，指重复但无实际逻辑的代码）
4. **无法选择性运行**：想只运行某一个测试？必须手动注释其他代码
5. **无法自动发现**：CI 系统无法自动找到并运行测试

单元测试框架就是为了解决这些问题而存在的。它提供：统一的断言宏、自动注册与发现测试用例、结构化的成功/失败报告、选择性运行（按名称过滤）等能力。

## C++ 主流测试框架对比

C++ 生态中有多个成熟的单元测试框架，以下是最常用的四个：

### Google Test (gtest)

Google Test 是 Google 开发的 C++ 测试框架，也是业界使用最广泛的选择之一。它提供丰富的断言宏（`ASSERT_EQ`、`EXPECT_TRUE` 等）、测试夹具类（`::testing::Test`）和参数化测试支持。Google Test 的特点是功能全面、文档丰富，但 API 相对繁琐——例如每个测试夹具都需要定义一个继承 `::testing::Test` 的类。

```cpp
// Google Test 风格
#include <gtest/gtest.h>

TEST(StringTest, ToUpper) {      // 测试套件名, 测试用例名
    EXPECT_EQ(toUpper("hello"), "HELLO");  // 非致命断言
    ASSERT_EQ(toUpper(""), "");            // 致命断言（失败则终止当前测试）
}
```

### Boost.Test

Boost.Test 是 Boost 库集合的一部分，功能非常强大，支持多种测试组织方式（自动注册、手动注册、测试套件嵌套等）。但它的 API 复杂度较高，配置选项多，学习曲线陡峭。而且 Boost.Test 依赖整个 Boost 库体系，引入一个测试框架可能带来大量不需要的依赖。

### doctest

doctest 是一个轻量级的 C++ 测试框架，API 与 Catch2 非常相似（`TEST_CASE`、`SUBCASE`、`REQUIRE` 等），但完全采用头文件模式（Header-Only），编译速度极快。doctest 的定位是"最快的 C++ 测试框架"，适合对编译时间敏感的项目。

### Catch2 v3

Catch2 是 Phil Nash 创建的现代 C++ 测试框架。v2 版本是头文件模式，v3 版本改为静态库模式以提升编译速度。Catch2 的核心理念是 **简洁的 API 设计**——用最少的代码写出表达力最强的测试。

```cpp
// Catch2 风格
#include <catch2/catch_test_macros.hpp>

TEST_CASE("toUpper 转换字符串为大写") {  // 用自然语言描述测试
    REQUIRE(toUpper("hello") == "HELLO");  // 失败时自动显示左右两边的值
    REQUIRE(toUpper("") == "");
}
```

### 对比总结

| 特性 | Google Test | Boost.Test | doctest | **Catch2 v3** |
|------|------------|------------|---------|---------------|
| 安装复杂度 | 中等（需编译） | 高（Boost 依赖） | 低（头文件） | **低（vcpkg 一行搞定）** |
| API 简洁度 | 中等 | 复杂 | 简洁 | **简洁** |
| 断言表达力 | `ASSERT_EQ(a, b)` | `BOOST_CHECK_EQUAL(a, b)` | `REQUIRE(a == b)` | **`REQUIRE(a == b)`** |
| 编译速度 | 快（预编译库） | 慢 | 极快 | **快（v3 预编译库）** |
| SECTION 机制 | 无（需夹具类） | 无 | 有（SUBCASE） | **有（SECTION）** |
| BDD 支持 | 无 | 无 | 无 | **有（SCENARIO/GIVEN/WHEN/THEN）** |
| Matcher 系统 | 有 | 有 | 无 | **有** |
| 生成器/参数化 | 有（宏驱动） | 有 | 无 | **有（GENERATE）** |
| 与 CMake 集成 | 好 | 中等 | 好 | **好（catch_discover_tests）** |

KrKr2 项目选择 Catch2 v3 的原因：
1. **API 简洁**：`REQUIRE(expr)` 比 `ASSERT_EQ(a, b)` 更自然，且失败时自动展示表达式两侧的值
2. **SECTION 机制**：同一个 TEST_CASE 内可以有多个 SECTION，共享初始化代码但独立运行——非常适合测试同一个对象的多个方面
3. **vcpkg 集成好**：项目已使用 vcpkg，`find_package(Catch2 3 REQUIRED)` 即可
4. **BDD 风格**：复杂业务逻辑可以用 GIVEN/WHEN/THEN 组织，可读性更好

## Catch2 v3 架构解析

### 从 Header-Only 到静态库

Catch2 v2 采用头文件模式（Header-Only）：你只需要 `#include "catch.hpp"` 一个文件即可使用全部功能。这在小项目中非常方便，但在大型项目中有严重的编译时间问题——每个包含 `catch.hpp` 的 `.cpp` 文件都要重新编译 Catch2 的全部模板代码，一个 200 文件的测试项目可能因此多花 5-10 分钟编译。

Catch2 v3 彻底重构为静态库模式（Static Library）。Catch2 的核心代码被预编译为 `.a`（Linux/macOS）或 `.lib`（Windows）文件，你的测试代码只需要包含轻量的头文件并链接预编译库。这带来了三个关键改进：

1. **编译速度大幅提升**：Catch2 自身只编译一次，后续每个测试文件的编译时间从数秒降到毫秒级
2. **头文件更细粒度**：v2 只有一个 `catch.hpp`，v3 拆分为多个头文件（`catch_test_macros.hpp`、`catch_matchers.hpp` 等），你只包含需要的部分
3. **更好的 IDE 支持**：预编译库意味着 IDE 不需要反复解析 Catch2 的模板，代码补全和跳转更快

### v3 头文件结构

```
catch2/
├── catch_test_macros.hpp        # TEST_CASE, SECTION, REQUIRE, CHECK 等核心宏
├── catch_session.hpp            # Catch::Session 类——自定义 main() 时使用
├── matchers/
│   ├── catch_matchers.hpp       # 匹配器基类和通用 Matcher
│   ├── catch_matchers_string.hpp    # 字符串匹配器（ContainsSubstring 等）
│   └── catch_matchers_floating_point.hpp  # 浮点数匹配器（WithinAbs 等）
├── generators/
│   └── catch_generators.hpp     # 参数化测试的 GENERATE 宏和生成器
├── catch_approx.hpp             # Approx 浮点数近似比较
└── catch_template_test_macros.hpp  # 模板测试宏（TEMPLATE_TEST_CASE）
```

在项目中，最常用的头文件是 `catch_test_macros.hpp`（提供 `TEST_CASE`、`REQUIRE` 等基础宏）和 `catch_session.hpp`（用于自定义 `main()` 函数）。KrKr2 的每个测试文件都只包含 `catch_test_macros.hpp`，而 `main.cpp` 文件包含 `catch_session.hpp`。

### CMake 目标

Catch2 v3 通过 CMake 提供两个链接目标（Link Target）：

| CMake 目标 | 用途 | 包含内容 |
|-----------|------|---------|
| `Catch2::Catch2` | 核心库 | 断言宏、SECTION、Matcher、Generator 等所有功能 |
| `Catch2::Catch2WithMain` | 核心库 + 默认 main | 自动提供 `int main()` 入口，省去手写 main 的麻烦 |

如果你的测试不需要在运行前做任何初始化（比如创建日志器、连接数据库等），用 `Catch2::Catch2WithMain` 最省事。但 KrKr2 项目的引擎代码在运行时会调用 `spdlog::get("core")` 等函数获取日志器，如果没有提前创建这些日志器就会崩溃。因此 KrKr2 **必须** 使用 `Catch2::Catch2` 并手写 `main()` 来初始化 spdlog——这一点我们会在第 02 章详细展开。

## 通过 vcpkg 安装 Catch2 v3

KrKr2 项目使用 vcpkg 管理所有第三方依赖（参见 [P02 — vcpkg 包管理](../../P02-vcpkg包管理/)），Catch2 v3 也不例外。安装只需要在 `vcpkg.json` 清单（Manifest）文件中添加一行依赖声明。

### 第一步：检查 vcpkg.json

打开项目根目录的 `vcpkg.json`，确认 `dependencies` 数组中包含 `catch2`：

```json
{
    "name": "krkr2",
    "version-string": "1.0.0",
    "dependencies": [
        "catch2",
        "spdlog",
        "ffmpeg",
        "..."
    ]
}
```

> **提示**：vcpkg 的 `catch2` 端口（Port）默认安装 v3 最新版。如果你需要锁定特定版本，可以在 `overrides` 中指定，但通常不需要。

### 第二步：vcpkg 自动安装

在 vcpkg 清单模式（Manifest Mode）下，运行 CMake 配置时 vcpkg 会自动安装所有依赖。你不需要手动执行 `vcpkg install`：

```bash
# Windows — vcpkg 自动安装依赖
cmake --preset="Windows Debug Config"
# 输出中会看到类似：
# -- Running vcpkg install
# -- Installing 1/N catch2:x64-windows...
# -- Elapsed time to handle catch2:x64-windows: 15.2s

# Linux
cmake --preset="Linux Debug Config"

# macOS
cmake --preset="MacOS Debug Config"
```

如果你没有使用 CMakePresets.json（比如在独立的学习项目中），需要手动指定 vcpkg 工具链文件：

```bash
# 手动指定 vcpkg 工具链
cmake -B build \
    -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
    -DCMAKE_BUILD_TYPE=Debug

# Windows PowerShell
cmake -B build `
    -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake" `
    -DCMAKE_BUILD_TYPE=Debug
```

### 安装验证

安装完成后，可以在 vcpkg 安装目录中确认 Catch2 的存在：

```bash
# Linux/macOS — 检查头文件
ls $VCPKG_ROOT/installed/x64-linux/include/catch2/
# 应该看到：catch_test_macros.hpp, catch_session.hpp 等

# Windows — 检查库文件
dir %VCPKG_ROOT%\installed\x64-windows\lib\
# 应该看到：Catch2.lib, Catch2Main.lib

# 或者通过 vcpkg list 确认
vcpkg list | grep catch2
# 输出类似：catch2:x64-linux    3.5.2    A modern, C++-native...
```

### 常见安装错误及解决方案

**错误 1：`Could not find a package configuration file provided by "Catch2"`**

```
CMake Error at tests/CMakeLists.txt:4 (find_package):
  Could not find a package configuration file provided by "Catch2" (requested version 3)
```

原因：vcpkg 没有正确安装 Catch2，或者 CMake 没有使用 vcpkg 工具链。解决方案：
- 确认 `vcpkg.json` 中有 `"catch2"` 依赖
- 确认 `CMAKE_TOOLCHAIN_FILE` 正确指向 vcpkg
- 运行 `vcpkg install catch2` 手动安装试试

**错误 2：`find_package(Catch2 3)` 找到了 v2 而非 v3**

原因：系统中同时安装了 Catch2 v2 和 v3。解决方案：
- 确保 vcpkg 的 `catch2` 端口是最新的（v3.x）
- 在 CMake 中用 `find_package(Catch2 3 REQUIRED)` 明确要求 v3

## CMake 集成配置

安装好 Catch2 后，需要在 CMakeLists.txt 中配置测试目标。下面是一个最小化的完整示例：

### 最简 CMake 配置（使用 Catch2WithMain）

```cmake
# CMakeLists.txt — 最简单的 Catch2 集成
cmake_minimum_required(VERSION 3.14)  # catch_discover_tests 需要 3.14+
project(my_test_project LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)           # Catch2 v3 需要 C++17

# 查找 Catch2 v3 包（vcpkg 自动提供路径）
find_package(Catch2 3 REQUIRED)

# 启用 CTest 测试功能
enable_testing()

# 创建测试可执行文件
add_executable(my_tests
    test_string.cpp   # 你的测试源文件
)

# 链接 Catch2WithMain（自动提供 main 函数）
target_link_libraries(my_tests
    PRIVATE Catch2::Catch2WithMain
)

# 让 CTest 自动发现所有 TEST_CASE
include(Catch2::Catch)             # 引入 Catch2 的 CMake 辅助模块
catch_discover_tests(my_tests)     # 扫描可执行文件，注册所有测试到 CTest
```

`catch_discover_tests()` 是 Catch2 提供的 CMake 函数，它在构建后自动运行测试可执行文件的 `--list-tests` 命令，把发现的每个 `TEST_CASE` 注册为一个独立的 CTest 测试。这意味着你可以用 `ctest` 命令选择性地运行某个测试，而不需要运行整个测试程序。

### KrKr2 的 CMake 配置（自定义 main）

KrKr2 项目因为需要初始化 spdlog 日志器，不能使用 `Catch2WithMain`。它的 CMake 配置稍有不同：

```cmake
# tests/CMakeLists.txt（简化版，展示核心思路）
# 来源：tests/CMakeLists.txt 第 1-18 行

find_package(Catch2 3 REQUIRED)   # 查找 Catch2 v3
enable_testing()

# 配置 test_config.h —— 生成 TEST_FILES_PATH 宏
set(TEST_FILES_PATH "${CMAKE_CURRENT_SOURCE_DIR}/test_files")
set(TEST_CONFIG_DIR "${CMAKE_CURRENT_BINARY_DIR}")
configure_file(
    test_config.h.in              # 模板文件
    test_config.h                 # 生成的头文件
)

# 添加各模块的测试子目录
add_subdirectory(unit-tests/core/movie)
add_subdirectory(unit-tests/core/tjs2)
add_subdirectory(unit-tests/core/visual)
add_subdirectory(unit-tests/plugins)
```

每个子目录中的 CMakeLists.txt 使用一个循环模式——为每个 `.cpp` 测试文件创建独立的可执行文件：

```cmake
# tests/unit-tests/plugins/CMakeLists.txt（简化版）
# 来源：tests/unit-tests/plugins/CMakeLists.txt 第 1-28 行

set(SOURCES
    psbfile-dll.cpp              # PSB 文件解析测试
)

# 从文件名中提取目标名（去掉 .cpp 后缀）
string(REPLACE ".cpp" "" TARGETS "${SOURCES}")

foreach(name ${TARGETS})
    # 每个测试文件 + main.cpp 组成一个独立可执行文件
    add_executable(${name} ${name}.cpp main.cpp)

    # 链接 Catch2 核心库（不带 main）+ 项目库
    target_link_libraries(${name}
        PRIVATE Catch2::Catch2        # Catch2 核心（不带 main）
        PUBLIC krkr2plugin krkr2core  # 项目插件库和核心库
    )

    # 包含 test_config.h 的路径
    target_include_directories(${name}
        PRIVATE ${TEST_CONFIG_DIR}
    )

    # 注册到 CTest
    include(Catch2::Catch)
    catch_discover_tests(${name})
endforeach()
```

这个模式的优点是：每添加一个新的测试文件，只需要在 `SOURCES` 列表中加一行，CMake 会自动为它创建可执行文件并注册到 CTest。

## 编写第一个 Catch2 测试

现在来从零编写一个完整的 Catch2 测试程序。我们先用最简单的 `Catch2WithMain` 方式，把注意力集中在测试语法本身。

### 被测代码：简单的数学工具库

创建一个 `math_utils.hpp` 头文件：

```cpp
// math_utils.hpp — 简单的数学工具函数
#pragma once
#include <cmath>      // std::abs, std::sqrt
#include <stdexcept>  // std::invalid_argument
#include <vector>     // std::vector
#include <numeric>    // std::accumulate

// 计算阶乘（非负整数）
// 参数 n 必须 >= 0，否则抛出异常
int factorial(int n) {
    if (n < 0) {
        throw std::invalid_argument(
            "阶乘不接受负数输入");
    }
    int result = 1;
    for (int i = 2; i <= n; ++i) {
        result *= i;  // 逐步累乘
    }
    return result;
}

// 计算向量的平均值
// 空向量抛出异常
double average(const std::vector<double>& values) {
    if (values.empty()) {
        throw std::invalid_argument(
            "无法计算空向量的平均值");
    }
    double sum = std::accumulate(
        values.begin(), values.end(), 0.0);
    return sum / static_cast<double>(values.size());
}

// 判断两个浮点数是否近似相等
// epsilon 为容差阈值
bool approxEqual(double a, double b,
                 double epsilon = 1e-9) {
    return std::abs(a - b) < epsilon;
}
```

### 测试代码

创建 `test_math.cpp`：

```cpp
// test_math.cpp — 第一个 Catch2 测试文件
#include <catch2/catch_test_macros.hpp>  // TEST_CASE, REQUIRE, CHECK
#include "math_utils.hpp"               // 被测代码

// TEST_CASE 宏定义一个测试用例
// 第一个参数是测试名称（自然语言描述）
// 第二个参数是标签（可选），用于过滤运行
TEST_CASE("factorial 计算阶乘", "[math]") {
    // REQUIRE 是"致命断言"——失败则立即终止当前测试用例
    REQUIRE(factorial(0) == 1);   // 0! = 1（数学定义）
    REQUIRE(factorial(1) == 1);   // 1! = 1
    REQUIRE(factorial(5) == 120); // 5! = 120
    REQUIRE(factorial(10) == 3628800);  // 10! = 3628800

    // CHECK 是"非致命断言"——失败后继续执行后续断言
    // 适合想一次看到所有失败的场景
    CHECK(factorial(3) == 6);     // 3! = 6
    CHECK(factorial(4) == 24);    // 4! = 24
}

TEST_CASE("factorial 负数输入抛出异常", "[math]") {
    // REQUIRE_THROWS_AS 验证表达式抛出指定类型的异常
    REQUIRE_THROWS_AS(
        factorial(-1),
        std::invalid_argument
    );
    REQUIRE_THROWS_AS(
        factorial(-100),
        std::invalid_argument
    );
}

TEST_CASE("average 计算平均值", "[math]") {
    // 单元素向量
    REQUIRE(average({42.0}) == 42.0);

    // 多元素向量——浮点数比较用 Approx
    REQUIRE(average({1.0, 2.0, 3.0})
            == Catch::Approx(2.0));

    // 包含负数的向量
    REQUIRE(average({-1.0, 0.0, 1.0})
            == Catch::Approx(0.0));
}

TEST_CASE("average 空向量抛出异常", "[math]") {
    REQUIRE_THROWS_AS(
        average({}),
        std::invalid_argument
    );
}

TEST_CASE("approxEqual 浮点近似比较", "[math]") {
    // 完全相等
    REQUIRE(approxEqual(1.0, 1.0));

    // 微小差异——在默认 epsilon 范围内
    REQUIRE(approxEqual(1.0, 1.0 + 1e-10));

    // 超出 epsilon 范围
    REQUIRE_FALSE(approxEqual(1.0, 1.1));

    // 自定义 epsilon
    REQUIRE(approxEqual(1.0, 1.05, 0.1));
    REQUIRE_FALSE(approxEqual(1.0, 1.05, 0.01));
}
```

### CMakeLists.txt

```cmake
# CMakeLists.txt — 第一个测试项目的完整构建文件
cmake_minimum_required(VERSION 3.14)
project(first_catch2_test LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 查找 Catch2 v3
find_package(Catch2 3 REQUIRED)

# 启用 CTest
enable_testing()

# 创建测试可执行文件
add_executable(test_math test_math.cpp)

# 链接 Catch2WithMain（自动提供 main）
target_link_libraries(test_math
    PRIVATE Catch2::Catch2WithMain
)

# 自动发现并注册测试
include(Catch2::Catch)
catch_discover_tests(test_math)
```

### 构建与运行

```bash
# 创建项目目录
mkdir first_catch2_test && cd first_catch2_test
# 把上面三个文件放入目录

# 配置（需要 VCPKG_ROOT 环境变量）
cmake -B build \
    -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake

# 编译
cmake --build build

# 方式 1：直接运行测试可执行文件
./build/test_math
# 输出类似：
# =========================================
# All tests passed (12 assertions in 5 test cases)

# 方式 2：通过 CTest 运行（推荐）
ctest --test-dir build --output-on-failure
# 输出类似：
# Test project /path/to/build
#     Start 1: factorial 计算阶乘
# 1/5 Test #1: factorial 计算阶乘 ............   Passed    0.01 sec
#     Start 2: factorial 负数输入抛出异常
# 2/5 Test #2: factorial 负数输入抛出异常 .....   Passed    0.01 sec
# ...
# 100% tests passed, 0 tests failed out of 5

# 方式 3：只运行带特定标签的测试
./build/test_math "[math]"

# 方式 4：按名称过滤
./build/test_math "factorial*"
# 只运行名称以 "factorial" 开头的测试用例
```

> **Windows 用户注意**：把 `./build/test_math` 替换为 `.\build\Debug\test_math.exe`（MSVC 默认把输出放在 `Debug/` 子目录下）。

## 动手实践

按照以下步骤，在你的机器上创建并运行第一个 Catch2 测试项目。

### 步骤 1：创建项目结构

```bash
# 创建目录
mkdir catch2_hello && cd catch2_hello

# 创建 vcpkg.json（声明 Catch2 依赖）
# Windows 用户可以用记事本手动创建
```

在 `catch2_hello/` 目录下创建 `vcpkg.json`：

```json
{
    "name": "catch2-hello",
    "version-string": "1.0.0",
    "dependencies": [
        "catch2"
    ]
}
```

### 步骤 2：编写被测代码

创建 `string_utils.hpp`：

```cpp
// string_utils.hpp — 用于测试的字符串工具
#pragma once
#include <string>
#include <algorithm>  // std::transform, std::reverse

// 将字符串转为大写
std::string toUpper(const std::string& s) {
    std::string result = s;
    std::transform(
        result.begin(), result.end(),
        result.begin(),
        [](unsigned char c) {
            return std::toupper(c);  // 逐字符转大写
        }
    );
    return result;
}

// 反转字符串
std::string reverse(const std::string& s) {
    std::string result = s;
    std::reverse(result.begin(), result.end());
    return result;
}

// 检查字符串是否为回文
bool isPalindrome(const std::string& s) {
    // 空字符串和单字符视为回文
    if (s.size() <= 1) return true;
    auto left = s.begin();
    auto right = s.end() - 1;
    while (left < right) {
        if (*left != *right) return false;
        ++left;
        --right;
    }
    return true;
}
```

### 步骤 3：编写测试

创建 `test_string.cpp`：

```cpp
// test_string.cpp — 字符串工具的测试
#include <catch2/catch_test_macros.hpp>
#include "string_utils.hpp"

TEST_CASE("toUpper 将字符串转为大写",
          "[string]") {
    REQUIRE(toUpper("hello") == "HELLO");
    REQUIRE(toUpper("") == "");
    REQUIRE(toUpper("ABC") == "ABC");
    REQUIRE(toUpper("Hello World")
            == "HELLO WORLD");
}

TEST_CASE("reverse 反转字符串", "[string]") {
    REQUIRE(reverse("hello") == "olleh");
    REQUIRE(reverse("") == "");
    REQUIRE(reverse("a") == "a");
    REQUIRE(reverse("abcde") == "edcba");
}

TEST_CASE("isPalindrome 检查回文",
          "[string]") {
    // 回文字符串
    REQUIRE(isPalindrome(""));
    REQUIRE(isPalindrome("a"));
    REQUIRE(isPalindrome("aba"));
    REQUIRE(isPalindrome("abba"));

    // 非回文字符串
    REQUIRE_FALSE(isPalindrome("abc"));
    REQUIRE_FALSE(isPalindrome("hello"));
}
```

### 步骤 4：创建 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.14)
project(catch2_hello LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Catch2 3 REQUIRED)
enable_testing()

add_executable(test_string test_string.cpp)
target_link_libraries(test_string
    PRIVATE Catch2::Catch2WithMain
)

include(Catch2::Catch)
catch_discover_tests(test_string)
```

### 步骤 5：构建并运行

```bash
# 配置
cmake -B build \
    -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake

# 编译
cmake --build build

# 运行测试
ctest --test-dir build --output-on-failure -V
# -V 表示 verbose，显示每个测试的详细输出
```

预期输出：

```
Test project /path/to/build
    Start 1: toUpper 将字符串转为大写
1/3 Test #1: toUpper 将字符串转为大写 ......   Passed    0.01 sec
    Start 2: reverse 反转字符串
2/3 Test #2: reverse 反转字符串 ...........   Passed    0.01 sec
    Start 3: isPalindrome 检查回文
3/3 Test #3: isPalindrome 检查回文 .........   Passed    0.01 sec

100% tests passed, 0 tests failed out of 3
```

> **验证要点**：如果你看到 `100% tests passed`，说明 Catch2 安装、CMake 配置、测试编写都正确。如果某个测试失败，`--output-on-failure` 选项会显示 Catch2 的详细失败信息，包括期望值和实际值的对比。

## 对照项目源码

KrKr2 项目的 Catch2 配置分布在以下文件中，你可以对照上面的教程内容逐一理解：

### 根测试配置

- **`tests/CMakeLists.txt`** 第 1-18 行 — 根测试目录的 CMake 配置。关键操作：`find_package(Catch2 3 REQUIRED)`（声明依赖 Catch2 v3），`configure_file(test_config.h.in test_config.h)`（生成包含 `TEST_FILES_PATH` 宏的头文件），以及 4 个 `add_subdirectory` 调用（movie、tjs2、visual、plugins）

### 插件测试配置

- **`tests/unit-tests/plugins/CMakeLists.txt`** 第 1-28 行 — 插件测试的 CMake 配置。展示了 foreach 模式：`string(REPLACE ".cpp" "" TARGETS "${SOURCES}")` 从源文件名生成目标名，然后循环创建可执行文件。注意链接库是 `Catch2::Catch2`（不带 Main）而非 `Catch2::Catch2WithMain`，因为项目使用自定义 `main.cpp`

### TJS2 测试配置

- **`tests/unit-tests/core/tjs2/CMakeLists.txt`** 第 1-29 行 — TJS2 脚本引擎测试的 CMake 配置。与插件的模式相同，但 SOURCES 包含两个文件（`tjs.cpp` 和 `tjsString.cpp`），链接目标是 `krkr2core`（不需要 plugin 库）

### 最简测试示例

- **`tests/unit-tests/core/movie/ffmpeg.cpp`** 第 1-8 行 — 项目中最简单的测试文件，只有一个 TEST_CASE 和一行函数调用 `TVPInitLibAVCodec()`。这展示了 Catch2 测试可以非常简洁

```cpp
// tests/unit-tests/core/movie/ffmpeg.cpp（完整内容）
// 来源：tests/unit-tests/core/movie/ffmpeg.cpp 第 1-8 行
#include <catch2/catch_test_macros.hpp>
#include "movie/ffmpeg/FFManager.h"

TEST_CASE("ffmpeg init", "[movie]") {
    TVPInitLibAVCodec();  // 验证 FFmpeg 初始化不崩溃
}
```

> **观察要点**：这个测试没有 `REQUIRE` 断言——它只验证函数调用不会抛出异常或崩溃（Crash）。这种"冒烟测试"（Smoke Test，指只验证程序能正常启动/运行不崩溃，不验证具体功能的测试）在大型项目中很常见。

## 本节小结

- **单元测试框架** 解决了手写测试的五大痛点：第一个失败就停止、没有结构化输出、大量样板代码、无法选择性运行、无法自动发现
- **C++ 四大测试框架** 各有特点：Google Test 功能全面但 API 繁琐，Boost.Test 强大但依赖重，doctest 编译极快但功能少，Catch2 v3 在 API 简洁性和功能完备性之间取得了最佳平衡
- **Catch2 v3 从 Header-Only 改为静态库**，编译速度大幅提升，头文件也拆分为更细粒度的模块
- **vcpkg 安装** 只需在 `vcpkg.json` 中声明依赖，CMake 配置时自动安装
- **CMake 集成** 核心三步：`find_package(Catch2 3)`、`target_link_libraries(... Catch2::Catch2WithMain)`、`catch_discover_tests(...)`
- **KrKr2 使用自定义 main()** 而非 `Catch2WithMain`，因为引擎代码需要预先初始化 spdlog 日志器

## 练习题与答案

### 题目 1：为什么 Catch2 v3 从头文件模式改为静态库模式？

<details>
<summary>查看答案</summary>

Catch2 v2 采用头文件模式（Header-Only），整个库的实现代码都在头文件中。这意味着每个 `#include "catch.hpp"` 的 `.cpp` 文件在编译时都需要重新编译 Catch2 的全部模板和函数实现，导致编译时间随测试文件数量线性增长。在大型项目（200+ 测试文件）中，这可能带来 5-10 分钟的额外编译时间。

Catch2 v3 改为静态库模式后，Catch2 的核心代码只编译一次，生成 `.a`/`.lib` 文件。每个测试文件只需要包含轻量的声明头文件（不包含实现），编译时间从"秒级/文件"降到"毫秒级/文件"。同时 v3 还将原来的单一头文件拆分为多个功能模块（`catch_test_macros.hpp`、`catch_matchers.hpp` 等），让你只包含需要的部分，进一步减少编译负担。

</details>

### 题目 2：编写一个完整的 Catch2 测试程序

为以下函数编写测试（至少 3 个 TEST_CASE，覆盖正常输入、边界情况和异常输入）：

```cpp
// 计算字符串中某个字符出现的次数
int countChar(const std::string& str, char target) {
    int count = 0;
    for (char c : str) {
        if (c == target) ++count;
    }
    return count;
}
```

<details>
<summary>查看答案</summary>

```cpp
// test_count_char.cpp — countChar 函数的完整测试
#include <catch2/catch_test_macros.hpp>
#include <string>

// 被测函数（通常在单独的头文件中）
int countChar(const std::string& str, char target) {
    int count = 0;
    for (char c : str) {
        if (c == target) ++count;
    }
    return count;
}

TEST_CASE("countChar 正常输入", "[string]") {
    REQUIRE(countChar("hello", 'l') == 2);
    REQUIRE(countChar("hello", 'h') == 1);
    REQUIRE(countChar("hello", 'o') == 1);
    REQUIRE(countChar("aaa", 'a') == 3);
}

TEST_CASE("countChar 字符不存在", "[string]") {
    REQUIRE(countChar("hello", 'z') == 0);
    REQUIRE(countChar("hello", 'H') == 0);
    // 注意：大小写敏感，'H' 和 'h' 是不同字符
}

TEST_CASE("countChar 边界情况", "[string]") {
    // 空字符串
    REQUIRE(countChar("", 'a') == 0);

    // 单字符字符串
    REQUIRE(countChar("a", 'a') == 1);
    REQUIRE(countChar("b", 'a') == 0);

    // 包含特殊字符
    REQUIRE(countChar("a b c", ' ') == 2);
    REQUIRE(countChar("a\nb\nc", '\n') == 2);
}
```

对应的 CMakeLists.txt：

```cmake
cmake_minimum_required(VERSION 3.14)
project(test_countchar LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Catch2 3 REQUIRED)
enable_testing()

add_executable(test_count_char test_count_char.cpp)
target_link_libraries(test_count_char
    PRIVATE Catch2::Catch2WithMain
)

include(Catch2::Catch)
catch_discover_tests(test_count_char)
```

</details>

### 题目 3：解释 `Catch2::Catch2` 与 `Catch2::Catch2WithMain` 的区别

在什么场景下必须使用 `Catch2::Catch2` 而不是 `Catch2::Catch2WithMain`？请结合 KrKr2 项目说明。

<details>
<summary>查看答案</summary>

**`Catch2::Catch2WithMain`** 包含 Catch2 的核心库以及一个默认的 `int main()` 入口函数。默认 main 会直接调用 `Catch::Session().run(argc, argv)` 来运行所有测试。适合不需要任何初始化工作的简单测试项目。

**`Catch2::Catch2`** 只包含核心库，不提供 main 函数。你必须自己编写 `main.cpp` 并手动调用 `Catch::Session().run()`。适合需要在测试运行前进行初始化的项目。

KrKr2 项目**必须**使用 `Catch2::Catch2` 的原因：KrKr2 的引擎代码在运行时会调用 `spdlog::get("core")`、`spdlog::get("tjs2")` 等函数获取预创建的日志器（Logger）。如果这些日志器在测试运行前没有被创建，`spdlog::get()` 会返回空指针，导致后续的日志调用崩溃。因此，KrKr2 在每个测试模块的 `main.cpp` 中先使用 `spdlog::stdout_color_mt("core")` 等调用创建所有必需的日志器，然后再调用 `Catch::Session().run()` 启动测试。这个自定义初始化步骤用 `Catch2WithMain` 无法实现。

</details>

## 下一步

[02 — TEST_CASE 与断言](./02-TEST_CASE与断言.md)：深入学习 Catch2 的核心断言宏（`REQUIRE`、`CHECK`、`REQUIRE_THROWS` 等），掌握字符串匹配、浮点数比较、异常验证等高级断言技巧。
