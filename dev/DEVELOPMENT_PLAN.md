# MOSS 开发计划

> 版本: v1.0
> 日期: 2026-04-04
> 状态: Draft

---

## 一、目标概览

本次开发围绕三条主线展开：

1. **P0 Bug 修复** — 修复阻塞 MOSS 模式编译和运行的关键 bug
2. **mexem 节点执行管理器** — 新增统一的节点编排、状态监控、启停控制模块
3. **架构清理与增强** — 消除模块间职责重叠，补齐缺失的核心功能

---

## 二、P0 Bug 修复（第一阶段，1-2 周）

### 2.1 mdds 模块

| # | 文件 | 问题 | 修复方案 |
|---|------|------|---------|
| B1 | `include/mdds/qos.h` | `QoSConfig::durability` 类型为 `QoSFlags`，但赋值 `QoSDurabilityFlags::VOLATILE` | 将 durability 字段类型改为 `QoSDurabilityFlags` |
| B2 | `src/discovery.cpp:65` | `htonl` 返回 32-bit 写入 `size_t`（8 字节），反序列化端读 8 字节导致错位 | 统一使用 `uint32_t` 作为长度字段 |
| B3 | `src/discovery_server.cpp` | 与 B2 对称的反序列化错位 | 与 B2 一起修复 |
| B4 | `include/mdds/data_reader.h:170` | Topic ID 过滤被禁用（TODO 注释），所有 reader 收到所有 topic | 实现 topic_id 匹配过滤 |
| B5 | `src/multicast_discovery.cpp` | `sender_id` 初始化为自身 participant_id 而非 0 | 初始化为 0 |
| B6 | `include/mdds/subscriber.h` | 回调 lambda 捕获 `this` 但 subscriber 可能已被销毁 | 使用 `weak_ptr` 捕获 |

### 2.2 mcom 模块

| # | 文件 | 问题 | 修复方案 |
|---|------|------|---------|
| B7 | `include/mcom/node/node.h` + `src/node/node.cpp` | 模板方法定义在 .cpp 中导致链接错误 | 将模板实现移到 .h 或 .ipp 文件 |
| B8 | `src/node/node.cpp` | `Node::destroy()` 持有 `state_mutex_` 时调用 `stop()`，`stop()` 也尝试获取同一 mutex → 死锁 | `destroy()` 先释放锁再调用 `stop()` |
| B9 | `include/mcom/service/proto_service_server.h:119` | `resp.New()` 创建空默认消息，未拷贝响应数据 | 使用 `resp` 直接序列化 |
| B10 | `src/service/service_client.cpp` | 每次 `send_request` 创建新的 publisher+subscriber，subscriber 泄漏 | 复用已创建的 publisher/subscriber |
| B11 | `src/action/action_server.cpp:119` | `publish_status_update()` 是空操作 | 实现实际的状态发布逻辑 |

### 2.3 mlog 模块

| # | 文件 | 问题 | 修复方案 |
|---|------|------|---------|
| B12 | `include/mlog/mlog.h` | 头文件中定义 `static std::shared_ptr<Logger>` 导致 ODR 违规 | 改为 `inline` 或移到 .cpp |
| B13 | `src/logger.cpp` | 日志队列使用 `sleep_for(10ms)` busy-wait | 改用 `std::condition_variable` |
| B14 | `src/formatter.cpp` | `format()` 实现忽略 `pattern_` 成员，使用硬编码格式 | 实现 pattern 解析和应用 |

### 2.4 config_server 模块

| # | 文件 | 问题 | 修复方案 |
|---|------|------|---------|
| B15 | `src/config_server.cpp` | `init()` 加载数据后 `start()` 重建 `store_` 导致数据丢失 | `start()` 不再重建 store_ |
| B16 | `src/domain_socket_server.cpp` | socket 路径不匹配（代码用 `/var/run`，文档写 `/tmp`） | 统一使用 `/tmp/moss_config.sock` |
| B17 | `src/domain_socket_server.cpp` | `subscribe` 是空操作 | 实现基于 Unix socket 的变更通知 |

### 2.5 mparam 模块

| # | 文件 | 问题 | 修复方案 |
|---|------|------|---------|
| B18 | `include/mparam/param_types.h` | union 中使用 `std::string*` 但无析构/拷贝/移动构造 | 改用 `std::variant` 或手动实现 Rule of 5 |

### 2.6 mshm 模块

| # | 文件 | 问题 | 修复方案 |
|---|------|------|---------|
| B19 | `src/shm_linux.c` | 多客户端 `dup()` 共享同一 eventfd，导致通知竞争 | 每个客户端创建独立 eventfd，server 管理 fd 列表 |
| B20 | `src/shm_linux.c` | ref_count 无原子保护，多线程竞争 | 使用 `__atomic_fetch_add` |

### 验收标准

- [ ] `cmake .. -DSIMULATION_MODE=OFF` 全模块编译通过
- [ ] mdds Topic 过滤测试：subscriber A 只收到 topic A 的消息
- [ ] mcom Service 调用端到端测试通过
- [ ] config_server init→start 不丢失数据
- [ ] 无 ASAN/TSAN 报告

---

## 三、mexem 节点执行管理器（第二阶段，3-4 周）

### 3.1 概述

**mexem** (MOSS Executor Manager) 是 MOSS 的节点执行管理器，负责：
- 多节点编排启动（DAG 调度）
- 实时查看所有节点状态
- 动态启停指定节点
- 节点健康监控（集成 msafety 心跳）

与现有 mlaunch 的关系：
- **mlaunch** = 底层进程管理库（fork/exec/monitor/respawn）
- **mexem** = 上层编排器 + CLI + 管理界面，调用 mlaunch API

```
┌──────────────────────────────────────────────┐
│                  mexem                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ CLI      │  │ Orchestrator│ │ Status   │  │
│  │ (命令行) │  │ (编排引擎) │  │ Monitor  │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       │              │              │         │
│       └──────────────┼──────────────┘         │
│                      ▼                        │
│              ┌──────────────┐                 │
│              │   mlaunch    │ (底层进程管理)   │
│              │   msafety    │ (心跳监控)      │
│              │   mlog       │ (日志)          │
│              └──────────────┘                 │
└──────────────────────────────────────────────┘
```

### 3.2 目录结构

```
mexem/
├── include/mexem/
│   ├── mexem.h                # 主头文件
│   ├── orchestrator.h         # 编排引擎
│   ├── node_registry.h        # 节点注册表
│   ├── status_monitor.h       # 状态监控
│   ├── cli.h                  # CLI 命令解析
│   └── types.h                # 类型定义
├── src/
│   ├── orchestrator.cpp
│   ├── node_registry.cpp
│   ├── status_monitor.cpp
│   ├── cli.cpp
│   └── main.cpp               # 入口
├── config/
│   └── moss_system.json       # 系统级编排配置示例
├── CMakeLists.txt
└── README.md
```

### 3.3 核心类型定义 (`types.h`)

```cpp
namespace mexem {

// 节点运行状态（从 msafety 心跳获取）
enum class NodeHealth {
    UNKNOWN,        // 未注册或刚启动
    HEALTHY,        // 心跳正常
    DEGRADED,       // 心跳延迟
    FAULT           // 心跳超时
};

// 编排状态
enum class OrchestratorState {
    IDLE,
    STARTING,       // 正在按 DAG 启动节点
    RUNNING,        // 所有节点运行中
    PARTIAL,        // 部分节点运行
    STOPPING,       // 正在停止
    STOPPED,
    FAILED          // 启动失败
};

// 节点描述（来自编排配置）
struct NodeDescriptor {
    std::string name;
    std::string executable;
    std::vector<std::string> arguments;
    std::vector<std::string> depends_on;   // 依赖的节点名
    bool respawn{false};
    uint32_t respawn_delay_ms{1000};
    uint32_t respawn_max_attempts{5};
    bool required{true};                   // 失败是否导致整体失败
    uint32_t node_id{0};                   // msafety node_id
};

// 系统级编排配置
struct SystemConfig {
    std::string system_name;
    uint8_t domain_id{0};
    std::vector<NodeDescriptor> nodes;
};

// 节点运行时信息
struct NodeRuntimeInfo {
    std::string name;
    pid_t pid{0};
    NodeHealth health{NodeHealth::UNKNOWN};
    mlaunch::NodeProcessState process_state{mlaunch::NodeProcessState::UNSTARTED};
    uint32_t restart_count{0};
    uint64_t last_heartbeat_us{0};
    uint64_t uptime_ms{0};
};

} // namespace mexem
```

### 3.4 编排配置格式 (`config/moss_system.json`)

```json
{
  "system_name": "moss_drive",
  "domain_id": 0,
  "nodes": [
    {
      "name": "perception_node",
      "executable": "./build/src/perception_node/perception_node",
      "arguments": [],
      "depends_on": [],
      "respawn": true,
      "respawn_delay_ms": 1000,
      "respawn_max_attempts": 5,
      "required": true,
      "node_id": 1
    },
    {
      "name": "localization_node",
      "executable": "./build/src/localization_node/localization_node",
      "arguments": [],
      "depends_on": [],
      "respawn": true,
      "required": true,
      "node_id": 2
    },
    {
      "name": "planning_node",
      "executable": "./build/src/planning_node/planning_node",
      "arguments": [],
      "depends_on": ["perception_node", "localization_node"],
      "respawn": true,
      "required": true,
      "node_id": 3
    },
    {
      "name": "control_node",
      "executable": "./build/src/control_node/control_node",
      "arguments": [],
      "depends_on": ["planning_node"],
      "respawn": true,
      "required": true,
      "node_id": 4
    },
    {
      "name": "safety_monitor_node",
      "executable": "./build/src/safety_monitor_node/safety_monitor_node",
      "arguments": [],
      "depends_on": [],
      "respawn": true,
      "required": true,
      "node_id": 5
    },
    {
      "name": "visualization_node",
      "executable": "./build/src/visualization_node/visualization_node",
      "arguments": [],
      "depends_on": ["perception_node", "localization_node", "planning_node", "control_node"],
      "respawn": false,
      "required": false,
      "node_id": 6
    }
  ]
}
```

### 3.5 Orchestrator 编排引擎 (`orchestrator.h`)

```cpp
namespace mexem {

class Orchestrator {
public:
    static std::shared_ptr<Orchestrator> create();

    // 配置加载
    bool load_system_config(const std::string& config_path);
    bool load_system_config(const SystemConfig& config);

    // 生命周期
    bool start_all();           // 按 DAG 顺序启动所有节点
    void stop_all();            // 按依赖反序停止所有节点
    void destroy();

    // 单节点控制
    bool start_node(const std::string& name);
    bool stop_node(const std::string& name);
    bool restart_node(const std::string& name);

    // 状态查询
    OrchestratorState get_state() const;
    NodeRuntimeInfo get_node_info(const std::string& name) const;
    std::vector<NodeRuntimeInfo> get_all_node_info() const;
    std::vector<std::string> get_startable_nodes() const;

    // 依赖图
    bool validate_dependencies() const;       // 检查循环依赖
    std::vector<std::string> get_startup_order() const;  // 返回拓扑排序

private:
    SystemConfig config_;
    OrchestratorState state_{OrchestratorState::IDLE};
    std::unique_ptr<mlaunch::DagScheduler> scheduler_;
    std::map<std::string, std::shared_ptr<mlaunch::NodeLauncher>> launchers_;
    std::unique_ptr<StatusMonitor> status_monitor_;
    mutable std::mutex mutex_;
};

} // namespace mexem
```

### 3.6 StatusMonitor 状态监控 (`status_monitor.h`)

```cpp
namespace mexem {

class StatusMonitor {
public:
    StatusMonitor();
    ~StatusMonitor();

    // 初始化，注册要监控的节点
    bool init(const std::vector<NodeDescriptor>& nodes);

    // 获取节点健康状态
    NodeHealth get_health(const std::string& name) const;
    uint64_t get_last_heartbeat_us(const std::string& name) const;

    // 获取所有节点状态
    std::map<std::string, NodeHealth> get_all_health() const;

    // 启动/停止监控线程
    void start();
    void stop();

    // 设置状态变更回调
    using HealthChangeCallback = std::function<void(const std::string& name, NodeHealth old_state, NodeHealth new_state)>;
    void set_health_change_callback(HealthChangeCallback cb);

private:
    void monitor_loop();

    struct NodeMonitorEntry {
        std::string name;
        uint32_t node_id;
        NodeHealth health{NodeHealth::UNKNOWN};
        uint64_t last_heartbeat_us{0};
    };

    std::map<std::string, NodeMonitorEntry> entries_;
    std::thread monitor_thread_;
    std::atomic<bool> running_{false};
    HealthChangeCallback health_callback_;
    mutable std::mutex mutex_;
};

} // namespace mexem
```

StatusMonitor 内部调用 `msafety_monitor_*` API：`init()` 时为每个节点调用 `msafety_monitor_add_node(node_id)`，监控线程定期调用 `msafety_monitor_get_health()` 并更新本地状态。

### 3.7 NodeRegistry 节点注册表 (`node_registry.h`)

```cpp
namespace mexem {

class NodeRegistry {
public:
    // 注册/注销节点
    void register_node(const std::string& name, const NodeDescriptor& desc);
    void unregister_node(const std::string& name);

    // 查询
    bool has_node(const std::string& name) const;
    const NodeDescriptor& get_descriptor(const std::string& name) const;
    std::vector<std::string> get_all_node_names() const;
    std::vector<std::string> get_dependencies(const std::string& name) const;
    std::vector<std::string> get_dependents(const std::string& name) const;

    // 验证
    bool validate() const;                    // 检查所有依赖引用有效
    bool has_cycle() const;                   // 检查循环依赖
    std::vector<std::string> topological_sort() const;

private:
    std::map<std::string, NodeDescriptor> nodes_;
};

} // namespace mexem
```

### 3.8 CLI 命令行接口 (`cli.h`)

mexem 提供命令行工具，支持以下命令：

```bash
# 启动整个系统（按 DAG 编排）
mexem start config/moss_system.json

# 停止整个系统
mexem stop

# 查看所有节点状态
mexem status

# 查看指定节点状态
mexem status perception_node

# 启动指定节点
mexem start-node perception_node

# 停止指定节点
mexem stop-node perception_node

# 重启指定节点
mexem restart-node perception_node

# 查看启动顺序
mexem topo

# 检查依赖关系
mexem check config/moss_system.json
```

CLI 内部流程：

```
mexem status
  → Orchestrator::get_all_node_info()
    → StatusMonitor::get_all_health()   (从 msafety 读心跳)
    → 每个 NodeLauncher::get_info()     (从 mlaunch 读进程状态)
  → 格式化输出表格
```

### 3.9 CLI 输出示例

```
$ mexem status

MOSS System: moss_drive  [RUNNING]  Uptime: 00:05:32

┌─────────────────────┬─────┬───────────┬─────────┬───────┬──────────┐
│ Node                │ PID │ Process   │ Health  │ Restarts │ Uptime  │
├─────────────────────┼─────┼───────────┼─────────┼───────┼──────────┤
│ perception_node     │ 1021│ RUNNING   │ HEALTHY │ 0       │ 00:05:32 │
│ localization_node   │ 1022│ RUNNING   │ HEALTHY │ 0       │ 00:05:32 │
│ planning_node       │ 1023│ RUNNING   │ HEALTHY │ 1       │ 00:04:10 │
│ control_node        │ 1024│ RUNNING   │ HEALTHY │ 0       │ 00:05:30 │
│ safety_monitor_node │ 1025│ RUNNING   │ HEALTHY │ 0       │ 00:05:32 │
│ visualization_node  │ 1026│ RUNNING   │ HEALTHY │ 0       │ 00:05:28 │
└─────────────────────┴─────┴───────────┴─────────┴───────┴──────────┘
```

```
$ mexem stop-node planning_node

[INFO] Stopping planning_node (PID 1023)...
[INFO] Waiting for dependents: control_node, visualization_node
[WARN] control_node depends on planning_node — stopping control_node first
[WARN] visualization_node depends on planning_node — stopping visualization_node first
[INFO] visualization_node stopped
[INFO] control_node stopped
[INFO] planning_node stopped
[INFO] 3 nodes stopped
```

### 3.10 启动编排时序

```
mexem start config/moss_system.json

Phase 1 (无依赖，并行启动):
  ├── perception_node     ──► fork() + msafety_agent_init(1)
  ├── localization_node   ──► fork() + msafety_agent_init(2)
  └── safety_monitor_node ──► fork() + msafety_agent_init(5)

  等待全部 HEALTHY (最多 10s)

Phase 2 (依赖 Phase 1):
  └── planning_node       ──► fork() + msafety_agent_init(3)

  等待 HEALTHY

Phase 3 (依赖 Phase 2):
  └── control_node        ──► fork() + msafety_agent_init(4)

  等待 HEALTHY

Phase 4 (依赖 Phase 1+2+3):
  └── visualization_node  ──► fork() + msafety_agent_init(6)

[INFO] All 6 nodes started successfully
```

### 3.11 依赖关系

mexem 依赖：

| 模块 | 用途 | 接口 |
|------|------|------|
| mlaunch | 进程 fork/exec/monitor/respawn | `NodeLauncher`, `ProcessSupervisor`, `DagScheduler` |
| msafety | 节点心跳健康检查 | `msafety_monitor_*` API |
| mlog | 日志输出 | `mlog::Logger` |

mexem **不依赖** mcom/mdds — mexem 是进程级管理器，不参与节点间通信。

### 3.12 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.14)
project(mexem)

set(CMAKE_CXX_STANDARD 17)

# 依赖 mlaunch, msafety, mlog
add_library(mexem SHARED
    src/orchestrator.cpp
    src/node_registry.cpp
    src/status_monitor.cpp
    src/cli.cpp
)

target_include_directories(mexem PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${MOSS_ROOT}/mlaunch/include
    ${MOSS_ROOT}/msafety/include
    ${MOSS_ROOT}/mlog/include
)

target_link_libraries(mexem PUBLIC
    mlaunch::mlaunch
    msafety::msafety
    mlog::mlog
    pthread
)

# CLI 可执行文件
add_executable(mexem_cli src/main.cpp)
target_link_libraries(mexem_cli PRIVATE mexem)
set_target_properties(mexem_cli PROPERTIES OUTPUT_NAME "mexem")
```

### 3.13 与 mlaunch 的集成方式

mexem **复用** mlaunch 的现有代码，不是替代：

```
mlaunch (已有，底层库)
  ├── DagScheduler        ◄── mexem::NodeRegistry 直接复用拓扑排序
  ├── ProcessSupervisor   ◄── mexem::Orchestrator 持有，管理所有进程
  ├── NodeLauncher        ◄── 每个节点一个 launcher 实例
  ├── Process             ◄── fork/exec/monitor/respawn 逻辑
  └── Lifecycle           ◄── 状态机

mexem (新增，上层编排器)
  ├── Orchestrator        ──► 组合 DagScheduler + 多个 NodeLauncher
  ├── StatusMonitor       ──► 调用 msafety_monitor_* API
  ├── NodeRegistry        ──► 节点元数据管理 + 依赖验证
  └── CLI                 ──► 命令行入口
```

需要对 mlaunch 做的改动：
1. `DagScheduler` 补齐多节点拓扑排序（当前是单节点简化模式）
2. `NodeLauncher` 增加 `get_pid()` 方法
3. `ProcessAgent` 实现真正的心跳上报（当前是 stub）

---

## 四、架构清理（第三阶段，2-3 周）

### 4.1 msomeip / mrouting 职责合并

当前问题：
- msomeip 和 mrouting 都有 `ShmAgentId`、`ServiceEntry` 等重复枚举
- mrouting 依赖 msomeip 但功能边界不清晰

方案：
- 将 mrouting 的路由逻辑合并进 msomeip，删除 mrouting 模块
- 统一枚举定义，消除重复

### 4.2 mcom 模板方法重构

当前问题：
- `create_publisher<T>()` 等模板方法定义在 `node.cpp` 中，导致链接错误
- 用户包含 `node.h` 但找不到模板实例化

方案：
- 将所有模板方法移到 `node.h` 或新建 `node_impl.h`（被 `node.h` include）
- 删除 `node.cpp` 中的模板定义

### 4.3 mshm 多客户端 eventfd 修复

当前问题：
- 所有客户端 `dup()` 服务端的 eventfd，通知竞争

方案：
- 每个客户端创建独立 eventfd
- 服务端维护 fd 列表，写入时 `write()` 到所有客户端 fd
- shm_header 中增加 `client_count` 和 `client_fds[MAX_CLIENTS]` 字段

### 4.4 config_server 数据持久化

当前问题：
- 配置仅在内存中，服务重启丢失

方案：
- `init()` 时从 JSON 文件加载配置
- `set()` 操作同时写回文件
- 支持 `export` / `import` 命令

---

## 五、里程碑时间线

```
Week 1-2:   P0 Bug 修复 (B1-B20)
             ├── mdds QoS + discovery + topic filter
             ├── mcom 模板 + deadlock + service leak
             ├── mlog ODR + busy-wait + formatter
             ├── config_server data loss + socket
             ├── mparam unsafe union
             └── mshm eventfd + refcount

Week 3:     mlaunch 增强 (为 mexem 做准备)
             ├── DagScheduler 多节点拓扑排序
             ├── NodeLauncher::get_pid()
             └── ProcessAgent 心跳实现

Week 4-5:   mexem 核心实现
             ├── types.h + NodeRegistry
             ├── Orchestrator (DAG 编排 + 单节点控制)
             ├── StatusMonitor (msafety 集成)
             └── CLI 命令实现

Week 6:     mexem 集成测试 + CLI 完善
             ├── moss_drive 编排配置
             ├── 端到端启动/停止/状态测试
             └── CLI 输出格式化

Week 7-8:   架构清理
             ├── msomeip/mrouting 合并
             ├── mshm eventfd 修复
             └── config_server 持久化
```

---

## 六、验收标准

### P0 Bug 修复验收

- [ ] 全模块 `cmake .. -DSIMULATION_MODE=OFF` 编译通过，无链接错误
- [ ] mdds topic filter 测试通过
- [ ] mcom Service RPC 端到端调用成功
- [ ] 无 ASAN/TSAN/UBSAN 报告

### mexem 验收

- [ ] `mexem start config/moss_system.json` 按 DAG 编排启动 6 个节点
- [ ] `mexem status` 显示所有节点 PID/Health/Uptime
- [ ] `mexem stop-node <name>` 停止单个节点及其下游依赖
- [ ] `mexem start-node <name>` 启动单个节点
- [ ] `mexem restart-node <name>` 重启单个节点
- [ ] 节点崩溃后 StatusMonitor 检测到 FAULT 并上报
- [ ] `mexem topo` 显示正确的拓扑排序
- [ ] `mexem check` 检测循环依赖并报错

### 架构清理验收

- [ ] msomeip 编译通过，无重复枚举
- [ ] mshm 多客户端通知无竞争
- [ ] config_server 重启后配置不丢失
