# CSB 文件加载与 Cocos Studio

## 本节目标

- 理解 Cocos Studio 的工作流：从可视化编辑器到 CSB 二进制文件
- 掌握 CSBReader/CSLoader 的加载机制与节点映射原理
- 学会使用 NodeMap 按名字查找 UI 控件并绑定逻辑
- 理解 KrKr2 中 CsdUIFactory 作为 CSB 纯代码备选方案的设计
- 能够在实际项目中灵活选择 CSB 加载或纯代码创建 UI

## 前置知识

- 已阅读 [01-Label-Button-Layout基础](./01-Label-Button-Layout基础.md)（了解 UI Widget 体系）
- 已阅读 [02-Scene-Layer-Node树](../01-Cocos2d-x架构/02-Scene-Layer-Node树.md)
- 熟悉 C++ 模板与 dynamic_cast

---

## 1. Cocos Studio 与 CSB 格式

### 1.1 什么是 Cocos Studio？

Cocos Studio 是 Cocos2d-x 官方的可视化 UI 编辑器（已停止维护，但生态中仍广泛使用）。工作流程：

```
Cocos Studio 编辑器          运行时
┌──────────────────┐      ┌─────────────────────────┐
│  拖拽 Widget      │      │  CSLoader::createNode()  │
│  设置属性/动画    │ ───→ │  解析 FlatBuffers        │
│  导出 .csb 文件   │      │  重建 Node 树            │
└──────────────────┘      └─────────────────────────┘
     .csd (XML源文件)           .csb (二进制)
```

### 1.2 CSB 文件格式

CSB（Cocos Studio Binary）是 FlatBuffers 序列化的二进制格式，包含：

| 数据 | 说明 |
|------|------|
| 节点树结构 | 每个节点的类型（Button/Text/Layout 等）、属性、子节点列表 |
| 属性值 | 坐标、大小、颜色、透明度、锚点、缩放、旋转等 |
| 资源引用 | 图片文件名、字体文件名 |
| 动画时间轴 | ActionTimeline——关键帧动画数据 |
| 节点名称 | 每个节点的 name 字段，用于程序中查找控件 |

与 JSON 格式（旧版 Cocos Studio 导出）相比，CSB 的优势：
- **加载速度快**——FlatBuffers 零拷贝解析，无需逐字符 parse
- **体积小**——二进制格式比 JSON 小 50-70%
- **类型安全**——schema 定义了每个字段的类型

### 1.3 KrKr2 中的 CSB 文件

```
krkr2/ui/cocos-studio/ui/
├── GameMainMenu.csb       # 浮动主菜单
├── FileSelectorForm.csb   # 文件选择器
├── PreferenceForm.csb     # 偏好设置
├── TextPadForm.csb        # 文本面板
├── ConsoleForm.csb        # 控制台
├── MessageBox.csb         # 消息对话框
├── NaviBar.csb            # 通用导航栏
├── Bottom.csb             # 通用底部栏
├── BottomNone.csb         # 空白底部栏
├── ...（共 27 个 .csb 文件）
└── locale/                # 多语言 XML 翻译资源
    ├── zh.xml
    ├── en.xml
    └── ...
```

---

## 2. CSLoader——标准加载方式

### 2.1 基本用法

```cpp
#include "cocostudio/ActionTimeline/CSLoader.h"

// 最基本的加载——返回一个 Node 树
auto node = CSLoader::createNode("ui/GameMainMenu.csb");
if (!node) {
    CCLOG("ERROR: Failed to load CSB file");
    return;
}
this->addChild(node);

// 按名字查找子节点
auto startBtn = node->getChildByName<Button*>("btn_start");
if (startBtn) {
    startBtn->addClickEventListener([](Ref*) {
        CCLOG("开始游戏");
    });
}
```

### 2.2 CSLoader 内部流程

```
CSLoader::createNode("xxx.csb")
  │
  ├── 读取文件二进制数据
  ├── FlatBuffers 反序列化
  │     → 得到 CSParseBinary 结构
  │
  ├── 遍历节点树（递归）
  │     for each nodeTree:
  │       ├── 根据 classname 创建对应 Widget
  │       │     "Button"  → Button::create()
  │       │     "Text"    → Text::create()
  │       │     "Layout"  → Layout::create()
  │       │     "Image"   → ImageView::create()
  │       │     ...
  │       ├── 设置属性（位置、大小、颜色...）
  │       ├── 设置 name（关键！用于后续查找）
  │       └── 递归创建子节点
  │
  └── 加载 ActionTimeline（如果有动画数据）
        → 挂载到根节点上
```

### 2.3 ActionTimeline——CSB 内置动画

```cpp
#include "cocostudio/ActionTimeline/CCActionTimeline.h"

// 加载节点和动画
auto node = CSLoader::createNode("ui/SplashScreen.csb");
auto timeline = CSLoader::createTimeline("ui/SplashScreen.csb");

if (timeline) {
    node->runAction(timeline);

    // 播放指定动画片段
    timeline->play("fadeIn", false);  // 名称, 是否循环

    // 检查动画是否存在
    if (timeline->IsAnimationInfoExists("idle")) {
        timeline->play("idle", true);  // 循环播放 idle 动画
    }

    // 设置动画结束回调
    timeline->setLastFrameCallFunc([](void) {
        CCLOG("动画播放完毕");
    });
}
```

---

## 3. CSBReader——KrKr2 的增强加载器

KrKr2 项目没有直接使用 `CSLoader::createNode`，而是封装了 `CSBReader` 类，在加载时自动建立**名称→节点映射**。

### 3.1 CSBReader 源码分析

```cpp
// BaseForm.cpp 第 56-77 行
Node* CSBReader::Load(const char* filename) {
    clear();           // 清空旧映射（CSBReader 继承自 NodeMap）
    FileName = filename;

    // 关键：CSLoader 的第二个参数是回调函数
    // 每创建一个节点都会调用此回调
    Node* ret = CSLoader::createNode(filename, [this](Ref* p) {
        Node* node = dynamic_cast<Node*>(p);
        std::string name = node->getName();
        if (!name.empty())
            operator[](name) = node;  // 名称不为空则存入映射

        // 自动播放标记为 "autoplay" 的动画
        int nAction = node->getNumberOfRunningActions();
        if (nAction == 1) {
            auto* action = dynamic_cast<
                cocostudio::timeline::ActionTimeline*>(
                node->getActionByTag(node->getTag())
            );
            if (action && action->IsAnimationInfoExists("autoplay")) {
                action->play("autoplay", true);
            }
        }
    });

    if (!ret) {
        TVPShowSimpleMessageBox(filename, "Fail to load ui file");
    }
    return ret;
}
```

**CSBReader 的两个增强功能**：

1. **自动建立名称映射**——加载过程中，每个有 name 的节点都被存入 `unordered_map<string, Node*>`，后续可以 O(1) 查找
2. **自动播放动画**——如果节点上有名为 `autoplay` 的 ActionTimeline 动画片段，自动开始循环播放

### 3.2 NodeMap——名称映射系统

```cpp
// BaseForm.h 第 44-71 行
class NodeMap : public std::unordered_map<std::string, Node*> {
protected:
    const char* FileName;

public:
    NodeMap();
    NodeMap(const char* filename, Node* node);

    // 模板方法——按名称查找并 dynamic_cast 到目标类型
    template <typename T = Node>
    T* findController(const std::string& name, bool notice = true) const {
        auto* node = findController<Node>(name, notice);
        if (node) {
            return dynamic_cast<T*>(node);
        }
        return nullptr;
    }

    // 查找 Widget 的便捷方法
    Widget* findWidget(const std::string& name, bool notice = true) const {
        return findController<Widget>(name, notice);
    }

    // 从已有节点树构建映射
    void initFromNode(Node* node);
};
```

**使用方式**：

```cpp
CSBReader reader;
Node* root = reader.Load("ui/PreferenceForm.csb");

// 用 findController 查找具体类型的控件
auto* titleText = reader.findController<Text>("title_text");
auto* saveBtn = reader.findController<Button>("btn_save");
auto* volumeSlider = reader.findController<Slider>("slider_volume");
auto* contentList = reader.findController<ListView>("list_content");

// 如果节点不存在，findController 默认弹出错误对话框
// 传 false 可以静默失败
auto* optional = reader.findController<Button>("btn_optional", false);
```

### 3.3 与直接 getChildByName 的对比

```cpp
// 方式1：标准 Cocos2d-x 查找（递归遍历，O(n)）
auto* btn = root->getChildByName<Button*>("btn_save");

// 方式2：CSBReader 的 NodeMap 查找（哈希表，O(1)）
auto* btn = reader.findController<Button>("btn_save");
```

| 特性 | getChildByName | CSBReader/NodeMap |
|------|----------------|-------------------|
| 时间复杂度 | O(n)，遍历子树 | O(1)，哈希查找 |
| 深度搜索 | 需要用 `getChildByName("btn", true)` | 自动扁平化所有节点 |
| 类型安全 | 需要手动模板参数 | `findController<T>` 自动 cast |
| 错误提示 | 返回 nullptr，静默失败 | 默认弹对话框提示缺失节点 |
| 多次查找 | 每次都 O(n) | 一次建表，后续全 O(1) |

---

## 4. iTVPBaseForm——表单基类

KrKr2 的所有 UI 表单都继承自 `iTVPBaseForm`，它定义了标准的三段式布局。

### 4.1 三段式布局架构

```
┌─────────────────────────────┐
│         NaviBar (10%)       │  ← 导航栏：返回按钮、标题、功能按钮
├─────────────────────────────┤
│                             │
│         Body (80%)          │  ← 主体内容区：列表/表单/文本
│                             │
│                             │
├─────────────────────────────┤
│        Footer (10%)         │  ← 底部栏：工具按钮（可选）
└─────────────────────────────┘
```

### 4.2 initFromFile 的双重构建模式

```cpp
// BaseForm.h 第 97-113 行
// 模式1：通过 CsdUIFactory 的构建函数（纯代码创建 UI）
bool initFromFile(
    const Csd::NodeBuilderFn& naviBarCall,   // NaviBar 构建函数
    const Csd::NodeBuilderFn& bodyCall,      // Body 构建函数
    const Csd::NodeBuilderFn& bottomBarCall, // Footer 构建函数
    Node* parent = nullptr
);

// 模式2：直接传入已有节点（从 CSB 加载后）
bool initFromFile(Node* naviBar, Node* body, Node* bottomBar,
                  Node* parent = nullptr);

// 模式3：只有 Body（无导航栏和底部栏）
bool initFromFile(const Csd::NodeBuilderFn& body);
```

### 4.3 实际加载流程

```cpp
// BaseForm.cpp 第 83-135 行（简化注释版）
bool iTVPBaseForm::initFromFile(
    const Csd::NodeBuilderFn& naviBarCall,
    const Csd::NodeBuilderFn& bodyCall,
    const Csd::NodeBuilderFn& bottomBarCall,
    Node* parent)
{
    const bool ret = Node::init();
    const auto scale = TVPMainScene::GetInstance()->getUIScale();

    // 1. 调用构建函数创建三段节点
    //    每个构建函数接收 (Size, scale) 参数
    auto* naviBar   = naviBarCall(rearrangeHeaderSize(parent), scale);
    auto* body      = bodyCall(rearrangeBodySize(parent), scale);
    auto* bottomBar = bottomBarCall(rearrangeFooterSize(parent), scale);

    RootNode = body;
    if (!RootNode) return false;

    if (!parent) parent = this;

    // 2. 设置布局参数
    if (naviBar) {
        // 从 NaviBar 节点树中查找 left/right 按钮
        NaviBar.Root  = naviBar->getChildByName("background");
        NaviBar.Left  = NaviBar.Root->getChildByName<Button*>("left");
        NaviBar.Right = NaviBar.Root->getChildByName<Button*>("right");
        bindHeaderController(NaviBar.Root);

        auto param = LinearLayoutParameter::create();
        param->setGravity(LinearLayoutParameter::LinearGravity::TOP);
        naviBar->setLayoutParameter(param);
        parent->addChild(naviBar);
    }

    if (bottomBar) {
        BottomBar.Root = bottomBar;
        bindFooterController(bottomBar);

        auto param = LinearLayoutParameter::create();
        param->setGravity(LinearLayoutParameter::LinearGravity::BOTTOM);
        bottomBar->setLayoutParameter(param);
        parent->addChild(BottomBar.Root);
    }

    // 3. Body 填充中间
    auto param = LinearLayoutParameter::create();
    param->setGravity(
        LinearLayoutParameter::LinearGravity::CENTER_VERTICAL
    );
    body->setLayoutParameter(param);
    parent->addChild(RootNode);

    bindBodyController(RootNode);
    return ret;
}
```

### 4.4 子类必须实现的三个绑定方法

```cpp
class iTVPBaseForm : public Node {
    // 子类必须实现这三个纯虚函数
    virtual void bindHeaderController(const Node* allNodes) = 0;
    virtual void bindBodyController(const Node* allNodes) = 0;
    virtual void bindFooterController(const Node* allNodes) = 0;
};
```

每个绑定方法的职责是：从已加载的节点树中查找控件，绑定事件回调。例如：

```cpp
class PreferenceForm : public iTVPBaseForm {
    void bindHeaderController(const Node* allNodes) override {
        // 查找导航栏中的返回按钮
        auto backBtn = allNodes->getChildByName<Button*>("left");
        if (backBtn) {
            backBtn->addClickEventListener([this](Ref*) {
                TVPMainScene::GetInstance()->popUIForm(
                    this, TVPMainScene::eLeaveAniLeaveFromLeft
                );
            });
        }
    }

    void bindBodyController(const Node* allNodes) override {
        // 查找并绑定偏好设置控件
        auto volumeSlider = allNodes->getChildByName<Slider*>("volume");
        if (volumeSlider) {
            volumeSlider->addEventListener(
                [](Ref* sender, Slider::EventType type) {
                    // 处理音量变化
                }
            );
        }
    }

    void bindFooterController(const Node* allNodes) override {
        // 如果没有 Footer，空实现即可
    }
};
```

---

## 5. CsdUIFactory——纯代码备选方案

### 5.1 为什么需要备选方案？

CSB 文件依赖 Cocos Studio 工具生成。当以下情况发生时，需要纯代码替代：
- Cocos Studio 已停止维护，新版系统可能无法运行
- CI 环境中无法安装 Cocos Studio 来重新生成 CSB
- CSB 文件损坏或版本不兼容
- 需要动态生成 UI（内容取决于运行时数据）

### 5.2 NodeBuilderFn 签名

```cpp
// CsdUIFactory.h 中的核心类型定义
namespace Csd {
    // 构建函数签名：接收 (尺寸, 缩放比) → 返回 Node*
    using NodeBuilderFn = std::function<Node*(const Size& size, float scale)>;
}
```

每个构建函数负责创建一个完整的 UI 子树。CsdUIFactory.h（1097 行）中为每个 CSB 文件都提供了对应的纯代码构建函数。

### 5.3 NaviBar 构建示例

```cpp
// CsdUIFactory.h 中 NaviBar 的纯代码构建（简化）
static Node* buildNaviBar(const Size& size, float scale) {
    auto root = Layout::create();
    root->setContentSize(size);

    auto bg = Layout::create();
    bg->setName("background");
    bg->setContentSize(size);
    bg->setBackGroundColorType(Layout::BackGroundColorType::SOLID);
    bg->setBackGroundColor(Color3B(50, 50, 70));

    // 左侧返回按钮
    auto leftBtn = Button::create("btn_back.png");
    leftBtn->setName("left");
    leftBtn->setPosition(Vec2(30, size.height / 2));
    bg->addChild(leftBtn);

    // 右侧功能按钮
    auto rightBtn = Button::create("btn_menu.png");
    rightBtn->setName("right");
    rightBtn->setPosition(Vec2(size.width - 30, size.height / 2));
    bg->addChild(rightBtn);

    // 中间标题
    auto title = Text::create("标题", "fonts/simhei.ttf", 20);
    title->setName("title");
    title->setPosition(Vec2(size.width / 2, size.height / 2));
    bg->addChild(title);

    root->addChild(bg);
    return root;
}
```

### 5.4 表单使用构建函数

```cpp
// 某个具体表单的 init 方法
bool PreferenceForm::init() {
    // 使用 CsdUIFactory 的构建函数
    return initFromFile(
        Csd::buildNaviBar,       // NaviBar 构建函数
        Csd::buildPreferenceBody, // Body 构建函数
        Csd::buildBottomNone,     // 无 Footer
        nullptr                   // parent（默认 self）
    );
}
```

### 5.5 CSB vs 纯代码的对比

| 维度 | CSB 文件 | CsdUIFactory 纯代码 |
|------|----------|---------------------|
| 创建方式 | Cocos Studio 可视化编辑 | C++ 手写代码 |
| 修改成本 | 需要重新打开编辑器导出 | 直接修改代码 |
| 版本管理 | 二进制 diff 不可读 | 代码 diff 清晰可读 |
| 加载性能 | 需要反序列化 | 直接创建，稍快 |
| 可维护性 | 依赖外部工具 | 自包含，无外部依赖 |
| 布局精度 | 所见即所得 | 需要手动计算坐标 |
| 团队协作 | 设计师可参与 | 只有程序员能修改 |

**KrKr2 的策略**：CSB 文件作为主要 UI 资源，CsdUIFactory 作为后备。两者通过 `NodeBuilderFn` 接口统一调用，上层代码不需要知道 UI 是从 CSB 加载的还是代码创建的。

---

## 动手实践

### 实践1：模拟 CSBReader 加载流程

```cpp
#include "cocos2d.h"
#include "ui/CocosGUI.h"
#include <unordered_map>
USING_NS_CC;
using namespace cocos2d::ui;

// 简化版 NodeMap——不依赖项目代码
class SimpleNodeMap {
    std::unordered_map<std::string, Node*> _map;

public:
    // 递归遍历节点树，收集所有有名字的节点
    void buildFromNode(Node* root) {
        _map.clear();
        collectNodes(root);
    }

    template<typename T = Node>
    T* find(const std::string& name) const {
        auto it = _map.find(name);
        if (it != _map.end()) {
            return dynamic_cast<T*>(it->second);
        }
        CCLOG("WARNING: Node '%s' not found in map", name.c_str());
        return nullptr;
    }

private:
    void collectNodes(Node* node) {
        const auto& name = node->getName();
        if (!name.empty()) {
            _map[name] = node;
        }
        for (auto* child : node->getChildren()) {
            collectNodes(child);
        }
    }
};

// 使用示例
bool MyScene::init() {
    if (!Scene::init()) return false;

    // 手动构建一个 UI 树（模拟 CSB 加载结果）
    auto panel = Layout::create();
    panel->setContentSize(Size(400, 300));
    panel->setName("root_panel");

    auto title = Text::create("设置", "fonts/simhei.ttf", 24);
    title->setName("title_text");
    title->setPosition(Vec2(200, 270));
    panel->addChild(title);

    auto saveBtn = Button::create("btn.png");
    saveBtn->setName("btn_save");
    saveBtn->setTitleText("保存");
    saveBtn->setPosition(Vec2(200, 50));
    panel->addChild(saveBtn);

    this->addChild(panel);

    // 用 SimpleNodeMap 查找控件
    SimpleNodeMap nodeMap;
    nodeMap.buildFromNode(panel);

    auto* foundTitle = nodeMap.find<Text>("title_text");
    if (foundTitle) {
        CCLOG("找到标题: %s", foundTitle->getString().c_str());
    }

    auto* foundBtn = nodeMap.find<Button>("btn_save");
    if (foundBtn) {
        foundBtn->addClickEventListener([](Ref*) {
            CCLOG("保存按钮被点击");
        });
    }

    return true;
}
```

### 实践2：实现三段式表单基类

```cpp
// 简化版 BaseForm——可独立编译
class SimpleBaseForm : public Node {
protected:
    Node* _naviBar = nullptr;
    Node* _body = nullptr;
    Node* _footer = nullptr;

    using BuilderFn = std::function<Node*(const Size&)>;

    bool initWithBuilders(BuilderFn navBuilder,
                          BuilderFn bodyBuilder,
                          BuilderFn footBuilder) {
        if (!Node::init()) return false;

        auto size = Director::getInstance()->getVisibleSize();
        float naviH = size.height * 0.1f;
        float footH = size.height * 0.1f;
        float bodyH = size.height * 0.8f;

        // NaviBar
        if (navBuilder) {
            _naviBar = navBuilder(Size(size.width, naviH));
            if (_naviBar) {
                _naviBar->setPosition(Vec2(0, bodyH + footH));
                this->addChild(_naviBar, 2);
            }
        }

        // Body
        if (bodyBuilder) {
            _body = bodyBuilder(Size(size.width, bodyH));
            if (_body) {
                _body->setPosition(Vec2(0, footH));
                this->addChild(_body, 1);
            }
        }

        // Footer
        if (footBuilder) {
            _footer = footBuilder(Size(size.width, footH));
            if (_footer) {
                _footer->setPosition(Vec2::ZERO);
                this->addChild(_footer, 2);
            }
        }

        return true;
    }
};

// 使用示例——设置页面
class SettingsForm : public SimpleBaseForm {
public:
    CREATE_FUNC(SettingsForm);

    bool init() override {
        return initWithBuilders(
            // NaviBar 构建函数
            [this](const Size& size) -> Node* {
                auto bar = LayerColor::create(
                    Color4B(50, 50, 70, 255), size.width, size.height
                );
                auto title = Label::createWithTTF(
                    "设置", "fonts/simhei.ttf", 20
                );
                title->setPosition(Vec2(size.width / 2, size.height / 2));
                bar->addChild(title);

                auto backBtn = Button::create("btn_back.png");
                backBtn->setPosition(Vec2(30, size.height / 2));
                backBtn->addClickEventListener([](Ref*) {
                    Director::getInstance()->popScene();
                });
                bar->addChild(backBtn);
                return bar;
            },
            // Body 构建函数
            [this](const Size& size) -> Node* {
                auto body = Layout::create();
                body->setContentSize(size);
                body->setLayoutType(Layout::Type::VERTICAL);
                // ... 添加设置项
                return body;
            },
            // 无 Footer
            nullptr
        );
    }
};
```

### 实践3：动画自动播放检测

```cpp
// 模拟 CSBReader 的自动播放逻辑
void checkAutoplayAnimations(Node* root) {
    // 遍历节点树
    std::function<void(Node*)> visit = [&](Node* node) {
        int nAction = node->getNumberOfRunningActions();
        if (nAction >= 1) {
            // 尝试获取 ActionTimeline
            auto* action = dynamic_cast<
                cocostudio::timeline::ActionTimeline*>(
                node->getActionByTag(node->getTag())
            );
            if (action) {
                // 检查是否有 "autoplay" 动画片段
                if (action->IsAnimationInfoExists("autoplay")) {
                    CCLOG("节点 '%s' 有 autoplay 动画，自动播放",
                          node->getName().c_str());
                    action->play("autoplay", true);
                }

                // 列出所有动画片段
                const auto& infos = action->getAnimationInfos();
                for (const auto& info : infos) {
                    CCLOG("  动画片段: %s (帧 %d-%d)",
                          info.name.c_str(),
                          info.startIndex,
                          info.endIndex);
                }
            }
        }

        for (auto* child : node->getChildren()) {
            visit(child);
        }
    };

    visit(root);
}
```

---

## 对照项目源码

### CSBReader 的完整加载过程

```cpp
// BaseForm.cpp 第 56-77 行
// CSBReader::Load 是所有 UI 表单的入口
Node* CSBReader::Load(const char* filename) {
    clear();
    FileName = filename;
    Node* ret = CSLoader::createNode(filename, [this](Ref* p) {
        Node* node = dynamic_cast<Node*>(p);
        std::string name = node->getName();
        if (!name.empty())
            operator[](name) = node;
        // ... autoplay 逻辑
    });
    if (!ret) {
        TVPShowSimpleMessageBox(filename, "Fail to load ui file");
    }
    return ret;
}
```

### NodeMap::initFromNode 的递归收集

```cpp
// BaseForm.cpp 第 40-48 行
void NodeMap::initFromNode(Node* node) {
    const Vector<Node*>& childlist = node->getChildren();
    for (auto child : childlist) {
        std::string name = child->getName();
        if (!name.empty())
            (*this)[name] = child;
        initFromNode(child);  // 递归遍历整棵子树
    }
}
```

### iTVPFloatForm 的浮动布局

```cpp
// BaseForm.cpp 第 147-163 行
// 浮动表单（不全屏，居中显示 75% 大小）
void iTVPFloatForm::rearrangeLayout() {
    float scale = TVPMainScene::GetInstance()->getUIScale();
    Size sceneSize = TVPMainScene::GetInstance()->getUINodeSize();
    setContentSize(sceneSize);
    Vec2 center = sceneSize / 2;
    sceneSize.height *= 0.75f;
    sceneSize.width *= 0.75f;
    if (RootNode) {
        sceneSize.width /= scale;
        sceneSize.height /= scale;
        RootNode->setContentSize(sceneSize);
        ui::Helper::doLayout(RootNode);  // 触发 Layout 重新排列
        RootNode->setScale(scale);
        RootNode->setAnchorPoint(Vec2(0.5f, 0.5f));
        RootNode->setPosition(center);
    }
}
```

**设计模式总结**：

```
CSB 文件路径                     纯代码构建函数
    │                                │
    ▼                                ▼
CSBReader::Load()            Csd::buildXxxBody()
    │                                │
    ▼                                ▼
NodeMap (名称→节点映射)      返回 Node* 子树
    │                                │
    └──────────┬─────────────────────┘
               ▼
    iTVPBaseForm::initFromFile()
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
NaviBar      Body       Footer
(10%)        (80%)      (10%)
    │          │          │
    ▼          ▼          ▼
bindHeader  bindBody  bindFooter
Controller  Controller Controller
```

---

## 本节小结

| 概念 | 说明 |
|------|------|
| **CSB 格式** | Cocos Studio 导出的 FlatBuffers 二进制 UI 文件 |
| **CSLoader** | Cocos2d-x 内置的 CSB 加载器，反序列化节点树 |
| **CSBReader** | KrKr2 封装，加载时自动建立名称映射 + 自动播放动画 |
| **NodeMap** | `unordered_map<string, Node*>`，O(1) 按名查找控件 |
| **iTVPBaseForm** | 三段式布局基类（NaviBar 10% + Body 80% + Footer 10%） |
| **CsdUIFactory** | CSB 的纯代码替代方案，1097 行代码手写所有 UI |
| **NodeBuilderFn** | 统一接口 `(Size, scale) → Node*`，抹平 CSB/纯代码差异 |
| **ActionTimeline** | CSB 内嵌的关键帧动画，支持按名称播放 |

---

## 练习题与答案

### 练习1：实现带缓存的 CSBReader

**题目**：标准 CSBReader 每次 Load 都重新解析 CSB 文件。实现一个 `CachedCSBReader`，对同一文件只解析一次，后续调用返回克隆的节点树。

<details>
<summary>参考答案</summary>

```cpp
class CachedCSBReader {
    // 缓存：filename → 原始节点树
    static std::unordered_map<std::string, Node*> _cache;

public:
    static Node* Load(const std::string& filename) {
        auto it = _cache.find(filename);
        if (it != _cache.end()) {
            // 已缓存——克隆节点树
            return cloneNodeTree(it->second);
        }

        // 首次加载
        CSBReader reader;
        Node* original = reader.Load(filename.c_str());
        if (!original) return nullptr;

        // 缓存原始节点（retain 防止被回收）
        original->retain();
        _cache[filename] = original;

        // 返回克隆
        return cloneNodeTree(original);
    }

    static void clearCache() {
        for (auto& pair : _cache) {
            pair.second->release();
        }
        _cache.clear();
    }

private:
    static Node* cloneNodeTree(Node* src) {
        // Cocos2d-x 没有通用的 Node 克隆方法
        // 对于 CSB UI，最简单的方式是重新加载
        // 这里展示概念——实际项目中可能需要重新 CSLoader
        return CSLoader::createNode(src->getName());
    }
};

std::unordered_map<std::string, Node*> CachedCSBReader::_cache;
```

> **注意**：Cocos2d-x 的 Node 没有通用的 clone 方法（只有 Sprite 支持）。实际的缓存策略更多是缓存 FlatBuffers 的二进制数据以避免重复磁盘 IO，而不是缓存节点树。

</details>

### 练习2：NodeMap 的线程安全问题

**题目**：`NodeMap` 使用 `unordered_map`。如果在子线程中加载 CSB（Cocos2d-x 支持异步加载），会有什么问题？如何解决？

<details>
<summary>参考答案</summary>

**问题**：`unordered_map` 不是线程安全的。如果主线程正在通过 `findController` 读取映射，同时子线程在 `Load` 中写入映射，会导致数据竞争（未定义行为）。

**解决方案**：

方案1：确保加载和查找在同一线程（主线程）

```cpp
// 异步加载只读文件，节点创建在主线程
Director::getInstance()->getScheduler()->performFunctionInCocosThread([=]() {
    CSBReader reader;
    Node* node = reader.Load("ui/Form.csb");
    // 所有 NodeMap 操作都在主线程
});
```

方案2：加读写锁

```cpp
class ThreadSafeNodeMap : public NodeMap {
    mutable std::shared_mutex _mutex;

public:
    template<typename T = Node>
    T* findControllerSafe(const std::string& name) const {
        std::shared_lock lock(_mutex);  // 读锁
        return findController<T>(name, false);
    }

    void insertSafe(const std::string& name, Node* node) {
        std::unique_lock lock(_mutex);  // 写锁
        (*this)[name] = node;
    }
};
```

**KrKr2 的做法**：所有 UI 加载都在主线程进行（Cocos2d-x 的 OpenGL 上下文绑定在主线程），所以不存在线程安全问题。

</details>

### 练习3：设计一个 UI 热重载系统

**题目**：在开发阶段，修改 CSB 文件后需要重启应用才能看到效果。设计一个热重载系统：监测 CSB 文件变化 → 自动重新加载 → 刷新 UI。写出核心类的接口和关键方法。

<details>
<summary>参考答案</summary>

```cpp
class CSBHotReloader {
    struct WatchEntry {
        std::string filepath;
        time_t lastModified;
        std::function<void(Node*)> onReload;  // 重载回调
        Node* currentRoot;
    };

    std::vector<WatchEntry> _watches;
    float _checkInterval = 1.0f;  // 每秒检查一次

public:
    // 注册监听
    void watch(const std::string& csbPath,
               Node* parentNode,
               std::function<void(Node*)> onReload) {
        WatchEntry entry;
        entry.filepath = FileUtils::getInstance()->fullPathForFilename(csbPath);
        entry.lastModified = getFileModTime(entry.filepath);
        entry.onReload = onReload;
        entry.currentRoot = nullptr;
        _watches.push_back(entry);
    }

    // 每帧检查（在 Scene::update 中调用）
    void update(float dt) {
        static float elapsed = 0;
        elapsed += dt;
        if (elapsed < _checkInterval) return;
        elapsed = 0;

        for (auto& entry : _watches) {
            time_t currentMod = getFileModTime(entry.filepath);
            if (currentMod > entry.lastModified) {
                entry.lastModified = currentMod;
                reload(entry);
            }
        }
    }

private:
    void reload(WatchEntry& entry) {
        CCLOG("[HotReload] 检测到变化: %s", entry.filepath.c_str());

        // 清除纹理缓存（CSB 可能引用了新图片）
        Director::getInstance()->getTextureCache()->removeUnusedTextures();

        // 重新加载
        CSBReader reader;
        Node* newRoot = reader.Load(entry.filepath.c_str());
        if (newRoot && entry.onReload) {
            entry.onReload(newRoot);
        }
    }

    time_t getFileModTime(const std::string& path) {
        struct stat st;
        if (stat(path.c_str(), &st) == 0) {
            return st.st_mtime;
        }
        return 0;
    }
};
```

</details>

---

## 下一步

下一节 [03-事件监听与分发](./03-事件监听与分发.md) 将学习 Cocos2d-x 的事件系统（EventDispatcher），包括触摸事件、键盘事件、自定义事件的监听与分发机制，这是 UI 交互的底层基础。
