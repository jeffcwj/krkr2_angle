# P02 — vcpkg 包管理

## 模块概述

vcpkg 是微软开源的 C/C++ 跨平台包管理器，能自动下载、编译和集成数千个第三方库。KrKr2 项目使用 vcpkg 的 manifest 模式管理超过 30 个第三方依赖库，并通过自定义 overlay ports 和 triplet 文件适配四个目标平台。

本模块从 vcpkg 的安装与基本使用讲起，深入到 overlay ports、自定义 triplet、二进制缓存等高级特性，再讲解 vcpkg 与 CMake 的集成机制（toolchain 文件、find_package），最后带你逐行解读 KrKr2 项目真实的 vcpkg 配置。完成本模块后，你将完全理解项目的依赖管理体系，并能独立添加、修改、调试第三方库依赖。

## 学习目标

完成本模块后，你将能够：
1. 安装和配置 vcpkg（Windows/Linux/macOS），理解 classic 模式与 manifest 模式的区别
2. 编写 `vcpkg.json` 和 `vcpkg-configuration.json` 管理项目依赖
3. 使用平台过滤器、features 控制、version overrides 等高级依赖声明语法
4. 创建 overlay ports 自定义或修补第三方库
5. 编写自定义 triplet 文件适配 Android 等特殊平台
6. 配置二进制缓存加速 CI/CD 构建
7. 理解 vcpkg toolchain 文件与 CMake 的集成原理（包括 Android 双 toolchain 叠加）
8. 熟练使用 `find_package` 的 Module 模式和 Config 模式
9. 独立阅读和修改 KrKr2 项目的全部 vcpkg 配置

## 前置要求

- 完成 **[P01 — 现代 CMake 与构建工具链](../P01-现代CMake与构建工具链/)**（熟悉 CMake 基础语法、目标、属性、CMakePresets）
- 基本的命令行操作能力
- 无需任何 vcpkg 经验

## 章节目录

| 章 | 标题 | 内容概要 |
|----|------|---------|
| 01 | [vcpkg 基础](./01-vcpkg基础/) | vcpkg 安装与配置（四平台）、manifest 模式详解、搜索与安装包 |
| 02 | [高级用法](./02-高级用法/) | overlay ports 自定义端口、自定义 triplet 文件、二进制缓存 |
| 03 | [vcpkg 与 CMake 集成](./03-vcpkg与CMake集成/) | toolchain 文件机制、Android 双 toolchain 叠加、find_package 用法 |
| 04 | [实战：分析 KrKr2 的 vcpkg 配置](./04-实战-分析KrKr2的vcpkg配置/) | vcpkg.json 逐项解读（30+ 依赖分类）、overlay ports 与自定义 triplet 实战 |

## 文件清单

```
P02-vcpkg包管理/
├── README.md                                        ← 本文件
├── 01-vcpkg基础/
│   ├── 01-安装与配置.md
│   ├── 02-manifest模式.md
│   └── 03-搜索与安装包.md
├── 02-高级用法/
│   ├── 01-overlay-ports.md
│   ├── 02-自定义triplet.md
│   └── 03-二进制缓存.md
├── 03-vcpkg与CMake集成/
│   ├── 01-toolchain文件.md
│   └── 02-find_package用法.md
└── 04-实战-分析KrKr2的vcpkg配置/
    ├── 01-vcpkg.json解读.md
    └── 02-overlay-ports与自定义triplet.md
```

## 预计学习时间

5-7 小时（如完全没有包管理器经验），3-4 小时（如熟悉 npm/pip 等其他语言的包管理器）。

## 下一步

完成本模块后，建议继续学习 **[M01 — 项目导览与环境搭建](../M01-项目导览与环境搭建/)**，实际搭建开发环境并构建运行 KrKr2 项目。
