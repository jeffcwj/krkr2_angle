# CMake 目标关系图

在复杂的 C++ 项目中，理解构建系统（Build System）如何组织代码是至关重要的。KrKr2 使用 CMake 作为跨平台构建工具，并定义了一套清晰的目标（Targets）依赖关系，以确保在不同操作系统下都能高效编译和运行。

## 1. 核心目标全貌

KrKr2 的 CMake 体系主要围绕以下三个核心目标构建：

- **krkr2**: 最终的产出物。在 Windows/Linux/macOS 上是可执行文件 (.exe 或二进制程序)；在 Android 上则编译为 JNI 调用的共享库 (.so)。
- **krkr2plugin**: 一个静态库 (**STATIC**)，包含了 `cpp/plugins/` 目录下所有的插件实现。
- **krkr2core**: 一个接口库 (**INTERFACE**)，它本身不产生编译产物，但它整合了 `cpp/core/` 下所有 9 个核心模块，作为依赖传递的枢纽。

## 2. 目标依赖关系图

以下是 KrKr2 构建目标的逻辑拓扑图：

```text
       [ krkr2 ] (Executable / Shared Library)
           |
           +------> [ krkr2plugin ] (STATIC Library)
           |             |
           |             +--- psdfile, psbfile, motionplayer, etc.
           |
           +------> [ krkr2core ] (INTERFACE Library)
                         |
                         +--- tjs2
                         +--- core_base_module
                         +--- core_environ_module
                         +--- core_extension_module
                         +--- core_plugin_module
                         +--- core_movie_module
                         +--- core_sound_module
                         +--- core_visual_module
                         +--- core_utils_module
```

### INTERFACE 库 vs STATIC 库的设计思考

- **STATIC 库 (krkr2plugin)**: 插件代码被编译为 `.a` 或 `.lib` 文件。由于插件数量众多且逻辑相对独立，这种方式可以加快增量编译速度，并且在链接时由链接器决定包含哪些符号。
- **INTERFACE 库 (krkr2core)**: 核心模块通过 INTERFACE 方式聚合，其主要目的是“依赖传播”。它告诉编译器：如果你链接了 `krkr2core`，那么你也会自动包含核心模块的所有头文件路径和编译选项。这种设计避免了在 `krkr2` 主程序中手动列出所有核心文件夹的繁琐操作。

## 3. 平台条件编译

在根目录的 `CMakeLists.txt` 中，KrKr2 通过条件分支来处理不同平台的特殊需求。

注意：KrKr2 使用了自定义定义的 CMake 变量（WINDOWS, LINUX, MACOS, ANDROID）来进行分支控制，这比标准的 `CMAKE_SYSTEM_NAME` 更直观：

```cmake
if(WINDOWS)
    # Windows 特定的源文件（如 WinMain.cpp）和库依赖
elseif(LINUX)
    # Linux 特定的 X11 或 SDL2 依赖
elseif(ANDROID)
    # Android JNI 配置、NDK 工具链设置
elseif(MACOS)
    # Apple 平台的 Bundle 配置与 Info.plist
endif()
```

## 4. CMake 预设 (CMakePresets.json)

为了简化繁琐的命令行构建参数，KrKr2 使用了 **CMake Presets**。这是一种标准的 JSON 格式文件，它定义了针对不同平台和编译类型的“一键配置”。

常见的预设包括：

- **Windows Debug / Release**: 使用 MSVC 编译器，generator 为 `Ninja`，binary 目录位于 `build/windows-debug`。
- **Linux Debug / Release**: 适配 GCC/Clang，同样使用 `Ninja` 构建系统以获得更快的编译速度。
- **MacOS Debug / Release**: 配置苹果平台的 Bundle 路径和 SDK 设置。

### 预设的内容分析

在 `CMakePresets.json` 中，你会看到类似的配置项：

- **generator**: Ninja（轻量级构建系统，编译速度极快）。
- **binaryDir**: `${sourceDir}/out/build/${presetName}`（指定编译中间产物存放位置）。
- **cacheVariables**:
  - `CMAKE_BUILD_TYPE`: Debug 或 Release。
  - `VCPKG_TARGET_TRIPLET`: 如 `x64-windows` 或 `x64-linux`，指示 vcpkg 使用哪套库。

## 练习题与答案

1. **选择题**：KrKr2 中的 `krkr2core` 目标在 CMake 中被定义为什么类型的库？
   A. STATIC（静态库）
   B. SHARED（动态库）
   C. INTERFACE（接口库）
   D. EXECUTABLE（可执行程序）

2. **填空题**：KrKr2 使用自定义的 CMake 变量 ______ 来区分 Android 平台的编译逻辑。

3. **简答题**：为什么 KrKr2 选择使用 `Ninja` 作为 CMake 预设中的生成器（Generator）？

<details>
<summary>查看答案</summary>

1. **C**（krkr2core 作为一个 INTERFACE 库，主要用于聚合和传递依赖）。
2. **ANDROID**（在 CMakeLists.txt 中通过 `if(ANDROID)` 进行条件分支判断）。
3. **回答**：`Ninja` 相比传统的 `Make` 或 `Visual Studio Solution` 更加轻量且专注于速度。它能更智能地处理依赖并行计算，极大地缩短了大型项目（如 KrKr2）的编译和链接时间。

</details>
