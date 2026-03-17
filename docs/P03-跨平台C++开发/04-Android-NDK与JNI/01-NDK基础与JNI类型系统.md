# Android NDK 与 JNI 深度指南

> **所属模块：** P03-跨平台 C++ 开发
> **前置知识：** [01-操作系统差异](../01-跨平台基础/01-操作系统差异.md)、[02-预处理器与条件编译](../02-预处理器与条件编译/01-预处理器与条件编译.md)、[03-平台抽象层设计模式](../03-平台抽象层设计模式/01-平台抽象层设计模式.md)
> **预计阅读时间：** 50 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 Android NDK 的作用、组成和与 SDK 的关系
2. 掌握 JNI（Java Native Interface）的完整类型映射和函数注册机制
3. 能够编写 Java 调用 C++ 和 C++ 调用 Java 的双向互操作代码
4. 理解 JNI 中的本地引用/全局引用管理和常见内存陷阱
5. 阅读并分析 KrKr2 项目中完整的 JNI 桥接代码（`krkr2_android.cpp` + `AndroidUtils.cpp`）
6. 独立为项目添加新的 JNI 函数

---

## 什么是 Android NDK

Android NDK（Native Development Kit）是 Google 提供的一套工具集，允许开发者在 Android 应用中使用 C 和 C++ 代码。它与 Android SDK 的关系如下：

```
┌──────────────────────────────────────────────┐
│              Android 应用                     │
├──────────────────────────────────────────────┤
│  Java/Kotlin 层（SDK）                        │
│  ├── Activity、Service、BroadcastReceiver     │
│  ├── UI（View、Layout、Fragment）              │
│  └── 系统 API（文件、网络、传感器）             │
├──────────────────────────────────────────────┤
│  JNI 桥接层                                   │
│  ├── native 方法声明（Java 侧）               │
│  └── JNIEXPORT 函数实现（C++ 侧）             │
├──────────────────────────────────────────────┤
│  Native C/C++ 层（NDK）                       │
│  ├── 应用核心逻辑（引擎、算法）                │
│  ├── 第三方 C/C++ 库（FFmpeg、SDL2）           │
│  └── NDK API（android/log.h、EGL、OpenSL ES） │
├──────────────────────────────────────────────┤
│  Linux 内核                                   │
└──────────────────────────────────────────────┘
```

### NDK 的核心组件

| 组件 | 说明 | KrKr2 中的使用 |
|------|------|----------------|
| **Clang/LLVM 工具链** | 交叉编译 C/C++ 代码到 ARM/x86 | 通过 CMake toolchain 自动调用 |
| **Sysroot** | Android 平台头文件和库 | `<jni.h>`、`<android/log.h>`、`<EGL/egl.h>` |
| **CMake 工具链文件** | `android.toolchain.cmake` | 在 `CMakePresets.json` 中通过 vcpkg 间接使用 |
| **ndk-build** | 旧式构建系统（Makefile） | KrKr2 不使用，已被 CMake 替代 |
| **ABI 支持** | arm64-v8a、armeabi-v7a、x86、x86_64 | KrKr2 支持 arm64-v8a 和 x86_64 |

### NDK 与 SDK 的分工

```
SDK 负责：                        NDK 负责：
├── UI 绘制                       ├── 性能敏感的计算
├── 系统服务访问                   ├── 已有 C/C++ 代码移植
├── 生命周期管理                   ├── 底层硬件访问
├── 权限请求                       ├── 第三方 C/C++ 库集成
└── 应用分发(APK/AAB)              └── 跨平台核心逻辑共享
```

**KrKr2 选择 NDK 的原因：** KrKr2 的引擎核心（TJS2 脚本引擎、渲染管线、音视频解码）全部是 C++ 代码，且需要在 Windows/Linux/macOS 上同时运行。使用 NDK 可以共享 95% 以上的核心代码，Android 端只需写一层薄薄的 JNI 桥接。

---

## JNI 基础：类型系统

JNI（Java Native Interface）是 Java 与 Native 代码之间的桥梁。要正确使用 JNI，首先必须理解它的类型映射。

### 基本类型映射

Java 和 C++ 的基本类型通过 JNI 定义了一一对应关系：

```cpp
// 这些类型定义在 <jni.h> 中
// Java 类型    →  JNI 类型    →  C++ 实际类型    →  JNI 签名
// boolean      →  jboolean   →  uint8_t         →  Z
// byte         →  jbyte      →  int8_t          →  B
// char         →  jchar      →  uint16_t        →  C
// short        →  jshort     →  int16_t         →  S
// int          →  jint       →  int32_t         →  I
// long         →  jlong      →  int64_t         →  J
// float        →  jfloat     →  float           →  F
// double       →  jdouble    →  double          →  D
// void         →  void       →  void            →  V
```

**注意：** `jboolean` 用的是 `uint8_t`，值只有 `JNI_TRUE`(1) 和 `JNI_FALSE`(0)，不要与 C++ 的 `bool` 混淆。

### 引用类型映射

```cpp
// Java 引用类型   →  JNI 类型        →  JNI 签名
// Object          →  jobject         →  Lpackage/ClassName;
// String          →  jstring         →  Ljava/lang/String;
// Class           →  jclass          →  Ljava/lang/Class;
// int[]           →  jintArray       →  [I
// float[]         →  jfloatArray     →  [F
// String[]        →  jobjectArray    →  [Ljava/lang/String;
// Object[]        →  jobjectArray    →  [Lpackage/ClassName;
```

### JNI 方法签名规则

JNI 使用一种特殊的字符串格式描述方法签名：`(参数类型)返回类型`。这是 JNI 中最容易出错的部分。

```
方法签名格式：(参数1参数2参数3...)返回类型

示例：
Java: void foo()                     → ()V
Java: int add(int a, int b)         → (II)I
Java: String getName()              → ()Ljava/lang/String;
Java: void setData(byte[] data)     → ([B)V
Java: long process(String s, int n) → (Ljava/lang/String;I)J
```

KrKr2 中的真实签名示例（来自 `AndroidUtils.cpp`）：

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 51-57 行
// Java 方法: static void updateMemoryInfo()
// JNI 签名: "()V" — 无参数，返回 void
JniHelper::getStaticMethodInfo(methodInfo, KR2ActJavaPath,
                               "updateMemoryInfo", "()V");

// 第 59-63 行
// Java 方法: static long getAvailMemory()
// JNI 签名: "()J" — 无参数，返回 long (J)
JniHelper::getStaticMethodInfo(methodInfo, KR2ActJavaPath,
                               "getAvailMemory", "()J");

// 第 518-521 行
// Java 方法: static void ShowMessageBox(String title, String text, String[] btns)
// JNI 签名: "(Ljava/lang/String;Ljava/lang/String;[Ljava/lang/String;)V"
JniHelper::getStaticMethodInfo(
    methodInfo, "org/tvp/kirikiri2/KR2Activity", "ShowMessageBox",
    "(Ljava/lang/String;Ljava/lang/String;[Ljava/lang/String;)V");
```

---

