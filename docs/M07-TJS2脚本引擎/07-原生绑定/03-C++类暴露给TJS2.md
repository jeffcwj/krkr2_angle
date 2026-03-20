# 03 - C++类暴露给TJS2：完整实战指南

## 术语预览

| 术语 | 含义 |
|------|------|
| **暴露（Expose）** | 将C++类的方法、属性注册到TJS2运行时，使脚本代码能够创建和操作该类的实例 |
| **原生绑定（Native Binding）** | C++代码与TJS2脚本之间的桥接机制，包括类型转换、生命周期管理和调用分派 |
| **低级宏方式** | 使用`TJS_BEGIN_NATIVE_MEMBERS`等宏直接操作iTJSDispatch2接口的绑定方法，KrKr2内置类（Array、Date等）采用此方式 |
| **ncbind高级方式** | 使用`NCB_REGISTER_CLASS`等宏通过模板元编程自动完成类型转换和参数分派的绑定方法，插件系统推荐使用 |
| **ClassID** | 每个原生类的唯一标识符（tjs_int32），由`TJSRegisterNativeClass`分配，用于实例关联和类型检查 |
| **粘滞实例（Sticky Instance）** | `ncbInstanceAdaptor`中`_sticky`标志为true的实例，析构时不删除底层C++对象，用于引用外部管理的对象 |
| **成员注册（Member Registration）** | 通过`RegisterNCM`或`ncbRegistClass::Method()`将C++函数/属性添加到TJS2类对象的过程 |
| **闭包修正（Closure Fixup）** | `tTJSNativeClass::FuncCall`在实例初始化时，将objthis为NULL的闭包重新绑定到新创建实例的过程 |

---

## 本节概述

前两节分别深入剖析了低级宏系统（`tjsNative.h/cpp`）和高级ncbind系统（`ncbind.hpp`）的内部机制。本节将这些知识整合为**可操作的实战指南**：从零开始将一个C++类暴露给TJS2脚本，分别演示两种绑定方式的完整流程，并通过详细的对比分析帮助开发者选择最合适的方案。

本节内容覆盖：
1. 两种方式的完整步骤对比
2. 低级宏方式完整实战（仿照tjsArray.cpp）
3. ncbind方式完整实战（仿照steam_api.cpp）
4. 真实代码库案例深度分析
5. 类型转换陷阱与自定义转换器
6. 生命周期管理策略
7. 混合使用与高级技巧

---

## 1. 两种绑定方式总览

### 1.1 决策树：选择哪种方式

在开始编码之前，需要根据项目需求选择绑定方式。以下决策树覆盖了所有常见场景：

```
你要绑定的类是...
│
├─ KrKr2核心引擎的内置类？
│  └─ 必须使用低级宏方式（TJS_BEGIN_NATIVE_MEMBERS）
│     理由：核心类在引擎初始化时注册，不经过插件加载流程
│
├─ 第三方插件中的类？
│  └─ 强烈推荐ncbind方式（NCB_REGISTER_CLASS）
│     理由：插件使用V2Link入口，ncbind的自动注册机制与之完美配合
│
├─ 需要极致性能的底层类？
│  └─ 考虑低级宏方式
│     理由：跳过ncbind的模板抽象层，直接操作tTJSVariant
│
├─ 需要复杂继承层次？
│  └─ ncbind方式更方便（支持NCB_SUBCLASS）
│
└─ 需要精细控制成员可见性？
   └─ 低级宏方式更灵活（支持HIDDEN方法等）
```

### 1.2 对比总表

下表从多个维度对比两种绑定方式：

| 维度 | 低级宏方式 | ncbind方式 |
|------|-----------|-----------|
| **代码量** | 多（需手动处理每个参数） | 少（自动类型推导） |
| **类型安全** | 低（手动转换tTJSVariant） | 高（编译期类型检查） |
| **编译速度** | 快（无深层模板实例化） | 慢（大量模板展开） |
| **运行性能** | 略快（无模板间接层） | 略慢（多一层间接调用） |
| **错误信息** | 清晰（宏展开简单） | 晦涩（模板错误堆栈深） |
| **参数处理** | 手动检查numparams和类型 | 自动匹配函数签名 |
| **可变参数** | 容易（直接遍历param数组） | 需要RawCallback | |
| **静态方法** | `TJS_END_NATIVE_STATIC_METHOD_DECL` | 需传无this的函数指针 |
| **隐藏成员** | `TJS_END_NATIVE_HIDDEN_METHOD_DECL` | 不直接支持 |
| **属性** | `TJS_BEGIN_NATIVE_PROP_DECL` | `NCB_PROPERTY` / `NCB_PROPERTY_RO` |
| **继承支持** | 手动处理IsInstanceOf | `NCB_SUBCLASS`自动处理 |
| **插件兼容** | 需手动处理V2Link | 自动注册，零样板 |
| **使用场景** | 引擎核心7个内置类 | 所有插件类 |
| **学习曲线** | 陡峭（需理解iTJSDispatch2） | 平缓（类似Boost.Python） |

> **实际数据**：KrKr2代码库中，7个内置类（Array、Date、Dictionary、Exception、Math、RandomGenerator、RegExp）全部使用低级宏方式。唯一使用`NCB_REGISTER_CLASS`的是`steam_api.cpp`（12行代码）。大量插件通过`NCB_PRE_REGIST_CALLBACK`和`NCB_POST_REGIST_CALLBACK`注册回调函数，而非完整类。

---

## 2. 低级宏方式完整实战

本节以一个**计时器类（Timer）**为例，演示使用低级宏从零开始将C++类暴露给TJS2的完整过程。这个示例涵盖了构造函数、实例方法、静态方法、属性、析构等所有常见需求。

### 2.1 C++类设计

首先定义要暴露的C++类。这个类本身是纯C++代码，不依赖TJS2的任何头文件：

```cpp
// tjs2Timer.h - 纯C++计时器类
// 这个类将被暴露给TJS2脚本使用
#ifndef __TJS2_TIMER_H__
#define __TJS2_TIMER_H__

#include <chrono>
#include <string>

// 计时器类：提供高精度计时功能
// 设计要点：
// 1. 所有公有方法都将映射为TJS2方法
// 2. 需要考虑哪些成员暴露、哪些隐藏
// 3. 使用tjs_*类型保持与TJS2类型系统兼容
class tTJS2Timer {
private:
    // 内部状态——不暴露给TJS2
    std::chrono::high_resolution_clock::time_point startTime_;
    std::chrono::high_resolution_clock::time_point pauseTime_;
    bool running_;
    bool paused_;
    std::wstring label_;      // 计时器标签
    static int instanceCount_; // 全局实例计数

public:
    // 构造函数——将映射为TJS2构造函数
    tTJS2Timer(const std::wstring& label = L"default")
        : running_(false), paused_(false), label_(label)
    {
        instanceCount_++;
    }

    // 析构函数——将通过Invalidate链调用
    ~tTJS2Timer() {
        instanceCount_--;
    }

    // === 实例方法 ===

    // 开始计时
    void start() {
        startTime_ = std::chrono::high_resolution_clock::now();
        running_ = true;
        paused_ = false;
    }

    // 停止计时，返回经过的毫秒数
    double stop() {
        if (!running_) return 0.0;
        running_ = false;
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration<double, std::milli>(
            end - (paused_ ? pauseTime_ : startTime_));
        paused_ = false;
        return duration.count();
    }

    // 暂停计时
    void pause() {
        if (running_ && !paused_) {
            pauseTime_ = std::chrono::high_resolution_clock::now();
            paused_ = true;
        }
    }

    // 恢复计时
    void resume() {
        if (running_ && paused_) {
            auto now = std::chrono::high_resolution_clock::now();
            startTime_ += (now - pauseTime_);
            paused_ = false;
        }
    }

    // 获取当前经过时间（不停止）
    double elapsed() const {
        if (!running_) return 0.0;
        auto now = std::chrono::high_resolution_clock::now();
        auto ref = paused_ ? pauseTime_ : now;
        return std::chrono::duration<double, std::milli>(
            ref - startTime_).count();
    }

    // 重置计时器
    void reset() {
        running_ = false;
        paused_ = false;
    }

    // === 属性访问器 ===

    bool isRunning() const { return running_; }
    bool isPaused() const { return paused_; }
    const std::wstring& getLabel() const { return label_; }
    void setLabel(const std::wstring& label) { label_ = label; }

    // === 静态方法 ===

    static int getInstanceCount() { return instanceCount_; }

    // 获取系统时钟分辨率（纳秒）
    static double getResolution() {
        using period = std::chrono::high_resolution_clock::period;
        return static_cast<double>(period::num) / period::den * 1e9;
    }
};

int tTJS2Timer::instanceCount_ = 0;

#endif
```

> **设计原则**：C++类本身应该是自包含的（self-contained），不依赖TJS2头文件。绑定层负责桥接两个世界。这种分离使得C++类可以独立测试，也可以被其他C++代码直接使用。

### 2.2 原生实例类

低级宏方式要求定义一个继承自`tTJSNativeInstance`的实例类。这个类是C++对象和TJS2对象之间的桥梁：

```cpp
// tjs2TimerNI.h - Timer的原生实例类
// NI = Native Instance，KrKr2的命名惯例
#include "tjsNative.h"
#include "tjs2Timer.h"

// 原生实例类：每个TJS2 Timer对象持有一个此实例
// 继承自tTJSNativeInstance（即iTJSNativeInstance的默认实现）
// tTJSNativeInstance提供三个虚函数：
//   Construct() - 构造时调用，接收TJS2构造参数
//   Invalidate() - 失效时调用（GC回收前或显式invalidate）
//   Destruct() - C++析构（释放内存）
class tTJS2TimerNI : public tTJSNativeInstance {
public:
    tTJS2Timer* timer_;  // 持有的C++计时器对象

    tTJS2TimerNI() : timer_(nullptr) {
        // 注意：不在这里创建timer_
        // 创建在Construct()中完成，因为需要TJS2构造参数
    }

    // Construct：TJS2的new操作触发
    // numparams/param: TJS2构造函数的参数
    // tjs: TJS2引擎实例指针
    tjs_error TJS_INTF_METHOD Construct(
        tjs_int numparams,
        tTJSVariant **param,
        iTJSDispatch2 *tjs) override
    {
        // 如果有参数，第一个参数作为标签
        if (numparams >= 1 && param[0]->Type() == tvtString) {
            timer_ = new tTJS2Timer(
                ttstr(*param[0]).c_str()  // ttstr: TJS2字符串类型
            );
        } else {
            timer_ = new tTJS2Timer();
        }
        return TJS_S_OK;  // 返回成功
    }

    // Invalidate：对象被标记为无效时调用
    // 可能在GC回收前、或脚本调用invalidate()时触发
    // 应释放资源但不释放自身内存（Destruct负责）
    void TJS_INTF_METHOD Invalidate() override {
        if (timer_) {
            delete timer_;
            timer_ = nullptr;
        }
    }

    // Destruct：C++析构
    // 由tTJSNativeInstance基类默认提供delete this
    // 通常不需要重写，除非有特殊清理需求
    // ~tTJS2TimerNI() 默认即可
};
```

> **生命周期要点**：`Construct()`创建资源 → `Invalidate()`释放资源 → `Destruct()`释放NI自身。`Invalidate()`可能被调用多次（虽然实际中很少），所以需要检查`timer_`是否为空。这是KrKr2中所有原生实例类的标准模式。

### 2.3 原生类定义与成员注册

这是低级宏方式的核心——使用`TJS_BEGIN_NATIVE_MEMBERS`系列宏注册所有方法和属性：

```cpp
// tjs2TimerClass.cpp - Timer类的TJS2绑定
#include "tjsNative.h"
#include "tjsArray.h"  // 参考Array的注册模式
#include "tjs2TimerNI.h"

// 定义全局ClassID变量（每个原生类必须有一个）
// ClassID在TJS_BEGIN_NATIVE_MEMBERS中被赋值
tjs_int32 tTJS2TimerClassID = -1;

// === 原生类定义 ===
// 继承自tTJSNativeClass，提供类级别的行为
class tTJS2TimerClass : public tTJSNativeClass {
    // typedef简化后续代码
    typedef tTJSNativeClass inherited;

public:
    tTJS2TimerClass()
        : tTJSNativeClass(TJS_W("Timer"))
        // TJS_W("Timer") 是TJS2的宽字符串宏
        // 这个名字就是TJS2脚本中使用的类名
    {
        // ============================================
        // TJS_BEGIN_NATIVE_MEMBERS 宏展开后做以下事情：
        // 1. 调用 TJSRegisterNativeClass("Timer")
        //    分配ClassID并存入tTJS2TimerClassID
        // 2. 开始一个成员注册块
        // ============================================
        TJS_BEGIN_NATIVE_MEMBERS(Timer)

        // ---- finalize方法（必须） ----
        // 每个原生类都必须有finalize方法
        // GC回收对象时自动调用
        TJS_DECL_EMPTY_FINALIZE_METHOD

        // ---- 构造函数 ----
        TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL(
            /* var. name  */ _this,
            /* var. type  */ tTJS2TimerNI,
            /* class name */ Timer)
        {
            // _this 是 tTJS2TimerNI* 类型
            // numparams, param 自动可用
            // 构造工作委托给NI的Construct方法
            // （已在tTJS2TimerNI::Construct中实现）
            return TJS_S_OK;
        }
        TJS_END_NATIVE_CONSTRUCTOR_DECL(Timer)

        // ---- 实例方法：start() ----
        TJS_BEGIN_NATIVE_METHOD_DECL(/* func. name */ start)
        {
            // TJS_GET_NATIVE_INSTANCE 宏：
            // 1. 从objthis获取ClassInstanceInfo
            // 2. 用tTJS2TimerClassID查找NI指针
            // 3. 失败则返回TJS_E_NATIVECLASSCRASH
            TJS_GET_NATIVE_INSTANCE(
                /* var. name */ _this,
                /* var. type */ tTJS2TimerNI);

            _this->timer_->start();

            // result是tTJSVariant*，可选的返回值
            // 如果调用方不需要返回值，result为NULL
            if (result) *result = tTJSVariant();  // void返回
            return TJS_S_OK;
        }
        TJS_END_NATIVE_METHOD_DECL(/* func. name */ start)

        // ---- 实例方法：stop() → 返回double ----
        TJS_BEGIN_NATIVE_METHOD_DECL(stop)
        {
            TJS_GET_NATIVE_INSTANCE(_this, tTJS2TimerNI);
            double elapsed = _this->timer_->stop();
            if (result) *result = tTJSVariant(elapsed);
            return TJS_S_OK;
        }
        TJS_END_NATIVE_METHOD_DECL(stop)

        // ---- 实例方法：pause() ----
        TJS_BEGIN_NATIVE_METHOD_DECL(pause)
        {
            TJS_GET_NATIVE_INSTANCE(_this, tTJS2TimerNI);
            _this->timer_->pause();
            if (result) *result = tTJSVariant();
            return TJS_S_OK;
        }
        TJS_END_NATIVE_METHOD_DECL(pause)

        // ---- 实例方法：resume() ----
        TJS_BEGIN_NATIVE_METHOD_DECL(resume)
        {
            TJS_GET_NATIVE_INSTANCE(_this, tTJS2TimerNI);
            _this->timer_->resume();
            if (result) *result = tTJSVariant();
            return TJS_S_OK;
        }
        TJS_END_NATIVE_METHOD_DECL(resume)

        // ---- 实例方法：elapsed() → 返回double ----
        TJS_BEGIN_NATIVE_METHOD_DECL(elapsed)
        {
            TJS_GET_NATIVE_INSTANCE(_this, tTJS2TimerNI);
            double ms = _this->timer_->elapsed();
            if (result) *result = tTJSVariant(ms);
            return TJS_S_OK;
        }
        TJS_END_NATIVE_METHOD_DECL(elapsed)

        // ---- 实例方法：reset() ----
        TJS_BEGIN_NATIVE_METHOD_DECL(reset)
        {
            TJS_GET_NATIVE_INSTANCE(_this, tTJS2TimerNI);
            _this->timer_->reset();
            if (result) *result = tTJSVariant();
            return TJS_S_OK;
        }
        TJS_END_NATIVE_METHOD_DECL(reset)

        // ---- 属性：running（只读） ----
        TJS_BEGIN_NATIVE_PROP_DECL(running)
        {
            // PropGet回调
            TJS_BEGIN_NATIVE_PROP_GETTER
                TJS_GET_NATIVE_INSTANCE(_this, tTJS2TimerNI);
                *result = tTJSVariant(
                    (tjs_int)_this->timer_->isRunning());
                return TJS_S_OK;
            TJS_END_NATIVE_PROP_GETTER

            // PropSet回调——拒绝设置
            TJS_DENY_NATIVE_PROP_SETTER
        }
        TJS_END_NATIVE_PROP_DECL(running)

        // ---- 属性：paused（只读） ----
        TJS_BEGIN_NATIVE_PROP_DECL(paused)
        {
            TJS_BEGIN_NATIVE_PROP_GETTER
                TJS_GET_NATIVE_INSTANCE(_this, tTJS2TimerNI);
                *result = tTJSVariant(
                    (tjs_int)_this->timer_->isPaused());
                return TJS_S_OK;
            TJS_END_NATIVE_PROP_GETTER

            TJS_DENY_NATIVE_PROP_SETTER
        }
        TJS_END_NATIVE_PROP_DECL(paused)

        // ---- 属性：label（读写） ----
        TJS_BEGIN_NATIVE_PROP_DECL(label)
        {
            TJS_BEGIN_NATIVE_PROP_GETTER
                TJS_GET_NATIVE_INSTANCE(_this, tTJS2TimerNI);
                *result = tTJSVariant(
                    ttstr(_this->timer_->getLabel().c_str()));
                return TJS_S_OK;
            TJS_END_NATIVE_PROP_GETTER

            TJS_BEGIN_NATIVE_PROP_SETTER
                TJS_GET_NATIVE_INSTANCE(_this, tTJS2TimerNI);
                _this->timer_->setLabel(
                    ttstr(*param).c_str());
                return TJS_S_OK;
            TJS_END_NATIVE_PROP_SETTER
        }
        TJS_END_NATIVE_PROP_DECL(label)

        // ---- 静态方法：getInstanceCount() ----
        // 使用 TJS_END_NATIVE_STATIC_METHOD_DECL
        // 静态方法不绑定到实例，不能用TJS_GET_NATIVE_INSTANCE
        TJS_BEGIN_NATIVE_METHOD_DECL(getInstanceCount)
        {
            // 不需要TJS_GET_NATIVE_INSTANCE——这是静态方法
            if (result) *result = tTJSVariant(
                (tjs_int)tTJS2Timer::getInstanceCount());
            return TJS_S_OK;
        }
        TJS_END_NATIVE_STATIC_METHOD_DECL(getInstanceCount)
        // 注意：使用 _STATIC_ 版本的END宏

        // ---- 静态方法：getResolution() ----
        TJS_BEGIN_NATIVE_METHOD_DECL(getResolution)
        {
            if (result) *result = tTJSVariant(
                tTJS2Timer::getResolution());
            return TJS_S_OK;
        }
        TJS_END_NATIVE_STATIC_METHOD_DECL(getResolution)

        // ============================================
        // TJS_END_NATIVE_MEMBERS 宏结束成员注册块
        // ============================================
        TJS_END_NATIVE_MEMBERS

    }  // 构造函数结束

    // CreateNativeInstance：工厂方法
    // tTJSNativeClass::FuncCall内部调用此方法创建NI
    tTJSNativeInstance* CreateNativeInstance() override {
        return new tTJS2TimerNI();
    }

    // 静态工厂方法：创建并注册Timer类
    static tTJS2TimerClass* Create() {
        tTJS2TimerClass* cls = new tTJS2TimerClass();
        return cls;
    }
};
```

> **宏展开对照**：上面代码中的`TJS_BEGIN_NATIVE_METHOD_DECL(start)`展开后等价于：
> ```cpp
> static tTJSNativeClassMethodCallback TJS_start_body;
> // 创建 tTJSNativeClassMethod 包装器
> struct TJS_NCM_start : public tTJSNativeClassMethod {
>     TJS_NCM_start() : tTJSNativeClassMethod(&TJS_start_body) {}
> } *TJS_NCM_start_obj = new TJS_NCM_start();
> static tjs_error TJS_start_body(
>     tTJSVariant *result,
>     tjs_int numparams,
>     tTJSVariant **param,
>     iTJSDispatch2 *objthis)
> {
>     // 你写的函数体在这里
> }
> ```
> 理解宏展开有助于调试编译错误——当编译器报错指向宏内部时，知道实际生成的代码结构非常重要。

### 2.4 注册到全局并在TJS2中使用

类定义完成后，需要将其注册到TJS2全局对象，使脚本代码可以通过类名访问：

```cpp
// tjs2TimerRegistration.cpp - 注册Timer类到TJS2全局

#include "tjs2TimerClass.cpp"  // 实际项目中用头文件
#include "tjsGlobal.h"

// 全局Timer类对象指针
static iTJSDispatch2* TimerClassObj = nullptr;

// 注册函数：在TJS2引擎初始化时调用
void RegisterTimerClass() {
    // 1. 创建Timer类对象
    tTJS2TimerClass* cls = tTJS2TimerClass::Create();

    // 2. 获取TJS2全局对象
    iTJSDispatch2* global = TJSGetGlobalObject();
    if (!global) {
        cls->Release();
        return;
    }

    // 3. 将Timer类注册到全局命名空间
    // PropSetByVS: 通过字符串设置属性
    // TJS_MEMBERENSURE: 如果成员不存在则创建
    // TJS_IGNOREPROP: 忽略属性getter/setter，直接设置
    tTJSVariant val(cls, cls);  // 创建引用cls的Variant
    tjs_error hr = global->PropSetByVS(
        TJS_MEMBERENSURE | TJS_IGNOREPROP,
        TJS_W("Timer"),           // 全局变量名
        &val,                      // 值
        global                     // context对象
    );

    // 4. Release引用
    // PropSetByVS内部会AddRef，所以这里Release不会销毁对象
    cls->Release();
    TimerClassObj = cls;  // 保存指针，供注销时使用

    if (TJS_FAILED(hr)) {
        // 注册失败的错误处理
        TVPThrowExceptionMessage(
            TJS_W("Cannot register Timer class"));
    }
}

// 注销函数：在TJS2引擎关闭时调用
void UnregisterTimerClass() {
    if (TimerClassObj) {
        // 从全局对象中删除Timer
        iTJSDispatch2* global = TJSGetGlobalObject();
        if (global) {
            global->DeleteMember(
                0,
                TJS_W("Timer"),
                nullptr,
                global
            );
        }
        TimerClassObj = nullptr;
    }
}
```

注册完成后，TJS2脚本中就可以使用Timer类了：

```javascript
// TJS2脚本 - 使用Timer类
var timer = new Timer("benchmark");  // 创建实例，传入标签

timer.start();                        // 开始计时

// ... 执行需要计时的操作 ...
for (var i = 0; i < 10000; i++) {
    var x = Math.sqrt(i);
}

var elapsed = timer.stop();           // 停止并获取毫秒数
System.inform("耗时: " + elapsed + " ms");

// 访问属性
System.inform("标签: " + timer.label);
System.inform("运行中: " + timer.running);  // false（已stop）

// 修改属性
timer.label = "test2";

// 暂停/恢复
timer.start();
timer.pause();
// ... 做其他事 ...
timer.resume();
var total = timer.stop();

// 静态方法
System.inform("实例数: " + Timer.getInstanceCount());
System.inform("时钟精度: " + Timer.getResolution() + " ns");

// 对象自动清理：离开作用域后GC会调用finalize→Invalidate
```

### 2.5 低级宏方式中的参数处理

低级宏方式最具挑战的部分是参数处理。以下是各种参数场景的处理模式：

```cpp
// === 示例1：带参数的方法 ===
// TJS2: timer.format(precision, unit)
TJS_BEGIN_NATIVE_METHOD_DECL(format)
{
    TJS_GET_NATIVE_INSTANCE(_this, tTJS2TimerNI);

    // 参数检查：numparams是实际传入的参数数量
    if (numparams < 1)
        return TJS_E_BADPARAMCOUNT;  // 参数不足

    // 获取参数：param[0], param[1], ...
    // 每个param[i]是tTJSVariant*

    // 获取整数参数（精度）
    tjs_int precision = 2;  // 默认值
    if (numparams >= 1 && param[0]->Type() != tvtVoid) {
        precision = (tjs_int)*param[0];
        // 强制转换：tTJSVariant的operator tjs_int()
    }

    // 获取字符串参数（单位），可选
    ttstr unit = TJS_W("ms");  // 默认值
    if (numparams >= 2 && param[1]->Type() != tvtVoid) {
        unit = ttstr(*param[1]);
        // ttstr构造：从tTJSVariant提取字符串
    }

    // TJS_PARAM_EXIST宏：检查参数是否存在且非void
    // 等价于 (numparams > n && param[n]->Type() != tvtVoid)

    // 构造格式化结果
    double ms = _this->timer_->elapsed();
    tjs_char buf[256];
    // ... 格式化逻辑 ...

    if (result) *result = tTJSVariant(ttstr(buf));
    return TJS_S_OK;
}
TJS_END_NATIVE_METHOD_DECL(format)

// === 示例2：可变参数方法 ===
// TJS2: timer.log(msg1, msg2, ...)
TJS_BEGIN_NATIVE_METHOD_DECL(log)
{
    TJS_GET_NATIVE_INSTANCE(_this, tTJS2TimerNI);

    // 遍历所有参数
    ttstr output;
    for (tjs_int i = 0; i < numparams; i++) {
        if (i > 0) output += TJS_W(" ");

        // 根据类型分别处理
        switch (param[i]->Type()) {
            case tvtVoid:
                output += TJS_W("(void)");
                break;
            case tvtInteger:
                output += ttstr((tjs_int)*param[i]);
                break;
            case tvtReal:
                output += ttstr((tjs_real)*param[i]);
                break;
            case tvtString:
                output += ttstr(*param[i]);
                break;
            case tvtObject: {
                // 对象参数：可以调用其toString
                iTJSDispatch2* obj = param[i]->AsObjectNoAddRef();
                if (obj) {
                    tTJSVariant strVal;
                    // 尝试调用toString方法
                    tjs_error hr = obj->FuncCall(
                        0, TJS_W("toString"),
                        nullptr, &strVal, 0, nullptr, obj);
                    if (TJS_SUCCEEDED(hr))
                        output += ttstr(strVal);
                    else
                        output += TJS_W("[object]");
                }
                break;
            }
            default:
                output += TJS_W("[unknown]");
                break;
        }
    }

    // 输出日志
    TVPAddLog(output);

    if (result) *result = tTJSVariant();
    return TJS_S_OK;
}
TJS_END_NATIVE_METHOD_DECL(log)

// === 示例3：返回对象的方法 ===
// TJS2: var info = timer.getInfo()  // 返回Dictionary
TJS_BEGIN_NATIVE_METHOD_DECL(getInfo)
{
    TJS_GET_NATIVE_INSTANCE(_this, tTJS2TimerNI);

    // 创建新的Dictionary对象
    iTJSDispatch2* dict = TJSCreateDictionaryObject();

    // 设置属性
    tTJSVariant val;

    val = tTJSVariant(_this->timer_->elapsed());
    dict->PropSetByVS(
        TJS_MEMBERENSURE, TJS_W("elapsed"),
        &val, dict);

    val = tTJSVariant(
        (tjs_int)_this->timer_->isRunning());
    dict->PropSetByVS(
        TJS_MEMBERENSURE, TJS_W("running"),
        &val, dict);

    val = tTJSVariant(
        ttstr(_this->timer_->getLabel().c_str()));
    dict->PropSetByVS(
        TJS_MEMBERENSURE, TJS_W("label"),
        &val, dict);

    // 返回Dictionary
    // tTJSVariant(obj, obj) 创建对象引用并AddRef
    if (result) *result = tTJSVariant(dict, dict);
    dict->Release();  // 平衡Create时的引用

    return TJS_S_OK;
}
TJS_END_NATIVE_METHOD_DECL(getInfo)
```

> **类型转换速查表**：在低级宏方式中，tTJSVariant和C++类型之间的转换是手动的：
>
> | C++类型 | → tTJSVariant | tTJSVariant → C++ |
> |---------|--------------|-------------------|
> | `tjs_int` | `tTJSVariant(intVal)` | `(tjs_int)*param[i]` |
> | `tjs_real`（double） | `tTJSVariant(realVal)` | `(tjs_real)*param[i]` |
> | `ttstr`（字符串） | `tTJSVariant(ttstr(...))` | `ttstr(*param[i])` |
> | `bool` | `tTJSVariant((tjs_int)boolVal)` | `(tjs_int)*param[i] != 0` |
> | `iTJSDispatch2*` | `tTJSVariant(obj, obj)` | `param[i]->AsObjectNoAddRef()` |
> | `void`（无返回值） | `tTJSVariant()`或不设置 | N/A |

---

## 3. ncbind方式完整实战

同样以Timer类为例，演示使用ncbind从零开始绑定的完整过程。对比低级宏方式，代码量将大幅减少。

### 3.1 C++类（复用）

ncbind方式使用**完全相同的C++类**（`tTJS2Timer`），无需修改。这是ncbind的核心优势之一：C++类不需要知道TJS2的存在。

但有一个重要约束：ncbind通过模板推导自动匹配参数，所以**C++方法的签名决定了TJS2端的参数类型**。

### 3.2 ncbind注册（完整代码）

```cpp
// tjs2TimerPlugin.cpp - 使用ncbind绑定Timer类
// 这是一个完整的插件文件

#include "ncbind.hpp"
#include "tjs2Timer.h"

// ============================================================
// NCB_REGISTER_CLASS 宏一行解决：
// 1. 定义 ncbClassInfo<tTJS2Timer> 的静态成员
// 2. 创建 ncbAutoRegister 子类，链入自动注册链
// 3. 在V2Link时自动调用内部的注册代码
// ============================================================
NCB_REGISTER_CLASS(tTJS2Timer)
{
    // --- 设置TJS2中的类名 ---
    // 如果不调用NCB_REGISTER_CLASS_DIFFER，
    // TJS2中的类名就是C++类名"tTJS2Timer"
    // 但我们想在TJS2中叫"Timer"，所以：
    // 实际上应该用 NCB_REGISTER_CLASS_DIFFER 来指定不同名称
    // NCB_REGISTER_CLASS_DIFFER(tTJS2Timer, Timer) { ... }

    // --- 构造函数 ---
    // Constructor() 无参数：只支持默认构造
    // Constructor<const std::wstring&>()：支持一个wstring参数
    Constructor<const std::wstring&>(0);
    // 参数0表示：前0个参数是必需的（即全部可选）
    // 这样TJS2端可以 new Timer() 或 new Timer("label")

    // --- 实例方法 ---
    // NCB_METHOD(TJS2方法名, C++方法指针)
    // ncbind自动推导参数类型和返回值类型
    NCB_METHOD(start,   &tTJS2Timer::start);
    NCB_METHOD(stop,    &tTJS2Timer::stop);
    NCB_METHOD(pause,   &tTJS2Timer::pause);
    NCB_METHOD(resume,  &tTJS2Timer::resume);
    NCB_METHOD(elapsed, &tTJS2Timer::elapsed);
    NCB_METHOD(reset,   &tTJS2Timer::reset);

    // --- 属性 ---
    // NCB_PROPERTY_RO(TJS2属性名, C++ getter方法)
    // RO = Read Only，只读属性
    NCB_PROPERTY_RO(running, &tTJS2Timer::isRunning);
    NCB_PROPERTY_RO(paused,  &tTJS2Timer::isPaused);

    // NCB_PROPERTY(TJS2属性名, getter, setter)
    // 读写属性
    NCB_PROPERTY(label,
        &tTJS2Timer::getLabel,
        &tTJS2Timer::setLabel);

    // --- 静态方法 ---
    // 用普通函数指针（不是成员函数指针）注册静态方法
    // 方法1：直接绑定静态成员函数
    NCB_METHOD(getInstanceCount,
        &tTJS2Timer::getInstanceCount);
    NCB_METHOD(getResolution,
        &tTJS2Timer::getResolution);
}
```

如果想让TJS2中的类名与C++类名不同（通常都需要这样做），使用`NCB_REGISTER_CLASS_DIFFER`：

```cpp
// 推荐方式：C++类名tTJS2Timer，TJS2中叫Timer
NCB_REGISTER_CLASS_DIFFER(tTJS2Timer, Timer)
{
    Constructor<const std::wstring&>(0);
    NCB_METHOD(start,   &tTJS2Timer::start);
    NCB_METHOD(stop,    &tTJS2Timer::stop);
    NCB_METHOD(pause,   &tTJS2Timer::pause);
    NCB_METHOD(resume,  &tTJS2Timer::resume);
    NCB_METHOD(elapsed, &tTJS2Timer::elapsed);
    NCB_METHOD(reset,   &tTJS2Timer::reset);
    NCB_PROPERTY_RO(running, &tTJS2Timer::isRunning);
    NCB_PROPERTY_RO(paused,  &tTJS2Timer::isPaused);
    NCB_PROPERTY(label,
        &tTJS2Timer::getLabel,
        &tTJS2Timer::setLabel);
    NCB_METHOD(getInstanceCount,
        &tTJS2Timer::getInstanceCount);
    NCB_METHOD(getResolution,
        &tTJS2Timer::getResolution);
}
```

> **代码量对比**：低级宏方式的Timer绑定需要约200行代码（NI类 + 类定义 + 所有方法/属性宏）。ncbind方式只需约20行。这就是ncbind存在的意义。

### 3.3 ncbind的自动注册流程

当插件被加载时，ncbind的注册自动发生。以下是完整的调用链：

```
插件加载入口 V2Link()
  └─ ncbAutoRegister::AllRegist(true)
       ├─ Phase 1: PreRegist — NCB_PRE_REGIST_CALLBACK
       ├─ Phase 2: ClassRegist — NCB_REGISTER_CLASS
       │    └─ ncbRegistNativeClass<tTJS2Timer>::RegistBegin()
       │         ├─ TJSCreateNativeClassForPlugin("Timer")
       │         │    └─ 创建 tTJSNativeClassForPlugin 对象
       │         │         └─ TJSRegisterNativeClass("Timer")
       │         │              └─ 分配ClassID
       │         ├─ 设置 procCreateNativeInstance 回调
       │         │    └─ 回调内创建 ncbInstanceAdaptor<tTJS2Timer>
       │         └─ RegisterNCM 注册每个方法/属性
       │              ├─ Method("start", ncbNativeClassMethod<...>)
       │              ├─ Method("stop", ncbNativeClassMethod<...>)
       │              ├─ Property("running", ncbNativeClassProperty<...>)
       │              └─ ...
       └─ Phase 3: PostRegist — NCB_POST_REGIST_CALLBACK

插件卸载入口 V2Unlink()
  └─ ncbAutoRegister::AllRegist(false)
       └─ ncbRegistNativeClass<tTJS2Timer>::UnregistEnd()
            └─ 从全局对象删除"Timer"成员
```

### 3.4 ncbind中的RawCallback

当ncbind的自动参数推导无法满足需求时（如可变参数、特殊返回值处理），可以使用RawCallback退回到手动模式：

```cpp
// RawCallback示例：实现可变参数的log方法
// 签名固定为：
// tjs_error callback(
//     tTJSVariant *result,
//     tjs_int numparams,
//     tTJSVariant **param,
//     iTJSDispatch2 *objthis)
static tjs_error TJS_INTF_METHOD
TimerLogCallback(
    tTJSVariant *result,
    tjs_int numparams,
    tTJSVariant **param,
    iTJSDispatch2 *objthis)
{
    // 通过ncbInstanceAdaptor获取C++实例
    // 这是ncbind方式获取NI的标准方法
    tTJS2Timer* timer =
        ncbInstanceAdaptor<tTJS2Timer>::GetNativeInstance(
            objthis);
    if (!timer)
        return TJS_E_NATIVECLASSCRASH;

    // 手动处理参数（与低级宏方式相同）
    ttstr output;
    for (tjs_int i = 0; i < numparams; i++) {
        if (i > 0) output += TJS_W(" ");
        output += ttstr(*param[i]);
    }

    TVPAddLog(output);

    if (result) *result = tTJSVariant();
    return TJS_S_OK;
}

// 在NCB_REGISTER_CLASS中注册RawCallback
NCB_REGISTER_CLASS_DIFFER(tTJS2Timer, Timer)
{
    Constructor<const std::wstring&>(0);
    NCB_METHOD(start, &tTJS2Timer::start);
    // ... 其他方法 ...

    // RawCallback注册
    RawCallback("log", &TimerLogCallback, 0);
    // 第三个参数0表示不需要特殊标志
}
```

---

## 4. 真实代码库案例深度分析

### 4.1 案例一：tjsArray.cpp（低级宏方式）

tjsArray.cpp是KrKr2中最复杂的内置类之一，使用低级宏方式注册。以下分析其关键模式：

```
源文件：cpp/core/tjs2/tjsArray.cpp
行数：约2000行
类层次：
  tTJSArrayNI (NativeInstance)
    └─ 持有 tTJSArrayData（实际的数组数据）
  tTJSArrayObject (NativeClass)
    └─ 继承 tTJSNativeClass
    └─ 注册约30个方法和属性
```

**关键代码片段（方法注册）**：

```cpp
// 来源：cpp/core/tjs2/tjsArray.cpp
// Array类的push方法注册

TJS_BEGIN_NATIVE_METHOD_DECL(push)
{
    TJS_GET_NATIVE_INSTANCE(_this, tTJSArrayNI);

    // push支持可变参数：array.push(1, 2, 3)
    // 遍历所有参数，逐个添加到数组末尾
    for (tjs_int i = 0; i < numparams; i++) {
        _this->Items.push_back(*param[i]);
        // Items是std::vector<tTJSVariant>
    }

    // 返回新的数组长度
    if (result) *result = tTJSVariant(
        (tjs_int)_this->Items.size());
    return TJS_S_OK;
}
TJS_END_NATIVE_METHOD_DECL(push)

// Array的count属性（类似JavaScript的length）
TJS_BEGIN_NATIVE_PROP_DECL(count)
{
    TJS_BEGIN_NATIVE_PROP_GETTER
        TJS_GET_NATIVE_INSTANCE(_this, tTJSArrayNI);
        *result = tTJSVariant(
            (tjs_int)_this->Items.size());
        return TJS_S_OK;
    TJS_END_NATIVE_PROP_GETTER

    TJS_BEGIN_NATIVE_PROP_SETTER
        TJS_GET_NATIVE_INSTANCE(_this, tTJSArrayNI);
        // 设置count会resize数组
        _this->Items.resize((tjs_int)*param);
        return TJS_S_OK;
    TJS_END_NATIVE_PROP_SETTER
}
TJS_END_NATIVE_PROP_DECL(count)
```

**从Array类学到的模式**：
1. **可变参数**：直接遍历`param`数组，`numparams`控制上限
2. **读写属性**：`count`属性既可读（返回size）又可写（调用resize）
3. **返回新长度**：与JavaScript的`Array.push()`行为一致
4. **类型灵活**：数组元素是`tTJSVariant`，无需类型检查

### 4.2 案例二：tjsMath.cpp（纯静态类）

Math类是一个特殊案例——它没有实例方法，所有成员都是静态的：

```
源文件：cpp/core/tjs2/tjsMath.cpp
特点：
  - 没有tTJSNativeInstance子类
  - 所有方法使用TJS_END_NATIVE_STATIC_METHOD_DECL
  - 所有属性是静态常量（PI, E等）
  - 不支持new Math()——纯命名空间
```

```cpp
// 来源：cpp/core/tjs2/tjsMath.cpp
// Math类的构造部分

TJS_BEGIN_NATIVE_MEMBERS(Math)

    TJS_DECL_EMPTY_FINALIZE_METHOD

    // 使用 _NO_INSTANCE 版本的构造函数宏
    // 这意味着不创建NI，new Math()会抛出异常
    TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL_NO_INSTANCE(Math)
    {
        return TJS_S_OK;
    }
    TJS_END_NATIVE_CONSTRUCTOR_DECL(Math)

    // 静态方法：Math.abs()
    TJS_BEGIN_NATIVE_METHOD_DECL(abs)
    {
        if (numparams < 1)
            return TJS_E_BADPARAMCOUNT;
        if (result) {
            // 根据参数类型选择整数或浮点abs
            switch (param[0]->Type()) {
                case tvtReal:
                    *result = tTJSVariant(
                        std::abs((tjs_real)*param[0]));
                    break;
                default:
                    *result = tTJSVariant(
                        std::abs((tjs_int)*param[0]));
                    break;
            }
        }
        return TJS_S_OK;
    }
    TJS_END_NATIVE_STATIC_METHOD_DECL(abs)
    // ^^^ _STATIC_ 版本：不绑定objthis

    // 静态只读属性：Math.PI
    TJS_BEGIN_NATIVE_PROP_DECL(PI)
    {
        TJS_BEGIN_NATIVE_PROP_GETTER
            *result = tTJSVariant(
                (tjs_real)3.14159265358979323846);
            return TJS_S_OK;
        TJS_END_NATIVE_PROP_GETTER

        TJS_DENY_NATIVE_PROP_SETTER
    }
    TJS_END_NATIVE_PROP_DECL(PI)

TJS_END_NATIVE_MEMBERS
```

**从Math类学到的模式**：
1. **纯静态类**：使用`TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL_NO_INSTANCE`
2. **类型感知**：`abs()`根据参数类型返回不同精度的结果
3. **常量属性**：使用只读属性+`TJS_DENY_NATIVE_PROP_SETTER`
4. **所有方法都是STATIC**：不需要`TJS_GET_NATIVE_INSTANCE`

### 4.3 案例三：steam_api.cpp（ncbind方式，最小示例）

这是整个KrKr2代码库中唯一使用`NCB_REGISTER_CLASS`的文件：

```cpp
// 来源：cpp/plugins/steam/steam_api.cpp
// 完整文件——只有12行

#include "ncbind.hpp"

class steam_api {
public:
    steam_api() {}
};

NCB_REGISTER_CLASS(steam_api) {
    Constructor();
    // 没有任何方法或属性
    // 这是一个空壳类——Steam功能在build中被禁用
}
```

**从steam_api学到的模式**：
1. **最小绑定**：只需`Constructor()`即可创建可实例化的TJS2类
2. **C++类无依赖**：`steam_api`类不包含任何TJS2头文件
3. **自动注册**：无需手动调用注册函数，`NCB_REGISTER_CLASS`自动处理
4. **类名即TJS2名**：未使用`DIFFER`变体，TJS2中类名也是`steam_api`

---

## 5. 类型转换详解与自定义转换器

### 5.1 ncbind的自动类型转换

ncbind通过`ncbTypeConvertor`模板系统自动处理C++和TJS2之间的类型转换。以下是内置支持的类型映射：

```
┌─────────────────────────────────────────────────────┐
│              ncbind 自动类型转换表                    │
├─────────────────┬───────────────────────────────────┤
│ C++ 类型         │ TJS2 类型                         │
├─────────────────┼───────────────────────────────────┤
│ int, long,       │ tvtInteger (tjs_int)              │
│ short, char,     │   通过 NCB_TYPECONV_CAST_INTEGER  │
│ tjs_int 等       │   静态转换                        │
├─────────────────┼───────────────────────────────────┤
│ float, double,   │ tvtReal (tjs_real)                │
│ tjs_real         │   通过 NCB_TYPECONV_CAST_REAL     │
│                  │   静态转换                        │
├─────────────────┼───────────────────────────────────┤
│ bool             │ tvtInteger                        │
│                  │   true→1, false→0                 │
├─────────────────┼───────────────────────────────────┤
│ const char*,     │ tvtString                         │
│ std::string      │   通过 ncbNarrowCharConvertor     │
│                  │   UTF-8 ↔ UTF-16 自动转码         │
├─────────────────┼───────────────────────────────────┤
│ const wchar_t*,  │ tvtString                         │
│ std::wstring     │   通过 ncbWideCharConvertor       │
│                  │   直接传递（TJS2内部用宽字符）     │
├─────────────────┼───────────────────────────────────┤
│ tTJSVariant      │ 任意类型                          │
│                  │   透传，不做转换                   │
├─────────────────┼───────────────────────────────────┤
│ iTJSDispatch2*   │ tvtObject                         │
│                  │   通过 NCB_TYPECONV_BOXING         │
│                  │   自动装箱/拆箱                    │
├─────────────────┼───────────────────────────────────┤
│ tTJSVariantClosure│ tvtObject                        │
│                  │   闭包转换                        │
└─────────────────┴───────────────────────────────────┘
```

### 5.2 常见类型转换陷阱

```cpp
// === 陷阱1：char*编码问题 ===
class MyClass {
public:
    // 错误：const char* 在不同平台编码不同
    void setName(const char* name) { /* ... */ }

    // 正确：使用wchar_t*或std::wstring
    void setName(const std::wstring& name) { /* ... */ }

    // 或者使用tjs_char（KrKr2的跨平台字符类型）
    void setName(const tjs_char* name) { /* ... */ }
};

// === 陷阱2：返回引用的生命周期 ===
class MyClass {
public:
    // 危险：返回内部引用
    // 如果C++对象被销毁，TJS2端持有的字符串变成悬垂引用
    const std::wstring& getName() const { return name_; }

    // 更安全的做法：返回值（ncbind会复制）
    std::wstring getName() const { return name_; }
};

// === 陷阱3：整数溢出 ===
// TJS2的整数是64位(tjs_int = __int64/long long)
// 但C++方法参数可能是32位int
class MyClass {
public:
    void setSize(int size) { /* 如果TJS2传入>2^31的值？截断！ */ }
    void setSize(tjs_int size) { /* 安全：64位整数 */ }
};

// === 陷阱4：void返回值 ===
// ncbind正确处理void返回——不设置result
// 但如果C++方法返回bool而你期望void，会出现意外的TJS2返回值
class MyClass {
public:
    bool start() { /* ... */ return true; }
    // TJS2端: var ret = obj.start();  // ret == 1，可能不是预期行为
};
```

### 5.3 自定义类型转换器

当需要支持ncbind不内置的类型时，可以通过特化模板添加自定义转换：

```cpp
// 自定义转换器示例：支持std::vector<int>
// 将C++的vector<int>与TJS2的Array互相转换

// 方法1：通过ncbTypeConvertor特化
namespace TJS {

// C++ vector<int> → TJS2 Array
template<>
struct ncbTypeConvertor::Conversion<
    std::vector<int>,      // FROM
    tTJSVariant>           // TO
{
    static void Convert(
        const std::vector<int>& from,
        tTJSVariant& to)
    {
        // 创建TJS2 Array
        iTJSDispatch2* arr = TJSCreateArrayObject();

        for (size_t i = 0; i < from.size(); i++) {
            tTJSVariant val((tjs_int)from[i]);
            arr->PropSetByNum(
                TJS_MEMBERENSURE,
                (tjs_int)i,
                &val,
                arr);
        }

        to = tTJSVariant(arr, arr);
        arr->Release();
    }
};

// TJS2 Array → C++ vector<int>
template<>
struct ncbTypeConvertor::Conversion<
    tTJSVariant,           // FROM
    std::vector<int>>      // TO
{
    static void Convert(
        const tTJSVariant& from,
        std::vector<int>& to)
    {
        iTJSDispatch2* arr =
            from.AsObjectNoAddRef();
        if (!arr) return;

        // 获取数组长度
        tTJSVariant countVal;
        arr->PropGet(0, TJS_W("count"),
            nullptr, &countVal, arr);
        tjs_int count = (tjs_int)countVal;

        to.resize(count);
        for (tjs_int i = 0; i < count; i++) {
            tTJSVariant elemVal;
            arr->PropGetByNum(0, i, &elemVal, arr);
            to[i] = (int)(tjs_int)elemVal;
        }
    }
};

}  // namespace TJS
```

```cpp
// 方法2：使用ncbPropAccessor作为中间层
// 更简单但效率略低的方式
class DataProcessor {
public:
    // 直接接受ncbPropAccessor参数
    // ncbPropAccessor封装了iTJSDispatch2*的常见操作
    void processArray(ncbPropAccessor* arr) {
        tjs_int count = arr->GetValue(
            TJS_W("count"), 0);

        for (tjs_int i = 0; i < count; i++) {
            tjs_int val = arr->GetValue(i, 0);
            // 处理val...
        }
    }

    // 或者使用tTJSVariant参数，手动解包
    void processData(tTJSVariant data) {
        // data可以是任意TJS2类型
        // 在方法内部根据Type()分别处理
        if (data.Type() == tvtObject) {
            // 处理对象...
        }
    }
};
```

---

## 6. 生命周期管理策略

### 6.1 两种绑定方式的对象生命周期对比

对象生命周期管理是原生绑定中最容易出错的部分。两种方式的生命周期模型有重要差异：

```
=== 低级宏方式 ===

TJS2: new Timer()
  │
  ├─ tTJSNativeClass::CreateNew()
  │    ├─ CreateBaseTJSObject()      → 创建TJS2对象壳
  │    ├─ FuncCall(NULL, ...)        → 初始化实例
  │    │    ├─ CreateNativeInstance() → new tTJS2TimerNI()
  │    │    ├─ NI->Construct(...)    → NI内部创建tTJS2Timer
  │    │    └─ NIS_REGISTER          → 将NI关联到TJS2对象
  │    └─ FuncCall("Timer", ...)     → 调用TJS2层构造函数（可选）
  │
  └─ 对象生命周期由TJS2 GC管理

TJS2: GC回收
  │
  ├─ finalize() → 空（TJS_DECL_EMPTY_FINALIZE_METHOD）
  ├─ NI->Invalidate() → delete timer_（释放C++资源）
  └─ NI->Destruct() → delete NI（释放NI自身）

所有权：TJS2 GC 完全控制
NI负责：创建和销毁C++对象
```

```
=== ncbind方式 ===

TJS2: new Timer()
  │
  ├─ tTJSNativeClassForPlugin::CreateNew()
  │    ├─ CreateBaseTJSObject()
  │    ├─ FuncCall(NULL, ...)
  │    │    ├─ procCreateNativeInstance()
  │    │    │    └─ new ncbInstanceAdaptor<tTJS2Timer>()
  │    │    │         └─ 内部调用 CreateAdaptor()
  │    │    │              └─ new tTJS2Timer(...)
  │    │    └─ NIS_REGISTER
  │    └─ FuncCall(className, ...)
  │
  └─ 对象生命周期由TJS2 GC管理

TJS2: GC回收
  │
  ├─ ncbInstanceAdaptor::Invalidate()
  │    └─ if (!_sticky) delete _instance;
  │       // _sticky=false: 拥有所有权，删除C++对象
  │       // _sticky=true:  不拥有所有权，不删除
  └─ ncbInstanceAdaptor::Destruct()
       └─ delete adaptor

关键区别：_sticky标志控制C++对象所有权
```

### 6.2 粘滞实例（Sticky Instance）详解

`_sticky`标志是ncbind独有的生命周期控制机制。它决定了"谁负责删除C++对象"：

```cpp
// 场景1：非粘滞（默认）——TJS2拥有C++对象
// C++对象由ncbind创建，由ncbind销毁
NCB_REGISTER_CLASS(tTJS2Timer) {
    Constructor<const std::wstring&>(0);
    // tTJS2Timer由CreateAdaptor内部new出来
    // GC回收时，Invalidate中delete
    // _sticky = false（默认）
}

// 场景2：粘滞——C++代码拥有对象，TJS2只是引用
// 用于将已存在的C++对象暴露给TJS2
void exposeCppObject(iTJSDispatch2* global) {
    // 已存在的C++对象
    tTJS2Timer* existingTimer = getGlobalTimer();

    // 创建ncbInstanceAdaptor并设为粘滞
    ncbInstanceAdaptor<tTJS2Timer>* adaptor =
        ncbInstanceAdaptor<tTJS2Timer>::
            CreateAdaptor(existingTimer);
    // CreateAdaptor内部检测参数类型：
    // void → new tTJS2Timer() → _sticky=false
    // tTJS2Timer* → 使用现有指针 → _sticky=true

    // 将adaptor注册到TJS2对象
    // 当TJS2 GC回收时，Invalidate不会delete existingTimer
    // existingTimer的生命周期由C++代码管理
}

// 场景3：工厂模式——C++创建对象但转移所有权给TJS2
class TimerFactory {
    static tTJS2Timer* create(const std::wstring& label) {
        return new tTJS2Timer(label);
    }
};

// 使用NCB_ATTACH_CLASS将已存在对象附加到TJS2
// NCB_ATTACH_CLASS与NCB_REGISTER_CLASS的区别：
// REGISTER: ncbind创建C++对象
// ATTACH: C++代码已经有对象，只需要包装
```

### 6.3 引用计数与循环引用

TJS2使用引用计数进行内存管理，存在循环引用导致内存泄漏的风险：

```cpp
// 危险模式：C++对象持有TJS2对象的引用
class NativeWidget : public tTJSNativeInstance {
    iTJSDispatch2* callback_;  // 持有TJS2回调对象

    tjs_error Construct(...) override {
        if (numparams >= 1) {
            callback_ = param[0]->AsObject();
            // AddRef! 现在callback_持有一个引用
        }
        return TJS_S_OK;
    }

    void Invalidate() override {
        if (callback_) {
            callback_->Release();  // 必须Release!
            callback_ = nullptr;
        }
    }
    // 如果忘记Release → TJS2对象永远不被回收
    // 如果TJS2对象也引用了Widget → 循环引用 → 泄漏
};

// 安全模式：使用弱引用或在适当时机断开
class SafeNativeWidget : public tTJSNativeInstance {
    iTJSDispatch2* callback_;
    bool callbackOwned_;

    void setCallback(iTJSDispatch2* cb) {
        if (callback_ && callbackOwned_) {
            callback_->Release();
        }
        callback_ = cb;
        if (cb) {
            cb->AddRef();
            callbackOwned_ = true;
        }
    }

    void Invalidate() override {
        setCallback(nullptr);  // 清理引用
    }
};
```

---

## 7. 混合使用与高级技巧

### 7.1 在ncbind类中使用TJS2 API

有时需要在ncbind绑定的类中直接调用TJS2 API（如创建Array、调用TJS2函数等）：

```cpp
// C++类中使用TJS2 API的示例
class AdvancedTimer {
public:
    // 返回一个TJS2 Dictionary（无法通过自动转换实现）
    tTJSVariant getStats() {
        // 手动创建Dictionary
        iTJSDispatch2* dict =
            TJSCreateDictionaryObject();

        tTJSVariant val;

        val = tTJSVariant(elapsed_);
        dict->PropSetByVS(
            TJS_MEMBERENSURE,
            TJS_W("elapsed"), &val, dict);

        val = tTJSVariant((tjs_int)lapCount_);
        dict->PropSetByVS(
            TJS_MEMBERENSURE,
            TJS_W("laps"), &val, dict);

        // 创建Array存储每圈时间
        iTJSDispatch2* arr =
            TJSCreateArrayObject();
        for (size_t i = 0; i < laps_.size(); i++) {
            val = tTJSVariant(laps_[i]);
            arr->PropSetByNum(
                TJS_MEMBERENSURE,
                (tjs_int)i, &val, arr);
        }
        val = tTJSVariant(arr, arr);
        dict->PropSetByVS(
            TJS_MEMBERENSURE,
            TJS_W("lapTimes"), &val, dict);
        arr->Release();

        // 包装为tTJSVariant返回
        tTJSVariant result(dict, dict);
        dict->Release();
        return result;
    }

private:
    double elapsed_ = 0;
    int lapCount_ = 0;
    std::vector<double> laps_;
};

// ncbind注册——tTJSVariant返回值直接透传
NCB_REGISTER_CLASS(AdvancedTimer) {
    Constructor();
    NCB_METHOD(getStats, &AdvancedTimer::getStats);
    // ncbind看到返回类型是tTJSVariant，直接透传不转换
}
```

### 7.2 跨绑定方式的交互

在同一个项目中，低级宏绑定的类和ncbind绑定的类可以互相交互：

```cpp
// 在ncbind绑定的类中使用低级宏绑定的Array
class DataCollector {
public:
    // 接受一个TJS2 Array参数
    void collect(iTJSDispatch2* arr) {
        if (!arr) return;

        // 获取Array的count属性
        tTJSVariant countVal;
        arr->PropGet(0, TJS_W("count"),
            nullptr, &countVal, arr);
        tjs_int count = (tjs_int)countVal;

        // 遍历Array元素
        for (tjs_int i = 0; i < count; i++) {
            tTJSVariant elem;
            arr->PropGetByNum(0, i, &elem, arr);
            // 处理每个元素...
            data_.push_back((double)(tjs_real)elem);
        }
    }

    // 返回结果作为TJS2 Array
    tTJSVariant getResults() {
        iTJSDispatch2* arr =
            TJSCreateArrayObject();

        for (size_t i = 0; i < data_.size(); i++) {
            tTJSVariant val(data_[i]);
            arr->PropSetByNum(
                TJS_MEMBERENSURE,
                (tjs_int)i, &val, arr);
        }

        tTJSVariant result(arr, arr);
        arr->Release();
        return result;
    }

private:
    std::vector<double> data_;
};

NCB_REGISTER_CLASS(DataCollector) {
    Constructor();
    // iTJSDispatch2* 参数通过NCB_TYPECONV_BOXING自动转换
    NCB_METHOD(collect, &DataCollector::collect);
    NCB_METHOD(getResults, &DataCollector::getResults);
}
```

### 7.3 调试技巧

绑定代码的调试通常比纯C++或纯TJS2代码更困难，因为错误可能发生在跨边界的位置：

```cpp
// 调试技巧1：添加日志追踪
// 在关键位置添加日志输出
class DebugTimer : public tTJSNativeInstance {
    tjs_error Construct(
        tjs_int numparams,
        tTJSVariant **param,
        iTJSDispatch2 *tjs) override
    {
        TVPAddLog(ttstr(TJS_W(
            "[Timer] Construct called, params="))
            + ttstr((tjs_int)numparams));

        // ... 正常构造逻辑 ...

        TVPAddLog(TJS_W("[Timer] Construct OK"));
        return TJS_S_OK;
    }

    void Invalidate() override {
        TVPAddLog(TJS_W(
            "[Timer] Invalidate called"));
        // ... 正常清理逻辑 ...
    }
};

// 调试技巧2：参数类型检查
TJS_BEGIN_NATIVE_METHOD_DECL(debugMethod)
{
    // 打印所有参数的类型信息
    for (tjs_int i = 0; i < numparams; i++) {
        ttstr typeStr;
        switch (param[i]->Type()) {
            case tvtVoid:    typeStr = TJS_W("void"); break;
            case tvtObject:  typeStr = TJS_W("object"); break;
            case tvtString:  typeStr = TJS_W("string"); break;
            case tvtInteger: typeStr = TJS_W("integer"); break;
            case tvtReal:    typeStr = TJS_W("real"); break;
            case tvtOctet:   typeStr = TJS_W("octet"); break;
        }
        TVPAddLog(ttstr(TJS_W("[param "))
            + ttstr(i) + TJS_W("] type=") + typeStr);
    }
    return TJS_S_OK;
}
TJS_END_NATIVE_METHOD_DECL(debugMethod)

// 调试技巧3：ncbind实例获取失败排查
// 如果GetNativeInstance返回nullptr，检查：
// 1. ClassID是否匹配（类注册名是否正确）
// 2. 对象是否已被Invalidate
// 3. objthis是否为null（静态方法不能用GetNativeInstance）
```

---

## 8. 跨平台注意事项

### 8.1 四平台差异对照

将C++类暴露给TJS2时，跨平台差异主要体现在编译器行为和字符编码两个方面：

| 维度 | Windows | Linux | macOS | Android |
|------|---------|-------|-------|---------|
| **wchar_t大小** | 2字节（UTF-16） | 4字节（UTF-32） | 4字节（UTF-32） | 4字节（UTF-32） |
| **tjs_char** | wchar_t(2B) | wchar_t(4B) | wchar_t(4B) | wchar_t(4B) |
| **编译器** | MSVC | GCC/Clang | Clang | NDK Clang |
| **模板展开** | 较宽松 | 严格 | 严格 | 严格 |
| **插件加载** | LoadLibrary | dlopen | dlopen | System.loadLibrary |
| **符号可见性** | __declspec(dllexport) | __attribute__((visibility)) | __attribute__((visibility)) | __attribute__((visibility)) |
| **异常处理** | SEH + C++ | C++异常 | C++异常 | C++异常 |

```cpp
// 跨平台编码安全模式
// 不要依赖wchar_t的大小，使用tjs_char和ttstr
class CrossPlatformClass {
public:
    // 错误：依赖wchar_t大小
    void setName(const wchar_t* name) {
        // Windows: 2字节字符，直接与TJS2兼容
        // Linux/macOS: 4字节字符，需要转换
        name_ = name;  // 可能出问题
    }

    // 正确：使用TJS2的字符串类型
    void setName(const ttstr& name) {
        // ttstr在所有平台上行为一致
        name_ = name;
    }

    // 正确（替代方案）：使用std::wstring
    // ncbind会自动处理编码转换
    void setNameW(const std::wstring& name) {
        name_ = ttstr(name.c_str());
    }

private:
    ttstr name_;
};
```

### 8.2 插件构建配置

KrKr2中的插件被编译为静态库（`STATIC`），直接链接到主二进制文件。但绑定代码的编写方式仍需要考虑跨平台：

```cmake
# CMakeLists.txt - 跨平台插件构建示例
add_library(my_timer_plugin STATIC
    tjs2TimerPlugin.cpp
)

target_include_directories(my_timer_plugin PRIVATE
    ${CMAKE_SOURCE_DIR}/cpp/core/tjs2    # TJS2头文件
    ${CMAKE_SOURCE_DIR}/cpp/core/plugin  # ncbind头文件
)

# 平台特定编译选项
if(WINDOWS)
    target_compile_definitions(my_timer_plugin PRIVATE
        _UNICODE UNICODE
        WIN32_LEAN_AND_MEAN
    )
elseif(ANDROID)
    target_compile_options(my_timer_plugin PRIVATE
        -fvisibility=hidden    # 隐藏默认符号
        -fno-rtti              # KrKr2不使用RTTI
    )
endif()
```

---

## 9. 常见错误与解决方案

### 9.1 错误排查速查表

| 错误现象 | 可能原因 | 解决方案 |
|----------|---------|---------|
| TJS_E_NATIVECLASSCRASH | 实例未正确关联或已被Invalidate | 检查TJS_GET_NATIVE_INSTANCE是否在正确位置 |
| 编译错误：模板推导失败 | ncbind无法匹配C++方法签名 | 检查方法参数类型是否有ncbTypeConvertor支持 |
| 链接错误：未定义的ClassID | 忘记定义全局ClassID变量 | 添加`tjs_int32 MyClassID = -1;` |
| 运行时：方法调用返回void | 忘记设置`*result` | 确保`if (result) *result = ...;` |
| GC不回收对象 | 循环引用 | 在Invalidate中Release所有持有的TJS2对象 |
| ncbind注册后TJS2找不到类 | V2Link未被调用 | 确保插件DLL入口正确导出V2Link函数 |
| 属性设置无效 | 使用了`TJS_DENY_NATIVE_PROP_SETTER` | 改为TJS_BEGIN_NATIVE_PROP_SETTER |
| `GetNativeInstance`返回nullptr | ClassID不匹配 | 确认NCB_REGISTER_CLASS的类名与查询一致 |
| 字符串乱码 | char*/wchar_t编码不匹配 | 使用ttstr或std::wstring，避免char* |
| 静态方法中访问实例崩溃 | 静态方法无objthis | 静态方法不能使用TJS_GET_NATIVE_INSTANCE |

### 9.2 典型错误代码与修复

```cpp
// 错误1：忘记finalize方法
TJS_BEGIN_NATIVE_MEMBERS(MyClass)
    // 缺少 TJS_DECL_EMPTY_FINALIZE_METHOD
    // 编译不会报错，但GC回收时行为未定义
    TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL(...)
    // ...
TJS_END_NATIVE_MEMBERS

// 修复：加上finalize
TJS_BEGIN_NATIVE_MEMBERS(MyClass)
    TJS_DECL_EMPTY_FINALIZE_METHOD  // ← 加上这行
    TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL(...)
    // ...
TJS_END_NATIVE_MEMBERS

// 错误2：ncbind方法签名不匹配
class Foo {
    // 这个方法返回const引用——ncbind可能无法正确处理
    const std::vector<int>& getData() const {
        return data_;
    }
};

NCB_REGISTER_CLASS(Foo) {
    // 编译错误：ncbTypeConvertor不知道如何转换vector<int>
    NCB_METHOD(getData, &Foo::getData);
}

// 修复：返回tTJSVariant或使用RawCallback
class Foo {
    tTJSVariant getData() const {
        // 手动转换为TJS2 Array
        iTJSDispatch2* arr = TJSCreateArrayObject();
        for (size_t i = 0; i < data_.size(); i++) {
            tTJSVariant val((tjs_int)data_[i]);
            arr->PropSetByNum(
                TJS_MEMBERENSURE, (tjs_int)i,
                &val, arr);
        }
        tTJSVariant result(arr, arr);
        arr->Release();
        return result;
    }
};

// 错误3：在Invalidate后访问已释放的资源
void TJS_INTF_METHOD Invalidate() override {
    delete resource_;
    // 忘记设置为nullptr！
}

// 之后某处代码：
void someMethod() {
    resource_->doSomething();  // 悬垂指针，未定义行为！
}

// 修复：Invalidate后置空
void TJS_INTF_METHOD Invalidate() override {
    delete resource_;
    resource_ = nullptr;  // ← 加上这行
}

void someMethod() {
    if (!resource_) return;   // ← 加上检查
    resource_->doSomething();
}
```

---

## 对照项目源码

本节所有示例和模式均来源于KrKr2项目的实际代码：

| 源文件 | 行号范围 | 说明 |
|--------|---------|------|
| `cpp/core/tjs2/tjsNative.h` | 293-480 | 低级宏定义：TJS_BEGIN_NATIVE_MEMBERS等全套宏 |
| `cpp/core/tjs2/tjsNative.h` | 185-225 | tTJSNativeClass类定义 |
| `cpp/core/tjs2/tjsNative.cpp` | 248-290 | RegisterNCM：成员注册到类对象 |
| `cpp/core/tjs2/tjsNative.cpp` | 305-378 | FuncCall(NULL)：实例初始化和闭包修正 |
| `cpp/core/tjs2/tjsNative.cpp` | 381-439 | CreateNew：完整的对象创建流程 |
| `cpp/core/tjs2/tjsArray.cpp` | 全文 | Array类：低级宏方式的完整范例（30+方法） |
| `cpp/core/tjs2/tjsMath.cpp` | 全文 | Math类：纯静态类的绑定范例 |
| `cpp/core/tjs2/tjsDate.cpp` | 全文 | Date类：带复杂构造函数的范例 |
| `cpp/core/plugin/ncbind.hpp` | 1664-1815 | ncbRegistClass：Method/Property/Constructor注册 |
| `cpp/core/plugin/ncbind.hpp` | 1843-1959 | ncbRegistNativeClass：插件类注册流程 |
| `cpp/core/plugin/ncbind.hpp` | 2157-2280 | 高级NCB_*宏定义 |
| `cpp/core/plugin/ncbind.hpp` | 119-232 | ncbInstanceAdaptor：实例适配器与sticky标志 |
| `cpp/plugins/steam/steam_api.cpp` | 全文（12行） | NCB_REGISTER_CLASS最小示例 |

---

## 本节小结

- **两种绑定方式各有所长**：低级宏方式提供最大灵活性和最佳性能，适合引擎核心类；ncbind方式大幅减少代码量，适合插件开发
- **选择依据**：引擎内置类用低级宏（7个现有类均如此），插件类用ncbind，需要可变参数或特殊控制时用RawCallback
- **生命周期是关键**：`Construct` → `Invalidate` → `Destruct` 三步走，Invalidate中必须释放所有资源并置空指针
- **ncbind的sticky标志**决定了C++对象所有权——默认非粘滞（TJS2拥有），手动附加时粘滞（C++拥有）
- **类型转换**：低级宏方式手动转换tTJSVariant，ncbind通过ncbTypeConvertor模板自动转换
- **循环引用**是内存泄漏的主要来源——NI持有的iTJSDispatch2*必须在Invalidate中Release
- **跨平台**：注意wchar_t大小差异和编译器模板行为差异，优先使用tjs_char和ttstr

---

## 练习题与答案

### 题目1：使用低级宏方式暴露一个Counter类

设计一个简单的Counter类，具有以下TJS2接口：
- 构造函数：可选接受初始值（默认0）
- `increment()` 方法：计数+1
- `decrement()` 方法：计数-1
- `value` 属性：读写，获取/设置当前计数值
- `reset()` 方法：重置为0

请写出完整的NI类和类注册代码（使用TJS_BEGIN_NATIVE_MEMBERS宏）。

<details>
<summary>查看答案</summary>

```cpp
// CounterNI.h - 原生实例类
#include "tjsNative.h"

class tCounterNI : public tTJSNativeInstance {
public:
    tjs_int count_;

    tCounterNI() : count_(0) {}

    tjs_error TJS_INTF_METHOD Construct(
        tjs_int numparams, tTJSVariant **param,
        iTJSDispatch2 *tjs) override
    {
        // 可选初始值参数
        if (numparams >= 1 && param[0]->Type() != tvtVoid) {
            count_ = (tjs_int)*param[0];
        }
        return TJS_S_OK;
    }

    void TJS_INTF_METHOD Invalidate() override {
        // Counter没有需要释放的资源
    }
};

// CounterClass.cpp - 类定义与注册
tjs_int32 tCounterClassID = -1;

class tCounterClass : public tTJSNativeClass {
public:
    tCounterClass() : tTJSNativeClass(TJS_W("Counter"))
    {
        TJS_BEGIN_NATIVE_MEMBERS(Counter)

        TJS_DECL_EMPTY_FINALIZE_METHOD

        TJS_BEGIN_NATIVE_CONSTRUCTOR_DECL(
            _this, tCounterNI, Counter)
        {
            return TJS_S_OK;
        }
        TJS_END_NATIVE_CONSTRUCTOR_DECL(Counter)

        TJS_BEGIN_NATIVE_METHOD_DECL(increment)
        {
            TJS_GET_NATIVE_INSTANCE(_this, tCounterNI);
            _this->count_++;
            if (result) *result = tTJSVariant(
                _this->count_);
            return TJS_S_OK;
        }
        TJS_END_NATIVE_METHOD_DECL(increment)

        TJS_BEGIN_NATIVE_METHOD_DECL(decrement)
        {
            TJS_GET_NATIVE_INSTANCE(_this, tCounterNI);
            _this->count_--;
            if (result) *result = tTJSVariant(
                _this->count_);
            return TJS_S_OK;
        }
        TJS_END_NATIVE_METHOD_DECL(decrement)

        TJS_BEGIN_NATIVE_METHOD_DECL(reset)
        {
            TJS_GET_NATIVE_INSTANCE(_this, tCounterNI);
            _this->count_ = 0;
            if (result) *result = tTJSVariant();
            return TJS_S_OK;
        }
        TJS_END_NATIVE_METHOD_DECL(reset)

        TJS_BEGIN_NATIVE_PROP_DECL(value)
        {
            TJS_BEGIN_NATIVE_PROP_GETTER
                TJS_GET_NATIVE_INSTANCE(
                    _this, tCounterNI);
                *result = tTJSVariant(_this->count_);
                return TJS_S_OK;
            TJS_END_NATIVE_PROP_GETTER

            TJS_BEGIN_NATIVE_PROP_SETTER
                TJS_GET_NATIVE_INSTANCE(
                    _this, tCounterNI);
                _this->count_ = (tjs_int)*param;
                return TJS_S_OK;
            TJS_END_NATIVE_PROP_SETTER
        }
        TJS_END_NATIVE_PROP_DECL(value)

        TJS_END_NATIVE_MEMBERS
    }

    tTJSNativeInstance* CreateNativeInstance() override {
        return new tCounterNI();
    }
};
```

TJS2测试脚本：
```javascript
var c = new Counter(10);      // 初始值10
c.increment();                // 11
c.increment();                // 12
c.decrement();                // 11
System.inform("值: " + c.value);  // 11
c.value = 100;                // 设置为100
c.reset();                    // 重置为0
System.inform("值: " + c.value);  // 0
```

</details>

### 题目2：使用ncbind暴露同一个Counter类

将题目1中的Counter类改用ncbind方式绑定。要求：
- C++类名为`NativeCounter`，TJS2中类名为`Counter`
- 包含所有同样的方法和属性
- 比较两种方式的代码量差异

<details>
<summary>查看答案</summary>

```cpp
// NativeCounter.h - 纯C++类（不依赖TJS2）
class NativeCounter {
    tjs_int count_;
public:
    NativeCounter() : count_(0) {}
    NativeCounter(tjs_int initial) : count_(initial) {}

    void increment() { count_++; }
    void decrement() { count_--; }
    void reset() { count_ = 0; }

    tjs_int getValue() const { return count_; }
    void setValue(tjs_int v) { count_ = v; }
};

// NativeCounterPlugin.cpp - ncbind绑定（完整文件）
#include "ncbind.hpp"
#include "NativeCounter.h"

NCB_REGISTER_CLASS_DIFFER(NativeCounter, Counter)
{
    Constructor<tjs_int>(0);  // 0个必需参数
    NCB_METHOD(increment, &NativeCounter::increment);
    NCB_METHOD(decrement, &NativeCounter::decrement);
    NCB_METHOD(reset, &NativeCounter::reset);
    NCB_PROPERTY(value,
        &NativeCounter::getValue,
        &NativeCounter::setValue);
}
```

**代码量对比**：
| 方式 | NI/Class代码行数 | 方法/属性注册行数 | 总计 |
|------|-----------------|-----------------|------|
| 低级宏 | ~15行（NI类） | ~60行（宏块） | ~75行 |
| ncbind | 0行（无NI类） | ~8行 | ~20行（含C++类） |

ncbind方式的代码量约为低级宏方式的**1/4**。主要节省来自：
1. 无需定义NI类（ncbInstanceAdaptor自动处理）
2. 无需手动提取参数（模板自动推导）
3. 无需手动转换类型（ncbTypeConvertor自动处理）
4. 无需手动设置result（返回值自动转换）

</details>

### 题目3：分析以下代码的内存泄漏问题并修复

```cpp
class EventEmitter : public tTJSNativeInstance {
    iTJSDispatch2* listeners_[16];
    int listenerCount_;
public:
    EventEmitter() : listenerCount_(0) {
        memset(listeners_, 0, sizeof(listeners_));
    }

    void addListener(iTJSDispatch2* cb) {
        if (listenerCount_ < 16) {
            listeners_[listenerCount_++] = cb;
            cb->AddRef();
        }
    }

    void emit() {
        for (int i = 0; i < listenerCount_; i++) {
            tTJSVariant result;
            listeners_[i]->FuncCall(
                0, nullptr, nullptr,
                &result, 0, nullptr,
                listeners_[i]);
        }
    }

    void Invalidate() override {
        // 这里有什么问题？
    }
};
```

<details>
<summary>查看答案</summary>

**问题分析**：

`Invalidate()`为空，没有Release任何listener。每个listener在`addListener`中被AddRef了一次，但从未被Release。这导致：

1. **内存泄漏**：所有listener对象永远不会被GC回收（引用计数永远≥1）
2. **循环引用风险**：如果listener闭包捕获了EventEmitter对象本身，形成循环引用，双方都不会被回收

**修复后的代码**：

```cpp
class EventEmitter : public tTJSNativeInstance {
    iTJSDispatch2* listeners_[16];
    int listenerCount_;
public:
    EventEmitter() : listenerCount_(0) {
        memset(listeners_, 0, sizeof(listeners_));
    }

    void addListener(iTJSDispatch2* cb) {
        if (listenerCount_ < 16 && cb) {
            listeners_[listenerCount_++] = cb;
            cb->AddRef();  // 增加引用计数
        }
    }

    void removeListener(int index) {
        if (index >= 0 && index < listenerCount_) {
            listeners_[index]->Release();  // 释放引用
            // 移动后续元素
            for (int i = index; i < listenerCount_ - 1; i++) {
                listeners_[i] = listeners_[i + 1];
            }
            listenerCount_--;
            listeners_[listenerCount_] = nullptr;
        }
    }

    void emit() {
        for (int i = 0; i < listenerCount_; i++) {
            if (listeners_[i]) {
                tTJSVariant result;
                listeners_[i]->FuncCall(
                    0, nullptr, nullptr,
                    &result, 0, nullptr,
                    listeners_[i]);
            }
        }
    }

    void TJS_INTF_METHOD Invalidate() override {
        // 关键修复：释放所有listener的引用
        for (int i = 0; i < listenerCount_; i++) {
            if (listeners_[i]) {
                listeners_[i]->Release();
                listeners_[i] = nullptr;
            }
        }
        listenerCount_ = 0;
    }
};
```

**要点总结**：
- AddRef和Release必须成对出现
- Invalidate是释放外部引用的最佳时机
- 释放后立即置空指针，防止悬垂引用
- 考虑添加removeListener方法，允许脚本层面控制生命周期

</details>

---

## 下一步

下一章 [08-实战-添加TJS2内置类](../08-实战-添加TJS2内置类/) 将运用本节学到的所有知识，从零开始设计、实现、测试一个完整的TJS2内置类，涵盖需求分析、类设计、成员注册、测试验证的全过程。

