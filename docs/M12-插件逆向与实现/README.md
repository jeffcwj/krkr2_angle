# M12 — 插件逆向与实现

> **前置知识：** [M09-插件系统与开发](../M09-插件系统与开发/)、[P09-逆向工程入门](../P09-逆向工程入门/)
> **难度：** ★★★★★
> **预计学习时间：** 约 20-25 小时

## 模块简介

KiriKiri/吉里吉里引擎（一款日本 Visual Novel 视觉小说游戏引擎）拥有丰富的插件生态，许多游戏依赖第三方插件实现特殊功能。然而，部分插件仅以编译后的 DLL（Dynamic Link Library，动态链接库——Windows 平台上的共享库格式）形式发布，没有公开源代码。KrKr2 模拟器（本项目，KiriKiri 引擎的跨平台重新实现）要在 Android/Linux/macOS 等非 Windows 平台上运行这些游戏，就必须**逆向分析**这些无源码插件的功能，并用 C++ 重新实现。

本模块将带你从理解 KiriKiri 插件接口出发，系统学习插件逆向的完整方法论，并通过两个真实案例（PackinOne 归档压缩插件和 TextRender 文字渲染插件）进行实战演练，最终将逆向成果集成到 KrKr2 项目中。

### 你将学到

1. KiriKiri 插件的二进制接口（V2Link/V2Unlink 入口函数、iTJSDispatch2 脚本对象接口、tp_stub 桥接头文件）
2. ncbind 注册框架（KrKr2 使用的现代插件注册系统）的内部工作原理
3. 使用反汇编器（Disassembler，将机器码还原为汇编指令的工具）和反编译器（Decompiler，将机器码还原为类 C 伪代码的工具）分析 DLL 插件
4. 从导出表（Export Table，DLL 对外公开的函数列表）识别插件功能入口
5. 推断未知数据结构和还原算法逻辑的实战技巧
6. 将逆向成果编译为静态库并集成到 KrKr2 构建系统

## 术语预览

本模块将涉及以下术语，先有个印象，各章正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| V2Link | V2Link | KiriKiri 插件 DLL 的初始化入口函数，引擎加载插件时首先调用它 |
| V2Unlink | V2Unlink | KiriKiri 插件 DLL 的卸载函数，引擎卸载插件时调用 |
| iTJSDispatch2 | iTJSDispatch2 | TJS2 脚本引擎的核心对象接口，所有脚本对象（函数、类、属性）都通过这个接口访问 |
| tp_stub | tp_stub.h | KiriKiri 官方提供的插件桥接头文件，让插件 DLL 能调用引擎内部的 TJS2 API |
| ncbind | ncbind | KrKr2 使用的 C++ 插件注册框架，用宏将 C++ 类/函数自动注册为 TJS2 脚本对象 |
| PE 格式 | Portable Executable | Windows 可执行文件和 DLL 的二进制格式，包含导出表、导入表等结构 |
| 导出表 | Export Table | PE 文件中记录对外公开函数名称和地址的数据结构 |
| IDA/Ghidra | IDA Pro / Ghidra | 业界主流的反汇编/反编译工具，用于分析二进制文件 |
| 符号恢复 | Symbol Recovery | 从无调试信息的二进制文件中推断函数名、类名、变量名的过程 |
| 静态链接 | Static Linking | 将库代码直接编入最终可执行文件，不需要运行时加载 DLL |

## 章节目录

| 章 | 标题 | 节数 | 主要内容 |
|----|------|------|----------|
| 01 | [KiriKiri 插件接口分析](./01-KiriKiri插件接口分析/) | 3 节 | V2Link/V2Unlink 入口机制、iTJSDispatch2 接口与 tp_stub 桥接、ncbind 注册框架内部原理 |
| 02 | [无源码插件清单](./02-无源码插件清单/) | 2 节 | KrKr2 插件全景图（已实现/有源码/无源码分类）、无源码插件功能分析与逆向优先级排序 |
| 03 | [逆向方法论](./03-逆向方法论/) | 3 节 | DLL 导出表分析与接口识别、数据结构推断与行为还原技术、逆向结果验证与自动化测试策略 |
| 04 | [实战：逆向 PackinOne](./04-实战-逆向PackinOne/) | 2 节 | PackinOne 插件功能分析与导出表解读、压缩/归档算法还原与 C++ 重新实现 |
| 05 | [实战：逆向 TextRender](./05-实战-逆向TextRender/) | 2 节 | TextRender 文字渲染接口分析与字体处理逻辑、渲染逻辑还原与跨平台（FreeType）适配 |
| 06 | [逆向成果集成](./06-逆向成果集成/) | 2 节 | 逆向代码编译为静态库并注册到 krkr2plugin、测试编写与插件清单维护流程 |

## 学习路线

```
M09-插件系统与开发          P09-逆向工程入门
     │                          │
     └──────────┬───────────────┘
                ▼
   01-KiriKiri插件接口分析
        （理解插件二进制接口）
                │
                ▼
      02-无源码插件清单
        （明确逆向目标）
                │
                ▼
        03-逆向方法论
        （掌握系统方法）
                │
         ┌──────┴──────┐
         ▼             ▼
  04-实战PackinOne  05-实战TextRender
   （归档/压缩）     （文字渲染）
         └──────┬──────┘
                ▼
      06-逆向成果集成
   （编译、注册、测试、发布）
                │
                ▼
      M13-UI框架替换实战
```

## 实践环境准备

学习本模块前，请确保你已具备以下工具和知识：

### 必备工具

| 工具 | 用途 | 获取方式 |
|------|------|----------|
| Ghidra | 免费反编译器，分析 DLL 二进制 | [ghidra-sre.org](https://ghidra-sre.org/) |
| IDA Free | 免费版反汇编器（可选，Ghidra 已够用） | [hex-rays.com](https://hex-rays.com/ida-free/) |
| PE Explorer / CFF Explorer | 查看 PE 文件结构（导出表/导入表） | CFF Explorer 免费 |
| x64dbg / WinDbg | 动态调试器，运行时观察插件行为 | [x64dbg.com](https://x64dbg.com/) |
| CMake + Ninja + MSVC/GCC | C++ 编译工具链（KrKr2 构建环境） | 参考 [M01 环境搭建](../M01-项目导览与环境搭建/) |
| Python 3 | 编写辅助分析脚本 | [python.org](https://www.org/) |

### 必备前置知识

- **M09 插件系统**：理解 KrKr2 的 ncbind/simplebinder 插件注册机制
- **P09 逆向工程**：会使用 Ghidra/IDA 进行基本的反汇编分析
- **C++ 基础**：熟悉指针、虚函数表（vtable）、COM 风格接口
- **TJS2 基础**（来自 M07）：理解 TJS2 对象模型和函数调用约定

## 与项目代码的对应关系

| 本模块主题 | 项目代码位置 | 说明 |
|-----------|-------------|------|
| 插件接口 | `cpp/core/plugin/` | ncbind 框架、tp_stub 桥接 |
| 已实现插件 | `cpp/plugins/` | psdfile、psbfile、motionplayer、layerExDraw 等 |
| 插件注册入口 | `cpp/plugins/CMakeLists.txt` | 静态库 krkr2plugin 的构建定义 |
| TJS2 对象接口 | `cpp/core/tjs2/tjsTypes.h` | iTJSDispatch2 基类定义 |
| 插件清单 | `doc/krkr2_plugins.md` | 完整的 DLL 插件列表（含源码/无源码标记） |

## 下一步

完成本模块后，继续学习 [M13-UI框架替换实战](../M13-UI框架替换实战/)，将 P12 中学到的现代跨平台 UI 知识应用到 KrKr2 项目中。
