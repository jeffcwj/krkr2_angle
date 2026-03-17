## CMAKE_CURRENT_SOURCE_DIR vs CMAKE_SOURCE_DIR
在多目录项目中，CMake 提供了几个重要的路径变量，理解它们的区别至关重要：
- **CMAKE_SOURCE_DIR**：项目根目录（包含最顶层 `CMakeLists.txt` 的目录）。无论你在哪个子目录，这个变量的值永远不变。
- **CMAKE_CURRENT_SOURCE_DIR**：当前正在处理的 `CMakeLists.txt` 所在的目录。
- **CMAKE_BINARY_DIR**：编译输出的根目录（通常是 `build/`）。
- **CMAKE_CURRENT_BINARY_DIR**：当前子目录对应的编译输出目录。

**建议：** 在子目录中引用文件时，优先使用 `CMAKE_CURRENT_SOURCE_DIR`，这样可以保证模块的移动不会破坏路径。

## 完整示例
让我们亲手创建一个包含两个子目录的项目：一个生成静态库 `math_utils`，另一个生成可执行文件 `main_app`。

### 目录结构
```text
my_project/
├── CMakeLists.txt
├── math/
│   ├── CMakeLists.txt
│   ├── adder.cpp
│   └── adder.h
└── app/
    ├── CMakeLists.txt
    └── main.cpp
```

### 1. 根目录 `CMakeLists.txt`
```cmake
cmake_minimum_required(VERSION 3.10)
project(MultiDirExample)

# 添加子目录
add_subdirectory(math)
add_subdirectory(app)
```

### 2. `math/adder.h`
```cpp
#pragma once
int add(int a, int b);
```

### 3. `math/adder.cpp`
```cpp
#include "adder.h"
int add(int a, int b) {
    return a + b;
}
```

### 4. `math/CMakeLists.txt`
```cmake
# 创建一个名为 math_lib 的静态库
add_library(math_lib STATIC adder.cpp)

# 指定头文件路径，方便别人使用
target_include_directories(math_lib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
```

### 5. `app/main.cpp`
```cpp
#include <iostream>
#include "adder.h"

int main() {
    std::cout << "1 + 2 = " << add(1, 2) << std::endl;
    return 0;
}
```

### 6. `app/CMakeLists.txt`
```cmake
add_executable(main_app main.cpp)

# 链接 math_lib 库
target_link_libraries(main_app PRIVATE math_lib)
```

### 构建与运行
在根目录下运行：
```bash
mkdir build
cd build
cmake ..
cmake --build .
# Windows: .\app\Debug\main_app.exe
# Linux/macOS: ./app/main_app
```

## 对照 KrKr2
在 KrKr2 的源代码中，根目录的 `CMakeLists.txt` 正是通过这种方式组织庞大的代码库的。你可以看到类似下面的结构：

```cmake
# 引入外部依赖库
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/cpp/external)

# 引入核心引擎模块 (cpp/core/)
# KRKR2CORE_PATH 变量通常在根目录定义
add_subdirectory(${KRKR2CORE_PATH})

# 引入插件模块 (cpp/plugins/)
add_subdirectory(${KRKR2PLUGIN_PATH})
```

这种结构允许核心开发者专注于 `cpp/core/` 下的代码，而插件开发者只需要关注 `cpp/plugins/`，互不干扰，逻辑清晰。

## 本节小结
1. `add_subdirectory()` 让项目结构化，易于维护。
2. 每个子目录必须有自己的 `CMakeLists.txt`。
3. 子目录继承父目录变量，但修改需用 `PARENT_SCOPE` 影响父目录。
4. 始终清楚 `CMAKE_CURRENT_SOURCE_DIR` 指向何处。

## 练习题与答案
### 题目 1：路径辨析
如果在 `/home/user/project/src/utils/CMakeLists.txt` 中使用了 `add_subdirectory`，此时 `CMAKE_SOURCE_DIR` 和 `CMAKE_CURRENT_SOURCE_DIR` 分别指向哪里？
<details><summary>查看答案</summary>
`CMAKE_SOURCE_DIR` 指向项目根目录 `/home/user/project/`。
`CMAKE_CURRENT_SOURCE_DIR` 指向当前目录 `/home/user/project/src/utils/`。
</details>

### 题目 2：变量修改
父目录定义了 `set(ENABLE_LOG ON)`。子目录执行了 `set(ENABLE_LOG OFF)`。
1. 子目录后续代码中 `ENABLE_LOG` 的值是多少？
2. 父目录在 `add_subdirectory` 之后的代码中 `ENABLE_LOG` 的值是多少？
3. 如何让子目录的修改影响到父目录？
<details><summary>查看答案</summary>
1. `OFF`。
2. `ON`（子目录的普通 set 不影响父目录作用域）。
3. 使用 `set(ENABLE_LOG OFF PARENT_SCOPE)`。
</details>

## 下一步
→ 继续阅读 [02-INTERFACE与PUBLIC与PRIVATE.md](./02-INTERFACE与PUBLIC与PRIVATE.md)
