# 第四步：日志接口与 CMake 统一构建

> **所属模块：** P03-跨平台C++开发
> **前置知识：** [线程接口与实现](./03-线程接口与实现.md)、[CMake 传递平台信息与 KrKr2 实例](../02-预处理器与条件编译/02-CMake传递平台信息与KrKr2实例.md)、[PAL 概念与编译期文件分离](../03-平台抽象层设计模式/01-PAL概念与编译期文件分离.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：
1. 设计一个支持日志级别、标签过滤的跨平台日志接口
2. 理解 C 可变参数（`va_list` / `va_start` / `va_end`）的工作原理和安全陷阱
3. 用 `OutputDebugStringA` + `stderr` 实现 Windows 日志后端
4. 用 ANSI 转义序列实现 POSIX 彩色终端日志
5. 用 `__android_log_vprint` 将日志输出到 Android logcat
6. 编写一个完整的 CMakeLists.txt，根据目标平台自动选择源文件、链接正确的库
7. 识别日志与 CMake 构建中的 3 个常见陷阱

---

## 为什么日志模块是 PAL 的必备组件？

**场景**：你已经为 Mini-PAL 实现了文件系统和线程模块。某天你在 Android 设备上跑测试，程序崩溃了——但屏幕上什么信息都没有。你在代码里加了 `printf("到这里了\n")`，编译、安装、运行……还是什么都看不到。

原因很简单：**Android 没有"控制台"这个概念**。`printf` / `std::cout` 输出到 `stdout`，但 Android 应用的 `stdout` 默认连接到 `/dev/null`——所有输出直接被丢弃。要在 Android 上看到日志，你必须使用专门的 `__android_log_print` API，日志会出现在 Android Studio 的 Logcat 面板里。

这就是典型的"同一段代码在三个平台上行为完全不同"问题：

| 平台 | `printf` / `cout` 去哪了？ | 开发者通常怎么看日志？ |
|------|---------------------------|----------------------|
| **Windows** | 控制台窗口（如果有的话） | Visual Studio 的"输出"窗口（需要 `OutputDebugStringA`） |
| **Linux / macOS** | 终端（stderr / stdout） | 终端直接可见，支持 ANSI 颜色 |
| **Android** | `/dev/null`（丢弃！） | Android Studio Logcat（需要 `<android/log.h>`） |

所以我们需要一个日志抽象层：对外提供统一的 `Log(level, format, ...)` 接口，内部根据平台选择正确的输出通道。

---

## 前置概念：C 可变参数函数（Variadic Functions）

在开始写日志接口之前，你必须先理解一个关键技术——**可变参数函数**（variadic function）。日志函数的签名是 `Log(LogLevel level, const char* fmt, ...)`，末尾那三个点 `...` 就表示"可以传任意数量、任意类型的参数"。

### 什么是可变参数？

可变参数是 C 语言从诞生之初就有的特性。最经典的可变参数函数就是 `printf`：

```cpp
printf("Hello, %s! You are %d years old.\n", name, age);
//      ^^^^^^^^ 固定参数（格式字符串）     ^^^^^^^^^^^^ 可变参数
```

`printf` 不知道你会传多少个参数、什么类型——它通过格式字符串中的 `%s`、`%d` 来"猜测"。这既是它的灵活之处，也是它的危险之处。

### `va_list` 四件套

要在你自己的函数中处理可变参数，需要使用 `<cstdarg>` 中的四个宏：

```cpp
#include <cstdarg>  // 必须包含这个头文件

void MyLog(const char* fmt, ...) {
    // 第 1 步：声明一个 va_list 变量——它是一个"指针"，指向参数列表
    va_list args;

    // 第 2 步：初始化——告诉它从哪个参数之后开始
    //          第二个参数必须是 ... 前面的最后一个命名参数
    va_start(args, fmt);

    // 第 3 步：使用——把 va_list 传给 vprintf / vsnprintf 等 v 系列函数
    vprintf(fmt, args);

    // 第 4 步：清理——必须调用，否则可能栈损坏
    va_end(args);
}
```

**类比**：想象你面前有一排礼物盒（参数），但你不知道有多少个、每个里面装了什么。`va_start` 是"打开第一个盒子"，`va_arg` 是"取出内容并移到下一个"，`va_end` 是"关上所有盒子"。而 `vsnprintf` 就像一个助手——你把格式字符串和整排盒子交给它，它自动帮你依次打开。

### 为什么用 `vsnprintf` 而不是 `va_arg`？

你可能见过这样的写法：

```cpp
int val = va_arg(args, int);  // 手动取一个 int 参数
```

但在日志场景中，我们不会手动逐个取参数。原因有三：
1. **类型安全**：`va_arg` 要求你指定正确的类型，传错了就是未定义行为（崩溃或乱码）
2. **个数未知**：你不知道用户传了几个参数
3. **格式字符串解析**：`vsnprintf` 已经帮你做了最难的部分——解析 `%s`、`%d`、`%f` 并正确取出对应参数

所以日志函数的标准做法是：**用 `va_start` + `va_end` 包裹，中间把 `va_list` 交给 `vsnprintf` 做格式化**。

```cpp
// 这是日志函数的核心模式——记住这个模式
void Log(const char* fmt, ...) {
    char buffer[1024];           // 准备一个缓冲区
    va_list args;
    va_start(args, fmt);
    vsnprintf(buffer, sizeof(buffer), fmt, args);  // 格式化到缓冲区
    va_end(args);
    // 现在 buffer 里是完整的日志文本，可以输出了
}
```

> **安全要点**：`vsnprintf` 的第二个参数 `sizeof(buffer)` 限制了写入的最大字节数，**保证不会缓冲区溢出**。对比之下，`vsprintf`（没有 `n`）不限制长度，是严重的安全隐患——永远不要用它。

---

## 设计日志接口

有了可变参数的知识，现在来设计 `log.h`。一个合格的日志系统至少需要三个要素：

1. **日志级别**：区分调试信息、一般信息、警告和错误，方便过滤
2. **标签**：标识日志来源（"MiniPAL"、"Renderer"、"Audio"），方便在大型项目中定位
3. **格式化输出**：支持 `printf` 风格的格式字符串

```cpp
// include/pal/log.h — 跨平台日志接口
#pragma once
#include "platform.h"
#include <cstdarg>   // va_list

namespace pal {

// ======== 日志级别 ========
// 从低到高排列，高级别的日志在发布版本中仍然输出
enum class LogLevel {
    Debug,    // 调试信息：仅开发时使用，发布版可关闭
    Info,     // 一般信息：程序正常运行的关键节点
    Warn,     // 警告：不影响功能但需要注意的异常情况
    Error     // 错误：功能已受影响，需要立即处理
};

// ======== 接口函数 ========

// 初始化日志系统
// @param tag  日志标签，用于标识日志来源（Android logcat 按标签过滤）
// 对照 KrKr2：spdlog::logger 构造时传入 logger name，如 "core"、"tjs2"
PAL_API void LogInit(const char* tag);

// 输出一条日志
// @param level  日志级别
// @param fmt    printf 风格的格式字符串
// @param ...    可变参数
// 对照 KrKr2：spdlog::info("message {}", arg) 使用 fmt 库语法
PAL_API void Log(LogLevel level, const char* fmt, ...);

// 关闭日志系统，刷新所有缓冲
PAL_API void LogShutdown();

} // namespace pal
```

### 为什么用 `enum class` 而不是普通 `enum`？

```cpp
// ❌ 普通 enum — 值泄漏到外层作用域
enum LogLevel { Debug, Info, Warn, Error };
int Debug = 42;  // 编译错误！Debug 已被 enum 占用

// ✅ enum class — 值被限定在枚举名下
enum class LogLevel { Debug, Info, Warn, Error };
int Debug = 42;  // 完全没问题，LogLevel::Debug 和 Debug 互不干扰
```

`enum class`（C++11 引入的强类型枚举）不会把枚举值泄漏到外层命名空间，避免了名称冲突。在实际项目中，`Debug`、`Info` 这种常见单词如果用普通 `enum` 定义，几乎必然跟其他变量或宏冲突。

---

## Windows 日志实现

Windows 平台有一个独特的日志通道：`OutputDebugStringA`。这个 Win32 API 把字符串发送到调试器的"输出"窗口——如果你用 Visual Studio 调试程序，日志会直接出现在下方的 Output 面板里。

```cpp
// src/win32/log.cpp — Windows 日志实现
#include "pal/log.h"
#include <windows.h>   // OutputDebugStringA
#include <cstdio>      // fprintf, vsnprintf
#include <cstdarg>     // va_list, va_start, va_end

namespace pal {

// 全局日志标签，默认 "MiniPAL"
// 用 static 限制为文件内部可见（内部链接）
static char g_tag[64] = "MiniPAL";

void LogInit(const char* tag) {
    // snprintf 安全拷贝——即使 tag 超长也不会溢出 g_tag
    snprintf(g_tag, sizeof(g_tag), "%s", tag);
}

void Log(LogLevel level, const char* fmt, ...) {
    // ---- 第一步：格式化用户消息 ----
    char msg[1024];
    va_list args;
    va_start(args, fmt);
    vsnprintf(msg, sizeof(msg), fmt, args);  // 安全格式化，最多写 1023 字节 + '\0'
    va_end(args);

    // ---- 第二步：确定级别前缀 ----
    const char* prefix = "";
    switch (level) {
        case LogLevel::Debug: prefix = "[DEBUG]"; break;
        case LogLevel::Info:  prefix = "[INFO] "; break;
        case LogLevel::Warn:  prefix = "[WARN] "; break;
        case LogLevel::Error: prefix = "[ERROR]"; break;
    }

    // ---- 第三步：组装完整日志行 ----
    char full[1200];  // 1024(msg) + 前缀 + tag + 余量
    snprintf(full, sizeof(full), "%s %s: %s\n", prefix, g_tag, msg);

    // ---- 第四步：双通道输出 ----
    OutputDebugStringA(full);    // → Visual Studio "输出"窗口
    fprintf(stderr, "%s", full); // → 控制台（如果有的话）
}

void LogShutdown() {
    // Windows 无需特殊清理
    // OutputDebugStringA 没有需要关闭的资源
}

} // namespace pal
```

### `OutputDebugStringA` 深入

你可能会问：为什么不只用 `printf` 或 `fprintf(stderr, ...)`？

| 输出方式 | 控制台窗口可见？ | VS 调试器可见？ | GUI 程序可见？ |
|---------|----------------|---------------|--------------|
| `printf` | ✅ 是 | ❌ 否 | ❌ 否（GUI 程序没有控制台） |
| `fprintf(stderr, ...)` | ✅ 是 | ❌ 否 | ❌ 否 |
| `OutputDebugStringA` | ❌ 否 | ✅ 是 | ✅ 是（调试器附加时） |
| 两者都用 | ✅ 是 | ✅ 是 | 部分可见 |

游戏引擎（如 KrKr2）通常是 GUI 程序（`WinMain` 入口），没有控制台窗口。这时候 `printf` 的输出无处可去，只有 `OutputDebugStringA` 才能把日志送到开发者面前。我们的 Mini-PAL 两者都用，确保覆盖所有使用场景。

> **注意**：`OutputDebugStringA` 是 ANSI 版本（接受 `char*`），还有 `OutputDebugStringW`（接受 `wchar_t*`）。我们用 A 版本是因为 UTF-8 字符串在 A 版本中也能正确显示（Windows 10 1903+ 支持 UTF-8 codepage）。

---

## POSIX 日志实现（Linux + macOS）

POSIX 平台（Linux 和 macOS）的日志直接输出到终端的 stderr 流。但我们要加两个增强功能：**时间戳**和**彩色输出**。

### ANSI 转义序列入门

你在终端里见过彩色文字吗？那就是 ANSI 转义序列的功劳。它的格式是：

```
\033[<参数>m
```

其中 `\033` 是 ESC 字符（ASCII 27），`[` 是固定前缀，`m` 是固定后缀，中间的参数决定颜色。最常用的颜色代码：

| 代码 | 颜色 | 常用于 |
|------|------|--------|
| `31` | 红色 | 错误 |
| `32` | 绿色 | 成功/信息 |
| `33` | 黄色 | 警告 |
| `36` | 青色 | 调试 |
| `0`  | 重置 | 恢复默认颜色 |

使用方式：先输出颜色代码，再输出文本，最后**必须**输出重置代码 `\033[0m`：

```cpp
// 输出红色的 "ERROR"
fprintf(stderr, "\033[31mERROR\033[0m: something went wrong\n");
//              ^^^^^^^^ 切换到红色  ^^^^^^^^ 恢复默认
```

如果忘记 `\033[0m`，之后所有终端输出都会是红色——这是一个常见坑。

### `localtime_r` vs `localtime_s`

日志加上时间戳能帮助定位"这个事件什么时候发生的"。但获取本地时间的 API 在不同平台上签名不同：

```cpp
// POSIX（Linux / macOS）— localtime_r：结果写入你提供的 struct tm
struct tm t;
localtime_r(&now, &t);   // 参数顺序：(输入指针, 输出指针)

// Windows — localtime_s：参数顺序正好反过来！
struct tm t;
localtime_s(&t, &now);   // 参数顺序：(输出指针, 输入指针)
```

两者功能一样（把 `time_t` 转成本地时间的 `struct tm`），但参数顺序相反——这种"差一点就一样"的 API 差异是跨平台开发中最容易出错的。还有一个不太安全的 `localtime()`（不带后缀），它返回一个全局静态的 `struct tm*`，**在多线程环境下不安全**——两个线程同时调用会互相覆盖结果。

现在来看完整的 POSIX 日志实现：

```cpp
// src/posix/log.cpp — POSIX 日志实现（Linux + macOS 共用）
#include "pal/log.h"
#include <cstdio>      // fprintf, vsnprintf
#include <cstdarg>     // va_list
#include <ctime>       // time, localtime_r, struct tm

namespace pal {

static char g_tag[64] = "MiniPAL";

void LogInit(const char* tag) {
    snprintf(g_tag, sizeof(g_tag), "%s", tag);
}

void Log(LogLevel level, const char* fmt, ...) {
    // ---- 格式化用户消息 ----
    char msg[1024];
    va_list args;
    va_start(args, fmt);
    vsnprintf(msg, sizeof(msg), fmt, args);
    va_end(args);

    // ---- 根据级别选择颜色前缀 ----
    const char* prefix = "";
    switch (level) {
        case LogLevel::Debug: prefix = "\033[36m[DEBUG]\033[0m"; break; // 青色
        case LogLevel::Info:  prefix = "\033[32m[INFO] \033[0m"; break; // 绿色
        case LogLevel::Warn:  prefix = "\033[33m[WARN] \033[0m"; break; // 黄色
        case LogLevel::Error: prefix = "\033[31m[ERROR]\033[0m"; break; // 红色
    }

    // ---- 添加时间戳 ----
    time_t now = time(nullptr);          // 获取当前 Unix 时间戳
    struct tm t;
    localtime_r(&now, &t);              // 线程安全的本地时间转换

    // ---- 输出到 stderr ----
    // 格式：HH:MM:SS [LEVEL] TAG: message
    fprintf(stderr, "%02d:%02d:%02d %s %s: %s\n",
            t.tm_hour, t.tm_min, t.tm_sec,
            prefix, g_tag, msg);
}

void LogShutdown() {
    fflush(stderr);  // 确保所有缓冲的日志已写入
}

} // namespace pal
```

> **Linux vs macOS 差异**：两者都用 stderr 输出，代码完全一致。但 macOS 的 `Console.app`（控制台应用）也会捕获 stderr 输出。如果你需要更原生的 macOS 日志集成，可以使用 `os_log` API（`<os/log.h>`），但这超出了入门实战的范围。

### 为什么输出到 `stderr` 而不是 `stdout`？

```
stdout  ——  标准输出  ——  程序的"正常结果"
stderr  ——  标准错误  ——  程序的"诊断信息"
```

日志是"诊断信息"，不是程序的"正常输出"。使用 stderr 有两个实际好处：

1. **管道不干扰**：`./myapp | grep pattern` 只过滤 stdout，stderr 的日志照常显示
2. **缓冲策略不同**：stderr 默认无缓冲（立即输出），stdout 默认行缓冲——程序崩溃时 stdout 缓冲区里的数据可能丢失，但 stderr 的日志已经输出了

---

## Android 日志实现

Android 的日志系统与桌面平台完全不同。Android 有一个系统级的日志服务叫 **logcat**，所有应用的日志都汇集到这里。开发者通过 Android Studio 的 Logcat 面板查看和过滤日志。

### logcat 架构简介

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   你的 C++ 代码   │    │  其他应用的日志   │    │  系统服务日志     │
│ __android_log_*  │    │  Log.d("tag",..) │    │  kernel, init..  │
└───────┬─────────┘    └───────┬─────────┘    └───────┬─────────┘
        │                      │                      │
        ▼                      ▼                      ▼
┌──────────────────────────────────────────────────────────────┐
│                    logcat 环形缓冲区                          │
│            （内核级日志收集器，所有日志汇总到这里）              │
└──────────────────────────────┬───────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Android Studio     │
                    │  Logcat 面板         │
                    │  可按 tag/level 过滤  │
                    └─────────────────────┘
```

NDK 提供了 `<android/log.h>` 头文件，其中最常用的是：

- `__android_log_print(prio, tag, fmt, ...)` — 直接输出
- `__android_log_vprint(prio, tag, fmt, va_list)` — 接受 `va_list`（我们用这个）
- `__android_log_write(prio, tag, msg)` — 输出已格式化的字符串

```cpp
// src/android/log.cpp — Android logcat 日志实现
#include "pal/log.h"
#include <android/log.h>   // __android_log_vprint, ANDROID_LOG_*
#include <cstdarg>         // va_list

namespace pal {

static char g_tag[64] = "MiniPAL";

void LogInit(const char* tag) {
    snprintf(g_tag, sizeof(g_tag), "%s", tag);
    // Android logcat 不需要初始化——系统服务始终在运行
}

void Log(LogLevel level, const char* fmt, ...) {
    // ---- 映射 LogLevel → Android 日志优先级 ----
    int android_level;
    switch (level) {
        case LogLevel::Debug: android_level = ANDROID_LOG_DEBUG; break;
        case LogLevel::Info:  android_level = ANDROID_LOG_INFO;  break;
        case LogLevel::Warn:  android_level = ANDROID_LOG_WARN;  break;
        case LogLevel::Error: android_level = ANDROID_LOG_ERROR; break;
        default:              android_level = ANDROID_LOG_INFO;  break;
    }

    // ---- 直接把 va_list 交给 Android API ----
    va_list args;
    va_start(args, fmt);
    __android_log_vprint(android_level, g_tag, fmt, args);
    //                   ^^^^^^^^^^^^  ^^^^^  ^^^  ^^^^
    //                   优先级        标签   格式  参数列表
    va_end(args);
}

void LogShutdown() {
    // Android logcat 无需清理——它是系统服务
}

} // namespace pal
```

### 为什么 Android 实现最简单？

注意 Android 版本没有自己做格式化（没有 `vsnprintf` + 缓冲区），直接把 `va_list` 交给了 `__android_log_vprint`。这是因为 logcat 系统自带格式化能力——你不需要自己拼接字符串。这也意味着：

1. **不需要时间戳**：logcat 自动给每条日志加上精确到毫秒的时间戳
2. **不需要颜色**：Logcat 面板根据优先级自动着色
3. **不需要刷新**：logcat 是实时的

> **对照 KrKr2**：`AndroidUtils.cpp` 中多处使用 `spdlog::info()` / `spdlog::error()`。spdlog 在 Android 上可配置 `android_sink`，底层也是调用 `__android_log_write`。我们这里直接使用 NDK API 是为了展示最底层的机制。

### 三平台日志实现对比

| 特性 | Windows | POSIX (Linux/macOS) | Android |
|------|---------|--------------------:|---------|
| 输出通道 | `OutputDebugStringA` + `stderr` | `stderr` | logcat 系统服务 |
| 时间戳 | 无（调试器自带） | 手动添加 `localtime_r` | 无（logcat 自带） |
| 颜色 | 无（调试器自带高亮） | ANSI 转义序列 | 无（Logcat 面板自带） |
| 格式化 | `vsnprintf` 手动拼接 | `vsnprintf` 手动拼接 | `__android_log_vprint` 内部处理 |
| 清理工作 | 无 | `fflush(stderr)` | 无 |
| 头文件 | `<windows.h>` | `<ctime>` | `<android/log.h>` |
| 链接库 | 无需额外链接 | 无需额外链接 | 必须链接 `log` 库 |

---

## CMake 统一构建：把一切组装起来

到目前为止，我们的 Mini-PAL 有四个模块：`platform.h`（头文件）、`filesystem`、`thread`、`log`，每个模块有 2-3 个平台实现。现在需要一个 CMakeLists.txt 把它们全部组装起来——**根据目标平台自动选择正确的源文件和链接库**。

### CMake 平台检测

CMake 内置变量 `CMAKE_SYSTEM_NAME` 在配置阶段自动被设置为目标平台的名称。它的值因构建方式而异：

| 构建方式 | `CMAKE_SYSTEM_NAME` 的值 | 说明 |
|---------|--------------------------|------|
| Windows 本地编译 | `"Windows"` | 自动检测 |
| Linux 本地编译 | `"Linux"` | 自动检测 |
| macOS 本地编译 | `"Darwin"` | 注意是 Darwin 不是 macOS |
| Android NDK 交叉编译 | `"Android"` | NDK 工具链文件自动设置 |

> **常见困惑**：macOS 的 `CMAKE_SYSTEM_NAME` 是 `"Darwin"`（macOS 的内核名称），不是 `"macOS"`。这是初学者经常写错的地方。

### 完整的 CMakeLists.txt

```cmake
# CMakeLists.txt — Mini-PAL 顶层构建文件
cmake_minimum_required(VERSION 3.20)
project(MiniPAL LANGUAGES CXX)

# C++17 标准，强制要求（REQUIRED = 编译器不支持就报错）
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ========== 第一步：平台检测 ==========
# 对照 KrKr2：根 CMakeLists.txt 第 13-36 行使用 if(ANDROID)/if(WINDOWS) 设置平台变量
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    set(PAL_PLATFORM "win32")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Android")
    set(PAL_PLATFORM "android")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set(PAL_PLATFORM "posix")          # macOS 走 POSIX 路径
    set(PAL_IS_MACOS TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(PAL_PLATFORM "posix")          # Linux 也走 POSIX 路径
    set(PAL_IS_LINUX TRUE)
else()
    message(FATAL_ERROR "不支持的平台: ${CMAKE_SYSTEM_NAME}")
endif()

# 打印平台信息（配置阶段可见）
message(STATUS "目标平台: ${PAL_PLATFORM} (${CMAKE_SYSTEM_NAME})")

# ========== 第二步：收集头文件 ==========
set(PAL_HEADERS
    include/pal/platform.h
    include/pal/filesystem.h
    include/pal/thread.h
    include/pal/log.h
)

# ========== 第三步：根据平台选择源文件 ==========
set(PAL_SOURCES "")

if(PAL_PLATFORM STREQUAL "win32")
    list(APPEND PAL_SOURCES
        src/win32/filesystem.cpp
        src/win32/thread.cpp
        src/win32/log.cpp
    )
elseif(PAL_PLATFORM STREQUAL "android")
    list(APPEND PAL_SOURCES
        src/android/filesystem.cpp
        src/posix/thread.cpp           # Android 复用 POSIX 线程实现
        src/android/log.cpp
    )
elseif(PAL_PLATFORM STREQUAL "posix")
    list(APPEND PAL_SOURCES
        src/posix/filesystem.cpp
        src/posix/thread.cpp
        src/posix/log.cpp
    )
endif()

# ========== 第四步：创建库目标 ==========
# STATIC = 静态库（.a / .lib），编译时链入可执行文件
add_library(minipal STATIC ${PAL_HEADERS} ${PAL_SOURCES})

# PUBLIC = 使用 minipal 的人也能 #include "pal/xxx.h"
target_include_directories(minipal PUBLIC include)

# ========== 第五步：平台特定链接 ==========
if(PAL_PLATFORM STREQUAL "win32")
    # PRIVATE = 只有 minipal 内部用，不传递给使用者
    target_link_libraries(minipal PRIVATE psapi shell32)
elseif(PAL_PLATFORM STREQUAL "android")
    # log 库提供 __android_log_vprint
    # android 库提供 AAssetManager 等 API
    target_link_libraries(minipal PRIVATE log android)
elseif(PAL_PLATFORM STREQUAL "posix")
    # find_package(Threads) 自动找到 pthread 库
    find_package(Threads REQUIRED)
    target_link_libraries(minipal PRIVATE Threads::Threads)
endif()

# ========== 第六步：测试（可选）==========
# option() 创建一个 CMake 缓存变量，用户可以用 -DPAL_BUILD_TESTS=OFF 关闭
option(PAL_BUILD_TESTS "构建测试" ON)

# Android 没有传统的命令行测试环境，所以跳过
if(PAL_BUILD_TESTS AND NOT CMAKE_SYSTEM_NAME STREQUAL "Android")
    enable_testing()
    add_subdirectory(tests)
endif()
```

### 关键 CMake 概念详解

**`target_link_libraries` 的 PRIVATE / PUBLIC / INTERFACE**

这三个关键字控制"依赖传播"：

```cmake
# PRIVATE — 只有 minipal 自己用，不传递
target_link_libraries(minipal PRIVATE log)
# → minipal 的使用者不需要链接 log

# PUBLIC — minipal 自己用，也传递给使用者
target_link_libraries(minipal PUBLIC some_lib)
# → 使用 minipal 的人也自动链接 some_lib

# INTERFACE — minipal 自己不用，但使用者需要
target_link_libraries(minipal INTERFACE header_only_lib)
# → minipal 编译时不链接，但使用者编译时会链接
```

我们的平台库（`psapi`、`log`、`Threads::Threads`）都用 PRIVATE，因为使用 Mini-PAL 的人不需要直接调用这些平台 API——他们只需调用 `pal::Log()`。

**`find_package(Threads REQUIRED)` 是什么？**

`find_package` 是 CMake 的"自动查找"机制。`Threads` 是 CMake 内置的"查找模块"——它会自动检测系统上的线程库（Linux 上是 `-lpthread`，macOS 上可能不需要额外链接）。`REQUIRED` 表示找不到就报错。

找到之后，它会创建一个"导入目标"叫 `Threads::Threads`——你可以像用普通库名一样用它。好处是你不需要手动写 `-lpthread`（不同平台的参数可能不同）。

**测试的 CMakeLists.txt**

```cmake
# tests/CMakeLists.txt — 测试目标
add_executable(test_pal test_pal.cpp)

# 链接 minipal 库——因为 minipal 的 include 目录是 PUBLIC 的，
# test_pal 自动能 #include "pal/log.h"
target_link_libraries(test_pal PRIVATE minipal)

# 注册到 CTest，让 `ctest` 命令能自动运行它
add_test(NAME test_pal COMMAND test_pal)
```

> **对照 KrKr2**：KrKr2 的 `cmake/CocosBuildHelpers.cmake` 中的 `cocos_use_pkg` 宏（第 200-230 行）做了类似的事——根据平台有条件地链接不同库。KrKr2 的根 CMakeLists.txt 第 40-55 行使用 `if(ANDROID)` / `if(WINDOWS)` 添加平台特定源文件。

### 构建流程图

```
cmake -B build -G Ninja
      │
      ▼
┌──────────────────────────┐
│ CMake 配置阶段            │
│ 1. 检测 CMAKE_SYSTEM_NAME │
│ 2. 设置 PAL_PLATFORM      │
│ 3. 选择源文件列表          │
│ 4. 查找平台库              │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ cmake --build build      │
│ 编译阶段                  │
│ 1. 只编译选中的平台源文件   │
│ 2. 链接选中的平台库        │
│ 3. 生成 libminipal.a      │
│ 4. 生成 test_pal (如果开启) │
└──────────────────────────┘
```

---

## 常见错误与解决方案

### 错误 1：忘记调用 `va_end`

```cpp
// ❌ 错误：没有 va_end
void Log(LogLevel level, const char* fmt, ...) {
    char msg[1024];
    va_list args;
    va_start(args, fmt);
    vsnprintf(msg, sizeof(msg), fmt, args);
    // 忘记 va_end(args) 就直接 return 了！
    fprintf(stderr, "%s\n", msg);
}
```

**后果**：在某些平台（特别是使用寄存器传参的架构，如 x86_64）上，`va_start` 会修改栈状态或保存寄存器。如果不调用 `va_end` 恢复，可能导致栈损坏、内存泄漏或后续函数调用出错。C 标准明确规定：每个 `va_start` 必须有配对的 `va_end`。

```cpp
// ✅ 正确：va_start 和 va_end 成对出现
void Log(LogLevel level, const char* fmt, ...) {
    char msg[1024];
    va_list args;
    va_start(args, fmt);
    vsnprintf(msg, sizeof(msg), fmt, args);
    va_end(args);  // ← 必须！
    fprintf(stderr, "%s\n", msg);
}
```

### 错误 2：使用 `vsprintf` 而非 `vsnprintf`

```cpp
// ❌ 危险：vsprintf 不限制写入长度
char msg[256];
va_list args;
va_start(args, fmt);
vsprintf(msg, fmt, args);   // 如果格式化结果超过 256 字节 → 缓冲区溢出！
va_end(args);
```

**后果**：缓冲区溢出是 C/C++ 最严重的安全漏洞之一。攻击者可以通过构造超长的日志消息覆盖返回地址，执行任意代码。即使不考虑安全，溢出也会覆盖相邻变量，导致诡异的 Bug。

```cpp
// ✅ 安全：vsnprintf 限制最大写入长度
char msg[256];
va_list args;
va_start(args, fmt);
vsnprintf(msg, sizeof(msg), fmt, args);  // 最多写 255 字节 + '\0'
va_end(args);
```

### 错误 3：Android 链接时忘记 `log` 库

```cmake
# ❌ 错误：Android 上链接失败
if(PAL_PLATFORM STREQUAL "android")
    target_link_libraries(minipal PRIVATE android)
    # 忘了 log 库！
endif()
```

**报错信息**：
```
undefined reference to '__android_log_vprint'
```

这是因为 `__android_log_vprint` 定义在 NDK 的 `liblog.so` 中。不链接它，链接器就找不到这个符号。

```cmake
# ✅ 正确：同时链接 log 和 android 库
if(PAL_PLATFORM STREQUAL "android")
    target_link_libraries(minipal PRIVATE log android)
endif()
```

---

## 动手实践

### 实践 1：完整构建并运行日志测试

创建以下测试文件，验证日志系统在你的平台上正常工作：

```cpp
// tests/test_log.cpp — 日志模块独立测试
#include "pal/log.h"
#include <cstdio>

int main() {
    // 初始化日志，设置标签
    pal::LogInit("LogTest");

    // 输出各级别日志
    pal::Log(pal::LogLevel::Debug, "这是调试信息: x = %d", 42);
    pal::Log(pal::LogLevel::Info,  "程序启动成功，版本 %s", "1.0.0");
    pal::Log(pal::LogLevel::Warn,  "配置文件未找到，使用默认值");
    pal::Log(pal::LogLevel::Error, "无法打开文件: %s (错误码: %d)", "data.bin", 2);

    // 测试长消息（接近缓冲区上限）
    char long_msg[900];
    for (int i = 0; i < 899; ++i) long_msg[i] = 'A';
    long_msg[899] = '\0';
    pal::Log(pal::LogLevel::Info, "长消息测试: %s", long_msg);

    // 测试空格式字符串
    pal::Log(pal::LogLevel::Info, "无参数的日志");

    // 测试多参数
    pal::Log(pal::LogLevel::Debug, "坐标: (%d, %d), 缩放: %.2f, 名称: %s",
             100, 200, 1.5, "player");

    pal::LogShutdown();

    printf("日志测试完成！请检查上方输出。\n");
    return 0;
}
```

构建和运行命令：

```bash
# Windows (PowerShell)
cmake -B build -G Ninja
cmake --build build
.\build\tests\test_log.exe

# Linux / macOS
cmake -B build -G Ninja
cmake --build build
./build/tests/test_log
```

**预期输出**（POSIX 平台，带颜色和时间戳）：
```
14:30:05 [DEBUG] LogTest: 这是调试信息: x = 42
14:30:05 [INFO]  LogTest: 程序启动成功，版本 1.0.0
14:30:05 [WARN]  LogTest: 配置文件未找到，使用默认值
14:30:05 [ERROR] LogTest: 无法打开文件: data.bin (错误码: 2)
14:30:05 [INFO]  LogTest: 长消息测试: AAAA...（截断到缓冲区大小）
14:30:05 [INFO]  LogTest: 无参数的日志
14:30:05 [DEBUG] LogTest: 坐标: (100, 200), 缩放: 1.50, 名称: player
日志测试完成！请检查上方输出。
```

### 实践 2：添加日志级别过滤功能

目前我们的日志系统输出所有级别的日志。在实际项目中，发布版本通常只输出 Warn 和 Error。尝试添加一个全局过滤功能：

```cpp
// 在 log.h 中添加：
PAL_API void LogSetMinLevel(LogLevel minLevel);

// 在每个平台的 log.cpp 中添加：
static LogLevel g_minLevel = LogLevel::Debug;  // 默认输出所有级别

void LogSetMinLevel(LogLevel minLevel) {
    g_minLevel = minLevel;
}

void Log(LogLevel level, const char* fmt, ...) {
    // 在函数最开头添加这行：
    if (level < g_minLevel) return;  // 低于最低级别的日志直接跳过

    // ... 后续代码不变 ...
}
```

测试代码：

```cpp
// tests/test_log_filter.cpp — 日志过滤测试
#include "pal/log.h"
#include <cstdio>

int main() {
    pal::LogInit("FilterTest");

    printf("--- 默认级别（Debug，输出所有）---\n");
    pal::Log(pal::LogLevel::Debug, "调试信息");
    pal::Log(pal::LogLevel::Info,  "一般信息");
    pal::Log(pal::LogLevel::Warn,  "警告信息");
    pal::Log(pal::LogLevel::Error, "错误信息");

    printf("\n--- 设置最低级别为 Warn ---\n");
    pal::LogSetMinLevel(pal::LogLevel::Warn);
    pal::Log(pal::LogLevel::Debug, "这条不应该出现");
    pal::Log(pal::LogLevel::Info,  "这条也不应该出现");
    pal::Log(pal::LogLevel::Warn,  "这条应该出现");
    pal::Log(pal::LogLevel::Error, "这条也应该出现");

    pal::LogShutdown();
    return 0;
}
```

> **思考题**：`enum class` 的比较运算符 `<` 在 C++ 中默认可用吗？答案是可以——`enum class` 底层是整数类型（默认 `int`），`<` 运算符对底层值有效。`Debug(0) < Info(1) < Warn(2) < Error(3)`。

---

## 对照项目源码

Mini-PAL 的日志系统是 KrKr2 日志架构的简化版。以下是关键对应关系：

相关文件：
- `cpp/core/environ/android/AndroidUtils.cpp` 第 1-50 行 — spdlog 初始化，配置 `android_sink` 输出到 logcat
- `cpp/core/environ/Platform.h` — 跨平台函数声明（`TVPPrintLog` 等）
- 根 `CMakeLists.txt` 第 13-55 行 — 平台检测和条件编译配置
- `cmake/CocosBuildHelpers.cmake` 第 200-230 行 — `cocos_use_pkg` 宏按平台链接库

| Mini-PAL | KrKr2 | 说明 |
|----------|-------|------|
| `pal::Log()` | `spdlog::info()` 等 | KrKr2 用成熟的日志库 spdlog |
| `LogLevel` 枚举 | `spdlog::level::level_enum` | 功能相同 |
| `LogInit(tag)` | `spdlog::create<sinks>(name)` | spdlog 按 logger 名称区分 |
| 直接调用 `__android_log_*` | spdlog `android_sink` | spdlog 封装了底层 API |
| CMake `if(CMAKE_SYSTEM_NAME)` | `if(ANDROID)` / `if(WINDOWS)` | KrKr2 用自定义变量名 |

---

## 本节小结

- **可变参数函数**通过 `va_list` + `va_start` / `va_end` 实现，日志函数中配合 `vsnprintf` 使用
- **Windows** 日志用 `OutputDebugStringA`（调试器可见）+ `fprintf(stderr)`（控制台可见）双通道输出
- **POSIX** 日志用 ANSI 转义序列实现彩色输出，用 `localtime_r` 添加线程安全的时间戳
- **Android** 日志直接使用 NDK 的 `__android_log_vprint`，输出到系统级的 logcat 服务
- **CMake 平台检测**通过 `CMAKE_SYSTEM_NAME` 实现，配合 `if/elseif/else` 选择源文件和链接库
- `target_link_libraries` 的 **PRIVATE/PUBLIC/INTERFACE** 控制依赖传播范围
- `find_package(Threads)` 自动查找线程库，比硬编码 `-lpthread` 更跨平台
- 永远用 `vsnprintf` 不用 `vsprintf`，永远配对 `va_start` / `va_end`

---

## 练习题与答案

### 题目 1：va_list 的安全使用

下面的代码有什么问题？如何修复？

```cpp
void PrintAll(const char* fmt, ...) {
    va_list args;
    va_start(args, fmt);

    char buf[512];
    vsprintf(buf, fmt, args);
    printf("%s\n", buf);

    // 在另一个函数中再次使用 args
    LogToFile(fmt, args);

    va_end(args);
}
```

<details>
<summary>查看答案</summary>

这段代码有 **两个** 问题：

**问题 1**：使用了 `vsprintf` 而非 `vsnprintf`，存在缓冲区溢出风险。

**问题 2**：`va_list args` 在 `vsprintf` 调用后状态是"已消费"的——在某些平台上（特别是 x86_64），`va_list` 在使用一次后就不能再直接使用了。如果要再次使用，必须先用 `va_copy` 复制一份。

修复后的代码：

```cpp
void PrintAll(const char* fmt, ...) {
    va_list args;
    va_start(args, fmt);

    // 修复 1：用 vsnprintf 限制长度
    char buf[512];
    vsnprintf(buf, sizeof(buf), fmt, args);
    printf("%s\n", buf);

    // 修复 2：用 va_copy 复制一份给第二次使用
    va_list args_copy;
    va_copy(args_copy, args);  // 创建一份独立的副本
    LogToFile(fmt, args_copy);
    va_end(args_copy);         // 副本也要 va_end

    va_end(args);
}
```

关键知识点：`va_copy`（C99/C++11）创建 `va_list` 的独立副本，允许你多次遍历可变参数。每个 `va_copy` 也需要配对的 `va_end`。

</details>

### 题目 2：CMake 平台检测调试

你在 macOS 上构建 Mini-PAL，但 CMake 报错：`不支持的平台: macOS`。配置文件中写的是：

```cmake
if(CMAKE_SYSTEM_NAME STREQUAL "macOS")
    set(PAL_PLATFORM "posix")
endif()
```

为什么报错？如何修复？

<details>
<summary>查看答案</summary>

macOS 上 `CMAKE_SYSTEM_NAME` 的值是 **`"Darwin"`**（macOS 内核的名称），不是 `"macOS"`。CMake 使用的是操作系统内核名称，不是商业品牌名。

修复：

```cmake
if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")    # Darwin，不是 macOS！
    set(PAL_PLATFORM "posix")
    set(PAL_IS_MACOS TRUE)
endif()
```

同类知识点：Windows 是 `"Windows"`（大写 W），Linux 是 `"Linux"`（大写 L），Android 是 `"Android"`。这些值都是固定的，大小写敏感。可以在 CMakeLists.txt 开头添加 `message(STATUS "系统: ${CMAKE_SYSTEM_NAME}")` 来确认。

</details>

### 题目 3：添加文件日志输出

在当前 POSIX 实现的基础上，添加一个将日志同时写入文件的功能。要求：
1. `LogInit` 时打开日志文件（`minipal.log`）
2. `Log` 时同时写入 stderr 和文件
3. `LogShutdown` 时关闭文件
4. 文件写入不要包含 ANSI 颜色代码（纯文本）

<details>
<summary>查看答案</summary>

```cpp
// src/posix/log.cpp — 增强版：支持文件输出
#include "pal/log.h"
#include <cstdio>
#include <cstdarg>
#include <ctime>

namespace pal {

static char g_tag[64] = "MiniPAL";
static FILE* g_logFile = nullptr;    // 日志文件句柄

void LogInit(const char* tag) {
    snprintf(g_tag, sizeof(g_tag), "%s", tag);

    // 打开日志文件（追加模式）
    g_logFile = fopen("minipal.log", "a");
    if (g_logFile) {
        fprintf(g_logFile, "=== 日志系统启动: %s ===\n", tag);
        fflush(g_logFile);  // 立即写入，防止崩溃丢失
    }
}

void Log(LogLevel level, const char* fmt, ...) {
    char msg[1024];
    va_list args;
    va_start(args, fmt);
    vsnprintf(msg, sizeof(msg), fmt, args);
    va_end(args);

    // 获取时间戳
    time_t now = time(nullptr);
    struct tm t;
    localtime_r(&now, &t);

    // 1. stderr 输出（带颜色）
    const char* color_prefix = "";
    switch (level) {
        case LogLevel::Debug: color_prefix = "\033[36m[DEBUG]\033[0m"; break;
        case LogLevel::Info:  color_prefix = "\033[32m[INFO] \033[0m"; break;
        case LogLevel::Warn:  color_prefix = "\033[33m[WARN] \033[0m"; break;
        case LogLevel::Error: color_prefix = "\033[31m[ERROR]\033[0m"; break;
    }
    fprintf(stderr, "%02d:%02d:%02d %s %s: %s\n",
            t.tm_hour, t.tm_min, t.tm_sec, color_prefix, g_tag, msg);

    // 2. 文件输出（不带颜色，纯文本）
    if (g_logFile) {
        const char* plain_prefix = "";
        switch (level) {
            case LogLevel::Debug: plain_prefix = "[DEBUG]"; break;
            case LogLevel::Info:  plain_prefix = "[INFO] "; break;
            case LogLevel::Warn:  plain_prefix = "[WARN] "; break;
            case LogLevel::Error: plain_prefix = "[ERROR]"; break;
        }
        fprintf(g_logFile, "%02d:%02d:%02d %s %s: %s\n",
                t.tm_hour, t.tm_min, t.tm_sec, plain_prefix, g_tag, msg);
        fflush(g_logFile);  // 每条日志都刷新，确保崩溃前的日志不丢失
    }
}

void LogShutdown() {
    if (g_logFile) {
        fprintf(g_logFile, "=== 日志系统关闭 ===\n");
        fclose(g_logFile);     // 关闭文件，释放资源
        g_logFile = nullptr;   // 防止 double-close
    }
    fflush(stderr);
}

} // namespace pal
```

要点：
- 文件使用 `"a"`（追加模式），不会覆盖之前的日志
- 每条日志后 `fflush`——性能有损耗，但保证崩溃时日志不丢失
- 文件输出使用纯文本前缀（不含 `\033[...m`），因为文件中的 ANSI 码会显示为乱码
- `LogShutdown` 中先写结束标记再关闭，并将指针置 `nullptr` 防止重复关闭

</details>

---

## 下一步

至此，Mini-PAL 的全部四个模块都已实现完毕：

1. ✅ `platform.h` — 平台检测与导出宏
2. ✅ `filesystem` — 文件系统操作（存在性检查、删除、重命名）
3. ✅ `thread` — 线程与互斥锁（PIMPL 模式）
4. ✅ `log` — 跨平台日志（可变参数、平台通道）
5. ✅ CMakeLists.txt — 自动平台选择与条件链接

下一节 [测试与总结](./05-测试与总结.md) 将编写跨平台统一测试，验证所有模块在不同平台上的正确性，并回顾整个 Mini-PAL 项目学到的跨平台开发核心模式。

