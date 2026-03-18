# PAL 概念与编译期文件分离

> **所属模块：** P03-跨平台 C++ 开发
> **前置知识：** [02-预处理器与条件编译](../02-预处理器与条件编译/)（`#ifdef`、CMake `target_compile_definitions`、文件分离策略）
> **预计阅读时间：** 40 分钟

## 本节目标

读完本节后，你将能够：
1. 理解平台抽象层（PAL）的核心思想——**隔离变化、统一接口**，以及为什么大型跨平台项目必须使用 PAL
2. 设计一个合格的 PAL 接口：函数签名如何做到平台无关、参数类型如何选择
3. 掌握"编译期文件分离"这一最常用的 PAL 实现模式，并能从零实现一个完整的例子
4. 阅读并分析 KrKr2 的 `cpp/core/environ/` PAL 架构，理解其设计决策
5. 对比编译期文件分离与 `#ifdef` 内联的优劣，判断何时该用哪种方式

---

## 为什么需要平台抽象层

### 场景：你的代码越来越像意大利面

假设你正在开发一个游戏引擎，最初只支持 Windows。代码很简洁：

```cpp
// save_game.cpp — 最初版本：只支持 Windows，干净利落
#include <windows.h>
#include <string>

void saveGame(const std::string& data) {
    // 获取存档路径：%APPDATA%\MyGame\save.dat
    char appdata[MAX_PATH];
    SHGetFolderPathA(nullptr, CSIDL_APPDATA, nullptr, 0, appdata);
    std::string path = std::string(appdata) + "\\MyGame\\save.dat";

    // 写入文件
    HANDLE hFile = CreateFileA(path.c_str(), GENERIC_WRITE, 0,
                               nullptr, CREATE_ALWAYS, 0, nullptr);
    DWORD written;
    WriteFile(hFile, data.c_str(), (DWORD)data.size(), &written, nullptr);
    CloseHandle(hFile);
}
```

后来老板说："我们要支持 Linux！" 于是你加了 `#ifdef`：

```cpp
void saveGame(const std::string& data) {
#ifdef _WIN32
    char appdata[MAX_PATH];
    SHGetFolderPathA(nullptr, CSIDL_APPDATA, nullptr, 0, appdata);
    std::string path = std::string(appdata) + "\\MyGame\\save.dat";
    HANDLE hFile = CreateFileA(path.c_str(), GENERIC_WRITE, 0,
                               nullptr, CREATE_ALWAYS, 0, nullptr);
    DWORD written;
    WriteFile(hFile, data.c_str(), (DWORD)data.size(), &written, nullptr);
    CloseHandle(hFile);
#elif defined(__linux__)
    std::string path = std::string(getenv("HOME")) + "/.mygame/save.dat";
    mkdir((std::string(getenv("HOME")) + "/.mygame").c_str(), 0755);
    int fd = open(path.c_str(), O_WRONLY | O_CREAT | O_TRUNC, 0644);
    write(fd, data.c_str(), data.size());
    close(fd);
#endif
}
```

然后是 macOS、然后是 Android⋯⋯一个 `saveGame` 函数变成了 80 行。更糟糕的是，你的项目中有 **50 个**类似的函数，每个都有四平台的 `#ifdef` 分支。

**结果**：
- 50 个函数 × 4 个平台 = 200 段平台特定代码散落在项目各处
- 修改一个平台的存档逻辑，需要在 50 个文件中搜索 `#ifdef _WIN32`
- 新加一个平台（比如 iOS），需要在 50 个文件中各加一个 `#elif`
- 每个 Code Review 都要在一堆 `#ifdef` 中找到当前平台的代码

这就是没有 PAL 的跨平台项目的典型困境。PAL 的解决方案是：

> **把"平台相关的代码"集中到一个专门的层中，上层业务代码只调用统一的接口。**

### PAL 的三层架构

```
┌─────────────────────────────────────┐
│         业务逻辑层（引擎核心）         │  ← 完全跨平台，不含任何 #ifdef
│   例：存档管理、脚本执行、渲染调度       │
├─────────────────────────────────────┤
│         平台抽象层（PAL 接口）         │  ← 统一的函数声明（.h 文件）
│   例：TVPWriteDataToFile()           │
├──────┬──────┬──────┬────────────────┤
│Win32 │Linux │macOS │   Android      │  ← 平台特定实现（各自的 .cpp）
│      │      │      │                │
└──────┴──────┴──────┴────────────────┘
  │      │      │      │
  ▼      ▼      ▼      ▼
 Win32  POSIX  Cocoa  JNI/NDK          ← 操作系统原生 API
```

有了 PAL，`saveGame` 变成了这样：

```cpp
// ✅ 有 PAL：业务代码干净清晰，没有任何 #ifdef
void saveGame(const std::string& data) {
    std::string path = TVPGetAppStoragePath()[0] + "/save.dat";
    TVPWriteDataToFile(path, data);
}
```

所有平台差异被"下推"到 PAL 层。业务代码的开发者**根本不需要知道** Windows 用 `CreateFileW`、Linux 用 `open`、Android 用 JNI。

### PAL 的核心设计原则

| 原则 | 说明 | 例子 |
|------|------|------|
| **接口平台无关** | PAL 的 `.h` 文件中不能出现任何平台特定的类型 | ❌ `HANDLE`、`jstring` — ✅ `std::string`、`int` |
| **实现平台隔离** | 每个平台的实现互相看不到 | Win32 的 `.cpp` 不会包含 `<unistd.h>` |
| **编译期选择** | 构建系统决定编译哪个实现，而不是运行时判断 | CMake 的 `if(WINDOWS)` |
| **统一数据类型** | 参数和返回值使用跨平台类型 | `uint64_t` 而非 `DWORD`；`std::string`（UTF-8）而非 `std::wstring` |
| **无泄漏原则** | 平台细节不能"泄漏"到接口中 | ❌ 返回 `FILETIME` — ✅ 返回 `int64_t`（Unix 时间戳） |

---

## 编译期文件分离：最常用的 PAL 模式

### 基本结构

编译期文件分离是实现 PAL 最简单、最直接的方式。核心思路：

1. **一个公共头文件**（`.h`）声明平台无关的接口
2. **每个平台一个实现文件**（`.cpp`），放在独立的目录或用平台后缀命名
3. **CMake 根据目标平台**只编译对应的实现文件

```
src/platform/
├── platform.h              ← 接口声明（所有平台共用）
├── platform_common.cpp     ← 平台无关的公共实现（如果有的话）
├── win32/
│   └── platform.cpp        ← Windows 实现
├── linux/
│   └── platform.cpp        ← Linux 实现
├── macos/
│   └── platform.mm         ← macOS 实现（.mm 因为用 Objective-C++）
└── android/
    └── platform.cpp        ← Android 实现
```

### 完整示例：跨平台文件系统 PAL

让我们从零实现一个迷你文件系统 PAL，支持四个平台。

**步骤 1：定义接口**

```cpp
// filesystem_pal.h — 跨平台文件系统接口
#pragma once

#include <cstdint>
#include <string>
#include <vector>

namespace pal {

// 文件元信息
struct FileInfo {
    uint64_t size;        // 文件大小（字节）
    int64_t  modTime;     // 最后修改时间（Unix 时间戳，秒）
    bool     isDirectory; // 是否为目录
};

// 获取文件信息
// 返回 true 表示成功，失败时 info 不修改
bool getFileInfo(const std::string& path, FileInfo& info);

// 列出目录内容
// 返回目录中所有条目的名称（不含 "." 和 ".."）
std::vector<std::string> listDirectory(const std::string& dirPath);

// 创建目录（含所有父目录）
// 类似 mkdir -p，已存在时返回 true
bool createDirectories(const std::string& path);

// 删除文件
bool deleteFile(const std::string& path);

// 获取应用数据目录
// Windows: %APPDATA%/AppName
// Linux:   ~/.local/share/AppName
// macOS:   ~/Library/Application Support/AppName
// Android: /data/data/包名/files
std::string getAppDataPath(const std::string& appName);

} // namespace pal
```

**设计要点解析**：

- **命名空间 `pal`**：避免与标准库或其他库的函数名冲突
- **`std::string` 使用 UTF-8 编码**：这是跨平台字符串的最佳选择（Windows 实现内部转为 UTF-16）
- **`int64_t` 表示时间戳**：而不是平台特定的 `FILETIME`（Windows）或 `time_t`（大小因平台而异）
- **`uint64_t` 表示文件大小**：而不是 `DWORD`（Windows，只有 32 位）或 `off_t`（Linux，大小不固定）
- **`bool` 返回值 + 输出参数**：简洁的错误处理模式，不依赖平台特定的错误码

**步骤 2：实现 Windows 版本**

```cpp
// win32/filesystem_pal.cpp — Windows 实现
#include "filesystem_pal.h"

#include <windows.h>
#include <shlobj.h>  // SHGetKnownFolderPath
#include <codecvt>   // UTF-8 ↔ UTF-16 转换

namespace {

// 辅助函数：UTF-8 → UTF-16
std::wstring utf8ToWide(const std::string& utf8) {
    if (utf8.empty()) return {};
    int len = MultiByteToWideChar(CP_UTF8, 0, utf8.c_str(),
                                  (int)utf8.size(), nullptr, 0);
    std::wstring wide(len, L'\0');
    MultiByteToWideChar(CP_UTF8, 0, utf8.c_str(),
                        (int)utf8.size(), &wide[0], len);
    return wide;
}

// 辅助函数：UTF-16 → UTF-8
std::string wideToUtf8(const std::wstring& wide) {
    if (wide.empty()) return {};
    int len = WideCharToMultiByte(CP_UTF8, 0, wide.c_str(),
                                  (int)wide.size(), nullptr, 0,
                                  nullptr, nullptr);
    std::string utf8(len, '\0');
    WideCharToMultiByte(CP_UTF8, 0, wide.c_str(),
                        (int)wide.size(), &utf8[0], len,
                        nullptr, nullptr);
    return utf8;
}

// 辅助函数：FILETIME → Unix 时间戳
int64_t fileTimeToUnix(const FILETIME& ft) {
    // FILETIME: 1601-01-01 起的 100 纳秒间隔
    // Unix:     1970-01-01 起的秒数
    ULARGE_INTEGER uli;
    uli.LowPart = ft.dwLowDateTime;
    uli.HighPart = ft.dwHighDateTime;
    return static_cast<int64_t>((uli.QuadPart - 116444736000000000ULL)
                                / 10000000ULL);
}

} // anonymous namespace

namespace pal {

bool getFileInfo(const std::string& path, FileInfo& info) {
    WIN32_FILE_ATTRIBUTE_DATA data;
    std::wstring wpath = utf8ToWide(path);
    if (!GetFileAttributesExW(wpath.c_str(),
                              GetFileExInfoStandard, &data)) {
        return false;
    }
    info.size = (static_cast<uint64_t>(data.nFileSizeHigh) << 32)
                | data.nFileSizeLow;
    info.modTime = fileTimeToUnix(data.ftLastWriteTime);
    info.isDirectory = (data.dwFileAttributes
                        & FILE_ATTRIBUTE_DIRECTORY) != 0;
    return true;
}

std::vector<std::string> listDirectory(const std::string& dirPath) {
    std::vector<std::string> result;
    std::wstring pattern = utf8ToWide(dirPath) + L"\\*";

    WIN32_FIND_DATAW fd;
    HANDLE hFind = FindFirstFileW(pattern.c_str(), &fd);
    if (hFind == INVALID_HANDLE_VALUE) return result;

    do {
        std::wstring name = fd.cFileName;
        if (name != L"." && name != L"..") {
            result.push_back(wideToUtf8(name));
        }
    } while (FindNextFileW(hFind, &fd));

    FindClose(hFind);
    return result;
}

bool createDirectories(const std::string& path) {
    // SHCreateDirectoryExW 自动创建中间目录
    std::wstring wpath = utf8ToWide(path);
    int result = SHCreateDirectoryExW(nullptr, wpath.c_str(), nullptr);
    return result == ERROR_SUCCESS || result == ERROR_ALREADY_EXISTS;
}

bool deleteFile(const std::string& path) {
    return DeleteFileW(utf8ToWide(path).c_str()) != 0;
}

std::string getAppDataPath(const std::string& appName) {
    PWSTR wpath = nullptr;
    SHGetKnownFolderPath(FOLDERID_RoamingAppData, 0, nullptr, &wpath);
    std::string result = wideToUtf8(wpath);
    CoTaskMemFree(wpath);
    return result + "\\" + appName;
}

} // namespace pal
```

**步骤 3：实现 Linux 版本**

```cpp
// linux/filesystem_pal.cpp — Linux 实现
#include "filesystem_pal.h"

#include <sys/stat.h>
#include <dirent.h>
#include <unistd.h>
#include <cstdlib>   // getenv

namespace pal {

bool getFileInfo(const std::string& path, FileInfo& info) {
    struct stat st;
    if (stat(path.c_str(), &st) != 0) return false;

    info.size = static_cast<uint64_t>(st.st_size);
    info.modTime = static_cast<int64_t>(st.st_mtime);
    info.isDirectory = S_ISDIR(st.st_mode);
    return true;
}

std::vector<std::string> listDirectory(const std::string& dirPath) {
    std::vector<std::string> result;
    DIR* dir = opendir(dirPath.c_str());
    if (!dir) return result;

    struct dirent* entry;
    while ((entry = readdir(dir)) != nullptr) {
        std::string name = entry->d_name;
        if (name != "." && name != "..") {
            result.push_back(name);
        }
    }
    closedir(dir);
    return result;
}

bool createDirectories(const std::string& path) {
    // 逐级创建目录
    std::string current;
    for (char c : path) {
        current += c;
        if (c == '/') {
            mkdir(current.c_str(), 0755);  // 忽略已存在的错误
        }
    }
    return mkdir(path.c_str(), 0755) == 0 || errno == EEXIST;
}

bool deleteFile(const std::string& path) {
    return unlink(path.c_str()) == 0;
}

std::string getAppDataPath(const std::string& appName) {
    // XDG Base Directory Specification
    const char* xdgData = getenv("XDG_DATA_HOME");
    if (xdgData && xdgData[0] != '\0') {
        return std::string(xdgData) + "/" + appName;
    }
    // 默认：~/.local/share/
    const char* home = getenv("HOME");
    return std::string(home ? home : "/tmp") +
           "/.local/share/" + appName;
}

} // namespace pal
```

**步骤 4：CMake 选择机制**

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.28)
project(filesystem_pal_demo CXX)
set(CMAKE_CXX_STANDARD 17)

# 公共文件（所有平台都编译）
set(PAL_COMMON_SOURCES
    filesystem_pal.h
)

# 根据平台选择实现文件
if(WIN32)
    set(PAL_PLATFORM_SOURCES win32/filesystem_pal.cpp)
elseif(ANDROID)
    set(PAL_PLATFORM_SOURCES android/filesystem_pal.cpp)
elseif(APPLE)
    set(PAL_PLATFORM_SOURCES macos/filesystem_pal.mm)
elseif(UNIX)  # Linux（UNIX 且不是 Apple）
    set(PAL_PLATFORM_SOURCES linux/filesystem_pal.cpp)
else()
    message(FATAL_ERROR "Unsupported platform")
endif()

add_library(filesystem_pal STATIC
    ${PAL_COMMON_SOURCES}
    ${PAL_PLATFORM_SOURCES}
)

target_include_directories(filesystem_pal PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

# Windows 需要链接 Shell32（SHGetKnownFolderPath 等）
if(WIN32)
    target_link_libraries(filesystem_pal PRIVATE shell32 ole32)
endif()

# 测试程序
add_executable(pal_test main.cpp)
target_link_libraries(pal_test PRIVATE filesystem_pal)
```

**步骤 5：平台无关的测试程序**

```cpp
// main.cpp — 完全跨平台，没有任何 #ifdef
#include "filesystem_pal.h"
#include <iostream>

int main() {
    // 1. 获取应用数据目录
    std::string appDir = pal::getAppDataPath("MyGame");
    std::cout << "App data path: " << appDir << std::endl;

    // 2. 创建目录
    if (pal::createDirectories(appDir)) {
        std::cout << "Directory created (or already exists)" << std::endl;
    }

    // 3. 获取文件信息
    pal::FileInfo info;
    if (pal::getFileInfo(".", info)) {
        std::cout << "Current dir size: " << info.size << " bytes"
                  << std::endl;
        std::cout << "Is directory: "
                  << (info.isDirectory ? "yes" : "no") << std::endl;
    }

    // 4. 列出当前目录内容
    auto entries = pal::listDirectory(".");
    std::cout << "Directory entries (" << entries.size() << "):"
              << std::endl;
    for (const auto& entry : entries) {
        std::cout << "  " << entry << std::endl;
    }

    return 0;
}
```

这段测试代码**完全不知道自己运行在哪个操作系统上**——这就是 PAL 的目标。

---

## KrKr2 的 PAL 架构分析

### 目录结构

KrKr2 的平台抽象层位于 `cpp/core/environ/`：

```
cpp/core/environ/
├── Platform.h              ← 公共接口声明
├── typedefine.h            ← 跨平台类型定义
├── win32/
│   └── Platform.cpp        ← Windows 实现（Win32 API）
├── linux/
│   └── Platform.cpp        ← Linux 实现（POSIX + sysinfo）
├── apple/macos/
│   └── platform.mm         ← macOS 实现（Cocoa/CoreFoundation）
├── android/
│   └── AndroidUtils.cpp    ← Android 实现（JNI + NDK）
├── sdl/
│   └── SDLApplication.cpp  ← SDL 后端（Linux/macOS 通用）
├── cocos2d/
│   └── AppDelegate.cpp     ← Cocos2d-x 框架桥接（所有平台共用）
│   └── CustomFileUtils.cpp ← 高层文件工具（基于 PAL 构建）
└── ui/
    └── (UI 表单相关)       ← Cocos2d-x UI 层
```

### 接口层分析：Platform.h

KrKr2 的 `Platform.h` 声明了所有平台需要实现的函数，以 `TVP` 为命名前缀：

```cpp
// cpp/core/environ/Platform.h — 关键接口（简化展示）
#pragma once
#include <string>
#include <vector>

// 内存信息（跨平台结构体，使用固定宽度类型）
struct TVPMemoryInfo {
    unsigned long MemTotal;     // 总物理内存（KB）
    unsigned long MemFree;      // 空闲物理内存（KB）
    unsigned long SwapTotal;    // 总交换空间（KB）
    unsigned long SwapFree;     // 空闲交换空间（KB）
    unsigned long VirtualTotal; // 总虚拟内存（KB）
    unsigned long VirtualUsed;  // 已用虚拟内存（KB）
};

// 文件元数据（跨平台结构体）
struct tTVP_stat {
    uint16_t st_mode;   // 文件类型与权限
    uint64_t st_size;   // 文件大小（字节）
    uint64_t st_atime;  // 最后访问时间
    uint64_t st_mtime;  // 最后修改时间
    uint64_t st_ctime;  // 创建时间
};

// --- 内存查询 ---
void TVPGetMemoryInfo(TVPMemoryInfo& m);
tjs_int TVPGetSystemFreeMemory();    // 返回 MB
tjs_int TVPGetSelfUsedMemory();      // 返回 MB

// --- 消息对话框 ---
int TVPShowSimpleMessageBox(const ttstr& text, const ttstr& caption,
                            const std::vector<ttstr>& vecButtons);

// --- 文件系统 ---
std::vector<std::string> TVPGetDriverPath();
std::vector<std::string> TVPGetAppStoragePath();
bool TVPDeleteFile(const std::string& filename);
bool TVPRenameFile(const std::string& from, const std::string& to);
bool TVPCopyFile(const std::string& from, const std::string& to);
bool TVP_stat(const char* name, tTVP_stat& s);

// --- 系统功能 ---
std::string TVPGetPackageVersionString();
void TVPExitApplication(int code);
void TVPRelinquishCPU();  // 让出 CPU 时间片
```

### 设计要点

1. **`TVP` 前缀命名规范**：所有 PAL 函数都以 `TVP` 开头（源自 "T Visual Presenter"），便于在代码中搜索和识别

2. **固定宽度类型**：`tTVP_stat` 使用 `uint16_t`、`uint64_t`，而不是 POSIX 的 `mode_t`（大小因平台而异）或 Windows 的 `DWORD`

3. **UTF-8 字符串**：文件操作使用 `std::string`（UTF-8 编码），Windows 实现内部做 UTF-8 → UTF-16 转换

4. **自由函数设计**：PAL 函数是普通函数而非类的成员方法。这是因为 PAL 提供的是"服务"（获取内存信息、删除文件），而不是"对象"

### Windows 实现示例

```cpp
// cpp/core/environ/win32/Platform.cpp — 关键片段
#include "Platform.h"
#include <windows.h>
#include <psapi.h>     // GetProcessMemoryInfo
#include <shellapi.h>  // SHFileOperation

void TVPGetMemoryInfo(TVPMemoryInfo& m) {
    MEMORYSTATUSEX status;
    status.dwLength = sizeof(status);
    GlobalMemoryStatusEx(&status);
    // MEMORYSTATUSEX 返回字节数，转换为 KB
    m.MemTotal = static_cast<unsigned long>(status.ullTotalPhys / 1024);
    m.MemFree = static_cast<unsigned long>(status.ullAvailPhys / 1024);
    m.SwapTotal = static_cast<unsigned long>(status.ullTotalPageFile / 1024);
    m.SwapFree = static_cast<unsigned long>(status.ullAvailPageFile / 1024);
    m.VirtualTotal = static_cast<unsigned long>(status.ullTotalVirtual / 1024);
    m.VirtualUsed = static_cast<unsigned long>(
        (status.ullTotalVirtual - status.ullAvailVirtual) / 1024);
}

tjs_int TVPGetSelfUsedMemory() {
    PROCESS_MEMORY_COUNTERS pmc;
    GetProcessMemoryInfo(GetCurrentProcess(), &pmc, sizeof(pmc));
    return static_cast<tjs_int>(pmc.WorkingSetSize / (1024 * 1024));
}

bool TVPDeleteFile(const std::string& filename) {
    std::wstring wpath = utf8_to_wstring(filename);
    wpath.push_back(L'\0');  // SHFileOperation 需要双 null 终止

    SHFILEOPSTRUCTW op = {};
    op.wFunc = FO_DELETE;
    op.pFrom = wpath.c_str();
    op.fFlags = FOF_ALLOWUNDO | FOF_NOCONFIRMATION | FOF_SILENT;
    return SHFileOperationW(&op) == 0;
}
```

### Linux 实现示例

```cpp
// cpp/core/environ/linux/Platform.cpp — 关键片段
#include "Platform.h"
#include <sys/stat.h>
#include <sys/sysinfo.h>
#include <unistd.h>

void TVPGetMemoryInfo(TVPMemoryInfo& m) {
    struct sysinfo si;
    sysinfo(&si);
    // sysinfo 返回字节数（需要乘以 mem_unit），转换为 KB
    m.MemTotal = si.totalram * si.mem_unit / 1024;
    m.MemFree = si.freeram * si.mem_unit / 1024;
    m.SwapTotal = si.totalswap * si.mem_unit / 1024;
    m.SwapFree = si.freeswap * si.mem_unit / 1024;
}

bool TVPDeleteFile(const std::string& filename) {
    return unlink(filename.c_str()) == 0;
}

bool TVP_stat(const char* name, tTVP_stat& s) {
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

**注意对比两个实现：**
- 函数签名完全相同（都实现 `Platform.h` 中声明的接口）
- 但用的系统 API 完全不同（`GlobalMemoryStatusEx` vs `sysinfo`，`SHFileOperation` vs `unlink`）
- 调用者（业务代码）完全不需要知道这些差异

### CMake 选择机制

```cmake
# cpp/core/environ/CMakeLists.txt（简化版）

# 公共源文件（所有平台都编译）
set(COMMON_SOURCES
    Platform.h
    typedefine.h
    cocos2d/AppDelegate.cpp
    cocos2d/CustomFileUtils.cpp
)

# 根据 KrKr2 自定义的平台变量选择实现
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

> **注意**：KrKr2 使用自定义的 CMake 变量 `WINDOWS`、`LINUX`、`MACOS`、`ANDROID`（在根 `CMakeLists.txt` 中设置），而不是 CMake 内置的 `WIN32`、`UNIX` 等变量。这是因为内置变量有一些不直观的行为（如 `UNIX` 在 macOS 和 Android 上都为真）。

---

## 编译期文件分离 vs `#ifdef` 内联：如何选择

| 维度 | `#ifdef` 内联 | 编译期文件分离 |
|------|--------------|--------------|
| 适用差异量 | < 30 行 | > 30 行 |
| 文件数量 | 1 个 | N+1 个（1 头文件 + N 平台） |
| 可读性 | 差异小时可接受 | 每个文件专注一个平台 |
| 编译时间 | 所有平台代码都会被预处理 | 只编译当前平台的文件 |
| IDE 支持 | 灰色代码（非活动分支） | 每个文件都是完整的、可编辑的 |
| 新增平台 | 在多个 `#ifdef` 中添加分支 | 新增一个 `.cpp` + 修改 CMake |
| 测试 | 可用 mock 宏 | 可用 mock 文件 |
| 项目级搜索 | 不好找"某平台的实现" | 文件名/目录名就说明平台 |

**经验法则**：

- **小差异**（路径分隔符、Sleep 函数）→ `#ifdef` 内联
- **中等差异**（文件操作、线程创建）→ 封装成 PAL 函数 + 文件分离
- **大差异**（UI 框架、音频后端）→ 完整的 PAL 层 + 文件分离

---

## 动手实践

### 练习：为迷你 PAL 添加新函数

基于前面的 `filesystem_pal.h`，添加一个新的 PAL 函数：

```cpp
// 添加到 filesystem_pal.h
namespace pal {
    // 获取当前可执行文件的路径
    // Windows: C:\Program Files\MyApp\app.exe
    // Linux:   /usr/local/bin/app
    // macOS:   /Applications/MyApp.app/Contents/MacOS/app
    // Android: /data/app/com.example.app/lib/arm64/libapp.so
    std::string getExecutablePath();
}
```

请分别实现四个平台的版本：

<details>
<summary>查看 Windows 实现</summary>

```cpp
// win32/filesystem_pal.cpp 中添加
#include <windows.h>

std::string pal::getExecutablePath() {
    wchar_t wpath[MAX_PATH];
    GetModuleFileNameW(nullptr, wpath, MAX_PATH);
    // 需要 UTF-16 → UTF-8 转换
    return wideToUtf8(wpath);
}
```

</details>

<details>
<summary>查看 Linux 实现</summary>

```cpp
// linux/filesystem_pal.cpp 中添加
#include <unistd.h>
#include <climits>  // PATH_MAX

std::string pal::getExecutablePath() {
    char path[PATH_MAX];
    // /proc/self/exe 是当前进程可执行文件的符号链接
    ssize_t len = readlink("/proc/self/exe", path, sizeof(path) - 1);
    if (len <= 0) return "";
    path[len] = '\0';
    return std::string(path);
}
```

</details>

<details>
<summary>查看 macOS 实现</summary>

```cpp
// macos/filesystem_pal.mm 中添加
#include <mach-o/dyld.h>  // _NSGetExecutablePath
#include <climits>

std::string pal::getExecutablePath() {
    char path[PATH_MAX];
    uint32_t size = sizeof(path);
    if (_NSGetExecutablePath(path, &size) != 0) return "";
    // 解析符号链接得到真实路径
    char realPath[PATH_MAX];
    if (realpath(path, realPath)) {
        return std::string(realPath);
    }
    return std::string(path);
}
```

</details>

<details>
<summary>查看 Android 实现</summary>

```cpp
// android/filesystem_pal.cpp 中添加
#include <unistd.h>
#include <climits>

std::string pal::getExecutablePath() {
    // Android 与 Linux 类似，使用 /proc/self/exe
    char path[PATH_MAX];
    ssize_t len = readlink("/proc/self/exe", path, sizeof(path) - 1);
    if (len <= 0) return "";
    path[len] = '\0';
    return std::string(path);
}
```

注意：Android 上更常用的是通过 JNI 调用 `context.getApplicationInfo().nativeLibraryDir` 获取 `.so` 文件路径。上面的 `/proc/self/exe` 方法对 native executable 有效，但对 .so 库形式的应用可能返回的是 `app_process` 的路径。

</details>

---

## 对照项目源码

本节介绍的编译期文件分离模式，在 KrKr2 项目中的对应位置：

| 概念 | 项目文件 |
|------|---------|
| PAL 接口声明 | `cpp/core/environ/Platform.h` |
| Windows 实现 | `cpp/core/environ/win32/Platform.cpp` |
| Linux 实现 | `cpp/core/environ/linux/Platform.cpp` |
| macOS 实现 | `cpp/core/environ/apple/macos/platform.mm` |
| Android 实现 | `cpp/core/environ/android/AndroidUtils.cpp` |
| CMake 平台选择 | `cpp/core/environ/CMakeLists.txt` |
| 跨平台类型定义 | `cpp/core/environ/typedefine.h` |
| 高层工具（基于 PAL） | `cpp/core/environ/cocos2d/CustomFileUtils.cpp` |

---

## 本节小结

- **PAL 的核心价值**：让业务代码不包含任何 `#ifdef`，所有平台差异被"下推"到 PAL 层
- **编译期文件分离**是最常用的 PAL 模式：一个头文件声明接口，每个平台一个 `.cpp` 实现，CMake 选择编译哪个
- **接口设计原则**：使用平台无关的类型（`std::string`、`uint64_t`），不暴露平台特定类型（`HANDLE`、`jstring`）
- **KrKr2 的 PAL 层**位于 `cpp/core/environ/`，使用 `TVP` 前缀命名，按 `win32/`、`linux/`、`android/`、`apple/` 子目录组织
- **选择标准**：差异 < 30 行用 `#ifdef` 内联，> 30 行用文件分离

---

## 练习题与答案

### 题目 1：PAL 接口设计

以下 PAL 接口设计有什么问题？请指出并修复。

```cpp
// ❓ 这个接口有什么问题？
#pragma once
#include <windows.h>

struct FileInfo {
    DWORD size;
    FILETIME modTime;
};

HANDLE openFile(LPCWSTR path);
void closeFile(HANDLE h);
```

<details>
<summary>查看答案</summary>

**问题 1**：`#include <windows.h>` — PAL 头文件包含了平台特定的头文件，导致在 Linux/macOS 上编译失败。

**问题 2**：`DWORD`、`FILETIME`、`HANDLE`、`LPCWSTR` — 全是 Windows 特有类型，不能作为跨平台接口。

**问题 3**：参数使用 `LPCWSTR`（UTF-16 宽字符串）— 应该使用 `std::string`（UTF-8）。

修复后：

```cpp
#pragma once
#include <cstdint>
#include <string>

struct FileInfo {
    uint64_t size;      // 使用固定宽度类型
    int64_t  modTime;   // Unix 时间戳（秒）
};

// 不直接暴露文件句柄；改为提供高层操作
bool getFileInfo(const std::string& path, FileInfo& info);
bool readFile(const std::string& path, std::vector<uint8_t>& data);
bool writeFile(const std::string& path, const void* data, size_t size);
```

</details>

### 题目 2：添加新平台

你的项目当前支持 Windows 和 Linux。现在需要添加 macOS 支持。使用编译期文件分离模式，你需要做哪些事情？列出具体步骤。

<details>
<summary>查看答案</summary>

1. **创建 macOS 实现目录和文件**：
   ```
   mkdir src/platform/macos
   touch src/platform/macos/platform.mm  # .mm 用于 Objective-C++
   ```

2. **实现所有 PAL 接口函数**：在 `macos/platform.mm` 中实现 `platform.h` 中声明的每个函数，使用 macOS 的 Cocoa/CoreFoundation API

3. **修改 CMakeLists.txt**：
   ```cmake
   # 在平台选择部分添加 macOS
   if(WIN32)
       set(PLATFORM_SOURCES win32/platform.cpp)
   elseif(APPLE)                                       # ← 新增
       set(PLATFORM_SOURCES macos/platform.mm)          # ← 新增
   elseif(UNIX)
       set(PLATFORM_SOURCES linux/platform.cpp)
   endif()
   ```

4. **添加 macOS 特有的链接依赖**（如果需要）：
   ```cmake
   if(APPLE)
       target_link_libraries(my_pal PRIVATE
           "-framework Foundation"
           "-framework AppKit"
       )
   endif()
   ```

5. **编译测试**：
   ```bash
   cmake -B build -DCMAKE_BUILD_TYPE=Debug
   cmake --build build
   ./build/pal_test
   ```

6. **确保测试程序（`main.cpp`）无需修改** — 这是 PAL 的核心价值。如果业务代码需要修改才能支持新平台，说明 PAL 接口设计有问题。

</details>

### 题目 3：判断 PAL 模式

以下场景各适合哪种模式？（A = `#ifdef` 内联，B = 编译期文件分离，C = PIMPL，D = 运行时多态）

1. 平台差异只有 3 行：Windows 用 `Sleep(ms)`，POSIX 用 `usleep(ms * 1000)`
2. 音频后端：Windows 用 WASAPI（300 行），Linux 用 ALSA（400 行），macOS 用 CoreAudio（350 行）
3. 渲染后端需要在程序运行时根据用户设置切换（OpenGL 或 Vulkan）

<details>
<summary>查看答案</summary>

1. **A — `#ifdef` 内联**。差异只有 3 行，不值得为此创建额外的文件。
2. **B — 编译期文件分离**。每个平台 300-400 行差异，且用完全不同的 API 体系。这是最典型的编译期文件分离场景。
3. **D — 运行时多态**（虚函数 + 工厂模式）。因为需要在运行时切换实现，编译期的方案（A 和 B）无法做到。需要定义一个 `Renderer` 基类，`OpenGLRenderer` 和 `VulkanRenderer` 是派生类，在运行时根据配置创建对应的实例。

注意：PIMPL（C）也可以用于场景 2，但编译期文件分离更简单。PIMPL 更适合需要隐藏平台特定的头文件（避免污染编译依赖）的情况——下一节会详细讲解。

</details>

---

## 下一步

[02-PIMPL 与运行时多态](./02-PIMPL与运行时多态.md) — 学习另外两种 PAL 实现模式：PIMPL（指针到实现）如何隐藏平台头文件依赖，以及运行时多态如何支持在程序运行中切换实现。
