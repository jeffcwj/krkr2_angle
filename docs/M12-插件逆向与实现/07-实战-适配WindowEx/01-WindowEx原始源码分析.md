# 01 — WindowEx 原始源码分析

> **所属模块：** M12-插件逆向与实现
> **前置知识：** [01-KiriKiri插件接口分析](../01-KiriKiri插件接口分析/)、[M09-插件系统与开发](../../M09-插件系统与开发/)
> **预计阅读时间：** 约 45 分钟
> **源码来源：** [https://github.com/wtnbgo/windowEx](https://github.com/wtnbgo/windowEx)（作者 wtnbgo，不是 wamsoft）

## 本节目标

读完本节后，你将能够：

1. 理解 windowEx 插件的完整架构——三大结构体（WindowEx / MenuItemEx / OverlayBitmap）的职责分工
2. 读懂 ncbind 框架的完整 TJS 注册流程——RawCallback/Variant/Method 三种注册方式的适用场景
3. 掌握 KiriKiri 消息接收器机制——`registerMessageReceiver` 钩子 + `onMessage` 消息分发的完整实现路径
4. 整理出 windowEx 在 Win32 上依赖的全部 API 清单，为后续跨平台适配奠定认知基础
5. 能够独立分析一个陌生的 ncbind 插件源码，识别其核心结构和 TJS 暴露接口

## 术语预览

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 消息接收器 | Message Receiver | KiriKiri 引擎对外暴露的钩子机制，允许 C++ 插件拦截 Win32 窗口消息 |
| WM_* 消息 | Windows Message | Windows 操作系统以整数编码的事件通知，如 WM_SIZE=0x0005 表示窗口大小改变 |
| HWND | Window Handle | Windows 系统为每个窗口分配的不透明句柄（32/64 位整数），通过它才能调用 Win32 窗口 API |
| ncbind | ncbind | KrKr2 使用的 C++ 到 TJS2 绑定框架，用宏将 C++ 类自动注册为脚本对象 |
| RawCallback | Raw Callback | ncbind 中最底层的注册方式，函数签名固定为 `(tTJSVariant*, tjs_int, tTJSVariant**, iTJSDispatch2*)` |
| GWL_STYLE | GetWindowLong Style | Win32 API 常量，表示获取/设置窗口样式位掩码，如 WS_MAXIMIZEBOX 控制最大化按钮 |
| OverlayBitmap | OverlayBitmap | windowEx 内部的透明子窗口子类，用于在游戏窗口上叠加位图（常用于自定义标题栏图像） |
| DWM | Desktop Window Manager | Windows Vista 起的窗口合成器（桌面窗口管理器），提供 Aero 玻璃效果和窗口圆角等特性 |

---

## 1. 插件概述

windowEx 是 KiriKiri2/KiriKiriZ 引擎的一个扩展插件，由 **wtnbgo**（GitHub：https://github.com/wtnbgo/windowEx）开发，MIT 许可证。它的核心目标是**把 Win32 窗口管理能力暴露给 TJS2 脚本层**，让游戏脚本能够：

- 控制窗口的最大化/最小化/还原
- 动态修改系统菜单（右键标题栏的菜单）
- 获取/设置窗口位置和大小
- 设置窗口图标（包括从外部文件加载）
- 监听扩展窗口事件（最小化、移动、改变显示器 DPI 等）
- 在游戏窗口上叠加自定义位图（OverlayBitmap 功能）
- 注册全局热键（RegisterHotKey）
- 监听设备变化通知（U 盘插拔等）
- 控制 IME（输入法编辑器）状态
- 控制 Windows 11 的窗口圆角偏好（DWM API）

> **为什么游戏需要这些功能？**
>
> KiriKiri 原生的 `Window` 对象已经提供了基础的窗口控制（如 `Window.innerWidth`、`Window.caption`），但许多游戏需要更精细的控制，比如：禁止玩家手动调整窗口大小、在窗口标题栏显示游戏 Logo 位图、当切换到其他应用时触发特定事件等。这些功能超出了 KiriKiri 核心的职责范围，因此通过 windowEx 插件扩展。

### 原始源码文件结构

```
wtnbgo/windowEx/
└── main.cpp          # 全部实现（单文件，1327+ 行）
    ├── 工具函数       # GetGlobalRecursive, CheckKirikiri2
    ├── struct WindowEx     # 主插件结构体（核心逻辑）
    │   ├── 工具方法   # GetInstance, GetHWND, SetRect, GetRect
    │   ├── 消息处理   # receiver(), regist(), onMessage()
    │   ├── TJS 方法   # minimize, maximize, getWindowRect 等
    │   ├── 成员变量   # self, menuex, sysMenuModified, ovbmp 等
    │   └── 保护方法   # _setOverlayBitmap, _getNotificationVariant
    ├── struct MenuItemEx   # 系统菜单扩展（MenuItemEx 子功能）
    ├── class OverlayBitmap # Win32 透明子窗口（层叠位图）
    ├── NCB_ATTACH_CLASS_WITH_HOOK   # TJS 类注册宏
    ├── MenuItemEx 注册
    └── System/Debug 扩展方法注册
```

---

## 2. 核心结构体：WindowEx

### 2.1 成员变量全景

```cpp
// 来自 stuffs/wtnbgo_windowex_src/main.cpp，约第 965-1000 行

struct WindowEx {
    // ------ 核心指针 ------
    iTJSDispatch2 *self;         // 指向关联的 TJS Window 对象
    iTJSDispatch2 *menuex;       // MenuItemEx 对象（自定义菜单项）
    iTJSDispatch2 *sysMenuModified; // 记录修改后的系统菜单
    iTJSDispatch2 *sysMenuModMap;   // 菜单 ID 映射表

    // ------ Win32 句柄 ------
    HWND cachedHWND;             // 缓存的窗口句柄（避免每次从 TJS 读取）
    HMENU sysMenu;               // 系统菜单句柄（右键标题栏）
    HICON externalIcon;          // 从外部文件加载的图标句柄
    HDEVNOTIFY deviceNotify;     // 设备通知注册句柄

    // ------ 行为标志 ------
    bool hasResizing;            // 是否注册了 onResizing 事件处理器
    bool hasMoving;              // 是否注册了 onMoving 事件处理器
    bool hasMove;                // 是否注册了 onMove 事件处理器
    bool hasNcMsMove;            // 是否注册了 onNcMouseMove 事件处理器
    bool disableResize;          // 禁止用户拖动边框改变大小
    bool disableMove;            // 禁止用户拖动标题栏移动窗口
    bool enableNCMEvent;         // 启用非客户区鼠标事件
    bool enableWinMsgHook;       // 是否开启 Windows 消息钩子

    // ------ OverlayBitmap ------
    OverlayBitmap *ovbmp;        // 叠加位图对象（可为 NULL）

    // ------ 消息钩子位图 ------
    DWORD bitHooks[0x400/32];    // 位集合，记录需要钩取的 WM_* 消息编号
                                 // (0x400 = 1024 个消息，每个 DWORD 存 32 个标志位)
};
```

> **关键设计**：`self` 成员是 `iTJSDispatch2*`，指向 TJS 层的 `Window` 对象。这使得 C++ 代码能够反向调用 TJS 脚本定义的事件处理函数（`onMinimize`、`onMaximize` 等），形成双向通信。

### 2.2 构造函数与析构函数

构造函数完成三件事：保存 `self` 指针、调用 `regist(true)` 注册消息接收器、初始化 `bitHooks`。

```cpp
// 来自 stuffs/wtnbgo_windowex_src/main.cpp，第 886-915 行

WindowEx(iTJSDispatch2 *obj)
    : self(obj), menuex(0),
      sysMenuModified(0), sysMenuModMap(0),
      cachedHWND(0),
      sysMenu(0),
      externalIcon(0),
      deviceNotify(0),
      hasResizing(false),
      hasMoving(false),
      hasMove(false),
      disableResize(false),
      disableMove(false),
      enableNCMEvent(false),
      enableWinMsgHook(false),
      ovbmp(NULL)
{
    cachedHWND = GetHWND(self);  // 从 TJS 读取 HWND 属性并缓存
    regist(true);                // 向引擎注册消息接收器（核心！）
    setMessageHookAll(false);    // 初始化消息钩子位集合全为 0
}

~WindowEx() {
    // 清理所有 Win32 资源
    if (deviceNotify)    ::UnregisterDeviceNotification(deviceNotify);
    if (menuex)          menuex         ->Release();    // 释放 TJS 对象引用
    if (sysMenuModified) sysMenuModified->Release();
    if (externalIcon)    ::DestroyIcon(externalIcon);   // 释放图标句柄
    resetSystemMenu();         // 恢复系统菜单到原始状态
    deleteOverlayBitmap();     // 销毁叠加位图子窗口
    regist(false);             // 从引擎注销消息接收器（必须！防止悬空指针）
}
```

---

## 3. 消息接收器架构（核心机制）

### 3.1 什么是消息接收器

KiriKiri2/Z 引擎提供了一个扩展点 `Window.registerMessageReceiver`，允许 C++ 插件注册一个回调函数。每当游戏窗口收到 Win32 消息（如鼠标点击、键盘事件、窗口移动等），引擎会在处理之前先调用所有注册的接收器函数，让插件有机会拦截或响应消息。

这是整个 windowEx 插件的**核心依赖**。没有消息接收器，`onMinimize`、`onMoving` 等所有扩展事件都无法工作。

### 3.2 注册流程

```cpp
// 来自 stuffs/wtnbgo_windowex_src/main.cpp，第 870-883 行

// 步骤1：定义静态回调函数（__stdcall 调用约定是 Win32 标准）
static bool __stdcall receiver(void *userdata, tTVPWindowMessage *mes) {
    // userdata 就是 self（iTJSDispatch2* 强转为 void*）
    WindowEx *inst = GetInstance((iTJSDispatch2*)userdata);
    // 找到 C++ 实例后，分发到 onMessage()
    return inst ? inst->onMessage(mes) : false;
}

// 步骤2：向引擎注册/注销接收器
void regist(bool en) {
    // wrmRegister = 注册, wrmUnregister = 注销（来自 tp_stub.h 枚举）
    tTJSVariant mode(en ? wrmRegister : wrmUnregister);
    // 把 C++ 函数指针包装为整数（TJS Variant 可存任意整数）
    tTJSVariant func((tTVInteger)(&receiver));
    // userdata 参数：把 self 指针也以整数形式传入
    tTJSVariant data((tTVInteger)(self));

    tTJSVariant rslt, *params[] = { &mode, &func, &data };
    // 调用 TJS Window 对象的 registerMessageReceiver 方法
    // 相当于 TJS 脚本里写: window.registerMessageReceiver(wrmRegister, func, data)
    self->FuncCall(0, TJS_W("registerMessageReceiver"), 0, &rslt, 3, params, self);
}
```

> **设计要点**：
> 1. `receiver` 是 `static` 函数——Win32 回调不能有 `this` 指针，必须用静态函数
> 2. `userdata` 机制解决"静态函数找实例"的问题：注册时把 `self`（TJS 对象指针）作为 `userdata` 传入，回调时再通过 `GetInstance` 从 TJS 对象恢复 C++ 实例指针
> 3. `~WindowEx()` 必须调用 `regist(false)`——否则对象销毁后引擎仍然调用已释放内存的 `receiver`，造成崩溃

### 3.3 onMessage 消息分发（150+ 行，DLL 中最大函数）

`onMessage` 是整个插件逻辑密度最高的函数，处理 20+ 种 Win32 消息类型：

```cpp
// 来自 stuffs/wtnbgo_windowex_src/main.cpp，约第 700-860 行（节选关键分支）

bool onMessage(tTVPWindowMessage *mes) {
    HWND hwnd = cachedHWND;

    switch (mes->Msg) {
    // --- 窗口大小改变 ---
    case WM_SIZE:
        switch (mes->WParam) {
        case SIZE_MINIMIZED:
            callback(EXEV_MINIMIZE);   // 触发 TJS onMinimize 事件
            break;
        case SIZE_MAXIMIZED:
            if (!callback(EXEV_QUERYMAX)) // 如果 onMaximizeQuery 返回 false，阻止最大化
                callback(EXEV_MAXIMIZE);
            break;
        case SIZE_RESTORED:
            callback(EXEV_SHOW);
            break;
        }
        return false; // 不拦截，让引擎继续处理

    // --- 系统命令（最大化/最小化/关闭按钮点击） ---
    case WM_SYSCOMMAND:
        if ((mes->WParam & 0xFFF0) == SC_MINIMIZE) {
            callback(EXEV_MINIMIZE);
        }
        break;

    // --- 非客户区鼠标命中测试（决定鼠标在标题栏/边框哪个区域） ---
    case WM_NCHITTEST:
        if (disableResize) {
            // 禁止调整大小：把所有边框区域都重映射为客户区
            mes->Result = ::DefWindowProc(hwnd, mes->Msg, mes->WParam, mes->LParam);
            if (mes->Result >= HTLEFT && mes->Result <= HTBOTTOMRIGHT)
                mes->Result = HTCLIENT; // 覆盖返回值
            return true;  // 拦截：返回 true 告诉引擎不要继续处理
        }
        if (disableMove) {
            mes->Result = ::DefWindowProc(hwnd, mes->Msg, mes->WParam, mes->LParam);
            if (mes->Result == HTCAPTION) mes->Result = HTCLIENT;
            return true;
        }
        break;

    // --- 窗口正在移动 ---
    case WM_MOVING:
        if (hasMoving) {
            RECT *r = (RECT*)mes->LParam;
            // 触发 onMoving(x, y, w, h) 回调
            callback(EXEV_MOVING, r->left, r->top, r->right-r->left, r->bottom-r->top);
        }
        break;

    // --- DPI 改变（多显示器、高分屏场景） ---
    case WM_DPICHANGED:
        callback(EXEV_DPICHANGE);
        break;

    // --- 全局热键触发 ---
    case WM_HOTKEY:
        {
            WORD mod = LOWORD(mes->LParam);
            int id = (int)mes->WParam;
            int ss = 0;
            if (mod & MOD_ALT)     ss |= TVP_SS_ALT;
            if (mod & MOD_CONTROL) ss |= TVP_SS_CTRL;
            if (mod & MOD_SHIFT)   ss |= TVP_SS_SHIFT;
            if (callback(EXEV_HOTKEY, id, ss)) return true;
        }
        break;
    }
    return false;  // 默认：不拦截消息
}
```

> **`tTVPWindowMessage` 结构**（来自 tp_stub.h）：
> ```cpp
> struct tTVPWindowMessage {
>     tjs_uint   Msg;    // 消息编号（如 WM_SIZE = 0x0005）
>     tjs_uint64 WParam; // 消息参数1（含义因消息而异）
>     tjs_int64  LParam; // 消息参数2
>     tjs_int64  Result; // 处理结果（可被插件覆盖）
> };
> ```

---

## 4. 完整 TJS 暴露接口清单

下表列出原始 Win32 版本中所有向 TJS 层暴露的方法和属性（共 30+ 个）：

| 名称 | 类型 | Win32 实现 | 说明 |
|------|------|-----------|------|
| `minimize()` | 方法 | `PostMessage(hwnd, WM_SYSCOMMAND, SC_MINIMIZE)` | 最小化窗口 |
| `maximize()` | 方法 | `PostMessage(hwnd, WM_SYSCOMMAND, SC_MAXIMIZE)` | 最大化窗口 |
| `showRestore()` | 方法 | `PostMessage(hwnd, WM_SYSCOMMAND, SC_RESTORE)` | 恢复窗口 |
| `focusMenuByKey()` | 方法 | `PostMessage(hwnd, WM_SYSCOMMAND, SC_KEYMENU)` | 键盘激活菜单 |
| `resetWindowIcon()` | 方法 | `PostMessage(hwnd, WM_SETICON)` | 重置窗口图标 |
| `setWindowIcon(path)` | 方法 | `ExtractIconW` + `PostMessage(WM_SETICON)` | 设置外部图标文件 |
| `setWindowCornerPreference(val)` | 方法 | `DwmSetWindowAttribute(DWMWA_WINDOW_CORNER_PREFERENCE)` | Win11 圆角控制 |
| `getWindowRect()` | 方法 | `GetWindowRect(hwnd, &rect)` | 获取窗口外边框矩形 |
| `getClientRect()` | 方法 | `GetClientRect` + `ClientToScreen` | 获取客户区矩形（屏幕坐标） |
| `setClientRect(dict)` | 方法 | `SetWindowPos` | 按客户区大小设置窗口位置 |
| `getNormalRect()` | 方法 | `GetWindowPlacement` + `GetMonitorInfoW` | 获取非最大化时的矩形 |
| `setOverlayBitmap(layer)` | 方法 | `CreateWindowExW` + `CreateDIBSection` + `BitBlt` | 叠加位图 |
| `bringTo(target)` | 方法 | `SetWindowPos(hwnd, HWND_TOP)` 等 | 调整 Z 顺序 |
| `sendToBack()` | 方法 | `SetWindowPos(hwnd, HWND_BOTTOM)` | 置于最后 |
| `setMessageHook(on, msg)` | 方法 | 更新 `bitHooks` 位集合 | 开启特定消息钩子 |
| `nonClientHitTest(x, y)` | 方法 | 内部坐标计算 | 非客户区命中测试 |
| `resetExSystemMenu()` | 方法 | `DeleteMenu` / 恢复系统菜单 | 重置扩展系统菜单 |
| `registerDeviceChange(guid)` | 方法 | `RegisterDeviceNotificationW` | 注册设备变化通知 |
| `registerHotKey(id, key, mod)` | 方法 | `RegisterHotKey` | 注册全局热键 |
| `acquireImeControl(on)` | 方法 | `ImmAssociateContextEx` | 控制 IME 输入法 |
| `resetImeContext()` | 方法 | `ImmAssociateContextEx(NULL, IACE_DEFAULT)` | 重置 IME 上下文 |
| `maximizeBox` | 属性 R/W | `GetWindowLong/SetWindowLong(GWL_STYLE, WS_MAXIMIZEBOX)` | 最大化按钮 |
| `minimizeBox` | 属性 R/W | `GetWindowLong/SetWindowLong(GWL_STYLE, WS_MINIMIZEBOX)` | 最小化按钮 |
| `maximized` | 属性 R/W | `IsZoomed(hwnd)` | 是否处于最大化状态 |
| `minimized` | 属性 R/W | `IsIconic(hwnd)` | 是否处于最小化状态 |
| `disableResize` | 属性 R/W | `WM_NCHITTEST` 拦截 | 禁止调整大小 |
| `disableMove` | 属性 R/W | `WM_NCHITTEST` 拦截 + 菜单禁用 | 禁止移动窗口 |
| `enableNCMouseEvent` | 属性 R/W | 控制非客户区鼠标事件转发 | 非客户区鼠标事件 |
| `exSystemMenu` | 属性 R/W | `GetSystemMenu` + `InsertMenuItemW` | 自定义系统菜单 |

### 扩展事件（通过 TJS 回调触发）

| 事件名 | 触发时机 | WM_* 来源 |
|--------|---------|-----------|
| `onMinimize` | 窗口最小化 | `WM_SIZE` (SIZE_MINIMIZED) |
| `onMaximize` | 窗口最大化 | `WM_SIZE` (SIZE_MAXIMIZED) |
| `onMaximizeQuery` | 最大化前询问（返回 true 阻止） | `WM_SIZE` |
| `onShow` | 窗口还原显示 | `WM_SIZE` (SIZE_RESTORED) |
| `onHide` | 窗口隐藏 | `WM_SHOWWINDOW` |
| `onResizing` | 窗口正在调整大小 | `WM_SIZING` |
| `onMoving` | 窗口正在移动（每帧触发） | `WM_MOVING` |
| `onMove` | 窗口移动完成 | `WM_MOVE` |
| `onMoveSizeBegin` | 开始拖动 | `WM_ENTERSIZEMOVE` |
| `onMoveSizeEnd` | 结束拖动 | `WM_EXITSIZEMOVE` |
| `onDPIChanged` | DPI 改变（外接高分辨率显示器） | `WM_DPICHANGED` |
| `onDisplayChanged` | 显示分辨率改变 | `WM_DISPLAYCHANGE` |
| `onDeviceChanged` | USB/设备插拔 | `WM_DEVICECHANGE` |
| `onEnterMenuLoop` | 进入菜单模式 | `WM_ENTERMENULOOP` |
| `onExitMenuLoop` | 退出菜单模式 | `WM_EXITMENULOOP` |
| `onActivateChanged` | 窗口激活/失焦 | `WM_ACTIVATE` |
| `onScreenSave` | 屏幕保护程序启动 | `WM_SYSCOMMAND(SC_SCREENSAVE)` |
| `onMonitorPower` | 显示器电源状态 | `WM_SYSCOMMAND(SC_MONITORPOWER)` |
| `onHotKeyPressed` | 全局热键触发 | `WM_HOTKEY` |
| `onNcMouseMove` | 非客户区鼠标移动 | `WM_NCMOUSEMOVE` |
| `onWindowsMessageHook` | 通用消息钩子（setMessageHook 开启后） | 任意 WM_* |

---

## 5. MenuItemEx 子功能

`MenuItemEx` 是 windowEx 提供的另一个 TJS 类，用于管理 Win32 系统菜单项的图标和状态。它被附加（attach）到 KiriKiri 的 `MenuItem` 类上。

```cpp
// 来自 stuffs/wtnbgo_windowex_src/main.cpp，约第 1240-1280 行（节选）

struct MenuItemEx {
    // Win32 菜单项 ID 缓存
    tjs_int menuId;

    // 设置菜单项图标
    // 实现：GetMenuItemInfo + SetMenuItemInfo + MENUITEMINFOW.hbmpItem
    static tjs_error TJS_INTF_METHOD setBitmap(
        tTJSVariant *r, tjs_int n, tTJSVariant **p, iTJSDispatch2 *obj)
    {
        MenuItemEx *self = GetInstance(obj);
        if (!self) return TJS_E_ACCESSDENYED;
        // 通过 ITJSDispatch2 获取 Layer 对象的像素数据
        // 然后调用 CreateDIBSection 创建 GDI 位图
        // 最后用 SetMenuItemInfo 把位图关联到菜单项
        return self->_setBitmap(n > 0 ? p[0] : NULL);
    }
};

// 注册到 TJS 的 MenuItem 类（不是 Window 类！）
NCB_ATTACH_CLASS_WITH_HOOK(MenuItemEx, MenuItem) {
    NCB_METHOD(setBitmap);
    // ... 其他菜单项方法
}
```

---

## 6. OverlayBitmap 子窗口

`OverlayBitmap` 是 windowEx 内部的一个辅助类（不直接暴露给 TJS），用于创建一个透明的 Win32 子窗口，叠加在游戏窗口之上绘制自定义位图。

典型用途：在游戏窗口的非客户区（通常是 `WS_POPUP` 无边框窗口的上方区域）显示游戏 Logo 图像。

```cpp
// 来自 stuffs/wtnbgo_windowex_src/main.cpp，约第 1030-1200 行（节选）

class OverlayBitmap {
    HWND overlay;    // 透明子窗口句柄
    HDC  bmpdc;      // GDI 设备上下文（Device Context，绘图的"画板"）
    HBITMAP bitmap;  // GDI 位图句柄

    // 核心流程：从 TJS Layer（KiriKiri 图层对象）复制像素数据到 GDI DIB（Device Independent Bitmap）
    static HBITMAP CopyLayerToBitmap(HDC dc, int transparentThreshold,
                                     iTJSDispatch2 *layer,
                                     tjs_int &w, tjs_int &h)
    {
        ncbPropAccessor obj(layer);
        // 从 TJS Layer 对象读取像素缓冲区指针（mainImageBuffer）
        PIX *pixelData = reinterpret_cast<PIX*>(
            obj.getIntPtrValue(TJS_W("mainImageBuffer")));
        w = obj.getIntValue(TJS_W("imageWidth"));
        h = obj.getIntValue(TJS_W("imageHeight"));

        // 创建 Win32 DIB（设备无关位图）
        BITMAPINFO info; /* ... 设置 biWidth/biHeight 等 ... */
        HBITMAP bmp = ::CreateDIBSection(dc, &info, DIB_RGB_COLORS,
                                         (LPVOID*)&pixelDst, NULL, 0);
        // 逐像素拷贝（同时处理透明度阈值 transparentThreshold）
        /* ... 像素拷贝循环 ... */
        return bmp;
    }

    // 初始化透明子窗口
    bool initOverlay() {
        // 注册自定义窗口类
        WNDCLASSEXW wc;
        wc.style = CS_NOCLOSE;
        ::RegisterClassExW(&wc);

        // 创建透明分层窗口
        overlay = ::CreateWindowExW(
            WS_EX_NOACTIVATE | WS_EX_TRANSPARENT | WS_EX_LAYERED,
            WindowClass, L"", WS_CHILD,
            0, 0, 1, 1, NULL, NULL, hInstance, NULL);
        return overlay != NULL;
    }

    bool setBitmap(iTJSDispatch2 *win, iTJSDispatch2 *layer) {
        HWND base = GetHWND(win);  // 获取游戏窗口句柄
        // 寻找滚动框子窗口（KiriKiri2 特有的内部窗口结构）
        ::EnumChildWindows(base, SearchScrollBox, (LPARAM)&hwnd);

        // 从 TJS Layer 复制像素到 GDI 位图
        bitmap = CopyLayerToBitmap(dc, 0, layer, bmpw, bmph, &bmpx, &bmpy);
        ::SelectObject(bmpdc, bitmap);

        // 定位和显示叠加窗口
        ::SetParent(overlay, hwnd);
        ::SetWindowPos(overlay, HWND_TOP, 0, 0, width, height, 0);
        ::ShowWindow(overlay, SW_SHOWNORMAL);
        return true;
    }
};
```

> **Win32 知识点**：
> - `CreateWindowExW` + `WS_EX_LAYERED`：创建"分层窗口"（Layered Window），支持每像素透明度，是实现游戏 UI 叠加的标准 Win32 技术
> - `CreateDIBSection`：创建设备无关位图，返回可直接写入像素的内存指针，效率高于 `SetDIBits`
> - `BitBlt`（Block Transfer）：GDI 最基础的位块传输函数，将一块矩形像素数据从源 DC 复制到目标 DC

---

## 7. Win32 API 完整依赖清单

以下 API 清单来自两个来源的交叉验证：
1. 原始源码 `stuffs/wtnbgo_windowex_src/main.cpp` 中的直接调用
2. IDA 分析文件 `stuffs/ida_scripts/windowex_analysis.txt` 中的导入表（USER32/GDI32/SHELL32/IMM32）

### USER32.dll（窗口管理）

```
PostMessageW        — 异步发送 WM_SYSCOMMAND/WM_SETICON 等消息
SendMessageW        — 同步发送消息（需要立即得到结果时）
DefWindowProc       — 调用默认窗口处理过程（处理后再返回）
GetWindowRect       — 获取窗口外边框矩形（屏幕坐标）
GetClientRect       — 获取客户区矩形（客户区坐标）
ClientToScreen      — 将客户区坐标转换为屏幕坐标
SetWindowPos        — 设置窗口位置/大小/Z顺序
GetWindowPlacement  — 获取窗口放置信息（含最大化前的正常位置）
GetWindowLong       — 读取窗口属性（如 GWL_STYLE 样式位掩码）
SetWindowLong       — 设置窗口属性
IsZoomed            — 判断窗口是否最大化
IsIconic            — 判断窗口是否最小化
GetSystemMenu       — 获取系统菜单句柄
InsertMenuItemW     — 插入菜单项
DeleteMenu          — 删除菜单项
EnableMenuItem      — 启用/禁用/灰化菜单项
GetMenuItemCount    — 获取菜单项数量
GetMenuItemID       — 获取菜单项 ID
RegisterHotKey      — 注册全局热键
UnregisterHotKey    — 注销全局热键
RegisterDeviceNotificationW  — 注册设备变化通知
UnregisterDeviceNotification — 注销设备变化通知
CreateWindowExW     — 创建扩展样式窗口（用于 OverlayBitmap）
RegisterClassExW    — 注册窗口类
EnumChildWindows    — 枚举子窗口（寻找 KiriKiri 滚动框）
SetParent           — 设置父窗口
ShowWindow          — 显示/隐藏窗口
InvalidateRect      — 标记矩形区域需要重绘
UpdateWindow        — 立即重绘窗口
GetDC               — 获取设备上下文
ReleaseDC           — 释放设备上下文
GetMonitorInfoW     — 获取显示器信息（多显示器支持）
MonitorFromWindow   — 获取窗口所在显示器
```

### GDI32.dll（图形绘制）

```
CreateCompatibleDC  — 创建与指定 DC 兼容的内存 DC
CreateDIBSection    — 创建设备无关位图（带直接内存访问）
SelectObject        — 将 GDI 对象（位图、画笔等）选入 DC
DeleteDC            — 删除内存 DC
DeleteObject        — 释放 GDI 对象（位图、画笔等）
BitBlt              — 位块传输（复制矩形像素区域）
BeginPaint          — 开始绘制（响应 WM_PAINT）
EndPaint            — 结束绘制
```

### SHELL32.dll（Shell 接口）

```
ExtractIconW        — 从 EXE/DLL/ICO 文件中提取图标
```

### IMM32.dll（输入法）

```
ImmAssociateContextEx — 关联/解关联输入法上下文（IME 控制）
```

### 动态加载：DWMAPI.dll（桌面窗口管理器）

```cpp
// 通过 LoadLibraryW("dwmapi.dll") + GetProcAddress 动态加载
// 目的：在不支持 DWM 的系统（如 Windows XP）上也能运行
DwmSetWindowAttribute  — 设置窗口 DWM 属性（圆角偏好等）
```

---

## 8. TJS 注册机制详解

windowEx 使用 ncbind 框架的三种注册方式，分别适用于不同场景：

### 8.1 RawCallback 方式（最底层，访问 TJS 底层接口）

```cpp
// 函数签名固定：(返回值指针, 参数数量, 参数数组, 调用者对象)
static tjs_error TJS_INTF_METHOD minimize(
    tTJSVariant *r,   // 返回值（可为 NULL）
    tjs_int n,        // 参数数量
    tTJSVariant **p,  // 参数数组
    iTJSDispatch2 *obj) // 调用者（Window 对象）
{
    return postSysCommand(obj, SC_MINIMIZE);
}

// 注册（在 NCB_ATTACH_CLASS_WITH_HOOK 宏块内）：
NCB_RAW_METHOD(minimize);    // 注册为 TJS 方法
```

> **TJS_INTF_METHOD** 宏在 Windows 上展开为 `__stdcall`（标准调用约定），在其他平台上为空。这是 Win32 COM 接口调用约定的遗留。

### 8.2 Variant 方式（属性读写）

```cpp
// 属性 getter：通过 ncbPropAccessor 读取 TJS 对象属性
static tjs_error TJS_INTF_METHOD getMaximizeBox(
    tTJSVariant *r, tjs_int n, tTJSVariant **p, iTJSDispatch2 *obj)
{
    HWND hwnd = GetHWND(obj);
    if (r) *r = (GetWindowLong(hwnd, GWL_STYLE) & WS_MAXIMIZEBOX) != 0;
    return TJS_S_OK;
}

// 注册为可读写属性：
NCB_PROPERTY_RW(maximizeBox, getMaximizeBox, setMaximizeBox);
```

### 8.3 NCB_ATTACH_CLASS_WITH_HOOK 宏（完整注册块）

```cpp
// 完整注册块——把所有方法/属性/事件名称字符串集中注册到 TJS Window 类
NCB_ATTACH_CLASS_WITH_HOOK(WindowEx, Window) {

    // --- 方法 ---
    NCB_RAW_METHOD(minimize);
    NCB_RAW_METHOD(maximize);
    NCB_RAW_METHOD(showRestore);
    NCB_RAW_METHOD(focusMenuByKey);
    NCB_RAW_METHOD(resetWindowIcon);
    NCB_RAW_METHOD(setWindowIcon);
    NCB_RAW_METHOD(setWindowCornerPreference);
    NCB_RAW_METHOD(getWindowRect);
    NCB_RAW_METHOD(getClientRect);
    NCB_RAW_METHOD(setClientRect);      // 注意：kirikiroid2 适配版中此方法不存在
    NCB_RAW_METHOD(getNormalRect);
    NCB_RAW_METHOD(setOverlayBitmap);
    NCB_RAW_METHOD(bringTo);
    NCB_RAW_METHOD(sendToBack);
    NCB_RAW_METHOD(setMessageHook);
    NCB_RAW_METHOD(nonClientHitTest);
    NCB_RAW_METHOD(resetExSystemMenu);
    NCB_RAW_METHOD(registerDeviceChange);
    NCB_RAW_METHOD(registerHotKey);
    NCB_RAW_METHOD(acquireImeControl);
    NCB_RAW_METHOD(resetImeContext);

    // --- 属性 ---
    NCB_PROPERTY_RW(maximizeBox,       getMaximizeBox,  setMaximizeBox);
    NCB_PROPERTY_RW(minimizeBox,       getMinimizeBox,  setMinimizeBox);
    NCB_PROPERTY_RW(maximized,         getMaximized,    setMaximized);
    NCB_PROPERTY_RW(minimized,         getMinimized,    setMinimized);
    NCB_PROPERTY_RW(disableResize,     getDisableResize, setDisableResize);
    NCB_PROPERTY_RW(disableMove,       getDisableMove,  setDisableMove);
    NCB_PROPERTY_RW(enableNCMouseEvent, getEnNCMEvent,  setEnNCMEvent);
    NCB_PROPERTY_RW(exSystemMenu,      getExSystemMenu, setExSystemMenu);

    // --- 事件名称字符串常量（让 TJS 脚本可以用变量引用事件名） ---
    NCB_PROPERTY_GETTER(onMinimize,    getEvMinimize);
    NCB_PROPERTY_GETTER(onMaximize,    getEvMaximize);
    // ... 其他事件名常量
}
```

---

## 9. 代码示例：完整注册流程演示

以下示例展示如何在自己的 ncbind 插件中实现与 windowEx 类似的消息接收器架构：

```cpp
// my_window_plugin.cpp — 演示 ncbind 消息接收器完整流程

#include "ncbind.hpp"
#include <windows.h>

struct MyWindowPlugin {
    iTJSDispatch2 *self;  // 关联的 TJS Window 对象

    // ① 静态回调：Win32 消息进来时引擎调用此函数
    static bool __stdcall receiver(void *userdata, tTVPWindowMessage *mes) {
        auto *inst = ncbInstanceAdaptor<MyWindowPlugin>::GetNativeInstance(
            (iTJSDispatch2*)userdata);
        if (!inst) return false;

        // 处理 WM_SIZE：窗口大小改变时触发 TJS 事件
        if (mes->Msg == WM_SIZE) {
            tTJSVariant w((tjs_int)LOWORD(mes->LParam));
            tTJSVariant h((tjs_int)HIWORD(mes->LParam));
            tTJSVariant *params[] = { &w, &h };
            tTJSVariant rslt;
            inst->self->FuncCall(0, TJS_W("onResized"), 0, &rslt, 2, params, inst->self);
        }
        return false;  // 不拦截消息
    }

    // ② 注册/注销消息接收器
    void regist(bool enable) {
        tTJSVariant mode(enable ? wrmRegister : wrmUnregister);
        tTJSVariant func((tTVInteger)(&receiver));
        tTJSVariant data((tTVInteger)(self));
        tTJSVariant rslt, *params[] = { &mode, &func, &data };
        self->FuncCall(0, TJS_W("registerMessageReceiver"), 0, &rslt, 3, params, self);
    }

    // ③ 构造/析构时自动注册/注销
    MyWindowPlugin(iTJSDispatch2 *obj) : self(obj) { regist(true); }
    ~MyWindowPlugin() { regist(false); }
};

// ④ 附加到 TJS Window 类
NCB_ATTACH_CLASS_WITH_HOOK(MyWindowPlugin, Window) {
    // （这里注册方法和属性）
}
```

---

## 10. 动手实践

按照以下步骤，验证你对 windowEx 源码的理解：

**步骤 1**：在 `stuffs/wtnbgo_windowex_src/main.cpp` 中找到 `onMessage` 函数，数出它处理了多少种不同的 `WM_*` 消息类型（提示：数 `case WM_` 的行数）。

**步骤 2**：阅读 `OverlayBitmap::CopyLayerToBitmap` 函数，理解它如何从 TJS `Layer` 对象的 `mainImageBuffer` 属性读取原始像素数据，并写出这段读取逻辑的伪代码（不需要 Win32 GDI 部分）。

**步骤 3**：在原始 `regist()` 函数中，`data` 参数被设置为 `(tTVInteger)(self)`。思考：如果这里误传了 `this`（C++ 实例指针）而不是 `self`（TJS 对象指针），会发生什么问题？（提示：看 `receiver` 函数如何使用 `userdata`）

---

## 对照项目源码

| 原始源码 | 项目适配版 | 差异 |
|--------|-----------|------|
| `stuffs/wtnbgo_windowex_src/main.cpp` | `cpp/plugins/windowEx.cpp` | 适配版大量 stub 化，详见下节 |
| `::GetWindowRect(hwnd, &rect)` | 读取 TJS 对象的 left/top/width/height 属性 | 跨平台替代方案 |
| `regist(bool en)` — 18 行完整实现 | `regist(bool en) {}` — 空函数体 | 完全 stub 化 |
| `onMessage(mes)` — 150+ 行 | 不存在此函数 | 整体删除 |

完整适配策略分析见下节：[02-跨平台适配策略与实现.md](./02-跨平台适配策略与实现.md)

---

## 本节小结

- windowEx 原始源码是单文件（main.cpp，1327 行），包含三个结构体：**WindowEx**（主逻辑）、**MenuItemEx**（菜单扩展）、**OverlayBitmap**（叠加位图子窗口）
- 核心机制是**消息接收器**：通过 `registerMessageReceiver` 注册静态函数 `receiver`，在 `onMessage` 中分发处理 20+ 种 WM_* 消息
- 插件暴露 **30+ 个** TJS 方法/属性，覆盖最大化/最小化/移动/图标/菜单/热键/设备通知/IME 等功能
- 完整依赖 4 个 Win32 DLL：USER32（窗口管理）、GDI32（图形）、SHELL32（图标）、IMM32（输入法）
- TJS 注册使用 ncbind 框架：`NCB_ATTACH_CLASS_WITH_HOOK` + `NCB_RAW_METHOD` + `NCB_PROPERTY_RW` 三种宏完成注册

---

## 练习题与答案

### 题目 1：消息拦截机制

`onMessage` 函数有时 `return true`，有时 `return false`。这两者有什么区别？在 `disableResize` 逻辑中为什么必须 `return true`？

<details>
<summary>查看答案</summary>

**区别**：
- `return false`：插件处理了消息，但不拦截——引擎会继续把这个消息交给默认处理流程（`DefWindowProc`）。适用于"我只是旁观/响应，不改变默认行为"的场景，如触发 `onMinimize` 事件后继续让系统处理最小化。
- `return true`：插件拦截了消息——引擎**不再调用** `DefWindowProc`，使用插件设置的 `mes->Result` 作为最终结果。适用于"我需要改变默认行为"的场景。

**disableResize 为什么必须 return true**：

`WM_NCHITTEST` 消息用于确定鼠标在窗口哪个区域（标题栏、边框、客户区等）。默认处理结果如果是 `HTLEFT`、`HTRIGHT` 等边框区域，用户就能拖动边框调整大小。

`disableResize` 逻辑首先调用 `DefWindowProc` 得到标准结果（存入 `mes->Result`），然后如果结果是某个边框区域（`HTLEFT` 到 `HTBOTTOMRIGHT`），就把它改写为 `HTCLIENT`（客户区）——这样 Windows 就不会触发调整大小的操作。

必须 `return true` 是因为：如果不拦截，引擎会用 `DefWindowProc` 的**原始结果**（边框区域代码）继续处理，覆盖掉我们写入 `mes->Result` 的修改，`disableResize` 就失效了。

```cpp
case WM_NCHITTEST:
    if (disableResize) {
        // 先用 DefWindowProc 得到标准结果
        mes->Result = ::DefWindowProc(hwnd, mes->Msg, mes->WParam, mes->LParam);
        // 如果是可调整大小的边框区域（HTLEFT=10 到 HTBOTTOMRIGHT=17）
        if (mes->Result >= HTLEFT && mes->Result <= HTBOTTOMRIGHT)
            mes->Result = HTCLIENT;  // 改写为客户区，禁止调整大小
        return true;  // 必须拦截，否则我们的 mes->Result 修改被覆盖！
    }
```

</details>

### 题目 2：OverlayBitmap 的 `userdata` 机制

为什么 `receiver` 函数的 `userdata` 参数传入的是 `self`（`iTJSDispatch2*` 类型的 TJS 对象指针），而不是直接传入 `this`（C++ 结构体指针）？传入 `this` 会有什么问题？

<details>
<summary>查看答案</summary>

**传入 this 的根本问题：生命周期不可控**

如果传入 `this`，当 `WindowEx` C++ 对象被销毁时（析构函数调用），`this` 指针就成了悬空指针（dangling pointer）。但引擎不知道这个指针已经失效，下次消息来时仍然会调用 `receiver(dangling_ptr, mes)`，导致访问已释放内存，程序崩溃。

`~WindowEx()` 的最后一行 `regist(false)` 就是为了在析构时注销接收器，但如果此时引擎刚好在处理消息（多线程或重入场景），时序问题仍可能导致崩溃。

**传入 self（TJS 对象）的好处**：

1. **生命周期由 TJS 管理**：TJS 对象有引用计数，只要游戏脚本持有 `Window` 对象的引用，`self` 指针就一定有效。
2. **安全的实例恢复**：`GetInstance(obj)` 通过 `ncbInstanceAdaptor` 从 TJS 对象取出 C++ 实例指针，如果 C++ 实例已经不存在（比如插件已卸载），`GetInstance` 会返回 `NULL`，`receiver` 函数安全地返回 `false` 而不崩溃。
3. **与 ncbind 架构一致**：ncbind 的整个设计都是以 `iTJSDispatch2*` 为锚点，C++ 实例通过 TJS 对象来寻址。

```cpp
static bool __stdcall receiver(void *userdata, tTVPWindowMessage *mes) {
    // 从 TJS 对象恢复 C++ 实例——如果实例已销毁，返回 NULL
    WindowEx *inst = GetInstance((iTJSDispatch2*)userdata);
    // NULL 检查保护：安全返回 false 而不是崩溃
    return inst ? inst->onMessage(mes) : false;
}
```

</details>

### 题目 3：Win32 API 依赖识别

如果你只有 windowEx.dll 二进制文件（没有源码），仅通过 IDA 的**导入表**（Import Table），你能推断出这个插件的哪些核心功能？请从导入表信息中列出 3 条具体推断。

<details>
<summary>查看答案</summary>

导入表中可见的 Win32 API 直接告诉我们插件使用了哪些功能：

**推断 1：从 `RegisterHotKey` / `UnregisterHotKey` → 全局热键功能**
这两个函数只用于注册/注销系统级热键，功能非常专一。看到它们就能确定插件提供了全局热键 API。

**推断 2：从 `CreateDIBSection` + `BitBlt` + `CreateWindowExW` → 自定义位图叠加窗口**
这三个函数的组合模式明确指向"创建一个 Win32 窗口并在上面绘制位图"。`CreateDIBSection` 创建像素缓冲，`BitBlt` 将像素复制到屏幕，`CreateWindowExW` 创建承载这个位图的窗口——即 OverlayBitmap 功能。

**推断 3：从 `RegisterDeviceNotificationW` / `UnregisterDeviceNotification` → 设备插拔通知**
这对函数专门用于订阅 Windows 设备变化事件（U 盘插拔、手柄连接等）。看到这两个函数，就能确定插件的 `registerDeviceChange` 功能存在。

**额外推断（奖励）**：
- `DwmSetWindowAttribute`（dwmapi.dll）→ Win11 窗口圆角控制（该函数仅用于此目的）
- `ImmAssociateContextEx`（imm32.dll）→ IME 输入法控制（该函数专门控制 IME 上下文关联）
- `ExtractIconW`（shell32.dll）→ 从文件提取图标（`setWindowIcon` 功能）

这正是第三节"无源码逆向对比与验证"的核心方法论——导入表是信息密度最高的侦察起点。

</details>

---

## 下一步

[02-跨平台适配策略与实现.md](./02-跨平台适配策略与实现.md) — 详细分析如何把以上所有 Win32 依赖逐一处理，将 windowEx 移植到 Android/Linux/macOS 平台。
