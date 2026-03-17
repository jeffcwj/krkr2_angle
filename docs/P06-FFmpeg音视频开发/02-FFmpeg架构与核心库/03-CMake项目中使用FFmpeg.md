# 四、在 CMake 项目中使用 FFmpeg
## 四、在 CMake 项目中使用 FFmpeg

### 4.1 通过 vcpkg 安装

KrKr2 使用 vcpkg 管理 FFmpeg 依赖。在 `vcpkg.json` 中：

```json
{
    "dependencies": [
        {
            "name": "ffmpeg",
            "features": ["avcodec", "avformat", "avfilter",
                         "swscale", "swresample"]
        }
    ]
}
```

### 4.2 CMake 中查找和链接

**方法一：使用 `pkg-config`（通用方式）：**

```cmake
cmake_minimum_required(VERSION 3.20)
project(ffmpeg_demo LANGUAGES CXX)

find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBAV REQUIRED IMPORTED_TARGET
    libavformat
    libavcodec
    libavutil
    libswscale
    libswresample
    libavfilter
)

add_executable(demo main.cpp)
target_link_libraries(demo PRIVATE PkgConfig::LIBAV)
```

**方法二：使用 vcpkg 的 FFmpeg CMake 集成：**

```cmake
cmake_minimum_required(VERSION 3.20)
project(ffmpeg_demo LANGUAGES CXX)

# vcpkg 会自动设置 CMAKE_TOOLCHAIN_FILE
find_package(PkgConfig REQUIRED)

# vcpkg 安装的 FFmpeg 同样通过 pkg-config 发现
pkg_check_modules(FFMPEG REQUIRED IMPORTED_TARGET
    libavformat
    libavcodec
    libavutil
    libswscale
    libswresample
)

add_executable(demo main.cpp)
target_link_libraries(demo PRIVATE PkgConfig::FFMPEG)
```

**方法三：手动查找（当 pkg-config 不可用时）：**

```cmake
cmake_minimum_required(VERSION 3.20)
project(ffmpeg_demo LANGUAGES CXX)

# 逐个查找 FFmpeg 库
find_path(AVFORMAT_INCLUDE_DIR libavformat/avformat.h)
find_library(AVFORMAT_LIBRARY avformat)
find_path(AVCODEC_INCLUDE_DIR libavcodec/avcodec.h)
find_library(AVCODEC_LIBRARY avcodec)
find_path(AVUTIL_INCLUDE_DIR libavutil/avutil.h)
find_library(AVUTIL_LIBRARY avutil)
find_library(SWSCALE_LIBRARY swscale)
find_library(SWRESAMPLE_LIBRARY swresample)

add_executable(demo main.cpp)
target_include_directories(demo PRIVATE
    ${AVFORMAT_INCLUDE_DIR}
    ${AVCODEC_INCLUDE_DIR}
    ${AVUTIL_INCLUDE_DIR}
)
target_link_libraries(demo PRIVATE
    ${AVFORMAT_LIBRARY}
    ${AVCODEC_LIBRARY}
    ${SWSCALE_LIBRARY}
    ${SWRESAMPLE_LIBRARY}
    ${AVUTIL_LIBRARY}
)
```

### 4.3 跨平台注意事项

| 平台 | 安装方式 | 注意事项 |
|------|---------|---------|
| **Windows** | vcpkg 或预编译二进制 | 需要同时部署 `.dll` 文件；静态链接需额外链接 `bcrypt`, `secur32`, `ws2_32` |
| **Linux** | vcpkg / apt / pacman | `apt install libavformat-dev libavcodec-dev ...`；注意版本（Ubuntu 22.04 默认 FFmpeg 4.x） |
| **macOS** | vcpkg / Homebrew | `brew install ffmpeg`；注意 Apple Silicon 的 Homebrew 路径 `/opt/homebrew/` |
| **Android** | vcpkg（交叉编译） | 使用 vcpkg 的 Android triplet；需要在 Gradle 中配置 CMake 路径 |

**Windows 静态链接的额外依赖：**

```cmake
if(WIN32)
    # FFmpeg 静态链接时需要这些 Windows 系统库
    target_link_libraries(demo PRIVATE
        bcrypt      # 加密
        secur32     # 安全
        ws2_32      # 网络
        Mfplat      # Media Foundation（某些硬件解码器需要）
        Mfuuid
        Strmiids    # DirectShow GUID
    )
endif()
```

**Android 的 CMake 配置：**

```cmake
if(ANDROID)
    # Android NDK 不自带 FFmpeg，通过 vcpkg 安装
    # vcpkg 的 Android triplet 会自动处理交叉编译
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(FFMPEG REQUIRED IMPORTED_TARGET
        libavformat libavcodec libavutil
        libswscale libswresample
    )
    target_link_libraries(demo PRIVATE
        PkgConfig::FFMPEG
        log         # Android 日志库
        android     # Android NDK 库
    )
endif()
```

---

## 五、常见错误与排查

### 错误 1：链接顺序导致 undefined reference

```
/usr/bin/ld: libavformat.a: undefined reference to `avcodec_parameters_copy'
```

**原因**：静态链接时，链接器从左到右处理库。如果 `avformat` 在 `avcodec` 后面，`avformat` 依赖的 `avcodec` 符号无法解析。

**解决**：确保链接顺序从高层到底层：
```cmake
target_link_libraries(demo PRIVATE avformat avcodec swscale swresample avutil)
```

### 错误 2：头文件路径不对

```
fatal error: libavformat/avformat.h: No such file or directory
```

**解决**：FFmpeg 头文件使用 `libavXXX/` 前缀，需要将 FFmpeg 的 include 目录添加到搜索路径：
```cmake
# 正确的 include 方式
#include <libavformat/avformat.h>   # ✅
#include <avformat.h>                # ❌ 缺少子目录前缀
```

### 错误 3：API 版本不匹配

```
error: 'avcodec_decode_video2' was not declared in this scope
```

**原因**：使用了旧 API，但 FFmpeg 版本 ≥ 5.0 已移除。

**解决**：使用新式 API：
```cpp
// 旧 → 新
avcodec_decode_video2()  →  avcodec_send_packet() + avcodec_receive_frame()
avcodec_decode_audio4()  →  avcodec_send_packet() + avcodec_receive_frame()
avpicture_fill()         →  av_image_fill_arrays()
avpicture_get_size()     →  av_image_get_buffer_size()
```

---

