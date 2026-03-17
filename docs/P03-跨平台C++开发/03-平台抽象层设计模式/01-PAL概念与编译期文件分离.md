# 平台抽象层（PAL）设计模式

> **所属模块：** P03-跨平台 C++ 开发
> **前置知识：** [01-操作系统差异](../01-跨平台基础/01-操作系统差异.md)、[02-预处理器与条件编译](../02-预处理器与条件编译/01-预处理器与条件编译.md)
> **预计阅读时间：** 40 分钟

## 本节目标

读完本节后，你将能够：
1. 理解平台抽象层（PAL）的核心思想：**隔离变化、统一接口**
2. 掌握三种主流 PAL 实现模式：**编译期文件分离、PIMPL、运行时多态**
3. 阅读并分析 KrKr2 的 `cpp/core/environ/` PAL 架构
4. 对比不同 PAL 模式的优劣，并根据场景选择合适方案
5. 能独立为项目添加新的平台抽象函数

---

## 什么是平台抽象层

平台抽象层（Platform Abstraction Layer，简称 PAL）是跨平台软件的核心架构模式。它的目标很简单：

> **让上层业务代码不需要知道自己运行在哪个操作系统上。**

```
┌─────────────────────────────────────┐
│         业务逻辑层（引擎核心）         │  ← 完全跨平台，不含任何 #ifdef
├─────────────────────────────────────┤
│         平台抽象层（PAL 接口）         │  ← 统一 API 声明
├──────┬──────┬──────┬────────────────┤
│Win32 │Linux │macOS │   Android      │  ← 平台特定实现
└──────┴──────┴──────┴────────────────┘
│      │      │      │
▼      ▼      ▼      ▼
Win32  POSIX  Cocoa  JNI/NDK          ← 操作系统原生 API
```

不使用 PAL 的代码：

```cpp
// ❌ 没有 PAL：业务代码中到处是 #ifdef
void saveGame(const std::string& data) {
#ifdef _WIN32
    std::wstring wpath = utf8ToWide(getSavePath());
    HANDLE hFile = CreateFileW(wpath.c_str(), GENERIC_WRITE, 0, nullptr,
                               CREATE_ALWAYS, 0, nullptr);
    DWORD written;
    WriteFile(hFile, data.c_str(), data.size(), &written, nullptr);
    CloseHandle(hFile);
#elif defined(__ANDROID__)
    JNIEnv* env = getJNIEnv();
    jclass cls = env->FindClass("com/example/Storage");
    jmethodID mid = env->GetStaticMethodID(cls, "saveFile", "(Ljava/lang/String;[B)V");
    // ... 更多 JNI 代码
#else
    std::ofstream out(getSavePath());
    out << data;
#endif
}
```

使用 PAL 的代码：

```cpp
// ✅ 有 PAL：业务代码干净清晰
void saveGame(const std::string& data) {
    // TVPWriteDataToFile 是 PAL 提供的统一接口
    // 内部自动处理 Windows UTF-16 转换、Android 存储权限等
    TVPWriteDataToFile(getSavePath(), data);
}
```

---

## 模式一：编译期文件分离（KrKr2 主要采用）

这是最简单也是最常见的 PAL 实现方式。核心思路：

1. **一个公共头文件**声明平台无关的接口
2. **每个平台一个实现文件**，放在独立目录中
3. **CMake 根据目标平台**选择编译哪些实现文件

### KrKr2 的 PAL 架构

```
cpp/core/environ/
├── Platform.h              ← 公共接口声明（73 行）
├── typedefine.h            ← 跨平台类型定义（115 行）
├── win32/
│   └── Platform.cpp        ← Windows 实现（使用 Win32 API）
├── linux/
│   └── Platform.cpp        ← Linux 实现（使用 POSIX + GTK）
├── apple/macos/
│   └── platform.mm         ← macOS 实现（使用 Cocoa/CoreFoundation）
├── android/
│   └── AndroidUtils.cpp    ← Android 实现（使用 JNI + Android NDK）
└── cocos2d/
    └── CustomFileUtils.cpp ← 跨平台高层工具（基于 PAL 函数构建）
```

### 接口层：Platform.h

KrKr2 的 `Platform.h`（位于 `cpp/core/environ/Platform.h`）声明了所有平台需要实现的函数：

```cpp
// cpp/core/environ/Platform.h（完整内容，73 行）
#pragma once
#include "tjsCommHead.h"
#include <string>
#include <vector>
#include <spdlog/spdlog.h>

// 内存信息查询
struct TVPMemoryInfo {
    unsigned long MemTotal;
    unsigned long MemFree;
    unsigned long SwapTotal;
    unsigned long SwapFree;
    unsigned long VirtualTotal;
    unsigned long VirtualUsed;
};

void TVPGetMemoryInfo(TVPMemoryInfo &m);      // 获取系统内存信息
tjs_int TVPGetSystemFreeMemory();              // 获取系统空闲内存(MB)
tjs_int TVPGetSelfUsedMemory();                // 获取进程使用内存(MB)

// 消息对话框
int TVPShowSimpleMessageBox(const ttstr &text, const ttstr &caption,
                            const std::vector<ttstr> &vecButtons);
int TVPShowSimpleMessageBox(const ttstr &text, const ttstr &caption);
int TVPShowSimpleMessageBoxYesNo(const ttstr &text, const ttstr &caption);

// 文件系统操作
std::vector<std::string> TVPGetDriverPath();   // 获取驱动器路径列表
std::vector<std::string> TVPGetAppStoragePath(); // 获取应用存储路径
bool TVPDeleteFile(const std::string &filename);
bool TVPRenameFile(const std::string &from, const std::string &to);
bool TVPCopyFile(const std::string &from, const std::string &to);

// 文件元数据
struct tTVP_stat {
    uint16_t st_mode;
    uint64_t st_size;
    uint64_t st_atime;
    uint64_t st_mtime;
    uint64_t st_ctime;
};
bool TVP_stat(const tjs_char *name, tTVP_stat &s);
bool TVP_stat(const char *name, tTVP_stat &s);

// 系统功能
std::string TVPGetPackageVersionString();
void TVPExitApplication(int code);
void TVPRelinquishCPU();                       // 让出 CPU 时间片
```

**设计要点：**
- 所有函数都是 **自由函数**（非成员函数），以 `TVP` 为前缀
- 参数使用 `std::string`（UTF-8 编码）作为跨平台字符串类型
- `tTVP_stat` 结构体使用固定宽度类型（`uint16_t`、`uint64_t`），避免平台差异
- 没有平台特定的类型（如 `HANDLE`、`jstring`）出现在接口中

### Windows 实现：win32/Platform.cpp

```cpp
// cpp/core/environ/win32/Platform.cpp（关键片段）
#include "Platform.h"
#include <windows.h>
#include <psapi.h>
#include <shellapi.h>

// 内存查询 — 使用 Win32 GlobalMemoryStatusEx
void TVPGetMemoryInfo(TVPMemoryInfo &m) {
    MEMORYSTATUSEX status;
    status.dwLength = sizeof(status);
    GlobalMemoryStatusEx(&status);
    m.MemTotal = static_cast<unsigned long>(status.ullTotalPhys / 1024);
    m.MemFree = static_cast<unsigned long>(status.ullAvailPhys / 1024);
    // ... 其他字段
}

// 获取进程使用内存 — 使用 GetProcessMemoryInfo
tjs_int TVPGetSelfUsedMemory() {
    PROCESS_MEMORY_COUNTERS pmc;
    GetProcessMemoryInfo(GetCurrentProcess(), &pmc, sizeof(pmc));
    return static_cast<tjs_int>(pmc.WorkingSetSize / (1024 * 1024));
}

// 文件删除 — 使用 SHFileOperationW（送回收站）
bool TVPDeleteFile(const std::string &filename) {
    // UTF-8 → UTF-16 转换
    std::wstring wpath = utf8_to_wstring(filename);
    wpath.push_back(L'\0'); // SHFileOperation 需要双 null 终止

    SHFILEOPSTRUCTW op = {};
    op.wFunc = FO_DELETE;
    op.pFrom = wpath.c_str();
    op.fFlags = FOF_ALLOWUNDO | FOF_NOCONFIRMATION | FOF_SILENT;
    return SHFileOperationW(&op) == 0;
}

// TVP_stat — 使用 _wstat64
bool TVP_stat(const char *name, tTVP_stat &s) {
    std::wstring wname = utf8_to_wstring(name);
    struct _stat64 st;
    if (_wstat64(wname.c_str(), &st) != 0) return false;
    s.st_mode = st.st_mode;
    s.st_size = st.st_size;
    s.st_atime = st.st_atime;
    s.st_mtime = st.st_mtime;
    s.st_ctime = st.st_ctime;
    return true;
}
```

### Linux 实现：linux/Platform.cpp

```cpp
// cpp/core/environ/linux/Platform.cpp（关键片段）
#include "Platform.h"
#include <fstream>
#include <sys/stat.h>
#include <sys/sysinfo.h>
#include <unistd.h>

// 内存查询 — 读取 /proc/meminfo
void TVPGetMemoryInfo(TVPMemoryInfo &m) {
    struct sysinfo si;
    sysinfo(&si);
    m.MemTotal = si.totalram * si.mem_unit / 1024;
    m.MemFree = si.freeram * si.mem_unit / 1024;
    m.SwapTotal = si.totalswap * si.mem_unit / 1024;
    m.SwapFree = si.freeswap * si.mem_unit / 1024;
}

// 文件删除 — 使用 POSIX unlink
bool TVPDeleteFile(const std::string &filename) {
    return unlink(filename.c_str()) == 0;
}

// TVP_stat — 使用 POSIX stat
bool TVP_stat(const char *name, tTVP_stat &s) {
    struct stat st;
    if (stat(name, &st) != 0) return false;
    s.st_mode = st.st_mode;
    s.st_size = st.st_size;
    s.st_atime = st.st_atime;
    s.st_mtime = st.st_mtime;
    s.st_ctime = st.st_ctime;
    return true;
}
```

### CMake 选择机制

```cmake
# cpp/core/environ/CMakeLists.txt（简化版）
# 根据平台变量选择编译哪些源文件

set(COMMON_SOURCES
    Platform.h
    typedefine.h
    cocos2d/AppDelegate.cpp
    cocos2d/CustomFileUtils.cpp
    # ... 其他公共源文件
)

if(WINDOWS)
    set(PLATFORM_SOURCES win32/Platform.cpp)
elseif(LINUX)
    set(PLATFORM_SOURCES linux/Platform.cpp)
elseif(MACOS)
    set(PLATFORM_SOURCES apple/macos/platform.mm)
elseif(ANDROID)
    set(PLATFORM_SOURCES android/AndroidUtils.cpp)
endif()

add_library(core_environ_module OBJECT
    ${COMMON_SOURCES}
    ${PLATFORM_SOURCES}
)
```

### 优缺点分析

| 方面 | 优点 | 缺点 |
|------|------|------|
| 复杂度 | 最简单，无需额外设计模式 | 函数签名必须完全一致 |
| 编译效率 | 每个平台只编译需要的文件 | — |
| 可读性 | 每个文件专注一个平台 | 需要在多个文件间切换阅读 |
| 灵活性 | 可以使用任何平台 API | 不支持运行时切换实现 |
| 测试 | 可用 mock 文件替换平台实现 | 需要条件编译测试代码 |

**适用场景：** 平台差异主要在**实现细节**而非接口层面，且不需要运行时切换。KrKr2 的绝大部分平台抽象都用这种模式。

---

