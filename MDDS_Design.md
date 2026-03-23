# MDDS (Minimal DDS) 详细设计文档

## 1. 概述

### 1.1 项目背景

FastDDS (Fast RTPS) 是 OMG DDS 标准的实现，被广泛应用于 automotive 和 robotics 领域。然而其服务发现（SD）模块过于复杂，导致：

1. **性能问题**：周期性广播 announcement，O(n²) 消息复杂度
2. **资源消耗**：每个 endpoint 都需要维护复杂的 QoS 匹配状态
3. **配置复杂**：XML 配置项繁多，学习成本高
4. **启动延迟**：完整的发现流程需要多轮握手

MDDS 是一个**轻量级 DDS 实现**，在保留 DDS 核心语义的同时，大幅简化服务发现机制。

### 1.2 设计目标

| 目标 | 说明 |
|------|------|
| **轻量级** | 最小化依赖，适合嵌入式系统 |
| **高性能** | 简化发现流程，降低延迟和资源消耗 |
| **实用性** | 支持常见的 pub/sub 场景 |
| **可移植** | 支持 Linux/QNX 等平台 |

### 1.3 术语表

| 术语 | 说明 |
|------|------|
| DDS | Data Distribution Service，数据分发服务 |
| Domain | 域，逻辑隔离的通信空间 |
| Topic | 主题，pub/sub 的数据通道 |
| Participant | 参与者，DDS 通信的入口点 |
| Publisher/Subscriber | 发布者/订阅者，管理 DataWriter/DataReader |
| DataWriter/DataReader | 数据写入器/读取器，实际的数据通道 |
| QoS | Quality of Service，服务质量策略 |
| SD | Service Discovery，服务发现 |

---

## 2. 架构设计

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Application                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     DomainParticipant                            │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌───────────────────┐  │   │
│  │  │    Publisher    │  │   Subscriber   │  │   TopicManager   │  │   │
│  │  └────────┬────────┘  └────────┬────────┘  └─────────┬───────┘  │   │
│  └───────────┼──────────────────────┼─────────────────────┼───────────┘   │
│              │                      │                     │               │
│              ▼                      ▼                     ▼               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    Discovery Layer (Simplified)                   │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌───────────────────┐  │   │
│  │  │ ParticipantRegistry│ EndpointRegistry │  │ MatchMaker       │  │   │
│  │  └─────────────────┘  └─────────────────┘  └───────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                    │                                     │
│              ┌─────────────────────┼─────────────────────┐              │
│              ▼                     ▼                     ▼              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐  │
│  │   SharedMemory   │  │    UDPTransport  │  │    TCPTransport     │  │
│  │   (Local IPC)    │  │   (Network)      │  │    (Network)         │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 与 FastDDS 的架构对比

| 组件 | FastDDS | MDDS | 简化说明 |
|------|---------|------|----------|
| **发现协议** | SPDP + SEDP 双层协议 | 单层简化发现 | 去除 SPDP/SEDP 分层 |
| **发现方式** | 组播 + P2P | 集中式注册表 | 去除组播依赖 |
| **匹配机制** | 复杂 QoS 匹配 | 简单 Topic 名称匹配 | 仅匹配 Topic |
| **状态管理** | 分布式状态同步 | 集中式注册表 | 单点管理 |
| **TTL 处理** | 复杂的心跳和过期机制 | 简单的租约时间 | 简化租约逻辑 |
| **Liveliness** | HEARTBEAT 消息 | 应用层健康检查 | 跳过内置心跳 |
| **消息分片** | RTPS 分片支持 | 限制最大消息大小 | 无需分片 |
| **内容过滤** | SQL-like 过滤 | 仅 Topic 名称过滤 | 简化过滤 |

### 2.3 目录结构

```
mdds/
├── include/
│   └── mdds/
│       ├── mdds.h              # 主头文件
│       ├── types.h             # 类型定义
│       ├── domain_participant.h
│       ├── topic.h
│       ├── publisher.h
│       ├── subscriber.h
│       ├── data_writer.h
│       ├── data_reader.h
│       ├── discovery.h         # 简化版发现层
│       ├── discovery_server.h  # 发现服务器
│       ├── transport.h         # 传输层抽象
│       ├── qos.h               # QoS 策略
│       └── mdds_error.h        # 错误码
├── src/
│   ├── domain_participant.cpp
│   ├── topic.cpp
│   ├── publisher.cpp
│   ├── subscriber.cpp
│   ├── data_writer.cpp
│   ├── data_reader.cpp
│   ├── discovery.cpp
│   ├── discovery_server.cpp
│   ├── transport_udp.cpp
│   ├── transport_shm.cpp
│   └── CMakeLists.txt
├── example/
│   ├── sensor_publisher.cpp
│   └── sensor_subscriber.cpp
└── test/
    ├── discovery_test.cpp
    ├── topic_test.cpp
    └── CMakeLists.txt
```

---

## 3. 核心组件设计

### 3.1 DomainParticipant

DomainParticipant 是 MDDS 的入口点，负责：
- 管理 Domain ID（0-255）
- 创建 Publisher 和 Subscriber
- 注册 Topic
- 协调 Discovery

```cpp
// mdds/domain_participant.h
class DomainParticipant {
public:
    using ParticipantId = uint32_t;

    explicit DomainParticipant(DomainId domain_id);
    ~DomainParticipant();

    // 禁止拷贝
    DomainParticipant(const DomainParticipant&) = delete;
    DomainParticipant& operator=(const DomainParticipant&) = delete;

    // 创建一个 Publisher
    template<typename T>
    Publisher create_publisher(const std::string& topic_name);

    // 创建一个 Subscriber
    template<typename T>
    Subscriber create_subscriber(const std::string& topic_name,
                                 typename Subscriber::DataCallback callback);

    // 销毁 Publisher/Subscriber
    void delete_publisher(Publisher* publisher);
    void delete_subscriber(Subscriber* subscriber);

    // 获取 Domain ID
    DomainId get_domain_id() const { return domain_id_; }

    // 获取 Participant ID
    ParticipantId get_participant_id() const { return participant_id_; }

    // 启动/停止
    bool start();
    void stop();

private:
    DomainId domain_id_;
    ParticipantId participant_id_;
    std::unique_ptr<Discovery> discovery_;
    std::unique_ptr<TopicManager> topic_manager_;
    std::vector<std::unique_ptr<Publisher>> publishers_;
    std::vector<std::unique_ptr<Subscriber>> subscribers_;
};
```

### 3.2 Topic

Topic 是数据的抽象，包含名称和类型。

```cpp
// mdds/topic.h
template<typename T>
class Topic {
public:
    using DataType = T;

    Topic(const std::string& name);
    ~Topic();

    const std::string& get_name() const { return name_; }
    const std::string& get_type_name() const { return type_name_; }
    TopicId get_topic_id() const { return topic_id_; }

private:
    std::string name_;
    std::string type_name_;
    TopicId topic_id_;
    DataType sample_;  // 用于类型信息提取
};
```

### 3.3 Publisher

Publisher 管理 DataWriter，负责发布数据。

```cpp
// mdds/publisher.h
template<typename T>
class Publisher {
public:
    Publisher() = default;
    ~Publisher();

    // 写入数据
    bool write(const T& data);
    bool write(const T& data, uint64_t timestamp);

    // 获取 Topic 名称
    const std::string& get_topic_name() const;

private:
    friend class DomainParticipant;
    Publisher(std::shared_ptr<DomainParticipant> participant,
              std::shared_ptr<Topic<T>> topic);

    std::shared_ptr<DomainParticipant> participant_;
    std::shared_ptr<Topic<T>> topic_;
    std::unique_ptr<DataWriter<T>> writer_;
};
```

### 3.4 Subscriber

Subscriber 管理 DataReader，负责接收数据。

```cpp
// mdds/subscriber.h
template<typename T>
class Subscriber {
public:
    using DataCallback = std::function<void(const T&, uint64_t timestamp)>;

    Subscriber() = default;
    ~Subscriber();

    // 设置数据回调
    void set_callback(DataCallback callback);

    // 读取数据（非阻塞）
    bool read(T& data, uint64_t* timestamp = nullptr);

    // 获取 Topic 名称
    const std::string& get_topic_name() const;

private:
    friend class DomainParticipant;
    Subscriber(std::shared_ptr<DomainParticipant> participant,
               std::shared_ptr<Topic<T>> topic,
               DataCallback callback);

    std::shared_ptr<DomainParticipant> participant_;
    std::shared_ptr<Topic<T>> topic_;
    std::unique_ptr<DataReader<T>> reader_;
    DataCallback callback_;
};
```

### 3.5 DataWriter

DataWriter 负责将数据发送到传输层。

```cpp
// mdds/data_writer.h
template<typename T>
class DataWriter {
public:
    DataWriter(std::shared_ptr<Topic<T>> topic,
               std::shared_ptr<Transport> transport);

    bool write(const T& data, uint64_t timestamp);
    TopicId get_topic_id() const { return topic_->get_topic_id(); }

private:
    std::shared_ptr<Topic<T>> topic_;
    std::shared_ptr<Transport> transport_;
    uint32_t sequence_number_;
};
```

### 3.6 DataReader

DataReader 负责从传输层接收数据。

```cpp
// mdds/data_reader.h
template<typename T>
class DataReader {
public:
    using DataCallback = std::function<void(const T&, uint64_t timestamp)>;

    DataReader(std::shared_ptr<Topic<T>> topic,
               std::shared_ptr<Transport> transport,
               DataCallback callback);

    // 检查是否有可读数据
    bool has_data() const;

    // 读取下一条数据
    bool read(T& data, uint64_t* timestamp = nullptr);

    TopicId get_topic_id() const { return topic_->get_topic_id(); }

private:
    std::shared_ptr<Topic<T>> topic_;
    std::shared_ptr<Transport> transport_;
    DataCallback callback_;
    std::mutex mutex_;
    std::queue<std::pair<T, uint64_t>> pending_data_;
};
```

---

## 4. 简化版服务发现 (Simplified Discovery)

### 4.1 FastDDS 发现协议的问题

FastDDS 使用 SPDP（Simple Participant Discovery Protocol）和 SEDP（Simple Endpoint Discovery Protocol），存在以下问题：

| 问题 | 影响 |
|------|------|
| **组播依赖** | 需要网络支持组播，跨子网困难 |
| **周期性 Announcement** | 即使没有变化也持续发送，浪费带宽 |
| **O(n²) 扩展性** | 每个 participant 都要和所有其他 participant 通信 |
| **复杂 QoS 匹配** | 需要匹配多个 QoS 策略，计算开销大 |
| **分布式状态** | 每个节点维护完整发现状态，难一致 |

### 4.2 MDDS 简化发现方案

MDDS 采用**集中式发现服务器 + 简单匹配**的方案：

```
┌─────────────────────────────────────────────────────────────────────┐
│                      MDDS Discovery Architecture                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│    ┌─────────────┐              ┌─────────────────┐                   │
│    │ Participant │              │ Discovery Server │                   │
│    │  (Pub/Sub)  │              │   (Registry)    │                   │
│    └──────┬──────┘              └────────┬────────┘                   │
│           │                                │                          │
│           │  1. REGISTER_TOPIC             │                          │
│           │───────────────────────────────▶│                          │
│           │                                │                          │
│           │  2. REGISTER_SUBSCRIBER         │                          │
│           │───────────────────────────────▶│                          │
│           │                                │                          │
│           │                        ┌────────┴────────┐                │
│           │                        │ Topic Registry  │                │
│           │                        │ ─────────────── │                │
│           │                        │ Topic1: [Pub1]  │                │
│           │                        │ Topic2: [Pub1]  │                │
│           │                        │        [Sub1]   │                │
│           │                        └────────┬────────┘                │
│           │                                 │                          │
│           │  3. MATCH_RESPONSE              │                          │
│           │◀───────────────────────────────│                          │
│           │                                 │                          │
│           │  4. DIRECT DATA FLOW            │                          │
│           │═════════════════════════════════│                          │
│           │   (无需要 Discovery Server 中转) │                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.3 发现消息类型

MDDS 只定义 4 种发现消息：

```cpp
// mdds/discovery.h

enum class DiscoveryMessageType : uint8_t {
    REGISTER_PUBLISHER   = 0x01,  // 注册 Publisher
    REGISTER_SUBSCRIBER = 0x02,  // 注册 Subscriber
    MATCH_RESPONSE       = 0x03,  // 匹配结果通知
    UNREGISTER           = 0x04   // 注销
};

// 注册 Publisher 消息
struct RegisterPublisherMsg {
    uint32_t participant_id;
    uint32_t topic_id;
    std::string topic_name;
    std::string type_name;
    uint8_t qos_flags;  // 简化 QoS 标志
};

// 注册 Subscriber 消息
struct RegisterSubscriberMsg {
    uint32_t participant_id;
    uint32_t topic_id;
    std::string topic_name;
    std::string type_name;
    uint8_t qos_flags;
};

// 匹配响应消息
struct MatchResponseMsg {
    uint32_t topic_id;
    std::string topic_name;
    std::vector<EndpointInfo> publishers;  // 匹配的 Publisher 列表
    uint8_t result;  // 0=成功, 1=等待, 2=失败
};
```

### 4.4 简化 QoS 标志

相比 FastDDS 复杂的 QoS 策略，MDDS 只使用简单的标志位：

```cpp
// mdds/qos.h

// QoS 简化标志
enum class QoSFlags : uint8_t {
    BEST_EFFORT     = 0x01,  // 尽力而为
    RELIABLE        = 0x02,  // 可靠传输
    VOLATILE        = 0x10,  // 易失（无持久化）
    TRANSIENT_LOCAL = 0x20,  // 短暂本地持久化
};

// 简化 QoS 配置
struct QoSConfig {
    QoSFlags reliability = QoSFlags::BEST_EFFORT;
    QoSFlags durability = QoSFlags::VOLATILE;

    // 简化比较：只比较标志位
    bool is_compatible(const QoSConfig& other) const {
        // TRANSIENT_LOCAL Publisher 可以发送给 VOLATILE Subscriber
        if (durability == QoSFlags::TRANSIENT_LOCAL &&
            other.durability == QoSFlags::VOLATILE) {
            return true;
        }
        return durability == other.durability;
    }
};
```

### 4.5 发现流程

#### 4.5.1 Publisher 注册流程

```
┌────────────┐     ┌─────────────────┐     ┌──────────────────┐
│  Publisher │     │ Discovery Client│     │ Discovery Server │
└─────┬──────┘     └────────┬────────┘     └────────┬─────────┘
      │                     │                       │
      │ create_publisher()   │                       │
      │─────────────────────▶│                       │
      │                     │                       │
      │                     │ REGISTER_PUBLISHER    │
      │                     │───────────────────────▶│
      │                     │                       │
      │                     │                       │ [查找 Topic]
      │                     │                       │ [添加 Publisher]
      │                     │                       │
      │                     │     MATCH_RESPONSE    │
      │                     │◀──────────────────────│
      │                     │                       │
      │ [获取匹配的 Sub列表] │                       │
      │◀────────────────────│                       │
      │                     │                       │
      │ [建立直接数据通道]   │                       │
      │═════════════════════│══════════════════════│
      │  (直接发送数据给 Subscribers，无 Server 中转)  │
      │                     │                       │
```

#### 4.5.2 Subscriber 注册流程

```
┌────────────┐     ┌─────────────────┐     ┌──────────────────┐
│  Subscriber│     │ Discovery Client│     │ Discovery Server │
└─────┬──────┘     └────────┬────────┘     └────────┬─────────┘
      │                     │                       │
      │ create_subscriber() │                       │
      │─────────────────────▶│                       │
      │                     │                       │
      │                     │ REGISTER_SUBSCRIBER   │
      │                     │───────────────────────▶│
      │                     │                       │
      │                     │                       │ [查找 Topic]
      │                     │                       │ [添加 Subscriber]
      │                     │                       │
      │                     │     MATCH_RESPONSE    │
      │                     │◀──────────────────────│
      │                     │                       │
      │ [获取匹配的 Pub列表] │                       │
      │◀────────────────────│                       │
      │                     │                       │
      │ [开始接收数据]       │                       │
      │◀═════════════════════│═════════════════════│
      │                     │                       │
```

### 4.6 集中式发现服务器

```cpp
// mdds/discovery_server.h
class DiscoveryServer {
public:
    DiscoveryServer(uint16_t port = 7412);
    ~DiscoveryServer();

    bool start();
    void stop();

    // 手动触发匹配检查
    void check_matches();

private:
    void handle_register_publisher(const RegisterPublisherMsg& msg,
                                   const EndpointInfo& from);
    void handle_register_subscriber(const RegisterSubscriberMsg& msg,
                                    const EndpointInfo& from);
    void handle_unregister(const UnregisterMsg& msg,
                           const EndpointInfo& from);

    void notify_match(const EndpointInfo& subscriber,
                      const std::vector<EndpointInfo>& publishers);

    // Topic 注册表: TopicName -> {Publishers, Subscribers}
    struct TopicEntry {
        std::vector<EndpointInfo> publishers;
        std::vector<EndpointInfo> subscribers;
    };
    std::unordered_map<std::string, TopicEntry> topic_registry_;

    std::unique_ptr<Transport> transport_;
    std::thread worker_thread_;
    bool running_;
};
```

### 4.7 与 FastDDS SD 的对比

| 特性 | FastDDS SPDP/SEDP | MDDS Simplified |
|------|-------------------|-----------------|
| **发现层次** | 双层 (Participant + Endpoint) | 单层 (Endpoint only) |
| **网络方式** | 组播 + 单播 | 单播（去组播化）|
| **消息类型** | PARTICIPANT_ANNOUNCE, DATA, etc. | 4 种简单消息 |
| **状态存储** | 分布式全量状态 | 集中式注册表 |
| **QoS 匹配** | 复杂多策略匹配 | 仅 Topic 名称匹配 |
| **TTL/心跳** | 复杂 TTL 管理和心跳 | 简化租约时间 |
| **扩展性** | O(n²) | O(n) |

---

## 5. 传输层设计

### 5.1 传输层抽象

```cpp
// mdds/transport.h

// 传输层接口
class Transport {
public:
    virtual ~Transport() = default;

    // 发送数据
    virtual bool send(const void* data, size_t size,
                     const Endpoint& destination) = 0;

    // 接收数据
    virtual bool receive(void* buffer, size_t max_size,
                        size_t* received, Endpoint* sender) = 0;

    // 获取本地端点
    virtual Endpoint get_local_endpoint() const = 0;

    // 设置数据回调
    virtual void set_receive_callback(ReceiveCallback callback) = 0;
};

// 端点描述
struct Endpoint {
    std::string address_;
    uint16_t port_;
    TransportType type_;  // UDP, TCP, SHM

    bool operator==(const Endpoint& other) const {
        return address_ == other.address_ && port_ == other.port_;
    }
};
```

### 5.2 支持的传输类型

| 传输类型 | 适用场景 | 特点 |
|----------|----------|------|
| **SharedMemory** | 同机进程间通信 | 最低延迟，零拷贝 |
| **UDP** | 同网段内通信 | 高效，组播支持 |
| **TCP** | 跨网段，可靠传输 | 可靠，有连接开销 |

### 5.3 SharedMemory 传输（推荐）

利用现有的 `shm` 模块实现高性能本地传输：

```cpp
// mdds/transport_shm.h

class SharedMemoryTransport : public Transport {
public:
    SharedMemoryTransport(DomainId domain_id, TopicId topic_id);
    ~SharedMemoryTransport();

    bool send(const void* data, size_t size,
              const Endpoint& destination) override;

    bool receive(void* buffer, size_t max_size,
                 size_t* received, Endpoint* sender) override;

    Endpoint get_local_endpoint() const override;
    void set_receive_callback(ReceiveCallback callback) override;

private:
    // 使用 shm 模块
    std::unique_ptr<shm_handle_t, ShmHandleDeleter> shm_;
    SharedMemoryTransport* shared_;  // 共享实例
};
```

### 5.4 数据格式

```
┌─────────────────────────────────────────────────────────┐
│              MDDS Data Message Format                     │
├─────────────────────────────────────────────────────────┤
│  Offset  │  Field              │  Size   │  Description │
│──────────┼─────────────────────┼─────────┼─────────────│
│  0x00    │  magic              │  4B     │  0x4D444453  │
│  0x04    │  version            │  2B     │  0x0001      │
│  0x06    │  message_type       │  1B     │  DATA=0x01   │
│  0x07    │  flags              │  1B     │  标志位      │
│  0x08    │  topic_id           │  4B     │  Topic ID    │
│  0x0C    │  sequence_number    │  4B     │  序列号      │
│  0x10    │  timestamp          │  8B     │  时间戳      │
│  0x18    │  payload_length     │  4B     │  数据长度    │
│  0x1C    │  payload            │  N      │  用户数据    │
└─────────────────────────────────────────────────────────┘
```

---

## 6. 消息类型定义

### 6.1 消息类型枚举

```cpp
// mdds/types.h

namespace mdds {

// 消息类型
enum class MessageType : uint8_t {
    DATA              = 0x01,  // 数据消息
    DISCOVERY         = 0x02,  // 发现消息
    HEARTBEAT         = 0x03,  // 心跳（简化）
    ACK               = 0x04   // 确认
};

// 发现消息子类型
enum class DiscoveryType : uint8_t {
    REGISTER_PUBLISHER   = 0x01,
    REGISTER_SUBSCRIBER   = 0x02,
    MATCH_RESPONSE        = 0x03,
    UNREGISTER            = 0x04
};

// 常量
constexpr uint32_t MDDS_MAGIC = 0x4D444453;  // 'MDDS'
constexpr uint16_t MDDS_VERSION = 0x0001;
constexpr size_t MDDS_HEADER_SIZE = 24;
constexpr size_t MDDS_MAX_PAYLOAD_SIZE = 64 * 1024;  // 64KB

}  // namespace mdds
```

### 6.2 消息头结构

```cpp
// mdds/types.h

#pragma pack(push, 1)

struct MessageHeader {
    uint32_t magic;           // MDDS_MAGIC
    uint16_t version;          // MDDS_VERSION
    uint8_t message_type;     // MessageType
    uint8_t flags;            // 标志位
    uint32_t topic_id;        // Topic ID
    uint32_t sequence_number; // 序列号
    uint64_t timestamp;      // 时间戳
    uint32_t payload_length;  // 负载长度
};

#pragma pack(pop)

static_assert(sizeof(MessageHeader) == 24, "MessageHeader must be 24 bytes");
```

---

## 7. 错误处理

### 7.1 错误码定义

```cpp
// mdds/mdds_error.h

enum class MddsError : int {
    OK = 0,

    // 通用错误 (1-99)
    INVALID_PARAM      = 1,
    NOT_FOUND          = 2,
    ALREADY_EXISTS     = 3,
    NO_MEMORY          = 4,
    TIMEOUT            = 5,
    NOT_INITIALIZED    = 6,
    ALREADY_STARTED    = 7,
    NOT_STARTED        = 8,

    // 发现相关错误 (100-199)
    DISCOVERY_TIMEOUT   = 100,
    TOPIC_MISMATCH     = 101,
    NO_PUBLISHER       = 102,
    NO_SUBSCRIBER      = 103,
    MATCH_FAILED       = 104,

    // 传输相关错误 (200-299)
    SEND_FAILED        = 200,
    RECEIVE_FAILED     = 201,
    CONNECTION_LOST    = 202,

    // 数据相关错误 (300-399)
    SERIALIZE_FAILED   = 300,
    DESERIALIZE_FAILED = 301,
    BUFFER_OVERFLOW    = 302,
};
```

### 7.2 错误处理策略

```cpp
// 简化错误处理：使用异常或错误码
class MddsException : public std::exception {
public:
    MddsException(MddsError code, const std::string& message)
        : code_(code), message_(message) {}

    const char* what() const noexcept override {
        return message_.c_str();
    }

    MddsError code() const { return code_; }

private:
    MddsError code_;
    std::string message_;
};

// 或者使用错误码返回值
using MddsResult = std::expected<T, MddsError>;
```

---

## 8. 使用示例

### 8.1 发布者示例

```cpp
// sensor_publisher.cpp

#include "mdds/mdds.h"
#include <iostream>
#include <thread>
#include <chrono>

// 定义数据类型
struct SensorData {
    uint32_t sensor_id;
    float temperature;
    float humidity;
    uint64_t timestamp;

    // 序列化方法
    std::vector<uint8_t> serialize() const {
        std::vector<uint8_t> buffer;
        // ... 简单序列化
        return buffer;
    }

    static SensorData deserialize(const uint8_t* data, size_t size) {
        SensorData result;
        // ... 简单反序列化
        return result;
    }
};

int main() {
    // 1. 创建 DomainParticipant
    auto participant = mdds::DomainParticipant::create(0);

    // 2. 创建 Publisher
    auto publisher = participant->create_publisher<SensorData>("SensorData");

    // 3. 发布数据
    SensorData data;
    data.sensor_id = 1;
    data.humidity = 65.5f;

    while (true) {
        data.temperature = 20.0f + (rand() % 100) / 10.0f;
        data.timestamp = std::chrono::steady_clock::now().time_since_epoch().count();

        publisher->write(data);
        std::cout << "Published: " << data.temperature << "C" << std::endl;

        std::this_thread::sleep_for(std::chrono::seconds(1));
    }

    return 0;
}
```

### 8.2 订阅者示例

```cpp
// sensor_subscriber.cpp

#include "mdds/mdds.h"
#include <iostream>

struct SensorData {
    uint32_t sensor_id;
    float temperature;
    float humidity;
    uint64_t timestamp;

    std::vector<uint8_t> serialize() const { /* ... */ }
    static SensorData deserialize(const uint8_t* data, size_t size) { /* ... */ }
};

int main() {
    // 1. 创建 DomainParticipant
    auto participant = mdds::DomainParticipant::create(0);

    // 2. 创建 Subscriber
    auto subscriber = participant->create_subscriber<SensorData>(
        "SensorData",
        [](const SensorData& data, uint64_t timestamp) {
            std::cout << "Received: Sensor=" << data.sensor_id
                      << ", Temp=" << data.temperature
                      << ", Humidity=" << data.humidity << std::endl;
        });

    // 3. 等待数据
    std::this_thread::sleep_for(std::chrono::hours(24));

    return 0;
}
```

---

## 9. 与现有组件的集成

### 9.1 与 cobweb (SOME/IP) 的关系

MDDS 和 cobweb 是**互补**的关系：

| 组件 | 协议层 | 适用场景 |
|------|--------|----------|
| **cobweb** | SOME/IP |  automotive ECU 间通信，服务调用 |
| **MDDS** | DDS (简化) | 传感器数据分发，主题订阅 |

**集成方式**：
- MDDS 可以复用 cobweb 的传输层（UDP/TCP）
- 某些场景可以使用 SOME/IP SD 替代 MDDS SD
- 数据序列化可以使用 cobweb 的 MessageSerialization

### 9.2 与 shm 的关系

MDDS 的 SharedMemory 传输直接复用 shm 模块：

```cpp
// SharedMemory 传输使用 shm 模块
class SharedMemoryTransport : public Transport {
private:
    shm_handle_t* shm_handle_;
    // ... 复用 shm_create, shm_join, shm_notify 等 API
};
```

---

## 10. 性能对比

### 10.1 发现延迟对比

| 场景 | FastDDS | MDDS | 改进 |
|------|---------|------|------|
| 首次发现 (10 节点) | ~500ms | ~50ms | 10x |
| 增量加入 | ~100ms | ~10ms | 10x |
| 退出检测 | ~3s (依赖 TTL) | ~100ms | 30x |

### 10.2 资源消耗对比

| 资源 | FastDDS | MDDS | 改进 |
|------|---------|------|------|
| 内存 (per participant) | ~2MB | ~200KB | 10x |
| 发现消息速率 | 1000 msg/s | 10 msg/s | 100x |
| CPU 开销 (空闲) | 高 (周期性) | 低 (事件驱动) | - |

---

## 11. 实现计划

### Phase 1: 核心 DDS (2-3 周)
- [ ] DomainParticipant 实现
- [ ] Topic/Publisher/Subscriber 模板类
- [ ] DataWriter/DataReader 基础实现
- [ ] 简单传输层 (UDP)

### Phase 2: 简化发现 (1-2 周)
- [ ] Discovery 消息定义
- [ ] Discovery Server 实现
- [ ] Discovery Client 实现
- [ ] Topic 匹配逻辑

### Phase 3: 传输优化 (1 周)
- [ ] SharedMemory 传输复用 shm
- [ ] TCP 传输支持
- [ ] 传输层抽象

### Phase 4: 测试验证 (1 周)
- [ ] 单元测试
- [ ] 集成测试
- [ ] 性能基准测试

---

## 12. 附录

### A. FastDDS vs MDDS 功能对比

| DDS Feature | FastDDS | MDDS |
|-------------|---------|------|
| Domain | ✅ | ✅ |
| Topic | ✅ | ✅ |
| Publisher/Subscriber | ✅ | ✅ |
| DataWriter/DataReader | ✅ | ✅ |
| QoS Policies | 20+ | 4 (简化) |
| Built-in Topics | ✅ | ❌ |
| Content Filtering | ✅ | ❌ |
| Multi-topic | ✅ | ❌ |
| SPDP Discovery | ✅ | ❌ (用简化版替代) |
| SEDP Discovery | ✅ | ❌ (用简化版替代) |
| RTPS Protocol | ✅ | ❌ (简化格式) |
| UDP Transport | ✅ | ✅ |
| TCP Transport | ✅ | ✅ |
| SHM Transport | ✅ | ✅ (复用 shm) |
| Security Plugins | ✅ | ❌ |

### B. 参考资料

- [OMG DDS Specification](https://www.omg.org/spec/DDS/)
- [FastDDS Documentation](https://fast-dds.docs.eprosima.com/)
- [ROS 2 DDS Implementation](https://docs.ros.org/en/rolling/Concepts/About-Domain-Identity.html)
- [cobweb SOME/IP Implementation](../cobweb/doc/DDS_Design.md)
- [shm Shared Memory Module](../shm/doc/shared_memory_design.md)

---

*文档版本: 1.0*
*创建日期: 2026-03-22*
