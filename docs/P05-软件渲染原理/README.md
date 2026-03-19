# P05 — 软件渲染原理

> **前置知识：** [P03-跨平台C++开发](../P03-跨平台C++开发/)、C++ 基础（指针、位运算、模板）
> **后续模块：** [M04-渲染子系统](../M04-渲染子系统/)、[P10-Cocos2d-x 框架](../P10-Cocos2d-x框架/)

## 模块概述

本模块讲解 CPU 端像素级渲染技术，即**不依赖 GPU** 的纯软件图像处理。KrKr2 引擎中的 `tvpgl` 库（14000+ 行 C 代码）是整个视觉子系统的底层基石——即使在启用 OpenGL 加速的情况下，图层合成、Alpha 混合、图像缩放等操作仍大量使用 CPU 软渲染。掌握这些原理，是深入理解 KrKr2 渲染管线（M04）的必要前提。

## 章节列表

| 章 | 标题 | 内容概要 | 预计阅读 |
|----|------|----------|----------|
| 01 | [像素与颜色空间](./01-像素与颜色空间/) | ARGB/RGBA 内存布局；预乘 Alpha；颜色空间转换（sRGB/线性）；KrKr2 的 `tjs_uint32` 像素格式 | 35 分钟 |
| 02 | [Alpha 混合与合成](./02-Alpha混合与合成/) | Porter-Duff 12 种合成模式；逐像素混合公式；查找表优化；KrKr2 的 `TVPAlphaBlend` 系列函数 | 40 分钟 |
| 03 | [图像缩放与插值](./03-图像缩放与插值/) | 最近邻/双线性/双三次/Lanczos 插值；旋转与仿射变换；KrKr2 的 `ResampleImage` 实现 | 35 分钟 |
| 04 | [SIMD 优化基础](./04-SIMD优化基础/) | SSE2/SSE4/AVX2 指令集；`_mm_*` intrinsics；批量像素处理；ARM NEON 对比 | 40 分钟 |
| 05 | [实战：从零实现软件光栅化器](./05-实战-软件光栅化器/) | 画线→填充→纹理映射→Alpha 混合→SIMD 加速的完整实现 | 50 分钟 |

## 术语预览

本模块涉及大量像素处理和 CPU 优化术语，先在这里建立印象，各章正文会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| ARGB | Alpha-Red-Green-Blue | 一种像素格式，4 个字节分别存储透明度和三原色——KrKr2 内部统一使用此格式 |
| 预乘 Alpha | Premultiplied Alpha | 将 RGB 预先乘以 Alpha 的存储方式，简化混合公式并避免半透明边缘白边 |
| sRGB / 线性 | sRGB / Linear Color Space | sRGB 是显示器使用的非线性颜色空间，计算混合时需先转为线性空间才能得到正确结果 |
| Porter-Duff | — | 1984 年由 Porter 和 Duff 提出的 12 种图像合成规则，定义了前景与背景如何按 Alpha 值组合 |
| Alpha 混合 | Alpha Blending | 最常用的 Porter-Duff 模式（SRC_OVER），用前景的 Alpha 值控制前景与背景的混合比例 |
| tvpgl | TVP Graphics Library | KiriKiri 引擎自带的像素操作 C 函数库，提供 `TVPAlphaBlend` 等 200 多个混合/合成函数 |
| 双线性插值 | Bilinear Interpolation | 图像缩放时用周围 4 个像素的加权平均估算新像素值，效果比最近邻平滑但会略模糊 |
| 双三次插值 | Bicubic Interpolation | 图像缩放时用周围 16 个像素做三次多项式插值，比双线性更锐利但计算量更大 |
| Lanczos | — | 基于 sinc 函数的高质量图像缩放算法，保边能力最强，计算量最大 |
| SIMD | Single Instruction, Multiple Data | 一条 CPU 指令同时处理多个数据——用于加速像素批量运算（如同时混合 4 个像素） |
| SSE2 / SSE4 | Streaming SIMD Extensions | Intel/AMD x86 CPU 的 128 位 SIMD 指令集，一次处理 4 个 32 位像素 |
| AVX2 | Advanced Vector Extensions 2 | x86 CPU 的 256 位 SIMD 指令集，一次处理 8 个 32 位像素 |
| NEON | ARM Advanced SIMD | ARM CPU（Android/iOS/Apple Silicon）的 128 位 SIMD 指令集，功能类似 SSE |
| 光栅化 | Rasterization | 将几何图形（线段、三角形）转换为像素网格上的点的过程 |

## 关键源文件

学习本模块时，建议对照以下 KrKr2 源文件阅读：

| 文件 | 行数 | 说明 |
|------|------|------|
| `cpp/core/visual/tvpgl.cpp` | ~14137 | 核心软渲染库：查找表初始化、像素操作函数、TLG 编解码辅助 |
| `cpp/core/visual/tvpgl.h` | ~1107 | 软渲染 API 声明：Alpha 混合、饱和加法、颜色乘法等内联函数 |
| `cpp/core/visual/gl/blend_function.cpp` | ~685 | 模板化混合函数：Photoshop 级合成模式（柔光/颜色减淡/颜色加深/叠加） |
| `cpp/core/visual/gl/ResampleImage.cpp` | ~919 | 图像缩放：支持 SSE2/AVX2 加速的重采样引擎 |
| `cpp/core/visual/gl/WeightFunctor.cpp` | ~26 | 插值权重函数：双线性/双三次/Spline/Gaussian/Blackman-Sinc |
| `cpp/core/visual/argb.cpp` | ~63 | ARGB 像素模板类：平均值计算 |
| `cpp/core/visual/LayerBitmapIntf.cpp` | ~4820 | 图层位图接口：所有混合模式的调度入口 |

## 学习路径

```
Ch1 像素基础 → Ch2 Alpha混合 → Ch3 图像缩放 → Ch4 SIMD优化 → Ch5 综合实战
     ↓              ↓              ↓              ↓
  理解内存布局    掌握合成公式    掌握插值算法    掌握向量化加速
```

完成本模块后，你将具备：
- 手写逐像素 Alpha 混合的能力
- 理解 KrKr2 `tvpgl` 库中每一个混合函数的数学原理
- 使用 SIMD 指令加速像素处理的能力
- 实现一个完整的 2D 软件光栅化器
