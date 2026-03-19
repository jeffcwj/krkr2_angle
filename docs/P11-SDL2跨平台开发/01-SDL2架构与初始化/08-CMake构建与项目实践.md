# CMake构建与项目实践

> **所属模块：** P11-SDL2跨平台开发
> **前置知识：** [07-渲染循环与帧率控制](./07-渲染循环与帧率控制.md)
> **预计阅读时间：** 18 分钟

## 本节目标

读完本节后，你将能够：
1. 用 CMake 构建跨四平台（Windows / Linux / macOS / Android）的 SDL2 + OpenGL 项目
2. 理解不同平台链接 OpenGL 库的差异及处理方式
3. 对照 KrKr2 项目源码，理解 Cocos2d-x 如何代替 SDL2 管理 GL 上下文
4. 排查 SDL2 + OpenGL 开发中的常见错误

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| CMake Target | CMake 目标 | CMake 中用 `add_executable` 或 `add_library` 创建的构建单元 |
| find_package | find_package | CMake 内置命令，自动查找系统中安装的库并设置头文件路径和链接信息 |
| Framework | 框架 | macOS/iOS 特有的库分发格式，把头文件、二进制和资源打包在一个 .framework 目录中 |
| NDK | Native Development Kit | Android 平台编写 C/C++ 代码所需的工具链，包含交叉编译器和系统库 |
| GLESv3 | OpenGL ES 3.0 | OpenGL 的移动端子集，功能比桌面 OpenGL 精简但覆盖了主要渲染需求 |

## CMake 构建配置

下面是一个完整的 CMakeLists.txt，展示如何用 CMake 编译 SDL2 + OpenGL 项目并覆盖四平台差异：

```cmake
# 示例 1：完整的跨平台 SDL2 + OpenGL CMake 配置
cmake_minimum_required(VERSION 3.20)
project(SDL2OpenGLDemo LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)

# 查找 SDL2（vcpkg 安装时会自动设好搜索路径）
find_package(SDL2 REQUIRED)

add_executable(gl_demo main.cpp)

# 链接 SDL2——兼容新旧两种 CMake 写法
if(TARGET SDL2::SDL2)
    # SDL2 2.24+ 的导入目标写法
    # SDL2::SDL2main 在 Windows 把 WinMain 转为标准 main
    target_link_libraries(gl_demo PRIVATE SDL2::SDL2 SDL2::SDL2main)
else()
    # 旧版 SDL2 的变量写法
    target_include_directories(gl_demo PRIVATE ${SDL2_INCLUDE_DIRS})
    target_link_libraries(gl_demo PRIVATE ${SDL2_LIBRARIES})
endif()

# 链接 OpenGL——各平台差异处理
if(ANDROID)
    # Android NDK 自带 GLESv3 和 EGL，不需要 find_package
    target_link_libraries(gl_demo PRIVATE GLESv3 EGL)
elseif(APPLE)
    # macOS 使用 Framework 方式链接
    find_library(OPENGL_FRAMEWORK OpenGL REQUIRED)
    target_link_libraries(gl_demo PRIVATE ${OPENGL_FRAMEWORK})
else()
    # Windows / Linux：CMake 内置的 FindOpenGL 模块
    find_package(OpenGL REQUIRED)
    target_link_libraries(gl_demo PRIVATE OpenGL::GL)
endif()

# Windows：Release 模式隐藏控制台窗口
if(WIN32)
    set_target_properties(gl_demo PROPERTIES
        WIN32_EXECUTABLE $<CONFIG:Release>)
endif()
```

### 项目目录结构

一个完整的 SDL2 + OpenGL 项目目录如下：

```
sdl2-opengl-demo/
├── CMakeLists.txt          # 上面的构建配置
├── main.cpp                # 渲染循环代码（上一节的示例 2）
├── vcpkg.json              # vcpkg 依赖清单（可选）
└── cmake/
    └── vcpkg_android.cmake # Android 交叉编译的 vcpkg 工具链（可选）
```

如果使用 vcpkg 管理 SDL2 依赖，`vcpkg.json` 内容：

```json
{
  "name": "sdl2-opengl-demo",
  "version-string": "1.0.0",
  "dependencies": [
    "sdl2"
  ]
}
```

### 构建命令（四平台）

```bash
# Windows（也可用 "Visual Studio 17 2022" 生成器）
cmake -B build -G Ninja && cmake --build build

# Linux（需安装 libsdl2-dev 和 mesa GL 头文件）
cmake -B build -G Ninja && cmake --build build

# macOS（系统自带 OpenGL.framework，只需 brew install sdl2）
cmake -B build -G Ninja && cmake --build build

# Android（需设置 ANDROID_NDK 环境变量）
cmake -B build -G Ninja \
    -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
    -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-24 -DANDROID=ON
cmake --build build
```

### 平台差异汇总

| 差异项 | Windows | Linux | macOS | Android |
|--------|---------|-------|-------|---------|
| GL 头文件 | `<GL/gl.h>` | `<GL/gl.h>` | `<OpenGL/gl3.h>` | `<GLES3/gl3.h>` |
| GL 链接库 | `OpenGL::GL` | `OpenGL::GL` | `OpenGL.framework` | `GLESv3 EGL` |
| SDL2 安装 | vcpkg / 手动 | apt / yum | brew / vcpkg | NDK 自带 |
| 控制台窗口 | 需隐藏(Release) | 无此问题 | 无此问题 | 无此问题 |
| GL 版本 | 通常 4.6 | 看显卡驱动 | 最高 4.1 | ES 3.0/3.1 |

macOS 的 OpenGL 最高只到 4.1（Apple 已弃用 OpenGL，推荐 Metal）。KrKr2 通过 Cocos2d-x 抽象层规避了这个问题。

## 对照项目源码

### KrKr2 的 GL 管理方式

KrKr2 项目**不直接**调用 SDL2 的 GL 管理函数（`SDL_GL_CreateContext` 等）。源码中搜索 `SDL_GL`、`SDL_CreateWindow`、`GLContext`、`GL_SetAttribute` 均无匹配。GL 上下文由 **Cocos2d-x** 全权管理：

| 功能 | SDL2 方式（本章所学） | KrKr2 实际方式 |
|------|-----------|---------------|
| 创建窗口 | `SDL_CreateWindow()` | Cocos2d-x `Director::getInstance()` |
| 创建 GL 上下文 | `SDL_GL_CreateContext()` | Cocos2d-x 内部调用平台原生 API |
| VSync 设置 | `SDL_GL_SetSwapInterval()` | Cocos2d-x `Director::setAnimationInterval()` |
| 帧交换 | `SDL_GL_SwapWindow()` | Cocos2d-x 内部在每帧末尾自动调用 |
| 视口管理 | 手动 `glViewport()` | Cocos2d-x `GLView` 自动管理 |

来看 `AppDelegate.cpp` 中的启动流程：

```cpp
// 示例 2：AppDelegate.cpp 中的关键流程（简化）
// 源文件：cpp/core/environ/cocos2d/AppDelegate.cpp
bool AppDelegate::applicationDidFinishLaunching() {
    // Cocos2d-x 的 Director 是整个引擎的核心单例
    // 它管理场景、渲染循环、帧率——相当于我们手写的 main 循环
    auto director = cocos2d::Director::getInstance();
    auto glview = director->getOpenGLView();

    if (!glview) {
        // Cocos2d-x 在 create() 内部完成了：
        //   1. SDL_Init (或平台原生初始化)
        //   2. 设置 GL 属性
        //   3. 创建窗口
        //   4. 创建 GL 上下文
        // 不需要手动调用 SDL_GL_CreateContext
        glview = cocos2d::GLViewImpl::create("KrKr2");
        director->setOpenGLView(glview);
    }

    // 设置帧率间隔（等效于 VSync + 帧率控制）
    // 1/60 = 每帧 16.67ms，即 60 FPS
    director->setAnimationInterval(1.0f / 60.0f);

    // ... 启动游戏场景
    return true;
}
```

### SDL2 在 KrKr2 中的实际角色

那 SDL2 在 KrKr2 中到底做什么？查看 `cpp/core/environ/sdl/tvpsdl.cpp`（约 50 行）：

```cpp
// 示例 3：tvpsdl.cpp 核心内容（简化）
// 源文件：cpp/core/environ/sdl/tvpsdl.cpp
// 只负责引擎子系统注册，不直接调用 SDL2 API
#include "Application.h"
#include "tjsCommHead.h"
// SDL2 作为 Cocos2d-x 的底层后端之一
// 实际的 SDL2 调用发生在 Cocos2d-x 内部
```

SDL2 在 KrKr2 中是 Cocos2d-x 的**可选后端**，架构如下：

```
┌──────────────────────────────────────┐
│ KrKr2 引擎核心                        │
│ （TJS2 / 渲染 / 音频 / 插件）         │
├──────────────────────────────────────┤
│ Cocos2d-x（Director / GLView / ...） │
├──────────┬─────────┬─────────────────┤
│ Win32    │ Cocoa   │ SDL2 后端       │
│ 后端     │ 后端    │ （Linux 等）    │
└──────────┴─────────┴─────────────────┘
```

### 为什么要学 SDL2 的 GL 管理

即便 KrKr2 不直接使用 SDL2 管理 GL，学习 SDL2 的 GL 接口仍然重要：

1. **理解底层原理** — Cocos2d-x 内部做的事和 SDL2 完全一样，学了 SDL2 就能理解 Cocos2d-x 内部在干什么
2. **调试能力** — GL 问题（黑屏、闪烁）需要理解底层。知道 `SDL_GL_SetSwapInterval` 和 `Director::setAnimationInterval` 的对应关系才能定位问题
3. **扩展能力** — 项目中期目标之一是替换 Cocos2d-x UI 层，届时可能需要自己管理 GL 上下文

## 常见错误与排查

### 错误 1：GL 上下文创建失败

**症状：** `SDL_GL_CreateContext` 返回 `NULL`

**排查步骤：**

```cpp
// 示例 4：GL 上下文创建失败的诊断代码
SDL_GLContext ctx = SDL_GL_CreateContext(window);
if (!ctx) {
    printf("错误: %s\n", SDL_GetError());
    // 常见错误消息及解决方案：

    // "Could not create GL context"
    //   原因：请求的 GL 版本不支持（如在 macOS 上请求 4.6）
    //   解决：降低版本号，如 3.3 → 3.1 → 2.1

    // "No matching GL pixel format available"
    //   原因：颜色/深度配置不支持（如请求 32 位深度缓冲）
    //   解决：降低深度位数，如 24 → 16
}
```

**GL 版本降级策略：**

```cpp
// 示例 5：尝试多个 GL 版本，直到成功
struct VersionFallback {
    int major;        // 主版本号
    int minor;        // 次版本号
    int profile;      // 配置文件（Core 或 Compatibility）
    const char* name; // 人类可读名称
};

// 从最新到最旧排列
VersionFallback versions[] = {
    {4, 6, SDL_GL_CONTEXT_PROFILE_CORE,          "4.6 Core"},
    {4, 1, SDL_GL_CONTEXT_PROFILE_CORE,          "4.1 Core"},   // macOS 最高支持
    {3, 3, SDL_GL_CONTEXT_PROFILE_CORE,          "3.3 Core"},
    {3, 1, SDL_GL_CONTEXT_PROFILE_CORE,          "3.1 Core"},
    {2, 1, SDL_GL_CONTEXT_PROFILE_COMPATIBILITY, "2.1 Compat"}, // 最后手段
};

SDL_GLContext createBestContext(SDL_Window* window) {
    for (auto& v : versions) {
        SDL_GL_SetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, v.major);
        SDL_GL_SetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, v.minor);
        SDL_GL_SetAttribute(SDL_GL_CONTEXT_PROFILE_MASK, v.profile);

        SDL_GLContext ctx = SDL_GL_CreateContext(window);
        if (ctx) {
            printf("成功创建 GL %s 上下文\n", v.name);
            return ctx;
        }
        printf("GL %s 不支持，尝试下一个...\n", v.name);
    }
    printf("所有 GL 版本都不支持！\n");
    return nullptr;
}
```

**注意：** 改变 GL 属性后可能需要重新创建窗口，因为像素格式在 `SDL_CreateWindow` 时就已确定。

### 错误 2：窗口创建时忘记 SDL_WINDOW_OPENGL

**症状：** `SDL_GL_CreateContext` 返回 NULL，报 "The specified window isn't an OpenGL window"

```cpp
// ❌ 缺少 SDL_WINDOW_OPENGL，后续 GL 上下文创建会失败
SDL_Window* w = SDL_CreateWindow("test",
    SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED,
    800, 600, SDL_WINDOW_SHOWN);

// ✅ 必须在创建窗口时声明是 GL 窗口
SDL_Window* w = SDL_CreateWindow("test",
    SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED,
    800, 600, SDL_WINDOW_OPENGL | SDL_WINDOW_SHOWN);
```

### 错误 3：属性设置在窗口创建之后

**症状：** GL 属性没生效（深度缓冲不工作、颜色精度不对）。`SDL_GL_SetAttribute` 必须在 `SDL_CreateWindow` **之前**调用——窗口创建时像素格式就已确定。

```cpp
// ❌ 顺序反了
SDL_Window* w = SDL_CreateWindow(..., SDL_WINDOW_OPENGL);
SDL_GL_SetAttribute(SDL_GL_DEPTH_SIZE, 24);  // 太晚，无效

// ✅ 先设属性再创建窗口
SDL_GL_SetAttribute(SDL_GL_DEPTH_SIZE, 24);
SDL_GL_SetAttribute(SDL_GL_DOUBLEBUFFER, 1);
SDL_Window* w = SDL_CreateWindow(..., SDL_WINDOW_OPENGL);
```

### 错误 4：忘记初始化 SDL_INIT_VIDEO

**症状：** 所有 SDL 视频/窗口函数都失败。视频子系统是创建窗口和 GL 上下文的前提。

```cpp
// ❌ 初始化了其他子系统但忘了 VIDEO
SDL_Init(SDL_INIT_AUDIO);
SDL_Window* w = SDL_CreateWindow(...);  // 失败！

// ✅ 必须包含 SDL_INIT_VIDEO
SDL_Init(SDL_INIT_VIDEO | SDL_INIT_AUDIO);
```

## 本节小结

- CMake 构建需要分平台链接不同的 GL 库：桌面用 `OpenGL::GL`，Android 用 `GLESv3 + EGL`，macOS 用 `OpenGL.framework`
- SDL2 的 CMake 目标名称因版本而异：≥2.24 用 `SDL2::SDL2`，旧版用 `${SDL2_LIBRARIES}` 变量
- KrKr2 项目中由 Cocos2d-x 全权代理 GL 管理——SDL2 是 Cocos2d-x 的可选后端之一
- SDL2 在 KrKr2 中的角色：`tvpsdl.cpp` 仅注册引擎子系统，实际 SDL2 调用发生在 Cocos2d-x 内部
- 常见错误四大类：GL 版本不支持、忘记 `SDL_WINDOW_OPENGL`、属性设置顺序错误、未初始化视频子系统

## 练习题与答案

### 题目 1：编写跨平台 GL 头文件包含

编写一个 `gl_compat.h` 头文件，根据当前平台自动包含正确的 OpenGL 头文件，并定义一个 `GL_VERSION_LABEL` 宏表示当前使用的 GL 版本字符串。

<details>
<summary>查看答案</summary>

```cpp
// gl_compat.h — 跨平台 GL 头文件适配
#ifndef GL_COMPAT_H
#define GL_COMPAT_H

#if defined(__ANDROID__)
    // Android 使用 OpenGL ES 3.0
    #include <GLES3/gl3.h>
    #include <EGL/egl.h>
    #define GL_VERSION_LABEL "OpenGL ES 3.0"
#elif defined(__APPLE__)
    // macOS 使用系统自带的 GL 头文件
    // gl3.h 对应 Core Profile 3.2+
    #include <OpenGL/gl3.h>
    #define GL_VERSION_LABEL "OpenGL 4.1 Core (macOS)"
#elif defined(_WIN32)
    // Windows 通常需要 GLAD 或 GLEW 来加载现代 GL 函数
    // 最基本的情况下用系统 gl.h（仅 GL 1.1）
    #include <GL/gl.h>
    #define GL_VERSION_LABEL "OpenGL (Windows, need GLAD/GLEW for modern)"
#else
    // Linux：mesa 提供 GL 头文件
    #include <GL/gl.h>
    #define GL_VERSION_LABEL "OpenGL (Linux)"
#endif

// 便利宏：打印当前 GL 配置
#include <cstdio>
inline void printGLInfo() {
    printf("编译时 GL 配置: %s\n", GL_VERSION_LABEL);
    // 运行时版本需要在 GL 上下文创建后调用 glGetString
}

#endif // GL_COMPAT_H
```

使用方式：
```cpp
#include "gl_compat.h"
#include <SDL.h>

int main(int argc, char* argv[]) {
    SDL_Init(SDL_INIT_VIDEO);
    printGLInfo();
    // ... 创建窗口和上下文后，还可以调用：
    // printf("运行时 GL 版本: %s\n", glGetString(GL_VERSION));
    return 0;
}
```

</details>

### 题目 2：补全 GL 降级初始化代码

下面的代码尝试创建 GL 3.3 Core 上下文，如果失败则降级到 GL 2.1 Compatibility。请补全 `TODO` 处的代码：

```cpp
SDL_GL_SetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, 3);
SDL_GL_SetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, 3);
SDL_GL_SetAttribute(SDL_GL_CONTEXT_PROFILE_MASK,
                    SDL_GL_CONTEXT_PROFILE_CORE);

SDL_GLContext ctx = SDL_GL_CreateContext(window);
if (!ctx) {
    // TODO: 降级到 GL 2.1 Compatibility 并重试
}
```

<details>
<summary>查看答案</summary>

```cpp
SDL_GL_SetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, 3);
SDL_GL_SetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, 3);
SDL_GL_SetAttribute(SDL_GL_CONTEXT_PROFILE_MASK,
                    SDL_GL_CONTEXT_PROFILE_CORE);

SDL_GLContext ctx = SDL_GL_CreateContext(window);
if (!ctx) {
    printf("GL 3.3 Core 不支持: %s，降级到 2.1\n",
           SDL_GetError());

    // 重新设置属性为 GL 2.1 Compatibility
    SDL_GL_SetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, 2);
    SDL_GL_SetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, 1);
    SDL_GL_SetAttribute(SDL_GL_CONTEXT_PROFILE_MASK,
                        SDL_GL_CONTEXT_PROFILE_COMPATIBILITY);

    // 注意：需要销毁旧窗口并重新创建
    // 因为像素格式在窗口创建时就确定了
    SDL_DestroyWindow(window);
    window = SDL_CreateWindow("GL 2.1 降级",
        SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED,
        800, 600, SDL_WINDOW_OPENGL);

    ctx = SDL_GL_CreateContext(window);
    if (!ctx) {
        printf("GL 2.1 也失败: %s\n", SDL_GetError());
        // 此时应该退出程序或使用软件渲染
    }
}
```

**关键点：** 改变 GL 属性后需要重新创建窗口，因为像素格式在 `SDL_CreateWindow` 时就已确定。

</details>

### 题目 3：写一个 GL 能力探测工具

编写一个程序，探测当前系统的 GL 版本、缓冲区配置和 VSync 支持。多重采样（Multisample）是一种通过在每个像素内取多个采样点来实现抗锯齿的技术。

<details>
<summary>查看答案</summary>

```cpp
#include <SDL.h>
#include <cstdio>

int main(int argc, char* argv[]) {
    SDL_Init(SDL_INIT_VIDEO);
    // 创建隐藏窗口纯探测用，不设 GL 属性限制让系统选最佳配置
    SDL_Window* w = SDL_CreateWindow("GL探测", SDL_WINDOWPOS_CENTERED,
        SDL_WINDOWPOS_CENTERED, 320, 240,
        SDL_WINDOW_OPENGL | SDL_WINDOW_HIDDEN);
    SDL_GLContext ctx = SDL_GL_CreateContext(w);
    if (!ctx) { printf("失败: %s\n", SDL_GetError()); return 1; }

    // SDL_GL_GetAttribute 读取实际生效的配置（可能与请求值不同）
    int major, minor, r, g, b, a, depth, stencil, db, msB, msS;
    SDL_GL_GetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, &major);
    SDL_GL_GetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, &minor);
    SDL_GL_GetAttribute(SDL_GL_RED_SIZE, &r);
    SDL_GL_GetAttribute(SDL_GL_GREEN_SIZE, &g);
    SDL_GL_GetAttribute(SDL_GL_BLUE_SIZE, &b);
    SDL_GL_GetAttribute(SDL_GL_ALPHA_SIZE, &a);
    SDL_GL_GetAttribute(SDL_GL_DEPTH_SIZE, &depth);
    SDL_GL_GetAttribute(SDL_GL_STENCIL_SIZE, &stencil);
    SDL_GL_GetAttribute(SDL_GL_DOUBLEBUFFER, &db);
    SDL_GL_GetAttribute(SDL_GL_MULTISAMPLEBUFFERS, &msB);
    SDL_GL_GetAttribute(SDL_GL_MULTISAMPLESAMPLES, &msS);

    printf("GL %d.%d | 颜色 R%dG%dB%dA%d | 深度 %d | 模板 %d | "
           "双缓冲 %s | 多重采样 %s\n",
           major, minor, r, g, b, a, depth, stencil,
           db ? "是" : "否", msB ? "是" : "否");
    // 检查自适应 VSync
    printf("自适应VSync: %s\n",
           SDL_GL_SetSwapInterval(-1) == 0 ? "支持" : "不支持");

    SDL_GL_DeleteContext(ctx);
    SDL_DestroyWindow(w);
    SDL_Quit();
}
```

</details>

## 下一步

本章（01-SDL2架构与初始化）全部结束。你已掌握 SDL2 的历史、子系统架构、初始化策略、OpenGL 集成、渲染循环、帧率控制及跨平台构建。→ 继续阅读 [第二章：窗口与事件循环](../02-窗口与事件循环/01-窗口创建与属性配置.md)。
