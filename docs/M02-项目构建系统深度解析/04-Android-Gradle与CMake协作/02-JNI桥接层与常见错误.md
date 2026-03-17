# gradle.properties：全局构建属性
## gradle.properties：全局构建属性

```properties
# 文件：platforms/android/gradle.properties（31 行）
org.gradle.jvmargs=-Xmx6g
android.useAndroidX=true
android.compileSdk=34
android.minSdk=29
android.targetSdk=34
```

关键配置：

| 属性 | 值 | 说明 |
|------|---|------|
| `org.gradle.jvmargs=-Xmx6g` | JVM 最大堆内存 6GB | C++ 编译消耗大量内存，默认 1-2GB 不够 |
| `android.useAndroidX` | `true` | 使用 AndroidX 库（替代旧的 Support Library） |

> **6GB 堆内存的必要性：** KrKr2 的 Android 构建需要同时编译两个 ABI 的 C++ 代码（上百个源文件），加上 Gradle 自身、Java 编译、资源处理等任务，内存消耗很高。如果设置过低，会出现 `OutOfMemoryError` 或 GC 频繁导致构建极慢。

---

## JNI 桥接层：krkr2_android.cpp

Android 的 Java 虚拟机不能直接执行 C++ 代码。JNI（Java Native Interface）是连接两者的桥梁。

### 完整启动流程

```
Android 系统启动应用
    │
    ├─ Zygote fork 新进程
    ├─ 创建 Application 实例
    ├─ 启动主 Activity
    │
    └─ Activity.onCreate()
        │
        ├─ System.loadLibrary("krkr2")  ← 加载 libkrkr2.so
        │   │
        │   └─ 自动调用 JNI_OnLoad()    ← krkr2_android.cpp 入口
        │       ├─ 初始化 spdlog 日志
        │       ├─ 设置 breakpad 崩溃处理
        │       └─ 注册 JNI 方法
        │
        └─ 调用 native 方法启动引擎
            └─ TVPAppDelegate::run()
                └─ Cocos2d-x 事件循环
```

### krkr2_android.cpp 关键代码分析

```cpp
// 文件：platforms/android/cpp/krkr2_android.cpp（简化核心逻辑）

#include <jni.h>
#include <dlfcn.h>
#include <spdlog/spdlog.h>
#include <spdlog/sinks/android_sink.h>

// breakpad 崩溃报告（Android 专用）
#include "client/linux/handler/exception_handler.h"

// 全局 breakpad 处理器
static google_breakpad::ExceptionHandler* g_handler = nullptr;

// breakpad 崩溃回调 — native crash 时生成 minidump
static bool crashCallback(const google_breakpad::MinidumpDescriptor& descriptor,
                          void* context, bool succeeded) {
    spdlog::critical("Native crash! Minidump: {}", descriptor.path());
    return succeeded;
}

// JNI_OnLoad — libkrkr2.so 被加载时自动调用
extern "C" JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    // 1. 初始化日志系统（输出到 Android logcat）
    auto android_sink = std::make_shared<spdlog::sinks::android_sink_mt>("krkr2");
    auto logger = std::make_shared<spdlog::logger>("core", android_sink);
    spdlog::set_default_logger(logger);
    spdlog::set_level(spdlog::level::debug);

    spdlog::info("KrKr2 native library loaded");

    // 2. 初始化 breakpad（崩溃报告）
    google_breakpad::MinidumpDescriptor descriptor("/data/local/tmp");
    g_handler = new google_breakpad::ExceptionHandler(
        descriptor, nullptr, crashCallback, nullptr, true, -1);

    // 3. 动态加载 SDL2 的 JNI_OnLoad
    void* sdl_handle = dlopen("libSDL2.so", RTLD_LAZY);
    if (sdl_handle) {
        using JNI_OnLoad_t = jint (*)(JavaVM*, void*);
        auto sdl_onload = (JNI_OnLoad_t)dlsym(sdl_handle, "JNI_OnLoad");
        if (sdl_onload) {
            sdl_onload(vm, reserved);  // 初始化 SDL2 JNI
        }
    }

    return JNI_VERSION_1_4;
}
```

#### 关键设计决策

**1. spdlog Android Sink：**

```cpp
auto android_sink = std::make_shared<spdlog::sinks::android_sink_mt>("krkr2");
```

桌面平台的 spdlog 输出到终端（`stdout_sink`），Android 平台输出到 **logcat**（Android 系统日志）。`android_sink_mt` 是线程安全版本（`mt` = multi-threaded）。在 Android Studio 的 Logcat 面板中，用 `krkr2` 标签过滤即可看到所有日志。

**2. Breakpad 崩溃报告：**

```cpp
google_breakpad::ExceptionHandler(descriptor, nullptr, crashCallback, nullptr, true, -1);
```

Android 的 native crash（段错误、空指针等）不会产生 Java 异常堆栈。如果不使用 breakpad：
- 应用直接闪退，logcat 只有一行 `Signal 11 (SIGSEGV)`
- 无法知道崩溃发生在哪个 C++ 函数

Breakpad 会在崩溃时生成 minidump 文件，后续可以用 `minidump_stackwalk` 工具分析完整的 native 调用栈。

**3. 动态加载 SDL2 JNI：**

```cpp
void* sdl_handle = dlopen("libSDL2.so", RTLD_LAZY);
```

SDL2 有自己的 `JNI_OnLoad` 函数需要调用。但一个 `.so` 文件只能有一个 `JNI_OnLoad` 入口点。KrKr2 通过 `dlopen` + `dlsym` 手动调用 SDL2 的 `JNI_OnLoad`，避免符号冲突。

---

## 常见错误与解决方案

### 错误 1：NDK 版本不匹配

```
NDK not configured. Download it with SDK manager.
NDK version '28.0.13004108' specified in build.gradle is not available.
```

**原因：** `ndkVersion` 指定的版本未安装。

**解决：**
```bash
# 方法 1：用 SDK Manager 安装
sdkmanager "ndk;28.0.13004108"

# 方法 2：用 Android Studio
# File → Settings → Languages & Frameworks → Android SDK → SDK Tools → NDK (Side by side)
```

### 错误 2：CMake 版本不匹配

```
CMake '3.31.1' was not found in PATH or by cmake.dir property.
```

**解决：**
```bash
# 安装 CMake（通过 SDK Manager）
sdkmanager "cmake;3.31.1"

# 或在 local.properties 中指定路径
echo "cmake.dir=/path/to/cmake/3.31.1" >> platforms/android/local.properties
```

### 错误 3：VCPKG_ROOT 未设置

```
FATAL_ERROR: 请设置 VCPKG_ROOT 环境变量
```

**解决（需要在 Gradle 构建前设置）：**

```bash
# Linux/macOS（~/.bashrc 或 ~/.zshrc）
export VCPKG_ROOT="$HOME/vcpkg"

# Windows（系统环境变量）
setx VCPKG_ROOT "C:\vcpkg"

# 或在 gradle.properties 中传递
echo "org.gradle.project.VCPKG_ROOT=/path/to/vcpkg" >> gradle.properties
```

### 错误 4：内存不足

```
java.lang.OutOfMemoryError: Java heap space
```

**解决：** 增大 `gradle.properties` 中的 JVM 堆：

```properties
# 从 6g 增大到 8g
org.gradle.jvmargs=-Xmx8g
```

---

## 动手实践

### 练习 1：查看实际的 CMake 调用命令

在项目的 Android 构建输出中查看 Gradle 传给 CMake 的实际参数：

```bash
# 执行构建（加 --info 显示详细日志）
./gradlew -p platforms/android assembleDebug --info 2>&1 | grep "cmake"
```

或者查看 CMake 缓存文件：

```bash
# arm64-v8a 的 CMake 缓存
cat out/android/app/.cxx/Debug/arm64-v8a/CMakeCache.txt | head -50
```

缓存文件中包含所有 CMake 变量的最终值，你可以看到 `ANDROID_ABI`、`CMAKE_TOOLCHAIN_FILE` 等的实际路径。

### 练习 2：添加 armeabi-v7a 支持

如果需要支持 32 位 ARM 设备：

1. 修改 `app/build.gradle`：
```groovy
ndk {
    abiFilters 'arm64-v8a', 'x86_64', 'armeabi-v7a'  // 添加 32 位
}
// 同时更新 splits
splits {
    abi {
        include 'arm64-v8a', 'x86_64', 'armeabi-v7a'
    }
}
```

2. 确保 `vcpkg_android.cmake` 中有对应映射：
```cmake
elseif(ANDROID_ABI STREQUAL "armeabi-v7a")
    set(VCPKG_TARGET_TRIPLET "arm-android")  // 已存在
```

3. 构建时 vcpkg 会自动为 `arm-android` triplet 编译所有依赖（首次较慢）。

### 练习 3：修改 SDK 版本

如果需要支持更老的 Android 设备（如 Android 8.0 / API 26）：

1. 修改 `app/build.gradle`：`minSdk 26`
2. 修改 `gradle.properties`：`android.minSdk=26`
3. 验证代码中没有使用 API 29+ 的 Android API（如 `ScopedStorage`）
4. 重新构建：`./gradlew clean assembleDebug`

---

