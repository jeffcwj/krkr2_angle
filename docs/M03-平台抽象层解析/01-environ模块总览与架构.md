# environ 模块总览与架构

> **所属模块**：M03-平台抽象层解析 · 第 1 章  
> **前置知识**：M01（项目导览）、P03（跨平台 C++ 开发）、CMake 基础  
> **预计阅读时间**：45 分钟

---

## 本节目标

1. 理解 `environ` 模块在 KrKr2 项目中的定位——它是引擎与操作系统之间的"翻译层"
2. 掌握 environ 目录的完整结构和每个子目录/文件的职责
3. 理解 environ 的 CMakeLists.txt 如何用生成器表达式实现条件编译
4. 看懂 environ 模块与其他 core 模块的链接关系
5. 建立整个平台抽象层的架构全景图

---

## 1. environ 模块在 KrKr2 中的定位

### 1.1 为什么需要平台抽象层

KrKr2 是一个跨平台的 KiriKiri 引擎模拟器，需要在 Windows、Linux、macOS、Android 四个平台上运行。这些平台在以下方面存在根本差异：

| 维度 | Windows | Linux | macOS | Android |
|------|---------|-------|-------|---------|
| 窗口系统 | Win32 API | X11/Wayland | Cocoa (AppKit) | Activity + SurfaceView |
| 文件路径 | `C:\Users\...` | `/home/...` | `/Users/...` | `/data/data/...` |
| 消息弹窗 | `MessageBoxW` | GTK `gtk_dialog_run` | NSAlert | JNI → Toast/Dialog |
| 内存查询 | `GlobalMemoryStatusEx` | `/proc/meminfo` | `sysctl` | `ActivityManager` |
| 输入法 | IMM32 | IBus/Fcitx | 系统输入法 | JNI → InputMethodManager |
| 文件统计 | `_wstat64` | `stat` | `stat` | `stat`（NDK） |

如果每个功能模块（渲染、音频、脚本引擎）都自己处理这些差异，代码会变成一团无法维护的 `#ifdef` 意大利面。

**environ 模块的解决方案**：将所有平台差异集中到一个模块中，对外提供统一的 C 风格接口。其他模块只需调用 `TVPGetMemoryInfo()`，不需要关心它在 Windows 上读 `MEMORYSTATUS` 还是在 Linux 上解析 `/proc/meminfo`。

### 1.2 架构层次

```
┌─────────────────────────────────────────────────────┐
│                    应用层                             │
│  platforms/windows/main.cpp                          │
│  platforms/linux/main.cpp                            │
│  platforms/android/cpp/krkr2_android.cpp             │
└──────────────────┬──────────────────────────────────┘
                   │ 创建 Application, 启动引擎
                   ▼
┌─────────────────────────────────────────────────────┐
│               environ 模块（平台抽象层）               │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Application  │  │  Platform.h  │  │ ConfigMgr  │ │
│  │ 生命周期管理 │  │  平台接口    │  │ 配置管理   │ │
│  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ │
│         │                 │                │        │
│  ┌──────┴─────────────────┴────────────────┴──────┐ │
│  │          平台实现层（条件编译选择）              │ │
│  │  win32/  │  linux/  │  android/  │  cocos2d/   │ │
│  └────────────────────────────────────────────────┘ │
└──────────────────┬──────────────────────────────────┘
                   │ 依赖
                   ▼
┌─────────────────────────────────────────────────────┐
│                    core 其他模块                      │
│  tjs2（脚本引擎）  visual（渲染）  sound（音频）      │
│  movie（视频）     base（基础库）                     │
└─────────────────────────────────────────────────────┘
```

**关键设计决策**：environ 是一个 **STATIC 库**（不是 INTERFACE 库），它编译为 `.a`/`.lib` 静态库，被 `krkr2` 可执行目标链接。它 PUBLIC 链接 `tjs2`（因为 Application.h 中暴露了 TJS2 的类型），PRIVATE 链接其他所有 core 模块。

---

## 2. 目录结构详解

让我们逐一分析 environ 目录下的每个文件和子目录：

```
krkr2/cpp/core/environ/
│
├── Application.h          ← tTVPApplication 类声明（第 3 章详解）
├── Application.cpp        ← 937 行，应用生命周期实现
│
├── Platform.h             ← 平台抽象接口声明（第 2 章详解）
│                            纯 C 风格函数，无类、无虚函数
│
├── DetectCPU.cpp          ← CPU 特性检测（SIMD 能力）
│                            目前大部分被 stub，仅 Apple ARM 有实际检测
│
├── CMakeLists.txt         ← 88 行，核心构建配置
│
├── win32/                 ← Windows 平台实现（31 个文件）
│   ├── Platform.cpp       ← Platform.h 的 Win32 实现
│   ├── SystemControl.cpp  ← 窗口系统控制（实际只有这两个被编译）
│   ├── TVPWindow.h/cpp    ← 窗口封装（已注释掉）
│   ├── MenuItemEx.h/cpp   ← 菜单扩展（已注释掉）
│   └── ... (其他 UI 相关文件，大部分已注释)
│
├── linux/                 ← Linux 平台实现
│   └── Platform.cpp       ← Platform.h 的 Linux/POSIX 实现
│                            使用 /proc/meminfo、GTK、std::filesystem
│
├── android/               ← Android 平台实现
│   └── AndroidUtils.cpp   ← 1002 行，JNI 桥接 + Platform.h 实现
│                            Android 还复用 linux/Platform.cpp 的部分函数
│
├── apple/                 ← Apple 平台
│   └── macos/             ← macOS 特定（目前为空或极少代码）
│
├── cocos2d/               ← Cocos2d-x 桥接层（第 4 章详解）
│   ├── AppDelegate.h/cpp  ← 应用代理（初始化 Director、GLView）
│   ├── MainScene.h/cpp    ← 主场景（驱动引擎主循环）
│   ├── CustomFileUtils.h/cpp/mm  ← 自定义文件系统
│   ├── CCKeyCodeConv.h/cpp      ← 键码转换映射
│   └── YUVSprite.h/cpp    ← YUV 视频精灵（自定义 Shader）
│
├── ConfigManager/         ← 配置管理系统（第 5 章详解）
│   ├── GlobalConfigManager.h/cpp     ← 全局配置
│   ├── IndividualConfigManager.h/cpp ← 游戏独立配置
│   └── LocaleConfigManager.h/cpp     ← 区域化配置
│
├── sdl/                   ← SDL 集成
│   └── tvpsdl.cpp         ← 50 行，SDL 启动桩代码
│
└── ui/                    ← 用户界面组件（33 个文件）
    ├── FileSelectorForm.h/cpp  ← 文件选择对话框
    ├── ConsoleForm.h/cpp       ← 调试控制台
    └── ... (其他 UI 组件)
```

### 2.1 文件数量与实际编译的差异

虽然 win32/ 下有 31 个文件，但 CMakeLists.txt 只编译其中 2 个：

```cmake
# CMakeLists.txt 中 win32 的实际编译列表（简化）
$<$<BOOL:${WIN32}>:
    win32/Platform.cpp
    win32/SystemControl.cpp
>
```

其余 29 个文件（TVPWindow、MenuItemEx 等）在源码中用注释标记为"已弃用"或"待移植"。这些是从原始 KiriKiri 引擎遗留下来的 Win32 专用 UI 代码，在跨平台重构中被 Cocos2d-x 的 UI 系统取代了。

### 2.2 文件规模统计

| 文件 | 行数 | 说明 |
|------|------|------|
| Application.cpp | 937 | 最大的核心文件，应用生命周期 |
| android/AndroidUtils.cpp | 1002 | Android JNI 桥接，最复杂的平台实现 |
| win32/Platform.cpp | 408 | Windows 平台实现 |
| linux/Platform.cpp | 335 | Linux/POSIX 平台实现 |
| Application.h | 211 | 应用类声明 |
| DetectCPU.cpp | 176 | CPU 检测（大部分 stub） |
| cocos2d/AppDelegate.cpp | 120 | Cocos2d-x 入口 |
| CMakeLists.txt | 88 | 构建配置 |
| Platform.h | 73 | 平台接口声明 |
| ConfigManager/GlobalConfigManager.h | 69 | 配置基类 |
| sdl/tvpsdl.cpp | 50 | SDL 桩 |

总计约 **3500+ 行** C++ 代码（不含被注释掉的文件）。

---

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

## 7. 平台抽象的设计哲学

environ 模块采用了**C 风格函数 + 条件编译**的抽象方式，而非 C++ 虚函数多态：

```cpp
// Platform.h 中的声明方式
void TVPGetMemoryInfo(TVPMemoryInfo& info);  // C 风格自由函数
void TVPShowSimpleMessageBox(const ttstr& text, const ttstr& caption);
bool TVPGetDriverPath(const std::string& driver, std::string& path);

// 对比：不使用的虚函数方式
class IPlatform {
    virtual void GetMemoryInfo(TVPMemoryInfo& info) = 0;
    virtual void ShowMessageBox(...) = 0;
};
```

**为什么选择 C 风格？**

1. **零虚函数开销**：没有 vtable 查找，直接函数调用
2. **链接时分发**：编译器/链接器在构建时就确定了用哪个平台的实现，运行时没有任何间接跳转
3. **简单直接**：不需要工厂模式、不需要注册机制、不需要运行时类型查询
4. **与 C 代码互操作**：KiriKiri 引擎有大量 C 风格的遗留代码，C 风格函数与之无缝对接

**代价**：每个平台的实现文件不能同时编译（不能在一个二进制中同时包含 win32/Platform.cpp 和 linux/Platform.cpp，因为它们定义了相同的函数）。这就是为什么 CMakeLists.txt 中用生成器表达式互斥选择。

---

## 动手实践

### 实践 1：追踪平台编译链

在你的 KrKr2 构建目录中，用以下命令查看 environ 库实际编译了哪些文件：

```bash
# Windows (MSVC)
cmake --build build --target environ --verbose 2>&1 | grep "\.cpp"

# Linux (GCC/Clang + Ninja)
ninja -C build -v environ 2>&1 | grep "\.cpp"

# 或者直接查看 CMake 生成的 build.ninja / .vcxproj 文件
# 搜索 "environ" 相关的编译规则
```

预期看到：
- Windows: `Application.cpp`, `DetectCPU.cpp`, `win32/Platform.cpp`, `win32/SystemControl.cpp`, `cocos2d/AppDelegate.cpp`, `cocos2d/MainScene.cpp`, ...
- Linux: `Application.cpp`, `DetectCPU.cpp`, `linux/Platform.cpp`, `sdl/tvpsdl.cpp`, ...

### 实践 2：添加一个新的平台接口函数

假设你需要添加一个获取屏幕 DPI 的函数。在 environ 的架构下，正确的做法是：

```cpp
// 步骤 1: 在 Platform.h 中声明接口
float TVPGetScreenDPI();

// 步骤 2: 在 win32/Platform.cpp 中实现
float TVPGetScreenDPI() {
    HDC hdc = GetDC(nullptr);
    float dpi = static_cast<float>(GetDeviceCaps(hdc, LOGPIXELSX));
    ReleaseDC(nullptr, hdc);
    return dpi;
}

// 步骤 3: 在 linux/Platform.cpp 中实现
float TVPGetScreenDPI() {
    // 从 X11 或 GTK 获取 DPI
    GdkScreen* screen = gdk_screen_get_default();
    return static_cast<float>(gdk_screen_get_resolution(screen));
}

// 步骤 4: 在 android/AndroidUtils.cpp 中实现
float TVPGetScreenDPI() {
    // 通过 JNI 调用 DisplayMetrics.densityDpi
    JNIEnv* env = GetJNIEnv();
    // ... JNI 调用 ...
    return dpi;
}
```

**注意**：不需要修改 CMakeLists.txt，因为这些文件已经在编译列表中了。新函数会自动被编译进对应平台的 environ 库中。

### 实践 3：验证依赖关系

用 CMake 的 `--graphviz` 选项生成依赖图：

```bash
cmake -B build --graphviz=deps.dot
# 在 deps.dot 中搜索 "environ" 节点，查看其入边（被谁依赖）和出边（依赖谁）
```

---

## 对照项目源码

本章涉及的核心源文件及其阅读建议：

| 文件 | 行数 | 阅读重点 |
|------|------|----------|
| `CMakeLists.txt` | 88 | 生成器表达式、链接关系 |
| `Platform.h` | 73 | 接口设计哲学 |
| `Application.h` | 211 | 类声明、消息类型枚举 |
| `DetectCPU.cpp` | 176 | CPU 检测模式（虽然是 stub） |

**建议的源码阅读顺序**：

1. `CMakeLists.txt` → 理解"编译了什么"
2. `Platform.h` → 理解"接口是什么"
3. `Application.h` → 理解"核心类是什么"
4. 然后进入第 2 章（Platform 实现）和第 3 章（Application 实现）

---

## 本节小结

| 要点 | 说明 |
|------|------|
| environ 的定位 | 引擎与 OS 之间的"翻译层"，集中处理所有平台差异 |
| 构建方式 | STATIC 库，用 CMake 生成器表达式按平台选择源文件 |
| 链接关系 | PUBLIC 链接 tjs2（头文件暴露），PRIVATE 链接其他 core 模块 |
| 平台抽象方式 | C 风格自由函数 + 链接时分发，非虚函数多态 |
| 两套宿主框架 | Cocos2d-x（Windows/macOS/Android）vs SDL2（桌面 Linux） |
| Android 特殊性 | 复用 linux/Platform.cpp + 额外的 AndroidUtils.cpp (JNI) |
| win32/ 遗留文件 | 31 个文件中只有 2 个被编译，其余为已弃用的 Win32 原生 UI 代码 |
| CPU 检测 | DetectCPU.cpp 大部分 stub，仅 Apple ARM NEON 有实际逻辑 |

---

## 练习题与答案

### 练习 1：生成器表达式分析

**问题**：以下 CMake 生成器表达式在 Android 平台上展开后的结果是什么？

```cmake
$<$<AND:$<BOOL:${LINUX}>,$<NOT:$<BOOL:${ANDROID}>>>:
    linux/Platform.cpp
>
```

已知在 Android NDK 构建时：`LINUX=TRUE`, `ANDROID=TRUE`。

<details>
<summary>查看答案</summary>

展开为**空**（不编译 `linux/Platform.cpp`）。

分析过程：
1. `$<BOOL:${LINUX}>` → `$<BOOL:TRUE>` → `1`
2. `$<BOOL:${ANDROID}>` → `$<BOOL:TRUE>` → `1`
3. `$<NOT:1>` → `0`
4. `$<AND:1,0>` → `0`
5. `$<$<AND:...>:linux/Platform.cpp>` → `$<0:linux/Platform.cpp>` → **空**

这就是为什么 Android 的 `linux/Platform.cpp` 是在另一个独立的生成器表达式块 `$<$<BOOL:${ANDROID}>:linux/Platform.cpp>` 中引入的，而非通过这个"桌面 Linux"块。

</details>

### 练习 2：依赖传播

**问题**：假设模块 A 链接了 environ（`target_link_libraries(A PRIVATE environ)`），那么模块 A 能否直接 `#include "tjs2/tjsTypes.h"`？为什么？

<details>
<summary>查看答案</summary>

**能**。虽然 A 对 environ 的链接是 PRIVATE，但 environ 对 tjs2 的链接是 **PUBLIC**。

CMake 的传递性规则：
- A → PRIVATE → environ：A 获得 environ 的 PUBLIC 和 INTERFACE 属性
- environ → PUBLIC → tjs2：environ 的使用者（A）也获得 tjs2 的头文件搜索路径

所以 A 可以直接包含 tjs2 的头文件。这就是 environ 将 tjs2 设为 PUBLIC 的原因——Application.h 中使用了 tjs2 的类型，任何包含 Application.h 的代码都需要 tjs2 的头文件。

</details>

### 练习 3：添加新平台

**问题**：如果你要为 KrKr2 添加 iOS 平台支持，需要在 environ 模块中做哪些修改？请列出至少 4 个具体步骤。

<details>
<summary>查看答案</summary>

需要的修改至少包括：

1. **创建 `ios/` 子目录** 并实现 `Platform.cpp`（或 `Platform.mm`），提供 Platform.h 中所有函数的 iOS 实现：
   - `TVPGetMemoryInfo` → 使用 `task_info` 或 `host_statistics`
   - `TVPShowSimpleMessageBox` → 使用 `UIAlertController`
   - `TVPGetAppStoragePath` → 使用 `NSSearchPathForDirectoriesInDomains`
   - `TVP_stat` → 使用标准 POSIX `stat`

2. **修改 CMakeLists.txt**，添加 iOS 条件编译块：
   ```cmake
   $<$<BOOL:${IOS}>:
       ios/Platform.mm
   >
   ```

3. **更新 cocos2d/CustomFileUtils.mm**（或创建 iOS 专用版本），处理 iOS 沙盒文件系统的特殊路径规则（如 Documents/ 和 Bundle/）

4. **在 cocos2d/AppDelegate.cpp 中添加 iOS 分支**（如果需要），处理 iOS 特有的生命周期事件（如 `applicationDidEnterBackground` 需要保存状态、`applicationWillTerminate` 需要清理资源）

5. **可选：更新 DetectCPU.cpp**，iOS 设备使用 ARM 处理器（A 系列芯片），NEON 始终可用，可复用 Apple ARM 的检测逻辑

</details>

---

## 下一步

下一章 **02-Platform 接口与平台实现对比** 将深入 `Platform.h` 中声明的每个接口函数，并逐行对比 Windows、Linux、Android 三大平台的具体实现。你将看到同一个 `TVPGetMemoryInfo()` 在三个平台上是如何用完全不同的系统 API 实现相同功能的。
