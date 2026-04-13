# MOSS P0 Bug 修复计划

> 日期: 2026-04-04
> 优先级: 所有 bug 在 mexem 开发前修复

---

## 一、mdds 模块 (6 bugs)

### B1: QoS durability 类型不匹配

**文件**: `mdds/include/mdds/qos.h`

**问题**: `QoSConfig::durability` 字段类型为 `QoSFlags`，但赋值的是 `QoSDurabilityFlags::VOLATILE`（不同枚举类型）。

```cpp
// 当前（错误）
struct QoSConfig {
    QoSFlags durability{QoSFlags::VOLATILE};  // 类型不匹配
};

// 修复后
struct QoSConfig {
    QoSDurabilityFlags durability{QoSDurabilityFlags::VOLATILE};
};
```

**影响**: 编译警告或运行时位运算结果错误。

---

### B2/B3: Discovery 序列化 size 错位

**文件**: `mdds/src/discovery.cpp:65` + `mdds/src/discovery_server.cpp`

**问题**: 序列化端将 `htonl()` 的 32-bit 结果赋给 `size_t`（64-bit），写入 8 字节；反序列化端读取 8 字节。在小端机器上低 4 字节正确，高 4 字节为 0，恰好碰巧正确。但在不同端序或编译器优化下会出错。

```cpp
// discovery.cpp 当前（错误）
size_t name_len_n = htonl(static_cast<uint32_t>(topic_name_len));
buffer.append(reinterpret_cast<const char*>(&name_len_n), sizeof(name_len_n));  // 写 8 字节

// discovery_server.cpp 当前（错误）
size_t name_len;
memcpy(&name_len, ptr, sizeof(size_t));  // 读 8 字节

// 修复：统一使用 uint32_t
uint32_t name_len_n = htonl(static_cast<uint32_t>(topic_name_len));
buffer.append(reinterpret_cast<const char*>(&name_len_n), sizeof(uint32_t));  // 写 4 字节

uint32_t name_len;
memcpy(&name_len, ptr, sizeof(uint32_t));  // 读 4 字节
name_len = ntohl(name_len);
```

---

### B4: Topic 过滤被禁用

**文件**: `mdds/include/mdds/data_reader.h:170-174`

**问题**: Topic ID 匹配检查被注释掉，所有 DataReader 收到所有 Topic 的数据。

```cpp
// 当前（错误）
bool matches = true;  // TODO: 实现 topic 过滤
// if (header.topic_id != topic_id_) return false;

// 修复
bool on_data_received(const uint8_t* data, size_t size, const MessageHeader& header) {
    if (header.topic_id != topic_id_) {
        return false;  // 不是本 reader 订阅的 topic
    }
    // ... 正常处理
}
```

---

### B5: Multicast sender_id 初始化错误

**文件**: `mdds/src/multicast_discovery.cpp`

**问题**: `sender_id` 初始化为自身 `participant_id`，应为 0（表示未分配）。

---

### B6: Subscriber this 悬空捕获

**文件**: `mdds/include/mdds/subscriber.h`

**问题**: 回调 lambda 捕获 `this`，但 subscriber 对象可能已被销毁。

```cpp
// 修复：使用 weak_ptr
auto weak_self = weak_from_this();
callback_ = [weak_self, cb](const T& data, const MessageHeader& header) {
    if (auto self = weak_self.lock()) {
        cb(data, header);
    }
};
```

---

## 二、mcom 模块 (5 bugs)

### B7: 模板方法定义在 .cpp 导致链接错误

**文件**: `mcom/include/mcom/node/node.h` + `mcom/src/node/node.cpp`

**问题**: `create_publisher<T>()` 等模板方法声明在 `node.h` 但实现在 `node.cpp`。其他翻译单元包含 `node.h` 时看不到模板定义，链接时报 undefined reference。

**修复**: 创建 `mcom/include/mcom/node/node_impl.h`，将所有模板实现移入。在 `node.h` 末尾 `#include "node_impl.h"`。

```cpp
// node_impl.h
template<typename T>
std::shared_ptr<mdds::Publisher<T>> Node::create_publisher(
    const std::string& topic_name, const mdds::QoSConfig& qos)
{
    std::lock_guard<std::mutex> lock(endpoints_mutex_);
    // ... 实现
}
```

**同时**从 `node.cpp` 中删除这些模板方法的定义。

---

### B8: Node::destroy() 死锁

**文件**: `mcom/src/node/node.cpp`

**问题**: `destroy()` 持有 `state_mutex_` 时调用 `stop()`，而 `stop()` 也尝试获取 `state_mutex_` → 死锁。

```cpp
// 当前（错误）
void Node::destroy() {
    std::lock_guard<std::mutex> lock(state_mutex_);
    if (state_ == NodeState::RUNNING) {
        stop();  // stop() 也尝试获取 state_mutex_ → 死锁
    }
    // ...
}

// 修复：先释放锁再调用 stop()
void Node::destroy() {
    {
        std::lock_guard<std::mutex> lock(state_mutex_);
        if (state_ == NodeState::RUNNING) {
            // 标记需要 stop，但不在此处调用
        }
    }
    if (is_running()) {
        stop();  // 在锁外调用
    }
    std::lock_guard<std::mutex> lock(state_mutex_);
    // ... 继续 destroy 逻辑
}
```

---

### B9: ServiceServer 响应序列化错误

**文件**: `mcom/include/mcom/service/proto_service_server.h:119`

**问题**: `resp.New()` 创建的是空默认 protobuf 消息，不是回调返回的响应数据。

```cpp
// 当前（错误）
auto response = resp.New();  // 创建空 TResponse
// 序列化空消息发送

// 修复：直接序列化 resp
std::string serialized;
resp.SerializeToString(&serialized);
// 发送 serialized
```

---

### B10: ServiceClient 每次调用泄漏 subscriber

**文件**: `mcom/src/service/service_client.cpp`

**问题**: `send_request()` 每次创建新的 publisher + subscriber，但从不销毁。subscriber 在请求完成后成为孤儿。

**修复**: 在构造函数中创建一次 publisher + subscriber，复用。

---

### B11: ActionServer publish_status_update 是空操作

**文件**: `mcom/src/action/action_server.cpp:119-122`

**问题**: 函数体为空 `{}`，Action 状态更新永远不会发送给 client。

**修复**: 实现通过 status topic 发布 GoalStatus。

---

## 三、mlog 模块 (3 bugs)

### B12: ODR 违规

**文件**: `mlog/include/mlog/mlog.h`

**问题**: 头文件中定义 `static std::shared_ptr<Logger> default_logger`，每个包含此头文件的翻译单元都有独立副本。

```cpp
// 当前（错误）
class LoggerFactory {
    static std::shared_ptr<Logger> default_logger;  // 每个 TU 一份
};

// 修复
class LoggerFactory {
    static inline std::shared_ptr<Logger> default_logger;  // C++17 inline variable
};
```

---

### B13: Busy-wait

**文件**: `mlog/src/logger.cpp`

**问题**: 日志消费者线程用 `sleep_for(10ms)` 轮询队列，导致高延迟或高 CPU。

```cpp
// 修复：使用 condition_variable
std::mutex queue_mutex_;
std::condition_variable queue_cv_;

// 生产者
queue_.push(entry);
queue_cv_.notify_one();

// 消费者
std::unique_lock<std::mutex> lock(queue_mutex_);
queue_cv_.wait(lock, [this] { return !queue_.empty() || !running_; });
```

---

### B14: format() 忽略 pattern_

**文件**: `mlog/src/formatter.cpp`

**问题**: `format()` 实现使用硬编码格式，完全忽略用户设置的 `pattern_` 字符串。

**修复**: 实现 pattern 解析器，支持占位符 `%d{HH:mm:ss}` `%l` `%t` `%n` `%f:%L` `%m`。

---

## 四、config_server 模块 (3 bugs)

### B15: init() → start() 数据丢失

**文件**: `config_server/src/config_server.cpp`

**问题**: `init()` 加载配置到 `store_`，然后 `start()` 重新创建 `store_ = std::make_unique<ConfigStore>()`，所有数据丢失。

**修复**: `start()` 不再重建 store_，直接使用 init() 中创建的。

---

### B16: Socket 路径不匹配

**文件**: `config_server/src/domain_socket_server.cpp`

**问题**: 代码绑定 `/var/run/moss_config.sock`，但文档和其他模块期望 `/tmp/moss_config.sock`。

**修复**: 统一为 `/tmp/moss_config.sock`，或支持配置。

---

### B17: subscribe 是空操作

**文件**: `config_server/src/domain_socket_server.cpp`

**问题**: 客户端可以 subscribe 配置变更，但服务端的 subscribe handler 是空实现。

**修复**: 维护 subscriber 列表，配置变更时通知所有 subscriber。

---

## 五、mparam 模块 (1 bug)

### B18: unsafe union

**文件**: `mparam/include/mparam/param_types.h`

**问题**: union 中包含 `std::string*`，但没有自定义析构函数、拷贝构造函数、移动构造函数。默认行为导致 double-free 或内存泄漏。

```cpp
// 修复：改用 std::variant
using ParamValue = std::variant<int64_t, double, std::string, bool>;
```

---

## 六、mshm 模块 (2 bugs)

### B19: eventfd 多客户端竞争

**文件**: `mshm/src/shm_linux.c`

**问题**: 所有客户端 `dup()` 服务端的 eventfd，多个客户端竞争同一个 fd 上的 `eventfd_read()`。

**修复方案**:
1. 服务端为每个客户端维护独立 eventfd 对
2. shm_header 中增加客户端注册机制
3. 服务端通知时遍历所有客户端 fd 写入

### B20: ref_count 无原子保护

**文件**: `mshm/src/shm_linux.c`

**问题**: `shm_header_t::ref_count` 的读写无原子操作保护。

```cpp
// 修复
__atomic_fetch_add(&header->ref_count, 1, __ATOMIC_SEQ_CST);
__atomic_fetch_sub(&header->ref_count, 1, __ATOMIC_SEQ_CST);
```

---

## 七、修复顺序

建议按以下顺序修复，每修完一组立即验证：

1. **mdds B1-B6** — 通信底层，影响所有上层
2. **mcom B7-B11** — API 层，用户直接使用
3. **mlog B12-B14** — 基础设施
4. **config_server B15-B17** — 独立模块
5. **mparam B18** — 独立模块
6. **mshm B19-B20** — 底层共享内存

每组修复后执行：
```bash
# 编译验证
cd /mnt/data/workspace/moss && mkdir -p build && cd build
cmake .. && make -j$(nproc)

# 单元测试
ctest --output-on-failure

# ASAN 验证
cmake .. -DCMAKE_CXX_FLAGS="-fsanitize=address" && make -j$(nproc)
ctest
```
