## 第四步：日志接口与实现

### 4.1 统一接口

```cpp
// include/pal/log.h — 跨平台日志接口
#pragma once
#include "platform.h"

namespace pal {

// 日志级别枚举
enum class LogLevel {
    Debug,    // 调试信息（发布版不输出）
    Info,     // 一般信息
    Warn,     // 警告
    Error     // 错误
};

// 初始化日志系统（某些平台需要准备工作）
PAL_API void LogInit(const char* tag);

// 输出日志
// 对照 KrKr2：TVPPrintLog(const char* str)
PAL_API void Log(LogLevel level, const char* fmt, ...);

// 关闭日志系统
PAL_API void LogShutdown();

} // namespace pal
```

### 4.2 Windows 日志实现

```cpp
// src/win32/log.cpp — Windows 日志实现
#include "pal/log.h"
#include <windows.h>
#include <cstdio>
#include <cstdarg>

namespace pal {

static char g_tag[64] = "MiniPAL";

void LogInit(const char* tag) {
    // 保存标签供后续使用
    snprintf(g_tag, sizeof(g_tag), "%s", tag);
}

void Log(LogLevel level, const char* fmt, ...) {
    // 格式化用户消息
    char msg[1024];
    va_list args;
    va_start(args, fmt);
    vsnprintf(msg, sizeof(msg), fmt, args);
    va_end(args);

    // 添加级别前缀
    const char* prefix = "";
    switch (level) {
        case LogLevel::Debug: prefix = "[DEBUG]"; break;
        case LogLevel::Info:  prefix = "[INFO] "; break;
        case LogLevel::Warn:  prefix = "[WARN] "; break;
        case LogLevel::Error: prefix = "[ERROR]"; break;
    }

    // 组装完整日志行
    char full[1200];
    snprintf(full, sizeof(full), "%s %s: %s\n", prefix, g_tag, msg);

    // 同时输出到调试器和控制台
    OutputDebugStringA(full);  // Visual Studio "输出"窗口可见
    fprintf(stderr, "%s", full);
}

void LogShutdown() {
    // Windows 无需特殊清理
}

} // namespace pal
```

### 4.3 POSIX 日志实现（Linux + macOS）

```cpp
// src/posix/log.cpp — POSIX 日志实现（stderr 输出）
#include "pal/log.h"
#include <cstdio>
#include <cstdarg>
#include <ctime>

namespace pal {

static char g_tag[64] = "MiniPAL";

void LogInit(const char* tag) {
    snprintf(g_tag, sizeof(g_tag), "%s", tag);
}

void Log(LogLevel level, const char* fmt, ...) {
    char msg[1024];
    va_list args;
    va_start(args, fmt);
    vsnprintf(msg, sizeof(msg), fmt, args);
    va_end(args);

    const char* prefix = "";
    switch (level) {
        case LogLevel::Debug: prefix = "\033[36m[DEBUG]\033[0m"; break; // 青色
        case LogLevel::Info:  prefix = "\033[32m[INFO] \033[0m"; break; // 绿色
        case LogLevel::Warn:  prefix = "\033[33m[WARN] \033[0m"; break; // 黄色
        case LogLevel::Error: prefix = "\033[31m[ERROR]\033[0m"; break; // 红色
    }

    // 添加时间戳（HH:MM:SS 格式）
    time_t now = time(nullptr);
    struct tm t;
    localtime_r(&now, &t);  // 线程安全版本（Windows 用 localtime_s）

    fprintf(stderr, "%02d:%02d:%02d %s %s: %s\n",
            t.tm_hour, t.tm_min, t.tm_sec,
            prefix, g_tag, msg);
}

void LogShutdown() {
    fflush(stderr);  // 确保所有日志已刷新
}

} // namespace pal
```

> **Linux vs macOS 差异**：两者都用 stderr，但 macOS 的 `Console.app` 也会捕获 stderr 输出。如果你需要更原生的 macOS 日志集成，可以使用 `os_log` API（`<os/log.h>`），但这超出了入门实战的范围。

### 4.4 Android 日志实现

```cpp
// src/android/log.cpp — Android logcat 日志实现
#include "pal/log.h"
#include <android/log.h>   // __android_log_print
#include <cstdarg>

namespace pal {

static char g_tag[64] = "MiniPAL";

void LogInit(const char* tag) {
    snprintf(g_tag, sizeof(g_tag), "%s", tag);
}

void Log(LogLevel level, const char* fmt, ...) {
    // 映射到 Android log 优先级
    int android_level;
    switch (level) {
        case LogLevel::Debug: android_level = ANDROID_LOG_DEBUG; break;
        case LogLevel::Info:  android_level = ANDROID_LOG_INFO;  break;
        case LogLevel::Warn:  android_level = ANDROID_LOG_WARN;  break;
        case LogLevel::Error: android_level = ANDROID_LOG_ERROR; break;
        default:              android_level = ANDROID_LOG_INFO;  break;
    }

    va_list args;
    va_start(args, fmt);
    // __android_log_vprint 直接输出到 logcat
    // 可在 Android Studio 的 Logcat 面板中按 g_tag 过滤
    __android_log_vprint(android_level, g_tag, fmt, args);
    va_end(args);
}

void LogShutdown() {
    // Android logcat 无需特殊清理
}

} // namespace pal
```

> **对照 KrKr2**：`AndroidUtils.cpp` 中多处使用 `spdlog::info()` / `spdlog::error()`，spdlog 在 Android 上可配置 `android_sink`，底层也是调用 `__android_log_write`。我们这里直接使用 NDK API 是为了展示最底层的机制。

## 第五步：CMake 构建配置

这是整个项目最关键的部分之一——如何让 CMake 根据目标平台**自动选择**正确的源文件。

```cmake
# CMakeLists.txt — 顶层构建文件
cmake_minimum_required(VERSION 3.20)
project(MiniPAL LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ========== 平台检测（CMake 层面）==========
# 对照 KrKr2：根 CMakeLists.txt 第 13-36 行的平台变量设置
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    set(PAL_PLATFORM "win32")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Android")
    set(PAL_PLATFORM "android")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set(PAL_PLATFORM "posix")       # macOS 走 POSIX 路径
    set(PAL_IS_MACOS TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(PAL_PLATFORM "posix")       # Linux 也走 POSIX 路径
    set(PAL_IS_LINUX TRUE)
else()
    message(FATAL_ERROR "不支持的平台: ${CMAKE_SYSTEM_NAME}")
endif()

message(STATUS "目标平台: ${PAL_PLATFORM} (${CMAKE_SYSTEM_NAME})")

# ========== 公共头文件 ==========
set(PAL_HEADERS
    include/pal/platform.h
    include/pal/filesystem.h
    include/pal/thread.h
    include/pal/log.h
)

# ========== 平台源文件 ==========
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
        src/posix/thread.cpp          # Android 复用 POSIX 线程实现
        src/android/log.cpp
    )
elseif(PAL_PLATFORM STREQUAL "posix")
    list(APPEND PAL_SOURCES
        src/posix/filesystem.cpp
        src/posix/thread.cpp
        src/posix/log.cpp
    )
endif()

# ========== 库目标 ==========
add_library(minipal STATIC ${PAL_HEADERS} ${PAL_SOURCES})
target_include_directories(minipal PUBLIC include)

# 平台特定链接库
if(PAL_PLATFORM STREQUAL "win32")
    # Windows 需要链接这些系统库
    target_link_libraries(minipal PRIVATE psapi shell32)
elseif(PAL_PLATFORM STREQUAL "android")
    # Android 需要链接 log 库和 android 库
    target_link_libraries(minipal PRIVATE log android)
elseif(PAL_PLATFORM STREQUAL "posix")
    # POSIX 平台需要 pthread
    find_package(Threads REQUIRED)
    target_link_libraries(minipal PRIVATE Threads::Threads)
endif()

# ========== 测试（可选）==========
option(PAL_BUILD_TESTS "构建测试" ON)
if(PAL_BUILD_TESTS AND NOT CMAKE_SYSTEM_NAME STREQUAL "Android")
    enable_testing()
    add_subdirectory(tests)
endif()
```

> **对照 KrKr2**：`cmake/CocosBuildHelpers.cmake` 中的 `cocos_use_pkg` 宏（第 200-230 行）做了类似的事——根据平台有条件地链接不同库。KrKr2 的根 CMakeLists.txt 第 40-55 行使用 `if(ANDROID)` / `if(WINDOWS)` 添加平台特定源文件。

### 测试 CMakeLists

```cmake
# tests/CMakeLists.txt
add_executable(test_pal test_pal.cpp)
target_link_libraries(test_pal PRIVATE minipal)
add_test(NAME test_pal COMMAND test_pal)
```

