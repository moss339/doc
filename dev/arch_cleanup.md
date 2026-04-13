# MOSS 架构清理计划

> 日期: 2026-04-04

---

## 一、msomeip / mrouting 职责合并

### 1.1 问题

msomeip 和 mrouting 存在大量重复：
- `ShmAgentId` 枚举在两个模块中重复定义（值相同但独立维护）
- `ServiceEntry` 结构体重复
- mrouting 依赖 msomeip 但只使用其部分类型，未复用其实现
- 两个模块都有自己的 SOME/IP 服务发现逻辑

### 1.2 方案

合并策略：
1. 将 mrouting 的路由表管理逻辑移入 msomeip 的 `routing/` 子目录
2. 删除 mrouting 模块
3. 统一类型定义到 `msomeip/include/msomeip/types.h`
4. 更新所有依赖 mrouting 的代码，改为 include msomeip

### 1.3 影响范围

- mrouting 无外部消费者（仅 msomeip 内部使用）
- 合并后 msomeip 成为单一 SOME/IP 模块

---

## 二、mshm eventfd 多客户端修复

### 2.1 当前架构

```
Server                Client A              Client B
  │                      │                     │
  │ shm_open             │                     │
  │ ftruncate            │                     │
  │ mmap                 │                     │
  │ eventfd_create ─────►│                     │
  │                      │ dup(eventfd) ◄──────│
  │                      │                     │
  │ eventfd_write(fd) ──►│ A 收到               │
  │                  ───►│ B 收到 (竞争!)       │
```

问题：所有客户端共享同一个 eventfd，`eventfd_read()` 竞争。

### 2.2 修复方案

```
Server                Client A              Client B
  │                      │                     │
  │ shm_open             │                     │
  │ eventfd_create_A ───►│ A 专用 fd           │
  │ eventfd_create_B ───►│                     │ B 专用 fd
  │                      │                     │
  │ write(fd_A) ────────►│ A 收到              │
  │ write(fd_B) ──────────────────────────────►│ B 收到
```

### 2.3 shm_header 修改

```c
typedef struct {
    // ... 现有字段 ...

    // 客户端管理
    uint32_t client_count;
    int32_t client_eventfds[8];   // 最多 8 个客户端

    // ... 原有 lock 字段 ...
} shm_header_t;
```

客户端注册流程：
1. 客户端创建自己的 eventfd
2. 客户端通过 `shm_register_client()` 将 fd 写入 header
3. 服务端通知时遍历 `client_eventfds[]` 逐个写入

---

## 三、config_server 数据持久化

### 3.1 当前问题

配置仅存在于内存中的 `ConfigStore`。config_server 重启后所有配置丢失。

### 3.2 方案

```
config_server/
├── src/
│   └── config_server.cpp
└── data/
    └── config.json      # 持久化文件
```

持久化流程：
1. `init()` 时从 `data/config.json` 加载
2. `set()` 操作同时更新内存和文件
3. 支持 `--import <file>` 和 `--export <file>` 命令
4. 文件格式与现有 JSON 配置兼容

### 3.3 文件格式

```json
{
  "version": 1,
  "updated_at": "2026-04-04T10:00:00Z",
  "entries": [
    {
      "key": "control.max_throttle",
      "value": "0.8",
      "type": "double"
    },
    {
      "key": "safety.min_safe_distance",
      "value": "5.0",
      "type": "double"
    }
  ]
}
```

---

## 四、清理优先级

| 任务 | 优先级 | 周期 | 依赖 |
|------|--------|------|------|
| mshm eventfd 修复 | P1 | 3天 | 无 |
| config_server 持久化 | P2 | 2天 | 无 |
| msomeip/mrouting 合并 | P3 | 5天 | 无 |
