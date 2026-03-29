# Node 模块设计文档

## 1. 概述

Node 模块是 mruntime 的核心组件，借鉴 ROS 节点思想，封装 mdds DDS 细节，提供简洁的发布/订阅 API 和生命周期管理。

### 1.1 设计目标

- **生命周期管理**: init() → start() → stop() → destroy() 状态机
- **封装 DDS 复杂性**: 用户无需直接操作 DomainParticipant
- **简洁的发布/订阅 API**: create_publisher<T>() / create_subscriber<T>()
- **线程安全**: 使用 mutex 保护共享状态

### 1.2 术语表

| 术语 | 说明 |
|------|------|
| Node | 节点主类，管理生命周期和端点 |
| NodeState | 节点状态枚举 |
| Publisher<T> | 模板发布者类 |
| Subscriber<T> | 模板订阅者类 |
| DomainParticipant | mdds 底层的域参与者 |

---

## 2. 架构设计

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                              mruntime                                │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                         Node                                  │  │
│  │                    (节点主类)                                 │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │  生命周期状态机                                          │  │  │
│  │  │  UNINITIALIZED → INITIALIZED → RUNNING → STOPPED       │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  │                           │                                     │  │
│  │                           ▼                                     │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │       mdds::DomainParticipant (底层 DDS 实现)            │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  │                           │                                     │  │
│  │         ┌─────────────────┼─────────────────┐               │  │
│  │         ▼                 ▼                 ▼               │  │
│  │  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐        │  │
│  │  │Publisher<T> │   │Subscriber<T>│   │   Clock     │        │  │
│  │  │ (发布者)    │   │ (订阅者)    │   │  (时钟)     │        │  │
│  │  └─────────────┘   └─────────────┘   └─────────────┘        │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 目录结构

```
mruntime/
├── include/mruntime/
│   ├── node.h              # Node 主类（含模板实现）
│   ├── node_state.h        # State 枚举和异常类
│   ├── node_config.h       # Config 配置结构体
│   ├── mruntime.h          # Runtime 主类
│   ├── clock.h             # 时钟接口
│   └── types.h             # 类型定义
├── src/
│   ├── mruntime.cpp         # Runtime 实现
│   ├── node.cpp            # Node 实现
│   ├── process.cpp         # 进程管理
│   └── clock.cpp           # 时钟实现
└── CMakeLists.txt
```

---

## 3. 生命周期状态机

### 3.1 状态定义

```cpp
enum class NodeState {
    UNINITIALIZED = 0,  // 初始状态
    INITIALIZED   = 1,  // 已初始化
    RUNNING       = 2,  // 运行中
    STOPPED       = 3,  // 已停止
    DESTROYED     = 4   // 已销毁
};
```

### 3.2 状态转换图

```
                    ┌─────────────────┐
                    │   UNINITIALIZED  │
                    └────────┬─────────┘
                             │ init()
                             ▼
                    ┌─────────────────┐
         ┌─────────│   INITIALIZED    │─────────┐
         │         └────────┬─────────┘         │
         │                  │ start()           │ destroy()
         │                  ▼                   │
         │         ┌─────────────────┐         │
         │         │     RUNNING     │─────────┘
         │         └────────┬─────────┘
         │                  │ stop()
         │                  ▼
         │         ┌─────────────────┐
         │         │     STOPPED    │
         │         └────────┬─────────┘
         │                  │ destroy()
         │                  ▼
         │         ┌─────────────────┐
         └────────▶│    DESTROYED    │
                   └─────────────────┘
```

### 3.3 状态转换规则

| 当前状态 | 操作 | 下一状态 | 说明 |
|---------|------|---------|------|
| UNINITIALIZED | init() | INITIALIZED | 创建 DomainParticipant |
| INITIALIZED | start() | RUNNING | 启动参与者 |
| INITIALIZED | destroy() | DESTROYED | 直接销毁 |
| RUNNING | stop() | STOPPED | 停止参与者 |
| STOPPED | destroy() | DESTROYED | 销毁节点 |
| RUNNING | destroy() | STOPPED→DESTROYED | 先停止再销毁 |
| DESTROYED | 任何操作 | - | 无效操作 |

---

## 4. 核心组件设计

### 4.1 NodeConfig 结构体

```cpp
struct NodeConfig {
    std::string node_name;           // 节点名称
    uint8_t domain_id{0};           // DDS 域 ID
    bool enable_multicast_discovery{false};  // 是否启用多播发现
};
```

### 4.2 Node 主类

```cpp
class Node : public std::enable_shared_from_this<Node> {
public:
    // 工厂方法
    static std::shared_ptr<Node> create(const std::string& node_name,
                                        mdds::DomainId domain_id = 0);

    // 生命周期管理
    bool init();      // 初始化，创建 DomainParticipant
    bool start();     // 启动节点
    void stop();       // 停止节点
    void destroy();   // 销毁节点

    // 状态查询
    bool is_running() const;
    NodeState get_state() const;
    const std::string& get_name() const;
    mdds::DomainId get_domain_id() const;

    // 发布/订阅创建
    template<typename T>
    std::shared_ptr<Publisher<T>> create_publisher(
        const std::string& topic_name,
        const mdds::QoSConfig& qos = mdds::default_qos::publisher());

    template<typename T>
    std::shared_ptr<Subscriber<T>> create_subscriber(
        const std::string& topic_name,
        typename mdds::Subscriber<T>::DataCallback callback,
        const mdds::QoSConfig& qos = mdds::default_qos::subscriber());

private:
    NodeConfig config_;
    NodeState state_{NodeState::UNINITIALIZED};
    mutable std::mutex state_mutex_;
    std::shared_ptr<mdds::DomainParticipant> participant_;
    std::vector<PublisherEntry> publishers_;
    std::vector<SubscriberEntry> subscribers_;
    std::mutex endpoints_mutex_;
};
```

### 4.3 Publisher<T> 模板类

```cpp
template<typename T>
class Publisher {
public:
    Publisher() = default;

    Publisher(std::weak_ptr<Node> node,
              std::shared_ptr<mdds::Publisher<T>> publisher)
        : node_(std::move(node)), publisher_(std::move(publisher)) {}

    // 写入数据
    bool write(const T& data);
    bool write(const T& data, uint64_t timestamp);

    // 获取主题信息
    const std::string& get_topic_name() const;
    mdds::TopicId get_topic_id() const;

    // 获取底层发布者
    std::shared_ptr<mdds::Publisher<T>> get_mdds_publisher() const;

private:
    std::weak_ptr<Node> node_;
    std::weak_ptr<mdds::Publisher<T>> publisher_;
};
```

### 4.4 Subscriber<T> 模板类

```cpp
template<typename T>
class Subscriber {
public:
    Subscriber() = default;

    Subscriber(std::weak_ptr<Node> node,
               std::shared_ptr<mdds::Subscriber<T>> subscriber)
        : node_(std::move(node)), subscriber_(std::move(subscriber)) {}

    // 设置数据回调
    void set_callback(typename mdds::Subscriber<T>::DataCallback callback);

    // 读取数据（非阻塞）
    bool read(T& data, uint64_t* timestamp = nullptr);
    bool has_data() const;

    // 获取主题信息
    const std::string& get_topic_name() const;
    mdds::TopicId get_topic_id() const;

    // 获取底层订阅者
    std::shared_ptr<mdds::Subscriber<T>> get_mdds_subscriber() const;

private:
    std::weak_ptr<Node> node_;
    std::weak_ptr<mdds::Subscriber<T>> subscriber_;
};
```

### 4.5 异常类

```cpp
class NodeException : public std::runtime_error {
public:
    explicit NodeException(const std::string& message) : std::runtime_error(message) {}
};

class NodeStateException : public NodeException {
public:
    NodeStateException(NodeState current, NodeState expected);
    NodeState get_current_state() const { return current_; }
    NodeState get_expected_state() const { return expected_; }
};
```

---

## 5. 使用示例

### 5.1 基本使用

```cpp
#include <mruntime/node.h>

// 定义数据类型（需提供 serialize/deserialize 方法）
struct SensorData {
    int id;
    float temperature;

    std::vector<uint8_t> serialize() const {
        return {reinterpret_cast<const uint8_t*>(this),
                reinterpret_cast<const uint8_t*>(this) + sizeof(SensorData)};
    }

    static SensorData deserialize(const uint8_t* data, size_t size) {
        SensorData result;
        memcpy(&result, data, sizeof(SensorData));
        return result;
    }
};

int main() {
    // 创建节点
    auto node = mruntime::Node::create("sensor_node", 0);

    // 初始化并启动
    node->init();
    node->start();

    // 创建发布者
    auto publisher = node->create_publisher<SensorData>("sensor_topic");

    // 创建订阅者
    auto subscriber = node->create_subscriber<SensorData>("sensor_topic",
        [](const SensorData& data, uint64_t timestamp) {
            printf("Received: id=%d, temp=%.2f\n", data.id, data.temperature);
        });

    // 发布数据
    SensorData data{1, 25.5f};
    publisher->write(data);

    // 停止并销毁
    node->stop();
    node->destroy();

    return 0;
}
```

### 5.2 生命周期管理

```cpp
auto node = mruntime::Node::create("my_node", 0);

// 初始状态: UNINITIALIZED
assert(node->get_state() == mruntime::NodeState::UNINITIALIZED);

node->init();  // INITIALIZED
node->start();  // RUNNING

// 创建发布/订阅
auto pub = node->create_publisher<Data>("topic");

node->stop();   // STOPPED
node->destroy(); // DESTROYED
```

### 5.3 错误处理

```cpp
auto node = mruntime::Node::create("my_node", 0);

// 在未初始化状态下创建发布者会抛出异常
try {
    node->create_publisher<Data>("topic");
} catch (const mruntime::NodeStateException& e) {
    // e.what() 返回: "Node state error: expected INITIALIZED but current state is UNINITIALIZED"
}
```

---

## 6. 线程安全性

### 6.1 保护机制

| 成员 | 保护方式 |
|------|---------|
| state_ | state_mutex_ |
| publishers_ | endpoints_mutex_ |
| subscribers_ | endpoints_mutex_ |
| participant_ | Node 生命周期保证 |

### 6.2 设计原则

1. **状态转换原子性**: 所有状态修改通过 mutex 保护
2. **端点管理线程安全**: 发布/订阅创建和销毁在 mutex 保护下进行
3. **弱引用模式**: Publisher/Subscriber 持有 Node 的 weak_ptr，避免循环引用
4. **无锁数据传递**: 数据写入通过 mdds 底层处理

---

## 7. 与现有组件的关系

### 7.1 依赖关系

```
Node
  ├── mruntime::Clock (获取时间戳)
  └── mdds::DomainParticipant (底层 DDS 实现)
        ├── mdds::Publisher<T>
        ├── mdds::Subscriber<T>
        ├── mdds::Topic<T>
        └── mdds::Transport
```

### 7.2 复用现有实现

| 组件 | 复用方式 |
|------|---------|
| enable_shared_from_this | 与 Runtime 相同模式 |
| 工厂方法 create() | 与 Runtime 相同模式 |
| Clock | 用于时间戳生成 |
| DomainParticipant | 底层 DDS 实现 |

---

## 8. 设计权衡

### 8.1 设计决策

1. **为什么使用工厂方法 create()?**
   - 保证对象通过 shared_ptr 管理
   - 统一创建方式，便于内存管理

2. **为什么 Publisher/Subscriber 使用 weak_ptr<Node>?**
   - 避免循环引用（Node 持有 Publisher，Publisher 持有 Node）
   - 允许 Node 先于 Publisher 销毁

3. **为什么状态转换返回 bool 而不是 void？**
   - 给调用者明确的成功/失败反馈
   - 便于错误处理和日志记录

### 8.2 限制

1. 不支持动态添加/移除 QoS 配置（需要在创建时指定）
2. 不支持主题的动态订阅/取消订阅（需要重建节点）
3. 数据类型 T 必须提供 serialize()/deserialize() 方法

---

## 9. 未来扩展

### 9.1 可能的功能扩展

1. **参数管理**: 集成 config_server 的参数管理
2. **服务调用**: 添加 create_service<T>() / call_service<T>()
3. **节点组**: 支持节点组合和管理
4. **生命周期钩子**: 提供 on_init() / on_start() / on_stop() 回调
5. **健康监控**: 集成心跳和故障检测

### 9.2 性能优化

1. 对象池复用 Publisher/Subscriber
2. 无锁数据结构
3. 批量发布支持
