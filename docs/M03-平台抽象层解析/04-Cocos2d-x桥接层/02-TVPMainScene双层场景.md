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

