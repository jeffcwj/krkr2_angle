# 02-自定义 Triplet (架构-系统-链接方式)

在 vcpkg 的世界里，**Triplet** 是一个极其核心的概念。如果你掌握了 Triplet，你就掌握了 vcpkg 在不同平台、不同编译环境下自由切换的“黑魔法”。

## 什么是 Triplet

Triplet（三元组）本质上是一个简单的文本文件名，它定义了库应该如何被编译和链接。一个标准的 Triplet 名称通常由以下三部分组成：

`[目标架构]-[目标操作系统]-[链接方式]`

例如：
-   `x64-windows`：64位 Windows，动态链接。
-   `x64-windows-static`：64位 Windows，静态链接。
-   `arm64-android`：64位 Android。
-   `x64-linux`：64位 Linux。
-   `x64-osx`：64位 macOS。

## vcpkg 内置 Triplet 列表

你可以通过运行以下命令查看 vcpkg 内置的所有 Triplet：

```bash
vcpkg help triplets
```

常见的内置 Triplet 包括：
-   **Windows**: `x86-windows`, `x64-windows`, `x64-windows-static`
-   **Linux**: `x64-linux`, `arm64-linux`
-   **macOS**: `x64-osx`, `arm64-osx`
-   **Android**: `arm64-android`, `x64-android`

## Triplet 文件中可以设置的变量

Triplet 文件实际上是一个 CMake 脚本。你可以在其中设置一系列 `VCPKG_` 开头的变量来控制编译行为：

| 变量名 | 说明 |
| :--- | :--- |
| `VCPKG_TARGET_ARCHITECTURE` | 目标架构（x64, x86, arm64, arm） |
| `VCPKG_CRT_LINKAGE` | C 运行时链接方式（static, dynamic） |
| `VCPKG_LIBRARY_LINKAGE` | 库的链接方式（static, dynamic） |
| `VCPKG_CMAKE_SYSTEM_NAME` | 目标系统名（Windows, Linux, Darwin, Android, iOS） |
| `VCPKG_CMAKE_SYSTEM_VERSION` | 目标系统版本（例如 Android 的 API Level） |
| `VCPKG_BUILD_TYPE` | 编译类型（release, debug, 为空则两者都编） |

## 为什么需要自定义 Triplet

内置的 Triplet 虽然覆盖了大部分通用场景，但在 KrKr2 这样复杂的跨平台项目中，我们需要更精细的控制：

1.  **Android 平台的特殊需求**：不同的 Android API Level 对应的 NDK 行为不同，我们需要通过 Triplet 锁定 API 版本。
2.  **强制静态/动态链接**：某些第三方库（如 SDL2）在 Android 下必须是动态链接的，以便被 Java 层加载。
3.  **自定义编译器标志**：我们需要开启特定的优化选项（如 LTO 或 SSE 指令集）。
4.  **排除特定架构**：为了减小安装后的体积，我们可以在 Triplet 中只保留 Release 版本。

## KrKr2 的自定义 Triplet 示例

在 KrKr2 项目中，我们维护了一套自定义的 Triplet 方案。

### 1. arm64-android.cmake (标准静态 Android)

```cmake
set(VCPKG_TARGET_ARCHITECTURE arm64)
set(VCPKG_CRT_LINKAGE static)
set(VCPKG_LIBRARY_LINKAGE static)
set(VCPKG_CMAKE_SYSTEM_NAME Android)
set(VCPKG_CMAKE_SYSTEM_VERSION 28) # 锁定 Android SDK 28 (Pie)
```

### 2. android-dynamic-libs.cmake (混合链接方式)

有时我们希望大部分库静态链接，但特定的几个库动态链接：

```cmake
set(VCPKG_TARGET_ARCHITECTURE arm64)
set(VCPKG_CRT_LINKAGE static)
set(VCPKG_LIBRARY_LINKAGE static)
set(VCPKG_CMAKE_SYSTEM_NAME Android)

# 为 SDL2 强制启用动态库模式
if(PORT STREQUAL "sdl2")
    set(VCPKG_LIBRARY_LINKAGE dynamic)
endif()
```

## 如何启用自定义 Triplet (Overlay Triplets)

你需要在项目根目录的 `vcpkg-configuration.json` 文件中配置 `overlay-triplets` 字段。

```json
{
  "overlay-triplets": [
    "./vcpkg-triplets"
  ]
}
```

配置完成后，你可以直接在命令行中使用这些 Triplet：

```bash
vcpkg install zlib:arm64-android --overlay-triplets=./vcpkg-triplets
```

如果你是在 Manifest 模式下配合 CMake 使用，可以在 `CMakePresets.json` 中配置：

```json
{
  "name": "android-arm64",
  "cacheVariables": {
    "VCPKG_TARGET_TRIPLET": "arm64-android",
    "VCPKG_OVERLAY_TRIPLETS": "${sourceDir}/vcpkg-triplets"
  }
}
```

## 编写自定义 Triplet 的完整步骤

1.  **确定名称**：例如，我们要为一个嵌入式设备创建一个 `x64-linux-embedded`。
2.  **创建目录**：在项目根目录下创建一个名为 `vcpkg-triplets` 的目录。
3.  **新建文件**：在目录中新建 `x64-linux-embedded.cmake`。
4.  **填写内容**：
    ```cmake
    set(VCPKG_TARGET_ARCHITECTURE x64)
    set(VCPKG_CRT_LINKAGE dynamic)
    set(VCPKG_LIBRARY_LINKAGE static)
    set(VCPKG_CMAKE_SYSTEM_NAME Linux)
    # 你还可以添加自定义编译器标志
    set(VCPKG_CXX_FLAGS "-O3 -march=native")
    ```
5.  **配置并使用**：在 `vcpkg-configuration.json` 中添加该路径，然后运行 `vcpkg install`。

## 跨平台说明

在不同平台上，Triplet 的变量值有所不同：

-   **Windows**: `VCPKG_CRT_LINKAGE` 非常重要。`dynamic` 对应 `/MD`，`static` 对应 `/MT`。如果两者不匹配，编译时会报 `LNK2038` 冲突错误。
-   **Linux**: `VCPKG_CMAKE_SYSTEM_NAME` 必须设为 `Linux`（注意首字母大写）。
-   **macOS**: 如果你希望编译 Universal Binary (包含 x64 和 arm64)，你需要在 Triplet 中设置 `VCPKG_OSX_ARCHITECTURES` 为 `x86_64;arm64`。
-   **Android**: 必须在环境变量或 Triplet 中指定 `ANDROID_NDK_HOME` 的路径，否则 vcpkg 找不到交叉编译器。

## 练习题与答案

### 题目 1：VCPKG_LIBRARY_LINKAGE 的作用

如果将 `VCPKG_LIBRARY_LINKAGE` 设置为 `static`，vcpkg 会如何处理那些仅支持动态链接的库？

<details>
<summary>查看答案</summary>

vcpkg 会优先尝试按照 `VCPKG_LIBRARY_LINKAGE` 指定的方式（静态链接）来编译库。如果库的构建脚本（Portfile）被正确编写，它会遵从这个全局设置。但对于那些从技术上完全不支持静态链接的库，它们可能会忽略这个变量，或者导致编译失败。

</details>


### 题目 2：Triplet 文件名的限制

你可以将一个自定义的 Triplet 文件命名为 `my-super-triplet.cmake`。此时，你应该如何通过命令行请求安装库？

<details>
<summary>查看答案</summary>

你需要使用文件名（不含扩展名）作为 Triplet 名称。例如，执行 `vcpkg install zlib:my-super-triplet`。同时不要忘了通过 `--overlay-triplets` 指定目录，或者在 `vcpkg-configuration.json` 中预先配置。

</details>

### 题目 3：混合架构与混合链接

在 Android 开发中，Java 调用的 JNI 库通常必须是动态库（`.so`），而 JNI 库依赖的底层 C++ 库我们通常希望静态链接进 `.so`。请问在 Triplet 中应该如何配置？

<details>
<summary>查看答案</summary>

你可以设置全局 `VCPKG_LIBRARY_LINKAGE static`。然后，在你的 JNI 库的 `portfile.cmake` 中手动覆盖或保持默认。或者，在 Triplet 中使用判断语句：
```cmake
set(VCPKG_LIBRARY_LINKAGE static)
if(PORT STREQUAL "my-jni-lib")
    set(VCPKG_LIBRARY_LINKAGE dynamic)
endif()
```
这样除了 `my-jni-lib` 之外的所有库都会被静态编译，最后链接进 `my-jni-lib.so` 中。

</details>
