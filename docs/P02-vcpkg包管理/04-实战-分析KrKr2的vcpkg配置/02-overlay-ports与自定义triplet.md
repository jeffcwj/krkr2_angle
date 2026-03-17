# 02-overlay-ports与自定义triplet

当 vcpkg 的官方仓库不能完全满足我们的项目需求时，KrKr2 使用了两个高级特性：**Overlay Ports**（覆盖端口）和 **Overlay Triplets**（覆盖三元组）。

## 1. vcpkg-configuration.json 解读

KrKr2 通过 `vcpkg-configuration.json` 来定义其独特的构建环境：

```json
{
  "default-registry": {
    "kind": "builtin", 
    "baseline": "b1e15efef6758eaa0beb0a8732cfa66f6a68a81d"
  },
  "overlay-ports": ["./vcpkg/ports"],
  "overlay-triplets": ["./vcpkg/triplets"]
}
```

- **baseline**: 这是一串 Git Commit Hash。它像时光机一样，将整个 vcpkg 仓库锁定在某一天的特定状态。这解决了“昨天还能编，今天就报错”的版本不一致问题。
- **overlay-ports**: 指定了一个本地目录。如果这个目录里有同名的库，vcpkg 会优先使用本地的版本，而不是官方仓库的版本。
- **overlay-triplets**: 同理，它允许我们定义自定义的编译目标配置（例如特殊的 Android 构建参数）。

## 2. 深入理解 Overlay Ports

在 KrKr2 项目中，我们一共自定义了 6 个端口：

1. **7zip**: 官方版本可能不完全适配 KrKr2 的解压逻辑，我们在此进行了代码补丁。
2. **cocos2dx**: 这是一个关键的自定义端口。官方 vcpkg 没有 Cocos2d-x，我们通过 overlay 将整个 Cocos2d-x 框架集成进来，并编写了适配 CMake 的 `portfile.cmake`。
3. **ffmpeg**: 官方编译选项过于臃肿。我们在 `portfile.cmake` 中禁用了不需要的解码器，以显著减小安装后的库体积。
4. **libarchive**: 针对 Android 平台修复了某些文件系统的兼容性补丁。
5. **libgdiplus**: 在 Linux 和 Android 上运行时，我们需要特殊的编译参数来连接本地图形驱动。
6. **unrar**: 官方版是闭源许可且需要特殊处理。通过自定义端口，我们能够更好地控制其编译流程。

每一个 `portfile.cmake` 都像是一段“构建菜谱”，它告诉 vcpkg：
- 从哪里下载源码。
- 如何应用 `.patch` 补丁文件。
- 传递哪些特定的 CMake 参数。
- 安装哪些头文件和静态库。

## 3. 自定义 Triplet 详解

Triplet（三元组）决定了“我们要把代码编译成什么样子”。KrKr2 为 Android 平台定制了 5 个核心 Triplet。

### 3.1 arm64-android.cmake 示例

这是目前最主流的 Android 64 位架构配置：

```cmake
set(VCPKG_TARGET_ARCHITECTURE arm64)
set(VCPKG_CRT_LINKAGE static)
set(VCPKG_LIBRARY_LINKAGE static)
set(VCPKG_CMAKE_SYSTEM_NAME Android)
set(VCPKG_SET_CHARSET_FLAG ON)
set(ANDROID_PLATFORM android-28)
set(VCPKG_BUILD_TYPE release)

set(VCPKG_ANDROID_DYNAMIC_LIBS
  "sdl2"
)
```

- **ARCHITECTURE arm64**: 针对现代 Android 手机的 64 位 ARM 架构。
- **LINKAGE static**: 为了方便发布，我们将大部分依赖库静态链接到最终的 `.so` 引擎中。
- **ANDROID_PLATFORM android-28**: 锁定 Android API 级别为 28（Android 9.0），保证兼容性。
- **VCPKG_ANDROID_DYNAMIC_LIBS**: 这是一个特殊的配置。由于 Android 系统加载机制的原因，某些库（如 SDL2）必须保持为动态库模式运行。

### 3.2 其它 Android Triplet
- **arm-android.cmake**: 针对老旧的 32 位 ARM 手机。
- **x64-android.cmake**: 用于在 Android 模拟器上高速运行。
- **x86-android.cmake**: 针对极少数基于 Intel 芯片的 Android 设备。

### 3.3 android-dynamic-libs.cmake
这是一个特殊的辅助三元组。它被设计用来统一管理那些在 Android 上**必须**使用动态链接的库。这种设计避免了在每个架构文件中重复编写复杂的 `if-else` 逻辑。

## 4. 总结：跨平台构建的最佳实践

通过分析 KrKr2 的配置，我们可以总结出以下几点：
- **Windows**: 使用官方 vcpkg 库即可满足大部分需求，偶尔需要 overlay 补丁。
- **Linux**: 依赖 `libgdiplus` 等库提供图形后端。
- **macOS**: 通常与 Linux 共享大部分 POSIX 兼容库。
- **Android**: 最为特殊。需要自定义 Triplet 锁定 API 版本，并根据架构（ARM/x86）动态调整编译参数。

## 练习题与答案

### 题目 1：在 `vcpkg-configuration.json` 中，`baseline` 的主要作用是什么？

<details>
<summary>查看答案</summary>

`baseline` 用于锁定 vcpkg 仓库的一个特定 Commit，从而固定所有依赖库的版本。这保证了即使在未来的某一天重新克隆项目，下载到的库版本仍然与今天一致，避免了库更新导致的编译错误。

</details>

### 题目 2：为什么我们在 Android 平台的 Triplet 中，通常将大部分库设为静态链接（`static`），却把 `sdl2` 设为动态链接？

<details>
<summary>查看答案</summary>

将库设为静态链接是为了将依赖直接打入主程序，简化 Android 项目的 `.so` 文件部署。但 `sdl2` 需要处理底层的窗口、输入和音频，它在 Android 上通常需要与 Java 层进行复杂的 JNI 调用，且系统加载器对 `sdl2` 的引用有特殊要求，因此将其设为动态链接会更加稳定且符合 Android 平台标准。

</details>

### 题目 3：如果我们发现官方的 `ffmpeg` 库缺少我们需要的一个解码器，应该在哪个目录下进行修改？

<details>
<summary>查看答案</summary>

应当在项目的 `vcpkg/ports/ffmpeg` 目录下修改 `portfile.cmake`。因为项目已经配置了 `overlay-ports`，vcpkg 会优先读取你在这个目录下定义的构建逻辑。

</details>

