# 目录结构与 CMake 构建

> **所属模块：** M05-音频子系统
> **前置知识：** [M02-项目构建系统深度解析](../../M02-项目构建系统深度解析/README.md)、[P06-FFmpeg音视频开发](../../P06-FFmpeg音视频开发/README.md)、[P07-OpenAL音频编程](../../P07-OpenAL音频编程/README.md)
> **预计阅读时间：** 25 分钟

## 本节目标

读完本节后，你将能够：
1. 说出 `cpp/core/sound/` 目录下每个源文件的职责
2. 理解 CMake 构建配置中静态库目标、依赖链接和平台条件编译的具体写法
3. 独立修改 CMakeLists.txt 添加新的音频源文件和第三方依赖
4. 解释 `win32/` 子目录命名的历史原因及其跨平台实际用途

## sound 模块在引擎中的位置

KrKr2 引擎的核心代码组织在 `cpp/core/` 下，共 9 个子系统模块，每个模块编译为独立的 CMake 静态库目标，最终通过 `krkr2core` INTERFACE 库链接到主可执行文件：

```
cpp/core/
├── base/       → core_base_module      # I/O、归档、事件、KAG 解析器
├── common/     → (无独立目标)            # 共享头文件 (Defer.h)
├── environ/    → core_environ_module    # 平台抽象层、UI 表单
├── extension/  → core_extension_module  # 引擎扩展 API
├── movie/      → core_movie_module      # 视频播放（FFmpeg）
├── plugin/     → core_plugin_module     # 插件加载、ncbind
├── sound/      → core_sound_module      # 音频解码、DSP、波形管理  ← 本模块
├── tjs2/       → tjs2                   # TJS2 脚本引擎
├── utils/      → core_utils_module      # 定时器、线程、RNG
└── visual/     → core_visual_module     # 渲染、图像加载、图层
```

`core_sound_module` 的 CMake 目标名称为 `core_sound_module`，编译为 **STATIC** 库（静态链接到最终二进制文件中）。

### 模块间依赖关系

```
core_sound_module
├── PUBLIC:  tjs2                    # TJS2 脚本绑定（暴露给使用者）
└── PRIVATE: core_base_module        # I/O 流、存储接口
             core_movie_module       # 共享 FFmpeg 库
             core_utils_module       # 定时器、线程工具
             core_environ_module     # 平台抽象层
```

PUBLIC 依赖意味着链接 `core_sound_module` 的目标也会自动获得 `tjs2` 的头文件和链接。PRIVATE 依赖只在 `core_sound_module` 内部使用。

## 目录结构详解

`cpp/core/sound/` 目录包含 30 个文件，分为根目录（跨平台核心逻辑）和 `win32/` 子目录（平台实现层）：

```
cpp/core/sound/
│
│  ── 解码器相关 ──
├── WaveIntf.h              # 核心头文件：tTVPWaveDecoder 接口、tTVPWaveFormat 结构体、
│                           #   tTVPWaveDecoderCreator 工厂接口、PCM 转换函数声明
├── WaveIntf.cpp            # 核心实现：RIFF WAV 解码器、Creator 注册表、PCM 格式转换、
│                           #   tTJSNC_WaveSoundBuffer TJS2 绑定（1788 行）
├── FFWaveDecoder.h         # FFmpeg 通用解码器：FFWaveDecoder + FFWaveDecoderCreator
├── FFWaveDecoder.cpp       # FFmpeg 解码实现：avformat/avcodec 调用
├── VorbisWaveDecoder.h     # Ogg Vorbis + Opus 解码器头文件
├── VorbisWaveDecoder.cpp   # Vorbis (libvorbisfile) + Opus (libopusfile) 实现
│
│  ── 播放缓冲区基础 ──
├── SoundBufferBaseIntf.h   # tTJSNI_BaseSoundBuffer：状态机、音量、淡入淡出
├── SoundBufferBaseIntf.cpp # 基础实现：状态切换、Fade 定时器、事件投递
│
│  ── 循环与分段播放 ──
├── WaveLoopManager.h       # tTVPWaveLoopManager：循环点、标签、链接条件
├── WaveLoopManager.cpp     # 循环管理实现
├── WaveSegmentQueue.h      # tTVPWaveSegmentQueue：分段队列播放
├── WaveSegmentQueue.cpp    # 分段队列实现
│
│  ── DSP 处理 ──
├── PhaseVocoderDSP.h       # 相位声码器：时间拉伸不变调
├── PhaseVocoderDSP.cpp     # 相位声码器 DSP 核心算法
├── PhaseVocoderFilter.h    # 相位声码器滤波器封装
├── PhaseVocoderFilter.cpp  # 滤波器链集成
├── MathAlgorithms.h        # FFT 和窗函数
├── MathAlgorithms.cpp      # 数学算法实现
│
│  ── PCM 格式转换 ──
├── WaveFormatConverter.cpp # 通用 PCM 转换（Int16↔Float32 循环函数）
│  # WaveFormatConverter_SSE.cpp  # SSE 优化路径（已禁用）
│  # xmmlib.cpp                   # SSE 内存操作辅助（已禁用）
│
│  ── 环形缓冲区 ──
├── RingBuffer.h            # 模板环形缓冲区（用于 L2 缓冲）
│
│  ── 遗留模块（CD/MIDI 存根）──
├── CDDAIntf.h / .cpp       # CD 音频接口（存根实现）
├── MIDIIntf.h / .cpp       # MIDI 接口（存根实现）
│
│  ── 平台实现层（win32/ 子目录）──
├── win32/
│   ├── SoundBufferBaseImpl.h   # tTJSNI_SoundBuffer：平台层声音缓冲区
│   ├── SoundBufferBaseImpl.cpp # 定时器调度、缓冲区生命周期管理
│   ├── WaveImpl.h              # tTJSNI_WaveSoundBuffer：完整播放器（344 行）
│   ├── WaveImpl.cpp            # 双层缓冲、解码线程、OpenAL 输出
│   ├── WaveMixer.h             # iTVPSoundBuffer 接口、TVPCreateSoundBuffer 工厂
│   ├── WaveMixer.cpp           # OpenAL 混音器后端实现
│   ├── CDDAImpl.h / .cpp       # CD 音频平台实现（存根）
│   ├── MIDIImpl.h / .cpp       # MIDI 平台实现（存根）
│   ├── tvpsnd.h                # 旧版声音插件接口
│   └── tvpsnd.cpp              # 声音插件桥接
│
│  ── 构建与文档 ──
├── CMakeLists.txt          # 本模块的 CMake 构建配置
└── AGENTS.md               # 模块文档（AI 辅助开发用）
```

### 关于 `win32/` 子目录命名

`win32/` 这个名称是 KiriKiri 原版代码遗留的历史命名。在原版 KiriKiri2 中，这些文件确实只运行在 Windows 平台上，使用 DirectSound 进行音频输出。

在 KrKr2 移植版中，`win32/` 下的代码已经完全重写为跨平台实现：
- **WaveMixer.cpp** 使用 OpenAL（非 DirectSound）
- **WaveImpl.cpp** 不依赖任何 Windows API
- **SoundBufferBaseImpl.cpp** 使用通用定时器（非 Windows Timer）

因此 `win32/` 实际上是"平台实现层"的含义，在所有平台（Windows、Linux、macOS、Android）上都会编译和使用。

## CMakeLists.txt 逐行解析

完整的 CMakeLists.txt 只有 62 行，但信息密度很高。以下是逐段分析：

### 项目声明与源文件列表

```cmake
# 文件：cpp/core/sound/CMakeLists.txt 第 1-29 行
cmake_minimum_required(VERSION 3.19)
project(core_sound_module LANGUAGES CXX)

set(SOUND_PATH ${CMAKE_CURRENT_LIST_DIR})    # 当前目录的绝对路径
set(SOUND_SOURCE_FILES
    # ── 根目录：跨平台核心逻辑 ──
    ${SOUND_PATH}/CDDAIntf.cpp               # CD 音频接口（存根）
    ${SOUND_PATH}/FFWaveDecoder.cpp          # FFmpeg 通用解码器
    ${SOUND_PATH}/MathAlgorithms.cpp         # FFT、窗函数
    ${SOUND_PATH}/MIDIIntf.cpp               # MIDI 接口（存根）
    ${SOUND_PATH}/PhaseVocoderDSP.cpp        # 相位声码器核心
    ${SOUND_PATH}/PhaseVocoderFilter.cpp     # 相位声码器滤波器
    ${SOUND_PATH}/SoundBufferBaseIntf.cpp    # 声音缓冲区基类
    ${SOUND_PATH}/VorbisWaveDecoder.cpp      # Vorbis + Opus 解码器
    ${SOUND_PATH}/WaveFormatConverter.cpp    # PCM 格式转换
#   ${SOUND_PATH}/WaveFormatConverter_SSE.cpp  # SSE 优化（已禁用）
    ${SOUND_PATH}/WaveIntf.cpp               # 核心：注册表+绑定+RIFF
    ${SOUND_PATH}/WaveLoopManager.cpp        # 循环管理器
    ${SOUND_PATH}/WaveSegmentQueue.cpp       # 分段队列
#   ${SOUND_PATH}/xmmlib.cpp                 # SSE 内存辅助（已禁用）

    # ── win32/ 子目录：平台实现层 ──
    ${SOUND_PATH}/win32/CDDAImpl.cpp         # CD 平台实现
    ${SOUND_PATH}/win32/MIDIImpl.cpp         # MIDI 平台实现
    ${SOUND_PATH}/win32/SoundBufferBaseImpl.cpp  # 缓冲区定时器
#   ${SOUND_PATH}/win32/tvpsnd.c             # 旧版 C 接口（已禁用）
    ${SOUND_PATH}/win32/tvpsnd.cpp           # 声音插件桥接
    ${SOUND_PATH}/win32/WaveImpl.cpp         # 完整播放器
    ${SOUND_PATH}/win32/WaveMixer.cpp        # OpenAL 混音器
)
```

**注意被注释掉的文件：**
- `WaveFormatConverter_SSE.cpp` 和 `xmmlib.cpp` —— SSE SIMD 加速路径被禁用。当前使用纯 C++ 的标量转换循环。这是因为跨平台兼容性问题（ARM 平台不支持 SSE）
- `tvpsnd.c` —— 旧版 C 语言声音插件接口，已被 `tvpsnd.cpp` 替代

### 头文件路径与库目标

```cmake
# 文件：cpp/core/sound/CMakeLists.txt 第 31-38 行
set(SOUND_HEADERS_DIR
    ${SOUND_PATH}/             # 根目录头文件
    ${SOUND_PATH}/win32        # win32/ 子目录头文件
)

add_library(${PROJECT_NAME} STATIC ${SOUND_SOURCE_FILES})  # 静态库
target_include_directories(${PROJECT_NAME} PUBLIC ${SOUND_HEADERS_DIR})
target_link_libraries(${PROJECT_NAME}
    PUBLIC tjs2                                    # TJS2 脚本绑定
    PRIVATE core_base_module core_movie_module      # I/O + FFmpeg
            core_utils_module core_environ_module   # 工具 + 平台层
)
```

关键设计决策：
1. **两个 include 路径**都设为 PUBLIC，因此依赖 `core_sound_module` 的代码可以直接 `#include "WaveIntf.h"` 或 `#include "WaveImpl.h"`
2. `tjs2` 是 PUBLIC 依赖 —— 因为 `WaveIntf.h` 中的类继承自 `tTJSNativeInstance`（TJS2 的基类），使用者需要 TJS2 头文件
3. 其他 4 个模块是 PRIVATE —— 只在 `.cpp` 文件中引用，不暴露给外部

### 第三方库查找与链接

```cmake
# 文件：cpp/core/sound/CMakeLists.txt 第 40-62 行
find_package(Vorbis CONFIG REQUIRED)   # Ogg Vorbis 解码
find_package(Opus CONFIG REQUIRED)     # Opus 编解码器
find_package(OpusFile CONFIG REQUIRED) # Opus 文件读取

if(ANDROID)
    find_package(oboe CONFIG REQUIRED) # Android 低延迟音频
    target_link_libraries(${PROJECT_NAME} PRIVATE
        oboe::oboe
    )
endif()

find_package(OpenAL CONFIG REQUIRED)   # 跨平台音频输出

target_link_libraries(${PROJECT_NAME} PRIVATE
    Vorbis::vorbis         # Vorbis 核心库
    Vorbis::vorbisfile     # Vorbis 文件 I/O
    Vorbis::vorbisenc      # Vorbis 编码器（虽然只做解码，但链接需要）
    Opus::opus             # Opus 核心库
    OpusFile::opusfile     # Opus 文件 I/O
    OpenAL::OpenAL         # OpenAL 音频输出
)
```

所有第三方依赖通过 vcpkg 管理（在 `krkr2/vcpkg.json` 中声明），使用 CMake `CONFIG` 模式查找。

依赖关系总结：

| 库 | vcpkg 包名 | 用途 | 平台 |
|----|-----------|------|------|
| Vorbis | libvorbis | Ogg Vorbis 音频解码 | 全平台 |
| Opus | opus | Opus 音频解码 | 全平台 |
| OpusFile | opusfile | Opus 文件 I/O 封装 | 全平台 |
| OpenAL | openal-soft | 音频混音与输出 | 全平台 |
| Oboe | oboe | Android 低延迟音频输出 | 仅 Android |

### 平台条件编译细节

Android 平台额外链接 Oboe 库。Oboe 是 Google 开发的 C++ 音频库，提供低延迟音频输出，在 Android 10+ 上使用 AAudio 后端，低版本使用 OpenSL ES 后端：

```cmake
if(ANDROID)
    find_package(oboe CONFIG REQUIRED)
    target_link_libraries(${PROJECT_NAME} PRIVATE oboe::oboe)
endif()
```

注意这里使用的是项目自定义的 `ANDROID` 变量（在根 CMakeLists.txt 中设置），而非 CMake 内置的 `CMAKE_SYSTEM_NAME`。这是 KrKr2 项目的惯例。

## 动手实践

### 练习 1：添加新源文件到构建

假设你要添加一个新的 FLAC 解码器文件 `FLACWaveDecoder.cpp`，需要修改 CMakeLists.txt：

```cmake
# 在 SOUND_SOURCE_FILES 列表中添加：
set(SOUND_SOURCE_FILES
    # ... 现有文件 ...
    ${SOUND_PATH}/FLACWaveDecoder.cpp      # 新增：FLAC 解码器
    # ... 现有文件 ...
)
```

同时需要添加 FLAC 库依赖：

```cmake
find_package(FLAC CONFIG REQUIRED)

target_link_libraries(${PROJECT_NAME} PRIVATE
    # ... 现有依赖 ...
    FLAC::FLAC          # FLAC 解码库
)
```

并在 `vcpkg.json` 中添加依赖声明：

```json
{
  "dependencies": [
    "flac"
  ]
}
```

### 练习 2：查看实际的编译命令

在构建目录中，可以通过以下命令查看 `core_sound_module` 的实际编译参数：

```bash
# Windows
cmake --preset="Windows Debug Config"
cmake --build --preset="Windows Debug Build" --target core_sound_module --verbose

# Linux
cmake --preset="Linux Debug Config"
cmake --build --preset="Linux Debug Build" --target core_sound_module --verbose

# macOS
cmake --preset="MacOS Debug Config"
cmake --build --preset="MacOS Debug Build" --target core_sound_module --verbose

# Android（通过 Gradle）
./platforms/android/gradlew -p ./platforms/android assembleDebug --info
```

verbose 输出中可以看到：
- 每个 `.cpp` 文件的完整编译命令（包含所有 `-I` include 路径和 `-D` 宏定义）
- 链接阶段的库搜索路径和链接顺序

### 练习 3：分析依赖传播

利用 CMake 的 `cmake --graphviz` 功能生成依赖图：

```bash
# 在构建目录中执行
cmake --graphviz=deps.dot --preset="Linux Debug Config"
# 用 Graphviz 工具查看
dot -Tpng deps.dot -o deps.png
```

也可以手动检查传递依赖。在 `core_sound_module` 中：
- `tjs2` 是 PUBLIC → 链接 sound 的目标自动获得 tjs2 头文件
- `core_base_module` 是 PRIVATE → 链接 sound 的目标**不会**获得 base 模块头文件

```cmake
# 验证：在另一个模块中使用 sound
target_link_libraries(my_target PRIVATE core_sound_module)
# 此时 my_target 可以 #include "WaveIntf.h"（sound 头文件）
# 此时 my_target 可以 #include "tjsNative.h"（tjs2 头文件，因为 PUBLIC 传播）
# 此时 my_target 不能 #include "StorageIntf.h"（base 头文件，因为 PRIVATE 不传播）
```

## 对照项目源码

相关文件：
- `cpp/core/sound/CMakeLists.txt` 全文（62 行）—— 本节分析的核心文件
- `cpp/core/CMakeLists.txt` —— 上层 INTERFACE 库，链接所有 9 个子系统模块
- `vcpkg.json`（项目根目录）—— 声明所有第三方依赖的 vcpkg 清单
- `cmake/vcpkg_android.cmake` —— Android 平台的 vcpkg 工具链集成

sound 模块的编译产物：
- Windows: `out/windows/debug/cpp/core/sound/core_sound_module.lib`
- Linux: `out/linux/debug/cpp/core/sound/libcore_sound_module.a`
- macOS: `out/macos/debug/cpp/core/sound/libcore_sound_module.a`
- Android: 编译进最终的 `libkrkr2.so` 共享库

## 常见错误与解决方案

### 错误 1：找不到 OpenAL 头文件

```
fatal error: AL/al.h: No such file or directory
```

**原因：** vcpkg 未正确安装 openal-soft，或 CMake 没有使用 vcpkg 工具链。

**解决：**
```bash
# 确认 vcpkg 已安装
vcpkg install openal-soft

# 确认 CMake 配置使用了 vcpkg 工具链
cmake -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake ..
```

### 错误 2：链接时 Vorbis 符号未定义

```
undefined reference to `ov_open_callbacks'
```

**原因：** `Vorbis::vorbisfile` 未被链接，或链接顺序错误。

**解决：** 确认 CMakeLists.txt 中 `target_link_libraries` 包含了 `Vorbis::vorbisfile`。CMake 的 target 链接会自动处理顺序，但如果手动指定库文件路径，需注意依赖顺序（vorbisfile 依赖 vorbis 依赖 ogg）。

### 错误 3：Android 构建找不到 Oboe

```
CMake Error: Could not find a package configuration file provided by "oboe"
```

**原因：** Android 的 vcpkg triplet 没有安装 oboe 包。

**解决：**
```bash
# 使用 Android triplet 安装
vcpkg install oboe:arm64-android
# 或检查 vcpkg.json 中是否声明了 oboe
```

## 本节小结

- `core_sound_module` 编译为静态库，包含 15 个活跃源文件（2 个 SSE 文件被禁用）
- 目录分为根目录（跨平台核心：解码器、DSP、循环管理）和 `win32/`（平台实现层：播放器、混音器）
- `win32/` 名称是历史遗留，实际在所有平台上使用
- CMake 配置核心是 `add_library(STATIC)` + `target_link_libraries(PUBLIC/PRIVATE)` 控制依赖传播
- 第三方依赖：Vorbis、Opus、OpenAL（全平台）+ Oboe（仅 Android）
- 所有依赖通过 vcpkg CONFIG 模式查找

## 练习题与答案

### 题目 1：为什么 `tjs2` 是 PUBLIC 依赖而 `core_base_module` 是 PRIVATE？

<details>
<summary>查看答案</summary>

`tjs2` 是 PUBLIC 依赖，因为 `WaveIntf.h`（sound 模块的公开头文件）中定义的类继承自 TJS2 的基类：

```cpp
// WaveIntf.h 中的声明
class tTJSNI_BaseWaveSoundBuffer : public tTJSNI_BaseSoundBuffer {
    // tTJSNI_BaseSoundBuffer 继承自 tTJSNativeInstance（tjs2 库的类）
};
```

任何 `#include "WaveIntf.h"` 的代码都需要 TJS2 的头文件来解析这些基类声明，所以 tjs2 必须是 PUBLIC。

`core_base_module` 是 PRIVATE 依赖，因为 sound 模块只在 `.cpp` 文件中使用 base 模块的功能（如 `StorageIntf.h` 读取文件流、`EventIntf.h` 投递事件），这些都不出现在 sound 模块的公开头文件中。外部代码使用 sound 模块时不需要访问 base 模块的接口。

</details>

### 题目 2：如果要添加 MP3 解码支持（使用 minimp3 库），需要修改哪些文件？写出具体步骤。

<details>
<summary>查看答案</summary>

需要修改 3 个文件，新建 2 个文件：

**步骤 1：创建解码器源文件**

```cpp
// 新建 cpp/core/sound/MP3WaveDecoder.h
#pragma once
#include "WaveIntf.h"

class MP3WaveDecoder : public tTVPWaveDecoder {
    // minimp3 解码上下文
    void *DecContext;
    tTVPWaveFormat Format;
    tjs_uint64 CurrentPos;

public:
    MP3WaveDecoder(tTJSBinaryStream *stream);
    ~MP3WaveDecoder() override;
    void GetFormat(tTVPWaveFormat &format) override;
    bool Render(void *buf, tjs_uint bufsamplelen, tjs_uint &rendered) override;
    bool SetPosition(tjs_uint64 samplepos) override;
};

class MP3WaveDecoderCreator : public tTVPWaveDecoderCreator {
public:
    tTVPWaveDecoder *Create(const ttstr &storagename,
                            const ttstr &extension) override;
};
```

```cpp
// 新建 cpp/core/sound/MP3WaveDecoder.cpp
#include "MP3WaveDecoder.h"
#define MINIMP3_IMPLEMENTATION
#include <minimp3_ex.h>
// ... 实现解码逻辑 ...
```

**步骤 2：修改 CMakeLists.txt**

```cmake
# 在 SOUND_SOURCE_FILES 中添加
${SOUND_PATH}/MP3WaveDecoder.cpp

# 添加 minimp3 依赖（如果通过 vcpkg 安装）
find_package(minimp3 CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE minimp3::minimp3)
```

**步骤 3：在注册表中注册**

修改 `WaveIntf.cpp` 中的 `tTVPWaveDecoderManager` 构造函数：

```cpp
#include "MP3WaveDecoder.h"  // 新增 include

struct tTVPWaveDecoderManager {
    // ... 现有成员 ...
    MP3WaveDecoderCreator mp3WaveDecoderCreator;  // 新增

    tTVPWaveDecoderManager() {
        TVPWaveDecoderManagerAvail = true;
        TVPRegisterWaveDecoderCreator(&mp3WaveDecoderCreator);  // 新增
        TVPRegisterWaveDecoderCreator(&ffWaveDecoderCreator);
        // ... 其他注册 ...
    }
};
```

**步骤 4：更新 vcpkg.json**

```json
{
  "dependencies": [
    "minimp3"
  ]
}
```

</details>

### 题目 3：SSE 优化路径被禁用后，PCM 格式转换使用的是什么实现？在哪个文件？

<details>
<summary>查看答案</summary>

SSE 路径（`WaveFormatConverter_SSE.cpp`）禁用后，使用的是 `WaveFormatConverter.cpp` 中的纯 C++ 标量循环实现。具体是两个函数指针：

```cpp
// WaveFormatConverter.cpp 中定义
void (*PCMConvertLoopInt16ToFloat32)(void *dest, const void *src, size_t numsamples);
void (*PCMConvertLoopFloat32ToInt16)(void *dest, const void *src, size_t numsamples);
```

这些函数指针在初始化时指向标量实现。在 `WaveIntf.cpp` 中被 `extern` 声明后直接调用：

```cpp
// WaveIntf.cpp 第 83-87 行
extern void (*PCMConvertLoopInt16ToFloat32)(void *dest, const void *src,
                                            size_t numsamples);
extern void (*PCMConvertLoopFloat32ToInt16)(void *dest, const void *src,
                                            size_t numsamples);
```

转换函数的调用链：
1. `TVPConvertPCMTo16bits()` → 调用 `TVPConvertFloatPCMTo16bits()` → 调用 `PCMConvertLoopFloat32ToInt16`
2. `TVPConvertPCMToFloat()` → 调用 `TVPConvertIntegerPCMToFloat()` → 调用 `PCMConvertLoopInt16ToFloat32`

如果将来要在 ARM 平台启用 NEON 优化，可以提供 NEON 版本的这两个函数并在运行时检测 CPU 特性后切换函数指针。

</details>

## 下一步

下一节 [核心类层次与继承链](./02-核心类层次与继承链.md) 将深入分析 sound 模块中从 `tTJSNativeInstance` 到 `tTJSNI_WaveSoundBuffer` 的完整类继承体系，理解每一层的职责划分。
