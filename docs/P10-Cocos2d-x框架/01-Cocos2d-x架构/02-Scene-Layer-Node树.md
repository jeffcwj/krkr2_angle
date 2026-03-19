# Scene / Layer / Node 树

> **所属模块：** P10-Cocos2d-x 框架
> **前置知识：** [01-Director与主循环](01-Director与主循环.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 Cocos2d-x 场景图（Scene Graph）的树形结构
2. 区分 Node、Scene、Layer 三个核心类的职责
3. 使用 Node 的基本属性（位置、缩放、旋转、锚点、可见性）
4. 构建多层次的节点树并理解渲染遍历流程
5. 对照 KrKr2 的 TVPMainScene 理解实际项目中的节点树设计

## 场景图（Scene Graph）概述

### 什么是场景图

场景图是一种**树形数据结构**，用于组织和管理所有需要渲染的对象。Cocos2d-x 的核心渲染机制就是**每帧遍历场景图**，按深度优先顺序访问所有节点，生成渲染命令。

```
Scene (根节点)
├── Background (Layer)
│   ├── Sky (Sprite)
│   └── Ground (Sprite)
├── GameLayer (Layer)
│   ├── Player (Sprite)
│   │   ├── Weapon (Sprite)
│   │   └── HealthBar (ProgressTimer)
│   └── Enemy (Sprite)
└── UILayer (Layer)
    ├── ScoreLabel (Label)
    └── PauseButton (Button)
```

场景图的关键特性：

| 特性 | 说明 |
|------|------|
| **层次继承** | 子节点的变换（位置、旋转、缩放）相对于父节点 |
| **渲染顺序** | 由 zOrder 和树遍历顺序决定，后绘制的覆盖先绘制的 |
| **生命周期绑定** | 父节点销毁时，所有子节点一起销毁 |
| **坐标空间** | 每个节点有自己的本地坐标系，由父节点的变换矩阵确定世界位置 |

## Node：一切节点的基类

### Node 的核心属性

`Node` 是 Cocos2d-x 中**所有可显示对象的基类**（类似 Unity 的 `GameObject` 或 DOM 中的 `Element`）。Scene、Sprite、Label、Layer 全都继承自 Node。

```cpp
#include "cocos2d.h"
USING_NS_CC;

// Node 的核心属性演示
auto node = Node::create();

// === 位置 ===
node->setPosition(Vec2(100, 200));  // 设置位置（相对于父节点）
node->setPositionX(100);            // 单独设置 X
node->setPositionY(200);            // 单独设置 Y
Vec2 pos = node->getPosition();     // 获取位置

// === 锚点 ===
// 锚点决定节点的"中心点"位置，范围 (0,0) ~ (1,1)
// (0,0) 左下角  (0.5,0.5) 中心  (1,1) 右上角
node->setAnchorPoint(Vec2(0.5f, 0.5f)); // 中心锚点（默认值）
node->setAnchorPoint(Vec2::ZERO);        // 左下角锚点

// === 尺寸 ===
node->setContentSize(Size(100, 50)); // 节点内容尺寸
Size size = node->getContentSize();

// === 缩放 ===
node->setScale(2.0f);       // 统一缩放 2 倍
node->setScaleX(1.5f);      // X 方向缩放
node->setScaleY(0.8f);      // Y 方向缩放

// === 旋转 ===
node->setRotation(45.0f);   // 顺时针旋转 45 度

// === 可见性 ===
node->setVisible(true);     // 显示
node->setVisible(false);    // 隐藏（仍参与逻辑更新，只是不渲染）

// === 透明度 ===
node->setOpacity(255);      // 完全不透明（0-255）
node->setOpacity(128);      // 半透明
node->setCascadeOpacityEnabled(true); // 子节点继承父透明度

// === 标签与名称 ===
node->setTag(42);                    // 整数标签（旧 API）
node->setName("player");            // 字符串名称（推荐）

// === 显示顺序 ===
node->setLocalZOrder(10);           // 本地 Z 序（同父节点下排序）
node->setGlobalZOrder(100);         // 全局 Z 序（跨节点树排序）
```

### 锚点（Anchor Point）详解

锚点是 Cocos2d-x 新手最容易困惑的概念。它决定了节点的**定位基准点**和**旋转/缩放中心**：

```
锚点 (0,0)          锚点 (0.5,0.5)         锚点 (1,1)
┌─────────┐         ┌─────────┐          ┌─────────┐
│         │         │         │          │         │
│         │         │    ×    │          │         │
│         │         │  (中心)  │          │         │
×(此处)    │         └─────────┘          │      (此处)×
└─────────┘                              └─────────┘

× = setPosition 指定的位置将对齐到锚点
```

```cpp
// 锚点实际效果演示
auto sprite = Sprite::create("icon.png"); // 假设 100×100 像素

// 锚点 (0.5, 0.5)（默认），position=(200,150)
// → 精灵中心在 (200,150)，左下角在 (150,100)，右上角在 (250,200)
sprite->setAnchorPoint(Vec2(0.5f, 0.5f));
sprite->setPosition(Vec2(200, 150));

// 锚点 (0, 0)，position=(200,150)
// → 精灵左下角在 (200,150)，右上角在 (300,250)
sprite->setAnchorPoint(Vec2::ZERO);
sprite->setPosition(Vec2(200, 150));

// 锚点 (1, 1)，position=(200,150)
// → 精灵右上角在 (200,150)，左下角在 (100,50)
sprite->setAnchorPoint(Vec2(1.0f, 1.0f));
sprite->setPosition(Vec2(200, 150));
```

> **实用技巧：** 对于 UI 元素，锚点 (0,0) 配合绝对定位很方便；对于游戏对象，锚点 (0.5,0.5) 更直观（坐标指向中心）。

### 子节点管理

```cpp
auto parent = Node::create();
auto child1 = Sprite::create("a.png");
auto child2 = Sprite::create("b.png");
auto child3 = Sprite::create("c.png");

// 添加子节点
parent->addChild(child1);           // 默认 zOrder=0
parent->addChild(child2, 5);        // zOrder=5（更高→后渲染→覆盖在上）
parent->addChild(child3, -1, "bg"); // zOrder=-1，名称="bg"

// 获取子节点
auto found = parent->getChildByName("bg");        // 按名称查找
auto found2 = parent->getChildByTag(42);           // 按 tag 查找
auto& children = parent->getChildren();            // 获取所有子节点
int count = parent->getChildrenCount();            // 子节点数量

// 移除子节点
parent->removeChild(child1);                  // 移除指定子节点
parent->removeChildByName("bg");              // 按名称移除
parent->removeAllChildren();                  // 移除所有子节点
child2->removeFromParent();                   // 子节点自己从父节点脱离

// 获取父节点
Node* p = child1->getParent(); // 返回父节点指针（或 nullptr）
```

## Scene：场景容器

### Scene 的职责

Scene 是**场景图的根节点**，也是 Director 管理的基本单位。每个 Scene 代表应用的一个"页面"或"关卡"。

```cpp
// 场景与 Director 的关系
class CC_DLL Scene : public Node {
public:
    static Scene* create();
    static Scene* createWithSize(const Size& size);
    static Scene* createWithPhysics(); // 带物理引擎的场景

    // Scene 没有额外的渲染逻辑，它只是一个特殊的 Node
    // 特殊之处：Director 只接受 Scene 作为 runWithScene/replaceScene 的参数
};
```

### 创建自定义 Scene

```cpp
// GameScene.h
#pragma once
#include "cocos2d.h"

class GameScene : public cocos2d::Scene {
public:
    // 标准的 create 模式
    static GameScene* create();
    bool init() override;

    // 场景生命周期回调
    void onEnter() override;              // 进入场景时（动画前）
    void onEnterTransitionDidFinish() override; // 进入过渡动画完成后
    void onExit() override;               // 离开场景时
    void cleanup() override;              // 场景被彻底清理时
};

// GameScene.cpp
#include "GameScene.h"
USING_NS_CC;

GameScene* GameScene::create() {
    auto scene = new(std::nothrow) GameScene();
    if (scene && scene->init()) {
        scene->autorelease();
        return scene;
    }
    CC_SAFE_DELETE(scene);
    return nullptr;
}

bool GameScene::init() {
    if (!Scene::init()) return false;

    // 在这里构建场景内容
    auto label = Label::createWithSystemFont(
        "Hello Scene", "Arial", 32);
    auto size = Director::getInstance()->getVisibleSize();
    label->setPosition(size / 2);
    addChild(label);

    return true;
}

void GameScene::onEnter() {
    Scene::onEnter(); // 必须调用父类
    CCLOG("GameScene::onEnter — 场景进入");
}

void GameScene::onEnterTransitionDidFinish() {
    Scene::onEnterTransitionDidFinish();
    CCLOG("GameScene::onEnterTransitionDidFinish — 过渡动画完成");
}

void GameScene::onExit() {
    CCLOG("GameScene::onExit — 场景离开");
    Scene::onExit(); // 必须调用父类
}

void GameScene::cleanup() {
    CCLOG("GameScene::cleanup — 场景清理");
    Scene::cleanup();
}
```

### 场景生命周期

```
Scene 生命周期回调顺序：

创建:   init() → onEnter() → onEnterTransitionDidFinish()
                                    │
                              正常运行（每帧 update）
                                    │
销毁:   onExitTransitionDidStart() → onExit() → cleanup()
```

> **注意：** `onEnter()/onExit()` 在过渡动画**开始**时调用，而非结束时。如果需要在过渡动画完成后执行逻辑（如开始游戏计时），使用 `onEnterTransitionDidFinish()`。

## Layer：逻辑分层（Cocos2d-x 3.x 中已弱化）

### Layer 的历史与现状

在 Cocos2d-x 2.x 中，Layer 是场景的核心组织单位，负责接收触摸、键盘事件。到了 **3.x 版本**，事件系统被重构为 `EventDispatcher`，Layer 的独特功能已经不再必要：

| 版本 | Layer 的角色 |
|------|-------------|
| 2.x | **核心类** — 事件接收、加速度计、键盘输入都通过 Layer |
| 3.x | **可选类** — 事件通过 EventDispatcher 注册到任意 Node，Layer 仅作为逻辑分组 |
| 4.x | **已废弃** — 完全用 Node 替代 |

```cpp
// 3.x 中的 Layer —— 本质就是一个 Node，只多了几个便捷方法
class CC_DLL Layer : public Node {
public:
    static Layer* create();

    // 这些方法在 3.x 中内部都是调用 EventDispatcher
    // 已不推荐使用，但为兼容性保留
    virtual bool onTouchBegan(Touch*, Event*);
    virtual void onTouchMoved(Touch*, Event*);
    virtual void onTouchEnded(Touch*, Event*);

    virtual void onKeyPressed(EventKeyboard::KeyCode, Event*);
    virtual void onKeyReleased(EventKeyboard::KeyCode, Event*);
};
```

### 推荐做法：用 Node 替代 Layer

```cpp
// 旧写法（不推荐）：使用 Layer 分层
class GameLayer : public Layer { /* ... */ };
class UILayer : public Layer { /* ... */ };

// 新写法（推荐）：直接使用 Node 分组
bool GameScene::init() {
    if (!Scene::init()) return false;

    // 用普通 Node 作为逻辑分组
    auto gameNode = Node::create();
    gameNode->setName("game_layer");
    addChild(gameNode, 0);  // zOrder=0 游戏层

    auto uiNode = Node::create();
    uiNode->setName("ui_layer");
    addChild(uiNode, 10);  // zOrder=10 UI 层（覆盖在游戏层上）

    // 往各层添加内容
    auto player = Sprite::create("player.png");
    gameNode->addChild(player);

    auto scoreLabel = Label::createWithSystemFont("0", "Arial", 24);
    uiNode->addChild(scoreLabel);

    return true;
}
```

## 节点树遍历与渲染流程

### visit() 递归遍历

Director 每帧调用 `Scene::render()`，内部会递归调用每个节点的 `visit()` 方法：

```cpp
// Node::visit() 简化逻辑
void Node::visit(Renderer* renderer, const Mat4& parentTransform,
                 uint32_t parentFlags) {
    if (!_visible) return; // 不可见则跳过整个子树

    // 1. 计算变换矩阵
    uint32_t flags = processParentFlags(parentTransform, parentFlags);

    // 2. 按 zOrder 排序子节点
    sortAllChildren();

    // 3. 先绘制 zOrder < 0 的子节点（在自身后面）
    for (auto child : _children) {
        if (child->getLocalZOrder() < 0) {
            child->visit(renderer, _modelViewTransform, flags);
        } else {
            break;
        }
    }

    // 4. 绘制自身
    this->draw(renderer, _modelViewTransform, flags);

    // 5. 再绘制 zOrder >= 0 的子节点（在自身前面）
    for (auto child : _children) {
        if (child->getLocalZOrder() >= 0) {
            child->visit(renderer, _modelViewTransform, flags);
        }
    }
}
```

遍历顺序图示：

```
Scene (visit)
  │
  ├─ [zOrder=-1] Background
  │   ├─ Sky.draw()
  │   └─ Ground.draw()
  │
  ├─ Scene.draw()  (通常为空)
  │
  ├─ [zOrder=0] GameLayer
  │   ├─ Player.draw()
  │   └─ Enemy.draw()
  │
  └─ [zOrder=10] UILayer
      ├─ ScoreLabel.draw()
      └─ PauseButton.draw()

渲染结果（从下到上叠加）：
  [最底] Sky → Ground → Player → Enemy → Score → Pause [最顶]
```

## 完整示例：构建多层场景

```cpp
// MultiLayerScene.h
#pragma once
#include "cocos2d.h"

class MultiLayerScene : public cocos2d::Scene {
public:
    static MultiLayerScene* create();
    bool init() override;
    void update(float dt) override;

private:
    cocos2d::Node* _gameNode = nullptr;
    cocos2d::Node* _uiNode = nullptr;
    cocos2d::Sprite* _player = nullptr;
    cocos2d::Label* _infoLabel = nullptr;
    float _time = 0;
};

// MultiLayerScene.cpp
#include "MultiLayerScene.h"
USING_NS_CC;

MultiLayerScene* MultiLayerScene::create() {
    auto s = new(std::nothrow) MultiLayerScene();
    if (s && s->init()) { s->autorelease(); return s; }
    CC_SAFE_DELETE(s);
    return nullptr;
}

bool MultiLayerScene::init() {
    if (!Scene::init()) return false;

    auto visSize = Director::getInstance()->getVisibleSize();
    auto origin = Director::getInstance()->getVisibleOrigin();

    // ===== 背景层 (zOrder = -1) =====
    auto bgNode = Node::create();
    bgNode->setName("background");
    addChild(bgNode, -1);

    // 纯色背景（用 LayerColor 快速创建）
    auto bg = LayerColor::create(Color4B(30, 30, 60, 255));
    bgNode->addChild(bg);

    // ===== 游戏层 (zOrder = 0) =====
    _gameNode = Node::create();
    _gameNode->setName("game");
    addChild(_gameNode, 0);

    // 玩家精灵（用彩色矩形模拟）
    _player = Sprite::create();
    _player->setTextureRect(Rect(0, 0, 40, 40));
    _player->setColor(Color3B::GREEN);
    _player->setPosition(Vec2(
        origin.x + visSize.width / 2,
        origin.y + visSize.height / 2
    ));
    _gameNode->addChild(_player, 0, "player");

    // 几个"敌人"
    for (int i = 0; i < 3; i++) {
        auto enemy = Sprite::create();
        enemy->setTextureRect(Rect(0, 0, 30, 30));
        enemy->setColor(Color3B::RED);
        enemy->setPosition(Vec2(
            origin.x + 200 + i * 200,
            origin.y + 100 + i * 80
        ));
        _gameNode->addChild(enemy, 0,
            StringUtils::format("enemy_%d", i));
    }

    // ===== UI层 (zOrder = 10) =====
    _uiNode = Node::create();
    _uiNode->setName("ui");
    addChild(_uiNode, 10);

    _infoLabel = Label::createWithSystemFont(
        "节点数: 0", "Arial", 20);
    _infoLabel->setAnchorPoint(Vec2(0, 1)); // 左上角对齐
    _infoLabel->setPosition(Vec2(
        origin.x + 10,
        origin.y + visSize.height - 10
    ));
    _uiNode->addChild(_infoLabel);

    // 注册帧更新
    scheduleUpdate();

    CCLOG("场景节点树构建完成");
    CCLOG("  背景层子节点: %d", bgNode->getChildrenCount());
    CCLOG("  游戏层子节点: %d", _gameNode->getChildrenCount());
    CCLOG("  UI层子节点:   %d", _uiNode->getChildrenCount());

    return true;
}

void MultiLayerScene::update(float dt) {
    _time += dt;

    // 玩家做圆周运动
    auto visSize = Director::getInstance()->getVisibleSize();
    float cx = visSize.width / 2;
    float cy = visSize.height / 2;
    float radius = 100.0f;
    _player->setPosition(Vec2(
        cx + radius * cosf(_time * 2.0f),
        cy + radius * sinf(_time * 2.0f)
    ));

    // 更新信息标签
    int totalNodes = getChildrenCount(); // 直接子节点数
    _infoLabel->setString(StringUtils::format(
        "场景直接子节点: %d | 时间: %.1fs",
        totalNodes, _time
    ));
}
```

## 动手实践

1. **构建三层场景**：创建一个包含背景层（`Layer::create()` + 纯色 `LayerColor`）、游戏层（带 3 个 `Sprite` 节点）和 UI 层（带 `Label` 显示帧率）的场景。通过 `setLocalZOrder` 控制层级顺序，确认 UI 始终显示在最上方。

2. **节点遍历可视化**：为场景中的每个节点重写 `visit` 方法，在其中打印节点名和 zOrder。运行程序后观察控制台输出，验证 Cocos2d-x 的遍历顺序是否符合"zOrder < 0 的子节点 → 自身 → zOrder ≥ 0 的子节点"。

3. **动态增删节点**：编写一个按钮回调，每次点击创建一个新 `Sprite` 添加到场景中的随机位置（`setPosition(rand()%800, rand()%600)`），再编写一个按钮删除最后添加的节点。验证 `removeFromParent` 是否立即生效还是延迟到帧末。

4. **对照 KrKr2 节点树**：阅读 `cpp/core/environ/cocos2d/MainScene.cpp` 中 `TVPMainScene::initialize()` 的代码，画出完整的节点树结构图（_gameNode、_uiNode 及其子节点）。标注每个节点的 zOrder 值，分析为什么 _uiNode 的 zOrder 要设得很大。

## 对照项目源码

### KrKr2 的 TVPMainScene 节点树结构

KrKr2 在 `MainScene.cpp` 中定义了一个清晰的双层节点树：

```cpp
// cpp/core/environ/cocos2d/MainScene.cpp 第 1762-1815 行（简化）
bool TVPMainScene::initialize() {
    // 游戏节点 —— 承载 KiriKiri 引擎的渲染输出
    _gameNode = Node::create();
    _gameNode->setLocalZOrder(GAME_SCENE_ORDER);  // 0
    addChild(_gameNode);

    // UI 节点 —— 承载文件选择器、设置面板等 UI 表单
    _uiNode = Node::create();
    _uiNode->setLocalZOrder(UI_NODE_ORDER);        // 很大的值
    addChild(_uiNode);

    // 注册键盘/触摸/手柄监听器
    // ...
}
```

节点层次枚举（`MainScene.cpp` 第 30-36 行）：

```cpp
enum SCENE_ORDER {
    GAME_SCENE_ORDER = 0,        // 游戏渲染层
    GAME_CONSOLE_ORDER = 10,     // 调试控制台
    GAME_WINMGR_ORDER = 15,     // 窗口管理器覆盖层
    GAME_MENU_ORDER = 20,       // 游戏内菜单
    UI_NODE_ORDER = 100          // UI 表单层
};
```

完整的节点树：

```
TVPMainScene (Scene)
├── [zOrder=0]   GameNode (Node)
│   ├── TVPWindowLayer      ← KiriKiri 渲染窗口
│   └── ...                 ← 可能有多个窗口
├── [zOrder=10]  ConsoleNode ← 调试控制台
├── [zOrder=15]  WinMgrOverlay ← 窗口切换 UI
├── [zOrder=20]  GameMainMenu ← 浮动菜单
└── [zOrder=100] UINode (Node)
    ├── MaskLayer            ← 触摸屏蔽层
    └── UIForm Stack         ← 文件选择器、设置等
```

相关文件：
- `cpp/core/environ/cocos2d/MainScene.h` 第 1-94 行 — TVPMainScene 类声明
- `cpp/core/environ/cocos2d/MainScene.cpp` 第 30-36 行 — SCENE_ORDER 枚举定义
- `cpp/core/environ/cocos2d/MainScene.cpp` 第 1762-1815 行 — initialize() 节点树构建

## 常见错误及解决方案

### 错误 1：忘记调用父类的 init()

```cpp
bool MyScene::init() {
    // 错误：忘记调用 Scene::init()
    auto label = Label::createWithSystemFont("Hi", "Arial", 24);
    addChild(label); // 可能崩溃 —— 内部状态未初始化
    return true;
}

// 正确：
bool MyScene::init() {
    if (!Scene::init()) return false; // 必须先调用
    auto label = Label::createWithSystemFont("Hi", "Arial", 24);
    addChild(label);
    return true;
}
```

### 错误 2：将已有父节点的子节点 addChild 到另一个父节点

```cpp
auto child = Sprite::create("icon.png");
parent1->addChild(child);
parent2->addChild(child); // 断言失败！一个节点只能有一个父节点
```

**解决方案：** 先从原父节点移除，再添加到新父节点：

```cpp
child->retain();           // 防止移除时被释放
child->removeFromParent(); // 从 parent1 移除
parent2->addChild(child);  // 添加到 parent2
child->release();          // 平衡 retain
```

### 错误 3：在 onExit() 中忘记调用父类方法

```cpp
void MyScene::onExit() {
    // 做一些清理...
    saveGameState();
    // 错误：忘记调用 Scene::onExit()
    // 导致子节点的 onExit 不被调用，事件监听器不被移除
}

// 正确：
void MyScene::onExit() {
    saveGameState();
    Scene::onExit(); // 必须调用！
}
```

## 本节小结

- **场景图** 是 Cocos2d-x 的核心数据结构，所有可渲染对象组织为树形层次
- **Node** 是一切节点的基类，提供位置、缩放、旋转、锚点、透明度等基本属性
- **Scene** 是场景图的根节点，Director 通过它管理不同的"页面"
- **Layer** 在 3.x 中已弱化，推荐用 Node 替代；事件处理通过 EventDispatcher
- **visit()** 按 zOrder 排序递归遍历节点树：zOrder < 0 先绘制，自身次之，zOrder >= 0 最后
- KrKr2 的 TVPMainScene 采用**双层节点树**设计：GameNode（游戏渲染）+ UINode（UI 表单）

## 练习题与答案

### 题目 1：节点树遍历顺序

给定以下节点树，写出 `draw()` 的调用顺序：

```
Scene
├── A (zOrder=5)
│   ├── A1 (zOrder=-1)
│   └── A2 (zOrder=0)
├── B (zOrder=-2)
│   └── B1 (zOrder=0)
└── C (zOrder=0)
```

<details>
<summary>查看答案</summary>

按 visit() 算法，先递归处理 zOrder < 0 的子节点，再画自身，再处理 zOrder >= 0 的子节点：

1. **B** (zOrder=-2，最先)
   - B 的子节点中无 zOrder<0 的 → 画 **B.draw()**
   - **B1.draw()** (zOrder=0)
2. **Scene.draw()** (Scene 自身 zOrder=0，但它是根，在 B 之后)
3. **C** (zOrder=0)
   - **C.draw()**
4. **A** (zOrder=5，最后)
   - **A1.draw()** (zOrder=-1，先于 A 自身)
   - **A.draw()**
   - **A2.draw()** (zOrder=0)

完整顺序：B → B1 → Scene → C → A1 → A → A2

</details>

### 题目 2：构建一个三层场景

创建一个 Scene，包含三个逻辑层：背景层（zOrder=-1）、游戏层（zOrder=0）、HUD 层（zOrder=10）。背景层放一个全屏半透明黑色遮罩，游戏层放一个在屏幕中央的绿色方块，HUD 层放一个左上角的 FPS 标签。

<details>
<summary>查看答案</summary>

```cpp
#include "cocos2d.h"
USING_NS_CC;

class ThreeLayerScene : public Scene {
public:
    static ThreeLayerScene* create() {
        auto s = new(std::nothrow) ThreeLayerScene();
        if (s && s->init()) { s->autorelease(); return s; }
        CC_SAFE_DELETE(s);
        return nullptr;
    }

    bool init() override {
        if (!Scene::init()) return false;

        auto visSize = Director::getInstance()->getVisibleSize();
        auto origin = Director::getInstance()->getVisibleOrigin();

        // === 背景层 (zOrder=-1) ===
        auto bgLayer = Node::create();
        addChild(bgLayer, -1, "bg_layer");

        auto mask = LayerColor::create(
            Color4B(0, 0, 0, 128)); // 半透明黑色
        bgLayer->addChild(mask);

        // === 游戏层 (zOrder=0) ===
        auto gameLayer = Node::create();
        addChild(gameLayer, 0, "game_layer");

        auto block = Sprite::create();
        block->setTextureRect(Rect(0, 0, 60, 60));
        block->setColor(Color3B::GREEN);
        block->setPosition(Vec2(
            origin.x + visSize.width / 2,
            origin.y + visSize.height / 2
        ));
        gameLayer->addChild(block);

        // === HUD层 (zOrder=10) ===
        auto hudLayer = Node::create();
        addChild(hudLayer, 10, "hud_layer");

        auto fpsLabel = Label::createWithSystemFont(
            "FPS: 60.0", "Arial", 18);
        fpsLabel->setAnchorPoint(Vec2(0, 1));
        fpsLabel->setPosition(Vec2(
            origin.x + 10,
            origin.y + visSize.height - 10
        ));
        hudLayer->addChild(fpsLabel);

        CCLOG("三层场景构建完成:");
        CCLOG("  bg_layer   zOrder=-1, children=%d",
              bgLayer->getChildrenCount());
        CCLOG("  game_layer zOrder=0,  children=%d",
              gameLayer->getChildrenCount());
        CCLOG("  hud_layer  zOrder=10, children=%d",
              hudLayer->getChildrenCount());

        return true;
    }
};
```

关键点：
- 三个 Node 作为逻辑层，通过不同 zOrder 控制渲染顺序
- `LayerColor` 是快速创建纯色矩形的便捷类
- HUD 标签用锚点 (0,1) 实现左上角对齐
- 使用字符串 name 而非 tag 标识节点

</details>

## 下一步

→ [03-内存管理与引用计数.md](03-内存管理与引用计数.md) — 理解 Cocos2d-x 的 Ref 引用计数和 autorelease 机制
