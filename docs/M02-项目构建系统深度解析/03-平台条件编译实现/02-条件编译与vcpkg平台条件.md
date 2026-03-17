# CMakeLists.txt 中的条件编译
## CMakeLists.txt 中的条件编译

### 自定义平台变量

KrKr2 没有直接使用 CMake 标准变量 `CMAKE_SYSTEM_NAME`，而是定义了自己的平台布尔变量：

```cmake
# 在根 CMakeLists.txt 第 7-14 行（简化）
# 平台入口源文件注册
if(WINDOWS)
    list(APPEND PLATFORM_SRCS platforms/windows/main.cpp ...)
elseif(LINUX)
    list(APPEND PLATFORM_SRCS platforms/linux/main.cpp)
elseif(MACOS)
    list(APPEND PLATFORM_SRCS platforms/apple/macos/main.cpp)
elseif(ANDROID)
    # Android 是共享库，不是可执行文件
endif()
```

这些变量的来源和映射：

| 自定义变量 | 来源 | 对应 `CMAKE_SYSTEM_NAME` |
|-----------|------|-------------------------|
| `WINDOWS` | CMakePresets.json 中 triplet 为 `x64-windows-*` 时由 vcpkg 自动设置 | `Windows` |
| `LINUX` | triplet 为 `x64-linux` 时由 vcpkg 设置，或 CMakeLists.txt 中显式检测 | `Linux` |
| `MACOS` | triplet 为 `*-osx` 时由 vcpkg 设置，或显式检测 | `Darwin` |
| `ANDROID` | Android NDK 工具链文件自动设置 `ANDROID=1` | `Android` |

> **为什么不直接用 CMAKE_SYSTEM_NAME？** 使用自定义变量有两个好处：(1) 名称更直观（`ANDROID` 比 `CMAKE_SYSTEM_NAME STREQUAL "Android"` 更简洁）；(2) 可以在多处使用而无需重复条件判断。

### 平台入口文件分支

根 CMakeLists.txt 中最重要的条件编译块是平台入口文件注册。完整分析：

```cmake
# 第 50-85 行（简化重组，保留核心逻辑）
if(ANDROID)
    # Android 构建共享库（.so），不是可执行文件
    add_library(krkr2 SHARED
        platforms/android/cpp/krkr2_android.cpp
    )
    # Android breakpad 崩溃报告
    find_package(breakpad CONFIG REQUIRED)
    target_link_libraries(krkr2 PRIVATE breakpad::breakpad)
else()
    # 桌面平台构建可执行文件
    if(WINDOWS)
        add_executable(krkr2
            platforms/windows/main.cpp
            platforms/windows/krkr2.rc    # Windows 资源文件（图标、版本信息）
        )
    elseif(LINUX)
        add_executable(krkr2
            platforms/linux/main.cpp
        )
    elseif(MACOS)
        add_executable(krkr2
            platforms/apple/macos/main.cpp
        )
    endif()
endif()
```

**Android 的根本差异：**

| 特性 | 桌面平台 | Android |
|------|---------|---------|
| 目标类型 | `add_executable()` | `add_library(SHARED)` |
| 入口函数 | `main()` | JNI `JNI_OnLoad()` / Java Activity |
| 产物 | `krkr2.exe` / `krkr2` | `libkrkr2.so` |
| 调用者 | 操作系统 | Android Java 虚拟机 |
| 崩溃报告 | 无（由操作系统处理） | breakpad（native crash 需要特殊处理） |

### MSVC 编译选项

```cmake
# 第 90-95 行
if(MSVC)
    target_compile_options(krkr2 PRIVATE
        /EHsc   # C++ 异常处理模型
        /MP     # 多处理器编译（MSVC 并行编译源文件）
        /utf-8  # 源文件和执行字符集均为 UTF-8
    )
endif()
```

各选项详解：

| 选项 | 全称 | 作用 | 不设置的后果 |
|------|------|------|------------|
| `/EHsc` | Exception Handling: Synchronous, C++ only | 启用 C++ 异常（`try`/`catch`），不捕获 SEH | 代码中的 `throw` 语句导致 undefined behavior |
| `/MP` | Multi-Processor compilation | 同时编译多个 `.cpp` 文件 | 编译速度慢 2-4 倍（单线程编译） |
| `/utf-8` | UTF-8 source and execution charset | 源代码和字符串字面量都用 UTF-8 | 中文/日文字符串可能出现编码错误（Windows 默认使用本地代码页） |

> **`/utf-8` 对 KrKr2 的重要性：** KrKr2 处理日语视觉小说游戏，源代码中包含日文字符串常量。如果不使用 `/utf-8`，MSVC 会按 Windows 系统本地编码（如 GBK、Shift-JIS）解释源文件，导致字符串乱码。

### 测试和工具的条件构建

```cmake
# 第 120-135 行
if(NOT ANDROID)
    # 测试框架
    find_package(Catch2 3 CONFIG REQUIRED)
    enable_testing()
    add_subdirectory(tests)

    # XP3 解包工具
    find_package(argparse CONFIG REQUIRED)
    add_subdirectory(tools/xp3)
endif()
```

**为什么 Android 排除测试和工具？**

1. **Catch2 测试框架**在 Android 上无法直接运行——没有终端可以查看测试输出
2. **XP3 解包工具**是命令行程序，不需要在 Android 上构建
3. `vcpkg.json` 中也通过 `"platform"` 字段排除了这些包：

```json
{
    "name": "catch2",
    "platform": "!android & !ios"
},
{
    "name": "argparse",
    "platform": "!android & !ios"
}
```

---

## vcpkg.json 中的平台条件

vcpkg.json 的 `"platform"` 字段使用 vcpkg 平台表达式语法（与 CMake 不同）：

```json
{
    "dependencies": [
        {
            "name": "breakpad",
            "platform": "android"
        },
        {
            "name": "dirent",
            "platform": "windows"
        },
        {
            "name": "glfw3",
            "platform": "!android & !ios"
        },
        {
            "name": "libgdiplus",
            "platform": "!windows"
        }
    ]
}
```

### 平台表达式语法

| 表达式 | 含义 | 示例 |
|--------|------|------|
| `windows` | 仅 Windows | `"dirent"` — POSIX `dirent.h` 的 Windows 实现 |
| `android` | 仅 Android | `"breakpad"` — native crash 报告 |
| `!android` | 除 Android 外所有平台 | 桌面专用库 |
| `!android & !ios` | 排除移动平台 | `"catch2"`、`"argparse"`、`"glfw3"` |
| `!windows` | 除 Windows 外所有平台 | `"libgdiplus"` — Windows 自带 GDI+ |

**为什么 `dirent` 仅在 Windows 上？** POSIX 系统（Linux/macOS/Android）原生提供 `<dirent.h>` 头文件用于目录遍历。Windows 没有，需要第三方实现。

**为什么 `libgdiplus` 排除 Windows？** GDI+ 是 Windows 原生 API。非 Windows 平台需要开源实现 `libgdiplus` 来提供兼容的图像处理功能。

**为什么 `glfw3` 排除移动平台？** GLFW 是桌面端的窗口/输入管理库，不支持 Android/iOS。移动平台使用各自的原生窗口系统（Android 用 `ANativeWindow`，通过 SDL2/Cocos2d-x 封装）。

### 依赖版本锁定

```json
{
    "overrides": [
        {
            "name": "opencv4",
            "version": "4.7.0",
            "port-version": 6
        }
    ]
}
```

`overrides` 强制锁定 opencv4 到特定版本。这通常是因为：
1. 更新版本引入了 API 变更，项目代码尚未适配
2. 更新版本在某些平台上构建失败
3. 需要确保所有开发者使用相同版本以避免行为差异

---

## vcpkg_android.cmake：Android 工具链叠加

Android 平台有一个特殊的 CMake 模块——`cmake/vcpkg_android.cmake`（99 行）。它解决了一个棘手的问题：**如何同时使用 Android NDK 工具链和 vcpkg 工具链**。

### 问题背景

CMake 的 `CMAKE_TOOLCHAIN_FILE` 只能设置一个文件：

```bash
# 只能选一个！
cmake -DCMAKE_TOOLCHAIN_FILE=/path/to/android.toolchain.cmake   # NDK 交叉编译
# 或
cmake -DCMAKE_TOOLCHAIN_FILE=/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake  # vcpkg 包管理
```

但 KrKr2 的 Android 构建**两个都需要**：
- Android NDK 工具链：指定目标架构（arm64-v8a）、API 级别、编译器路径
- vcpkg 工具链：管理第三方库的查找路径

### 解决方案：VCPKG_CHAINLOAD_TOOLCHAIN_FILE

`vcpkg_android.cmake` 使用 vcpkg 的**链式加载**机制：

```cmake
# cmake/vcpkg_android.cmake 核心逻辑（简化）

# 1. 检查必要环境变量
if(NOT DEFINED ENV{ANDROID_NDK_HOME})
    message(FATAL_ERROR "请设置 ANDROID_NDK_HOME 环境变量")
endif()
if(NOT DEFINED ENV{VCPKG_ROOT})
    message(FATAL_ERROR "请设置 VCPKG_ROOT 环境变量")
endif()

# 2. ABI 到 vcpkg triplet 的映射
# Android NDK 使用 ABI 名称，vcpkg 使用 triplet 名称
if(ANDROID_ABI STREQUAL "arm64-v8a")
    set(VCPKG_TARGET_TRIPLET "arm64-android")
elseif(ANDROID_ABI STREQUAL "x86_64")
    set(VCPKG_TARGET_TRIPLET "x64-android")
elseif(ANDROID_ABI STREQUAL "armeabi-v7a")
    set(VCPKG_TARGET_TRIPLET "arm-android")
elseif(ANDROID_ABI STREQUAL "x86")
    set(VCPKG_TARGET_TRIPLET "x86-android")
endif()

# 3. 链式加载：vcpkg 工具链加载 NDK 工具链
set(VCPKG_CHAINLOAD_TOOLCHAIN_FILE
    "$ENV{ANDROID_NDK_HOME}/build/cmake/android.toolchain.cmake")

# 4. 设置 vcpkg 工具链为主工具链
set(CMAKE_TOOLCHAIN_FILE
    "$ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake")
```

**执行流程：**

```
CMake 配置
  │
  ├─ 加载 CMAKE_TOOLCHAIN_FILE (vcpkg.cmake)
  │   │
  │   ├─ vcpkg 设置包搜索路径
  │   │
  │   └─ 发现 VCPKG_CHAINLOAD_TOOLCHAIN_FILE
  │       │
  │       └─ 加载 android.toolchain.cmake
  │           ├─ 设置 CMAKE_C_COMPILER = NDK clang
  │           ├─ 设置 CMAKE_CXX_COMPILER = NDK clang++
  │           ├─ 设置 CMAKE_SYSROOT = NDK sysroot
  │           └─ 设置 ANDROID_ABI, ANDROID_PLATFORM 等
  │
  └─ 两个工具链都生效！
```

### ABI 与 Triplet 映射表

| Android ABI | vcpkg Triplet | 目标 CPU | 最低 API |
|-------------|---------------|---------|---------|
| `arm64-v8a` | `arm64-android` | ARM 64-bit (AArch64) | 21 |
| `x86_64` | `x64-android` | x86 64-bit | 21 |
| `armeabi-v7a` | `arm-android` | ARM 32-bit | 16 |
| `x86` | `x86-android` | x86 32-bit | 16 |

KrKr2 的 `app/build.gradle` 只启用了 `arm64-v8a` 和 `x86_64`（64 位），不支持 32 位架构：

```groovy
// platforms/android/app/build.gradle
ndk {
    abiFilters 'arm64-v8a', 'x86_64'
}
```

---

