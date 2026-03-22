# TextRender 接口分析与字体处理

> **所属模块：** M12-插件逆向与实现
> **前置知识：** [03-逆向方法论 / 01-DLL导出表与接口识别](../03-逆向方法论/01-DLL导出表与接口识别.md)，[04-实战-逆向PackinOne / 01-PackinOne功能分析与导出表](../04-实战-逆向PackinOne/01-PackinOne功能分析与导出表.md)
> **预计阅读时间：** 35 分钟

## 本节目标

读完本节后，你将能够：
1. 通过 IDA 分析理解 textrender.dll 的导出表和类注册结构
2. 识别 TextRender 插件注册的 TJS 类、方法和属性
3. 理解 KiriKiri 文本渲染插件的字体管理架构
4. 使用 RTTI 信息和方法签名推断文本渲染接口
5. 设计跨平台文本渲染的替代方案

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| RTTI | Run-Time Type Information | C++ 运行时类型信息，编译器在二进制中嵌入的类名和继承关系数据 |
| ncbind | Native Class Bind | KiriKiri 的 C++ 到 TJS 绑定框架，用模板把 C++ 类自动注册为脚本对象 |
| 字形 | Glyph | 一个字符在特定字体、字号下的具体图像表示（包含轮廓、位图等） |
| 抗锯齿 | Anti-aliasing | 通过平滑边缘像素的颜色过渡来消除文字/图形锯齿状边缘的技术 |
| 基线 | Baseline | 文字排版中字符底部对齐的虚拟水平线，字母"g"的尾巴会延伸到基线以下 |
| 描边 | Stroke/Outline | 在文字轮廓外侧绘制一圈边框，常用于让文字在复杂背景上更清晰可读 |

---

## 一、TextRender 插件概述

### 1.1 TextRender 在 KiriKiri 生态中的角色

TextRender 是 KiriKiri/吉里吉里2 引擎的**文本渲染增强插件**。原版 KiriKiri 引擎自带基本的文字显示功能（通过 Windows GDI 的 `DrawText` 实现），但在以下场景下不够用：

1. **精细排版控制**：游戏需要精确控制字间距（Letter Spacing，相邻字符之间的额外间隔）、行间距（Line Spacing，相邻行之间的垂直距离）、文字缩放比例
2. **富文本效果**：描边（文字外圈的轮廓线）、阴影（文字后方的偏移副本）、渐变填充（从一种颜色平滑过渡到另一种颜色）
3. **非 Windows 平台**：GDI 是 Windows 独有的图形接口，Linux/macOS/Android 上不可用
4. **性能优化**：游戏对话场景中每帧可能渲染上百个字符，需要字形缓存（把已经渲染好的字符图像保存在内存中，下次直接复用而非重新渲染）

TextRender 插件通过 TJS 脚本接口暴露了一整套文本渲染 API，让游戏脚本能够精确控制文字的外观和排版。

### 1.2 IDA 分析概览

通过 IDA Pro 7.6 对实际的 textrender.dll（来自"星光咖啡馆与死神之蝶"游戏，222 KB）的分析，我们获得了以下关键信息：

```
文件大小:   221,696 字节
总函数数:   828 个
最大函数:   8,379 字节 (sub_1000DFE0 — 核心渲染函数)
导出函数:   V2Link, V2Unlink, DllEntryPoint
PDB 路径:   c:\dwork\kirikiri\krkrtemplate\plugins\textrender\Release\textrender.pdb
```

PDB 路径（Program Database，微软编译器生成的调试信息文件路径）泄露了开发者的工作目录结构：`c:\dwork\kirikiri\krkrtemplate\plugins\textrender\`。这告诉我们 textrender 是在一个名为 `krkrtemplate` 的 KiriKiri 模板项目中开发的。

### 1.3 导出表分析

textrender.dll 的导出表非常简洁，只有标准的 KiriKiri 插件入口：

```
序号      地址         名称              说明
[1]      0x1001B680   V2Link           插件加载入口
[2]      0x1001B6A0   V2Unlink         插件卸载入口
[3]      0x1001B680   _V2Link@4        stdcall 修饰名（同 V2Link）
[4]      0x1001B6A0   _V2Unlink@0      stdcall 修饰名（同 V2Unlink）
[268552104] 0x1001C7A8 DllEntryPoint   DLL 加载入口（CRT 初始化）
```

注意 `V2Link` 和 `_V2Link@4` 指向同一地址，`V2Unlink` 和 `_V2Unlink@0` 同理。`@4` 和 `@0` 是 stdcall 调用约定的名称修饰——`@N` 表示参数占用 N 字节栈空间。`V2Link` 接受一个 `iTJSDispatch2*` 指针（4 字节），所以是 `@4`；`V2Unlink` 无参数，所以是 `@0`。

---

## 二、V2Link 反汇编与类注册分析

### 2.1 V2Link 的完整反汇编

```asm
; V2Link(iTJSDispatch2* dispatch) — textrender.dll 入口
; 地址: 0x1001B680
;
; 参数: [esp+4] = iTJSDispatch2* — 引擎传入的全局脚本调度器指针
; 返回: eax = 0 (S_OK) 表示成功

0x1001B680: mov  eax, [esp+arg_0]      ; eax = dispatch 指针
0x1001B684: mov  dword_10033950, eax   ; 保存到全局变量（tp_stub 解析用）
0x1001B689: call sub_1001B620          ; 调用类注册函数
0x1001B68E: mov  ecx, dword_1003394C   ; 读取引用计数
0x1001B694: mov  dword_1003393C, ecx   ; 保存初始引用计数
0x1001B69A: xor  eax, eax              ; eax = 0 (S_OK)
0x1001B69C: retn 4                     ; stdcall 返回，清理 4 字节参数
```

对比 PackinOne 的 V2Link：

```
PackinOne V2Link:                    TextRender V2Link:
  mov eax, [esp+arg_0]                mov eax, [esp+arg_0]
  push eax                             mov dword_10033950, eax
  call sub_1000D7A0     ← tp_stub     call sub_1001B620      ← 注册
  add esp, 4                          
  call sub_10001000     ← 注册        mov ecx, dword_1003394C
  mov ecx, dword_1009403C             mov dword_1003393C, ecx
  mov dword_10094000, ecx             xor eax, eax
  xor eax, eax                        retn 4
  retn 4
```

两者的模式完全一致：保存 dispatch 指针 → 调用注册函数 → 保存引用计数 → 返回 S_OK。区别在于 PackinOne 先调用 `tp_stub` 解析函数再注册，而 TextRender 把 tp_stub 解析嵌入到了注册函数 `sub_1001B620` 内部。

### 2.2 V2Unlink 反汇编

```asm
; V2Unlink() — 插件卸载
; 地址: 0x1001B6A0

0x1001B6A0: mov  eax, dword_1003394C    ; 当前引用计数
0x1001B6A5: cmp  eax, dword_1003393C    ; 与初始值比较
0x1001B6AB: jle  short loc_1001B6B3     ; 如果 <= 初始值，可以卸载
0x1001B6AD: mov  eax, 80004005h         ; E_FAIL — 还有引用未释放
0x1001B6B2: retn                        ; 拒绝卸载
0x1001B6B3: call sub_1001B650           ; 执行清理
0x1001B6B8: xor  eax, eax              ; S_OK
0x1001B6BA: retn
```

这里有一个重要的安全检查：如果引用计数（Reference Count，跟踪有多少对象正在使用此插件）高于初始值，说明还有 TJS 对象持有 TextRender 创建的实例，此时拒绝卸载并返回 `E_FAIL`（`0x80004005`，COM 标准错误码，表示通用失败）。这防止了"先卸载插件，再使用插件对象"导致的野指针（Dangling Pointer，指向已释放内存的指针）崩溃。

### 2.3 类注册函数的方法名识别

IDA 分析中的 `sub_10005980` 是 TextRender 的核心注册函数。通过反汇编中引用的字符串常量，我们可以还原注册的所有方法名：

```asm
; sub_10005980 — 部分反汇编，展示方法注册模式
0x100059C3: push offset aSetoption      ; "setOption"
0x100059CA: call sub_100070D0           ; 注册方法

0x100059EC: push offset aSetdefault     ; "setDefault"
0x100059F3: call sub_100070D0           ; 注册方法

0x10005A0D: push offset aSetrendersize  ; "setRenderSize"
0x10005A14: call sub_100070D0           ; 注册方法
```

每次注册调用的模式是：
1. 把方法名字符串地址压栈
2. 把对应的 C++ 函数指针压栈或通过 `call` 获取
3. 调用 `sub_100070D0`（封装了 `TJSNativeClassRegisterNCM` 的注册逻辑）

---

## 三、RTTI 信息与类继承分析

### 3.1 什么是 RTTI 逆向

RTTI（Run-Time Type Information，运行时类型信息）是 MSVC 编译器在 C++ 二进制中嵌入的元数据。当代码使用 `dynamic_cast` 或 `typeid` 时，编译器会为涉及的类生成 RTTI 记录，包含类名（以 MSVC 特有的"修饰名"格式存储）和继承层次。对于逆向工程师来说，RTTI 是极其宝贵的信息源——它直接告诉你"这个二进制里有哪些 C++ 类、它们之间的继承关系是什么"。

在 IDA 的字符串列表中，RTTI 类名以 `.?AV` 开头（AV = class，AU = struct），后跟修饰过的类名和命名空间，最后以 `@@` 结尾。例如：

```
.?AVTextRender@@          → class TextRender
.?AVTextRenderBase@@      → class TextRenderBase
.?AViTJSDispatch2@@       → class iTJSDispatch2（i 前缀表示接口）
.?AVtTJSNativeInstance@@  → class tTJSNativeInstance
```

### 3.2 TextRender 的类继承树

从 IDA 提取的 RTTI 信息中，我们发现了以下类层次结构：

```
RTTI 地址        修饰名                          还原类名
0x10031FC8      .?AVTextRender@@                TextRender
0x10031FE4      .?AVTextRenderBase@@            TextRenderBase
0x10032054      .?AViTJSDispatch2@@             iTJSDispatch2
0x10032070      .?AVtTJSDispatch@@              tTJSDispatch
0x100320F4      .?AVtTJSNativeInstance@@         tTJSNativeInstance
0x10032118      .?AViTJSNativeInstance@@         iTJSNativeInstance
```

结合 ncbind 框架的适配器类 RTTI：

```
0x10031F18      ncbNativeClassFactory<TextRenderBase>
0x10031F50      ncbInstanceAdaptor<TextRenderBase>
0x10031F88      ncbNativeClassAutoRegister<TextRenderBase>
```

由此可以还原完整的类继承关系：

```
┌──────────────────────────────────────────────────────────┐
│                    iTJSNativeInstance                      │
│              （TJS 原生实例接口 — 纯虚基类）              │
└──────────────────┬───────────────────────────────────────┘
                   │ 继承
┌──────────────────▼───────────────────────────────────────┐
│                  tTJSNativeInstance                        │
│      （TJS 原生实例默认实现 — 提供引用计数等基础设施）    │
└──────────────────┬───────────────────────────────────────┘
                   │ 继承
┌──────────────────▼───────────────────────────────────────┐
│                   TextRenderBase                          │
│         （文本渲染基类 — 定义接口和通用属性）             │
└──────────────────┬───────────────────────────────────────┘
                   │ 继承
┌──────────────────▼───────────────────────────────────────┐
│                    TextRender                              │
│     （文本渲染实现类 — 包含具体的渲染算法和字体管理）     │
└──────────────────────────────────────────────────────────┘
```

### 3.3 ncbind 适配器模式解析

注意 RTTI 中出现的三个 ncbind 模板实例都以 `TextRenderBase` 为参数（而非 `TextRender`）。这说明开发者在注册 TJS 类时，选择了 `TextRenderBase` 作为注册目标：

```cpp
// 推断的注册代码（根据 RTTI 还原）
NCB_REGISTER_CLASS(TextRenderBase) {
    // ncbind 框架会自动实例化以下模板：
    // - ncbNativeClassFactory<TextRenderBase>     → 创建/销毁实例
    // - ncbInstanceAdaptor<TextRenderBase>         → TJS 对象与 C++ 对象的桥接
    // - ncbNativeClassAutoRegister<TextRenderBase> → V2Link 时自动注册
    
    // 属性和方法注册（下一节详细分析）...
};
```

`ncbInstanceAdaptor<TextRenderBase>` 是关键适配器——它继承自 `tTJSNativeInstance`，内部持有一个 `TextRenderBase*` 指针，负责在 TJS 对象被垃圾回收时调用 C++ 析构函数。这就是 ncbind 框架实现"C++ 对象生命周期跟随 TJS 对象"的机制。

---

## 四、TextRender 属性与方法完整清单

### 4.1 从 RTTI 还原属性列表

IDA 中的 RTTI 字符串不仅包含类名，还包含 ncbind 为每个注册的属性和方法生成的模板实例。这些模板实例的类型签名直接反映了 C++ 成员函数的原型。

**只读属性**（只有 getter，无 setter）：

| RTTI 修饰名片段 | C++ 签名 | 推断属性类型 | 可能的属性名 |
|-----------------|----------|-------------|-------------|
| `PropertyCommand<TextRenderBase, const wchar_t*(TextRender::*)()>` | `const wchar_t* TextRender::getXxx() const` | 宽字符串（Wide String） | `face`（字体名称） |
| `PropertyCommand<TextRenderBase, float(TextRender::*)()>` | `float TextRender::getXxx() const` | 浮点数 | `size` / `spacing` |
| `PropertyCommand<TextRenderBase, int(TextRender::*)()>` | `int TextRender::getXxx() const` | 整数 | `color` / `style` |
| `PropertyCommand<TextRenderBase, bool(TextRender::*)()>` | `bool TextRender::getXxx() const` | 布尔值 | `bold` / `italic` |

**读写属性**（同时有 getter 和 setter）：

| RTTI 修饰名片段 | getter 签名 | setter 签名 | 推断类型 |
|-----------------|------------|------------|---------|
| `PropertyCommand<..., P82@AEXM@Z>` | `float get()` | `void set(float)` | 浮点读写属性 |
| `PropertyCommand<..., P82@AEX_N@Z>` | `bool get()` | `void set(bool)` | 布尔读写属性 |
| `PropertyCommand<..., P81@AEXI@Z>` | `unsigned int get()` | `void set(unsigned int)` | 无符号整数读写属性 |
| `PropertyCommand<..., P81@AEXH@Z>` | `int get()` | `void set(int)` | 有符号整数读写属性 |
| `PropertyCommand<..., P81@AEXM@Z>` | `float get()` | `void set(float)` | 浮点读写属性（定义在 Base 上） |
| `PropertyCommand<..., P82@AEXPB_W@Z>` | `const wchar_t* get()` | `void set(const wchar_t*)` | 字符串读写属性 |

> **MSVC 修饰名速读**：`P8TextRender@@BEMXZ` 分解为：`P8` = 成员指针，`TextRender` = 类名，`BE` = const 成员函数，`M` = float 返回值，`XZ` = 无参数。`AE` 表示非 const（即 setter），后面的字母是参数类型：`M` = float，`H` = int，`I` = unsigned int，`_N` = bool，`PB_W` = const wchar_t*。

### 4.2 从 RTTI 还原方法列表

同样地，ncbind 为每个注册方法生成 `InvokeCommand` 模板实例：

| RTTI 修饰名片段 | 还原的 C++ 签名 | 推断功能 |
|-----------------|----------------|---------|
| `InvokeCommand<TextRenderBase, int(TextRenderBase::*)(float,float)>` | `int method(float x, float y)` | 坐标相关操作（命中测试/位置计算） |
| `InvokeCommand<TextRenderBase, bool(TextRender::*)(int,float,float)>` | `bool method(int ch, float x, float y)` | 绘制单个字符到指定坐标 |
| `InvokeCommand<TextRenderBase, bool(TextRender::*)(const wchar_t*,int,int,int,bool)>` | `bool method(const wchar_t* text, int x, int y, int maxWidth, bool wrap)` | 绘制文本（带换行控制） |
| `InvokeCommand<TextRenderBase, void(TextRender::*)()>` | `void method()` | 无参操作（`clear` / `reset`） |
| `InvokeCommand<TextRenderBase, void(TextRender::*)(float,float)>` | `void method(float w, float h)` | 设置渲染尺寸 |
| `InvokeCommand<TextRenderBase, tTJSString(TextRenderBase::*)(const wchar_t*)>` | `tTJSString method(const wchar_t* input)` | 字符串处理/转换 |
| `InvokeCommand<TextRenderBase, tTJSVariant(TextRenderBase::*)(int)>` | `tTJSVariant method(int index)` | 按索引获取信息 |
| `InvokeCommand<TextRenderBase, tTJSVariant(TextRenderBase::*)()>` | `tTJSVariant method()` | 获取状态信息 |
| `InvokeCommand<TextRenderBase, tTJSVariant(TextRenderBase::*)(int,int)>` | `tTJSVariant method(int a, int b)` | 范围查询 |
| `InvokeCommand<TextRenderBase, void(TextRenderBase::*)(tTJSVariant)>` | `void method(tTJSVariant opt)` | 设置选项/配置 |
| `InvokeCommand<TextRenderBase, bool(TextRender::*)(float,float)>` | `bool method(float x, float y)` | 坐标测试（返回是否在范围内） |
| `InvokeCommand<TextRenderBase, int(TextRender::*)(int)>` | `int method(int param)` | 按整数参数查询 |
| `InvokeCommand<TextRenderBase, float(TextRender::*)(int)>` | `float method(int param)` | 获取浮点度量值 |

结合 `sub_10005980` 反汇编中识别到的字符串常量 `"setOption"`、`"setDefault"`、`"setRenderSize"`，我们可以将部分方法名与签名对应起来：

```cpp
// 推断的 TextRender TJS 接口
class TextRenderBase : public tTJSNativeInstance {
public:
    // —— 属性（Property） ——
    // 字体名称（如 "MS Gothic"、"SimSun"）
    const wchar_t* get_face() const;
    void set_face(const wchar_t* fontName);
    
    // 字体大小（单位：像素）
    float get_size() const;
    void set_size(float pixels);
    
    // 粗体/斜体开关
    bool get_bold() const;
    void set_bold(bool enabled);
    
    // 文字颜色（ARGB 整数）
    int get_color() const;
    void set_color(int argb);
    
    // —— 方法（Method） ——
    void setOption(tTJSVariant option);     // 设置渲染选项
    void setDefault();                       // 恢复默认设置
    void setRenderSize(float w, float h);   // 设置渲染画布尺寸
    
    // 文本绘制
    bool drawText(const wchar_t* text,      // 绘制文本到指定位置
                  int x, int y,
                  int maxWidth, bool wrap);
    
    // 字符级操作
    bool drawChar(int charCode,             // 绘制单个字符
                  float x, float y);
    
    // 度量查询
    tTJSVariant getMetrics();               // 获取当前字体度量信息
    tTJSString processText(const wchar_t*); // 文本预处理
};
```

---

## 五、字体管理与渲染架构

### 5.1 textrender.dll 不依赖 GDI 的重大发现

IDA 导入表分析揭示了一个关键事实：**textrender.dll 只导入 KERNEL32.dll，完全不导入 GDI32.dll**。这意味着它**不使用 Windows GDI 进行文字渲染**。

```
textrender.dll 导入表:
  仅 KERNEL32.dll — HeapAlloc, HeapFree, VirtualAlloc, CriticalSection 系列...
  无 GDI32.dll（无 CreateFont, GetGlyphOutline, TextOut 等）
  无 USER32.dll

对比 PackinOne.dll 导入表:
  KERNEL32, USER32, GDI32（含 AddFontMemResourceEx, AddFontResourceExW）...
```

这是一个极其重要的线索。不使用 GDI 意味着 textrender.dll 很可能内嵌了自己的字体光栅化引擎（Font Rasterizer，将矢量字体轮廓转换为像素位图的算法）。常见的可能性包括：

1. **FreeType**：开源字体引擎，广泛用于 Linux/Android/游戏引擎。但 IDA 字符串搜索中未发现 "FreeType"、"freetype"、"ft_" 等特征字符串
2. **自研光栅化器**：开发者可能实现了一个轻量级的 TrueType/OpenType 解析器。最大函数 `sub_1000DFE0`（8,379 字节）的规模支持这个推测——一个完整的字形渲染函数确实可以达到这个大小
3. **stb_truetype**：Sean Barrett 的单头文件 TrueType 光栅化库，嵌入后不会留下明显的库标识字符串

考虑到 textrender.dll 的大小（222 KB）和函数数量（828 个），自研或嵌入 stb_truetype 的可能性最高。

### 5.2 核心渲染函数分析

最大函数 `sub_1000DFE0`（8,379 字节，约 1,900 条指令）很可能是文本渲染的核心函数。从反汇编中可以观察到：

```asm
; sub_1000DFE0 入口 — 典型的 SEH（结构化异常处理）prologue
0x1000DFE0: push    ebp
0x1000DFE1: mov     ebp, esp
0x1000DFE3: and     esp, 0FFFFFFF8h       ; 栈 8 字节对齐（用于浮点运算）
0x1000DFE6: push    0FFFFFFFFh
0x1000DFE8: push    offset SEH_1000DFE0   ; SEH 异常处理器
0x1000DFED: mov     eax, large fs:0
0x1000DFF3: push    eax
0x1000DFF4: sub     esp, 1A8h             ; 分配 424 字节局部变量空间
```

关键特征：
- **栈 8 字节对齐**（`and esp, 0FFFFFFF8h`）——这表明函数大量使用浮点运算（FPU 指令要求对齐的操作数）
- **424 字节局部变量**——大量的临时数据，可能包含字形位图缓冲区、坐标变换矩阵
- **SEH 保护**——渲染过程中可能出现异常（如字体数据损坏），需要安全处理
- **虚函数调用**（`mov eax, [edx+0Ch]; call eax`）——通过虚表调用，说明使用了多态

该函数被 `sub_100121A0`（4,402 字节）调用了 3 次，后者可能是"逐行渲染"的循环函数。

### 5.3 字形缓存推测

对象偏移分析（从反汇编中的 `[esi+offset]` 模式推断成员变量布局）：

```
偏移量          推测用途              依据
[esi+0]        虚表指针 (vptr)      标准 C++ 对象布局
[esi+0Ch]      虚函数槽 #3          call [edx+0Ch]
[esi+114h]     渲染状态标志          sub_100121A0 中频繁读取
[esi+118h]     字符串/缓冲区指针     lea esi, [ebp+118h] 后传给字符串处理函数
[esi+1C4h]     容器数据指针          与 [esi+1D0h] 配对，像 std::vector
[esi+1D0h]     容器当前大小          cmp 和 bounds check
[esi+1D4h]     容器容量              与 1D0h 比较做边界检查
```

`[esi+1C4h]` / `[esi+1D0h]` / `[esi+1D4h]` 这组偏移量的访问模式（指针 + size + capacity，配合 `_invalid_parameter_noinfo` 边界检查）几乎可以确定是 `std::vector` 的内部结构。这很可能是**字形缓存**（Glyph Cache）——一个存储已渲染字形位图的动态数组。

### 5.4 跨平台字体方案对比

既然 textrender.dll 不依赖 GDI，在 KrKr2 跨平台模拟器中重新实现时，我们有以下选择：

| 方案 | 优势 | 劣势 | 适用场景 |
|------|------|------|---------|
| **FreeType** | 业界标准、功能完整、跨平台 | 库较大（~500KB），需要额外的 harfbuzz 做复杂排版 | 推荐：需要精确还原原版渲染效果 |
| **stb_truetype** | 单文件、零依赖、易集成 | 功能有限、不支持 OpenType 高级特性 | 适合轻量需求 |
| **平台原生 API** | 利用系统字体渲染器、无需额外库 | 跨平台行为不一致、难以精确控制 | 不推荐：与原版差异大 |
| **ICU + HarfBuzz + FreeType** | 完整的国际化文本渲染栈 | 依赖链复杂、构建成本高 | 需要 CJK 竖排、阿拉伯语等复杂排版 |

对于 KrKr2 模拟器项目，推荐使用 **FreeType** 方案，原因：
1. 项目已通过 vcpkg 管理依赖，添加 FreeType 成本低
2. 需要精确控制字形度量（metrics）以匹配原版 TextRender 的像素级渲染
3. KiriKiri 游戏大量使用日文/中文字体，FreeType 的 CJK 支持成熟

---

## 动手实践

### 实践 1：根据 RTTI 还原 TextRender 头文件

根据前面的分析，创建一个与原版接口兼容的 C++ 头文件：

```cpp
// textrender.h — 根据 IDA 逆向结果还原的 TextRender 接口
// 来源: textrender.dll RTTI 分析 + V2Link 反汇编
#pragma once

#include <string>
#include <vector>
#include <cstdint>

// 前向声明 TJS 类型（实际来自 tp_stub.h）
class iTJSDispatch2;
class tTJSVariant;
class tTJSString;
using tjs_char = wchar_t;

// TextRenderBase — 文本渲染基类
// RTTI: .?AVTextRenderBase@@
// 注册方式: ncbNativeClassAutoRegister<TextRenderBase>
class TextRenderBase {
public:
    virtual ~TextRenderBase() = default;

    // —— 只读属性 ——
    virtual const tjs_char* getFace() const = 0;  // 字体名称
    virtual float getSize() const = 0;             // 字体大小（像素）
    virtual int getColor() const = 0;              // 文字颜色（ARGB）
    virtual bool getBold() const = 0;              // 是否粗体

    // —— 读写属性 ——
    virtual void setFace(const tjs_char* name) = 0;
    virtual void setSize(float pixels) = 0;
    virtual void setColor(int argb) = 0;
    virtual void setBold(bool enabled) = 0;

    // —— 方法 ——
    virtual void setOption(const tTJSVariant& opt) = 0;
    virtual void setDefault() = 0;
    virtual void setRenderSize(float w, float h) = 0;

    // 文本度量
    virtual int measureWidth(float maxWidth, float lineHeight) = 0;
    virtual tTJSVariant getMetrics() = 0;
    virtual tTJSString processText(const tjs_char* input) = 0;
};

// TextRender — 实际实现类
// RTTI: .?AVTextRender@@
class TextRender : public TextRenderBase {
public:
    TextRender();
    ~TextRender() override;

    // 实现所有虚函数...
    const tjs_char* getFace() const override;
    float getSize() const override;
    int getColor() const override;
    bool getBold() const override;
    void setFace(const tjs_char* name) override;
    void setSize(float pixels) override;
    void setColor(int argb) override;
    void setBold(bool enabled) override;
    void setOption(const tTJSVariant& opt) override;
    void setDefault() override;
    void setRenderSize(float w, float h) override;
    int measureWidth(float maxWidth, float lineHeight) override;
    tTJSVariant getMetrics() override;
    tTJSString processText(const tjs_char* input) override;

    // TextRender 特有方法（从 RTTI InvokeCommand 推断）
    bool drawText(const tjs_char* text, int x, int y,
                  int maxWidth, bool wrap);
    bool drawChar(int charCode, float x, float y);
    bool hitTest(float x, float y);

private:
    std::wstring face_;                   // 当前字体名称
    float size_ = 24.0f;                  // 字体大小
    int color_ = 0xFFFFFFFF;              // 白色（ARGB）
    bool bold_ = false;                   // 粗体标志
    float renderWidth_ = 0;              // 渲染画布宽度
    float renderHeight_ = 0;             // 渲染画布高度
    std::vector<uint8_t> glyphCache_;    // 字形缓存（对应 IDA 中的 vector 偏移）
};
```

### 实践 2：MSVC 修饰名解码器

编写一个小工具来解码 IDA 中看到的 MSVC 修饰名：

```cpp
// undecorate_name.cpp — MSVC 修饰名解码工具
// 编译: cl /EHsc undecorate_name.cpp /link dbghelp.lib
#include <cstdio>
#include <cstring>
#include <Windows.h>
#include <DbgHelp.h>  // UnDecorateSymbolName

int main(int argc, char* argv[]) {
    if (argc < 2) {
        printf("用法: undecorate_name <修饰名>\n");
        printf("示例: undecorate_name "
               "\"?AVTextRender@@\"\n");
        return 1;
    }

    const char* decorated = argv[1];
    char undecorated[1024] = {0};

    // UNDNAME_COMPLETE = 0x0000 — 完整解码
    // UNDNAME_NO_ARGUMENTS = 0x2000 — 省略函数参数
    DWORD flags = 0x0000;

    DWORD result = UnDecorateSymbolName(
        decorated,       // 输入：MSVC 修饰名
        undecorated,     // 输出缓冲区
        sizeof(undecorated),
        flags            // 解码选项
    );

    if (result > 0) {
        printf("修饰名: %s\n", decorated);
        printf("还原名: %s\n", undecorated);
    } else {
        printf("解码失败 (错误码: %lu)\n",
               GetLastError());
    }
    return 0;
}
```

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(undecorate_name CXX)
set(CMAKE_CXX_STANDARD 17)

add_executable(undecorate_name undecorate_name.cpp)
target_link_libraries(undecorate_name PRIVATE dbghelp)
```

运行示例：
```bash
> undecorate_name ".?AVTextRender@@"
修饰名: .?AVTextRender@@
还原名: class TextRender

> undecorate_name ".?AU?$ncbInstanceAdaptor@VTextRenderBase@@@@"
修饰名: .?AU?$ncbInstanceAdaptor@VTextRenderBase@@@@
还原名: struct ncbInstanceAdaptor<class TextRenderBase>
```

### 实践 3：模拟 TextRender 的 ncbind 注册

```cpp
// mock_textrender_register.cpp
// 演示如何用 ncbind 注册一个与 TextRender 接口兼容的类
// 编译: 需要在 KrKr2 项目环境中，包含 ncbind.hpp
#include <cstdio>
#include <string>

// 模拟 TJS 类型（简化版）
using tjs_char = wchar_t;

class MockTextRenderBase {
public:
    virtual ~MockTextRenderBase() = default;

    // 只读属性（getter only）
    const tjs_char* getFace() const {
        return face_.c_str();
    }
    float getSize() const { return size_; }
    bool getBold() const { return bold_; }
    int getColor() const { return color_; }

    // 读写属性
    void setFace(const tjs_char* name) {
        face_ = name;
        printf("字体已设置为: %ls\n", name);
    }
    void setSize(float s) {
        size_ = s;
        printf("字体大小已设置为: %.1f px\n", s);
    }
    void setBold(bool b) {
        bold_ = b;
        printf("粗体: %s\n", b ? "开" : "关");
    }
    void setColor(int c) {
        color_ = c;
        printf("颜色已设置为: 0x%08X\n", c);
    }

    // 方法
    void setDefault() {
        face_ = L"MS Gothic";
        size_ = 24.0f;
        bold_ = false;
        color_ = 0xFFFFFFFF;
        printf("已恢复默认设置\n");
    }

    void setRenderSize(float w, float h) {
        printf("渲染尺寸: %.0f x %.0f\n", w, h);
    }

private:
    std::wstring face_ = L"MS Gothic";
    float size_ = 24.0f;
    bool bold_ = false;
    int color_ = 0xFFFFFFFF;
};

int main() {
    MockTextRenderBase tr;

    // 模拟 TJS 脚本调用序列
    printf("=== 模拟 TextRender 操作 ===\n\n");

    tr.setDefault();
    printf("当前字体: %ls, 大小: %.1f\n\n",
           tr.getFace(), tr.getSize());

    tr.setFace(L"SimSun");
    tr.setSize(28.0f);
    tr.setBold(true);
    tr.setColor(0xFF00FF00);  // 绿色
    tr.setRenderSize(800.0f, 600.0f);

    printf("\n最终状态:\n");
    printf("  字体: %ls\n", tr.getFace());
    printf("  大小: %.1f px\n", tr.getSize());
    printf("  粗体: %s\n", tr.getBold() ? "是" : "否");
    printf("  颜色: 0x%08X\n", tr.getColor());
    return 0;
}
```

```
=== 模拟 TextRender 操作 ===

已恢复默认设置
当前字体: MS Gothic, 大小: 24.0

字体已设置为: SimSun
字体大小已设置为: 28.0 px
粗体: 开
颜色已设置为: 0xFF00FF00
渲染尺寸: 800 x 600

最终状态:
  字体: SimSun
  大小: 28.0 px
  粗体: 是
  颜色: 0xFF00FF00
```

### 实践 4：用 IDA 脚本提取更多注册信息

使用 `stuffs/ida_scripts/` 中的工具脚本进一步分析 TextRender 的注册函数：

```bash
# 反汇编注册函数（sub_10005980，3310 字节）
idat64.exe -A -S"disasm_func.py 0x10005980 tr_register.txt" ^
  "stuffs/星光咖啡馆与死神之蝶kr插件/textrender.i64"

# 查找注册函数的所有交叉引用
idat64.exe -A -S"get_xrefs.py 0x10005980 tr_xrefs.txt" ^
  "stuffs/星光咖啡馆与死神之蝶kr插件/textrender.i64"

# 搜索所有包含 "Text" 的函数名
idat64.exe -A -S"search_funcs.py Text tr_text_funcs.txt" ^
  "stuffs/星光咖啡馆与死神之蝶kr插件/textrender.i64"

# 读取注册函数附近的数据区（可能有方法名字符串表）
idat64.exe -A -S"read_data.py 0x10005980 64 hex" ^
  "stuffs/星光咖啡馆与死神之蝶kr插件/textrender.i64"
```

### 实践 5：跨平台字体枚举

```cpp
// font_enumerate.cpp — 跨平台字体枚举示例
// 演示在不同平台上如何获取可用字体列表
#include <cstdio>
#include <vector>
#include <string>

#ifdef _WIN32
  #include <Windows.h>
  // Windows: 使用 EnumFontFamilies
  int CALLBACK FontEnumProc(
      const LOGFONTW* lf,     // 字体逻辑信息
      const TEXTMETRICW* tm,  // 字体度量
      DWORD fontType,         // 字体类型
      LPARAM lParam) {        // 用户数据
    auto* fonts = reinterpret_cast<
        std::vector<std::wstring>*>(lParam);
    fonts->push_back(lf->lfFaceName);
    return 1;  // 继续枚举
  }

  std::vector<std::wstring> enumerateFonts() {
      std::vector<std::wstring> fonts;
      HDC hdc = GetDC(nullptr);  // 屏幕 DC
      LOGFONTW lf = {};
      lf.lfCharSet = DEFAULT_CHARSET;
      EnumFontFamiliesExW(
          hdc, &lf, FontEnumProc,
          reinterpret_cast<LPARAM>(&fonts), 0);
      ReleaseDC(nullptr, hdc);
      return fonts;
  }

#elif defined(__linux__)
  #include <fontconfig/fontconfig.h>
  // Linux: 使用 fontconfig
  std::vector<std::wstring> enumerateFonts() {
      std::vector<std::wstring> fonts;
      FcInit();
      FcPattern* pat = FcPatternCreate();
      FcObjectSet* os = FcObjectSetBuild(
          FC_FAMILY, nullptr);
      FcFontSet* fs = FcFontList(nullptr, pat, os);
      if (fs) {
          for (int i = 0; i < fs->nfont; i++) {
              FcChar8* family = nullptr;
              FcPatternGetString(
                  fs->fonts[i], FC_FAMILY,
                  0, &family);
              if (family) {
                  // 简化: UTF-8 到 wstring 转换
                  std::string s(
                      reinterpret_cast<char*>(family));
                  fonts.push_back(
                      std::wstring(s.begin(), s.end()));
              }
          }
          FcFontSetDestroy(fs);
      }
      FcObjectSetDestroy(os);
      FcPatternDestroy(pat);
      FcFini();
      return fonts;
  }

#elif defined(__APPLE__)
  #include <CoreText/CoreText.h>
  // macOS: 使用 CoreText
  std::vector<std::wstring> enumerateFonts() {
      std::vector<std::wstring> fonts;
      CFArrayRef descriptors =
          CTFontManagerCopyAvailableFontFamilyNames();
      if (descriptors) {
          CFIndex count = CFArrayGetCount(descriptors);
          for (CFIndex i = 0; i < count; i++) {
              auto name = static_cast<CFStringRef>(
                  CFArrayGetValueAtIndex(descriptors, i));
              // CFString 转 wstring（简化）
              char buf[256];
              CFStringGetCString(
                  name, buf, sizeof(buf),
                  kCFStringEncodingUTF8);
              std::string s(buf);
              fonts.push_back(
                  std::wstring(s.begin(), s.end()));
          }
          CFRelease(descriptors);
      }
      return fonts;
  }

#elif defined(__ANDROID__)
  // Android: 扫描 /system/fonts/ 目录
  #include <dirent.h>
  std::vector<std::wstring> enumerateFonts() {
      std::vector<std::wstring> fonts;
      DIR* dir = opendir("/system/fonts");
      if (dir) {
          struct dirent* entry;
          while ((entry = readdir(dir)) != nullptr) {
              std::string name(entry->d_name);
              // 过滤 .ttf 和 .otf 文件
              if (name.size() > 4) {
                  auto ext = name.substr(
                      name.size() - 4);
                  if (ext == ".ttf" || ext == ".otf") {
                      // 去掉扩展名作为字体名
                      auto base = name.substr(
                          0, name.size() - 4);
                      fonts.push_back(std::wstring(
                          base.begin(), base.end()));
                  }
              }
          }
          closedir(dir);
      }
      return fonts;
  }
#endif

int main() {
    printf("=== 系统可用字体列表 ===\n\n");
    auto fonts = enumerateFonts();
    printf("共发现 %zu 个字体:\n", fonts.size());
    for (size_t i = 0; i < fonts.size() && i < 20; i++) {
        printf("  [%2zu] %ls\n", i + 1, fonts[i].c_str());
    }
    if (fonts.size() > 20) {
        printf("  ... (省略 %zu 个)\n",
               fonts.size() - 20);
    }
    return 0;
}
```

---

## 对照项目源码

TextRender 插件在 KrKr2 项目中尚未实现（属于无源码插件），但以下项目文件为实现提供了基础设施：

相关文件：
- `cpp/core/plugin/ncbind.hpp` 第 1-2382 行 — ncbind 框架完整实现，`NCB_REGISTER_CLASS` 宏、`ncbInstanceAdaptor` 模板、`PropertyCommand` / `InvokeCommand` 模板都定义在此文件中。TextRender 的 RTTI 中出现的每个模板实例都能在这里找到定义
- `cpp/core/plugin/ncbind.cpp` 第 1-29 行 — ncbind 模块加载入口，`LoadModule()` 函数触发所有 `ncbNativeClassAutoRegister` 的自动注册
- `cpp/plugins/CMakeLists.txt` 第 1-74 行 — 插件构建配置，新增 TextRender 实现需要在此文件添加源文件
- `doc/krkr2_plugins.md` 第 1-54 行 — 插件清单，textrender 标记为 `nan`（无源码），实现后需更新状态

IDA 分析数据文件：
- `stuffs/ida_scripts/textrender_analysis.txt` — 2483 行完整分析报告（导出表、导入表、函数列表、RTTI、反汇编）
- `stuffs/ida_scripts/extract_textrender.py` — 生成上述报告的 IDAPython 脚本

---

## 常见错误与排查

### 错误 1：MSVC 修饰名解码不完整

**症状**：使用 `undname` 或 `UnDecorateSymbolName` 解码 RTTI 字符串时，输出为空或乱码。

**原因**：RTTI 的 `.?AV` 前缀不是标准的函数修饰名格式。MSVC 的 RTTI TypeDescriptor 使用的名称格式与函数修饰名略有不同。

**解决方案**：
```bash
# RTTI 类名需要去掉 .?AV 前缀和 @@ 后缀手动解读
# .?AVTextRender@@  →  class TextRender
# .?AUsome_struct@@ →  struct some_struct
# .?AV?$template@VArg@@@@ → class template<class Arg>

# 对于复杂模板，可以用 undname 工具（MSVC 自带）：
undname "?method@TextRender@@QAEHXZ"
# 输出: public: int __thiscall TextRender::method(void)
```

### 错误 2：混淆 TextRenderBase 和 TextRender 的注册目标

**症状**：实现 ncbind 注册时，在 `NCB_REGISTER_CLASS(TextRender)` 中注册属性，但 RTTI 显示模板参数是 `TextRenderBase`。

**原因**：原版 textrender.dll 使用 `TextRenderBase` 作为注册目标类，而非 `TextRender`。ncbind 的 `PropertyCommand` 和 `InvokeCommand` 模板中的第一个参数（类名）必须与注册目标一致。

**解决方案**：
```cpp
// 正确：以 TextRenderBase 为注册目标
NCB_REGISTER_CLASS(TextRenderBase) {
    NCB_PROPERTY_RW(face, getFace, setFace);
    NCB_PROPERTY_RW(size, getSize, setSize);
    NCB_METHOD(setDefault);
    NCB_METHOD(setRenderSize);
    // ...
}

// 错误：以 TextRender 为注册目标
// NCB_REGISTER_CLASS(TextRender) { ... }
// 这会导致 RTTI 不匹配，某些边界情况下的类型转换可能失败
```

### 错误 3：忽略 textrender.dll 不导入 GDI32 的事实

**症状**：在跨平台实现中使用 Windows GDI API（`CreateFont`、`GetGlyphOutline`）渲染文字，认为"原版就是这么做的"。

**原因**：实际 IDA 分析证明 textrender.dll **不依赖 GDI32.dll**，它使用内嵌的软件光栅化器。如果你的实现依赖 GDI，在渲染效果上会与原版有差异（GDI 的抗锯齿算法、字形 hints 处理方式与自研渲染器不同）。

**解决方案**：使用 FreeType 或其他可控的软件字体渲染库，这样能在所有平台上获得一致的渲染结果，并更容易调参匹配原版效果。

---

## 本节小结

- textrender.dll 是一个 222 KB 的 KiriKiri 文本渲染增强插件，包含 828 个函数
- V2Link/V2Unlink 遵循标准的 KiriKiri 插件入口模式：保存 dispatch → 注册类 → 保存引用计数 → 返回 S_OK
- V2Unlink 包含引用计数安全检查，防止在有活跃对象时卸载插件
- RTTI 揭示了完整的类层次：TextRender → TextRenderBase → tTJSNativeInstance → iTJSNativeInstance
- ncbind 模板实例的类型签名可以精确还原属性类型（string/float/int/bool/unsigned int）和方法签名（参数类型和返回值）
- **关键发现**：textrender.dll 不导入 GDI32.dll，使用自研/嵌入的软件字体光栅化器
- 最大函数 sub_1000DFE0（8,379 字节）是核心渲染函数，大量使用浮点运算
- 跨平台替代方案推荐 FreeType，因其精确控制和 CJK 支持

---

## 练习题与答案

### 题目 1：RTTI 修饰名解码

以下是从 textrender.dll 中提取的 RTTI 字符串，请解码并说明每个类的用途：

```
a) .?AU?$ncbNativeClassProperty@U?$PropertyCommand@VTextRenderBase@@P8TextRender@@BEMXZ...
b) .?AU?$ncbNativeClassMethod@U?$InvokeCommand@VTextRenderBase@@P8TextRender@@AE_NPB_WHHH_N@Z...
c) .?AU?$ncbNativeClassFactory@VTextRenderBase@@@@
```

<details>
<summary>查看答案</summary>

**a)** `ncbNativeClassProperty< PropertyCommand<TextRenderBase, float (TextRender::*)() const> >`
- 这是一个**只读浮点属性**的注册记录
- `BE` = const 成员函数，`M` = float 返回值，`XZ` = 无参数
- 可能对应 `size`（字体大小）属性的 getter

**b)** `ncbNativeClassMethod< InvokeCommand<TextRenderBase, bool (TextRender::*)(const wchar_t*, int, int, int, bool)> >`
- 这是一个**方法**的注册记录
- `AE` = 非 const，`_N` = bool 返回值
- 参数：`PB_W` = const wchar_t*，`H` = int（×3），`_N` = bool
- 这很可能是 `drawText(text, x, y, maxWidth, wrap)` 方法

**c)** `ncbNativeClassFactory<TextRenderBase>`
- 这是 ncbind 的**工厂类**，负责创建和销毁 TextRenderBase 的实例
- 当 TJS 脚本执行 `new TextRender()` 时，ncbind 通过这个工厂类调用 C++ 的 `new TextRender()`
- 当 TJS 垃圾回收器回收对象时，工厂类负责调用 `delete`

</details>

### 题目 2：为什么 textrender.dll 不导入 GDI32？

请从技术角度分析 textrender.dll 不使用 Windows GDI 进行文字渲染的可能原因（至少列出 3 个），并说明这对跨平台移植有什么影响。

<details>
<summary>查看答案</summary>

**不使用 GDI 的可能原因：**

1. **渲染一致性**：GDI 的文字渲染效果会受 Windows 版本、ClearType 设置、系统 DPI 等因素影响。游戏需要在所有玩家机器上呈现完全相同的文字外观，自研渲染器可以保证这一点

2. **精细控制**：GDI 的 `TextOut`/`DrawText` API 对字间距、基线偏移、子像素定位（Sub-pixel Positioning，在像素级以下精度放置字符）的控制能力有限。游戏的排版需求（如视觉小说的竖排文字、特殊对话框排版）可能超出 GDI 能力

3. **性能优化**：GDI 每次调用都涉及用户态到内核态的切换（因为 GDI 在 Windows 内核中实现），对于每帧渲染上百个字符的场景开销较大。自研渲染器可以直接在用户态内存中操作像素，避免这些切换

4. **直接像素操作**：KiriKiri 的 Layer 系统直接操作 BGRA 像素缓冲区，自研渲染器可以直接写入这个缓冲区，而 GDI 需要额外的 DIBSection + BitBlt 来传输像素

**对跨平台移植的影响：**

这实际上是**有利的**：由于原版不依赖 GDI，我们的跨平台实现不需要在 Linux/macOS/Android 上模拟 GDI 行为。使用 FreeType 等跨平台库就能实现功能等效的文本渲染，而且更容易在渲染效果上接近原版（因为双方都是软件光栅化）。

</details>

### 题目 3：编写 ncbind 属性注册代码

根据以下 RTTI 签名，编写对应的 ncbind 注册代码：

```
PropertyCommand<TextRenderBase, float(TextRender::*)() const, void(TextRender::*)(float)>
PropertyCommand<TextRenderBase, bool(TextRender::*)() const, void(TextRender::*)(bool)>
InvokeCommand<TextRenderBase, void(TextRender::*)(float,float)>
```

<details>
<summary>查看答案</summary>

```cpp
// 根据 RTTI 签名编写的 ncbind 注册代码
class TextRender : public TextRenderBase {
public:
    // float 读写属性
    float getSize() const { return size_; }
    void setSize(float v) { size_ = v; }

    // bool 读写属性
    bool getBold() const { return bold_; }
    void setBold(bool v) { bold_ = v; }

    // void(float,float) 方法
    void setRenderSize(float w, float h) {
        renderW_ = w;
        renderH_ = h;
    }

private:
    float size_ = 24.0f;
    bool bold_ = false;
    float renderW_ = 0, renderH_ = 0;
};

// ncbind 注册（对应三个 RTTI 签名）
NCB_REGISTER_CLASS(TextRenderBase) {
    // PropertyCommand<..., float getter, float setter>
    NCB_PROPERTY_RW(size, getSize, setSize);

    // PropertyCommand<..., bool getter, bool setter>
    NCB_PROPERTY_RW(bold, getBold, setBold);

    // InvokeCommand<..., void(float,float)>
    NCB_METHOD(setRenderSize);
}
```

**关键点**：
- `NCB_REGISTER_CLASS` 的参数是 `TextRenderBase`（不是 `TextRender`），与 RTTI 中 `ncbNativeClassAutoRegister<TextRenderBase>` 一致
- `NCB_PROPERTY_RW` 生成的模板实例会包含 getter 和 setter 两个函数指针，对应 RTTI 中的 `PropertyCommand` 模板参数
- `NCB_METHOD` 生成 `InvokeCommand` 模板实例，参数类型和返回值由编译器从函数指针自动推导

</details>

---

## 下一步

下一节 [02-渲染逻辑还原与跨平台适配](./02-渲染逻辑还原与跨平台适配.md) 将深入分析 TextRender 的核心渲染函数（sub_1000DFE0），还原文本排版和字形绘制算法，并设计跨平台的 FreeType 替代实现。
