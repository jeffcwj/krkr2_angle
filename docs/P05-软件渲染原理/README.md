# P05 — 软件渲染原理

> **前置知识：** [P03-跨平台C++开发](../P03-跨平台C++开发/)、C++ 基础（指针、位运算、模板）
> **后续模块：** [M04-渲染子系统](../M04-渲染子系统/)、[P10-Cocos2d-x 框架](../P10-Cocos2d-x框架/)

## 模块概述

本模块讲解 CPU 端像素级渲染技术，即**不依赖 GPU** 的纯软件图像处理。KrKr2 引擎中的 `tvpgl` 库（14000+ 行 C 代码）是整个视觉子系统的底层基石——即使在启用 OpenGL 加速的情况下，图层合成、Alpha 混合、图像缩放等操作仍大量使用 CPU 软渲染。掌握这些原理，是深入理解 KrKr2 渲染管线（M04）的必要前提。

## 章节列表

| 章 | 标题 | 内容概要 | 预计阅读 |
|----|------|----------|----------|
| 01 | [像素与颜色空间](./01-像素与颜色空间/01-像素与颜色空间.md) | ARGB/RGBA 内存布局；预乘 Alpha；颜色空间转换（sRGB/线性）；KrKr2 的 `tjs_uint32` 像素格式 | 35 分钟 |
| 02 | [Alpha 混合与合成](./02-Alpha混合与合成/01-Alpha混合与合成.md) | Porter-Duff 12 种合成模式；逐像素混合公式；查找表优化；KrKr2 的 `TVPAlphaBlend` 系列函数 | 40 分钟 |
| 03 | [图像缩放与插值](./03-图像缩放与插值/01-图像缩放与插值.md) | 最近邻/双线性/双三次/Lanczos 插值；旋转与仿射变换；KrKr2 的 `ResampleImage` 实现 | 35 分钟 |
| 04 | [SIMD 优化基础](./04-SIMD优化基础/01-SIMD优化基础.md) | SSE2/SSE4/AVX2 指令集；`_mm_*` intrinsics；批量像素处理；ARM NEON 对比 | 40 分钟 |
| 05 | [实战：从零实现软件光栅化器](./05-实战-软件光栅化器/01-实战-软件光栅化器.md) | 画线→填充→纹理映射→Alpha 混合→SIMD 加速的完整实现 | 50 分钟 |

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
