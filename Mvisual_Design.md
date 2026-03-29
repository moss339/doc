# MVisual 模块设计文档

## 1. 概述

MVisual 是 MOSS 项目的可视化数据桥接模块，负责将 MOSS 系统中的数据（如 DDS 主题、SOME/IP 事件）转换为 FoxGlove Studio 可消费的格式，实现汽车/嵌入式数据的实时可视化。

### 1.1 设计目标

- **FoxGlove 协议兼容**: 完整支持 FoxGlove WebSocket 协议
- **多数据类型支持**: 支持 Image、PointCloud、DiagnosticArray、Plot 等可视化数据类型
- **零拷贝**: 利用 mshm 共享内存避免不必要的数据拷贝
- **高性能**: 异步消息处理，支持高频率传感器数据流
- **可扩展**: 插件化设计，支持自定义消息类型

### 1.2 术语表

| 术语 | 说明 |
|------|------|
| FoxGlove Studio | 开源机器人可视化工具 |
| FoxGlove Protocol | FoxGlove 的 WebSocket JSON 协议 |
| Channel | 数据通道，类似于 DDS Topic |
| StudioBridge | FoxGlove Studio 桥接器核心类 |
| DataAdapter | 数据类型适配器基类 |
| MessagePublisher | 消息发布者，管理频道订阅 |

---

## 2. FoxGlove Studio 协议分析

### 2.1 协议概述

FoxGlove Studio 使用 WebSocket 进行通信，消息格式为 JSON。协议定义在 foxglove/ws-protocol 中。

### 2.2 消息类型

#### 2.2.1 客户端到服务端消息

| 消息类型 | Channel ID | 说明 |
|----------|------------|------|
| `subscribe` | 1 | 订阅频道 |
| `unsubscribe` | 2 | 取消订阅 |
| `advertise` | 3 | 宣布发布频道 |
| `unadvertise` | 4 | 取消发布频道 |
| `publish` | 5 | 发布消息 |
| `service` | 6 | 服务请求 |
| `serviceResponse` | 7 | 服务响应 |

#### 2.2.2 服务端到客户端消息

| 消息类型 | Channel ID | 说明 |
|----------|------------|------|
| `serverError` | 10 | 服务器错误 |
| `advertise` | 11 | 服务器确认/广播频道 |
| `unadvertise` | 12 | 服务器确认/移除频道 |
| `message` | 13 | 数据消息 |
| `serviceResponse` | 14 | 服务响应 |
| `time` | 15 | 时间同步 |

### 2.3 支持的数据类型（Schema）

| 类型 | Schema Name | 说明 |
|------|-------------|------|
| Image | `sensor_msgs/Image` | 图像数据 |
| PointCloud2 | `sensor_msgs/PointCloud2` | 点云数据 |
| LaserScan | `sensor_msgs/LaserScan` | 激光扫描 |
| DiagnosticArray | `diagnostic_msgs/DiagnosticArray` | 诊断信息 |
| Imu | `sensor_msgs/Imu` | IMU 数据 |
| NavSatFix | `sensor_msgs/NavSatFix` | GPS 数据 |
| JointState | `sensor_msgs/JointState` | 关节状态 |

---

## 3. 架构设计

### 3.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MOSS 可视化架构                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         Application                                  │    │
│  │  ┌─────────────────┐         ┌─────────────────┐                  │    │
│  │  │  DDS Publisher  │         │ SOME/IP Event   │                  │    │
│  │  └────────┬────────┘         └────────┬────────┘                  │    │
│  └────────────┼─────────────────────────────┼────────────────────────────┘    │
│               ▼                             ▼                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      StudioBridge                                    │    │
│  │  ┌──────────────────────────────────────────────────────────────┐    │    │
│  │  │                 MessageRouter                                 │    │    │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │    │    │
│  │  │  │ ChannelMgr  │  │ SessionMgr  │  │  ProtocolEncoder│  │    │    │
│  │  │  └─────────────┘  └─────────────┘  └─────────────────┘  │    │    │
│  │  └──────────────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                          │
│                                    ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    WebSocket Server (Asio)                          │    │
│  │                         port: 8765                                  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                          │
│                                    ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    FoxGlove Studio                                  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 目录结构

```
mvisual/
├── include/mvisual/
│   ├── mvisual.h              # 主包含头文件
│   ├── studio_bridge.h        # StudioBridge 主类
│   ├── session.h               # WebSocket 会话
│   ├── channel.h              # Channel 定义
│   ├── protocol.h              # FoxGlove 协议编码/解码
│   ├── adapter.h               # 数据适配器基类
│   ├── adapters/
│   │   ├── image_adapter.h     # 图像适配器
│   │   ├── pointcloud_adapter.h # 点云适配器
│   │   ├── diagnostic_adapter.h # 诊断适配器
│   │   └── plot_adapter.h      # 绘图适配器
│   ├── config.h                # 配置结构
│   └── types.h                 # 类型定义
├── src/
│   ├── studio_bridge.cpp
│   ├── session.cpp
│   ├── channel.cpp
│   ├── protocol.cpp
│   ├── adapter.cpp
│   └── adapters/
│       ├── image_adapter.cpp
│       ├── pointcloud_adapter.cpp
│       ├── diagnostic_adapter.cpp
│       └── plot_adapter.cpp
├── test/
│   └── test_studio_bridge.cpp
├── example/
│   └── basic_example.cpp
└── CMakeLists.txt
```

---

## 4. 核心组件设计

### 4.1 类型定义

```cpp
// FoxGlove 协议消息类型
enum class FoxgloveMessageType : uint8_t {
    SUBSCRIBE       = 1,
    UNSUBSCRIBE     = 2,
    ADVERTISE       = 3,
    UNADVERTISE     = 4,
    PUBLISH         = 5,
    SERVICE_REQUEST = 6,
    SERVICE_RESPONSE = 7,
    SERVER_ERROR    = 10,
    ADVERTISE_RESP  = 11,
    UNADVERTISE_RESP = 12,
    MESSAGE_DATA    = 13,
    SERVICE_RESP    = 14,
    TIME_SYNC       = 15
};

// Channel 信息
struct Channel {
    uint32_t id;
    std::string topic;
    std::string schema_name;
    std::string schema;
    Encoding encoding;
    std::string schema_data;
};

// 配置结构
struct Config {
    uint16_t port = 8765;
    size_t max_frame_size = 64 * 1024 * 1024;
    size_t max_session_count = 100;
    bool enable_tls = false;
    std::string cert_file;
    std::string key_file;
};
```

### 4.2 核心 API

```cpp
class StudioBridge : public std::enable_shared_from_this<StudioBridge> {
public:
    static BridgePtr create(const Config& config = Config());
    ~StudioBridge();

    bool start();
    void stop();
    bool is_running() const { return running_; }

    void register_adapter(AdapterPtr adapter);
    void publish(uint32_t channel_id, const void* data, size_t size);
    void publish(uint32_t channel_id, const void* data, size_t size, uint64_t timestamp);

    void set_on_session_connect(SessionCallback cb);
    void set_on_session_disconnect(SessionCallback cb);
    void set_on_error(ErrorCallback cb);
};
```

### 4.3 数据适配器基类

```cpp
class DataAdapter {
public:
    virtual ~DataAdapter() = default;
    virtual std::string get_schema_name() const = 0;
    virtual bool encode(const void* moss_data, size_t moss_size,
                       std::vector<uint8_t>& foxglove_data,
                       uint64_t& timestamp) = 0;
    virtual Channel create_channel(const std::string& topic) const = 0;
    virtual bool validate(const void* data, size_t size) const = 0;
};
```

---

## 5. FoxGlove Studio 连接指南

### 5.1 连接步骤

1. 启动 MVisual Bridge
2. 打开 FoxGlove Studio
3. 选择 "Open Connection" -> "FoxGlove Studio"
4. 输入 WebSocket URL: `ws://<bridge-ip>:8765`
5. 连接成功后自动发现可用数据通道

### 5.2 支持的可视化面板

| 面板类型 | 对应数据 | 适配器 |
|----------|----------|--------|
| Image | `/camera/*/image` | ImageAdapter |
| Point Cloud | `/lidar/*/points` | PointCloudAdapter |
| Diagnostic | `/diagnostics/*` | DiagnosticAdapter |
| Plot | `/sensor/*/data` | PlotAdapter |
| 3D | `/perception/*/objects` | (扩展) |

---

## 6. 使用示例

```cpp
#include "mvisual/studio_bridge.h"
#include "mvisual/adapters/image_adapter.h"

int main() {
    mvisual::Config config;
    config.port = 8765;

    auto bridge = mvisual::StudioBridge::create(config);
    bridge->register_adapter(std::make_shared<mvisual::ImageAdapter>());
    bridge->set_on_session_connect([](auto session) {
        printf("Client connected: %s\n", session->get_id().c_str());
    });

    if (!bridge->start()) {
        fprintf(stderr, "Failed to start StudioBridge\n");
        return 1;
    }

    printf("MVisual Bridge listening on port %d\n", config.port);

    while (bridge->is_running()) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    return 0;
}
```

---

## 7. 依赖关系

```
mvisual
├── mshm (共享内存)
├── mruntime (Node 生命周期)
├── mlog (日志)
└── Asio (WebSocket 服务器)
```

---

## 8. 构建配置

```bash
cd mvisual && mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j4
```
