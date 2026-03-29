# 配置服务器模块设计文档

## 1. 概述

config_server 模块是 MOSS 项目的配置管理核心组件，提供统一的参数存储、查询和变更通知机制。支持本地 Domain Socket 通信和云端 MQTT 两种更新通道。

### 1.1 设计目标

- **统一配置管理**: 集中存储和管理系统参数
- **跨进程通信**: 基于 Domain Socket 的高效进程间通信
- **云端同步**: 支持 MQTT 协议与云端进行配置同步
- **类型安全 SDK**: 模板化 SDK 支持多种数据类型
- **版本管理**: 配置变更版本追踪

### 1.2 术语表

| 术语 | 说明 |
|------|------|
| ConfigItem | 配置项，包含键、值、版本和时间戳 |
| ConfigStore | 配置存储，内存 + 文件持久化 |
| Domain Socket | Unix 域套接字，本地进程通信机制 |
| MQTT | Message Queuing Telemetry Transport，物联网通信协议 |
| SDK | Software Development Kit，客户端开发包 |

---

## 2. 架构设计

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         config_server                                │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │   ConfigStore    │  │ DomainSocketServer│  │   MqttInterface  │ │
│  │  (内存+文件存储)  │  │   (本地通信)      │  │   (云端通信)      │ │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘ │
│           │                    │                     │            │
│           └────────────────────┼─────────────────────┘            │
│                                │                                    │
└────────────────────────────────┼────────────────────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
        ┌──────────┐      ┌──────────┐        ┌──────────┐
        │  应用1   │      │  应用2   │        │ 云端    │
        │(SDK客户端)│      │(SDK客户端)│        │(MQTT)   │
        └──────────┘      └──────────┘        └──────────┘
              │                  │                  │
              ▼                  ▼                  ▼
        /var/run/config_server.sock        MQTT Broker
```

### 2.2 目录结构

```
config_server/
├── include/config_server/
│   ├── types.h                  # 类型定义
│   ├── config_store.h           # 配置存储核心
│   ├── config_server.h          # 主服务类
│   ├── domain_socket_server.h   # Domain Socket 服务器
│   └── mqtt_interface.h          # MQTT 接口
├── src/
│   ├── config_server.cpp        # 主服务实现
│   ├── config_store.cpp         # 存储实现
│   ├── domain_socket_server.cpp # Socket 服务器实现
│   ├── mqtt_interface.cpp        # MQTT 实现
│   └── main.cpp                 # 入口
├── sdk/
│   ├── include/config_client/
│   │   ├── types.h              # 客户端类型定义
│   │   └── config_client.h       # SDK 主类
│   └── src/
│       ├── config_client.cpp     # SDK 实现
│       └── domain_socket_client.cpp
└── CMakeLists.txt
```

---

## 3. 核心数据结构

### 3.1 类型定义 (types.h)

```cpp
namespace config_server {

struct ConfigItem {
    std::string key;        // 参数键，如 "vehicle.speed_limit"
    std::string value;      // 参数值（JSON格式存储）
    int64_t version;        // 版本号，用于更新检测
    int64_t timestamp;      // 更新时间戳
};

struct ConfigChange {
    std::vector<ConfigItem> changed_items;
    std::string source;     // "local" | "cloud" | "sdk"
};

struct CloudUpdate {
    std::vector<ConfigItem> items;
    std::string signature;  // 签名验证
};

enum class ErrorCode : int {
    SUCCESS = 0,
    NOT_FOUND = 1,
    INVALID_KEY = 2,
    INVALID_VALUE = 3,
    INTERNAL_ERROR = 4,
    VERSION_CONFLICT = 5,
    NETWORK_ERROR = 6
};

} // namespace config_server
```

### 3.2 配置存储 (config_store.h)

```cpp
class ConfigStore {
public:
    bool load_from_file(const std::string& path);
    bool save_to_file(const std::string& path);

    // 内存操作
    std::optional<ConfigItem> get(const std::string& key);
    bool set(const ConfigItem& item);
    bool remove(const std::string& key);
    std::vector<ConfigItem> get_all();

    // 版本管理
    bool update_if_newer(const std::string& key, const ConfigItem& item);

private:
    std::string file_path_;
    std::vector<ConfigItem> items_;
    mutable std::shared_mutex mutex_;
    int64_t next_version_;
};
```

---

## 4. Domain Socket 通信协议

### 4.1 Socket 路径

- **服务器**: `/var/run/config_server.sock`
- **协议**: JSON over Unix Domain Socket

### 4.2 消息格式

| 操作 | 请求格式 | 响应格式 |
|------|----------|----------|
| GET | `{"cmd":"get","key":"xxx"}` | `{"code":0,"value":"xxx","version":1}` |
| SET | `{"cmd":"set","key":"xxx","value":"yyy"}` | `{"code":0}` |
| LIST | `{"cmd":"list"}` | `{"code":0,"items":[...]}` |
| SUBSCRIBE | `{"cmd":"subscribe"}` | `{"code":0,"msg":"subscribed"}` |

### 4.3 错误码

| code | 说明 |
|------|------|
| 0 | 成功 |
| 1 | 配置项不存在 |
| 2 | 无效的键 |
| 3 | 无效的值 |
| 4 | 内部错误 |
| 5 | 版本冲突 |
| 6 | 网络错误 |

---

## 5. SDK 设计

### 5.1 模板化类型支持

SDK 使用模板支持各种基本类型：

| 类型 | 示例 |
|------|------|
| int | `client->set("vehicle.max_speed", 120)` |
| double | `client->set("sensor.threshold", 0.85)` |
| bool | `client->set("feature.enabled", true)` |
| std::string | `client->set("vehicle.name", std::string("Tesla"))` |

### 5.2 客户端 API

```cpp
namespace config_client {

template<typename T>
struct TypedConfigValue {
    T value;
    int64_t version;
    bool valid;
};

class ConfigClient : public std::enable_shared_from_this<ConfigClient> {
public:
    static std::shared_ptr<ConfigClient> create(
        const std::string& socket_path = "/var/run/config_server.sock");

    // 模板化同步 API
    template<typename T>
    std::optional<TypedConfigValue<T>> get(const std::string& key);

    template<typename T>
    bool set(const std::string& key, const T& value);

    // 异步 API
    template<typename T>
    std::future<std::optional<TypedConfigValue<T>>> async_get(const std::string& key);

    template<typename T>
    std::future<bool> async_set(const std::string& key, const T& value);

    // 变更回调
    using ChangeCallback = std::function<void(const ConfigChange&)>;
    void set_change_callback(ChangeCallback callback);

private:
    std::string socket_path_;
    int sock_fd_;
    bool connected_;
    ChangeCallback change_callback_;
};

} // namespace config_client
```

### 5.3 使用示例

```cpp
auto client = ConfigClient::create();

// 模板化调用
client->set("vehicle.speed_limit", 120);
client->set("vehicle.name", std::string("Tesla"));
client->set("vehicle.enable_feature", true);

auto speed = client->get<int>("vehicle.speed_limit");
if (speed && speed->valid) {
    int val = speed->value;
}

// 异步调用
auto future = client->async_set("vehicle.speed_limit", 150);
bool result = future.get();
```

---

## 6. MQTT 云端同步

### 6.1 MQTT Topics

| Topic | 方向 | 说明 |
|-------|------|------|
| `config/cloud/update` | 云端 → 设备 | 云端下发配置更新 |
| `config/device/report` | 设备 → 云端 | 设备上报配置状态 |

### 6.2 云端更新消息格式

```json
{
    "items": [
        {
            "key": "vehicle.speed_limit",
            "value": "120",
            "version": 5,
            "timestamp": 1711234567
        }
    ],
    "signature": "base64_encoded_signature"
}
```

### 6.3 MQTT 接口

```cpp
class MqttInterface {
public:
    bool connect(const std::string& broker_host, uint16_t port,
                 const std::string& client_id);
    bool subscribe(const std::string& topic);
    bool publish(const std::string& topic, const std::string& payload);
    void disconnect();

    void set_message_callback(MessageCallback callback);
    void set_change_callback(std::function<void(const ConfigChange&)> callback);

private:
    ConfigStore& store_;
    struct mosquitto* mosq_;
    std::atomic<bool> connected_;
};
```

---

## 7. 配置持久化

### 7.1 文件格式 (JSON)

```json
{
    "name": "config_server",
    "next_version": 10,
    "items": [
        {
            "key": "vehicle.speed_limit",
            "value": "120",
            "version": 1,
            "timestamp": 1711234567
        }
    ]
}
```

### 7.2 加载和保存

- **启动时**: 从文件加载配置到内存
- **修改时**: 内存操作后异步保存到文件
- **版本控制**: 使用 `next_version` 全局递增

---

## 8. 线程安全

### 8.1 锁策略

| 组件 | 锁类型 | 说明 |
|------|--------|------|
| ConfigStore | shared_mutex | 读写锁支持并发读 |
| DomainSocketServer | mutex | 保护订阅者列表 |
| MqttInterface | atomic | 简单标志位 |

### 8.2 线程安全级别

| 操作 | 线程安全 | 说明 |
|------|---------|------|
| ConfigStore::get | 是 | 共享锁 |
| ConfigStore::set | 是 | 独占锁 |
| ConfigClient::get/set | 是 | Socket 操作加锁 |

---

## 9. 依赖

| 依赖 | 说明 |
|------|------|
| mshm | 共享内存、eventfd |
| cJSON | JSON 解析 |
| mosquitto | MQTT 客户端库 |
| pthread | 多线程支持 |

---

## 10. 构建配置

### 10.1 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.15)
project(config_server CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/sdk/include
    ${MOSS_ROOT}/mshm/include
)

add_executable(config_server
    src/main.cpp
    src/config_server.cpp
    src/config_store.cpp
    src/domain_socket_server.cpp
    src/mqtt_interface.cpp
)

target_link_libraries(config_server
    shm
    mosquitto
    cjson
    pthread
)

add_library(config_client STATIC
    sdk/src/config_client.cpp
)

install(TARGETS config_server RUNTIME DESTINATION bin)
install(DIRECTORY include/ DESTINATION include/config_server)
install(DIRECTORY sdk/include/ DESTINATION include/config_client)
```

---

## 11. 使用示例

### 11.1 启动服务器

```bash
./config_server /etc/config_server.json
```

### 11.2 客户端使用

```cpp
#include "config_client/config_client.h"

int main() {
    auto client = config_client::ConfigClient::create();

    // 设置配置
    client->set("vehicle.speed_limit", 120);
    client->set("vehicle.name", std::string("Tesla"));
    client->set("feature.enabled", true);

    // 获取配置
    auto speed = client->get<int>("vehicle.speed_limit");
    if (speed && speed->valid) {
        printf("Speed limit: %d\n", speed->value);
    }

    // 异步设置
    auto future = client->async_set("vehicle.speed_limit", 150);
    bool result = future.get();

    return 0;
}
```

---

## 12. 后续扩展

1. **TLS 加密**: 支持云端通信加密
2. **权限控制**: 配置分组和访问权限
3. **审计日志**: 配置变更历史记录
4. **热加载**: 无需重启服务即可更新配置
