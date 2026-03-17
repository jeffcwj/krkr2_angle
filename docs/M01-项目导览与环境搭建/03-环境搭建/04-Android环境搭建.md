# Android 环境搭建

> **所属模块：** M01-项目导览与环境搭建
>
> **前置知识：** [01-Windows环境搭建](./01-Windows环境搭建.md)、[02-Linux环境搭建](./02-Linux环境搭建.md)、[03-macOS环境搭建](./03-macOS环境搭建.md)
>
> **预计阅读时间：** 60 分钟

本节将完整搭建 KR2 的 Android 构建环境。

本节所有版本号都来自项目真实文件。

## 本节目标

读完本节后，你将能够：

1. 安装 Android SDK（Android Studio 与命令行两种方式）。
2. 安装并锁定项目要求的 NDK 版本。
3. 配置 JDK 17 与 Android/vcpkg 环境变量。
4. 理解 Gradle 的关键 Android 配置。
5. 理解 CMake Android 交叉编译链路。
6. 理解 vcpkg Android triplet 配置。
7. 使用 ADB 完成模拟器与真机调试。

## 1. 先核对项目真实配置

请先阅读这些文件，再执行安装。

- `krkr2/platforms/android/build.gradle`
- `krkr2/platforms/android/gradle.properties`
- `krkr2/platforms/android/app/build.gradle`
- `krkr2/cmake/vcpkg_android.cmake`
- `krkr2/CMakePresets.json`
- `krkr2/vcpkg/triplets/arm64-android.cmake`
- `krkr2/vcpkg/triplets/x64-android.cmake`
- `krkr2/vcpkg/triplets/arm-android.cmake`
- `krkr2/vcpkg/triplets/x86-android.cmake`
- `krkr2/vcpkg/triplets/android-dynamic-libs.cmake`

关键结论如下：

- Android Gradle 插件：`8.7.3`
- Kotlin Android 插件：`1.9.22`
- `compileSdk=34`
- `minSdkVersion=29`
- `targetSdkVersion=34`
- `ndkVersion='28.0.13004108'`
- ABI：`arm64-v8a`、`x86_64`
- CMake 版本：`3.31.1`
- CMake 路径：`../../../CMakeLists.txt`

另外，`CMakePresets.json` 当前没有 Android preset。

这不是错误。

本项目 Android 入口是 Gradle externalNativeBuild。

## 2. 安装 Android SDK（Android Studio 方式）

该方式适合首次搭环境。

1. 安装 Android Studio。
2. 打开 `Settings/Preferences -> Android SDK`。
3. 在 `SDK Platforms` 勾选 Android 14（API 34）。
4. 在 `SDK Tools` 勾选 Command-line Tools、Platform-Tools、NDK、CMake。
5. 打开 `Show Package Details` 并选择 NDK `28.0.13004108`。

默认 SDK 路径参考：

- Windows：`C:\Users\<用户名>\AppData\Local\Android\Sdk`
- Linux：`/home/<用户名>/Android/Sdk`
- macOS：`/Users/<用户名>/Library/Android/sdk`

## 3. 安装 Android SDK（命令行方式）

该方式适合 CI 与自动化。

### 3.1 配置 PATH

Windows：

```powershell
$env:ANDROID_SDK="C:\Users\$env:USERNAME\AppData\Local\Android\Sdk"
$env:Path += ";$env:ANDROID_SDK\cmdline-tools\latest\bin;$env:ANDROID_SDK\platform-tools"
sdkmanager.bat --version
```

Linux：

```bash
export ANDROID_SDK=$HOME/Android/Sdk
export PATH=$PATH:$ANDROID_SDK/cmdline-tools/latest/bin:$ANDROID_SDK/platform-tools
sdkmanager --version
```

macOS：

```bash
export ANDROID_SDK=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_SDK/cmdline-tools/latest/bin:$ANDROID_SDK/platform-tools
sdkmanager --version
```

### 3.2 安装组件

Windows：

```powershell
sdkmanager.bat --sdk_root=$env:ANDROID_SDK "platform-tools" "platforms;android-34" "build-tools;34.0.0" "ndk;28.0.13004108" "cmake;3.31.1"
yes | sdkmanager.bat --licenses
```

Linux：

```bash
sdkmanager --sdk_root=$ANDROID_SDK "platform-tools" "platforms;android-34" "build-tools;34.0.0" "ndk;28.0.13004108" "cmake;3.31.1"
yes | sdkmanager --licenses
```

macOS：

```bash
sdkmanager --sdk_root=$ANDROID_SDK "platform-tools" "platforms;android-34" "build-tools;34.0.0" "ndk;28.0.13004108" "cmake;3.31.1"
yes | sdkmanager --licenses
```

### 3.3 验证安装

Windows：

```powershell
Get-ChildItem "$env:ANDROID_SDK\ndk"
Get-ChildItem "$env:ANDROID_SDK\platforms"
```

Linux/macOS：

```bash
ls $ANDROID_SDK/ndk
ls $ANDROID_SDK/platforms
```

## 4. 安装 JDK 17 与环境变量

KR2 Android 构建要求 JDK 17。

### 4.1 Java 检查

```bash
java -version
```

### 4.2 JAVA_HOME 配置

Windows：

```powershell
$env:JAVA_HOME="C:\Program Files\Eclipse Adoptium\jdk-17"
$env:Path += ";$env:JAVA_HOME\bin"
java -version
```

Linux：

```bash
export JAVA_HOME=/usr/lib/jvm/temurin-17-jdk-amd64
export PATH=$PATH:$JAVA_HOME/bin
java -version
```

macOS：

```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 17)
export PATH=$PATH:$JAVA_HOME/bin
java -version
```

### 4.3 Android/vcpkg 环境变量

Windows：

```powershell
$env:ANDROID_SDK="C:\Users\$env:USERNAME\AppData\Local\Android\Sdk"
$env:ANDROID_NDK="$env:ANDROID_SDK\ndk\28.0.13004108"
$env:ANDROID_NDK_HOME=$env:ANDROID_NDK
$env:VCPKG_ROOT="D:\tools\vcpkg"
```

Linux：

```bash
export ANDROID_SDK=$HOME/Android/Sdk
export ANDROID_NDK=$ANDROID_SDK/ndk/28.0.13004108
export ANDROID_NDK_HOME=$ANDROID_NDK
export VCPKG_ROOT=$HOME/vcpkg
```

macOS：

```bash
export ANDROID_SDK=$HOME/Library/Android/sdk
export ANDROID_NDK=$ANDROID_SDK/ndk/28.0.13004108
export ANDROID_NDK_HOME=$ANDROID_NDK
export VCPKG_ROOT=$HOME/vcpkg
```

## 5. Gradle 配置详解

对照 `krkr2/platforms/android/app/build.gradle`：

- `compileSdk` 来自 `PROP_COMPILE_SDK_VERSION=34`
- `minSdkVersion=29`
- `targetSdkVersion=34`
- `ndkVersion='28.0.13004108'`
- `abiFilters='arm64-v8a','x86_64'`
- `splits.abi.include='arm64-v8a','x86_64'`

说明：

- `minSdkVersion` 决定最低可安装系统。
- `targetSdkVersion` 决定系统行为策略。
- ABI 过滤和分包必须一致。

## 6. CMake for Android 原理

对照 `krkr2/cmake/vcpkg_android.cmake`：

1. 检查 `ANDROID_NDK_HOME`。
2. 检查 `VCPKG_ROOT`。
3. 按 `ANDROID_ABI` 映射 `VCPKG_TARGET_TRIPLET`。
4. 组合 vcpkg 与 Android toolchain。

映射表：

| ANDROID_ABI | VCPKG_TARGET_TRIPLET |
|---|---|
| arm64-v8a | arm64-android |
| armeabi-v7a | arm-android |
| x86_64 | x64-android |
| x86 | x86-android |

关键组合：

- `CMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake`
- `VCPKG_CHAINLOAD_TOOLCHAIN_FILE=$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake`

## 7. vcpkg Android triplet 配置详解

目录：`krkr2/vcpkg/triplets/`

核心文件：

- `arm64-android.cmake`
- `arm-android.cmake`
- `x64-android.cmake`
- `x86-android.cmake`

共同参数：

- `VCPKG_CRT_LINKAGE static`
- `VCPKG_LIBRARY_LINKAGE static`
- `VCPKG_CMAKE_SYSTEM_NAME Android`
- `VCPKG_CMAKE_SYSTEM_VERSION 28`

特殊文件：`android-dynamic-libs.cmake`

- `sdl2` 在该文件中设为动态链接。

## 8. ADB 调试工具使用

### 8.1 设备连接

Windows：

```powershell
adb start-server
adb devices -l
```

Linux/macOS：

```bash
adb start-server
adb devices -l
```

### 8.2 安装与启动

Windows：

```powershell
adb install -r .\platforms\android\out\android\app\outputs\apk\debug\krkr2-1.5.0-all.apk
adb shell monkey -p org.github.krkr2 -c android.intent.category.LAUNCHER 1
```

Linux/macOS：

```bash
adb install -r ./platforms/android/out/android/app/outputs/apk/debug/krkr2-1.5.0-all.apk
adb shell monkey -p org.github.krkr2 -c android.intent.category.LAUNCHER 1
```

### 8.3 日志与卸载

Windows：

```powershell
adb logcat -d | Select-String "org.github.krkr2"
adb uninstall org.github.krkr2
```

Linux/macOS：

```bash
adb logcat -d | grep org.github.krkr2
adb uninstall org.github.krkr2
```

## 9. 模拟器 vs 真机调试

| 维度 | 模拟器（x86_64） | 真机（arm64-v8a） |
|---|---|---|
| 速度 | 快 | 中等 |
| ABI 一致性 | 低 | 高 |
| 真实性 | 中 | 高 |
| 适用场景 | 日常回归 | 提交前验证 |

推荐策略：先模拟器回归，再真机确认。

## 10. 常见问题排查

### 10.1 NDK 版本不匹配

现象：Gradle 报未安装或版本错误。

解决：安装 `28.0.13004108`，并检查 `ANDROID_NDK_HOME`。

### 10.2 SDK 路径错误

现象：`sdkmanager`、`adb` 不可用。

解决：检查 `ANDROID_SDK` 与 PATH。

### 10.3 Gradle Sync 失败

现象：Android Studio 同步失败。

解决：先运行 `gradlew tasks`，再检查 JDK 17 与网络。

### 10.4 externalNativeBuild 失败

现象：CMake 阶段报错。

解决：检查 ABI 列表、CMake 路径、`ANDROID_NDK_HOME`、`VCPKG_ROOT`。

### 10.5 Windows 下 glib/Meson 路径问题

现象：路径出现 `"/./"`。

解决：修正 `VCPKG_ROOT/downloads/tools/meson-*/mesonbuild/minstall.py` 的 `install_data` 路径逻辑。

## 11. 动手实践

目标：完成一次 Debug 构建并安装。

### 步骤 1：检查变量

Windows：

```powershell
echo $env:JAVA_HOME
echo $env:ANDROID_SDK
echo $env:ANDROID_NDK_HOME
echo $env:VCPKG_ROOT
```

Linux/macOS：

```bash
echo $JAVA_HOME
echo $ANDROID_SDK
echo $ANDROID_NDK_HOME
echo $VCPKG_ROOT
```

### 步骤 2：检查工具版本

Windows：

```powershell
java -version
sdkmanager.bat --version
adb version
cmake --version
ninja --version
```

Linux/macOS：

```bash
java -version
sdkmanager --version
adb version
cmake --version
ninja --version
```

### 步骤 3：构建 APK

Windows：

```powershell
.\platforms\android\gradlew.bat -p .\platforms\android clean assembleDebug
```

Linux/macOS：

```bash
./platforms/android/gradlew -p ./platforms/android clean assembleDebug
```

### 步骤 4：安装与启动

Windows：

```powershell
adb install -r .\platforms\android\out\android\app\outputs\apk\debug\krkr2-1.5.0-all.apk
adb shell monkey -p org.github.krkr2 -c android.intent.category.LAUNCHER 1
```

Linux/macOS：

```bash
adb install -r ./platforms/android/out/android/app/outputs/apk/debug/krkr2-1.5.0-all.apk
adb shell monkey -p org.github.krkr2 -c android.intent.category.LAUNCHER 1
```

### 步骤 5：抓日志

Windows：

```powershell
adb logcat -d | Select-String "org.github.krkr2"
```

Linux/macOS：

```bash
adb logcat -d | grep org.github.krkr2
```

## 12. 对照项目源码

请重点对照以下文件：

- `krkr2/platforms/android/build.gradle`
- `krkr2/platforms/android/settings.gradle`
- `krkr2/platforms/android/gradle.properties`
- `krkr2/platforms/android/app/build.gradle`
- `krkr2/cmake/vcpkg_android.cmake`
- `krkr2/vcpkg/triplets/arm64-android.cmake`
- `krkr2/vcpkg/triplets/x64-android.cmake`
- `krkr2/vcpkg/triplets/android-dynamic-libs.cmake`
- `krkr2/CMakePresets.json`

## 13. 本节小结

- Android 构建入口是 Gradle Wrapper。
- NDK 固定为 `28.0.13004108`。
- JDK 固定为 17。
- ABI 策略是 `arm64-v8a + x86_64`。
- vcpkg 与 Android toolchain 通过 chainload 方式组合。

## 14. 练习题与答案

### 题目 1：为什么必须固定 NDK 版本？

<details>
<summary>查看答案</summary>

因为项目已经在 `app/build.gradle` 固定 `ndkVersion '28.0.13004108'`。固定版本可以减少工具链差异，避免“本地能编译、CI 失败”的情况。

</details>

### 题目 2：`ANDROID_ABI=x86_64` 对应哪个 triplet？

<details>
<summary>查看答案</summary>

对应 `x64-android`，映射规则在 `cmake/vcpkg_android.cmake`。

</details>

### 题目 3：`abiFilters` 与 `splits.abi.include` 不一致会导致什么？

<details>
<summary>查看答案</summary>

可能导致 APK 缺少某 ABI 的 so，设备运行时会崩溃或无法启动。

</details>

### 题目 4：给出最小 ADB 排障命令。

<details>
<summary>查看答案</summary>

```bash
adb start-server
adb devices -l
adb install -r <apk-path>
adb shell monkey -p org.github.krkr2 -c android.intent.category.LAUNCHER 1
adb logcat -d
```

</details>

## 15. 下一步

下一节进入首次构建与运行：

- [01-编译项目](../04-第一次构建与运行/01-编译项目.md)
