# 测试目录结构与 CMake 配置

> **所属模块：** M10-测试与质量保证
> **前置知识：** [Catch2 简介与安装](../01-Catch2测试框架/01-Catch2简介与安装.md)、[TEST_CASE 与断言](../01-Catch2测试框架/02-TEST_CASE与断言.md)、[P01-现代 CMake 与构建工具链](../../P01-现代CMake与构建工具链/README.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：

1. 说出 KrKr2 测试目录的完整层级结构，并解释每一层的职责
2. 理解根 `tests/CMakeLists.txt` 的每一行配置含义
3. 掌握子模块 CMakeLists.txt 的"foreach 一源一可执行文件"模式
4. 区分不同测试模块的链接目标（Link Target）差异及原因
5. 独立为一个新模块创建符合项目规范的测试 CMake 配置

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| CTest | CTest | CMake 自带的测试驱动程序，负责发现、运行、汇总测试结果 |
| 测试发现 | Test Discovery | 编译后自动扫描可执行文件中的 TEST_CASE 并注册到 CTest 的过程 |
| 配置文件生成 | configure_file | CMake 命令，把模板文件（`.in`）中的变量占位符替换为实际值后输出到构建目录 |
| 构建目录 | Build Directory | CMake 生成编译产物的目录（即 `out/<平台>/debug`），与源码目录分离 |
| INTERFACE 库 | INTERFACE Library | 只传播头文件搜索路径和链接依赖、自身不生成 .a/.lib 文件的 CMake 目标 |
| PRIVATE/PUBLIC 链接 | PRIVATE/PUBLIC Linking | 控制依赖是否向下游传播——PRIVATE 仅自己用，PUBLIC 还会传递给链接本目标的其他目标 |
| 测试夹具 | Test Fixture | 测试运行前预先准备好的数据文件（如 .tjs 脚本、.psb 二进制），用于提供可重复的测试输入 |

---

## 测试目录总览

KrKr2 的测试代码全部集中在项目根目录下的 `tests/` 文件夹。这个文件夹与引擎源码 `cpp/` 平级，形成了"源码归源码、测试归测试"的清晰分离。我们先看完整的目录树：

```
tests/                              ← 测试根目录
├── CMakeLists.txt                  ← 根 CMake：查找 Catch2、注册子目录、配置夹具路径
├── test_config.h.in                ← 模板文件：CMake 会把它变成 test_config.h
├── AGENTS.md                       ← 测试目录的知识库文档
├── test_files/                     ← 测试夹具（Test Fixture）数据文件
│   ├── tjs2/                       ← TJS2 脚本夹具（.tjs 文件）
│   │   ├── test_array.tjs          ← 测试 TJS2 数组操作的脚本
│   │   ├── test_string.tjs         ← 测试 TJS2 字符串操作的脚本
│   │   └── ...                     ← 更多 .tjs 脚本
│   └── emote/                      ← E-mote 插件夹具（.pimg / .psb 二进制文件）
│       ├── sample.pimg             ← 测试用 E-mote 图片
│       └── sample.psb              ← 测试用 PSB 数据包
└── unit-tests/                     ← 单元测试代码
    ├── core/                       ← 引擎核心模块测试
    │   ├── movie/                  ← 视频模块测试
    │   │   ├── CMakeLists.txt      ← 链接 core_movie_module
    │   │   ├── main.cpp            ← 自定义入口（初始化 spdlog）
    │   │   └── ffmpeg.cpp          ← FFmpeg 初始化测试
    │   ├── tjs2/                   ← TJS2 脚本引擎测试
    │   │   ├── CMakeLists.txt      ← 链接 krkr2core
    │   │   ├── main.cpp            ← 自定义入口（初始化 spdlog）
    │   │   ├── tjs.cpp             ← TJS2 引擎加载/脚本执行测试
    │   │   └── tjsString.cpp       ← TJS2 字符串处理测试
    │   └── visual/                 ← 渲染模块测试（目前为空）
    │       └── CMakeLists.txt      ← 框架已搭好，SOURCES 为空
    └── plugins/                    ← 插件模块测试
        ├── CMakeLists.txt          ← 链接 krkr2plugin + krkr2core
        ├── main.cpp                ← 自定义入口（初始化 spdlog，含 "plugin" logger）
        └── psbfile-dll.cpp         ← PSB 文件解析器测试（最丰富的测试文件）
```

### 目录分层设计哲学

这个目录结构体现了三条设计原则：

**原则一：按被测模块分目录。** `unit-tests/core/movie/` 测试 `cpp/core/movie/` 的代码，`unit-tests/plugins/` 测试 `cpp/plugins/` 的代码。目录路径映射关系一目了然，新开发者看到测试文件就知道它测的是哪个模块。

**原则二：每个测试目录自包含。** 每个目录都有自己的 `CMakeLists.txt`（定义编译规则）和 `main.cpp`（定义程序入口）。这意味着每个测试模块可以独立编译、独立运行，互不干扰。一个模块的测试挂了，不会影响其他模块。

**原则三：夹具文件集中存放。** 所有测试数据（.tjs 脚本、.psb 二进制等）统一放在 `test_files/` 目录，而不是散落在各测试目录中。通过 CMake 的 `configure_file` 机制把这个路径注入到代码中（后面会详讲），所有测试都通过同一个宏 `TEST_FILES_PATH` 访问夹具。

下面这张图展示了从源码模块到测试模块的映射关系：

```
项目源码                                  测试代码
─────────────                           ─────────────
cpp/core/movie/    ──────────────────►  tests/unit-tests/core/movie/
cpp/core/tjs2/     ──────────────────►  tests/unit-tests/core/tjs2/
cpp/core/visual/   ──────────────────►  tests/unit-tests/core/visual/  (空)
cpp/plugins/       ──────────────────►  tests/unit-tests/plugins/

共享资源：tests/test_files/ ← 所有测试模块通过 TEST_FILES_PATH 宏访问
```

---

## 根 CMakeLists.txt 逐行解析

根配置文件 `tests/CMakeLists.txt` 只有 18 行，但每一行都有明确职责。下面是项目实际的完整文件内容（来源：`tests/CMakeLists.txt` 第 1-18 行）：

```cmake
# tests/CMakeLists.txt — 测试根配置（完整 18 行）

cmake_minimum_required(VERSION 3.16)   # [1] 要求 CMake 最低 3.16
project(Tests LANGUAGES CXX)           # [2] 声明测试子项目，仅使用 C++ 语言

find_package(Catch2 3 REQUIRED)        # [3] 查找 Catch2 v3，找不到就报错终止

include(CTest)                         # [4] 启用 CTest 测试驱动
include(Catch)                         # [5] 加载 Catch2 的 CMake 集成模块

set(TEST_CONFIG_DIR "${CMAKE_CURRENT_BINARY_DIR}")  # [6] 记录构建目录路径

add_subdirectory(unit-tests/core/movie)    # [7] 注册视频模块测试
add_subdirectory(unit-tests/core/visual)   # [8] 注册渲染模块测试
add_subdirectory(unit-tests/core/tjs2)     # [9] 注册 TJS2 模块测试

add_subdirectory(unit-tests/plugins)       # [10] 注册插件模块测试

set(TEST_FILES_PATH ${CMAKE_CURRENT_SOURCE_DIR}/test_files)  # [11] 夹具目录路径
configure_file(test_config.h.in test_config.h)               # [12] 生成配置头文件
```

我们逐行拆解每个命令的作用：

### 第 1-2 行：项目声明

```cmake
cmake_minimum_required(VERSION 3.16)
project(Tests LANGUAGES CXX)
```

`cmake_minimum_required(VERSION 3.16)` 告诉 CMake："我需要至少 3.16 版本才能正确处理这个文件"。为什么是 3.16 而不是更高版本？因为 3.16 引入了 `target_precompile_headers` 和改进的 `find_package` 行为，足以满足测试构建的需求。实际上项目主 CMakeLists.txt 要求 3.28+，但测试部分保守地只要求 3.16，这是一个好的兼容性实践。

`project(Tests LANGUAGES CXX)` 声明了一个名为 `Tests` 的子项目。`LANGUAGES CXX` 显式指定只使用 C++ 编译器——测试不需要 C 编译器或其他语言。这行还会自动设置一系列变量如 `Tests_SOURCE_DIR`、`Tests_BINARY_DIR` 等。

### 第 3 行：查找 Catch2

```cmake
find_package(Catch2 3 REQUIRED)
```

这行代码让 CMake 去系统中（或 vcpkg 包管理器中）寻找 Catch2 库。数字 `3` 表示要求主版本号 ≥ 3，这很重要——Catch2 v2 和 v3 的 API 完全不同（v2 使用头文件包含模式，v3 使用链接库模式）。`REQUIRED` 关键字表示找不到就直接终止 CMake 配置过程并报错。

在 KrKr2 项目中，Catch2 是通过 vcpkg 安装的。`vcpkg.json` 清单文件中声明了 `catch2` 依赖，vcpkg 会在配置阶段自动下载、编译并安装到 `${VCPKG_ROOT}/installed/` 目录下。`find_package` 通过 vcpkg 提供的工具链文件（Toolchain File）自动找到正确的路径。

> **常见错误：** 如果你看到 `Could not find a package configuration file provided by "Catch2"` 错误，说明 vcpkg 还没有安装 Catch2。解决方法是确保 `vcpkg.json` 中包含 `"catch2"` 依赖，然后重新运行 CMake 配置。

### 第 4-5 行：启用 CTest 与 Catch2 集成

```cmake
include(CTest)
include(Catch)
```

**`include(CTest)`** 启用 CMake 内置的测试驱动框架 CTest。CTest 是 CMake 自带的测试运行器（Test Runner），它的核心功能是：自动发现项目中注册的所有测试、按需运行、收集结果并生成报告。执行 `include(CTest)` 后，你就可以在构建目录中运行 `ctest` 命令来执行所有测试了。

**`include(Catch)`** 加载 Catch2 提供的 CMake 集成模块。这个模块定义了一个关键函数 `catch_discover_tests()`，它的工作原理是：编译完测试可执行文件后，运行这个可执行文件的 `--list-tests` 命令，解析输出中的所有 TEST_CASE 名称，然后把它们逐一注册到 CTest 中。这样每个 TEST_CASE 都会变成 CTest 中的一个独立测试条目。

两者的配合关系：

```
CTest（测试驱动器）      Catch 模块（桥梁）          Catch2（测试框架）
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ ctest 命令   │ ←── │ catch_discover   │ ←── │ TEST_CASE 宏     │
│ 发现 + 运行  │     │ _tests() 自动    │     │ 定义测试逻辑     │
│ 汇总结果     │     │ 注册每个用例     │     │ 断言 + SECTION   │
└──────────────┘     └──────────────────┘     └──────────────────┘
```

### 第 6 行：设置配置目录

```cmake
set(TEST_CONFIG_DIR "${CMAKE_CURRENT_BINARY_DIR}")
```

这行创建了一个变量 `TEST_CONFIG_DIR`，它指向当前 CMakeLists.txt 对应的构建目录（Build Directory）。在 KrKr2 中，如果你用 `cmake --preset="Windows Debug Config"` 配置，这个路径大概是 `out/windows/debug/tests/`。

为什么需要这个变量？因为第 12 行的 `configure_file` 会把生成的 `test_config.h` 输出到构建目录中，而各子目录的 CMakeLists.txt 需要知道去哪里找这个头文件。`TEST_CONFIG_DIR` 就是那个"去哪里找"的路径，子目录通过 `target_include_directories(${name} PRIVATE "${TEST_CONFIG_DIR}")` 将它加入头文件搜索路径。

### 第 7-10 行：注册子目录

```cmake
add_subdirectory(unit-tests/core/movie)
add_subdirectory(unit-tests/core/visual)
add_subdirectory(unit-tests/core/tjs2)

add_subdirectory(unit-tests/plugins)
```

`add_subdirectory()` 告诉 CMake："去指定路径找到那个目录的 CMakeLists.txt 并执行它"。执行顺序就是这里写的顺序——先 movie，再 visual，然后 tjs2，最后 plugins。

注意 `unit-tests/core/visual` 目前虽然被注册了，但它的 `SOURCES` 变量为空（即没有任何测试文件），所以 CMake 处理它时不会生成任何可执行文件。这是一个"占位"做法——框架已搭好，日后往 `SOURCES` 里加文件就能立即工作。

这里还有一个细节：`core/` 下的三个子目录和 `plugins/` 之间有一个空行分隔。这是代码风格上的分组——core 模块测试和插件测试在逻辑上属于不同层级。

### 第 11-12 行：配置测试夹具路径

```cmake
set(TEST_FILES_PATH ${CMAKE_CURRENT_SOURCE_DIR}/test_files)
configure_file(test_config.h.in test_config.h)
```

这两行是 KrKr2 测试架构中最巧妙的设计之一。它们解决了一个经典问题：**测试代码如何在运行时找到测试数据文件？**

`CMAKE_CURRENT_SOURCE_DIR` 指向 `tests/` 目录在源码树中的绝对路径（例如 `/home/dev/krkr2/tests/`）。加上 `/test_files` 后得到测试夹具目录的完整路径。

`configure_file(test_config.h.in test_config.h)` 把模板文件 `test_config.h.in` 中的占位符替换为实际值，然后输出到构建目录。模板文件内容只有 6 行：

```c
// tests/test_config.h.in — 配置模板（完整 6 行）
#ifndef KRKR2_TEST_CONFIG_H
#define KRKR2_TEST_CONFIG_H

#cmakedefine TEST_FILES_PATH "@TEST_FILES_PATH@"  // 关键行

#endif
```

CMake 会把 `@TEST_FILES_PATH@` 替换为上一行设置的绝对路径。`#cmakedefine` 是 CMake 专用的预处理指令，效果等同于 `#define`，但如果变量未定义则会变成 `/* #undef TEST_FILES_PATH */`。替换后生成的 `test_config.h` 看起来像这样：

```c
// 生成的 test_config.h（构建目录中）
#ifndef KRKR2_TEST_CONFIG_H
#define KRKR2_TEST_CONFIG_H

#define TEST_FILES_PATH "/home/dev/krkr2/tests/test_files"  // 绝对路径

#endif
```

测试代码只需 `#include "test_config.h"` 就能用 `TEST_FILES_PATH` 宏拼接出任何夹具文件的完整路径：

```cpp
// 在测试代码中使用夹具路径的典型写法
#include "test_config.h"  // 获取 TEST_FILES_PATH 宏

TEST_CASE("加载 PSB 文件") {
    // 拼接出夹具文件的完整路径
    std::string psb_path = std::string(TEST_FILES_PATH) + "/emote/sample.psb";
    // 现在可以用 psb_path 打开文件了
}
```

> **为什么不硬编码路径？** 因为不同开发者的代码存放位置不同——Windows 上可能是 `C:\Users\alice\krkr2\tests\test_files`，Linux 上可能是 `/home/bob/krkr2/tests/test_files`。用 CMake 变量替换，路径自动适配每台机器。

---

## 子模块 CMakeLists.txt——"一源一可执行文件"模式

KrKr2 测试架构最核心的设计模式是：**每个 .cpp 源文件编译成一个独立的可执行文件**。这与许多项目"所有测试编译到一个大可执行文件"的做法不同。我们以 `tests/unit-tests/plugins/CMakeLists.txt`（28 行）为蓝本来详解这个模式。

### 完整文件内容

```cmake
# tests/unit-tests/plugins/CMakeLists.txt — 插件测试（完整 28 行）

cmake_minimum_required(VERSION 3.16)
project(TestPlugins LANGUAGES CXX)        # 子项目名：TestPlugins

# === 第一部分：声明源文件 ===
set(SOURCES
        psbfile-dll.cpp                    # PSB 文件解析器测试
)

# === 第二部分：源文件名 → 可执行目标名 ===
string(REPLACE ".cpp" "" BASENAMES_SOURCES "${SOURCES}")
# 结果：BASENAMES_SOURCES = "psbfile-dll"
set(TARGETS ${BASENAMES_SOURCES})
# 结果：TARGETS = "psbfile-dll"

# === 第三部分：为每个目标创建可执行文件 ===
foreach(name ${TARGETS})
    add_executable(${name} ${name}.cpp main.cpp)
    # 每个可执行文件 = 对应的测试源文件 + 共享的 main.cpp
endforeach()

# === 第四部分：配置链接和测试发现 ===
set(ALL_TARGETS
        ${TARGETS}
)

foreach(name ${ALL_TARGETS})
    target_link_libraries(${name}
        PRIVATE
            Catch2::Catch2              # Catch2 库（PRIVATE = 不传播）
        PUBLIC
            krkr2plugin krkr2core       # 被测模块库（PUBLIC = 向下传播）
    )
    target_include_directories(${name} PRIVATE "${TEST_CONFIG_DIR}")
    catch_discover_tests(${name})       # 自动发现并注册测试
endforeach()
```

### 逐步分析

**步骤 1：声明源文件列表**

```cmake
set(SOURCES
        psbfile-dll.cpp
)
```

`SOURCES` 变量列出所有测试源文件。如果要加新测试，只需在这里追加一行即可。例如：

```cmake
set(SOURCES
        psbfile-dll.cpp
        psdfile-dll.cpp      # 新增的 PSD 解析器测试
        csvparser-dll.cpp    # 新增的 CSV 解析器测试
)
```

**步骤 2：自动推导目标名**

```cmake
string(REPLACE ".cpp" "" BASENAMES_SOURCES "${SOURCES}")
set(TARGETS ${BASENAMES_SOURCES})
```

`string(REPLACE)` 是 CMake 的字符串替换命令。它把每个源文件名中的 `.cpp` 后缀去掉，得到"基础名"。例如 `psbfile-dll.cpp` → `psbfile-dll`。这个基础名同时用作：
- 可执行文件名（编译后生成 `psbfile-dll` 或 `psbfile-dll.exe`）
- CMake 目标名（用于 `target_link_libraries` 等命令）

这种自动命名避免了手动维护"文件名→目标名"的映射表，减少出错机会。

**步骤 3：循环创建可执行文件**

```cmake
foreach(name ${TARGETS})
    add_executable(${name} ${name}.cpp main.cpp)
endforeach()
```

对 `TARGETS` 列表中的每个名字，创建一个可执行目标。每个可执行文件包含两个源文件：
- `${name}.cpp`——测试逻辑（如 `psbfile-dll.cpp` 中的 TEST_CASE）
- `main.cpp`——共享的程序入口（初始化 spdlog 日志后调用 `Catch::Session().run()`）

> **为什么不用 `Catch2::Catch2WithMain`？** Catch2 v3 提供了 `Catch2WithMain` 目标，它内置了一个默认的 `main()` 函数。但 KrKr2 引擎代码在运行时会调用 `spdlog::get("core")` 等日志接口，如果这些 logger 没有提前创建，就会返回空指针并崩溃。因此项目使用自定义 `main.cpp`，在调用 `Catch::Session().run()` 之前先初始化所有必要的 spdlog logger。详见下一节"自定义 main 与 spdlog 集成"。

**步骤 4：配置链接和测试发现**

```cmake
foreach(name ${ALL_TARGETS})
    target_link_libraries(${name}
        PRIVATE Catch2::Catch2
        PUBLIC  krkr2plugin krkr2core
    )
    target_include_directories(${name} PRIVATE "${TEST_CONFIG_DIR}")
    catch_discover_tests(${name})
endforeach()
```

第二个 `foreach` 为每个目标配置三件事：

1. **链接库**：`Catch2::Catch2` 用 PRIVATE 链接（测试框架只给自己用），`krkr2plugin` 和 `krkr2core` 用 PUBLIC 链接（被测库的传递依赖也需要可见）。
2. **头文件路径**：把 `TEST_CONFIG_DIR`（即构建目录）加入搜索路径，这样 `#include "test_config.h"` 才能找到生成的配置头文件。
3. **测试发现**：`catch_discover_tests()` 在编译后运行可执行文件的 `--list-tests` 选项，把发现的每个 TEST_CASE 注册到 CTest。

---

## 四个测试模块的链接目标差异

KrKr2 有 4 个测试子目录，它们的 CMakeLists.txt 结构完全相同（同一个 foreach 模板），唯一的区别在于 `target_link_libraries` 中链接的目标。下表对比了所有差异：

| 测试模块 | 项目名 | 链接目标 | 测试文件 | 测试数量 |
|----------|--------|---------|----------|---------|
| `core/movie/` | TestMovieModule | `core_movie_module` | ffmpeg.cpp | 1 个 |
| `core/tjs2/` | TestTjs2 | `krkr2core` | tjs.cpp, tjsString.cpp | 2 个 |
| `core/visual/` | TestVisualModule | `krkr2core` | （空） | 0 个 |
| `plugins/` | TestPlugins | `krkr2plugin krkr2core` | psbfile-dll.cpp | 1 个 |

### 为什么链接目标不同？

每个测试模块只需要链接它**实际依赖**的库。理解这些差异需要知道 KrKr2 的 CMake 目标层级：

```
krkr2core (INTERFACE 库)
├── tjs2                    ← TJS2 脚本引擎模块
├── core_base_module        ← 归档/IO 模块
├── core_environ_module     ← 平台抽象层模块
├── core_movie_module       ← 视频播放模块
├── core_sound_module       ← 音频模块
├── core_visual_module      ← 渲染模块
├── core_utils_module       ← 工具模块
├── core_extension_module   ← 扩展 API 模块
└── core_plugin_module      ← 插件加载/绑定模块

krkr2plugin (STATIC 库)
├── psbfile                 ← PSB 文件解析插件
├── psdfile                 ← PSD 文件解析插件
├── layerExDraw             ← 扩展图层绘制插件
├── motionplayer            ← E-mote 动画播放插件
└── fstat                   ← 文件状态插件
```

- **movie 测试**只需要 `core_movie_module`，因为它只测 FFmpeg 初始化功能，不需要完整的引擎环境。链接单个模块比链接整个 `krkr2core` 更快、依赖更少。
- **tjs2 测试**需要 `krkr2core`（整个核心 INTERFACE 库），因为 TJS2 引擎初始化时会触及多个核心模块（base、environ 等）的代码。
- **visual 测试**预留了 `krkr2core` 链接，但目前没有任何测试文件。
- **plugins 测试**需要 `krkr2plugin`（所有插件代码）和 `krkr2core`（引擎核心），因为插件代码内部依赖引擎核心的数据类型和接口。

### PRIVATE 与 PUBLIC 链接的语义

在 `target_link_libraries` 中，`PRIVATE` 和 `PUBLIC` 控制的是依赖传播行为：

```cmake
target_link_libraries(psbfile-dll
    PRIVATE Catch2::Catch2        # PRIVATE：只有 psbfile-dll 自己能用 Catch2 的头文件
    PUBLIC  krkr2plugin krkr2core  # PUBLIC：krkr2plugin 的依赖也传递给 psbfile-dll
)
```

用一个类比来理解：假设 A 链接 B。
- **PRIVATE**：A 用了 B 的功能，但 A 的"客户"不知道 B 的存在。类似于你在厨房用了某个调料（B），但端出来的菜（A）不会让食客直接接触到那个调料。
- **PUBLIC**：A 用了 B 的功能，而且 A 的"客户"也需要知道 B。类似于你做的菜里有花生（B），食客必须知道有花生（因为可能过敏）。

在测试场景中：
- `Catch2::Catch2` 用 PRIVATE——测试可执行文件用 Catch2 框架，但没有其他目标会链接这个测试可执行文件，所以不需要传播。
- `krkr2core` 用 PUBLIC——因为 `krkr2core` 是 INTERFACE 库，它自身的传递依赖（如 spdlog、FFmpeg 头文件路径等）需要对测试可执行文件可见。如果用 PRIVATE，那些传递依赖就会丢失，导致编译错误。

---

## 动手实践：为 `core/utils` 模块创建测试目录

现在我们来亲手为一个还没有测试的模块——`core/utils`（工具模块，包含计时器、线程、剪贴板等功能）——创建完整的测试配置。

### 第 1 步：创建目录结构

```bash
# 在项目根目录执行
mkdir -p tests/unit-tests/core/utils
```

创建后目录结构变为：

```
tests/unit-tests/core/
├── movie/
├── tjs2/
├── visual/
└── utils/          ← 新创建
```

### 第 2 步：编写 CMakeLists.txt

在 `tests/unit-tests/core/utils/` 目录下创建 `CMakeLists.txt`：

```cmake
# tests/unit-tests/core/utils/CMakeLists.txt — 工具模块测试

cmake_minimum_required(VERSION 3.16)
project(TestUtilsModule LANGUAGES CXX)  # 项目名遵循 Test{模块名} 惯例

# 声明测试源文件列表——目前只有一个
set(SOURCES
        timer.cpp                        # 计时器功能测试
)

# 去掉 .cpp 后缀得到目标名：timer
string(REPLACE ".cpp" "" BASENAMES_SOURCES "${SOURCES}")
set(TARGETS ${BASENAMES_SOURCES})

# 为每个目标创建可执行文件，每个都包含共享的 main.cpp
foreach(name ${TARGETS})
    add_executable(${name} ${name}.cpp main.cpp)
endforeach()

# 收集所有目标（预留扩展空间）
set(ALL_TARGETS
        ${TARGETS}
)

# 配置链接、头文件路径、测试发现
foreach(name ${ALL_TARGETS})
    target_link_libraries(${name}
        PRIVATE
            Catch2::Catch2              # 测试框架
        PUBLIC
            krkr2core                   # 核心库（utils 模块通过 INTERFACE 传递）
    )
    target_include_directories(${name} PRIVATE "${TEST_CONFIG_DIR}")
    catch_discover_tests(${name})
endforeach()
```

### 第 3 步：编写 main.cpp

```cpp
// tests/unit-tests/core/utils/main.cpp — 自定义测试入口

#include <catch2/catch_session.hpp>  // Catch::Session
#include <spdlog/spdlog.h>           // spdlog 日志库
#include <spdlog/sinks/stdout_color_sinks.h>  // 控制台彩色输出

int main(int argc, char* argv[]) {
    // 创建引擎代码所需的 spdlog logger
    // 如果被测代码调用 spdlog::get("core") 但 logger 不存在，
    // 会返回 nullptr 并在后续调用时崩溃
    auto core_logger = spdlog::stdout_color_mt("core");
    auto tjs2_logger = spdlog::stdout_color_mt("tjs2");

    // 运行 Catch2 测试
    int result = Catch::Session().run(argc, argv);
    return result;
}
```

### 第 4 步：编写测试文件

```cpp
// tests/unit-tests/core/utils/timer.cpp — 计时器测试示例

#include <catch2/catch_test_macros.hpp>  // TEST_CASE, REQUIRE 等宏
#include <thread>                         // std::this_thread::sleep_for
#include <chrono>                         // 时间库

// 假设被测的计时器头文件路径如下
// #include "utils/Timer.h"

TEST_CASE("基本计时功能", "[utils][timer]") {
    // 这里放实际的计时器测试逻辑
    auto start = std::chrono::steady_clock::now();

    // 模拟一个耗时操作
    std::this_thread::sleep_for(std::chrono::milliseconds(50));

    auto end = std::chrono::steady_clock::now();
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start
    ).count();

    // 验证耗时在合理范围内（50ms ± 30ms 容差）
    REQUIRE(elapsed >= 40);   // 至少 40ms
    REQUIRE(elapsed <= 200);  // 不超过 200ms（给 CI 慢环境留余量）
}
```

### 第 5 步：在根 CMakeLists.txt 中注册

编辑 `tests/CMakeLists.txt`，在 core 模块组中添加一行：

```cmake
add_subdirectory(unit-tests/core/movie)
add_subdirectory(unit-tests/core/visual)
add_subdirectory(unit-tests/core/tjs2)
add_subdirectory(unit-tests/core/utils)    # ← 新增这一行

add_subdirectory(unit-tests/plugins)
```

### 第 6 步：构建并运行

```bash
# Windows
cmake --preset="Windows Debug Config"
cmake --build --preset="Windows Debug Build"
ctest --test-dir out/windows/debug --output-on-failure -R timer
# -R timer 表示只运行名字匹配 "timer" 的测试

# Linux
cmake --preset="Linux Debug Config"
cmake --build --preset="Linux Debug Build"
ctest --test-dir out/linux/debug --output-on-failure -R timer

# macOS
cmake --preset="MacOS Debug Config"
cmake --build --preset="MacOS Debug Build"
ctest --test-dir out/macos/debug --output-on-failure -R timer
```

预期输出：

```
Test project /home/dev/krkr2/out/linux/debug
    Start 1: 基本计时功能
1/1 Test #1: 基本计时功能 .....................   Passed    0.06 sec

100% tests passed, 0 tests failed out of 1

Total Test time (real) =   0.06 sec
```

> **跨平台注意事项：** `sleep_for` 的精度在不同操作系统上有差异。Windows 的默认线程调度时间片约 15.6ms，Linux 通常 1-4ms，macOS 约 1ms。因此在断言中使用较宽的容差（如 40-200ms）以避免在特定平台上出现间歇性失败（Flaky Test，即一种时而通过时而失败的不稳定测试）。

---

## 对照项目源码

本节涉及的所有项目源文件清单：

| 文件路径 | 行数 | 作用 |
|----------|------|------|
| `tests/CMakeLists.txt` 第 1-18 行 | 18 | 根测试配置：查找 Catch2、注册子目录、配置夹具路径 |
| `tests/test_config.h.in` 第 1-6 行 | 6 | 模板文件，`configure_file` 将其转换为包含 `TEST_FILES_PATH` 宏的头文件 |
| `tests/unit-tests/plugins/CMakeLists.txt` 第 1-28 行 | 28 | 插件测试构建规则：foreach 模式蓝本，链接 `krkr2plugin + krkr2core` |
| `tests/unit-tests/core/tjs2/CMakeLists.txt` 第 1-29 行 | 29 | TJS2 测试构建规则：两个源文件，链接 `krkr2core` |
| `tests/unit-tests/core/movie/CMakeLists.txt` 第 1-28 行 | 28 | 视频模块测试：一个源文件，链接 `core_movie_module`（最小依赖） |
| `tests/unit-tests/core/visual/CMakeLists.txt` 第 1-27 行 | 27 | 渲染模块测试框架：SOURCES 为空，纯占位 |

### 关键发现

1. **所有子目录 CMakeLists.txt 的模板完全一致**——只需修改三个位置即可复用：`project()` 名称、`SOURCES` 列表、`target_link_libraries` 中的被测库名。
2. **`core/visual/` 是一个有价值的贡献点**——框架已搭好但没有实际测试，适合作为新贡献者的"第一个 PR"目标。
3. **`TEST_CONFIG_DIR` 变量从父目录传递到子目录**——CMake 中父 CMakeLists.txt 设置的变量对 `add_subdirectory` 引入的子目录默认可见（因为变量作用域会继承到子作用域）。

### 常见错误与解决方案

**错误 1：`TEST_CONFIG_DIR` 未定义**

```
CMake Error: target_include_directories called with PRIVATE
  but "${TEST_CONFIG_DIR}" evaluates to empty string
```

**原因：** 子目录的 CMakeLists.txt 被单独运行，而不是通过根 `tests/CMakeLists.txt` 的 `add_subdirectory` 调用。
**解决：** 确保从根测试目录开始配置，而不是直接进入子目录运行 CMake。

**错误 2：`catch_discover_tests` 找不到**

```
CMake Error at CMakeLists.txt:XX (catch_discover_tests):
  Unknown CMake command "catch_discover_tests"
```

**原因：** 根 CMakeLists.txt 缺少 `include(Catch)` 语句。
**解决：** 在根 `tests/CMakeLists.txt` 中确保有 `include(Catch)` 行（位于 `find_package(Catch2 3 REQUIRED)` 之后）。

**错误 3：链接错误——未定义的符号**

```
undefined reference to `spdlog::logger::info(const char*, ...)'
```

**原因：** `target_link_libraries` 中被测库用了 `PRIVATE` 而不是 `PUBLIC`，导致传递依赖（如 spdlog）丢失。
**解决：** 将被测库（如 `krkr2core`）的链接方式从 `PRIVATE` 改为 `PUBLIC`。

---

## 本节小结

- KrKr2 测试目录采用"**按模块分目录 + 夹具集中存放**"的组织方式，源码与测试的路径一一对应
- 根 `tests/CMakeLists.txt`（18 行）负责四件事：查找 Catch2、启用 CTest + Catch 集成、注册子目录、通过 `configure_file` 生成夹具路径头文件
- 子目录 CMakeLists.txt 使用"**一源一可执行文件**"的 foreach 模式——`string(REPLACE)` 去掉 `.cpp` 后缀得到目标名，循环创建可执行文件
- 每个测试可执行文件由"**测试源文件 + 共享 main.cpp**"两个文件组成，因为项目需要自定义 main 来初始化 spdlog logger
- 四个测试模块链接不同的 CMake 目标（`core_movie_module` / `krkr2core` / `krkr2plugin + krkr2core`），遵循"**最小依赖**"原则
- `PRIVATE` 和 `PUBLIC` 链接控制依赖传播——Catch2 用 PRIVATE（不传播），被测库用 PUBLIC（传递其间接依赖）
- `TEST_CONFIG_DIR` 和 `TEST_FILES_PATH` 两个变量通过 CMake 变量作用域继承，从根目录传递到所有子目录
- 新增测试模块只需：创建目录、复制 CMakeLists.txt 模板、修改三处（项目名、源文件、链接库）、在根配置中 `add_subdirectory`

---

## 练习题与答案

### 题目 1：分析 CMake 变量传递

假设你在 `tests/unit-tests/core/tjs2/CMakeLists.txt` 中添加了以下调试打印：

```cmake
message(STATUS "TEST_CONFIG_DIR = ${TEST_CONFIG_DIR}")
message(STATUS "TEST_FILES_PATH = ${TEST_FILES_PATH}")
```

请问这两个 `message` 分别会输出什么？为什么？

<details>
<summary>查看答案</summary>

**`TEST_CONFIG_DIR`** 会输出根测试构建目录的路径，例如 `/home/dev/krkr2/out/linux/debug/tests/`。

原因：`TEST_CONFIG_DIR` 在 `tests/CMakeLists.txt` 第 9 行设置为 `${CMAKE_CURRENT_BINARY_DIR}`，CMake 变量默认在子作用域（`add_subdirectory` 引入的子目录）中可见。

**`TEST_FILES_PATH`** 会输出空字符串或未定义。

原因：`TEST_FILES_PATH` 在 `tests/CMakeLists.txt` 第 17 行设置，但 `add_subdirectory(unit-tests/core/tjs2)` 在第 13 行执行——**比 `set(TEST_FILES_PATH ...)` 早**！CMake 是顺序执行的，在第 13 行处理子目录时，第 17 行的变量还没有被设置。

不过这不影响测试运行，因为 `TEST_FILES_PATH` 是通过 `configure_file` 写入头文件的 C 预处理宏，而不是 CMake 构建时需要的变量。子目录的 CMakeLists.txt 不直接使用 `TEST_FILES_PATH` 变量——它只使用 `TEST_CONFIG_DIR` 来找到生成的头文件。

</details>

### 题目 2：为 `core/sound` 模块编写 CMakeLists.txt

请为音频模块 `core/sound` 编写完整的测试 CMakeLists.txt。要求：
- 包含两个测试源文件：`wave_decoder.cpp` 和 `audio_mixer.cpp`
- 链接正确的 CMake 目标
- 遵循项目现有模式

<details>
<summary>查看答案</summary>

```cmake
# tests/unit-tests/core/sound/CMakeLists.txt — 音频模块测试

cmake_minimum_required(VERSION 3.16)
project(TestSoundModule LANGUAGES CXX)

# 声明测试源文件
set(SOURCES
        wave_decoder.cpp      # 波形解码器测试
        audio_mixer.cpp       # 音频混合器测试
)

# 去掉 .cpp 得到目标名：wave_decoder, audio_mixer
string(REPLACE ".cpp" "" BASENAMES_SOURCES "${SOURCES}")
set(TARGETS ${BASENAMES_SOURCES})

# 每个测试源文件 + main.cpp 编译为独立可执行文件
foreach(name ${TARGETS})
    add_executable(${name} ${name}.cpp main.cpp)
endforeach()

set(ALL_TARGETS
        ${TARGETS}
)

# 配置链接和测试发现
foreach(name ${ALL_TARGETS})
    target_link_libraries(${name}
        PRIVATE
            Catch2::Catch2          # 测试框架（PRIVATE，不传播）
        PUBLIC
            krkr2core               # 核心库（PUBLIC，传递 spdlog 等间接依赖）
    )
    target_include_directories(${name} PRIVATE "${TEST_CONFIG_DIR}")
    catch_discover_tests(${name})
endforeach()
```

**为什么链接 `krkr2core` 而不是 `core_sound_module`？** 音频模块内部依赖 base 模块（流式读取）和 tjs2 模块（TJS 对象绑定），这些传递依赖通过 `krkr2core` INTERFACE 库自动解析。如果只链接 `core_sound_module`，可能缺少间接依赖而导致链接错误。

不要忘记在 `tests/CMakeLists.txt` 中添加 `add_subdirectory(unit-tests/core/sound)`，以及创建对应的 `main.cpp`（初始化 "core" 和 "tjs2" logger）。

</details>

### 题目 3：diagnose 构建错误

你创建了一个新的测试模块，CMakeLists.txt 如下，但编译时报错。请找出所有错误并修复：

```cmake
cmake_minimum_required(VERSION 3.16)
project(TestNewModule LANGUAGES CXX)

set(SOURCES
        my_test.cpp
)

string(REPLACE ".cpp" "" BASENAMES_SOURCES "${SOURCES}")

foreach(name ${BASENAMES_SOURCES})
    add_executable(${name} ${name}.cpp)
endforeach()

foreach(name ${BASENAMES_SOURCES})
    target_link_libraries(${name}
        PRIVATE
            Catch2::Catch2
            krkr2core
    )
    catch_discover_tests(${name})
endforeach()
```

编译错误：
1. `undefined reference to 'main'`
2. `fatal error: 'test_config.h' file not found`
3. 运行时崩溃：`spdlog::get("core")` 返回空指针

<details>
<summary>查看答案</summary>

三个错误及修复：

**错误 1：`add_executable` 缺少 `main.cpp`**

```cmake
# 错误：
add_executable(${name} ${name}.cpp)
# 修复：
add_executable(${name} ${name}.cpp main.cpp)
```

KrKr2 项目使用自定义 `main.cpp`，每个测试可执行文件都必须包含它。没有 `main.cpp` 就没有 `main()` 函数入口，链接器报 `undefined reference to 'main'`。

**错误 2：缺少 `target_include_directories`**

```cmake
# 在 target_link_libraries 之后添加：
target_include_directories(${name} PRIVATE "${TEST_CONFIG_DIR}")
```

没有这行，编译器找不到构建目录中生成的 `test_config.h`，因此报 `file not found`。

**错误 3：`krkr2core` 链接方式应为 PUBLIC，且需要创建 main.cpp 初始化 spdlog**

```cmake
# 错误：
PRIVATE krkr2core
# 修复：
PUBLIC krkr2core
```

同时，必须创建 `main.cpp` 文件并在其中初始化 spdlog logger：

```cpp
#include <catch2/catch_session.hpp>
#include <spdlog/spdlog.h>
#include <spdlog/sinks/stdout_color_sinks.h>

int main(int argc, char* argv[]) {
    auto core_logger = spdlog::stdout_color_mt("core");
    auto tjs2_logger = spdlog::stdout_color_mt("tjs2");
    return Catch::Session().run(argc, argv);
}
```

运行时崩溃的直接原因是 `spdlog::get("core")` 返回空指针。即使链接方式正确，如果 `main.cpp` 中没有创建 "core" logger，引擎代码的日志调用仍会崩溃。

</details>

---

## 下一步

下一节 [自定义 main 与 spdlog 集成](./02-自定义main与spdlog集成.md) 将深入分析每个测试目录中 `main.cpp` 的写法，讲解为什么 KrKr2 不能使用 Catch2 默认入口，以及 spdlog logger 初始化的细节和最佳实践。

