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

## JNI 函数注册：两种方式

### 方式一：静态注册（命名约定）

这是 KrKr2 使用的主要方式。C++ 函数名按特定规则命名，JVM 在加载 native 库时自动查找匹配函数。

**命名规则：** `Java_包名_类名_方法名`，其中包名的 `.` 替换为 `_`。

```cpp
// Java 侧声明（KR2Activity.java 第 167-168 行）：
// package org.tvp.kirikiri2;
// private static native void nativeTouchesBegin(final int id, final float x, final float y);

// C++ 侧实现（krkr2_android.cpp 第 107-114 行）：
// 命名规则：Java_org_tvp_kirikiri2_KR2Activity_nativeTouchesBegin
extern "C" {
JNIEXPORT void JNICALL Java_org_tvp_kirikiri2_KR2Activity_nativeTouchesBegin(
    JNIEnv *env, jclass thiz, jint id, jfloat x, jfloat y) {
    intptr_t idlong = id;
    Android_PushEvents([idlong, x, y]() {
        cocos2d::Director::getInstance()->getOpenGLView()->handleTouchesBegin(
            1, (intptr_t *)&idlong, (float *)&x, (float *)&y);
    });
}
}
```

**关键细节：**
- `extern "C"` — 阻止 C++ 名称修饰（name mangling），否则 JVM 找不到函数
- `JNIEXPORT` — 确保函数从共享库中导出（在 Linux/Android 上通常是 `__attribute__((visibility("default")))`）
- `JNICALL` — 指定调用约定（在大多数平台上为空宏）
- 第一个参数总是 `JNIEnv *env` — JNI 环境指针，所有 JNI 操作都通过它进行
- 第二个参数：`static` 方法为 `jclass`（类引用），实例方法为 `jobject`（对象引用）
- 从第三个参数开始对应 Java 方法的参数

### 方式二：动态注册（RegisterNatives）

在 `JNI_OnLoad` 中手动注册函数映射，不需要遵循命名约定：

```cpp
// 动态注册示例 — KrKr2 未使用此方式，但很多项目用它
#include <jni.h>

// C++ 函数可以任意命名
static void myTouchBegin(JNIEnv *env, jclass cls, jint id, jfloat x, jfloat y) {
    // 处理触摸开始
}

static jboolean myKeyAction(JNIEnv *env, jclass cls, jint keyCode, jboolean isPress) {
    // 处理按键
    return JNI_TRUE;
}

// 注册表
static JNINativeMethod gMethods[] = {
    // { Java方法名, JNI签名, C++函数指针 }
    {"nativeTouchesBegin", "(IFF)V",  (void *)myTouchBegin},
    {"nativeKeyAction",    "(IZ)Z",   (void *)myKeyAction},
};

// 在 JNI_OnLoad 中注册
JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env = nullptr;
    if (vm->GetEnv(reinterpret_cast<void **>(&env), JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }

    jclass cls = env->FindClass("org/tvp/kirikiri2/KR2Activity");
    if (cls == nullptr) return JNI_ERR;

    // 批量注册
    int methodCount = sizeof(gMethods) / sizeof(gMethods[0]);
    if (env->RegisterNatives(cls, gMethods, methodCount) < 0) {
        return JNI_ERR;
    }

    env->DeleteLocalRef(cls);
    return JNI_VERSION_1_6;
}
```

### 两种方式对比

| 特性 | 静态注册（命名约定） | 动态注册（RegisterNatives） |
|------|---------------------|---------------------------|
| 配置工作量 | 零，按规则命名即可 | 需要手动编写注册表和 JNI_OnLoad |
| 函数查找速度 | 首次调用时 JVM 搜索符号表 | 注册后直接跳转，略快 |
| 函数名可读性 | 函数名很长（含完整包路径） | 函数名可以自由命名 |
| 灵活性 | 低，包名/类名改变需修改函数名 | 高，只需改注册表 |
| KrKr2 使用 | ✅ 主要使用 | 未使用 |

---

## Java 调用 C++（下行调用）

这是最常见的 JNI 调用方向。Java 声明 `native` 方法，C++ 提供实现。

### KrKr2 的完整下行调用链路

以触摸事件为例，展示从 Java UI 到 C++ 引擎的完整链路：

```
Java: KR2GLSurfaceView.onTouchEvent()
  └─→ Java: KR2Activity.nativeTouchesBegin(id, x, y)     [native 方法]
      └─→ JNI: Java_org_tvp_kirikiri2_KR2Activity_nativeTouchesBegin()
          └─→ C++: Android_PushEvents(lambda)              [投递到引擎线程]
              └─→ C++: cocos2d::GLView::handleTouchesBegin() [Cocos2d 处理]
```

**为什么需要 `Android_PushEvents`？** 因为 JNI 调用发生在 Android UI 线程，而 Cocos2d-x 的渲染和事件处理在 GL 线程。直接在 JNI 函数中操作 Cocos2d 对象会导致线程安全问题。`Android_PushEvents` 是一个线程安全的事件队列：

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 659-690 行
// 无锁事件队列实现 — 使用 atomic 的 compare_exchange_weak
struct _eventQueueNode {
    std::function<void()> func;  // 待执行的回调
    _eventQueueNode *prev;       // 链表前驱
    _eventQueueNode *next;       // 链表后继
};

static std::atomic<_eventQueueNode *> _lastQueuedEvents(nullptr);

// 生产者：JNI 线程调用，将事件入队
void Android_PushEvents(const std::function<void()> &func) {
    _eventQueueNode *node = new _eventQueueNode;
    node->func = func;
    node->next = nullptr;
    node->prev = nullptr;
    // CAS 操作：原子地将新节点插入队列头部
    while (!_lastQueuedEvents.compare_exchange_weak(node->prev, node)) {
    }
}

// 消费者：GL 线程每帧调用（通过 Cocos2d Scheduler）
static void _processEvents(float) {
    _eventQueueNode *q = _lastQueuedEvents.exchange(nullptr);
    if (q) {
        q->next = nullptr;
        while (q->prev) {          // 反转链表（因为入队是头插法）
            q->prev->next = q;
            q = q->prev;
        }
    }
    while (q) {                    // 按入队顺序执行所有事件
        q->func();
        _eventQueueNode *nq = q->next;
        delete q;
        q = nq;
    }
}
```

### KrKr2 所有 native 方法一览

以下是 `KR2Activity.java`（第 130-187 行）中声明的所有 native 方法，以及它们在 C++ 侧的实现位置：

| Java 声明 | C++ 实现文件:行号 | 功能 |
|-----------|------------------|------|
| `native void onMessageBoxOK(int)` | `krkr2_android.cpp:91` | 消息框按钮回调 |
| `native void onMessageBoxText(String)` | `krkr2_android.cpp:97` | 消息框文本回调 |
| `native void initDump(String)` | `krkr2_android.cpp:80` | 初始化 Breakpad 崩溃转储 |
| `native void nativeOnLowMemory()` | `krkr2_android.cpp:386` | 低内存警告 |
| `native void nativeTouchesBegin(int, float, float)` | `krkr2_android.cpp:107` | 触摸开始 |
| `native void nativeTouchesEnd(int, float, float)` | `krkr2_android.cpp:116` | 触摸结束 |
| `native void nativeTouchesMove(int[], float[], float[])` | `krkr2_android.cpp:125` | 触摸移动 |
| `native void nativeTouchesCancel(int[], float[], float[])` | `krkr2_android.cpp:173` | 触摸取消 |
| `native boolean nativeKeyAction(int, boolean)` | `krkr2_android.cpp:232` | 按键事件 |
| `native void nativeCharInput(int)` | `krkr2_android.cpp:309` | 字符输入 |
| `native void nativeCommitText(String, int)` | `krkr2_android.cpp:318` | 输入法提交文本 |
| `native void nativeInsertText(String)` | `krkr2_android.cpp:278` | 插入文本 |
| `native void nativeDeleteBackward()` | `krkr2_android.cpp:300` | 删除字符 |
| `native void nativeHoverMoved(float, float)` | `krkr2_android.cpp:339` | 鼠标悬停移动 |
| `native void nativeMouseScrolled(float)` | `krkr2_android.cpp:363` | 鼠标滚轮 |

---

## C++ 调用 Java（上行调用）

反向调用更复杂——C++ 需要通过 JNI 反射机制查找 Java 类和方法，然后调用。

### 原始 JNI API 方式

```cpp
// 标准 JNI 流程：C++ 调用 Java 的 static void updateMemoryInfo()
void callUpdateMemoryInfo(JNIEnv *env) {
    // 步骤 1：查找类（参数是 JNI 格式的全限定名，用 / 而非 .）
    jclass cls = env->FindClass("org/tvp/kirikiri2/KR2Activity");
    if (cls == nullptr) {
        // 类未找到，可能是类名拼错或 ClassLoader 问题
        return;
    }

    // 步骤 2：查找方法 ID（需要方法名和 JNI 签名）
    jmethodID mid = env->GetStaticMethodID(cls, "updateMemoryInfo", "()V");
    if (mid == nullptr) {
        // 方法未找到，检查签名是否正确
        env->DeleteLocalRef(cls);
        return;
    }

    // 步骤 3：调用方法（根据返回类型选择不同的 Call 函数）
    env->CallStaticVoidMethod(cls, mid);
    // 其他返回类型：
    //   env->CallStaticIntMethod(cls, mid, args...);      // 返回 int
    //   env->CallStaticLongMethod(cls, mid, args...);     // 返回 long
    //   env->CallStaticObjectMethod(cls, mid, args...);   // 返回 Object
    //   env->CallStaticBooleanMethod(cls, mid, args...);  // 返回 boolean

    // 步骤 4：清理本地引用
    env->DeleteLocalRef(cls);
}
```

### Cocos2d-x JniHelper 封装

KrKr2 使用 Cocos2d-x 提供的 `JniHelper` 简化上行调用。`JniHelper` 封装了 `FindClass` + `GetMethodID` 的流程，减少样板代码：

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 309-318 行
// 使用 JniHelper 调用 Java 的 GetVersion() 方法
std::string TVPGetPackageVersionString() {
    JniMethodInfo methodInfo;
    // getStaticMethodInfo 封装了 FindClass + GetStaticMethodID
    // 参数：输出结构体、类路径、方法名、签名
    if (JniHelper::getStaticMethodInfo(methodInfo, KR2ActJavaPath,
                                       "GetVersion",
                                       "()Ljava/lang/String;")) {
        // CallStaticObjectMethod 返回 jstring（是 jobject 的子类型）
        jstring result = (jstring)methodInfo.env->CallStaticObjectMethod(
            methodInfo.classID, methodInfo.methodID);
        // jstring2string 封装了 GetStringUTFChars + ReleaseStringUTFChars
        std::string ret = JniHelper::jstring2string(result);
        // 必须手动释放本地引用
        methodInfo.env->DeleteLocalRef(result);
        methodInfo.env->DeleteLocalRef(methodInfo.classID);
        return ret;
    }
    return "";
}
```

### 实例方法调用（非 static）

调用非 static 的 Java 方法需要一个对象实例：

```cpp
// AndroidUtils.cpp 第 347-357 行
// 调用 Android Context.getFilesDir() 获取内部存储路径
static std::string GetInternalStoragePath() {
    // 先获取 KR2Activity 的单例实例
    jobject sInstance = GetKR2ActInstance();
    JniMethodInfo methodInfo;
    // getMethodInfo（非 static 版本）
    if (!JniHelper::getMethodInfo(methodInfo,
                                  "android/content/ContextWrapper",
                                  "getFilesDir",
                                  "()Ljava/io/File;")) {
        return "";
    }
    // CallObjectMethod（非 CallStaticObjectMethod）
    jobject FileObj = methodInfo.env->CallObjectMethod(
        sInstance, methodInfo.methodID);
    return File_getAbsolutePath(FileObj);
}
```

### 字段访问

JNI 也可以读取 Java 对象的字段：

```cpp
// AndroidUtils.cpp 第 200-209 行
// 读取 ApplicationInfo.sourceDir 字段
static std::string GetApkStoragePath() {
    JniMethodInfo methodInfo;
    jobject sInstance = GetKR2ActInstance();
    // 获取 ApplicationInfo 对象
    if (!JniHelper::getMethodInfo(methodInfo, "android/content/Context",
                                  "getApplicationInfo",
                                  "()Landroid/content/pm/ApplicationInfo;")) {
        methodInfo.env->DeleteLocalRef(sInstance);
        return "";
    }
    jobject ApplicationInfo = methodInfo.env->CallObjectMethod(
        sInstance, methodInfo.methodID);
    // 查找 ApplicationInfo 类
    jclass clsAppInfo = methodInfo.env->FindClass(
        "android/content/pm/ApplicationInfo");
    // 获取字段 ID — 注意签名是字段类型，不是方法签名
    jfieldID id_sourceDir = methodInfo.env->GetFieldID(
        clsAppInfo, "sourceDir", "Ljava/lang/String;");
    methodInfo.env->DeleteLocalRef(sInstance);
    // 读取字段值
    return JniHelper::jstring2string(
        (jstring)methodInfo.env->GetObjectField(ApplicationInfo, id_sourceDir));
}
```

---

## JNI 引用管理（关键！）

JNI 的引用管理是最容易出 Bug 的部分。理解本地引用、全局引用和弱全局引用的区别至关重要。

### 本地引用（Local Reference）

```cpp
// 本地引用在以下情况自动释放：
// 1. native 方法返回时（栈帧销毁）
// 2. 手动调用 DeleteLocalRef

void Java_com_example_MyClass_myMethod(JNIEnv *env, jobject thiz) {
    jclass cls = env->FindClass("java/lang/String");  // 本地引用 1
    jstring str = env->NewStringUTF("hello");          // 本地引用 2
    jobject obj = env->NewObject(cls, ...);             // 本地引用 3

    // 方法返回时，cls、str、obj 自动释放
    // 但在循环中创建大量本地引用时必须手动释放！
}

// ⚠️ 常见陷阱：循环中的本地引用泄漏
void processItems(JNIEnv *env, jobjectArray items) {
    int count = env->GetArrayLength(items);
    for (int i = 0; i < count; ++i) {
        jobject item = env->GetObjectArrayElement(items, i);
        // 处理 item ...

        // ❌ 不释放——如果 count 很大，会耗尽本地引用表（默认 512 个）
        // ✅ 必须手动释放
        env->DeleteLocalRef(item);
    }
}
```

### 全局引用（Global Reference）

```cpp
// 全局引用在显式调用 DeleteGlobalRef 前一直有效
// 适用于需要跨多次 JNI 调用保持的引用

static jclass g_StringClass = nullptr;  // 全局引用缓存

JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env;
    vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6);

    // 将本地引用转为全局引用
    jclass localCls = env->FindClass("java/lang/String");
    g_StringClass = (jclass)env->NewGlobalRef(localCls);
    env->DeleteLocalRef(localCls);  // 原始本地引用不再需要

    return JNI_VERSION_1_6;
}

// 在 JNI_OnUnload 中释放
void JNI_OnUnload(JavaVM *vm, void *reserved) {
    JNIEnv *env;
    vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6);
    if (g_StringClass) {
        env->DeleteGlobalRef(g_StringClass);
        g_StringClass = nullptr;
    }
}
```

### KrKr2 中的引用管理实践

```cpp
// AndroidUtils.cpp 第 515-549 行
// TVPShowSimpleMessageBox — 注意引用清理的严谨性
int TVPShowSimpleMessageBox(const char *pszText, const char *pszTitle,
                            unsigned int nButton, const char **btnText) {
    JniMethodInfo methodInfo;
    if (JniHelper::getStaticMethodInfo(
            methodInfo, "org/tvp/kirikiri2/KR2Activity", "ShowMessageBox",
            "(Ljava/lang/String;Ljava/lang/String;[Ljava/lang/String;)V")) {

        // 创建 Java 字符串（本地引用）
        jstring jstrTitle = methodInfo.env->NewStringUTF(pszTitle);
        jstring jstrText = methodInfo.env->NewStringUTF(pszText);

        // 创建 String 数组
        jclass strcls = methodInfo.env->FindClass("java/lang/String");
        jobjectArray btns = methodInfo.env->NewObjectArray(nButton, strcls, nullptr);
        for (unsigned int i = 0; i < nButton; ++i) {
            jstring jstrBtn = methodInfo.env->NewStringUTF(btnText[i]);
            methodInfo.env->SetObjectArrayElement(btns, i, jstrBtn);
            methodInfo.env->DeleteLocalRef(jstrBtn);  // ✅ 循环内立即释放
        }

        // 调用 Java 方法
        methodInfo.env->CallStaticVoidMethod(
            methodInfo.classID, methodInfo.methodID, jstrTitle, jstrText, btns);

        // ✅ 逐个释放所有本地引用
        methodInfo.env->DeleteLocalRef(jstrTitle);
        methodInfo.env->DeleteLocalRef(jstrText);
        methodInfo.env->DeleteLocalRef(btns);
        methodInfo.env->DeleteLocalRef(methodInfo.classID);

        // 等待用户点击按钮（条件变量同步）
        std::unique_lock<std::mutex> lk(MessageBoxLock);
        while (MsgBoxRet == -2) {
            MessageBoxCond.wait_for(lk, std::chrono::milliseconds(200));
        }
        return MsgBoxRet;
    }
    return -1;
}
```

---

## JNI 字符串处理

Java 的 `String` 使用 UTF-16 编码，C++ 通常使用 UTF-8。JNI 提供了两套转换函数：

### Modified UTF-8 vs 标准 UTF-8

```cpp
// GetStringUTFChars — 返回 Modified UTF-8 编码
// Modified UTF-8 与标准 UTF-8 的区别：
// 1. 空字符 '\0' 编码为 0xC0 0x80（两字节），而非 0x00
// 2. 补充平面字符（U+10000+）编码为两个 3 字节序列（代理对）
//    而非标准 UTF-8 的单个 4 字节序列
// 对于 ASCII 和 BMP 字符（日常使用 99%+ 的场景），两者完全相同

// KrKr2 的字符串处理模式（krkr2_android.cpp 第 280-297 行）：
JNIEXPORT void JNICALL Java_org_tvp_kirikiri2_KR2Activity_nativeInsertText(
    JNIEnv *env, jclass cls, jstring text) {
    // 步骤 1：获取 C 字符串指针（JVM 可能复制，也可能直接返回内部指针）
    const char *pszText = env->GetStringUTFChars(text, nullptr);
    if (pszText && *pszText) {
        std::string str = pszText;  // 复制到 std::string
        Android_PushEvents([str]() {
            cocos2d::IMEDispatcher::sharedDispatcher()->dispatchInsertText(
                str.c_str(), str.length());
        });
    }
    // 步骤 2：必须释放！否则内存泄漏（或阻止 GC 回收原始 String）
    env->ReleaseStringUTFChars(text, pszText);
}
```

### 安全的字符串处理封装

```cpp
// 推荐的 RAII 封装，避免忘记 Release
class JniString {
    JNIEnv *env_;
    jstring jstr_;
    const char *utf8_;

public:
    JniString(JNIEnv *env, jstring jstr)
        : env_(env), jstr_(jstr),
          utf8_(jstr ? env->GetStringUTFChars(jstr, nullptr) : nullptr) {}

    ~JniString() {
        if (utf8_) env_->ReleaseStringUTFChars(jstr_, utf8_);
    }

    // 禁止拷贝
    JniString(const JniString &) = delete;
    JniString &operator=(const JniString &) = delete;

    const char *c_str() const { return utf8_ ? utf8_ : ""; }
    bool empty() const { return !utf8_ || !*utf8_; }
    operator std::string() const { return c_str(); }
};

// 使用示例
void processText(JNIEnv *env, jstring text) {
    JniString str(env, text);
    if (!str.empty()) {
        spdlog::info("收到文本: {}", str.c_str());
    }
    // 析构时自动 ReleaseStringUTFChars
}
```

---

## KrKr2 Android 启动流程

理解 KrKr2 在 Android 上的完整启动链路，有助于理解 JNI 的实际应用场景。

```
┌───────────────────────────────────────────────┐
│ 1. Android 系统启动 KR2Activity               │
│    KR2Activity.onCreate()                     │
│    ├── super.onCreate() → Cocos2dxActivity    │
│    ├── sInstance = this                       │
│    └── initDump(path)  ──────────────────┐    │
├──────────────────────────────────────────┤    │
│ 2. Cocos2dxActivity 加载 native 库        │    │
│    System.loadLibrary("krkr2")           │    │
│    └── 触发 cocos_android_app_init() ────┤    │
├──────────────────────────────────────────┤    │
│ 3. cocos_android_app_init (JNI)          │◄───┘
│    krkr2_android.cpp 第 37-67 行          │
│    ├── 初始化 spdlog (core/tjs2/plugin)  │
│    ├── dlopen("libSDL2.so")              │
│    │   └── 调用 SDL2 的 JNI_OnLoad       │
│    └── 创建 TVPAppDelegate               │
├──────────────────────────────────────────┤
│ 4. Cocos2d-x 引擎初始化                   │
│    Director::init()                       │
│    └── TVPAppDelegate::run()             │
│        └── applicationDidFinishLaunching │
│            ├── 初始化渲染管线              │
│            ├── 初始化 TJS2 引擎           │
│            └── 加载游戏脚本               │
├──────────────────────────────────────────┤
│ 5. 运行时 JNI 交互                        │
│    ├── 触摸/按键 → nativeXxx → C++ 引擎  │
│    ├── C++ → JniHelper → Java 系统 API   │
│    └── 低内存 → nativeOnLowMemory        │
└───────────────────────────────────────────┘
```

**启动入口分析**（`krkr2_android.cpp` 第 37-67 行）：

```cpp
// Android 的入口不是 main()，而是这个被 Cocos2d-x 回调的函数
[[maybe_unused]] void cocos_android_app_init(JNIEnv *env) {
    // 1. 初始化日志系统 — 使用 Android Logcat 作为后端
    spdlog::set_pattern("%v");
    spdlog::set_level(spdlog::level::debug);
    static auto core_logger =
        spdlog::android_logger_mt("core", "KrKr2NativeCore");
    // ... tjs2_logger, plugin_logger
    spdlog::set_default_logger(core_logger);

    // 2. 动态加载 SDL2 并调用其 JNI_OnLoad
    // 为什么用 dlopen 而不是直接链接？
    // 因为 SDL2 需要在 JNI_OnLoad 中注册自己的 native 方法
    // 但 krkr2 库先于 SDL2 加载，所以需要手动触发
    JavaVM *vm{};
    env->GetJavaVM(&vm);
    void *handle = dlopen("libSDL2.so", RTLD_LAZY);
    if (handle) {
        typedef jint (*JNI_OnLoad)(JavaVM *, void *);
        void *sdl2Init = dlsym(handle, "JNI_OnLoad");
        if (!sdl2Init ||
            ((JNI_OnLoad)sdl2Init)(vm, nullptr) != JNI_VERSION_1_4) {
            spdlog::critical("invoke libSDL2.so JNI_OnLoad method failed");
        }
    }

    // 3. 创建应用委托 — 引擎核心从这里开始
    static std::unique_ptr<TVPAppDelegate> pAppDelegate =
        std::make_unique<TVPAppDelegate>();
}
```

---

## 按键映射：Android KeyCode → Cocos2d KeyCode

KrKr2 的按键处理展示了 JNI 中枚举值映射的典型模式：

```cpp
// krkr2_android.cpp 第 221-276 行
// Android 键码定义（对应 android.view.KeyEvent 的常量）
#define KEYCODE_BACK       0x04  // 返回键
#define KEYCODE_MENU       0x52  // 菜单键
#define KEYCODE_DPAD_UP    0x13  // 方向上
#define KEYCODE_DPAD_DOWN  0x14  // 方向下
#define KEYCODE_DPAD_LEFT  0x15  // 方向左
#define KEYCODE_DPAD_RIGHT 0x16  // 方向右
#define KEYCODE_ENTER      0x42  // 确认
#define KEYCODE_PLAY       0x7e  // 播放
#define KEYCODE_DPAD_CENTER 0x17 // 方向中心
#define KEYCODE_DEL        0x43  // 删除

JNIEXPORT jboolean JNICALL Java_org_tvp_kirikiri2_KR2Activity_nativeKeyAction(
    JNIEnv *env, jclass cls, jint keyCode, jboolean isPress) {
    cocos2d::EventKeyboard::KeyCode pKeyCode;
    switch (keyCode) {
        case KEYCODE_BACK:
            // 返回键映射为 ESC — 视觉小说中通常用于打开菜单或退出
            pKeyCode = cocos2d::EventKeyboard::KeyCode::KEY_ESCAPE;
            break;
        case KEYCODE_MENU:
            pKeyCode = cocos2d::EventKeyboard::KeyCode::KEY_MENU;
            break;
        // ... 其他映射
        default:
            return JNI_FALSE;  // 未处理的键码
    }

    // 将按键事件投递到引擎线程
    Android_PushEvents([pKeyCode, isPress]() {
        cocos2d::EventKeyboard event(pKeyCode, isPress);
        cocos2d::Director::getInstance()
            ->getEventDispatcher()
            ->dispatchEvent(&event);
    });
    return JNI_TRUE;  // 已处理
}
```

Java 侧的按键拦截（`KR2Activity.java` 第 210-235 行）：

```java
// KR2GLSurfaceView 重写了 onKeyDown 和 onKeyUp
@Override
public boolean onKeyDown(final int pKeyCode, final KeyEvent pKeyEvent) {
    return switch (pKeyCode) {
        case KeyEvent.KEYCODE_BACK, KeyEvent.KEYCODE_MENU,
             KeyEvent.KEYCODE_DPAD_LEFT, KeyEvent.KEYCODE_DPAD_RIGHT,
             KeyEvent.KEYCODE_DPAD_UP, KeyEvent.KEYCODE_DPAD_DOWN,
             KeyEvent.KEYCODE_ENTER, KeyEvent.KEYCODE_MEDIA_PLAY_PAUSE,
             KeyEvent.KEYCODE_DPAD_CENTER -> {
            nativeKeyAction(pKeyCode, true);  // 调用 native 方法
            yield true;  // 表示已消费该事件
        }
        default -> super.onKeyDown(pKeyCode, pKeyEvent);  // 未处理交给父类
    };
}
```

---

## 动手实践

### 实践 1：编写一个简单的 JNI 函数

创建一个 Java 类调用 C++ 函数计算两个整数之和：

**Java 侧：**

```java
// Calculator.java
package com.example.jnidemo;

public class Calculator {
    // 加载 native 库
    static {
        System.loadLibrary("calculator");
    }

    // 声明 native 方法
    public static native int add(int a, int b);
    public static native String multiply(int a, int b);

    public static void main(String[] args) {
        System.out.println("3 + 5 = " + add(3, 5));
        System.out.println("3 * 5 = " + multiply(3, 5));
    }
}
```

**C++ 侧：**

```cpp
// calculator.cpp
#include <jni.h>
#include <string>
#include <sstream>

extern "C" {

// 对应 Calculator.add(int, int)
JNIEXPORT jint JNICALL Java_com_example_jnidemo_Calculator_add(
    JNIEnv *env, jclass cls, jint a, jint b) {
    return a + b;  // jint 与 int32_t 兼容，直接运算
}

// 对应 Calculator.multiply(int, int)
JNIEXPORT jstring JNICALL Java_com_example_jnidemo_Calculator_multiply(
    JNIEnv *env, jclass cls, jint a, jint b) {
    std::ostringstream oss;
    oss << a << " × " << b << " = " << (a * b);
    // NewStringUTF 创建一个 Java String 并返回
    return env->NewStringUTF(oss.str().c_str());
}

}
```

**CMakeLists.txt：**

```cmake
cmake_minimum_required(VERSION 3.22)
project(jni_demo)

# Android 构建时 NDK 自动设置工具链
add_library(calculator SHARED calculator.cpp)

# 链接 Android 日志库（可选，用于 __android_log_print）
find_library(log-lib log)
target_link_libraries(calculator ${log-lib})
```

### 实践 2：C++ 调用 Java 获取 Android SDK 版本

```cpp
// 获取 Android SDK 版本号（对照 AndroidUtils.cpp 第 725-734 行）
#include <jni.h>

// 需要 JNIEnv*，在 JNI 函数内可直接使用，其他场景需通过 JavaVM 获取
static int GetAndroidSDKVersion(JNIEnv *env) {
    // 1. 查找 android.os.Build.VERSION 类
    //    注意内部类用 $ 分隔：Build$VERSION
    jclass cls = env->FindClass("android/os/Build$VERSION");
    if (cls == nullptr) return -1;

    // 2. 获取静态字段 SDK_INT 的 ID
    //    字段签名只有类型，没有括号：int → "I"
    jfieldID fieldId = env->GetStaticFieldID(cls, "SDK_INT", "I");
    if (fieldId == nullptr) {
        env->DeleteLocalRef(cls);
        return -1;
    }

    // 3. 读取字段值
    jint sdkVersion = env->GetStaticIntField(cls, fieldId);
    env->DeleteLocalRef(cls);

    return static_cast<int>(sdkVersion);
}
```

---

## 常见错误及解决方案

### 错误 1：JNI 函数名拼错导致 UnsatisfiedLinkError

```
java.lang.UnsatisfiedLinkError: No implementation found for
    void org.tvp.kirikiri2.KR2Activity.nativeTouchesBegin(int, float, float)
```

**原因：** C++ 函数名与 Java 方法声明不匹配。可能是：
- 包名中的 `.` 没有替换为 `_`
- 类名拼写错误
- 忘记 `extern "C"` 导致 C++ 名称修饰

**排查方法：**

```bash
# 检查 .so 文件导出的符号
# 在 Android NDK 工具链中：
$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-nm -D libkrkr2.so | grep nativeTouchesBegin

# 预期输出（T = 已导出的函数）：
# 0000abcd T Java_org_tvp_kirikiri2_KR2Activity_nativeTouchesBegin

# 如果没有输出，检查：
# 1. 是否有 extern "C"
# 2. 是否有 JNIEXPORT
# 3. 函数名拼写是否完全匹配
```

### 错误 2：在非 JNI 线程中使用 JNIEnv

```
FATAL EXCEPTION: Thread-42
JNI DETECTED ERROR IN APPLICATION:
    using a stale JNIEnv from a different thread
```

**原因：** `JNIEnv *` 是线程专属的，不能跨线程使用。

**解决方案：**

```cpp
// 错误 ❌：在新线程中使用主线程的 JNIEnv
void startWorkerThread(JNIEnv *env) {
    std::thread([env]() {
        // env 属于主线程，在此使用会崩溃！
        env->FindClass("...");
    }).detach();
}

// 正确 ✅：通过 JavaVM 在新线程中获取 JNIEnv
static JavaVM *g_vm = nullptr;

JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    g_vm = vm;  // 保存 JavaVM 指针（全局唯一，线程安全）
    return JNI_VERSION_1_6;
}

void startWorkerThread() {
    std::thread([]() {
        JNIEnv *env = nullptr;
        // 将当前线程附加到 JVM
        bool needDetach = false;
        if (g_vm->GetEnv(reinterpret_cast<void **>(&env),
                         JNI_VERSION_1_6) == JNI_EDETACHED) {
            g_vm->AttachCurrentThread(&env, nullptr);
            needDetach = true;
        }

        // 现在可以安全使用 env
        jclass cls = env->FindClass("...");
        // ...

        // 用完后分离线程
        if (needDetach) {
            g_vm->DetachCurrentThread();
        }
    }).detach();
}
```

### 错误 3：本地引用表溢出

```
JNI ERROR (app bug): local reference table overflow (max=512)
```

**原因：** 在循环中创建大量 JNI 对象但未释放本地引用。

**解决方案：**

```cpp
// 方案 1：在循环体内手动 DeleteLocalRef（推荐）
for (int i = 0; i < 10000; ++i) {
    jstring str = env->NewStringUTF("test");
    // ... 使用 str
    env->DeleteLocalRef(str);  // ✅ 用完立即释放
}

// 方案 2：使用 PushLocalFrame / PopLocalFrame
env->PushLocalFrame(16);  // 预留 16 个本地引用槽位
for (int i = 0; i < 10000; ++i) {
    jstring str = env->NewStringUTF("test");
    // ... 使用 str
    // 不需要手动 Delete，PopLocalFrame 会批量释放

    if ((i + 1) % 10 == 0) {
        env->PopLocalFrame(nullptr);   // 释放本帧所有引用
        env->PushLocalFrame(16);       // 开启新帧
    }
}
env->PopLocalFrame(nullptr);
```

---

## 对照项目源码

KrKr2 的 JNI 代码主要分布在两个文件中：

### 文件 1：`platforms/android/cpp/krkr2_android.cpp`（394 行）

**职责：** Java → C++ 的下行调用入口点（所有 `native` 方法的实现）

| 行号范围 | 功能 | 关键技术 |
|----------|------|----------|
| 37-67 | `cocos_android_app_init` 启动入口 | dlopen/dlsym 加载 SDL2 |
| 80-89 | `initDump` 初始化崩溃转储 | google_breakpad MinidumpDescriptor |
| 91-105 | `onMessageBoxOK/Text` 消息框回调 | condition_variable 线程同步 |
| 107-171 | 触摸事件处理 | JNI 数组操作、事件队列 |
| 221-276 | 按键映射与派发 | Android KeyCode → Cocos2d KeyCode |
| 278-327 | 文本输入处理 | GetStringUTFChars 字符串转换 |
| 330-335 | 获取配置项 | GlobalConfigManager 集成 |
| 339-393 | 鼠标/滚轮事件 | 坐标系转换（屏幕→Cocos2d） |

### 文件 2：`cpp/core/environ/android/AndroidUtils.cpp`（1002 行）

**职责：** C++ → Java 的上行调用（PAL 函数的 Android 实现）

| 行号范围 | 功能 | 关键技术 |
|----------|------|----------|
| 42-78 | 内存信息查询 | JniHelper + 频率节流（3秒缓存） |
| 94-174 | 设备 ID 获取 | 复杂的 JNI 反射调用链 |
| 191-221 | APK/包名查询 | Context.getApplicationInfo 字段访问 |
| 292-307 | 系统语言查询 | java.util.Locale 反射 |
| 347-357 | 内部存储路径 | ContextWrapper.getFilesDir |
| 375-463 | 外部存储/驱动器路径 | getExternalFilesDirs + /proc/mounts 解析 |
| 507-617 | 消息框/输入框 | NewObjectArray + condition_variable 同步 |
| 659-690 | 无锁事件队列 | atomic compare_exchange_weak |
| 695-712 | 启动检查 | Breakpad dump + Scheduler 注册 |
| 810-824 | 文件夹创建 | JNI 调用 Java Storage Access Framework |
| 826-885 | 数据写入 | POSIX fwrite 回退到 Java WriteFile |
| 939-969 | 文件删除/重命名 | JNI 调用 Java 文件操作 |

---

## 本节小结

- **Android NDK** 允许在 Android 上运行 C/C++ 代码，KrKr2 用它共享 95%+ 的跨平台引擎核心
- **JNI 类型映射** 是基础中的基础：基本类型一一对应，引用类型需要手动管理
- **JNI 方法签名** 格式为 `(参数类型)返回类型`，是调用 Java 方法的必备知识
- **静态注册**（命名约定）简单直接，KrKr2 全部使用此方式；**动态注册**（RegisterNatives）更灵活
- **下行调用**（Java→C++）：Java 声明 `native` 方法，C++ 实现 `JNIEXPORT` 函数
- **上行调用**（C++→Java）：通过 `FindClass` + `GetMethodID` + `CallXxxMethod` 三步走
- **引用管理**是 JNI 最易出错的部分：循环中必须手动 `DeleteLocalRef`，跨调用的引用用 `NewGlobalRef`
- **线程安全**：`JNIEnv*` 线程专属，跨线程需 `AttachCurrentThread`；KrKr2 用 `Android_PushEvents` 事件队列解决
- **KrKr2 的 JNI 架构**：两个核心文件分工明确——`krkr2_android.cpp` 处理下行调用，`AndroidUtils.cpp` 处理上行调用

---

## 练习题与答案

### 题目 1：JNI 方法签名

写出以下 Java 方法的 JNI 签名：

```java
public static byte[] compress(String data, int level);
public boolean openFile(String path, String mode);
public static void log(String tag, String msg, int priority);
```

<details>
<summary>查看答案</summary>

```
// static byte[] compress(String data, int level)
// 参数：String → Ljava/lang/String;  int → I
// 返回：byte[] → [B
// 签名：(Ljava/lang/String;I)[B

// boolean openFile(String path, String mode)
// 参数：String → Ljava/lang/String;  String → Ljava/lang/String;
// 返回：boolean → Z
// 签名：(Ljava/lang/String;Ljava/lang/String;)Z

// static void log(String tag, String msg, int priority)
// 参数：String → Ljava/lang/String;  String → Ljava/lang/String;  int → I
// 返回：void → V
// 签名：(Ljava/lang/String;Ljava/lang/String;I)V
```

**记忆技巧：**
- 基本类型用首字母大写：I(int), J(long), Z(boolean), B(byte), F(float), D(double)
- 对象类型：`L全限定名;`（注意分号不能少！）
- 数组：`[` + 元素类型
- long 用 J（因为 L 被对象类型占了）
- boolean 用 Z（因为 B 被 byte 占了）

</details>

### 题目 2：引用泄漏排查

以下代码有一处引用泄漏，找出并修复：

```cpp
jobjectArray getFileList(JNIEnv *env, jstring dirPath) {
    const char *path = env->GetStringUTFChars(dirPath, nullptr);
    if (!path) return nullptr;

    std::vector<std::string> files = listDirectory(path);
    // 忘记 ReleaseStringUTFChars — 泄漏！

    jclass strCls = env->FindClass("java/lang/String");
    jobjectArray result = env->NewObjectArray(files.size(), strCls, nullptr);

    for (size_t i = 0; i < files.size(); ++i) {
        jstring jstr = env->NewStringUTF(files[i].c_str());
        env->SetObjectArrayElement(result, i, jstr);
        // 忘记 DeleteLocalRef(jstr) — 如果文件很多会溢出！
    }

    return result;
}
```

<details>
<summary>查看答案</summary>

修正后的代码：

```cpp
jobjectArray getFileList(JNIEnv *env, jstring dirPath) {
    const char *path = env->GetStringUTFChars(dirPath, nullptr);
    if (!path) return nullptr;

    std::vector<std::string> files = listDirectory(path);

    // ✅ 修复 1：释放字符串
    env->ReleaseStringUTFChars(dirPath, path);

    jclass strCls = env->FindClass("java/lang/String");
    jobjectArray result = env->NewObjectArray(
        static_cast<jsize>(files.size()), strCls, nullptr);

    for (size_t i = 0; i < files.size(); ++i) {
        jstring jstr = env->NewStringUTF(files[i].c_str());
        env->SetObjectArrayElement(result, i, jstr);
        // ✅ 修复 2：循环内释放本地引用
        env->DeleteLocalRef(jstr);
    }

    // ✅ 可选：释放 strCls（函数返回时会自动释放，但显式释放是好习惯）
    env->DeleteLocalRef(strCls);

    return result;
}
```

**两处泄漏：**
1. `GetStringUTFChars` 后必须调用 `ReleaseStringUTFChars`，否则 JVM 无法回收原始 `jstring`
2. 循环中的 `NewStringUTF` 创建本地引用但不释放。如果 `files` 有几百个元素，就会超过 512 的本地引用表限制

</details>

### 题目 3：为 KrKr2 添加新的 JNI 函数

假设需要添加一个 native 方法 `nativeSetScreenBrightness(float brightness)`，让 C++ 引擎能控制屏幕亮度。请写出：
1. Java 侧的 native 声明
2. C++ 侧的 JNI 函数实现（使用 Android WindowManager API）

<details>
<summary>查看答案</summary>

**Java 侧**（在 `KR2Activity.java` 中添加）：

```java
// native 声明
public static native void nativeSetScreenBrightness(float brightness);

// Java 端的实际亮度设置方法（供 C++ 回调）
public static void setScreenBrightness(float brightness) {
    if (sInstance == null) return;
    sInstance.runOnUiThread(() -> {
        android.view.WindowManager.LayoutParams lp =
            sInstance.getWindow().getAttributes();
        // brightness 范围 0.0f ~ 1.0f，-1.0f 表示使用系统默认
        lp.screenBrightness = Math.max(0.0f, Math.min(1.0f, brightness));
        sInstance.getWindow().setAttributes(lp);
    });
}
```

**C++ 侧**（在 `krkr2_android.cpp` 的 `extern "C"` 块中添加）：

```cpp
// 方案 A：直接回调 Java 方法设置亮度
JNIEXPORT void JNICALL
Java_org_tvp_kirikiri2_KR2Activity_nativeSetScreenBrightness(
    JNIEnv *env, jclass cls, jfloat brightness) {
    // 将亮度设置投递到引擎线程，再回调 Java
    float b = brightness;
    Android_PushEvents([b]() {
        JniMethodInfo methodInfo;
        if (JniHelper::getStaticMethodInfo(
                methodInfo, "org/tvp/kirikiri2/KR2Activity",
                "setScreenBrightness", "(F)V")) {
            methodInfo.env->CallStaticVoidMethod(
                methodInfo.classID, methodInfo.methodID, b);
            methodInfo.env->DeleteLocalRef(methodInfo.classID);
        }
    });
}

// 方案 B（更简单）：如果不需要经过引擎线程，直接调用
JNIEXPORT void JNICALL
Java_org_tvp_kirikiri2_KR2Activity_nativeSetScreenBrightness(
    JNIEnv *env, jclass cls, jfloat brightness) {
    // 直接调用 Java 端的 setScreenBrightness
    JniMethodInfo methodInfo;
    if (JniHelper::getStaticMethodInfo(
            methodInfo, "org/tvp/kirikiri2/KR2Activity",
            "setScreenBrightness", "(F)V")) {
        methodInfo.env->CallStaticVoidMethod(
            methodInfo.classID, methodInfo.methodID, brightness);
        methodInfo.env->DeleteLocalRef(methodInfo.classID);
    }
}
```

**说明：** 实际项目中更推荐方案 B，因为亮度设置不涉及引擎状态，无需绕道引擎线程。但如果需要引擎记录当前亮度值或做其他处理，则用方案 A。

</details>

---

## 下一步

下一节 [实战：迷你 PAL](../05-实战-迷你PAL/) 将综合前四节的知识，从零构建一个支持 Windows/Linux/macOS/Android 的迷你平台抽象层（文件操作、线程、日志），亲手体验跨平台 C++ 开发的完整流程。
