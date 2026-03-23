# MDDS 去中心化发现机制设计

## 1. 概述

### 1.1 设计目标

MDDS 原有架构依赖中心化的 DiscoveryServer，存在单点故障风险。本设计提出**轻量级去中心化发现机制**，特点：

- **无中心服务器** - 节点间通过 UDP Multicast 直接发现
- **快速发现** - 基于 Multicast 的即时发现，无需轮询
- **低开销** - 仅在状态变化时发送 announcement
- **自组织** - 节点自主加入/离开网络

### 1.2 与原架构对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        原架构 (中心化)                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│      ┌──────────┐                                   ┌──────────┐           │
│      │    P1    │───────────────────────────────────│    P2    │           │
│      └────┬─────┘                                   └────┬─────┘           │
│           │                                               │                  │
│           │              ┌───────────────┐               │                  │
│           └──────────────│ DiscoveryServer│───────────────┘                  │
│                          │   (127.0.0.1)  │                                │
│                          └───────────────┘                                │
│                                                                            │
│  问题:                                                                      │
│  - DiscoveryServer 是单点故障                                               │
│  - 所有节点必须预先知道 server 地址                                          │
│  - 扩展性受 server 性能限制                                                 │
│                                                                            │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        新架构 (去中心化)                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│      ┌──────────┐                                                           │
│      │    P1    │◄─────── Multicast (239.255.0.1:7411) ───────►┌──────────┐│
│      └────┬─────┘                                                       └────┬─────┘│
│           │                                                                   │     │
│           │           ┌─────────────────────────────────────────────────────┘     │
│           │           │                                                         │
│           │           ▼                                                         │
│      ┌────┴────────────────────────────────────────────────────────────────┐    │
│      │                    P2P Mesh (逻辑网络)                                │    │
│      │                                                                      │    │
│      │   所有节点通过 Multicast 互相发现                                      │    │
│      │   无需预配置任何中心节点                                              │    │
│      │                                                                      │    │
│      └──────────────────────────────────────────────────────────────────────┘    │
│                                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 2. 技术方案

### 2.1 Multicast 组播配置

| 参数 | 值 | 说明 |
|------|-----|------|
| Multicast IP | 239.255.0.1 | Site-local multicast (239.255.0.0 - 239.255.255.255) |
| Discovery Port | 7411 | 发现消息专用端口 (与数据传输端口 7412 区分) |
| Announcement Interval | 5s | 定期宣告间隔 (仅在启动/状态变化时立即宣告) |
| Peer Timeout | 15s | 超过此时间未收到心跳则移除对等节点 |

### 2.2 发现消息类型

```cpp
enum class DiscoveryMsgType : uint8_t {
    ANNOUNCE    = 0x10,  // 节点宣告 (发布者/订阅者在线)
    QUERY       = 0x11,  // 查询话题
    RESPONSE    = 0x12,  // 查询响应
    BYE         = 0x13   // 节点离开
};
```

### 2.3 消息格式

```
┌──────────────────────────────────────────────────────────────┐
│                    Discovery Message                         │
├──────────────────────────────────────────────────────────────┤
│  Header (16 bytes)                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ magic: 4 bytes     = 0x4D444453 ("MDDS")              │ │
│  │ version: 1 byte    = 0x01                             │ │
│  │ type: 1 byte       = DiscoveryMsgType                  │ │
│  │ flags: 1 byte      = 0x00 (reserved)                 │ │
│  │ domain_id: 2 bytes = Domain ID                        │ │
│  │ length: 2 bytes    = payload length                   │ │
│  │ sender_id: 4 bytes = Participant ID                   │ │
│  └────────────────────────────────────────────────────────┘ │
│  Payload (variable)                                        │
└──────────────────────────────────────────────────────────────┘
```

#### ANNOUNCE 消息 (宣告在线)

```
┌──────────────────────────────────────────────────────────────┐
│  Endpoint Info (28 bytes)                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ participant_id: 4 bytes                               │ │
│  │ topic_id: 4 bytes                                     │ │
│  │ topic_name: 32 bytes (null-padded)                    │ │
│  │ role: 1 byte (0x01=Publisher, 0x02=Subscriber)       │ │
│  │ transport: 1 byte (0x01=UDP, 0x02=SHM)               │ │
│  │ port: 2 bytes (UDP port, or 0 for SHM)               │ │
│  │ address: 16 bytes (IP string, null-padded)           │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

#### QUERY 消息 (查询话题)

```
┌──────────────────────────────────────────────────────────────┐
│  Query Payload (36 bytes)                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ topic_name: 32 bytes                                  │ │
│  │ role: 1 byte (0x01=找Publisher, 0x02=找Subscriber) │ │
│  │ reserved: 3 bytes                                    │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

#### RESPONSE 消息 (响应查询)

```
┌──────────────────────────────────────────────────────────────┐
│  Response Payload                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ topic_name: 32 bytes                                  │ │
│  │ endpoint_count: 1 byte                                │ │
│  │ endpoints: endpoint_count × 28 bytes                   │ │
│  │   - participant_id: 4 bytes                          │ │
│  │   - port: 2 bytes                                    │ │
│  │   - transport: 1 byte                                │ │
│  │   - address: 16 bytes                                │ │
│  │   - (topic_id 不返回，因为通过 topic_name 匹配)        │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

#### BYE 消息 (节点离开)

```
┌──────────────────────────────────────────────────────────────┐
│  Bye Payload (6 bytes)                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ participant_id: 4 bytes                               │ │
│  │ reserved: 2 bytes                                    │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

## 3. 发现流程

### 3.1 节点启动流程

```
1. 节点启动，创建 UDP Multicast Socket
   └─ 加入 Multicast 组 239.255.0.1

2. 启动接收线程，持续监听 Multicast 消息

3. 发送 ANNOUNCE 宣告所有已注册的 pub/sub
   └─ 每个 topic 单独发送一条 ANNOUNCE

4. 维护本地 Peer Table
   └─ 记录已发现的对等节点信息
```

### 3.2 发布者发现订阅者流程

```
┌──────────┐                          ┌──────────┐
│ Publisher│                          │Subscriber│
└────┬─────┘                          └────┬─────┘
     │                                      │
     │  1. ANNOUNCE (topic="SensorData",    │
     │              role=Publisher)          │
     │ ──────────────────────────────────►  │
     │         (Multicast)                   │
     │                                      │
     │  2. 发现 Subscriber 存在               │
     │     直接建立 P2P 连接                  │
     │                                      │
     │  3. 开始发送数据 (UDP/shm)             │
     │ ═══════════════════════════════════► │
```

### 3.3 订阅者发现发布者流程

```
┌──────────┐                          ┌──────────┐
│Publisher │                          │Subscriber│
└────┬─────┘                          └────┬─────┘
     │                                      │
     │  1. ANNOUNCE (topic="SensorData",    │
     │              role=Publisher)          │
     │ ──────────────────────────────────► │
     │         (Multicast)                   │
     │                                      │
     │  2. 订阅者收到 announcement          │
     │     记录发布者信息到 Peer Table       │
     │     发现匹配的 Publisher              │
     │                                      │
     │  3. 直接建立 P2P 连接                │
     │◄════════════════════════════════════ │
     │                                      │
     │  4. 开始接收数据                      │
```

### 3.4 订阅者主动查询流程

```
┌──────────┐                          ┌──────────┐
│Subscriber│                          │Publisher │
└────┬─────┘                          └────┬─────┘
     │                                      │
     │  1. QUERY (topic="SensorData",       │
     │           role=找Publisher)           │
     │ ──────────────────────────────────► │
     │         (Multicast)                   │
     │                                      │
     │  2. 发布者收到 QUERY                  │
     │     检查自己是否有匹配的 topic         │
     │                                      │
     │  3. RESPONSE (topic="SensorData",    │
     │              endpoints=[pub_info])    │
     │ ◄────────────────────────────────── │
     │                                      │
     │  4. 订阅者收到 RESPONSE               │
     │     直接建立连接                      │
```

## 4. 数据结构

### 4.1 Peer Table 条目

```cpp
struct PeerEntry {
    uint32_t participant_id;
    std::string address;
    uint16_t port;
    TransportType transport;
    std::vector<std::string> published_topics;    // 此节点发布的 topic
    std::vector<std::string> subscribed_topics;  // 此节点订阅的 topic
    std::chrono::steady_clock::time_point last_announce;
    bool is_alive() const {
        return std::chrono::steady_clock::now() - last_announce
               < std::chrono::seconds(PEER_TIMEOUT_SEC);
    }
};
```

### 4.2 Topic Match Table

```cpp
struct TopicMatch {
    std::string topic_name;
    std::vector<PeerEntry> publishers;
    std::vector<PeerEntry> subscribers;
};
```

## 5. 实现要点

### 5.1 Multicast Socket 配置

```cpp
int create_multicast_socket(uint16_t port) {
    int sock = socket(AF_INET, SOCK_DGRAM, 0);

    // 允许复用地址
    int reuse = 1;
    setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

    // 加入 Multicast 组
    struct ip_mreq mreq;
    mreq.imr_multiaddr.s_addr = inet_addr("239.255.0.1");
    mreq.imr_interface.s_addr = htonl(INADDR_ANY);
    setsockopt(sock, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, sizeof(mreq));

    // 设置 Multicast TTL
    int ttl = 1;  // 本地网络
    setsockopt(sock, IPPROTO_IP, IP_MULTICAST_TTL, &ttl, sizeof(ttl));

    // 绑定端口
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    bind(sock, (struct sockaddr*)&addr, sizeof(addr));

    return sock;
}
```

### 5.2 定期清理过期 Peer

```cpp
void PeerDatabase::cleanup_stale_peers() {
    std::lock_guard<std::mutex> lock(mutex_);
    for (auto it = peers_.begin(); it != peers_.end(); ) {
        if (!it->second.is_alive()) {
            it = peers_.erase(it);
        } else {
            ++it;
        }
    }
}
```

### 5.3 心跳机制

- 每个节点每 5 秒发送一次 ANNOUNCE
- 超过 15 秒未收到某节点的 ANNOUNCE，则认为该节点已离开
- 节点离开时发送 BYE 消息（可选，友好退出）

## 6. API 扩展

### 6.1 DomainParticipant 扩展

```cpp
class DomainParticipant {
public:
    // 设置发现模式
    void set_discovery_mode(DiscoveryMode mode) {
        discovery_mode_ = mode;
    }

    // 启用/禁用去中心化发现
    void enable_multicast_discovery(bool enable) {
        enable_multicast_discovery_ = enable;
    }

    // 设置 Multicast 地址和端口
    void set_multicast_endpoint(const std::string& ip, uint16_t port) {
        multicast_ip_ = ip;
        multicast_port_ = port;
    }

private:
    DiscoveryMode discovery_mode_{DiscoveryMode::CENTRALIZED};
    bool enable_multicast_discovery_{false};
    std::string multicast_ip_{"239.255.0.1"};
    uint16_t multicast_port_{7411};
};
```

### 6.2 发现模式枚举

```cpp
enum class DiscoveryMode {
    CENTRALIZED,     // 中心化 DiscoveryServer
    MULTICAST,       // 去中心化 Multicast 发现
    HYBRID           // 混合模式：优先 Multicast，fallback 到中心化
};
```

## 7. 兼容性考虑

### 7.1 与原 DiscoveryServer 兼容

- 保持原有 Discovery 类和 DiscoveryServer 类不变
- 新增 `MulticastDiscovery` 类独立实现
- 通过 `DiscoveryMode` 枚举选择发现机制
- 支持运行时切换发现模式

### 7.2 网络兼容性

- Multicast 需要网络设备支持
- 如果 Multicast 不可用，自动 fallback 到中心化模式
- 支持配置不同的 Multicast 地址

## 8. 性能特性

| 指标 | 中心化 | 去中心化 (Multicast) |
|------|--------|---------------------|
| 发现延迟 | 10-50ms (依赖 server) | <5ms (广播) |
| 消息复杂度 | O(n) | O(1) per announcement |
| 扩展性 | 受 server 限制 | 受网络带宽限制 |
| 容错性 | 单点故障 | 无单点故障 |
| 网络依赖 | 需要知道 server 地址 | 只需网络支持 multicast |

## 9. 后续优化方向

1. **分层 Multicast** - 大规模网络使用分层 Multicast 减少广播风暴
2. **Gossip 协议** - 使用 Gossip 进行状态同步
3. **DHT 辅助** - 分布式 Hash Table 加速查找
4. **QoS 感知** - 在发现阶段考虑 QoS 匹配
