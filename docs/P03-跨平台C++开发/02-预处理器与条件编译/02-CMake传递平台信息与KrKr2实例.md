## CMake 向 C++ 传递平台信息

### KrKr2 的做法

KrKr2 项目在根 `CMakeLists.txt` 中定义了自定义平台变量，并通过 `target_compile_definitions` 传递给 C++ 代码：

```cmake
# krkr2/CMakeLists.txt（简化）

# 自定义平台检测变量
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    set(WINDOWS TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(LINUX TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set(MACOS TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Android")
    set(ANDROID TRUE)
endif()

# 将平台信息传递给 C++ 代码
target_compile_definitions(krkr2core INTERFACE
    $<$<BOOL:${WINDOWS}>:WINDOWS=1>
    $<$<BOOL:${LINUX}>:LINUX=1>
    $<$<BOOL:${MACOS}>:MACOS=1>
    $<$<BOOL:${ANDROID}>:ANDROID=1>
)
```

这样 C++ 代码中就可以使用 `WINDOWS`、`LINUX`、`MACOS`、`ANDROID` 宏：

```cpp
// 使用 KrKr2 自定义的平台宏（通过 CMake 传入）
#ifdef WINDOWS
    #include <windows.h>
    void showMessage(const char* msg) {
        MessageBoxA(nullptr, msg, "KrKr2", MB_OK);
    }
#elif defined(LINUX)
    #include <gtk/gtk.h>
    void showMessage(const char* msg) {
        GtkWidget* dialog = gtk_message_dialog_new(nullptr,
            GTK_DIALOG_MODAL, GTK_MESSAGE_INFO, GTK_BUTTONS_OK, "%s", msg);
        gtk_dialog_run(GTK_DIALOG(dialog));
        gtk_widget_destroy(dialog);
    }
#elif defined(ANDROID)
    #include <android/log.h>
    void showMessage(const char* msg) {
        __android_log_print(ANDROID_LOG_INFO, "KrKr2", "%s", msg);
    }
#elif defined(MACOS)
    // macOS 使用 Objective-C++ 的 NSAlert 或 Cocos2d UI
    void showMessage(const char* msg);
#endif
```

### `target_compile_definitions` 详解

```cmake
# 基本用法：为目标添加编译定义
target_compile_definitions(my_target PRIVATE
    MY_MACRO              # 定义 MY_MACRO（无值）
    VERSION=3             # 定义 VERSION 为 3
    NAME="hello"          # 定义 NAME 为字符串 "hello"
)

# 可见性关键字
# PRIVATE   — 仅当前目标可见
# PUBLIC    — 当前目标 + 链接此目标的所有目标
# INTERFACE — 仅链接此目标的其他目标（当前目标不可见）

# 生成器表达式：条件定义
target_compile_definitions(my_target PRIVATE
    # 仅 Debug 配置时定义
    $<$<CONFIG:Debug>:_DEBUG>

    # 仅 Windows 平台时定义
    $<$<PLATFORM_ID:Windows>:WIN32_LEAN_AND_MEAN>

    # 组合条件：Debug + Windows
    $<$<AND:$<CONFIG:Debug>,$<PLATFORM_ID:Windows>>:WIN_DEBUG_MODE>
)
```

### `configure_file` 方式

另一种常见做法是通过 `configure_file` 生成配置头文件：

```cmake
# CMakeLists.txt
set(PROJECT_VERSION_MAJOR 2)
set(PROJECT_VERSION_MINOR 0)
set(HAS_OPENGL TRUE)
set(HAS_VULKAN FALSE)

configure_file(
    ${CMAKE_SOURCE_DIR}/config.h.in
    ${CMAKE_BINARY_DIR}/config.h
)
```

```cpp
// config.h.in（模板文件）
#pragma once

#define PROJECT_VERSION_MAJOR @PROJECT_VERSION_MAJOR@
#define PROJECT_VERSION_MINOR @PROJECT_VERSION_MINOR@

#cmakedefine HAS_OPENGL
#cmakedefine HAS_VULKAN
// HAS_OPENGL 为 TRUE → 生成 #define HAS_OPENGL
// HAS_VULKAN 为 FALSE → 生成 /* #undef HAS_VULKAN */
```

```cpp
// 生成的 config.h
#pragma once

#define PROJECT_VERSION_MAJOR 2
#define PROJECT_VERSION_MINOR 0

#define HAS_OPENGL
/* #undef HAS_VULKAN */
```

```cpp
// 使用生成的配置
#include "config.h"

#ifdef HAS_OPENGL
    #include <GL/gl.h>
    // OpenGL 渲染代码
#endif

#ifdef HAS_VULKAN
    #include <vulkan/vulkan.h>
    // Vulkan 渲染代码（当前未启用）
#endif
```

---

## KrKr2 条件编译实例分析

### 实例 1：平台入口点

KrKr2 的每个平台有独立的入口文件，但它们都调用相同的核心逻辑：

```
platforms/
├── windows/main.cpp     ← #include <windows.h>, WinMain 入口
├── linux/main.cpp       ← #include <gtk/gtk.h>, int main() 入口
├── apple/macos/main.cpp ← #include <cocos2d.h>, int main() 入口
└── android/cpp/
    └── krkr2_android.cpp ← JNI_OnLoad, Java 桥接
```

这种方式称为**文件级条件编译**——不是用 `#ifdef` 在一个文件内区分平台，而是为每个平台提供独立的源文件，通过 CMake 选择编译哪些文件：

```cmake
# 根据平台选择入口文件
if(WINDOWS)
    add_executable(krkr2 WIN32 platforms/windows/main.cpp)
elseif(LINUX)
    add_executable(krkr2 platforms/linux/main.cpp)
elseif(MACOS)
    add_executable(krkr2 MACOSX_BUNDLE platforms/apple/macos/main.cpp)
elseif(ANDROID)
    add_library(krkr2 SHARED platforms/android/cpp/krkr2_android.cpp)
endif()
```

### 实例 2：编译定义传递

KrKr2 的 `cpp/core/CMakeLists.txt` 中为核心库添加了多个编译定义：

```cmake
# cpp/core/CMakeLists.txt（实际代码简化版）
target_compile_definitions(krkr2core INTERFACE
    TJS_TEXT_OUT_CRLF           # TJS2 脚本输出使用 CRLF 换行
    __STDC_CONSTANT_MACROS      # C99 常量宏（INT64_C 等）
    USE_UNICODE_FSTRING         # 使用 Unicode 字符串
)
```

这些定义使用 `INTERFACE` 可见性——`krkr2core` 是 INTERFACE 库，它本身不编译任何代码，但所有链接到它的目标（如 `krkr2` 可执行文件）都会继承这些定义。

### 实例 3：OpenMP 条件启用

```cmake
# 根 CMakeLists.txt
if(NOT APPLE)
    find_package(OpenMP)
    if(OpenMP_CXX_FOUND)
        target_link_libraries(krkr2core INTERFACE OpenMP::OpenMP_CXX)
    endif()
endif()
```

Apple 平台默认不支持 OpenMP（Apple Clang 未内置），因此 KrKr2 在 macOS 上跳过 OpenMP。对应的 C++ 代码中：

```cpp
// 使用 _OPENMP 宏判断 OpenMP 是否可用
#ifdef _OPENMP
    #include <omp.h>
    #pragma omp parallel for
    for (int i = 0; i < count; ++i) {
        processPixel(i);
    }
#else
    // 串行回退
    for (int i = 0; i < count; ++i) {
        processPixel(i);
    }
#endif
```

### 实例 4：environ 目录的条件编译

`cpp/core/environ/` 目录包含五个平台实现子目录：

```
environ/
├── cocos2d/       ← 所有平台共用的 Cocos2d 桥接
├── win32/         ← Windows 专用
├── linux/         ← Linux 专用
├── android/       ← Android 专用
├── apple/         ← macOS 专用
├── sdl/           ← SDL2 后端（Linux/macOS 可选）
└── ui/            ← Cocos2d UI 组件（所有平台）
```

CMake 根据平台变量选择编译哪些子目录中的源文件，而不是在单个文件中用大量 `#ifdef`。这是一种更清晰的跨平台代码组织方式。

---

