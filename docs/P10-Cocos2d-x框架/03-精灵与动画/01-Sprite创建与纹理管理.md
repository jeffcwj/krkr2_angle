# Sprite 创建与纹理管理

> **所属模块：** P10-Cocos2d-x 框架
> **前置知识：** [02-场景与节点树](../02-场景与节点树/)、[P04-OpenGL图形编程](../../P04-OpenGL图形编程/)（纹理基础）
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 掌握 Sprite 的多种创建方式（文件、纹理、矩形区域）
2. 理解 TextureCache 的纹理缓存机制和生命周期
3. 使用异步加载避免主线程阻塞
4. 理解纹理像素格式（RGBA8888 / RGBA4444 / RGB565 等）对内存的影响
5. 对照 KrKr2 源码理解 DrawSprite 的自定义纹理更新机制

## Sprite 基础

### 什么是 Sprite

`Sprite`（精灵）是 Cocos2d-x 中最常用的可视化节点，它将一张**纹理**（Texture）或纹理的一个**矩形区域**映射到一个四边形（Quad）上渲染。几乎所有游戏中看到的图像元素（角色、背景、UI 图标、子弹）都是 Sprite：

```
Sprite 的本质：

纹理（GPU 内存中的图像）
┌──────────────────────┐
│  ████████████████████ │
│  █ Player Idle  █████ │  ← 纹理可以包含多个图像
│  ████████████████████ │     Sprite 只显示其中一个矩形区域
│  █ Player Walk  █████ │
│  ████████████████████ │
└──────────────────────┘
         ↓ 采样区域 (Rect)
┌────────────┐
│  Sprite    │ → 渲染为屏幕上的四边形
│  (Quad)    │    顶点 + UV 坐标 + 颜色
└────────────┘
```

### 创建 Sprite 的多种方式

#### 方式 1：从图片文件创建（最常用）

```cpp
#include "cocos2d.h"
USING_NS_CC;

// 从 PNG/JPG/WebP 等文件创建
auto sprite = Sprite::create("player.png");
sprite->setPosition(Vec2(480, 320));
scene->addChild(sprite);

// 文件搜索路径：FileUtils 按搜索路径依次查找
// 默认搜索路径包括 Resources 目录
FileUtils::getInstance()->addSearchPath("images/");
auto sprite2 = Sprite::create("enemy.png");  // 在 images/ 下查找
```

> **注意：** 如果文件不存在，`Sprite::create()` 返回一个**空白白色方块**（不是 nullptr）。Cocos2d-x 3.x 不会返回 nullptr，而是使用默认纹理。检查方法：
> ```cpp
> auto sprite = Sprite::create("nonexistent.png");
> // sprite != nullptr，但显示为白色方块
> // 可通过检查纹理判断
> if (sprite->getTexture() == Director::getInstance()
>         ->getTextureCache()->getTextureForKey("/cc_2x2_white_image")) {
>     CCLOG("Warning: texture not found!");
> }
> ```

#### 方式 2：从纹理对象创建

```cpp
// 先获取纹理，再创建 Sprite
auto texture = Director::getInstance()->getTextureCache()
    ->addImage("spritesheet.png");

// 从整张纹理创建
auto sprite = Sprite::createWithTexture(texture);

// 从纹理的指定区域创建
Rect region(0, 0, 64, 64);  // 取左上角 64×64 区域
auto sprite2 = Sprite::createWithTexture(texture, region);
```

#### 方式 3：从矩形区域创建

```cpp
// 从文件的指定区域创建
Rect rect(128, 0, 64, 64);  // x=128, y=0, w=64, h=64
auto sprite = Sprite::create("tileset.png", rect);

// 注意：rect 的坐标是以纹理左上角为原点的像素坐标
// (0,0) = 纹理左上角
// 这与 Cocos2d-x 的 GL 坐标系（左下角原点）不同！
```

#### 方式 4：创建纯色 Sprite

```cpp
// 创建纯色矩形（不需要图片文件）
auto whiteSprite = Sprite::create();
whiteSprite->setTextureRect(Rect(0, 0, 100, 50));
whiteSprite->setColor(Color3B::RED);
// 显示为 100×50 的红色矩形
scene->addChild(whiteSprite);
```

### Sprite 关键属性

```cpp
auto sprite = Sprite::create("player.png");

// 位置与变换（继承自 Node）
sprite->setPosition(Vec2(480, 320));
sprite->setScale(2.0f);
sprite->setRotation(45.0f);

// 锚点 — 默认 (0.5, 0.5) 即中心点
sprite->setAnchorPoint(Vec2(0.5f, 0.5f));  // 中心
sprite->setAnchorPoint(Vec2(0, 0));         // 左下角
sprite->setAnchorPoint(Vec2(1, 1));         // 右上角

// 翻转
sprite->setFlippedX(true);   // 水平翻转
sprite->setFlippedY(true);   // 垂直翻转

// 颜色与透明度
sprite->setColor(Color3B(255, 128, 0));  // 橙色色调
sprite->setOpacity(128);                  // 半透明 (0-255)

// 混合模式
sprite->setBlendFunc(BlendFunc::ADDITIVE);    // 加法混合（发光效果）
sprite->setBlendFunc(BlendFunc::ALPHA_NON_PREMULTIPLIED);  // 非预乘 Alpha

// 内容大小
auto size = sprite->getContentSize();  // 纹理区域的像素大小
CCLOG("Sprite size: %.0f × %.0f", size.width, size.height);
```

混合模式对比：

| BlendFunc | 效果 | 适用场景 |
|-----------|------|----------|
| `ALPHA_PREMULTIPLIED`（默认） | 预乘 Alpha 混合 | 大多数普通精灵 |
| `ALPHA_NON_PREMULTIPLIED` | 标准 Alpha 混合 | 非预乘 Alpha 纹理 |
| `ADDITIVE` | 加法混合（src + dst） | 火焰、光效、粒子 |
| `DISABLE` | 不混合（直接覆盖） | 不透明全屏背景 |

## TextureCache 纹理缓存

### 缓存机制

`TextureCache` 是 Cocos2d-x 的纹理缓存管理器。当你第一次通过 `Sprite::create("image.png")` 加载图片时，TextureCache 会：
1. 从磁盘读取图片文件
2. 解码为像素数据（CPU 内存）
3. 上传到 GPU 创建 OpenGL 纹理
4. 以文件路径为 key 缓存纹理对象
5. 后续再用同一路径创建 Sprite 时，直接从缓存取出（不重复加载）

```cpp
auto cache = Director::getInstance()->getTextureCache();

// 预加载纹理（同步，阻塞主线程）
auto tex = cache->addImage("player.png");
CCLOG("Texture size: %d × %d",
      tex->getPixelsWide(), tex->getPixelsHigh());

// 检查纹理是否已在缓存中
auto cached = cache->getTextureForKey("player.png");
if (cached) {
    CCLOG("Texture already cached!");
}

// Sprite::create 内部使用 TextureCache
auto sprite1 = Sprite::create("player.png");  // 第一次：加载+缓存
auto sprite2 = Sprite::create("player.png");  // 第二次：直接从缓存取
// sprite1 和 sprite2 共享同一个 Texture2D 对象
CCLOG("Same texture: %s",
      sprite1->getTexture() == sprite2->getTexture() ? "YES" : "NO");
// 输出：Same texture: YES
```

### 异步加载

同步加载大纹理会阻塞主线程导致卡顿。使用 `addImageAsync()` 在后台线程加载：

```cpp
auto cache = Director::getInstance()->getTextureCache();

// 异步加载（不阻塞主线程）
cache->addImageAsync("large_background.png",
    [](Texture2D* texture) {
        if (texture) {
            CCLOG("Loaded: %d × %d",
                  texture->getPixelsWide(),
                  texture->getPixelsHigh());
            
            // 在回调中创建 Sprite（主线程安全）
            auto sprite = Sprite::createWithTexture(texture);
            // ... 添加到场景
        }
    });

// 批量异步加载
std::vector<std::string> textures = {
    "bg.png", "tileset.png", "characters.png",
    "effects.png", "ui_atlas.png"
};

int loaded = 0;
int total = static_cast<int>(textures.size());

for (const auto& path : textures) {
    cache->addImageAsync(path,
        [&loaded, total](Texture2D*) {
            loaded++;
            CCLOG("Progress: %d/%d", loaded, total);
            if (loaded >= total) {
                CCLOG("All textures loaded!");
            }
        });
}
```

### 缓存清理

```cpp
auto cache = Director::getInstance()->getTextureCache();

// 移除单个纹理（通过 key）
cache->removeTextureForKey("old_level.png");

// 移除单个纹理（通过对象）
cache->removeTexture(texture);

// 移除所有未使用的纹理（引用计数 == 1 的纹理，即只有缓存持有）
cache->removeUnusedTextures();

// 移除所有纹理（慎用！会导致所有 Sprite 变白）
cache->removeAllTextures();

// 获取缓存状态
CCLOG("Cache description:\n%s",
      cache->getCachedTextureInfo().c_str());
// 输出示例：
// "player.png" rc=3 id=1 512 x 512 @ 32 bpp => 1048576 KB
// "bg.png" rc=2 id=2 1024 x 768 @ 32 bpp => 3145728 KB
```

> **最佳实践：** 在场景切换时调用 `removeUnusedTextures()` 释放旧场景不再使用的纹理，防止内存持续增长。

### 纹理像素格式与内存优化

```cpp
// 设置默认像素格式（影响之后所有新加载的纹理）
Texture2D::setDefaultAlphaPixelFormat(
    backend::PixelFormat::RGBA4444);

// 加载纹理时使用新的默认格式
auto tex = cache->addImage("ui_icons.png");
// 加载完成后恢复默认
Texture2D::setDefaultAlphaPixelFormat(
    backend::PixelFormat::RGBA8888);
```

像素格式对比：

| 格式 | 每像素位数 | 相对内存 | 质量 | 适用场景 |
|------|-----------|----------|------|----------|
| RGBA8888 | 32 bpp | 100% | 最高 | 高质量图片（默认） |
| RGBA4444 | 16 bpp | 50% | 中等 | UI 图标、简单图形 |
| RGB565 | 16 bpp | 50% | 中等（无 Alpha） | 不透明背景图 |
| RGB888 | 24 bpp | 75% | 高（无 Alpha） | 不透明照片 |
| RGBA5551 | 16 bpp | 50% | 中等（1位 Alpha） | 简单透明/不透明 |
| A8 | 8 bpp | 25% | 仅透明度 | 字体、遮罩 |
| I8 | 8 bpp | 25% | 灰度 | 灰度图 |

内存计算：

```
纹理内存 = 宽 × 高 × 每像素字节数

例：1024×1024 纹理
- RGBA8888: 1024 × 1024 × 4 = 4,194,304 bytes = 4 MB
- RGBA4444: 1024 × 1024 × 2 = 2,097,152 bytes = 2 MB
- RGB565:   1024 × 1024 × 2 = 2,097,152 bytes = 2 MB
- A8:       1024 × 1024 × 1 = 1,048,576 bytes = 1 MB
```

## 常见错误与解决方案

### 错误 1：纹理黑边（非 2 的幂次纹理）

```cpp
// 旧版 OpenGL ES 要求纹理尺寸是 2 的幂次（如 64, 128, 256, 512, 1024）
// Cocos2d-x 3.x 默认支持 NPOT（Non-Power-Of-Two），但某些设备可能出问题

// ❌ 可能出现黑边或拉伸
auto sprite = Sprite::create("100x75_photo.png");

// ✅ 检查 GPU 是否支持 NPOT
bool supportsNPOT = Configuration::getInstance()
    ->supportsNPOT();
CCLOG("NPOT support: %s", supportsNPOT ? "YES" : "NO");

// 如果不支持，将纹理尺寸调整为 2 的幂次
// 或在纹理参数中设置 clamp 模式
Texture2D::TexParams params;
params.minFilter = backend::SamplerFilter::LINEAR;
params.magFilter = backend::SamplerFilter::LINEAR;
params.sAddressMode = backend::SamplerAddressMode::CLAMP_TO_EDGE;
params.tAddressMode = backend::SamplerAddressMode::CLAMP_TO_EDGE;
sprite->getTexture()->setTexParameters(params);
```

### 错误 2：纹理重复加载浪费内存

```cpp
// ❌ 错误理解：认为每次 create 都会重新加载
for (int i = 0; i < 100; ++i) {
    auto sprite = Sprite::create("bullet.png");
    // 不用担心！TextureCache 保证只加载一次
    // 100 个 sprite 共享同一个 Texture2D
}

// ❌ 真正的浪费：同一图片用不同路径加载
auto s1 = Sprite::create("images/player.png");
auto s2 = Sprite::create("./images/player.png");
auto s3 = Sprite::create("../res/images/player.png");
// 三次加载了三份纹理！因为路径字符串不同

// ✅ 正确：统一使用相同的路径字符串
// 或使用 FileUtils::fullPathForFilename() 规范化路径
auto fullPath = FileUtils::getInstance()
    ->fullPathForFilename("player.png");
```

### 错误 3：场景切换后纹理未释放

```cpp
// ❌ 场景切换后旧纹理仍在缓存中
void GameScene::onExit() {
    Scene::onExit();
    // 忘记清理纹理 → 内存泄漏
}

// ✅ 在场景退出时清理未使用的纹理
void GameScene::onExit() {
    Scene::onExit();
    Director::getInstance()->getTextureCache()
        ->removeUnusedTextures();
    // 只有引用计数为 1（仅缓存持有）的纹理会被清理
    // 正在使用的纹理不受影响
}
```

## 对照项目源码

KrKr2 中的 `TVPWindowLayer` 使用了一种特殊的纹理更新方式 — 它不是从图片文件创建 Sprite，而是**每帧动态更新纹理内容**：

> **文件：** `cpp/core/environ/cocos2d/MainScene.cpp` 第 365-500 行

```cpp
// TVPWindowLayer 中的 DrawSprite（简化）
class DrawSprite : public Sprite {
    // 持有一个可写纹理
    Texture2D* _drawTexture = nullptr;
    unsigned char* _pixelData = nullptr;  // CPU 端像素缓冲区
    int _texWidth, _texHeight;
    
public:
    void initWithSize(int w, int h) {
        _texWidth = w;
        _texHeight = h;
        _pixelData = new unsigned char[w * h * 4];  // RGBA
        
        // 创建空纹理
        _drawTexture = new Texture2D();
        _drawTexture->initWithData(
            _pixelData, w * h * 4,
            backend::PixelFormat::RGBA8888,
            w, h, Size(w, h));
        
        setTexture(_drawTexture);
        setTextureRect(Rect(0, 0, w, h));
    }
    
    // KiriKiri 引擎渲染后，将像素数据上传到 GPU
    void updateTexture(const void* data, int x, int y,
                       int w, int h) {
        // 局部更新纹理（只更新变化的矩形区域）
        glBindTexture(GL_TEXTURE_2D,
                      _drawTexture->getName());
        glTexSubImage2D(GL_TEXTURE_2D, 0,
                        x, y, w, h,
                        GL_RGBA, GL_UNSIGNED_BYTE, data);
    }
};
```

这种技术的要点：
1. **不使用 TextureCache**：DrawSprite 直接管理自己的纹理，不经过缓存系统
2. **glTexSubImage2D 局部更新**：只更新变化的区域，避免每帧上传整张纹理
3. **CPU→GPU 数据流**：KiriKiri 引擎在 CPU 端完成渲染，结果通过纹理上传显示

## 动手实践

### 实践：纹理管理器性能测试

```cpp
#include "cocos2d.h"
USING_NS_CC;

class TextureTestScene : public Scene {
    Label* _infoLabel = nullptr;
    
public:
    CREATE_FUNC(TextureTestScene);
    
    bool init() override {
        if (!Scene::init()) return false;
        
        auto size = Director::getInstance()->getVisibleSize();
        
        _infoLabel = Label::createWithSystemFont("", "Arial", 16);
        _infoLabel->setAnchorPoint(Vec2(0, 1));
        _infoLabel->setPosition(Vec2(10, size.height - 10));
        addChild(_infoLabel, 100);
        
        // 测试 1：创建大量共享纹理的 Sprite
        testSharedTexture();
        
        // 测试 2：显示缓存信息
        updateCacheInfo();
        
        return true;
    }
    
    void testSharedTexture() {
        auto size = Director::getInstance()->getVisibleSize();
        
        // 创建 100 个共享同一纹理的 Sprite
        for (int i = 0; i < 100; ++i) {
            auto sprite = Sprite::create("icon.png");
            sprite->setPosition(Vec2(
                rand() % (int)size.width,
                rand() % (int)size.height));
            sprite->setScale(0.5f + (rand() % 100) / 100.0f);
            sprite->setRotation(rand() % 360);
            addChild(sprite);
        }
        
        // 所有 100 个 sprite 共享一个纹理 → GPU 内存只占一份
    }
    
    void updateCacheInfo() {
        auto cache = Director::getInstance()->getTextureCache();
        auto info = cache->getCachedTextureInfo();
        _infoLabel->setString("Texture Cache:\n" + info);
    }
};
```

## 本节小结

- **Sprite** 是最常用的渲染节点，将纹理映射到四边形上显示
- 创建方式：从文件（`create`）、从纹理（`createWithTexture`）、从区域（`create` + Rect）、纯色（`setTextureRect`）
- **TextureCache** 自动缓存已加载的纹理，同一路径只加载一次，多个 Sprite 共享纹理
- **addImageAsync** 异步加载纹理，避免主线程阻塞造成卡顿
- 像素格式影响内存占用：RGBA4444（50%）和 RGB565（50%）可显著节省内存
- 场景切换时调用 `removeUnusedTextures()` 清理不再使用的纹理
- KrKr2 使用 `glTexSubImage2D` 实现纹理局部更新，将 KiriKiri 引擎的 CPU 渲染结果上传到 GPU

## 练习题与答案

### 题目 1：纹理内存计算

一个游戏有以下纹理需求：
- 背景图：1920×1080，不透明
- 角色图集：2048×2048，需要透明度
- UI 图标：512×512，需要透明度但质量要求不高

请计算使用最优像素格式时的总内存占用，并与全部使用 RGBA8888 时对比。

<details>
<summary>查看答案</summary>

**最优格式选择：**

| 纹理 | 尺寸 | 最优格式 | 原因 |
|------|------|----------|------|
| 背景图 | 1920×1080 | RGB565（16bpp） | 不需要透明度 |
| 角色图集 | 2048×2048 | RGBA8888（32bpp） | 需要高质量+透明度 |
| UI 图标 | 512×512 | RGBA4444（16bpp） | 质量要求不高 |

**计算：**

全部 RGBA8888：
```
背景：1920 × 1080 × 4 = 8,294,400 bytes ≈ 7.9 MB
角色：2048 × 2048 × 4 = 16,777,216 bytes = 16.0 MB
UI：  512 × 512 × 4  = 1,048,576 bytes = 1.0 MB
总计：24.9 MB
```

最优格式：
```
背景（RGB565）：1920 × 1080 × 2 = 4,147,200 bytes ≈ 3.95 MB
角色（RGBA8888）：2048 × 2048 × 4 = 16,777,216 bytes = 16.0 MB
UI（RGBA4444）：512 × 512 × 2 = 524,288 bytes = 0.5 MB
总计：20.45 MB
```

节省：24.9 - 20.45 = **4.45 MB（节省约 18%）**

代码实现：
```cpp
auto cache = Director::getInstance()->getTextureCache();

// 背景用 RGB565
Texture2D::setDefaultAlphaPixelFormat(backend::PixelFormat::RGB565);
cache->addImage("background.png");

// 角色用 RGBA8888
Texture2D::setDefaultAlphaPixelFormat(backend::PixelFormat::RGBA8888);
cache->addImage("characters.png");

// UI 用 RGBA4444
Texture2D::setDefaultAlphaPixelFormat(backend::PixelFormat::RGBA4444);
cache->addImage("ui_icons.png");

// 恢复默认
Texture2D::setDefaultAlphaPixelFormat(backend::PixelFormat::RGBA8888);
```

</details>

### 题目 2：实现纹理预加载管理器

编写一个简单的纹理预加载管理器，要求：
1. 支持批量添加纹理路径
2. 异步加载所有纹理
3. 提供进度回调
4. 加载完成后的回调

<details>
<summary>查看答案</summary>

```cpp
#include "cocos2d.h"
#include <vector>
#include <functional>
USING_NS_CC;

class TexturePreloader {
public:
    using ProgressCallback = std::function<void(int loaded, int total)>;
    using CompleteCallback = std::function<void()>;
    
    void addTexture(const std::string& path) {
        _paths.push_back(path);
    }
    
    void addTextures(const std::vector<std::string>& paths) {
        _paths.insert(_paths.end(), paths.begin(), paths.end());
    }
    
    void startLoading(ProgressCallback onProgress,
                      CompleteCallback onComplete) {
        _total = static_cast<int>(_paths.size());
        _loaded = 0;
        _onProgress = onProgress;
        _onComplete = onComplete;
        
        if (_total == 0) {
            if (_onComplete) _onComplete();
            return;
        }
        
        auto cache = Director::getInstance()->getTextureCache();
        for (const auto& path : _paths) {
            cache->addImageAsync(path,
                [this](Texture2D* tex) {
                    _loaded++;
                    if (_onProgress) {
                        _onProgress(_loaded, _total);
                    }
                    if (_loaded >= _total && _onComplete) {
                        _onComplete();
                    }
                });
        }
    }
    
    // 清理所有预加载的纹理
    void unloadAll() {
        auto cache = Director::getInstance()->getTextureCache();
        for (const auto& path : _paths) {
            cache->removeTextureForKey(path);
        }
        _paths.clear();
    }
    
private:
    std::vector<std::string> _paths;
    int _total = 0;
    int _loaded = 0;
    ProgressCallback _onProgress;
    CompleteCallback _onComplete;
};

// 使用示例
void MyScene::preloadAssets() {
    auto preloader = new TexturePreloader();
    preloader->addTextures({
        "level1/bg.png", "level1/tileset.png",
        "level1/enemies.png", "level1/items.png"
    });
    
    preloader->startLoading(
        // 进度回调
        [](int loaded, int total) {
            float pct = 100.0f * loaded / total;
            CCLOG("Loading: %.0f%% (%d/%d)", pct, loaded, total);
        },
        // 完成回调
        [preloader]() {
            CCLOG("All textures loaded!");
            delete preloader;  // 清理预加载器
        });
}
```

</details>

### 题目 3：解释 KrKr2 的 DrawSprite 为什么不用 TextureCache

KrKr2 的 DrawSprite 直接创建 Texture2D 而不通过 TextureCache。请解释原因，以及如果使用 TextureCache 会有什么问题。

<details>
<summary>查看答案</summary>

**不使用 TextureCache 的原因：**

1. **纹理内容动态变化：** DrawSprite 的纹理每帧都可能被 KiriKiri 引擎更新（通过 `glTexSubImage2D`）。TextureCache 以文件路径为 key 缓存纹理，但 DrawSprite 的纹理没有对应的文件路径 — 它的内容来自 CPU 端的实时渲染

2. **纹理不可共享：** TextureCache 的设计假设多个 Sprite 可以共享同一个纹理。但 DrawSprite 的纹理是该窗口独有的（每个 TVPWindowLayer 有自己的渲染内容），共享没有意义

3. **生命周期不同：** TextureCache 中的纹理在整个场景存在期间保持有效。DrawSprite 的纹理需要与窗口同生同灭，由 TVPWindowLayer 直接管理更合适

4. **性能考虑：** TextureCache 的 `addImage` 会从磁盘读取文件 → 解码 → 上传 GPU。DrawSprite 跳过了前两步，直接用内存数据创建纹理，再通过 `glTexSubImage2D` 局部更新

**如果强行使用 TextureCache：**
- 需要生成一个虚假的 key（如 "window_layer_0"）来注册纹理
- 每帧更新纹理内容时，缓存无法感知变化 → 可能返回过期的纹理
- `removeUnusedTextures()` 可能误删正在使用的动态纹理
- 增加不必要的间接层，降低性能

</details>

## 下一步

[02-SpriteFrame与SpriteSheet](02-SpriteFrame与SpriteSheet.md) — 学习精灵帧、图集打包和 SpriteFrameCache 的使用。