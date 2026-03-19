# KrKr2 中的 SDL2 集成架构

> **所属模块：** P11-SDL2跨平台开发
> **前置知识：** [03-自定义事件与多窗口](../02-窗口与事件循环/03-自定义事件与多窗口.md)、[P10-Cocos2d-x框架](../../P10-Cocos2dx框架/README.md)
> **预计阅读时间：** 20 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 SDL2 在 KrKr2 引擎架构中的定位——它不是被引擎直接调用的，而是作为 Cocos2d-x 的后端
2. 追踪 KrKr2 从启动到 SDL2 初始化的完整调用链
3. 了解 KrKr2 各平台入口如何最终汇聚到 Cocos2d-x → SDL2 这条路径
4. 分析 `tvpsdl.cpp` 的作用与局限

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 平台抽象层 | Platform Abstraction Layer (PAL) | KrKr2 中隔离平台差异的一层代码，让上层引擎逻辑无需关心运行在哪个操作系统 |
| AppDelegate | Application Delegate | Cocos2d-x 的应用生命周期管理类，KrKr2 继承它来控制启动/暂停/恢复流程 |
| Director | Director (Cocos2d-x) | Cocos2d-x 的"导演"单例，负责场景调度、帧循环、OpenGL 上下文管理 |
| ANativeWindow | Android Native Window | Android NDK 提供的原生窗口句柄，SDL2 在 Android 上通过它获取渲染表面 |
| GLView | OpenGL View | Cocos2d-x 中封装 OpenGL 上下文和窗口的抽象类，在 SDL2 后端由 `GLViewImpl` 实现 |

## 架构总览：SDL2 在哪一层？

KrKr2 的架构是多层嵌套的。从上到下：

```
┌─────────────────────────────────────────┐
│           TJS2 脚本 / KAG 标签          │  ← 游戏开发者写的脚本
├─────────────────────────────────────────┤
│         KrKr2 引擎核心 (C++)            │  ← visual/sound/movie/base/plugin
│   渲染层 │ 音频层 │ 视频层 │ 归档/IO    │
├─────────────────────────────────────────┤
│          Cocos2d-x 框架                 │  ← 场景管理、精灵渲染、UI
│   Director │ Scene │ Sprite │ GLView   │
├─────────────────────────────────────────┤
│     平台后端（SDL2 / 原生 API）          │  ← 窗口、事件、OpenGL 上下文
│   SDL2 (Linux/Android) │ Win32 API     │
├─────────────────────────────────────────┤
│           操作系统 / 驱动               │
└─────────────────────────────────────────┘
```

**关键认知：** KrKr2 的引擎核心代码**从不直接调用 SDL2 函数**。你在 `cpp/core/` 中搜索 `SDL_CreateWindow`、`SDL_GL`、`SDL_PollEvent` 都找不到匹配——因为这些调用全在 Cocos2d-x 内部完成。SDL2 对 KrKr2 引擎来说是完全透明的。

这与传统的 SDL2 游戏开发模式截然不同。在典型的 SDL2 项目中，开发者直接调用 `SDL_Init`、`SDL_CreateWindow` 等函数。而在 KrKr2 中，Cocos2d-x 封装了这一切，引擎只与 Cocos2d-x 的 API 交互。

## 启动流程追踪

### 各平台入口

KrKr2 在四个平台上有不同的入口文件，但最终都汇聚到同一个 Cocos2d-x 生命周期：

```
platforms/windows/main.cpp    → WinMain()
platforms/linux/main.cpp      → main()
platforms/apple/macos/main.cpp → main()
platforms/android/cpp/krkr2_android.cpp → JNI_OnLoad()
                ↓
        TVPAppDelegate::run()
                ↓
     applicationDidFinishLaunching()
                ↓
       cocos2d::Director::getInstance()
                ↓
         GLView 创建（平台相关）
                ↓
          SDL2 初始化（如果使用 SDL2 后端）
```

### Linux 入口详解

Linux 平台是最典型的 SDL2 路径。来看实际代码：

```cpp
// 文件：platforms/linux/main.cpp
// Linux 使用标准 main() 入口
int main(int argc, char* argv[]) {
    // 1. 创建 KrKr2 的应用代理
    TVPAppDelegate app;
    
    // 2. 调用 run()——这会进入 Cocos2d-x 的主循环
    return app.run();
}
```

`TVPAppDelegate` 继承自 Cocos2d-x 的 `cocos2d::Application`。`run()` 方法的调用链如下：

```cpp
// 文件：cpp/core/environ/cocos2d/AppDelegate.cpp（简化）
bool TVPAppDelegate::applicationDidFinishLaunching() {
    // 1. 获取 Director 单例——Cocos2d-x 的核心调度器
    auto director = cocos2d::Director::getInstance();
    
    // 2. 创建 GLView——这一步触发 SDL2 窗口和 GL 上下文创建
    auto glview = director->getOpenGLView();
    if (!glview) {
        // GLViewImpl::create() 内部调用 SDL_CreateWindow + SDL_GL_CreateContext
        glview = cocos2d::GLViewImpl::create("KrKr2");
        director->setOpenGLView(glview);
    }
    
    // 3. 设置帧率
    director->setAnimationInterval(1.0f / 60);
    
    // 4. 创建初始场景并运行
    auto scene = /* KrKr2 的主场景 */;
    director->runWithScene(scene);
    
    return true;
}
```

**注意第 2 步**：`GLViewImpl::create()` 是 SDL2 真正被调用的地方。这个类在 Cocos2d-x 框架内部，不在 KrKr2 的代码中。Cocos2d-x 的 `GLViewImpl`（SDL2 后端版本）在构造时调用：

1. `SDL_Init(SDL_INIT_VIDEO)` — 初始化 SDL2 视频子系统
2. `SDL_GL_SetAttribute(...)` — 设置 OpenGL 属性（版本、颜色深度等）
3. `SDL_CreateWindow(...)` — 创建窗口
4. `SDL_GL_CreateContext(...)` — 创建 OpenGL 上下文

### Android 入口详解

Android 的路径更复杂，涉及 Java → JNI → C++ 的跨语言调用：

```
Java 层:
  AndroidManifest.xml → Cocos2dxActivity (主 Activity)
      ↓ onCreate()
  Cocos2dxGLSurfaceView (OpenGL ES 渲染 Surface)
      ↓ onSurfaceCreated()
  Cocos2dxRenderer.nativeInit() → JNI 调用

JNI 桥接层:
  platforms/android/cpp/krkr2_android.cpp
      ↓ JNI_OnLoad() 注册本地方法
      ↓ nativeInit() 被 Java 层调用

C++ 层:
  TVPAppDelegate::run()
      ↓ applicationDidFinishLaunching()
  Director + GLView（Android 上使用 EGL 而非 SDL2）
```

**Android 的特殊性：** 在 Android 上，Cocos2d-x 默认使用 EGL + ANativeWindow 而非 SDL2 来管理 OpenGL 上下文。SDL2 在 Android 上的角色更像是一个可选的 HAL（硬件抽象层），负责音频、输入事件的转发，但窗口和 GL 上下文由 Cocos2d-x 的 Android 专用代码管理。

### Windows 入口

```cpp
// 文件：platforms/windows/main.cpp
int APIENTRY WinMain(HINSTANCE hInstance, HINSTANCE, LPTSTR, int) {
    TVPAppDelegate app;
    return app.run();
}
```

Windows 上 Cocos2d-x 使用 Win32 API（`CreateWindowEx` + `wglCreateContext`）而非 SDL2。SDL2 在 Windows 版本中不参与窗口管理。

## tvpsdl.cpp 分析

KrKr2 中唯一直接提到 SDL 的引擎代码是 `cpp/core/environ/sdl/tvpsdl.cpp`。这个文件非常短（约 50 行），功能有限：

```cpp
// 文件：cpp/core/environ/sdl/tvpsdl.cpp（实际代码简化）
// 这个文件的作用是在 SDL2 环境下初始化 KrKr2 的引擎子系统
// 注意：它并不调用 SDL_Init 或 SDL_CreateWindow
// 那些工作由 Cocos2d-x 框架在更早的阶段完成

#include "tvpsdl.h"
// ... 引擎子系统初始化代码

void TVPInitializeSDLEnviron() {
    // 初始化 KrKr2 自己的子系统（存储、脚本引擎等）
    // 不涉及 SDL2 API 调用
}
```

**这个文件的名字容易产生误解**——它不是"SDL2 的初始化代码"，而是"在 SDL2 平台环境下的引擎初始化代码"。SDL2 相关的底层操作全部在 Cocos2d-x 内部完成。

## 为什么 KrKr2 不直接使用 SDL2？

| 考量 | 直接使用 SDL2 | 通过 Cocos2d-x 间接使用 |
|------|---------------|------------------------|
| 渲染能力 | 需要手写 OpenGL 渲染管线 | Cocos2d-x 提供完整的 2D 渲染引擎 |
| UI 系统 | 需要自建 | Cocos2d-x 有 Widget、Button、Label 等 |
| 场景管理 | 需要自建 | Director + Scene + Layer 体系 |
| 跨平台 | SDL2 已跨平台，但功能较底层 | Cocos2d-x 在 SDL2 之上再封装一层 |
| 维护成本 | 大量底层代码需要维护 | 框架承担大部分工作 |

KrKr2 选择 Cocos2d-x 是因为视觉小说引擎需要丰富的 2D 渲染能力（图层混合、转场效果、文字排版），这些在 SDL2 层面需要大量自建代码，而 Cocos2d-x 开箱即用。

## 各平台的后端选择

```
                    KrKr2 引擎核心
                        │
                    Cocos2d-x
                   ╱    │    ╲
                  ╱     │     ╲
         SDL2 后端   Win32 后端   EGL/Android 后端
         (Linux)    (Windows)     (Android)
           │           │              │
        libSDL2    user32.dll    ANativeWindow
        + X11/     + opengl32    + EGL + GLESv2
        Wayland      .dll
```

| 平台 | 窗口后端 | GL 上下文 | SDL2 参与度 |
|------|---------|-----------|------------|
| Linux | SDL2 → X11/Wayland | SDL2 → GLX/EGL | **高**：窗口、事件、GL 全由 SDL2 管理 |
| Windows | Win32 API | WGL | **无**：不使用 SDL2 |
| macOS | SDL2 / Cocoa | SDL2 / NSOpenGL | **中**：取决于编译配置 |
| Android | ANativeWindow (Java) | EGL | **低**：SDL2 仅用于音频/输入转发 |

## 常见错误与排查

### 错误 1：试图在 KrKr2 引擎代码中调用 SDL2 函数

```cpp
// ❌ 在 cpp/core/ 中直接调用 SDL2——违反架构分层
#include <SDL.h>
void MyModule::init() {
    SDL_Init(SDL_INIT_VIDEO);  // 错误：这应该由 Cocos2d-x 管理
}

// ✅ 通过 Cocos2d-x API 操作窗口和渲染
void MyModule::init() {
    auto director = cocos2d::Director::getInstance();
    auto glview = director->getOpenGLView();
    // 通过 GLView 获取窗口信息
}
```

### 错误 2：以为 tvpsdl.cpp 负责 SDL2 初始化

```
如果你在调试 SDL2 初始化问题，不要在 tvpsdl.cpp 中查找。
真正的 SDL2 初始化在 Cocos2d-x 的 GLViewImpl 类中：
  - 搜索 Cocos2d-x 源码中的 GLViewImpl::initWithRect()
  - 或搜索 SDL_CreateWindow 调用
```

### 错误 3：在 Android 上依赖 SDL2 窗口功能

```cpp
// ❌ Android 上 SDL2 不管理窗口，这些调用无效
SDL_SetWindowSize(window, 1920, 1080);
SDL_SetWindowFullscreen(window, SDL_WINDOW_FULLSCREEN);

// ✅ Android 窗口由系统管理，通过 JNI 与 Java 层交互
// 或使用 Cocos2d-x 的 GLView::setFrameSize()
```

## 本节小结

- KrKr2 引擎核心**不直接调用 SDL2**——SDL2 是 Cocos2d-x 的后端之一
- 启动链路：`main()` → `TVPAppDelegate::run()` → `Director` → `GLViewImpl::create()` → SDL2 初始化
- `tvpsdl.cpp` 名字易误解，它是"SDL2 平台下的引擎初始化"，不是"SDL2 的初始化"
- Linux 上 SDL2 参与度最高（窗口+事件+GL），Windows 不用 SDL2，Android 仅部分使用
- 理解这个架构后，你就知道：改 SDL2 行为要去 Cocos2d-x 层，改引擎行为在 KrKr2 核心层

## 练习题与答案

### 题目 1：画出 KrKr2 在 Linux 上从 main() 到 SDL_CreateWindow 的调用链

<details>
<summary>查看答案</summary>

```
main() [platforms/linux/main.cpp]
  └→ TVPAppDelegate::run()
       └→ Application::run() [Cocos2d-x]
            └→ applicationDidFinishLaunching() [AppDelegate.cpp]
                 └→ GLViewImpl::create("KrKr2") [Cocos2d-x SDL2 后端]
                      └→ GLViewImpl::initWithRect()
                           ├→ SDL_Init(SDL_INIT_VIDEO)
                           ├→ SDL_GL_SetAttribute(...)  // 设置 GL 版本、颜色深度
                           ├→ SDL_CreateWindow(...)      // 创建窗口
                           └→ SDL_GL_CreateContext(...)   // 创建 GL 上下文
```

关键点：`SDL_CreateWindow` 在 Cocos2d-x 框架内部被调用，KrKr2 引擎代码中没有直接的 SDL2 调用。

</details>

### 题目 2：为什么 KrKr2 选择 Cocos2d-x 而不是直接使用 SDL2？

<details>
<summary>查看答案</summary>

KrKr2 是视觉小说引擎，需要丰富的 2D 渲染能力：
1. **图层系统**：视觉小说需要多层图片叠加（背景、立绘、文字框），Cocos2d-x 的 Node/Sprite/Layer 体系直接支持
2. **转场效果**：场景切换的渐变、百叶窗等效果，Cocos2d-x 内置了数十种 Transition
3. **UI 控件**：菜单、按钮、滑块等 UI 元素，Cocos2d-x 提供现成的 Widget 系统
4. **文字排版**：Label 类支持多行文字、字体加载、描边阴影等
5. **触摸/手势**：EventDispatcher 封装了多点触摸、手势识别

如果直接用 SDL2，以上所有功能都需要自己从零实现。SDL2 只提供窗口、事件、GL 上下文等底层功能，不包含任何 2D 渲染引擎。

</details>

### 题目 3：在哪个文件中能找到真正的 SDL_CreateWindow 调用？

<details>
<summary>查看答案</summary>

在 **Cocos2d-x 框架的源码**中，而非 KrKr2 的代码中。具体路径取决于 Cocos2d-x 版本和配置：

- Cocos2d-x 3.x SDL2 后端：`cocos/platform/desktop/CCGLViewImpl-desktop.cpp`
- 搜索关键字：`SDL_CreateWindow` 或 `initWithRect`

在 KrKr2 项目中，Cocos2d-x 作为依赖库被引入（通过 vcpkg 或子模块），其源码不在 `cpp/core/` 中。如果需要调试 SDL2 窗口创建，需要进入 Cocos2d-x 的源码目录。

**注意**：`cpp/core/environ/sdl/tvpsdl.cpp` 中**不包含** `SDL_CreateWindow` 调用。

</details>

## 下一步

你已理解 SDL2 在 KrKr2 架构中的定位。→ 继续阅读 [02-SDL2与Cocos2d-x桥接](./02-SDL2与Cocos2d-x桥接.md)，深入分析 Cocos2d-x 如何封装 SDL2 的窗口、事件和 GL 上下文。
