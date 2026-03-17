# 本节目标
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

