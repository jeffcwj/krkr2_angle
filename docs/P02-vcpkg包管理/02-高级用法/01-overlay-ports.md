# 01-Overlay Ports (覆盖/自定义端口)

在 vcpkg 的基础用法中，我们通常使用官方仓库提供的库。但在实际开发中，我们经常会遇到官方库版本太旧、存在 Bug 或者官方仓库根本没有某个库的情况。这时，我们就需要用到 vcpkg 的高级特性：**Overlay Ports (覆盖端口)**。

## 什么是 Overlay Ports

Overlay Ports 允许你定义一个本地目录，其中包含你自己编写或修改的库定义（Port）。当你请求安装一个库时，vcpkg 会优先在你的 Overlay Ports 目录中查找，如果找到了，就使用你的定义，否则再去官方仓库中查找。

这就像是给 vcpkg 装上了一个“插件系统”，你可以随时替换、修改或新增库的安装脚本。

## 为什么需要 Overlay Ports

在 KrKr2 项目中，Overlay Ports 是不可或缺的，主要原因包括：

1.  **官方包有 Bug**：有些库在特定平台（如 Android 或 macOS）上编译失败，我们需要手动添加补丁。
2.  **自定义配置**：官方默认可能只编译动态库，而我们需要静态库，或者需要开启某些特定的 CMake 选项。
3.  **官方缺失的包**：有些比较冷门或专有的库（如 KrKr2 用到的 `unrar`）官方并没有提供。
4.  **版本锁定**：官方仓库升级后，某些 API 可能发生变化导致项目编译失败。通过 Overlay Ports，我们可以将库锁定在特定版本。

## Overlay Port 的目录结构

一个 Overlay Port 实际上就是一个文件夹，其名称必须与库的名称一致。每个文件夹内至少包含两个文件：

-   `vcpkg.json`：定义库的元数据（名称、版本、依赖等）。
-   `portfile.cmake`：具体的安装脚本。

例如，一个名为 `my-lib` 的自定义端口结构如下：

```text
my-overlay-ports/
└── my-lib/
    ├── vcpkg.json
    ├── portfile.cmake
    └── usage (可选)
```

## 如何创建一个 Overlay Port

让我们以创建一个简单的 `hello-world` 库为例。

### 第一步：准备 vcpkg.json

在 `my-overlay-ports/hello-world/` 目录下创建 `vcpkg.json`：

```json
{
  "name": "hello-world",
  "version": "1.0.0",
  "description": "一个简单的 Overlay Port 示例",
  "dependencies": [
    "vcpkg-cmake",
    "vcpkg-cmake-config"
  ]
}
```

### 第二步：编写 portfile.cmake

`portfile.cmake` 是一个 CMake 脚本，告诉 vcpkg 如何下载、编译和安装。

```cmake
# 获取源代码（假设代码在 GitHub 上）
vcpkg_from_github(
    OUT_SOURCE_PATH SOURCE_PATH
    REPO my-user/hello-world
    REF v1.0.0
    SHA512 0  # 第一次运行可以填 0，vcpkg 会提示正确值
    HEAD_REF main
)

# 配置 CMake 选项
vcpkg_cmake_configure(
    SOURCE_PATH "${SOURCE_PATH}"
)

# 编译并安装
vcpkg_cmake_install()

# 自动处理 CMake 配置文件的导出路径
vcpkg_cmake_config_fixup(PACKAGE_NAME hello-world)

# 安装版权信息（vcpkg 要求必须安装 copyright 文件）
file(INSTALL "${SOURCE_PATH}/LICENSE" DESTINATION "${CURRENT_PACKAGES_DIR}/share/hello-world" RENAME copyright)
```

## portfile.cmake 常用函数详解

在编写 `portfile.cmake` 时，你会频繁使用以下工具函数：

### 1. 源码获取
- `vcpkg_from_github`: 从 GitHub 下载。
- `vcpkg_from_git`: 从 Git 仓库克隆。
- `vcpkg_download_distfile`: 直接下载单个压缩包。

### 2. 编译流程
- `vcpkg_cmake_configure`: 包装了 `cmake -S ... -B ...`，自动传入 vcpkg 的 triplet 配置。
- `vcpkg_cmake_install`: 运行 `cmake --build --target install`。
- `vcpkg_cmake_config_fixup`: 修复安装后的 `lib/cmake` 路径，使其符合 vcpkg 标准。

### 3. 文件处理
- `file(INSTALL ...)`: 标准 CMake 指令，用于将文件复制到 `${CURRENT_PACKAGES_DIR}`。
- `vcpkg_copy_pdbs()`: 自动处理 Windows 下的符号文件。

## 如何启用 Overlay Ports

你可以在命令行中直接指定路径：

```bash
vcpkg install my-lib --overlay-ports=./my-overlay-ports
```

但在 KrKr2 等大型项目中，我们通常通过 `vcpkg-configuration.json` 进行全局配置：

```json
{
  "overlay-ports": [
    "./vcpkg-ports",
    "../external/my-ports"
  ]
}
```

配置后，vcpkg 在执行任何操作（如 `install`、`remove`、`upgrade`）时，都会自动搜索这些路径下的 Port 定义。

## KrKr2 项目中的 Overlay Ports 实例

在 KrKr2 项目中，我们维护了 6 个自定义的 Port，它们解决了许多跨平台编译的顽疾：

1.  **7zip**：官方库只有源码，没有直接可用的 CMake 脚本。我们编写了自定义 Port，使其支持所有平台（Windows, Linux, macOS, Android）。
2.  **cocos2dx**：KrKr2 使用了 Cocos2d-x 的某些组件，官方并没有提供。通过 Overlay Port，我们将 cocos2dx 整合进了 vcpkg。
3.  **ffmpeg**：ffmpeg 在不同平台上的依赖项差异巨大。在 Windows 上我们需要汇编器（nasm），在 Android 上我们需要交叉编译器。自定义 Port 处理了这些复杂的依赖。
4.  **libarchive**：我们为 libarchive 添加了自定义补丁，以便在 Android 下更好地处理特定格式的解压。
5.  **libgdiplus**：官方版在 Linux 和 macOS 上的依赖处理不尽如人意。通过 Overlay Port，我们统一了它的编译方式。
6.  **unrar**：由于 unrar 的许可证比较特殊且源码包结构非标准，我们编写了自定义安装脚本。

## 跨平台注意事项

当你在不同平台上测试 Overlay Ports 时，请务必关注以下内容：

-   **Windows**: 默认使用 MSVC。如果使用 Clang-cl 或 MinGW，请确保编译标志匹配。
-   **Linux**: 注意 `apt-get` 预安装的开发头文件可能会影响编译。建议尽量使用 vcpkg 内置的依赖。
-   **macOS**: 如果是 Apple Silicon (M1/M2)，请注意架构标志。
-   **Android**: 必须在 `portfile.cmake` 中通过变量 `ANDROID`（vcpkg 设置的）来区分。例如：

```cmake
if(VCPKG_TARGET_IS_ANDROID)
    message(STATUS "正在为 Android 编译...")
    # 这里可以添加 Android 特有的 CMake 参数
endif()
```

## 练习题与答案

### 题目 1：Overlay Ports 的查找优先级

如果有两个同名的库（例如 `zlib`），一个在 vcpkg 官方仓库，一个在你配置的 Overlay Ports 目录中。当你执行 `vcpkg install zlib` 时，哪一个会被安装？

<details>
<summary>查看答案</summary>

Overlay Ports 中的库会被优先安装。vcpkg 会首先遍历配置的所有 Overlay 路径，如果找到匹配的库名，就会停止查找并使用该定义；只有在所有 Overlay 路径中都找不到时，才会去官方仓库查找。

</details>

### 题目 2：portfile.cmake 的核心职责

在编写一个 Overlay Port 时，`portfile.cmake` 脚本至少需要完成哪三件事才能让库成功安装并可用？

<details>
<summary>查看答案</summary>

1.  **获取源码**：通过 `vcpkg_from_github` 或类似函数下载源码。
2.  **编译与安装**：通过 `vcpkg_cmake_configure` 和 `vcpkg_cmake_install` 将编译产物放入指定目录。
3.  **安装版权信息**：必须将许可证文件安装到 `share/${PORT}/copyright` 路径，否则 vcpkg 会报错。

</details>

### 题目 3：修改现有库的补丁

如果你发现官方的 `libpng` 在 Android 28 平台上编译报错，需要应用一个名为 `fix-android.patch` 的文件，你该如何在 Overlay Ports 中实现？

<details>
<summary>查看答案</summary>

1.  将官方 `libpng` 目录的所有内容复制到你的 Overlay Ports 目录下。
2.  将 `fix-android.patch` 放入该目录。
3.  在 `portfile.cmake` 的 `vcpkg_from_github` 函数中，添加 `PATCHES fix-android.patch` 参数。
4.  这样当你下次安装 `libpng` 时，vcpkg 会自动应用该补丁。

</details>
