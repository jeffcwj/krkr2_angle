# 01-根 CMakeLists 解读

在前面的章节中，你已经学习了 CMake 的基础语法、生成器表达式（Generator Expressions）以及如何集成 vcpkg。现在，我们要深入 KrKr2 项目的“大脑”——位于项目根目录下的 `CMakeLists.txt`。

这份文件共有 143 行，它不仅是整个构建过程的起点，还负责配置全局环境、处理跨平台差异以及协调各个子模块。

---

## 1. 文件概览

根目录下的 `CMakeLists.txt` 主要承担以下职责：
- **环境初始化**：设置 CMake 最低版本要求，检测并启用编译器缓存。
- **全局宏定义**：统一 Debug 和 Release 模式下的预处理器宏。
- **包管理集成**：针对不同平台（特别是 Android）配置 vcpkg 工具链。
- **项目属性设定**：定义项目名称、C++ 标准版本等。
- **平台入口分发**：根据操作系统决定生成可执行文件（Executable）还是共享库（Shared Library）。
- **子目录组装**：通过 `add_subdirectory` 将 core 核心库、plugins 插件库和第三方库连接在一起。

---

## 2. 逐段解读

我们将代码拆分为逻辑段落，为你详细解析每一行的作用。

### 第一段：版本与 Ccache 加速

```cmake
cmake_minimum_required(VERSION 3.28)

find_program(CCACHE_PROGRAM ccache)

if (CCACHE_PROGRAM)
    set(CMAKE_C_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
    set(CMAKE_CXX_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
    message(STATUS "Ccache found: ${CCACHE_PROGRAM}")
endif ()
```

**解析：**
1. `cmake_minimum_required(VERSION 3.28)`：KrKr2 使用了一些较新的 CMake 特性，因此要求 3.28 及以上版本。
2. `find_program(CCACHE_PROGRAM ccache)`：尝试在你的系统中寻找 `ccache`（编译器缓存工具）。
3. `CMAKE_CXX_COMPILER_LAUNCHER`：如果找到了 ccache，CMake 会在调用编译器（如 GCC 或 Clang）时套上一层 ccache。这能显著提升二次编译的速度，尤其是在频繁切换分支或清理构建目录时。

### 第二段：生成器表达式定义宏

```cmake
add_compile_definitions(
        $<$<CONFIG:Debug>:DEBUG>
        $<$<CONFIG:Debug>:_DEBUG>   # 对于 MSVC 兼容性
        $<$<NOT:$<CONFIG:Debug>>:NDEBUG>
)
```

**解析：**
这里运用了你在第 5 章学到的**生成器表达式**：
- `$<$<CONFIG:Debug>:DEBUG>`：如果当前是 Debug 模式，就定义宏 `DEBUG`。
- `_DEBUG` 是为了照顾 Windows 平台上的 MSVC 编译器，它的标准库会检查这个宏。
- `$<$<NOT:$<CONFIG:Debug>>:NDEBUG>`：如果不是 Debug 模式（即 Release、RelWithDebInfo 等），则定义 `NDEBUG`。这通常用于关闭 `assert()` 断言，优化代码性能。

### 第三段：vcpkg 工具链与 Android 适配

```cmake
if(ANDROID)
    set(VCPKG_TARGET_ANDROID ON)
    set(ENV{ANDROID_NDK_HOME} "$ENV{ANDROID_NDK}")
    add_compile_options(-Wno-inconsistent-missing-override)
    include(cmake/vcpkg_android.cmake)
else()
    set(CMAKE_TOOLCHAIN_FILE $ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake)
endif()
```

**解析：**
1. **Android 特殊处理**：如果是构建 Android 版本，我们需要设置一些环境变量（如 `ANDROID_NDK_HOME`）并引入专门的 Android 适配脚本。`-Wno-inconsistent-missing-override` 是为了忽略某些库在 Android NDK 下不规范的 C++ 代码警告。
2. **常规平台**：对于 Windows、Linux 和 macOS，我们直接指向 `VCPKG_ROOT` 下的标准 `vcpkg.cmake` 工具链文件。这能确保 `find_package` 可以自动找到你安装的第三方库。

### 第四段：项目名称与基础配置

```cmake
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

**解析：**
1. `set(APP_NAME krkr2)`：我们定义了一个变量，方便后面统一管理项目名称。
2. `project(${APP_NAME})`：正式声明项目。这会产生 `${PROJECT_NAME}` 变量，值为 `krkr2`。
3. `option()`：定义了两个开关。你在使用 `cmake -DENABLE_TESTS=OFF ..` 命令行时可以动态修改它们。默认是开启的。
4. `CMAKE_CXX_STANDARD 17`：KrKr2 使用 C++17 标准，这是目前的行业主流，能够使用 `std::filesystem` 等强大的新特性。
5. `CMAKE_POSITION_INDEPENDENT_CODE ON`：启用**地址无关代码**（Position Independent Code），这是构建动态库（.so/.dll）时必需的。

### 第五段：MSVC 特殊编译选项

```cmake
if(MSVC)
    add_compile_options("$<$<COMPILE_LANGUAGE:CXX>:/EHsc>" "/MP" "/utf-8")
    add_link_options("/ignore:4099" "/INCREMENTAL" "/DEBUG:FASTLINK")
endif()
```

**解析：**
当 CMake 检测到你使用的是 Visual Studio 的编译器（MSVC）时，会额外添加这些参数：
- `/EHsc`：指定 C++ 的异常处理模型（Standard C++ Exception Handling）。
- `/MP`：**多处理器编译**（Multi-Processor）。让 MSVC 开启多个线程同时编译，大大缩短构建时间。
- `/utf-8`：强制使用 UTF-8 编码读取源文件。如果你在源文件里写了中文注释，这个选项能有效防止乱码导致的编译错误。
- `/INCREMENTAL`：开启**增量链接**。只重写改变的部分，加快链接速度。

### 第六段：四平台入口分发

```cmake
if(ANDROID)
    add_library(${PROJECT_NAME} SHARED ${CMAKE_CURRENT_SOURCE_DIR}/platforms/android/cpp/krkr2_android.cpp)
    find_package(unofficial-breakpad CONFIG REQUIRED)
    target_link_libraries(${PROJECT_NAME} PUBLIC unofficial::breakpad::libbreakpad_client)
elseif(LINUX)
    add_executable(${PROJECT_NAME} ${CMAKE_CURRENT_SOURCE_DIR}/platforms/linux/main.cpp)
elseif(WINDOWS)
    add_executable(${PROJECT_NAME}
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/windows/main.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/windows/game.rc
    )
elseif(MACOS)
    set(APP_UI_RES
            ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/Icon.icns
            ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/Info.plist
    )
    add_executable(${PROJECT_NAME}
            ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/main.cpp
            ${CMAKE_CURRENT_SOURCE_DIR}/platforms/apple/macos/Prefix.pch
            ${APP_UI_RES}
    )
endif()
```

**解析：**
这里是整份文件的核心分支逻辑：
- **Android**：由于 Android 应用是通过 JNI（Java Native Interface）调用 C++，因此必须构建为 `SHARED` 动态库。
- **Windows**：除了 `main.cpp`，还加入了一个 `.rc` 资源文件。这个资源文件里通常包含了应用的图标（Icon）和版本信息。
- **macOS**：苹果系统要求极其严格。我们需要显式将 `Info.plist`（配置文件）和 `Icon.icns`（图标）添加到可执行文件的构建列表中。

### 第七段：子目录组装与链接

```cmake
target_include_directories(${PROJECT_NAME} PUBLIC ${KRKR2CORE_PATH}/environ/cocos2d)

add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/cpp/external)
add_subdirectory(${KRKR2CORE_PATH})
add_subdirectory(${KRKR2PLUGIN_PATH})

target_link_libraries(${PROJECT_NAME} PUBLIC krkr2plugin krkr2core)
```

**解析：**
1. `add_subdirectory`：CMake 会进入子目录并寻找那里的 `CMakeLists.txt`。
2. `target_link_libraries`：这是“组装”的时刻。我们将主应用（krkr2）与插件库（krkr2plugin）以及核心库（krkr2core）连接在一起。即使子模块有很多复杂的依赖，CMake 也会通过**链接属性传播**（Link Properties Propagation）帮你处理好。

### 第八段：测试与工具开关

```cmake
if(ENABLE_TESTS AND NOT (IOS OR ANDROID))
    enable_testing()
    add_subdirectory(tests)
endif()

if(BUILD_TOOLS AND NOT (IOS OR ANDROID))
    add_subdirectory(tools)
endif()
```

**解析：**
---

## 3. 项目架构图

通过上面的分析，我们可以画出 KrKr2 的依赖关系图：

```text
       [ krkr2 (Executable/Shared Lib) ]
                |
       +--------+---------+
       |                  |
[ krkr2plugin ]    [ krkr2core ] (INTERFACE)
       |                  |
       |          +-------+-------+-------+
       |          |               |       |
    (Plugins)  [tjs2]       [visual]  [base] ... (9 Modules)
```

**关键点：**
- `krkr2` 是最终的产物（在 Windows 是 .exe，在 Android 是 .so）。
- `krkr2core` 作为一个 `INTERFACE` 库，起到了“粘合剂”的作用，它本身不含代码，但定义了所有核心模块的依赖关系。

---

## 练习题与答案

### 题目 1：概念理解

在根 `CMakeLists.txt` 中，为什么 Android 平台使用 `add_library(... SHARED ...)` 而不是 `add_executable()`？

<details>
<summary>查看答案</summary>

因为 Android 系统的原生代码（NDK）是以插件形式被 Java/Kotlin 层调用的。Android 系统会通过 `System.loadLibrary()` 加载一个动态链接库（.so 文件），而不是直接运行一个普通的 C++ 二进制可执行文件。因此必须声明为 `SHARED` 库。

</details>

### 题目 2：实战操作

如果你想在编译时临时关闭测试模块的构建，应该在命令行输入什么指令？

<details>
<summary>查看答案</summary>

应该在执行 `cmake` 配置命令时，通过 `-D` 参数修改选项：
```bash
cmake -DENABLE_TESTS=OFF ..
```

</details>

### 题目 3：代码分析

分析以下代码段，解释 `/MP` 选项的作用，并说明为什么它被包裹在 `if(MSVC)` 中。
```cmake
if(MSVC)
    add_compile_options("/MP")
endif()
```

<details>
<summary>查看答案</summary>

- **作用**：`/MP` 代表 Multi-Processor Compilation（多处理器编译）。它告诉 MSVC 编译器开启多个进程同时编译源文件，利用多核 CPU 的性能来大幅提升编译速度。
- **原因**：`/MP` 是 MSVC 编译器特有的命令行参数。如果你在 Linux 上使用 GCC 或 Clang 时传入这个参数，编译器会报错退出。因此必须使用 `if(MSVC)` 进行平台判断。

</details>

