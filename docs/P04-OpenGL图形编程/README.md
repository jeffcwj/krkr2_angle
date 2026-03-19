# P04 — OpenGL 图形编程

> **前置知识：** [P01-现代 CMake 与构建工具链](../P01-现代CMake与构建工具链/)、[P03-跨平台 C++ 开发](../P03-跨平台C++开发/)
> **难度：** ★★★★
> **后续模块：** [P05-软件渲染原理](../P05-软件渲染原理/)、[P10-Cocos2d-x 框架](../P10-Cocos2d-x框架/)、[M04-渲染子系统](../M04-渲染子系统/)

## 模块简介

KrKr2 模拟器的核心渲染引擎基于 Cocos2d-x，底层使用 OpenGL（桌面）和 OpenGL ES（移动端）进行 GPU 加速渲染。理解 OpenGL 渲染管线是阅读和修改 KrKr2 渲染代码（`cpp/core/visual/`）的必备前置技能。

本模块从 GPU 渲染管线的基本原理开始，逐步深入到着色器编程、纹理处理、帧缓冲操作，并重点讲解 OpenGL 与 OpenGL ES 的差异（移动端适配要点）和纹理压缩格式（PVRTC/ETC/ASTC）。最终通过一个完整的 2D 精灵渲染器实战项目，将所有知识串联起来。

## 章节目录

| 章 | 标题 | 主要内容 |
|----|------|----------|
| 01 | [GPU 渲染管线概述](./01-GPU渲染管线概述/) | GPU 架构；渲染管线各阶段（顶点→光栅化→片段）；OpenGL 状态机；GL 上下文与线程 |
| 02 | [OpenGL 基础](./02-OpenGL基础/) | VAO/VBO/EBO 概念与使用；着色器编译链接；uniform/attribute；基础绘制命令 |
| 03 | [着色器编程](./03-着色器编程/) | GLSL 语法；顶点着色器/片段着色器；varying/in/out；内置变量；调试技巧 |
| 04 | [纹理与采样](./04-纹理与采样/) | 纹理单元；采样器参数；mipmap 生成；纹理格式（RGBA8/RGB565/RGBA4444 等） |
| 05 | [帧缓冲与后处理](./05-帧缓冲与后处理/) | FBO/RBO 概念；离屏渲染；多 pass 处理；后处理效果（模糊/颜色校正） |
| 06 | [OpenGL ES 差异](./06-OpenGL-ES差异/) | ES 2.0/3.0 与桌面 GL 的区别；精度限定符；兼容写法；EGL 上下文管理 |
| 07 | [纹理压缩格式](./07-纹理压缩格式/) | PVRTC/ETC1/ETC2/ASTC 原理；GPU 原生解码；运行时格式选择；KrKr2 的纹理策略 |
| 08 | [实战：2D 精灵渲染器](./08-实战-实现一个2D渲染器/) | 从零实现纹理加载→精灵绘制→Alpha 混合→批量渲染→跨平台运行 |

## 术语预览

上方章节目录中出现了大量 OpenGL 专业术语，先在这里建立印象，各章正文会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| VAO | Vertex Array Object | 记录"顶点数据怎么读"的配置对象，绑定后 GPU 就知道如何解析你的顶点缓冲 |
| VBO | Vertex Buffer Object | 存放顶点坐标、颜色、纹理坐标等数据的 GPU 端缓冲区 |
| EBO | Element Buffer Object | 存放顶点索引的缓冲区，允许多个三角形共享顶点以节省显存 |
| GLSL | OpenGL Shading Language | OpenGL 的着色器编程语言，语法类似 C，在 GPU 上运行 |
| uniform / attribute | — | 着色器输入变量的两种类型：uniform 是全局常量（如投影矩阵），attribute 是逐顶点数据（如位置） |
| varying / in / out | — | 着色器之间传递数据的关键字，顶点着色器的 out 会插值后传给片段着色器的 in |
| Mipmap | — | 预生成的多级缩小版纹理（1/2、1/4、1/8...），让远处物体纹理不闪烁，GPU 自动选择合适的级别 |
| FBO | Framebuffer Object | GPU 上的一块"画布"，可以把渲染结果画到这里而不是直接显示到屏幕（离屏渲染的基础） |
| RBO | Renderbuffer Object | 类似纹理但不能被采样的存储，专用于深度测试/模板测试的高效缓冲 |
| 离屏渲染 | Off-screen Rendering | 先把画面渲染到一个不可见的 FBO 上，做完后处理再搬到屏幕——实现模糊、泛光等效果的基础 |
| 多 Pass | Multi-pass Rendering | 把渲染拆成多个阶段，每个阶段（Pass）做一件事，比如第一次画场景、第二次加模糊 |
| 精度限定符 | Precision Qualifier | OpenGL ES 中用 `lowp`/`mediump`/`highp` 指定变量精度，影响移动端性能和精度 |
| EGL | — | 连接 OpenGL ES 与平台窗口系统的标准接口（Android/嵌入式设备上用它创建 GL 上下文） |
| PVRTC | PowerVR Texture Compression | PowerVR GPU（iOS/部分 Android）的专用纹理压缩格式，大幅减少显存占用 |
| ETC1 / ETC2 | Ericsson Texture Compression | Android 平台的标准纹理压缩格式，ETC1 不支持透明通道，ETC2 支持 |
| ASTC | Adaptive Scalable Texture Compression | 新一代通用纹理压缩格式，支持多种压缩率，OpenGL ES 3.2+ 标配 |
| Alpha 混合 | Alpha Blending | 用像素的透明度值（Alpha）将前景和背景颜色按比例混合，实现半透明效果 |
| draw call | — | CPU 向 GPU 发出的一次"画这些东西"的指令，数量越多 CPU 开销越大 |
| 批量渲染 | Batching | 把多个使用相同纹理/着色器的绘制请求合并为一次 draw call，减少 CPU→GPU 通信开销 |

## 学习路线

```
01-GPU渲染管线概述
    ↓ 理解 GPU 如何把顶点变成像素
02-OpenGL基础
    ↓ 掌握 OpenGL 核心 API 和对象模型
03-着色器编程
    ↓ 学会用 GLSL 编写 GPU 程序
04-纹理与采样
    ↓ 掌握纹理加载、采样和格式
05-帧缓冲与后处理
    ↓ 理解离屏渲染和多 pass 技术
06-OpenGL ES差异
    ↓ 掌握移动端 GL 的限制和适配
07-纹理压缩格式
    ↓ 了解移动端 GPU 纹理优化
08-实战：2D精灵渲染器
    → 综合运用，构建一个完整的跨平台 2D 渲染器
```

## 与 KrKr2 项目的关联

- **`cpp/core/visual/`** — 渲染子系统核心，包含图层管理、图像加载器、过渡效果、OGL 渲染后端
- **`cpp/core/visual/ogl/`** — OpenGL 渲染管理器（RenderManager_ogl），纹理上传、批量渲染
- **`cpp/core/visual/Load*.cpp`** — 各种图像格式加载器（PNG/JPEG/TLG/WEBP/BPG/JXR/PVR）
- **`cpp/core/environ/cocos2d/`** — Cocos2d-x 集成层，负责 OpenGL 上下文创建和场景管理
- **`ui/cocos-studio/`** — Cocos Studio UI 资产，使用 OpenGL 渲染的 UI 组件

学完本模块后，你将具备阅读 KrKr2 渲染代码的全部 OpenGL 知识基础，为后续的 M04（渲染子系统深度解析）打下坚实基础。

## 编写状态

| 章 | 小节文件 | 状态 |
|----|----------|------|
| 01 | 01-GPU架构与OpenGL简介.md / 02-OpenGL渲染管线详解.md / 03-上下文版本与实践.md | ✅ 已完成 |
| 02 | 01-缓冲对象与VBO.md / 02-VAO与EBO.md / 03-着色器程序与实践.md | ✅ 已完成 |
| 03 | 01-GLSL语言与着色器阶段.md / 02-Uniform与平台差异.md | ✅ 已完成 |
| 04 | 01-纹理基础与参数配置.md / 02-多纹理与像素传输.md | ✅ 已完成 |
| 05 | 01-帧缓冲基础与创建.md / 02-后处理与多Pass管线.md / 03-实践与常见错误.md | ✅ 已完成 |
| 06 | 01-ES版本演进与核心差异.md / 02-精度限定符与兼容编码.md | ✅ 已完成 |
| 07 | 01-GPU纹理压缩全景.md / 02-实践与总结.md | ✅ 已完成 |
| 08 | 01-架构设计与着色器.md / 02-顶点批处理与纹理.md / 03-实践与常见错误.md | ✅ 已完成 |
