# ExecuteCode 调度循环

> **所属模块：** M07-TJS2脚本引擎
> **前置知识：** [字节码编译·操作码生成](../04-字节码编译/01-GenNodeCode与操作码生成.md)、[寄存器分配与帧管理](../04-字节码编译/02-寄存器分配与帧管理.md)、[常量池与代码优化](../04-字节码编译/03-常量池与代码优化.md)
> **预计阅读时间：** 55 分钟

## 本节目标

读完本节后，你将能够：

1. 理解 TJS2 虚拟机的整体执行架构——从脚本文本到运行结果的完整链路
2. 掌握 `ExecuteAsFunction` 的寄存器布局方案：变量区、保留区、返回值、参数区的精确内存排列
3. 理解 `tTJSVariantArrayStack` 如何为每次函数调用分配/回收寄存器内存
4. 深入 `ExecuteCode` 主调度循环的 `while(true) + switch` 结构，掌握指令获取与分发机制
5. 理解 `flag` 布尔变量在条件跟踪中的核心作用，以及它与 `VM_TT`/`VM_TF`/`VM_JF`/`VM_JNF` 四条指令的配合

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 调度循环 | Dispatch Loop | VM 的核心执行引擎——一个无限循环，每次迭代取一条指令、执行、推进指令指针 |
| 寄存器式 VM | Register-based VM | 与栈式 VM（如 JVM）不同，操作数直接通过寄存器编号寻址，减少 push/pop 开销 |
| 指令指针 | Instruction Pointer (IP) | 指向当前正在执行的字节码指令的指针，在 TJS2 中是 `tjs_int32 *code` |
| 寄存器基址 | Register Area Base (ra) | 指向寄存器数组的指针，所有寄存器操作通过 `ra + 偏移` 寻址 |
| 数据区基址 | Data Area Base (da) | 指向常量池数组的指针，`VM_CONST` 等指令通过 `da + 索引` 取常量 |
| 变体数组栈 | Variant Array Stack | 一个预分配的 `tTJSVariant` 数组池，避免每次函数调用都 `new`/`delete` |
| 对象代理 | Object Proxy | `tTJSObjectProxy` 实例，实现双重派发——先查 objthis 再查 global 的成员查找策略 |
| 条件标志 | Condition Flag | 一个独立的布尔变量 `flag`，用于存储比较/测试指令的结果，跳转指令据此决定分支 |

---

## §1 TJS2 虚拟机执行架构总览

### 1.1 从脚本到执行的完整链路

TJS2 脚本从文本到运行结果，经历以下 5 个阶段：

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  TJS2    │    │  Token   │    │   AST    │    │ Bytecode │    │   VM     │
│  源代码  │───→│  流      │───→│  语法树  │───→│  指令流  │───→│  执行    │
│ (.tjs)   │    │ (Lexer)  │    │ (Parser) │    │ (CodeGen)│    │ (Exec)   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
   Ch01            Ch02            Ch03            Ch04          ← 本章 →
```

前四章分别讲解了词法分析（Lexer 将源代码拆分为 Token 流）、语法分析（Parser 将 Token 流构建为 AST 语法树）、字节码编译（CodeGen 将 AST 转换为字节码指令流）。本章聚焦最后一个环节：**虚拟机执行**（VM Execution）——将字节码指令逐条解释执行，产生实际运行效果。

### 1.2 寄存器式 VM vs 栈式 VM

虚拟机有两大架构流派：

| 特性 | 栈式 VM（Stack-based） | 寄存器式 VM（Register-based） |
|------|----------------------|---------------------------|
| 代表实现 | JVM、CPython、.NET CLR | Lua 5、Dalvik（Android）、**TJS2** |
| 操作数来源 | 操作数栈顶部 | 通过寄存器编号直接寻址 |
| 指令格式 | 通常无操作数（0-address） | 通常 2-3 个操作数（寄存器编号） |
| 指令数量 | 一个操作需要多条指令（push+push+op） | 一条指令完成（op reg1 reg2 reg3） |
| 指令条数 | 更多（约 1.5-2x） | 更少 |
| 单条指令宽度 | 窄（1-2 字节） | 宽（4-16 字节） |
| 总代码体积 | 相当 | 相当 |
| 运行效率 | 较慢（频繁的栈操作） | 较快（减少数据搬运） |

TJS2 选择了**寄存器式架构**。每条指令的操作数用寄存器偏移量表示，比如 `VM_ADD %8 %16` 表示"将寄存器偏移 8 处的值与寄存器偏移 16 处的值相加，结果存回偏移 8 处"。

> **为什么用字节偏移而不是编号？** TJS2 的寄存器偏移量以**字节**为单位（`sizeof(tTJSVariant)` 的倍数），而非简单的整数编号。这样在运行时可以直接用指针加法定位寄存器，避免乘法运算——`(tTJSVariant*)((char*)ra + offset)` 比 `ra[index]`（需要编译器生成 `index * sizeof(tTJSVariant)`）更高效。这就是 `TJS_GET_VM_REG_ADDR` 宏的工作原理。

### 1.3 核心数据结构关系

VM 执行涉及的核心数据结构关系如下：

```
tTJSScriptBlock (脚本块 —— 一个 .tjs 文件对应一个)
 └── tTJSInterCodeContext[] (上下文数组 —— 每个函数/类/属性一个)
      ├── CodeArea: tjs_int32*     ← 字节码指令数组（32位整数流）
      ├── DataArea: tTJSVariant*   ← 常量池数组（字符串、数值等常量）
      ├── MaxVariableCount: int    ← 局部变量最大数量
      ├── MaxFrameCount: int       ← 嵌套帧最大寄存器数
      ├── VariableReserveCount: int ← 保留寄存器数（通常为2）
      ├── FuncDeclArgCount: int    ← 函数声明参数个数
      ├── FuncDeclCollapseBase: int ← 可变参数折叠起始位置
      └── SuperClassGetter         ← 父类获取器（仅 ctClass 类型有）

tTJSVariantArrayStack (变体数组栈 —— 全局单例)
 └── tVariantArray[] (分段数组)
      ├── Array: tTJSVariant*     ← 预分配的 Variant 数组
      ├── Using: int              ← 已使用量
      └── Allocated: int          ← 总分配量
```

每当调用一个 TJS2 函数时，VM 从 `tTJSVariantArrayStack` 分配一块连续的 `tTJSVariant` 数组作为寄存器空间，在函数返回后归还。这个机制避免了频繁的堆分配。

---

## §2 VariantArrayStack —— 寄存器内存池

### 2.1 为什么需要内存池

每次 TJS2 函数调用都需要分配一组寄存器。如果每次都用 `new tTJSVariant[n]` + `delete[]`，会带来两个严重问题：

1. **堆分配开销**：`tTJSVariant` 是一个比较复杂的联合体类型（内部包含引用计数的字符串、对象指针等），频繁 new/delete 会产生大量内存碎片
2. **析构开销**：每个 `tTJSVariant` 的析构函数需要检查类型并释放对应资源（如减少字符串引用计数），大量短生命周期的 Variant 对象会导致大量析构调用

`tTJSVariantArrayStack` 通过**预分配 + 分段复用**的策略解决这两个问题。

### 2.2 数据结构

```cpp
// tjsInterCodeExec.h
class tTJSVariantArrayStack {
    struct tVariantArray {
        tTJSVariant *Array;     // 预分配的 Variant 数组
        tjs_int Using;          // 当前已使用的元素数量
        tjs_int Allocated;      // 数组总容量
    };

    tVariantArray *Arrays;              // 分段数组的数组
    tjs_int NumArraysAllocated;         // 已分配的段数
    tjs_int NumArraysUsing;             // 正在使用的段数
    tVariantArray *Current;             // 当前活跃段的指针
    tjs_int CompactVariantArrayMagic;   // 压缩触发计数器
    tjs_int OperationDisabledCount;     // 禁用操作计数（嵌套保护）
};
```

这里的关键思路是**分段数组**（segmented array）。整个栈不是一块连续内存，而是由多个段组成。每个段是一块 `tTJSVariant` 数组：

```
Arrays[0]: [████████████████████░░░░░░]  ← Using=16, Allocated=26
                                          ^Current
Arrays[1]: [░░░░░░░░░░░░░░░░░░░░░░░░░░]  ← 空闲段
```

### 2.3 分配流程（Allocate）

```cpp
// 简化的分配逻辑
tTJSVariant* tTJSVariantArrayStack::Allocate(tjs_int num) {
    // 1. 检查当前段是否有足够空间
    if (Current->Using + num > Current->Allocated) {
        // 2. 当前段空间不足，扩展到新段
        IncreaseVariantArray(num);
    }
    // 3. 从当前段的 Using 位置分配
    tTJSVariant *ret = Current->Array + Current->Using;
    Current->Using += num;
    return ret;
}
```

关键行为：

- **快速路径**：当前段有足够空间时，仅需一次指针加法和一次整数加法——极快
- **慢速路径**：当前段不够时，`IncreaseVariantArray` 会切换到下一个空闲段（或分配新段）
- **返回指针**：返回的是 `tTJSVariant*`，调用方直接使用这块内存作为寄存器数组

### 2.4 回收流程（Deallocate）

```cpp
void tTJSVariantArrayStack::Deallocate(tjs_int num, tTJSVariant *ptr) {
    // 1. 清空归还区域中所有 Variant（释放引用计数等资源）
    for (int i = 0; i < num; i++)
        ptr[i].Clear();
    
    // 2. 回退 Using 指针
    Current->Using -= num;
    
    // 3. 如果当前段已完全空闲，切回上一个段
    if (Current->Using == 0 && NumArraysUsing > 1) {
        DecreaseVariantArray();
    }
}
```

注意 `Deallocate` 会**立即清空**归还的 Variant 数组——这是必要的，因为 `tTJSVariant` 可能持有对象引用，不清空会导致内存泄漏。

### 2.5 内存压缩（Compact）

为避免内存只增不减，`tTJSVariantArrayStack` 有一个压缩机制：

```cpp
void tTJSVariantArrayStack::InternalCompact() {
    // 周期性检查（通过 CompactVariantArrayMagic 计数器）
    // 如果存在完全空闲的段，释放它们
    // 如果当前段的空闲比例超过阈值，缩小其分配量
}
```

全局函数 `TJSVariantArrayStackCompact()` 和 `TJSVariantArrayStackCompactNow()` 供外部触发压缩——通常在 GC（垃圾回收）周期中调用。

### 2.6 线程安全与引用计数

`tTJSVariantArrayStack` 是一个**线程局部**的数据结构（通过 `TJSVariantArrayStackAddRef()` / `TJSVariantArrayStackRelease()` 管理生命周期）。每个线程有自己的实例，所以不需要加锁：

```cpp
// ExecuteAsFunction 调用前后的配对操作
TJSVariantArrayStackAddRef();    // 确保栈存在
tTJSVariant *regs = TJSVariantArrayStack->Allocate(num_alloc);
// ... 执行 ...
TJSVariantArrayStack->Deallocate(num_alloc, regs);
TJSVariantArrayStackRelease();   // 减少引用计数
```

---

## §3 ExecuteAsFunction —— 函数调用入口

### 3.1 函数签名

```cpp
// tjsInterCodeExec.cpp 第 635 行
void tTJSInterCodeContext::ExecuteAsFunction(
    iTJSDispatch2 *objthis,      // 调用时的 this 对象
    tTJSVariant **args,          // 参数指针数组
    tjs_int numargs,             // 实际参数个数
    tTJSVariant *result,         // 返回值存放位置（可为 nullptr 表示忽略返回值）
    tjs_int start_ip             // 起始指令指针（通常为 0）
);
```

`ExecuteAsFunction` 是每个 TJS2 函数执行的**统一入口**。无论是顶层脚本、普通函数、类构造器、属性 getter/setter，都通过这个方法进入执行。它负责三件事：

1. **分配寄存器空间**（从 VariantArrayStack）
2. **布置寄存器初始状态**（objthis、参数、返回值槽位）
3. **调用 ExecuteCode 执行字节码**

### 3.2 寄存器分配

```cpp
// 计算需要分配的寄存器总数
tjs_int num_alloc = MaxVariableCount      // 局部变量数量
                  + VariableReserveCount  // 保留寄存器（通常为 2）
                  + 1                     // 返回值寄存器 ra[0]
                  + MaxFrameCount;        // 嵌套帧寄存器

// 从全局 VariantArrayStack 分配
TJSVariantArrayStackAddRef();
tTJSVariant *regs = TJSVariantArrayStack->Allocate(num_alloc);

// 计算寄存器基址 ra
tTJSVariant *ra = regs + MaxVariableCount + VariableReserveCount;
```

分配后的内存布局如下图（假设 `MaxVariableCount=3`，`VariableReserveCount=2`，`MaxFrameCount=4`）：

```
regs 指向这里
│
│ 局部变量区            保留区   返回值  帧寄存器区
│ (MaxVariableCount=3) (Reserve=2)      (MaxFrameCount=4)
▼                                ▲
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│var0│var1│var2│res1│res0│ rv │ f0 │ f1 │ f2 │ f3 │
│    │    │    │    │    │    │    │    │    │    │
│ -5 │ -4 │ -3 │ -2 │ -1 │  0 │ +1 │ +2 │ +3 │ +4 │ ← ra 相对索引
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
                          │
                          ra 指向这里
```

> **关键洞察**：`ra` 不是指向数组起点，而是指向**返回值寄存器**（索引 0 处）。局部变量和保留区在 `ra` 的**负偏移**方向，帧寄存器在**正偏移**方向。这样做的好处是：字节码中的正偏移寄存器用于临时值和帧操作，负偏移寄存器用于参数和变量——编译器生成代码时只需关心偏移方向。

### 3.3 保留寄存器的含义

`VariableReserveCount` 通常为 2（除了 `ctProperty` 类型为 0），对应两个特殊用途的寄存器：

| 寄存器 | 位置 | 内容 | 用途 |
|--------|------|------|------|
| `ra[-2]` | 保留 1 | `tTJSObjectProxy` 或全局对象 | 成员查找的"双重派发"代理 |
| `ra[-1]` | 保留 2 | 原始 objthis | 当闭包没有自己的 ObjThis 时的回退对象 |
| `ra[0]` | 返回值 | void（初始为空） | `VM_SRV` 指令将值写入此处 |

### 3.4 objthis 代理设置

```cpp
tTJSObjectProxy proxy;
if (objthis) {
    // 有 this 对象时：创建双重派发代理
    proxy.SetObjects(objthis, Block->GetTJS()->GetGlobalNoAddRef());
    ra[-2] = &proxy;        // ra[-2] 存代理
} else {
    // 无 this 对象时：全局对象同时充当 object 和 context
    ra[-2].SetObject(global, global);
}
ra[-1].SetObject(objthis, objthis);    // ra[-1] 始终存原始 objthis
ra[0].Clear();                          // ra[0] 清空为 void（返回值初始值）
```

`tTJSObjectProxy` 是一个轻量级的 `iTJSDispatch2` 实现，它的成员查找逻辑是：**先在 objthis 上查找，如果找不到（返回 `TJS_E_MEMBERNOTFOUND`），再到全局对象上查找**。这实现了 TJS2 的"当前对象优先，全局兜底"的名称解析语义。

```
成员查找流程：
   obj.member
       │
       ▼
  ┌──────────────────┐
  │ tTJSObjectProxy  │
  │                  │
  │  1. 查找 objthis │──→ 找到 → 返回
  │                  │
  │  2. 查找 global  │──→ 找到 → 返回
  │                  │
  │  3. 都没找到     │──→ TJS_E_MEMBERNOTFOUND
  └──────────────────┘
```

### 3.5 参数传递

函数参数存放在 `ra[-3]`、`ra[-4]`、`ra[-5]` 等位置（从 ra[-3] 开始向负方向增长）。这些位置实际上就是局部变量区的一部分：

```cpp
// 参数传递逻辑（简化）
if (numargs >= FuncDeclArgCount) {
    // 实参 >= 形参：拷贝声明数量的参数
    for (int i = 0; i < FuncDeclArgCount; i++)
        ra[-3 - i].CopyRef(*args[i]);
} else {
    // 实参 < 形参：拷贝可用参数，剩余清空为 void
    for (int i = 0; i < numargs; i++)
        ra[-3 - i].CopyRef(*args[i]);
    for (int i = numargs; i < FuncDeclArgCount; i++)
        ra[-3 - i].Clear();
}
```

带参数时的寄存器布局示例（函数声明 `function foo(a, b, c)`）：

```
┌────┬────┬────┬────┬────┬────┬────┬ ...
│ a  │ b  │ c  │prxy│this│ rv │ ...
│arg0│arg1│arg2│    │    │    │
│ra-5│ra-4│ra-3│ra-2│ra-1│ra 0│
└────┴────┴────┴────┴────┴────┴────┘
```

### 3.6 可变参数折叠

如果函数声明使用了 `*` 收集参数（如 `function foo(a, b, *)`），`FuncDeclCollapseBase` 会被设置为对应的折叠起始位置。此时多余的参数会被收集到一个 TJS2 Array 对象中：

```cpp
if (FuncDeclCollapseBase >= 0) {
    // 创建一个 Array 对象
    // 将 FuncDeclCollapseBase 之后的所有参数装入 Array
    // 将 Array 存入 ra[-3 - FuncDeclCollapseBase]
}
```

例如 `function foo(a, *)` 调用 `foo(1, 2, 3)` 时：

```
ra[-3] = 1         (参数 a)
ra[-4] = [2, 3]    (可变参数数组 *)
```

### 3.7 执行与清理

完成寄存器布置后，调用 `ExecuteCode` 执行字节码：

```cpp
// 进入主执行循环
ExecuteCode(ra, start_ip, args, numargs, result);

// 清理：释放 objthis 代理的引用
ra[-2].Clear();

// 归还寄存器空间
TJSVariantArrayStack->Deallocate(num_alloc, regs);
TJSVariantArrayStackRelease();
```

注意清理顺序：先 `Clear()` ra[-2]（释放代理持有的对象引用），再 `Deallocate`（清空并归还整个寄存器数组）。

---

## §4 ExecuteCode —— 主调度循环

### 4.1 函数签名与局部变量

`ExecuteCode` 是 TJS2 虚拟机的核心——一个巨大的 `while(true) + switch` 循环，逐条取指令、执行、推进指令指针。

```cpp
// tjsInterCodeExec.cpp 第 852 行
tjs_int tTJSInterCodeContext::ExecuteCode(
    tTJSVariant *ra_org,        // 寄存器基址（由 ExecuteAsFunction 计算好的 ra）
    tjs_int startip,            // 起始指令指针（字节码数组中的索引）
    tTJSVariant **args,         // 原始参数数组（传递给内部函数调用）
    tjs_int numargs,            // 原始参数个数
    tTJSVariant *result,        // 返回值存放位置
    bool tryCatch               // 是否处于 try-catch 块中（影响异常日志）
);
```

进入函数后，首先初始化五个关键局部变量：

```cpp
tjs_int32 *codesave;                           // 保存当前指令位置（异常回溯用）
tjs_int32 *code = codesave = CodeArea + startip;  // 指令指针，初始化为起始位置
tTJSVariant *ra = ra_org;                      // 寄存器基址（本地副本）
tTJSVariant *da = DataArea;                    // 数据区基址（常量池指针）
bool flag = false;                             // 条件标志
```

这五个变量的关系可以用一张图表示：

```
CodeArea (字节码数组)
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬──────┐
│  0  │  1  │  2  │  3  │  4  │  5  │ ... │  N  │ N+1  │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴──────┘
                    ▲
                    │
                   code (指令指针，不断前进)

DataArea (常量池数组)
┌─────────┬─────────┬─────────┬─────────┐
│ "hello" │  42     │  3.14   │ "world" │
└─────────┴─────────┴─────────┴─────────┘
     ▲
     │
    da (数据区基址，固定不变)

ra (寄存器数组)                    flag
┌────┬────┬────┬────┬────┐      ┌───────┐
│ra-2│ra-1│ra 0│ra+1│ra+2│      │ false │
└────┴────┴────┴────┴────┘      └───────┘
               ▲
               │
              ra (寄存器基址，固定不变)
```

> **为什么需要 `codesave`？** 当 VM 执行过程中抛出异常时，我们需要知道**是哪条指令**导致的异常。`codesave` 在每次循环迭代开始时保存当前 `code` 的值，这样异常处理代码就能通过 `codesave - CodeArea` 计算出出错指令的偏移量，再通过 `CodePosToSrcPos()` 映射回源代码行号。

### 4.2 栈跟踪器集成

如果启用了栈跟踪（`TJSStackTracerEnabled()` 返回 true），VM 会将 `codesave` 指针的地址注册到全局栈跟踪器中：

```cpp
if (TJSStackTracerEnabled())
    TJSStackTracerSetCodePointer(CodeArea, &codesave);
```

这里注册的是 `&codesave`（指针的地址），而不是 `codesave` 本身。这意味着栈跟踪器可以在任何时刻读取 `codesave` 的**当前值**来获知 VM 正在执行的指令位置——即使 `codesave` 不断变化，指向 `codesave` 的指针始终有效。这是一种非常巧妙的"实时观测"设计。

### 4.3 主循环结构

主循环的结构极其简洁——一个无限循环加一个 switch 语句：

```cpp
while (true) {
    codesave = code;         // 保存当前指令位置
    switch (*code) {         // 取当前指令的操作码
        case VM_NOP:         // 空操作
            code++;          // 推进到下一条指令
            break;

        case VM_CONST:       // 加载常量
            TJS_GET_VM_REG(ra, code[1]).CopyRef(TJS_GET_VM_REG(da, code[2]));
            code += 3;       // 操作码 + 目标寄存器 + 常量索引 = 3个word
            break;

        case VM_CP:          // 寄存器拷贝
            TJS_GET_VM_REG(ra, code[1]).CopyRef(TJS_GET_VM_REG(ra, code[2]));
            code += 3;
            break;

        // ... 60+ 个 case ...

        case VM_RET:         // 函数返回
            return code + 1 - CodeArea;  // 返回下一条指令的偏移量

        default:
            ThrowInvalidVMCode();  // 未知操作码，抛出异常
    }
}
```

每次循环做三件事：

1. **保存位置**：`codesave = code`——记录当前指令位置供异常回溯使用
2. **取指令**：`*code`——读取当前 `code` 指针指向的 32 位整数值作为操作码（opcode）
3. **分发执行**：`switch` 语句根据操作码跳转到对应的处理逻辑

### 4.4 指令指针推进规则

不同指令占用的 word 数不同，`code` 的推进量取决于指令格式：

| 指令格式 | Word 数 | code 推进 | 示例 |
|----------|---------|-----------|------|
| 无操作数 | 1 | `code++` | `VM_NOP`、`VM_NF`、`VM_REGMEMBER`、`VM_DEBUGGER` |
| 1 操作数 | 2 | `code += 2` | `VM_TT %r`、`VM_CL %r`、`VM_INC %r`、`VM_RET`（虽然直接 return） |
| 2 操作数 | 3 | `code += 3` | `VM_CONST %r1 #d`、`VM_CP %r1 %r2`、`VM_ADD %r1 %r2` |
| 3 操作数 | 4 | `code += 4` | `VM_GPD %r1 %obj .prop`、`VM_TYPEOFD %r1 %obj .prop` |
| 4 操作数 | 5 | `code += 5` | `VM_ADDPD %r1 %obj .prop %r2`（属性复合赋值） |
| 可变长度 | N | 由函数返回 | `VM_CALL`/`VM_NEW`（参数列表长度不定） |

> **关键区别**：大多数指令的 word 数在编译时就确定了，所以直接 `code += N`。但 `VM_CALL`、`VM_CALLD`、`VM_CALLI`、`VM_NEW` 这几条指令的操作数中包含参数列表，长度在运行时才知道，因此它们的处理函数（`CallFunction`、`CallFunctionDirect`、`CallFunctionIndirect`）会返回实际消耗的 word 数，由调用方用 `code += 返回值` 推进。

### 4.5 退出循环的唯一方式

`while(true)` 是一个无限循环，只有两种方式退出：

1. **正常返回**：执行到 `VM_RET` 或 `VM_EXTRY` 指令时，通过 `return code + 1 - CodeArea` 返回。返回值是**下一条指令的偏移量**（相对于 `CodeArea` 的索引），这在异常处理和嵌套调用中用于恢复执行位置
2. **异常退出**：执行过程中抛出异常，被 `ExecuteCode` 的外层 `try/catch` 捕获（详见 §5 异常处理）

> **`VM_EXTRY` 等同于 `VM_RET`**——两者的实现代码完全相同（`return code + 1 - CodeArea`）。`VM_EXTRY`（Exit Try）在语义上表示"离开 try 块"，而 `VM_RET` 表示"函数返回"，但在字节码层面它们的行为一致：都让 `ExecuteCode` 返回下一条指令的偏移量给调用方。

### 4.6 数据搬运指令

VM 中最基础的操作是数据搬运——在寄存器和常量池之间移动数据：

```cpp
case VM_CONST:  // 加载常量：ra[code[1]] = da[code[2]]
    TJS_GET_VM_REG(ra, code[1]).CopyRef(TJS_GET_VM_REG(da, code[2]));
    code += 3;
    break;

case VM_CP:     // 寄存器拷贝：ra[code[1]] = ra[code[2]]
    TJS_GET_VM_REG(ra, code[1]).CopyRef(TJS_GET_VM_REG(ra, code[2]));
    code += 3;
    break;

case VM_CL:     // 清空寄存器：ra[code[1]] = void
    TJS_GET_VM_REG(ra, code[1]).Clear();
    code += 2;
    break;

case VM_CCL:    // 连续清空：从 ra[code[1]] 开始清空 code[2] 个寄存器
    ContinuousClear(ra, code);
    code += 3;
    break;
```

`CopyRef` 是 `tTJSVariant` 的浅拷贝方法——对于字符串和对象，它增加引用计数而不是深拷贝数据。`Clear` 将 Variant 重置为 `void` 类型（TJS2 的"无值"状态，类似 JavaScript 的 `undefined`）。

`ContinuousClear` 的实现非常直接：

```cpp
// tjsInterCodeExec.cpp 第 1391 行
void tTJSInterCodeContext::ContinuousClear(
    tTJSVariant *ra, const tjs_int32 *code) {
    tTJSVariant *r = TJS_GET_VM_REG_ADDR(ra, code[1]);  // 起始寄存器
    tTJSVariant *rl = r + code[2];   // code[2] 是个数（不是字节偏移！）
    while (r < rl)
        (r++)->Clear();              // 逐个清空
}
```

> **注意**：`VM_CCL` 的第二个操作数 `code[2]` 是**元素个数**而非字节偏移——这是整个指令集中少数几个例外之一。大多数操作数中的寄存器位置都用字节偏移表示，但 `VM_CCL` 的计数操作数直接就是整数个数。

### 4.7 算术与逻辑运算指令

TJS2 对 14 种二元运算符（`||=`、`&&=`、`|=`、`^=`、`&=`、`>>=`、`<<=`、`>>>=`、`+=`、`-=`、`%=`、`/=`、`\=`、`*=`）使用了一个巧妙的宏来批量生成 case 分支：

```cpp
// tjsInterCodeExec.cpp 第 1010 行
#define TJS_DEF_VM_P(vmcode, rope)                      \
    case VM_##vmcode:                                   \
        TJS_GET_VM_REG(ra, code[1]).rope(               \
            TJS_GET_VM_REG(ra, code[2]));               \
        code += 3;                                      \
        break;                                          \
    case VM_##vmcode##PD:                               \
        OperatePropertyDirect(ra, code, TJS_OP_##vmcode);  \
        code += 5;                                      \
        break;                                          \
    case VM_##vmcode##PI:                               \
        OperatePropertyIndirect(ra, code, TJS_OP_##vmcode);\
        code += 5;                                      \
        break;                                          \
    case VM_##vmcode##P:                                \
        OperateProperty(ra, code, TJS_OP_##vmcode);     \
        code += 4;                                      \
        break
```

每个 `TJS_DEF_VM_P` 宏展开后生成 **4 个 case 分支**：

| 后缀 | 含义 | 操作 | Word 数 |
|------|------|------|---------|
| （无后缀） | 寄存器直接运算 | `ra[code[1]] op= ra[code[2]]` | 3 |
| `PD` | 属性直接访问运算 | `obj.name op= value` | 5 |
| `PI` | 属性间接访问运算 | `obj[key] op= value` | 5 |
| `P` | 当前对象属性运算 | `this.name op= value` | 4 |

然后用 14 次宏调用生成全部 56 条指令：

```cpp
TJS_DEF_VM_P(LOR,  logicalorequal);   // ||=
TJS_DEF_VM_P(LAND, logicalandequal);   // &&=
TJS_DEF_VM_P(BOR,  operator|=);       // |=
TJS_DEF_VM_P(BXOR, operator^=);       // ^=
TJS_DEF_VM_P(BAND, operator&=);       // &=
TJS_DEF_VM_P(SAR,  operator>>=);      // >>=（算术右移）
TJS_DEF_VM_P(SAL,  operator<<=);      // <<=
TJS_DEF_VM_P(SR,   rbitshiftequal);    // >>>=（逻辑右移）
TJS_DEF_VM_P(ADD,  operator+=);       // +=
TJS_DEF_VM_P(SUB,  operator-=);       // -=
TJS_DEF_VM_P(MOD,  operator%=);       // %=
TJS_DEF_VM_P(DIV,  operator/=);       // /=
TJS_DEF_VM_P(IDIV, idivequal);        // \=（整数除法）
TJS_DEF_VM_P(MUL,  operator*=);       // *=
```

> **TJS2 的设计哲学**：与 Java/Python 等 VM 不同，TJS2 不是把"取属性值→运算→写回属性"拆成三条指令，而是用单条 `VM_ADDPD` 类指令一步完成。这减少了指令数量，但增加了 case 分支数量——典型的"宽指令集"设计。

一元运算符（`++`、`--`、`!`、`~`）没有使用宏，而是手动编写。`VM_INC`/`VM_DEC` 同样有 `PD`/`PI`/`P` 三个变体，但由于它们是无额外操作数的一元运算，用了 `OperatePropertyDirect0`/`OperatePropertyIndirect0`/`OperateProperty0`（注意后缀 `0`，表示零额外操作数）：

```cpp
case VM_INC:     // 寄存器自增：ra[code[1]]++
    TJS_GET_VM_REG(ra, code[1]).increment();
    code += 2;
    break;
case VM_INCPD:   // 属性直接自增：obj.name++
    OperatePropertyDirect0(ra, code, TJS_OP_INC);
    code += 4;
    break;
case VM_INCPI:   // 属性间接自增：obj[key]++
    OperatePropertyIndirect0(ra, code, TJS_OP_INC);
    code += 4;
    break;
case VM_INCP:    // 当前对象属性自增：this.name++
    OperateProperty0(ra, code, TJS_OP_INC);
    code += 3;
    break;
```

### 4.8 类型转换指令

TJS2 提供了一组类型转换指令，对应 TJS2 脚本中的类型转换操作符（`int`、`real`、`string`）和内置函数（`typeof`、`#`、`$`）：

```cpp
case VM_INT:     // 转整数：ra[code[1]] = (int)ra[code[1]]
    TJS_GET_VM_REG(ra, code[1]).ToInteger();
    code += 2;
    break;

case VM_REAL:    // 转浮点：ra[code[1]] = (real)ra[code[1]]
    TJS_GET_VM_REG(ra, code[1]).ToReal();
    code += 2;
    break;

case VM_STR:     // 转字符串：ra[code[1]] = (string)ra[code[1]]
    TJS_GET_VM_REG(ra, code[1]).ToString();
    code += 2;
    break;

case VM_OCTET:   // 转八位组序列：ra[code[1]] = (octet)ra[code[1]]
    TJS_GET_VM_REG(ra, code[1]).ToOctet();
    code += 2;
    break;

case VM_NUM:     // 转数值：ra[code[1]] = +ra[code[1]]
    TJS_GET_VM_REG(ra, code[1]).tonumber();
    code += 2;
    break;

case VM_CHS:     // 取负：ra[code[1]] = -ra[code[1]]
    TJS_GET_VM_REG(ra, code[1]).changesign();
    code += 2;
    break;
```

`typeof` 操作符有三种变体：

```cpp
case VM_TYPEOF:   // typeof 寄存器值
    TypeOf(TJS_GET_VM_REG(ra, code[1]));
    code += 2;
    break;

case VM_TYPEOFD:  // typeof obj.member（直接属性访问）
    TypeOfMemberDirect(ra, code, TJS_MEMBERMUSTEXIST);
    code += 4;
    break;

case VM_TYPEOFI:  // typeof obj[key]（间接属性访问）
    TypeOfMemberIndirect(ra, code, TJS_MEMBERMUSTEXIST);
    code += 4;
    break;
```

`TypeOf` 函数（第 2601 行）将 Variant 类型映射为字符串：`tvtVoid`→`"void"`、`tvtObject`→`"Object"`、`tvtString`→`"String"`、`tvtInteger`→`"Integer"`、`tvtReal`→`"Real"`、`tvtOctet`→`"Octet"`。注意这些字符串是通过 `TJSMapGlobalStringMap` 预先驻留（intern）的，避免每次 typeof 都创建新字符串对象。

### 4.9 函数调用与返回指令

函数调用是 VM 中最复杂的操作之一：

```cpp
case VM_CALL:    // 直接调用：result = func(args...)
case VM_NEW:     // 构造调用：result = new Class(args...)
    code += CallFunction(ra, code, args, numargs);
    break;

case VM_CALLD:   // 属性直接调用：result = obj.method(args...)
    code += CallFunctionDirect(ra, code, args, numargs);
    break;

case VM_CALLI:   // 属性间接调用：result = obj[key](args...)
    code += CallFunctionIndirect(ra, code, args, numargs);
    break;
```

`CallFunction`/`CallFunctionDirect`/`CallFunctionIndirect` 返回值是**本条指令占用的 word 数**，因此 `code += 返回值` 会正确推进到下一条指令。这些函数内部使用 `TJS_BEGIN_FUNC_CALL_ARGS` 宏来解析参数列表（详见下一节"VM指令集详解"）。

返回相关的指令：

```cpp
case VM_SRV:     // 设置返回值：*result = ra[code[1]]
    if (result)
        result->CopyRef(TJS_GET_VM_REG(ra, code[1]));
    code += 2;
    break;

case VM_RET:     // 函数返回
    return code + 1 - CodeArea;

case VM_EXTRY:   // 离开 try 块（等同于 VM_RET）
    return code + 1 - CodeArea;
```

`VM_SRV`（Set Return Value）将寄存器的值拷贝到 `result` 指针指向的位置。`result` 可以为 `nullptr`（表示调用方不需要返回值），此时 `VM_SRV` 是一个空操作。通常一个函数的字节码末尾会是 `VM_SRV %寄存器` + `VM_RET` 的组合。

### 4.10 属性访问指令

属性访问分为"直接访问"（成员名在编译时已知，用常量池中的字符串表示）和"间接访问"（成员名在运行时才确定，用寄存器中的值表示）：

```cpp
// 直接属性读取
case VM_GPD:     // ra[code[1]] = ra[code[2]].da[code[3]]
    GetPropertyDirect(ra, code, 0);
    code += 4;
    break;

case VM_GPDS:    // 同上但跳过 property handler（直接读值而非触发 getter）
    GetPropertyDirect(ra, code, TJS_IGNOREPROP);
    code += 4;
    break;

// 直接属性写入
case VM_SPD:     // ra[code[2]].da[code[3]] = ra[code[1]]
    SetPropertyDirect(ra, code, 0);
    code += 4;
    break;

case VM_SPDE:    // 同上但确保成员存在（不存在则创建）
    SetPropertyDirect(ra, code, TJS_MEMBERENSURE);
    code += 4;
    break;

case VM_SPDEH:   // 同上但创建隐藏成员
    SetPropertyDirect(ra, code, TJS_MEMBERENSURE | TJS_HIDDENMEMBER);
    code += 4;
    break;

case VM_SPDS:    // 同上但跳过 property handler（直接写值而非触发 setter）
    SetPropertyDirect(ra, code, TJS_MEMBERENSURE | TJS_IGNOREPROP);
    code += 4;
    break;
```

标志位的含义：

| 标志 | 含义 | 使用场景 |
|------|------|----------|
| `0` | 默认行为 | 正常属性读写，触发 getter/setter |
| `TJS_IGNOREPROP` | 忽略属性处理器 | 直接读写底层值，跳过 property 对象的 get/set |
| `TJS_MEMBERENSURE` | 确保成员存在 | 如果成员不存在则自动创建 |
| `TJS_HIDDENMEMBER` | 隐藏成员 | 创建的成员不可枚举（用于类内部成员注册） |

### 4.11 异常处理指令

```cpp
case VM_ENTRY:   // 进入 try 块
    code = CodeArea + ExecuteCodeInTryBlock(
        ra,
        code - CodeArea + 3,              // try 块的起始 IP（跳过 VM_ENTRY 本身的 3 个 word）
        args, numargs, result,
        TJS_FROM_VM_CODE_ADDR(code[1]) + code - CodeArea,  // catch 块的 IP
        TJS_FROM_VM_REG_ADDR(code[2])     // 异常对象存放的寄存器偏移
    );
    break;

case VM_EXTRY:   // 离开 try 块
    return code + 1 - CodeArea;

case VM_THROW:   // 抛出异常
    ThrowScriptException(
        TJS_GET_VM_REG(ra, code[1]),         // 异常值
        Block,                                // 所属脚本块
        CodePosToSrcPos(code - CodeArea)      // 源码位置
    );
    code += 2;  // 实际上不会执行到这里（ThrowScriptException 会抛出 C++ 异常）
    break;
```

`VM_ENTRY` 的三个操作数含义：

1. `code[0]` = `VM_ENTRY`（操作码本身）
2. `code[1]` = catch 块的**代码偏移量**（字节偏移，需要用 `TJS_FROM_VM_CODE_ADDR` 转换为 word 索引）
3. `code[2]` = 异常对象存放的**寄存器偏移量**（字节偏移，需要用 `TJS_FROM_VM_REG_ADDR` 转换为 Variant 索引）

### 4.12 对象与类相关指令

```cpp
case VM_CHGTHIS:  // 改变闭包的 objthis
    TJS_GET_VM_REG(ra, code[1]).ChangeClosureObjThis(
        TJS_GET_VM_REG(ra, code[2]).AsObjectNoAddRef());
    code += 3;
    break;

case VM_GLOBAL:   // 获取全局对象
    TJS_GET_VM_REG(ra, code[1]) = Block->GetTJS()->GetGlobalNoAddRef();
    code += 2;
    break;

case VM_ADDCI:    // 添加类实例信息
    AddClassInstanceInfo(ra, code);
    code += 3;
    break;

case VM_REGMEMBER: // 注册对象成员（将当前上下文的所有成员复制到 objthis）
    RegisterObjectMember(ra[-1].AsObjectNoAddRef());
    code++;
    break;

case VM_DEBUGGER:  // 触发原生调试器断点
    TJSNativeDebuggerBreak();
    code++;
    break;
```

`VM_REGMEMBER` 在类构造过程中使用——当 `new MyClass()` 执行时，VM 先创建一个空的 `tTJSCustomObject`，然后执行类体（class body）字节码。类体中的 `VM_REGMEMBER` 指令将当前 `InterCodeContext`（代表类定义）的所有成员（方法、属性、常量等）复制到新创建的对象实例上。

其他辅助指令：

```cpp
case VM_ASC:     // 取字符编码：ra[code[1]] = 字符串首字符的Unicode码
    CharacterCodeOf(TJS_GET_VM_REG(ra, code[1]));
    code += 2;
    break;

case VM_CHR:     // Unicode码转字符：ra[code[1]] = 对应的单字符字符串
    CharacterCodeFrom(TJS_GET_VM_REG(ra, code[1]));
    code += 2;
    break;

case VM_INV:     // 使对象无效化（invalidate）
    TJS_GET_VM_REG(ra, code[1]) =
        TJS_GET_VM_REG(ra, code[1]).Type() != tvtObject
        ? false
        : (TJS_GET_VM_REG(ra, code[1])
               .AsObjectClosureNoAddRef()
               .Invalidate(0, nullptr, nullptr,
                           ra[-1].AsObjectNoAddRef()) == TJS_S_TRUE);
    code += 2;
    break;

case VM_CHKINS:  // instanceof 检查
    InstanceOf(TJS_GET_VM_REG(ra, code[2]),
               TJS_GET_VM_REG(ra, code[1]));
    code += 3;
    break;

case VM_EVAL:    // 动态求值（eval）
    Eval(TJS_GET_VM_REG(ra, code[1]),
         TJSEvalOperatorIsOnGlobal ? nullptr : ra[-1].AsObjectNoAddRef(),
         true);   // true = 执行表达式
    code += 2;
    break;

case VM_EEXP:    // 动态求值（eval expression）
    Eval(TJS_GET_VM_REG(ra, code[1]),
         TJSEvalOperatorIsOnGlobal ? nullptr : ra[-1].AsObjectNoAddRef(),
         false);  // false = 执行语句
    code += 2;
    break;
```

---

## §5 条件标志（flag）机制

### 5.1 为什么不用寄存器存布尔值

大多数寄存器式 VM（如 Lua 5）将比较结果直接存入寄存器，跳转指令再从寄存器读取。TJS2 选择了一种更简洁的方案：使用一个**独立的布尔变量 `flag`** 来存储条件结果。

这个设计有两个优势：

1. **节省寄存器**：不需要为每个条件分支分配一个临时寄存器来存放布尔值
2. **指令更紧凑**：跳转指令只需一个操作数（跳转目标地址），不需要额外指定"从哪个寄存器读条件"

缺点是：**flag 是全局的**，两条连续的条件指令之间不能插入其他条件指令（否则 flag 会被覆盖）。编译器必须保证"设置 flag"和"使用 flag"之间没有其他修改 flag 的指令。

### 5.2 设置 flag 的指令

有 6 条指令会**写入** flag：

```cpp
// 真值测试：flag = bool(ra[code[1]])
case VM_TT:
    flag = TJS_GET_VM_REG(ra, code[1]).operator bool();
    code += 2;
    break;

// 假值测试：flag = !bool(ra[code[1]])
case VM_TF:
    flag = !(TJS_GET_VM_REG(ra, code[1]).operator bool());
    code += 2;
    break;

// 相等比较（类型转换后比较，类似 JavaScript 的 ==）
case VM_CEQ:
    flag = TJS_GET_VM_REG(ra, code[1])
               .NormalCompare(TJS_GET_VM_REG(ra, code[2]));
    code += 3;
    break;

// 严格相等比较（不做类型转换，类似 JavaScript 的 ===）
case VM_CDEQ:
    flag = TJS_GET_VM_REG(ra, code[1])
               .DiscernCompare(TJS_GET_VM_REG(ra, code[2]));
    code += 3;
    break;

// 小于比较：flag = (ra[code[2]] < ra[code[1]])
// 注意：实际调用的是 GreaterThan，操作数顺序反转！
case VM_CLT:
    flag = TJS_GET_VM_REG(ra, code[1])
               .GreaterThan(TJS_GET_VM_REG(ra, code[2]));
    code += 3;
    break;

// 大于比较：flag = (ra[code[2]] > ra[code[1]])
case VM_CGT:
    flag = TJS_GET_VM_REG(ra, code[1])
               .LittlerThan(TJS_GET_VM_REG(ra, code[2]));
    code += 3;
    break;
```

> **操作数顺序陷阱**：`VM_CLT`（Compare Less Than）调用的是 `GreaterThan`——看起来反直觉，但这是因为 TJS2 的比较语义是 `code[1].GreaterThan(code[2])`，即"code[1] 是否大于 code[2]"。当编译器需要测试 `a < b` 时，它生成 `VM_CLT %b %a`（即 `b.GreaterThan(a)` = `b > a` = `a < b`）。方法名和指令名的命名方向恰好相反，阅读源码时需要特别注意。

### 5.3 使用 flag 的指令

有 4 条指令会**读取** flag：

```cpp
// 条件跳转：如果 flag 为 true 则跳转
case VM_JF:
    if (flag)
        TJS_ADD_VM_CODE_ADDR(code, code[1]);  // 跳转
    else
        code += 2;                             // 不跳转，继续执行
    break;

// 条件跳转：如果 flag 为 false 则跳转
case VM_JNF:
    if (!flag)
        TJS_ADD_VM_CODE_ADDR(code, code[1]);
    else
        code += 2;
    break;

// 将 flag 值存入寄存器
case VM_SETF:
    TJS_GET_VM_REG(ra, code[1]) = flag;
    code += 2;
    break;

// 将 flag 取反后存入寄存器
case VM_SETNF:
    TJS_GET_VM_REG(ra, code[1]) = !flag;
    code += 2;
    break;
```

还有一条指令直接**翻转** flag：

```cpp
// 取反 flag
case VM_NF:
    flag = !flag;
    code++;
    break;
```

### 5.4 条件分支的完整执行流程

以 TJS2 脚本 `if (x > 10) { ... } else { ... }` 为例，编译器生成的字节码序列如下：

```
指令 0:  VM_CONST   %8, #0       ; %8 = 10（从常量池加载）
指令 3:  VM_CGT     %-3, %8      ; flag = (%-3 > %8)，即 x > 10
指令 6:  VM_JNF     +offset      ; 如果 flag==false，跳转到 else 分支
指令 8:  ...                     ; if 分支体
指令 N:  VM_JMP     +offset2     ; 跳过 else 分支
指令 N+2: ...                    ; else 分支体
指令 M:  ...                     ; if-else 之后的代码
```

执行流程图：

```
VM_CONST %8, #0     ←── 加载常量 10
         │
         ▼
VM_CGT %-3, %8      ←── 比较 x 和 10，结果写入 flag
         │
    ┌────┴────┐
    │ flag?   │
    ▼         ▼
  true      false
    │         │
    ▼         ▼
  if体     VM_JNF 跳转
    │         │
    ▼         ▼
 VM_JMP    else体
    │         │
    └────┬────┘
         ▼
      后续代码
```

### 5.5 无条件跳转

```cpp
case VM_JMP:   // 无条件跳转
    TJS_ADD_VM_CODE_ADDR(code, code[1]);
    break;
```

`VM_JMP` 用于循环的回跳（`while`/`for` 循环体末尾）、`break`/`continue`、以及 `if-else` 中 if 分支末尾跳过 else 分支。跳转目标通过 `TJS_ADD_VM_CODE_ADDR` 宏实现——它将 `code[1]` 作为**字节偏移量**加到 `code` 指针上（详见 §6）。

### 5.6 逻辑非指令

`VM_LNOT` 对寄存器中的值执行逻辑非操作，与 `VM_NF` 不同：

```cpp
case VM_LNOT:   // 逻辑非：ra[code[1]] = !ra[code[1]]（操作寄存器值）
    TJS_GET_VM_REG(ra, code[1]).logicalnot();
    code += 2;
    break;

case VM_NF:     // 标志取反：flag = !flag（操作 flag 变量）
    flag = !flag;
    code++;
    break;
```

`VM_LNOT` 修改的是**寄存器中的 Variant 值**（调用 `logicalnot()` 方法），而 `VM_NF` 修改的是**flag 布尔变量**。两者作用于完全不同的目标。

---

## §6 地址转换宏

### 6.1 为什么需要地址转换

TJS2 的字节码中，寄存器偏移量和跳转偏移量都以**字节**为单位存储，而不是以"元素个数"为单位。这样做的原因是：运行时可以直接用指针算术定位目标，省去乘法运算。

但这意味着编译期（生成字节码时）和运行期（执行字节码时）需要在"逻辑索引"和"字节偏移"之间进行转换。`tjsInterCodeGen.h` 定义了一组宏来完成这项工作。

### 6.2 宏定义与解释

```cpp
// tjsInterCodeGen.h 第 34-41 行

// 逻辑索引 → 字节偏移（编译期使用）
#define TJS_TO_VM_CODE_ADDR(x)   ((x) * (tjs_int)sizeof(tjs_uint32))
#define TJS_TO_VM_REG_ADDR(x)    ((x) * (tjs_int)sizeof(tTJSVariant))

// 字节偏移 → 逻辑索引（运行期使用）
#define TJS_FROM_VM_CODE_ADDR(x) ((tjs_int)(x) / (tjs_int)sizeof(tjs_uint32))
#define TJS_FROM_VM_REG_ADDR(x)  ((tjs_int)(x) / (tjs_int)sizeof(tTJSVariant))

// 指令指针跳转：将字节偏移加到 code 指针上
#define TJS_ADD_VM_CODE_ADDR(dest, x) ((*(char **)&(dest)) += (x))

// 寄存器寻址：基址 + 字节偏移 → tTJSVariant 指针
#define TJS_GET_VM_REG_ADDR(base, x) \
    ((tTJSVariant *)((char *)(base) + (tjs_int)(x)))

// 寄存器寻址：基址 + 字节偏移 → tTJSVariant 引用
#define TJS_GET_VM_REG(base, x) (*(TJS_GET_VM_REG_ADDR(base, x)))
```

### 6.3 逐个宏详解

#### TJS_TO_VM_CODE_ADDR / TJS_TO_VM_REG_ADDR

**用途**：编译期将逻辑索引转换为字节偏移量，写入字节码操作数。

```cpp
// 编译器想表示"跳转到前方第5条指令处"
// 字节码中存储的是字节偏移：5 * sizeof(tjs_uint32) = 5 * 4 = 20
int code_offset = TJS_TO_VM_CODE_ADDR(5);   // 结果：20

// 编译器想表示"第3个寄存器"
// 字节码中存储的是字节偏移：3 * sizeof(tTJSVariant) = 3 * 16 = 48（假设 Variant 为 16 字节）
int reg_offset = TJS_TO_VM_REG_ADDR(3);     // 结果：48
```

#### TJS_FROM_VM_CODE_ADDR / TJS_FROM_VM_REG_ADDR

**用途**：运行期将字节码中的字节偏移量转换回逻辑索引。

```cpp
// VM_ENTRY 指令的 code[1] 存储的是 catch 块的字节偏移
// 需要转换为 word 索引才能与 code 指针配合使用
tjs_int catch_index = TJS_FROM_VM_CODE_ADDR(code[1]);
// catch 块的绝对位置
tjs_int catch_ip = catch_index + (code - CodeArea);

// VM_ENTRY 指令的 code[2] 存储的是异常寄存器的字节偏移
tjs_int exobj_reg = TJS_FROM_VM_REG_ADDR(code[2]);
```

#### TJS_ADD_VM_CODE_ADDR

**用途**：运行期跳转——将字节偏移量直接加到指令指针上。

```cpp
// 等价于：code = (tjs_int32 *)((char *)code + code[1])
TJS_ADD_VM_CODE_ADDR(code, code[1]);
```

这个宏的关键技巧是 `(*(char **)&(dest))` ——先将 `dest`（类型为 `tjs_int32*`）的地址转为 `char**`，再解引用得到 `char*`，然后在 `char*` 上做字节级加法。这样即使 `code[1]` 是字节偏移量（可能不是 4 的倍数——虽然在实践中总是 4 的倍数），也能正确偏移。

> **跳转方向**：`code[1]` 可以是正数（向前跳转，用于 `while`/`for` 循环体末尾回跳）或负数的字节偏移（向后跳转）。但在 TJS2 的实现中，跳转偏移通常是正数——因为编译器总是先输出条件判断代码再输出分支体，跳转目标在当前位置的后方。回跳到循环头部用的是负偏移。

#### TJS_GET_VM_REG_ADDR / TJS_GET_VM_REG

**用途**：运行期寄存器寻址——根据字节偏移量从寄存器基址定位一个 `tTJSVariant`。

```cpp
// 假设 ra 指向寄存器数组中间（返回值寄存器处）
// code[1] = 48（字节偏移，即第 3 个 Variant）
tTJSVariant *target = TJS_GET_VM_REG_ADDR(ra, code[1]);
// 等价于：(tTJSVariant *)((char *)ra + 48)

// TJS_GET_VM_REG 多加一层解引用
TJS_GET_VM_REG(ra, code[1]) = 42;
// 等价于：*((tTJSVariant *)((char *)ra + code[1])) = 42
```

> **负偏移也能工作**：`code[1]` 可以是负数。例如访问 `ra[-3]`（第一个参数）时，`code[1]` 的值是 `-3 * sizeof(tTJSVariant)`。`char*` 加法对负数同样有效，因此宏可以正确访问 `ra` 前方的寄存器。

### 6.4 地址转换示意图

```
编译期                                    运行期
───────                                  ───────

逻辑索引 5                               字节码中存储 20
    │                                         │
    │ TJS_TO_VM_CODE_ADDR(5)                  │
    │ = 5 * sizeof(tjs_uint32)                │
    │ = 5 * 4 = 20                            │
    ▼                                         ▼
  写入字节码 ─────────────────────→  读取字节码
                                              │
                                    TJS_FROM_VM_CODE_ADDR(20)
                                    = 20 / sizeof(tjs_uint32)
                                    = 20 / 4 = 5
                                              │
                                              ▼
                                        逻辑索引 5

寄存器编号 3                             字节码中存储 48
    │                                         │
    │ TJS_TO_VM_REG_ADDR(3)                   │
    │ = 3 * sizeof(tTJSVariant)               │
    │ = 3 * 16 = 48                           │
    ▼                                         ▼
  写入字节码 ─────────────────────→  TJS_GET_VM_REG(ra, 48)
                                    = *(tTJSVariant*)((char*)ra + 48)
                                    = ra[3] （第3个寄存器）
```

### 6.5 常见错误与排查

**错误 1：混淆字节偏移与逻辑索引**

```cpp
// 错误：直接用 code[1] 作为数组索引
tTJSVariant *bad = &ra[code[1]];     // ❌ code[1] 是字节偏移，不是数组索引！

// 正确：用宏进行地址转换
tTJSVariant *good = TJS_GET_VM_REG_ADDR(ra, code[1]);  // ✅
```

**错误 2：跳转时忘记字节偏移**

```cpp
// 错误：用逻辑索引跳转
code += code[1];          // ❌ code[1] 是字节偏移，但 code++ 是 word 级别

// 正确：用宏做字节级跳转
TJS_ADD_VM_CODE_ADDR(code, code[1]);  // ✅
```

**错误 3：在序列化时忘记转换**

在字节码导出（`ExportByteCode`）时，32 位的 code word 需要压缩为 16 位。此时地址偏移量也需要相应调整，否则跳转目标会指向错误位置。这是字节码序列化中最容易出错的地方之一。

---

## §7 动手实践

### 示例 1：手写一个最小 VM 调度循环

下面我们用 C++ 从零实现一个极简版的 TJS2 调度循环，支持 5 条指令：`NOP`、`CONST`（加载整数常量）、`ADD`（加法）、`PRINT`（打印寄存器值）和 `HALT`（停机）。

```cpp
// mini_vm.cpp —— 最小寄存器式 VM
#include <iostream>
#include <cstdint>
#include <vector>
#include <variant>

// 操作码定义
enum Opcode : int32_t {
    OP_NOP   = 0,  // 空操作                    | 1 word
    OP_CONST = 1,  // CONST reg, imm            | 3 words
    OP_ADD   = 2,  // ADD dst, src              | 3 words
    OP_PRINT = 3,  // PRINT reg                 | 2 words
    OP_HALT  = 4,  // 停机                       | 1 word
};

// 简化的 Variant 类型（只支持 void 和 integer）
struct MiniVariant {
    enum Type { Void, Integer } type = Void;
    int64_t value = 0;

    MiniVariant() : type(Void), value(0) {}
    MiniVariant(int64_t v) : type(Integer), value(v) {}

    void Clear() { type = Void; value = 0; }

    // 模拟 TJS2 的 CopyRef
    void CopyRef(const MiniVariant &src) {
        type = src.type;
        value = src.value;
    }

    // 模拟 TJS2 的 operator+=
    void AddAssign(const MiniVariant &other) {
        value += other.value;
        type = Integer;
    }

    friend std::ostream& operator<<(std::ostream &os, const MiniVariant &v) {
        if (v.type == Void) os << "void";
        else os << v.value;
        return os;
    }
};

// 执行函数 —— 模拟 ExecuteCode
void Execute(int32_t *codeArea, MiniVariant *dataArea,
             MiniVariant *ra, int startip) {
    int32_t *code = codeArea + startip;  // 指令指针

    while (true) {
        switch (*code) {
            case OP_NOP:
                code++;
                break;

            case OP_CONST:  // CONST reg_offset, data_index
                ra[code[1]].CopyRef(dataArea[code[2]]);
                code += 3;
                break;

            case OP_ADD:    // ADD dst_offset, src_offset
                ra[code[1]].AddAssign(ra[code[2]]);
                code += 3;
                break;

            case OP_PRINT:  // PRINT reg_offset
                std::cout << "reg[" << code[1] << "] = "
                          << ra[code[1]] << std::endl;
                code += 2;
                break;

            case OP_HALT:   // 停机
                std::cout << "VM halted." << std::endl;
                return;

            default:
                std::cerr << "Unknown opcode: " << *code << std::endl;
                return;
        }
    }
}

int main() {
    // 常量池（模拟 DataArea）
    MiniVariant dataArea[] = {
        MiniVariant(10),    // #0 = 10
        MiniVariant(32),    // #1 = 32
    };

    // 字节码（模拟 CodeArea）
    // 计算 10 + 32 并打印结果
    int32_t codeArea[] = {
        OP_CONST, 0, 0,    // reg[0] = dataArea[0]  → reg[0] = 10
        OP_CONST, 1, 1,    // reg[1] = dataArea[1]  → reg[1] = 32
        OP_ADD,   0, 1,    // reg[0] += reg[1]      → reg[0] = 42
        OP_PRINT, 0,       // print(reg[0])         → "reg[0] = 42"
        OP_HALT,           // 停机
    };

    // 寄存器数组（模拟 VariantArrayStack 分配）
    MiniVariant registers[8];

    Execute(codeArea, dataArea, registers, 0);
    return 0;
}
```

**编译与运行**：

```bash
# Linux / macOS
g++ -std=c++17 -o mini_vm mini_vm.cpp && ./mini_vm

# Windows (MSVC)
cl /std:c++17 /EHsc mini_vm.cpp && mini_vm.exe
```

**预期输出**：

```
reg[0] = 42
VM halted.
```

### 示例 2：添加条件跳转支持（模拟 flag 机制）

在示例 1 的基础上添加 `CMP_GT`（大于比较）、`JF`（flag 为 true 则跳转）、`JNF`（flag 为 false 则跳转）三条指令：

```cpp
// mini_vm_branch.cpp —— 支持条件跳转的 VM
#include <iostream>
#include <cstdint>

enum Opcode : int32_t {
    OP_NOP    = 0,
    OP_CONST  = 1,   // CONST reg, imm_index     | 3 words
    OP_ADD    = 2,   // ADD dst, src              | 3 words
    OP_PRINT  = 3,   // PRINT reg                 | 2 words
    OP_HALT   = 4,   // HALT                      | 1 word
    OP_CMP_GT = 5,   // CMP_GT reg1, reg2         | 3 words  flag = reg1 > reg2
    OP_JF     = 6,   // JF offset                 | 2 words  if flag, jump
    OP_JNF    = 7,   // JNF offset                | 2 words  if !flag, jump
    OP_JMP    = 8,   // JMP offset                | 2 words  unconditional jump
    OP_DEC    = 9,   // DEC reg                   | 2 words  reg--
};

struct Variant {
    int64_t value = 0;
    Variant() = default;
    Variant(int64_t v) : value(v) {}
    friend std::ostream& operator<<(std::ostream &os, const Variant &v) {
        os << v.value; return os;
    }
};

void Execute(int32_t *code, Variant *data, Variant *ra) {
    bool flag = false;  // 模拟 TJS2 的 flag 变量

    while (true) {
        switch (*code) {
            case OP_NOP:    code++; break;
            case OP_CONST:  ra[code[1]].value = data[code[2]].value; code += 3; break;
            case OP_ADD:    ra[code[1]].value += ra[code[2]].value; code += 3; break;
            case OP_DEC:    ra[code[1]].value--; code += 2; break;
            case OP_PRINT:  std::cout << "reg[" << code[1] << "] = " << ra[code[1]] << std::endl; code += 2; break;

            case OP_CMP_GT:  // flag = reg[code[1]] > reg[code[2]]
                flag = ra[code[1]].value > ra[code[2]].value;
                code += 3;
                break;

            case OP_JF:      // if (flag) jump forward by code[1] words
                if (flag) code += code[1];
                else code += 2;
                break;

            case OP_JNF:     // if (!flag) jump forward by code[1] words
                if (!flag) code += code[1];
                else code += 2;
                break;

            case OP_JMP:     // unconditional jump
                code += code[1];
                break;

            case OP_HALT:
                std::cout << "VM halted." << std::endl;
                return;

            default:
                std::cerr << "Unknown opcode: " << *code << std::endl;
                return;
        }
    }
}

int main() {
    Variant data[] = { Variant(5), Variant(1), Variant(0) };
    // 程序：计算 5 + 4 + 3 + 2 + 1（求和 1..5）
    // reg[0] = 5 (计数器)，reg[1] = 0 (累加器)，reg[2] = 0 (零值比较基准)
    int32_t prog[] = {
        // 初始化
        OP_CONST, 0, 0,       // 0:  reg[0] = 5（计数器）
        OP_CONST, 1, 2,       // 3:  reg[1] = 0（累加器）
        OP_CONST, 2, 2,       // 6:  reg[2] = 0（零值基准）
        // 循环开始 (IP=9)
        OP_CMP_GT, 0, 2,     // 9:  flag = reg[0] > 0 ?
        OP_JNF,    10,        // 12: if !flag (reg[0]<=0), jump to HALT（+10 → IP=22）
        OP_ADD,    1, 0,      // 14: reg[1] += reg[0]
        OP_DEC,    0,         // 17: reg[0]--
        OP_JMP,    -10,       // 19: jump back to loop start（-10 → IP=9）
        // 循环结束
        OP_PRINT,  1,         // 21: print reg[1]（结果应为 15）
        OP_HALT,              // 23: 停机
    };

    Variant regs[8];
    Execute(prog, data, regs);
    return 0;
}
```

**预期输出**：

```
reg[1] = 15
VM halted.
```

> **对比 TJS2**：这里的 `OP_JF`/`OP_JNF` 使用 word 级别的相对偏移（`code += code[1]`），而 TJS2 使用字节级别的偏移（`TJS_ADD_VM_CODE_ADDR(code, code[1])`）。TJS2 的方式更灵活但需要宏辅助，我们的简化版用 word 偏移更直观。

### 示例 3：模拟 VariantArrayStack 的分配/回收

```cpp
// variant_stack_sim.cpp —— 模拟 VariantArrayStack 的分段分配
#include <iostream>
#include <cstring>

struct Variant {
    int value = 0;
    bool in_use = false;
    void Clear() { value = 0; in_use = false; }
};

// 简化版 VariantArrayStack
class VariantStack {
    static const int SEGMENT_SIZE = 16;  // 每个段的容量

    struct Segment {
        Variant data[SEGMENT_SIZE];
        int using_count = 0;
    };

    Segment segments[4];       // 最多 4 个段
    int current_segment = 0;   // 当前活跃段索引
    int num_segments = 1;      // 已使用的段数

public:
    // 分配 num 个 Variant
    Variant* Allocate(int num) {
        Segment &seg = segments[current_segment];

        // 当前段空间不足？切换到下一个段
        if (seg.using_count + num > SEGMENT_SIZE) {
            if (current_segment + 1 >= 4) {
                std::cerr << "Stack overflow!" << std::endl;
                return nullptr;
            }
            current_segment++;
            if (current_segment >= num_segments)
                num_segments = current_segment + 1;
            std::cout << "  [Stack] Switched to segment " << current_segment << std::endl;
        }

        Segment &cur = segments[current_segment];
        Variant *ret = &cur.data[cur.using_count];
        cur.using_count += num;

        std::cout << "  [Stack] Allocated " << num << " variants"
                  << " (segment " << current_segment
                  << ", using " << cur.using_count
                  << "/" << SEGMENT_SIZE << ")" << std::endl;
        return ret;
    }

    // 回收 num 个 Variant
    void Deallocate(int num, Variant *ptr) {
        // 清空归还的 Variant
        for (int i = 0; i < num; i++)
            ptr[i].Clear();

        Segment &seg = segments[current_segment];
        seg.using_count -= num;

        std::cout << "  [Stack] Deallocated " << num << " variants"
                  << " (segment " << current_segment
                  << ", using " << seg.using_count
                  << "/" << SEGMENT_SIZE << ")" << std::endl;

        // 当前段完全空闲，切回上一段
        if (seg.using_count == 0 && current_segment > 0) {
            current_segment--;
            std::cout << "  [Stack] Switched back to segment " << current_segment << std::endl;
        }
    }
};

// 模拟函数调用
void SimulateCall(VariantStack &stack, const char *name, int regs_needed) {
    std::cout << "Call " << name << " (need " << regs_needed << " regs):" << std::endl;
    Variant *regs = stack.Allocate(regs_needed);
    if (regs) {
        // 模拟函数执行...
        regs[0].value = 42;
        std::cout << "  [Exec] " << name << " executed, reg[0] = " << regs[0].value << std::endl;
    }
    stack.Deallocate(regs_needed, regs);
    std::cout << std::endl;
}

int main() {
    VariantStack stack;

    std::cout << "=== Simulating TJS2 function call stack ===" << std::endl << std::endl;

    // 模拟嵌套函数调用
    std::cout << "--- Level 1: main() ---" << std::endl;
    Variant *main_regs = stack.Allocate(10);  // main 函数需要 10 个寄存器

    std::cout << std::endl << "--- Level 2: foo() called from main ---" << std::endl;
    Variant *foo_regs = stack.Allocate(6);   // foo 需要 6 个寄存器

    std::cout << std::endl << "--- Level 3: bar() called from foo ---" << std::endl;
    SimulateCall(stack, "bar", 4);            // bar 需要 4 个寄存器（调用完立即返回）

    std::cout << "--- foo() returns ---" << std::endl;
    stack.Deallocate(6, foo_regs);

    std::cout << std::endl << "--- main() returns ---" << std::endl;
    stack.Deallocate(10, main_regs);

    return 0;
}
```

**预期输出**（段容量为 16 时）：

```
=== Simulating TJS2 function call stack ===

--- Level 1: main() ---
  [Stack] Allocated 10 variants (segment 0, using 10/16)

--- Level 2: foo() called from main ---
  [Stack] Allocated 6 variants (segment 0, using 16/16)

--- Level 3: bar() called from foo ---
Call bar (need 4 regs):
  [Stack] Switched to segment 1
  [Stack] Allocated 4 variants (segment 1, using 4/16)
  [Exec] bar executed, reg[0] = 42
  [Stack] Deallocated 4 variants (segment 1, using 0/16)
  [Stack] Switched back to segment 0

--- foo() returns ---
  [Stack] Deallocated 6 variants (segment 0, using 10/16)

--- main() returns ---
  [Stack] Deallocated 10 variants (segment 0, using 0/16)
```

### 示例 4：验证地址转换宏的正确性

```cpp
// addr_macro_test.cpp —— 验证 TJS2 地址转换宏
#include <iostream>
#include <cstdint>
#include <cstring>
#include <cassert>

// 模拟 TJS2 类型定义
using tjs_int = int32_t;
using tjs_int32 = int32_t;
using tjs_uint32 = uint32_t;

// 简化的 Variant（16 字节对齐）
struct alignas(16) tTJSVariant {
    int64_t value;
    int64_t padding;
};

// 从 tjsInterCodeGen.h 复制的宏定义
#define TJS_TO_VM_CODE_ADDR(x)   ((x) * (tjs_int)sizeof(tjs_uint32))
#define TJS_TO_VM_REG_ADDR(x)    ((x) * (tjs_int)sizeof(tTJSVariant))
#define TJS_FROM_VM_CODE_ADDR(x) ((tjs_int)(x) / (tjs_int)sizeof(tjs_uint32))
#define TJS_FROM_VM_REG_ADDR(x)  ((tjs_int)(x) / (tjs_int)sizeof(tTJSVariant))
#define TJS_ADD_VM_CODE_ADDR(dest, x) ((*(char **)&(dest)) += (x))
#define TJS_GET_VM_REG_ADDR(base, x) \
    ((tTJSVariant *)((char *)(base) + (tjs_int)(x)))
#define TJS_GET_VM_REG(base, x) (*(TJS_GET_VM_REG_ADDR(base, x)))

int main() {
    std::cout << "sizeof(tjs_uint32) = " << sizeof(tjs_uint32) << std::endl;
    std::cout << "sizeof(tTJSVariant) = " << sizeof(tTJSVariant) << std::endl;

    // 测试 1：代码地址转换（逻辑索引 ↔ 字节偏移）
    std::cout << "\n--- Code address conversion ---" << std::endl;
    for (int i = 0; i < 5; i++) {
        tjs_int byte_offset = TJS_TO_VM_CODE_ADDR(i);
        tjs_int back = TJS_FROM_VM_CODE_ADDR(byte_offset);
        std::cout << "  index " << i << " -> byte " << byte_offset
                  << " -> index " << back << std::endl;
        assert(back == i);  // 往返转换必须一致
    }

    // 测试 2：寄存器地址转换
    std::cout << "\n--- Register address conversion ---" << std::endl;
    for (int i = -2; i <= 3; i++) {
        tjs_int byte_offset = TJS_TO_VM_REG_ADDR(i);
        tjs_int back = TJS_FROM_VM_REG_ADDR(byte_offset);
        std::cout << "  reg[" << i << "] -> byte " << byte_offset
                  << " -> reg[" << back << "]" << std::endl;
        assert(back == i);
    }

    // 测试 3：TJS_GET_VM_REG 寻址
    std::cout << "\n--- Register addressing ---" << std::endl;
    tTJSVariant regs[8] = {};
    tTJSVariant *ra = &regs[3];  // ra 指向中间位置（模拟返回值寄存器）

    // 正向访问
    TJS_GET_VM_REG(ra, TJS_TO_VM_REG_ADDR(0)).value = 100;  // ra[0]
    TJS_GET_VM_REG(ra, TJS_TO_VM_REG_ADDR(1)).value = 200;  // ra[1]
    TJS_GET_VM_REG(ra, TJS_TO_VM_REG_ADDR(2)).value = 300;  // ra[2]

    // 负向访问（参数/局部变量）
    TJS_GET_VM_REG(ra, TJS_TO_VM_REG_ADDR(-1)).value = -10;  // ra[-1]
    TJS_GET_VM_REG(ra, TJS_TO_VM_REG_ADDR(-2)).value = -20;  // ra[-2]

    for (int i = -2; i <= 2; i++) {
        auto &v = TJS_GET_VM_REG(ra, TJS_TO_VM_REG_ADDR(i));
        std::cout << "  ra[" << i << "].value = " << v.value << std::endl;
    }

    // 测试 4：TJS_ADD_VM_CODE_ADDR 跳转
    std::cout << "\n--- Code pointer jumping ---" << std::endl;
    tjs_int32 codeArea[] = {0, 1, 2, 3, 4, 5, 6, 7};
    tjs_int32 *code = codeArea;
    std::cout << "  Initial: *code = " << *code << std::endl;

    // 向前跳 3 个 word
    TJS_ADD_VM_CODE_ADDR(code, TJS_TO_VM_CODE_ADDR(3));
    std::cout << "  After +3 words: *code = " << *code << std::endl;
    assert(*code == 3);

    // 向后跳 1 个 word
    TJS_ADD_VM_CODE_ADDR(code, TJS_TO_VM_CODE_ADDR(-1));
    std::cout << "  After -1 word: *code = " << *code << std::endl;
    assert(*code == 2);

    std::cout << "\nAll assertions passed!" << std::endl;
    return 0;
}
```

**预期输出**：

```
sizeof(tjs_uint32) = 4
sizeof(tTJSVariant) = 16

--- Code address conversion ---
  index 0 -> byte 0 -> index 0
  index 1 -> byte 4 -> index 1
  index 2 -> byte 8 -> index 2
  index 3 -> byte 12 -> index 3
  index 4 -> byte 16 -> index 4

--- Register address conversion ---
  reg[-2] -> byte -32 -> reg[-2]
  reg[-1] -> byte -16 -> reg[-1]
  reg[0] -> byte 0 -> reg[0]
  reg[1] -> byte 16 -> reg[1]
  reg[2] -> byte 32 -> reg[2]
  reg[3] -> byte 48 -> reg[3]

--- Register addressing ---
  ra[-2].value = -20
  ra[-1].value = -10
  ra[0].value = 100
  ra[1].value = 200
  ra[2].value = 300

--- Code pointer jumping ---
  Initial: *code = 0
  After +3 words: *code = 3
  After -1 word: *code = 2

All assertions passed!
```

### 示例 5：字节码反汇编器（Disassembler）

利用对 VM 指令格式的理解，写一个简单的反汇编器，将 TJS2 风格的字节码还原为可读文本：

```cpp
// tjs2_disasm.cpp —— TJS2 字节码反汇编器
#include <iostream>
#include <cstdint>
#include <string>
#include <vector>
#include <sstream>

using tjs_int32 = int32_t;

// 操作码枚举（简化版，只包含本节讲解的核心指令）
enum VMCode : tjs_int32 {
    VM_NOP = 0, VM_CONST, VM_CP, VM_CL, VM_CCL,
    VM_TT, VM_TF, VM_CEQ, VM_CDEQ, VM_CLT, VM_CGT,
    VM_SETF, VM_SETNF, VM_LNOT, VM_NF,
    VM_JF, VM_JNF, VM_JMP,
    VM_INC, VM_DEC = 22,
    VM_ADD = 42,
    VM_SRV = 100, VM_RET, VM_ENTRY, VM_EXTRY, VM_THROW,
    VM_HALT = 255,  // 自定义的停机指令
};

// 操作码名称表
const char* OpcodeName(tjs_int32 op) {
    switch (op) {
        case VM_NOP:    return "NOP";
        case VM_CONST:  return "CONST";
        case VM_CP:     return "CP";
        case VM_CL:     return "CL";
        case VM_CCL:    return "CCL";
        case VM_TT:     return "TT";
        case VM_TF:     return "TF";
        case VM_CEQ:    return "CEQ";
        case VM_CDEQ:   return "CDEQ";
        case VM_CLT:    return "CLT";
        case VM_CGT:    return "CGT";
        case VM_SETF:   return "SETF";
        case VM_SETNF:  return "SETNF";
        case VM_LNOT:   return "LNOT";
        case VM_NF:     return "NF";
        case VM_JF:     return "JF";
        case VM_JNF:    return "JNF";
        case VM_JMP:    return "JMP";
        case VM_INC:    return "INC";
        case VM_DEC:    return "DEC";
        case VM_ADD:    return "ADD";
        case VM_SRV:    return "SRV";
        case VM_RET:    return "RET";
        case VM_ENTRY:  return "ENTRY";
        case VM_EXTRY:  return "EXTRY";
        case VM_THROW:  return "THROW";
        case VM_HALT:   return "HALT";
        default:        return "???";
    }
}

// 反汇编一条指令，返回该指令占用的 word 数
int Disassemble(const tjs_int32 *code, int ip) {
    std::cout << "  [" << ip << "] ";

    switch (code[0]) {
        case VM_NOP:
            std::cout << "NOP" << std::endl;
            return 1;

        case VM_CONST:
            std::cout << "CONST  %" << code[1] << ", #" << code[2] << std::endl;
            return 3;

        case VM_CP:
            std::cout << "CP     %" << code[1] << ", %" << code[2] << std::endl;
            return 3;

        case VM_CL:
            std::cout << "CL     %" << code[1] << std::endl;
            return 2;

        case VM_CCL:
            std::cout << "CCL    %" << code[1] << ", count=" << code[2] << std::endl;
            return 3;

        case VM_TT:
            std::cout << "TT     %" << code[1]
                      << "    ; flag = bool(%" << code[1] << ")" << std::endl;
            return 2;

        case VM_TF:
            std::cout << "TF     %" << code[1]
                      << "    ; flag = !bool(%" << code[1] << ")" << std::endl;
            return 2;

        case VM_CEQ:
            std::cout << "CEQ    %" << code[1] << ", %" << code[2]
                      << "    ; flag = (%" << code[1] << " == %" << code[2] << ")" << std::endl;
            return 3;

        case VM_CGT:
            std::cout << "CGT    %" << code[1] << ", %" << code[2]
                      << "    ; flag = (%" << code[2] << " > %" << code[1] << ")" << std::endl;
            return 3;

        case VM_JF:
            std::cout << "JF     " << code[1]
                      << "    ; if flag, goto [" << (ip + code[1]) << "]" << std::endl;
            return 2;

        case VM_JNF:
            std::cout << "JNF    " << code[1]
                      << "    ; if !flag, goto [" << (ip + code[1]) << "]" << std::endl;
            return 2;

        case VM_JMP:
            std::cout << "JMP    " << code[1]
                      << "    ; goto [" << (ip + code[1]) << "]" << std::endl;
            return 2;

        case VM_INC:
            std::cout << "INC    %" << code[1] << std::endl;
            return 2;

        case VM_ADD:
            std::cout << "ADD    %" << code[1] << ", %" << code[2] << std::endl;
            return 3;

        case VM_SRV:
            std::cout << "SRV    %" << code[1]
                      << "    ; *result = %" << code[1] << std::endl;
            return 2;

        case VM_RET:
            std::cout << "RET" << std::endl;
            return 1;

        case VM_HALT:
            std::cout << "HALT" << std::endl;
            return 1;

        default:
            std::cout << OpcodeName(code[0]) << "  (opcode " << code[0] << ")" << std::endl;
            return 1;  // 未知指令假定 1 word
    }
}

int main() {
    // 示例字节码：计算 1+2+3+...+n（n 从常量池加载）
    tjs_int32 program[] = {
        VM_CONST, 0, 0,    // [0]  reg[0] = data[0] (n)
        VM_CONST, 1, 1,    // [3]  reg[1] = data[1] (sum=0)
        VM_CONST, 2, 1,    // [6]  reg[2] = data[1] (0, for comparison)
        // loop:
        VM_CGT,   2, 0,    // [9]  flag = (reg[0] > reg[2]) ?
        VM_JNF,   7,       // [12] if !flag, goto [19] (exit loop)
        VM_ADD,   1, 0,    // [14] reg[1] += reg[0]
        VM_DEC,   0,       // [17] reg[0]--      (opcode 22 in our enum)
        VM_JMP,   -10,     // [19] goto [9] (loop)
        VM_SRV,   1,       // [21] *result = reg[1]
        VM_RET,            // [23] return
    };

    std::cout << "=== TJS2 Bytecode Disassembly ===" << std::endl;
    int ip = 0;
    int total = sizeof(program) / sizeof(tjs_int32);
    while (ip < total) {
        int size = Disassemble(&program[ip], ip);
        ip += size;
    }

    return 0;
}
```

**预期输出**：

```
=== TJS2 Bytecode Disassembly ===
  [0] CONST  %0, #0
  [3] CONST  %1, #1
  [6] CONST  %2, #1
  [9] CGT    %2, %0    ; flag = (%0 > %2)
  [12] JNF    7    ; if !flag, goto [19]
  [14] ADD    %0, %1
  [17] DEC  (opcode 22)
  [19] JMP    -10    ; goto [9]
  [21] SRV    %1    ; *result = %1
  [23] RET
```

### 示例 6：跟踪 flag 变化——调试辅助工具

```cpp
// flag_tracer.cpp —— 在 VM 执行过程中跟踪 flag 状态变化
#include <iostream>
#include <cstdint>

using tjs_int32 = int32_t;

// 带 flag 跟踪的 VM 执行器
class TracingVM {
    bool flag_ = false;
    int step_ = 0;

    void TraceFlag(const char *instr, bool new_flag) {
        bool old = flag_;
        flag_ = new_flag;
        std::cout << "  Step " << (step_++) << ": " << instr
                  << " -> flag: " << (old ? "true" : "false")
                  << " => " << (flag_ ? "true" : "false");
        if (old != flag_) std::cout << "  [CHANGED]";
        std::cout << std::endl;
    }

public:
    void Execute(tjs_int32 *code, int64_t *regs) {
        while (true) {
            switch (*code) {
                case 0: // NOP
                    code++;
                    break;

                case 1: // LOAD reg, imm
                    regs[code[1]] = code[2];
                    std::cout << "  Step " << (step_++) << ": LOAD reg["
                              << code[1] << "] = " << code[2] << std::endl;
                    code += 3;
                    break;

                case 2: // CMP_EQ reg1, reg2 → flag = (reg1 == reg2)
                    TraceFlag("CMP_EQ", regs[code[1]] == regs[code[2]]);
                    code += 3;
                    break;

                case 3: // CMP_GT reg1, reg2 → flag = (reg1 > reg2)
                    TraceFlag("CMP_GT", regs[code[1]] > regs[code[2]]);
                    code += 3;
                    break;

                case 4: // JF offset → if flag, jump
                    std::cout << "  Step " << (step_++) << ": JF (flag="
                              << (flag_ ? "true" : "false") << ") -> "
                              << (flag_ ? "JUMP" : "FALL-THROUGH") << std::endl;
                    if (flag_) code += code[1];
                    else code += 2;
                    break;

                case 5: // JNF offset → if !flag, jump
                    std::cout << "  Step " << (step_++) << ": JNF (flag="
                              << (flag_ ? "true" : "false") << ") -> "
                              << (!flag_ ? "JUMP" : "FALL-THROUGH") << std::endl;
                    if (!flag_) code += code[1];
                    else code += 2;
                    break;

                case 6: // NF → flag = !flag
                    TraceFlag("NF (negate)", !flag_);
                    code++;
                    break;

                case 7: // TT reg → flag = bool(reg)
                    TraceFlag("TT", regs[code[1]] != 0);
                    code += 2;
                    break;

                case 99: // HALT
                    std::cout << "  Step " << (step_++) << ": HALT (final flag="
                              << (flag_ ? "true" : "false") << ")" << std::endl;
                    return;

                default:
                    std::cerr << "Unknown opcode: " << *code << std::endl;
                    return;
            }
        }
    }
};

int main() {
    int64_t regs[8] = {};

    // 测试程序：检查 reg[0] > 5，然后取反 flag，再检查 reg[1] == 0
    tjs_int32 prog[] = {
        1,  0, 10,     // LOAD reg[0] = 10
        1,  1, 0,      // LOAD reg[1] = 0
        1,  2, 5,      // LOAD reg[2] = 5
        3,  0, 2,      // CMP_GT reg[0], reg[2] → flag = (10 > 5) = true
        4,  4,          // JF +4 → if flag, skip next 2 instructions
        1,  3, 999,    // LOAD reg[3] = 999 (should be skipped)
        6,             // NF → flag = !flag → false
        7,  1,         // TT reg[1] → flag = bool(0) = false
        2,  1, 2,      // CMP_EQ reg[1], reg[2] → flag = (0 == 5) = false
        99,            // HALT
    };

    std::cout << "=== Flag Tracing Demo ===" << std::endl;
    TracingVM vm;
    vm.Execute(prog, regs);

    return 0;
}
```

**预期输出**：

```
=== Flag Tracing Demo ===
  Step 0: LOAD reg[0] = 10
  Step 1: LOAD reg[1] = 0
  Step 2: LOAD reg[2] = 5
  Step 3: CMP_GT -> flag: false => true  [CHANGED]
  Step 4: JF (flag=true) -> JUMP
  Step 5: TT -> flag: true => false  [CHANGED]
  Step 6: CMP_EQ -> flag: false => false
  Step 7: HALT (final flag=false)
```

---

## §8 对照项目源码

本节涉及的项目源码文件和关键行号：

| 文件 | 行号范围 | 内容 |
|------|----------|------|
| `cpp/core/tjs2/tjsInterCodeExec.cpp` | 851-855 | `ExecuteCode` 函数签名和参数说明 |
| `cpp/core/tjs2/tjsInterCodeExec.cpp` | 857-871 | 局部变量初始化（`codesave`、`code`、`ra`、`da`、`flag`）和 `while(true)` 循环入口 |
| `cpp/core/tjs2/tjsInterCodeExec.cpp` | 872-1302 | 完整的 `switch(*code)` 分支——所有 VM 指令的 case 处理 |
| `cpp/core/tjs2/tjsInterCodeExec.cpp` | 1010-1043 | `TJS_DEF_VM_P` 宏定义和 14 次宏调用（生成 56 条算术/逻辑属性指令） |
| `cpp/core/tjs2/tjsInterCodeExec.cpp` | 1300-1333 | `ExecuteCode` 的异常处理：`catch(eTJSSilent)` / `catch(eTJSScriptError)` / `catch(eTJS)` / `catch(exception)` / `catch(const char*)` |
| `cpp/core/tjs2/tjsInterCodeExec.cpp` | 1336-1388 | `ExecuteCodeInTryBlock` 函数——try/catch 块的实际执行 |
| `cpp/core/tjs2/tjsInterCodeExec.cpp` | 1391-1397 | `ContinuousClear` 函数——`VM_CCL` 的实现 |
| `cpp/core/tjs2/tjsInterCodeExec.h` | 19-50 | `tTJSVariantArrayStack` 类定义 |
| `cpp/core/tjs2/tjsInterCodeGen.h` | 34-41 | 所有地址转换宏定义（`TJS_TO_VM_CODE_ADDR`、`TJS_GET_VM_REG` 等） |
| `cpp/core/tjs2/tjsInterCodeGen.h` | 42 | `TJS_NORMAL_AND_PROPERTY_ACCESSER` 宏定义 |
| `cpp/core/tjs2/tjsInterCodeGen.h` | 44-127 | `tTJSVMCodes` 枚举——完整操作码定义 |

> **建议阅读顺序**：先看 `tjsInterCodeGen.h` 的宏定义和枚举（理解指令格式），再看 `tjsInterCodeExec.cpp` 的 `ExecuteCode` 函数（理解指令执行），最后看 `ExecuteAsFunction`（理解调用约定）。

---

## §9 本节小结

- **TJS2 采用寄存器式 VM 架构**，每条指令通过字节偏移量直接寻址寄存器，避免栈式 VM 的频繁 push/pop 开销
- **`tTJSVariantArrayStack`** 是一个分段内存池，通过预分配+分段复用策略为函数调用分配寄存器空间，避免频繁的堆分配和析构开销
- **`ExecuteAsFunction`** 是函数执行的统一入口，负责计算寄存器布局（局部变量区 + 保留区 + 返回值 + 帧区）、设置 objthis 代理、传递参数，然后调用 `ExecuteCode`
- **`ExecuteCode`** 是 VM 的核心——一个 `while(true) + switch(*code)` 的调度循环，每次迭代取一条指令、执行、推进 `code` 指针
- **条件标志 `flag`** 是独立于寄存器的布尔变量，由 `VM_TT`/`VM_TF`/`VM_CEQ`/`VM_CDEQ`/`VM_CLT`/`VM_CGT` 设置，由 `VM_JF`/`VM_JNF`/`VM_SETF`/`VM_SETNF` 使用
- **地址转换宏** 在"逻辑索引"和"字节偏移"之间进行转换：`TJS_TO_VM_*` 用于编译期，`TJS_FROM_VM_*`/`TJS_GET_VM_*`/`TJS_ADD_VM_*` 用于运行期
- **`TJS_DEF_VM_P` 宏**批量生成 14×4=56 条算术/逻辑属性指令，是 TJS2"宽指令集"设计的典型体现
- **`VM_RET`/`VM_EXTRY`** 是退出 `while(true)` 循环的唯一正常方式，返回下一条指令的偏移量给调用方

---

## §10 练习题与答案

### 题目 1：指令格式分析

给定以下 TJS2 字节码片段（word 索引从 0 开始），写出每条指令的名称、操作数含义、以及 `code` 指针推进的步长：

```
[0]  VM_CONST  (=1)
[1]  48         (操作数 1)
[2]  0          (操作数 2)
[3]  VM_CONST  (=1)
[4]  64         (操作数 1)
[5]  1          (操作数 2)
[6]  VM_ADD    (=42)
[7]  48         (操作数 1)
[8]  64         (操作数 2)
[9]  VM_SRV    (=100)
[10] 48         (操作数 1)
[11] VM_RET    (=101)
```

<details>
<summary>查看答案</summary>

| Word索引 | 指令 | 操作数含义 | code 推进 |
|----------|------|-----------|----------|
| 0-2 | `VM_CONST` | `ra[48字节偏移] = da[0]`，即将常量池第 0 个值加载到第 3 个寄存器（48/16=3） | `code += 3` |
| 3-5 | `VM_CONST` | `ra[64字节偏移] = da[1]`，即将常量池第 1 个值加载到第 4 个寄存器（64/16=4） | `code += 3` |
| 6-8 | `VM_ADD` | `ra[48] += ra[64]`，即 reg[3] += reg[4] | `code += 3` |
| 9-10 | `VM_SRV` | `if(result) result->CopyRef(ra[48])`，设置返回值为 reg[3] | `code += 2` |
| 11 | `VM_RET` | `return 12 - CodeArea`，函数返回 | 不推进，直接 return |

**整个程序的语义**：将常量池前两个值相加，返回结果。等价于 TJS2 脚本 `return data[0] + data[1];`

</details>

### 题目 2：flag 机制追踪

以下字节码片段模拟了 `while (x > 0) { x--; }` 循环。假设初始时 `ra[%-3]`（即 x）的值为 3，`ra[%8]` 的值为 0（常量 0）。请追踪每次循环迭代中 `flag` 的值变化，以及 `x` 的值变化：

```
loop:
  [0]  VM_CGT    %-3, %8      ; flag = (x 的寄存器.LittlerThan(0 的寄存器))，即 flag = (x > 0)
  [3]  VM_JNF    +8           ; if !flag, goto [11]（退出循环）
  [5]  VM_DEC    %-3          ; x--
  [7]  VM_JMP    -7           ; goto [0]（回到循环开头）
exit:
  [9]  ...
```

<details>
<summary>查看答案</summary>

**初始状态**：x = 3, flag = false

| 迭代 | 执行指令 | x 值 | flag 值 | 跳转结果 |
|------|----------|------|---------|----------|
| 1 | `VM_CGT %-3, %8` | 3 | **true**（3 > 0） | — |
| 1 | `VM_JNF +8` | 3 | true | 不跳转（flag 为 true） |
| 1 | `VM_DEC %-3` | **2** | true | — |
| 1 | `VM_JMP -7` | 2 | true | 跳回 [0] |
| 2 | `VM_CGT %-3, %8` | 2 | **true**（2 > 0） | — |
| 2 | `VM_JNF +8` | 2 | true | 不跳转 |
| 2 | `VM_DEC %-3` | **1** | true | — |
| 2 | `VM_JMP -7` | 1 | true | 跳回 [0] |
| 3 | `VM_CGT %-3, %8` | 1 | **true**（1 > 0） | — |
| 3 | `VM_JNF +8` | 1 | true | 不跳转 |
| 3 | `VM_DEC %-3` | **0** | true | — |
| 3 | `VM_JMP -7` | 0 | true | 跳回 [0] |
| 4 | `VM_CGT %-3, %8` | 0 | **false**（0 不大于 0） | — |
| 4 | `VM_JNF +8` | 0 | false | **跳转到 [11]**（退出循环） |

**总结**：循环执行 3 次，x 从 3 递减到 0 后退出。flag 在前 3 次迭代的 CGT 指令中为 true，第 4 次迭代为 false 触发退出。

</details>

### 题目 3：地址转换计算

假设 `sizeof(tTJSVariant)` = 16 字节，`sizeof(tjs_uint32)` = 4 字节。回答以下问题：

1. 编译器要在字节码中表示"第 -2 个寄存器"（即 ra[-2]），应该写入什么数值？
2. 编译器要表示"向前跳过 5 条指令"的跳转偏移量，应该写入什么数值？
3. 给定字节码操作数值 `-96`（寄存器偏移），它对应 ra 数组的第几个元素？
4. 给定字节码操作数值 `20`（代码偏移），它对应向前跳过几个 word？

<details>
<summary>查看答案</summary>

1. **`TJS_TO_VM_REG_ADDR(-2)` = -2 × 16 = -32**。字节码中写入 `-32`。运行时通过 `TJS_GET_VM_REG(ra, -32)` 定位到 `ra[-2]`。

2. **`TJS_TO_VM_CODE_ADDR(5)` = 5 × 4 = 20**。字节码中写入 `20`。运行时通过 `TJS_ADD_VM_CODE_ADDR(code, 20)` 将 code 指针向前移动 20 字节（= 5 个 word）。

3. **`TJS_FROM_VM_REG_ADDR(-96)` = -96 / 16 = -6**。对应 `ra[-6]`，即 ra 基址往回第 6 个 Variant 位置。在典型的寄存器布局中，这可能是第 4 个参数（ra[-3] 到 ra[-N] 是参数区）。

4. **`TJS_FROM_VM_CODE_ADDR(20)` = 20 / 4 = 5**。对应向前跳过 5 个 word（即 5 条单 word 指令，或更少的多 word 指令——具体取决于指令格式）。

</details>

---

## 下一步

下一节 [VM 指令集详解](./02-VM指令集详解.md) 将系统性地介绍 TJS2 VM 的全部指令集——按功能分类、列出完整的操作码表、详解属性访问指令和函数调用指令的参数传递机制。

