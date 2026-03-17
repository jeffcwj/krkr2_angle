# 01-overlay-ports（覆盖/自定义端口）

> **所属模块：** P02-vcpkg包管理  
> **前置知识：** [../01-基础概念/01-什么是vcpkg与manifest模式.md](../01-基础概念/01-什么是vcpkg与manifest模式.md)、[../01-基础概念/03-vcpkg.json详解.md](../01-基础概念/03-vcpkg.json详解.md)  
> **预计阅读时间：** 30-40 分钟  
> **适用平台：** Windows / Linux / macOS / Android

## 本节目标

读完本节后，你将能够：

1. 解释 overlay-ports 的覆盖机制与查找优先级。
2. 理解 overlay-ports 的三类高频使用场景。
3. 写出一个最小可用 overlay-port。
4. 完成“复制→修改→测试”的覆盖流程。
5. 在 `vcpkg-configuration.json` 中配置 overlay-ports。
6. 对照 KrKr2 项目理解实际落地方式。

## overlay-ports 是什么

vcpkg 默认从官方 registry 解析端口。
overlay-ports 允许你在官方 registry 之前增加本地端口目录。

如果本地目录有同名端口，vcpkg 优先使用本地端口。
如果本地目录没有同名端口，vcpkg 回退到官方端口。

所以 overlay-ports 有两种作用：

- 覆盖官方端口。
- 补充官方缺失端口。

## 查找优先级

```text
vcpkg install <port>
  |
  +-- 查命令行 --overlay-ports
  +-- 查 vcpkg-configuration.json 的 overlay-ports
  |
  +-- overlay 命中同名端口 -> 使用 overlay
  +-- overlay 未命中 -> 使用官方 registry
```

结论：**overlay 同名端口优先级更高**。

## 使用场景

1. **补丁官方包**：当官方端口在某平台构建失败时，通过 overlay 加 patch 立即修复。
2. **添加私有包**：企业内部库、闭源库、特定许可证库放入 overlay 统一管理。
3. **固定特定版本**：固定 `REF` 与 `SHA512`，保证可重复构建。

## overlay-port 目录结构

最小结构：

```text
my-overlay-ports/
└── hello-world/
    ├── vcpkg.json
    └── portfile.cmake
```

常见工程结构：

```text
my-overlay-ports/
└── hello-world/
    ├── vcpkg.json
    ├── portfile.cmake
    ├── usage
    ├── CMakeLists.txt
    ├── hello-world-config.cmake.in
    └── 0001-fix-android.patch
```

## `vcpkg.json` 元数据块

```json
{
  "name": "hello-world",
  "version": "1.0.0",
  "description": "Overlay Port 示例",
  "dependencies": [
    "vcpkg-cmake",
    "vcpkg-cmake-config"
  ]
}
```

字段要点：

- `name` 必须与目录名一致。
- `dependencies` 通常包含 CMake 辅助端口。

## `portfile.cmake` 编写详解

### 源码获取

`vcpkg_from_github` 示例：

```cmake
vcpkg_from_github(
    OUT_SOURCE_PATH SOURCE_PATH
    REPO my-user/hello-world
    REF v1.0.0
    SHA512 0
    HEAD_REF main
)
```

`vcpkg_download_distfile` 示例：

```cmake
vcpkg_download_distfile(
    archive
    URLS https://example.com/foo-1.2.3.tar.gz
    FILENAME foo-1.2.3.tar.gz
    SHA512 1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
)
```

### 配置与安装

`vcpkg_cmake_configure` 示例：

```cmake
vcpkg_cmake_configure(
    SOURCE_PATH "${SOURCE_PATH}"
    OPTIONS
        -DBUILD_TESTING=OFF
)
```

`vcpkg_cmake_install` 示例：

```cmake
vcpkg_cmake_install()
```

`vcpkg_cmake_config_fixup` 示例：

```cmake
vcpkg_cmake_config_fixup(PACKAGE_NAME hello-world)
```

### 收尾步骤

```cmake
vcpkg_copy_pdbs()
vcpkg_install_copyright(FILE_LIST "${SOURCE_PATH}/LICENSE")
```

## 从零创建 overlay-port

### 第 1 步：目录

```text
krkr2/vcpkg/ports/demo-hello/
├── vcpkg.json
└── portfile.cmake
```

### 第 2 步：`vcpkg.json`

```json
{
  "name": "demo-hello",
  "version": "1.0.0",
  "description": "用于学习 overlay-port 的示例",
  "dependencies": [
    "vcpkg-cmake",
    "vcpkg-cmake-config"
  ]
}
```

### 第 3 步：`portfile.cmake`

```cmake
vcpkg_from_github(
    OUT_SOURCE_PATH SOURCE_PATH
    REPO my-org/demo-hello
    REF v1.0.0
    SHA512 9c6b8f1121a6cc42f9f0f2ab4d4bf8af47c1eb11f97d31b4ad2106f0d10d7f3ac81adf8d0a53f1bc2f68f9be4a80e0f1b0508ed0d861aa8b09da0ea86b95dd77
)

vcpkg_cmake_configure(SOURCE_PATH "${SOURCE_PATH}")
vcpkg_cmake_install()
vcpkg_cmake_config_fixup(PACKAGE_NAME demo-hello)
vcpkg_copy_pdbs()
vcpkg_install_copyright(FILE_LIST "${SOURCE_PATH}/LICENSE")
```

### 第 4 步：安装验证

```bash
vcpkg install demo-hello --overlay-ports=./vcpkg/ports
```

### 第 5 步：工程验证

```cmake
find_package(demo-hello CONFIG REQUIRED)
target_link_libraries(my_app PRIVATE demo-hello::demo-hello)
```

## 修改现有端口：复制→修改→测试

### 复制

复制官方同名端口到 overlay 目录。

### 修改

添加 patch 并在源码获取函数挂载：

```cmake
vcpkg_from_github(
    OUT_SOURCE_PATH SOURCE_PATH
    REPO pnggroup/libpng
    REF v1.6.43
    SHA512 2a394f1b8c8af58e0df7e960f7ac8c5cbacbc1120f91834f4179c453fbd25b4ee4ad6f48d5f3c6db46f33a8ef6a66a219a4db4c79d1ac4cf8e21f7400f8d8f22
    PATCHES
        fix-android.patch
)
```

### 测试

```bash
vcpkg install libpng:x64-windows --overlay-ports=./vcpkg/ports
vcpkg install libpng:x64-linux --overlay-ports=./vcpkg/ports
vcpkg install libpng:arm64-osx --overlay-ports=./vcpkg/ports
vcpkg install libpng:arm64-android --overlay-ports=./vcpkg/ports
```

## 在 `vcpkg-configuration.json` 中配置 overlay-ports

项目级配置示例：

```json
{
  "overlay-ports": [
    "./vcpkg/ports",
    "../external/team-ports"
  ]
}
```

KrKr2 实际配置（`krkr2/vcpkg-configuration.json`）如下：

```json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/vcpkg-tool/main/docs/vcpkg-configuration.schema.json",
  "default-registry": {
    "kind": "builtin",
    "baseline": "b1e15efef6758eaa0beb0a8732cfa66f6a68a81d"
  },
  "overlay-ports": [
    "./vcpkg/ports"
  ],
  "overlay-triplets": [
    "./vcpkg/triplets"
  ]
}
```

## KrKr2 overlay-ports 实战分析

项目里没有 `krkr2/overlay-ports/`，实际使用目录是 `krkr2/vcpkg/ports/`。

已存在端口：

1. `7zip`
2. `cocos2dx`
3. `ffmpeg`
4. `libarchive`
5. `libgdiplus`
6. `unrar`

观察点：`7zip` 带 Android 补丁，`unrar` 使用 distfile 下载流程，`ffmpeg` 含大量平台分支，`libgdiplus` 与 `cocos2dx` 含多个 patch。

## 跨平台注意事项

### Windows

- 常见工具链为 MSVC。
- 部分包依赖 NASM、pkg-config。

### Linux

- 可能与系统预装头文件冲突。
- 建议优先使用 vcpkg 依赖闭环。

### macOS

- 区分 `arm64` 与 `x64`。
- 检查 Deployment Target 与 SDK。

### Android

- 对齐 triplet、NDK、API Level。
- 避免交叉编译参数错配。

## 常见错误与排查

### 错误 1：`port not found`

排查顺序：

1. overlay 路径是否正确。
2. 目录名与 `name` 是否一致。
3. 命令执行目录是否正确。

### 错误 2：缺少版权文件

排查顺序：

1. 是否安装了版权文件。
2. 许可证路径是否存在。

### 错误 3：`find_package` 失败

排查顺序：

1. 是否调用 `vcpkg_cmake_config_fixup()`。
2. `share/<port>/` 是否有配置文件。
3. 包名与命名空间是否匹配。

## 动手实践

目标：完成一个最小 overlay-port，并验证覆盖行为。

步骤：

1. 在 `krkr2/vcpkg/ports` 新建 `demo-overlay`。
2. 编写 `vcpkg.json` 与 `portfile.cmake`。
3. 执行 `vcpkg install demo-overlay`。
4. 覆盖一个官方端口并增加 patch。
5. 运行目标 triplet 测试并记录结果。

## 本节小结

- overlay-ports 用于覆盖和补充官方端口。
- 场景包括修补、私有包、版本锁定。
- 核心文件为 `vcpkg.json` 与 `portfile.cmake`。
- 覆盖流程是“复制→修改→测试”。
- KrKr2 通过 `vcpkg/ports` 做项目级 overlay。

## 练习题与答案

### 题目 1：优先级

如果同名端口同时存在于官方 registry 与 overlay，`vcpkg install` 会选哪个？

<details>
<summary>查看答案</summary>

优先选择 overlay 端口。
只有 overlay 未命中时才使用官方端口。

</details>

### 题目 2：最小模板

最小 overlay-port 至少要哪些文件？

<details>
<summary>查看答案</summary>

至少需要两个文件：

1. `vcpkg.json`
2. `portfile.cmake`

</details>

### 题目 3：覆盖流程

写出覆盖官方端口并加补丁的完整步骤。

<details>
<summary>查看答案</summary>

步骤如下：

1. 复制官方端口到 overlay（保持同名）。
2. 添加补丁文件。
3. 在源码获取函数中配置 `PATCHES`。
4. 运行目标 triplet 安装测试。
5. 在业务工程验证 `find_package` 与链接。

</details>

## 下一步

继续阅读：[02-自定义triplet.md](./02-自定义triplet.md)。
下一节将讲解 triplet 如何与 overlay-port 组合控制依赖构建行为。
