# MLaunch 模块设计文档

## 1. 概述

MLaunch 是 MOSS 项目的进程启动和生命周期管理模块，借鉴 ROS roslaunch/ros2 launch 和 Apollo cyber_launch 的设计经验，专为汽车/嵌入式场景优化。

### 1.1 核心设计理念

**每个节点独立配置文件** — 这是 mlaunch 与 roslaunch 的核心区别：
- 不再使用一个大的 launch 文件启动所有节点
- 每个节点有自己独立的 JSON 配置文件
- 通过 `parent` 字段引用共享配置，实现配置复用

### 1.2 设计目标

- **轻量级**: 适合嵌入式系统的最小资源占用
- **确定性**: 启动顺序可预测，支持依赖声明
- **声明式配置**: JSON 格式的节点配置文件
- **进程管理**: 完整的进程生命周期 (spawn/monitor/restart)
- **故障恢复**: 与 msafety 模块集成，支持自动重启

### 1.3 术语表

| 术语 | 说明 |
|------|------|
| Node Config | 节点配置文件 (JSON)，每个节点独立 |
| Node Process | 被启动的节点进程 |
| NodeLauncher | 节点启动器，负责解析配置和进程管理 |
| ProcessAgent | 节点进程内的轻量代理，负责心跳上报 |
| Parent Config | 父配置，被节点配置引用以继承共享参数 |

---

## 2. 架构设计

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                           mlaunch 模块                                │
│                                                                      │
│  每个节点独立配置，节点间通过 parent 共享配置                           │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                     NodeLauncher                              │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │   │
│  │  │ Config      │  │ Process     │  │ Lifecycle       │   │   │
│  │  │ Parser      │  │ Supervisor   │  │ Manager        │   │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│         ┌────────────────────┼────────────────────┐                 │
│         ▼                    ▼                    ▼                 │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐      │
│  │  Node A     │      │  Node B     │      │  Node C     │      │
│  │  (独立配置)  │      │  (独立配置)  │      │  (独立配置)  │      │
│  └─────────────┘      └─────────────┘      └─────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 目录结构

```
mlaunch/
├── include/mlaunch/
│   ├── mlaunch.h              # 主头文件
│   ├── types.h                # 类型定义
│   ├── config.h               # 节点配置结构
│   ├── launch_file.h         # 配置文件加载器
│   ├── process.h             # 进程管理
│   ├── process_agent.h       # 进程代理
│   ├── dag_scheduler.h       # DAG 调度器
│   └── lifecycle.h            # 生命周期管理
├── src/
│   ├── launch_file.cpp       # 配置文件加载实现
│   ├── process.cpp
│   ├── process_agent.cpp
│   ├── dag_scheduler.cpp
│   ├── lifecycle.cpp
│   └── main.cpp              # 入口程序
├── examples/
│   ├── nodes/
│   │   ├── sensor/          # 传感器节点示例
│   │   │   ├── sensor_node_a.json
│   │   │   └── sensor_node_b.json
│   │   └── fusion/          # 融合节点示例
│   │       └── fusion_node.json
│   └── shared/               # 共享配置示例
│       └── default.json
└── CMakeLists.txt
```

---

## 3. 节点配置

### 3.1 节点配置结构

每个节点有独立的 JSON 配置文件，不再共享一个大的启动文件。

**最小配置示例:**

```json
{
  "name": "sensor_node_a",
  "package": "sensor_package",
  "version": "1.0.0",
  "executable": "/usr/local/bin/sensor_node",
  "arguments": ["--device", "/dev/sensor_a"],
  "domain_id": "0",
  "log_level": "info"
}
```

**完整配置示例:**

```json
{
  "name": "sensor_node_a",
  "package": "sensor_package",
  "version": "1.0.0",
  "executable": "/usr/local/bin/sensor_node",
  "arguments": ["--device", "/dev/sensor_a"],
  "depends_on": [],
  
  "respawn": true,
  "respawn_delay_ms": 1000,
  "respawn_max_attempts": 5,
  "required": true,
  
  "domain_id": "0",
  "log_level": "info",
  
  "parent": "../shared/default.json"
}
```

### 3.2 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | ✅ | 节点名称，唯一标识 |
| `package` | string | ✅ | 所属包名 |
| `version` | string | ❌ | 版本号，默认 1.0.0 |
| `executable` | string | ✅ | 可执行文件路径 |
| `arguments` | array | ❌ | 命令行参数列表 |
| `depends_on` | array | ❌ | 依赖的节点名称列表 |
| `respawn` | bool | ❌ | 失败后自动重启，默认 false |
| `respawn_delay_ms` | int | ❌ | 重启延迟(ms)，默认 1000 |
| `respawn_max_attempts` | int | ❌ | 最大重启次数，默认 5 |
| `required` | bool | ❌ | 节点失败是否导致整体失败，默认 true |
| `domain_id` | string | ❌ | DDS domain ID |
| `log_level` | string | ❌ | 日志级别: debug/info/warn/error |
| `parent` | string | ❌ | 父配置路径，用于继承共享参数 |

### 3.3 共享配置 (Parent Config)

通过 `parent` 字段引用共享配置，实现配置复用：

**共享配置示例 (shared/default.json):**

```json
{
  "name": "default_config",
  "package": "common",
  "version": "1.0.0",
  "domain_id": "0",
  "log_level": "info",
  "respawn": false,
  "respawn_delay_ms": 1000,
  "respawn_max_attempts": 3,
  "required": true
}
```

**使用共享配置的节点:**

```json
{
  "name": "sensor_node_b",
  "package": "sensor_package",
  "executable": "/usr/local/bin/sensor_node",
  "parent": "../shared/default.json",
  "respawn": true
}
```

节点配置会继承父配置中未指定的字段，并可覆盖已指定的字段。

---

## 4. 使用方法

### 4.1 编译

```bash
cd mlaunch/build
cmake .. && make -j4
```

### 4.2 启动节点

```bash
# 启动单个节点
./mlaunch examples/nodes/sensor/sensor_node_a.json

# 节点会自动继承 parent 指定的共享配置
```

### 4.3 启动顺序

节点按照 `depends_on` 声明的依赖关系自动调度：

```
节点 A (无依赖)
   ↓
节点 B (depends_on: [A])
   ↓
节点 C (depends_on: [B])
```

### 4.4 故障恢复

当节点异常退出时：
1. 如果 `respawn=true`，延迟 `respawn_delay_ms` 后自动重启
2. 重启次数超过 `respawn_max_attempts` 后停止重启
3. 如果 `required=true`，节点失败会导致整体 launch 失败

---

## 5. 与 ROS Launch 对比

| 特性 | ROS roslaunch | mlaunch |
|------|---------------|---------|
| 配置文件 | 单一 launch 文件启动多节点 | 每个节点独立配置 |
| 配置复用 | 通过 include 嵌套 | 通过 parent 继承 |
| 依赖声明 | implicit/graph-based | explicit depends_on |
| 进程监控 | roslaunch + nodelet | NodeLauncher + ProcessSupervisor |
| 故障恢复 | respawn + respawn_delay_start | respawn + respawn_delay_ms + respawn_max_attempts |

---

## 6. 示例

### 6.1 文件结构

```
examples/
├── nodes/
│   ├── sensor/
│   │   ├── sensor_node_a.json   # 独立配置，完整参数
│   │   └── sensor_node_b.json   # 继承共享配置
│   └── fusion/
│       └── fusion_node.json      # 依赖传感器节点
└── shared/
    └── default.json             # 共享默认配置
```

### 6.2 启动示例

```bash
# 分别启动每个节点
./mlaunch examples/nodes/sensor/sensor_node_a.json &
./mlaunch examples/nodes/sensor/sensor_node_b.json &
./mlaunch examples/nodes/fusion/fusion_node.json &

# 或使用进程管理器批量启动
```
