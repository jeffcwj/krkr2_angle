# 02-core 模块 CMake 解读

在上一章中，你已经看到了根 `CMakeLists.txt` 是如何将 `krkr2core` 链接到主程序的。本章我们将深入 `cpp/core/CMakeLists.txt`，看看这个核心库是如何组织的。

---

## 1. INTERFACE 库的神奇之处

当你打开 `cpp/core/CMakeLists.txt` 时，你可能会惊讶地发现它几乎没有自己的 `.cpp` 源文件。

```cmake
cmake_minimum_required(VERSION 3.28)
project(krkr2core)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/common)

add_library(${PROJECT_NAME} INTERFACE)
```

**解析：**
- `add_library(${PROJECT_NAME} INTERFACE)`：这是关键。`INTERFACE` 类型的库并不直接生成 `.lib`、`.a` 或 `.so` 文件。它更像是一个“代理人”，负责收集所有的编译选项、宏定义、包含路径和链接库。
- 当主程序链接 `krkr2core` 时，它会自动获得所有被标记为 `INTERFACE` 的属性。这种模式非常适合用于由多个子模块组成的大型核心库。

---

## 2. 9 个子模块的组装

`krkr2core` 的任务就是把分散在各个文件夹下的 9 个核心功能块揉成一团。

```cmake
add_subdirectory(tjs2)
add_subdirectory(base)
add_subdirectory(environ)
add_subdirectory(extension)
add_subdirectory(plugin)
add_subdirectory(movie)
add_subdirectory(sound)
add_subdirectory(visual)
add_subdirectory(utils)

target_link_libraries(${PROJECT_NAME} INTERFACE
        tjs2
        core_base_module
        core_environ_module
        core_extension_module
        core_plugin_module
        core_movie_module
        core_sound_module
        core_visual_module
        core_utils_module
)
```

**子模块说明：**
1. **tjs2**：脚本引擎的核心（词法分析、解析器、虚拟机）。
2. **base**：基础架构，处理 XP3 封包、文件系统、事件循环。
3. **environ**：环境抽象层，对接 Cocos2d-x 以及各个操作系统的窗口、输入、配置管理。
4. **extension**：引擎扩展 API。
5. **plugin**：插件加载和绑定系统（ncbind）。
6. **movie**：视频播放，内含基于 FFmpeg 的完整播放栈。
7. **sound**：音频解码（Vorbis, FFmpeg）与混合引擎。
8. **visual**：渲染引擎，包括图层管理、图像加载器（PNG/JPEG/WEBP/TLG）和转场效果。
9. **utils**：通用工具，如定时器、多线程、剪贴板、随机数生成器。

---

## 3. 链接传播 (Link Properties Propagation)

为什么这里所有的 `target_link_libraries` 都使用 `INTERFACE`？

**原因：**
因为 `krkr2core` 本身就是 `INTERFACE` 类型。如果你在 `INTERFACE` 库上使用 `PUBLIC` 或 `PRIVATE` 关键字，CMake 会报错。所有的属性都必须声明为 `INTERFACE`，这样链接它的主程序才能正确地接收到这些子模块的导出信息。

---

## 4. 核心编译定义 (Compile Definitions)

```cmake
target_include_directories(${PROJECT_NAME} INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}
)

target_compile_definitions(${PROJECT_NAME} INTERFACE
    -DTJS_TEXT_OUT_CRLF
    -D__STDC_CONSTANT_MACROS
    -DUSE_UNICODE_FSTRING
)
```

**术语解析：**
- `TJS_TEXT_OUT_CRLF`：让 TJS2 引擎在输出文本时使用 Windows 风格的换行符（\r\n）。
- `__STDC_CONSTANT_MACROS`：这是使用 FFmpeg 等 C 库时的一个古老约定，允许在 C++ 中使用某些宏（如 `UINT64_C`）。
- `USE_UNICODE_FSTRING`：启用 Unicode 格式化字符串支持，这对处理含有中文字符的游戏路径至关重要。

---

## 5. 平台条件链接 (Conditional Linking)

KrKr2 在核心库这一级就开始处理平台差异了。

```cmake
if(NOT APPLE)
    find_package(OpenMP REQUIRED)
    target_link_libraries(${PROJECT_NAME} INTERFACE OpenMP::OpenMP_CXX)
    target_compile_options(${PROJECT_NAME} INTERFACE ${OpenMP_CXX_FLAGS})
endif ()

if(ANDROID)
    target_link_libraries(${PROJECT_NAME} INTERFACE
        log android EGL GLESv2 GLESv1_CM OpenSLES
    )
endif()
```

**解析：**
1. **OpenMP**：这是一种并行计算框架，在视觉（visual）模块中用于加速转场效果的计算。由于 Apple 系统（macOS/iOS）默认不提供良好的 OpenMP 支持，KrKr2 在 Apple 平台上禁用了它。
2. **Android 系统库**：Android 需要显式链接一系列底层系统库。
   - `log`：用于在 logcat 中打印输出。
   - `android`：访问 Android 的 Native 窗口系统。
   - `EGL/GLESv2`：OpenGL ES 2.0 渲染接口。
   - `OpenSLES`：用于音频播放的原生音频 API。

---

## 6. vcpkg 第三方库集成

KrKr2 强力依赖 Cocos2d-x 作为跨平台渲染框架。

```cmake
find_package(cocos2dx CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} INTERFACE
    cocos2dx::cocos2d
    $<$<BOOL:${ANDROID}>:$<LINK_LIBRARY:WHOLE_ARCHIVE,cocos2dx::cpp_android_spec>>
)
```

**解析：**
1. `find_package(cocos2dx CONFIG REQUIRED)`：通过 vcpkg 查找已安装的 Cocos2d-x 库。`CONFIG` 模式表示我们直接加载库自带的 CMake 配置文件（通常是 `cocos2dxConfig.cmake`）。
2. **WHOLE_ARCHIVE 的重要性**：在 Android 构建中，我们使用了一个复杂的生成器表达式 `$<$<BOOL:${ANDROID}>:$<LINK_LIBRARY:WHOLE_ARCHIVE,cocos2dx::cpp_android_spec>>`。
   - `WHOLE_ARCHIVE`（全量链接）是一个特殊的链接器标志。它告诉链接器：**不要自作聪明地删除“看起来没用到”的代码**。
   - 在 Android 的 JNI 环境中，Cocos2d-x 的某些 C++ 代码会被 Java 端动态反射调用。链接器如果不了解这一点，就会认为这些代码是冗余的并将其删除，导致程序启动时直接崩溃。通过强制全量链接，我们保住了这些代码。

---

## 练习题与答案

### 题目 1：概念分析

在 `cpp/core/CMakeLists.txt` 中，为什么 `krkr2core` 库被声明为 `INTERFACE` 库，而不是普通的 `STATIC`（静态）或 `SHARED`（动态）库？

<details>
<summary>查看答案</summary>

因为 `krkr2core` 充当的是核心模块聚合器的角色。它本身不包含任何源文件，只是通过 `add_subdirectory` 将 9 个具体的子模块组织在一起，并定义了这些模块对外的公共接口（包含路径、宏定义、链接项）。声明为 `INTERFACE` 既能实现这一目的，又不需要在链接阶段生成额外的空二进制库文件。

</details>

### 题目 2：平台差异

为什么在 Android 平台上需要链接 `log` 和 `OpenSLES` 库？其他平台（如 Windows）为什么不需要？

<details>
<summary>查看答案</summary>

- `log` 库提供了 Android 特有的日志系统（logcat）接口。
- `OpenSLES` 是 Android 的高性能底层音频 API。
其他平台如 Windows 通常通过 Windows API 或跨平台库（如 Cocos2d-x 内置的第三方音频库）来处理这些功能，不需要显式链接 Android 的专属底层库。

</details>

### 题目 3：代码实战

如果现在你要为 `krkr2core` 增加一个新的子模块 `core_network_module`，目录名为 `cpp/core/network/`，请写出在 `cpp/core/CMakeLists.txt` 中需要增加的两行代码。

<details>
<summary>查看答案</summary>

```cmake
add_subdirectory(network)
target_link_libraries(${PROJECT_NAME} INTERFACE core_network_module)
```
注意：第二行代码必须加在 `target_link_libraries(${PROJECT_NAME} INTERFACE ...)` 的参数列表中。

</details>
