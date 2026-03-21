# Kotlin/Native 与 Cinterop 基础

> **所属模块：** P12-现代跨平台UI
> **前置知识：** [01-方案对比/01-主流跨平台UI框架总览](../01-方案对比/01-主流跨平台UI框架总览.md)、[01-方案对比/02-渲染方式与C++互操作能力对比](../01-方案对比/02-渲染方式与C++互操作能力对比.md)
> **预计阅读时间：** 45 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 Kotlin/Native 的编译模型和内存管理机制，知道它与 JVM Kotlin 的核心区别
2. 使用 cinterop 工具从 C/C++ 头文件生成 Kotlin 绑定，实现 Kotlin 代码直接调用 C 函数
3. 编写完整的 Kotlin/Native 项目，通过 Gradle 配置 cinterop 并链接 C/C++ 静态库或动态库
4. 理解 Kotlin/Native 的内存模型（新版 GC）及其对 C 互操作的影响
5. 为 KrKr2 的 Compose Multiplatform 集成方案打下 C++ 互操作基础

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| Kotlin/Native | Kotlin/Native | Kotlin 的原生编译目标，通过 LLVM 将 Kotlin 代码编译为机器码，无需 JVM |
| cinterop | C Interop Tool | Kotlin/Native 提供的 C 互操作工具，从 C 头文件自动生成 Kotlin 绑定代码 |
| def 文件 | Definition File | cinterop 的配置文件，指定要绑定的 C 头文件、编译选项和链接库 |
| KLib | Kotlin Library | Kotlin/Native 的库格式（类似 JAR），包含编译后的 LLVM IR 和元数据 |
| CPointer | C Pointer | Kotlin/Native 中表示 C 指针的类型，是 C 互操作的核心数据类型 |
| pinned | Pinned Object | 将 Kotlin 对象"钉"在内存中不被 GC 移动，以便安全地将指针传递给 C 代码 |
| MemScope | Memory Scope | Kotlin/Native 的作用域化内存分配器，自动管理 C 互操作中分配的原生内存 |
| StableRef | Stable Reference | 让 C 代码持有 Kotlin 对象引用的机制，防止对象被 GC 回收 |
| LLVM IR | LLVM Intermediate Representation | LLVM 编译器的中间表示，Kotlin/Native 编译器先生成 IR 再由 LLVM 后端生成机器码 |

---

## Kotlin/Native 编译模型

### 什么是 Kotlin/Native？

Kotlin/Native 是 JetBrains 开发的 Kotlin 编译器后端，它将 Kotlin 源代码编译为**原生机器码**（而非 JVM 字节码）。编译流程如下：

```
Kotlin 源码 (.kt)
       │
       ▼
Kotlin/Native 编译器前端
(语法解析、类型检查、IR 生成)
       │
       ▼
Kotlin IR (中间表示)
       │
       ▼
LLVM IR 生成器
(将 Kotlin IR 转换为 LLVM IR)
       │
       ▼
LLVM 后端
(优化 + 机器码生成)
       │
       ▼
原生可执行文件 / 动态库 / 静态库
(.exe / .so / .dylib / .a / .framework)
```

与 JVM Kotlin 的关键区别：

| 特性 | JVM Kotlin | Kotlin/Native |
|------|-----------|---------------|
| **运行环境** | 需要 JVM（Java 虚拟机） | 直接运行在操作系统上，无需虚拟机 |
| **编译产物** | .class / .jar 字节码 | 原生机器码（.exe / .so / .dylib） |
| **垃圾回收** | JVM GC（G1/ZGC 等成熟方案） | Kotlin/Native 自有 GC（基于追踪式 GC，2022 年新版） |
| **反射** | 完整支持 `kotlin-reflect` | 有限支持（性能考虑） |
| **C 互操作** | 需要 JNI（Java Native Interface） | **原生支持 cinterop**，无中间层 |
| **启动速度** | 慢（JVM 预热） | 快（直接执行机器码） |
| **内存占用** | 较大（JVM 开销） | 较小（无虚拟机开销） |
| **目标平台** | 任何有 JVM 的平台 | macOS/iOS/Linux/Windows/Android NDK 等 |

> **为什么 KrKr2 关心 Kotlin/Native？** 因为 Compose Multiplatform（Kotlin 版本的声明式 UI 框架）在非 Android 平台上使用 Kotlin/Native 编译。如果我们要用 Compose Multiplatform 替换 KrKr2 的 UI 层，就需要通过 Kotlin/Native 的 cinterop 来调用 C++ 引擎代码。

### Kotlin/Native 的内存管理

Kotlin/Native 在 2022 年经历了一次重大的内存模型改革。新版内存管理器（New Memory Manager）使用了**追踪式垃圾回收（Tracing GC）**，与旧版的"冻结+引用计数"模型完全不同：

**新版 GC（默认，Kotlin 1.7.20+）**：
- 支持**跨线程共享对象**，无需手动冻结
- 使用标记-清除（Mark and Sweep）算法的变体
- 支持并发标记（Concurrent Marking），减少 GC 暂停时间
- 与 C 互操作时，需要注意 GC 可能**移动对象**——传递给 C 代码的指针必须"钉住"（pin）

```kotlin
// 新版内存模型示例：跨线程共享可变状态
import kotlin.concurrent.AtomicInt

// 原子整数——多线程安全的计数器
val counter = AtomicInt(0)

fun incrementFromAnyThread() {
    counter.incrementAndGet()  // 任意线程都可以安全调用
    println("计数器: ${counter.value}")
}
```

**GC 对 C 互操作的影响**：

当你将 Kotlin 对象的指针传递给 C 代码时，GC 可能在 C 代码使用该指针期间移动对象，导致悬空指针（dangling pointer）。解决方案：

```kotlin
import kotlinx.cinterop.*

fun passArrayToC(data: ByteArray) {
    // pinned() 将数组"钉"在内存中，GC 不会移动它
    data.usePinned { pinned ->
        // pinned.addressOf(0) 返回稳定的 C 指针
        val cPointer: CPointer<ByteVar> = pinned.addressOf(0)
        // 现在可以安全地将 cPointer 传递给 C 函数
        c_process_data(cPointer, data.size)
    }
    // usePinned 块结束后自动解除钉住，GC 恢复正常
}
```

---

## cinterop 工具详解

### cinterop 是什么？

cinterop（C Interop Tool）是 Kotlin/Native 工具链的核心组件，它的作用是：**读取 C 语言的头文件（.h），自动生成对应的 Kotlin 绑定代码**。生成的绑定代码让 Kotlin 程序可以像调用本地函数一样调用 C 函数，无需手写 JNI 或 FFI 代码。

```
C 头文件 (.h)                    cinterop 工具
─────────────                    ─────────────
typedef struct {         ──►     class Point(rawPtr: NativePtr) :
    float x;                         CStructVar() {
    float y;                         var x: Float
} Point;                             var y: Float
                                 }
void draw(Point* p);     ──►     fun draw(p: CValuesRef<Point>?)
int get_count();          ──►     fun get_count(): Int
```

### def 文件语法

cinterop 通过 `.def` 文件（定义文件）来配置绑定参数。def 文件使用简单的键值对语法：

```
# game_engine.def — KrKr2 引擎 C 接口的 cinterop 定义文件

# 包名：生成的 Kotlin 绑定代码的包名
package = com.krkr2.native

# 要解析的头文件列表（相对于 def 文件或 compilerOpts 中的 include 路径）
headers = krkr2_api.h

# C 编译器选项（传给 clang 预处理器）
# -I 指定头文件搜索路径
compilerOpts = -I/usr/local/include -I../cpp/core/environ

# 链接器选项
# -L 指定库搜索路径，-l 指定要链接的库
linkerOpts = -L/usr/local/lib -lkrkr2core

# 头文件过滤器：只为匹配的头文件生成绑定（减少生成代码量）
headerFilter = krkr2_api.h

# 排除的函数（不生成绑定）
excludedFunctions = internal_debug_func

# 排除的宏
excludedMacros = DEBUG_ONLY_MACRO

# 静态库路径（直接链接，不需要 -l）
staticLibraries = libkrkr2core.a

# 库搜索路径（针对 staticLibraries）
libraryPaths = ../build/lib
```

### 示例 1：最简单的 cinterop 绑定

让我们从一个最简单的例子开始——将一个 C 数学库绑定到 Kotlin/Native：

**C 头文件**：

```c
// mathlib.h — 一个简单的 C 数学库
#ifndef MATHLIB_H
#define MATHLIB_H

#ifdef __cplusplus
extern "C" {
#endif

// 二维向量结构体
typedef struct {
    float x;  // X 坐标
    float y;  // Y 坐标
} Vec2;

// 向量加法
Vec2 vec2_add(Vec2 a, Vec2 b);

// 向量点积（内积）—— 两个向量对应分量相乘再求和
float vec2_dot(Vec2 a, Vec2 b);

// 向量长度（模）—— sqrt(x² + y²)
float vec2_length(Vec2 v);

// 向量归一化 —— 返回同方向的单位向量（长度为 1）
Vec2 vec2_normalize(Vec2 v);

// 计算两点间距离
float vec2_distance(Vec2 a, Vec2 b);

// 版本信息
const char* mathlib_version(void);

#ifdef __cplusplus
}
#endif

#endif // MATHLIB_H
```

**C 实现文件**：

```c
// mathlib.c — 实现
#include "mathlib.h"
#include <math.h>

Vec2 vec2_add(Vec2 a, Vec2 b) {
    return (Vec2){a.x + b.x, a.y + b.y};
}

float vec2_dot(Vec2 a, Vec2 b) {
    return a.x * b.x + a.y * b.y;
}

float vec2_length(Vec2 v) {
    return sqrtf(v.x * v.x + v.y * v.y);
}

Vec2 vec2_normalize(Vec2 v) {
    float len = vec2_length(v);
    if (len < 1e-6f) return (Vec2){0.0f, 0.0f};
    return (Vec2){v.x / len, v.y / len};
}

float vec2_distance(Vec2 a, Vec2 b) {
    Vec2 diff = {a.x - b.x, a.y - b.y};
    return vec2_length(diff);
}

const char* mathlib_version(void) {
    return "mathlib 1.0.0";
}
```

**def 文件**：

```
# mathlib.def
package = com.example.mathlib
headers = mathlib.h
compilerOpts = -I./native/include
staticLibraries = libmathlib.a
libraryPaths = ./native/lib
```

**Kotlin/Native 调用代码**：

```kotlin
// src/nativeMain/kotlin/Main.kt
// 使用 cinterop 生成的绑定调用 C 数学库
import com.example.mathlib.*  // cinterop 生成的包
import kotlinx.cinterop.*

fun main() {
    // 1. 打印库版本
    // mathlib_version() 返回 CPointer<ByteVar>，用 toKString() 转为 Kotlin String
    val version = mathlib_version()?.toKString() ?: "未知"
    println("数学库版本: $version")

    // 2. 创建向量并运算
    // cinterop 将 C 的 Vec2 struct 映射为 Kotlin 的 CValue<Vec2>
    // 使用 cValue 块来创建 C 结构体
    val a = cValue<Vec2> {
        x = 3.0f
        y = 4.0f
    }
    val b = cValue<Vec2> {
        x = 1.0f
        y = 2.0f
    }

    // 3. 向量加法
    val sum = vec2_add(a, b)
    // sum 是 CValue<Vec2>，用 useContents 访问其字段
    sum.useContents {
        println("a + b = ($x, $y)")  // 输出: a + b = (4.0, 6.0)
    }

    // 4. 点积
    val dot = vec2_dot(a, b)
    println("a · b = $dot")  // 输出: a · b = 11.0

    // 5. 向量长度
    val lenA = vec2_length(a)
    println("|a| = $lenA")  // 输出: |a| = 5.0

    // 6. 归一化
    val normalized = vec2_normalize(a)
    normalized.useContents {
        println("â = ($x, $y)")  // 输出: â = (0.6, 0.8)
    }

    // 7. 两点距离
    val dist = vec2_distance(a, b)
    println("d(a, b) = $dist")  // 输出: d(a, b) = 2.828...
}
```

> **注意 `cValue` 和 `useContents`**：C 结构体在 Kotlin/Native 中不能像普通 Kotlin 对象那样直接创建和访问。`cValue<T> { ... }` 在栈上分配 C 结构体并初始化字段；`useContents { ... }` 临时解引用 C 值以访问字段。这是因为 C 结构体的内存布局由 C ABI 决定，不受 Kotlin GC 管理。

---

## Gradle 配置 cinterop

### 示例 2：完整的 Gradle 项目配置

在实际项目中，cinterop 通过 Gradle 的 Kotlin Multiplatform 插件自动运行。以下是完整的项目配置：

**项目结构**：

```
kotlin-native-mathlib/
├── build.gradle.kts           # Gradle 构建脚本
├── settings.gradle.kts        # 项目设置
├── gradle.properties          # Gradle 属性
├── native/
│   ├── include/
│   │   └── mathlib.h          # C 头文件
│   ├── lib/
│   │   ├── linux-x64/
│   │   │   └── libmathlib.a   # Linux x64 静态库
│   │   ├── macos-arm64/
│   │   │   └── libmathlib.a   # macOS arm64 静态库
│   │   └── windows-x64/
│   │       └── mathlib.lib    # Windows x64 静态库
│   └── mathlib.def            # cinterop 定义文件
└── src/
    └── nativeMain/
        └── kotlin/
            └── Main.kt        # Kotlin 主程序
```

**build.gradle.kts**：

```kotlin
// build.gradle.kts — Kotlin/Native 项目构建配置
plugins {
    kotlin("multiplatform") version "2.1.0"
}

group = "com.example"
version = "1.0.0"

repositories {
    mavenCentral()
}

kotlin {
    // 定义编译目标平台
    // 根据当前操作系统选择目标
    val hostOs = System.getProperty("os.name")
    val nativeTarget = when {
        hostOs == "Mac OS X" -> macosArm64("native")   // macOS ARM64
        hostOs == "Linux" -> linuxX64("native")         // Linux x64
        hostOs.startsWith("Windows") -> mingwX64("native") // Windows x64
        else -> throw GradleException("不支持的操作系统: $hostOs")
    }

    nativeTarget.apply {
        // 配置 cinterop
        compilations.getByName("main") {
            cinterops {
                // 创建名为 "mathlib" 的 cinterop 配置
                val mathlib by creating {
                    // def 文件路径（相对于项目根目录）
                    defFile(project.file("native/mathlib.def"))

                    // 额外的编译器选项
                    compilerOpts("-I${project.file("native/include")}")

                    // 额外的头文件搜索路径
                    includeDirs.allHeaders(project.file("native/include"))
                }
            }
        }

        // 配置二进制输出
        binaries {
            executable {
                entryPoint = "main"  // Kotlin main() 函数

                // 链接器选项（平台相关）
                when {
                    hostOs == "Mac OS X" -> {
                        linkerOpts(
                            "-L${project.file("native/lib/macos-arm64")}",
                            "-lmathlib"
                        )
                    }
                    hostOs == "Linux" -> {
                        linkerOpts(
                            "-L${project.file("native/lib/linux-x64")}",
                            "-lmathlib",
                            "-lm"  // 链接数学库（sqrtf 等）
                        )
                    }
                    hostOs.startsWith("Windows") -> {
                        linkerOpts(
                            "-L${project.file("native/lib/windows-x64")}",
                            "-lmathlib"
                        )
                    }
                }
            }
        }
    }
}
```

**settings.gradle.kts**：

```kotlin
// settings.gradle.kts
rootProject.name = "kotlin-native-mathlib"
```

**gradle.properties**：

```properties
# gradle.properties
kotlin.code.style=official
# 启用新版内存管理器（Kotlin 1.7.20+ 默认开启）
kotlin.native.binary.memoryModel=experimental
```

**编译和运行命令**（四平台）：

```bash
# Linux
./gradlew runDebugExecutableNative
# 输出路径: build/bin/native/debugExecutable/kotlin-native-mathlib.kexe

# macOS
./gradlew runDebugExecutableNative
# 输出路径: build/bin/native/debugExecutable/kotlin-native-mathlib.kexe

# Windows (使用 MinGW 工具链)
gradlew.bat runDebugExecutableNative
# 输出路径: build\bin\native\debugExecutable\kotlin-native-mathlib.exe

# Android (NDK 交叉编译——需要额外配置 androidNativeArm64 目标)
# Compose Multiplatform 在 Android 上使用 JVM，不需要 Kotlin/Native
# 只有 iOS/macOS/Linux/Windows 桌面端才用 Kotlin/Native
```

---

## 指针与内存操作

### CPointer 类型体系

cinterop 将 C 的指针类型映射为 Kotlin 的 `CPointer<T>` 类型族。理解这些类型是使用 C 互操作的基础：

| C 类型 | Kotlin/Native 类型 | 说明 |
|--------|-------------------|------|
| `void*` | `COpaquePointer` | 不透明指针（不知道指向什么类型） |
| `int*` | `CPointer<IntVar>` | 指向 int 变量的指针 |
| `float*` | `CPointer<FloatVar>` | 指向 float 变量的指针 |
| `char*` | `CPointer<ByteVar>` | 指向字节的指针（C 字符串） |
| `const char*` | `CPointer<ByteVar>` | 同上（const 在 Kotlin 侧不区分） |
| `struct Foo*` | `CPointer<Foo>` | 指向结构体的指针 |
| `int**` | `CPointer<CPointerVar<IntVar>>` | 指向指针的指针（二级指针） |
| `void (*)(int)` | `CPointer<CFunction<(Int) -> Unit>>` | 函数指针 |
| `NULL` | `null` | 空指针对应 Kotlin 的 null |

### 示例 3：指针操作与内存分配

```kotlin
// src/nativeMain/kotlin/PointerDemo.kt
// 演示 Kotlin/Native 的 C 指针操作和内存管理
import kotlinx.cinterop.*

fun pointerDemo() {
    // ========== 1. memScoped 自动管理原生内存 ==========
    // memScoped 创建一个内存作用域，块结束时自动释放所有分配的原生内存
    memScoped {
        // 分配一个 C int 变量
        val intPtr: CPointer<IntVar> = alloc<IntVar>().ptr
        intPtr.pointed.value = 42
        println("intPtr 指向的值: ${intPtr.pointed.value}")  // 42

        // 分配一个 C float 数组（10 个元素）
        val floatArray: CPointer<FloatVar> = allocArray<FloatVar>(10)
        for (i in 0 until 10) {
            floatArray[i] = i.toFloat() * 1.5f  // 直接用下标访问
        }
        println("floatArray[5] = ${floatArray[5]}")  // 7.5

        // 分配 C 字符串
        val cString: CPointer<ByteVar> = "Hello from Kotlin/Native".cstr.ptr
        // 注意：cstr 在 memScoped 内分配，块结束后失效
        println("C 字符串: ${cString.toKString()}")

        // 分配 C 结构体（假设有 Vec2）
        val point = alloc<Vec2>()
        point.x = 10.0f
        point.y = 20.0f
        println("点坐标: (${point.x}, ${point.y})")
    }
    // memScoped 结束，上面分配的所有原生内存自动释放

    // ========== 2. nativeHeap 手动管理内存 ==========
    // 当需要在 memScoped 之外使用原生内存时
    val heapInt = nativeHeap.alloc<IntVar>()
    heapInt.value = 100
    println("堆分配的值: ${heapInt.value}")
    nativeHeap.free(heapInt)  // 必须手动释放！忘记释放 = 内存泄漏

    // ========== 3. Kotlin 数组与 C 指针互转 ==========
    val kotlinArray = byteArrayOf(72, 101, 108, 108, 111)  // "Hello" 的 ASCII

    // 方式 A：usePinned —— 钉住 Kotlin 数组，获取 C 指针
    kotlinArray.usePinned { pinned ->
        val cPtr: CPointer<ByteVar> = pinned.addressOf(0)
        // cPtr 在此块内有效，可以传给 C 函数
        println("第一个字节: ${cPtr[0]}")  // 72 = 'H'
    }

    // 方式 B：手动分配 C 内存并拷贝
    memScoped {
        val cBuffer = allocArray<ByteVar>(kotlinArray.size)
        kotlinArray.forEachIndexed { index, byte ->
            cBuffer[index] = byte
        }
        // cBuffer 现在包含 kotlinArray 的拷贝
        println("拷贝后第一个字节: ${cBuffer[0]}")  // 72
    }
}
```

---

## 回调函数与函数指针

C 库经常使用回调函数（callback），即将一个函数的指针传给另一个函数，让后者在适当时机调用。cinterop 将 C 函数指针映射为 `CPointer<CFunction<...>>`：

### 示例 4：Kotlin 回调传递给 C

假设我们的 C 库有一个排序函数，接受自定义比较器回调：

```c
// sortlib.h — 带回调的 C 排序库
#ifndef SORTLIB_H
#define SORTLIB_H

#ifdef __cplusplus
extern "C" {
#endif

// 比较函数类型：返回负数(a<b)、零(a==b)、正数(a>b)
typedef int (*compare_func)(const void* a, const void* b);

// 进度回调类型：排序过程中通知进度
typedef void (*progress_callback)(int current, int total, void* user_data);

// 带进度回调的排序函数
void sort_array(
    int* array,           // 待排序数组
    int size,             // 数组长度
    compare_func cmp,     // 比较函数
    progress_callback on_progress,  // 进度回调（可为 NULL）
    void* user_data       // 传给回调的用户数据
);

// 遍历函数：对数组每个元素调用回调
typedef void (*element_callback)(int index, int value, void* user_data);
void for_each(int* array, int size, element_callback cb, void* user_data);

#ifdef __cplusplus
}
#endif

#endif
```

Kotlin/Native 调用代码：

```kotlin
// src/nativeMain/kotlin/CallbackDemo.kt
// 演示将 Kotlin 函数作为 C 回调传递
import com.example.sortlib.*
import kotlinx.cinterop.*

fun callbackDemo() {
    memScoped {
        // 创建 C 数组
        val size = 10
        val array = allocArray<IntVar>(size)
        val values = intArrayOf(5, 3, 8, 1, 9, 2, 7, 4, 6, 0)
        values.forEachIndexed { i, v -> array[i] = v }

        println("排序前: ${(0 until size).map { array[it] }}")

        // 方式 1：使用 staticCFunction 创建 C 兼容的函数指针
        // staticCFunction 要求：lambda 不能捕获任何外部变量
        val comparator = staticCFunction { a: COpaquePointer?, b: COpaquePointer? ->
            // 将 void* 转为 int*，解引用获取值
            val va = a!!.reinterpret<IntVar>().pointed.value
            val vb = b!!.reinterpret<IntVar>().pointed.value
            va - vb  // 升序排序
        }

        // 方式 2：进度回调（同样用 staticCFunction）
        val progressCallback = staticCFunction {
                current: Int, total: Int, _: COpaquePointer? ->
            // 注意：不能在这里访问任何 Kotlin 对象！
            // staticCFunction 的 lambda 在 C 调用约定下执行
            println("排序进度: $current / $total")
        }

        // 调用 C 排序函数
        sort_array(array, size, comparator, progressCallback, null)

        println("排序后: ${(0 until size).map { array[it] }}")

        // 方式 3：使用 StableRef 传递 Kotlin 对象给 C 回调
        // 当回调需要访问 Kotlin 状态时
        val collector = mutableListOf<String>()
        val stableRef = StableRef.create(collector)  // 创建稳定引用

        val elementCallback = staticCFunction {
                index: Int, value: Int, userData: COpaquePointer? ->
            // 从 void* 恢复 Kotlin 对象
            val list = userData!!.asStableRef<MutableList<String>>().get()
            list.add("[$index] = $value")
        }

        for_each(array, size, elementCallback, stableRef.asCPointer())

        println("收集结果: ${collector.joinToString(", ")}")

        stableRef.dispose()  // 必须手动释放 StableRef！否则内存泄漏
    }
}
```

> **`staticCFunction` 的限制**：`staticCFunction` 创建的函数指针只能包含**不捕获任何变量**的 lambda。这是因为 C 函数指针只是一个代码地址，没有"闭包环境"的概念。如果需要传递上下文数据，必须通过 `void* user_data` 参数 + `StableRef` 机制。

---

## 示例 5：完整的 KrKr2 引擎绑定原型

下面是一个模拟 KrKr2 引擎 C 接口的完整 cinterop 绑定示例，展示了如何将游戏引擎的核心功能暴露给 Kotlin/Native：

**C 接口定义**：

```c
// krkr2_api.h — KrKr2 引擎的简化 C 接口
// 注意：实际项目中这个头文件需要从 C++ 类导出 extern "C" 接口
#ifndef KRKR2_API_H
#define KRKR2_API_H

#include <stdint.h>
#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

// ========== 引擎生命周期 ==========
typedef struct KrKr2Engine KrKr2Engine;  // 不透明句柄

// 创建引擎实例
KrKr2Engine* krkr2_create(void);

// 初始化引擎（加载配置、初始化子系统）
bool krkr2_init(KrKr2Engine* engine, const char* game_path);

// 运行一帧游戏逻辑（在游戏循环中每帧调用）
void krkr2_update(KrKr2Engine* engine, float delta_time);

// 渲染当前帧到指定的 OpenGL 纹理
void krkr2_render_to_texture(KrKr2Engine* engine, uint32_t gl_texture_id);

// 销毁引擎实例
void krkr2_destroy(KrKr2Engine* engine);

// ========== 脚本执行 ==========
// 执行 TJS2 脚本
bool krkr2_exec_script(KrKr2Engine* engine, const char* script_path);

// 调用 TJS2 函数
const char* krkr2_call_function(
    KrKr2Engine* engine,
    const char* func_name,
    const char* args_json  // 参数以 JSON 字符串传递
);

// ========== 事件回调 ==========
typedef void (*KrKr2EventCallback)(
    const char* event_type,  // 事件类型
    const char* event_data,  // 事件数据 (JSON)
    void* user_data
);

// 注册事件监听器
void krkr2_set_event_callback(
    KrKr2Engine* engine,
    KrKr2EventCallback callback,
    void* user_data
);

// ========== 配置 ==========
const char* krkr2_get_config(KrKr2Engine* engine, const char* key);
void krkr2_set_config(KrKr2Engine* engine, const char* key, const char* value);

// ========== 存档 ==========
bool krkr2_save_game(KrKr2Engine* engine, int slot);
bool krkr2_load_game(KrKr2Engine* engine, int slot);

// ========== 音频 ==========
void krkr2_set_volume(KrKr2Engine* engine, int channel, float volume);
float krkr2_get_volume(KrKr2Engine* engine, int channel);

#ifdef __cplusplus
}
#endif

#endif
```

**def 文件**：

```
# krkr2.def
package = com.krkr2.native
headers = krkr2_api.h
compilerOpts = -I./native/include
headerFilter = krkr2_api.h
```

**Kotlin/Native 封装层**：

```kotlin
// src/nativeMain/kotlin/KrKr2Bridge.kt
// 对 cinterop 生成的原始绑定进行 Kotlin 风格的封装
import com.krkr2.native.*
import kotlinx.cinterop.*

// Kotlin 风格的事件数据类
data class GameEvent(
    val type: String,   // 事件类型：scene_changed / choice / dialog 等
    val data: String    // 事件数据 (JSON 字符串)
)

// 事件监听器接口
fun interface GameEventListener {
    fun onEvent(event: GameEvent)
}

/**
 * KrKr2 引擎的 Kotlin 封装
 * 将底层 C 指针操作封装为安全的 Kotlin API
 */
class KrKr2Bridge private constructor(
    private val engine: CPointer<KrKr2Engine>
) {
    private var eventListener: GameEventListener? = null
    private var eventStableRef: StableRef<KrKr2Bridge>? = null

    companion object {
        /**
         * 创建并初始化引擎
         * @param gamePath 游戏数据目录路径
         * @return 初始化成功返回 KrKr2Bridge 实例，失败返回 null
         */
        fun create(gamePath: String): KrKr2Bridge? {
            val engine = krkr2_create() ?: return null

            val success = krkr2_init(engine, gamePath)
            if (!success) {
                krkr2_destroy(engine)
                return null
            }

            return KrKr2Bridge(engine)
        }
    }

    /** 运行一帧游戏逻辑 */
    fun update(deltaTime: Float) {
        krkr2_update(engine, deltaTime)
    }

    /** 渲染到指定的 OpenGL 纹理 */
    fun renderToTexture(glTextureId: UInt) {
        krkr2_render_to_texture(engine, glTextureId)
    }

    /** 执行 TJS2 脚本文件 */
    fun execScript(scriptPath: String): Boolean {
        return krkr2_exec_script(engine, scriptPath)
    }

    /** 调用 TJS2 函数，返回 JSON 格式的结果 */
    fun callFunction(funcName: String, argsJson: String = "{}"): String? {
        val result = krkr2_call_function(engine, funcName, argsJson)
        return result?.toKString()
    }

    /** 设置事件监听器 */
    fun setEventListener(listener: GameEventListener?) {
        // 清理旧的 StableRef
        eventStableRef?.dispose()
        eventStableRef = null
        eventListener = listener

        if (listener != null) {
            // 创建 StableRef 以便 C 回调能访问 this 对象
            val ref = StableRef.create(this)
            eventStableRef = ref

            krkr2_set_event_callback(
                engine,
                staticCFunction { eventType, eventData, userData ->
                    val bridge = userData!!.asStableRef<KrKr2Bridge>().get()
                    val event = GameEvent(
                        type = eventType?.toKString() ?: "",
                        data = eventData?.toKString() ?: ""
                    )
                    bridge.eventListener?.onEvent(event)
                },
                ref.asCPointer()
            )
        } else {
            krkr2_set_event_callback(engine, null, null)
        }
    }

    /** 读取配置 */
    fun getConfig(key: String): String? {
        return krkr2_get_config(engine, key)?.toKString()
    }

    /** 设置配置 */
    fun setConfig(key: String, value: String) {
        krkr2_set_config(engine, key, value)
    }

    /** 保存游戏 */
    fun saveGame(slot: Int): Boolean = krkr2_save_game(engine, slot)

    /** 加载存档 */
    fun loadGame(slot: Int): Boolean = krkr2_load_game(engine, slot)

    /** 设置音量 (0.0 ~ 1.0) */
    fun setVolume(channel: Int, volume: Float) {
        krkr2_set_volume(engine, channel, volume.coerceIn(0.0f, 1.0f))
    }

    /** 获取音量 */
    fun getVolume(channel: Int): Float = krkr2_get_volume(engine, channel)

    /** 销毁引擎，释放所有资源 */
    fun destroy() {
        eventStableRef?.dispose()
        eventStableRef = null
        eventListener = null
        krkr2_destroy(engine)
    }
}

// 使用示例
fun main() {
    val bridge = KrKr2Bridge.create("/path/to/game") ?: run {
        println("引擎初始化失败")
        return
    }

    // 设置事件监听
    bridge.setEventListener { event ->
        println("游戏事件: [${event.type}] ${event.data}")
    }

    // 执行脚本
    bridge.execScript("startup.tjs")

    // 游戏主循环
    var running = true
    while (running) {
        bridge.update(1.0f / 60.0f)  // 60 FPS
        // bridge.renderToTexture(textureId) // 渲染到纹理
    }

    bridge.destroy()
}
```

---

## 常见错误与排查

### 错误 1：cinterop 找不到头文件

**现象**：Gradle 构建报错 `error: 'mathlib.h' file not found`。

**原因**：def 文件中的 `compilerOpts -I` 路径不正确，或头文件不在指定路径下。

**排查**：

```bash
# 1. 检查 def 文件中的路径是否正确（相对于项目根目录）
cat native/mathlib.def

# 2. 确认头文件确实存在
ls native/include/mathlib.h

# 3. 在 build.gradle.kts 中用绝对路径调试
# compilerOpts("-I${project.file("native/include").absolutePath}")

# 4. 运行 cinterop 命令查看详细错误
./gradlew cinteropMathlibNative --info
```

### 错误 2：链接时找不到符号

**现象**：编译成功但链接报错 `undefined reference to 'vec2_add'`。

**原因**：静态库路径错误、静态库架构不匹配（如用 x86 库链接 arm64 目标）、或忘记在 def 文件/build.gradle.kts 中指定库。

**排查**：

```bash
# 1. 检查静态库是否存在
ls native/lib/linux-x64/libmathlib.a

# 2. 检查库的架构
file native/lib/linux-x64/libmathlib.a
# 应该显示: current ar archive 或包含 x86_64 的信息

# 3. 检查库中的符号
nm native/lib/linux-x64/libmathlib.a | grep vec2_add
# 应该看到 T vec2_add（T 表示已定义的文本段符号）

# 4. 如果是 C++ 编译的库，函数名会被 name mangling（名称修饰）
# 必须用 extern "C" 包裹导出函数，否则 Kotlin 找不到
```

### 错误 3：staticCFunction lambda 编译报错

**现象**：`staticCFunction { ... }` 编译报错 `Kotlin: Expression is not compilable to static C function`。

**原因**：lambda 捕获了外部变量。`staticCFunction` 只接受**完全无捕获**的 lambda，因为 C 函数指针没有闭包机制。

**修复**：

```kotlin
// 错误：捕获了外部变量 prefix
val prefix = "log: "
val callback = staticCFunction { msg: CPointer<ByteVar>? ->
    println(prefix + msg?.toKString())  // 编译错误！捕获了 prefix
}

// 正确：通过 user_data 参数传递上下文
data class Context(val prefix: String)
val ctx = Context("log: ")
val stableRef = StableRef.create(ctx)

val callback = staticCFunction { msg: CPointer<ByteVar>?, userData: COpaquePointer? ->
    val context = userData!!.asStableRef<Context>().get()
    println(context.prefix + msg?.toKString())
}

// 使用后释放
stableRef.dispose()
```

---

## 动手实践

### 练习 1：编译并运行 mathlib 示例

**目标**：搭建完整的 Kotlin/Native + cinterop 开发环境，成功编译运行 mathlib 示例。

**步骤**：

1. 安装 Kotlin/Native 工具链（通过 SDKMAN 或 IntelliJ IDEA）
2. 用上面的项目结构创建目录和文件
3. 用 `gcc -c mathlib.c -o mathlib.o && ar rcs libmathlib.a mathlib.o` 编译 C 静态库
4. 运行 `./gradlew runDebugExecutableNative`
5. 验证输出是否正确（向量运算结果）

### 练习 2：为 C 回调库编写 Kotlin 封装

**目标**：为一个带回调的 C 库编写类型安全的 Kotlin 封装层。

**步骤**：

1. 编写一个 C 库，提供"定时器"功能：`create_timer(interval_ms, callback, user_data)`
2. 编写 def 文件和 Gradle 配置
3. 在 Kotlin 侧封装为 `class Timer(interval: Duration, block: () -> Unit)`
4. 使用 `StableRef` 将 Kotlin lambda 传递给 C 回调

---

## 对照项目源码

KrKr2 目前使用 C++ 编写，尚未引入 Kotlin/Native。但以下代码是未来 cinterop 绑定时需要导出的 C 接口候选：

相关文件：

- `cpp/core/environ/cocos2d/AppDelegate.cpp` — 引擎启动和主循环入口。cinterop 需要将其封装为 `krkr2_create()` / `krkr2_update()` 风格的 C 接口
- `cpp/core/tjs2/` — TJS2 脚本引擎。需要导出 `exec_script()` / `call_function()` 等接口
- `cpp/core/environ/ConfigManager/` — 配置管理器。需要导出 `get_config()` / `set_config()` 接口
- `cpp/core/sound/` — 音频子系统。需要导出音量控制、播放/暂停等接口
- `cpp/core/base/` — 存档系统（SaveStruct）。需要导出 `save_game()` / `load_game()` 接口

> **关键挑战**：KrKr2 的 C++ 代码大量使用类、虚函数和模板，这些不能直接被 cinterop 绑定。需要先编写一个 `extern "C"` 的 C 兼容接口层（如上面的 `krkr2_api.h`），将 C++ 对象包装为不透明句柄（opaque handle）暴露给 Kotlin。

---

## 本节小结

- **Kotlin/Native** 通过 LLVM 后端将 Kotlin 编译为原生机器码，支持 macOS/iOS/Linux/Windows 等平台，无需 JVM
- **cinterop** 是 Kotlin/Native 的 C 互操作工具，从 `.h` 头文件自动生成 Kotlin 绑定代码，将 C 函数、结构体、枚举、宏等映射为对应的 Kotlin 类型
- **def 文件** 是 cinterop 的配置入口，指定头文件路径、编译器选项、链接库等参数
- **CPointer<T>** 类型族是 C 指针在 Kotlin 中的表示，配合 `pointed`、`reinterpret`、`addressOf` 等方法操作指针
- **memScoped** 提供作用域化的原生内存管理，块结束时自动释放所有分配的 C 内存；`nativeHeap` 提供手动管理的堆分配
- **`usePinned`** 将 Kotlin 数组"钉"在内存中，防止 GC 移动，以便安全地将指针传递给 C 函数
- **`staticCFunction`** 将 Kotlin lambda 转为 C 函数指针，但 lambda 不能捕获任何外部变量
- **`StableRef`** 让 C 代码持有 Kotlin 对象引用，通过 `void* user_data` 传递上下文——使用后必须调用 `dispose()` 释放
- **C++ 互操作** 需要先编写 `extern "C"` 接口层，将 C++ 对象包装为不透明句柄

---

## 练习题与答案

### 题目 1：cinterop 类型映射

以下 C 函数声明，cinterop 会生成怎样的 Kotlin 函数签名？

```c
int process_buffer(const uint8_t* data, size_t length, void (*callback)(int result, void* ctx), void* ctx);
```

<details>
<summary>查看答案</summary>

cinterop 生成的 Kotlin 签名为：

```kotlin
fun process_buffer(
    data: CValuesRef<uint8_tVar>?,    // const uint8_t* → 可空的 C 值引用
    length: ULong,                     // size_t → ULong (64位平台)
    callback: CPointer<CFunction<(Int, COpaquePointer?) -> Unit>>?,  // 函数指针
    ctx: COpaquePointer?               // void* → 可空的不透明指针
): Int                                 // int → Int
```

**类型映射规则解析**：
- `const uint8_t*` → `CValuesRef<uint8_tVar>?`：`CValuesRef` 是比 `CPointer` 更宽泛的类型，既接受 `CPointer` 也接受 `CValue`
- `size_t` → `ULong`：在 64 位平台上 `size_t` 是 8 字节无符号整数
- `void (*)(int, void*)` → `CPointer<CFunction<(Int, COpaquePointer?) -> Unit>>?`：函数指针的完整类型
- `void*` → `COpaquePointer?`：不透明指针（不知道指向什么类型）

**调用示例**：

```kotlin
val result = process_buffer(
    data = kotlinByteArray.refTo(0),  // Kotlin 数组的 C 指针引用
    length = kotlinByteArray.size.toULong(),
    callback = staticCFunction { result, ctx ->
        println("处理结果: $result")
    },
    ctx = null
)
```

</details>

### 题目 2：内存安全分析

以下 Kotlin/Native 代码有什么问题？如何修复？

```kotlin
fun createString(): CPointer<ByteVar> {
    memScoped {
        val str = "Hello World".cstr.ptr
        return str  // 返回指向 memScoped 内存的指针
    }
}
```

<details>
<summary>查看答案</summary>

**问题**：`memScoped` 块结束时，其中分配的所有原生内存都会被自动释放。`str` 指向的内存在 `memScoped` 块结束后变成悬空指针（dangling pointer）——访问已释放的内存属于未定义行为，可能导致崩溃、数据损坏或安全漏洞。

**修复方案**：

```kotlin
// 方案 A：使用 nativeHeap 分配（调用者负责释放）
fun createString(): CPointer<ByteVar> {
    val bytes = "Hello World".encodeToByteArray()
    val buffer = nativeHeap.allocArray<ByteVar>(bytes.size + 1)  // +1 for null terminator
    bytes.forEachIndexed { i, b -> buffer[i] = b }
    buffer[bytes.size] = 0  // null 终止符
    return buffer
    // 调用者必须在使用完后调用 nativeHeap.free(buffer)
}

// 方案 B：使用 Arena（自定义内存池）
val arena = Arena()
fun createStringInArena(): CPointer<ByteVar> {
    return arena.alloc("Hello World".cstr).ptr
}
// 使用完后调用 arena.clear() 释放所有分配

// 方案 C：返回 Kotlin String 而非 C 指针（推荐）
fun createString(): String {
    return "Hello World"  // 直接返回 Kotlin 对象
}
// 只在真正需要传给 C 函数时才转换为 C 指针
```

**最佳实践**：尽量让原生内存的生命周期在调用点可见（使用 `memScoped`），而不是让内存的分配和释放分散在不同函数中。如果确实需要跨函数传递原生指针，使用 `Arena` 或 `nativeHeap` 并在文档中明确标注所有权。

</details>

### 题目 3：设计 KrKr2 的 cinterop 接口

假设你需要将 KrKr2 的视觉渲染模块（`cpp/core/visual/`）暴露给 Kotlin/Native，请：

1. 设计至少 5 个 `extern "C"` 函数声明
2. 编写对应的 `.def` 文件
3. 说明 C++ 对象（如 `tTVPBaseBitmap`）如何通过不透明句柄暴露

<details>
<summary>查看答案</summary>

**1. extern "C" 函数声明**：

```c
// krkr2_visual_api.h
#ifndef KRKR2_VISUAL_API_H
#define KRKR2_VISUAL_API_H

#include <stdint.h>
#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

// 不透明句柄类型
typedef struct KrBitmap KrBitmap;
typedef struct KrLayer KrLayer;

// 创建位图（从文件加载）
KrBitmap* kr_bitmap_load(const char* file_path);

// 创建空白位图
KrBitmap* kr_bitmap_create(int width, int height);

// 获取位图尺寸
void kr_bitmap_get_size(const KrBitmap* bmp, int* width, int* height);

// 获取像素数据指针（RGBA8 格式）
const uint8_t* kr_bitmap_get_pixels(const KrBitmap* bmp);

// 将位图内容复制到 OpenGL 纹理
bool kr_bitmap_upload_to_texture(const KrBitmap* bmp, uint32_t gl_texture_id);

// 销毁位图
void kr_bitmap_destroy(KrBitmap* bmp);

// 创建渲染图层
KrLayer* kr_layer_create(int width, int height);

// 设置图层可见性
void kr_layer_set_visible(KrLayer* layer, bool visible);

// 设置图层位置
void kr_layer_set_position(KrLayer* layer, int x, int y);

// 设置图层透明度 (0.0 ~ 1.0)
void kr_layer_set_opacity(KrLayer* layer, float opacity);

// 销毁图层
void kr_layer_destroy(KrLayer* layer);

#ifdef __cplusplus
}
#endif

#endif
```

**2. def 文件**：

```
# krkr2_visual.def
package = com.krkr2.native.visual
headers = krkr2_visual_api.h
compilerOpts = -I./native/include
headerFilter = krkr2_visual_api.h
staticLibraries = libkrkr2visual.a
libraryPaths = ./native/lib
```

**3. C++ 不透明句柄实现**：

```cpp
// krkr2_visual_api.cpp — C++ 侧实现
#include "krkr2_visual_api.h"
#include "visual/tTVPBaseBitmap.h"  // KrKr2 内部头文件

// KrBitmap 实际上是 tTVPBaseBitmap 的包装
struct KrBitmap {
    tTVPBaseBitmap* impl;  // 持有 C++ 对象指针
};

KrBitmap* kr_bitmap_load(const char* file_path) {
    auto bmp = new KrBitmap();
    bmp->impl = new tTVPBaseBitmap();
    // 调用 KrKr2 内部的图片加载逻辑
    if (!bmp->impl->LoadFromFile(file_path)) {
        delete bmp->impl;
        delete bmp;
        return nullptr;
    }
    return bmp;
}

void kr_bitmap_destroy(KrBitmap* bmp) {
    if (bmp) {
        delete bmp->impl;
        delete bmp;
    }
}

// ... 其他函数类似：在 extern "C" 函数中操作 C++ 对象
```

**不透明句柄的优势**：
- Kotlin 侧只看到 `CPointer<KrBitmap>`，不需要知道 `tTVPBaseBitmap` 的内部布局
- C++ 对象的构造、析构、继承等特性完全封装在 C 接口层内部
- 接口稳定：即使 C++ 类的实现变化，C 接口和 Kotlin 绑定都不需要修改

</details>

---

## 下一步

下一节我们将学习 Compose Multiplatform 的渲染架构——它如何使用 Skia 进行跨平台渲染，以及如何与 C++ 游戏引擎共享渲染上下文：

→ [02-Skia渲染共享与平台集成.md](./02-Skia渲染共享与平台集成.md)
