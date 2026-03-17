# Android 环境搭建

本章节将引导你为 KR2 项目配置 Android 交叉编译环境。你可以使用 Windows、Linux 或 macOS 任何一个系统作为宿主机来编译 Android 版本的程序。

## 1. 前提条件

- 宿主系统：Windows 10/11、Linux (Ubuntu 22.04+) 或 macOS 14+。
- 硬盘空间：至少预留 30GB 空间。Android SDK 和 NDK 比较占空间。

## 2. 安装 Android SDK 和 NDK

KR2 对 NDK 的版本有严格要求，请务必安装指定版本。

1.  **Android Studio**：
    - 下载并安装 [Android Studio](https://developer.android.google.cn/studio)。
    - 在 Android Studio 中打开 **Settings -> Languages & Frameworks -> Android SDK**。
    - 在 **SDK Platforms** 选项卡下，勾选 Android 14 (API level 34) 或更高。
    - 在 **SDK Tools** 选项卡下，勾选：
        - **NDK (Side by side)**：点击右下角 "Show Package Details"，勾选版本 **28.0.13004108**。
        - **Android SDK Command-line Tools (latest)**
        - **CMake** (3.31.1+，建议通过独立安装程序安装最新版)

2.  **环境变量配置**：
    你需要配置以下三个关键环境变量：
    - `ANDROID_SDK`：Android SDK 的安装路径（例如 `C:\Users\用户名\AppData\Local\Android\Sdk`）。
    - `ANDROID_NDK`：NDK 的根路径（例如 `.../Sdk/ndk/28.0.13004108`）。
    - `JAVA_HOME`：JDK 17 的安装路径。

## 3. 安装 JDK 17

KR2 的 Android 构建脚本要求使用 JDK 17。

- 推荐安装 [Eclipse Temurin JDK 17](https://adoptium.net/temurin/releases/?version=17)。
- 安装完成后，请确认 `java -version` 输出版本为 17。

## 4. 安装基础构建工具

Android 交叉编译仍然需要宿主环境的构建工具支持。请根据你的宿主平台，参考前三章的方法安装以下工具：

- **CMake 3.31.1+**
- **Ninja**
- **vcpkg** (务必设置 `VCPKG_ROOT`)
- **Bison 3.8.2+** (Windows 下安装 winflexbison)
- **Python 3.x**
- **NASM**

## 5. 编译 Android 项目

KR2 的 Android 编译使用 Gradle 构建系统。

- **使用命令行编译**：
    在项目根目录下运行：
    ```bash
    ./platforms/android/gradlew -p ./platforms/android assembleDebug
    ```
    这条命令会调用 Gradle Wrapper 自动下载对应的 Gradle 版本，并启动编译任务。

- **使用 Android Studio 编译**：
    在 Android Studio 中打开项目文件夹下的 `platforms/android` 目录。IDE 将自动同步 Gradle 配置。同步完成后，点击工具栏上的 "Run" (绿色三角形) 按钮，即可将项目安装到连接的 Android 设备或模拟器上。

## 6. FAQ：常见问题与修复

### 6.1 glib 安装失败（Windows 交叉编译 Android）

在 Windows 上交叉编译 Android 版本时，由于 Meson 编译系统的一个路径处理漏洞，可能会导致 glib 库安装失败。具体表现为路径中出现了错误的 `"/./"` 分隔符。

**修复步骤**：
1.  找到 vcpkg 下载的工具目录，路径类似于：`VCPKG_ROOT/downloads/tools/meson-1.6.1-哈希值/mesonbuild/minstall.py`。
2.  打开该文件，搜索 `install_data` 函数。
3.  在该函数中手动修改代码，确保生成的安装路径经过正则替换或简单的字符串替换，消除 `"/./"` 带来的路径解析错误。

### 6.2 Docker 替代方案

如果你不想在本地宿主环境中安装繁琐的 Android 工具链，可以使用 KR2 提供的 Docker 构建环境。

在项目根目录下运行：
```bash
docker build -f dockers/android.Dockerfile -t android-builder .
```
该镜像已预先配置好所有 Android 构建所需的 NDK、SDK 和工具链环境。

## 7. 验证工具安装

在终端执行以下命令，确保 Android 开发环境配置正确：

```bash
# 检查 JAVA_HOME
echo $JAVA_HOME

# 检查 SDK 命令 (Windows 下使用 sdkmanager.bat)
sdkmanager --version

# 检查 NDK 目录 (Windows 下使用 ls 或 dir)
ls $ANDROID_NDK

# 检查 Gradle (使用项目自带的 Wrapper)
./platforms/android/gradlew -v
```

## 8. 练习题与答案

1.  在 Windows 系统上为 Android 交叉编译 KR2 项目时，为什么要关注 `VCPKG_ROOT` 目录下的 `mesonbuild/minstall.py` 文件？
2.  KR2 项目推荐使用的 Android NDK 版本是多少？安装该版本时有什么特殊的注意事项？
3.  为什么 KR2 项目不需要单独在宿主系统上安装 Gradle？

<details>
<summary>查看答案</summary>

1.  **回答**：因为在 Windows 宿主机上交叉编译 Android 时，Meson 编译系统（常被 glib 等库使用）的一个旧版本实现中存在路径合并漏洞，导致安装路径中出现 `"/./"` 字符而引发错误。开发者需要手动修改 `minstall.py` 文件中的 `install_data` 函数来修复该路径问题。
2.  **回答**：推荐版本是 **28.0.13004108**。安装时，必须在 Android Studio 的 SDK Tools 选项卡中勾选 "Show Package Details"，才能从列表中精确选择这一特定版本，而不是只安装最新版。
3.  **回答**：因为项目自带了 **Gradle Wrapper**（位于 `platforms/android` 目录下的 `gradlew` 和 `gradlew.bat`）。它会自动下载并使用项目配置文件中指定的 Gradle 版本，确保所有开发者的构建环境完全一致。

</details>


