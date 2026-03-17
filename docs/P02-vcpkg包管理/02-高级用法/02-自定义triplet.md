# 02-自定义 Triplet（架构-系统-链接方式）

> **所属模块：** P02-vcpkg包管理 / 02-高级用法  
> **前置知识：** [01-overlay-ports.md](./01-overlay-ports.md)  
> **预计阅读时间：** 30-40 分钟  
> **后续章节：** [03-二进制缓存.md](./03-二进制缓存.md)  
> **适用平台：** Windows / Linux / macOS / Android

Triplet 是 vcpkg 最核心的配置对象之一。  
你在命令里写的 `x64-windows`、`arm64-android`，本质上都对应一个 `.cmake` 文件。  
这个文件会告诉 vcpkg：目标系统是什么、架构是什么、库要静态还是动态、是否需要链式工具链、是否传入额外编译参数。

很多人把 Triplet 当作“固定枚举值”，这是误区。  
在项目实践里，Triplet 可以自定义，并且经常需要自定义，尤其是 Android、混合链接策略、跨平台统一构建场景。

## 本节目标

读完本节后，你将能够：

1. 准确解释 Triplet 的三元组语义：平台 + 架构 + 链接方式。
2. 识别常见内置 Triplet，并理解其适用场景。
3. 掌握 Triplet 文件常见字段及含义。
4. 从零创建并启用自定义 Triplet。
5. 使用 overlay-triplets 让 vcpkg 识别项目内 Triplet。
6. 结合 KrKr2 的 `vcpkg_android.cmake` 理解 Android Triplet 特殊配置。
7. 根据工程目标选择静态链接与动态链接策略。

## 1. Triplet 是什么

Triplet（三元组）可以理解为“依赖构建配置模板”。  
同一个依赖包，使用不同 Triplet 构建，最终产物会不同。  
例如 `zlib:x64-windows` 与 `zlib:x64-windows-static`，虽然都是 zlib，但链接模型和运行时约束不同。

常见命名模式是：

`[架构]-[平台]-[链接倾向]`

示例：

- `x64-windows`
- `x64-windows-static`
- `x64-linux`
- `arm64-android`
- `arm64-osx`

你可以把 Triplet 看成“目标二进制合同（contract）”。  
只要合同变了，依赖编译结果就可能变化，因此构建缓存、ABI 兼容、部署策略都会被影响。

## 2. 内置 Triplet 列表与含义

查看内置列表：

```bash
vcpkg help triplets
```

典型内置项（按平台分组）：

- Windows：`x86-windows`、`x64-windows`、`x64-windows-static`、`arm64-windows`
- Linux：`x64-linux`、`arm64-linux`、`arm-linux`
- macOS：`x64-osx`、`arm64-osx`
- Android：`arm64-android`、`arm-android`、`x64-android`、`x86-android`

这些 Triplet 覆盖了大部分通用场景，但不代表能满足所有项目。  
例如你需要固定 Android API Level、只构建 Release、局部端口动态链接时，往往必须自定义。

## 3. Triplet 文件常见字段

Triplet 文件是 CMake 脚本，通常包含以下字段：

| 字段 | 作用 | 例子 |
| :--- | :--- | :--- |
| `VCPKG_TARGET_ARCHITECTURE` | 目标架构 | `x64` / `arm64` / `x86` / `arm` |
| `VCPKG_CRT_LINKAGE` | C 运行时链接方式 | `static` / `dynamic` |
| `VCPKG_LIBRARY_LINKAGE` | 依赖库链接方式 | `static` / `dynamic` |
| `VCPKG_CMAKE_SYSTEM_NAME` | 目标系统名 | `Windows` / `Linux` / `Darwin` / `Android` |
| `VCPKG_CMAKE_SYSTEM_VERSION` | 目标系统版本 | Android 常设为 API Level，如 `28` |
| `VCPKG_BUILD_TYPE` | 构建类型裁剪 | `release` 或 `debug` |
| `VCPKG_MAKE_BUILD_TRIPLET` | 传给 autotools host | `--host=aarch64-linux-android` |
| `VCPKG_CMAKE_CONFIGURE_OPTIONS` | 额外 CMake 配置参数 | `-DANDROID_ABI=arm64-v8a` |

最小可用示例：

```cmake
set(VCPKG_TARGET_ARCHITECTURE x64)
set(VCPKG_CRT_LINKAGE dynamic)
set(VCPKG_LIBRARY_LINKAGE static)
set(VCPKG_CMAKE_SYSTEM_NAME Linux)
set(VCPKG_BUILD_TYPE release)
```

说明：这表示目标 Linux x64，CRT 动态，库静态，只构建 Release。

## 4. 为什么要自定义 Triplet

KrKr2 这类跨平台项目常见需求：

1. Android 固定 API Level，避免不同 NDK 默认值导致的不一致。
2. 大部分库静态，少数库动态（比如特定运行时加载需求）。
3. 对端口注入平台参数，如 `ANDROID_ABI`。
4. 按工程目标裁剪构建类型，减少 CI 时间。

如果完全依赖内置 Triplet，往往会遇到“能编但不满足发布约束”的情况。

## 5. 自定义 Triplet 创建流程

### 步骤 1：确定目标约束

先明确四件事：

- 目标平台
- 目标架构
- CRT 链接方式
- 依赖库链接方式

### 步骤 2：创建 Triplet 文件

建议放在项目目录：

```text
krkr2/vcpkg/triplets/
```

新建文件：`x64-linux-release-static.cmake`

```cmake
set(VCPKG_TARGET_ARCHITECTURE x64)
set(VCPKG_CRT_LINKAGE dynamic)
set(VCPKG_LIBRARY_LINKAGE static)
set(VCPKG_CMAKE_SYSTEM_NAME Linux)
set(VCPKG_BUILD_TYPE release)
```

### 步骤 3：让 vcpkg 识别该目录

命令行方式：

```bash
vcpkg install zlib:x64-linux-release-static --overlay-triplets=./vcpkg/triplets
```

配置文件方式（`vcpkg-configuration.json`）：

```json
{
  "overlay-triplets": [
    "./vcpkg/triplets"
  ]
}
```

### 步骤 4：验证构建结果

重点看三点：

1. 日志里 Triplet 名称是否正确。
2. 库后缀是否符合预期（静态/动态）。
3. 系统名与架构是否与目标一致。

## 6. overlay-triplets 配置方式

overlay-triplets 的本质是“附加 Triplet 搜索路径”。  
没有它，vcpkg 通常只会看默认 Triplet 集合。

常见三种接入方式：

### 方式 A：命令行临时注入

```bash
vcpkg install fmt:arm64-android --overlay-triplets=./vcpkg/triplets
```

适合本地试验，不适合长期团队协作。

### 方式 B：配置文件持久化

```json
{
  "overlay-triplets": [
    "./vcpkg/triplets"
  ]
}
```

适合团队与 CI，减少“命令参数遗漏”。

### 方式 C：CMake 变量传递

```cmake
set(VCPKG_OVERLAY_TRIPLETS "${CMAKE_SOURCE_DIR}/vcpkg/triplets" CACHE STRING "")
set(VCPKG_TARGET_TRIPLET "arm64-android" CACHE STRING "")
```

适合多 preset 构建流水线。

## 7. Android Triplet 特殊配置（结合 KrKr2）

KrKr2 里真实存在以下 Triplet 文件：

- `vcpkg/triplets/arm64-android.cmake`
- `vcpkg/triplets/arm-android.cmake`
- `vcpkg/triplets/x64-android.cmake`
- `vcpkg/triplets/x86-android.cmake`
- `vcpkg/triplets/android-dynamic-libs.cmake`

还存在一个关键脚本：

- `cmake/vcpkg_android.cmake`

### 7.1 NDK 路径检查

`cmake/vcpkg_android.cmake` 明确检查两个环境变量：

- `ANDROID_NDK_HOME`
- `VCPKG_ROOT`

缺失任意一个会直接 `FATAL_ERROR`。  
这能避免“静默使用错误工具链”导致的隐蔽问题。

### 7.2 ABI 到 Triplet 的映射

KrKr2 脚本中的映射：

| ANDROID_ABI | VCPKG_TARGET_TRIPLET |
| :--- | :--- |
| `arm64-v8a` | `arm64-android` |
| `armeabi-v7a` | `arm-android` |
| `x86_64` | `x64-android` |
| `x86` | `x86-android` |

这个映射必须准确，否则常见结果是链接期报错：对象文件架构不匹配。

### 7.3 API Level 配置

KrKr2 Android Triplet 统一设置：

```cmake
set(VCPKG_CMAKE_SYSTEM_VERSION 28)
```

它决定目标 Android API Level。  
API Level 改变后，系统符号可用性、端口条件编译分支、兼容区间都可能变化。

### 7.4 ABI 与 host 参数注入

以 `arm64-android.cmake` 为例：

```cmake
set(VCPKG_MAKE_BUILD_TRIPLET "--host=aarch64-linux-android")
set(VCPKG_CMAKE_CONFIGURE_OPTIONS -DANDROID_ABI=arm64-v8a)
include(${CMAKE_CURRENT_LIST_DIR}/android-dynamic-libs.cmake)
```

这里同时照顾 autotools 与 CMake 两类端口。  
最后 `include` 的覆盖文件用于处理“局部动态链接”策略。

## 8. 静态链接 vs 动态链接选择策略

### 静态链接适合场景

- 希望部署文件尽量少。
- 希望减少运行时依赖查找问题。
- 希望版本完全内聚在可执行文件中。

### 动态链接适合场景

- 需要运行时替换或单独升级某个库。
- 某些平台或框架要求动态库加载。
- 希望减小主二进制大小。

### KrKr2 的策略

KrKr2 Android Triplet 默认：

```cmake
set(VCPKG_LIBRARY_LINKAGE static)
```

并通过 `android-dynamic-libs.cmake` 局部覆盖：

```cmake
set(DYNAMIC_LIBRARIES sdl2)
if(PORT IN_LIST DYNAMIC_LIBRARIES)
    set(VCPKG_LIBRARY_LINKAGE dynamic)
endif ()
```

这相当于“全局静态 + 局部动态”，兼顾部署稳定性与个别端口需求。

## 9. KrKr2 项目的 Triplet 使用分析

从工程分层看：

1. `cmake/vcpkg_android.cmake` 负责入口条件和工具链拼接。
2. `vcpkg/triplets/*.cmake` 负责具体依赖构建策略。
3. `android-dynamic-libs.cmake` 负责端口级覆盖规则。

这种结构的优势：

- 规则集中，便于团队审查。
- Android ABI 变化时改动点明确。
- 可以快速把“临时修复”沉淀成稳定模板。

## 10. 动手实践

目标：创建一个 Android API 29 的自定义 Triplet，并验证生效。

### 实践步骤

1. 在 `krkr2/vcpkg/triplets/` 下新建 `arm64-android-api29.cmake`。  
2. 写入以下内容：

```cmake
set(VCPKG_TARGET_ARCHITECTURE arm64)
set(VCPKG_CRT_LINKAGE static)
set(VCPKG_LIBRARY_LINKAGE static)
set(VCPKG_CMAKE_SYSTEM_NAME Android)
set(VCPKG_CMAKE_SYSTEM_VERSION 29)
set(VCPKG_MAKE_BUILD_TRIPLET "--host=aarch64-linux-android")
set(VCPKG_CMAKE_CONFIGURE_OPTIONS -DANDROID_ABI=arm64-v8a)
include(${CMAKE_CURRENT_LIST_DIR}/android-dynamic-libs.cmake)
```

3. 执行安装命令：

```bash
vcpkg install zlib:arm64-android-api29 --overlay-triplets=./vcpkg/triplets
```

4. 在日志确认以下关键项：

- `VCPKG_TARGET_TRIPLET=arm64-android-api29`
- `ANDROID_ABI=arm64-v8a`
- `CMAKE_SYSTEM_VERSION=29`

## 11. 本节小结

- Triplet 是 vcpkg 的核心控制面，不只是名称标签。
- 自定义 Triplet 的关键是：写好 `.cmake` + 正确接入 overlay-triplets。
- Android 场景必须重视 NDK、ABI、API Level、工具链链式加载。
- KrKr2 采用“全局静态 + 局部动态”策略，是兼顾可部署性与运行时需求的典型方案。
- 构建异常时，优先核查 Triplet 名称、路径、ABI 映射、CRT 一致性。

## 12. 练习题与答案

### 题目 1：Triplet 调用

你新增了 `vcpkg/triplets/x64-linux-fastdebug.cmake`，请写出安装 `fmt` 的命令，要求显式使用 overlay-triplets。

<details>
<summary>查看答案</summary>

```bash
vcpkg install fmt:x64-linux-fastdebug --overlay-triplets=./vcpkg/triplets
```

</details>

### 题目 2：Android 映射

在 KrKr2 的 Android 辅助脚本里，如果 `ANDROID_ABI=x86_64`，应映射到哪个 Triplet？

<details>
<summary>查看答案</summary>

应映射到 `x64-android`。  
因为脚本使用 ABI 到 Triplet 的固定映射，`x86_64` 对应 vcpkg 的 x64 Android 目标。

</details>

### 题目 3：链接策略

如果你希望“默认全部静态，只有 `sdl2` 动态”，Triplet 侧应如何写？

<details>
<summary>查看答案</summary>

```cmake
set(VCPKG_LIBRARY_LINKAGE static)
set(DYNAMIC_LIBRARIES sdl2)
if(PORT IN_LIST DYNAMIC_LIBRARIES)
    set(VCPKG_LIBRARY_LINKAGE dynamic)
endif ()
```

</details>

## 13. 下一步

请继续阅读：[03-二进制缓存.md](./03-二进制缓存.md)

下一节会解决三个问题：

1. 如何开启 vcpkg 二进制缓存。  
2. 如何在 CI 里复用缓存降低构建时间。  
3. 如何将 Triplet 与缓存策略组合，得到稳定可复现的跨平台构建流程。
