# create a virtual target SYNC_RESOURCE-${cocos_target}
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

