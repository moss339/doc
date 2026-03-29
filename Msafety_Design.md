# Msafety 模块设计文档

## 1. 概述

Msafety 是 MOSS 项目的功能安全模块，专注于本地节点监控，通过心跳机制检测节点存活性。当节点崩溃或卡死时，监控器能够及时检测并通知上层应用。

### 1.1 设计目标

- **心跳检测**: 周期性检测节点存活性，500ms 无心跳判定为故障
- **轻量级**: heartbeat_agent 独立线程，不阻塞主业务
- **低延迟**: 使用共享内存进行进程间通信，零拷贝
- **可扩展**: 支持多节点并发监控，提供回调机制通知故障

### 1.2 术语表

| 术语 | 说明 |
|------|------|
| heartbeat_agent | 每个被监控节点加载的轻量代理 |
| safety_monitor | 独立监控进程，轮询检测节点状态 |
| heartbeat | 心跳时间戳，写入共享内存 |
| node_id | 节点唯一标识符 |
| is_alive | 节点存活状态标志 |

---

## 2. 架构设计

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         msafety 模块                                 │
│                                                                      │
│  ┌─────────────────────┐              ┌─────────────────────┐      │
│  │   heartbeat_agent    │              │   safety_monitor    │      │
│  │   (每个被监控节点)    │              │   (独立监控进程)     │      │
│  │                     │              │                     │      │
│  │  ┌───────────────┐  │              │  ┌───────────────┐  │      │
│  │  │ 心跳线程      │  │    共享内存    │  │ 监控线程      │  │      │
│  │  │ (100ms 周期)  │──┼──────────────┼──│ (50ms 周期)  │  │      │
│  │  └───────────────┘  │   /moss_     │  └───────────────┘  │      │
│  │         │          │   health_    │         │            │      │
│  │         ▼          │   <node_id>  │         ▼            │      │
│  │  ┌───────────────┐  │              │  ┌───────────────┐  │      │
│  │  │ msafety_      │  │              │  │ 故障回调      │  │      │
│  │  │ heartbeat_t   │  │              │  │ (通知上层)    │  │      │
│  │  └───────────────┘  │              │  └───────────────┘  │      │
│  └─────────────────────┘              └─────────────────────┘      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 模块组成

| 组件 | 说明 | 位置 |
|------|------|------|
| heartbeat_agent | 心跳代理，每个节点一个实例 | `src/heartbeat_agent.cpp` |
| safety_monitor | 监控器，可监控多个节点 | `src/safety_monitor.cpp` |
| health_store | 健康状态存储和报告生成 | `src/health_store.cpp` |

---

## 3. 数据结构

### 3.1 心跳数据结构

```c
typedef struct {
    uint32_t version;           // 版本号
    uint64_t last_heartbeat;    // 最后心跳时间戳(微秒)
    uint32_t node_id;           // 节点ID
    uint32_t process_id;        // 进程ID
    uint8_t  status;            // 节点状态
    uint8_t  reserved[3];       // 保留字段
} msafety_heartbeat_t;
```

### 3.2 健康报告结构

```c
typedef struct {
    uint32_t node_id;           // 节点ID
    uint64_t last_heartbeat;    // 最后心跳时间戳
    uint64_t last_check;        // 最后检测时间戳
    uint8_t  status;            // 节点状态
    uint8_t  is_alive;          // 存活标志 (0=故障, 1=存活)
} msafety_health_report_t;
```

### 3.3 节点状态枚举

```c
enum msafety_status {
    MSAFETY_STATUS_HEALTHY = 0,   // 健康
    MSAFETY_STATUS_DEGRADED = 1,  // 亚健康
    MSAFETY_STATUS_FAULT = 2      // 故障
};
```

### 3.4 错误码

```c
enum msafety_error {
    MSAFETY_OK = 0,
    MSAFETY_ERR_INVALID_PARAM = -1,
    MSAFETY_ERR_SHM_OPEN = -2,
    MSAFETY_ERR_SHM_CREATE = -3,
    MSAFETY_ERR_SHM_WRITE = -4,
    MSAFETY_ERR_SHM_READ = -5,
    MSAFETY_ERR_NOT_FOUND = -6,
    MSAFETY_ERR_ALREADY_EXISTS = -7,
    MSAFETY_ERR_TIMEOUT = -8,
    MSAFETY_ERR_NO_MEMORY = -9,
    MSAFETY_ERR_INTERNAL = -10
};
```

---

## 4. API 设计

### 4.1 heartbeat_agent API (节点调用)

| 函数 | 说明 |
|------|------|
| `int msafety_agent_init(uint32_t node_id)` | 初始化心跳代理，创建共享内存 |
| `int msafety_agent_report_status(uint8_t status)` | 主动上报节点状态 |
| `int msafety_agent_stop(void)` | 停止心跳代理，清理资源 |

#### 使用示例

```c
#include "msafety/heartbeat_agent.h"

// 节点初始化时启动心跳代理
int ret = msafety_agent_init(12345);
if (ret != MSAFETY_OK) {
    // 处理错误
}

// 节点检测到异常时主动上报
msafety_agent_report_status(MSAFETY_STATUS_DEGRADED);

// 节点退出时停止
msafety_agent_stop();
```

### 4.2 safety_monitor API (监控进程调用)

| 函数 | 说明 |
|------|------|
| `int msafety_monitor_init(void)` | 初始化监控器 |
| `int msafety_monitor_add_node(uint32_t node_id)` | 添加监控节点 |
| `int msafety_monitor_remove_node(uint32_t node_id)` | 移除监控节点 |
| `int msafety_monitor_set_callback(void (*callback)(uint32_t, uint8_t))` | 设置故障回调 |
| `int msafety_monitor_get_health(uint32_t node_id, msafety_health_report_t* report)` | 获取健康报告 |
| `int msafety_monitor_run(bool* running)` | 启动监控循环 |
| `int msafety_monitor_stop(void)` | 停止监控 |
| `int msafety_monitor_cleanup(void)` | 清理所有资源 |

#### 使用示例

```c
#include "msafety/safety_monitor.h"

// 故障回调函数
void on_fault(uint32_t node_id, uint8_t status) {
    printf("Node %u fault detected, status=%d\n", node_id, status);
}

int main() {
    bool running = true;

    msafety_monitor_init();
    msafety_monitor_set_callback(on_fault);

    msafety_monitor_add_node(12345);
    msafety_monitor_add_node(12346);

    msafety_monitor_run(&running);

    // 等待 Ctrl+C 或收到停止信号
    while (running) {
        sleep(1);
    }

    msafety_monitor_cleanup();
    return 0;
}
```

---

## 5. 共享内存设计

### 5.1 命名规范

每个被监控节点独占一块共享内存：
- 命名格式: `/moss_health_<node_id>`
- 例如: `/moss_health_12345`

### 5.2 内存布局

```
偏移量    大小    字段
------    ----    ----
0x00      4       version
0x04      8       last_heartbeat
0x0C      4       node_id
0x10      4       process_id
0x14      1       status
0x15      3       reserved
---------------------------
Total:    24 bytes
```

### 5.3 访问控制

- 心跳代理 (写): O_CREAT | O_RDWR, 权限 0644
- 监控器 (读): O_RDWR, 权限 0644

---

## 6. 时序图

### 6.1 节点启动与监控

```
节点 A                    共享内存                监控器
  │                         │                      │
  │ msafety_agent_init(1)   │                      │
  │─────────────────────────► create /moss_health_1│
  │                         │                      │
  │                         │◄─────────────────────│ msafety_monitor_add_node(1)
  │                         │                      │
  │                         │                      │
  │  heartbeat thread       │  monitor thread      │
  │      │                  │      │               │
  │      │ write heartbeat  │      │               │
  │      │──────────────────►│      │ read          │
  │      │                  │      │───────────────►│
  │      │ (100ms)          │      │ (50ms)         │
```

### 6.2 节点故障检测

```
节点 A                    共享内存                监控器
  │                         │                      │
  │  (节点崩溃)             │                      │
  │      │                  │      │               │
  │      │ 心跳停止         │      │               │
  │      │ X                │      │ read (超时)   │
  │      │                  │      │──────────────►│
  │      │                  │      │               │
  │      │                  │      │ 检测到 500ms  │
  │      │                  │      │ 无心跳        │
  │      │                  │      │               │
  │      │                  │      │ callback     │
  │      │                  │      │ (FAULT)      │
```

---

## 7. 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| HEARTBEAT_INTERVAL_US | 100000 (100ms) | 心跳发送间隔 |
| MONITOR_INTERVAL_US | 50000 (50ms) | 监控轮询间隔 |
| HEARTBEAT_TIMEOUT_US | 500000 (500ms) | 心跳超时阈值 |

---

## 8. 依赖关系

```
msafety
└── 独立模块，无外部依赖
    ├── POSIX 共享内存 (shm_open, mmap)
    ├── POSIX 线程 (pthread)
    └── C++17 标准库
```

---

## 9. 构建与使用

### 9.1 构建命令

```bash
cd msafety && mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DMSAFETY_BUILD_TESTS=ON
make -j4
```

### 9.2 运行测试

```bash
ctest --output-on-failure
```

### 9.3 性能指标

| 指标 | 目标 | 说明 |
|------|------|------|
| 心跳延迟 | < 1ms | 从写入到可读的时间 |
| 监控开销 | < 5% CPU | 单节点监控的 CPU 占用 |
| 检测时间 | ≤ 550ms | 从节点崩溃到回调触发的最大延迟 |

---

## 10. 目录结构

```
msafety/
├── CMakeLists.txt
├── include/msafety/
│   ├── types.h             # 类型定义、错误码
│   ├── health_report.h      # 健康报告 API
│   ├── heartbeat_agent.h    # 心跳代理 API
│   ├── safety_monitor.h     # 安全监控器 API
│   └── msafety.h            # 聚合头文件
├── src/
│   ├── heartbeat_agent.cpp  # 心跳代理实现
│   ├── health_store.cpp     # 健康状态存储实现
│   └── safety_monitor.cpp    # 监控器实现
└── test/
    └── test_heartbeat.cpp   # 单元测试
```
