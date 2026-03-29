# MLaunch 模块设计文档

## 1. 概述

MLaunch 是 MOSS 项目的进程启动和生命周期管理模块，借鉴 ROS roslaunch/ros2 launch 和 Apollo cyber_launch 的设计经验，专为汽车/嵌入式场景优化。

### 1.1 设计目标

- **轻量级**: 适合嵌入式系统的最小资源占用
- **确定性**: 启动顺序可预测，支持依赖声明
- **声明式配置**: JSON 格式的启动配置文件
- **进程管理**: 完整的进程生命周期 (spawn/monitor/restart)
- **故障恢复**: 与 msafety 模块集成，支持自动重启

### 1.2 术语表

| 术语 | 说明 |
|------|------|
| Launch File | 启动配置文件 (JSON) |
| Node Process | 被启动的节点进程 |
| Launch Manager | 主控进程，负责解析配置和进程管理 |
| Process Agent | 节点进程内的轻量代理，负责心跳上报 |
| DAG | 有向无环图，描述节点启动顺序依赖 |

---

## 2. 架构设计

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                           mlaunch 模块                                │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                     Launch Manager                             │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐   │   │
│  │  │ Config     │  │ Process     │  │ DAG Scheduler       │   │   │
│  │  │ Parser     │  │ Supervisor   │  │ (启动顺序调度)       │   │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│         ┌────────────────────┼────────────────────┐                 │
│         ▼                    ▼                    ▼                 │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐          │
│  │  Node A     │      │  Node B     │      │  Node C     │          │
│  │ ┌─────────┐ │      │ ┌─────────┐ │      │ ┌─────────┐ │          │
│  │ │Process  │ │      │ │Process  │ │      │ │Process  │ │          │
│  │ │Agent    │ │      │ │Agent    │ │      │ │Agent    │ │          │
│  │ └─────────┘ │      │ └─────────┘ │      │ └─────────┘ │          │
│  └─────────────┘      └─────────────┘      └─────────────┘          │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 目录结构

```
mlaunch/
├── include/mlaunch/
│   ├── mlaunch.h              # 主头文件
│   ├── types.h                # 类型定义
│   ├── config.h               # 配置结构
│   ├── launch_file.h          # Launch 文件解析
│   ├── process.h              # 进程管理
│   ├── process_agent.h        # 进程代理
│   ├── dag_scheduler.h        # DAG 调度器
│   └── lifecycle.h            # 生命周期管理
├── src/
│   ├── launch_file.cpp
│   ├── process.cpp
│   ├── process_agent.cpp
│   ├── dag_scheduler.cpp
│   ├── lifecycle.cpp
│   └── main.cpp
├── examples/
│   └── demo_launch.json
└── CMakeLists.txt
```

---

## 3. 配置格式设计

### 3.1 Launch 文件格式 (JSON)

```json
{
  "package": "sensor_package",
  "name": "sensor_launch",
  "version": "1.0.0",

  "parameters": {
    "domain_id": 0,
    "log_level": "info"
  },

  "nodes": [
    {
      "name": "sensor_node_a",
      "executable": "./bin/sensor_node",
      "arguments": ["--config", "sensor_a.yaml"],
      "respawn": true,
      "respawn_delay_ms": 1000,
      "respawn_max_attempts": 5,
      "required": true,
      "depends_on": []
    },
    {
      "name": "sensor_node_b",
      "executable": "./bin/sensor_node",
      "depends_on": ["sensor_node_a"]
    },
    {
      "name": "fusion_node",
      "executable": "/opt/moss/bin/fusion_node",
      "depends_on": ["sensor_node_b"]
    }
  ]
}
```

### 3.2 生命周期状态

```cpp
enum class LaunchState {
    UNINITIALIZED = 0,
    PARSING,
    SCHEDULING,
    STARTING,
    RUNNING,
    DEGRADED,
    STOPPING,
    STOPPED,
    FAILED
};

enum class NodeProcessState {
    UNSTARTED = 0,
    STARTING,
    RUNNING,
    RESPAWNING,
    STOPPING,
    STOPPED,
    FAILED,
    MAX_RESPAWN_EXCEEDED
};
```

---

## 4. 核心 API 设计

### 4.1 Launch Manager API

```cpp
class LaunchManager : public std::enable_shared_from_this<LaunchManager> {
public:
    static std::shared_ptr<LaunchManager> create();

    bool load_launch_file(const std::string& path);
    bool load_launch_config(const LaunchConfig& config);

    bool start();
    void stop();
    void destroy();

    LaunchState get_state() const;
    std::vector<ProcessInfo> get_processes() const;
    ProcessInfo get_process(const std::string& node_name) const;

    bool start_node(const std::string& node_name);
    bool stop_node(const std::string& node_name);
    bool restart_node(const std::string& node_name);
};
```

### 4.2 Process Agent API

```cpp
class ProcessAgent {
public:
    static std::shared_ptr<ProcessAgent> create(uint32_t node_id,
                                                const std::string& node_name);

    bool start_heartbeat(uint32_t interval_ms = 100);
    void stop_heartbeat();
    bool report_status(ProcessStatus status);
    bool notify_ready();
    bool notify_shutdown();
};
```

---

## 5. 使用示例

### 5.1 命令行使用

```bash
# 启动 launch 文件
mlaunch launch sensor_system.json

# 列出已启动的节点
mlaunch list

# 停止指定节点
mlaunch stop sensor_node_a

# 重启指定节点
mlaunch restart fusion_node
```

### 5.2 库方式使用

```cpp
#include <mlaunch/mlaunch.h>

int main() {
    auto manager = mlaunch::LaunchManager::create();

    manager->set_node_exit_callback([](const std::string& name, int code, int sig) {
        printf("Node %s exited\n", name.c_str());
    });

    if (!manager->load_launch_file("/opt/moss/launch/sensor.json")) {
        return 1;
    }

    if (!manager->start()) {
        return 1;
    }

    while (manager->get_state() == mlaunch::LaunchState::RUNNING) {
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }

    return 0;
}
```

---

## 6. 依赖关系

```
mlaunch
├── mshm (共享内存通信)
├── mlog (日志)
└── msafety (可选，心跳监控集成)
```

---

## 7. 构建配置

```bash
cd mlaunch && mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DMLAUNCH_BUILD_TESTS=ON
make -j4
ctest --output-on-failure
```
