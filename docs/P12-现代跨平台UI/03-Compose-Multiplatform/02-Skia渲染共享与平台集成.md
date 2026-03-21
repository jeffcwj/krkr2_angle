# Skia 渲染共享与平台集成

> **所属模块：** P12-现代跨平台 UI（Flutter/Compose）
> **前置知识：** [01-Kotlin-Native 与 Cinterop 基础](./01-Kotlin-Native与Cinterop基础.md)、[P04-OpenGL 图形编程](../../P04-OpenGL图形编程/README.md)
> **预计阅读时间：** 55 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：

1. 理解 Compose Multiplatform 底层使用 Skia 渲染引擎的完整架构
2. 掌握 Skiko（Skia for Kotlin）库的核心 API 与渲染管线
3. 在 Compose Multiplatform 中创建自定义 Canvas 渲染组件
4. 设计 C++ 游戏引擎与 Compose Multiplatform 共享 Skia/OpenGL 上下文的方案
5. 分析四平台（Windows/Linux/macOS/Android）下 Skia 渲染后端的差异与适配策略

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| Skia | Skia Graphics Library | Google 开源的 2D 图形渲染引擎，Chrome、Android、Flutter 的底层画布 |
| Skiko | Skia for Kotlin | JetBrains 封装 Skia 为 Kotlin/JVM 与 Kotlin/Native 可用的绑定库 |
| SkiaLayer | SkiaLayer | Skiko 中负责管理渲染表面和 GPU 上下文的核心类 |
| RenderTarget | Render Target | GPU 渲染输出的目标表面，可以是窗口表面或离屏 FBO（帧缓冲对象） |
| DirectContext | Direct Context | Skia 的 GPU 上下文对象，管理 GPU 资源（纹理、着色器缓存、管线状态）的生命周期 |
| Surface | Surface (Skia) | Skia 中代表一块可绘制区域的对象，承载 Canvas 的所有绘制操作 |
| Canvas | Canvas (Skia) | Skia 的核心绘图 API，提供 drawRect/drawPath/drawImage 等 2D 绘制命令 |
| BackendRenderTarget | Backend Render Target | 对底层图形 API（OpenGL/Vulkan/Metal）渲染目标的 Skia 封装 |
| 共享上下文 | Shared GL Context | 两个 OpenGL 上下文共享纹理等 GPU 资源的机制，允许不同线程/模块访问相同纹理 |
| 纹理互操作 | Texture Interop | 在不同渲染系统之间传递纹理数据的技术方案 |

---

## 1. Skia 渲染引擎概述

### 1.1 什么是 Skia？

Skia 是 Google 主导开发的开源 2D 图形渲染引擎。它不是一个独立应用，而是一个**底层渲染库**——其他软件调用 Skia 的 API 来绘制文字、图形、图片等 2D 内容。Skia 的设计目标是高性能、跨平台、可嵌入，因此它被用在了许多你每天都在使用的软件中：

- **Chrome / Chromium 浏览器**：网页中所有 2D 内容（文字、CSS 边框、Canvas 元素）都由 Skia 绘制
- **Android 操作系统**：从 Android 1.0 开始，所有 UI 控件的绘制底层都是 Skia
- **Flutter 框架**：Flutter 的 `CustomPaint`、文字渲染、图片解码全部基于 Skia（Flutter 3.x 后还有 Impeller 作为替代方案）
- **Compose Multiplatform**：JetBrains 通过 Skiko 库将 Skia 引入 Kotlin 生态

Skia 的核心 API 是 C++ 编写的，通过 `SkCanvas`（画布）、`SkPaint`（画笔）、`SkPath`（路径）等类提供绘图能力。它本身**不管理窗口**——窗口创建、事件循环、GPU 上下文初始化都由上层负责，Skia 只负责"往一块内存/GPU 表面上画东西"。

```
┌─────────────────────────────────────────────┐
│              应用程序 / 框架                  │
│   (Chrome / Android / Flutter / Compose)     │
├─────────────────────────────────────────────┤
│              Skia 2D 渲染引擎                │
│   SkCanvas / SkPaint / SkPath / SkImage     │
├─────────────────────────────────────────────┤
│            GPU 后端 (可选择)                  │
│   OpenGL │ Vulkan │ Metal │ Direct3D │ CPU  │
├─────────────────────────────────────────────┤
│              操作系统 / 硬件                  │
│   Windows │ Linux │ macOS │ Android │ iOS   │
└─────────────────────────────────────────────┘
```

### 1.2 Skia 的渲染后端

Skia 支持多种渲染后端（Backend），即底层图形 API。选择不同后端，Skia 会将绘制命令翻译为对应 API 的调用：

| 后端 | 说明 | 适用平台 |
|------|------|----------|
| **OpenGL / OpenGL ES** | 最广泛支持的 GPU 后端，Skia 默认首选 | Windows、Linux、Android、macOS（已废弃但仍可用） |
| **Vulkan** | 新一代低开销 GPU API，性能更高但编程复杂 | Windows、Linux、Android |
| **Metal** | Apple 专属 GPU API，macOS/iOS 上性能最佳 | macOS、iOS |
| **Direct3D** | Windows 专属，Skia 支持 Direct3D 11/12 | Windows |
| **CPU（软件渲染）** | 不使用 GPU，纯 CPU 光栅化，用于无 GPU 环境或测试 | 所有平台 |

对于 KrKr2 项目，最重要的后端是 **OpenGL**——因为 Cocos2d-x 引擎本身使用 OpenGL ES 进行渲染。如果要让 Compose Multiplatform 的 Skia 渲染与 Cocos2d-x 的 OpenGL 渲染共存，两者需要在同一个 OpenGL 上下文（或共享上下文）中工作。

### 1.3 Skia 核心渲染流程

理解 Skia 的渲染流程对后续的共享方案设计至关重要。Skia 从接收绘制命令到最终像素输出，经历以下阶段：

```
绘制命令                    记录 / 立即执行           GPU 提交
    │                            │                      │
    ▼                            ▼                      ▼
SkCanvas.drawXXX()  ──►  SkPicture (录制模式)  ──►  flush()
    │                     或直接提交到 GPU              │
    │                                                   ▼
    │                                            GrDirectContext
    │                                            执行 GPU 命令
    ▼                                                   │
SkPaint 控制样式                                        ▼
 (颜色/抗锯齿/                                   帧缓冲 / 纹理
  混合模式/着色器)                                 (最终像素)
```

**关键概念说明：**

1. **SkCanvas**：所有绘制操作的入口。调用 `drawRect()`、`drawPath()`、`drawImage()` 等方法发出绘制命令
2. **SkPaint**：控制"怎么画"——颜色、透明度、抗锯齿、混合模式（BlendMode）、着色器（Shader）等
3. **GrDirectContext**：GPU 模式下的上下文管理器。它持有 GPU 连接（OpenGL 上下文或 Vulkan Device），管理纹理缓存、着色器编译缓存、管线对象池等 GPU 资源
4. **flush()**：将积累的 GPU 命令真正提交给 GPU 执行。在与其他 OpenGL 代码共存时，flush 的时机至关重要

---

## 2. Skiko：Skia 的 Kotlin 绑定

### 2.1 Skiko 是什么？

Skiko（全称 Skia for Kotlin）是 JetBrains 开发的库，将 C++ Skia 引擎封装为 Kotlin 可调用的 API。它是 Compose Multiplatform 在桌面端和 Web 端的渲染基础——Android 端的 Compose 直接使用 Android 系统内置的 Skia，而桌面端（Windows/Linux/macOS）则通过 Skiko 自带一份 Skia 二进制库。

Skiko 的封装层次：

```
┌──────────────────────────────────┐
│     Compose Multiplatform UI     │  ← Kotlin DSL 声明式 UI
├──────────────────────────────────┤
│     Compose Runtime / Renderer   │  ← 布局计算、重组调度
├──────────────────────────────────┤
│          Skiko (Kotlin API)      │  ← org.jetbrains.skia.*
│   Canvas / Paint / Path / Image  │
├──────────────────────────────────┤
│       Skiko Native (JNI/K/N)     │  ← C++ 胶水层
├──────────────────────────────────┤
│         Skia C++ Engine          │  ← 实际渲染
├──────────────────────────────────┤
│     OpenGL / Vulkan / Metal      │  ← GPU 后端
└──────────────────────────────────┘
```

### 2.2 Skiko 在不同平台的实现方式

Skiko 根据目标平台使用不同的技术将 Kotlin 代码连接到 C++ Skia：

| 平台 | 绑定方式 | Skia 二进制来源 | 默认 GPU 后端 |
|------|----------|----------------|--------------|
| **Windows** | JNI（Java Native Interface） | Skiko 打包的 `.dll`（`skiko-windows-x64.dll`） | Direct3D 12（可选 OpenGL） |
| **Linux** | JNI | Skiko 打包的 `.so`（`libskiko-linux-x64.so`） | OpenGL |
| **macOS** | JNI | Skiko 打包的 `.dylib`（`libskiko-macos-arm64.dylib`） | Metal |
| **Android** | Android 系统 Skia（Compose 不用 Skiko） | 系统内置 | OpenGL ES / Vulkan |

> **注意**：Android 上的 Compose（即 Jetpack Compose）直接使用 Android 系统内置的 Skia 和 `android.graphics.Canvas`，**不经过 Skiko**。只有桌面端和 Web 端才使用 Skiko。但 Compose Multiplatform 的 Kotlin API 在所有平台是统一的。

### 2.3 Skiko 核心类与 API

Skiko 将 Skia C++ API 映射为 Kotlin 类。以下是最常用的核心类：

```kotlin
// 文件：Skiko 核心类概览（基于 org.jetbrains.skia 包）

import org.jetbrains.skia.*

// 1. Canvas — 画布，所有绘制操作的入口
// 对应 C++ 的 SkCanvas
val canvas: Canvas = surface.canvas
canvas.drawRect(Rect.makeXYWH(10f, 10f, 100f, 50f), paint)  // 画矩形
canvas.drawCircle(100f, 100f, 50f, paint)                     // 画圆
canvas.drawString("Hello Skia", 50f, 200f, font, paint)       // 画文字
canvas.drawImage(image, 0f, 0f)                               // 画图片

// 2. Paint — 画笔，控制绘制样式
// 对应 C++ 的 SkPaint
val paint = Paint().apply {
    color = Color.makeRGB(255, 0, 0)    // 红色
    mode = PaintMode.FILL               // 填充模式（FILL / STROKE / STROKE_AND_FILL）
    isAntiAlias = true                  // 开启抗锯齿
    strokeWidth = 2f                    // 描边宽度（STROKE 模式下生效）
    blendMode = BlendMode.SRC_OVER     // 混合模式（源覆盖目标，默认值）
}

// 3. Path — 路径，描述复杂形状
// 对应 C++ 的 SkPath
val path = Path().apply {
    moveTo(0f, 0f)          // 移动到起点
    lineTo(100f, 0f)        // 画直线到 (100, 0)
    quadTo(100f, 100f, 0f, 100f)  // 二次贝塞尔曲线
    closePath()             // 封闭路径
}
canvas.drawPath(path, paint)

// 4. Surface — 绘制表面，承载 Canvas
// 对应 C++ 的 SkSurface
val surface = Surface.makeRasterN32Premul(800, 600)  // CPU 表面
val canvas2 = surface.canvas                          // 获取 Canvas
// ... 绘制操作 ...
val image = surface.makeImageSnapshot()               // 截取为 Image

// 5. Font — 字体对象
val typeface = Typeface.makeFromFile("NotoSansSC-Regular.otf")  // 从文件加载字体
val font = Font(typeface, 24f)  // 字号 24
canvas.drawString("中文测试", 10f, 40f, font, paint)
```

上面的代码展示了 Skiko 的 5 个核心类。它们的关系可以这样理解：`Surface` 是"画纸"，`Canvas` 是"画纸上的操作接口"，`Paint` 是"画笔"，`Path` 是"预定义的形状模板"，`Font` 是"字体模板"。

---

## 3. Compose Multiplatform 的渲染管线

### 3.1 从声明式 UI 到像素

Compose Multiplatform 的渲染管线是一个多阶段流水线。理解这个管线对于找到"在哪里插入自定义渲染"和"在哪里与 C++ 引擎对接"至关重要：

```
阶段 1: 组合 (Composition)
┌─────────────────────────────┐
│  @Composable 函数执行        │
│  → 生成/更新 UI 节点树       │
│  → 检测 State 变化触发重组   │
└──────────────┬──────────────┘
               │
阶段 2: 布局 (Layout)
┌──────────────▼──────────────┐
│  测量每个节点的大小           │
│  → 确定每个节点在屏幕上的位置 │
│  → 输出：节点 + 位置 + 大小  │
└──────────────┬──────────────┘
               │
阶段 3: 绘制 (Drawing)
┌──────────────▼──────────────┐
│  遍历节点树，调用 draw 方法   │
│  → 每个节点向 Canvas 发出    │
│    绘制命令                   │
│  → Canvas 命令提交给 Skia    │
└──────────────┬──────────────┘
               │
阶段 4: 渲染 (Rendering)
┌──────────────▼──────────────┐
│  Skia 将命令翻译为 GPU 调用  │
│  → OpenGL / Vulkan / Metal  │
│  → GPU 光栅化 → 像素数据     │
│  → SwapBuffers 显示到屏幕   │
└─────────────────────────────┘
```

**关键洞察**：阶段 3（绘制）是我们介入的主要位置。通过 Compose 的 `Canvas` 组件或 `Modifier.drawBehind` / `Modifier.drawWithContent`，可以在绘制阶段插入自定义的 Skia 绘制命令。而阶段 4（渲染）是与 C++ 引擎共享 GPU 资源的关键位置。

### 3.2 桌面端渲染架构（SkiaLayer）

在桌面端（Windows/Linux/macOS），Compose Multiplatform 使用 `SkiaLayer` 作为渲染表面管理器。`SkiaLayer` 负责：

1. **创建操作系统窗口**：通过 AWT/Swing 创建原生窗口
2. **初始化 GPU 上下文**：根据平台选择 OpenGL/Metal/Direct3D，创建 `GrDirectContext`
3. **管理渲染循环**：每帧创建 `Surface` → 获取 `Canvas` → 执行绘制 → `flush()` → `SwapBuffers`
4. **处理窗口事件**：resize、DPI 变化、窗口失焦等

```kotlin
// 简化的 SkiaLayer 渲染循环伪代码
// （实际代码在 Skiko 源码 org.jetbrains.skiko.SkiaLayer 中）

class SkiaLayer {
    private var directContext: DirectContext? = null  // GPU 上下文
    private var renderTarget: BackendRenderTarget? = null  // 渲染目标

    fun renderFrame() {
        // 1. 确保 GPU 上下文有效
        val context = directContext ?: initializeGpuContext()

        // 2. 获取当前帧的渲染目标（窗口后缓冲）
        val rt = BackendRenderTarget.makeGL(
            width, height,
            sampleCount = 0,       // MSAA 采样数
            stencilBits = 8,       // 模板缓冲位数
            fboId = 0,             // 0 = 默认帧缓冲（窗口表面）
            fbFormat = 0x8058      // GL_RGBA8
        )

        // 3. 从渲染目标创建 Surface
        val surface = Surface.makeFromBackendRenderTarget(
            context, rt, SurfaceOrigin.BOTTOM_LEFT,
            SurfaceColorType.RGBA_8888, ColorSpace.sRGB
        )!!

        // 4. 获取 Canvas 并执行 Compose 的绘制命令
        val canvas = surface.canvas
        canvas.clear(Color.WHITE)
        composeScene.render(canvas, nanoTime)  // Compose 渲染入口

        // 5. 提交 GPU 命令并交换缓冲
        context.flush()
        swapBuffers()

        // 6. 清理本帧资源
        surface.close()
        rt.close()
    }
}
```

上面的伪代码揭示了几个与 C++ 引擎共享相关的关键信息：

- **`DirectContext`** 是 GPU 资源的管理者，如果我们能获取到它（或创建一个共享的），就能让 C++ 引擎和 Compose 共享 GPU 纹理
- **`BackendRenderTarget`** 封装了底层 FBO，C++ 引擎可以渲染到一个 FBO，然后让 Skia 将该 FBO 作为纹理读取
- **`flush()`** 的时机决定了 GPU 命令的执行顺序——C++ 引擎和 Compose 的渲染必须正确同步

---

## 4. 在 Compose 中自定义渲染

### 4.1 Canvas 组件基础

Compose Multiplatform 提供了 `Canvas` 组件（注意：这是 Compose 的 `Canvas` 组件，不是 Skia 的 `Canvas` 类），允许在声明式 UI 中插入自定义绘制逻辑：

```kotlin
// 示例 1：Compose Canvas 基础绘制
// 文件：src/main/kotlin/BasicCanvasExample.kt

import androidx.compose.foundation.Canvas
import androidx.compose.foundation.layout.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.geometry.Size
import androidx.compose.ui.graphics.*
import androidx.compose.ui.graphics.drawscope.*
import androidx.compose.ui.unit.dp
import androidx.compose.ui.window.singleWindowApplication

fun main() = singleWindowApplication(
    title = "Compose Canvas 基础示例"
) {
    Column(modifier = Modifier.fillMaxSize().padding(16.dp)) {
        // Canvas 组件 — 提供 DrawScope 上下文
        Canvas(
            modifier = Modifier
                .fillMaxWidth()
                .height(400.dp)
        ) {
            // DrawScope 提供 size（画布大小）和各种 draw 方法
            val canvasWidth = size.width
            val canvasHeight = size.height

            // 1. 画背景矩形（深蓝色）
            drawRect(
                color = Color(0xFF1A1A2E),  // ARGB 格式
                topLeft = Offset.Zero,
                size = size
            )

            // 2. 画网格线（半透明白色）
            val gridSpacing = 50f
            val gridPaint = Color.White.copy(alpha = 0.1f)
            var x = 0f
            while (x < canvasWidth) {
                drawLine(gridPaint, Offset(x, 0f), Offset(x, canvasHeight))
                x += gridSpacing
            }
            var y = 0f
            while (y < canvasHeight) {
                drawLine(gridPaint, Offset(0f, y), Offset(canvasWidth, y))
                y += gridSpacing
            }

            // 3. 画填充圆（红色，居中）
            drawCircle(
                color = Color(0xFFE94560),
                radius = 80f,
                center = Offset(canvasWidth / 2, canvasHeight / 2)
            )

            // 4. 画描边矩形（绿色）
            drawRect(
                color = Color(0xFF0F3460),
                topLeft = Offset(canvasWidth / 2 - 120f, canvasHeight / 2 - 60f),
                size = Size(240f, 120f),
                style = Stroke(width = 3f)  // 描边模式
            )

            // 5. 画路径（三角形，黄色半透明）
            val trianglePath = Path().apply {
                moveTo(canvasWidth / 2, canvasHeight / 2 - 100f)  // 顶点
                lineTo(canvasWidth / 2 - 80f, canvasHeight / 2 + 60f)  // 左下
                lineTo(canvasWidth / 2 + 80f, canvasHeight / 2 + 60f)  // 右下
                close()
            }
            drawPath(
                path = trianglePath,
                color = Color(0x80FFD700),  // 半透明金色
                style = Fill
            )
        }
    }
}
```

### 4.2 使用原生 Skia API（nativeCanvas）

Compose 的 `DrawScope` 提供了简化的绘图 API，但有时需要直接访问底层 Skia `Canvas` 以使用更高级的功能（如着色器、图像滤镜、混合模式）。在桌面端，可以通过 `drawIntoCanvas` 获取原生 Skia Canvas：

```kotlin
// 示例 2：访问原生 Skia Canvas 进行高级绘制
// 文件：src/main/kotlin/NativeCanvasExample.kt

import androidx.compose.foundation.Canvas
import androidx.compose.foundation.layout.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.drawscope.drawIntoCanvas
import androidx.compose.ui.graphics.nativeCanvas
import androidx.compose.ui.unit.dp
import androidx.compose.ui.window.singleWindowApplication
import org.jetbrains.skia.*          // Skia 原生 API
import org.jetbrains.skia.Font
import org.jetbrains.skia.Paint

fun main() = singleWindowApplication(
    title = "原生 Skia Canvas 示例"
) {
    Canvas(
        modifier = Modifier.fillMaxSize().padding(16.dp)
    ) {
        drawIntoCanvas { composeCanvas ->
            // 获取底层 Skia Canvas（桌面端返回 org.jetbrains.skia.Canvas）
            val skiaCanvas: org.jetbrains.skia.Canvas = composeCanvas.nativeCanvas

            // --- 使用 Skia 原生 API 绘制 ---

            // 1. 创建带有线性渐变的画笔
            val gradientPaint = Paint().apply {
                shader = Shader.makeLinearGradient(
                    x0 = 0f, y0 = 0f,
                    x1 = size.width, y1 = size.height,
                    colors = intArrayOf(
                        Color.makeRGB(255, 0, 128),   // 粉红
                        Color.makeRGB(0, 128, 255),    // 蓝色
                        Color.makeRGB(128, 255, 0)     // 绿色
                    ),
                    positions = floatArrayOf(0f, 0.5f, 1f),  // 颜色位置
                    style = GradientStyle.DEFAULT
                )
                isAntiAlias = true
            }

            // 2. 画一个带渐变填充的圆角矩形
            val rrect = RRect.makeXYWH(
                50f, 50f, size.width - 100f, 200f,
                20f  // 圆角半径
            )
            skiaCanvas.drawRRect(rrect, gradientPaint)

            // 3. 使用 Skia Font 绘制文字（支持更多字体特性）
            val typeface = Typeface.makeDefault()
            val font = Font(typeface, 32f).apply {
                isSubpixel = true    // 亚像素渲染，文字更清晰
                edging = FontEdging.SUBPIXEL_ANTI_ALIAS
            }
            val textPaint = Paint().apply {
                color = Color.WHITE
                isAntiAlias = true
            }
            skiaCanvas.drawString(
                "Skia 原生渲染 — 渐变 + 圆角",
                80f, 160f, font, textPaint
            )

            // 4. 使用 ImageFilter 添加模糊效果
            val blurPaint = Paint().apply {
                color = Color.makeRGB(255, 165, 0)  // 橙色
                imageFilter = ImageFilter.makeBlur(
                    sigmaX = 5f, sigmaY = 5f,   // 高斯模糊半径
                    mode = FilterTileMode.CLAMP  // 边缘处理模式
                )
            }
            skiaCanvas.drawCircle(
                size.width / 2, 400f, 60f, blurPaint
            )

            // 5. 路径特效 — 虚线描边
            val dashPaint = Paint().apply {
                color = Color.makeRGB(0, 255, 200)
                mode = PaintMode.STROKE
                strokeWidth = 3f
                pathEffect = PathEffect.makeDash(
                    floatArrayOf(15f, 8f, 5f, 8f),  // 线段长度交替
                    phase = 0f  // 起始偏移
                )
            }
            skiaCanvas.drawRRect(
                RRect.makeXYWH(50f, 300f, size.width - 100f, 150f, 15f),
                dashPaint
            )

            // 清理资源
            gradientPaint.close()
            textPaint.close()
            blurPaint.close()
            dashPaint.close()
        }
    }
}
```

**`drawIntoCanvas` + `nativeCanvas`** 的组合是在 Compose 中使用完整 Skia 能力的标准方式。在桌面端，`nativeCanvas` 返回的就是 `org.jetbrains.skia.Canvas`；在 Android 端返回的是 `android.graphics.Canvas`。如果需要写跨平台代码，应优先使用 Compose 自带的 `DrawScope` API。

### 4.3 动画渲染与帧驱动

游戏引擎需要每帧更新渲染内容。在 Compose 中，可以使用 `LaunchedEffect` + `withFrameNanos` 实现帧驱动的动画渲染：

```kotlin
// 示例 3：帧驱动动画 — 模拟游戏渲染循环
// 文件：src/main/kotlin/AnimatedCanvasExample.kt

import androidx.compose.foundation.Canvas
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.drawscope.DrawScope
import androidx.compose.ui.window.singleWindowApplication
import kotlin.math.*

// 粒子数据类
data class Particle(
    var x: Float,        // 位置 X
    var y: Float,        // 位置 Y
    var vx: Float,       // 速度 X
    var vy: Float,       // 速度 Y
    var radius: Float,   // 半径
    var color: Color,    // 颜色
    var life: Float      // 剩余寿命（0.0 ~ 1.0）
)

fun main() = singleWindowApplication(
    title = "帧驱动动画示例"
) {
    // 粒子列表（状态驱动重组）
    val particles = remember { mutableStateListOf<Particle>() }
    // 帧计数器（每次变化触发重绘）
    var frameCount by remember { mutableLongStateOf(0L) }

    // 帧驱动循环 — 每帧更新粒子状态
    LaunchedEffect(Unit) {
        while (true) {
            // withFrameNanos 挂起直到下一次 VSync（垂直同步信号）
            withFrameNanos { nanos ->
                val dt = 0.016f  // 假设 60fps，每帧约 16ms

                // 生成新粒子
                if (particles.size < 200) {
                    val angle = (nanos / 1_000_000_000.0 * 2.0).toFloat()
                    particles.add(Particle(
                        x = 400f, y = 300f,
                        vx = cos(angle) * (50f + (nanos % 100)),
                        vy = sin(angle) * (50f + (nanos % 80)) - 100f,
                        radius = 3f + (nanos % 5),
                        color = Color.hsv(
                            hue = ((nanos / 10_000_000) % 360).toFloat(),
                            saturation = 0.8f,
                            value = 1.0f
                        ),
                        life = 1.0f
                    ))
                }

                // 更新所有粒子
                val iterator = particles.listIterator()
                while (iterator.hasNext()) {
                    val p = iterator.next()
                    p.x += p.vx * dt          // 位移
                    p.y += p.vy * dt
                    p.vy += 200f * dt          // 重力
                    p.life -= 0.5f * dt        // 衰减
                    if (p.life <= 0f) {
                        iterator.remove()       // 移除死亡粒子
                    }
                }

                frameCount++  // 触发重绘
            }
        }
    }

    // 渲染 — 每次 frameCount 变化时重绘
    Canvas(modifier = Modifier.fillMaxSize()) {
        // 使用 frameCount 确保编译器不会优化掉重组
        @Suppress("UNUSED_EXPRESSION")
        frameCount

        // 半透明黑色背景（产生拖尾效果）
        drawRect(Color(0xDD000000.toInt()))

        // 绘制所有粒子
        for (p in particles) {
            drawCircle(
                color = p.color.copy(alpha = p.life.coerceIn(0f, 1f)),
                radius = p.radius * p.life,
                center = Offset(p.x, p.y)
            )
        }
    }
}
```

上述示例展示了在 Compose 中实现游戏级帧驱动渲染的模式。核心机制是 `withFrameNanos`——它等待 VSync 信号（垂直同步，即显示器准备好接收下一帧的时刻），然后触发一次更新。这与传统游戏引擎的 `requestAnimationFrame`（Web）或 `glutDisplayFunc`（GLUT）是同一概念。

**性能注意事项**：
- `mutableStateListOf` 的修改会触发 Compose 重组，大量粒子时可能成为瓶颈
- 生产级实现应使用 `SnapshotStateList` 或直接操作 Skia Canvas 避免 Compose 重组开销
- 对于真正的游戏渲染，更好的方式是让 C++ 引擎在独立线程渲染到纹理，Compose 只负责显示该纹理

---

## 5. C++ 引擎与 Compose Multiplatform 的渲染共享方案

### 5.1 为什么需要渲染共享？

KrKr2 是一个 C++ 游戏引擎，使用 Cocos2d-x（OpenGL ES）进行渲染。如果我们想用 Compose Multiplatform 替换 UI 层，面临一个核心问题：**游戏画面（C++ 渲染）和 UI 界面（Compose 渲染）如何合成到同一个窗口？**

有三种基本思路：

```
方案 A：纹理共享（共享 GPU 纹理）
┌──────────────────────────┐
│        最终窗口            │
│  ┌────────────────────┐  │
│  │  Compose UI 层      │  │ ← Compose 渲染（Skia/Skiko）
│  │  ┌──────────────┐  │  │
│  │  │ 游戏画面纹理  │  │  │ ← C++ 引擎渲染到 FBO → Compose 显示为 Image
│  │  └──────────────┘  │  │
│  └────────────────────┘  │
└──────────────────────────┘

方案 B：像素拷贝（CPU 传输）
┌──────────────────────────┐
│        最终窗口            │
│  ┌────────────────────┐  │
│  │  Compose UI 层      │  │
│  │  ┌──────────────┐  │  │
│  │  │ 游戏画面位图  │  │  │ ← C++ 引擎渲染 → glReadPixels → 位图 → Compose Image
│  │  └──────────────┘  │  │
│  └────────────────────┘  │
└──────────────────────────┘

方案 C：分层窗口（各自独立窗口叠加）
┌──────────────────────────┐
│  Compose 窗口 (透明)      │ ← 顶层：UI 按钮、对话框
├──────────────────────────┤
│  游戏引擎窗口              │ ← 底层：C++ 渲染的游戏画面
└──────────────────────────┘
```

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| A: 纹理共享 | 零拷贝、高性能、GPU 完成全部合成 | 实现复杂、需要共享 GL 上下文、平台差异大 | 性能要求高的游戏引擎 |
| B: 像素拷贝 | 实现简单、无需共享上下文 | glReadPixels 慢、CPU-GPU 往返、延迟高 | 原型验证、低帧率场景 |
| C: 分层窗口 | 完全解耦、各自独立渲染 | 窗口同步困难、透明度处理复杂、移动端不可用 | 仅桌面端调试工具 |

**结论**：对于 KrKr2 这样需要实时游戏渲染 + UI 覆盖的场景，**方案 A（纹理共享）是正确选择**。方案 B 可作为快速原型验证方案。方案 C 仅用于极特殊情况。

### 5.2 方案 A 详解：OpenGL 纹理共享

OpenGL 纹理共享的核心思路：C++ 引擎渲染到一个 FBO（帧缓冲对象），该 FBO 的颜色附件是一个纹理。然后在 Compose 侧，通过 Skia 的 `BackendTexture` API 将同一个 OpenGL 纹理包装为 Skia Image，最终在 Compose 的 `Canvas` 中绘制。

**前提条件**：C++ 引擎和 Skiko 必须在同一个 OpenGL 上下文中工作，或者使用共享上下文（`wglShareLists` / `glXCreateContext` 共享）。

```kotlin
// 示例 4：将 OpenGL 纹理包装为 Skia Image（桌面端）
// 文件：src/main/kotlin/TextureInterop.kt

import org.jetbrains.skia.*

/**
 * 将 C++ 引擎渲染的 OpenGL 纹理包装为 Skia Image
 *
 * @param directContext  Skia 的 GPU 上下文（从 SkiaLayer 获取）
 * @param glTextureId    C++ 引擎渲染的 OpenGL 纹理 ID
 * @param width          纹理宽度（像素）
 * @param height         纹理高度（像素）
 * @return               可在 Compose Canvas 中绘制的 Skia Image
 */
fun wrapGlTextureAsSkiaImage(
    directContext: DirectContext,
    glTextureId: Int,
    width: Int,
    height: Int
): Image {
    // 1. 创建 BackendTexture — 告诉 Skia 这个 OpenGL 纹理的元信息
    //    Skia 不会管理这个纹理的生命周期（不会删除它）
    val backendTexture = BackendTexture.makeGL(
        width = width,
        height = height,
        mipmapped = false,           // 是否有 mipmap 链
        glTextureInfo = GlTextureInfo(
            target = 0x0DE1,          // GL_TEXTURE_2D
            id = glTextureId,         // C++ 引擎创建的纹理 ID
            format = 0x8058           // GL_RGBA8（内部格式）
        )
    )

    // 2. 从 BackendTexture 创建 Skia Image
    //    SurfaceOrigin.BOTTOM_LEFT: OpenGL 纹理坐标 Y 轴朝上
    //    如果 C++ 引擎使用标准 OpenGL 坐标，选 BOTTOM_LEFT
    //    如果 C++ 引擎已翻转（如使用 FBO），可能选 TOP_LEFT
    val image = Image.makeFromTexture(
        context = directContext,
        backendTexture = backendTexture,
        origin = SurfaceOrigin.BOTTOM_LEFT,
        colorType = ColorType.RGBA_8888,
        alphaType = AlphaType.PREMUL,   // 预乘 Alpha
        colorSpace = ColorSpace.sRGB
    )

    return image
}

/**
 * 在 Compose 的绘制阶段使用包装后的纹理
 */
// 在 @Composable 中：
// Canvas(modifier = Modifier.fillMaxSize()) {
//     drawIntoCanvas { canvas ->
//         val skiaCanvas = canvas.nativeCanvas
//         val gameImage = wrapGlTextureAsSkiaImage(
//             directContext, engineTextureId, 1280, 720
//         )
//         skiaCanvas.drawImage(gameImage, 0f, 0f)
//         gameImage.close()  // 关闭 Image（不会删除底层 GL 纹理）
//     }
// }
```

**关键细节说明**：

1. **`BackendTexture.makeGL()`** 不会创建新的 OpenGL 纹理，它只是创建一个 Skia 端的"描述符"，告诉 Skia"有一个 GL 纹理在这个 ID 上，格式是 RGBA8"
2. **`Image.makeFromTexture()`** 创建一个 Skia Image 对象来引用该纹理。关闭此 Image **不会** 删除底层 GL 纹理——纹理的生命周期仍由 C++ 引擎管理
3. **`SurfaceOrigin`** 非常重要：OpenGL 和 Skia 的 Y 轴方向相反。标准 OpenGL 纹理坐标 (0,0) 在左下角（`BOTTOM_LEFT`），而屏幕坐标 (0,0) 在左上角。选错会导致画面上下颠倒

### 5.3 方案 B 详解：像素拷贝方案（快速原型）

当无法共享 GL 上下文时（例如 C++ 引擎和 Compose 运行在不同进程），可以退而求其次使用像素拷贝：

```kotlin
// 示例 5：像素拷贝方案 — C++ 渲染结果通过内存传输到 Compose
// 文件：src/main/kotlin/PixelCopyInterop.kt

import org.jetbrains.skia.*
import kotlinx.cinterop.*   // Kotlin/Native 内存操作

/**
 * C++ 侧接口（通过 cinterop 调用）
 * 
 * 假设已有 cinterop 绑定：
 *   external fun engine_render_frame(pixelBuffer: CPointer<UByteVar>, width: Int, height: Int)
 *   external fun engine_get_frame_width(): Int
 *   external fun engine_get_frame_height(): Int
 */

/**
 * 从 C++ 引擎获取渲染帧并转换为 Skia Image
 * 
 * @return 包含游戏画面的 Skia Image（CPU 模式，不需要 GPU 上下文）
 */
fun captureEngineFrame(): Image {
    val width = 1280   // engine_get_frame_width()
    val height = 720   // engine_get_frame_height()
    val bytesPerPixel = 4  // RGBA
    val rowBytes = width * bytesPerPixel
    val totalBytes = rowBytes * height

    // 1. 分配像素缓冲区
    //    C++ 引擎调用 glReadPixels() 将渲染结果写入此缓冲区
    val pixelBuffer = ByteArray(totalBytes)

    // 2. 调用 C++ 引擎渲染并读回像素
    //    engine_render_frame(pixelBuffer.refTo(0), width, height)
    //    实际实现中，C++ 侧会：
    //      glBindFramebuffer(GL_READ_FRAMEBUFFER, engineFBO);
    //      glReadPixels(0, 0, width, height, GL_RGBA, GL_UNSIGNED_BYTE, buffer);
    //      // 注意：glReadPixels 读出的数据 Y 轴是翻转的（底部在前）

    // 3. 从像素数据创建 Skia Image
    val imageInfo = ImageInfo(
        width = width,
        height = height,
        colorType = ColorType.RGBA_8888,
        alphaType = AlphaType.PREMUL
    )

    val image = Image.makeRaster(
        imageInfo = imageInfo,
        bytes = pixelBuffer,
        rowBytes = rowBytes.toLong()
    )

    return image
}

/**
 * 性能对比：
 * 
 * 分辨率      像素拷贝耗时（估算）    纹理共享耗时
 * 1280x720    ~3-5ms（CPU瓶颈）      ~0.1ms（几乎零开销）
 * 1920x1080   ~8-12ms               ~0.1ms
 * 2560x1440   ~15-20ms              ~0.1ms
 * 
 * 结论：像素拷贝在 1080p 下就已接近一帧的时间预算（16ms@60fps），
 * 仅适合原型验证或低帧率场景。
 */
```

---

## 6. 跨平台 GPU 上下文共享详解

### 6.1 共享上下文的原理

OpenGL 的"共享上下文"（Shared Context）允许两个独立的 OpenGL 上下文共享部分 GPU 资源。共享的资源包括纹理对象、缓冲区对象（VBO/FBO）、着色器程序等"有名字的对象"（Named Objects）。**不共享**的资源包括 VAO（顶点数组对象）、FBO 绑定状态、渲染状态（混合模式、深度测试等）。

```
共享上下文工作原理：

上下文 A（C++ 引擎线程）          上下文 B（Compose/Skia 线程）
┌────────────────────┐           ┌────────────────────┐
│ GL 状态（独立）      │           │ GL 状态（独立）      │
│ - 当前绑定的纹理     │           │ - 当前绑定的纹理     │
│ - 混合模式           │           │ - 混合模式           │
│ - 视口设置           │           │ - 视口设置           │
├────────────────────┤           ├────────────────────┤
│                    │           │                    │
│    ┌───────────────┴───────────┴──────────────┐    │
│    │          共享的 GPU 资源                    │    │
│    │  - 纹理对象 (Texture ID = 42)             │    │
│    │  - 着色器程序 (Program ID = 7)            │    │
│    │  - VBO (Buffer ID = 15)                  │    │
│    └──────────────────────────────────────────┘    │
└────────────────────┘           └────────────────────┘
```

在不同操作系统上创建共享上下文的 API 不同：

| 平台 | 创建共享上下文的 API | 关键参数 |
|------|---------------------|----------|
| **Windows** | `wglCreateContext` + `wglShareLists(contextA, contextB)` | 两个上下文必须使用相同像素格式 |
| **Linux (X11)** | `glXCreateContext(display, visualInfo, shareContext, True)` | 第三个参数传入要共享的上下文 |
| **macOS** | `CGLCreateContext(pixelFormat, shareContext, &newContext)` 或 `NSOpenGLContext.init(format:share:)` | CGL/NSOpenGL 均支持 share 参数 |
| **Android** | `eglCreateContext(display, config, shareContext, attribs)` | 第三个参数传入要共享的 EGL 上下文 |

### 6.2 各平台共享上下文实现

以下是在四个平台上创建共享 GL 上下文的 C++ 代码：

```cpp
// 示例 6：四平台共享 GL 上下文创建
// 文件：src/platform/SharedGLContext.cpp

#include <cstdio>

// ============================
// Windows 实现
// ============================
#ifdef _WIN32
#include <windows.h>
#include <GL/gl.h>
#include <GL/wglext.h>  // WGL 扩展

struct SharedGLContextWin32 {
    HGLRC engineContext;   // C++ 引擎的 GL 上下文
    HGLRC composeContext;  // 为 Compose/Skia 创建的共享上下文
    HDC   hdc;             // 设备上下文（窗口关联）

    bool create(HWND hwnd) {
        hdc = GetDC(hwnd);

        // 设置像素格式（两个上下文必须一致）
        PIXELFORMATDESCRIPTOR pfd = {};
        pfd.nSize = sizeof(pfd);
        pfd.nVersion = 1;
        pfd.dwFlags = PFD_DRAW_TO_WINDOW | PFD_SUPPORT_OPENGL | PFD_DOUBLEBUFFER;
        pfd.iPixelType = PFD_TYPE_RGBA;
        pfd.cColorBits = 32;   // RGBA8
        pfd.cDepthBits = 24;   // 24位深度缓冲
        pfd.cStencilBits = 8;  // 8位模板缓冲

        int pixelFormat = ChoosePixelFormat(hdc, &pfd);
        SetPixelFormat(hdc, pixelFormat, &pfd);

        // 创建引擎上下文
        engineContext = wglCreateContext(hdc);
        if (!engineContext) {
            printf("错误：创建引擎 GL 上下文失败\n");
            return false;
        }

        // 创建共享上下文（关键！）
        composeContext = wglCreateContext(hdc);
        if (!composeContext) {
            printf("错误：创建 Compose GL 上下文失败\n");
            return false;
        }

        // 让两个上下文共享资源
        // wglShareLists 必须在两个上下文都未被 MakeCurrent 之前调用
        // 或者至少在共享之前没有创建过任何 GL 对象
        BOOL result = wglShareLists(engineContext, composeContext);
        if (!result) {
            printf("错误：wglShareLists 失败，错误码 %lu\n", GetLastError());
            return false;
        }

        printf("Windows 共享上下文创建成功\n");
        return true;
    }

    void makeEngineCurrent() {
        wglMakeCurrent(hdc, engineContext);   // 切换到引擎上下文
    }

    void makeComposeCurrent() {
        wglMakeCurrent(hdc, composeContext);  // 切换到 Compose 上下文
    }

    void destroy() {
        wglDeleteContext(composeContext);
        wglDeleteContext(engineContext);
    }
};
#endif

// ============================
// Linux (X11 + GLX) 实现
// ============================
#ifdef __linux__
#include <X11/Xlib.h>
#include <GL/glx.h>

struct SharedGLContextLinux {
    Display*    display;
    GLXContext  engineContext;
    GLXContext  composeContext;
    GLXDrawable drawable;

    bool create(Display* dpy, Window win) {
        display = dpy;
        drawable = win;

        // 选择 visual（像素格式）
        int attribs[] = {
            GLX_RGBA,
            GLX_DEPTH_SIZE, 24,
            GLX_STENCIL_SIZE, 8,
            GLX_DOUBLEBUFFER,
            None
        };
        XVisualInfo* vi = glXChooseVisual(display, DefaultScreen(display), attribs);
        if (!vi) {
            printf("错误：GLX 无法找到合适的 Visual\n");
            return false;
        }

        // 创建引擎上下文
        engineContext = glXCreateContext(display, vi, NULL, True);

        // 创建共享上下文 — 第三个参数传入 engineContext 实现共享
        composeContext = glXCreateContext(display, vi, engineContext, True);

        XFree(vi);

        if (!engineContext || !composeContext) {
            printf("错误：创建 GLX 上下文失败\n");
            return false;
        }

        printf("Linux 共享上下文创建成功\n");
        return true;
    }

    void makeEngineCurrent() {
        glXMakeCurrent(display, drawable, engineContext);
    }

    void makeComposeCurrent() {
        glXMakeCurrent(display, drawable, composeContext);
    }

    void destroy() {
        glXDestroyContext(display, composeContext);
        glXDestroyContext(display, engineContext);
    }
};
#endif
```

macOS 和 Android 的共享上下文实现：

```cpp
// 示例 6 续：macOS 与 Android 共享上下文

// ============================
// macOS (CGL) 实现
// ============================
#ifdef __APPLE__
#include <OpenGL/OpenGL.h>   // CGL API
#include <OpenGL/gl3.h>

struct SharedGLContextMacOS {
    CGLContextObj engineContext;
    CGLContextObj composeContext;

    bool create() {
        // 像素格式属性
        CGLPixelFormatAttribute attribs[] = {
            kCGLPFAAccelerated,           // 硬件加速
            kCGLPFAOpenGLProfile,         // 指定 OpenGL 版本
            (CGLPixelFormatAttribute)kCGLOGLPVersion_GL4_Core,  // OpenGL 4.1 Core
            kCGLPFAColorSize, (CGLPixelFormatAttribute)24,
            kCGLPFADepthSize, (CGLPixelFormatAttribute)24,
            kCGLPFAStencilSize, (CGLPixelFormatAttribute)8,
            kCGLPFADoubleBuffer,
            (CGLPixelFormatAttribute)0    // 终止符
        };

        CGLPixelFormatObj pixelFormat;
        GLint numFormats;
        CGLChoosePixelFormat(attribs, &pixelFormat, &numFormats);

        // 创建引擎上下文（第二个参数 NULL = 不共享）
        CGLCreateContext(pixelFormat, NULL, &engineContext);

        // 创建共享上下文（第二个参数传入 engineContext）
        CGLCreateContext(pixelFormat, engineContext, &composeContext);

        CGLReleasePixelFormat(pixelFormat);

        printf("macOS 共享上下文创建成功\n");
        return (engineContext && composeContext);
    }

    void makeEngineCurrent() {
        CGLSetCurrentContext(engineContext);
    }

    void makeComposeCurrent() {
        CGLSetCurrentContext(composeContext);
    }

    void destroy() {
        CGLDestroyContext(composeContext);
        CGLDestroyContext(engineContext);
    }
};

// 注意：macOS 从 10.14 (Mojave) 开始废弃了 OpenGL，推荐使用 Metal。
// 但 OpenGL 4.1 仍然可用（截至 macOS 14 Sonoma）。
// 对于 KrKr2，如果需要长期支持 macOS，应考虑迁移到 Metal 后端。
#endif

// ============================
// Android (EGL) 实现
// ============================
#ifdef __ANDROID__
#include <EGL/egl.h>
#include <GLES3/gl3.h>
#include <android/log.h>

#define LOG_TAG "SharedGLContext"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)

struct SharedGLContextAndroid {
    EGLDisplay eglDisplay;
    EGLContext engineContext;
    EGLContext composeContext;
    EGLSurface eglSurface;
    EGLConfig  eglConfig;

    bool create(EGLNativeWindowType nativeWindow) {
        // 获取 EGL 显示连接
        eglDisplay = eglGetDisplay(EGL_DEFAULT_DISPLAY);
        eglInitialize(eglDisplay, nullptr, nullptr);

        // 配置属性
        EGLint configAttribs[] = {
            EGL_SURFACE_TYPE, EGL_WINDOW_BIT,
            EGL_RENDERABLE_TYPE, EGL_OPENGL_ES3_BIT,  // OpenGL ES 3.0
            EGL_RED_SIZE, 8,
            EGL_GREEN_SIZE, 8,
            EGL_BLUE_SIZE, 8,
            EGL_ALPHA_SIZE, 8,
            EGL_DEPTH_SIZE, 24,
            EGL_STENCIL_SIZE, 8,
            EGL_NONE
        };

        EGLint numConfigs;
        eglChooseConfig(eglDisplay, configAttribs, &eglConfig, 1, &numConfigs);

        // 创建窗口表面
        eglSurface = eglCreateWindowSurface(eglDisplay, eglConfig, nativeWindow, nullptr);

        // 上下文属性（OpenGL ES 3.0）
        EGLint contextAttribs[] = {
            EGL_CONTEXT_CLIENT_VERSION, 3,
            EGL_NONE
        };

        // 创建引擎上下文
        engineContext = eglCreateContext(eglDisplay, eglConfig, EGL_NO_CONTEXT, contextAttribs);

        // 创建共享上下文 — 第三个参数传入 engineContext
        composeContext = eglCreateContext(eglDisplay, eglConfig, engineContext, contextAttribs);

        if (engineContext == EGL_NO_CONTEXT || composeContext == EGL_NO_CONTEXT) {
            LOGE("创建 EGL 共享上下文失败: 0x%x", eglGetError());
            return false;
        }

        LOGI("Android 共享上下文创建成功");
        return true;
    }

    void makeEngineCurrent() {
        eglMakeCurrent(eglDisplay, eglSurface, eglSurface, engineContext);
    }

    void makeComposeCurrent() {
        eglMakeCurrent(eglDisplay, eglSurface, eglSurface, composeContext);
    }

    void destroy() {
        eglDestroyContext(eglDisplay, composeContext);
        eglDestroyContext(eglDisplay, engineContext);
        eglDestroySurface(eglDisplay, eglSurface);
        eglTerminate(eglDisplay);
    }
};
#endif
```

### 6.3 同步机制：glFenceSync

当 C++ 引擎和 Compose/Skia 使用共享上下文时，需要确保 C++ 引擎完成对纹理的渲染后，Compose/Skia 才能读取该纹理。OpenGL 提供了 **Fence Sync**（栅栏同步）机制：

```cpp
// C++ 引擎侧：渲染完毕后插入 fence
void engineRenderComplete() {
    // ... 引擎渲染到 FBO ...

    // 确保所有 GL 命令已提交到 GPU（但不等待完成）
    glFlush();

    // 创建 fence sync 对象 — GPU 到达此命令时会发出信号
    GLsync fence = glFenceSync(
        GL_SYNC_GPU_COMMANDS_COMPLETE,  // 条件：GPU 命令全部完成
        0                                // 标志位（目前必须为 0）
    );

    // 将 fence 传递给 Compose 侧（通过共享变量、队列等）
    sharedFence = fence;
}

// Compose/Skia 侧：等待 fence 后再读取纹理
void composeWaitForEngine() {
    GLsync fence = sharedFence;

    if (fence) {
        // 等待 fence — GPU 完成 C++ 引擎的渲染后才继续
        GLenum result = glClientWaitSync(
            fence,
            GL_SYNC_FLUSH_COMMANDS_BIT,  // 自动 flush
            5000000000ULL                 // 超时 5 秒（纳秒）
        );

        switch (result) {
            case GL_ALREADY_SIGNALED:     // fence 已经发出信号（最快路径）
            case GL_CONDITION_SATISFIED:  // 在超时前满足条件
                // 安全：可以读取纹理了
                break;
            case GL_TIMEOUT_EXPIRED:
                printf("警告：等待引擎渲染超时\n");
                break;
            case GL_WAIT_FAILED:
                printf("错误：glClientWaitSync 失败\n");
                break;
        }

        // 清理 fence 对象
        glDeleteSync(fence);
        sharedFence = nullptr;
    }
}
```

**为什么需要 Fence Sync？**

GPU 命令是异步执行的——`glDrawArrays()` 返回并不意味着 GPU 已经画完。如果 C++ 引擎刚提交绘制命令就让 Compose 去读取纹理，可能读到"画了一半"的内容（撕裂、花屏）。Fence Sync 确保 GPU 按顺序完成工作。

---

## 7. KrKr2 与 Compose Multiplatform 集成方案

### 7.1 整体架构设计

基于前面的技术分析，KrKr2 与 Compose Multiplatform 的集成可以设计为以下三层架构：

```
┌──────────────────────────────────────────────────────────┐
│                 Compose Multiplatform UI                  │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │ 菜单/对话框  │  │  设置面板     │  │  游戏画面视图  │  │
│  │ (Compose)   │  │  (Compose)   │  │  (GameView)    │  │
│  └─────────────┘  └──────────────┘  └───────┬────────┘  │
├──────────────────────────────────────────────┼───────────┤
│                 桥接层 (Bridge)               │           │
│  ┌──────────────┐  ┌──────────────────┐     │           │
│  │ EventBridge  │  │ TextureBridge    │◄────┘           │
│  │ (触摸/键盘)  │  │ (纹理传递)       │                  │
│  └──────┬───────┘  └────────┬─────────┘                  │
├─────────┼──────────────────┼────────────────────────────┤
│         │   Kotlin/Native  │  cinterop                   │
│         │   (JNI on JVM)   │                             │
├─────────┼──────────────────┼────────────────────────────┤
│         ▼                  ▼                             │
│  ┌──────────────────────────────────────────┐           │
│  │           KrKr2 C++ Engine               │           │
│  │  ┌────────────┐  ┌──────────────────┐   │           │
│  │  │ 游戏逻辑    │  │ Cocos2d-x 渲染   │   │           │
│  │  │ TJS2 脚本  │  │ (OpenGL ES)      │   │           │
│  │  └────────────┘  └──────────────────┘   │           │
│  └──────────────────────────────────────────┘           │
└──────────────────────────────────────────────────────────┘
```

### 7.2 渲染流程时序

每帧的渲染分为三个阶段，严格按顺序执行：

```
时间轴 ────────────────────────────────────────────►

阶段 1：C++ 引擎渲染（引擎线程）
├─ glBindFramebuffer(GL_FRAMEBUFFER, engineFBO)
├─ Cocos2d-x Scene::render()
│   ├─ 背景层绘制
│   ├─ 角色/立绘层绘制
│   └─ 特效层绘制
├─ glFlush()
├─ GLsync fence = glFenceSync(...)
└─ 通知 Compose 线程："本帧渲染完毕"

阶段 2：Compose 合成（UI 线程）
├─ glClientWaitSync(fence, ...)  // 等待引擎渲染完成
├─ 将 engineFBO 纹理包装为 Skia Image
├─ Compose 绘制阶段：
│   ├─ Canvas { drawImage(gameImage, ...) }  // 游戏画面
│   ├─ UI 按钮、对话框叠加绘制
│   └─ 动画效果
├─ Skia flush()
└─ SwapBuffers()  // 显示到屏幕

阶段 3：帧同步
├─ VSync 等待
└─ 开始下一帧
```

### 7.3 桥接层实现

桥接层（Bridge）是连接 C++ 引擎和 Compose UI 的中间件，包含纹理桥接和事件桥接两部分：

```cpp
// C++ 侧：纹理桥接接口
// 文件：src/bridge/TextureBridge.h

#pragma once
#include <GL/gl.h>
#include <cstdint>
#include <mutex>

// 双缓冲纹理 — 避免引擎写入和 Compose 读取同一纹理
struct DoubleBufferedTexture {
    GLuint textures[2];    // 两个纹理，交替使用
    GLuint fbos[2];        // 对应的 FBO
    int writeIndex = 0;    // 当前引擎写入的纹理索引
    int readIndex = 1;     // 当前 Compose 读取的纹理索引
    GLsync fences[2] = {nullptr, nullptr};  // 每个纹理的 fence
    int width = 0;
    int height = 0;
    std::mutex swapMutex;  // 交换锁

    // 初始化双缓冲纹理
    bool init(int w, int h) {
        width = w;
        height = h;
        glGenTextures(2, textures);
        glGenFramebuffers(2, fbos);

        for (int i = 0; i < 2; i++) {
            // 创建 RGBA8 纹理
            glBindTexture(GL_TEXTURE_2D, textures[i]);
            glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA8,
                         width, height, 0,
                         GL_RGBA, GL_UNSIGNED_BYTE, nullptr);
            glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
            glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

            // 将纹理附加到 FBO
            glBindFramebuffer(GL_FRAMEBUFFER, fbos[i]);
            glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,
                                   GL_TEXTURE_2D, textures[i], 0);

            // 检查 FBO 完整性
            GLenum status = glCheckFramebufferStatus(GL_FRAMEBUFFER);
            if (status != GL_FRAMEBUFFER_COMPLETE) {
                return false;  // FBO 创建失败
            }
        }

        glBindFramebuffer(GL_FRAMEBUFFER, 0);
        return true;
    }

    // 引擎开始渲染前调用 — 绑定写入 FBO
    void beginEngineRender() {
        glBindFramebuffer(GL_FRAMEBUFFER, fbos[writeIndex]);
        glViewport(0, 0, width, height);
    }

    // 引擎渲染完毕后调用 — 插入 fence 并交换缓冲
    void endEngineRender() {
        glFlush();
        // 如果旧 fence 还在，先清理
        if (fences[writeIndex]) {
            glDeleteSync(fences[writeIndex]);
        }
        fences[writeIndex] = glFenceSync(GL_SYNC_GPU_COMMANDS_COMPLETE, 0);

        // 交换读写索引
        std::lock_guard<std::mutex> lock(swapMutex);
        std::swap(writeIndex, readIndex);
    }

    // Compose 侧调用 — 获取可读取的纹理 ID
    GLuint getReadableTexture() {
        std::lock_guard<std::mutex> lock(swapMutex);
        int idx = readIndex;

        // 等待该纹理的渲染完成
        if (fences[idx]) {
            glClientWaitSync(fences[idx], GL_SYNC_FLUSH_COMMANDS_BIT, 5000000000ULL);
            glDeleteSync(fences[idx]);
            fences[idx] = nullptr;
        }

        return textures[idx];
    }

    void destroy() {
        for (int i = 0; i < 2; i++) {
            if (fences[i]) glDeleteSync(fences[i]);
        }
        glDeleteFramebuffers(2, fbos);
        glDeleteTextures(2, textures);
    }
};

// C API 供 Kotlin cinterop 调用
extern "C" {
    void* texture_bridge_create(int width, int height);
    void  texture_bridge_begin_render(void* bridge);
    void  texture_bridge_end_render(void* bridge);
    uint32_t texture_bridge_get_texture(void* bridge);
    void  texture_bridge_destroy(void* bridge);
}
```

### 7.4 Kotlin 侧集成

```kotlin
// Compose 侧：使用 TextureBridge 显示游戏画面
// 文件：src/commonMain/kotlin/GameView.kt

import androidx.compose.foundation.Canvas
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.drawscope.drawIntoCanvas
import androidx.compose.ui.graphics.nativeCanvas
import org.jetbrains.skia.*

@Composable
fun GameView(
    directContext: DirectContext,  // 从 SkiaLayer 获取
    gameWidth: Int = 1280,
    gameHeight: Int = 720
) {
    // 帧计数器 — 驱动每帧重绘
    var frameCount by remember { mutableLongStateOf(0L) }

    // 帧驱动循环
    LaunchedEffect(Unit) {
        while (true) {
            withFrameNanos {
                frameCount++
            }
        }
    }

    Canvas(modifier = Modifier.fillMaxSize()) {
        @Suppress("UNUSED_EXPRESSION")
        frameCount

        drawIntoCanvas { composeCanvas ->
            val skiaCanvas = composeCanvas.nativeCanvas

            // 1. 从 C++ 桥接层获取当前帧的纹理 ID
            //    texture_bridge_get_texture 内部会等待 fence
            val textureId = NativeBridge.getGameTexture()

            if (textureId > 0u) {
                // 2. 包装为 Skia Image
                val backendTexture = BackendTexture.makeGL(
                    width = gameWidth,
                    height = gameHeight,
                    mipmapped = false,
                    glTextureInfo = GlTextureInfo(
                        target = 0x0DE1,       // GL_TEXTURE_2D
                        id = textureId.toInt(),
                        format = 0x8058        // GL_RGBA8
                    )
                )

                val gameImage = Image.makeFromTexture(
                    context = directContext,
                    backendTexture = backendTexture,
                    origin = SurfaceOrigin.BOTTOM_LEFT,
                    colorType = ColorType.RGBA_8888,
                    alphaType = AlphaType.PREMUL,
                    colorSpace = ColorSpace.sRGB
                )

                // 3. 绘制游戏画面（缩放到 Canvas 大小）
                val scaleX = size.width / gameWidth
                val scaleY = size.height / gameHeight
                val scale = minOf(scaleX, scaleY)  // 保持宽高比

                skiaCanvas.save()
                skiaCanvas.scale(scale, scale)
                skiaCanvas.drawImage(gameImage, 0f, 0f)
                skiaCanvas.restore()

                // 4. 清理（不删除底层 GL 纹理）
                gameImage.close()
                backendTexture.close()
            }
        }
    }
}

// NativeBridge — 通过 cinterop 或 JNI 调用 C++ 函数
expect object NativeBridge {
    fun getGameTexture(): UInt
    fun sendTouchEvent(x: Float, y: Float, action: Int)
    fun sendKeyEvent(keyCode: Int, isDown: Boolean)
}
```

---

## 8. 跨平台渲染后端差异与适配

### 8.1 四平台渲染后端对比

Compose Multiplatform（通过 Skiko）在不同平台使用不同的默认渲染后端，这直接影响了与 C++ 引擎的集成策略：

| 特性 | Windows | Linux | macOS | Android |
|------|---------|-------|-------|---------|
| Skiko 默认后端 | Direct3D 12 | OpenGL | Metal | 系统 Skia（非 Skiko） |
| 可选后端 | OpenGL | Vulkan | OpenGL（已废弃） | Vulkan |
| KrKr2 引擎后端 | OpenGL ES (ANGLE) | OpenGL ES | OpenGL ES | OpenGL ES |
| 纹理共享难度 | **高**（需统一到 OpenGL 或用 D3D-GL 互操作） | **低**（双方都用 OpenGL） | **中**（Metal-GL 互操作） | **中**（EGL 共享上下文） |
| 推荐集成方案 | 强制 Skiko 使用 OpenGL 后端 | 直接共享 GL 上下文 | 使用 IOSurface 做纹理中转 | EGL 共享上下文 |

### 8.2 Windows：Direct3D 与 OpenGL 互操作

Windows 上 Skiko 默认使用 Direct3D 12，而 KrKr2 使用 OpenGL ES（通过 ANGLE 转译为 Direct3D）。这产生了一个有趣的局面：两边的 GPU 命令最终都走 Direct3D，但中间隔了一层翻译。

**方案 1：强制 Skiko 使用 OpenGL 后端**（推荐）

```kotlin
// 在应用启动前设置系统属性，强制 Skiko 使用 OpenGL
// 文件：src/main/kotlin/Main.kt

fun main() {
    // 方法 1：系统属性（JVM 参数）
    System.setProperty("skiko.renderApi", "OPENGL")

    // 方法 2：环境变量（启动前设置）
    // SKIKO_RENDER_API=OPENGL

    // 之后正常启动 Compose 应用
    singleWindowApplication(title = "KrKr2 + Compose") {
        App()
    }
}
```

强制 OpenGL 后端后，Skiko 在 Windows 上使用 WGL（Windows GL）创建 OpenGL 上下文，可以通过 `wglShareLists` 与 KrKr2 的 ANGLE OpenGL 上下文共享资源。

> **注意**：ANGLE 创建的是 EGL 上下文，不是原生 WGL 上下文。因此 `wglShareLists` 不能直接用于 ANGLE。需要使用 ANGLE 的 EGL 接口 `eglCreateContext(..., shareContext, ...)` 来创建共享上下文，或者让 KrKr2 也使用原生 OpenGL（不经过 ANGLE）。

**方案 2：通过 DXGI 共享纹理**（高级）

如果无法统一到 OpenGL，可以使用 Windows 的 DXGI（DirectX Graphics Infrastructure）共享纹理机制。C++ 引擎通过 ANGLE 的 D3D 后端渲染到一个 DXGI 共享纹理，Skiko 通过 Direct3D 12 读取该共享纹理。这个方案性能最高但实现最复杂，此处不展开。

### 8.3 Linux：最简单的共享方案

Linux 是四平台中纹理共享最简单的：Skiko 默认使用 OpenGL，KrKr2 也使用 OpenGL ES（Linux 上的 OpenGL 驱动兼容 ES 子集），双方可以直接创建共享 GLX 上下文。

```kotlin
// Linux 平台无需额外配置
// Skiko 默认使用 OpenGL，直接共享即可

// 验证当前渲染后端
fun checkRenderApi() {
    val renderApi = System.getProperty("skiko.renderApi")
    println("当前渲染后端: $renderApi")  // 应输出 "OPENGL"
}
```

### 8.4 macOS：Metal 与 OpenGL 桥接

macOS 上 Skiko 默认使用 Metal，但 KrKr2 使用 OpenGL。Apple 从 macOS 10.14 开始废弃 OpenGL（标记为 Deprecated），但截至 macOS 15 仍然可用。

**方案 1：强制 Skiko 使用 OpenGL**

```kotlin
// macOS 上强制 OpenGL 后端
System.setProperty("skiko.renderApi", "OPENGL")
// 注意：Apple 废弃了 OpenGL，编译时会有大量 deprecation 警告
// 但功能上仍然正常工作
```

**方案 2：使用 IOSurface 做纹理中转**（推荐长期方案）

IOSurface 是 macOS/iOS 提供的跨进程、跨 API 的纹理共享机制。C++ 引擎通过 OpenGL 渲染到 IOSurface 关联的纹理，Skia 通过 Metal 读取同一个 IOSurface：

```
C++ 引擎 (OpenGL)                    Compose/Skia (Metal)
       │                                     │
       ▼                                     ▼
OpenGL 纹理 ◄──── IOSurface ────► Metal 纹理
       │         (共享内存)          │
       ▼                              ▼
 渲染到 FBO                     MTLTexture 采样
```

```cpp
// macOS IOSurface 纹理共享示意
// 文件：src/platform/macos/IOSurfaceBridge.mm

#import <IOSurface/IOSurface.h>
#import <OpenGL/gl3.h>
#import <Metal/Metal.h>

// 创建 IOSurface
IOSurfaceRef createSharedSurface(int width, int height) {
    NSDictionary* properties = @{
        (id)kIOSurfaceWidth: @(width),
        (id)kIOSurfaceHeight: @(height),
        (id)kIOSurfaceBytesPerElement: @(4),         // RGBA
        (id)kIOSurfacePixelFormat: @('BGRA'),        // 像素格式
    };
    return IOSurfaceCreate((__bridge CFDictionaryRef)properties);
}

// OpenGL 侧：将 IOSurface 绑定为 GL 纹理
GLuint bindIOSurfaceToGL(IOSurfaceRef surface, int width, int height) {
    GLuint texture;
    glGenTextures(1, &texture);
    glBindTexture(GL_TEXTURE_RECTANGLE, texture);

    // CGLTexImageIOSurface2D 将 IOSurface 关联到 GL 纹理
    CGLContextObj cglContext = CGLGetCurrentContext();
    CGLTexImageIOSurface2D(
        cglContext,
        GL_TEXTURE_RECTANGLE,    // 目标（macOS 使用 RECTANGLE）
        GL_RGBA8,                // 内部格式
        width, height,
        GL_BGRA,                 // 像素格式
        GL_UNSIGNED_INT_8_8_8_8_REV,  // 数据类型
        surface,                 // IOSurface
        0                        // plane
    );

    return texture;
}

// Metal 侧：将 IOSurface 转为 MTLTexture
// （在 Kotlin 侧通过 cinterop 调用）
// id<MTLTexture> metalTexture =
//     [metalDevice newTextureWithDescriptor:desc
//                                iosurface:surface
//                                    plane:0];
```

### 8.5 Android：EGL 共享上下文

Android 上 Jetpack Compose 使用系统内置的 Skia（不经过 Skiko），渲染通过 `android.view.Surface` + `HardwareRenderer`。与 C++ 引擎共享纹理的最佳方式是 EGL 共享上下文 + `SurfaceTexture`：

```kotlin
// Android 平台：通过 SurfaceTexture 接收 C++ 引擎纹理
// 文件：src/androidMain/kotlin/GameTextureView.kt

import android.graphics.SurfaceTexture
import android.view.TextureView
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.viewinterop.AndroidView

@Composable
fun GameTextureView(
    modifier: Modifier = Modifier
) {
    // 使用 AndroidView 嵌入原生 TextureView
    AndroidView(
        modifier = modifier,
        factory = { context ->
            TextureView(context).apply {
                surfaceTextureListener = object : TextureView.SurfaceTextureListener {
                    override fun onSurfaceTextureAvailable(
                        surfaceTexture: SurfaceTexture, width: Int, height: Int
                    ) {
                        // 将 SurfaceTexture 传递给 C++ 引擎
                        // C++ 侧通过 EGL 在该 Surface 上创建上下文并渲染
                        NativeBridge.setSurfaceTexture(surfaceTexture)
                        NativeBridge.startEngine(width, height)
                    }

                    override fun onSurfaceTextureSizeChanged(
                        surface: SurfaceTexture, width: Int, height: Int
                    ) {
                        NativeBridge.onResize(width, height)
                    }

                    override fun onSurfaceTextureDestroyed(surface: SurfaceTexture): Boolean {
                        NativeBridge.stopEngine()
                        return true  // 返回 true 表示我们会释放 SurfaceTexture
                    }

                    override fun onSurfaceTextureUpdated(surface: SurfaceTexture) {
                        // 每次 C++ 引擎有新帧时回调
                    }
                }
            }
        }
    )
}
```

---

## 9. 常见错误与排查

### 错误 1：纹理显示为黑色或全白

**症状**：`Image.makeFromTexture()` 成功创建，但 `drawImage` 绘制出来是纯黑或纯白。

**可能原因**：

1. **未等待 fence**：C++ 引擎还没渲染完，Compose 就去读取了空纹理
2. **纹理格式不匹配**：`GlTextureInfo` 中的 `format` 与实际 `glTexImage2D` 的内部格式不一致
3. **上下文未共享**：两个 OpenGL 上下文没有正确建立共享关系，纹理 ID 在另一个上下文中无效
4. **SurfaceOrigin 错误**：选了 `TOP_LEFT` 但实际是 `BOTTOM_LEFT`，导致坐标映射到纹理外

**排查步骤**：

```cpp
// 1. 验证纹理内容（C++ 侧）
void debugTexture(GLuint textureId, int width, int height) {
    // 创建临时 FBO 绑定该纹理
    GLuint debugFBO;
    glGenFramebuffers(1, &debugFBO);
    glBindFramebuffer(GL_READ_FRAMEBUFFER, debugFBO);
    glFramebufferTexture2D(GL_READ_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,
                           GL_TEXTURE_2D, textureId, 0);

    // 读回像素（检查是否有内容）
    std::vector<uint8_t> pixels(width * height * 4);
    glReadPixels(0, 0, width, height, GL_RGBA, GL_UNSIGNED_BYTE, pixels.data());

    // 检查第一个像素
    printf("像素[0] RGBA = (%d, %d, %d, %d)\n",
           pixels[0], pixels[1], pixels[2], pixels[3]);

    // 统计非零像素
    int nonZero = 0;
    for (size_t i = 0; i < pixels.size(); i += 4) {
        if (pixels[i] || pixels[i+1] || pixels[i+2] || pixels[i+3]) {
            nonZero++;
        }
    }
    printf("非零像素: %d / %d\n", nonZero, width * height);

    glDeleteFramebuffers(1, &debugFBO);
}
```

### 错误 2：画面撕裂或闪烁

**症状**：游戏画面出现横向撕裂线，或某些帧闪烁为旧内容。

**原因**：C++ 引擎和 Compose 线程在争抢同一个纹理——引擎正在写入时 Compose 同时读取。

**解决方案**：使用双缓冲纹理（参见 7.3 节的 `DoubleBufferedTexture`）+ Fence Sync。确保：

1. 引擎写入纹理 A 时，Compose 读取纹理 B
2. 引擎写完后交换 A 和 B 的角色
3. 每次交换前插入 Fence，Compose 读取前等待 Fence

### 错误 3：macOS 上 `CGLTexImageIOSurface2D` 返回错误

**症状**：`CGLTexImageIOSurface2D` 返回 `kCGLBadDrawable` 或 `kCGLBadValue`。

**常见原因**：

1. IOSurface 的像素格式不匹配：`kIOSurfacePixelFormat` 必须是 `'BGRA'`（注意是 BGRA 不是 RGBA）
2. OpenGL 纹理目标必须是 `GL_TEXTURE_RECTANGLE`，不能用 `GL_TEXTURE_2D`
3. 纹理的宽高必须与 IOSurface 一致

```objc
// 正确设置（注意 BGRA 和 RECTANGLE）
glBindTexture(GL_TEXTURE_RECTANGLE, texture);  // 必须是 RECTANGLE
CGLTexImageIOSurface2D(ctx, GL_TEXTURE_RECTANGLE,
    GL_RGBA8, w, h,
    GL_BGRA,                              // 必须 BGRA
    GL_UNSIGNED_INT_8_8_8_8_REV,          // macOS 特有的字节序
    ioSurface, 0);
```

---

## 10. Compose Multiplatform vs Flutter：Skia 共享对比

在 P12 的第 2 章中我们学习了 Flutter 的纹理共享方案（External Texture）。这里将 Compose Multiplatform 的方案与 Flutter 做一个直接对比：

| 对比维度 | Flutter | Compose Multiplatform |
|----------|---------|----------------------|
| **Skia 使用方式** | Flutter 完全控制 Skia，用户无法直接访问 | 通过 Skiko 暴露完整 Skia API |
| **纹理注册机制** | `FlutterEngineRegisterExternalTexture` → Texture Widget | 直接使用 `Image.makeFromTexture` |
| **回调模式** | 引擎主动回调 `gl_external_texture_frame_callback` | 应用侧主动读取纹理（pull 模式） |
| **同步机制** | Flutter 引擎内部处理 | 开发者需自行管理 Fence Sync |
| **平台适配** | Flutter 引擎统一抹平 | 开发者需处理各平台后端差异 |
| **C++ 互操作** | Platform Channel（消息传递） | cinterop / JNI（直接调用） |
| **灵活性** | 较低（受限于 Flutter 引擎 API） | 较高（可直接操作 Skia 和 GL） |
| **实现复杂度** | 中等（API 清晰但文档少） | 较高（需要深入了解 Skia 和 GL） |

**KrKr2 项目的选择建议**：

- **如果团队熟悉 Dart/Flutter**：选 Flutter，因为 External Texture 机制已经封装好了大部分跨平台差异
- **如果团队熟悉 Kotlin/C++**：选 Compose Multiplatform，因为 cinterop 提供更直接的 C++ 互操作，且 Skia API 完全开放
- **如果需要最大灵活性**：选 Compose Multiplatform，可以直接操作底层 Skia/GL 对象
- **如果追求最快开发速度**：选 Flutter，生态更成熟，文档更丰富

---

## 动手实践

### 练习 1：创建 Skia 自定义绘制组件

按以下步骤创建一个 Compose Desktop 项目，使用原生 Skia API 绘制动画图形：

1. 使用 IntelliJ IDEA 创建 Compose Multiplatform 桌面项目
2. 在 `build.gradle.kts` 中确认依赖 `compose.desktop.currentOs`
3. 创建一个 `@Composable` 函数，使用 `Canvas` 组件 + `drawIntoCanvas` + `nativeCanvas`
4. 使用 Skia 的 `Shader.makeLinearGradient()` 创建渐变填充
5. 使用 `withFrameNanos` 实现颜色随时间变化的动画效果
6. 运行并验证渐变动画是否流畅（应达到 60fps）

### 练习 2：模拟纹理共享流程

创建一个双 Canvas 布局，模拟 C++ 引擎与 Compose UI 的纹理共享：

1. Canvas A（"引擎侧"）：使用 Skia 渲染一个旋转的彩色方块（模拟游戏画面）
2. 将 Canvas A 的渲染结果捕获为 `Image`（使用 `Surface.makeRasterN32Premul` + `makeImageSnapshot`）
3. Canvas B（"UI 侧"）：显示捕获的 Image，并在上方叠加 Compose UI 元素（按钮、文字）
4. 验证两个 Canvas 的内容是否正确合成

---

## 对照项目源码

虽然 KrKr2 目前尚未集成 Compose Multiplatform，但以下现有源码文件与本节内容直接相关：

- `cpp/core/environ/cocos2d/AppDelegate.cpp` — KrKr2 当前的渲染入口，理解现有渲染循环是集成的前提
- `cpp/core/visual/ogl/` — OpenGL 渲染相关代码，了解当前使用的 GL 调用模式
- `cpp/core/environ/ui/` — 当前 Cocos2d 实现的 UI 组件，是 Compose 要替换的目标
- `platforms/android/cpp/krkr2_android.cpp` — Android JNI 桥接，可参考其 JNI 模式设计 Compose 的 JNI 桥接
- `ui/cocos-studio/` — 当前 UI 布局文件，需要在 Compose 中重新实现

**关键挑战**：KrKr2 使用 Cocos2d-x 管理窗口和渲染循环。要集成 Compose Multiplatform，需要决定"谁拥有窗口和渲染循环"——是 Cocos2d-x 创建窗口然后嵌入 Compose，还是 Compose 创建窗口然后嵌入 Cocos2d-x 的渲染。后者（Compose 为主）更符合"用现代 UI 替换旧 UI"的目标。

---

## 本节小结

1. **Skia** 是 Google 的开源 2D 渲染引擎，被 Chrome、Android、Flutter 和 Compose Multiplatform 共同使用
2. **Skiko**（Skia for Kotlin）是 JetBrains 的 Skia 绑定，在桌面端自带 Skia 二进制库，Android 端使用系统 Skia
3. Compose Multiplatform 的渲染管线分为组合→布局→绘制→渲染四个阶段，自定义渲染介入点在绘制阶段
4. 通过 `drawIntoCanvas` + `nativeCanvas` 可以在 Compose 中直接访问 Skia 原生 API
5. C++ 引擎与 Compose 共享渲染有三种方案：**纹理共享**（推荐）、像素拷贝、分层窗口
6. 纹理共享需要**共享 GL 上下文** + **Fence Sync** + **双缓冲**确保正确性和性能
7. 四平台的渲染后端各不相同（Windows: D3D12, Linux: OpenGL, macOS: Metal, Android: GL ES），需要分别适配
8. 与 Flutter 相比，Compose Multiplatform 提供更底层的 Skia 访问能力，但需要开发者自行处理更多平台差异
9. KrKr2 集成 Compose Multiplatform 的推荐架构是"Compose 为主、引擎渲染到共享纹理、桥接层传递纹理和事件"

---

## 练习题与答案

### 题目 1：Skia 渲染后端选择分析

问题：假设 KrKr2 需要在 Windows 上同时运行 Cocos2d-x（OpenGL ES via ANGLE）和 Compose Multiplatform UI。请分析以下三种方案的优缺点，并推荐最佳方案：

A. Skiko 使用 Direct3D 12（默认），通过 DXGI 共享纹理
B. Skiko 强制 OpenGL，通过共享 GL 上下文
C. 使用像素拷贝（glReadPixels + Image.makeRaster）

<details>
<summary>查看答案</summary>

**方案 A（D3D12 + DXGI 共享）**：
- 优点：性能最高（D3D12 是 Windows 原生 API，无翻译开销）；DXGI 共享纹理效率高
- 缺点：实现极其复杂；ANGLE 的 D3D 后端版本可能与 Skiko 的 D3D12 不兼容（ANGLE 默认用 D3D11）；跨 API 共享的调试困难
- 适用：团队有丰富的 DirectX 开发经验

**方案 B（OpenGL 统一）**：
- 优点：两侧都用 OpenGL，可以直接共享纹理；实现相对简单；参考资料多
- 缺点：Windows 上 Skiko 的 OpenGL 后端性能略低于 D3D12；ANGLE 创建 EGL 上下文与 Skiko 创建 WGL 上下文的共享需要额外适配；Apple 已废弃 OpenGL 影响长期维护
- 适用：快速验证原型、团队熟悉 OpenGL

**方案 C（像素拷贝）**：
- 优点：零耦合，不需要任何共享上下文；最容易实现
- 缺点：性能极差（1080p 下 glReadPixels 耗时 8-12ms，接近一帧预算）；CPU-GPU 数据往返导致延迟
- 适用：仅限原型验证或截图功能

**推荐方案 B**。理由：
1. OpenGL 是 KrKr2 当前唯一支持的渲染 API，统一到 OpenGL 改动最小
2. 共享 GL 上下文是一项成熟技术，参考实现多
3. 可以通过设置 `skiko.renderApi=OPENGL` 一行代码切换 Skiko 后端
4. 如果未来 KrKr2 迁移到 Vulkan/Metal，再考虑方案 A

</details>

### 题目 2：双缓冲纹理同步

问题：以下代码中存在一个竞态条件（Race Condition），请找出问题并修复：

```cpp
struct TextureBridge {
    GLuint textures[2];
    int writeIdx = 0, readIdx = 1;
    GLsync fence = nullptr;

    void engineFinish() {
        glFlush();
        fence = glFenceSync(GL_SYNC_GPU_COMMANDS_COMPLETE, 0);
        // 交换读写索引
        std::swap(writeIdx, readIdx);
    }

    GLuint composeGetTexture() {
        if (fence) {
            glClientWaitSync(fence, GL_SYNC_FLUSH_COMMANDS_BIT, 5000000000ULL);
            glDeleteSync(fence);
            fence = nullptr;
        }
        return textures[readIdx];
    }
};
```

<details>
<summary>查看答案</summary>

**问题**：`engineFinish()` 和 `composeGetTexture()` 运行在不同线程，对 `writeIdx`、`readIdx`、`fence` 的访问没有同步保护。具体竞态场景：

1. Compose 线程正在执行 `glClientWaitSync(fence, ...)`
2. 引擎线程同时调用 `engineFinish()`，覆盖了 `fence`（新的 fence 覆盖旧的）
3. Compose 线程执行 `glDeleteSync(fence)`，删除的是**新** fence 而非旧的
4. 结果：新帧的 fence 被误删，Compose 可能读到未完成的纹理

**修复方案**：使用 mutex 保护共享状态

```cpp
struct TextureBridge {
    GLuint textures[2];
    int writeIdx = 0, readIdx = 1;
    GLsync fences[2] = {nullptr, nullptr};  // 每个纹理独立 fence
    std::mutex mu;

    void engineFinish() {
        glFlush();

        std::lock_guard<std::mutex> lock(mu);
        // 为当前写入的纹理设置 fence
        if (fences[writeIdx]) {
            glDeleteSync(fences[writeIdx]);
        }
        fences[writeIdx] = glFenceSync(GL_SYNC_GPU_COMMANDS_COMPLETE, 0);
        // 交换索引
        std::swap(writeIdx, readIdx);
    }

    GLuint composeGetTexture() {
        std::lock_guard<std::mutex> lock(mu);
        int idx = readIdx;
        // 等待该纹理的渲染完成
        if (fences[idx]) {
            glClientWaitSync(fences[idx], GL_SYNC_FLUSH_COMMANDS_BIT, 5000000000ULL);
            glDeleteSync(fences[idx]);
            fences[idx] = nullptr;
        }
        return textures[idx];
    }
};
```

关键改进：
1. 每个纹理使用独立的 fence（`fences[2]`），避免 fence 覆盖问题
2. 使用 `std::mutex` 保护索引交换和 fence 访问
3. `composeGetTexture` 先拷贝 `readIdx` 的值再操作，防止读取过程中被交换

</details>

### 题目 3：设计 KrKr2 的 Compose 集成桥接层

问题：请为 KrKr2 设计一个完整的 C++ 桥接层接口（头文件），满足以下需求：

1. 初始化/销毁游戏引擎
2. 传递窗口大小变化事件
3. 传递触摸/鼠标事件给游戏引擎
4. 获取游戏渲染帧的纹理 ID
5. 需要支持四个平台

<details>
<summary>查看答案</summary>

```cpp
// 文件：src/bridge/krkr2_compose_bridge.h
#pragma once

#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

// ============================
// 引擎生命周期
// ============================

/**
 * 初始化 KrKr2 引擎
 * @param width  渲染宽度（像素）
 * @param height 渲染高度（像素）
 * @param shared_gl_context 平台特定的共享 GL 上下文指针（可选）
 *        - Windows: HGLRC
 *        - Linux:   GLXContext
 *        - macOS:   CGLContextObj
 *        - Android: EGLContext
 *        传入 NULL 则引擎自行创建上下文
 * @return 0 成功，非 0 错误码
 */
int krkr2_init(int width, int height, void* shared_gl_context);

/**
 * 销毁引擎并释放所有资源
 */
void krkr2_destroy(void);

/**
 * 通知引擎窗口大小变化
 */
void krkr2_resize(int width, int height);

// ============================
// 渲染
// ============================

/**
 * 请求引擎渲染一帧
 * 引擎会渲染到内部 FBO，调用方通过 krkr2_get_texture 获取结果
 * @param timestamp_ns 当前帧的时间戳（纳秒）
 */
void krkr2_render_frame(int64_t timestamp_ns);

/**
 * 获取当前可读取的游戏纹理
 * @param out_texture_id [输出] OpenGL 纹理 ID
 * @param out_width      [输出] 纹理宽度
 * @param out_height     [输出] 纹理高度
 * @return 0 成功（纹理可安全读取），非 0 表示无可用帧
 */
int krkr2_get_texture(uint32_t* out_texture_id, int* out_width, int* out_height);

// ============================
// 输入事件
// ============================

// 触摸/鼠标动作枚举
enum KrKr2TouchAction {
    KRKR2_TOUCH_DOWN   = 0,  // 按下
    KRKR2_TOUCH_UP     = 1,  // 抬起
    KRKR2_TOUCH_MOVE   = 2,  // 移动
    KRKR2_TOUCH_CANCEL = 3   // 取消
};

/**
 * 发送触摸/鼠标事件
 * @param x       X 坐标（相对于游戏画面，像素）
 * @param y       Y 坐标
 * @param action  触摸动作
 * @param pointer_id 触摸点 ID（多点触控时区分不同手指）
 */
void krkr2_send_touch(float x, float y, int action, int pointer_id);

/**
 * 发送键盘事件
 * @param key_code   平台无关的按键码（映射自 TJS2 VK_* 常量）
 * @param is_down    true = 按下，false = 抬起
 * @param modifiers  修饰键位掩码（Shift=1, Ctrl=2, Alt=4）
 */
void krkr2_send_key(int key_code, int is_down, int modifiers);

/**
 * 发送文本输入（用于输入法/IME）
 * @param text UTF-8 编码的文本
 */
void krkr2_send_text(const char* text);

// ============================
// 音频（可选，引擎通常自行管理）
// ============================

/**
 * 暂停/恢复音频播放（应用切到后台时调用）
 */
void krkr2_audio_pause(void);
void krkr2_audio_resume(void);

#ifdef __cplusplus
}
#endif
```

设计要点说明：
1. 所有函数使用 `extern "C"` 确保 C 链接，方便 Kotlin cinterop 绑定
2. 使用不透明指针 `void*` 传递平台特定对象，保持接口统一
3. `krkr2_get_texture` 使用输出参数而非返回值，因为需要同时返回多个信息
4. 触摸事件设计支持多点触控（pointer_id），键盘事件使用平台无关的按键码
5. 文本输入单独一个接口，因为 IME 输入可能是多字符

</details>

---

## 下一步

[下一节：04-UI 与渲染引擎分离 / 01-抽象 UI 接口层设计](../04-UI与渲染引擎分离/01-抽象UI接口层设计.md) — 学习如何设计一个抽象 UI 接口层，使 KrKr2 的 UI 能够在不同框架（Cocos2d/Flutter/Compose）之间自由切换。

</content>
</invoke>