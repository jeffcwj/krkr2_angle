# 本节目标
# 自定义 CMake 模块

> **所属模块：** M02-项目构建系统深度解析
> **前置知识：** [01-根CMakeLists深度解读](./01-根CMakeLists深度解读.md)、[P01/Ch2-CMake语法与命令](../P01-现代CMake与构建工具链/02-CMake语法与命令/)、[P02/Ch3-vcpkg与CMake集成](../P02-vcpkg包管理/03-vcpkg与CMake集成/)
> **预计阅读时间：** 45 分钟（约 9000 字）
> **适用平台：** Windows / Linux / macOS / Android

## 本节目标

读完本节后，你将能够：

1. 理解 CMake 中 `function()` 与 `macro()` 的语法差异和作用域区别
2. 逐行解读 `cmake/CocosBuildHelpers.cmake`（381 行）中全部 14 个函数/宏的实现
3. 逐行解读 `cmake/vcpkg_android.cmake`（99 行）的双 toolchain 叠加机制
4. 理解辅助脚本 `cmake/scripts/sync_folder.py` 在构建流程中的角色
5. 掌握自己编写 CMake 辅助模块的方法，并能扩展现有模块

## 1. CMake 辅助模块的作用

在大型项目中，`CMakeLists.txt` 如果把所有逻辑都写在一起，会迅速膨胀到数千行，难以维护。CMake 提供了**模块化机制**——你可以把常用的函数、宏、查找逻辑抽取到独立的 `.cmake` 文件中，然后通过 `include()` 命令在需要时加载。

KrKr2 项目的 `cmake/` 目录结构如下：

```
krkr2/cmake/
├── CocosBuildHelpers.cmake     # Cocos2d-x 构建辅助模块（381 行，14 个函数/宏）
├── vcpkg_android.cmake          # Android vcpkg 集成模块（99 行）
└── scripts/
    └── sync_folder.py           # 资源同步辅助脚本（104 行）
```

在根 `CMakeLists.txt` 中，这两个模块的加载方式如下：

```cmake
# 文件：krkr2/CMakeLists.txt 第 17-24 行
if(ANDROID)
    set(VCPKG_TARGET_ANDROID ON)
    set(ENV{ANDROID_NDK_HOME} "$ENV{ANDROID_NDK}")
    add_compile_options(-Wno-inconsistent-missing-override)
    include(cmake/vcpkg_android.cmake)  # Android 平台：加载 vcpkg_android 模块
else()
    set(CMAKE_TOOLCHAIN_FILE $ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake)
endif()

# 文件：krkr2/CMakeLists.txt 第 100-101 行
if(NOT ANDROID)
    include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/CocosBuildHelpers.cmake")
    # ... 使用其中的函数
endif()
```

注意两个关键细节：

- **`vcpkg_android.cmake`** 在 `project()` 之前加载（第 21 行），因为它需要设置 `CMAKE_TOOLCHAIN_FILE`，而 toolchain 必须在 `project()` 调用前就绑定
- **`CocosBuildHelpers.cmake`** 在 Android 平台**不加载**（被 `if(NOT ANDROID)` 保护），因为 Android 的资源管理由 Gradle 的 assets 机制处理，不需要 CMake 侧的资源拷贝

## 2. function() 与 macro() —— CMake 的两种封装方式

在深入源码之前，必须先搞清楚 `function()` 和 `macro()` 的区别。`CocosBuildHelpers.cmake` 中两者都大量使用，混淆它们是 CMake 开发中最常见的错误之一。

### 2.1 语法对比

```cmake
# function 定义——有自己的作用域
function(my_function arg1 arg2)
    set(local_var "hello")          # 只在函数内可见
    set(${arg1} "world" PARENT_SCOPE)  # 需要 PARENT_SCOPE 才能修改调用者的变量
    message(STATUS "arg2 = ${arg2}")
endfunction()

# macro 定义——没有自己的作用域，直接在调用者作用域执行
macro(my_macro arg1 arg2)
    set(local_var "hello")          # 直接修改调用者作用域的 local_var！
    set(${arg1} "world")            # 不需要 PARENT_SCOPE，因为就在调用者作用域
    message(STATUS "arg2 = ${arg2}")
endmacro()
```

### 2.2 核心差异表

| 特性 | `function()` | `macro()` |
|------|------------|----------|
| **作用域** | 创建新的作用域 | 在调用者作用域内展开（类似 C 宏） |
| **变量修改** | 需要 `PARENT_SCOPE` 才能影响外部 | 直接修改调用者作用域的变量 |
| **ARGN/ARGC** | 是真正的 CMake 变量 | 是字符串替换，不是真正的变量 |
| **return()** | 退出当前函数 | **退出调用者**（不是退出宏本身！） |
| **调试** | 栈帧清晰，容易追踪 | 展开后的代码可能难以调试 |
| **推荐场景** | 大多数情况 | 需要修改调用者变量、简短逻辑 |

### 2.3 `return()` 陷阱——为什么宏里不能随便用 return

这是一个非常隐蔽的坑，KrKr2 项目中的 `cocos_copy_target_res` 就使用了 `return()`，但它被定义为 `function`，所以没问题。如果改成 `macro`，行为会完全不同：

```cmake
# 安全：function 中的 return() 只退出函数自身
function(safe_check target)
    if(NOT TARGET ${target})
        message(WARNING "${target} 不存在")
        return()  # 退出 safe_check，调用者继续执行
    endif()
    # ...处理逻辑
endfunction()

# 危险：macro 中的 return() 会退出调用者！
macro(dangerous_check target)
    if(NOT TARGET ${target})
        message(WARNING "${target} 不存在")
        return()  # 退出的是调用 dangerous_check 的函数/脚本！
    endif()
    # ...处理逻辑
endmacro()
```

### 2.4 cmake_parse_arguments —— 优雅的参数解析

`CocosBuildHelpers.cmake` 大量使用 `cmake_parse_arguments()` 来解析命名参数。这个命令来自 `CMakeParseArguments` 模块（文件第 1 行 `include(CMakeParseArguments)`）：

```cmake
# cmake_parse_arguments 的三类参数
cmake_parse_arguments(
    <prefix>            # 解析后变量的前缀
    "<options>"         # 布尔开关（无值）
    "<oneValueArgs>"    # 单值参数
    "<multiValueArgs>"  # 多值参数
    ${ARGN}             # 待解析的参数列表
)

# 实例：
function(my_copy_files target)
    set(options VERBOSE)                    # 布尔：VERBOSE
    set(oneValueArgs DESTINATION)           # 单值：DESTINATION <path>
    set(multiValueArgs FILES FOLDERS)       # 多值：FILES a.txt b.txt FOLDERS dir1 dir2
    cmake_parse_arguments(opt "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

    # 解析后可用的变量：
    # opt_VERBOSE      -> TRUE 或 FALSE
    # opt_DESTINATION  -> 路径字符串
    # opt_FILES        -> 文件列表
    # opt_FOLDERS      -> 目录列表

    if(opt_VERBOSE)
        message(STATUS "复制到: ${opt_DESTINATION}")
    endif()
endfunction()

# 调用方式：
my_copy_files(myapp
    VERBOSE
    DESTINATION "${CMAKE_BINARY_DIR}/Resources"
    FILES config.json readme.txt
    FOLDERS assets/ fonts/
)
```

## 3. CocosBuildHelpers.cmake 逐段解读

现在进入正题。`cmake/CocosBuildHelpers.cmake` 是从 Cocos2d-x 引擎继承而来的构建辅助模块，包含 14 个函数/宏，按功能分为五大类：

```
┌─────────────────────────────────────────────────────────┐
│              CocosBuildHelpers.cmake（381 行）             │
├─────────────────┬───────────────────────────────────────┤
│  资源管理（5个）  │ cocos_copy_target_res()              │
│                  │ cocos_def_copy_resource_target()      │
│                  │ cocos_get_resource_path()             │
│                  │ cocos_mark_multi_resources()          │
│                  │ cocos_mark_resources()                │
├─────────────────┼───────────────────────────────────────┤
│  Lua 脚本（1个） │ cocos_copy_lua_scripts()             │
├─────────────────┼───────────────────────────────────────┤
│  依赖管理（4个）  │ search_depend_libs_recursive()       │
│                  │ get_target_depends_ext_dlls()         │
│                  │ cocos_copy_target_dll()               │
│                  │ cocos_use_pkg()                       │
├─────────────────┼───────────────────────────────────────┤
│  IDE 支持（2个）  │ cocos_mark_code_files()              │
│                  │ source_group_single_file()            │
├─────────────────┼───────────────────────────────────────┤
│  应用配置（3个）  │ setup_cocos_app_config()             │
│                  │ cocos_set_default_value()             │
│                  │ cocos_find_package()                  │
├─────────────────┼───────────────────────────────────────┤
│  Xcode 支持（3个）│ cocos_config_app_xcode_property()    │
│                  │ cocos_config_target_xcode_property()  │
│                  │ set_xcode_property()                  │
└─────────────────┴───────────────────────────────────────┘
```

### 3.1 文件头部——模块包含与依赖

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 1-3 行
include(CMakeParseArguments)    # 引入参数解析模块，提供 cmake_parse_arguments() 命令
find_package(Python REQUIRED)   # 查找 Python 解释器，用于运行 sync_folder.py 资源同步脚本
```

`find_package(Python REQUIRED)` 会设置 `Python_EXECUTABLE` 变量，指向系统中找到的 Python 解释器路径。这个变量在后续的 `add_custom_command` 中被引用，用于在构建时执行 Python 脚本。

**常见错误**：如果系统没有安装 Python，`cmake --configure` 阶段就会失败。错误信息如下：

```
CMake Error at cmake/CocosBuildHelpers.cmake:3 (find_package):
  Could not find a package configuration file provided by "Python"
```

**解决方案**：安装 Python 3 并确保 `python3` 在 PATH 中。Windows 上通过 Microsoft Store 或 python.org 安装；Linux 上 `sudo apt install python3`；macOS 上 `brew install python3`。

### 3.2 资源管理函数群

资源管理是 Cocos2d-x 项目最复杂的构建需求之一。游戏资源（图片、音频、UI 定义文件）需要在构建后被拷贝到可执行文件旁边的 `Resources/` 目录，否则运行时会找不到资源。

#### 3.2.1 cocos_def_copy_resource_target()——创建资源同步虚拟目标

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 28-38 行
