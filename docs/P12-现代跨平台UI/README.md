# P12 — 现代跨平台 UI（Flutter / Compose Multiplatform）

> **所属系列：** P 系列（前置技术模块）
> **前置知识：** [P03-跨平台 C++ 开发](../P03-跨平台C++开发/README.md)、[P10-Cocos2d-x 框架](../P10-Cocos2d-x框架/README.md)（推荐）
> **后续模块：** [M13-UI 框架替换实战](../M13-UI框架替换实战/README.md)

## 模块简介

KrKr2 模拟器当前使用 Cocos2d-x 框架来构建用户界面（如设置菜单、文件浏览器、存档管理等）。Cocos2d-x 主要面向游戏渲染，其 UI 控件系统（Widget）功能有限、外观陈旧，不适合构建现代化的应用级 UI。本模块介绍两种主流的现代跨平台 UI 方案——Flutter（Google 推出的声明式 UI 框架，使用 Dart 语言，通过 Skia 引擎自绘 UI）和 Compose Multiplatform（JetBrains 基于 Android Jetpack Compose 扩展的跨平台 UI 框架，使用 Kotlin 语言），以及 Qt、Dear ImGui 等替代方案。

本模块的核心目标不是"学会写 Flutter/Compose 应用"，而是掌握 **如何将现代 UI 框架嵌入到 C++ 渲染引擎中**——这涉及引擎嵌入 API、跨语言通信（C++ ↔ Dart / C++ ↔ Kotlin）、纹理共享（让 UI 框架和 C++ 渲染引擎共享同一块 GPU 纹理）等技术难点。

## 术语预览

本模块将涉及以下术语，先有个印象，各章节中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 声明式 UI | Declarative UI | 用"描述最终界面长什么样"的方式编写 UI，框架自动处理更新——区别于"一步步命令如何改变界面"的命令式 UI |
| Flutter Engine | Flutter Engine | Flutter 框架的底层 C/C++ 运行时，负责 Dart 代码执行、Skia 渲染、平台事件处理 |
| Platform Channel | Platform Channel | Flutter 提供的跨语言通信通道，让 Dart 代码和 C++/Java/Swift 代码互相调用方法 |
| Texture Widget | Texture Widget | Flutter 中用于显示外部纹理（如来自 C++ 渲染引擎的画面）的控件 |
| Kotlin/Native | Kotlin/Native | Kotlin 的编译目标之一，可将 Kotlin 代码编译为原生机器码，无需 JVM |
| cinterop | C Interop | Kotlin/Native 提供的 C 语言互操作工具，让 Kotlin 直接调用 C/C++ 函数 |
| Skia | Skia | Google 开发的 2D 图形渲染引擎，被 Flutter、Chrome、Android 等项目使用 |
| FFI | Foreign Function Interface | 外部函数接口，允许一种编程语言调用另一种语言编写的函数 |
| 共享纹理 | Shared Texture | UI 框架和 C++ 引擎使用同一块 GPU 纹理内存，避免画面在 CPU 和 GPU 之间反复拷贝 |
| 事件桥接 | Event Bridge | 在 UI 框架和 C++ 引擎之间传递用户输入事件（触摸、键盘等）的机制 |

## 章节目录

| 章 | 节 | 主要内容 | 预计行数 |
|----|-----|---------|---------|
| 01-方案对比 | 01-主流跨平台UI框架总览 | Flutter、Compose Multiplatform、Qt、Dear ImGui 四种方案的架构、语言、渲染方式、社区生态对比 | 400+ |
| | 02-渲染方式与C++互操作能力对比 | 自绘 vs 原生控件；C++ 互操作能力评估；KrKr2 场景下的方案选型决策 | 400+ |
| 02-Flutter引擎嵌入 | 01-FlutterEngine-API与嵌入原理 | Flutter Engine 架构；嵌入 API（FlutterEngine/FlutterProjectArgs）；在 C++ 宿主中创建和管理 Flutter 引擎实例 | 450+ |
| | 02-Platform-Channel与纹理共享 | MethodChannel/EventChannel 原理与用法；Texture Widget 外部纹理注册；C++ 渲染引擎画面注入 Flutter | 450+ |
| 03-Compose-Multiplatform | 01-Kotlin-Native与Cinterop基础 | Kotlin/Native 编译模型；cinterop .def 文件；Kotlin 调用 C 函数；C 回调 Kotlin | 400+ |
| | 02-Skia渲染共享与平台集成 | Compose 的 Skia 渲染后端；与 C++ OpenGL 上下文共享；桌面端和移动端的集成差异 | 400+ |
| 04-UI与渲染引擎分离 | 01-抽象UI接口层设计 | 接口定义原则；IUIManager/IUIView/IUIEvent 抽象；渲染后端作为纹理提供者的模式 | 400+ |
| | 02-事件桥接与共享纹理 | 输入事件双向传递；OpenGL 纹理共享（EGLImage/IOSurface/DXGI）；帧同步策略 | 400+ |
| 05-实战设计 | 01-KrKr2-UI抽象层接口定义 | 基于 KrKr2 现有 UI 需求定义完整的 IKrKr2UI 接口；Cocos2d 实现类 | 450+ |
| | 02-Cocos2d实现与Flutter原型 | 用现有 Cocos2d 实现接口作为参考；用 Flutter 实现同一接口的原型；性能与体验对比 | 450+ |

## 学习路线

```
01-方案对比（了解选项和决策依据）
    ↓
02-Flutter引擎嵌入（深入方案 A）
    ↓
03-Compose-Multiplatform（深入方案 B）
    ↓
04-UI与渲染引擎分离（与具体框架无关的架构设计）
    ↓
05-实战设计（应用到 KrKr2 项目）
    ↓
→ M13-UI框架替换实战（真正动手替换）
```

## 目标读者

- 已掌握 C++ 跨平台开发基础（P03）
- 了解 Cocos2d-x 框架基本用法（P10，推荐但非强制）
- 不需要事先会 Flutter/Dart 或 Kotlin——本模块从零讲解
- 希望了解如何在 C++ 项目中集成现代 UI 框架
