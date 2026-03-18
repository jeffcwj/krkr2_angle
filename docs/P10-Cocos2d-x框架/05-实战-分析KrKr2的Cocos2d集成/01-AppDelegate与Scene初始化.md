# AppDelegate 与 Scene 初始化

> **所属模块：** P10-Cocos2d-x 框架
> **前置知识：** [Director 与主循环](../01-Cocos2d-x架构/01-Director与主循环.md)、[Scene-Layer-Node 树](../01-Cocos2d-x架构/02-Scene-Layer-Node树.md)、[内存管理与引用计数](../01-Cocos2d-x架构/03-内存管理与引用计数.md)
> **预计阅读时间：** 30 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 KrKr2 从平台入口到 Cocos2d-x 引擎启动的完整流程
2. 解读 `TVPAppDelegate` 的每一行代码及其设计意图
3. 分析 `TVPMainScene::initialize()` 如何构建双节点树架构
4. 掌握设计分辨率（Design Resolution）在不同平台的适配策略
5. 理解 scheduled launch（延迟启动）模式及其解决的竞态问题

## 平台入口：从 main 到 AppDelegate

### 各平台入口文件总览

KrKr2 支持 Windows、Linux、macOS 和 Android 四个平台，每个平台有自己的入口文件，但它们最终都汇聚到同一个启动函数——`TVPAppDelegate::run()`。

```
平台入口调用流程：

┌──────────────────────────────────────────────────────┐
│  Windows: platforms/windows/main.cpp                 │
│  _tWinMain() ──────────────────────┐                 │
├──────────────────────────────────────┤                │
│  Linux:   platforms/linux/main.cpp   │                │
│  main() ───────────────────────────┤                 │
├──────────────────────────────────────┤  ┌───────────┐│
│  macOS:   platforms/apple/macos/     │  │ 公共流程  ││
│  main.cpp  main() ─────────────────┤──▶│           ││
├──────────────────────────────────────┤  │ 1. spdlog ││
│  Android: platforms/android/cpp/     │  │ 2. new    ││
│  krkr2_android.cpp                   │  │    Delegate│
│  cocos_android_app_init() ─────────┘  │ 3. run()  ││
│                                        └───────────┘│
└──────────────────────────────────────────────────────┘
```

### Windows 入口详解

```cpp
// 文件：platforms/windows/main.cpp（完整 44 行）
#include "main.h"
#include <cocos2d.h>
#include <spdlog/spdlog.h>
#include <spdlog/sinks/stdout_color_sinks.h>
#include <shellapi.h>          // CommandLineToArgvW
#include <boost/locale.hpp>    // UTF 转换

#include "tjsString.h"
#include "environ/cocos2d/AppDelegate.h"
#include "environ/ui/MainFileSelectorForm.h"
USING_NS_CC;

int WINAPI _tWinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance,
                     LPTSTR lpCmdLine, int nCmdShow) {
    UNREFERENCED_PARAMETER(hPrevInstance);
    UNREFERENCED_PARAMETER(lpCmdLine);

    // 第一步：处理命令行参数（支持拖拽 .xp3 文件到 exe 图标）
    int argc = 0;
    LPWSTR *argv = CommandLineToArgvW(GetCommandLineW(), &argc);
    if(argc > 1) {
        // 将 wstring 转为 UTF-8，存入全局文件选择器路径
        std::wstring xp3Path = argv[1];
        std::string xp3PathUtf8 =
            boost::locale::conv::utf_to_utf<char>(xp3Path);
        spdlog::info("XP3 文件路径: {}", xp3PathUtf8);
        TVPMainFileSelectorForm::filePath = xp3PathUtf8;
    }
    LocalFree(argv);  // 释放 CommandLineToArgvW 分配的内存

    // 第二步：初始化日志系统
    spdlog::set_level(spdlog::level::debug);
    static auto core_logger = spdlog::stdout_color_mt("core");
    static auto tjs2_logger = spdlog::stdout_color_mt("tjs2");
    static auto plugin_logger = spdlog::stdout_color_mt("plugin");
    spdlog::set_default_logger(core_logger);

    // 第三步：创建 AppDelegate 并启动 Cocos2d-x 主循环
    static auto pAppDelegate = std::make_unique<TVPAppDelegate>();
    return pAppDelegate->run();  // 永不返回，直到窗口关闭
}
```

关键设计点：

| 要素 | 说明 |
|------|------|
| `CommandLineToArgvW` | Win32 API，解析宽字符命令行，支持拖放文件到 exe 启动 |
| `boost::locale::conv::utf_to_utf` | Windows 内部用 UTF-16（wstring），Cocos2d/KrKr2 用 UTF-8（string），需要转换 |
| `spdlog::stdout_color_mt` | 创建线程安全（mt）的彩色控制台日志器，分三个 logger 方便过滤 |
| `std::make_unique<TVPAppDelegate>` | 用智能指针管理 AppDelegate 生命周期，避免内存泄漏 |
| `pAppDelegate->run()` | 进入 Cocos2d-x 主循环，内部调用 `applicationDidFinishLaunching()` 后启动帧循环 |

### Linux 入口详解

```cpp
// 文件：platforms/linux/main.cpp（完整 26 行）
#include <memory>
#include <gtk/gtk.h>           // GTK 初始化（用于文件对话框）

#include <spdlog/spdlog.h>
#include <spdlog/sinks/stdout_color_sinks.h>
#include "environ/cocos2d/AppDelegate.h"
#include "environ/ui/MainFileSelectorForm.h"

int main(int argc, char **argv) {
    gtk_init(&argc, &argv);    // 初始化 GTK（即使不用 GTK 窗口，文件对话框需要）
    spdlog::set_level(spdlog::level::debug);

    static auto core_logger = spdlog::stdout_color_mt("core");
    static auto tjs2_logger = spdlog::stdout_color_mt("tjs2");
    static auto plugin_logger = spdlog::stdout_color_mt("plugin");

    if(argc > 1) {
        // Linux 的 argv 已经是 UTF-8，无需转换
        TVPMainFileSelectorForm::filePath = argv[1];
    }
    spdlog::set_default_logger(core_logger);

    static auto pAppDelegate = std::make_unique<TVPAppDelegate>();
    return pAppDelegate->run();
}
```

对比 Windows 入口，Linux 版本的差异：

| 差异点 | Windows | Linux |
|--------|---------|-------|
| 入口函数 | `_tWinMain`（Win32 GUI 应用） | 标准 `main` |
| 字符编码 | UTF-16 → UTF-8 需要 `boost::locale` 转换 | 原生 UTF-8，直接赋值 |
| 额外初始化 | 无 | `gtk_init()`（文件选择对话框依赖） |
| 命令行解析 | `CommandLineToArgvW` + `LocalFree` | 直接使用 `argv` |

### Android 入口特殊性

Android 的入口不是 `main()`，而是 JNI 回调：

```cpp
// 文件：platforms/android/cpp/krkr2_android.cpp（节选）
void cocos_android_app_init(JNIEnv *env) {
    // 由 Cocos2d-x Java 层在 Activity 创建时调用
    static auto pAppDelegate = std::make_unique<TVPAppDelegate>();
    // Android 不需要手动 run()，Cocos2d-x Java 层会驱动主循环
}
```

Android 的生命周期由 Java/Activity 驱动，C++ 层通过 JNI 桥接，AppDelegate 的 `run()` 由 Cocos2d-x 的 Java 基础设施自动调用。

### 四平台启动流程对比图

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│   Windows   │   │    Linux    │   │    macOS    │   │   Android   │
│ _tWinMain() │   │   main()   │   │   main()   │   │ cocos_android│
│             │   │             │   │             │   │ _app_init() │
└──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
       │                 │                 │                 │
       ▼                 ▼                 ▼                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  1. 初始化 spdlog（core/tjs2/plugin 三个 logger）              │
  │  2. 处理命令行参数（文件路径 → TVPMainFileSelectorForm）        │
  │  3. std::make_unique<TVPAppDelegate>()                         │
  │  4. pAppDelegate->run()                                        │
  └─────────────────────────────┬───────────────────────────────────┘
                                │
                                ▼
                   ┌────────────────────────┐
                   │ initGLContextAttrs()   │
                   │ applicationDidFinish   │
                   │   Launching()          │
                   │ 进入帧循环（永久）       │
                   └────────────────────────┘
```

## TVPAppDelegate 逐行解析

### 类定义

```cpp
// 文件：cpp/core/environ/cocos2d/AppDelegate.h（完整 26 行）
#pragma once
#include <cocos2d.h>

class TVPAppDelegate : public cocos2d::Application {
    // GL 上下文属性设置
    void initGLContextAttrs() override;

    // 引擎启动后的初始化入口
    bool applicationDidFinishLaunching() override;

    // 生命周期回调
    void applicationDidEnterBackground() override;
    void applicationWillEnterForeground() override;
};
```

`TVPAppDelegate` 继承自 `cocos2d::Application`，这是 Cocos2d-x 的标准模式。`Application` 基类管理平台主循环，子类通过重写虚函数来注入自定义逻辑。

### GL 上下文配置

```cpp
// 文件：cpp/core/environ/cocos2d/AppDelegate.cpp 第 111-114 行
void TVPAppDelegate::initGLContextAttrs() {
    // 参数依次为：Red, Green, Blue, Alpha, Depth, Stencil 的位深
    GLContextAttrs glContextAttrs = { 8, 8, 8, 8, 24, 8 };
    cocos2d::GLView::setGLContextAttrs(glContextAttrs);
}
```

这些值的含义：

| 参数 | 值 | 说明 |
|------|----|------|
| Red bits | 8 | 红色通道 8 位（0-255） |
| Green bits | 8 | 绿色通道 8 位 |
| Blue bits | 8 | 蓝色通道 8 位 |
| Alpha bits | 8 | 透明度通道 8 位 |
| Depth bits | 24 | 深度缓冲 24 位（足够 3D 精度，2D 游戏也需要用于 zOrder 排序） |
| Stencil bits | 8 | 模板缓冲 8 位（用于 ClippingNode 裁剪等效果） |

这是 `run()` 调用链中**最先执行**的函数，在创建 GL 窗口之前设置。如果这些值设置不当（如 Depth=0），会导致节点层级渲染混乱。

### applicationDidFinishLaunching 核心流程

这是整个 KrKr2 启动的核心函数，逐段分析：

#### 阶段一：SDL 与线程初始化

```cpp
// 文件：cpp/core/environ/cocos2d/AppDelegate.cpp 第 31-34 行
bool TVPAppDelegate::applicationDidFinishLaunching() {
    SDL_SetMainReady();                          // 告知 SDL "我已在主线程"
    TVPMainThreadID = std::this_thread::get_id();// 记录主线程 ID，用于后续断言
    spdlog::debug("App Finish Launching");
```

`SDL_SetMainReady()` 是一个关键调用——KrKr2 使用 SDL2 作为底层输入/窗口抽象，但 Cocos2d-x 也在管理窗口。这一行告诉 SDL "不需要你来初始化主循环，我自己管理"，避免 SDL 和 Cocos2d-x 抢夺事件循环控制权。

#### 阶段二：Director 与 GLView 创建

```cpp
    // 第 36-50 行
    auto director = cocos2d::Director::getInstance();
    auto glview = director->getOpenGLView();
    if(!glview) {
        // 创建 OpenGL 窗口，标题为 "krkr2"
        glview = cocos2d::GLViewImpl::create("krkr2");
        director->setOpenGLView(glview);

#if CC_TARGET_PLATFORM == CC_PLATFORM_WIN32
        // Windows 平台特殊处理：添加可调节边框和最大化按钮
        HWND hwnd = glview->getWin32Window();
        if(hwnd) {
            LONG style = GetWindowLong(hwnd, GWL_STYLE);
            style |= WS_THICKFRAME | WS_MAXIMIZEBOX;
            SetWindowLong(hwnd, GWL_STYLE, style);
        }
#endif
    }
```

Windows 特殊处理的原因：Cocos2d-x 默认创建的窗口是**固定大小不可调整**的（游戏通常如此），但 KrKr2 作为模拟器需要用户能自由调整窗口大小，所以手动添加了 `WS_THICKFRAME`（可拖拽边框）和 `WS_MAXIMIZEBOX`（最大化按钮）。

#### 阶段三：设计分辨率适配

```cpp
    // 第 52-66 行
#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID || \
     CC_TARGET_PLATFORM == CC_PLATFORM_IOS)
    // 移动端：EXACT_FIT 精确拉伸
    cocos2d::Size screenSize = glview->getFrameSize();
    if(screenSize.width < screenSize.height) {
        std::swap(screenSize.width, screenSize.height); // 强制横屏
    }
    cocos2d::Size ds = designSize;  // designSize = (960, 640)
    ds.height = ds.width * screenSize.height / screenSize.width;
    glview->setDesignResolutionSize(
        screenSize.width, screenSize.height,
        ResolutionPolicy::EXACT_FIT);
#else
    // 桌面端：FIXED_WIDTH 固定宽度
    glview->setDesignResolutionSize(
        designSize.width, designSize.height,
        ResolutionPolicy::FIXED_WIDTH);
#endif
```

两种策略的区别：

```
EXACT_FIT（移动端）：
┌────────────────────────┐
│  设计分辨率 = 屏幕分辨率  │  拉伸填满屏幕
│  可能变形但无黑边         │  适合全屏移动设备
└────────────────────────┘

FIXED_WIDTH（桌面端）：
┌──────────────────┐
│  宽度固定 = 960    │  高度按窗口比例自适应
│  高度随窗口变化     │  不变形，可能上下有空白
│  ↕ 自适应          │  适合可调大小的桌面窗口
└──────────────────┘
```

**为什么移动端用 EXACT_FIT？** 移动设备屏幕比例固定且全屏运行，精确拉伸能利用所有像素。**为什么桌面端用 FIXED_WIDTH？** 桌面窗口可以任意调整，固定宽度能保证 UI 元素的水平布局一致。

#### 阶段四：资源路径与帧率配置

```cpp
    // 第 68-86 行
    std::vector<std::string> searchPath;
    searchPath.emplace_back("res");
    cocos2d::FileUtils::getInstance()->setSearchPaths(searchPath);

    director->setDisplayStats(false);         // 关闭 FPS 显示（调试时可打开）
    director->setAnimationInterval(1.0f / 60);// 目标帧率 60 FPS
```

`setSearchPaths({"res"})` 告诉 Cocos2d-x 的文件系统在 `res/` 目录下查找资源文件（纹理、CSB、字体等）。这对应 `ui/cocos-studio/` 编译输出的目标目录。

#### 阶段五：创建 MainScene 并运行

```cpp
    // 第 87-108 行
    TVPInitUIExtension();     // 初始化 UI 扩展组件

    LocaleConfigManager::GetInstance()->Initialize(TVPGetCurrentLanguage());
    // 创建主场景（自动释放对象）
    TVPMainScene *scene = TVPMainScene::CreateInstance();
    director->runWithScene(scene);  // 启动场景

    // 延迟一帧后执行启动逻辑
    scene->scheduleOnce(
        [](float dt) {
            TVPMainScene::GetInstance()->unschedule("launch");
            TVPGlobalPreferenceForm::Initialize();
            if(!TVPCheckStartupArg()) {
                // 没有命令行参数指定游戏时，弹出文件选择器
                TVPMainScene::GetInstance()->pushUIForm(
                    TVPMainFileSelectorForm::create());
            }
        },
        0, "launch");

    return true;
}
```

**为什么用 `scheduleOnce` 延迟启动？** 这是一个经典的竞态问题解决方案：

```
问题：
  runWithScene(scene) → Director 开始管理 scene
  但 scene 的 onEnter()、布局计算可能还没完成
  如果立即 pushUIForm()，UI 尺寸计算可能基于未就绪的布局

解决：
  scheduleOnce(callback, delay=0, "launch")
  ↓
  在下一帧（所有 onEnter/布局完成后）再执行 UI 操作
  ↓
  此时 scene 的 ContentSize、UINode 等已正确初始化
```

### 生命周期回调

```cpp
// 前台/后台切换处理
void TVPAppDelegate::applicationWillEnterForeground() {
    ::Application->OnActivate();                   // 通知 KrKr2 引擎恢复
    cocos2d::Director::getInstance()->startAnimation(); // 恢复渲染循环
}

void TVPAppDelegate::applicationDidEnterBackground() {
    ::Application->OnDeactivate();                 // 通知 KrKr2 引擎暂停
    cocos2d::Director::getInstance()->stopAnimation();  // 暂停渲染循环
}
```

这两个函数在**移动端尤为重要**——用户切到后台时必须暂停渲染和音频，否则会被系统 kill 掉或浪费电量。桌面端虽然也会触发（最小化窗口），但影响较小。

## TVPMainScene::initialize 深度分析

### 工厂方法与单例模式

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 1753-1825 行

// 单例存储
static TVPMainScene *_instance = nullptr;

TVPMainScene *TVPMainScene::GetInstance() { return _instance; }

TVPMainScene *TVPMainScene::CreateInstance() {
    _instance = create();  // create() 是自定义工厂方法
    return _instance;
}

TVPMainScene *TVPMainScene::create() {
    TVPMainScene *ret = new TVPMainScene;  // 不用 cocos2d 的 CREATE_FUNC
    ret->initialize();                      // 手动初始化
    ret->autorelease();                     // 交给引用计数管理

    // 计算触摸移动阈值（基于 DPI）
    _touchMoveThresholdSq = cocos2d::Device::getDPI() / 10.0f;
    _touchMoveThresholdSq *= _touchMoveThresholdSq; // 存储平方值避免 sqrt
    return ret;
}
```

**为什么不用标准的 `CREATE_FUNC` 宏？** 因为 TVPMainScene 的构造过程比标准 Scene 复杂得多——它需要自定义的 `initialize()` 而非 Cocos2d-x 默认的 `init()`，并且需要额外的 DPI 计算。

### 双节点树初始化

```cpp
void TVPMainScene::initialize() {
    // 获取屏幕和设计尺寸
    auto glview = cocos2d::Director::getInstance()->getOpenGLView();
    cocos2d::Size screenSize = glview->getFrameSize();
    cocos2d::Size designSize = glview->getDesignResolutionSize();
    ScreenRatio = screenSize.height / designSize.height;

    // 根据实际屏幕比例调整设计宽度
    designSize.width = designSize.height * screenSize.width / screenSize.height;
    initWithSize(designSize);

    // 黑色底层（防止透明区域穿帮）
    addChild(LayerColor::create(Color4B::BLACK,
                                designSize.width, designSize.height));

    // ★ 核心架构：双节点树
    GameNode = cocos2d::Node::create();
    GameNode->setContentSize(designSize);
    GameNode->setAnchorPoint(Vec2(0, 0));
    addChild(GameNode, GAME_SCENE_ORDER);   // zOrder = 0

    UINode = cocos2d::Node::create();
    UINode->setContentSize(designSize);
    UINode->setAnchorPoint(Vec2::ZERO);
    UINode->setPosition(Vec2::ZERO);
    UISize = designSize;
    addChild(UINode, UI_NODE_ORDER);        // zOrder = 100（枚举值见下文）
```

双节点树是 KrKr2 最核心的架构设计：

```
TVPMainScene (Scene)
│
├── LayerColor(BLACK)         [底层] — 防止透明穿帮
│
├── GameNode                  [zOrder = 0]  GAME_SCENE_ORDER
│   ├── TVPWindowLayer×N      [zOrder = 0]  游戏内容窗口
│   ├── TVPConsoleWindow      [zOrder = 10] GAME_CONSOLE_ORDER
│   ├── WindowManagerOverlay  [zOrder = 15] GAME_WINMGR_ORDER
│   └── TVPGameMainMenu       [zOrder = 20] GAME_MENU_ORDER
│
└── UINode                    [zOrder = 100] UI_NODE_ORDER
    ├── FileSelectorForm      UI 表单栈
    ├── PreferenceForm        （通过 pushUIForm/popUIForm 管理）
    └── ...
```

**为什么分两棵树？** 这解决了一个根本性的架构问题：

| 需求 | GameNode | UINode |
|------|----------|--------|
| 内容来源 | KiriKiri 游戏引擎渲染 | KrKr2 模拟器 UI |
| 更新频率 | 每帧由 `::Application->Run()` 驱动 | 用户交互时才更新 |
| 坐标系 | 游戏内坐标（可能旋转/缩放） | 屏幕固定坐标 |
| 生命周期 | 跟随游戏启动/退出 | 跟随模拟器 |

### 事件监听器注册

```cpp
    // 键盘事件
    EventListenerKeyboard *keylistener = EventListenerKeyboard::create();
    keylistener->onKeyPressed =
        CC_CALLBACK_2(TVPMainScene::onKeyPressed, this);
    keylistener->onKeyReleased =
        CC_CALLBACK_2(TVPMainScene::onKeyReleased, this);
    _eventDispatcher->addEventListenerWithFixedPriority(keylistener, 1);

    // 触摸事件（单点触摸）
    _touchListener = EventListenerTouchOneByOne::create();
    _touchListener->onTouchBegan =
        CC_CALLBACK_2(TVPMainScene::onTouchBegan, this);
    _touchListener->onTouchMoved =
        CC_CALLBACK_2(TVPMainScene::onTouchMoved, this);
    _touchListener->onTouchEnded =
        CC_CALLBACK_2(TVPMainScene::onTouchEnded, this);
    _touchListener->onTouchCancelled =
        CC_CALLBACK_2(TVPMainScene::onTouchCancelled, this);
    _eventDispatcher->addEventListenerWithSceneGraphPriority(
        _touchListener, this);

    // 手柄控制器事件
    EventListenerController *ctrllistener = EventListenerController::create();
    ctrllistener->onAxisEvent =
        CC_CALLBACK_3(TVPMainScene::onAxisEvent, this);
    ctrllistener->onKeyDown =
        CC_CALLBACK_3(TVPMainScene::onPadKeyDown, this);
    ctrllistener->onKeyUp =
        CC_CALLBACK_3(TVPMainScene::onPadKeyUp, this);
    ctrllistener->onKeyRepeat =
        CC_CALLBACK_3(TVPMainScene::onPadKeyRepeat, this);
    _eventDispatcher->addEventListenerWithSceneGraphPriority(
        ctrllistener, this);

    // 启动手柄发现（Windows 和 iOS）
    cocos2d::Controller::startDiscoveryController();
}
```

注意两种不同的优先级模式：

| 方法 | 用于 | 特点 |
|------|------|------|
| `addEventListenerWithFixedPriority(listener, 1)` | 键盘 | 固定优先级，不受节点树影响 |
| `addEventListenerWithSceneGraphPriority(listener, node)` | 触摸/手柄 | 优先级跟随节点的 zOrder |

键盘用固定优先级是因为无论当前 UI 焦点在哪，键盘输入都需要被场景捕获（例如 ESC 键退出）。触摸用场景图优先级是因为需要让最上层的 UI 元素优先处理触摸。

## 常见错误与解决方案

### 错误一：窗口创建后立即操作 UI 导致崩溃

```
症状：在 applicationDidFinishLaunching 中 runWithScene 后立即
      调用 pushUIForm，程序崩溃或 UI 位置错乱

原因：Scene 的 onEnter() 还未执行完毕，ContentSize 尚未生效

解决方案（KrKr2 的做法）：
  scene->scheduleOnce(callback, 0, "launch");
  // 延迟到下一帧，确保 Scene 完全就绪
```

### 错误二：设计分辨率与实际分辨率混淆

```cpp
// 错误示例：直接用 getFrameSize 定位 UI 元素
auto size = glview->getFrameSize();        // ← 这是物理像素！
sprite->setPosition(size.width / 2, 0);    // 在高 DPI 屏幕上位置偏移

// 正确做法：用设计分辨率（或 VisibleSize）
auto vsize = Director::getInstance()->getVisibleSize();
sprite->setPosition(vsize.width / 2, 0);   // 在所有分辨率下表现一致
```

### 错误三：Android 平台忘记处理横竖屏切换

```cpp
// KrKr2 的处理方式（AppDelegate.cpp 第 56-58 行）
if(screenSize.width < screenSize.height) {
    std::swap(screenSize.width, screenSize.height); // 强制横屏
}
// 如果忘记这步，竖屏手机上游戏会以竖屏模式显示，布局完全错乱
```

## 动手实践

### 实验一：跟踪启动流程

在以下位置添加日志，观察调用顺序：

```cpp
// 1. AppDelegate.cpp 的 initGLContextAttrs() 开头
spdlog::info("[Boot] Step 1: initGLContextAttrs");

// 2. applicationDidFinishLaunching() 开头
spdlog::info("[Boot] Step 2: applicationDidFinishLaunching");

// 3. TVPMainScene::create() 开头
spdlog::info("[Boot] Step 3: TVPMainScene::create");

// 4. TVPMainScene::initialize() 开头
spdlog::info("[Boot] Step 4: TVPMainScene::initialize");

// 5. scheduleOnce 的 lambda 内
spdlog::info("[Boot] Step 5: scheduled launch (next frame)");
```

预期输出：
```
[Boot] Step 1: initGLContextAttrs
[Boot] Step 2: applicationDidFinishLaunching
[Boot] Step 3: TVPMainScene::create
[Boot] Step 4: TVPMainScene::initialize
[Boot] Step 5: scheduled launch (next frame)
```

### 实验二：修改设计分辨率观察效果

```cpp
// 在 AppDelegate.cpp 中修改 designSize
static cocos2d::Size designSize(1280, 720);  // 原始 960×640 改为 1280×720

// 编译运行后观察：
// - 窗口初始大小变化
// - UI 元素相对位置是否正确
// - 文件选择器表单是否自适应新尺寸
```

### 实验三：对比 FIXED_WIDTH 与 EXACT_FIT

```cpp
// 在桌面端切换为 EXACT_FIT
glview->setDesignResolutionSize(
    designSize.width, designSize.height,
    ResolutionPolicy::EXACT_FIT);  // 原为 FIXED_WIDTH

// 然后手动拖拽窗口改变宽高比
// 观察：游戏画面是否变形（拉伸/压缩）
// 对比：FIXED_WIDTH 模式下拖拽窗口，画面不变形但高度自适应
```

## 对照项目源码

相关文件：
- `cpp/core/environ/cocos2d/AppDelegate.h` 全部 26 行 — 类声明，4 个虚函数重写
- `cpp/core/environ/cocos2d/AppDelegate.cpp` 全部 120 行 — 完整启动流程
- `cpp/core/environ/cocos2d/MainScene.h` 全部 94 行 — MainScene 公共接口，含 pushUIForm/popUIForm 等
- `cpp/core/environ/cocos2d/MainScene.cpp` 第 1753-1815 行 — 单例管理 + initialize() + create()
- `platforms/windows/main.cpp` 全部 44 行 — Windows 入口
- `platforms/linux/main.cpp` 全部 26 行 — Linux 入口
- `platforms/android/cpp/krkr2_android.cpp` — Android JNI 入口（节选）

关键常量定义：
- `designSize(960, 640)` — AppDelegate.cpp 第 12 行，基准设计分辨率
- `SCENE_ORDER` 枚举 — MainScene.cpp 第 39-45 行，节点层级顺序

## 本节小结

- KrKr2 四个平台的入口函数都遵循相同模式：初始化日志 → 处理命令行 → 创建 AppDelegate → run()
- `applicationDidFinishLaunching` 是真正的启动核心：创建 GLView → 设置分辨率 → 创建 MainScene → 延迟启动 UI
- 移动端用 EXACT_FIT（全屏拉伸），桌面端用 FIXED_WIDTH（宽度固定高度自适应）
- TVPMainScene 采用双节点树架构：GameNode（游戏内容，zOrder=0）+ UINode（模拟器 UI，zOrder=100）
- `scheduleOnce` 延迟一帧的模式解决了 Scene 未就绪时操作 UI 的竞态问题
- 三种事件监听器（键盘/触摸/手柄）使用不同优先级策略，匹配各自的交互特性

## 练习题与答案

### 题目 1：启动顺序排列

以下函数在 KrKr2 启动时的调用顺序是什么？请排列：
A. `TVPMainScene::initialize()`
B. `initGLContextAttrs()`
C. `director->runWithScene(scene)`
D. `applicationDidFinishLaunching()`
E. `scheduleOnce` 的 lambda 回调

<details>
<summary>查看答案</summary>

正确顺序：**B → D → A → C → E**

1. `initGLContextAttrs()` — 最先调用，在 `run()` 内部、创建窗口之前
2. `applicationDidFinishLaunching()` — `run()` 创建窗口后回调
3. `TVPMainScene::initialize()` — 在 `CreateInstance()` → `create()` 内部调用
4. `director->runWithScene(scene)` — 将场景交给 Director 管理
5. `scheduleOnce` 的 lambda — 下一帧才执行（延迟 0 帧 = 下一个 update 周期）

关键理解：E 虽然在代码中写在 `runWithScene` 之后，但 `scheduleOnce` 只是注册回调，实际执行要等到下一帧的 Scheduler 更新阶段。

</details>

### 题目 2：分析设计分辨率代码

阅读以下代码，回答问题：

```cpp
static cocos2d::Size designSize(960, 640);

// 移动端分支
cocos2d::Size screenSize = glview->getFrameSize();
if(screenSize.width < screenSize.height) {
    std::swap(screenSize.width, screenSize.height);
}
cocos2d::Size ds = designSize;
ds.height = ds.width * screenSize.height / screenSize.width;
glview->setDesignResolutionSize(screenSize.width, screenSize.height,
                                ResolutionPolicy::EXACT_FIT);
```

问：如果一台 Android 手机的屏幕分辨率是 2340×1080（竖屏持握），最终传给 `setDesignResolutionSize` 的参数是什么？

<details>
<summary>查看答案</summary>

**答案：`setDesignResolutionSize(2340, 1080, EXACT_FIT)`**

逐步推导：
1. `glview->getFrameSize()` 返回 `(1080, 2340)`（竖屏时 width < height）
2. `screenSize.width(1080) < screenSize.height(2340)` 为 true
3. `std::swap` 后：`screenSize = (2340, 1080)`（强制横屏）
4. `ds = designSize = (960, 640)`
5. `ds.height = 960 * 1080 / 2340 ≈ 443`（但这个 ds 变量实际没有被用到！）
6. 最终传入的是 `screenSize`：`(2340, 1080)`

注意：变量 `ds` 被计算了但**未被使用**，实际传给 `setDesignResolutionSize` 的是 `screenSize`。这可能是一个历史遗留代码，`ds` 原本可能用于不同的适配策略。

</details>

### 题目 3：实现一个简化版 AppDelegate

编写一个最小化的 Cocos2d-x AppDelegate，要求：
1. 创建 960×640 设计分辨率的窗口
2. 创建一个场景，场景内有两个 Node（GameLayer 和 UILayer），zOrder 分别为 0 和 100
3. 延迟一帧后在 UILayer 中添加一个 "Hello KrKr2" 的 Label

<details>
<summary>查看答案</summary>

```cpp
// MyAppDelegate.h
#pragma once
#include <cocos2d.h>

class MyAppDelegate : public cocos2d::Application {
    void initGLContextAttrs() override;
    bool applicationDidFinishLaunching() override;
    void applicationDidEnterBackground() override {}
    void applicationWillEnterForeground() override {}
};

// MyAppDelegate.cpp
#include "MyAppDelegate.h"
USING_NS_CC;

static cocos2d::Size designSize(960, 640);

void MyAppDelegate::initGLContextAttrs() {
    GLContextAttrs attrs = {8, 8, 8, 8, 24, 8};
    GLView::setGLContextAttrs(attrs);
}

bool MyAppDelegate::applicationDidFinishLaunching() {
    auto director = Director::getInstance();
    auto glview = director->getOpenGLView();
    if(!glview) {
        glview = GLViewImpl::create("My KrKr2");
        director->setOpenGLView(glview);
    }

    // 设计分辨率
    glview->setDesignResolutionSize(
        designSize.width, designSize.height,
        ResolutionPolicy::FIXED_WIDTH);

    director->setAnimationInterval(1.0f / 60);

    // 创建场景
    auto scene = Scene::create();

    // 双节点树架构
    auto gameLayer = Node::create();
    gameLayer->setContentSize(designSize);
    scene->addChild(gameLayer, 0);        // zOrder = 0

    auto uiLayer = Node::create();
    uiLayer->setContentSize(designSize);
    scene->addChild(uiLayer, 100);        // zOrder = 100

    director->runWithScene(scene);

    // 延迟一帧添加 Label（模仿 KrKr2 的 scheduleOnce 模式）
    scene->scheduleOnce(
        [uiLayer](float dt) {
            auto label = Label::createWithSystemFont(
                "Hello KrKr2", "Arial", 48);
            auto size = uiLayer->getContentSize();
            label->setPosition(size.width / 2, size.height / 2);
            uiLayer->addChild(label);
        },
        0, "delayed_init");

    return true;
}

// main.cpp（桌面端入口）
#include "MyAppDelegate.h"
int main(int argc, char **argv) {
    MyAppDelegate app;
    return app.run();
}
```

这个简化版保留了 KrKr2 的三个核心设计模式：
1. **双节点树**（GameLayer + UILayer 分离）
2. **延迟初始化**（scheduleOnce 避免竞态）
3. **设计分辨率适配**（FIXED_WIDTH 策略）

</details>

## 下一步

[渲染桥接与窗口层](./02-渲染桥接与窗口层.md) — 深入分析 TVPWindowLayer 如何将 KiriKiri 引擎的窗口系统桥接到 Cocos2d-x 的节点树中，实现游戏画面的渲染与触摸交互。
