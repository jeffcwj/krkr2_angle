# 4. vcpkg_android.cmake 逐段解读
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

