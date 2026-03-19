# Action 动画系统

## 本节目标

- 理解 Cocos2d-x Action 系统的设计哲学与类层次结构
- 掌握瞬时动作（ActionInstant）与持续动作（ActionInterval）的区别
- 学会使用组合动作（Sequence、Spawn、Repeat）构建复杂动画
- 理解缓动函数（EaseIn/Out）的数学原理与视觉效果
- 能够在实际项目中用 Action 实现 UI 动效与游戏动画

## 前置知识

- 已阅读 [01-Sprite创建与纹理管理](./01-Sprite创建与纹理管理.md)（了解节点基本操作）
- 已阅读 [02-父子关系与zOrder](../02-场景与节点树/02-父子关系与zOrder.md)（了解节点树更新机制）
- 熟悉 C++ 继承与多态

---

## 1. Action 系统架构概览

### 1.1 为什么需要 Action？

在没有 Action 系统的情况下，实现一个"精灵从左移动到右"需要：

```cpp
// 手动方式——在 update 回调中逐帧更新
void MyScene::update(float dt) {
    auto sprite = getChildByName("hero");
    if (!sprite) return;

    float speed = 200.0f; // 像素/秒
    Vec2 pos = sprite->getPosition();
    pos.x += speed * dt;

    if (pos.x >= 480.0f) {
        pos.x = 480.0f;
        // 移动结束，需要手动清理状态
        unscheduleUpdate();
    }
    sprite->setPosition(pos);
}
```

问题很明显：
1. **状态管理复杂**——需要手动跟踪动画是否完成
2. **难以组合**——多个动画同时或顺序执行时，update 函数会膨胀
3. **无法复用**——每个动画都要手写逻辑

Action 系统将动画抽象为**可组合的对象**：

```cpp
// Action 方式——一行搞定
sprite->runAction(MoveTo::create(2.0f, Vec2(480, 240)));
```

### 1.2 类层次结构

```
Ref
 └── Action                          // 抽象基类
      ├── FiniteTimeAction           // 有限时间动作
      │    ├── ActionInstant         // 瞬时动作（duration = 0）
      │    │    ├── Show / Hide
      │    │    ├── FlipX / FlipY
      │    │    ├── Place
      │    │    ├── CallFunc
      │    │    └── RemoveSelf
      │    └── ActionInterval        // 持续动作（duration > 0）
      │         ├── MoveTo / MoveBy
      │         ├── RotateTo / RotateBy
      │         ├── ScaleTo / ScaleBy
      │         ├── FadeTo / FadeIn / FadeOut
      │         ├── TintTo / TintBy
      │         ├── BezierTo / BezierBy
      │         ├── JumpTo / JumpBy
      │         ├── Sequence          // 顺序组合
      │         ├── Spawn             // 并行组合
      │         ├── Repeat / RepeatForever
      │         ├── DelayTime
      │         └── EaseXxx           // 缓动包装器
      ├── Follow                      // 跟随动作（无限时间）
      └── Speed                       // 速度包装器
```

### 1.3 核心更新机制

Action 的驱动来自 `ActionManager`，它被 Director 的主循环调度：

```
Director::mainLoop()
  → Scheduler::update(dt)
    → ActionManager::update(dt)
      → 遍历所有 target 的 Action 列表
        → action->step(dt)
          → action->update(percent)  // percent = elapsed / duration
```

```cpp
// ActionInterval::step 的核心逻辑（简化版）
void ActionInterval::step(float dt) {
    _elapsed += dt;
    float percent = _elapsed / _duration;
    if (percent > 1.0f) percent = 1.0f;

    this->update(percent);  // 子类实现具体插值逻辑

    if (percent >= 1.0f) {
        // 动作完成，ActionManager 会在下一帧移除
    }
}
```

**关键点**：Action 并不是节点的成员——它由 `ActionManager` 统一管理，通过 `target` 指针关联到节点。

---

## 2. 瞬时动作（ActionInstant）

瞬时动作在一帧内完成，常用于逻辑控制。

### 2.1 常用瞬时动作

```cpp
// Show / Hide——切换可见性
sprite->runAction(Hide::create());
sprite->runAction(Show::create());

// Place——瞬间移动到指定位置
sprite->runAction(Place::create(Vec2(100, 200)));

// FlipX / FlipY——水平/垂直翻转
sprite->runAction(FlipX::create(true));   // 翻转
sprite->runAction(FlipX::create(false));  // 恢复

// RemoveSelf——从父节点移除自身
// 常用于子弹命中后的清理
bullet->runAction(RemoveSelf::create());
```

### 2.2 CallFunc——动作中嵌入回调

`CallFunc` 是最强大的瞬时动作，它让你在动作序列中插入任意 C++ 逻辑：

```cpp
// 基本用法
auto callback = CallFunc::create([]() {
    CCLOG("动画播放完毕！");
});

// 带 Node* sender 参数
auto callWithSender = CallFunc::create([](Node* sender) {
    // sender 就是执行这个 Action 的节点
    sender->removeFromParent();
});

// 在序列中使用——移动完成后打印日志
sprite->runAction(Sequence::create(
    MoveTo::create(1.0f, Vec2(400, 300)),
    CallFunc::create([]() {
        CCLOG("到达目的地！");
    }),
    nullptr  // Sequence 的变参列表必须以 nullptr 结尾
));
```

### 2.3 常见错误：忘记 nullptr 终止

```cpp
// ❌ 错误——Sequence 变参列表没有 nullptr 结尾
// 会导致未定义行为（读取栈上的垃圾值）
sprite->runAction(Sequence::create(
    MoveTo::create(1.0f, Vec2(400, 300)),
    CallFunc::create([]() { CCLOG("done"); })
    // 缺少 nullptr！
));

// ✅ 正确
sprite->runAction(Sequence::create(
    MoveTo::create(1.0f, Vec2(400, 300)),
    CallFunc::create([]() { CCLOG("done"); }),
    nullptr
));

// ✅ 也可以用 Vector 版本避免这个问题
Vector<FiniteTimeAction*> actions;
actions.pushBack(MoveTo::create(1.0f, Vec2(400, 300)));
actions.pushBack(CallFunc::create([]() { CCLOG("done"); }));
sprite->runAction(Sequence::create(actions));
```

---

## 3. 持续动作（ActionInterval）

### 3.1 MoveTo vs MoveBy

Cocos2d-x 的命名惯例：`XxxTo` 表示绝对目标值，`XxxBy` 表示相对增量。

```cpp
// MoveTo——移动到绝对坐标 (400, 300)
// 无论精灵当前在哪里，最终都到 (400, 300)
sprite->runAction(MoveTo::create(2.0f, Vec2(400, 300)));

// MoveBy——相对当前位置偏移 (100, 50)
// 如果当前在 (200, 200)，最终到 (300, 250)
sprite->runAction(MoveBy::create(2.0f, Vec2(100, 50)));
```

**选择原则**：
- 需要到达特定位置 → `MoveTo`
- 需要方向性移动（如"向右走 100 像素"）→ `MoveBy`
- 需要反向动作（reverse）→ 只能用 `MoveBy`

### 3.2 旋转、缩放、透明度

```cpp
// RotateBy——顺时针旋转 360 度，耗时 1 秒
sprite->runAction(RotateBy::create(1.0f, 360.0f));

// ScaleTo——缩放到 2 倍大小
sprite->runAction(ScaleTo::create(0.5f, 2.0f));

// ScaleBy——在当前缩放基础上再放大 1.5 倍
sprite->runAction(ScaleBy::create(0.5f, 1.5f));

// FadeOut——淡出（opacity 从当前值到 0）
sprite->runAction(FadeOut::create(1.0f));

// FadeIn——淡入（opacity 从 0 到 255）
sprite->runAction(FadeIn::create(1.0f));

// FadeTo——淡到指定透明度（0-255）
sprite->runAction(FadeTo::create(1.0f, 128));  // 半透明

// TintTo——变色（RGB）
sprite->runAction(TintTo::create(1.0f, 255, 0, 0));  // 变红
```

### 3.3 贝塞尔曲线与跳跃

```cpp
// BezierTo——沿三次贝塞尔曲线移动
ccBezierConfig bezier;
bezier.controlPoint_1 = Vec2(100, 300);  // 控制点1
bezier.controlPoint_2 = Vec2(300, 300);  // 控制点2
bezier.endPosition    = Vec2(400, 100);  // 终点
sprite->runAction(BezierTo::create(2.0f, bezier));

// JumpTo——跳跃到目标点
// 参数：时间、终点、跳跃高度、跳跃次数
sprite->runAction(JumpTo::create(2.0f, Vec2(400, 100), 80.0f, 3));

// JumpBy——相对跳跃
sprite->runAction(JumpBy::create(1.0f, Vec2(200, 0), 60.0f, 2));
```

### 3.4 update() 插值原理

所有 ActionInterval 子类的核心是 `update(float t)` 方法，`t` 的范围是 `[0, 1]`：

```cpp
// MoveTo::update 的简化实现
void MoveTo::update(float t) {
    if (_target) {
        // 线性插值：startPos + (endPos - startPos) * t
        Vec2 newPos;
        newPos.x = _startPosition.x + _positionDelta.x * t;
        newPos.y = _startPosition.y + _positionDelta.y * t;
        _target->setPosition(newPos);
    }
}

// RotateBy::update
void RotateBy::update(float t) {
    if (_target) {
        _target->setRotation(_startAngle + _diffAngle * t);
    }
}

// FadeTo::update
void FadeTo::update(float t) {
    if (_target) {
        GLubyte opacity = _fromOpacity + (_toOpacity - _fromOpacity) * t;
        _target->setOpacity(opacity);
    }
}
```

**关键洞察**：当 `t` 从 0 线性增长到 1 时，动画是匀速的。缓动函数的作用就是改变 `t` 的增长曲线。

---

## 4. 组合动作

### 4.1 Sequence——顺序执行

```cpp
// 先移动，再旋转，再淡出
auto seq = Sequence::create(
    MoveTo::create(1.0f, Vec2(400, 300)),
    RotateBy::create(0.5f, 360.0f),
    FadeOut::create(0.5f),
    nullptr
);
sprite->runAction(seq);
// 总时长 = 1.0 + 0.5 + 0.5 = 2.0 秒
```

### 4.2 Spawn——并行执行

```cpp
// 同时移动 + 旋转 + 缩放
auto spawn = Spawn::create(
    MoveTo::create(2.0f, Vec2(400, 300)),
    RotateBy::create(2.0f, 720.0f),
    ScaleTo::create(2.0f, 2.0f),
    nullptr
);
sprite->runAction(spawn);
// 总时长 = max(2.0, 2.0, 2.0) = 2.0 秒
```

**注意**：Spawn 中各动作时长不同时，总时长取最大值，短的动作先完成。

### 4.3 Repeat 与 RepeatForever

```cpp
// Repeat——重复指定次数
auto blink = Sequence::create(
    FadeOut::create(0.3f),
    FadeIn::create(0.3f),
    nullptr
);
sprite->runAction(Repeat::create(blink, 5));  // 闪烁 5 次

// RepeatForever——无限重复
auto patrol = Sequence::create(
    MoveBy::create(2.0f, Vec2(200, 0)),
    MoveBy::create(2.0f, Vec2(-200, 0)),
    nullptr
);
sprite->runAction(RepeatForever::create(patrol));  // 永远来回巡逻
```

### 4.4 DelayTime——延迟

```cpp
// 等待 1 秒后再移动
sprite->runAction(Sequence::create(
    DelayTime::create(1.0f),
    MoveTo::create(2.0f, Vec2(400, 300)),
    nullptr
));
```

### 4.5 复杂组合示例：弹幕效果

```cpp
// 子弹从屏幕外飞入 → 到达目标点 → 爆炸效果 → 移除自身
auto bulletAction = Sequence::create(
    // 阶段1：飞行
    Spawn::create(
        MoveTo::create(0.8f, Vec2(targetX, targetY)),
        RotateBy::create(0.8f, 720.0f),
        nullptr
    ),
    // 阶段2：命中爆炸
    Spawn::create(
        ScaleTo::create(0.2f, 3.0f),
        FadeOut::create(0.2f),
        nullptr
    ),
    // 阶段3：清理
    CallFunc::create([this]() {
        // 播放爆炸音效
        AudioEngine::play2d("explosion.mp3");
    }),
    RemoveSelf::create(),
    nullptr
);
bullet->runAction(bulletAction);
```

---

## 5. 缓动函数（Ease Actions）

### 5.1 什么是缓动？

线性动画看起来机械、不自然。现实世界中的运动都有**加速和减速**过程。缓动函数通过改变 `t` 的映射曲线，让动画更自然。

```
线性（无缓动）：  t_out = t
EaseIn（加速）：  t_out = t^n        （开始慢，结束快）
EaseOut（减速）： t_out = 1-(1-t)^n  （开始快，结束慢）
EaseInOut（两端缓）：开始慢→中间快→结束慢
```

### 5.2 常用缓动类型

```cpp
auto move = MoveTo::create(2.0f, Vec2(400, 300));

// Quad（二次方）——轻微缓动
sprite->runAction(EaseQuadraticActionIn::create(move->clone()));

// Cubic（三次方）——中等缓动
sprite->runAction(EaseCubicActionInOut::create(move->clone()));

// Exponential（指数）——强烈缓动
sprite->runAction(EaseExponentialIn::create(move->clone()));
sprite->runAction(EaseExponentialOut::create(move->clone()));

// Elastic（弹性）——像橡皮筋一样弹跳
sprite->runAction(EaseElasticOut::create(move->clone()));

// Bounce（弹跳）——像球落地反弹
sprite->runAction(EaseBounceOut::create(move->clone()));

// Back（回弹）——超过终点后回弹
sprite->runAction(EaseBackOut::create(move->clone()));
```

> **注意**：缓动是**包装器**，它包装一个 ActionInterval 并改变其时间曲线。
> 所以要用 `clone()` 复制原始动作，否则同一个 Action 对象不能被多次使用。

### 5.3 缓动的数学原理

```cpp
// EaseIn 的 update 实现（简化）
void EaseIn::update(float t) {
    // rate 通常为 2.0（二次方）或 3.0（三次方）
    float easedT = powf(t, _rate);
    _inner->update(easedT);  // 用变换后的 t 驱动内部动作
}

// EaseOut 的实现
void EaseOut::update(float t) {
    float easedT = powf(t, 1.0f / _rate);
    _inner->update(easedT);
}

// EaseElasticOut 的实现
void EaseElasticOut::update(float t) {
    if (t == 0 || t == 1) {
        _inner->update(t);
    } else {
        float s = _period / 4.0f;
        float easedT = powf(2, -10 * t)
                      * sinf((t - s) * M_PI * 2.0f / _period)
                      + 1.0f;
        _inner->update(easedT);
    }
}
```

### 5.4 缓动选择指南

| 场景 | 推荐缓动 | 原因 |
|------|-----------|------|
| UI 弹窗出现 | EaseBackOut | 稍微超过目标再回弹，有弹性感 |
| UI 弹窗消失 | EaseBackIn | 先回缩再飞走 |
| 角色跳跃 | EaseQuadraticOut | 模拟重力的抛物线 |
| 物品掉落 | EaseBounceOut | 落地反弹效果 |
| 弹性动效 | EaseElasticOut | 橡皮筋式弹跳 |
| 通用 UI 过渡 | EaseCubicActionInOut | 两端平滑，中间流畅 |
| 匀速运动 | 不使用缓动 | 如匀速滚动的背景 |

---

## 6. Action 管理与生命周期

### 6.1 stopAction 与 stopAllActions

```cpp
// 运行动作并保存 Tag
auto move = MoveTo::create(2.0f, Vec2(400, 300));
move->setTag(100);
sprite->runAction(move);

// 通过 Tag 停止特定动作
sprite->stopActionByTag(100);

// 停止所有动作
sprite->stopAllActions();

// 暂停/恢复节点的所有动作
sprite->pause();    // 暂停动作 + 调度器
sprite->resume();   // 恢复
```

### 6.2 Action 的克隆与反转

```cpp
// clone()——深拷贝一个动作
auto move = MoveBy::create(1.0f, Vec2(100, 0));
auto moveClone = move->clone();

// 同一个 Action 对象不能同时 runAction 到两个节点
// ❌ 错误
sprite1->runAction(move);
sprite2->runAction(move);  // 未定义行为！

// ✅ 正确
sprite1->runAction(move);
sprite2->runAction(move->clone());

// reverse()——反转动作（仅 XxxBy 类支持）
auto moveRight = MoveBy::create(1.0f, Vec2(100, 0));
auto moveLeft  = moveRight->reverse();  // 等价于 MoveBy(-100, 0)

// 来回移动
sprite->runAction(Sequence::create(
    moveRight,
    moveLeft,
    nullptr
));
```

### 6.3 常见错误：Action 复用导致崩溃

```cpp
// ❌ 同一个 Action 对象绑定到多个节点——必崩溃
auto fade = FadeOut::create(1.0f);
for (auto& child : children) {
    child->runAction(fade);  // 第二次 runAction 时 fade 已被管理
}

// ✅ 每次 clone
auto fade = FadeOut::create(1.0f);
for (auto& child : children) {
    child->runAction(fade->clone());
}
```

### 6.4 节点移除时的 Action 清理

```cpp
// 当节点被 removeFromParent 时，ActionManager 自动清理关联的所有 Action
// 不需要手动 stopAllActions

sprite->removeFromParent();  
// 内部调用 ActionManager::removeAllActionsFromTarget(sprite)

// 但如果你在 CallFunc 回调中引用了已移除的节点，会导致野指针
// ❌ 危险
auto seq = Sequence::create(
    DelayTime::create(2.0f),
    CallFunc::create([sprite]() {
        // 如果 sprite 在 2 秒内被其他逻辑移除了，这里就是野指针
        sprite->setPosition(Vec2::ZERO);
    }),
    nullptr
);
someOtherNode->runAction(seq);  // 注意：Action 绑定在 someOtherNode 上

// ✅ 安全——用弱引用或检查
auto seq = Sequence::create(
    DelayTime::create(2.0f),
    CallFunc::create([sprite]() {
        if (sprite && sprite->getParent()) {
            sprite->setPosition(Vec2::ZERO);
        }
    }),
    nullptr
);
```

---

## 7. Speed 包装器

`Speed` 可以动态调整动作的播放速率：

```cpp
auto move = MoveTo::create(4.0f, Vec2(400, 300));
auto speed = Speed::create(move, 1.0f);  // 初始 1x 速度
speed->setTag(200);
sprite->runAction(speed);

// 后续可以动态调整
auto speedAction = dynamic_cast<Speed*>(sprite->getActionByTag(200));
if (speedAction) {
    speedAction->setSpeed(2.0f);   // 2x 加速
    speedAction->setSpeed(0.5f);   // 0.5x 慢放
    speedAction->setSpeed(0.0f);   // 暂停（但动作仍在运行）
}
```

**使用场景**：游戏加速/减速、调试动画。

---

## 动手实践

### 实践1：角色出场动画

实现一个角色从屏幕外飞入、弹跳落地、显示名字的完整出场动画：

```cpp
#include "cocos2d.h"
USING_NS_CC;

class CharacterEntrance : public Scene {
public:
    static Scene* createScene() {
        auto scene = CharacterEntrance::create();
        return scene;
    }

    CREATE_FUNC(CharacterEntrance);

    bool init() override {
        if (!Scene::init()) return false;

        auto visibleSize = Director::getInstance()->getVisibleSize();
        auto origin = Director::getInstance()->getVisibleOrigin();
        float centerX = origin.x + visibleSize.width / 2;
        float groundY = origin.y + 100;

        // 角色精灵——初始位置在屏幕上方
        auto hero = Sprite::create("hero.png");
        hero->setPosition(Vec2(centerX, visibleSize.height + 100));
        hero->setScale(0.5f);
        hero->setOpacity(0);
        this->addChild(hero, 10, "hero");

        // 名字标签——初始不可见
        auto nameLabel = Label::createWithTTF("勇者", "fonts/simhei.ttf", 24);
        nameLabel->setPosition(Vec2(centerX, groundY - 40));
        nameLabel->setOpacity(0);
        nameLabel->setScale(0.0f);
        this->addChild(nameLabel, 10, "name");

        // 出场动画序列
        auto entrance = Sequence::create(
            // 阶段1：淡入 + 从上方落下（弹跳效果）
            Spawn::create(
                FadeIn::create(0.3f),
                EaseBounceOut::create(
                    MoveTo::create(1.5f, Vec2(centerX, groundY))
                ),
                ScaleTo::create(1.5f, 1.0f),
                nullptr
            ),
            // 阶段2：落地震动——快速左右晃动
            Sequence::create(
                MoveBy::create(0.05f, Vec2(-5, 0)),
                MoveBy::create(0.05f, Vec2(10, 0)),
                MoveBy::create(0.05f, Vec2(-10, 0)),
                MoveBy::create(0.05f, Vec2(5, 0)),
                nullptr
            ),
            // 阶段3：显示名字
            CallFunc::create([nameLabel]() {
                nameLabel->runAction(Spawn::create(
                    FadeIn::create(0.5f),
                    EaseBackOut::create(ScaleTo::create(0.5f, 1.0f)),
                    nullptr
                ));
            }),
            nullptr
        );

        hero->runAction(entrance);
        return true;
    }
};
```

### 实践2：无限背景滚动

```cpp
// 实现两张背景图无缝循环滚动
bool ScrollingBg::init() {
    if (!Scene::init()) return false;

    auto visibleSize = Director::getInstance()->getVisibleSize();

    // 两张相同背景首尾相接
    auto bg1 = Sprite::create("background.png");
    bg1->setAnchorPoint(Vec2::ZERO);
    bg1->setPosition(Vec2::ZERO);
    this->addChild(bg1, 0, "bg1");

    auto bg2 = Sprite::create("background.png");
    bg2->setAnchorPoint(Vec2::ZERO);
    bg2->setPosition(Vec2(bg1->getContentSize().width, 0));
    this->addChild(bg2, 0, "bg2");

    float bgWidth = bg1->getContentSize().width;
    float scrollTime = 10.0f;  // 滚动一屏的时间

    // 对两张背景分别执行无限滚动
    auto scrollAction = [bgWidth, scrollTime](Node* bg, float startX) {
        return RepeatForever::create(
            Sequence::create(
                Place::create(Vec2(startX, 0)),
                MoveBy::create(scrollTime, Vec2(-bgWidth, 0)),
                nullptr
            )
        );
    };

    bg1->runAction(scrollAction(bg1, 0));
    bg2->runAction(scrollAction(bg2, bgWidth));

    return true;
}
```

### 实践3：UI 按钮点击反馈

```cpp
// 点击按钮时的缩放反馈动画
void addButtonFeedback(Node* button) {
    auto listener = EventListenerTouchOneByOne::create();
    listener->setSwallowTouches(true);

    listener->onTouchBegan = [button](Touch* touch, Event* event) -> bool {
        auto bounds = button->getBoundingBox();
        if (bounds.containsPoint(touch->getLocation())) {
            // 按下——缩小
            button->stopAllActions();
            button->runAction(
                EaseBackOut::create(ScaleTo::create(0.1f, 0.9f))
            );
            return true;
        }
        return false;
    };

    listener->onTouchEnded = [button](Touch* touch, Event* event) {
        // 松手——弹回原尺寸
        button->stopAllActions();
        button->runAction(
            EaseElasticOut::create(ScaleTo::create(0.3f, 1.0f))
        );
    };

    listener->onTouchCancelled = [button](Touch* touch, Event* event) {
        button->stopAllActions();
        button->runAction(ScaleTo::create(0.1f, 1.0f));
    };

    button->getEventDispatcher()->addEventListenerWithSceneGraphPriority(
        listener, button
    );
}
```

---

## 对照项目源码

在 KrKr2 项目中，Action 被广泛用于 UI 动效。以下是 `MainScene.cpp` 中的实际用法：

### UI 表单推入/弹出动画

```cpp
// MainScene.cpp 第 1850-1900 行附近
// pushUIForm 中的滑入动画
void TVPMainScene::pushUIForm(iBaseForm* form, ...) {
    // 表单从右侧滑入
    Node* formNode = form->getNode();
    auto visibleSize = Director::getInstance()->getVisibleSize();

    formNode->setPositionX(visibleSize.width);  // 初始在屏幕右侧
    _uiNode->addChild(formNode);

    // 使用 EaseCubicActionOut 实现平滑滑入
    formNode->runAction(
        EaseCubicActionOut::create(
            MoveTo::create(0.3f, Vec2::ZERO)
        )
    );
}

// popUIForm 中的滑出动画
void TVPMainScene::popUIForm() {
    auto topForm = _formStack.back();
    Node* formNode = topForm->getNode();
    auto visibleSize = Director::getInstance()->getVisibleSize();

    formNode->runAction(Sequence::create(
        EaseCubicActionIn::create(
            MoveTo::create(0.3f, Vec2(visibleSize.width, 0))
        ),
        CallFunc::create([topForm, this]() {
            topForm->getNode()->removeFromParent();
            _formStack.popBack();
        }),
        nullptr
    ));
}
```

**分析**：
- 推入用 `EaseCubicActionOut`（开始快→结束慢），符合"飞入后减速停稳"的直觉
- 弹出用 `EaseCubicActionIn`（开始慢→结束快），符合"加速飞出"的直觉
- 弹出完成后用 `CallFunc` 清理节点——这是典型的 Action 生命周期管理模式

---

## 常见错误及解决方案

### 错误 1：Action 执行完后节点位置异常

使用 `MoveBy` 移动后发现节点位置不是预期值。原因通常是同一节点上同时运行了多个互相冲突的 Action（如两个 `MoveTo`），或者 Action 的 `duration` 为 0 导致瞬间完成但状态未正确更新。

**解决：** 运行新 Action 前用 `node->stopAllActions()` 清除旧 Action，或用 `node->stopActionByTag(tag)` 精确停止。

### 错误 2：Sequence 中忘记 nullptr 结尾

`Sequence::create(action1, action2)` 不加 `nullptr` 结尾，导致读取栈上的垃圾数据，运行时崩溃或行为异常。

**解决：** 始终使用 `Sequence::create(action1, action2, nullptr)` 或 `Sequence::createWithTwoActions(a, b)`。Cocos2d-x 4.0 的变参列表必须以 `nullptr` 结尾。

### 错误 3：RepeatForever 的 Action 被复用

将同一个 Action 对象传给多个节点的 `runAction`，导致 Action 的 target 被覆盖、状态混乱。

**解决：** 每个节点需要独立的 Action 实例，用 `action->clone()` 创建副本。

## 本节小结

| 概念 | 说明 |
|------|------|
| **ActionInstant** | 瞬时完成：Show/Hide、CallFunc、RemoveSelf 等 |
| **ActionInterval** | 持续动作：MoveTo/By、RotateTo/By、FadeTo 等 |
| **Sequence** | 顺序执行多个动作，变参以 nullptr 结尾 |
| **Spawn** | 并行执行多个动作，总时长取最大值 |
| **Repeat/Forever** | 重复执行，Forever 为无限循环 |
| **Ease 缓动** | 包装器，改变 t 的增长曲线使动画更自然 |
| **clone()** | 同一 Action 不能多次 runAction，必须克隆 |
| **reverse()** | 仅 XxxBy 类支持，生成反向动作 |
| **Speed** | 动态调整动作播放速率 |
| **ActionManager** | Director 驱动，统一管理所有 Action 的更新与清理 |

---

## 练习题与答案

### 练习1：实现呼吸灯效果

**题目**：让一个精灵的透明度在 100 和 255 之间循环变化，模拟呼吸灯效果，单次渐变时间 1.5 秒。

<details>
<summary>参考答案</summary>

```cpp
auto breathe = RepeatForever::create(
    Sequence::create(
        FadeTo::create(1.5f, 100),
        FadeTo::create(1.5f, 255),
        nullptr
    )
);
sprite->setOpacity(255);
sprite->runAction(breathe);
```

使用 `EaseSineInOut` 可以让效果更自然：

```cpp
auto breathe = RepeatForever::create(
    Sequence::create(
        EaseSineInOut::create(FadeTo::create(1.5f, 100)),
        EaseSineInOut::create(FadeTo::create(1.5f, 255)),
        nullptr
    )
);
```

</details>

### 练习2：连锁爆炸

**题目**：有 5 个精灵排成一排，从左到右依次爆炸（放大 + 淡出 + 移除），每个间隔 0.3 秒。

<details>
<summary>参考答案</summary>

```cpp
Vector<Node*> sprites;
// 假设 sprites 已经按从左到右排好

for (int i = 0; i < sprites.size(); i++) {
    auto sprite = sprites.at(i);
    float delay = i * 0.3f;

    auto explode = Sequence::create(
        DelayTime::create(delay),
        Spawn::create(
            EaseExponentialOut::create(ScaleTo::create(0.4f, 3.0f)),
            FadeOut::create(0.4f),
            nullptr
        ),
        RemoveSelf::create(),
        nullptr
    );
    sprite->runAction(explode);
}
```

</details>

### 练习3：找出 Bug

**题目**：以下代码想让 10 个精灵同时执行移动动画，但运行时只有最后一个精灵在移动，其余精灵瞬间到达终点。找出 Bug 并修复。

```cpp
auto move = EaseCubicActionInOut::create(
    MoveTo::create(2.0f, Vec2(400, 300))
);
for (auto& sprite : sprites) {
    sprite->runAction(move);
}
```

<details>
<summary>参考答案</summary>

**Bug**：同一个 Action 对象被多次 `runAction`。Action 对象内部维护了 `_elapsed`、`_target` 等状态，复用会导致状态混乱。

**修复**：每次循环都 `clone()`：

```cpp
auto move = EaseCubicActionInOut::create(
    MoveTo::create(2.0f, Vec2(400, 300))
);
for (auto& sprite : sprites) {
    sprite->runAction(move->clone());
}
```

</details>

---

## 下一步

下一节 [01-Label-Button-Layout基础](../04-UI组件与布局/01-Label-Button-Layout基础.md) 将学习 Cocos2d-x 的 UI 组件体系，包括文本渲染（Label）、按钮（Button）和布局容器（Layout），这些组件正是 Action 动画最常见的应用目标。
