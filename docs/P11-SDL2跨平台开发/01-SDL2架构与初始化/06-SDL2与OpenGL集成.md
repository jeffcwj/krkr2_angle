# SDL2 与 OpenGL 集成（上）——GL 属性与上下文创建

> **所属模块：** P11-SDL2跨平台开发
> **前置知识：** [05-对照项目源码与常见错误](./05-对照项目源码与常见错误.md)、[P04-OpenGL图形编程](../../P04-OpenGL图形编程/README.md)
> **预计阅读时间：** 15 分钟

## 本节目标

读完本节后，你将能够：
1. 使用 SDL2 创建 OpenGL 上下文（Context，GPU 与应用程序之间的通信桥梁）并配置 GL 属性
2. 理解各 GL 属性的含义和典型取值
3. 编写跨平台的 GL 版本选择策略（桌面 OpenGL vs 移动端 OpenGL ES）
4. 正确处理 GL 窗口和上下文的创建顺序

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| GL 上下文 | OpenGL Context | GPU 的"工作环境"，所有 GL 调用都需要一个活跃的上下文才能执行 |
| GL 属性 | GL Attribute | 创建上下文前设置的参数（如 OpenGL 版本、颜色深度），决定 GPU 的工作模式 |
| 颜色深度 | Color Depth | 每个像素用多少位（bit）表示颜色，8 位 RGBA 即 32 位共约 1677 万色 |
| 深度缓冲 | Depth Buffer | 记录每个像素离摄像机多远的缓冲区，用于判断物体前后遮挡关系 |
| 模板缓冲 | Stencil Buffer | 标记像素"可画/不可画"的缓冲区，用于实现遮罩、轮廓等效果 |
| 像素格式 | Pixel Format | 操作系统用来描述每个像素的颜色/深度/模板位数的配置 |
| 双缓冲 | Double Buffering | 用两块显存交替显示，一块给用户看、一块给 GPU 画，避免画面撕裂 |

## 为什么需要 SDL2 来管理 OpenGL？

OpenGL 本身**不负责**创建窗口和管理上下文——它只是一套图形绘制接口。你需要一个"中间层"来完成以下工作：

1. **创建窗口** — 向操作系统申请一个可以绘制的窗口
2. **创建 GL 上下文** — 告诉 GPU"准备好接收 GL 指令"
3. **管理帧刷新** — 每帧结束时把后台缓冲区的内容交换到前台显示
4. **处理输入事件** — 键盘、鼠标、触摸等事件的接收和分发

没有 SDL2（或类似的库如 GLFW、SFML），你就得直接调用操作系统原生 API：

| 平台 | 原生窗口 API | 原生 GL 上下文 API |
|------|-------------|-------------------|
| Windows | Win32 `CreateWindowEx()` | WGL（`wglCreateContext`） |
| Linux | X11 / Wayland | GLX（`glXCreateContext`） |
| macOS | Cocoa `NSWindow` | CGL / NSOpenGLContext |
| Android | `ANativeWindow` | EGL（`eglCreateContext`） |

SDL2 把这四种截然不同的 API 统一封装成一套接口，让你用相同的代码在四个平台上创建窗口和 GL 上下文。

> **KrKr2 的做法：** 项目中并不直接使用 SDL2 创建 GL 上下文，而是由 Cocos2d-x 框架代劳。Cocos2d-x 内部根据平台选择 EGL / WGL / CGL，SDL2 在 KrKr2 中主要负责事件处理和平台初始化。但理解 SDL2 的 GL 管理机制是理解 Cocos2d-x 底层行为的基础。

## 设置 GL 属性

在创建窗口和上下文之前，必须先通过 `SDL_GL_SetAttribute()` 告诉 SDL2 你需要什么样的 GL 环境。这些属性**必须在创建窗口之前设置**，因为窗口创建时 SDL2 会根据这些属性选择合适的像素格式（Pixel Format，操作系统用来描述每个像素的颜色/深度/模板位数的配置）。

### 核心属性一览

```cpp
#include <SDL.h>
#include <cstdio>

// 示例 1：设置基本 GL 属性
void setupGLAttributes() {
    // ---- OpenGL 版本 ----
    // 请求 OpenGL 3.3 Core Profile
    // Major = 主版本号，Minor = 次版本号
    SDL_GL_SetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, 3);
    SDL_GL_SetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, 3);

    // Core Profile = 只允许使用现代 GL 函数，禁用废弃的固定管线函数
    // 另一个选项是 SDL_GL_CONTEXT_PROFILE_COMPATIBILITY，兼容旧代码但性能略差
    SDL_GL_SetAttribute(SDL_GL_CONTEXT_PROFILE_MASK,
                        SDL_GL_CONTEXT_PROFILE_CORE);

    // ---- 颜色缓冲 ----
    // 每个通道 8 位，RGBA 共 32 位 = 约 1677 万色
    SDL_GL_SetAttribute(SDL_GL_RED_SIZE, 8);     // 红色通道 8 位
    SDL_GL_SetAttribute(SDL_GL_GREEN_SIZE, 8);   // 绿色通道 8 位
    SDL_GL_SetAttribute(SDL_GL_BLUE_SIZE, 8);    // 蓝色通道 8 位
    SDL_GL_SetAttribute(SDL_GL_ALPHA_SIZE, 8);   // 透明通道 8 位

    // ---- 深度和模板缓冲 ----
    // 深度缓冲 24 位，足够大多数 3D 场景的远近判断精度
    SDL_GL_SetAttribute(SDL_GL_DEPTH_SIZE, 24);
    // 模板缓冲 8 位，用于遮罩效果（如只渲染某个区域内的内容）
    SDL_GL_SetAttribute(SDL_GL_STENCIL_SIZE, 8);

    // ---- 双缓冲 ----
    // 1 = 启用双缓冲（几乎所有情况都应该启用）
    SDL_GL_SetAttribute(SDL_GL_DOUBLEBUFFER, 1);

    printf("GL 属性设置完成\n");
}
```

### 各属性详解

| 属性 | 典型值 | 作用 | 不设会怎样 |
|------|--------|------|-----------|
| `SDL_GL_CONTEXT_MAJOR_VERSION` | 3 | GL 主版本号 | 系统随机选一个版本，可能是 2.1 |
| `SDL_GL_CONTEXT_MINOR_VERSION` | 3 | GL 次版本号 | 同上 |
| `SDL_GL_CONTEXT_PROFILE_MASK` | CORE / COMPATIBILITY / ES | 配置文件类型 | 默认 COMPATIBILITY |
| `SDL_GL_RED/GREEN/BLUE_SIZE` | 8 | 颜色通道位数 | 默认值因平台而异 |
| `SDL_GL_ALPHA_SIZE` | 8 | 透明通道位数 | 可能为 0（不支持透明） |
| `SDL_GL_DEPTH_SIZE` | 24 | 深度缓冲位数 | 可能为 16（远景精度不足） |
| `SDL_GL_STENCIL_SIZE` | 8 | 模板缓冲位数 | 可能为 0（无法使用模板功能） |
| `SDL_GL_DOUBLEBUFFER` | 1 | 双缓冲开关 | 可能单缓冲导致画面闪烁 |
| `SDL_GL_MULTISAMPLEBUFFERS` | 1 | 多重采样抗锯齿开关 | 无抗锯齿 |
| `SDL_GL_MULTISAMPLESAMPLES` | 4 | 采样点数量 | 无抗锯齿 |

### 跨平台版本选择策略

不同平台支持的 OpenGL 版本不同，移动端只支持 OpenGL ES：

```cpp
#include <SDL.h>
#include <cstdio>

// 示例 2：跨平台 GL 版本选择
struct GLConfig {
    int majorVersion;  // GL 主版本号
    int minorVersion;  // GL 次版本号
    int profileMask;   // Core / Compatibility / ES
    const char* name;  // 配置名称（用于日志）
};

// 根据平台选择合适的 GL 版本
GLConfig getGLConfig() {
    GLConfig config;

#if defined(__ANDROID__)
    // Android 只支持 OpenGL ES
    // ES 3.0 覆盖 Android 5.0+ 的大部分设备
    config.majorVersion = 3;
    config.minorVersion = 0;
    config.profileMask = SDL_GL_CONTEXT_PROFILE_ES;
    config.name = "OpenGL ES 3.0";

#elif defined(__APPLE__)
    // macOS 最高支持 OpenGL 4.1（Apple 已弃用 OpenGL，推荐 Metal）
    // Core Profile 是 macOS 上获取 3.2+ 的唯一方式
    config.majorVersion = 4;
    config.minorVersion = 1;
    config.profileMask = SDL_GL_CONTEXT_PROFILE_CORE;
    config.name = "OpenGL 4.1 Core (macOS)";

#elif defined(_WIN32)
    // Windows 支持到 OpenGL 4.6（取决于显卡驱动）
    config.majorVersion = 3;
    config.minorVersion = 3;
    config.profileMask = SDL_GL_CONTEXT_PROFILE_CORE;
    config.name = "OpenGL 3.3 Core (Windows)";

#elif defined(__linux__)
    // Linux 支持取决于驱动（Mesa / NVIDIA 私有驱动）
    config.majorVersion = 3;
    config.minorVersion = 3;
    config.profileMask = SDL_GL_CONTEXT_PROFILE_CORE;
    config.name = "OpenGL 3.3 Core (Linux)";

#else
    // 未知平台的保守选择
    config.majorVersion = 2;
    config.minorVersion = 1;
    config.profileMask = SDL_GL_CONTEXT_PROFILE_COMPATIBILITY;
    config.name = "OpenGL 2.1 Compatibility (fallback)";
#endif

    return config;
}

int main(int argc, char* argv[]) {
    // 初始化 SDL2 视频子系统
    if (SDL_Init(SDL_INIT_VIDEO) < 0) {
        printf("SDL 初始化失败: %s\n", SDL_GetError());
        return 1;
    }

    // 获取平台对应的 GL 配置
    GLConfig config = getGLConfig();
    printf("选择 GL 配置: %s\n", config.name);

    // 应用配置
    SDL_GL_SetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, config.majorVersion);
    SDL_GL_SetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, config.minorVersion);
    SDL_GL_SetAttribute(SDL_GL_CONTEXT_PROFILE_MASK, config.profileMask);
    SDL_GL_SetAttribute(SDL_GL_RED_SIZE, 8);
    SDL_GL_SetAttribute(SDL_GL_GREEN_SIZE, 8);
    SDL_GL_SetAttribute(SDL_GL_BLUE_SIZE, 8);
    SDL_GL_SetAttribute(SDL_GL_ALPHA_SIZE, 8);
    SDL_GL_SetAttribute(SDL_GL_DEPTH_SIZE, 24);
    SDL_GL_SetAttribute(SDL_GL_STENCIL_SIZE, 8);
    SDL_GL_SetAttribute(SDL_GL_DOUBLEBUFFER, 1);

    printf("GL 属性设置完成，准备创建窗口\n");

    SDL_Quit();
    return 0;
}
```

## 创建窗口与 GL 上下文

设置好 GL 属性后，创建窗口时需要加上 `SDL_WINDOW_OPENGL` 标志，告诉 SDL2 这个窗口要用来做 GL 渲染：

```cpp
#include <SDL.h>
#include <cstdio>

// 示例 3：创建 GL 窗口和上下文
int main(int argc, char* argv[]) {
    if (SDL_Init(SDL_INIT_VIDEO) < 0) {
        printf("SDL 初始化失败: %s\n", SDL_GetError());
        return 1;
    }

    // 设置 GL 属性（必须在创建窗口之前）
    SDL_GL_SetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, 3);
    SDL_GL_SetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, 3);
    SDL_GL_SetAttribute(SDL_GL_CONTEXT_PROFILE_MASK,
                        SDL_GL_CONTEXT_PROFILE_CORE);
    SDL_GL_SetAttribute(SDL_GL_DOUBLEBUFFER, 1);
    SDL_GL_SetAttribute(SDL_GL_DEPTH_SIZE, 24);

    // 创建窗口——注意 SDL_WINDOW_OPENGL 标志
    // SDL_WINDOW_RESIZABLE 允许用户拖拽调整窗口大小
    SDL_Window* window = SDL_CreateWindow(
        "SDL2 + OpenGL 示例",          // 窗口标题
        SDL_WINDOWPOS_CENTERED,         // X 位置（居中）
        SDL_WINDOWPOS_CENTERED,         // Y 位置（居中）
        800, 600,                       // 宽 x 高（像素）
        SDL_WINDOW_OPENGL |             // 关键：启用 GL 支持
        SDL_WINDOW_RESIZABLE            // 允许调整大小
    );

    if (window == nullptr) {
        printf("窗口创建失败: %s\n", SDL_GetError());
        SDL_Quit();
        return 1;
    }
    printf("窗口创建成功\n");

    // 创建 GL 上下文——这一步才真正初始化 GPU 的 GL 环境
    // 一个窗口可以有多个上下文，但同一时刻只能有一个"当前"上下文
    SDL_GLContext glContext = SDL_GL_CreateContext(window);
    if (glContext == nullptr) {
        printf("GL 上下文创建失败: %s\n", SDL_GetError());
        SDL_DestroyWindow(window);
        SDL_Quit();
        return 1;
    }
    printf("GL 上下文创建成功\n");

    // 验证实际获得的 GL 版本（可能和请求的不同）
    int actualMajor, actualMinor;
    SDL_GL_GetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, &actualMajor);
    SDL_GL_GetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, &actualMinor);
    printf("实际 GL 版本: %d.%d\n", actualMajor, actualMinor);

    // 启用 VSync（垂直同步）
    // 0 = 不同步（尽可能快地渲染，可能导致画面撕裂）
    // 1 = 同步（帧率锁定为显示器刷新率，通常 60 FPS）
    // -1 = 自适应 VSync（来得及就同步，来不及就不同步）
    if (SDL_GL_SetSwapInterval(1) < 0) {
        printf("VSync 设置失败: %s，回退到不同步\n", SDL_GetError());
        SDL_GL_SetSwapInterval(0);
    } else {
        printf("VSync 已启用\n");
    }

    // 清理资源
    SDL_GL_DeleteContext(glContext);
    SDL_DestroyWindow(window);
    SDL_Quit();
    printf("资源清理完成\n");
    return 0;
}
```

### 创建流程图

```
┌─────────────────────────────────────────────────┐
│ SDL_Init(SDL_INIT_VIDEO)                        │
│ 初始化视频子系统                                  │
└──────────────┬──────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────┐
│ SDL_GL_SetAttribute(...)                        │
│ 设置 GL 版本、颜色深度、双缓冲等                   │
│ ⚠️ 必须在创建窗口之前调用                          │
└──────────────┬──────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────┐
│ SDL_CreateWindow(..., SDL_WINDOW_OPENGL)        │
│ 创建窗口，SDL2 根据属性选择像素格式                 │
└──────────────┬──────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────┐
│ SDL_GL_CreateContext(window)                    │
│ 创建 GL 上下文，GPU 准备就绪                       │
└──────────────┬──────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────┐
│ SDL_GL_SetSwapInterval(1)                       │
│ 启用 VSync                                      │
└──────────────┬──────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────┐
│ 渲染循环：glClear → 绘制 → SDL_GL_SwapWindow    │
└─────────────────────────────────────────────────┘
```

## 本节小结

- OpenGL 本身不管窗口和上下文创建，需要 SDL2（或 GLFW/SFML）充当中间层
- `SDL_GL_SetAttribute()` **必须在 `SDL_CreateWindow()` 之前调用**，设置 GL 版本、颜色深度、缓冲区配置等
- 跨平台差异：Windows/Linux 用 OpenGL Core 3.3+，macOS 最高 4.1 Core，Android 用 OpenGL ES 3.0
- `SDL_CreateWindow()` 必须包含 `SDL_WINDOW_OPENGL` 标志
- `SDL_GL_CreateContext()` 真正初始化 GPU 环境，创建后可通过 `SDL_GL_GetAttribute()` 验证实际配置

## 下一步

→ 继续阅读 [07-渲染循环与帧率控制](./07-渲染循环与帧率控制.md)，学习双缓冲与 VSync 的工作原理、完整渲染循环的编写，以及可变/固定时间步长帧率控制策略。
