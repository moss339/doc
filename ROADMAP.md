# MOSS 项目里程碑路线图

> 最后更新：2026-03-29

---

## 项目愿景

MOSS (Middleware for Object-Oriented Shared Services) 是面向汽车/嵌入式系统的轻量级中间件，目标是为车载感知、规划、控制提供高性能数据分发与服务通信基础设施。

---

## 模块现状总览

| 模块 | 语言 | 状态 | 依赖 |
|------|------|------|------|
| mshm | C | ✅ 已完成 | 底层依赖 |
| **mdds** | C++17 | ✅ 已完成 | mshm |
| **mcom** | C++17 | 🔄 开发中 | mdds, mshm |
| mruntime | C++17 | ✅ 已完成 | 独立 |
| mlog | C++20 | ✅ 已完成 | 独立 |
| msafety | C++17 | ✅ 已完成 | 独立 |
| msomeip | C++17 | 🔄 开发中 | mshm |
| mrouting | C++17 | 🔄 开发中 | msomeip, mshm |
| config_server | C++17 | 🔄 开发中 | 独立 |
| **mvisual** | C++17 | **📋 规划中** | mshm, mruntime, mlog, mcom |
| mlaunch | C++17 | 📋 规划中 | mshm, mlog, msafety, mcom |
| mparam | C++17 | 📋 规划中 | mshm |

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

## 后续模块优先级（参考）

```
mcom 和 mvisual 完成后的扩展方向：

1. mlaunch（启动系统）
   └─ 依赖：mshm, mlog, msafety, mcom
   └─ 场景：多节点进程管理、DAG 调度

2. mparam（参数系统）
   └─ 依赖：mshm, mcom
   └─ 场景：运行时参数读写、变更通知
```

---

## 文档更新记录

| 日期 | 更新内容 |
|------|----------|
| 2026-03-29 | 初始化里程碑文档，MVisual 优先级 P0→P3 |
| 2026-03-29 | 新增 mcom 模块，mservice/maction 合并，架构重组 |
