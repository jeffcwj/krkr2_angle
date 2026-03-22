# 01 - FlutterEngine 集成与 CMake 配置

> **所属模块：** M13-UI框架替换实战  
> **前置知识：** [02-UI抽象接口设计/02-KrKr2抽象层实战实现](../02-UI抽象接口设计/02-KrKr2抽象层实战实现.md)  
> **预计阅读时间：** 55 分钟

---

## 本节目标

- 理解 Flutter Embedder API 的核心设计与嵌入原理
- 掌握将 FlutterEngine 集成到现有 CMake C++ 项目的完整流程
- 学会在 Desktop（Windows/Linux/macOS）、Android、iOS 三类平台上分别配置构建
- 能够成功初始化 FlutterEngine 并运行第一个 Flutter Widget

---

## 术语预览

| 术语 | 解释 |
|---|---|
| **Flutter Embedder API** | Flutter 提供的底层 C API，允许任意宿主应用将 Flutter 渲染到自己的 OpenGL/Metal/Vulkan 上下文中 |
| **FlutterEngine** | Flutter 运行时的核心对象，管理 Dart VM、渲染管线、Platform Channel 等子系统 |
| **AOT 产物** | Ahead-Of-Time 编译产物，Flutter release 模式下 Dart 代码被编译为机器码（.so/.dylib/.dll） |
| **flutter_engine.h** | Embedder API 的主头文件，定义了 `FlutterEngineInitialize`、`FlutterEngineSendWindowMetricsEvent` 等函数 |
| **FlutterProjectArgs** | 初始化 FlutterEngine 时传入的配置结构体，包含 assets 路径、AOT 库路径等 |
| **FetchContent** | CMake 3.11+ 内置模块，可在配置阶段从 URL 下载并集成外部依赖 |
| **vcpkg** | 微软开源的 C++ 包管理器，支持 Windows/Linux/macOS 平台 |

---

## 一、Flutter Embedder API 概述

### 1.1 Flutter 的分层架构

Flutter 从设计之初就考虑到了跨平台嵌入的需求，其架构分为三层：

```
┌────────────────────────────────────────────────────────┐
│  Flutter Framework（Dart）                              │
│  Widget → RenderObject → Layer → Scene                 │
├────────────────────────────────────────────────────────┤
│  Flutter Engine（C++）                                  │
│  Dart VM | Skia/Impeller | Platform Channels | IO      │
├────────────────────────────────────────────────────────┤
│  Embedder（宿主应用实现）                                │
│  OpenGL/Metal/Vulkan 上下文 | 事件注入 | 任务运行器     │
└────────────────────────────────────────────────────────┘
```

**第一层**（Flutter Framework）是纯 Dart 代码，包含 Widget 树、动画系统、手势识别等。  
**第二层**（Flutter Engine）是 C++ 实现的运行时，包含 Dart VM、Skia/Impeller 渲染器、线程调度器。  
**第三层**（Embedder）需要由宿主应用（即 KrKr2）来实现，提供渲染上下文和系统事件。

KrKr2 已经有自己的 OpenGL 上下文（Cocos2d-x 使用 OpenGL ES 2/3），因此可以将 Flutter 的渲染输出直接共享到 Cocos2d-x 的纹理系统中。

### 1.2 Embedder API 的核心函数

Flutter Embedder API（`flutter/embedder.h`）定义了大约 30 个 C 函数：

```c
// 初始化并运行 FlutterEngine
FlutterEngineResult FlutterEngineInitialize(
    size_t version,
    const FlutterRendererConfig* config,
    const FlutterProjectArgs* args,
    void* user_data,
    FLUTTER_API_SYMBOL(FlutterEngine)* engine_out
);

// 向 Flutter 发送窗口尺寸变化事件
FlutterEngineResult FlutterEngineSendWindowMetricsEvent(
    FLUTTER_API_SYMBOL(FlutterEngine) engine,
    const FlutterWindowMetricsEvent* flutter_metrics
);

// 向 Flutter 发送指针输入事件
FlutterEngineResult FlutterEngineSendPointerEvent(
    FLUTTER_API_SYMBOL(FlutterEngine) engine,
    const FlutterPointerEvent* events,
    size_t events_count
);

// 向 Platform Channel 发送消息
FlutterEngineResult FlutterEngineSendPlatformMessage(
    FLUTTER_API_SYMBOL(FlutterEngine) engine,
    const FlutterPlatformMessage* message
);

// 通知 Flutter 执行一帧渲染
FlutterEngineResult FlutterEngineOnVsync(
    FLUTTER_API_SYMBOL(FlutterEngine) engine,
    intptr_t baton,
    uint64_t frame_start_time_nanos,
    uint64_t frame_target_time_nanos
);

// 关闭并销毁 FlutterEngine
FlutterEngineResult FlutterEngineShutdown(
    FLUTTER_API_SYMBOL(FlutterEngine) engine
);
```

这些函数都是线程安全的，可以从任意线程调用（不过 Vsync 通知必须来自特定的 Platform Task Runner）。

### 1.3 KrKr2 集成方案概览

KrKr2 使用的集成方案如下：

```
KrKr2 主循环（Cocos2d-x）
    │
    ├── FlutterEngine（独立 Dart VM）
    │       ├── Platform Task Runner ← 由 KrKr2 主线程驱动
    │       ├── Render Task Runner   ← 由 KrKr2 渲染线程驱动
    │       ├── IO Task Runner       ← 独立线程
    │       └── UI Task Runner       ← 独立线程（Dart VM 执行）
    │
    └── 纹理桥接层
            ├── Flutter 渲染 → OpenGL FBO
            └── Cocos2d-x 读取 FBO 作为外部纹理
```

---

## 二、获取 Flutter Engine 预编译库

### 2.1 方式一：使用 Flutter SDK 自带的 engine（推荐桌面端）

Flutter SDK 在 `bin/cache/artifacts/engine/` 目录下为每个平台预置了 engine 库。

```bash
# 查看 Flutter SDK 中的 engine 路径
flutter --version
# 找到 Flutter 安装目录，engine 在：
# {FLUTTER_SDK}/bin/cache/artifacts/engine/
#   ├── linux-x64/
#   │   ├── libflutter_engine.so
#   │   └── flutter_embedder.h
#   ├── windows-x64/
#   │   ├── flutter_engine.dll
#   │   ├── flutter_engine.dll.lib
#   │   └── flutter_embedder.h
#   └── darwin-x64/
#       ├── FlutterEmbedder.framework/
#       └── flutter_embedder.h

# 如果缓存不存在，运行任意 flutter 命令触发下载
flutter devices
```

**注意**：`flutter_embedder.h` 中的 `FLUTTER_ENGINE_VERSION` 常量必须与你使用的 engine 库完全一致，否则 `FlutterEngineInitialize` 会返回 `kInvalidLibraryVersion` 错误。

### 2.2 方式二：通过 GCS 下载特定版本（推荐 CI/CD）

Google 将每个 Flutter 版本的 engine 产物都上传到 GCS（Google Cloud Storage）：

```bash
# 获取当前 Flutter 版本对应的 engine hash
flutter --version
# 输出类似：Engine • revision 4d721f339f (16 hours ago) • 2024-01-15 12:34:56 +0000

# 或者读取 .flutter_tool_state 文件
cat $(flutter --version | grep "Flutter • " | head -1)/bin/internal/engine.version
# 输出：4d721f339fa4aa8caef83f66bdbb0e28b59e5c78

# 从 GCS 下载对应 engine（以 Linux x64 为例）
ENGINE_HASH="4d721f339fa4aa8caef83f66bdbb0e28b59e5c78"
BASE_URL="https://storage.googleapis.com/flutter_infra_release/flutter/${ENGINE_HASH}"
wget "${BASE_URL}/linux-x64/linux-x64-embedder.zip"
unzip linux-x64-embedder.zip -d flutter_engine/
```

下载后目录结构：
```
flutter_engine/
├── flutter_embedder.h    ← Embedder API 头文件
└── libflutter_engine.so  ← 动态链接库（Linux）
```

### 2.3 方式三：Android/iOS 使用 Gradle/CocoaPods

Android 和 iOS 平台不需要手动下载 engine，通过包管理工具自动处理：

**Android** — 在 `android/app/build.gradle` 中添加：
```groovy
dependencies {
    // Flutter engine 通过 flutter.gradle 插件自动引入
    // 只需确保 flutter.gradle 被正确 apply
    implementation "io.flutter:flutter_embedding_release:1.0.0-${getFlutterEngineVersion()}"
}
```

**iOS** — 在 `ios/Podfile` 中添加：
```ruby
target 'Runner' do
  use_frameworks!
  use_modular_headers!
  
  # flutter_engine 框架由 FlutterPluginRegistrant 自动引入
  pod 'Flutter', :path => Flutter.flutter_root
end
```

---

## 三、CMake 集成配置

### 3.1 项目结构规划

在 KrKr2 的 CMake 项目中，为 Flutter 嵌入创建独立的 CMake 模块：

```
krkr2/
├── CMakeLists.txt                 ← 顶层 CMake
├── cmake/
│   ├── FlutterEngine.cmake        ← Flutter engine 查找/下载逻辑
│   └── FlutterAssets.cmake        ← Flutter asset 打包逻辑
├── cpp/
│   └── core/
│       └── environ/
│           └── ui/
│               ├── abstract/      ← 已有的抽象接口
│               └── flutter/       ← 新建：Flutter 后端实现
│                   ├── FlutterUIManager.h
│                   ├── FlutterUIManager.cpp
│                   ├── FlutterEngineWrapper.h
│                   └── FlutterEngineWrapper.cpp
└── flutter_module/                ← Flutter 项目（独立 pubspec.yaml）
    ├── pubspec.yaml
    ├── lib/
    │   └── main.dart
    └── CMakeLists.txt             ← Flutter build 集成
```

### 3.2 FlutterEngine.cmake — 核心查找模块

```cmake
# cmake/FlutterEngine.cmake
# 查找或下载 Flutter Engine 预编译库
# 
# 用法：
#   include(FlutterEngine)
#   target_link_libraries(your_target PRIVATE Flutter::Engine)
#
# 输入变量：
#   FLUTTER_ENGINE_DIR   — 手动指定 engine 目录（可选）
#   FLUTTER_SDK_PATH     — Flutter SDK 路径（默认从 PATH 查找）

cmake_minimum_required(VERSION 3.16)

# --- 1. 确定 Flutter SDK 路径 ---
if(NOT FLUTTER_SDK_PATH)
    find_program(FLUTTER_EXECUTABLE flutter HINTS $ENV{FLUTTER_ROOT}/bin)
    if(NOT FLUTTER_EXECUTABLE)
        message(FATAL_ERROR 
            "Flutter SDK not found. Either:\n"
            "  1. Set FLUTTER_SDK_PATH cmake variable\n"
            "  2. Add flutter to PATH\n"
            "  3. Set FLUTTER_ROOT environment variable"
        )
    endif()
    # 从 flutter 可执行文件推算 SDK 路径
    get_filename_component(FLUTTER_BIN_DIR ${FLUTTER_EXECUTABLE} DIRECTORY)
    get_filename_component(FLUTTER_SDK_PATH ${FLUTTER_BIN_DIR} DIRECTORY)
endif()

message(STATUS "Flutter SDK: ${FLUTTER_SDK_PATH}")

# --- 2. 读取 engine 版本 hash ---
file(READ "${FLUTTER_SDK_PATH}/bin/internal/engine.version" FLUTTER_ENGINE_HASH)
string(STRIP ${FLUTTER_ENGINE_HASH} FLUTTER_ENGINE_HASH)
message(STATUS "Flutter Engine hash: ${FLUTTER_ENGINE_HASH}")

# --- 3. 确定平台相关路径 ---
if(WIN32)
    set(FLUTTER_PLATFORM "windows-x64")
    set(FLUTTER_ENGINE_LIB_NAME "flutter_engine.dll.lib")
    set(FLUTTER_ENGINE_DLL_NAME "flutter_engine.dll")
elseif(APPLE)
    if(IOS)
        set(FLUTTER_PLATFORM "ios-release")
        set(FLUTTER_ENGINE_FRAMEWORK "Flutter.xcframework")
    else()
        set(FLUTTER_PLATFORM "darwin-x64")
        set(FLUTTER_ENGINE_FRAMEWORK "FlutterEmbedder.framework")
    endif()
else()
    set(FLUTTER_PLATFORM "linux-x64")
    set(FLUTTER_ENGINE_LIB_NAME "libflutter_engine.so")
endif()

# --- 4. 查找预编译 engine ---
if(NOT FLUTTER_ENGINE_DIR)
    set(FLUTTER_ENGINE_DIR 
        "${FLUTTER_SDK_PATH}/bin/cache/artifacts/engine/${FLUTTER_PLATFORM}"
    )
endif()

message(STATUS "Flutter Engine dir: ${FLUTTER_ENGINE_DIR}")

# --- 5. 创建 IMPORTED target ---
if(NOT TARGET Flutter::Engine)
    add_library(Flutter::Engine SHARED IMPORTED GLOBAL)
    
    if(WIN32)
        set_target_properties(Flutter::Engine PROPERTIES
            IMPORTED_IMPLIB "${FLUTTER_ENGINE_DIR}/${FLUTTER_ENGINE_LIB_NAME}"
            IMPORTED_LOCATION "${FLUTTER_ENGINE_DIR}/${FLUTTER_ENGINE_DLL_NAME}"
            INTERFACE_INCLUDE_DIRECTORIES "${FLUTTER_ENGINE_DIR}"
        )
    elseif(APPLE AND NOT IOS)
        set_target_properties(Flutter::Engine PROPERTIES
            IMPORTED_LOCATION 
                "${FLUTTER_ENGINE_DIR}/${FLUTTER_ENGINE_FRAMEWORK}/flutter_embedder"
            INTERFACE_INCLUDE_DIRECTORIES 
                "${FLUTTER_ENGINE_DIR}/${FLUTTER_ENGINE_FRAMEWORK}/Headers"
        )
    else()
        set_target_properties(Flutter::Engine PROPERTIES
            IMPORTED_LOCATION "${FLUTTER_ENGINE_DIR}/${FLUTTER_ENGINE_LIB_NAME}"
            INTERFACE_INCLUDE_DIRECTORIES "${FLUTTER_ENGINE_DIR}"
        )
    endif()
endif()

# --- 6. 验证头文件存在 ---
if(NOT EXISTS "${FLUTTER_ENGINE_DIR}/flutter_embedder.h" AND NOT APPLE)
    message(FATAL_ERROR 
        "flutter_embedder.h not found in ${FLUTTER_ENGINE_DIR}\n"
        "Run 'flutter devices' to trigger SDK download, or set FLUTTER_ENGINE_DIR manually."
    )
endif()

message(STATUS "Flutter::Engine target configured successfully")
```

### 3.3 顶层 CMakeLists.txt 集成

```cmake
# CMakeLists.txt（顶层，仅展示 Flutter 相关部分）
cmake_minimum_required(VERSION 3.16)
project(KrKr2)

# 加载自定义 CMake 模块
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

# 条件编译开关：是否启用 Flutter UI 后端
option(KRKR2_USE_FLUTTER_UI "Enable Flutter UI backend" OFF)

if(KRKR2_USE_FLUTTER_UI)
    include(FlutterEngine)
    
    # 添加 Flutter 后端源文件
    target_sources(krkr2 PRIVATE
        cpp/core/environ/ui/flutter/FlutterEngineWrapper.cpp
        cpp/core/environ/ui/flutter/FlutterUIManager.cpp
    )
    
    # 链接 Flutter Engine
    target_link_libraries(krkr2 PRIVATE Flutter::Engine)
    
    # 定义编译宏，让 UIBackendFactory 选择 Flutter 后端
    target_compile_definitions(krkr2 PRIVATE KRKR2_UI_BACKEND_FLUTTER=1)
    
    # 安装时复制 flutter_engine 动态库
    if(WIN32)
        install(FILES 
            "${FLUTTER_ENGINE_DIR}/flutter_engine.dll"
            DESTINATION bin
        )
    elseif(UNIX AND NOT APPLE)
        install(FILES 
            "${FLUTTER_ENGINE_DIR}/libflutter_engine.so"
            DESTINATION lib
        )
    endif()
endif()
```

### 3.4 FetchContent 方案（无 Flutter SDK 环境时）

如果 CI/CD 环境中没有 Flutter SDK，可以用 FetchContent 直接下载 engine：

```cmake
# cmake/FlutterEngineDownload.cmake
# 使用 FetchContent 下载指定版本的 Flutter Engine
# 适用于 CI/CD 环境（无 Flutter SDK）

include(FetchContent)

# 设置要下载的 engine 版本
set(FLUTTER_ENGINE_HASH "4d721f339fa4aa8caef83f66bdbb0e28b59e5c78"
    CACHE STRING "Flutter engine git hash")

# 根据平台确定下载 URL
if(WIN32)
    set(_flutter_archive "windows-x64-embedder.zip")
elseif(APPLE)
    set(_flutter_archive "darwin-x64-embedder.zip")
else()
    set(_flutter_archive "linux-x64-embedder.zip")
endif()

set(FLUTTER_ENGINE_DOWNLOAD_URL
    "https://storage.googleapis.com/flutter_infra_release/flutter/${FLUTTER_ENGINE_HASH}/${_flutter_archive}"
)

message(STATUS "Downloading Flutter Engine from: ${FLUTTER_ENGINE_DOWNLOAD_URL}")

FetchContent_Declare(
    flutter_engine
    URL ${FLUTTER_ENGINE_DOWNLOAD_URL}
    URL_HASH SHA256 ${FLUTTER_ENGINE_SHA256}  # 从 GCS 对应的 .sha256 文件获取
)

FetchContent_MakeAvailable(flutter_engine)

# flutter_engine_SOURCE_DIR 现在指向解压目录
set(FLUTTER_ENGINE_DIR ${flutter_engine_SOURCE_DIR} CACHE PATH "" FORCE)
```

---

## 四、FlutterEngineWrapper 实现

### 4.1 头文件设计

```cpp
// cpp/core/environ/ui/flutter/FlutterEngineWrapper.h
#pragma once

#include <flutter/flutter_embedder.h>
#include <functional>
#include <memory>
#include <string>
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>

namespace krkr2 {
namespace ui {
namespace flutter_backend {

/**
 * FlutterEngineWrapper — 对 Flutter Embedder API 的 RAII 封装
 *
 * 负责：
 *   1. FlutterEngine 的初始化和生命周期管理
 *   2. Task Runner 的实现（将 Flutter 任务调度到正确线程）
 *   3. OpenGL 渲染上下文的提供
 *   4. Platform Channel 的基础消息路由
 */
class FlutterEngineWrapper {
public:
    /**
     * 初始化参数
     */
    struct InitArgs {
        std::string assets_path;      ///< Flutter assets 目录路径（含 kernel_blob.bin）
        std::string icu_data_path;    ///< icudtl.dat 路径
        std::string aot_library_path; ///< AOT 编译产物路径（release 模式下必须）
        
        // OpenGL 回调：FlutterEngine 需要宿主提供 GL 上下文操作
        std::function<bool()> gl_make_current;       ///< 激活 GL 上下文
        std::function<bool()> gl_clear_current;      ///< 解除 GL 上下文
        std::function<bool()> gl_present;            ///< 呈现帧（交换缓冲区或通知纹理就绪）
        std::function<uint32_t()> gl_fbo_callback;   ///< 返回目标 FBO ID
        std::function<void*(const char*)> gl_proc_resolver; ///< 解析 GL 函数指针
        
        // Platform Channel 消息回调
        std::function<void(const std::string& channel, 
                           const uint8_t* data, size_t size)> platform_message_callback;
    };

    explicit FlutterEngineWrapper(InitArgs args);
    ~FlutterEngineWrapper();

    // 禁止拷贝，允许移动
    FlutterEngineWrapper(const FlutterEngineWrapper&) = delete;
    FlutterEngineWrapper& operator=(const FlutterEngineWrapper&) = delete;

    /**
     * 初始化并启动 FlutterEngine
     * @return true 表示成功
     */
    bool initialize();

    /**
     * 关闭并销毁 FlutterEngine
     * 调用后对象不可再使用
     */
    void shutdown();

    /** 通知 Flutter 窗口尺寸变化 */
    void sendWindowMetrics(int width, int height, double pixel_ratio);

    /** 向 Flutter 发送触摸/鼠标事件 */
    void sendPointerEvent(const FlutterPointerEvent& event);

    /** 向指定 Platform Channel 发送消息 */
    void sendPlatformMessage(const std::string& channel,
                             const uint8_t* data, size_t size,
                             FlutterDataCallback reply_callback,
                             void* user_data);

    /**
     * 驱动 Platform Task Runner 执行待处理任务
     * 必须在主线程（或 Platform Task Runner 所在线程）定期调用
     */
    void runPlatformTasks();

    /**
     * 通知 VSync 信号（每帧调用一次）
     * @param baton     Flutter 请求 VSync 时提供的令牌
     * @param frame_ns  当前帧开始时间（纳秒，从 epoch 起）
     * @param deadline_ns 下一帧 deadline（纳秒）
     */
    void onVsync(intptr_t baton, uint64_t frame_ns, uint64_t deadline_ns);

    /** 获取原生 FlutterEngine 句柄（谨慎使用） */
    FLUTTER_API_SYMBOL(FlutterEngine) getHandle() const { return engine_; }

    bool isRunning() const { return engine_ != nullptr; }

private:
    // 配置 OpenGL 渲染器
    void setupRendererConfig(FlutterRendererConfig& config);
    // 配置 ProjectArgs
    void setupProjectArgs(FlutterProjectArgs& args);
    
    // Task Runner 实现
    static bool runs_task_on_current_thread_callback(void* user_data);
    static void post_task_callback(FlutterTask task, uint64_t target_time, void* user_data);

    InitArgs args_;
    FLUTTER_API_SYMBOL(FlutterEngine) engine_ = nullptr;

    // Platform Task Runner 的任务队列
    struct PendingTask {
        FlutterTask task;
        uint64_t target_time_ns;
    };
    std::queue<PendingTask> platform_tasks_;
    std::mutex platform_tasks_mutex_;
    
    // 标记 Platform Task Runner 所在的线程 ID
    std::thread::id platform_thread_id_;
};

} // namespace flutter_backend
} // namespace ui
} // namespace krkr2
```

### 4.2 核心初始化实现

```cpp
// cpp/core/environ/ui/flutter/FlutterEngineWrapper.cpp
#include "FlutterEngineWrapper.h"
#include <cassert>
#include <chrono>
#include <stdexcept>

namespace krkr2 {
namespace ui {
namespace flutter_backend {

FlutterEngineWrapper::FlutterEngineWrapper(InitArgs args)
    : args_(std::move(args))
    , platform_thread_id_(std::this_thread::get_id())  // 构造时记录线程 ID
{}

FlutterEngineWrapper::~FlutterEngineWrapper() {
    if (engine_) {
        shutdown();
    }
}

bool FlutterEngineWrapper::initialize() {
    // 1. 配置 OpenGL 渲染器
    FlutterRendererConfig renderer_config = {};
    setupRendererConfig(renderer_config);

    // 2. 配置 Project Args
    FlutterProjectArgs project_args = {};
    setupProjectArgs(project_args);

    // 3. 初始化 FlutterEngine
    FlutterEngineResult result = FlutterEngineInitialize(
        FLUTTER_ENGINE_VERSION,   // 版本号必须精确匹配
        &renderer_config,
        &project_args,
        this,                     // user_data，在回调中还原为 this
        &engine_
    );

    if (result != kSuccess) {
        // 常见错误码：
        //   kInvalidLibraryVersion — engine 版本不匹配，检查 FLUTTER_ENGINE_VERSION
        //   kInvalidArguments      — ProjectArgs 字段有误
        //   kInternalInconsistency — 内部错误，通常是 AOT 库加载失败
        const char* error_str = "unknown";
        switch (result) {
            case kInvalidLibraryVersion:  error_str = "InvalidLibraryVersion"; break;
            case kInvalidArguments:       error_str = "InvalidArguments"; break;
            case kInternalInconsistency:  error_str = "InternalInconsistency"; break;
        }
        // 记录错误（此处假设 KrKr2 有 KLOG 宏）
        // KLOGE("FlutterEngine", "FlutterEngineInitialize failed: %s", error_str);
        return false;
    }

    // 4. 运行 FlutterEngine（启动 Dart VM）
    result = FlutterEngineRunInitialized(engine_);
    if (result != kSuccess) {
        FlutterEngineShutdown(engine_);
        engine_ = nullptr;
        return false;
    }

    return true;
}

void FlutterEngineWrapper::setupRendererConfig(FlutterRendererConfig& config) {
    config.type = kOpenGL;
    config.open_gl.struct_size = sizeof(FlutterOpenGLRendererConfig);
    
    // make_current：FlutterEngine 在需要 GL 操作时调用
    config.open_gl.make_current = [](void* user_data) -> bool {
        auto* self = static_cast<FlutterEngineWrapper*>(user_data);
        return self->args_.gl_make_current ? self->args_.gl_make_current() : false;
    };
    
    // clear_current：完成 GL 操作后清理上下文
    config.open_gl.clear_current = [](void* user_data) -> bool {
        auto* self = static_cast<FlutterEngineWrapper*>(user_data);
        return self->args_.gl_clear_current ? self->args_.gl_clear_current() : false;
    };
    
    // present：Flutter 完成一帧渲染后调用，通知宿主帧已就绪
    config.open_gl.present = [](void* user_data) -> bool {
        auto* self = static_cast<FlutterEngineWrapper*>(user_data);
        return self->args_.gl_present ? self->args_.gl_present() : false;
    };
    
    // fbo_callback：返回 Flutter 应该渲染到的 FBO ID
    // 返回 0 表示默认帧缓冲，非零表示离屏 FBO（用于纹理共享）
    config.open_gl.fbo_callback = [](void* user_data) -> uint32_t {
        auto* self = static_cast<FlutterEngineWrapper*>(user_data);
        return self->args_.gl_fbo_callback ? self->args_.gl_fbo_callback() : 0;
    };
    
    // gl_proc_resolver：解析 OpenGL 扩展函数
    // 必须实现，否则 Flutter 无法找到 glFramebufferTexture2D 等 ES 3.0 函数
    config.open_gl.gl_proc_resolver = [](void* user_data, const char* name) -> void* {
        auto* self = static_cast<FlutterEngineWrapper*>(user_data);
        return self->args_.gl_proc_resolver ? self->args_.gl_proc_resolver(name) : nullptr;
    };
}

void FlutterEngineWrapper::setupProjectArgs(FlutterProjectArgs& args) {
    args.struct_size = sizeof(FlutterProjectArgs);
    args.assets_path = args_.assets_path.c_str();
    args.icu_data_path = args_.icu_data_path.c_str();
    
    // AOT 模式：release build 必须提供 AOT 库
    // Debug 模式：可以省略，使用 JIT 模式
#if defined(NDEBUG)
    if (!args_.aot_library_path.empty()) {
        args.aot_library_path = args_.aot_library_path.c_str();
    }
#endif
    
    // Platform Message 回调：将 Dart → C++ 的 Channel 消息路由到宿主
    args.platform_message_callback = [](const FlutterPlatformMessage* message,
                                         void* user_data) {
        auto* self = static_cast<FlutterEngineWrapper*>(user_data);
        if (self->args_.platform_message_callback) {
            self->args_.platform_message_callback(
                message->channel,
                message->message,
                message->message_size
            );
        }
    };
    
    // 配置 Platform Task Runner（绑定到当前线程）
    FlutterTaskRunnerDescription platform_task_runner = {};
    platform_task_runner.struct_size = sizeof(FlutterTaskRunnerDescription);
    platform_task_runner.user_data = this;
    
    // 检查当前线程是否是 Platform Task Runner 线程
    platform_task_runner.runs_task_on_current_thread_callback = 
        &FlutterEngineWrapper::runs_task_on_current_thread_callback;
    
    // 投递任务到 Platform Task Runner 的队列
    platform_task_runner.post_task_callback = 
        &FlutterEngineWrapper::post_task_callback;
    
    // 需要持久保存，task runner 生命周期需覆盖 engine 生命周期
    // 此处简化处理：直接使用 static（实际项目中应使用成员变量）
    static FlutterCustomTaskRunners task_runners = {};
    task_runners.struct_size = sizeof(FlutterCustomTaskRunners);
    task_runners.platform_task_runner = &platform_task_runner;
    args.custom_task_runners = &task_runners;
}

// --- Task Runner 静态回调 ---

bool FlutterEngineWrapper::runs_task_on_current_thread_callback(void* user_data) {
    auto* self = static_cast<FlutterEngineWrapper*>(user_data);
    return std::this_thread::get_id() == self->platform_thread_id_;
}

void FlutterEngineWrapper::post_task_callback(FlutterTask task, 
                                               uint64_t target_time_ns, 
                                               void* user_data) {
    auto* self = static_cast<FlutterEngineWrapper*>(user_data);
    std::lock_guard<std::mutex> lock(self->platform_tasks_mutex_);
    self->platform_tasks_.push({task, target_time_ns});
}

void FlutterEngineWrapper::runPlatformTasks() {
    // 在主循环每帧调用，处理积累的 Platform Task Runner 任务
    auto now_ns = static_cast<uint64_t>(
        std::chrono::duration_cast<std::chrono::nanoseconds>(
            std::chrono::steady_clock::now().time_since_epoch()
        ).count()
    );
    
    std::queue<PendingTask> ready_tasks;
    {
        std::lock_guard<std::mutex> lock(platform_tasks_mutex_);
        while (!platform_tasks_.empty()) {
            auto& front = platform_tasks_.front();
            if (front.target_time_ns <= now_ns) {
                ready_tasks.push(front);
                platform_tasks_.pop();
            } else {
                break;  // 任务队列按时间排序，后续任务更晚
            }
        }
    }
    
    while (!ready_tasks.empty()) {
        FlutterEngineRunTask(engine_, &ready_tasks.front().task);
        ready_tasks.pop();
    }
}

// --- 公开 API 实现 ---

void FlutterEngineWrapper::sendWindowMetrics(int width, int height, double pixel_ratio) {
    if (!engine_) return;
    
    FlutterWindowMetricsEvent metrics = {};
    metrics.struct_size = sizeof(FlutterWindowMetricsEvent);
    metrics.width = static_cast<size_t>(width);
    metrics.height = static_cast<size_t>(height);
    metrics.pixel_ratio = pixel_ratio;
    
    FlutterEngineSendWindowMetricsEvent(engine_, &metrics);
}

void FlutterEngineWrapper::sendPointerEvent(const FlutterPointerEvent& event) {
    if (!engine_) return;
    FlutterEngineSendPointerEvent(engine_, &event, 1);
}

void FlutterEngineWrapper::onVsync(intptr_t baton, 
                                    uint64_t frame_ns, 
                                    uint64_t deadline_ns) {
    if (!engine_) return;
    FlutterEngineOnVsync(engine_, baton, frame_ns, deadline_ns);
}

void FlutterEngineWrapper::shutdown() {
    if (engine_) {
        FlutterEngineShutdown(engine_);
        engine_ = nullptr;
    }
}

} // namespace flutter_backend
} // namespace ui
} // namespace krkr2
```

---

## 五、平台差异处理

### 5.1 Windows 特殊处理

Windows 平台需要注意 DLL 加载路径问题：

```cpp
// Windows 平台：确保 flutter_engine.dll 在可执行文件同目录
// 可以在程序启动时动态设置 DLL 搜索路径

#ifdef _WIN32
#include <windows.h>

void setupFlutterDllPath(const std::wstring& flutter_engine_dir) {
    // 将 flutter_engine.dll 所在目录添加到 DLL 搜索路径
    // Windows 8.1+ 推荐使用 AddDllDirectory 代替 SetDllDirectory
    if (!AddDllDirectory(flutter_engine_dir.c_str())) {
        // 降级方案
        SetDllDirectoryW(flutter_engine_dir.c_str());
    }
}
#endif
```

Windows 的 GL 函数解析需通过 `wglGetProcAddress`：

```cpp
#ifdef _WIN32
#include <GL/gl.h>
#include <wingdi.h>

void* resolveGLProc_Windows(const char* name) {
    void* proc = (void*)wglGetProcAddress(name);
    if (!proc) {
        // wglGetProcAddress 对核心函数（GL 1.1 及以下）返回 nullptr
        // 需要回退到 GetProcAddress 从 opengl32.dll 获取
        HMODULE gl_module = GetModuleHandleA("opengl32.dll");
        if (gl_module) {
            proc = (void*)GetProcAddress(gl_module, name);
        }
    }
    return proc;
}
#endif
```

### 5.2 Linux/macOS GL 函数解析

```cpp
#if defined(__linux__)
#include <GL/glx.h>
void* resolveGLProc_Linux(const char* name) {
    return (void*)glXGetProcAddressARB(
        reinterpret_cast<const GLubyte*>(name)
    );
}
#elif defined(__APPLE__) && !defined(TARGET_OS_IPHONE)
#include <OpenGL/gl.h>
#include <dlfcn.h>

static void* gl_handle = nullptr;

void* resolveGLProc_macOS(const char* name) {
    if (!gl_handle) {
        gl_handle = dlopen(
            "/System/Library/Frameworks/OpenGL.framework/Versions/Current/OpenGL",
            RTLD_NOW
        );
    }
    return gl_handle ? dlsym(gl_handle, name) : nullptr;
}
#endif
```

### 5.3 Android 集成差异

Android 平台上，FlutterEngine 以 Java 类的形式提供：

```java
// Android：通过 Java API 集成 FlutterEngine
// 这部分代码写在 Android 模块的 Java/Kotlin 层

import io.flutter.embedding.engine.FlutterEngine;
import io.flutter.embedding.engine.FlutterEngineCache;
import io.flutter.embedding.engine.dart.DartExecutor;

public class KrKr2MainActivity extends AppCompatActivity {
    
    private static final String FLUTTER_ENGINE_ID = "krkr2_flutter_engine";
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        // 预热 FlutterEngine（避免首次显示时的初始化延迟）
        FlutterEngine flutterEngine = new FlutterEngine(this);
        flutterEngine.getDartExecutor().executeDartEntrypoint(
            DartExecutor.DartEntrypoint.createDefault()
        );
        
        // 缓存 engine，供 FlutterFragment 复用
        FlutterEngineCache.getInstance().put(FLUTTER_ENGINE_ID, flutterEngine);
    }
}
```

Android 上嵌入 Flutter 视图（作为 Cocos2d 的覆盖层）：

```java
// 在 Cocos2d Activity 的布局上叠加 FlutterView
import io.flutter.embedding.android.FlutterFragment;
import io.flutter.embedding.android.FlutterView;

// 方案 A：FlutterFragment（推荐，生命周期自动管理）
FlutterFragment flutterFragment = FlutterFragment
    .withCachedEngine(FLUTTER_ENGINE_ID)
    .build();

getSupportFragmentManager()
    .beginTransaction()
    .add(R.id.flutter_container, flutterFragment, "flutter_fragment")
    .commit();

// 方案 B：FlutterSurfaceView（更灵活，适合自定义布局）
FlutterView flutterView = new FlutterView(this, 
    FlutterView.RenderMode.surface);
flutterView.attachToFlutterEngine(flutterEngine);
((FrameLayout) findViewById(R.id.flutter_container)).addView(flutterView);
```

---

## 六、动手实践

### 实践 6.1：验证 Flutter Engine 加载

在开始集成之前，先写一个最小验证程序确认 engine 可以正常加载：

```cpp
// tools/verify_flutter_engine/main.cpp
// 编译命令（Linux）：
//   g++ -std=c++17 main.cpp -L/path/to/engine -lflutter_engine \
//       -I/path/to/engine -o verify_flutter_engine

#include <flutter/flutter_embedder.h>
#include <cstdio>

int main() {
    printf("Flutter Embedder API version: %zu\n", FLUTTER_ENGINE_VERSION);
    
    // 尝试初始化一个最小 engine（会因缺少 assets 而失败，但能验证库加载）
    FlutterRendererConfig config = {};
    config.type = kOpenGL;
    config.open_gl.struct_size = sizeof(FlutterOpenGLRendererConfig);
    config.open_gl.make_current = [](void*) -> bool { return false; };
    config.open_gl.clear_current = [](void*) -> bool { return false; };
    config.open_gl.present = [](void*) -> bool { return false; };
    config.open_gl.fbo_callback = [](void*) -> uint32_t { return 0; };
    
    FlutterProjectArgs args = {};
    args.struct_size = sizeof(FlutterProjectArgs);
    args.assets_path = "/nonexistent/path";
    args.icu_data_path = "/nonexistent/icudtl.dat";
    
    FLUTTER_API_SYMBOL(FlutterEngine) engine = nullptr;
    FlutterEngineResult result = FlutterEngineInitialize(
        FLUTTER_ENGINE_VERSION, &config, &args, nullptr, &engine
    );
    
    printf("FlutterEngineInitialize result: %d\n", result);
    // 期望输出：kInvalidArguments (1) 或 kInternalInconsistency (3)
    // 如果输出 kInvalidLibraryVersion (2)，说明版本不匹配，需要重新下载正确版本的 engine
    
    if (result != kInvalidLibraryVersion) {
        printf("Engine library loaded successfully!\n");
    } else {
        printf("ERROR: Engine version mismatch. "
               "Expected FLUTTER_ENGINE_VERSION=%zu\n", FLUTTER_ENGINE_VERSION);
    }
    
    return (result != kInvalidLibraryVersion) ? 0 : 1;
}
```

### 实践 6.2：构建 Flutter Module 的 assets

FlutterEngine 需要 Flutter Module 编译出的 assets 才能运行 Dart 代码：

```bash
# 在 flutter_module/ 目录下
cd krkr2/flutter_module

# Debug 模式（含 kernel_blob.bin，支持热重载）
flutter build bundle
# 产物路径：build/flutter_assets/

# Release 模式（含 AOT 编译的 app.so）
flutter build bundle --release

# 查看产物结构
ls build/flutter_assets/
# AssetManifest.json
# FontManifest.json
# NOTICES.Z
# fonts/MaterialIcons-Regular.otf
# kernel_blob.bin     ← Debug 模式专用，Release 中不存在
# vm_snapshot_data
# isolate_snapshot_data

# Release 模式 AOT 库（Android）
ls build/app/outputs/flutter-apk/  # 包含 libapp.so
```

---

## 对照项目源码

```
krkr2/
├── cmake/
│   └── FlutterEngine.cmake         ← 本节实现的 CMake 模块
├── cpp/core/environ/ui/flutter/
│   ├── FlutterEngineWrapper.h      ← FlutterEngine RAII 封装头文件
│   └── FlutterEngineWrapper.cpp    ← FlutterEngine RAII 封装实现
└── flutter_module/
    ├── pubspec.yaml                ← Flutter 模块配置
    └── lib/main.dart               ← Dart 入口
```

---

## 本节小结

- Flutter 的三层架构（Framework / Engine / Embedder）中，KrKr2 需要实现 Embedder 层
- Embedder API 以 C 函数形式暴露，通过 `FlutterEngineInitialize` 初始化，传入渲染配置和项目参数
- `FlutterRendererConfig` 中的四个 GL 回调（make_current / clear_current / present / fbo_callback）是实现纹理共享的关键
- Platform Task Runner 必须由宿主应用实现，并在主循环中定期调用 `runPlatformTasks()`
- Windows 需要特别处理 DLL 搜索路径和 GL 函数解析；Android 使用 Java API 而非直接调用 C API

---

## 练习题与答案

**题 1：** `FlutterEngineInitialize` 返回 `kInvalidLibraryVersion` 错误的根本原因是什么？如何解决？

<details>
<summary>查看答案</summary>

**原因：** 代码中的 `FLUTTER_ENGINE_VERSION` 常量（来自 `flutter_embedder.h`）与实际链接的 `libflutter_engine.so` 的内部版本号不一致。这个常量是一个整数，每当 Embedder API 有破坏性变更时就会递增。

**解决方案：**
1. 确保 `flutter_embedder.h` 和 `libflutter_engine.so` 来自同一个 Flutter SDK 版本
2. 通过 `bin/internal/engine.version` 文件确认 engine hash，使用 `FlutterEngine.cmake` 中的版本检查逻辑
3. 如果使用 FetchContent 方案，确保下载的 zip 文件版本与 `FLUTTER_ENGINE_VERSION` 对应

</details>

---

**题 2：** 为什么 Platform Task Runner 必须绑定到 KrKr2 的主线程（而不是单独创建新线程）？

<details>
<summary>查看答案</summary>

Platform Task Runner 代表的是"平台线程"，与操作系统的 UI 线程（或主消息队列）对齐。Flutter 的很多 Platform Channel 操作必须在平台线程上执行，例如：
- 调用系统对话框
- 访问平台 UI 元素
- 某些 Android/iOS API（主线程限制）

如果绑定到独立线程，Platform Channel 调用可能触发线程检查断言，或导致 Android/iOS 平台 API 的线程安全问题。KrKr2 的主线程已经是 Cocos2d-x 的 GL 线程，也是大多数系统回调的执行线程，因此是最合适的 Platform Task Runner 宿主。

</details>

---

**题 3：** `FlutterProjectArgs.aot_library_path` 在 Debug 和 Release 构建中分别需要如何配置？

<details>
<summary>查看答案</summary>

- **Debug 构建**：`aot_library_path` 可以为空或 `nullptr`。Flutter 使用 JIT（Just-In-Time）模式，Dart 代码以字节码形式存储在 `assets/kernel_blob.bin` 中，运行时动态编译。此时支持热重载（hot reload）。

- **Release 构建**：必须提供 AOT 编译产物路径：
  - Android：`libapp.so`（由 `flutter build apk --release` 生成）
  - Linux/Windows：`libapp.so` / `app.dll`（由 `flutter build bundle --release` 生成）
  - iOS：`App.framework/App`（已嵌入 IPA 中）
  
  Release 模式下 `kernel_blob.bin` 不存在，如果路径配置错误会得到 `kInternalInconsistency` 错误。

</details>

---

## 下一步

本节完成了 FlutterEngine 的集成和 CMake 配置。下一节 [02-PlatformChannel实现UI交互](02-PlatformChannel实现UI交互.md) 将实现 Dart ↔ C++ 双向通信机制，让 Flutter UI 能够调用 KrKr2 的 C++ 功能，并接收来自 C++ 的事件通知。
