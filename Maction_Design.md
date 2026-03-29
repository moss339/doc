# MAction 模块设计文档

## 1. 概述

MAction 是 MOSS 项目的动作模块，提供长时间运行任务的 Goal/Result/Feedback 机制，支持取消和进度跟踪。基于 msomeip Method + Event 组合实现。

### 1.1 设计目标

- **可取消**: 支持长时间任务的取消操作
- **进度反馈**: 实时反馈动作执行进度
- **状态跟踪**: 完整的状态机管理
- **超时控制**: 可配置的执行超时

### 1.2 术语表

| 术语 | 说明 |
|------|------|
| Goal | 动作目标，一个 action 的起始请求 |
| Result | 动作结果，动作完成后的最终响应 |
| Feedback | 动作反馈，动作执行过程中的进度更新 |
| Action Server | 动作服务器，处理 goal 请求并执行动作 |
| Action Client | 动作客户端，发送 goal 并接收 result/feedback |
| GoalHandle | Goal 句柄，用于跟踪和管理 Goal |

---

## 2. 架构设计

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        maction 架构                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────┐         ┌─────────────────────┐          │
│  │   ActionClient      │         │   ActionServer     │          │
│  │                     │◀════════▶│                     │          │
│  │  send_goal()       │  SOME/IP │  execute_goal()    │          │
│  │  cancel_goal()      │  Method  │  publish_feedback │          │
│  │  get_result()      │    +     │  send_result()    │          │
│  │  subscribe_fb()     │  Event   │  cancel_goal()   │          │
│  └─────────────────────┘         └─────────────────────┘          │
│                 │                           │                       │
│                 │     ┌─────────────────┐   │                       │
│                 │     │  ActionState    │   │                       │
│                 │     │  - ACTIVE       │   │                       │
│                 │     │  - SUCCEEDED    │   │                       │
│                 │     │  - ABORTED      │   │                       │
│                 │     │  - CANCELED     │   │                       │
│                 │     └─────────────────┘   │                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 目录结构

```
maction/
├── include/maction/
│   ├── maction.h              # 主头文件
│   ├── types.h                 # 类型定义
│   ├── action_client.h        # 动作客户端
│   ├── action_server.h        # 动作服务器
│   ├── action_manager.h       # 动作管理器
│   ├── goal_handle.h          # Goal 句柄
│   └── error.h                # 错误码定义
├── src/
│   ├── action_client.cpp
│   ├── action_server.cpp
│   ├── action_manager.cpp
│   ├── goal_handle.cpp
│   └── CMakeLists.txt
└── test/
    └── test_action.cpp
```

---

## 3. 核心设计

### 3.1 Action 状态机

```
                        ┌─────────────────┐
                        │     READY       │
                        └────────┬────────┘
                                 │ accept_goal()
                                 ▼
                        ┌─────────────────┐
            ┌───────────│    EXECUTING    │───────────┐
            │           └────────┬────────┘           │
            │                    │                    │
            │         ┌──────────┼──────────┐        │
            │         ▼          ▼          ▼        │
            │   ┌──────────┐ ┌──────────┐ ┌──────────┐│
            │   │PREEMPTING│ │SUCCEEDING│ │ABORTING  ││
            │   └────┬─────┘ └────┬─────┘ └────┬─────┘│
            │        │            │            │      │
            │        └────────────┼────────────┘      │
            │                     ▼                   │
            │               ┌──────────┐              │
            │               │SUCCEEDED│              │
            │               └──────────┘              │
            ▼                                          │
                        ┌─────────────────┐            │
                        │    CANCELED    │◀───────────┘
                        └─────────────────┘
```

### 3.2 SOME/IP 消息映射

| maction 操作 | SOME/IP 类型 | Method/Event ID |
|-------------|-------------|-----------------|
| send_goal | Method REQUEST | 0x0001 |
| cancel_goal | Method REQUEST | 0x0002 |
| get_result | Method REQUEST | 0x0003 |
| feedback | Event | 0x8001 |
| status_update | Event | 0x8002 |

### 3.3 类型定义

```cpp
enum class ActionStatus : uint8_t {
    READY = 0x00,
    EXECUTING = 0x01,
    PREEMPTING = 0x02,
    PREEMPTED = 0x03,
    SUCCEEDING = 0x04,
    SUCCEEDED = 0x05,
    ABORTING = 0x06,
    ABORTED = 0x07,
    CANCELED = 0x08,
    REJECTED = 0x09,
};

struct GoalInfo {
    uint32_t goal_id;
    uint64_t timestamp;
    std::vector<uint8_t> goal_data;
};

struct ResultInfo {
    uint32_t goal_id;
    ActionStatus status;
    std::vector<uint8_t> result_data;
};

struct FeedbackInfo {
    uint32_t goal_id;
    float progress;
    std::vector<uint8_t> feedback_data;
};

using GoalHandler = std::function<std::tuple<bool, ActionStatus, std::vector<uint8>>(
    const GoalInfo& goal)>;
using FeedbackHandler = std::function<void(const FeedbackInfo& feedback)>;
```

### 3.4 GoalHandle 类

```cpp
class GoalHandle {
public:
    explicit GoalHandle(uint32_t goal_id);
    ~GoalHandle();

    uint32_t get_goal_id() const { return goal_id_; }
    ActionStatus get_status() const;
    void set_status(ActionStatus status);

    bool is_terminal() const {
        auto status = get_status();
        return status == ActionStatus::SUCCEEDED ||
               status == ActionStatus::ABORTED ||
               status == ActionStatus::CANCELED;
    }

    bool cancel_requested() const { return cancel_requested_.load(); }
    void request_cancel() { cancel_requested_.store(true); }

    void set_result(ActionStatus status, const std::vector<uint8_t>& result);
    bool has_result() const;
    ResultInfo get_result() const;
};
```

### 3.5 ActionClient 类

```cpp
class ActionClient : public std::enable_shared_from_this<ActionClient> {
public:
    ActionClient(std::shared_ptr<msomeip::Application> app,
                 ActionClientConfig config);
    ~ActionClient();

    void init();

    std::shared_ptr<GoalHandle> send_goal(
        const std::vector<uint8_t>& goal_data,
        std::chrono::milliseconds timeout = std::chrono::milliseconds(5000));

    std::future<std::shared_ptr<GoalHandle>> send_goal_async(
        const std::vector<uint8_t>& goal_data);

    bool cancel_goal(uint32_t goal_id);

    std::optional<ResultInfo> get_result(
        uint32_t goal_id,
        std::chrono::milliseconds timeout = std::chrono::milliseconds(5000));

    void subscribe_feedback(FeedbackHandler handler);
    void unsubscribe_feedback();
};
```

### 3.6 ActionServer 类

```cpp
class ActionServer : public std::enable_shared_from_this<ActionServer> {
public:
    ActionServer(std::shared_ptr<msomeip::Application> app,
                 ActionServerConfig config);
    ~ActionServer();

    void init();

    void register_goal_handler(GoalHandler handler);
    void register_cancel_handler(CancelHandler handler);

    void start();
    void stop();

    void publish_feedback(uint32_t goal_id, float progress,
                         const std::vector<uint8_t>& feedback_data);
    void publish_status_update(uint32_t goal_id, ActionStatus status);
    void send_result(uint32_t goal_id, ActionStatus status,
                     const std::vector<uint8_t>& result_data);

    void accept_goal(uint32_t goal_id);
    void reject_goal(uint32_t goal_id);
};
```

---

## 4. 使用示例

### 4.1 动作服务端示例

```cpp
class NavigationActionServer {
public:
    NavigationActionServer(std::shared_ptr<msomeip::Application> app)
        : server_(app, {0x5678, 0x0001, 0x0001}) {}

    void init() {
        server_.register_goal_handler(
            [this](const maction::GoalInfo& goal) {
                for (int i = 0; i <= 10; i++) {
                    if (goal_handle_->cancel_requested()) {
                        server_.send_result(goal.goal_id,
                                          maction::ActionStatus::CANCELED, {});
                        return {false, maction::ActionStatus::CANCELED, {}};
                    }

                    float progress = i / 10.0f;
                    server_.publish_feedback(goal.goal_id, progress, {});
                    std::this_thread::sleep_for(std::chrono::seconds(1));
                }

                server_.send_result(goal.goal_id,
                                   maction::ActionStatus::SUCCEEDED,
                                   serialize_result("Done"));
                return {true, maction::ActionStatus::SUCCEEDED, {}};
            });

        server_.init();
        server_.start();
    }

private:
    std::shared_ptr<maction::GoalHandle> goal_handle_;
    maction::ActionServer server_;
};
```

### 4.2 动作客户端示例

```cpp
class NavigationActionClient {
public:
    NavigationActionClient(std::shared_ptr<msomeip::Application> app)
        : client_(app, {0x5678, 0x0001, 0x0001}) {}

    void init() {
        client_.subscribe_feedback(
            [](const maction::FeedbackInfo& fb) {
                std::cout << "Progress: " << (fb.progress * 100) << "%\n";
            });
        client_.init();
    }

    void navigate_to(float x, float y, float z) {
        auto goal_handle = client_.send_goal(serialize_target(x, y, z));
        if (!goal_handle) {
            std::cerr << "Goal rejected\n";
            return;
        }

        while (!goal_handle->is_terminal()) {
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }

        if (goal_handle->has_result()) {
            auto result = goal_handle->get_result();
            std::cout << "Navigation finished: " << (int)result.status << "\n";
        }
    }

private:
    maction::ActionClient client_;
};
```

---

## 5. 依赖关系

```
maction
├── msomeip (必需)
│   ├── Application
│   └── Method/Event
├── mservice (必需)
│   └── ServiceClient/ServiceServer
└── mshm (可选)
```

---

## 6. 构建配置

```bash
cd maction && mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j4
```
