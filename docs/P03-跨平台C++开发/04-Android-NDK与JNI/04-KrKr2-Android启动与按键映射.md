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

