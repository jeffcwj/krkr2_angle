# M03 - 平台抽象层解析

> **类型**：M 系列（源码解析模块）  
> **前置模块**：M01（项目导览）、P03（跨平台 C++ 开发）  
> **对应源码**：`krkr2/cpp/core/environ/`

---

## 模块概述

本模块深入解析 KrKr2 项目的**平台抽象层（PAL）**——`environ` 模块。这是整个模拟器的"操作系统适配层"，负责将 Windows/Linux/macOS/Android 四大平台的差异封装为统一的 C++ 接口。

通过本模块的学习，你将理解：
- 如何用 C 风格函数 + 条件编译实现零开销的平台抽象
- `tTVPApplication` 如何管理应用生命周期和消息循环
- Cocos2d-x 框架如何作为桥接层驱动引擎核心
- 配置管理系统的多层架构（Global / Individual / Locale）

---

## 章节目录

| 章节 | 文件 | 内容概要 |
|------|------|----------|
| 第 1 章 | [01-environ模块总览与架构.md](./01-environ模块总览与架构.md) | 模块职责、目录结构、CMake 构建配置、架构全景图 |
| 第 2 章 | [02-Platform接口与平台实现对比.md](./02-Platform接口与平台实现对比.md) | Platform.h 接口设计、Win32/Linux/Android 三平台实现逐行对比 |
| 第 3 章 | [03-Application生命周期与消息系统.md](./03-Application生命周期与消息系统.md) | tTVPApplication 类、启动流程、消息循环、内存管理、事件分发 |
| 第 4 章 | [04-Cocos2d-x桥接层.md](./04-Cocos2d-x桥接层.md) | AppDelegate、MainScene、CustomFileUtils、Director 集成 |
| 第 5 章 | [05-配置管理系统.md](./05-配置管理系统.md) | iSysConfigManager 基类、Global/Individual/Locale 三级配置、模板特化 |

---

## 对应源码文件

```
krkr2/cpp/core/environ/
├── Application.h / .cpp       ← 应用生命周期（第 3 章）
├── Platform.h                 ← 平台抽象接口（第 2 章）
├── CMakeLists.txt             ← 构建配置（第 1 章）
├── DetectCPU.cpp              ← CPU 特性检测（第 1 章）
├── win32/Platform.cpp         ← Windows 实现（第 2 章）
├── linux/Platform.cpp         ← Linux 实现（第 2 章）
├── android/AndroidUtils.cpp   ← Android 实现（第 2 章）
├── cocos2d/                   ← Cocos2d-x 桥接（第 4 章）
│   ├── AppDelegate.cpp
│   ├── MainScene.h / .cpp
│   └── CustomFileUtils.h / .cpp
├── ConfigManager/             ← 配置管理（第 5 章）
│   ├── GlobalConfigManager.h / .cpp
│   ├── IndividualConfigManager.h / .cpp
│   └── LocaleConfigManager.h / .cpp
├── sdl/tvpsdl.cpp             ← SDL 集成桩（第 1 章）
└── ui/                        ← UI 组件（本模块简要提及）
```

---

## 学习路线建议

1. 先通读第 1 章了解整体架构
2. 第 2 章和第 3 章是核心，建议边读源码边看教程
3. 第 4 章需要 Cocos2d-x 基础知识（P10 模块，但本章会提供足够的上下文）
4. 第 5 章相对独立，可根据需要选读
