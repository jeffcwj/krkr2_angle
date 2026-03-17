# Android Gradle 与 CMake 协作

> **所属模块：** M02-项目构建系统深度解析
> **前置知识：** [P01-Ch4/01-Presets配置详解](../P01-现代CMake与构建工具链/04-CMake-Presets/01-Presets配置详解.md)、[本模块第 01 章](./01-根CMakeLists深度解读.md)、[本模块第 03 章](./03-平台条件编译实现.md)
> **预计阅读时间：** 40 分钟
> **关键源文件：** `platforms/android/build.gradle`、`platforms/android/app/build.gradle`（140 行）、`platforms/android/settings.gradle`、`platforms/android/gradle.properties`

## 本节目标

读完本节后，你将能够：

1. 理解 Android 构建的双层架构——Gradle 作为外层驱动器，CMake 作为内层本地代码编译器
2. 逐行分析 `app/build.gradle` 中 `externalNativeBuild` 的完整配置
3. 理解多 ABI 构建（arm64-v8a / x86_64）的原理和配置方式
4. 掌握 JNI 桥接层的作用和 `krkr2_android.cpp` 的启动流程
5. 理解 Android 签名配置、版本管理和 APK 分包策略
6. 能够修改 Android 构建配置（如添加新 ABI、调整 SDK 版本、添加新依赖）

## Android 构建的双层架构

Android 本地代码（C/C++）项目的构建与桌面平台有根本性区别。桌面平台直接使用 CMake：

```bash
# 桌面平台：CMake 直接驱动一切
cmake --preset "Linux Debug Config"
cmake --build --preset "Linux Debug Build"
```

Android 平台有一个额外的外层——**Gradle**：

```bash
# Android 平台：Gradle 驱动 CMake
./gradlew assembleDebug
# Gradle 内部会调用：
#   cmake -B ... -G Ninja \
#     -DANDROID_ABI=arm64-v8a \
#     -DANDROID_PLATFORM=android-29 \
#     -DCMAKE_TOOLCHAIN_FILE=.../android.toolchain.cmake \
#     ...
#   cmake --build ...
```

**为什么需要 Gradle？**

| 职责 | 桌面 (CMake 直接) | Android (Gradle + CMake) |
|------|-------------------|-------------------------|
| C/C++ 编译 | CMake | CMake（被 Gradle 调用） |
| Java/Kotlin 编译 | 不需要 | Gradle (javac/kotlinc) |
| 资源打包 | 手动/脚本 | Gradle (AAPT2) |
| APK/AAB 生成 | 不需要 | Gradle |
| 签名 | 不需要 | Gradle (apksigner) |
| 依赖管理 (Java) | 不需要 | Gradle (Maven Central) |
| 多 ABI 构建 | 不需要 | Gradle（为每个 ABI 调用一次 CMake） |

简而言之：**Gradle 是 Android 的"总包"，CMake 只负责 C/C++ 部分**。

### 构建流程全景图

```
开发者执行: ./gradlew assembleDebug
    │
    ├─ Gradle 解析 settings.gradle
    │   └─ 发现 3 个子项目: :sdl, :cocos2dx, :app
    │
    ├─ Gradle 配置阶段 (读取所有 build.gradle)
    │
    └─ Gradle 执行阶段
        │
        ├─ :sdl       → 编译 SDL2 Java 绑定 + native 库
        ├─ :cocos2dx   → 编译 Cocos2d-x Java 层
        │
        └─ :app        → 主应用
            │
            ├─ 1. externalNativeBuild (调用 CMake)
            │   ├─ ABI: arm64-v8a
            │   │   └─ cmake → ninja → libkrkr2.so
            │   └─ ABI: x86_64
            │       └─ cmake → ninja → libkrkr2.so
            │
            ├─ 2. 编译 Java/Kotlin 源码
            │   └─ javac → .class → dex
            │
            ├─ 3. 处理资源和清单
            │   └─ aapt2 → resources.arsc + AndroidManifest.xml
            │
            ├─ 4. 打包 APK
            │   └─ 合并 dex + .so + resources → krkr2.apk
            │
            └─ 5. 签名
                └─ apksigner → krkr2-signed.apk
```

---

## settings.gradle：项目结构声明

```groovy
// 文件：platforms/android/settings.gradle（24 行）
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

rootProject.name = 'krkr2'

include ':sdl'
project(':sdl').projectDir = new File('../../external/libsdl/')

include ':cocos2dx'
project(':cocos2dx').projectDir = new File('../../external/libcocos2dx/')

include ':app'
```

### 逐段分析

**`pluginManagement`** — 声明 Gradle 插件的下载源。`google()` 提供 Android Gradle Plugin (AGP)，`mavenCentral()` 提供通用 Java/Kotlin 插件。

**`include` + `projectDir`** — 声明三个子项目：

| 子项目 | 实际目录 | 产物 | 说明 |
|--------|---------|------|------|
| `:sdl` | `external/libsdl/` | SDL2 Java 绑定 + JNI native | Simple DirectMedia Layer，负责窗口/输入/音频 |
| `:cocos2dx` | `external/libcocos2dx/` | Cocos2d-x Java 层 | 渲染框架的 Android Java 部分 |
| `:app` | `platforms/android/app/` | KrKr2 主 APK | 包含 JNI 桥接、CMake 本地构建、资源 |

注意 `:sdl` 和 `:cocos2dx` 的 `projectDir` 使用了相对路径 `../../external/`，从 `platforms/android/` 回退两级到项目根目录。

---

## 根 build.gradle：全局插件版本

```groovy
// 文件：platforms/android/build.gradle（21 行）
buildscript {
    repositories {
        google()
        mavenCentral()
    }
}

plugins {
    id 'com.android.application' version '8.7.3' apply false
    id 'org.jetbrains.kotlin.android' version '1.9.22' apply false
}

allprojects {
    buildDir = new File(rootProject.rootDir, "../../out/android/${project.name}")
}
```

### 关键配置

**AGP 版本 8.7.3：** Android Gradle Plugin 的版本决定了支持的 Android SDK 版本范围、Gradle 特性和 CMake 集成行为。8.7.x 是 2024 年的稳定版本。

**Kotlin 1.9.22：** 虽然 KrKr2 的核心是 C++，但 Android 层使用 Kotlin/Java 处理 Activity 生命周期、UI、权限等。

**`apply false`：** 只声明插件版本，不在根项目中应用。每个子项目在自己的 `build.gradle` 中决定是否应用。

**构建输出重定向：**

```groovy
buildDir = new File(rootProject.rootDir, "../../out/android/${project.name}")
```

将构建产物从默认的 `platforms/android/app/build/` 重定向到项目根目录的 `out/android/app/`。这与桌面平台的输出目录 `out/windows/`、`out/linux/` 保持一致的组织结构：

```
krkr2/
└── out/
    ├── windows/debug/       ← 桌面构建
    ├── linux/debug/
    ├── macos/debug/
    └── android/             ← Android 构建
        ├── app/             ← 主应用产物
        ├── sdl/             ← SDL2 产物
        └── cocos2dx/        ← Cocos2d-x 产物
```

---

## app/build.gradle：核心构建配置（140 行）

这是 KrKr2 Android 构建最重要的文件。我们逐块分析。

### 插件声明与 Android 基本配置

```groovy
plugins {
    id 'com.android.application'
}

android {
    namespace 'org.psnxs.krkr2'
    compileSdk 34
    ndkVersion '28.0.13004108'
```

| 配置项 | 值 | 说明 |
|--------|---|------|
| `namespace` | `org.psnxs.krkr2` | Android 应用包名（Java 包命名空间） |
| `compileSdk` | 34 | 编译时使用的 Android SDK 版本（Android 14） |
| `ndkVersion` | `28.0.13004108` | **锁定 NDK 版本**，确保所有开发者和 CI 使用相同版本 |

> **为什么锁定 NDK 版本？** 不同 NDK 版本的 Clang 编译器版本、系统头文件和预编译库可能不同。不锁定版本会导致"在我机器上能编译，你的不行"的问题。

### defaultConfig 块

```groovy
defaultConfig {
    applicationId 'org.psnxs.krkr2'
    minSdk 29
    targetSdk 34
    versionCode 1
    versionName "0.0.1"

    ndk {
        abiFilters 'arm64-v8a', 'x86_64'
    }

    externalNativeBuild {
        cmake {
            arguments "-DANDROID=ON",
                      "-DCMAKE_TOOLCHAIN_FILE=${rootProject.rootDir}/../../cmake/vcpkg_android.cmake",
                      "-DCMAKE_CXX_STANDARD=17"
            cppFlags "-std=c++17 -frtti -fexceptions"
            version '3.31.1'
        }
    }
}
```

逐字段详解：

#### 版本和 SDK 配置

| 字段 | 值 | 说明 |
|------|---|------|
| `applicationId` | `org.psnxs.krkr2` | Google Play 上的唯一标识符 |
| `minSdk` | 29 | 最低支持 Android 10（2019 年发布） |
| `targetSdk` | 34 | 目标 Android 14（决定行为兼容性模式） |
| `versionCode` | 1 | 内部版本号（整数，每次发布递增） |
| `versionName` | `"0.0.1"` | 用户可见的版本号 |

**`minSdk 29` 的含义：** KrKr2 不支持低于 Android 10 的设备。这允许使用 Android 10 引入的 API，如 `ScopedStorage`（分区存储）。KrKr2 需要读取游戏文件，分区存储对文件访问有重大影响。

#### ndk.abiFilters — ABI 过滤

```groovy
ndk {
    abiFilters 'arm64-v8a', 'x86_64'
}
```

`abiFilters` 控制为哪些 CPU 架构编译本地代码。KrKr2 只支持 64 位：

| ABI | CPU 架构 | 设备类型 | KrKr2 支持 |
|-----|---------|---------|-----------|
| `arm64-v8a` | ARM 64-bit | 绝大多数手机/平板 | ✅ |
| `x86_64` | x86 64-bit | 模拟器、ChromeOS、部分平板 | ✅ |
| `armeabi-v7a` | ARM 32-bit | 老旧设备 | ❌ 不支持 |
| `x86` | x86 32-bit | 老旧模拟器 | ❌ 不支持 |

**为什么不支持 32 位？**
1. Google Play 自 2019 年 8 月起要求新应用必须提供 64 位支持
2. 32 位设备市场份额持续下降（2024 年已低于 5%）
3. KrKr2 使用的某些第三方库（如 FFmpeg）在 32 位下需要额外适配
4. 减少 APK 体积和维护成本

#### externalNativeBuild.cmake — CMake 集成

```groovy
externalNativeBuild {
    cmake {
        arguments "-DANDROID=ON",
                  "-DCMAKE_TOOLCHAIN_FILE=${rootProject.rootDir}/../../cmake/vcpkg_android.cmake",
                  "-DCMAKE_CXX_STANDARD=17"
        cppFlags "-std=c++17 -frtti -fexceptions"
        version '3.31.1'
    }
}
```

**`arguments`** — 传递给 `cmake` 命令的参数：

| 参数 | 说明 |
|------|------|
| `-DANDROID=ON` | 设置 KrKr2 自定义变量，让 CMakeLists.txt 中的 `if(ANDROID)` 为真 |
| `-DCMAKE_TOOLCHAIN_FILE=.../vcpkg_android.cmake` | 使用 vcpkg Android 工具链（内部链式加载 NDK 工具链） |
| `-DCMAKE_CXX_STANDARD=17` | 强制 C++17 标准 |

**路径分析：** `${rootProject.rootDir}` 是 `platforms/android/`，`../../cmake/` 回退到项目根目录的 `cmake/` 目录。完整路径：`krkr2/cmake/vcpkg_android.cmake`。

**`cppFlags`** — 直接传给 C++ 编译器的选项：

| 选项 | 说明 |
|------|------|
| `-std=c++17` | 与 `CMAKE_CXX_STANDARD=17` 相同（双重保险） |
| `-frtti` | 启用 RTTI（运行时类型信息）— Android NDK 默认**关闭** RTTI |
| `-fexceptions` | 启用 C++ 异常 — Android NDK 默认**关闭**异常 |

> **为什么 Android NDK 默认关闭 RTTI 和异常？** 这是为了减小二进制体积和提升性能。但 KrKr2 的核心代码（TJS2 脚本引擎、Cocos2d-x 框架）大量使用 `dynamic_cast`（需要 RTTI）和 `try/catch`（需要异常），所以必须显式启用。

**`version '3.31.1'`** — 要求使用 CMake 3.31.1。Gradle 会从 Android SDK 管理器中查找此版本。如果未安装，构建会报错并提示安装。

### externalNativeBuild 路径声明

```groovy
externalNativeBuild {
    cmake {
        path "${rootProject.rootDir}/../../CMakeLists.txt"
    }
}
```

注意这里有两个不同位置的 `externalNativeBuild` 块：

1. **`defaultConfig.externalNativeBuild.cmake`** — 配置 cmake 的**参数**（arguments, cppFlags, version）
2. **`android.externalNativeBuild.cmake`** — 声明 cmake 的**入口文件路径**

`path` 指向项目根目录的 `CMakeLists.txt`（即 `krkr2/CMakeLists.txt`，143 行）。Gradle 会为每个 ABI 分别调用 cmake，指定不同的 `ANDROID_ABI` 值。

**Gradle 实际执行的 cmake 命令（以 arm64-v8a 为例）：**

```bash
cmake \
  -H/path/to/krkr2 \
  -B/path/to/krkr2/out/android/app/.cxx/Debug/arm64-v8a \
  -G Ninja \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-29 \
  -DANDROID_NDK=/path/to/ndk/28.0.13004108 \
  -DANDROID=ON \
  -DCMAKE_TOOLCHAIN_FILE=/path/to/krkr2/cmake/vcpkg_android.cmake \
  -DCMAKE_CXX_STANDARD=17 \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-std=c++17 -frtti -fexceptions"
```

### 签名配置

```groovy
signingConfigs {
    release {
        def keyFile = file("keystore.jks")
        if (keyFile.exists()) {
            storeFile keyFile
            storePassword System.getenv("KEYSTORE_PASSWORD") ?: "android"
            keyAlias "krkr2"
            keyPassword System.getenv("KEY_PASSWORD") ?: "android"
        }
    }
}

buildTypes {
    release {
        minifyEnabled false
        signingConfig signingConfigs.release
    }
    debug {
        debuggable true
        jniDebuggable true
    }
}
```

#### 签名机制详解

Android 要求所有 APK 必须签名才能安装。签名方式因构建类型而异：

| 构建类型 | 签名方式 | 密钥来源 |
|---------|---------|---------|
| Debug | 自动使用 Android SDK 的 debug.keystore | `~/.android/debug.keystore` |
| Release | 使用项目的 `keystore.jks` | 环境变量或默认密码 |

**安全设计：**
- 密码优先从环境变量读取（`System.getenv("KEYSTORE_PASSWORD")`）
- 如果环境变量不存在，回退到默认密码 `"android"`（仅用于开发）
- `keystore.jks` 文件**不应该**提交到 Git（应在 `.gitignore` 中排除）
- CI 环境中通过 secrets 注入环境变量

> **常见错误：** 如果 `keystore.jks` 文件不存在，`keyFile.exists()` 返回 false，签名配置为空。此时 Release 构建会失败（因为引用了空的 `signingConfigs.release`）。解决方案：在 CI 中先生成密钥库。

#### Debug 构建的特殊选项

```groovy
debug {
    debuggable true      // 允许 Android Studio 附加调试器
    jniDebuggable true   // 允许调试 JNI 本地代码（gdb/lldb）
}
```

`jniDebuggable true` 是调试 C++ 代码的关键。它告诉 Gradle 在 APK 中包含调试符号，并允许 Android Studio 的 Native Debugger 附加到 native 线程。

### APK 分包策略

```groovy
splits {
    abi {
        enable true
        reset()
        include 'arm64-v8a', 'x86_64'
        universalApk false
    }
}
```

`splits.abi` 控制是生成**一个大 APK**（包含所有 ABI 的 .so）还是**每个 ABI 独立的 APK**：

| 配置 | `universalApk false` + `enable true` | `universalApk true` | `enable false` |
|------|--------------------------------------|---------------------|----------------|
| 产物数量 | 2 个 APK（每 ABI 一个） | 3 个 APK（2 个分 ABI + 1 个全量） | 1 个 APK（包含所有 .so） |
| APK 体积 | 最小（每个约 50%） | 全量包最大 | 最大 |
| 分发方式 | Google Play 自动分发 | 全量包备用 | 简单但浪费流量 |

KrKr2 选择 `universalApk false`，意味着：
- 构建产生 2 个 APK：`app-arm64-v8a-debug.apk` 和 `app-x86_64-debug.apk`
- 每个 APK 只包含对应架构的 `libkrkr2.so`
- 用户安装时下载量减少约 50%

### Prefab 和 Java 依赖

```groovy
buildFeatures {
    prefab true
}

dependencies {
    implementation project(':sdl')
    implementation project(':cocos2dx')
    implementation 'com.google.oboe:oboe:1.9.0'
    implementation 'com.google.android.material:material:1.11.0'
    implementation 'androidx.appcompat:appcompat:1.7.0'
}
```

#### Prefab

`prefab true` 启用 Android 的 Prefab 包管理器，它允许 AAR（Android Archive）中的本地库被 CMake `find_package()` 发现。

工作流程：
1. Gradle 下载 AAR 包（如 Oboe 音频库）
2. Prefab 提取 AAR 中的 `.so` 文件和头文件
3. 生成 CMake 配置文件（`xxx-config.cmake`）
4. CMake 中可以用 `find_package(oboe CONFIG REQUIRED)` 找到这些库

#### 依赖解析

| 依赖 | 类型 | 用途 |
|------|------|------|
| `project(':sdl')` | 本地子项目 | SDL2 — 窗口/输入/音频底层抽象 |
| `project(':cocos2dx')` | 本地子项目 | Cocos2d-x — 渲染框架 Java 层 |
| `oboe:1.9.0` | Maven (Google) | 低延迟音频（Android AAudio/OpenSL ES 封装） |
| `material:1.11.0` | Maven (Google) | Material Design UI 组件 |
| `appcompat:1.7.0` | Maven (AndroidX) | 向后兼容的 Activity/Fragment |

---

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

## 对照项目源码

| 文件路径 | 行号 | 关键内容 |
|---------|------|---------|
| `platforms/android/settings.gradle` | 1-24 | 三个子项目声明（:sdl, :cocos2dx, :app） |
| `platforms/android/build.gradle` | 1-21 | AGP 8.7.3、Kotlin 1.9.22、输出目录重定向 |
| `platforms/android/app/build.gradle` | 1-140 | 完整应用配置：NDK、CMake集成、签名、分包 |
| `platforms/android/gradle.properties` | 1-31 | JVM 6G 堆、compileSdk/minSdk/targetSdk |
| `platforms/android/cpp/krkr2_android.cpp` | 1-65 | JNI 桥接：spdlog android sink、breakpad、SDL2 动态加载 |
| `cmake/vcpkg_android.cmake` | 1-99 | ABI→triplet 映射、双工具链叠加 |
| `CMakeLists.txt` | 50-65 | `if(ANDROID) add_library(SHARED)` — Android 构建共享库 |

---

## 本节小结

1. **Android 构建是双层架构**：Gradle（外层）负责 Java 编译、资源打包、APK 签名；CMake（内层）负责 C++ 编译
2. **settings.gradle** 声明三个子项目（:sdl, :cocos2dx, :app），使用相对路径指向 `external/` 目录
3. **app/build.gradle** 通过 `externalNativeBuild.cmake` 块将 CMake 参数（工具链、标准、ABI）传递给本地构建
4. **多 ABI 构建**：Gradle 为每个 `abiFilters` 中的 ABI 分别调用一次 CMake，生成独立的 `libkrkr2.so`
5. **JNI 桥接**：`krkr2_android.cpp` 的 `JNI_OnLoad()` 是 native 代码的入口点，负责初始化日志、崩溃报告和 SDL2
6. **签名配置** 通过环境变量注入密码，避免硬编码敏感信息
7. **APK 分包**（`splits.abi`）为每个架构生成独立 APK，减少用户下载量
8. **Prefab** 允许 AAR 中的本地库被 CMake `find_package()` 发现

---

## 练习题与答案

### 题目 1：externalNativeBuild 的两个位置

`app/build.gradle` 中有两个 `externalNativeBuild` 块——一个在 `defaultConfig` 内，一个在 `android` 直接子级。它们各自的作用是什么？如果把 `path` 放到 `defaultConfig` 内的块中会怎样？

<details>
<summary>查看答案</summary>

**两个块的区别：**

1. **`android.externalNativeBuild.cmake`**（`android` 直接子级）：
   - 声明 CMake 的**入口文件路径**（`path`）
   - 这是一次性的全局声明，告诉 Gradle "我的 CMakeLists.txt 在哪里"

2. **`android.defaultConfig.externalNativeBuild.cmake`**（`defaultConfig` 内）：
   - 配置 CMake 的**构建参数**（`arguments`、`cppFlags`、`version`）
   - 可以被 `buildTypes` 或 `productFlavors` 覆盖（比如 Debug 和 Release 可以有不同的参数）

**如果把 `path` 放到 `defaultConfig` 内：**

Gradle 会报错：`path` 是 `android.externalNativeBuild.cmake` 的属性，不是 `defaultConfig.externalNativeBuild.cmake` 的属性。两个块虽然都叫 `externalNativeBuild`，但它们是 Gradle DSL 中不同层级的不同对象。

</details>

### 题目 2：多 ABI 构建的 CMake 调用次数

如果 `abiFilters` 设置为 `'arm64-v8a', 'x86_64'`，构建 Debug 版本时 Gradle 会调用几次 CMake 配置（`cmake -B ...`）和几次 CMake 构建（`cmake --build ...`）？

<details>
<summary>查看答案</summary>

**CMake 配置（configure）：2 次**
- 第 1 次：`-DANDROID_ABI=arm64-v8a`，输出到 `.cxx/Debug/arm64-v8a/`
- 第 2 次：`-DANDROID_ABI=x86_64`，输出到 `.cxx/Debug/x86_64/`

**CMake 构建（build）：2 次**
- 第 1 次：构建 `arm64-v8a` 目录下的 targets → `libkrkr2.so`（ARM 64-bit）
- 第 2 次：构建 `x86_64` 目录下的 targets → `libkrkr2.so`（x86 64-bit）

每个 ABI 是完全独立的构建——有自己的 CMake 缓存、自己的 Ninja 构建目录、自己的 `.so` 输出。它们之间没有共享任何编译产物。

**增量构建时：** 如果只修改了 C++ 源码，两次 configure 可能会跳过（使用缓存），但两次 build 都会执行（检查文件变更后只重编译修改的文件）。

如果添加 `armeabi-v7a`，则变为 3 次配置 + 3 次构建。

</details>

### 题目 3：JNI_OnLoad 冲突

KrKr2 的 `krkr2_android.cpp` 中使用 `dlopen("libSDL2.so")` 动态加载 SDL2 并手动调用其 `JNI_OnLoad`。如果改为直接链接 SDL2（不用 dlopen），会出现什么问题？如何验证？

<details>
<summary>查看答案</summary>

**问题：`JNI_OnLoad` 符号冲突。**

一个共享库（`.so`）只能导出一个 `JNI_OnLoad` 函数。如果 `libkrkr2.so` 直接链接了 SDL2 的代码（静态链接），那么会有两个 `JNI_OnLoad` 定义：
- `krkr2_android.cpp` 中的 `JNI_OnLoad`
- SDL2 源码中的 `JNI_OnLoad`

链接器行为取决于链接顺序：
1. **GNU ld / LLD：** 报 `multiple definition of 'JNI_OnLoad'` 错误
2. **如果一个是 WHOLE_ARCHIVE：** 可能链接成功但使用了错误的版本

**如果 SDL2 是动态库（`libSDL2.so` 单独的 .so）：**
- `System.loadLibrary("SDL2")` 加载时会自动调用 SDL2 的 `JNI_OnLoad`
- `System.loadLibrary("krkr2")` 加载时调用 krkr2 的 `JNI_OnLoad`
- 不会冲突！

KrKr2 使用 `dlopen` 的原因可能是 SDL2 以静态方式嵌入 `libkrkr2.so`，同时需要保留 SDL2 的 JNI 初始化逻辑。`dlopen` 自身的 `.so` 然后 `dlsym` 查找 SDL2 的 `JNI_OnLoad` 是一种规避方式。

**验证方法：**
```bash
# 检查 libkrkr2.so 中有几个 JNI_OnLoad 符号
nm -D libkrkr2.so | grep JNI_OnLoad
# 应该只有一个（krkr2 的）

# 检查 SDL2 是否作为静态库链接进来
readelf -d libkrkr2.so | grep NEEDED
# 如果没有 libSDL2.so，说明 SDL2 是静态链接的
```

</details>

---

## 下一步

下一节 [05-自定义CMake模块](./05-自定义CMake模块.md) 将深入分析 `cmake/` 目录下的两个自定义模块：`CocosBuildHelpers.cmake`（381 行的 Cocos2d-x 构建辅助宏集合）和 `vcpkg_android.cmake`（99 行的 Android vcpkg 集成），了解项目如何通过自定义 CMake 函数和宏扩展构建系统的能力。
