# KrKr2 UI 抽象层接口定义

> **所属模块：** P12-现代跨平台UI
> **前置知识：** [抽象UI接口层设计](../04-UI与渲染引擎分离/01-抽象UI接口层设计.md)、[事件桥接与共享纹理](../04-UI与渲染引擎分离/02-事件桥接与共享纹理.md)
> **预计阅读时间：** 55 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：
1. 根据 KrKr2 的实际 UI 需求，定义一套完整的 UI 抽象层接口（IUIBackend、IUIComponent 等）
2. 设计支持 Cocos2d-x、Flutter、Compose Multiplatform 三种后端的接口协议
3. 实现 UI 组件的生命周期管理（创建、挂载、更新、卸载、销毁）
4. 定义渲染桥接接口（IRenderBridge），使游戏画面能通过纹理传递给任意 UI 后端
5. 设计接口版本控制与能力查询机制，确保新旧后端的向前/向后兼容

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| UI 后端 | UI Backend | 具体实现 UI 渲染和事件处理的框架（如 Cocos2d-x、Flutter） |
| 接口协议 | Interface Protocol | 一组纯虚函数定义，规定了后端必须实现哪些功能 |
| 渲染桥接 | Render Bridge | 将游戏引擎的渲染输出传递给 UI 框架显示的中间层 |
| 组件生命周期 | Component Lifecycle | UI 组件从创建到销毁的完整状态流转过程 |
| 能力查询 | Capability Query | 运行时检查 UI 后端是否支持某个特性的机制 |
| 工厂模式 | Factory Pattern | 通过统一接口创建不同类型对象的设计模式 |
| 依赖注入 | Dependency Injection | 将依赖关系从外部传入而非内部创建的设计原则 |
| PIMPL | Pointer to IMPLementation | 用指针隐藏实现细节的 C++ 惯用法，实现编译防火墙 |

---

## 一、KrKr2 UI 需求分析

在定义接口之前，必须先梳理 KrKr2 作为视觉小说引擎需要哪些 UI 功能。通过分析项目现有的 `cpp/core/environ/ui/` 目录和 `ui/cocos-studio/` 资源，可以归纳出以下 UI 组件需求：

### 1.1 必需 UI 组件清单

| 组件 | 现有实现 | 功能描述 | 优先级 |
|------|----------|----------|--------|
| 消息对话框 | `TVPMessageBoxForm` | 系统确认/取消对话框（如"是否退出"） | P0 必需 |
| 主窗口 | `TVPMainForm` | 游戏主画面容器 + 菜单栏 | P0 必需 |
| 软键盘 | `TVPPadForm` | Android/触屏设备的虚拟键盘 | P1 重要 |
| 字体选择器 | `TVPFontSelectForm` | 游戏内字体切换界面 | P2 次要 |
| 文件选择器 | (无独立实现) | 加载存档/选择游戏文件 | P1 重要 |
| 设置面板 | `ConfigManager` 系列 | 游戏设置（音量/文字速度/全屏等） | P1 重要 |
| 进度条 | (内联实现) | 加载进度、视频播放进度 | P2 次要 |
| 右键菜单 | (TJS2 脚本控制) | 存档/读档/设置/退出菜单 | P0 必需 |
| Toast 提示 | (无) | 自动存档提示、截图成功等 | P2 次要 |

### 1.2 渲染需求

KrKr2 的渲染架构决定了 UI 抽象层必须支持以下渲染模式：

```
模式 A：UI 框架渲染游戏画面（推荐）
┌─────────────────────────────────┐
│        UI 框架窗口              │
│  ┌───────────────────────────┐  │
│  │  游戏画面纹理（共享纹理）  │  │
│  │  (由 Cocos2d-x 渲染)      │  │
│  └───────────────────────────┘  │
│  ┌──────┐ ┌──────┐ ┌────────┐  │
│  │存档  │ │设置  │ │ 退出   │  │ ← UI 框架原生渲染
│  └──────┘ └──────┘ └────────┘  │
└─────────────────────────────────┘

模式 B：游戏引擎渲染 UI 覆盖层
┌─────────────────────────────────┐
│      Cocos2d-x 窗口             │
│  ┌───────────────────────────┐  │
│  │  游戏画面（原生渲染）      │  │
│  │  + UI 覆盖层（Cocos2d-x） │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

模式 A 是目标架构（Flutter/Compose 作为宿主），模式 B 是当前架构（Cocos2d-x 同时做游戏渲染和 UI）。抽象层需要同时支持两种模式，以便渐进式迁移。

---

## 二、示例 1：核心接口定义——IUIBackend

`IUIBackend` 是整个 UI 抽象层的顶层接口，定义了 UI 后端必须实现的所有能力。任何新增的 UI 框架（Flutter、Compose、Dear ImGui）都需要实现这个接口。

```cpp
// krkr2_ui_backend.h — KrKr2 UI 后端顶层接口
#pragma once

#include <cstdint>
#include <string>
#include <memory>
#include <vector>
#include <functional>
#include <optional>

namespace krkr2::ui {

// ========== 基础类型（与具体框架无关）==========

// 颜色（RGBA，每通道 0~255）
struct Color {
    uint8_t r = 0, g = 0, b = 0, a = 255;
    
    static Color black()   { return {0, 0, 0, 255}; }
    static Color white()   { return {255, 255, 255, 255}; }
    static Color transparent() { return {0, 0, 0, 0}; }
};

// 二维坐标
struct Vec2 {
    float x = 0, y = 0;
};

// 矩形区域
struct UIRect {
    float x = 0, y = 0, width = 0, height = 0;
};

// 字体描述
struct FontDesc {
    std::string family = "sans-serif";  // 字体族
    float       size   = 16.0f;        // 字号（逻辑像素）
    bool        bold   = false;
    bool        italic = false;
};

// 图像数据（用于传递图标/缩略图）
struct ImageData {
    std::vector<uint8_t> pixels;  // RGBA8 像素数据
    int32_t width  = 0;
    int32_t height = 0;
    int32_t stride = 0;           // 每行字节数
};

// ========== UI 后端能力枚举 ==========

// 后端支持的能力标记——不同框架能力不同
enum class UICapability : uint32_t {
    MessageBox      = 1 << 0,   // 支持消息对话框
    FileDialog      = 1 << 1,   // 支持文件选择器
    FontSelector    = 1 << 2,   // 支持字体选择器
    SoftKeyboard    = 1 << 3,   // 支持软键盘
    ContextMenu     = 1 << 4,   // 支持右键菜单
    Settings        = 1 << 5,   // 支持设置面板
    Toast           = 1 << 6,   // 支持 Toast 提示
    Progress        = 1 << 7,   // 支持进度条
    TextureSharing  = 1 << 8,   // 支持纹理共享（GPU 级别）
    NativeAnimation = 1 << 9,   // 支持原生动画
    Accessibility   = 1 << 10,  // 支持无障碍功能
    DarkMode        = 1 << 11,  // 支持深色模式
};

// 位运算支持
inline UICapability operator|(UICapability a, UICapability b) {
    return static_cast<UICapability>(
        static_cast<uint32_t>(a) | static_cast<uint32_t>(b));
}
inline bool operator&(UICapability a, UICapability b) {
    return (static_cast<uint32_t>(a) & static_cast<uint32_t>(b)) != 0;
}

// ========== 对话框结果 ==========

enum class DialogResult {
    OK,         // 用户点击"确定"
    Cancel,     // 用户点击"取消"
    Yes,        // 用户点击"是"
    No,         // 用户点击"否"
    Closed      // 用户关闭了对话框（X 按钮/返回键）
};

// 对话框按钮类型
enum class DialogButtons {
    OK,             // 只有"确定"
    OKCancel,       // "确定" + "取消"
    YesNo,          // "是" + "否"
    YesNoCancel     // "是" + "否" + "取消"
};

// 对话框图标类型
enum class DialogIcon {
    None,
    Info,
    Warning,
    Error,
    Question
};

// ========== 文件对话框选项 ==========

struct FileDialogOptions {
    std::string title = "选择文件";
    std::string defaultPath;
    std::vector<std::pair<std::string, std::string>> filters;
    // 示例: {{"TJS脚本", "*.tjs"}, {"所有文件", "*.*"}}
    bool allowMultiple = false;
    bool selectFolder  = false;
};

// ========== 菜单项 ==========

struct MenuItem {
    std::string id;           // 唯一标识
    std::string label;        // 显示文本
    std::string shortcutKey;  // 快捷键提示文本
    bool        enabled = true;
    bool        checked = false;
    bool        separator = false; // true 表示分隔线
    std::vector<MenuItem> children; // 子菜单
};

// ========== 回调类型 ==========

using DialogCallback     = std::function<void(DialogResult)>;
using FileCallback       = std::function<void(const std::vector<std::string>&)>;
using MenuCallback       = std::function<void(const std::string& itemId)>;
using ProgressCallback   = std::function<void(float progress)>; // 0.0~1.0
using TextInputCallback  = std::function<void(const std::string& text)>;

// ========== 顶层接口：IUIBackend ==========

class IUIBackend {
public:
    virtual ~IUIBackend() = default;

    // ---- 生命周期 ----
    
    // 初始化 UI 后端（在 GL 上下文创建后调用）
    // 参数 windowHandle 是平台原生窗口句柄：
    //   Windows: HWND, macOS: NSWindow*, Linux: Window (X11), 
    //   Android: ANativeWindow*
    virtual bool initialize(void* windowHandle, 
                            int32_t width, int32_t height) = 0;
    
    // 关闭并释放所有 UI 资源
    virtual void shutdown() = 0;
    
    // 每帧更新（在游戏主循环中调用）
    virtual void update(float deltaTime) = 0;

    // ---- 能力查询 ----
    
    // 查询后端支持哪些能力
    virtual UICapability capabilities() const = 0;
    
    // 查询后端名称（如 "Cocos2d-x", "Flutter", "Compose"）
    virtual const char* backendName() const = 0;
    
    // 查询接口版本（用于兼容性检查）
    virtual uint32_t interfaceVersion() const = 0;

    // ---- 对话框 ----
    
    // 显示消息对话框（异步，结果通过回调返回）
    virtual void showMessageBox(
        const std::string& title,
        const std::string& message,
        DialogButtons buttons,
        DialogIcon icon,
        DialogCallback callback) = 0;
    
    // 显示文件选择对话框
    virtual void showFileDialog(
        const FileDialogOptions& options,
        FileCallback callback) = 0;

    // ---- 菜单 ----
    
    // 显示右键/上下文菜单
    virtual void showContextMenu(
        const std::vector<MenuItem>& items,
        Vec2 position,
        MenuCallback callback) = 0;
    
    // 设置应用主菜单栏（桌面平台）
    virtual void setMenuBar(
        const std::vector<MenuItem>& items,
        MenuCallback callback) = 0;

    // ---- 设置面板 ----
    
    // 显示设置面板
    virtual void showSettings() = 0;

    // ---- Toast 提示 ----
    
    // 显示短暂提示消息
    virtual void showToast(const std::string& message, 
                           float durationSec = 2.0f) = 0;

    // ---- 进度条 ----
    
    // 显示/更新进度条
    virtual void showProgress(const std::string& title, 
                              float progress) = 0;
    virtual void hideProgress() = 0;

    // ---- 软键盘（移动平台）----
    
    // 显示/隐藏软键盘
    virtual void showSoftKeyboard(TextInputCallback callback) = 0;
    virtual void hideSoftKeyboard() = 0;

    // ---- 字体选择器 ----
    
    // 显示字体选择器
    virtual void showFontSelector(
        const FontDesc& currentFont,
        std::function<void(const FontDesc&)> callback) = 0;

    // ---- 渲染桥接 ----
    
    // 获取渲染桥接接口（用于纹理共享）
    virtual class IRenderBridge* renderBridge() = 0;
};

} // namespace krkr2::ui
```

这个接口设计有以下关键考量：

1. **异步回调模式**：所有用户交互操作（对话框、文件选择等）都使用回调返回结果，不会阻塞游戏主循环。这对于 Flutter/Compose 等异步 UI 框架很重要——它们的对话框本身就是异步的
2. **能力查询**：不同后端能力不同。Cocos2d-x 后端可能不支持原生文件对话框（移动平台），Dear ImGui 后端可能不支持无障碍功能。调用前先查询能力可以优雅降级
3. **平台窗口句柄**：`void*` 类型的 windowHandle 虽然类型不安全，但这是跨平台 C++ 传递原生句柄的标准做法

---

## 三、示例 2：渲染桥接接口——IRenderBridge

`IRenderBridge` 是连接游戏渲染引擎和 UI 框架的关键接口。它定义了纹理共享的协议：

```cpp
// krkr2_render_bridge.h — 渲染桥接接口
#pragma once

#include "krkr2_ui_backend.h"

namespace krkr2::ui {

// 纹理格式
enum class TextureFormat {
    RGBA8,      // 每通道 8 位，最常用
    BGRA8,      // Windows/macOS 原生像素格式
    RGB565,     // 16 位低质量（省内存）
    RGBA16F     // 16 位浮点（HDR 场景）
};

// 纹理描述符
struct TextureDesc {
    int32_t       width   = 0;
    int32_t       height  = 0;
    TextureFormat format  = TextureFormat::RGBA8;
    bool          mipmaps = false;  // 是否生成 Mipmap
};

// 共享纹理句柄——对 UI 后端来说是不透明的
// 不同后端用不同方式解释这个句柄：
//   GL 后端：glTexture ID
//   Metal 后端：MTLTexture 指针
//   D3D 后端：ID3D11ShaderResourceView 指针
using SharedTextureHandle = uintptr_t;

// 纹理锁——RAII 风格的纹理访问控制
// 在 lock 期间保证纹理数据不会被修改
class TextureLock {
public:
    virtual ~TextureLock() = default;
    
    // 获取锁定的纹理句柄
    virtual SharedTextureHandle handle() const = 0;
    
    // 纹理是否有效（可能由于上下文丢失而失效）
    virtual bool valid() const = 0;
};

// ========== 渲染桥接接口 ==========

class IRenderBridge {
public:
    virtual ~IRenderBridge() = default;

    // ---- 纹理管理 ----

    // 创建共享纹理（由游戏引擎调用）
    // 返回纹理 ID，后续操作使用此 ID 引用
    virtual int32_t createSharedTexture(const TextureDesc& desc) = 0;

    // 销毁共享纹理
    virtual void destroySharedTexture(int32_t textureId) = 0;

    // 调整共享纹理大小（窗口大小改变时）
    virtual bool resizeSharedTexture(int32_t textureId,
                                     int32_t width, 
                                     int32_t height) = 0;

    // ---- 生产者端（游戏引擎调用）----

    // 开始写入一帧到共享纹理
    // 返回纹理锁——在锁的生命期内可以安全渲染到纹理
    virtual std::unique_ptr<TextureLock> beginWrite(
        int32_t textureId) = 0;

    // 结束写入（释放 TextureLock 时自动调用，也可显式调用）
    virtual void endWrite(int32_t textureId) = 0;

    // ---- 消费者端（UI 框架调用）----

    // 获取最新帧的纹理句柄（用于 UI 框架采样显示）
    virtual std::unique_ptr<TextureLock> acquireLatestFrame(
        int32_t textureId) = 0;

    // ---- 同步控制 ----

    // 等待生产者完成当前帧
    // 返回 true 表示帧已就绪，false 表示超时
    virtual bool waitForFrame(int32_t textureId, 
                              uint64_t timeoutMs) = 0;

    // 获取当前帧号
    virtual uint64_t currentFrameNumber(int32_t textureId) const = 0;

    // ---- 状态查询 ----

    // 获取纹理描述信息
    virtual TextureDesc getTextureDesc(int32_t textureId) const = 0;

    // 检查纹理是否有效
    virtual bool isTextureValid(int32_t textureId) const = 0;

    // ---- 像素回读（调试/截图用，非性能关键路径）----

    // 将纹理内容读回 CPU 内存
    virtual bool readPixels(int32_t textureId, 
                            ImageData& outImage) = 0;
};

} // namespace krkr2::ui
```

### 3.1 TextureLock 的 RAII 设计

`TextureLock` 使用 RAII（Resource Acquisition Is Initialization）模式——获取锁时自动加锁，析构时自动解锁。这避免了忘记释放锁导致的死锁问题：

```cpp
// 使用示例：自动锁管理
void gameRenderLoop(krkr2::ui::IRenderBridge* bridge, int texId) {
    // beginWrite 返回 unique_ptr<TextureLock>
    // 当 lock 离开作用域时自动释放
    {
        auto lock = bridge->beginWrite(texId);
        if (!lock || !lock->valid()) {
            return; // 纹理不可用，跳过这一帧
        }
        
        // 获取纹理句柄（GL 纹理 ID）
        auto handle = lock->handle();
        
        // ... 渲染游戏画面到这个纹理 ...
        
    } // lock 析构 → 自动调用 endWrite → 插入 Fence
    
    // 这里纹理已经安全释放，消费者可以读取
}
```

---

## 四、示例 3：UI 组件基类与生命周期管理

每个 UI 组件（对话框、菜单、进度条等）都有创建、显示、隐藏、销毁的生命周期。以下基类管理这个生命周期：

```cpp
// krkr2_ui_component.h — UI 组件基类
#pragma once

#include "krkr2_ui_backend.h"
#include <string>
#include <functional>
#include <iostream>
#include <chrono>
#include <cassert>

namespace krkr2::ui {

// 组件状态
enum class ComponentState {
    Created,    // 已创建，未显示
    Showing,    // 正在显示动画中
    Visible,    // 已显示，可交互
    Hiding,     // 正在隐藏动画中
    Hidden,     // 已隐藏，不可见但资源仍在
    Destroyed   // 已销毁，资源已释放
};

// 获取状态名称（调试用）
inline const char* stateName(ComponentState s) {
    switch (s) {
    case ComponentState::Created:   return "Created";
    case ComponentState::Showing:   return "Showing";
    case ComponentState::Visible:   return "Visible";
    case ComponentState::Hiding:    return "Hiding";
    case ComponentState::Hidden:    return "Hidden";
    case ComponentState::Destroyed: return "Destroyed";
    }
    return "Unknown";
}

// UI 组件基类
class UIComponent {
public:
    UIComponent(std::string id) : id_(std::move(id)) {
        state_ = ComponentState::Created;
        logTransition("(init)", ComponentState::Created);
    }

    virtual ~UIComponent() {
        if (state_ != ComponentState::Destroyed) {
            std::cerr << "[UIComponent:" << id_ << "] 警告: "
                      << "组件未正确销毁就被析构" << std::endl;
        }
    }

    // 禁止拷贝
    UIComponent(const UIComponent&) = delete;
    UIComponent& operator=(const UIComponent&) = delete;

    // ---- 生命周期方法 ----

    // 显示组件
    void show() {
        assert(state_ == ComponentState::Created 
            || state_ == ComponentState::Hidden);
        transition(ComponentState::Showing);
        onShow();
    }

    // 隐藏组件
    void hide() {
        assert(state_ == ComponentState::Visible 
            || state_ == ComponentState::Showing);
        transition(ComponentState::Hiding);
        onHide();
    }

    // 销毁组件（释放所有资源）
    void destroy() {
        if (state_ == ComponentState::Visible 
            || state_ == ComponentState::Showing) {
            hide(); // 先隐藏再销毁
        }
        transition(ComponentState::Destroyed);
        onDestroy();
    }

    // 每帧更新（处理动画等）
    void update(float dt) {
        switch (state_) {
        case ComponentState::Showing:
            showProgress_ += dt / showDuration_;
            if (showProgress_ >= 1.0f) {
                showProgress_ = 1.0f;
                transition(ComponentState::Visible);
                onBecameVisible();
            }
            break;
        case ComponentState::Hiding:
            hideProgress_ += dt / hideDuration_;
            if (hideProgress_ >= 1.0f) {
                hideProgress_ = 1.0f;
                transition(ComponentState::Hidden);
                onBecameHidden();
            }
            break;
        case ComponentState::Visible:
            onUpdate(dt);
            break;
        default:
            break;
        }
    }

    // ---- 属性访问 ----

    const std::string& id() const { return id_; }
    ComponentState state() const { return state_; }
    bool isVisible() const { 
        return state_ == ComponentState::Visible 
            || state_ == ComponentState::Showing; 
    }
    
    // 动画透明度（0.0=隐藏, 1.0=完全显示）
    float opacity() const {
        switch (state_) {
        case ComponentState::Showing: return showProgress_;
        case ComponentState::Visible: return 1.0f;
        case ComponentState::Hiding:  return 1.0f - hideProgress_;
        default: return 0.0f;
        }
    }

protected:
    // 子类重写这些方法实现具体行为
    virtual void onShow() {}
    virtual void onHide() {}
    virtual void onBecameVisible() {}
    virtual void onBecameHidden() {}
    virtual void onUpdate(float dt) {}
    virtual void onDestroy() {}

    // 动画时长（子类可调整）
    float showDuration_ = 0.3f;  // 显示动画 0.3 秒
    float hideDuration_ = 0.2f;  // 隐藏动画 0.2 秒

private:
    std::string    id_;
    ComponentState state_;
    float          showProgress_ = 0.0f;
    float          hideProgress_ = 0.0f;

    void transition(ComponentState newState) {
        logTransition(stateName(state_), newState);
        state_ = newState;
        showProgress_ = 0.0f;
        hideProgress_ = 0.0f;
    }

    void logTransition(const char* from, ComponentState to) {
        std::cout << "[UIComponent:" << id_ << "] " 
                  << from << " → " << stateName(to) << std::endl;
    }
};

// ========== 具体组件接口 ==========

// 消息对话框组件
class IMessageBoxComponent : public UIComponent {
public:
    using UIComponent::UIComponent;

    virtual void configure(
        const std::string& title,
        const std::string& message,
        DialogButtons buttons,
        DialogIcon icon,
        DialogCallback callback) = 0;
};

// 右键菜单组件
class IContextMenuComponent : public UIComponent {
public:
    using UIComponent::UIComponent;

    virtual void configure(
        const std::vector<MenuItem>& items,
        Vec2 position,
        MenuCallback callback) = 0;
};

// 进度条组件
class IProgressComponent : public UIComponent {
public:
    using UIComponent::UIComponent;

    virtual void configure(const std::string& title) = 0;
    virtual void setProgress(float value) = 0; // 0.0 ~ 1.0
    virtual float progress() const = 0;
};

// Toast 提示组件
class IToastComponent : public UIComponent {
public:
    using UIComponent::UIComponent;

    virtual void configure(const std::string& message,
                           float duration) = 0;
};

} // namespace krkr2::ui
```

组件生命周期状态机：

```
                show()
Created ──────────────→ Showing ─── (动画完成) ──→ Visible
                                                     │
                                                     │ hide()
                                                     ▼
Destroyed ←── destroy() ── Hidden ←── (动画完成) ── Hiding
     ▲                       │
     │                       │ show()
     │                       ▼
      └──── destroy() ──── Showing → ...
```

---

## 五、示例 4：UIManager 统一管理器

前面定义了 `IUIBackend`（后端接口）和 `UIComponent`（组件基类），但调用方不应直接操作这些底层接口。`UIManager` 是面向上层业务的统一入口，负责：

1. **后端工厂注册**——允许注册多个 UI 后端的创建函数，运行时按优先级选择最佳后端
2. **组件池管理**——复用已创建的组件实例，避免频繁创建/销毁带来的性能开销
3. **便捷方法**——提供 `showMessageBox()`、`showContextMenu()` 等一行调用的高层 API

```cpp
// krkr2_ui_manager.h — UI 统一管理器
#pragma once

#include "krkr2_ui_backend.h"
#include "krkr2_ui_component.h"
#include "krkr2_render_bridge.h"

#include <unordered_map>
#include <functional>
#include <memory>
#include <string>
#include <vector>
#include <algorithm>
#include <iostream>
#include <cassert>

namespace krkr2::ui {

// 后端工厂函数类型——调用时创建一个 IUIBackend 实例
using BackendFactory = std::function<std::unique_ptr<IUIBackend>()>;

// 后端注册信息
struct BackendInfo {
    std::string    name;       // 后端名称（如 "Cocos2d-x"）
    int32_t        priority;   // 优先级（数字越大越优先）
    BackendFactory factory;    // 创建函数
    UICapability   caps;       // 该后端支持的能力
};

// ========== UIManager ==========

class UIManager {
public:
    UIManager() = default;
    ~UIManager() { shutdown(); }

    // 禁止拷贝
    UIManager(const UIManager&) = delete;
    UIManager& operator=(const UIManager&) = delete;

    // ---- 后端注册 ----

    // 注册一个 UI 后端工厂
    // priority: 数字越大越优先选择
    // caps: 该后端支持的能力集合
    void registerBackend(const std::string& name,
                         int32_t priority,
                         UICapability caps,
                         BackendFactory factory) {
        backends_.push_back({name, priority, factory, caps});
        // 按优先级降序排列
        std::sort(backends_.begin(), backends_.end(),
                  [](const BackendInfo& a, const BackendInfo& b) {
                      return a.priority > b.priority;
                  });
        std::cout << "[UIManager] 注册后端: " << name
                  << " (优先级=" << priority << ")" << std::endl;
    }

    // ---- 初始化与关闭 ----

    // 初始化——选择最高优先级的后端并创建实例
    bool initialize(void* windowHandle, int32_t w, int32_t h) {
        if (backends_.empty()) {
            std::cerr << "[UIManager] 错误: 没有注册任何后端"
                      << std::endl;
            return false;
        }
        // 按优先级逐个尝试
        for (auto& info : backends_) {
            auto backend = info.factory();
            if (backend && backend->initialize(windowHandle, w, h)) {
                activeBackend_ = std::move(backend);
                activeBackendName_ = info.name;
                std::cout << "[UIManager] 已激活后端: " 
                          << info.name << std::endl;
                return true;
            }
            std::cerr << "[UIManager] 后端 " << info.name
                      << " 初始化失败，尝试下一个" << std::endl;
        }
        std::cerr << "[UIManager] 错误: 所有后端初始化失败"
                  << std::endl;
        return false;
    }

    // 关闭——销毁所有组件并释放后端
    void shutdown() {
        // 先销毁所有活跃组件
        for (auto& [id, comp] : components_) {
            if (comp->state() != ComponentState::Destroyed) {
                comp->destroy();
            }
        }
        components_.clear();
        
        if (activeBackend_) {
            activeBackend_->shutdown();
            activeBackend_.reset();
            std::cout << "[UIManager] 已关闭后端: "
                      << activeBackendName_ << std::endl;
        }
    }

    // 每帧更新
    void update(float dt) {
        if (activeBackend_) {
            activeBackend_->update(dt);
        }
        // 更新所有活跃组件的动画
        for (auto& [id, comp] : components_) {
            comp->update(dt);
        }
        // 清理已销毁的组件
        cleanupDestroyed();
    }

    // ---- 能力查询 ----

    bool hasCapability(UICapability cap) const {
        return activeBackend_ && 
               (activeBackend_->capabilities() & cap);
    }

    const char* backendName() const {
        return activeBackend_ ? 
               activeBackend_->backendName() : "none";
    }

    // ---- 便捷方法（面向业务层的高层 API）----

    // 显示消息对话框（一行调用）
    void showMessageBox(const std::string& title,
                        const std::string& message,
                        DialogButtons buttons,
                        DialogIcon icon,
                        DialogCallback callback) {
        if (!activeBackend_) {
            std::cerr << "[UIManager] 后端未初始化" << std::endl;
            if (callback) callback(DialogResult::Closed);
            return;
        }
        if (!hasCapability(UICapability::MessageBox)) {
            std::cerr << "[UIManager] 当前后端不支持 MessageBox"
                      << std::endl;
            if (callback) callback(DialogResult::Closed);
            return;
        }
        activeBackend_->showMessageBox(
            title, message, buttons, icon, callback);
    }

    // 显示右键菜单
    void showContextMenu(const std::vector<MenuItem>& items,
                         Vec2 position,
                         MenuCallback callback) {
        if (!activeBackend_) return;
        if (!hasCapability(UICapability::ContextMenu)) {
            std::cerr << "[UIManager] 当前后端不支持 ContextMenu"
                      << std::endl;
            return;
        }
        activeBackend_->showContextMenu(items, position, callback);
    }

    // 显示 Toast 提示
    void showToast(const std::string& message, 
                   float duration = 2.0f) {
        if (!activeBackend_) return;
        if (hasCapability(UICapability::Toast)) {
            activeBackend_->showToast(message, duration);
        } else {
            // 降级：输出到控制台
            std::cout << "[Toast] " << message << std::endl;
        }
    }

    // 获取渲染桥接
    IRenderBridge* renderBridge() {
        return activeBackend_ ? 
               activeBackend_->renderBridge() : nullptr;
    }

    // ---- 组件管理 ----

    // 获取已注册的后端列表
    const std::vector<BackendInfo>& registeredBackends() const {
        return backends_;
    }

    // 获取当前活跃后端
    IUIBackend* activeBackend() {
        return activeBackend_.get();
    }

private:
    std::vector<BackendInfo>    backends_;        // 已注册的后端
    std::unique_ptr<IUIBackend> activeBackend_;   // 当前活跃后端
    std::string                 activeBackendName_;
    
    // 组件池：按 ID 管理所有活跃组件
    std::unordered_map<std::string, 
                       std::unique_ptr<UIComponent>> components_;

    // 清理已销毁的组件
    void cleanupDestroyed() {
        for (auto it = components_.begin(); 
             it != components_.end(); ) {
            if (it->second->state() == ComponentState::Destroyed) {
                it = components_.erase(it);
            } else {
                ++it;
            }
        }
    }
};

} // namespace krkr2::ui
```

### 5.1 UIManager 的设计要点

UIManager 采用了以下关键设计模式：

| 设计模式 | 在 UIManager 中的应用 | 为什么选择 |
|----------|----------------------|-----------|
| 工厂模式（Factory Pattern） | `BackendFactory` 函数创建后端实例 | 解耦后端创建和使用，支持延迟创建 |
| 策略模式（Strategy Pattern） | 运行时选择不同 UI 后端 | 同一接口多种实现，可热切换 |
| 优雅降级（Graceful Degradation） | 能力查询 + 后备方案 | 后端能力不同时自动降级 |
| 对象池（Object Pool） | `components_` 管理组件生命周期 | 避免频繁分配/释放内存 |

特别注意 `showToast()` 方法中的降级逻辑——当后端不支持 Toast 时，自动降级为控制台输出。这是实际项目中常见的做法：移动端 Flutter 后端有原生 Toast，而 Dear ImGui 后端可能没有，但业务代码不应因此崩溃。

---

## 六、示例 5：后端注册与运行时切换机制

实际项目中，UI 后端的选择不仅发生在启动时——有时需要在运行时切换。例如开发者可能在 Cocos2d-x 后端和 Flutter 后端之间切换来对比效果，或者当 Flutter 引擎加载失败时回退到 Cocos2d-x。

以下代码展示了完整的后端注册和热切换流程：

```cpp
// krkr2_backend_registry.h — 后端注册与热切换
#pragma once

#include "krkr2_ui_manager.h"
#include <iostream>
#include <chrono>

namespace krkr2::ui {

// ========== 模拟 Cocos2d-x 后端 ==========
// 实际实现在下一节(02-Cocos2d实现与Flutter原型.md)详细展开

class Cocos2dUIBackend : public IUIBackend {
public:
    bool initialize(void* wh, int32_t w, int32_t h) override {
        std::cout << "[Cocos2d] 初始化 " << w << "x" << h 
                  << std::endl;
        width_ = w; height_ = h;
        return true; // Cocos2d-x 总是可用的（当前引擎）
    }
    
    void shutdown() override {
        std::cout << "[Cocos2d] 关闭" << std::endl;
    }
    
    void update(float dt) override {
        // Cocos2d-x 的 UI 更新由引擎主循环驱动
    }
    
    UICapability capabilities() const override {
        // Cocos2d-x 后端支持基础 UI 但不支持文件对话框
        return UICapability::MessageBox 
             | UICapability::ContextMenu
             | UICapability::Settings
             | UICapability::Progress
             | UICapability::SoftKeyboard;
    }
    
    const char* backendName() const override { 
        return "Cocos2d-x"; 
    }
    uint32_t interfaceVersion() const override { return 1; }
    
    void showMessageBox(const std::string& title,
                        const std::string& message,
                        DialogButtons buttons,
                        DialogIcon icon,
                        DialogCallback cb) override {
        std::cout << "[Cocos2d-MessageBox] " << title 
                  << ": " << message << std::endl;
        // 模拟用户点击"确定"
        if (cb) cb(DialogResult::OK);
    }
    
    void showFileDialog(const FileDialogOptions& opts,
                        FileCallback cb) override {
        // Cocos2d-x 不支持文件对话框
        std::cerr << "[Cocos2d] 不支持文件对话框" << std::endl;
        if (cb) cb({});
    }
    
    void showContextMenu(const std::vector<MenuItem>& items,
                         Vec2 pos, MenuCallback cb) override {
        std::cout << "[Cocos2d-Menu] 显示菜单，" 
                  << items.size() << " 项" << std::endl;
    }
    
    void setMenuBar(const std::vector<MenuItem>& items,
                    MenuCallback cb) override {
        // 移动平台无菜单栏
    }
    
    void showSettings() override {
        std::cout << "[Cocos2d] 显示设置面板" << std::endl;
    }
    
    void showToast(const std::string& msg, float dur) override {
        std::cout << "[Cocos2d-Toast] " << msg << std::endl;
    }
    
    void showProgress(const std::string& t, float p) override {
        std::cout << "[Cocos2d-Progress] " << t << ": " 
                  << (int)(p * 100) << "%" << std::endl;
    }
    void hideProgress() override {}
    
    void showSoftKeyboard(TextInputCallback cb) override {
        std::cout << "[Cocos2d] 显示软键盘" << std::endl;
    }
    void hideSoftKeyboard() override {}
    
    void showFontSelector(const FontDesc& cur,
        std::function<void(const FontDesc&)> cb) override {
        std::cerr << "[Cocos2d] 不支持字体选择器" << std::endl;
    }
    
    IRenderBridge* renderBridge() override { 
        return nullptr; // 下一节实现
    }

private:
    int32_t width_ = 0, height_ = 0;
};

// ========== 模拟 Flutter 后端 ==========

class FlutterUIBackend : public IUIBackend {
public:
    bool initialize(void* wh, int32_t w, int32_t h) override {
        std::cout << "[Flutter] 初始化 FlutterEngine " 
                  << w << "x" << h << std::endl;
        // 实际项目中这里会调用 FlutterEngineRun()
        return true;
    }
    
    void shutdown() override {
        std::cout << "[Flutter] 关闭 FlutterEngine" << std::endl;
    }
    
    void update(float dt) override {
        // Flutter 有自己的事件循环，这里触发 vsync
    }
    
    UICapability capabilities() const override {
        // Flutter 后端支持几乎所有能力
        return UICapability::MessageBox 
             | UICapability::FileDialog
             | UICapability::FontSelector
             | UICapability::SoftKeyboard
             | UICapability::ContextMenu
             | UICapability::Settings
             | UICapability::Toast
             | UICapability::Progress
             | UICapability::TextureSharing
             | UICapability::NativeAnimation
             | UICapability::DarkMode;
    }
    
    const char* backendName() const override { 
        return "Flutter"; 
    }
    uint32_t interfaceVersion() const override { return 1; }
    
    void showMessageBox(const std::string& title,
                        const std::string& message,
                        DialogButtons buttons,
                        DialogIcon icon,
                        DialogCallback cb) override {
        std::cout << "[Flutter-Dialog] " << title 
                  << ": " << message << std::endl;
        if (cb) cb(DialogResult::OK);
    }
    
    void showFileDialog(const FileDialogOptions& opts,
                        FileCallback cb) override {
        std::cout << "[Flutter-FilePicker] " << opts.title
                  << std::endl;
        if (cb) cb({"/path/to/selected/file.tjs"});
    }
    
    void showContextMenu(const std::vector<MenuItem>& items,
                         Vec2 pos, MenuCallback cb) override {
        std::cout << "[Flutter-Menu] 显示菜单 @(" 
                  << pos.x << "," << pos.y << "), " 
                  << items.size() << " 项" << std::endl;
    }
    
    void setMenuBar(const std::vector<MenuItem>& items,
                    MenuCallback cb) override {
        std::cout << "[Flutter-MenuBar] 设置菜单栏" << std::endl;
    }
    
    void showSettings() override {
        std::cout << "[Flutter] 显示 Material Design 设置面板"
                  << std::endl;
    }
    
    void showToast(const std::string& msg, float dur) override {
        std::cout << "[Flutter-Toast] " << msg 
                  << " (" << dur << "s)" << std::endl;
    }
    
    void showProgress(const std::string& t, float p) override {
        std::cout << "[Flutter-Progress] " << t 
                  << ": " << (int)(p * 100) << "%" << std::endl;
    }
    void hideProgress() override {}
    
    void showSoftKeyboard(TextInputCallback cb) override {
        std::cout << "[Flutter] 显示平台键盘" << std::endl;
    }
    void hideSoftKeyboard() override {}
    
    void showFontSelector(const FontDesc& cur,
        std::function<void(const FontDesc&)> cb) override {
        std::cout << "[Flutter-FontPicker] 当前: " 
                  << cur.family << std::endl;
        if (cb) cb({"Noto Sans CJK", 18.0f, false, false});
    }
    
    IRenderBridge* renderBridge() override {
        return nullptr; // 下一节实现纹理桥接
    }
};

} // namespace krkr2::ui
```

### 6.1 后端注册与自动选择

注册后端时，`UIManager` 根据优先级自动选择最佳可用后端：

```cpp
// 注册示例——通常在应用启动早期执行
void registerAllBackends(krkr2::ui::UIManager& mgr) {
    using namespace krkr2::ui;
    
    // Cocos2d-x 后端——优先级 10（保底方案，总是可用）
    mgr.registerBackend(
        "Cocos2d-x", 
        10,  // 低优先级
        UICapability::MessageBox | UICapability::ContextMenu
            | UICapability::Settings | UICapability::Progress
            | UICapability::SoftKeyboard,
        []() { return std::make_unique<Cocos2dUIBackend>(); }
    );
    
    // Flutter 后端——优先级 50（首选方案）
    mgr.registerBackend(
        "Flutter",
        50,  // 高优先级
        UICapability::MessageBox | UICapability::FileDialog
            | UICapability::FontSelector 
            | UICapability::SoftKeyboard
            | UICapability::ContextMenu 
            | UICapability::Settings
            | UICapability::Toast | UICapability::Progress
            | UICapability::TextureSharing
            | UICapability::NativeAnimation
            | UICapability::DarkMode,
        []() { return std::make_unique<FlutterUIBackend>(); }
    );
}
```

初始化时 `UIManager::initialize()` 会先尝试 Flutter（优先级 50），若失败则回退到 Cocos2d-x（优先级 10）。这种分层回退策略（Tiered Fallback Strategy）确保了应用在任何环境下都能正常运行。

---

## 七、示例 6：完整集成——从启动到游戏循环

以下是将 UIManager 集成到 KrKr2 游戏主循环的完整示例。这是一个自包含的可编译程序：

```cpp
// main_ui_demo.cpp — KrKr2 UI 抽象层完整集成演示
// 编译: g++ -std=c++17 -o ui_demo main_ui_demo.cpp
#include <iostream>
#include <string>
#include <memory>
#include <vector>
#include <functional>
#include <unordered_map>
#include <algorithm>
#include <thread>
#include <chrono>
#include <cassert>
#include <cstdint>
#include <optional>

// ---- 内联所有头文件的定义（实际项目中分文件） ----
// 以下类型定义参见前面的示例 1-5
// 这里为了自包含，重复关键定义

namespace krkr2::ui {

struct Color {
    uint8_t r = 0, g = 0, b = 0, a = 255;
};

struct Vec2 { float x = 0, y = 0; };
struct UIRect { float x = 0, y = 0, w = 0, h = 0; };
struct FontDesc {
    std::string family = "sans-serif";
    float size = 16.0f;
    bool bold = false, italic = false;
};

enum class UICapability : uint32_t {
    MessageBox = 1 << 0, ContextMenu = 1 << 4,
    Settings = 1 << 5,   Toast = 1 << 6,
    Progress = 1 << 7,   TextureSharing = 1 << 8
};
inline UICapability operator|(UICapability a, UICapability b) {
    return static_cast<UICapability>(
        static_cast<uint32_t>(a) | static_cast<uint32_t>(b));
}
inline bool operator&(UICapability a, UICapability b) {
    return (static_cast<uint32_t>(a) 
          & static_cast<uint32_t>(b)) != 0;
}

enum class DialogResult { OK, Cancel, Yes, No, Closed };
enum class DialogButtons { OK, OKCancel, YesNo, YesNoCancel };
enum class DialogIcon { None, Info, Warning, Error, Question };

using DialogCallback = std::function<void(DialogResult)>;
using MenuCallback = std::function<void(const std::string&)>;
struct MenuItem {
    std::string id, label;
    bool separator = false;
    std::vector<MenuItem> children;
};

// 简化版 IUIBackend（完整版参见示例 1）
class IUIBackend {
public:
    virtual ~IUIBackend() = default;
    virtual bool initialize(void* wh, int32_t w, int32_t h) = 0;
    virtual void shutdown() = 0;
    virtual void update(float dt) = 0;
    virtual UICapability capabilities() const = 0;
    virtual const char* backendName() const = 0;
    virtual void showMessageBox(const std::string& title,
        const std::string& msg, DialogButtons btn,
        DialogIcon icon, DialogCallback cb) = 0;
    virtual void showContextMenu(
        const std::vector<MenuItem>& items,
        Vec2 pos, MenuCallback cb) = 0;
    virtual void showToast(const std::string& msg, 
                           float dur) = 0;
};

// Cocos2d-x 后端实现
class Cocos2dBackend : public IUIBackend {
public:
    bool initialize(void* wh, int32_t w, int32_t h) override {
        std::cout << "  [Cocos2d] 初始化完成 (" 
                  << w << "x" << h << ")" << std::endl;
        return true;
    }
    void shutdown() override {
        std::cout << "  [Cocos2d] 已关闭" << std::endl;
    }
    void update(float dt) override {}
    UICapability capabilities() const override {
        return UICapability::MessageBox 
             | UICapability::ContextMenu;
    }
    const char* backendName() const override { 
        return "Cocos2d-x"; 
    }
    void showMessageBox(const std::string& title,
        const std::string& msg, DialogButtons btn,
        DialogIcon icon, DialogCallback cb) override {
        std::cout << "  [Cocos2d-MsgBox] " << title 
                  << ": " << msg << std::endl;
        if (cb) cb(DialogResult::OK);
    }
    void showContextMenu(const std::vector<MenuItem>& items,
        Vec2 pos, MenuCallback cb) override {
        std::cout << "  [Cocos2d-Menu] " << items.size() 
                  << " 项 @(" << pos.x << "," << pos.y 
                  << ")" << std::endl;
        // 模拟用户选择第一项
        if (cb && !items.empty()) cb(items[0].id);
    }
    void showToast(const std::string& msg, float) override {
         std::cout << "  [Cocos2d] (无原生Toast) " 
                  << msg << std::endl;
    }
};

} // namespace krkr2::ui
```

接下来是 `main()` 函数——演示完整的启动、注册、初始化、游戏循环和关闭流程：

```cpp
// main() — 完整集成演示
int main() {
    using namespace krkr2::ui;
    
    std::cout << "===== KrKr2 UI 抽象层集成演示 =====" 
              << std::endl;
    
    // 第一步：创建管理器
    // UIManager 是全局单例，管理所有 UI 后端和组件
    std::unique_ptr<IUIBackend> backend;
    
    // 第二步：尝试创建后端（模拟优先级选择）
    std::cout << "\n[1] 尝试创建 Flutter 后端..." << std::endl;
    // 实际项目中检查 Flutter 引擎是否可用
    bool flutterAvailable = false; // 模拟 Flutter 不可用
    
    if (flutterAvailable) {
        // backend = std::make_unique<FlutterBackend>();
        std::cout << "  Flutter 引擎已加载" << std::endl;
    } else {
        std::cout << "  Flutter 不可用，回退到 Cocos2d-x" 
                  << std::endl;
        backend = std::make_unique<Cocos2dBackend>();
    }
    
    // 第三步：初始化后端
    std::cout << "\n[2] 初始化后端..." << std::endl;
    void* fakeWindow = reinterpret_cast<void*>(0x12345678);
    if (!backend->initialize(fakeWindow, 1280, 720)) {
        std::cerr << "后端初始化失败!" << std::endl;
        return 1;
    }
    std::cout << "  当前后端: " << backend->backendName()
              << std::endl;
    
    // 第四步：查询能力
    std::cout << "\n[3] 能力查询..." << std::endl;
    auto caps = backend->capabilities();
    std::cout << "  MessageBox: " 
              << (caps & UICapability::MessageBox ? "Y" : "N")
              << std::endl;
    std::cout << "  ContextMenu: " 
              << (caps & UICapability::ContextMenu ? "Y" : "N")
              << std::endl;
    std::cout << "  TextureSharing: " 
              << (caps & UICapability::TextureSharing ? "Y" : "N")
              << std::endl;
    
    // 第五步：模拟游戏循环中使用 UI
    std::cout << "\n[4] 模拟游戏循环..." << std::endl;
    
    // 5a: 游戏启动时显示欢迎消息
    backend->showMessageBox(
        "KrKr2", 
        "欢迎使用 KiriKiri2 模拟器",
        DialogButtons::OK,
        DialogIcon::Info,
        [](DialogResult r) {
            std::cout << "  用户响应: " 
                      << (r == DialogResult::OK ? "确定" : "其他")
                      << std::endl;
        });
    
    // 5b: 模拟右键菜单
    std::vector<MenuItem> menu = {
        {"save",   "保存存档"},
        {"load",   "读取存档"},
        {"config", "游戏设置"},
        {"quit",   "退出游戏"}
    };
    
    backend->showContextMenu(
        menu, {400.0f, 300.0f},
        [](const std::string& id) {
            std::cout << "  菜单选择: " << id << std::endl;
        });
    
    // 5c: 模拟 3 帧游戏循环
    for (int frame = 0; frame < 3; ++frame) {
        float dt = 1.0f / 60.0f; // 60 FPS
        backend->update(dt);
    }
    
    // 5d: Toast 提示
    backend->showToast("已自动保存", 2.0f);
    
    // 第六步：关闭
    std::cout << "\n[5] 关闭..." << std::endl;
    backend->shutdown();
    backend.reset();
    
    std::cout << "\n===== 演示结束 =====" << std::endl;
    return 0;
}
```

**预期输出：**

```
===== KrKr2 UI 抽象层集成演示 =====

[1] 尝试创建 Flutter 后端...
  Flutter 不可用，回退到 Cocos2d-x

[2] 初始化后端...
  [Cocos2d] 初始化完成 (1280x720)
  当前后端: Cocos2d-x

[3] 能力查询...
  MessageBox: Y
  ContextMenu: Y
  TextureSharing: N

[4] 模拟游戏循环...
  [Cocos2d-MsgBox] KrKr2: 欢迎使用 KiriKiri2 模拟器
  用户响应: 确定
  [Cocos2d-Menu] 4 项 @(400,300)
  菜单选择: save
  [Cocos2d] (无原生Toast) 已自动保存

[5] 关闭...
  [Cocos2d] 已关闭

===== 演示结束 =====
```

---

## 八、常见错误与排查

### 错误 1：回调中访问已销毁的对象

**症状**：对话框回调触发时程序崩溃（段错误/野指针）

**原因**：回调捕获了 `this` 指针，但回调触发时对象已被销毁

```cpp
// ❌ 错误写法——this 可能在回调触发前已被销毁
class GameScene {
    void showExitDialog() {
        uiManager->showMessageBox(
            "退出", "确认退出?",
            DialogButtons::YesNo, DialogIcon::Question,
            [this](DialogResult r) {  // 危险: 捕获 this
                if (r == DialogResult::Yes) {
                    this->exitGame(); // this 可能已无效!
                }
            });
        // 如果 GameScene 在对话框显示期间被销毁...
    }
};

// ✅ 正确写法——使用 weak_ptr 防止悬挂引用
class GameScene : public std::enable_shared_from_this<GameScene> {
    void showExitDialog() {
        std::weak_ptr<GameScene> weakSelf = shared_from_this();
        uiManager->showMessageBox(
            "退出", "确认退出?",
            DialogButtons::YesNo, DialogIcon::Question,
            [weakSelf](DialogResult r) {
                if (auto self = weakSelf.lock()) {
                    // self 仍然有效
                    if (r == DialogResult::Yes) {
                        self->exitGame();
                    }
                }
                // else: GameScene 已销毁，安全忽略
            });
    }
};
```

**排查方法**：在 `UIComponent` 析构函数中检查是否有未完成的回调；使用 AddressSanitizer（ASan）检测 use-after-free。

### 错误 2：在非主线程调用 UI 方法

**症状**：Android 上崩溃，Windows 上界面冻结，macOS 上断言失败

**原因**：所有 UI 操作必须在主线程（UI 线程）执行。Flutter 的 platform channel、Android 的 View 系统、macOS 的 AppKit 都有线程亲和性（Thread Affinity，即某些操作只能在特定线程执行的限制）

```cpp
// ❌ 错误——在工作线程中直接调用 UI
void loadGameData() {
    std::thread worker([&]() {
        // ... 加载数据 ...
        uiManager->showToast("加载完成"); // 崩溃!
    });
    worker.detach();
}

// ✅ 正确——通过消息队列投递到主线程
void loadGameData() {
    std::thread worker([&]() {
        // ... 加载数据 ...
        // 投递到主线程执行（假设有 runOnMainThread 工具）
        runOnMainThread([&]() {
            uiManager->showToast("加载完成"); // 安全
        });
    });
    worker.detach();
}
```

**各平台主线程投递方式：**

| 平台 | 主线程投递 API |
|------|--------------|
| Windows | `PostMessage(hwnd, WM_USER, ...)` 或 `SendMessage` |
| macOS | `dispatch_async(dispatch_get_main_queue(), ^{ ... })` |
| Android | `Activity.runOnUiThread(Runnable)` 或 `Handler(Looper.getMainLooper())` |
| Linux (X11) | `XSendEvent` 或 GLib `g_idle_add()` |

### 错误 3：忘记查询能力就调用后端方法

**症状**：调用 `showFileDialog()` 时，Cocos2d-x 后端返回空结果或直接忽略

**原因**：不是所有后端都支持所有功能。直接调用不支持的功能不会崩溃（接口要求安全返回），但用户体验很差——用户点了按钮没有任何反应

```cpp
// ❌ 错误——假设后端支持所有功能
void onOpenFile() {
    uiManager->activeBackend()->showFileDialog(
        opts, callback); // Cocos2d 后端: 静默失败
}

// ✅ 正确——先查询能力，不支持时提供替代方案
void onOpenFile() {
    if (uiManager->hasCapability(UICapability::FileDialog)) {
        uiManager->activeBackend()->showFileDialog(
            opts, callback);
    } else {
        // 降级方案：使用系统原生文件对话框
        // 或显示手动输入路径的文本框
        uiManager->showMessageBox(
            "功能不可用",
            "当前 UI 后端不支持文件选择器。\n"
            "请手动输入文件路径，或切换到 Flutter 后端。",
            DialogButtons::OK, DialogIcon::Info, nullptr);
    }
}
```

---

## 动手实践

### 练习 1：为 IUIBackend 添加"关于"对话框接口

目标：扩展 `IUIBackend` 接口，新增一个 `showAboutDialog()` 方法，显示引擎版本、构建信息和致谢列表。

**步骤：**

1. 在 `UICapability` 中添加 `AboutDialog = 1 << 12`
2. 在 `IUIBackend` 中添加纯虚方法：

```cpp
// 关于信息结构体
struct AboutInfo {
    std::string appName;       // "KrKr2 模拟器"
    std::string version;       // "2.0.0-beta"
    std::string buildDate;     // __DATE__
    std::string compiler;      // 编译器版本
    std::vector<std::string> credits; // 致谢列表
};

// 在 IUIBackend 中新增
virtual void showAboutDialog(const AboutInfo& info) = 0;
```

3. 在 `Cocos2dUIBackend` 中实现——用 `TVPMessageBoxForm` 拼接文本显示
4. 在 `FlutterUIBackend` 中实现——通过 Platform Channel 调用 Flutter 的 `showAboutDialog()`

### 练习 2：实现接口版本兼容性检查

目标：当 UIManager 加载一个后端时，检查后端的 `interfaceVersion()` 是否与管理器兼容。

**规则：**
- 管理器版本 `N` 兼容后端版本 `N` 和 `N-1`（向后兼容一个版本）
- 管理器版本 `N` 不兼容后端版本 `N-2` 或更早
- 后端版本比管理器新时（`backend > manager`），拒绝加载

```cpp
// 在 UIManager 中添加版本检查
static constexpr uint32_t kManagerVersion = 2;

bool isCompatible(uint32_t backendVersion) const {
    if (backendVersion > kManagerVersion) {
        // 后端太新，管理器不认识新接口
        return false;
    }
    if (backendVersion < kManagerVersion - 1) {
        // 后端太旧，可能缺少必要方法
        return false;
    }
    return true; // 版本 N 或 N-1，兼容
}
```

验证：写一个测试程序，注册版本 1、2、3 的后端，确认管理器版本 2 只接受版本 1 和 2。

---

## 对照项目源码

KrKr2 项目中现有的 UI 实现分散在多个文件中。以下是本节接口设计的对照参考：

| 本节接口 | 项目现有实现 | 文件位置 |
|----------|------------|----------|
| `IUIBackend::showMessageBox()` | `TVPMessageBoxForm` | `cpp/core/environ/ui/MessageBox.cpp` — 使用 Cocos2d-x 的 `CSLoader` 加载 `.csb` 布局文件，设置按钮回调 |
| `IUIBackend::showContextMenu()` | TJS2 脚本控制 | `cpp/core/tjs2/` — KrKr2 原版的右键菜单由 TJS2 脚本 `menu.ks` 控制，不是原生 UI |
| `IUIBackend::showSoftKeyboard()` | `TVPPadForm` | `cpp/core/environ/ui/PadForm.cpp` — Android 专用虚拟键盘，使用 Cocos2d-x Scene 实现 |
| `IUIBackend::showFontSelector()` | `TVPFontSelectForm` | `cpp/core/environ/ui/FontSelect.cpp` — 列出系统字体供用户选择 |
| `UIComponent` 生命周期 | `cocos2d::Scene` 管理 | `cpp/core/environ/cocos2d/AppDelegate.cpp` — 当前 UI 的显示/隐藏通过 Cocos2d Scene 栈管理 |

**关键差异**：

1. **当前实现没有统一接口**——每个 UI 组件直接依赖 Cocos2d-x API，没有抽象层。要替换 UI 框架需要改动所有 UI 代码
2. **当前实现混合了业务逻辑和 UI 逻辑**——`TVPMessageBoxForm` 既负责渲染按钮，又处理用户输入，还管理显示动画。本节设计将这些职责分离到 `UIComponent`（生命周期）和 `IUIBackend`（渲染）
3. **当前没有能力查询**——所有 UI 功能假设 Cocos2d-x 总是可用。引入 `UICapability` 后，可以在 Flutter 后端添加新功能而不影响 Cocos2d-x 后端

---

## 本节小结

1. **KrKr2 需要 9 类 UI 组件**：消息对话框、主窗口、软键盘、字体选择器、文件选择器、设置面板、进度条、右键菜单、Toast 提示——覆盖视觉小说引擎的全部 UI 场景
2. **IUIBackend 是顶层接口**：定义了 UI 后端必须实现的所有方法，使用异步回调模式避免阻塞游戏主循环
3. **IRenderBridge 连接游戏和 UI**：通过共享纹理和 RAII 锁机制，将游戏画面安全传递给 UI 框架显示
4. **UIComponent 管理生命周期**：6 个状态（Created→Showing→Visible→Hiding→Hidden→Destroyed）加动画过渡，使组件行为可预测
5. **UIManager 是业务层入口**：封装了后端注册、优先级选择、能力查询、优雅降级等复杂逻辑，业务代码只需一行调用
6. **后端按优先级自动选择**：Flutter 优先级 50 > Cocos2d-x 优先级 10，初始化失败时自动回退
7. **能力查询避免运行时错误**：调用前检查 `UICapability`，不支持的功能提供降级方案而非静默失败
8. **回调安全是第一要务**：使用 `weak_ptr` 防止悬挂引用，严格在主线程调用 UI 方法
9. **接口设计与现有代码的差异**：当前 KrKr2 没有 UI 抽象层，每个组件直接耦合 Cocos2d-x——本节设计为渐进式迁移奠定基础

---

## 练习题与答案

### 题目 1：设计题——为 IUIBackend 添加主题切换支持

KrKr2 需要支持浅色/深色主题切换。请设计接口方案：

1. 定义 `ThemeMode` 枚举（Light、Dark、System）
2. 在 `IUIBackend` 中添加 `setTheme()` 和 `currentTheme()` 方法
3. 添加主题变更回调通知

<details>
<summary>查看答案</summary>

```cpp
// 主题模式
enum class ThemeMode {
    Light,    // 浅色主题
    Dark,     // 深色主题
    System    // 跟随系统设置
};

// 主题颜色方案
struct ThemeColors {
    Color background;    // 背景色
    Color foreground;    // 前景色（文字）
    Color primary;       // 主色调
    Color secondary;     // 次要色
    Color error;         // 错误色
    Color surface;       // 表面色（卡片/对话框背景）
};

// 主题变更回调
using ThemeChangeCallback = 
    std::function<void(ThemeMode, const ThemeColors&)>;

// 在 IUIBackend 中新增：
class IUIBackend {
public:
    // ... 已有方法 ...

    // 设置主题模式
    virtual void setTheme(ThemeMode mode) = 0;
    
    // 获取当前实际主题（System 会解析为 Light 或 Dark）
    virtual ThemeMode currentTheme() const = 0;
    
    // 获取当前主题颜色
    virtual ThemeColors themeColors() const = 0;
    
    // 注册主题变更回调（系统切换深色模式时触发）
    virtual void onThemeChanged(ThemeChangeCallback cb) = 0;
};

// 在 UICapability 中添加：
// DarkMode = 1 << 11  // 已存在
```

**使用示例：**

```cpp
// 初始化后设置主题
backend->setTheme(ThemeMode::System); // 跟随系统

// 监听主题变更
backend->onThemeChanged(
    [](ThemeMode mode, const ThemeColors& colors) {
        std::cout << "主题切换: " 
                  << (mode == ThemeMode::Dark ? "深色" : "浅色")
                  << std::endl;
        // 更新游戏内 UI 颜色...
    });
```

</details>

### 题目 2：代码题——实现带超时的对话框

某些场景（如自动化测试、无人值守模式）需要对话框超时自动关闭。请实现一个 `showTimedMessageBox()` 方法，在 `timeoutSec` 秒后自动返回 `DialogResult::Closed`。

<details>
<summary>查看答案</summary>

```cpp
#include <iostream>
#include <functional>
#include <string>
#include <chrono>
#include <thread>
#include <atomic>
#include <memory>

// 简化的对话框结果和类型
enum class DialogResult { OK, Cancel, Closed };
enum class DialogButtons { OK, OKCancel };
using DialogCallback = std::function<void(DialogResult)>;

// 带超时的消息对话框
class TimedMessageBox {
public:
    // timeoutSec: 超时秒数（0 表示不超时）
    // callback: 用户操作或超时后调用
    void show(const std::string& title,
              const std::string& message,
              DialogButtons buttons,
              float timeoutSec,
              DialogCallback callback) {
        responded_ = std::make_shared<std::atomic<bool>>(false);
        callback_ = callback;
        
        // 显示对话框（模拟）
        std::cout << "[TimedDialog] " << title << ": " 
                  << message << " (超时=" << timeoutSec 
                  << "s)" << std::endl;
        
        if (timeoutSec > 0) {
            // 启动超时线程
            auto responded = responded_;
            auto cb = callback;
            std::thread([responded, cb, timeoutSec]() {
                auto deadline = std::chrono::steady_clock::now()
                    + std::chrono::milliseconds(
                        (int)(timeoutSec * 1000));
                
                while (std::chrono::steady_clock::now() 
                       < deadline) {
                    if (responded->load()) return; // 已响应
                    std::this_thread::sleep_for(
                        std::chrono::milliseconds(100));
                }
                
                // 超时——自动关闭
                if (!responded->exchange(true)) {
                    std::cout << "[TimedDialog] 超时自动关闭"
                              << std::endl;
                    if (cb) cb(DialogResult::Closed);
                }
            }).detach();
        }
    }
    
    // 模拟用户点击
    void respond(DialogResult result) {
        if (responded_ && !responded_->exchange(true)) {
            std::cout << "[TimedDialog] 用户响应" << std::endl;
            if (callback_) callback_(result);
        }
    }

private:
    std::shared_ptr<std::atomic<bool>> responded_;
    DialogCallback callback_;
};

int main() {
    TimedMessageBox box;
    
    // 测试 1: 超时自动关闭
    std::cout << "=== 测试超时 ===" << std::endl;
    box.show("确认", "是否保存?", DialogButtons::OKCancel, 
             1.0f,
             [](DialogResult r) {
                 std::cout << "结果: " 
                    << (r == DialogResult::Closed 
                        ? "超时关闭" : "用户操作")
                    << std::endl;
             });
    
    // 等待超时触发
    std::this_thread::sleep_for(std::chrono::seconds(2));
    
    // 测试 2: 用户及时响应
    std::cout << "\n=== 测试用户响应 ===" << std::endl;
    TimedMessageBox box2;
    box2.show("退出", "确认退出?", DialogButtons::OKCancel,
              5.0f,
              [](DialogResult r) {
                  std::cout << "结果: " 
                     << (r == DialogResult::OK ? "确定" 
                         : "其他") << std::endl;
              });
    
    // 模拟用户立即点击
    box2.respond(DialogResult::OK);
    
    std::this_thread::sleep_for(
        std::chrono::milliseconds(200));
    return 0;
}
```

**预期输出：**

```
=== 测试超时 ===
[TimedDialog] 确认: 是否保存? (超时=1s)
[TimedDialog] 超时自动关闭
结果: 超时关闭

=== 测试用户响应 ===
[TimedDialog] 退出: 确认退出? (超时=5s)
[TimedDialog] 用户响应
结果: 确定
```

</details>

### 题目 3：分析题——评估接口设计的线程安全性

阅读本节的 `IUIBackend` 和 `UIManager` 代码，回答：

1. `UIManager::showMessageBox()` 是线程安全的吗？为什么？
2. 如果游戏引擎在工作线程调用 `showToast()`，会发生什么？
3. 提出至少两种改进方案使接口线程安全

<details>
<summary>查看答案</summary>

**1. UIManager::showMessageBox() 不是线程安全的。**

原因分析：
- `activeBackend_` 是裸指针访问（`std::unique_ptr`），没有互斥锁保护
- 如果主线程正在 `shutdown()`（重置 `activeBackend_`），工作线程同时调用 `showMessageBox()` 会导致 use-after-free
- `hasCapability()` 和 `showMessageBox()` 之间存在 TOCTOU（Time-of-Check-Time-of-Use）竞态——检查时后端存在，调用时可能已销毁

**2. 工作线程调用 showToast() 的后果：**

- **Windows**：可能死锁（Win32 消息循环在主线程），或者 UI 状态损坏
- **Android**：直接崩溃——`android.view.ViewRootImpl$CalledFromWrongThreadException`
- **macOS**：AppKit 断言失败 `NSInternalInconsistencyException`
- **Linux**：X11 不是线程安全的，可能导致连接损坏

**3. 改进方案：**

**方案 A：互斥锁 + 主线程断言**

```cpp
class UIManager {
    mutable std::mutex mtx_;
    std::thread::id mainThreadId_;
    
    void showMessageBox(...) {
        // 断言：必须在主线程调用
        assert(std::this_thread::get_id() == mainThreadId_
               && "UI 方法必须在主线程调用");
        std::lock_guard lock(mtx_);
        if (!activeBackend_) return;
        activeBackend_->showMessageBox(...);
    }
};
```

**方案 B：消息队列模式（推荐）**

```cpp
class UIManager {
    // 线程安全的命令队列
    ThreadSafeQueue<std::function<void()>> commandQueue_;
    
    // 任意线程可调用——命令入队
    void showMessageBox(...) {
        commandQueue_.push([=, this]() {
            if (activeBackend_) {
                activeBackend_->showMessageBox(...);
            }
        });
    }
    
    // 主线程每帧调用——执行队列中的命令
    void update(float dt) {
        std::function<void()> cmd;
        while (commandQueue_.tryPop(cmd)) {
            cmd();
        }
        // ... 其余更新逻辑 ...
    }
};
```

方案 B 更优，因为：
- 所有 UI 操作自动在主线程执行
- 调用方不需要关心线程问题
- 性能开销极小（只是入队一个 lambda）
- 这也是 Flutter、Qt、Android 等主流框架的做法

</details>

---

## 下一步

下一节 [Cocos2d 实现与 Flutter 原型](./02-Cocos2d实现与Flutter原型.md) 将基于本节定义的接口，实际实现两个后端：

1. **Cocos2d-x 后端**——包装 KrKr2 现有的 `TVPMessageBoxForm`、`TVPPadForm` 等实现
2. **Flutter 后端原型**——通过 Platform Channel 调用 Dart UI
3. **性能对比**——启动时间、内存、帧率
4. **迁移策略**——如何从 Cocos2d-x 渐进迁移到 Flutter

