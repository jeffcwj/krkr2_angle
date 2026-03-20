# 自定义 main 与 spdlog 集成

> **所属模块：** M10-测试与质量保证
> **前置知识：** [测试目录结构与 CMake 配置](./01-测试目录结构与CMake配置.md)、[Catch2 简介与安装](../01-Catch2测试框架/01-Catch2简介与安装.md)
> **预计阅读时间：** 20 分钟

## 本节目标

读完本节后，你将能够：

1. 解释为什么 KrKr2 不使用 Catch2 提供的默认 `main()` 函数
2. 逐行理解项目中三个 `main.cpp` 文件的每一行代码
3. 掌握 spdlog 全局 logger 注册机制（Registry）的工作原理
4. 独立编写符合项目规范的测试入口文件
5. 诊断因 logger 缺失导致的运行时崩溃问题

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| spdlog | spdlog | C++ 高性能日志库，KrKr2 全项目使用它输出调试信息 |
| Logger | Logger (日志记录器) | spdlog 中的日志输出通道，每个 logger 有独立的名字和输出目标 |
| 全局注册表 | Global Registry | spdlog 内部的一个全局字典，存储所有已创建的 logger，可通过名字查找 |
| Sink | Sink (输出槽) | 日志的输出目标——可以是控制台、文件、网络等 |
| Catch::Session | Catch::Session | Catch2 的测试运行会话对象，调用 `run()` 方法执行所有测试 |

---

## 为什么需要自定义 main()

### Catch2 默认入口的局限

Catch2 v3 提供了两种使用方式：

| 方式 | CMake 目标 | 特点 |
|------|-----------|------|
| 自定义 main | `Catch2::Catch2` | 你自己写 `main()` 函数，在里面手动调用 `Catch::Session().run()` |
| 默认 main | `Catch2::Catch2WithMain` | Catch2 提供现成的 `main()` 函数，你什么都不用写 |

对于大多数简单项目，`Catch2WithMain` 完全够用。它的 `main()` 函数大致等价于：

```cpp
// Catch2WithMain 内置的 main() 函数（简化版）
int main(int argc, char* argv[]) {
    return Catch::Session().run(argc, argv);
    // 就这么简单——直接运行测试
}
```

但 KrKr2 不能用这个默认入口，原因只有一个：**引擎代码在运行时会调用 `spdlog::get()` 获取日志记录器，如果 logger 没有提前创建，`spdlog::get()` 返回空指针，后续调用空指针的方法会导致段错误（Segmentation Fault，即程序试图访问无效内存地址而被操作系统终止）。**

### 崩溃场景复现

下面用一个完整的代码示例演示这个崩溃是怎么发生的：

```cpp
// 演示：spdlog::get() 返回空指针时的崩溃
#include <spdlog/spdlog.h>
#include <iostream>

int main() {
    // 没有创建任何 logger，直接 get
    auto logger = spdlog::get("core");

    // logger 此时是 nullptr（空指针）
    if (logger == nullptr) {
        std::cout << "logger 是空指针！" << std::endl;
    }

    // 如果引擎代码不检查空指针而直接调用：
    // logger->info("engine started");
    // ↑ 这行会触发段错误（Segmentation Fault），程序崩溃
    // 因为你在对一个空指针调用成员方法

    return 0;
}
```

输出：

```
logger 是空指针！
```

KrKr2 引擎的核心代码中散布着大量 `spdlog::get("core")->info(...)` 这样的调用。这些代码在正常运行（引擎启动时会创建 logger）没问题，但在测试环境下如果不提前初始化 logger 就会崩溃。

### 解决方案：在测试入口中预创建 Logger

自定义 `main()` 函数的核心任务就是：**在 Catch2 测试运行之前，先创建引擎代码所需的所有 spdlog logger。** 这样当测试过程中引擎代码调用 `spdlog::get("core")` 时，就能正常拿到已注册的 logger 对象。

执行时序如下：

```
程序启动
    │
    ▼
main() 函数执行
    │
    ├── ① spdlog::stdout_color_mt("core")   ← 创建 "core" logger
    ├── ② spdlog::stdout_color_mt("tjs2")   ← 创建 "tjs2" logger
    ├── ③ spdlog::stdout_color_mt("plugin") ← 创建 "plugin" logger（仅插件测试）
    │
    ▼
Catch::Session().run(argc, argv) 开始执行测试
    │
    ├── TEST_CASE "xxx" 执行
    │   └── 引擎代码调用 spdlog::get("core")->info(...)  ← 正常工作
    ├── TEST_CASE "yyy" 执行
    │   └── 引擎代码调用 spdlog::get("tjs2")->debug(...) ← 正常工作
    │
    ▼
所有测试完成，返回结果码
```

---

## 三个 main.cpp 逐行对比

KrKr2 项目有三个 `main.cpp` 文件，分布在不同的测试目录中。它们的代码几乎完全一样，唯一区别在于创建的 logger 数量不同。

### plugins/main.cpp（完整 18 行）

来源：`tests/unit-tests/plugins/main.cpp` 第 1-18 行

```cpp
//
// Created by lidong on 25-6-21.
//

#include <catch2/catch_session.hpp>       // 提供 Catch::Session 类

#include <spdlog/sinks/stdout_color_sinks.h>  // 提供 stdout_color_mt 函数

int main(int argc, char *argv[]) {

    // 创建三个全局 logger：core、tjs2、plugin
    static auto core_logger = spdlog::stdout_color_mt("core");
    static auto tjs2_logger = spdlog::stdout_color_mt("tjs2");
    static auto plugin_logger = spdlog::stdout_color_mt("plugin");

    // 运行 Catch2 测试并获取结果码
    int result = Catch::Session().run(argc, argv);

    return result;
}
```

### core/movie/main.cpp 和 core/tjs2/main.cpp（完整 17 行）

来源：`tests/unit-tests/core/movie/main.cpp` 第 1-17 行（tjs2 版本完全相同）

```cpp
//
// Created by lidong on 25-6-21.
//

#include <catch2/catch_session.hpp>

#include <spdlog/sinks/stdout_color_sinks.h>

int main(int argc, char *argv[]) {

    // 只创建两个 logger：core、tjs2（不需要 plugin logger）
    static auto core_logger = spdlog::stdout_color_mt("core");
    static auto tjs2_logger = spdlog::stdout_color_mt("tjs2");

    int result = Catch::Session().run(argc, argv);

    return result;
}
```

### 差异对比表

| 文件 | 行数 | "core" | "tjs2" | "plugin" | 原因 |
|------|------|--------|--------|----------|------|
| `plugins/main.cpp` | 18 | ✅ | ✅ | ✅ | 插件代码引用所有三个 logger |
| `core/movie/main.cpp` | 17 | ✅ | ✅ | ❌ | 视频模块不涉及插件代码 |
| `core/tjs2/main.cpp` | 17 | ✅ | ✅ | ❌ | TJS2 引擎不涉及插件代码 |

为什么 `plugins/main.cpp` 多创建了一个 `"plugin"` logger？因为插件代码（如 `cpp/plugins/psbfile/` 中的 PSB 解析器）在初始化时会调用 `spdlog::get("plugin")` 来记录日志。如果不创建这个 logger，运行插件相关的测试就会崩溃。而核心模块的测试不会触及插件代码路径，所以不需要 `"plugin"` logger。

---

## spdlog 全局注册机制深入理解

理解 `main.cpp` 中的代码需要知道 spdlog 的**全局注册表**（Global Registry）是怎么工作的。

### 注册表是什么

spdlog 内部维护了一个全局的"字典"数据结构（`std::unordered_map`），键是 logger 名字（字符串），值是 logger 对象（`std::shared_ptr<spdlog::logger>`）。所有通过工厂函数（如 `stdout_color_mt`）创建的 logger 都会自动注册到这个字典中。

```
spdlog 全局注册表（Global Registry）
┌──────────────────────────────────────────────┐
│  "core"   →  shared_ptr<logger>  (控制台彩色) │
│  "tjs2"   →  shared_ptr<logger>  (控制台彩色) │
│  "plugin" →  shared_ptr<logger>  (控制台彩色) │
└──────────────────────────────────────────────┘
        ↑ 注册                    ↑ 查找
  spdlog::stdout_color_mt()  spdlog::get()
```

### 关键 API 详解

**`spdlog::stdout_color_mt("core")`——创建并注册**

```cpp
// 函数签名（简化）：
std::shared_ptr<spdlog::logger> stdout_color_mt(const std::string& logger_name);
```

这个函数做了三件事：
1. 创建一个新的 logger 对象
2. 为它配置一个"彩色标准输出"的 Sink（输出槽——即日志输出到控制台，并用 ANSI 颜色代码区分不同级别）
3. 把 logger 注册到全局注册表，键名为传入的字符串（如 `"core"`）

函数名中的后缀含义：`stdout` = 标准输出，`color` = 彩色，`mt` = 多线程安全（Multi-Threaded，内部使用互斥锁保护并发写入）。如果不需要多线程安全，可以用 `stdout_color_st`（st = Single-Threaded，单线程版本，性能更好但不安全）。

**`spdlog::get("core")`——查找已注册的 logger**

```cpp
// 函数签名（简化）：
std::shared_ptr<spdlog::logger> get(const std::string& logger_name);
```

从全局注册表中查找指定名字的 logger。如果找到则返回对应的 `shared_ptr`，如果找不到则返回 `nullptr`（空指针）。

这就是崩溃的根源：引擎代码假设 logger 一定存在，直接对返回值调用方法：

```cpp
// 引擎代码中的典型调用（不检查空指针）
spdlog::get("core")->info("Engine version: {}", version);
//                  ↑ 如果 get() 返回 nullptr，这里就崩溃
```

### `static` 关键字的作用

注意 `main.cpp` 中 logger 变量都用了 `static` 修饰：

```cpp
static auto core_logger = spdlog::stdout_color_mt("core");
```

这里的 `static` 有两个作用：

1. **延长生命周期**：`static` 局部变量的生命周期从首次执行到程序结束。即使 `main()` 函数返回了，`core_logger` 指向的 logger 对象不会被销毁。这确保了在程序清理阶段（如全局对象的析构函数中）如果还有日志调用，logger 仍然有效。

2. **保证只初始化一次**：如果 `main()` 被意外调用两次（虽然正常情况不会），`static` 变量只在第一次通过时初始化，避免重复创建同名 logger 导致异常。

不使用 `static` 也能工作（因为 logger 注册到全局注册表后由 `shared_ptr` 管理生命周期），但 `static` 提供了额外的安全保障。

### 完整示例：从零演示注册表工作流

```cpp
// 完整可运行示例：演示 spdlog 全局注册表机制
#include <spdlog/spdlog.h>
#include <spdlog/sinks/stdout_color_sinks.h>
#include <iostream>

int main() {
    // 1. 注册表初始状态为空
    auto before = spdlog::get("my_logger");
    std::cout << "注册前: "
              << (before ? "找到了" : "未找到（nullptr）")
              << std::endl;

    // 2. 创建并注册 logger
    auto logger = spdlog::stdout_color_mt("my_logger");
    // 此时全局注册表中有了 "my_logger" 条目

    // 3. 从注册表查找
    auto after = spdlog::get("my_logger");
    std::cout << "注册后: "
              << (after ? "找到了" : "未找到")
              << std::endl;

    // 4. 两种方式获得的是同一个 logger 对象
    std::cout << "是同一对象: "
              << (logger.get() == after.get() ? "是" : "否")
              << std::endl;

    // 5. 通过任一引用都可以输出日志
    logger->info("通过创建时的引用输出");
    after->info("通过 get() 获得的引用输出");

    // 6. 清理：移除并释放（通常程序结束时自动完成）
    spdlog::drop("my_logger");
    auto dropped = spdlog::get("my_logger");
    std::cout << "移除后: "
              << (dropped ? "找到了" : "未找到（nullptr）")
              << std::endl;

    return 0;
}
```

输出：

```
注册前: 未找到（nullptr）
注册后: 找到了
是同一对象: 是
[2026-03-21 10:00:00.000] [my_logger] [info] 通过创建时的引用输出
[2026-03-21 10:00:00.000] [my_logger] [info] 通过 get() 获得的引用输出
移除后: 未找到（nullptr）
```

编译方法（需要 spdlog 已通过 vcpkg 安装）：

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.16)
project(spdlog_demo LANGUAGES CXX)
find_package(spdlog REQUIRED)
add_executable(demo main.cpp)
target_link_libraries(demo PRIVATE spdlog::spdlog)
```

---

## 动手实践：编写安全的测试入口

在实际项目中，有时你不确定被测代码到底会调用哪些 logger。下面展示一种"防御式"写法，比项目现有的 `main.cpp` 更安全：

```cpp
// 防御式 main.cpp——安全地创建所有可能需要的 logger
#include <catch2/catch_session.hpp>
#include <spdlog/spdlog.h>
#include <spdlog/sinks/stdout_color_sinks.h>

// 辅助函数：安全创建 logger（如果已存在则跳过）
void ensure_logger(const std::string& name) {
    if (!spdlog::get(name)) {  // 检查注册表中是否已有同名 logger
        spdlog::stdout_color_mt(name);  // 不存在则创建
    }
}

int main(int argc, char* argv[]) {
    // 创建 KrKr2 引擎可能用到的所有 logger
    ensure_logger("core");    // 核心引擎日志
    ensure_logger("tjs2");    // TJS2 脚本引擎日志
    ensure_logger("plugin");  // 插件系统日志

    // 设置日志级别为 warn，避免测试输出过多日志干扰结果
    spdlog::set_level(spdlog::level::warn);

    // 运行 Catch2 测试
    int result = Catch::Session().run(argc, argv);

    // 清理所有 logger（可选，程序退出时自动清理）
    spdlog::shutdown();

    return result;
}
```

这种写法的三个改进：

1. **`ensure_logger` 防重复创建**：如果某个 logger 已经存在（比如被测代码自己创建了），不会因为重复调用 `stdout_color_mt` 而抛出异常。spdlog 在尝试注册同名 logger 时会抛出 `spdlog_ex` 异常。
2. **`spdlog::set_level(warn)`**：把所有 logger 的输出级别调高到 warn，避免测试运行时满屏都是 info/debug 日志，让 Catch2 的测试结果更清晰。
3. **`spdlog::shutdown()`**：显式清理资源。虽然程序退出时会自动清理，但在某些平台（尤其是 Windows）下显式 shutdown 可以避免析构顺序导致的崩溃。

---

## 对照项目源码

本节涉及的所有项目源文件：

| 文件路径 | 行数 | 关键内容 |
|----------|------|---------|
| `tests/unit-tests/plugins/main.cpp` 第 1-18 行 | 18 | 创建 3 个 logger：core、tjs2、plugin |
| `tests/unit-tests/core/movie/main.cpp` 第 1-17 行 | 17 | 创建 2 个 logger：core、tjs2 |
| `tests/unit-tests/core/tjs2/main.cpp` 第 1-17 行 | 17 | 创建 2 个 logger：core、tjs2（与 movie 完全相同） |

### 关键发现

1. **所有 main.cpp 都使用 `stdout_color_mt`**——选择多线程安全版本（mt 后缀），因为 Catch2 在某些模式下可能使用多线程运行测试。
2. **Header 包含顺序**：`catch2/catch_session.hpp` 在前，`spdlog` 在后。这不是必须的，但遵循了"先第三方框架、后日志库"的习惯。
3. **没有设置日志级别**——意味着 logger 使用默认的 `info` 级别，测试时会看到引擎的 info 日志输出。这在调试测试失败时很有用，但在正常运行时可能产生噪音。
4. **没有 `spdlog::shutdown()` 调用**——项目选择依赖程序退出时的自动清理。这种做法在 Linux/macOS 上没问题，但在 Windows 上偶尔可能出现退出时的崩溃（全局对象析构顺序不确定性，即 Static Initialization Order Fiasco 的反向问题——Static Destruction Order Fiasco）。

### 常见错误与解决方案

**错误 1：重复创建同名 logger**

```
terminate called after throwing an instance of 'spdlog::spdlog_ex'
  what(): logger with name 'core' already exists
```

**原因：** 在 main 或测试代码中调用了两次 `spdlog::stdout_color_mt("core")`。
**解决：** 使用 `ensure_logger` 模式（先 `get` 检查再创建），或确保只在 `main()` 中创建一次。

**错误 2：测试中看不到日志输出**

**原因：** 日志级别被设置为 `err` 或 `critical`，低于 `info` 级别的日志被过滤了。
**解决：** 在 main 中添加 `spdlog::set_level(spdlog::level::debug)` 来显示所有级别的日志。

**错误 3：运行时崩溃但错误信息不明**

```
Segmentation fault (core dumped)
```

**原因：** 高概率是缺少某个 logger。被测代码调用了 `spdlog::get("xxx")` 但 main 中没有创建 `"xxx"` logger。
**解决：** 用 GDB 或 LLDB 调试器定位崩溃点，通常在 `spdlog::get()->info()` 调用处。找到缺失的 logger 名后在 main 中补上。

跨平台调试命令：

```bash
# Linux — 使用 GDB
gdb ./psbfile-dll
(gdb) run
# 崩溃后查看调用栈
(gdb) backtrace

# macOS — 使用 LLDB
lldb ./psbfile-dll
(lldb) run
(lldb) bt

# Windows — 使用 Visual Studio 调试器
# 在 VS 中打开项目，设置测试可执行文件为启动项目，F5 运行
# 崩溃时 VS 会自动显示调用栈
```

---

## 本节小结

- KrKr2 使用**自定义 `main()`** 而非 Catch2 默认入口，原因是引擎代码需要 spdlog logger 提前就绪
- 三个 `main.cpp` 的核心逻辑相同：**创建 logger → 运行 Catch2 测试 → 返回结果码**
- plugins 版本多创建了 `"plugin"` logger（3 个），core 版本只需 `"core"` 和 `"tjs2"`（2 个）
- spdlog 的**全局注册表**机制：`stdout_color_mt()` 创建并注册 logger，`get()` 按名字查找
- `static` 关键字延长 logger 变量的生命周期，确保程序清理阶段 logger 仍然有效
- 防御式写法建议：用 `ensure_logger` 模式（先检查再创建）避免重复注册异常
- 调试 logger 缺失导致的崩溃：用 GDB/LLDB/VS 定位 `spdlog::get()` 返回 nullptr 的位置

---

## 练习题与答案

### 题目 1：判断 logger 创建时机

以下测试代码能正常运行吗？如果不能，请说明原因并修复。

```cpp
// main.cpp
#include <catch2/catch_session.hpp>
#include <spdlog/sinks/stdout_color_sinks.h>

int main(int argc, char* argv[]) {
    int result = Catch::Session().run(argc, argv);

    static auto core_logger = spdlog::stdout_color_mt("core");
    static auto tjs2_logger = spdlog::stdout_color_mt("tjs2");

    return result;
}
```

<details>
<summary>查看答案</summary>

**不能正常运行**。

问题出在 logger 创建的**时机**：`spdlog::stdout_color_mt()` 调用在 `Catch::Session().run()` **之后**。这意味着当 Catch2 执行测试用例时，全局注册表中还没有任何 logger。如果测试用例触发的引擎代码调用了 `spdlog::get("core")`，会返回 `nullptr` 并在后续调用时崩溃。

修复方法——把 logger 创建移到 `run()` 之前：

```cpp
int main(int argc, char* argv[]) {
    // 必须在 run() 之前创建 logger
    static auto core_logger = spdlog::stdout_color_mt("core");
    static auto tjs2_logger = spdlog::stdout_color_mt("tjs2");

    int result = Catch::Session().run(argc, argv);
    return result;
}
```

**关键原则：** 所有被测代码可能需要的全局状态，必须在 `Catch::Session().run()` 调用之前准备好。

</details>

### 题目 2：为 `core/sound` 模块编写 main.cpp

已知音频模块的代码中会调用以下 spdlog logger：
- `spdlog::get("core")` — 核心引擎日志
- `spdlog::get("tjs2")` — TJS2 引擎日志（某些音频脚本绑定会用到）

请为 `tests/unit-tests/core/sound/main.cpp` 编写完整代码。

<details>
<summary>查看答案</summary>

```cpp
// tests/unit-tests/core/sound/main.cpp — 音频模块测试入口
//
// Created for KrKr2 test infrastructure
//

#include <catch2/catch_session.hpp>       // Catch::Session
#include <spdlog/sinks/stdout_color_sinks.h>  // stdout_color_mt

int main(int argc, char *argv[]) {
    // 创建音频模块代码所需的 logger
    // 不需要 "plugin" logger，因为 sound 模块不涉及插件代码
    static auto core_logger = spdlog::stdout_color_mt("core");
    static auto tjs2_logger = spdlog::stdout_color_mt("tjs2");

    // 运行所有测试
    int result = Catch::Session().run(argc, argv);

    return result;
}
```

这个文件与 `core/movie/main.cpp` 和 `core/tjs2/main.cpp` 完全一致，因为它们的依赖 logger 集合相同（都是 "core" + "tjs2"）。

如果未来音频模块增加了插件依赖（比如通过插件加载自定义音频解码器），则需要补上 `spdlog::stdout_color_mt("plugin")` 这一行。

</details>

### 题目 3：分析多线程安全性

下面的代码在多线程环境下安全吗？请分析可能的问题。

```cpp
// 在某个测试中
TEST_CASE("并发日志测试") {
    auto logger = spdlog::get("core");

    std::vector<std::thread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back([&logger, i]() {
            logger->info("线程 {} 写入日志", i);
        });
    }

    for (auto& t : threads) {
        t.join();
    }
}
```

<details>
<summary>查看答案</summary>

**这段代码是安全的**，原因有两个：

1. **`spdlog::get()` 是线程安全的**：全局注册表内部有互斥锁保护，多线程并发调用 `get()` 不会产生数据竞争。

2. **`logger->info()` 是线程安全的**：因为 KrKr2 使用的是 `stdout_color_mt`（mt = Multi-Threaded），这个版本的 logger 内部对每次写入都有互斥锁保护，保证多线程同时写日志不会出现输出交错或数据损坏。

如果项目使用的是 `stdout_color_st`（st = Single-Threaded），那么这段代码就**不安全**了——多个线程同时调用 `info()` 可能导致输出内容交错、缓冲区损坏甚至崩溃。

**性能注意事项**：`_mt` 版本每次写日志都要获取互斥锁，在高频日志场景下可能成为性能瓶颈。但对于测试场景来说完全可以接受。如果性能敏感的生产代码中需要高吞吐日志，可以考虑使用 spdlog 的异步模式（`spdlog::async_factory`），它使用无锁队列缓冲日志消息。

</details>

---

## 下一步

下一节 [TEST_FILES_PATH 与测试夹具](./03-TEST_FILES_PATH与测试夹具.md) 将深入讲解测试夹具（Test Fixture）的管理方式，包括 `configure_file` 的详细工作原理、夹具文件的组织规范、以及如何在测试中加载和使用二进制/脚本夹具文件。

