# 02-find_package 用法：在 CMake 中寻找依赖库

在配置好 vcpkg 工具链后，你的 CMake 项目现在已经能够“看见” vcpkg 安装的库了。接下来，我们需要通过 CMake 的标准命令 `find_package` 将这些库引入到你的编译流程中。

---

## find_package 的两种搜索模式

CMake 提供了两种寻找库的模式。了解它们的区别是解决编译错误的关键。

### 1. Module 模式（模块模式）

CMake 会寻找名为 `Find<PackageName>.cmake` 的文件。这些文件通常由 CMake 官方提供，或者是第三方库开发者手动编写并放在项目的 `cmake/` 目录下。

- **特点**：不需要库本身支持 CMake。
- **例子**：`find_package(FFmpeg REQUIRED)` 会尝试寻找 `FindFFMPEG.cmake`。

### 2. Config 模式（配置模式）

CMake 会寻找名为 `<PackageName>Config.cmake` 或 `<lowercasePackageName>-config.cmake` 的文件。这些文件通常是由库在安装时自动生成的。

- **特点**：最符合现代 CMake 标准，推荐使用。
- **例子**：`find_package(spdlog CONFIG REQUIRED)`。

---

## vcpkg 如何支持 find_package？

当你设置了 `VCPKG_TOOLCHAIN_FILE` 后，vcpkg 会执行以下操作：
1. **注入 CMAKE_PREFIX_PATH**：它将所有 vcpkg 安装库的路径（如 `vcpkg_installed/x64-windows/share`）添加到搜索路径中。
2. **自动路径重定向**：如果 vcpkg 发现某个库已经安装，它会确保 CMake 优先搜索 vcpkg 目录而不是系统目录。

### CONFIG 关键字的含义

在 vcpkg 中，大多数现代库都推荐使用 `CONFIG` 关键字。这告诉 CMake：“不要去用那种老旧的 Find 模块，直接去找库生成的 Config 文件”。这能极大地减少“找错库”或“链接失败”的概率。

---

## 标准用法：find_package + target_link_libraries

在现代 CMake 中，我们不再直接操作头文件路径和链接库名称，而是操作 **Target（目标）**。

```cmake
# 1. 寻找库
find_package(fmt CONFIG REQUIRED)

# 2. 定义你自己的目标（例如一个可执行程序）
add_executable(my_app main.cpp)

# 3. 链接库
# 这里的 fmt::fmt 是 fmt 库定义的导出目标（Exported Target）
# 它会自动帮你配置：包含路径（include）、编译定义、链接库文件
target_link_libraries(my_app PRIVATE fmt::fmt)
```

---

## KrKr2 中的真实用法示例

在 KrKr2 项目中，我们大量使用了这种模式。以下是几个典型例子：

### 1. 现代库 (spdlog / Catch2)

```cmake
# 用于日志记录
find_package(spdlog CONFIG REQUIRED)
target_link_libraries(krkr2core INTERFACE spdlog::spdlog)

# 用于单元测试
find_package(Catch2 CONFIG REQUIRED)
target_link_libraries(unit-tests PRIVATE Catch2::Catch2WithMain)
```

### 2. 传统库或自定义模块 (FFmpeg)

FFmpeg 本身并不提供 CMake Config 文件。为了支持它，KrKr2 在 `cmake/` 目录下提供了一个自定义的 `FindFFMPEG.cmake`。

```cmake
# 这种情况下不加 CONFIG 关键字
find_package(FFmpeg REQUIRED)
# 链接时通常使用 Find 模块里定义的变量
target_link_libraries(krkr2core INTERFACE ${FFMPEG_LIBRARIES})
```

### 3. 多组件库 (Cocos2d-x)

有些库非常庞大，被拆分成了多个组件（Components）。我们可以通过 `COMPONENTS` 关键字按需引入：

```cmake
find_package(cocos2dx CONFIG REQUIRED COMPONENTS core network ui)
target_link_libraries(krkr2core INTERFACE cocos2d-x::core cocos2d-x::ui)
```

---

## 常用参数：QUIET 与 REQUIRED

- **REQUIRED**：如果找不到库，CMake 配置阶段会直接报错并停止。这是最常用的选项，因为缺少依赖库通常意味着无法编译。
- **QUIET**：如果找不到库，CMake 不会打印任何警告或错误。只有当你需要根据某个库是否存在来决定某些编译逻辑时才使用它。

---

## 排查 find_package 失败的常见方法

如果你发现 `find_package` 报错找不到库，请检查以下几点：

1. **检查 Triplet**：确认你安装库的 Triplet 与 CMake 配置时指定的 Triplet 是否完全一致。例如，你安装了 `x64-windows`，但 CMake 却在寻找 `x64-windows-static`。
2. **检查 vcpkg 导出代码**：有些库在安装后会打印出对应的 `find_package` 代码。你可以通过 `vcpkg search <库名>` 或 `vcpkg list` 再次查看。
3. **查看 CMake 搜索日志**：
   在命令行中运行：
   ```bash
   cmake -B build --debug-find
   ```
   这会详细列出 CMake 寻找每一个库时搜索过的所有路径。

---

## 练习题与答案

### 题目 1：在 `find_package` 中使用 `CONFIG` 关键字有什么具体作用？

<details>
<summary>查看答案</summary>

1. **精确查找**：它强制 CMake 寻找库自带的 `<PackageName>Config.cmake` 文件，而不是使用通用的 `Find<PackageName>.cmake` 脚本。
2. **避免歧义**：现代库通常会通过 Config 文件导出其 Target（如 `spdlog::spdlog`），这种 Target 包含了完整的依赖关系和包含路径。
3. **性能提升**：减少了 CMake 在系统路径下搜索各种 Find 脚本的时间。

</details>

### 题目 2：假设你正在为 KrKr2 编写一个新的单元测试，需要引入 `Catch2` 库。请写出在 `CMakeLists.txt` 中引入并链接该库的完整代码。

<details>
<summary>查看答案</summary>

```cmake
# 1. 引入 Catch2
find_package(Catch2 CONFIG REQUIRED)

# 2. 定义测试目标
add_executable(my_new_test test_main.cpp test_cases.cpp)

# 3. 链接 Catch2 (使用带 main 函数的版本，这样你就不需要自己写 int main)
target_link_libraries(my_new_test PRIVATE Catch2::Catch2WithMain)
```

</details>

### 题目 3：如果你在一个没有 vcpkg 的环境（如纯净的 Linux 系统）中尝试运行 `find_package(spdlog CONFIG REQUIRED)`，通常会发生什么？如何修复？

<details>
<summary>查看答案</summary>

**现象：** CMake 会报错提示找不到 `spdlogConfig.cmake` 或 `spdlog-config.cmake`。

**原因：** 虽然 Linux 的包管理器（如 `apt`）可能安装了 `libspdlog-dev`，但如果它没有提供这些 Config 文件，或者这些文件不在 CMake 的默认搜索路径中，CMake 就会失败。

**修复方法：**
1. **安装 vcpkg** 并在配置时指定 `CMAKE_TOOLCHAIN_FILE`。
2. **通过 apt 安装时手动指定路径**：设置 `CMAKE_PREFIX_PATH` 指向库的安装目录（例如 `/usr/lib/cmake/spdlog`）。
3. **回退到非 Config 模式**：如果系统包管理器只提供了 `Findspdlog.cmake`（非常罕见），则去掉 `CONFIG` 关键字。

</details>

