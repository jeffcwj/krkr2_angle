# 本节目标
# INTERFACE 库与模块链接

> **所属模块：** M02-项目构建系统深度解析
> **前置知识：** [P01-Ch3/02-INTERFACE与PUBLIC与PRIVATE](../P01-现代CMake与构建工具链/03-多目标与子目录/02-INTERFACE与PUBLIC与PRIVATE.md)、[本模块第 01 章](./01-根CMakeLists深度解读.md)
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 INTERFACE 库的本质——它不产生任何二进制文件，只传播编译属性
2. 逐行分析 `krkr2core` INTERFACE 库如何将 9 个子模块聚合为一个统一链接目标
3. 理解 `krkr2plugin` STATIC 库的设计以及与 INTERFACE 库的区别
4. 掌握依赖传播链：krkr2（可执行文件）→ krkr2plugin + krkr2core → 各子模块 → 第三方库
5. 能够为项目添加新的核心模块或插件模块

## INTERFACE 库的本质

在 P01 中我们学过 CMake 的三种库类型：

| 库类型 | 产生二进制 | 编译属性传播 | 典型用途 |
|--------|-----------|-------------|----------|
| STATIC | `.a` / `.lib` | 仅 PUBLIC/INTERFACE 属性 | 静态链接库 |
| SHARED | `.so` / `.dll` | 仅 PUBLIC/INTERFACE 属性 | 动态链接库 |
| **INTERFACE** | **无** | **仅 INTERFACE 属性** | **聚合目标、header-only 库** |

INTERFACE 库的核心特征是**不编译任何源文件**。它存在的意义是作为**属性传播的载体**——把包含目录、编译定义、链接库等信息打包成一个"虚拟目标"，下游只需链接这个目标就能继承所有属性。

KrKr2 的 `krkr2core` 就是这种设计的典型应用。

---

## krkr2core：INTERFACE 聚合库（cpp/core/CMakeLists.txt）

### 完整源码（67 行）

```cmake
# 文件：krkr2/cpp/core/CMakeLists.txt（完整 67 行）
cmake_minimum_required(VERSION 3.28)
project(krkr2core)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/common)

add_library(${PROJECT_NAME} INTERFACE)

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

target_include_directories(${PROJECT_NAME} INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}
)

target_compile_definitions(${PROJECT_NAME} INTERFACE
    -DTJS_TEXT_OUT_CRLF
    -D__STDC_CONSTANT_MACROS
    -DUSE_UNICODE_FSTRING
)

if(NOT APPLE)
    find_package(OpenMP REQUIRED)
    target_link_libraries(${PROJECT_NAME} INTERFACE OpenMP::OpenMP_CXX)
    target_compile_options(${PROJECT_NAME} INTERFACE ${OpenMP_CXX_FLAGS})
endif ()

if(ANDROID)
    target_link_libraries(${PROJECT_NAME} INTERFACE
        log
        android
        EGL
        GLESv2
        GLESv1_CM
        OpenSLES
    )
endif()

# cocos2dx
find_package(cocos2dx CONFIG REQUIRED)

target_link_libraries(${PROJECT_NAME} INTERFACE
    cocos2dx::cocos2d
    $<$<BOOL:${ANDROID}>:$<LINK_LIBRARY:WHOLE_ARCHIVE,cocos2dx::cpp_android_spec>>
)
```

### 逐段深度分析

#### 第 1-6 行：项目声明与 INTERFACE 库创建

```cmake
cmake_minimum_required(VERSION 3.28)
project(krkr2core)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/common)

add_library(${PROJECT_NAME} INTERFACE)
```

**`include_directories()`** 是一个**目录级**命令（非目标级），它将 `common/` 目录添加到当前目录及子目录中所有目标的包含路径。`common/` 目录只有一个文件 `Defer.h`（RAII 清理工具），被多个子模块使用。

> **设计评审：** 现代 CMake 最佳实践推荐使用 `target_include_directories()` 而非 `include_directories()`。但由于 `common/` 被几乎所有子模块使用，目录级命令在这里是合理的简化。

**`add_library(${PROJECT_NAME} INTERFACE)`** — 创建 INTERFACE 库。注意这里**没有指定源文件**，INTERFACE 库也不允许指定源文件（CMake 3.19 之前）。

#### 第 8-16 行：注册 9 个子模块

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
```

每个子目录都有自己的 `CMakeLists.txt`，定义了一个独立的 CMake 目标。子模块与目标的映射关系：

| 子目录 | CMake 目标名 | 库类型 | 主要职责 |
|--------|-------------|--------|----------|
| `tjs2/` | `tjs2` | STATIC | TJS2 脚本引擎（词法/语法分析、字节码、VM） |
| `base/` | `core_base_module` | STATIC | 归档格式（XP3/ZIP/TAR/7z）、流、事件、KAG 解析 |
| `environ/` | `core_environ_module` | STATIC | 平台抽象层、Cocos2d-x 桥接、UI、配置管理 |
| `extension/` | `core_extension_module` | STATIC | 引擎扩展 API |
| `plugin/` | `core_plugin_module` | STATIC | 插件加载/绑定（ncbind 系统） |
| `movie/` | `core_movie_module` | STATIC | 视频播放（FFmpeg 完整播放器栈） |
| `sound/` | `core_sound_module` | STATIC | 音频解码（Vorbis/FFmpeg）、DSP、波形管理 |
| `visual/` | `core_visual_module` | STATIC | 渲染层、位图、图像加载器、过渡效果、OpenGL |
| `utils/` | `core_utils_module` | STATIC | 定时器、线程、剪贴板、手柄输入、RNG |

所有子模块都是 **STATIC 库**——它们会被编译为 `.a`/`.lib` 文件，最终静态链接进主程序。

#### 第 18-28 行：INTERFACE 链接（核心设计）

```cmake
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

这是 **INTERFACE 聚合模式**的核心。`target_link_libraries` 的 `INTERFACE` 关键字意味着：

- `krkr2core` 自身**不链接**这些库（它是 INTERFACE 库，没有二进制可链接）
- 任何**链接 `krkr2core` 的目标**会自动继承这 9 个库的链接

用依赖图表示：

```
krkr2 (可执行文件/共享库)
  │
  ├── target_link_libraries(krkr2 PUBLIC krkr2core)
  │   │
  │   └── krkr2core (INTERFACE 库 — 传播以下所有)
  │       ├── tjs2
  │       ├── core_base_module
  │       ├── core_environ_module
  │       ├── core_extension_module
  │       ├── core_plugin_module
  │       ├── core_movie_module
  │       ├── core_sound_module
  │       ├── core_visual_module
  │       └── core_utils_module
  │
  └── target_link_libraries(krkr2 PUBLIC krkr2plugin)
      └── krkr2plugin (STATIC 库)
```

**为什么不直接让 krkr2 链接 9 个模块？**

如果不用 INTERFACE 聚合，根 CMakeLists.txt 需要：

```cmake
# 不使用 INTERFACE 聚合（冗长且难维护）
target_link_libraries(krkr2 PUBLIC
    tjs2 core_base_module core_environ_module
    core_extension_module core_plugin_module
    core_movie_module core_sound_module
    core_visual_module core_utils_module
    # 还需要手动添加每个模块的编译定义和包含目录...
)
target_compile_definitions(krkr2 PUBLIC
    -DTJS_TEXT_OUT_CRLF -D__STDC_CONSTANT_MACROS -DUSE_UNICODE_FSTRING
)
# 还需要 OpenMP、Android 系统库、cocos2dx...
```

INTERFACE 聚合的优势：
1. **封装性**：核心库的内部模块组成对外部透明
2. **可维护性**：添加/删除子模块只需修改 `core/CMakeLists.txt`
3. **属性传播**：编译定义、包含目录、第三方库依赖自动传递
4. **单一依赖点**：下游只需 `target_link_libraries(xxx PUBLIC krkr2core)` 一行

#### 第 30-38 行：包含目录与编译定义传播

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

**`target_include_directories(... INTERFACE)`** 把 `cpp/core/` 目录本身加入包含路径。注意与第 4 行的 `include_directories()` 区别：

| 命令 | 作用范围 | 传播行为 | 影响目标 |
|------|---------|---------|---------|
| `include_directories(common/)` | 当前目录 + 所有子目录 | 不传播给链接者 | 9 个子模块编译时生效 |
| `target_include_directories(INTERFACE core/)` | 仅链接 krkr2core 的目标 | INTERFACE 传播 | krkr2、krkr2plugin 编译时生效 |

**`target_compile_definitions(... INTERFACE)`** 传播三个预处理宏：

| 宏 | 含义 | 影响 |
|----|------|------|
| `TJS_TEXT_OUT_CRLF` | TJS2 脚本引擎文本输出使用 `\r\n` 换行 | 影响 `tjs2/` 模块的文本序列化行为 |
| `__STDC_CONSTANT_MACROS` | 启用 C99 定宽整数常量宏（`INT64_C` 等） | FFmpeg 头文件需要此宏才能使用 `AV_NOPTS_VALUE` 等常量 |
| `USE_UNICODE_FSTRING` | 使用 Unicode 版本的格式化字符串 | 影响 KiriKiri 的 `TVPFormatString` 系列函数 |

> **为什么使用 INTERFACE 传播编译定义？** 这些宏不仅在 `core/` 子模块内部使用，还在链接 `krkr2core` 的代码中可能被引用。例如，`krkr2plugin` 中的插件代码调用 TJS2 API 时，需要 `TJS_TEXT_OUT_CRLF` 来保证文本格式一致。INTERFACE 传播确保所有使用者自动获得正确的宏定义。

#### 第 40-44 行：OpenMP 条件链接

```cmake
if(NOT APPLE)
    find_package(OpenMP REQUIRED)
    target_link_libraries(${PROJECT_NAME} INTERFACE OpenMP::OpenMP_CXX)
    target_compile_options(${PROJECT_NAME} INTERFACE ${OpenMP_CXX_FLAGS})
endif ()
```

**为什么排除 Apple 平台？** Apple Clang 默认不附带 OpenMP 运行时库（`libomp`）。虽然可以通过 Homebrew 安装 `libomp`，但 KrKr2 选择在 macOS 上禁用 OpenMP 以简化构建流程。

**OpenMP 在项目中的用途：** 主要用于图像处理和音频 DSP 中的并行循环加速。例如 `visual/` 模块的像素混合运算和 `sound/` 模块的波形处理。

**传播链分析：**
1. `find_package(OpenMP REQUIRED)` — 查找 OpenMP，生成导入目标 `OpenMP::OpenMP_CXX`
2. `target_link_libraries(... INTERFACE OpenMP::OpenMP_CXX)` — 链接 OpenMP 库（`-lgomp` 或 `-lomp`）
3. `target_compile_options(... INTERFACE ${OpenMP_CXX_FLAGS})` — 添加编译选项（`-fopenmp`）
4. 任何链接 `krkr2core` 的目标自动获得 OpenMP 编译和链接支持

**跨平台 OpenMP 行为：**

| 平台 | 编译器 | OpenMP 支持 | 编译选项 | 链接库 |
|------|--------|------------|---------|--------|
| Windows | MSVC | ✅ 内置 | `/openmp` | 自动 |
| Linux | GCC | ✅ 内置 | `-fopenmp` | `-lgomp` |
| macOS | Apple Clang | ❌ 已排除 | — | — |
| Android | NDK Clang | ✅ 内置 | `-fopenmp` | `-lomp`（静态） |

#### 第 46-55 行：Android 系统库链接

```cmake
if(ANDROID)
    target_link_libraries(${PROJECT_NAME} INTERFACE
        log
        android
        EGL
        GLESv2
        GLESv1_CM
        OpenSLES
    )
endif()
```

Android NDK 提供了一组预编译的系统库，无需通过 `find_package()` 查找，直接用库名即可链接。每个库的作用：

| 库名 | 头文件 | 用途 | 使用模块 |
|------|--------|------|---------|
| `log` | `<android/log.h>` | Android 日志系统（logcat） | 全局（spdlog android sink） |
| `android` | `<android/native_activity.h>` | Android 原生 API（Asset、窗口管理） | `environ/android/` |
| `EGL` | `<EGL/egl.h>` | OpenGL ES 上下文创建和表面管理 | `visual/` + Cocos2d-x |
| `GLESv2` | `<GLES2/gl2.h>` | OpenGL ES 2.0 渲染 API | `visual/` + Cocos2d-x |
| `GLESv1_CM` | `<GLES/gl.h>` | OpenGL ES 1.x 兼容模式 | Cocos2d-x 向后兼容 |
| `OpenSLES` | `<SLES/OpenSLES.h>` | 音频输出（低延迟） | `sound/` 模块备选后端 |

> **为什么 INTERFACE 传播 Android 系统库？** 这些库在 `core/` 子模块的头文件中被引用。例如，`environ/android/` 模块的公开头文件中包含了 `<android/log.h>`，任何 `#include` 这些头文件的代码都需要链接 `log` 库。INTERFACE 传播确保依赖自动传递。

#### 第 57-63 行：Cocos2d-x 链接与生成器表达式

```cmake
# cocos2dx
find_package(cocos2dx CONFIG REQUIRED)

target_link_libraries(${PROJECT_NAME} INTERFACE
    cocos2dx::cocos2d
    $<$<BOOL:${ANDROID}>:$<LINK_LIBRARY:WHOLE_ARCHIVE,cocos2dx::cpp_android_spec>>
)
```

这段代码包含了本文件中最复杂的 CMake 技巧——**嵌套生成器表达式**。我们逐层拆解：

**第一层：`cocos2dx::cocos2d`**

这是 Cocos2d-x 的主库导入目标，通过 `find_package(cocos2dx CONFIG)` 从 vcpkg 安装的 `cocos2dxConfig.cmake` 中获取。所有平台都链接此目标。

**第二层：`$<$<BOOL:${ANDROID}>:...>`**

这是一个**条件生成器表达式**（conditional generator expression）：

```
$<$<BOOL:${ANDROID}>:value_if_true>
```

- 外层 `$<condition:value>` — 如果条件为真，展开为 `value`；否则展开为空字符串
- 内层 `$<BOOL:${ANDROID}>` — 将 `${ANDROID}` 转为布尔值（非空非零为真）
- 效果：仅在 Android 平台上展开内部内容

**第三层：`$<LINK_LIBRARY:WHOLE_ARCHIVE,cocos2dx::cpp_android_spec>`**

`LINK_LIBRARY` 是 CMake 3.24+ 引入的生成器表达式，用于控制链接方式：

```
$<LINK_LIBRARY:feature,library>
```

`WHOLE_ARCHIVE` 特性告诉链接器将库中的**所有**目标文件都链接进来，而不是仅链接被引用的。对应的链接器选项：

| 链接器 | 选项 |
|--------|------|
| GNU ld / LLD | `-Wl,--whole-archive libxxx.a -Wl,--no-whole-archive` |
| MSVC link | `/WHOLEARCHIVE:libxxx.lib` |

**为什么 Android 需要 WHOLE_ARCHIVE？**

`cocos2dx::cpp_android_spec` 包含 JNI 注册函数和 `__attribute__((constructor))` 初始化器。这些符号不被直接引用，普通链接会将其丢弃，导致运行时 JNI 函数找不到。`WHOLE_ARCHIVE` 强制保留所有符号。

> **常见错误：** 如果去掉 `WHOLE_ARCHIVE`，Android 版本会在启动时崩溃，logcat 报 `java.lang.UnsatisfiedLinkError: No implementation found for...`。这是因为 JNI native 方法注册函数被链接器优化掉了。

---

