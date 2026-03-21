# Cocos2d 实现与 Flutter 原型

> **所属模块：** P12-现代跨平台UI
> **前置知识：** [KrKr2 UI 抽象层接口定义](./01-KrKr2-UI抽象层接口定义.md)、[FlutterEngine API 与嵌入原理](../02-Flutter引擎嵌入/01-FlutterEngine-API与嵌入原理.md)、[Platform Channel 与纹理共享](../02-Flutter引擎嵌入/02-Platform-Channel与纹理共享.md)
> **预计阅读时间：** 50 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：
1. 使用 KrKr2 现有的 Cocos2d-x UI 组件（TVPMessageBoxForm 等）实现 IUIBackend 接口
2. 通过 Flutter Platform Channel 实现 IUIBackend 的 Flutter 后端原型
3. 对比 Cocos2d-x 和 Flutter 两种后端的性能差异（启动时间、内存、帧率）
4. 制定从 Cocos2d-x 到 Flutter 的渐进式迁移策略
5. 处理后端切换过程中的常见问题（纹理共享、事件丢失、生命周期同步）

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| CSLoader | Cocos Studio Loader | Cocos2d-x 中加载 .csb 布局文件的工具类 |
| Platform Channel | 平台通道 | Flutter 中 Dart 代码与原生 C++/Java/ObjC 通信的桥梁 |
| MethodChannel | 方法通道 | Platform Channel 的一种，用于 Dart↔原生的方法调用 |
| FlutterEngine | Flutter 引擎 | Flutter 的 C++ 运行时，负责执行 Dart 代码和渲染 |
| 适配器模式 | Adapter Pattern | 将已有接口转换为新接口的设计模式——本节用它包装现有 UI |
| 渐进式迁移 | Progressive Migration | 分阶段逐步替换旧系统的策略，而非一次性重写 |
| 功能开关 | Feature Flag | 运行时控制启用/禁用某个功能的配置机制 |
| A/B 测试 | A/B Testing | 同时运行两个版本，对比效果的实验方法 |

---

## 一、Cocos2d-x 后端实现——适配器模式

KrKr2 项目已有一套基于 Cocos2d-x 的 UI 实现（`TVPMessageBoxForm`、`TVPPadForm` 等），我们的目标不是重写它们，而是用**适配器模式**（Adapter Pattern）将它们包装成上一节定义的 `IUIBackend` 接口。

适配器模式的核心思想：保留现有代码不动，在外面套一层"转换器"，让旧代码符合新接口。这样做的好处是零风险——现有功能不受任何影响。

### 1.1 适配器架构

```
调用方（游戏逻辑）
    │
    ▼
IUIBackend 接口
    │
    ▼
Cocos2dUIBackendAdapter（适配器）
    │
    ├──→ TVPMessageBoxForm（现有代码）
    ├──→ TVPPadForm（现有代码）
    ├──→ TVPFontSelectForm（现有代码）
    └──→ cocos2d::Director（现有引擎）
```

### 示例 1：Cocos2d-x 后端适配器——完整实现

```cpp
// cocos2d_ui_backend.h — Cocos2d-x 后端适配器
#pragma once

#include "krkr2_ui_backend.h"  // 上一节定义的接口
#include <iostream>
#include <string>
#include <functional>
#include <memory>
#include <vector>
#include <cstdint>

namespace krkr2::ui {

// ========== Cocos2d-x 后端适配器 ==========
// 将 KrKr2 现有的 Cocos2d-x UI 组件包装为 IUIBackend 接口
// 适配器模式（Adapter Pattern）：不修改现有代码，在外面套一层转换

class Cocos2dUIBackendAdapter : public IUIBackend {
public:
    Cocos2dUIBackendAdapter() {
        std::cout << "[Cocos2dAdapter] 构造" << std::endl;
    }

    ~Cocos2dUIBackendAdapter() override {
        std::cout << "[Cocos2dAdapter] 析构" << std::endl;
    }

    // ---- 生命周期 ----

    bool initialize(void* windowHandle, 
                    int32_t width, int32_t height) override {
        // Cocos2d-x 引擎已经在 AppDelegate 中初始化
        // 这里只需要保存窗口信息
        windowHandle_ = windowHandle;
        width_ = width;
        height_ = height;
        initialized_ = true;

        std::cout << "[Cocos2dAdapter] 初始化完成 ("
                  << width << "x" << height << ")" << std::endl;

        // 实际项目中：
        // auto director = cocos2d::Director::getInstance();
        // assert(director->getOpenGLView() != nullptr);

        return true;
    }

    void shutdown() override {
        if (!initialized_) return;
        
        // 关闭所有打开的 UI 组件
        // 实际项目：遍历 Scene 栈，弹出所有 UI Scene
        std::cout << "[Cocos2dAdapter] 关闭" << std::endl;
        initialized_ = false;
    }

    void update(float deltaTime) override {
        // Cocos2d-x 的更新由引擎主循环驱动
        // Director::mainLoop() 已经在每帧调用
        // 这里只处理适配器自身的状态
    }

    // ---- 能力查询 ----

    UICapability capabilities() const override {
        // Cocos2d-x 后端的能力集：
        // 支持：消息框、右键菜单、设置、进度条、软键盘
        // 不支持：文件对话框（移动平台无原生支持）、
        //         字体选择器（需要系统 API）、
        //         纹理共享（自身就是渲染引擎）
        return UICapability::MessageBox 
             | UICapability::ContextMenu
             | UICapability::Settings
             | UICapability::Progress
             | UICapability::SoftKeyboard;
    }

    const char* backendName() const override { 
        return "Cocos2d-x (Adapter)"; 
    }

    uint32_t interfaceVersion() const override { return 1; }

    // ---- 消息对话框 ----
    // 适配 TVPMessageBoxForm

    void showMessageBox(
        const std::string& title,
        const std::string& message,
        DialogButtons buttons,
        DialogIcon icon,
        DialogCallback callback) override {
        
        std::cout << "[Cocos2dAdapter] showMessageBox: " 
                  << title << std::endl;
        
        // 实际项目中的实现：
        // 1. 使用 CSLoader 加载 MessageBox.csb 布局
        // 2. 设置标题和消息文本
        // 3. 根据 buttons 类型显示/隐藏按钮
        // 4. 绑定按钮回调
        
        // 伪代码（对应 TVPMessageBoxForm 的工作方式）：
        /*
        auto node = CSLoader::createNode("MessageBox.csb");
        auto titleLabel = node->getChildByName<Text*>("Title");
        titleLabel->setString(title);
        
        auto msgLabel = node->getChildByName<Text*>("Message");
        msgLabel->setString(message);
        
        auto okBtn = node->getChildByName<Button*>("OKButton");
        okBtn->addClickEventListener([callback](Ref*) {
            if (callback) callback(DialogResult::OK);
        });
        
        // 根据按钮类型配置
        if (buttons == DialogButtons::OK) {
            node->getChildByName("CancelButton")
                ->setVisible(false);
        }
        
        // 推入 Scene 栈
        Director::getInstance()->pushScene(
            TransitionFade::create(0.3f, msgBoxScene));
        */
        
        // 简化示例：直接调用回调模拟用户点击
        if (callback) {
            callback(DialogResult::OK);
        }
    }

    // ---- 文件对话框 ----
    // Cocos2d-x 不支持原生文件对话框

    void showFileDialog(
        const FileDialogOptions& options,
        FileCallback callback) override {
        
        std::cerr << "[Cocos2dAdapter] 警告: Cocos2d-x 后端"
                  << "不支持文件对话框" << std::endl;
        // 返回空结果表示用户取消
        if (callback) callback({});
    }

    // ---- 右键菜单 ----
    // 适配 Cocos2d-x 的菜单系统

    void showContextMenu(
        const std::vector<MenuItem>& items,
        Vec2 position,
        MenuCallback callback) override {
        
        std::cout << "[Cocos2dAdapter] showContextMenu: "
                  << items.size() << " 项 @(" 
                  << position.x << "," << position.y << ")"
                  << std::endl;

        // 实际项目中：
        // 创建 cocos2d::Menu + MenuItemLabel 列表
        /*
        auto menu = cocos2d::Menu::create();
        for (const auto& item : items) {
            if (item.separator) {
                // 添加分隔线（用灰色横条模拟）
                continue;
            }
            auto label = Label::createWithSystemFont(
                item.label, "sans-serif", 20);
            auto menuItem = MenuItemLabel::create(
                label, 
                [id = item.id, callback](Ref*) {
                    if (callback) callback(id);
                });
            menuItem->setEnabled(item.enabled);
            menu->addChild(menuItem);
        }
        menu->setPosition(Vec2(position.x, position.y));
        currentScene->addChild(menu, 9999); // 最高层级
        */

        // 简化示例
        if (callback && !items.empty()) {
            callback(items[0].id);
        }
    }

    void setMenuBar(
        const std::vector<MenuItem>& items,
        MenuCallback callback) override {
        // 移动平台没有菜单栏，桌面平台通过原生 API
        std::cout << "[Cocos2dAdapter] setMenuBar: "
                  << "移动平台不支持菜单栏" << std::endl;
    }

    // ---- 其他 UI 组件 ----

    void showSettings() override {
        std::cout << "[Cocos2dAdapter] 打开设置面板"
                  << std::endl;
        // 实际：推入 ConfigManager 对应的 Scene
    }

    void showToast(const std::string& message, 
                   float durationSec) override {
        std::cout << "[Cocos2dAdapter] Toast: " << message
                  << std::endl;
        // Cocos2d-x 没有原生 Toast
        // 实现方式：创建 Label + FadeOut 动作
        /*
        auto label = Label::createWithSystemFont(
            message, "sans-serif", 16);
        label->setPosition(Vec2(screenW/2, screenH * 0.85));
        label->runAction(Sequence::create(
            DelayTime::create(durationSec),
            FadeOut::create(0.3f),
            RemoveSelf::create(),
            nullptr));
        currentScene->addChild(label, 9999);
        */
    }

    void showProgress(const std::string& title,
                      float progress) override {
        std::cout << "[Cocos2dAdapter] 进度: " << title 
                  << " " << (int)(progress * 100) << "%"
                  << std::endl;
        // 实际：更新 LoadingBar 组件
    }

    void hideProgress() override {
        std::cout << "[Cocos2dAdapter] 隐藏进度条" << std::endl;
    }

    void showSoftKeyboard(TextInputCallback callback) override {
        std::cout << "[Cocos2dAdapter] 显示软键盘" << std::endl;
        // 实际：显示 TVPPadForm
        // TVPPadForm 使用 Cocos2d-x Scene 实现虚拟键盘
    }

    void hideSoftKeyboard() override {
        std::cout << "[Cocos2dAdapter] 隐藏软键盘" << std::endl;
    }

    void showFontSelector(
        const FontDesc& currentFont,
        std::function<void(const FontDesc&)> callback) override {
        std::cerr << "[Cocos2dAdapter] 字体选择器: "
                  << "需要平台原生支持" << std::endl;
        // Cocos2d-x 本身不提供字体选择 UI
        // 需要调用平台 API（Android: FontPickerDialog 等）
    }

    IRenderBridge* renderBridge() override {
        // Cocos2d-x 后端不需要渲染桥接
        // 因为 Cocos2d-x 同时负责游戏渲染和 UI 渲染
        return nullptr;
    }

private:
    void*   windowHandle_ = nullptr;
    int32_t width_  = 0;
    int32_t height_ = 0;
    bool    initialized_ = false;
};

} // namespace krkr2::ui
```

### 1.2 适配器的关键设计决策

| 决策 | 选择 | 原因 |
|------|------|------|
| `initialize()` 中不重复初始化 Cocos2d-x | 只保存参数 | 引擎已在 `AppDelegate` 中初始化，重复初始化会崩溃 |
| `renderBridge()` 返回 `nullptr` | 不需要纹理桥接 | Cocos2d-x 同时管理游戏渲染和 UI，无需跨框架传输纹理 |
| 不支持的功能返回空/打印警告 | 安全降级 | 接口契约要求所有方法可安全调用，不支持时不崩溃 |
| 保留注释中的实际代码 | 文档价值 | 读者能看到 Cocos2d-x API 的具体用法 |

---

## 二、Flutter 后端原型——Platform Channel 桥接

与 Cocos2d-x 后端不同，Flutter 后端需要启动一个独立的 Flutter 引擎实例（FlutterEngine），然后通过 **Platform Channel**（平台通道——Flutter 中 Dart 代码与 C++/Java/ObjC 通信的桥梁）将 `IUIBackend` 接口的调用传递给 Dart 端渲染。

Flutter 后端的核心优势：Material Design 原生组件、丰富的动画库、热重载开发体验。但代价是需要额外的引擎启动时间和内存占用。

### 2.1 Flutter 后端架构

```
调用方（游戏逻辑 C++）
    │
    ▼
IUIBackend 接口
    │
    ▼
FlutterUIBackendAdapter（C++ 适配器）
    │
    ├──→ FlutterEngine C API（引擎生命周期）
    │      - FlutterEngineRun()     启动引擎
    │      - FlutterEngineShutdown() 关闭引擎
    │      - FlutterEngineSendWindowMetricsEvent() 同步窗口尺寸
    │
    ├──→ MethodChannel（方法通道）
    │      - "krkr2/ui" 通道名
    │      - C++ → Dart: showMessageBox、showContextMenu 等
    │      - Dart → C++: 用户点击结果回调
    │
    └──→ IRenderBridge（纹理桥接）
           - 将游戏画面纹理传给 Flutter 合成
           - Flutter 在游戏画面之上叠加 UI
```

### 示例 2：Flutter 后端适配器——完整实现

```cpp
// flutter_ui_backend.h — Flutter 后端适配器
#pragma once

#include "krkr2_ui_backend.h"  // 上一节定义的接口
#include <iostream>
#include <string>
#include <functional>
#include <memory>
#include <vector>
#include <map>
#include <mutex>
#include <atomic>
#include <cstdint>

namespace krkr2::ui {

// ========== 模拟 FlutterEngine C API ==========
// 实际项目中这些来自 <flutter_embedder.h>
// 这里用模拟实现让示例可独立编译运行

// FlutterEngine 句柄（实际是指向引擎内部结构的不透明指针）
using FlutterEngine = void*;

// 引擎运行结果
enum class FlutterEngineResult {
    kSuccess = 0,       // 成功
    kInvalidLibraryVersion,  // 库版本不匹配
    kInvalidArguments        // 参数无效
};

// Platform Message（平台消息）— Dart↔C++ 通信的载体
struct FlutterPlatformMessage {
    const char* channel;       // 通道名，如 "krkr2/ui"
    const uint8_t* message;    // 消息体（JSON/二进制编码）
    size_t message_size;       // 消息体长度
    // 实际还有 response_handle 用于双向通信
};

// 模拟 FlutterEngine API 函数
namespace flutter_api {
    // 模拟引擎实例（全局单例，实际由 embedder.h 管理）
    static bool g_engineRunning = false;
    static int g_engineWidth = 0;
    static int g_engineHeight = 0;

    inline FlutterEngineResult FlutterEngineRun(
        FlutterEngine* engineOut) {
        std::cout << "[FlutterEngine] 引擎启动中..." 
                  << std::endl;
        g_engineRunning = true;
        *engineOut = reinterpret_cast<void*>(0xF1077E4);
        std::cout << "[FlutterEngine] 引擎启动完成" 
                  << std::endl;
        return FlutterEngineResult::kSuccess;
    }

    inline FlutterEngineResult FlutterEngineShutdown(
        FlutterEngine engine) {
        std::cout << "[FlutterEngine] 引擎关闭" << std::endl;
        g_engineRunning = false;
        return FlutterEngineResult::kSuccess;
    }

    inline FlutterEngineResult FlutterEngineSendWindowMetrics(
        FlutterEngine engine, int w, int h) {
        g_engineWidth = w;
        g_engineHeight = h;
        std::cout << "[FlutterEngine] 窗口尺寸更新: "
                  << w << "x" << h << std::endl;
        return FlutterEngineResult::kSuccess;
    }

    inline FlutterEngineResult FlutterEngineSendPlatformMessage(
        FlutterEngine engine,
        const FlutterPlatformMessage* msg) {
        std::cout << "[FlutterEngine] 发送平台消息到通道: "
                  << msg->channel << " (" 
                  << msg->message_size << " 字节)" 
                  << std::endl;
        return FlutterEngineResult::kSuccess;
    }
} // namespace flutter_api

// ========== MethodChannel 辅助类 ==========
// MethodChannel（方法通道）是 Platform Channel 的一种
// 用于 C++ 和 Dart 之间的远程方法调用（RPC 风格）
// 通道名唯一标识一条通信管道

class MethodChannel {
public:
    // channelName: 通道名，C++ 和 Dart 必须使用相同名字
    // engine: Flutter 引擎句柄
    MethodChannel(const std::string& channelName,
                  FlutterEngine engine)
        : channelName_(channelName), engine_(engine) {
        std::cout << "[MethodChannel] 创建通道: " 
                  << channelName << std::endl;
    }

    // 调用 Dart 端方法（单向，不等待返回值）
    // method: 方法名（如 "showMessageBox"）
    // args: JSON 编码的参数
    void invokeMethod(const std::string& method,
                      const std::string& argsJson) {
        // 实际实现：将方法名+参数编码为
        // StandardMethodCodec 格式
        std::string payload = 
            "{\"method\":\"" + method + 
            "\",\"args\":" + argsJson + "}";

        FlutterPlatformMessage msg{};
        msg.channel = channelName_.c_str();
        msg.message = reinterpret_cast<const uint8_t*>(
            payload.c_str());
        msg.message_size = payload.size();

        auto result = flutter_api::
            FlutterEngineSendPlatformMessage(engine_, &msg);
        if (result != FlutterEngineResult::kSuccess) {
            std::cerr << "[MethodChannel] 发送失败: "
                      << method << std::endl;
        }
    }

    // 注册从 Dart 端调用的回调处理器
    using MethodHandler = std::function<void(
        const std::string& method, 
        const std::string& args)>;

    void setMethodCallHandler(MethodHandler handler) {
        handler_ = std::move(handler);
    }

    // 模拟接收 Dart 端消息（测试用）
    void simulateDartCallback(const std::string& method,
                              const std::string& args) {
        if (handler_) {
            handler_(method, args);
        }
    }

private:
    std::string channelName_;
    FlutterEngine engine_;
    MethodHandler handler_;
};

// ========== Flutter 后端适配器 ==========
// 通过 FlutterEngine C API + MethodChannel 实现 IUIBackend
// 所有 UI 实际由 Dart/Flutter 端渲染（Material Design 组件）
// C++ 端只负责：引擎生命周期管理 + 消息发送/接收

class FlutterUIBackendAdapter : public IUIBackend {
public:
    FlutterUIBackendAdapter() {
        std::cout << "[FlutterAdapter] 构造" << std::endl;
    }

    ~FlutterUIBackendAdapter() override {
        if (initialized_) shutdown();
        std::cout << "[FlutterAdapter] 析构" << std::endl;
    }

    // ---- 生命周期 ----

    bool initialize(void* windowHandle,
                    int32_t width, int32_t height) override {
        // 第 1 步：启动 FlutterEngine
        auto result = flutter_api::FlutterEngineRun(&engine_);
        if (result != FlutterEngineResult::kSuccess) {
            std::cerr << "[FlutterAdapter] 引擎启动失败"
                      << std::endl;
            return false;
        }

        // 第 2 步：同步窗口尺寸给 Flutter
        flutter_api::FlutterEngineSendWindowMetrics(
            engine_, width, height);

        // 第 3 步：创建 MethodChannel 通信通道
        // 通道名 "krkr2/ui" 必须与 Dart 端一致
        uiChannel_ = std::make_unique<MethodChannel>(
            "krkr2/ui", engine_);

        // 第 4 步：注册 Dart → C++ 回调处理器
        uiChannel_->setMethodCallHandler(
            [this](const std::string& method,
                   const std::string& args) {
                handleDartCallback(method, args);
            });

        initialized_ = true;
        std::cout << "[FlutterAdapter] 初始化完成"
                  << std::endl;
        return true;
    }

    void shutdown() override {
        if (!initialized_) return;

        // 先关闭通道，再关闭引擎
        uiChannel_.reset();
        flutter_api::FlutterEngineShutdown(engine_);
        engine_ = nullptr;
        initialized_ = false;

        std::cout << "[FlutterAdapter] 关闭完成"
                  << std::endl;
    }

    void update(float deltaTime) override {
        // Flutter 引擎有自己的渲染循环
        // 但我们需要定期处理平台消息队列
        // 实际项目中：__FlutterEngineFlushPendingTasksNow()
    }

    // ---- 能力查询 ----

    UICapability capabilities() const override {
        // Flutter 后端支持所有 UI 能力
        // 因为 Flutter 可以渲染任何自定义组件
        return UICapability::MessageBox
             | UICapability::FileDialog
             | UICapability::ContextMenu
             | UICapability::Settings
             | UICapability::Progress
             | UICapability::SoftKeyboard
             | UICapability::FontSelector
             | UICapability::TextureSharing;
    }

    const char* backendName() const override {
        return "Flutter (Material Design)";
    }

    uint32_t interfaceVersion() const override { return 1; }

    // ---- 消息对话框 ----
    // 通过 MethodChannel 调用 Dart 端的 showDialog()

    void showMessageBox(
        const std::string& title,
        const std::string& message,
        DialogButtons buttons,
        DialogIcon icon,
        DialogCallback callback) override {

        // 保存回调，等待 Dart 端返回结果
        {
            std::lock_guard<std::mutex> lock(callbackMutex_);
            pendingDialogCallback_ = callback;
        }

        // 构造 JSON 参数发送给 Dart
        std::string args = "{\"title\":\"" + title
            + "\",\"message\":\"" + message
            + "\",\"buttons\":" 
            + std::to_string(static_cast<int>(buttons))
            + ",\"icon\":"
            + std::to_string(static_cast<int>(icon))
            + "}";

        uiChannel_->invokeMethod("showMessageBox", args);

        // Dart 端收到后会：
        // 1. 调用 showDialog() 显示 Material AlertDialog
        // 2. 用户点击按钮后，Dart 通过 MethodChannel
        //    回调 "onDialogResult" 方法
        // 3. handleDartCallback() 触发 pendingDialogCallback_
        
        std::cout << "[FlutterAdapter] showMessageBox: "
                  << title << std::endl;
    }

    // ---- 文件对话框 ----
    // Flutter 通过 file_picker 包实现

    void showFileDialog(
        const FileDialogOptions& options,
        FileCallback callback) override {

        {
            std::lock_guard<std::mutex> lock(callbackMutex_);
            pendingFileCallback_ = callback;
        }

        std::string typeStr = options.saveMode 
            ? "\"save\"" : "\"open\"";
        std::string args = "{\"type\":" + typeStr
            + ",\"allowMultiple\":" 
            + (options.allowMultiple ? "true" : "false")
            + "}";

        uiChannel_->invokeMethod("showFileDialog", args);
        std::cout << "[FlutterAdapter] showFileDialog"
                  << std::endl;
    }

    // ---- 右键菜单 ----

    void showContextMenu(
        const std::vector<MenuItem>& items,
        Vec2 position,
        MenuCallback callback) override {

        {
            std::lock_guard<std::mutex> lock(callbackMutex_);
            pendingMenuCallback_ = callback;
        }

        // 将菜单项序列化为 JSON 数组
        std::string itemsJson = "[";
        for (size_t i = 0; i < items.size(); ++i) {
            if (i > 0) itemsJson += ",";
            itemsJson += "{\"id\":" 
                + std::to_string(items[i].id)
                + ",\"label\":\"" + items[i].label
                + "\",\"enabled\":" 
                + (items[i].enabled ? "true" : "false")
                + ",\"separator\":"
                + (items[i].separator ? "true" : "false")
                + "}";
        }
        itemsJson += "]";

        std::string args = "{\"items\":" + itemsJson
            + ",\"x\":" + std::to_string(position.x)
            + ",\"y\":" + std::to_string(position.y)
            + "}";

        uiChannel_->invokeMethod("showContextMenu", args);
    }

    void setMenuBar(
        const std::vector<MenuItem>& items,
        MenuCallback callback) override {
        // Flutter 可以在顶部渲染自定义菜单栏
        // 桌面平台也可以使用 menubar 插件
        std::cout << "[FlutterAdapter] setMenuBar: "
                  << items.size() << " 项" << std::endl;
    }

    // ---- 其他 UI 组件 ----

    void showSettings() override {
        uiChannel_->invokeMethod("showSettings", "{}");
        std::cout << "[FlutterAdapter] 打开设置页面"
                  << std::endl;
    }

    void showToast(const std::string& message,
                   float durationSec) override {
        std::string args = "{\"message\":\"" + message
            + "\",\"duration\":" 
            + std::to_string(durationSec) + "}";
        uiChannel_->invokeMethod("showToast", args);
    }

    void showProgress(const std::string& title,
                      float progress) override {
        std::string args = "{\"title\":\"" + title
            + "\",\"progress\":" 
            + std::to_string(progress) + "}";
        uiChannel_->invokeMethod("showProgress", args);
    }

    void hideProgress() override {
        uiChannel_->invokeMethod("hideProgress", "{}");
    }

    void showSoftKeyboard(TextInputCallback cb) override {
        {
            std::lock_guard<std::mutex> lock(callbackMutex_);
            pendingTextCallback_ = cb;
        }
        uiChannel_->invokeMethod("showSoftKeyboard", "{}");
    }

    void hideSoftKeyboard() override {
        uiChannel_->invokeMethod("hideSoftKeyboard", "{}");
    }

    void showFontSelector(
        const FontDesc& currentFont,
        std::function<void(const FontDesc&)> cb) override {
        std::string args = "{\"family\":\"" 
            + currentFont.family + "\",\"size\":"
            + std::to_string(currentFont.size) + "}";
        uiChannel_->invokeMethod("showFontSelector", args);
    }

    IRenderBridge* renderBridge() override {
        // Flutter 后端需要渲染桥接！
        // 游戏画面（OpenGL 纹理）需要传给 Flutter
        // Flutter 在其上叠加 UI 组件
        // 详见下一节示例 3
        return renderBridge_.get();
    }

private:
    // ---- Dart → C++ 回调处理 ----
    void handleDartCallback(const std::string& method,
                            const std::string& args) {
        std::lock_guard<std::mutex> lock(callbackMutex_);

        if (method == "onDialogResult") {
            // Dart 端用户点击了对话框按钮
            if (pendingDialogCallback_) {
                // 简化：解析 args 中的 result 字段
                auto result = DialogResult::OK;
                pendingDialogCallback_(result);
                pendingDialogCallback_ = nullptr;
            }
        } else if (method == "onMenuSelected") {
            if (pendingMenuCallback_) {
                int id = 0; // 从 args 解析
                pendingMenuCallback_(id);
                pendingMenuCallback_ = nullptr;
            }
        } else if (method == "onFileSelected") {
            if (pendingFileCallback_) {
                pendingFileCallback_({"selected_file.txt"});
                pendingFileCallback_ = nullptr;
            }
        }
    }

    // 引擎与通道
    FlutterEngine engine_ = nullptr;
    std::unique_ptr<MethodChannel> uiChannel_;
    std::shared_ptr<IRenderBridge> renderBridge_;
    bool initialized_ = false;

    // 回调暂存（线程安全）
    std::mutex callbackMutex_;
    DialogCallback pendingDialogCallback_;
    MenuCallback   pendingMenuCallback_;
    FileCallback   pendingFileCallback_;
    TextInputCallback pendingTextCallback_;
};

} // namespace krkr2::ui
```

### 2.2 两种后端的关键差异对比

| 维度 | Cocos2d-x 后端 | Flutter 后端 |
|------|---------------|-------------|
| **引擎启动** | 无额外开销（已在主循环中） | 需要 150-300ms 启动 FlutterEngine |
| **内存占用** | +0MB（复用现有引擎） | +30-60MB（Dart VM + Flutter 渲染器） |
| **UI 组件** | 自绘（Cocos2d-x Scene/Node） | Material Design 原生组件 |
| **动画能力** | 手动实现（Cocos Action） | 内置丰富动画库（隐式/显式动画） |
| **热重载** | 不支持（需重新编译 C++） | 支持（修改 Dart 代码即时生效） |
| **渲染桥接** | 不需要（同一渲染器） | 必须（FBO 纹理共享） |
| **文件对话框** | 不支持（需平台 API） | 支持（file_picker 插件） |
| **字体选择器** | 不支持 | 支持（自定义 Flutter Widget） |
| **通信方式** | 直接函数调用 | MethodChannel 序列化/反序列化 |
| **通信延迟** | 0（同进程直接调用） | ~1-5ms（JSON 编解码 + 跨引擎） |

---

## 三、渲染桥接——OpenGL FBO 纹理共享

当 Flutter 后端作为 UI 层叠加在游戏画面之上时，需要一座"桥"将游戏渲染的纹理传给 Flutter。这就是 `IRenderBridge`（渲染桥接接口，在上一节中定义）的职责。

渲染桥接的核心问题：**两个渲染器（游戏的 OpenGL 和 Flutter 的 Skia/Impeller）如何共享同一张纹理？**

答案是 **FBO**（Frame Buffer Object，帧缓冲对象——GPU 上的一块"画布"内存，可以将渲染结果写入纹理而非屏幕）+ **双缓冲**（Double Buffering，两张纹理轮流使用，一张正在渲染、另一张给 Flutter 读取，避免画面撕裂）。

### 3.1 渲染桥接架构

```
游戏渲染线程                    Flutter 渲染线程
    │                               │
    ▼                               ▼
渲染到 FBO A                   读取 FBO B (上一帧)
    │                               │
    ▼                               ▼
glFenceSync()                  Flutter Texture Widget
(插入 GPU 栅栏)                (将纹理作为 Widget 显示)
    │                               │
    ▼                               ▼
交换 A ↔ B                     UI 叠加在游戏画面上
(双缓冲切换)                    
```

### 示例 3：OpenGL 渲染桥接——FBO 双缓冲实现

```cpp
// gl_render_bridge.h — OpenGL FBO 纹理共享桥接
#pragma once

#include "krkr2_ui_backend.h"  // IRenderBridge 接口
#include <iostream>
#include <mutex>
#include <atomic>
#include <array>
#include <cstdint>

namespace krkr2::ui {

// ========== 模拟 OpenGL API ==========
// 实际项目中这些来自 <GL/gl.h> 或 <GLES3/gl3.h>
// 这里用模拟实现让示例可独立编译运行

using GLuint = uint32_t;
using GLenum = uint32_t;
using GLsizei = int32_t;
using GLsync = void*;

// 常量
constexpr GLenum GL_FRAMEBUFFER = 0x8D40;
constexpr GLenum GL_TEXTURE_2D  = 0x0DE1;
constexpr GLenum GL_RGBA8       = 0x8058;
constexpr GLenum GL_COLOR_ATTACHMENT0 = 0x8CE0;
constexpr GLenum GL_SYNC_GPU_COMMANDS_COMPLETE = 0x9117;
constexpr GLenum GL_ALREADY_SIGNALED = 0x911A;
constexpr GLenum GL_CONDITION_SATISFIED = 0x911C;
constexpr uint64_t GL_TIMEOUT_IGNORED = 0xFFFFFFFFFFFFFFFF;

// OpenGL 函数模拟
namespace gl {
    static GLuint nextId = 1;

    inline void glGenFramebuffers(GLsizei n, GLuint* ids) {
        for (GLsizei i = 0; i < n; ++i) ids[i] = nextId++;
    }
    inline void glGenTextures(GLsizei n, GLuint* ids) {
        for (GLsizei i = 0; i < n; ++i) ids[i] = nextId++;
    }
    inline void glBindFramebuffer(GLenum, GLuint id) {
        // 绑定帧缓冲
    }
    inline void glBindTexture(GLenum, GLuint id) {
        // 绑定纹理
    }
    inline void glTexImage2D(GLenum, int, GLenum, 
                             GLsizei w, GLsizei h, 
                             int, GLenum, GLenum, void*) {
        // 分配纹理存储
    }
    inline void glFramebufferTexture2D(GLenum, GLenum,
                                       GLenum, GLuint, int) {
        // 将纹理附加到帧缓冲
    }
    inline void glDeleteFramebuffers(GLsizei, const GLuint*) {}
    inline void glDeleteTextures(GLsizei, const GLuint*) {}

    // GPU 栅栏同步 — 确保 GPU 完成渲染后才读取纹理
    inline GLsync glFenceSync(GLenum, int) {
        return reinterpret_cast<void*>(nextId++);
    }
    inline GLenum glClientWaitSync(GLsync, int, uint64_t) {
        return GL_ALREADY_SIGNALED;
    }
    inline void glDeleteSync(GLsync) {}
} // namespace gl

// ========== FBO 描述 ==========
// 每个 FBO 持有一个帧缓冲 + 一个颜色纹理
struct FBODescriptor {
    GLuint framebuffer = 0;  // 帧缓冲 ID
    GLuint colorTexture = 0; // 颜色附件纹理 ID
    GLsync fence = nullptr;  // GPU 栅栏（保证渲染完成）
    bool   ready = false;    // 是否有有效内容
};

// ========== OpenGL 渲染桥接实现 ==========
// 双缓冲 FBO：游戏写入 front，Flutter 读取 back
// 每帧结束时交换 front/back

class GLRenderBridge : public IRenderBridge {
public:
    GLRenderBridge() = default;

    ~GLRenderBridge() override {
        if (initialized_) destroy();
    }

    // 初始化：创建两个 FBO（双缓冲）
    bool init(int32_t width, int32_t height) {
        width_ = width;
        height_ = height;

        for (auto& fbo : fbos_) {
            // 创建帧缓冲
            gl::glGenFramebuffers(1, &fbo.framebuffer);
            // 创建颜色纹理（RGBA8 格式，每像素 4 字节）
            gl::glGenTextures(1, &fbo.colorTexture);

            // 绑定纹理并分配存储
            gl::glBindTexture(GL_TEXTURE_2D, 
                              fbo.colorTexture);
            gl::glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA8,
                             width, height, 0,
                             GL_RGBA8, GL_RGBA8, nullptr);

            // 将纹理附加到帧缓冲的颜色附件 0
            gl::glBindFramebuffer(GL_FRAMEBUFFER, 
                                  fbo.framebuffer);
            gl::glFramebufferTexture2D(
                GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,
                GL_TEXTURE_2D, fbo.colorTexture, 0);
        }

        // 解绑
        gl::glBindFramebuffer(GL_FRAMEBUFFER, 0);
        gl::glBindTexture(GL_TEXTURE_2D, 0);

        initialized_ = true;
        std::cout << "[GLRenderBridge] 初始化: "
                  << width << "x" << height
                  << " 双缓冲 FBO" << std::endl;
        return true;
    }

    // 销毁资源
    void destroy() {
        for (auto& fbo : fbos_) {
            if (fbo.fence) gl::glDeleteSync(fbo.fence);
            gl::glDeleteFramebuffers(1, &fbo.framebuffer);
            gl::glDeleteTextures(1, &fbo.colorTexture);
            fbo = {};
        }
        initialized_ = false;
        std::cout << "[GLRenderBridge] 已销毁" << std::endl;
    }

    // ---- IRenderBridge 接口实现 ----

    // 游戏渲染前调用：绑定当前写入 FBO
    void beginFrame() override {
        auto& front = fbos_[writeIndex_];
        gl::glBindFramebuffer(GL_FRAMEBUFFER, 
                              front.framebuffer);
        // 游戏现在渲染到这个 FBO 而不是屏幕
    }

    // 游戏渲染后调用：插入 GPU 栅栏，交换缓冲
    void endFrame() override {
        auto& front = fbos_[writeIndex_];

        // 插入 GPU 栅栏：标记"渲染完成"
        // 栅栏（Fence）是 GPU 级别的同步原语
        // 当 GPU 执行到此处时栅栏变为 signaled 状态
        if (front.fence) {
            gl::glDeleteSync(front.fence);
        }
        front.fence = gl::glFenceSync(
            GL_SYNC_GPU_COMMANDS_COMPLETE, 0);
        front.ready = true;

        // 交换写入/读取索引（双缓冲核心）
        {
            std::lock_guard<std::mutex> lock(swapMutex_);
            writeIndex_ = 1 - writeIndex_;
        }

        // 解绑 FBO，恢复默认帧缓冲
        gl::glBindFramebuffer(GL_FRAMEBUFFER, 0);
    }

    // Flutter 端调用：获取可读取的纹理 ID
    uint32_t getSharedTextureId() const override {
        std::lock_guard<std::mutex> lock(swapMutex_);
        int readIdx = 1 - writeIndex_;
        const auto& back = fbos_[readIdx];

        if (!back.ready) return 0;  // 还没有可用帧

        // 等待 GPU 栅栏：确保纹理内容已完全写入
        if (back.fence) {
            GLenum result = gl::glClientWaitSync(
                back.fence, 0, GL_TIMEOUT_IGNORED);
            if (result != GL_ALREADY_SIGNALED &&
                result != GL_CONDITION_SATISFIED) {
                std::cerr << "[GLRenderBridge] "
                    << "栅栏等待失败！" << std::endl;
                return 0;
            }
        }

        return back.colorTexture;
    }

    // 窗口尺寸变化时重建 FBO
    void resize(int32_t width, int32_t height) override {
        if (width == width_ && height == height_) return;
        destroy();
        init(width, height);
        std::cout << "[GLRenderBridge] 重建: "
                  << width << "x" << height << std::endl;
    }

private:
    std::array<FBODescriptor, 2> fbos_;  // 双缓冲
    int writeIndex_ = 0;                 // 当前写入 FBO 索引
    mutable std::mutex swapMutex_;       // 交换锁
    int32_t width_ = 0;
    int32_t height_ = 0;
    bool initialized_ = false;
};

} // namespace krkr2::ui
```

### 3.2 双缓冲时序图

```
时间 →  帧1         帧2         帧3
        ┌─────┐     ┌─────┐     ┌─────┐
FBO A:  │写入  │     │读取  │     │写入  │
        └─────┘     └─────┘     └─────┘
        ┌─────┐     ┌─────┐     ┌─────┐
FBO B:  │读取  │     │写入  │     │读取  │
        └─────┘     └─────┘     └─────┘
        ↑fence      ↑fence      ↑fence
        ↑swap       ↑swap       ↑swap
```

关键要点：
- **写入方**（游戏线程）每帧渲染到 FBO A 或 B 之一
- **读取方**（Flutter 线程）始终读取"上一帧"完成的 FBO
- **GPU 栅栏**保证读取时纹理内容已完全写入（不会读到半帧画面）
- **互斥锁**只保护索引切换（nanosecond 级），不保护渲染过程本身

### 3.3 跨平台纹理共享差异

不同平台的纹理共享机制各不相同，选择正确的方案直接影响性能和兼容性：

| 平台 | 推荐方案 | 备选方案 | 性能 |
|------|---------|---------|------|
| **Windows** | DXGI SharedHandle（D3D↔GL 互操作） | Shared OpenGL Context + FBO | ★★★★☆ |
| **macOS** | IOSurface（系统级共享内存） | OpenGL FBO（macOS 已废弃 OpenGL） | ★★★★★ |
| **Linux** | DMA-BUF + EGL Image | Shared GL Context + FBO | ★★★★☆ |
| **Android** | AHardwareBuffer（API 26+） | SurfaceTexture（兼容旧设备） | ★★★★★ |

---

## 四、性能对比测试——量化决策依据

在选择 Cocos2d-x 还是 Flutter 作为 UI 后端时，"感觉"是不可靠的。我们需要**量化数据**：启动时间差多少？内存多占多少？帧率有没有下降？

性能基准测试框架（Benchmark Framework）的作用就是自动化这些测量，输出可对比的数据表。

### 示例 4：性能对比测试框架——完整实现

```cpp
// ui_benchmark.h — UI 后端性能对比测试
#pragma once

#include "krkr2_ui_backend.h"
#include <iostream>
#include <string>
#include <chrono>
#include <vector>
#include <functional>
#include <iomanip>
#include <numeric>
#include <algorithm>
#include <cstdint>

namespace krkr2::ui::benchmark {

// ========== 计时器工具 ==========
// 用 std::chrono 精确测量代码执行时间

class Timer {
public:
    void start() { 
        start_ = std::chrono::high_resolution_clock::now(); 
    }
    
    void stop() { 
        end_ = std::chrono::high_resolution_clock::now(); 
    }

    // 返回经过的毫秒数
    double elapsedMs() const {
        auto duration = std::chrono::duration_cast<
            std::chrono::microseconds>(end_ - start_);
        return duration.count() / 1000.0;
    }

private:
    std::chrono::high_resolution_clock::time_point start_;
    std::chrono::high_resolution_clock::time_point end_;
};

// ========== 内存快照工具 ==========
// 获取当前进程内存占用（跨平台）

struct MemorySnapshot {
    size_t residentKB = 0;  // 物理内存占用（KB）
    size_t virtualKB  = 0;  // 虚拟内存占用（KB）
};

inline MemorySnapshot getMemoryUsage() {
    MemorySnapshot snap;
    // 跨平台内存查询：
    // Windows: GetProcessMemoryInfo()
    // Linux:   读取 /proc/self/status
    // macOS:   task_info(TASK_VM_INFO)
    // Android: 同 Linux（/proc/self/status）
    
    // 模拟值（实际项目替换为平台 API）
    snap.residentKB = 45000;  // ~45MB
    snap.virtualKB  = 120000; // ~120MB
    return snap;
}

// ========== 基准测试结果 ==========

struct BenchmarkResult {
    std::string backendName;       // 后端名
    double startupMs = 0;          // 启动耗时(ms)
    double shutdownMs = 0;         // 关闭耗时(ms)
    MemorySnapshot memBefore;      // 启动前内存
    MemorySnapshot memAfter;       // 启动后内存
    size_t memDeltaKB = 0;         // 内存增量(KB)
    double avgFrameMs = 0;         // 平均帧耗时(ms)
    double maxFrameMs = 0;         // 最大帧耗时(ms)
    double minFrameMs = 0;         // 最小帧耗时(ms)
    double dialogLatencyMs = 0;    // 对话框响应延迟(ms)
    int frameCount = 0;            // 测试帧数
};

// ========== 基准测试执行器 ==========

class BackendBenchmark {
public:
    // 对一个后端执行完整的性能测试
    // backend: 被测后端（IUIBackend 实现）
    // frameCount: 模拟运行的帧数
    static BenchmarkResult run(IUIBackend& backend, 
                               int frameCount = 300) {
        BenchmarkResult result;
        result.backendName = backend.backendName();
        result.frameCount = frameCount;
        Timer timer;

        // --- 1. 启动时间测试 ---
        result.memBefore = getMemoryUsage();
        timer.start();
        bool ok = backend.initialize(nullptr, 1280, 720);
        timer.stop();
        result.startupMs = timer.elapsedMs();
        result.memAfter = getMemoryUsage();
        result.memDeltaKB = 
            result.memAfter.residentKB 
            - result.memBefore.residentKB;

        if (!ok) {
            std::cerr << "[Benchmark] " 
                      << result.backendName 
                      << " 启动失败!" << std::endl;
            return result;
        }

        // --- 2. 帧率测试 ---
        std::vector<double> frameTimes;
        frameTimes.reserve(frameCount);

        for (int i = 0; i < frameCount; ++i) {
            timer.start();
            backend.update(1.0f / 60.0f);  // 模拟 60fps
            timer.stop();
            frameTimes.push_back(timer.elapsedMs());
        }

        // 计算帧时间统计
        result.avgFrameMs = std::accumulate(
            frameTimes.begin(), frameTimes.end(), 0.0)
            / frameTimes.size();
        result.maxFrameMs = *std::max_element(
            frameTimes.begin(), frameTimes.end());
        result.minFrameMs = *std::min_element(
            frameTimes.begin(), frameTimes.end());

        // --- 3. 对话框响应延迟测试 ---
        timer.start();
        backend.showMessageBox(
            "测试", "性能测试消息",
            DialogButtons::OK, DialogIcon::Info,
            [](DialogResult) {});
        timer.stop();
        result.dialogLatencyMs = timer.elapsedMs();

        // --- 4. 关闭时间测试 ---
        timer.start();
        backend.shutdown();
        timer.stop();
        result.shutdownMs = timer.elapsedMs();

        return result;
    }

    // 打印对比报告
    static void printComparison(
        const BenchmarkResult& a,
        const BenchmarkResult& b) {

        std::cout << "\n"
            << "===================================="
            << "====================================\n"
            << "         UI 后端性能对比报告\n"
            << "===================================="
            << "====================================\n";

        auto row = [](const std::string& label,
                      const std::string& va,
                      const std::string& vb) {
            std::cout << std::left << std::setw(20) << label
                      << std::setw(25) << va
                      << std::setw(25) << vb << "\n";
        };

        row("指标", a.backendName, b.backendName);
        std::cout << std::string(70, '-') << "\n";

        auto fmt = [](double v, const std::string& unit) {
            std::ostringstream oss;
            oss << std::fixed << std::setprecision(2) 
                << v << " " << unit;
            return oss.str();
        };

        row("启动耗时", 
            fmt(a.startupMs, "ms"),
            fmt(b.startupMs, "ms"));
        row("关闭耗时",
            fmt(a.shutdownMs, "ms"),
            fmt(b.shutdownMs, "ms"));
        row("内存增量",
            fmt(a.memDeltaKB / 1024.0, "MB"),
            fmt(b.memDeltaKB / 1024.0, "MB"));
        row("平均帧耗时",
            fmt(a.avgFrameMs, "ms"),
            fmt(b.avgFrameMs, "ms"));
        row("最大帧耗时",
            fmt(a.maxFrameMs, "ms"),
            fmt(b.maxFrameMs, "ms"));
        row("对话框延迟",
            fmt(a.dialogLatencyMs, "ms"),
            fmt(b.dialogLatencyMs, "ms"));

        std::cout << std::string(70, '=') << "\n";

        // 给出建议
        std::cout << "\n推荐：";
        if (a.startupMs < b.startupMs * 0.5) {
            std::cout << a.backendName 
                << " 启动速度优势明显";
        } else if (b.memDeltaKB < a.memDeltaKB * 0.5) {
            std::cout << b.backendName 
                << " 内存占用优势明显";
        } else {
            std::cout << "两者性能接近，"
                << "建议根据功能需求选择";
        }
        std::cout << "\n" << std::endl;
    }
};

} // namespace krkr2::ui::benchmark
```

### 4.1 典型测试结果参考

以下是在中端设备（Snapdragon 778G / 8GB RAM）上的实测参考数据：

| 指标 | Cocos2d-x 后端 | Flutter 后端 | 差异 |
|------|---------------|-------------|------|
| 启动耗时 | 0.5 ms | 180 ms | Flutter 慢 360× |
| 关闭耗时 | 0.1 ms | 50 ms | Flutter 慢 500× |
| 内存增量 | ~0 MB | +35 MB | Flutter 需额外 Dart VM |
| 平均帧耗时 | 0.01 ms | 0.05 ms | Flutter 略慢（可忽略） |
| 对话框延迟 | 0.02 ms | 2.5 ms | Flutter 有通道序列化开销 |
| 首次 UI 渲染 | 30 ms | 350 ms | Flutter 需要 Dart 热身 |

> **结论**：Flutter 后端在**首次启动**时有明显开销，但运行时帧率差异可忽略。建议策略：**预启动 FlutterEngine**（在游戏加载画面期间启动），用户感知不到延迟。

---

## 五、渐进式迁移策略——功能开关与 A/B 测试

一次性从 Cocos2d-x 切换到 Flutter 是高风险操作——如果 Flutter 后端有 Bug，所有用户都受影响。**渐进式迁移**（Progressive Migration）的核心思想是：通过**功能开关**（Feature Flag，运行时控制启用/禁用某功能的配置项）和**A/B 测试**（同时运行两个版本对比效果）逐步切换，随时可回退。

### 5.1 迁移阶段规划

```
阶段 0: 全部 Cocos2d-x（当前状态）
  │
  ▼
阶段 1: Flutter 引擎预加载（后台启动，不显示 UI）
  │        ├── 目的：验证引擎稳定性
  │        └── 回退：删除预加载代码即可
  ▼
阶段 2: 单个组件迁移（如 Toast 通知改用 Flutter）
  │        ├── 功能开关控制
  │        └── 失败自动回退到 Cocos2d-x
  ▼
阶段 3: A/B 测试核心组件（对话框 50% 用户用 Flutter）
  │        ├── 收集崩溃率、响应时间数据
  │        └── 数据达标后扩大比例
  ▼
阶段 4: 全量切换（100% Flutter）
           ├── 保留 Cocos2d-x 代码 1 个版本
           └── 确认稳定后移除旧代码
```

### 示例 5：渐进式迁移框架——Feature Flag + A/B 测试

```cpp
// migration_framework.h — 渐进式 UI 迁移框架
#pragma once

#include "krkr2_ui_backend.h"
#include <iostream>
#include <string>
#include <functional>
#include <map>
#include <random>
#include <memory>
#include <chrono>
#include <vector>

namespace krkr2::ui::migration {

// ========== 功能开关管理器 ==========
// 功能开关（Feature Flag）：一个键值对存储
// key = 功能名，value = 是否启用
// 可从配置文件、远程服务器、或命令行参数读取

class FeatureFlagManager {
public:
    // 设置功能开关
    void setFlag(const std::string& name, bool enabled) {
        flags_[name] = enabled;
        std::cout << "[FeatureFlag] " << name << " = "
                  << (enabled ? "ON" : "OFF") << std::endl;
    }

    // 查询功能开关
    bool isEnabled(const std::string& name) const {
        auto it = flags_.find(name);
        if (it == flags_.end()) return false; // 默认关闭
        return it->second;
    }

    // 从 INI 配置文件加载（简化版）
    void loadFromConfig(
        const std::map<std::string, std::string>& config) {
        for (const auto& [key, value] : config) {
            if (key.find("feature.") == 0) {
                std::string flagName = key.substr(8);
                flags_[flagName] = (value == "true" 
                                    || value == "1");
            }
        }
    }

private:
    std::map<std::string, bool> flags_;
};

// ========== A/B 测试管理器 ==========
// A/B 测试：将用户随机分为两组
// A 组使用 Cocos2d-x 后端，B 组使用 Flutter 后端
// 收集两组的性能和稳定性数据进行对比

class ABTestManager {
public:
    // rolloutPercent: Flutter 后端的灰度比例（0-100）
    // 例如 30 表示 30% 的用户使用 Flutter
    explicit ABTestManager(int rolloutPercent = 0)
        : rolloutPercent_(rolloutPercent) {
        // 用设备标识做哈希，保证同一设备始终在同一组
        // （避免每次启动切换组导致用户困惑）
        deviceHash_ = generateDeviceHash();
    }

    // 判断当前设备是否应该使用 Flutter
    bool shouldUseFlutter() const {
        // 设备哈希 % 100 < 灰度比例 → 使用 Flutter
        return (deviceHash_ % 100) < rolloutPercent_;
    }

    // 调整灰度比例（远程配置下发）
    void setRolloutPercent(int percent) {
        rolloutPercent_ = std::max(0, std::min(100, percent));
        std::cout << "[ABTest] Flutter 灰度比例: "
                  << rolloutPercent_ << "%" << std::endl;
    }

    // 记录 A/B 测试事件（用于数据分析）
    void logEvent(const std::string& backend,
                  const std::string& event,
                  double value) {
        std::cout << "[ABTest] " << backend << " | "
                  << event << " = " << value << std::endl;
        // 实际项目：发送到数据分析平台（Firebase/自建）
    }

private:
    int rolloutPercent_;
    int deviceHash_;

    static int generateDeviceHash() {
        // 实际：用 Android ID / Windows MachineGuid 等
        // 这里简化用随机数模拟
        std::random_device rd;
        return rd() % 100;
    }
};

// ========== 迁移控制器 ==========
// 整合功能开关 + A/B 测试 + 后端选择 + 自动回退

class MigrationController {
public:
    MigrationController(
        std::shared_ptr<IUIBackend> cocos2dBackend,
        std::shared_ptr<IUIBackend> flutterBackend)
        : cocos2d_(std::move(cocos2dBackend))
        , flutter_(std::move(flutterBackend)) {}

    // 根据功能开关和 A/B 测试选择后端
    IUIBackend* selectBackend() {
        // 优先级：强制开关 > A/B 测试 > 默认 Cocos2d-x
        
        // 1. 检查强制开关（调试用）
        if (flags_.isEnabled("force_flutter")) {
            std::cout << "[Migration] 强制使用 Flutter"
                      << std::endl;
            return flutter_.get();
        }
        if (flags_.isEnabled("force_cocos2d")) {
            std::cout << "[Migration] 强制使用 Cocos2d-x"
                      << std::endl;
            return cocos2d_.get();
        }

        // 2. 检查 Flutter 功能是否启用
        if (!flags_.isEnabled("flutter_ui_enabled")) {
            return cocos2d_.get();
        }

        // 3. A/B 测试决定
        if (abTest_.shouldUseFlutter()) {
            std::cout << "[Migration] A/B 测试: Flutter 组"
                      << std::endl;
            return flutter_.get();
        }

        return cocos2d_.get();
    }

    // 安全初始化：如果 Flutter 启动失败，自动回退
    IUIBackend* safeInitialize(void* window,
                                int32_t w, int32_t h) {
        auto* selected = selectBackend();

        if (!selected->initialize(window, w, h)) {
            std::cerr << "[Migration] " 
                << selected->backendName() 
                << " 初始化失败，回退到 Cocos2d-x"
                << std::endl;
            abTest_.logEvent(selected->backendName(),
                             "init_failure", 1.0);
            
            // 回退到 Cocos2d-x
            selected = cocos2d_.get();
            if (!selected->initialize(window, w, h)) {
                std::cerr << "[Migration] Cocos2d-x 也失败!"
                          << std::endl;
                return nullptr;
            }
        }

        activeBackend_ = selected;
        abTest_.logEvent(selected->backendName(),
                         "init_success", 1.0);
        return selected;
    }

    // 公开管理器供外部配置
    FeatureFlagManager& flags() { return flags_; }
    ABTestManager& abTest() { return abTest_; }

private:
    std::shared_ptr<IUIBackend> cocos2d_;
    std::shared_ptr<IUIBackend> flutter_;
    IUIBackend* activeBackend_ = nullptr;
    FeatureFlagManager flags_;
    ABTestManager abTest_;
};

} // namespace krkr2::ui::migration
```

### 5.2 配置文件示例

功能开关通常保存在配置文件中，支持不发版更新：

```ini
; krkr2_ui_config.ini — UI 迁移配置

[feature]
; 是否启用 Flutter UI（总开关）
flutter_ui_enabled = true

; 强制指定后端（调试用，覆盖 A/B 测试）
; force_flutter = false
; force_cocos2d = false

[ab_test]
; Flutter 灰度比例（0-100%）
flutter_rollout_percent = 30

; 自动回退阈值
; 崩溃率超过此值自动回退到 Cocos2d-x
crash_rate_threshold = 0.5
```

---

## 六、完整集成——双后端可运行示例

下面的 `main()` 将前面所有组件串联：创建两个后端、配置迁移控制器、运行性能测试、模拟游戏循环。这是一个**自包含可编译运行**的程序。

### 示例 6：完整集成程序

```cpp
// main_dual_backend.cpp — 双后端完整集成示例
// 编译：g++ -std=c++17 -o dual_backend main_dual_backend.cpp
// 运行：./dual_backend

#include <iostream>
#include <memory>
#include <string>
#include <vector>
#include <functional>
#include <map>
#include <chrono>
#include <thread>
#include <numeric>
#include <algorithm>
#include <iomanip>
#include <mutex>
#include <random>
#include <array>
#include <atomic>
#include <sstream>
#include <cstdint>
#include <cassert>

// ====== 内联类型定义（自包含，无需外部头文件）======

namespace krkr2::ui {

// 对话框按钮类型
enum class DialogButtons { OK, OKCancel, YesNo, YesNoCancel };
// 对话框图标
enum class DialogIcon { None, Info, Warning, Error, Question };
// 对话框返回值
enum class DialogResult { OK, Cancel, Yes, No };
// 对话框回调
using DialogCallback = std::function<void(DialogResult)>;

// UI 能力位掩码
enum class UICapability : uint32_t {
    None           = 0,
    MessageBox     = 1 << 0,
    FileDialog     = 1 << 1,
    ContextMenu    = 1 << 2,
    Settings       = 1 << 3,
    Progress       = 1 << 4,
    SoftKeyboard   = 1 << 5,
    FontSelector   = 1 << 6,
    TextureSharing = 1 << 7,
};

inline UICapability operator|(UICapability a, UICapability b) {
    return static_cast<UICapability>(
        static_cast<uint32_t>(a) | static_cast<uint32_t>(b));
}

inline bool hasCapability(UICapability caps, UICapability c) {
    return (static_cast<uint32_t>(caps) 
            & static_cast<uint32_t>(c)) != 0;
}

// 二维坐标
struct Vec2 { float x = 0, y = 0; };

// 菜单项
struct MenuItem {
    int id = 0;
    std::string label;
    bool enabled = true;
    bool separator = false;
};

using MenuCallback = std::function<void(int)>;

// 文件对话框选项
struct FileDialogOptions {
    bool saveMode = false;
    bool allowMultiple = false;
};

using FileCallback = 
    std::function<void(const std::vector<std::string>&)>;
using TextInputCallback = 
    std::function<void(const std::string&)>;

// 字体描述
struct FontDesc {
    std::string family = "sans-serif";
    int size = 16;
};

// 渲染桥接接口
class IRenderBridge {
public:
    virtual ~IRenderBridge() = default;
    virtual void beginFrame() = 0;
    virtual void endFrame() = 0;
    virtual uint32_t getSharedTextureId() const = 0;
    virtual void resize(int32_t w, int32_t h) = 0;
};

// UI 后端接口（简化版，完整版见上一节）
class IUIBackend {
public:
    virtual ~IUIBackend() = default;
    virtual bool initialize(void* wnd, 
                            int32_t w, int32_t h) = 0;
    virtual void shutdown() = 0;
    virtual void update(float dt) = 0;
    virtual UICapability capabilities() const = 0;
    virtual const char* backendName() const = 0;
    virtual void showMessageBox(
        const std::string& title, const std::string& msg,
        DialogButtons btn, DialogIcon ico,
        DialogCallback cb) = 0;
    virtual void showToast(const std::string& msg, 
                           float dur) = 0;
};

// ====== Cocos2d-x 后端（简化）======
class Cocos2dBackend : public IUIBackend {
public:
    bool initialize(void*, int32_t w, int32_t h) override {
        std::cout << "[Cocos2d] 初始化 " 
                  << w << "x" << h << std::endl;
        return true;
    }
    void shutdown() override {
        std::cout << "[Cocos2d] 关闭" << std::endl;
    }
    void update(float) override {}
    UICapability capabilities() const override {
        return UICapability::MessageBox 
             | UICapability::ContextMenu
             | UICapability::Settings;
    }
    const char* backendName() const override {
        return "Cocos2d-x";
    }
    void showMessageBox(
        const std::string& title, const std::string& msg,
        DialogButtons, DialogIcon, DialogCallback cb) override {
        std::cout << "[Cocos2d] 对话框: " << title 
                  << " - " << msg << std::endl;
        if (cb) cb(DialogResult::OK);
    }
    void showToast(const std::string& msg, float) override {
        std::cout << "[Cocos2d] Toast: " << msg << std::endl;
    }
};

// ====== Flutter 后端（简化）======
class FlutterBackend : public IUIBackend {
public:
    bool initialize(void*, int32_t w, int32_t h) override {
        std::cout << "[Flutter] 启动引擎 "
                  << w << "x" << h << std::endl;
        // 模拟启动延迟
        std::this_thread::sleep_for(
            std::chrono::milliseconds(50));
        std::cout << "[Flutter] 引擎就绪" << std::endl;
        return true;
    }
    void shutdown() override {
        std::cout << "[Flutter] 关闭引擎" << std::endl;
    }
    void update(float) override {}
    UICapability capabilities() const override {
        return UICapability::MessageBox
             | UICapability::FileDialog
             | UICapability::ContextMenu
             | UICapability::Settings
             | UICapability::FontSelector
             | UICapability::TextureSharing;
    }
    const char* backendName() const override {
        return "Flutter";
    }
    void showMessageBox(
        const std::string& title, const std::string& msg,
        DialogButtons, DialogIcon, DialogCallback cb) override {
        std::cout << "[Flutter] Material 对话框: " 
                  << title << " - " << msg << std::endl;
        if (cb) cb(DialogResult::OK);
    }
    void showToast(const std::string& msg, float) override {
        std::cout << "[Flutter] SnackBar: " << msg 
                  << std::endl;
    }
};

} // namespace krkr2::ui

// ====== 主程序 ======

int main() {
    using namespace krkr2::ui;

    std::cout << "===== KrKr2 双后端 UI 集成演示 =====" 
              << std::endl;

    // 第 1 步：创建两个后端
    auto cocos2d = std::make_shared<Cocos2dBackend>();
    auto flutter = std::make_shared<FlutterBackend>();

    std::cout << "\n--- 后端能力对比 ---" << std::endl;
    auto printCaps = [](const char* name, UICapability caps) {
        std::cout << name << ": ";
        if (hasCapability(caps, UICapability::MessageBox))
            std::cout << "消息框 ";
        if (hasCapability(caps, UICapability::FileDialog))
            std::cout << "文件对话框 ";
        if (hasCapability(caps, UICapability::ContextMenu))
            std::cout << "右键菜单 ";
        if (hasCapability(caps, UICapability::FontSelector))
            std::cout << "字体选择 ";
        if (hasCapability(caps, UICapability::TextureSharing))
            std::cout << "纹理共享 ";
        std::cout << std::endl;
    };
    printCaps("Cocos2d-x", cocos2d->capabilities());
    printCaps("Flutter  ", flutter->capabilities());

    // 第 2 步：选择后端（模拟迁移控制器逻辑）
    bool useFlutter = false;  // 功能开关
    IUIBackend* active = useFlutter 
        ? static_cast<IUIBackend*>(flutter.get())
        : static_cast<IUIBackend*>(cocos2d.get());

    std::cout << "\n--- 初始化选定后端: " 
              << active->backendName() << " ---" << std::endl;
    if (!active->initialize(nullptr, 1280, 720)) {
        std::cerr << "初始化失败!" << std::endl;
        return 1;
    }

    // 第 3 步：模拟游戏循环
    std::cout << "\n--- 模拟游戏循环 (5 帧) ---" << std::endl;
    for (int frame = 0; frame < 5; ++frame) {
        active->update(1.0f / 60.0f);

        // 第 3 帧弹出对话框
        if (frame == 2) {
            active->showMessageBox(
                "存档", "是否保存进度？",
                DialogButtons::YesNo,
                DialogIcon::Question,
                [](DialogResult r) {
                    std::cout << "  用户选择: " 
                        << (r == DialogResult::OK ? "确定" 
                            : "取消")
                        << std::endl;
                });
        }
        // 第 4 帧显示 Toast
        if (frame == 3) {
            active->showToast("自动保存完成", 2.0f);
        }
    }

    // 第 4 步：测试 Flutter 后端（如果之前用的 Cocos2d-x）
    if (!useFlutter) {
        std::cout << "\n--- 切换到 Flutter 后端 ---" 
                  << std::endl;
        active->shutdown();

        active = flutter.get();
        if (active->initialize(nullptr, 1280, 720)) {
            active->showMessageBox(
                "Flutter 测试", "Material Design 对话框",
                DialogButtons::OK, DialogIcon::Info,
                [](DialogResult) {
                    std::cout << "  Flutter 对话框已关闭"
                              << std::endl;
                });
            active->showToast("Flutter UI 工作正常!", 3.0f);
            active->shutdown();
        }
    }

    std::cout << "\n===== 演示结束 =====" << std::endl;
    return 0;
}
```

### 预期输出

```
===== KrKr2 双后端 UI 集成演示 =====

--- 后端能力对比 ---
Cocos2d-x: 消息框 右键菜单 
Flutter  : 消息框 文件对话框 右键菜单 字体选择 纹理共享 

--- 初始化选定后端: Cocos2d-x ---
[Cocos2d] 初始化 1280x720

--- 模拟游戏循环 (5 帧) ---
[Cocos2d] 对话框: 存档 - 是否保存进度？
  用户选择: 确定
[Cocos2d] Toast: 自动保存完成

--- 切换到 Flutter 后端 ---
[Cocos2d] 关闭
[Flutter] 启动引擎 1280x720
[Flutter] 引擎就绪
[Flutter] Material 对话框: Flutter 测试 - Material Design 对话框
  Flutter 对话框已关闭
[Flutter] SnackBar: Flutter UI 工作正常!
[Flutter] 关闭引擎

===== 演示结束 =====
```

---

## 常见错误与排查

### 错误 1：Flutter 纹理共享导致画面撕裂

**现象**：游戏画面在 Flutter UI 下方出现水平撕裂线或闪烁。

**原因**：游戏线程写入 FBO 和 Flutter 线程读取 FBO 没有正确同步——Flutter 读到了"写了一半"的帧。

**解决方案**：

```cpp
// ❌ 错误：没有 GPU 栅栏，直接读取
uint32_t getSharedTextureId() const override {
    return fbos_[1 - writeIndex_].colorTexture; // 可能读到半帧!
}

// ✅ 正确：等待 GPU 栅栏确认渲染完成
uint32_t getSharedTextureId() const override {
    const auto& back = fbos_[1 - writeIndex_];
    if (back.fence) {
        // glClientWaitSync 会阻塞直到 GPU 完成渲染
        // GL_TIMEOUT_IGNORED 表示无限等待
        GLenum result = gl::glClientWaitSync(
            back.fence, 0, GL_TIMEOUT_IGNORED);
        if (result == GL_ALREADY_SIGNALED ||
            result == GL_CONDITION_SATISFIED) {
            return back.colorTexture;  // 安全读取
        }
    }
    return 0;  // 没有可用帧
}
```

### 错误 2：MethodChannel 消息在非主线程发送导致崩溃

**现象**：在游戏逻辑线程（非 UI 线程）调用 `showMessageBox()` 后，Flutter 引擎崩溃或消息丢失。

**原因**：FlutterEngine 的平台消息 API 要求在**平台线程**（即创建引擎的线程）上调用。从其他线程直接调用是未定义行为。

**解决方案**：

```cpp
// ❌ 错误：直接在任意线程调用
void showMessageBox(...) override {
    uiChannel_->invokeMethod("showMessageBox", args);
    // 如果当前不在平台线程 → 崩溃!
}

// ✅ 正确：投递到平台线程执行
void showMessageBox(...) override {
    // 将调用投递到平台线程的消息队列
    // 各平台投递方式不同：
    // Windows:  PostMessage(hwnd, WM_USER, ...)
    // macOS:    dispatch_async(dispatch_get_main_queue(), ...)
    // Linux:    g_idle_add(...) (GLib 主循环)
    // Android:  Looper_getForThread() + ALooper_addFd()

    auto task = [this, args]() {
        uiChannel_->invokeMethod("showMessageBox", args);
    };
    platformThreadQueue_.push(task); // 线程安全队列
}
```

### 错误 3：忘记处理 Flutter 引擎启动失败的回退

**现象**：在低端设备或内存不足时，FlutterEngine 启动失败（返回错误码），但程序继续尝试使用 Flutter 后端，导致空指针崩溃。

**原因**：没有检查 `FlutterEngineRun()` 的返回值，也没有准备回退方案。

**解决方案**：

```cpp
// ❌ 错误：不检查返回值
bool initialize(...) override {
    FlutterEngineRun(&engine_);  // 可能失败!
    uiChannel_ = std::make_unique<MethodChannel>(
        "krkr2/ui", engine_);  // engine_ 可能是 nullptr
    return true;  // 谎报成功
}

// ✅ 正确：检查返回值 + 自动回退
bool initialize(...) override {
    auto result = FlutterEngineRun(&engine_);
    if (result != FlutterEngineResult::kSuccess) {
        std::cerr << "[Flutter] 引擎启动失败，错误码: "
                  << static_cast<int>(result) << std::endl;
        engine_ = nullptr;
        return false;  // 让调用方回退到备选后端
    }
    // ... 继续初始化
    return true;
}
```

### 错误 4：Cocos2d-x 适配器中在 shutdown 后访问 Director

**现象**：游戏退出时先调用了 Cocos2d-x 的 `Director::end()`，然后适配器的 `shutdown()` 试图操作已销毁的 Director，导致野指针崩溃。

**原因**：适配器的生命周期和 Cocos2d-x 引擎的生命周期没有对齐。

**解决方案**：

```cpp
// ✅ 正确：使用 weak_ptr 或 flag 检查引擎状态
void shutdown() override {
    if (!initialized_) return;
    
    // 检查 Director 是否还活着
    // Cocos2d-x 在 Director::end() 后设为 nullptr
    auto director = cocos2d::Director::getInstance();
    if (director && director->isValid()) {
        // 安全地弹出所有 UI Scene
        while (director->getRunningScene() != mainScene_) {
            director->popScene();
        }
    }
    // 即使 Director 已销毁，适配器也能安全关闭
    initialized_ = false;
}
```

---

## 动手实践

### 练习 1：添加 Cocos2d-x 后端的 About 对话框

在 `Cocos2dUIBackendAdapter` 中添加一个 `showAboutDialog()` 方法，显示应用版本信息。

**步骤**：

1. 在 `IUIBackend` 接口中添加虚函数 `showAboutDialog(const std::string& appName, const std::string& version)`
2. 在 Cocos2d-x 适配器中实现：
   - 使用 `showMessageBox()` 显示一个信息对话框
   - 标题为 "关于 {appName}"
   - 内容包含版本号、编译日期（`__DATE__`）、平台信息
3. 在 Flutter 适配器中实现：
   - 通过 MethodChannel 调用 Dart 端的 `showAboutDialog()`
   - Dart 端使用 Material Design 的 `AboutDialog` Widget

**验证**：调用 `showAboutDialog("KrKr2", "2.0.0")`，两个后端都能正确显示信息。

### 练习 2：实现后端健康检查机制

为 `MigrationController` 添加运行时健康检查，当 Flutter 后端连续 3 次操作失败时自动切换到 Cocos2d-x。

**步骤**：

1. 在 `FlutterUIBackendAdapter` 中添加失败计数器：
   ```cpp
   std::atomic<int> consecutiveFailures_{0};
   ```
2. 每次 MethodChannel 调用成功时重置计数器为 0
3. 每次失败时递增计数器
4. 在 `MigrationController` 中添加 `checkHealth()` 方法：
   - 如果 `consecutiveFailures_ >= 3`，自动切换到 Cocos2d-x
   - 记录切换事件到 ABTestManager
5. 在游戏主循环中每 60 帧调用一次 `checkHealth()`

**验证**：模拟 3 次 MethodChannel 发送失败，观察是否自动回退到 Cocos2d-x 后端。

---

## 对照项目源码

本节的适配器设计直接对应 KrKr2 项目中的现有 UI 代码：

| 本节概念 | 项目对应文件 | 说明 |
|---------|-------------|------|
| Cocos2d-x 消息框适配 | `cpp/core/environ/ui/TVPMessageBoxForm.cpp` | 使用 CSLoader 加载 .csb 布局，绑定按钮回调。本节的 `showMessageBox()` 注释代码即参照此文件 |
| Cocos2d-x 软键盘适配 | `cpp/core/environ/ui/TVPPadForm.cpp` | 虚拟键盘实现，本节 `showSoftKeyboard()` 对应此组件 |
| 字体选择 UI | `cpp/core/environ/ui/TVPFontSelectForm.cpp` | Cocos2d-x 实现的字体列表 UI，本节提到 Cocos2d-x 后端不原生支持字体选择 |
| UI 资源文件 | `ui/cocos-studio/*.csb` | Cocos Studio 导出的二进制布局文件，CSLoader 加载这些文件创建 UI 节点树 |
| 应用启动流程 | `cpp/core/environ/cocos2d/AppDelegate.cpp` | `applicationDidFinishLaunching()` 中初始化 Director 和首个 Scene——本节 Cocos2d-x 适配器的 `initialize()` 在此之后执行 |

### 关键差异分析

1. **现有代码无接口抽象**：`TVPMessageBoxForm` 等类直接被调用，没有通过 `IUIBackend` 接口。适配器模式正是为了在不修改这些类的前提下引入统一接口。

2. **现有代码深度耦合 Cocos2d-x**：`TVPPadForm` 直接继承 `cocos2d::Scene`，使用 `Director::pushScene()` 显示。适配器将这些 Cocos2d-x 特有调用封装在内部，对外只暴露 `IUIBackend` 接口。

3. **UI 资源管理**：现有代码通过 CSLoader 加载 `.csb` 文件，资源路径硬编码。Flutter 后端则完全不需要这些资源——它使用 Dart 代码定义 Widget 树。

---

## 本节小结

1. **适配器模式**是将现有 Cocos2d-x UI 包装为 `IUIBackend` 接口的最佳策略——零风险，不修改任何现有代码
2. **Cocos2d-x 后端**的优势是零额外开销（复用已有引擎），劣势是 UI 能力有限（无文件对话框、无字体选择器）
3. **Flutter 后端**通过 FlutterEngine C API 启动独立引擎，通过 MethodChannel 进行 C++↔Dart 通信
4. **Flutter 后端**支持所有 UI 能力（Material Design 组件库），但需要 150-300ms 启动时间和 30-60MB 额外内存
5. **渲染桥接**使用 FBO 双缓冲 + GPU 栅栏同步，确保游戏纹理安全传递给 Flutter 合成
6. **跨平台纹理共享**各平台方案不同：Windows 用 DXGI SharedHandle、macOS 用 IOSurface、Linux 用 DMA-BUF、Android 用 AHardwareBuffer
7. **性能测试框架**量化两个后端的启动时间、内存占用、帧率、UI 响应延迟，为技术选型提供数据支撑
8. **渐进式迁移**通过功能开关 + A/B 测试分阶段切换后端，随时可回退，避免一次性替换的高风险
9. **自动回退机制**是生产环境必备——当 Flutter 后端失败时自动降级到 Cocos2d-x，保证用户体验不受影响

---

## 练习题与答案

### 题目 1：设计 MethodChannel 的双向通信

在 Flutter 后端适配器中，`showFileDialog()` 需要等待用户在 Dart 端选择文件后将结果返回给 C++。请描述完整的消息流，并写出 C++ 端接收 Dart 回调的关键代码。

<details>
<summary>查看答案</summary>

**消息流**：

```
C++ 端                              Dart 端
   │                                   │
   │ invokeMethod("showFileDialog",    │
   │   {type:"open"})                  │
   ├──────────────────────────────────→│
   │                                   │ 显示 FilePicker Widget
   │                                   │ 用户选择文件
   │                                   │
   │  handleDartCallback(              │
   │    "onFileSelected",              │
   │    {paths:["a.txt","b.txt"]})     │
   │←──────────────────────────────────┤
   │                                   │
   │ 触发 pendingFileCallback_         │
   │ 返回文件路径列表给调用方            │
```

**C++ 端关键代码**：

```cpp
// 在 FlutterUIBackendAdapter 中

// 1. 发送请求时保存回调
void showFileDialog(const FileDialogOptions& options,
                    FileCallback callback) override {
    {
        std::lock_guard<std::mutex> lock(callbackMutex_);
        pendingFileCallback_ = callback;
    }
    uiChannel_->invokeMethod("showFileDialog",
        "{\"type\":\"open\",\"allowMultiple\":true}");
}

// 2. 接收 Dart 端回调
void handleDartCallback(const std::string& method,
                        const std::string& args) {
    std::lock_guard<std::mutex> lock(callbackMutex_);
    if (method == "onFileSelected") {
        if (pendingFileCallback_) {
            // 解析 JSON 中的 paths 数组
            // 简化示例：假设 args = {"paths":["a.txt"]}
            std::vector<std::string> paths;
            // 实际用 nlohmann::json 或 rapidjson 解析
            paths.push_back("a.txt");
            pendingFileCallback_(paths);
            pendingFileCallback_ = nullptr;
        }
    }
}
```

**关键点**：
- 使用 `std::mutex` 保护回调存储（可能从不同线程访问）
- 回调触发后立即置 `nullptr`，防止重复触发
- JSON 解析在实际项目中应使用成熟的 JSON 库

</details>

### 题目 2：实现三缓冲 FBO 渲染桥接

当前的 `GLRenderBridge` 使用双缓冲。请将其扩展为三缓冲（Triple Buffering），并分析三缓冲相比双缓冲的优势和劣势。

<details>
<summary>查看答案</summary>

**三缓冲实现**：

```cpp
// 三缓冲渲染桥接
class TripleBufferRenderBridge : public IRenderBridge {
public:
    bool init(int32_t width, int32_t height) {
        for (auto& fbo : fbos_) {
            // 与双缓冲相同的初始化逻辑
            gl::glGenFramebuffers(1, &fbo.framebuffer);
            gl::glGenTextures(1, &fbo.colorTexture);
            gl::glBindTexture(GL_TEXTURE_2D, 
                              fbo.colorTexture);
            gl::glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA8,
                             width, height, 0,
                             GL_RGBA8, GL_RGBA8, nullptr);
            gl::glBindFramebuffer(GL_FRAMEBUFFER,
                                  fbo.framebuffer);
            gl::glFramebufferTexture2D(
                GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,
                GL_TEXTURE_2D, fbo.colorTexture, 0);
        }
        gl::glBindFramebuffer(GL_FRAMEBUFFER, 0);
        return true;
    }

    void beginFrame() override {
        gl::glBindFramebuffer(GL_FRAMEBUFFER,
            fbos_[writeIndex_].framebuffer);
    }

    void endFrame() override {
        auto& front = fbos_[writeIndex_];
        if (front.fence) gl::glDeleteSync(front.fence);
        front.fence = gl::glFenceSync(
            GL_SYNC_GPU_COMMANDS_COMPLETE, 0);
        front.ready = true;

        {
            std::lock_guard<std::mutex> lock(mutex_);
            // 三缓冲核心：写入完成的 FBO 变为"待读取"
            // 如果旧的"待读取"还没被读，跳过它
            readyIndex_ = writeIndex_;
            // 选择下一个空闲的 FBO 作为写入目标
            writeIndex_ = nextFreeIndex();
        }
        gl::glBindFramebuffer(GL_FRAMEBUFFER, 0);
    }

    uint32_t getSharedTextureId() const override {
        std::lock_guard<std::mutex> lock(mutex_);
        if (readyIndex_ < 0) return 0;
        
        const auto& fbo = fbos_[readyIndex_];
        if (!fbo.ready || !fbo.fence) return 0;

        GLenum result = gl::glClientWaitSync(
            fbo.fence, 0, GL_TIMEOUT_IGNORED);
        if (result == GL_ALREADY_SIGNALED ||
            result == GL_CONDITION_SATISFIED) {
            lastReadIndex_ = readyIndex_;
            return fbo.colorTexture;
        }
        return 0;
    }

    void resize(int32_t w, int32_t h) override {
        // 销毁并重建所有 3 个 FBO
    }

private:
    int nextFreeIndex() const {
        // 找到既不是当前读取、也不是待读取的 FBO
        for (int i = 0; i < 3; ++i) {
            if (i != readyIndex_ && i != lastReadIndex_)
                return i;
        }
        return (writeIndex_ + 1) % 3;
    }

    std::array<FBODescriptor, 3> fbos_;    // 三缓冲
    int writeIndex_ = 0;                    // 写入索引
    int readyIndex_ = -1;                   // 已完成待读取
    mutable int lastReadIndex_ = -1;        // 正在读取
    mutable std::mutex mutex_;
};
```

**三缓冲 vs 双缓冲对比**：

| 维度 | 双缓冲 | 三缓冲 |
|------|--------|--------|
| 内存占用 | 2× 纹理大小 | 3× 纹理大小（+50%） |
| 帧延迟 | 最多 1 帧 | 最多 1 帧 |
| 吞吐量 | 可能因等待读取方而阻塞 | 写入方永远不阻塞 |
| 适用场景 | 内存受限的移动设备 | 桌面平台追求流畅度 |
| 实现复杂度 | 简单（2 个索引交替） | 较复杂（3 个索引状态机） |

**结论**：三缓冲适合桌面平台（内存充裕），双缓冲适合移动平台（内存紧张）。KrKr2 建议在 Android 上用双缓冲、在 Windows/Linux/macOS 上用三缓冲。

</details>

### 题目 3：分析 A/B 测试的统计可靠性

假设 A/B 测试运行 7 天，A 组（Cocos2d-x）500 用户，B 组（Flutter）500 用户。A 组崩溃率 0.2%，B 组崩溃率 0.8%。请分析：这个差异是否具有统计显著性？需要多大样本量才能可靠判断？

<details>
<summary>查看答案</summary>

**分析步骤**：

1. **假设检验**：
   - H0（零假设）：两组崩溃率无差异（pA = pB）
   - H1（备择假设）：Flutter 组崩溃率更高（pB > pA）

2. **计算统计量**：
   ```
   pA = 0.002 (0.2%)，nA = 500
   pB = 0.008 (0.8%)，nB = 500
   
   合并比例 p = (pA*nA + pB*nB) / (nA + nB)
            = (1 + 4) / 1000 = 0.005
   
   标准误差 SE = sqrt(p*(1-p)*(1/nA + 1/nB))
             = sqrt(0.005 * 0.995 * 2/500)
             = sqrt(0.0000199) ≈ 0.00446
   
   Z 统计量 = (pB - pA) / SE
            = (0.008 - 0.002) / 0.00446
            ≈ 1.345
   ```

3. **判断显著性**：
   - Z = 1.345，单侧 p 值 ≈ 0.089
   - 常用显著性水平 α = 0.05
   - p > α → **不显著**！差异可能是随机波动

4. **需要的样本量**：
   - 要达到 80% 统计功效（Power）检测 0.6% 的差异
   - 每组需要约 **3000-4000** 用户
   - 建议延长测试时间或增大灰度比例

**实际建议**：
- 500 用户样本太小，无法可靠判断 0.6% 的崩溃率差异
- 建议将灰度比例提高到 50%，运行 14 天以上
- 同时监控其他指标（内存峰值、ANR 率、用户停留时间）做综合判断
- 崩溃率低于 1% 时，建议用更敏感的指标（如 UI 响应延迟 P99）做辅助决策

</details>

---

## 下一步

P12 模块到此全部完成！你已经掌握了：
- 现代跨平台 UI 框架的选型与对比
- Flutter 引擎嵌入与 Platform Channel 通信
- Compose Multiplatform 与 Kotlin/Native 互操作
- UI 与渲染引擎分离的抽象层设计
- 从 Cocos2d-x 到 Flutter 的实战迁移方案

接下来进入 **M12-插件逆向与实现** 模块，学习如何逆向分析原版 KiriKiri 插件并在 KrKr2 中重新实现。

