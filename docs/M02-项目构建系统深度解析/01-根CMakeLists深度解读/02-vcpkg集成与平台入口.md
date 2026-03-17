# 第 3 段：Android 与桌面平台的 vcpkg 集成（第 17-24 行）
## 第 3 段：Android 与桌面平台的 vcpkg 集成（第 17-24 行）

### 源码

```cmake
# 文件：krkr2/CMakeLists.txt 第 17-24 行
if(ANDROID)
    set(VCPKG_TARGET_ANDROID ON)
    set(ENV{ANDROID_NDK_HOME} "$ENV{ANDROID_NDK}")
    add_compile_options(-Wno-inconsistent-missing-override)
    include(cmake/vcpkg_android.cmake)
else()
    set(CMAKE_TOOLCHAIN_FILE $ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake)
endif()
```

### 逐行解析

这段代码处理一个关键问题：**如何在 Android 和桌面平台上分别集成 vcpkg**。

**第 17 行：`if(ANDROID)`**

`ANDROID` 变量由 Android NDK 的 toolchain 文件（`android.toolchain.cmake`）设置。当通过 Gradle 构建 Android 目标时，Gradle 会自动传入这个 toolchain 文件，因此 `ANDROID` 为 `TRUE`。在桌面平台上，这个变量未定义，`if(ANDROID)` 为假。

**第 18 行：`set(VCPKG_TARGET_ANDROID ON)`**

设置一个自定义标志变量，用于在后面包含的 `vcpkg_android.cmake` 中做条件判断。注意这**不是**vcpkg 官方变量——它是项目自定义的触发器。

**第 19 行：`set(ENV{ANDROID_NDK_HOME} "$ENV{ANDROID_NDK}")`**

这行做的是**环境变量重映射**。Android SDK 标准设置的环境变量名是 `ANDROID_NDK`，但 `vcpkg_android.cmake` 脚本期望的是 `ANDROID_NDK_HOME`。这行代码将前者的值复制到后者，确保两个工具链都能找到 NDK。

`ENV{}` 语法用于在 CMake 中读写**进程环境变量**（不是 CMake 缓存变量）：

```cmake
set(ENV{VAR_NAME} "value")   # 写入环境变量
message("$ENV{VAR_NAME}")    # 读取环境变量
```

**第 20 行：`add_compile_options(-Wno-inconsistent-missing-override)`**

Android 的 NDK Clang 编译器在编译 KrKr2 的某些代码时会报 `-Winconsistent-missing-override` 警告——这是因为继承的虚函数有些加了 `override` 有些没加。`-Wno-` 前缀关闭这个警告。

> **设计权衡：** 更好的做法是修复所有缺少 `override` 的地方，但由于 KrKr2 移植了大量原版 KiriKiri2 代码，全面修复工作量大且可能引入 bug，所以暂时选择抑制警告。

**第 21 行：`include(cmake/vcpkg_android.cmake)`**

包含自定义的 Android vcpkg 集成脚本。这个脚本在第 05 章详细分析。它的核心功能是**将 vcpkg toolchain 和 Android NDK toolchain 叠加**——通过 `VCPKG_CHAINLOAD_TOOLCHAIN_FILE` 机制实现。

**第 23 行：`set(CMAKE_TOOLCHAIN_FILE $ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake)`**

在桌面平台（Windows/Linux/macOS）上，直接设置 `CMAKE_TOOLCHAIN_FILE` 指向 vcpkg 的 toolchain 文件。这个文件会在 `project()` 命令执行时被加载，接管包查找机制。

### 为什么 Android 不能直接设置 CMAKE_TOOLCHAIN_FILE？

Android 构建已经有一个 toolchain 文件（`android.toolchain.cmake`），CMake 在一次配置中只能有一个 `CMAKE_TOOLCHAIN_FILE`。vcpkg 的解决方案是**链式加载**：

```
CMAKE_TOOLCHAIN_FILE = vcpkg.cmake
    └── VCPKG_CHAINLOAD_TOOLCHAIN_FILE = android.toolchain.cmake
```

vcpkg.cmake 在执行完自己的逻辑后，会自动 `include()` chainload 的 toolchain 文件。

### 常见错误

**问题 1：`VCPKG_ROOT` 环境变量未设置**

```
CMake Error at CMakeLists.txt:23:
  set CMAKE_TOOLCHAIN_FILE to "/scripts/buildsystems/vcpkg.cmake"
```

路径中缺少 VCPKG_ROOT 前缀，说明环境变量为空。解决方案：

```bash
# Linux/macOS
export VCPKG_ROOT=/path/to/vcpkg

# Windows (PowerShell)
$env:VCPKG_ROOT = "C:\src\vcpkg"

# Windows (CMD)
set VCPKG_ROOT=C:\src\vcpkg
```

**问题 2：Android 构建找不到包**

如果 vcpkg 安装了桌面版本的包但没有 Android 版本，`find_package()` 会失败。需要用正确的 triplet 安装：

```bash
vcpkg install --triplet arm64-android
```

---

## 第 4 段：项目声明与全局设置（第 26-38 行）

### 源码

```cmake
# 文件：krkr2/CMakeLists.txt 第 26-38 行
set(APP_NAME krkr2)

project(${APP_NAME})

option(ENABLE_TESTS "enable tests execute build(exclude android)" ON)
option(BUILD_TOOLS "build tools execute build(exclude android ios)" ON)

set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

set(KRKR2CORE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cpp/core)
set(KRKR2PLUGIN_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cpp/plugins)
```

### 逐行解析

**第 26 行：`set(APP_NAME krkr2)`**

将项目名存入变量，避免后续硬编码字符串。在 CMake 中，使用变量引用项目名是最佳实践——如果未来需要改名（比如从 `krkr2` 改为 `krkr2-emu`），只需修改这一处。

**第 28 行：`project(${APP_NAME})`**

`project()` 命令是 CMake 构建的**核心声明**。它做的事情远比看起来多：

1. 设置 `PROJECT_NAME` 变量为 `krkr2`
2. 设置 `CMAKE_PROJECT_NAME` 变量（仅在根 CMakeLists.txt 中）
3. **触发 toolchain 文件加载**——`CMAKE_TOOLCHAIN_FILE` 在此时被 include
4. 检测编译器（C/CXX），设置 `CMAKE_C_COMPILER`、`CMAKE_CXX_COMPILER`
5. 设置项目级变量：`PROJECT_SOURCE_DIR`、`PROJECT_BINARY_DIR`

> **重要：** 在 `project()` 之前设置 `CMAKE_TOOLCHAIN_FILE` 是因为 toolchain 必须在编译器检测**之前**加载。如果顺序反了，vcpkg 的 toolchain 就不会生效。

**第 30-31 行：选项定义**

```cmake
option(ENABLE_TESTS "enable tests execute build(exclude android)" ON)
option(BUILD_TOOLS "build tools execute build(exclude android ios)" ON)
```

`option()` 定义布尔缓存变量，可在配置时通过命令行覆盖：

```bash
cmake -B build -DENABLE_TESTS=OFF -DBUILD_TOOLS=OFF
```

两个选项默认都是 `ON`，但在 Android/iOS 构建中会被后面的条件判断跳过（第 136-143 行）。

**第 33-35 行：语言标准和 PIC**

```cmake
set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
```

- `CMAKE_CXX_STANDARD 17`：要求所有目标使用 C++17 标准。KrKr2 使用了 `std::optional`、`std::filesystem`、结构化绑定等 C++17 特性
- `CMAKE_C_STANDARD 17`：C 语言也设为 C17（C11 的 bug 修复版本）
- `CMAKE_POSITION_INDEPENDENT_CODE ON`：生成位置无关代码（`-fPIC`）。这在 Linux 上构建共享库时是**必须的**，在 Android 上构建 `.so` 也需要

**第 37-38 行：路径变量**

```cmake
set(KRKR2CORE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cpp/core)
set(KRKR2PLUGIN_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cpp/plugins)
```

定义两个路径变量供后续使用。`${CMAKE_CURRENT_SOURCE_DIR}` 是当前 CMakeLists.txt 所在目录的绝对路径。这些变量在子目录的 CMakeLists.txt 中也可访问（因为 CMake 变量默认向子作用域传播）。

---

## 第 4.5 段：MSVC 编译选项（第 40-44 行）

### 源码

```cmake
# 文件：krkr2/CMakeLists.txt 第 40-44 行
if(MSVC)
    add_compile_options("$<$<COMPILE_LANGUAGE:CXX>:/EHsc>" "/MP" "/utf-8")
    add_link_options("/ignore:4099" "/INCREMENTAL" "/DEBUG:FASTLINK")
    # set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()
```

### 逐行解析

**`if(MSVC)`** — 仅在使用 MSVC 编译器时生效。MinGW、Clang-CL 等不会进入此分支。

**编译选项解析：**

| 选项 | 含义 | 为什么需要 |
|------|------|-----------|
| `$<$<COMPILE_LANGUAGE:CXX>:/EHsc>` | 仅对 C++ 文件启用标准 C++ 异常处理 | KrKr2 的 TJS2 引擎使用 try/catch，没有这个选项 MSVC 不会生成异常处理代码 |
| `/MP` | 多进程并行编译（每个 .cpp 一个进程） | 加速编译，类似于 `make -j` 但在 MSVC 的 cl.exe 层面实现 |
| `/utf-8` | 将源文件和执行字符集都设为 UTF-8 | KrKr2 源码包含中文注释和 Unicode 字符串字面量 |

> **注意：** `$<$<COMPILE_LANGUAGE:CXX>:/EHsc>` 使用了生成器表达式确保 `/EHsc` 只传给 C++ 编译，不传给 C 编译。C 语言没有异常机制，传 `/EHsc` 会产生警告。

**链接选项解析：**

| 选项 | 含义 | 为什么需要 |
|------|------|-----------|
| `/ignore:4099` | 忽略 LNK4099 警告（找不到 PDB 文件） | 第三方库（vcpkg 安装的）经常没有附带 PDB，这个警告无害 |
| `/INCREMENTAL` | 启用增量链接 | 只重新链接改动的 .obj，大幅缩短链接时间 |
| `/DEBUG:FASTLINK` | 使用快速链接的调试信息格式 | 调试信息留在各 .obj 中不合并，链接更快（需 VS 调试器支持） |

**被注释掉的行：`CMAKE_MSVC_RUNTIME_LIBRARY`**

```cmake
# set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
```

这行如果启用，会将 CRT（C 运行时库）从默认的 DLL 版本（`/MD`）切换为静态版本（`/MT`）。当前被注释说明项目使用的是 DLL 版 CRT——这与 `CMakePresets.json` 中的 triplet `x64-windows-static-md` 一致（static 库 + MD 运行时）。

---

## 第 5 段：四平台入口文件注册（第 46-84 行）

### 源码

```cmake
# 文件：krkr2/CMakeLists.txt 第 46-84 行
if(ANDROID)
    add_library(${PROJECT_NAME} SHARED
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/android/cpp/krkr2_android.cpp)
    find_package(unofficial-breakpad CONFIG REQUIRED)
    target_link_libraries(${PROJECT_NAME} PUBLIC
        unofficial::breakpad::libbreakpad_client)
elseif(LINUX)
    add_executable(${PROJECT_NAME}
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/linux/main.cpp)
elseif(WINDOWS)
    add_executable(${PROJECT_NAME}
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/windows/main.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/windows/game.rc)
elseif(IOS)
    # ... (完整代码已注释掉，iOS 平台暂未维护)
elseif(MACOS)
    set(APP_UI_RES
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/Icon.icns
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/Info.plist)
    add_executable(${PROJECT_NAME}
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/main.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/Prefix.pch
        ${APP_UI_RES})
endif()
```

### 关键设计决策分析

**1. Android 是共享库（SHARED），其他平台是可执行文件（EXECUTABLE）**

这是最重要的架构差异。Android 应用由 Java/Kotlin 层驱动，C++ 代码编译为 `.so` 共享库，在运行时通过 JNI 加载。而桌面平台直接生成可执行文件。

```
Android:    Java Activity → System.loadLibrary("krkr2") → .so
桌面平台:    操作系统 → 直接运行 krkr2.exe / krkr2
```

**2. Android 额外链接 Breakpad**

```cmake
find_package(unofficial-breakpad CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PUBLIC unofficial::breakpad::libbreakpad_client)
```

[Breakpad](https://chromium.googlesource.com/breakpad/breakpad/) 是 Google 开发的崩溃报告框架。Android 上 C++ 崩溃不会产生 Java 异常，默认只有一个 SIGSEGV 信号然后进程静默退出。Breakpad 能捕获 native crash 并生成 minidump 文件，用于事后调试。

桌面平台为什么不需要？因为：
- Windows 有 WER（Windows Error Reporting）和直接的调试器附加
- Linux 有 core dump（`ulimit -c unlimited`）
- macOS 有 CrashReporter

**3. Windows 包含 .rc 资源文件**

```cmake
${CMAKE_CURRENT_SOURCE_DIR}/platforms/windows/game.rc
```

`.rc`（Resource Script）文件定义了 Windows 应用的图标、版本信息、字符串表等。没有这个文件，生成的 .exe 在资源管理器中会显示默认图标。

**4. macOS 包含 App Bundle 资源**

```cmake
set(APP_UI_RES
    ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/Icon.icns
    ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/Info.plist)
```

macOS 应用以 `.app` Bundle 形式分发。`Icon.icns` 是应用图标（macOS 专用格式），`Info.plist` 定义 Bundle 元数据（应用名、版本号、最低系统版本等）。

**5. iOS 代码被完整注释掉**

```cmake
elseif(IOS)
#        list(APPEND GAME_HEADER ...)
#        set(APP_UI_RES ...)
#        list(APPEND GAME_SOURCE ...)
```

这表明 iOS 移植曾经开始过但未完成。注释保留了代码结构作为未来参考。

### 自定义平台变量 vs CMAKE_SYSTEM_NAME

注意这里用的是 `WINDOWS`、`LINUX`、`MACOS` 等**自定义变量**，而不是 CMake 内置的 `CMAKE_SYSTEM_NAME`。这些变量在 `CMakePresets.json` 中通过 `cacheVariables` 设置：

```json
{
  "name": "Windows Config",
  "cacheVariables": { "WINDOWS": true }
}
```

```json
{
  "name": "Linux Config",
  "cacheVariables": { "LINUX": true }
}
```

为什么不用标准的 `CMAKE_SYSTEM_NAME`？因为 `CMAKE_SYSTEM_NAME` 的值在不同环境下不完全一致（如 `Darwin` vs `macOS`），而自定义布尔变量更简洁直观。

---

## 第 6 段：子目录与链接（第 86-97 行）

### 源码

```cmake
# 文件：krkr2/CMakeLists.txt 第 86-97 行
target_include_directories(${PROJECT_NAME} PUBLIC ${KRKR2CORE_PATH}/environ/cocos2d)

# external lib
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/cpp/external)

# build library
add_subdirectory(${KRKR2CORE_PATH})
add_subdirectory(${KRKR2PLUGIN_PATH})

target_link_libraries(${PROJECT_NAME} PUBLIC
    krkr2plugin krkr2core
)
```

### 解析

**第 86 行：公开头文件目录**

`${KRKR2CORE_PATH}/environ/cocos2d` 目录包含 `AppDelegate.h` 等 Cocos2d-x 桥接头文件。使用 `PUBLIC` 意味着链接 `krkr2` 的目标也能访问这些头文件（虽然作为最终可执行文件，这里 PUBLIC 和 PRIVATE 效果相同）。

**第 89-93 行：三个 add_subdirectory 调用**

```
cpp/external/  → 第三方库（libbpg、minizip）
cpp/core/      → 核心引擎（INTERFACE 库 krkr2core）
cpp/plugins/   → 插件集合（STATIC 库 krkr2plugin）
```

`add_subdirectory()` 让 CMake 进入子目录解析其 `CMakeLists.txt`，子目录中定义的目标自动注册到全局。顺序很重要——`external` 在前，因为 `core` 和 `plugins` 可能依赖它。

**第 95-97 行：链接核心库**

```cmake
target_link_libraries(${PROJECT_NAME} PUBLIC
    krkr2plugin krkr2core
)
```

最终可执行文件（或 Android 上的 .so）链接两个库。由于 `krkr2core` 是 INTERFACE 库，链接它实际上是继承它传播的所有编译定义、包含目录和下游库依赖（如 cocos2dx、OpenMP、FFmpeg 等）。详见第 02 章。

---

