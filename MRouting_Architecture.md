# MRouting 架构与设计

## 1. 概述

MRouting 是 MOSS 项目中 SOME/IP 协议栈的集中路由管理器节点，类似于 vsomeip 中的 routing manager 功能，但独立为单独进程。

### 1.1 设计目标

- **集中路由**: 所有服务注册和发现信息集中管理，无需每个节点都维护完整路由表
- **跨进程通信**: MRouting 与 MSomeip 应用之间通过共享内存 (mshm) 通信，高效低延迟
- **简化部署**: 应用节点无需承担路由管理职责，降低资源消耗
- **网络隔离**: 跨网段/跨 ECU 的服务路由统一由 MRouting 处理

### 1.2 与 VSomeip 的区别

| 特性 | VSomeip | MRouting + MSomeip |
|------|---------|-------------------|
| 路由管理 | 内嵌在每个节点 | 独立 MRouting 进程 |
| 进程间通信 | DBus | 共享内存 (mshm) |
| 路由表同步 | 广播式 | 共享内存直接访问 |
| 资源消耗 | 每个节点都要 | 仅 MRouting 消耗 |

---

## 2. 系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         MRouting 进程                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Route Manager                          │  │
│  │  ┌────────────────┐  ┌────────────────┐  ┌───────────┐  │  │
│  │  │ Service Router │  │ Subscription    │  │ Endpoint  │  │  │
│  │  │                │  │ Router          │  │ Manager   │  │  │
│  │  └────────────────┘  └────────────────┘  └───────────┘  │  │
│  │  ┌────────────────┐  ┌────────────────┐                  │  │
│  │  │ SD Cache       │  │ Route Policy   │                  │  │
│  │  │ (Service Map)  │  │ Engine         │                  │  │
│  │  └────────────────┘  └────────────────┘                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ▲                                    │
│                     共享内存 (mshm)                                │
└──────────────────────────────│──────────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
┌───────┴───────┐    ┌────────┴────────┐    ┌───────┴───────┐
│   MSomeip     │    │   MSomeip       │    │   MSomeip     │
│   App 1       │    │   App 2         │    │   App N       │
│  (Service A)  │    │  (Service B)    │    │  (Client)     │
└───────────────┘    └─────────────────┘    └───────────────┘
```

### 2.2 组件交互图

```
┌─────────────────────────────────────────────────────────────────┐
│                         MRouting 进程                            │
│                                                                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐ │
│  │ SD Handler  │◄──►│   Route     │◄──►│  Endpoint Manager    │ │
│  │             │    │   Table    │    │  (IP:Port 表)       │ │
│  └──────┬──────┘    └──────┬──────┘    └─────────────────────┘ │
│         │                  │                                    │
│         ▼                  ▼                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Shared Memory Region (mshm)                    │ │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐ │ │
│  │  │ Route Cmd  │  │ Route Rsp  │  │ Service Registry       │ │ │
│  │  │   Queue    │  │   Queue    │  │ (ServiceId → Endpoints)│ │ │
│  │  └────────────┘  └────────────┘  └────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
         ▲                   ▲                   ▲
         │ shm_join          │ shm_join          │ shm_join
         │                   │                   │
┌────────┴───────┐   ┌───────┴────────┐   ┌──────┴───────┐
│   MSomeip      │   │    MSomeip     │   │   MSomeip    │
│   App 1        │   │    App 2       │   │   App N      │
│ ┌────────────┐ │   │  ┌───────────┐  │   │ ┌──────────┐ │
│ │Local SD    │ │   │  │Local SD  │  │   │ │Local SD  │ │
│ │Agent       │ │   │  │Agent     │  │   │ │Agent     │ │
│ └────────────┘ │   │  └───────────┘  │   │ └──────────┘ │
└───────────────┘   └─────────────────┘   └──────────────┘
```

---

## 3. 模块设计

### 3.1 MRouting 模块结构

```
mrouting/
├── include/
│   └── mrouting/
│       ├── route_manager.h       # 路由管理器主类
│       ├── route_table.h          # 路由表定义
│       ├── sd_handler.h           # SD 消息处理器
│       ├── endpoint_manager.h     # 端点管理器
│       ├── shm_interface.h        # 共享内存接口
│       ├── types.h                # 类型定义
│       └── message.h              # 路由消息格式
├── src/
│   ├── route_manager.cpp
│   ├── route_table.cpp
│   ├── sd_handler.cpp
│   ├── endpoint_manager.cpp
│   ├── shm_interface.cpp
│   └── main.cpp
└── CMakeLists.txt
```

### 3.2 核心组件

#### 3.2.1 RouteManager

路由管理器主类，负责：
- 初始化和管理共享内存接口
- 处理来自应用节点的路由请求
- 维护全局服务路由表
- 协调 SD Handler 和 Endpoint Manager

#### 3.2.2 RouteTable

路由表核心数据结构：
```cpp
struct RouteEntry {
    ServiceIdTuple service_id;       // Service + Instance
    MajorVersion major_version;
    MinorVersion minor_version;
    std::vector<Endpoint> endpoints; // 服务端点列表
    std::chrono::steady_clock::time_point ttl_expire;
    bool is_local;                    // 是否本地服务
};

struct SubscriptionEntry {
    ServiceIdTuple service_id;
    EventgroupId eventgroup_id;
    std::vector<Endpoint> subscribers; // 订阅者列表
    MajorVersion major_version;
    uint32_t ttl;
};
```

#### 3.2.3 SDHandler

SD 消息处理器：
- 接收来自应用节点的 SD 消息（通过共享内存）
- 解析 Offer Service、Find Service、Subscribe 等消息
- 更新 RouteTable 中的服务注册和订阅信息
- 触发必要的路由更新通知

#### 3.2.4 EndpointManager

端点管理器：
- 维护 ServiceId → Endpoint 的映射
- 支持多播和单播端点
- 管理 TCP/UDP 端点选择

#### 3.2.5 ShmInterface

共享内存通信接口：
- 创建共享内存服务端（MRouting 作为 Server）
- 处理命令队列和响应队列
- 提供事件通知机制

---

## 4. 共享内存接口设计

### 4.1 共享内存区域布局

```
┌─────────────────────────────────────────────────────────────┐
│                   Shared Memory Layout                       │
├─────────────────────────────────────────────────────────────┤
│  Header (64 bytes)                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ magic_number: uint32_t = 0x4D525450 ("MRTP")           │ │
│  │ version: uint16_t                                       │ │
│  │ flags: uint16_t                                         │ │
│  │ command_queue_offset: uint32_t                          │ │
│  │ command_queue_size: uint32_t                            │ │
│  │ response_queue_offset: uint32_t                         │ │
│  │ response_queue_size: uint32_t                            │ │
│  │ service_registry_offset: uint32_t                        │ │
│  │ service_registry_size: uint32_t                          │ │
│  │ client_count: uint32_t                                   │ │
│  │ reserved: padding to 64 bytes                            │ │
│  └────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│  Command Queue (可变长)                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ count: uint32_t                                         │ │
│  │ read_index: uint32_t                                    │ │
│  │ write_index: uint32_t                                   │ │
│  │ commands[]: RouteCommand                                │ │
│  │   - type: RouteCommandType                              │ │
│  │   - payload_length: uint32_t                            │ │
│  │   - payload: uint8_t[]                                   │ │
│  └────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│  Response Queue (可变长)                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ count: uint32_t                                         │ │
│  │ read_index: uint32_t                                    │ │
│  │ write_index: uint32_t                                   │ │
│  │ responses[]: RouteResponse                              │ │
│  └────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│  Service Registry (可变长)                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ count: uint32_t                                         │ │
│  │ services[]: ServiceRegistryEntry                       │ │
│  │   - service_id: uint16_t                                │ │
│  │   - instance_id: uint16_t                               │ │
│  │   - major_version: uint8_t                              │ │
│  │   - minor_version: uint32_t                             │ │
│  │   - endpoint_count: uint32_t                            │ │
│  │   - endpoints[]: Endpoint                                │ │
│  │   - ttl: uint32_t                                        │ │
│  │   - is_local: bool                                      │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 路由命令格式

```cpp
enum class RouteCommandType : uint8_t {
    REGISTER_SERVICE = 0x01,      // 应用注册服务 (Offer)
    UNREGISTER_SERVICE = 0x02,    // 应用注销服务
    FIND_SERVICE = 0x03,          // 应用查找服务
    SUBSCRIBE = 0x04,             // 订阅事件组
    UNSUBSCRIBE = 0x05,           // 取消订阅
    LOOKUP_SERVICE = 0x06,        // 查询服务endpoint
    LOOKUP_SUBSCRIBERS = 0x07,    // 查询某服务的订阅者
    HEARTBEAT = 0x08,             // 心跳保活
};

struct RouteCommand {
    uint32_t sequence;             // 命令序列号
    uint16_t client_id;            // 发送方 ClientID
    RouteCommandType type;
    uint32_t payload_length;
    // payload follows
};
```

### 4.3 路由响应格式

```cpp
enum class RouteResponseType : uint8_t {
    SERVICE_AVAILABLE = 0x10,     // 服务可用通知
    SERVICE_UNAVAILABLE = 0x11,   // 服务不可用通知
    SUBSCRIPTION_ACK = 0x12,      // 订阅确认
    SUBSCRIPTION_NACK = 0x13,      // 订阅拒绝
    LOOKUP_RESULT = 0x14,         // 查询结果
    ERROR = 0x1F,                  // 错误响应
};

struct RouteResponse {
    uint32_t sequence;             // 对应命令的序列号
    RouteResponseType type;
    uint32_t payload_length;
    // payload follows
};
```

---

## 5. MSomeip 侧的修改

### 5.1 新增 ShmAgent 组件

在 MSomeip 中新增 ShmAgent，负责：
- 连接 MRouting 的共享内存
- 发送路由命令（Offer Service、Find Service 等）
- 接收路由响应和通知
- 将远程服务信息同步到本地 ServiceDiscovery

### 5.2 MSomeip 架构变更

```
┌─────────────────────────────────────────────────────────────────┐
│                        MSomeip Runtime                           │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                      Application                            │ │
│  └────────────────────────────┬───────────────────────────────┘ │
│                               │                                   │
│  ┌────────────────────────────▼───────────────────────────────┐ │
│  │                    ServiceDiscovery                          │ │
│  │  ┌────────────────┐           ┌────────────────┐            │ │
│  │  │ Local SD Agent │           │ Remote SD Handler│           │ │
│  │  │ (本地服务广播)  │           │ (来自 ShmAgent) │            │ │
│  │  └────────────────┘           └────────────────┘            │ │
│  └────────────────────────────┬───────────────────────────────┘ │
│                               │                                   │
│  ┌────────────────────────────▼───────────────────────────────┐ │
│  │                      ShmAgent                               │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │  shm_join("mrouting_shm") → 共享内存连接            │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                               │                                   │
└───────────────────────────────┼───────────────────────────────────┘
                                │ 共享内存
                                ▼
                    ┌───────────────────────┐
                    │      MRouting         │
                    │    (Route Manager)    │
                    └───────────────────────┘
```

### 5.3 ShmAgent API

```cpp
class ShmAgent {
public:
    // 连接/断开 MRouting
    bool connect(const std::string& routing_shm_name = "mrouting_shm");
    void disconnect();

    // 发送路由命令
    bool send_register_service(const ServiceConfig& config);
    bool send_unregister_service(ServiceId service, InstanceId instance);
    bool send_find_service(ServiceId service, InstanceId instance = 0xFFFF);
    bool send_subscribe(ServiceId service, InstanceId instance,
                        EventgroupId eventgroup, MajorVersion major_version);
    bool send_unsubscribe(ServiceId service, InstanceId instance,
                          EventgroupId eventgroup);

    // 查询接口
    std::vector<Endpoint> lookup_service_endpoints(ServiceId service,
                                                    InstanceId instance);
    std::vector<Endpoint> lookup_subscribers(ServiceId service,
                                               InstanceId instance,
                                               EventgroupId eventgroup);

    // 设置回调
    void set_service_available_handler(AvailabilityHandler handler);
    void set_subscription_ack_handler(SubscriptionHandler handler);

    // 获取通知 fd
    int get_notify_fd() const;

private:
    void worker_thread();
    void process_responses();

    shm_handle_t* shm_handle_;
    std::thread worker_thread_;
    std::atomic<bool> running_{false};

    AvailabilityHandler availability_handler_;
    SubscriptionHandler subscription_handler_;
};
```

---

## 6. 工作流程

### 6.1 服务注册流程 (Offer Service)

```
┌─────────┐         ┌─────────┐         ┌─────────┐
│ App     │         │ ShmAgent│         │MRouting │
│(Service)│         │         │         │         │
└────┬────┘         └────┬────┘         └────┬────┘
     │                   │                   │
     │ offer_service()   │                   │
     │ ─────────────────►│                   │
     │                   │                   │
     │  REGISTER_SERVICE │                   │
     │ ─────────────────►│                   │
     │                   │                   │
     │                   │ shm_notify()      │
     │                   │ ─────────────────►│
     │                   │                   │
     │                   │     (更新路由表)   │
     │                   │                   │
     │  LOOKUP_RESULT    │                   │
     │ ◄─────────────────│                   │
     │                   │                   │
     │                   │  广播 SERVICE_AVAILABLE │
     │                   │ ─────────────────►│ (所有客户端)
     │                   │                   │
```

### 6.2 服务发现流程 (Find Service)

```
┌─────────┐         ┌─────────┐         ┌─────────┐
│ Client  │         │ ShmAgent│         │MRouting │
│ App     │         │         │         │         │
└────┬────┘         └────┬────┘         └────┬────┘
     │                   │                   │
     │ find_service()    │                   │
     │ ─────────────────►│                   │
     │                   │                   │
     │  FIND_SERVICE     │                   │
     │ ─────────────────►│                   │
     │                   │ shm_notify()      │
     │                   │ ─────────────────►│
     │                   │                   │
     │                   │     (查找路由表)   │
     │                   │                   │
     │  LOOKUP_RESULT    │                   │
     │ ◄─────────────────│                   │
     │                   │                   │
     │                   │  SERVICE_AVAILABLE│
     │                   │ ◄─────────────────│
     │                   │                   │
```

### 6.3 订阅流程 (Subscribe Eventgroup)

```
┌─────────┐         ┌─────────┐         ┌─────────┐
│ Client  │         │ ShmAgent│         │MRouting │
│ App     │         │         │         │         │
└────┬────┘         └────┬────┘         └────┬────┘
     │                   │                   │
     │ subscribe()       │                   │
     │ ─────────────────►│                   │
     │                   │                   │
     │ SUBSCRIBE         │                   │
     │ ─────────────────►│                   │
     │                   │ shm_notify()      │
     │                   │ ─────────────────►│
     │                   │                   │
     │                   │     (查找服务+订阅者)│
     │                   │                   │
     │ SUBSCRIPTION_ACK │                   │
     │ ◄─────────────────│                   │
     │                   │                   │
     │                   │  转发 SUBSCRIBE   │
     │                   │ ─────────────────►│ (到服务端)
     │                   │                   │
```

---

## 7. 错误处理与超时

### 7.1 心跳保活机制

- MSomeip 应用定期向 MRouting 发送 HEARTBEAT 命令
- 超过 3 次未收到心跳的应用，MRouting 将其标记为离线
- 应用离线后，其注册的服务和订阅将被清除

### 7.2 服务 TTL 管理

- MRouting 维护每个注册服务的 TTL
- TTL 过期前，MRouting 向应用发送刷新请求
- 应用无响应则从路由表中移除

### 7.3 错误响应

| 错误码 | 含义 |
|--------|------|
| 0x01 | 服务未找到 |
| 0x02 | 订阅已存在 |
| 0x03 | 订阅不存在 |
| 0x04 | 共享内存错误 |
| 0x05 | 队列满 |
| 0xFF | 未知错误 |

---

## 8. 配置

### 8.1 MRouting 配置

```yaml
# mrouting.yaml
mrouting:
  shm_name: "mrouting_shm"        # 共享内存名称
  shm_data_size: 65536            # 共享内存数据区大小

  sd:
    multicast_address: "224.244.224.245"
    port: 30490

  routing:
    heartbeat_interval_ms: 1000    # 心跳间隔
    heartbeat_max_miss: 3         # 最大丢失心跳数
    service_ttl_ms: 60000         # 服务 TTL

  queue:
    command_queue_size: 256        # 命令队列深度
    response_queue_size: 512       # 响应队列深度
```

### 8.2 MSomeip 配置扩展

```yaml
# msomeip.yaml
msomeip:
  application:
    name: "my_app"
    client_id: 0x1001

  routing:
    enabled: true                  # 启用 MRouting
    shm_name: "mrouting_shm"       # 连接到的共享内存
    heartbeat_interval_ms: 1000    # 心跳间隔
```

---

## 9. 安全性考虑

### 9.1 访问控制

- MRouting 可配置允许连接的客户端列表
- 基于 ClientID 进行认证

### 9.2 数据一致性

- 使用共享内存的进程间锁保证路由表一致性
- 命令队列使用原子操作保证顺序

---

## 10. 未来扩展

### 10.1 多 MRouting 集群

- 支持多个 MRouting 实例实现负载均衡和容错
- 跨集群服务同步

### 10.2 策略引擎

- 支持基于策略的服务路由
- 灰度发布、流量染色等高级功能

### 10.3 TCP 通道

- 对于跨网络通信，支持 MRouting 与 MSomeip 通过 TCP 通道通信
- 适用于 ECU 之间的 SOME/IP 路由

---

## 11. 文件结构

```
moss/
├── mshm/                      # 共享内存公共模块
├── msomeip/                   # SOME/IP 协议栈
│   └── msomeip/
│       └── include/msomeip/
│           ├── application.h      # 现有 Application 类
│           ├── runtime.h          # 现有 Runtime 类
│           ├── shm_agent.h        # [新增] 共享内存代理
│           └── ...
├── mrouting/                  # [新增] 路由管理节点
│   ├── include/
│   │   └── mrouting/
│   │       ├── route_manager.h
│   │       ├── route_table.h
│   │       ├── sd_handler.h
│   │       ├── endpoint_manager.h
│   │       ├── shm_interface.h
│   │       ├── types.h
│   │       └── message.h
│   └── src/
│       ├── route_manager.cpp
│       ├── route_table.cpp
│       ├── sd_handler.cpp
│       ├── endpoint_manager.cpp
│       ├── shm_interface.cpp
│       └── main.cpp
└── doc/
    ├── MRouting_Architecture.md   # [新增] 本文档
    ├── shared_memory_design.md
    └── ...
```
