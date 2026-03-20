# TEST_CASE 与断言

> **所属模块：** M10-测试与质量保证
> **前置知识：** [01 — Catch2 简介与安装](./01-Catch2简介与安装.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：

1. 使用 `TEST_CASE` 定义测试用例，并用标签（Tag）对测试进行分类和过滤
2. 区分 `REQUIRE`（致命断言）和 `CHECK`（非致命断言）的使用场景
3. 使用 `REQUIRE_THROWS`、`REQUIRE_THROWS_AS`、`REQUIRE_NOTHROW` 等异常断言验证错误处理
4. 用 `Catch::Approx` 和 Matcher 进行浮点数和字符串的精确比较
5. 使用 `CAPTURE` 和 `INFO` 宏在断言失败时输出调试信息

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 致命断言 | Fatal Assertion | 失败后立即终止当前测试用例，后续代码不再执行——对应 `REQUIRE` |
| 非致命断言 | Non-fatal Assertion | 失败后记录错误但继续执行后续代码，让你一次看到所有失败——对应 `CHECK` |
| 标签 | Tag | 附加在 TEST_CASE 上的分类标记（如 `[math]`、`[slow]`），用于选择性运行测试 |
| 表达式分解 | Expression Decomposition | Catch2 自动将 `REQUIRE(a == b)` 分解为左右两个值，失败时同时显示 a 和 b 的实际值 |
| 近似比较 | Approximate Comparison | 使用 `Catch::Approx` 对浮点数进行带容差（Tolerance）的比较，避免浮点精度问题 |
| 异常断言 | Exception Assertion | 验证某个表达式是否抛出了指定类型的异常——如 `REQUIRE_THROWS_AS` |

## TEST_CASE 详解

### 基本语法

`TEST_CASE` 是 Catch2 中定义测试用例的核心宏。它的完整签名如下：

```cpp
TEST_CASE("测试名称", "[标签1][标签2]") {
    // 测试代码
}
```

- **第一个参数**（必填）：测试名称，使用自然语言描述这个测试要验证什么。名称在整个测试程序中必须唯一
- **第二个参数**（可选）：标签列表，每个标签用方括号包裹。标签用于对测试分组和过滤

```cpp
#include <catch2/catch_test_macros.hpp>

// 最简形式——只有名称，没有标签
TEST_CASE("1 加 1 等于 2") {
    REQUIRE(1 + 1 == 2);
}

// 带标签——可以按标签过滤运行
TEST_CASE("字符串拼接", "[string][basic]") {
    std::string hello = "Hello";
    hello += " World";
    REQUIRE(hello == "Hello World");
}

// 多标签——一个测试可以有任意多个标签
TEST_CASE("大数阶乘", "[math][slow][!mayfail]") {
    // [!mayfail] 是 Catch2 的特殊标签，
    // 表示此测试允许失败但不影响整体结果
    REQUIRE(factorial(20) == 2432902008176640000LL);
}
```

### 标签系统

标签是 Catch2 中非常强大的测试组织机制。常用的标签模式包括：

| 标签格式 | 含义 | 使用场景 |
|----------|------|---------|
| `[math]` | 普通分组标签 | 按功能域分组（`[string]`、`[io]`、`[network]`） |
| `[.slow]` | 隐藏标签（以 `.` 开头） | 耗时测试，默认不运行，需要 `[.slow]` 显式指定 |
| `[!mayfail]` | 允许失败 | 已知不稳定的测试，失败不影响整体结果 |
| `[!shouldfail]` | 必须失败 | 验证某个 bug 是否仍存在（修复后改为正常标签） |
| `[#开发者名]` | 作者标签 | 标记谁写的测试，方便追踪 |

通过命令行参数按标签过滤运行测试：

```bash
# 只运行带 [math] 标签的测试
./test_app "[math]"

# 运行带 [math] 或 [string] 标签的测试
./test_app "[math],[string]"

# 运行带 [math] 且带 [fast] 标签的测试（AND 逻辑）
./test_app "[math][fast]"

# 排除带 [slow] 标签的测试
./test_app "~[slow]"

# 运行所有测试（包括隐藏标签的）
./test_app "[.]"

# 按名称模糊匹配
./test_app "factorial*"
```

> **最佳实践**：KrKr2 项目使用模块名作为标签——`[movie]`、`[tjs2]`、`[plugin]` 等。这样你可以只运行与当前开发模块相关的测试，而不用等待整个测试套件跑完。

## 断言：REQUIRE 与 CHECK

断言（Assertion）是测试的核心——它们验证代码的实际行为是否符合预期。Catch2 提供两类断言：致命断言（Fatal Assertion）和非致命断言（Non-fatal Assertion）。

### REQUIRE——致命断言

`REQUIRE(expression)` 是最常用的断言宏。如果表达式为 `false`，当前测试用例**立即终止**，后续代码不再执行。适合验证前置条件——如果前置条件不满足，后续断言也没有意义。

```cpp
#include <catch2/catch_test_macros.hpp>
#include <vector>

TEST_CASE("REQUIRE 致命断言示例", "[basic]") {
    std::vector<int> v = {10, 20, 30};

    // 先验证容器不为空（前置条件）
    REQUIRE(!v.empty());          // 如果为空，后面访问 v[0] 会越界

    // 前置条件满足后，安全地验证具体值
    REQUIRE(v[0] == 10);          // 验证第一个元素
    REQUIRE(v.size() == 3);       // 验证元素个数
    REQUIRE(v.back() == 30);      // 验证最后一个元素
}
```

### CHECK——非致命断言

`CHECK(expression)` 在表达式为 `false` 时**记录失败但继续执行**后续代码。适合一次验证多个独立条件，想看到所有失败而不是只看到第一个。

```cpp
#include <catch2/catch_test_macros.hpp>
#include <string>

std::string formatName(const std::string& first,
                       const std::string& last) {
    return first + " " + last;
}

TEST_CASE("CHECK 非致命断言示例", "[basic]") {
    // 多个独立检查——即使前面的失败，后面的仍然执行
    CHECK(formatName("张", "三") == "张 三");
    CHECK(formatName("李", "四") == "李 四");
    CHECK(formatName("", "王") == " 王");        // 验证空名
    CHECK(formatName("赵", "") == "赵 ");         // 验证空姓
}
```

如果第一个 `CHECK` 失败，后面三个仍然会执行。测试报告会显示所有失败的断言。

### REQUIRE 与 CHECK 的选择原则

```
┌──────────────────────────────────────────────┐
│  断言选择决策树                                │
│                                              │
│  这个断言是后续断言的前置条件吗？               │
│  ├── 是 → 用 REQUIRE（失败则终止，避免崩溃）    │
│  └── 否 → 这些断言之间是独立的吗？              │
│        ├── 是 → 用 CHECK（看到所有失败）        │
│        └── 否 → 用 REQUIRE（有依赖关系）        │
└──────────────────────────────────────────────┘
```

经验法则：**不确定时用 REQUIRE**——它更安全。`CHECK` 只在你确定断言之间完全独立时使用。

### 表达式分解（Expression Decomposition）

Catch2 的一大亮点是**表达式分解**——当断言失败时，Catch2 自动将表达式两侧的值展示出来。这与 Google Test 的 `ASSERT_EQ(a, b)` 完全不同：

```cpp
TEST_CASE("表达式分解演示", "[basic]") {
    int a = 42;
    int b = 43;

    // Catch2 风格——自动分解
    REQUIRE(a == b);
    // 失败输出：
    // REQUIRE( a == b )
    // with expansion:
    //   42 == 43

    // 对比 Google Test 风格：
    // ASSERT_EQ(a, b);
    // 失败输出：
    //   Expected: a
    //     Which is: 42
    //   To be equal to: b
    //     Which is: 43
}
```

Catch2 的表达式分解不仅支持 `==`，还支持所有比较运算符：

```cpp
TEST_CASE("各种比较运算符", "[basic]") {
    int x = 5;

    REQUIRE(x > 0);      // 失败时显示：5 > 0
    REQUIRE(x < 10);     // 失败时显示：5 < 10
    REQUIRE(x >= 5);     // 失败时显示：5 >= 5
    REQUIRE(x <= 5);     // 失败时显示：5 <= 5
    REQUIRE(x != 0);     // 失败时显示：5 != 0

    std::string s = "hello";
    REQUIRE(s == "hello");  // 字符串也自动展示
    // 如果失败，显示："hello" == "hello"
}
```

> **注意**：表达式分解有一个限制——不支持复合逻辑运算符 `&&` 和 `||`。如果你需要验证复合条件，请拆分为多个 `REQUIRE`，或者使用 `REQUIRE((a > 0 && a < 100))` 加额外括号（但这样失败时只显示 `true/false`，不展示具体值）。

## 异常断言

测试不仅要验证"正确输入产生正确输出"，还要验证"错误输入产生正确的错误处理"。Catch2 提供了一组异常断言宏。

### REQUIRE_THROWS —— 验证抛出任意异常

```cpp
#include <catch2/catch_test_macros.hpp>
#include <stdexcept>
#include <vector>

TEST_CASE("REQUIRE_THROWS 验证抛出异常",
          "[exception]") {
    std::vector<int> v = {1, 2, 3};

    // at() 越界时抛出 std::out_of_range
    REQUIRE_THROWS(v.at(100));

    // 不关心异常类型，只要求抛出了异常即可
    REQUIRE_THROWS(v.at(-1));
}
```

### REQUIRE_THROWS_AS —— 验证抛出指定类型

```cpp
#include <catch2/catch_test_macros.hpp>
#include <stdexcept>

int divide(int a, int b) {
    if (b == 0) {
        throw std::invalid_argument(
            "除数不能为零");
    }
    return a / b;
}

TEST_CASE("REQUIRE_THROWS_AS 验证异常类型",
          "[exception]") {
    // 验证抛出 std::invalid_argument（而非其他异常）
    REQUIRE_THROWS_AS(
        divide(10, 0),
        std::invalid_argument
    );

    // 也可以用基类匹配
    REQUIRE_THROWS_AS(
        divide(10, 0),
        std::exception  // invalid_argument 是其子类
    );
}
```

### REQUIRE_THROWS_WITH —— 验证异常消息

```cpp
TEST_CASE("REQUIRE_THROWS_WITH 验证异常消息",
          "[exception]") {
    // 验证异常的 what() 消息完全匹配
    REQUIRE_THROWS_WITH(
        divide(10, 0),
        "除数不能为零"
    );

    // 也支持 Matcher（如 ContainsSubstring）
    // 后面的 Matcher 章节会详细讲
}
```

### REQUIRE_NOTHROW —— 验证不抛出异常

```cpp
TEST_CASE("REQUIRE_NOTHROW 验证不抛出异常",
          "[exception]") {
    // 验证正常输入不会抛出异常
    REQUIRE_NOTHROW(divide(10, 2));
    REQUIRE_NOTHROW(divide(0, 1));
    REQUIRE_NOTHROW(divide(-10, 3));
}
```

### CHECK 版本的异常断言

上述所有 `REQUIRE_THROWS*` 宏都有对应的 `CHECK_THROWS*` 版本，区别同样是致命/非致命：

```cpp
TEST_CASE("CHECK 版本的异常断言",
          "[exception]") {
    // 非致命——失败后继续
    CHECK_THROWS(divide(10, 0));
    CHECK_THROWS_AS(
        divide(10, 0),
        std::invalid_argument);
    CHECK_NOTHROW(divide(10, 2));
}
```

## 浮点数比较：Catch::Approx

浮点数（Floating Point）运算天然存在精度损失（IEEE 754 标准的固有限制），直接用 `==` 比较浮点数几乎必定失败：

```cpp
TEST_CASE("浮点数直接比较的陷阱", "[float]") {
    double result = 0.1 + 0.2;
    // REQUIRE(result == 0.3);  // ❌ 很可能失败！
    // 因为 0.1 + 0.2 的实际值是 0.30000000000000004
}
```

Catch2 提供了 `Catch::Approx` 类来进行**带容差（Tolerance）的浮点比较**：

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/catch_approx.hpp>  // Approx 类
#include <cmath>                    // std::sqrt

using Catch::Approx;  // 简化使用

TEST_CASE("Approx 浮点近似比较", "[float]") {
    // 基本用法——默认容差为 epsilon * 100
    REQUIRE(0.1 + 0.2 == Approx(0.3));

    // 数学函数结果
    REQUIRE(std::sqrt(4.0) == Approx(2.0));
    REQUIRE(std::sin(0.0) == Approx(0.0));

    // 自定义绝对容差（epsilon）
    REQUIRE(1.0 == Approx(1.001).epsilon(0.01));
    // 1.0 和 1.001 的差为 0.001，小于 1% 容差

    // 自定义绝对余量（margin）
    REQUIRE(100.0 == Approx(100.5).margin(1.0));
    // 差值 0.5 小于 margin 1.0

    // 零值比较——epsilon 基于相对值，零附近需要 margin
    REQUIRE(0.0 == Approx(0.0));
    REQUIRE(1e-15 == Approx(0.0).margin(1e-10));
}

TEST_CASE("Approx 用于容器", "[float]") {
    std::vector<double> actual = {1.0, 2.0, 3.0};
    std::vector<double> expected = {
        1.0000001, 2.0000001, 3.0000001
    };

    // 逐元素比较
    for (size_t i = 0; i < actual.size(); ++i) {
        REQUIRE(actual[i]
                == Approx(expected[i]).margin(1e-5));
    }
}
```

### Approx 的三种精度控制

| 方法 | 含义 | 适用场景 |
|------|------|---------|
| `.epsilon(e)` | 相对容差：\|a - b\| < e × max(\|a\|, \|b\|) | 一般浮点计算 |
| `.margin(m)` | 绝对容差：\|a - b\| < m | 零值附近、已知绝对误差范围 |
| `.scale(s)` | 改变 epsilon 的参考值 | 特殊场景（极少使用） |

## CAPTURE 和 INFO：调试信息输出

当测试失败时，Catch2 默认显示断言表达式和分解后的值。但在循环或复杂逻辑中，你可能需要更多上下文信息来定位问题。

### CAPTURE —— 捕获变量值

`CAPTURE(var1, var2, ...)` 在断言失败时自动显示指定变量的名称和值：

```cpp
#include <catch2/catch_test_macros.hpp>
#include <vector>
#include <string>

TEST_CASE("CAPTURE 捕获变量值", "[debug]") {
    std::vector<int> data = {2, 4, 6, 8, 10};

    for (size_t i = 0; i < data.size(); ++i) {
        CAPTURE(i, data[i]);  // 失败时显示 i 和 data[i] 的值
        REQUIRE(data[i] % 2 == 0);  // 验证每个元素是偶数
    }
}
```

如果 `data[2]` 是奇数导致测试失败，输出类似：

```
REQUIRE( data[i] % 2 == 0 )
with expansion:
  7 % 2 == 0
with messages:
  i := 2
  data[i] := 7
```

### INFO —— 自定义消息

`INFO(message)` 允许你添加自定义字符串消息，只在后续断言失败时才显示：

```cpp
TEST_CASE("INFO 自定义消息", "[debug]") {
    std::string config = "production";
    int maxRetries = 3;

    INFO("当前配置: " << config);
    INFO("最大重试次数: " << maxRetries);

    for (int attempt = 1;
         attempt <= maxRetries; ++attempt) {
        INFO("第 " << attempt << " 次尝试");
        // 假设 connectToServer() 返回 bool
        bool connected = (attempt == maxRetries);
        REQUIRE(connected);
        // 如果第 1 次尝试失败，输出会包含：
        //   当前配置: production
        //   最大重试次数: 3
        //   第 1 次尝试
    }
}
```

`INFO` 是**作用域感知**的——它只在定义它的作用域内有效，离开作用域后自动清除。这意味着循环中的 `INFO` 每次迭代都会更新。

### WARN 和 FAIL

```cpp
TEST_CASE("WARN 和 FAIL", "[debug]") {
    // WARN 输出警告信息（不影响测试结果）
    WARN("这是一条警告信息，测试仍然继续");

    // FAIL 直接标记当前测试失败并终止
    // FAIL("强制失败");  // 取消注释则测试失败

    // FAIL_CHECK 标记失败但继续执行
    // FAIL_CHECK("非致命失败");

    REQUIRE(true);  // 正常通过
}
```

## 常见错误及解决方案

### 错误 1：表达式中使用逻辑运算符

```cpp
// ❌ 错误：Catch2 无法分解 && 和 || 表达式
REQUIRE(a > 0 && a < 100);
// 编译警告：cannot decompose expression with '&&'

// ✅ 正确方案 1：拆分为多个断言
REQUIRE(a > 0);
REQUIRE(a < 100);

// ✅ 正确方案 2：加额外括号（但失败信息不够详细）
REQUIRE((a > 0 && a < 100));
```

### 错误 2：REQUIRE_THROWS_AS 中类型带引用

```cpp
// ❌ 错误：不要在 REQUIRE_THROWS_AS 中加 const& 
REQUIRE_THROWS_AS(
    func(),
    const std::runtime_error&  // 错误！
);

// ✅ 正确：只写类型名
REQUIRE_THROWS_AS(
    func(),
    std::runtime_error  // 正确
);
```

### 错误 3：浮点数直接比较

```cpp
// ❌ 错误：直接比较浮点数
REQUIRE(computeArea(r) == 3.14159);  // 几乎必定失败

// ✅ 正确：使用 Approx
REQUIRE(computeArea(r)
        == Catch::Approx(3.14159).epsilon(0.001));
```

## 动手实践

### 练习项目：温度转换器

创建一个温度转换工具库，并为它编写完整的单元测试。

#### 被测代码：`temperature.hpp`

```cpp
// temperature.hpp — 温度转换工具
#pragma once
#include <stdexcept>
#include <cmath>   // std::abs

// 绝对零度（开尔文 0K = -273.15°C）
constexpr double ABSOLUTE_ZERO_C = -273.15;
constexpr double ABSOLUTE_ZERO_F = -459.67;

// 摄氏度转华氏度：F = C × 9/5 + 32
double celsiusToFahrenheit(double celsius) {
    if (celsius < ABSOLUTE_ZERO_C) {
        throw std::invalid_argument(
            "温度低于绝对零度");
    }
    return celsius * 9.0 / 5.0 + 32.0;
}

// 华氏度转摄氏度：C = (F - 32) × 5/9
double fahrenheitToCelsius(double fahrenheit) {
    if (fahrenheit < ABSOLUTE_ZERO_F) {
        throw std::invalid_argument(
            "温度低于绝对零度");
    }
    return (fahrenheit - 32.0) * 5.0 / 9.0;
}

// 摄氏度转开尔文：K = C + 273.15
double celsiusToKelvin(double celsius) {
    if (celsius < ABSOLUTE_ZERO_C) {
        throw std::invalid_argument(
            "温度低于绝对零度");
    }
    return celsius + 273.15;
}

// 开尔文转摄氏度：C = K - 273.15
double kelvinToCelsius(double kelvin) {
    if (kelvin < 0.0) {
        throw std::invalid_argument(
            "开尔文温度不能为负");
    }
    return kelvin - 273.15;
}
```

#### 测试代码：`test_temperature.cpp`

```cpp
// test_temperature.cpp — 温度转换器的完整测试
#include <catch2/catch_test_macros.hpp>
#include <catch2/catch_approx.hpp>
#include "temperature.hpp"

using Catch::Approx;

TEST_CASE("摄氏度转华氏度", "[temperature]") {
    // 已知参考点
    REQUIRE(celsiusToFahrenheit(0.0)
            == Approx(32.0));       // 冰点
    REQUIRE(celsiusToFahrenheit(100.0)
            == Approx(212.0));      // 沸点
    REQUIRE(celsiusToFahrenheit(-40.0)
            == Approx(-40.0));      // 两种刻度的交叉点

    // 体温
    REQUIRE(celsiusToFahrenheit(37.0)
            == Approx(98.6));
}

TEST_CASE("华氏度转摄氏度", "[temperature]") {
    REQUIRE(fahrenheitToCelsius(32.0)
            == Approx(0.0));
    REQUIRE(fahrenheitToCelsius(212.0)
            == Approx(100.0));
    REQUIRE(fahrenheitToCelsius(-40.0)
            == Approx(-40.0));
}

TEST_CASE("摄氏度与开尔文互转",
          "[temperature]") {
    REQUIRE(celsiusToKelvin(0.0)
            == Approx(273.15));
    REQUIRE(celsiusToKelvin(-273.15)
            == Approx(0.0));     // 绝对零度
    REQUIRE(kelvinToCelsius(0.0)
            == Approx(-273.15));
    REQUIRE(kelvinToCelsius(373.15)
            == Approx(100.0));
}

TEST_CASE("往返转换一致性", "[temperature]") {
    // 摄氏→华氏→摄氏 应该得到原始值
    double original = 37.5;
    double roundTrip = fahrenheitToCelsius(
        celsiusToFahrenheit(original));
    REQUIRE(roundTrip == Approx(original));

    // 摄氏→开尔文→摄氏
    roundTrip = kelvinToCelsius(
        celsiusToKelvin(original));
    REQUIRE(roundTrip == Approx(original));
}

TEST_CASE("低于绝对零度抛出异常",
          "[temperature][exception]") {
    REQUIRE_THROWS_AS(
        celsiusToFahrenheit(-300.0),
        std::invalid_argument);
    REQUIRE_THROWS_AS(
        fahrenheitToCelsius(-500.0),
        std::invalid_argument);
    REQUIRE_THROWS_AS(
        celsiusToKelvin(-300.0),
        std::invalid_argument);
    REQUIRE_THROWS_AS(
        kelvinToCelsius(-1.0),
        std::invalid_argument);

    // 验证异常消息
    REQUIRE_THROWS_WITH(
        celsiusToFahrenheit(-300.0),
        "温度低于绝对零度");
}
```

#### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.14)
project(temperature_test LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Catch2 3 REQUIRED)
enable_testing()

add_executable(test_temperature
    test_temperature.cpp)
target_link_libraries(test_temperature
    PRIVATE Catch2::Catch2WithMain)

include(Catch2::Catch)
catch_discover_tests(test_temperature)
```

#### 运行

```bash
cmake -B build \
    -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake
cmake --build build
ctest --test-dir build --output-on-failure -V
```

预期输出中所有 5 个测试用例（包含 13+ 个断言）全部通过。

## 对照项目源码

KrKr2 项目中断言的实际使用可以在以下文件中观察到：

### PSB 文件测试中的断言

- **`tests/unit-tests/plugins/psbfile-dll.cpp`** 第 30-80 行 — 展示了 `REQUIRE` 与 `CHECK` 的混合使用。在验证 PSB 文件头信息时用 `REQUIRE`（因为头信息错误意味着文件没有正确加载，后续测试无意义），在验证具体属性值时用 `CHECK`（多个属性独立验证，想一次看到所有差异）

```cpp
// tests/unit-tests/plugins/psbfile-dll.cpp 第 55-70 行（简化）
// 来源：tests/unit-tests/plugins/psbfile-dll.cpp
TEST_CASE("psb file parsing", "[plugin]") {
    // REQUIRE 验证前置条件——文件加载成功
    auto psb = loadPSBFile(
        TEST_FILES_PATH "/emote/ezsave.pimg");
    REQUIRE(psb != nullptr);  // 必须成功加载

    SECTION("验证文件头") {
        REQUIRE(psb->getVersion() == 3);  // 前置
        // CHECK 验证独立属性
        CHECK(psb->getHeaderSize() > 0);
        CHECK(psb->getOffsetNames() > 0);
    }
}
```

### TJS2 字符串测试中的断言

- **`tests/unit-tests/core/tjs2/tjsString.cpp`** 第 1-22 行 — 展示了最简单的字符串转换测试。使用 `REQUIRE` 验证日文字符串的编码转换结果

```cpp
// tests/unit-tests/core/tjs2/tjsString.cpp（完整内容）
// 来源：tests/unit-tests/core/tjs2/tjsString.cpp 第 1-22 行
#include <catch2/catch_test_macros.hpp>
#include "tjs2/tjsString.h"

TEST_CASE("tjs string conversion", "[tjs2]") {
    // 验证日文字符串的 UTF-16 → std::string 转换
    tTJSString s(TJS_W("テスト"));
    std::string result = s.AsStdString();
    REQUIRE(!result.empty());  // 转换结果非空
}
```

### CAPTURE 在项目中的使用

- **`tests/unit-tests/plugins/psbfile-dll.cpp`** 第 120-150 行 — 在循环验证多个属性时使用 `CAPTURE` 记录当前迭代的索引和键名，方便定位失败位置

```cpp
// psbfile-dll.cpp 中 CAPTURE 的使用模式（简化）
for (size_t i = 0; i < properties.size(); ++i) {
    CAPTURE(i, properties[i].name);
    REQUIRE(properties[i].value != nullptr);
}
```

## 本节小结

- **`REQUIRE`（致命断言）** 在失败时立即终止当前测试用例，适合验证前置条件
- **`CHECK`（非致命断言）** 在失败后继续执行，适合验证多个独立条件
- **表达式分解** 是 Catch2 的核心特性——`REQUIRE(a == b)` 失败时自动显示 a 和 b 的实际值
- **异常断言** 四件套：`REQUIRE_THROWS`（任意异常）、`REQUIRE_THROWS_AS`（指定类型）、`REQUIRE_THROWS_WITH`（消息匹配）、`REQUIRE_NOTHROW`（无异常）
- **`Catch::Approx`** 解决浮点数比较问题，支持 `.epsilon()`（相对容差）和 `.margin()`（绝对容差）
- **`CAPTURE`** 和 **`INFO`** 在断言失败时提供额外上下文信息，是循环中调试的利器
- 所有 `REQUIRE_*` 宏都有对应的 `CHECK_*` 版本

## 练习题与答案

### 题目 1：REQUIRE 与 CHECK 的选择

以下代码中，哪些断言应该用 `REQUIRE`，哪些应该用 `CHECK`？请说明理由并改写代码。

```cpp
TEST_CASE("用户信息验证") {
    auto user = getUserById(42);
    ???(user != nullptr);          // A
    ???(user->name == "张三");     // B
    ???(user->age == 25);          // C
    ???(user->email.contains("@")); // D
}
```

<details>
<summary>查看答案</summary>

```cpp
TEST_CASE("用户信息验证") {
    auto user = getUserById(42);

    // A: 必须用 REQUIRE——如果 user 为空指针，
    // 后续的 user->name 等操作会导致空指针解引用崩溃
    REQUIRE(user != nullptr);

    // B, C, D: 用 CHECK——这三个断言相互独立，
    // name 错误不影响 age 的验证，
    // 用 CHECK 可以一次看到所有字段的差异
    CHECK(user->name == "张三");
    CHECK(user->age == 25);
    CHECK(user->email.contains("@"));
}
```

**理由**：A 是后续所有断言的前置条件（user 必须非空），所以用 `REQUIRE`。B、C、D 之间没有依赖关系，用 `CHECK` 可以一次看到所有失败的字段，而不是改一个跑一次。

</details>

### 题目 2：编写异常断言测试

为以下函数编写完整的异常测试（使用 `REQUIRE_THROWS_AS`、`REQUIRE_THROWS_WITH`、`REQUIRE_NOTHROW`）：

```cpp
double safeSqrt(double x) {
    if (x < 0) {
        throw std::domain_error(
            "负数没有实数平方根");
    }
    return std::sqrt(x);
}
```

<details>
<summary>查看答案</summary>

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/catch_approx.hpp>
#include <cmath>
#include <stdexcept>

double safeSqrt(double x) {
    if (x < 0) {
        throw std::domain_error(
            "负数没有实数平方根");
    }
    return std::sqrt(x);
}

TEST_CASE("safeSqrt 正常输入不抛出异常",
          "[math]") {
    REQUIRE_NOTHROW(safeSqrt(0.0));
    REQUIRE_NOTHROW(safeSqrt(1.0));
    REQUIRE_NOTHROW(safeSqrt(100.0));
    REQUIRE_NOTHROW(safeSqrt(0.0001));

    // 同时验证计算结果
    REQUIRE(safeSqrt(4.0)
            == Catch::Approx(2.0));
    REQUIRE(safeSqrt(9.0)
            == Catch::Approx(3.0));
}

TEST_CASE("safeSqrt 负数输入抛出 domain_error",
          "[math][exception]") {
    // 验证异常类型
    REQUIRE_THROWS_AS(
        safeSqrt(-1.0),
        std::domain_error);
    REQUIRE_THROWS_AS(
        safeSqrt(-100.0),
        std::domain_error);

    // 验证异常消息
    REQUIRE_THROWS_WITH(
        safeSqrt(-1.0),
        "负数没有实数平方根");
}

TEST_CASE("safeSqrt 边界值", "[math]") {
    // 零是合法输入
    REQUIRE_NOTHROW(safeSqrt(0.0));
    REQUIRE(safeSqrt(0.0)
            == Catch::Approx(0.0));

    // 极小正数也是合法输入
    REQUIRE_NOTHROW(safeSqrt(1e-300));

    // 极小负数也应该抛出异常
    REQUIRE_THROWS_AS(
        safeSqrt(-1e-300),
        std::domain_error);
}
```

</details>

### 题目 3：解释表达式分解

以下 Catch2 断言失败时，输出信息分别是什么样的？

```cpp
int x = 10;
std::string s = "Hello";
REQUIRE(x == 20);        // (A)
REQUIRE(s == "World");   // (B)
REQUIRE((x > 0 && x < 5));  // (C)
```

<details>
<summary>查看答案</summary>

**(A)** `REQUIRE(x == 20)` 的失败输出：
```
REQUIRE( x == 20 )
with expansion:
  10 == 20
```
Catch2 的表达式分解自动展示了 `x` 的实际值 `10`。

**(B)** `REQUIRE(s == "World")` 的失败输出：
```
REQUIRE( s == "World" )
with expansion:
  "Hello" == "World"
```
字符串值也被完整展示。

**(C)** `REQUIRE((x > 0 && x < 5))` 的失败输出：
```
REQUIRE( (x > 0 && x < 5) )
with expansion:
  false
```
由于加了额外括号，Catch2 无法分解复合表达式，只显示最终的布尔结果 `false`。如果需要看到具体值，应该拆分为两个断言：
```cpp
REQUIRE(x > 0);   // 通过
REQUIRE(x < 5);   // 失败，显示 "10 < 5"
```

</details>

## 下一步

[03 — SECTION 与参数化测试](./03-SECTION与参数化测试.md)：学习 Catch2 最强大的特性之一——SECTION 嵌套机制（在同一个 TEST_CASE 中组织多个独立子场景），以及 GENERATE 宏实现数据驱动的参数化测试和自定义 Matcher。

