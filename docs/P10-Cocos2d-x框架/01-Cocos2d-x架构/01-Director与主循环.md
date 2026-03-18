# Director 与主循环

> **所属模块：** P10-Cocos2d-x 框架
> **前置知识：** [P04-OpenGL 图形编程](../../P04-OpenGL图形编程/README.md)、C++ 基础（单例模式、虚函数）
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 Cocos2d-x 的 Director 单例模式及其职责
2. 描述主循环（Main Loop）的完整执行流程
3. 使用帧调度器（Scheduler）注册定时回调
4. 理解 deltaTime 的含义与帧率控制机制
5. 对照 KrKr2 源码理解 Director 在实际项目中的用法

## Director：引擎的总导演

### 什么是 Director

Director（导演）是 Cocos2d-x 的**核心单例对象**，负责管理整个游戏的生命周期。它的职责包括：

- **场景管理**：切换、压栈、弹出场景
- **主循环驱动**：每一帧调用 `drawScene()` 推进游戏逻辑和渲染
- **OpenGL 上下文管理**：持有 GLView，控制视口和投影矩阵
- **帧率控制**：设置目标 FPS，计算 deltaTime
- **全局状态**：暂停/恢复、获取窗口尺寸、坐标转换

Director 在英文文献中也叫 "Game Loop Controller"，其设计灵感来自电影导演——场景（Scene）是一幕戏，导演决定何时开拍、何时切换。

### 获取 Director 实例

Director 是一个典型的**单例模式**（Singleton Pattern）实现：

```cpp
#include "cocos2d.h"

// 获取 Director 单例 —— 整个应用生命周期内只有一个实例
auto director = cocos2d::Director::getInstance();

// 获取当前运行的场景
auto currentScene = director->getRunningScene();

// 获取 OpenGL 视图
auto glView = director->getOpenGLView();

// 获取可视区域大小（设计分辨率）
auto visibleSize = director->getVisibleSize();

// 获取可视区域原点（多分辨率适配时可能不为 (0,0)）
auto origin = director->getVisibleOrigin();
```

> **为什么用单例？** 整个应用只需要一个主循环驱动器。多个 Director 会导致渲染冲突、帧调度混乱。单例保证全局唯一性，任何地方都能通过 `Director::getInstance()` 访问。

### Director 的核心属性

```cpp
// Director 内部关键成员（简化展示）
class CC_DLL Director : public Ref {
protected:
    Scheduler*    _scheduler;       // 帧调度器
    ActionManager* _actionManager;  // 动作管理器
    EventDispatcher* _eventDispatcher; // 事件分发器

    Scene*  _runningScene;          // 当前运行的场景
    Scene*  _nextScene;             // 下一帧要切换的场景
    Vector<Scene*> _scenesStack;    // 场景栈（pushScene/popScene）

    GLView* _openGLView;            // OpenGL 视图
    Renderer* _renderer;            // 渲染器

    float   _deltaTime;             // 上一帧到当前帧的时间间隔（秒）
    float   _animationInterval;     // 目标帧间隔（1/FPS）
    float   _frameRate;             // 实际帧率

    bool    _paused;                // 是否暂停
    bool    _invalid;               // 是否需要退出
    unsigned int _totalFrames;      // 总帧数
};
```

这些成员覆盖了引擎运行的所有核心状态。特别注意 `_scheduler`、`_actionManager`、`_eventDispatcher` 三大子系统——它们在每一帧都会被驱动。

## 主循环（Main Loop）详解

### 主循环流程

Cocos2d-x 的主循环是一个经典的**游戏循环**（Game Loop），每帧按固定顺序执行以下步骤：

```
┌─────────────────────────────────────────────┐
│                  主循环开始                    │
│                                              │
│  1. 计算 deltaTime（距上一帧的时间差）         │
│                                              │
│  2. 处理场景切换（如果 _nextScene != nullptr） │
│                                              │
│  3. Scheduler::update(dt)                    │
│     ├── 系统调度器更新                        │
│     ├── 用户注册的 schedule 回调              │
│     └── ActionManager::update(dt)            │
│                                              │
│  4. EventDispatcher 分发事件                  │
│                                              │
│  5. Scene::render() → 遍历节点树渲染          │
│     ├── visit() 递归遍历子节点                │
│     ├── draw() 生成渲染命令                   │
│     └── Renderer::render() 执行 GL 调用      │
│                                              │
│  6. 交换缓冲区（SwapBuffers）                 │
│                                              │
│  7. 清理 autorelease 池                      │
│                                              │
│  8. 帧率控制（sleep 到目标帧间隔）             │
│                                              │
└──────────────────── 循环 ────────────────────┘
```

### Director::mainLoop() 源码分析

Cocos2d-x 3.x 中 `Director::mainLoop()` 的核心实现如下：

```cpp
// cocos2d-x/cocos/base/CCDirector.cpp（简化）
void Director::mainLoop() {
    if (_purgeDirectorInNextLoop) {
        // 清理并退出
        _purgeDirectorInNextLoop = false;
        purgeDirector();
        return;
    }
    if (_restartDirectorInNextLoop) {
        _restartDirectorInNextLoop = false;
        restartDirector();
        return;
    }

    // 核心：绘制场景（包含逻辑更新和渲染）
    drawScene();

    // 清理本帧的 autorelease 对象
    PoolManager::getInstance()->getCurrentPool()->clear();
}
```

`drawScene()` 是真正的重头戏：

```cpp
void Director::drawScene() {
    // 1. 计算 deltaTime
    calculateDeltaTime();

    // 2. 如果没暂停，更新调度器
    if (!_paused) {
        _scheduler->update(_deltaTime);
        _eventDispatcher->dispatchEvent(_eventAfterUpdate);
    }

    // 3. 清除 GL 缓冲区
    _renderer->clear();

    // 4. 处理场景切换
    if (_nextScene) {
        setNextScene();
    }

    // 5. 渲染当前场景
    if (_runningScene) {
        _runningScene->render(_renderer, Mat4::IDENTITY, nullptr);
        _eventDispatcher->dispatchEvent(_eventAfterVisit);
    }

    // 6. 执行渲染命令
    _renderer->render();
    _eventDispatcher->dispatchEvent(_eventAfterDraw);

    // 7. 交换缓冲区
    if (_openGLView) {
        _openGLView->swapBuffers();
    }

    // 8. 帧率统计
    _totalFrames++;
    calculateFrameRate();
}
```

### deltaTime：帧间时间差

`deltaTime`（通常缩写为 `dt`）是**上一帧到当前帧的实际经过时间**，单位为秒：

```cpp
void Director::calculateDeltaTime() {
    auto now = std::chrono::steady_clock::now();

    // 计算与上一帧的时间差（秒）
    _deltaTime = std::chrono::duration_cast<
        std::chrono::microseconds>(now - _lastUpdate).count()
        / 1000000.0f;

    // 防止异常值（如调试暂停后恢复）
    _deltaTime = MAX(0, _deltaTime);
    if (_deltaTime > 0.2f) {
        _deltaTime = 1.0f / 60.0f;  // 钳制到最大 200ms
    }

    _lastUpdate = now;
}
```

> **为什么需要 deltaTime？** 不同设备的帧率不同（60fps 的设备每帧 16.67ms，30fps 的设备每帧 33.33ms）。如果用固定值移动物体，低帧率设备上物体会变慢。乘以 `dt` 可以保证**移动速度与帧率无关**（帧率无关运动）。

```cpp
// 错误：速度依赖帧率
void update(float dt) {
    sprite->setPositionX(sprite->getPositionX() + 5.0f);
    // 60fps → 300像素/秒，30fps → 150像素/秒
}

// 正确：使用 dt 实现帧率无关运动
void update(float dt) {
    float speed = 300.0f; // 像素/秒
    sprite->setPositionX(sprite->getPositionX() + speed * dt);
    // 无论帧率多少，都是 300像素/秒
}
```

## 帧调度器（Scheduler）

### Scheduler 的作用

Scheduler 是 Cocos2d-x 的**定时回调管理器**，负责在每一帧（或按指定间隔）调用注册的回调函数。它是实现游戏逻辑更新的主要方式。

### 注册定时回调的三种方式

#### 方式一：scheduleUpdate（每帧调用）

```cpp
#include "cocos2d.h"

// 自定义场景类
class GameScene : public cocos2d::Scene {
public:
    static GameScene* create() {
        auto scene = new(std::nothrow) GameScene();
        if (scene && scene->init()) {
            scene->autorelease();
            return scene;
        }
        CC_SAFE_DELETE(scene);
        return nullptr;
    }

    bool init() override {
        if (!Scene::init()) return false;

        // 注册每帧更新回调 —— 每帧调用 update(dt)
        scheduleUpdate();
        return true;
    }

    // 每帧回调函数（必须叫 update，参数是 deltaTime）
    void update(float dt) override {
        // dt 是距上一帧的时间（秒），60fps 时约 0.0167
        CCLOG("帧更新: dt = %.4f 秒", dt);
    }
};
```

#### 方式二：schedule（自定义间隔）

```cpp
bool GameScene::init() {
    if (!Scene::init()) return false;

    // 每 0.5 秒调用一次 timerCallback
    schedule(
        CC_SCHEDULE_SELECTOR(GameScene::timerCallback),
        0.5f  // 间隔（秒），0 表示每帧
    );

    // C++11 lambda 写法（推荐）
    schedule([this](float dt) {
        CCLOG("Lambda 定时器: dt = %.4f", dt);
    }, 1.0f, "my_timer_key"); // 每 1 秒调用，key 用于后续取消

    return true;
}

void GameScene::timerCallback(float dt) {
    CCLOG("定时回调: 每 0.5 秒触发一次");
}
```

#### 方式三：scheduleOnce（延迟执行一次）

```cpp
bool GameScene::init() {
    if (!Scene::init()) return false;

    // 3 秒后执行一次，然后自动取消
    scheduleOnce([](float dt) {
        CCLOG("3 秒后执行：游戏开始！");
    }, 3.0f, "start_game");

    return true;
}
```

### 取消定时回调

```cpp
// 取消 scheduleUpdate 注册的 update()
unscheduleUpdate();

// 取消特定的 schedule 回调
unschedule(CC_SCHEDULE_SELECTOR(GameScene::timerCallback));

// 按 key 取消 lambda 回调
unschedule("my_timer_key");

// 取消当前节点的所有定时回调
unscheduleAllCallbacks();
```

### Scheduler 优先级

Scheduler 按**优先级**（priority）排序执行回调：

```cpp
// 数字越小，优先级越高，越先执行
// 系统默认优先级：
//   Scheduler::PRIORITY_SYSTEM = INT_MIN  （最高，引擎内部用）
//   Scheduler::PRIORITY_NON_SYSTEM_MIN = INT_MIN + 1
//   默认的 scheduleUpdate 优先级 = 0

// 设置高优先级更新（比默认的 0 更早执行）
scheduleUpdateWithPriority(-1);

// 低优先级更新（比默认的 0 更晚执行）
scheduleUpdateWithPriority(1);
```

```
执行顺序：
  PRIORITY_SYSTEM (INT_MIN) → 引擎内部
  -1 → 高优先级用户逻辑
   0 → 默认优先级用户逻辑
   1 → 低优先级用户逻辑
```

## 帧率控制

### 设置目标帧率

```cpp
auto director = Director::getInstance();

// 设置动画间隔（帧间隔），参数是秒
director->setAnimationInterval(1.0f / 60.0f);  // 60 FPS
director->setAnimationInterval(1.0f / 30.0f);  // 30 FPS

// 获取当前帧率
float fps = director->getFrameRate(); // 实际帧率
```

### 显示调试信息

```cpp
// 左下角显示 FPS、drawCall 数、顶点数
director->setDisplayStats(true);

// 输出示例：
// GL verts = 4        ← 本帧提交的顶点数
// GL calls = 1        ← 本帧 draw call 数
// 60.0 / 0.017        ← 帧率 / 帧时间
```

### KrKr2 中的帧率设置

KrKr2 在 `AppDelegate.cpp` 中设置了 60 FPS：

```cpp
// cpp/core/environ/cocos2d/AppDelegate.cpp 第 69 行
director->setAnimationInterval(1.0f / 60);
```

同时在 `TVPMainScene::update()` 中自行计算和显示 FPS：

```cpp
// cpp/core/environ/cocos2d/MainScene.cpp 第 2021-2070 行（简化）
void TVPMainScene::update(float delta) {
    ::Application->Run(); // 驱动 KiriKiri 引擎主循环

    // FPS 统计
    ++_frames;
    _accumDt += delta;
    if (_accumDt > CC_DIRECTOR_STATS_INTERVAL) {
        _frameRate = _frames / _accumDt;
        _frames = 0;
        _accumDt = 0;
        if (_fpsLabel) {
            char buf[32];
            snprintf(buf, sizeof(buf), "%.1f", _frameRate);
            _fpsLabel->setString(buf);
        }
    }
}
```

注意这里的关键设计：KrKr2 **利用 Cocos2d-x 的帧调度来驱动 KiriKiri 引擎的主循环**（`::Application->Run()`）。这是典型的"宿主引擎驱动嵌入引擎"模式。

## 完整示例：自定义主循环驱动

下面是一个完整的示例，演示如何利用 Director 和 Scheduler 构建游戏逻辑：

```cpp
// MainScene.h
#pragma once
#include "cocos2d.h"

class MainScene : public cocos2d::Scene {
public:
    static MainScene* create();
    bool init() override;
    void update(float dt) override;

private:
    void onSecondTimer(float dt);

    cocos2d::Label* _fpsLabel = nullptr;
    cocos2d::Label* _timeLabel = nullptr;
    float _elapsedTime = 0.0f;
    int _frameCount = 0;
};

// MainScene.cpp
#include "MainScene.h"

USING_NS_CC; // 展开为 using namespace cocos2d

MainScene* MainScene::create() {
    auto scene = new(std::nothrow) MainScene();
    if (scene && scene->init()) {
        scene->autorelease();
        return scene;
    }
    CC_SAFE_DELETE(scene);
    return nullptr;
}

bool MainScene::init() {
    if (!Scene::init()) return false;

    auto visibleSize = Director::getInstance()->getVisibleSize();
    auto origin = Director::getInstance()->getVisibleOrigin();

    // 创建 FPS 显示标签
    _fpsLabel = Label::createWithSystemFont("FPS: 0", "Arial", 24);
    _fpsLabel->setPosition(Vec2(
        origin.x + 100,
        origin.y + visibleSize.height - 30
    ));
    addChild(_fpsLabel, 10); // zOrder=10，显示在最上层

    // 创建经过时间标签
    _timeLabel = Label::createWithSystemFont("Time: 0.0s", "Arial", 20);
    _timeLabel->setPosition(Vec2(
        origin.x + visibleSize.width / 2,
        origin.y + visibleSize.height / 2
    ));
    addChild(_timeLabel);

    // 注册每帧更新
    scheduleUpdate();

    // 注册每秒定时器
    schedule(
        CC_SCHEDULE_SELECTOR(MainScene::onSecondTimer),
        1.0f // 每 1 秒
    );

    CCLOG("MainScene 初始化完成");
    return true;
}

void MainScene::update(float dt) {
    _frameCount++;
    _elapsedTime += dt;

    // 每 0.5 秒更新 FPS 显示
    static float accum = 0;
    static int frames = 0;
    accum += dt;
    frames++;
    if (accum >= 0.5f) {
        float fps = frames / accum;
        _fpsLabel->setString(
            StringUtils::format("FPS: %.1f", fps)
        );
        accum = 0;
        frames = 0;
    }

    // 更新时间显示
    _timeLabel->setString(
        StringUtils::format("Time: %.1fs", _elapsedTime)
    );
}

void MainScene::onSecondTimer(float dt) {
    CCLOG("已运行 %d 帧，经过 %.1f 秒",
          _frameCount, _elapsedTime);
}
```

### 对应的 AppDelegate 启动代码

```cpp
// AppDelegate.cpp
#include "AppDelegate.h"
#include "MainScene.h"

USING_NS_CC;

bool AppDelegate::applicationDidFinishLaunching() {
    auto director = Director::getInstance();

    // 创建 OpenGL 视图
    auto glView = director->getOpenGLView();
    if (!glView) {
        #if (CC_TARGET_PLATFORM == CC_PLATFORM_WIN32)
        glView = GLViewImpl::createWithRect(
            "Demo", Rect(0, 0, 960, 640));
        #else
        glView = GLViewImpl::create("Demo");
        #endif
        director->setOpenGLView(glView);
    }

    // 设计分辨率（逻辑坐标系大小）
    glView->setDesignResolutionSize(
        960, 640, ResolutionPolicy::SHOW_ALL);

    // 60 FPS
    director->setAnimationInterval(1.0f / 60);

    // 显示调试统计
    director->setDisplayStats(true);

    // 创建并运行主场景
    auto scene = MainScene::create();
    director->runWithScene(scene);

    return true;
}
```

## 动手实践

### 练习：实现一个帧率监控器

按以下步骤创建一个带帧率监控的空场景：

1. 创建 Cocos2d-x 空项目（使用 `cocos new` 命令或手动配置 CMake）
2. 创建 `MonitorScene` 类继承 `Scene`
3. 在 `init()` 中注册 `scheduleUpdate()`
4. 在 `update(dt)` 中统计帧率，每 0.5 秒刷新一次 Label 显示
5. 额外添加一个 1 秒定时器，打印最近 1 秒的最大/最小帧时间

预期输出：

```
[console] 最近1秒: 最大帧时间 = 18.2ms, 最小帧时间 = 15.8ms, 平均FPS = 60.2
[console] 最近1秒: 最大帧时间 = 17.5ms, 最小帧时间 = 16.0ms, 平均FPS = 59.8
```

## 对照项目源码

KrKr2 中 Director 的使用集中在以下文件：

### `AppDelegate.cpp`（第 40-95 行）—— Director 初始化

```cpp
// cpp/core/environ/cocos2d/AppDelegate.cpp
bool TVPAppDelegate::applicationDidFinishLaunching() {
    auto director = Director::getInstance();
    auto glview = director->getOpenGLView();
    if (!glview) {
        // 平台特定的 GLView 创建
        glview = cocos2d::GLViewImpl::createWithRect(
            "krkr2", cocos2d::Rect(0, 0, 960, 640));
    }
    director->setOpenGLView(glview);

    // 设计分辨率 960×640
    glview->setDesignResolutionSize(
        960, 640, ResolutionPolicy::EXACT_FIT);

    director->setAnimationInterval(1.0f / 60); // 60 FPS

    // 创建主场景
    auto scene = TVPMainScene::CreateInstance();
    director->runWithScene(scene);
}
```

### `MainScene.cpp`（第 2021-2070 行）—— 帧更新

KrKr2 的 `TVPMainScene` 重写了 `update()` 方法，在每帧中：
1. 调用 `::Application->Run()` 驱动 KiriKiri 引擎
2. 统计并显示 FPS

相关文件：
- `cpp/core/environ/cocos2d/AppDelegate.cpp` 第 40-95 行 — Director 完整初始化
- `cpp/core/environ/cocos2d/MainScene.cpp` 第 2021-2070 行 — 帧更新与 FPS 统计
- `platforms/windows/main.cpp` 第 30-44 行 — Windows 入口调用 `TVPAppDelegate::run()`
- `platforms/linux/main.cpp` 第 15-26 行 — Linux 入口调用 `TVPAppDelegate::run()`

## 常见错误及解决方案

### 错误 1：在 Director 初始化前使用它

```cpp
// 错误：在 applicationDidFinishLaunching 之前调用
auto size = Director::getInstance()->getVisibleSize();
// 结果：GLView 为空，getVisibleSize 返回 (0, 0)
```

**解决方案：** 确保在 `applicationDidFinishLaunching()` 中先设置 GLView：

```cpp
bool AppDelegate::applicationDidFinishLaunching() {
    auto director = Director::getInstance();
    auto glView = GLViewImpl::create("App");
    director->setOpenGLView(glView); // 必须先设置

    auto size = director->getVisibleSize(); // 现在可以正确获取
}
```

### 错误 2：scheduleUpdate 后忘记实现 update

```cpp
class MyScene : public Scene {
    bool init() override {
        Scene::init();
        scheduleUpdate(); // 注册了更新
        return true;
    }
    // 忘记重写 update(float) → 编译通过但无任何效果
};
```

**解决方案：** 必须重写 `update(float dt)` 方法。编译器不会报错（因为基类有默认空实现），但逻辑不会执行。

### 错误 3：schedule 的 key 冲突

```cpp
// 两个 lambda 使用相同 key，后者覆盖前者
schedule([](float dt) { /* A */ }, 1.0f, "timer");
schedule([](float dt) { /* B */ }, 2.0f, "timer"); // A 被覆盖！
```

**解决方案：** 每个 schedule 使用唯一的 key：

```cpp
schedule([](float dt) { /* A */ }, 1.0f, "timer_a");
schedule([](float dt) { /* B */ }, 2.0f, "timer_b");
```

## 本节小结

- **Director** 是 Cocos2d-x 的全局单例，管理场景生命周期、主循环、帧率和 GL 上下文
- **主循环** 每帧依次执行：计算 dt → 更新调度器 → 渲染场景 → 交换缓冲区 → 清理内存
- **deltaTime** 是帧间时间差（秒），用于实现帧率无关的游戏逻辑
- **Scheduler** 提供三种注册方式：`scheduleUpdate`（每帧）、`schedule`（自定义间隔）、`scheduleOnce`（延迟一次）
- KrKr2 利用 `scheduleUpdate` 在每帧驱动 KiriKiri 引擎的 `::Application->Run()`

## 练习题与答案

### 题目 1：Director 主循环执行顺序

以下步骤是 Director 主循环中 `drawScene()` 的关键阶段，请按正确的执行顺序排列：

A. 渲染场景（Scene::render）
B. 交换缓冲区（SwapBuffers）
C. 计算 deltaTime
D. 更新调度器（Scheduler::update）
E. 处理场景切换
F. 执行渲染命令（Renderer::render）

<details>
<summary>查看答案</summary>

正确顺序：**C → D → E → A → F → B**

1. **C** — `calculateDeltaTime()`：计算距上帧的时间差
2. **D** — `_scheduler->update(_deltaTime)`：更新所有注册的定时回调
3. **E** — `setNextScene()`：如果有待切换的场景，在此执行
4. **A** — `_runningScene->render()`：遍历节点树，生成渲染命令
5. **F** — `_renderer->render()`：执行缓冲的 GL 渲染命令
6. **B** — `_openGLView->swapBuffers()`：前后缓冲区交换，画面呈现

</details>

### 题目 2：实现一个倒计时定时器

使用 Cocos2d-x 的 Scheduler 实现一个 10 秒倒计时，每秒更新 Label 显示剩余时间，倒计时结束后显示"Time's Up!"并停止定时器。

<details>
<summary>查看答案</summary>

```cpp
#include "cocos2d.h"
USING_NS_CC;

class CountdownScene : public Scene {
public:
    static CountdownScene* create() {
        auto s = new(std::nothrow) CountdownScene();
        if (s && s->init()) { s->autorelease(); return s; }
        CC_SAFE_DELETE(s);
        return nullptr;
    }

    bool init() override {
        if (!Scene::init()) return false;

        auto size = Director::getInstance()->getVisibleSize();
        auto origin = Director::getInstance()->getVisibleOrigin();

        // 创建倒计时标签
        _label = Label::createWithSystemFont("10", "Arial", 72);
        _label->setPosition(Vec2(
            origin.x + size.width / 2,
            origin.y + size.height / 2
        ));
        addChild(_label);

        _remaining = 10; // 10 秒倒计时

        // 每 1 秒触发一次
        schedule([this](float dt) {
            _remaining--;
            if (_remaining > 0) {
                _label->setString(std::to_string(_remaining));
            } else {
                _label->setString("Time's Up!");
                unschedule("countdown"); // 停止定时器
            }
        }, 1.0f, "countdown");

        return true;
    }

private:
    Label* _label = nullptr;
    int _remaining = 0;
};

// 在 AppDelegate 中启动：
// director->runWithScene(CountdownScene::create());
```

关键点：
- 使用 `schedule(lambda, interval, key)` 注册 1 秒间隔回调
- 倒计时结束后用 `unschedule("countdown")` 取消，避免继续触发
- `_remaining` 为成员变量，lambda 中通过 `this` 捕获

</details>

### 题目 3：为什么 KrKr2 在 update() 中调用 ::Application->Run()？

阅读 KrKr2 的 `MainScene.cpp`，解释为什么 KiriKiri 引擎的主循环（`::Application->Run()`）被放在 Cocos2d-x 的 `update()` 回调中执行，而不是独立运行。

<details>
<summary>查看答案</summary>

KrKr2 采用"宿主引擎驱动嵌入引擎"的架构：

1. **单线程约束**：OpenGL 渲染必须在主线程执行。Cocos2d-x 的主循环控制着 GL 上下文，如果 KiriKiri 引擎独立运行自己的主循环，两个循环会争抢主线程控制权，导致渲染冲突或死锁。

2. **帧同步**：将 KiriKiri 引擎的逻辑更新（`::Application->Run()`）放在 Cocos2d-x 的 `update()` 中，保证两个引擎的帧步调一致。每帧先执行 KiriKiri 逻辑（更新游戏状态、生成渲染数据），再由 Cocos2d-x 渲染。

3. **生命周期统一**：Cocos2d-x 管理窗口创建、事件分发、暂停/恢复。如果 KiriKiri 有独立循环，暂停/恢复等生命周期事件无法正确传递。

4. **渲染桥接**：KiriKiri 的渲染输出通过 `DrawSprite`（Cocos2d-x Sprite 子类）显示。只有在同一帧的同一渲染流水线中，KiriKiri 的图像才能正确合成到 Cocos2d-x 的场景图中。

简言之：一山不容二主循环。Cocos2d-x 是宿主，KiriKiri 是客人，客人的逻辑在宿主的每帧回调中执行。

</details>

## 下一步

→ [02-Scene/Layer/Node树.md](02-Scene-Layer-Node树.md) — 深入了解 Cocos2d-x 的场景图核心类层次结构
