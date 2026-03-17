# 02-CMake 核心概念

> **所属模块：** P01-现代CMake与构建工具链
> **前置知识：** [01-构建系统的演进](./01-构建系统的演进.md)
> **预计阅读时间：** 20 分钟

## 本节目标

读完本节后，你将能够：
1. 掌握 `CMakeLists.txt` 的作用与命名规范
2. 理解什么是"源外构建（Out-of-source Build）"及其重要性
3. 理解 CMake 的三个基本阶段：配置、生成和构建

## CMakeLists.txt：项目的剧本

在 CMake 的世界里，一切都始于一个名为 `CMakeLists.txt` 的文件。

- **它是入口**：CMake 会首先寻找并读取这个文件。
- **它是大小写敏感的**：必须叫 `CMakeLists.txt`，不能叫 `cmakelists.txt`。
- **它是声明式的**：你在里面告诉 CMake "我想要一个叫 hello 的可执行文件，它由 main.cpp 编译而来"，而不是写具体的编译命令。

## 源外构建（Out-of-source Build）

这是一个非常重要的习惯，也是 CMake 强烈推荐的做法。

### 什么是"源内构建"（In-source Build）？
如果你在代码所在的目录下直接运行 CMake，它会生成大量的临时文件（如 `CMakeFiles/`, `CMakeCache.txt`, `Makefile` 等），这些文件会和你的 `.cpp`、`.h` 源码混杂在一起。这会污染你的源码目录，让版本控制（如 Git）变得非常痛苦。

### 什么是"源外构建"？
我们将所有的编译产物放在一个独立的文件夹里（通常叫 `build`）。

**操作步骤：**
1. 在项目根目录下创建一个 `build` 文件夹。
2. 进入 `build` 文件夹运行 CMake。
3. 所有的临时文件和最终生成的可执行文件都会留在 `build` 文件夹内。

如果你想重新开始，只需要删除 `build` 文件夹，你的源码依然干干净净。

```bash
# 典型的操作流程
mkdir build
cd build
cmake ..  # ".." 代表告诉 CMake 去上一级目录找 CMakeLists.txt
```

## 生成器（Generator）

还记得我们说过 CMake 不直接编译代码吗？它需要一个**生成器**来生成真正的构建文件。

- **Unix Makefiles**：生成经典的 Makefile（Linux/macOS 默认）。
- **Visual Studio 17 2022**：生成 Windows 下的 `.sln` 和 `.vcxproj`。
- **Xcode**：生成 macOS/iOS 下的 `.xcodeproj`。
- **Ninja**：一个跨平台、极速的构建系统，这也是 KrKr2 项目推荐使用的生成器。

你可以通过 `-G` 参数指定生成器，例如：`cmake -G Ninja ..`

## 目标（Target）：CMake 的灵魂

在现代 CMake（3.0+）中，**一切皆目标（Everything is a target）**。一个 CMake 项目实际上是多个目标的集合。

常见的四种目标：
1. **可执行文件（add_executable）**：你的程序（如 `main.exe` 或 `game`）。
2. **静态库（add_library STATIC）**：编译后的 `.lib` 或 `.a` 文件，最终会打包进可执行文件。
3. **共享库/动态库（add_library SHARED）**：运行时加载的 `.dll` 或 `.so` 文件。
4. **接口库（add_library INTERFACE）**：只有头文件（Header-only）的库。

### 属性（Property）

目标就像是一个对象（Object），它有自己的属性（Property）。
- **编译选项（Compile Options）**：例如开启 C++17 支持。
- **包含目录（Include Directories）**：头文件放在哪里。
- **链接库（Link Libraries）**：依赖哪些库。

**记住一个原则：不要直接修改全局变量，要始终通过目标来修改。**

```cmake
# 给 my_game 目标添加包含目录
target_include_directories(my_game PUBLIC include/)
# 给 my_game 目标链接一个库
target_link_libraries(my_game PRIVATE some_library)
```

## CMake 的三个阶段

理解这三个阶段，能帮你快速定位 90% 的构建错误。

1. **配置阶段（Configure）**：
   CMake 读取 `CMakeLists.txt`，检查系统环境（编译器在哪？系统是什么版本？）。这个阶段会生成一个名为 `CMakeCache.txt` 的文件，记录你的配置选项。
2. **生成阶段（Generate）**：
   CMake 根据配置阶段的结果，生成具体的构建文件（Makefile, .sln, Ninja build files）。
3. **构建阶段（Build）**：
   你运行构建工具（或者让 CMake 帮你运行，如 `cmake --build build`），开始真正的编译工作，生成二进制文件。

### 流程图

```text
  [ CMakeLists.txt ]
          |
          v
  +------------------+
  |    配置阶段      |  (检查环境、变量赋值)
  +------------------+
          |
          v
  +------------------+
  |    生成阶段      |  (产出 Makefile/Ninja 等)
  +------------------+
          |
          v
  +------------------+
  |    构建阶段      |  (编译器开始工作：.cpp -> .o -> .exe)
  +------------------+
```

## 本节小结

- `CMakeLists.txt` 是项目的核心配置文件。
- 始终坚持使用"源外构建"（Out-of-source Build），保持源码整洁。
- 目标（Target）是 CMake 的核心管理单位。
- CMake 构建分为配置、生成和构建三个阶段。

## 练习题与答案

### 题目 1：在项目根目录下创建一个名为 build 的文件夹，并在其中运行 cmake ..，这属于哪种构建方式？为什么要这么做？

<details>
<summary>查看答案</summary>

这属于"源外构建"（Out-of-source Build）。这样做是为了防止 CMake 生成的临时文件（如缓存、中间生成脚本等）污染源码目录。这也有利于版本管理（不需要在 .gitignore 中写一长串临时文件名），并且清理项目非常简单：直接删除整个 build 文件夹即可。

</details>

### 题目 2：如果你在控制台看到 "CMake Error: Could not create named generator..." 错误，这通常发生在 CMake 的哪个阶段？

<details>
<summary>查看答案</summary>

这发生在"配置阶段"（Configure）。这个错误通常意味着你指定的生成器（如 -G "Ninja"）在当前系统中没有安装或不在 PATH 路径中。

</details>

### 题目 3：请列举出至少三种 CMake 目标的类型。

<details>
<summary>查看答案</summary>

1. 可执行文件（add_executable）
2. 静态库（add_library STATIC）
3. 共享库/动态库（add_library SHARED）
4. 接口库/头文件库（add_library INTERFACE）

</details>

## 下一步

→ 继续阅读 [03-第一个CMake项目](./03-第一个CMake项目.md)

