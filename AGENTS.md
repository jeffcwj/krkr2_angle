# PROJECT KNOWLEDGE BASE

**Generated:** 2026-03-16

## 语言规范

- **语言**：文档、文档名称、注释、回答均用中文
- 
## 每次必读注意事项！！！
- 写代码优先修复问题而不是写兜底代码（if!=null之类的），写兜底代码无法解决问题
- 写入超过3000字符时，请分段写入（推荐），否则可能永远卡住。如果真想写大量内容，用python命令行写入（现在不推荐！），python不要写.py文件脚本写入而是直接命令行里代码写入
- 有阶段性成果时提交 git、更新AGENTS.md、docs里相关文档（技术性文档、PRD或教程等）和相关README.md
- **子agent并行工作时，主agent必须同时自己也继续写文档/干活，绝不能空等**。子agent干子agent的，主agent干主agent的，充分利用并行时间
- 

## OVERVIEW

KrKr2 Emulator — cross-platform C++17 emulator for KiriKiri/T Visual Presenter engine games. Targets Android (arm64/x86_64), Windows (x64), Linux (x64), macOS (arm64). Uses Cocos2d-x for rendering, TJS2 as embedded scripting engine, FFmpeg for video/audio, vcpkg for dependencies, CMake + Ninja for builds.

## STRUCTURE

```
krkr2/
├── cpp/
│   ├── core/           # Engine internals (9 modules as INTERFACE lib)
│   │   ├── base/       # Archive formats (XP3/ZIP/TAR/7z), streams, events, KAG parser
│   │   ├── common/     # Shared header (Defer.h)
│   │   ├── environ/    # Platform abstraction: cocos2d bridge, win32/linux/android/sdl/apple, UI forms, ConfigManager
│   │   ├── extension/  # Engine extension API
│   │   ├── movie/      # Video playback — ffmpeg/ subfolder is a full media player stack
│   │   ├── plugin/     # Plugin loading/binding (ncbind system)
│   │   ├── sound/      # Audio decoders (Vorbis, FFmpeg), DSP, wave management
│   │   ├── tjs2/       # TJS2 scripting engine (lexer, parser, bytecode, VM) — has bison grammars
│   │   ├── utils/      # Timers, threads, clipboard, pad input, RNG, velocity tracker
│   │   └── visual/     # Rendering: layers, bitmaps, image loaders (PNG/JPEG/TLG/WEBP/BPG/JXR/PVR), transitions, OGL
│   ├── external/       # Vendored libs: libbpg, minizip (formatting disabled)
│   └── plugins/        # Plugin implementations (STATIC lib, linked into main binary)
│       ├── psdfile/    # PSD file parser
│       ├── psbfile/    # PSB (E-mote) file parser
│       ├── motionplayer/ # E-mote motion player
│       ├── layerex_draw/ # Extended layer drawing (GDI+/general)
│       ├── fstat/      # File stat plugin
│       ├── steam/      # Steam integration (disabled in build)
│       └── (loose .cpp) # scriptsEx, csvParser, dirlist, saveStruct, windowEx, xp3filter, etc.
├── platforms/
│   ├── android/        # Gradle project + JNI bridge (krkr2_android.cpp), SDL2, Cocos2dx Java
│   ├── windows/        # WinMain entry + resources (.rc)
│   ├── linux/          # main.cpp entry
│   └── apple/macos/    # main.cpp + Info.plist + icon
├── tests/              # Catch2 v3 unit tests, organized by module
│   ├── unit-tests/core/{movie,tjs2,visual}/
│   ├── unit-tests/plugins/
│   └── test_files/     # Test fixtures (.tjs scripts, .pimg/.psb binaries)
├── tools/xp3/          # CLI XP3 archive extractor (argparse)
├── ui/cocos-studio/    # Cocos Studio UI assets (.csb, locale XMLs)
├── cmake/              # CocosBuildHelpers.cmake, vcpkg_android.cmake
├── scripts/            # build-linux.sh, build-windows.bat
├── dockers/            # linux.Dockerfile, android.Dockerfile
├── doc/                # FAQ, supported games list, plugin docs
└── vcpkg/              # Overlay ports + custom triplets
```

## WHERE TO LOOK

| Task | Location | Notes |
|------|----------|-------|
| Add image format support | `cpp/core/visual/Load*.cpp` | Follow existing Load{PNG,JPEG,TLG,...}.cpp pattern |
| Add new plugin | `cpp/plugins/` | Add .cpp + register in CMakeLists.txt; use ncbind or NCB_PRE_REGIST_CALLBACK |
| Platform-specific behavior | `cpp/core/environ/{win32,android,linux,apple,sdl}/` | PAL layer |
| Boot/startup flow | `platforms/*/main.cpp` -> `cpp/core/environ/cocos2d/AppDelegate.cpp` | All platforms construct TVPAppDelegate then call run() |
| TJS2 scripting changes | `cpp/core/tjs2/` | Bison grammars in `bison/`, lexer in tjsLex.cpp |
| Audio/video playback | `cpp/core/sound/`, `cpp/core/movie/ffmpeg/` | Full ffmpeg player stack in movie/ |
| UI forms/menus | `cpp/core/environ/ui/` | Cocos2d-based forms; assets in `ui/cocos-studio/` |
| Configuration system | `cpp/core/environ/ConfigManager/` | Global, Individual, Locale config managers |
| Archive handling | `cpp/core/base/{XP3,ZIP,TAR,7z}Archive.cpp` | XP3 is KiriKiri's native format |
| CI/build issues | `.github/workflows/` | Format check gates all builds; vcpkg NuGet needs NUGET_API_KEY secret |
| Test additions | `tests/unit-tests/<module>/` | Add TEST_CASE in .cpp, update CMakeLists; use TEST_FILES_PATH for fixtures |

## CODE MAP

**CMake Targets:**
- `krkr2` — main executable (or .so on Android)
- `krkr2core` — INTERFACE lib linking all core modules
- `krkr2plugin` — STATIC lib with all plugins
- Core modules: `tjs2`, `core_base_module`, `core_environ_module`, `core_extension_module`, `core_plugin_module`, `core_movie_module`, `core_sound_module`, `core_visual_module`, `core_utils_module`
- Plugin libs: `psdfile`, `psbfile`, `layerExDraw`, `motionplayer`, `fstat`

**Key entry flow:** `platforms/*/main.cpp` -> `TVPAppDelegate::run()` -> `applicationDidFinishLaunching()` -> Cocos2d Director/Scene init

## CONVENTIONS

- **C++ standard:** C++17 (`CMAKE_CXX_STANDARD 17`)
- **Formatting:** clang-format 20, LLVM-based. `IndentWidth: 4`, `ColumnLimit: 80`, `SpaceBeforeParens: Never`, `NamespaceIndentation: All`, `SortIncludes: Never`. CI enforces via `code-format-check.yml`
- **Formatting exceptions:** `cpp/external/`, `cpp/core/plugin/` have `.clang-format` with `DisableFormat: true`
- **Static analysis:** `.clang-tidy` with broad modernize/bugprone/cert/cppcoreguidelines checks (CLion-generated)
- **Build:** CMake 3.28+ with Ninja generator, CMakePresets.json for all platforms
- **Dependencies:** vcpkg manifest mode (`vcpkg.json`). Overlay ports in `vcpkg/`. Binary caching via NuGet on GitHub Packages
- **Platform flags:** Custom CMake variables `WINDOWS`, `LINUX`, `MACOS`, `ANDROID` (not standard CMAKE_SYSTEM_NAME)
- **Compile defs:** `TJS_TEXT_OUT_CRLF`, `__STDC_CONSTANT_MACROS`, `USE_UNICODE_FSTRING`
- **Plugin pattern:** Plugins register via ncbind macros (NCB_PRE_REGIST_CALLBACK, NCB_REGISTER_CLASS) or simplebinder
- **Testing:** Catch2 v3, custom main() per test target (spdlog init + Catch::Session). Fixtures via TEST_FILES_PATH (configure_file)
- **Logging:** spdlog throughout (loggers: "core", "tjs2", "plugin")
- **Android:** Shared lib (.so) + Gradle wrapper; JNI bridge in `platforms/android/cpp/`; generates keystore at CI time

## ANTI-PATTERNS (THIS PROJECT)

- No TODO/FIXME/HACK markers found in source code (clean codebase)
- Steam/DrawDeviceForSteam/json plugins are commented out in CMakeLists — do not uncomment without verifying dependencies
- IOS platform code is fully commented out in root CMakeLists.txt — incomplete/unmaintained
- vcpkg is NOT pinned to a specific revision in CI — builds may break on upstream changes

## COMMANDS

```bash
# Windows
./scripts/build-windows.bat
# or manually:
cmake --preset="Windows Debug Config"
cmake --build --preset="Windows Debug Build"

# Linux
./scripts/build-linux.sh
# or:
cmake --preset="Linux Debug Config"
cmake --build --preset="Linux Debug Build"

# macOS
cmake --preset="MacOS Debug Config"
cmake --build --preset="MacOS Debug Build"

# Android
./platforms/android/gradlew -p ./platforms/android assembleDebug

# Docker
docker build -f dockers/linux.Dockerfile -t linux-builder .
docker build -f dockers/android.Dockerfile -t android-builder .

# Format check
clang-format -i --verbose $(find ./cpp ./platforms ./tests ./tools -regex ".+\.\(cpp\|cc\|h\|hpp\|inc\)")

# Tests (after build)
ctest --test-dir out/<platform>/debug --output-on-failure
```

## NOTES

- **Env vars required:** `VCPKG_ROOT` on all platforms; `ANDROID_SDK` + `ANDROID_NDK` for Android; `winflexbison` in PATH for Windows
- **CI secret:** `NUGET_API_KEY` required for vcpkg binary caching in GitHub Actions
- **opencv4 pinned** to version 4.7.0#6 via vcpkg overrides — do not upgrade without testing
- **OpenMP** enabled on all platforms except Apple
- **ccache** supported — auto-detected by CMakeLists.txt
- Subdirectory AGENTS.md files exist for: `cpp/core/`, `cpp/plugins/`, `cpp/core/tjs2/`, `cpp/core/visual/`, `cpp/core/environ/`, `cpp/core/movie/ffmpeg/`, `platforms/`, `cpp/core/sound/`, `cpp/core/base/`, `tests/`

## 更新注意事项

- **语言要求：** 新增文档、注释、文档文件名均使用中文
- **AGENTS.md 维护：** 新增模块/目录时同步更新对应 AGENTS.md；子文件不得重复父文件已有内容
- **文档目录：** `docs/` 为教程文档根目录，按模块分文件夹、章节分子文件夹、节分 .md 文件
- **插件清单：** `doc/krkr2_plugins.md` 记录了完整插件列表（含已实现/未实现/无源码），新增插件实现后需更新
- **格式规范：** C++ 代码遵循 clang-format 20 (LLVM)，CI 自动检查；文档使用 Markdown
- **依赖管理：** 新增第三方库必须通过 vcpkg.json 添加，优先使用 overlay ports (`vcpkg/`)
- **测试要求：** 新增功能需附带 Catch2 单元测试，放在 `tests/unit-tests/<模块>/`

## 教程撰写目标

- **详尽为先：** 教程必须详细，文本字数可以多，唯独不能缺斤少两、缺知识少章节。必须尽可能覆盖完善，确保读者到了实战阶段不会出现"还不会"或"缺某技术栈"导致看不懂、无法编写代码或修复 Bug 的情况
- **跨平台面面俱到：** 每涉及平台相关内容，必须覆盖 Windows/Linux/macOS/Android 四个平台，不可只讲一个平台跳过其他。包括环境配置、编译差异、平台 API 差异、条件编译、调试方法等均需逐平台说明
- **练习题必备：** 每一个 .md 文件末尾必须包含"练习题与答案"章节，至少 2-3 道题，涵盖本节核心知识点。题目需有明确答案（代码题给出完整可运行代码，概念题给出详细解答）
- **无前置知识断层：** 如果某节内容依赖读者尚未学过的知识，必须在该节前置说明或内联补充，绝不可假设读者"应该知道"
- **代码即实战：** 代码示例必须完整可编译运行，不允许出现省略号(...)或"此处省略"等占位。展示项目实际代码时标注精确文件路径和行号

### 教程质量硬性标准（必须遵守）

> **背景：** 首轮编写的文件普遍在 99-251 行，内容单薄、知识点浅尝辄止。以下标准基于业界优质教程（官方 CMake Tutorial 805行/节、learnmoderncpp 500-900行/节）制定，确保内容密度达到"读完就能干活"的水平。

#### 1. 行数下限（最低门槛）

| 文件类型 | 最低行数 | 推荐行数 | 上限（超出需拆分） |
|----------|----------|----------|---------------------|
| P 系列教程节（前置知识） | **300 行** | 400-500 行 | 500 行 |
| M 系列教程节（项目模块） | **300 行** | 400-600 行 | 500 行 |
| 环境搭建类节 | **250 行** | 300-400 行 | 500 行 |
| 实战/综合类节 | **350 行** | 450-600 行 | 600 行 |

- **低于最低行数的文件不得标记为"已完成"**
- 行数超过上限时，拆分为多个子节（每节仍需满足最低行数）

#### 2. 每节必须包含的内容结构（模板）

每个 .md 文件必须按以下顺序包含所有章节（缺任何一个即不合格）：

```markdown
# [节标题]

> **所属模块：** [模块编号-模块名]
> **前置知识：** [列出需要先读的章节，用相对链接]
> **预计阅读时间：** [X 分钟]（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：
1. [具体可验证的能力 1]
2. [具体可验证的能力 2]
3. ...

## [正文内容 — 多个子章节]

### [子标题 1] — 概念解释 + 代码示例
### [子标题 2] — 深度分析 + 对照项目源码
### ... （至少 3-5 个子标题，逐步递进复杂度）

## 动手实践

> 分步骤的实操练习，读者跟着做就能得到结果

## 对照项目源码

> **M 系列必须包含此节**，P 系列如涉及项目代码也需包含
> 列出具体文件路径和行号范围，解释项目如何使用本节概念

相关文件：
- `cpp/core/xxx/yyy.cpp` 第 XX-YY 行 — [说明]

## 本节小结

- 要点 1
- 要点 2
- ...

## 练习题与答案

### 题目 1：[题目描述]

<details>
<summary>查看答案</summary>

[完整答案，代码题必须给出完整可运行代码]

</details>

### 题目 2：...

## 下一步

[下一节的相对链接和简介]
```

#### 3. 内容密度检查清单（每节发布前必须逐项核对）

- [ ] **概念解释充分**：每个新概念首次出现时有 ≥3 句话的解释（是什么、为什么需要、怎么用），术语首次出现时给出英文原文
- [ ] **代码示例充足**：全文至少包含 **5 个**完整可运行的代码示例，示例之间递进复杂度
- [ ] **代码有注释**：每个代码块的关键行都有中文行内注释
- [ ] **无占位符**：不允许出现 `(待续)`、`...`（省略号代替代码）、`此处省略` 等占位文本
- [ ] **错误与排查**：至少包含 2-3 个"常见错误及解决方案"（读者可能踩的坑）
- [ ] **对比与选择**：涉及多种方案时，必须给出对比表格或优劣分析，不可只介绍一种
- [ ] **跨平台差异**：凡涉及平台相关的操作/API/路径/命令，必须给出四平台（Win/Linux/macOS/Android）的具体说明
- [ ] **图示辅助**：复杂流程/架构用 Mermaid 或 ASCII 图表说明，不可纯文字描述
- [ ] **练习题有效**：至少 2-3 道题，题目覆盖本节核心知识点；答案使用 `<details>` 折叠；代码答案完整可运行
- [ ] **链接完整**：前置知识、下一步均使用相对路径链接，链接目标文件确实存在

#### 4. 代码示例硬性规则

- 所有 C++ 示例必须能用 CMake + 指定工具链编译通过
- 示例尽量自包含（单文件即可编译），如需多文件则给出完整目录结构和 CMakeLists.txt
- 同一章内的示例逐步增加复杂度（基础→中级→高级/实战）
- M 系列示例优先使用项目实际代码片段，标注来源文件路径和行号
- 代码块必须使用 fenced code block + 语言标签（```cpp / ```cmake / ```bash / ```gradle 等）
- 关键行必须有中文注释
- 示例应有可观察的输出（控制台打印/渲染结果/文件生成），让读者能验证自己做对了

#### 5. 禁止事项（红线）

- ❌ 单节文件低于 300 行（环境搭建类低于 250 行）就标记完成
- ❌ 出现 `(待续)`、`TODO`、`...`（代码省略号）、`此处省略` 等占位符
- ❌ 只介绍一个平台的操作而跳过其他三个
- ❌ 代码示例不完整（缺 include、缺 main、缺 CMakeLists）
- ❌ 练习题没有答案，或答案不在 `<details>` 折叠标签中
- ❌ 假设读者已知某概念但本教程前置章节中未覆盖
- ❌ 同一节中对同一概念只给一个例子就跳到下个话题
- ❌ 纯文字描述复杂架构/流程而不配图表

### 教程编写进度

| 模块 | 状态 | 文件数 | 备注 |
|------|------|--------|------|
| P01-现代CMake与构建工具链 | ✅ 已完成 | 16 md + README | 全部增强完成（350-500行/节） |
| P02-vcpkg包管理 | ✅ 已完成 | 10 md + README | 全部10文件完成（350-484行/节） |
| P03-跨平台C++开发 | ✅ 已完成 | 5 md + README | 全部5章完成（450-1441行/节） |
| P04-OpenGL图形编程 | ✅ 已完成 | 8 md + README | 全部8章完成（449-858行/章） |
| P05-软件渲染原理 | ✅ 已完成 | 5 md + README | Ch1-5全部完成（503-969行） |
| M01-项目导览与环境搭建 | ✅ 已完成 | 12 md + README | 全部增强完成（306-523行/节） |
| M02-项目构建系统深度解析 | ✅ 已完成 | 5 md + README | 全部5章完成（836-1291行/章） |
| M03-平台抽象层解析 | ✅ 已完成 | 5 md + README | Ch1-5全部完成（608-1768行/章） |
| P06-FFmpeg音视频开发 | ✅ 已完成 | 7 md + README | Ch1-7全部完成（794-1361行/章） |

## 项目目标

### 近期目标
1. **完善教程文档体系** — 在 `docs/` 下建立面向 C++ 基础开发者的完整学习路径，覆盖所有技术栈
2. **补全插件实现** — 逆向并实现 `doc/krkr2_plugins.md` 中标记为 `nan`（无源码）的插件
3. **扩大游戏兼容性** — 增加更多已验证支持的游戏（当前仅 1 款）

### 中期目标
4. **替换 UI 框架** — 用现代跨平台 UI 方案（Flutter/Compose Multiplatform）替换 Cocos2d-x UI 层
5. **完善 iOS 支持** — 恢复并完成目前被注释掉的 iOS 平台代码
6. **性能优化** — 启用 SSE/SIMD 音频路径、优化渲染管线

### 长期目标
7. **完整 KiriKiri 兼容性** — 实现所有原版 KiriKiri2/KiriKiriZ 插件 API
8. **社区建设** — 通过教程文档降低参与门槛，吸引更多贡献者
