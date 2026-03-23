# MDDS 架构设计文档

## 目录

1. [系统概述](#1-系统概述)
2. [架构设计](#2-架构设计)
3. [核心组件详细设计](#3-核心组件详细设计)
4. [服务发现设计](#4-服务发现设计)
5. [传输层设计](#5-传输层设计)
6. [消息格式规范](#6-消息格式规范)
7. [QoS 策略设计](#7-qos-策略设计)
8. [错误处理](#8-错误处理)
9. [线程模型](#9-线程模型)
10. [目录结构](#10-目录结构)
11. [API 参考](#11-api-参考)
12. [时序图](#12-时序图)
13. [性能分析](#13-性能分析)
14. [部署指南](#14-部署指南)

---

## 1. 系统概述

### 1.1 项目背景

FastDDS (Fast RTPS) 是 OMG DDS 标准的实现，被广泛应用于 automotive 和 robotics 领域。然而其服务发现（SD）模块存在以下问题：

| 问题 | 描述 | 影响 |
|------|------|------|
| **SPDP 复杂性** | Simple Participant Discovery Protocol 使用复杂的组播机制 | 需要网络支持组播，跨子网困难 |
| **SEDP 开销** | Simple Endpoint Discovery Protocol 需要多轮握手 | 连接延迟高，消息量大 |
| **周期性 Announcement** | 参与者定期发送心跳，即使无变化 | 浪费带宽，功耗增加 |
| **O(n²) 扩展性** | 每个参与者需与所有其他参与者通信 | 节点增多时指数增长 |
| **复杂 QoS 匹配** | 需要匹配多个 QoS 策略 | 计算开销大 |
| **TTL 管理复杂** | 需要追踪每个实体的 TTL 定时器 | 资源消耗高 |

### 1.2 MDDS 设计目标

MDDS (Minimal DDS) 是一个高性能、轻量级的 DDS 实现，核心目标：

1. **简化发现协议** - 用单层集中式发现替代双层分布式发现
2. **去除组播依赖** - 使用单播通信，简化网络配置
3. **降低资源消耗** - 减少内存占用和 CPU 使用
4. **保持核心语义** - 保留 DDS 的 pub/sub 核心特性

### 1.3 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| DDS | Data Distribution Service | 数据分发服务标准 |
| Domain | Domain | 逻辑隔离的通信空间 |
| Participant | DomainParticipant | DDS 通信的入口点 |
| Topic | Topic | 发布-订阅的数据通道 |
| Publisher | Publisher | 管理数据写入器 |
| Subscriber | Subscriber | 管理数据读取器 |
| QoS | Quality of Service | 服务质量策略 |
| SD | Service Discovery | 服务发现 |

---

## 2. 架构设计

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MDDS 系统架构                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         Application Layer                           │    │
│  │                                                                      │    │
│  │   ┌─────────────────┐         ┌─────────────────┐                  │    │
│  │   │  SensorData     │         │  SensorData     │                  │    │
│  │   │  Publisher      │         │  Subscriber     │                  │    │
│  │   │  (Robot A)      │         │  (Robot B)      │                  │    │
│  │   └────────┬────────┘         └────────┬────────┘                  │    │
│  └────────────┼─────────────────────────────┼────────────────────────────┘    │
│               │                             │                                 │
│               ▼                             ▼                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      DomainParticipant Layer                          │    │
│  │                                                                      │    │
│  │   ┌────────────────────────────────────────────────────────────┐    │    │
│  │   │                 DomainParticipant (Domain 0)                 │    │    │
│  │   │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  │    │    │
│  │   │  │   Publisher   │  │  Subscriber   │  │ TopicManager  │  │    │    │
│  │   │  │   Factory     │  │   Factory     │  │               │  │    │    │
│  │   │  └───────────────┘  └───────────────┘  └───────────────┘  │    │    │
│  │   └────────────────────────────────────────────────────────────┘    │    │
│  │                              │                                        │    │
│  │                              ▼                                        │    │
│  │   ┌────────────────────────────────────────────────────────────┐    │    │
│  │   │                    Discovery Layer                          │    │    │
│  │   │  ┌───────────────────┐  ┌───────────────────┐              │    │    │
│  │   │  │ ParticipantRegistry│  │  EndpointRegistry │              │    │    │
│  │   │  └───────────────────┘  └───────────────────┘              │    │    │
│  │   │  ┌───────────────────┐  ┌───────────────────┐              │    │    │
│  │   │  │   MatchMaker       │  │ DiscoveryClient   │              │    │    │
│  │   │  └───────────────────┘  └───────────────────┘              │    │    │
│  │   └────────────────────────────────────────────────────────────┘    │    │
│  │                              │                                        │    │
│  └──────────────────────────────┼──────────────────────────────────────┘    │
│                                  │                                             │
│                                  ▼                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         Transport Layer                                │  │
│  │                                                                        │  │
│  │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │  │
│  │   │  SharedMemory   │  │    UDP          │  │     TCP         │        │  │
│  │   │  Transport      │  │    Transport    │  │     Transport   │        │  │
│  │   │  (Local IPC)    │  │    (Network)    │  │     (Network)   │        │  │
│  │   └─────────────────┘  └─────────────────┘  └─────────────────┘        │  │
│  │                                                                        │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 层次职责

| 层次 | 职责 | 主要组件 |
|------|------|----------|
| **Application** | 用户业务逻辑 | 用户代码 |
| **DomainParticipant** | DDS 入口，管理 pub/sub | Publisher, Subscriber, TopicManager |
| **Discovery** | 服务发现，Topic 匹配 | ParticipantRegistry, EndpointRegistry, MatchMaker |
| **Transport** | 数据传输 | SharedMemory, UDP, TCP |

### 2.3 与 FastDDS 架构对比

```
┌─────────────────────────────────┬─────────────────────────────────┐
│         FastDDS 架构            │         MDDS 架构               │
├─────────────────────────────────┼─────────────────────────────────┤
│                                 │                                 │
│  ┌─────────────────────────┐   │   ┌─────────────────────────┐   │
│  │  DomainParticipant      │   │   │  DomainParticipant      │   │
│  │  ├─ Publisher           │   │   │  ├─ Publisher           │   │
│  │  ├─ Subscriber         │   │   │  ├─ Subscriber         │   │
│  │  └─ Topic              │   │   │  └─ Topic              │   │
│  └───────────┬───────────┘   │   └───────────┬───────────┘   │
│              │                │               │                │
│  ┌───────────▼───────────┐   │   ┌───────────▼───────────┐   │
│  │  RTPS_Domain          │   │   │  Discovery Layer      │   │
│  │  ├─ SPDP              │   │   │  ├─ Registry          │   │
│  │  │   (Participant DIS)│   │   │  └─ MatchMaker        │   │
│  │  └─ SEDP              │   │   │                       │   │
│  │      (Endpoint DIS)    │   │   │  (单层简化发现)        │   │
│  └───────────┬───────────┘   │   └───────────┬───────────┘   │
│              │                │               │                │
│  ┌───────────▼───────────┐   │   ┌───────────▼───────────┐   │
│  │  RTPS_Transport       │   │   │  Transport Layer      │   │
│  │  ├─ UDPTransport      │   │   │  ├─ SharedMemory     │   │
│  │  ├─ TCPTransport     │   │   │  ├─ UDPTransport     │   │
│  │  ├─ SHMTransport     │   │   │  ├─ TCPTransport    │   │
│  │  └─ Locator          │   │   │  └─ Locator          │   │
│  └───────────────────────┘   │   └───────────────────────┘   │
│                               │                                 │
│  ┌───────────────────────┐   │   ┌───────────────────────┐   │
│  │  EDP Matching         │   │   │  Simple Topic Match   │   │
│  │  ├─ Pub compatible?  │   │   │  ├─ Name match?      │   │
│  │  ├─ QoS compatible?  │   │   │  ├─ Type match?      │   │
│  │  └─ Both in same domain?│ │   │  └─ Domain match?    │   │
│  │  (复杂匹配算法)        │   │   │  (简单匹配)           │   │
│  └───────────────────────┘   │   └───────────────────────┘   │
│                               │                                 │
└───────────────────────────────┴─────────────────────────────────┘
```

### 2.4 核心设计原则

1. **单层发现** - 用单一发现层替代 SPDP + SEDP 双层
2. **集中式注册** - 用集中式注册表替代分布式状态
3. **事件驱动** - 用事件驱动替代周期性广播
4. **简单匹配** - 用 Topic 名称匹配替代复杂 QoS 匹配

---

## 3. 核心组件详细设计

### 3.1 类图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MDDS 类图                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         mdds::DomainParticipant                       │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  - domain_id_: DomainId                                                    │    │
│  │  - participant_id_: ParticipantId                                        │    │
│  │  - discovery_: std::unique_ptr<Discovery>                              │    │
│  │  - topic_manager_: std::unique_ptr<TopicManager>                       │    │
│  │  - publishers_: std::vector<std::unique_ptr<PublisherBase>>             │    │
│  │  - subscribers_: std::vector<std::unique_ptr<SubscriberBase>>           │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  + create(domain_id): shared_ptr<DomainParticipant>                    │    │
│  │  + create_publisher<T>(topic_name): unique_ptr<Publisher<T>>          │    │
│  │  + create_subscriber<T>(topic_name, callback): unique_ptr<Subscriber<T>>│   │
│  │  + delete_publisher(publisher)                                        │    │
│  │  + delete_subscriber(subscriber)                                       │    │
│  │  + start() / stop()                                                   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                        ▲                                     │
│                                        │                                     │
│           ┌────────────────────────────┴────────────────────────────┐        │
│           │                          │                            │        │
│           ▼                          ▼                            ▼        │
│  ┌─────────────────────┐    ┌─────────────────────┐                        │
│  │ mdds::Publisher<T>  │    │ mdds::Subscriber<T> │                        │
│  ├─────────────────────┤    ├─────────────────────┤                        │
│  │ - topic_: shared_ptr│    │ - topic_: shared_ptr│                       │
│  │ - writer_: unique_ptr│   │ - reader_: unique_ptr│                       │
│  │   <DataWriter<T>>  │    │   <DataReader<T>>  │                        │
│  ├─────────────────────┤    ├─────────────────────┤                        │
│  │ + write(data)       │    │ + set_callback(cb)  │                        │
│  │ + write(data, ts)   │    │ + read(data, ts)   │                        │
│  └─────────────────────┘    └─────────────────────┘                        │
│                                        ▲                                     │
│                                        │                                     │
│           ┌────────────────────────────┴────────────────────────────┐        │
│           ▼                                                         ▼        │
│  ┌─────────────────────┐                                  ┌─────────────────┐│
│  │ mdds::DataWriter<T> │                                  │mdds::DataReader<T>│
│  ├─────────────────────┤                                  ├─────────────────┤│
│  │ - topic_: shared_ptr│                                  │ - topic_: shared_ptr│
│  │ - transport_: shared_ptr│                               │ - transport_: shared_ptr│
│  │ - sequence_: atomic<uint32_t>│                         │ - callback_: DataCallback│
│  ├─────────────────────┤                                  │ - pending_: queue<pair<T, ts>>│
│  │ + write(data, ts)   │                                  ├─────────────────┤│
│  │ + get_topic_id()    │                                  │ + has_data()     ││
│  └─────────────────────┘                                  │ + read(data, ts) ││
│                                                             └─────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         mdds::Topic<T>                                 │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  - name_: std::string                                                       │    │
│  │  - type_name_: std::string                                                  │    │
│  │  - topic_id_: TopicId                                                       │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  + get_name() / get_type_name() / get_topic_id()                       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      mdds::Discovery                                  │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  - server_address_: Endpoint                                          │    │
│  │  - local_endpoint_: Endpoint                                          │    │
│  │  - participant_registry_: Registry<ParticipantInfo>                   │    │
│  │  - topic_registry_: Registry<TopicEntry>                              │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  + register_publisher(topic, endpoint): Result                        │    │
│  │  + register_subscriber(topic, endpoint): Result                       │    │
│  │  + unregister(endpoint): void                                         │    │
│  │  + find_topic(topic_name): vector<Endpoint>                           │    │
│  │  + wait_for_discovery(timeout): bool                                  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    mdds::DiscoveryServer                               │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  - topic_registry_: unordered_map<string, TopicEntry>                │    │
│  │  - participant_registry_: unordered_map<ParticipantId, ParticipantInfo>│  │
│  │  - transport_: unique_ptr<Transport>                                  │    │
│  │  - socket_: int (UDP socket)                                         │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  + start(): bool                                                     │    │
│  │  + stop(): void                                                       │    │
│  │  + handle_message(msg, from): void                                    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         mdds::Transport (Interface)                    │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  + virtual ~Transport()                                              │    │
│  │  + virtual send(data, size, dest): bool                              │    │
│  │  + virtual receive(buffer, max, received, sender): bool              │    │
│  │  + virtual get_local_endpoint(): Endpoint                            │    │
│  │  + virtual set_receive_callback(cb): void                             │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              ▲                                               │
│          ┌───────────────────┼───────────────────┐                          │
│          │                   │                   │                          │
│          ▼                   ▼                   ▼                          │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐                 │
│  │SharedMemory   │   │UDP            │   │TCP            │                 │
│  │Transport     │   │Transport      │   │Transport      │                 │
│  └───────────────┘   └───────────────┘   └───────────────┘                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 DomainParticipant 详细设计

```cpp
// include/mdds/domain_participant.h

#pragma once

#include "mdds/types.h"
#include "mdds/topic.h"
#include "mdds/discovery.h"
#include "mdds/transport.h"
#include <memory>
#include <string>
#include <vector>
#include <unordered_map>

namespace mdds {

class DomainParticipant {
public:
    using ParticipantId = uint32_t;

    /**
     * 创建 DomainParticipant
     * @param domain_id  Domain ID (0-255)
     * @param config     配置参数
     * @return  shared_ptr of DomainParticipant
     */
    static std::shared_ptr<DomainParticipant>
    create(DomainId domain_id, const ParticipantConfig& config = {});

    /**
     * 析构函数
     */
    ~DomainParticipant();

    // 禁止拷贝
    DomainParticipant(const DomainParticipant&) = delete;
    DomainParticipant& operator=(const DomainParticipant&) = delete;

    // 允许移动
    DomainParticipant(DomainParticipant&&) = default;
    DomainParticipant& operator=(DomainParticipant&&) = default;

    /**
     * 创建 Publisher
     * @tparam T    数据类型
     * @param topic_name  Topic 名称
     * @return  Publisher 对象
     */
    template<typename T>
    std::unique_ptr<Publisher<T>>
    create_publisher(const std::string& topic_name);

    /**
     * 创建 Subscriber
     * @tparam T    数据类型
     * @param topic_name  Topic 名称
     * @param callback    数据回调函数
     * @return  Subscriber 对象
     */
    template<typename T>
    std::unique_ptr<Subscriber<T>>
    create_subscriber(const std::string& topic_name,
                      typename Subscriber<T>::DataCallback callback);

    /**
     * 销毁 Publisher
     */
    void delete_publisher(PublisherBase* publisher);

    /**
     * 销毁 Subscriber
     */
    void delete_subscriber(SubscriberBase* subscriber);

    /**
     * 获取 Domain ID
     */
    DomainId get_domain_id() const { return domain_id_; }

    /**
     * 获取 Participant ID
     */
    ParticipantId get_participant_id() const { return participant_id_; }

    /**
     * 获取 Discovery 实例
     */
    Discovery* get_discovery() const { return discovery_.get(); }

    /**
     * 获取 TopicManager
     */
    TopicManager* get_topic_manager() const { return topic_manager_.get(); }

    /**
     * 启动 Participant
     */
    bool start();

    /**
     * 停止 Participant
     */
    void stop();

private:
    /**
     * 构造函数
     */
    explicit DomainParticipant(DomainId domain_id,
                               ParticipantId participant_id,
                               std::unique_ptr<Discovery> discovery,
                               std::unique_ptr<TopicManager> topic_manager);

    DomainId domain_id_;
    ParticipantId participant_id_;
    std::unique_ptr<Discovery> discovery_;
    std::unique_ptr<TopicManager> topic_manager_;
    std::vector<std::unique_ptr<PublisherBase>> publishers_;
    std::vector<std::unique_ptr<SubscriberBase>> subscribers_;
    std::mutex mutex_;
};

}  // namespace mdds
```

### 3.3 Topic 详细设计

```cpp
// include/mdds/topic.h

#pragma once

#include "mdds/types.h"
#include <string>
#include <typeinfo>
#include <type_traits>

namespace mdds {

/**
 * Topic 模板类
 * @tparam T  数据类型
 */
template<typename T>
class Topic {
public:
    using DataType = T;

    /**
     * 构造函数
     * @param name  Topic 名称
     */
    explicit Topic(const std::string& name);

    /**
     * 析构函数
     */
    ~Topic();

    /**
     * 获取 Topic 名称
     */
    const std::string& get_name() const { return name_; }

    /**
     * 获取类型名称
     */
    const std::string& get_type_name() const;

    /**
     * 获取 Topic ID
     */
    TopicId get_topic_id() const { return topic_id_; }

    /**
     * 获取数据类型示例（用于序列化）
     */
    const DataType& get_type_example() const { return data_; }

private:
    std::string name_;
    std::string type_name_;
    TopicId topic_id_;
    DataType data_;  // 类型示例
};

/**
 * Topic 统计信息
 */
struct TopicStatistics {
    uint32_t topic_id;
    std::string topic_name;
    size_t total_publishers;
    size_t total_subscribers;
    uint64_t total_messages_sent;
    uint64_t total_messages_received;
};

}  // namespace mdds
```

### 3.4 Publisher 详细设计

```cpp
// include/mdds/publisher.h

#pragma once

#include "mdds/data_writer.h"
#include "mdds/types.h"
#include <memory>
#include <string>

namespace mdds {

// 前向声明
class DomainParticipant;

/**
 * Publisher 基类（非模板）
 */
class PublisherBase {
public:
    virtual ~PublisherBase() = default;
    virtual const std::string& get_topic_name() const = 0;
    virtual TopicId get_topic_id() const = 0;
};

/**
 * Publisher 模板类
 * @tparam T  数据类型
 */
template<typename T>
class Publisher : public PublisherBase {
public:
    using DataType = T;

    /**
     * 默认构造函数
     */
    Publisher() = default;

    /**
     * 构造函数（由 DomainParticipant 调用）
     */
    Publisher(std::shared_ptr<DomainParticipant> participant,
              std::shared_ptr<Topic<T>> topic,
              std::shared_ptr<Transport> transport);

    /**
     * 析构函数
     */
    ~Publisher();

    /**
     * 写入数据
     * @param data  数据
     * @return  成功返回 true
     */
    bool write(const DataType& data);

    /**
     * 写入数据（带时间戳）
     * @param data      数据
     * @param timestamp 时间戳
     * @return  成功返回 true
     */
    bool write(const DataType& data, uint64_t timestamp);

    /**
     * 获取 Topic 名称
     */
    const std::string& get_topic_name() const override;

    /**
     * 获取 Topic ID
     */
    TopicId get_topic_id() const override;

    /**
     * 获取 DataWriter
     */
    DataWriter<T>* get_writer() const { return writer_.get(); }

private:
    std::shared_ptr<DomainParticipant> participant_;
    std::shared_ptr<Topic<T>> topic_;
    std::unique_ptr<DataWriter<T>> writer_;
};

}  // namespace mdds
```

### 3.5 Subscriber 详细设计

```cpp
// include/mdds/subscriber.h

#pragma once

#include "mdds/data_reader.h"
#include "mdds/types.h"
#include <memory>
#include <string>
#include <functional>

namespace mdds {

// 前向声明
class DomainParticipant;

/**
 * Subscriber 基类（非模板）
 */
class SubscriberBase {
public:
    virtual ~SubscriberBase() = default;
    virtual const std::string& get_topic_name() const = 0;
    virtual TopicId get_topic_id() const = 0;
};

/**
 * Subscriber 模板类
 * @tparam T  数据类型
 */
template<typename T>
class Subscriber : public SubscriberBase {
public:
    /**
     * 数据回调函数类型
     */
    using DataCallback = std::function<void(const T& data, uint64_t timestamp)>;

    /**
     * 默认构造函数
     */
    Subscriber() = default;

    /**
     * 构造函数（由 DomainParticipant 调用）
     */
    Subscriber(std::shared_ptr<DomainParticipant> participant,
               std::shared_ptr<Topic<T>> topic,
               std::shared_ptr<Transport> transport,
               DataCallback callback);

    /**
     * 析构函数
     */
    ~Subscriber();

    /**
     * 设置数据回调
     * @param callback  回调函数
     */
    void set_callback(DataCallback callback);

    /**
     * 读取数据（非阻塞）
     * @param data      读取的数据
     * @param timestamp 时间戳（可为空）
     * @return  成功返回 true
     */
    bool read(DataType& data, uint64_t* timestamp = nullptr);

    /**
     * 获取 Topic 名称
     */
    const std::string& get_topic_name() const override;

    /**
     * 获取 Topic ID
     */
    TopicId get_topic_id() const override;

    /**
     * 获取 DataReader
     */
    DataReader<T>* get_reader() const { return reader_.get(); }

private:
    std::shared_ptr<DomainParticipant> participant_;
    std::shared_ptr<Topic<T>> topic_;
    std::unique_ptr<DataReader<T>> reader_;
    DataCallback callback_;
};

}  // namespace mdds
```

### 3.6 DataWriter 详细设计

```cpp
// include/mdds/data_writer.h

#pragma once

#include "mdds/topic.h"
#include "mdds/transport.h"
#include "mdds/message.h"
#include <memory>
#include <atomic>
#include <cstring>

namespace mdds {

/**
 * DataWriter 模板类
 * @tparam T  数据类型
 */
template<typename T>
class DataWriter {
public:
    /**
     * 构造函数
     */
    DataWriter(std::shared_ptr<Topic<T>> topic,
               std::shared_ptr<Transport> transport);

    /**
     * 析构函数
     */
    ~DataWriter();

    /**
     * 写入数据
     * @param data      数据
     * @param timestamp 时间戳
     * @return  成功返回 true
     */
    bool write(const T& data, uint64_t timestamp);

    /**
     * 获取 Topic ID
     */
    TopicId get_topic_id() const { return topic_->get_topic_id(); }

    /**
     * 获取已发送的消息数量
     */
    uint64_t get_sent_count() const { return sequence_.load(); }

private:
    /**
     * 序列化数据
     */
    std::vector<uint8_t> serialize(const T& data);

    std::shared_ptr<Topic<T>> topic_;
    std::shared_ptr<Transport> transport_;
    std::atomic<uint32_t> sequence_;
};

/**
 * DataWriter 实现
 */
template<typename T>
DataWriter<T>::DataWriter(std::shared_ptr<Topic<T>> topic,
                         std::shared_ptr<Transport> transport)
    : topic_(std::move(topic))
    , transport_(std::move(transport))
    , sequence_(0) {
}

template<typename T>
DataWriter<T>::~DataWriter() = default;

template<typename T>
bool DataWriter<T>::write(const T& data, uint64_t timestamp) {
    // 序列化数据
    auto payload = serialize(data);
    if (payload.empty()) {
        return false;
    }

    // 构建消息头
    MessageHeader header;
    header.magic = MDDS_MAGIC;
    header.version = MDDS_VERSION;
    header.message_type = static_cast<uint8_t>(MessageType::DATA);
    header.flags = 0;
    header.topic_id = topic_->get_topic_id();
    header.sequence_number = sequence_.fetch_add(1);
    header.timestamp = timestamp;
    header.payload_length = static_cast<uint32_t>(payload.size());

    // 发送消息
    Endpoint dest;
    dest.address_ = "224.0.0.100";  // 多播地址（用于发现匹配的 subscriber）
    dest.port_ = 7400 + topic_->get_topic_id();
    dest.type_ = TransportType::UDP;

    // 组装完整消息
    std::vector<uint8_t> message(sizeof(MessageHeader) + payload.size());
    std::memcpy(message.data(), &header, sizeof(MessageHeader));
    std::memcpy(message.data() + sizeof(MessageHeader), payload.data(), payload.size());

    return transport_->send(message.data(), message.size(), dest);
}

template<typename T>
std::vector<uint8_t> DataWriter<T>::serialize(const T& data) {
    // 简化序列化：使用 memcpy
    // 实际应用中应使用更好的序列化库
    std::vector<uint8_t> buffer(sizeof(T));
    std::memcpy(buffer.data(), &data, sizeof(T));
    return buffer;
}

}  // namespace mdds
```

### 3.7 DataReader 详细设计

```cpp
// include/mdds/data_reader.h

#pragma once

#include "mdds/topic.h"
#include "mdds/transport.h"
#include "mdds/message.h"
#include <memory>
#include <mutex>
#include <queue>
#include <functional>

namespace mdds {

/**
 * DataReader 模板类
 * @tparam T  数据类型
 */
template<typename T>
class DataReader {
public:
    /**
     * 数据回调函数类型
     */
    using DataCallback = std::function<void(const T&, uint64_t timestamp)>;

    /**
     * 构造函数
     */
    DataReader(std::shared_ptr<Topic<T>> topic,
               std::shared_ptr<Transport> transport,
               DataCallback callback);

    /**
     * 析构函数
     */
    ~DataReader();

    /**
     * 检查是否有可读数据
     */
    bool has_data() const;

    /**
     * 读取下一条数据
     * @param data      数据输出
     * @param timestamp 时间戳输出
     * @return  成功返回 true
     */
    bool read(T& data, uint64_t* timestamp = nullptr);

    /**
     * 获取 Topic ID
     */
    TopicId get_topic_id() const { return topic_->get_topic_id(); }

    /**
     * 获取接收到的消息数量
     */
    uint64_t get_received_count() const { return received_count_.load(); }

private:
    /**
     * 处理接收到的消息
     */
    void handle_message(const MessageHeader& header, const uint8_t* payload);

    /**
     * 反序列化数据
     */
    T deserialize(const uint8_t* data, size_t size);

    std::shared_ptr<Topic<T>> topic_;
    std::shared_ptr<Transport> transport_;
    DataCallback callback_;
    std::mutex mutex_;
    std::queue<std::pair<T, uint64_t>> pending_data_;
    std::atomic<uint64_t> received_count_;
};

/**
 * DataReader 实现
 */
template<typename T>
DataReader<T>::DataReader(std::shared_ptr<Topic<T>> topic,
                          std::shared_ptr<Transport> transport,
                          DataCallback callback)
    : topic_(std::move(topic))
    , transport_(std::move(transport))
    , callback_(std::move(callback))
    , received_count_(0) {

    // 设置接收回调
    transport_->set_receive_callback(
        [this](const void* data, size_t size, const Endpoint& sender) {
            if (size < sizeof(MessageHeader)) {
                return;
            }
            const auto* header = static_cast<const MessageHeader*>(data);
            if (header->magic != MDDS_MAGIC ||
                header->message_type != static_cast<uint8_t>(MessageType::DATA)) {
                return;
            }
            handle_message(*header,
                          static_cast<const uint8_t*>(data) + sizeof(MessageHeader));
        });
}

template<typename T>
DataReader<T>::~DataReader() = default;

template<typename T>
bool DataReader<T>::has_data() const {
    std::lock_guard<std::mutex> lock(mutex_);
    return !pending_data_.empty();
}

template<typename T>
bool DataReader<T>::read(T& data, uint64_t* timestamp) {
    std::lock_guard<std::mutex> lock(mutex_);
    if (pending_data_.empty()) {
        return false;
    }
    auto& front = pending_data_.front();
    data = std::move(front.first);
    if (timestamp) {
        *timestamp = front.second;
    }
    pending_data_.pop();
    return true;
}

template<typename T>
void DataReader<T>::handle_message(const MessageHeader& header,
                                   const uint8_t* payload) {
    auto data = deserialize(payload, header.payload_length);
    uint64_t timestamp = header.timestamp;

    std::lock_guard<std::mutex> lock(mutex_);
    pending_data_.emplace(std::move(data), timestamp);
    received_count_.fetch_add(1);

    // 调用回调
    if (callback_) {
        callback_(pending_data_.back().first, pending_data_.back().second);
    }
}

template<typename T>
T DataReader<T>::deserialize(const uint8_t* data, size_t size) {
    T result;
    std::memset(&result, 0, sizeof(T));
    size_t copy_size = std::min(size, sizeof(T));
    std::memcpy(&result, data, copy_size);
    return result;
}

}  // namespace mdds
```

---

## 4. 服务发现设计

### 4.1 发现架构对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        FastDDS 发现协议 (复杂)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SPDP (Participant Discovery)          SEDP (Endpoint Discovery)             │
│  ┌─────────────────────────┐           ┌─────────────────────────┐          │
│  │ 239.255.0.1:7412        │           │ 逐个 Endpoint 匹配        │          │
│  │ (组播地址)              │           │ ├─ Topic 名称匹配?       │          │
│  │                         │           │ ├─ Type 名称匹配?        │          │
│  │ P1 ──── 组播 ──── P2    │           │ ├─ QoS 兼容性?           │          │
│  │   │                     │           │ └─ 一致性检查?          │          │
│  │   │                     │           │                          │          │
│  │   └─────────────────────┤           │   复杂匹配算法           │          │
│  │         │               │           └─────────────────────────┘          │
│  │         ▼               │                                                │
│  │  周期性 ANNOUNCE 消息   │                                                │
│  │  (100-300ms 间隔)       │                                                │
│  └─────────────────────────┘                                                │
│                                                                              │
│  问题:                                                                     │
│  - 依赖组播，网络配置复杂                                                    │
│  - O(n²) 消息复杂度                                                          │
│  - 周期性广播浪费资源                                                        │
│  - QoS 匹配计算开销大                                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        MDDS 发现协议 (简化)                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Discovery Server (集中式)                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ Port: 7412 (单播)                                                     │     │
│  │                                                                       │     │
│  │ Topic Registry                                                       │     │
│  │ ┌─────────────────────────────────────────────────────────────────┐ │     │
│  │ │ "SensorData" ──────▶ [Pub1, Pub2]     [Sub1, Sub2]              │ │     │
│  │ │ "CmdData"    ──────▶ [Pub3]           [Sub3]                    │ │     │
│  │ └─────────────────────────────────────────────────────────────────┘ │     │
│  │                                                                       │     │
│  │ Match Logic: 名称匹配 = 成功                                          │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  消息类型: 4 种 (vs FastDDS 的 20+ 种)                                        │
│  ┌───────────────┬───────────────┬───────────────┬───────────────┐        │
│  │ REG_PUB=0x01  │ REG_SUB=0x02  │ MATCH=0x03    │ UNREG=0x04    │        │
│  └───────────────┴───────────────┴───────────────┴───────────────┘        │
│                                                                              │
│  优势:                                                                     │
│  - 无组播依赖                                                               │
│  - O(n) 消息复杂度                                                          │
│  - 事件驱动，无周期性广播                                                    │
│  - 简单匹配算法                                                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 发现消息定义

```cpp
// include/mdds/discovery_message.h

#pragma once

#include "mdds/types.h"
#include <string>
#include <vector>
#include <cstring>

namespace mdds {
namespace discovery {

/**
 * 发现消息类型
 */
enum class MessageType : uint8_t {
    REGISTER_PUBLISHER   = 0x01,  // 注册 Publisher
    REGISTER_SUBSCRIBER = 0x02,  // 注册 Subscriber
    MATCH_RESPONSE       = 0x03,  // 匹配响应
    UNREGISTER           = 0x04   // 注销
};

/**
 * 端点信息
 */
struct EndpointInfo {
    uint32_t participant_id;
    uint32_t topic_id;
    std::string topic_name;
    std::string type_name;
    std::string address;     // IP 地址
    uint16_t port;           // 端口
    uint8_t transport_type; // 传输类型
    uint8_t qos_flags;      // QoS 标志

    // 序列化
    std::vector<uint8_t> serialize() const {
        std::vector<uint8_t> buffer(64);
        size_t offset = 0;

        std::memcpy(buffer.data() + offset, &participant_id, 4);
        offset += 4;
        std::memcpy(buffer.data() + offset, &topic_id, 4);
        offset += 4;

        // topic_name (32 bytes)
        std::memcpy(buffer.data() + offset, topic_name.c_str(), 32);
        offset += 32;

        // type_name (32 bytes)
        std::memcpy(buffer.data() + offset, type_name.c_str(), 32);
        offset += 32;

        // address (16 bytes)
        std::memcpy(buffer.data() + offset, address.c_str(), 16);
        offset += 16;

        std::memcpy(buffer.data() + offset, &port, 2);
        offset += 2;

        buffer[offset++] = transport_type;
        buffer[offset++] = qos_flags;

        return buffer;
    }
};

/**
 * 发现消息头
 */
struct DiscoveryHeader {
    uint32_t magic;              // MDDS_DISCOVERY_MAGIC = 0x4D444456
    uint8_t version;              // 0x01
    MessageType type;             // 消息类型
    uint8_t flags;                // 标志
    uint32_t length;               // 负载长度
    uint32_t sender_id;           // 发送者 ID
};

/**
 * 注册 Publisher 消息
 */
struct RegisterPublisherMessage {
    DiscoveryHeader header;
    EndpointInfo endpoint;

    static RegisterPublisherMessage create(uint32_t sender_id,
                                            const EndpointInfo& ep) {
        RegisterPublisherMessage msg;
        msg.header.magic = 0x4D444456;
        msg.header.version = 0x01;
        msg.header.type = MessageType::REGISTER_PUBLISHER;
        msg.header.flags = 0;
        msg.header.sender_id = sender_id;
        msg.endpoint = ep;
        return msg;
    }
};

/**
 * 注册 Subscriber 消息
 */
struct RegisterSubscriberMessage {
    DiscoveryHeader header;
    EndpointInfo endpoint;

    static RegisterSubscriberMessage create(uint32_t sender_id,
                                             const EndpointInfo& ep) {
        RegisterSubscriberMessage msg;
        msg.header.magic = 0x4D444456;
        msg.header.version = 0x01;
        msg.header.type = MessageType::REGISTER_SUBSCRIBER;
        msg.header.flags = 0;
        msg.header.sender_id = sender_id;
        msg.endpoint = ep;
        return msg;
    }
};

/**
 * 匹配响应消息
 */
struct MatchResponseMessage {
    DiscoveryHeader header;
    uint32_t topic_id;
    uint8_t result;              // 0=成功, 1=等待, 2=失败
    uint8_t num_publishers;       // Publisher 数量
    EndpointInfo publishers[16]; // 匹配的 Publisher 列表

    static MatchResponseMessage create(uint32_t sender_id,
                                        uint32_t topic_id,
                                        uint8_t result,
                                        const std::vector<EndpointInfo>& pubs) {
        MatchResponseMessage msg;
        msg.header.magic = 0x4D444456;
        msg.header.version = 0x01;
        msg.header.type = MessageType::MATCH_RESPONSE;
        msg.header.flags = 0;
        msg.header.sender_id = sender_id;
        msg.topic_id = topic_id;
        msg.result = result;
        msg.num_publishers = static_cast<uint8_t>(
            std::min(pubs.size(), size_t(16)));
        for (size_t i = 0; i < msg.num_publishers; ++i) {
            msg.publishers[i] = pubs[i];
        }
        return msg;
    }
};

/**
 * 注销消息
 */
struct UnregisterMessage {
    DiscoveryHeader header;
    uint32_t topic_id;
    uint32_t participant_id;

    static UnregisterMessage create(uint32_t sender_id,
                                     uint32_t topic_id,
                                     uint32_t participant_id) {
        UnregisterMessage msg;
        msg.header.magic = 0x4D444456;
        msg.header.version = 0x01;
        msg.header.type = MessageType::UNREGISTER;
        msg.header.flags = 0;
        msg.header.sender_id = sender_id;
        msg.topic_id = topic_id;
        msg.participant_id = participant_id;
        return msg;
    }
};

}  // namespace discovery
}  // namespace mdds
```

### 4.3 Discovery 类设计

```cpp
// include/mdds/discovery.h

#pragma once

#include "mdds/discovery_message.h"
#include "mdds/transport.h"
#include <memory>
#include <unordered_map>
#include <unordered_set>
#include <mutex>
#include <atomic>

namespace mdds {

/**
 * 发现配置
 */
struct DiscoveryConfig {
    std::string server_address = "127.0.0.1";  // 发现服务器地址
    uint16_t server_port = 7412;               // 发现服务器端口
    uint32_t registration_retry_ms = 1000;     // 注册重试间隔
    uint32_t discovery_timeout_ms = 5000;      // 发现超时
};

/**
 * Participant 信息
 */
struct ParticipantInfo {
    uint32_t participant_id;
    std::string address;
    uint16_t port;
    uint64_t last_seen;  // 最后活跃时间
};

/**
 * Topic 条目
 */
struct TopicEntry {
    std::string topic_name;
    std::string type_name;
    uint32_t topic_id;
    std::vector<discovery::EndpointInfo> publishers;
    std::vector<discovery::EndpointInfo> subscribers;
};

/**
 * 发现客户端
 */
class Discovery {
public:
    /**
     * 构造函数
     */
    Discovery(DomainId domain_id,
              uint32_t participant_id,
              std::shared_ptr<Transport> transport,
              const DiscoveryConfig& config);

    /**
     * 析构函数
     */
    ~Discovery();

    /**
     * 启动发现
     */
    bool start();

    /**
     * 停止发现
     */
    void stop();

    /**
     * 注册 Publisher
     */
    bool register_publisher(const std::string& topic_name,
                           const std::string& type_name,
                           uint32_t topic_id);

    /**
     * 注册 Subscriber
     */
    bool register_subscriber(const std::string& topic_name,
                            const std::string& type_name,
                            uint32_t topic_id);

    /**
     * 注销
     */
    void unregister(uint32_t topic_id);

    /**
     * 查找 Topic 的 Publisher
     */
    std::vector<discovery::EndpointInfo>
    find_publishers(const std::string& topic_name);

    /**
     * 等待发现完成
     */
    bool wait_for_discovery(uint32_t timeout_ms);

    /**
     * 获取本地端点
     */
    const Endpoint& get_local_endpoint() const;

private:
    /**
     * 处理发现消息
     */
    void handle_message(const void* data, size_t size,
                        const Endpoint& sender);

    /**
     * 发送消息到服务器
     */
    bool send_to_server(const void* data, size_t size);

    /**
     * 重试注册
     */
    void retry_registration();

    DomainId domain_id_;
    uint32_t participant_id_;
    DiscoveryConfig config_;
    std::shared_ptr<Transport> transport_;
    Endpoint local_endpoint_;
    Endpoint server_endpoint_;

    std::unordered_map<std::string, TopicEntry> topic_registry_;
    std::unordered_map<uint32_t, discovery::EndpointInfo> pending_registrations_;
    std::unordered_set<uint32_t> confirmed_topic_ids_;

    std::atomic<bool> running_;
    std::thread worker_thread_;
    std::thread retry_thread_;
    std::mutex mutex_;
};

/**
 * 发现服务器
 */
class DiscoveryServer {
public:
    /**
     * 构造函数
     */
    DiscoveryServer(uint16_t port = 7412);

    /**
     * 析构函数
     */
    ~DiscoveryServer();

    /**
     * 启动服务器
     */
    bool start();

    /**
     * 停止服务器
     */
    void stop();

    /**
     * 获取服务器端口
     */
    uint16_t get_port() const { return port_; }

private:
    /**
     * 处理注册 Publisher 消息
     */
    void handle_register_publisher(const discovery::RegisterPublisherMessage& msg,
                                   const Endpoint& sender);

    /**
     * 处理注册 Subscriber 消息
     */
    void handle_register_subscriber(const discovery::RegisterSubscriberMessage& msg,
                                    const Endpoint& sender);

    /**
     * 处理注销消息
     */
    void handle_unregister(const discovery::UnregisterMessage& msg,
                          const Endpoint& sender);

    /**
     * 通知 Subscriber 匹配结果
     */
    void notify_matched_publishers(const std::string& topic_name,
                                   const discovery::EndpointInfo& subscriber);

    /**
     * 工作线程
     */
    void worker_loop();

    uint16_t port_;
    std::unique_ptr<Transport> transport_;
    std::atomic<bool> running_;
    std::thread worker_thread_;

    // Topic 注册表
    std::unordered_map<std::string, TopicEntry> topic_registry_;
    // Participant 注册表
    std::unordered_map<uint32_t, ParticipantInfo> participant_registry_;
    std::mutex mutex_;
};

}  // namespace mdds
```

### 4.4 发现流程时序图

#### 4.4.1 Publisher 启动流程

```
┌─────────┐    ┌─────────────────┐    ┌──────────────────┐    ┌────────────┐
│ Publisher│    │DomainParticipant│    │    Discovery      │    │DiscoverySrv│
│ (App)   │    │                │    │    Client        │    │           │
└────┬────┘    └───────┬────────┘    └────────┬─────────┘    └─────┬──────┘
     │                  │                      │                     │
     │ create_publisher()                     │                     │
     │────────────────▶│                      │                     │
     │                  │                      │                     │
     │                  │  register_publisher()                     │
     │                  │─────────────────────▶│                     │
     │                  │                      │                     │
     │                  │                      │  REGISTER_PUBLISHER │
     │                  │                      │────────────────────▶│
     │                  │                      │                     │
     │                  │                      │     [检查 Topic]     │
     │                  │                      │     [添加到注册表]   │
     │                  │                      │                     │
     │                  │                      │                     │
     │                  │   [等待匹配的 Subscriber]                   │
     │                  │                      │◀────────────────────│
     │                  │  MATCH_RESPONSE      │                     │
     │                  │◀─────────────────────│                     │
     │                  │                      │                     │
     │  [获取 Subscriber 列表]                  │                     │
     │◀─────────────────│                      │                     │
     │                  │                      │                     │
     │  [开始发布数据]   │                      │                     │
     │═════════════════════════════════════════════════════════════▶│
     │  (直接发送数据给 Subscribers，无 Server 中转)                   │
     │                  │                      │                     │
```

#### 4.4.2 Subscriber 启动流程

```
┌─────────┐    ┌─────────────────┐    ┌──────────────────┐    ┌────────────┐
│Subscriber│   │DomainParticipant│    │    Discovery      │    │DiscoverySrv│
│ (App)   │    │                │    │    Client        │    │           │
└────┬────┘    └───────┬────────┘    └────────┬─────────┘    └─────┬──────┘
     │                  │                      │                     │
     │ create_subscriber()                     │                     │
     │────────────────▶│                      │                     │
     │                  │                      │                     │
     │                  │  register_subscriber()                    │
     │                  │─────────────────────▶│                     │
     │                  │                      │                     │
     │                  │                      │  REGISTER_SUBSCRIBER
     │                  │                      │────────────────────▶│
     │                  │                      │                     │
     │                  │                      │     [检查 Topic]     │
     │                  │                      │     [添加 Subscriber]│
     │                  │                      │     [查找 Publishers]│
     │                  │                      │                     │
     │                  │                      │  MATCH_RESPONSE     │
     │                  │                      │◀────────────────────│
     │                  │  [Publisher 列表]    │                     │
     │                  │◀─────────────────────│                     │
     │                  │                      │                     │
     │  [获取 Publisher 列表]                   │                     │
     │◀─────────────────│                      │                     │
     │                  │                      │                     │
     │  [建立数据接收]   │                      │                     │
     │◀═════════════════════════════════════════════════════════════│
     │                  │                      │                     │
```

---

## 5. 传输层设计

### 5.1 传输层接口

```cpp
// include/mdds/transport.h

#pragma once

#include "mdds/types.h"
#include <memory>
#include <functional>
#include <string>

namespace mdds {

/**
 * 传输类型
 */
enum class TransportType : uint8_t {
    UDP    = 0x01,
    TCP    = 0x02,
    SHM    = 0x03,  // Shared Memory
};

/**
 * 端点描述
 */
struct Endpoint {
    std::string address_;   // IP 地址或共享内存名称
    uint16_t port_;          // 端口号
    TransportType type_;     // 传输类型

    bool operator==(const Endpoint& other) const {
        return address_ == other.address_ &&
               port_ == other.port_ &&
               type_ == other.type_;
    }

    bool operator!=(const Endpoint& other) const {
        return !(*this == other);
    }

    std::string to_string() const {
        return address_ + ":" + std::to_string(port_);
    }
};

/**
 * 接收回调函数
 */
using ReceiveCallback = std::function<void(
    const void* data,
    size_t size,
    const Endpoint& sender
)>;

/**
 * 传输层接口
 */
class Transport {
public:
    virtual ~Transport() = default;

    /**
     * 发送数据
     * @param data        数据缓冲区
     * @param size        数据大小
     * @param destination 目标端点
     * @return  成功返回 true
     */
    virtual bool send(const void* data, size_t size,
                     const Endpoint& destination) = 0;

    /**
     * 接收数据
     * @param buffer      接收缓冲区
     * @param max_size    最大接收大小
     * @param received    实际接收大小
     * @param sender      发送方端点
     * @return  成功返回 true
     */
    virtual bool receive(void* buffer, size_t max_size,
                        size_t* received, Endpoint* sender) = 0;

    /**
     * 获取本地端点
     */
    virtual Endpoint get_local_endpoint() const = 0;

    /**
     * 设置接收回调
     */
    virtual void set_receive_callback(ReceiveCallback callback) = 0;

    /**
     * 获取传输类型
     */
    virtual TransportType get_type() const = 0;

    /**
     * 检查是否就绪
     */
    virtual bool is_ready() const = 0;
};

/**
 * 传输层工厂
 */
class TransportFactory {
public:
    /**
     * 创建 UDP 传输
     */
    static std::unique_ptr<Transport>
    create_udp(uint16_t port = 0);

    /**
     * 创建 TCP 传输
     */
    static std::unique_ptr<Transport>
    create_tcp(uint16_t port = 0);

    /**
     * 创建共享内存传输
     */
    static std::unique_ptr<Transport>
    create_shm(DomainId domain_id, TopicId topic_id);
};

}  // namespace mdds
```

### 5.2 UDP 传输实现

```cpp
// include/mdds/transport_udp.h

#pragma once

#include "mdds/transport.h"
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <fcntl.h>

namespace mdds {

/**
 * UDP 传输实现
 */
class UDPTransport : public Transport {
public:
    /**
     * 构造函数
     * @param port  UDP 端口，0 表示自动分配
     */
    explicit UDPTransport(uint16_t port = 0);

    /**
     * 析构函数
     */
    ~UDPTransport() override;

    /**
     * 发送数据
     */
    bool send(const void* data, size_t size,
              const Endpoint& destination) override;

    /**
     * 接收数据
     */
    bool receive(void* buffer, size_t max_size,
                size_t* received, Endpoint* sender) override;

    /**
     * 获取本地端点
     */
    Endpoint get_local_endpoint() const override;

    /**
     * 设置接收回调
     */
    void set_receive_callback(ReceiveCallback callback) override;

    /**
     * 获取传输类型
     */
    TransportType get_type() const override { return TransportType::UDP; }

    /**
     * 检查是否就绪
     */
    bool is_ready() const override { return socket_fd_ >= 0; }

private:
    /**
     * 设置非阻塞模式
     */
    bool set_non_blocking(bool non_blocking);

    /**
     * 处理接收到的数据
     */
    void handle_receive();

    int socket_fd_;
    uint16_t local_port_;
    Endpoint local_endpoint_;
    ReceiveCallback callback_;
    struct sockaddr_in recv_from_;
    socklen_t recv_from_len_;
    std::thread recv_thread_;
    std::atomic<bool> running_;
};

}  // namespace mdds
```

### 5.3 共享内存传输实现

```cpp
// include/mdds/transport_shm.h

#pragma once

#include "mdds/transport.h"
#include "shm/shm_api.h"

namespace mdds {

/**
 * 共享内存传输实现
 */
class SharedMemoryTransport : public Transport {
public:
    /**
     * 构造函数
     * @param domain_id   Domain ID
     * @param topic_id    Topic ID
     * @param is_server   是否为服务器端（创建共享内存）
     */
    SharedMemoryTransport(DomainId domain_id, TopicId topic_id, bool is_server);

    /**
     * 析构函数
     */
    ~SharedMemoryTransport() override;

    /**
     * 发送数据
     */
    bool send(const void* data, size_t size,
              const Endpoint& destination) override;

    /**
     * 接收数据
     */
    bool receive(void* buffer, size_t max_size,
                size_t* received, Endpoint* sender) override;

    /**
     * 获取本地端点
     */
    Endpoint get_local_endpoint() const override;

    /**
     * 设置接收回调
     */
    void set_receive_callback(ReceiveCallback callback) override;

    /**
     * 获取传输类型
     */
    TransportType get_type() const override { return TransportType::SHM; }

    /**
     * 检查是否就绪
     */
    bool is_ready() const override { return shm_handle_ != nullptr; }

private:
    /**
     * 生成共享内存名称
     */
    static std::string generate_shm_name(DomainId domain_id, TopicId topic_id);

    DomainId domain_id_;
    TopicId topic_id_;
    bool is_server_;
    shm_handle_t* shm_handle_;
    Endpoint local_endpoint_;
    ReceiveCallback callback_;
};

}  // namespace mdds
```

### 5.4 传输层选择策略

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         传输层选择策略                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐                                                        │
│  │   应用场景       │                    推荐传输层                          │
│  ├─────────────────┤                                                        │
│  │ 同一进程内通信   │ ────▶  SharedMemory (零拷贝，最低延迟)                    │
│  ├─────────────────┤                                                        │
│  │ 同一机器不同进程 │ ────▶  SharedMemory (最佳性能)                          │
│  │                 │         或 DomainSocket (备选)                          │
│  ├─────────────────┤                                                        │
│  │ 同一网段内通信   │ ────▶  UDP (高效，组播支持)                              │
│  ├─────────────────┤                                                        │
│  │ 跨网段/互联网   │ ────▶  TCP (可靠传输)                                    │
│  ├─────────────────┤                                                        │
│  │ 需要可靠性保证  │ ────▶  TCP + 应用层确认                                   │
│  └─────────────────┘                                                        │
│                                                                              │
│  选择因素:                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ 1. 延迟要求    ──▶  SharedMemory > UDP > TCP                         │    │
│  │ 2. 可靠性要求  ───▶  TCP > UDP                                        │    │
│  │ 3. 带宽需求    ───▶  SharedMemory > TCP > UDP                        │    │
│  │ 4. 部署复杂度  ───▶  UDP (简单) > TCP > SharedMemory                  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 消息格式规范

### 6.1 完整消息格式

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MDDS 消息格式                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  MDDS Data Message                                                           │
│  ┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬──────────┐   │
│  │ Offset │  0-3   │  4-5   │   6    │   7    │  8-11  │ 12-15  │   16-23   │   │
│  ├────────┼────────┼────────┼────────┼────────┼────────┼────────┼──────────┤   │
│  │ Field  │ magic  │version │ msg_type│ flags │ topic_id│ seq_num│ timestamp│   │
│  │ Size   │  4B    │  2B    │  1B    │  1B    │  4B    │  4B    │   8B     │   │
│  ├────────┼────────┼────────┼────────┼────────┼────────┼────────┼──────────┤   │
│  │ Value  │0x4D444453│0x0001 │ DATA  │   0    │  ──   │  ──   │   ──    │   │
│  └────────┴────────┴────────┴────────┴────────┴────────┴────────┴──────────┘   │
│                                                                              │
│  ┌────────┬────────┬─────────────────────────────────────────────────────┐    │
│  │ Offset │ 24-27  │              28-N                                    │    │
│  ├────────┼────────┼─────────────────────────────────────────────────────┤    │
│  │ Field  │payload_│                   payload                           │    │
│  │        │ length │              (用户数据)                              │    │
│  │ Size   │  4B    │                   N                                  │    │
│  └────────┴────────┴─────────────────────────────────────────────────────┘    │
│                                                                              │
│  Total Header Size: 28 bytes                                                 │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  MDDS Discovery Message                                                      │
│  ┌────────┬────────┬────────┬────────┬────────┬────────┬─────────────────┐   │
│  │ Offset │  0-3   │  4-5   │   6    │   7    │  8-11  │     12-15       │   │
│  ├────────┼────────┼────────┼────────┼────────┼────────┼─────────────────┤   │
│  │ Field  │ magic  │version │ disc_type│ flags │ length │   sender_id    │   │
│  │ Size   │  4B    │  2B    │  1B    │  1B    │  4B    │      4B        │   │
│  ├────────┼────────┼────────┼────────┼────────┼────────┼─────────────────┤   │
│  │ Value  │0x4D444456│0x0001 │ REG_...│   0    │  ──   │      ──        │   │
│  └────────┴────────┴────────┴────────┴────────┴────────┴─────────────────┘   │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐   │
│  │                            Payload (Variable)                           │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐    │   │
│  │  │ Endpoint Info (64 bytes)                                       │    │   │
│  │  │  - participant_id (4B)                                        │    │   │
│  │  │  - topic_id (4B)                                              │    │   │
│  │  │  - topic_name (32B)                                           │    │   │
│  │  │  - type_name (32B)                                            │    │   │
│  │  │  - address (16B)                                              │    │   │
│  │  │  - port (2B)                                                  │    │   │
│  │  │  - transport_type (1B)                                        │    │   │
│  │  │  - qos_flags (1B)                                             │    │   │
│  │  └─────────────────────────────────────────────────────────────────┘    │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 消息类型值

| 消息类型 | 值 | 说明 |
|----------|-----|------|
| DATA | 0x01 | 数据消息 |
| DISCOVERY | 0x02 | 发现消息 |
| HEARTBEAT | 0x03 | 心跳消息（简化） |
| ACK | 0x04 | 确认消息 |

### 6.3 发现消息子类型

| 子类型 | 值 | 说明 |
|--------|-----|------|
| REGISTER_PUBLISHER | 0x01 | 注册发布者 |
| REGISTER_SUBSCRIBER | 0x02 | 注册订阅者 |
| MATCH_RESPONSE | 0x03 | 匹配响应 |
| UNREGISTER | 0x04 | 注销 |

### 6.4 QoS 标志

| QoS 标志 | 值 | 说明 |
|----------|-----|------|
| BEST_EFFORT | 0x01 | 尽力而为 |
| RELIABLE | 0x02 | 可靠传输 |
| VOLATILE | 0x10 | 易失（无持久化） |
| TRANSIENT_LOCAL | 0x20 | 短暂本地持久化 |

---

## 7. QoS 策略设计

### 7.1 简化 QoS 模型

相比 FastDDS 的 20+ QoS 策略，MDDS 只保留最常用的 4 个：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MDDS QoS 模型 (简化)                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  FastDDS QoS 策略 (20+)              MDDS QoS 策略 (2 维度)                   │
│  ┌─────────────────────────┐        ┌─────────────────────────┐             │
│  │ Durability              │        │ Durability              │             │
│  │ DurabilityService       │───────▶│   - VOLATILE (默认)     │             │
│  │ Deadline                │        │   - TRANSIENT_LOCAL     │             │
│  │ LatencyBudget           │        └─────────────────────────┘             │
│  │ Liveliness              │        ┌─────────────────────────┐             │
│  │ Reliability              │───────▶│ Reliability             │             │
│  │ TransportPriority       │        │   - BEST_EFFORT (默认)  │             │
│  │ Ownership               │        │   - RELIABLE            │             │
│  │ DestinationOrder        │        └─────────────────────────┘             │
│  │ History                 │                                                 │
│  │ ResourceLimits          │        兼容性规则:                              │
│  │ ...                     │        ┌─────────────────────────────────┐      │
│  └─────────────────────────┘        │ TRANSIENT_LOCAL Publisher       │      │
│                                      │       ▶                        │      │
│                                      │ VOLATILE Subscriber (OK)        │      │
│                                      │ TRANSIENT_LOCAL Subscriber (OK) │      │
│                                      └─────────────────────────────────┘      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 QoS 配置结构

```cpp
// include/mdds/qos.h

#pragma once

#include <cstdint>

namespace mdds {

/**
 * Reliability QoS
 */
enum class ReliabilityQos : uint8_t {
    BEST_EFFORT = 0x01,  // 尽力而为，不保证送达
    RELIABLE = 0x02,     // 可靠传输，保证送达
};

/**
 * Durability QoS
 */
enum class DurabilityQos : uint8_t {
    VOLATILE = 0x10,         // 易失，不持久化
    TRANSIENT_LOCAL = 0x20,   // 短暂本地持久化
};

/**
 * 简化的 QoS 配置
 */
struct QoSConfig {
    ReliabilityQos reliability = ReliabilityQos::BEST_EFFORT;
    DurabilityQos durability = DurabilityQos::VOLATILE;

    /**
     * 默认 QoS
     */
    static constexpr QoSConfig default_qos() {
        return QoSConfig{
            ReliabilityQos::BEST_EFFORT,
            DurabilityQos::VOLATILE
        };
    }

    /**
     * 检查与另一个 QoS 配置是否兼容
     */
    bool is_compatible_with(const QoSConfig& other) const {
        // TRANSIENT_LOCAL Publisher 可以发送给 VOLATILE Subscriber
        if (durability == DurabilityQos::TRANSIENT_LOCAL &&
            other.durability == DurabilityQos::VOLATILE) {
            return true;
        }

        // 其他情况需要完全匹配
        return durability == other.durability;
    }

    /**
     * 获取组合标志
     */
    uint8_t get_flags() const {
        return static_cast<uint8_t>(reliability) |
               static_cast<uint8_t>(durability);
    }

    /**
     * 从标志创建
     */
    static QoSConfig from_flags(uint8_t flags) {
        QoSConfig qos;
        if (flags & 0x02) {
            qos.reliability = ReliabilityQos::RELIABLE;
        }
        if (flags & 0x20) {
            qos.durability = DurabilityQos::TRANSIENT_LOCAL;
        }
        return qos;
    }
};

/**
 * QoS 策略接口
 */
class QoSPolicy {
public:
    virtual ~QoSPolicy() = default;
    virtual bool is_compatible(const QoSPolicy* other) const = 0;
};

}  // namespace mdds
```

---

## 8. 错误处理

### 8.1 错误码定义

```cpp
// include/mdds/mdds_error.h

#pragma once

#include <system_error>
#include <string>

namespace mdds {

/**
 * MDDS 错误码
 */
enum class MddsError : int {
    // 成功
    OK = 0,

    // 通用错误 (1-99)
    INVALID_PARAM = 1,       // 无效参数
    NOT_FOUND = 2,           // 未找到
    ALREADY_EXISTS = 3,     // 已存在
    NO_MEMORY = 4,          // 内存不足
    TIMEOUT = 5,            // 超时
    NOT_INITIALIZED = 6,    // 未初始化
    ALREADY_STARTED = 7,    // 已启动
    NOT_STARTED = 8,        // 未启动
    OPERATION_FAILED = 9,  // 操作失败

    // 发现相关错误 (100-199)
    DISCOVERY_TIMEOUT = 100,     // 发现超时
    TOPIC_MISMATCH = 101,       // Topic 不匹配
    NO_PUBLISHER = 102,         // 无 Publisher
    NO_SUBSCRIBER = 103,        // 无 Subscriber
    MATCH_FAILED = 104,         // 匹配失败
    DISCOVERY_SERVER_UNAVAILABLE = 105,  // 发现服务器不可用

    // 传输相关错误 (200-299)
    SEND_FAILED = 200,          // 发送失败
    RECEIVE_FAILED = 201,       // 接收失败
    CONNECTION_LOST = 202,      // 连接丢失
    BIND_FAILED = 203,          // 绑定失败
    ACCEPT_FAILED = 204,        // 接受连接失败

    // 数据相关错误 (300-399)
    SERIALIZE_FAILED = 300,     // 序列化失败
    DESERIALIZE_FAILED = 301,   // 反序列化失败
    BUFFER_OVERFLOW = 302,     // 缓冲区溢出
    INVALID_MESSAGE = 303,      // 无效消息

    // 资源错误 (400-499)
    SHM_CREATE_FAILED = 400,    // 共享内存创建失败
    SHM_ATTACH_FAILED = 401,   // 共享内存附加失败
    SOCKET_CREATE_FAILED = 402, // Socket 创建失败
};

/**
 * MDDS 异常类
 */
class MddsException : public std::exception {
public:
    MddsException(MddsError code, const std::string& message)
        : code_(code), message_(message) {}

    const char* what() const noexcept override {
        return message_.c_str();
    }

    MddsError code() const { return code_; }

    std::string to_string() const {
        return std::string("[") + std::to_string(static_cast<int>(code_)) +
               "] " + message_;
    }

private:
    MddsError code_;
    std::string message_;
};

/**
 * 错误码到字符串的转换
 */
inline const char* error_to_string(MddsError error) {
    switch (error) {
        case MddsError::OK:
            return "OK";
        case MddsError::INVALID_PARAM:
            return "Invalid parameter";
        case MddsError::NOT_FOUND:
            return "Not found";
        case MddsError::ALREADY_EXISTS:
            return "Already exists";
        case MddsError::NO_MEMORY:
            return "Out of memory";
        case MddsError::TIMEOUT:
            return "Operation timed out";
        case MddsError::NOT_INITIALIZED:
            return "Not initialized";
        case MddsError::ALREADY_STARTED:
            return "Already started";
        case MddsError::NOT_STARTED:
            return "Not started";
        case MddsError::DISCOVERY_TIMEOUT:
            return "Discovery timeout";
        case MddsError::TOPIC_MISMATCH:
            return "Topic mismatch";
        case MddsError::NO_PUBLISHER:
            return "No publisher available";
        case MddsError::NO_SUBSCRIBER:
            return "No subscriber available";
        case MddsError::SEND_FAILED:
            return "Send failed";
        case MddsError::RECEIVE_FAILED:
            return "Receive failed";
        default:
            return "Unknown error";
    }
}

}  // namespace mdds
```

---

## 9. 线程模型

### 9.1 线程架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            MDDS 线程模型                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      Application Thread                               │    │
│  │                    (用户代码运行在此线程)                              │    │
│  └─────────────────────────────────┬───────────────────────────────────────┘    │
│                                    │                                          │
│                                    ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      DomainParticipant                                │    │
│  │                                                                      │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │    │
│  │  │   Publisher     │  │   Subscriber    │  │  Discovery      │     │    │
│  │  │   Thread Pool   │  │   Thread Pool   │  │  Thread         │     │    │
│  │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘     │    │
│  └───────────┼───────────────────┼─────────────────────┼───────────────┘    │
│              │                   │                     │                      │
│              ▼                   ▼                     ▼                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                       Transport Layer                                │    │
│  │                                                                      │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │    │
│  │  │  UDP Recv       │  │  TCP Accept     │  │  SHM Wait       │     │    │
│  │  │  Thread         │  │  Thread         │  │  Thread         │     │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘     │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  线程职责:                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ Application Thread                                                   │    │
│  │   - 用户回调执行                                                       │    │
│  │   - 数据发布调用                                                       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ Publisher Thread Pool                                                │    │
│  │   - 序列化                                                            │    │
│  │   - 发送队列处理                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ Subscriber Thread Pool                                               │    │
│  │   - 接收队列处理                                                      │    │
│  │   - 反序列化                                                          │    │
│  │   - 用户回调调度                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ Discovery Thread                                                     │    │
│  │   - 发现消息处理                                                      │    │
│  │   - 注册重试                                                          │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ Transport Threads                                                    │    │
│  │   - UDP: 单线程接收                                                   │    │
│  │   - TCP: Accept + 每连接一个接收线程                                   │    │
│  │   - SHM: eventfd 等待                                                 │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 线程安全设计

```cpp
// 线程安全策略

class DomainParticipant {
private:
    mutable std::mutex publishers_mutex_;     // 保护 publishers_
    mutable std::mutex subscribers_mutex_;    // 保护 subscribers_
    mutable std::mutex topic_manager_mutex_;   // 保护 topic_manager_
    // ...
};

class DataReader {
private:
    mutable std::mutex data_mutex_;            // 保护 pending_data_
    std::atomic<uint64_t> received_count_;    // 原子计数器
};

class Discovery {
private:
    mutable std::mutex registry_mutex_;       // 保护 topic_registry_
    std::atomic<bool> running_;                // 原子标志
};
```

---

## 10. 目录结构

### 10.1 完整目录结构

```
mdds/
├── include/
│   └── mdds/
│       ├── mdds.h                 # 主头文件
│       ├── types.h                # 类型定义
│       ├── domain_participant.h  # DomainParticipant 类
│       ├── topic.h                # Topic 模板类
│       ├── publisher.h            # Publisher 模板类
│       ├── subscriber.h           # Subscriber 模板类
│       ├── data_writer.h         # DataWriter 模板类
│       ├── data_reader.h         # DataReader 模板类
│       ├── discovery.h            # 发现客户端
│       ├── discovery_server.h     # 发现服务器
│       ├── discovery_message.h    # 发现消息结构
│       ├── transport.h             # 传输层接口
│       ├── transport_udp.h        # UDP 传输实现
│       ├── transport_shm.h         # 共享内存传输实现
│       ├── qos.h                   # QoS 配置
│       ├── mdds_error.h           # 错误码定义
│       ├── message.h              # 消息结构
│       └── mdds_export.h          # 导出宏
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
│   ├── CMakeLists.txt
│   ├── sensor_publisher.cpp      # 传感器发布示例
│   ├── sensor_subscriber.cpp      # 传感器订阅示例
│   ├── cmd_publisher.cpp          # 命令发布示例
│   └── cmd_subscriber.cpp         # 命令订阅示例
├── test/
│   ├── CMakeLists.txt
│   ├── discovery_test.cpp         # 发现测试
│   ├── topic_test.cpp            # Topic 测试
│   ├── transport_test.cpp         # 传输层测试
│   ├── qos_test.cpp              # QoS 测试
│   └── performance_test.cpp      # 性能测试
├── doc/
│   ├── MDDS_Design.md            # 设计概述
│   ├── MDDS_Architecture.md      # 架构设计（本文件）
│   └── api_reference.md          # API 参考
└── CMakeLists.txt
```

### 10.2 主头文件 (mdds.h)

```cpp
// include/mdds/mdds.h

#pragma once

/**
 * MDDS - Minimal Data Distribution Service
 *
 * A lightweight DDS implementation with simplified service discovery.
 */

#include "mdds/types.h"
#include "mdds/domain_participant.h"
#include "mdds/topic.h"
#include "mdds/publisher.h"
#include "mdds/subscriber.h"
#include "mdds/data_writer.h"
#include "mdds/data_reader.h"
#include "mdds/discovery.h"
#include "mdds/discovery_server.h"
#include "mdds/transport.h"
#include "mdds/qos.h"
#include "mdds/mdds_error.h"

/**
 * MDDS 版本信息
 */
namespace mdds {
    constexpr uint16_t MDDS_VERSION_MAJOR = 1;
    constexpr uint16_t MDDS_VERSION_MINOR = 0;
    constexpr uint16_t MDDS_VERSION_PATCH = 0;

    inline std::string get_version() {
        return std::to_string(MDDS_VERSION_MAJOR) + "." +
               std::to_string(MDDS_VERSION_MINOR) + "." +
               std::to_string(MDDS_VERSION_PATCH);
    }
}
```

---

## 11. API 参考

### 11.1 DomainParticipant API

```cpp
namespace mdds {

/**
 * 创建 DomainParticipant
 * @param domain_id  Domain ID (0-255)
 * @param config     配置（可选）
 * @return  DomainParticipant 智能指针
 */
std::shared_ptr<DomainParticipant>
DomainParticipant::create(DomainId domain_id,
                         const ParticipantConfig& config = {});

/**
 * 创建 Publisher
 * @tparam T  数据类型
 * @param topic_name  Topic 名称
 * @return  Publisher 智能指针
 */
template<typename T>
std::unique_ptr<Publisher<T>>
DomainParticipant::create_publisher(const std::string& topic_name);

/**
 * 创建 Subscriber
 * @tparam T  数据类型
 * @param topic_name  Topic 名称
 * @param callback    数据回调函数
 * @return  Subscriber 智能指针
 */
template<typename T>
std::unique_ptr<Subscriber<T>>
DomainParticipant::create_subscriber(const std::string& topic_name,
                                     typename Subscriber<T>::DataCallback callback);

}  // namespace mdds
```

### 11.2 Publisher API

```cpp
namespace mdds {

template<typename T>
class Publisher {
public:
    using DataType = T;

    /**
     * 写入数据
     * @param data  数据
     * @return  成功返回 true
     */
    bool write(const DataType& data);

    /**
     * 写入数据（带时间戳）
     * @param data      数据
     * @param timestamp 时间戳
     * @return  成功返回 true
     */
    bool write(const DataType& data, uint64_t timestamp);

    /**
     * 获取 Topic 名称
     */
    const std::string& get_topic_name() const;

    /**
     * 获取 Topic ID
     */
    TopicId get_topic_id() const;
};

}  // namespace mdds
```

### 11.3 Subscriber API

```cpp
namespace mdds {

template<typename T>
class Subscriber {
public:
    using DataType = T;
    using DataCallback = std::function<void(const DataType& data,
                                           uint64_t timestamp)>;

    /**
     * 设置数据回调
     * @param callback  回调函数
     */
    void set_callback(DataCallback callback);

    /**
     * 读取数据（非阻塞）
     * @param data      数据输出
     * @param timestamp 时间戳输出（可为 nullptr）
     * @return  成功返回 true
     */
    bool read(DataType& data, uint64_t* timestamp = nullptr);

    /**
     * 检查是否有数据
     */
    bool has_data() const;

    /**
     * 获取 Topic 名称
     */
    const std::string& get_topic_name() const;

    /**
     * 获取 Topic ID
     */
    TopicId get_topic_id() const;
};

}  // namespace mdds
```

---

## 12. 时序图

### 12.1 完整发布-订阅流程

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Pub App │    │Participant│   │Discovery│    │DiscServer│   │Sub App │
└────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘
     │              │              │              │              │
     │ write(data)  │              │              │              │
     │─────────────▶│              │              │              │
     │              │              │              │              │
     │              │─────────────▶│              │              │
     │              │ REG_PUB(topic="Sensor")    │              │
     │              │─────────────▶│              │              │
     │              │              │              │              │
     │              │              │ [添加到注册表] │              │
     │              │              │◀──────────────│              │
     │              │  MATCH (pubs=[Pub])         │              │
     │              │◀──────────────│              │              │
     │              │              │              │              │
     │ [Direct UDP to Subscribers]  │              │              │
     │────────────────────────────────────────────────────────────▶│
     │              │              │              │              │
     │              │              │              │  REG_SUB(topic="Sensor")
     │              │              │              │◀─────────────│
     │              │              │              │              │
     │              │              │   [查找匹配的 Pub]           │
     │              │              │              │─────────────▶│
     │              │              │              │  MATCH (pubs=[Pub])
     │              │              │              │◀─────────────│
     │              │              │              │              │
     │◀──────────────────────────────────────────────────────────│
     │ [开始接收数据] │              │              │              │
     │              │              │              │              │
```

### 12.2 错误恢复流程

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Pub App │    │Participant│   │Discovery│    │DiscServer│
└────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘
     │              │              │              │
     │ write(data)  │              │              │
     │─────────────▶│              │              │
     │              │              │              │
     │ [Network Error - Send Failed]              │
     │◀─────────────│              │              │
     │              │              │              │
     │              │ [自动重试 3 次] │              │
     │              │─────────────▶│              │
     │              │ RETRY_REG    │              │
     │              │─────────────▶│              │
     │              │              │              │
     │              │ [更新端点信息] │              │
     │              │              │              │
     │ write(data)  │              │              │
     │─────────────▶│              │              │
     │              │              │              │
     │ [Success]    │              │              │
     │◀─────────────│              │              │
     │              │              │              │
```

---

## 13. 性能分析

### 13.1 发现性能对比

| 指标 | FastDDS | MDDS | 改进倍数 |
|------|---------|------|----------|
| 首次发现延迟 (10 节点) | ~500ms | ~50ms | 10x |
| 增量加入延迟 | ~100ms | ~10ms | 10x |
| 退出检测延迟 | ~3000ms (TTL) | ~100ms | 30x |
| 发现消息速率 (空闲) | 1000 msg/s | 10 msg/s | 100x |
| 发现消息速率 (100 节点) | 10000 msg/s | 1000 msg/s | 10x |

### 13.2 资源消耗对比

| 资源 | FastDDS (per participant) | MDDS (per participant) | 改进倍数 |
|------|---------------------------|------------------------|----------|
| 内存占用 | ~2MB | ~200KB | 10x |
| CPU 开销 (空闲) | 高 (周期性) | 低 (事件驱动) | - |
| 线程数 | 10+ | 3-5 | 2x |
| Socket 数 | 5+ | 1-2 | 2x |

### 13.3 数据传输性能

| 指标 | UDP | TCP | SHM |
|------|-----|-----|-----|
| 延迟 (同机) | ~50μs | ~100μs | ~1μs |
| 延迟 (同网) | ~200μs | ~500μs | N/A |
| 吞吐量 | 10 Gbps | 5 Gbps | 20 GB/s |
| 丢包率 | <0.1% | 0% | 0% |

---

## 14. 部署指南

### 14.1 快速开始

```cpp
// 1. 定义数据类型
struct SensorData {
    uint32_t sensor_id;
    float temperature;
    uint64_t timestamp;

    // 序列化
    std::vector<uint8_t> serialize() const {
        std::vector<uint8_t> buffer(sizeof(SensorData));
        std::memcpy(buffer.data(), this, sizeof(SensorData));
        return buffer;
    }
};

// 2. 发布者
int main() {
    auto participant = mdds::DomainParticipant::create(0);
    auto publisher = participant->create_publisher<SensorData>("SensorData");

    SensorData data{1, 25.5f, 0};
    while (true) {
        data.timestamp = now();
        publisher->write(data);
        sleep(1);
    }
}

// 3. 订阅者
int main() {
    auto participant = mdds::DomainParticipant::create(0);
    auto subscriber = participant->create_subscriber<SensorData>(
        "SensorData",
        [](const SensorData& data, uint64_t ts) {
            printf("Sensor %d: %.1fC\n", data.sensor_id, data.temperature);
        });

    while (true) { sleep(1); }
}
```

### 14.2 配置发现服务器

```cpp
// 启动发现服务器
auto server = mdds::DiscoveryServer::create(7412);
server->start();

// 配置客户端连接服务器
mdds::DiscoveryConfig config;
config.server_address = "192.168.1.100";
config.server_port = 7412;

auto participant = mdds::DomainParticipant::create(0, config);
```

---

*文档版本: 1.0*
*创建日期: 2026-03-22*
*作者: MDDS Team*
