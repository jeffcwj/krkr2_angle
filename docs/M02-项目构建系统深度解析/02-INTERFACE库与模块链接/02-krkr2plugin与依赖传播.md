# krkr2plugin：STATIC 聚合库（cpp/plugins/CMakeLists.txt）
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

