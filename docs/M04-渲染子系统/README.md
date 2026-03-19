# M04 — 渲染子系统

> **模块对应源码：** `cpp/core/visual/`（CMake 目标：`core_visual_module`）
> **前置知识：** [P04-OpenGL图形编程](../P04-OpenGL图形编程/README.md)、[P05-软件渲染原理](../P05-软件渲染原理/README.md)、[P10-Cocos2d-x框架](../P10-Cocos2d-x框架/README.md)
> **学习时长：** 约 20-25 小时

## 模块简介

渲染子系统（`core_visual_module`）是 KrKr2 引擎中规模最大、功能最复杂的核心模块。它负责：

- **图层管理** — 树形图层结构、图层属性（透明度、混合模式、可见性）、绘制顺序
- **图像加载** — 支持 PNG、JPEG、TLG（KiriKiri 引擎原生的图像压缩格式）、WebP、BPG、JPEG XR、PVR 七种格式
- **像素操作** — tvpgl 库（KiriKiri 引擎自带的像素处理 C 函数库）提供 200+ 像素级操作函数（混合、合成、滤镜、变换）
- **OpenGL 渲染** — GPU 加速渲染管线、纹理压缩（PVRTC/ETC/ASTC，见下方术语预览）、纹理图集（多张小图拼合为一张大纹理以减少 draw call）
- **软件渲染** — 纯 CPU 渲染路径、ARGB 颜色空间操作（每个像素 4 字节：透明度+三原色）、SIMD 优化框架（一条指令同时处理多个像素）
- **过渡效果** — 场景切换动画（淡入淡出、滑动、百叶窗、自定义规则图等）
- **字体渲染** — FreeType（开源字体光栅化引擎）集成、预渲染字体、字符数据管理
- **视频叠加** — 视频播放覆盖层接口

## 术语预览

本模块涉及大量渲染领域专业术语，先在这里建立印象，各章正文会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| tvpgl | TVP Graphics Library | KiriKiri 引擎自带的像素操作 C 函数库（14000+ 行），提供 200 多个混合、合成、滤镜函数 |
| ARGB | Alpha-Red-Green-Blue | KrKr2 内部使用的像素格式，4 个字节分别存储透明度和三原色（`tjs_uint32` 类型） |
| 预乘 Alpha | Premultiplied Alpha | 将 RGB 预先乘以 Alpha 值的存储方式，能简化混合公式并避免半透明边缘出现白边 |
| SIMD | Single Instruction, Multiple Data | 一条 CPU 指令同时处理多个数据的并行技术（如 SSE2 一次处理 4 个像素的混合运算） |
| TLG | TLG5/TLG6 Image Format | KiriKiri 引擎原生的图像压缩格式，针对视觉小说的立绘和背景优化，支持无损和有损压缩 |
| PVRTC | PowerVR Texture Compression | PowerVR GPU（iOS/部分 Android）专用的纹理压缩格式，显存占用仅为 RGBA 的 1/8 |
| ETC | Ericsson Texture Compression | Android 平台标准纹理压缩格式，ETC1 不支持透明通道，ETC2 支持 |
| ASTC | Adaptive Scalable Texture Compression | 新一代通用纹理压缩格式，支持多种压缩率和通道配置 |
| FreeType | — | 开源字体光栅化引擎，将 TTF/OTF 字体的矢量轮廓转换为像素位图以供显示 |
| 纹理图集 | Texture Atlas | 把多张小图拼合到一张大纹理上，减少纹理切换次数从而降低 draw call 开销 |
| 过渡效果 | Transition Effect | 场景切换时的动画效果（淡入淡出、滑动、百叶窗等），由 TransIntf 接口统一管理 |
| 规则图 | Rule Image | 一张灰度图定义过渡动画的扩散模式——像素越黑越早切换，实现自定义过渡形状 |
| RenderManager | — | 渲染管理器抽象基类，KrKr2 同时提供 OGL（GPU 加速）和 software（纯 CPU）两个实现 |
| draw call | — | CPU 向 GPU 发出的一次"画这些东西"的指令，频繁发出会成为性能瓶颈 |

## 源码结构

```
cpp/core/visual/
├── ogl/                           # OpenGL 渲染后端
│   ├── RenderManager_ogl.cpp      #   OGL 渲染管理器实现
│   ├── pvrtc.*, pvr.h             #   PVRTC 纹理压缩
│   ├── etcpak.*                   #   ETC 纹理打包
│   ├── astcrt.*                   #   ASTC 实时纹理压缩
│   └── imagepacker.*              #   纹理图集打包算法
├── gl/                            # GL 抽象层（着色器、缓冲区）
├── impl/                          # 平台相关实现
│
│── LayerIntf.{h,cpp}              # 图层接口（核心抽象）
│── LayerBitmapIntf.{h,cpp}        # 位图图层接口
│── LayerManager.{h,cpp}           # 图层管理器（树操作、绘制排序）
│── LayerTreeOwner.h               # 图层树所有者接口
│── LayerTreeOwnerImpl.{h,cpp}     # 图层树所有者实现
│── BitmapLayerTreeOwner.{h,cpp}   # 位图图层树所有者
│
│── BitmapIntf.{h,cpp}             # 位图接口（像素缓冲区管理）
│── RenderManager.{h,cpp}          # 渲染管理器抽象基类
│── RenderManager_software.h       # 软件渲染管理器
│── WindowIntf.{h,cpp}             # 窗口接口
│
│── Load{PNG,JPEG,TLG,WEBP,...}.cpp # 图像格式加载器（每种格式一个文件）
│── SaveTLG{5,6}.cpp               # TLG 格式保存器
│── GraphicsLoaderIntf.{h,cpp}     # 图像加载调度框架
│── GraphicsLoadThread.{h,cpp}     # 异步图像加载线程
│── ImageFunction.{h,cpp}          # 图像处理函数集
│
│── tvpgl.{h,cpp}                  # 像素操作库（200+ 函数）
│── tvpgl_asm_init.h               # SIMD 优化初始化
│── argb.{h,cpp}                   # ARGB 颜色空间操作
│── tvpps.inc                      # Photoshop 混合模式宏
│
│── TransIntf.{h,cpp}              # 过渡效果接口
│── transhandler.h                 # 过渡处理器
│
│── CharacterData.{h,cpp}          # 字符渲染数据
│── FontImpl.{h,cpp}               # 字体实现
│── FontSystem.{h,cpp}             # 字体系统管理
│── FreeType.{h,cpp}               # FreeType 库封装
│── FreeTypeFontRasterizer.{h,cpp} # FreeType 光栅化器
│── PrerenderedFont.{h,cpp}        # 预渲染位图字体
│
│── VideoOvlIntf.{h,cpp}           # 视频叠加接口
│── ComplexRect.{h,cpp}            # 复杂矩形运算
│── RectItf.{h,cpp}                # 矩形接口
│── MenuItemIntf.{h,cpp}           # 菜单项接口
└── drawable.h                     # 可绘制对象接口
```

## 章节目录

本模块分为 7 章，建议按顺序学习：

### 第 1 章：visual 模块总览

从宏观角度了解整个渲染子系统的架构设计。

| 节 | 文件 | 内容 |
|----|------|------|
| 01 | [模块架构与文件组织](01-visual模块总览/01-模块架构与文件组织.md) | 文件组织、CMake 构建、模块依赖关系 |
| 02 | [渲染管线概览](01-visual模块总览/02-渲染管线概览.md) | RenderManager 抽象、OGL/软件双后端、帧循环 |
| 03 | [窗口与显示管理](01-visual模块总览/03-窗口与显示管理.md) | WindowIntf、显示管理、四平台窗口差异 |

### 第 2 章：图层系统

KiriKiri 的核心渲染概念 — 一切可见元素都是图层。

| 节 | 文件 | 内容 |
|----|------|------|
| 01 | [图层接口与继承体系](02-图层系统/01-图层接口与继承体系.md) | LayerIntf、LayerBitmapIntf、接口设计模式 |
| 02 | [图层树与管理器](02-图层系统/02-图层树与管理器.md) | LayerManager、树操作、绘制顺序、裁剪 |
| 03 | [位图操作与绘制](02-图层系统/03-位图操作与绘制.md) | BitmapIntf、像素缓冲区、绘制 API |

### 第 3 章：图像加载器

支持 7 种图像格式的可扩展加载框架。

| 节 | 文件 | 内容 |
|----|------|------|
| 01 | [图像加载框架](03-图像加载器/01-图像加载框架.md) | GraphicsLoaderIntf、加载器注册、异步加载 |
| 02 | [内置格式解析](03-图像加载器/02-内置格式解析.md) | PNG/JPEG/WebP/BPG/JXR/PVR 各格式实现 |
| 03 | [TLG 格式深度解析](03-图像加载器/03-TLG格式深度解析.md) | KiriKiri 原生 TLG5/TLG6 格式编解码 |

### 第 4 章：OGL 后端

GPU 加速渲染管线 — 移动端和桌面端的 OpenGL 实现。

| 节 | 文件 | 内容 |
|----|------|------|
| 01 | [RenderManager 架构](04-OGL后端/01-RenderManager架构.md) | 抽象基类、OGL 实现、初始化流程 |
| 02 | [纹理管理与压缩](04-OGL后端/02-纹理管理与压缩.md) | PVRTC/ETC/ASTC 格式、纹理图集、GPU 内存 |
| 03 | [渲染流程与优化](04-OGL后端/03-渲染流程与优化.md) | Draw call、批处理、GL 状态管理、移动优化 |

### 第 5 章：软件渲染

纯 CPU 像素操作 — tvpgl 库的 200+ 函数与 SIMD 优化框架。

| 节 | 文件 | 内容 |
|----|------|------|
| 01 | [tvpgl 像素操作库](05-软件渲染/01-tvpgl像素操作库.md) | 函数分类、宏系统、函数指针分发 |
| 02 | [ARGB 混合与合成](05-软件渲染/02-ARGB混合与合成.md) | ARGB 颜色空间、混合公式、PS 混合模式 |
| 03 | [SSE/ASM 优化路径](05-软件渲染/03-SSE-ASM优化路径.md) | CPU 检测、SIMD 分发架构、优化实战 |

### 第 6 章：过渡效果

场景切换的动画系统 — 从内置效果到自定义规则图。

| 节 | 文件 | 内容 |
|----|------|------|
| 01 | [过渡系统架构](06-过渡效果/01-过渡系统架构.md) | TransIntf、过渡处理器、时间线管理 |
| 02 | [内置过渡效果](06-过渡效果/02-内置过渡效果.md) | 淡入淡出、滑动、百叶窗等内置效果 |
| 03 | [自定义过渡实现](06-过渡效果/03-自定义过渡实现.md) | 规则图过渡、自定义过渡效果开发 |

### 第 7 章：实战 — 添加新图像格式

综合实战：从零实现一个新的图像格式支持。

| 节 | 文件 | 内容 |
|----|------|------|
| 01 | [图像格式接口](07-实战-添加新图像格式/01-图像格式接口.md) | 接口约定、回调签名、元数据处理 |
| 02 | [编解码器实现](07-实战-添加新图像格式/02-编解码器实现.md) | 完整实现 AVIF 格式加载器 |
| 03 | [集成与测试](07-实战-添加新图像格式/03-集成与测试.md) | CMake 集成、注册、单元测试、性能对比 |

## 学习路线建议

```
第 1 章（总览）──→ 第 2 章（图层）──→ 第 3 章（图像加载）──┐
                                                              │
     ┌──────────────────────────────────────────────────────── ┘
     │
     ├──→ 第 4 章（OGL 后端）
     │
     ├──→ 第 5 章（软件渲染）
     │
     └──→ 第 6 章（过渡效果）──→ 第 7 章（实战）
```

- 第 1-3 章为**核心基础**，必须按顺序学习
- 第 4-6 章可**并行学习**，彼此独立
- 第 7 章为**综合实战**，需完成前 6 章后再开始

## 前置知识检查

开始本模块前，请确认你已掌握：

- [x] C++17 基础（模板、智能指针、RAII）— 见 [P03](../P03-跨平台C++开发/README.md)
- [x] OpenGL 基础（着色器、纹理、帧缓冲）— 见 [P04](../P04-OpenGL图形编程/README.md)
- [x] 软件渲染原理（像素操作、alpha 混合）— 见 [P05](../P05-软件渲染原理/README.md)
- [x] CMake 构建系统 — 见 [P01](../P01-现代CMake与构建工具链/README.md)
- [x] KrKr2 项目结构 — 见 [M01](../M01-项目导览与环境搭建/README.md)

## 下一步

完成本模块后，继续学习：
- [M05-音频子系统](../M05-音频子系统/README.md) — 音频解码、DSP、波形管理
