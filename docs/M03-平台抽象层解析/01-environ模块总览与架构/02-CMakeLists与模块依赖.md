## 3. CMakeLists.txt 深度解读

environ 的 CMakeLists.txt 是理解整个模块构建方式的关键。让我们逐段分析：

### 3.1 完整源码与注释

```cmake
# 文件: krkr2/cpp/core/environ/CMakeLists.txt

# 声明静态库目标
add_library(environ STATIC
    Application.cpp          # 核心：应用生命周期
    DetectCPU.cpp           # CPU 特性检测

    # ======= 平台条件编译 =======
    # 使用生成器表达式 $<$<BOOL:${VAR}>:...> 选择平台文件
    # 这些在 CMake 配置阶段不展开，在生成阶段根据实际值展开

    # Windows 平台
    $<$<BOOL:${WIN32}>:
        win32/Platform.cpp
        win32/SystemControl.cpp
    >

    # Linux 平台（注意：Android NDK 编译时也定义了 LINUX，需要排除）
    $<$<AND:$<BOOL:${LINUX}>,$<NOT:$<BOOL:${ANDROID}>>>:
        linux/Platform.cpp
    >

    # Android 平台（同时包含 linux/Platform.cpp 和 android/AndroidUtils.cpp）
    $<$<BOOL:${ANDROID}>:
        linux/Platform.cpp          # 复用 Linux 的部分实现
        android/AndroidUtils.cpp    # Android 特有的 JNI 实现
    >

    # Cocos2d-x 桥接层（所有非桌面Linux平台都使用）
    $<$<NOT:$<AND:$<BOOL:${LINUX}>,$<NOT:$<BOOL:${ANDROID}>>>>:
        cocos2d/AppDelegate.cpp
        cocos2d/MainScene.cpp
        cocos2d/CustomFileUtils.cpp
        cocos2d/CCKeyCodeConv.cpp
        cocos2d/YUVSprite.cpp
    >

    # macOS 平台特殊文件
    $<$<BOOL:${APPLE}>:
        cocos2d/CustomFileUtils.mm   # Objective-C++ 文件
    >

    # SDL 集成（桌面 Linux 使用 SDL 而非 Cocos2d-x）
    $<$<AND:$<BOOL:${LINUX}>,$<NOT:$<BOOL:${ANDROID}>>>:
        sdl/tvpsdl.cpp
    >

    # 配置管理器（所有平台共用）
    ConfigManager/GlobalConfigManager.cpp
    ConfigManager/IndividualConfigManager.cpp
    ConfigManager/LocaleConfigManager.cpp
)

# ======= 链接依赖 =======
target_link_libraries(environ
    PUBLIC
        tjs2            # TJS2 脚本引擎（PUBLIC 因为 Application.h 暴露了 TJS2 类型）

    PRIVATE
        base            # 基础工具库
        visual          # 渲染子系统
        sound           # 音频子系统
        movie           # 视频子系统
        # 其他 core 模块...
)
```

### 3.2 关键设计决策解析

**为什么用 STATIC 而非 INTERFACE？**

```
INTERFACE 库：纯头文件库，不编译任何源文件
STATIC 库：编译源文件为 .a/.lib，链接时打包进最终可执行文件
```

environ 有大量 `.cpp` 实现文件（Application.cpp、各平台 Platform.cpp），所以必须是 STATIC 库。

**为什么 tjs2 是 PUBLIC 链接？**

```cpp
// Application.h 中暴露了 TJS2 的类型
#include "tjs2/tjsTypes.h"  // tjs_uint32 等类型
class tTVPApplication {
    tTJSVariant* someField;  // TJS2 的 Variant 类型
    // ...
};
```

任何 `#include "Application.h"` 的翻译单元都需要看到 TJS2 的头文件，所以 tjs2 必须 PUBLIC 传播。

**为什么 visual/sound/movie 是 PRIVATE？**

这些模块只在 `Application.cpp` 的实现中被调用（如初始化渲染、启动音频），不暴露在头文件中，所以 PRIVATE 足够。

### 3.3 平台文件选择逻辑

用一张表格总结各平台实际编译的文件：

| 文件 | Windows | Linux (桌面) | Android | macOS |
|------|---------|-------------|---------|-------|
| Application.cpp | ✅ | ✅ | ✅ | ✅ |
| DetectCPU.cpp | ✅ | ✅ | ✅ | ✅ |
| win32/Platform.cpp | ✅ | ❌ | ❌ | ❌ |
| win32/SystemControl.cpp | ✅ | ❌ | ❌ | ❌ |
| linux/Platform.cpp | ❌ | ✅ | ✅ | ❌ |
| android/AndroidUtils.cpp | ❌ | ❌ | ✅ | ❌ |
| cocos2d/*.cpp | ✅ | ❌ | ✅ | ✅ |
| cocos2d/*.mm | ❌ | ❌ | ❌ | ✅ |
| sdl/tvpsdl.cpp | ❌ | ✅ | ❌ | ❌ |
| ConfigManager/*.cpp | ✅ | ✅ | ✅ | ✅ |

**重要发现**：
- **Android 复用 linux/Platform.cpp** —— Android NDK 基于 Linux 内核，所以内存查询（`/proc/meminfo`）、文件操作（`stat`/`unlink`）等可以直接复用 Linux 实现。Android 特有的功能（如 JNI 弹窗、存储路径发现）则放在 `AndroidUtils.cpp`。
- **桌面 Linux 用 SDL，其他平台用 Cocos2d-x** —— 这是因为桌面 Linux 可以直接用 SDL2 管理窗口和 GL 上下文，不需要 Cocos2d-x 这个"重量级"框架。
- **macOS 编译 .mm 文件** —— Objective-C++ 文件用于调用 Apple 的 Cocoa API（如 `NSBundle` 获取资源路径）。

---

## 4. 模块间依赖关系

environ 模块处于 KrKr2 依赖图的**中间层**——它依赖底层的 core 模块，同时被上层的平台入口依赖：

```
                    ┌─────────────┐
                    │  krkr2 可执行 │
                    │  (main.cpp)  │
                    └──────┬──────┘
                           │ 链接
                           ▼
                    ┌─────────────┐
                    │   environ   │
                    │ (STATIC 库) │
                    └──┬────┬─────┘
                       │    │
              PUBLIC   │    │ PRIVATE
              链接     │    │ 链接
                       ▼    ▼
              ┌──────┐ ┌──────────────────────────┐
              │ tjs2 │ │ base, visual, sound,     │
              │      │ │ movie, ...               │
              └──────┘ └──────────────────────────┘
```

### 4.1 为什么 environ 依赖几乎所有 core 模块？

因为 `Application.cpp` 负责**引擎初始化**，它需要：
- 调用 `tjs2` 初始化脚本引擎
- 调用 `visual` 初始化渲染子系统（设置画面大小、创建图层管理器）
- 调用 `sound` 初始化音频（打开 OpenAL 设备）
- 调用 `movie` 初始化视频播放器（FFmpeg 解码器）
- 调用 `base` 使用基础工具（日志、字符串转换、文件 I/O）

这种"扇出"（fan-out）依赖模式是游戏引擎的典型特征——应用层需要协调所有子系统的生命周期。

---

## 5. DetectCPU.cpp 分析

CPU 特性检测是 environ 模块中一个独立的功能点：

```cpp
// DetectCPU.cpp 核心逻辑（简化）
// 文件: krkr2/cpp/core/environ/DetectCPU.cpp

// 目前大部分功能被 stub（空实现）
// 唯一有实际逻辑的是 Apple ARM 平台的 NEON 检测

#if defined(__APPLE__) && defined(__ARM_NEON__)
// Apple Silicon（M1/M2）始终支持 NEON
static bool s_hasNeon = true;
#else
// 其他平台：暂未实现完整的 CPUID 检测
static bool s_hasSSE2 = false;
static bool s_hasAVX2 = false;
#endif

// 全局初始化函数
void TVPDetectCPU() {
#if defined(__APPLE__) && defined(__ARM_NEON__)
    // Apple ARM: NEON 始终可用，无需检测
    // 直接设置相关的函数指针为 NEON 版本
#elif defined(__x86_64__) || defined(_M_X64)
    // x86-64: TODO - 应该用 CPUID 检测 SSE2/AVX2
    // 当前为空实现，使用纯 C 回退路径
#endif
}
```

**为什么大部分被 stub？**

KrKr2 项目仍在积极开发中。CPU 检测的完整实现需要：
1. x86 CPUID 指令解析（已在 P05 中讲解）
2. 函数指针表初始化（将 `TVPAlphaBlend` 等指向最优实现）
3. 运行时特性协商

目前项目优先确保跨平台功能正确，性能优化（SIMD 分发）是后续任务。这也解释了为什么 `visual/tvpgl.cpp` 中的大部分像素操作仍然使用纯 C 实现。

---

## 6. SDL 集成与 Cocos2d-x 的关系

environ 模块有两套"宿主框架"方案：

| 框架 | 适用平台 | 提供功能 |
|------|----------|----------|
| **Cocos2d-x** | Windows、macOS、Android | 窗口管理、GL 上下文、输入事件、主循环、文件系统 |
| **SDL2** | 桌面 Linux | 窗口管理、GL 上下文、输入事件 |

```
桌面 Linux 路径:
  main.cpp → SDL_Init → 创建 SDL_Window → GL 上下文 → 引擎主循环

其他平台路径:
  main.cpp → cocos2d::Application::run() → AppDelegate
           → Director → GLView → MainScene::update → 引擎主循环
```

**为什么桌面 Linux 不用 Cocos2d-x？**

Cocos2d-x 在桌面 Linux 上的支持不如 SDL2 成熟。SDL2 是 Linux 上事实上的标准多媒体库，与 X11/Wayland 的集成更好，依赖更轻量。

`sdl/tvpsdl.cpp` 目前只有 50 行，是一个桩（stub）文件，包含 SDL 的初始化和基本事件循环框架。完整的 SDL 集成仍在开发中。

---

