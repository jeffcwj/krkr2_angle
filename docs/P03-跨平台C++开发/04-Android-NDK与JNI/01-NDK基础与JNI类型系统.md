# NDK 基础与 JNI 类型系统

> **所属模块：** P03-跨平台 C++ 开发
> **前置知识：** [01-跨平台基础](../01-跨平台基础/01-文件系统与线程差异.md)、[02-预处理器与条件编译](../02-预处理器与条件编译/01-预处理器基础与平台宏.md)、[03-平台抽象层设计模式](../03-平台抽象层设计模式/01-PAL概念与编译期文件分离.md)
> **预计阅读时间：** 45 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 Android NDK 的作用、组成结构和与 SDK 的分工关系
2. 解释为什么游戏引擎（如 KrKr2）必须使用 NDK 而不能纯 Java 开发
3. 掌握 JNI 基本类型和引用类型的完整映射表，以及每种类型的内存大小差异
4. 正确编写 JNI 方法签名字符串，避免签名不匹配导致的 `NoSuchMethodError`
5. 通过阅读 KrKr2 项目中的真实 JNI 签名，验证自己的签名编写能力

---

## 为什么需要 NDK？—— 一个真实场景

### 场景：你继承了一个 C++ 游戏引擎

假设你是一名 Android 开发者，你的公司决定把一款原本只在 Windows 上运行的视觉小说引擎移植到 Android 平台。这个引擎有以下特点：

- **30 万行 C++ 代码**：包括脚本引擎（TJS2）、渲染管线、音视频解码器
- **依赖大量 C/C++ 第三方库**：FFmpeg（音视频）、zlib（压缩）、SDL2（输入）、OpenGL ES（渲染）
- **性能要求极高**：需要 60fps 实时渲染、低延迟音频播放

面对这种情况，你有两个选择：

**选择 A**：用 Java/Kotlin 重写所有 C++ 代码

```
工作量预估：
├── TJS2 脚本引擎：~8 万行 C++ → 可能需要 12+ 万行 Java
├── 渲染管线：~5 万行 C++ → 需要大量 JNI 调用 OpenGL ES（讽刺的是还是要 JNI）
├── FFmpeg 集成：Java 没有原生 FFmpeg → 还是需要 JNI
├── 性能损失：Java 的 GC 停顿会导致画面卡顿
└── 维护成本：两套代码永远不会完全同步
```

**选择 B**：用 NDK 直接在 Android 上运行现有 C++ 代码

```
工作量预估：
├── 核心引擎：直接编译，不需要改动（共享 95%+ 的代码）
├── 平台层：写一层薄薄的 JNI 桥接（~1000 行）
├── 性能：C++ 直接跑在 ARM 处理器上，没有 GC 停顿
└── 维护成本：只维护一套核心代码
```

**KrKr2 选择了方案 B**。事实上，几乎所有跨平台游戏引擎（Unity、Unreal、Cocos2d-x、Godot）都使用 NDK。这就是 NDK 存在的根本原因：**让已有的 C/C++ 代码能在 Android 上运行**。

### NDK 到底是什么？

NDK 的全称是 **Native Development Kit**（原生开发工具包）。"Native"（原生）是相对于 Java/Kotlin 的"托管"代码而言的——C/C++ 代码直接编译成 ARM/x86 机器码，不需要 Java 虚拟机（JVM/ART）来执行。

用一个比喻来理解：

```
Android 系统就像一座大楼：

┌─────────────────────────────────────────┐
│  楼上住户（Java/Kotlin 代码）            │
│  ├── 有物业管理（ART 运行时 + GC）       │
│  ├── 可以用电梯、门禁（系统 API）        │
│  └── 生活便利但行动受限                  │
├─────────────────────────────────────────┤
│  一楼大厅（JNI 桥接层）                  │
│  └── 楼上楼下的人在这里交接物品           │
├─────────────────────────────────────────┤
│  地下车库（C/C++ 代码）                  │
│  ├── 没有物业管（自己管理内存）           │
│  ├── 直接连通外部道路（硬件访问）         │
│  └── 活动自由但需要自己负责安全           │
├─────────────────────────────────────────┤
│  地基（Linux 内核）                      │
└─────────────────────────────────────────┘
```

---

## NDK 的核心组件

NDK 不是一个单独的程序，而是一整套工具和文件的集合。下面我们逐一介绍每个组件：

### 1. 交叉编译工具链（Clang/LLVM）

Android 设备使用 ARM 或 x86 处理器，而你的开发电脑通常是 x86_64 架构。**交叉编译**（Cross Compilation）是指在一种架构的机器上，编译出另一种架构的可执行代码。

```
你的电脑（x86_64）               Android 设备（ARM64）
┌──────────────┐                ┌──────────────┐
│  hello.cpp   │    编译         │  hello.so    │
│  (C++ 源码)  │ ──────────→    │  (ARM64 机器码) │
└──────────────┘  NDK Clang     └──────────────┘
                  交叉编译器
```

NDK 内置了 Clang/LLVM 编译器，针对四种 CPU 架构（ABI）预配置好了：

| ABI 名称 | CPU 架构 | 说明 | KrKr2 是否支持 |
|-----------|----------|------|----------------|
| `arm64-v8a` | ARM 64 位 | 现代手机主流架构 | ✅ 是 |
| `armeabi-v7a` | ARM 32 位 | 旧设备兼容 | ❌ 已注释掉 |
| `x86_64` | Intel/AMD 64 位 | 模拟器和少量平板 | ✅ 是 |
| `x86` | Intel/AMD 32 位 | 旧模拟器 | ❌ 不支持 |

在 KrKr2 的 `platforms/android/app/build.gradle` 中可以看到 ABI 配置：

```gradle
// platforms/android/app/build.gradle 第 16-18 行
ndk {
    jobs Runtime.runtime.availableProcessors()
    abiFilters 'arm64-v8a', 'x86_64' //, 'armeabi-v7a'
}
```

### 2. 系统头文件和库（Sysroot）

NDK 提供了 Android 平台的 C/C++ 头文件，让你的代码能调用 Android 特有的 API：

```cpp
// Android 日志输出 —— 类似于 printf，但输出到 logcat
#include <android/log.h>
__android_log_print(ANDROID_LOG_INFO, "MyApp", "Hello from C++!");

// JNI 接口 —— Java 和 C++ 之间的桥梁
#include <jni.h>

// EGL —— OpenGL ES 的显示管理
#include <EGL/egl.h>

// OpenGL ES —— 图形渲染
#include <GLES2/gl2.h>

// OpenSL ES —— 音频播放（已被 AAudio/Oboe 替代）
#include <SLES/OpenSLES.h>
```

### 3. CMake 工具链文件

NDK 自带一个 CMake 工具链文件（`android.toolchain.cmake`），告诉 CMake 如何使用 NDK 的编译器。在 KrKr2 中，Gradle 会自动调用 CMake 并传入这个工具链文件：

```gradle
// platforms/android/app/build.gradle 第 32-38 行
externalNativeBuild {
    cmake {
        version '3.31.1'
        buildStagingDirectory '../../../out/android/cmake-build'
        path '../../../CMakeLists.txt'  // 指向项目根目录的 CMakeLists.txt
    }
}
```

这意味着 Android 构建和 Windows/Linux/macOS 构建共用同一个 `CMakeLists.txt`，通过条件编译（第 02 章学过的 `#ifdef ANDROID`）来处理平台差异。

### 4. NDK 版本选择

不同 NDK 版本支持不同的 Android API Level（最低系统版本）。KrKr2 使用的配置：

```gradle
// platforms/android/app/build.gradle 第 7 行
ndkVersion '28.0.13004108'  // NDK r28
```

---

## NDK 与 SDK 的分工

一个常见的困惑是："既然 NDK 这么强大，为什么不全用 C++ 写 Android 应用？"答案是：**Android 的核心系统服务（Activity、UI、权限）只暴露了 Java/Kotlin API**。

```
SDK 负责（Java/Kotlin）：               NDK 负责（C/C++）：
├── Activity 生命周期管理               ├── 性能敏感的计算（渲染、解码）
├── UI 控件（Button、TextView）         ├── 已有 C/C++ 代码库的移植
├── 系统服务（通知、传感器、GPS）        ├── 第三方 C/C++ 库集成
├── 权限请求（存储、相机）              ├── 底层硬件访问（OpenGL ES、音频）
├── 网络请求（HTTP、WebSocket）         ├── 跨平台核心逻辑共享
└── 应用分发（APK/AAB 打包）            └── 密集计算（图像处理、AI 推理）
```

**KrKr2 的分工方式**：

| 功能 | 实现语言 | 文件位置 |
|------|----------|----------|
| 引擎核心（TJS2、渲染、音频） | C++ | `cpp/core/` |
| JNI 桥接（触摸、按键、对话框） | C++ | `platforms/android/cpp/krkr2_android.cpp` |
| Android 平台工具（内存、文件、设备 ID） | C++（调用 Java） | `cpp/core/environ/android/AndroidUtils.cpp` |
| Activity 生命周期、UI、权限 | Java | `platforms/android/app/java/org/tvp/kirikiri2/KR2Activity.java` |

---

## JNI 是什么？—— Java 与 C++ 的翻译官

### 问题：两个世界的语言不通

Java 和 C++ 是完全不同的语言，它们的内存模型、类型系统、对象生命周期都不一样：

| 特性 | Java | C++ |
|------|------|------|
| 内存管理 | 垃圾回收器（GC）自动管理 | 手动管理（或智能指针） |
| 字符串 | `String`（UTF-16，不可变对象） | `std::string`（UTF-8，可变） |
| 整数大小 | `int` 固定 32 位 | `int` 大小取决于平台 |
| 布尔值 | `boolean` 是独立类型 | `bool` 本质是整数 |
| 对象模型 | 单继承 + 接口 + GC | 多继承 + 手动析构 |
| 异常 | 必须声明（checked exception） | 可选的 try-catch |

**JNI**（Java Native Interface，Java 原生接口）就是连接这两个世界的桥梁。它定义了一套标准的类型映射和函数调用规则，让 Java 代码能调用 C++ 函数，C++ 代码也能调用 Java 方法。

### JNI 的工作流程

```
Java 代码                    JNI 层                     C++ 代码
┌──────────┐              ┌──────────┐              ┌──────────┐
│ 调用      │   参数转换   │ 类型映射  │   参数转换   │ 执行      │
│ native   │ ──────────→ │ jint     │ ──────────→ │ int32_t  │
│ method() │              │ jstring  │              │ const    │
│          │ ←────────── │ 返回值   │ ←────────── │ char*    │
│ 接收返回值│   结果转换   │ 转换     │   结果转换   │ 返回结果  │
└──────────┘              └──────────┘              └──────────┘
```

---

## JNI 基本类型映射

JNI 最核心的设计就是类型映射——它为 Java 的每种基本类型定义了一个对应的 C/C++ 类型。这些类型定义在 `<jni.h>` 头文件中。

### 完整映射表

```cpp
// 这些类型定义在 NDK 的 <jni.h> 头文件中
// 每一行的格式：Java 类型 → JNI C 类型 → 实际 C++ 类型 → 字节大小 → JNI 签名符号

// boolean  →  jboolean  →  uint8_t    →  1 字节  →  Z
// byte     →  jbyte     →  int8_t     →  1 字节  →  B
// char     →  jchar     →  uint16_t   →  2 字节  →  C
// short    →  jshort    →  int16_t    →  2 字节  →  S
// int      →  jint      →  int32_t    →  4 字节  →  I
// long     →  jlong     →  int64_t    →  8 字节  →  J （注意不是 L！）
// float    →  jfloat    →  float      →  4 字节  →  F
// double   →  jdouble   →  double     →  8 字节  →  D
// void     →  void      →  void       →  -       →  V
```

### 三个容易踩的坑

**坑 1：jboolean 不是 C++ 的 bool**

```cpp
// ❌ 错误：直接把 jboolean 当 bool 用
JNIEXPORT void JNICALL Java_MyClass_setEnabled(
    JNIEnv *env, jobject obj, jboolean enabled) {
    // jboolean 是 uint8_t（0 或 1），不是 C++ 的 bool
    if (enabled) {  // 这样写碰巧能用，但类型不安全
        doSomething();
    }
}

// ✅ 正确：用 JNI 常量比较
JNIEXPORT void JNICALL Java_MyClass_setEnabled(
    JNIEnv *env, jobject obj, jboolean enabled) {
    if (enabled == JNI_TRUE) {  // 明确比较，类型安全
        doSomething();
    }
}

// ✅ 或者：显式转换为 bool
bool cppEnabled = (enabled == JNI_TRUE);
```

**坑 2：jchar 是 16 位无符号（UTF-16），不是 8 位 char**

```cpp
// ❌ 错误：把 jchar 当 ASCII char 处理
JNIEXPORT void JNICALL Java_MyClass_processChar(
    JNIEnv *env, jobject obj, jchar ch) {
    char c = ch;  // 危险！如果 ch > 255（比如中文字符），会截断
    printf("%c", c);  // 输出乱码
}

// ✅ 正确：认识到 jchar 是 UTF-16 编码单元
JNIEXPORT void JNICALL Java_MyClass_processChar(
    JNIEnv *env, jobject obj, jchar ch) {
    // jchar 是 uint16_t，对应 Java 的 UTF-16 字符
    if (ch < 128) {
        // ASCII 范围内，可以安全转为 char
        char c = static_cast<char>(ch);
        printf("%c", c);
    } else {
        // 非 ASCII（如中文），需要 UTF-16 → UTF-8 转换
        printf("Unicode: U+%04X\n", ch);
    }
}
```

**坑 3：jlong 的签名符号是 J，不是 L**

```
❌ 初学者常犯的错误：
Java: long getValue()
错误签名: "()L"   ← L 是引用类型的前缀！
正确签名: "()J"   ← J 才是 long 的符号

助记法：L 被引用类型占用了（Ljava/lang/String;），
       所以 long 只好用 J（Jong? Long 的变体）
```

### 验证类型大小的代码

下面是一个完整的可编译示例，用来验证 JNI 类型的实际大小：

```cpp
// jni_type_sizes.cpp
// 编译：无需 Android 环境，任何支持 <cstdint> 的 C++11 编译器即可
#include <cstdint>
#include <cstdio>

// 模拟 jni.h 中的类型定义（实际使用时直接 #include <jni.h>）
typedef uint8_t  jboolean;
typedef int8_t   jbyte;
typedef uint16_t jchar;
typedef int16_t  jshort;
typedef int32_t  jint;
typedef int64_t  jlong;
typedef float    jfloat;
typedef double   jdouble;

int main() {
    printf("JNI 基本类型大小验证：\n");
    printf("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n");
    printf("jboolean: %zu 字节 (Java boolean)\n", sizeof(jboolean));
    printf("jbyte:    %zu 字节 (Java byte)\n",    sizeof(jbyte));
    printf("jchar:    %zu 字节 (Java char)\n",    sizeof(jchar));
    printf("jshort:   %zu 字节 (Java short)\n",   sizeof(jshort));
    printf("jint:     %zu 字节 (Java int)\n",     sizeof(jint));
    printf("jlong:    %zu 字节 (Java long)\n",    sizeof(jlong));
    printf("jfloat:   %zu 字节 (Java float)\n",   sizeof(jfloat));
    printf("jdouble:  %zu 字节 (Java double)\n",  sizeof(jdouble));

    // 对比：C++ 原生类型（大小可能因平台而异）
    printf("\nC++ 原生类型对比：\n");
    printf("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n");
    printf("bool:          %zu 字节\n", sizeof(bool));
    printf("char:          %zu 字节\n", sizeof(char));
    printf("int:           %zu 字节\n", sizeof(int));
    printf("long:          %zu 字节\n", sizeof(long));      // 可能是 4 或 8！
    printf("long long:     %zu 字节\n", sizeof(long long));

    return 0;
}
```

在 Windows 上编译运行：
```bash
cl /EHsc jni_type_sizes.cpp && jni_type_sizes.exe
```

在 Linux/macOS 上编译运行：
```bash
g++ -std=c++11 -o jni_type_sizes jni_type_sizes.cpp && ./jni_type_sizes
```

预期输出：
```
JNI 基本类型大小验证：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
jboolean: 1 字节 (Java boolean)
jbyte:    1 字节 (Java byte)
jchar:    2 字节 (Java char)
jshort:   2 字节 (Java short)
jint:     4 字节 (Java int)
jlong:    8 字节 (Java long)
jfloat:   4 字节 (Java float)
jdouble:  8 字节 (Java double)

C++ 原生类型对比：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
bool:          1 字节
char:          1 字节
int:           4 字节
long:          4 字节    ← Windows 上是 4 字节
long long:     8 字节
```

注意 `long` 在 Windows 上是 4 字节，但在 Linux/macOS 64 位上是 8 字节。这就是为什么 JNI 使用 `int64_t` 而不是 `long long` 来定义 `jlong`——**JNI 类型保证跨平台大小一致**。

---

## JNI 引用类型映射

Java 中所有非基本类型都是引用类型（指向堆上对象的指针）。JNI 把它们统一映射为 `jobject` 的子类型。

### 引用类型层次

```
jobject                 ← 所有 Java 对象的基类
├── jclass              ← java.lang.Class 对象
├── jstring             ← java.lang.String 对象
├── jthrowable          ← java.lang.Throwable 及其子类
├── jarray              ← 所有数组的基类
│   ├── jbooleanArray   ← boolean[]
│   ├── jbyteArray      ← byte[]
│   ├── jcharArray      ← char[]
│   ├── jshortArray     ← short[]
│   ├── jintArray       ← int[]
│   ├── jlongArray      ← long[]
│   ├── jfloatArray     ← float[]
│   ├── jdoubleArray    ← double[]
│   └── jobjectArray    ← Object[]（包括 String[] 等）
└── jweak               ← 弱引用
```

### 引用类型的 JNI 签名规则

引用类型的签名比基本类型复杂，有三条规则：

**规则 1：类签名 = `L` + 全限定类名（`.` 替换为 `/`）+ `;`**

```
Java 类型                    JNI 签名
─────────────────────────────────────────
Object                  →   Ljava/lang/Object;
String                  →   Ljava/lang/String;
java.io.File            →   Ljava/io/File;
android.app.Activity    →   Landroid/app/Activity;
org.tvp.kirikiri2.KR2Activity  →  Lorg/tvp/kirikiri2/KR2Activity;
```

**规则 2：数组签名 = `[` + 元素签名**

```
Java 类型                    JNI 签名
─────────────────────────────────────────
int[]                   →   [I
byte[]                  →   [B
float[]                 →   [F
String[]                →   [Ljava/lang/String;
int[][]（二维数组）       →   [[I
Object[]                →   [Ljava/lang/Object;
```

**规则 3：方法签名 = `(` + 参数签名... + `)` + 返回类型签名**

```
Java 方法声明                              JNI 签名
──────────────────────────────────────────────────────────────
void foo()                              →  ()V
int add(int a, int b)                   →  (II)I
String getName()                        →  ()Ljava/lang/String;
void setData(byte[] data)              →  ([B)V
long process(String s, int n)          →  (Ljava/lang/String;I)J
boolean exists(File f)                 →  (Ljava/io/File;)Z
void update(String t, String d, String[] b)
                                       →  (Ljava/lang/String;Ljava/lang/String;[Ljava/lang/String;)V
```

### 常见签名错误

**错误 1：忘记引用类型末尾的分号**

```
❌ 错误：(Ljava/lang/String)V    ← 少了分号
✅ 正确：(Ljava/lang/String;)V   ← 注意 String 后面的分号
```

**错误 2：用 `.` 代替 `/`**

```
❌ 错误：(Ljava.lang.String;)V   ← 用了点号
✅ 正确：(Ljava/lang/String;)V   ← 必须用斜杠
```

**错误 3：数组签名和引用签名混淆**

```
❌ 错误：[Ljava/lang/String      ← 少了分号
✅ 正确：[Ljava/lang/String;     ← 分号是类签名的一部分

❌ 错误：Ljava/lang/String[];    ← Java 语法，不是 JNI 签名
✅ 正确：[Ljava/lang/String;     ← [ 放在最前面
```

---

## 实战：手写 JNI 签名

让我们通过一个完整的练习来巩固签名编写能力。

### 步骤 1：创建 Java 类

```java
// Calculator.java
package com.example.jni;

public class Calculator {
    // 方法 1：无参数，返回 int
    public native int getVersion();

    // 方法 2：两个 int 参数，返回 int
    public native int add(int a, int b);

    // 方法 3：String 参数，返回 boolean
    public native boolean validateInput(String input);

    // 方法 4：int 数组参数，返回 double
    public native double average(int[] numbers);

    // 方法 5：复杂参数组合
    public native String format(String template, int count, float ratio);
}
```

### 步骤 2：推导签名

```
方法 1: getVersion()
  参数：(无)
  返回：int → I
  签名："()I"

方法 2: add(int, int)
  参数：int → I, int → I
  返回：int → I
  签名："(II)I"

方法 3: validateInput(String)
  参数：String → Ljava/lang/String;
  返回：boolean → Z
  签名："(Ljava/lang/String;)Z"

方法 4: average(int[])
  参数：int[] → [I
  返回：double → D
  签名："([I)D"

方法 5: format(String, int, float)
  参数：String → Ljava/lang/String;  int → I  float → F
  返回：String → Ljava/lang/String;
  签名："(Ljava/lang/String;IF)Ljava/lang/String;"
```

### 步骤 3：编写对应的 C++ 实现

```cpp
// calculator_jni.cpp
#include <jni.h>
#include <cstring>
#include <cstdio>
#include <numeric>  // std::accumulate

extern "C" {

// 方法 1：无参数，返回 int
JNIEXPORT jint JNICALL
Java_com_example_jni_Calculator_getVersion(JNIEnv *env, jobject obj) {
    return 1;  // 版本号 1
}

// 方法 2：两个 int 参数，返回 int
JNIEXPORT jint JNICALL
Java_com_example_jni_Calculator_add(JNIEnv *env, jobject obj,
                                     jint a, jint b) {
    return a + b;  // jint 就是 int32_t，可以直接运算
}

// 方法 3：String 参数，返回 boolean
JNIEXPORT jboolean JNICALL
Java_com_example_jni_Calculator_validateInput(JNIEnv *env, jobject obj,
                                               jstring input) {
    // 从 Java String 获取 UTF-8 C 字符串
    const char *str = env->GetStringUTFChars(input, nullptr);
    if (str == nullptr) {
        return JNI_FALSE;  // 内存分配失败
    }

    // 检查输入是否非空
    jboolean result = (strlen(str) > 0) ? JNI_TRUE : JNI_FALSE;

    // ⚠️ 必须释放！否则内存泄漏
    env->ReleaseStringUTFChars(input, str);
    return result;
}

// 方法 4：int 数组参数，返回 double
JNIEXPORT jdouble JNICALL
Java_com_example_jni_Calculator_average(JNIEnv *env, jobject obj,
                                         jintArray numbers) {
    // 获取数组长度
    jsize length = env->GetArrayLength(numbers);
    if (length == 0) {
        return 0.0;
    }

    // 获取数组元素的指针
    jint *elements = env->GetIntArrayElements(numbers, nullptr);
    if (elements == nullptr) {
        return 0.0;  // 内存分配失败
    }

    // 计算平均值
    jlong sum = 0;
    for (jsize i = 0; i < length; i++) {
        sum += elements[i];
    }
    jdouble avg = static_cast<jdouble>(sum) / length;

    // ⚠️ 必须释放！第三个参数 0 表示将修改拷贝回原数组
    // 如果不需要写回，用 JNI_ABORT
    env->ReleaseIntArrayElements(numbers, elements, JNI_ABORT);
    return avg;
}

// 方法 5：复杂参数组合
JNIEXPORT jstring JNICALL
Java_com_example_jni_Calculator_format(JNIEnv *env, jobject obj,
                                        jstring tmpl, jint count,
                                        jfloat ratio) {
    const char *tmplStr = env->GetStringUTFChars(tmpl, nullptr);
    if (tmplStr == nullptr) {
        return nullptr;
    }

    // 格式化字符串
    char buffer[256];
    snprintf(buffer, sizeof(buffer), "%s: count=%d, ratio=%.2f",
             tmplStr, count, ratio);

    env->ReleaseStringUTFChars(tmpl, tmplStr);

    // 从 C 字符串创建 Java String 返回
    return env->NewStringUTF(buffer);
}

}  // extern "C"
```

---

## 对照项目源码

### KrKr2 中的真实 JNI 签名

KrKr2 项目中大量使用了 JNI 签名来从 C++ 调用 Java 方法。让我们对照几个真实案例：

**案例 1：获取内存信息（无参数，返回 void / long）**

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 51-64 行

// Java 端：static void updateMemoryInfo()
// 签名分析：无参数 → ()，返回 void → V
JniHelper::getStaticMethodInfo(methodInfo, KR2ActJavaPath,
                               "updateMemoryInfo", "()V");
//                                                  ^^^^ 签名

// Java 端：static long getAvailMemory()
// 签名分析：无参数 → ()，返回 long → J
JniHelper::getStaticMethodInfo(methodInfo, KR2ActJavaPath,
                               "getAvailMemory", "()J");
//                                                ^^^^ 签名
```

**案例 2：显示消息对话框（三个参数，两个 String + 一个 String[]）**

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 518-521 行

// Java 端：static void ShowMessageBox(String title, String text, String[] btns)
// 签名分析：
//   参数1 String → Ljava/lang/String;
//   参数2 String → Ljava/lang/String;
//   参数3 String[] → [Ljava/lang/String;
//   返回 void → V
JniHelper::getStaticMethodInfo(
    methodInfo, "org/tvp/kirikiri2/KR2Activity", "ShowMessageBox",
    "(Ljava/lang/String;Ljava/lang/String;[Ljava/lang/String;)V");
//   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ 签名
```

**案例 3：获取存储路径（无参数，返回 String[]）**

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 417-418 行

// Java 端：String[] getStoragePath()
// 签名分析：无参数 → ()，返回 String[] → [Ljava/lang/String;
JniHelper::getMethodInfo(methodInfo, KR2ActJavaPath, "getStoragePath",
                         "()[Ljava/lang/String;");
//                        ^^^^^^^^^^^^^^^^^^^^^^ 签名
```

**案例 4：检查路径可写性（String 参数，返回 boolean）**

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 750-752 行

// Java 端：static boolean isWritableNormalOrSaf(String path)
// 签名分析：参数 String → Ljava/lang/String;，返回 boolean → Z
JniHelper::getStaticMethodInfo(
    methodInfo, "org/tvp/kirikiri2/KR2Activity", "isWritableNormalOrSaf",
    "(Ljava/lang/String;)Z");
//   ^^^^^^^^^^^^^^^^^^^^^ 签名
```

**案例 5：获取 Activity 实例（无参数，返回自定义类型）**

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 176-182 行

// Java 端：static KR2Activity GetInstance()
// 签名分析：无参数 → ()，返回 KR2Activity → Lorg/tvp/kirikiri2/KR2Activity;
// 但 KrKr2 用了动态拼接：
std::string strtmp("()L");
strtmp += KR2ActJavaPath;       // "org/tvp/kirikiri2/KR2Activity"
strtmp += ";";
// 最终签名："()Lorg/tvp/kirikiri2/KR2Activity;"
JniHelper::getStaticMethodInfo(methodInfo, KR2ActJavaPath,
                               "GetInstance", strtmp.c_str());
```

注意这里用了字符串拼接来构造签名——因为类路径在代码中是常量 `KR2ActJavaPath`，拼接可以避免重复写路径。

---

## 动手实践

### 练习 1：签名推导

为下面的 Java 方法写出 JNI 签名：

```java
public class FileManager {
    // 1. 创建目录
    public native boolean createDirectory(String path);

    // 2. 读取文件内容
    public native byte[] readFile(String path, int offset, int length);

    // 3. 批量删除文件
    public native int deleteFiles(String[] paths, boolean recursive);

    // 4. 获取文件信息
    public native long getFileSize(String path);
}
```

<details>
<summary>查看答案</summary>

```
1. createDirectory(String path) → boolean
   签名："(Ljava/lang/String;)Z"

2. readFile(String path, int offset, int length) → byte[]
   签名："(Ljava/lang/String;II)[B"

3. deleteFiles(String[] paths, boolean recursive) → int
   签名："([Ljava/lang/String;Z)I"

4. getFileSize(String path) → long
   签名："(Ljava/lang/String;)J"
   注意：long 的签名是 J，不是 L！
```

</details>

### 练习 2：编写完整的 JNI 函数

写一个 C++ 函数，实现 Java 方法 `int sumArray(int[] arr)` 的 JNI 绑定。要求：
1. 计算数组所有元素的和
2. 正确处理空数组和 null 输入
3. 记得释放数组元素

```java
package com.example;
public class MathUtils {
    public native int sumArray(int[] arr);
}
```

<details>
<summary>查看答案</summary>

```cpp
#include <jni.h>

extern "C"
JNIEXPORT jint JNICALL
Java_com_example_MathUtils_sumArray(JNIEnv *env, jobject obj,
                                     jintArray arr) {
    // 处理 null 输入
    if (arr == nullptr) {
        return 0;
    }

    // 获取数组长度
    jsize length = env->GetArrayLength(arr);
    if (length == 0) {
        return 0;
    }

    // 获取数组元素指针
    jint *elements = env->GetIntArrayElements(arr, nullptr);
    if (elements == nullptr) {
        return 0;  // 内存分配失败
    }

    // 计算总和
    jint sum = 0;
    for (jsize i = 0; i < length; i++) {
        sum += elements[i];
    }

    // 释放数组元素（JNI_ABORT = 不拷贝回原数组）
    env->ReleaseIntArrayElements(arr, elements, JNI_ABORT);

    return sum;
}
```

JNI 签名是 `([I)I`：参数 `int[]` 签名为 `[I`，返回 `int` 签名为 `I`。

</details>

---

## 本节小结

- **NDK** 是 Google 提供的工具集，让 C/C++ 代码能在 Android 上运行。它包含交叉编译器（Clang）、系统头文件（Sysroot）、CMake 工具链文件
- **NDK 存在的根本原因**：大量优秀的 C/C++ 代码库（游戏引擎、多媒体、科学计算）需要在 Android 上复用
- **JNI** 是 Java 与 C/C++ 之间的桥梁，它定义了类型映射和函数调用规则
- **JNI 基本类型**使用固定大小的整数类型（`int32_t`、`int64_t` 等），保证跨平台一致性
- **JNI 签名**是描述方法参数和返回类型的特殊字符串格式，签名错误会导致运行时 `NoSuchMethodError`
- KrKr2 大量使用 `JniHelper::getStaticMethodInfo()` 来动态查找 Java 方法，签名必须完全匹配

---

## 练习题与答案

### 题目 1：为什么 KrKr2 需要 NDK？

KrKr2 的引擎核心（TJS2 脚本引擎、渲染管线、FFmpeg 音视频解码）全部用 C++ 编写。请解释为什么不能用 Java 重写这些模块，以及 NDK 方案的具体优势。

<details>
<summary>查看答案</summary>

不能用 Java 重写的原因：
1. **工作量巨大**：30 万行 C++ 代码重写为 Java 可能需要 1-2 年，且容易引入新 Bug
2. **性能损失**：Java 的垃圾回收器（GC）会导致不可预测的暂停（STW），影响 60fps 实时渲染和低延迟音频
3. **FFmpeg 无法用 Java 替代**：FFmpeg 是 C 库，Java 没有对等实现。即使封装也需要 JNI
4. **跨平台代码无法共享**：重写后只能在 Android 上用，Windows/Linux/macOS 还是需要 C++ 版本

NDK 方案的优势：
1. **代码复用 95%+**：核心引擎代码不需要改动，只需添加平台层
2. **原生性能**：C++ 直接编译为 ARM 机器码，没有 GC 停顿
3. **统一维护**：所有平台共用一套核心代码，Bug 修复一次全平台生效
4. **第三方库生态**：可以直接使用 FFmpeg、SDL2、OpenGL ES 等 C/C++ 库

</details>

### 题目 2：找出以下 JNI 签名中的错误

```
1. Java: void print(String msg)
   签名: (Ljava/lang/String)V

2. Java: long[] getSizes()
   签名: ()[J

3. Java: boolean check(int code, String name)
   签名: (ILjava.lang.String;)Z

4. Java: File getFile(String path)
   签名: (Ljava/lang/String;)Ljava/io/File;
```

<details>
<summary>查看答案</summary>

```
1. ❌ 错误：String 签名缺少末尾分号
   修正：(Ljava/lang/String;)V
                          ^ 加上分号

2. ✅ 正确：long[] 签名是 [J，方法签名 ()[J 无误
   （虽然 long 签名是 J 不是 L，但这里写对了）

3. ❌ 错误：类路径中使用了点号 (.) 而不是斜杠 (/)
   修正：(ILjava/lang/String;)Z
              ^         ^ 点号改为斜杠

4. ✅ 正确：File 的完整签名是 Ljava/io/File;，参数和返回值格式正确
```

</details>

### 题目 3：阅读 KrKr2 源码，推导签名

查看 `cpp/core/environ/android/AndroidUtils.cpp` 中的 `TVPExitApplication` 函数（第 904-915 行），它调用了 Java 静态方法 `KR2Activity.exit()`。

问题：
1. 这个 Java 方法的签名是什么？
2. 为什么退出前要调用 `TVPDeliverCompactEvent(TVP_COMPACT_LEVEL_MAX)`？
3. C++ 的 `exit(code)` 和 JNI 调用 `KR2Activity.exit()` 有什么区别？

<details>
<summary>查看答案</summary>

1. **签名是 `"()V"`**：`exit()` 方法无参数，返回 void。对应源码第 910 行：
   ```cpp
   JniHelper::getStaticMethodInfo(t, "org/tvp/kirikiri2/KR2Activity",
                                  "exit", "()V");
   ```

2. **`TVPDeliverCompactEvent(TVP_COMPACT_LEVEL_MAX)` 的作用**：触发引擎的最大级别资源回收。在退出前释放所有缓存的纹理、音频缓冲区、脚本对象等。这是一种优雅退出（graceful shutdown），确保资源不泄漏。

3. **两者的区别**：
   - `KR2Activity.exit()`：通知 Java 层进行 Activity 的正常关闭流程（调用 `finish()`、保存状态、通知系统），这是 Android 推荐的退出方式
   - `exit(code)`：直接终止 C 运行时进程，跳过 Java 层的清理。KrKr2 在调用 Java exit 后紧接着调用 C 的 `exit(code)`，确保即使 Java 层没有正常关闭，进程也会终止

</details>

---

## 下一步

下一节 [02-JNI函数注册与双向调用](02-JNI函数注册与双向调用.md) 将讲解两种 JNI 函数注册方式（静态注册和动态注册），以及如何从 C++ 主动调用 Java 方法。你将学会阅读 KrKr2 中完整的 `krkr2_android.cpp` JNI 桥接代码。
