# mexem 设计文档

> mexem = MOSS Executor Manager
> 版本: v1.0
> 日期: 2026-04-04

---

## 1. 概述

mexem 是 MOSS 中间件的节点执行管理器，提供系统级的节点编排、状态监控和启停控制能力。

### 1.1 定位

| 模块 | 职责 | 粒度 |
|------|------|------|
| **mlaunch** | 进程管理库（fork/exec/monitor/respawn） | 单进程 |
| **mexem** | 系统级编排器（DAG调度/状态监控/CLI） | 多进程系统 |
| **msafety** | 心跳健康检查基础设施 | 单进程心跳 |

mexem 是 mlaunch 的上层消费者，不替代 mlaunch。

### 1.2 设计原则

1. **复用优先** — 复用 mlaunch 的进程管理、msafety 的心跳检测
2. **声明式配置** — JSON 编排文件描述节点和依赖关系
3. **安全停止** — 停止节点时自动停止其下游依赖
4. **实时可见** — CLI status 命令实时展示所有节点状态

---

## 2. 架构

### 2.1 分层架构

```
┌─────────────────────────────────────────────────────┐
│                    用户交互层                         │
│  ┌──────────┐                                       │
│  │   CLI    │  mexem start / status / stop-node     │
│  └────┬─────┘                                       │
│       │                                             │
├───────┼─────────────────────────────────────────────┤
│       ▼              编排层                          │
│  ┌──────────────────────────────────────────────┐  │
│  │              Orchestrator                     │  │
│  │  ┌─────────────┐  ┌───────────────────────┐  │  │
│  │  │NodeRegistry │  │ DAG 依赖图管理        │  │  │
│  │  │ (节点元数据) │  │ (拓扑排序/循环检测)   │  │  │
│  │  └─────────────┘  └───────────────────────┘  │  │
│  └──────────────────┬───────────────────────────┘  │
│                     │                               │
├─────────────────────┼───────────────────────────────┤
│                     ▼          执行层               │
│  ┌──────────────────────────────────────────────┐  │
│  │          mlaunch (现有库)                      │  │
│  │  NodeLauncher ─► ProcessSupervisor ─► Process │  │
│  │  DagScheduler (拓扑排序)                      │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
├─────────────────────────────────────────────────────┤
│                     监控层                           │
│  ┌──────────────────────────────────────────────┐  │
│  │         StatusMonitor                         │  │
│  │  调用 msafety_monitor_* API 读取心跳          │  │
│  │  维护 NodeHealth 状态表                        │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### 2.2 数据流

**启动流程：**
```
CLI: mexem start config.json
  │
  ├─1─► Orchestrator::load_system_config()
  │       └─► parse JSON → SystemConfig
  │       └─► NodeRegistry::register_node() × N
  │       └─► NodeRegistry::validate() + has_cycle()
  │
  ├─2─► Orchestrator::start_all()
  │       └─► NodeRegistry::topological_sort() → 启动顺序
  │       └─► Phase 1: 启动无依赖节点 (并行)
  │       │     └─► NodeLauncher::start() × 3
  │       │     └─► StatusMonitor::wait_healthy(timeout)
  │       └─► Phase 2: 启动依赖 Phase 1 的节点
  │       │     └─► NodeLauncher::start() × 1
  │       │     └─► wait_healthy()
  │       └─► ...直到所有节点启动
  │
  └─3─► 输出结果
```

**状态监控：**
```
StatusMonitor::monitor_loop() (每 200ms)
  │
  ├─► 遍历所有注册节点
  │     └─► msafety_monitor_get_health(node_id, &report)
  │           └─► 读取 POSIX SHM /moss_health_{node_id}
  │           └─► 检查 last_heartbeat 时效性
  │     └─► 更新本地 NodeHealth
  │     └─► 如果状态变更 → 触发 HealthChangeCallback
  │
  └─► CLI: mexem status
        └─► Orchestrator::get_all_node_info()
              └─► StatusMonitor::get_all_health()
              └─► NodeLauncher::get_info() (每个 launcher)
              └─► 返回 NodeRuntimeInfo 列表
```

---

## 3. 关键设计决策

### 3.1 为什么不直接用 mlaunch？

mlaunch 的 API 设计为**单节点管理器**：
- `NodeLauncher` 只管理一个节点
- `DagScheduler` 目前是单节点简化模式
- 没有系统级的 `start_all()` / `stop_all()`
- 没有 CLI 界面
- 没有节点状态聚合展示

mexem 在 mlaunch 之上提供**系统级视角**。

### 3.2 为什么不用 mcom？

mexem 是**进程级管理器**，不是通信中间件：
- 不需要 Topic/Service/Action
- 不需要 DDS 发现
- 不需要 protobuf 序列化

mexem 通过 msafety（POSIX SHM 心跳）获取节点健康状态，这是轻量级的进程间通信，不需要 mcom 的重量级基础设施。

### 3.3 停止节点时为什么需要停止下游依赖？

如果 planning_node 被停止，但 control_node 仍在运行：
- control_node 订阅 `/planning/trajectory`，但收不到数据
- control_node 可能使用过期数据或默认行为
- 导致车辆控制异常

因此 mexem 默认会**级联停止**下游依赖。可通过 `--no-cascade` 选项覆盖。

### 3.4 StatusMonitor 为什么不直接调用 msafety_agent API？

`msafety_agent_*` 是**被监控进程内部**调用的（agent 写心跳）。
`msafety_monitor_*` 是**外部监控进程**调用的（monitor 读心跳）。

mexem 作为外部管理器，使用 `msafety_monitor_*` API 读取各节点的心跳数据。

---

## 4. CLI 命令参考

### 4.1 系统级命令

| 命令 | 说明 | 示例 |
|------|------|------|
| `mexem start <config>` | 按 DAG 启动系统 | `mexem start config/moss_system.json` |
| `mexem stop` | 停止所有节点 | `mexem stop` |
| `mexem status` | 显示所有节点状态 | `mexem status` |
| `mexem topo` | 显示启动顺序 | `mexem topo` |
| `mexem check <config>` | 验证配置和依赖 | `mexem check config/moss_system.json` |

### 4.2 节点级命令

| 命令 | 说明 | 示例 |
|------|------|------|
| `mexem start-node <name>` | 启动指定节点 | `mexem start-node perception_node` |
| `mexem stop-node <name>` | 停止指定节点 | `mexem stop-node planning_node` |
| `mexem restart-node <name>` | 重启指定节点 | `mexem restart-node control_node` |
| `mexem info <name>` | 显示节点详情 | `mexem info perception_node` |

### 4.3 选项

| 选项 | 说明 | 适用命令 |
|------|------|---------|
| `--no-cascade` | 停止时不级联停止下游 | stop-node |
| `--force` | 强制停止 (SIGKILL) | stop, stop-node |
| `--timeout <sec>` | 等待健康超时 | start, start-node |
| `--json` | JSON 格式输出 | status, info |
| `--watch` | 持续刷新状态 | status |

---

## 5. 错误处理

### 5.1 启动失败场景

| 场景 | 处理 |
|------|------|
| 可执行文件不存在 | 拒绝启动，报错 |
| 依赖循环 | `check` 命令检测，`start` 拒绝执行 |
| 依赖节点未运行 | 等待依赖节点 HEALTHY，超时报错 |
| 节点启动后立即退出 | 按 respawn 配置重试，超过 max_attempts 标记 FAILED |
| required 节点失败 | 停止整个系统 |
| 非 required 节点失败 | 继续运行，系统状态标记为 PARTIAL |

### 5.2 运行时故障

| 场景 | 处理 |
|------|------|
| 节点心跳超时 | StatusMonitor 检测到 FAULT，触发回调 |
| 配置的 node_id 与实际不符 | StatusMonitor 报告 UNKNOWN，日志警告 |
| 节点被外部 kill -9 | mlaunch monitor 线程检测到退出，触发 respawn |
| 停止节点时有下游依赖运行 | 默认级联停止，或 `--no-cascade` 跳过 |

---

## 6. 与现有模块的集成点

### 6.1 mlaunch 集成

```cpp
// Orchestrator 内部
auto launcher = mlaunch::NodeLauncher::create();
launcher->load_config(desc_to_node_config(node_desc));
launcher->set_exit_callback([this, name](int code) {
    this->on_node_exit(name, code);
});
launcher->start();
```

需要 mlaunch 新增的 API：

```cpp
// mlaunch::NodeLauncher 新增
pid_t get_pid() const;                    // 获取子进程 PID
std::chrono::steady_clock::time_point get_start_time() const;
```

```cpp
// mlaunch::DagScheduler 增强（从单节点到多节点）
std::vector<std::string> get_startable_nodes(
    const std::vector<std::string>& completed_nodes,
    const std::vector<std::string>& failed_nodes
);
bool validate_dependencies(const std::map<std::string, std::vector<std::string>>& deps);
std::vector<std::string> topological_sort(const std::map<std::string, std::vector<std::string>>& deps);
```

### 6.2 msafety 集成

```cpp
// StatusMonitor::init()
msafety_monitor_init();
for (auto& [name, desc] : node_descriptors) {
    msafety_monitor_add_node(desc.node_id);
}

// StatusMonitor::monitor_loop()
msafety_health_report_t report;
for (auto& [name, entry] : entries_) {
    if (msafety_monitor_get_health(entry.node_id, &report) == MSAFETY_OK) {
        entry.health = report.is_alive ? NodeHealth::HEALTHY : NodeHealth::FAULT;
        entry.last_heartbeat_us = report.last_heartbeat;
    }
}
```

### 6.3 节点侧集成（被管理的节点）

被管理的节点只需在 main() 中加两行代码：

```cpp
int main() {
    // msafety 心跳上报（节点启动后立即调用）
    msafety_agent_init(NODE_ID);
    msafety_agent_report_status(MSAFETY_STATUS_HEALTHY);

    // ... 节点正常业务逻辑 ...

    // 节点退出时清理
    msafety_agent_stop();
    return 0;
}
```

或者通过 ProcessAgent（mlaunch 提供）自动完成。

---

## 7. 性能要求

| 指标 | 目标 |
|------|------|
| `mexem start` 6 节点系统 | < 5s (从执行到全部 HEALTHY) |
| `mexem status` 响应 | < 100ms |
| StatusMonitor 检测节点故障 | < 500ms (与 msafety 一致) |
| `mexem stop` 全系统停止 | < 3s (SIGTERM) |
| CLI 内存占用 | < 10MB |
| 空闲 CPU 占用 | < 0.5% |
