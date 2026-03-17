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

## krkr2plugin：STATIC 聚合库（cpp/plugins/CMakeLists.txt）

### 完整源码（74 行）

```cmake
# 文件：krkr2/cpp/plugins/CMakeLists.txt（完整 74 行）
cmake_minimum_required(VERSION 3.28)
project(krkr2plugin)

set(PLUGIN_ROOT_SRCS
    ExtObj.cpp
    KAGParserEx.cpp
    LayerBitmapFont.cpp
    LayerExBase.cpp
    LongExposure.cpp
    Menu.cpp
    csvParser.cpp
    dirlist.cpp
    fftgraph.cpp
    saveStruct.cpp
    scriptsEx.cpp
    textrender.cpp
    varfile.cpp
    windowEx.cpp
    wutcwf.cpp
    xp3filter.cpp
)

add_library(krkr2plugin STATIC
    ${PLUGIN_ROOT_SRCS}
)

target_include_directories(krkr2plugin PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
)

# 子目录插件（各自的 CMakeLists.txt 定义独立目标）
add_subdirectory(psdfile)
add_subdirectory(psbfile)
add_subdirectory(layerex_draw)
add_subdirectory(motionplayer)
add_subdirectory(fstat)
# add_subdirectory(json)    # 已禁用
# add_subdirectory(steam)   # 已禁用
# add_subdirectory(DrawDeviceForSteam)  # 已禁用

target_link_libraries(krkr2plugin PUBLIC
    psdfile
    psbfile
    layerExDraw
    motionplayer
    fstat
)

target_link_libraries(krkr2plugin PUBLIC
    krkr2core
)

# 第三方格式化和日志库
find_package(fmt CONFIG REQUIRED)
find_package(spdlog CONFIG REQUIRED)
target_link_libraries(krkr2plugin PUBLIC
    fmt::fmt
    spdlog::spdlog
)
```

### 逐段深度分析

#### STATIC 库 vs INTERFACE 库

`krkr2plugin` 和 `krkr2core` 形成了鲜明对比：

| 特性 | krkr2core (INTERFACE) | krkr2plugin (STATIC) |
|------|----------------------|---------------------|
| 有源文件 | ❌ 无 | ✅ 16 个 .cpp 文件 |
| 产生二进制 | ❌ 无 | ✅ `libkrkr2plugin.a` / `krkr2plugin.lib` |
| 链接子模块 | INTERFACE 传播 | PUBLIC 直接链接 |
| 编译开销 | 零 | 编译 16 个源文件 + 5 个子目录 |
| 设计意图 | 聚合核心模块的属性 | 将所有插件编译为一个静态库 |

#### 根目录源文件（16 个）

这 16 个 `.cpp` 文件是"松散插件"——它们不需要独立的子目录结构，每个文件就是一个完整的插件实现：

| 文件 | 功能 | 注册方式 |
|------|------|---------|
| `ExtObj.cpp` | 扩展对象（扩展 TJS2 内置类） | NCB_PRE_REGIST_CALLBACK |
| `KAGParserEx.cpp` | KAG 脚本解析器扩展 | ncbind |
| `LayerBitmapFont.cpp` | 位图字体渲染 | ncbind |
| `LayerExBase.cpp` | 图层扩展基类 | ncbind |
| `LongExposure.cpp` | 长曝光效果（图像处理） | ncbind |
| `Menu.cpp` | 菜单系统 | ncbind |
| `csvParser.cpp` | CSV 文件解析 | ncbind |
| `dirlist.cpp` | 目录列表遍历 | ncbind |
| `fftgraph.cpp` | FFT 频谱图渲染 | ncbind |
| `saveStruct.cpp` | 存档结构序列化 | ncbind |
| `scriptsEx.cpp` | TJS2 脚本扩展函数 | NCB_PRE_REGIST_CALLBACK |
| `textrender.cpp` | 高级文本渲染 | ncbind |
| `varfile.cpp` | 变量文件读写 | ncbind |
| `windowEx.cpp` | 窗口扩展功能 | ncbind |
| `wutcwf.cpp` | 工具函数集合 | ncbind |
| `xp3filter.cpp` | XP3 归档解密过滤器 | simplebinder |

#### 子目录插件（5 个活跃 + 3 个禁用）

较复杂的插件有独立子目录：

```
plugins/
├── psdfile/        # PSD 文件解析器（多个源文件）
├── psbfile/        # PSB (E-mote) 格式解析器
├── layerex_draw/   # 扩展绘图（GDI+ / 通用）
├── motionplayer/   # E-mote 动画播放器
├── fstat/          # 文件状态查询
├── json/           # JSON 解析（已禁用）
├── steam/          # Steam 集成（已禁用）
└── DrawDeviceForSteam/  # Steam 渲染设备（已禁用）
```

禁用的三个插件（`json`、`steam`、`DrawDeviceForSteam`）在 CMakeLists.txt 中被注释掉。注意 `json` 插件被禁用可能是因为项目已经通过其他方式（如 nlohmann/json 或 TJS2 内置）处理 JSON。

#### PUBLIC 链接 krkr2core

```cmake
target_link_libraries(krkr2plugin PUBLIC
    krkr2core
)
```

这行看似简单，实则是整个构建系统的**关键连接点**。效果：

1. `krkr2plugin` 获得 `krkr2core` 的所有 INTERFACE 属性（包含目录、编译定义、OpenMP 等）
2. 因为使用 `PUBLIC`，链接 `krkr2plugin` 的目标（即 `krkr2`）也会继承这些属性
3. 形成完整传播链：`krkr2` → `krkr2plugin` → `krkr2core` → 9 个核心模块 + 第三方库

---

## 完整依赖传播链

综合根 CMakeLists.txt（第 01 章分析过）和本章的两个库，KrKr2 的完整依赖图：

```
krkr2（可执行文件 / Android .so）
│
├─ krkr2plugin（STATIC 库）
│  ├─ 16 个根目录插件源文件
│  ├─ psdfile, psbfile, layerExDraw, motionplayer, fstat
│  ├─ fmt::fmt, spdlog::spdlog
│  └─ krkr2core（INTERFACE 库，PUBLIC 传播）
│     ├─ tjs2, core_base_module, core_environ_module
│     ├─ core_extension_module, core_plugin_module
│     ├─ core_movie_module, core_sound_module
│     ├─ core_visual_module, core_utils_module
│     ├─ 编译定义: TJS_TEXT_OUT_CRLF, __STDC_CONSTANT_MACROS, USE_UNICODE_FSTRING
│     ├─ 包含目录: cpp/core/
│     ├─ OpenMP::OpenMP_CXX（非 Apple）
│     ├─ Android 系统库: log, android, EGL, GLESv2, GLESv1_CM, OpenSLES
│     └─ cocos2dx::cocos2d (+WHOLE_ARCHIVE cpp_android_spec on Android)
│
├─ krkr2core（INTERFACE 库，PUBLIC 传播 — 与上面相同）
│  └─ （同上，重复传播被 CMake 自动去重）
│
├─ 平台入口源文件:
│  ├─ Windows: platforms/windows/main.cpp + .rc 资源
│  ├─ Linux:   platforms/linux/main.cpp
│  ├─ macOS:   platforms/apple/macos/main.cpp
│  └─ Android: platforms/android/cpp/krkr2_android.cpp
│
├─ MSVC 编译选项: /EHsc /MP /utf-8（仅 Windows）
└─ Cocos 资源拷贝: cocos_copy_target_res()（非 Android）
```

> **注意重复链接：** 根 CMakeLists.txt 中 `krkr2` 同时链接了 `krkr2plugin` 和 `krkr2core`。由于 `krkr2plugin` 已经 PUBLIC 链接了 `krkr2core`，直接链接 `krkr2core` 看起来是冗余的。CMake 会自动去重，不会导致链接错误。这可能是为了代码清晰——明确表示 `krkr2` 依赖两个顶层目标。

---

## 动手实践

### 练习 1：创建自己的 INTERFACE 聚合库

创建一个包含两个子模块的微型项目，体会 INTERFACE 聚合模式：

```
mini_project/
├── CMakeLists.txt          # 根
├── core/
│   ├── CMakeLists.txt      # INTERFACE 聚合
│   ├── math/
│   │   ├── CMakeLists.txt
│   │   ├── math_ops.h
│   │   └── math_ops.cpp
│   └── string/
│       ├── CMakeLists.txt
│       ├── string_ops.h
│       └── string_ops.cpp
└── app/
    └── main.cpp
```

**core/math/math_ops.h:**
```cpp
// core/math/math_ops.h
#pragma once

namespace mini {
// 计算阶乘（递归实现）
int factorial(int n);
// 计算斐波那契数列第 n 项
int fibonacci(int n);
}  // namespace mini
```

**core/math/math_ops.cpp:**
```cpp
// core/math/math_ops.cpp
#include "math_ops.h"

namespace mini {
int factorial(int n) {
    if (n <= 1) return 1;       // 递归终止条件
    return n * factorial(n - 1); // 递归计算
}
int fibonacci(int n) {
    if (n <= 0) return 0;
    if (n == 1) return 1;
    return fibonacci(n - 1) + fibonacci(n - 2);
}
}  // namespace mini
```

**core/math/CMakeLists.txt:**
```cmake
# 创建 STATIC 库（子模块）
add_library(mini_math STATIC math_ops.cpp)
# PUBLIC 传播包含目录 — 链接者可以 #include "math_ops.h"
target_include_directories(mini_math PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
```

**core/string/string_ops.h:**
```cpp
// core/string/string_ops.h
#pragma once
#include <string>

namespace mini {
// 反转字符串
std::string reverse(const std::string& s);
// 转大写
std::string to_upper(const std::string& s);
}  // namespace mini
```

**core/string/string_ops.cpp:**
```cpp
// core/string/string_ops.cpp
#include "string_ops.h"
#include <algorithm>  // std::transform, std::reverse
#include <cctype>     // std::toupper

namespace mini {
std::string reverse(const std::string& s) {
    std::string result = s;
    std::reverse(result.begin(), result.end());  // 就地反转
    return result;
}
std::string to_upper(const std::string& s) {
    std::string result = s;
    // 逐字符转大写
    std::transform(result.begin(), result.end(), result.begin(),
                   [](unsigned char c) { return std::toupper(c); });
    return result;
}
}  // namespace mini
```

**core/string/CMakeLists.txt:**
```cmake
add_library(mini_string STATIC string_ops.cpp)
target_include_directories(mini_string PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
```

**core/CMakeLists.txt（INTERFACE 聚合）：**
```cmake
# 创建 INTERFACE 聚合库 — 关键！不编译任何文件
add_library(mini_core INTERFACE)

# 注册子模块
add_subdirectory(math)
add_subdirectory(string)

# INTERFACE 链接 — 让 mini_core 的链接者自动获得两个子模块
target_link_libraries(mini_core INTERFACE
    mini_math
    mini_string
)

# 可选：添加全局编译定义
target_compile_definitions(mini_core INTERFACE
    -DMINI_VERSION="1.0.0"
)
```

**app/main.cpp:**
```cpp
// app/main.cpp
#include <iostream>
#include "math_ops.h"    // 来自 mini_math（通过 mini_core INTERFACE 传播）
#include "string_ops.h"  // 来自 mini_string（通过 mini_core INTERFACE 传播）

int main() {
    // 使用 math 子模块
    std::cout << "5! = " << mini::factorial(5) << std::endl;         // 输出: 120
    std::cout << "fib(10) = " << mini::fibonacci(10) << std::endl;   // 输出: 55

    // 使用 string 子模块
    std::cout << mini::reverse("Hello") << std::endl;    // 输出: olleH
    std::cout << mini::to_upper("hello") << std::endl;    // 输出: HELLO

    // 使用通过 INTERFACE 传播的编译定义
    #ifdef MINI_VERSION
    std::cout << "Version: " << MINI_VERSION << std::endl; // 输出: Version: 1.0.0
    #endif

    return 0;
}
```

**根 CMakeLists.txt:**
```cmake
cmake_minimum_required(VERSION 3.28)
project(mini_project)

set(CMAKE_CXX_STANDARD 17)

# 注册 core 聚合库
add_subdirectory(core)

# 创建应用 — 只需链接 mini_core 一个目标！
add_executable(app app/main.cpp)
target_link_libraries(app PRIVATE mini_core)
# 无需单独链接 mini_math 和 mini_string — INTERFACE 传播自动处理
```

**构建和运行：**

```bash
# Windows (MSVC)
cmake -B build -G "Ninja" -DCMAKE_CXX_COMPILER=cl
cmake --build build
./build/app.exe

# Linux (GCC)
cmake -B build -G "Ninja"
cmake --build build
./build/app

# macOS (Apple Clang)
cmake -B build -G "Ninja"
cmake --build build
./build/app
```

**预期输出：**
```
5! = 120
fib(10) = 55
olleH
HELLO
Version: 1.0.0
```

### 练习 2：为 KrKr2 添加新的核心子模块

假设你要为 KrKr2 添加一个 `network` 网络模块，步骤如下：

1. 创建目录 `cpp/core/network/`
2. 创建 `cpp/core/network/CMakeLists.txt`：

```cmake
# cpp/core/network/CMakeLists.txt
add_library(core_network_module STATIC
    NetworkManager.cpp
    HttpClient.cpp
    WebSocket.cpp
)

target_include_directories(core_network_module PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
)

# 链接 krkr2core 以获取公共编译定义和包含目录
# 注意：这里不需要链接，因为 krkr2core 是 INTERFACE 库
# 子模块只需要声明自己的依赖
find_package(CURL CONFIG REQUIRED)
target_link_libraries(core_network_module PRIVATE CURL::libcurl)
```

3. 在 `cpp/core/CMakeLists.txt` 中注册：

```cmake
# 在现有的 add_subdirectory 列表末尾添加
add_subdirectory(network)

# 在 target_link_libraries 中添加
target_link_libraries(${PROJECT_NAME} INTERFACE
    tjs2
    core_base_module
    # ... 现有模块 ...
    core_utils_module
    core_network_module    # 新增！
)
```

4. 在 `vcpkg.json` 中添加 CURL 依赖（如果尚未存在）

完成后，所有链接 `krkr2core` 的目标自动获得 `network` 模块的功能——无需修改根 CMakeLists.txt。

---

## 对照项目源码

| 文件路径 | 行号 | 关键内容 |
|---------|------|---------|
| `cpp/core/CMakeLists.txt` | 1-67 | krkr2core INTERFACE 库完整定义 |
| `cpp/plugins/CMakeLists.txt` | 1-74 | krkr2plugin STATIC 库完整定义 |
| `CMakeLists.txt` | 115-117 | `target_link_libraries(krkr2 PUBLIC krkr2plugin krkr2core)` — 顶层链接 |
| `cpp/core/tjs2/CMakeLists.txt` | — | 子模块示例：tjs2 STATIC 库定义 |
| `cpp/core/visual/CMakeLists.txt` | — | 子模块示例：core_visual_module 定义 |
| `cmake/vcpkg_android.cmake` | 1-99 | Android 工具链叠加（影响 find_package 查找路径） |

建议用编辑器打开以上文件，对照本节内容逐行阅读。

---

## 本节小结

1. **INTERFACE 库不产生二进制**，它是属性传播的容器——包含目录、编译定义、链接依赖都通过 INTERFACE 关键字传播给链接者
2. **krkr2core** 使用 INTERFACE 聚合模式将 9 个 STATIC 子模块打包为一个统一链接目标，下游只需一行 `target_link_libraries` 就能获得所有核心功能
3. **krkr2plugin** 使用 STATIC 库，将 16 个松散插件源文件和 5 个子目录插件编译为一个静态库，并 PUBLIC 链接 krkr2core
4. **编译定义传播** 确保 `TJS_TEXT_OUT_CRLF` 等宏在整个依赖树中一致
5. **平台条件链接** 通过 `if(NOT APPLE)` 和 `if(ANDROID)` 处理 OpenMP 和系统库的跨平台差异
6. **生成器表达式** `$<LINK_LIBRARY:WHOLE_ARCHIVE,...>` 解决 Android JNI 符号被优化掉的问题
7. **依赖传播链** `krkr2` → `krkr2plugin` → `krkr2core` → 子模块 → 第三方库，自动、无遗漏

---

## 练习题与答案

### 题目 1：INTERFACE 库的属性传播

以下 CMake 代码中，`app` 目标会获得哪些编译定义？

```cmake
add_library(liba INTERFACE)
target_compile_definitions(liba INTERFACE -DFOO=1)

add_library(libb STATIC b.cpp)
target_compile_definitions(libb PUBLIC -DBAR=2)
target_link_libraries(libb PUBLIC liba)

add_library(libc INTERFACE)
target_compile_definitions(libc INTERFACE -DBAZ=3)

add_executable(app main.cpp)
target_link_libraries(app PRIVATE libb libc)
```

<details>
<summary>查看答案</summary>

`app` 目标会获得 **`FOO=1`、`BAR=2`、`BAZ=3`** 三个编译定义。

传播链分析：
1. `app` PRIVATE 链接 `libb` → 获得 `libb` 的 PUBLIC 定义 `BAR=2`
2. `libb` PUBLIC 链接 `liba` → `liba` 的 INTERFACE 定义 `FOO=1` 传播到 `libb` 的 PUBLIC 接口 → 进而传播到 `app`
3. `app` PRIVATE 链接 `libc` → 获得 `libc` 的 INTERFACE 定义 `BAZ=3`

关键点：PRIVATE 链接**不阻止**上游 PUBLIC/INTERFACE 属性的传播。PRIVATE 只影响**当前目标**是否把自己的属性继续向下传播，不影响它从上游**接收**属性。

</details>

### 题目 2：为什么 krkr2plugin 用 PUBLIC 而不是 PRIVATE 链接 krkr2core？

如果把 `target_link_libraries(krkr2plugin PUBLIC krkr2core)` 改为 `PRIVATE`，会发生什么？

<details>
<summary>查看答案</summary>

如果改为 `PRIVATE`：

1. **`krkr2plugin` 自身编译正常** — 它仍然能使用 krkr2core 的所有属性
2. **`krkr2` 可执行文件编译失败** — 因为 `krkr2` 链接 `krkr2plugin` 时，不再自动继承 `krkr2core` 的 INTERFACE 属性

具体表现：
- `krkr2` 的源文件找不到 `cpp/core/` 下的头文件（包含目录未传播）
- `krkr2` 缺少 `TJS_TEXT_OUT_CRLF` 等编译定义（可能导致链接错误或运行时行为不一致）
- `krkr2` 未链接 OpenMP、Android 系统库、cocos2dx 等（链接错误：undefined reference）

当然，根 CMakeLists.txt 中 `krkr2` 还直接链接了 `krkr2core`，所以实际上改为 PRIVATE 不会完全导致失败——但这是冗余设计带来的安全网。如果去掉根 CMakeLists.txt 中对 `krkr2core` 的直接链接，改为 PRIVATE 就会导致上述所有问题。

最佳实践：当 STATIC 库的公开头文件中 `#include` 了上游库的头文件时，必须使用 PUBLIC 链接。`krkr2plugin` 的插件代码大量使用 TJS2 和核心模块的头文件，所以 PUBLIC 是必须的。

</details>

### 题目 3：WHOLE_ARCHIVE 实战

下面的代码尝试在 Android 上使用 WHOLE_ARCHIVE，但有一个错误。找出错误并修正：

```cmake
target_link_libraries(myapp PUBLIC
    $<$<PLATFORM_ID:Android>:$<LINK_LIBRARY:WHOLE_ARCHIVE,myjni_lib>>
)
```

<details>
<summary>查看答案</summary>

代码本身**语法上没有错误**，但与 KrKr2 的实现方式不同，这可能导致一个隐藏问题。

KrKr2 使用 `$<BOOL:${ANDROID}>` 而非 `$<PLATFORM_ID:Android>`。区别在于：

- `$<PLATFORM_ID:Android>` — 检查 `CMAKE_SYSTEM_NAME` 是否为 `Android`（CMake 标准变量）
- `$<BOOL:${ANDROID}>` — 检查自定义变量 `ANDROID` 是否为真

**潜在问题：** 如果项目使用 CMake 工具链文件设置了 `CMAKE_SYSTEM_NAME=Android`，那么 `$<PLATFORM_ID:Android>` 是正确的。但 KrKr2 项目定义了自定义变量 `ANDROID`（在 `cmake/vcpkg_android.cmake` 中），并且某些交叉编译配置中 `CMAKE_SYSTEM_NAME` 可能不是标准的 `Android`。

**更可靠的修正**（匹配 KrKr2 风格）：

```cmake
target_link_libraries(myapp PUBLIC
    $<$<BOOL:${ANDROID}>:$<LINK_LIBRARY:WHOLE_ARCHIVE,myjni_lib>>
)
```

**另一个常见错误：** 忘记要求 CMake 3.24+。`LINK_LIBRARY` 生成器表达式在 3.24 版本才引入。如果 `cmake_minimum_required` 设置低于 3.24，此代码会产生配置错误。

</details>

---

## 下一步

下一节 [03-平台条件编译实现](./03-平台条件编译实现.md) 将深入分析 CMakePresets.json 的平台配置、条件编译宏体系、以及项目如何在一套代码中支持 Windows/Linux/macOS/Android 四个平台的差异化构建。
