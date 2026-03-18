# 父子关系与 zOrder

> **所属模块：** P10-Cocos2d-x 框架
> **前置知识：** [01-坐标系与变换矩阵](01-坐标系与变换矩阵.md)、[02-Scene-Layer-Node树](../01-Cocos2d-x架构/02-Scene-Layer-Node树.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 Node 的父子关系机制（addChild / removeChild / getParent）
2. 掌握 localZOrder 和 globalZOrder 的区别及排序规则
3. 理解 visit() 递归遍历与渲染顺序的关系
4. 使用 Tag 和 Name 系统查找子节点
5. 对照 KrKr2 源码理解 `SCENE_ORDER` 枚举如何组织节点层级

## 父子节点基础

### addChild 与节点树构建

Cocos2d-x 的 UI 通过**节点树**（Scene Graph）组织。每个 `Node` 可以有一个父节点和任意多个子节点，形成树状层级：

```
Scene (根节点)
├── Background (Sprite)
├── GameLayer (Node)
│   ├── Player (Sprite)
│   ├── Enemy1 (Sprite)
│   └── Enemy2 (Sprite)
└── UILayer (Node)
    ├── ScoreLabel (Label)
    └── PauseButton (Button)
```

使用 `addChild()` 方法将子节点添加到父节点：

```cpp
#include "cocos2d.h"
USING_NS_CC;

// 创建父节点
auto gameLayer = Node::create();
gameLayer->setName("GameLayer");

// 创建子节点
auto player = Sprite::create("player.png");
player->setPosition(Vec2(480, 320));  // 相对于 gameLayer 的本地坐标

// 添加到父节点（三种重载）
gameLayer->addChild(player);             // 默认 zOrder=0, tag=-1
gameLayer->addChild(player, 5);          // 指定 zOrder=5
gameLayer->addChild(player, 5, 100);     // 指定 zOrder=5, tag=100
gameLayer->addChild(player, 5, "player"); // 指定 zOrder=5, name="player"
```

`addChild()` 的完整签名：

```cpp
// Node.h 中的声明
virtual void addChild(Node* child);
virtual void addChild(Node* child, int localZOrder);
virtual void addChild(Node* child, int localZOrder, int tag);
virtual void addChild(Node* child, int localZOrder, const std::string& name);
```

> **注意：** 一个节点只能有一个父节点。如果 `child` 已经有父节点，`addChild()` 会触发断言失败。你必须先从旧父节点 `removeChild()` 后再添加到新父节点。

### removeChild 与节点移除

```cpp
// 从父节点移除子节点
gameLayer->removeChild(player);           // 移除并释放（cleanup=true）
gameLayer->removeChild(player, false);    // 移除但不释放（保留 action 等）

// 通过 tag 移除
gameLayer->removeChildByTag(100);
gameLayer->removeChildByName("player");

// 移除所有子节点
gameLayer->removeAllChildren();
gameLayer->removeAllChildrenWithCleanup(false);

// 从父节点中移除自己
player->removeFromParent();
player->removeFromParentAndCleanup(false);
```

`cleanup` 参数控制是否清除节点上运行的 Action 和定时器：

| cleanup 值 | Action | Schedule | 用途 |
|------------|--------|----------|------|
| `true`（默认） | 停止并移除 | 取消所有 | 彻底移除，准备回收 |
| `false` | 保留 | 保留 | 暂时移除，稍后重新添加 |

### 父子关系的传递效应

父节点的变换（位置、旋转、缩放）会自动传递给所有子节点：

```cpp
auto parent = Node::create();
parent->setPosition(Vec2(100, 100));
parent->setScale(2.0f);       // 放大 2 倍
parent->setRotation(45.0f);   // 顺时针旋转 45°

auto child = Sprite::create("icon.png");
child->setPosition(Vec2(50, 0));  // 本地坐标 (50, 0)
parent->addChild(child);

// child 的世界坐标不是 (150, 100)
// 而是经过父节点的缩放和旋转变换后的结果：
// 1. 缩放：本地 (50, 0) × 2 = (100, 0)
// 2. 旋转 45°：(100×cos45° - 0×sin45°, 100×sin45° + 0×cos45°) ≈ (70.7, 70.7)
// 3. 平移：(70.7 + 100, 70.7 + 100) ≈ (170.7, 170.7)
Vec2 worldPos = parent->convertToWorldSpace(child->getPosition());
// worldPos ≈ (170.7, 170.7)
```

变换传递链的可视化：

```
          世界空间
          ┌─────────────────────────┐
          │                         │
          │    Parent (100,100)     │
          │    scale=2, rot=45°    │
          │    ┌───────────┐        │
          │    │  Child     │        │
          │    │  local(50,0)│       │
          │    │  world≈     │       │
          │    │  (170,170)  │       │
          │    └───────────┘        │
          └─────────────────────────┘

变换矩阵传递：
World_Matrix(child) = World_Matrix(parent) × Local_Matrix(child)
                    = [T(100,100) × R(45°) × S(2)] × [T(50,0)]
```

除了变换传递，以下属性也会影响子节点：

| 父节点属性 | 对子节点的影响 | 说明 |
|------------|---------------|------|
| `visible` | 父节点不可见 → 子节点也不渲染 | visit() 跳过不可见节点的整个子树 |
| `opacity` | 仅当 `cascadeOpacityEnabled=true` 时传递 | 默认 true（3.x），子节点透明度 = 父×子 |
| `color` | 仅当 `cascadeColorEnabled=true` 时传递 | 默认 true（3.x），颜色分量逐一相乘 |
| `position/scale/rotation` | 始终传递 | 通过变换矩阵级联 |

```cpp
// 透明度传递示例
auto parent = Node::create();
parent->setOpacity(128);                  // 50% 透明
parent->setCascadeOpacityEnabled(true);   // 3.x 默认已为 true

auto child = Sprite::create("icon.png");
child->setOpacity(200);                   // 约 78% 不透明
parent->addChild(child);

// child 实际渲染透明度 = 128/255 × 200/255 ≈ 0.39 → 约 100/255
// 即视觉上约 39% 不透明
```

## zOrder 与渲染顺序

### localZOrder 详解

`localZOrder`（简称 zOrder）决定了同一父节点下子节点的**渲染顺序**。值越小越先渲染（越靠后面），值越大越后渲染（越靠前面）：

```cpp
auto scene = Scene::create();

auto bg = Sprite::create("bg.png");
scene->addChild(bg, -10);       // 最先渲染 — 最底层

auto midLayer = Node::create();
scene->addChild(midLayer, 0);   // 中间层

auto uiLayer = Node::create();
scene->addChild(uiLayer, 10);   // 最后渲染 — 最顶层
```

渲染顺序可视化：

```
屏幕（观察者视角）
━━━━━━━━━━━━━━━━━━━━━━
  uiLayer     (z=10)  ← 最顶层（最后绘制，覆盖下方）
──────────────────────
  midLayer    (z=0)   ← 中间层
──────────────────────
  bg          (z=-10) ← 最底层（最先绘制，被上方覆盖）
━━━━━━━━━━━━━━━━━━━━━━
```

当 zOrder 相同时，按 `addChild()` 的调用顺序决定（先添加先渲染）：

```cpp
auto scene = Scene::create();

// 三个节点 zOrder 都是 0
auto a = Sprite::create("a.png");
auto b = Sprite::create("b.png");
auto c = Sprite::create("c.png");

scene->addChild(a);  // 先添加 → 先渲染 → 最底
scene->addChild(b);  // 中间
scene->addChild(c);  // 后添加 → 后渲染 → 最顶
```

运行时修改 zOrder：

```cpp
// 运行时改变渲染顺序
player->setLocalZOrder(100);  // 提升到最顶层

// 或通过 reorderChild（效果相同）
parent->reorderChild(player, 100);
```

> **性能提示：** 频繁调用 `setLocalZOrder()` 会触发父节点的子节点重排序。如果需要大量节点频繁改变绘制顺序，考虑使用 `globalZOrder` 代替。

### globalZOrder 详解

`globalZOrder` 打破了节点树的层级限制，在**全局范围**内控制渲染顺序：

```cpp
// globalZOrder 默认为 0
// 当 globalZOrder 非零时，它会覆盖 localZOrder 的排序

auto scene = Scene::create();

// 底层背景 — 普通 localZOrder
auto bg = Sprite::create("bg.png");
scene->addChild(bg, -10);

// 游戏层中的特效粒子 — 需要显示在所有 UI 之上
auto gameLayer = Node::create();
scene->addChild(gameLayer, 0);

auto particle = ParticleExplosion::create();
gameLayer->addChild(particle);
particle->setGlobalZOrder(1000.0f);  // 全局最顶层

// UI层 — localZOrder=10，但被粒子的 globalZOrder 覆盖
auto uiLayer = Node::create();
scene->addChild(uiLayer, 10);

auto dialog = Sprite::create("dialog.png");
uiLayer->addChild(dialog);
// dialog 的 globalZOrder 为默认 0
// particle 的 globalZOrder 为 1000
// 因此 particle 会渲染在 dialog 之上
```

`localZOrder` 与 `globalZOrder` 的对比：

| 特性 | localZOrder | globalZOrder |
|------|-------------|--------------|
| 类型 | `int` | `float` |
| 作用范围 | 同一父节点的兄弟之间 | 全场景所有节点 |
| 默认值 | 0 | 0.0f |
| 设置方式 | `addChild(child, z)` 或 `setLocalZOrder(z)` | `setGlobalZOrder(z)` |
| 优先级 | 低（当 globalZOrder 非零时被覆盖） | 高（非零时忽略 localZOrder） |
| 适用场景 | 常规 UI 层级管理 | 粒子特效、全局弹窗、调试覆盖层 |

### visit() 遍历与排序算法

Cocos2d-x 每帧通过 `visit()` 方法递归遍历节点树，按 zOrder 排序后绘制：

```
visit(renderer, parentTransform, flags) {
    // 1. 如果不可见，跳过整个子树
    if (!visible) return;
    
    // 2. 按 localZOrder 排序子节点
    sortAllChildren();
    
    // 3. 绘制 zOrder < 0 的子节点（在自身之后/下方）
    for (child : children where child.zOrder < 0)
        child.visit(renderer, transform, flags);
    
    // 4. 绘制自身
    this->draw(renderer, transform, flags);
    
    // 5. 绘制 zOrder >= 0 的子节点（在自身之前/上方）
    for (child : children where child.zOrder >= 0)
        child.visit(renderer, transform, flags);
}
```

这意味着 zOrder < 0 的子节点渲染在父节点**下方**，zOrder >= 0 的子节点渲染在父节点**上方**：

```cpp
auto parent = Sprite::create("parent.png");

auto behind = Sprite::create("behind.png");
parent->addChild(behind, -1);    // zOrder < 0 → 在 parent 下方

auto inFront = Sprite::create("front.png");
parent->addChild(inFront, 1);    // zOrder >= 0 → 在 parent 上方

// 渲染顺序（从底到顶）：behind → parent → inFront
```

## 节点查找：Tag 与 Name

### Tag 系统（整数标识）

```cpp
// 使用枚举定义 Tag，避免魔数
enum NodeTag {
    TAG_PLAYER = 100,
    TAG_ENEMY  = 200,
    TAG_BULLET = 300,
};

auto player = Sprite::create("player.png");
scene->addChild(player, 0, TAG_PLAYER);

// 查找子节点
auto found = scene->getChildByTag(TAG_PLAYER);
if (found) {
    found->setPosition(Vec2(100, 200));
}

// 修改 Tag
player->setTag(TAG_ENEMY);  // 改变身份（例如变身系统）
```

### Name 系统（字符串标识，3.x 推荐）

```cpp
auto player = Sprite::create("player.png");
player->setName("Player");
scene->addChild(player, 0, "Player");  // name 版 addChild

// 按名称查找（仅直接子节点）
auto found = scene->getChildByName("Player");

// 递归查找（搜索整个子树）— 使用路径语法
// 格式："/子节点名/孙节点名" 或 "//任意深度查找"
auto gameLayer = Node::create();
gameLayer->setName("GameLayer");
scene->addChild(gameLayer);

auto player2 = Sprite::create("player.png");
player2->setName("Player");
gameLayer->addChild(player2);

// 从 scene 查找 GameLayer 下的 Player
auto deepFound = scene->getChildByName("Player");  // 仅查直接子节点，找不到
// 需要先获取 GameLayer 再查
auto gl = scene->getChildByName("GameLayer");
auto p = gl->getChildByName("Player");  // 找到了
```

### Tag vs Name 对比

| 特性 | Tag | Name |
|------|-----|------|
| 类型 | `int` | `std::string` |
| 唯一性 | 不强制唯一（多个节点可同 tag） | 不强制唯一 |
| 性能 | 快（整数比较） | 慢（字符串比较） |
| 可读性 | 低（需要查枚举定义） | 高（直接描述节点用途） |
| 推荐场景 | 性能敏感 + 节点数量大 | 调试/开发阶段 + 复杂 UI |
| getChildBy | `getChildByTag(int)` | `getChildByName(string)` |

### 遍历所有子节点

```cpp
// 方式一：getChildren() 获取 Vector 引用
auto& children = parent->getChildren();
for (auto child : children) {
    CCLOG("Child: %s, z=%d", child->getName().c_str(),
          child->getLocalZOrder());
}

// 方式二：enumerateChildren — 支持通配符搜索
// 回调返回 true 表示停止搜索
parent->enumerateChildren("Enemy*", [](Node* node) -> bool {
    CCLOG("Found enemy: %s", node->getName().c_str());
    return false;  // 继续搜索
});

// 方式三：递归搜索（"//" 前缀表示搜索所有后代）
scene->enumerateChildren("//Bullet*", [](Node* node) -> bool {
    node->removeFromParent();  // 移除所有子弹
    return false;
});

// 获取子节点数量
ssize_t count = parent->getChildrenCount();
```

## 常见错误与解决方案

### 错误 1：重复添加已有父节点的子节点

```cpp
// ❌ 错误：child 已经有父节点
auto child = Sprite::create("icon.png");
parentA->addChild(child);
parentB->addChild(child);  // 断言失败！CCASSERT(child->_parent == nullptr)

// ✅ 正确：先移除再添加
parentA->addChild(child);
child->retain();            // 防止 removeFromParent 释放
child->removeFromParent();
parentB->addChild(child);
child->release();           // 平衡 retain

// ✅ 更安全的写法
child->retain();
if (child->getParent()) {
    child->removeFromParent();
}
parentB->addChild(child);
child->release();
```

### 错误 2：在遍历子节点时修改子节点列表

```cpp
// ❌ 错误：边遍历边删除导致迭代器失效
auto& children = parent->getChildren();
for (auto child : children) {
    if (shouldRemove(child)) {
        parent->removeChild(child);  // 崩溃！修改了正在遍历的容器
    }
}

// ✅ 正确：先收集再删除
std::vector<Node*> toRemove;
for (auto child : parent->getChildren()) {
    if (shouldRemove(child)) {
        toRemove.push_back(child);
    }
}
for (auto node : toRemove) {
    node->removeFromParent();
}

// ✅ 或者使用倒序遍历（Cocos2d-x Vector 支持）
auto& children2 = parent->getChildren();
for (ssize_t i = children2.size() - 1; i >= 0; --i) {
    if (shouldRemove(children2.at(i))) {
        parent->removeChild(children2.at(i));
    }
}
```

### 错误 3：zOrder 设置不生效

```cpp
// ❌ 错误：addChild 之后立即改 zOrder，但期望在同帧立即看到效果
parent->addChild(nodeA, 5);
parent->addChild(nodeB, 10);
nodeA->setLocalZOrder(20);
// nodeA 的 zOrder 已更新，但排序在下一次 visit() 时才生效
// 同帧内获取 getChildren() 可能尚未重排

// ✅ 正确理解：setLocalZOrder 标记 _reorderChildDirty=true
// 下一帧 visit() 调用 sortAllChildren() 时才真正重排
// 如果需要立即排序：
parent->sortAllChildren();  // 手动触发排序
```

## 对照项目源码

KrKr2 项目在 `MainScene.cpp` 中定义了一组 zOrder 常量来组织节点层级：

> **文件：** `cpp/core/environ/cocos2d/MainScene.cpp` 第 30-36 行

```cpp
// KrKr2 的节点层级定义
enum SCENE_ORDER {
    GAME_SCENE_ORDER   = 0,    // 游戏场景层 — 渲染游戏画面
    GAME_CONSOLE_ORDER = 10,   // 控制台层 — 调试输出
    GAME_WINMGR_ORDER  = 15,   // 窗口管理器覆盖层
    GAME_MENU_ORDER    = 20,   // 主菜单浮窗层
    UI_NODE_ORDER      = 100,  // UI 表单层 — 设置/文件选择等
};
```

这些常量在 `TVPMainScene::initialize()` 中使用：

> **文件：** `cpp/core/environ/cocos2d/MainScene.cpp` 第 1762-1815 行

```cpp
bool TVPMainScene::initialize() {
    if (!Scene::init()) return false;
    
    // GameNode — 游戏核心内容（z=0 层级）
    _gameNode = Node::create();
    _gameNode->setName("GameNode");
    addChild(_gameNode, GAME_SCENE_ORDER);     // zOrder = 0
    
    // UINode — UI 系统（z=100 层级）
    _uiNode = Node::create();
    _uiNode->setName("UINode");
    addChild(_uiNode, UI_NODE_ORDER);          // zOrder = 100
    
    // 控制台、窗口管理器等在 GameNode 和 UINode 之间
    // GAME_CONSOLE_ORDER=10, GAME_WINMGR_ORDER=15, GAME_MENU_ORDER=20
    // 都介于 GAME_SCENE_ORDER(0) 和 UI_NODE_ORDER(100) 之间
}
```

节点树结构：

```
TVPMainScene (Scene)
├── GameNode         (z=0)   ← 游戏窗口层（TVPWindowLayer）
│   └── TVPWindowLayer...    ← 承载 KiriKiri 渲染内容
├── Console          (z=10)  ← 调试控制台
├── WindowMgr        (z=15)  ← 窗口管理器覆盖层
├── GameMenu         (z=20)  ← 浮动菜单
└── UINode           (z=100) ← UI 表单栈
    ├── FileSelector  ← pushUIForm() 添加
    ├── Settings      ← pushUIForm() 添加
    └── ...
```

关键设计要点：

1. **大间距 zOrder**：使用 0/10/15/20/100 而非连续整数，便于后续在中间插入新层级
2. **GameNode/UINode 双树**：游戏内容和 UI 完全隔离，UI 始终覆盖在游戏之上
3. **枚举管理**：用枚举代替魔数，所有层级定义集中管理

## 动手实践

### 实践 1：构建一个多层场景

创建一个包含背景、游戏对象和 UI 的三层场景：

```cpp
#include "cocos2d.h"
USING_NS_CC;

class DemoScene : public Scene {
public:
    static DemoScene* create() {
        auto scene = new DemoScene();
        if (scene && scene->init()) {
            scene->autorelease();
            return scene;
        }
        CC_SAFE_DELETE(scene);
        return nullptr;
    }
    
    bool init() override {
        if (!Scene::init()) return false;
        
        auto visibleSize = Director::getInstance()->getVisibleSize();
        
        // 第一层：背景（zOrder = -10）
        auto bg = LayerColor::create(Color4B(50, 50, 80, 255));
        addChild(bg, -10, "Background");
        
        // 第二层：游戏对象（zOrder = 0）
        auto gameLayer = Node::create();
        gameLayer->setName("GameLayer");
        addChild(gameLayer, 0);
        
        // 添加三个精灵，不同 zOrder
        for (int i = 0; i < 3; ++i) {
            auto sprite = Sprite::create("icon.png");
            sprite->setPosition(Vec2(
                visibleSize.width * (0.25f + 0.25f * i),
                visibleSize.height * 0.5f
            ));
            sprite->setName("Sprite_" + std::to_string(i));
            gameLayer->addChild(sprite, i);  // z = 0, 1, 2
            
            // 打印信息
            CCLOG("Added %s at z=%d", sprite->getName().c_str(), i);
        }
        
        // 第三层：UI（zOrder = 10）
        auto uiLayer = Node::create();
        uiLayer->setName("UILayer");
        addChild(uiLayer, 10);
        
        auto label = Label::createWithSystemFont(
            "Score: 0", "Arial", 24);
        label->setPosition(Vec2(
            visibleSize.width * 0.5f,
            visibleSize.height - 30));
        uiLayer->addChild(label, 0, "ScoreLabel");
        
        // 验证节点查找
        auto found = gameLayer->getChildByName("Sprite_1");
        CCLOG("Found: %s (z=%d)",
              found->getName().c_str(),
              found->getLocalZOrder());
        
        return true;
    }
};
```

### 实践 2：动态改变 zOrder

```cpp
// 在 DemoScene 中添加触摸交互
bool init() override {
    // ... 前面的代码 ...
    
    // 添加触摸监听器
    auto listener = EventListenerTouchOneByOne::create();
    listener->onTouchBegan = [this](Touch* touch, Event*) -> bool {
        auto gameLayer = getChildByName("GameLayer");
        if (!gameLayer) return false;
        
        auto& children = gameLayer->getChildren();
        for (auto child : children) {
            // 检测触摸是否在精灵范围内
            auto sprite = dynamic_cast<Sprite*>(child);
            if (sprite) {
                auto localPos = sprite->convertToNodeSpace(
                    touch->getLocation());
                auto size = sprite->getContentSize();
                Rect rect(0, 0, size.width, size.height);
                
                if (rect.containsPoint(localPos)) {
                    // 点击的精灵提升到最顶层
                    int maxZ = 0;
                    for (auto c : children) {
                        maxZ = std::max(maxZ,
                                       c->getLocalZOrder());
                    }
                    sprite->setLocalZOrder(maxZ + 1);
                    CCLOG("Raised %s to z=%d",
                          sprite->getName().c_str(),
                          sprite->getLocalZOrder());
                    return true;
                }
            }
        }
        return false;
    };
    
    _eventDispatcher->addEventListenerWithSceneGraphPriority(
        listener, this);
    
    return true;
}
```

## 本节小结

- **addChild/removeChild** 是节点树操作的核心方法，`cleanup` 参数控制是否清理 Action/Schedule
- **localZOrder** 控制同一父节点下兄弟的渲染顺序，值小的先渲染（在下方）
- **globalZOrder** 打破层级限制，在全场景范围内控制渲染顺序（非零时覆盖 localZOrder）
- **visit()** 按 `zOrder < 0` → 自身 → `zOrder >= 0` 的顺序递归渲染
- **Tag**（整数）和 **Name**（字符串）用于标识和查找子节点，Name 更适合开发调试
- KrKr2 使用 `SCENE_ORDER` 枚举定义大间距 zOrder（0/10/15/20/100），将游戏和 UI 隔离到独立子树

## 练习题与答案

### 题目 1：zOrder 排序推理

给定以下代码，请写出从底到顶的渲染顺序：

```cpp
auto scene = Scene::create();
auto a = Sprite::create("a.png");
auto b = Sprite::create("b.png");
auto c = Sprite::create("c.png");
auto d = Sprite::create("d.png");

scene->addChild(a, 5);    // A
scene->addChild(b, -3);   // B
scene->addChild(c, 5);    // C
scene->addChild(d, 0);    // D
```

<details>
<summary>查看答案</summary>

渲染顺序（从底到顶）：**B → D → A → C**

排序规则：
1. 首先按 `localZOrder` 升序排列：B(-3) < D(0) < A(5) = C(5)
2. A 和 C 的 zOrder 相同（都是 5），按 `addChild` 调用顺序：A 先添加 → A 先渲染
3. 最终顺序：B(-3) → D(0) → A(5) → C(5)

验证代码：
```cpp
// 在 scene 中添加以下代码验证
schedule([=](float) {
    auto& children = scene->getChildren();
    std::string order;
    for (auto child : children) {
        order += child->getName() + "(z=" +
                 std::to_string(child->getLocalZOrder()) + ") ";
    }
    CCLOG("Render order: %s", order.c_str());
    unschedule("debug");
}, "debug");
```

</details>

### 题目 2：修复节点转移 Bug

以下代码试图将 player 从 teamA 转移到 teamB，但会崩溃。找出 Bug 并修复：

```cpp
auto teamA = Node::create();
auto teamB = Node::create();
scene->addChild(teamA);
scene->addChild(teamB);

auto player = Sprite::create("player.png");
teamA->addChild(player, 0, "player");

// 转移到 teamB
teamB->addChild(player, 0, "player");  // 崩溃！
```

<details>
<summary>查看答案</summary>

**Bug 原因：** `player` 已经有父节点 `teamA`，直接 `addChild` 到 `teamB` 会触发断言 `CCASSERT(_parent == nullptr)`。

**修复方案：**

```cpp
auto teamA = Node::create();
auto teamB = Node::create();
scene->addChild(teamA);
scene->addChild(teamB);

auto player = Sprite::create("player.png");
teamA->addChild(player, 0, "player");

// 正确的转移方式
player->retain();             // 防止 removeFromParent 后引用计数归零被释放
player->removeFromParent();   // 从 teamA 移除
teamB->addChild(player, 0, "player");  // 添加到 teamB
player->release();            // 平衡 retain（addChild 已经 retain 了一次）
```

关键要点：
1. 必须先 `retain()` 保持引用，否则 `removeFromParent()` 会导致引用计数归零，对象被释放
2. `removeFromParent()` 默认 cleanup=true，会停止 player 上的所有 Action
3. 如果需要保留 Action，使用 `removeFromParentAndCleanup(false)`
4. 最后 `release()` 平衡 `retain()`，因为 `addChild` 内部会再次 `retain`

</details>

### 题目 3：设计 zOrder 层级体系

参考 KrKr2 的 `SCENE_ORDER` 枚举，为一个 RPG 游戏设计 zOrder 层级体系。要求包含：地图层、角色层、特效层、对话框层、系统 UI 层。解释为什么使用大间距而非连续数字。

<details>
<summary>查看答案</summary>

```cpp
// RPG 游戏 zOrder 层级设计
enum RPG_LAYER_ORDER {
    LAYER_MAP        = 0,      // 地图背景（地形、建筑）
    LAYER_SHADOW     = 5,      // 角色阴影
    LAYER_CHARACTER  = 10,     // 角色层（NPC、玩家、怪物）
    LAYER_EFFECT     = 20,     // 战斗特效、粒子
    LAYER_WEATHER    = 30,     // 天气效果（雨、雪）
    LAYER_DIALOG     = 50,     // 对话框、剧情面板
    LAYER_SYSTEM_UI  = 100,    // 系统 UI（血条、小地图、菜单）
    LAYER_LOADING    = 200,    // 加载画面（覆盖一切）
    LAYER_DEBUG      = 999,    // 调试信息（仅开发时显示）
};

// 使用示例
auto scene = Scene::create();

auto mapLayer = TMXTiledMap::create("town.tmx");
scene->addChild(mapLayer, LAYER_MAP, "MapLayer");

auto charLayer = Node::create();
scene->addChild(charLayer, LAYER_CHARACTER, "CharLayer");

auto effectLayer = Node::create();
scene->addChild(effectLayer, LAYER_EFFECT, "EffectLayer");

auto dialogLayer = Node::create();
scene->addChild(dialogLayer, LAYER_DIALOG, "DialogLayer");

auto uiLayer = Node::create();
scene->addChild(uiLayer, LAYER_SYSTEM_UI, "SystemUI");
```

**使用大间距的原因：**

1. **可扩展性：** 未来如果需要在角色层和特效层之间插入"投射物层"（如箭矢），可以使用 zOrder=15，无需修改已有层级
2. **语义清晰：** 0/10/20/50/100 一看就知道是分层设计，而 0/1/2/3/4 容易混淆
3. **子层级支持：** 每个大层内部还可以用小数值区分，如角色层内 zOrder=10 的 NPC 和 zOrder=11 的玩家
4. **KrKr2 的实践验证：** KrKr2 使用 0/10/15/20/100 的间距，在多年开发中证明了这种设计的可维护性

</details>

## 下一步

[03-场景切换与过渡](03-场景切换与过渡.md) — 学习 Director 的场景切换机制（replaceScene / pushScene / popScene）以及过渡动画效果。

