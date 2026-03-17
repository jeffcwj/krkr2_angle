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
## create a virtual target SYNC_RESOURCE-${cocos_target}
## Update resource files in Resources/ folder everytime when `Run/Debug` target.
function(cocos_def_copy_resource_target cocos_target)
    # 创建一个自定义目标，每次构建时都会执行
    add_custom_target(SYNC_RESOURCE-${cocos_target} ALL
        COMMAND ${CMAKE_COMMAND} -E echo "Copying resources for ${cocos_target} ..."
    )
    # 让主目标依赖资源同步目标，确保资源在主目标之前同步
    add_dependencies(${cocos_target} SYNC_RESOURCE-${cocos_target})
    # 在 IDE 中将此目标放入 "Utils" 文件夹，保持项目结构整洁
    set_target_properties(SYNC_RESOURCE-${cocos_target} PROPERTIES
        FOLDER Utils
    )
endfunction()
```

**关键概念**：`add_custom_target(... ALL)` 中的 `ALL` 关键字意味着这个目标会在默认构建时自动执行（即 `cmake --build` 或 IDE 的 Build All）。没有 `ALL` 的话，只有显式构建这个目标名或者被其他目标依赖时才会执行。

**在 KrKr2 中的调用链**：

```
setup_cocos_app_config(krkr2)           # 第 110 行
  └── cocos_def_copy_resource_target(krkr2)   # Linux 和 Windows 时调用
        └── 创建 SYNC_RESOURCE-krkr2 目标
              └── krkr2 依赖于 SYNC_RESOURCE-krkr2
```

#### 3.2.2 cocos_copy_target_res()——将资源文件夹同步到输出目录

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 5-26 行
# copy resource `FILES` and `FOLDERS` to TARGET_FILE_DIR/Resources
function(cocos_copy_target_res cocos_target)
    set(oneValueArgs LINK_TO)           # 单值参数：目标路径
    set(multiValueArgs FOLDERS)         # 多值参数：源文件夹列表
    cmake_parse_arguments(opt "" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

    # 检查资源同步目标是否存在
    if(NOT TARGET SYNC_RESOURCE-${cocos_target})
        message(WARNING "SyncResource target for ${cocos_target} is not defined")
        return()    # function 中的 return()，安全退出
    endif()

    # 遍历每个源文件夹，添加构建后拷贝命令
    foreach(cc_folder ${opt_FOLDERS})
        get_filename_component(link_folder_abs ${opt_LINK_TO} ABSOLUTE)
        # POST_BUILD：在 SYNC_RESOURCE-${cocos_target} 目标构建完成后执行
        add_custom_command(TARGET SYNC_RESOURCE-${cocos_target} POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E echo "    copying to ${link_folder_abs}"
            # 调用 Python 脚本做增量同步（只拷贝有变化的文件）
            COMMAND ${Python_EXECUTABLE} ARGS
                ${CMAKE_CURRENT_SOURCE_DIR}/cmake/scripts/sync_folder.py
                -s ${cc_folder}           # 源目录
                -d ${link_folder_abs}     # 目标目录
        )
    endforeach()
endfunction()
```

**为什么用 Python 脚本而不是 `cmake -E copy_directory`？** 因为 `copy_directory` 是全量拷贝，每次都会拷贝所有文件。而 `sync_folder.py`（后面会详细分析）实现了**增量同步**——只拷贝修改时间更新的文件，对于含有数百个资源文件的游戏项目，这能节省大量构建时间。

**在根 CMakeLists.txt 中的调用**：

```cmake
# 文件：krkr2/CMakeLists.txt 第 130-133 行
if(LINUX OR WINDOWS)
    cocos_get_resource_path(APP_RES_DIR ${APP_NAME})     # 获取资源输出路径
    cocos_copy_target_res(${APP_NAME}
        LINK_TO ${APP_RES_DIR}                            # 目标路径
        FOLDERS ${GAME_RES_FOLDER}                        # 源文件夹: ui/cocos-studio
    )
endif()
```

#### 3.2.3 cocos_get_resource_path()——计算资源输出路径

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 76-79 行
function(cocos_get_resource_path output cocos_target)
    get_target_property(rt_output ${cocos_target} RUNTIME_OUTPUT_DIRECTORY)
    # 在运行时输出目录下创建 Resources 子目录
    # CMAKE_CFG_INTDIR：多配置生成器（MSVC/Xcode）下为 Debug/Release 等，Ninja/Makefile 下为 "."
    set(${output} "${rt_output}/${CMAKE_CFG_INTDIR}/Resources" PARENT_SCOPE)
endfunction()
```

注意 `PARENT_SCOPE` 的使用——因为 `function()` 有自己的作用域，要让 `output` 变量对调用者可见，必须加 `PARENT_SCOPE`。

#### 3.2.4 cocos_mark_multi_resources()——标记资源文件

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 82-103 行
# mark `FILES` and files in `FOLDERS` as resource files
function(cocos_mark_multi_resources res_out)
    set(oneValueArgs RES_TO)
    set(multiValueArgs FILES FOLDERS)
    cmake_parse_arguments(opt "" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

    set(tmp_file_list)
    # 处理单独指定的文件
    foreach(cc_file ${opt_FILES})
        get_filename_component(cc_file_abs ${cc_file} ABSOLUTE)
        get_filename_component(file_dir ${cc_file_abs} DIRECTORY)
        cocos_mark_resources(FILES ${cc_file_abs} BASEDIR ${file_dir} RESOURCEBASE ${opt_RES_TO})
    endforeach()
    list(APPEND tmp_file_list ${opt_FILES})

    # 处理整个文件夹（递归搜集所有文件）
    foreach(cc_folder ${opt_FOLDERS})
        file(GLOB_RECURSE folder_files "${cc_folder}/*")   # 递归搜集文件夹下所有文件
        list(APPEND tmp_file_list ${folder_files})
        cocos_mark_resources(FILES ${folder_files} BASEDIR ${cc_folder} RESOURCEBASE ${opt_RES_TO})
    endforeach()
    set(${res_out} ${tmp_file_list} PARENT_SCOPE)    # 将文件列表返回给调用者
endfunction()
```

这个函数的作用是将资源文件标记为 macOS Bundle 的资源（通过内部调用 `cocos_mark_resources`），同时返回完整的文件列表。在 KrKr2 中的调用：

```cmake
# 文件：krkr2/CMakeLists.txt 第 106 行
cocos_mark_multi_resources(common_res_files RES_TO "Resources" FOLDERS ${GAME_RES_FOLDER})
```

#### 3.2.5 cocos_mark_resources()——资源文件的底层标记

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 177-202 行
function(cocos_mark_resources)
    set(oneValueArgs BASEDIR RESOURCEBASE)
    set(multiValueArgs FILES)
    cmake_parse_arguments(opt "" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

    if(NOT opt_RESOURCEBASE)
        set(opt_RESOURCEBASE Resources)     # 默认资源基础路径
    endif()

    get_filename_component(BASEDIR_ABS ${opt_BASEDIR} ABSOLUTE)
    foreach(RES_FILE ${opt_FILES} ${opt_UNPARSED_ARGUMENTS})
        get_filename_component(RES_FILE_ABS ${RES_FILE} ABSOLUTE)
        # 计算资源相对路径，保持目录结构
        file(RELATIVE_PATH RES ${BASEDIR_ABS} ${RES_FILE_ABS})
        get_filename_component(RES_LOC ${RES} PATH)
        # 设置 macOS/iOS Bundle 资源位置
        set_source_files_properties(${RES_FILE} PROPERTIES
            MACOSX_PACKAGE_LOCATION "${opt_RESOURCEBASE}/${RES_LOC}"
            HEADER_FILE_ONLY 1      # 告诉编译器不要编译此文件
        )

        # 在 Xcode/Visual Studio 中设置 IDE 分组
        if(XCODE OR VS)
            string(REPLACE "/" "\\" ide_source_group "${opt_RESOURCEBASE}/${RES_LOC}")
            source_group("${ide_source_group}" FILES ${RES_FILE})
        endif()
    endforeach()
endfunction()
```

**关键属性解释**：

- `MACOSX_PACKAGE_LOCATION`：macOS/iOS 专有属性，指定文件在 `.app` Bundle 中的位置。例如设置为 `Resources/fonts`，文件就会被放到 `MyApp.app/Contents/Resources/fonts/`
- `HEADER_FILE_ONLY 1`：告诉 CMake 这个文件不是源代码，不需要交给编译器编译。资源文件（.png, .csb 等）如果被 `target_sources` 添加到目标中，CMake 默认会尝试编译它们
- `source_group()`：仅影响 IDE 的项目树显示，对编译没有任何影响

### 3.3 Lua 脚本处理

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 41-73 行
function(cocos_copy_lua_scripts cocos_target src_dir dst_dir)
    set(luacompile_target COPY_LUA-${cocos_target})
    # 如果目标不存在，先创建
    if(NOT TARGET ${luacompile_target})
        add_custom_target(${luacompile_target} ALL
            COMMAND ${CMAKE_COMMAND} -E echo "Copying lua scripts ..."
        )
        add_dependencies(${cocos_target} ${luacompile_target})
        set_target_properties(${luacompile_target} PROPERTIES FOLDER Utils)
    endif()

    if(MSVC)
        # MSVC 使用多配置生成器，$<CONFIG> 是生成器表达式，构建时解析为 Debug/Release
        add_custom_command(TARGET ${luacompile_target} POST_BUILD
            COMMAND ${Python_EXECUTABLE} ARGS
                ${CMAKE_CURRENT_SOURCE_DIR}/cmake/scripts/sync_folder.py
                -s ${src_dir} -d ${dst_dir}
                -l ${LUAJIT32_COMMAND}     # LuaJIT 32位编译器路径
                -m $<CONFIG>               # 构建配置（Debug 模式跳过编译）
        )
    else()
        if("${CMAKE_BUILD_TYPE}" STREQUAL "")
            # 未指定构建类型，仅拷贝不编译
            add_custom_command(TARGET ${luacompile_target} POST_BUILD
                COMMAND ${Python_EXECUTABLE} ARGS
                    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/scripts/sync_folder.py
                    -s ${src_dir} -d ${dst_dir}
            )
        else()
            # 指定了构建类型，同时处理 32 位和 64 位 LuaJIT 编译
            add_custom_command(TARGET ${luacompile_target} POST_BUILD
                COMMAND ${Python_EXECUTABLE} ARGS
                    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/scripts/sync_folder.py
                    -s ${src_dir} -d ${dst_dir}
                    -l ${LUAJIT32_COMMAND} -m ${CMAKE_BUILD_TYPE}
                COMMAND ${Python_EXECUTABLE} ARGS
                    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/scripts/sync_folder.py
                    -s ${src_dir} -d ${dst_dir}/64bit
                    -l ${LUAJIT64_COMMAND} -m ${CMAKE_BUILD_TYPE}
            )
        endif()
    endif()
endfunction()
```

**注意**：KrKr2 项目目前**不使用 Lua**，因此这个函数在实际构建中不会被调用。它是 Cocos2d-x 引擎原始代码的遗留部分。了解它的实现有助于理解 Cocos2d-x 的构建思路，以及如果将来项目需要 Lua 脚本支持时如何扩展。

### 3.4 依赖管理函数群

#### 3.4.1 search_depend_libs_recursive()——递归搜索依赖库

这是整个文件中最复杂的函数，实现了**BFS（广度优先搜索）遍历目标的完整依赖树**：

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 105-137 行
function(search_depend_libs_recursive cocos_target all_depends_out)
    set(all_depends_inner)                          # 已发现的所有依赖（结果集）
    set(targets_prepare_search ${cocos_target})     # 待搜索队列（BFS 队列）

    while(true)
        foreach(tmp_target ${targets_prepare_search})
            get_target_property(target_type ${tmp_target} TYPE)
            # 只处理库和可执行文件类型的目标
            if(${target_type} STREQUAL "SHARED_LIBRARY" OR
               ${target_type} STREQUAL "STATIC_LIBRARY" OR
               ${target_type} STREQUAL "MODULE_LIBRARY" OR
               ${target_type} STREQUAL "EXECUTABLE")
                # 获取此目标的直接链接依赖
                get_target_property(tmp_depend_libs ${tmp_target} LINK_LIBRARIES)
                # 从搜索队列中移除已处理的目标
                list(REMOVE_ITEM targets_prepare_search ${tmp_target})
                # 如果尚未加入结果集，则加入
                list(FIND all_depends_inner ${tmp_target} index)
                if(index EQUAL -1)
                    list(APPEND all_depends_inner ${tmp_target})
                endif()
                # 将未发现的依赖加入搜索队列
                foreach(depend_lib ${tmp_depend_libs})
                    if(TARGET ${depend_lib})
                        list(FIND all_depends_inner ${depend_lib} dep_index)
                        if(dep_index EQUAL -1)
                            list(APPEND targets_prepare_search ${depend_lib})
                        endif()
                    endif()
                endforeach()
            else()
                # INTERFACE 库或其他类型，从队列移除但不搜索其依赖
                list(REMOVE_ITEM targets_prepare_search ${tmp_target})
            endif()
        endforeach()
        # 队列为空时退出循环
        list(LENGTH targets_prepare_search targets_prepare_search_size)
        if(targets_prepare_search_size LESS 1)
            break()
        endif()
    endwhile(true)

    set(${all_depends_out} ${all_depends_inner} PARENT_SCOPE)
endfunction()
```

**BFS 算法图示**：

```
搜索过程（以 krkr2 为例）：

    队列: [krkr2]  →  发现依赖: krkr2plugin, krkr2core
                  
    队列: [krkr2plugin, krkr2core]
          ├── krkr2plugin (STATIC) → 依赖: psdfile, psbfile, layerExDraw, ...
          └── krkr2core (INTERFACE) → TYPE=INTERFACE_LIBRARY → 跳过搜索
    
    队列: [psdfile, psbfile, layerExDraw, ...]
          ├── psdfile (STATIC) → 无进一步依赖
          ├── psbfile (STATIC) → 无进一步依赖
          └── ...
    
    队列: [] → 退出

    结果: [krkr2, krkr2plugin, psdfile, psbfile, layerExDraw, ...]
```

**重要细节**：`INTERFACE` 类型的库（如 `krkr2core`）被跳过不搜索。这是因为 INTERFACE 库没有自己的编译产物，它的依赖是通过 `target_link_libraries` 的 `INTERFACE` 传播属性自动传递的，CMake 内部已经处理了这种传播。

#### 3.4.2 get_target_depends_ext_dlls()——获取外部 DLL 依赖

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 139-159 行
function(get_target_depends_ext_dlls cocos_target all_depend_dlls_out)
    set(depend_libs)
    set(all_depend_ext_dlls)
    # 先递归获取所有依赖
    search_depend_libs_recursive(${cocos_target} depend_libs)

    foreach(depend_lib ${depend_libs})
        if(TARGET ${depend_lib})
            get_target_property(target_type ${depend_lib} TYPE)
            if(${target_type} STREQUAL "SHARED_LIBRARY" OR
               ${target_type} STREQUAL "STATIC_LIBRARY" OR
               ${target_type} STREQUAL "MODULE_LIBRARY" OR
               ${target_type} STREQUAL "EXECUTABLE")
                # 检查是否有 IMPORTED_IMPLIB——只有导入的共享库才有
                get_target_property(found_shared_lib ${depend_lib} IMPORTED_IMPLIB)
                if(found_shared_lib)
                    # 获取 DLL 的实际位置
                    get_target_property(tmp_dlls ${depend_lib} IMPORTED_LOCATION)
                    list(APPEND all_depend_ext_dlls ${tmp_dlls})
                endif()
            endif()
        endif()
    endforeach()

    set(${all_depend_dlls_out} ${all_depend_ext_dlls} PARENT_SCOPE)
endfunction()
```

**IMPORTED_IMPLIB vs IMPORTED_LOCATION**：在 Windows 上，共享库由两部分组成：
- `.lib`（导入库）→ 对应 `IMPORTED_IMPLIB`，编译时链接用
- `.dll`（动态库）→ 对应 `IMPORTED_LOCATION`，运行时加载用

这个函数找到所有需要在运行时存在的 `.dll` 文件路径。

#### 3.4.3 cocos_copy_target_dll()——拷贝 DLL 到输出目录

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 161-175 行
function(cocos_copy_target_dll cocos_target)
    get_target_depends_ext_dlls(${cocos_target} all_depend_dlls)
    # 去重
    if(all_depend_dlls)
        list(REMOVE_DUPLICATES all_depend_dlls)
    endif()
    foreach(cc_dll_file ${all_depend_dlls})
        get_filename_component(cc_dll_name ${cc_dll_file} NAME)
        # POST_BUILD：构建完成后执行，只拷贝有变化的 DLL
        add_custom_command(TARGET ${cocos_target} POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                ${cc_dll_file}
                "$<TARGET_FILE_DIR:${cocos_target}>/${cc_dll_name}"
        )
    endforeach()
endfunction()
```

**在 KrKr2 中的调用**（仅 Windows）：

```cmake
# 文件：krkr2/CMakeLists.txt 第 127 行
elseif(WINDOWS)
    target_sources(${PROJECT_NAME} PRIVATE ${common_res_files})
    cocos_copy_target_dll(${APP_NAME})    # 拷贝依赖的 DLL 到 exe 目录
endif()
```

`$<TARGET_FILE_DIR:${cocos_target}>` 是一个生成器表达式，在构建时解析为目标可执行文件所在的目录。例如：`out/windows/debug/bin/krkr2/Debug/`。

### 3.5 IDE 支持函数

#### 3.5.1 cocos_mark_code_files() 和 source_group_single_file()

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 204-237 行
# 将目标的源文件按目录结构分组显示在 IDE 中
function(cocos_mark_code_files cocos_target)
    set(oneValueArgs GROUPBASE)
    cmake_parse_arguments(opt "" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})
    if(NOT opt_GROUPBASE)
        set(root_dir ${CMAKE_CURRENT_SOURCE_DIR})
    else()
        set(root_dir ${opt_GROUPBASE})
    endif()

    # 获取目标的所有源文件
    get_property(file_list TARGET ${cocos_target} PROPERTY SOURCES)
    # 逐个文件设置 IDE 分组
    foreach(single_file ${file_list})
        source_group_single_file(${single_file} GROUP_TO "Source Files" BASE_PATH "${root_dir}")
    endforeach()
endfunction()

# 将单个文件按相对路径映射到 IDE 文件夹
function(source_group_single_file single_file)
    set(oneValueArgs GROUP_TO BASE_PATH)
    cmake_parse_arguments(opt "" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})
    get_filename_component(abs_path ${single_file} ABSOLUTE)
    file(RELATIVE_PATH relative_path_with_name ${opt_BASE_PATH} ${abs_path})
    get_filename_component(relative_path ${relative_path_with_name} PATH)
    # 将 "/" 替换为 "\"（Windows IDE 使用反斜杠作为路径分隔符）
    string(REPLACE "/" "\\" ide_file_group "${opt_GROUP_TO}/${relative_path}")
    source_group("${ide_file_group}" FILES ${single_file})
endfunction()
```

这两个函数只在 Xcode 或 Visual Studio 下有效。它们让 IDE 中的文件树反映实际的目录结构，而不是把所有文件平铺显示。

### 3.6 应用配置函数

#### 3.6.1 setup_cocos_app_config()——一站式应用配置

这是在根 `CMakeLists.txt` 中被直接调用的核心函数：

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 239-263 行
function(setup_cocos_app_config app_name)
    # 1. 设置输出路径：bin/${app_name}/
    set_target_properties(${app_name} PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin/${app_name}"
    )

    # 2. 平台特定配置
    if(APPLE)
        # macOS/iOS：设置为 .app Bundle
        set_target_properties(${app_name} PROPERTIES MACOSX_BUNDLE 1)
    elseif(MSVC)
        # Windows：将默认的控制台应用切换为 Windows 应用（无控制台窗口）
        set_property(TARGET ${app_name} APPEND PROPERTY LINK_FLAGS "/SUBSYSTEM:WINDOWS")
    endif()

    # 3. IDE 源文件分组
    if(XCODE OR VS)
        cocos_mark_code_files(${app_name})
    endif()

    # 4. Xcode 特殊属性（iOS Bitcode 等）
    if(XCODE)
        cocos_config_app_xcode_property(${app_name})
    endif()

    # 5. Linux/Windows：创建资源同步目标
    if(LINUX OR WINDOWS)
        cocos_def_copy_resource_target(${app_name})
    endif()
endfunction()
```

**调用效果**（以 Windows 平台为例）：

```
setup_cocos_app_config(krkr2)
  ├── RUNTIME_OUTPUT_DIRECTORY = out/windows/debug/bin/krkr2/
  ├── LINK_FLAGS += /SUBSYSTEM:WINDOWS
  ├── cocos_mark_code_files(krkr2)     → VS 中显示文件树
  └── cocos_def_copy_resource_target(krkr2)  → 创建 SYNC_RESOURCE-krkr2
```

#### 3.6.2 cocos_set_default_value() 和 cocos_find_package()

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 265-309 行

# 如果变量未定义，设置默认值（注意是 macro，直接影响调用者作用域）
macro(cocos_set_default_value cc_variable cc_value)
    if(NOT DEFINED ${cc_variable})
        set(${cc_variable} ${cc_value})
    endif()
endmacro()

# 增强版 find_package：处理变量名不一致的情况
macro(cocos_find_package pkg_name pkg_prefix)
    if(NOT ${pkg_prefix}_FOUND)
        find_package(${pkg_name} ${ARGN})       # 传递额外参数（REQUIRED, COMPONENTS 等）
    endif()
    # 有些包设置 _INCLUDE_DIR（单数），有些设置 _INCLUDE_DIRS（复数），统一处理
    if(NOT ${pkg_prefix}_INCLUDE_DIRS AND ${pkg_prefix}_INCLUDE_DIR)
        set(${pkg_prefix}_INCLUDE_DIRS ${${pkg_prefix}_INCLUDE_DIR})
    endif()
    if(NOT ${pkg_prefix}_LIBRARIES AND ${pkg_prefix}_LIBRARY)
        set(${pkg_prefix}_LIBRARIES ${${pkg_prefix}_LIBRARY})
    endif()
    message(STATUS "${pkg_name} include dirs: ${${pkg_prefix}_INCLUDE_DIRS}")
endmacro()
```

`cocos_find_package` 之所以用 `macro` 而不是 `function`，是因为它需要直接在调用者作用域中设置 `${pkg_prefix}_INCLUDE_DIRS` 等变量。如果用 `function`，这些变量每个都需要加 `PARENT_SCOPE`，非常繁琐。

### 3.7 cocos_use_pkg()——传统风格的包应用函数

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 315-381 行
function(cocos_use_pkg target pkg)
    set(prefix ${pkg})

    # 1. 包含目录
    set(_include_dirs)
    if(NOT _include_dirs)
        set(_include_dirs ${${prefix}_INCLUDE_DIRS})
    endif()
    if(NOT _include_dirs)
        set(_include_dirs ${${prefix}_INCLUDE_DIR})    # 兼容旧风格变量名
    endif()
    if(_include_dirs)
        include_directories(${_include_dirs})           # 全局添加包含目录
    endif()

    # 2. 库搜索目录
    set(_library_dirs)
    if(NOT _library_dirs)
        set(_library_dirs ${${prefix}_LIBRARY_DIRS})
    endif()
    if(_library_dirs)
        link_directories(${_library_dirs})              # 全局添加库目录
    endif()

    # 3. 链接库
    set(_libs)
    if(NOT _libs)
        set(_libs ${${prefix}_LIBRARIES})
    endif()
    if(NOT _libs)
        set(_libs ${${prefix}_LIBRARY})
    endif()
    if(_libs)
        target_link_libraries(${target} ${_libs})
    endif()

    # 4. 预处理定义
    set(_defs)
    if(NOT _defs)
        set(_defs ${${prefix}_DEFINITIONS})
    endif()
    if(_defs)
        add_definitions(${_defs})                       # 全局添加预处理定义
    endif()

    # 5. DLL 收集（仅 MSVC）
    set(_dlls)
    if(NOT _dlls)
        set(_dlls ${${prefix}_DLLS})
    endif()
    if(_dlls)
        if(MSVC)
            get_property(pre_dlls TARGET ${target} PROPERTY CC_DEPEND_DLLS)
            if(pre_dlls)
                set(_dlls ${pre_dlls} ${_dlls})
            endif()
            set_property(TARGET ${target} PROPERTY CC_DEPEND_DLLS ${_dlls})
        endif()
    endif()
endfunction()
```

> **架构反思**：`cocos_use_pkg()` 使用了 `include_directories()`、`link_directories()`、`add_definitions()` 等**全局命令**，这是 CMake 2.x 时代的写法。现代 CMake 推荐使用 `target_include_directories()`、`target_link_libraries()` 等**目标级命令**来避免污染全局作用域。KrKr2 没有调用这个函数——它使用 vcpkg 的现代 `find_package()` + imported targets 方式管理依赖，这是更好的实践。

### 3.8 Xcode 支持函数

```cmake
# 文件：cmake/CocosBuildHelpers.cmake 第 272-294 行

# 遍历应用的所有依赖目标，设置 Xcode 属性
macro(cocos_config_app_xcode_property cocos_app)
    set(depend_libs)
    search_depend_libs_recursive(${cocos_app} depend_libs)
    foreach(depend_lib ${depend_libs})
        if(TARGET ${depend_lib})
            cocos_config_target_xcode_property(${depend_lib})
        endif()
    endforeach()
endmacro()

# 为单个目标设置 iOS 专有的 Xcode 属性
macro(cocos_config_target_xcode_property cocos_target)
    if(IOS)
        set_xcode_property(${cocos_target} ENABLE_BITCODE "NO")     # 关闭 Bitcode
        set_xcode_property(${cocos_target} ONLY_ACTIVE_ARCH "YES")  # 只编译当前架构
    endif()
endmacro()

# 底层函数：设置任意 Xcode 属性
function(set_xcode_property TARGET XCODE_PROPERTY XCODE_VALUE)
    set_property(TARGET ${TARGET}
        PROPERTY XCODE_ATTRIBUTE_${XCODE_PROPERTY} ${XCODE_VALUE}
    )
endfunction(set_xcode_property)
```

这三个函数/宏形成层次结构：`cocos_config_app_xcode_property` → `cocos_config_target_xcode_property` → `set_xcode_property`。当前 KrKr2 的 iOS 支持被注释掉了，所以这些函数实际上不会执行任何操作（`if(IOS)` 始终为假）。

## 4. vcpkg_android.cmake 逐段解读

`cmake/vcpkg_android.cmake` 是 KrKr2 项目中最精巧的自定义模块，只有 99 行，但解决了一个关键问题：**如何同时使用 vcpkg 的 toolchain 和 Android NDK 的 toolchain**。

### 4.1 问题背景

CMake 只允许设置一个 `CMAKE_TOOLCHAIN_FILE`。但 Android 交叉编译需要 NDK 的 toolchain（定义交叉编译器、sysroot 等），vcpkg 也需要自己的 toolchain（重定向 `find_package` 到 vcpkg 安装的包）。如何让两者共存？

vcpkg 提供了 `VCPKG_CHAINLOAD_TOOLCHAIN_FILE` 机制：

```
CMake 启动
  └── 加载 CMAKE_TOOLCHAIN_FILE = vcpkg.cmake
        └── vcpkg.cmake 内部检查 VCPKG_CHAINLOAD_TOOLCHAIN_FILE
              └── 如果设置了，加载 android.toolchain.cmake
                    └── 两个 toolchain 的效果叠加
```

### 4.2 逐段分析

```cmake
# 文件：cmake/vcpkg_android.cmake 第 1-19 行
#
# vcpkg_android.cmake 
#
# Helper script when using vcpkg with cmake. 
# It should be triggered via the variable VCPKG_TARGET_ANDROID
#
# This script will:
# 1 & 2. check the presence of needed env variables: ANDROID_NDK_HOME and VCPKG_ROOT
# 3. set VCPKG_TARGET_TRIPLET according to ANDROID_ABI
# 4. Combine vcpkg and Android toolchains by setting CMAKE_TOOLCHAIN_FILE 
#    and VCPKG_CHAINLOAD_TOOLCHAIN_FILE

# Note: VCPKG_TARGET_ANDROID is not an official vcpkg variable. 
# it is introduced for the need of this script

if (VCPKG_TARGET_ANDROID)
```

整个文件被 `if(VCPKG_TARGET_ANDROID)` 包围。这个变量在根 `CMakeLists.txt` 第 18 行设置：

```cmake
if(ANDROID)
    set(VCPKG_TARGET_ANDROID ON)    # 触发 vcpkg_android.cmake 的逻辑
```

#### 4.2.1 环境变量检查

```cmake
# 文件：cmake/vcpkg_android.cmake 第 22-44 行

# 检查 1：ANDROID_NDK_HOME 环境变量
if (NOT DEFINED ENV{ANDROID_NDK_HOME})
    message(FATAL_ERROR "
    Please set an environment variable ANDROID_NDK_HOME
    For example:
    export ANDROID_NDK_HOME=/home/your-account/Android/Sdk/ndk-bundle
    Or:
    export ANDROID_NDK_HOME=/home/your-account/Android/android-ndk-r21b
    ")
endif()

# 检查 2：VCPKG_ROOT 环境变量
if (NOT DEFINED ENV{VCPKG_ROOT})
    message(FATAL_ERROR "
    Please set an environment variable VCPKG_ROOT
    For example:
    export VCPKG_ROOT=/path/to/vcpkg
    ")
endif()
```

`message(FATAL_ERROR ...)` 会**立即终止** CMake 配置过程，并输出错误信息。这比让用户遇到后续难以理解的链接错误要好得多——**尽早失败，给出清晰的修复指引**。

#### 4.2.2 ABI 到 Triplet 的映射

```cmake
# 文件：cmake/vcpkg_android.cmake 第 47-79 行

# Android ABI → vcpkg triplet 映射表
#
# |VCPKG_TARGET_TRIPLET       | ANDROID_ABI          |
# |---------------------------|----------------------|
# |arm64-android              | arm64-v8a            |
# |arm-android                | armeabi-v7a          |
# |x64-android                | x86_64               |
# |x86-android                | x86                  |

if (ANDROID_ABI MATCHES "arm64-v8a")
    set(VCPKG_TARGET_TRIPLET "arm64-android" CACHE STRING "" FORCE)
elseif(ANDROID_ABI MATCHES "armeabi-v7a")
    set(VCPKG_TARGET_TRIPLET "arm-android" CACHE STRING "" FORCE)
elseif(ANDROID_ABI MATCHES "x86_64")
    set(VCPKG_TARGET_TRIPLET "x64-android" CACHE STRING "" FORCE)
elseif(ANDROID_ABI MATCHES "x86")
    set(VCPKG_TARGET_TRIPLET "x86-android" CACHE STRING "" FORCE)
else()
    message(FATAL_ERROR "
    Please specify ANDROID_ABI
    For example
    cmake ... -DANDROID_ABI=armeabi-v7a

    Possible ABIs are: arm64-v8a, armeabi-v7a, x64-android, x86-android
    ")
endif()
message("vcpkg_android.cmake: VCPKG_TARGET_TRIPLET was set to ${VCPKG_TARGET_TRIPLET}")
```

**关键细节**：`CACHE STRING "" FORCE` 的含义：

- `CACHE`：将变量写入 CMake 缓存（`CMakeCache.txt`），持久化
- `STRING ""`：类型为字符串，描述为空
- `FORCE`：即使缓存中已有此变量，也强制覆盖。这是必要的，因为 Gradle 可能会多次调用 CMake（为不同的 ABI 各调用一次），每次 `ANDROID_ABI` 不同

**KrKr2 实际构建的 ABI**：

```gradle
// 文件：platforms/android/app/build.gradle 第 22 行
ndk {
    abiFilters 'arm64-v8a', 'x86_64'    // 只编译 64 位架构
}
```

因此实际触发的映射只有两个：`arm64-v8a → arm64-android` 和 `x86_64 → x64-android`。

#### 4.2.3 双 Toolchain 叠加

```cmake
# 文件：cmake/vcpkg_android.cmake 第 82-99 行

# vcpkg_toolchain = $VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake
# android_toolchain = $ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake
#
# 策略：vcpkg toolchain 为主，通过 CHAINLOAD 机制加载 Android toolchain

# Android toolchain 作为"被链载"的 toolchain
set(VCPKG_CHAINLOAD_TOOLCHAIN_FILE
    $ENV{ANDROID_NDK_HOME}/build/cmake/android.toolchain.cmake)

# vcpkg toolchain 作为主 toolchain
set(CMAKE_TOOLCHAIN_FILE
    $ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake)

message("vcpkg_android.cmake: CMAKE_TOOLCHAIN_FILE was set to ${CMAKE_TOOLCHAIN_FILE}")
message("vcpkg_android.cmake: VCPKG_CHAINLOAD_TOOLCHAIN_FILE was set to ${VCPKG_CHAINLOAD_TOOLCHAIN_FILE}")

endif(VCPKG_TARGET_ANDROID)
```

**叠加后的完整效果**：

```
┌───────────────────────────────────────────────────────────────┐
│                     CMake 配置阶段                              │
├───────────────────────────────────────────────────────────────┤
│  1. CMake 加载 CMAKE_TOOLCHAIN_FILE = vcpkg.cmake             │
│     ├── 设置 CMAKE_PREFIX_PATH 指向 vcpkg 安装目录             │
│     ├── 重定向 find_package() 到 vcpkg 的包                    │
│     └── 检测到 VCPKG_CHAINLOAD_TOOLCHAIN_FILE                  │
│         └── 2. 加载 android.toolchain.cmake                    │
│             ├── 设置 CMAKE_C_COMPILER = NDK 的 clang           │
│             ├── 设置 CMAKE_CXX_COMPILER = NDK 的 clang++       │
│             ├── 设置 CMAKE_SYSROOT = NDK 的 sysroot            │
│             ├── 设置 ANDROID_ABI, ANDROID_PLATFORM 等变量       │
│             └── 定义交叉编译的系统名和处理器                      │
├───────────────────────────────────────────────────────────────┤
│  最终效果：                                                     │
│  - 编译器和 sysroot：来自 Android NDK                           │
│  - find_package 路径：来自 vcpkg（指向对应 triplet 的安装目录）    │
│  - 两者完美共存，互不干扰                                        │
└───────────────────────────────────────────────────────────────┘
```

## 5. sync_folder.py 辅助脚本分析

`cmake/scripts/sync_folder.py` 是被 `CocosBuildHelpers.cmake` 中多个函数调用的 Python 脚本，实现增量资源同步。完整 104 行，核心逻辑如下：

```python
# 文件：cmake/scripts/sync_folder.py（简化解读）

def copy_if_newer(src, dst):
    """只在源文件比目标文件新时才拷贝"""
    if not os.path.exists(dst):
        shutil.copy(src, dst)       # 目标不存在，直接拷贝
    else:
        stat_src = os.stat(src)
        stat_dst = os.stat(dst)
        if stat_src.st_mtime > stat_dst.st_mtime:
            shutil.copy(src, dst)   # 源更新，覆盖拷贝

def sync_folder(src_dir, dst_dir, luajit, compile):
    """递归同步文件夹"""
    if os.path.isfile(src_dir):
        # 单文件：如果是 Lua 文件且需要编译，走编译路径
        if luajit and need_compile:
            compile_if_newer(src_dir, dst_dir, luajit)
        else:
            copy_if_newer(src_dir, dst_dir)
    elif os.path.isdir(src_dir):
        # 目录：递归处理每个子项
        for name in os.listdir(src_dir):
            sync_folder(os.path.join(src_dir, name),
                        os.path.join(dst_dir, name), luajit, need_compile)
```

**命令行参数**：
- `-s`：源目录
- `-d`：目标目录
- `-l`：LuaJIT 编译器路径（可选，KrKr2 不使用）
- `-m`：构建模式（Debug 模式跳过 Lua 编译）

**性能优势**：对于 KrKr2 的 `ui/cocos-studio` 目录（包含 `.csb` UI 定义文件和多语言 `.xml` 文件），增量同步只需要比较时间戳，而不是每次都全量拷贝，在日常开发中能节省数秒的构建时间。

## 6. 动手实践

### 练习 1：编写一个简单的 CMake 辅助模块

创建文件 `cmake/MyHelpers.cmake`：

```cmake
# cmake/MyHelpers.cmake

# 打印目标的所有属性（调试用）
function(print_target_info target_name)
    if(NOT TARGET ${target_name})
        message(WARNING "print_target_info: ${target_name} 不是一个有效目标")
        return()
    endif()

    get_target_property(target_type ${target_name} TYPE)
    get_target_property(target_sources ${target_name} SOURCES)
    get_target_property(target_libs ${target_name} LINK_LIBRARIES)

    message(STATUS "===== 目标信息: ${target_name} =====")
    message(STATUS "  类型: ${target_type}")
    message(STATUS "  源文件: ${target_sources}")
    message(STATUS "  链接库: ${target_libs}")
    message(STATUS "========================================")
endfunction()

# 批量设置编译警告选项
function(set_strict_warnings target_name)
    if(MSVC)
        target_compile_options(${target_name} PRIVATE /W4 /WX)
    else()
        target_compile_options(${target_name} PRIVATE
            -Wall -Wextra -Wpedantic -Werror
        )
    endif()
endfunction()
```

在 `CMakeLists.txt` 中使用：

```cmake
include(cmake/MyHelpers.cmake)

add_executable(myapp main.cpp)
set_strict_warnings(myapp)
print_target_info(myapp)
```

### 练习 2：扩展 vcpkg_android.cmake 添加 RISC-V 支持

假设将来 Android 支持 RISC-V 架构，你可以这样扩展映射表：

```cmake
# 在 vcpkg_android.cmake 的 ABI 映射区域添加：
elseif(ANDROID_ABI MATCHES "riscv64")
    set(VCPKG_TARGET_TRIPLET "riscv64-android" CACHE STRING "" FORCE)
```

同时需要创建对应的 vcpkg triplet 文件 `vcpkg/triplets/riscv64-android.cmake`：

```cmake
set(VCPKG_TARGET_ARCHITECTURE riscv64)
set(VCPKG_CRT_LINKAGE dynamic)
set(VCPKG_LIBRARY_LINKAGE static)
set(VCPKG_CMAKE_SYSTEM_NAME Android)
```

## 7. 对照项目源码

### 调用关系全景图

```
krkr2/CMakeLists.txt
│
├── 第 21 行: include(cmake/vcpkg_android.cmake)        ← Android 专用
│   ├── 检查 ANDROID_NDK_HOME
│   ├── 检查 VCPKG_ROOT
│   ├── ANDROID_ABI → VCPKG_TARGET_TRIPLET 映射
│   └── 设置 CMAKE_TOOLCHAIN_FILE + VCPKG_CHAINLOAD_TOOLCHAIN_FILE
│
└── 第 101 行: include("cmake/CocosBuildHelpers.cmake")  ← 非 Android
    │
    ├── 第 106 行: cocos_mark_multi_resources()           ← Windows/Apple
    │   └── cocos_mark_resources()                        ← 标记 Bundle 资源
    │
    ├── 第 110 行: setup_cocos_app_config(krkr2)
    │   ├── RUNTIME_OUTPUT_DIRECTORY
    │   ├── MACOSX_BUNDLE (Apple)
    │   ├── /SUBSYSTEM:WINDOWS (MSVC)
    │   ├── cocos_mark_code_files() (IDE)
    │   ├── cocos_config_app_xcode_property() (Xcode)
    │   └── cocos_def_copy_resource_target() (Linux/Windows)
    │
    ├── 第 127 行: cocos_copy_target_dll(krkr2)           ← Windows 专用
    │   ├── get_target_depends_ext_dlls()
    │   │   └── search_depend_libs_recursive()            ← BFS 遍历依赖树
    │   └── copy_if_different 每个 DLL
    │
    └── 第 132 行: cocos_copy_target_res(krkr2)           ← Linux/Windows
        └── sync_folder.py 增量同步资源
```

### 关键源码文件索引

| 文件 | 行数 | 本节涉及内容 |
|------|------|-------------|
| `cmake/CocosBuildHelpers.cmake` | 381 | 全文逐段分析 |
| `cmake/vcpkg_android.cmake` | 99 | 全文逐段分析 |
| `cmake/scripts/sync_folder.py` | 104 | 核心逻辑分析 |
| `CMakeLists.txt` 第 17-24 行 | — | vcpkg_android 的 include 调用 |
| `CMakeLists.txt` 第 99-133 行 | — | CocosBuildHelpers 的 include 和函数调用 |

## 8. 本节小结

- CMake 的 `function()` 有独立作用域，`macro()` 在调用者作用域展开，选择时首选 `function`，只在需要直接修改调用者变量时用 `macro`
- `cmake_parse_arguments()` 是编写高质量 CMake 函数的标配，支持布尔开关、单值和多值三种参数类型
- `CocosBuildHelpers.cmake` 包含 14 个函数/宏，核心功能是**资源管理**（构建后同步资源文件到输出目录）和 **DLL 管理**（Windows 上自动拷贝依赖的 DLL）
- `search_depend_libs_recursive()` 使用 BFS 算法遍历完整的 CMake 依赖树，是 DLL 拷贝和 Xcode 属性设置的基础
- `vcpkg_android.cmake` 通过 `VCPKG_CHAINLOAD_TOOLCHAIN_FILE` 实现了 vcpkg + Android NDK 的双 toolchain 叠加
- Android ABI（`arm64-v8a` 等）到 vcpkg triplet（`arm64-android` 等）的映射是自动完成的
- `sync_folder.py` 实现了基于文件时间戳的增量同步，避免全量拷贝
- 一些函数（如 `cocos_use_pkg`、`cocos_copy_lua_scripts`）是 Cocos2d-x 的遗留代码，KrKr2 实际上并不使用

## 9. 练习题与答案

### 题目 1：function() 与 macro() 的作用域区别

以下 CMake 代码的输出是什么？请逐行分析。

```cmake
set(MY_VAR "original")

function(func_test var_name)
    set(${var_name} "from_function")
    set(LOCAL_A "function_local")
endfunction()

macro(macro_test var_name)
    set(${var_name} "from_macro")
    set(LOCAL_B "macro_local")
endmacro()

func_test(MY_VAR)
message("After func_test: MY_VAR=${MY_VAR}, LOCAL_A=${LOCAL_A}")

macro_test(MY_VAR)
message("After macro_test: MY_VAR=${MY_VAR}, LOCAL_B=${LOCAL_B}")
```

<details>
<summary>查看答案</summary>

输出结果：

```
After func_test: MY_VAR=original, LOCAL_A=
After macro_test: MY_VAR=from_macro, LOCAL_B=macro_local
```

**分析**：

1. `func_test(MY_VAR)` 在函数内部执行 `set(MY_VAR "from_function")`，但因为 `function` 有自己的作用域，这只修改了函数局部的 `MY_VAR`，调用者的 `MY_VAR` 不受影响。`LOCAL_A` 同理，只存在于函数内部。要修改调用者的变量，必须使用 `set(${var_name} "from_function" PARENT_SCOPE)`。

2. `macro_test(MY_VAR)` 在宏内部执行 `set(MY_VAR "from_macro")`，因为 `macro` 没有自己的作用域，直接在调用者作用域执行，所以 `MY_VAR` 被成功修改。`LOCAL_B` 也直接出现在调用者作用域中。

</details>

### 题目 2：分析 DLL 拷贝链路

在 Windows 上构建 KrKr2 时，`cocos_copy_target_dll(krkr2)` 会触发什么操作？请描述完整的函数调用链和最终效果。

<details>
<summary>查看答案</summary>

完整调用链：

```
cocos_copy_target_dll(krkr2)
│
├── 1. 调用 get_target_depends_ext_dlls(krkr2, all_depend_dlls)
│   │
│   ├── 1.1 调用 search_depend_libs_recursive(krkr2, depend_libs)
│   │   ├── BFS 从 krkr2 开始遍历
│   │   ├── 发现 krkr2plugin (STATIC), krkr2core (INTERFACE, 跳过)
│   │   ├── 继续遍历 krkr2plugin 的依赖...
│   │   └── 返回完整依赖列表
│   │
│   └── 1.2 遍历依赖列表，检查每个目标的 IMPORTED_IMPLIB
│       ├── 找到有 IMPORTED_IMPLIB 的目标（vcpkg 安装的共享库）
│       └── 获取其 IMPORTED_LOCATION（.dll 路径）
│
├── 2. list(REMOVE_DUPLICATES all_depend_dlls)  去重
│
└── 3. 对每个 DLL 添加 POST_BUILD 命令：
    └── cmake -E copy_if_different <dll路径> <krkr2.exe所在目录>/
```

最终效果：构建完成后，所有 vcpkg 提供的动态库（如 SDL2.dll、avcodec-*.dll 等）会被自动拷贝到 krkr2.exe 所在目录，确保运行时能找到这些 DLL。

注意：KrKr2 的 vcpkg.json 中大部分依赖使用静态链接（vcpkg 默认 triplet `x64-windows` 是动态链接，但 `x64-windows-static` 是静态链接），实际需要拷贝的 DLL 数量取决于使用的 triplet。

</details>

### 题目 3：编写自定义 CMake 函数

编写一个 CMake 函数 `copy_runtime_assets(target)`，实现以下功能：

1. 在目标构建完成后，将 `assets/` 目录下的所有文件拷贝到目标可执行文件旁边的 `data/` 目录
2. 使用增量拷贝（`copy_if_different`）
3. 支持 Windows、Linux、macOS 三个平台
4. 在 IDE 中将辅助目标放入 "Assets" 文件夹

<details>
<summary>查看答案</summary>

```cmake
function(copy_runtime_assets target)
    # 创建辅助目标
    set(sync_target SYNC_ASSETS-${target})
    if(NOT TARGET ${sync_target})
        add_custom_target(${sync_target} ALL
            COMMENT "Syncing runtime assets for ${target}..."
        )
        add_dependencies(${target} ${sync_target})
        set_target_properties(${sync_target} PROPERTIES FOLDER "Assets")
    endif()

    # 搜集 assets/ 下的所有文件
    set(assets_dir "${CMAKE_CURRENT_SOURCE_DIR}/assets")
    file(GLOB_RECURSE asset_files RELATIVE "${assets_dir}" "${assets_dir}/*")

    # 为每个文件添加拷贝命令
    foreach(asset_file ${asset_files})
        # 获取文件的目录部分，用于创建目标目录结构
        get_filename_component(asset_subdir "${asset_file}" DIRECTORY)

        add_custom_command(TARGET ${sync_target} POST_BUILD
            # 确保目标子目录存在
            COMMAND ${CMAKE_COMMAND} -E make_directory
                "$<TARGET_FILE_DIR:${target}>/data/${asset_subdir}"
            # 增量拷贝
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                "${assets_dir}/${asset_file}"
                "$<TARGET_FILE_DIR:${target}>/data/${asset_file}"
        )
    endforeach()

    message(STATUS "Configured asset sync for ${target}: ${assets_dir} -> data/")
endfunction()
```

使用方式：

```cmake
add_executable(myapp main.cpp)
copy_runtime_assets(myapp)
```

**注意事项**：

- `file(GLOB_RECURSE ...)` 的 `RELATIVE` 参数让返回的路径是相对于 `assets_dir` 的，这样可以在目标目录中重建相同的目录结构
- `$<TARGET_FILE_DIR:${target}>` 在所有平台上都能正确解析到可执行文件所在目录
- `make_directory` 是幂等的（目录已存在时不报错），所以每次构建都安全调用
- 这个方案的缺点是 `GLOB_RECURSE` 只在 CMake 配置时执行一次，新增的资源文件需要重新运行 `cmake --configure` 才能被发现。可以考虑改用 Python 脚本（类似 sync_folder.py）来避免这个问题

</details>

## 10. 下一步

至此，M02 模块的全部 5 章已经完成。你已经完整理解了 KrKr2 项目的构建系统——从根 CMakeLists.txt 到 INTERFACE 库设计，从平台条件编译到 Android Gradle 集成，再到自定义 CMake 辅助模块。

建议接下来学习：

- [M03-TJS2 脚本引擎](../M03-TJS2脚本引擎/)：深入 KrKr2 最核心的组件——TJS2 脚本引擎的词法分析、语法解析和虚拟机实现
- [P03-Git 版本控制](../P03-Git版本控制/)：如果你对 Git 还不熟悉，建议先学习版本控制基础
- [P04-现代C++17](../P04-现代C++17/)：KrKr2 大量使用 C++17 特性，这个前置模块将帮助你理解项目代码中的现代 C++ 用法
