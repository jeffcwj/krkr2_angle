# 01-根 CMakeLists 解读

> **所属模块：** P01-现代CMake与构建工具链  
> **章节位置：** 06-实战-分析KrKr2的CMake / 第1节  
> **前置知识：**  
> - [01-CMake是什么/02-CMake核心概念](../../01-CMake是什么/02-CMake核心概念.md)  
> - [02-CMake语法与命令/01-变量与缓存](../../02-CMake语法与命令/01-变量与缓存.md)  
> - [02-CMake语法与命令/02-目标与属性](../../02-CMake语法与命令/02-目标与属性.md)  
> - [02-CMake语法与命令/03-生成器表达式](../../02-CMake语法与命令/03-生成器表达式.md)  
> - [03-多目标与子目录/02-INTERFACE与PUBLIC与PRIVATE](../../03-多目标与子目录/02-INTERFACE与PUBLIC与PRIVATE.md)  
> **预计阅读时间：** 45-60 分钟  
> **对照源码：** `krkr2/CMakeLists.txt`（1-143 行）

## 本节目标

读完本节后，你将能够：

1. 解释根 `CMakeLists.txt` 每一段的用途与顺序。
2. 理解 KrKr2 的平台分支为何这样组织。
3. 掌握 `option()`、`find_package()`、`target_link_libraries()` 在项目里的真实用法。
4. 知道什么时候用显式列表，什么时候谨慎用 `file(GLOB)`。
5. 能完成三类改动：加源文件、加依赖、加编译选项。

---

## 0. 根文件的角色：构建总调度器

根 `CMakeLists.txt` 负责“编排”，不负责业务实现。

它完成四件事：

- 初始化构建环境。
- 平台分发入口目标。
- 组装 core/plugins/external 子工程。
- 处理资源、测试、工具等外围构建。

这就是典型的目标导向（target-based）工程入口。

---

## 1. 逐段解读 `krkr2/CMakeLists.txt`

### 1.1 版本与缓存加速（1-9）

```cmake
cmake_minimum_required(VERSION 3.28)

find_program(CCACHE_PROGRAM ccache)

if (CCACHE_PROGRAM)
    set(CMAKE_C_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
    set(CMAKE_CXX_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
    message(STATUS "Ccache found: ${CCACHE_PROGRAM}")
endif ()
```

解释：

1. `cmake_minimum_required` 固定语义基线。
2. `find_program` 搜索 ccache。
3. 找到后通过 `CMAKE_<LANG>_COMPILER_LAUNCHER` 接管编译调用。

这段是“可选优化”，找不到 ccache 也不会失败。

**知识回顾：** 对应 P01 的变量和 Ninja 增量构建章节。

### 1.2 生成器表达式宏定义（11-15）

```cmake
add_compile_definitions(
        $<$<CONFIG:Debug>:DEBUG>
        $<$<CONFIG:Debug>:_DEBUG>
        $<$<NOT:$<CONFIG:Debug>>:NDEBUG>
)
```

解释：

- Debug 配置下注入 `DEBUG`、`_DEBUG`。
- 非 Debug 配置下注入 `NDEBUG`。

`_DEBUG` 兼容 MSVC 生态。
`NDEBUG` 常用于关闭断言。

**知识回顾：** 对应 P01 的生成器表达式章节。

### 1.3 工具链与 Android 特殊分支（17-24）

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

解释：

- Android 使用专用 vcpkg 适配脚本。
- 非 Android 走标准 `CMAKE_TOOLCHAIN_FILE`。

这里的警告抑制参数用于降低第三方代码告警噪音。

**知识回顾：** 对应 P02 的 vcpkg toolchain 机制。

### 1.4 项目名、选项、语言标准（26-35）

```cmake
set(APP_NAME krkr2)
project(${APP_NAME})

option(ENABLE_TESTS "enable tests execute build(exclude android)" ON)
option(BUILD_TOOLS "build tools execute build(exclude android ios)" ON)

set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
```

#### `option()` 全量说明

本文件仅两个可配置项：

1. `ENABLE_TESTS`：控制是否加入 `tests/`。
2. `BUILD_TOOLS`：控制是否加入 `tools/`。

二者默认 `ON`，但最终还要受平台条件限制。

命令行覆盖示例：

```bash
cmake -S . -B out/build -DENABLE_TESTS=OFF -DBUILD_TOOLS=ON
```

`CMAKE_POSITION_INDEPENDENT_CODE ON` 让共享库链接更稳妥。

### 1.5 路径变量与 MSVC 分支（37-44）

```cmake
set(KRKR2CORE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cpp/core)
set(KRKR2PLUGIN_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cpp/plugins)

if(MSVC)
    add_compile_options("$<$<COMPILE_LANGUAGE:CXX>:/EHsc>" "/MP" "/utf-8")
    add_link_options("/ignore:4099" "/INCREMENTAL" "/DEBUG:FASTLINK")
endif()
```

解释：

- 路径变量减少硬编码。
- MSVC 分支加速本地开发编译与链接。

`/MP` 与 `/INCREMENTAL` 都是典型提速选项。

### 1.6 平台入口目标定义（46-84）

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
elseif(IOS)
    # iOS 当前未启用
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

这是入口分发核心。

#### 平台检测逻辑

当前工程使用 `ANDROID/LINUX/WINDOWS/IOS/MACOS` 变量。
维护时要确保这些变量定义与 CMake 平台状态一致。

#### 目标类型差异

- Android：`SHARED`，供应用层加载。
- Linux/Windows/macOS：`add_executable`。

#### `find_package()` 全量说明

根文件里显式 `find_package()` 只有一个：

- `unofficial-breakpad`（仅 Android）

用途：移动端崩溃采集。

### 1.7 子目录组装与链接（86-97）

```cmake
target_include_directories(${PROJECT_NAME} PUBLIC ${KRKR2CORE_PATH}/environ/cocos2d)

add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/cpp/external)
add_subdirectory(${KRKR2CORE_PATH})
add_subdirectory(${KRKR2PLUGIN_PATH})

target_link_libraries(${PROJECT_NAME} PUBLIC
    krkr2plugin krkr2core
)
```

解释：

- include 路径公开给主目标依赖链。
- 依次接入 external/core/plugins。
- 主目标链接 `krkr2plugin` 与 `krkr2core`。

这里体现了前文讲过的链接传播思想。

### 1.8 资源与目标属性（99-134）

```cmake
if(NOT ANDROID)
    include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/CocosBuildHelpers.cmake")
    set(GAME_RES_FOLDER "${CMAKE_CURRENT_SOURCE_DIR}/ui/cocos-studio")

    if(WINDOWS OR APPLE)
        cocos_mark_multi_resources(common_res_files RES_TO "Resources" FOLDERS ${GAME_RES_FOLDER})
    endif()

    setup_cocos_app_config(${APP_NAME})

    if(APPLE)
        set_target_properties(${APP_NAME} PROPERTIES RESOURCE "${APP_UI_RES}")
        target_sources(${APP_NAME} PRIVATE ${common_res_files})
        if(MACOSX)
            set_target_properties(${APP_NAME} PROPERTIES
                    MACOSX_BUNDLE_INFO_PLIST "${CMAKE_CURRENT_SOURCE_DIR}/apple/macos/Info.plist"
            )
        endif()
    elseif(WINDOWS)
        target_sources(${PROJECT_NAME} PRIVATE ${common_res_files})
        cocos_copy_target_dll(${APP_NAME})
    endif()

    if(LINUX OR WINDOWS)
        cocos_get_resource_path(APP_RES_DIR ${APP_NAME})
        cocos_copy_target_res(${APP_NAME} LINK_TO ${APP_RES_DIR} FOLDERS ${GAME_RES_FOLDER})
    endif()
endif()
```

这段控制运行时资源布局。

目标属性重点：

- `RESOURCE`
- `MACOSX_BUNDLE_INFO_PLIST`

这是目标属性（target properties）在真实项目中的典型使用。

### 1.9 测试与工具开关落地（136-143）

```cmake
if(ENABLE_TESTS AND NOT (IOS OR ANDROID))
    enable_testing()
    add_subdirectory(tests)
endif()

if(BUILD_TOOLS AND NOT (IOS OR ANDROID))
    add_subdirectory(tools)
endif()
```

这两段把“选项 + 平台”合并成最终构建决策。

---

## 2. 条件分支总览（if/elseif/endif）

| 区块 | 条件 | 作用 | 风险 |
|---|---|---|---|
| 工具链 | `if(ANDROID)` | Android 专用 vcpkg 流程 | 环境变量不完整 |
| 编译器 | `if(MSVC)` | MSVC 参数优化 | 误传到 GCC/Clang |
| 平台入口 | `if/elseif` 平台链 | 选择入口与目标类型 | 平台变量错判 |
| 资源 | `if(NOT ANDROID)` | 桌面/Apple 资源处理 | 运行时缺资源 |
| 测试工具 | `if(ENABLE_TESTS...)` | tests/tools 开关 | 选项理解偏差 |

---

## 3. 源文件收集策略：显式列表 vs `file(GLOB)`

根文件入口采用显式列举，这是大型工程常见选择。

显式列表优点：

1. 变更可审计。
2. 平台差异可控。
3. IDE 与构建图一致。

`file(GLOB)` 更适合资源目录批量处理。

示例：

```cmake
target_sources(${PROJECT_NAME} PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/platforms/linux/main.cpp
)

file(GLOB KRKR2_UI_FILES "${CMAKE_CURRENT_SOURCE_DIR}/ui/cocos-studio/*")
```

---

## 4. 目标属性设置分析

根文件中的目标级操作主要有三类：

1. 路径可见性：`target_include_directories`
2. 链接关系：`target_link_libraries`
3. Bundle 属性：`set_target_properties`

这正是 P01 前面“目标与属性”章节的实践版。

---

## 5. 常见修改场景

### 场景 A：新增源文件

```cmake
elseif(WINDOWS)
    add_executable(${PROJECT_NAME}
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/windows/main.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/windows/game.rc
        ${CMAKE_CURRENT_SOURCE_DIR}/platforms/windows/crash_report.cpp
    )
```

### 场景 B：新增依赖

```cmake
if(LINUX OR WINDOWS OR MACOS)
    find_package(zstd CONFIG REQUIRED)
    target_link_libraries(${PROJECT_NAME} PUBLIC zstd::libzstd_shared)
endif()
```

### 场景 C：新增调试编译宏

```cmake
target_compile_definitions(${PROJECT_NAME} PRIVATE
    $<$<CONFIG:Debug>:KRKR2_TRACE_ALLOC>
)
```

---

## 动手实践

目标：关闭测试、保留工具、验证 Debug 宏。

### 步骤 1：配置

```bash
cmake -S . -B out/p01-practice -DENABLE_TESTS=OFF -DBUILD_TOOLS=ON -DCMAKE_BUILD_TYPE=Debug
```

### 步骤 2：添加宏

```cmake
target_compile_definitions(${PROJECT_NAME} PRIVATE
    $<$<CONFIG:Debug>:KRKR2_P01_PRACTICE>
)
```

### 步骤 3：构建

```bash
cmake --build out/p01-practice --config Debug
```

### 步骤 4：验证

```cpp
#ifdef KRKR2_P01_PRACTICE
static const char* kP01PracticeFlag = "enabled";
#endif
```

---

## 本节小结

- 根 `CMakeLists.txt` 是 KrKr2 的构建总入口。
- `option()` 只有两个，但都需要和平台条件一起看。
- 显式 `find_package()` 仅 Android 的 breakpad。
- 平台分支决定了目标类型和入口源。
- 资源处理依赖 Cocos helper，当前未使用标准 `install()`。
- 修改时优先目标级命令，避免全局污染。

---

## 练习题与答案

### 题目 1：选项与平台叠加

在 Android 上设置 `-DENABLE_TESTS=ON -DBUILD_TOOLS=ON`，`tests/` 与 `tools/` 会加入吗？

<details>
<summary>查看答案</summary>

不会。

因为条件写的是：

- `if(ENABLE_TESTS AND NOT (IOS OR ANDROID))`
- `if(BUILD_TOOLS AND NOT (IOS OR ANDROID))`

Android 下第二个条件恒为假。

</details>

### 题目 2：Android 目标类型

为什么 Android 分支使用 `add_library(... SHARED ...)` 而不是 `add_executable()`？

<details>
<summary>查看答案</summary>

Android 原生代码通常作为 `.so` 被应用层加载。
入口生命周期由 Android Framework 管理，不是直接运行独立可执行文件。

</details>

### 题目 3：显式列表与 GLOB

核心模块新增大量源文件时，你推荐哪种方式？请说明原因。

<details>
<summary>查看答案</summary>

推荐显式列表。

原因：

1. 变更可审计。
2. 平台差异可控。
3. 构建图与 IDE 一致。

</details>

### 题目 4：安装规则补充

写出一个最小 `install()` 片段，把可执行文件安装到 `bin`，资源目录安装到 `share/krkr2/ui/cocos-studio`。

<details>
<summary>查看答案</summary>

```cmake
install(TARGETS ${PROJECT_NAME}
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
)

install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/ui/cocos-studio/"
    DESTINATION share/krkr2/ui/cocos-studio
)
```

</details>

---

## 下一步

- [02-core模块CMake解读](./02-core模块CMake解读.md)

下一节会解释 `krkr2core` 这个 `INTERFACE` 聚合目标如何把 9 个核心模块组织起来，并与根文件链接逻辑闭环。
