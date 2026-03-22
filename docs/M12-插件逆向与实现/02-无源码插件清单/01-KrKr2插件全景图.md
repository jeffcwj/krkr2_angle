# KrKr2 插件全景图

> **所属模块：** M12-插件逆向与实现
> **前置知识：** [ncbind 注册框架详解](../01-KiriKiri插件接口分析/03-ncbind注册框架详解.md)、[M09-插件系统与开发](../../M09-插件系统与开发/README.md)
> **预计阅读时间：** 25 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：
1. 列举 KrKr2 项目中所有已集成的插件，说出每个插件的功能和注册方式
2. 区分"项目内有源码"、"外部有源码"、"完全无源码"三类插件
3. 解释 `krkr2plugin` 静态库的 CMake 构建结构和链接关系
4. 识别哪些标记为 `nan`（无源码）的插件实际上已有实现，纠正插件清单中的不准确信息
5. 根据插件分类表，判断某个游戏缺失的 DLL 对应哪类插件、应采取何种处理策略

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 静态链接插件 | Statically Linked Plugin | 编译时直接合并进主程序的插件，运行时不需要单独的 DLL 文件 |
| 动态链接插件 | Dynamically Linked Plugin | 运行时从独立 DLL 文件加载的插件，原版 KiriKiri 使用这种方式 |
| ncbind | NativeClass Binder | KiriKiri 的插件注册框架，通过宏把 C++ 类/函数绑定到 TJS 脚本引擎 |
| simplebinder | Simple Binder | KrKr2 项目中另一套轻量级 TJS 绑定方案，基于模板实现 |
| krkr2plugin | KrKr2 Plugin Library | CMake 构建目标名，包含所有插件的静态库，链接进最终可执行文件 |
| 插件清单 | Plugin Manifest | `doc/krkr2_plugins.md` 文件，记录了原版 KiriKiri 游戏中常见的所有 DLL 插件及其源码位置 |
| PE 导出表 | PE Export Table | Windows DLL 文件中记录对外公开函数名和地址的数据结构，逆向分析的入口 |

---

## 一、为什么需要插件全景图

### 1.1 KrKr2 的插件策略：从动态加载到静态链接

原版 KiriKiri2 引擎运行在 Windows 上，采用**动态链接插件**（Dynamically Linked Plugin）架构——每个插件编译为独立的 `.dll` 文件，引擎在运行时通过 `Plugins.link("xxx.dll")` 加载。这种设计的优点是灵活（游戏开发者可以按需选择插件），缺点是依赖 Windows 的 DLL 加载机制（`LoadLibrary` / `GetProcAddress`），无法直接移植到 Android、Linux 和 macOS。

KrKr2 作为跨平台模拟器，采取了不同的策略：**静态链接插件**（Statically Linked Plugin）。所有插件在编译期就合并进 `krkr2plugin` 静态库，再链接到最终的可执行文件（或 Android 上的 `.so` 共享库）。当 TJS 脚本调用 `Plugins.link("xxx.dll")` 时，KrKr2 不再从文件系统加载 DLL，而是在内部插件注册表中查找已编译的插件：

```cpp
// cpp/core/plugin/ncbind.cpp 第 3-29 行（简化）
// LoadModule 是 KrKr2 处理 Plugins.link() 的核心函数
// 它在 _internal_plugins 映射表中查找插件名
static int LoadModule(const tjs_char* name) {
    ttstr modname(name);
    // 从 _internal_plugins（std::map）中查找插件
    auto it = _internal_plugins.find(modname);
    if (it == _internal_plugins.end()) {
        // 没找到——这个 DLL 还没有被实现
        TVPAddLog(ttstr("Plugin not found: ") + modname);
        return TJS_E_FAIL;
    }
    // 找到了——按 3 阶段执行注册回调
    auto& lists = it->second;
    for (int phase = 0; phase < 3; phase++) {
        for (auto& reg : lists[phase]) {
            reg->Execute();  // 执行注册
        }
    }
    return TJS_S_OK;
}
```

这意味着：**KrKr2 能运行哪些游戏，取决于它实现了哪些插件。** 如果某个游戏调用 `Plugins.link("TextRender.dll")`，而 KrKr2 中没有 TextRender 的实现，游戏就会报错或功能缺失。

### 1.2 插件全景图的用途

建立一份完整的插件全景图有三个目的：

1. **评估兼容性**：拿到一个新游戏时，提取其 `startup.tjs` 中所有 `Plugins.link()` 调用，对照全景图即可判断 KrKr2 能否运行
2. **确定逆向优先级**：无源码插件中，哪些被最多游戏使用？哪些功能最关键？优先逆向高优先级插件
3. **防止重复劳动**：有些插件在 `doc/krkr2_plugins.md` 中标记为 `nan`（无源码），但实际上项目中已有实现。全景图纠正这些不准确信息

下图展示了 KrKr2 的插件从源码到运行的完整流程：

```
┌─────────────────────────────────────────────────────────────────┐
│                      KrKr2 插件构建流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  源码层                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ 单文件插件    │  │ 子目录插件    │  │  无源码插件   │          │
│  │ scriptsEx.cpp│  │ psdfile/     │  │ PackinOne.dll│          │
│  │ csvParser.cpp│  │ psbfile/     │  │ TextRender   │          │
│  │ dirlist.cpp  │  │ motionplayer/│  │ (需要逆向)   │          │
│  │ ...（16个）   │  │ layerex_draw/│  │              │          │
│  └──────┬───────┘  │ fstat/       │  └──────────────┘          │
│         │          └──────┬───────┘                             │
│         ▼                 ▼                                      │
│  ┌─────────────────────────────────┐                            │
│  │   krkr2plugin（STATIC 库）       │                            │
│  │   target_sources + subdirectory │                            │
│  └──────────────┬──────────────────┘                            │
│                 ▼                                                │
│  ┌─────────────────────────────────┐                            │
│  │   krkr2core（INTERFACE 库）      │                            │
│  │   tjs2 + base + visual + ...    │                            │
│  └──────────────┬──────────────────┘                            │
│                 ▼                                                │
│  ┌─────────────────────────────────┐                            │
│  │   krkr2（最终可执行文件/.so）     │                            │
│  └─────────────────────────────────┘                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、krkr2plugin 的 CMake 构建结构

在深入每个插件之前，先理解它们是怎么被组织和编译的。`krkr2plugin` 的 CMake 配置在 `cpp/plugins/CMakeLists.txt`（共 74 行），结构清晰：

### 2.1 构建目标定义

```cmake
# cpp/plugins/CMakeLists.txt 第 1-4 行
cmake_minimum_required(VERSION 3.19)
project(krkr2plugin)

# 关键：STATIC 库——所有插件编译后合并为一个 .a/.lib 文件
add_library(${PROJECT_NAME} STATIC)
```

`STATIC` 关键字意味着所有插件代码会被编译成目标文件（`.o` / `.obj`），打包进一个静态库文件（Linux/macOS 上是 `libkrkr2plugin.a`，Windows 上是 `krkr2plugin.lib`）。链接器在生成最终可执行文件时，会把其中被引用的代码段合并进去。

### 2.2 单文件插件（16 个）

```cmake
# cpp/plugins/CMakeLists.txt 第 6-23 行
target_sources(${PROJECT_NAME} PUBLIC
    scriptsEx.cpp       # TJS 脚本扩展函数（Debug.message 等）
    win32dialog.cpp     # Win32 对话框桥接
    dirlist.cpp         # 目录列表功能
    csvParser.cpp       # CSV 文件解析器
    layerExMovie.cpp    # 图层视频叠加
    varfile.cpp         # 变量文件读写
    saveStruct.cpp      # 存档结构序列化
    getabout.cpp        # 系统信息获取
    addFont.cpp         # 自定义字体加载
    wutcwf.cpp          # 文件写入工具
    windowEx.cpp        # 窗口扩展功能
    getSample.cpp       # 音频采样获取
    layerExPerspective.cpp  # 透视变换图层
    fftgraph.cpp        # FFT 频谱图
    LayerExBase.cpp     # 图层扩展基类
    xp3filter.cpp       # XP3 归档加密过滤器
)
```

这 16 个 `.cpp` 文件直接列在 `target_sources` 中，是"单文件插件"——每个文件自包含所有逻辑，不需要额外的头文件或子目录。它们通过 ncbind 宏在编译期自动注册到 `_internal_plugins` 映射表。

### 2.3 子目录插件（5 个）

```cmake
# cpp/plugins/CMakeLists.txt 第 52-68 行
add_subdirectory(psdfile)       # PSD 文件解析器（独立子库）
add_subdirectory(psbfile)       # PSB/E-mote 格式解析器
add_subdirectory(layerex_draw)  # 扩展绘图图层（GDI+/通用）
add_subdirectory(motionplayer)  # E-mote 动画播放器
add_subdirectory(fstat)         # 文件状态查询

# 以下三个已被注释掉——依赖未满足，不要启用
#add_subdirectory(json)
#add_subdirectory(steam)
#add_subdirectory(DrawDeviceForSteam)

# 子目录插件各自编译为独立静态库，再链接进 krkr2plugin
target_link_libraries(${PROJECT_NAME} PUBLIC
    psdfile         # PSD 解析
    psbfile         # PSB 解析
    layerExDraw     # 扩展绘图
    motionplayer    # 动画播放
    fstat           # 文件状态
)
```

子目录插件的特点是代码量大、结构复杂，需要独立的 CMakeLists.txt。例如 `psdfile/` 内含 `psdparse/` 子库，总计 11 个 `.cpp` 文件；`psbfile/` 含 `types/` 和 `resources/` 子目录，共 13 个 `.cpp` 文件。

### 2.4 被禁用的插件（3 个）

CMake 中有三个被注释掉的插件：

| 插件目录 | 功能 | 禁用原因 |
|---------|------|---------|
| `json/` | JSON 解析（json.dll） | 依赖未配置，vcpkg.json 中无对应包 |
| `steam/` | Steam 平台集成（steam_api.dll） | 需要 Steamworks SDK，非开源 |
| `DrawDeviceForSteam/` | Steam 专用渲染设备 | 依赖 steam 插件 |

> **注意：** 不要随意取消注释这些插件。它们的依赖需要先在 `vcpkg.json` 中添加并测试通过，否则编译会失败。

---

## 三、插件三分类全景表

根据源码可用性，KrKr2 涉及的所有插件可分为三类。下面逐类列出，每个插件标注 DLL 名称、功能、注册方式和源码位置。

### 3.1 A 类：项目内有源码（已编译进 krkr2plugin）

这些插件的源码在 `cpp/plugins/` 目录下，已经编译进 `krkr2plugin` 静态库，是 KrKr2 当前直接支持的插件。

#### 单文件插件（16 个）

| DLL 名称 | 源码文件 | 功能说明 | 注册方式 | 行数 |
|----------|---------|---------|---------|------|
| scriptsEx.dll | `scriptsEx.cpp` | 为 TJS 的 Scripts 对象添加 `execStorage`、`evalStorage` 等方法，扩展脚本执行能力 | NCB_ATTACH_CLASS | 993 |
| win32dialog.dll | `win32dialog.cpp` | 桥接 Win32 对话框 API（MessageBox、文件选择等）到 TJS 脚本 | NCB_PRE_REGIST_CALLBACK | — |
| dirlist.dll | `dirlist.cpp` | 提供目录遍历功能，返回指定路径下的文件/子目录列表 | NCB_ATTACH_FUNCTION | — |
| csvParser.dll | `csvParser.cpp` | 解析 CSV 格式文件，支持自定义分隔符、引号转义 | NCB_PRE_REGIST_CALLBACK | 466 |
| layerExMovie.dll | `layerExMovie.cpp` | 在图层上叠加播放视频，实现视频与游戏画面的混合 | NCB_ATTACH_CLASS | — |
| varfile.dll | `varfile.cpp` | 将 TJS 变量（字典/数组）序列化到文件，实现持久化存储 | NCB_ATTACH_FUNCTION | — |
| saveStruct.dll | `saveStruct.cpp` | 存档数据的结构化序列化/反序列化，支持嵌套数据类型 | NCB_ATTACH_FUNCTION | — |
| getabout.dll | `getabout.cpp` | 获取系统信息（OS 版本、内存等），注册为 `System.getAbout` | NCB_ATTACH_FUNCTION | 5 |
| addFont.dll | `addFont.cpp` | 运行时加载自定义字体文件（TTF/OTF），供文字渲染使用 | NCB_ATTACH_FUNCTION | — |
| wutcwf.dll | `wutcwf.cpp` | 文件写入工具，提供带缓冲的文件输出功能 | NCB_ATTACH_FUNCTION | — |
| windowEx.dll | `windowEx.cpp` | 扩展 Window 类功能（全屏切换、窗口置顶、截图等） | NCB_ATTACH_CLASS | — |
| getSample.dll | `getSample.cpp` | 获取音频波形采样数据，用于音频可视化 | NCB_ATTACH_FUNCTION | — |
| layerExPerspective.dll | `layerExPerspective.cpp` | 图层透视变换（四点自由变换），实现 3D 透视效果 | NCB_ATTACH_CLASS | — |
| fftgraph.dll | `fftgraph.cpp` | 快速傅里叶变换（FFT，一种将时域信号转为频域的数学算法）频谱图绘制 | NCB_ATTACH_CLASS | — |
| LayerExBase.cpp | `LayerExBase.cpp` | 图层扩展基类，为其他 layerEx 系列插件提供公共方法 | NCB_ATTACH_CLASS | — |
| xp3filter.dll | `xp3filter.cpp` | XP3 归档加密过滤器，处理游戏资源包的加解密 | NCB_POST_REGIST_CALLBACK | 568 |

> **重要发现：** `getabout.dll` 和 `xp3filter.dll` 在 `doc/krkr2_plugins.md` 插件清单中被标记为 `nan`（无源码），但实际上项目中已经有完整实现。这是插件清单的不准确信息。

#### 子目录插件（5 个启用 + 3 个禁用）

| DLL 名称 | 源码目录 | 功能说明 | 文件数 | 状态 |
|----------|---------|---------|--------|------|
| PSBFile.dll | `psbfile/` | PSB（E-mote 场景包）格式解析器，支持图像/动画/音频等多种资源类型 | 13 个 .cpp | ✅ 启用 |
| psdfile（无DLL对应） | `psdfile/` | PSD（Photoshop 文件）解析器，支持图层/通道/资源读取 | 11 个 .cpp | ✅ 启用 |
| LayerExDraw.dll | `layerex_draw/` | 扩展绘图图层，windows/ 使用 GDI+，general/ 使用跨平台实现 | 5 个 .cpp | ✅ 启用 |
| motionplayer.dll | `motionplayer/` | E-mote 动画播放器，驱动 Live2D 风格的角色动态表情 | 4 个 .cpp | ✅ 启用 |
| fstat.dll | `fstat/` | 文件状态查询（大小、修改时间、是否存在等） | 1 个 .cpp | ✅ 启用 |
| json.dll | `json/` | JSON 格式解析/生成 | 1 个 .cpp | ❌ 禁用 |
| steam_api（无DLL对应） | `steam/` | Steam 平台集成（成就、云存储等） | 2 个 .cpp | ❌ 禁用 |
| DrawDeviceForSteam（无DLL对应） | `DrawDeviceForSteam/` | Steam 专用渲染设备 | 1 个 .cpp | ❌ 禁用 |

> **注意：** `motionplayer.dll` 在插件清单中也被标记为 `nan`（无源码），但项目中有完整的 4 文件实现。这是另一处不准确信息。

#### A 类小结

项目内共有 **24 个插件**（16 单文件 + 5 子目录启用 + 3 子目录禁用），其中 21 个已编译进 krkr2plugin。这些插件覆盖了脚本扩展、文件 I/O、图层特效、音视频、存档等核心功能。

### 3.2 B 类：外部有源码（可移植但尚未集成）

这些插件在 `doc/krkr2_plugins.md` 中标注了 GitHub 或外部下载链接，源码可公开获取，但尚未被移植到 KrKr2 项目中。它们是"随时可以集成"的插件——只需下载源码、适配 ncbind 注册、添加到 CMakeLists.txt 即可。

| DLL 名称 | 源码位置 | 功能说明 | 集成难度 |
|----------|---------|---------|---------|
| AlphaMovie.dll | [kaede-software.com](http://kaede-software.com/krlm/plugin/alphamovie.zip) | 带透明通道的视频播放（Alpha Movie，在视频画面上叠加半透明效果） | 中等——需要对接 KrKr2 的 FFmpeg 播放器 |
| DrawDeviceD3D.dll | [krkrz/krkr2](https://github.com/krkrz/krkr2/tree/master/kirikiri2/trunk/kirikiri2/src/plugins/win32/drawdeviceD3D) | Direct3D 渲染设备，原版 KiriKiri 的 D3D 后端 | 高——KrKr2 使用 OpenGL，需要整体替换 |
| DrawDeviceD3DZ.dll | [krkrz/krkrz](https://github.com/krkrz/krkrz/tree/last_hodgepodge_repository/src/plugins/win32/drawdeviceD3D) | KiriKiriZ 版本的 D3D 渲染设备 | 高——同上 |
| ExtKAGParser.dll | [keepcreating](http://keepcreating.g2.xrea.com/krkrplugins/ExtKAGParser/ExtKAGParser-0143.zip) | 扩展 KAG 解析器，增强场景脚本解析能力 | 低——项目中已有 KAGParser/ 目录可参考 |
| KAGParserEx.dll | [wtnbgo/KAGParserEx](https://github.com/wtnbgo/KAGParserEx) | 另一种 KAG 解析器扩展 | 低 |
| KAGParserExb.dll | [sakano/krkr_archives](https://github.com/sakano/krkr_archives/tree/master/kagex_plugin/KAGParserExb) | KAG 扩展解析器变体 | 低 |
| KaichoTrans.dll | [keepcreating](http://keepcreating.g2.xrea.com/krkrplugins/KaichoTrans/KaichoTrans.zip) | 界面切换过渡效果（页面转场动画） | 中等 |
| LayerExImage.dll | [wtnbgo/layerExImage](https://github.com/wtnbgo/layerExImage) | 图层图像扩展操作（缩放、旋转、色彩调整） | 低 |
| LayerExSave.dll | [wtnbgo/layerExSave](https://github.com/wtnbgo/layerExSave) | 图层内容导出为图片文件（PNG/BMP） | 低 |
| csvParser.dll | [wtnbgo/csvParser](https://github.com/wtnbgo/csvParser) | CSV 解析器（项目中已有实现，此为原版参考） | 已集成 |
| expat.dll | [krkrz/krkrz](https://github.com/krkrz/krkrz/tree/last_hodgepodge_repository/src/plugins/win32/expat) | Expat XML 解析库的 TJS 绑定 | 中等——需要添加 expat 到 vcpkg |
| extrans.dll | [krkrz/SamplePlugin](https://github.com/krkrz/SamplePlugin/tree/master/extrans) | 扩展场景过渡效果集合 | 低 |
| fftgraph.dll | [krkrz/fftgraph](https://github.com/krkrz/fftgraph) | FFT 频谱可视化（项目中已有实现） | 已集成 |
| savestruct.dll | [wtnbgo/saveStruct](https://github.com/wtnbgo/saveStruct) | 存档序列化（项目中已有实现） | 已集成 |
| fstat.dll | [wtnbgo/fstat](https://github.com/wtnbgo/fstat) | 文件状态查询（项目中已有实现） | 已集成 |
| getSample.dll | [wtnbgo/getSample](https://github.com/wtnbgo/getSample) | 音频采样（项目中已有实现） | 已集成 |
| layerExAreaAverage.dll | [wtnbgo/layerExAreaAverage](https://github.com/wtnbgo/layerExAreaAverage) | 图层区域平均色计算 | 低 |
| layerExBTOA.dll | [wtnbgo/layerExBTOA](https://github.com/wtnbgo/layerExBTOA) | 图层亮度到透明度转换（BToA = Brightness To Alpha） | 低 |
| layerExMovie.dll | [krkrz/krkrz](https://github.com/krkrz/krkrz/tree/last_hodgepodge_repository/src/plugins/win32/layerExMovie) | 图层视频叠加（项目中已有实现） | 已集成 |
| layerExRaster.dll | [wtnbgo/layerExRaster](https://github.com/wtnbgo/layerExRaster) | 图层光栅化效果（水波纹、扭曲等） | 中等 |
| layerExShimmer.dll | [keepcreating](http://keepcreating.g2.xrea.com/krkrplugins/ShimmerPlugin/layerExShimmer.zip) | 图层热气流/闪烁效果 | 中等 |
| minizip.dll | [wtnbgo/minizip](https://github.com/wtnbgo/minizip) | ZIP 压缩/解压（项目 external/ 中已有 minizip） | 已集成 |
| perspective.dll | [krkrz/krkrz](https://github.com/krkrz/krkrz/tree/last_hodgepodge_repository/src/plugins/win32/layerExPerspective) | 透视变换（项目中已有实现） | 已集成 |
| scriptsEx.dll | [wtnbgo/scriptsEx](https://github.com/wtnbgo/scriptsEx) | 脚本扩展（项目中已有实现） | 已集成 |
| shrinkCopy.dll | [wtnbgo/shrinkCopy](https://github.com/wtnbgo/shrinkCopy) | 图层缩小拷贝（高质量下采样） | 低 |
| sqlite3.dll | [krkrz/krkrz](https://github.com/krkrz/krkrz/tree/last_hodgepodge_repository/src/plugins/win32/sqlite3) | SQLite3 数据库的 TJS 绑定 | 中等——需要添加 sqlite3 到 vcpkg |
| varfile.dll | [wtnbgo/varfile](https://github.com/wtnbgo/varfile) | 变量文件（项目中已有实现） | 已集成 |
| win32dialog.dll | [wtnbgo/win32dialog](https://github.com/wtnbgo/win32dialog) | 对话框（项目中已有实现） | 已集成 |
| windowEx.dll | [wtnbgo/windowEx](https://github.com/wtnbgo/windowEx) | 窗口扩展（项目中已有实现） | 已集成 |
| wutcwf.dll | [krkrz/SamplePlugin](https://github.com/krkrz/SamplePlugin/tree/master/wutcwf) | 文件写入工具（项目中已有实现） | 已集成 |
| addFont.dll | [wtnbgo/addFont](https://github.com/wtnbgo/addFont) | 字体加载（项目中已有实现） | 已集成 |
| gfxEffect.dll | [kaede-software.com](http://kaede-software.com/krlm/plugin/gfx_effect.zip) | 图形特效集（粒子、光晕等） | 中等 |
| extNagano.dll | [web.archive.org](https://web.archive.org/web/20120604091809fw_/http://ymtkyk.sakura.ne.jp/krkr.STG/plugin/extNagano.html) | 长野扩展——射击游戏（STG）辅助功能 | 高——专用性强 |

#### B 类小结

B 类共 **31 个插件**条目，其中 **13 个已在项目中集成**（标记"已集成"），真正需要新集成的约 **18 个**。集成难度从低（纯 TJS 绑定、无外部依赖）到高（需要 D3D→OpenGL 移植）不等。

### 3.3 C 类：完全无源码（需要逆向工程）

这些插件在 `doc/krkr2_plugins.md` 中标记为 `nan`，且项目中确实没有对应实现。它们只能通过逆向工程原始 DLL 来还原功能。

| DLL 名称 | 功能说明 | 使用频率 | 逆向难度 |
|----------|---------|---------|---------|
| PackinOne.dll | 归档压缩工具——将多个文件打包为单个归档文件，可能使用自定义压缩算法 | 中等——部分游戏用于资源打包 | 中等——核心是压缩算法还原 |
| TextRender.dll | 文字渲染引擎——提供高级文字排版、描边、阴影、Ruby 注音（汉字上方标注读音的小字）等功能 | 高——大量文字类游戏依赖 | 高——涉及字体光栅化和 GPU 渲染 |
| kirikiroid2.dll | KiriKiriDroid2 核心运行时——这是 Android 版 KiriKiri 的核心模块 | 特殊——仅部分移植版游戏使用 | 高——功能庞大 |
| layerExAlpha.dll | 图层透明度扩展——提供高级 Alpha 混合模式（预乘 Alpha、色相混合等） | 中等 | 低——功能相对单一 |
| lzfs.dll | LZ 压缩文件系统——提供 LZ 系列算法（一类基于滑动窗口的无损压缩算法）的文件系统访问 | 低——少数游戏使用 | 中等——需要识别具体 LZ 变体 |
| multiimage.dll | 多图像处理——同时操作多张图片（合成、比较、批量变换等） | 低 | 中等 |
| util_generic.dll | 通用工具集——提供字符串处理、数学计算、数组操作等辅助函数 | 高——很多游戏的基础工具库 | 低——函数多但每个函数逻辑简单 |
| util_graph.dll | 图形工具集——提供像素级图像操作（色彩转换、滤镜、直方图等） | 中等 | 中等——涉及图像处理算法 |
| util_system.dll | 系统工具集——提供系统信息查询、进程管理、注册表访问等功能 | 中等 | 低——大部分是系统 API 封装 |

> **特别说明：** `kirikiroid2.dll` 比较特殊。它是 KiriKiriDroid2（另一个 Android KiriKiri 模拟器项目）的核心模块，不是标准的 KiriKiri 插件。KrKr2 项目本身就是一个完整的 KiriKiri 模拟器，理论上不需要这个 DLL。只有当某些游戏经过 KiriKiriDroid2 修改后才会调用它。

#### C 类小结

真正需要逆向的插件共 **9 个**，但扣除 `kirikiroid2.dll`（特殊情况），实际需要逆向的是 **8 个**。按逆向难度和使用频率，后续章节将重点实战 PackinOne 和 TextRender。

### 3.4 清单勘误表

在梳理全景图的过程中，我们发现 `doc/krkr2_plugins.md` 存在若干不准确的标记。以下是需要更正的条目：

| DLL 名称 | 清单标记 | 实际状态 | 项目内源码位置 |
|----------|---------|---------|--------------|
| getabout.dll | `nan`（无源码） | ✅ 已有完整实现 | `cpp/plugins/getabout.cpp`（5 行） |
| xp3filter.dll | `nan`（无源码） | ✅ 已有完整实现 | `cpp/plugins/xp3filter.cpp`（568 行） |
| motionplayer.dll | `nan`（无源码） | ✅ 已有完整实现 | `cpp/plugins/motionplayer/`（4 文件） |
| PSBFile.dll | `M2 Inc. PSB Library` | ✅ 已有完整实现 | `cpp/plugins/psbfile/`（13 文件） |
| emoteplayer.dll | `M2 Inc.` | ⚠️ 由 motionplayer 提供功能 | `cpp/plugins/motionplayer/` |

> 这些勘误说明：**不能完全依赖插件清单来判断某个 DLL 是否已实现。** 在处理新游戏的兼容性问题时，应先在 `cpp/plugins/` 中搜索对应的源码文件或 ncbind 注册宏。

---

## 四、插件统计汇总

### 4.1 数量统计

```
┌─────────────────────────────────────────────────────────┐
│                    KrKr2 插件统计                         │
├──────────────────────┬──────────────────────────────────┤
│ A 类（项目内有源码）  │ 24 个（21 启用 + 3 禁用）         │
│ B 类（外部有源码）    │ 18 个（可移植但未集成）           │
│ C 类（完全无源码）    │ 9 个（需要逆向工程）              │
├──────────────────────┼──────────────────────────────────┤
│ 合计                 │ 51 个独立插件                      │
│ 已可用               │ 21 个（41%）                      │
│ 可快速集成           │ ~10 个低难度 B 类（20%）           │
│ 需要逆向             │ 8 个（排除 kirikiroid2.dll）       │
└──────────────────────┴──────────────────────────────────┘
```

### 4.2 按注册方式统计

通过 ncbind 宏注册的插件分布：

| 注册宏 | 使用数量 | 典型插件 |
|--------|---------|---------|
| NCB_ATTACH_CLASS | 6 | scriptsEx、windowEx、layerExMovie、layerExPerspective、fftgraph、LayerExBase |
| NCB_ATTACH_FUNCTION | 7 | dirlist、varfile、saveStruct、getabout、addFont、wutcwf、getSample |
| NCB_PRE_REGIST_CALLBACK | 2 | csvParser、win32dialog |
| NCB_POST_REGIST_CALLBACK | 1 | xp3filter |
| NCB_REGISTER_CLASS | 1 | steam_api（已禁用） |
| 子目录（独立注册） | 5 | psdfile、psbfile、layerex_draw、motionplayer、fstat |

> NCB_ATTACH_CLASS 和 NCB_ATTACH_FUNCTION 合计占 13/16 个单文件插件，是最常用的两种注册方式。这与 M12-01 章的分析一致：大多数插件是往已有 TJS 类上"挂载"方法，而非创建全新的类。

---

## 五、实战：查询插件实现状态

下面通过具体的代码示例，演示如何判断一个 DLL 是否已在 KrKr2 中实现。

### 示例 1：检查单文件插件是否存在

当拿到一个游戏的 `startup.tjs`，看到 `Plugins.link("csvParser.dll")`，如何确认 KrKr2 是否支持？

```cpp
// 方法 1：在 CMakeLists.txt 中搜索文件名
// 打开 cpp/plugins/CMakeLists.txt，查找 "csvParser"
// 如果在 target_sources 中找到 csvParser.cpp —— 已实现

// 方法 2：在源码中搜索 NCB_MODULE_NAME
// 每个 ncbind 插件必须定义 NCB_MODULE_NAME 宏
// 搜索 "csvParser" 作为模块名

// cpp/plugins/csvParser.cpp 第 1-3 行
#define NCB_MODULE_NAME TJS_W("csvParser.dll")
// ↑ 这就是注册名——与 Plugins.link() 的参数完全匹配

// 当 TJS 脚本执行 Plugins.link("csvParser.dll") 时
// LoadModule 会在 _internal_plugins 中查找 "csvParser.dll"
// 找到后执行注册回调，完成插件加载
```

### 示例 2：检查子目录插件

```cpp
// 子目录插件的 NCB_MODULE_NAME 定义在主入口文件中
// 例如 psbfile/ 的入口文件 main.cpp：

// cpp/plugins/psbfile/main.cpp（简化）
#define NCB_MODULE_NAME TJS_W("PSBFile.dll")
// ↑ 注意大小写：DLL 名是 "PSBFile.dll"
// TJS 脚本中必须写 Plugins.link("PSBFile.dll")

// motionplayer/ 的入口文件：
// cpp/plugins/motionplayer/main.cpp
#define NCB_MODULE_NAME TJS_W("motionplayer.dll")
```

### 示例 3：编写查找脚本

在项目根目录下运行以下命令，可以快速列出所有已注册的插件名：

```bash
# Windows (PowerShell)
# 搜索所有 NCB_MODULE_NAME 定义，提取 DLL 名称
findstr /r /s "NCB_MODULE_NAME" cpp\plugins\*.cpp cpp\plugins\*\*.cpp

# Linux / macOS
grep -r "NCB_MODULE_NAME" cpp/plugins/ --include="*.cpp"

# 示例输出：
# cpp/plugins/csvParser.cpp:    #define NCB_MODULE_NAME TJS_W("csvParser.dll")
# cpp/plugins/scriptsEx.cpp:    #define NCB_MODULE_NAME TJS_W("scriptsEx.dll")
# cpp/plugins/getabout.cpp:     #define NCB_MODULE_NAME TJS_W("getabout.dll")
# cpp/plugins/xp3filter.cpp:    #define NCB_MODULE_NAME TJS_W("xp3filter.dll")
# cpp/plugins/psbfile/main.cpp: #define NCB_MODULE_NAME TJS_W("PSBFile.dll")
# ... 等等
```

### 示例 4：检查游戏兼容性

```cpp
// 假设你拿到一个游戏，startup.tjs 中有以下 Plugins.link 调用：
// Plugins.link("csvParser.dll");    → A 类，已实现 ✅
// Plugins.link("scriptsEx.dll");    → A 类，已实现 ✅
// Plugins.link("TextRender.dll");   → C 类，无源码 ❌
// Plugins.link("PackinOne.dll");    → C 类，无源码 ❌
// Plugins.link("layerExBTOA.dll");  → B 类，有外部源码可移植 ⚠️

// 兼容性判断：
// - TextRender 和 PackinOne 都是 C 类 → 需要逆向
// - layerExBTOA 是 B 类 → 可以从 GitHub 下载源码移植
// - 结论：该游戏目前无法完整运行，需要先处理上述 3 个插件

// 实际在 KrKr2 代码中，未实现的插件会走到这个分支：
// cpp/core/plugin/ncbind.cpp LoadModule 函数
static int LoadModule(const tjs_char* name) {
    ttstr modname(name);
    auto it = _internal_plugins.find(modname);
    if (it == _internal_plugins.end()) {
        // 走到这里 = 插件未实现
        // KrKr2 会在日志中输出警告
        TVPAddLog(ttstr("Plugin not found: ") + modname);
        return TJS_E_FAIL;
        // TJS 脚本会捕获到这个错误
        // 游戏可能弹出错误对话框或直接退出
    }
    // ... 正常注册流程
}
```

### 示例 5：遍历所有已注册插件

```cpp
// 以下代码展示如何在运行时列出所有已注册的插件
// 可以添加到调试模式中，帮助开发者了解当前构建包含哪些插件

#include <iostream>
#include <map>
#include <string>
#include <list>

// 模拟 _internal_plugins 结构
// 实际定义在 ncbind.hpp 的匿名命名空间中
struct MockAutoRegister {
    std::string name;      // 注册器名称
    int phase;             // 注册阶段（0/1/2）
};

// 简化版：遍历插件注册表
void ListAllPlugins() {
    // 实际代码中 _internal_plugins 是 std::map<ttstr, INTERNAL_PLUGIN_LISTS>
    // INTERNAL_PLUGIN_LISTS 是 std::list<ncbAutoRegister*>[3]（3 个阶段）
    
    // 模拟遍历
    std::map<std::string, int> plugins = {
        {"csvParser.dll", 3},      // 3 个注册回调
        {"scriptsEx.dll", 2},      // 2 个注册回调
        {"getabout.dll", 1},       // 1 个注册回调
        {"xp3filter.dll", 1},      // 1 个注册回调
        {"PSBFile.dll", 5},        // 5 个注册回调
    };
    
    std::cout << "=== KrKr2 已注册插件列表 ===" << std::endl;
    std::cout << "插件名称\t\t注册回调数" << std::endl;
    std::cout << "------------------------------------" << std::endl;
    
    int total = 0;
    for (const auto& [name, count] : plugins) {
        std::cout << name << "\t\t" << count << std::endl;
        total++;
    }
    
    std::cout << "------------------------------------" << std::endl;
    std::cout << "共 " << total << " 个插件已注册" << std::endl;
}

int main() {
    ListAllPlugins();
    return 0;
}

// 输出：
// === KrKr2 已注册插件列表 ===
// 插件名称              注册回调数
// ------------------------------------
// PSBFile.dll            5
// csvParser.dll          3
// getabout.dll           1
// scriptsEx.dll          2
// xp3filter.dll          1
// ------------------------------------
// 共 5 个插件已注册
```

---

## 动手实践

### 练习 1：从游戏脚本提取插件需求

假设你拿到一个 KiriKiri 游戏的 `startup.tjs` 文件，内容如下：

```javascript
// startup.tjs —— 游戏启动脚本
Plugins.link("csvParser.dll");
Plugins.link("saveStruct.dll");
Plugins.link("windowEx.dll");
Plugins.link("TextRender.dll");
Plugins.link("layerExRaster.dll");
Plugins.link("util_generic.dll");
```

**步骤 1：** 逐个对照全景图，填写以下表格：

| 插件 | 分类 | 状态 | 处理方案 |
|------|------|------|---------|
| csvParser.dll | A 类 | ✅ 已实现 | 无需操作 |
| saveStruct.dll | ? | ? | ? |
| windowEx.dll | ? | ? | ? |
| TextRender.dll | ? | ? | ? |
| layerExRaster.dll | ? | ? | ? |
| util_generic.dll | ? | ? | ? |

**步骤 2：** 统计结果——该游戏有几个插件已实现、几个需要移植、几个需要逆向？

**步骤 3：** 根据统计结果，判断该游戏在当前版本的 KrKr2 上能否运行。如果不能，列出需要优先处理的插件。

> **参考答案见文末练习题。**

### 练习 2：添加一个 B 类插件的占位符

当某个 B 类插件尚未移植，但你希望游戏不因为 `Plugins.link()` 失败而崩溃时，可以创建一个**占位插件**（Stub Plugin）：

```cpp
// 文件：cpp/plugins/layerExRaster_stub.cpp
// 功能：layerExRaster.dll 的占位实现
//       让 Plugins.link("layerExRaster.dll") 成功
//       但所有方法都是空操作（no-op）

// 第一步：定义模块名——必须与游戏中调用的 DLL 名一致
#define NCB_MODULE_NAME TJS_W("layerExRaster.dll")

// 第二步：包含 ncbind 头文件
#include "ncbind.hpp"

// 第三步：用最简单的注册方式——空的 POST 回调
// 这确保 LoadModule 能找到 "layerExRaster.dll"
// 但不注册任何方法（游戏调用相关方法时会静默失败）
NCB_POST_REGIST_CALLBACK(stub_layerExRaster) {
    // 空实现——仅让 Plugins.link 成功
    // 可以添加一条日志提示开发者
    TVPAddLog(TJS_W("layerExRaster.dll: stub loaded (not implemented)"));
}
```

然后在 CMakeLists.txt 中添加：

```cmake
# cpp/plugins/CMakeLists.txt
target_sources(${PROJECT_NAME} PUBLIC
    # ... 原有文件 ...
    layerExRaster_stub.cpp   # 占位插件
)
```

### 练习 3：编写插件兼容性检查工具

```cpp
// 文件：tools/check_plugin_compat.cpp
// 功能：读取游戏的 startup.tjs，提取所有 Plugins.link() 调用
//       对照已知插件列表，输出兼容性报告

#include <iostream>
#include <fstream>
#include <string>
#include <vector>
#include <set>
#include <regex>

// 已实现的插件列表（A 类 + 已集成的 B 类）
const std::set<std::string> IMPLEMENTED_PLUGINS = {
    "scriptsEx.dll", "win32dialog.dll", "dirlist.dll",
    "csvParser.dll", "layerExMovie.dll", "varfile.dll",
    "saveStruct.dll", "getabout.dll", "addFont.dll",
    "wutcwf.dll", "windowEx.dll", "getSample.dll",
    "layerExPerspective.dll", "fftgraph.dll",
    "LayerExBase.dll", "xp3filter.dll",
    "PSBFile.dll", "motionplayer.dll", "fstat.dll",
    "LayerExDraw.dll"
};

// 有外部源码的插件（B 类未集成）
const std::set<std::string> EXTERNAL_SOURCE_PLUGINS = {
    "AlphaMovie.dll", "ExtKAGParser.dll", "KAGParserEx.dll",
    "layerExAreaAverage.dll", "layerExBTOA.dll",
    "layerExRaster.dll", "layerExShimmer.dll",
    "shrinkCopy.dll", "sqlite3.dll", "expat.dll"
};

// 从 TJS 脚本中提取 Plugins.link() 调用
std::vector<std::string> ExtractPluginLinks(
        const std::string& filepath) {
    std::vector<std::string> plugins;
    std::ifstream file(filepath);
    std::string line;
    // 匹配 Plugins.link("xxx.dll") 格式
    std::regex pattern(
        R"(Plugins\.link\s*\(\s*\"([^\"]+)\"\s*\))");
    
    while (std::getline(file, line)) {
        std::smatch match;
        if (std::regex_search(line, match, pattern)) {
            plugins.push_back(match[1].str());
        }
    }
    return plugins;
}

int main(int argc, char* argv[]) {
    if (argc < 2) {
        std::cerr << "用法: check_plugin_compat <startup.tjs路径>"
                  << std::endl;
        return 1;
    }
    
    auto plugins = ExtractPluginLinks(argv[1]);
    
    int ok = 0, portable = 0, missing = 0;
    
    std::cout << "=== 插件兼容性报告 ===" << std::endl;
    for (const auto& plugin : plugins) {
        if (IMPLEMENTED_PLUGINS.count(plugin)) {
            std::cout << "  ✅ " << plugin << " — 已实现"
                      << std::endl;
            ok++;
        } else if (EXTERNAL_SOURCE_PLUGINS.count(plugin)) {
            std::cout << "  ⚠️ " << plugin << " — 有源码可移植"
                      << std::endl;
            portable++;
        } else {
            std::cout << "  ❌ " << plugin << " — 无源码/需逆向"
                      << std::endl;
            missing++;
        }
    }
    
    std::cout << "\n--- 统计 ---" << std::endl;
    std::cout << "已实现: " << ok << std::endl;
    std::cout << "可移植: " << portable << std::endl;
    std::cout << "缺失:   " << missing << std::endl;
    std::cout << "兼容性: "
              << (missing == 0 ? "可运行" : "不可运行")
              << std::endl;
    
    return missing > 0 ? 1 : 0;
}

// 编译: g++ -std=c++17 -o check_plugin_compat check_plugin_compat.cpp
// 运行: ./check_plugin_compat path/to/startup.tjs
```

---

## 对照项目源码

以下是本节涉及的项目源码文件汇总：

| 文件路径 | 行号 | 内容说明 |
|---------|------|---------|
| `cpp/plugins/CMakeLists.txt` | 1-74 | krkr2plugin 完整构建配置 |
| `cpp/plugins/CMakeLists.txt` | 6-23 | 16 个单文件插件列表 |
| `cpp/plugins/CMakeLists.txt` | 52-59 | 5 个子目录插件 + 3 个禁用插件 |
| `cpp/plugins/CMakeLists.txt` | 61-68 | 子目录插件的链接配置 |
| `cpp/core/plugin/ncbind.cpp` | 3-29 | LoadModule 函数——插件查找和注册入口 |
| `cpp/core/plugin/ncbind.hpp` | 2100-2382 | 所有 NCB 宏定义 |
| `cpp/plugins/getabout.cpp` | 1-5 | 最简单的 ncbind 插件（5 行） |
| `cpp/plugins/xp3filter.cpp` | 545-568 | NCB_POST_REGIST_CALLBACK 用法 |
| `cpp/plugins/csvParser.cpp` | 425-466 | NCB_PRE_REGIST_CALLBACK 用法 |
| `cpp/plugins/scriptsEx.cpp` | 960-993 | NCB_ATTACH_CLASS 用法 |
| `doc/krkr2_plugins.md` | 1-54 | 原始插件清单（含不准确标记） |

---

## 常见错误与排查

### 错误 1：NCB_MODULE_NAME 大小写不匹配

```
// 症状：Plugins.link("psbfile.dll") 失败
// 原因：模块名大小写不匹配
// PSBFile.dll ≠ psbfile.dll

// 排查：查看插件源码中的 NCB_MODULE_NAME 定义
// cpp/plugins/psbfile/main.cpp:
#define NCB_MODULE_NAME TJS_W("PSBFile.dll")  // 注意大写

// 修复：在 TJS 脚本中使用正确的大小写
Plugins.link("PSBFile.dll");  // ✅ 正确
Plugins.link("psbfile.dll");  // ❌ 找不到
```

> **要点：** `_internal_plugins` 是 `std::map<ttstr, ...>`，`ttstr` 的比较是**大小写敏感**的。DLL 名称必须与 NCB_MODULE_NAME 定义完全一致。

### 错误 2：占位插件忘记添加到 CMakeLists.txt

```
// 症状：编写了 stub 文件但编译后仍找不到插件
// 原因：只创建了 .cpp 文件，忘记在 CMakeLists.txt 中添加

// 排查步骤：
// 1. 确认 .cpp 文件在 cpp/plugins/ 目录下
// 2. 打开 cpp/plugins/CMakeLists.txt
// 3. 在 target_sources 中添加文件名
// 4. 重新运行 cmake 配置（不只是编译！）

// cmake 配置命令（Windows）：
// cmake --preset="Windows Debug Config"
// cmake --build --preset="Windows Debug Build"
```

### 错误 3：误判插件实现状态

```
// 症状：在 doc/krkr2_plugins.md 中看到 "nan" 就认为未实现
// 实际：getabout.dll、xp3filter.dll、motionplayer.dll 都已有实现

// 正确的判断流程：
// 1. 先搜索 cpp/plugins/ 下的源码文件
// 2. 搜索 NCB_MODULE_NAME 定义
// 3. 最后才参考 doc/krkr2_plugins.md
// 因为源码文件是"地面真相"，而清单可能过时
```

---

## 本节小结

- KrKr2 采用**静态链接插件**架构，所有插件编译进 `krkr2plugin` 静态库，运行时通过 `LoadModule` 在内部注册表中查找
- 项目内有 **24 个插件**（21 启用 + 3 禁用），涵盖脚本扩展、文件 I/O、图层特效、音视频等功能
- 外部有源码的插件 **31 个**，其中 13 个已集成，18 个可移植
- 完全无源码的插件 **9 个**（实际需要逆向的 8 个），是本模块后续章节的重点
- `doc/krkr2_plugins.md` 存在至少 **5 处不准确标记**（getabout、xp3filter、motionplayer、PSBFile、emoteplayer），判断插件实现状态应以源码为准
- 插件名称是**大小写敏感**的，`Plugins.link()` 的参数必须与 `NCB_MODULE_NAME` 定义完全匹配
- 评估游戏兼容性的正确流程：提取 `Plugins.link()` 调用 → 对照三分类表 → 统计缺失 → 制定处理方案

---

## 练习题与答案

### 题目 1：游戏兼容性分析

一个 KiriKiri 游戏的 `startup.tjs` 中包含以下插件加载语句：

```javascript
Plugins.link("csvParser.dll");
Plugins.link("saveStruct.dll");
Plugins.link("windowEx.dll");
Plugins.link("TextRender.dll");
Plugins.link("layerExRaster.dll");
Plugins.link("util_generic.dll");
```

请回答：
1. 每个插件属于哪个分类（A/B/C）？
2. 该游戏目前能否在 KrKr2 上运行？
3. 如果不能，应按什么顺序处理缺失的插件？

<details>
<summary>查看答案</summary>

| 插件 | 分类 | 状态 | 处理方案 |
|------|------|------|---------|
| csvParser.dll | A 类 | ✅ 已实现 | 无需操作 |
| saveStruct.dll | A 类 | ✅ 已实现 | 无需操作 |
| windowEx.dll | A 类 | ✅ 已实现 | 无需操作 |
| TextRender.dll | C 类 | ❌ 无源码 | 需要逆向 |
| layerExRaster.dll | B 类 | ⚠️ 有外部源码 | 从 GitHub 移植 |
| util_generic.dll | C 类 | ❌ 无源码 | 需要逆向 |

**结论：** 该游戏目前**不能**在 KrKr2 上运行，因为 TextRender 和 util_generic 未实现。

**处理顺序：**
1. **优先移植 layerExRaster**（B 类，有源码，难度低，最快见效）
2. **其次逆向 util_generic**（C 类，函数多但每个简单，逆向难度低）
3. **最后逆向 TextRender**（C 类，最复杂，涉及字体渲染和 GPU 操作）

排序依据：先处理有源码的（成本最低），再处理难度低的无源码插件，最后处理高难度的。

</details>

### 题目 2：NCB_MODULE_NAME 的作用

请解释以下代码的作用，并说明如果将 `TJS_W("getabout.dll")` 改为 `TJS_W("GetAbout.dll")` 会发生什么：

```cpp
#define NCB_MODULE_NAME TJS_W("getabout.dll")
#include "ncbind.hpp"

NCB_ATTACH_FUNCTION(getAbout, System, TVPGetAbout);
```

<details>
<summary>查看答案</summary>

**代码作用：**
1. `NCB_MODULE_NAME` 定义了插件的注册名称——`"getabout.dll"`
2. 当 ncbind.hpp 被包含时，内部的静态注册器会使用这个名称作为 key，将注册回调存入 `_internal_plugins["getabout.dll"]` 中
3. `NCB_ATTACH_FUNCTION` 将 C++ 函数 `TVPGetAbout` 注册为 TJS 中 `System.getAbout` 方法
4. 当 TJS 脚本执行 `Plugins.link("getabout.dll")` 时，`LoadModule` 查找 `"getabout.dll"` 并执行注册

**如果改为 `"GetAbout.dll"`：**

游戏中调用 `Plugins.link("getabout.dll")` 将会**失败**（返回 `TJS_E_FAIL`）。因为 `_internal_plugins` 是 `std::map<ttstr, ...>`，查找使用字符串精确匹配：
- 注册的 key：`"GetAbout.dll"`
- 查找的 key：`"getabout.dll"`
- 结果：不匹配，找不到插件

只有当游戏脚本也改为 `Plugins.link("GetAbout.dll")` 时才能匹配成功。

**最佳实践：** NCB_MODULE_NAME 应与原版 KiriKiri DLL 的实际文件名保持一致（包括大小写），因为游戏脚本中的 `Plugins.link()` 调用是固定的。

</details>

### 题目 3：设计插件兼容性报告格式

请设计一个 JSON 格式的插件兼容性报告，要求包含以下信息：
- 游戏名称
- 检查日期
- 所需插件列表（每个插件的分类、状态、处理建议）
- 总体兼容性评分（0-100）

<details>
<summary>查看答案</summary>

```json
{
    "game_name": "示例游戏 v1.0",
    "check_date": "2026-03-21",
    "krkr2_version": "1.0.0",
    "plugins": [
        {
            "name": "csvParser.dll",
            "category": "A",
            "status": "implemented",
            "action": "none",
            "notes": "已在 krkr2plugin 中实现"
        },
        {
            "name": "TextRender.dll",
            "category": "C",
            "status": "missing",
            "action": "reverse_engineer",
            "notes": "高优先级，大量游戏依赖",
            "estimated_effort": "40h"
        },
        {
            "name": "layerExRaster.dll",
            "category": "B",
            "status": "portable",
            "action": "port_from_source",
            "source_url": "https://github.com/wtnbgo/layerExRaster",
            "estimated_effort": "4h"
        }
    ],
    "summary": {
        "total_plugins": 6,
        "implemented": 3,
        "portable": 1,
        "missing": 2,
        "compatibility_score": 50,
        "verdict": "不可运行——需要处理 TextRender 和 layerExRaster"
    }
}
```

评分规则：`compatibility_score = (implemented / total_plugins) * 100`

如果分数为 100 = 所有插件已实现，可直接运行。低于 100 时根据缺失插件的重要性判断是否可以降级运行（某些插件缺失不影响核心游戏流程）。

</details>

---

## 下一步

下一节 [无源码插件功能分析与优先级](./02-无源码插件功能分析与优先级.md) 将深入分析 C 类（无源码）插件的具体功能、使用场景和逆向优先级排序，为后续的逆向实战做准备。

