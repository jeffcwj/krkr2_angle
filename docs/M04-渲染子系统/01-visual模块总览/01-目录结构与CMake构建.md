# 目录结构与 CMake 构建

> **所属模块：** M04-渲染子系统
> **前置知识：** [P01-现代CMake与构建工具链](../../P01-现代CMake与构建工具链/)、[M01-项目导览与环境搭建](../../M01-项目导览与环境搭建/)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：

1. 描述 `cpp/core/visual/` 目录的完整结构和各子目录职责
2. 理解 `core_visual_module` CMake target 的构建配置
3. 列出 visual 模块依赖的所有第三方库及其用途
4. 解释 PUBLIC/PRIVATE/INTERFACE 链接语义在 visual 模块中的应用
5. 识别平台相关的源文件和条件编译配置

---

## 1. visual 模块概述

`cpp/core/visual/` 是 KrKr2 引擎中最复杂的核心模块之一，负责整个渲染子系统的实现。它包含：

- **图层管理**：Layer/LayerBitmap/LayerManager 体系
- **图像加载**：PNG/JPEG/WebP/TLG/BPG/JXR/PVR 等格式支持
- **渲染后端**：OpenGL 硬件加速与软件渲染两种模式
- **过渡效果**：场景切换时的视觉特效
- **字体渲染**：FreeType 字体光栅化
- **视频叠加**：视频播放时的 overlay 处理

该模块编译为 **静态库**（STATIC library），最终链接进 `krkr2core` 接口库，供应用程序使用。

---

## 2. 目录结构详解

```
cpp/core/visual/
├── CMakeLists.txt          # 模块构建配置
├── AGENTS.md               # 模块知识库
│
├── [根目录源文件]           # 核心接口与实现
│   ├── LayerIntf.{h,cpp}       # 图层接口（1205 行）
│   ├── LayerBitmapIntf.{h,cpp} # 位图图层接口
│   ├── LayerManager.{h,cpp}    # 图层管理器（567 行）
│   ├── RenderManager.{h,cpp}   # 抽象渲染管理器（316 行）
│   ├── WindowIntf.{h,cpp}      # 窗口接口
│   ├── BitmapIntf.{h,cpp}      # 位图接口
│   ├── TransIntf.{h,cpp}       # 过渡效果接口
│   ├── VideoOvlIntf.{h,cpp}    # 视频叠加接口
│   ├── GraphicsLoaderIntf.{h,cpp}  # 图像加载器接口
│   ├── GraphicsLoadThread.{h,cpp}  # 异步加载线程
│   │
│   ├── [图像加载器]
│   │   ├── LoadPNG.cpp         # PNG 格式加载
│   │   ├── LoadJPEG.cpp        # JPEG 格式加载
│   │   ├── LoadWEBP.cpp        # WebP 格式加载
│   │   ├── LoadTLG.{cpp,h}     # TLG 原生格式加载
│   │   ├── LoadBPG.cpp         # BPG 格式加载
│   │   ├── LoadJXR.cpp         # JPEG XR 格式加载
│   │   └── LoadPVRv3.cpp       # PVR v3 纹理加载
│   │
│   ├── [TLG 保存器]
│   │   ├── SaveTLG5.cpp        # TLG5 格式保存
│   │   └── SaveTLG6.cpp        # TLG6 格式保存
│   │
│   ├── [软件渲染]
│   │   ├── tvpgl.{h,cpp}       # 像素操作库
│   │   ├── tvpgl_asm_init.h    # ASM/SIMD 初始化
│   │   ├── argb.{h,cpp}        # ARGB 像素操作
│   │   └── tvpps.inc           # 像素着色器 include
│   │
│   ├── [字体系统]
│   │   ├── FontSystem.{h,cpp}          # 字体管理
│   │   ├── FontImpl.{h,cpp}            # 字体实现
│   │   ├── FreeType.{h,cpp}            # FreeType 封装
│   │   ├── FreeTypeFontRasterizer.{h,cpp}  # FT 光栅化
│   │   ├── PrerenderedFont.{h,cpp}     # 预渲染字体
│   │   └── CharacterData.{h,cpp}       # 字符数据
│   │
│   └── [辅助类]
│       ├── ComplexRect.{h,cpp}     # 复杂矩形区域
│       ├── RectItf.{h,cpp}         # 矩形接口
│       ├── ImageFunction.{h,cpp}   # 图像处理函数
│       ├── MenuItemIntf.{h,cpp}    # 菜单项接口
│       ├── LayerTreeOwnerImpl.{h,cpp}      # 图层树所有者
│       └── BitmapLayerTreeOwner.{h,cpp}    # 位图图层树所有者
│
├── gl/                     # 通用图像算法（平台无关）
│   ├── blend_function.cpp      # 混合函数
│   ├── ResampleImage.cpp       # 图像重采样
│   └── WeightFunctor.cpp       # 权重函数（插值）
│
├── ogl/                    # OpenGL 渲染后端
│   ├── RenderManager_ogl.cpp   # OGL 渲染管理器实现
│   ├── imagepacker.{h,cpp}     # 纹理图集打包
│   ├── pvrtc.{h,cpp}           # PVRTC 纹理压缩
│   ├── etcpak.{h,cpp}          # ETC 纹理压缩
│   └── astcrt.{h,cpp}          # ASTC 纹理压缩
│
└── impl/                   # 平台相关实现
    ├── LayerImpl.{h,cpp}           # 图层实现
    ├── LayerBitmapImpl.{h,cpp}     # 位图图层实现
    ├── WindowImpl.{h,cpp}          # 窗口实现
    ├── VideoOvlImpl.{h,cpp}        # 视频叠加实现
    ├── MenuItemImpl.{h,cpp}        # 菜单项实现
    ├── GraphicsLoaderImpl.{h,cpp}  # 图像加载器实现
    ├── DrawDevice.{h,cpp}          # 绘制设备
    ├── BasicDrawDevice.{h,cpp}     # 基础绘制设备
    ├── PassThroughDrawDevice.{h,cpp}   # 直通绘制设备
    ├── TVPScreen.{h,cpp}           # 屏幕管理
    ├── BitmapBitsAlloc.{h,cpp}     # 位图内存分配
    ├── BitmapInfomation.{h,cpp}    # 位图信息
    ├── DInputMgn.{h,cpp}           # DirectInput 管理
    │
    └── [被注释/平台可选的实现]
        ├── GDIFontRasterizer.cpp   # Windows GDI 字体（已注释）
        ├── NativeFreeTypeFace.cpp  # 原生 FT 字体（已注释）
        ├── TVPSysFont.cpp          # 系统字体（已注释）
        └── VSyncTimingThread.cpp   # VSync 线程（已注释）
```

### 2.1 子目录职责说明

| 子目录 | 职责 | 典型文件 |
|--------|------|---------|
| `gl/` | 通用像素/图像算法，平台无关 | `ResampleImage.cpp`（Lanczos/双线性插值） |
| `ogl/` | OpenGL 渲染后端，GPU 纹理处理 | `RenderManager_ogl.cpp`，纹理压缩工具 |
| `impl/` | 平台相关的具体实现 | `WindowImpl.cpp`，`LayerImpl.cpp` |

### 2.2 文件命名规范

visual 模块遵循以下命名约定：

```cpp
// 接口类：*Intf.{h,cpp}
LayerIntf.h          // 图层抽象接口
TransIntf.h          // 过渡效果接口

// 实现类：*Impl.{h,cpp}（通常在 impl/ 子目录）
LayerImpl.h          // 图层具体实现
WindowImpl.h         // 窗口具体实现

// 加载器：Load*.cpp（每种格式一个文件）
LoadPNG.cpp          // PNG 加载器
LoadJPEG.cpp         // JPEG 加载器

// 保存器：Save*.cpp
SaveTLG5.cpp         // TLG5 保存器
```

---

## 3. CMake 构建配置分析

### 3.1 完整 CMakeLists.txt 解析

```cmake
# cpp/core/visual/CMakeLists.txt

cmake_minimum_required(VERSION 3.19)
project(core_visual_module LANGUAGES CXX)

# ============== 1. 定义源文件列表 ==============
set(VISUAL_PATH ${CMAKE_CURRENT_LIST_DIR})
set(VISUAL_SOURCE_FILES
    # 根目录源文件
    ${VISUAL_PATH}/LoadWEBP.cpp
    ${VISUAL_PATH}/TransIntf.cpp
    ${VISUAL_PATH}/MenuItemIntf.cpp
    ${VISUAL_PATH}/SaveTLG5.cpp
    ${VISUAL_PATH}/FontSystem.cpp
    ${VISUAL_PATH}/tvpps.inc
    ${VISUAL_PATH}/LoadPVRv3.cpp
    ${VISUAL_PATH}/RectItf.cpp
    ${VISUAL_PATH}/VideoOvlIntf.cpp
    ${VISUAL_PATH}/LayerBitmapIntf.cpp
    ${VISUAL_PATH}/FontImpl.cpp
    ${VISUAL_PATH}/RenderManager.cpp
    ${VISUAL_PATH}/LoadJPEG.cpp
    ${VISUAL_PATH}/GraphicsLoaderIntf.cpp
    ${VISUAL_PATH}/SaveTLG6.cpp
    ${VISUAL_PATH}/FreeType.cpp
    ${VISUAL_PATH}/LoadJXR.cpp
    ${VISUAL_PATH}/ImageFunction.cpp
    ${VISUAL_PATH}/tvpgl.cpp
    ${VISUAL_PATH}/LoadTLG.cpp
    ${VISUAL_PATH}/LayerTreeOwnerImpl.cpp
    ${VISUAL_PATH}/GraphicsLoadThread.cpp
    ${VISUAL_PATH}/LayerManager.cpp
    ${VISUAL_PATH}/PrerenderedFont.cpp
    ${VISUAL_PATH}/LoadBPG.cpp
    ${VISUAL_PATH}/WindowIntf.cpp
    ${VISUAL_PATH}/FreeTypeFontRasterizer.cpp
    ${VISUAL_PATH}/LoadPNG.cpp
    ${VISUAL_PATH}/LayerIntf.cpp
    ${VISUAL_PATH}/ComplexRect.cpp
    ${VISUAL_PATH}/CharacterData.cpp
    ${VISUAL_PATH}/BitmapLayerTreeOwner.cpp
    ${VISUAL_PATH}/BitmapIntf.cpp
    ${VISUAL_PATH}/argb.cpp

    # gl/ 子目录：通用图像算法
    ${VISUAL_PATH}/gl/blend_function.cpp
    ${VISUAL_PATH}/gl/ResampleImage.cpp
    ${VISUAL_PATH}/gl/WeightFunctor.cpp

    # ogl/ 子目录：OpenGL 后端
    ${VISUAL_PATH}/ogl/astcrt.cpp
    ${VISUAL_PATH}/ogl/etcpak.cpp
    ${VISUAL_PATH}/ogl/imagepacker.cpp
    ${VISUAL_PATH}/ogl/pvrtc.cpp
    ${VISUAL_PATH}/ogl/RenderManager_ogl.cpp

    # impl/ 子目录：平台实现
    ${VISUAL_PATH}/impl/BasicDrawDevice.cpp
    ${VISUAL_PATH}/impl/BitmapBitsAlloc.cpp
    ${VISUAL_PATH}/impl/BitmapInfomation.cpp
    ${VISUAL_PATH}/impl/DInputMgn.cpp
    ${VISUAL_PATH}/impl/DrawDevice.cpp
    # ${VISUAL_PATH}/impl/GDIFontRasterizer.cpp    # Windows GDI（已注释）
    ${VISUAL_PATH}/impl/GraphicsLoaderImpl.cpp
    ${VISUAL_PATH}/impl/LayerBitmapImpl.cpp
    ${VISUAL_PATH}/impl/LayerImpl.cpp
    ${VISUAL_PATH}/impl/MenuItemImpl.cpp
    # ${VISUAL_PATH}/impl/NativeFreeTypeFace.cpp   # 原生字体（已注释）
    ${VISUAL_PATH}/impl/PassThroughDrawDevice.cpp
    ${VISUAL_PATH}/impl/TVPScreen.cpp
    # ${VISUAL_PATH}/impl/TVPSysFont.cpp           # 系统字体（已注释）
    ${VISUAL_PATH}/impl/VideoOvlImpl.cpp
    # ${VISUAL_PATH}/impl/VSyncTimingThread.cpp    # VSync 线程（已注释）
    ${VISUAL_PATH}/impl/WindowImpl.cpp
)

# ============== 2. 定义头文件目录 ==============
set(VISUAL_HEADERS_DIR
    ${VISUAL_PATH}/
    ${VISUAL_PATH}/gl
    ${VISUAL_PATH}/ogl
    ${VISUAL_PATH}/impl
)

# ============== 3. 创建静态库 ==============
add_library(${PROJECT_NAME} STATIC ${VISUAL_SOURCE_FILES})
target_include_directories(${PROJECT_NAME} PUBLIC ${VISUAL_HEADERS_DIR})

# ============== 4. 链接内部模块 ==============
target_link_libraries(${PROJECT_NAME} 
    PUBLIC tjs2                     # TJS2 脚本引擎（PUBLIC：使用者可见）
    PRIVATE core_base_module        # 基础模块（I/O、归档）
            core_environ_module     # 平台抽象层
            core_plugin_module      # 插件系统
            core_sound_module       # 音频系统
            core_utils_module       # 工具模块
)

# ============== 5. 链接第三方库 ==============
# BPG 图像格式
target_link_libraries(${PROJECT_NAME} PRIVATE libbpg)

# WebP 图像格式
find_package(WebP CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} INTERFACE 
    WebP::webp WebP::webpdecoder WebP::webpdemux)

# Cocos2d-x 渲染框架
find_package(cocos2dx CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE cocos2dx::cocos2d)

# JPEG XR 图像格式
find_package(JXR REQUIRED)
target_compile_definitions(${PROJECT_NAME} PRIVATE -D__ANSI__)
target_include_directories(${PROJECT_NAME} PRIVATE ${JXR_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME} PRIVATE ${JXR_LIBRARIES})

# JPEG（使用 libjpeg-turbo）
find_package(libjpeg-turbo CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PUBLIC
    $<IF:$<TARGET_EXISTS:libjpeg-turbo::turbojpeg>,
        libjpeg-turbo::turbojpeg,
        libjpeg-turbo::turbojpeg-static>
)

# OpenCV（图像处理）
set(OpenCV_ROOT "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/share/opencv4")
find_package(OpenCV REQUIRED)
target_link_libraries(${PROJECT_NAME} PUBLIC opencv_imgproc opencv_core)

# LZ4 压缩（TLG 格式使用）
find_package(lz4 CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PUBLIC lz4::lz4)
```

### 3.2 target 名称与类型

```cmake
project(core_visual_module LANGUAGES CXX)
add_library(${PROJECT_NAME} STATIC ${VISUAL_SOURCE_FILES})
```

- **target 名称**：`core_visual_module`
- **target 类型**：`STATIC`（静态库）
- **输出文件**：
  - Windows: `core_visual_module.lib`
  - Linux/macOS: `libcore_visual_module.a`
  - Android: `libcore_visual_module.a`

### 3.3 链接语义详解

CMake 的 `target_link_libraries` 支持三种链接模式，visual 模块中都有使用：

| 模式 | 语义 | visual 模块中的用例 |
|------|------|---------------------|
| `PUBLIC` | 依赖传递给使用者 | `tjs2`、`libjpeg-turbo`、`opencv_*`、`lz4` |
| `PRIVATE` | 仅本模块使用 | `cocos2dx`、`libbpg`、`JXR`、其他 core 模块 |
| `INTERFACE` | 不参与本模块编译，仅传递给使用者 | `WebP::*` |

**为什么这样设计？**

```cpp
// PUBLIC 依赖示例：tjs2
// visual 模块的头文件中使用了 TJS2 类型，使用者也需要这些类型
// 例如 LayerIntf.h 包含 tjsNative.h
#include "tjsNative.h"  // tjs2 的头文件
class tTJSNI_BaseLayer : public tTJSNativeInstance { ... };

// PRIVATE 依赖示例：cocos2dx
// 仅在 .cpp 实现中使用，头文件不暴露
// RenderManager_ogl.cpp
#include "cocos2d.h"
void RenderManager_ogl::render() {
    cocos2d::Director::getInstance()->...
}

// INTERFACE 依赖示例：WebP
// visual 模块本身不直接链接 WebP，但使用者需要
// （可能是为了某些延迟加载场景）
```

---

## 4. 第三方依赖详解

### 4.1 依赖库清单

| 库 | 用途 | 链接方式 | 来源 |
|----|------|---------|------|
| `tjs2` | TJS2 脚本引擎绑定 | PUBLIC | 内部模块 |
| `core_base_module` | I/O 流、归档读取 | PRIVATE | 内部模块 |
| `core_environ_module` | 平台抽象层 | PRIVATE | 内部模块 |
| `core_plugin_module` | 插件系统 | PRIVATE | 内部模块 |
| `core_sound_module` | 音频系统 | PRIVATE | 内部模块 |
| `core_utils_module` | 工具函数 | PRIVATE | 内部模块 |
| `cocos2dx::cocos2d` | 渲染框架 | PRIVATE | vcpkg |
| `libbpg` | BPG 图像解码 | PRIVATE | external/ |
| `WebP::*` | WebP 图像解码 | INTERFACE | vcpkg |
| `JXR` | JPEG XR 解码 | PRIVATE | vcpkg |
| `libjpeg-turbo` | JPEG 解码（SIMD 优化） | PUBLIC | vcpkg |
| `opencv_imgproc/core` | 图像处理 | PUBLIC | vcpkg |
| `lz4::lz4` | LZ4 压缩（TLG 格式） | PUBLIC | vcpkg |

### 4.2 依赖关系图

```
krkr2 (应用程序)
└── krkr2core (INTERFACE 库)
    └── core_visual_module (STATIC 库)
        │
        ├── PUBLIC 依赖（传递给使用者）
        │   ├── tjs2                    # TJS2 脚本引擎
        │   ├── libjpeg-turbo::turbojpeg
        │   ├── opencv_imgproc
        │   ├── opencv_core
        │   └── lz4::lz4
        │
        ├── PRIVATE 依赖（仅本模块使用）
        │   ├── core_base_module        # I/O 基础
        │   ├── core_environ_module     # 平台抽象
        │   ├── core_plugin_module      # 插件系统
        │   ├── core_sound_module       # 音频系统
        │   ├── core_utils_module       # 工具函数
        │   ├── cocos2dx::cocos2d       # 渲染框架
        │   ├── libbpg                  # BPG 图像
        │   └── JXR_LIBRARIES           # JPEG XR
        │
        └── INTERFACE 依赖（仅传递）
            ├── WebP::webp
            ├── WebP::webpdecoder
            └── WebP::webpdemux
```

### 4.3 图像格式与对应库

```
┌─────────────┬────────────────────┬──────────────────┐
│ 图像格式     │ 加载器文件          │ 依赖库           │
├─────────────┼────────────────────┼──────────────────┤
│ PNG         │ LoadPNG.cpp        │ libpng (via cocos2dx) │
│ JPEG        │ LoadJPEG.cpp       │ libjpeg-turbo    │
│ WebP        │ LoadWEBP.cpp       │ WebP::*          │
│ TLG         │ LoadTLG.cpp        │ lz4::lz4         │
│ BPG         │ LoadBPG.cpp        │ libbpg           │
│ JPEG XR     │ LoadJXR.cpp        │ JXR_LIBRARIES    │
│ PVR v3      │ LoadPVRv3.cpp      │ (内置解析)        │
└─────────────┴────────────────────┴──────────────────┘
```

---

## 5. 平台相关配置

### 5.1 当前平台支持状态

visual 模块的主源码是**平台无关**的，编译进同一个静态库。平台差异主要通过以下方式处理：

1. **core 顶层配置**：`cpp/core/CMakeLists.txt` 对 Android 添加系统库
2. **environ 模块**：提供平台抽象层
3. **注释的 impl 文件**：特定平台的可选实现

### 5.2 被注释的平台实现

以下文件在 CMakeLists.txt 中被注释，表示当前构建配置中不启用：

| 文件 | 平台 | 用途 | 启用条件 |
|------|------|------|---------|
| `GDIFontRasterizer.cpp` | Windows | 使用 GDI 渲染字体 | 需要 Windows SDK |
| `NativeFreeTypeFace.cpp` | 跨平台 | 原生 FreeType 字体 | 替代 FT 封装 |
| `TVPSysFont.cpp` | 跨平台 | 系统字体管理 | 可选功能 |
| `VSyncTimingThread.cpp` | 跨平台 | VSync 同步线程 | 可选优化 |

### 5.3 如何启用平台特定实现

如果需要在 Windows 上启用 GDI 字体渲染，可以修改 CMakeLists.txt：

```cmake
# 方法 1：直接取消注释
${VISUAL_PATH}/impl/GDIFontRasterizer.cpp

# 方法 2：使用条件编译（推荐）
$<$<BOOL:${WINDOWS}>:${VISUAL_PATH}/impl/GDIFontRasterizer.cpp>
```

**完整示例：**

```cmake
set(VISUAL_SOURCE_FILES
    # ... 其他源文件 ...
    
    # 平台条件源文件
    $<$<BOOL:${WINDOWS}>:${VISUAL_PATH}/impl/GDIFontRasterizer.cpp>
    $<$<BOOL:${WINDOWS}>:${VISUAL_PATH}/impl/VSyncTimingThread.cpp>
)
```

### 5.4 Android 平台的额外链接

在 `cpp/core/CMakeLists.txt` 中，Android 平台会额外链接以下系统库：

```cmake
if(ANDROID)
    target_link_libraries(krkr2core INTERFACE
        log         # Android 日志
        android     # Android NDK 基础库
        EGL         # EGL 图形接口
        GLESv2      # OpenGL ES 2.0
        GLESv1_CM   # OpenGL ES 1.x
        OpenSLES    # 音频（由 sound 模块使用）
    )
endif()
```

---

## 6. 模块在项目中的位置

### 6.1 构建层次

```
krkr2/
├── CMakeLists.txt              # 根 CMake
│   └── add_subdirectory(cpp/core)
│
└── cpp/core/
    ├── CMakeLists.txt          # core 层 CMake
    │   ├── add_subdirectory(visual)     # 添加 visual 模块
    │   ├── add_subdirectory(sound)      # 添加 sound 模块
    │   ├── ...
    │   └── add_library(krkr2core INTERFACE)
    │       └── target_link_libraries(... core_visual_module ...)
    │
    └── visual/
        └── CMakeLists.txt      # visual 模块 CMake
            └── add_library(core_visual_module STATIC ...)
```

### 6.2 头文件包含路径

visual 模块将以下目录暴露为 PUBLIC include 路径：

```cmake
target_include_directories(${PROJECT_NAME} PUBLIC ${VISUAL_HEADERS_DIR})

# VISUAL_HEADERS_DIR 包含：
# - cpp/core/visual/
# - cpp/core/visual/gl/
# - cpp/core/visual/ogl/
# - cpp/core/visual/impl/
```

这意味着其他模块可以直接包含 visual 的头文件：

```cpp
// 在其他模块中
#include "LayerIntf.h"          // 来自 visual/
#include "RenderManager.h"      // 来自 visual/
#include "LayerBitmapImpl.h"    // 来自 visual/impl/
```

---

## 动手实践

### 实践 1：查看 visual 模块的编译输出

在构建目录中找到 visual 模块的静态库：

```bash
# Windows（假设使用 Debug 配置）
dir out\windows\debug\cpp\core\visual\

# 输出示例：
# core_visual_module.lib
# core_visual_module.dir\  # 编译中间文件

# Linux
ls out/linux/debug/cpp/core/visual/

# 输出示例：
# libcore_visual_module.a
# CMakeFiles/core_visual_module.dir/  # 编译中间文件
```

### 实践 2：检查依赖传递

使用 CMake 的 `--graphviz` 选项生成依赖图：

```bash
cd out/windows/debug
cmake --graphviz=deps.dot .

# 生成 deps.dot 文件，可用 Graphviz 可视化
dot -Tpng deps.dot -o deps.png
```

### 实践 3：添加新的图像格式支持

假设要添加 AVIF 格式支持，需要：

1. **创建加载器文件**：

```cpp
// cpp/core/visual/LoadAVIF.cpp
#include "GraphicsLoaderIntf.h"
#include <avif/avif.h>

void TVPLoadAVIF(void *formatdata, void *callbackdata,
                 tTVPGraphicSizeCallback sizecallback,
                 tTVPGraphicScanLineCallback scanlinecallback,
                 tTVPMetaInfoPushCallback metainfocallback,
                 tTJSBinaryStream *src, tjs_int keyidx,
                 tTVPGraphicLoadMode mode) {
    // AVIF 解码逻辑
}

void TVPRegisterAVIFLoader() {
    // 注册到加载器链
}
```

2. **修改 CMakeLists.txt**：

```cmake
# 添加源文件
set(VISUAL_SOURCE_FILES
    # ... 其他文件 ...
    ${VISUAL_PATH}/LoadAVIF.cpp
)

# 添加依赖
find_package(libavif CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE avif)
```

3. **更新 vcpkg.json**：

```json
{
  "dependencies": [
    "libavif"
  ]
}
```

---

## 对照项目源码

本节涉及的关键源文件：

| 文件路径 | 行数 | 说明 |
|----------|------|------|
| `cpp/core/visual/CMakeLists.txt` | 1-111 | 完整的构建配置 |
| `cpp/core/CMakeLists.txt` | 1-50 | core 层构建，包含 Android 配置 |
| `cpp/core/visual/AGENTS.md` | 1-57 | 模块知识库，包含目录说明 |

**CMakeLists.txt 关键行解读：**

```cmake
# 第 77 行：创建静态库
add_library(${PROJECT_NAME} STATIC ${VISUAL_SOURCE_FILES})

# 第 79 行：链接内部模块，注意 PUBLIC/PRIVATE 区分
target_link_libraries(${PROJECT_NAME} PUBLIC tjs2 
    PRIVATE core_base_module core_environ_module ...)

# 第 86-89 行：链接 Cocos2d-x
find_package(cocos2dx CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE cocos2dx::cocos2d)
```

---

## 本节小结

- visual 模块（`core_visual_module`）是 **STATIC 库**，包含约 50+ 源文件
- 目录分为三层：根目录（核心接口）、`gl/`（通用算法）、`ogl/`（GPU 后端）、`impl/`（平台实现）
- 依赖 7+ 个第三方库，通过 vcpkg 管理
- 使用 `PUBLIC`/`PRIVATE`/`INTERFACE` 控制依赖传递
- 部分 impl 文件被注释，可按需启用平台特定实现
- 图像格式支持遵循 `Load*.cpp` 模式，易于扩展

---

## 练习题与答案

### 题目 1：链接语义判断

以下依赖应该使用哪种链接方式？说明理由。

```
A. FreeType 字体库（仅在 .cpp 中使用，头文件不暴露 FT 类型）
B. PNG 解码库（头文件中有 png_struct* 类型）
C. 仅用于单元测试的 mock 库
```

<details>
<summary>查看答案</summary>

**A. FreeType → PRIVATE**

理由：仅在实现文件中使用，不需要传递给使用者。

```cmake
target_link_libraries(visual PRIVATE Freetype::Freetype)
```

**B. PNG 解码库 → PUBLIC**

理由：头文件中暴露了 PNG 类型，使用者需要访问这些类型。

```cmake
target_link_libraries(visual PUBLIC PNG::PNG)
```

**C. Mock 库 → 不应链接到 visual**

理由：测试库应该链接到测试 target，而不是生产代码。

```cmake
# 在测试 CMakeLists.txt 中
target_link_libraries(visual_tests PRIVATE gmock)
```

</details>

### 题目 2：添加 HEIF 格式支持

请写出添加 HEIF 图像格式支持所需修改的 CMakeLists.txt 片段。

要求：
- 源文件：`LoadHEIF.cpp`
- 依赖库：`libheif`（通过 vcpkg）
- 链接方式：PRIVATE

<details>
<summary>查看答案</summary>

```cmake
# 1. 在 VISUAL_SOURCE_FILES 中添加
set(VISUAL_SOURCE_FILES
    # ... 现有文件 ...
    ${VISUAL_PATH}/LoadHEIF.cpp
)

# 2. 查找并链接 libheif
find_package(libheif CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE heif)

# 3. 如果需要包含目录（通常 vcpkg 自动处理）
# target_include_directories(${PROJECT_NAME} PRIVATE ${LIBHEIF_INCLUDE_DIRS})
```

同时需要更新 `vcpkg.json`：

```json
{
  "dependencies": [
    "libheif"
  ]
}
```

</details>

### 题目 3：平台条件编译

如何修改 CMakeLists.txt，使 `GDIFontRasterizer.cpp` 仅在 Windows 平台编译？

<details>
<summary>查看答案</summary>

```cmake
# 方法 1：使用 generator expression（推荐）
set(VISUAL_SOURCE_FILES
    # ... 其他文件 ...
    $<$<BOOL:${WINDOWS}>:${VISUAL_PATH}/impl/GDIFontRasterizer.cpp>
)

# 方法 2：使用 if 语句
if(WINDOWS)
    list(APPEND VISUAL_SOURCE_FILES
        ${VISUAL_PATH}/impl/GDIFontRasterizer.cpp
    )
endif()

# 方法 3：使用 CMake 内置变量
if(WIN32)
    list(APPEND VISUAL_SOURCE_FILES
        ${VISUAL_PATH}/impl/GDIFontRasterizer.cpp
    )
endif()
```

注意：KrKr2 项目使用自定义的 `WINDOWS` 变量，而不是 CMake 内置的 `WIN32`，需要查看项目的平台变量定义。

</details>

---

## 下一步

[02-类层次与接口设计](./02-类层次与接口设计.md) — 深入理解 visual 模块的类继承结构和 Intf/Impl 设计模式。
