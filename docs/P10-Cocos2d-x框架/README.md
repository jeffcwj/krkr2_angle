# P10 — Cocos2d-x 框架

> **前置依赖：** P04-OpenGL 图形编程、P03-跨平台 C++ 开发
> **目标：** 掌握 Cocos2d-x 3.x 的核心架构与 API，能读懂并修改 KrKr2 的 Cocos2d 集成层

## 模块概述

KrKr2 使用 Cocos2d-x 3.17.2 作为跨平台渲染与 UI 框架。Cocos2d-x 负责 OpenGL 上下文管理、场景图渲染、UI 组件、触摸/键盘事件分发、资源加载等基础设施。KrKr2 在此基础上构建了自己的窗口桥接层（`TVPWindowLayer`）、UI 表单系统（`iTVPBaseForm`）和 YUV 视频渲染精灵（`TVPYUVSprite`），将 KiriKiri 引擎的渲染输出嵌入 Cocos2d-x 的场景图中。

本模块从 Cocos2d-x 的架构原理讲起，逐步深入到节点树、精灵、动画、UI 组件，最后通过实战章节分析 KrKr2 如何将 KiriKiri 引擎与 Cocos2d-x 框架集成。

## 学习路径

```
Cocos2d-x 架构（Director / 主循环 / 内存管理）
  │
  ▼
场景与节点树（坐标系 / 父子关系 / zOrder / 场景切换）
  │
  ▼
精灵与动画（Sprite / SpriteFrame / Action 系统）
  │
  ▼
UI 组件与布局（Label / Button / CSB 文件 / 事件监听）
  │
  ▼
实战：分析 KrKr2 的 Cocos2d 集成
  ├── AppDelegate 与 Scene 初始化
  ├── 渲染桥接与窗口层（TVPWindowLayer）
  └── UI 表单系统与菜单（iTVPBaseForm / GameMainMenu）
```

## 章节目录

### [01 — Cocos2d-x 架构](01-Cocos2d-x架构/)

| 节 | 标题 | 内容 |
|----|------|------|
| 01 | Director 与主循环 | Director 单例模式、主循环流程（输入→逻辑→渲染）、帧调度与 deltaTime |
| 02 | Scene / Layer / Node 树 | 场景图核心类层次、Node 基类属性、Layer 与 Scene 的职责划分 |
| 03 | 内存管理与引用计数 | Ref 基类、autorelease 池、create() 工厂模式、循环引用防范 |

### [02 — 场景与节点树](02-场景与节点树/)

| 节 | 标题 | 内容 |
|----|------|------|
| 01 | 坐标系与变换矩阵 | 右手坐标系（左下角原点）、世界/本地坐标转换、Mat4 变换矩阵 |
| 02 | 父子关系与 zOrder | addChild / removeChild、localZOrder / globalZOrder、渲染顺序算法 |
| 03 | 场景切换与过渡 | Director::replaceScene / pushScene / popScene、TransitionScene 动画 |

### [03 — 精灵与动画](03-精灵与动画/)

| 节 | 标题 | 内容 |
|----|------|------|
| 01 | Sprite 创建与纹理管理 | Sprite::create 系列方法、Texture2D 与 TextureCache、像素格式 |
| 02 | SpriteFrame 与 SpriteSheet | SpriteFrameCache、plist 图集、TexturePacker 工作流 |
| 03 | Action 动画系统 | 即时/持续动作、Sequence / Spawn / RepeatForever、自定义 Action |

### [04 — UI 组件与布局](04-UI组件与布局/)

| 节 | 标题 | 内容 |
|----|------|------|
| 01 | Label / Button / Layout 基础 | ui::Text / ui::Button / ui::Layout 创建与配置、九宫格拉伸 |
| 02 | CSB 文件加载与 Cocos Studio | CSLoader::createNode、.csb 二进制格式、节点名称查找 |
| 03 | 事件监听与分发 | EventDispatcher 机制、触摸/键盘/自定义事件、事件优先级 |

### [05 — 实战：分析 KrKr2 的 Cocos2d 集成](05-实战-分析KrKr2的Cocos2d集成/)

| 节 | 标题 | 内容 |
|----|------|------|
| 01 | AppDelegate 与 Scene 初始化 | 启动流程、GLView 创建、设计分辨率、TVPMainScene 构建 |
| 02 | 渲染桥接与窗口层 | TVPWindowLayer 架构、DrawSprite 渲染、触摸→鼠标事件转换 |
| 03 | UI 表单系统与菜单 | iTVPBaseForm / CSBReader、pushUIForm/popUIForm 动画栈、GameMainMenu |

## 术语预览

本模块涉及大量 Cocos2d-x 框架术语，先在这里建立印象，各章正文会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| Director | — | Cocos2d-x 的"总导演"单例——负责驱动主循环、管理场景切换和帧调度 |
| Scene | — | 一个完整的"画面"，包含所有要显示的节点；同一时刻只有一个 Scene 在运行 |
| Node | — | 场景图的基本构建块——所有可见和不可见的对象都继承自 Node，拥有位置、大小、子节点列表 |
| Layer | — | 一种特殊的 Node，用于按层组织内容（如背景层、UI 层），Cocos2d-x 3.x 中已弱化 |
| Sprite | 精灵 | 显示一张图片的 Node，是游戏中最常用的可见元素 |
| SpriteFrame | 精灵帧 | 大图集（SpriteSheet）中的一个矩形区域，多个 SpriteFrame 共享一张纹理以减少 GPU 切换 |
| Action | 动作 | 让 Node 动起来的对象——移动、旋转、淡入淡出等动画效果都通过 Action 实现 |
| zOrder | — | 节点的绘制顺序值——zOrder 越大越后绘制（覆盖在上面），用于控制谁挡住谁 |
| autorelease | — | Cocos2d-x 的内存管理机制——对象创建后放入自动释放池，帧末自动减引用计数 |
| CSB | Cocos Studio Binary | Cocos Studio 导出的 UI 布局二进制文件，KrKr2 用 CSLoader 加载它来构建菜单和表单界面 |
| EventDispatcher | 事件分发器 | Cocos2d-x 的事件系统中枢——管理触摸、键盘、自定义事件的监听和分发 |
| Mat4 | 4×4 Matrix | 4×4 变换矩阵——用于表示节点的位移、旋转、缩放，是坐标系变换的数学基础 |

## 与其他模块的关系

- **P04-OpenGL 图形编程** — Cocos2d-x 底层使用 OpenGL ES 2.0 / 3.0，理解 GL 管线有助于读懂渲染代码
- **P03-跨平台 C++ 开发** — Cocos2d-x 本身是跨平台框架，其平台抽象设计值得对照学习
- **M04-渲染子系统** — KrKr2 的渲染层直接构建在 Cocos2d-x 之上，本模块是 M04 的直接前置
- **M03-平台抽象层解析** — KrKr2 的平台抽象层中 cocos2d 路径是最主要的实现

## 适合谁学

- 需要理解 KrKr2 渲染架构的开发者
- 想修改或扩展 KrKr2 UI 界面的开发者
- 有 OpenGL 基础但未接触过 Cocos2d-x 的 C++ 开发者

## 学习建议

1. 建议先完成 P04（OpenGL）再学习本模块，了解着色器和渲染管线基础
2. 安装 Cocos2d-x 3.17 源码作为参考（KrKr2 使用的版本）
3. 第 05 章实战部分需要对照 KrKr2 源码阅读，建议打开 IDE 同步浏览
4. Cocos2d-x 3.x 的 API 与 4.x 有较大差异，本教程专注于 3.x 版本
5. 重点理解场景图和节点树概念——这是理解 KrKr2 渲染桥接层的关键

## 开始学习

→ [01-Cocos2d-x架构/01-Director与主循环.md](01-Cocos2d-x架构/01-Director与主循环.md)
