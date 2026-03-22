# 02 - Compose 双端适配与 C++ 互调

> **所属模块：** M13-UI框架替换实战  
> **前置知识：** [04-Compose嵌入实战/01-KotlinNative与cinterop](01-KotlinNative与cinterop.md)  
> **预计阅读时间：** 65 分钟

---

## 本节目标

- 理解 Compose Multiplatform 的渲染机制与平台层架构
- 掌握在 Desktop（Linux/Windows/macOS）和 Android 双端共享 Compose UI 代码
- 学会通过函数指针实现 C++ 调用 Kotlin Composable（反向调用）
- 能够构建一个完整的 KrKr2 主菜单 Compose UI，并集成到 C++ 宿主程序

---

## 术语预览

| 术语 | 解释 |
|---|---|
| **Compose Multiplatform** | JetBrains 基于 Jetpack Compose 扩展的跨平台 UI 框架，支持 Desktop、Android、iOS 和 Web |
| **`@Composable`** | Kotlin 注解，标记一个函数为 Compose 组件（可组合函数），描述 UI 结构而非命令式绘制 |
| **remember** | Compose 中用于在重组（recomposition）之间保持状态的函数，防止每次渲染都重新初始化 |
| **State / MutableState** | Compose 的响应式状态容器，值变化时自动触发相关 Composable 重新渲染 |
| **ComposeWindow** | Compose Desktop 的独立窗口类型，基于 Skia 渲染 |
| **ComposePanel** | Compose Desktop 的嵌入面板，可嵌入 Swing/AWT 窗口中 |
| **`singleWindowApplication`** | Compose Desktop 创建单窗口应用的便捷函数 |
| **exported function** | KN 中通过 `@CName` 注解导出的 Kotlin 函数，可直接从 C++ 通过函数指针调用 |

---

## 一、Compose Multiplatform 架构概览

### 1.1 Compose 的渲染管线

Compose Multiplatform 在不同平台上使用不同的底层渲染技术：

```
┌────────────────────────────────────────────────────────────┐
│  Compose UI 代码（共享，@Composable 函数）                  │
│  Widget → Layout → Drawing Commands                        │
├──────────────┬──────────────────────┬──────────────────────┤
│  Desktop     │  Android             │  iOS（实验性）        │
│  Skia/GPU    │  Android Canvas/GPU  │  Metal + UIKit        │
│  via Skiko   │  （Jetpack Compose） │  via Skiko            │
├──────────────┴──────────────────────┴──────────────────────┤
│  Skiko（Skia 的 Kotlin 多平台绑定）                         │
│  提供统一的 2D 渲染 API                                     │
└────────────────────────────────────────────────────────────┘
```

**Skiko** 是 JetBrains 维护的 Skia 图形库的 Kotlin 多平台封装。Desktop 平台上，Compose 在一个独立的 OpenGL/Vulkan/Metal/DirectX 上下文中渲染，与 KrKr2 的 Cocos2d-x 上下文并列存在。

### 1.2 KrKr2 的集成方案

KrKr2 使用 Compose 替换 UI 层，采用两种集成模式：

**模式 A：独立 Compose 窗口（Desktop 推荐）**
```
KrKr2 主窗口（Cocos2d-x OpenGL）
    │
    ├── 游戏内容区域（Cocos2d-x 渲染）
    └── [独立 ComposeWindow] ← 对话框/菜单（独立窗口）
          Compose 窗口层叠在主窗口上
```

**模式 B：Compose 嵌入 Android Activity（Android 推荐）**
```
KrKr2 Android Activity
    │
    ├── GLSurfaceView（Cocos2d-x 渲染游戏内容）
    └── ComposeView（叠加在 GLSurfaceView 上方）
          通过 FrameLayout 覆盖层实现
```

---

## 二、Compose Desktop 集成

### 2.1 Gradle 配置（Desktop 目标）

```kotlin
// kotlin_module/build.gradle.kts
plugins {
    kotlin("multiplatform") version "1.9.22"
    id("org.jetbrains.compose") version "1.6.1"
}

kotlin {
    jvm("desktop") {
        compilations.all {
            kotlinOptions.jvmTarget = "17"
        }
    }
    
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation(compose.runtime)
                implementation(compose.foundation)
                implementation(compose.material3)
                implementation(compose.ui)
            }
        }
        
        val desktopMain by getting {
            dependencies {
                implementation(compose.desktop.currentOs)
                // Compose Desktop 需要 Swing 运行时
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-swing:1.7.3")
            }
        }
    }
}

compose.desktop {
    application {
        mainClass = "MainKt"
        
        nativeDistributions {
            targetFormats(
                org.jetbrains.compose.desktop.application.dsl.TargetFormat.Deb,  // Linux
                org.jetbrains.compose.desktop.application.dsl.TargetFormat.Msi,  // Windows
                org.jetbrains.compose.desktop.application.dsl.TargetFormat.Dmg   // macOS
            )
            packageName = "krkr2"
            packageVersion = "2.0.0"
        }
    }
}
```

### 2.2 共享 UI 代码（commonMain）

```kotlin
// kotlin_module/src/commonMain/kotlin/ui/KrKr2MainMenu.kt
package ui

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp

/**
 * KrKr2 主菜单 UI（平台无关的 Compose 代码）
 *
 * @param version       版本字符串（由 C++ 通过回调提供）
 * @param platform      平台名称
 * @param onGameSelect  "选择游戏"按钮点击回调
 * @param onConfig      "设置"按钮点击回调
 * @param onQuit        "退出"按钮点击回调
 */
@Composable
fun KrKr2MainMenu(
    version: String = "2.0.0",
    platform: String = "desktop",
    onGameSelect: () -> Unit = {},
    onConfig: () -> Unit = {},
    onQuit: () -> Unit = {},
) {
    // 使用 Material3 深色主题
    MaterialTheme(
        colorScheme = darkColorScheme(
            primary      = Color(0xFF7B61FF),
            surface      = Color(0xFF1A1A2E),
            background   = Color(0xFF16213E),
        )
    ) {
        Box(
            modifier = Modifier
                .fillMaxSize()
                .background(MaterialTheme.colorScheme.background),
            contentAlignment = Alignment.Center
        ) {
            Column(
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.spacedBy(16.dp)
            ) {
                // 标题
                Text(
                    text = "KrKr2",
                    fontSize = 48.sp,
                    fontWeight = FontWeight.Bold,
                    color = MaterialTheme.colorScheme.primary
                )
                
                Text(
                    text = "v$version · $platform",
                    fontSize = 14.sp,
                    color = Color.Gray
                )
                
                Spacer(Modifier.height(32.dp))
                
                // 主要操作按钮
                MenuButton(text = "选择游戏", onClick = onGameSelect)
                MenuButton(text = "设置",     onClick = onConfig)
                
                Spacer(Modifier.height(16.dp))
                
                // 退出按钮（次要样式）
                OutlinedButton(
                    onClick = onQuit,
                    modifier = Modifier.width(200.dp)
                ) {
                    Text("退出")
                }
            }
        }
    }
}

@Composable
private fun MenuButton(text: String, onClick: () -> Unit) {
    Button(
        onClick = onClick,
        modifier = Modifier
            .width(200.dp)
            .height(48.dp),
        colors = ButtonDefaults.buttonColors(
            containerColor = MaterialTheme.colorScheme.primary
        )
    ) {
        Text(text, fontSize = 16.sp)
    }
}

// ─── 配置对话框 ───────────────────────────────────────────

/**
 * KrKr2 设置对话框
 */
@Composable
fun KrKr2ConfigDialog(
    initialTab: String = "general",
    onDismiss: () -> Unit = {},
    onSave: (Map<String, Any>) -> Unit = {},
) {
    var selectedTab by remember { mutableStateOf(initialTab) }
    val tabs = listOf("general", "video", "audio")
    
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("设置") },
        text = {
            Column {
                // 标签页
                TabRow(selectedTabIndex = tabs.indexOf(selectedTab)) {
                    tabs.forEach { tab ->
                        Tab(
                            selected = selectedTab == tab,
                            onClick  = { selectedTab = tab },
                            text     = { Text(tab.replaceFirstChar { it.uppercase() }) }
                        )
                    }
                }
                
                Spacer(Modifier.height(16.dp))
                
                // 根据选中标签页显示内容
                when (selectedTab) {
                    "general" -> GeneralSettings()
                    "video"   -> VideoSettings()
                    "audio"   -> AudioSettings()
                }
            }
        },
        confirmButton = {
            TextButton(onClick = { onSave(emptyMap()); onDismiss() }) {
                Text("保存")
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) { Text("取消") }
        }
    )
}

@Composable
private fun GeneralSettings() {
    var autoSave by remember { mutableStateOf(true) }
    var language by remember { mutableStateOf("zh-CN") }
    
    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        Row(
            verticalAlignment = Alignment.CenterVertically,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("自动存档", modifier = Modifier.weight(1f))
            Switch(checked = autoSave, onCheckedChange = { autoSave = it })
        }
        Text("语言：$language", style = MaterialTheme.typography.bodyMedium)
    }
}

@Composable
private fun VideoSettings() {
    var fullscreen by remember { mutableStateOf(false) }
    var scale by remember { mutableStateOf(1.0f) }
    
    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            Text("全屏", modifier = Modifier.weight(1f))
            Switch(checked = fullscreen, onCheckedChange = { fullscreen = it })
        }
        Text("缩放：${(scale * 100).toInt()}%")
        Slider(value = scale, onValueChange = { scale = it }, valueRange = 0.5f..2.0f)
    }
}

@Composable
private fun AudioSettings() {
    var volume by remember { mutableStateOf(0.8f) }
    
    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        Text("主音量：${(volume * 100).toInt()}%")
        Slider(value = volume, onValueChange = { volume = it }, valueRange = 0f..1f)
    }
}
```

### 2.3 Desktop 入口（desktopMain）

```kotlin
// kotlin_module/src/desktopMain/kotlin/Main.kt
package main

import androidx.compose.runtime.*
import androidx.compose.ui.window.*
import ui.KrKr2MainMenu
import ui.KrKr2ConfigDialog

/**
 * KrKr2 Compose Desktop 主入口
 * 提供两种启动方式：
 *   1. 独立模式（直接运行 Compose 应用）
 *   2. 嵌入模式（从 C++ 调用，作为覆盖层窗口）
 */

// ─── 独立模式 ──────────────────────────────────────────────

fun main() = singleWindowApplication(
    title = "KrKr2",
    state = WindowState(width = 800.dp, height = 600.dp)
) {
    KrKr2App()
}

@Composable
fun ApplicationScope.KrKr2App() {
    var showConfig by remember { mutableStateOf(false) }
    var version by remember { mutableStateOf("2.0.0") }
    
    KrKr2MainMenu(
        version = version,
        platform = "desktop",
        onGameSelect = { /* 通过 cinterop 调用 C++ */ },
        onConfig = { showConfig = true },
        onQuit = { exitApplication() }
    )
    
    if (showConfig) {
        KrKr2ConfigDialog(
            onDismiss = { showConfig = false }
        )
    }
}

// ─── 嵌入模式（供 C++ 调用）───────────────────────────────

/**
 * 创建并显示覆盖层 Compose 窗口
 * 此函数通过 @CName 导出为 C 函数，供 C++ 调用
 */
@OptIn(ExperimentalComposeUiApi::class)
fun showMainMenuOverlay(
    parentX: Int, parentY: Int,
    width: Int, height: Int,
    onGameSelect: () -> Unit,
    onConfig: () -> Unit,
    onQuit: () -> Unit,
) {
    // 在独立线程启动 Compose 应用（避免阻塞 C++ 主线程）
    Thread {
        application {
            Window(
                onCloseRequest = ::exitApplication,
                title = "",
                state = WindowState(
                    position = WindowPosition(parentX.dp, parentY.dp),
                    width = width.dp, height = height.dp
                ),
                // 无装饰（无标题栏），作为覆盖层
                undecorated = true,
                transparent = true,
                alwaysOnTop = true
            ) {
                KrKr2MainMenu(
                    onGameSelect = onGameSelect,
                    onConfig = onConfig,
                    onQuit = onQuit
                )
            }
        }
    }.also {
        it.isDaemon = true
        it.name = "KrKr2-Compose-UI"
        it.start()
    }
}
```

---

## 三、Android 集成

### 3.1 Android 目标的 Gradle 配置

Android 上 Compose 不走 KN 路径，而是使用标准 JVM（Kotlin/JVM + AGP）：

```kotlin
// kotlin_module/build.gradle.kts（Android 目标补充）
plugins {
    kotlin("multiplatform") version "1.9.22"
    id("com.android.library")
    id("org.jetbrains.compose") version "1.6.1"
}

kotlin {
    androidTarget {
        compilations.all {
            kotlinOptions.jvmTarget = "17"
        }
    }
    
    sourceSets {
        val androidMain by getting {
            dependencies {
                implementation("androidx.activity:activity-compose:1.8.2")
                implementation("androidx.compose.ui:ui:1.6.1")
                implementation("androidx.compose.material3:material3:1.2.0")
            }
        }
    }
}

android {
    compileSdk = 34
    defaultConfig {
        minSdk = 24
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}
```

### 3.2 Android Compose 嵌入（在 GLSurfaceView 上叠加）

```kotlin
// kotlin_module/src/androidMain/kotlin/KrKr2ComposeOverlay.kt
package ui

import android.content.Context
import android.view.ViewGroup
import android.widget.FrameLayout
import androidx.activity.ComponentActivity
import androidx.compose.runtime.*
import androidx.compose.ui.platform.ComposeView
import androidx.compose.ui.platform.ViewCompositionStrategy

/**
 * 将 Compose UI 作为覆盖层叠加到 KrKr2 Android Activity 上
 *
 * 使用方法（在 KrKr2MainActivity 中）：
 * ```java
 * KrKr2ComposeOverlay overlay = new KrKr2ComposeOverlay(this);
 * overlay.show();
 * ```
 */
class KrKr2ComposeOverlay(private val activity: ComponentActivity) {
    
    private var composeView: ComposeView? = null
    
    /**
     * 显示 Compose 覆盖层
     * 将 ComposeView 添加到 Activity 的根 ViewGroup 上
     */
    fun show() {
        val rootView = activity.window.decorView
            .findViewById<ViewGroup>(android.R.id.content)
        
        if (composeView != null) return  // 已显示
        
        val view = ComposeView(activity).apply {
            // 与宿主 Activity 的生命周期绑定
            setViewCompositionStrategy(
                ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
            )
            
            setContent {
                KrKr2MainMenuAndroid(
                    onGameSelect = {
                        // 通过 JNI 调用 C++ 显示文件选择对话框
                        KrKr2JNI.showGameSelectDialog()
                    },
                    onConfig = {
                        KrKr2JNI.showConfigDialog()
                    },
                    onQuit = {
                        activity.finish()
                    }
                )
            }
            
            layoutParams = FrameLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT,
                ViewGroup.LayoutParams.MATCH_PARENT
            )
        }
        
        rootView.addView(view)
        composeView = view
    }
    
    /** 隐藏 Compose 覆盖层（游戏进行中时调用） */
    fun hide() {
        composeView?.let {
            (it.parent as? ViewGroup)?.removeView(it)
            composeView = null
        }
    }
}

/**
 * Android 特定的 JNI 桥接
 * C++ 函数通过 JNI 注册，在 KrKr2 初始化时自动绑定
 */
object KrKr2JNI {
    init {
        System.loadLibrary("krkr2")  // 加载 libkrkr2.so
    }
    
    external fun showGameSelectDialog(): String?
    external fun showConfigDialog()
    external fun getVersion(): String
    external fun getPlatform(): String
}
```

### 3.3 Android Compose UI（可复用大部分 commonMain 代码）

```kotlin
// kotlin_module/src/androidMain/kotlin/ui/KrKr2MainMenuAndroid.kt
package ui

import androidx.compose.runtime.*

/**
 * Android 版主菜单
 * 复用 commonMain 的 KrKr2MainMenu Composable，提供 Android 特定的回调实现
 */
@Composable
fun KrKr2MainMenuAndroid(
    onGameSelect: () -> Unit,
    onConfig: () -> Unit,
    onQuit: () -> Unit,
) {
    var showConfig by remember { mutableStateOf(false) }
    
    KrKr2MainMenu(  // 来自 commonMain，完全共享 UI 代码
        version = remember { KrKr2JNI.getVersion() },
        platform = "android",
        onGameSelect = onGameSelect,
        onConfig = { showConfig = true },
        onQuit = onQuit
    )
    
    if (showConfig) {
        KrKr2ConfigDialog(
            onDismiss = { showConfig = false },
            onSave = { settings ->
                // 保存设置到 KrKr2
                // KrKr2JNI.saveSettings(settings)
                showConfig = false
            }
        )
    }
}
```

---

## 四、C++ 调用 Kotlin（反向调用）

### 4.1 Desktop 平台：JVM 反向调用

Desktop 平台上 Compose 运行在 JVM，C++ 需要通过 JNI 调用 Kotlin 函数。但对于 KrKr2 这样的场景，更简单的方式是使用**回调函数指针**：C++ 持有一个由 Kotlin 注册的回调，需要时直接调用。

```cpp
// cpp/core/environ/ui/compose/ComposeUIBridge.h
// C++ → Compose 反向调用桥接
#pragma once
#include <functional>
#include <string>

namespace krkr2 {
namespace ui {
namespace compose_backend {

/**
 * Compose UI 事件回调函数类型
 * （由 Kotlin 端注册，C++ 端调用）
 */
using ShowMainMenuFn   = std::function<void()>;
using HideMainMenuFn   = std::function<void()>;
using ShowConfigFn     = std::function<void(const std::string& initial_tab)>;
using ShowToastFn      = std::function<void(const std::string& msg, int type)>;

/**
 * ComposeUIBridge — C++ 端持有的 Compose UI 控制接口
 *
 * Kotlin/Compose 初始化时，向 C++ 注册这些回调函数。
 * C++ 需要显示/隐藏 UI 时，通过这些函数通知 Compose。
 */
class ComposeUIBridge {
public:
    static ComposeUIBridge& instance() {
        static ComposeUIBridge s_instance;
        return s_instance;
    }
    
    // --- Kotlin 调用这些函数注册回调 ---
    void registerShowMainMenu(ShowMainMenuFn fn)  { show_main_menu_ = std::move(fn); }
    void registerHideMainMenu(HideMainMenuFn fn)  { hide_main_menu_ = std::move(fn); }
    void registerShowConfig  (ShowConfigFn fn)    { show_config_     = std::move(fn); }
    void registerShowToast   (ShowToastFn fn)     { show_toast_      = std::move(fn); }
    
    // --- C++ 调用这些函数控制 Compose UI ---
    void showMainMenu() {
        if (show_main_menu_) show_main_menu_();
    }
    
    void hideMainMenu() {
        if (hide_main_menu_) hide_main_menu_();
    }
    
    void showConfigDialog(const std::string& tab = "general") {
        if (show_config_) show_config_(tab);
    }
    
    void showToast(const std::string& message, int type = 0) {
        if (show_toast_) show_toast_(message, type);
    }

private:
    ComposeUIBridge() = default;
    
    ShowMainMenuFn  show_main_menu_;
    HideMainMenuFn  hide_main_menu_;
    ShowConfigFn    show_config_;
    ShowToastFn     show_toast_;
};

} // namespace compose_backend
} // namespace ui
} // namespace krkr2
```

### 4.2 通过 C API 导出 Kotlin 注册函数

```c
// C API 头文件（供 Kotlin cinterop 使用）
// krkr2_compose_callbacks_api.h

#pragma once
#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

/** Compose → C++ 回调：游戏选择对话框 */
typedef char* (*OnGameSelectFn)(void* user_data);

/** Compose → C++ 回调：设置对话框 */
typedef void (*OnShowConfigFn)(const char* tab, void* user_data);

/** Compose → C++ 回调：退出 */
typedef void (*OnQuitFn)(void* user_data);

/**
 * 注册 Compose UI 发出的事件回调
 * 由 Kotlin 在 Compose 初始化时调用，向 C++ 传入 lambda 的函数指针
 */
void krkr2_compose_register_callbacks(
    OnGameSelectFn  on_game_select,
    OnShowConfigFn  on_show_config,
    OnQuitFn        on_quit,
    void*           user_data
);

/**
 * C++ → Compose：通知显示主菜单
 * Kotlin 端必须在 Compose 准备好后调用 krkr2_compose_set_ready(true)
 */
void krkr2_compose_show_main_menu(void);

/**
 * C++ → Compose：通知隐藏主菜单
 */
void krkr2_compose_hide_main_menu(void);

/**
 * C++ → Compose：显示 Toast
 */
void krkr2_compose_show_toast(const char* message, int type);

/**
 * C++ → Compose：设置就绪状态
 */
void krkr2_compose_set_ready(bool ready);

/**
 * 查询 Compose UI 是否就绪
 */
bool krkr2_compose_is_ready(void);

#ifdef __cplusplus
} // extern "C"
#endif
```

```cpp
// C API 实现
// krkr2_compose_callbacks_api.cpp
#include "krkr2_compose_callbacks_api.h"
#include "ComposeUIBridge.h"
#include <functional>
#include <atomic>

static std::atomic<bool> g_compose_ready{false};

// 保存注册的回调（原始 C 函数指针 + user_data）
static OnGameSelectFn  g_on_game_select = nullptr;
static OnShowConfigFn  g_on_show_config = nullptr;
static OnQuitFn        g_on_quit        = nullptr;
static void*           g_user_data      = nullptr;

extern "C" {

void krkr2_compose_register_callbacks(
    OnGameSelectFn on_game_select,
    OnShowConfigFn on_show_config,
    OnQuitFn       on_quit,
    void*          user_data)
{
    g_on_game_select = on_game_select;
    g_on_show_config = on_show_config;
    g_on_quit        = on_quit;
    g_user_data      = user_data;
}

void krkr2_compose_show_main_menu(void) {
    krkr2::ui::compose_backend::ComposeUIBridge::instance().showMainMenu();
}

void krkr2_compose_hide_main_menu(void) {
    krkr2::ui::compose_backend::ComposeUIBridge::instance().hideMainMenu();
}

void krkr2_compose_show_toast(const char* message, int type) {
    if (message) {
        krkr2::ui::compose_backend::ComposeUIBridge::instance()
            .showToast(message, type);
    }
}

void krkr2_compose_set_ready(bool ready) {
    g_compose_ready.store(ready);
}

bool krkr2_compose_is_ready(void) {
    return g_compose_ready.load();
}

} // extern "C"

// C++ 内部：触发 Kotlin 回调
namespace krkr2 {
namespace ui {
namespace compose_backend {

// 供 C++ 代码调用：请求 Kotlin 显示游戏选择对话框
std::string requestGameSelect() {
    if (!g_on_game_select) return "";
    char* result = g_on_game_select(g_user_data);
    if (!result) return "";
    std::string s(result);
    // 注意：result 由 Kotlin 分配，用完后 Kotlin 负责释放
    // （或约定使用 krkr2_free_string）
    return s;
}

} // namespace compose_backend
} // namespace ui
} // namespace krkr2
```

### 4.3 Kotlin 侧：注册回调到 C++

```kotlin
// kotlin_module/src/commonMain/kotlin/ComposeCallbackRegistrar.kt
import krkr2_compose.*
import kotlinx.cinterop.*

/**
 * Compose 初始化时，向 C++ 注册所有事件回调
 * 在 Compose 应用启动后立即调用
 */
object ComposeCallbackRegistrar {
    
    // 保存回调 lambda 的 StableRef（防止 GC 回收）
    private var stableRef: StableRef<CallbackContext>? = null
    
    data class CallbackContext(
        val onGameSelect: () -> String?,
        val onShowConfig: (String) -> Unit,
        val onQuit: () -> Unit
    )
    
    fun register(
        onGameSelect: () -> String?,
        onShowConfig: (String) -> Unit,
        onQuit: () -> Unit,
    ) {
        val context = CallbackContext(onGameSelect, onShowConfig, onQuit)
        val ref = StableRef.create(context)
        stableRef = ref
        
        krkr2_compose_register_callbacks(
            on_game_select = staticCFunction { userData ->
                val ctx = userData!!.asStableRef<CallbackContext>().get()
                val result = ctx.onGameSelect()
                if (result == null) return@staticCFunction null
                // 分配 C 字符串返回给 C++（C++ 负责调用 krkr2_free_string 释放）
                val cStr = result.encodeToByteArray()
                val buf = nativeHeap.allocArray<ByteVar>(cStr.size + 1)
                cStr.forEachIndexed { i, b -> buf[i] = b }
                buf[cStr.size] = 0  // null 终止符
                buf
            },
            on_show_config = staticCFunction { tab, userData ->
                val ctx = userData!!.asStableRef<CallbackContext>().get()
                ctx.onShowConfig(tab?.toKString() ?: "general")
            },
            on_quit = staticCFunction { userData ->
                val ctx = userData!!.asStableRef<CallbackContext>().get()
                ctx.onQuit()
            },
            user_data = ref.asCPointer()
        )
        
        // 通知 C++ Compose UI 已就绪
        krkr2_compose_set_ready(true)
    }
    
    fun unregister() {
        krkr2_compose_set_ready(false)
        stableRef?.dispose()
        stableRef = null
    }
}
```

---

## 五、双端代码共享策略

### 5.1 平台无关 vs 平台特定代码划分

```
kotlin_module/src/
├── commonMain/kotlin/       ← 所有平台共享（~70% 代码量）
│   ├── ui/
│   │   ├── KrKr2MainMenu.kt        ← @Composable UI（无平台依赖）
│   │   ├── KrKr2ConfigDialog.kt    ← 设置对话框
│   │   └── KrKr2Theme.kt           ← 主题定义
│   ├── KrKr2UIManager.kt           ← C API 封装
│   ├── EventBridge.kt              ← C++ → Kotlin 事件回调
│   └── ComposeCallbackRegistrar.kt ← Kotlin → C++ 回调注册
│
├── desktopMain/kotlin/      ← Desktop 特定（JVM，~15%）
│   ├── Main.kt                     ← Desktop 入口
│   └── PlatformFileDialog.kt       ← 原生文件对话框（JFileChooser / 系统API）
│
└── androidMain/kotlin/      ← Android 特定（JVM + Android SDK，~15%）
    ├── KrKr2ComposeOverlay.kt      ← ComposeView 叠加层
    └── KrKr2JNI.kt                 ← JNI 桥接
```

### 5.2 expect/actual 机制处理平台差异

Compose Multiplatform 使用 `expect`/`actual` 处理平台特定实现：

```kotlin
// commonMain：声明期望（expect）
expect object PlatformFileDialog {
    /** 显示原生文件选择对话框，返回选中路径 */
    suspend fun show(filter: String): String?
}

// desktopMain：Desktop 实现（actual）
actual object PlatformFileDialog {
    actual suspend fun show(filter: String): String? {
        // Desktop 使用 java.awt.FileDialog
        return kotlinx.coroutines.withContext(
            kotlinx.coroutines.Dispatchers.Main
        ) {
            val dialog = java.awt.FileDialog(null as java.awt.Frame?)
            dialog.file = filter
            dialog.isVisible = true
            dialog.file?.let { "${dialog.directory}$it" }
        }
    }
}

// androidMain：Android 实现（actual）
actual object PlatformFileDialog {
    actual suspend fun show(filter: String): String? {
        // Android 使用 SAF（Storage Access Framework）
        // 此处需要 ActivityResult API，实际实现需要 Activity 引用
        return null  // 简化：实际通过 ActivityResultLauncher 实现
    }
}
```

---

## 六、动手实践

### 实践 6.1：在 Desktop 独立运行 Compose UI

```bash
# 在 kotlin_module/ 目录下
cd krkr2/kotlin_module

# 运行 Compose Desktop 应用（独立窗口）
./gradlew :desktop:run

# 期望：弹出 KrKr2 主菜单窗口，显示"选择游戏"和"设置"按钮
```

### 实践 6.2：测试 C++ ↔ Kotlin 回调链

```kotlin
// src/desktopTest/kotlin/CallbackTest.kt
import krkr2_compose.*
import kotlin.test.*

class CallbackTest {
    
    @Test
    fun testCallbackRegistration() {
        var gameSelectCalled = false
        var configTabReceived = ""
        
        ComposeCallbackRegistrar.register(
            onGameSelect = {
                gameSelectCalled = true
                "/test/game/path"
            },
            onShowConfig = { tab ->
                configTabReceived = tab
            },
            onQuit = {}
        )
        
        // 验证 Compose 已就绪
        assertTrue(krkr2_compose_is_ready(), "Compose should be ready after registration")
        
        // 模拟 C++ 侧触发"显示设置"（通过 C 函数调用回调）
        // 注意：真实场景中这由 C++ 触发，这里仅验证注册状态
        
        ComposeCallbackRegistrar.unregister()
        assertFalse(krkr2_compose_is_ready(), "Compose should not be ready after unregister")
    }
}
```

---

## 对照项目源码

```
krkr2/
├── cpp/core/environ/ui/compose/
│   ├── ComposeUIBridge.h              ← C++ → Compose 回调桥接头文件
│   ├── krkr2_compose_callbacks_api.h  ← C API（供 Kotlin cinterop 使用）
│   └── krkr2_compose_callbacks_api.cpp← C API 实现
└── kotlin_module/src/
    ├── commonMain/kotlin/
    │   ├── ui/
    │   │   ├── KrKr2MainMenu.kt       ← 跨平台 Compose UI
    │   │   └── KrKr2ConfigDialog.kt   ← 设置对话框
    │   └── ComposeCallbackRegistrar.kt← Kotlin → C++ 回调注册
    ├── desktopMain/kotlin/
    │   └── Main.kt                    ← Desktop 入口
    └── androidMain/kotlin/
        ├── KrKr2ComposeOverlay.kt     ← Android ComposeView 叠加
        └── KrKr2JNI.kt               ← Android JNI 桥接
```

---

## 本节小结

- Compose Multiplatform 的 UI 代码（`@Composable` 函数）可在 Desktop 和 Android 之间共享约 70%
- Desktop 平台使用 `ComposeWindow`/`ComposePanel`（基于 JVM + Skiko），Android 使用 `ComposeView`（嵌入 View 层级）
- C++ 调用 Kotlin 的推荐方式：Kotlin 在初始化时通过 `staticCFunction` 注册 C 函数指针到 C++，C++ 保存这些指针并在需要时调用
- `expect`/`actual` 机制用于处理文件对话框等平台特定 API，保持 commonMain 代码的平台无关性
- Compose 应用与 KrKr2 主循环在不同线程运行，回调必须是线程安全的

---

## 练习题与答案

**题 1：** `staticCFunction { ... }` 与普通的 Kotlin lambda 有什么区别？为什么传给 C 函数指针时必须用 `staticCFunction`？

<details>
<summary>查看答案</summary>

**普通 Kotlin lambda** 是一个对象（实现了函数接口），存储在 KN 堆上，有地址但地址是 Kotlin 对象地址，不是 C 调用约定的函数地址。

**`staticCFunction`** 生成的是一个真正的 C 函数指针（机器码地址），符合 C 调用约定（cdecl/stdcall），可直接赋值给 C 的 `typedef void(*FnPtr)()`。

关键区别：
- `staticCFunction` 的参数类型必须是 C 兼容类型（CPointer、基本类型等），不能是 Kotlin 类
- `staticCFunction` 内部**不能捕获外部变量**（因为 C 函数指针没有闭包语义）。需要传递状态时，必须通过 `user_data`（void* 参数）+ `StableRef` 来实现
- 普通 lambda 可以捕获外部变量，但无法转为 C 函数指针

错误示例：
```kotlin
val multiplier = 2
val fn: CPointer<CFunction<(Int) -> Int>> = staticCFunction { x ->
    x * multiplier  // ❌ 编译错误：不能捕获 multiplier
}
```

正确做法：通过 user_data 传递 multiplier
```kotlin
data class Context(val multiplier: Int)
val ctx = StableRef.create(Context(2))
val fn: CPointer<CFunction<(Int, COpaquePointer?) -> Int>> = 
    staticCFunction { x, userData ->
        val context = userData!!.asStableRef<Context>().get()
        x * context.multiplier
    }
```

</details>

---

**题 2：** Compose Desktop 窗口（ComposeWindow）与 Cocos2d-x OpenGL 窗口在 Windows 平台上并存时，可能有哪些兼容性问题？

<details>
<summary>查看答案</summary>

主要兼容性问题：

1. **OpenGL 上下文冲突**：Cocos2d-x 使用 GLFW 或 SDL 管理 OpenGL 上下文，Compose Desktop 使用 Skiko（通过 Swing/AWT 的 JOGL 或内置 OpenGL 上下文）。两者如果在同一线程操作 GL 可能互相干扰。  
   **解决方案**：Compose Desktop 在独立线程（EDT - Event Dispatch Thread）运行，与 Cocos2d-x 的渲染线程天然隔离。

2. **消息泵冲突**：Windows 上每个窗口都有消息队列（MSG），Cocos2d-x 和 AWT 各自有消息泵（GetMessage/DispatchMessage 循环）。并发消息处理可能导致某些键盘/鼠标事件被"吃掉"。  
   **解决方案**：使用独立进程运行 Compose UI（最干净，但 IPC 成本高），或确保两个消息泵在各自线程上运行。

3. **Z-Order 问题**：Compose Desktop 窗口和 Cocos2d-x 窗口的层叠顺序（z-order）在用户切换程序时可能错乱。  
   **解决方案**：Compose 窗口设置 `alwaysOnTop = true`（覆盖层模式），或将其设为 Cocos2d-x 窗口的子窗口（`WS_CHILD` 样式）。

4. **DPI 缩放不一致**：Cocos2d-x 和 Compose Desktop 对 HiDPI/DPI 缩放的处理方式不同，可能导致两个窗口的视觉大小不一致。  
   **解决方案**：统一读取系统 DPI 值（`GetDpiForWindow`），手动计算各自的 pixel ratio。

</details>

---

**题 3：** 在 Android 上，`ComposeView` 叠加在 `GLSurfaceView` 上方时，触摸事件会如何分发？如何确保触摸事件正确地传递到 Cocos2d-x 游戏层？

<details>
<summary>查看答案</summary>

**Android 触摸事件分发机制：** 触摸事件从 Activity 的 `decorView` 开始，自上而下（ViewGroup → View）调用 `dispatchTouchEvent`。如果上层 View（ComposeView）消费了事件（`onTouchEvent` 返回 true），下层 View（GLSurfaceView）不会收到事件。

**问题所在：** ComposeView 默认会消费它接受到的所有触摸事件，导致 KrKr2 的游戏层无法收到玩家触摸。

**解决方案：**

1. **状态感知方案**（推荐）：
   ```kotlin
   // 在游戏进行时，隐藏 ComposeView（或将其设置为 GONE）
   // 这样触摸事件直接到达 GLSurfaceView
   composeView.visibility = View.GONE  // 游戏进行中
   composeView.visibility = View.VISIBLE  // 显示菜单时
   ```

2. **透传方案**（叠加层需要半透明时）：
   ```kotlin
   // 自定义 ComposeView，重写 dispatchTouchEvent
   class PassthroughComposeView(context: Context) : ComposeView(context) {
       var isMenuVisible = false
       
       override fun dispatchTouchEvent(ev: MotionEvent): Boolean {
           if (!isMenuVisible) return false  // 不消费，传递给下层
           return super.dispatchTouchEvent(ev)
       }
   }
   ```

3. **区域判断方案**：
   ```kotlin
   // 只在触摸点在 UI 元素区域内时才消费事件
   // 通过 HitTestResult 判断触摸是否命中 Compose 组件
   ```

最佳实践：游戏进行中完全隐藏 ComposeView，只在需要显示 UI 时才显示，避免事件穿透带来的复杂性。

</details>

---

## 下一步

本节完成了 Compose Multiplatform 双端集成和 C++ ↔ Kotlin 反向调用的实现。下一章 [05-渲染引擎与UI的桥接](../05-渲染引擎与UI的桥接/01-纹理共享方案与外部纹理.md) 将深入探讨 Flutter/Compose 渲染输出与 Cocos2d-x 纹理系统之间的零拷贝共享方案。
