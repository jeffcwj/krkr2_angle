# M08 — 归档与 IO 系统

## 模块概述

归档与 IO 系统是 KrKr2 引擎的"文件管家"——它负责把游戏运行所需的所有资源（图片、脚本、音频、视频等）从各种打包格式中读取出来，并通过统一的流（Stream，即一种按字节顺序读写数据的抽象接口）接口提供给引擎的其他子系统使用。

在 KiriKiri/吉里吉里引擎的生态中，游戏资源通常被打包到 **XP3 归档文件**（KiriKiri 的原生打包格式，类似 ZIP 但支持分段压缩和加密过滤）中发布。但 KrKr2 模拟器为了兼容更多场景，还支持 ZIP、TAR、7z 等通用归档格式。所有这些格式通过一个统一的抽象基类 `tTVPArchive` 接入引擎，上层代码完全不需要关心底层用的是哪种格式——这就是**多态**（Polymorphism，面向对象编程中"同一个接口、不同实现"的设计思想）的经典应用。

除了归档读取，本模块还涵盖了 KrKr2 的完整 IO 栈：从底层的二进制流（`tTJSBinaryStream`，TJS2 引擎中所有流操作的基类）、内存流、部分流（只暴露原始流中某一段数据的包装流），到上层的文本流（支持多种字符编码和加密的文本读写流），再到将所有存储介质统一管理的存储媒体管理器（Storage Media Manager，一个注册和查找不同存储后端的中心化管理系统）。

理解本模块后，你将能够：阅读和修改 `cpp/core/base/` 下所有与归档和 IO 相关的源文件，为引擎添加新的归档格式支持，或者自定义资源加密/解密方案。

## 学习目标

完成本模块后，你将能够：

1. **理解 KrKr2 的存储架构** — 掌握从 `media://` 路径到实际文件/归档中数据的完整解析流程
2. **阅读 XP3 归档格式** — 深入理解 XP3 的二进制结构（魔数头、块索引、分段压缩），能手动解析 XP3 文件
3. **理解多格式归档支持** — 知道 ZIP、TAR、7z 各自的实现方式和适用场景
4. **掌握流的继承体系** — 从 `tTJSBinaryStream` 到各种派生流，理解装饰器模式（Decorator Pattern，一种通过包装已有对象来增加功能的设计模式）在 IO 中的应用
5. **使用统一存储 API** — 通过 `TVPCreateStream()` 和 `Storages` TJS2 类访问任意位置的资源
6. **实现自定义归档格式** — 从零设计并实现一个新的归档格式，完整集成到 KrKr2 引擎中

## 前置要求

- [M01-项目导览与环境搭建](../M01-项目导览与环境搭建/) — 能编译和运行 KrKr2 项目
- [M07-TJS2脚本引擎](../M07-TJS2脚本引擎/) — 理解 `tTJSBinaryStream` 基类和 TJS2 对象系统（本模块的流体系建立在 TJS2 的流基类之上）

## 术语预览

本模块将涉及以下术语，先有个印象，后续章节会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| XP3 | XP3 Archive | KiriKiri 引擎的原生归档格式，支持分段压缩、加密过滤、嵌入 EXE 等特性 |
| tTVPArchive | TVP Archive Base Class | 所有归档格式实现的抽象基类，定义了 `GetCount()`、`GetName()`、`CreateStreamByIndex()` 等纯虚函数 |
| 分段压缩 | Segmented Compression | XP3 特有的压缩方式——把一个文件拆成多个段（Segment），每段独立压缩，支持随机访问而不需要解压整个文件 |
| 段缓存 | Segment Cache | XP3 引擎中缓存已解压段数据的 LRU（Least Recently Used，最近最少使用——一种缓存淘汰策略，优先丢弃最久没用过的数据）缓存 |
| 提取过滤器 | Extraction Filter | XP3 的解密回调机制——归档被读取时，引擎会调用注册的过滤器函数对数据进行解密/解混淆 |
| tTJSBinaryStream | TJS Binary Stream | TJS2 引擎中所有二进制流操作的基类，提供 `Read()`、`Write()`、`Seek()` 等基本接口 |
| tTVPMemoryStream | TVP Memory Stream | 基于可增长内存缓冲区的流实现，数据完全在内存中，适合临时数据或小文件 |
| tTVPPartialStream | TVP Partial Stream | 只暴露底层流中某一段（指定偏移+长度）的只读包装流 |
| iTVPStorageMedia | TVP Storage Media Interface | 存储媒体的抽象接口，每种存储后端（文件系统、HTTP 等）实现此接口并注册到管理器 |
| 自动路径 | Auto Path | KrKr2 的资源发现机制——预先扫描目录/归档，建立文件名到完整路径的映射表，脚本中只需写文件名即可找到资源 |
| 存储媒体管理器 | Storage Media Manager | `tTVPStorageMediaManager` 类，用哈希表管理所有注册的存储媒体，负责路径规范化和媒体分发 |
| LRU 缓存 | Least Recently Used Cache | 一种缓存淘汰策略——当缓存满时，优先移除最久没被访问过的条目，为新数据腾出空间 |

## 章节目录

| 章 | 标题 | 主要内容 |
|----|------|----------|
| 01 | [归档系统概述](./01-归档系统概述/) | 3 节：KrKr2 存储架构总览（媒体系统、`>` 归档分隔符、路径规范化）；tTVPArchive 基类设计（引用计数、哈希索引、虚函数接口）；归档格式对比与选择（XP3/ZIP/TAR/7z 四种格式的特点和适用场景比较） |
| 02 | [XP3 归档格式详解](./02-XP3归档格式详解/) | 3 节：XP3 文件结构与头部解析（11 字节魔数、块索引、File/info/segm/adlr 子块）；分段压缩与段缓存（独立压缩段、zlib 解压、LRU 缓存淘汰）；提取过滤器与加密（解密回调、内容过滤器、xp3filter 插件机制） |
| 03 | [其他归档格式实现](./03-其他归档格式实现/) | 3 节：ZIP 归档实现（自定义 minizip 重实现、中央目录解析、ZIP64 扩展）；TAR 归档实现（POSIX/GNU 头部解析、长文件名支持、校验和验证）；7z 归档实现（LZMA SDK 集成、流适配器包装、优化的直读策略） |
| 04 | [流与 IO 抽象层](./04-流与IO抽象层/) | 3 节：tTJSBinaryStream 体系（基类设计、Seek 模式、派生流层次）；内存流与部分流（tTVPMemoryStream 可增长缓冲、tTVPPartialStream 窗口化读取、TArchiveStream 偏移包装）；文本流与编码处理（加密模式、BOM 检测、uchardet 编码自动识别、zlib 压缩文本） |
| 05 | [统一存储系统](./05-统一存储系统/) | 3 节：存储媒体管理器（iTVPStorageMedia 接口、媒体注册机制、`media://domain/path` 路径格式）；路径解析与自动搜索（路径规范化算法、自动路径表、归档缓存池）；TJS2 Storages 类（脚本层存储 API、文件列表枚举、路径操作方法） |
| 06 | [实战 — 自定义归档格式](./06-实战-自定义归档格式/) | 3 节：需求分析与格式设计（目标场景分析、二进制格式规范定义）；归档读取器实现（继承 tTVPArchive、实现索引解析和流创建）；集成测试与性能优化（单元测试编写、性能基准测试、与现有格式的对比） |

## 涉及的源码文件

本模块分析的源码文件全部位于 `cpp/core/base/` 目录下：

| 文件 | 行数 | 职责 |
|------|------|------|
| `StorageIntf.h` | 383 | 归档基类 `tTVPArchive`、存储媒体接口 `iTVPStorageMedia`、路径工具函数声明 |
| `StorageIntf.cpp` | 1388 | 存储媒体管理器、归档缓存、自动路径系统、`TVPCreateStream()` 统一入口 |
| `XP3Archive.h` | 212 | XP3 归档类声明、段缓存、XP3 流、提取过滤器类型定义 |
| `XP3Archive.cpp` | 1024 | XP3 格式解析、分段读取、缓存管理、嵌入 EXE 扫描 |
| `ZIPArchive.cpp` | 2138 | 自定义 minizip 重实现，完整的 ZIP/ZIP64 中央目录解析和 inflate 解压 |
| `TARArchive.cpp` | 146 | TAR 归档解析，POSIX/GNU 头部支持，长文件名处理 |
| `7zArchive.cpp` | 260 | 7z 归档支持，LZMA SDK C API 集成，流适配器 |
| `BinaryStream.h/cpp` | 117 | `tTJSBinaryStream` 基类的辅助函数 |
| `UtilStreams.h` | 246 | 内存流、部分流等工具流的声明 |
| `UtilStreams.cpp` | 935 | 工具流实现 + 多线程归档解包框架 |
| `TextStream.h/cpp` | 448 | 文本读写流，支持加密、BOM 检测、编码转换 |
| `CharacterSet.h/cpp` | 276 | UTF-8 与宽字符转换，编码检测（uchardet 集成） |
| `tar.h` | 134 | TAR 头部结构定义（POSIX tar header） |

## 预计学习时间

14-18 小时（含动手实践）

## 学习路线建议

```
M07 TJS2 脚本引擎
    │（理解 tTJSBinaryStream 基类和 TJS2 对象系统）
    ▼
M08 归档与 IO 系统（本模块）
    │
    ├──→ M09 插件系统与开发（理解插件如何注册提取过滤器）
    ├──→ M12 插件逆向与实现（理解归档加密在逆向中的角色）
    └──→ 日常开发：添加新归档格式、自定义资源加密方案
```
