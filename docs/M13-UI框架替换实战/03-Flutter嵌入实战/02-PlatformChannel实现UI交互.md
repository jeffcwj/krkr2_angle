# 02 - Platform Channel 实现 UI 交互

> **所属模块：** M13-UI框架替换实战  
> **前置知识：** [03-Flutter嵌入实战/01-FlutterEngine集成与CMake配置](01-FlutterEngine集成与CMake配置.md)  
> **预计阅读时间：** 60 分钟

---

## 本节目标

- 深入理解 Flutter Platform Channel 的三种类型及其适用场景
- 掌握在 C++ Embedder 层实现 MethodChannel 的完整机制
- 实现 Dart → C++ 单向调用（Dart 调用 KrKr2 功能）
- 实现 C++ → Dart 单向推送（KrKr2 向 Flutter UI 发送事件）
- 搭建完整的双向 Platform Channel 路由层

---

## 术语预览

| 术语 | 解释 |
|---|---|
| **Platform Channel** | Flutter 跨层通信机制，使 Dart 代码和宿主平台（C++/Java/Swift）之间能传递消息 |
| **MethodChannel** | 最常用的 Channel 类型，支持"调用→等待结果"的 RPC 模式 |
| **EventChannel** | 单向流式 Channel，宿主平台向 Dart 推送事件流（类似 Stream） |
| **BasicMessageChannel** | 最底层的 Channel，发送任意格式消息，无结果等待语义 |
| **MessageCodec** | Channel 消息的编解码器，负责将 Dart 对象序列化为字节数组 |
| **StandardMethodCodec** | Flutter 默认的 MethodChannel 编解码器，支持基本类型和集合 |
| **FlutterPlatformMessage** | Embedder API 中传递 Platform Channel 消息的 C 结构体 |
| **reply_handle** | 方法调用的回复句柄，必须在处理完方法后调用以返回结果给 Dart |

---

## 一、Platform Channel 体系结构

### 1.1 三种 Channel 对比

Flutter 提供三种 Platform Channel，各有不同的通信语义：

```
┌──────────────────────────────────────────────────────────────────┐
│  Dart 层（flutter_module/lib/）                                   │
│                                                                  │
│  MethodChannel          EventChannel         BasicMessageChannel │
│  发方法调用→等结果        订阅事件流            发送任意消息        │
└────────┬───────────────────┬─────────────────────┬──────────────┘
         │  Platform Channel │ 二进制消息           │
         │  （channel name   │ 传输                │
         │   + codec编解码）  │                     │
┌────────▼───────────────────▼─────────────────────▼──────────────┐
│  C++ Embedder 层（FlutterEngineWrapper + 宿主回调）               │
│                                                                  │
│  处理方法调用            推送事件字节流          收发消息           │
│  返回结果/错误           StreamHandler           MessageHandler  │
└──────────────────────────────────────────────────────────────────┘
```

**何时选哪种 Channel：**

| 场景 | 推荐 Channel |
|---|---|
| Dart 调用 C++ 函数并等待返回值（如打开文件对话框） | MethodChannel |
| C++ 向 Dart 实时推送数据流（如游戏帧率、内存使用） | EventChannel |
| C++ 向 Dart 发送一次性通知（如游戏加载完成） | BasicMessageChannel 或 MethodChannel |
| 双向高频通信（如鼠标坐标实时同步） | BasicMessageChannel（自定义二进制编解码） |

### 1.2 消息格式与编解码

Platform Channel 消息在底层是字节数组（`uint8_t*`）。`StandardMethodCodec` 定义了以下类型的编解码规则：

| Dart 类型 | C++ 等价类型 | 编码标志字节 |
|---|---|---|
| `null` | `nullptr` | 0x00 |
| `bool` | `bool` | 0x01/0x02 |
| `int`（< 32位）| `int32_t` | 0x03 |
| `int`（> 32位）| `int64_t` | 0x04 |
| `double` | `double` | 0x06 |
| `String` | `std::string`（UTF-8）| 0x07 |
| `Uint8List` | `std::vector<uint8_t>` | 0x08 |
| `List` | `std::vector<...>` | 0x0C |
| `Map` | `std::unordered_map<...>` | 0x0D |

KrKr2 中不需要从零实现编解码——使用 `flutter/standard_message_codec.h` 或自行封装简单的解码器即可。

---

## 二、MethodChannel：C++ 实现

### 2.1 整体调用流程

Dart 调用 MethodChannel 的完整流程：

```
[Dart]
  MethodChannel('krkr2/dialog')
    .invokeMethod('showConfigDialog', {'theme': 'dark'})
         │
         │ 序列化（StandardMethodCodec）
         ▼
[FlutterEngine]
  FlutterEngineSendPlatformMessage ← 从 Dart 发出
         │
         │ FlutterPlatformMessage 结构体
         ▼
[C++ platform_message_callback]
  channel = "krkr2/dialog"
  message = [编码的方法名 + 参数]
         │
         │ 解码，路由到对应 Handler
         ▼
[MethodCallHandler::handle("showConfigDialog", args)]
  → 打开 KrKr2 的 ConfigDialog
  → 用户操作完成
  → 序列化结果
         │
         │ FlutterEngineSendPlatformMessageResponse
         ▼
[Dart] Future 完成，返回结果
```

### 2.2 StandardMethodCodec 最小实现

为了在 C++ Embedder 层解析 Dart 的 MethodChannel 消息，需要实现（或引用）StandardMethodCodec 的解码逻辑：

```cpp
// cpp/core/environ/ui/flutter/StandardMethodCodec.h
#pragma once
#include <string>
#include <vector>
#include <map>
#include <variant>
#include <cstdint>
#include <stdexcept>

namespace krkr2 {
namespace ui {
namespace flutter_backend {

// Flutter 值类型（覆盖 StandardMessageCodec 支持的所有类型）
using FlutterNull = std::monostate;
using FlutterValue = std::variant<
    FlutterNull,                                      // null
    bool,                                             // bool
    int32_t,                                          // int（小）
    int64_t,                                          // int（大）
    double,                                           // double
    std::string,                                      // String
    std::vector<uint8_t>,                             // Uint8List
    std::vector<int32_t>,                             // Int32List
    std::vector<int64_t>,                             // Int64List
    std::vector<double>                               // Float64List
    // List 和 Map 需要递归定义，此处简化为 string→value map
>;

// 方法调用结构
struct MethodCall {
    std::string method;
    std::map<std::string, FlutterValue> args;  // 参数（简化：假设 Map<String, dynamic>）
};

// 方法结果：成功结果 或 错误
struct MethodResult {
    bool success = true;
    FlutterValue value;         // 成功时的返回值
    std::string error_code;     // 失败时的错误码
    std::string error_message;  // 失败时的错误信息
};

/**
 * StandardMethodCodec 的 C++ 解码器（只读）
 * 
 * 编码格式（envelope）：
 *   成功：[0x00] [value]
 *   失败：[0x01] [error_code: string] [error_message: string] [details: value]
 *
 * MethodCall 格式：
 *   [method_name: string] [args: value]
 */
class StandardMethodCodecDecoder {
public:
    /**
     * 解码 MethodCall（Dart 发来的调用）
     * @param data      消息字节数组
     * @param size      字节数组长度
     * @return          解码后的 MethodCall
     * @throws std::runtime_error 格式不合法时抛出
     */
    static MethodCall decodeMethodCall(const uint8_t* data, size_t size);

    /**
     * 编码 MethodResult（C++ 返回给 Dart 的结果）
     * @param result    方法结果
     * @return          编码后的字节数组
     */
    static std::vector<uint8_t> encodeResult(const MethodResult& result);

    /**
     * 编码一个简单的 String 返回值（便捷方法）
     */
    static std::vector<uint8_t> encodeStringResult(const std::string& value);

    /**
     * 编码一个错误返回
     */
    static std::vector<uint8_t> encodeError(
        const std::string& code,
        const std::string& message
    );

    /**
     * 编码 null 返回（方法调用成功但无返回值）
     */
    static std::vector<uint8_t> encodeNullResult();

private:
    // 内部读取辅助
    static std::string readString(const uint8_t* data, size_t size, size_t& offset);
    static FlutterValue readValue(const uint8_t* data, size_t size, size_t& offset);
    
    // 内部写入辅助
    static void writeString(std::vector<uint8_t>& buf, const std::string& s);
    static void writeValue(std::vector<uint8_t>& buf, const FlutterValue& value);
};

} // namespace flutter_backend
} // namespace ui
} // namespace krkr2
```

StandardMethodCodec 编码实现（核心部分）：

```cpp
// cpp/core/environ/ui/flutter/StandardMethodCodec.cpp
#include "StandardMethodCodec.h"
#include <cstring>
#include <stdexcept>

namespace krkr2 {
namespace ui {
namespace flutter_backend {

// Flutter StandardMessageCodec 类型标志字节
static constexpr uint8_t kNull        = 0;
static constexpr uint8_t kTrue        = 1;
static constexpr uint8_t kFalse       = 2;
static constexpr uint8_t kInt32       = 3;
static constexpr uint8_t kInt64       = 4;
static constexpr uint8_t kFloat64     = 6;
static constexpr uint8_t kString      = 7;
static constexpr uint8_t kUint8List   = 8;
static constexpr uint8_t kList        = 12;
static constexpr uint8_t kMap         = 13;

// 读取 size_t 类型的 VarInt（Flutter 使用 1~5 字节变长编码）
static size_t readSize(const uint8_t* data, size_t total, size_t& offset) {
    if (offset >= total) throw std::runtime_error("Unexpected EOF reading size");
    uint8_t b = data[offset++];
    if (b < 254) return b;
    if (b == 254) {
        if (offset + 2 > total) throw std::runtime_error("Unexpected EOF reading uint16 size");
        uint16_t v;
        memcpy(&v, data + offset, 2);
        offset += 2;
        return v;
    }
    // b == 255
    if (offset + 4 > total) throw std::runtime_error("Unexpected EOF reading uint32 size");
    uint32_t v;
    memcpy(&v, data + offset, 4);
    offset += 4;
    return v;
}

std::string StandardMethodCodecDecoder::readString(
    const uint8_t* data, size_t size, size_t& offset)
{
    if (offset >= size || data[offset] != kString) {
        throw std::runtime_error("Expected string type tag");
    }
    offset++;
    size_t len = readSize(data, size, offset);
    if (offset + len > size) throw std::runtime_error("String out of bounds");
    std::string s(reinterpret_cast<const char*>(data + offset), len);
    offset += len;
    return s;
}

MethodCall StandardMethodCodecDecoder::decodeMethodCall(
    const uint8_t* data, size_t size)
{
    size_t offset = 0;
    MethodCall call;
    call.method = readString(data, size, offset);
    
    // 参数：可能是 null 或 Map
    if (offset < size && data[offset] == kNull) {
        // 无参数
        return call;
    }
    if (offset < size && data[offset] == kMap) {
        offset++;  // 跳过 kMap 标志
        size_t count = readSize(data, size, offset);
        for (size_t i = 0; i < count; ++i) {
            std::string key = readString(data, size, offset);
            FlutterValue val = readValue(data, size, offset);
            call.args[key] = val;
        }
    }
    return call;
}

FlutterValue StandardMethodCodecDecoder::readValue(
    const uint8_t* data, size_t size, size_t& offset)
{
    if (offset >= size) throw std::runtime_error("Unexpected EOF reading value");
    uint8_t type = data[offset++];
    switch (type) {
        case kNull:    return FlutterNull{};
        case kTrue:    return true;
        case kFalse:   return false;
        case kInt32: {
            if (offset + 4 > size) throw std::runtime_error("EOF reading int32");
            int32_t v; memcpy(&v, data + offset, 4); offset += 4;
            return v;
        }
        case kInt64: {
            if (offset + 8 > size) throw std::runtime_error("EOF reading int64");
            int64_t v; memcpy(&v, data + offset, 8); offset += 8;
            return v;
        }
        case kFloat64: {
            // 8字节对齐
            size_t align = (8 - offset % 8) % 8;
            offset += align;
            if (offset + 8 > size) throw std::runtime_error("EOF reading double");
            double v; memcpy(&v, data + offset, 8); offset += 8;
            return v;
        }
        case kString: {
            // 回退一位重用 readString（它会检查 kString 标志）
            offset--;
            return readString(data, size, offset);
        }
        default:
            throw std::runtime_error(
                std::string("Unsupported type tag: ") + std::to_string(type)
            );
    }
}

// --- 编码（C++ → Dart）---

static void writeSize(std::vector<uint8_t>& buf, size_t size) {
    if (size < 254) {
        buf.push_back(static_cast<uint8_t>(size));
    } else if (size <= 0xFFFF) {
        buf.push_back(254);
        uint16_t v = static_cast<uint16_t>(size);
        buf.insert(buf.end(), reinterpret_cast<uint8_t*>(&v),
                   reinterpret_cast<uint8_t*>(&v) + 2);
    } else {
        buf.push_back(255);
        uint32_t v = static_cast<uint32_t>(size);
        buf.insert(buf.end(), reinterpret_cast<uint8_t*>(&v),
                   reinterpret_cast<uint8_t*>(&v) + 4);
    }
}

void StandardMethodCodecDecoder::writeString(
    std::vector<uint8_t>& buf, const std::string& s)
{
    buf.push_back(kString);
    writeSize(buf, s.size());
    buf.insert(buf.end(), s.begin(), s.end());
}

std::vector<uint8_t> StandardMethodCodecDecoder::encodeNullResult() {
    return {0x00, kNull};  // 成功 envelope + null 值
}

std::vector<uint8_t> StandardMethodCodecDecoder::encodeStringResult(
    const std::string& value)
{
    std::vector<uint8_t> buf = {0x00};  // 成功 envelope
    writeString(buf, value);
    return buf;
}

std::vector<uint8_t> StandardMethodCodecDecoder::encodeError(
    const std::string& code, const std::string& message)
{
    std::vector<uint8_t> buf = {0x01};  // 错误 envelope
    writeString(buf, code);
    writeString(buf, message);
    buf.push_back(kNull);  // details = null
    return buf;
}

} // namespace flutter_backend
} // namespace ui
} // namespace krkr2
```

---

## 三、Platform Channel 路由器

### 3.1 ChannelRouter 设计

在 C++ 端，需要一个路由器把不同 channel 名称的消息分发到不同的 Handler：

```cpp
// cpp/core/environ/ui/flutter/ChannelRouter.h
#pragma once
#include "StandardMethodCodec.h"
#include <flutter/flutter_embedder.h>
#include <functional>
#include <string>
#include <unordered_map>
#include <memory>

namespace krkr2 {
namespace ui {
namespace flutter_backend {

/**
 * MethodCallHandler — 处理特定 Channel 上的方法调用
 * 每个 Handler 管理一组方法名到处理函数的映射
 */
class MethodCallHandler {
public:
    using HandlerFn = std::function<
        MethodResult(const std::string& method, 
                     const std::map<std::string, FlutterValue>& args)
    >;

    void addMethod(const std::string& method_name, HandlerFn handler) {
        handlers_[method_name] = std::move(handler);
    }

    MethodResult handle(const MethodCall& call) {
        auto it = handlers_.find(call.method);
        if (it == handlers_.end()) {
            return MethodResult{
                false, FlutterNull{},
                "NOT_IMPLEMENTED",
                "Method '" + call.method + "' not implemented"
            };
        }
        try {
            return it->second(call.method, call.args);
        } catch (const std::exception& e) {
            return MethodResult{
                false, FlutterNull{},
                "INTERNAL_ERROR",
                e.what()
            };
        }
    }

private:
    std::unordered_map<std::string, HandlerFn> handlers_;
};

/**
 * ChannelRouter — Platform Channel 消息的中央路由器
 *
 * 用法：
 *   ChannelRouter router(engine_wrapper);
 *   auto handler = router.registerMethodChannel("krkr2/dialog");
 *   handler->addMethod("showConfigDialog", [](auto& method, auto& args) {
 *       // 处理逻辑
 *       return MethodResult{true};
 *   });
 */
class ChannelRouter {
public:
    explicit ChannelRouter(FlutterEngineWrapper* engine);

    /**
     * 注册一个 MethodChannel Handler
     * @return  Handler 对象，调用者在上面添加方法处理函数
     */
    MethodCallHandler* registerMethodChannel(const std::string& channel_name);

    /**
     * 收到来自 FlutterEngine 的 Platform Channel 消息时调用
     * （由 FlutterEngineWrapper 的 platform_message_callback 触发）
     */
    void handleMessage(const std::string& channel,
                       const uint8_t* data, size_t size,
                       const FlutterPlatformMessageResponseHandle* response_handle);

    /**
     * 向 Dart 主动推送消息（EventChannel 使用）
     * channel_name 须与 Dart 端 EventChannel 名称一致
     */
    void sendEvent(const std::string& channel_name,
                   const std::vector<uint8_t>& encoded_data);

private:
    FlutterEngineWrapper* engine_;
    std::unordered_map<std::string, std::unique_ptr<MethodCallHandler>> method_channels_;
};

} // namespace flutter_backend
} // namespace ui
} // namespace krkr2
```

```cpp
// cpp/core/environ/ui/flutter/ChannelRouter.cpp
#include "ChannelRouter.h"
#include "FlutterEngineWrapper.h"

namespace krkr2 {
namespace ui {
namespace flutter_backend {

ChannelRouter::ChannelRouter(FlutterEngineWrapper* engine)
    : engine_(engine) {}

MethodCallHandler* ChannelRouter::registerMethodChannel(
    const std::string& channel_name)
{
    auto handler = std::make_unique<MethodCallHandler>();
    auto* raw = handler.get();
    method_channels_[channel_name] = std::move(handler);
    return raw;
}

void ChannelRouter::handleMessage(
    const std::string& channel,
    const uint8_t* data, size_t size,
    const FlutterPlatformMessageResponseHandle* response_handle)
{
    auto it = method_channels_.find(channel);
    if (it == method_channels_.end()) {
        // 未注册的 channel：发送空响应（避免 Dart 侧超时）
        if (response_handle) {
            FlutterEngineSendPlatformMessageResponse(
                engine_->getHandle(),
                response_handle,
                nullptr, 0
            );
        }
        return;
    }

    MethodCall call;
    try {
        call = StandardMethodCodecDecoder::decodeMethodCall(data, size);
    } catch (const std::exception& e) {
        if (response_handle) {
            auto err = StandardMethodCodecDecoder::encodeError(
                "DECODE_ERROR", e.what()
            );
            FlutterEngineSendPlatformMessageResponse(
                engine_->getHandle(),
                response_handle,
                err.data(), err.size()
            );
        }
        return;
    }

    MethodResult result = it->second->handle(call);

    if (response_handle) {
        std::vector<uint8_t> encoded;
        if (result.success) {
            // 根据结果类型选择编码方式
            if (std::holds_alternative<FlutterNull>(result.value)) {
                encoded = StandardMethodCodecDecoder::encodeNullResult();
            } else if (std::holds_alternative<std::string>(result.value)) {
                encoded = StandardMethodCodecDecoder::encodeStringResult(
                    std::get<std::string>(result.value)
                );
            } else {
                encoded = StandardMethodCodecDecoder::encodeNullResult();
            }
        } else {
            encoded = StandardMethodCodecDecoder::encodeError(
                result.error_code, result.error_message
            );
        }
        
        FlutterEngineSendPlatformMessageResponse(
            engine_->getHandle(),
            response_handle,
            encoded.data(), encoded.size()
        );
    }
}

void ChannelRouter::sendEvent(
    const std::string& channel_name,
    const std::vector<uint8_t>& encoded_data)
{
    FlutterPlatformMessage message = {};
    message.struct_size = sizeof(FlutterPlatformMessage);
    message.channel = channel_name.c_str();
    message.message = encoded_data.data();
    message.message_size = encoded_data.size();
    message.response_handle = nullptr;  // 事件推送不需要回复句柄
    
    FlutterEngineSendPlatformMessage(engine_->getHandle(), &message);
}

} // namespace flutter_backend
} // namespace ui
} // namespace krkr2
```

---

## 四、KrKr2 Channel 注册

### 4.1 所有 Channel 的统一注册入口

```cpp
// cpp/core/environ/ui/flutter/KrKr2Channels.h
#pragma once
#include "ChannelRouter.h"
#include "FlutterEngineWrapper.h"
#include <string>

namespace krkr2 {
namespace ui {
namespace flutter_backend {

// Channel 名称常量（Dart 端和 C++ 端必须一致）
static constexpr char kDialogChannel[]   = "krkr2/dialog";
static constexpr char kSystemChannel[]   = "krkr2/system";
static constexpr char kGameChannel[]     = "krkr2/game";
static constexpr char kEventsChannel[]   = "krkr2/events";  // EventChannel

/**
 * 注册所有 KrKr2 Platform Channel 方法处理器
 * 在 FlutterEngine 初始化完成后调用一次
 */
void registerKrKr2Channels(ChannelRouter* router);

} // namespace flutter_backend
} // namespace ui
} // namespace krkr2
```

```cpp
// cpp/core/environ/ui/flutter/KrKr2Channels.cpp
#include "KrKr2Channels.h"
// 引入 KrKr2 核心功能头文件
// #include "core/environ/dialog/ConfigDialog.h"
// #include "core/system/AppConfig.h"

namespace krkr2 {
namespace ui {
namespace flutter_backend {

void registerKrKr2Channels(ChannelRouter* router) {
    
    // ═══════════════════════════════════════════════
    // krkr2/dialog — 对话框控制
    // ═══════════════════════════════════════════════
    auto* dialog_handler = router->registerMethodChannel(kDialogChannel);
    
    // showConfigDialog：打开设置对话框
    // Dart 调用：channel.invokeMethod('showConfigDialog', {'tab': 'video'})
    // 返回：null（对话框为模态，关闭后 Future 完成）
    dialog_handler->addMethod("showConfigDialog",
        [](const std::string&, const std::map<std::string, FlutterValue>& args)
        -> MethodResult
        {
            std::string initial_tab = "general";
            auto it = args.find("tab");
            if (it != args.end() && 
                std::holds_alternative<std::string>(it->second)) {
                initial_tab = std::get<std::string>(it->second);
            }
            
            // 实际调用 KrKr2 的对话框（此处为示例）
            // ConfigDialog dialog;
            // dialog.setInitialTab(initial_tab);
            // dialog.showModal();
            
            return MethodResult{true};  // null 结果
        }
    );
    
    // showGameSelect：打开游戏选择对话框
    // 返回：选中的游戏路径字符串，或 null（用户取消）
    dialog_handler->addMethod("showGameSelect",
        [](const std::string&, const std::map<std::string, FlutterValue>&)
        -> MethodResult
        {
            // GameSelectDialog dialog;
            // std::string selected = dialog.showModal();
            // if (selected.empty()) return MethodResult{true};  // 用户取消
            
            // 模拟返回：
            std::string selected_path = "/games/my_game/";
            return MethodResult{true, selected_path};
        }
    );
    
    // ═══════════════════════════════════════════════
    // krkr2/system — 系统信息查询
    // ═══════════════════════════════════════════════
    auto* system_handler = router->registerMethodChannel(kSystemChannel);
    
    // getVersion：获取 KrKr2 版本号
    system_handler->addMethod("getVersion",
        [](const std::string&, const std::map<std::string, FlutterValue>&)
        -> MethodResult
        {
            return MethodResult{true, std::string("2.0.0-alpha")};
        }
    );
    
    // getPlatform：获取当前运行平台
    system_handler->addMethod("getPlatform",
        [](const std::string&, const std::map<std::string, FlutterValue>&)
        -> MethodResult
        {
#if defined(_WIN32)
            return MethodResult{true, std::string("windows")};
#elif defined(__ANDROID__)
            return MethodResult{true, std::string("android")};
#elif defined(__APPLE__)
            return MethodResult{true, std::string("macos")};
#else
            return MethodResult{true, std::string("linux")};
#endif
        }
    );
    
    // ═══════════════════════════════════════════════
    // krkr2/game — 游戏控制
    // ═══════════════════════════════════════════════
    auto* game_handler = router->registerMethodChannel(kGameChannel);
    
    // pause：暂停游戏
    game_handler->addMethod("pause",
        [](const std::string&, const std::map<std::string, FlutterValue>&)
        -> MethodResult
        {
            // TVPMainWindow->pause();
            return MethodResult{true};
        }
    );
    
    // resume：恢复游戏
    game_handler->addMethod("resume",
        [](const std::string&, const std::map<std::string, FlutterValue>&)
        -> MethodResult
        {
            // TVPMainWindow->resume();
            return MethodResult{true};
        }
    );
    
    // saveScreenshot：截图并返回 base64 编码图片
    game_handler->addMethod("saveScreenshot",
        [](const std::string&, const std::map<std::string, FlutterValue>&)
        -> MethodResult
        {
            // TODO: 实现截图逻辑
            return MethodResult{
                false, FlutterNull{},
                "NOT_IMPLEMENTED", "Screenshot not yet implemented"
            };
        }
    );
}

} // namespace flutter_backend
} // namespace ui
} // namespace krkr2
```

---

## 五、Dart 端 Channel 封装

### 5.1 flutter_module/lib/channels/krkr2_channels.dart

在 Flutter 模块中，封装所有与 KrKr2 通信的 Channel：

```dart
// flutter_module/lib/channels/krkr2_channels.dart
import 'package:flutter/services.dart';

/// KrKr2 Platform Channel 封装
/// 提供类型安全的 Dart API，隐藏底层 Channel 细节
class KrKr2Channels {
  // 单例
  static final KrKr2Channels instance = KrKr2Channels._();
  KrKr2Channels._();

  // Channel 实例
  static const _dialogChannel = MethodChannel('krkr2/dialog');
  static const _systemChannel = MethodChannel('krkr2/system');
  static const _gameChannel   = MethodChannel('krkr2/game');
  static const _eventsChannel = EventChannel('krkr2/events');

  // ─────────── Dialog ───────────

  /// 打开设置对话框
  /// [tab] 初始显示的标签页，如 'video', 'audio', 'general'
  Future<void> showConfigDialog({String tab = 'general'}) async {
    await _dialogChannel.invokeMethod<void>(
      'showConfigDialog',
      {'tab': tab},
    );
  }

  /// 打开游戏选择对话框
  /// 返回选中的游戏路径，如果用户取消返回 null
  Future<String?> showGameSelect() async {
    return _dialogChannel.invokeMethod<String>('showGameSelect');
  }

  // ─────────── System ───────────

  /// 获取 KrKr2 版本号
  Future<String> getVersion() async {
    return await _systemChannel.invokeMethod<String>('getVersion')
        ?? 'unknown';
  }

  /// 获取当前运行平台（'windows' | 'android' | 'macos' | 'linux'）
  Future<String> getPlatform() async {
    return await _systemChannel.invokeMethod<String>('getPlatform')
        ?? 'unknown';
  }

  // ─────────── Game ───────────

  /// 暂停游戏
  Future<void> pause() => _gameChannel.invokeMethod('pause');

  /// 恢复游戏
  Future<void> resume() => _gameChannel.invokeMethod('resume');

  // ─────────── EventChannel ───────────

  /// 订阅 KrKr2 事件流
  /// 事件格式：Map<String, dynamic>，含 'type' 字段区分事件类型
  Stream<Map<String, dynamic>> get events {
    return _eventsChannel
        .receiveBroadcastStream()
        .map((event) => Map<String, dynamic>.from(event as Map));
  }
}
```

### 5.2 使用示例

```dart
// flutter_module/lib/main.dart
import 'package:flutter/material.dart';
import 'channels/krkr2_channels.dart';

void main() {
  runApp(const KrKr2App());
}

class KrKr2App extends StatelessWidget {
  const KrKr2App({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'KrKr2 UI',
      theme: ThemeData.dark(),
      home: const MainMenuScreen(),
    );
  }
}

class MainMenuScreen extends StatefulWidget {
  const MainMenuScreen({super.key});

  @override
  State<MainMenuScreen> createState() => _MainMenuScreenState();
}

class _MainMenuScreenState extends State<MainMenuScreen> {
  String _platform = 'Loading...';
  String _version  = 'Loading...';

  @override
  void initState() {
    super.initState();
    _loadSystemInfo();
    
    // 订阅来自 C++ 的事件流
    KrKr2Channels.instance.events.listen((event) {
      final type = event['type'] as String?;
      if (type == 'game_loaded') {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('游戏已加载：${event['title']}')),
        );
      }
    });
  }

  Future<void> _loadSystemInfo() async {
    final platform = await KrKr2Channels.instance.getPlatform();
    final version  = await KrKr2Channels.instance.getVersion();
    setState(() {
      _platform = platform;
      _version  = version;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('KrKr2 v$_version ($_platform)')),
      body: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          ElevatedButton(
            onPressed: () async {
              final path = await KrKr2Channels.instance.showGameSelect();
              if (path != null) {
                debugPrint('Selected game: $path');
              }
            },
            child: const Text('选择游戏'),
          ),
          const SizedBox(height: 16),
          ElevatedButton(
            onPressed: () => KrKr2Channels.instance.showConfigDialog(tab: 'video'),
            child: const Text('设置'),
          ),
        ],
      ),
    );
  }
}
```

---

## 六、C++ → Dart 事件推送

### 6.1 EventChannel 在 C++ 端的实现

EventChannel 不像 MethodChannel 那样有明确的请求-响应对，它由 C++ 主动向 Dart 推送事件。在 Embedder 层，推送通过 `ChannelRouter::sendEvent()` 实现。

但 EventChannel 在 Dart 端需要先调用 `receiveBroadcastStream()` 订阅，这会发送一个名为 `listen` 的 MethodChannel 调用通知 C++ 端开始推送。

```cpp
// 注册 EventChannel 的 listen/cancel 控制
// flutter_module 的 EventChannel 在开始订阅时会发送
//   channel: "krkr2/events"
//   method:  "listen"
// 在取消订阅时发送
//   method:  "cancel"

auto* events_control = router->registerMethodChannel(kEventsChannel);

events_control->addMethod("listen",
    [](const std::string&, const std::map<std::string, FlutterValue>&)
    -> MethodResult
    {
        // Dart 开始订阅事件流
        // 此时可以启动数据采集（如性能监控定时器）
        return MethodResult{true};
    }
);

events_control->addMethod("cancel",
    [](const std::string&, const std::map<std::string, FlutterValue>&)
    -> MethodResult
    {
        // Dart 取消订阅，停止推送
        return MethodResult{true};
    }
);
```

### 6.2 推送事件到 Dart

```cpp
// FlutterUIManager 中，当 KrKr2 触发某事件时，向 Dart 推送通知

void FlutterUIManager::notifyGameLoaded(const std::string& game_title) {
    // 构造事件 Map：{'type': 'game_loaded', 'title': '<游戏名>'}
    // 用 StandardMethodCodec 编码一个 Map
    
    // 简化方案：手动构造 StandardMessageCodec 格式的 Map
    std::vector<uint8_t> buf;
    buf.push_back(0x0D);  // kMap
    
    // 写入 size=2（两个键值对）
    buf.push_back(2);
    
    // key: "type"
    buf.push_back(0x07);  // kString
    buf.push_back(4);     // "type".length
    buf.insert(buf.end(), {'t','y','p','e'});
    
    // value: "game_loaded"
    buf.push_back(0x07);  // kString
    buf.push_back(11);    // "game_loaded".length
    const char* ev = "game_loaded";
    buf.insert(buf.end(), ev, ev + 11);
    
    // key: "title"
    buf.push_back(0x07);
    buf.push_back(5);
    buf.insert(buf.end(), {'t','i','t','l','e'});
    
    // value: game_title
    buf.push_back(0x07);
    buf.push_back(static_cast<uint8_t>(game_title.size()));
    buf.insert(buf.end(), game_title.begin(), game_title.end());
    
    channel_router_->sendEvent(kEventsChannel, buf);
}
```

---

## 七、动手实践

### 实践 7.1：测试 MethodChannel 调用链

验证整个 Channel 路径（Dart → C++ → Dart）是否正常工作：

```dart
// 在 flutter_module/test/channel_test.dart 中添加测试
import 'package:flutter/services.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();

  const channel = MethodChannel('krkr2/system');

  setUp(() {
    // Mock C++ 端的处理器（单元测试中无真实 C++ 后端）
    TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
        .setMockMethodCallHandler(channel, (MethodCall call) async {
      if (call.method == 'getVersion') return '2.0.0-test';
      if (call.method == 'getPlatform') return 'test';
      return null;
    });
  });

  test('getVersion returns correct version', () async {
    final result = await channel.invokeMethod<String>('getVersion');
    expect(result, equals('2.0.0-test'));
  });

  test('getPlatform returns platform string', () async {
    final result = await channel.invokeMethod<String>('getPlatform');
    expect(result, isNotNull);
    expect(['windows', 'android', 'macos', 'linux', 'test'],
           contains(result));
  });
}
```

---

## 对照项目源码

```
krkr2/
└── cpp/core/environ/ui/flutter/
    ├── StandardMethodCodec.h      ← 编解码器头文件
    ├── StandardMethodCodec.cpp    ← 编解码器实现
    ├── ChannelRouter.h            ← Channel 路由器头文件
    ├── ChannelRouter.cpp          ← Channel 路由器实现
    ├── KrKr2Channels.h            ← KrKr2 Channel 注册声明
    └── KrKr2Channels.cpp          ← KrKr2 Channel 注册实现

krkr2/flutter_module/lib/
└── channels/
    └── krkr2_channels.dart        ← Dart 端 Channel 封装
```

---

## 本节小结

- Platform Channel 有三种类型：MethodChannel（RPC）、EventChannel（事件流）、BasicMessageChannel（任意消息）
- Embedder 层通过 `platform_message_callback` 接收 Dart 消息，通过 `FlutterEngineSendPlatformMessageResponse` 回复结果
- `StandardMethodCodec` 使用类型标志字节 + 变长编码，需要在 C++ 端手动实现解码逻辑
- `ChannelRouter` 负责将消息按 channel 名称路由到对应 Handler，实现关注点分离
- C++ → Dart 的事件推送使用 `FlutterEngineSendPlatformMessage`（无 response_handle），EventChannel 的 Dart 端通过 `receiveBroadcastStream()` 订阅

---

## 练习题与答案

**题 1：** MethodChannel 调用返回 `MissingPluginException` 是什么原因？在 Embedder 层如何排查？

<details>
<summary>查看答案</summary>

`MissingPluginException` 通常意味着 Dart 发出的方法调用在 C++ 端没有收到任何响应（response_handle 从未被回复）。

**排查步骤：**
1. 检查 `platform_message_callback` 是否正确注册到 `FlutterProjectArgs`
2. 检查 ChannelRouter 是否注册了对应的 channel 名称（注意大小写和拼写）
3. 检查 `FlutterEngineSendPlatformMessageResponse` 是否在每个代码路径上都被调用（包括错误路径）
4. 如果 response_handle 为 nullptr，说明是单向消息（C++ 发给 Dart 的），不需要回复

常见错误：在某个分支中忘记调用 `FlutterEngineSendPlatformMessageResponse`，导致 Dart 的 Future 永远 pending，最终超时抛出异常。

</details>

---

**题 2：** 为什么 `FlutterEngineSendPlatformMessage`（C++ 向 Dart 推送）不需要 response_handle？EventChannel 的推送与 MethodChannel 的调用在通信语义上有何本质区别？

<details>
<summary>查看答案</summary>

**原因：** `response_handle` 是"回复令牌"，仅在 Dart 发起调用（期待 C++ 回复）时由 Flutter Engine 分配。当 C++ 主动向 Dart 推送时，不存在"等待回复"的语义——推送是 fire-and-forget 的。

**本质区别：**
- **MethodChannel**：Dart → C++，类似 RPC，Dart 的 `invokeMethod` 返回 `Future<T>`，调用者等待 C++ 处理完成并回复结果。是双向的一次交互。
- **EventChannel**：C++ → Dart，类似 Server-Sent Events，C++ 可以在任意时刻推送，Dart 通过 `Stream` 监听。是单向的持续流。

在实现层面，两者都使用相同的 `FlutterEngineSendPlatformMessage` / `FlutterEngineSendPlatformMessageResponse` API，区别仅在于：
- MethodChannel 调用：Dart 发消息时带有 response_handle，C++ 必须调用 Response 函数回复
- EventChannel 推送：C++ 直接发消息，response_handle 为 nullptr，Dart 不会等待回复

</details>

---

**题 3：** 如果 Dart 调用 `showConfigDialog` 时传入了 int 类型的 `tab` 参数（而不是 String），C++ 端的处理器应如何健壮地处理这种类型不匹配？

<details>
<summary>查看答案</summary>

```cpp
dialog_handler->addMethod("showConfigDialog",
    [](const std::string&, const std::map<std::string, FlutterValue>& args)
    -> MethodResult
    {
        std::string initial_tab = "general";
        auto it = args.find("tab");
        if (it != args.end()) {
            // 用 std::visit 处理所有可能的类型
            std::visit([&initial_tab](auto&& v) {
                using T = std::decay_t<decltype(v)>;
                if constexpr (std::is_same_v<T, std::string>) {
                    initial_tab = v;
                } else if constexpr (std::is_same_v<T, int32_t>) {
                    // 将 int 映射到 tab 名称（或直接返回错误）
                    static const char* tabs[] = {"general", "video", "audio"};
                    if (v >= 0 && v < 3) initial_tab = tabs[v];
                }
                // 其他类型：保持默认值 "general"
            }, it->second);
        }
        
        // 继续处理...
        return MethodResult{true};
    }
);
```

健壮处理的关键：
1. 使用 `std::visit` 或 `std::holds_alternative` 进行类型检查，而不是直接 `std::get`（类型不匹配时会抛出 `std::bad_variant_access`）
2. 为意外类型提供合理的默认行为，而不是直接返回错误（向后兼容）
3. 记录警告日志，便于调试

</details>

---

## 下一步

本节完成了 Platform Channel 的完整实现，建立了 Dart ↔ C++ 双向通信机制。下一章 [04-Compose嵌入实战](../04-Compose嵌入实战/01-KotlinNative与cinterop.md) 将探索 Kotlin/Native + Compose Multiplatform 作为替代 UI 框架的集成方案。
