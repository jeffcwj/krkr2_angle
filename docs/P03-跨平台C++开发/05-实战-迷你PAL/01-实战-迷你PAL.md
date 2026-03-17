# 实战：从零构建迷你平台抽象层（Mini-PAL）

> **所属模块：** P03-跨平台C++开发
> **前置知识：** [操作系统差异](../01-跨平台基础/01-操作系统差异.md)、[预处理器与条件编译](../02-预处理器与条件编译/01-预处理器与条件编译.md)、[平台抽象层设计模式](../03-平台抽象层设计模式/01-平台抽象层设计模式.md)、[Android NDK与JNI](../04-Android-NDK与JNI/01-Android-NDK与JNI.md)
> **预计阅读时间：** 45 分钟

## 本节目标

读完本节后，你将能够：

1. **独立设计** 一个完整的跨平台抽象层（PAL），包含文件系统、线程、日志三大模块
2. **动手实现** 四平台（Windows/Linux/macOS/Android）的具体后端代码
3. **运用条件编译** 和 CMake 平台检测将不同实现组装为统一库
4. **理解 KrKr2 项目** 中 `Platform.h` / `ThreadIntf.h` / `typedefine.h` 的设计动机
5. **编写跨平台单元测试** 验证 PAL 在各平台上的行为一致性

## 为什么需要这个实战？

在前三节中，我们分别学习了操作系统差异、条件编译技术和平台抽象层设计模式。但这些都是"知道"层面的知识——你知道 `#ifdef _WIN32` 怎么写，知道 PIMPL 和策略模式怎么用。**"知道"和"能做"之间有巨大鸿沟**。

本节的目标是让你**亲手写出**一个麻雀虽小、五脏俱全的 PAL 库，具体包括：

| 模块 | 统一接口 | Win32 实现 | POSIX 实现 | Android 特殊处理 |
|------|----------|-----------|------------|-----------------|
| 文件系统 | `pal::FileExists`, `pal::DeleteFile`, `pal::GetFileSize` | `GetFileAttributesW` | `stat()` | `AAssetManager` 只读资产 |
| 线程 | `pal::Thread`, `pal::Mutex` | `CreateThread` / `CRITICAL_SECTION` | `pthread_create` / `pthread_mutex` | 同 POSIX（NDK 支持 pthread） |
| 日志 | `pal::Log` | `OutputDebugStringA` | `fprintf(stderr, ...)` | `__android_log_print` |

> **对照 KrKr2 源码**：这三个模块分别对应 `Platform.h`（文件操作 + 日志）、`ThreadIntf.h`（线程）和各平台 `Platform.cpp`（具体实现）。

## 项目结构设计

我们将按照 P03 第三节介绍的"头文件 + 平台子目录"模式组织代码：

```
mini-pal/
├── CMakeLists.txt          # 顶层构建文件
├── include/
│   └── pal/
│       ├── platform.h      # 平台检测宏
│       ├── filesystem.h    # 文件系统接口
│       ├── thread.h        # 线程接口
│       └── log.h           # 日志接口
├── src/
│   ├── win32/
│   │   ├── filesystem.cpp  # Windows 文件系统实现
│   │   ├── thread.cpp      # Windows 线程实现
│   │   └── log.cpp         # Windows 日志实现
│   ├── posix/
│   │   ├── filesystem.cpp  # Linux/macOS 文件系统实现
│   │   ├── thread.cpp      # POSIX 线程实现
│   │   └── log.cpp         # stderr 日志实现
│   └── android/
│       ├── filesystem.cpp  # Android 文件系统实现（AAsset）
│       └── log.cpp         # Android logcat 日志实现
├── tests/
│   ├── CMakeLists.txt
│   └── test_pal.cpp        # 跨平台统一测试
└── README.md
```

注意 Android 的线程实现复用 `posix/thread.cpp`（NDK 完整支持 POSIX 线程），但文件系统和日志有特殊实现。

### 对照 KrKr2 源码结构

```
cpp/core/environ/
├── Platform.h              # ← 对应我们的 filesystem.h + log.h
├── typedefine.h            # ← 对应我们的 platform.h（类型别名）
├── win32/Platform.cpp      # ← 对应我们的 src/win32/
├── linux/Platform.cpp      # ← 对应我们的 src/posix/
├── apple/macos/platform.mm # ← 对应我们的 src/posix/（macOS 也是 POSIX）
└── android/AndroidUtils.cpp # ← 对应我们的 src/android/
```

KrKr2 的设计理念和我们完全一致：**一个头文件声明接口，多个平台子目录提供实现**。

## 第一步：平台检测头文件

```cpp
// include/pal/platform.h — 平台检测与基础类型定义
#pragma once

// ========== 平台检测 ==========
// 检测顺序很重要：Android 也定义了 __linux__，必须先检测 Android
#if defined(__ANDROID__)
    #define PAL_PLATFORM_ANDROID 1
    #define PAL_PLATFORM_NAME "Android"
#elif defined(_WIN32) || defined(_WIN64)
    #define PAL_PLATFORM_WINDOWS 1
    #define PAL_PLATFORM_NAME "Windows"
#elif defined(__APPLE__)
    #include <TargetConditionals.h>
    #if TARGET_OS_MAC
        #define PAL_PLATFORM_MACOS 1
        #define PAL_PLATFORM_NAME "macOS"
    #endif
#elif defined(__linux__)
    #define PAL_PLATFORM_LINUX 1
    #define PAL_PLATFORM_NAME "Linux"
#else
    #error "不支持的平台"
#endif

// ========== 便利分组宏 ==========
// POSIX 系平台（Linux + macOS + Android）共享大量 API
#if defined(PAL_PLATFORM_LINUX) || defined(PAL_PLATFORM_MACOS) || \
    defined(PAL_PLATFORM_ANDROID)
    #define PAL_PLATFORM_POSIX 1
#endif

// 桌面平台（Windows + Linux + macOS）
#if defined(PAL_PLATFORM_WINDOWS) || defined(PAL_PLATFORM_LINUX) || \
    defined(PAL_PLATFORM_MACOS)
    #define PAL_PLATFORM_DESKTOP 1
#endif

// ========== 导出宏 ==========
#if defined(PAL_PLATFORM_WINDOWS)
    #ifdef PAL_BUILD_DLL
        #define PAL_API __declspec(dllexport)
    #elif defined(PAL_USE_DLL)
        #define PAL_API __declspec(dllimport)
    #else
        #define PAL_API  // 静态库不需要导出
    #endif
#else
    #define PAL_API __attribute__((visibility("default")))
#endif

#include <cstdint>
#include <string>
```

> **对照 KrKr2**：`typedefine.h` 用 `TARGET_WINDOWS` 宏做类似检测（第 5-12 行），但 KrKr2 使用 CMake 变量 `WINDOWS`/`LINUX`/`MACOS`/`ANDROID` 配合 `target_compile_definitions`，在构建系统层面定义平台宏。我们这里用编译器内置宏做纯 C++ 层面的检测，两种方式各有优劣：
>
> | 方式 | 优点 | 缺点 |
> |------|------|------|
> | 编译器内置宏 | 无需构建系统配合，头文件自包含 | 无法检测自定义平台变体 |
> | CMake 定义宏 | 灵活可控，支持交叉编译 | 依赖构建系统正确传递 |

## 第二步：文件系统接口与实现

### 2.1 统一接口

```cpp
// include/pal/filesystem.h — 文件系统统一接口
#pragma once
#include "platform.h"
#include <string>
#include <cstdint>

namespace pal {

// 文件信息结构体——对照 KrKr2 的 tTVP_stat（Platform.h 第 61-67 行）
struct FileInfo {
    uint64_t size;       // 文件大小（字节）
    uint64_t mtime;      // 最后修改时间（Unix 时间戳）
    bool     is_dir;     // 是否为目录
};

// ========== 文件系统操作 ==========

// 检查文件是否存在
// 对照 KrKr2：TVP_stat() 间接实现存在性检查
PAL_API bool FileExists(const std::string& path);

// 获取文件信息
// 对照 KrKr2：TVP_stat(const char* name, tTVP_stat& s)
PAL_API bool GetFileInfo(const std::string& path, FileInfo& info);

// 删除文件
// 对照 KrKr2：TVPDeleteFile(const std::string& filename)
PAL_API bool DeleteFile(const std::string& path);

// 重命名/移动文件
// 对照 KrKr2：TVPRenameFile(const std::string& from, const std::string& to)
PAL_API bool RenameFile(const std::string& from, const std::string& to);

// 获取应用数据目录（每个平台不同）
// 对照 KrKr2：TVPGetAppStoragePath()
PAL_API std::string GetAppDataDir();

} // namespace pal
```

### 2.2 Windows 实现

```cpp
// src/win32/filesystem.cpp — Windows 文件系统实现
#include "pal/filesystem.h"

#ifndef WIN32_LEAN_AND_MEAN
#define WIN32_LEAN_AND_MEAN  // 排除不常用的 Windows 头文件，加速编译
#endif
#include <windows.h>
#include <shlobj.h>          // SHGetFolderPathA

namespace pal {

bool FileExists(const std::string& path) {
    // GetFileAttributesA 返回 INVALID_FILE_ATTRIBUTES 表示文件不存在
    DWORD attrs = GetFileAttributesA(path.c_str());
    return (attrs != INVALID_FILE_ATTRIBUTES);
}

bool GetFileInfo(const std::string& path, FileInfo& info) {
    WIN32_FILE_ATTRIBUTE_DATA data;
    // GetFileAttributesExA 一次获取所有文件属性
    if (!GetFileAttributesExA(path.c_str(), GetFileExInfoStandard, &data)) {
        return false;  // 文件不存在或无权限
    }

    // 合并高低 32 位为 64 位文件大小
    info.size = (static_cast<uint64_t>(data.nFileSizeHigh) << 32)
                | data.nFileSizeLow;

    // FILETIME 转 Unix 时间戳：Windows epoch = 1601-01-01，Unix = 1970-01-01
    // 差值 = 116444736000000000 个 100ns 间隔
    ULARGE_INTEGER ull;
    ull.LowPart  = data.ftLastWriteTime.dwLowDateTime;
    ull.HighPart = data.ftLastWriteTime.dwHighDateTime;
    info.mtime = (ull.QuadPart - 116444736000000000ULL) / 10000000ULL;

    info.is_dir = (data.dwFileAttributes & FILE_ATTRIBUTE_DIRECTORY) != 0;
    return true;
}

bool DeleteFile(const std::string& path) {
    // 注意：Windows API 的 DeleteFileA 不能删除目录
    return ::DeleteFileA(path.c_str()) != 0;
}

bool RenameFile(const std::string& from, const std::string& to) {
    // MoveFileA：如果目标已存在会失败；MoveFileExA 可指定覆盖标志
    return ::MoveFileA(from.c_str(), to.c_str()) != 0;
}

std::string GetAppDataDir() {
    char path[MAX_PATH];
    // CSIDL_APPDATA = 用户漫游 AppData 目录（如 C:\Users\xxx\AppData\Roaming）
    if (SUCCEEDED(SHGetFolderPathA(nullptr, CSIDL_APPDATA, nullptr, 0, path))) {
        return std::string(path) + "\\MiniPAL";
    }
    return ".";  // 回退到当前目录
}

} // namespace pal
```

> **对照 KrKr2 源码**：`win32/Platform.cpp` 使用完全相同的 Win32 API 模式。例如 `TVPGetSelfUsedMemory()` 调用 `GetProcessMemoryInfo`（第 26-30 行），`TVPDeleteFile` 也是调用 `::DeleteFileA`。

### 2.3 POSIX 实现（Linux + macOS 共用）

```cpp
// src/posix/filesystem.cpp — POSIX 文件系统实现（Linux 和 macOS 共用）
#include "pal/filesystem.h"
#include <sys/stat.h>       // stat()
#include <unistd.h>         // unlink(), access()
#include <cstdio>           // rename()
#include <cstdlib>          // getenv()
#include <pwd.h>            // getpwuid()

namespace pal {

bool FileExists(const std::string& path) {
    // access() 检查文件是否存在（F_OK = 仅检查存在性）
    return access(path.c_str(), F_OK) == 0;
}

bool GetFileInfo(const std::string& path, FileInfo& info) {
    struct stat st;
    // stat() 获取文件元数据；失败返回 -1
    if (stat(path.c_str(), &st) != 0) {
        return false;
    }

    info.size  = static_cast<uint64_t>(st.st_size);
    info.mtime = static_cast<uint64_t>(st.st_mtime);  // 已经是 Unix 时间戳
    info.is_dir = S_ISDIR(st.st_mode);                 // S_ISDIR 宏判断目录
    return true;
}

bool DeleteFile(const std::string& path) {
    // unlink() 删除文件（不能删除目录，删目录用 rmdir）
    return unlink(path.c_str()) == 0;
}

bool RenameFile(const std::string& from, const std::string& to) {
    // POSIX rename() 是原子操作（同文件系统内）
    // 如果目标已存在，会被覆盖（与 Windows MoveFileA 行为不同！）
    return std::rename(from.c_str(), to.c_str()) == 0;
}

std::string GetAppDataDir() {
#if defined(PAL_PLATFORM_MACOS)
    // macOS 惯例：~/Library/Application Support/MiniPAL
    const char* home = getenv("HOME");
    if (!home) {
        // HOME 未设置时通过 getpwuid 获取
        struct passwd* pw = getpwuid(getuid());
        home = pw ? pw->pw_dir : "/tmp";
    }
    return std::string(home) + "/Library/Application Support/MiniPAL";
#else
    // Linux 惯例：遵循 XDG Base Directory 规范
    const char* xdg = getenv("XDG_DATA_HOME");
    if (xdg && xdg[0] != '\0') {
        return std::string(xdg) + "/MiniPAL";
    }
    // XDG_DATA_HOME 未设置时默认 ~/.local/share
    const char* home = getenv("HOME");
    if (!home) {
        struct passwd* pw = getpwuid(getuid());
        home = pw ? pw->pw_dir : "/tmp";
    }
    return std::string(home) + "/.local/share/MiniPAL";
#endif
}

} // namespace pal
```

### 2.4 Android 特殊实现

Android 的文件系统有一个独特之处：APK 内的资产文件（`assets/`）无法通过标准 POSIX API 访问，必须使用 NDK 的 `AAssetManager`。

```cpp
// src/android/filesystem.cpp — Android 文件系统实现
#include "pal/filesystem.h"
#include <sys/stat.h>
#include <unistd.h>
#include <cstdio>
#include <android/asset_manager.h>     // AAssetManager API
#include <android/asset_manager_jni.h>

// 全局 AAssetManager 指针，在 JNI_OnLoad 或 Activity 初始化时设置
static AAssetManager* g_asset_manager = nullptr;

namespace pal {

// 在 JNI 初始化时调用此函数设置 AssetManager
void SetAssetManager(AAssetManager* mgr) {
    g_asset_manager = mgr;
}

bool FileExists(const std::string& path) {
    // 优先检查常规文件系统（内部存储/外部存储）
    if (access(path.c_str(), F_OK) == 0) {
        return true;
    }
    // 然后检查 APK 内资产
    if (g_asset_manager) {
        AAsset* asset = AAssetManager_open(
            g_asset_manager, path.c_str(), AASSET_MODE_UNKNOWN);
        if (asset) {
            AAsset_close(asset);
            return true;
        }
    }
    return false;
}

bool GetFileInfo(const std::string& path, FileInfo& info) {
    // 先尝试常规 stat
    struct stat st;
    if (stat(path.c_str(), &st) == 0) {
        info.size  = static_cast<uint64_t>(st.st_size);
        info.mtime = static_cast<uint64_t>(st.st_mtime);
        info.is_dir = S_ISDIR(st.st_mode);
        return true;
    }
    // APK 资产不支持 mtime，只能获取大小
    if (g_asset_manager) {
        AAsset* asset = AAssetManager_open(
            g_asset_manager, path.c_str(), AASSET_MODE_UNKNOWN);
        if (asset) {
            info.size  = static_cast<uint64_t>(AAsset_getLength(asset));
            info.mtime = 0;        // APK 资产没有修改时间
            info.is_dir = false;    // APK 资产不支持目录查询
            AAsset_close(asset);
            return true;
        }
    }
    return false;
}

bool DeleteFile(const std::string& path) {
    // 只能删除常规文件系统上的文件，APK 资产不可修改
    return unlink(path.c_str()) == 0;
}

bool RenameFile(const std::string& from, const std::string& to) {
    return std::rename(from.c_str(), to.c_str()) == 0;
}

std::string GetAppDataDir() {
    // Android 内部存储路径通常由 Java 层传入
    // 典型路径：/data/data/<package_name>/files
    // 这里硬编码一个默认值，实际项目中应通过 JNI 获取
    return "/data/data/org.example.minipal/files";
}

} // namespace pal
```

> **对照 KrKr2**：`AndroidUtils.cpp`（1002 行）中大量使用 `AAssetManager` 进行资产读取，并通过 `JniHelper::callObjectMethod` 从 Java 层获取存储路径——这正是我们简化版 `GetAppDataDir()` 的完整形态。

## 第三步：线程接口与实现

### 3.1 统一接口

```cpp
// include/pal/thread.h — 线程与互斥锁接口
#pragma once
#include "platform.h"
#include <functional>
#include <string>

namespace pal {

// ========== 互斥锁 ==========
// 对照 KrKr2：项目使用 std::mutex 和 std::lock_guard（C++11 标准）
// 这里演示 OS 原生 API，帮助理解底层原理
class PAL_API Mutex {
public:
    Mutex();
    ~Mutex();

    void Lock();
    void Unlock();

    // 禁止拷贝
    Mutex(const Mutex&) = delete;
    Mutex& operator=(const Mutex&) = delete;

private:
    struct Impl;           // PIMPL 隐藏平台细节
    Impl* impl_;
};

// RAII 锁守卫——自动加锁/解锁，类似 std::lock_guard
class PAL_API LockGuard {
public:
    explicit LockGuard(Mutex& m) : mutex_(m) { mutex_.Lock(); }
    ~LockGuard() { mutex_.Unlock(); }

    LockGuard(const LockGuard&) = delete;
    LockGuard& operator=(const LockGuard&) = delete;

private:
    Mutex& mutex_;
};

// ========== 线程 ==========
// 对照 KrKr2：ThreadIntf.h 定义了 tTVPThreadPriority 和 TVPExecThreadTask
class PAL_API Thread {
public:
    using TaskFunc = std::function<void()>;

    Thread();
    ~Thread();

    // 启动线程执行 func
    bool Start(TaskFunc func);
    // 等待线程结束
    void Join();
    // 线程是否正在运行
    bool IsRunning() const;

    Thread(const Thread&) = delete;
    Thread& operator=(const Thread&) = delete;

private:
    struct Impl;
    Impl* impl_;
};

// 让出当前线程的 CPU 时间片
// 对照 KrKr2：TVPRelinquishCPU()
PAL_API void YieldThread();

// 获取 CPU 核心数
// 对照 KrKr2：TVPGetProcessorNum()
PAL_API int GetProcessorCount();

} // namespace pal
```

### 3.2 Windows 线程实现

```cpp
// src/win32/thread.cpp — Windows 线程实现
#include "pal/thread.h"
#include <windows.h>

namespace pal {

// ===== Mutex::Impl =====
struct Mutex::Impl {
    CRITICAL_SECTION cs;   // Windows 临界区——比 Mutex 内核对象更快（用户态）
};

Mutex::Mutex() : impl_(new Impl) {
    InitializeCriticalSection(&impl_->cs);
}

Mutex::~Mutex() {
    DeleteCriticalSection(&impl_->cs);
    delete impl_;
}

void Mutex::Lock()   { EnterCriticalSection(&impl_->cs); }
void Mutex::Unlock() { LeaveCriticalSection(&impl_->cs); }

// ===== Thread::Impl =====
struct Thread::Impl {
    HANDLE handle = nullptr;          // 线程句柄
    Thread::TaskFunc func;            // 要执行的函数
    bool running = false;

    // Windows 线程入口函数必须是 DWORD WINAPI (*)(LPVOID)
    static DWORD WINAPI ThreadProc(LPVOID param) {
        auto* self = static_cast<Thread::Impl*>(param);
        self->func();                 // 执行用户任务
        self->running = false;
        return 0;
    }
};

Thread::Thread() : impl_(new Impl) {}

Thread::~Thread() {
    Join();                           // 析构时确保线程已结束
    if (impl_->handle) {
        CloseHandle(impl_->handle);   // 释放线程句柄
    }
    delete impl_;
}

bool Thread::Start(TaskFunc func) {
    if (impl_->running) return false; // 防止重复启动
    impl_->func = std::move(func);
    impl_->running = true;
    impl_->handle = CreateThread(
        nullptr,                      // 默认安全属性
        0,                            // 默认栈大小
        Impl::ThreadProc,             // 线程入口
        impl_,                        // 传递 Impl 指针
        0,                            // 立即启动
        nullptr                       // 不需要线程 ID
    );
    return impl_->handle != nullptr;
}

void Thread::Join() {
    if (impl_->handle && impl_->running) {
        WaitForSingleObject(impl_->handle, INFINITE);  // 无限等待
    }
}

bool Thread::IsRunning() const { return impl_->running; }

void YieldThread() {
    // SwitchToThread() 只让给同优先级线程；Sleep(0) 让给所有就绪线程
    SwitchToThread();
}

int GetProcessorCount() {
    SYSTEM_INFO si;
    GetSystemInfo(&si);
    return static_cast<int>(si.dwNumberOfProcessors);
}

} // namespace pal
```

### 3.3 POSIX 线程实现（Linux + macOS + Android 共用）

```cpp
// src/posix/thread.cpp — POSIX 线程实现
#include "pal/thread.h"
#include <pthread.h>         // POSIX 线程
#include <sched.h>           // sched_yield()
#include <unistd.h>          // sysconf()

namespace pal {

// ===== Mutex::Impl =====
struct Mutex::Impl {
    pthread_mutex_t mtx;
};

Mutex::Mutex() : impl_(new Impl) {
    // PTHREAD_MUTEX_INITIALIZER 只能用于静态初始化
    // 动态分配的 mutex 必须调用 pthread_mutex_init
    pthread_mutex_init(&impl_->mtx, nullptr);
}

Mutex::~Mutex() {
    pthread_mutex_destroy(&impl_->mtx);
    delete impl_;
}

void Mutex::Lock()   { pthread_mutex_lock(&impl_->mtx); }
void Mutex::Unlock() { pthread_mutex_unlock(&impl_->mtx); }

// ===== Thread::Impl =====
struct Thread::Impl {
    pthread_t tid = 0;
    Thread::TaskFunc func;
    bool running = false;

    // POSIX 线程入口函数必须是 void* (*)(void*)
    static void* ThreadProc(void* param) {
        auto* self = static_cast<Thread::Impl*>(param);
        self->func();
        self->running = false;
        return nullptr;
    }
};

Thread::Thread() : impl_(new Impl) {}

Thread::~Thread() {
    Join();
    delete impl_;
}

bool Thread::Start(TaskFunc func) {
    if (impl_->running) return false;
    impl_->func = std::move(func);
    impl_->running = true;
    int ret = pthread_create(
        &impl_->tid,      // 输出线程 ID
        nullptr,           // 默认属性（可调栈大小、调度策略等）
        Impl::ThreadProc,  // 线程入口
        impl_              // 参数
    );
    return ret == 0;
}

void Thread::Join() {
    if (impl_->tid && impl_->running) {
        pthread_join(impl_->tid, nullptr);  // 等待线程结束
    }
}

bool Thread::IsRunning() const { return impl_->running; }

void YieldThread() {
    // 对照 KrKr2：linux/Platform.cpp 第 56 行 sched_yield()
    sched_yield();
}

int GetProcessorCount() {
    // _SC_NPROCESSORS_ONLN = 当前在线的 CPU 核心数
    return static_cast<int>(sysconf(_SC_NPROCESSORS_ONLN));
}

} // namespace pal
```

> **关键平台差异**：
>
> | 特性 | Windows | POSIX |
> |------|---------|-------|
> | 线程入口签名 | `DWORD WINAPI f(LPVOID)` | `void* f(void*)` |
> | 互斥锁 | `CRITICAL_SECTION`（用户态快锁） | `pthread_mutex_t` |
> | 等待线程结束 | `WaitForSingleObject` | `pthread_join` |
> | 让出时间片 | `SwitchToThread()` | `sched_yield()` |
> | 获取核心数 | `GetSystemInfo` | `sysconf(_SC_NPROCESSORS_ONLN)` |

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

## 第六步：跨平台统一测试

```cpp
// tests/test_pal.cpp — 跨平台统一测试
#include "pal/filesystem.h"
#include "pal/thread.h"
#include "pal/log.h"
#include <cassert>
#include <fstream>
#include <atomic>

// 简易测试宏
#define TEST(name) \
    static void test_##name(); \
    struct Register_##name { \
        Register_##name() { test_##name(); } \
    } reg_##name; \
    static void test_##name()

#define ASSERT_TRUE(x)  do { if (!(x)) { \
    pal::Log(pal::LogLevel::Error, "断言失败: %s (行 %d)", #x, __LINE__); \
    assert(false); \
} } while(0)

#define ASSERT_FALSE(x) ASSERT_TRUE(!(x))
#define ASSERT_EQ(a, b) ASSERT_TRUE((a) == (b))

// ========== 文件系统测试 ==========
TEST(file_exists) {
    // 创建临时文件
    const std::string path = "test_temp_file.txt";
    {
        std::ofstream ofs(path);
        ofs << "hello mini-pal";
    }
    ASSERT_TRUE(pal::FileExists(path));
    ASSERT_FALSE(pal::FileExists("not_exist_file_xyz.txt"));

    // 清理
    pal::DeleteFile(path);
    ASSERT_FALSE(pal::FileExists(path));
    pal::Log(pal::LogLevel::Info, "文件存在性测试通过");
}

TEST(file_info) {
    const std::string path = "test_info_file.txt";
    {
        std::ofstream ofs(path);
        ofs << "0123456789";  // 10 字节
    }

    pal::FileInfo info;
    ASSERT_TRUE(pal::GetFileInfo(path, info));
    ASSERT_EQ(info.size, 10u);
    ASSERT_FALSE(info.is_dir);
    ASSERT_TRUE(info.mtime > 0);  // 修改时间应大于 0

    pal::DeleteFile(path);
    pal::Log(pal::LogLevel::Info, "文件信息测试通过");
}

TEST(file_rename) {
    const std::string src = "rename_src.txt";
    const std::string dst = "rename_dst.txt";
    {
        std::ofstream ofs(src);
        ofs << "rename test";
    }

    ASSERT_TRUE(pal::RenameFile(src, dst));
    ASSERT_FALSE(pal::FileExists(src));  // 原文件应消失
    ASSERT_TRUE(pal::FileExists(dst));   // 新文件应存在

    pal::DeleteFile(dst);
    pal::Log(pal::LogLevel::Info, "文件重命名测试通过");
}

// ========== 线程测试 ==========
TEST(thread_basic) {
    std::atomic<int> counter{0};

    pal::Thread t;
    t.Start([&counter]() {
        for (int i = 0; i < 100; ++i) {
            counter.fetch_add(1, std::memory_order_relaxed);
        }
    });

    t.Join();
    ASSERT_EQ(counter.load(), 100);
    ASSERT_FALSE(t.IsRunning());
    pal::Log(pal::LogLevel::Info, "基础线程测试通过");
}

TEST(mutex_protection) {
    pal::Mutex mtx;
    int shared_value = 0;
    const int iterations = 10000;

    pal::Thread t1, t2;

    auto increment = [&]() {
        for (int i = 0; i < iterations; ++i) {
            pal::LockGuard lock(mtx);
            ++shared_value;  // 被 mutex 保护
        }
    };

    t1.Start(increment);
    t2.Start(increment);
    t1.Join();
    t2.Join();

    // 如果 mutex 正常工作，结果应为 2 * iterations
    ASSERT_EQ(shared_value, 2 * iterations);
    pal::Log(pal::LogLevel::Info, "互斥锁保护测试通过");
}

TEST(processor_count) {
    int count = pal::GetProcessorCount();
    ASSERT_TRUE(count >= 1);  // 至少 1 个核心
    pal::Log(pal::LogLevel::Info, "CPU 核心数: %d", count);
}

// ========== 日志测试 ==========
TEST(log_levels) {
    pal::Log(pal::LogLevel::Debug, "调试消息: %d", 42);
    pal::Log(pal::LogLevel::Info,  "信息消息: %s", "mini-pal");
    pal::Log(pal::LogLevel::Warn,  "警告消息: 磁盘空间不足");
    pal::Log(pal::LogLevel::Error, "错误消息: 文件打开失败");
    pal::Log(pal::LogLevel::Info, "日志级别测试通过");
}

// ========== 应用数据目录测试 ==========
TEST(app_data_dir) {
    std::string dir = pal::GetAppDataDir();
    ASSERT_FALSE(dir.empty());
    pal::Log(pal::LogLevel::Info, "应用数据目录: %s", dir.c_str());
}

int main() {
    pal::LogInit("MiniPAL-Test");
    pal::Log(pal::LogLevel::Info, "===== Mini-PAL 测试开始 =====");
    pal::Log(pal::LogLevel::Info, "当前平台: %s", PAL_PLATFORM_NAME);
    pal::Log(pal::LogLevel::Info, "===== 所有测试通过 =====");
    pal::LogShutdown();
    return 0;
}
```

## 常见错误与排查

### 错误 1：Android 链接失败 `undefined reference to '__android_log_print'`

```
ld: error: undefined symbol: __android_log_print
```

**原因**：未链接 Android 的 `log` 库。

**解决方案**：在 CMakeLists.txt 中添加：
```cmake
if(CMAKE_SYSTEM_NAME STREQUAL "Android")
    target_link_libraries(minipal PRIVATE log)
endif()
```

### 错误 2：Windows 编译报 `DeleteFile` 宏冲突

```
error C2059: syntax error: '('
```

**原因**：Windows SDK 将 `DeleteFile` 定义为宏（展开为 `DeleteFileA` 或 `DeleteFileW`），与我们的 `pal::DeleteFile` 冲突。

**解决方案**：

```cpp
// 方法 1：在包含 windows.h 后取消宏定义
#include <windows.h>
#undef DeleteFile

// 方法 2：使用 WIN32_LEAN_AND_MEAN 减少宏污染
#define WIN32_LEAN_AND_MEAN
#include <windows.h>
// 但 DeleteFile 仍然会被定义，还是需要 #undef

// 方法 3（推荐）：直接调用 Win32 API 带 :: 限定
bool pal::DeleteFile(const std::string& path) {
    return ::DeleteFileA(path.c_str()) != 0;  // :: 前缀绕过宏
}
```

> **对照 KrKr2**：`Platform.h` 第 51-59 行就遇到了类似问题——`sys/stat.h` 中 `st_atime`/`st_mtime`/`st_ctime` 被定义为宏，KrKr2 直接 `#undef` 清除。

### 错误 3：POSIX 平台 `pthread_create` 未找到

```
undefined reference to `pthread_create'
```

**原因**：未链接 pthread 库。GCC 需要显式 `-lpthread`。

**解决方案**：

```cmake
# 错误做法——直接写 -lpthread，不可移植
# target_link_libraries(minipal PRIVATE pthread)

# 正确做法——使用 CMake 的 FindThreads 模块
find_package(Threads REQUIRED)
target_link_libraries(minipal PRIVATE Threads::Threads)
```

`Threads::Threads` 是 CMake 提供的 imported target，会根据平台自动选择正确的链接参数（Linux 用 `-lpthread`，macOS 可能不需要额外参数）。

## 动手实践

完成以下步骤在你的机器上构建并运行 Mini-PAL：

### 步骤 1：创建目录结构

```bash
# Linux/macOS
mkdir -p mini-pal/{include/pal,src/{win32,posix,android},tests}

# Windows (cmd)
mkdir mini-pal\include\pal mini-pal\src\win32 mini-pal\src\posix mini-pal\src\android mini-pal\tests
```

### 步骤 2：复制本节所有代码到对应文件

将上面第一步到第六步的代码分别复制到对应路径。

### 步骤 3：构建

```bash
# Windows（需要 Visual Studio 或 MinGW）
cmake -B build -G "Ninja" -DCMAKE_BUILD_TYPE=Debug
cmake --build build

# Linux
cmake -B build -G "Ninja" -DCMAKE_BUILD_TYPE=Debug
cmake --build build

# macOS
cmake -B build -G "Ninja" -DCMAKE_BUILD_TYPE=Debug
cmake --build build

# Android（通过 NDK 工具链交叉编译，不含测试）
cmake -B build-android -G "Ninja" \
    -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
    -DANDROID_ABI=arm64-v8a \
    -DANDROID_PLATFORM=android-21 \
    -DPAL_BUILD_TESTS=OFF \
    -DCMAKE_BUILD_TYPE=Debug
cmake --build build-android
```

### 步骤 4：运行测试

```bash
# 桌面平台
cd build && ctest --output-on-failure

# 预期输出（Windows 示例）：
# [INFO]  MiniPAL-Test: ===== Mini-PAL 测试开始 =====
# [INFO]  MiniPAL-Test: 当前平台: Windows
# [INFO]  MiniPAL-Test: 文件存在性测试通过
# [INFO]  MiniPAL-Test: 文件信息测试通过
# [INFO]  MiniPAL-Test: 文件重命名测试通过
# [INFO]  MiniPAL-Test: 基础线程测试通过
# [INFO]  MiniPAL-Test: 互斥锁保护测试通过
# [INFO]  MiniPAL-Test: CPU 核心数: 8
# [INFO]  MiniPAL-Test: 日志级别测试通过
# [INFO]  MiniPAL-Test: 应用数据目录: C:\Users\xxx\AppData\Roaming\MiniPAL
# [INFO]  MiniPAL-Test: ===== 所有测试通过 =====
```

### 步骤 5：扩展挑战

完成基础构建后，尝试以下扩展：

1. **添加 `pal::CreateDirectory` 函数**：Windows 用 `CreateDirectoryA`，POSIX 用 `mkdir`
2. **添加文件读写函数**：`pal::ReadFile` / `pal::WriteFile`（Android 需要兼容 AAsset）
3. **添加条件变量**：`pal::CondVar`——Windows 用 `CONDITION_VARIABLE`，POSIX 用 `pthread_cond_t`
4. **集成到 KrKr2**：尝试用 Mini-PAL 替换 `Platform.h` 中的某个函数实现

## 对照项目源码

本节 Mini-PAL 的每个模块都有对应的 KrKr2 源码实现：

| Mini-PAL 模块 | KrKr2 对应文件 | 关键函数/结构 |
|---------------|---------------|-------------|
| `pal/platform.h` | `cpp/core/environ/typedefine.h` 第 5-12 行 | `TARGET_WINDOWS` 宏、类型别名 |
| `pal/filesystem.h` | `cpp/core/environ/Platform.h` 第 33-42 行 | `TVPDeleteFile`, `TVPRenameFile`, `TVP_stat` |
| `pal/thread.h` | `cpp/core/utils/ThreadIntf.h` 第 20-43 行 | `tTVPThreadPriority`, `TVPExecThreadTask` |
| `pal/log.h` | `cpp/core/environ/Platform.h` 第 48 行 | `TVPPrintLog` |
| `src/win32/filesystem.cpp` | `cpp/core/environ/win32/Platform.cpp` 第 1-100 行 | Win32 API 调用模式 |
| `src/posix/filesystem.cpp` | `cpp/core/environ/linux/Platform.cpp` 第 1-65 行 | `/proc/meminfo` 解析、`stat()` |
| `src/android/filesystem.cpp` | `cpp/core/environ/android/AndroidUtils.cpp` 全文 | `AAssetManager` 使用 |
| `CMakeLists.txt` | `krkr2/CMakeLists.txt` 第 40-55 行 | 平台条件编译模式 |

> **设计差异**：KrKr2 使用**自由函数**（`TVPDeleteFile`），我们使用 **namespace 包裹的函数**（`pal::DeleteFile`）。两种方式各有优劣——自由函数兼容 C 接口，namespace 防止命名冲突。KrKr2 选择自由函数是因为需要兼容 TJS2 脚本引擎的 C 风格绑定。

## 本节小结

- **PAL 的核心思想**：一个头文件定义接口，多个平台子目录提供实现，CMake 按平台选择编译
- **文件系统差异**：Windows 用 `GetFileAttributesA`/`DeleteFileA`，POSIX 用 `stat`/`unlink`，Android 额外需要 `AAssetManager` 处理 APK 资产
- **线程差异**：Windows 用 `CreateThread`/`CRITICAL_SECTION`，POSIX 用 `pthread_create`/`pthread_mutex`，Android 复用 POSIX（NDK 完整支持 pthread）
- **日志差异**：Windows 用 `OutputDebugStringA`，POSIX 用 `fprintf(stderr)`（带 ANSI 颜色），Android 用 `__android_log_print`（输出到 logcat）
- **CMake 平台选择**：通过 `CMAKE_SYSTEM_NAME` 判断平台，条件添加源文件和链接库
- **宏冲突是跨平台常见痛点**：Windows SDK 的宏（`DeleteFile`、`CreateFile` 等）经常与用户代码冲突，需要 `#undef` 或 `WIN32_LEAN_AND_MEAN`
- **PIMPL 模式**在 PAL 中的实际价值：隐藏平台头文件（`<windows.h>` / `<pthread.h>`），避免头文件污染传播到使用方

## 练习题与答案

### 题目 1：为 Mini-PAL 添加 `pal::CopyFile` 函数

要求：
- 在 `filesystem.h` 中声明 `PAL_API bool CopyFile(const std::string& src, const std::string& dst);`
- 分别实现 Windows 版（使用 `::CopyFileA`）和 POSIX 版（使用 `std::ifstream` + `std::ofstream` 逐块复制）
- 需要处理源文件不存在的情况

<details>
<summary>查看答案</summary>

**filesystem.h 追加：**

```cpp
// 复制文件
// 对照 KrKr2：TVPCopyFile(const std::string& from, const std::string& to)
PAL_API bool CopyFile(const std::string& src, const std::string& dst);
```

**src/win32/filesystem.cpp 追加：**

```cpp
bool CopyFile(const std::string& src, const std::string& dst) {
    // ::CopyFileA 第三个参数 FALSE 表示覆盖已存在的目标
    return ::CopyFileA(src.c_str(), dst.c_str(), FALSE) != 0;
}
```

**src/posix/filesystem.cpp 追加：**

```cpp
#include <fstream>

bool CopyFile(const std::string& src, const std::string& dst) {
    std::ifstream in(src, std::ios::binary);
    if (!in.is_open()) {
        return false;  // 源文件不存在或无权限
    }

    std::ofstream out(dst, std::ios::binary | std::ios::trunc);
    if (!out.is_open()) {
        return false;  // 无法创建目标文件
    }

    // 4KB 缓冲区逐块复制
    char buf[4096];
    while (in.read(buf, sizeof(buf))) {
        out.write(buf, in.gcount());
    }
    // 写入最后不足 4KB 的部分
    out.write(buf, in.gcount());

    return out.good();
}
```

**测试代码：**

```cpp
TEST(file_copy) {
    const std::string src = "copy_src.txt";
    const std::string dst = "copy_dst.txt";
    {
        std::ofstream ofs(src);
        ofs << "copy test data 12345";
    }

    ASSERT_TRUE(pal::CopyFile(src, dst));
    ASSERT_TRUE(pal::FileExists(dst));

    pal::FileInfo src_info, dst_info;
    pal::GetFileInfo(src, src_info);
    pal::GetFileInfo(dst, dst_info);
    ASSERT_EQ(src_info.size, dst_info.size);  // 大小应相同

    pal::DeleteFile(src);
    pal::DeleteFile(dst);
    pal::Log(pal::LogLevel::Info, "文件复制测试通过");
}
```

</details>

### 题目 2：解释为什么 Android 的线程实现可以复用 POSIX 代码，但日志和文件系统不行？

<details>
<summary>查看答案</summary>

**线程可以复用的原因：**

Android NDK 完整实现了 POSIX 线程 API（`<pthread.h>`），包括 `pthread_create`、`pthread_mutex_init`、`pthread_join` 等。Android 的 Bionic C 库（替代 glibc）对 POSIX 线程的支持与 Linux 几乎完全一致。因此 `src/posix/thread.cpp` 无需任何修改即可在 Android 上编译运行。

**日志不能复用的原因：**

POSIX 的日志方式是输出到 `stderr`（或 `syslog`），但 Android 有自己的日志系统——logcat。用 `fprintf(stderr, ...)` 在 Android 上虽然技术上可行（输出会被丢弃或重定向），但：
1. 开发者无法在 Android Studio 的 Logcat 面板中看到输出
2. 无法按 tag 过滤
3. 无法区分日志级别（Debug/Info/Warn/Error 在 logcat 中有颜色区分）

因此必须使用 `<android/log.h>` 的 `__android_log_print` API。

**文件系统不能复用的原因：**

Android 的 APK 是一个 ZIP 压缩包，其中 `assets/` 目录下的文件**不会解压到文件系统**。标准 POSIX 的 `stat()`、`fopen()` 等 API 无法访问 APK 内的资产文件。必须通过 NDK 提供的 `AAssetManager` API 读取。对于应用的内部存储（`/data/data/<pkg>/files/`），POSIX API 可以正常使用，但路径需要从 Java 层获取。

**总结规律**：判断能否复用 POSIX 实现的关键是看**Android NDK 是否完整支持该 POSIX API**：
- `pthread` → 完整支持 ✅ → 可复用
- `stdio`/`stat` → 部分支持（不含 APK 资产）→ 需要额外 Android 实现
- `syslog` → 不支持（Android 用 logcat）→ 需要完全重写

</details>

### 题目 3：修改 CMakeLists.txt，使 Mini-PAL 支持编译为动态库（.dll/.so）

要求：添加一个 CMake 选项 `PAL_SHARED`，为 `ON` 时编译为 `SHARED` 库，并自动设置导出宏。

<details>
<summary>查看答案</summary>

在 `CMakeLists.txt` 中修改库目标部分：

```cmake
# ========== 库类型选项 ==========
option(PAL_SHARED "编译为动态库" OFF)

if(PAL_SHARED)
    add_library(minipal SHARED ${PAL_HEADERS} ${PAL_SOURCES})
    # 设置导出宏——编译库本身时定义 PAL_BUILD_DLL
    target_compile_definitions(minipal PRIVATE PAL_BUILD_DLL)
    # 使用库的下游代码需要定义 PAL_USE_DLL（通过 INTERFACE 传播）
    target_compile_definitions(minipal INTERFACE PAL_USE_DLL)
else()
    add_library(minipal STATIC ${PAL_HEADERS} ${PAL_SOURCES})
endif()

target_include_directories(minipal PUBLIC include)

# 动态库设置版本和 soname（Linux/macOS）
if(PAL_SHARED AND NOT PAL_PLATFORM STREQUAL "win32")
    set_target_properties(minipal PROPERTIES
        VERSION 1.0.0        # 实际文件名：libminipal.so.1.0.0
        SOVERSION 1          # 符号链接：libminipal.so.1
    )
endif()

# Windows 动态库需要设置导出所有符号或使用 __declspec
if(PAL_SHARED AND PAL_PLATFORM STREQUAL "win32")
    set_target_properties(minipal PROPERTIES
        WINDOWS_EXPORT_ALL_SYMBOLS OFF  # 不自动导出，使用 PAL_API 手动控制
    )
endif()
```

**使用方式：**

```bash
# 编译为静态库（默认）
cmake -B build -DPAL_SHARED=OFF

# 编译为动态库
cmake -B build -DPAL_SHARED=ON
```

**注意**：`platform.h` 中的 `PAL_API` 宏已经正确处理了 `PAL_BUILD_DLL` / `PAL_USE_DLL` / 默认（静态库）三种情况，所以只需在 CMake 层面设置定义即可。

</details>

## 下一步

恭喜你完成了 P03 跨平台 C++ 开发的全部内容！你现在掌握了：
- 操作系统核心差异（P03-01）
- 条件编译技术（P03-02）
- 平台抽象层设计模式（P03-03）
- Android NDK 与 JNI 开发（P03-04）
- 从零构建跨平台库的完整流程（本节）

接下来建议进入 [P04-OpenGL/OpenGL ES 基础](../../P04-OpenGL与OpenGL-ES基础/README.md)，学习图形渲染的跨平台知识——这是理解 KrKr2 渲染模块（`core/visual/`）的前置技能。
