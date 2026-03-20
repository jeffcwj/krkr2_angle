# SECTION 与参数化测试

> **所属模块：** M10-测试与质量保证
> **前置知识：** [02 — TEST_CASE 与断言](./02-TEST_CASE与断言.md)
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 SECTION 的执行模型——每个 SECTION 独立运行，共享 TEST_CASE 级别的初始化代码
2. 使用嵌套 SECTION 组织复杂的多层测试场景
3. 用 BDD 风格宏（`SCENARIO`/`GIVEN`/`WHEN`/`THEN`）编写行为驱动测试
4. 使用 `GENERATE` 宏和生成器实现数据驱动的参数化测试
5. 编写自定义 Matcher 实现复杂的匹配逻辑

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| SECTION | Section | Catch2 中将一个 TEST_CASE 拆分为多个独立子场景的机制，每个 SECTION 从 TEST_CASE 开头重新执行 |
| 执行模型 | Execution Model | SECTION 的运行方式——TEST_CASE 被执行多次，每次只进入一个叶子 SECTION |
| BDD | Behavior-Driven Development | 行为驱动开发，用自然语言（GIVEN/WHEN/THEN）描述测试场景的方法论 |
| 生成器 | Generator | Catch2 中用于产生测试数据序列的机制，配合 GENERATE 宏实现参数化测试 |
| 匹配器 | Matcher | 比简单的 `==` 更灵活的断言工具，支持字符串包含、范围比较、容器元素检查等复杂验证 |
| 参数化测试 | Parameterized Test | 用不同的输入数据自动重复运行同一段测试逻辑，避免为每组数据写重复的测试代码 |

## SECTION 机制详解

### 为什么需要 SECTION

在没有 SECTION 的传统测试框架（如 Google Test）中，如果你想测试同一个对象的多个方面，要么写多个 TEST_CASE（每个都重复初始化代码），要么用测试夹具类（代码冗长）。SECTION 解决了这个问题：**在同一个 TEST_CASE 内组织多个独立子场景，共享初始化代码但独立运行。**

```cpp
#include <catch2/catch_test_macros.hpp>
#include <vector>

TEST_CASE("vector 基本操作", "[container]") {
    // 初始化代码——每个 SECTION 运行前都会执行
    std::vector<int> v;
    REQUIRE(v.empty());  // 前置条件

    SECTION("push_back 添加元素") {
        v.push_back(1);
        v.push_back(2);
        REQUIRE(v.size() == 2);
        REQUIRE(v[0] == 1);
        REQUIRE(v[1] == 2);
    }

    SECTION("reserve 不改变 size") {
        v.reserve(100);
        REQUIRE(v.empty());       // size 不变
        REQUIRE(v.capacity() >= 100);  // 容量增加
    }

    SECTION("resize 改变 size") {
        v.resize(5);
        REQUIRE(v.size() == 5);
        // 默认值为 0
        for (int x : v) {
            REQUIRE(x == 0);
        }
    }
}
```

上面的代码看起来像是三个 SECTION 顺序执行，但实际上 Catch2 的执行方式完全不同。

### SECTION 的执行模型

**关键概念**：TEST_CASE 会被**执行多次**，每次只进入一个叶子 SECTION。每次执行都从 TEST_CASE 开头开始，重新运行初始化代码。

```
TEST_CASE "vector 基本操作" 的执行流程：

第 1 次执行：
  ├── std::vector<int> v;          ← 初始化
  ├── REQUIRE(v.empty());          ← 前置检查
  └── SECTION "push_back 添加元素"  ← 进入第 1 个 SECTION
      ├── v.push_back(1); ...
      └── REQUIRE(v.size() == 2);

第 2 次执行：
  ├── std::vector<int> v;          ← 重新初始化（全新的 v！）
  ├── REQUIRE(v.empty());          ← 前置检查
  └── SECTION "reserve 不改变 size" ← 进入第 2 个 SECTION
      ├── v.reserve(100);
      └── REQUIRE(v.empty());

第 3 次执行：
  ├── std::vector<int> v;          ← 又重新初始化
  ├── REQUIRE(v.empty());          ← 前置检查
  └── SECTION "resize 改变 size"   ← 进入第 3 个 SECTION
      ├── v.resize(5);
      └── REQUIRE(v.size() == 5);
```

这意味着：
1. **每个 SECTION 看到的是全新初始化的状态**——不受其他 SECTION 的影响
2. **SECTION 之间完全隔离**——第 1 个 SECTION 的 `push_back` 不会影响第 2 个 SECTION 的 `v`
3. **初始化代码天然被共享**——不需要像 Google Test 那样写夹具类

### 嵌套 SECTION

SECTION 可以嵌套——外层 SECTION 提供中间状态，内层 SECTION 测试不同分支：

```cpp
#include <catch2/catch_test_macros.hpp>
#include <map>
#include <string>

TEST_CASE("map 操作", "[container]") {
    std::map<std::string, int> scores;

    SECTION("空 map") {
        REQUIRE(scores.empty());
        REQUIRE(scores.size() == 0);
    }

    SECTION("插入元素后") {
        scores["张三"] = 90;
        scores["李四"] = 85;
        REQUIRE(scores.size() == 2);

        SECTION("查找已有元素") {
            auto it = scores.find("张三");
            REQUIRE(it != scores.end());
            REQUIRE(it->second == 90);
        }

        SECTION("查找不存在的元素") {
            auto it = scores.find("王五");
            REQUIRE(it == scores.end());
        }

        SECTION("删除元素") {
            scores.erase("张三");
            REQUIRE(scores.size() == 1);
            REQUIRE(scores.count("张三") == 0);
            REQUIRE(scores.count("李四") == 1);
        }
    }
}
```

嵌套 SECTION 的执行路径：

```
第 1 次：空 map
第 2 次：插入元素后 → 查找已有元素
第 3 次：插入元素后 → 查找不存在的元素
第 4 次：插入元素后 → 删除元素
```

注意"插入元素后"的代码（`scores["张三"] = 90; scores["李四"] = 85;`）在第 2、3、4 次执行中都会运行——它是三个内层 SECTION 的共享初始化。

## BDD 风格：SCENARIO / GIVEN / WHEN / THEN

BDD（Behavior-Driven Development，行为驱动开发）是一种用自然语言描述软件行为的方法论。Catch2 内置了 BDD 风格的宏，它们本质上是 `TEST_CASE` 和 `SECTION` 的别名，但让测试读起来更像用户故事（User Story）。

| BDD 宏 | 等价于 | 用途 |
|--------|--------|------|
| `SCENARIO(name, tags)` | `TEST_CASE("Scenario: " + name, tags)` | 定义一个测试场景 |
| `GIVEN(desc)` | `SECTION("Given: " + desc)` | 描述初始条件 |
| `WHEN(desc)` | `SECTION("When: " + desc)` | 描述触发动作 |
| `THEN(desc)` | `SECTION("Then: " + desc)` | 描述预期结果 |
| `AND_GIVEN(desc)` | `SECTION("And given: " + desc)` | 额外初始条件 |
| `AND_WHEN(desc)` | `SECTION("And when: " + desc)` | 额外触发动作 |
| `AND_THEN(desc)` | `SECTION("And then: " + desc)` | 额外预期结果 |

```cpp
#include <catch2/catch_test_macros.hpp>
#include <vector>

SCENARIO("向 vector 添加元素",
         "[container][bdd]") {

    GIVEN("一个空的 vector") {
        std::vector<int> v;
        REQUIRE(v.empty());

        WHEN("添加一个元素") {
            v.push_back(42);

            THEN("size 变为 1") {
                REQUIRE(v.size() == 1);
            }

            THEN("元素值正确") {
                REQUIRE(v[0] == 42);
            }
        }

        WHEN("添加三个元素") {
            v.push_back(1);
            v.push_back(2);
            v.push_back(3);

            THEN("size 变为 3") {
                REQUIRE(v.size() == 3);
            }

            THEN("可以通过 back 获取最后一个元素") {
                REQUIRE(v.back() == 3);
            }
        }
    }
}
```

CTest 运行时，每个叶子路径会显示完整的 BDD 描述：

```
Scenario: 向 vector 添加元素
  Given: 一个空的 vector
    When: 添加一个元素
      Then: size 变为 1 ............... Passed
Scenario: 向 vector 添加元素
  Given: 一个空的 vector
    When: 添加一个元素
      Then: 元素值正确 ............... Passed
Scenario: 向 vector 添加元素
  Given: 一个空的 vector
    When: 添加三个元素
      Then: size 变为 3 ............... Passed
...
```

> **何时使用 BDD 风格**：当测试涉及复杂的业务逻辑（如"用户下单 → 扣减库存 → 生成订单"）时，BDD 风格让测试本身成为可读的规格说明。对于纯算法或工具函数的测试，普通 `TEST_CASE` + `SECTION` 即可。

## GENERATE：参数化测试

### 基本用法

当你需要用多组数据测试同一段逻辑时，可以用 `GENERATE` 宏生成参数。Catch2 会为每个生成的值单独运行一次测试：

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/generators/catch_generators.hpp>
#include <cmath>  // std::abs

TEST_CASE("绝对值函数", "[math][generate]") {
    // GENERATE 产生多个值，每个值运行一次
    auto value = GENERATE(1, -1, 0, 42, -42);
    CAPTURE(value);  // 失败时显示当前值

    // std::abs(x) 应该 >= 0
    REQUIRE(std::abs(value) >= 0);

    // std::abs(x) 对正数和负数应该相同
    REQUIRE(std::abs(value) == std::abs(-value));
}
```

这等价于手动写 5 个类似的测试，但代码量只有一份。

### 内置生成器

Catch2 提供了多种内置生成器（都在 `<catch2/generators/catch_generators.hpp>` 头文件中）：

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/generators/catch_generators.hpp>
#include <catch2/generators/catch_generators_range.hpp>
#include <catch2/generators/catch_generators_adapters.hpp>

TEST_CASE("内置生成器", "[generate]") {
    SECTION("values — 列举值") {
        auto x = GENERATE(values({1, 2, 3, 4, 5}));
        REQUIRE(x > 0);
        REQUIRE(x <= 5);
    }

    SECTION("range — 范围") {
        // range(start, end) 生成 [start, end) 的整数
        auto x = GENERATE(range(1, 11));  // 1 到 10
        REQUIRE(x >= 1);
        REQUIRE(x <= 10);
    }

    SECTION("table — 表格数据") {
        // table 生成 tuple，适合多参数场景
        auto [input, expected] = GENERATE(table<int, int>({
            {0, 1},     // 0! = 1
            {1, 1},     // 1! = 1
            {5, 120},   // 5! = 120
            {10, 3628800}  // 10! = 3628800
        }));
        CAPTURE(input, expected);
        REQUIRE(factorial(input) == expected);
    }
}
```

### 组合生成器

```cpp
TEST_CASE("组合生成器", "[generate]") {
    // 两个 GENERATE 会产生笛卡尔积
    auto a = GENERATE(1, 2, 3);
    auto b = GENERATE(10, 20);
    // 实际运行 6 次：(1,10) (1,20) (2,10) (2,20) (3,10) (3,20)

    CAPTURE(a, b);
    REQUIRE(a * b > 0);  // 全部为正数
    REQUIRE(a * b <= 60); // 最大 3*20=60
}
```

### filter 和 map 适配器

```cpp
#include <catch2/generators/catch_generators_adapters.hpp>

TEST_CASE("生成器适配器", "[generate]") {
    // filter：只保留满足条件的值
    auto even = GENERATE(
        filter(
            [](int x) { return x % 2 == 0; },
            range(1, 21)  // 1-20 中的偶数
        )
    );
    REQUIRE(even % 2 == 0);

    // take：只取前 N 个值
    auto first3 = GENERATE(
        take(3, range(100, 200))
    );
    // 只运行 3 次：100, 101, 102
    REQUIRE(first3 >= 100);
}
```

## 自定义 Matcher

Matcher（匹配器）是比简单 `==` 比较更灵活的断言工具。Catch2 内置了字符串、浮点数、容器等常用 Matcher，你也可以自定义。

### 内置 Matcher

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/matchers/catch_matchers_string.hpp>
#include <catch2/matchers/catch_matchers_vector.hpp>
#include <catch2/matchers/catch_matchers_floating_point.hpp>

using namespace Catch::Matchers;

TEST_CASE("字符串 Matcher", "[matcher]") {
    std::string msg = "Error: file not found";

    // 包含子串
    REQUIRE_THAT(msg,
        ContainsSubstring("not found"));

    // 以指定前缀开头
    REQUIRE_THAT(msg, StartsWith("Error"));

    // 以指定后缀结尾
    REQUIRE_THAT(msg, EndsWith("found"));

    // 正则匹配
    REQUIRE_THAT(msg,
        Matches("Error:.*not found"));

    // 组合 Matcher（AND 逻辑）
    REQUIRE_THAT(msg,
        ContainsSubstring("Error")
        && ContainsSubstring("file"));

    // 组合 Matcher（OR 逻辑）
    REQUIRE_THAT(msg,
        ContainsSubstring("warning")
        || ContainsSubstring("Error"));

    // 取反
    REQUIRE_THAT(msg,
        !ContainsSubstring("success"));
}

TEST_CASE("浮点 Matcher", "[matcher]") {
    double result = 3.14159;

    // 绝对误差范围
    REQUIRE_THAT(result,
        WithinAbs(3.14, 0.01));

    // 相对误差范围
    REQUIRE_THAT(result,
        WithinRel(3.14159, 1e-5));
}

TEST_CASE("容器 Matcher", "[matcher]") {
    std::vector<int> v = {1, 2, 3, 4, 5};

    // 包含某个元素
    REQUIRE_THAT(v,
        VectorContains(3));

    // 包含另一个容器的所有元素
    REQUIRE_THAT(v,
        Contains(std::vector<int>{2, 4}));

    // 等于某个容器（顺序敏感）
    REQUIRE_THAT(v,
        Equals(std::vector<int>{1, 2, 3, 4, 5}));
}
```

### 编写自定义 Matcher

当内置 Matcher 无法满足需求时，你可以继承 `Catch::Matchers::MatcherBase` 编写自己的 Matcher：

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/matchers/catch_matchers_templated.hpp>
#include <string>
#include <algorithm>

// 自定义 Matcher：检查字符串是否全部为大写
struct IsAllUpperCase
    : Catch::Matchers::MatcherGenericBase {

    // match 方法：返回 true 表示匹配成功
    bool match(const std::string& s) const {
        return std::all_of(
            s.begin(), s.end(),
            [](unsigned char c) {
                // 非字母字符跳过，字母必须大写
                return !std::isalpha(c)
                    || std::isupper(c);
            }
        );
    }

    // describe 方法：描述匹配条件（用于失败消息）
    std::string describe() const override {
        return "全部为大写字母";
    }
};

// 工厂函数（让调用者不需要手动构造对象）
auto isAllUpperCase() {
    return IsAllUpperCase{};
}

TEST_CASE("自定义 Matcher 使用示例",
          "[matcher][custom]") {
    REQUIRE_THAT("HELLO WORLD", isAllUpperCase());
    REQUIRE_THAT("ABC 123", isAllUpperCase());
    REQUIRE_THAT(
        std::string("hello"),
        !isAllUpperCase()  // 取反：不是全大写
    );
}
```

## 常见错误及解决方案

### 错误 1：SECTION 之间共享状态

```cpp
// ❌ 错误理解：以为 SECTION 顺序执行
TEST_CASE("SECTION 误区") {
    int count = 0;

    SECTION("第一个") {
        count = 10;
        REQUIRE(count == 10);
    }

    SECTION("第二个") {
        // ❌ 错误：以为 count 是 10（第一个 SECTION 设的）
        // 实际上 count 是 0（每次从 TEST_CASE 开头重新执行）
        REQUIRE(count == 0);  // ✅ 这才是正确的
    }
}
```

### 错误 2：GENERATE 放在 SECTION 内部

```cpp
// ⚠️ 注意：GENERATE 可以放在 SECTION 内部，
// 但要理解它与 SECTION 的交互
TEST_CASE("GENERATE 在 SECTION 内") {
    SECTION("偶数") {
        auto x = GENERATE(2, 4, 6);
        REQUIRE(x % 2 == 0);
    }
    SECTION("奇数") {
        auto x = GENERATE(1, 3, 5);
        REQUIRE(x % 2 == 1);
    }
    // 总共运行 6 次：3 次偶数 + 3 次奇数
}
```

### 错误 3：REQUIRE_THAT 忘记 using namespace

```cpp
// ❌ 错误：没有引入命名空间
REQUIRE_THAT(s, ContainsSubstring("hello"));
// 编译错误：ContainsSubstring 未定义

// ✅ 正确：引入 Catch::Matchers 命名空间
using namespace Catch::Matchers;
REQUIRE_THAT(s, ContainsSubstring("hello"));

// 或者使用完整限定名
REQUIRE_THAT(s,
    Catch::Matchers::ContainsSubstring("hello"));
```

## 动手实践

### 练习项目：简易银行账户

用 SECTION 和 BDD 风格测试一个银行账户类。

#### 被测代码：`bank_account.hpp`

```cpp
// bank_account.hpp — 简易银行账户
#pragma once
#include <stdexcept>
#include <string>
#include <vector>

struct Transaction {
    std::string description;  // 交易描述
    double amount;            // 金额（正=存入，负=取出）
};

class BankAccount {
    double balance_;                      // 余额
    std::vector<Transaction> history_;    // 交易历史

public:
    explicit BankAccount(double initial = 0.0)
        : balance_(initial) {
        if (initial < 0) {
            throw std::invalid_argument(
                "初始余额不能为负");
        }
    }

    double balance() const { return balance_; }

    // 存款
    void deposit(double amount) {
        if (amount <= 0) {
            throw std::invalid_argument(
                "存款金额必须为正数");
        }
        balance_ += amount;
        history_.push_back({"存款", amount});
    }

    // 取款
    void withdraw(double amount) {
        if (amount <= 0) {
            throw std::invalid_argument(
                "取款金额必须为正数");
        }
        if (amount > balance_) {
            throw std::runtime_error("余额不足");
        }
        balance_ -= amount;
        history_.push_back({"取款", -amount});
    }

    // 获取交易历史
    const std::vector<Transaction>&
    transactions() const {
        return history_;
    }
};
```

#### 测试代码：`test_bank.cpp`

```cpp
// test_bank.cpp — 银行账户的 BDD 风格测试
#include <catch2/catch_test_macros.hpp>
#include <catch2/catch_approx.hpp>
#include <catch2/generators/catch_generators.hpp>
#include "bank_account.hpp"

using Catch::Approx;

SCENARIO("银行账户基本操作",
         "[bank][bdd]") {

    GIVEN("一个初始余额为 1000 的账户") {
        BankAccount account(1000.0);
        REQUIRE(account.balance()
                == Approx(1000.0));

        WHEN("存入 500") {
            account.deposit(500.0);

            THEN("余额变为 1500") {
                REQUIRE(account.balance()
                        == Approx(1500.0));
            }

            THEN("交易历史记录了存款") {
                auto& txns = account.transactions();
                REQUIRE(txns.size() == 1);
                REQUIRE(txns[0].description == "存款");
                REQUIRE(txns[0].amount
                        == Approx(500.0));
            }
        }

        WHEN("取出 300") {
            account.withdraw(300.0);

            THEN("余额变为 700") {
                REQUIRE(account.balance()
                        == Approx(700.0));
            }
        }

        WHEN("尝试取出超过余额的金额") {
            THEN("抛出 runtime_error") {
                REQUIRE_THROWS_AS(
                    account.withdraw(2000.0),
                    std::runtime_error);
                REQUIRE_THROWS_WITH(
                    account.withdraw(2000.0),
                    "余额不足");
            }

            THEN("余额不变") {
                try {
                    account.withdraw(2000.0);
                } catch (...) {}
                REQUIRE(account.balance()
                        == Approx(1000.0));
            }
        }
    }

    GIVEN("创建负余额账户") {
        THEN("抛出 invalid_argument") {
            REQUIRE_THROWS_AS(
                BankAccount(-100.0),
                std::invalid_argument);
        }
    }
}

TEST_CASE("参数化测试：多金额存款",
          "[bank][generate]") {
    auto amount = GENERATE(100.0, 500.0,
                           1000.0, 9999.99);
    CAPTURE(amount);

    BankAccount account(0.0);
    account.deposit(amount);
    REQUIRE(account.balance()
            == Approx(amount));
}
```

#### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.14)
project(bank_test LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Catch2 3 REQUIRED)
enable_testing()

add_executable(test_bank test_bank.cpp)
target_link_libraries(test_bank
    PRIVATE Catch2::Catch2WithMain)

include(Catch2::Catch)
catch_discover_tests(test_bank)
```

## 对照项目源码

KrKr2 项目的测试代码大量使用了本节介绍的 SECTION 和嵌套 SECTION 模式。下面逐个分析关键文件。

### 1. `tests/unit-tests/plugins/psbfile-dll.cpp` — 嵌套 SECTION 经典示例

这是项目中最丰富的测试文件（215 行），完整展示了 SECTION 的分层组织能力。

**文件结构**（第 12-206 行）：

```cpp
// tests/unit-tests/plugins/psbfile-dll.cpp 第 12-17 行
// TEST_CASE 级初始化：加载 PSB 文件并验证基本属性
TEST_CASE("read psbfile ezsave.pimg") {
    PSB::PSBFile f;
    REQUIRE(f.loadPSBFile(
        TEST_FILES_PATH "/emote/ezsave.pimg"));
    const PSB::PSBHeader &header = f.getPSBHeader();
    REQUIRE(f.getType() == PSB::PSBType::Pimg);
    CAPTURE(header.version, f.getType());
    // CAPTURE 记录关键上下文，失败时自动显示

    auto objs = f.getObjects();

    SECTION("check width height") {
        // 第一层 SECTION：验证图片尺寸
        // ... 验证 width == 1280, height == 720
    }

    SECTION("get layers") {
        // 第一层 SECTION：获取图层列表
        auto layers = ...;
        REQUIRE(layers->size() == 32);

        SECTION("check layer properties") {
            // 第二层嵌套 SECTION：逐图层验证属性
            // 在 for 循环中批量检查 32 个图层
            // 的 height, width, name, visible 等
        }
    }

    SECTION("collect resources") {
        // 第一层 SECTION：验证资源收集
        auto resMetadata =
            f.getTypeHandler()->collectResources(f, true);
        REQUIRE(!resMetadata.empty());
    }
}
```

**执行路径分析**：

```
psbfile-dll.cpp TEST_CASE "read psbfile ezsave.pimg" 的执行：

第 1 次执行：
  ├── 加载 ezsave.pimg + 验证类型     ← 共享初始化
  └── SECTION "check width height"    ← 验证尺寸

第 2 次执行：
  ├── 加载 ezsave.pimg + 验证类型     ← 重新初始化
  └── SECTION "get layers"
      └── SECTION "check layer properties"  ← 嵌套！
          └── for 循环验证 32 个图层属性

第 3 次执行：
  ├── 加载 ezsave.pimg + 验证类型     ← 又重新初始化
  └── SECTION "collect resources"     ← 验证资源收集
```

注意第 2 次执行中，"get layers" SECTION 内的 `REQUIRE(layers->size() == 32)` 是嵌套 SECTION "check layer properties" 的**共享前置检查**——如果图层数不是 32，内层 SECTION 没有运行的意义。这正是嵌套 SECTION 的设计哲学：**外层建立中间状态，内层在此基础上深入验证。**

**关键技法提取**：

| 技法 | 源码位置 | 说明 |
|------|----------|------|
| `CAPTURE` 记录上下文 | 第 17 行 | `CAPTURE(header.version, f.getType())` — 失败时自动打印版本和类型 |
| 嵌套 SECTION | 第 35-195 行 | "get layers" 内嵌 "check layer properties"，共享 `layers` 变量 |
| 循环内批量断言 | 第 120-194 行 | for 循环中逐元素 REQUIRE，失败时 CAPTURE 能定位到具体索引 |
| 多个平级 SECTION | 第 21/30/198 行 | "check width height"、"get layers"、"collect resources" 互不干扰 |

### 2. `tests/unit-tests/core/tjs2/tjs.cpp` — SECTION 作为独立测试场景

```cpp
// tests/unit-tests/core/tjs2/tjs.cpp 第 8-73 行
TEST_CASE("tjs2 test") {
    // TEST_CASE 级初始化：创建 TJS2 引擎实例
    auto *engine = new tTJS();
    TVPRegExpClass::register_me(engine);
    // 注册正则表达式类（引擎需要它才能运行脚本）

    SECTION("array") {
        // 加载并执行数组测试脚本
        auto stream = TVPCreateStream(
            TEST_FILES_PATH "/tjs2/array.tjs");
        // ... 编译并执行脚本
    }

    SECTION("class") {
        // 加载并执行类测试脚本
        auto stream = TVPCreateStream(
            TEST_FILES_PATH "/tjs2/class.tjs");
        // ...
    }

    // 更多 SECTION: regexp, string, operator...
    // 每个 SECTION 测试一种 TJS2 语言特性

    engine->Shutdown();  // 共享清理代码
    engine->Release();
}
```

这里每个 SECTION 代表一种 TJS2 语言特性的独立测试。**共享初始化**是引擎实例的创建和正则类注册，**共享清理**是引擎的关闭和释放。由于 SECTION 的执行模型（每次从 TEST_CASE 开头重新执行），每个脚本测试都在全新的引擎实例上运行，完全隔离。

### 相关文件索引

| 文件路径 | 行号范围 | 本节相关内容 |
|----------|----------|-------------|
| `tests/unit-tests/plugins/psbfile-dll.cpp` | 第 12-206 行 | 嵌套 SECTION、CAPTURE、循环断言 |
| `tests/unit-tests/core/tjs2/tjs.cpp` | 第 8-73 行 | 平级 SECTION 组织独立测试场景 |
| `tests/unit-tests/core/tjs2/tjsString.cpp` | 第 6-22 行 | 最简 SECTION 示例（单个字符串转换测试） |
| `tests/unit-tests/core/movie/ffmpeg.cpp` | 第 1-8 行 | 无 SECTION 的最简 TEST_CASE（对比参照） |

## 本节小结

- **SECTION 执行模型**：TEST_CASE 被执行多次，每次只进入一个叶子 SECTION。每次执行都从 TEST_CASE 开头重新运行，保证各 SECTION 之间**完全隔离**
- **嵌套 SECTION**：外层 SECTION 建立中间状态，内层 SECTION 在此基础上测试不同分支。执行路径是所有叶子节点的组合
- **BDD 风格**：`SCENARIO`/`GIVEN`/`WHEN`/`THEN` 是 `TEST_CASE`/`SECTION` 的别名，让测试读起来像用户故事，适合业务逻辑复杂的场景
- **GENERATE 参数化**：用 `GENERATE` 宏产生多组测试数据，Catch2 为每个值单独运行一次测试。多个 GENERATE 产生笛卡尔积
- **内置生成器**：`values`（列举）、`range`（范围）、`table`（表格/多参数）、`filter`（过滤）、`take`（取前 N 个）
- **Matcher 匹配器**：比简单 `==` 更灵活的断言工具，支持字符串包含/前缀/后缀/正则、浮点误差、容器元素检查，可以用 `&&`/`||`/`!` 组合
- **自定义 Matcher**：继承 `MatcherGenericBase`，实现 `match()` 和 `describe()` 方法
- **项目实践**：KrKr2 的 `psbfile-dll.cpp` 展示了嵌套 SECTION + CAPTURE + 循环断言的完整模式；`tjs.cpp` 展示了 SECTION 作为独立测试场景的组织方式

## 练习题与答案

### 题目 1：SECTION 执行次数分析

以下代码中，TEST_CASE 总共会被执行几次？每次执行分别进入哪些 SECTION？

```cpp
TEST_CASE("执行次数分析") {
    int x = 0;

    SECTION("A") {
        x = 1;
        SECTION("A1") {
            REQUIRE(x == 1);
        }
        SECTION("A2") {
            REQUIRE(x == 1);
        }
    }

    SECTION("B") {
        x = 2;
        REQUIRE(x == 2);
    }
}
```

<details>
<summary>查看答案</summary>

TEST_CASE 总共会被执行 **3 次**：

1. **第 1 次**：进入 SECTION "A" → 进入 SECTION "A1"
   - `x` 被设为 1，然后 `REQUIRE(x == 1)` 通过
2. **第 2 次**：进入 SECTION "A" → 进入 SECTION "A2"
   - `x` 被重新设为 1（因为从 TEST_CASE 开头重新执行），然后 `REQUIRE(x == 1)` 通过
3. **第 3 次**：进入 SECTION "B"
   - `x` 被设为 2，`REQUIRE(x == 2)` 通过

关键理解：
- 每次执行都从 `int x = 0;` 重新开始
- "A" 有 2 个子 SECTION，所以 "A" 被执行 2 次（每次进入不同的子 SECTION）
- 叶子 SECTION 总数 = 执行次数 = A1 + A2 + B = 3 次

</details>

### 题目 2：用 BDD 风格重写测试

将以下普通测试改写为 BDD 风格（使用 `SCENARIO`/`GIVEN`/`WHEN`/`THEN`）：

```cpp
TEST_CASE("stack 操作") {
    std::stack<int> s;

    SECTION("push 一个元素后 top 是该元素") {
        s.push(42);
        REQUIRE(s.top() == 42);
        REQUIRE(s.size() == 1);
    }

    SECTION("push 再 pop 后为空") {
        s.push(42);
        s.pop();
        REQUIRE(s.empty());
    }
}
```

<details>
<summary>查看答案</summary>

```cpp
#include <catch2/catch_test_macros.hpp>
#include <stack>

SCENARIO("stack 基本操作", "[stack][bdd]") {

    GIVEN("一个空的 stack") {
        std::stack<int> s;
        REQUIRE(s.empty());

        WHEN("push 一个元素 42") {
            s.push(42);

            THEN("top 返回 42 且 size 为 1") {
                REQUIRE(s.top() == 42);
                REQUIRE(s.size() == 1);
            }

            THEN("再 pop 后 stack 为空") {
                s.pop();
                REQUIRE(s.empty());
            }
        }
    }
}
```

关键改写要点：
1. `TEST_CASE` → `SCENARIO`，名称用场景描述
2. 初始化代码放在 `GIVEN` 中，描述初始条件
3. 触发动作放在 `WHEN` 中（`s.push(42)` 是两个测试共享的动作）
4. 断言放在 `THEN` 中，描述预期结果
5. BDD 风格让测试输出自动显示层级缩进的场景描述

</details>

### 题目 3：用 GENERATE 实现参数化测试

编写一个参数化测试，验证 `std::string::substr` 函数对以下输入的正确性：

| 原字符串 | 起始位置 | 长度 | 期望结果 |
|----------|----------|------|----------|
| `"Hello World"` | 0 | 5 | `"Hello"` |
| `"Hello World"` | 6 | 5 | `"World"` |
| `"Hello World"` | 0 | 0 | `""` |
| `"abcdef"` | 2 | 3 | `"cde"` |

要求使用 `GENERATE` + `table` 生成器，并使用 `CAPTURE` 记录测试上下文。

<details>
<summary>查看答案</summary>

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/generators/catch_generators.hpp>
#include <string>

TEST_CASE("std::string::substr 参数化测试",
          "[string][generate]") {

    // 使用 table 生成器提供多组测试数据
    // 元组类型：(原字符串, 起始位置, 长度, 期望结果)
    auto [input, pos, len, expected] = GENERATE(
        table<std::string, size_t, size_t,
              std::string>({
            {"Hello World", 0, 5, "Hello"},
            {"Hello World", 6, 5, "World"},
            {"Hello World", 0, 0, ""},
            {"abcdef",      2, 3, "cde"},
        })
    );

    // CAPTURE 记录当前测试数据——失败时自动显示
    CAPTURE(input, pos, len, expected);

    // 执行 substr 并验证结果
    std::string result = input.substr(pos, len);
    REQUIRE(result == expected);
}
```

这个测试会运行 4 次（每组数据一次）。如果任何一组失败，`CAPTURE` 会在错误信息中显示当前的 `input`、`pos`、`len`、`expected` 值，方便定位问题。

</details>

## 下一步

下一章我们将深入 KrKr2 项目的实际测试架构，学习测试目录的组织结构和 CMake 配置：

→ [01 — 测试目录结构与 CMake 配置](../02-项目测试架构/01-测试目录结构与CMake配置.md)

