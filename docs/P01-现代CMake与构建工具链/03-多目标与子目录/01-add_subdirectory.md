# add_subdirectory：把目录层级映射为构建层级

> **所属模块：** P01-现代CMake与构建工具链
> **前置知识：** [02-多目标工程的基本组织.md](./02-多目标工程的基本组织.md)
> **预计阅读时间：** 28 分钟

## 本节目标

读完本节后，你将能够：
1. 写出 `add_subdirectory(source_dir [binary_dir] [EXCLUDE_FROM_ALL])` 完整语法并解释参数语义。
2. 理解子目录变量作用域，正确使用 `PARENT_SCOPE`。
3. 理清子目录间目标可见性、定义顺序与传播规则。
4. 理解 `include()` 与 `add_subdirectory()` 的边界。
5. 了解 `FetchContent` 在外部项目引入中的位置。
6. 结合 KrKr2 根构建脚本理解真实组织策略。

## add_subdirectory 完整语法与参数

```cmake
add_subdirectory(source_dir [binary_dir] [EXCLUDE_FROM_ALL])
```

### 参数 source_dir

- 指向子目录源码路径。
- 该目录内必须有 `CMakeLists.txt`。

```cmake
cmake_minimum_required(VERSION 3.20)
project(SourceDirDemo)

add_subdirectory(src)
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/tools)
```

### 参数 binary_dir

- 指定子目录构建输出目录。

```cmake
cmake_minimum_required(VERSION 3.20)
project(BinaryDirDemo)

add_subdirectory(lib "${CMAKE_BINARY_DIR}/variants/lib_debug_like")
```

### 参数 EXCLUDE_FROM_ALL

- 子目录目标默认不进入 `all`。

```cmake
cmake_minimum_required(VERSION 3.20)
project(ExcludeFromAllDemo)

add_subdirectory(core)
add_subdirectory(samples EXCLUDE_FROM_ALL)

add_executable(app main.cpp)
target_link_libraries(app PRIVATE core_lib)
```

## 路径变量：CMAKE_SOURCE_DIR 与 CMAKE_CURRENT_SOURCE_DIR

- `CMAKE_SOURCE_DIR`：顶层源码目录。
- `CMAKE_CURRENT_SOURCE_DIR`：当前处理目录。
- `CMAKE_BINARY_DIR`：顶层构建目录。
- `CMAKE_CURRENT_BINARY_DIR`：当前目录构建目录。

## 变量作用域：子目录如何回写父目录

默认行为：
1. 父目录变量对子目录可见。
2. 子目录普通 `set()` 不会改动父目录。
3. 回写父目录要用 `PARENT_SCOPE`。

```cmake
# root/CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(ScopeDemo)

set(ENABLE_LOG ON)
add_subdirectory(module)
message(STATUS "ENABLE_LOG=${ENABLE_LOG}")
```

```cmake
# module/CMakeLists.txt
set(ENABLE_LOG OFF)
set(ENABLE_LOG OFF PARENT_SCOPE)
```

## 子目录间目标可见性规则

### 规则 1：target 名称全局唯一

同一配置图里 target 不可重名。

### 规则 2：先定义后使用

```cmake
# 根目录
add_subdirectory(math)
add_subdirectory(app)
```

```cmake
# app/CMakeLists.txt
add_executable(calc_app main.cpp)
target_link_libraries(calc_app PRIVATE math_lib)
```

### 规则 3：PUBLIC / PRIVATE / INTERFACE 控制传播

- `PRIVATE`：仅当前目标。
- `PUBLIC`：当前目标与下游。
- `INTERFACE`：仅下游。

## include() vs add_subdirectory() 对比

| 维度 | include() | add_subdirectory() |
|---|---|---|
| 本质 | 复用脚本逻辑 | 引入子构建单元 |
| 作用域 | 当前作用域 | 新建目录作用域 |
| 是否要求子目录有 CMakeLists | 否 | 是 |

## 深层嵌套项目结构组织策略

推荐“顶层聚合 + 模块自治”。

```text
engine/
├── CMakeLists.txt
├── cmake/
│   ├── Warnings.cmake
│   └── Sanitizers.cmake
├── modules/
│   ├── runtime/
│   │   └── CMakeLists.txt
│   ├── script/
│   │   └── CMakeLists.txt
│   └── render/
│       └── CMakeLists.txt
└── apps/
    └── player/
        └── CMakeLists.txt
```

顶层脚本建议只保留 `include(...)` 与 `add_subdirectory(...)` 聚合逻辑。

## FetchContent 模块简介

`FetchContent` 在配置阶段拉取外部源码，再并入当前构建图。

```cmake
cmake_minimum_required(VERSION 3.20)
project(FetchDemo)

include(FetchContent)

FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 10.2.1
)

FetchContent_MakeAvailable(fmt)

add_executable(fetch_app main.cpp)
target_link_libraries(fetch_app PRIVATE fmt::fmt)
```

```cpp
#include <fmt/core.h>

int main() {
    fmt::print("FetchContent 成功：{}\n", 42);
    return 0;
}
```

## 多层级项目实例（由浅入深）

### 阶段 A：最小双目录

```text
demo/
├── CMakeLists.txt
├── math/
│   ├── CMakeLists.txt
│   ├── adder.cpp
│   └── adder.h
└── app/
    ├── CMakeLists.txt
    └── main.cpp
```

```cmake
# demo/CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(MultiDirDemo)

add_subdirectory(math)
add_subdirectory(app)
```

```cmake
# demo/math/CMakeLists.txt
add_library(math_lib STATIC adder.cpp)
target_include_directories(math_lib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
```

```cpp
// demo/math/adder.cpp
#include "adder.h"
int add(int a, int b) { return a + b; }
```

```cmake
# demo/app/CMakeLists.txt
add_executable(main_app main.cpp)
target_link_libraries(main_app PRIVATE math_lib)
```

```cpp
// demo/app/main.cpp
#include <iostream>
#include "adder.h"

int main() {
    std::cout << "1 + 2 = " << add(1, 2) << std::endl;
    return 0;
}
```

### 阶段 B：samples 默认不构建

```cmake
add_subdirectory(samples EXCLUDE_FROM_ALL)
```

## KrKr2 项目子目录组织分析

读取 `krkr2/CMakeLists.txt` 可见：

```cmake
set(KRKR2CORE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cpp/core)
set(KRKR2PLUGIN_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cpp/plugins)

add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/cpp/external)
add_subdirectory(${KRKR2CORE_PATH})
add_subdirectory(${KRKR2PLUGIN_PATH})
```

并且测试与工具按选项启用：

```cmake
if(ENABLE_TESTS AND NOT (IOS OR ANDROID))
    enable_testing()
    add_subdirectory(tests)
endif()

if(BUILD_TOOLS AND NOT (IOS OR ANDROID))
    add_subdirectory(tools)
endif()
```

## 动手实践

目标：创建 `app + calc_lib + text_lib + samples`，验证 `EXCLUDE_FROM_ALL` 与 `PARENT_SCOPE`。

```cmake
# practice/CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(AddSubdirectoryPractice)

set(PROJECT_FLAVOR "dev")

add_subdirectory(libs/calc)
add_subdirectory(libs/text)
add_subdirectory(app)
add_subdirectory(samples EXCLUDE_FROM_ALL)

message(STATUS "[root] PROJECT_FLAVOR=${PROJECT_FLAVOR}")
```

在 `libs/text/CMakeLists.txt` 中加入 `set(PROJECT_FLAVOR "dev-with-text" PARENT_SCOPE)`，并观察父目录输出变化。

## 常见错误及解决方案

### 错误 1：子目录缺少 CMakeLists.txt

**现象：** `add_subdirectory(mylib)` 报 `CMake Error: The source directory ... does not contain a CMakeLists.txt file.`。

**原因：** `add_subdirectory()` 要求目标目录下必须有 `CMakeLists.txt`。如果你只是想引入一个 `.cmake` 脚本文件，应该用 `include()`。

**解决：**
```cmake
# 如果 mylib/ 下有 CMakeLists.txt → 用 add_subdirectory
add_subdirectory(mylib)

# 如果只是一个 .cmake 脚本 → 用 include
include(cmake/MyModule.cmake)
```

### 错误 2：目标重名导致配置失败

**现象：** `CMake Error: add_library cannot create target "utils" because another target with the same name already exists.`。

**原因：** 在整个 CMake 配置图中，**目标名称必须全局唯一**。两个子目录各自定义了同名目标 `utils` 就会冲突。

**解决：**
```cmake
# 为目标名加模块前缀
add_library(core_utils STATIC utils.cpp)    # core/ 下
add_library(plugin_utils STATIC utils.cpp)  # plugin/ 下
```

### 错误 3：子目录定义顺序错误导致链接失败

**现象：** `target_link_libraries(app PRIVATE mylib)` 报 `Cannot specify link libraries for target "app" which is not built by this project.` 或找不到 `mylib`。

**原因：** `add_subdirectory()` 按调用顺序处理。如果 `app` 的子目录在 `mylib` 之前被处理，且 `app/CMakeLists.txt` 中引用了 `mylib`，此时 `mylib` 目标尚未定义。

**解决：**
```cmake
# 根 CMakeLists.txt：被依赖的模块先声明
add_subdirectory(libs/mylib)   # 先定义 mylib
add_subdirectory(app)          # 后使用 mylib
```

## 本节小结

- `add_subdirectory` 是多目录 CMake 工程的主干命令。
- 三个参数分别控制源码目录、输出目录、默认构建集合。
- 子目录作用域默认隔离，回写父目录要用 `PARENT_SCOPE`。
- 目标可见性依赖定义顺序与 PUBLIC/PRIVATE/INTERFACE。
- `include()` 用于脚本复用，`add_subdirectory()` 用于模块复用。

## 练习题与答案

### 题目 1：语法题

写出 `add_subdirectory` 完整语法，并解释 `EXCLUDE_FROM_ALL` 的行为。

<details>
<summary>查看答案</summary>

```cmake
add_subdirectory(source_dir [binary_dir] [EXCLUDE_FROM_ALL])
```

`EXCLUDE_FROM_ALL` 使该子目录目标默认不参与 `all`，但若被其他目标显式依赖，仍会被构建。

</details>

### 题目 2：作用域题

父目录：

```cmake
set(USE_SIMD ON)
add_subdirectory(engine)
message(STATUS "USE_SIMD=${USE_SIMD}")
```

子目录：

```cmake
set(USE_SIMD OFF)
```

问：父目录输出什么？如何改为输出 `OFF`？

<details>
<summary>查看答案</summary>

父目录输出 `ON`，因为子目录普通 `set()` 不回写父目录。

改为 `OFF`：

```cmake
set(USE_SIMD OFF PARENT_SCOPE)
```

</details>

### 题目 3：命令选型题

需求 A：复用 `cmake/Warnings.cmake`。  
需求 B：引入带独立 `CMakeLists.txt` 的 `network/` 模块。  
分别使用什么命令？

<details>
<summary>查看答案</summary>

需求 A：

```cmake
include(cmake/Warnings.cmake)
```

需求 B：

```cmake
add_subdirectory(network)
```

</details>

## 下一步

继续阅读 [02-INTERFACE与PUBLIC与PRIVATE.md](./02-INTERFACE与PUBLIC与PRIVATE.md)。
