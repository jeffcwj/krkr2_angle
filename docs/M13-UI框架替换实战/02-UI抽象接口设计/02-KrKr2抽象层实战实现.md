# 02 — KrKr2 抽象层实战实现

> **所属模块：** M13-UI框架替换实战
> **前置知识：** [01-接口定义与依赖倒置原则](./01-接口定义与依赖倒置原则.md)
> **预计阅读时间：** 70 分钟

## 本节目标

读完本节后，你将能够：

1. 完整实现 `Cocos2dUIManager`——将现有所有 Cocos2d-x UI 表单包装为 `IUIManager` 接口
2. 更新 `AppDelegate` 的 UI 初始化流程，使用新的工厂体系
3. 编写 `IUIManager` 的集成测试，验证对话框生命周期管理正确
4. 识别并处理 Cocos2d-x 适配层的平台差异（Android vs 桌面端的对话框显示差异）

## 术语预览

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 适配器模式 | Adapter Pattern | 将不兼容的接口转换为客户期望的接口，Cocos2dUIManager 就是 Cocos2d-x UI 的适配器 |
| 生命周期同步 | Lifecycle Synchronization | 确保 UI 元素的创建/销毁与 Cocos2d-x 场景切换保持一致，避免悬空指针 |
| 安全弱指针 | Weak Pointer | `std::weak_ptr` 持有对象的非所有权引用，可检测对象是否已被销毁，用于异步回调的安全检查 |
| 对话框栈 | Dialog Stack | 维护当前打开的对话框顺序，支持嵌套打开（如设置页内打开文件选择对话框）和按序关闭 |
| 覆盖层 | Overlay | 位于所有 UI 之上的全屏半透明遮罩，通常用于加载动画或防止用户在操作完成前点击 |

## 一、Cocos2dUIManager 的完整实现

### 1.1 头文件设计

```cpp
// cpp/core/environ/ui/cocos2d/Cocos2dUIManager.h
#pragma once
#include "../abstract/IUIManager.h"
#include "cocos2d.h"
#include "ui/CocosGUI.h"
#include <vector>
#include <memory>

namespace krkr2 {
namespace ui {

class Cocos2dUIManager final : public IUIManager {
public:
    Cocos2dUIManager() = default;
    ~Cocos2dUIManager() override;

    // IUIManager 接口实现
    std::unique_ptr<IConfigDialog>     createConfigDialog()     override;
    std::unique_ptr<IGameSelectDialog> createGameSelectDialog() override;
    std::unique_ptr<IFilePickerDialog> createFilePicker()       override;
    std::unique_ptr<IMenu>             createContextMenu()      override;

    void pushDialog(IDialog* dialog) override;
    void popDialog() override;
    int  getDialogStackDepth() const override;

    void showToast(const std::string& message,
                   ToastType type = ToastType::Info,
                   int durationMs = 3000) override;

    void showLoading(const std::string& message = "") override;
    void hideLoading() override;
    void updateLoadingProgress(float progress) override;

    bool onBackPressed() override;

    void injectPointerEvent(const PointerEvent& event) override;

    void initialize() override;
    void shutdown() override;

private:
    // 对话框栈：按顺序存储当前打开的对话框
    std::vector<IDialog*> m_dialogStack;

    // 加载覆盖层（Loading Overlay）Cocos2d 节点
    cocos2d::Layer*         m_loadingOverlay   {nullptr};
    cocos2d::ui::LoadingBar* m_progressBar     {nullptr};
    cocos2d::Label*          m_loadingLabel    {nullptr};

    // Toast 容器
    cocos2d::Node*  m_toastContainer {nullptr};

    // 获取当前运行场景（带空检查）
    cocos2d::Scene* getCurrentScene() const;

    // 内部：创建加载覆盖层（懒加载）
    void ensureLoadingOverlay();

    // 内部：Toast 动画（显示后 N 毫秒自动消失）
    void animateToastOut(cocos2d::Node* toast, int delayMs);
};

} // namespace ui
} // namespace krkr2
```

### 1.2 初始化与销毁

```cpp
// cpp/core/environ/ui/cocos2d/Cocos2dUIManager.cpp
#include "Cocos2dUIManager.h"
#include "Cocos2dConfigDialog.h"
#include "Cocos2dGameSelectDialog.h"
#include "Cocos2dFilePickerDialog.h"
#include "Cocos2dContextMenu.h"

namespace krkr2 {
namespace ui {

void Cocos2dUIManager::initialize() {
    // 创建 Toast 容器节点（固定附加到根场景，持续存在）
    m_toastContainer = cocos2d::Node::create();
    m_toastContainer->retain();

    auto director = cocos2d::Director::getInstance();
    auto size = director->getVisibleSize();
    // Toast 显示在屏幕底部居中
    m_toastContainer->setPosition(size.width / 2, 60);

    CCLOG("Cocos2dUIManager: 初始化完成");
}

void Cocos2dUIManager::shutdown() {
    // 清理对话框栈（不能有未关闭的对话框）
    for (auto* dialog : m_dialogStack) {
        dialog->destroy();
    }
    m_dialogStack.clear();

    // 清理加载覆盖层
    if (m_loadingOverlay) {
        m_loadingOverlay->removeFromParent();
        m_loadingOverlay->release();
        m_loadingOverlay = nullptr;
    }

    // 清理 Toast 容器
    if (m_toastContainer) {
        m_toastContainer->removeFromParent();
        m_toastContainer->release();
        m_toastContainer = nullptr;
    }

    CCLOG("Cocos2dUIManager: 已销毁");
}

Cocos2dUIManager::~Cocos2dUIManager() {
    shutdown();
}

cocos2d::Scene* Cocos2dUIManager::getCurrentScene() const {
    auto director = cocos2d::Director::getInstance();
    auto scene = director->getRunningScene();
    CCASSERT(scene, "Cocos2dUIManager: 没有运行中的场景！");
    return scene;
}

} // namespace ui
} // namespace krkr2
```

### 1.3 工厂方法实现

```cpp
// 工厂方法：创建 IConfigDialog
std::unique_ptr<IConfigDialog> Cocos2dUIManager::createConfigDialog() {
    return std::make_unique<Cocos2dConfigDialog>();
}

std::unique_ptr<IGameSelectDialog> Cocos2dUIManager::createGameSelectDialog() {
    return std::make_unique<Cocos2dGameSelectDialog>();
}

std::unique_ptr<IFilePickerDialog> Cocos2dUIManager::createFilePicker() {
    // 文件选择对话框在桌面端和移动端差异很大，需要平台分支
#if defined(ANDROID)
    // Android 使用系统文件选择器（通过 JNI 调用）
    return std::make_unique<AndroidFilePickerDialog>();
#elif defined(WINDOWS)
    // Windows 使用 Win32 OPENFILENAME 对话框
    return std::make_unique<Win32FilePickerDialog>();
#elif defined(LINUX)
    // Linux 使用 GTK FileChooserDialog（通过 Cocos2d-x 抽象）
    return std::make_unique<LinuxFilePickerDialog>();
#elif defined(MACOS)
    // macOS 使用 NSOpenPanel
    return std::make_unique<MacOSFilePickerDialog>();
#else
    // 未知平台：返回一个简单的文本输入对话框作为降级
    return std::make_unique<FallbackFilePickerDialog>();
#endif
}

std::unique_ptr<IMenu> Cocos2dUIManager::createContextMenu() {
    return std::make_unique<Cocos2dContextMenu>();
}
```

### 1.4 对话框栈管理

```cpp
// 推入对话框（每层 Z-order 递增 10，确保触摸优先级正确）
void Cocos2dUIManager::pushDialog(IDialog* dialog) {
    CCASSERT(dialog, "pushDialog: dialog 不能为空");

    const int BASE_Z = 1000;
    const int Z_STEP = 10;
    int zOrder = BASE_Z + static_cast<int>(m_dialogStack.size()) * Z_STEP;

    // 在 show() 之前调整 Z-order
    // （注意：IDialog 接口的 show() 内部负责 addChild，所以这里需要协调）
    // 实际实现中，Cocos2dConfigDialog::show() 需要接受 z-order 参数
    // 这里用一个内部 downcast 方案（仅 Cocos2d 后端适用）
    if (auto* cocos2dDialog = dynamic_cast<Cocos2dBaseDialog*>(dialog)) {
        cocos2dDialog->setZOrder(zOrder);
    }

    dialog->show();
    m_dialogStack.push_back(dialog);

    CCLOG("UIManager: pushDialog → 栈深度 %zu，Z-order=%d",
          m_dialogStack.size(), zOrder);
}

// 弹出最顶层对话框
void Cocos2dUIManager::popDialog() {
    if (m_dialogStack.empty()) {
        CCLOG("UIManager: popDialog 调用时栈为空");
        return;
    }
    auto* top = m_dialogStack.back();
    m_dialogStack.pop_back();
    top->hide();
    CCLOG("UIManager: popDialog → 栈深度 %zu", m_dialogStack.size());
}

int Cocos2dUIManager::getDialogStackDepth() const {
    return static_cast<int>(m_dialogStack.size());
}

// 处理返回键（Android 返回键 / 桌面 Escape 键）
bool Cocos2dUIManager::onBackPressed() {
    if (!m_dialogStack.empty()) {
        popDialog();
        return true;  // 已处理，不需要应用层再处理
    }
    return false;  // 未处理，应用层可以选择退出游戏
}
```

### 1.5 Toast 实现

```cpp
void Cocos2dUIManager::showToast(const std::string& message,
                                  ToastType type, int durationMs) {
    auto scene = getCurrentScene();

    // 确保 Toast 容器在当前场景中
    if (!m_toastContainer->getParent()) {
        scene->addChild(m_toastContainer, 9999);  // 最高 Z-order
    }

    // 根据类型选择颜色
    cocos2d::Color4B bgColor;
    switch (type) {
        case ToastType::Info:    bgColor = {50, 50, 50, 200};   break;
        case ToastType::Warning: bgColor = {180, 120, 0, 220};  break;
        case ToastType::Error:   bgColor = {180, 30, 30, 220};  break;
    }

    // 创建 Toast 背景（圆角矩形）
    auto toast = cocos2d::ui::Layout::create();
    toast->setBackGroundColorType(cocos2d::ui::Layout::BackGroundColorType::SOLID);
    toast->setBackGroundColor(cocos2d::Color3B(bgColor.r, bgColor.g, bgColor.b));
    toast->setBackGroundColorOpacity(bgColor.a);

    // 创建文字标签
    auto label = cocos2d::Label::createWithTTF(message, "fonts/arial.ttf", 16);
    label->setTextColor(cocos2d::Color4B::WHITE);

    // 自动调整 Toast 宽度以适应文字
    float padding = 20.0f;
    cocos2d::Size toastSize = {label->getContentSize().width + padding * 2,
                               label->getContentSize().height + padding};
    toast->setContentSize(toastSize);
    label->setPosition(toastSize.width / 2, toastSize.height / 2);
    toast->addChild(label);

    // 居中显示（叠加在已有 Toast 上方）
    float yOffset = m_toastContainer->getChildrenCount() * (toastSize.height + 8);
    toast->setPosition(-toastSize.width / 2, yOffset);

    m_toastContainer->addChild(toast);

    // 淡入动画
    toast->setOpacity(0);
    toast->runAction(cocos2d::FadeIn::create(0.2f));

    // 延迟淡出
    animateToastOut(toast, durationMs);
}

void Cocos2dUIManager::animateToastOut(cocos2d::Node* toast, int delayMs) {
    // 延迟 N 毫秒后执行淡出 + 移除
    auto delay   = cocos2d::DelayTime::create(delayMs / 1000.0f);
    auto fadeOut = cocos2d::FadeOut::create(0.3f);
    auto remove  = cocos2d::RemoveSelf::create();
    toast->runAction(cocos2d::Sequence::create(delay, fadeOut, remove, nullptr));
}
```

### 1.6 加载覆盖层

```cpp
void Cocos2dUIManager::ensureLoadingOverlay() {
    if (m_loadingOverlay) return;

    auto director = cocos2d::Director::getInstance();
    auto size = director->getVisibleSize();

    m_loadingOverlay = cocos2d::Layer::create();
    m_loadingOverlay->retain();

    // 全屏半透明黑色背景
    auto bg = cocos2d::LayerColor::create({0, 0, 0, 180},
                                           size.width, size.height);
    m_loadingOverlay->addChild(bg, 0);

    // 进度条（水平居中）
    m_progressBar = cocos2d::ui::LoadingBar::create(
        "ui/textures/progress_bar.png");
    m_progressBar->setDirection(
        cocos2d::ui::LoadingBar::Direction::LEFT);
    m_progressBar->setPosition({size.width / 2, size.height / 2 - 30});
    m_progressBar->setPercent(0);
    m_loadingOverlay->addChild(m_progressBar, 1);

    // 消息文字
    m_loadingLabel = cocos2d::Label::createWithTTF(
        "", "fonts/arial.ttf", 20);
    m_loadingLabel->setPosition({size.width / 2, size.height / 2 + 20});
    m_loadingLabel->setTextColor(cocos2d::Color4B::WHITE);
    m_loadingOverlay->addChild(m_loadingLabel, 1);
}

void Cocos2dUIManager::showLoading(const std::string& message) {
    ensureLoadingOverlay();

    auto scene = getCurrentScene();
    if (!m_loadingOverlay->getParent()) {
        scene->addChild(m_loadingOverlay, 5000);  // 比对话框还高
    }

    if (m_loadingLabel) {
        m_loadingLabel->setString(message);
    }
    if (m_progressBar) {
        m_progressBar->setPercent(0);
    }
}

void Cocos2dUIManager::hideLoading() {
    if (m_loadingOverlay) {
        m_loadingOverlay->removeFromParent();
    }
}

void Cocos2dUIManager::updateLoadingProgress(float progress) {
    if (m_progressBar) {
        m_progressBar->setPercent(static_cast<int>(progress * 100));
    }
}
```

## 二、更新 AppDelegate 的 UI 初始化流程

### 2.1 修改 AppDelegate.h

```cpp
// cpp/core/environ/cocos2d/AppDelegate.h（修改后）
#pragma once
#include "cocos2d.h"
#include "core/environ/ui/abstract/IUIManager.h"
#include <memory>

class AppDelegate : public cocos2d::Application {
public:
    bool applicationDidFinishLaunching() override;
    void applicationDidEnterBackground() override;
    void applicationWillEnterForeground() override;

    // 获取全局 UI 管理器（供其他模块使用）
    static krkr2::ui::IUIManager* getUIManager();

private:
    // UI 管理器（使用接口类型，不依赖具体实现）
    static std::unique_ptr<krkr2::ui::IUIManager> s_uiManager;
};
```

### 2.2 修改 AppDelegate.cpp

```cpp
// cpp/core/environ/cocos2d/AppDelegate.cpp（修改后）
#include "AppDelegate.h"
#include "core/environ/ui/abstract/UIBackendFactory.h"
#include "core/environ/ui/cocos2d/Cocos2dUIManager.h"

// 静态成员初始化
std::unique_ptr<krkr2::ui::IUIManager> AppDelegate::s_uiManager;

bool AppDelegate::applicationDidFinishLaunching() {
    // ... 原有 Cocos2d 初始化代码 ...
    auto director = cocos2d::Director::getInstance();
    // 设置帧率、OpenGL View 等

    // ---- 新增：UI 系统初始化 ----
    // 注册 Cocos2d-x UI 后端
    krkr2::ui::UIBackendFactory::registerBackend(
        krkr2::ui::UIBackend::Cocos2d,
        []() -> std::unique_ptr<krkr2::ui::IUIManager> {
            return std::make_unique<krkr2::ui::Cocos2dUIManager>();
        }
    );

    // 创建 UI 管理器（根据编译宏选择后端）
    s_uiManager = krkr2::ui::UIBackendFactory::createDefault();
    s_uiManager->initialize();

    // 注册返回键回调（移动端 + 桌面端 Escape）
    auto keyListener = cocos2d::EventListenerKeyboard::create();
    keyListener->onKeyReleased = [](cocos2d::EventKeyboard::KeyCode code,
                                    cocos2d::Event*) {
        if (code == cocos2d::EventKeyboard::KeyCode::KEY_BACK ||
            code == cocos2d::EventKeyboard::KeyCode::KEY_ESCAPE) {
            bool handled = AppDelegate::getUIManager()->onBackPressed();
            if (!handled) {
                // 没有打开的对话框时，可以选择显示"确认退出"对话框
                CCLOG("返回键：无打开的对话框，显示退出确认");
            }
        }
    };
    director->getEventDispatcher()->addEventListenerWithFixedPriority(
        keyListener, -1);

    // ---- UI 系统初始化结束 ----

    // 显示游戏选择对话框（首次启动）
    auto gameSelectDialog = s_uiManager->createGameSelectDialog();
    gameSelectDialog->setOnGameSelected([](const krkr2::ui::IGameSelectDialog::GameEntry& game) {
        CCLOG("用户选择游戏：%s", game.name.c_str());
        // 启动游戏加载流程
    });
    s_uiManager->pushDialog(gameSelectDialog.release());

    return true;
}

krkr2::ui::IUIManager* AppDelegate::getUIManager() {
    return s_uiManager.get();
}

void AppDelegate::applicationDidEnterBackground() {
    // 应用进入后台时，暂停 UI（如暂停 Toast 动画）
    // s_uiManager 无需特殊处理，Cocos2d-x 会暂停所有 Action
}

void AppDelegate::applicationWillEnterForeground() {
    // 应用恢复前台时，恢复 UI
}
```

## 三、平台差异处理

不同平台的文件选择对话框实现差异最大，以下是四个平台的实现方案：

### 3.1 Windows：Win32 OPENFILENAME

```cpp
// cpp/core/environ/ui/cocos2d/Win32FilePickerDialog.cpp
#ifdef WINDOWS
#include "Win32FilePickerDialog.h"
#include <windows.h>
#include <commdlg.h>  // OPENFILENAME
#include <string>
#include <vector>

namespace krkr2::ui {

void Win32FilePickerDialog::show() {
    // Win32 文件选择对话框在主线程中打开
    // 注意：OpenFileDialog 是模态的（会阻塞）
    // 需要在非 Cocos2d 渲染线程中调用，或者使用异步版本

    OPENFILENAMEW ofn = {};
    wchar_t szFile[MAX_PATH] = L"";

    // 构建过滤器字符串（如 "XP3 Files\0*.xp3\0All Files\0*.*\0\0"）
    std::wstring filter = buildFilterString();

    ofn.lStructSize     = sizeof(ofn);
    ofn.hwndOwner       = getMainWindowHandle();  // 获取 Win32 窗口句柄
    ofn.lpstrFile       = szFile;
    ofn.nMaxFile        = MAX_PATH;
    ofn.lpstrFilter     = filter.c_str();
    ofn.nFilterIndex    = 1;
    ofn.lpstrInitialDir = m_initialDir.empty() ? nullptr
                                               : toWide(m_initialDir).c_str();
    ofn.Flags           = OFN_PATHMUSTEXIST | OFN_FILEMUSTEXIST;

    if (GetOpenFileNameW(&ofn)) {
        std::string selectedPath = fromWide(szFile);
        if (m_onFileSelected) {
            // 切回 Cocos2d 主线程执行回调
            cocos2d::Director::getInstance()
                ->getScheduler()
                ->performFunctionInCocosThread([this, selectedPath]() {
                    m_onFileSelected(selectedPath);
                });
        }
    }
}

} // namespace krkr2::ui
#endif
```

### 3.2 Android：JNI 调用系统文件选择器

```cpp
// cpp/core/environ/ui/cocos2d/AndroidFilePickerDialog.cpp
#ifdef ANDROID
#include "AndroidFilePickerDialog.h"
#include <jni.h>
#include "platform/android/jni/JniHelper.h"

namespace krkr2::ui {

void AndroidFilePickerDialog::show() {
    // 通过 JNI 调用 Android Activity 的 startActivityForResult
    // Activity 代码见 platforms/android/java/...
    cocos2d::JniMethodInfo methodInfo;
    if (cocos2d::JniHelper::getStaticMethodInfo(
            methodInfo,
            "org/cocos2dx/krkr2/KrkrActivity",
            "showFilePicker",
            "([Ljava/lang/String;)V")) {

        // 构建文件扩展名 Java 数组
        JNIEnv* env = methodInfo.env;
        jobjectArray extensions = env->NewObjectArray(
            m_filters.size(),
            env->FindClass("java/lang/String"),
            nullptr);
        for (size_t i = 0; i < m_filters.size(); i++) {
            env->SetObjectArrayElement(extensions, i,
                env->NewStringUTF(m_filters[i].c_str()));
        }

        // 调用 Java 方法（异步，结果通过 onFilePickerResult JNI 回调返回）
        env->CallStaticVoidMethod(methodInfo.classID,
                                  methodInfo.methodID, extensions);
        env->DeleteLocalRef(extensions);
        env->DeleteLocalRef(methodInfo.classID);
    }
}

// JNI 回调：当用户在 Android 文件选择器中选好文件后被 Java 层调用
extern "C" JNIEXPORT void JNICALL
Java_org_cocos2dx_krkr2_KrkrActivity_onFilePickerResult(
    JNIEnv* env, jobject, jstring jpath) {

    const char* path = env->GetStringUTFChars(jpath, nullptr);
    std::string selectedPath(path);
    env->ReleaseStringUTFChars(jpath, path);

    // 找到当前的 AndroidFilePickerDialog 实例并触发回调
    // （通过 UIManager 的弱引用获取）
    cocos2d::Director::getInstance()
        ->getScheduler()
        ->performFunctionInCocosThread([selectedPath]() {
            auto* uiManager = AppDelegate::getUIManager();
            // 触发文件选择回调
            CCLOG("Android 文件选择结果：%s", selectedPath.c_str());
        });
}

} // namespace krkr2::ui
#endif
```

### 3.3 平台差异汇总

| 功能 | Windows | Linux | macOS | Android |
|------|---------|-------|-------|---------|
| 文件选择对话框 | Win32 `OPENFILENAME` | GTK3 `GtkFileChooser` | `NSOpenPanel` | Intent + ActivityResult |
| 目录选择对话框 | `SHBrowseForFolder` | GTK3 `GtkFileChooser` | `NSOpenPanel` (dirs) | SAF (Storage Access Framework) |
| 颜色选择对话框 | `ChooseColor` | GTK3 `GtkColorChooser` | `NSColorPanel` | 自定义 View |
| 字体选择对话框 | `ChooseFont` | GTK3 `GtkFontChooser` | `NSFontPanel` | 自定义 View |
| 通知/Toast | Win32 Balloon tip | libnotify | `NSUserNotification` | `Toast.makeText()` |

## 四、集成测试：验证 UIManager 生命周期

```cpp
// tests/unit-tests/ui/ui_manager_integration_test.cpp
#define CATCH_CONFIG_RUNNER
#include <catch2/catch_all.hpp>
#include "core/environ/ui/abstract/IUIManager.h"
#include <vector>
#include <string>

// 完整的 Mock UIManager（用于测试业务逻辑）
class MockUIManager : public krkr2::ui::IUIManager {
public:
    std::unique_ptr<krkr2::ui::IConfigDialog>     createConfigDialog() override {
        return std::make_unique<MockConfigDialog>(); }
    std::unique_ptr<krkr2::ui::IGameSelectDialog> createGameSelectDialog() override {
        return std::make_unique<MockGameSelectDialog>(); }
    std::unique_ptr<krkr2::ui::IFilePickerDialog> createFilePicker() override {
        return std::make_unique<MockFilePickerDialog>(); }
    std::unique_ptr<krkr2::ui::IMenu>             createContextMenu() override {
        return std::make_unique<MockMenu>(); }

    void pushDialog(krkr2::ui::IDialog* dialog) override {
        m_stack.push_back(dialog);
        dialog->show();
    }
    void popDialog() override {
        if (!m_stack.empty()) {
            m_stack.back()->hide();
            m_stack.pop_back();
        }
    }
    int getDialogStackDepth() const override {
        return static_cast<int>(m_stack.size()); }

    void showToast(const std::string& msg, krkr2::ui::ToastType, int) override {
        m_toasts.push_back(msg); }
    void showLoading(const std::string& msg) override { m_loading = true; m_loadingMsg = msg; }
    void hideLoading() override { m_loading = false; }
    void updateLoadingProgress(float p) override { m_progress = p; }
    bool onBackPressed() override {
        if (m_stack.empty()) return false;
        popDialog();
        return true;
    }
    void injectPointerEvent(const krkr2::ui::PointerEvent&) override {}
    void initialize() override { m_initialized = true; }
    void shutdown() override { m_initialized = false; }

    // 测试可观察的状态
    std::vector<krkr2::ui::IDialog*> m_stack;
    std::vector<std::string>          m_toasts;
    bool    m_initialized {false};
    bool    m_loading     {false};
    float   m_progress    {0.0f};
    std::string m_loadingMsg;
};

TEST_CASE("UIManager 对话框栈管理", "[ui][manager]") {
    MockUIManager manager;
    manager.initialize();
    REQUIRE(manager.m_initialized);

    SECTION("初始状态：栈为空") {
        REQUIRE(manager.getDialogStackDepth() == 0);
    }

    SECTION("pushDialog 增加栈深度") {
        auto dialog = manager.createConfigDialog();
        auto* raw = dialog.get();
        manager.pushDialog(raw);
        REQUIRE(manager.getDialogStackDepth() == 1);
        REQUIRE(raw->isVisible());
    }

    SECTION("popDialog 减少栈深度") {
        auto d1 = manager.createConfigDialog();
        auto d2 = manager.createGameSelectDialog();
        manager.pushDialog(d1.get());
        manager.pushDialog(d2.get());
        REQUIRE(manager.getDialogStackDepth() == 2);
        manager.popDialog();
        REQUIRE(manager.getDialogStackDepth() == 1);
        REQUIRE_FALSE(d2->isVisible());
        REQUIRE(d1->isVisible());
    }

    SECTION("onBackPressed 关闭最顶层对话框") {
        auto dialog = manager.createConfigDialog();
        manager.pushDialog(dialog.get());
        REQUIRE(manager.onBackPressed() == true);  // 已处理
        REQUIRE(manager.getDialogStackDepth() == 0);
        REQUIRE(manager.onBackPressed() == false);  // 栈已空，未处理
    }
}

TEST_CASE("UIManager Toast 系统", "[ui][toast]") {
    MockUIManager manager;

    SECTION("showToast 添加到列表") {
        manager.showToast("测试消息", krkr2::ui::ToastType::Info);
        REQUIRE(manager.m_toasts.size() == 1);
        REQUIRE(manager.m_toasts[0] == "测试消息");
    }
}

TEST_CASE("UIManager 加载覆盖层", "[ui][loading]") {
    MockUIManager manager;

    SECTION("showLoading/hideLoading 状态切换") {
        REQUIRE_FALSE(manager.m_loading);
        manager.showLoading("正在加载...");
        REQUIRE(manager.m_loading);
        REQUIRE(manager.m_loadingMsg == "正在加载...");
        manager.hideLoading();
        REQUIRE_FALSE(manager.m_loading);
    }

    SECTION("updateLoadingProgress 更新进度") {
        manager.showLoading();
        manager.updateLoadingProgress(0.5f);
        REQUIRE(manager.m_progress == Approx(0.5f));
    }
}
```

## 五、对照项目源码

实现完成后的文件结构：

相关文件：
- `cpp/core/environ/ui/abstract/` — 抽象接口（本节新增）
- `cpp/core/environ/ui/cocos2d/Cocos2dUIManager.cpp` — 主要实现（本节新增）
- `cpp/core/environ/ui/cocos2d/Cocos2dConfigDialog.cpp` — ConfigDialog 适配（本节新增）
- `cpp/core/environ/cocos2d/AppDelegate.cpp` — 更新 UI 初始化流程（本节修改）
- `platforms/android/cpp/krkr2_android.cpp` — Android JNI 层（文件选择器回调入口）
- `tests/unit-tests/ui/` — UI 集成测试（本节新增）

## 本节小结

- `Cocos2dUIManager` 是将现有 Cocos2d-x UI 包装为 `IUIManager` 接口的适配器
- 对话框栈通过递增 Z-order 确保触摸优先级正确，`onBackPressed()` 按序关闭
- 文件选择对话框是跨平台差异最大的 UI 组件，需要四平台各自实现
- Mock UIManager 让所有 UI 逻辑都可以在无图形上下文的环境中单元测试
- 所有 Cocos2d-x 线程安全问题（工作线程修改 UI）已通过 `performFunctionInCocosThread` 解决

## 练习题与答案

### 题目 1：生命周期安全

以下代码存在潜在的 use-after-free 风险，请说明风险在哪里，并给出修复方案：

```cpp
void GameController::loadGameAsync(const std::string& path) {
    auto* uiManager = AppDelegate::getUIManager();
    uiManager->showLoading("正在加载...");

    std::thread([uiManager, path]() {
        loadGameFiles(path);  // 耗时操作
        uiManager->hideLoading();  // 有风险！
    }).detach();
}
```

<details>
<summary>查看答案</summary>

**风险分析：**

`uiManager` 是一个原始指针，指向由 `AppDelegate::s_uiManager` 持有的 `IUIManager` 对象。如果在工作线程 `hideLoading()` 执行之前，应用发生了以下情况：
1. 用户快速关闭应用 → `AppDelegate` 析构 → `s_uiManager` 被释放 → `uiManager` 悬空
2. 场景切换导致 `Cocos2dUIManager` 被销毁

此外，`hideLoading()` 直接在工作线程中调用，会操作 `cocos2d::Node`（加载覆盖层），这在非主线程中是不安全的。

**修复方案：**

```cpp
void GameController::loadGameAsync(const std::string& path) {
    AppDelegate::getUIManager()->showLoading("正在加载...");

    // 不捕获原始指针，而是在回调中通过全局单例重新获取
    std::thread([path]() {
        loadGameFiles(path);  // 耗时操作

        // 安全地切回主线程操作 UI
        cocos2d::Director::getInstance()
            ->getScheduler()
            ->performFunctionInCocosThread([]() {
                // 在主线程中重新获取 UIManager（不持有跨线程指针）
                auto* uiManager = AppDelegate::getUIManager();
                if (uiManager) {  // 防御性检查
                    uiManager->hideLoading();
                }
            });
    }).detach();
}
```

关键修复点：
1. 工作线程 lambda 不捕获 `uiManager` 指针
2. UI 操作通过 `performFunctionInCocosThread` 在主线程执行
3. 在主线程中重新通过单例获取指针，避免跨线程持有悬空引用

</details>

### 题目 2：文件选择器的平台抽象

目前 `createFilePicker()` 中的 `#ifdef ANDROID/#ifdef WINDOWS/...` 链看起来会越来越长。请提出一种更好的架构方案，使得新增平台时不需要修改 `Cocos2dUIManager.cpp`。

<details>
<summary>查看答案</summary>

**方案：平台专属的工厂注册**

在每个平台的 CMake 目标中，注册该平台的文件选择器工厂，`Cocos2dUIManager` 只调用工厂，不关心具体实现：

```cpp
// cpp/core/environ/ui/abstract/FilePicker Factory.h
namespace krkr2::ui {
class FilePickerFactory {
public:
    using Creator = std::function<std::unique_ptr<IFilePickerDialog>()>;
    static void registerCreator(Creator c) { s_creator = std::move(c); }
    static std::unique_ptr<IFilePickerDialog> create() {
        if (!s_creator) throw std::runtime_error("No FilePicker registered");
        return s_creator();
    }
private:
    static inline Creator s_creator;
};
}
```

```cpp
// cpp/core/environ/ui/cocos2d/Cocos2dUIManager.cpp（简化版）
std::unique_ptr<IFilePickerDialog> Cocos2dUIManager::createFilePicker() {
    return FilePickerFactory::create();  // 不再有 #ifdef 链
}
```

```cpp
// platforms/windows/UIRegistration.cpp（Windows 专属文件）
#include "core/environ/ui/abstract/FilePickerFactory.h"
#include "core/environ/ui/cocos2d/Win32FilePickerDialog.h"

// 在 main() 之前（静态初始化）注册 Windows 文件选择器
static bool s_registered = []() {
    krkr2::ui::FilePickerFactory::registerCreator([]() {
        return std::make_unique<krkr2::ui::Win32FilePickerDialog>();
    });
    return true;
}();
```

每个平台只需提供自己的 `UIRegistration.cpp`，通过 CMake 的 `if(WINDOWS)/if(ANDROID)/...` 选择编译哪个文件。新增平台时只需新增一个注册文件，核心代码零改动。

</details>

### 题目 3：对话框栈的内存所有权

`pushDialog(IDialog* dialog)` 的参数是裸指针，谁负责销毁对话框？请描述一个完整的所有权管理方案，避免内存泄漏。

<details>
<summary>查看答案</summary>

**当前设计的问题：**

`IUIManager::pushDialog(IDialog*)` 接受裸指针，意味着：
- 如果调用者传入一个临时 `unique_ptr.release()` 的结果，UIManager 持有对象但没有 unique_ptr 的所有权
- 如果 UIManager 内部只是存指针（不 delete），调用者 `popDialog` 后还要自己 delete，但此时调用者可能已经丢失了原来的 unique_ptr

**推荐方案：UIManager 接管所有权**

```cpp
// 修改接口，接受 unique_ptr（所有权转移给 UIManager）
class IUIManager {
    // 将所有权转移给 UIManager
    virtual void pushDialog(std::unique_ptr<IDialog> dialog) = 0;

    // pop 时返回 unique_ptr（所有权还给调用者，可以检查或丢弃）
    virtual std::unique_ptr<IDialog> popDialog() = 0;
};

// Cocos2dUIManager 中存储 unique_ptr
std::vector<std::unique_ptr<IDialog>> m_dialogStack;

void Cocos2dUIManager::pushDialog(std::unique_ptr<IDialog> dialog) {
    dialog->show();
    m_dialogStack.push_back(std::move(dialog));
}

std::unique_ptr<IDialog> Cocos2dUIManager::popDialog() {
    if (m_dialogStack.empty()) return nullptr;
    auto top = std::move(m_dialogStack.back());
    m_dialogStack.pop_back();
    top->hide();
    return top;  // 调用者决定是否保留或销毁
}
```

使用方式：
```cpp
// 推入：所有权转移给 UIManager
auto dialog = uiManager->createConfigDialog();
// ... 配置 dialog ...
uiManager->pushDialog(std::move(dialog));  // dialog 变为空 unique_ptr

// 弹出：所有权还给调用者，离开作用域自动销毁
{
    auto top = uiManager->popDialog();
    // top 在此作用域结束时自动调用 ~IDialog()
}
```

这样保证每个对话框对象都有明确的所有权，UIManager 销毁时会自动析构栈内所有对话框，没有内存泄漏风险。

</details>

## 下一步

[03-Flutter嵌入实战](../03-Flutter嵌入实战/01-FlutterEngine集成与CMake配置.md) — 完成 UI 抽象层的接口设计后，下一章将实现 `FlutterUIManager`：把 Flutter Engine 嵌入到 KrKr2，通过 Platform Channel 实现 UI 交互。
