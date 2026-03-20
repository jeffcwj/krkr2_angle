# M09 — 插件系统与开发

## 模块概述

KrKr2 模拟器的**插件系统**是引擎功能扩展的核心机制。在原版 KiriKiri2 引擎中，插件以 `.dll`（Windows 动态链接库）的形式在运行时动态加载，通过 `V2Link` / `V2Unlink` 接口与脚本引擎交互。而在 KrKr2 跨平台模拟器中，由于需要支持 Android、Linux、macOS 等平台，所有插件均改为**静态链接**（Static Linking，即在编译时将插件代码直接编入主程序，而非运行时加载独立文件）模式——插件代码在编译阶段就链接进主程序，运行时通过"虚拟加载"（模拟原版的 `.tpm` / `.dll` 加载过程）完成注册。

本模块将带你深入理解 KrKr2 的插件加载架构、两大绑定框架（ncbind 和 simplebinder）、现有插件的源码实现，并手把手教你从零开发一个新插件。完成本模块后，你将具备独立为 KrKr2 开发和贡献插件的能力。

### 为什么需要学习插件系统？

1. **引擎扩展的唯一入口**：KrKr2 的所有功能扩展（脚本函数扩展、图像格式支持、文件系统操作等）都通过插件机制实现
2. **理解 C++↔TJS2 桥接**：插件是 C++ 原生代码与 TJS2 脚本引擎之间的桥梁，掌握绑定框架才能让两个世界互通
3. **逆向工程的前置知识**：后续 M12（插件逆向与实现）模块需要你先理解插件注册机制，才能还原无源码插件
4. **贡献项目的必备技能**：`doc/krkr2_plugins.md` 中仍有多个无源码插件等待实现，这是参与项目贡献最直接的方式

## 学习目标

完成本模块后，你将能够：

1. **画出插件加载的完整流程图**——从 TJS2 脚本调用 `Plugins.link()` 到 C++ 类注册到全局作用域
2. **使用 ncbind 框架**注册 C++ 类、方法、属性到 TJS2 脚本引擎
3. **使用 simplebinder 框架**作为替代方案完成同样的绑定任务
4. **阅读现有插件源码**（scriptsEx、csvParser、psdfile 等）并理解其注册逻辑
5. **从零开发一个完整插件**——包括 C++ 实现、TJS2 绑定、CMake 构建配置和单元测试
6. **为项目贡献新插件**——了解插件清单、提交规范和代码审查流程

## 前置要求

- **[M01-项目导览与环境搭建](../M01-项目导览与环境搭建/)**：能构建运行项目
- **[M07-TJS2脚本引擎](../M07-TJS2脚本引擎/)**：理解 TJS2 的对象模型（`iTJSDispatch2` 接口、`tTJSVariant` 变体类型、原生类注册机制）

## 术语预览

本模块涉及大量插件系统专用术语，先有个整体印象：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 静态链接 | Static Linking | 编译时将插件代码直接合并进主程序，运行时无需加载额外文件 |
| 动态加载 | Dynamic Loading | 运行时从独立的 `.dll` / `.so` 文件中加载代码（原版 KiriKiri 的方式） |
| ncbind | Native Class Binder | KrKr2 的主要 C++↔TJS2 绑定框架，通过宏和模板元编程实现自动注册 |
| simplebinder | Simple Binder | ncbind 的替代方案，提供更简洁的流式 API（Fluent API，一种链式调用风格） |
| `V2Link` / `V2Unlink` | — | 原版 KiriKiri 插件必须导出的两个函数，分别用于插件加载和卸载 |
| `iTJSDispatch2` | — | TJS2 引擎的核心接口，所有 TJS2 对象（类、函数、属性）都通过此接口操作 |
| `tTJSVariant` | — | TJS2 的"万能变量"类型，可存放整数、浮点数、字符串、对象等任意值 |
| `ncbAutoRegister` | — | ncbind 的自动注册器，利用 C++ 静态变量初始化实现编译期注册 |
| `NCB_REGISTER_CLASS` | — | ncbind 宏，用于将一个 C++ 类注册为新的 TJS2 全局类 |
| `NCB_ATTACH_CLASS` | — | ncbind 宏，用于给已有的 TJS2 类（如 `Scripts`）附加新方法/属性 |
| `ncbInstanceAdaptor` | — | ncbind 的实例适配器（Adaptor），将 C++ 对象包装为 TJS2 原生实例 |
| 三阶段注册 | Three-phase Registration | ncbind 的注册分为 PreRegist → ClassRegist → PostRegist 三个阶段依次执行 |
| 模块名映射 | Module Name Mapping | KrKr2 将 `.tpm` 文件名转换为 `.dll` 名称，用于查找对应的静态注册表 |

## 章节目录

| 章 | 标题 | 主要内容 |
|----|------|---------|
| 01 | [插件系统架构](./01-插件系统架构/) | 静态链接模型与原版对比、`TVPLoadPlugin` 加载流程、`ncbAutoRegister` 三阶段注册机制 |
| 02 | [ncbind 绑定框架](./02-ncbind绑定框架/) | `ncbTypeConvertor` 类型转换系统、`NCB_REGISTER_CLASS` / `NCB_ATTACH_CLASS` 宏体系、属性/方法/回调绑定 |
| 03 | [simplebinder 详解](./03-simplebinder详解/) | `BindUtil` 流式 API、与 ncbind 的功能对比、`ClassStore` / `InstanceWrapper` 高级用法 |
| 04 | [现有插件源码导读](./04-现有插件源码导读/) | scriptsEx（脚本扩展）与 csvParser（CSV 解析器）、psdfile（PSD 文件解析）与 psbfile（PSB/E-mote 文件解析）、layerExDraw（扩展绘图）与 fstat（文件状态查询） |
| 05 | [实战：从零开发插件](./05-实战-从零开发插件/) | 需求分析与接口设计、C++ 实现与 TJS2 绑定（同时演示 ncbind 和 simplebinder 两种方式）、CMake 注册与 Catch2 单元测试 |
| 06 | [插件清单与状态](./06-插件清单与状态/) | 全部 54 个插件的实现状态总览、12 个无源码插件的逆向分析思路、为项目贡献新插件的完整流程 |

## 预计学习时间

约 **10-14 小时**（含动手实践和练习题）

## 学习路线建议

```
01-插件系统架构（理解"插件怎么被加载的"）
        ↓
02-ncbind 绑定框架（掌握主要绑定工具）
        ↓
03-simplebinder 详解（掌握替代方案，扩展视野）
        ↓
04-现有插件源码导读（看真实代码巩固理解）
        ↓
05-实战：从零开发插件（动手写一个完整插件）
        ↓
06-插件清单与状态（了解项目全貌，准备贡献）
```

建议按顺序学习。01-03 章是理论基础，04 章是源码阅读实践，05 章是动手实战，06 章是项目贡献指南。如果你已经熟悉 C++ 模板元编程，可以加速阅读 02-03 章；如果你的目标是尽快贡献代码，可以在完成 01-02 章后直接跳到 05 章实战。

## 相关源码目录

| 目录 / 文件 | 说明 |
|-------------|------|
| `cpp/core/plugin/` | 插件框架核心代码（ncbind、加载器、注册器） |
| `cpp/plugins/` | 所有插件的实现代码（静态库） |
| `cpp/plugins/simplebinder/` | simplebinder 替代绑定框架 |
| `cpp/plugins/CMakeLists.txt` | 插件构建配置（注册新插件的入口） |
| `doc/krkr2_plugins.md` | 完整插件清单（54 个插件的来源和状态） |
