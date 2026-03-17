## KrKr2 的类型兼容层：typedefine.h

KrKr2 有一个独特的跨平台挑战：原版 KiriKiri 引擎是纯 Windows 程序，大量代码使用 Windows 专有类型（`DWORD`、`HANDLE`、`HRESULT` 等）。为了在非 Windows 平台上编译这些代码而不大量修改，KrKr2 采用了**类型别名兼容层**：

```cpp
// cpp/core/environ/typedefine.h（精简版）
#pragma once
#include <stdint.h>

// 平台检测
#if defined(_WIN32) || defined(_WIN64)
#define TARGET_WINDOWS 1
#include <windows.h>    // Windows: 直接用原生类型
#else
#define TARGET_WINDOWS 0

// 非 Windows: 用 stdint 类型别名模拟 Windows 类型
typedef int32_t  LONG;
typedef uint32_t ULONG;
typedef uint32_t DWORD;
typedef uint16_t WORD;
typedef uint8_t  BYTE;
typedef void*    HBITMAP;
typedef void*    HDC;
typedef void*    HFONT;
typedef void*    HICON;
typedef LONG     HRESULT;

// Windows 结构体模拟
typedef struct _FILETIME {
    DWORD dwLowDateTime;
    DWORD dwHighDateTime;
} FILETIME;

// ... 更多 Windows 类型别名
#endif

// 所有平台通用的固定宽度类型
typedef uint8_t  UINT8;
typedef uint16_t UINT16;
typedef uint32_t UINT32;
typedef uint64_t UINT64;
```

这种方案让原版 KiriKiri 的代码无需大量修改就能在四个平台编译，是一种务实的跨平台移植策略。

---

## 动手实践

### 实践：为 KrKr2 添加一个新的 PAL 函数

假设我们需要添加一个函数 `TVPGetCPUModelString()`，返回当前 CPU 的型号字符串。按照 KrKr2 的文件分离模式：

**步骤 1：** 在 `Platform.h` 中声明接口

```cpp
// 在 Platform.h 末尾添加
std::string TVPGetCPUModelString();
```

**步骤 2：** Windows 实现

```cpp
// win32/Platform.cpp 中添加
#include <intrin.h>

std::string TVPGetCPUModelString() {
    int cpuInfo[4] = {};
    char brand[48] = {};

    // 调用 CPUID 指令获取 CPU 品牌字符串
    __cpuid(cpuInfo, 0x80000002);
    memcpy(brand, cpuInfo, sizeof(cpuInfo));
    __cpuid(cpuInfo, 0x80000003);
    memcpy(brand + 16, cpuInfo, sizeof(cpuInfo));
    __cpuid(cpuInfo, 0x80000004);
    memcpy(brand + 32, cpuInfo, sizeof(cpuInfo));

    return std::string(brand);
}
```

**步骤 3：** Linux 实现

```cpp
// linux/Platform.cpp 中添加
#include <fstream>

std::string TVPGetCPUModelString() {
    std::ifstream cpuinfo("/proc/cpuinfo");
    std::string line;
    while (std::getline(cpuinfo, line)) {
        if (line.find("model name") != std::string::npos) {
            auto pos = line.find(':');
            if (pos != std::string::npos) {
                return line.substr(pos + 2); // 跳过 ": "
            }
        }
    }
    return "Unknown CPU";
}
```

**步骤 4：** macOS 实现

```cpp
// apple/macos/platform.mm 中添加
#include <sys/sysctl.h>

std::string TVPGetCPUModelString() {
    char brand[256] = {};
    size_t size = sizeof(brand);
    sysctlbyname("machdep.cpu.brand_string", brand, &size, nullptr, 0);
    return std::string(brand);
}
```

**步骤 5：** Android 实现

```cpp
// android/AndroidUtils.cpp 中添加
#include <fstream>

std::string TVPGetCPUModelString() {
    // Android 同 Linux，读取 /proc/cpuinfo
    std::ifstream cpuinfo("/proc/cpuinfo");
    std::string line;
    while (std::getline(cpuinfo, line)) {
        // ARM 上字段名是 "Hardware" 而非 "model name"
        if (line.find("Hardware") != std::string::npos ||
            line.find("model name") != std::string::npos) {
            auto pos = line.find(':');
            if (pos != std::string::npos) {
                return line.substr(pos + 2);
            }
        }
    }
    return "Unknown ARM CPU";
}
```

不需要修改 CMakeLists.txt——因为这些 `.cpp` 文件已经被 CMake 选择编译了。

---

