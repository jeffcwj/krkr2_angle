# 02-overlay-ports与自定义triplet

> **所属模块：** P02-vcpkg包管理  
> **前置知识：** [01-vcpkg-configuration与baseline](./01-vcpkg-configuration与baseline.md)  
> **预计阅读时间：** 35 分钟

## 元数据块

- **文档类型：** 实战分析
- **章节定位：** P02 第4章第2节（章末与模块末衔接）
- **核心范围：** `vcpkg/ports`、`vcpkg/triplets`、`cmake/vcpkg_android.cmake`
- **本节产出：** 能解释并修改 KrKr2 的 overlay 依赖配置

## 本节目标

读完本节后，你将能够：

1. 说明 KrKr2 为什么需要 overlay-ports。
2. 逐个分析 6 个自定义 port 的构建逻辑。
3. 理解 Android triplet 与动态库白名单机制。
4. 理解 `vcpkg_android.cmake` 与 triplet 的协同关系。
5. 完成修改现有 port 的实战流程。

## 一、入口配置

KrKr2 使用如下 vcpkg 配置：

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

这三项分别控制：

- registry 快照版本
- 本地端口覆盖
- 本地 triplet 覆盖

## 二、为什么要自定义 ports

### 1) 官方端口无法保证项目四平台稳定

KrKr2 同时支持 Windows、Linux、macOS、Android。
跨平台构建稳定性必须由项目自行兜底。

### 2) 官方默认构建选项不符合项目目标

例如 ffmpeg：

- 项目要库，不要命令行工具
- Android 要开启 `jni`、`mediacodec`

### 3) 真实兼容问题必须以 patch 形式沉淀

项目中的 patch 文件名很直接：

- `fix_timespec_get_broken_on_android.patch`
- `0001-android-ffmpeg.patch`
- `fix-win64.patch`

### 4) 有些依赖没有现成官方 port

`cocos2dx` 就是典型，自建 overlay 才能统一纳管。

## 三、逐个分析 `vcpkg/ports/` 下的端口

当前共有 6 个：

1. `7zip`
2. `cocos2dx`
3. `ffmpeg`
4. `libarchive`
5. `libgdiplus`
6. `unrar`

## 3.1 7zip

### 目录结构

- `portfile.cmake`
- `vcpkg.json`
- `CMakeLists.txt`
- `7zip-config.cmake.in`
- `fix_timespec_get_broken_on_android.patch`

### `portfile.cmake` 解读

- 从 `ip7z/7zip` 拉源码
- 应用 Android patch
- 注入自定义 CMake 配置
- 标准安装并清理 debug include

### patch 作用

关键条件：

```cpp
#if defined(TIME_UTC) && (!defined(__ANDROID__) || __ANDROID_API__ >= 29)
```

目标是规避 Android 低 API 的时间函数兼容问题。

## 3.2 cocos2dx

### 目录结构

- `portfile.cmake`
- `vcpkg.json`
- `DownloadDeps.cmake`
- `cocos2dx-config.cmake.in`
- `patch/` 大量补丁

### `portfile.cmake` 解读

- 下载固定版本源码包
- 一次应用多平台补丁
- 覆盖上游 CMake 构建脚本
- 关闭测试与 JS/Lua 组件

### 代表 patch

- `fix-win64.patch`：Win64 API 指针修复
- `fix-unzip.patch`：minizip 头与内存接口修复
- `fix-chipmunk-Hasty.patch`：物理步进路径统一

## 3.3 ffmpeg

### 目录结构

- `portfile.cmake`（593 行）
- `vcpkg.json`
- `build.sh.in`
- `FindFFMPEG.cmake.in`
- `vcpkg-cmake-wrapper.cmake`
- 3 个 patch

### `portfile.cmake` 解读

- 按平台拼接 configure 选项
- 裁剪 `ffmpeg/ffplay/ffprobe` 程序
- 保留引擎必需库
- release/debug 分离构建
- 安装后修正 pkg-config 与 CMake 包配置

### patch 作用

- `0001-android-ffmpeg.patch`：Android ioctl 参数兼容
- `0001-operand-shr-error.patch`：x86 汇编约束修复
- `0001-fixed-mac.patch`：mac aarch64 参数类型修复

## 3.4 libarchive

### 目录结构

- `portfile.cmake`
- `vcpkg.json`
- `usage`

### `portfile.cmake` 解读

- 关闭 OpenSSL 与测试
- 安装后清理 debug/include/share/man
- 替换头文件宏表达式，稳定静态链接行为

## 3.5 libgdiplus

### 目录结构

- `portfile.cmake`
- `vcpkg.json`
- `CMakeLists.txt`
- `config.h.in`
- `usage`
- 3 个 patch

### `portfile.cmake` 解读

- 拉取源码并应用 mac/quartz/linux 补丁
- 注入自定义 CMake 配置
- Linux 使用 X11
- Android/macOS 关闭 pango

### patch 作用

- Quartz API 与类型冲突修复
- Linux pango/png 参数兼容修复

## 3.6 unrar

### 目录结构

- `portfile.cmake`
- `vcpkg.json`
- `CMakeLists.txt`
- `unrar-config.cmake.in`
- `usage`
- `0001-fix-mac.patch`

### `portfile.cmake` 解读

- 从 RARLab 下载源码包
- 应用 mac 补丁
- 注入 CMake 构建与 config 模板
- 安装后清理 debug/include

### patch 作用

- 处理 mac 与其他 Unix 的 `stat` 时间字段差异

## 四、Triplet 使用分析

`vcpkg/triplets/` 包含：

- `arm64-android.cmake`
- `arm-android.cmake`
- `x64-android.cmake`
- `x86-android.cmake`
- `android-dynamic-libs.cmake`

四个 ABI triplet 共同点：

- static CRT
- static library linkage
- Android system name
- API 级别 28

## 五、Android 特殊配置

`android-dynamic-libs.cmake`：

```cmake
set(DYNAMIC_LIBRARIES sdl2)

if(PORT IN_LIST DYNAMIC_LIBRARIES)
    set(VCPKG_LIBRARY_LINKAGE dynamic)
endif ()
```

意义：

- 全局静态策略不变
- `sdl2` 按端口名单独动态化

## 六、`vcpkg_android.cmake` 联动机制

脚本做 5 件事：

1. 校验 `ANDROID_NDK_HOME`
2. 校验 `VCPKG_ROOT`
3. 按 `ANDROID_ABI` 映射 triplet
4. 设置 vcpkg toolchain
5. 链入 NDK toolchain

ABI 映射：

- `arm64-v8a -> arm64-android`
- `armeabi-v7a -> arm-android`
- `x86_64 -> x64-android`
- `x86 -> x86-android`

## 七、overlay-ports + overlay-triplets 组合

执行路径：

1. CMake 启动 vcpkg
2. vcpkg 读取配置
3. 加载 overlay triplet
4. 加载 overlay port
5. 以 triplet 参数执行 portfile

组合价值：

- 端口逻辑可控
- ABI 策略可控
- 构建结果可复现

## 八、修改现有 port 实战流程

### 步骤 1

定义问题类型：编译错误或功能开关。

### 步骤 2

选择入口：改 portfile 或新增 patch。

### 步骤 3

补丁命名明确，一补丁一问题。

### 步骤 4

清缓存并重建目标 triplet。

### 步骤 5

检查 installed 目录产物是否符合预期。

### 步骤 6

完成四平台最小回归。

## 九、常见问题与调试

### 问题 1：改动不生效

- 检查 `overlay-ports` 路径
- 检查端口名一致性
- 清缓存重装

### 问题 2：Android triplet 错误

- 检查 `ANDROID_ABI`
- 检查映射分支
- 检查日志输出

### 问题 3：静态/动态混用异常

- 检查 triplet 默认链接
- 检查动态库白名单
- 检查 `PORT` 名称匹配

## 动手实践

### 实践 1：7zip Android patch

1. 阅读 patch 条件
2. 解释 API 级别影响
3. 重新安装 Android 依赖

### 实践 2：SDL2 动态白名单

1. 阅读 `android-dynamic-libs.cmake`
2. 验证 `sdl2` 动态链接

### 实践 3：cocos2dx Win64 修复

1. 阅读 `fix-win64.patch`
2. 标记 `LongPtr` 相关改动

## 对照项目源码

- `vcpkg-configuration.json` 第 1-13 行
- `vcpkg.json` 第 1-94 行
- `vcpkg/ports/7zip/portfile.cmake` 第 1-25 行
- `vcpkg/ports/7zip/fix_timespec_get_broken_on_android.patch` 第 1-13 行
- `vcpkg/ports/cocos2dx/portfile.cmake` 第 1-67 行
- `vcpkg/ports/cocos2dx/patch/fix-win64.patch` 第 1-91 行
- `vcpkg/ports/ffmpeg/portfile.cmake` 第 1-593 行
- `vcpkg/ports/ffmpeg/0001-android-ffmpeg.patch` 第 1-13 行
- `vcpkg/ports/libarchive/portfile.cmake` 第 1-33 行
- `vcpkg/ports/libgdiplus/portfile.cmake` 第 1-39 行
- `vcpkg/ports/unrar/portfile.cmake` 第 1-29 行
- `vcpkg/triplets/android-dynamic-libs.cmake` 第 1-5 行
- `cmake/vcpkg_android.cmake` 第 20-99 行

## 本节小结

- overlay-ports 解决端口与补丁可控问题。
- overlay-triplets 解决 ABI 与链接策略可控问题。
- KrKr2 通过两者组合实现跨平台稳定构建。

## 练习题与答案

### 题目 1

为什么 KrKr2 需要 overlay-ports，而不是只用官方端口。

<details>
<summary>查看答案</summary>

因为官方端口无法保证项目在 Windows、Linux、macOS、Android 同时稳定。  
KrKr2 还需要项目级开关和补丁，例如 ffmpeg 裁剪和 Android/macOS 兼容修复。  
这些都需要通过 overlay-ports 固化。

</details>

### 题目 2

`android-dynamic-libs.cmake` 为什么只维护少量白名单端口。

<details>
<summary>查看答案</summary>

项目默认策略是静态链接，便于部署和控制依赖。  
只有确实需要动态链接的端口才进入白名单。  
这样能在减少复杂度的同时满足平台特殊需求。

</details>

### 题目 3

`vcpkg_android.cmake` 中 ABI 到 triplet 的映射有什么价值。

<details>
<summary>查看答案</summary>

它把 Android 构建参数统一收敛为可控 triplet。  
不同 ABI 会自动映射到对应 triplet，避免手工选择错误。  
这保证了依赖构建参数和目标 ABI 一致。

</details>

### 题目 4

`unrar` 的 mac 补丁说明了哪类典型跨平台问题。

<details>
<summary>查看答案</summary>

说明“同为 Unix 也可能有结构体字段差异”。  
macOS 与其他 Unix 的 `stat` 时间字段命名不同。  
如果不做条件分支，就会出现编译错误。

</details>

## 下一步

下一模块进入 **M02-项目构建系统深度解析**。  
建议先阅读根目录 `CMakeLists.txt` 与 `CMakePresets.json`。
