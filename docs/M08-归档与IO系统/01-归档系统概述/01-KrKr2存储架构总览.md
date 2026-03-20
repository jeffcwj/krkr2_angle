# KrKr2 存储架构总览

> **所属模块：** M08-归档与IO系统
> **前置知识：** [M01-项目导览与环境搭建](../../M01-项目导览与环境搭建/)、[M07-TJS2脚本引擎](../../M07-TJS2脚本引擎/)（了解 tTJSBinaryStream 基类）
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：

1. **描述 KrKr2 存储系统的整体架构** — 理解从 TJS2 脚本中的文件路径到实际数据读取的完整链路
2. **理解存储媒体（Storage Media）的概念** — 知道 KrKr2 如何通过 `media://domain/path` 格式统一管理不同的存储后端
3. **掌握 `>` 归档分隔符的含义** — 理解 KrKr2 如何用路径中的 `>` 符号表示"进入归档文件内部"
4. **理解路径规范化（Path Normalization）的作用** — 知道引擎如何将各种写法的路径统一为标准格式

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| 存储媒体 | Storage Media | KrKr2 中对不同存储后端的抽象——文件系统、HTTP、归档等都是一种"存储媒体" |
| 媒体管理器 | Storage Media Manager | 管理所有已注册存储媒体的中心组件，负责根据路径前缀分发读取请求 |
| 归档分隔符 | Archive Delimiter | KrKr2 用 `>` 符号分隔归档文件路径和归档内部路径，如 `game.xp3>image/bg01.png` |
| 路径规范化 | Path Normalization | 将各种路径写法（大小写混合、正反斜杠）统一为标准格式的过程 |
| 自动路径 | Auto Path | 引擎预先扫描的路径列表，允许脚本只用文件名就能找到完整路径 |
| 统一资源定位 | Unified Resource Location | KrKr2 的路径格式 `media://domain/path`，类似 URL，用于唯一定位任意存储位置的资源 |

## 为什么需要统一的存储架构

在讲解具体实现之前，我们先理解一个根本性的问题：**为什么游戏引擎需要一个专门的存储架构，而不是直接用操作系统的文件 API？**

### 游戏资源管理的挑战

一个典型的视觉小说游戏可能包含以下资源：

```
game/
├── data/
│   ├── bgimage/          # 背景图片，可能有 200+ 张
│   ├── fgimage/          # 前景/立绘图片，可能有 500+ 张
│   ├── scenario/         # 剧本脚本，几十个 .ks 文件
│   ├── bgm/              # 背景音乐，几十首
│   ├── se/               # 音效，上百个
│   └── video/            # 视频片段
├── data.xp3              # 上述资源的打包版本
├── patch.xp3             # 补丁包（覆盖 data.xp3 中的部分文件）
└── game.exe              # 游戏可执行文件（可能内嵌 XP3 归档）
```

如果引擎直接使用操作系统的文件 API（如 C 语言的 `fopen()`），会面临以下问题：

| 问题 | 说明 |
|------|------|
| **归档访问** | 游戏发布时资源被打包到 `.xp3` 文件中，操作系统不认识这个格式 |
| **加密保护** | 资源可能被加密，需要在读取时自动解密 |
| **路径差异** | Windows 用 `\` 分隔路径，Linux/macOS 用 `/`，Android 的资源在 APK 包内 |
| **补丁覆盖** | 需要支持补丁包中的文件自动覆盖主包中的同名文件 |
| **资源发现** | 脚本中写 `Storages.open("bg01.png")`，引擎需要自动找到这个文件在哪里 |
| **多来源统一** | 同一个文件名可能存在于散装文件夹、XP3 归档、甚至 EXE 内嵌归档中 |

KrKr2 的存储架构就是为了解决这些问题而设计的。它在操作系统和游戏逻辑之间插入了一个**抽象层**（Abstraction Layer，即把底层复杂细节隐藏起来，只暴露简单接口的设计方法），让上层代码可以用统一的方式访问任意来源的资源。

### 整体架构图

```
┌─────────────────────────────────────────────────────┐
│                   TJS2 脚本层                        │
│  Storages.open("bg01.png")                          │
│  var stream = new FileStream("file://./data/a.txt") │
└───────────────────────┬─────────────────────────────┘
                        │ TVPCreateStream()
                        ▼
┌─────────────────────────────────────────────────────┐
│              统一存储入口 (StorageIntf.cpp)           │
│                                                     │
│  1. 路径规范化：统一斜杠、小写化                       │
│  2. 自动路径查找：文件名 → 完整路径                    │
│  3. 归档分隔符解析：识别 > 符号                        │
│  4. 媒体分发：根据 media:// 前缀选择后端               │
└───────────┬──────────┬──────────┬───────────────────┘
            │          │          │
            ▼          ▼          ▼
┌─────────┐ ┌────────┐ ┌──────────────────┐
│ file:// │ │ http://│ │ 归档媒体         │
│ 本地文件 │ │ 网络   │ │ (XP3/ZIP/TAR/7z) │
│ 系统    │ │ 资源   │ │                  │
└─────────┘ └────────┘ └──────┬───────────┘
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
              ┌────────┐ ┌──────┐ ┌─────────┐
              │  XP3   │ │ ZIP  │ │ TAR/7z  │
              │ Archive│ │Archve│ │ Archive │
              └────────┘ └──────┘ └─────────┘
```

这个架构的核心思想是**分层解耦**（Layered Decoupling，把系统分成多层，每层只依赖下一层的接口，不直接跨层调用）：

- **脚本层**只知道文件名或路径，不关心文件在哪里
- **统一存储入口**负责路径解析和分发，不关心具体格式
- **归档实现**只负责解析特定格式，不关心谁在调用

## 存储媒体（Storage Media）系统

### 什么是存储媒体

在 KrKr2 中，**存储媒体**（Storage Media）是对不同数据来源的抽象。每种存储媒体代表一种获取数据的方式：

- **file** 媒体：从本地文件系统读取文件
- **arc** 媒体：从归档文件（XP3、ZIP 等）中读取资源
- **http/https** 媒体：从网络下载资源（原版 KiriKiri 支持，KrKr2 中未启用）

每种存储媒体都实现了 `iTVPStorageMedia` 接口（interface，即一组纯虚函数的集合，定义了"能做什么"但不实现"怎么做"），并注册到全局的存储媒体管理器中。

### 存储媒体接口定义

以下是 `iTVPStorageMedia` 接口的核心方法。这段代码来自 `cpp/core/base/StorageIntf.h`：

```cpp
// 文件路径：cpp/core/base/StorageIntf.h（简化版本）
// 存储媒体接口——所有存储后端必须实现这些方法
class iTVPStorageMedia {
public:
    virtual ~iTVPStorageMedia() = default;

    // GetName() — 返回媒体名称，如 "file"、"arc"
    // 这个名称会出现在路径的 "media://" 前缀中
    virtual void GetName(ttstr &name) = 0;

    // NormalizeDomainName() — 规范化域名部分
    // 比如将 "C:\Users\Game" 统一为 "c:/users/game"
    virtual bool NormalizeDomainName(ttstr &name) = 0;

    // NormalizePathName() — 规范化路径部分
    // 统一斜杠方向、大小写等
    virtual bool NormalizePathName(ttstr &name) = 0;

    // CheckExistentStorage() — 检查指定路径的资源是否存在
    virtual bool CheckExistentStorage(const ttstr &name) = 0;

    // Open() — 打开指定路径的资源，返回一个流对象
    virtual tTJSBinaryStream* Open(const ttstr &name, tjs_uint32 flags) = 0;

    // GetListAt() — 列出指定目录下的所有文件/子目录
    virtual void GetListAt(const ttstr &name,
                          iTVPStorageLister *lister) = 0;

    // GetLocallyAccessibleName() — 获取操作系统可直接访问的本地路径
    // 对于归档内的文件，这个方法返回空（因为 OS 无法直接访问归档内部）
    virtual void GetLocallyAccessibleName(ttstr &name) = 0;
};
```

这个接口的设计体现了**依赖倒置原则**（Dependency Inversion Principle, DIP，高层模块不应依赖低层模块，二者都应依赖抽象接口）——引擎核心代码只依赖 `iTVPStorageMedia` 接口，而不依赖任何具体的文件系统或归档实现。

### 媒体注册与查找

存储媒体通过 `tTVPStorageMediaManager` 管理器进行注册和查找。管理器内部使用**哈希表**（Hash Table，一种通过键的哈希值快速查找数据的数据结构，平均查找时间为 O(1)）存储所有注册的媒体：

```cpp
// 文件路径：cpp/core/base/StorageIntf.cpp（简化版本）
// 注册一个新的存储媒体到管理器
void tTVPStorageMediaManager::Register(iTVPStorageMedia *media) {
    ttstr name;
    media->GetName(name);  // 获取媒体名称，如 "file"

    // 检查是否已注册同名媒体
    if (MediaHash.find(name) != MediaHash.end()) {
        // 已存在同名媒体，抛出异常
        TVPThrowExceptionMessage(
            TJS_W("Storage media '%1' is already registered."), name);
    }

    // 将媒体添加到哈希表中
    // 键是媒体名称（如 "file"），值是媒体对象指针
    MediaHash[name] = media;

    // 同时添加到有序列表，用于按注册顺序遍历
    MediaList.push_back(media);
}

// 根据路径查找对应的存储媒体
iTVPStorageMedia* tTVPStorageMediaManager::GetMedia(
    const ttstr &name) {
    // 从路径中提取媒体名称
    // 例如 "file://./data/bg01.png" → 提取 "file"
    ttstr medianame;
    // ... 解析 name 中 :// 之前的部分 ...

    auto it = MediaHash.find(medianame);
    if (it == MediaHash.end()) {
        TVPThrowExceptionMessage(
            TJS_W("Storage media '%1' is not found."), medianame);
    }
    return it->second;
}
```

### 路径格式：`media://domain/path`

KrKr2 的资源路径遵循一种类似 URL 的格式：

```
media://domain/path
```

各部分的含义：

| 部分 | 含义 | 示例 |
|------|------|------|
| `media` | 存储媒体名称 | `file`、`arc` |
| `://` | 固定分隔符 | 与 URL 相同 |
| `domain` | 域名（通常是根目录） | `.`（当前目录）、`C:`（盘符） |
| `/path` | 资源路径 | `/data/bg01.png` |

实际使用示例：

```
file://./data/bg01.png          ← 当前目录下的 data/bg01.png
file://C:/Games/Novel/bg.png    ← Windows 绝对路径
arc://./data.xp3>bg01.png       ← data.xp3 归档中的 bg01.png
```

> **注意**：在 TJS2 脚本中，开发者通常不需要写完整的 `media://` 路径。引擎会自动推断媒体类型——如果路径包含 `>`，就使用归档媒体；否则使用文件媒体。

## `>` 归档分隔符详解

### 为什么需要归档分隔符

当游戏资源被打包到归档文件中后，一个资源的"位置"需要两个信息：

1. 归档文件在文件系统中的路径（比如 `C:/Games/Novel/data.xp3`）
2. 资源在归档内部的路径（比如 `bgimage/bg01.png`）

KrKr2 使用 `>` 符号将这两部分连接起来，形成一个完整的路径：

```
C:/Games/Novel/data.xp3>bgimage/bg01.png
│                      │ │
│                      │ └─ 归档内部路径（archive-internal path）
│                      └─── 归档分隔符
└──────────────────────── 归档文件路径（host file path）
```

### 多层嵌套归档

`>` 分隔符支持多层嵌套。虽然实际游戏中很少见，但理论上可以这样写：

```
outer.xp3>inner.xp3>data/image.png
```

这表示：先打开 `outer.xp3`，从中提取 `inner.xp3`，再从 `inner.xp3` 中读取 `data/image.png`。

### 源码中的分隔符处理

以下代码展示了引擎如何解析包含 `>` 的路径。来自 `cpp/core/base/StorageIntf.cpp`：

```cpp
// 文件路径：cpp/core/base/StorageIntf.cpp（简化版本）
// TVPCreateStream() — 创建流的统一入口
// 这是整个存储系统最核心的函数之一
tTJSBinaryStream* TVPCreateStream(
    const ttstr &name,      // 资源路径，如 "data.xp3>bg01.png"
    tjs_uint32 flags) {     // 打开标志（读/写/创建等）

    // 第一步：查找路径中的 > 分隔符
    tjs_int archdelimpos = name.LastIndexOf(TJS_W('>'));

    if (archdelimpos != -1) {
        // 路径中包含 > —— 这是归档内部的资源
        // 分离归档路径和内部路径
        ttstr archname = name.SubString(0, archdelimpos);
        // archname = "data.xp3"
        ttstr localname = name.SubString(archdelimpos + 1);
        // localname = "bg01.png"

        // 从归档缓存中获取或打开归档
        tTVPArchive *archive = TVPOpenArchive(archname);

        // 在归档中查找并创建流
        tTJSBinaryStream *stream =
            archive->CreateStream(localname, flags);

        return stream;
    } else {
        // 路径中没有 > —— 这是普通文件
        // 交给存储媒体系统处理
        return TVPOpenFileStream(name, flags);
    }
}
```

### 归档分隔符的规范化

在路径规范化过程中，`>` 分隔符会被特殊处理——归档路径和内部路径分别独立规范化：

```cpp
// 文件路径：cpp/core/base/StorageIntf.cpp（简化版本）
// 规范化包含归档分隔符的路径
ttstr TVPNormalizeStorageName(const ttstr &name) {
    tjs_int archdelimpos = name.LastIndexOf(TJS_W('>'));

    if (archdelimpos != -1) {
        // 分别规范化归档路径和内部路径
        ttstr archpart = name.SubString(0, archdelimpos);
        ttstr innerpart = name.SubString(archdelimpos + 1);

        // 归档路径按文件系统规则规范化
        archpart = TVPNormalizeStorageName(archpart);  // 递归处理（支持多层嵌套）

        // 内部路径按归档内部规则规范化
        TVPNormalizeInArchiveStorageName(innerpart);   // 小写化 + 统一斜杠

        return archpart + TJS_W(">") + innerpart;
    }

    // 普通路径的规范化...
    // （下文详细讲解）
}
```

> **关键细节**：注意这里用的是 `LastIndexOf` 而不是 `IndexOf`。这意味着如果路径中有多个 `>`，最后一个 `>` 才是真正的分隔符。这是为了正确处理归档路径本身包含 `>` 字符的极端情况。

## 路径规范化（Path Normalization）

### 为什么需要路径规范化

不同操作系统和用户输入可能产生各种写法不一致的路径：

```
C:\Games\Novel\data\bg01.png    ← Windows 风格反斜杠
C:/Games/Novel/data/bg01.png    ← Unix 风格正斜杠
c:/games/novel/data/BG01.PNG    ← 大小写不同
./data/../data/bg01.png         ← 包含 . 和 .. 相对路径
```

如果不做规范化，同一个文件可能因为路径写法不同而被当成不同的资源，导致重复加载或查找失败。

### 规范化规则

KrKr2 的路径规范化分为两个场景：

#### 1. 普通文件路径规范化

```cpp
// 文件路径：cpp/core/base/StorageIntf.cpp（简化版本）
// TVPNormalizeStorageName() — 规范化普通文件路径
void TVPNormalizeStorageName(ttstr &name) {
    // 规则 1：反斜杠 → 正斜杠
    // "C:\data\bg01.png" → "C:/data/bg01.png"
    tjs_char *p = name.Independ();
    while (*p) {
        if (*p == TJS_W('\\')) *p = TJS_W('/');
        p++;
    }

    // 规则 2：去除末尾斜杠（目录路径除外）
    // "data/images/" → "data/images"

    // 规则 3：解析 . 和 ..
    // "./data/../data/bg01.png" → "data/bg01.png"
}
```

#### 2. 归档内部路径规范化

归档内部的路径有额外的规范化要求——**全部转为小写**：

```cpp
// 文件路径：cpp/core/base/StorageIntf.h（简化版本）
// NormalizeInArchiveStorageName() — 归档内部路径规范化
static void NormalizeInArchiveStorageName(ttstr &name) {
    // 规则 1：反斜杠 → 正斜杠
    // "bgimage\bg01.png" → "bgimage/bg01.png"

    // 规则 2：全部转为小写
    // "BGImage/BG01.PNG" → "bgimage/bg01.png"
    tjs_char *p = name.Independ();
    while (*p) {
        if (*p == TJS_W('\\')) *p = TJS_W('/');
        if (*p >= TJS_W('A') && *p <= TJS_W('Z'))
            *p += TJS_W('a') - TJS_W('A');  // 大写转小写
        p++;
    }
}
```

> **为什么归档内部路径要全部小写？**
>
> 这是为了实现**大小写不敏感的文件查找**。原版 KiriKiri 运行在 Windows 上，而 Windows 的文件系统（NTFS/FAT32）本身就是大小写不敏感的。但 KrKr2 需要在 Linux（大小写敏感的 ext4 文件系统）和 Android 上运行，因此需要在引擎层面统一为小写来模拟 Windows 的行为。

### 规范化的完整流程示意

```
用户输入: "C:\Games\Novel\Data.XP3>BGImage\BG01.PNG"
    │
    ├─ 分离归档路径: "C:\Games\Novel\Data.XP3"
    │   └─ 规范化: 反斜杠→正斜杠
    │       → "C:/Games/Novel/Data.XP3"
    │
    └─ 分离内部路径: "BGImage\BG01.PNG"
        └─ 规范化: 反斜杠→正斜杠 + 全部小写
            → "bgimage/bg01.png"

最终结果: "C:/Games/Novel/Data.XP3>bgimage/bg01.png"
```

## 自动路径（Auto Path）系统

### 什么是自动路径

**自动路径**（Auto Path）是 KrKr2 的资源发现机制。它解决的问题是：游戏脚本中写 `Storages.open("bg01.png")`，但 `bg01.png` 实际上在 `data.xp3` 归档的 `bgimage/` 目录下——引擎怎么知道完整路径是什么？

自动路径的工作原理：

1. **启动时预扫描**：引擎启动时扫描指定目录和归档，建立"文件名 → 完整路径"的映射表
2. **运行时查表**：脚本请求资源时，先在映射表中查找文件名，获取完整路径
3. **优先级规则**：如果多个位置有同名文件，后注册的路径优先（这就是补丁包机制的基础）

### 自动路径的数据结构

```cpp
// 文件路径：cpp/core/base/StorageIntf.cpp（简化版本）

// TVPAutoPathTable — 文件名到完整路径的映射表
// 使用哈希表实现 O(1) 查找
static std::unordered_map<ttstr, ttstr> TVPAutoPathTable;

// TVPAutoPathList — 已注册的自动搜索路径列表
static std::vector<ttstr> TVPAutoPathList;

// 添加一个自动搜索路径
void TVPAddAutoPath(const ttstr &path) {
    // 1. 将路径添加到列表
    TVPAutoPathList.push_back(path);

    // 2. 扫描该路径下的所有文件
    //    对于归档路径（包含 >），扫描归档内部
    //    对于普通路径，扫描文件系统目录
    std::vector<ttstr> files;
    TVPGetListAt(path, files);

    // 3. 将每个文件名映射到完整路径
    for (auto &file : files) {
        ttstr fullpath = path + file;
        ttstr filename = TVPExtractStorageName(file);
        // 如果已有同名映射，新路径覆盖旧路径
        TVPAutoPathTable[filename] = fullpath;
    }
}

// 通过文件名查找完整路径
bool TVPResolveAutoPath(const ttstr &name, ttstr &result) {
    auto it = TVPAutoPathTable.find(name);
    if (it != TVPAutoPathTable.end()) {
        result = it->second;
        return true;  // 找到了
    }
    return false;  // 没找到，name 可能已经是完整路径
}
```

### 自动路径的实际使用场景

```javascript
// TJS2 脚本示例

// 在游戏启动脚本 startup.tjs 中注册自动搜索路径
Storages.addAutoPath("data.xp3>");   // 注册主资源包
Storages.addAutoPath("patch.xp3>");  // 注册补丁包（优先级更高）
Storages.addAutoPath("./override/"); // 注册本地覆盖目录（开发用）

// 之后在游戏脚本中，只需要写文件名
// 引擎会自动在注册的路径中查找
var layer = new Layer(window, window.primaryLayer);
layer.loadImages("bg01");  // 自动解析为完整路径

// 如果 bg01.png 同时存在于 data.xp3 和 patch.xp3 中，
// 由于 patch.xp3 后注册，patch.xp3 中的版本会被使用
// ——这就是"热补丁"机制的原理！
```

### 自动路径的优先级机制

```
注册顺序：
1. Storages.addAutoPath("data.xp3>");     ← 先注册
2. Storages.addAutoPath("patch.xp3>");    ← 后注册
3. Storages.addAutoPath("./override/");   ← 最后注册

当请求 "bg01.png" 时，查找优先级：
./override/bg01.png   ← 最优先（最后注册）
patch.xp3>bg01.png    ← 次优先
data.xp3>bg01.png     ← 最低优先（最先注册）
```

这个机制使得游戏更新变得非常简单——只需要发布一个 `patch.xp3`，不需要修改原始的 `data.xp3`。

## TVPCreateStream() — 统一流创建入口

### 完整的流创建流程

`TVPCreateStream()` 是整个存储系统中被调用最频繁的函数——每次加载图片、播放音频、读取脚本，最终都会调用到这里。下面是它的完整工作流程：

```
TVPCreateStream("bg01.png", TJS_BS_READ)
    │
    ├─ 第 1 步：自动路径解析
    │   查找 TVPAutoPathTable["bg01.png"]
    │   → 找到 "data.xp3>bgimage/bg01.png"
    │
    ├─ 第 2 步：检测归档分隔符
    │   路径包含 ">" → 这是归档资源
    │   分离: archname = "data.xp3"
    │         localname = "bgimage/bg01.png"
    │
    ├─ 第 3 步：获取归档对象
    │   从 TVPArchiveCache 中查找 "data.xp3"
    │   如果缓存未命中 → 打开并解析归档文件
    │   → 返回 tTVPXP3Archive 对象
    │
    ├─ 第 4 步：在归档中创建流
    │   archive->CreateStream("bgimage/bg01.png")
    │   → 查找文件索引
    │   → 创建 tTVPXP3ArchiveStream 对象
    │
    └─ 返回流对象给调用方
```

### 流创建标志（flags）

`TVPCreateStream()` 的第二个参数 `flags` 控制流的打开方式：

```cpp
// 文件路径：cpp/core/tjs2/tjsCommHead.h（简化版本）
// 流打开标志常量定义

// TJS_BS_READ — 以只读方式打开
// 最常用的标志，用于加载资源
#define TJS_BS_READ     0

// TJS_BS_WRITE — 以只写方式打开
// 用于写入存档、日志等
#define TJS_BS_WRITE    1

// TJS_BS_APPEND — 以追加方式打开
// 在文件末尾添加数据
#define TJS_BS_APPEND   2

// TJS_BS_UPDATE — 以读写方式打开
// 既可以读也可以写
#define TJS_BS_UPDATE   3
```

> **注意**：归档内的文件只支持 `TJS_BS_READ`。如果尝试以写入方式打开归档内的文件，引擎会抛出异常。这是合理的——归档文件是只读的打包格式。

### 跨平台的流创建差异

不同平台上，底层文件系统的流创建实现有所不同：

| 平台 | 文件流实现 | 关键差异 |
|------|-----------|---------|
| Windows | `CreateFileW()` API | 使用宽字符 (UTF-16) 路径，支持长路径前缀 `\\?\` |
| Linux | `open()` 系统调用 | 使用 UTF-8 路径，大小写敏感 |
| macOS | `open()` 系统调用 | 使用 UTF-8 路径，HFS+ 默认大小写不敏感 |
| Android | `AAssetManager_open()` | APK 内资源通过 Android Asset Manager 访问 |

```cpp
// 文件路径：cpp/core/environ/Platform.cpp（简化概念示例）
// 各平台的文件流打开实现

#ifdef _WIN32
// Windows：使用 CreateFileW 打开文件
tTJSBinaryStream* TVPCreateFileStream(
    const ttstr &name, tjs_uint32 flags) {
    HANDLE handle = CreateFileW(
        name.c_str(),               // UTF-16 宽字符路径
        GENERIC_READ,               // 读取权限
        FILE_SHARE_READ,            // 允许其他进程同时读取
        nullptr,                    // 默认安全属性
        OPEN_EXISTING,              // 只打开已存在的文件
        FILE_ATTRIBUTE_NORMAL,      // 普通文件属性
        nullptr                     // 无模板文件
    );
    if (handle == INVALID_HANDLE_VALUE) {
        // 文件打开失败，抛出异常
        TVPThrowExceptionMessage(
            TJS_W("Cannot open file '%1'."), name);
    }
    return new tTVPFileStream(handle);
}
#elif defined(__ANDROID__)
// Android：使用 AAssetManager 访问 APK 内资源
tTJSBinaryStream* TVPCreateFileStream(
    const ttstr &name, tjs_uint32 flags) {
    // 将宽字符路径转为 UTF-8
    std::string utf8name = ttstr_to_utf8(name);

    // 通过 Android Asset Manager 打开
    AAsset *asset = AAssetManager_open(
        g_assetManager,             // 全局 Asset Manager
        utf8name.c_str(),           // UTF-8 路径
        AASSET_MODE_RANDOM          // 支持随机访问（seek）
    );
    if (!asset) {
        // 回退到普通文件系统
        return TVPCreateLocalFileStream(name, flags);
    }
    return new tTVPAndroidAssetStream(asset);
}
#else
// Linux/macOS：使用标准 POSIX open()
tTJSBinaryStream* TVPCreateFileStream(
    const ttstr &name, tjs_uint32 flags) {
    std::string utf8name = ttstr_to_utf8(name);
    int fd = open(utf8name.c_str(), O_RDONLY);
    if (fd < 0) {
        TVPThrowExceptionMessage(
            TJS_W("Cannot open file '%1'."), name);
    }
    return new tTVPPosixFileStream(fd);
}
#endif
```

## 动手实践

### 练习 1：追踪一次资源加载的完整路径

打开 KrKr2 项目源码，找到 `cpp/core/base/StorageIntf.cpp`，从 `TVPCreateStream()` 函数开始，手动追踪以下路径的解析过程：

```
输入路径: "data.xp3>scenario/prologue.ks"
```

逐步回答：

1. `TVPCreateStream()` 在哪一行检测到 `>` 分隔符？
2. 它调用了哪个函数来打开归档文件？
3. 归档缓存 `TVPArchiveCache` 在哪里定义？它的缓存容量是多少？

### 练习 2：编写一个简单的路径规范化函数

基于本节学到的规则，用 C++ 编写一个简化版的路径规范化函数：

```cpp
#include <iostream>
#include <string>
#include <algorithm>

// 简化版路径规范化——只处理斜杠替换和小写转换
std::string normalizeArchivePath(const std::string &path) {
    std::string result = path;

    // 查找 > 分隔符
    size_t delimPos = result.rfind('>');

    if (delimPos != std::string::npos) {
        // 归档路径部分：只替换斜杠
        for (size_t i = 0; i < delimPos; ++i) {
            if (result[i] == '\\') result[i] = '/';
        }
        // 归档内部路径：替换斜杠 + 小写化
        for (size_t i = delimPos + 1; i < result.size(); ++i) {
            if (result[i] == '\\') result[i] = '/';
            result[i] = std::tolower(
                static_cast<unsigned char>(result[i]));
        }
    } else {
        // 普通路径：只替换斜杠
        for (auto &c : result) {
            if (c == '\\') c = '/';
        }
    }

    return result;
}

int main() {
    // 测试用例 1：包含归档分隔符的路径
    std::string path1 = "C:\\Games\\Data.XP3>BGImage\\BG01.PNG";
    std::cout << "输入: " << path1 << std::endl;
    std::cout << "输出: " << normalizeArchivePath(path1) << std::endl;
    // 期望输出: C:/Games/Data.XP3>bgimage/bg01.png

    // 测试用例 2：普通文件路径
    std::string path2 = "C:\\Games\\Novel\\data\\bg01.png";
    std::cout << "输入: " << path2 << std::endl;
    std::cout << "输出: " << normalizeArchivePath(path2) << std::endl;
    // 期望输出: C:/Games/Novel/data/bg01.png

    // 测试用例 3：多层嵌套归档
    std::string path3 = "outer.xp3>inner.xp3>Data\\IMG.PNG";
    std::cout << "输入: " << path3 << std::endl;
    std::cout << "输出: " << normalizeArchivePath(path3) << std::endl;
    // 期望输出: outer.xp3>inner.xp3>data/img.png
    // 注意：rfind 只找最后一个 >，所以 outer.xp3>inner.xp3 被视为归档路径部分

    return 0;
}
```

编译并运行：

```bash
# Linux/macOS
g++ -std=c++17 -o normalize normalize.cpp && ./normalize

# Windows (MSVC)
cl /EHsc /std:c++17 normalize.cpp && normalize.exe
```

## 对照项目源码

本节涉及的核心源码文件：

| 文件路径 | 行号范围 | 内容说明 |
|---------|----------|---------|
| `cpp/core/base/StorageIntf.h` | 第 1-60 行 | `iTVPStorageMedia` 接口定义 |
| `cpp/core/base/StorageIntf.h` | 第 60-120 行 | `tTVPArchive` 基类声明（含 `NormalizeInArchiveStorageName()`） |
| `cpp/core/base/StorageIntf.cpp` | 第 1-100 行 | `tTVPStorageMediaManager` 实现（注册/查找/路径解析） |
| `cpp/core/base/StorageIntf.cpp` | 第 700-900 行 | `TVPCreateStream()`、自动路径系统 |
| `cpp/core/base/StorageIntf.cpp` | 第 1100-1388 行 | `tTJSNC_Storages`（TJS2 Storages 类的原生实现） |

建议阅读顺序：

1. 先读 `StorageIntf.h`，理解接口设计
2. 再读 `StorageIntf.cpp` 中的 `TVPCreateStream()`，理解调用链
3. 最后读自动路径相关代码，理解资源发现机制

## 常见错误与排查

### 错误 1：资源文件找不到（"Storage XXX is not found"）

**症状**：运行游戏时控制台输出 `Storage 'bg01.png' is not found`

**可能原因**：
- 自动路径未注册（`Storages.addAutoPath()` 未调用或路径不正确）
- 路径大小写问题（Linux 上 `BG01.PNG` 和 `bg01.png` 是不同文件）
- 归档文件损坏或路径错误

**排查方法**：
```javascript
// 在 TJS2 脚本中检查文件是否存在
System.inform(Storages.isExistentStorage("bg01.png"));
// 输出 1（存在）或 0（不存在）

// 查看自动路径解析结果
System.inform(Storages.getPlacedPath("bg01.png"));
// 输出完整路径，如 "data.xp3>bgimage/bg01.png"
```

### 错误 2：跨平台路径问题

**症状**：游戏在 Windows 上正常运行，但在 Linux/Android 上找不到文件

**原因**：Windows 的 NTFS 文件系统大小写不敏感，但 Linux 的 ext4 和 Android 的文件系统大小写敏感

**解决方案**：确保所有资源文件名使用小写，或确认归档内部路径规范化正确生效

### 错误 3：归档分隔符被误解析

**症状**：路径中包含 `>` 字符的文件名被错误地解析为归档路径

**原因**：虽然极少见，但如果文件名本身包含 `>`（如 `log>2024.txt`），引擎会把它当成归档分隔符

**解决方案**：避免在文件名中使用 `>` 字符。KrKr2 不提供转义机制。

## 本节小结

- KrKr2 使用**存储媒体系统**抽象不同的数据来源，每种媒体实现 `iTVPStorageMedia` 接口
- 路径格式为 `media://domain/path`，在实际使用中通常可以省略 `media://` 前缀
- **`>` 归档分隔符**用于表示"进入归档内部"，支持多层嵌套
- **路径规范化**统一斜杠方向和大小写，归档内部路径会全部转为小写以实现跨平台兼容
- **自动路径系统**建立"文件名→完整路径"的映射表，支持补丁优先级机制
- `TVPCreateStream()` 是统一的流创建入口，它串联了路径解析、归档访问、流创建的完整流程

## 练习题与答案

### 题目 1：路径规范化结果

给定以下输入路径，写出 KrKr2 路径规范化后的结果：

```
输入 A: "C:\Games\Novel\DATA.XP3>BGImage\BG01.PNG"
输入 B: "file://./data/test.xp3>Scripts\Main.TJS"
输入 C: "patch.xp3>patch.xp3>Image\CG01.PNG"
```

<details>
<summary>查看答案</summary>

```
输出 A: "C:/Games/Novel/DATA.XP3>bgimage/bg01.png"
解释：归档路径部分（> 之前）只做斜杠替换，保留大小写
      归档内部路径（> 之后）做斜杠替换 + 全部小写

输出 B: "file://./data/test.xp3>scripts/main.tjs"
解释：media:// 前缀保留不变
      归档内部路径 "Scripts\Main.TJS" → "scripts/main.tjs"

输出 C: "patch.xp3>patch.xp3>image/cg01.png"
解释：rfind('>') 找到最后一个 >（第二个）
      归档路径部分是 "patch.xp3>patch.xp3"（保留大小写，斜杠替换）
      归档内部路径是 "Image\CG01.PNG" → "image/cg01.png"
```

</details>

### 题目 2：自动路径优先级

以下代码注册了三个自动搜索路径。如果文件 `config.tjs` 同时存在于这三个位置，引擎会使用哪个版本？

```javascript
Storages.addAutoPath("base.xp3>");
Storages.addAutoPath("update_v2.xp3>");
Storages.addAutoPath("./debug/");
```

<details>
<summary>查看答案</summary>

引擎会使用 `./debug/config.tjs`。

**原因**：自动路径的优先级规则是"**后注册的优先**"。当多个路径中存在同名文件时，后注册的路径会覆盖先注册的路径在映射表中的条目。

优先级从高到低：
1. `./debug/config.tjs`（最后注册，最高优先级）
2. `update_v2.xp3>config.tjs`（第二个注册）
3. `base.xp3>config.tjs`（最先注册，最低优先级）

这个机制在实际开发中非常有用：
- 游戏发布时将资源打包到 `base.xp3`
- 发布更新时提供 `update_v2.xp3`，自动覆盖旧资源
- 开发调试时在 `./debug/` 放置修改后的文件，无需重新打包

</details>

### 题目 3：实现一个简单的存储媒体

基于 `iTVPStorageMedia` 接口，实现一个"内存存储媒体"，它从预先注册的内存缓冲区中读取数据。

<details>
<summary>查看答案</summary>

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <vector>
#include <cstring>

// 简化的二进制流基类
class SimpleBinaryStream {
public:
    virtual ~SimpleBinaryStream() = default;
    virtual size_t Read(void *buffer, size_t size) = 0;
    virtual size_t GetSize() = 0;
};

// 内存流实现——从一段内存缓冲区中读取数据
class MemoryStream : public SimpleBinaryStream {
    const uint8_t *data_;  // 指向数据缓冲区的指针
    size_t size_;           // 数据总大小
    size_t pos_;            // 当前读取位置
public:
    MemoryStream(const uint8_t *data, size_t size)
        : data_(data), size_(size), pos_(0) {}

    size_t Read(void *buffer, size_t size) override {
        // 计算实际可读取的字节数
        size_t readable = std::min(size, size_ - pos_);
        // 复制数据到调用方的缓冲区
        std::memcpy(buffer, data_ + pos_, readable);
        pos_ += readable;  // 更新读取位置
        return readable;    // 返回实际读取的字节数
    }

    size_t GetSize() override { return size_; }
};

// 内存存储媒体——从预注册的内存缓冲区中提供数据
class MemoryStorageMedia {
    // 文件名 → 数据缓冲区的映射表
    std::unordered_map<std::string, std::vector<uint8_t>> files_;

public:
    // 获取媒体名称
    std::string GetName() const { return "memory"; }

    // 注册一个内存文件
    void RegisterFile(const std::string &name,
                     const std::vector<uint8_t> &data) {
        files_[name] = data;
    }

    // 检查文件是否存在
    bool CheckExistent(const std::string &name) const {
        return files_.find(name) != files_.end();
    }

    // 打开文件，返回流对象
    SimpleBinaryStream* Open(const std::string &name) {
        auto it = files_.find(name);
        if (it == files_.end()) {
            throw std::runtime_error(
                "File not found: " + name);
        }
        return new MemoryStream(
            it->second.data(), it->second.size());
    }
};

int main() {
    MemoryStorageMedia media;

    // 注册测试文件到内存媒体
    std::vector<uint8_t> helloData = {
        'H', 'e', 'l', 'l', 'o', ',', ' ',
        'K', 'r', 'K', 'r', '2', '!'
    };
    media.RegisterFile("hello.txt", helloData);

    // 检查文件存在性
    std::cout << "hello.txt 存在: "
              << media.CheckExistent("hello.txt")
              << std::endl;  // 输出: 1
    std::cout << "world.txt 存在: "
              << media.CheckExistent("world.txt")
              << std::endl;  // 输出: 0

    // 读取文件内容
    auto *stream = media.Open("hello.txt");
    char buffer[64] = {};
    size_t bytesRead = stream->Read(buffer, sizeof(buffer));
    std::cout << "读取 " << bytesRead << " 字节: "
              << buffer << std::endl;
    // 输出: 读取 13 字节: Hello, KrKr2!

    delete stream;
    return 0;
}
```

编译并运行：

```bash
# Linux/macOS
g++ -std=c++17 -o memory_media memory_media.cpp && ./memory_media

# Windows (MSVC)
cl /EHsc /std:c++17 memory_media.cpp && memory_media.exe
```

</details>

## 下一步

下一节 [tTVPArchive 基类设计](./02-tTVPArchive基类设计.md) 将深入分析归档系统的核心抽象——`tTVPArchive` 基类。我们会看到它如何通过引用计数管理生命周期、如何用哈希表实现 O(1) 文件查找、以及四种归档格式如何通过继承来复用这些能力。

