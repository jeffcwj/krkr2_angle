# PIMPL 与运行时多态

> **所属模块：** P03-跨平台C++开发
> **前置知识：** [PAL 概念与编译期文件分离](./01-PAL概念与编译期文件分离.md)
> **预计阅读时间：** 20 分钟

## 本节目标

读完本节后，你将能够：
1. 理解 PIMPL（Pointer to Implementation，指针到实现）模式的核心思想——用前向声明 + 指针隐藏平台细节
2. 用 `std::unique_ptr` 实现一个跨平台的 PIMPL 类，分别为 Windows 和 POSIX 编写实现
3. 理解运行时多态（抽象基类 + 工厂方法）在跨平台开发中的应用场景
4. 对比文件分离、PIMPL、运行时多态三种模式的适用场景，判断 KrKr2 为何以文件分离为主

---

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

## 动手实践

### 实践 1：用 PIMPL 封装一个跨平台计时器

创建一个 `Timer` 类，在 Windows 上使用 `QueryPerformanceCounter`，在 POSIX 上使用 `clock_gettime`。头文件不暴露任何平台细节：

**公共头文件 `timer.h`：**

```cpp
// timer.h — 跨平台高精度计时器（PIMPL 模式）
#pragma once
#include <memory>

class Timer {
public:
    Timer();                         // 创建并启动计时器
    ~Timer();                        // 析构（释放 Impl）

    Timer(Timer&& other) noexcept;   // 移动构造
    Timer& operator=(Timer&& other) noexcept;  // 移动赋值

    Timer(const Timer&) = delete;    // 禁止拷贝
    Timer& operator=(const Timer&) = delete;

    void reset();                    // 重置计时器
    double elapsedMs() const;        // 返回从上次 reset 到现在的毫秒数

private:
    struct Impl;                     // 前向声明——使用者不需要知道 Impl 的定义
    std::unique_ptr<Impl> pImpl_;    // 指向平台特定实现的指针
};
```

**Windows 实现 `timer_win32.cpp`：**

```cpp
// timer_win32.cpp — Windows 高精度计时器实现
#include "timer.h"
#include <windows.h>

struct Timer::Impl {
    LARGE_INTEGER frequency;   // 计时器频率（每秒多少次计数）
    LARGE_INTEGER startTime;   // 起始计数值

    Impl() {
        // QueryPerformanceFrequency 获取高精度计时器的频率
        // 在现代 Windows 上通常返回 10,000,000（10 MHz）
        QueryPerformanceFrequency(&frequency);
        QueryPerformanceCounter(&startTime);
    }

    void reset() {
        QueryPerformanceCounter(&startTime);  // 记录当前计数值
    }

    double elapsedMs() const {
        LARGE_INTEGER now;
        QueryPerformanceCounter(&now);
        // 计算差值并转换为毫秒
        return static_cast<double>(now.QuadPart - startTime.QuadPart)
             * 1000.0 / static_cast<double>(frequency.QuadPart);
    }
};

// 构造/析构转发——注意 ~Timer() 必须在 Impl 定义可见处实现
Timer::Timer() : pImpl_(std::make_unique<Impl>()) {}
Timer::~Timer() = default;
Timer::Timer(Timer&& other) noexcept = default;
Timer& Timer::operator=(Timer&& other) noexcept = default;

void Timer::reset() { pImpl_->reset(); }
double Timer::elapsedMs() const { return pImpl_->elapsedMs(); }
```

**POSIX 实现 `timer_posix.cpp`：**

```cpp
// timer_posix.cpp — POSIX 高精度计时器实现（Linux/macOS/Android）
#include "timer.h"
#include <time.h>

struct Timer::Impl {
    struct timespec startTime;  // POSIX 时间结构（秒 + 纳秒）

    Impl() {
        // CLOCK_MONOTONIC 不受系统时间修改影响，适合测量时间间隔
        clock_gettime(CLOCK_MONOTONIC, &startTime);
    }

    void reset() {
        clock_gettime(CLOCK_MONOTONIC, &startTime);
    }

    double elapsedMs() const {
        struct timespec now;
        clock_gettime(CLOCK_MONOTONIC, &now);
        // 将秒差 + 纳秒差合并为毫秒
        double sec = static_cast<double>(now.tv_sec - startTime.tv_sec);
        double nsec = static_cast<double>(now.tv_nsec - startTime.tv_nsec);
        return sec * 1000.0 + nsec / 1000000.0;
    }
};

Timer::Timer() : pImpl_(std::make_unique<Impl>()) {}
Timer::~Timer() = default;
Timer::Timer(Timer&& other) noexcept = default;
Timer& Timer::operator=(Timer&& other) noexcept = default;

void Timer::reset() { pImpl_->reset(); }
double Timer::elapsedMs() const { return pImpl_->elapsedMs(); }
```

**CMakeLists.txt 平台选择：**

```cmake
cmake_minimum_required(VERSION 3.22)
project(pimpl_timer)

add_executable(timer_demo main.cpp)

# 根据平台选择实现文件
if(WIN32)
    target_sources(timer_demo PRIVATE timer_win32.cpp)
else()
    target_sources(timer_demo PRIVATE timer_posix.cpp)
endif()
```

**测试 `main.cpp`：**

```cpp
#include "timer.h"
#include <iostream>
#include <thread>
#include <chrono>

int main() {
    Timer t;

    // 等待 100 毫秒
    std::this_thread::sleep_for(std::chrono::milliseconds(100));

    double elapsed = t.elapsedMs();
    std::cout << "经过时间: " << elapsed << " ms" << std::endl;
    // 预期输出：经过时间: 100.xxx ms（误差通常在 ±5ms 以内）

    t.reset();
    std::this_thread::sleep_for(std::chrono::milliseconds(50));
    std::cout << "重置后经过: " << t.elapsedMs() << " ms" << std::endl;
    // 预期输出：重置后经过: 50.xxx ms

    return 0;
}
```

### 实践 2：观察 PIMPL 的头文件隔离效果

在上面的 `timer.h` 中，注意 **`<windows.h>` 和 `<time.h>` 都不会出现**。这意味着 `#include "timer.h"` 的代码不会受到 Windows SDK 宏污染（如 `DeleteFile`、`CreateFile`、`min`/`max` 等被定义为宏）的影响。

你可以验证这一点：在 `main.cpp` 中尝试定义一个名为 `min` 的函数——如果 `<windows.h>` 被泄露，这个函数名会和宏冲突导致编译报错；用了 PIMPL 后就不会。

---

## 常见错误及解决方案

### 错误 1：在头文件中定义 PIMPL 析构函数

```cpp
// ❌ 错误：在头文件中内联 ~Timer() = default;
// timer.h
class Timer {
    struct Impl;
    std::unique_ptr<Impl> pImpl_;
public:
    Timer();
    ~Timer() = default;  // 编译错误！此处 Impl 是不完整类型
};
```

**原因：** `std::unique_ptr<Impl>` 的析构函数需要知道 `Impl` 的完整定义才能调用 `delete`。在头文件中 `Impl` 只有前向声明，是不完整类型。

**解决方案：** 把 `~Timer()` 的定义放到 `.cpp` 文件中（`Impl` 已完整定义的地方）：

```cpp
// timer.h — 只声明
~Timer();

// timer_win32.cpp — 在 Impl 定义之后实现
Timer::~Timer() = default;  // ✅ 此处 Impl 已完整定义
```

### 错误 2：忘记提供移动操作

```cpp
// ❌ 如果不声明移动构造/赋值，Timer 就无法放入 std::vector 等容器
std::vector<Timer> timers;
timers.push_back(Timer{});  // 编译错误：Timer 不可拷贝也不可移动
```

**解决方案：** 在头文件中声明移动操作，在 `.cpp` 中 `= default`：

```cpp
// timer.h
Timer(Timer&& other) noexcept;
Timer& operator=(Timer&& other) noexcept;

// timer_win32.cpp
Timer::Timer(Timer&& other) noexcept = default;
Timer& Timer::operator=(Timer&& other) noexcept = default;
```

### 错误 3：用裸指针 `Impl*` 代替 `unique_ptr<Impl>`

```cpp
// ❌ 裸指针需要手动管理内存
class Timer {
    struct Impl;
    Impl* pImpl_;  // 忘记 delete 就会内存泄漏
};
```

**解决方案：** 始终使用 `std::unique_ptr<Impl>`。它自动管理生命周期，且不增加运行时开销（`unique_ptr` 的大小与裸指针相同）。

---

## 本节小结

- **PIMPL 模式**通过前向声明 + 指针将平台头文件的依赖封锁在 `.cpp` 中，使用者只看到干净的接口头文件
- **核心优势**：头文件隔离（避免宏污染）、ABI 稳定性（修改 Impl 不影响调用方二进制）、编译隔离（修改实现不触发使用方重编译）
- **代价**：每次调用多一层指针间接（性能影响通常可忽略）、堆上分配 Impl 对象、需要编写转发函数
- **运行时多态**（接口+工厂）比 PIMPL 更灵活——支持运行时切换实现、可注入 Mock 做单元测试。代价是 vtable 间接调用和更复杂的类层次
- **三种模式的选择原则**：编译期平台已确定 → 文件分离（最简单）；需要隐藏平台头文件 → PIMPL；需要运行时切换或 Mock 测试 → 运行时多态
- **KrKr2 以文件分离为主**，因为编译时即确定目标平台，且自由函数风格与原版 KiriKiri 引擎的 C 风格 API 一致

---

## 练习题与答案

### 题目 1：PIMPL 析构函数问题

以下代码会在哪一步产生编译错误？请解释原因并修复。

```cpp
// widget.h
#pragma once
#include <memory>

class Widget {
public:
    Widget();
    // ~Widget() 没有声明——编译器会在头文件中隐式生成
private:
    struct Impl;
    std::unique_ptr<Impl> pImpl_;
};
```

<details>
<summary>查看答案</summary>

**编译错误位置**：在任何 `#include "widget.h"` 的 `.cpp` 文件中，当编译器尝试生成 `Widget` 的隐式析构函数时报错。

**原因**：编译器在头文件中生成 `~Widget()` 的代码时，需要调用 `unique_ptr<Impl>` 的析构函数。而 `unique_ptr` 的析构函数内部需要 `delete pImpl_`，这要求 `Impl` 是完整类型。但在头文件中 `Impl` 只有前向声明，是不完整类型。

**修复**：显式声明析构函数，把定义放到 `Impl` 可见的 `.cpp` 中：

```cpp
// widget.h
class Widget {
public:
    Widget();
    ~Widget();  // ✅ 声明但不在此定义
private:
    struct Impl;
    std::unique_ptr<Impl> pImpl_;
};

// widget.cpp
struct Widget::Impl { /* ... */ };
Widget::Widget() : pImpl_(std::make_unique<Impl>()) {}
Widget::~Widget() = default;  // ✅ 此处 Impl 完整可见
```

</details>

### 题目 2：PIMPL vs 运行时多态

KrKr2 的音频模块在不同平台使用不同后端（Windows 用 WASAPI，Android 用 OpenSL ES，Linux/macOS 用 OpenAL）。如果你要设计这个模块，你会选择 PIMPL 还是运行时多态？请给出理由。

<details>
<summary>查看答案</summary>

**推荐：运行时多态（接口+工厂）。** 理由：

1. **同一平台可能有多种后端**：Windows 上可能同时支持 WASAPI（低延迟）和 OpenAL（兼容性好），需要运行时根据硬件能力选择。PIMPL 在编译期就确定了唯一实现，无法做到这一点

2. **回退机制**：如果首选后端初始化失败，需要回退到备选方案。运行时多态的工厂方法可以自然地实现 `try WASAPI → fallback to OpenAL` 的逻辑

3. **测试友好**：可以创建 `MockAudioBackend` 实现 `IAudioBackend` 接口，在单元测试中验证音频调用逻辑而不需要真实音频硬件

4. **实际代码印证**：本节示例中的 `IAudioBackend` 就展示了这种模式——工厂方法 `create()` 根据平台和初始化结果选择合适的后端

**PIMPL 不合适的原因**：PIMPL 只隐藏实现细节，不支持运行时切换。如果用 PIMPL，就必须在编译期决定使用哪个后端，丢失了运行时回退的能力。

**不过**，如果确定每个平台只有一个音频实现且不需要 Mock 测试，PIMPL 也是可行的选择——更简单、没有 vtable 开销。

</details>

### 题目 3：手写一个 PIMPL 的网络 Socket 封装

设计一个 `Socket` 类的头文件（只需头文件，不需要实现），使用 PIMPL 模式隐藏 Windows 的 `SOCKET` 和 POSIX 的 `int fd`。要求支持：构造（指定 IP 和端口）、发送、接收、关闭、移动语义。

<details>
<summary>查看答案</summary>

```cpp
// socket.h — 跨平台 TCP Socket（PIMPL 模式）
#pragma once
#include <memory>
#include <string>
#include <cstdint>
#include <cstddef>

class Socket {
public:
    // 构造：连接到指定地址和端口
    Socket(const std::string& host, uint16_t port);
    ~Socket();

    // 禁止拷贝（Socket 持有独占资源）
    Socket(const Socket&) = delete;
    Socket& operator=(const Socket&) = delete;

    // 允许移动（转移所有权）
    Socket(Socket&& other) noexcept;
    Socket& operator=(Socket&& other) noexcept;

    // 发送数据，返回实际发送的字节数（-1 表示错误）
    int send(const void* data, size_t length);

    // 接收数据，返回实际接收的字节数（0 表示对端关闭，-1 表示错误）
    int recv(void* buffer, size_t maxLength);

    // 关闭连接
    void close();

    // 检查连接是否有效
    bool isConnected() const;

private:
    struct Impl;                     // 前向声明
    std::unique_ptr<Impl> pImpl_;    // Windows: 内部持有 SOCKET
                                     // POSIX:   内部持有 int fd
};
```

**关键设计点：**
- `struct Impl;` 前向声明——在 Windows `.cpp` 中 `Impl` 包含 `SOCKET`，在 POSIX `.cpp` 中包含 `int fd`
- 禁止拷贝（网络连接不能被复制）
- 允许移动（所有权可以转移，如放入容器或从工厂函数返回）
- 析构、移动构造、移动赋值的**定义**必须放在 `.cpp` 中

</details>

---

## 下一步

下一节将介绍 KrKr2 如何定义自己的跨平台类型兼容层（`typedefine.h`），以及如何用 `typedef` / `using` 统一各平台的基础类型差异：

→ [KrKr2 类型兼容层与实践](./03-KrKr2类型兼容层与实践.md)

