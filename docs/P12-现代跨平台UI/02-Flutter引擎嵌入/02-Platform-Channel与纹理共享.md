# Platform Channel 与纹理共享

> **所属模块：** P12-现代跨平台 UI（Flutter / Compose）
> **前置知识：** [02-Flutter引擎嵌入/01-FlutterEngine-API与嵌入原理](01-FlutterEngine-API与嵌入原理.md)
> **预计阅读时间：** 40 分钟（按每分钟 200 字估算）

## 本节目标

读完本节后，你将能够：
1. 理解 Flutter Platform Channel 的三种类型及其通信机制
2. 使用 MethodChannel 在 Dart 和 C++ 之间进行方法调用
3. 使用 EventChannel 实现 C++ 向 Dart 的事件流推送
4. 理解 Flutter 的 Texture Widget 纹理共享机制
5. 实现 C++ 渲染引擎输出画面到 Flutter Texture Widget 的完整流程
6. 设计 KrKr2 游戏画面在 Flutter UI 中的显示方案

## 术语预览

本节将涉及以下术语，先有个印象，正文中会逐一详细讲解：

| 术语 | 英文 | 一句话解释 |
|------|------|-----------|
| Platform Channel | Platform Channel | Flutter 与宿主平台（C++/Java/ObjC）之间的双向通信管道 |
| MethodChannel | MethodChannel | 基于请求-响应模式的 Platform Channel，类似 RPC（远程过程调用）函数调用 |
| EventChannel | EventChannel | 基于事件流的 Platform Channel，宿主向 Dart 持续推送数据 |
| BasicMessageChannel | BasicMessageChannel | 最底层的消息通道，发送和接收任意格式的二进制数据 |
| 消息编解码器 | Message Codec | 将 Dart 对象序列化为二进制数据（或反向解码）的组件，如 StandardMessageCodec（标准编解码器）、JSONMessageCodec（JSON 编解码器） |
| Texture Widget | Texture Widget | Flutter 中用于显示外部纹理（由宿主 C++ 代码生成的 GPU 纹理）的控件 |
| 外部纹理 | External Texture | 由 Flutter Engine 之外的代码（如游戏渲染引擎）创建的 GPU 纹理，通过注册机制让 Engine 能引用它 |
| 纹理注册表 | Texture Registry | Flutter Engine 维护的纹理映射表，将整数 ID 映射到宿主提供的纹理回调 |
| 像素缓冲区 | Pixel Buffer | 一块连续的 CPU 内存，存储像素数据（如 RGBA 格式），可作为纹理数据源上传到 GPU |

## Platform Channel 通信机制

### 为什么需要 Platform Channel

Flutter 的 UI 逻辑用 Dart 编写，运行在 Dart VM 中；而 KrKr2 的核心逻辑（TJS2 脚本引擎、图层渲染、音视频播放）用 C++ 编写。两者运行在不同的执行环境中，需要一个通信桥梁：

```
┌─────────────────────┐         ┌─────────────────────┐
│     Flutter UI      │         │     KrKr2 核心      │
│     (Dart VM)       │◄═══════►│     (C++ Native)    │
│                     │ Platform│                     │
│  MaterialApp        │ Channel │  TJS2 VM            │
│  游戏菜单           │         │  图层渲染            │
│  存档管理           │         │  音视频播放          │
│  设置界面           │         │  KAG 脚本解析        │
└─────────────────────┘         └─────────────────────┘
```

Platform Channel 就是这个桥梁。它允许：
- Dart 调用 C++ 函数（如"获取游戏存档列表"、"跳转到指定剧本位置"）
- C++ 向 Dart 推送事件（如"游戏画面更新了"、"TJS2 脚本报错了"）
- 双向传输结构化数据（字符串、整数、列表、字典等）

### Platform Channel 的底层实现

所有 Platform Channel 最终都通过 Embedder API 的 `FlutterPlatformMessage` 进行传输：

```c
// Embedder API 中的平台消息结构
typedef struct {
    size_t struct_size;
    const char* channel;          // 通道名称（UTF-8 字符串）
    const uint8_t* message;       // 消息内容（二进制数据）
    size_t message_size;          // 消息长度（字节）
    const FlutterPlatformMessageResponseHandle*
        response_handle;          // 响应句柄（用于回复请求）
} FlutterPlatformMessage;

// 宿主 → Dart：发送平台消息
FlutterEngineResult FlutterEngineSendPlatformMessage(
    FLUTTER_API_SYMBOL(FlutterEngine) engine,
    const FlutterPlatformMessage* message
);

// 宿主回复 Dart 的请求
FlutterEngineResult FlutterEngineSendPlatformMessageResponse(
    FLUTTER_API_SYMBOL(FlutterEngine) engine,
    const FlutterPlatformMessageResponseHandle* handle,
    const uint8_t* data,
    size_t data_length
);
```

消息流向：

```
Dart 侧                    C++ 宿主侧
────────                    ──────────
MethodChannel.invokeMethod("getScore")
      │
      ▼ (序列化为二进制)
  StandardMethodCodec.encodeMethodCall()
      │
      ▼ (Engine 内部传输)
  platform_message_callback(message)  ← FlutterProjectArgs 中注册的回调
      │
      ▼ (宿主解析通道名和消息内容)
  处理请求，调用 C++ 逻辑
      │
      ▼ (构造响应)
  FlutterEngineSendPlatformMessageResponse(handle, response_data)
      │
      ▼ (Engine 内部传输回 Dart)
  Dart Future<int> completes with 42
```

### 三种 Channel 类型对比

| 特性 | MethodChannel | EventChannel | BasicMessageChannel |
|------|--------------|-------------|-------------------|
| 通信模式 | 请求-响应（双向） | 事件流（C++ → Dart） | 消息发送（双向） |
| 典型场景 | 调用 C++ 函数获取结果 | C++ 持续推送状态更新 | 底层自定义协议 |
| Dart 返回类型 | `Future<T>` | `Stream<T>` | `Future<T>` |
| 默认编解码器 | StandardMethodCodec | StandardMethodCodec | StandardMessageCodec |
| KrKr2 用途 | 获取存档列表、设置游戏参数 | 游戏状态变化通知、日志推送 | 大块二进制数据传输 |

## MethodChannel 详解

### 工作原理

MethodChannel 实现了类似 RPC（Remote Procedure Call，远程过程调用——一种让程序调用另一个进程/线程中函数的技术）的通信模式。Dart 侧调用一个"方法名"并传入参数，C++ 侧收到请求后处理并返回结果：

```
Dart:   result = await channel.invokeMethod('方法名', 参数);
                        │
                        ▼
C++:    收到 {method: "方法名", args: 参数}
        处理逻辑...
        return 结果;
                        │
                        ▼
Dart:   result == 结果   ✓
```

### 消息编解码器

MethodChannel 使用 **StandardMethodCodec**（标准方法编解码器）来序列化方法调用和响应。底层的 **StandardMessageCodec** 支持以下 Dart ↔ C++ 类型映射：

| Dart 类型 | C++ 对应 | 编码标记字节 | 说明 |
|-----------|---------|-------------|------|
| `null` | `nullptr` / 空标志 | 0x00 | 空值 |
| `bool` | `bool` | 0x01 (false) / 0x02 (true) | 布尔值 |
| `int` (≤32bit) | `int32_t` | 0x03 | 32 位有符号整数 |
| `int` (≤64bit) | `int64_t` | 0x04 | 64 位有符号整数 |
| `double` | `double` | 0x06 | 64 位浮点数 |
| `String` | `std::string` (UTF-8) | 0x07 | UTF-8 字符串，前缀长度 |
| `Uint8List` | `std::vector<uint8_t>` | 0x08 | 字节数组（高效，无拷贝） |
| `Int32List` | `std::vector<int32_t>` | 0x09 | 32 位整数数组 |
| `Int64List` | `std::vector<int64_t>` | 0x0A | 64 位整数数组 |
| `Float32List` | `std::vector<float>` | 0x0B | 单精度浮点数组 |
| `Float64List` | `std::vector<double>` | 0x0C | 双精度浮点数组 |
| `List` | 递归编码 | 0x0D | 异构列表 |
| `Map` | 递归编码 | 0x0E | 键值对映射 |

### 示例 1：C++ MethodChannel 处理器（完整可编译）

以下示例展示如何在 C++ 宿主中处理来自 Dart 的 MethodChannel 调用。我们实现一个简单的"游戏引擎控制"通道：

```cpp
// method_channel_handler.cpp
// 完整的 C++ MethodChannel 处理器示例
#include <flutter_embedder.h>
#include <cstdint>
#include <cstring>
#include <functional>
#include <iostream>
#include <map>
#include <string>
#include <vector>

// ===== StandardMethodCodec 简易解码器 =====
// 实际项目中推荐使用 flutter/cpp_client_wrapper 库

// 从二进制数据中读取 UTF-8 字符串
static std::string ReadString(const uint8_t*& ptr) {
    // 读取字符串长度（变长编码）
    uint32_t length = 0;
    uint8_t byte = *ptr++;
    if (byte < 254) {
        length = byte;
    } else if (byte == 254) {
        // 2 字节长度
        length = ptr[0] | (ptr[1] << 8);
        ptr += 2;
    } else {
        // 4 字节长度
        length = ptr[0] | (ptr[1] << 8)
               | (ptr[2] << 16) | (ptr[3] << 24);
        ptr += 4;
    }
    std::string result(reinterpret_cast<const char*>(ptr), length);
    ptr += length;
    return result;
}

// 从二进制数据中读取一个值（简化版，只处理常用类型）
struct DartValue {
    enum Type { kNull, kBool, kInt, kDouble, kString };
    Type type = kNull;
    bool bool_val = false;
    int64_t int_val = 0;
    double double_val = 0.0;
    std::string string_val;
};

static DartValue ReadValue(const uint8_t*& ptr) {
    DartValue val;
    uint8_t tag = *ptr++;
    switch (tag) {
        case 0x00: // null
            val.type = DartValue::kNull;
            break;
        case 0x01: // false
            val.type = DartValue::kBool;
            val.bool_val = false;
            break;
        case 0x02: // true
            val.type = DartValue::kBool;
            val.bool_val = true;
            break;
        case 0x03: { // int32
            int32_t v;
            std::memcpy(&v, ptr, 4);
            ptr += 4;
            val.type = DartValue::kInt;
            val.int_val = v;
            break;
        }
        case 0x04: { // int64
            int64_t v;
            std::memcpy(&v, ptr, 8);
            ptr += 8;
            val.type = DartValue::kInt;
            val.int_val = v;
            break;
        }
        case 0x06: { // double（对齐到 8 字节边界）
            // 跳过填充字节至 8 字节对齐
            double v;
            std::memcpy(&v, ptr, 8);
            ptr += 8;
            val.type = DartValue::kDouble;
            val.double_val = v;
            break;
        }
        case 0x07: // string
            val.type = DartValue::kString;
            val.string_val = ReadString(ptr);
            break;
        default:
            std::cerr << "未处理的类型标记: 0x"
                      << std::hex << (int)tag << std::endl;
            break;
    }
    return val;
}

// ===== 编码响应 =====

// 编码成功响应（StandardMethodCodec 格式）
static std::vector<uint8_t> EncodeSuccessResponse(
    const std::string& result
) {
    std::vector<uint8_t> data;
    data.push_back(0x00);  // 成功标志字节
    data.push_back(0x07);  // 类型：字符串
    // 写入字符串长度
    uint32_t len = static_cast<uint32_t>(result.size());
    if (len < 254) {
        data.push_back(static_cast<uint8_t>(len));
    } else {
        data.push_back(254);
        data.push_back(len & 0xFF);
        data.push_back((len >> 8) & 0xFF);
    }
    // 写入字符串内容
    data.insert(data.end(), result.begin(), result.end());
    return data;
}

// 编码整数成功响应
static std::vector<uint8_t> EncodeIntResponse(int64_t value) {
    std::vector<uint8_t> data;
    data.push_back(0x00);  // 成功标志
    if (value >= -2147483648LL && value <= 2147483647LL) {
        data.push_back(0x03);  // int32
        int32_t v = static_cast<int32_t>(value);
        auto bytes = reinterpret_cast<const uint8_t*>(&v);
        data.insert(data.end(), bytes, bytes + 4);
    } else {
        data.push_back(0x04);  // int64
        auto bytes = reinterpret_cast<const uint8_t*>(&value);
        data.insert(data.end(), bytes, bytes + 8);
    }
    return data;
}

// 编码错误响应
static std::vector<uint8_t> EncodeErrorResponse(
    const std::string& code,
    const std::string& message
) {
    std::vector<uint8_t> data;
    data.push_back(0x01);  // 错误标志字节
    // 写入错误码字符串
    data.push_back(0x07);
    data.push_back(static_cast<uint8_t>(code.size()));
    data.insert(data.end(), code.begin(), code.end());
    // 写入错误消息字符串
    data.push_back(0x07);
    data.push_back(static_cast<uint8_t>(message.size()));
    data.insert(data.end(), message.begin(), message.end());
    // 写入 null 作为 details
    data.push_back(0x00);
    return data;
}

// ===== 全局状态（模拟游戏引擎）=====
static int g_game_score = 0;
static std::string g_current_scene = "title_screen";
static bool g_game_paused = false;

// ===== Platform Message 回调 =====
void HandlePlatformMessage(
    const FlutterPlatformMessage* message,
    void* user_data
) {
    FlutterEngine engine = static_cast<FlutterEngine>(user_data);
    std::string channel(message->channel);

    // 只处理我们的通道
    if (channel != "com.krkr2/game_engine") {
        // 未知通道，返回 null 表示未处理
        FlutterEngineSendPlatformMessageResponse(
            engine, message->response_handle, nullptr, 0);
        return;
    }

    // 解析方法调用（StandardMethodCodec 格式）
    const uint8_t* ptr = message->message;
    // 第一个字段是方法名（string 类型）
    DartValue method_val = ReadValue(ptr);
    std::string method = method_val.string_val;

    std::cout << "[Channel] 收到方法调用: " << method << std::endl;

    std::vector<uint8_t> response;

    if (method == "getScore") {
        // 返回当前分数
        response = EncodeIntResponse(g_game_score);
    } else if (method == "setScore") {
        // 读取参数（int）
        DartValue arg = ReadValue(ptr);
        g_game_score = static_cast<int>(arg.int_val);
        std::cout << "  分数已设置为: " << g_game_score << std::endl;
        response = EncodeSuccessResponse("OK");
    } else if (method == "getCurrentScene") {
        response = EncodeSuccessResponse(g_current_scene);
    } else if (method == "pauseGame") {
        g_game_paused = true;
        std::cout << "  游戏已暂停" << std::endl;
        response = EncodeSuccessResponse("paused");
    } else if (method == "resumeGame") {
        g_game_paused = false;
        std::cout << "  游戏已恢复" << std::endl;
        response = EncodeSuccessResponse("resumed");
    } else {
        response = EncodeErrorResponse(
            "UNIMPLEMENTED",
            "方法 '" + method + "' 未实现"
        );
    }

    // 发送响应
    FlutterEngineSendPlatformMessageResponse(
        engine,
        message->response_handle,
        response.data(),
        response.size()
    );
}

// 注册到 FlutterProjectArgs
// args.platform_message_callback = HandlePlatformMessage;
```

### 示例 2：Dart 侧 MethodChannel 调用

对应上面的 C++ 处理器，Dart 侧的调用代码如下：

```dart
// lib/game_bridge.dart
// Dart 侧通过 MethodChannel 调用 C++ 游戏引擎功能
import 'package:flutter/services.dart';

/// KrKr2 游戏引擎桥接类
/// 封装所有与 C++ 宿主的 MethodChannel 通信
class GameBridge {
  // 通道名称必须与 C++ 侧完全一致
  static const _channel = MethodChannel('com.krkr2/game_engine');

  /// 获取当前游戏分数
  static Future<int> getScore() async {
    // invokeMethod 返回 Future，自动处理异步等待
    final int score = await _channel.invokeMethod('getScore');
    return score;
  }

  /// 设置游戏分数
  static Future<void> setScore(int score) async {
    await _channel.invokeMethod('setScore', score);
  }

  /// 获取当前场景名称
  static Future<String> getCurrentScene() async {
    final String scene = await _channel.invokeMethod('getCurrentScene');
    return scene;
  }

  /// 暂停游戏
  static Future<void> pauseGame() async {
    await _channel.invokeMethod('pauseGame');
  }

  /// 恢复游戏
  static Future<void> resumeGame() async {
    await _channel.invokeMethod('resumeGame');
  }

  /// 安全调用：带错误处理
  static Future<T?> safeCall<T>(String method, [dynamic args]) async {
    try {
      return await _channel.invokeMethod<T>(method, args);
    } on PlatformException catch (e) {
      // C++ 侧返回了错误响应
      print('Platform Channel 错误: ${e.code} - ${e.message}');
      return null;
    } on MissingPluginException {
      // 通道未注册（C++ 侧没有处理此通道）
      print('MethodChannel 未注册: com.krkr2/game_engine');
      return null;
    }
  }
}

// 使用示例（在 Flutter Widget 中）
// void _loadGameInfo() async {
//   final score = await GameBridge.getScore();
//   final scene = await GameBridge.getCurrentScene();
//   setState(() {
//     _score = score;
//     _currentScene = scene;
//   });
// }
```

## EventChannel 详解

### 工作原理

EventChannel 实现了 C++ 到 Dart 的单向事件流。与 MethodChannel 的请求-响应模式不同，EventChannel 允许 C++ 侧持续推送事件到 Dart 侧，Dart 侧通过 `Stream` 监听这些事件：

```
C++ 宿主                          Dart
─────────                        ──────
                    listen()  ←── stream.listen((event) { ... })
  ┌─────┐              │
  │事件1│──推送────────►│ onData(事件1)
  │事件2│──推送────────►│ onData(事件2)
  │事件3│──推送────────►│ onData(事件3)
  └─────┘              │
                    cancel() ←── subscription.cancel()
```

### 示例 3：C++ EventChannel — 游戏状态事件流

```cpp
// event_channel_handler.cpp
// C++ 侧实现 EventChannel，向 Dart 推送游戏状态变化
#include <flutter_embedder.h>
#include <atomic>
#include <iostream>
#include <mutex>
#include <string>
#include <thread>
#include <vector>

// EventChannel 管理器
class GameEventChannel {
public:
    // 单例
    static GameEventChannel& getInstance() {
        static GameEventChannel instance;
        return instance;
    }

    // 设置 Engine 引用
    void setEngine(FlutterEngine engine) {
        engine_ = engine;
    }

    // 处理 Dart 侧的 listen / cancel 请求
    void handleStreamMessage(
        const FlutterPlatformMessage* message
    ) {
        const uint8_t* ptr = message->message;
        // StandardMethodCodec: 第一个字段是方法名
        // EventChannel 内部使用 "listen" 和 "cancel" 方法
        uint8_t tag = *ptr++;
        if (tag != 0x07) return;  // 期望字符串
        // 读取方法名
        uint8_t len = *ptr++;
        std::string method(reinterpret_cast<const char*>(ptr), len);
        ptr += len;

        if (method == "listen") {
            std::cout << "[EventChannel] Dart 开始监听" << std::endl;
            listening_ = true;
            // 回复成功
            uint8_t success[] = {0x00, 0x00};  // 成功 + null
            FlutterEngineSendPlatformMessageResponse(
                engine_, message->response_handle,
                success, sizeof(success));
        } else if (method == "cancel") {
            std::cout << "[EventChannel] Dart 取消监听" << std::endl;
            listening_ = false;
            uint8_t success[] = {0x00, 0x00};
            FlutterEngineSendPlatformMessageResponse(
                engine_, message->response_handle,
                success, sizeof(success));
        }
    }

    // 从 C++ 侧推送事件到 Dart
    void pushEvent(const std::string& event_type,
                   const std::string& data) {
        if (!listening_ || !engine_) return;

        // 构造事件消息（StandardMethodCodec 成功响应格式）
        std::vector<uint8_t> payload;
        payload.push_back(0x00);  // 成功标志
        // 事件内容：编码为字符串
        std::string event_json =
            "{\"type\":\"" + event_type + "\","
            "\"data\":\"" + data + "\"}";
        payload.push_back(0x07);  // 字符串类型
        uint32_t json_len = static_cast<uint32_t>(event_json.size());
        if (json_len < 254) {
            payload.push_back(static_cast<uint8_t>(json_len));
        } else {
            payload.push_back(254);
            payload.push_back(json_len & 0xFF);
            payload.push_back((json_len >> 8) & 0xFF);
        }
        payload.insert(payload.end(),
                       event_json.begin(), event_json.end());

        // 发送平台消息（Engine → Dart）
        FlutterPlatformMessage msg = {};
        msg.struct_size = sizeof(FlutterPlatformMessage);
        msg.channel = "com.krkr2/game_events";
        msg.message = payload.data();
        msg.message_size = payload.size();
        msg.response_handle = nullptr;  // 事件不需要响应

        FlutterEngineSendPlatformMessage(engine_, &msg);
    }

    // 推送游戏状态变化
    void notifySceneChanged(const std::string& scene_name) {
        pushEvent("scene_changed", scene_name);
    }

    void notifyScoreChanged(int score) {
        pushEvent("score_changed", std::to_string(score));
    }

    void notifyError(const std::string& error_msg) {
        pushEvent("error", error_msg);
    }

private:
    FlutterEngine engine_ = nullptr;
    std::atomic<bool> listening_{false};
};

// 在游戏逻辑中调用：
// GameEventChannel::getInstance().notifySceneChanged("chapter_02");
// GameEventChannel::getInstance().notifyScoreChanged(1500);
```

Dart 侧监听：

```dart
// lib/game_events.dart
// Dart 侧监听 C++ 推送的游戏事件
import 'dart:convert';
import 'package:flutter/services.dart';

class GameEvents {
  static const _eventChannel = EventChannel('com.krkr2/game_events');

  /// 监听游戏事件流
  static Stream<GameEvent> get onGameEvent {
    return _eventChannel
        .receiveBroadcastStream()
        .map((dynamic event) {
      // 解析 JSON 事件数据
      final Map<String, dynamic> data = json.decode(event as String);
      return GameEvent(
        type: data['type'] as String,
        data: data['data'] as String,
      );
    });
  }
}

/// 游戏事件数据类
class GameEvent {
  final String type;  // 事件类型：scene_changed / score_changed / error
  final String data;  // 事件数据

  GameEvent({required this.type, required this.data});
}

// 在 Widget 中使用：
// StreamBuilder<GameEvent>(
//   stream: GameEvents.onGameEvent,
//   builder: (context, snapshot) {
//     if (snapshot.hasData) {
//       final event = snapshot.data!;
//       return Text('事件: ${event.type} - ${event.data}');
//     }
//     return const Text('等待游戏事件...');
//   },
// )
```

---

## 外部纹理机制（External Texture）

前面我们学习了 Platform Channel 的消息通信，它解决了 C++ 与 Dart 之间"传数据"的问题。但对于游戏引擎嵌入 Flutter 的场景，还有一个更核心的需求：**如何把游戏引擎渲染的画面（像素数据）高效地显示在 Flutter 的 Widget 树中？**

这就是 Flutter 外部纹理（External Texture）机制要解决的问题。外部纹理允许宿主应用（即我们的 C++ 游戏引擎）将 GPU 纹理或像素缓冲区注册到 Flutter 引擎中，然后在 Dart 侧通过 `Texture` Widget 直接显示，**无需将像素数据从 GPU 下载到 CPU 再上传回 GPU**——这对于实时渲染场景至关重要。

### 外部纹理的工作原理

Flutter 的外部纹理机制基于**纹理 ID 映射**：宿主应用告诉 Flutter 引擎"我有一个纹理，ID 是 N"，Flutter 引擎在渲染时回调宿主应用获取该纹理的实际 GPU 句柄（如 OpenGL 纹理 ID），然后直接在 Skia（Flutter 的 2D 渲染引擎，负责将 Widget 树光栅化为像素）渲染管线中采样该纹理。

整个流程如下：

```
┌─────────────────────────────────────────────────────────────┐
│                    外部纹理工作流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  C++ 宿主侧                     Flutter 引擎                │
│  ──────────                     ──────────                  │
│                                                             │
│  1. 创建 GPU 纹理               2. 注册纹理                  │
│     (OpenGL/Vulkan/Metal)          RegisterExternalTexture  │
│          │                              │                   │
│          ▼                              ▼                   │
│  3. 渲染游戏画面到纹理           4. Dart 创建 Texture Widget  │
│     (FBO/RenderTarget)              Texture(textureId: N)   │
│          │                              │                   │
│          ▼                              ▼                   │
│  5. 通知 Flutter 纹理已更新      6. Flutter 调用回调获取纹理  │
│     MarkExternalTextureFrameAvailable   gl_external_texture │
│          │                              _frame_callback     │
│          ▼                              │                   │
│  7. 回调返回 OpenGL 纹理 ID      ◄──────┘                   │
│     + 宽高 + 像素格式                                        │
│          │                                                  │
│          ▼                                                  │
│  8. Skia 直接采样该纹理 → 显示在 Widget 树中                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Embedder API 纹理相关函数

Flutter Embedder API 提供了三个核心函数来管理外部纹理：

```cpp
// embedder.h 中的纹理管理 API

// 1. 注册外部纹理 —— 告诉 Flutter "我有一个 ID 为 texture_identifier 的纹理"
//    必须在 Flutter Engine 运行后调用
//    返回 kSuccess 表示注册成功
FlutterEngineResult FlutterEngineRegisterExternalTexture(
    FLUTTER_API_SYMBOL(FlutterEngine) engine,  // Flutter 引擎句柄
    int64_t texture_identifier                  // 纹理 ID（宿主自己分配，从 1 开始）
);

// 2. 标记纹理帧已就绪 —— 告诉 Flutter "纹理 N 有新内容了，下次渲染时来取"
//    每当游戏引擎更新了纹理内容后调用此函数
//    Flutter 会在下一个 VSync 时回调获取纹理数据
FlutterEngineResult FlutterEngineMarkExternalTextureFrameAvailable(
    FLUTTER_API_SYMBOL(FlutterEngine) engine,
    int64_t texture_identifier
);

// 3. 注销外部纹理 —— 告诉 Flutter "纹理 N 不再使用了"
//    在销毁 GPU 纹理资源前必须先调用此函数
//    否则 Flutter 渲染时会访问已释放的纹理，导致崩溃
FlutterEngineResult FlutterEngineUnregisterExternalTexture(
    FLUTTER_API_SYMBOL(FlutterEngine) engine,
    int64_t texture_identifier
);
```

纹理生命周期状态机如下：

```
┌──────────┐  Register   ┌────────────┐  MarkFrame   ┌───────────┐
│ 未注册    │────────────►│ 已注册/空闲 │─────────────►│ 帧已就绪   │
│ (无效ID)  │             │ (等待内容)  │◄─────────────│ (待采样)   │
└──────────┘             └────────────┘  回调完成      └───────────┘
                               │                           │
                               │ Unregister                │ Unregister
                               ▼                           ▼
                          ┌──────────┐               ┌──────────┐
                          │ 已注销    │               │ 已注销    │
                          └──────────┘               └──────────┘
```

### 纹理回调结构体

当 Flutter 需要获取外部纹理的实际 GPU 数据时，会调用初始化引擎时注册的回调函数。回调需要填充 `FlutterOpenGLTexture` 结构体：

```cpp
// FlutterOpenGLTexture —— 描述一个 OpenGL 外部纹理的完整信息
typedef struct {
    // OpenGL 纹理目标类型，通常是 GL_TEXTURE_2D (0x0DE1)
    // 也可以是 GL_TEXTURE_RECTANGLE (0x84F5) 或 GL_TEXTURE_EXTERNAL_OES (0x8D65)
    uint32_t target;

    // OpenGL 纹理名称（即 glGenTextures 返回的纹理 ID）
    // Flutter 的 Skia 渲染器会用 glBindTexture(target, name) 绑定此纹理
    uint32_t name;

    // 纹理格式，通常是 GL_RGBA8 (0x8058) 或 GL_BGRA8_EXT
    uint32_t format;

    // 用户数据指针，会原样传递给 destruction_callback
    // 可用于传递清理所需的上下文信息
    void* user_data;

    // 纹理使用完毕后的销毁回调
    // Flutter 不再需要此帧的纹理数据时调用
    // 注意：这不是销毁纹理本身，而是释放"这一帧"的临时资源
    VoidCallback destruction_callback;

    // 纹理的宽度（像素）
    size_t width;

    // 纹理的高度（像素）
    size_t height;
} FlutterOpenGLTexture;
```

初始化 Flutter 引擎时，需要在 `FlutterOpenGLRendererConfig` 中注册纹理回调：

```cpp
// 在 FlutterOpenGLRendererConfig 中设置纹理回调
FlutterOpenGLRendererConfig opengl_config = {};
opengl_config.struct_size = sizeof(FlutterOpenGLRendererConfig);

// ... 其他回调（make_current, clear_current, present 等）...

// 外部纹理回调 —— Flutter 需要纹理数据时调用
// 参数：
//   texture_id: 之前 Register 时传入的 ID
//   width/height: Flutter 期望的纹理尺寸
//   texture_out: 需要填充的纹理信息
// 返回 true 表示纹理数据已填充，false 表示该纹理不可用
opengl_config.gl_external_texture_frame_callback =
    [](void* user_data,
       int64_t texture_id,
       size_t width,
       size_t height,
       FlutterOpenGLTexture* texture_out) -> bool {
    // 根据 texture_id 查找对应的纹理提供者
    auto* manager = static_cast<TextureManager*>(user_data);
    return manager->resolveTexture(texture_id, width, height, texture_out);
};
```

> **关键点**：`gl_external_texture_frame_callback` 在 Flutter 的渲染线程（Raster Runner）上调用，不是在平台线程上。因此回调内部必须注意线程安全，不能直接操作主线程的 UI 资源。

---

## 示例 4：C++ 外部纹理提供者

下面是一个完整的 C++ 外部纹理管理器实现。它管理多个纹理 ID 的注册/注销，并在 Flutter 回调时提供 OpenGL 纹理句柄：

```cpp
// texture_manager.h
// 管理 Flutter 外部纹理的注册、更新和生命周期
#pragma once

#include <flutter_embedder.h>
#include <GL/gl.h>
#include <mutex>
#include <unordered_map>
#include <functional>
#include <cstdint>
#include <cstring>
#include <iostream>

// 单个纹理的描述信息
struct TextureInfo {
    GLuint gl_texture_id;   // OpenGL 纹理名称
    uint32_t width;         // 纹理宽度（像素）
    uint32_t height;        // 纹理高度（像素）
    GLenum format;          // 像素格式（GL_RGBA8 等）
    bool frame_available;   // 是否有新帧可用
};

class TextureManager {
public:
    explicit TextureManager(FlutterEngine engine)
        : engine_(engine), next_texture_id_(1) {}

    ~TextureManager() {
        // 析构时注销所有纹理
        std::lock_guard<std::mutex> lock(mutex_);
        for (auto& [id, info] : textures_) {
            FlutterEngineUnregisterExternalTexture(engine_, id);
            // 注意：不在这里删除 GL 纹理，因为 GL 上下文可能已销毁
        }
        textures_.clear();
    }

    /// 注册一个新的外部纹理
    /// @param gl_texture OpenGL 纹理 ID（由调用者创建并管理）
    /// @param width 纹理宽度
    /// @param height 纹理高度
    /// @return Flutter 纹理 ID（传给 Dart 的 Texture Widget）
    int64_t registerTexture(GLuint gl_texture,
                            uint32_t width,
                            uint32_t height) {
        std::lock_guard<std::mutex> lock(mutex_);

        int64_t texture_id = next_texture_id_++;

        // 记录纹理信息
        TextureInfo info{};
        info.gl_texture_id = gl_texture;
        info.width = width;
        info.height = height;
        info.format = GL_RGBA8;
        info.frame_available = false;
        textures_[texture_id] = info;

        // 向 Flutter 引擎注册
        FlutterEngineResult result =
            FlutterEngineRegisterExternalTexture(engine_, texture_id);
        if (result != kSuccess) {
            std::cerr << "注册外部纹理失败: " << result << std::endl;
            textures_.erase(texture_id);
            return -1;
        }

        std::cout << "注册外部纹理成功: flutter_id=" << texture_id
                  << " gl_id=" << gl_texture
                  << " size=" << width << "x" << height << std::endl;
        return texture_id;
    }

    /// 通知 Flutter 纹理内容已更新
    /// 游戏引擎每次渲染完一帧到 FBO 后调用此方法
    void markFrameAvailable(int64_t texture_id) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            auto it = textures_.find(texture_id);
            if (it == textures_.end()) return;
            it->second.frame_available = true;
        }
        // 通知 Flutter —— 此调用是线程安全的
        FlutterEngineMarkExternalTextureFrameAvailable(engine_, texture_id);
    }

    /// 注销外部纹理
    void unregisterTexture(int64_t texture_id) {
        std::lock_guard<std::mutex> lock(mutex_);
        auto it = textures_.find(texture_id);
        if (it == textures_.end()) return;

        FlutterEngineUnregisterExternalTexture(engine_, texture_id);
        textures_.erase(it);
        std::cout << "注销外部纹理: " << texture_id << std::endl;
    }

    /// Flutter 渲染线程回调 —— 提供纹理数据
    /// 此方法在 Raster 线程上调用，必须线程安全
    bool resolveTexture(int64_t texture_id,
                        size_t width,
                        size_t height,
                        FlutterOpenGLTexture* texture_out) {
        std::lock_guard<std::mutex> lock(mutex_);

        auto it = textures_.find(texture_id);
        if (it == textures_.end()) {
            return false;  // 未知纹理 ID
        }

        TextureInfo& info = it->second;

        // 填充 Flutter 纹理结构体
        texture_out->target = GL_TEXTURE_2D;          // 纹理目标类型
        texture_out->name = info.gl_texture_id;        // OpenGL 纹理 ID
        texture_out->format = info.format;             // 像素格式
        texture_out->width = info.width;               // 宽度
        texture_out->height = info.height;             // 高度
        texture_out->user_data = nullptr;              // 无额外数据
        texture_out->destruction_callback = nullptr;   // 纹理由我们管理，无需回调销毁

        info.frame_available = false;  // 重置标志
        return true;
    }

    /// 更新纹理的 GL 句柄（当游戏引擎重新创建纹理时使用）
    void updateGLTexture(int64_t texture_id,
                         GLuint new_gl_texture,
                         uint32_t width,
                         uint32_t height) {
        std::lock_guard<std::mutex> lock(mutex_);
        auto it = textures_.find(texture_id);
        if (it == textures_.end()) return;

        it->second.gl_texture_id = new_gl_texture;
        it->second.width = width;
        it->second.height = height;
    }

private:
    FlutterEngine engine_;           // Flutter 引擎句柄
    int64_t next_texture_id_;        // 下一个可用的纹理 ID
    std::mutex mutex_;               // 保护 textures_ 的互斥锁
    std::unordered_map<int64_t, TextureInfo> textures_;  // 纹理 ID → 信息
};
```

使用示例——将游戏引擎画面渲染到 FBO（帧缓冲对象，即 GPU 上的一块离屏画布），然后将 FBO 的颜色附件纹理注册为 Flutter 外部纹理：

```cpp
// game_renderer.cpp
// 演示：将游戏画面渲染到 FBO，然后通过 TextureManager 提供给 Flutter
#include "texture_manager.h"
#include <GL/glew.h>
#include <iostream>

class GameRenderer {
public:
    GameRenderer(TextureManager& tex_mgr, int width, int height)
        : texture_manager_(tex_mgr)
        , width_(width)
        , height_(height)
        , fbo_(0)
        , color_texture_(0)
        , flutter_texture_id_(-1) {}

    /// 初始化 FBO 和纹理
    bool initialize() {
        // 创建 OpenGL 纹理作为 FBO 的颜色附件
        glGenTextures(1, &color_texture_);
        glBindTexture(GL_TEXTURE_2D, color_texture_);
        // 分配 RGBA 纹理存储（不填充初始数据）
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA8,
                     width_, height_, 0,
                     GL_RGBA, GL_UNSIGNED_BYTE, nullptr);
        // 设置采样参数——线性过滤，边缘钳制
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);

        // 创建 FBO 并附加颜色纹理
        glGenFramebuffers(1, &fbo_);
        glBindFramebuffer(GL_FRAMEBUFFER, fbo_);
        glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,
                               GL_TEXTURE_2D, color_texture_, 0);

        // 检查 FBO 完整性
        GLenum status = glCheckFramebufferStatus(GL_FRAMEBUFFER);
        if (status != GL_FRAMEBUFFER_COMPLETE) {
            std::cerr << "FBO 不完整: 0x" << std::hex << status << std::endl;
            return false;
        }
        glBindFramebuffer(GL_FRAMEBUFFER, 0);  // 恢复默认帧缓冲

        // 向 Flutter 注册此纹理
        flutter_texture_id_ = texture_manager_.registerTexture(
            color_texture_, width_, height_);

        if (flutter_texture_id_ < 0) {
            std::cerr << "Flutter 纹理注册失败" << std::endl;
            return false;
        }

        std::cout << "GameRenderer 初始化完成: "
                  << "FBO=" << fbo_
                  << " texture=" << color_texture_
                  << " flutter_id=" << flutter_texture_id_
                  << " size=" << width_ << "x" << height_ << std::endl;
        return true;
    }

    /// 渲染一帧游戏画面
    void renderFrame() {
        // 绑定 FBO —— 后续所有绘制都写入我们的纹理
        glBindFramebuffer(GL_FRAMEBUFFER, fbo_);
        glViewport(0, 0, width_, height_);

        // 清屏为深蓝色（模拟游戏场景背景）
        glClearColor(0.1f, 0.1f, 0.3f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT);

        // ========== 在这里绘制游戏内容 ==========
        // 实际项目中，这里调用 Cocos2d-x 的渲染管线：
        //   cocos2d::Director::getInstance()->drawScene();
        // 或者直接用 OpenGL 绘制：
        //   drawSprite(player_texture, player_x, player_y);
        //   drawText(score_text, 10, 10);
        //   drawParticles(particle_system);
        // ========================================

        // 确保 GPU 完成渲染（避免 Flutter 采样到未完成的帧）
        glFlush();

        // 恢复默认帧缓冲
        glBindFramebuffer(GL_FRAMEBUFFER, 0);

        // 通知 Flutter：纹理有新内容了
        texture_manager_.markFrameAvailable(flutter_texture_id_);
    }

    /// 获取 Flutter 纹理 ID（传给 Dart 的 Texture Widget）
    int64_t getFlutterTextureId() const { return flutter_texture_id_; }

    /// 清理资源
    void cleanup() {
        if (flutter_texture_id_ >= 0) {
            texture_manager_.unregisterTexture(flutter_texture_id_);
            flutter_texture_id_ = -1;
        }
        if (fbo_) {
            glDeleteFramebuffers(1, &fbo_);
            fbo_ = 0;
        }
        if (color_texture_) {
            glDeleteTextures(1, &color_texture_);
            color_texture_ = 0;
        }
    }

    ~GameRenderer() { cleanup(); }

private:
    TextureManager& texture_manager_;
    int width_, height_;
    GLuint fbo_;              // 帧缓冲对象
    GLuint color_texture_;    // FBO 的颜色附件纹理
    int64_t flutter_texture_id_;  // Flutter 侧的纹理 ID
};
```

---

## 示例 5：Dart Texture Widget 显示外部纹理

在 Dart 侧，使用 `Texture` Widget 显示 C++ 提供的外部纹理。`Texture` Widget 接受一个 `textureId` 参数，对应 C++ 侧 `registerTexture` 返回的 ID：

```dart
// lib/game_view.dart
// Dart 侧：用 Texture Widget 显示 C++ 游戏引擎渲染的画面
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

class GameView extends StatefulWidget {
  const GameView({super.key});

  @override
  State<GameView> createState() => _GameViewState();
}

class _GameViewState extends State<GameView> {
  // 与 C++ 通信的 MethodChannel
  static const _channel = MethodChannel('com.krkr2/game_engine');

  int? _textureId;       // Flutter 外部纹理 ID
  bool _isLoading = true; // 加载状态
  String? _error;         // 错误信息

  @override
  void initState() {
    super.initState();
    _initGameTexture();
  }

  /// 初始化游戏纹理
  /// 通过 MethodChannel 请求 C++ 创建 FBO 并注册外部纹理
  Future<void> _initGameTexture() async {
    try {
      // 调用 C++ 侧的 createGameTexture 方法
      // 返回 Flutter 纹理 ID
      final int textureId = await _channel.invokeMethod(
        'createGameTexture',
        {
          'width': 1280,   // 游戏渲染分辨率
          'height': 720,
        },
      );

      setState(() {
        _textureId = textureId;
        _isLoading = false;
      });

      // 通知 C++ 开始渲染循环
      await _channel.invokeMethod('startRendering');
    } on PlatformException catch (e) {
      setState(() {
        _error = '初始化游戏纹理失败: ${e.message}';
        _isLoading = false;
      });
    }
  }

  @override
  void dispose() {
    // 通知 C++ 停止渲染并释放纹理
    if (_textureId != null) {
      _channel.invokeMethod('destroyGameTexture', {
        'textureId': _textureId,
      });
    }
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_isLoading) {
      return const Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            CircularProgressIndicator(),
            SizedBox(height: 16),
            Text('正在初始化游戏引擎...'),
          ],
        ),
      );
    }

    if (_error != null) {
      return Center(
        child: Text(_error!, style: const TextStyle(color: Colors.red)),
      );
    }

    // 核心：使用 Texture Widget 显示外部纹理
    // textureId 对应 C++ 侧 TextureManager::registerTexture 的返回值
    return AspectRatio(
      aspectRatio: 16 / 9,  // 游戏画面宽高比
      child: Texture(textureId: _textureId!),
    );
  }
}

// 完整的游戏页面：Texture 叠加 Flutter UI
class GamePage extends StatelessWidget {
  const GamePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Stack(
        children: [
          // 底层：游戏画面（C++ 渲染）
          const Positioned.fill(
            child: GameView(),
          ),

          // 顶层：Flutter UI 覆盖层（对话框、菜单等）
          Positioned(
            bottom: 0,
            left: 0,
            right: 0,
            child: Container(
              height: 200,
              color: Colors.black54,
              padding: const EdgeInsets.all(16),
              child: const Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    '第一章：序幕',
                    style: TextStyle(
                      color: Colors.white,
                      fontSize: 18,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                  SizedBox(height: 8),
                  Text(
                    '那是一个阳光明媚的早晨，樱花在微风中轻轻摇曳...',
                    style: TextStyle(color: Colors.white70, fontSize: 14),
                  ),
                ],
              ),
            ),
          ),

          // 右上角：设置按钮
          Positioned(
            top: 40,
            right: 16,
            child: IconButton(
              icon: const Icon(Icons.settings, color: Colors.white),
              onPressed: () {
                // 打开设置页面
              },
            ),
          ),
        ],
      ),
    );
  }
}
```

上面的代码展示了 KrKr2 最理想的 UI 架构：**游戏画面通过外部纹理在底层渲染，Flutter UI 作为覆盖层叠加在上方**。这样既保留了 C++ 游戏引擎的高性能渲染能力，又能享受 Flutter 的现代 UI 开发体验。

### KrKr2 纹理共享方案分析

在 KrKr2 模拟器中，Cocos2d-x 负责渲染视觉小说的背景图、立绘、特效和过渡动画。要将这些画面嵌入 Flutter UI，需要设计一个**纹理共享管线（Texture Sharing Pipeline）**——即一套让 Cocos2d-x 的渲染输出无缝传递给 Flutter 的机制。

目前 KrKr2 的渲染入口在 `cpp/core/environ/cocos2d/AppDelegate.cpp`，通过 Cocos2d-x Director 驱动场景渲染。我们有两种纹理共享策略：

**策略 A：FBO 捕获（推荐起步方案）**

```
Cocos2d-x 渲染管线                Flutter 引擎
────────────────                ────────────
     │                               │
     ▼                               │
绑定自定义 FBO                        │
(替换默认帧缓冲)                      │
     │                               │
     ▼                               │
Director::drawScene()                │
(所有图层渲染到 FBO)                   │
     │                               │
     ▼                               │
解绑 FBO                             │
     │                               │
     ▼                               ▼
TextureManager::markFrameAvailable ──► gl_external_texture_frame_callback
                                      │
                                      ▼
                                   Skia 采样 FBO 纹理
                                      │
                                      ▼
                                   Texture Widget 显示
```

这种方案的优点是**对 Cocos2d-x 代码改动最小**——只需在 `Director::drawScene()` 调用前后加上 FBO 绑定/解绑逻辑。缺点是多了一次全屏 FBO 拷贝的开销，但对于视觉小说这种渲染负载较低的场景完全可以接受。

**策略 B：共享 GL 上下文（高级方案）**

```
Cocos2d-x 和 Flutter 共享同一个 EGL/WGL/CGL 上下文
  → Cocos2d-x 创建的纹理可以直接被 Flutter 引用
  → 无需额外拷贝，零开销
  → 但需要仔细管理 GL 状态机，避免两个引擎互相干扰
```

策略 B 性能最优，但实现复杂度高，容易出现 GL 状态泄漏（state leaking）问题——一个引擎修改了 GL 状态（如混合模式、视口、着色器程序），另一个引擎不知情导致渲染错误。建议先用策略 A 验证架构，稳定后再迁移到策略 B。

---

## 跨平台纹理共享差异

外部纹理的底层实现因平台的图形 API 不同而有很大差异。下表对比了四个目标平台的纹理共享机制：

| 平台 | 图形 API | 纹理共享方式 | 关键 API | 注意事项 |
|------|---------|------------|---------|---------|
| **Windows** | OpenGL (WGL) / ANGLE | 共享 WGL 上下文或 ANGLE 的 EGL 上下文 | `wglShareLists()`、`eglCreateContext(share_context)` | ANGLE 将 GL 调用转译为 D3D11，性能好但调试困难 |
| **Linux** | OpenGL (EGL/GLX) | EGL 图像共享（EGLImage）或共享 GL 上下文 | `eglCreateImage()`、`glEGLImageTargetTexture2DOES()` | Mesa 驱动对 EGLImage 支持良好；Nvidia 专有驱动需要额外配置 |
| **macOS** | Metal / OpenGL (CGL) | IOSurface 共享纹理 | `IOSurfaceCreate()`、`CGLTexImageIOSurface2D()` | macOS 12+ 已废弃 OpenGL，推荐用 Metal；Flutter macOS 默认使用 Metal 后端 |
| **Android** | OpenGL ES (EGL) | SurfaceTexture / HardwareBuffer | `AHardwareBuffer_allocate()`、`EGL_ANDROID_get_native_client_buffer` | Android 8.0+ 支持 HardwareBuffer 零拷贝共享；低版本用 SurfaceTexture（有额外拷贝） |

### Windows 平台详解

Windows 上，Flutter 默认使用 ANGLE（Almost Native Graphics Layer Engine，一个将 OpenGL ES 调用转译为 DirectX 的兼容层）作为渲染后端。这意味着外部纹理共享需要通过 ANGLE 的 EGL 接口：

```cpp
// Windows (ANGLE) 纹理共享示例
// ANGLE 提供的 EGL 上下文可以与宿主应用共享

// 1. 获取 ANGLE 的 EGL Display（Flutter 引擎创建的）
EGLDisplay display = eglGetDisplay(EGL_DEFAULT_DISPLAY);

// 2. 创建共享上下文（与 Flutter 的渲染上下文共享纹理命名空间）
EGLContext shared_context = eglCreateContext(
    display,
    config,
    flutter_gl_context,    // Flutter 引擎的 EGL 上下文
    context_attribs        // EGL_CONTEXT_CLIENT_VERSION = 3
);

// 3. 在共享上下文中创建的纹理，Flutter 可以直接访问
eglMakeCurrent(display, surface, surface, shared_context);
GLuint shared_texture;
glGenTextures(1, &shared_texture);
// ... 渲染到 shared_texture ...
// Flutter 的 gl_external_texture_frame_callback 可以直接返回此纹理 ID
```

### Android 平台详解

Android 平台推荐使用 `HardwareBuffer`（硬件缓冲区，Android 8.0+ 提供的高性能跨进程/跨 API 共享机制），它支持在 CPU、GPU、摄像头、视频解码器之间零拷贝共享图像数据：

```cpp
// Android HardwareBuffer 纹理共享示例
#include <android/hardware_buffer.h>
#include <EGL/egl.h>
#include <EGL/eglext.h>

// 1. 分配 HardwareBuffer
AHardwareBuffer_Desc desc = {};
desc.width = 1280;
desc.height = 720;
desc.layers = 1;
desc.format = AHARDWAREBUFFER_FORMAT_R8G8B8A8_UNORM;  // RGBA8
desc.usage = AHARDWAREBUFFER_USAGE_GPU_SAMPLED_IMAGE   // GPU 可采样
           | AHARDWAREBUFFER_USAGE_GPU_COLOR_OUTPUT;    // GPU 可写入

AHardwareBuffer* buffer = nullptr;
int result = AHardwareBuffer_allocate(&desc, &buffer);

// 2. 将 HardwareBuffer 包装为 EGLImage
EGLClientBuffer client_buffer =
    eglGetNativeClientBufferANDROID(buffer);  // 获取 EGL 客户端缓冲区

EGLImageKHR image = eglCreateImageKHR(
    display,
    EGL_NO_CONTEXT,                   // 不绑定到特定上下文
    EGL_NATIVE_BUFFER_ANDROID,        // Android 原生缓冲区类型
    client_buffer,
    nullptr                           // 无额外属性
);

// 3. 将 EGLImage 绑定到 OpenGL 纹理
GLuint texture;
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_2D, texture);
glEGLImageTargetTexture2DOES(GL_TEXTURE_2D, image);
// 现在 texture 和 buffer 指向同一块 GPU 内存
// Cocos2d-x 渲染到 texture → Flutter 直接采样 → 零拷贝！
```

> **性能提示**：HardwareBuffer 是 Android 上性能最佳的纹理共享方式。对于 Android 8.0 以下的设备，需要退化为 `SurfaceTexture`（基于 GL_TEXTURE_EXTERNAL_OES 的纹理共享），但会有一次额外的 GPU-to-GPU 拷贝。

---

## 常见错误与排查

### 错误 1：Channel 消息无响应

**现象**：Dart 调用 `MethodChannel.invokeMethod()` 后一直等待，C++ 侧没有收到消息。

**原因**：Channel 名称不匹配，或 C++ 侧没有在正确的时机注册消息处理器。

**排查步骤**：

```cpp
// 1. 确认 Channel 名称完全一致（区分大小写！）
// Dart 侧：
const channel = MethodChannel('com.krkr2/game_engine');  // 注意斜杠方向
// C++ 侧：
const char* channel_name = "com.krkr2/game_engine";  // 必须完全一致

// 2. 确认在 FlutterEngineRun 之后才注册处理器
// 错误：在引擎启动前注册 → 引擎会覆盖/忽略
FlutterEngineSendPlatformMessage(engine, &message);  // engine 必须已 Running

// 3. 确认回复消息在 Platform 线程上发送
// Flutter 的 Platform Channel 消息默认在 Platform Runner 上处理
// 如果你在其他线程处理消息，回复时必须切回 Platform 线程
FlutterEngineRunTask(engine, &task);  // 在 Platform Runner 上执行
```

### 错误 2：外部纹理显示为黑色或闪烁

**现象**：Dart 的 `Texture` Widget 能显示，但内容全黑或帧间闪烁。

**原因**：

1. **GL 上下文不匹配**：渲染纹理的 GL 上下文与 Flutter 的 GL 上下文不共享。两个独立的 GL 上下文创建的纹理互不可见——纹理 ID 虽然是数字，但它只在创建它的上下文（及其共享组）内有效。
2. **未调用 `glFlush()`/`glFinish()`**：渲染命令还在 GPU 命令队列中排队，Flutter 采样时纹理内容尚未写入完成。
3. **纹理格式不匹配**：`FlutterOpenGLTexture.format` 与实际纹理格式不一致。

**修复**：

```cpp
// 确保 GL 上下文共享
// 创建宿主 GL 上下文时，传入 Flutter 的上下文作为共享上下文
EGLContext host_context = eglCreateContext(
    display, config,
    flutter_context,   // ← 关键：共享 Flutter 的上下文
    attribs
);

// 渲染完成后强制同步
glFlush();   // 提交所有 GL 命令到 GPU
// 或者用 glFinish() —— 等待 GPU 完成所有命令（更安全但更慢）

// 然后再 markFrameAvailable
texture_manager->markFrameAvailable(texture_id);
```

### 错误 3：Android 上纹理方向颠倒

**现象**：Android 设备上游戏画面上下颠倒。

**原因**：Android 的 SurfaceTexture 纹理坐标系与 OpenGL 标准不同。SurfaceTexture 使用的是 Android 屏幕坐标系（y 轴向下），而 OpenGL 的纹理坐标 y 轴向上。

**修复方案**：

```cpp
// 方案 A：在 Fragment Shader 中翻转 UV
// varying vec2 v_texCoord;
// void main() {
//     vec2 flipped = vec2(v_texCoord.x, 1.0 - v_texCoord.y);
//     gl_FragColor = texture2D(u_texture, flipped);
// }

// 方案 B：在 FlutterOpenGLTexture 回调中设置翻转
// Flutter 的 Skia 渲染器会根据纹理的 target 类型自动处理部分翻转
// 对于 GL_TEXTURE_EXTERNAL_OES，需要额外设置变换矩阵
texture_out->target = GL_TEXTURE_EXTERNAL_OES;  // Android SurfaceTexture

// 方案 C：渲染到 FBO 时使用翻转的投影矩阵
// 修改正交投影矩阵的 top/bottom 参数即可
glm::mat4 projection = glm::ortho(
    0.0f, (float)width,
    (float)height, 0.0f,   // 注意：height 在前（top），0 在后（bottom）
    -1.0f, 1.0f
);
```

---

## 动手实践

### 练习 1：实现简单的颜色渐变纹理提供者

**目标**：创建一个 C++ 程序，生成随时间变化的颜色渐变纹理，通过 Flutter 外部纹理显示。

**步骤**：

1. 创建一个 640×480 的 OpenGL 纹理
2. 用 `glTexSubImage2D()` 每帧上传 CPU 生成的渐变色像素数据
3. 通过 `TextureManager` 注册为 Flutter 外部纹理
4. 在 Dart 侧用 `Texture` Widget 显示
5. 使用 `Timer.periodic` 每秒打印帧率

**验证**：你应该能在 Flutter 窗口中看到一个平滑变化的彩色渐变画面，帧率稳定在 30-60 FPS。

### 练习 2：实现双向 Platform Channel 通信

**目标**：实现一个完整的双向通信示例——Dart 发送命令给 C++，C++ 通过 EventChannel 推送状态更新。

**步骤**：

1. C++ 侧：实现 MethodChannel 处理器，支持 `play`、`pause`、`setVolume` 三个命令
2. C++ 侧：实现 EventChannel，每秒推送当前播放时间和状态
3. Dart 侧：创建 UI 界面，包含播放/暂停按钮、音量滑块和时间显示
4. 使用 `StreamBuilder` 监听 EventChannel 实时更新 UI

**验证**：点击播放按钮后，时间显示应开始递增；拖动音量滑块后，C++ 侧应打印新的音量值。

---

## 对照项目源码

KrKr2 项目中，虽然尚未实现 Flutter 嵌入，但以下代码是未来集成时的关键参考：

相关文件：

- `cpp/core/environ/cocos2d/AppDelegate.cpp` — 应用启动入口和渲染循环。未来 Flutter 嵌入时，需要在这里初始化 Flutter 引擎、设置共享 GL 上下文
- `cpp/core/environ/ui/` — 当前基于 Cocos2d 的 UI 实现（TVPMainForm、TVPMessageBoxForm 等）。Flutter 替换方案需要提供相同功能的 Dart Widget
- `ui/cocos-studio/` — Cocos Studio UI 布局文件（.csb 格式）和多语言 XML。Flutter 替换后这些文件将被 Dart Widget + ARB 国际化文件取代
- `cpp/core/environ/ConfigManager/` — 配置管理器。Platform Channel 集成时，这些配置需要通过 MethodChannel 暴露给 Dart 侧
- `cpp/core/visual/` — 视觉渲染模块（图层、位图、图片加载器）。纹理共享方案需要将这些渲染输出通过 FBO 捕获传递给 Flutter

---

## 本节小结

- **Platform Channel** 是 Flutter 与宿主应用（C++/Java/Swift）双向通信的标准机制，底层基于 `FlutterPlatformMessage` 的异步消息传递
- **三种 Channel 类型**各有用途：`MethodChannel`（一问一答的 RPC 调用）、`EventChannel`（C++ 向 Dart 的持续事件推送）、`BasicMessageChannel`（自由格式消息交换）
- **StandardMethodCodec** 使用自定义二进制编码格式（非 JSON/Protobuf），支持 null、bool、int32/int64、float64、String、Uint8List、List、Map 等类型的高效序列化
- **外部纹理（External Texture）** 是 Flutter 显示 GPU 渲染内容的核心机制：注册纹理 ID → 渲染到 FBO → 通知帧就绪 → Flutter 回调获取 GL 纹理句柄 → Skia 直接采样
- **FlutterOpenGLTexture** 结构体描述了纹理的完整信息（target、name、format、宽高、销毁回调），是 C++ 与 Flutter 渲染管线的桥梁
- **KrKr2 推荐策略**：先用 FBO 捕获方案（改动最小），将 Cocos2d-x 的渲染输出通过外部纹理传递给 Flutter；稳定后可迁移到共享 GL 上下文方案以消除拷贝开销
- **跨平台差异显著**：Windows 使用 ANGLE/EGL 共享上下文，Linux 使用 EGLImage，macOS 使用 IOSurface，Android 使用 HardwareBuffer（8.0+）或 SurfaceTexture（低版本）
- **线程安全至关重要**：`gl_external_texture_frame_callback` 在 Raster 线程调用，`FlutterPlatformMessage` 回复必须在 Platform 线程发送，跨线程操作需要互斥锁保护
- **Channel 名称必须精确匹配**：Dart 和 C++ 两侧的 Channel 名称（字符串）必须完全一致，包括大小写和斜杠方向

---

## 练习题与答案

### 题目 1：Platform Channel 消息流分析

请描述以下场景的完整消息流：Dart 侧调用 `MethodChannel('com.krkr2/audio').invokeMethod('setVolume', 0.8)`，C++ 侧处理后返回 `{success: true, currentVolume: 0.8}`。

要求：列出从 Dart 发起调用到收到返回值的每一步，包括编码、线程切换和解码过程。

<details>
<summary>查看答案</summary>

**完整消息流**：

1. **Dart 侧编码**（UI 线程 / Isolate 主线程）：
   - `MethodChannel` 使用 `StandardMethodCodec` 将方法名 `'setVolume'` 和参数 `0.8` 编码为二进制消息
   - 编码格式：`[方法名长度][方法名UTF8字节][参数类型标签0x06][double的8字节IEEE754]`
   - 调用 `dart:ui` 的 `PlatformDispatcher.sendPlatformMessage()` 发送

2. **Flutter 引擎调度**（Engine 内部）：
   - 引擎将消息投递到 Platform Runner 的任务队列
   - Platform Runner 在下一次事件循环中取出消息

3. **C++ 侧接收**（Platform 线程）：
   - 引擎调用 `FlutterPlatformMessageCallback`（初始化时通过 `FlutterProjectArgs.platform_message_callback` 注册的回调）
   - 回调收到 `FlutterPlatformMessage` 结构体，包含 channel 名称和二进制消息

4. **C++ 侧解码与处理**（Platform 线程）：
   - 解析 StandardMethodCodec 二进制格式：提取方法名 `"setVolume"` 和参数 `0.8`（double）
   - 执行音量设置逻辑
   - 构造返回值 Map `{success: true, currentVolume: 0.8}`

5. **C++ 侧编码并回复**（Platform 线程）：
   - 将返回值用 StandardMethodCodec 编码：`[0x00（成功标志）][类型标签0x0D（Map）][键值对数据]`
   - 调用 `FlutterEngineSendPlatformMessageResponse(engine, handle, data, size)` 发送回复

6. **Flutter 引擎调度**（Engine 内部）：
   - 引擎将回复消息投递回 UI Runner

7. **Dart 侧解码**（UI 线程）：
   - `StandardMethodCodec` 解码二进制回复
   - 检查首字节：`0x00` 表示成功，解析后续数据为 `Map<String, dynamic>`
   - `invokeMethod` 的 `Future` 完成，返回 `{success: true, currentVolume: 0.8}`

**线程切换总结**：UI 线程 → Platform 线程 → Platform 线程 → UI 线程（共 2 次线程切换）。

</details>

### 题目 2：外部纹理生命周期管理

请回答以下问题：

1. 如果在调用 `FlutterEngineUnregisterExternalTexture()` 之前就删除了 OpenGL 纹理（调用了 `glDeleteTextures()`），会发生什么？
2. 如果注册了外部纹理但从未调用 `MarkExternalTextureFrameAvailable()`，Dart 侧的 `Texture` Widget 会显示什么？
3. `FlutterOpenGLTexture.destruction_callback` 的作用是什么？它在什么时候被调用？

<details>
<summary>查看答案</summary>

**1. 先删除 GL 纹理再注销 Flutter 纹理**：

这是一个**未定义行为（Undefined Behavior）**，可能导致：
- **崩溃**：Flutter 的 Raster 线程在下一次渲染时调用 `gl_external_texture_frame_callback`，回调返回了已删除的纹理 ID，Skia 执行 `glBindTexture()` 时触发 GL 错误或段错误
- **渲染伪影**：GL 可能复用了被删除纹理的 ID（GL 纹理名称会被回收），导致 Flutter 显示其他纹理的内容
- **正确做法**：先调用 `FlutterEngineUnregisterExternalTexture()` → 等待 Flutter 确认不再使用 → 再调用 `glDeleteTextures()`。或者在 `destruction_callback` 中删除纹理

**2. 注册但从未 MarkFrameAvailable**：

`Texture` Widget 会显示**透明/空白**内容。原因：
- Flutter 只有在收到 `MarkExternalTextureFrameAvailable` 通知后才会调用 `gl_external_texture_frame_callback`
- 没有通知 = 没有回调 = 没有纹理数据 = Widget 显示为空
- `Texture` Widget 本身的区域仍然占据布局空间，但不渲染任何像素

**3. destruction_callback 的作用**：

`destruction_callback` 是 Flutter 在**不再需要某一帧的纹理数据时**调用的清理回调。注意区分：
- 它**不是**销毁纹理本身的回调（纹理可能被多帧复用）
- 它用于释放**单帧临时资源**，例如：如果你使用双缓冲或三缓冲，`destruction_callback` 通知你"这一帧的纹理可以被写入新数据了"
- 调用时机：Flutter 的 Raster 线程完成当前帧的渲染后，在提交帧之前调用
- `user_data` 参数会原样传递给回调，可用于标识哪个缓冲区被释放

```cpp
// 典型的 destruction_callback 用法（双缓冲场景）
void on_frame_consumed(void* user_data) {
    auto* buffer_id = static_cast<int*>(user_data);
    // 将此缓冲区标记为可写入
    buffer_pool.release(*buffer_id);
}
```

</details>

### 题目 3：设计 KrKr2 的 Flutter 通信架构

假设你需要为 KrKr2 设计完整的 Flutter 集成通信架构，请：

1. 列出至少 5 个需要通过 Platform Channel 暴露的 C++ API
2. 说明每个 API 应该用 MethodChannel 还是 EventChannel，为什么
3. 画出整体通信架构图（ASCII 图即可）

<details>
<summary>查看答案</summary>

**1. 需要暴露的 C++ API**：

| API | Channel 类型 | 原因 |
|-----|-------------|------|
| `loadScript(path)` — 加载 TJS2 脚本 | MethodChannel | 一问一答：传入路径，返回加载结果（成功/失败） |
| `saveGame(slot)` / `loadGame(slot)` — 存档管理 | MethodChannel | 一问一答：传入存档槽位，返回操作结果 |
| `setVolume(channel, value)` — 音量控制 | MethodChannel | 一问一答：设置特定声道音量，返回确认 |
| `gameStateEvents` — 游戏状态变化事件流 | EventChannel | C++ 持续推送：场景切换、对话进度、选择支出现等事件 |
| `audioProgress` — 音频播放进度 | EventChannel | C++ 持续推送：当前播放位置、总时长、波形数据等 |
| `getConfig(key)` / `setConfig(key, value)` — 配置读写 | MethodChannel | 一问一答：读取/设置配置项 |
| `pluginCall(name, method, args)` — 插件方法调用 | MethodChannel | 一问一答：桥接 TJS2 插件调用 |

**2. Channel 类型选择原则**：

- **MethodChannel**：适用于"请求-响应"模式，即 Dart 发起、C++ 处理后返回结果。特点是每次调用独立、有明确的返回值
- **EventChannel**：适用于"订阅-推送"模式，即 C++ 侧主动产生数据，Dart 侧被动接收。特点是数据流持续不断、Dart 侧用 Stream 监听

**3. 整体通信架构图**：

```
┌─────────────────────────────────────────────────────────────┐
│                      Dart (Flutter UI)                       │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │ GamePage     │  │ SettingsPage │  │ SaveLoadPage       │ │
│  │ (Texture +   │  │ (音量/配置)   │  │ (存档列表)          │ │
│  │  对话覆盖层)  │  └──────┬───────┘  └────────┬───────────┘ │
│  └──────┬───────┘         │                    │            │
│         │                 │                    │            │
│  ┌──────┴─────────────────┴────────────────────┴──────────┐ │
│  │              Platform Channel 通信层                     │ │
│  │  MethodChannel: com.krkr2/script                       │ │
│  │  MethodChannel: com.krkr2/save                         │ │
│  │  MethodChannel: com.krkr2/audio                        │ │
│  │  MethodChannel: com.krkr2/config                       │ │
│  │  MethodChannel: com.krkr2/plugin                       │ │
│  │  EventChannel:  com.krkr2/game_events                  │ │
│  │  EventChannel:  com.krkr2/audio_progress               │ │
│  │  External Texture: textureId=1 (游戏画面)               │ │
│  └────────────────────────┬───────────────────────────────┘ │
└───────────────────────────┼─────────────────────────────────┘
                            │ FlutterPlatformMessage
                            │ FlutterOpenGLTexture
                            ▼
┌───────────────────────────┼─────────────────────────────────┐
│                     Flutter Engine                           │
│              (消息路由 + Skia 渲染 + 纹理采样)                │
└───────────────────────────┼─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      C++ 宿主层                              │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │ TJS2 引擎    │  │ 音频子系统    │  │ 配置管理器          │ │
│  │ (脚本执行)   │  │ (FFmpeg)     │  │ (ConfigManager)    │ │
│  └──────┬───────┘  └──────┬───────┘  └────────┬───────────┘ │
│         │                 │                    │            │
│  ┌──────┴─────────────────┴────────────────────┴──────────┐ │
│  │            Cocos2d-x 渲染引擎 → FBO → 外部纹理          │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

</details>

---

## 下一步

下一节我们将学习另一个重要的跨平台 UI 方案——Compose Multiplatform。它基于 Kotlin 和 Skia 渲染引擎，提供了与 Flutter 不同的 C++ 互操作路径：

→ [03-Compose-Multiplatform/01-Kotlin-Native与Cinterop基础.md](../03-Compose-Multiplatform/01-Kotlin-Native与Cinterop基础.md)

