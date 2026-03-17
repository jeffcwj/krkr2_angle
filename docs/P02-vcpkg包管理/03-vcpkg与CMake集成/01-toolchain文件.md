# 01-toolchain文件：将 vcpkg 注入 CMake

在上一章中，我们已经学会了如何使用 vcpkg 安装各种库。但是，如果你现在直接打开一个 `CMakeLists.txt` 并写下 `find_package(fmt)`，CMake 仍然会告诉你找不到这个库。这是因为 CMake 默认只会在系统标准路径（如 `/usr/local` 或 Windows 的 `Program Files`）下搜索库，而不会主动去 vcpkg 的安装目录寻找。

为了让 CMake 意识到 vcpkg 的存在，我们需要用到 **CMake Toolchain File（工具链文件）**。

---

## 什么是 CMake Toolchain 文件？

CMake 工具链文件是一个 `.cmake` 脚本，它在 CMake 配置阶段（Configure）的最早期被加载。它的主要作用是告诉 CMake：
1. 使用哪个编译器（C/C++ 编译器路径）。
2. 目标平台是什么（Android、iOS、Windows 等）。
3. **在哪里寻找依赖库和头文件。**

vcpkg 正是通过提供一个通用的工具链文件，在 CMake 寻找库之前，预先将 vcpkg 的安装路径注入到 CMake 的搜索变量中。

这个神奇的文件位于：`$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake`。

---

## 设置 Toolchain 的三种方式

要让项目使用 vcpkg，你需要确保 CMake 在启动时加载了上述文件。以下是三种最常用的设置方法：

### 1. 命令行参数（最直接）

在调用 `cmake` 命令进行配置时，手动传入 `-DCMAKE_TOOLCHAIN_FILE` 参数。

**Windows:**
```powershell
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE="C:/vcpkg/scripts/buildsystems/vcpkg.cmake"
```

**Linux / macOS:**
```bash
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE="$HOME/vcpkg/scripts/buildsystems/vcpkg.cmake"
```

> **注意**：路径必须使用正斜杠 `/`，或者在 Windows 上对反斜杠进行转义。

### 2. CMakePresets.json 中设置（推荐）

这是现代 CMake 项目（包括 KrKr2）的标准做法。在 `CMakePresets.json` 的 `configurePresets` 中指定工具链文件路径。

```json
{
  "version": 3,
  "configurePresets": [
    {
      "name": "default",
      "binaryDir": "${sourceDir}/build",
      "cacheVariables": {
        "CMAKE_TOOLCHAIN_FILE": {
          "type": "FILEPATH",
          "value": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"
        }
      }
    }
  ]
}
```

这种方式的好处是，团队成员只需要设置好 `VCPKG_ROOT` 环境变量，IDE（如 VS Code 或 Visual Studio）就能自动识别并加载工具链。

### 3. vcpkg 2024+ 自动检测

在 2024 年以后的版本中，如果你的环境变量中设置了 `VCPKG_ROOT`，并且在项目的 `CMakeLists.txt` 的 `project()` 命令之前没有手动设置工具链，CMake 有时会自动尝试寻找。但为了稳定性，**强烈建议显式指定工具链文件**。

---

## VCPKG_TARGET_TRIPLET 的作用

我们在 vcpkg 基础中提到过 Triplet（三元组），它决定了库的编译架构和链接方式。在 CMake 中，vcpkg 工具链会自动识别你的系统并选择默认 Triplet。

如果你想强制指定，可以通过 `-DVCPKG_TARGET_TRIPLET` 设置：

- **Windows 静态链接 (MD)**: `x64-windows-static-md`（KrKr2 默认使用此设置，避免发布时缺少 DLL）
- **Linux**: `x64-linux`
- **macOS**: `arm64-osx` 或 `x64-osx`

在 `CMakePresets.json` 中设置示例：
```json
"cacheVariables": {
  "VCPKG_TARGET_TRIPLET": "x64-windows-static-md"
}
```

---

## Android 的特殊情况：双工具链叠加

在开发 Android 版 KrKr2 时，我们会遇到一个难题：
- Android 开发必须使用 Android NDK 提供的工具链（`android.toolchain.cmake`）。
- 使用 vcpkg 库又必须使用 vcpkg 的工具链（`vcpkg.cmake`）。

CMake 官方只允许设置 **一个** `CMAKE_TOOLCHAIN_FILE`。为了解决冲突，vcpkg 提供了一个 **Chainload（链式加载）** 机制。

### VCPKG_CHAINLOAD_TOOLCHAIN_FILE 机制

我们可以将 vcpkg 工具链设为主工具链，然后告诉 vcpkg：“在你干完活之后，再去加载 Android 的工具链”。

在 KrKr2 项目中，我们编写了一个专门的脚本 `cmake/vcpkg_android.cmake` 来处理逻辑。其核心逻辑如下：

```cmake
# cmake/vcpkg_android.cmake 逻辑详解

# 1. 检查环境变量
if(NOT DEFINED ENV{ANDROID_NDK_HOME})
    message(FATAL_ERROR "请设置 ANDROID_NDK_HOME 环境变量")
endif()

if(NOT DEFINED ENV{VCPKG_ROOT})
    message(FATAL_ERROR "请设置 VCPKG_ROOT 环境变量")
endif()

# 2. 根据 CMake 传入的 ANDROID_ABI 映射到 vcpkg 的 Triplet
if(ANDROID_ABI STREQUAL "arm64-v8a")
    set(VCPKG_TARGET_TRIPLET "arm64-android")
elseif(ANDROID_ABI STREQUAL "armeabi-v7a")
    set(VCPKG_TARGET_TRIPLET "arm-android")
elseif(ANDROID_ABI STREQUAL "x86_64")
    set(VCPKG_TARGET_TRIPLET "x64-android")
elseif(ANDROID_ABI STREQUAL "x86")
    set(VCPKG_TARGET_TRIPLET "x86-android")
endif()

# 3. 核心步骤：设置链式加载
# 让 vcpkg 去加载 NDK 的工具链
set(VCPKG_CHAINLOAD_TOOLCHAIN_FILE "$ENV{ANDROID_NDK_HOME}/build/cmake/android.toolchain.cmake" CACHE STRING "")

# 4. 将主工具链指向 vcpkg
set(CMAKE_TOOLCHAIN_FILE "$ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake" CACHE STRING "")
```

通过这种方式，Android 构建系统既能识别 NDK 的编译器配置，又能正常使用 vcpkg 安装的库。

---

## 练习题与答案

### 题目 1：在 Windows 上，如果你希望 CMake 项目使用 vcpkg 且不生成动态库（即使用静态链接），你应该如何设置命令行的参数？

<details>
<summary>查看答案</summary>

你需要同时指定工具链文件和目标 Triplet。假设 `VCPKG_ROOT` 已设置：

```powershell
cmake -B build -S . `
  -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake" `
  -DVCPKG_TARGET_TRIPLET="x64-windows-static"
```

注意：在 KrKr2 中，由于 Cocos2d-x 的特殊需求，我们通常使用 `x64-windows-static-md`。

</details>

### 题目 2：为什么 Android 开发需要使用 `VCPKG_CHAINLOAD_TOOLCHAIN_FILE` 而不是直接设置 `CMAKE_TOOLCHAIN_FILE` 为 NDK 的路径？

<details>
<summary>查看答案</summary>

1. CMake 只能接受一个 `CMAKE_TOOLCHAIN_FILE`。
2. 如果直接设为 NDK 路径，CMake 就不知道去哪里找 vcpkg 安装的库。
3. 如果直接设为 vcpkg 路径，CMake 就不知道如何交叉编译 Android 程序（找不到 NDK 的编译器和系统头文件）。
4. 使用 `VCPKG_CHAINLOAD_TOOLCHAIN_FILE` 可以让 vcpkg 充当“代理”，它在完成自己的路径注入任务后，会加载并执行 NDK 的工具链逻辑，从而实现两者的并存。

</details>
