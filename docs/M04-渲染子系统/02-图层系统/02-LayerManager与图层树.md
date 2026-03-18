# LayerManager 与图层树

> **所属模块：** M04-渲染子系统
> **前置知识：** [01-LayerIntf与LayerBitmapIntf](./01-LayerIntf与LayerBitmapIntf.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 `tTVPLayerManager` 的生命周期与职责边界
2. 掌握图层树的 Parent/Children 关系与遍历方式
3. 使用 Z-Order 管理接口控制图层堆叠顺序
4. 理解焦点管理链路与 `JoinFocusChain` 机制
5. 运用模态层（Modal Layer）实现对话框阻塞
6. 掌握鼠标/触摸捕获（Capture）机制

---

## 2.1 tTVPLayerManager 类概述

### 2.1.1 接口定义：iTVPLayerManager

`iTVPLayerManager` 是图层管理器的纯虚接口，定义于 `LayerManager.h`：

```cpp
// cpp/core/visual/LayerManager.h 第 19-79 行
class iTVPLayerManager {
public:
    // ---------- 生命周期 ----------
    virtual void AddRef() = 0;
    virtual void Release() = 0;

    // ---------- Primary 层访问 ----------
    virtual tTJSNI_BaseLayer* GetPrimaryLayer() const = 0;

    // ---------- 焦点管理 ----------
    virtual tTJSNI_BaseLayer* GetFocusedLayer() const = 0;
    virtual bool SetFocusTo(tTJSNI_BaseLayer* layer, bool direction = true) = 0;

    // ---------- 输入事件通知 ----------
    virtual void NotifyClick(tjs_int x, tjs_int y) = 0;
    virtual void NotifyDoubleClick(tjs_int x, tjs_int y) = 0;
    virtual void NotifyMouseDown(tjs_int x, tjs_int y, 
                                 tTVPMouseButton mb, tjs_uint32 flags) = 0;
    virtual void NotifyMouseUp(tjs_int x, tjs_int y,
                               tTVPMouseButton mb, tjs_uint32 flags) = 0;
    virtual void NotifyMouseMove(tjs_int x, tjs_int y, tjs_uint32 flags) = 0;
    virtual void NotifyMouseWheel(tjs_uint32 shift, tjs_int delta,
                                  tjs_int x, tjs_int y) = 0;
    
    // ---------- 触摸事件通知 ----------
    virtual void NotifyTouchDown(tjs_real x, tjs_real y, tjs_real cx,
                                 tjs_real cy, tjs_uint32 id) = 0;
    virtual void NotifyTouchUp(tjs_real x, tjs_real y, tjs_real cx,
                               tjs_real cy, tjs_uint32 id) = 0;
    virtual void NotifyTouchMove(tjs_real x, tjs_real y, tjs_real cx,
                                 tjs_real cy, tjs_uint32 id) = 0;
    virtual void NotifyTouchScaling(tjs_real startdist, tjs_real curdist,
                                    tjs_real cx, tjs_real cy, tjs_int flag) = 0;
    virtual void NotifyTouchRotate(tjs_real startangle, tjs_real curangle,
                                   tjs_real dist, tjs_real cx, tjs_real cy,
                                   tjs_int flag) = 0;
    virtual void NotifyMultiTouch() = 0;

    // ---------- 键盘事件通知 ----------
    virtual void NotifyKeyDown(tjs_uint key, tjs_uint32 shift) = 0;
    virtual void NotifyKeyUp(tjs_uint key, tjs_uint32 shift) = 0;
    virtual void NotifyKeyPress(tjs_char key) = 0;

    // ---------- 画面更新 ----------
    virtual void RequestInvalidation(const tTVPRect& rect) = 0;
    virtual void UpdateToDrawDevice() = 0;
};
```

**核心职责：**

| 职责 | 说明 |
|------|------|
| Primary 层管理 | 管理整个图层树的根节点 |
| 焦点管理 | 追踪当前拥有键盘焦点的图层 |
| 输入分发 | 将鼠标、键盘、触摸事件路由到正确的图层 |
| 更新调度 | 收集脏区域，触发重绘 |

### 2.1.2 实现类：tTVPLayerManager

```cpp
// cpp/core/visual/LayerManager.h 第 81-220 行
class tTVPLayerManager : public iTVPLayerManager {
private:
    // ---------- 引用计数 ----------
    tjs_int RefCount = 1;

    // ---------- 关联的 TreeOwner ----------
    iTVPLayerTreeOwner* LayerTreeOwner = nullptr;  // 所属窗口

    // ---------- Primary 层 ----------
    tTJSNI_BaseLayer* Primary = nullptr;           // 图层树根节点

    // ---------- 焦点层 ----------
    tTJSNI_BaseLayer* FocusedLayer = nullptr;      // 当前焦点层

    // ---------- 模态层栈 ----------
    std::vector<tTJSNI_BaseLayer*> ModalLayerVector;  // 模态层栈

    // ---------- 鼠标捕获 ----------
    tTJSNI_BaseLayer* CaptureOwner = nullptr;      // 捕获鼠标的图层
    tTJSNI_BaseLayer* LastMouseMoveSent = nullptr; // 上次收到 MouseMove 的图层

    // ---------- 触摸捕获 ----------
    struct tTouchCaptureInfo {
        tjs_uint32 id;
        tTJSNI_BaseLayer* owner;
    };
    std::vector<tTouchCaptureInfo> TouchCapture;   // 多点触控捕获列表

    // ---------- 脏区域 ----------
    tTVPComplexRect UpdateRegion;                  // 待重绘区域

    // ---------- 扁平化节点列表 ----------
    std::vector<tTJSNI_BaseLayer*> AllNodes;       // 按 Z 序排列的所有可见图层
    bool AllNodesValid = false;                    // AllNodes 是否有效
};
```

**关键成员解释：**

1. **`Primary`**：图层树的根节点，通过 `AttachPrimary()` 设置
2. **`FocusedLayer`**：当前持有键盘焦点的图层
3. **`ModalLayerVector`**：模态层栈，栈顶图层及其子孙可接收输入
4. **`CaptureOwner`**：鼠标捕获者，捕获期间所有鼠标事件都发送给它
5. **`TouchCapture`**：多点触控捕获，每个触点 ID 可被不同图层捕获
6. **`AllNodes`**：扁平化的图层列表，用于快速 Z 序遍历

### 2.1.3 生命周期：引用计数管理

```cpp
// cpp/core/visual/LayerManager.cpp 第 18-35 行
void tTVPLayerManager::AddRef() {
    RefCount++;
}

void tTVPLayerManager::Release() {
    if(RefCount == 1) {
        delete this;  // 引用计数归零时自毁
    } else {
        RefCount--;
    }
}
```

**使用模式：**

```cpp
// 获取 LayerManager（通过窗口）
iTVPLayerManager* mgr = window->GetLayerManager();
mgr->AddRef();  // 持有引用

// 使用完毕
mgr->Release();  // 释放引用
```

---

## 2.2 图层树结构

### 2.2.1 Parent/Children 关系

每个图层通过 `Parent` 指针指向父图层，通过 `Children` 容器管理子图层：

```cpp
// cpp/core/visual/LayerIntf.h 第 180-200 行
class tTJSNI_BaseLayer {
protected:
    tTJSNI_BaseLayer* Parent = nullptr;            // 父图层
    tObjectList<tTJSNI_BaseLayer> Children;        // 子图层列表
    
    tjs_uint OrderIndex = 0;        // 相对顺序（在兄弟中的位置）
    tjs_uint OverallOrderIndex = 0; // 全局顺序（在整棵树中的位置）
    
    bool AbsoluteOrderMode = false; // 是否使用绝对顺序模式
    tjs_int AbsoluteOrderIndex = 0; // 绝对顺序索引
};
```

### 2.2.2 加入/离开图层树

```cpp
// cpp/core/visual/LayerIntf.cpp - Join() 方法简化版
void tTJSNI_BaseLayer::Join(tTJSNI_BaseLayer* parent) {
    if(Parent) 
        TVPThrowExceptionMessage(TVPAlreadyJoined);  // 已有父节点则报错
    
    // 检查 Manager 一致性
    if(parent->Manager != Manager && Manager != nullptr)
        TVPThrowExceptionMessage(TVPCannotJoinDifferentTree);
    
    Parent = parent;
    parent->Children.Add(this);  // 加入父节点的子列表
    
    // 继承 Manager
    if(parent->Manager) {
        Manager = parent->Manager;
        Manager->InvalidateOverallIndex();  // 使全局索引失效
    }
    
    parent->NotifyChildrenVisualStateChanged();  // 通知父节点
}

// Part() 方法 - 离开图层树
void tTJSNI_BaseLayer::Part() {
    if(!Parent) return;  // 没有父节点则无操作
    
    Parent->SeverChild(this);  // 从父节点移除
    Parent = nullptr;
    Manager = nullptr;  // 清除 Manager 引用
}
```

### 2.2.3 图层树遍历宏

KrKr2 提供了一组遍历子图层的宏，位于 `LayerIntf.h`：

```cpp
// 安全遍历（遍历期间可修改 Children）
#define TVP_LAYER_FOR_EACH_CHILD_BEGIN(child) \
    { \
        tObjectListSafeLockHolder<tTJSNI_BaseLayer> holder(Children); \
        tjs_int count = Children.GetSafeLockedObjectCount(); \
        for(tjs_int i = 0; i < count; i++) { \
            tTJSNI_BaseLayer* child = Children.GetSafeLockedObjectAt(i); \
            if(!child) continue;

#define TVP_LAYER_FOR_EACH_CHILD_END \
        } \
    }

// 非安全遍历（遍历期间不可修改 Children，但更快）
#define TVP_LAYER_FOR_EACH_CHILD_NOLOCK_BEGIN(child) \
    { \
        Children.Compact(); \
        tjs_int count = Children.GetActualCount(); \
        for(tjs_int i = 0; i < count; i++) { \
            tTJSNI_BaseLayer* child = Children[i];

#define TVP_LAYER_FOR_EACH_CHILD_NOLOCK_END \
        } \
    }
```

**使用示例：**

```cpp
// 遍历所有子图层并设置可见性
void SetAllChildrenVisible(tTJSNI_BaseLayer* parent, bool visible) {
    TVP_LAYER_FOR_EACH_CHILD_NOLOCK_BEGIN(child)
        child->SetVisible(visible);
        // 递归处理子图层的子图层
        SetAllChildrenVisible(child, visible);
    TVP_LAYER_FOR_EACH_CHILD_NOLOCK_END
}
```

### 2.2.4 AllNodes 扁平化列表

`LayerManager` 维护一个按 Z 序排列的扁平化图层列表：

```cpp
// cpp/core/visual/LayerManager.cpp 第 120-165 行
void tTVPLayerManager::RecreateOverallOrderIndex() {
    if(AllNodesValid) return;  // 已有效则跳过
    
    AllNodes.clear();
    
    // 递归收集所有可见图层
    if(Primary) {
        CollectAllNodes(Primary);
    }
    
    // 按收集顺序分配 OverallOrderIndex
    for(tjs_uint i = 0; i < AllNodes.size(); i++) {
        AllNodes[i]->OverallOrderIndex = i;
    }
    
    AllNodesValid = true;
}

void tTVPLayerManager::CollectAllNodes(tTJSNI_BaseLayer* layer) {
    // 先加入自身
    AllNodes.push_back(layer);
    
    // 再按顺序加入所有子图层
    TVP_LAYER_FOR_EACH_CHILD_NOLOCK_BEGIN(child)
        CollectAllNodes(child);  // 递归
    TVP_LAYER_FOR_EACH_CHILD_NOLOCK_END
}
```

**Z 序规则：**

1. 父图层在子图层之前（深度优先）
2. 兄弟图层按 `OrderIndex` 排列
3. 索引越大，图层越靠前（绘制在上方）

---

## 2.3 Z 序管理

### 2.3.1 OrderIndex 与 OverallOrderIndex

| 属性 | 作用域 | 说明 |
|------|--------|------|
| `OrderIndex` | 兄弟间 | 在同一父图层下的相对顺序 |
| `OverallOrderIndex` | 全局 | 在整棵树中的绝对顺序（用于渲染和事件分发） |
| `AbsoluteOrderIndex` | 兄弟间 | 绝对顺序模式下的索引值 |

### 2.3.2 调整 Z 序的方法

```cpp
// cpp/core/visual/LayerIntf.cpp 第 1323-1399 行

// 移动到指定兄弟之前（更靠后/下方）
void tTJSNI_BaseLayer::MoveBefore(tTJSNI_BaseLayer* lay) {
    if(Parent) Parent->SetAbsoluteOrderMode(false);  // 切换到相对模式
    
    CheckZOrderMoveRule(lay);  // 检查规则
    
    tjs_int this_order = GetOrderIndex();
    tjs_int lay_order = lay->GetOrderIndex();
    
    if(this_order < lay_order)
        Parent->ChildChangeOrder(this_order, lay_order);      // 向前移动
    else
        Parent->ChildChangeOrder(this_order, lay_order + 1);  // 向后移动
}

// 移动到指定兄弟之后（更靠前/上方）
void tTJSNI_BaseLayer::MoveBehind(tTJSNI_BaseLayer* lay) {
    if(Parent) Parent->SetAbsoluteOrderMode(false);
    
    CheckZOrderMoveRule(lay);
    
    tjs_int this_order = GetOrderIndex();
    tjs_int lay_order = lay->GetOrderIndex();
    
    if(this_order < lay_order)
        Parent->ChildChangeOrder(this_order, lay_order - 1);
    else
        Parent->ChildChangeOrder(this_order, lay_order);
}

// 移动到最底层
void tTJSNI_BaseLayer::BringToBack() {
    if(!Parent) TVPThrowExceptionMessage(TVPCannotMovePrimaryOrSiblingless);
    Parent->SetAbsoluteOrderMode(false);
    SetOrderIndex(0);
}

// 移动到最顶层
void tTJSNI_BaseLayer::BringToFront() {
    if(!Parent) TVPThrowExceptionMessage(TVPCannotMovePrimaryOrSiblingless);
    Parent->SetAbsoluteOrderMode(false);
    SetOrderIndex(Parent->Children.GetActualCount() - 1);
}
```

### 2.3.3 邻近图层访问

```cpp
// cpp/core/visual/LayerIntf.cpp 第 1205-1246 行

// 获取 Z 序上方（索引更小）的邻居
tTJSNI_BaseLayer* tTJSNI_BaseLayer::GetNeighborAbove(bool loop) {
    if(!Manager) return nullptr;
    
    tjs_uint index = GetOverallOrderIndex();
    std::vector<tTJSNI_BaseLayer*>& allnodes = Manager->GetAllNodes();
    
    if(allnodes.size() == 0) return nullptr;
    
    if(index == 0) {
        // 已是第一个（Primary）
        if(loop)
            return *(allnodes.end() - 1);  // 循环到最后一个
        else
            return nullptr;
    }
    
    return *(allnodes.begin() + index - 1);
}

// 获取 Z 序下方（索引更大）的邻居
tTJSNI_BaseLayer* tTJSNI_BaseLayer::GetNeighborBelow(bool loop) {
    if(!Manager) return nullptr;
    
    tjs_uint index = GetOverallOrderIndex();
    std::vector<tTJSNI_BaseLayer*>& allnodes = Manager->GetAllNodes();
    
    if(allnodes.size() == 0) return nullptr;
    
    if(index == allnodes.size() - 1) {
        // 已是最后一个
        if(loop)
            return *(allnodes.begin());  // 循环到第一个
        else
            return nullptr;
    }
    
    return *(allnodes.begin() + index + 1);
}
```

---

## 2.4 焦点管理

### 2.4.1 焦点层追踪

```cpp
// cpp/core/visual/LayerManager.cpp 第 200-280 行
bool tTVPLayerManager::SetFocusTo(tTJSNI_BaseLayer* layer, bool direction) {
    if(FocusedLayer == layer) return true;  // 已是焦点层
    
    // 检查是否可聚焦
    if(layer) {
        if(!layer->GetNodeFocusable()) return false;
        if(!layer->ParentFocusable()) return false;
        if(layer->IsDisabledByMode()) return false;  // 被模态层禁用
    }
    
    tTJSNI_BaseLayer* prevfocused = FocusedLayer;
    
    // 触发 onBeforeFocus 事件
    if(layer) {
        layer = layer->FireBeforeFocus(prevfocused, direction);
        if(!layer) return false;  // 事件处理中取消了焦点切换
    }
    
    FocusedLayer = layer;
    
    // 触发失焦事件
    if(prevfocused)
        prevfocused->FireBlur(layer);
    
    // 触发聚焦事件
    if(layer)
        layer->FireFocus(prevfocused, direction);
    
    return true;
}
```

### 2.4.2 焦点链遍历

```cpp
// cpp/core/visual/LayerIntf.cpp 第 3562-3631 行

// 查找下一个可聚焦图层（向前）
tTJSNI_BaseLayer* tTJSNI_BaseLayer::_GetNextFocusable() {
    tTJSNI_BaseLayer* p = this;
    tTJSNI_BaseLayer* current = this;
    
    p = p->GetNeighborBelow(true);  // 从 Z 序下方开始
    if(current == p) return nullptr;
    if(!p) return nullptr;
    
    current = p;
    do {
        if(p->GetNodeFocusable() && p->JoinFocusChain)
            return p;  // 找到可聚焦且加入焦点链的图层
    } while(p = p->GetNeighborBelow(true), p && p != current);
    
    return nullptr;
}

// 查找上一个可聚焦图层（向后）
tTJSNI_BaseLayer* tTJSNI_BaseLayer::_GetPrevFocusable() {
    tTJSNI_BaseLayer* p = this;
    tTJSNI_BaseLayer* current = this;
    
    p = p->GetNeighborAbove(true);  // 从 Z 序上方开始
    if(current == p) return nullptr;
    if(!p) return nullptr;
    
    current = p;
    do {
        if(p->GetNodeFocusable() && p->JoinFocusChain)
            return p;
    } while(p = p->GetNeighborAbove(true), p && p != current);
    
    return nullptr;
}
```

### 2.4.3 JoinFocusChain 属性

`JoinFocusChain` 属性决定图层是否参与 Tab 键焦点切换：

```cpp
// tTJSNI_BaseLayer 成员
bool JoinFocusChain = true;  // 默认参与焦点链

// 设置方法
void tTJSNI_BaseLayer::SetJoinFocusChain(bool b) {
    JoinFocusChain = b;
}
```

**典型用法：**

- 背景图层：`JoinFocusChain = false`（不需要键盘焦点）
- 按钮/输入框：`JoinFocusChain = true`（需要键盘焦点）

---

## 2.5 模态层系统

### 2.5.1 模态层概念

模态层（Modal Layer）是一种阻塞式交互模式：当模态层激活时，只有该图层及其子孙可以接收输入事件。

```
┌─────────────────────────────────────────┐
│ Primary Layer (被模态层禁用)              │
│  ┌───────────────────────────────────┐  │
│  │ Background Layer (被禁用)          │  │
│  └───────────────────────────────────┘  │
│  ┌───────────────────────────────────┐  │
│  │ Dialog Layer (模态层 - 可交互)     │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │ Button Layer (可交互)        │  │  │
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### 2.5.2 设置/移除模态层

```cpp
// cpp/core/visual/LayerManager.cpp 第 400-450 行

void tTVPLayerManager::SetModeTo(tTJSNI_BaseLayer* layer) {
    // 保存当前 Enabled 状态
    SaveEnabledWork();
    
    // 将图层压入模态层栈
    ModalLayerVector.push_back(layer);
    
    // 移除模态层之外的焦点
    if(FocusedLayer && FocusedLayer->IsDisabledByMode()) {
        // 当前焦点图层被禁用，转移焦点到模态层
        tTJSNI_BaseLayer* newfocus = layer->SearchFirstFocusable(true);
        if(newfocus)
            SetFocusTo(newfocus, true);
        else
            SetFocusTo(nullptr, true);  // 无可聚焦图层
    }
    
    // 通知 Enabled 状态变化
    NotifyNodeEnabledState();
}

void tTVPLayerManager::RemoveModeFrom(tTJSNI_BaseLayer* layer) {
    SaveEnabledWork();
    
    // 从栈中移除
    auto it = std::find(ModalLayerVector.begin(), ModalLayerVector.end(), layer);
    if(it != ModalLayerVector.end()) {
        ModalLayerVector.erase(it);
    }
    
    NotifyNodeEnabledState();
}

tTJSNI_BaseLayer* tTVPLayerManager::GetCurrentModalLayer() {
    if(ModalLayerVector.empty()) return nullptr;
    return ModalLayerVector.back();  // 返回栈顶
}
```

### 2.5.3 检查模态层禁用状态

```cpp
// cpp/core/visual/LayerIntf.cpp 第 3728-3736 行
bool tTJSNI_BaseLayer::IsDisabledByMode() {
    if(!Manager) return false;
    
    tTJSNI_BaseLayer* current = Manager->GetCurrentModalLayer();
    if(!current) return false;  // 没有模态层
    
    // 检查自己是否是模态层或其子孙
    return !IsAncestorOrSelf(current);
}

// 辅助方法：检查 layer 是否是自己的祖先或自己
bool tTJSNI_BaseLayer::IsAncestorOrSelf(tTJSNI_BaseLayer* layer) {
    tTJSNI_BaseLayer* p = this;
    while(p) {
        if(p == layer) return true;
        p = p->Parent;
    }
    return false;
}
```

---

## 2.6 鼠标/触摸捕获

### 2.6.1 鼠标捕获

```cpp
// cpp/core/visual/LayerManager.cpp 第 500-550 行

void tTVPLayerManager::SetCapture(tTJSNI_BaseLayer* layer) {
    CaptureOwner = layer;
}

void tTVPLayerManager::ReleaseCapture() {
    CaptureOwner = nullptr;
}

// 鼠标移动分发逻辑
void tTVPLayerManager::NotifyMouseMove(tjs_int x, tjs_int y, tjs_uint32 flags) {
    tTJSNI_BaseLayer* target;
    
    if(CaptureOwner) {
        target = CaptureOwner;  // 捕获期间强制发送给捕获者
    } else {
        target = GetMostFrontChildAt(x, y, nullptr, false);
    }
    
    // 处理 MouseEnter/MouseLeave
    if(target != LastMouseMoveSent) {
        if(LastMouseMoveSent)
            LastMouseMoveSent->FireMouseLeave();
        if(target)
            target->FireMouseEnter();
        LastMouseMoveSent = target;
    }
    
    if(target) {
        // 坐标转换为图层局部坐标
        tjs_int lx = x, ly = y;
        target->FromPrimaryCoordinates(lx, ly);
        target->FireMouseMove(lx, ly, flags);
    }
}
```

### 2.6.2 多点触摸捕获

```cpp
// cpp/core/visual/LayerManager.cpp 第 600-680 行

void tTVPLayerManager::SetTouchCapture(tjs_uint32 id, tTJSNI_BaseLayer* layer) {
    // 检查是否已有此 id 的捕获
    for(auto& info : TouchCapture) {
        if(info.id == id) {
            info.owner = layer;  // 更新
            return;
        }
    }
    // 新增捕获
    TouchCapture.push_back({id, layer});
}

void tTVPLayerManager::ReleaseTouchCapture(tjs_uint32 id) {
    TouchCapture.erase(
        std::remove_if(TouchCapture.begin(), TouchCapture.end(),
            [id](const tTouchCaptureInfo& info) { return info.id == id; }),
        TouchCapture.end()
    );
}

tTJSNI_BaseLayer* tTVPLayerManager::GetTouchCaptureOwner(tjs_uint32 id) {
    for(auto& info : TouchCapture) {
        if(info.id == id)
            return info.owner;
    }
    return nullptr;
}

// 触摸移动分发逻辑
void tTVPLayerManager::NotifyTouchMove(tjs_real x, tjs_real y,
                                       tjs_real cx, tjs_real cy,
                                       tjs_uint32 id) {
    tTJSNI_BaseLayer* target = GetTouchCaptureOwner(id);
    
    if(!target) {
        // 无捕获，查找触点下的图层
        target = GetMostFrontChildAt(static_cast<tjs_int>(x),
                                     static_cast<tjs_int>(y),
                                     nullptr, false);
    }
    
    if(target) {
        tjs_real lx = x, ly = y;
        target->FromPrimaryCoordinates(lx, ly);
        target->FireTouchMove(lx, ly, cx, cy, id);
    }
}
```

---

## 动手实践

### 练习 1：创建图层树并遍历

```cpp
#include "LayerIntf.h"
#include "LayerManager.h"

void CreateAndTraverseLayerTree() {
    // 创建 Primary 层
    tTJSNI_BaseLayer* primary = new tTJSNI_BaseLayer();
    primary->SetSize(800, 600);
    primary->SetType(ltOpaque);
    
    // 创建子图层
    tTJSNI_BaseLayer* bg = new tTJSNI_BaseLayer();
    bg->SetSize(800, 600);
    bg->Join(primary);  // 加入图层树
    
    tTJSNI_BaseLayer* panel = new tTJSNI_BaseLayer();
    panel->SetSize(400, 300);
    panel->SetPosition(200, 150);
    panel->Join(primary);
    
    tTJSNI_BaseLayer* button = new tTJSNI_BaseLayer();
    button->SetSize(100, 40);
    button->SetPosition(150, 130);
    button->Join(panel);  // panel 的子图层
    
    // 遍历并打印结构
    std::function<void(tTJSNI_BaseLayer*, int)> printTree = 
        [&](tTJSNI_BaseLayer* layer, int depth) {
            // 缩进
            for(int i = 0; i < depth; i++) std::cout << "  ";
            std::cout << "Layer at (" << layer->GetLeft() << ", " 
                      << layer->GetTop() << ") size " 
                      << layer->GetWidth() << "x" << layer->GetHeight() 
                      << std::endl;
            
            TVP_LAYER_FOR_EACH_CHILD_NOLOCK_BEGIN(child)
                printTree(child, depth + 1);
            TVP_LAYER_FOR_EACH_CHILD_NOLOCK_END
        };
    
    printTree(primary, 0);
    
    // 输出：
    // Layer at (0, 0) size 800x600
    //   Layer at (0, 0) size 800x600
    //   Layer at (200, 150) size 400x300
    //     Layer at (150, 130) size 100x40
}
```

### 练习 2：Z 序调整

```cpp
void ZOrderDemo() {
    tTJSNI_BaseLayer* primary = CreatePrimaryLayer(800, 600);
    
    // 创建 3 个兄弟图层
    tTJSNI_BaseLayer* a = CreateChildLayer(primary, 100, 100, 200, 200);
    tTJSNI_BaseLayer* b = CreateChildLayer(primary, 150, 150, 200, 200);
    tTJSNI_BaseLayer* c = CreateChildLayer(primary, 200, 200, 200, 200);
    
    // 初始顺序：a(0) -> b(1) -> c(2)
    // c 在最上方
    
    // 把 a 移到最上方
    a->BringToFront();
    // 新顺序：b(0) -> c(1) -> a(2)
    
    // 把 a 移到 b 之后
    a->MoveBehind(b);
    // 新顺序：b(0) -> a(1) -> c(2)
    
    // 把 c 移到最下方
    c->BringToBack();
    // 新顺序：c(0) -> b(1) -> a(2)
}
```

### 练习 3：模态对话框

```cpp
void ModalDialogDemo() {
    tTJSNI_BaseLayer* primary = CreatePrimaryLayer(800, 600);
    tTJSNI_BaseLayer* mainContent = CreateChildLayer(primary, 0, 0, 800, 600);
    
    // 创建模态对话框
    tTJSNI_BaseLayer* dialog = CreateChildLayer(primary, 200, 150, 400, 300);
    tTJSNI_BaseLayer* okButton = CreateChildLayer(dialog, 150, 220, 100, 40);
    
    // 设置为模态
    dialog->SetMode();
    
    // 此时 mainContent 被禁用
    assert(mainContent->IsDisabledByMode() == true);
    assert(dialog->IsDisabledByMode() == false);
    assert(okButton->IsDisabledByMode() == false);
    
    // 点击 OK 按钮后移除模态
    dialog->RemoveMode();
    
    // 恢复
    assert(mainContent->IsDisabledByMode() == false);
}
```

### 练习 4：焦点链遍历

```cpp
void FocusChainDemo() {
    tTJSNI_BaseLayer* primary = CreatePrimaryLayer(800, 600);
    
    // 创建可聚焦图层
    tTJSNI_BaseLayer* input1 = CreateFocusableLayer(primary, 100, 100, 200, 30);
    tTJSNI_BaseLayer* input2 = CreateFocusableLayer(primary, 100, 150, 200, 30);
    tTJSNI_BaseLayer* button = CreateFocusableLayer(primary, 100, 200, 100, 40);
    
    // 创建背景层（不参与焦点链）
    tTJSNI_BaseLayer* bg = CreateBackgroundLayer(primary);
    bg->SetJoinFocusChain(false);
    
    // 设置初始焦点
    primary->Manager->SetFocusTo(input1, true);
    
    // Tab 切换
    tTJSNI_BaseLayer* next = input1->GetNextFocusable();  // -> input2
    primary->Manager->SetFocusTo(next, true);
    
    next = next->GetNextFocusable();  // -> button（跳过 bg）
    primary->Manager->SetFocusTo(next, true);
    
    next = next->GetNextFocusable();  // -> input1（循环）
}
```

### 练习 5：鼠标捕获实现拖拽

```cpp
void DragDemo(tTJSNI_BaseLayer* draggable) {
    tjs_int startX, startY;      // 拖拽开始时的鼠标位置
    tjs_int origLeft, origTop;   // 拖拽开始时图层位置
    
    // onMouseDown: 开始捕获
    draggable->OnMouseDown = [&](tjs_int x, tjs_int y, tTVPMouseButton mb) {
        if(mb == mbLeft) {
            draggable->Manager->SetCapture(draggable);
            startX = x; startY = y;
            origLeft = draggable->GetLeft();
            origTop = draggable->GetTop();
        }
    };
    
    // onMouseMove: 更新位置
    draggable->OnMouseMove = [&](tjs_int x, tjs_int y) {
        if(draggable->Manager->GetCaptureOwner() == draggable) {
            tjs_int dx = x - startX;
            tjs_int dy = y - startY;
            draggable->SetPosition(origLeft + dx, origTop + dy);
        }
    };
    
    // onMouseUp: 释放捕获
    draggable->OnMouseUp = [&](tjs_int x, tjs_int y, tTVPMouseButton mb) {
        if(mb == mbLeft) {
            draggable->Manager->ReleaseCapture();
        }
    };
}
```

---

## 对照项目源码

### LayerManager.h

- **第 19-79 行**：`iTVPLayerManager` 接口定义
- **第 81-220 行**：`tTVPLayerManager` 类定义

### LayerManager.cpp

- **第 18-35 行**：引用计数管理
- **第 120-165 行**：`RecreateOverallOrderIndex()` 扁平化
- **第 200-280 行**：焦点管理实现
- **第 400-450 行**：模态层管理
- **第 500-680 行**：鼠标/触摸捕获

### LayerIntf.cpp

- **第 1137-1147 行**：`RecreateOrderIndex()` 重建 OrderIndex
- **第 1205-1246 行**：邻近图层访问
- **第 1323-1399 行**：Z 序调整方法
- **第 3514-3532 行**：焦点设置
- **第 3562-3631 行**：焦点链遍历
- **第 3716-3736 行**：模态层禁用检查

---

## 本节小结

1. **tTVPLayerManager** 是图层系统的核心协调器，管理 Primary 层、焦点、模态、捕获和脏区域
2. **图层树** 通过 Parent/Children 关系组织，使用遍历宏高效遍历
3. **Z 序** 由 OrderIndex（兄弟间）和 OverallOrderIndex（全局）共同决定
4. **焦点链** 通过 `JoinFocusChain` 属性控制参与 Tab 遍历的图层
5. **模态层** 使用栈结构，栈顶图层及其子孙独占输入
6. **捕获机制** 确保拖拽等操作期间事件不会丢失

---

## 练习题与答案

### 题目 1：图层树深度计算

编写函数计算一个图层在树中的深度（Primary 层深度为 0）。

<details>
<summary>查看答案</summary>

```cpp
int GetLayerDepth(tTJSNI_BaseLayer* layer) {
    int depth = 0;
    tTJSNI_BaseLayer* p = layer;
    while(p->GetParent()) {
        depth++;
        p = p->GetParent();
    }
    return depth;
}

// 测试
void TestDepth() {
    tTJSNI_BaseLayer* primary = CreatePrimaryLayer(800, 600);
    tTJSNI_BaseLayer* a = CreateChildLayer(primary, 0, 0, 100, 100);
    tTJSNI_BaseLayer* b = CreateChildLayer(a, 0, 0, 50, 50);
    tTJSNI_BaseLayer* c = CreateChildLayer(b, 0, 0, 25, 25);
    
    assert(GetLayerDepth(primary) == 0);
    assert(GetLayerDepth(a) == 1);
    assert(GetLayerDepth(b) == 2);
    assert(GetLayerDepth(c) == 3);
}
```

</details>

### 题目 2：模态层栈

如果依次设置 A、B、C 为模态层，然后移除 B，此时哪个图层是当前模态层？为什么？

<details>
<summary>查看答案</summary>

**答案：C 仍然是当前模态层。**

**原因：**
1. 设置 A 为模态：栈 = [A]
2. 设置 B 为模态：栈 = [A, B]
3. 设置 C 为模态：栈 = [A, B, C]
4. 移除 B：栈 = [A, C]（B 从中间移除）
5. `GetCurrentModalLayer()` 返回栈顶，即 C

这种设计允许嵌套模态对话框，即使中间的对话框被关闭，最上层的对话框仍然保持模态状态。

```cpp
void TestModalStack() {
    auto mgr = GetLayerManager();
    auto A = CreateLayer(), B = CreateLayer(), C = CreateLayer();
    
    mgr->SetModeTo(A);  // 栈: [A]
    mgr->SetModeTo(B);  // 栈: [A, B]
    mgr->SetModeTo(C);  // 栈: [A, B, C]
    
    assert(mgr->GetCurrentModalLayer() == C);
    
    mgr->RemoveModeFrom(B);  // 栈: [A, C]
    
    assert(mgr->GetCurrentModalLayer() == C);  // C 仍是栈顶
}
```

</details>

### 题目 3：捕获与命中测试

在鼠标捕获状态下，如果鼠标移动到另一个图层上方，`GetMostFrontChildAt` 会返回哪个图层？捕获图层还是鼠标下方的图层？

<details>
<summary>查看答案</summary>

**答案：`GetMostFrontChildAt` 会返回鼠标下方的图层（不受捕获影响），但事件分发会强制发送给捕获图层。**

**解释：**

`GetMostFrontChildAt` 是纯粹的命中测试函数，只根据坐标和可见性判断：

```cpp
tTJSNI_BaseLayer* tTVPLayerManager::GetMostFrontChildAt(tjs_int x, tjs_int y, ...) {
    // 这里不检查 CaptureOwner
    return Primary->GetMostFrontChildAt(x, y, ...);
}
```

但在事件分发时，捕获会被优先考虑：

```cpp
void tTVPLayerManager::NotifyMouseMove(tjs_int x, tjs_int y, tjs_uint32 flags) {
    tTJSNI_BaseLayer* target;
    
    if(CaptureOwner) {
        target = CaptureOwner;  // 捕获优先！
    } else {
        target = GetMostFrontChildAt(x, y, nullptr, false);
    }
    
    // 发送事件给 target...
}
```

**意义：** 这种设计允许你在捕获期间仍能查询鼠标真正位于哪个图层上（比如用于高亮拖放目标），同时保证事件正确发送给捕获者。

</details>

---

## 下一步

[03-属性传播与事件处理](./03-属性传播与事件处理.md) — 学习 Visible/Enabled 属性的级联传播机制，以及图层事件的冒泡与分发流程。
