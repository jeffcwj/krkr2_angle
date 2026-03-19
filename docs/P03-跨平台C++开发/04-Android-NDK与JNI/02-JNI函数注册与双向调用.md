# JNI 函数注册与双向调用

> **所属模块：** P03-跨平台C++开发
> **前置知识：** [NDK 基础与 JNI 类型系统](./01-NDK基础与JNI类型系统.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 JNI 静态注册（命名约定）和动态注册（`RegisterNatives`）两种方式的工作原理和适用场景
2. 追踪 KrKr2 中一个完整的 Java→C++ 下行调用链路（从 `onTouchEvent` 到 Cocos2d 引擎处理）
3. 理解 `Android_PushEvents` 无锁事件队列的设计——为什么 JNI 调用不能直接操作引擎对象
4. 使用 `FindClass` + `GetMethodID` + `CallXxxMethod` 三步流程从 C++ 调用 Java 方法
5. 使用 Cocos2d-x 的 `JniHelper` 简化上行调用的样板代码
6. 通过 `GetFieldID` + `GetObjectField` 从 C++ 读取 Java 对象的字段

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

## 常见错误及解决方案

### 错误 1：JNI 签名中漏掉分号

```cpp
// ❌ 错误：对象类型签名缺少末尾分号
env->GetStaticMethodID(cls, "myMethod", "(Ljava/lang/String)V");
//                                                         ^ 少了分号！

// ✅ 正确：对象类型签名必须以分号结尾
env->GetStaticMethodID(cls, "myMethod", "(Ljava/lang/String;)V");
//                                                          ^ 必须有分号
```

**后果：** `GetStaticMethodID` 返回 `nullptr`，后续调用 `CallStaticVoidMethod` 传入空 ID 会导致 JVM 崩溃。

**排查工具：** 使用 `javap -s` 命令自动生成 JNI 签名：

```bash
# 编译 Java 类后查看签名
javap -s -classpath . com.example.MyClass
# 输出示例：
#   public static void myMethod(java.lang.String);
#     descriptor: (Ljava/lang/String;)V
```

### 错误 2：FindClass 在非主线程返回 nullptr

```cpp
// 在 C++ 工作线程中：
jclass cls = env->FindClass("org/tvp/kirikiri2/KR2Activity");
// cls 可能为 nullptr！即使类确实存在
```

**原因：** `FindClass` 使用调用线程关联的 `ClassLoader`。通过 `AttachCurrentThread` 附加的线程使用系统 `ClassLoader`，它**不包含应用的类**。

**解决方案：** 在 `JNI_OnLoad` 中（此时使用正确的 `ClassLoader`）缓存 `jclass` 为全局引用：

```cpp
static jclass g_KR2ActivityClass = nullptr;

JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env;
    vm->GetEnv(reinterpret_cast<void **>(&env), JNI_VERSION_1_6);

    // 在主线程中查找并缓存为全局引用
    jclass localRef = env->FindClass("org/tvp/kirikiri2/KR2Activity");
    g_KR2ActivityClass = (jclass)env->NewGlobalRef(localRef);
    env->DeleteLocalRef(localRef);

    return JNI_VERSION_1_6;
}

// 之后任何线程都可以安全使用 g_KR2ActivityClass
```

### 错误 3：混淆 static 方法和实例方法的调用

```cpp
// ❌ 错误：对 static 方法使用 CallVoidMethod（应该用 CallStaticVoidMethod）
env->CallVoidMethod(cls, mid);  // 崩溃！CallVoidMethod 第一个参数应是 jobject

// ✅ 正确：
// static 方法 → GetStaticMethodID + CallStaticVoidMethod
// 实例方法 → GetMethodID + CallVoidMethod(jobject实例, mid, ...)
```

**记忆规则：** 方法名中带 `Static` 的是一对——`GetStaticMethodID` 配 `CallStaticXxxMethod`，`GetMethodID` 配 `CallXxxMethod`。

---

## 本节小结

- **静态注册**按 `Java_包名_类名_方法名` 命名约定，JVM 自动查找。优点是零配置，缺点是函数名很长
- **动态注册**在 `JNI_OnLoad` 中通过 `RegisterNatives` 手动建立映射。优点是函数名自由、查找更快，缺点是需要额外注册代码
- **KrKr2 全部使用静态注册**——简单直接，与 Cocos2d-x 风格一致
- **下行调用**（Java→C++）：Java 声明 `native` 方法，C++ 用 `JNIEXPORT` 导出实现。KrKr2 通过 `Android_PushEvents` 将 JNI 线程的调用安全投递到 GL 线程
- **上行调用**（C++→Java）：三步流程 `FindClass` → `GetMethodID` → `CallXxxMethod`。KrKr2 通过 `JniHelper` 封装减少样板代码
- **字段访问**使用 `GetFieldID` + `GetXxxField`，签名只写字段类型（没有括号）
- **`JNIEnv*` 是线程专属的**——不能跨线程传递。需要通过 `JavaVM::AttachCurrentThread` 获取当前线程的 `JNIEnv*`
- **引用管理**是 JNI 易错点：本地引用必须及时 `DeleteLocalRef`，跨调用保存用 `NewGlobalRef`

---

## 练习题与答案

### 题目 1：写出 JNI 签名

写出以下 Java 方法的 JNI 方法签名（descriptor）：

```java
public static String concat(String a, String b);
public int[] getPixels(int x, int y, int width, int height);
public static void callback(byte[] data, long timestamp, boolean compressed);
```

<details>
<summary>查看答案</summary>

```
// static String concat(String a, String b)
// 参数：String → Ljava/lang/String;  String → Ljava/lang/String;
// 返回：String → Ljava/lang/String;
// 签名：(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;

// int[] getPixels(int x, int y, int width, int height)
// 参数：int → I, int → I, int → I, int → I
// 返回：int[] → [I
// 签名：(IIII)[I

// static void callback(byte[] data, long timestamp, boolean compressed)
// 参数：byte[] → [B, long → J, boolean → Z
// 返回：void → V
// 签名：([BJZ)V
```

**易混淆点：**
- `long` 的签名是 `J`（不是 `L`，因为 `L` 被对象类型占了）
- `boolean` 的签名是 `Z`（不是 `B`，因为 `B` 是 `byte`）
- 数组类型是 `[` + 元素签名，如 `byte[]` → `[B`，`int[]` → `[I`
- 对象类型**必须**以分号结尾：`Ljava/lang/String;`

</details>

### 题目 2：分析 Android_PushEvents 的线程安全性

KrKr2 的 `Android_PushEvents` 使用 `compare_exchange_weak` 实现无锁队列。请回答：
1. 为什么用 `compare_exchange_weak` 而不是 `compare_exchange_strong`？
2. 如果把 `_lastQueuedEvents` 从 `std::atomic` 改成普通指针会怎样？

<details>
<summary>查看答案</summary>

**1. weak vs strong：**

`compare_exchange_weak` 可能在值匹配时仍然失败（称为"伪失败"，spurious failure），但它在某些 CPU 架构（如 ARM，即 Android 的主要架构）上比 `compare_exchange_strong` 更高效。因为 `Android_PushEvents` 已经在 `while` 循环中调用，伪失败只会导致多循环一次，不影响正确性。在高频调用（如每帧多次触摸事件）的场景下，选择 `weak` 版本可以减少 CPU 开销。

**2. 如果不用 atomic：**

去掉 `atomic` 后，多个 JNI 线程同时调用 `Android_PushEvents` 时会发生数据竞争（data race）：
- 两个线程同时读取 `_lastQueuedEvents` 的值
- 两个线程都把自己的 `node->prev` 指向同一个旧节点
- 两个线程都把 `_lastQueuedEvents` 写为自己的节点
- 结果：其中一个线程的节点丢失，对应的事件永远不会被执行

这是典型的 ABA 问题的简化版。用 `std::atomic` + CAS 操作保证了"读取→比较→写入"是原子的，不会出现中间态。

</details>

### 题目 3：从 C++ 调用 Java Toast

写一个 C++ 函数，通过 JNI 在 Android 屏幕上显示一个 Toast 消息。假设你已经有 `JavaVM* g_vm` 和一个工具函数 `JNIEnv* GetEnv()` 可用。

<details>
<summary>查看答案</summary>

```cpp
#include <jni.h>
#include <string>

// 假设 g_vm 和 GetEnv() 已经可用
extern JavaVM *g_vm;
JNIEnv *GetEnv();

void showToast(const std::string &message) {
    JNIEnv *env = GetEnv();
    if (!env) return;

    // 1. 查找 KR2Activity 类
    jclass activityClass = env->FindClass("org/tvp/kirikiri2/KR2Activity");
    if (!activityClass) return;

    // 2. 获取 showToast 的方法 ID
    //    假设 Java 侧有：public static void showToast(String msg)
    jmethodID showToastMethod = env->GetStaticMethodID(
        activityClass, "showToast", "(Ljava/lang/String;)V");
    if (!showToastMethod) {
        env->DeleteLocalRef(activityClass);
        return;
    }

    // 3. 创建 Java String
    jstring jmsg = env->NewStringUTF(message.c_str());

    // 4. 调用 Java 方法
    env->CallStaticVoidMethod(activityClass, showToastMethod, jmsg);

    // 5. 释放本地引用
    env->DeleteLocalRef(jmsg);
    env->DeleteLocalRef(activityClass);
}
```

**注意事项：**
- Toast 必须在 UI 线程显示，所以 Java 侧的 `showToast` 方法内部需要 `runOnUiThread`
- 如果从非 JNI 线程调用，需要先 `AttachCurrentThread` 获取有效的 `JNIEnv*`
- 如果频繁调用，应该缓存 `jclass` 和 `jmethodID`（它们在类卸载前保持有效）

</details>

---

## 下一步

下一节将深入 JNI 的另一个核心难点——引用管理和字符串处理。你将学习本地引用、全局引用、弱全局引用的区别，以及如何安全地在 JNI 中处理字符串转换：

→ [引用管理与字符串处理](./03-引用管理与字符串处理.md)

