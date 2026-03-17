# 02-INTERFACE 与 PUBLIC 与 PRIVATE

> **所属模块：** P01-现代CMake与构建工具链  
> **前置知识：** [01-add_subdirectory.md](./01-add_subdirectory.md)  
> **预计阅读时间：** 30 分钟  
> **适用平台：** Windows / Linux / macOS / Android  
> **建议 CMake 版本：** 3.28+

## 本节目标
读完本节后，你将能够：

1. 准确解释 `PRIVATE`、`PUBLIC`、`INTERFACE` 的定义。
2. 掌握 `target_link_libraries()` 的传播规则与传递性。
3. 掌握 `target_include_directories()` 的传播规则。
4. 掌握 `target_compile_definitions()` 的传播规则。
5. 理解 `INTERFACE` 库在 header-only 与聚合目标中的价值。
6. 结合 KrKr2 实例分析 `krkr2core` 的设计。

## 三种可见性的完整定义
### PRIVATE

- 当前目标自己使用。
- 下游不继承。

### PUBLIC

- 当前目标自己使用。
- 下游也继承。

### INTERFACE

- 当前目标自己不使用。
- 只给下游继承。

## 图书馆类比

- `PRIVATE`：馆员内部手册，读者看不到。
- `PUBLIC`：馆员和读者都要遵守的规则。
- `INTERFACE`：馆员不看内容，只给读者发规则。

## `target_link_libraries()` 的传播规则

| 关键字 | 当前目标参与链接 | 下游继承链接信息 |
| :--- | :---: | :---: |
| PRIVATE | ✅ | ❌ |
| PUBLIC | ✅ | ✅ |
| INTERFACE | ❌ | ✅ |

## 依赖图：A→B→C 的传递性
```mermaid
graph LR
    A[库 A] --> B[库 B]
    B --> C[应用 C]
```

当 B 依赖 A 时：

1. `B PRIVATE A`：A 的接口到 B 截止。
2. `B PUBLIC A`：A 的接口可继续到 C。
3. `B INTERFACE A`：B 不消费 A，但转交给 C。

## 示例 1：链接传播（可运行）

```cmake
cmake_minimum_required(VERSION 3.28)
project(LinkDemo LANGUAGES CXX)

add_library(A STATIC a.cpp)
target_compile_definitions(A PUBLIC A_FLAG=1)

add_library(B STATIC b.cpp)
target_link_libraries(B PUBLIC A)

add_executable(C main.cpp)
target_link_libraries(C PRIVATE B)
```

```cpp
// a.cpp
int fa() { return 10; }
```

```cpp
// b.cpp
int fa();
int fb() { return fa() + 1; }
```

```cpp
// main.cpp
#include <iostream>
int fb();

int main() {
#ifdef A_FLAG
    std::cout << "继承到 A_FLAG" << std::endl;
#else
    std::cout << "未继承到 A_FLAG" << std::endl;
#endif
    std::cout << fb() << std::endl;
    return 0;
}
```

## `target_include_directories()` 的传播规则

- `PRIVATE`：仅当前目标可见。
- `PUBLIC`：当前与下游都可见。
- `INTERFACE`：仅下游可见。

## 示例 2：头文件传播（可运行）

```cmake
cmake_minimum_required(VERSION 3.28)
project(IncludeDemo LANGUAGES CXX)

add_library(mylib STATIC lib/src/lib.cpp)
target_include_directories(mylib
    PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/lib/include
    PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/lib/src
)

add_executable(app app/main.cpp)
target_link_libraries(app PRIVATE mylib)
```

```cpp
// lib/include/lib_api.h
#pragma once
int add(int a, int b);
```

```cpp
// lib/src/lib.cpp
#include "lib_api.h"
int add(int a, int b) { return a + b; }
```

```cpp
// app/main.cpp
#include <iostream>
#include "lib_api.h"

int main() {
    std::cout << add(2, 5) << std::endl;
    return 0;
}
```

## `target_compile_definitions()` 的传播规则
- `PRIVATE`：宏仅当前目标可见。
- `PUBLIC`：宏当前目标与下游都可见。
- `INTERFACE`：宏仅下游可见。

## 示例 3：宏传播（可运行）

```cmake
cmake_minimum_required(VERSION 3.28)
project(DefinitionDemo LANGUAGES CXX)

add_library(flags INTERFACE)
target_compile_definitions(flags INTERFACE USE_FAST_PATH=1)

add_library(core STATIC core.cpp)
target_link_libraries(core PUBLIC flags)

add_executable(game main.cpp)
target_link_libraries(game PRIVATE core)
```

```cpp
// core.cpp
int score() {
#ifdef USE_FAST_PATH
    return 100;
#else
    return 1;
#endif
}
```

```cpp
// main.cpp
#include <iostream>
int score();

int main() {
    std::cout << score() << std::endl;
    return 0;
}
```

## INTERFACE 库的特殊性

`INTERFACE` 库常用于：

1. header-only 库。
2. 聚合多个目标。
3. 统一传播 include、宏、编译选项。

## 示例 4：header-only 库（可运行）

```cmake
cmake_minimum_required(VERSION 3.28)
project(HeaderOnlyDemo LANGUAGES CXX)

add_library(math_headers INTERFACE)
target_include_directories(math_headers INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include)
target_compile_definitions(math_headers INTERFACE MATH_LEVEL=1)

add_executable(demo main.cpp)
target_link_libraries(demo PRIVATE math_headers)
```

```cpp
// include/math_utils.h
#pragma once
inline int square(int x) { return x * x; }
```

```cpp
// main.cpp
#include <iostream>
#include "math_utils.h"

int main() {
    std::cout << square(7) << std::endl;
    return 0;
}
```

## 示例 5：聚合目标（可运行）

```cmake
cmake_minimum_required(VERSION 3.28)
project(AggregatorDemo LANGUAGES CXX)

add_library(physics STATIC physics.cpp)
add_library(graphics STATIC graphics.cpp)

add_library(engine_bundle INTERFACE)
target_link_libraries(engine_bundle INTERFACE physics graphics)
target_compile_definitions(engine_bundle INTERFACE ENGINE_LEVEL=3)

add_executable(game main.cpp)
target_link_libraries(game PRIVATE engine_bundle)
```

```cpp
// physics.cpp
int physics_tick() { return 16; }
```

```cpp
// graphics.cpp
int draw_calls() { return 42; }
```

```cpp
// main.cpp
#include <iostream>
int physics_tick();
int draw_calls();

int main() {
    std::cout << physics_tick() << std::endl;
    std::cout << draw_calls() << std::endl;
    std::cout << ENGINE_LEVEL << std::endl;
    return 0;
}
```

## KrKr2 项目中的 `krkr2core` 设计分析
在 `krkr2/cpp/core/CMakeLists.txt`：

- 第 6 行：`add_library(${PROJECT_NAME} INTERFACE)`。
- 第 18-28 行：`target_link_libraries(${PROJECT_NAME} INTERFACE tjs2 core_base_module core_environ_module core_extension_module core_plugin_module core_movie_module core_sound_module core_visual_module core_utils_module)` 聚合多个模块。
- 第 30-38 行：`target_include_directories` 与 `target_compile_definitions` 使用 `INTERFACE`。
- 第 44-45 行：OpenMP 依赖通过 `INTERFACE` 传播。

在 `krkr2/CMakeLists.txt`：

- 第 86 行：顶层目标设置 `target_include_directories(${PROJECT_NAME} PUBLIC ${KRKR2CORE_PATH}/environ/cocos2d)`。
- 第 95-97 行：顶层目标 `PUBLIC` 链接 `krkr2plugin` 与 `krkr2core`。

设计意义：

1. 统一导出核心依赖，简化平台层目标。
2. 减少重复配置，降低维护成本。
3. 使传播链路清晰可控。

## 常见错误与修复

### 错误 1：一律 PUBLIC

问题：头文件泄露、重编译放大、依赖污染。

修复：默认 `PRIVATE`，仅接口必需项使用 `PUBLIC`。

### 错误 2：有源文件却用 INTERFACE

问题：`.cpp` 不会被正确编译进目标。

修复：有源文件请使用 `STATIC` 或 `SHARED`。

### 错误 3：循环依赖

问题：A 依赖 B，B 又依赖 A，链接与传播复杂。

修复：抽公共接口层并改为单向依赖。

## 动手实践
1. 用“示例 1”创建最小工程并构建。
2. 先保持 `B PUBLIC A` 运行观察输出。
3. 改成 `B PRIVATE A`，重新构建并比较输出。
4. 新增一个 `INTERFACE` 聚合层，验证宏与链接传播。

参考命令：

```bash
cmake -S . -B build -G Ninja
cmake --build build
```

Windows 运行：

```bash
./build/C.exe
```

Linux/macOS 运行：

```bash
./build/C
```

## 本节小结
- `PRIVATE`：自己用，不传播。
- `PUBLIC`：自己用，也传播。
- `INTERFACE`：自己不用，只传播。
- 三种语义在 link/include/definitions 上一致。
- `INTERFACE` 对 header-only 与聚合目标非常关键。
- `krkr2core` 是 KrKr2 中典型的接口聚合层实践。

## 练习题与答案
### 题目 1：场景判断

`NetLib` 内部实现依赖 OpenSSL，公开头文件不依赖。
应使用哪种可见性。

<details>
<summary>查看答案</summary>

应使用 `PRIVATE`：

```cmake
add_library(NetLib STATIC net.cpp)
target_link_libraries(NetLib PRIVATE OpenSSL::SSL)
```

</details>

### 题目 2：传播链推断

若 `C -> B -> A` 且 `B PRIVATE A`。
A 用 `target_compile_definitions(A PUBLIC A_READY=1)`。
C 能否看到 `A_READY`。

<details>
<summary>查看答案</summary>

不能。
`PRIVATE` 会在 B 处截断从 A 到 C 的接口传播。

</details>

### 题目 3：聚合目标写法

请写出 `runtime_bundle`，聚合 `physics`、`graphics`、`audio`。

<details>
<summary>查看答案</summary>

```cmake
add_library(physics STATIC physics.cpp)
add_library(graphics STATIC graphics.cpp)
add_library(audio STATIC audio.cpp)

add_library(runtime_bundle INTERFACE)
target_link_libraries(runtime_bundle INTERFACE physics graphics audio)
```

</details>

## 下一步

继续阅读 [03-实战-构建多模块库.md](./03-实战-构建多模块库.md)。
下一节将把本节规则落地到完整多模块工程。
