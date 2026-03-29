# MParam 模块设计文档

## 1. 概述

MParam 是 MOSS 项目的参数管理核心组件，提供统一参数存储、查询和变更通知机制，专为汽车/嵌入式系统优化。

### 1.1 设计目标

- **类型安全**: 模板化参数接口，支持 int/double/bool/string 等基本类型
- **层次化键空间**: 支持命名空间，如 `vehicle.speed_limit`
- **变更通知**: 观察者模式，参数值变化时主动通知订阅者
- **跨进程通信**: 基于 mshm 共享内存，高效进程间参数共享
- **持久化支持**: JSON 文件存储，支持启动时加载

### 1.2 术语表

| 术语 | 说明 |
|------|------|
| Parameter | 参数，键值对 + 元数据 |
| ParameterValue | 参数值，支持多种类型 |
| ParameterServer | 参数服务器，管理所有参数 |
| ParameterClient | 参数客户端，SDK 接口 |
| ParameterNamespace | 参数命名空间，层次化键管理 |
| WatchCallback | 参数变更回调函数 |

---

## 2. 架构设计

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                            mparam                                    │
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │ ParameterServer  │  │  ShmTransport    │  │  FilePersistence │   │
│  │  (核心服务器)     │  │  (共享内存传输)   │  │  (JSON持久化)    │   │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘   │
│           └──────────────────────┼──────────────────────┘            │
│                                  │                                   │
│  ┌──────────────────────────────┼──────────────────────────────┐   │
│  │                   ParameterStore                            │   │
│  │  "vehicle.speed_limit" → {int, 120, v=5}                    │   │
│  │  "sensors.lidar.enable" → {bool, true, v=2}               │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
        ┌──────────┐        ┌──────────┐        ┌──────────┐
        │ Node A   │        │ Node B   │        │ Node C   │
        └──────────┘        └──────────┘        └──────────┘
```

### 2.2 目录结构

```
mparam/
├── include/mparam/
│   ├── mparam.h              # 主包含头文件
│   ├── param_types.h         # 类型定义
│   ├── param_store.h         # 参数存储核心
│   ├── param_server.h        # 参数服务器
│   ├── param_client.h        # 客户端 SDK
│   └── param_traits.h        # 类型萃取
├── src/
│   ├── param_server.cpp
│   ├── param_store.cpp
│   └── shm_transport.cpp
└── CMakeLists.txt
```

---

## 3. 核心设计

### 3.1 类型定义

```cpp
union ParamValue {
    int64_t as_int;
    double as_double;
    bool as_bool;
    std::string* as_string;
};

enum class ParamType : uint8_t {
    NONE = 0, INT = 1, DOUBLE = 2, BOOL = 3, STRING = 4, JSON = 5
};

struct Parameter {
    std::string key;
    ParamValue value;
    ParamType type;
    int64_t version;
    int64_t timestamp;
    uint32_t flags;
};

struct ParameterChange {
    std::string key;
    Parameter old_value;
    Parameter new_value;
    int64_t change_time;
};
```

### 3.2 参数存储

```cpp
class ParameterStore {
public:
    bool has(const std::string& key) const;
    std::optional<Parameter> get(const std::string& key) const;
    bool set(const std::string& key, const Parameter& param);
    bool remove(const std::string& key);

    std::vector<Parameter> get_all() const;
    std::vector<std::string> get_namespaces() const;

    using ChangeCallback = std::function<void(const ParameterChange&)>;
    size_t add_watcher(const std::string& key, ChangeCallback callback);
    size_t add_global_watcher(ChangeCallback callback);
};
```

### 3.3 参数服务器

```cpp
class ParameterServer : public std::enable_shared_from_this<ParameterServer> {
public:
    static std::shared_ptr<ParameterServer> create(
        const std::string& shm_name = "/moss_param");

    template<typename T>
    std::optional<TypedParameter<T>> get(const std::string& key);

    template<typename T>
    bool set(const std::string& key, const T& value);

    template<typename T>
    bool declare(const std::string& key, const T& default_value,
                 const std::string& description = "");

    size_t watch(const std::string& key,
                 std::function<void(const ParameterChange&)> callback);

    bool load_from_file(const std::string& path);
    bool save_to_file(const std::string& path) const;
};
```

### 3.4 客户端 SDK

```cpp
class ParameterClient : public std::enable_shared_from_this<ParameterClient> {
public:
    static std::shared_ptr<ParameterClient> create(
        const std::string& shm_name = "/moss_param");

    template<typename T>
    std::optional<TypedParameter<T>> get(const std::string& key);

    template<typename T>
    bool set(const std::string& key, const T& value);

    template<typename T>
    std::future<std::optional<TypedParameter<T>>> async_get(
        const std::string& key);

    template<typename T>
    size_t watch(const std::string& key, ChangeCallback<T> callback);

    bool is_connected() const;
};
```

---

## 4. 使用示例

### 4.1 参数服务器

```cpp
int main() {
    auto server = mparam::ParameterServer::create("/moss_param");
    server->init();

    server->declare("vehicle.speed_limit", 120, "Max speed in km/h");
    server->declare("sensors.lidar.enable", true, "Enable LiDAR");

    server->set("vehicle.speed_limit", 100);

    server->watch("vehicle.speed_limit",
        [](const mparam::ParameterChange& change) {
            std::cout << "speed_limit changed\n";
        });

    server->save_to_file("/etc/moss/params.json");
    return 0;
}
```

### 4.2 参数客户端

```cpp
int main() {
    auto client = mparam::ParameterClient::create("/moss_param");
    client->init();

    auto speed = client->get<int64_t>("vehicle.speed_limit");
    if (speed && speed->valid) {
        std::cout << "Speed limit: " << speed->value << "\n";
    }

    client->set("vehicle.speed_limit", static_cast<int64_t>(150));

    client->watch<int64_t>("vehicle.speed_limit",
        [](const std::string& key, auto old_val, auto new_val) {
            std::cout << key << " changed: " << old_val.value
                      << " -> " << new_val.value << "\n";
        });

    return 0;
}
```

---

## 5. JSON 持久化格式

```json
{
    "mparam_version": 1,
    "parameters": [
        {
            "key": "vehicle.speed_limit",
            "type": "int",
            "value": 120,
            "version": 5,
            "description": "Maximum vehicle speed"
        },
        {
            "key": "sensors.lidar.enable",
            "type": "bool",
            "value": true,
            "version": 2
        }
    ]
}
```

---

## 6. 依赖关系

```
mparam
├── mshm (共享内存通信)
├── mlog (可选，日志输出)
└── cJSON (JSON 解析)
```

---

## 7. 与 config_server 的区别

| 特性 | mparam | config_server |
|------|--------|---------------|
| 定位 | 运行时参数 | 配置持久化 |
| 访问频率 | 高频 | 低频 |
| 变更通知 | 原生支持 | 云端同步 |
| 持久化 | JSON 文件 | JSON + MQTT |

---

## 8. 构建配置

```bash
cd mparam && mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j4
```
