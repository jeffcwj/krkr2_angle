# M02 - 项目构建系统深度解析

> **前置模块：** [P01-现代CMake与构建工具链](../P01-现代CMake与构建工具链/README.md)、[P02-vcpkg包管理](../P02-vcpkg包管理/README.md)
> **目标：** 逐行解读 KrKr2 项目的完整 CMake 构建体系，理解每一行构建脚本的设计意图

## 模块概述

本模块是 **M 系列（项目模块解析）** 的第二个模块，聚焦于 KrKr2 项目的构建系统。在 P01 和 P02 中，你已经学习了 CMake 和 vcpkg 的通用知识；本模块将这些知识应用到 KrKr2 的实际构建脚本上，逐行分析每个构建文件的设计决策。

读完本模块后，你将能够：

- 完整理解根 `CMakeLists.txt` 的每一行代码及其作用
- 掌握 INTERFACE 库 `krkr2core` 的设计模式和依赖传播机制
- 理解四平台（Windows/Linux/macOS/Android）条件编译的实现方式
- 掌握 Android Gradle 与 CMake 的协作机制
- 能够修改和扩展项目的自定义 CMake 模块

## 章节目录

| 序号 | 标题 | 小节 | 核心内容 |
|------|------|------|----------|
| 01 | [根 CMakeLists.txt 深度解读](./01-根CMakeLists深度解读/) | 01-CMake版本与全局设置 / 02-vcpkg集成与平台入口 / 03-子目录链接与资源拷贝 | 逐段分析 143 行根构建文件：选项定义、平台检测、目标注册、链接依赖、资源拷贝 |
| 02 | [INTERFACE 库与模块链接](./02-INTERFACE库与模块链接/) | 01-INTERFACE库本质与krkr2core / 02-krkr2plugin与依赖传播 / 03-动手实践与总结 | `krkr2core` 的 INTERFACE 传播机制、9 个子模块的链接关系、插件 STATIC 库设计 |
| 03 | [平台条件编译实现](./03-平台条件编译实现/) | 01-CMakePresets与平台Preset / 02-条件编译与vcpkg平台条件 / 03-源代码中的条件编译与实践 | 自定义平台变量、CMakePresets.json 预设体系、平台源文件选择策略 |
| 04 | [Android Gradle 与 CMake 协作](./04-Android-Gradle与CMake协作/) | 01-双层架构与Gradle配置 / 02-JNI桥接层与常见错误 / 03-动手实践与总结 | Gradle 项目结构、`build.gradle` CMake 配置、NDK ABI 过滤、JNI 加载 |
| 05 | [自定义 CMake 模块](./05-自定义CMake模块/) | 01-CMake辅助模块与封装方式 / 02-CocosBuildHelpers详解 / 03-vcpkg-android与sync-folder / 04-对照源码与总结 | `CocosBuildHelpers.cmake` 资源管理、`vcpkg_android.cmake` 双 toolchain 叠加 |

## 涉及的源码文件

本模块分析的核心文件：

```
krkr2/
├── CMakeLists.txt                    # 根构建文件（143 行）— 第 01 章
├── CMakePresets.json                  # 多平台预设（156 行）— 第 03 章
├── vcpkg.json                         # 依赖清单（94 行）— 第 02、05 章
├── cpp/
│   ├── core/CMakeLists.txt           # 核心 INTERFACE 库（67 行）— 第 02 章
│   └── plugins/CMakeLists.txt        # 插件 STATIC 库（74 行）— 第 02 章
├── cmake/
│   ├── CocosBuildHelpers.cmake       # Cocos2d-x 构建辅助（381 行）— 第 05 章
│   └── vcpkg_android.cmake           # Android vcpkg 集成（99 行）— 第 05 章
└── platforms/android/
    ├── build.gradle                   # 根 Gradle 配置 — 第 04 章
    ├── app/build.gradle              # 应用模块 Gradle 配置 — 第 04 章
    ├── settings.gradle               # 多模块项目设置 — 第 04 章
    └── gradle.properties             # 全局属性定义 — 第 04 章
```

## 学习建议

1. **边读边对照源码**：建议用编辑器打开对应的源文件，边读教程边看实际代码
2. **按顺序阅读**：章节之间有递进关系，第 01 章是后续章节的基础
3. **动手修改**：每章的"动手实践"环节建议实际操作，加深理解
4. **跨模块参考**：本模块会频繁引用 P01（CMake 基础）和 P02（vcpkg 基础）的概念，如遇不理解的术语请回查

## 编写状态

| 章 | 小节文件 | 状态 |
|----|----------|------|
| 01 | 01-CMake版本与全局设置.md / 02-vcpkg集成与平台入口.md / 03-子目录链接与资源拷贝.md | ✅ 已完成 |
| 02 | 01-INTERFACE库本质与krkr2core.md / 02-krkr2plugin与依赖传播.md / 03-动手实践与总结.md | ✅ 已完成 |
| 03 | 01-CMakePresets与平台Preset.md / 02-条件编译与vcpkg平台条件.md / 03-源代码中的条件编译与实践.md | ✅ 已完成 |
| 04 | 01-双层架构与Gradle配置.md / 02-JNI桥接层与常见错误.md / 03-动手实践与总结.md | ✅ 已完成 |
| 05 | 01-CMake辅助模块与封装方式.md / 02-CocosBuildHelpers详解.md / 03-vcpkg-android与sync-folder.md / 04-对照源码与总结.md | ✅ 已完成 |
