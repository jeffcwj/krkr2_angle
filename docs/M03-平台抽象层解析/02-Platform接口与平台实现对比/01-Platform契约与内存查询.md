# Platform 接口与平台实现对比

> **所属模块：** M03-平台抽象层解析
> **前置知识：** [M03 第1章 - environ 模块总览与架构](./01-environ模块总览与架构.md)、[P03 - 跨平台 C++ 开发](../P03-跨平台C++开发/README.md)
> **预计阅读时间：** 90 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 `Platform.h` 头文件定义的 **全部 PAL 接口函数**，包括内存查询、消息框、文件操作、路径发现、输入法控制等。
2. **逐函数对比** Win32、Linux、Android 三平台的实现差异，包括 API 选型、编码处理、错误恢复策略。
3. 掌握 KrKr2 PAL 层的设计模式：**C 风格头文件声明 + 平台子目录独立 .cpp 实现 + CMake 生成器表达式选择编译**。
4. 识别各平台实现中的关键陷阱（Windows UTF-16 转换、Linux `/proc` 文件系统解析、Android JNI 生命周期管理）。
5. 能够为新平台（如 macOS/iOS）添加 Platform 接口的完整实现。

---

## 1. Platform.h —— PAL 层的"契约"

`Platform.h`（`cpp/core/environ/Platform.h`，共 73 行）是 KrKr2 整个 PAL（Platform Abstraction Layer）的核心契约文件。它不包含任何实现代码，只声明了 **所有平台必须提供** 的函数签名和数据结构。

### 1.1 文件完整结构

```cpp
// cpp/core/environ/Platform.h（73 行）
#pragma once
#include "tjsCommHead.h"
#include <string>
#include <vector>
#include <spdlog/spdlog.h>

// ======== 第一组：内存信息查询 ========
struct TVPMemoryInfo {  // 所有字段单位：kB
    unsigned long MemTotal;      // 物理内存总量
    unsigned long MemFree;       // 可用物理内存
    unsigned long SwapTotal;     // 交换分区总量
    unsigned long SwapFree;      // 可用交换分区
    unsigned long VirtualTotal;  // 虚拟内存总量
    unsigned long VirtualUsed;   // 已用虚拟内存
};

void TVPGetMemoryInfo(TVPMemoryInfo &m);
tjs_int TVPGetSystemFreeMemory();  // 返回 MB
tjs_int TVPGetSelfUsedMemory();    // 返回 MB

// ======== 第二组：用户界面（消息框/输入框）========
extern "C" int TVPShowSimpleMessageBox(
    const char *text, const char *caption,
    unsigned int nButton, const char **btnText);  // C 风格，供 JNI 等调用

int TVPShowSimpleMessageBox(
    const ttstr &text, const ttstr &caption,
    const std::vector<ttstr> &vecButtons);
int TVPShowSimpleMessageBox(
    const ttstr &text, const ttstr &caption);
int TVPShowSimpleMessageBoxYesNo(
    const ttstr &text, const ttstr &caption);
int TVPShowSimpleInputBox(
    ttstr &text, const ttstr &caption,
    const ttstr &prompt,
    const std::vector<ttstr> &vecButtons);

// ======== 第三组：路径与存储 ========
std::vector<std::string> TVPGetDriverPath();
std::vector<std::string> TVPGetAppStoragePath();
bool TVPCheckStartupPath(const std::string &path);
std::string TVPGetPackageVersionString();
void TVPExitApplication(int code);
void TVPCheckMemory();
const std::string &TVPGetInternalPreferencePath();
bool TVPDeleteFile(const std::string &filename);
bool TVPRenameFile(const std::string &from, const std::string &to);
bool TVPCopyFile(const std::string &from, const std::string &to);

// ======== 第四组：输入法控制 ========
void TVPShowIME(int x, int y, int w, int h);
void TVPHideIME();

// ======== 第五组：系统工具 ========
void TVPRelinquishCPU();
void TVPPrintLog(const char *str);

// ======== 第六组：文件状态查询 ========
struct tTVP_stat {
    uint16_t st_mode;
    uint64_t st_size;
    uint64_t st_atime;
    uint64_t st_mtime;
    uint64_t st_ctime;
};

bool TVP_stat(const tjs_char *name, tTVP_stat &s);
bool TVP_stat(const char *name, tTVP_stat &s);
bool TVP_utime(const char *name, time_t modtime);

// ======== 第七组：应用间通信 ========
void TVPSendToOtherApp(const std::string &filename);
```

### 1.2 设计哲学解读

**为什么用 C 风格自由函数而非类继承？**

许多跨平台引擎使用抽象基类 + 虚函数表（vtable）来实现 PAL，例如：

```cpp
// 典型的 OOP 风格 PAL（KrKr2 没有采用）
class IPlatform {
public:
    virtual void getMemoryInfo(MemInfo &m) = 0;
    virtual int showMessageBox(...) = 0;
};
// 各平台继承并实现
class Win32Platform : public IPlatform { ... };
class LinuxPlatform : public IPlatform { ... };
```

KrKr2 选择了更简单的方案：

| 方案 | 优点 | 缺点 |
|------|------|------|
| OOP 虚函数 | 接口清晰、可运行时切换 | vtable 开销、需要工厂模式、多继承复杂 |
| **C 风格函数（KrKr2 实际方案）** | 零开销、链接期选择、简单直接 | 无法运行时切换、接口分散在头文件中 |

KrKr2 的理由：游戏引擎 **不需要** 运行时切换平台（你不会在 Windows 上切到 Android 实现），所以编译期选择足矣。CMake 生成器表达式负责在构建时挑选正确的 `.cpp`：

```cmake
# cpp/core/environ/CMakeLists.txt（第 30-55 行）
target_sources(core_environ_module PRIVATE
    # 公共源文件
    Application.cpp
    DetectCPU.cpp
    # 平台条件编译
    $<$<BOOL:${WINDOWS}>:
        win32/Platform.cpp
        win32/SystemControl.cpp
    >
    $<$<BOOL:${LINUX}>:
        linux/Platform.cpp
    >
    $<$<BOOL:${ANDROID}>:
        android/AndroidUtils.cpp
    >
)
```

### 1.3 接口函数分组速查表

| 组别 | 函数 | 用途 | 调用频率 |
|------|------|------|----------|
| 内存 | `TVPGetMemoryInfo` / `TVPGetSystemFreeMemory` / `TVPGetSelfUsedMemory` | OOM 预警、内存统计 | 每帧/定时 |
| 对话框 | `TVPShowSimpleMessageBox` (3 重载) / `TVPShowSimpleInputBox` | 错误提示、用户交互 | 偶发 |
| 路径 | `TVPGetDriverPath` / `TVPGetAppStoragePath` / `TVPCheckStartupPath` | 文件浏览器、启动路径检测 | 启动时 |
| 文件 | `TVPDeleteFile` / `TVPRenameFile` / `TVPCopyFile` / `TVP_stat` / `TVP_utime` | 存档管理、文件操作 | 频繁 |
| 输入法 | `TVPShowIME` / `TVPHideIME` | 软键盘控制（移动端为主） | 交互时 |
| 系统 | `TVPRelinquishCPU` / `TVPPrintLog` / `TVPExitApplication` | CPU 让出、日志、退出 | 每帧/偶发 |
| 通信 | `TVPSendToOtherApp` | 分享文件到外部应用 | 偶发 |

---

## 2. 内存查询：三平台逐行对比

内存信息是引擎运行时最频繁查询的 PAL 接口之一。`Application.cpp` 中的 `TVPCheckMemory()` 会定期调用它来决定是否需要释放缓存纹理。

### 2.1 Win32 实现

```cpp
// cpp/core/environ/win32/Platform.cpp 第 20-43 行
#include <windows.h>
#include <Psapi.h>
#pragma comment(lib, "psapi.lib")

tjs_int TVPGetSystemFreeMemory() {
    MEMORYSTATUS info;
    GlobalMemoryStatus(&info);             // Win32 API：填充内存状态结构
    return info.dwAvailPhys / (1024 * 1024);  // 字节 → MB
}

tjs_int TVPGetSelfUsedMemory() {
    PROCESS_MEMORY_COUNTERS info;
    GetProcessMemoryInfo(                  // 需要 psapi.lib
        GetCurrentProcess(), &info, sizeof(info));
    return info.WorkingSetSize / (1024 * 1024);  // 工作集 → MB
}

void TVPGetMemoryInfo(TVPMemoryInfo &m) {
    MEMORYSTATUS status;
    status.dwLength = sizeof(status);      // 必须设置 dwLength
    GlobalMemoryStatus(&status);

    m.MemTotal    = status.dwTotalPhys      / 1024;  // 字节 → kB
    m.MemFree     = status.dwAvailPhys      / 1024;
    m.SwapTotal   = status.dwTotalPageFile   / 1024;
    m.SwapFree    = status.dwAvailPageFile   / 1024;
    m.VirtualTotal= status.dwTotalVirtual    / 1024;
    m.VirtualUsed = (status.dwTotalVirtual
                   - status.dwAvailVirtual)  / 1024;
}
```

**关键点**：
- `MEMORYSTATUS` 是 32 位结构，在 >4GB 内存的机器上会溢出。更现代的做法是用 `GlobalMemoryStatusEx` 返回 `MEMORYSTATUSEX`（64 位字段）。KrKr2 没有升级，因为引擎本身内存占用远低于 4GB。
- `GetProcessMemoryInfo` 需要链接 `psapi.lib`，用 `#pragma comment(lib, ...)` 在源码中声明，避免手动在 CMake 中添加。

### 2.2 Linux 实现

```cpp
// cpp/core/environ/linux/Platform.cpp 第 10-53 行
#include <sys/sysinfo.h>

void TVPGetMemoryInfo(TVPMemoryInfo &m) {
    // 直接解析 /proc/meminfo 文本文件
    FILE *meminfo = fopen("/proc/meminfo", "r");
    char buffer[100] = {0};
    char *end;

    static const char
        pszMemFree[]      = "MemFree:",
        pszMemTotal[]     = "MemTotal:",
        pszSwapTotal[]    = "SwapTotal:",
        pszSwapFree[]     = "SwapFree:",
        pszVmallocTotal[] = "VmallocTotal:",
        pszVmallocUsed[]  = "VmallocUsed:";

    // 逐行读取，找到匹配的字段名就解析数值
    while (fgets(buffer, sizeof(buffer), meminfo)) {
        if (strstr(buffer, pszMemFree) == buffer) {
            // sizeof("MemFree:") 包含 '\0'，刚好跳过标签
            m.MemFree = strtol(buffer + sizeof(pszMemFree), &end, 10);
        } else if (strstr(buffer, pszMemTotal) == buffer) {
            m.MemTotal = strtol(buffer + sizeof(pszMemTotal), &end, 10);
        }
        // ... SwapTotal, SwapFree, VmallocTotal, VmallocUsed 类似
    }
    fclose(meminfo);
}
```

**关键点**：
- `/proc/meminfo` 的值已经是 **kB**，不需要额外除法，直接赋值给 `TVPMemoryInfo`（也是 kB）。
- 代码中有一个潜在 Bug：`found++` 后的 `break` 语句会导致只要碰到 SwapTotal/SwapFree/VmallocTotal/VmallocUsed 中的任何一个就立即停止循环，后续字段无法被读取。这是因为 `/proc/meminfo` 中这些字段是按固定顺序排列的，作者假设了顺序（MemTotal → MemFree → SwapTotal → SwapFree → ...），但 `break` 的放置位置不够精确。

```cpp
// Linux 的进程内存查询
tjs_int TVPGetSystemFreeMemory() {
    struct sysinfo info;
    if (sysinfo(&info) == -1) return -1;
    return (info.freeram * info.mem_unit) / (1024 * 1024);
}

tjs_int TVPGetSelfUsedMemory() {
    std::ifstream statm{"/proc/self/statm"};
    tjs_int pages = 0;
    statm >> pages;  // 第一个字段 = 总内存页数
    return (pages * sysconf(_SC_PAGESIZE)) / (1024 * 1024);
}
```

- `sysinfo()` 是 POSIX 系统调用，一次性获取全部系统信息。
- `/proc/self/statm` 是 Linux 特有的进程内存快照文件，字段以页（page）为单位。`sysconf(_SC_PAGESIZE)` 获取页大小（通常 4096 字节）。

### 2.3 Android 实现

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 46-88 行
static tjs_uint32 _lastMemoryInfoQuery = 0;
static tjs_int _availMemory, usedMemory;

static void updateMemoryInfo() {
    // 节流：3 秒内不重复查询
    if (TVPGetRoughTickCount32() - _lastMemoryInfoQuery > 3000) {
        JniMethodInfo methodInfo;
        // 调用 Java 层 KR2Activity.updateMemoryInfo()
        if (JniHelper::getStaticMethodInfo(
                methodInfo, KR2ActJavaPath,
                "updateMemoryInfo", "()V")) {
            methodInfo.env->CallStaticVoidMethod(
                methodInfo.classID, methodInfo.methodID);
            methodInfo.env->DeleteLocalRef(methodInfo.classID);
        }
        // 再分别获取 availMemory 和 usedMemory
        // ...（类似的 JNI 调用）
        _lastMemoryInfoQuery = TVPGetRoughTickCount32();
    }
}

tjs_int TVPGetSystemFreeMemory() {
    updateMemoryInfo();
    return _availMemory;  // 已缓存的值
}
```

**关键点**：
- Android **没有 `/proc/meminfo` 直读权限**（某些厂商限制了非 root 访问），所以通过 JNI 调用 Java 层的 `ActivityManager.getMemoryInfo()` 来获取。
- 引入了 **3 秒节流机制**：`TVPGetRoughTickCount32()` 差值大于 3000ms 才重新查询。这是因为 JNI 调用开销远大于直接读文件。
- 缓存值存储在模块级静态变量中（`_availMemory`、`usedMemory`），线程安全性依赖于引擎单线程主循环调用。

### 2.4 三平台对比表

| 维度 | Win32 | Linux | Android |
|------|-------|-------|---------|
| **系统 API** | `GlobalMemoryStatus` | `/proc/meminfo` + `sysinfo()` | JNI → `ActivityManager` |
| **进程内存** | `GetProcessMemoryInfo` | `/proc/self/statm` | JNI → `getUsedMemory()` |
| **单位转换** | 字节 → kB/MB | kB（原生）/ 页 → MB | Java 层返回字节 → MB |
| **缓存策略** | 无（每次实时查询） | 无（每次实时查询） | 3 秒节流缓存 |
| **性能开销** | 微秒级 | 微秒级 | 毫秒级（JNI 跨语言） |
| **额外依赖** | `psapi.lib` | `<sys/sysinfo.h>` | Cocos2d-x `JniHelper` |

---

