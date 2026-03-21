# FlutterEngine API 与嵌入原理

> **所属模块：** P12-现代跨平台 UI（Flutter / Compose）
> **前置知识：** [01-方案对比/01-主流跨平台UI框架总览](../01-方案对比/01-主流跨平台UI框架总览.md)、[01-方案对比/02-渲染方式与C++互操作能力对比](../01-方案对比/02-渲染方式与C++互操作能力对比.md)
> **预计阅读时间：** 45 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：
1. 理解 Flutter Engine 的整体架构和各层职责
2. 掌握 FlutterEngine Embedder API 的核心数据结构与函数
3. 在 C++ 宿主程序中初始化、运行和销毁 Flutter Engine 实例
4. 管理 Flutter Engine 的生命周期（启动、暂停、恢复、销毁）
5. 理解 Flutter 渲染后端的配置（OpenGL / Vulkan / Metal / Software）
6. 将 Flutter Engine 嵌入到现有 C++ 渲染循环中（如 Cocos2d-x）

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| Flutter Engine | Flutter Engine | Flutter 框架的 C/C++ 核心运行时，负责 Dart VM、渲染、文本排版和插件分发 |
| Embedder API | Embedder API | Flutter 提供的一组 C 语言函数接口，允许宿主程序（host）将 Flutter Engine 嵌入到任意平台 |
| AOT Snapshot | Ahead-Of-Time Snapshot | Dart 代码预编译后的二进制机器码快照，Release 模式下用它代替 JIT 解释执行以提升性能 |
| ICU 数据 | ICU Data | 国际化组件库（International Components for Unicode）的数据文件，Flutter 用它处理文本排版和本地化 |
| 渲染后端 | Renderer Backend | Flutter Engine 实际执行 GPU 绘制的底层实现，可选 OpenGL ES、Vulkan、Metal 或纯 CPU 软件渲染 |
| 任务运行器 | Task Runner | Flutter Engine 内部的线程调度器，分为 Platform、UI、Render、IO 四个专用运行器 |
| 平台视图 | Platform View | 将原生平台控件（如 Android View、iOS UIView）嵌入 Flutter 渲染树的机制 |
| Compositor | Compositor | Flutter Engine 的合成器，负责将多个渲染层（Flutter 层 + Platform View 层）合并为最终画面 |
| 像素比 | Device Pixel Ratio | 逻辑像素与物理像素的比值，高 DPI 屏幕上此值大于 1（如 Retina 屏为 2.0 或 3.0） |

## Flutter Engine 架构总览

### 什么是 Flutter Engine

Flutter Engine 是 Flutter 框架的底层运行时核心。当我们说"Flutter 应用"时，实际上涉及三个层次：

1. **Framework 层**（Dart）：Material / Cupertino 组件库、Widget 树、渲染管线的 Dart 部分
2. **Engine 层**（C/C++）：Dart VM 运行时、Skia 渲染引擎、文本排版（libTxt / HarfBuzz / ICU）、Platform Channel 通信
3. **Embedder 层**（平台相关）：将 Engine 嵌入到具体操作系统的胶水代码，负责窗口管理、输入事件转发、GPU 上下文创建

对于 KrKr2 这样的已有 C++ 项目，我们关注的重点是 **Engine 层** 和 **Embedder 层**——如何把 Flutter Engine 作为一个"渲染组件"嵌入到 Cocos2d-x 的渲染循环中。

### 架构分层图

```
┌─────────────────────────────────────────────────────┐
│                   Flutter Framework                  │
│  ┌─────────┐  ┌──────────┐  ┌───────────────────┐  │
│  │ Material │  │ Cupertino│  │  Widgets / State  │  │
│  └────┬────┘  └────┬─────┘  └────────┬──────────┘  │
│       └─────────────┼────────────────┘              │
│              ┌──────┴───────┐                       │
│              │ Rendering Lib│ (Dart)                 │
│              └──────┬───────┘                       │
├─────────────────────┼───────────────────────────────┤
│                     │  Flutter Engine (C/C++)        │
│  ┌─────────┐  ┌────┴────┐  ┌──────────┐  ┌──────┐ │
│  │ Dart VM │  │  Skia   │  │ libTxt   │  │ I/O  │ │
│  │(Runtime)│  │(2D 渲染)│  │(文本排版)│  │(异步)│ │
│  └────┬────┘  └────┬────┘  └────┬─────┘  └──┬───┘ │
│       └─────────────┼───────────┼────────────┘     │
│              ┌──────┴───────────┴──────┐           │
│              │     Embedder API (C)    │           │
│              │  flutter_engine.h       │           │
│              └──────────┬─────────────┘            │
├─────────────────────────┼──────────────────────────┤
│                         │  Embedder / 宿主程序      │
│  ┌──────────┐  ┌───────┴───────┐  ┌────────────┐  │
│  │ 窗口管理 │  │ GPU 上下文创建 │  │ 输入事件   │  │
│  │(GLFW/SDL)│  │(EGL/WGL/Metal)│  │ 转发       │  │
│  └──────────┘  └───────────────┘  └────────────┘  │
└─────────────────────────────────────────────────────┘
```

这张图展示了三层的职责边界。Embedder API（`flutter_engine.h`）是我们与 Flutter Engine 交互的唯一接口，它是一组纯 C 函数，不依赖任何 C++ 特性，这使得它可以被任何支持 C ABI 的语言调用。

### 四大任务运行器

Flutter Engine 内部使用四个专用线程（Task Runner）来隔离不同类型的工作，这是理解嵌入行为的关键：

| 运行器 | 英文名 | 职责 | 线程要求 |
|--------|--------|------|----------|
| 平台运行器 | Platform Runner | 处理平台事件（触摸、键盘、生命周期）、Plugin 调用 | **必须**是创建 Engine 的线程（通常是主线程） |
| UI 运行器 | UI Runner | 运行 Dart 代码、构建 Widget 树、执行布局计算 | 专用线程，不可阻塞 |
| 渲染运行器 | Render Runner | 执行 Skia 渲染命令、管理 GPU 资源、合成最终画面 | 专用线程，持有 GPU 上下文 |
| IO 运行器 | IO Runner | 异步 I/O 操作（图片解码、文件读取、网络） | 专用线程，可多个 |

```
主线程 (Platform Runner)
  │
  ├──► UI Runner ──► Dart VM 执行
  │                   │
  │                   ▼
  │              Widget Tree → Element Tree → RenderObject Tree
  │                   │
  │                   ▼ (提交渲染指令)
  │
  ├──► Render Runner ──► Skia 绘制 → GPU 提交
  │
  └──► IO Runner ──► 图片解码 / 文件读写
```

**关键约束**：Platform Runner 必须与宿主应用的主线程一致。对于 KrKr2 而言，这意味着 Flutter Engine 的 Platform Runner 必须运行在 Cocos2d-x 的主线程（即 `Director::getInstance()->getScheduler()` 所在的线程）上。

## Embedder API 核心数据结构

Flutter Engine 的 Embedder API 定义在一个单独的头文件 `flutter_engine.h`（有时也叫 `embedder.h`）中。这个头文件是纯 C 接口，总共约 2000 行，定义了所有嵌入所需的结构体和函数。下面逐一讲解最重要的数据结构。

### FlutterProjectArgs — 项目参数

`FlutterProjectArgs` 是配置 Flutter Engine 实例的核心结构体，告诉 Engine 去哪找资源、怎么渲染、怎么与宿主通信。这个结构体字段非常多，我们按功能分组讲解：

```c
// flutter_engine.h 中的核心配置结构体（简化展示关键字段）
typedef struct {
    // ===== 基础配置 =====
    size_t struct_size;              // 结构体大小，用于版本兼容
    const char* assets_path;         // Flutter 资源目录路径（含 kernel_blob.bin 等）
    const char* icu_data_path;       // ICU 数据文件路径（icudtl.dat）

    // ===== AOT 编译产物（Release 模式）=====
    const char* aot_library_path;    // AOT 快照动态库路径（libapp.so）

    // ===== Dart 入口 =====
    const char* custom_dart_entrypoint; // 自定义 Dart 入口函数名（默认 "main"）
    int dart_entrypoint_argc;        // 传给 Dart main() 的参数个数
    const char* const* dart_entrypoint_argv; // 参数值数组

    // ===== 任务运行器 =====
    const FlutterCustomTaskRunners* custom_task_runners; // 自定义线程调度

    // ===== 渲染回调 =====
    const FlutterRendererConfig* renderer_config; // 渲染后端配置
    const FlutterCompositor* compositor;          // 合成器配置

    // ===== 平台消息 =====
    FlutterPlatformMessageCallback platform_message_callback; // Platform Channel 回调

    // ===== 语义化（无障碍访问）=====
    FlutterUpdateSemanticsCallback2 update_semantics_callback2;

    // ===== 日志 =====
    FlutterLogMessageCallback log_message_callback; // Dart print() 输出回调
} FlutterProjectArgs;
```

**关键字段解释**：

- **`struct_size`**：必须设置为 `sizeof(FlutterProjectArgs)`。Flutter Engine 通过这个值判断宿主编译时使用的 API 版本——如果宿主是用旧版头文件编译的，`struct_size` 会比新版小，Engine 就知道后面的新字段不存在，从而保持向后兼容。
- **`assets_path`**：指向 `flutter_assets` 目录，里面包含 Dart 编译产物（Debug 模式下是 `kernel_blob.bin` 内核快照，Release 模式下是 AOT 快照文件）和字体、图片等资源。
- **`icu_data_path`**：ICU 数据文件 `icudtl.dat` 的路径。Flutter 用它处理 Unicode 文本排版、日期格式化、双向文本（BiDi）等国际化功能。缺少此文件 Engine 将无法启动。
- **`custom_task_runners`**：允许宿主提供自定义的线程运行器。如果不设置，Engine 会自己创建线程。对于嵌入到 Cocos2d-x 的场景，我们需要自定义 Platform Runner 让它跑在 Cocos2d-x 的主线程上。

### FlutterRendererConfig — 渲染后端配置

Flutter Engine 支持多种渲染后端，通过 `FlutterRendererConfig` 告诉 Engine 使用哪种：

```c
typedef enum {
    kOpenGL,    // OpenGL / OpenGL ES
    kSoftware,  // CPU 软件渲染
    kMetal,     // Apple Metal（仅 macOS / iOS）
    kVulkan,    // Vulkan
} FlutterRendererType;

typedef struct {
    FlutterRendererType type;       // 选择渲染后端类型
    union {
        FlutterOpenGLRendererConfig open_gl;     // OpenGL 配置
        FlutterSoftwareRendererConfig software;  // 软件渲染配置
        FlutterMetalRendererConfig metal;        // Metal 配置
        FlutterVulkanRendererConfig vulkan;      // Vulkan 配置
    };
} FlutterRendererConfig;
```

对于 KrKr2 项目，因为已有的渲染管线基于 OpenGL ES（通过 Cocos2d-x），最自然的选择是 `kOpenGL`。下面是 OpenGL 配置的核心回调：

```c
typedef struct {
    size_t struct_size;

    // Engine 需要时调用：让宿主切换 GL 上下文为当前
    BoolCallback make_current;

    // 渲染完成后调用：让宿主执行 SwapBuffers
    BoolCallback clear_current;

    // 获取当前 FBO 编号（Engine 往这个 FBO 上绘制）
    UIntCallback fbo_callback;

    // 使资源上下文成为当前（用于 IO Runner 的纹理上传）
    BoolCallback make_resource_current;

    // 获取 GL 函数指针（类似 eglGetProcAddress）
    FlutterGLProcResolver gl_proc_resolver;

    // 外部纹理回调（用于 Texture Widget）
    FlutterGLExternalTextureCallback gl_external_texture_frame_callback;
} FlutterOpenGLRendererConfig;
```

每个回调都会收到一个 `void* user_data` 参数，宿主可以用它传递自己的上下文指针（如 EGL Display、EGL Surface 等）。这些回调会在 Render Runner 线程上被调用，因此必须是线程安全的。

### 各渲染后端对比

| 后端 | 适用平台 | 优势 | KrKr2 适用性 |
|------|----------|------|-------------|
| OpenGL ES | 全平台 | 与 Cocos2d-x 共享 GL 上下文最简单 | ⭐ **首选** |
| Vulkan | Android 7+、Linux、Windows | 性能更好，多线程渲染 | 中期可选 |
| Metal | macOS / iOS | Apple 平台原生，性能最佳 | macOS 可选 |
| Software | 全平台 | 无需 GPU，调试用 | 仅调试/测试 |

### FlutterEngine 句柄与生命周期函数

Embedder API 的核心函数围绕 `FlutterEngine` 不透明句柄展开：

```c
// FlutterEngine 是一个不透明指针（opaque handle）
// 你不能访问其内部字段，只能通过 API 函数操作它
typedef struct _FlutterEngine* FLUTTER_API_SYMBOL(FlutterEngine);

// ===== 生命周期函数 =====

// 创建并初始化 Engine（但不运行 Dart 代码）
FlutterEngineResult FlutterEngineInitialize(
    size_t version,                    // FLUTTER_ENGINE_VERSION 宏
    const FlutterRendererConfig* config, // 渲染后端配置
    const FlutterProjectArgs* args,    // 项目参数
    void* user_data,                   // 透传给所有回调的用户数据指针
    FLUTTER_API_SYMBOL(FlutterEngine)* engine_out  // 输出：Engine 句柄
);

// 开始运行 Dart 代码（调用 main() 或自定义入口）
FlutterEngineResult FlutterEngineRunInitialized(
    FLUTTER_API_SYMBOL(FlutterEngine) engine
);

// 组合版本：Initialize + RunInitialized 一步完成
FlutterEngineResult FlutterEngineRun(
    size_t version,
    const FlutterRendererConfig* config,
    const FlutterProjectArgs* args,
    void* user_data,
    FLUTTER_API_SYMBOL(FlutterEngine)* engine_out
);

// 请求 Engine 关闭（异步，会清理 Dart isolate）
FlutterEngineResult FlutterEngineDeinitialize(
    FLUTTER_API_SYMBOL(FlutterEngine) engine
);

// 销毁 Engine 并释放所有资源
FlutterEngineResult FlutterEngineShutdown(
    FLUTTER_API_SYMBOL(FlutterEngine) engine
);
```

**生命周期状态机**：

```
                FlutterEngineInitialize()
 [未创建] ──────────────────────────────► [已初始化]
                                              │
                FlutterEngineRunInitialized()  │
                                              ▼
                                          [运行中]
                                              │
                FlutterEngineDeinitialize()    │
                                              ▼
                                          [已停止]
                                              │
                FlutterEngineShutdown()        │
                                              ▼
                                          [已销毁]
```

**两阶段启动 vs 一步启动**：

- `FlutterEngineRun()` = `FlutterEngineInitialize()` + `FlutterEngineRunInitialized()`，适合简单场景
- 两阶段启动的优势：在 `Initialize` 和 `RunInitialized` 之间可以做额外配置（比如注册 Platform Channel、预加载纹理），这对嵌入到复杂宿主非常有用

## 在 C++ 宿主中初始化 Flutter Engine

### 示例 1：最小化 Flutter Engine 嵌入（GLFW + OpenGL）

这是一个完整的、可编译运行的最小示例，展示如何在 GLFW（一个轻量级跨平台窗口库）窗口中嵌入 Flutter Engine：

```cpp
// minimal_flutter_host.cpp
// 编译要求：需要 flutter_engine 动态库 + GLFW3 + OpenGL
#include <flutter_embedder.h>  // Flutter Embedder API 头文件
#include <GLFW/glfw3.h>        // GLFW 窗口管理
#include <cassert>
#include <iostream>
#include <string>

// 全局状态（实际项目中应封装为类）
struct HostState {
    GLFWwindow* window = nullptr;         // GLFW 窗口句柄
    FlutterEngine engine = nullptr;       // Flutter Engine 句柄
    double pixel_ratio = 1.0;             // 设备像素比
    int window_width = 800;               // 窗口宽度（逻辑像素）
    int window_height = 600;              // 窗口高度（逻辑像素）
};

static HostState g_state;

// ===== OpenGL 渲染回调 =====

// 将 GL 上下文设为当前（Engine 渲染前调用）
bool OnMakeCurrent(void* user_data) {
    glfwMakeContextCurrent(g_state.window);
    return true;
}

// 清除当前 GL 上下文（Engine 渲染后调用）
bool OnClearCurrent(void* user_data) {
    glfwMakeContextCurrent(nullptr);
    return true;
}

// 告诉 Engine 绘制到哪个 FBO（0 = 默认帧缓冲，即屏幕）
uint32_t OnGetFBO(void* user_data) {
    return 0;  // 直接绘制到窗口的默认帧缓冲
}

// Engine 渲染完一帧后调用：执行缓冲区交换，显示画面
bool OnPresent(void* user_data) {
    glfwSwapBuffers(g_state.window);
    return true;
}

// 获取 GL 函数地址（Engine 内部的 Skia 需要动态加载 GL 函数）
void* OnGetProcAddress(void* user_data, const char* name) {
    return reinterpret_cast<void*>(glfwGetProcAddress(name));
}

// ===== 窗口事件回调 =====

// 窗口大小变化时通知 Engine 更新视口
void OnWindowResize(GLFWwindow* window, int width, int height) {
    g_state.window_width = width;
    g_state.window_height = height;

    // 构造窗口度量信息
    FlutterWindowMetricsEvent event = {};
    event.struct_size = sizeof(FlutterWindowMetricsEvent);
    event.width = static_cast<size_t>(width * g_state.pixel_ratio);
    event.height = static_cast<size_t>(height * g_state.pixel_ratio);
    event.pixel_ratio = g_state.pixel_ratio;

    // 发送给 Engine，触发重新布局和绘制
    FlutterEngineSendWindowMetricsEvent(g_state.engine, &event);
}

int main(int argc, const char* argv[]) {
    // 第 1 步：初始化 GLFW 窗口
    if (!glfwInit()) {
        std::cerr << "GLFW 初始化失败" << std::endl;
        return -1;
    }

    // 设置 OpenGL 版本（Flutter 需要 OpenGL 3.2+ Core Profile）
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 2);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
    #ifdef __APPLE__
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);  // macOS 必须
    #endif

    g_state.window = glfwCreateWindow(
        g_state.window_width, g_state.window_height,
        "Flutter Embedded Demo", nullptr, nullptr
    );
    if (!g_state.window) {
        std::cerr << "窗口创建失败" << std::endl;
        glfwTerminate();
        return -1;
    }

    // 计算设备像素比（高 DPI 屏幕支持）
    int fb_width, fb_height;
    glfwGetFramebufferSize(g_state.window, &fb_width, &fb_height);
    g_state.pixel_ratio = static_cast<double>(fb_width)
                        / static_cast<double>(g_state.window_width);

    // 注册窗口大小变化回调
    glfwSetWindowSizeCallback(g_state.window, OnWindowResize);

    // 第 2 步：配置 Flutter 渲染后端（OpenGL）
    FlutterRendererConfig renderer_config = {};
    renderer_config.type = kOpenGL;
    renderer_config.open_gl.struct_size = sizeof(FlutterOpenGLRendererConfig);
    renderer_config.open_gl.make_current = OnMakeCurrent;
    renderer_config.open_gl.clear_current = OnClearCurrent;
    renderer_config.open_gl.present = OnPresent;
    renderer_config.open_gl.fbo_callback = OnGetFBO;
    renderer_config.open_gl.gl_proc_resolver = OnGetProcAddress;

    // 第 3 步：配置项目参数
    std::string assets_path = "./flutter_assets";   // Flutter 资源目录
    std::string icu_data = "./icudtl.dat";           // ICU 数据文件

    FlutterProjectArgs project_args = {};
    project_args.struct_size = sizeof(FlutterProjectArgs);
    project_args.assets_path = assets_path.c_str();
    project_args.icu_data_path = icu_data.c_str();

    // 第 4 步：启动 Flutter Engine
    FlutterEngineResult result = FlutterEngineRun(
        FLUTTER_ENGINE_VERSION,  // API 版本号（宏定义）
        &renderer_config,        // 渲染配置
        &project_args,           // 项目参数
        &g_state,                // user_data，透传给所有回调
        &g_state.engine          // 输出：Engine 句柄
    );

    if (result != kSuccess) {
        std::cerr << "Flutter Engine 启动失败，错误码: "
                  << result << std::endl;
        glfwDestroyWindow(g_state.window);
        glfwTerminate();
        return -1;
    }

    std::cout << "Flutter Engine 启动成功！" << std::endl;

    // 第 5 步：发送初始窗口尺寸
    OnWindowResize(g_state.window,
                   g_state.window_width,
                   g_state.window_height);

    // 第 6 步：主循环 — 处理平台事件
    while (!glfwWindowShouldClose(g_state.window)) {
        glfwWaitEvents();  // 阻塞等待事件（省电）
        // 注意：Flutter 的渲染由 Engine 内部的 Render Runner 驱动
        // 我们不需要在这里调用任何绘制函数
    }

    // 第 7 步：关闭 Engine
    FlutterEngineShutdown(g_state.engine);
    g_state.engine = nullptr;

    glfwDestroyWindow(g_state.window);
    glfwTerminate();

    std::cout << "程序正常退出" << std::endl;
    return 0;
}
```

**代码要点解析**：

1. **GL 上下文管理**：`make_current` 和 `clear_current` 回调让 Engine 能在 Render Runner 线程上获取/释放 GL 上下文。GLFW 的 `glfwMakeContextCurrent` 就是干这个的。
2. **FBO 回调**：返回 0 表示绘制到窗口默认帧缓冲。如果要实现离屏渲染（如将 Flutter 画面渲染到一个纹理上再合成到 Cocos2d-x 场景中），需要创建自定义 FBO 并返回其 ID。
3. **像素比**：高 DPI 屏幕（如 macOS Retina）上，窗口的逻辑尺寸和帧缓冲的物理像素尺寸不同。必须正确计算 `pixel_ratio` 并传给 Engine，否则渲染会模糊或错位。
4. **主循环**：用 `glfwWaitEvents()` 而非 `glfwPollEvents()`，因为 Flutter Engine 有自己的渲染循环，不需要宿主以 60fps 驱动它。事件到来时唤醒即可。

### 示例 2：CMakeLists.txt — 构建 Flutter 嵌入项目

上面的 C++ 代码需要配合 CMake 构建系统。以下是完整的 `CMakeLists.txt`：

```cmake
# CMakeLists.txt — Flutter Engine 嵌入示例构建脚本
cmake_minimum_required(VERSION 3.16)
project(flutter_embedded_demo LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ===== 查找依赖 =====

# GLFW3 — 跨平台窗口管理库
find_package(glfw3 3.3 REQUIRED)

# OpenGL — 图形渲染 API
find_package(OpenGL REQUIRED)

# ===== Flutter Engine 配置 =====
# Flutter Engine 动态库和头文件的路径
# 从 flutter/engine 仓库构建，或从 Flutter SDK 的 artifacts 中提取
set(FLUTTER_ENGINE_DIR "${CMAKE_SOURCE_DIR}/third_party/flutter_engine"
    CACHE PATH "Flutter Engine 目录（含 libflutter_engine.so 和头文件）")

# 导入 Flutter Engine 动态库
add_library(flutter_engine SHARED IMPORTED)

if(WIN32)
    # Windows: flutter_engine.dll + flutter_engine.dll.lib（导入库）
    set_target_properties(flutter_engine PROPERTIES
        IMPORTED_LOCATION "${FLUTTER_ENGINE_DIR}/flutter_engine.dll"
        IMPORTED_IMPLIB "${FLUTTER_ENGINE_DIR}/flutter_engine.dll.lib"
    )
elseif(APPLE)
    # macOS: FlutterEmbedder.framework 或 libflutter_engine.dylib
    set_target_properties(flutter_engine PROPERTIES
        IMPORTED_LOCATION "${FLUTTER_ENGINE_DIR}/libflutter_engine.dylib"
    )
else()
    # Linux: libflutter_engine.so
    set_target_properties(flutter_engine PROPERTIES
        IMPORTED_LOCATION "${FLUTTER_ENGINE_DIR}/libflutter_engine.so"
    )
endif()

# 设置 Flutter Engine 头文件搜索路径
target_include_directories(flutter_engine INTERFACE
    "${FLUTTER_ENGINE_DIR}/include"
)

# ===== 主目标 =====
add_executable(flutter_demo
    minimal_flutter_host.cpp
)

target_link_libraries(flutter_demo PRIVATE
    flutter_engine        # Flutter Engine 动态库
    glfw                  # GLFW 窗口管理
    OpenGL::GL            # OpenGL
)

# ===== 构建后：复制运行时依赖 =====

# 复制 Flutter Engine 动态库到输出目录
if(WIN32)
    add_custom_command(TARGET flutter_demo POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${FLUTTER_ENGINE_DIR}/flutter_engine.dll"
            "$<TARGET_FILE_DIR:flutter_demo>"
        COMMENT "复制 flutter_engine.dll 到输出目录"
    )
endif()

# 复制 ICU 数据文件
add_custom_command(TARGET flutter_demo POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${FLUTTER_ENGINE_DIR}/icudtl.dat"
        "$<TARGET_FILE_DIR:flutter_demo>"
    COMMENT "复制 icudtl.dat 到输出目录"
)
```

### 跨平台构建命令

```bash
# ===== Windows（MSVC）=====
cmake -B build -G "Ninja" ^
    -DCMAKE_BUILD_TYPE=Debug ^
    -DFLUTTER_ENGINE_DIR=C:/flutter_engine/windows-x64
cmake --build build

# ===== Linux（GCC / Clang）=====
cmake -B build -G "Ninja" \
    -DCMAKE_BUILD_TYPE=Debug \
    -DFLUTTER_ENGINE_DIR=/opt/flutter_engine/linux-x64
cmake --build build

# ===== macOS（Clang）=====
cmake -B build -G "Ninja" \
    -DCMAKE_BUILD_TYPE=Debug \
    -DFLUTTER_ENGINE_DIR=/opt/flutter_engine/darwin-x64
cmake --build build
```

### 获取 Flutter Engine 动态库

Flutter Engine 的预编译产物可以从 Flutter SDK 的存储桶获取。以下是各平台的获取方式：

```bash
# 查看当前 Flutter SDK 使用的 Engine 版本
flutter --version  # 输出中有 "Engine • revision XXXXXX"

# 方法 1：从 Flutter SDK 的 artifacts 目录复制（推荐）
# Windows
# SDK_PATH/bin/cache/artifacts/engine/windows-x64/
#   ├── flutter_engine.dll
#   ├── flutter_engine.dll.lib
#   └── flutter_embedder.h

# Linux
# SDK_PATH/bin/cache/artifacts/engine/linux-x64/
#   ├── libflutter_engine.so
#   └── flutter_embedder.h

# macOS
# SDK_PATH/bin/cache/artifacts/engine/darwin-x64/
#   ├── libflutter_engine.dylib
#   └── flutter_embedder.h

# 方法 2：从 Google 存储桶下载（指定 Engine 版本哈希）
# https://storage.googleapis.com/flutter_infra_release/flutter/
#   ENGINE_HASH/linux-x64/linux-x64-embedder.zip
```

### 示例 3：两阶段启动 — 在初始化和运行之间注册 Channel

在实际嵌入场景中（如 KrKr2），通常需要在 Engine 启动 Dart 代码之前就注册好 Platform Channel（一种 Flutter 与宿主通信的机制，下一节详讲），这时就需要两阶段启动：

```cpp
// two_phase_startup.cpp
// 展示 Initialize → 配置 → Run 的两阶段启动模式
#include <flutter_embedder.h>
#include <iostream>
#include <string>
#include <vector>
#include <functional>

// 平台消息回调：接收 Dart 侧通过 Platform Channel 发来的消息
void OnPlatformMessage(
    const FlutterPlatformMessage* message,
    void* user_data
) {
    // message->channel 是通道名称（如 "com.krkr2/game_engine"）
    // message->message 是消息内容（二进制数据）
    // message->message_size 是消息长度
    std::string channel(message->channel);
    std::cout << "收到 Platform Message，通道: " << channel
              << "，大小: " << message->message_size << " 字节"
              << std::endl;

    // 构造响应
    const char* response = "OK";
    FlutterEngineSendPlatformMessageResponse(
        static_cast<FlutterEngine>(user_data),  // Engine 句柄
        message->response_handle,                // 响应句柄
        reinterpret_cast<const uint8_t*>(response),
        2  // "OK" 的字节长度
    );
}

int main() {
    // ===== 渲染配置（此处用软件渲染简化示例）=====
    FlutterRendererConfig renderer_config = {};
    renderer_config.type = kSoftware;
    renderer_config.software.struct_size =
        sizeof(FlutterSoftwareRendererConfig);
    // 软件渲染的表面呈现回调
    renderer_config.software.surface_present_callback =
        [](void* user_data,
           const void* allocation,
           size_t row_bytes,
           size_t height) -> bool {
        // 此处可以将像素数据写入文件或显示到窗口
        std::cout << "软件渲染帧: "
                  << row_bytes / 4 << "x" << height
                  << std::endl;
        return true;
    };

    // ===== 项目参数 =====
    FlutterProjectArgs args = {};
    args.struct_size = sizeof(FlutterProjectArgs);
    args.assets_path = "./flutter_assets";
    args.icu_data_path = "./icudtl.dat";
    // 注册平台消息回调
    args.platform_message_callback = OnPlatformMessage;

    // ===== 第 1 阶段：初始化（不运行 Dart）=====
    FlutterEngine engine = nullptr;
    FlutterEngineResult result = FlutterEngineInitialize(
        FLUTTER_ENGINE_VERSION,
        &renderer_config,
        &args,
        nullptr,    // user_data
        &engine
    );

    if (result != kSuccess) {
        std::cerr << "Engine 初始化失败: " << result << std::endl;
        return -1;
    }
    std::cout << "Engine 初始化完成（Dart 尚未运行）" << std::endl;

    // ===== 初始化与运行之间：可做额外配置 =====
    // 例如：发送初始配置到 Platform Channel
    // 例如：预注册外部纹理
    // 例如：设置系统主题（亮色/暗色）

    // 通知 Engine 系统区域设置
    FlutterLocale locale = {};
    locale.struct_size = sizeof(FlutterLocale);
    locale.language_code = "zh";
    locale.country_code = "CN";
    locale.script_code = "Hans";
    const FlutterLocale* locale_ptr = &locale;
    FlutterEngineUpdateLocales(engine, &locale_ptr, 1);

    // ===== 第 2 阶段：运行 Dart 代码 =====
    result = FlutterEngineRunInitialized(engine);
    if (result != kSuccess) {
        std::cerr << "Engine 运行失败: " << result << std::endl;
        FlutterEngineShutdown(engine);
        return -1;
    }
    std::cout << "Dart 代码已开始运行" << std::endl;

    // 发送初始窗口尺寸
    FlutterWindowMetricsEvent metrics = {};
    metrics.struct_size = sizeof(FlutterWindowMetricsEvent);
    metrics.width = 800;
    metrics.height = 600;
    metrics.pixel_ratio = 1.0;
    FlutterEngineSendWindowMetricsEvent(engine, &metrics);

    // 简单主循环（实际项目中应接入平台事件循环）
    std::cout << "按 Enter 退出..." << std::endl;
    std::cin.get();

    // 关闭 Engine
    FlutterEngineDeinitialize(engine);
    FlutterEngineShutdown(engine);
    std::cout << "Engine 已安全关闭" << std::endl;

    return 0;
}
```

## 自定义任务运行器（Custom Task Runners）

### 为什么需要自定义运行器

默认情况下，Flutter Engine 会为四个运行器各创建一个线程。但在嵌入到现有渲染框架（如 Cocos2d-x）时，我们需要：

- **Platform Runner 跑在宿主主线程**：因为平台事件（触摸、键盘）和 UI 操作必须在主线程处理
- **Render Runner 共享 GL 上下文**：如果要与 Cocos2d-x 共享同一个 GL 上下文，可能需要控制 Render Runner 的线程

Flutter Engine 通过 `FlutterCustomTaskRunners` 允许宿主接管线程调度：

```c
// 任务运行器定义
typedef struct {
    size_t struct_size;

    // 向此运行器投递一个任务（callback + target_time）
    // Engine 调用此函数说："请在 target_time 时刻执行 task"
    void (*post_task_callback)(
        FlutterTask task,          // 要执行的任务
        uint64_t target_time_nanos,// 期望执行时间（纳秒）
        void* user_data            // 用户数据
    );

    // 返回当前运行器所在线程的唯一标识
    // Engine 用它判断当前是否在正确的线程上
    size_t (*runs_task_on_current_thread_callback)(void* user_data);

    void* user_data;  // 透传给上面两个回调
} FlutterTaskRunnerDescription;

typedef struct {
    size_t struct_size;
    const FlutterTaskRunnerDescription* platform_task_runner;
    const FlutterTaskRunnerDescription* render_task_runner;
    // UI Runner 和 IO Runner 目前不支持自定义
} FlutterCustomTaskRunners;
```

### 示例 4：将 Platform Runner 挂载到 Cocos2d-x 主线程

下面展示如何将 Flutter 的 Platform Runner 集成到 Cocos2d-x 的调度器（Scheduler）中，让 Flutter 的平台任务在 Cocos2d-x 的主线程上执行：

```cpp
// cocos_flutter_task_runner.h
// 将 Flutter Platform Runner 集成到 Cocos2d-x 调度器
#pragma once
#include <flutter_embedder.h>
#include <cocos2d.h>     // Cocos2d-x 头文件
#include <mutex>
#include <queue>
#include <thread>

// 任务队列项：封装 Flutter 任务和目标执行时间
struct PendingTask {
    FlutterTask task;          // Flutter 任务句柄
    uint64_t target_time_nanos;// 期望执行时间
};

class CocosFlutterTaskRunner {
public:
    static CocosFlutterTaskRunner& getInstance() {
        static CocosFlutterTaskRunner instance;
        return instance;
    }

    // 初始化：记录主线程 ID，注册到 Cocos2d-x 调度器
    void init(FlutterEngine engine) {
        engine_ = engine;
        main_thread_id_ = std::this_thread::get_id();

        // 注册到 Cocos2d-x 的每帧更新
        auto scheduler = cocos2d::Director::getInstance()
                            ->getScheduler();
        scheduler->schedule(
            [this](float dt) { this->processFlutterTasks(); },
            this,
            0.0f,   // 每帧都执行（interval = 0）
            false,   // 不暂停
            "flutter_task_runner"  // 调度器键名
        );
    }

    // Flutter 调用此函数投递任务
    static void postTask(
        FlutterTask task,
        uint64_t target_time_nanos,
        void* user_data
    ) {
        auto& runner = getInstance();
        std::lock_guard<std::mutex> lock(runner.mutex_);
        runner.pending_tasks_.push({task, target_time_nanos});
    }

    // 判断当前是否在 Platform Runner 线程上
    static bool isOnPlatformThread(void* user_data) {
        auto& runner = getInstance();
        return std::this_thread::get_id() == runner.main_thread_id_;
    }

    // 获取当前时间（纳秒，与 Flutter Engine 时钟对齐）
    static uint64_t getCurrentTimeNanos() {
        return FlutterEngineGetCurrentTime();
    }

private:
    // Cocos2d-x 每帧调用：执行到期的 Flutter 任务
    void processFlutterTasks() {
        std::queue<PendingTask> tasks_to_run;
        {
            std::lock_guard<std::mutex> lock(mutex_);
            uint64_t now = getCurrentTimeNanos();
            // 取出所有已到期的任务
            while (!pending_tasks_.empty()) {
                auto& front = pending_tasks_.front();
                if (front.target_time_nanos <= now) {
                    tasks_to_run.push(front);
                    pending_tasks_.pop();
                } else {
                    break;  // 队列按时间排序，后面的也没到期
                }
            }
        }
        // 在主线程上执行任务（不持有锁，避免死锁）
        while (!tasks_to_run.empty()) {
            auto& t = tasks_to_run.front();
            FlutterEngineRunTask(engine_, &t.task);
            tasks_to_run.pop();
        }
    }

    FlutterEngine engine_ = nullptr;
    std::thread::id main_thread_id_;
    std::mutex mutex_;
    std::priority_queue<
        PendingTask,
        std::vector<PendingTask>,
        // 最小堆：最早到期的任务排在前面
        decltype([](const PendingTask& a, const PendingTask& b) {
            return a.target_time_nanos > b.target_time_nanos;
        })
    > pending_tasks_;
};
```

使用方式：

```cpp
// 在创建 Engine 时配置自定义 Platform Runner
FlutterTaskRunnerDescription platform_runner = {};
platform_runner.struct_size = sizeof(FlutterTaskRunnerDescription);
platform_runner.post_task_callback = CocosFlutterTaskRunner::postTask;
platform_runner.runs_task_on_current_thread_callback =
    [](void* user_data) -> size_t {
        return CocosFlutterTaskRunner::isOnPlatformThread(user_data);
    };
platform_runner.user_data = nullptr;

FlutterCustomTaskRunners custom_runners = {};
custom_runners.struct_size = sizeof(FlutterCustomTaskRunners);
custom_runners.platform_task_runner = &platform_runner;
// render_task_runner 不设置 → Engine 自己创建 Render Runner 线程

FlutterProjectArgs args = {};
args.struct_size = sizeof(FlutterProjectArgs);
args.custom_task_runners = &custom_runners;
// ... 其他配置
```

**要点**：
- `post_task_callback` 可能从任意线程调用，必须用互斥锁保护任务队列
- `FlutterEngineRunTask` 必须在对应的运行器线程上调用
- 优先队列确保任务按 `target_time_nanos` 顺序执行

## 输入事件转发

Flutter Engine 需要接收来自宿主的输入事件（鼠标、触摸、键盘）才能响应用户交互。Embedder API 提供了以下事件发送函数。

### 鼠标 / 指针事件

```c
typedef struct {
    size_t struct_size;
    FlutterPointerPhase phase;  // 事件阶段
    size_t timestamp;           // 时间戳（微秒）
    double x;                   // X 坐标（物理像素）
    double y;                   // Y 坐标（物理像素）
    int32_t device;             // 设备 ID（支持多指触摸）
    FlutterPointerSignalKind signal_kind;  // 信号类型
    double scroll_delta_x;      // 水平滚动量
    double scroll_delta_y;      // 垂直滚动量
    FlutterPointerDeviceKind device_kind;  // 设备类型
    int64_t buttons;            // 按下的按钮掩码
} FlutterPointerEvent;

// 事件阶段枚举
typedef enum {
    kCancel,  // 取消（如手指移出窗口）
    kUp,      // 抬起（释放按钮 / 手指离开屏幕）
    kDown,    // 按下（按钮按下 / 手指触摸屏幕）
    kMove,    // 移动（拖拽中）
    kAdd,     // 指针进入追踪区域（鼠标进入窗口）
    kRemove,  // 指针离开追踪区域（鼠标离开窗口）
    kHover,   // 悬停（鼠标移动但未按下按钮）
    kPanZoomStart, // 触控板缩放/平移开始
    kPanZoomUpdate,// 触控板缩放/平移更新
    kPanZoomEnd,   // 触控板缩放/平移结束
} FlutterPointerPhase;

// 发送指针事件给 Engine
FlutterEngineResult FlutterEngineSendPointerEvent(
    FLUTTER_API_SYMBOL(FlutterEngine) engine,
    const FlutterPointerEvent* events,  // 事件数组
    size_t events_count                 // 事件个数
);
```

### 示例 5：GLFW 鼠标事件转发到 Flutter

```cpp
// mouse_event_forwarding.cpp
// 将 GLFW 的鼠标事件转发给 Flutter Engine
#include <flutter_embedder.h>
#include <GLFW/glfw3.h>
#include <chrono>

// 获取微秒级时间戳（Flutter 要求微秒）
static size_t GetTimestampMicros() {
    auto now = std::chrono::steady_clock::now();
    auto duration = now.time_since_epoch();
    return std::chrono::duration_cast<std::chrono::microseconds>(
        duration
    ).count();
}

// 追踪鼠标按钮状态
static bool g_mouse_button_pressed = false;

// GLFW 鼠标按钮回调 → Flutter 指针事件
void OnMouseButton(
    GLFWwindow* window,
    int button,          // GLFW_MOUSE_BUTTON_LEFT 等
    int action,          // GLFW_PRESS 或 GLFW_RELEASE
    int mods             // 修饰键
) {
    FlutterEngine engine = /* 从窗口用户指针获取 */
        static_cast<FlutterEngine>(glfwGetWindowUserPointer(window));

    double x, y;
    glfwGetCursorPos(window, &x, &y);  // 获取当前鼠标位置

    // 计算物理像素坐标（高 DPI 缩放）
    int win_w, win_h, fb_w, fb_h;
    glfwGetWindowSize(window, &win_w, &win_h);
    glfwGetFramebufferSize(window, &fb_w, &fb_h);
    double pixel_ratio = static_cast<double>(fb_w)
                       / static_cast<double>(win_w);
    x *= pixel_ratio;
    y *= pixel_ratio;

    FlutterPointerEvent event = {};
    event.struct_size = sizeof(FlutterPointerEvent);
    event.timestamp = GetTimestampMicros();
    event.x = x;
    event.y = y;
    event.device = 0;  // 鼠标是设备 0
    event.device_kind = kFlutterPointerDeviceKindMouse;

    if (action == GLFW_PRESS) {
        event.phase = kDown;
        g_mouse_button_pressed = true;
        // 设置按钮掩码
        if (button == GLFW_MOUSE_BUTTON_LEFT) {
            event.buttons = kFlutterPointerButtonMousePrimary;
        } else if (button == GLFW_MOUSE_BUTTON_RIGHT) {
            event.buttons = kFlutterPointerButtonMouseSecondary;
        } else if (button == GLFW_MOUSE_BUTTON_MIDDLE) {
            event.buttons = kFlutterPointerButtonMouseMiddle;
        }
    } else {  // GLFW_RELEASE
        event.phase = kUp;
        g_mouse_button_pressed = false;
        event.buttons = 0;
    }

    FlutterEngineSendPointerEvent(engine, &event, 1);
}

// GLFW 鼠标移动回调 → Flutter 指针事件
void OnMouseMove(GLFWwindow* window, double x, double y) {
    FlutterEngine engine =
        static_cast<FlutterEngine>(glfwGetWindowUserPointer(window));

    // 高 DPI 坐标转换
    int win_w, fb_w;
    glfwGetWindowSize(window, &win_w, nullptr);
    glfwGetFramebufferSize(window, &fb_w, nullptr);
    double pixel_ratio = static_cast<double>(fb_w)
                       / static_cast<double>(win_w);
    x *= pixel_ratio;
    y *= pixel_ratio;

    FlutterPointerEvent event = {};
    event.struct_size = sizeof(FlutterPointerEvent);
    event.timestamp = GetTimestampMicros();
    event.x = x;
    event.y = y;
    event.device = 0;
    event.device_kind = kFlutterPointerDeviceKindMouse;

    // 根据按钮状态选择 Move（拖拽）或 Hover（悬停）
    if (g_mouse_button_pressed) {
        event.phase = kMove;
        event.buttons = kFlutterPointerButtonMousePrimary;
    } else {
        event.phase = kHover;
        event.buttons = 0;
    }

    FlutterEngineSendPointerEvent(engine, &event, 1);
}

// GLFW 滚轮回调 → Flutter 滚动事件
void OnMouseScroll(GLFWwindow* window, double dx, double dy) {
    FlutterEngine engine =
        static_cast<FlutterEngine>(glfwGetWindowUserPointer(window));

    double x, y;
    glfwGetCursorPos(window, &x, &y);

    FlutterPointerEvent event = {};
    event.struct_size = sizeof(FlutterPointerEvent);
    event.timestamp = GetTimestampMicros();
    event.x = x;
    event.y = y;
    event.device = 0;
    event.device_kind = kFlutterPointerDeviceKindMouse;
    event.phase = kHover;
    event.signal_kind = kFlutterPointerSignalKindScroll;
    event.scroll_delta_x = dx * 20.0;  // 像素滚动量
    event.scroll_delta_y = -dy * 20.0; // Flutter Y 轴方向与 GLFW 相反

    FlutterEngineSendPointerEvent(engine, &event, 1);
}

// 注册所有鼠标回调
void SetupMouseCallbacks(GLFWwindow* window) {
    glfwSetMouseButtonCallback(window, OnMouseButton);
    glfwSetCursorPosCallback(window, OnMouseMove);
    glfwSetScrollCallback(window, OnMouseScroll);
}
```

### 键盘事件

Flutter Engine 也接收键盘事件，用于文本输入和快捷键：

```c
typedef struct {
    size_t struct_size;
    FlutterKeyEventType type;   // kFlutterKeyEventTypeUp / Down / Repeat
    size_t timestamp;           // 微秒时间戳
    uint64_t physical;          // 物理按键码（USB HID 标准）
    uint64_t logical;           // 逻辑按键码（考虑键盘布局）
    const char* character;      // 输入的字符（UTF-8，如 "a"、"你"）
    bool synthesized;           // 是否为合成事件
} FlutterKeyEvent;

// 发送键盘事件
FlutterEngineResult FlutterEngineSendKeyEvent(
    FLUTTER_API_SYMBOL(FlutterEngine) engine,
    const FlutterKeyEvent* event,
    FlutterKeyEventCallback callback,  // 事件是否被 Flutter 消费的回调
    void* user_data
);
```

**注意**：物理按键码使用 USB HID 标准编码，而非平台特定扫描码。Flutter 的 `flutter/keyboard_maps.h` 提供了完整的映射表。

## KrKr2 嵌入场景分析

### 现有架构回顾

KrKr2 使用 Cocos2d-x 作为渲染框架，当前的 UI 层由 `cpp/core/environ/ui/` 中的代码实现（如 `TVPMainForm`、`TVPMessageBoxForm`）。这些 UI 组件直接使用 Cocos2d-x 的节点系统和 Cocos Studio 的 `.csb` 布局文件（位于 `ui/cocos-studio/`）。

要将 Flutter 引入 KrKr2，有两种集成策略：

### 策略 A：Flutter 作为 UI 覆盖层（Overlay）

```
┌──────────────────────────────────────┐
│          Flutter UI 层               │  ← 最上层：菜单、对话框、设置
│    (Texture Widget / 离屏 FBO)       │
├──────────────────────────────────────┤
│        Cocos2d-x 渲染场景           │  ← 游戏画面：图层、立绘、背景
│    (Director → Scene → Sprite)       │
├──────────────────────────────────────┤
│        平台窗口 / Surface            │  ← GLFW / SDL / Android Surface
└──────────────────────────────────────┘
```

实现方式：
1. Flutter Engine 渲染到一个离屏 FBO（帧缓冲对象），不直接输出到屏幕
2. 将 FBO 的颜色附件作为纹理（OpenGL 纹理 ID）
3. 在 Cocos2d-x 的渲染循环最后阶段，用一个全屏 Sprite 将 Flutter 纹理覆盖绘制到屏幕上
4. 透明区域不遮挡底层的游戏画面（Flutter UI 背景设为透明）

**优点**：改动最小，Cocos2d-x 渲染完全不受影响，Flutter 只管 UI 层
**缺点**：两个渲染管线同时运行，内存和 GPU 开销较大

### 策略 B：Flutter 作为主渲染框架

```
┌──────────────────────────────────────┐
│          Flutter Framework           │  ← 所有 UI 由 Flutter Widget 实现
│    ┌────────────────────────────┐    │
│    │   Texture Widget           │    │  ← 游戏画面通过 Texture Widget 嵌入
│    │   (显示 KrKr2 渲染输出)    │    │
│    └────────────────────────────┘    │
├──────────────────────────────────────┤
│        Flutter Engine (Skia)         │  ← 唯一的渲染管线
├──────────────────────────────────────┤
│        平台窗口 / Surface            │
└──────────────────────────────────────┘
```

实现方式：
1. Flutter 成为主窗口框架，替代 Cocos2d-x 的窗口管理
2. KrKr2 的游戏渲染（图层合成、立绘、背景）输出到一个纹理
3. Flutter 通过 `Texture` Widget 显示这个纹理
4. 所有 UI（菜单、存档、设置）用 Flutter Widget 实现

**优点**：统一渲染管线，UI 表现力强，支持动画和手势
**缺点**：改动巨大，需要重写渲染管线的输出方式

### 推荐：渐进式策略（A → B）

对于 KrKr2 项目，推荐先实施策略 A（UI 覆盖层），验证 Flutter 嵌入可行后，再逐步迁移到策略 B：

| 阶段 | 目标 | 改动范围 |
|------|------|----------|
| 阶段 1 | Flutter 渲染到离屏 FBO，Cocos2d-x 将其作为纹理覆盖 | 新增 Flutter 嵌入层，不改现有代码 |
| 阶段 2 | 将设置界面从 Cocos Studio 迁移到 Flutter | 删除部分 `.csb` 文件和对应 C++ 代码 |
| 阶段 3 | 将所有 UI 迁移到 Flutter，评估是否切换到策略 B | 逐步移除 Cocos2d-x UI 依赖 |

## 动手实践

### 练习 1：搭建最小 Flutter 嵌入环境

**目标**：在你的开发机上成功运行示例 1（最小化 Flutter Engine 嵌入）。

**步骤**：

1. 安装 Flutter SDK（确保 `flutter doctor` 无错误）
2. 创建一个简单的 Flutter 项目作为 UI：
   ```bash
   flutter create flutter_ui_demo
   cd flutter_ui_demo
   flutter build bundle  # 生成 flutter_assets 目录
   ```
3. 从 Flutter SDK 中提取 Engine 动态库和头文件（参考上文"获取 Flutter Engine 动态库"）
4. 将示例 1 的 `minimal_flutter_host.cpp` 和 `CMakeLists.txt` 放入项目目录
5. 构建并运行：
   ```bash
   cmake -B build -DFLUTTER_ENGINE_DIR=<你的engine路径>
   cmake --build build
   # 将 flutter_assets 目录和 icudtl.dat 复制到可执行文件旁
   ./build/flutter_demo
   ```

**预期结果**：看到一个窗口，显示 Flutter 默认的计数器应用界面。

### 练习 2：添加自定义 Platform Runner

**目标**：基于示例 4，实现一个简单的自定义 Platform Runner（不使用 Cocos2d-x，直接在 GLFW 主循环中处理）。

**提示**：在 `glfwPollEvents()` 之后调用一个函数来处理队列中的 Flutter 任务。

## 对照项目源码

虽然 KrKr2 当前尚未集成 Flutter Engine，但我们可以参考现有的 UI 架构来理解将来的嵌入点：

相关文件：

- `cpp/core/environ/cocos2d/AppDelegate.cpp` — Cocos2d-x 应用入口，`applicationDidFinishLaunching()` 是 Flutter Engine 初始化的最佳挂载点（在 Cocos2d-x 初始化完成后、进入主循环前）
- `cpp/core/environ/ui/TVPMainForm.cpp` — 主窗口表单，未来将被 Flutter UI 替代
- `cpp/core/environ/ui/TVPMessageBoxForm.cpp` — 消息对话框，Flutter 可用 `showDialog()` 替代
- `platforms/android/cpp/krkr2_android.cpp` — Android JNI 桥接，Flutter 在 Android 上的嵌入也通过 JNI（`FlutterJNI` 类）
- `platforms/windows/main.cpp` — Windows 入口，Flutter 的 Windows Embedder 使用 Win32 API 创建窗口

## 常见错误与排查

### 错误 1：`FlutterEngineRun` 返回 `kInvalidArguments`

**症状**：Engine 启动失败，返回错误码 1。

**原因与解决**：

```
常见原因：
1. struct_size 未设置 → 确保 args.struct_size = sizeof(FlutterProjectArgs)
2. assets_path 路径不存在 → 检查 flutter_assets 目录是否在正确位置
3. icu_data_path 文件不存在 → 检查 icudtl.dat 是否已复制
4. 渲染回调为 nullptr → OpenGL 的 make_current、present、fbo_callback 必须设置
5. API 版本不匹配 → 确保传入 FLUTTER_ENGINE_VERSION 而非硬编码数字
```

### 错误 2：渲染画面全黑 / 不显示

**症状**：Engine 启动成功，但窗口一片黑色。

**排查步骤**：

```
1. 检查 FBO 回调：是否返回了正确的 FBO ID？
   - 如果绘制到窗口默认帧缓冲，返回 0
   - 如果使用离屏 FBO，确保 FBO 已正确创建和绑定

2. 检查 present 回调：是否调用了 SwapBuffers？
   - GLFW: glfwSwapBuffers(window)
   - EGL: eglSwapBuffers(display, surface)

3. 检查窗口尺寸：是否发送了 WindowMetricsEvent？
   - 没有发送尺寸信息，Engine 不知道视口大小，不会渲染

4. 检查 GL 上下文：make_current 回调是否生效？
   - 在回调中加日志确认是否被调用

5. 检查像素比：pixel_ratio 是否正确？
   - 如果 pixel_ratio 为 0，引擎会跳过渲染
```

### 错误 3：崩溃在 `FlutterEngineRunTask`

**症状**：在自定义 Task Runner 中执行任务时崩溃。

**原因**：`FlutterEngineRunTask` 被在错误的线程上调用。Platform Task 必须在 Platform Runner 线程上执行，Render Task 必须在 Render Runner 线程上执行。

**解决**：确保 `runs_task_on_current_thread_callback` 正确返回当前线程标识，且任务只在对应线程的 `processFlutterTasks()` 中执行。

## 本节小结

- Flutter Engine 是一个 C/C++ 运行时，通过 **Embedder API**（纯 C 接口，定义在 `flutter_engine.h` / `embedder.h`）与宿主程序交互
- 核心数据结构包括 `FlutterProjectArgs`（项目配置）、`FlutterRendererConfig`（渲染后端）、`FlutterCustomTaskRunners`（线程调度）
- Engine 使用**四个任务运行器**（Platform / UI / Render / IO）隔离不同类型的工作，其中 Platform Runner 必须跑在宿主主线程
- Engine 支持**两阶段启动**（Initialize → Run），允许在启动 Dart 前做额外配置
- 渲染后端可选 OpenGL、Vulkan、Metal 或软件渲染，对 KrKr2 项目首选 **OpenGL ES**（与 Cocos2d-x 共享上下文）
- 输入事件通过 `FlutterEngineSendPointerEvent` / `FlutterEngineSendKeyEvent` 转发给 Engine
- KrKr2 的 Flutter 集成推荐采用**渐进式策略**：先作为 UI 覆盖层（离屏 FBO + 纹理合成），后续可迁移为主渲染框架

## 练习题与答案

### 题目 1：Flutter Engine 的四个任务运行器分别负责什么？为什么 Platform Runner 必须跑在宿主主线程上？

<details>
<summary>查看答案</summary>

**四个任务运行器及职责**：

1. **Platform Runner**：处理平台事件（触摸、键盘、生命周期变更）、Plugin 调用。这是 Engine 与宿主操作系统交互的唯一通道。
2. **UI Runner**：运行 Dart 虚拟机，执行 Widget 构建（`build()`）、布局计算（`layout()`）、动画帧回调。不执行实际 GPU 操作。
3. **Render Runner**：将 UI Runner 生成的渲染指令通过 Skia 转化为 GPU 命令，管理 OpenGL/Vulkan/Metal 上下文和资源。
4. **IO Runner**：处理 I/O 密集操作，主要是图片解码（将压缩图片解码为 GPU 纹理）和文件读写。

**Platform Runner 必须跑在主线程的原因**：

几乎所有操作系统都要求 UI 操作（创建窗口、处理输入事件、显示对话框等）在主线程上执行。这是 Windows 的消息循环（`GetMessage/DispatchMessage`）、macOS 的 `NSApplication` 运行循环、Android 的主 Looper 和 Linux 的 X11/Wayland 事件循环的共同要求。如果 Platform Runner 运行在非主线程上，平台 API 调用会崩溃或产生未定义行为。

</details>

### 题目 2：请写出一个完整的 `FlutterRendererConfig` 配置，使用 OpenGL 后端，FBO 编号为 42（自定义离屏 FBO），并在控制台输出每次 present 的时间戳。

<details>
<summary>查看答案</summary>

```cpp
// opengl_renderer_config.cpp
#include <flutter_embedder.h>
#include <iostream>
#include <chrono>

// GL 上下文管理回调（假设使用 EGL）
bool MakeCurrent(void* user_data) {
    // 实际项目中调用 eglMakeCurrent(display, surface, surface, context)
    std::cout << "[GL] make_current 被调用" << std::endl;
    return true;
}

bool ClearCurrent(void* user_data) {
    // 实际项目中调用 eglMakeCurrent(display, EGL_NO_SURFACE, ...)
    std::cout << "[GL] clear_current 被调用" << std::endl;
    return true;
}

// 返回自定义离屏 FBO 的编号
uint32_t GetFBO(void* user_data) {
    return 42;  // 自定义 FBO ID
}

// present 回调：输出时间戳
bool Present(void* user_data) {
    auto now = std::chrono::steady_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        now.time_since_epoch()
    ).count();
    std::cout << "[Present] 时间戳: " << ms << " ms" << std::endl;
    // 实际项目中调用 eglSwapBuffers 或将 FBO 纹理提交给合成器
    return true;
}

// 获取 GL 函数指针
void* GetProcAddress(void* user_data, const char* name) {
    // 实际项目中调用 eglGetProcAddress(name)
    return nullptr;
}

// 使资源上下文成为当前（IO Runner 用）
bool MakeResourceCurrent(void* user_data) {
    // 实际项目中切换到共享的资源 GL 上下文
    return true;
}

FlutterRendererConfig CreateOpenGLConfig() {
    FlutterRendererConfig config = {};
    config.type = kOpenGL;  // 选择 OpenGL 后端

    config.open_gl.struct_size = sizeof(FlutterOpenGLRendererConfig);
    config.open_gl.make_current = MakeCurrent;
    config.open_gl.clear_current = ClearCurrent;
    config.open_gl.present = Present;
    config.open_gl.fbo_callback = GetFBO;        // 返回 FBO 42
    config.open_gl.gl_proc_resolver = GetProcAddress;
    config.open_gl.make_resource_current = MakeResourceCurrent;

    return config;
}

int main() {
    FlutterRendererConfig config = CreateOpenGLConfig();
    std::cout << "渲染配置创建完成" << std::endl;
    std::cout << "  后端类型: OpenGL" << std::endl;
    std::cout << "  FBO ID: " << config.open_gl.fbo_callback(nullptr)
              << std::endl;
    // 模拟调用 present
    config.open_gl.present(nullptr);
    return 0;
}
```

编译运行后输出：
```
渲染配置创建完成
  后端类型: OpenGL
  FBO ID: 42
[Present] 时间戳: 1742539200000 ms
```

</details>

### 题目 3：如果要在 KrKr2 中使用 Flutter 作为 UI 覆盖层（策略 A），请描述 OpenGL 层面需要执行哪些步骤来实现 Flutter 渲染到离屏 FBO 再合成到 Cocos2d-x 场景的过程。

<details>
<summary>查看答案</summary>

**实现步骤**：

1. **创建离屏 FBO 和纹理**：
   ```cpp
   GLuint fbo, texture;
   glGenFramebuffers(1, &fbo);      // 创建帧缓冲对象
   glGenTextures(1, &texture);       // 创建颜色纹理
   glBindTexture(GL_TEXTURE_2D, texture);
   glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA,
                window_width, window_height,
                0, GL_RGBA, GL_UNSIGNED_BYTE, nullptr);
   // 设置纹理过滤参数
   glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
   glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
   // 将纹理附着到 FBO
   glBindFramebuffer(GL_FRAMEBUFFER, fbo);
   glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,
                          GL_TEXTURE_2D, texture, 0);
   ```

2. **配置 Flutter Engine 使用该 FBO**：
   - 在 `fbo_callback` 中返回 `fbo` 的 ID
   - Flutter Engine 会将所有 UI 渲染到这个 FBO 上

3. **在 Cocos2d-x 渲染循环的尾部合成**：
   ```cpp
   // 在 Cocos2d-x Scene 中添加一个自定义绘制节点
   class FlutterOverlayNode : public cocos2d::Node {
       void draw(cocos2d::Renderer* renderer,
                 const cocos2d::Mat4& transform,
                 uint32_t flags) override {
           // 绑定 Flutter 的纹理
           glBindTexture(GL_TEXTURE_2D, flutter_texture);
           // 绘制全屏四边形（混合模式开启，支持透明）
           glEnable(GL_BLEND);
           glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
           drawFullscreenQuad();  // 用简单的顶点着色器+片段着色器
           glDisable(GL_BLEND);
       }
   };
   ```

4. **同步渲染时序**：
   - Flutter Engine 的 Render Runner 在自己的线程上渲染到 FBO
   - Cocos2d-x 在主线程上读取 FBO 的纹理
   - 需要使用 GL 同步对象（`glFenceSync` / `glWaitSync`）确保 Flutter 渲染完成后 Cocos2d-x 才读取纹理

5. **处理窗口大小变化**：
   - 窗口大小改变时，需要重新创建 FBO 和纹理（新尺寸）
   - 同时通过 `FlutterEngineSendWindowMetricsEvent` 通知 Engine

6. **处理透明度**：
   - Flutter UI 的背景色设为透明（`MaterialApp(theme: ThemeData(scaffoldBackgroundColor: Colors.transparent))`）
   - FBO 的纹理格式使用 `GL_RGBA`（含 Alpha 通道）
   - 合成时使用 `GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA` 混合模式

</details>

## 下一步

[02-Platform-Channel与纹理共享](02-Platform-Channel与纹理共享.md) — 学习 Flutter 与 C++ 宿主之间的通信机制（Platform Channel）和纹理共享技术（Texture Widget），这是实现 KrKr2 游戏画面在 Flutter UI 中显示的关键技术。

