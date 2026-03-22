# 01 - Kotlin/Native 与 cinterop

> **所属模块：** M13-UI框架替换实战  
> **前置知识：** [03-Flutter嵌入实战/02-PlatformChannel实现UI交互](../03-Flutter嵌入实战/02-PlatformChannel实现UI交互.md)  
> **预计阅读时间：** 60 分钟

---

## 本节目标

- 理解 Kotlin/Native（KN）的运行时模型与内存管理机制
- 掌握使用 cinterop 工具从 C 头文件自动生成 Kotlin 绑定的完整流程
- 学会在 Gradle 项目中配置 cinterop 任务
- 能够从 Kotlin 代码直接调用 KrKr2 的 C++ 接口

---

## 术语预览

| 术语 | 解释 |
|---|---|
| **Kotlin/Native (KN)** | Kotlin 的一个编译目标，将 Kotlin 代码编译为原生机器码（.so/.dylib/.dll），无需 JVM |
| **cinterop** | KN 内置工具，读取 C 头文件（.h）并生成对应的 Kotlin 绑定代码 |
| **def 文件** | cinterop 的配置文件（`.def`），描述要绑定的头文件、库、编译选项 |
| **CPointer** | KN 中对 C 指针类型的封装，如 `CPointer<ByteVar>` 对应 `char*` |
| **memScoped** | KN 提供的作用域内存管理块，在块退出时自动释放在其中分配的 C 内存 |
| **StableRef** | KN 中将 Kotlin 对象固定在内存中（避免 GC 移动），用于传递给 C 回调 |
| **Kotlin/Native GC** | KN 3.0+ 使用的新 GC，基于标记-清除，支持并发回收 |
| **interop stub** | cinterop 生成的 Kotlin 函数，作为 C 函数调用的桥接代理 |

---

## 一、Kotlin/Native 运行时模型

### 1.1 KN 与 JVM Kotlin 的核心区别

Kotlin/Native 将 Kotlin 代码直接编译为原生机器码，这带来了与 JVM Kotlin 截然不同的内存和并发模型：

```
┌──────────────────────┐    ┌─────────────────────────┐
│   Kotlin/JVM         │    │   Kotlin/Native          │
│                      │    │                          │
│  JVM 字节码          │    │  原生机器码               │
│  JVM GC（分代）      │    │  KN GC（并发标记-清除）   │
│  线程共享内存（自由） │    │  线程间对象传递需 freeze  │
│  反射（运行时）       │    │  无运行时反射             │
│  动态类加载          │    │  静态链接                 │
│  JNI 跨语言调用      │    │  cinterop 直接绑定 C API │
└──────────────────────┘    └─────────────────────────┘
```

**KN 的关键约束（旧版本，< 1.7.20）：**
- 对象在被跨线程传递前必须 `freeze()`（冻结后不可变）
- 主线程（UI 线程）与后台线程不能直接共享 Kotlin 对象

**KN 的新内存模型（≥ 1.7.20 + Kotlin 1.9+）：**
- 取消了强制 `freeze()` 要求
- 允许多线程之间自由共享 Kotlin 对象（配合 Mutex）
- 但 C 侧的原始指针仍需手动管理生命周期

### 1.2 KN 内存区域划分

```
┌────────────────────────────────────────────────────┐
│  Kotlin/Native 进程内存                             │
│                                                    │
│  ┌──────────────┐  ┌────────────────────────────┐  │
│  │  KN 堆       │  │  C 内存（malloc/free）      │  │
│  │  Kotlin 对象 │  │  通过 cinterop 访问         │  │
│  │  GC 管理     │  │  需要手动 free 或用 Arena   │  │
│  └──────────────┘  └────────────────────────────┘  │
│                                                    │
│  ┌────────────────────────────────────────────┐    │
│  │  StableRef 区域                             │    │
│  │  固定 Kotlin 对象，使其地址对 C 回调可见    │    │
│  └────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────┘
```

---

## 二、cinterop 工作原理

### 2.1 从 C 头文件到 Kotlin 绑定的转换流程

```
KrKr2 C API 头文件（krkr2_ui_api.h）
            │
            │ cinterop 工具读取
            ▼
   .def 配置文件（krkr2.def）
            │
            │ Kotlin 编译器处理
            ▼
   自动生成的 Kotlin 绑定（krkr2.klib）
            │
            │ Kotlin 代码 import
            ▼
   Kotlin 代码调用 C 函数（如调用本地函数一样）
```

### 2.2 cinterop 支持的 C 特性

| C 特性 | cinterop 处理方式 |
|---|---|
| 基本类型（int, float, char*） | 直接映射到 Kotlin 原始类型或 `CPointer` |
| 结构体 | 生成对应的 Kotlin `CStructVar` 子类 |
| 枚举 | 生成 Kotlin `Int` 常量（或 enum class） |
| 函数指针 | 生成对应函数类型的 `CPointer` |
| 宏定义（简单常量） | 转换为 Kotlin `const val` |
| `#include` 的类型 | 递归处理依赖头文件 |
| C++ 类 | **不支持**，必须提供 C 封装接口 |

> **关键限制**：cinterop 只能绑定 C（不是 C++）接口。KrKr2 的 C++ 类必须提供额外的 `extern "C"` 封装 API。

---

## 三、为 KrKr2 设计 C 封装 API

### 3.1 原则：最小化 C 接口

cinterop 要求 C 风格接口，因此需要为 KrKr2 的 UI 抽象层设计一套 C 封装：

```c
// cpp/core/environ/ui/krkr2_ui_c_api.h
// KrKr2 UI C API — 供 Kotlin/Native cinterop 使用
//
// 设计原则：
//   1. 所有函数使用 extern "C" 导出
//   2. 使用不透明指针（void*）表示 C++ 对象
//   3. 参数和返回值只使用基本类型、C 字符串、不透明指针
//   4. 调用者负责释放带 _alloc 后缀的函数返回的字符串

#pragma once
#include <stdint.h>
#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

// ─────────────────────────────────────────────────
// UIManager 生命周期
// ─────────────────────────────────────────────────

/** 获取全局 UIManager 实例（不拥有所有权） */
void* krkr2_ui_get_manager(void);

/** 初始化 UIManager（程序启动时调用一次） */
bool krkr2_ui_initialize(void);

/** 销毁 UIManager */
void krkr2_ui_shutdown(void);

// ─────────────────────────────────────────────────
// 对话框
// ─────────────────────────────────────────────────

/** 显示设置对话框（阻塞直到用户关闭） */
void krkr2_ui_show_config_dialog(void* manager);

/** 
 * 显示游戏选择对话框
 * @return  用户选择的路径（调用者必须调用 krkr2_free_string 释放），
 *          用户取消时返回 NULL
 */
char* krkr2_ui_show_game_select(void* manager);

/**
 * 显示文件选择对话框
 * @param filter   文件过滤器（如 "*.ks;*.tjs"）
 * @return  选中的路径，需要 krkr2_free_string 释放；取消返回 NULL
 */
char* krkr2_ui_show_file_picker(void* manager, const char* filter);

// ─────────────────────────────────────────────────
// Toast 通知
// ─────────────────────────────────────────────────

/** Toast 类型 */
typedef enum {
    KRKR2_TOAST_INFO    = 0,
    KRKR2_TOAST_WARNING = 1,
    KRKR2_TOAST_ERROR   = 2,
} Krkr2ToastType;

/**
 * 显示 Toast 消息
 * @param message   UTF-8 消息文本
 * @param type      消息类型
 * @param duration_ms 显示时长（毫秒）
 */
void krkr2_ui_show_toast(void* manager,
                          const char* message,
                          Krkr2ToastType type,
                          int duration_ms);

// ─────────────────────────────────────────────────
// 系统信息
// ─────────────────────────────────────────────────

/** 
 * 获取 KrKr2 版本字符串
 * @return  版本字符串，需要 krkr2_free_string 释放
 */
char* krkr2_get_version(void);

/** 
 * 获取当前平台名称（"windows" | "android" | "macos" | "linux"）
 * @return  平台名字符串，需要 krkr2_free_string 释放
 */
char* krkr2_get_platform(void);

// ─────────────────────────────────────────────────
// 回调注册（C++ → Kotlin）
// ─────────────────────────────────────────────────

/** 游戏事件回调类型 */
typedef void (*Krkr2GameEventCallback)(
    const char* event_type,   ///< 事件类型（如 "game_loaded"）
    const char* event_data,   ///< JSON 格式的事件数据（可为 NULL）
    void* user_data           ///< 透传的用户数据指针
);

/**
 * 注册游戏事件监听器
 * @param callback   事件回调函数指针
 * @param user_data  透传给回调的用户数据（通常是 Kotlin StableRef）
 */
void krkr2_register_game_event_callback(
    Krkr2GameEventCallback callback,
    void* user_data
);

/** 取消注册游戏事件监听器 */
void krkr2_unregister_game_event_callback(void);

// ─────────────────────────────────────────────────
// 内存管理
// ─────────────────────────────────────────────────

/**
 * 释放由 KrKr2 分配的字符串
 * 所有返回 char* 的函数都需要调用此函数释放
 */
void krkr2_free_string(char* str);

#ifdef __cplusplus
} // extern "C"
#endif
```

### 3.2 C 封装的 C++ 实现

```cpp
// cpp/core/environ/ui/krkr2_ui_c_api.cpp
#include "krkr2_ui_c_api.h"
#include "abstract/IUIManager.h"
#include <cstring>
#include <cstdlib>
#include <memory>

// 全局 UIManager 实例（简化示例）
static krkr2::ui::IUIManager* g_ui_manager = nullptr;

// 回调函数指针（全局，KrKr2 同一时刻只支持一个 Kotlin 监听器）
static Krkr2GameEventCallback g_event_callback = nullptr;
static void* g_event_user_data = nullptr;

extern "C" {

void* krkr2_ui_get_manager(void) {
    return static_cast<void*>(g_ui_manager);
}

bool krkr2_ui_initialize(void) {
    // 在实际代码中从 UIBackendFactory 获取实例
    // g_ui_manager = UIBackendFactory::create();
    return g_ui_manager != nullptr;
}

void krkr2_ui_shutdown(void) {
    if (g_ui_manager) {
        g_ui_manager->shutdown();
        g_ui_manager = nullptr;
    }
}

void krkr2_ui_show_config_dialog(void* manager) {
    auto* mgr = static_cast<krkr2::ui::IUIManager*>(manager);
    if (mgr) mgr->showConfigDialog();
}

char* krkr2_ui_show_game_select(void* manager) {
    auto* mgr = static_cast<krkr2::ui::IUIManager*>(manager);
    if (!mgr) return nullptr;
    
    std::string path = mgr->showGameSelectDialog();
    if (path.empty()) return nullptr;
    
    // 分配 C 字符串（调用者负责用 krkr2_free_string 释放）
    char* result = static_cast<char*>(malloc(path.size() + 1));
    memcpy(result, path.c_str(), path.size() + 1);
    return result;
}

char* krkr2_ui_show_file_picker(void* manager, const char* filter) {
    auto* mgr = static_cast<krkr2::ui::IUIManager*>(manager);
    if (!mgr) return nullptr;
    
    std::string path = mgr->showFilePicker(filter ? filter : "");
    if (path.empty()) return nullptr;
    
    char* result = static_cast<char*>(malloc(path.size() + 1));
    memcpy(result, path.c_str(), path.size() + 1);
    return result;
}

void krkr2_ui_show_toast(void* manager, const char* message,
                          Krkr2ToastType type, int duration_ms) {
    auto* mgr = static_cast<krkr2::ui::IUIManager*>(manager);
    if (!mgr || !message) return;
    
    krkr2::ui::ToastType kt;
    switch (type) {
        case KRKR2_TOAST_WARNING: kt = krkr2::ui::ToastType::Warning; break;
        case KRKR2_TOAST_ERROR:   kt = krkr2::ui::ToastType::Error;   break;
        default:                  kt = krkr2::ui::ToastType::Info;    break;
    }
    mgr->showToast(message, kt, duration_ms);
}

char* krkr2_get_version(void) {
    const char* ver = "2.0.0-alpha";
    char* result = static_cast<char*>(malloc(strlen(ver) + 1));
    strcpy(result, ver);
    return result;
}

char* krkr2_get_platform(void) {
#if defined(_WIN32)
    const char* platform = "windows";
#elif defined(__ANDROID__)
    const char* platform = "android";
#elif defined(__APPLE__)
    const char* platform = "macos";
#else
    const char* platform = "linux";
#endif
    char* result = static_cast<char*>(malloc(strlen(platform) + 1));
    strcpy(result, platform);
    return result;
}

void krkr2_register_game_event_callback(
    Krkr2GameEventCallback callback, void* user_data)
{
    g_event_callback = callback;
    g_event_user_data = user_data;
}

void krkr2_unregister_game_event_callback(void) {
    g_event_callback = nullptr;
    g_event_user_data = nullptr;
}

void krkr2_free_string(char* str) {
    free(str);
}

} // extern "C"

// C++ 内部调用：触发 Kotlin 回调
namespace krkr2 {
namespace ui {

void fireGameEvent(const char* event_type, const char* event_data) {
    if (g_event_callback) {
        g_event_callback(event_type, event_data, g_event_user_data);
    }
}

} // namespace ui
} // namespace krkr2
```

---

## 四、cinterop 配置文件（.def）

### 4.1 krkr2.def 文件详解

```ini
# kotlin_module/src/nativeInterop/cinterop/krkr2.def
# 
# cinterop 配置文件语法说明：
#   headers       — 要处理的头文件列表
#   headerFilter  — 只导出匹配 glob 的头文件中的符号（避免导入系统头文件符号）
#   compilerOpts  — 额外的编译器选项（传给 clang）
#   linkerOpts    — 链接时选项
#   staticLibraries — 要静态链接的库
#   libraryPaths  — 库文件搜索路径

# 头文件：指定要绑定的 C API
headers = krkr2_ui_c_api.h

# 只导出来自 krkr2_ui_c_api.h 的符号，不导出 stdint.h 等系统头文件的符号
headerFilter = krkr2_*

# 头文件搜索路径（相对于项目根目录）
compilerOpts = -I../../cpp/core/environ/ui

# 平台相关链接选项
# Linux:
linkerOpts.linux_x64 = -L../../build/linux -lkrkr2_ui
# Windows（.lib 文件）:
linkerOpts.mingw_x64 = -L../../build/windows -lkrkr2_ui
# macOS:
linkerOpts.macos_x64 = -L../../build/macos -lkrkr2_ui -framework CoreFoundation
```

### 4.2 Gradle 配置

```kotlin
// kotlin_module/build.gradle.kts
plugins {
    kotlin("multiplatform") version "1.9.22"
}

kotlin {
    // 目标平台（根据项目需要选择）
    linuxX64("linux")
    mingwX64("windows")
    macosX64("macos")
    
    // Android 目标（使用 JVM Kotlin + JNI，不是纯 KN）
    // androidTarget()  ← 此行不同于 Compose 章，详见第04章

    targets.withType<org.jetbrains.kotlin.gradle.plugin.mpp.KotlinNativeTarget> {
        compilations["main"].cinterops {
            // 声明 cinterop 任务，名称对应 .def 文件名
            create("krkr2") {
                defFile(project.file("src/nativeInterop/cinterop/krkr2.def"))
                
                // 可在此处覆盖 .def 文件中的设置
                includeDirs("../../cpp/core/environ/ui")
                
                // 额外的编译器选项
                compilerOpts("-std=c11")
            }
        }
        
        compilations["main"].kotlinOptions {
            // 启用更激进的优化（release 构建）
            freeCompilerArgs += listOf("-opt")
        }
        
        binaries {
            // 输出动态库（供 KrKr2 C++ 加载）
            sharedLib("krkr2_compose_ui") {
                baseName = "krkr2_compose_ui"
                // 导出的顶层函数（从 C++ 调用 Kotlin）
                export(project(":compose_ui"))
            }
        }
    }
    
    sourceSets {
        val commonMain by getting {
            dependencies {
                // Compose Multiplatform（第04章使用）
                // implementation(compose.runtime)
            }
        }
        
        val linuxMain by getting {
            dependencies {}
        }
        
        val windowsMain by getting {
            dependencies {}
        }
    }
}
```

---

## 五、在 Kotlin 中调用 C API

### 5.1 cinterop 生成的 Kotlin 绑定示例

cinterop 处理 `krkr2.def` 后，会生成如下 Kotlin 代码（自动生成，无需手写）：

```kotlin
// 自动生成（仅供参考，实际在 .klib 中）
package krkr2  // 由 .def 文件名决定

// C 枚举 → Kotlin enum-like 常量
object Krkr2ToastType {
    const val KRKR2_TOAST_INFO    = 0
    const val KRKR2_TOAST_WARNING = 1
    const val KRKR2_TOAST_ERROR   = 2
}

// C 函数 → Kotlin 函数（通过 interop stub）
external fun krkr2_ui_get_manager(): COpaquePointer?
external fun krkr2_ui_initialize(): Boolean
external fun krkr2_ui_shutdown()
external fun krkr2_ui_show_config_dialog(manager: COpaquePointer?)
external fun krkr2_ui_show_game_select(manager: COpaquePointer?): CPointer<ByteVar>?
external fun krkr2_ui_show_toast(
    manager: COpaquePointer?,
    message: CPointer<ByteVar>?,
    type: Int,
    duration_ms: Int
)
external fun krkr2_free_string(str: CPointer<ByteVar>?)
```

### 5.2 封装为安全的 Kotlin 类

在实际使用时，应在 cinterop 生成的绑定之上封装一个 Kotlin 友好的类：

```kotlin
// kotlin_module/src/commonMain/kotlin/KrKr2UIManager.kt
import krkr2.*
import kotlinx.cinterop.*

/**
 * KrKr2UIManager — 对 C API 的 Kotlin 友好封装
 *
 * 负责：
 *   1. 管理 C 字符串的生命周期（自动 free）
 *   2. 将 C 的不透明指针转换为 Kotlin 语义
 *   3. 处理 null 安全
 */
class KrKr2UIManager {
    
    // 不透明的 C++ UIManager 指针（从 C API 获取）
    private val nativeManager: COpaquePointer? = krkr2_ui_get_manager()
    
    init {
        require(nativeManager != null) {
            "KrKr2 UIManager not initialized. Call krkr2_ui_initialize() first."
        }
    }
    
    /** 显示设置对话框（阻塞） */
    fun showConfigDialog() {
        krkr2_ui_show_config_dialog(nativeManager)
    }
    
    /**
     * 显示游戏选择对话框
     * @return  选中的路径，用户取消返回 null
     */
    fun showGameSelect(): String? {
        // krkr2_ui_show_game_select 返回需要手动 free 的 char*
        val rawPtr = krkr2_ui_show_game_select(nativeManager) ?: return null
        
        // toKString() 将 C 字符串转换为 Kotlin String（复制数据）
        val result = rawPtr.toKString()
        
        // 释放 C 字符串（Kotlin String 已复制，原始 C 内存可以安全释放）
        krkr2_free_string(rawPtr)
        
        return result
    }
    
    /**
     * 显示文件选择器
     * @param filter  文件过滤器，如 "*.ks;*.tjs"
     * @return  选中路径，取消返回 null
     */
    fun showFilePicker(filter: String = "*.*"): String? {
        // 将 Kotlin String 转为 C 字符串（在 memScoped 块中自动管理生命周期）
        return memScoped {
            val filterC = filter.cstr.ptr
            val rawPtr = krkr2_ui_show_file_picker(nativeManager, filterC)
                ?: return@memScoped null
            
            val result = rawPtr.toKString()
            krkr2_free_string(rawPtr)
            result
        }
    }
    
    /** 显示信息 Toast */
    fun showInfoToast(message: String, durationMs: Int = 3000) {
        memScoped {
            krkr2_ui_show_toast(
                nativeManager,
                message.cstr.ptr,
                Krkr2ToastType.KRKR2_TOAST_INFO,
                durationMs
            )
        }
    }
    
    /** 显示错误 Toast */
    fun showErrorToast(message: String, durationMs: Int = 5000) {
        memScoped {
            krkr2_ui_show_toast(
                nativeManager,
                message.cstr.ptr,
                Krkr2ToastType.KRKR2_TOAST_ERROR,
                durationMs
            )
        }
    }
}
```

### 5.3 C 回调中使用 StableRef（C++ 调用 Kotlin）

当 C++ 需要回调 Kotlin 代码时（如游戏事件通知），需要将 Kotlin lambda 固定在内存中：

```kotlin
// kotlin_module/src/commonMain/kotlin/EventBridge.kt
import krkr2.*
import kotlinx.cinterop.*

/**
 * EventBridge — 管理从 C++ 到 Kotlin 的事件回调
 *
 * 核心问题：C 函数指针只能持有函数地址和一个 void* 用户数据。
 * 要将 Kotlin lambda 传给 C，需要：
 *   1. 将 lambda 包装在 StableRef 中（固定内存地址，避免 GC 移动）
 *   2. 把 StableRef 的原始地址作为 void* user_data 传给 C
 *   3. 在 C 回调中，从 void* 还原 StableRef，再调用 lambda
 */
class EventBridge(
    private val onEvent: (type: String, data: String?) -> Unit
) {
    // StableRef 持有 lambda 的固定引用（防止 GC 回收）
    private val stableRef: StableRef<(String, String?) -> Unit> =
        StableRef.create(onEvent)
    
    /** 向 KrKr2 注册事件回调 */
    fun register() {
        // C 回调函数（静态函数指针）
        val callback: Krkr2GameEventCallback = staticCFunction {
            event_type, event_data, user_data ->
            
            // 从 void* user_data 还原 Kotlin lambda
            val ref = user_data!!.asStableRef<(String, String?) -> Unit>()
            val handler = ref.get()
            
            // 调用 Kotlin lambda
            handler(
                event_type?.toKString() ?: "",
                event_data?.toKString()
            )
        }
        
        krkr2_register_game_event_callback(
            callback,
            stableRef.asCPointer()  // 将 StableRef 转为 void*
        )
    }
    
    /** 注销事件回调并释放 StableRef */
    fun unregister() {
        krkr2_unregister_game_event_callback()
        stableRef.dispose()  // 必须显式释放，否则 lambda 永远不会被 GC
    }
}

// 使用示例：
fun setupEventBridge() {
    val bridge = EventBridge { type, data ->
        when (type) {
            "game_loaded" -> println("游戏已加载: $data")
            "game_error"  -> println("游戏错误: $data")
        }
    }
    bridge.register()
    
    // 程序退出时记得调用：
    // bridge.unregister()
}
```

---

## 六、内存安全最佳实践

### 6.1 C 字符串的黄金规则

```kotlin
// ✅ 正确：使用 memScoped 传入 C 字符串
fun correctUsage(input: String) {
    memScoped {
        // input.cstr.ptr 在 memScoped 退出时自动释放
        val cStr = input.cstr.ptr
        someCFunction(cStr)
    }
    // 退出 memScoped 后，cStr 指向的内存已释放，不能再使用
}

// ✅ 正确：接收 C 分配的字符串，立即复制后 free
fun receiveString(): String? {
    val rawPtr = krkr2_get_version() ?: return null
    val result = rawPtr.toKString()  // 复制到 Kotlin String
    krkr2_free_string(rawPtr)        // 释放 C 内存
    return result
}

// ❌ 错误：使用 cstr.ptr 超出 memScoped 作用域
fun wrongUsage(): CPointer<ByteVar> {
    return memScoped {
        "hello".cstr.ptr  // ← 内存在 memScoped 退出后释放！
        // 返回悬空指针 — 未定义行为
    }
}

// ❌ 错误：忘记 free C 分配的字符串（内存泄漏）
fun leakExample() {
    val rawPtr = krkr2_get_version() ?: return
    val str = rawPtr.toKString()
    // 忘记调用 krkr2_free_string(rawPtr) ← 内存泄漏！
}
```

### 6.2 StableRef 生命周期管理

```kotlin
// StableRef 必须在适当时机 dispose()
// 否则被引用的 Kotlin 对象永远不会被 GC

class Resource : AutoCloseable {
    private val callback: (String) -> Unit = { msg -> println(msg) }
    private val ref = StableRef.create(callback)
    
    val rawPointer: COpaquePointer = ref.asCPointer()
    
    override fun close() {
        // 配合 use {} 块自动调用
        ref.dispose()
    }
}

// 推荐用法：
Resource().use { res ->
    krkr2_register_game_event_callback(
        staticCFunction { _, data, ud ->
            ud!!.asStableRef<(String) -> Unit>().get()(
                data?.toKString() ?: ""
            )
        },
        res.rawPointer
    )
    // 使用完毕后，use 块自动调用 close() → dispose()
}
```

---

## 七、动手实践

### 实践 7.1：运行第一个 cinterop 绑定

```bash
# 在 kotlin_module/ 目录下
cd krkr2/kotlin_module

# 生成 cinterop 绑定并编译（Linux 目标）
./gradlew linkLinuxX64  # 或 linkDebugSharedLinux

# 查看 cinterop 生成的 .klib 文件
ls build/classes/kotlin/linux/main/kinterop/
# 应该看到 krkr2-cinterop-krkr2.klib

# 运行单元测试
./gradlew linuxX64Test
```

### 实践 7.2：验证内存安全

```kotlin
// 在 src/linuxMain/kotlin/MemoryTest.kt 中
import krkr2.*
import kotlinx.cinterop.*
import kotlin.test.*

class MemoryTest {
    
    @Test
    fun testStringRoundtrip() {
        // 测试：调用 C API → 接收 C 字符串 → 正确释放
        val rawPtr = krkr2_get_platform() 
            ?: fail("krkr2_get_platform returned null")
        
        val platform = rawPtr.toKString()
        krkr2_free_string(rawPtr)
        
        assertNotNull(platform)
        assertContains(listOf("windows", "android", "macos", "linux"), platform)
    }
    
    @Test
    fun testMemScopedDoesNotLeak() {
        // 测试：memScoped 中传入字符串不泄漏
        // 此测试主要靠 Valgrind/AddressSanitizer 验证，这里只验证功能
        repeat(1000) {
            memScoped {
                val cStr = "test_message_$it".cstr.ptr
                // 模拟调用（此处只是验证不崩溃）
                @Suppress("UNUSED_VARIABLE")
                val _ = cStr.toKString()
            }
        }
        // 如果有内存泄漏，重复1000次后会明显
    }
}
```

---

## 对照项目源码

```
krkr2/
├── cpp/core/environ/ui/
│   ├── krkr2_ui_c_api.h           ← KrKr2 UI C 封装 API（头文件）
│   └── krkr2_ui_c_api.cpp         ← KrKr2 UI C 封装实现
└── kotlin_module/
    ├── build.gradle.kts            ← Gradle 多平台配置（含 cinterop）
    └── src/
        ├── nativeInterop/
        │   └── cinterop/
        │       └── krkr2.def       ← cinterop 配置文件
        └── commonMain/kotlin/
            ├── KrKr2UIManager.kt   ← C API 的 Kotlin 安全封装
            └── EventBridge.kt      ← StableRef 管理的事件回调桥接
```

---

## 本节小结

- Kotlin/Native 通过 cinterop 工具从 C 头文件自动生成 Kotlin 绑定，无需手写 JNI 代码
- cinterop **只支持 C**（不支持 C++），KrKr2 的 C++ 接口必须用 `extern "C"` 封装后才能绑定
- `CPointer<ByteVar>` 对应 `char*`，`COpaquePointer` 对应 `void*`，`toKString()` 将 C 字符串复制为 Kotlin String
- `memScoped {}` 自动管理在其中分配的 C 内存（退出时释放），不可将内部指针泄漏到作用域外
- C 分配的字符串（如 `krkr2_get_version()` 返回的）必须手动调用 `krkr2_free_string()` 释放
- `StableRef.create(lambda)` 将 Kotlin lambda 固定在内存中，使其地址可安全传递给 C 函数指针；使用完后必须调用 `dispose()`

---

## 练习题与答案

**题 1：** 下面的代码有什么问题？如何修复？
```kotlin
fun getGamePath(): String {
    val manager = krkr2_ui_get_manager()
    val pathPtr = krkr2_ui_show_file_picker(manager, "*.ks".cstr.ptr)
    return pathPtr?.toKString() ?: ""
}
```

<details>
<summary>查看答案</summary>

**问题一：** `"*.ks".cstr.ptr` 在 `memScoped` 外部使用，指向的 C 内存可能已被释放（悬空指针），属于未定义行为。

**问题二：** `krkr2_ui_show_file_picker` 返回的 `pathPtr` 是 C 分配的字符串，没有调用 `krkr2_free_string` 释放，造成内存泄漏。

**修复：**
```kotlin
fun getGamePath(): String {
    val manager = krkr2_ui_get_manager()
    return memScoped {
        val filterPtr = "*.ks".cstr.ptr  // 在 memScoped 内使用
        val pathPtr = krkr2_ui_show_file_picker(manager, filterPtr)
            ?: return@memScoped ""
        val result = pathPtr.toKString()
        krkr2_free_string(pathPtr)  // 释放 C 分配的字符串
        result
    }
}
```

</details>

---

**题 2：** 为什么 cinterop 不能直接绑定 C++ 类（如 `IUIManager`）？从语言设计角度解释。

<details>
<summary>查看答案</summary>

C++ 类在 ABI（二进制接口）层面比 C 复杂得多：
1. **名称修饰（Name Mangling）**：C++ 函数名在编译后会被编译器修饰（不同编译器格式不同），cinterop 无法解析
2. **虚函数表（vtable）**：虚函数通过指针间接调用，布局依赖编译器实现
3. **构造/析构函数**：涉及 RAII 和异常处理机制，C 没有等价概念
4. **异常（exception）**：C++ 异常跨语言传播是未定义行为
5. **模板**：C++ 模板在编译时展开，无运行时表示

C 语言的 ABI 则是稳定的：普通函数使用 cdecl 或 stdcall 约定，结构体布局规则明确，没有隐式操作（无构造函数、析构函数）。这就是为什么几乎所有跨语言 FFI（包括 cinterop、JNI、Python ctypes）都要求先封装为 C 接口。

</details>

---

**题 3：** `StableRef<T>.dispose()` 如果被调用两次会发生什么？如何防止这种情况？

<details>
<summary>查看答案</summary>

`StableRef.dispose()` 第二次调用会导致**双重释放（double free）**，这是未定义行为，通常导致程序崩溃（段错误）或堆内存损坏。

**防范措施：**

1. **使用 `AutoCloseable` + `use {}`**（最推荐）：
   ```kotlin
   class SafeRef<T : Any>(value: T) : AutoCloseable {
       private val ref = StableRef.create(value)
       private var disposed = false
       val pointer: COpaquePointer get() = ref.asCPointer()
       override fun close() {
           if (!disposed) {
               disposed = true
               ref.dispose()
           }
       }
   }
   SafeRef(myLambda).use { safeRef ->
       // 使用 safeRef.pointer
   }
   ```

2. **设置 nullable 标志**：
   ```kotlin
   private var stableRef: StableRef<T>? = StableRef.create(value)
   fun dispose() {
       stableRef?.dispose()
       stableRef = null
   }
   ```

3. **使用 `AtomicReference`（多线程安全）**：
   ```kotlin
   private val stableRef = atomic<StableRef<T>?>(StableRef.create(value))
   fun dispose() {
       stableRef.getAndSet(null)?.dispose()
   }
   ```

</details>

---

## 下一步

本节完成了 Kotlin/Native cinterop 的完整配置，建立了从 C++ 到 Kotlin 的类型安全绑定。下一节 [02-Compose双端适配与C++互调](02-Compose双端适配与C++互调.md) 将在此基础上集成 Compose Multiplatform，构建真正的 UI 界面。
