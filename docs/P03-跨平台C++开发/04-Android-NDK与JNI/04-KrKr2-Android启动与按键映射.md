# KrKr2 Android 启动流程与输入事件处理

> **所属模块：** P03-跨平台C++开发
> **前置知识：** [JNI 类型系统](./01-NDK基础与JNI类型系统.md)、[JNI 函数注册与双向调用](./02-JNI函数注册与双向调用.md)、[引用管理与字符串处理](./03-引用管理与字符串处理.md)
> **预计阅读时间：** 40 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 Android 应用从 Java Activity 到 C++ 引擎的完整启动链路
2. 掌握 `cocos_android_app_init()` 的三阶段初始化（日志、动态库、委托）
3. 理解跨线程事件投递机制（`Android_PushEvents` 无锁队列）
4. 实现 Android 键码到引擎键码的映射表
5. 理解触摸事件从 Java `MotionEvent` 到 C++ `handleTouches*` 的完整数据流
6. 掌握 Breakpad 崩溃转储的 JNI 集成方式

---

## 为什么需要学习启动流程？

**场景**：假设你加入了 KrKr2 项目，被分配了一个 Bug——"某些 Android 设备上游戏启动后黑屏 3 秒才出画面"。你打开 Logcat，看到日志显示 `cocos_android_app_init` 在 `dlopen("libSDL2.so")` 处卡了很久。但你不知道这个函数在启动链路的哪个位置，也不知道它之前和之后还发生了什么。

这就是为什么理解启动流程至关重要——**不了解全貌，就无法定位局部问题**。

在桌面平台（Windows/Linux/macOS），程序入口是 `main()` 函数，启动流程一目了然。但在 Android 上，启动流程跨越了多个层：

```
Android 系统 → Java Activity → Cocos2d-x 框架 → JNI 回调 → C++ 引擎
```

每一层都可能出问题，而且调试方式各不相同。本节将逐层拆解 KrKr2 的启动流程，让你对每个环节都了然于胸。

---

## Android 应用的启动入口

### 对比：桌面平台 vs Android

在桌面平台上，KrKr2 的入口非常直接：

```cpp
// platforms/windows/main.cpp — Windows 入口
int _tWinMain(HINSTANCE hInstance, ...) {
    // 初始化日志
    // 构造 TVPAppDelegate
    // 调用 run()
}

// platforms/linux/main.cpp — Linux 入口
int main(int argc, char* argv[]) {
    // 初始化日志
    // 构造 TVPAppDelegate
    // 调用 run()
}
```

这些都是 C++ 程序员熟悉的 `main()` 入口。但在 Android 上，**没有 `main()` 函数**。

为什么？因为 Android 应用的生命周期由系统管理。Android 系统首先启动一个 Java 虚拟机（ART），然后创建你的 `Activity`，调用 `onCreate()`。你的 C++ 代码是被 Java 代码"拉起来"的，而不是自己启动的。

### Java 层：KR2Activity.onCreate()

KrKr2 的 Android 入口是 `KR2Activity`，它继承自 Cocos2d-x 提供的 `Cocos2dxActivity`：

```java
// platforms/android/app/java/org/tvp/kirikiri2/KR2Activity.java 第 98-103 行
public class KR2Activity extends Cocos2dxActivity
    implements ActivityCompat.OnRequestPermissionsResultCallback {

    @SuppressLint("StaticFieldLeak")
    static public KR2Activity sInstance;  // 全局单例，C++ 通过 JNI 访问

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);  // 触发 Cocos2dxActivity 初始化
        sInstance = this;                     // 保存实例引用
        initDump(this.getFilesDir().getAbsolutePath() + "/dump");  // 初始化崩溃转储
    }
}
```

这里有三个关键点：

1. **`super.onCreate()`**：调用父类 `Cocos2dxActivity` 的初始化，这会触发 `System.loadLibrary("krkr2")`，加载我们的 C++ 动态库（`.so` 文件）
2. **`sInstance = this`**：保存 Activity 的全局引用。后面 C++ 代码通过 JNI 调用 Java 方法时，需要这个引用来获取 Android 上下文（Context）
3. **`initDump(path)`**：这是一个 `native` 方法，调用 C++ 代码初始化 Google Breakpad 崩溃转储

### 库加载触发 C++ 初始化

当 `Cocos2dxActivity.onCreate()` 调用 `System.loadLibrary("krkr2")` 时，Android 系统会：

1. 在 APK 的 `lib/arm64-v8a/`（或 `lib/x86_64/`）目录下找到 `libkrkr2.so`
2. 将 `.so` 文件映射到进程内存空间
3. 调用 `.so` 中的 `JNI_OnLoad()` 函数（如果有的话）
4. 随后 Cocos2d-x 框架会回调 `cocos_android_app_init()` 函数

这个回调就是 KrKr2 在 Android 上的"真正入口"。

---

## cocos_android_app_init：三阶段初始化

这个函数位于 `platforms/android/cpp/krkr2_android.cpp` 第 37-67 行，是整个 Android 端的核心初始化逻辑。让我们逐行分析：

```cpp
// platforms/android/cpp/krkr2_android.cpp 第 37-67 行
[[maybe_unused]] void cocos_android_app_init(JNIEnv *env) {
    // ====== 阶段 1：初始化日志系统 ======
    spdlog::set_pattern("%v");           // 日志格式：仅输出消息内容
    spdlog::set_level(spdlog::level::debug);  // 开启 debug 级别

    // 创建三个独立的日志通道，输出到 Android Logcat
    static auto core_logger =
        spdlog::android_logger_mt("core", "KrKr2NativeCore");
    static auto tjs2_logger =
        spdlog::android_logger_mt("tjs2", "KrKr2NativeTjs2");
    static auto plugin_logger =
        spdlog::android_logger_mt("plugin", "KrKr2NativePlugin");

    // 设置默认日志器为 core
    spdlog::set_default_logger(core_logger);

    // ====== 阶段 2：动态加载 SDL2 ======
    JavaVM *vm{};
    env->GetJavaVM(&vm);  // 从 JNIEnv 获取 JavaVM 指针

    void *handle = dlopen("libSDL2.so", RTLD_LAZY);  // 运行时加载 SDL2
    if (handle) {
        // 获取 SDL2 的 JNI_OnLoad 函数指针
        typedef jint (*JNI_OnLoad)(JavaVM *, void *);
        void *sdl2Init = dlsym(handle, "JNI_OnLoad");
        // 手动调用 SDL2 的 JNI_OnLoad，让它注册自己的 native 方法
        if (!sdl2Init ||
            ((JNI_OnLoad)sdl2Init)(vm, nullptr) != JNI_VERSION_1_4) {
            spdlog::critical("invoke libSDL2.so JNI_OnLoad method failed");
        }
    } else {
        spdlog::critical("load libSDL2.so failed");
    }

    // ====== 阶段 3：创建引擎委托 ======
    static std::unique_ptr<TVPAppDelegate> pAppDelegate =
        std::make_unique<TVPAppDelegate>();
}
```

### 阶段 1：日志系统初始化

**为什么要最先初始化日志？** 因为后续的所有步骤都可能失败，如果日志系统还没就绪，你就无法在 Logcat 中看到错误信息。

KrKr2 使用 spdlog 库，并创建了三个独立的日志通道：

| 日志器名称 | Logcat Tag | 用途 |
|-----------|------------|------|
| `core` | `KrKr2NativeCore` | 引擎核心日志 |
| `tjs2` | `KrKr2NativeTjs2` | TJS2 脚本引擎日志 |
| `plugin` | `KrKr2NativePlugin` | 插件系统日志 |

在 Android Studio 的 Logcat 中，你可以按 Tag 过滤，快速定位特定模块的问题：

```bash
# 在终端中使用 adb logcat 过滤
adb logcat -s KrKr2NativeCore:D KrKr2NativeTjs2:D KrKr2NativePlugin:D
```

### 阶段 2：动态加载 SDL2

这是整个启动流程中最"反常"的部分。通常我们在 CMake 中用 `target_link_libraries` 链接库，但 KrKr2 选择了 `dlopen` 运行时加载。**为什么？**

原因是 **JNI_OnLoad 的调用时机问题**：

```
正常流程（行不通）：
1. System.loadLibrary("SDL2")  → SDL2 的 JNI_OnLoad 被调用
2. System.loadLibrary("krkr2") → krkr2 依赖 SDL2

KrKr2 的实际情况：
1. System.loadLibrary("krkr2") → krkr2 先被加载
2. 但 SDL2 还没初始化！
3. SDL2 的 native 方法没有注册，调用就会崩溃
```

所以 KrKr2 采用了一个巧妙的方案：在 `cocos_android_app_init` 中手动用 `dlopen` 加载 SDL2，然后用 `dlsym` 找到 SDL2 的 `JNI_OnLoad` 函数指针，手动调用它。这样 SDL2 就能在正确的时机完成初始化。

```cpp
// 这段代码的本质是：手动触发 SDL2 的 JNI 注册
void *handle = dlopen("libSDL2.so", RTLD_LAZY);
void *sdl2Init = dlsym(handle, "JNI_OnLoad");
((JNI_OnLoad)sdl2Init)(vm, nullptr);  // 手动调用
```

> **类比**：就像你请了一位翻译（SDL2），但翻译比你晚到会议室。你不能等他自己来，只好主动打电话让他赶紧就位（`dlopen` + `dlsym` + 手动调用 `JNI_OnLoad`）。

### 阶段 3：创建引擎委托

```cpp
static std::unique_ptr<TVPAppDelegate> pAppDelegate =
    std::make_unique<TVPAppDelegate>();
```

`TVPAppDelegate` 是 KrKr2 的核心应用类，继承自 Cocos2d-x 的 `Application`。创建它之后，Cocos2d-x 框架会在合适的时机调用 `applicationDidFinishLaunching()`，从而启动渲染管线、TJS2 脚本引擎和游戏脚本。

注意这里使用了 `static` 局部变量——这是故意的。`static` 保证 `pAppDelegate` 的生命周期贯穿整个程序运行期，不会被提前析构。

---

## 完整启动链路图

将上面的所有步骤串联起来，完整的启动链路如下：

```
┌─────────────────────────────────────────────────────────┐
│ 1. Android 系统                                          │
│    └─ 启动 KR2Activity.onCreate()                        │
│       ├─ super.onCreate() → Cocos2dxActivity 初始化      │
│       │  └─ System.loadLibrary("krkr2")                  │
│       │     └─ 加载 libkrkr2.so 到进程内存                │
│       ├─ sInstance = this  (保存 Activity 引用)           │
│       └─ initDump(path)   (初始化崩溃转储)                │
├─────────────────────────────────────────────────────────┤
│ 2. Cocos2d-x 框架回调                                    │
│    └─ cocos_android_app_init(JNIEnv *env)                │
│       ├─ 阶段 1：spdlog 日志初始化                        │
│       │  └─ 创建 core/tjs2/plugin 三个日志通道            │
│       ├─ 阶段 2：dlopen("libSDL2.so")                    │
│       │  └─ 手动调用 SDL2 的 JNI_OnLoad                  │
│       └─ 阶段 3：创建 TVPAppDelegate                     │
├─────────────────────────────────────────────────────────┤
│ 3. Cocos2d-x 引擎启动                                    │
│    └─ Director::init()                                    │
│       └─ TVPAppDelegate::run()                           │
│          └─ applicationDidFinishLaunching()               │
│             ├─ 初始化渲染管线                              │
│             ├─ 初始化 TJS2 脚本引擎                       │
│             └─ 加载游戏脚本 (.tjs)                        │
├─────────────────────────────────────────────────────────┤
│ 4. 运行时循环                                             │
│    ├─ Java UI 线程：触摸/按键 → nativeXxx()               │
│    ├─ GL 渲染线程：_processEvents() → 分发事件            │
│    ├─ C++ → JniHelper → Java 系统 API                    │
│    └─ 低内存回调 → nativeOnLowMemory()                   │
└─────────────────────────────────────────────────────────┘
```

---

## 崩溃转储：Breakpad 集成

### 为什么需要崩溃转储？

**场景**：用户反馈"游戏在某个场景必定闪退"，但你手头没有用户的设备，也无法复现。如果没有崩溃转储，你只能猜测原因。而有了 Breakpad，闪退时会自动生成 `.dmp` 文件，包含崩溃时的调用栈、寄存器状态等信息。

### KrKr2 的 Breakpad 集成

```cpp
// platforms/android/cpp/krkr2_android.cpp 第 25-35 行

// 崩溃回调：收到转储文件后的处理
static bool DumpCallback(
    const google_breakpad::MinidumpDescriptor &descriptor,
    void *context, bool succeeded) {
    return succeeded;  // 简单返回是否成功
}

extern bool TVPSystemUninitCalled;  // 引擎是否正在退出

// 崩溃过滤器：决定是否生成转储
static bool DumpFilter(void *data) {
    // 如果引擎正在正常退出，忽略所有异常
    // 避免正常退出时产生无意义的崩溃报告
    return !TVPSystemUninitCalled;
}
```

初始化代码通过 JNI 从 Java 层调用：

```cpp
// platforms/android/cpp/krkr2_android.cpp 第 80-89 行
void Java_org_tvp_kirikiri2_KR2Activity_initDump(
    JNIEnv *env, jclass cls, jstring path) {
    const char *pszPath = env->GetStringUTFChars(path, nullptr);
    if (pszPath && *pszPath) {
        // 创建转储文件描述符（指定存储目录）
        static google_breakpad::MinidumpDescriptor descriptor(pszPath);
        // 创建异常处理器
        static google_breakpad::ExceptionHandler eh(
            descriptor,      // 转储文件目录
            DumpFilter,       // 过滤器：正常退出时不转储
            DumpCallback,     // 回调：转储完成后执行
            nullptr,          // 回调上下文
            true,             // 安装信号处理器
            -1                // 服务器 FD（-1 = 不使用）
        );
    }
    env->ReleaseStringUTFChars(path, pszPath);
}
```

Java 层在 `onCreate` 中调用这个 native 方法：

```java
// KR2Activity.java 第 102 行
initDump(this.getFilesDir().getAbsolutePath() + "/dump");
// 转储文件存储在：/data/data/org.tvp.kirikiri2/files/dump/
```

### 常见错误 ❌

**错误 1**：忘记在 `DumpFilter` 中排除正常退出

```cpp
// ❌ 错误：正常退出也会生成崩溃报告，日志目录塞满无用文件
static bool DumpFilter(void *data) {
    return true;  // 总是生成转储
}

// ✅ 正确：检查退出标志
static bool DumpFilter(void *data) {
    return !TVPSystemUninitCalled;
}
```

**错误 2**：Breakpad 对象不是 `static`

```cpp
// ❌ 错误：eh 是局部变量，函数返回后被析构，信号处理器被移除
void initDump(const char *path) {
    google_breakpad::MinidumpDescriptor descriptor(path);
    google_breakpad::ExceptionHandler eh(descriptor, ...);
    // 函数结束，eh 析构，崩溃转储失效！
}

// ✅ 正确：使用 static 保持生命周期
void initDump(const char *path) {
    static google_breakpad::MinidumpDescriptor descriptor(path);
    static google_breakpad::ExceptionHandler eh(descriptor, ...);
    // static 变量存活到程序结束
}
```

---

## 跨线程事件队列：Android_PushEvents

### 问题：Java 线程 vs 渲染线程

Android 应用中存在多个线程，其中最重要的两个：

| 线程 | 职责 | 示例 |
|------|------|------|
| **Java UI 线程**（主线程） | 处理 Android 系统事件 | 触摸、按键、Activity 生命周期 |
| **GL 渲染线程** | 执行 OpenGL 渲染 | Cocos2d-x 的 Director、Scene、Scheduler |

**问题来了**：当用户在屏幕上点击时，Android 系统在 **UI 线程** 上调用 `onTouchEvent()`，然后通过 JNI 调用 C++ 的 `nativeTouchesBegin()`。但 Cocos2d-x 的渲染逻辑运行在 **GL 线程** 上。如果直接在 UI 线程中调用 Cocos2d-x 的 API，就会产生**线程安全问题**（竞态条件、数据损坏、崩溃）。

```
用户触摸屏幕
    ↓
Android UI 线程：onTouchEvent()
    ↓
JNI 调用：nativeTouchesBegin()  ← 仍在 UI 线程上！
    ↓
❌ 直接调用 Cocos2d-x API → 线程不安全！
✅ 放入事件队列 → GL 线程稍后安全处理
```

### KrKr2 的解决方案：无锁链表队列

KrKr2 使用了一个巧妙的无锁（lock-free）链表来跨线程传递事件。核心代码在 `AndroidUtils.cpp` 第 659-690 行：

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 659-690 行

// 事件节点：链表的基本元素
struct _eventQueueNode {
    std::function<void()> func;  // 要执行的操作（lambda）
    _eventQueueNode *prev;        // 前一个节点
    _eventQueueNode *next;        // 后一个节点
};

// 原子指针：链表尾部（所有线程共享）
static std::atomic<_eventQueueNode *> _lastQueuedEvents(nullptr);

// 入队操作：在任意线程中调用（通常是 UI 线程）
void Android_PushEvents(const std::function<void()> &func) {
    _eventQueueNode *node = new _eventQueueNode;
    node->func = func;
    node->next = nullptr;
    node->prev = nullptr;
    // CAS（Compare-And-Swap）循环：无锁入队
    // node->prev 被设置为当前的尾节点
    // 然后尝试将 _lastQueuedEvents 更新为 node
    // 如果失败（另一个线程抢先了），重试
    while (!_lastQueuedEvents.compare_exchange_weak(node->prev, node)) {
    }
}

// 出队并执行：在 GL 线程中每帧调用
static void _processEvents(float) {
    // 原子地取走整个队列
    _eventQueueNode *q = _lastQueuedEvents.exchange(nullptr);
    if (q) {
        // 反转链表（因为入队是后进的在前）
        q->next = nullptr;
        while (q->prev) {
            q->prev->next = q;
            q = q->prev;
        }
    }
    // 按顺序执行所有事件
    while (q) {
        q->func();                // 在 GL 线程上安全执行
        _eventQueueNode *nq = q->next;
        delete q;                  // 释放节点
        q = nq;
    }
}
```

### 事件调度器注册

`_processEvents` 函数需要每帧被调用，KrKr2 在 `TVPCheckStartupArg()` 中将它注册到 Cocos2d-x 的调度器：

```cpp
// cpp/core/environ/android/AndroidUtils.cpp 第 695-712 行
bool TVPCheckStartupArg() {
    // 检查并发送崩溃转储
    TVPCheckAndSendDumps(Android_GetDumpStoragePath(), ...);

    // 注册事件分发器到 Cocos2d-x 的 Scheduler
    cocos2d::Director *director = cocos2d::Director::getInstance();

    // 通过继承 hack 访问 protected 方法 schedulePerFrame
    class HackForScheduler : public cocos2d::Scheduler {
    public:
        void regProcessEvents() {
            // 优先级 -1：在其他调度任务之前执行
            schedulePerFrame(_processEvents, &_lastQueuedEvents, -1, false);
        }
    };
    static_cast<HackForScheduler *>(director->getScheduler())
        ->regProcessEvents();

    return false;
}
```

> **为什么要用 `HackForScheduler`？** 因为 `schedulePerFrame` 是 `Scheduler` 的 `protected` 方法，外部代码无法直接调用。KrKr2 通过定义一个子类并使用 `static_cast` 来"绕过"访问控制。这是一种常见但不推荐的技巧（hack），在实际项目中偶尔可见。

### 可视化：事件流转过程

```
Java UI 线程                              GL 渲染线程
─────────────                             ────────────
onTouchEvent()                            每帧循环开始
    ↓                                         ↓
nativeTouchesBegin()                      _processEvents()
    ↓                                         ↓
Android_PushEvents(lambda)                exchange(nullptr) 取走队列
    ↓                                         ↓
[node] → [node] → [node]  ← 无锁链表 →   反转链表，逐个执行
                                              ↓
                                          handleTouchesBegin()
                                          （安全地在 GL 线程执行）
```

### 为什么不用 mutex？

你可能会问：直接用 `std::mutex` 保护一个 `std::queue` 不行吗？技术上可以，但有性能问题：

| 方案 | 优点 | 缺点 |
|------|------|------|
| `std::mutex` + `std::queue` | 代码简单，容易理解 | UI 线程可能被阻塞（如果 GL 线程正在处理事件） |
| CAS 无锁链表（KrKr2 方案） | UI 线程永远不会被阻塞 | 代码复杂，需要手动管理内存 |
| `performFunctionInCocosThread` | Cocos2d-x 内置支持 | 内部也用了 mutex，有额外的锁开销 |

对于输入事件这种高频操作（每秒可能触发数十次 `touchMove`），无锁方案能避免 UI 线程的任何卡顿。

---

## 按键映射：Android KeyCode → Cocos2d-x KeyCode

### 为什么需要映射？

Android 和 Cocos2d-x 使用完全不同的键码体系：

- **Android** 使用 `android.view.KeyEvent` 中定义的常量，如 `KEYCODE_BACK = 4`
- **Cocos2d-x** 使用自己的 `EventKeyboard::KeyCode` 枚举，如 `KEY_ESCAPE`

KrKr2 作为一个视觉小说引擎，不需要处理所有按键——只需要几个关键按键用于游戏操作。

### Java 层：按键拦截

首先，在 Java 层，`KR2GLSurfaceView` 重写了 `onKeyDown` 和 `onKeyUp` 来拦截特定按键：

```java
// KR2Activity.java 第 210-235 行
// KR2GLSurfaceView 继承自 Cocos2dxGLSurfaceView
@Override
public boolean onKeyDown(final int pKeyCode, final KeyEvent pKeyEvent) {
    return switch (pKeyCode) {
        // 只拦截以下按键，其余交给系统处理
        case KeyEvent.KEYCODE_BACK,          // 返回键
             KeyEvent.KEYCODE_MENU,          // 菜单键
             KeyEvent.KEYCODE_DPAD_LEFT,     // 方向左
             KeyEvent.KEYCODE_DPAD_RIGHT,    // 方向右
             KeyEvent.KEYCODE_DPAD_UP,       // 方向上
             KeyEvent.KEYCODE_DPAD_DOWN,     // 方向下
             KeyEvent.KEYCODE_ENTER,         // 确认键
             KeyEvent.KEYCODE_MEDIA_PLAY_PAUSE,  // 播放/暂停
             KeyEvent.KEYCODE_DPAD_CENTER -> {    // 方向中心
            nativeKeyAction(pKeyCode, true);   // 通过 JNI 传递给 C++
            yield true;                         // 告诉系统"我已处理"
        }
        default -> super.onKeyDown(pKeyCode, pKeyEvent);  // 未处理的交给父类
    };
}
```

> **注意**：这里使用了 Java 14 的 `switch` 表达式语法（`yield`）。`yield true` 表示返回 `true`，即告诉 Android 系统"这个按键已被消费，不要再传递给其他组件"。

### C++ 层：键码转换

Java 层拦截到按键后，通过 JNI 调用 C++ 的 `nativeKeyAction`：

```cpp
// platforms/android/cpp/krkr2_android.cpp 第 221-276 行

// 定义 Android 键码常量（与 android.view.KeyEvent 对应）
#define KEYCODE_BACK       0x04  // 返回键
#define KEYCODE_MENU       0x52  // 菜单键
#define KEYCODE_DPAD_UP    0x13  // 方向上
#define KEYCODE_DPAD_DOWN  0x14  // 方向下
#define KEYCODE_DPAD_LEFT  0x15  // 方向左
#define KEYCODE_DPAD_RIGHT 0x16  // 方向右
#define KEYCODE_ENTER      0x42  // 确认键
#define KEYCODE_PLAY       0x7e  // 播放
#define KEYCODE_DPAD_CENTER 0x17 // 方向中心
#define KEYCODE_DEL        0x43  // 删除键

JNIEXPORT jboolean JNICALL
Java_org_tvp_kirikiri2_KR2Activity_nativeKeyAction(
    JNIEnv *env, jclass cls,
    jint keyCode,        // Android 键码
    jboolean isPress) {  // true=按下, false=抬起

    cocos2d::EventKeyboard::KeyCode pKeyCode;
    switch (keyCode) {
        case KEYCODE_BACK:
            // 返回键 → ESC：视觉小说中通常用于打开菜单或返回上一层
            pKeyCode = cocos2d::EventKeyboard::KeyCode::KEY_ESCAPE;
            break;
        case KEYCODE_MENU:
            pKeyCode = cocos2d::EventKeyboard::KeyCode::KEY_MENU;
            break;
        case KEYCODE_DPAD_UP:
            pKeyCode = cocos2d::EventKeyboard::KeyCode::KEY_DPAD_UP;
            break;
        case KEYCODE_DPAD_DOWN:
            pKeyCode = cocos2d::EventKeyboard::KeyCode::KEY_DPAD_DOWN;
            break;
        case KEYCODE_DPAD_LEFT:
            pKeyCode = cocos2d::EventKeyboard::KeyCode::KEY_DPAD_LEFT;
            break;
        case KEYCODE_DPAD_RIGHT:
            pKeyCode = cocos2d::EventKeyboard::KeyCode::KEY_DPAD_RIGHT;
            break;
        case KEYCODE_ENTER:
            pKeyCode = cocos2d::EventKeyboard::KeyCode::KEY_ENTER;
            break;
        case KEYCODE_PLAY:
            pKeyCode = cocos2d::EventKeyboard::KeyCode::KEY_PLAY;
            break;
        case KEYCODE_DPAD_CENTER:
            pKeyCode = cocos2d::EventKeyboard::KeyCode::KEY_DPAD_CENTER;
            break;
        case KEYCODE_DEL:
            // 删除键 → 退格键
            pKeyCode = cocos2d::EventKeyboard::KeyCode::KEY_BACKSPACE;
            break;
        default:
            return JNI_FALSE;  // 未识别的键码，返回"未处理"
    }

    // 通过事件队列投递到 GL 线程
    Android_PushEvents([pKeyCode, isPress]() {
        cocos2d::EventKeyboard event(pKeyCode, isPress);
        cocos2d::Director::getInstance()
            ->getEventDispatcher()
            ->dispatchEvent(&event);
    });
    return JNI_TRUE;  // 返回"已处理"
}
```

### 完整键码映射表

| Android 键码 | 十六进制值 | Cocos2d-x 键码 | 在视觉小说中的用途 |
|-------------|-----------|---------------|-------------------|
| `KEYCODE_BACK` | `0x04` | `KEY_ESCAPE` | 打开菜单 / 返回 |
| `KEYCODE_MENU` | `0x52` | `KEY_MENU` | 系统菜单 |
| `KEYCODE_DPAD_UP` | `0x13` | `KEY_DPAD_UP` | 选项上移 |
| `KEYCODE_DPAD_DOWN` | `0x14` | `KEY_DPAD_DOWN` | 选项下移 |
| `KEYCODE_DPAD_LEFT` | `0x15` | `KEY_DPAD_LEFT` | 回看历史 |
| `KEYCODE_DPAD_RIGHT` | `0x16` | `KEY_DPAD_RIGHT` | 快进文本 |
| `KEYCODE_ENTER` | `0x42` | `KEY_ENTER` | 确认选项 / 下一句 |
| `KEYCODE_PLAY` | `0x7e` | `KEY_PLAY` | 自动播放 |
| `KEYCODE_DPAD_CENTER` | `0x17` | `KEY_DPAD_CENTER` | 确认（遥控器） |
| `KEYCODE_DEL` | `0x43` | `KEY_BACKSPACE` | 删除文本输入 |

---

## 触摸事件处理

### 触摸事件的四种类型

Android 触摸事件分为四种基本类型，KrKr2 为每种都提供了对应的 JNI 函数：

| 事件类型 | Java 方法 | C++ JNI 函数 | 触发时机 |
|---------|----------|-------------|---------|
| 手指按下 | `ACTION_DOWN` / `ACTION_POINTER_DOWN` | `nativeTouchesBegin` | 手指刚接触屏幕 |
| 手指移动 | `ACTION_MOVE` | `nativeTouchesMove` | 手指在屏幕上滑动 |
| 手指抬起 | `ACTION_UP` / `ACTION_POINTER_UP` | `nativeTouchesEnd` | 手指离开屏幕 |
| 触摸取消 | `ACTION_CANCEL` | `nativeTouchesCancel` | 系统打断触摸（如来电） |

### Java 层：触摸事件分发

```java
// KR2Activity.java 第 255-312 行
@Override
public boolean onTouchEvent(final MotionEvent pMotionEvent) {
    // 提取所有触摸点的信息
    final int pointerNumber = pMotionEvent.getPointerCount();
    final int[] ids = new int[pointerNumber];
    final float[] xs = new float[pointerNumber];
    final float[] ys = new float[pointerNumber];

    for (int i = 0; i < pointerNumber; i++) {
        ids[i] = pMotionEvent.getPointerId(i);  // 每个手指的唯一 ID
        xs[i] = pMotionEvent.getX(i);             // X 坐标（像素）
        ys[i] = pMotionEvent.getY(i);             // Y 坐标（像素）
    }

    switch (pMotionEvent.getAction() & MotionEvent.ACTION_MASK) {
        case MotionEvent.ACTION_DOWN:
            // 第一根手指按下
            nativeTouchesBegin(ids[0], xs[0], ys[0]);
            break;
        case MotionEvent.ACTION_POINTER_DOWN:
            // 后续手指按下（多点触控）
            int idx = pMotionEvent.getAction()
                >> MotionEvent.ACTION_POINTER_INDEX_SHIFT;
            nativeTouchesBegin(
                pMotionEvent.getPointerId(idx),
                pMotionEvent.getX(idx),
                pMotionEvent.getY(idx));
            break;
        case MotionEvent.ACTION_MOVE:
            // 手指移动——传递所有触摸点
            nativeTouchesMove(ids, xs, ys);
            break;
        case MotionEvent.ACTION_UP:
            nativeTouchesEnd(ids[0], xs[0], ys[0]);
            break;
        case MotionEvent.ACTION_CANCEL:
            nativeTouchesCancel(ids, xs, ys);
            break;
    }
    return true;  // 消费事件
}
```

### C++ 层：单点 vs 多点触控

以 `nativeTouchesBegin`（单点触控）为例，代码非常简洁：

```cpp
// platforms/android/cpp/krkr2_android.cpp 第 107-114 行
JNIEXPORT void JNICALL
Java_org_tvp_kirikiri2_KR2Activity_nativeTouchesBegin(
    JNIEnv *env, jclass thiz,
    jint id,      // 触摸点 ID
    jfloat x,     // X 坐标
    jfloat y) {   // Y 坐标
    intptr_t idlong = id;  // jint → intptr_t（适配 64 位）
    Android_PushEvents([idlong, x, y]() {
        cocos2d::Director::getInstance()
            ->getOpenGLView()
            ->handleTouchesBegin(
                1,                     // 触摸点数量
                (intptr_t *)&idlong,   // ID 数组
                (float *)&x,           // X 坐标数组
                (float *)&y);          // Y 坐标数组
    });
}
```

`nativeTouchesMove`（多点触控）更复杂，因为要处理数组数据：

```cpp
// platforms/android/cpp/krkr2_android.cpp 第 125-171 行
JNIEXPORT void JNICALL
Java_org_tvp_kirikiri2_KR2Activity_nativeTouchesMove(
    JNIEnv *env, jclass thiz,
    jintArray ids,       // 所有触摸点的 ID 数组
    jfloatArray xs,      // 所有触摸点的 X 坐标数组
    jfloatArray ys) {    // 所有触摸点的 Y 坐标数组
    int size = env->GetArrayLength(ids);

    // 优化：单点触控走快速路径
    if (size == 1) {
        intptr_t idlong;
        jint id;
        jfloat x, y;
        // GetXxxArrayRegion：复制数组元素，无需 Release
        env->GetIntArrayRegion(ids, 0, 1, &id);
        env->GetFloatArrayRegion(xs, 0, 1, &x);
        env->GetFloatArrayRegion(ys, 0, 1, &y);
        idlong = id;
        Android_PushEvents([idlong, x, y]() {
            cocos2d::Director::getInstance()
                ->getOpenGLView()
                ->handleTouchesMove(
                    1, (intptr_t *)&idlong,
                    (float *)&x, (float *)&y);
        });
        return;
    }

    // 多点触控：复制全部数据
    jint id[size];  // 注意：这里使用了 VLA（变长数组），非标准 C++
    std::vector<jfloat> x(size), y(size);

    env->GetIntArrayRegion(ids, 0, size, id);
    env->GetFloatArrayRegion(xs, 0, size, &x[0]);
    env->GetFloatArrayRegion(ys, 0, size, &y[0]);

    // jint → intptr_t 的类型转换
    std::vector<intptr_t> idlong(size);
    for (int i = 0; i < size; i++)
        idlong[i] = id[i];

    // lambda 捕获 vector（按值复制），保证 GL 线程执行时数据仍有效
    Android_PushEvents([idlong, x, y]() {
        cocos2d::Director::getInstance()
            ->getOpenGLView()
            ->handleTouchesMove(
                idlong.size(),
                (intptr_t *)&idlong[0],
                (float *)&x[0],
                (float *)&y[0]);
    });
}
```

### 关键设计决策分析

**为什么用 `GetIntArrayRegion` 而不是 `GetIntArrayElements`？**

| API | 作用 | 是否需要 Release | 适用场景 |
|-----|------|:---------------:|---------|
| `GetIntArrayRegion` | 复制数组片段到本地缓冲区 | 否 ✅ | 读取少量元素 |
| `GetIntArrayElements` | 获取数组指针（可能是直接引用） | 是（必须 Release） | 需要就地修改或大量数据 |

KrKr2 选择 `GetIntArrayRegion` 是正确的——触摸点通常只有 1-5 个，数据量很小，直接复制更安全。

**为什么 lambda 按值捕获？**

```cpp
// ✅ 正确：按值捕获，数据复制到 lambda 中
Android_PushEvents([idlong, x, y]() {
    // idlong, x, y 是独立副本，GL 线程安全访问
});

// ❌ 错误：按引用捕获，JNI 函数返回后局部变量已销毁
Android_PushEvents([&idlong, &x, &y]() {
    // 悬垂引用！访问已释放的栈内存 → 崩溃或数据损坏
});
```

---

## 其他 JNI 回调

### 文本输入

KrKr2 处理文本输入有两种路径：

```cpp
// 路径 1：IME 插入文本（软键盘输入）
// platforms/android/cpp/krkr2_android.cpp 第 278-298 行
JNIEXPORT void JNICALL
Java_org_tvp_kirikiri2_KR2Activity_nativeInsertText(
    JNIEnv *env, jclass cls, jstring text) {
    const char *pszText = env->GetStringUTFChars(text, nullptr);
    if (pszText && *pszText) {
        std::string str = pszText;  // 复制到 std::string
        Android_PushEvents([str]() {
            cocos2d::IMEDispatcher::sharedDispatcher()
                ->dispatchInsertText(str.c_str(), str.length());
        });
    }
    env->ReleaseStringUTFChars(text, pszText);  // 必须释放
}

// 路径 2：字符输入（物理键盘或特殊输入法）
// platforms/android/cpp/krkr2_android.cpp 第 309-316 行
JNIEXPORT void JNICALL
Java_org_tvp_kirikiri2_KR2Activity_nativeCharInput(
    JNIEnv *env, jclass cls, jint keyCode) {
    TVPMainScene *pScene = TVPMainScene::GetInstance();
    if (!pScene) return;
    // 使用 Cocos2d-x 内置的线程安全调度
    pScene->getScheduler()->performFunctionInCocosThread(
        [keyCode] { TVPMainScene::onCharInput(keyCode); });
}
```

注意两种路径使用了不同的线程跨越方式：
- `nativeInsertText` 使用自定义的 `Android_PushEvents`（无锁队列）
- `nativeCharInput` 使用 Cocos2d-x 的 `performFunctionInCocosThread`（内部用 mutex）

### 低内存回调

```cpp
// platforms/android/cpp/krkr2_android.cpp 第 386-393 行
JNIEXPORT void JNICALL
Java_org_tvp_kirikiri2_KR2Activity_nativeOnLowMemory(
    JNIEnv *env, jclass cls) {
    Android_PushEvents([]() {
        ::Application->OnLowMemory();  // 通知引擎释放缓存
    });
}
```

对应 Java 层：
```java
// KR2Activity.java 第 122-125 行
@Override
public void onLowMemory() {
    nativeOnLowMemory();  // 系统通知内存不足时调用
}
```

### 鼠标悬停和滚轮（Android TV / 外接鼠标）

```cpp
// platforms/android/cpp/krkr2_android.cpp 第 339-384 行
JNIEXPORT void JNICALL
Java_org_tvp_kirikiri2_KR2Activity_nativeHoverMoved(
    JNIEnv *env, jclass cls, jfloat x, jfloat y) {
    Android_PushEvents([x, y]() {
        cocos2d::GLView *glview =
            cocos2d::Director::getInstance()->getOpenGLView();
        // 将屏幕坐标转换为 Cocos2d-x 逻辑坐标
        float _scaleX = glview->getScaleX();
        float _scaleY = glview->getScaleY();
        const cocos2d::Rect _viewPortRect = glview->getViewPortRect();

        float cursorX = (x - _viewPortRect.origin.x) / _scaleX;
        float cursorY =
            (_viewPortRect.origin.y + _viewPortRect.size.height - y)
            / _scaleY;
        // 注意 Y 轴翻转：Android 的 Y 轴向下，Cocos2d-x 的 Y 轴向上

        cocos2d::EventMouse event(
            cocos2d::EventMouse::MouseEventType::MOUSE_MOVE);
        event.setCursorPosition(cursorX, cursorY);
        cocos2d::Director::getInstance()
            ->getEventDispatcher()
            ->dispatchEvent(&event);
    });
}
```

坐标转换公式：
```
Cocos2d-x X = (屏幕 X - 视口左边距) / X 缩放比
Cocos2d-x Y = (视口上边距 + 视口高度 - 屏幕 Y) / Y 缩放比
                 ↑ Y 轴翻转
```

---

## 动手实践

### 练习 1：实现一个简化的跨线程事件队列

创建一个不依赖 Cocos2d-x 的简化版事件队列：

```cpp
// event_queue.h
#pragma once
#include <atomic>
#include <functional>
#include <iostream>
#include <thread>
#include <chrono>

struct EventNode {
    std::function<void()> func;
    EventNode *prev = nullptr;
    EventNode *next = nullptr;
};

class EventQueue {
    std::atomic<EventNode *> tail_{nullptr};
public:
    // 任意线程调用：入队
    void push(const std::function<void()> &func) {
        EventNode *node = new EventNode;
        node->func = func;
        node->prev = nullptr;
        node->next = nullptr;
        while (!tail_.compare_exchange_weak(node->prev, node)) {
            // CAS 失败，自动重试（node->prev 被更新为最新的 tail）
        }
    }

    // 消费者线程调用：取走全部并执行
    void processAll() {
        EventNode *q = tail_.exchange(nullptr);
        if (!q) return;

        // 反转链表：最先入队的排在最前
        q->next = nullptr;
        while (q->prev) {
            q->prev->next = q;
            q = q->prev;
        }

        // 按顺序执行
        while (q) {
            q->func();
            EventNode *next = q->next;
            delete q;
            q = next;
        }
    }
};

// main.cpp — 测试
int main() {
    EventQueue queue;

    // 模拟多个"UI 线程"推送事件
    std::thread t1([&queue]() {
        for (int i = 0; i < 5; i++) {
            queue.push([i]() {
                std::cout << "线程1 事件 " << i << std::endl;
            });
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    });

    std::thread t2([&queue]() {
        for (int i = 0; i < 5; i++) {
            queue.push([i]() {
                std::cout << "线程2 事件 " << i << std::endl;
            });
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    });

    // 模拟"渲染线程"每 50ms 处理一次
    for (int frame = 0; frame < 5; frame++) {
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        std::cout << "--- 第 " << frame << " 帧 ---" << std::endl;
        queue.processAll();
    }

    t1.join();
    t2.join();

    // 处理剩余事件
    queue.processAll();

    return 0;
}
```

CMakeLists.txt：
```cmake
cmake_minimum_required(VERSION 3.20)
project(EventQueueDemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(event_queue_demo main.cpp)

# 需要线程库
find_package(Threads REQUIRED)
target_link_libraries(event_queue_demo Threads::Threads)
```

预期输出（顺序可能不同，但每帧内是有序的）：
```
--- 第 0 帧 ---
线程1 事件 0
线程2 事件 0
线程1 事件 1
线程2 事件 1
--- 第 1 帧 ---
线程1 事件 2
线程2 事件 2
...
```

### 练习 2：实现一个键码映射器

```cpp
// key_mapper.cpp
#include <iostream>
#include <map>
#include <string>

// 模拟 Android KeyEvent 常量
namespace AndroidKeyCode {
    constexpr int KEYCODE_BACK       = 0x04;
    constexpr int KEYCODE_MENU       = 0x52;
    constexpr int KEYCODE_DPAD_UP    = 0x13;
    constexpr int KEYCODE_DPAD_DOWN  = 0x14;
    constexpr int KEYCODE_DPAD_LEFT  = 0x15;
    constexpr int KEYCODE_DPAD_RIGHT = 0x16;
    constexpr int KEYCODE_ENTER      = 0x42;
    constexpr int KEYCODE_DEL        = 0x43;
}

// 模拟引擎键码
enum class EngineKey {
    ESCAPE, MENU, UP, DOWN, LEFT, RIGHT, ENTER, BACKSPACE, UNKNOWN
};

std::string engineKeyToString(EngineKey key) {
    switch (key) {
        case EngineKey::ESCAPE:    return "ESCAPE";
        case EngineKey::MENU:      return "MENU";
        case EngineKey::UP:        return "UP";
        case EngineKey::DOWN:      return "DOWN";
        case EngineKey::LEFT:      return "LEFT";
        case EngineKey::RIGHT:     return "RIGHT";
        case EngineKey::ENTER:     return "ENTER";
        case EngineKey::BACKSPACE: return "BACKSPACE";
        case EngineKey::UNKNOWN:   return "UNKNOWN";
    }
    return "UNKNOWN";
}

// 键码映射器
class KeyMapper {
    std::map<int, EngineKey> mapping_;
public:
    KeyMapper() {
        // 初始化映射表（与 KrKr2 一致）
        mapping_[AndroidKeyCode::KEYCODE_BACK]       = EngineKey::ESCAPE;
        mapping_[AndroidKeyCode::KEYCODE_MENU]       = EngineKey::MENU;
        mapping_[AndroidKeyCode::KEYCODE_DPAD_UP]    = EngineKey::UP;
        mapping_[AndroidKeyCode::KEYCODE_DPAD_DOWN]  = EngineKey::DOWN;
        mapping_[AndroidKeyCode::KEYCODE_DPAD_LEFT]  = EngineKey::LEFT;
        mapping_[AndroidKeyCode::KEYCODE_DPAD_RIGHT] = EngineKey::RIGHT;
        mapping_[AndroidKeyCode::KEYCODE_ENTER]      = EngineKey::ENTER;
        mapping_[AndroidKeyCode::KEYCODE_DEL]        = EngineKey::BACKSPACE;
    }

    // 映射键码，返回是否成功
    bool mapKey(int androidKeyCode, EngineKey &outKey) const {
        auto it = mapping_.find(androidKeyCode);
        if (it != mapping_.end()) {
            outKey = it->second;
            return true;
        }
        outKey = EngineKey::UNKNOWN;
        return false;
    }

    // 模拟按键处理
    bool handleKeyAction(int keyCode, bool isPress) {
        EngineKey engineKey;
        if (!mapKey(keyCode, engineKey)) {
            std::cout << "未识别的键码: 0x" << std::hex << keyCode
                      << std::dec << std::endl;
            return false;
        }
        std::cout << (isPress ? "按下" : "抬起")
                  << ": " << engineKeyToString(engineKey)
                  << " (Android 0x" << std::hex << keyCode
                  << std::dec << ")" << std::endl;
        return true;
    }
};

int main() {
    KeyMapper mapper;

    // 模拟按键序列
    std::cout << "=== 模拟按键事件 ===" << std::endl;
    mapper.handleKeyAction(0x04, true);   // 按下返回键
    mapper.handleKeyAction(0x04, false);  // 抬起返回键
    mapper.handleKeyAction(0x42, true);   // 按下确认键
    mapper.handleKeyAction(0x42, false);  // 抬起确认键
    mapper.handleKeyAction(0x13, true);   // 按下方向上
    mapper.handleKeyAction(0x99, true);   // 按下未知键

    return 0;
}
```

预期输出：
```
=== 模拟按键事件 ===
按下: ESCAPE (Android 0x4)
抬起: ESCAPE (Android 0x4)
按下: ENTER (Android 0x42)
抬起: ENTER (Android 0x42)
按下: UP (Android 0x13)
未识别的键码: 0x99
```

---

## 对照项目源码

本节涉及的 KrKr2 源文件和它们的职责：

| 文件路径 | 行号范围 | 内容 |
|---------|---------|------|
| `platforms/android/cpp/krkr2_android.cpp` | 全文 (394行) | JNI 桥梁：启动入口、按键映射、触摸事件、文本输入、鼠标悬停 |
| `cpp/core/environ/android/AndroidUtils.cpp` | 659-690 | 无锁事件队列（`Android_PushEvents` + `_processEvents`） |
| `cpp/core/environ/android/AndroidUtils.cpp` | 695-712 | 事件调度器注册（`TVPCheckStartupArg`） |
| `cpp/core/environ/android/AndroidUtils.cpp` | 507-551 | 消息框（`TVPShowSimpleMessageBox`）—— 展示 C++ 等待 Java UI 的同步模式 |
| `platforms/android/app/java/.../KR2Activity.java` | 98-103 | Java 启动入口（`onCreate`） |
| `platforms/android/app/java/.../KR2Activity.java` | 189-322 | 自定义 GLSurfaceView（按键、触摸、悬停、滚轮事件处理） |

阅读顺序建议：
1. 先看 `KR2Activity.java` 的 `onCreate()` 理解 Java 层启动
2. 再看 `krkr2_android.cpp` 的 `cocos_android_app_init()` 理解 C++ 层启动
3. 然后看 `AndroidUtils.cpp` 的事件队列理解跨线程机制
4. 最后看各个 `nativeXxx` 函数理解输入事件流转

---

## 本节小结

- Android 应用没有 `main()` 函数，启动链路是 `Activity.onCreate()` → `System.loadLibrary()` → `cocos_android_app_init()`
- KrKr2 的 Android 入口 `cocos_android_app_init` 分三个阶段：初始化 spdlog 日志 → 动态加载 SDL2 → 创建 `TVPAppDelegate`
- SDL2 使用 `dlopen` + `dlsym` 手动加载，因为它的 `JNI_OnLoad` 需要在特定时机被调用
- 跨线程事件使用 CAS 无锁链表队列（`Android_PushEvents`），避免 UI 线程阻塞
- 按键映射将 Android 的 10 个键码转换为 Cocos2d-x 的 `EventKeyboard::KeyCode`
- 触摸事件分为 Begin/Move/End/Cancel 四种，多点触控时使用数组传递
- 所有 JNI 回调都通过 `Android_PushEvents` 投递到 GL 线程，保证线程安全
- Google Breakpad 集成用于生产环境的崩溃转储收集

---

## 练习题与答案

### 题目 1：为什么 KrKr2 使用 `dlopen` 加载 SDL2，而不是直接在 CMake 中链接？

<details>
<summary>查看答案</summary>

因为库加载顺序问题。在 Android 上，`System.loadLibrary("krkr2")` 会先加载 `libkrkr2.so`，此时 SDL2 还没有初始化。SDL2 需要通过其 `JNI_OnLoad` 函数来注册自己的 native 方法，但这个函数只在库被 `System.loadLibrary` 加载时自动调用。

由于 `krkr2` 先于 `SDL2` 被加载，SDL2 的 `JNI_OnLoad` 还没有被调用，SDL2 的 native 方法没有注册。如果 KrKr2 此时调用 SDL2 的功能，就会导致 `UnsatisfiedLinkError`。

解决方案是在 `cocos_android_app_init` 中使用 `dlopen("libSDL2.so", RTLD_LAZY)` 手动加载 SDL2，然后用 `dlsym` 获取 `JNI_OnLoad` 的函数指针并手动调用，确保 SDL2 的 native 方法在需要之前完成注册。

</details>

### 题目 2：如果将 `Android_PushEvents` 中的 lambda 从按值捕获改为按引用捕获，会发生什么？请写出错误代码和正确代码。

<details>
<summary>查看答案</summary>

**错误代码**（按引用捕获）：

```cpp
JNIEXPORT void JNICALL nativeTouchesBegin(
    JNIEnv *env, jclass thiz, jint id, jfloat x, jfloat y) {
    intptr_t idlong = id;
    // ❌ 按引用捕获：idlong, x, y 是 JNI 函数的局部变量
    Android_PushEvents([&idlong, &x, &y]() {
        // 当 GL 线程执行这个 lambda 时，
        // nativeTouchesBegin 已经返回，
        // idlong, x, y 所在的栈帧已经被回收
        // 访问这些引用 → 未定义行为（通常是崩溃或读到垃圾数据）
        cocos2d::Director::getInstance()
            ->getOpenGLView()
            ->handleTouchesBegin(1, (intptr_t *)&idlong,
                                 (float *)&x, (float *)&y);
    });
    // 函数返回，idlong, x, y 被销毁
}
```

**正确代码**（按值捕获）：

```cpp
JNIEXPORT void JNICALL nativeTouchesBegin(
    JNIEnv *env, jclass thiz, jint id, jfloat x, jfloat y) {
    intptr_t idlong = id;
    // ✅ 按值捕获：lambda 内部有 idlong, x, y 的独立副本
    Android_PushEvents([idlong, x, y]() {
        // idlong, x, y 是 lambda 自己的成员，
        // 不依赖外部栈帧，GL 线程安全访问
        cocos2d::Director::getInstance()
            ->getOpenGLView()
            ->handleTouchesBegin(1, (intptr_t *)&idlong,
                                 (float *)&x, (float *)&y);
    });
}
```

关键原因：`Android_PushEvents` 将 lambda 入队后立即返回，但 lambda 要到下一帧的 `_processEvents` 才被执行。在此期间，原函数的栈帧已被回收。按引用捕获意味着 lambda 内部持有已失效的指针（悬垂引用），访问它们是未定义行为。

</details>

### 题目 3：请解释 `compare_exchange_weak` 在事件队列中的作用，并说明为什么用 `weak` 而不是 `strong`。

<details>
<summary>查看答案</summary>

`compare_exchange_weak` 是 CAS（Compare-And-Swap）操作的弱版本。在事件队列中：

```cpp
void Android_PushEvents(const std::function<void()> &func) {
    _eventQueueNode *node = new _eventQueueNode;
    node->func = func;
    node->prev = nullptr;
    // CAS 循环
    while (!_lastQueuedEvents.compare_exchange_weak(node->prev, node)) {
    }
}
```

**工作原理**：
1. `node->prev` 初始为 `nullptr`
2. `compare_exchange_weak` 检查 `_lastQueuedEvents` 是否等于 `node->prev`
3. 如果相等：将 `_lastQueuedEvents` 更新为 `node`，返回 `true`（成功入队）
4. 如果不等：将 `node->prev` 更新为 `_lastQueuedEvents` 的当前值，返回 `false`（重试）

**作用**：保证在多线程并发入队时，每个节点都能正确链接到队列中，不会丢失任何事件。

**为什么用 `weak` 而不是 `strong`**：
- `compare_exchange_weak` 可能"虚假失败"（spurious failure），即使值匹配也可能返回 `false`
- 但在循环中使用时，虚假失败只是多循环一次，不影响正确性
- `weak` 版本在某些 CPU 架构（如 ARM）上比 `strong` 更高效，因为 `strong` 需要额外的指令来防止虚假失败
- KrKr2 运行在 Android 上，主要目标是 ARM 架构，所以用 `weak` 是性能最优选择

</details>

---

## 下一步

恭喜你完成了 KrKr2 Android 启动流程与输入事件的学习！下一节我们将进入实战环节：

→ [动手实践与总结](./05-动手实践与总结.md)

在下一节中，你将综合运用前四节学到的 JNI 知识，自己动手实现一个完整的 Android NDK 项目，包括 JNI 类型转换、函数注册、引用管理和跨线程通信。
