# 本节目标
# 平台条件编译实现

> **所属模块：** M02-项目构建系统深度解析
> **前置知识：** [P01-Ch4/01-Presets配置详解](../P01-现代CMake与构建工具链/04-CMake-Presets/01-Presets配置详解.md)、[本模块第 01 章](./01-根CMakeLists深度解读.md)、[本模块第 02 章](./02-INTERFACE库与模块链接.md)
> **预计阅读时间：** 35 分钟
> **关键源文件：** `CMakePresets.json`（156 行）、`CMakeLists.txt`（143 行）、`cmake/vcpkg_android.cmake`（99 行）

## 本节目标

读完本节后，你将能够：

1. 理解 CMakePresets.json 的完整结构——configurePresets 的继承体系和平台条件
2. 掌握 KrKr2 的四平台（Windows/Linux/macOS/Android）构建配置差异
3. 理解 `condition` 字段如何实现平台自动检测
4. 分析项目中的条件编译宏体系（`WINDOWS`、`LINUX`、`MACOS`、`ANDROID`）
5. 能够为项目添加新的平台配置或修改现有配置

## CMakePresets.json 的作用

CMakePresets.json 是 CMake 3.19 引入的标准化构建配置文件。它解决的核心问题是：**将"如何配置构建"从口口相传变为代码化、版本化的配置**。

在没有 Presets 之前，构建一个跨平台项目通常需要：

```bash
# 每个开发者需要记住这些参数，不同平台还不一样
cmake -B build -G "Ninja" \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_TOOLCHAIN_FILE=/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake \
  -DVCPKG_TARGET_TRIPLET=x64-windows-static-md \
  -DCMAKE_CXX_STANDARD=17 \
  ...
```

有了 Presets 之后：

```bash
# 一条命令，所有参数自动应用
cmake --preset "Windows Debug Config"
cmake --build --preset "Windows Debug Build"
```

---

## KrKr2 的 Presets 完整结构

### 源码总览（CMakePresets.json，156 行）

KrKr2 的 Presets 采用**继承体系**设计——用 hidden base presets 定义平台共性，visible presets 继承并添加 Debug/Release 差异：

```
CMakePresets.json
├── version: 6（需要 CMake 3.25+）
├── configurePresets
│   ├── [hidden] windows-base        ← Windows 平台公共配置
│   │   ├── Windows Debug Config     ← 继承 + Debug
│   │   └── Windows Release Config   ← 继承 + Release
│   ├── [hidden] mingw-base          ← MinGW 平台公共配置
│   │   ├── MinGW Debug Config       ← 继承 + Debug
│   │   └── MinGW Release Config     ← 继承 + Release
│   ├── [hidden] linux-base          ← Linux 平台公共配置
│   │   ├── Linux Debug Config       ← 继承 + Debug
│   │   └── Linux Release Config     ← 继承 + Release
│   └── [hidden] macos-base          ← macOS 平台公共配置
│       ├── MacOS Debug Config       ← 继承 + Debug
│       └── MacOS Release Config     ← 继承 + Release
└── buildPresets
    ├── Windows Debug Build → configurePreset: "Windows Debug Config"
    ├── Windows Release Build → configurePreset: "Windows Release Config"
    ├── ... (每个 configurePreset 对应一个 buildPreset)
    └── MacOS Release Build → configurePreset: "MacOS Release Config"
```

### Hidden Base Preset 详解

#### Windows Base

```json
{
    "name": "windows-base",
    "hidden": true,
    "generator": "Ninja",
    "cacheVariables": {
        "CMAKE_TOOLCHAIN_FILE": {
            "value": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake",
            "type": "FILEPATH"
        },
        "VCPKG_TARGET_TRIPLET": "x64-windows-static-md"
    },
    "condition": {
        "type": "equals",
        "lhs": "${hostSystemName}",
        "rhs": "Windows"
    }
}
```

逐字段分析：

| 字段 | 值 | 说明 |
|------|---|------|
| `name` | `"windows-base"` | 内部标识符，子 preset 通过 `inherits` 引用 |
| `hidden` | `true` | 不出现在 `cmake --list-presets` 输出中 |
| `generator` | `"Ninja"` | **所有平台统一使用 Ninja**，而非各平台默认生成器 |
| `CMAKE_TOOLCHAIN_FILE` | `$env{VCPKG_ROOT}/...` | vcpkg 工具链文件路径，`$env{}` 引用环境变量 |
| `VCPKG_TARGET_TRIPLET` | `x64-windows-static-md` | vcpkg 三元组：x64 架构、静态链接、动态 CRT |
| `condition` | `hostSystemName == "Windows"` | **仅在 Windows 主机上可见** |

**关键概念——`condition` 字段：**

`condition` 控制 preset 是否在当前主机上可用。它不影响交叉编译目标平台，只检查**执行 cmake 命令的机器**。

```json
{
    "type": "equals",
    "lhs": "${hostSystemName}",
    "rhs": "Windows"
}
```

- `${hostSystemName}` — CMake 宏，展开为当前操作系统名称
- 可能的值：`"Windows"`、`"Linux"`、`"Darwin"`（macOS）

效果：在 Linux 机器上运行 `cmake --list-presets` 时，不会看到 Windows 相关的 preset。

**`x64-windows-static-md` 三元组解析：**

| 组成部分 | 含义 |
|---------|------|
| `x64` | 目标架构：64 位 x86 |
| `windows` | 目标操作系统 |
| `static` | vcpkg 库**静态链接**（生成 `.lib`，不需要分发 DLL） |
| `md` | 使用**动态** C 运行时（`/MD` 编译选项），与 Windows 系统 DLL 共享 CRT |

> **为什么用 static 库 + 动态 CRT（md）？** 静态链接第三方库简化了分发（不需要带一堆 DLL），但使用动态 CRT 避免了 C 运行时冲突——如果不同模块使用不同版本的静态 CRT，会导致内存管理混乱（一个模块 malloc，另一个模块 free 会崩溃）。

#### Linux Base

```json
{
    "name": "linux-base",
    "hidden": true,
    "generator": "Ninja",
    "cacheVariables": {
        "CMAKE_TOOLCHAIN_FILE": {
            "value": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake",
            "type": "FILEPATH"
        }
    },
    "condition": {
        "type": "equals",
        "lhs": "${hostSystemName}",
        "rhs": "Linux"
    }
}
```

与 Windows 的关键差异：

| 差异点 | Windows | Linux | 原因 |
|--------|---------|-------|------|
| `VCPKG_TARGET_TRIPLET` | `x64-windows-static-md` | **未指定** | Linux 默认使用 `x64-linux`（动态链接），项目选择依赖系统默认行为 |
| condition | `hostSystemName == "Windows"` | `hostSystemName == "Linux"` | 平台检测 |

> **未指定 triplet 的含义：** 当不设置 `VCPKG_TARGET_TRIPLET` 时，vcpkg 会根据主机环境自动检测。在 x64 Linux 上，默认为 `x64-linux`（动态链接）。这意味着 Linux 构建需要将 `.so` 文件与可执行文件一起分发，或者安装到系统路径。

#### macOS Base

```json
{
    "name": "macos-base",
    "hidden": true,
    "generator": "Ninja",
    "cacheVariables": {
        "CMAKE_TOOLCHAIN_FILE": {
            "value": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake",
            "type": "FILEPATH"
        }
    },
    "condition": {
        "type": "equals",
        "lhs": "${hostSystemName}",
        "rhs": "Darwin"
    }
}
```

注意 macOS 的 `hostSystemName` 是 `"Darwin"`（Darwin 是 macOS 的内核名称），不是 `"macOS"` 或 `"Mac"`。这是一个常见的踩坑点。

#### MinGW Base

```json
{
    "name": "mingw-base",
    "hidden": true,
    "generator": "Ninja",
    "cacheVariables": {
        "CMAKE_TOOLCHAIN_FILE": {
            "value": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake",
            "type": "FILEPATH"
        },
        "VCPKG_TARGET_TRIPLET": "x64-mingw-static"
    },
    "condition": {
        "type": "equals",
        "lhs": "${hostSystemName}",
        "rhs": "Windows"
    }
}
```

MinGW 是在 Windows 上使用 GCC 工具链的方案。与 MSVC 的 `windows-base` 对比：

| 差异点 | MSVC (windows-base) | MinGW (mingw-base) |
|--------|--------------------|--------------------|
| 编译器 | MSVC (`cl.exe`) | GCC (`g++.exe`) |
| triplet | `x64-windows-static-md` | `x64-mingw-static` |
| CRT | 动态 `/MD` | 静态（MinGW 自带 CRT） |
| 生成的二进制 | 依赖 MSVC 运行时 | 自包含（可能依赖 `libstdc++`） |
| condition | `Windows` | `Windows`（同一主机可选两种） |

> **注意：** MinGW 和 MSVC 的 condition 都是 `Windows`，所以在 Windows 上 `cmake --list-presets` 会同时显示两组 preset。开发者根据安装的编译器选择使用哪组。

### Visible Presets（Debug/Release）

以 Windows 为例：

```json
{
    "name": "Windows Debug Config",
    "inherits": "windows-base",
    "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug"
    },
    "binaryDir": "${sourceDir}/out/windows/debug"
},
{
    "name": "Windows Release Config",
    "inherits": "windows-base",
    "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Release"
    },
    "binaryDir": "${sourceDir}/out/windows/release"
}
```

关键字段：

| 字段 | 说明 |
|------|------|
| `inherits` | 继承 `windows-base` 的所有配置（generator、toolchain、triplet、condition） |
| `CMAKE_BUILD_TYPE` | `Debug` 包含调试信息（`-g`/`/Zi`），`Release` 开启优化（`-O2`/`/O2`） |
| `binaryDir` | 构建产物输出目录，`${sourceDir}` 展开为项目根目录 |

输出目录结构：

```
krkr2/
└── out/
    ├── windows/
    │   ├── debug/      ← Windows Debug Config
    │   └── release/    ← Windows Release Config
    ├── linux/
    │   ├── debug/      ← Linux Debug Config
    │   └── release/    ← Linux Release Config
    ├── macos/
    │   ├── debug/      ← MacOS Debug Config
    │   └── release/    ← MacOS Release Config
    └── mingw/
        ├── debug/      ← MinGW Debug Config
        └── release/    ← MinGW Release Config
```

### Build Presets

```json
{
    "name": "Windows Debug Build",
    "configurePreset": "Windows Debug Config"
}
```

Build presets 非常简洁——它们只需引用对应的 configure preset。构建参数（生成器、输出目录等）从 configure preset 继承。

---

## Android：为什么没有 Preset？

你可能注意到 CMakePresets.json 中**没有 Android 的 preset**。这是因为 Android 构建通过 **Gradle + Android NDK** 驱动，而非直接调用 cmake 命令。

Android 的构建入口：

```bash
# 不是 cmake --preset "Android Debug Config"
# 而是：
./platforms/android/gradlew -p ./platforms/android assembleDebug
```

Gradle 会调用 CMake，但参数由 `app/build.gradle` 中的 `externalNativeBuild` 块控制（详见第 04 章）。

---

