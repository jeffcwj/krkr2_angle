# Label、Button、Layout 基础

## 本节目标

- 理解 Cocos2d-x UI 组件体系（ui::Widget 继承树）与普通 Node 的区别
- 掌握三种 Label 类型（TTF、BMFont、SystemFont）的适用场景与性能差异
- 学会使用 ui::Button 的九宫格缩放与点击事件绑定
- 理解 ui::Layout 的四种布局模式（NONE、LINEAR_HORIZONTAL、LINEAR_VERTICAL、RELATIVE）
- 能够用纯代码搭建一个完整的 UI 面板

## 前置知识

- 已阅读 [03-Action动画系统](../03-精灵与动画/03-Action动画系统.md)（了解 UI 动效基础）
- 已阅读 [02-Scene-Layer-Node树](../01-Cocos2d-x架构/02-Scene-Layer-Node树.md)
- 熟悉 C++ 继承与虚函数

---

## 1. UI 组件体系概览

### 1.1 Widget vs Node

Cocos2d-x 有两套节点体系：

```
Node（基础节点）
 ├── Sprite
 ├── Label
 ├── Scene / Layer
 └── ...

ui::Widget（UI 节点，继承自 ProtectedNode → Node）
 ├── ui::Layout
 ├── ui::Button
 ├── ui::Text        （对 Label 的 Widget 封装）
 ├── ui::ImageView   （对 Sprite 的 Widget 封装）
 ├── ui::TextField
 ├── ui::CheckBox
 ├── ui::Slider
 ├── ui::ScrollView
 ├── ui::ListView
 └── ui::PageView
```

**关键区别**：

| 特性 | Node/Sprite/Label | ui::Widget 系列 |
|------|--------------------|-----------------|
| 触摸事件 | 手动添加 EventListener | 内置 `addTouchEventListener` |
| 焦点系统 | 无 | 支持焦点导航（手柄/键盘）|
| 裁剪 | 需要 ClippingNode | Layout 内置裁剪支持 |
| 布局管理 | 手动计算坐标 | Layout 自动排列 |
| Cocos Studio | 不直接支持 | CSB 文件原生支持 |
| 大小变化通知 | 无 | `onSizeChanged` 回调 |
| 性能 | 更轻量 | 稍重（多了触摸/焦点/布局计算）|

**选择原则**：
- 纯展示（游戏场景中的精灵、特效） → 用 Node/Sprite/Label
- 需要交互（按钮、输入框、列表） → 用 ui::Widget 系列
- 需要布局管理（表单、面板） → 用 ui::Layout

### 1.2 ProtectedNode 是什么？

`ProtectedNode` 是 Widget 的直接父类，它区分了"受保护子节点"和"普通子节点"：

```cpp
class ProtectedNode : public Node {
    // 普通子节点——用户添加的
    void addChild(Node* child) override;

    // 受保护子节点——Widget 内部使用（如 Button 的背景图）
    void addProtectedChild(Node* child);
};
```

**为什么需要这个？** Button 内部有背景图、标题文字等子节点，这些不应该被 `getChildren()` 遍历到，也不应该被用户的 `removeAllChildren()` 误删。

---

## 2. Label——文本渲染

### 2.1 三种创建方式

```cpp
// 方式1：TTF 字体（矢量字体，最常用）
auto ttfLabel = Label::createWithTTF(
    "你好世界",              // 文本内容
    "fonts/simhei.ttf",     // 字体文件路径
    24                       // 字号（像素）
);

// 方式2：BMFont 位图字体（最高性能）
auto bmLabel = Label::createWithBMFont(
    "fonts/arial.fnt",       // .fnt 描述文件
    "Score: 12345"           // 只支持 .fnt 中定义的字符
);

// 方式3：系统字体（无需字体文件，但不跨平台一致）
auto sysLabel = Label::createWithSystemFont(
    "系统字体",
    "Arial",                 // 系统字体名
    24
);
```

### 2.2 性能对比

| 类型 | 渲染方式 | 中文支持 | 性能 | 适用场景 |
|------|----------|----------|------|----------|
| TTF | GPU 纹理缓存 | ✅ 完整 | 中等 | 通用文本、对话框 |
| BMFont | 预渲染纹理图集 | ⚠️ 需工具生成 | 最高 | 数字、分数、计时器 |
| SystemFont | 系统 API 渲染 | ✅ 完整 | 最低 | 调试文本、原型 |

**BMFont 为什么快？** 每个字符都是纹理图集中的一个矩形区域，渲染时只需要查表定位 UV 坐标，没有字体解析和光栅化开销。但需要预先用工具（如 BMFont、Hiero）生成。

### 2.3 TTF Label 高级配置

```cpp
// 使用 TTFConfig 进行精细控制
TTFConfig config;
config.fontFilePath = "fonts/simhei.ttf";
config.fontSize = 32;
config.outlineSize = 2;        // 描边宽度（0=无描边）
config.glyphs = GlyphCollection::DYNAMIC;  // 动态字形缓存
config.customGlyphs = nullptr;

auto label = Label::createWithTTF(config, "描边文字");
label->setTextColor(Color4B::WHITE);
label->enableOutline(Color4B::BLACK, 2);  // 黑色描边

// 阴影
label->enableShadow(
    Color4B(0, 0, 0, 128),    // 阴影颜色
    Size(2, -2),                // 偏移
    3                           // 模糊半径
);

// 发光（仅 TTF）
label->enableGlow(Color4B::YELLOW);

// 换行与对齐
label->setDimensions(300, 0);  // 宽度 300，高度自适应
label->setAlignment(TextHAlignment::CENTER, TextVAlignment::TOP);
label->setOverflow(Label::Overflow::SHRINK);  // 文本过长时缩小字号
```

### 2.4 Label 文本更新

```cpp
auto scoreLabel = Label::createWithTTF("Score: 0", "fonts/mono.ttf", 20);

// 更新文本——直接 setString
void updateScore(int score) {
    // ⚠️ 每次 setString 对 TTF Label 会触发纹理重建
    // 频繁更新（如每帧）时考虑用 BMFont
    scoreLabel->setString(StringUtils::format("Score: %d", score));
}
```

### 2.5 常见错误：字体文件路径

```cpp
// ❌ 错误——字体文件不存在，Label 创建返回 nullptr
auto label = Label::createWithTTF("text", "nonexistent.ttf", 24);
// label 是 nullptr，后续操作崩溃

// ✅ 正确——始终检查返回值
auto label = Label::createWithTTF("text", "fonts/simhei.ttf", 24);
if (!label) {
    CCLOG("ERROR: Failed to create TTF label, check font path");
    return;
}
```

---

## 3. ui::Button

### 3.1 创建与基本用法

```cpp
#include "ui/CocosGUI.h"
using namespace cocos2d::ui;

// 创建按钮（三态图片：正常、按下、禁用）
auto btn = Button::create(
    "btn_normal.png",     // 正常状态
    "btn_pressed.png",    // 按下状态（可选，传 "" 表示不设置）
    "btn_disabled.png"    // 禁用状态（可选）
);
btn->setPosition(Vec2(240, 160));
btn->setTitleText("开始游戏");
btn->setTitleFontName("fonts/simhei.ttf");
btn->setTitleFontSize(20);
btn->setTitleColor(Color3B::WHITE);

// 绑定点击事件
btn->addTouchEventListener([](Ref* sender, Widget::TouchEventType type) {
    switch (type) {
        case Widget::TouchEventType::BEGAN:
            CCLOG("按钮按下");
            break;
        case Widget::TouchEventType::MOVED:
            CCLOG("手指移动");
            break;
        case Widget::TouchEventType::ENDED:
            CCLOG("按钮点击！执行逻辑");
            break;
        case Widget::TouchEventType::CANCELED:
            CCLOG("触摸取消");
            break;
    }
});

// 只关心点击完成的简写
btn->addClickEventListener([](Ref* sender) {
    CCLOG("点击！");
});

this->addChild(btn);
```

### 3.2 九宫格缩放（Scale9）

普通图片拉伸时四角会变形。九宫格将图片分为 9 个区域，四角保持不变，四边单向拉伸，中心双向拉伸：

```
┌───┬──────┬───┐
│ 1 │  2   │ 3 │  ← 1,3 角不变；2 水平拉伸
├───┼──────┼───┤
│ 4 │  5   │ 6 │  ← 4,6 垂直拉伸；5 双向拉伸
├───┼──────┼───┤
│ 7 │  8   │ 9 │  ← 7,9 角不变；8 水平拉伸
└───┴──────┴───┘
```

```cpp
// 启用九宫格
auto btn = Button::create("btn_bg.png");
btn->setScale9Enabled(true);

// 设置九宫格边距（左、上、右、下）
// 假设按钮图片是 100x50，四角各保留 10 像素
btn->setCapInsets(Rect(10, 10, 80, 30));

// 现在可以任意设置大小而不变形
btn->setContentSize(Size(300, 60));
```

### 3.3 Button 状态管理

```cpp
// 禁用按钮
btn->setEnabled(false);     // 不可点击
btn->setBright(false);      // 视觉变暗（与 setEnabled 独立）

// 通常两个一起用
btn->setEnabled(false);
btn->setBright(false);

// 恢复
btn->setEnabled(true);
btn->setBright(true);

// 按下时缩放效果（内置）
btn->setPressedActionEnabled(true);  // 按下时自动缩小到 0.9x
```

---

## 4. ui::Layout——布局容器

### 4.1 四种布局模式

```cpp
#include "ui/CocosGUI.h"
using namespace cocos2d::ui;

// 创建 Layout
auto layout = Layout::create();
layout->setContentSize(Size(400, 300));
layout->setPosition(Vec2(80, 70));

// 背景色（用于调试布局）
layout->setBackGroundColorType(Layout::BackGroundColorType::SOLID);
layout->setBackGroundColor(Color3B(50, 50, 50));
layout->setBackGroundColorOpacity(200);

// 设置布局模式
layout->setLayoutType(Layout::Type::VERTICAL);  // 垂直排列
```

四种模式对比：

```
NONE（默认）           LINEAR_HORIZONTAL      LINEAR_VERTICAL
手动坐标定位           子节点从左到右排列      子节点从上到下排列
┌──────────┐          ┌──────────┐           ┌──────────┐
│   ●      │          │ ● ● ● ● │           │    ●     │
│     ●    │          │          │           │    ●     │
│  ●       │          │          │           │    ●     │
│       ●  │          │          │           │    ●     │
└──────────┘          └──────────┘           └──────────┘

RELATIVE
相对布局——子节点相对于父容器或兄弟节点定位
```

### 4.2 LinearLayoutParameter——间距与对齐

```cpp
auto layout = Layout::create();
layout->setLayoutType(Layout::Type::VERTICAL);
layout->setContentSize(Size(300, 400));

for (int i = 0; i < 4; i++) {
    auto btn = Button::create("btn.png");
    btn->setTitleText(StringUtils::format("按钮 %d", i + 1));

    // 线性布局参数
    auto param = LinearLayoutParameter::create();
    param->setGravity(
        LinearLayoutParameter::LinearGravity::CENTER_HORIZONTAL
    );
    param->setMargin(Margin(0, 10, 0, 10));  // 上下间距 10
    btn->setLayoutParameter(param);

    layout->addChild(btn);
}

this->addChild(layout);
```

### 4.3 RelativeLayoutParameter——相对定位

```cpp
auto layout = Layout::create();
layout->setLayoutType(Layout::Type::RELATIVE);
layout->setContentSize(Size(400, 300));

// 居中的标题
auto title = Text::create("设置", "fonts/simhei.ttf", 28);
auto titleParam = RelativeLayoutParameter::create();
titleParam->setAlign(
    RelativeLayoutParameter::RelativeAlign::CENTER_IN_PARENT
);
titleParam->setRelativeName("title");
title->setLayoutParameter(titleParam);
layout->addChild(title);

// 位于标题下方的按钮
auto confirmBtn = Button::create("btn.png");
confirmBtn->setTitleText("确认");
auto btnParam = RelativeLayoutParameter::create();
btnParam->setAlign(
    RelativeLayoutParameter::RelativeAlign::LOCATION_BELOW_CENTER
);
btnParam->setRelativeToWidgetName("title");  // 相对于 title
btnParam->setMargin(Margin(0, 20, 0, 0));
confirmBtn->setLayoutParameter(btnParam);
layout->addChild(confirmBtn);
```

### 4.4 裁剪与滚动

```cpp
// 启用裁剪——超出 Layout 边界的子节点不显示
layout->setClippingEnabled(true);
layout->setClippingType(Layout::ClippingType::SCISSOR);
// SCISSOR：GPU 裁剪，性能好但只支持矩形
// STENCIL：模板裁剪，支持任意形状但性能较差
```

如果需要滚动，用 `ui::ScrollView`（它继承自 Layout）：

```cpp
auto scrollView = ScrollView::create();
scrollView->setContentSize(Size(300, 200));   // 可视区域
scrollView->setInnerContainerSize(Size(300, 800));  // 内容区域
scrollView->setDirection(ScrollView::Direction::VERTICAL);
scrollView->setBounceEnabled(true);  // 弹性回弹

// 添加内容到 scrollView
for (int i = 0; i < 20; i++) {
    auto item = Text::create(
        StringUtils::format("Item %d", i),
        "fonts/simhei.ttf", 20
    );
    item->setPosition(Vec2(150, 780 - i * 40));
    scrollView->addChild(item);
}
```

---

## 5. ui::Text 与 ui::ImageView

这两个是 Label 和 Sprite 的 Widget 封装版本，支持触摸事件和布局参数：

```cpp
// ui::Text——Widget 版 Label
auto text = Text::create("Hello", "fonts/simhei.ttf", 24);
text->setTextColor(Color4B::YELLOW);
text->setTouchEnabled(true);
text->addClickEventListener([](Ref*) {
    CCLOG("文字被点击");
});

// ui::ImageView——Widget 版 Sprite
auto img = ImageView::create("icon.png");
img->setScale9Enabled(true);
img->setCapInsets(Rect(5, 5, 40, 40));
img->setContentSize(Size(100, 100));
img->setTouchEnabled(true);
```

**何时用 Text vs Label？**
- 在 Layout 中需要自动排列 → 用 `Text`
- 在 ScrollView/ListView 中 → 用 `Text`
- 纯展示、不需要布局管理 → 用 `Label`（更轻量）

---

## 动手实践

### 实践1：设置面板

```cpp
#include "cocos2d.h"
#include "ui/CocosGUI.h"
USING_NS_CC;
using namespace cocos2d::ui;

class SettingsPanel : public Scene {
public:
    CREATE_FUNC(SettingsPanel);

    bool init() override {
        if (!Scene::init()) return false;

        auto visibleSize = Director::getInstance()->getVisibleSize();
        auto origin = Director::getInstance()->getVisibleOrigin();

        // 半透明遮罩
        auto mask = LayerColor::create(Color4B(0, 0, 0, 150));
        this->addChild(mask, 0);

        // 面板容器
        auto panel = Layout::create();
        panel->setContentSize(Size(360, 400));
        panel->setPosition(Vec2(
            origin.x + (visibleSize.width - 360) / 2,
            origin.y + (visibleSize.height - 400) / 2
        ));
        panel->setBackGroundColorType(Layout::BackGroundColorType::SOLID);
        panel->setBackGroundColor(Color3B(40, 40, 60));
        panel->setBackGroundColorOpacity(240);
        panel->setLayoutType(Layout::Type::VERTICAL);
        this->addChild(panel, 1);

        // 标题
        auto title = Text::create("游戏设置", "fonts/simhei.ttf", 28);
        title->setTextColor(Color4B::WHITE);
        auto titleParam = LinearLayoutParameter::create();
        titleParam->setGravity(
            LinearLayoutParameter::LinearGravity::CENTER_HORIZONTAL
        );
        titleParam->setMargin(Margin(0, 20, 0, 20));
        title->setLayoutParameter(titleParam);
        panel->addChild(title);

        // 分隔线
        auto line = ImageView::create("white_pixel.png");
        line->setScale9Enabled(true);
        line->setContentSize(Size(320, 1));
        line->setColor(Color3B(100, 100, 120));
        auto lineParam = LinearLayoutParameter::create();
        lineParam->setGravity(
            LinearLayoutParameter::LinearGravity::CENTER_HORIZONTAL
        );
        lineParam->setMargin(Margin(0, 0, 0, 15));
        line->setLayoutParameter(lineParam);
        panel->addChild(line);

        // 设置项：音量滑块
        addSliderRow(panel, "音乐音量", 80);
        addSliderRow(panel, "音效音量", 100);

        // 底部按钮行
        auto btnRow = Layout::create();
        btnRow->setContentSize(Size(360, 60));
        btnRow->setLayoutType(Layout::Type::HORIZONTAL);
        auto btnRowParam = LinearLayoutParameter::create();
        btnRowParam->setGravity(
            LinearLayoutParameter::LinearGravity::CENTER_HORIZONTAL
        );
        btnRowParam->setMargin(Margin(0, 30, 0, 0));
        btnRow->setLayoutParameter(btnRowParam);

        auto confirmBtn = createStyledButton("确认", Color3B(60, 160, 60));
        auto cancelBtn  = createStyledButton("取消", Color3B(160, 60, 60));

        auto p1 = LinearLayoutParameter::create();
        p1->setMargin(Margin(40, 10, 20, 10));
        confirmBtn->setLayoutParameter(p1);

        auto p2 = LinearLayoutParameter::create();
        p2->setMargin(Margin(20, 10, 40, 10));
        cancelBtn->setLayoutParameter(p2);

        btnRow->addChild(confirmBtn);
        btnRow->addChild(cancelBtn);
        panel->addChild(btnRow);

        return true;
    }

private:
    void addSliderRow(Layout* parent, const std::string& label, int value) {
        auto row = Layout::create();
        row->setContentSize(Size(320, 40));
        row->setLayoutType(Layout::Type::HORIZONTAL);

        auto text = Text::create(label, "fonts/simhei.ttf", 18);
        text->setTextColor(Color4B(200, 200, 200, 255));
        auto tp = LinearLayoutParameter::create();
        tp->setMargin(Margin(20, 10, 15, 10));
        text->setLayoutParameter(tp);

        auto slider = Slider::create();
        slider->loadBarTexture("slider_bg.png");
        slider->loadProgressBarTexture("slider_progress.png");
        slider->loadSlidBallTextures(
            "slider_ball.png", "slider_ball.png", ""
        );
        slider->setPercent(value);
        slider->addEventListener([label](Ref* sender, Slider::EventType type) {
            if (type == Slider::EventType::ON_PERCENTAGE_CHANGED) {
                auto s = dynamic_cast<Slider*>(sender);
                CCLOG("%s: %d%%", label.c_str(), s->getPercent());
            }
        });
        auto sp = LinearLayoutParameter::create();
        sp->setMargin(Margin(0, 12, 20, 12));
        slider->setLayoutParameter(sp);

        row->addChild(text);
        row->addChild(slider);

        auto rowParam = LinearLayoutParameter::create();
        rowParam->setGravity(
            LinearLayoutParameter::LinearGravity::CENTER_HORIZONTAL
        );
        rowParam->setMargin(Margin(0, 5, 0, 5));
        row->setLayoutParameter(rowParam);
        parent->addChild(row);
    }

    Button* createStyledButton(const std::string& title, Color3B color) {
        auto btn = Button::create("btn_bg.png");
        btn->setScale9Enabled(true);
        btn->setCapInsets(Rect(8, 8, 34, 34));
        btn->setContentSize(Size(120, 40));
        btn->setTitleText(title);
        btn->setTitleFontName("fonts/simhei.ttf");
        btn->setTitleFontSize(18);
        btn->setColor(color);
        btn->setPressedActionEnabled(true);
        return btn;
    }
};
```

### 实践2：动态列表

```cpp
// 使用 ListView 创建可滚动列表
bool DynamicList::init() {
    if (!Scene::init()) return false;

    auto listView = ListView::create();
    listView->setContentSize(Size(300, 400));
    listView->setPosition(Vec2(90, 40));
    listView->setDirection(ScrollView::Direction::VERTICAL);
    listView->setBounceEnabled(true);
    listView->setGravity(ListView::Gravity::CENTER_HORIZONTAL);
    listView->setItemsMargin(8.0f);  // 每项间距 8px

    // 添加 30 个列表项
    for (int i = 0; i < 30; i++) {
        auto item = Layout::create();
        item->setContentSize(Size(280, 50));
        item->setBackGroundColorType(Layout::BackGroundColorType::SOLID);
        item->setBackGroundColor(
            (i % 2 == 0) ? Color3B(60, 60, 80) : Color3B(50, 50, 70)
        );
        item->setBackGroundColorOpacity(200);
        item->setTouchEnabled(true);

        auto text = Text::create(
            StringUtils::format("第 %d 项", i + 1),
            "fonts/simhei.ttf", 18
        );
        text->setPosition(Vec2(140, 25));
        item->addChild(text);

        // 点击反馈
        item->addClickEventListener([i](Ref*) {
            CCLOG("选中第 %d 项", i + 1);
        });

        listView->pushBackCustomItem(item);
    }

    // 滚动事件
    listView->addEventListener([](Ref* sender, ListView::EventType type) {
        if (type == ListView::EventType::ON_SELECTED_ITEM_END) {
            auto lv = dynamic_cast<ListView*>(sender);
            CCLOG("选中索引: %ld", lv->getCurSelectedIndex());
        }
    });

    this->addChild(listView);
    return true;
}
```

### 实践3：自适应文本框

```cpp
// Label 自动换行 + 字号缩放
auto createAdaptiveLabel(const std::string& content, float maxWidth,
                          float maxHeight, float baseFontSize) -> Label* {
    auto label = Label::createWithTTF(content, "fonts/simhei.ttf", baseFontSize);
    if (!label) return nullptr;

    // 设置最大宽度，启用自动换行
    label->setDimensions(maxWidth, 0);
    label->setAlignment(TextHAlignment::LEFT, TextVAlignment::TOP);

    // 如果文本高度超出限制，缩小字号
    while (label->getContentSize().height > maxHeight && baseFontSize > 10) {
        baseFontSize -= 2;
        label->setSystemFontSize(baseFontSize);

        // TTF 需要重新设置 TTFConfig
        TTFConfig config = label->getTTFConfig();
        config.fontSize = baseFontSize;
        label->setTTFConfig(config);
    }

    // 仍然超出则启用 SHRINK 模式
    if (label->getContentSize().height > maxHeight) {
        label->setOverflow(Label::Overflow::SHRINK);
        label->setDimensions(maxWidth, maxHeight);
    }

    return label;
}
```

---

## 对照项目源码

KrKr2 项目中 UI 组件的使用集中在 `BaseForm.h` 和 `CsdUIFactory.h`。

### BaseForm 的布局结构

```cpp
// BaseForm.h 第 80-120 行
// 每个 UI 表单都有标准三段式布局
class iTVPBaseForm {
    // NaviBar（顶部导航栏）——固定高度
    Node* _navibar;

    // Body（主体内容区）——填充剩余空间
    Node* _body;

    // Footer（底部工具栏）——可选，固定高度
    Node* _footer;
};

// 布局计算——BaseForm.cpp
void iTVPBaseForm::rearrangeLayout() {
    auto size = getNode()->getContentSize();

    // NaviBar 固定在顶部
    if (_navibar) {
        _navibar->setPosition(Vec2(0, size.height - _navibarHeight));
        _navibar->setContentSize(Size(size.width, _navibarHeight));
    }

    // Footer 固定在底部
    float footerH = _footer ? _footerHeight : 0;
    if (_footer) {
        _footer->setPosition(Vec2::ZERO);
        _footer->setContentSize(Size(size.width, footerH));
    }

    // Body 填充中间
    float bodyH = size.height - _navibarHeight - footerH;
    if (_body) {
        _body->setPosition(Vec2(0, footerH));
        _body->setContentSize(Size(size.width, bodyH));
    }
}
```

### CsdUIFactory 的纯代码 UI 创建

```cpp
// CsdUIFactory.h 中用纯 C++ 创建 UI（无 CSB 文件）
// 这是当 Cocos Studio 不可用时的备选方案

// 创建一个标准按钮
static Button* createDefaultButton(const std::string& title) {
    auto btn = Button::create();
    // 不使用图片，使用颜色背景
    btn->setScale9Enabled(true);
    btn->loadTextures("ui_btn_normal.png", "ui_btn_pressed.png", "");
    btn->setTitleText(title);
    btn->setTitleFontSize(16);
    btn->setPressedActionEnabled(true);
    return btn;
}
```

**关键观察**：KrKr2 同时使用了 CSB 文件（Cocos Studio 导出）和纯代码创建 UI，`CsdUIFactory.h`（1097 行）作为后备方案，用纯代码复刻了所有 CSB 文件的 UI 结构。这是一种常见的防御性设计——当资源文件损坏或平台不支持时仍能工作。

---

## 本节小结

| 概念 | 说明 |
|------|------|
| **Widget vs Node** | Widget 内置触摸、焦点、布局；Node 更轻量 |
| **Label (TTF)** | 矢量字体，支持中文，通用首选 |
| **Label (BMFont)** | 预渲染纹理，性能最高，适合数字/分数 |
| **Button** | 三态图片 + 点击事件 + 九宫格缩放 |
| **Layout** | 四种模式：NONE/H/V/RELATIVE |
| **LinearLayoutParameter** | 控制子节点在线性布局中的对齐和边距 |
| **RelativeLayoutParameter** | 子节点相对于父容器或兄弟节点定位 |
| **Scale9** | 九宫格缩放，四角不变形 |
| **ScrollView/ListView** | 可滚动容器，继承自 Layout |
| **Text/ImageView** | Label/Sprite 的 Widget 封装版本 |

---

## 练习题与答案

### 练习1：实现 Tab 切换

**题目**：创建一个顶部有 3 个 Tab 按钮（"基本"、"高级"、"关于"）的面板，点击 Tab 切换下方内容区域的显示。

<details>
<summary>参考答案</summary>

```cpp
bool TabPanel::init() {
    if (!Scene::init()) return false;

    // Tab 栏
    auto tabBar = Layout::create();
    tabBar->setContentSize(Size(480, 50));
    tabBar->setPosition(Vec2(0, 270));
    tabBar->setLayoutType(Layout::Type::HORIZONTAL);
    this->addChild(tabBar);

    // 内容面板（3 个，默认只显示第一个）
    std::vector<Layout*> pages;
    for (int i = 0; i < 3; i++) {
        auto page = Layout::create();
        page->setContentSize(Size(480, 270));
        page->setPosition(Vec2::ZERO);
        page->setBackGroundColorType(Layout::BackGroundColorType::SOLID);
        page->setBackGroundColor(Color3B(40, 40, 60));

        auto content = Text::create(
            StringUtils::format("第 %d 页内容", i + 1),
            "fonts/simhei.ttf", 24
        );
        content->setPosition(Vec2(240, 135));
        page->addChild(content);

        page->setVisible(i == 0);
        this->addChild(page);
        pages.push_back(page);
    }

    // Tab 按钮
    std::vector<std::string> tabNames = {"基本", "高级", "关于"};
    std::vector<Button*> tabs;

    for (int i = 0; i < 3; i++) {
        auto tab = Button::create("tab_bg.png");
        tab->setScale9Enabled(true);
        tab->setContentSize(Size(150, 50));
        tab->setTitleText(tabNames[i]);
        tab->setTitleFontSize(18);

        auto param = LinearLayoutParameter::create();
        param->setMargin(Margin(5, 0, 5, 0));
        tab->setLayoutParameter(param);

        tab->addClickEventListener(
            [i, pages, &tabs](Ref*) {
                for (int j = 0; j < 3; j++) {
                    pages[j]->setVisible(j == i);
                }
            }
        );

        tabBar->addChild(tab);
        tabs.push_back(tab);
    }

    return true;
}
```

</details>

### 练习2：数字跳动效果

**题目**：实现一个分数 Label，当分数变化时数字从旧值"跳动"到新值（如从 0 快速跳到 1000，中间显示渐变的数字）。

<details>
<summary>参考答案</summary>

```cpp
class AnimatedScore : public Node {
    Label* _label;
    int _currentValue = 0;
    int _targetValue = 0;
    float _elapsed = 0;
    float _duration = 0;
    bool _animating = false;

public:
    CREATE_FUNC(AnimatedScore);

    bool init() override {
        if (!Node::init()) return false;
        _label = Label::createWithTTF("0", "fonts/mono.ttf", 32);
        this->addChild(_label);
        return true;
    }

    void setScore(int target, float duration = 0.5f) {
        _targetValue = target;
        _duration = duration;
        _elapsed = 0;
        _animating = true;
        this->scheduleUpdate();
    }

    void update(float dt) override {
        if (!_animating) return;

        _elapsed += dt;
        float t = _elapsed / _duration;
        if (t >= 1.0f) {
            t = 1.0f;
            _animating = false;
            this->unscheduleUpdate();
        }

        // 缓动函数：EaseOut
        float easedT = 1.0f - powf(1.0f - t, 3.0f);
        int displayValue = _currentValue
            + static_cast<int>((_targetValue - _currentValue) * easedT);

        _label->setString(StringUtils::format("%d", displayValue));

        if (!_animating) {
            _currentValue = _targetValue;
        }
    }
};

// 使用
auto score = AnimatedScore::create();
score->setScore(1000, 0.8f);  // 0.8 秒内从当前值跳到 1000
```

</details>

---

## 下一步

下一节 [02-CSB文件加载与Cocos Studio](./02-CSB文件加载与Cocos Studio.md) 将学习 Cocos Studio 的 CSB 文件格式、加载机制，以及 KrKr2 项目中如何结合 CSBReader 与 CsdUIFactory 实现 UI 表单系统。
