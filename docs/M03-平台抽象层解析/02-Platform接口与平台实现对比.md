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

## 3. 消息框与输入框：GUI 差异最大的接口

用户交互对话框是平台差异最显著的区域。Windows 有原生 Win32 API，Linux 依赖 GTK，Android 需要通过 JNI 回调 Java UI 线程。

### 3.1 Win32：MessageBoxW

```cpp
// cpp/core/environ/win32/Platform.cpp 第 144-166 行
int TVPShowSimpleMessageBox(const ttstr &text, const ttstr &caption,
                            const std::vector<ttstr> &vecButtons) {
    switch (vecButtons.size()) {
    case 1:
        MessageBoxW(0,
            text.toWString().c_str(),    // UTF-16 宽字符串
            caption.toWString().c_str(),
            /*MB_OK*/ 0);
        return 0;
    case 2:
        switch (MessageBoxW(0,
                text.toWString().c_str(),
                caption.toWString().c_str(),
                /*MB_YESNO*/ 4)) {
        case 6:  return 0;  // IDYES = 6 → 返回 0（第一个按钮）
        default: return 1;  // IDNO 等 → 返回 1（第二个按钮）
        }
    }
    return -1;
}
```

**关键点**：
- `ttstr::toWString()` 将内部 UTF-16 编码转为 `std::wstring`，与 Windows `wchar_t`（16 位）兼容。
- 硬编码了 `MB_OK = 0` 和 `MB_YESNO = 4` 的数值（不用宏），可能是为了避免头文件污染或历史遗留。
- 只支持 1-2 个按钮。3 个或更多按钮返回 -1（不支持）。

### 3.2 Linux：GTK 对话框

```cpp
// cpp/core/environ/linux/Platform.cpp 第 140-183 行
#include <gtk/gtk.h>
#include <Defer.h>

int TVPShowSimpleMessageBox(const ttstr &text, const ttstr &caption,
                            const std::vector<ttstr> &vecButtons) {
    GtkWidget *dialog = nullptr;
    DEFER({                               // 作用域退出时自动销毁
        if (dialog) gtk_widget_destroy(dialog);
    });

    switch (vecButtons.size()) {
    case 1:
        dialog = gtk_message_dialog_new(
            NULL,                          // 无父窗口
            GTK_DIALOG_MODAL,              // 模态
            GTK_MESSAGE_INFO,              // 信息图标
            GTK_BUTTONS_OK,                // OK 按钮
            "%s", text.AsStdString().c_str()
        );
        gtk_window_set_title(GTK_WINDOW(dialog),
                             caption.AsStdString().c_str());
        gtk_dialog_run(GTK_DIALOG(dialog));  // 阻塞直到用户点击
        return 0;
    case 2:
        dialog = gtk_message_dialog_new(
            NULL, GTK_DIALOG_MODAL, GTK_MESSAGE_INFO,
            GTK_BUTTONS_YES_NO,            // 是/否按钮
            "%s", text.AsStdString().c_str()
        );
        gtk_window_set_title(GTK_WINDOW(dialog),
                             caption.AsStdString().c_str());
        int result = gtk_dialog_run(GTK_DIALOG(dialog));
        return (result == GTK_RESPONSE_YES) ? 0 : 1;
    }
    return -1;
}
```

**关键点**：
- 使用 `DEFER` 宏（项目自定义，来自 `common/Defer.h`）确保对话框在任何退出路径下都被销毁，避免 GTK 资源泄漏。
- `AsStdString()` 返回 UTF-8 `std::string`，GTK 原生支持 UTF-8，不需要编码转换。
- GTK 需要运行时安装 `libgtk-3-dev`，这是 Linux 构建的额外依赖。

### 3.3 Android：JNI + 条件变量等待

Android 的消息框实现最复杂，因为 UI 必须在 **Java 主线程** 上运行，而 C++ 引擎运行在 **GL 线程**：

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 507-551 行
namespace kr2android {
    std::condition_variable MessageBoxCond;
    std::mutex MessageBoxLock;
    int MsgBoxRet = -2;  // -2 = 未响应
}

int TVPShowSimpleMessageBox(const char *pszText, const char *pszTitle,
                            unsigned int nButton, const char **btnText) {
    JniMethodInfo methodInfo;
    if (JniHelper::getStaticMethodInfo(
            methodInfo, "org/tvp/kirikiri2/KR2Activity",
            "ShowMessageBox",
            "(Ljava/lang/String;Ljava/lang/String;"
            "[Ljava/lang/String;)V")) {

        // 1. 构造 Java 字符串参数
        jstring jstrTitle = methodInfo.env->NewStringUTF(pszTitle);
        jstring jstrText  = methodInfo.env->NewStringUTF(pszText);

        // 2. 构造 Java String[] 数组
        jclass strcls = methodInfo.env->FindClass("java/lang/String");
        jobjectArray btns = methodInfo.env->NewObjectArray(
            nButton, strcls, nullptr);
        for (unsigned int i = 0; i < nButton; ++i) {
            jstring jstrBtn = methodInfo.env->NewStringUTF(btnText[i]);
            methodInfo.env->SetObjectArrayElement(btns, i, jstrBtn);
            methodInfo.env->DeleteLocalRef(jstrBtn);  // 立即释放局部引用
        }

        // 3. 调用 Java 方法（非阻塞，Java 端会弹出 AlertDialog）
        methodInfo.env->CallStaticVoidMethod(
            methodInfo.classID, methodInfo.methodID,
            jstrTitle, jstrText, btns);

        // 4. 清理局部引用
        methodInfo.env->DeleteLocalRef(jstrTitle);
        methodInfo.env->DeleteLocalRef(jstrText);
        methodInfo.env->DeleteLocalRef(btns);
        methodInfo.env->DeleteLocalRef(methodInfo.classID);

        // 5. 等待 Java 端回调设置 MsgBoxRet
        std::unique_lock<std::mutex> lk(MessageBoxLock);
        while (MsgBoxRet == -2) {
            MessageBoxCond.wait_for(lk, std::chrono::milliseconds(200));
            if (MsgBoxRet == -2) {
                TVPForceSwapBuffer();  // 保持 GL 上下文活跃
            }
        }
        return MsgBoxRet;
    }
    return -1;
}
```

**关键点**：
- **跨线程同步**：C++ 端使用 `std::condition_variable` 阻塞等待，Java 端用户点击按钮后通过 JNI 回调设置 `MsgBoxRet` 并 `notify_one()`。
- **GL 上下文保活**：等待期间每 200ms 调用 `TVPForceSwapBuffer()`（即 `eglSwapBuffers`），防止 Android 系统因 GL 线程长时间无响应而回收上下文。
- **JNI 局部引用管理**：每个 `NewStringUTF` / `NewObjectArray` 创建的对象必须 `DeleteLocalRef`，否则在循环中会超出 JNI 局部引用表上限（默认 512 个）。
- 这是一个 **经典的 C++/Java 混合编程模式**：C++ 发起请求 → Java 显示 UI → 用户操作 → Java 回调 → C++ 解除等待。

### 3.4 消息框实现对比

```
┌──────────────────────────────────────────────────────────┐
│                    TVPShowSimpleMessageBox                 │
├──────────┬───────────────┬───────────────┬────────────────┤
│          │    Win32       │    Linux      │    Android     │
├──────────┼───────────────┼───────────────┼────────────────┤
│ GUI 库   │ Win32 API     │ GTK3          │ Java AlertDlg  │
│ 线程模型 │ 同线程阻塞    │ 同线程阻塞    │ 跨线程等待     │
│ 编码     │ UTF-16(W)     │ UTF-8         │ Modified UTF-8 │
│ 清理策略 │ 无需(系统管理)│ DEFER 销毁    │ DeleteLocalRef │
│ 按钮上限 │ 2             │ 2             │ 无限(Java数组) │
│ 输入框   │ ❌ 未实现     │ ❌ 未实现     │ ✅ JNI 实现    │
└──────────┴───────────────┴───────────────┴────────────────┘
```

> **注意**：Win32 和 Linux 的 `TVPShowSimpleInputBox` 均打印了 `warn("... not implement")`，只有 Android 完整实现了输入框功能。这是因为桌面平台可以通过键盘直接在游戏窗口内输入，而移动端必须弹出系统输入法。

---

## 4. 文件操作：编码陷阱最密集的区域

文件操作函数（`TVPDeleteFile`、`TVPRenameFile`、`TVP_stat`、`TVP_utime`）看似简单，却是平台差异的重灾区，核心问题在于 **文件名编码**。

### 4.1 Win32：UTF-8 → UTF-16 手动转换

```cpp
// cpp/core/environ/win32/Platform.cpp 第 319-339 行
bool TVPDeleteFile(const std::string &filename) {
    /* 1. UTF-8 → UTF-16 */
    int len = MultiByteToWideChar(CP_UTF8, 0,
        filename.c_str(), -1, nullptr, 0);
    if (len <= 0) return false;
    std::wstring path(len, 0);
    MultiByteToWideChar(CP_UTF8, 0,
        filename.c_str(), -1, &path[0], len);
    path.pop_back();  // 去掉末尾 '\0'

    /* 2. 构造双 '\0' 结尾（SHFileOperation 要求） */
    path.push_back(L'\0');
    path.push_back(L'\0');

    /* 3. 递归删除（文件或目录均可） */
    SHFILEOPSTRUCTW op = {};
    op.wFunc  = FO_DELETE;
    op.pFrom  = path.c_str();
    op.fFlags = FOF_NOCONFIRMATION | FOF_NOERRORUI | FOF_SILENT;

    return SHFileOperationW(&op) == 0;
}
```

**关键点**：
- Windows 原生 API 使用 UTF-16（`wchar_t`），但引擎内部统一用 UTF-8 `std::string`，所以每次文件操作都需要 `MultiByteToWideChar` 转换。
- 使用 `SHFileOperationW` 而非 `_wunlink`，因为它可以**递归删除目录**。`pFrom` 必须以**双 `\0`** 结尾，这是 Shell API 的特殊要求。
- `FOF_NOCONFIRMATION | FOF_NOERRORUI | FOF_SILENT` 三个标志确保静默删除、不弹确认框。

```cpp
// Win32 的 TVPRenameFile（第 348-364 行）
bool TVPRenameFile(const std::string &from, const std::string &to) {
    // UTF-8 → UTF-16 转换（与 Delete 类似）
    // ...
    return MoveFileExW(wfrom.c_str(), wto.c_str(),
        MOVEFILE_REPLACE_EXISTING | MOVEFILE_COPY_ALLOWED) != 0;
}
```

- `MoveFileExW` 带 `MOVEFILE_REPLACE_EXISTING` 允许覆盖目标文件，带 `MOVEFILE_COPY_ALLOWED` 允许跨卷移动（先复制后删除）。

### 4.2 Linux：POSIX 原生 UTF-8

```cpp
// cpp/core/environ/linux/Platform.cpp 第 289-296 行
bool TVPDeleteFile(const std::string &filename) {
    return unlink(filename.c_str()) == 0;  // 直接调用 POSIX unlink
}

bool TVPRenameFile(const std::string &from, const std::string &to) {
    tjs_int ret = rename(from.c_str(), to.c_str());
    return !ret;
}
```

**关键点**：
- Linux 文件系统原生支持 UTF-8，不需要任何编码转换。
- `unlink` 只能删除文件，**不能删除目录**（与 Win32 的 `SHFileOperationW` 不同）。如果需要递归删除目录，需要额外实现（项目中 `TVPCreateFolders` 使用了 `std::filesystem::create_directory`，但删除没有对应的递归版本）。
- `rename` 在同一文件系统上是原子操作，但**不支持跨文件系统移动**（与 Win32 的 `MOVEFILE_COPY_ALLOWED` 不同）。

### 4.3 Android：JNI 委托 Java 文件操作

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 939-969 行
bool TVPDeleteFile(const std::string &filename) {
    JniMethodInfo methodInfo;
    if (JniHelper::getStaticMethodInfo(methodInfo,
            "org/tvp/kirikiri2/KR2Activity",
            "DeleteFile", "(Ljava/lang/String;)Z")) {
        jstring jstr = methodInfo.env->NewStringUTF(filename.c_str());
        bool ret = methodInfo.env->CallStaticBooleanMethod(
            methodInfo.classID, methodInfo.methodID, jstr);
        methodInfo.env->DeleteLocalRef(jstr);
        methodInfo.env->DeleteLocalRef(methodInfo.classID);
        return ret;
    }
    return false;
}
```

**为什么 Android 不直接用 POSIX `unlink`？**

因为 Android 从 API 29 (Android 10) 开始引入了 **Scoped Storage**（分区存储），应用无法直接通过 POSIX API 访问外部存储。必须通过 Java 的 `ContentResolver` / SAF（Storage Access Framework）来操作文件。所以 KrKr2 把所有文件操作都委托给 Java 层，由 Java 根据路径判断使用直接 I/O 还是 SAF。

### 4.4 TVP_stat 对比

```
Win32:   _wstat64(宽字符路径, struct _stat64)   → 填充 tTVP_stat
Linux:   stat(UTF-8 路径, struct stat)          → 填充 tTVP_stat
Android: stat(UTF-8 路径, struct stat)          → 填充 tTVP_stat
                                                    + static_assert(sizeof(st_size)==8)
```

Android 的 `TVP_stat` 有一个有趣的 `static_assert`：

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 989-999 行
bool TVP_stat(const char *name, tTVP_stat &s) {
    struct stat t;
    static_assert(sizeof(t.st_size) == 8, "");  // 确保 64 位文件大小
    bool ret = !stat(name, &t);
    // ...
}
```

这个断言确保 Android NDK 编译时 `st_size` 是 64 位的（支持大于 4GB 的文件）。在旧版 NDK 中，`stat` 默认使用 32 位 `st_size`，需要 `stat64` 或编译选项 `_FILE_OFFSET_BITS=64`。

### 4.5 文件操作对比总表

| 函数 | Win32 | Linux | Android |
|------|-------|-------|---------|
| `TVPDeleteFile` | `SHFileOperationW`（递归） | `unlink`（仅文件） | JNI → Java `DeleteFile` |
| `TVPRenameFile` | `MoveFileExW`（跨卷） | `rename`（同卷） | JNI → Java `RenameFile` |
| `TVP_stat` | `_wstat64`（UTF-16） | `stat`（UTF-8） | `stat`（UTF-8） |
| `TVP_utime` | `_wutime`（UTF-16） | `utimes`（UTF-8） | 共用 `linux/Platform.cpp` |
| **编码转换** | 每次 `MultiByteToWideChar` | 无需 | 无需（JNI NewStringUTF） |
| **大文件支持** | `_stat64`（原生 64 位） | `stat`（依赖编译选项） | `static_assert` 验证 |

---

## 5. 路径发现：启动时的关键差异

### 5.1 TVPGetDriverPath —— 可用驱动器/挂载点

**Win32**：枚举盘符 C:-Z:

```cpp
// cpp/core/environ/win32/Platform.cpp 第 179-193 行
std::vector<std::string> TVPGetDriverPath() {
    std::vector<std::string> ret;
    char drv[4] = {'C', ':', '/', 0};
    for (char c = 'C'; c <= 'Z'; ++c) {
        drv[0] = c;
        switch (GetDriveTypeA(drv)) {
        case DRIVE_REMOVABLE:  // U盘
        case DRIVE_FIXED:      // 本地硬盘
        case DRIVE_REMOTE:     // 网络驱动器
            ret.emplace_back(drv);
            break;
        }
    }
    return ret;  // 例如 {"C:/", "D:/", "E:/"}
}
```

**Linux**：返回根目录

```cpp
// cpp/core/environ/linux/Platform.cpp 第 300 行
std::vector<std::string> TVPGetDriverPath() {
    return {"/"};  // Linux 统一挂载树，只返回根
}
```

**Android**：JNI 获取存储路径或解析 `/proc/mounts`

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 413-463 行
std::vector<std::string> TVPGetDriverPath() {
    std::vector<std::string> ret;
    // 优先尝试 Java 端 getStoragePath()
    jobject sInstance = GetKR2ActInstance();
    JniMethodInfo methodInfo;
    if (JniHelper::getMethodInfo(methodInfo, KR2ActJavaPath,
            "getStoragePath", "()[Ljava/lang/String;")) {
        // ... 获取 Java String[] 转为 vector
    }

    if (!ret.empty()) return ret;

    // 回退：解析 /proc/mounts 查找 vfat 分区
    FILE *fp = fopen("/proc/mounts", "r");
    std::set<std::string> mounted;
    while (fgets(buffer, sizeof(buffer), fp)) {
        // 解析：设备 挂载点 文件系统 选项
        // 过滤：rw, vfat 或 /mnt 开头
        // 排除：/mnt/secure, /mnt/asec, tmpfs 等
    }
    return ret;
}
```

**对比**：

| 平台 | 策略 | 返回示例 |
|------|------|----------|
| Win32 | 枚举 A-Z 盘符，检查驱动器类型 | `["C:/", "D:/"]` |
| Linux | 统一根目录 | `["/"]` |
| Android | JNI 获取 → 回退 `/proc/mounts` 解析 | `["/storage/emulated/0", "/storage/sdcard1"]` |

### 5.2 TVPGetAppStoragePath —— 应用数据存储位置

```
Win32:   [当前工作目录]            // _wgetcwd() 获取
Linux:   [可执行文件所在目录]      // readlink("/proc/self/exe")
Android: [内部存储] + [外部存储]   // getFilesDir() + getExternalFilesDirs()
```

Android 的实现最复杂，因为它需要兼顾：
- 内部存储（`getFilesDir()`）—— 始终可用，但空间有限
- 外部存储（`getExternalFilesDirs()`）—— 空间大，但可能不可用（SD 卡移除）
- API 级别差异：API < 19 用 `getExternalFilesDir()`（单数），API >= 19 用 `getExternalFilesDirs()`（复数）

### 5.3 TVPCheckStartupPath —— 启动路径权限检查

```
Win32:   return true;    // 桌面端无需检查权限
Linux:   return true;    // 同上
Android: 完整的权限检查流程 →
    1. 检查路径是否可写（isWritableNormalOrSaf）
    2. 尝试创建 savedata 子目录
    3. 失败时弹出权限请求对话框
    4. Lollipop+ 提供 SAF 权限获取选项
```

这是典型的 **桌面端简单、移动端复杂** 的模式。

---

## 6. 系统工具函数对比

### 6.1 TVPRelinquishCPU —— 让出 CPU 时间片

```cpp
// Win32
void TVPRelinquishCPU() { Sleep(0); }         // 让出当前时间片

// Linux
#include <sched.h>
void TVPRelinquishCPU() { sched_yield(); }    // POSIX 线程让出

// Android: 共用 linux/Platform.cpp 的实现
```

`Sleep(0)` 和 `sched_yield()` 语义相同：告诉操作系统调度器"我可以暂时不运行，让其他同优先级线程先跑"。这在引擎主循环中很重要，可以防止 100% CPU 占用。

### 6.2 TVPPrintLog —— 日志输出

```cpp
// Win32
void TVPPrintLog(const char *str) {
    printf("%s", str);     // 标准输出
}

// Android
void TVPPrintLog(const char *str) {
    __android_log_print(ANDROID_LOG_INFO,
        "kr2 debug info", "%s", str);  // Android Logcat
}

// Linux: 无单独实现（可能使用 spdlog 全局 logger）
```

### 6.3 TVPGetRoughTickCount32 —— 粗略时间戳

```cpp
// Win32
tjs_uint32 TVPGetRoughTickCount32() {
    return timeGetTime();  // Windows 多媒体定时器，毫秒精度
}

// Linux / Android
tjs_uint32 TVPGetRoughTickCount32() {
    tjs_uint32 uptime = 0;
    struct timespec on;
    if (clock_gettime(CLOCK_MONOTONIC, &on) == 0)
        uptime = on.tv_sec * 1000 + on.tv_nsec / 1000000;
    return uptime;  // 单调时钟，毫秒精度
}
```

| | Win32 | Linux/Android |
|---|---|---|
| API | `timeGetTime()` | `clock_gettime(CLOCK_MONOTONIC)` |
| 精度 | 1-15ms（取决于 `timeBeginPeriod`） | 纳秒级（但返回毫秒） |
| 单调性 | 是（不受系统时间调整影响） | 是（`CLOCK_MONOTONIC`） |
| 溢出 | ~49.7 天（32 位毫秒） | ~49.7 天（截断为 32 位） |

### 6.4 TVPExitApplication —— 退出流程

三平台的退出流程有微妙差异：

```cpp
// Win32 (第 295-306 行)
void TVPExitApplication(int code) {
    TVPDeliverCompactEvent(TVP_COMPACT_LEVEL_MAX);  // 触发最大级别资源释放
    if (!TVPIsSoftwareRenderManager())
        iTVPTexture2D::RecycleProcess();             // GL 纹理回收
    exit(code);
}

// Linux (第 279-283 行)
void TVPExitApplication(int code) {
    TVPDeliverCompactEvent(TVP_COMPACT_LEVEL_MAX);
    exit(code);  // 没有纹理回收！
}

// Android (第 904-915 行)
void TVPExitApplication(int code) {
    TVPDeliverCompactEvent(TVP_COMPACT_LEVEL_MAX);
    if (!TVPIsSoftwareRenderManager())
        iTVPTexture2D::RecycleProcess();
    // 额外步骤：通知 Java 层退出
    JniMethodInfo t;
    if (JniHelper::getStaticMethodInfo(t, KR2ActJavaPath, "exit", "()V")) {
        t.env->CallStaticVoidMethod(t.classID, t.methodID);
        t.env->DeleteLocalRef(t.classID);
    }
    exit(code);
}
```

**差异分析**：
- Linux 版少了 `iTVPTexture2D::RecycleProcess()`——这可能是因为 Linux 桌面版默认使用软件渲染，也可能是遗漏。
- Android 版额外通过 JNI 调用 Java 的 `KR2Activity.exit()`，让 Java 层有机会执行自己的清理（如释放 `AudioTrack`、关闭 `Surface` 等）。

---

## 7. 额外平台函数（非 Platform.h 声明）

各平台 `.cpp` 文件中还包含一些 **未在 `Platform.h` 中声明** 的辅助函数：

### 7.1 Win32 独有函数

```cpp
// dlopen/dlsym 兼容层（第 60-66 行）
void *dlopen(const char *filename, int flag) {
    return (void *)LoadLibraryA(filename);
}
void *dlsym(void *handle, const char *funcname) {
    return (void *)GetProcAddress((HMODULE)handle, funcname);
}

// usleep 兼容（第 68-71 行）
extern "C" int usleep(unsigned long us) {
    Sleep(us / 1000);  // 微秒 → 毫秒
    return 0;
}

// TVPGetDefaultFileDir（第 77-87 行）
// TVPCheckStartupArg（第 93-142 行）—— 命令行参数解析
// TVPCreateFolders（第 211-258 行）—— 递归创建目录
// TVPGetCurrentLanguage（第 271-291 行）—— 获取系统语言
```

Win32 版额外提供了 POSIX 兼容函数（`dlopen`/`dlsym`/`usleep`），使引擎核心代码可以使用统一的动态库加载和微秒级睡眠 API。

### 7.2 Android 独有函数

```cpp
// 事件队列系统（第 659-690 行）
struct _eventQueueNode {
    std::function<void()> func;
    _eventQueueNode *prev, *next;
};
static std::atomic<_eventQueueNode *> _lastQueuedEvents(nullptr);

void Android_PushEvents(const std::function<void()> &func) {
    _eventQueueNode *node = new _eventQueueNode;
    node->func = func;
    node->prev = nullptr;
    // lock-free 链表插入
    while (!_lastQueuedEvents.compare_exchange_weak(node->prev, node)) {}
}
```

这是一个 **无锁（lock-free）事件队列**，用于将 Java UI 线程的回调安全地投递到 C++ GL 线程。使用 `std::atomic` + CAS（Compare-And-Swap）操作实现线程安全。

```
┌─────────────┐     Android_PushEvents()     ┌──────────────┐
│  Java/JNI   │ ──────────────────────────→  │ 无锁事件队列  │
│  UI 线程    │                               │ (atomic CAS) │
└─────────────┘                               └──────┬───────┘
                                                     │ _processEvents()
                                                     ▼
                                              ┌──────────────┐
                                              │  Cocos2d-x   │
                                              │  GL 线程调度  │
                                              └──────────────┘
```

`_processEvents` 被注册到 Cocos2d-x 的 `Scheduler` 中（在 `TVPCheckStartupArg` 里），每帧自动调用。

---

## 8. 动手实践：为 macOS 实现 Platform 接口

现在你已经理解了三平台的实现差异，让我们为 macOS 编写一个最小 Platform 实现框架：

```cpp
// 文件：cpp/core/environ/apple/Platform_macOS.cpp
#include "Platform.h"
#include <mach/mach.h>           // macOS 内存查询
#include <sys/sysctl.h>          // 系统信息
#include <sys/stat.h>
#include <unistd.h>
#include <mach-o/dyld.h>         // _NSGetExecutablePath
#include <spdlog/spdlog.h>

// ======== 内存查询 ========
void TVPGetMemoryInfo(TVPMemoryInfo &m) {
    // 方法 1：sysctl 获取物理内存总量
    int mib[2] = {CTL_HW, HW_MEMSIZE};
    uint64_t memsize = 0;
    size_t len = sizeof(memsize);
    sysctl(mib, 2, &memsize, &len, nullptr, 0);
    m.MemTotal = memsize / 1024;  // 字节 → kB

    // 方法 2：mach API 获取空闲内存
    mach_port_t host = mach_host_self();
    vm_statistics64_data_t vmstat;
    mach_msg_type_number_t count = HOST_VM_INFO64_COUNT;
    host_statistics64(host, HOST_VM_INFO64,
        (host_info64_t)&vmstat, &count);

    vm_size_t pagesize;
    host_page_size(host, &pagesize);
    m.MemFree = (vmstat.free_count * pagesize) / 1024;

    // macOS 没有传统 swap 概念，使用 xpc 或 sysctl 查询
    m.SwapTotal = 0;
    m.SwapFree = 0;
    m.VirtualTotal = 0;
    m.VirtualUsed = 0;
}

tjs_int TVPGetSystemFreeMemory() {
    TVPMemoryInfo m;
    TVPGetMemoryInfo(m);
    return m.MemFree / 1024;  // kB → MB
}

tjs_int TVPGetSelfUsedMemory() {
    struct mach_task_basic_info info;
    mach_msg_type_number_t size = MACH_TASK_BASIC_INFO_COUNT;
    task_info(mach_task_self(), MACH_TASK_BASIC_INFO,
        (task_info_t)&info, &size);
    return info.resident_size / (1024 * 1024);
}

// ======== 消息框（使用 Cocoa NSAlert 需要 Objective-C++）========
// 在 .mm 文件中实现，这里声明 stub
int TVPShowSimpleMessageBox(const ttstr &text, const ttstr &caption,
                            const std::vector<ttstr> &vecButtons) {
    // TODO: 用 NSAlert 实现
    spdlog::get("core")->warn("macOS message box not yet implemented");
    return 0;
}

// ======== 文件操作（与 Linux 相同，macOS 也是 POSIX）========
bool TVPDeleteFile(const std::string &filename) {
    return unlink(filename.c_str()) == 0;
}

bool TVPRenameFile(const std::string &from, const std::string &to) {
    return rename(from.c_str(), to.c_str()) == 0;
}

// ======== 路径发现 ========
std::vector<std::string> TVPGetDriverPath() {
    return {"/"};  // macOS 与 Linux 一样使用统一挂载树
}

std::string TVPGetDefaultFileDir() {
    char path[1024];
    uint32_t size = sizeof(path);
    _NSGetExecutablePath(path, &size);  // macOS 专用
    // 需要 realpath 解析符号链接
    char resolved[PATH_MAX];
    realpath(path, resolved);
    // 截取目录部分
    std::string fullpath(resolved);
    return fullpath.substr(0, fullpath.find_last_of('/'));
}

// ======== 系统工具 ========
void TVPRelinquishCPU() { sched_yield(); }  // 与 Linux 相同

void TVPPrintLog(const char *str) {
    printf("%s", str);  // 或使用 os_log
}
```

**实践要点**：
1. macOS 内存查询使用 **Mach 内核 API**（`host_statistics64`），不能用 Linux 的 `/proc/meminfo`
2. 消息框需要 Objective-C++（`.mm` 文件）调用 `NSAlert`
3. 文件操作与 Linux 几乎相同（都是 POSIX），但路径发现用 `_NSGetExecutablePath` 而非 `/proc/self/exe`
4. CMakeLists.txt 需要添加 macOS 条件：`$<$<BOOL:${MACOS}>:apple/Platform_macOS.cpp>`

---

## 9. 常见错误及解决方案

### 错误 1：Windows 上中文文件名操作失败

**现象**：`TVPDeleteFile("存档/save01.dat")` 返回 false。

**原因**：`MultiByteToWideChar` 的第一个参数是 `CP_UTF8`，如果传入的字符串不是有效 UTF-8（比如 GBK 编码），转换会失败。

**解决方案**：
```cpp
// 在调用 TVPDeleteFile 前确保路径是 UTF-8
// 如果源数据是 GBK，先转换：
#include <boost/locale.hpp>
std::string utf8_path = boost::locale::conv::to_utf<char>(
    gbk_path, "GBK");
TVPDeleteFile(utf8_path);
```

### 错误 2：Android 上 JNI 局部引用泄漏

**现象**：在循环中频繁调用 `TVPDeleteFile` 后 App 崩溃，Logcat 显示 `JNI local reference table overflow`。

**原因**：JNI 局部引用表默认上限 512 个。如果在循环中创建 jstring 但忘记 `DeleteLocalRef`，会逐渐填满。

**解决方案**：
```cpp
// 已正确实现：每个 NewStringUTF 后都有 DeleteLocalRef
jstring jstr = methodInfo.env->NewStringUTF(filename.c_str());
// ... 使用 jstr ...
methodInfo.env->DeleteLocalRef(jstr);  // ✅ 必须释放

// 如果在大循环中，还可以用 PushLocalFrame/PopLocalFrame 批量管理
methodInfo.env->PushLocalFrame(16);  // 预留 16 个局部引用
// ... 循环体 ...
methodInfo.env->PopLocalFrame(nullptr);  // 自动释放所有
```

### 错误 3：Linux 上 TVP_stat 获取不到大文件大小

**现象**：文件大于 2GB 时，`tTVP_stat.st_size` 返回错误值。

**原因**：如果编译时未定义 `_FILE_OFFSET_BITS=64`，Linux 默认使用 32 位 `struct stat`，`st_size` 是 `off_t`（32 位），无法表示大于 2GB 的文件。

**解决方案**：
```cmake
# CMakeLists.txt 中添加编译定义
target_compile_definitions(core_environ_module PRIVATE
    _FILE_OFFSET_BITS=64
)
```

KrKr2 的 Android 版用 `static_assert(sizeof(t.st_size) == 8, "")` 来检测这个问题，Linux 版没有这个断言（可考虑添加）。

---

## 10. 对照项目源码

### 关键文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `cpp/core/environ/Platform.h` | 73 | PAL 接口声明（全部） |
| `cpp/core/environ/win32/Platform.cpp` | 408 | Win32 完整实现 |
| `cpp/core/environ/linux/Platform.cpp` | 335 | Linux 完整实现 |
| `cpp/core/environ/android/AndroidUtils.cpp` | 1002 | Android 完整实现（含 JNI 辅助） |
| `cpp/core/environ/CMakeLists.txt` | 88 | 平台条件编译选择 |

### 重点行号索引

- **内存查询**：Win32 第 20-43 行 / Linux 第 10-53,82-95 行 / Android 第 46-88 行
- **消息框**：Win32 第 144-177 行 / Linux 第 140-193 行 / Android 第 515-567 行
- **文件删除**：Win32 第 319-339 行 / Linux 第 289-291 行 / Android 第 939-952 行
- **路径发现**：Win32 第 179-199 行 / Linux 第 300-322 行 / Android 第 406-463 行
- **退出流程**：Win32 第 295-306 行 / Linux 第 279-283 行 / Android 第 904-915 行

---

## 本节小结

1. `Platform.h` 是 KrKr2 PAL 层的"契约"，定义了 7 组约 25 个函数和 2 个结构体。
2. 采用 **C 风格自由函数 + 编译期平台选择** 方案，零运行时开销。
3. **内存查询**：Win32 用 Win32 API，Linux 解析 `/proc/meminfo`，Android 通过 JNI 委托 Java 层（带 3 秒缓存）。
4. **消息框**：Win32 用 `MessageBoxW`，Linux 用 GTK，Android 用 JNI + 条件变量跨线程等待。
5. **文件操作**：Win32 必须做 UTF-8 → UTF-16 编码转换，Linux 原生 UTF-8，Android 委托 Java 层应对 Scoped Storage。
6. **路径发现**：Win32 枚举盘符，Linux 返回根目录，Android 通过 JNI 获取多层存储路径。
7. 各平台还有独有的辅助函数：Win32 的 POSIX 兼容层（dlopen/usleep）、Android 的无锁事件队列。
8. 为新平台（macOS）添加实现时，核心模式不变：实现 Platform.h 中声明的所有函数 + CMake 条件编译。

---

## 练习题与答案

### 题目 1：为什么 KrKr2 选择 C 风格自由函数而非虚函数 PAL？

请从性能、使用场景和编译模型三个角度分析。

<details>
<summary>查看答案</summary>

**性能角度**：C 风格函数在编译期就确定了调用目标，没有虚函数表（vtable）查找开销。对于每帧调用的函数（如 `TVPGetSystemFreeMemory`、`TVPRelinquishCPU`），这减少了一次间接跳转。

**使用场景角度**：游戏引擎不需要运行时切换平台实现——你不会在 Windows 上临时切换到 Android 的内存查询实现。所以运行时多态（虚函数）带来的灵活性在这个场景下没有价值。

**编译模型角度**：CMake 生成器表达式 `$<$<BOOL:${PLATFORM}>:...>` 在配置阶段就选定了源文件，链接器只会看到一个平台的 `.cpp`。这比维护抽象基类 + 工厂模式 + 平台子类的继承体系简单得多，也更容易被新贡献者理解。

</details>

### 题目 2：请写出一个完整的 TVPGetMemoryInfo 实现（macOS 版）

要求使用 Mach 内核 API，填充 `TVPMemoryInfo` 的至少 MemTotal 和 MemFree 字段。

<details>
<summary>查看答案</summary>

```cpp
#include "Platform.h"
#include <mach/mach.h>
#include <sys/sysctl.h>

void TVPGetMemoryInfo(TVPMemoryInfo &m) {
    // 1. 物理内存总量：sysctl
    int mib[2] = {CTL_HW, HW_MEMSIZE};
    uint64_t memsize = 0;
    size_t len = sizeof(memsize);
    if (sysctl(mib, 2, &memsize, &len, nullptr, 0) == 0) {
        m.MemTotal = static_cast<unsigned long>(memsize / 1024); // 字节 → kB
    }

    // 2. 空闲内存：Mach host_statistics64
    mach_port_t host = mach_host_self();
    vm_statistics64_data_t vmstat;
    mach_msg_type_number_t count = HOST_VM_INFO64_COUNT;
    if (host_statistics64(host, HOST_VM_INFO64,
            (host_info64_t)&vmstat, &count) == KERN_SUCCESS) {
        vm_size_t pagesize;
        host_page_size(host, &pagesize);
        m.MemFree = static_cast<unsigned long>(
            vmstat.free_count * pagesize / 1024); // 字节 → kB
    }

    // 3. macOS 使用动态 swap 文件，不像 Linux 有固定分区
    //    可通过 sysctl vm.swapusage 获取，但简化处理设为 0
    m.SwapTotal = 0;
    m.SwapFree = 0;
    m.VirtualTotal = 0;
    m.VirtualUsed = 0;
}
```

编译命令（macOS）：
```bash
clang++ -std=c++17 -framework CoreFoundation \
    -I/path/to/krkr2/cpp/core/environ \
    -I/path/to/krkr2/cpp/core/tjs2 \
    Platform_macOS.cpp -c -o Platform_macOS.o
```

</details>

### 题目 3：Android 的 TVPShowSimpleMessageBox 中为什么要在等待循环里调用 TVPForceSwapBuffer？

<details>
<summary>查看答案</summary>

`TVPForceSwapBuffer()` 调用 `eglSwapBuffers()`，作用是保持 OpenGL ES 上下文活跃。原因：

1. **Android 系统看门狗**：如果 GL 线程长时间不提交帧（通常 5-10 秒），Android 的 `SurfaceFlinger` 或 `WindowManagerService` 可能认为应用无响应（ANR），回收 EGL Surface 或弹出"应用无响应"对话框。

2. **EGL 上下文保活**：`eglSwapBuffers` 告诉系统"我还在正常运行，只是在等待用户输入"。即使画面没有变化，交换缓冲区也会刷新 VSync 计数器，避免被判定为卡死。

3. **200ms 间隔**：`wait_for(lk, std::chrono::milliseconds(200))` 每 200ms 唤醒一次，检查用户是否已点击按钮（`MsgBoxRet != -2`），如果还没有就 swap 一次缓冲区，然后继续等待。这个间隔足够低以避免 ANR，又不会浪费太多 CPU。

如果去掉 `TVPForceSwapBuffer()` 调用，在较慢的设备上，用户看到消息框后还没来得及点击，应用可能就被系统判定为 ANR 并被强制关闭。

</details>

---

## 下一步

[第 3 章：Application 生命周期与消息系统](./03-Application生命周期与消息系统.md) —— 深入分析 `Application.h/cpp` 的启动流程、主循环、消息分发与生命周期管理。
