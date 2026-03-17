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

