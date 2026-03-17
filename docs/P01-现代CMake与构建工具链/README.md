# P01 — 现代 CMake 与构建工具链

## 模块概述

CMake 是当前 C++ 生态中最主流的跨平台构建系统生成器。KrKr2 项目使用 CMake 3.28+ 配合 Ninja 构建后端，通过 `CMakePresets.json` 统一管理 Windows、Linux、macOS、Android 四个平台的构建配置。

本模块从零开始讲解 CMake 的核心概念、语法命令、多目标组织方式、Presets 机制和 Ninja 构建后端，最终带你深入解读 KrKr2 项目真实的 CMake 构建脚本。完成本模块后，你不仅能看懂任何 CMake 项目，还能独立为项目添加新模块、新平台支持。

## 学习目标

完成本模块后，你将能够：
1. 理解构建系统的演进历史和 CMake 的设计理念
2. 熟练使用 CMake 的变量、缓存、目标、属性、生成器表达式等核心概念
3. 使用 `add_subdirectory()` 组织多目标、多模块项目
4. 理解 `INTERFACE`、`PUBLIC`、`PRIVATE` 三种链接可见性的区别与使用场景
5. 使用 `CMakePresets.json` 统一管理多平台构建配置
6. 使用 Ninja 作为 CMake 的构建后端，提升编译速度
7. 独立阅读和分析 KrKr2 项目的 CMake 构建脚本（根目录、core 模块、平台条件编译）

## 前置要求

- C++ 基础语法（能编写并编译简单的 C++ 程序）
- 基本的命令行操作能力（Windows CMD/PowerShell 或 Linux/macOS Terminal）
- 无需任何 CMake 经验

## 章节目录

| 章 | 标题 | 内容概要 |
|----|------|---------|
| 01 | [CMake 是什么](./01-CMake是什么/) | 构建系统的演进历史、CMake 核心概念（生成器/目标/属性）、手把手创建第一个 CMake 项目 |
| 02 | [CMake 语法与命令](./02-CMake语法与命令/) | 变量与缓存、目标与属性、生成器表达式 |
| 03 | [多目标与子目录](./03-多目标与子目录/) | `add_subdirectory` 机制、INTERFACE/PUBLIC/PRIVATE 可见性、实战构建多模块库 |
| 04 | [CMakePresets](./04-CMakePresets/) | Presets 配置详解（继承/条件/变量）、多平台预设管理实战 |
| 05 | [Ninja 构建后端](./05-Ninja构建后端/) | Ninja 简介与安装（四平台）、CMake 搭配 Ninja 的最佳实践 |
| 06 | [实战：分析 KrKr2 的 CMake](./06-实战-分析KrKr2的CMake/) | 根 CMakeLists.txt 解读、core 模块 CMake 解读、平台条件编译 |

## 文件清单

```
P01-现代CMake与构建工具链/
├── README.md                                    ← 本文件
├── 01-CMake是什么/
│   ├── 01-构建系统的演进.md
│   ├── 02-CMake核心概念.md
│   └── 03-第一个CMake项目.md
├── 02-CMake语法与命令/
│   ├── 01-变量与缓存.md
│   ├── 02-目标与属性.md
│   └── 03-生成器表达式.md
├── 03-多目标与子目录/
│   ├── 01-add_subdirectory.md
│   ├── 02-INTERFACE与PUBLIC与PRIVATE.md
│   └── 03-实战-构建多模块库.md
├── 04-CMakePresets/
│   ├── 01-Presets配置详解.md
│   └── 02-多平台预设管理.md
├── 05-Ninja构建后端/
│   ├── 01-Ninja简介与安装.md
│   └── 02-CMake搭配Ninja.md
└── 06-实战-分析KrKr2的CMake/
    ├── 01-根CMakeLists解读.md
    ├── 02-core模块CMake解读.md
    └── 03-平台条件编译.md
```

## 预计学习时间

6-8 小时（如完全没有 CMake 经验），3-4 小时（如有基本了解）。

## 下一步

完成本模块后，建议继续学习 **[P02 — vcpkg 包管理](../P02-vcpkg包管理/)**，了解 KrKr2 如何管理第三方依赖库。
