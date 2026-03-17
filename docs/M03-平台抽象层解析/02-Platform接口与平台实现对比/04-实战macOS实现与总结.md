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
