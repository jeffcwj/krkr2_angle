# M13 — UI 框架替换实战

> **所属系列：** 项目模块（M 系列）
> **前置模块：** [M03-平台抽象层解析](../M03-平台抽象层解析/)、[M04-渲染子系统](../M04-渲染子系统/)、[P12-现代跨平台UI](../P12-现代跨平台UI/)
> **难度：** ★★★★☆
> **预计学习时间：** 12-18 小时

## 模块简介

KrKr2 目前使用 **Cocos2d-x** 作为渲染引擎，其 UI 层（对话框、菜单、设置页、文件选择器等）均通过 Cocos2d-x 的 `cocos2d::ui::` 组件实现，资源文件存放在 `ui/cocos-studio/` 目录下。

随着移动端和桌面端 UI 框架的演进，Cocos2d-x 的 UI 组件已逐渐落后。**Flutter** 和 **Jetpack Compose** 等现代 UI 框架提供了更好的开发体验、更丰富的组件库和更强的跨平台能力。本模块将指导你：

1. **深入理解**现有 Cocos2d-x UI 层的架构与事件流
2. **设计平台无关的 UI 抽象接口**，解耦渲染引擎与 UI 实现
3. **在 C++ 项目中嵌入 Flutter 引擎**，通过 Platform Channel 实现双向通信
4. **集成 Compose Multiplatform**，通过 Kotlin/Native + cinterop 与 C++ 层交互
5. **实现渲染纹理桥接**，让 Flutter/Compose Widget 与 Cocos2d-x 渲染无缝共存
6. **制定并执行渐进式迁移策略**，保证业务连续性

本模块是整个教程体系的**最终实战模块**，综合运用了 M03（平台抽象层）、M04（渲染子系统）和 P12（现代跨平台 UI）的全部知识。

## 术语预览

本模块将涉及以下核心术语，先建立印象，正文中会逐一深讲：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 平台通道 | Platform Channel | Flutter 与宿主（C++/Android/iOS）之间的消息传递机制，类似进程间通信 |
| 纹理共享 | Texture Sharing | 把 Flutter 渲染结果作为 OpenGL 纹理提供给 Cocos2d-x 使用，避免两套渲染器各画各的 |
| 外部纹理 | External Texture | Flutter 引擎提供的接口，允许宿主程序向 Flutter 注入由外部 GPU 资源生成的纹理帧 |
| cinterop | C Interop | Kotlin/Native 的 C 语言互操作工具，自动从 .h 头文件生成 Kotlin 绑定代码 |
| 渐进式迁移 | Incremental Migration | 不一次性替换整个 UI，而是逐模块替换，每步都保持可运行状态的迁移策略 |
| 离屏渲染 | Off-screen Rendering | 把 UI 绘制到一个看不见的缓冲区，再将结果合并到主渲染管线中 |
| FlutterEngine | Flutter Engine | Flutter 框架的 C++ 核心引擎，可以嵌入到任何宿主应用中 |

## 章节目录

| 章 | 标题 | 主要内容 | 节数 |
|----|------|---------|------|
| 01 | [当前 Cocos2d-x UI 分析](./01-当前Cocos2d-UI分析/) | `environ/ui/` 所有表单源码解析；Cocos Studio 资源文件（`.csb`，一种二进制 UI 布局格式）解析；UI 事件流（触摸事件→Cocos2d 分发→业务回调）全链路追踪 | 2 节 |
| 02 | [UI 抽象接口设计](./02-UI抽象接口设计/) | 定义平台无关的 UI 接口（`IDialog`/`IMenu`/`ISettingsPage`/`IFilePicker`）；事件回调接口（`IUIEventHandler`）；依赖倒置原则在 KrKr2 中的应用 | 2 节 |
| 03 | [Flutter 嵌入实战](./03-Flutter嵌入实战/) | `FlutterEngine` 集成到 CMake C++ 项目；Platform Channel 实现 UI 交互（方法调用与事件流）；FlutterEngine 生命周期管理 | 2 节 |
| 04 | [Compose 嵌入实战](./04-Compose嵌入实战/) | Kotlin/Native + cinterop 生成 C++ 绑定；Compose Desktop/Android 双端适配；C++ 与 Kotlin 互调的内存模型与线程安全 | 2 节 |
| 05 | [渲染引擎与 UI 的桥接](./05-渲染引擎与UI的桥接/) | 将 Flutter/Compose 渲染结果作为 OpenGL 纹理（外部纹理）注入 Cocos2d-x 场景；输入事件双向传递（触摸/键盘/鼠标）；生命周期同步（暂停/恢复/销毁） | 2 节 |
| 06 | [迁移策略与测试](./06-迁移策略与测试/) | 渐进式迁移方案（Phase 1 Flutter 壳 + Cocos 渲染 → Phase 2 逐步替换）；A/B 测试框架搭建；回归验证与兼容性测试 | 2 节 |

## 学习路线

```
M13 学习顺序（建议严格按章节顺序）：

01 当前Cocos2d-UI分析
    ├── 01 表单源码与资源文件解析
    └── 02 事件流追踪与设计缺陷分析
            ↓
02 UI抽象接口设计
    ├── 01 接口定义与依赖倒置
    └── 02 KrKr2抽象层实战实现
            ↓
       ┌────┴────┐
       ▼         ▼
03 Flutter嵌入  04 Compose嵌入
（可并行学习）
       └────┬────┘
            ▼
05 渲染引擎与UI的桥接
    ├── 01 纹理共享方案
    └── 02 事件与生命周期同步
            ↓
06 迁移策略与测试
    ├── 01 渐进式迁移方案
    └── 02 A/B测试与回归验证
```

## 前置检查

在开始本模块之前，请确认你已经：

- [ ] 完成 [M03-平台抽象层解析](../M03-平台抽象层解析/)，理解 KrKr2 的 PAL（平台抽象层，Platform Abstraction Layer）设计
- [ ] 完成 [M04-渲染子系统](../M04-渲染子系统/)，理解 Cocos2d-x 渲染管线与纹理系统
- [ ] 完成 [P12-现代跨平台UI](../P12-现代跨平台UI/)，了解 Flutter Engine 嵌入 API 和 Compose Multiplatform 基础
- [ ] 能够在本地成功构建 KrKr2（参见 [M01-项目导览与环境搭建](../M01-项目导览与环境搭建/)）
- [ ] 对 OpenGL 纹理和 FBO（帧缓冲对象，Frame Buffer Object）有基本了解（参见 [P04-OpenGL图形编程](../P04-OpenGL图形编程/)）

## 项目相关文件索引

| 文件/目录 | 用途 |
|-----------|------|
| `cpp/core/environ/ui/` | 现有 Cocos2d-x UI 表单实现（`.cpp/.h`） |
| `ui/cocos-studio/` | Cocos Studio UI 资源（`.csb` 二进制布局 + locale XML） |
| `cpp/core/environ/cocos2d/AppDelegate.cpp` | 应用启动与 Cocos Director 初始化 |
| `cpp/core/environ/ConfigManager/` | 配置管理器（UI 设置项读写） |
| `platforms/android/cpp/krkr2_android.cpp` | Android JNI 桥接（与 Java/Kotlin 通信的入口） |

## 完成本模块后你将能够

1. 独立分析任意 Cocos2d-x 项目的 UI 层架构，识别其设计边界
2. 设计符合开闭原则（Open-Closed Principle）的 UI 抽象接口，支持多种 UI 实现后端
3. 将 Flutter Engine 嵌入任意 C++ 宿主项目，实现双向通信
4. 使用 Kotlin/Native cinterop 从 C++ 调用 Compose UI，并处理跨语言内存管理
5. 实现 Flutter/Compose 与 OpenGL 渲染引擎之间的纹理共享，做到零拷贝合并
6. 制定并执行渐进式 UI 迁移计划，在不中断项目运行的前提下完成框架替换

---

*下一章：[01-当前Cocos2d-UI分析](./01-当前Cocos2d-UI分析/01-表单源码与资源文件解析.md)*
