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

## 二、TVPMainScene：双层场景架构

### 2.1 场景节点层级

`TVPMainScene` 继承自 `cocos2d::Scene` 和 `cocos2d::IMEDelegate`，是整个游戏运行时的唯一场景。它的核心设计是将**游戏渲染**和 **UI 表单**分离到两个独立的节点树中：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 39-45 行
enum SCENE_ORDER {
    GAME_SCENE_ORDER,    // 0 — 游戏窗口层（最底层）
    GAME_CONSOLE_ORDER,  // 1 — 控制台窗口
    GAME_WINMGR_ORDER,   // 2 — 窗口管理覆盖层 / 虚拟鼠标光标
    GAME_MENU_ORDER,     // 3 — 游戏内菜单
    UI_NODE_ORDER,       // 4 — UI 表单栈（最顶层）
};
```

场景初始化的完整流程：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 1762-1815 行
void TVPMainScene::initialize() {
    auto glview = cocos2d::Director::getInstance()->getOpenGLView();
    cocos2d::Size screenSize = glview->getFrameSize();
    cocos2d::Size designSize = glview->getDesignResolutionSize();
    // 根据物理屏幕比例重新计算设计分辨率
    ScreenRatio = screenSize.height / designSize.height;
    designSize.width = designSize.height * screenSize.width / screenSize.height;
    initWithSize(designSize);

    // 黑色背景
    addChild(LayerColor::create(Color4B::BLACK, designSize.width, designSize.height));

    // 游戏渲染节点 — z-order = 0（最底层）
    GameNode = cocos2d::Node::create();
    GameNode->setContentSize(designSize);
    GameNode->setAnchorPoint(Vec2(0, 0));
    addChild(GameNode, GAME_SCENE_ORDER);

    // UI 表单节点 — z-order = 4（最顶层）
    UINode = cocos2d::Node::create();
    UINode->setContentSize(designSize);
    UINode->setAnchorPoint(Vec2::ZERO);
    UINode->setPosition(Vec2::ZERO);
    UISize = designSize;
    addChild(UINode, UI_NODE_ORDER);

    // 注册键盘事件监听器（固定优先级 1）
    EventListenerKeyboard *keylistener = EventListenerKeyboard::create();
    keylistener->onKeyPressed = CC_CALLBACK_2(TVPMainScene::onKeyPressed, this);
    keylistener->onKeyReleased = CC_CALLBACK_2(TVPMainScene::onKeyReleased, this);
    _eventDispatcher->addEventListenerWithFixedPriority(keylistener, 1);

    // 注册触摸事件监听器（场景级别）
    _touchListener = EventListenerTouchOneByOne::create();
    _touchListener->onTouchBegan = CC_CALLBACK_2(TVPMainScene::onTouchBegan, this);
    _touchListener->onTouchMoved = CC_CALLBACK_2(TVPMainScene::onTouchMoved, this);
    _touchListener->onTouchEnded = CC_CALLBACK_2(TVPMainScene::onTouchEnded, this);
    _touchListener->onTouchCancelled = CC_CALLBACK_2(TVPMainScene::onTouchCancelled, this);
    _eventDispatcher->addEventListenerWithSceneGraphPriority(_touchListener, this);

    // 注册手柄/游戏控制器事件监听器
    EventListenerController *ctrllistener = EventListenerController::create();
    ctrllistener->onAxisEvent = CC_CALLBACK_3(TVPMainScene::onAxisEvent, this);
    ctrllistener->onKeyDown = CC_CALLBACK_3(TVPMainScene::onPadKeyDown, this);
    ctrllistener->onKeyUp = CC_CALLBACK_3(TVPMainScene::onPadKeyUp, this);
    ctrllistener->onKeyRepeat = CC_CALLBACK_3(TVPMainScene::onPadKeyRepeat, this);
    _eventDispatcher->addEventListenerWithSceneGraphPriority(ctrllistener, this);
    cocos2d::Controller::startDiscoveryController();  // 开始扫描手柄设备
}
```

### 2.2 双节点设计的意义

```
TVPMainScene
├── LayerColor (黑色背景)
├── GameNode (z=0)           ← 游戏内容在这里渲染
│   ├── TVPWindowLayer       ← KiriKiri 游戏窗口（可多个）
│   ├── ConsoleWindow (z=1)  ← 启动时的控制台输出
│   ├── WindowMgrOverlay(z=2)← 窗口切换界面
│   └── GameMainMenu (z=3)   ← 游戏内菜单（汉堡菜单）
└── UINode (z=4)             ← UI 表单栈在这里
    ├── GlobalPreferenceForm ← 全局设置
    ├── FileSelectorForm     ← 文件选择器
    └── ...更多 UI 表单...
```

这种分离设计有三个重要优势：

1. **渲染隔离**：游戏内容和 UI 完全独立渲染，UI 叠加在游戏上方不会干扰游戏的渲染管线
2. **事件隔离**：当 UINode 有子节点时（即有 UI 表单打开），触摸事件只传递给 UI，不传递给游戏层（见 `onTouchBegan` 的判断逻辑）
3. **生命周期独立**：游戏运行时可以随时 push/pop UI 表单，游戏逻辑不受影响

### 2.3 游戏启动流程

当用户在文件选择器中选择一个游戏后，调用 `startupFrom(path)`：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 1926-1963 行
bool TVPMainScene::startupFrom(const std::string &path) {
    if(!TVPCheckStartupPath(path)) return false;

    // 加载该游戏的个性化配置
    IndividualConfigManager *pGlobalCfgMgr = IndividualConfigManager::GetInstance();
    pGlobalCfgMgr->UsePreferenceAt(TVPBaseFileSelectorForm::pathSplit(path).first);

    // 关闭所有 UI 表单
    if(UINode->getChildrenCount()) {
        popUIForm(nullptr);
    }

    // 保持屏幕常亮（游戏运行中不自动锁屏）
    if(GlobalConfigManager::GetInstance()->GetValue<bool>("keep_screen_alive", true)) {
        Device::setKeepScreenOn(true);
    }

    // 初始化按键映射表（支持用户自定义键位）
    for(int i = 0; i < std::size(_keymap); ++i) {
        _keymap[i] = i;  // 默认1:1映射
    }
    const auto &keymap = pGlobalCfgMgr->GetKeyMap();
    for(const auto &it : keymap) {
        if(!it.second) continue;
        _keymap[it.first] = _keymap[it.second];  // 应用用户自定义映射
    }

    // 延迟一帧执行实际启动（确保 UI 动画完成）
    scheduleOnce([this, path](float delay) { doStartup(delay, path); }, 0, "startup");
    return true;
}
```

`doStartup` 是实际的引擎启动逻辑：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 1965-2013 行
void TVPMainScene::doStartup(float dt, std::string path) {
    unschedule("startup");

    // 1. 创建控制台窗口（显示启动日志）
    _consoleWin = TVPConsoleWindow::create(24, nullptr);
    auto glview = cocos2d::Director::getInstance()->getOpenGLView();
    cocos2d::Size screenSize = glview->getFrameSize();
    float scale = screenSize.height / getContentSize().height;
    _consoleWin->setScale(1 / scale);
    _consoleWin->setContentSize(getContentSize() * scale);
    GameNode->addChild(_consoleWin, GAME_CONSOLE_ORDER);

    // 2. 启动 KiriKiri 引擎核心 — 这里会加载 TJS2 脚本、初始化所有子系统
    ::Application->StartApplication(path);

    // 3. 执行一帧更新（让引擎创建游戏窗口）
    update(0);

    // 4. 创建游戏内菜单
    GLubyte handlerOpacity =
        pGlobalCfgMgr->GetValue<float>("menu_handler_opa", 0.15f) * 255;
    _gameMenu = TVPGameMainMenu::create(handlerOpacity);
    GameNode->addChild(_gameMenu, GAME_MENU_ORDER);
    _gameMenu->shrinkWithTime(1);  // 1秒后自动收缩菜单

    // 5. 移除控制台，开始正常渲染
    if(_consoleWin) {
        _consoleWin->removeFromParent();
        _consoleWin = nullptr;
        scheduleUpdate();  // 开始每帧调用 update()
        cocos2d::Director::getInstance()->purgeCachedData();
    }

    // 6. 显示所有游戏窗口
    TVPWindowLayer *pWin = _lastWindowLayer;
    while(pWin) { pWin->setVisible(true); pWin = pWin->_prevWindow; }

    // 7. 可选的 FPS 显示
    if(pGlobalCfgMgr->GetValue<bool>("showfps", false)) {
        _fpsLabel = cocos2d::Label::createWithTTF("", "NotoSansCJK-Regular.ttc", 16);
        _fpsLabel->setAnchorPoint(Vec2(0, 1));
        _fpsLabel->setPosition(Vec2(0, GameNode->getContentSize().height));
        GameNode->addChild(_fpsLabel, GAME_MENU_ORDER);
    }

    // 8. 设置帧率限制
    int fps = pGlobalCfgMgr->GetValue<int>("fps_limit", 60);
    cocos2d::Director::getInstance()->setAnimationInterval(1.0f / fps);
}
```

### 2.4 主循环更新

每帧的 `update()` 函数是引擎的心脏跳动：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 2021-2070 行
void TVPMainScene::update(float delta) {
    ::Application->Run();              // 驱动 KiriKiri 引擎执行一帧逻辑
    iTVPTexture2D::RecycleProcess();   // 回收不再使用的 GPU 纹理
    if(_postUpdate) _postUpdate();     // 执行后置更新回调

    // FPS 统计显示（可选）
    if(_fpsLabel) {
        unsigned int drawCount;
        uint64_t vmemsize;
        TVPGetRenderManager()->GetRenderStat(drawCount, vmemsize);
        // ... 平滑 FPS 计算和显示 ...
    }
}
```

核心就是 `::Application->Run()`，它会驱动 TJS2 脚本执行、处理事件队列、更新动画状态。KrKr2 的游戏逻辑完全由这个调用驱动，Cocos2d-x 只负责"每帧调用一次"。

---

## 三、TVPWindowLayer：触摸到鼠标的事件转换

### 3.1 类设计

`TVPWindowLayer` 是桥接层中最复杂的类（约 1250 行），它继承自 `cocos2d::extension::ScrollView` 和 `iWindowLayer` 接口：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 365-398 行
class TVPWindowLayer : public cocos2d::extension::ScrollView,
                       public iWindowLayer {
    typedef cocos2d::extension::ScrollView inherit;
    tTJSNI_Window *TJSNativeInstance;      // 对应的 TJS2 窗口对象
    tjs_int ActualZoomDenom;               // 缩放分母
    tjs_int ActualZoomNumer;               // 缩放分子
    Sprite *DrawSprite = nullptr;          // 游戏画面绘制精灵
    Node *PrimaryLayerArea = nullptr;      // 主渲染区域节点
    int LayerWidth = 0, LayerHeight = 0;   // 游戏画面尺寸
    TVPWindowLayer *_prevWindow, *_nextWindow;  // 双向链表（多窗口管理）
    // ...
};
```

**为什么继承 ScrollView？** 因为 KiriKiri 游戏的画面分辨率可能与设备屏幕不匹配。ScrollView 提供了内置的缩放（pinch-zoom）和平移（pan）功能，让用户可以放大/缩小/拖动游戏画面。

### 3.2 触摸事件到鼠标事件的映射规则

这是整个桥接层最精妙的设计——将移动端多点触摸映射为桌面端鼠标操作：

```
┌──────────────────────────────────────────────┐
│            触摸 → 鼠标 映射规则               │
├──────────────────────────────────────────────┤
│  1 个手指触摸   → 鼠标左键 (mbLeft)           │
│  2 个手指触摸   → 鼠标右键 (mbRight)          │
│  3 个手指触摸   → 鼠标中键 (mbMiddle)         │
│                                              │
│  触摸后立即抬起  → 点击 (Click)               │
│  触摸后 >150ms  → 拖拽 (Drag)                │
│  或移动超过阈值                               │
└──────────────────────────────────────────────┘
```

核心实现在 `onTouchBegan` 中：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 566-593 行
bool onTouchBegan(Touch *touch, Event *unused_event) override {
    if(_windowMgrOverlay)
        return inherit::onTouchBegan(touch, unused_event);  // 窗口管理模式走 ScrollView 逻辑

    if(std::find(_touches.begin(), _touches.end(), touch) == _touches.end()) {
        _touches.push_back(touch);  // 记录当前所有触摸点
    }

    switch(_touches.size()) {
        case 1:  // 单指 → 左键
            _touchPoint = touch->getLocation();
            _touchMoved = false;
            _touchLength = 0.0f;
            _touchBeginTick = TVPGetRoughTickCount32();  // 记录按下时间
            _mouseBtn = ::mbLeft;
            break;
        case 2:  // 双指 → 右键
            _mouseBtn = ::mbRight;
            _touchPoint = (_touchPoint + touch->getLocation()) / 2;  // 取两指中点
            break;
        case 3:  // 三指 → 中键
            _mouseBtn = ::mbMiddle;
        default: break;
    }
    return true;
}
```

触摸移动的处理逻辑包含拖拽判定：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 595-631 行
void onTouchMoved(Touch *touch, Event *unused_event) override {
    if(_touches.size() == 1) {
        if(!_touchMoved &&
           (TVPGetRoughTickCount32() - _touchBeginTick > 150 ||    // 超过 150ms
            _touchPoint.getDistanceSq(touch->getLocation()) >
                _touchMoveThresholdSq)) {                           // 或移动超过阈值
            // 第一次判定为拖拽：先发送 MouseMove + MouseDown
            Vec2 nsp = PrimaryLayerArea->convertToNodeSpace(_touchPoint);
            _LastMouseX = nsp.x;
            _LastMouseY = PrimaryLayerArea->getContentSize().height - nsp.y;
            // 注意 Y 轴翻转：Cocos2d-x 是左下角原点，KiriKiri 是左上角原点
            TVPPostInputEvent(new tTVPOnMouseMoveInputEvent(...));
            TVPPostInputEvent(new tTVPOnMouseDownInputEvent(...));
            _touchMoved = true;
        } else if(_touchMoved) {
            // 后续拖拽：持续发送 MouseMove（可丢弃优先级）
            TVPPostInputEvent(new tTVPOnMouseMoveInputEvent(...),
                              TVP_EPT_DISCARDABLE);  // 可丢弃 = 如果队列积压可跳过
        }
    }
}
```

> **坐标系转换要点：** Cocos2d-x 使用左下角为原点的坐标系（Y 轴向上），而 KiriKiri 使用左上角为原点（Y 轴向下）。转换公式为：`krkr_y = contentSize.height - cocos_y`。这个转换贯穿整个 TVPWindowLayer。

### 3.3 触摸结束 — 点击 vs 拖拽

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 634-673 行
void onTouchEnded(Touch *touch, Event *unused_event) override {
    if(_touches.size() == 1) {
        Vec2 nsp = PrimaryLayerArea->convertTouchToNodeSpace(touch);
        _LastMouseX = nsp.x;
        _LastMouseY = PrimaryLayerArea->getContentSize().height - nsp.y;

        if(!_touchMoved) {
            // 没有移动过 = 这是一次点击
            TVPPostInputEvent(new tTVPOnMouseMoveInputEvent(...));
            TVPPostInputEvent(new tTVPOnMouseDownInputEvent(...));
            TVPPostInputEvent(new tTVPOnClickInputEvent(...));  // 额外的 Click 事件
        }
        // 无论点击还是拖拽，最后都发送 MouseUp
        TVPPostInputEvent(new tTVPOnMouseUpInputEvent(...));
    }
    _touches.erase(touchIter);
}
```

完整的事件序列对比：

| 操作 | 事件序列 | 对应桌面操作 |
|------|----------|-------------|
| 单指快速点击 | Move → Down → Click → Up | 鼠标左键单击 |
| 单指长按+拖动 | Move → Down → Move... → Up | 鼠标左键拖拽 |
| 双指点击 | (同上，但 button=mbRight) | 鼠标右键单击 |
| 三指点击 | (同上，但 button=mbMiddle) | 鼠标中键单击 |

### 3.4 模态窗口的特殊处理

KiriKiri 引擎支持模态窗口（阻塞式对话框），这在事件驱动的 Cocos2d-x 中需要特殊实现：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 835-862 行
void ShowWindowAsModal() override {
    in_mode_ = true;
    setVisible(true);
    BringToFront();

    // 如果控制台还在，先移除
    if(_consoleWin) {
        _consoleWin->removeFromParent();
        _consoleWin = nullptr;
        TVPMainScene::GetInstance()->scheduleUpdate();
    }

    Director *director = Director::getInstance();
    modal_result_ = 0;
    // 阻塞式主循环 — 在模态窗口关闭前不会返回
    while(this == _currentWindowLayer && !modal_result_) {
        int remain = TVPDrawSceneOnce(30);  // 以 30fps 手动驱动渲染
        TVPProcessInputEvents();             // 处理输入事件
        if(::Application->IsTarminate()) {
            modal_result_ = mrCancel;        // 应用退出时强制取消
        } else if(modal_result_ != 0) {
            break;
        } else if(remain > 0) {
            std::this_thread::sleep_for(std::chrono::milliseconds(remain));
        }
    }
    in_mode_ = false;
}
```

> **常见问题：为什么模态窗口用 while 循环而不是回调？** 因为原版 KiriKiri 的 TJS2 脚本使用同步阻塞式的模态对话框 API。如果改为异步回调，所有使用模态对话框的游戏脚本都需要重写。这是一个务实的兼容性选择——牺牲一点优雅换取 100% 的脚本兼容性。

### 3.5 多窗口管理

TVPWindowLayer 使用双向链表管理多个游戏窗口：

```cpp
// 构造函数中将自己加入链表
TVPWindowLayer(tTJSNI_Window *w) : TJSNativeInstance(w) {
    _nextWindow = nullptr;
    _prevWindow = _lastWindowLayer;
    _lastWindowLayer = this;
    if(_prevWindow) _prevWindow->_nextWindow = this;
}

// 析构函数中从链表移除
~TVPWindowLayer() override {
    if(_lastWindowLayer == this) _lastWindowLayer = _prevWindow;
    if(_nextWindow) _nextWindow->_prevWindow = _prevWindow;
    if(_prevWindow) _prevWindow->_nextWindow = _nextWindow;
    // 如果当前窗口被销毁，自动切换到另一个可见窗口
    if(_currentWindowLayer == this) { /* ... 查找替代窗口 ... */ }
}
```

全局静态变量 `_currentWindowLayer` 指向当前活跃的窗口，`_lastWindowLayer` 指向链表尾部。`BringToFront()` 方法将指定窗口设为活跃窗口，隐藏之前的窗口。

---

## 四、UI 表单栈系统

KrKr2 的 UI 层（文件选择器、全局设置、游戏内偏好等）使用"表单栈"模式管理：`pushUIForm()` 将新表单压入栈顶，`popUIForm()` 弹出栈顶表单，`popAllUIForm()` 批量关闭所有表单。这三个方法定义在 `TVPMainScene` 中，操作的容器是 `UINode` 节点。

### 4.1 动画类型枚举

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.h 第 27-39 行
enum eEnterAni {
    eEnterAniNone,           // 无动画，直接显示
    eEnterAniOverFromRight,  // 从右侧滑入（默认）
    eEnterFromBottom,        // 从底部弹出
};

enum eLeaveAni {
    eLeaveAniNone,           // 无动画，直接移除
    eLeaveAniLeaveFromLeft,  // 向左滑出（默认，与从右滑入对应）
    eLeaveToBottom,          // 向下收回（与从底部弹出对应）
};
```

**设计思路：** 入场和退场动画成对匹配。`eEnterAniOverFromRight` 搭配 `eLeaveAniLeaveFromLeft`，`eEnterFromBottom` 搭配 `eLeaveToBottom`。这样用户心理模型保持一致——表单从哪来就回哪去。

### 4.2 pushUIForm — 表单入栈

`pushUIForm` 的核心逻辑是将新 UI 节点添加到 `UINode`，同时根据动画类型执行过渡效果：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 1827-1872 行
void TVPMainScene::pushUIForm(cocos2d::Node *ui, eEnterAni ani) {
    if(!ui) {
        CCLOGERROR("TVPMainScene::pushUIForm: ui is nullptr");
        return;
    }
    TVPControlAdDialog(0x10002, 1, 0);  // 通知广告系统（Android 端可能展示广告）
    int n = UINode->getChildrenCount();  // 当前已有多少个表单

    if(ani == eEnterAniNone) {
        UINode->addChild(ui);  // 无动画，直接添加

    } else if(ani == eEnterAniOverFromRight) {
        if(n > 0) {
            cocos2d::Size size = UINode->getContentSize();
            // 把当前栈顶表单向左移动 1/5 宽度（"被推开"的效果）
            cocos2d::Node *lastui = UINode->getChildren().back();
            lastui->runAction(EaseQuadraticActionOut::create(
                MoveTo::create(UI_CHANGE_DURATION, Vec2(size.width / -5, 0))));

            // 创建半透明黑色遮罩（视觉层次感）
            cocos2d::Node *ColorMask =
                MaskLayer::create(Color4B(0, 0, 0, 0), size.width, size.height);
            ColorMask->setPosition(Vec2(-size.width, 0));
            ui->addChild(ColorMask);
            ColorMask->runAction(FadeTo::create(UI_CHANGE_DURATION, 128));

            // 新表单从屏幕右侧滑入
            ui->setPosition(size.width, 0);
            ui->runAction(EaseQuadraticActionOut::create(
                MoveTo::create(UI_CHANGE_DURATION, Vec2::ZERO)));

            // 动画结束后移除遮罩
            runAction(Sequence::createWithTwoActions(
                DelayTime::create(UI_CHANGE_DURATION),
                CallFunc::create([=]() { ColorMask->removeFromParent(); })));
        }
        UINode->addChild(ui);

    } else if(ani == eEnterFromBottom) {
        cocos2d::Size size = UINode->getContentSize();
        // 底部弹出式：先创建全屏遮罩，将表单作为遮罩的子节点
        cocos2d::Node *ColorMask =
            MaskLayer::create(Color4B(0, 0, 0, 0), size.width, size.height);
        ColorMask->runAction(FadeTo::create(UI_CHANGE_DURATION, 128));
        ui->setPositionY(-ui->getContentSize().height);  // 初始位置在屏幕下方
        ColorMask->addChild(ui);
        UINode->addChild(ColorMask);  // 遮罩加入UINode，表单是遮罩的子节点
        ui->runAction(EaseQuadraticActionOut::create(
            MoveTo::create(UI_CHANGE_DURATION, Vec2::ZERO)));  // 向上滑入
    }
}
```

**关键细节解读：**

1. **MaskLayer 的双重作用**：它不仅提供视觉上的半透明黑色遮罩效果（从透明渐变到 50% 黑色，即 alpha=128），还通过拦截触摸事件防止用户在过渡动画期间误触
2. **`eEnterFromBottom` 的父子关系差异**：底部弹出式的表单是 `MaskLayer` 的子节点（而不是 `UINode` 的直接子节点），这意味着 `popUIForm` 需要特殊处理这种情况
3. **`EaseQuadraticActionOut`**：使用缓动函数让动画开始时快、结束时慢，感觉更自然
4. **`UI_CHANGE_DURATION`**：一个常量（通常 0.3 秒），控制所有 UI 过渡动画的时长

### 4.3 popUIForm — 表单出栈

出栈逻辑与入栈对称，但需要处理 `eEnterFromBottom` 创建的特殊父子关系：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 1874-1924 行
void TVPMainScene::popUIForm(cocos2d::Node *form, eLeaveAni ani) {
    int n = UINode->getChildrenCount();
    if(n <= 0) return;
    if(n == 1) {
        TVPControlAdDialog(0x10002, 0, 0);  // 最后一个表单关闭时通知广告系统
    }
    auto children = UINode->getChildren();

    if(ani == eLeaveAniNone) {
        // 无动画：直接移除，并恢复上一个表单的位置
        if(n > 1) {
            Node *lastui = children.at(n - 2);
            lastui->setPosition(0, 0);  // 恢复原位
        }
        Node *ui = children.back();
        if(form) CCAssert(form == ui, "must be the same form");
        ui->removeFromParent();

    } else if(ani == eLeaveAniLeaveFromLeft) {
        Node *ui = children.back();
        cocos2d::Size size = UINode->getContentSize();
        if(n > 1) {
            // 上一个表单从 -1/5 宽度滑回原位
            Node *lastui = children.at(n - 2);
            lastui->setPosition(size.width / -5, 0);
            lastui->runAction(EaseQuadraticActionOut::create(
                MoveTo::create(UI_CHANGE_DURATION, Vec2::ZERO)));
        }
        // 创建遮罩+当前表单向右滑出
        cocos2d::Node *ColorMask =
            MaskLayer::create(Color4B(0, 0, 0, 128), size.width, size.height);
        ColorMask->setPosition(Vec2(-size.width, 0));
        ui->addChild(ColorMask);
        ColorMask->runAction(FadeOut::create(UI_CHANGE_DURATION));
        ui->runAction(EaseQuadraticActionOut::create(
            MoveTo::create(UI_CHANGE_DURATION, Vec2(size.width, 0))));
        // 动画结束后移除节点
        this->runAction(Sequence::createWithTwoActions(
            DelayTime::create(UI_CHANGE_DURATION),
            CallFunc::create([ui] { ui->removeFromParent(); })));

    } else if(ani == eLeaveToBottom) {
        // 底部弹出的表单：遮罩是 UINode 的子节点，表单是遮罩的子节点
        cocos2d::Node *ColorMask = children.back();  // 这里取到的是遮罩
        ColorMask->runAction(FadeOut::create(UI_CHANGE_DURATION));
        Node *ui = ColorMask->getChildren().at(0);   // 真正的表单
        ui->runAction(EaseQuadraticActionIn::create(
            MoveTo::create(UI_CHANGE_DURATION,
                           Vec2(0, -ui->getContentSize().height))));  // 向下滑出
        runAction(Sequence::createWithTwoActions(
            DelayTime::create(UI_CHANGE_DURATION),
            CallFunc::create([=]() { ColorMask->removeFromParent(); })));
    }
}
```

### 4.4 popAllUIForm — 批量关闭

当游戏启动时（`startupFrom` 调用），需要一次性关闭所有 UI 表单：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 2273-2289 行
void TVPMainScene::popAllUIForm() {
    TVPControlAdDialog(0x10002, 0, 0);
    auto children = UINode->getChildren();
    for(auto ui : children) {
        cocos2d::Size size = getContentSize();
        cocos2d::Node *ColorMask =
            MaskLayer::create(Color4B(0, 0, 0, 128), size.width, size.height);
        ColorMask->setPosition(Vec2(-size.width, 0));
        ui->addChild(ColorMask);
        ColorMask->runAction(FadeOut::create(UI_CHANGE_DURATION));
        // 所有表单统一向右滑出
        ui->runAction(EaseQuadraticActionOut::create(
            MoveTo::create(UI_CHANGE_DURATION, Vec2(size.width, 0))));
        runAction(Sequence::createWithTwoActions(
            DelayTime::create(UI_CHANGE_DURATION),
            CallFunc::create([=]() { ui->removeFromParent(); })));
    }
}
```

> **注意：** `popAllUIForm` 不区分表单的入场类型，统一使用向右滑出动画。这是一个简化设计——批量关闭时不需要逐个匹配退出动画。

### 4.5 iTVPBaseForm — 表单基类

所有 UI 表单（文件选择器、全局设置、游戏偏好等）都继承自 `iTVPBaseForm`。这个基类定义了标准的三段式布局：

```
┌──────────────────────────────────┐
│         Header (10%)             │  ← 导航栏：返回按钮 + 标题
├──────────────────────────────────┤
│                                  │
│         Body   (80%)             │  ← 主体内容区域
│                                  │
├──────────────────────────────────┤
│         Footer (10%)             │  ← 底部工具栏
└──────────────────────────────────┘
```

```cpp
// 文件：cpp/core/environ/ui/BaseForm.h 第 83-150 行
class iTVPBaseForm : public cocos2d::Node {
public:
    iTVPBaseForm() : RootNode(nullptr) {}
    void Show();
    virtual void rearrangeLayout();
    virtual void onKeyPressed(cocos2d::EventKeyboard::KeyCode keyCode,
                              cocos2d::Event *event);

protected:
    // 三段式布局初始化：传入导航栏、主体、底栏的构建函数
    bool initFromFile(const Csd::NodeBuilderFn &naviBarCall,
                      const Csd::NodeBuilderFn &bodyCall,
                      const Csd::NodeBuilderFn &bottomBarCall,
                      Node *parent = nullptr);

    // 布局尺寸计算 —— 按比例分配屏幕高度
    static cocos2d::Size rearrangeHeaderSize(const Node *parent) {
        const auto &pSize = parent->getContentSize();
        return { pSize.width, pSize.height * 0.1f };  // 高度的 10%
    }
    static cocos2d::Size rearrangeBodySize(const Node *parent) {
        const auto &pSize = parent->getContentSize();
        return { pSize.width, pSize.height * 0.8f };  // 高度的 80%
    }
    static cocos2d::Size rearrangeFooterSize(const Node *parent) {
        const auto &pSize = parent->getContentSize();
        return { pSize.width, pSize.height * 0.1f };  // 高度的 10%
    }

    // 子类必须实现的三个绑定方法
    virtual void bindHeaderController(const Node *allNodes) = 0;
    virtual void bindBodyController(const Node *allNodes) = 0;
    virtual void bindFooterController(const Node *allNodes) = 0;

    cocos2d::ui::Widget *RootNode;
    struct {
        cocos2d::ui::Button *Left;   // 导航栏左按钮（通常是返回）
        cocos2d::ui::Button *Right;  // 导航栏右按钮
        cocos2d::Node *Root;
    } NaviBar{};
    struct {
        cocos2d::Node *Root;
    } BottomBar{};
};
```

**关键设计点：**

1. **`Csd::NodeBuilderFn`** 是一个工厂函数类型，负责从 `.csb` 文件（Cocos Studio 导出的二进制 UI 布局文件）加载节点树。UI 资源文件位于 `ui/cocos-studio/` 目录
2. **`NodeMap`** 类（`BaseForm.h` 第 44-71 行）提供按名称查找 UI 控件的能力：`findController<Button>("btn_back")` 可以在整棵节点树中查找名为 `btn_back` 的按钮
3. **`onKeyPressed`** 虚方法使得每个表单可以处理键盘事件（如返回键）。回顾 `TVPMainScene::onKeyPressed`，当 UINode 有子节点时，键盘事件会优先分发给栈顶表单
4. **`iTVPFloatForm`** 是另一个基类变体，用于不需要三段式布局的浮动面板（如弹出菜单），它空实现了三个 `bind*Controller` 方法

### 4.6 事件隔离机制

UI 表单栈的一个重要特性是**事件隔离**：当有表单打开时，触摸和键盘事件只传递给 UI 层，游戏层不会收到任何输入。

```cpp
// TVPMainScene::onTouchBegan 中的判断（第 2334 行）
if(UINode->getChildrenCount())
    return false;  // UINode 有子节点（表单打开中）→ 不处理游戏层触摸

// TVPMainScene::onKeyPressed 中的判断（第 2131-2137 行）
Vector<Node *> &uiChild = UINode->getChildren();
if(!uiChild.empty()) {
    // 键盘事件转发给栈顶表单
    iTVPBaseForm *uiform = static_cast<iTVPBaseForm *>(uiChild.back());
    if(uiform) uiform->onKeyPressed(keyCode, event);
    return;  // 不再传递给游戏窗口
}
```

这个设计确保了：打开设置界面时不会误触游戏、在文件选择器中按键不会影响游戏逻辑。整个 UI 表单栈系统的数据流如下：

```
用户操作 → Cocos2d-x 事件分发器
           ├─ UINode 有子节点？ → 是 → 栈顶 iTVPBaseForm::onKeyPressed()
           │                    → 否 → TVPMainScene 处理 → 转发给 TVPWindowLayer
           └─ 触摸事件同理
```

---

## 五、输入系统总览

KrKr2 桥接层支持四种输入方式，统一转换为 KiriKiri 引擎的键盘/鼠标事件：

```
┌─────────────────────────────────────────────────────────────┐
│                    KrKr2 输入系统架构                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  物理键盘 ─→ onKeyPressed/Released ─┐                       │
│                                     │                       │
│  手柄按键 ─→ onPadKeyDown/Up ───────┼→ VK Code → _keymap[] │
│                                     │    → InternalKeyDown  │
│  手柄摇杆 ─→ onAxisEvent ───────────┼→ 虚拟鼠标移动         │
│                                     │                       │
│  屏幕触摸 ─→ onTouchBegan/Moved/End ┼→ 鼠标事件(左/右/中键) │
│                                     │  或虚拟鼠标模式        │
│  软键盘   ─→ insertText ────────────┼→ OnKeyPress (字符)    │
│                                     │                       │
│                   ↓                 │                       │
│         TVPWindowLayer 接收统一事件  │                       │
│         → TJS2 脚本层处理           │                       │
└─────────────────────────────────────────────────────────────┘
```

### 5.1 键盘输入处理

键盘事件的处理流程分为三步：特殊键处理 → VK 码转换 → 按键映射 → 分发给窗口。

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 2130-2189 行
void TVPMainScene::onKeyPressed(EventKeyboard::KeyCode keyCode, Event *event) {
    // 第一步：UI 层拦截 — 如果有表单打开，键盘事件只给表单
    Vector<Node *> &uiChild = UINode->getChildren();
    if(!uiChild.empty()) {
        iTVPBaseForm *uiform = static_cast<iTVPBaseForm *>(uiChild.back());
        if(uiform) uiform->onKeyPressed(keyCode, event);
        return;
    }

    // 第二步：特殊按键处理
    switch(keyCode) {
        case EventKeyboard::KeyCode::KEY_MENU:
            // Android 菜单键 → 切换游戏内汉堡菜单
            if(_gameMenu) _gameMenu->toggle();
            return;
        case EventKeyboard::KeyCode::KEY_BACK:
            // Android 返回键 → 映射为 Escape 键
            if(_gameMenu && !_gameMenu->isShrinked()) {
                _gameMenu->shrink();  // 先尝试收起菜单
                return;
            }
            keyCode = EventKeyboard::KeyCode::KEY_ESCAPE;
            break;
        default: break;
    }

    // 第三步：Cocos2d-x KeyCode → Windows VK 码
    unsigned int code = TVPConvertKeyCodeToVKCode(keyCode);
    if(!code || code >= 0x200) return;  // 无效键码，丢弃

    // 第四步：应用用户自定义键位映射
    code = _keymap[code];

    // 第五步：更新按键状态 & 分发
    _scancode[code] = 0x11;  // bit 0 = 当前按下, bit 4 = 曾经按下
    if(_currentWindowLayer) {
        _currentWindowLayer->InternalKeyDown(code, TVPGetCurrentShiftKeyState());
    }
}
```

**`_scancode` 数组详解：** 每个元素是一个位域，bit 0 表示"当前正在按下"，bit 4 表示"自上次查询后曾经被按下过"。`0x11` 表示两个标志同时置位。在 `onKeyReleased` 中：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 2192-2248 行
void TVPMainScene::onKeyReleased(EventKeyboard::KeyCode keyCode, Event *event) {
    // KEY_BACK → KEY_ESCAPE 的映射同样在释放时生效
    if(keyCode == EventKeyboard::KeyCode::KEY_BACK)
        keyCode = EventKeyboard::KeyCode::KEY_ESCAPE;

    // KEY_PLAY 特殊处理：切换自动阅读模式
    if(keyCode == EventKeyboard::KeyCode::KEY_PLAY) {
        iTJSDispatch2 *global = TVPGetScriptDispatch();
        tTJSVariant var;
        // 通过 TJS2 脚本接口调用 kag.enterAutoMode() 或 kag.cancelAutoMode()
        if(global->PropGet(0, TJS_W("kag"), nullptr, &var, global) == TJS_S_OK) {
            iTJSDispatch2 *kag = var.AsObjectNoAddRef();
            // 读取 kag.autoMode 属性判断当前状态
            if(kag->PropGet(0, TJS_W("autoMode"), nullptr, &var, kag) == TJS_S_OK) {
                if(var.operator bool()) {
                    // 已在自动模式 → 调用 cancelAutoMode()
                } else {
                    // 不在自动模式 → 调用 enterAutoMode()
                }
            }
        }
        return;
    }

    unsigned int code = TVPConvertKeyCodeToVKCode(keyCode);
    if(!code || code >= 0x200) return;
    code = _keymap[code];
    bool isPressed = _scancode[code] & 1;   // 检查 bit 0
    _scancode[code] &= 0x10;               // 清除 bit 0，保留 bit 4
    if(isPressed && _currentWindowLayer) {
        _currentWindowLayer->OnKeyUp(code, TVPGetCurrentShiftKeyState());
    }
}
```

> **KEY_PLAY 的妙用：** 这是 Android 媒体键映射。一些蓝牙遥控器或耳机线控的播放键会触发此键码。KrKr2 将其映射为"自动阅读模式切换"——一种无需触摸屏幕即可自动翻页的模式，对视觉小说游戏非常实用。

### 5.2 按键映射表

`_keymap[512]` 数组提供了一层可配置的按键重映射。默认初始化为恒等映射（`_keymap[i] = i`），用户可以在游戏配置中自定义：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 1970-1378 行（startupFrom 函数内）
// 初始化为恒等映射
for(int i = 0; i < std::size(_keymap); ++i) {
    _keymap[i] = i;
}
// 应用用户自定义映射
const auto &keymap = pGlobalCfgMgr->GetKeyMap();
for(const auto &it : keymap) {
    if(!it.second) continue;
    _keymap[it.first] = _keymap[it.second];
}
```

这意味着用户可以将任意按键映射到另一个按键。例如，将方向键映射为 WASD，或将手柄 A 键映射为回车键。映射发生在 VK 码层面，对游戏脚本完全透明。

### 5.3 手柄按键输入

手柄按键（如 A/B/X/Y、十字键等）的处理流水线与键盘几乎完全相同，只是使用了不同的键码转换函数：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 2573-2599 行
void TVPMainScene::onPadKeyDown(cocos2d::Controller *ctrl, int keyCode,
                                cocos2d::Event *e) {
    if(!UINode->getChildren().empty()) return;  // UI 层优先
    unsigned int code = TVPConvertPadKeyCodeToVKCode(keyCode);  // 手柄专用转换
    if(!code || code >= 0x200) return;
    code = _keymap[code];          // 同样经过按键映射
    _scancode[code] = 0x11;        // 同样的状态标志
    if(_currentWindowLayer) {
        _currentWindowLayer->InternalKeyDown(code, TVPGetCurrentShiftKeyState());
    }
}

void TVPMainScene::onPadKeyUp(cocos2d::Controller *ctrl, int keyCode,
                              cocos2d::Event *e) {
    unsigned int code = TVPConvertPadKeyCodeToVKCode(keyCode);
    if(!code || code >= 0x200) return;
    code = _keymap[code];
    bool isPressed = _scancode[code] & 1;
    _scancode[code] &= 0x10;
    if(isPressed && _currentWindowLayer) {
        _currentWindowLayer->OnKeyUp(code, TVPGetCurrentShiftKeyState());
    }
}
```

注意手柄按键和键盘共享相同的 `_keymap[]` 和 `_scancode[]` 数组。这意味着：
- 用户可以用同一套配置文件同时映射键盘和手柄
- 手柄和键盘按下同一个逻辑键时不会冲突（`_scancode` 正确追踪）

### 5.4 手柄摇杆输入

手柄摇杆（Axis）不触发键盘事件，而是直接控制虚拟鼠标光标的移动：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 2524-2571 行
void TVPMainScene::onAxisEvent(cocos2d::Controller *ctrl, int keyCode,
                               cocos2d::Event *e) {
    if(!_currentWindowLayer || !_currentWindowLayer->PrimaryLayerArea)
        return;
    if(!_virutalMouseMode || _windowMgrOverlay)
        return;  // 仅在虚拟鼠标模式下生效

    const float threashold = 0.1f;  // 死区阈值（10%）
    const cocos2d::Controller::KeyStatus &keyStatus =
        ctrl->getKeyStatus(keyCode);
    if(std::abs(keyStatus.value) < threashold) return;  // 死区内忽略

    // 将死区外的值线性映射到 0~1 范围
    float offv = keyStatus.value;
    if(offv > 0) offv = (offv - threashold) / (1 - threashold);
    else         offv = (offv + threashold) / (1 - threashold);

    // 获取当前鼠标位置，根据摇杆方向偏移
    Vec2 pt = Vec2(_currentWindowLayer->_LastMouseX,
                   _currentWindowLayer->_LastMouseY);
    switch(keyCode) {
        case cocos2d::Controller::JOYSTICK_LEFT_X:
            pt.x += offv * 16;  // 水平偏移 16 像素/帧
            break;
        case cocos2d::Controller::JOYSTICK_LEFT_Y:
            pt.y += offv * 16;  // 垂直偏移 16 像素/帧
            break;
        default: return;
    }

    // 坐标转换 + 边界限制 + 更新光标位置
    pt = _currentWindowLayer->PrimaryLayerArea->convertToWorldSpace(pt);
    pt = GameNode->convertToNodeSpace(pt);
    // ... 边界裁剪（确保光标不超出屏幕）...
    _mouseCursor->setPosition(newpt);
    _currentWindowLayer->onMouseMove(GameNode->convertToWorldSpace(newpt));
}
```

**关键设计点：**

1. **死区处理（0.1f）**：摇杆归中时不可能完美回到 0.0，所以需要一个阈值忽略微小偏移
2. **线性映射**：死区外的值被重新映射到 0~1 范围，确保从死区边缘开始就有平滑的移动
3. **固定速度（16 像素/帧）**：摇杆偏移量直接乘以 16，在 60fps 下约等于每秒 960 像素的最大移动速度
4. **仅限虚拟鼠标模式**：如果用户没有开启虚拟鼠标光标，摇杆输入被忽略

### 5.5 IME 文本输入

对于需要文字输入的场景（如游戏内聊天、存档命名），KrKr2 实现了 Cocos2d-x 的 `IMEDelegate` 接口：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 2458-2522 行
bool TVPMainScene::attachWithIME() {
    bool ret = IMEDelegate::attachWithIME();
    if(ret) {
        auto pGlView = Director::getInstance()->getOpenGLView();
        pGlView->setIMEKeyboardState(true);  // 弹出软键盘
    }
    return ret;
}

void TVPMainScene::insertText(const char *text, size_t len) {
    std::string utf8(text, len);
    onTextInput(utf8);
}

void TVPMainScene::onTextInput(const std::string &text) {
    std::u16string buf;
    if(StringUtils::UTF8ToUTF16(text, buf)) {
        for(int i = 0; i < buf.size(); ++i) {
            // 逐字符发送给当前窗口
            _currentWindowLayer->OnKeyPress(buf[i], 0, false, false);
        }
    }
}

void TVPMainScene::deleteBackward() {
#ifndef _WIN32
    // 非 Windows 平台：退格键通过 IME 回调处理
    if(_currentWindowLayer) {
        _currentWindowLayer->InternalKeyDown(VK_BACK, TVPGetCurrentShiftKeyState());
        _currentWindowLayer->OnKeyUp(VK_BACK, TVPGetCurrentShiftKeyState());
    }
#endif
}
```

**IME 流程：**
1. 游戏脚本请求文字输入 → `attachWithIME()` → 弹出系统软键盘
2. 用户输入文字 → `insertText()` → UTF-8 转 UTF-16 → 逐字符 `OnKeyPress()`
3. 用户按退格 → `deleteBackward()` → 模拟 `VK_BACK` 按键事件
4. 输入完成 → `detachWithIME()` → 收起软键盘

> **`#ifndef _WIN32` 的原因：** Windows 上退格键已经通过 `onKeyPressed` 正常路径处理了，不需要 IME 回调重复发送。而 Android/Linux/macOS 的软键盘退格通过 IME 回调触发。

---

## 六、虚拟鼠标光标系统

触摸屏设备缺少鼠标光标，但很多 KiriKiri 游戏的 UI 设计依赖"鼠标悬停"（hover）效果。KrKr2 为此实现了一套虚拟鼠标光标系统：用户触摸屏幕时不是直接点击，而是移动一个可见的光标精灵，光标到达目标位置后再"点击"。

### 6.1 光标创建与显示

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 2295-2327 行
Sprite *TVPCreateCUR() {
    // 从内置资源加载 default.cur 文件（Windows 标准光标格式）
    std::string fullPath =
        FileUtils::getInstance()->fullPathForFilename("default.cur");
    Data buf = FileUtils::getInstance()->getDataFromFile(fullPath);
    tTVPMemoryStream stream(buf.getBytes(), buf.getSize());
    return TVPLoadCursorCUR(&stream);  // 解析 .cur 格式为 Cocos2d-x Sprite
}

void TVPMainScene::showVirtualMouseCursor(bool bVisible) {
    if(!bVisible) {
        if(_mouseCursor) _mouseCursor->setVisible(false);
        _virutalMouseMode = bVisible;
        return;
    }
    if(!_mouseCursor) {
        _mouseCursor = TVPCreateCUR();
        if(!_mouseCursor) return;

        // 从配置读取光标大小，默认 0.5（对应 1x 缩放）
        _mouseCursorScale =
            _mouseCursor->getScale() *
            convertCursorScale(
                IndividualConfigManager::GetInstance()->GetValue<float>(
                    "vcursor_scale", 0.5f));
        _mouseCursor->setScale(_mouseCursorScale);
        // 光标添加到 GameNode 层，与游戏窗口同一渲染空间
        GameNode->addChild(_mouseCursor, GAME_WINMGR_ORDER);
        _mouseCursor->setPosition(GameNode->getContentSize() / 2);  // 初始居中
    }
    _mouseCursor->setVisible(true);
    _virutalMouseMode = bVisible;
}
```

### 6.2 光标缩放曲线

`convertCursorScale` 使用分段线性函数将用户配置值（0~1）转换为实际缩放倍数：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 2604-2610 行
float TVPMainScene::convertCursorScale(float val /*0 ~ 1*/) {
    if(val <= 0.5f) {
        return 0.25f + (val * 2) * 0.75f;  // 0→0.25x, 0.5→1.0x
    } else {
        return 1.f + (val - 0.5f) * 2.f;   // 0.5→1.0x, 1.0→2.0x（注意代码中是3x）
    }
}
```

映射关系图：

```
缩放倍数
3.0x ┤                                    ╱
     │                                ╱╱
2.0x ┤                            ╱╱
     │                        ╱╱
1.0x ┤────────────────╱╱╱╱╱╱
     │            ╱╱
0.25x┤────╱╱╱╱╱╱
     └──────────────┬───────────────────
    0.0           0.5                1.0
                 配置值 (vcursor_scale)
```

这种非线性映射的好处：配置值在 0~0.5 区间时缩放变化缓慢（0.25x~1.0x），适合精细调节；在 0.5~1.0 区间时变化加快（1.0x~3.0x），适合需要大光标的用户。

### 6.3 虚拟鼠标模式下的触摸处理

在虚拟鼠标模式下，`TVPMainScene` 接管触摸事件（不再直接交给 `TVPWindowLayer`）：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 2333-2375 行
bool TVPMainScene::onTouchBegan(cocos2d::Touch *touch, cocos2d::Event *event) {
    if(UINode->getChildrenCount()) return false;  // UI 优先
    if(!_currentWindowLayer) return false;
    if(!_virutalMouseMode || _windowMgrOverlay)
        return _currentWindowLayer->onTouchBegan(touch, event);  // 非虚拟鼠标模式走正常流程

    // ========= 虚拟鼠标模式 =========
    _mouseTouches.insert(touch);
    switch(_mouseTouches.size()) {
        case 1:  // 单指 → 左键
            _mouseTouchPoint = GameNode->convertToNodeSpace(touch->getLocation());
            _mouseBeginPoint = _mouseCursor->getPosition();  // 记录光标当前位置
            _touchBeginTick = TVPGetRoughTickCount32();
            _mouseMoved = false;
            _mouseClickedDown = false;
            _mouseBtn = ::mbLeft;
            _mouseCursor->stopAllActions();
            _mouseCursor->setOpacity(255);     // 恢复不透明
            _mouseCursor->setScale(_mouseCursorScale);

            // 启动 1 秒定时器：按住不动超过 1 秒 → 自动点击
            _mouseCursor->runAction(Sequence::createWithTwoActions(
                DelayTime::create(1),
                CallFuncN::create([this](Node *p) {
                    p->setScale(_mouseCursorScale * 0.8f);  // 缩小光标作为视觉反馈
                    _currentWindowLayer->onMouseMove(
                        GameNode->convertToWorldSpace(_mouseBeginPoint));
                    _currentWindowLayer->onMouseDown(
                        GameNode->convertToWorldSpace(_mouseBeginPoint));
                    _mouseClickedDown = true;
                    _mouseMoved = true;
                })));
            break;
        case 2:  // 双指 → 右键
            _mouseBtn = ::mbRight;
            // 两指中点作为触摸参考点
            _mouseTouchPoint = (_mouseTouchPoint +
                GameNode->convertToNodeSpace(touch->getLocation())) / 2;
            break;
    }
    return true;
}
```

### 6.4 虚拟鼠标的拖动与点击判定

触摸移动时，光标跟随手指位移（不是直接跳到手指位置，而是相对移动）：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 2377-2420 行
void TVPMainScene::onTouchMoved(cocos2d::Touch *touch, cocos2d::Event *event) {
    if(!_virutalMouseMode || _windowMgrOverlay)
        return _currentWindowLayer->onTouchMoved(touch, event);

    if(_mouseTouches.size()) {
        Vec2 pt;
        if(_mouseTouches.size() == 1) {
            pt = touch->getLocation();
        } else if(_mouseTouches.size() == 2) {
            // 双指时取中点
            auto it = _mouseTouches.begin();
            pt = (*it)->getLocation();
            pt = (pt + (*++it)->getLocation()) / 2;
        }
        // 计算相对位移
        Vec2 newPoint = GameNode->convertToNodeSpace(pt);
        Vec2 moveDistance = newPoint - _mouseTouchPoint;
        // 光标新位置 = 初始位置 + 相对位移
        pt = _mouseBeginPoint + moveDistance;

        // 边界裁剪：确保光标不超出 GameNode 范围
        cocos2d::Size size = GameNode->getContentSize();
        Vec2 newpt = pt;
        if(pt.x < 0) newpt.x = 0;
        else if(pt.x > size.width) newpt.x = size.width;
        if(pt.y < 0) newpt.y = 0;
        else if(pt.y > size.height) newpt.y = size.height;
        _mouseBeginPoint += newpt - pt;  // 补偿边界裁剪的偏移

        _mouseCursor->setPosition(newpt);
        _currentWindowLayer->onMouseMove(GameNode->convertToWorldSpace(newpt));

        if(!_mouseMoved) {
            if(TVPGetRoughTickCount32() - _touchBeginTick > 1000) {
                // 按住超过 1 秒 → 自动开始拖拽
                _currentWindowLayer->onMouseDown(
                    GameNode->convertToWorldSpace(newpt));
                _mouseClickedDown = true;
                _mouseMoved = true;
            } else if(moveDistance.getLengthSq() > _touchMoveThresholdSq) {
                // 移动超过阈值 → 取消按住定时器，标记为纯移动（不点击）
                _mouseCursor->stopAllActions();
                _mouseMoved = true;
            }
        }
    }
}
```

**相对移动 vs 绝对定位：** 虚拟鼠标使用相对移动模式——手指在屏幕上移动多少，光标就移动多少。这比绝对定位（光标直接跳到手指位置）更精确，因为用户可以在大屏幕上通过"抬起-放下-继续滑动"来精确定位到小目标。

触摸结束时的点击判定：

```cpp
// 文件：cpp/core/environ/cocos2d/MainScene.cpp 第 2422-2445 行
void TVPMainScene::onTouchEnded(cocos2d::Touch *touch, cocos2d::Event *event) {
    if(!_virutalMouseMode || _windowMgrOverlay)
        return _currentWindowLayer->onTouchEnded(touch, event);

    if(_mouseTouches.size() == 1) {
        Vec2 pt = _mouseCursor->getPosition();
        if(!_mouseClickedDown &&
           TVPGetRoughTickCount32() - _touchBeginTick < 150) {
            // 快速点触（< 150ms）→ 在光标位置模拟完整点击
            _currentWindowLayer->onMouseClick(
                GameNode->convertToWorldSpace(pt));
        } else if(_mouseClickedDown) {
            // 之前已经 MouseDown → 发送 MouseUp 完成拖拽
            _currentWindowLayer->onMouseUp(
                GameNode->convertToWorldSpace(pt));
        }
        // 其他情况（移动了但没有 clickDown）→ 什么都不做，只是移动了光标
    }
    _mouseTouches.erase(touch);
    if(_mouseTouches.size() == 0) {
        _mouseMoved = false;
        _mouseClickedDown = false;
        _refadeMouseCursor();           // 渐隐光标（非操作时半透明）
        _mouseCursor->setScale(_mouseCursorScale);  // 恢复原始大小
        _mouseCursor->stopAllActions();
    }
}
```

虚拟鼠标的完整交互状态机：

```
       ┌──────────────────────────────────────────────────┐
       │              虚拟鼠标触摸状态机                   │
       └──────────────────────────────────────────────────┘

  touchBegan ───→ 等待中（_mouseMoved=false, _mouseClickedDown=false）
                     │
         ┌───────────┼────────────────────┐
         │           │                    │
    < 150ms 抬手   1秒定时器触发      移动超过阈值
         │           │                    │
         ↓           ↓                    ↓
    快速点击      自动点击开始         纯移动模式
    onMouseClick  onMouseDown          （仅移动光标）
                  光标缩小0.8x          stopAllActions
                     │                    │
                  继续拖动              抬手
                  onMouseMove           → 无事件
                     │
                  抬手
                  onMouseUp
```

---

## 常见错误及解决方案

### 错误 1：坐标系混淆导致点击位置偏移

**现象：** 触摸屏幕某个位置，但游戏响应的却是镜像位置（上下颠倒）。

**原因：** Cocos2d-x 使用左下角为原点（Y 轴向上），而 KiriKiri 使用左上角为原点（Y 轴向下）。如果忘记了坐标转换，触摸位置的 Y 值就是反的。

**解决：** 始终使用 `contentSize.height - cocos_y` 进行 Y 轴翻转：

```cpp
// 正确写法
Vec2 nsp = PrimaryLayerArea->convertToNodeSpace(touchLocation);
_LastMouseX = nsp.x;
_LastMouseY = PrimaryLayerArea->getContentSize().height - nsp.y;  // Y 轴翻转

// 错误写法（忘记翻转）
_LastMouseY = nsp.y;  // 这会导致上下颠倒
```

### 错误 2：模态窗口中 UI 无响应

**现象：** 调用 `ShowWindowAsModal()` 后，游戏显示正常但触摸/键盘完全无反应。

**原因：** 模态窗口内部使用 `while` 循环手动驱动渲染（`TVPDrawSceneOnce`），但忘记调用 `TVPProcessInputEvents()` 处理输入事件队列。

**解决：** 确保模态循环中同时处理渲染和输入：

```cpp
while(!modal_result_) {
    int remain = TVPDrawSceneOnce(30);  // 渲染
    TVPProcessInputEvents();             // 处理输入 ← 不能省略
    if(remain > 0) {
        std::this_thread::sleep_for(std::chrono::milliseconds(remain));
    }
}
```

### 错误 3：UI 表单动画残留导致触摸穿透

**现象：** 关闭 UI 表单后，原来表单所在区域无法点击游戏。

**原因：** `popUIForm` 使用延迟动画移除节点（`DelayTime + CallFunc + removeFromParent`）。在动画播放期间，节点仍然存在于 `UINode` 中，`UINode->getChildrenCount()` 返回非零值，导致触摸事件被拦截。

**解决：** 这是设计行为，不需要"修复"。如果确实需要在动画期间允许触摸穿透，需要在 `popUIForm` 中立即更改节点的事件拦截行为，但这可能导致用户在过渡期间误触游戏。

---

## 动手实践

### 实践 1：追踪触摸事件流

在 `MainScene.cpp` 中添加临时日志，观察不同触摸操作产生的事件序列：

```cpp
// 在 TVPWindowLayer::onTouchBegan 的开头添加：
CCLOG("TouchBegan: touches=%zu, btn=%d, point=(%f,%f)",
      _touches.size(), (int)_mouseBtn,
      touch->getLocation().x, touch->getLocation().y);

// 在 TVPWindowLayer::onTouchMoved 中拖拽判定处添加：
CCLOG("TouchMoved: moved=%s, distance=%f, elapsed=%u",
      _touchMoved ? "true" : "false",
      _touchPoint.getDistanceSq(touch->getLocation()),
      TVPGetRoughTickCount32() - _touchBeginTick);

// 在 TVPWindowLayer::onTouchEnded 中添加：
CCLOG("TouchEnded: wasMoved=%s, btn=%d", _touchMoved ? "true" : "false", (int)_mouseBtn);
```

编译运行后，打开游戏并执行以下操作，观察控制台输出：
1. 单指快速点击 → 应看到 `TouchBegan(touches=1)` → `TouchEnded(wasMoved=false)`
2. 单指长按后拖动 → 应看到 `TouchBegan` → 多个 `TouchMoved(moved=true)` → `TouchEnded(wasMoved=true)`
3. 双指点击 → 应看到 `TouchBegan(touches=1)` → `TouchBegan(touches=2, btn=1)`（mbRight=1）

### 实践 2：修改虚拟鼠标光标缩放曲线

尝试修改 `convertCursorScale` 函数，使光标大小变化更敏感：

```cpp
float TVPMainScene::convertCursorScale(float val) {
    // 原始曲线：0→0.25x, 0.5→1.0x, 1.0→2.0x
    // 修改为：0→0.5x, 0.5→1.0x, 1.0→4.0x（更大范围）
    if(val <= 0.5f) {
        return 0.5f + (val * 2) * 0.5f;
    } else {
        return 1.f + (val - 0.5f) * 6.f;  // 最大 4.0x
    }
}
```

编译后在虚拟鼠标模式下测试，通过设置中的 `vcursor_scale` 滑块调节，对比修改前后的手感差异。

---

## 对照项目源码

| 源文件路径 | 行号范围 | 本节涉及内容 |
|-----------|---------|-------------|
| `cpp/core/environ/cocos2d/AppDelegate.cpp` | 1-120 | Section 一：引擎初始化、分辨率策略 |
| `cpp/core/environ/cocos2d/AppDelegate.h` | 1-26 | Section 一：AppDelegate 类定义 |
| `cpp/core/environ/cocos2d/MainScene.h` | 1-94 | 全文：TVPMainScene 类定义、枚举、接口声明 |
| `cpp/core/environ/cocos2d/MainScene.cpp` | 39-45 | Section 二：SCENE_ORDER 枚举 |
| `cpp/core/environ/cocos2d/MainScene.cpp` | 365-1610 | Section 三：TVPWindowLayer（触摸映射、模态窗口、多窗口管理） |
| `cpp/core/environ/cocos2d/MainScene.cpp` | 1726-1751 | Section 四：MaskLayer 实现 |
| `cpp/core/environ/cocos2d/MainScene.cpp` | 1827-1924 | Section 四：pushUIForm/popUIForm |
| `cpp/core/environ/cocos2d/MainScene.cpp` | 2130-2248 | Section 五：键盘输入处理 |
| `cpp/core/environ/cocos2d/MainScene.cpp` | 2291-2456 | Section 六：虚拟鼠标光标系统 |
| `cpp/core/environ/cocos2d/MainScene.cpp` | 2524-2602 | Section 五：手柄/摇杆输入 |
| `cpp/core/environ/cocos2d/MainScene.cpp` | 2604-2610 | Section 六：convertCursorScale |
| `cpp/core/environ/ui/BaseForm.h` | 1-216 | Section 四：iTVPBaseForm、NodeMap、TTouchEventRouter |


---

## 本节小结

本节深入剖析了 KrKr2 如何通过 Cocos2d-x 实现完整的桌面视觉小说引擎运行环境，核心要点如下：

1. **TVPAppDelegate** 是引擎启动入口 —— 负责 OpenGL 上下文初始化、分辨率策略配置（`ResolutionPolicy::SHOW_ALL` 保持画面不变形）、以及启动 KiriKiri 核心引擎线程
2. **TVPMainScene** 采用双节点架构 —— `_gameNode`（Z=0）承载游戏画面（TVPWindowLayer），`_uiNode`（Z=1）承载 UI 表单层；通过 SCENE_ORDER 枚举严格管理渲染顺序
3. **TVPWindowLayer** 实现了触摸→鼠标事件的完整映射 —— 单指=左键，双指=右键，长按拖动=鼠标移动；同时支持多窗口管理和模态窗口锁定
4. **UI 表单栈系统** 使用 `pushUIForm`/`popUIForm` 管理界面层级 —— MaskLayer 实现模态遮罩与事件隔离，`iTVPBaseForm` 定义了统一的表单接口
5. **输入系统全覆盖** —— 键盘（Android 虚拟键 → KiriKiri VK 码映射）、手柄（十字键 → 方向键、肩键 → PageUp/Down）、IME（Cocos2d-x TextField → KiriKiri 输入法接口）
6. **虚拟鼠标光标系统** 是触屏设备的核心创新 —— 通过精灵节点模拟桌面光标，支持非线性缩放曲线（`convertCursorScale`）、拖放操作、以及精确的触摸偏移计算

> **关键设计理念**：桥接层的目标是让上层 KiriKiri 引擎代码"感觉"自己运行在传统 Win32 环境中 —— 所有 Cocos2d-x 特有的坐标系差异（左下角 vs 左上角）、事件模型差异（多点触摸 vs 单点鼠标）都在桥接层内部消化，不泄漏到引擎核心。

---

## 练习题与答案

### 题目 1：坐标系转换

KrKr2 项目中，Cocos2d-x 使用左下角为原点（Y 轴向上），而 KiriKiri 引擎使用左上角为原点（Y 轴向下）。请写出一个完整的坐标转换函数，将 Cocos2d-x 的触摸坐标转换为 KiriKiri 引擎坐标。假设画面内容区域高度为 `contentHeight`。

<details>
<summary>查看答案</summary>

```cpp
#include <cstdio>

struct Point {
    float x;
    float y;
};

// 将 Cocos2d-x 坐标（左下角原点，Y 向上）
// 转换为 KiriKiri 坐标（左上角原点，Y 向下）
Point cocosToKiriKiri(float cocosX, float cocosY,
                       float contentHeight) {
    Point result;
    result.x = cocosX;  // X 轴方向一致，无需转换
    result.y = contentHeight - cocosY;  // Y 轴翻转
    return result;
}

int main() {
    float contentHeight = 720.0f;  // 假设内容高度 720px

    // 测试用例 1：Cocos 左下角 (0, 0) -> KiriKiri 左上角 (0, 720)
    Point p1 = cocosToKiriKiri(0, 0, contentHeight);
    printf("Cocos(0, 0) -> KiriKiri(%.0f, %.0f)\n",
           p1.x, p1.y);
    // 输出：Cocos(0, 0) -> KiriKiri(0, 720)

    // 测试用例 2：Cocos 中心 (640, 360) -> KiriKiri (640, 360)
    Point p2 = cocosToKiriKiri(640, 360, contentHeight);
    printf("Cocos(640, 360) -> KiriKiri(%.0f, %.0f)\n",
           p2.x, p2.y);
    // 输出：Cocos(640, 360) -> KiriKiri(640, 360)

    // 测试用例 3：Cocos 右上角 (1280, 720) -> KiriKiri (1280, 0)
    Point p3 = cocosToKiriKiri(1280, 720, contentHeight);
    printf("Cocos(1280, 720) -> KiriKiri(%.0f, %.0f)\n",
           p3.x, p3.y);
    // 输出：Cocos(1280, 720) -> KiriKiri(1280, 0)

    return 0;
}
```

**关键要点**：
- X 坐标不变（两个坐标系的 X 轴方向一致）
- Y 坐标需要用内容高度减去原始 Y 值来翻转
- 在实际项目代码中，这个转换发生在 `TVPWindowLayer::onTouchBegan` 等触摸回调中，使用 `_node->getContentSize().height - location.y` 完成

</details>

### 题目 2：多点触摸事件映射

请解释 KrKr2 中如何将移动端多点触摸映射为桌面端鼠标事件，并回答以下问题：

1. 为什么单指触摸映射为鼠标左键（`mbLeft`），而双指触摸映射为鼠标右键（`mbRight`）？
2. 如果用户先按下第一根手指，再按下第二根手指，然后抬起第二根手指，最后抬起第一根手指 —— 请按顺序列出产生的所有 KiriKiri 鼠标事件。
3. 在这个过程中，`_touchCount` 字段是如何变化的？

<details>
<summary>查看答案</summary>

**问题 1 解答**：

视觉小说引擎中，左键是主要交互按键（推进对话、选择选项），右键通常打开菜单或回退。在触屏设备上：
- **单指点击**是最自然的操作，对应最常用的左键
- **双指点击**是次自然的手势，对应次常用的右键

这种映射符合用户直觉：单指 = 确认/前进，双指 = 菜单/后退。

**问题 2 解答**：

按时间顺序产生的事件：

```
时刻 1: 第一根手指按下
  -> OnTouchDown(x, y, mbLeft, ...)     // 左键按下

时刻 2: 第二根手指按下（touches.size() >= 2）
  -> OnTouchDown(x, y, mbRight, ...)    // 右键按下（btn=1）

时刻 3: 第二根手指抬起
  -> 检测到 touches 减少但 _touchCount > 1
  -> 根据实现，可能产生 OnTouchUp 对应右键

时刻 4: 第一根手指抬起
  -> OnTouchUp(x, y, mbLeft, ...)       // 左键释放
  -> 如果手指未移动过（wasMoved=false），
     额外产生 OnClick(x, y, mbLeft)
```

**问题 3 解答**：

`_touchCount` 的变化过程：

```
初始状态:  _touchCount = 0
手指 1 按下: _touchCount = 1   // onTouchBegan 中 ++
手指 2 按下: _touchCount = 2   // onTouchBegan 中 ++
手指 2 抬起: _touchCount = 1   // onTouchEnded 中 --
手指 1 抬起: _touchCount = 0   // onTouchEnded 中 --
```

参考源码：`MainScene.cpp` 第 365-1610 行（TVPWindowLayer 的触摸处理实现）。

</details>

### 题目 3：虚拟鼠标光标的非线性缩放

项目中 `convertCursorScale` 函数使用分段线性函数实现非线性缩放：

```cpp
float TVPMainScene::convertCursorScale(float val) {
    if(val <= 0.5f) {
        return 0.25f + val * 1.5f;  // [0, 0.5] -> [0.25, 1.0]
    }
    return 1.f + (val - 0.5f) * 2.f;  // (0.5, 1.0] -> (1.0, 2.0]
}
```

请回答：
1. 为什么不使用简单的线性映射 `return val * 2.0f`？
2. 如果要将缩放范围改为 [0.5, 3.0]，同时保持中点（val=0.5）映射到 1.0，请写出修改后的函数。

<details>
<summary>查看答案</summary>

**问题 1 解答**：

简单线性映射 `val * 2.0f` 会产生以下问题：

| 输入值 | 线性映射 | 非线性映射 |
|--------|----------|-----------|
| 0.0 | 0.0x（不可见！） | 0.25x（仍可见） |
| 0.25 | 0.5x | 0.625x |
| 0.5 | 1.0x | 1.0x |
| 0.75 | 1.5x | 1.5x |
| 1.0 | 2.0x | 2.0x |

关键差异在 **小值区间**：线性映射在 `val=0` 时光标缩放为 0（完全不可见），这对用户体验是灾难性的。非线性映射确保即使在最小设置下，光标仍有 0.25 倍大小可见。

此外，人眼对大小变化的感知是非线性的 —— 从 0.25x 到 1.0x 的变化在视觉上比 1.0x 到 2.0x 更显著，分段线性函数通过在小值区间使用较缓的斜率来补偿这种感知差异。

**问题 2 解答**：

```cpp
float TVPMainScene::convertCursorScale(float val) {
    // 目标：[0, 0.5] -> [0.5, 1.0], (0.5, 1.0] -> (1.0, 3.0]
    // 中点 val=0.5 映射到 1.0（不变）
    if(val <= 0.5f) {
        // 斜率 = (1.0 - 0.5) / (0.5 - 0) = 1.0
        return 0.5f + val * 1.0f;
    }
    // 斜率 = (3.0 - 1.0) / (1.0 - 0.5) = 4.0
    return 1.f + (val - 0.5f) * 4.f;
}

// 验证：
// val=0.0 -> 0.5 + 0 = 0.5   (最小 0.5x)
// val=0.5 -> 0.5 + 0.5 = 1.0 (中点 1.0x)
// val=1.0 -> 1.0 + 0.5*4 = 3.0 (最大 3.0x)
```

参考源码：`MainScene.cpp` 第 2604-2610 行。

</details>

---

## 下一步

下一节 [05-配置管理系统](05-配置管理系统.md) 将分析 KrKr2 的三级配置架构（全局配置、个别游戏配置、本地化配置），了解 `ConfigManager` 如何管理跨平台的设置持久化与运行时读写。
