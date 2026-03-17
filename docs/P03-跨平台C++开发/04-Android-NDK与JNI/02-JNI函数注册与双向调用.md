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

