# SDL2 与 Cocos2d-x 桥接

> **所属模块：** P11-SDL2跨平台开发
> **前置知识：** [01-KrKr2中的SDL2集成架构](./01-KrKr2中的SDL2集成架构.md)、[P10-Cocos2d-x框架](../../P10-Cocos2dx框架/README.md)
> **预计阅读时间：** 22 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 Cocos2d-x 的 GLViewImpl 如何封装 SDL2 的窗口和 GL 上下文
2. 分析事件从 SDL2 到 Cocos2d-x EventDispatcher 的转发流程
3. 理解音频从 SDL2 Audio 到 Cocos2d-x AudioEngine 的桥接方式
4. 掌握如何在 Cocos2d-x 项目中自定义 SDL2 行为

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| GLViewImpl | GL View Implementation | Cocos2d-x 中 GLView 的 SDL2 平台具体实现类，封装了窗口创建和 GL 上下文管理 |
| EventDispatcher | Event Dispatcher | Cocos2d-x 的事件分发系统，将底层事件（触摸、键盘）路由到注册了监听器的节点 |
| EventListenerKeyboard | Keyboard Event Listener | Cocos2d-x 中监听键盘事件的类，开发者通过它接收按键事件 |
| AudioEngine | Audio Engine | Cocos2d-x 的音频播放接口，底层可使用 SDL2 Audio、OpenAL 或平台原生 API |
| 帧回调 | Frame Callback | Cocos2d-x 的 Director 在每帧调用的回调链，包括事件处理、逻辑更新、渲染 |

## GLViewImpl：窗口与 GL 上下文桥接

Cocos2d-x 定义了一个抽象基类 `GLView`，声明了窗口操作的接口（设置尺寸、获取帧率、处理触摸等）。`GLViewImpl` 是这个接口在 SDL2 平台上的具体实现。

### GLViewImpl 的生命周期

```
GLViewImpl::create("KrKr2")
    │
    ├─ 1. SDL_Init(SDL_INIT_VIDEO)
    │      初始化 SDL2 视频子系统
    │
    ├─ 2. SDL_GL_SetAttribute(...)
    │      设置 OpenGL 属性：
    │      • GL 版本（桌面 4.1 / ES 2.0）
    │      • 颜色深度（RGBA 各 8 位）
    │      • 双缓冲
    │      • 深度缓冲位数
    │
    ├─ 3. SDL_CreateWindow(title, x, y, w, h, flags)
    │      创建窗口，flags 包含 SDL_WINDOW_OPENGL
    │
    ├─ 4. SDL_GL_CreateContext(window)
    │      创建 OpenGL 上下文并绑定到窗口
    │
    ├─ 5. gladLoadGLLoader / glewInit
    │      加载 OpenGL 函数指针
    │
    └─ 6. 返回 GLViewImpl 实例
           Director 持有此实例，在帧循环中使用
```

### 核心代码分析

以下是 Cocos2d-x GLViewImpl（SDL2 后端）的关键方法，展示了桥接是如何实现的：

```cpp
// Cocos2d-x 框架代码（非 KrKr2 代码）——展示桥接原理
// 文件位置：cocos/platform/desktop/CCGLViewImpl-desktop.cpp（SDL2 版本）

class GLViewImpl : public GLView {
private:
    SDL_Window* _window;      // SDL2 窗口句柄
    SDL_GLContext _glContext;  // SDL2 GL 上下文

public:
    // 创建窗口和 GL 上下文
    bool initWithRect(const std::string& viewName, Rect rect, float frameZoomFactor) {
        // 设置 GL 属性
        SDL_GL_SetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, 2);  // OpenGL ES 2.0
        SDL_GL_SetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, 0);
        SDL_GL_SetAttribute(SDL_GL_DOUBLEBUFFER, 1);           // 双缓冲
        SDL_GL_SetAttribute(SDL_GL_DEPTH_SIZE, 24);            // 24 位深度

        // 创建窗口
        _window = SDL_CreateWindow(
            viewName.c_str(),
            SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED,
            (int)rect.size.width, (int)rect.size.height,
            SDL_WINDOW_OPENGL | SDL_WINDOW_SHOWN | SDL_WINDOW_RESIZABLE
        );
        if (!_window) {
            // SDL2 错误：SDL_GetError() 获取详细信息
            return false;
        }

        // 创建 GL 上下文
        _glContext = SDL_GL_CreateContext(_window);
        if (!_glContext) {
            SDL_DestroyWindow(_window);
            return false;
        }

        // VSync 设置：0=关闭 1=开启 -1=自适应
        SDL_GL_SetSwapInterval(1);

        return true;
    }

    // 每帧结束时交换缓冲区
    void swapBuffers() override {
        SDL_GL_SwapWindow(_window);  // 将后缓冲区显示到屏幕
    }

    // 窗口大小改变时的回调
    void setFrameSize(float width, float height) override {
        GLView::setFrameSize(width, height);
        SDL_SetWindowSize(_window, (int)width, (int)height);
    }

    // 全屏切换
    void setFullscreen(bool fullscreen) {
        SDL_SetWindowFullscreen(_window,
            fullscreen ? SDL_WINDOW_FULLSCREEN_DESKTOP : 0);
    }

    ~GLViewImpl() {
        SDL_GL_DeleteContext(_glContext);
        SDL_DestroyWindow(_window);
    }
};
```

**关键方法映射表：**

| Cocos2d-x 接口 (GLView) | SDL2 底层调用 | 作用 |
|--------------------------|--------------|------|
| `initWithRect()` | `SDL_CreateWindow` + `SDL_GL_CreateContext` | 创建窗口和 GL 上下文 |
| `swapBuffers()` | `SDL_GL_SwapWindow` | 双缓冲交换，显示当前帧 |
| `setFrameSize()` | `SDL_SetWindowSize` | 调整窗口大小 |
| `pollEvents()` | `SDL_PollEvent` | 获取输入事件 |
| 析构函数 | `SDL_DestroyWindow` + `SDL_GL_DeleteContext` | 清理资源 |

## 事件桥接：SDL2 → Cocos2d-x

Cocos2d-x 有自己的事件系统（`EventDispatcher`），与 SDL2 的事件队列是独立的。桥接层的工作是：从 SDL2 事件队列取出事件 → 转换为 Cocos2d-x 事件对象 → 派发给 Cocos2d-x 的 EventDispatcher。

### 事件转换流程

```
SDL2 事件队列
    │ SDL_PollEvent()
    ▼
GLViewImpl::pollEvents()
    │ switch (event.type)
    ├─ SDL_KEYDOWN/UP     → EventKeyboard → EventDispatcher
    ├─ SDL_MOUSEBUTTONDOWN → EventTouch   → EventDispatcher
    ├─ SDL_MOUSEMOTION     → EventMouse   → EventDispatcher
    ├─ SDL_FINGERDOWN/UP   → EventTouch   → EventDispatcher
    ├─ SDL_WINDOWEVENT     → GLView 内部处理（调整视口等）
    └─ SDL_QUIT            → Director::end()
```

### 键盘事件转换

```cpp
// 示例 1：SDL2 键盘事件 → Cocos2d-x EventKeyboard（桥接层代码简化）
void GLViewImpl::pollEvents() {
    SDL_Event event;
    while (SDL_PollEvent(&event)) {
        switch (event.type) {
        case SDL_KEYDOWN:
        case SDL_KEYUP: {
            // 1. SDL2 keycode → Cocos2d-x KeyCode 映射
            EventKeyboard::KeyCode ccKey = sdlKeyToCocosKey(event.key.keysym.sym);
            
            // 2. 创建 Cocos2d-x 键盘事件
            EventKeyboard ccEvent(ccKey, event.type == SDL_KEYDOWN);
            
            // 3. 通过 EventDispatcher 派发
            auto dispatcher = Director::getInstance()->getEventDispatcher();
            dispatcher->dispatchEvent(&ccEvent);
            break;
        }
        // ... 其他事件类型
        }
    }
}

// SDL2 KeyCode → Cocos2d-x KeyCode 映射表（部分）
EventKeyboard::KeyCode sdlKeyToCocosKey(SDL_Keycode sdlKey) {
    switch (sdlKey) {
        case SDLK_RETURN:    return EventKeyboard::KeyCode::KEY_ENTER;
        case SDLK_ESCAPE:    return EventKeyboard::KeyCode::KEY_ESCAPE;
        case SDLK_SPACE:     return EventKeyboard::KeyCode::KEY_SPACE;
        case SDLK_LEFT:      return EventKeyboard::KeyCode::KEY_LEFT_ARROW;
        case SDLK_RIGHT:     return EventKeyboard::KeyCode::KEY_RIGHT_ARROW;
        case SDLK_UP:        return EventKeyboard::KeyCode::KEY_UP_ARROW;
        case SDLK_DOWN:      return EventKeyboard::KeyCode::KEY_DOWN_ARROW;
        // ... 完整映射约 100+ 个键
        default:             return EventKeyboard::KeyCode::KEY_NONE;
    }
}
```

### 触摸/鼠标事件转换

桌面平台上，Cocos2d-x 将鼠标事件同时转换为触摸事件和鼠标事件，以兼容触摸屏设计的 UI：

```cpp
// 示例 2：鼠标事件的双重转发
case SDL_MOUSEBUTTONDOWN: {
    float x = event.button.x;
    float y = event.button.y;
    
    // 转为 Cocos2d-x 坐标（左下角为原点，SDL2 是左上角）
    y = _screenSize.height - y;
    
    // 1. 作为鼠标事件派发
    EventMouse mouseEvent(EventMouse::MouseEventType::MOUSE_DOWN);
    mouseEvent.setCursorPosition(x, y);
    mouseEvent.setMouseButton((EventMouse::MouseButton)event.button.button);
    dispatcher->dispatchEvent(&mouseEvent);
    
    // 2. 同时作为触摸事件派发（兼容触摸 UI）
    if (event.button.button == SDL_BUTTON_LEFT) {
        Touch* touch = new Touch();
        touch->setTouchInfo(0, x, y);  // ID=0 模拟单点触摸
        
        EventTouch touchEvent;
        touchEvent.setEventCode(EventTouch::EventCode::BEGAN);
        touchEvent.setTouches({touch});
        dispatcher->dispatchEvent(&touchEvent);
    }
    break;
}
```

**坐标系差异：** SDL2 使用屏幕坐标系（左上角为原点，Y 轴向下），Cocos2d-x 使用 OpenGL 坐标系（左下角为原点，Y 轴向上）。桥接层必须做 `y = screenHeight - y` 的转换。

## 帧循环桥接

Cocos2d-x 的 Director 驱动整个帧循环。在 SDL2 后端，帧循环的结构是：

```cpp
// 示例 3：Director 帧循环中的 SDL2 集成点
// 文件：cocos/base/CCDirector.cpp（简化）

void Director::mainLoop() {
    while (!_purgeDirectorInNextLoop) {
        // 1. 处理 SDL2 事件 → 转为 Cocos2d-x 事件
        _openGLView->pollEvents();  // 调用 GLViewImpl::pollEvents()
        
        // 2. 计算 deltaTime
        calculateDeltaTime();
        
        // 3. 调度器更新（定时器、动画、Action）
        _scheduler->update(_deltaTime);
        
        // 4. 事件派发
        _eventDispatcher->dispatchEvent(_eventAfterUpdate);
        
        // 5. 渲染
        drawScene();
        
        // 6. 交换缓冲区（SDL2 调用）
        _openGLView->swapBuffers();  // 调用 SDL_GL_SwapWindow()
    }
}
```

每一帧的 SDL2 参与点：
1. **帧开头**：`pollEvents()` → `SDL_PollEvent()` 获取输入
2. **帧结尾**：`swapBuffers()` → `SDL_GL_SwapWindow()` 显示画面
3. **帧率控制**：`SDL_Delay()` 或 VSync（`SDL_GL_SetSwapInterval`）限制帧率

## 音频桥接

Cocos2d-x 的音频系统（`AudioEngine`）在不同平台使用不同后端。在使用 SDL2 的平台上，可选择 SDL2_mixer 作为音频后端：

| 平台 | 音频后端 | SDL2 参与 |
|------|---------|-----------|
| Linux | SDL2_mixer / OpenAL | ✅ SDL2_mixer 基于 SDL2 Audio |
| Windows | XAudio2 / WASAPI | ❌ 不使用 SDL2 |
| macOS | OpenAL / AVFoundation | ❌ 不使用 SDL2 |
| Android | OpenSL ES / AAudio | ❌ 不使用 SDL2 |

```cpp
// 示例 4：KrKr2 音频播放路径（Linux 上）
// KrKr2 引擎代码 → Cocos2d-x AudioEngine → SDL2_mixer → SDL2 Audio → ALSA/PulseAudio

// KrKr2 层面——通过 Cocos2d-x API 播放音频
int audioId = AudioEngine::play2d("bgm.ogg", true, 0.8f);
// 参数：文件路径，是否循环，音量

// Cocos2d-x 内部（SDL2_mixer 后端）
// Mix_PlayMusic(music, -1);     // SDL2_mixer 播放
// Mix_VolumeMusic(volume * 128); // 设置音量

// SDL2 层面
// SDL_OpenAudioDevice(...)      // 打开音频设备
// SDL_QueueAudio(...)           // 送入音频数据
```

**KrKr2 的音频系统实际上更复杂：** KrKr2 有自己的音频解码和播放管线（见 M05 音频子系统），它使用 FFmpeg 解码音频、自建 DSP 处理链、通过平台 API 输出。Cocos2d-x 的 AudioEngine 主要用于 UI 音效，游戏 BGM 和语音走 KrKr2 自己的管线。

## 实战：自定义 SDL2 行为

虽然 KrKr2 不直接调用 SDL2，但你可以通过修改 Cocos2d-x 的 GLViewImpl 来定制 SDL2 行为：

```cpp
// 示例 5：自定义 GLViewImpl 添加窗口图标
class CustomGLView : public GLViewImpl {
public:
    static CustomGLView* create(const std::string& name) {
        auto view = new CustomGLView();
        if (view->initWithRect(name, Rect(0, 0, 960, 640), 1.0f)) {
            // 在窗口创建后设置图标
            SDL_Surface* icon = SDL_LoadBMP("icon.bmp");
            if (icon) {
                SDL_SetWindowIcon(view->getSDLWindow(), icon);
                SDL_FreeSurface(icon);
            }
            view->autorelease();
            return view;
        }
        delete view;
        return nullptr;
    }

    // 暴露 SDL_Window 指针供外部使用
    SDL_Window* getSDLWindow() const { return _window; }
};

// 在 AppDelegate 中使用自定义 GLView
bool TVPAppDelegate::applicationDidFinishLaunching() {
    auto director = Director::getInstance();
    auto glview = CustomGLView::create("KrKr2");
    director->setOpenGLView(glview);
    // ...
}
```

```cpp
// 示例 6：在 Cocos2d-x 中访问 SDL2 原始事件
// 方法：在 GLViewImpl::pollEvents() 中注入自定义处理

class CustomGLView : public GLViewImpl {
public:
    void pollEvents() override {
        SDL_Event event;
        while (SDL_PollEvent(&event)) {
            // 自定义处理：捕获截图快捷键
            if (event.type == SDL_KEYDOWN &&
                event.key.keysym.sym == SDLK_F12) {
                captureScreenshot();
                continue;  // 不传递给 Cocos2d-x
            }
            // 其余事件正常传递
            handleSDLEvent(event);  // 调用父类的事件处理
        }
    }
};
```

## 常见错误与排查

### 错误 1：坐标系不匹配

```cpp
// ❌ 直接使用 SDL2 鼠标坐标传给 Cocos2d-x 节点
int mouseY = event.button.y;  // SDL2：左上角为原点
node->setPosition(mouseX, mouseY);  // Cocos2d-x：左下角为原点

// ✅ 转换坐标系
int mouseY = screenHeight - event.button.y;
node->setPosition(mouseX, mouseY);
```

### 错误 2：在 Cocos2d-x 回调中调用 SDL2 函数

```cpp
// ❌ 在 Cocos2d-x 触摸回调中直接调用 SDL2——破坏抽象层
auto listener = EventListenerTouchOneByOne::create();
listener->onTouchBegan = [](Touch* touch, Event* event) {
    SDL_ShowSimpleMessageBox(SDL_MESSAGEBOX_INFO, "Hi", "Touched!", nullptr);
    // 问题：SDL2 函数不应在 Cocos2d-x 层调用
    return true;
};

// ✅ 使用 Cocos2d-x 自己的 UI 系统
listener->onTouchBegan = [](Touch* touch, Event* event) {
    auto label = Label::createWithSystemFont("Touched!", "Arial", 24);
    // 使用 Cocos2d-x UI 元素
    return true;
};
```

### 错误 3：SDL2 和 Cocos2d-x 同时处理同一事件

```cpp
// ❌ 用 SDL_SetEventFilter 吃掉事件后，Cocos2d-x 收不到
SDL_SetEventFilter([](void*, SDL_Event* e) {
    if (e->type == SDL_KEYDOWN) return 0;  // 所有键盘事件被丢弃
    return 1;
}, nullptr);
// 结果：Cocos2d-x 的 EventListenerKeyboard 永远收不到按键

// ✅ 如果要拦截事件，在 Cocos2d-x 的 EventDispatcher 层面做
auto listener = EventListenerKeyboard::create();
listener->onKeyPressed = [](EventKeyboard::KeyCode key, Event* event) {
    event->stopPropagation();  // 阻止事件继续传播
};
```

## 本节小结

- GLViewImpl 是 Cocos2d-x 到 SDL2 的核心桥接类，封装了窗口和 GL 上下文
- 事件桥接链路：`SDL_PollEvent` → `GLViewImpl::pollEvents` → 键码映射 → `EventDispatcher`
- 坐标系转换是桥接中最常出错的地方（SDL2 左上 vs Cocos2d-x 左下）
- 帧循环中 SDL2 参与两个关键点：帧开头的事件轮询和帧结尾的缓冲区交换
- 音频桥接仅在 Linux 上通过 SDL2_mixer，其他平台使用原生 API
- 自定义 SDL2 行为需要继承 GLViewImpl，不要在 Cocos2d-x 层直接调 SDL2 函数

## 练习题与答案

### 题目 1：说明为什么 Cocos2d-x 需要坐标系转换

<details>
<summary>查看答案</summary>

SDL2 和大多数窗口系统使用**屏幕坐标系**：原点在左上角，X 轴向右，Y 轴**向下**。这与显示器的扫描线方向一致（从上到下逐行扫描）。

Cocos2d-x 使用 **OpenGL 坐标系**：原点在左下角，X 轴向右，Y 轴**向上**。这与数学中的笛卡尔坐标系一致，也是 OpenGL 的标准坐标系。

当 SDL2 报告鼠标点击位置 `(100, 50)` 时，如果屏幕高度是 600：
- SDL2 含义：距左上角右移 100，**下移** 50
- Cocos2d-x 含义需要：距左下角右移 100，**上移** 550（即 600 - 50）

转换公式：`cocos_y = screen_height - sdl_y`

如果不做转换，用户点击屏幕上方时 Cocos2d-x 会认为点击在下方，导致按钮等 UI 元素无法正确响应。

</details>

### 题目 2：如何在不修改 Cocos2d-x 源码的情况下添加全局按键拦截？

<details>
<summary>查看答案</summary>

使用 Cocos2d-x 的 `EventDispatcher` 注册一个高优先级的键盘监听器：

```cpp
auto listener = EventListenerKeyboard::create();
listener->onKeyPressed = [](EventKeyboard::KeyCode key, Event* event) {
    if (key == EventKeyboard::KeyCode::KEY_F12) {
        // 截图逻辑
        utils::captureScreen([](bool success, const std::string& path) {
            if (success) printf("截图保存到: %s\n", path.c_str());
        }, "screenshot.png");
        event->stopPropagation();  // 阻止事件继续传播
    }
};

// 设置最高优先级（数字越小优先级越高）
Director::getInstance()->getEventDispatcher()
    ->addEventListenerWithFixedPriority(listener, -1);
```

**不要**使用 `SDL_SetEventFilter` 来拦截——那会在 SDL2 层面丢弃事件，Cocos2d-x 完全看不到。应该在 Cocos2d-x 的 EventDispatcher 层面用 `stopPropagation()` 来阻止事件传播。

</details>

### 题目 3：Director::mainLoop 中 SDL2 参与了哪些步骤？

<details>
<summary>查看答案</summary>

Director 的帧循环中，SDL2 直接参与两个步骤：

1. **帧开头——事件轮询**：
   - `_openGLView->pollEvents()` → `SDL_PollEvent()`
   - 从 SDL2 事件队列取出所有待处理事件
   - 转换为 Cocos2d-x 事件并派发

2. **帧结尾——缓冲区交换**：
   - `_openGLView->swapBuffers()` → `SDL_GL_SwapWindow()`
   - 将后缓冲区（当前帧的渲染结果）显示到屏幕
   - 如果开启了 VSync（`SDL_GL_SetSwapInterval(1)`），此调用会阻塞到下一个垂直同步信号

间接参与：
- 帧率控制：如果不使用 VSync，可通过 `SDL_Delay()` 实现
- GL 函数调用：所有 `glDrawArrays`、`glClear` 等都通过 SDL2 创建的 GL 上下文执行

</details>

## 下一步

你已理解 Cocos2d-x 如何桥接 SDL2。→ 继续阅读 [03-扩展SDL2功能实践](./03-扩展SDL2功能实践.md)，学习如何在 KrKr2 项目中扩展 SDL2 功能并进行性能调优。
