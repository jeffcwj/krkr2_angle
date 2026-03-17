# 02-INTERFACE与PUBLIC与PRIVATE

> **所属模块：** P01-现代CMake与构建工具链
> **前置知识：** 01-add_subdirectory
> **预计阅读时间：** 25 分钟

## 本节目标
在现代 CMake 中，“目标”（Target）是核心。目标不仅仅是生成的文件，它还是属性（Properties）的集合。最重要的属性包括头文件包含路径、预处理器宏、编译选项和链接库。本节我们将彻底讲清这三个关键字如何决定属性在“生产者”与“消费者”之间的传播规则。

## 三个关键字回顾
在第 2 章中，我们曾简要提到过这三个关键字，但在多目录和多模块的项目中，它们的意义才真正体现出来：
1. **PRIVATE**：只给我的“实现”用，别人不需要知道。
2. **INTERFACE**：我不自己用，我是为了“给别人用”才存在的。
3. **PUBLIC**：我自己要用，而且每一个链接我的目标也必须跟着我用。

## PRIVATE：仅目标自身使用
当你为某个目标定义了 `PRIVATE` 属性时：
- **编译时：** 该属性仅在编译该目标自身的源文件时生效。
- **链接时：** 任何链接（依赖）该目标的其他目标都不会获得该属性。

**应用场景：** 一个库在内部逻辑中引用了一个私有头文件目录，或者定义了一个只在库内部使用的调试宏，而库的接口（Header）中并不涉及这些。

## PUBLIC：自身使用 + 传播给依赖者
当你为某个目标定义了 `PUBLIC` 属性时：
- **编译时：** 编译该目标时，属性生效。
- **链接时：** 该属性会“传播”给所有直接链接该目标的其他目标。

**应用场景：** 一个库的头文件（接口）中引用了第三方库的头文件，那么任何使用该库的人也必须能够找到第三方库的头文件。

## INTERFACE：仅传播给依赖者
当你为某个目标定义了 `INTERFACE` 属性时：
- **编译时：** 编译该目标自身时（如果有的话），该属性**不**生效。
- **链接时：** 该属性会“传播”给所有直接链接该目标的其他目标。

**应用场景：** 纯头文件库（Header-only Library），或者作为聚合器将多个底层库打包提供给上层使用（如 KrKr2 的核心库组织方式）。

## 图解传播规则
假设项目中有 A、B、C 三个目标，链接关系为：**C 链接 B，B 链接 A**（C → B → A）。

| 如果 B 链接 A 的方式是 | A 的 PRIVATE 属性会传给 C 吗？ | A 的 PUBLIC 属性会传给 C 吗？ | A 的 INTERFACE 属性会传给 C 吗？ |
| :--- | :---: | :---: | :---: |
| **B link A PRIVATE** | ❌ 否 | ❌ 否 | ❌ 否 |
| **B link A PUBLIC** | ❌ 否 | ✅ 是 | ✅ 是 |
| **B link A INTERFACE** | ❌ 否 | ✅ 是 | ✅ 是 |

> **注意：** 传播是具有传递性的。如果 B PUBLIC 链接了 A，那么 C 链接 B 时，B 会把它从 A 那里获得的公共属性一股脑全部传给 C。

## 常见误区
在使用这三个关键字时，初学者往往会走入以下误区：

### 误区 1：“把所有东西都设为 PUBLIC 不就完了？”
很多开发者为了省事，把所有东西都设为 `PUBLIC`：
- **后果 1：编译变慢。** 无关紧要的头文件包含路径被所有模块读取。
- **后果 2：不必要的重编译。** 如果你修改了某个被所有模块公共包含的头文件，整个项目都得重编。
- **后果 3：依赖污染。** 上层模块意外引用了不应该暴露的内部接口。

**原则：** 除非确实有必要暴露，否则优先使用 `PRIVATE`。

### 误区 2：“INTERFACE 库没有源文件，有什么用？”
这是一个现代 CMake 的高级技巧。在 KrKr2 项目中，你会看到很多模块（如 `krkr2core`）被定义为 `INTERFACE` 库。
- **作用 1：聚合。** 一个复杂的引擎可能有几十个子模块。将它们统一通过一个 `INTERFACE` 目标导出，上层代码只需要链接这一个目标。
- **作用 2：属性传播。** 为一类目标统一定义编译标志或预处理器宏。

## KrKr2 实例分析
让我们来看看 KrKr2 的真实项目结构是如何运用这些概念的：

### 1. krkr2core 是 INTERFACE 库
在 `cpp/core/CMakeLists.txt` 中：
```cmake
add_library(krkr2core INTERFACE)
target_link_libraries(krkr2core INTERFACE 
    tjs2 
    core_base_module 
    core_environ_module 
    # ... 其他 9 个模块
)
```
这意味着，任何链接了 `krkr2core` 的可执行文件（如 Windows 的主程序或 Android 的 `.so`）会自动获得所有子模块的头文件路径和链接库。

### 2. krkr2plugin 是 STATIC 库
在 `cpp/plugins/CMakeLists.txt` 中：
```cmake
add_library(krkr2plugin STATIC ${PLUGIN_SOURCES})
target_link_libraries(krkr2plugin PUBLIC krkr2core)
```
注意这里的 `PUBLIC`：
- `krkr2plugin` 自身（STATIC 库）在编译时需要 `krkr2core` 的头文件。
- 当主程序链接 `krkr2plugin` 时，主程序也需要 `krkr2core` 的头文件来调用插件接口。

## 本节小结
1. `PRIVATE`：仅我所见。
2. `PUBLIC`：我见，你也见。
3. `INTERFACE`：我用不到，但你是我的依赖者，你必须见到。
4. 正确设置可见性是提高大型 C++ 项目构建速度和稳定性的基石。

## 练习题与答案
### 题目 1：场景分析
你正在编写一个名为 `NetLib` 的静态库，它在内部实现中调用了 `OpenSSL` 库，但你的头文件 `NetLib.h` 中完全没有包含 `openssl/*.h`。你应该如何链接 `OpenSSL`？
<details><summary>查看答案</summary>
应该使用 `target_link_libraries(NetLib PRIVATE OpenSSL::SSL)`。
因为 OpenSSL 的头文件和符号仅在 NetLib 的内部 `.cpp` 文件中使用，不需要暴露给 NetLib 的使用者。
</details>

### 题目 2：聚合库设计
创建一个名为 `Engine` 的 `INTERFACE` 库，将 `Physics` 库和 `Graphics` 库包含进去。编写代码片段。
<details><summary>查看答案</summary>
```cmake
add_library(Engine INTERFACE)
target_link_libraries(Engine INTERFACE Physics Graphics)
```
任何链接 `Engine` 的目标都会自动链接 `Physics` 和 `Graphics`。
</details>

## 下一步
→ 继续阅读 [03-实战-构建多模块库.md](./03-实战-构建多模块库.md)

