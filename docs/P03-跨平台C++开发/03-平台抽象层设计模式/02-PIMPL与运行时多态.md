## 模式二：PIMPL（指针到实现）

PIMPL（Pointer to Implementation）将接口与实现分离，通过前向声明和指针隐藏平台特定的细节：

```
┌──────────────────────┐
│    FileHandle.h       │ ← 公共头文件（无平台细节）
│  class FileHandle {   │
│    struct Impl;       │ ← 前向声明
│    Impl* pImpl;       │ ← 指向具体实现的指针
│  };                   │
└──────────────────────┘
          │
    ┌─────┴──────┐
    ▼            ▼
┌─────────┐  ┌─────────┐
│ Win32   │  │ POSIX   │
│ Impl    │  │ Impl    │  ← 平台特定的 Impl 定义
│(HANDLE) │  │(int fd) │    在 .cpp 文件中，不暴露给使用者
└─────────┘  └─────────┘
```

### 完整示例

```cpp
// file_handle.h — 公共头文件
#pragma once
#include <cstdint>
#include <string>
#include <memory>

class FileHandle {
public:
    explicit FileHandle(const std::string& path);
    ~FileHandle();

    // 禁止拷贝，允许移动
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    FileHandle(FileHandle&& other) noexcept;
    FileHandle& operator=(FileHandle&& other) noexcept;

    bool isValid() const;
    size_t read(void* buffer, size_t size);
    size_t write(const void* buffer, size_t size);
    int64_t size() const;
    void close();

private:
    struct Impl;                      // 前向声明——不需要知道 Impl 的具体定义
    std::unique_ptr<Impl> pImpl_;     // 指向实现的智能指针
};
```

```cpp
// file_handle_win32.cpp — Windows 实现
#include "file_handle.h"
#include <windows.h>

struct FileHandle::Impl {
    HANDLE hFile = INVALID_HANDLE_VALUE;

    explicit Impl(const std::string& path) {
        std::wstring wpath(path.begin(), path.end()); // 简化版转换
        hFile = CreateFileW(wpath.c_str(), GENERIC_READ | GENERIC_WRITE,
                            0, nullptr, OPEN_EXISTING, 0, nullptr);
    }

    ~Impl() {
        if (hFile != INVALID_HANDLE_VALUE) {
            CloseHandle(hFile);
        }
    }

    bool isValid() const { return hFile != INVALID_HANDLE_VALUE; }

    size_t read(void* buffer, size_t size) {
        DWORD bytesRead = 0;
        ReadFile(hFile, buffer, static_cast<DWORD>(size), &bytesRead, nullptr);
        return bytesRead;
    }

    size_t write(const void* buffer, size_t size) {
        DWORD bytesWritten = 0;
        WriteFile(hFile, buffer, static_cast<DWORD>(size), &bytesWritten, nullptr);
        return bytesWritten;
    }

    int64_t fileSize() const {
        LARGE_INTEGER li;
        GetFileSizeEx(hFile, &li);
        return li.QuadPart;
    }
};

FileHandle::FileHandle(const std::string& path)
    : pImpl_(std::make_unique<Impl>(path)) {}

FileHandle::~FileHandle() = default;

FileHandle::FileHandle(FileHandle&& other) noexcept = default;
FileHandle& FileHandle::operator=(FileHandle&& other) noexcept = default;

bool FileHandle::isValid() const { return pImpl_->isValid(); }
size_t FileHandle::read(void* buffer, size_t size) { return pImpl_->read(buffer, size); }
size_t FileHandle::write(const void* buffer, size_t size) { return pImpl_->write(buffer, size); }
int64_t FileHandle::size() const { return pImpl_->fileSize(); }
void FileHandle::close() { pImpl_.reset(); }
```

```cpp
// file_handle_posix.cpp — Linux/macOS/Android 实现
#include "file_handle.h"
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>

struct FileHandle::Impl {
    int fd = -1;

    explicit Impl(const std::string& path) {
        fd = open(path.c_str(), O_RDWR);
    }

    ~Impl() {
        if (fd >= 0) ::close(fd);
    }

    bool isValid() const { return fd >= 0; }

    size_t read(void* buffer, size_t size) {
        ssize_t result = ::read(fd, buffer, size);
        return result > 0 ? static_cast<size_t>(result) : 0;
    }

    size_t write(const void* buffer, size_t size) {
        ssize_t result = ::write(fd, buffer, size);
        return result > 0 ? static_cast<size_t>(result) : 0;
    }

    int64_t fileSize() const {
        struct stat st;
        if (fstat(fd, &st) != 0) return -1;
        return st.st_size;
    }
};

FileHandle::FileHandle(const std::string& path)
    : pImpl_(std::make_unique<Impl>(path)) {}

FileHandle::~FileHandle() = default;

FileHandle::FileHandle(FileHandle&& other) noexcept = default;
FileHandle& FileHandle::operator=(FileHandle&& other) noexcept = default;

bool FileHandle::isValid() const { return pImpl_->isValid(); }
size_t FileHandle::read(void* buffer, size_t size) { return pImpl_->read(buffer, size); }
size_t FileHandle::write(const void* buffer, size_t size) { return pImpl_->write(buffer, size); }
int64_t FileHandle::size() const { return pImpl_->fileSize(); }
void FileHandle::close() { pImpl_.reset(); }
```

### PIMPL 的优缺点

| 方面 | 优点 | 缺点 |
|------|------|------|
| 头文件隔离 | 平台头文件（如 `<windows.h>`）不会泄露到使用者 | 需要额外的指针间接层 |
| ABI 稳定性 | 修改 Impl 不影响使用者的二进制兼容性 | 堆分配 Impl 有轻微性能开销 |
| 编译速度 | 修改实现不触发使用者重新编译 | 实现代码有模板代码（转发函数） |

**适用场景：** 提供 SDK/库给第三方使用时、需要 ABI 稳定性时、平台头文件特别"脏"时（如 `<windows.h>` 会引入大量宏污染）。

---

## 模式三：运行时多态（接口+工厂）

使用抽象基类定义接口，运行时根据平台创建具体实现：

```cpp
// audio_backend.h — 抽象接口
#pragma once
#include <memory>
#include <string>
#include <cstdint>

class IAudioBackend {
public:
    virtual ~IAudioBackend() = default;

    virtual bool init(int sampleRate, int channels) = 0;
    virtual bool play(const int16_t* data, size_t sampleCount) = 0;
    virtual void stop() = 0;
    virtual float getVolume() const = 0;
    virtual void setVolume(float vol) = 0;

    // 工厂方法：根据平台创建合适的后端
    static std::unique_ptr<IAudioBackend> create();
};
```

```cpp
// audio_openal.cpp — OpenAL 实现（所有平台可用）
#include "audio_backend.h"
#include <AL/al.h>
#include <AL/alc.h>

class OpenALBackend : public IAudioBackend {
    ALCdevice* device_ = nullptr;
    ALCcontext* context_ = nullptr;
    ALuint source_ = 0;

public:
    bool init(int sampleRate, int channels) override {
        device_ = alcOpenDevice(nullptr);
        if (!device_) return false;
        context_ = alcCreateContext(device_, nullptr);
        alcMakeContextCurrent(context_);
        alGenSources(1, &source_);
        return true;
    }

    bool play(const int16_t* data, size_t sampleCount) override {
        ALuint buffer;
        alGenBuffers(1, &buffer);
        alBufferData(buffer, AL_FORMAT_STEREO16, data,
                     sampleCount * sizeof(int16_t), 44100);
        alSourceQueueBuffers(source_, 1, &buffer);
        alSourcePlay(source_);
        return true;
    }

    void stop() override { alSourceStop(source_); }
    float getVolume() const override { float v; alGetSourcef(source_, AL_GAIN, &v); return v; }
    void setVolume(float vol) override { alSourcef(source_, AL_GAIN, vol); }

    ~OpenALBackend() override {
        if (source_) alDeleteSources(1, &source_);
        if (context_) { alcMakeContextCurrent(nullptr); alcDestroyContext(context_); }
        if (device_) alcCloseDevice(device_);
    }
};
```

```cpp
// audio_wasapi.cpp — Windows WASAPI 实现
#include "audio_backend.h"
#ifdef _WIN32
#include <Audioclient.h>
#include <Mmdeviceapi.h>

class WASAPIBackend : public IAudioBackend {
    // Windows 低延迟音频实现
    // ...
};
#endif
```

```cpp
// audio_backend.cpp — 工厂方法
#include "audio_backend.h"

std::unique_ptr<IAudioBackend> IAudioBackend::create() {
#ifdef _WIN32
    // Windows 优先使用 WASAPI（低延迟），回退到 OpenAL
    auto wasapi = std::make_unique<WASAPIBackend>();
    if (wasapi->init(44100, 2)) return wasapi;
    return std::make_unique<OpenALBackend>();
#elif defined(__ANDROID__)
    // Android 使用 OpenSL ES 或 AAudio
    return std::make_unique<OpenSLESBackend>();
#else
    // Linux/macOS 使用 OpenAL
    return std::make_unique<OpenALBackend>();
#endif
}
```

### 运行时多态的优缺点

| 方面 | 优点 | 缺点 |
|------|------|------|
| 灵活性 | 可以运行时切换后端实现 | vtable 间接调用有轻微开销 |
| 扩展性 | 添加新后端只需添加子类 | 接口设计必须足够通用 |
| 测试 | 可以注入 Mock 实现做单元测试 | 接口变更影响所有实现 |
| 清晰度 | 接口与实现完全分离 | 类层次结构增加理解成本 |

**适用场景：** 同一平台上可能有多种实现可选时（如音频后端、渲染后端）、需要单元测试 Mock 时。

---

## 三种模式对比

| 特性 | 文件分离 | PIMPL | 运行时多态 |
|------|---------|-------|-----------|
| 实现复杂度 | ★☆☆ | ★★☆ | ★★★ |
| 运行时开销 | 零 | 指针间接 | vtable 间接 |
| 运行时切换 | ❌ | ❌ | ✅ |
| ABI 稳定性 | 低 | 高 | 中 |
| 头文件隔离 | 中 | 高 | 中 |
| 可测试性 | 中 | 中 | 高 |
| KrKr2 使用 | ✅ 主要 | 少量 | 少量 |

**KrKr2 的选择：** 项目以**文件分离**为主。原因是：
1. 项目不需要运行时切换平台实现（编译时就确定了目标平台）
2. 自由函数比类层次更简单，符合原版 KiriKiri 引擎的 C 风格 API
3. 四个平台的差异主要在实现细节，接口层面高度一致

---

