# SpriteFrame 与 SpriteSheet

> **所属模块：** P10-Cocos2d-x 框架
> **前置知识：** [01-Sprite创建与纹理管理](01-Sprite创建与纹理管理.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 SpriteFrame 的概念（纹理 + 矩形区域 + 元数据）
2. 使用 SpriteSheet（.plist + .png）和 SpriteFrameCache
3. 理解图集打包的原理与性能优势（减少 Draw Call）
4. 使用 SpriteFrame 创建帧动画（Animation）
5. 了解常用的图集打包工具（TexturePacker、Cocos Studio）

## SpriteFrame 概念

### 从 Sprite 到 SpriteFrame

在上一节中，我们通过 `Sprite::create("player.png")` 创建精灵，每个图片文件对应一张单独的纹理。但在实际游戏开发中，角色的不同动作帧（行走、攻击、跳跃）可能有数十张小图。如果每张小图单独加载为一个纹理：

```
❌ 单独纹理的问题：
player_idle_01.png  → Texture2D #1  → glBindTexture → Draw Call #1
player_idle_02.png  → Texture2D #2  → glBindTexture → Draw Call #2
player_idle_03.png  → Texture2D #3  → glBindTexture → Draw Call #3
player_walk_01.png  → Texture2D #4  → glBindTexture → Draw Call #4
...
20 张小图 → 20 个纹理 → 20 次 Draw Call
```

**Draw Call** 是 CPU 向 GPU 发出的绘制指令。每次切换纹理（`glBindTexture`）都需要一次 Draw Call。Draw Call 数量过多是移动设备上的主要性能瓶颈。

解决方案：将所有小图**打包到一张大纹理**（Atlas / SpriteSheet），每个小图变成大纹理中的一个矩形区域，即 **SpriteFrame**：

```
✅ SpriteSheet 方案：
characters.png（一张大纹理）
┌────┬────┬────┬────┐
│idle│idle│idle│idle│
│ 01 │ 02 │ 03 │ 04 │  所有帧打包在同一张纹理中
├────┼────┼────┼────┤
│walk│walk│walk│walk│  切换帧 = 只需更新 UV 坐标
│ 01 │ 02 │ 03 │ 04 │  不需要切换纹理
├────┼────┼────┼────┤  → 1 次 Draw Call
│atk │atk │atk │    │
│ 01 │ 02 │ 03 │    │
└────┴────┴────┴────┘

characters.plist（描述每个帧在纹理中的位置）
```

`SpriteFrame` 封装了：
- 所属的 `Texture2D` 引用
- 在纹理中的矩形区域（`Rect`）
- 是否旋转（图集打包优化空间利用）
- 原始大小和偏移量（裁剪透明区域后的补偿）

### SpriteFrame 的数据结构

```cpp
// SpriteFrame 的核心属性
class SpriteFrame : public Ref {
    Texture2D* _texture;     // 所属纹理
    Rect _rect;              // 在纹理中的区域
    bool _rotated;           // 是否被旋转90°
    Vec2 _offset;            // 偏移量（裁剪补偿）
    Size _originalSize;      // 原始图片大小
    // ...
};
```

## SpriteSheet 文件格式

### .plist 文件结构

图集打包工具会生成两个文件：
1. `.png` — 合并后的大纹理图片
2. `.plist` — XML 格式的元数据，描述每帧的位置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>frames</key>
    <dict>
        <!-- 每个帧的描述 -->
        <key>player_idle_01.png</key>
        <dict>
            <key>frame</key>
            <string>{{0,0},{64,64}}</string>
            <!-- 在大纹理中的区域：x=0, y=0, w=64, h=64 -->
            
            <key>offset</key>
            <string>{0,0}</string>
            <!-- 裁剪透明区域后的偏移量 -->
            
            <key>rotated</key>
            <false/>
            <!-- 是否被旋转90° -->
            
            <key>sourceColorRect</key>
            <string>{{0,0},{64,64}}</string>
            <!-- 裁剪前在原图中的有效区域 -->
            
            <key>sourceSize</key>
            <string>{64,64}</string>
            <!-- 原图大小 -->
        </dict>
        
        <key>player_idle_02.png</key>
        <dict>
            <key>frame</key>
            <string>{{64,0},{64,64}}</string>
            <key>offset</key>
            <string>{0,0}</string>
            <key>rotated</key>
            <false/>
            <key>sourceColorRect</key>
            <string>{{0,0},{64,64}}</string>
            <key>sourceSize</key>
            <string>{64,64}</string>
        </dict>
        <!-- 更多帧... -->
    </dict>
    
    <key>metadata</key>
    <dict>
        <key>format</key>
        <integer>2</integer>
        <!-- plist 格式版本 -->
        
        <key>textureFileName</key>
        <string>characters.png</string>
        <!-- 对应的纹理文件名 -->
        
        <key>size</key>
        <string>{256,192}</string>
        <!-- 纹理大小 -->
    </dict>
</dict>
</plist>
```

### 旋转优化

打包工具可能将某些帧旋转 90° 以更紧密地填充空间：

```
不旋转：                        旋转后：
┌──────┬──┐                    ┌──────┬──┐
│ 64×32│  │  ← 浪费空间        │ 64×32│32│ ← 旋转后的 32×64 
│      │  │                    │      │×│    变成 64×32
├──────┤  │                    ├──────┤64│    更好地填充空间
│      │  │                    │      │ │
└──────┴──┘                    └──────┴──┘
```

当 `_rotated = true` 时，Cocos2d-x 在渲染时自动处理 UV 坐标的旋转，开发者无需关心。

## SpriteFrameCache 使用

### 加载图集

```cpp
#include "cocos2d.h"
USING_NS_CC;

// 获取 SpriteFrameCache 单例
auto frameCache = SpriteFrameCache::getInstance();

// 加载图集（plist + png）
frameCache->addSpriteFramesWithFile("characters.plist");
// 自动从 plist 的 metadata 中读取纹理文件名（characters.png）

// 也可以显式指定纹理
frameCache->addSpriteFramesWithFile(
    "characters.plist", "characters.png");

// 通过帧名获取 SpriteFrame
auto frame = frameCache->getSpriteFrameByName("player_idle_01.png");
if (frame) {
    // 用 SpriteFrame 创建 Sprite
    auto sprite = Sprite::createWithSpriteFrame(frame);
    sprite->setPosition(Vec2(480, 320));
    scene->addChild(sprite);
}

// 或直接用帧名创建 Sprite
auto sprite2 = Sprite::createWithSpriteFrameName("player_idle_01.png");
```

### 切换显示帧

```cpp
// 创建 Sprite 后，运行时切换帧
auto player = Sprite::createWithSpriteFrameName("player_idle_01.png");
scene->addChild(player);

// 切换到另一帧（同一图集内，不产生额外 Draw Call）
auto walkFrame = frameCache->getSpriteFrameByName("player_walk_01.png");
player->setSpriteFrame(walkFrame);

// 或用帧名直接设置
player->setSpriteFrame("player_walk_02.png");
```

### 帧动画（SpriteFrame + Animation）

使用多个 SpriteFrame 创建帧动画：

```cpp
// 方式 1：手动创建 Animation
Vector<SpriteFrame*> frames;
for (int i = 1; i <= 4; ++i) {
    auto name = StringUtils::format("player_walk_%02d.png", i);
    auto frame = SpriteFrameCache::getInstance()
        ->getSpriteFrameByName(name);
    if (frame) {
        frames.pushBack(frame);
    }
}

// 创建动画：每帧 0.15 秒
auto animation = Animation::createWithSpriteFrames(frames, 0.15f);

// 创建 Animate 动作
auto animate = Animate::create(animation);

// 循环播放
auto repeat = RepeatForever::create(animate);
player->runAction(repeat);
```

```cpp
// 方式 2：使用 AnimationCache（推荐用于复用动画）
auto animation = Animation::create();
animation->setDelayPerUnit(0.15f);  // 每帧间隔
animation->setRestoreOriginalFrame(true);  // 动画结束后恢复原始帧

for (int i = 1; i <= 8; ++i) {
    auto name = StringUtils::format("player_run_%02d.png", i);
    animation->addSpriteFrame(
        SpriteFrameCache::getInstance()->getSpriteFrameByName(name));
}

// 注册到 AnimationCache
AnimationCache::getInstance()->addAnimation(animation, "player_run");

// 之后在任何地方使用
auto cached = AnimationCache::getInstance()
    ->getAnimation("player_run");
player->runAction(RepeatForever::create(Animate::create(cached)));
```

### 缓存清理

```cpp
auto frameCache = SpriteFrameCache::getInstance();

// 移除指定 plist 的所有帧
frameCache->removeSpriteFramesFromFile("characters.plist");

// 移除指定纹理相关的所有帧
auto tex = Director::getInstance()->getTextureCache()
    ->getTextureForKey("characters.png");
frameCache->removeSpriteFramesFromTexture(tex);

// 移除所有未使用的帧（引用计数==1）
frameCache->removeUnusedSpriteFrames();

// 移除所有帧（慎用）
frameCache->removeSpriteFrames();
```

## 图集打包工具

### TexturePacker（业界标准）

TexturePacker 是最流行的图集打包工具，支持导出 Cocos2d-x 格式：

```
使用流程：
1. 将所有小图拖入 TexturePacker
2. 选择 Output Format: Cocos2d-x
3. 设置最大纹理尺寸（如 2048×2048）
4. 选择打包算法（MaxRects 最常用）
5. 设置像素格式（RGBA8888 / RGBA4444 等）
6. 导出 .plist + .png

命令行打包：
TexturePacker --format cocos2d-x \
    --sheet characters.png \
    --data characters.plist \
    --max-size 2048 \
    --opt RGBA4444 \
    --trim \
    ./sprites/*.png
```

关键设置：

| 选项 | 推荐值 | 说明 |
|------|--------|------|
| Max Size | 2048×2048 | 大多数设备支持的最大纹理尺寸 |
| Algorithm | MaxRects | 最优的矩形打包算法 |
| Trim | 开启 | 裁剪透明区域节省空间 |
| Rotation | 开启 | 允许旋转帧以更紧凑排列 |
| Pixel Format | 视情况 | RGBA8888(高质量) / RGBA4444(省内存) |
| Padding | 1-2 px | 帧间距，防止渲染时纹理采样出错 |
| Extrude | 1 px | 边缘扩展，防止缝隙 |

### Cocos Studio（已停止维护）

Cocos Studio 的 SpriteSheet Editor 可以创建图集，但该工具已停止维护。KrKr2 项目中的 `.csb` 文件即由 Cocos Studio 导出。

### 手动创建 SpriteFrame

如果不使用打包工具，可以手动创建 SpriteFrame（适合简单的均匀网格图集）：

```cpp
// 假设 tileset.png 是一张 256×256 的图，每个 tile 32×32
auto texture = Director::getInstance()->getTextureCache()
    ->addImage("tileset.png");

auto frameCache = SpriteFrameCache::getInstance();

int tileSize = 32;
int cols = 256 / tileSize;  // 8 列
int rows = 256 / tileSize;  // 8 行

for (int row = 0; row < rows; ++row) {
    for (int col = 0; col < cols; ++col) {
        Rect rect(col * tileSize, row * tileSize,
                  tileSize, tileSize);
        
        auto frame = SpriteFrame::createWithTexture(
            texture, rect);
        
        auto name = StringUtils::format(
            "tile_%d_%d", row, col);
        frameCache->addSpriteFrame(frame, name);
    }
}

// 使用
auto grassTile = Sprite::createWithSpriteFrameName("tile_0_0");
auto waterTile = Sprite::createWithSpriteFrameName("tile_0_1");
```

## Draw Call 优化原理

### SpriteBatchNode（已过时但值得理解）

Cocos2d-x 2.x 时代使用 `SpriteBatchNode` 来批量渲染共享同一纹理的 Sprite：

```cpp
// Cocos2d-x 2.x 方式（已过时）
auto batch = SpriteBatchNode::create("characters.png");
scene->addChild(batch);

for (int i = 0; i < 100; ++i) {
    auto sprite = Sprite::createWithSpriteFrameName("enemy.png");
    batch->addChild(sprite);  // 必须添加到 BatchNode
}
// 100 个 sprite → 1 次 Draw Call
```

### 自动批处理（3.x+）

Cocos2d-x 3.x 引入了**自动批处理**（Auto Batching），只要满足以下条件就会自动合并 Draw Call：

```
自动批处理条件：
1. 使用相同的纹理（Texture2D）
2. 使用相同的混合模式（BlendFunc）
3. 使用相同的 Shader（GLProgram）
4. 渲染顺序连续（中间没有其他纹理的节点打断）
```

```cpp
// 3.x+ 自动批处理 — 不需要 SpriteBatchNode
auto scene = Scene::create();

// 这些 Sprite 使用同一张图集，自动批处理
for (int i = 0; i < 100; ++i) {
    auto sprite = Sprite::createWithSpriteFrameName("enemy.png");
    sprite->setPosition(Vec2(rand() % 960, rand() % 640));
    scene->addChild(sprite);
}
// 100 个 sprite，只要纹理相同且连续 → 1 次 Draw Call

// ❌ 打断批处理的情况
auto bg = Sprite::create("background.png");   // 纹理 A
scene->addChild(bg, 0);                        // Draw Call #1

auto enemy1 = Sprite::createWithSpriteFrameName("enemy.png"); // 纹理 B
scene->addChild(enemy1, 1);                    // Draw Call #2

auto cloud = Sprite::create("cloud.png");      // 纹理 C（打断！）
scene->addChild(cloud, 2);                     // Draw Call #3

auto enemy2 = Sprite::createWithSpriteFrameName("enemy.png"); // 纹理 B
scene->addChild(enemy2, 3);                    // Draw Call #4（无法与 #2 合并）
```

优化策略：

```cpp
// ✅ 将相同纹理的节点安排在连续的 zOrder
auto bg = Sprite::create("background.png");
scene->addChild(bg, 0);

// 所有敌人连续排列
for (int i = 0; i < 50; ++i) {
    auto enemy = Sprite::createWithSpriteFrameName("enemy.png");
    scene->addChild(enemy, 1);  // 连续 zOrder → 自动批处理
}

auto cloud = Sprite::create("cloud.png");
scene->addChild(cloud, 2);  // 在敌人之后
// 结果：3 次 Draw Call（bg + enemies + cloud）
```

## 常见错误与解决方案

### 错误 1：SpriteFrame 名称不匹配

```cpp
// ❌ 使用了错误的帧名
auto sprite = Sprite::createWithSpriteFrameName("player_idle_1.png");
// 实际帧名可能是 "player_idle_01.png"（带前导零）

// ✅ 检查 plist 文件中的确切帧名
// 或使用 getSpriteFrameByName 返回值检查
auto frame = SpriteFrameCache::getInstance()
    ->getSpriteFrameByName("player_idle_1.png");
if (!frame) {
    CCLOG("ERROR: Frame 'player_idle_1.png' not found!");
    CCLOG("Did you mean 'player_idle_01.png'?");
}
```

### 错误 2：未加载 plist 就使用帧名

```cpp
// ❌ 忘记加载图集
auto sprite = Sprite::createWithSpriteFrameName("enemy_01.png");
// frame 为 nullptr → 显示白色方块或崩溃

// ✅ 确保在使用前加载 plist
SpriteFrameCache::getInstance()
    ->addSpriteFramesWithFile("enemies.plist");
auto sprite = Sprite::createWithSpriteFrameName("enemy_01.png");
```

### 错误 3：Draw Call 过多

```cpp
// ❌ 不同图集的帧交叉排列 → 频繁切换纹理
for (int i = 0; i < 50; ++i) {
    scene->addChild(Sprite::createWithSpriteFrameName("enemy.png"));  // 图集A
    scene->addChild(Sprite::createWithSpriteFrameName("bullet.png")); // 图集B
}
// 100 个 sprite → 100 次 Draw Call（A-B-A-B-A-B...）

// ✅ 按纹理分组
for (int i = 0; i < 50; ++i) {
    scene->addChild(
        Sprite::createWithSpriteFrameName("enemy.png"), 1);
}
for (int i = 0; i < 50; ++i) {
    scene->addChild(
        Sprite::createWithSpriteFrameName("bullet.png"), 2);
}
// 100 个 sprite → 2 次 Draw Call
```

## 对照项目源码

KrKr2 的 UI 资源使用 Cocos Studio 导出的 `.csb` 文件（二进制格式），这些文件内部引用了 SpriteFrame：

> **文件：** `ui/cocos-studio/ui/` 目录

```
ui/cocos-studio/ui/
├── CommonUI.csb           ← 通用 UI 组件
├── FileSelectScene.csb    ← 文件选择界面
├── GameMainMenu.csb       ← 游戏主菜单
├── GlobalSettingUI.csb    ← 全局设置
├── IndividualSettingUI.csb ← 个别设置
├── MessageBox.csb         ← 消息框
├── NaviBar.csb            ← 导航栏
├── TextInputDialog.csb    ← 文本输入对话框
└── ... (共 27 个 .csb 文件)
```

CSB 文件加载时，Cocos Studio 的 Reader 会自动从文件中读取 SpriteFrame 引用并创建对应的 UI 控件：

> **文件：** `cpp/core/environ/ui/BaseForm.cpp` 第 20-50 行

```cpp
// CSBReader 加载 UI
Node* CSBReader::Load(const std::string& csbFile) {
    // 内部解析 .csb 二进制格式
    // 自动加载引用的纹理和 SpriteFrame
    // 返回构建好的节点树
    return CSLoader::createNode(csbFile);
}
```

## 动手实践

### 实践：创建角色动画系统

```cpp
#include "cocos2d.h"
USING_NS_CC;

class AnimatedCharacter : public Sprite {
    std::string _currentAnim;
    
public:
    static AnimatedCharacter* create(const std::string& atlas) {
        auto character = new AnimatedCharacter();
        if (character && character->init()) {
            character->autorelease();
            // 加载图集
            SpriteFrameCache::getInstance()
                ->addSpriteFramesWithFile(atlas);
            return character;
        }
        CC_SAFE_DELETE(character);
        return nullptr;
    }
    
    // 播放动画
    void playAnimation(const std::string& name,
                       int frameCount,
                       float fps = 12.0f,
                       bool loop = true) {
        if (_currentAnim == name) return;  // 避免重复播放
        _currentAnim = name;
        
        stopAllActions();  // 停止当前动画
        
        Vector<SpriteFrame*> frames;
        for (int i = 1; i <= frameCount; ++i) {
            auto frameName = StringUtils::format(
                "%s_%02d.png", name.c_str(), i);
            auto frame = SpriteFrameCache::getInstance()
                ->getSpriteFrameByName(frameName);
            if (frame) frames.pushBack(frame);
        }
        
        if (frames.empty()) {
            CCLOG("Animation '%s' has no frames!", name.c_str());
            return;
        }
        
        auto animation = Animation::createWithSpriteFrames(
            frames, 1.0f / fps);
        auto animate = Animate::create(animation);
        
        if (loop) {
            runAction(RepeatForever::create(animate));
        } else {
            runAction(animate);
        }
    }
    
    // 切换动作
    void idle() { playAnimation("player_idle", 4, 8.0f); }
    void walk() { playAnimation("player_walk", 8, 12.0f); }
    void attack() { playAnimation("player_attack", 6, 16.0f, false); }
};

// 使用
auto player = AnimatedCharacter::create("characters.plist");
player->setPosition(Vec2(480, 320));
player->idle();  // 播放待机动画
scene->addChild(player);

// 切换动画
player->walk();    // 切换到行走
player->attack();  // 切换到攻击（单次播放）
```

## 本节小结

- **SpriteFrame** 是纹理中的一个矩形区域，包含位置、旋转、偏移等元数据
- **SpriteSheet**（图集）将多个小图打包到一张大纹理，通过 `.plist` 描述每帧位置
- **SpriteFrameCache** 管理 SpriteFrame 的缓存，通过帧名快速查找
- 图集的核心优势是**减少 Draw Call**：相同纹理的连续 Sprite 自动批处理
- Cocos2d-x 3.x 支持**自动批处理**，不再需要手动使用 SpriteBatchNode
- **TexturePacker** 是最常用的图集打包工具，支持 Cocos2d-x 格式导出
- KrKr2 的 UI 使用 Cocos Studio 导出的 `.csb` 文件，内部包含 SpriteFrame 引用

## 练习题与答案

### 题目 1：Draw Call 计算

以下场景有多少次 Draw Call？假设 hero.png 和 enemy.png 在同一图集（atlas_a.png），bullet.png 在另一图集（atlas_b.png）：

```cpp
scene->addChild(Sprite::create("background.png"), 0);     // 单独纹理
scene->addChild(Sprite::createWithSpriteFrameName("hero.png"), 1);
scene->addChild(Sprite::createWithSpriteFrameName("enemy.png"), 1);
scene->addChild(Sprite::create("cloud.png"), 2);           // 单独纹理
scene->addChild(Sprite::createWithSpriteFrameName("bullet.png"), 3);
scene->addChild(Sprite::createWithSpriteFrameName("hero.png"), 4);
```

<details>
<summary>查看答案</summary>

**5 次 Draw Call**

分析：
1. `background.png` — 单独纹理 → **Draw Call #1**
2. `hero.png` (atlas_a) — 新纹理 → **Draw Call #2**
3. `enemy.png` (atlas_a) — 同一纹理，连续 → **合并到 #2**
4. `cloud.png` — 不同纹理 → **Draw Call #3**
5. `bullet.png` (atlas_b) — 不同纹理 → **Draw Call #4**
6. `hero.png` (atlas_a) — 虽然和 #2 同纹理，但被 #3/#4 打断 → **Draw Call #5**

优化方案：将同一图集的节点安排在连续的 zOrder 中，避免被其他纹理的节点打断。

</details>

### 题目 2：手动创建走格动画

不使用图集打包工具，假设 `walk_sheet.png` 是一张 384×64 的图片（6 帧，每帧 64×64），编写代码创建循环行走动画。

<details>
<summary>查看答案</summary>

```cpp
#include "cocos2d.h"
USING_NS_CC;

// 加载纹理
auto texture = Director::getInstance()->getTextureCache()
    ->addImage("walk_sheet.png");

// 创建 6 个 SpriteFrame
Vector<SpriteFrame*> frames;
int frameWidth = 64;
int frameHeight = 64;
int frameCount = 6;

for (int i = 0; i < frameCount; ++i) {
    Rect rect(i * frameWidth, 0, frameWidth, frameHeight);
    auto frame = SpriteFrame::createWithTexture(texture, rect);
    frames.pushBack(frame);
    
    // 可选：注册到缓存方便复用
    auto name = StringUtils::format("walk_%02d", i);
    SpriteFrameCache::getInstance()->addSpriteFrame(frame, name);
}

// 创建 Sprite 并播放动画
auto character = Sprite::createWithSpriteFrame(frames.at(0));
character->setPosition(Vec2(480, 320));
scene->addChild(character);

auto animation = Animation::createWithSpriteFrames(frames, 0.1f);
// 0.1 秒/帧 = 10 FPS 动画
auto animate = Animate::create(animation);
character->runAction(RepeatForever::create(animate));
```

关键要点：
- `Rect(i * 64, 0, 64, 64)` — 每帧在纹理中的位置
- 纹理坐标以左上角为原点（注意与 GL 坐标系的区别）
- `Animation::createWithSpriteFrames(frames, 0.1f)` — 0.1 秒间隔 = 10 FPS

</details>

## 下一步

[03-Action动画系统](03-Action动画系统.md) — 学习 Cocos2d-x 的 Action 系统（移动、缩放、旋转、序列、组合等动画动作）。