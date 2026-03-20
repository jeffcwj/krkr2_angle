# CMake 注册与测试

> **所属模块：** M09-插件系统与开发
> **前置知识：** [需求分析与设计](./01-需求分析与设计.md)、[C++实现与TJS绑定](./02-C++实现与TJS绑定.md)、[P01-现代CMake与构建工具链](../../P01-现代CMake与构建工具链/README.md)、[P02-vcpkg包管理](../../P02-vcpkg包管理/README.md)
> **预计阅读时间：** 40 分钟

## 本节目标

读完本节后，你将能够：
1. 为 textMetrics 插件编写完整的 CMakeLists.txt，支持跨平台条件编译
2. 将新插件集成到 `cpp/plugins/CMakeLists.txt` 父级构建系统中
3. 在 vcpkg.json 中正确声明平台相关依赖（FreeType、Fontconfig）
4. 使用 Catch2 v3 为插件编写单元测试，遵循项目现有测试模式
5. 在 Windows、Linux、macOS、Android 四个平台上验证构建结果
6. 排查插件注册失败、链接错误、字体查找失败等常见问题

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 静态库目标 | STATIC Library Target | CMake 中用 `add_library(name STATIC)` 创建的编译单元，生成 `.a`（Linux/macOS）或 `.lib`（Windows）文件，最终链接进主程序 |
| 条件编译 | Conditional Compilation | 通过 `if(WINDOWS)` 等 CMake 判断或 `#ifdef` 预处理器指令，让同一项目在不同平台编译不同源文件 |
| vcpkg 清单模式 | vcpkg Manifest Mode | 通过项目根目录的 `vcpkg.json` 文件声明依赖，CMake 配置时自动安装——KrKr2 使用的依赖管理方式 |
| Catch2 v3 | Catch2 v3 | C++ 单元测试框架的第三代版本，支持 `TEST_CASE` / `SECTION` / `REQUIRE` 宏进行断言式测试 |
| 测试发现 | Test Discovery | CMake 的 `catch_discover_tests()` 命令，自动扫描 Catch2 测试用例并注册到 CTest，无需手动维护测试列表 |
| 平台三元组 | Platform Triplet | vcpkg 用 `x64-windows` / `x64-linux` / `arm64-android` 等字符串标识目标平台的 CPU 架构 + 操作系统组合 |

---

## 一、textMetrics 插件的 CMakeLists.txt

在前两节中，我们完成了 TextMetrics 插件的全部 C++ 代码：核心类 `TextMetrics`、抽象接口 `TextMetricsImpl`、三个平台实现（Windows GDI / FreeType / Core Text）、以及 ncbind 注册入口 `main.cpp`。现在需要编写 CMake 构建脚本，让这些代码能够在四个平台上正确编译和链接。

### 1.1 目录结构回顾

在开始编写 CMakeLists.txt 之前，先回顾 textMetrics 插件的文件布局：

```
cpp/plugins/textMetrics/
├── CMakeLists.txt             # 本节要编写的构建脚本
├── main.cpp                   # ncbind 注册入口（NCB_REGISTER_CLASS）
├── TextMetrics.h              # 核心类声明
├── TextMetrics.cpp            # 核心类实现（平台无关逻辑）
├── TextMetricsImpl.h          # 抽象接口 + 工厂方法（含条件 #include）
├── TextMetricsImpl_Win.cpp    # Windows GDI 实现
├── TextMetricsImpl_FT.cpp     # FreeType 实现（Linux / Android）
└── TextMetricsImpl_CT.cpp     # Core Text 实现（macOS）
```

### 1.2 完整 CMakeLists.txt

以下是 `cpp/plugins/textMetrics/CMakeLists.txt` 的完整内容。我们采用与 layerExDraw 类似的**条件编译**（Conditional Compilation，根据目标平台选择不同源文件编译）策略，但使用更简洁的"单文件切换"模式（一个目录内用 `if` 选择不同 `.cpp`），而非 layerExDraw 的"双目录"模式（`windows/` vs `general/` 两个子目录）：

```cmake
# cpp/plugins/textMetrics/CMakeLists.txt
# TextMetrics 插件 — 字体度量计算
cmake_minimum_required(VERSION 3.19)
project(textMetrics LANGUAGES CXX)

# 创建静态库目标
# STATIC 表示生成 .a（Linux/macOS）或 .lib（Windows）
# 最终会链接进 krkr2plugin → krkr2 主程序
add_library(${PROJECT_NAME} STATIC)

# === 平台无关源文件 ===
# main.cpp: ncbind 注册入口
# TextMetrics.cpp: 核心逻辑（调用 impl 的虚函数）
target_sources(${PROJECT_NAME} PUBLIC
    main.cpp
    TextMetrics.cpp
)

# === 平台相关源文件 — 条件编译 ===
# KrKr2 使用自定义 CMake 变量: WINDOWS / LINUX / MACOS / ANDROID
# 注意：不是 CMake 内置的 CMAKE_SYSTEM_NAME，而是项目在
# 根 CMakeLists.txt 中通过 set(WINDOWS TRUE) 等方式定义的
if(WINDOWS)
    # Windows: 使用 GDI API（系统自带，无需额外依赖）
    target_sources(${PROJECT_NAME} PRIVATE
        TextMetricsImpl_Win.cpp
    )
    # GDI 需要链接 gdi32.lib
    target_link_libraries(${PROJECT_NAME} PRIVATE gdi32)
elseif(MACOS)
    # macOS: 使用 Core Text 框架（系统自带）
    target_sources(${PROJECT_NAME} PRIVATE
        TextMetricsImpl_CT.cpp
    )
    # 链接 CoreText 和 CoreFoundation 框架
    # find_library 在 macOS 上查找 .framework
    find_library(CORE_TEXT CoreText REQUIRED)
    find_library(CORE_FOUNDATION CoreFoundation REQUIRED)
    target_link_libraries(${PROJECT_NAME} PRIVATE
        ${CORE_TEXT}
        ${CORE_FOUNDATION}
    )
else()
    # Linux 和 Android: 使用 FreeType（通过 vcpkg 安装）
    target_sources(${PROJECT_NAME} PRIVATE
        TextMetricsImpl_FT.cpp
    )
    # find_package 从 vcpkg 查找 FreeType
    find_package(Freetype REQUIRED)
    target_link_libraries(${PROJECT_NAME} PRIVATE
        Freetype::Freetype
    )
    # Linux 额外需要 Fontconfig（字体查找库）
    # Android 不需要——直接扫描 /system/fonts/ 目录
    if(LINUX)
        find_package(Fontconfig REQUIRED)
        target_link_libraries(${PROJECT_NAME} PRIVATE
            Fontconfig::Fontconfig
        )
    endif()
endif()

# === 链接父级插件库 ===
# 所有子目录插件都必须链接 krkr2plugin
# 这提供了 ncbind 宏所需的头文件和符号
target_link_libraries(${PROJECT_NAME} PRIVATE krkr2plugin)
```

### 1.3 逐行解析关键设计决策

**为什么用 `STATIC` 而不是 `SHARED`？** KrKr2 的所有插件都以静态库形式链接进主程序——不是动态加载的 `.dll` / `.so`。查看 `cpp/plugins/CMakeLists.txt` 第 4 行：`add_library(${PROJECT_NAME} STATIC)` 就是父级插件库的声明，所有子插件也必须是 `STATIC`。

**`PUBLIC` vs `PRIVATE` 的 `target_sources`：** 平台无关的 `main.cpp` 和 `TextMetrics.cpp` 用 `PUBLIC`，因为它们声明的符号需要被父级库 `krkr2plugin` 看到（ncbind 注册宏会生成全局构造函数）。平台相关的 `.cpp` 用 `PRIVATE`，因为它们只实现虚函数，不暴露新符号。

**`if(WINDOWS)` vs `if(WIN32)`：** KrKr2 项目使用自定义的平台变量 `WINDOWS` / `LINUX` / `MACOS` / `ANDROID`（参见 AGENTS.md 第 100 行），而**不是** CMake 内置的 `WIN32` / `UNIX` / `APPLE`。使用错误的变量会导致条件永远为假，编译时报"找不到 TextMetricsImpl 的实现"链接错误。

**macOS 的 `find_library`：** Core Text 和 CoreFoundation 是 macOS 系统框架（Framework），不通过 vcpkg 安装。CMake 的 `find_library` 在 macOS 上会自动搜索 `/System/Library/Frameworks/` 和 Xcode SDK 中的框架路径。

### 1.4 两种条件编译模式对比

上一章分析 layerExDraw 时，我们了解了它的"双目录"条件编译模式。textMetrics 采用了更简洁的"单目录"模式。两者各有适用场景：

| 对比维度 | 双目录模式（layerExDraw） | 单目录模式（textMetrics） |
|----------|--------------------------|--------------------------|
| 目录结构 | `windows/main.cpp` + `general/main.cpp` | 同一目录下 `_Win.cpp` / `_FT.cpp` / `_CT.cpp` |
| CMake 写法 | `if(WINDOWS)` 选择不同子目录的源文件 | `if(WINDOWS)` 选择同目录下不同后缀的源文件 |
| 头文件共享 | 两个目录各自维护头文件，可能有重复 | 统一头文件 `TextMetricsImpl.h`，所有平台共用 |
| 适用场景 | 平台间实现差异极大，连接口都不同 | 平台间共享统一抽象接口，仅实现不同 |
| 代码量 | 较多（每个目录完整独立） | 较少（共用接口 + 各平台实现） |
| 实际例子 | layerExDraw（GDI+ vs libgdiplus 接口完全不同） | textMetrics（统一 `TextMetricsImpl` 虚接口） |

**决策指南：** 如果你的插件在不同平台上使用完全不同的第三方库且 API 不兼容（如 GDI+ vs libgdiplus 的头文件和类型完全不同），选双目录模式。如果所有平台可以共用一个抽象接口（如我们的 `measureWidth` / `measureHeight` 虚函数），选单目录模式。

---

## 二、集成到父级构建系统

插件自身的 CMakeLists.txt 写好后，还需要在父级 `cpp/plugins/CMakeLists.txt` 中注册，否则 CMake 不会编译它。

### 2.1 修改 cpp/plugins/CMakeLists.txt

需要添加两处：`add_subdirectory` 和 `target_link_libraries`。以下展示完整的修改后文件（高亮新增行）：

```cmake
# cpp/plugins/CMakeLists.txt（节选关键部分）
# ... 前面的 target_sources、target_include_directories 保持不变 ...

# === 子目录插件 ===
add_subdirectory(psdfile)
add_subdirectory(psbfile)
add_subdirectory(layerex_draw)
add_subdirectory(motionplayer)
add_subdirectory(fstat)
add_subdirectory(textMetrics)    # 新增：textMetrics 插件

# === 链接所有子目录插件到 krkr2plugin ===
target_link_libraries(${PROJECT_NAME} PUBLIC
        psdfile
        psbfile
        layerExDraw
        motionplayer
        fstat
        textMetrics              # 新增：链接 textMetrics
)
```

### 2.2 注册顺序说明

`add_subdirectory` 的顺序**不影响最终功能**——CMake 会自动解析依赖图。但为了代码可读性，建议按字母顺序或功能分组排列。项目现有排列按创建时间，我们将 textMetrics 添加到末尾即可。

`target_link_libraries` 中的目标名必须与子目录 CMakeLists.txt 中的 `project()` 名称一致。我们的子目录用 `project(textMetrics)`，所以这里也写 `textMetrics`。如果名称不匹配，CMake 配置阶段就会报错：

```
CMake Error at cpp/plugins/CMakeLists.txt:XX (target_link_libraries):
  Target "text_metrics" was not created in this directory and is not an
  IMPORTED target.
```

### 2.3 链接传播机制

为什么子插件内部已经 `target_link_libraries(textMetrics PRIVATE krkr2plugin)` 了，父级还要再链接一次？这涉及 CMake 的**链接传播**（Link Propagation）机制：

```
krkr2（主程序）
  └── 链接 krkr2plugin（STATIC 库）
        ├── 链接 psdfile（STATIC 库）
        ├── 链接 layerExDraw（STATIC 库）
        ├── 链接 fstat（STATIC 库）
        └── 链接 textMetrics（STATIC 库）  ← 必须在此声明
              ├── 链接 krkr2plugin（获取头文件）
              ├── 链接 gdi32（Windows）
              ├── 链接 Freetype（Linux/Android）
              └── 链接 Fontconfig（Linux）
```

- **子插件 → krkr2plugin**（`PRIVATE`）：子插件编译时需要 ncbind 头文件，但不需要把这个依赖传播给上层
- **krkr2plugin → 子插件**（`PUBLIC`）：父级库把子插件的目标文件（`.o`）打包进自己的 `.a` / `.lib`，最终主程序才能找到 ncbind 注册的全局构造函数

如果忘记在父级添加 `target_link_libraries`，编译不会报错（子目录照常编译），但**链接阶段**主程序会找不到 `TextMetrics` 的注册符号，导致插件"静默消失"——TJS 脚本调用 `new TextMetrics()` 时抛出"class not found"异常。

---

## 三、vcpkg 依赖声明

textMetrics 在 Linux/Android 平台使用 FreeType 和 Fontconfig，这两个库需要在 `vcpkg.json` 中声明，否则 `find_package` 会失败。

### 3.1 修改 vcpkg.json

在 `vcpkg.json` 的 `dependencies` 数组中添加：

```json
{
  "dependencies": [
    // ... 现有依赖保持不变 ...
    {
      "name": "freetype",
      "platform": "linux | android"
    },
    {
      "name": "fontconfig",
      "platform": "linux"
    }
    // ... 其余依赖 ...
  ]
}
```

### 3.2 平台限定符详解

vcpkg 的 `platform` 字段使用**平台表达式**（Platform Expression），语法如下：

| 表达式 | 含义 | 匹配的三元组（Triplet，vcpkg 标识目标平台的字符串） |
|--------|------|------|
| `linux` | 仅 Linux | `x64-linux` |
| `android` | 仅 Android | `arm64-android`, `x86_64-android` |
| `linux \| android` | Linux 或 Android | 以上全部 |
| `!windows` | 非 Windows（含 Linux、macOS、Android） | 除 `x64-windows` 外的全部 |
| `windows` | 仅 Windows | `x64-windows` |
| `osx` | 仅 macOS | `arm64-osx`, `x64-osx` |

**为什么 FreeType 只在 `linux | android`？** 因为：
- **Windows** 使用系统自带的 GDI API，不需要 FreeType
- **macOS** 使用系统自带的 Core Text 框架，不需要 FreeType
- **Linux** 没有系统级字体渲染 API，FreeType 是标准选择
- **Android** NDK 不暴露 Skia/HarfBuzz 等系统字体库，FreeType 是唯一可靠方案

**为什么 Fontconfig 只在 `linux`？** Fontconfig（字体配置和查找库，通过字体名称找到 `.ttf` 文件路径）仅在 Linux 桌面环境有意义。Android 的字体固定在 `/system/fonts/` 目录，直接枚举文件名即可，不需要 Fontconfig。

### 3.3 检查现有依赖是否冲突

添加新依赖前，检查 `vcpkg.json` 中是否已有相同或冲突的包：

```bash
# Windows — 在项目根目录执行
findstr "freetype" vcpkg.json
findstr "fontconfig" vcpkg.json

# Linux/macOS
grep "freetype" vcpkg.json
grep "fontconfig" vcpkg.json
```

如果返回空结果，说明项目中尚未使用这两个包，可以安全添加。如果已存在但平台限定不同，需要合并平台表达式（用 `|` 连接）。

> **注意：** KrKr2 项目已经在 vcpkg.json 中声明了 `libgdiplus`（`"platform": "!windows"`），这是 layerExDraw 的依赖。FreeType 和 Fontconfig 是**独立的**包，不会与 libgdiplus 冲突。

### 3.4 vcpkg 安装验证

修改 `vcpkg.json` 后，下次运行 CMake 配置时 vcpkg 会自动安装新依赖。可以手动触发验证：

```bash
# Windows
cmake --preset="Windows Debug Config"
# 观察输出中是否有：
# -- Installing vcpkg packages...
# 不应该出现 freetype/fontconfig（Windows 不需要）

# Linux
cmake --preset="Linux Debug Config"
# 应该看到：
# Installing 2/2 freetype:x64-linux...
# Installing 2/2 fontconfig:x64-linux...

# macOS
cmake --preset="MacOS Debug Config"
# 不应该出现 freetype/fontconfig（macOS 不需要）

# Android
# 通过 Gradle 触发
./platforms/android/gradlew -p ./platforms/android assembleDebug
# 应该看到 freetype 安装，但不安装 fontconfig
```

如果安装失败，检查 `VCPKG_ROOT` 环境变量是否正确设置，以及网络是否可达。

---

## 四、Catch2 单元测试编写

KrKr2 使用 Catch2 v3 作为测试框架。项目中已有的插件测试位于 `tests/unit-tests/plugins/`——目前只有 psbfile 的测试（`psbfile-dll.cpp`）。我们要为 textMetrics 添加新的测试文件。

### 4.1 项目测试架构回顾

```
tests/
├── CMakeLists.txt         # 顶层：find_package(Catch2 3)，设置 TEST_FILES_PATH
├── test_config.h.in       # 模板文件，生成 test_config.h（包含测试资源路径）
├── test_files/            # 测试用的固定文件（字体文件等）
└── unit-tests/
    ├── core/
    │   ├── movie/         # 视频模块测试
    │   ├── tjs2/          # TJS2 引擎测试
    │   └── visual/        # 渲染模块测试
    └── plugins/           # 插件测试（我们在此添加）
        ├── CMakeLists.txt # 插件测试的 CMake 配置
        ├── main.cpp       # 自定义入口（初始化 spdlog）
        └── psbfile-dll.cpp # 现有的 psbfile 测试
```

Catch2 v3 与 v2 的关键区别：v3 需要单独的 `main.cpp`（v2 可以用 `Catch2::Catch2WithMain` 自动生成）。KrKr2 项目使用**自定义 main()**，因为需要在测试运行前初始化 spdlog 日志系统（创建 "core"、"tjs2"、"plugin" 三个 logger）。

### 4.2 测试文件：textMetrics-test.cpp

以下是完整的测试文件。注意：由于 TextMetrics 依赖 ncbind 框架和 TJS 运行时，直接单元测试完整的插件注册流程比较困难（需要启动 TJS VM）。我们的策略是**测试平台实现层**（TextMetricsImpl），跳过 ncbind 注册层：

```cpp
// tests/unit-tests/plugins/textMetrics-test.cpp
// TextMetrics 插件单元测试

#include <catch2/catch_test_macros.hpp>
#include <catch2/matchers/catch_matchers_floating_point.hpp>

#include "test_config.h"

// 直接包含实现头文件，绕过 ncbind 依赖
#include "textMetrics/TextMetricsImpl.h"

#include <memory>
#include <string>

// 工厂方法创建平台实现
// 在 TextMetricsImpl.h 中声明：
// std::unique_ptr<TextMetricsImpl> createTextMetricsImpl(
//     const std::string& fontName, int fontSize, int fontStyle);

TEST_CASE("TextMetricsImpl 构造与基本属性",
          "[textMetrics]") {
    // 使用系统默认字体
    // Windows: "MS Gothic", Linux: "Noto Sans CJK SC"
    // macOS: "Hiragino Sans", Android: "Noto Sans CJK"
    // 此处用空字符串触发 fallback 逻辑
    auto impl = createTextMetricsImpl("", 16, 0);
    REQUIRE(impl != nullptr);

    SECTION("字体大小正确") {
        // 工厂方法应保留请求的字体大小
        REQUIRE(impl->getFontSize() == 16);
    }

    SECTION("字体风格为 NORMAL") {
        REQUIRE(impl->getFontStyle() == 0);
    }
}

TEST_CASE("TextMetricsImpl 宽度测量",
          "[textMetrics]") {
    auto impl = createTextMetricsImpl("", 24, 0);
    REQUIRE(impl != nullptr);

    SECTION("空字符串宽度为 0") {
        int width = impl->measureWidth("");
        REQUIRE(width == 0);
    }

    SECTION("ASCII 字符串宽度 > 0") {
        int width = impl->measureWidth("Hello");
        // 24px 字号下 "Hello" 至少有 30 像素宽
        REQUIRE(width > 30);
        // 不应超过 200 像素（合理性检查）
        REQUIRE(width < 200);
    }

    SECTION("CJK 字符宽度约为字号") {
        // 中文字符是等宽的，每个字宽约等于字号
        int width = impl->measureWidth("测");
        // 24px 字号，单个汉字宽度应在 20-28 之间
        REQUIRE(width >= 20);
        REQUIRE(width <= 28);
    }

    SECTION("多字符宽度递增") {
        int w1 = impl->measureWidth("A");
        int w2 = impl->measureWidth("AB");
        int w3 = impl->measureWidth("ABC");
        // 宽度应严格递增
        REQUIRE(w2 > w1);
        REQUIRE(w3 > w2);
    }
}

TEST_CASE("TextMetricsImpl 高度测量",
          "[textMetrics]") {
    auto impl = createTextMetricsImpl("", 32, 0);
    REQUIRE(impl != nullptr);

    SECTION("高度与字号正相关") {
        int height = impl->measureHeight("H");
        // 32px 字号，行高通常在 32-48 之间
        // （包含上升量 ascent 和下降量 descent）
        REQUIRE(height >= 32);
        REQUIRE(height <= 48);
    }

    SECTION("不同文本高度相同（单行）") {
        int h1 = impl->measureHeight("A");
        int h2 = impl->measureHeight("Hello World");
        // 单行文本高度由字体决定，与内容无关
        REQUIRE(h1 == h2);
    }
}

TEST_CASE("TextMetricsImpl 字体切换",
          "[textMetrics]") {
    auto impl = createTextMetricsImpl("", 16, 0);
    REQUIRE(impl != nullptr);

    SECTION("切换字号后宽度变化") {
        int w_small = impl->measureWidth("Test");

        // 切换到更大的字号
        impl->setFont("", 32, 0);
        int w_large = impl->measureWidth("Test");

        // 32px 字号应比 16px 宽
        REQUIRE(w_large > w_small);
    }

    SECTION("切换粗体后宽度可能变化") {
        int w_normal = impl->measureWidth("Test");

        // 切换到粗体（STYLE_BOLD = 1）
        impl->setFont("", 16, 1);
        int w_bold = impl->measureWidth("Test");

        // 粗体宽度通常 >= 普通宽度
        // 某些字体粗体宽度相同，所以用 >= 而不是 >
        REQUIRE(w_bold >= w_normal);
    }
}
```

### 4.3 测试设计要点

上述测试遵循几个重要原则：

**1. 绕过 ncbind 直接测试实现层**

TextMetrics 的 ncbind 注册（`NCB_REGISTER_CLASS`）需要完整的 TJS2 虚拟机运行环境，在单元测试中启动 VM 成本极高。所以我们直接测试 `TextMetricsImpl`——这是纯 C++ 类，不依赖 TJS。这也验证了"抽象接口 + 平台实现"架构的测试友好性。

**2. 使用范围断言而非精确值**

字体渲染结果因平台、字体版本、DPI 设置而异。`REQUIRE(width == 72)` 这样的精确断言必然在某个平台上失败。正确做法是使用范围断言：`REQUIRE(width >= 20); REQUIRE(width <= 28);`。

**3. SECTION 实现测试隔离**

Catch2 的 `SECTION`（测试分段）机制会为每个 SECTION 重新执行 `TEST_CASE` 的前置代码。这意味着每个 SECTION 都拿到一个全新的 `impl` 实例，互不干扰。例如"切换字号"的 SECTION 不会影响"切换粗体"的结果。

**4. Fallback 字体测试**

测试中使用空字符串 `""` 作为字体名，触发各平台的 fallback 逻辑（Windows → "MS Gothic"、Linux → "Noto Sans CJK SC" 等）。这确保了 fallback 机制在所有平台上都能正常工作。如果需要测试特定字体，可以在 `test_files/` 下放置 `.ttf` 文件，通过 `TEST_FILES_PATH` 宏引用：

```cpp
// 使用测试专用字体文件（如果需要精确测试）
#include "test_config.h"
std::string fontPath = std::string(TEST_FILES_PATH)
                     + "/fonts/NotoSansSC-Regular.ttf";
auto impl = createTextMetricsImpl(fontPath, 16, 0);
```

### 4.4 修改测试 CMakeLists.txt

将新测试文件注册到 `tests/unit-tests/plugins/CMakeLists.txt`：

```cmake
# tests/unit-tests/plugins/CMakeLists.txt
cmake_minimum_required(VERSION 3.16)
project(TestPlugins LANGUAGES CXX)

# 源文件列表 — 每个 .cpp 生成一个独立的测试可执行文件
set(SOURCES
        psbfile-dll.cpp
        textMetrics-test.cpp    # 新增
)

# 将 .cpp 后缀去掉得到可执行文件名
string(REPLACE ".cpp" "" BASENAMES_SOURCES "${SOURCES}")
set(TARGETS ${BASENAMES_SOURCES})

# 每个源文件 + main.cpp → 一个独立可执行文件
foreach(name ${TARGETS})
    add_executable(${name} ${name}.cpp main.cpp)
endforeach()

set(ALL_TARGETS
        ${TARGETS}
)

# 统一链接 Catch2 + 项目库
foreach(name ${ALL_TARGETS})
    target_link_libraries(${name}
        PRIVATE
            Catch2::Catch2
        PUBLIC
            krkr2plugin krkr2core
    )
    # 让测试找到 test_config.h（包含 TEST_FILES_PATH）
    target_include_directories(${name}
        PRIVATE "${TEST_CONFIG_DIR}"
    )
    # 自动发现所有 TEST_CASE 并注册到 CTest
    catch_discover_tests(${name})
endforeach()
```

**关键改动只有一行**：在 `SOURCES` 中添加 `textMetrics-test.cpp`。CMake 的 `foreach` 循环会自动为它创建可执行文件 `textMetrics-test`，链接 Catch2 和项目库，并注册到 CTest。

### 4.5 测试运行命令

```bash
# === Windows ===
# 先构建
cmake --preset="Windows Debug Config"
cmake --build --preset="Windows Debug Build"
# 运行所有测试
ctest --test-dir out/windows/debug --output-on-failure
# 仅运行 textMetrics 测试
ctest --test-dir out/windows/debug -R textMetrics --output-on-failure

# === Linux ===
cmake --preset="Linux Debug Config"
cmake --build --preset="Linux Debug Build"
ctest --test-dir out/linux/debug -R textMetrics --output-on-failure

# === macOS ===
cmake --preset="MacOS Debug Config"
cmake --build --preset="MacOS Debug Build"
ctest --test-dir out/macos/debug -R textMetrics --output-on-failure

# === 直接运行测试可执行文件（更详细的输出）===
# Windows
./out/windows/debug/tests/unit-tests/plugins/textMetrics-test.exe
# Linux
./out/linux/debug/tests/unit-tests/plugins/textMetrics-test
# macOS
./out/macos/debug/tests/unit-tests/plugins/textMetrics-test
```

`-R textMetrics` 是 CTest 的正则过滤参数（`-R` 代表 `--tests-regex`），只运行名称匹配 `textMetrics` 的测试用例。`--output-on-failure` 让失败的测试打印完整输出，方便调试。

---

## 五、跨平台构建验证

插件开发完成后，必须在四个平台上验证构建。以下是每个平台的完整验证流程和可能遇到的问题。

### 5.1 Windows 验证

```bash
# 环境要求：
# - Visual Studio 2022（或更高版本）
# - CMake 3.28+
# - vcpkg（VCPKG_ROOT 已设置）
# - winflexbison（PATH 中可找到）

# 配置（首次会安装 vcpkg 依赖）
cmake --preset="Windows Debug Config"

# 构建
cmake --build --preset="Windows Debug Build"

# 验证 textMetrics 库文件生成
dir out\windows\debug\cpp\plugins\textMetrics\textMetrics.lib
# 应显示 .lib 文件（静态库）

# 运行测试
ctest --test-dir out\windows\debug -R textMetrics --output-on-failure
```

**Windows 特有问题：**
- **GDI 函数未定义**：确认 CMakeLists.txt 中有 `target_link_libraries(... PRIVATE gdi32)`
- **Unicode 编码**：如果源码中有中文字符串，确保文件保存为 UTF-8 with BOM（MSVC 默认不识别无 BOM 的 UTF-8）
- **CreateFontW 链接错误**：Windows API 函数在 `gdi32.lib` 中，不需要额外的 SDK 配置

### 5.2 Linux 验证

```bash
# 环境要求：
# - GCC 9+ 或 Clang 10+
# - CMake 3.28+
# - vcpkg（VCPKG_ROOT 已设置）
# - 系统字体（通常预装 Noto 或 DejaVu 系列）

# 配置（vcpkg 自动安装 freetype 和 fontconfig）
cmake --preset="Linux Debug Config"

# 观察 vcpkg 安装日志，确认：
# Installing freetype:x64-linux ...
# Installing fontconfig:x64-linux ...

# 构建
cmake --build --preset="Linux Debug Build"

# 验证静态库
ls -la out/linux/debug/cpp/plugins/textMetrics/libtextMetrics.a
# 应显示 .a 文件

# 运行测试
ctest --test-dir out/linux/debug -R textMetrics --output-on-failure
```

**Linux 特有问题：**
- **Fontconfig 找不到字体**：Docker 环境中可能没有安装字体包。安装 `fonts-noto-cjk` 或在测试中使用自带的 `.ttf` 文件
- **FreeType 头文件路径**：vcpkg 安装的 FreeType 头文件在 `<vcpkg_installed>/<triplet>/include/freetype2/` 下，`find_package(Freetype)` 会自动处理
- **pkg-config 缺失**：某些 vcpkg 包需要 `pkg-config`，确保已安装：`sudo apt install pkg-config`

### 5.3 macOS 验证

```bash
# 环境要求：
# - Xcode 14+（含 Command Line Tools）
# - CMake 3.28+
# - vcpkg（VCPKG_ROOT 已设置）

# 配置
cmake --preset="MacOS Debug Config"
# 不应安装 freetype/fontconfig（macOS 使用 Core Text）

# 构建
cmake --build --preset="MacOS Debug Build"

# 验证
ls -la out/macos/debug/cpp/plugins/textMetrics/libtextMetrics.a

# 运行测试
ctest --test-dir out/macos/debug -R textMetrics --output-on-failure
```

**macOS 特有问题：**
- **Core Text 框架未找到**：确认 Xcode Command Line Tools 已安装：`xcode-select --install`
- **arm64 vs x86_64**：M1/M2 Mac 默认编译 arm64。如需 x86_64，在 CMakePresets.json 中设置 `CMAKE_OSX_ARCHITECTURES`
- **CFRelease 顺序**：Core Text 对象必须按创建的逆序释放（详见上一节的实现说明）

### 5.4 Android 验证

```bash
# 环境要求：
# - Android SDK + NDK r25+
# - Java 17+（Gradle 需要）
# - vcpkg（VCPKG_ROOT 已设置）

# Android 通过 Gradle 触发 CMake 构建
./platforms/android/gradlew -p ./platforms/android assembleDebug

# 验证 textMetrics 被编译进 .so
# 查看最终的共享库大小是否增加
ls -la platforms/android/app/build/intermediates/cmake/debug/obj/arm64-v8a/libkrkr2.so

# 使用 nm 或 readelf 检查符号（可选）
# arm64 工具链中的 nm 在 NDK 中
$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-nm \
    platforms/android/app/build/intermediates/cmake/debug/obj/arm64-v8a/libkrkr2.so \
    | grep -i textmetrics
```

**Android 特有问题：**
- **FreeType 交叉编译**：vcpkg 会自动处理 Android 交叉编译，但需要正确配置 `VCPKG_TARGET_TRIPLET`（通常是 `arm64-android` 或 `x64-android`）
- **字体路径硬编码**：`/system/fonts/` 在真机上可能需要 root 权限才能列目录。更安全的方式是使用 Android Asset Manager 或预知的字体文件名
- **NDK API Level**：`std::filesystem` 需要 NDK API 29+。如果目标 minSdkVersion < 29，需要用 `opendir` / `readdir` 替代

### 5.5 构建验证检查清单

完成四平台构建后，用以下清单逐项确认：

| 检查项 | Windows | Linux | macOS | Android |
|--------|---------|-------|-------|---------|
| CMake 配置成功 | ✅ | ✅ | ✅ | ✅ |
| vcpkg 依赖安装正确 | 无额外依赖 | freetype + fontconfig | 无额外依赖 | freetype |
| 静态库生成 | `.lib` | `.a` | `.a` | 打包进 `.so` |
| 链接无错误 | ✅ | ✅ | ✅ | ✅ |
| 单元测试通过 | ✅ | ✅ | ✅ | N/A（Android 无 CTest） |
| 主程序启动正常 | ✅ | ✅ | ✅ | ✅ |
| TJS `new TextMetrics()` 成功 | ✅ | ✅ | ✅ | ✅ |

---

## 六、调试技巧与排查指南

### 6.1 插件注册失败的排查

最常见的问题是"插件编译成功但 TJS 找不到类"。排查步骤：

```
问题：TJS 执行 var tm = new TextMetrics("", 16); 抛出 "class not found"

排查流程：
┌─────────────────────────────────────────┐
│ 1. 检查 ncbind 注册宏是否正确            │
│    NCB_REGISTER_CLASS(TextMetrics) {     │
│        // 必须有 Constructor<>           │
│    }                                     │
├─────────────────────────────────────────┤
│ 2. 检查 textMetrics 是否链接进 krkr2plugin│
│    cpp/plugins/CMakeLists.txt:           │
│    target_link_libraries(... textMetrics)│
├─────────────────────────────────────────┤
│ 3. 检查注册回调是否被调用                 │
│    在 main.cpp 开头添加：                 │
│    spdlog::get("plugin")->info(          │
│        "TextMetrics registering");       │
│    重新构建运行，查看日志                 │
├─────────────────────────────────────────┤
│ 4. 检查静态初始化顺序                     │
│    MSVC 有时会在链接时丢弃"未引用"的 .obj │
│    解决：在 CMakeLists.txt 中用 PUBLIC    │
│    而非 PRIVATE 添加 main.cpp            │
└─────────────────────────────────────────┘
```

### 6.2 链接错误排查

**典型错误 1：未定义引用（Undefined Reference）**

```
ld: error: undefined reference to `TextMetricsImpl::measureWidth(std::string const&)'
```

**原因：** 平台相关的 `.cpp` 文件未被编译。检查 CMakeLists.txt 中的 `if(WINDOWS)` / `elseif(MACOS)` / `else()` 分支是否覆盖了当前平台。

**验证方法：** 在 CMakeLists.txt 中添加调试输出：

```cmake
if(WINDOWS)
    message(STATUS "textMetrics: 使用 Windows GDI 实现")
    target_sources(${PROJECT_NAME} PRIVATE TextMetricsImpl_Win.cpp)
elseif(MACOS)
    message(STATUS "textMetrics: 使用 macOS Core Text 实现")
    target_sources(${PROJECT_NAME} PRIVATE TextMetricsImpl_CT.cpp)
else()
    message(STATUS "textMetrics: 使用 FreeType 实现")
    target_sources(${PROJECT_NAME} PRIVATE TextMetricsImpl_FT.cpp)
endif()
```

重新运行 CMake 配置，观察输出中是否出现了预期的 `message`。

**典型错误 2：find_package 失败**

```
CMake Error at cpp/plugins/textMetrics/CMakeLists.txt:XX (find_package):
  Could not find a package configuration file provided by "Freetype"
```

**原因：** vcpkg 未安装 FreeType，或 `vcpkg.json` 中未声明。

**解决：**
1. 确认 `vcpkg.json` 中有 `freetype` 依赖
2. 确认 `VCPKG_ROOT` 环境变量正确
3. 手动安装验证：`$VCPKG_ROOT/vcpkg install freetype:x64-linux`

**典型错误 3：重复符号（Duplicate Symbol）**

```
ld: error: duplicate symbol: TextMetricsImpl::measureWidth(...)
```

**原因：** 多个平台的实现文件被同时编译（`if/elseif/else` 逻辑有误，或错误地将所有 `.cpp` 都放在了 `PUBLIC` 源文件中）。

**解决：** 确保每个平台只编译一个 `TextMetricsImpl_*.cpp`。使用 `cmake --build --target textMetrics --verbose` 查看实际编译了哪些文件。

### 6.3 字体相关问题排查

**Linux 下 Fontconfig 找不到字体：**

```cpp
// 在 TextMetricsImpl_FT.cpp 中添加调试日志
FcPattern* pattern = FcNameParse(
    (const FcChar8*)fontName.c_str()
);
FcConfigSubstitute(nullptr, pattern, FcMatchPattern);
FcDefaultSubstitute(pattern);
FcResult result;
FcPattern* match = FcFontMatch(nullptr, pattern, &result);
if (!match) {
    spdlog::get("plugin")->error(
        "Fontconfig: 找不到字体 '{}'", fontName
    );
    // fallback: 尝试 "sans-serif"
    pattern = FcNameParse((const FcChar8*)"sans-serif");
    FcConfigSubstitute(nullptr, pattern, FcMatchPattern);
    FcDefaultSubstitute(pattern);
    match = FcFontMatch(nullptr, pattern, &result);
}
// 获取匹配的字体文件路径
FcChar8* path = nullptr;
FcPatternGetString(match, FC_FILE, 0, &path);
spdlog::get("plugin")->info(
    "Fontconfig: 匹配到字体文件 '{}'",
    reinterpret_cast<const char*>(path)
);
```

**Docker/CI 环境字体缺失：**

```dockerfile
# 在 Linux Dockerfile 中安装字体
RUN apt-get update && apt-get install -y \
    fonts-noto-cjk \
    fonts-dejavu-core \
    && fc-cache -fv
```

**Android 字体枚举：**

```cpp
// 列出 /system/fonts/ 下的所有字体文件
#include <dirent.h>

void listAndroidFonts() {
    DIR* dir = opendir("/system/fonts/");
    if (!dir) {
        spdlog::get("plugin")->error(
            "无法打开 /system/fonts/"
        );
        return;
    }
    struct dirent* entry;
    while ((entry = readdir(dir)) != nullptr) {
        // 只列出 .ttf 和 .otf 文件
        std::string name = entry->d_name;
        if (name.ends_with(".ttf")
            || name.ends_with(".otf")) {
            spdlog::get("plugin")->info(
                "Android 字体: {}", name
            );
        }
    }
    closedir(dir);
}
```

---

## 动手实践

按以下步骤在本地完成 textMetrics 插件的构建集成：

### 步骤 1：创建目录和文件

```bash
# 在项目根目录执行
mkdir -p cpp/plugins/textMetrics

# 将前两节编写的源文件复制到此目录
# （如果你跟着教程走，应该已经有这些文件了）
# 确认文件列表：
ls cpp/plugins/textMetrics/
# 预期输出：
# CMakeLists.txt  main.cpp  TextMetrics.h  TextMetrics.cpp
# TextMetricsImpl.h  TextMetricsImpl_Win.cpp
# TextMetricsImpl_FT.cpp  TextMetricsImpl_CT.cpp
```

### 步骤 2：编写 CMakeLists.txt

将本节第一章的完整 CMakeLists.txt 内容写入 `cpp/plugins/textMetrics/CMakeLists.txt`。

### 步骤 3：修改父级 CMakeLists.txt

编辑 `cpp/plugins/CMakeLists.txt`，添加 `add_subdirectory(textMetrics)` 和 `target_link_libraries(... textMetrics)`。

### 步骤 4：修改 vcpkg.json

添加 `freetype`（`linux | android`）和 `fontconfig`（`linux`）依赖。

### 步骤 5：配置并构建

```bash
# 以 Linux 为例
cmake --preset="Linux Debug Config"
# 观察输出，确认：
# 1. vcpkg 安装了 freetype 和 fontconfig
# 2. CMake 成功配置 textMetrics 目标
# 3. 无错误或警告

cmake --build --preset="Linux Debug Build"
# 确认编译成功，无链接错误
```

### 步骤 6：编写并运行测试

将本节第四章的测试代码写入 `tests/unit-tests/plugins/textMetrics-test.cpp`，修改测试 CMakeLists.txt，然后：

```bash
# 重新配置（让 CMake 发现新的测试文件）
cmake --preset="Linux Debug Config"
cmake --build --preset="Linux Debug Build"

# 运行测试
ctest --test-dir out/linux/debug -R textMetrics --output-on-failure

# 预期输出：
# Test project /path/to/out/linux/debug
#     Start 1: TextMetricsImpl 构造与基本属性/字体大小正确
# 1/8 Test #1: ... Passed  0.01 sec
#     Start 2: TextMetricsImpl 构造与基本属性/字体风格为 NORMAL
# 2/8 Test #2: ... Passed  0.01 sec
# ... 全部 8 个测试通过 ...
# 100% tests passed, 0 tests failed out of 8
```

### 步骤 7：验证 TJS 集成（可选）

构建完主程序后，编写一个简单的 TJS 测试脚本：

```javascript
// test_textmetrics.tjs
// 测试 TextMetrics 插件是否正确注册

var tm = new TextMetrics("", 24);
System.inform("fontName = " + tm.fontName);
System.inform("fontSize = " + tm.fontSize);

var width = tm.measureWidth("Hello, 世界！");
System.inform("width = " + width);

var height = tm.measureHeight("Hello");
System.inform("height = " + height);

var info = tm.measureText("测试文本");
System.inform("ascent = " + info["ascent"]);
System.inform("descent = " + info["descent"]);

var lines = tm.wrapText("这是一段很长的文本，需要自动换行处理。在窗口宽度有限的情况下，文本应该被正确地分割成多行。", 200);
for (var i = 0; i < lines.count; i++) {
    System.inform("line[" + i + "] = " + lines[i]);
}
```

---

## 对照项目源码

以下表格列出本节涉及的项目源文件，读者可以对照学习：

| 文件路径 | 行数范围 | 本节相关内容 |
|----------|----------|-------------|
| `cpp/plugins/CMakeLists.txt` | 第 1-74 行 | 父级插件库的完整结构，add_subdirectory + target_link_libraries 模式 |
| `cpp/plugins/CMakeLists.txt` | 第 52-58 行 | 子目录插件注册（psdfile、psbfile、layerex_draw、motionplayer、fstat） |
| `cpp/plugins/CMakeLists.txt` | 第 61-68 行 | 子插件链接声明（PUBLIC 链接传播） |
| `cpp/plugins/fstat/CMakeLists.txt` | 第 1-10 行 | 最简单的子插件 CMake 示例（无外部依赖） |
| `cpp/plugins/psdfile/CMakeLists.txt` | 第 1-16 行 | 带子库（psdparse）的子插件示例 |
| `cpp/plugins/layerex_draw/CMakeLists.txt` | 第 1-30 行 | 双目录条件编译 + vcpkg 依赖（libgdiplus）示例 |
| `vcpkg.json` | 第 1-94 行 | 完整的项目依赖清单，平台限定符使用示例 |
| `tests/CMakeLists.txt` | 第 1-18 行 | 顶层测试配置：find_package(Catch2)、TEST_FILES_PATH |
| `tests/unit-tests/plugins/CMakeLists.txt` | 第 1-28 行 | 插件测试的 CMake 模式（foreach 自动创建可执行文件） |
| `tests/unit-tests/plugins/main.cpp` | 第 1-18 行 | 自定义测试入口（spdlog 初始化 + Catch::Session） |
| `tests/unit-tests/plugins/psbfile-dll.cpp` | 第 1-215 行 | 现有插件测试示例（TEST_CASE + SECTION + REQUIRE 模式） |
| `tests/test_config.h.in` | 第 1-6 行 | CMake configure_file 模板，生成 TEST_FILES_PATH 宏 |

---

## 常见错误与排查

### 错误 1：使用 CMAKE_SYSTEM_NAME 而非项目自定义变量

```cmake
# ❌ 错误写法
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    target_sources(${PROJECT_NAME} PRIVATE TextMetricsImpl_Win.cpp)
endif()

# ✅ 正确写法
if(WINDOWS)
    target_sources(${PROJECT_NAME} PRIVATE TextMetricsImpl_Win.cpp)
endif()
```

**原因：** KrKr2 在根 CMakeLists.txt 中定义了 `WINDOWS` / `LINUX` / `MACOS` / `ANDROID` 四个布尔变量。这些变量在交叉编译时（如用 Linux 主机编译 Android 目标）仍然正确反映目标平台，而 `CMAKE_SYSTEM_NAME` 在某些 vcpkg 工具链文件中可能被覆盖。

### 错误 2：忘记在父级 CMakeLists.txt 中链接新插件

```cmake
# 只添加了 add_subdirectory，忘记了 target_link_libraries
add_subdirectory(textMetrics)  # ← 这行有了

# ❌ 但忘记了这行：
target_link_libraries(${PROJECT_NAME} PUBLIC textMetrics)
```

**症状：** 编译成功，但运行时 TJS 找不到 `TextMetrics` 类。因为 `textMetrics.a` / `textMetrics.lib` 虽然生成了，但没有链接进 `krkr2plugin`，主程序中不包含它的符号。

**排查：** 用 `nm`（Linux/macOS）或 `dumpbin`（Windows）检查最终二进制中是否包含 TextMetrics 相关符号：

```bash
# Linux/macOS
nm out/linux/debug/krkr2 | grep -i textmetrics
# 应看到 T（text段，已定义）符号

# Windows
dumpbin /symbols out\windows\debug\krkr2.exe | findstr TextMetrics
```

### 错误 3：Catch2 测试链接失败——找不到 TextMetricsImpl

```
ld: error: undefined reference to `createTextMetricsImpl(...)'
```

**原因：** 测试可执行文件链接了 `krkr2plugin` 和 `krkr2core`，但 `krkr2plugin` 可能没有正确传播 `textMetrics` 的符号。

**解决方案 A：** 确认 `cpp/plugins/CMakeLists.txt` 中用 `PUBLIC` 链接了 `textMetrics`：

```cmake
target_link_libraries(${PROJECT_NAME} PUBLIC textMetrics)
#                                      ^^^^^^ 必须是 PUBLIC
```

**解决方案 B：** 如果 A 不生效，在测试 CMakeLists.txt 中直接链接 `textMetrics`：

```cmake
foreach(name ${ALL_TARGETS})
    target_link_libraries(${name}
        PRIVATE Catch2::Catch2
        PUBLIC krkr2plugin krkr2core textMetrics
    )
endforeach()
```

---

## 本节小结

1. **textMetrics CMakeLists.txt** 使用"单目录 + 条件编译"模式，通过 `if(WINDOWS)` / `elseif(MACOS)` / `else()` 选择不同平台的实现文件
2. **父级集成** 需要在 `cpp/plugins/CMakeLists.txt` 中同时添加 `add_subdirectory` 和 `target_link_libraries`，缺少任一步都会导致插件"静默消失"
3. **vcpkg 依赖** 使用平台限定符精确控制：FreeType 仅在 `linux | android`，Fontconfig 仅在 `linux`，Windows 和 macOS 使用系统 API 无需额外依赖
4. **Catch2 测试** 采用"绕过 ncbind 测试实现层"策略，避免启动 TJS VM 的开销；使用范围断言适应跨平台字体差异
5. **测试 CMake 集成** 只需在 `SOURCES` 列表中添加新文件名，foreach 循环自动处理其余工作
6. **跨平台验证** 需要检查四个平台：Windows（GDI）、Linux（FreeType + Fontconfig）、macOS（Core Text）、Android（FreeType）
7. **项目使用自定义平台变量** `WINDOWS` / `LINUX` / `MACOS` / `ANDROID`，不要使用 CMake 内置的 `WIN32` / `UNIX` / `APPLE`
8. **静态库链接传播**：子插件用 `PRIVATE` 链接 `krkr2plugin`（获取头文件），父级用 `PUBLIC` 链接子插件（传播符号到主程序）
9. **调试插件注册失败** 的关键是检查 ncbind 宏 → 父级链接 → spdlog 日志 → 符号表
10. **Docker/CI 环境** 需要额外安装字体包（如 `fonts-noto-cjk`），否则字体相关测试会因找不到字体而失败

---

## 练习题与答案

### 题目 1：为 textMetrics 添加 Release 构建优化

当前 CMakeLists.txt 只关注功能正确性。请修改它，使 Release 模式下开启以下优化：
- Windows: 启用 `/O2` 和 `/GL`（全程序优化）
- Linux/macOS: 启用 `-O2` 和 `-flto`（链接时优化，Link-Time Optimization，编译器在链接阶段跨编译单元优化的技术）
- 所有平台: 定义 `NDEBUG` 宏（禁用 assert）

<details>
<summary>查看答案</summary>

```cmake
# 在 CMakeLists.txt 末尾添加（target_link_libraries 之后）

# Release 构建优化
# CMAKE_BUILD_TYPE 在单配置生成器（Ninja/Make）中有效
# 多配置生成器（Visual Studio）使用 generator expressions
target_compile_options(${PROJECT_NAME} PRIVATE
    # generator expression: 仅在 Release 配置时生效
    $<$<CONFIG:Release>:
        # MSVC 编译器
        $<$<CXX_COMPILER_ID:MSVC>:/O2 /GL>
        # GCC 或 Clang
        $<$<OR:$<CXX_COMPILER_ID:GNU>,$<CXX_COMPILER_ID:Clang>>:-O2 -flto>
    >
)

# Release 模式定义 NDEBUG
target_compile_definitions(${PROJECT_NAME} PRIVATE
    $<$<CONFIG:Release>:NDEBUG>
)

# MSVC 全程序优化需要链接器也开启 /LTCG
if(MSVC)
    target_link_options(${PROJECT_NAME} PRIVATE
        $<$<CONFIG:Release>:/LTCG>
    )
endif()
```

**关键点：**
- 使用 CMake 的 **Generator Expression**（生成器表达式，用 `$<...>` 语法在生成构建系统时动态求值的表达式）区分 Debug 和 Release 配置
- `$<CONFIG:Release>` 在 Release 构建时为真
- `$<CXX_COMPILER_ID:MSVC>` 区分编译器，避免把 MSVC 参数传给 GCC
- `/GL` + `/LTCG` 是 MSVC 的链接时优化组合，`-flto` 是 GCC/Clang 的等价物

</details>

### 题目 2：编写一个测试 wrapText 中日混排换行的 TEST_CASE

请编写一个 Catch2 测试用例，验证 `wrapText` 在处理中日文混排文本时的正确性。要求：
- 输入文本包含中文、日文假名、ASCII 字符
- 设定 `maxWidth` 为 100 像素
- 验证返回的行数 > 1（文本确实被换行了）
- 验证每行宽度不超过 `maxWidth`（需要调用 `measureWidth` 验证）

<details>
<summary>查看答案</summary>

```cpp
TEST_CASE("TextMetricsImpl wrapText 中日混排",
          "[textMetrics][wrapText]") {
    // 24px 字号，每个 CJK 字符约 24px 宽
    auto impl = createTextMetricsImpl("", 24, 0);
    REQUIRE(impl != nullptr);

    // 中日文混排测试文本
    // 包含：中文汉字 + 日文平假名 + ASCII
    std::string text =
        "今日はとても良い天気ですね。"
        "Hello World! 你好世界！"
        "テストtestテスト";

    int maxWidth = 100;  // 100px，约能放 4 个 CJK 字符

    // 调用 wrapText 获取换行结果
    auto lines = impl->wrapText(text, maxWidth);

    SECTION("文本被换行成多行") {
        // 文本总长远超 100px，必须换行
        REQUIRE(lines.size() > 1);
        // 原文约 30+ 字符，100px 约放 4 个
        // 至少应有 5 行以上
        REQUIRE(lines.size() >= 5);
    }

    SECTION("每行宽度不超过 maxWidth") {
        for (size_t i = 0; i < lines.size(); i++) {
            int lineWidth = impl->measureWidth(lines[i]);
            INFO("行 " << i << ": \""
                 << lines[i] << "\" 宽度="
                 << lineWidth);
            // 允许 1-2 像素误差
            // （CJK 字符不可分割，最后一个字符
            //  可能略微超出）
            REQUIRE(lineWidth <= maxWidth + 2);
        }
    }

    SECTION("所有行拼接等于原文") {
        std::string joined;
        for (const auto& line : lines) {
            joined += line;
        }
        // 换行可能移除空格，所以用 find
        // 或直接比较去空格后的结果
        // 这里简单检查长度一致性
        REQUIRE(joined.length() > 0);
        REQUIRE(joined.length() <= text.length());
    }
}
```

**关键点：**
- 使用 `INFO` 宏（Catch2 的上下文信息宏，仅在断言失败时显示）在测试失败时打印每行内容和宽度，方便定位问题
- 允许 1-2 像素误差，因为 CJK 字符是不可分割的原子单位
- 中日混排文本包含三种字符集，测试了 wrapText 的多语言处理能力
- `[textMetrics][wrapText]` 是 Catch2 的标签系统，可以用 `-c "[wrapText]"` 只运行带此标签的测试

</details>

### 题目 3：解释以下 CMake 配置错误的根因并给出修复方案

```cmake
# 某开发者写的 textMetrics CMakeLists.txt
cmake_minimum_required(VERSION 3.19)
project(text_metrics)  # 注意：用了下划线

add_library(${PROJECT_NAME} STATIC)
target_sources(${PROJECT_NAME} PUBLIC
    main.cpp TextMetrics.cpp
    TextMetricsImpl_Win.cpp
    TextMetricsImpl_FT.cpp
    TextMetricsImpl_CT.cpp
)
target_link_libraries(${PROJECT_NAME} PRIVATE krkr2plugin)
```

在 `cpp/plugins/CMakeLists.txt` 中写了：

```cmake
add_subdirectory(textMetrics)
target_link_libraries(${PROJECT_NAME} PUBLIC textMetrics)
```

请指出至少 3 个错误并给出修复方案。

<details>
<summary>查看答案</summary>

**错误 1：项目名不匹配**

子目录用 `project(text_metrics)`（下划线），父级链接 `textMetrics`（驼峰）。CMake 会报错 `Target "textMetrics" was not created in this directory`。

修复：统一命名为 `project(textMetrics)`。

**错误 2：所有平台实现文件被同时编译**

三个 `TextMetricsImpl_*.cpp` 全部在 `target_sources` 中，没有条件编译。在 Linux 上编译时，`TextMetricsImpl_Win.cpp` 会因 `#include <windows.h>` 找不到而报错。在 Windows 上，`TextMetricsImpl_FT.cpp` 会因 `#include <freetype/freetype.h>` 找不到而报错。

修复：使用 `if(WINDOWS)` / `elseif(MACOS)` / `else()` 条件选择。

**错误 3：缺少平台相关链接库**

没有链接 `gdi32`（Windows）、`CoreText` + `CoreFoundation`（macOS）、`Freetype`（Linux/Android）、`Fontconfig`（Linux）。即使源文件编译通过（假设头文件能找到），链接阶段也会报 undefined reference。

修复：在对应的 `if` 分支中添加 `find_package` 和 `target_link_libraries`。

**错误 4（额外发现）：平台实现文件用了 PUBLIC**

`TextMetricsImpl_*.cpp` 应该用 `PRIVATE`，因为它们只是虚函数实现，不暴露新的公共符号。用 `PUBLIC` 在某些情况下可能导致上层目标也尝试编译这些文件。

修复：平台无关文件（main.cpp、TextMetrics.cpp）用 `PUBLIC`，平台相关文件用 `PRIVATE`。

</details>

---

## 下一步

本章（第五章"实战——从零开发插件"）到此完成。三节内容覆盖了插件开发的完整流程：从需求分析、C++ 实现、TJS 绑定，到 CMake 构建、vcpkg 集成、单元测试。

下一章 → [已实现插件总览](../06-插件清单与状态/01-已实现插件总览.md)：系统梳理 KrKr2 中所有已实现的插件，包括功能概述、注册方式、依赖关系和源码位置。

