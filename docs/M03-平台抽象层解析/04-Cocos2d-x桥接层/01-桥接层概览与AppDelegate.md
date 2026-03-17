# 第四节：Cocos2d-x 桥接层深度解析

> **所属模块：** M03-平台抽象层解析
> **前置知识：** [第三节：Application 生命周期与消息系统](./03-Application生命周期与消息系统.md)、[P04-OpenGL 图形编程](../P04-OpenGL图形编程/README.md)
> **预计阅读时间：** 45 分钟（约 9000 字）

## 本节目标

读完本节后，你将能够：
1. 理解 `TVPAppDelegate` 如何桥接 Cocos2d-x 框架与 KrKr2 引擎核心
2. 掌握 `TVPMainScene` 的 GameNode/UINode 双层场景架构设计
3. 分析 `TVPWindowLayer` 如何将移动端触摸事件转换为桌面端鼠标事件
4. 理解 UI 表单栈（pushUIForm/popUIForm）的动画机制与 `iTVPBaseForm` 继承体系
5. 掌握虚拟鼠标光标系统的实现原理与手柄输入映射

## 概览：桥接层在架构中的位置

KrKr2 作为一个将桌面视觉小说引擎移植到移动端的模拟器，面临一个核心挑战：**原版 KiriKiri 引擎假定运行在 Windows 桌面环境**（有鼠标、键盘、Win32 窗口），而 Cocos2d-x 运行在**触摸屏移动设备**上。桥接层的职责就是消除这个差异。

```
┌─────────────────────────────────────────────────────┐
│                  平台入口 (main.cpp)                  │
│          Windows / Linux / macOS / Android           │
└────────────────────┬────────────────────────────────┘
                     │ 构造 TVPAppDelegate, 调用 run()
                     ▼
┌─────────────────────────────────────────────────────┐
│              TVPAppDelegate (桥接层入口)               │
│    继承 cocos2d::Application                         │
│    applicationDidFinishLaunching() = 引擎启动点       │
└────────────────────┬────────────────────────────────┘
                     │ 创建 Director, GLView, MainScene
                     ▼
┌─────────────────────────────────────────────────────┐
│              TVPMainScene (主场景)                     │
│    ┌──────────┐    ┌──────────┐                      │
│    │ GameNode │    │ UINode   │                      │
│    │ (z=0)    │    │ (z=4)    │                      │
│    │ 游戏渲染  │    │ UI表单栈  │                      │
│    └────┬─────┘    └────┬─────┘                      │
│         │               │                            │
│    TVPWindowLayer   iTVPBaseForm                     │
│    (触摸→鼠标转换)   (导航栏+内容+底栏)                │
└─────────────────────────────────────────────────────┘
```

涉及的核心源文件：

| 文件 | 行数 | 职责 |
|------|------|------|
| `cpp/core/environ/cocos2d/AppDelegate.h` | 26 | TVPAppDelegate 类声明 |
| `cpp/core/environ/cocos2d/AppDelegate.cpp` | 120 | 应用启动、GL 上下文、前后台切换 |
| `cpp/core/environ/cocos2d/MainScene.h` | 94 | TVPMainScene 类声明 |
| `cpp/core/environ/cocos2d/MainScene.cpp` | 2728 | 主场景、窗口层、输入系统、虚拟光标 |
| `cpp/core/environ/ui/BaseForm.h` | 216 | UI 表单基类、CSB 读取器、触摸路由 |

---

## 一、TVPAppDelegate：Cocos2d-x 应用入口

### 1.1 类结构

`TVPAppDelegate` 继承自 `cocos2d::Application`，是整个应用的生命周期管理者。在所有平台的 `main()` 函数中，都会构造这个对象并调用 `run()`：

```cpp
// 文件：cpp/core/environ/cocos2d/AppDelegate.h（完整）
#pragma once
#include <cocos2d.h>

class TVPAppDelegate : public cocos2d::Application {
    // OpenGL 上下文属性初始化
    void initGLContextAttrs() override;

    // 应用启动完成 — 引擎初始化的核心入口
    bool applicationDidFinishLaunching() override;

    // 切换到后台（Home 键、切换应用等）
    void applicationDidEnterBackground() override;

    // 从后台恢复到前台
    void applicationWillEnterForeground() override;
};
```

Cocos2d-x 的 `Application` 基类提供了跨平台的应用生命周期抽象。`TVPAppDelegate` 只需重写 4 个虚函数，框架会在合适的时机调用它们。

### 1.2 initGLContextAttrs — OpenGL 上下文配置

```cpp
// 文件：cpp/core/environ/cocos2d/AppDelegate.cpp 第 111-114 行
void TVPAppDelegate::initGLContextAttrs() {
    // 依次为：红、绿、蓝、透明、深度、模板 的位数
    GLContextAttrs glContextAttrs = { 8, 8, 8, 8, 24, 8 };
    cocos2d::GLView::setGLContextAttrs(glContextAttrs);
}
```

各参数含义：

| 参数 | 值 | 说明 |
|------|----|------|
| Red bits | 8 | 红色通道 8 位（0-255） |
| Green bits | 8 | 绿色通道 8 位 |
| Blue bits | 8 | 蓝色通道 8 位 |
| Alpha bits | 8 | 透明通道 8 位（支持半透明渲染） |
| Depth bits | 24 | 深度缓冲 24 位（足够 3D 场景精度） |
| Stencil bits | 8 | 模板缓冲 8 位（用于裁剪、遮罩效果） |

这是标准的 RGBA8888 + D24S8 配置，兼容所有主流 GPU。KrKr2 虽然主要做 2D 渲染，但 24 位深度缓冲确保了 z-order 排序的精度，8 位模板缓冲用于 Cocos2d-x 的 ClippingNode 等裁剪功能。

> **跨平台差异：** 此函数在所有平台上行为一致。Android 上 EGL 会自动选择最接近的配置；桌面平台上 GLFW/SDL 直接请求精确匹配。如果设备不支持 D24S8，Cocos2d-x 会自动降级。

### 1.3 applicationDidFinishLaunching — 启动核心流程

这是整个 KrKr2 引擎最重要的初始化函数，逐行解析如下：

```cpp
// 文件：cpp/core/environ/cocos2d/AppDelegate.cpp 第 31-108 行
bool TVPAppDelegate::applicationDidFinishLaunching() {
    // 第一步：SDL 初始化标记
    SDL_SetMainReady();   // 告诉 SDL "主线程已就绪"，跳过 SDL 自己的 main 劫持
    TVPMainThreadID = std::this_thread::get_id();  // 记录主线程 ID，后续用于线程安全检查

    // 第二步：获取 Cocos2d-x Director 单例并创建 GLView
    auto director = cocos2d::Director::getInstance();
    auto glview = director->getOpenGLView();
    if(!glview) {
        glview = cocos2d::GLViewImpl::create("krkr2");  // 创建名为 "krkr2" 的 OpenGL 窗口
        director->setOpenGLView(glview);

        // Win32 平台特殊处理：添加可调节边框和最大化按钮
#if CC_TARGET_PLATFORM == CC_PLATFORM_WIN32
        HWND hwnd = glview->getWin32Window();
        if(hwnd) {
            LONG style = GetWindowLong(hwnd, GWL_STYLE);
            style |= WS_THICKFRAME | WS_MAXIMIZEBOX;  // 允许拖拽调整窗口大小 + 最大化
            SetWindowLong(hwnd, GWL_STYLE, style);
    }
    return true;
}
```

**关键点解析：**
- `SDL_SetMainReady()` 是一个 C 函数（通过 `extern "C"` 声明），因为 KrKr2 使用 SDL 作为底层窗口/输入抽象，但入口由 Cocos2d-x 管理，所以需要手动告知 SDL "不用劫持 main"
- Win32 上默认的 Cocos2d-x 窗口是固定大小的，KrKr2 通过 Win32 API 添加 `WS_THICKFRAME`（可拖拽边框）和 `WS_MAXIMIZEBOX`（最大化按钮），让用户可以自由调整窗口大小

### 1.4 分辨率策略 — 移动端 vs 桌面端

```cpp
    // 第三步：设计分辨率配置
    static cocos2d::Size designSize(960, 640);  // 基础设计分辨率

    // 移动平台：使用 EXACT_FIT 策略
#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID || \
     CC_TARGET_PLATFORM == CC_PLATFORM_IOS)
    cocos2d::Size screenSize = glview->getFrameSize();  // 获取物理屏幕尺寸
    if(screenSize.width < screenSize.height) {
        std::swap(screenSize.width, screenSize.height); // 确保横屏模式
    }
    cocos2d::Size ds = designSize;
    ds.height = ds.width * screenSize.height / screenSize.width;  // 按屏幕比例调整高度
    glview->setDesignResolutionSize(
        screenSize.width, screenSize.height,
        ResolutionPolicy::EXACT_FIT);  // 精确填充，无黑边
#else
    // 桌面平台：使用 FIXED_WIDTH 策略
    glview->setDesignResolutionSize(
        designSize.width, designSize.height,
        ResolutionPolicy::FIXED_WIDTH);  // 固定宽度，高度按比例
#endif
```

两种分辨率策略的对比：

| 策略 | 平台 | 行为 | 适用场景 |
|------|------|------|----------|
| `EXACT_FIT` | Android/iOS | 拉伸填满整个屏幕，无黑边 | 移动端全屏游戏 |
| `FIXED_WIDTH` | Windows/Linux/macOS | 固定宽度 960px，高度按窗口比例自适应 | 桌面窗口化运行 |

> **为什么移动端用 EXACT_FIT？** 视觉小说游戏的画面由脚本精确控制布局，轻微的拉伸变形不明显。而且移动端用户期望全屏无黑边的体验。桌面端用 FIXED_WIDTH 是因为窗口比例可能被用户随时调整。

### 1.5 启动后续流程：场景创建与延迟加载

```cpp
    // 第四步：资源搜索路径、FPS、UI 扩展初始化
    std::vector<std::string> searchPath;
    searchPath.emplace_back("res");
    cocos2d::FileUtils::getInstance()->setSearchPaths(searchPath);
    director->setDisplayStats(false);           // 关闭内置 FPS 显示
    director->setAnimationInterval(1.0f / 60);  // 60 FPS

    TVPInitUIExtension();  // 初始化 UI 扩展组件
    LocaleConfigManager::GetInstance()->Initialize(TVPGetCurrentLanguage());

    // 第五步：创建主场景并运行
    TVPMainScene *scene = TVPMainScene::CreateInstance();  // 单例创建
    director->runWithScene(scene);  // 启动 Cocos2d-x 主循环

    // 第六步：延迟一帧后初始化全局配置和文件选择器
    scene->scheduleOnce([](float dt) {
        TVPMainScene::GetInstance()->unschedule("launch");
        TVPGlobalPreferenceForm::Initialize();  // 初始化全局偏好设置
        if(!TVPCheckStartupArg()) {
            // 如果没有命令行指定游戏路径，显示文件选择器
            TVPMainScene::GetInstance()->pushUIForm(
                TVPMainFileSelectorForm::create());
        }
    }, 0, "launch");

    return true;  // 返回 true 表示初始化成功
}
```

**为什么要用 `scheduleOnce` 延迟？** 因为 Cocos2d-x 的 `runWithScene` 之后需要至少一帧来完成场景的初始化（节点树构建、OpenGL 资源加载等）。如果立即创建 UI 表单，可能引用到尚未就绪的节点。延迟一帧确保场景完全初始化后再进行 UI 操作。

### 1.6 前后台切换

```cpp
// 文件：cpp/core/environ/cocos2d/AppDelegate.cpp 第 21-29 行
void TVPAppDelegate::applicationWillEnterForeground() {
    ::Application->OnActivate();                       // 通知引擎核心恢复运行
    cocos2d::Director::getInstance()->startAnimation(); // 恢复 Cocos2d-x 渲染循环
}

void TVPAppDelegate::applicationDidEnterBackground() {
    ::Application->OnDeactivate();                     // 通知引擎核心暂停
    cocos2d::Director::getInstance()->stopAnimation();  // 暂停渲染，节省电量
}
```

三方协作关系：

```
用户按 Home 键
    │
    ▼
iOS/Android 系统 ──→ Cocos2d-x Application ──→ TVPAppDelegate
                                                    │
                    ┌───────────────────────────────┘
                    │
                    ▼
    ┌──────────────────────────────────┐
    │ ::Application->OnDeactivate()   │  ← 暂停 TJS2 脚本、音频、视频
    │ Director->stopAnimation()       │  ← 停止 OpenGL 渲染循环
    └──────────────────────────────────┘
```

`::Application` 是 KrKr2 引擎核心的 `tTVPApplication` 全局指针（注意前面的 `::` 表示全局命名空间，区分于 `cocos2d::Application`）。第三节已详细分析过它的 `OnActivate()`/`OnDeactivate()` 机制。

---

