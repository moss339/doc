# MOSS 项目里程碑路线图

> 最后更新：2026-04-19

---

## 项目愿景

MOSS (Middleware for Object-Oriented Shared Services) 是面向汽车/嵌入式系统的轻量级中间件，目标是为车载感知、规划、控制提供高性能数据分发与服务通信基础设施。

---

## 模块现状总览

| 模块 | 语言 | 状态 | 依赖 |
|------|------|------|------|
| mshm | C | ✅ 已完成 | 底层依赖 |
| **mdds** | C++17 | ✅ 已完成 | mshm |
| **mcom** | C++17 | ✅ 已完成 | mdds, mshm |
| mruntime | C++17 | ✅ 已完成 | 独立 |
| mlog | C++20 | ✅ 已完成 | 独立 |
| msafety | C++17 | ✅ 已完成 | 独立 |
| msomeip | C++17 | ✅ 已完成 | mshm |
| mrouting | C++17 | ✅ 已完成 | msomeip, mshm |
| config_server | C++17 | ✅ 已完成 | 独立 |
| **mexem** | C++17 | ✅ 已完成 | mlaunch, msafety, mlog |
| **mvisual** | C++17 | ✅ 已完成 | mshm, mruntime, mlog, mcom |
| mlaunch | C++17 | ✅ 已完成 | mshm, mlog, msafety |
| mparam | C++17 | ✅ 已完成 | mshm |
| **mupgrade** | C++17 | ✅ 已完成 | mlog, mexem |
| **mdiag** | C++17 | ✅ 已完成 | mcom, mlog |
| **mrecord** | C++17 | ✅ 已完成 | mdds, mlog |
| **mperf** | C++17 | ✅ 已完成 | mlog, mruntime |
| **mtls** | C++17 | ✅ 已完成 | mcom, mlog |
| **mbridge** | C++17 | ✅ 已完成 | mcom, mdds, msomeip |
| **mcal** | C++17 | ✅ 已完成 | mparam, mcom, mlog |

### 模块重组说明

- **mservice** → 合并入 `mcom/service/`
- **maction** → 合并入 `mcom/action/`
- **mcom** = Topic + Service + Action + Node 四层统一通信中间件

---

## mcom 统一通信中间件里程碑

### 架构层次

```
mdds (外部依赖，DDS Topic 底层)
    ↑
mcom/topic   (封装 mdds，提供 Topic 接口)
    ↑
mcom/service (基于 topic 实现 RPC Request/Response)
    ↑
mcom/action  (基于 service 实现 Goal/Result/Feedback)
    ↑
mcom/node    (统一封装，用户入口)
```

### 内部目录结构

```
mcom/
├── include/mcom/
│   ├── topic/        # Topic 层
│   ├── service/      # Service 层
│   ├── action/       # Action 层
│   └── node/         # Node 统一封装
└── src/
```

---

### Milestone 1：mcom 骨架搭建 ⭐ P0

**目标：** 建立 mcom 目录结构，迁移 mservice/maction 代码

**任务：**
- [x] 创建 mcom 目录结构
- [x] 迁移 mservice → mcom/service/
- [x] 迁移 maction → mcom/action/
- [x] 创建 mcom/topic/ 层（封装 mdds）
- [x] 创建 mcom/node/ 层（统一入口）
- [x] 创建 CMakeLists.txt 构建配置
- [ ] 删除旧 mservice/ maction/ 目录
- [ ] 编写 Mcom_Design.md 设计文档

**验收标准：** mcom 可独立编译通过 ✅ (已通过)

---

### Milestone 2：mcom/service 服务层实现 ⭐⭐ P1

**目标：** 完成 Service RPC 通信

**任务：**
- [x] 完善 ServiceClient 同步/异步调用
- [x] 完善 ServiceServer 方法注册
- [x] 与 mdds Topic 层对接
- [ ] 编写 service_example.cpp 示例
- [ ] 单元测试覆盖

**验收标准：** 同一 Node 内 Service 调用成功返回 🔄 (进行中)

---

### Milestone 3：mcom/action 动作层实现 ⭐⭐ P1

**目标：** 完成 Action Goal/Result/Feedback 机制

**任务：**
- [ ] 完善 ActionClient send_goal/cancel_goal
- [ ] 完善 ActionServer accept/reject/execute
- [ ] 实现 GoalHandle 状态机
- [ ] 与 Service 层对接
- [ ] 编写 action_example.cpp 示例

**验收标准：** Action 能完成 Goal → Feedback → Result 全流程，支持取消

---

### Milestone 4：mcom/node 统一封装 ⭐⭐ P1

**目标：** 提供用户友好的一致性 API

**任务：**
- [ ] 实现 Node::create_publisher/subscriber
- [ ] 实现 Node::create_service_client/server
- [ ] 实现 Node::create_action_client/server
- [ ] 实现 Node::spin() 事件循环
- [ ] 统一错误处理和日志

**验收标准：** 用户可通过 Node 统一接口使用 Topic/Service/Action

---

## MVisual 可视化模块里程碑

### 背景

MVisual 负责将 MOSS 系统中的数据（DDS 主题、SOME/IP 事件）转换为 FoxGlove Studio 可消费的格式，实现汽车/嵌入式数据的实时可视化。

**已有基础：**
- 完整设计文档（`doc/Mvisual_Design.md`）
- 核心框架代码 2000+ 行（StudioBridge、Session、Channel、Protocol）
- 4个数据适配器（Image、PointCloud、Diagnostic、Plot）
- FoxGlove WebSocket 协议完整实现

---

### Milestone 1：编译验证 + FoxGlove 连通测试 ⭐ P0

**目标：** 代码能编译通过，与 FoxGlove Studio 建立实际连接

**任务：**
- [ ] 修复编译错误（如有）
- [ ] 完成 CMake 构建配置
- [ ] 验证 WebSocket 端口可访问
- [ ] 验证 FoxGlove Studio 能发现频道

**验收标准：** FoxGlove Studio Connect 面板能显示 MVisual Bridge 的频道列表

---

### Milestone 2：Image 数据端到端流 ⭐⭐ P1

**目标：** 实现 Image 数据从 mdds → MVisual → FoxGlove 的完整数据流

**任务：**
- [ ] 实现 `mdds::Subscriber` 接入 mvisual
- [ ] 完善 ImageAdapter，支持 `sensor_msgs/Image` 编码
- [ ] 实现通道自动注册（mdds topic → foxglove channel）
- [ ] 编写 Image 示例 publisher（模拟相机数据）
- [ ] FoxGlove Studio Image Panel 渲染验证

**验收标准：** FoxGlove Studio 的 Image 面板能实时显示模拟相机数据

---

### Milestone 3：PointCloud 数据 + 零拷贝优化 ⭐⭐ P1

**目标：** 支持激光雷达点云数据，利用共享内存实现零拷贝

**任务：**
- [ ] 完善 PointCloudAdapter
- [ ] 利用 mshm 共享内存，避免数据拷贝
- [ ] 高频率点云流（≥ 10Hz）稳定性测试
- [ ] 编写 PointCloud 示例 publisher
- [ ] FoxGlove Studio 3D Panel 渲染验证

**验收标准：** 30Hz 点云数据在 FoxGlove 3D Panel 流畅渲染

---

### Milestone 4：多数据源集成 ⭐⭐⭐ P2

**目标：** 同时接入 mdds 和 msomeip 数据源

**任务：**
- [ ] 实现 msomeip 事件订阅接口
- [ ] 统一 mdds/msomeip 数据路由到 MVisual
- [ ] Session 管理多客户端并发
- [ ] 通道元数据管理（topic → channel mapping）
- [ ] 压力测试：多客户端 + 多数据源

**验收标准：** 同时向 mdds topic 和 msomeip event 订阅数据，FoxGlove 多面板同时显示

---

### Milestone 5：性能优化 + 生产准备 ⭐⭐⭐ P2

**目标：** 达到生产级性能要求

**任务：**
- [ ] 消息频率基准测试（Image、PointCloud、Diagnostic）
- [ ] 延迟分析（mdds → mvisual → foxglove 全链路）
- [ ] 内存池优化，减少动态分配
- [ ] 连接稳定性（长时间运行测试）
- [ ] 错误处理与日志完善

**验收标准：** 1080p@30fps Image 延迟 < 50ms，CPU 占用 < 5%

---

### Milestone 6：Diagnostic + Plot 增强 ⭐ P3

**目标：** 完善诊断和绘图功能

**任务：**
- [ ] DiagnosticArray 编码适配
- [ ] 多 topic 绘图支持（PlotAdapter 增强）
- [ ] FoxGlove Studio Diagnostic Panel 集成
- [ ] 配置化通道管理（JSON 配置）

**验收标准：** Diagnostic 面板显示节点健康状态，Plot 面板绘制时序数据

---

## 后续模块优先级

### P0: P0 Bug 修复

详见 [doc/dev/bug_fix_plan.md](dev/bug_fix_plan.md)

修复 mdds/mcom/mlog/config_server/mparam/mshm 中阻塞编译和运行的 20 个关键 bug。

### P1: mexem 节点执行管理器 ⭐ 新增

| 模块 | 状态 | 依赖 | 说明 |
|------|------|------|------|
| **mexem** | 📋 规划中 | mlaunch, msafety, mlog | 系统级节点编排、状态监控、启停控制 |

- 详见 [doc/dev/mexem_design.md](dev/mexem_design.md)
- 开发计划详见 [doc/dev/DEVELOPMENT_PLAN.md](dev/DEVELOPMENT_PLAN.md)

mexem 提供：
- 按 DAG 依赖编排启动所有节点
- `mexem status` 实时查看所有节点健康状态
- `mexem stop-node/start-node/restart-node` 动态控制单个节点
- 集成 msafety 心跳检测节点故障

### P2: mparam（参数系统）

| 模块 | 依赖 | 场景 |
|------|------|------|
| mparam | mshm, mcom | 运行时参数读写、变更通知 |

### P3: 架构清理

详见 [doc/dev/arch_cleanup.md](dev/arch_cleanup.md)

- msomeip / mrouting 职责合并
- mshm eventfd 多客户端修复
- config_server 数据持久化

### P4: mupgrade 远程管理 CLI ⭐ 新增

| 模块 | 状态 | 依赖 | 说明 |
|------|------|------|------|
| **mupgrade** | ✅ 已完成 | mlog, mexem | SSH 远程节点管理、升级、监控 |

mupgrade 提供：
- 节点管理：`mupgrade node start/stop/restart/list`
- 升级管理：`mupgrade upgrade start/progress/rollback`（原子升级 + 自动回滚）
- 日志管理：`mupgrade log get/stream`
- 配置管理：`mupgrade config get/set`
- 系统监控：`mupgrade system status/top/degrade`
- 权限分级：READONLY/OPERATOR/ADMIN（映射 Linux 用户组）
- 审计日志：所有操作记录到 `/var/log/mupgrade/audit.log`

GitHub: https://github.com/moss339/mupgrade

---

## 未来模块规划

以下模块已纳入未来开发计划，按交付优先级分阶段实施：

### Phase 1: 交付必需 (P0) ✅ 已完成

| 模块 | 功能描述 | 依赖 | 状态 |
|------|----------|------|------|
| **mdiag** | UDS 诊断模块，支持 ISO 14229 标准 | mcom, mlog | ✅ 已完成 |
| **mrecord** | 数据录制/回放模块 (MCAP 格式) | mdds, mlog | ✅ 已完成 |

### Phase 2: 生产就绪 (P1) ✅ 已完成

| 模块 | 功能描述 | 依赖 | 状态 |
|------|----------|------|------|
| **mperf** | 性能监控模块，Prometheus 格式导出 | mlog, mruntime | ✅ 已完成 |
| **mtls** | 安全传输层 (TLS/DTLS) | mcom, mlog | ✅ 已完成 |

### Phase 3: 生态集成 (P2) ✅ 已完成

| 模块 | 功能描述 | 依赖 | 状态 |
|------|----------|------|------|
| **mbridge** | 外部系统桥接 (ROS2/AUTOSAR AP) | mcom, mdds, msomeip | ✅ 已完成 |
| **mcal** | 标定模块，在线标定接口 | mparam, mcom | ✅ 已完成 |

#### mdiag 详细规划

```
功能需求:
├── UDS (Unified Diagnostic Services) 协议支持
├── DTC (Diagnostic Trouble Code) 管理
├── 故障码存储与检索
├── Snapshot 数据捕获
└── 符合 ISO 14229 标准
```

#### mrecord 详细规划

```
功能需求:
├── Topic 数据录制 (MCAP 格式)
├── 时间索引与回放
├── 循环缓冲存储
├── 触发式录制 (事件触发)
└── 元数据管理
```

#### mperf 详细规划

```
功能需求:
├── 节点 CPU/内存实时监控
├── Topic 吞吐量/延迟统计
├── 通信链路质量
├── 性能计数器导出 (Prometheus 格式)
└── 热点分析接口
```

#### mtls 详细规划

```
功能需求:
├── TLS/DTLS 加密通道
├── 证书管理 (X.509)
├── 密钥交换与轮换
├── 安全会话管理
└── HSM 集成接口
```

#### mbridge 详细规划

```
功能需求:
├── ROS2 桥接 (rclcpp <-> mcom)
├── AUTOSAR Adaptive 桥接
├── SOME/IP <-> DDS 转换
└── 消息格式自动映射
```

#### mcal 详细规划 ✅ 已完成

```
功能需求:
├── 参数管理 (CalParam, CalRegistry)
├── 在线标定接口 (CalServer, CalClient)
├── 版本管理 (CalStorage)
├── 约束验证 (CalValidator)
└── 支持 INT/DOUBLE/BOOL/STRING/ARRAY/MATRIX 类型
```

---

## 文档更新记录

| 日期 | 更新内容 |
|------|----------|
| 2026-03-29 | 初始化里程碑文档，MVisual 优先级 P0→P3 |
| 2026-03-29 | 新增 mcom 模块，mservice/maction 合并，架构重组 |
| 2026-04-04 | 新增开发计划，新增 mexem 模块，P0 Bug 修复计划，架构清理计划 |
| 2026-04-13 | msomeip 模块开发：修复4个Bug（stop崩溃、TCP通路、TP类型丢失、SD Option长度），集成TP分片/重组，集成ShmAgent |
| 2026-04-19 | 新增 mupgrade 模块：远程管理 CLI，支持节点管理、升级、日志、配置、监控；GitHub 仓库已创建 |
| 2026-04-19 | 新增未来模块规划：mdiag、mrecord、mperf、mtls、mbridge、mcal |
| 2026-04-21 | **mcal 模块完成**：在线标定模块，支持参数管理、版本控制、约束验证；Phase 1-3 全部完成 |
