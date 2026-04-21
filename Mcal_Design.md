# MCAL 在线标定模块设计文档

> 版本: v1.0
> 日期: 2026-04-21
> 状态: Draft

---

## 一、概述

**mcal** (MOSS Calibration) 是 MOSS 的在线标定模块，提供运行时参数标定能力，支持感知/控制算法的实时调参。

### 核心功能

| 功能 | 描述 |
|------|------|
| 参数管理 | 标定参数定义、存储、版本控制 |
| 在线标定 | 运行时通过 RPC 接口调整参数 |
| 版本管理 | 标定数据版本控制、回滚支持 |
| 精度验证 | 标定结果验证与报告生成 |

---

## 二、架构设计

### 2.1 模块层次

```
                    ┌─────────────────┐
                    │   CalClient     │  (用户侧客户端)
                    └────────┬────────┘
                             │ RPC
                    ┌────────▼────────┐
                    │   CalServer     │  (标定服务端)
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌───────▼───────┐   ┌───────▼───────┐   ┌───────▼───────┐
│  CalRegistry  │   │ CalStorage    │   │ CalValidator  │
│  (参数注册表) │   │ (持久化存储)  │   │ (精度验证器)  │
└───────────────┘   └───────────────┘   └───────────────┘
```

### 2.2 依赖关系

```
mcal
├── mparam (参数存储基础)
├── mcom (RPC 通信)
└── mlog (日志)
```

### 2.3 目录结构

```
mcal/
├── include/mcal/
│   ├── mcal.h              # 主头文件
│   ├── types.h             # 类型定义
│   ├── cal_param.h         # 标定参数定义
│   ├── cal_registry.h      # 参数注册表
│   ├── cal_server.h        # 标定服务端
│   ├── cal_client.h        # 标定客户端
│   ├── cal_storage.h       # 持久化存储
│   └── cal_validator.h     # 精度验证器
├── src/
│   ├── cal_param.cpp
│   ├── cal_registry.cpp
│   ├── cal_server.cpp
│   ├── cal_client.cpp
│   ├── cal_storage.cpp
│   └── cal_validator.cpp
├── test/
│   ├── test_cal_param.cpp
│   ├── test_cal_registry.cpp
│   ├── test_cal_server.cpp
│   └── test_cal_client.cpp
├── proto/
│   └── cal.proto           # RPC 消息定义
├── CMakeLists.txt
└── README.md
```

---

## 三、核心类型定义

### 3.1 types.h

```cpp
namespace moss {
namespace mcal {

constexpr const char* MCAL_VERSION = "1.0.0";

// 标定参数类型
enum class CalParamType : uint8_t {
    INT = 0,
    DOUBLE = 1,
    BOOL = 2,
    STRING = 3,
    INT_ARRAY = 4,
    DOUBLE_ARRAY = 5,
    MATRIX = 6       // 二维矩阵 (用于相机内参等)
};

// 标定参数约束
struct CalConstraint {
    bool has_min;
    bool has_max;
    double min_value;
    double max_value;
    std::vector<int> matrix_shape;  // for MATRIX type

    CalConstraint() : has_min(false), has_max(false),
                      min_value(0), max_value(0) {}
};

// 标定状态
enum class CalStatus : uint8_t {
    OK = 0,            // 正常
    OUT_OF_RANGE = 1,  // 超出约束
    INVALID = 2,       // 无效值
    LOCKED = 3         // 锁定不可修改
};

// 错误码
enum class CalError {
    OK = 0,
    PARAM_NOT_FOUND = 1,
    INVALID_VALUE = 2,
    CONSTRAINT_VIOLATION = 3,
    VERSION_MISMATCH = 4,
    STORAGE_ERROR = 5,
    RPC_ERROR = 6,
    LOCKED = 7,
};

inline const char* cal_error_str(CalError err) {
    switch (err) {
        case CalError::OK: return "OK";
        case CalError::PARAM_NOT_FOUND: return "Parameter not found";
        case CalError::INVALID_VALUE: return "Invalid value";
        case CalError::CONSTRAINT_VIOLATION: return "Constraint violation";
        case CalError::VERSION_MISMATCH: return "Version mismatch";
        case CalError::STORAGE_ERROR: return "Storage error";
        case CalError::RPC_ERROR: return "RPC error";
        case CalError::LOCKED: return "Parameter locked";
        default: return "Unknown error";
    }
}

}  // namespace mcal
}  // namespace moss
```

### 3.2 cal_param.h

```cpp
namespace moss {
namespace mcal {

// 标定参数值
using CalValue = std::variant<
    int64_t,                    // INT
    double,                     // DOUBLE
    bool,                       // BOOL
    std::string,                // STRING
    std::vector<int64_t>,       // INT_ARRAY
    std::vector<double>,        // DOUBLE_ARRAY
    std::vector<std::vector<double>>  // MATRIX
>;

// 标定参数
struct CalParam {
    std::string key;            // 参数标识
    std::string group;          // 所属分组 (如 "camera", "lidar")
    std::string description;    // 描述
    CalParamType type;
    CalValue value;
    CalValue default_value;     // 默认值
    CalConstraint constraint;   // 约束条件
    int64_t version;            // 版本号
    int64_t timestamp_us;       // 最后修改时间
    CalStatus status;
    uint32_t flags;             // 位标志 (locked, read-only, etc.)

    // 标定相关
    double confidence;          // 置信度 [0, 1]
    std::string calibration_note; // 标定备注

    CalParam();
    bool validate() const;
    bool check_constraint(const CalValue& val) const;
};

// 参数变更记录
struct CalChange {
    std::string key;
    CalValue old_value;
    CalValue new_value;
    int64_t old_version;
    int64_t new_version;
    int64_t timestamp_us;
    std::string changed_by;     // 修改者标识
    std::string reason;         // 修改原因
};

}  // namespace mcal
}  // namespace moss
```

---

## 四、核心组件设计

### 4.1 CalRegistry - 参数注册表

负责管理所有标定参数的定义和生命周期。

```cpp
class CalRegistry {
public:
    CalRegistry();
    ~CalRegistry();

    // 参数注册
    bool register_param(const CalParam& param);
    bool unregister_param(const std::string& key);

    // 参数查询
    bool has(const std::string& key) const;
    std::optional<CalParam> get(const std::string& key) const;
    std::vector<CalParam> get_all() const;
    std::vector<CalParam> get_by_group(const std::string& group) const;
    std::vector<std::string> get_groups() const;

    // 参数更新
    CalError set(const std::string& key, const CalValue& value,
                 const std::string& changed_by = "",
                 const std::string& reason = "");

    // 重置
    CalError reset(const std::string& key);
    void reset_all();

    // 锁定/解锁
    bool lock(const std::string& key);
    bool unlock(const std::string& key);
    bool is_locked(const std::string& key) const;

    // 变更监听
    using ChangeCallback = std::function<void(const CalChange&)>;
    size_t add_watcher(const std::string& key, ChangeCallback cb);
    size_t add_global_watcher(ChangeCallback cb);
    void remove_watcher(size_t id);

private:
    std::map<std::string, CalParam> params_;
    std::map<std::string, std::vector<ChangeCallback>> watchers_;
    std::vector<ChangeCallback> global_watchers_;
    size_t next_watcher_id_;
    mutable std::mutex mutex_;
};
```

### 4.2 CalServer - 标定服务端

提供 RPC 接口，响应客户端的标定请求。

```cpp
class CalServer : public std::enable_shared_from_this<CalServer> {
public:
    static std::shared_ptr<CalServer> create(
        uint16_t service_id = 0x8001,
        uint16_t instance_id = 0x0001);

    CalServer(uint16_t service_id, uint16_t instance_id);
    ~CalServer();

    // 生命周期
    bool init();
    void start();
    void stop();
    void destroy();

    // 注册表访问
    CalRegistry& registry();

    // 存储管理
    bool load_from_file(const std::string& path);
    bool save_to_file(const std::string& path);
    bool enable_auto_save(const std::string& path, int interval_sec = 60);

    // 验证器
    void set_validator(std::shared_ptr<CalValidator> validator);

private:
    // RPC 方法处理器
    void handle_get_param(const Request& req, Response& resp);
    void handle_set_param(const Request& req, Response& resp);
    void handle_list_params(const Request& req, Response& resp);
    void handle_reset_param(const Request& req, Response& resp);
    void handle_get_history(const Request& req, Response& resp);

    std::unique_ptr<CalRegistry> registry_;
    std::unique_ptr<CalStorage> storage_;
    std::shared_ptr<CalValidator> validator_;
    std::shared_ptr<ServiceServer> rpc_server_;
    std::thread auto_save_thread_;
    std::atomic<bool> running_;
};

using CalServerPtr = std::shared_ptr<CalServer>;
```

### 4.3 CalClient - 标定客户端

用户侧客户端，提供便捷的标定接口。

```cpp
class CalClient {
public:
    static std::shared_ptr<CalClient> create(
        uint16_t service_id = 0x8001,
        uint16_t instance_id = 0x0001);

    CalClient(uint16_t service_id, uint16_t instance_id);
    ~CalClient();

    // 连接管理
    bool connect();
    void disconnect();
    bool is_connected() const;

    // 参数获取
    std::optional<CalParam> get(const std::string& key);
    std::future<std::optional<CalParam>> async_get(const std::string& key);
    std::vector<CalParam> list_all();
    std::vector<CalParam> list_by_group(const std::string& group);

    // 参数设置
    CalError set(const std::string& key, const CalValue& value,
                 const std::string& reason = "");
    std::future<CalError> async_set(const std::string& key,
                                     const CalValue& value,
                                     const std::string& reason = "");

    // 重置
    CalError reset(const std::string& key);

    // 批量操作
    CalError set_batch(const std::map<std::string, CalValue>& params,
                       const std::string& reason = "");

    // 变更监听
    using ChangeCallback = std::function<void(const CalChange&)>;
    size_t subscribe(const std::string& key, ChangeCallback cb);
    void unsubscribe(size_t subscription_id);

    // 模板便捷方法
    template<typename T>
    std::optional<T> get_value(const std::string& key);

    template<typename T>
    CalError set_value(const std::string& key, const T& value,
                       const std::string& reason = "");

private:
    std::shared_ptr<ServiceClient> rpc_client_;
    bool connected_;
};

using CalClientPtr = std::shared_ptr<CalClient>;
```

### 4.4 CalStorage - 持久化存储

负责标定数据的持久化和版本管理。

```cpp
class CalStorage {
public:
    CalStorage();
    ~CalStorage();

    // 文件操作
    bool save(const std::string& path, const CalRegistry& registry);
    bool load(const std::string& path, CalRegistry& registry);

    // 版本管理
    bool save_version(const std::string& path, const CalRegistry& registry,
                      const std::string& version_name);
    std::vector<std::string> list_versions(const std::string& path);
    bool load_version(const std::string& path, const std::string& version_name,
                      CalRegistry& registry);
    bool delete_version(const std::string& path, const std::string& version_name);

    // 回滚
    bool rollback(const std::string& path, int steps, CalRegistry& registry);
    bool rollback_to_version(const std::string& path,
                             const std::string& version_name,
                             CalRegistry& registry);

    // 历史记录
    bool append_history(const std::string& path, const CalChange& change);
    std::vector<CalChange> get_history(const std::string& path,
                                       const std::string& key = "",
                                       int max_entries = 100);

private:
    std::string history_path_;
};
```

### 4.5 CalValidator - 精度验证器

验证标定结果的正确性和精度。

```cpp
class CalValidator {
public:
    virtual ~CalValidator() = default;

    // 验证单个参数
    virtual CalError validate(const std::string& key, const CalValue& value);

    // 验证参数组
    virtual CalError validate_group(const std::string& group,
                                    const std::map<std::string, CalValue>& params);

    // 生成验证报告
    struct ValidationReport {
        std::string group;
        bool passed;
        std::vector<std::string> errors;
        std::vector<std::string> warnings;
        double overall_confidence;
        int64_t timestamp_us;
    };

    virtual ValidationReport generate_report(const std::string& group);

    // 注册自定义验证器
    using ValidatorFunc = std::function<CalError(const CalValue&)>;
    void register_validator(const std::string& key, ValidatorFunc func);
    void register_group_validator(const std::string& group,
                                  std::function<CalError(const std::map<std::string, CalValue>&)> func);

protected:
    std::map<std::string, ValidatorFunc> validators_;
    std::map<std::string, std::function<CalError(const std::map<std::string, CalValue>&)>> group_validators_;
};
```

---

## 五、RPC 接口定义

### 5.1 cal.proto

```protobuf
syntax = "proto3";

package moss.proto.cal;

// 标定参数值
message CalValueMsg {
    oneof value {
        int64 int_val = 1;
        double double_val = 2;
        bool bool_val = 3;
        string string_val = 4;
        IntArray int_array = 5;
        DoubleArray double_array = 6;
        Matrix matrix = 7;
    }
}

message IntArray {
    repeated int64 values = 1;
}

message DoubleArray {
    repeated double values = 1;
}

message Matrix {
    repeated double values = 1;  // 行优先存储
    int32 rows = 2;
    int32 cols = 3;
}

// 方法 ID 定义
enum CalMethodId {
    GET_PARAM = 1;
    SET_PARAM = 2;
    LIST_PARAMS = 3;
    RESET_PARAM = 4;
    GET_HISTORY = 5;
    LOCK_PARAM = 6;
    UNLOCK_PARAM = 7;
    BATCH_SET = 8;
}

// 获取参数
message GetParamRequest {
    string key = 1;
}

message GetParamResponse {
    int32 error = 1;
    string key = 2;
    CalValueMsg value = 3;
    int64 version = 4;
    string group = 5;
    string description = 6;
}

// 设置参数
message SetParamRequest {
    string key = 1;
    CalValueMsg value = 2;
    string changed_by = 3;
    string reason = 4;
    int64 expected_version = 5;  // 乐观锁
}

message SetParamResponse {
    int32 error = 1;
    int64 new_version = 2;
    int64 timestamp_us = 3;
}

// 列出参数
message ListParamsRequest {
    string group = 1;  // 可选过滤
}

message ListParamsResponse {
    int32 error = 1;
    repeated GetParamResponse params = 2;
}

// 重置参数
message ResetParamRequest {
    string key = 1;
}

message ResetParamResponse {
    int32 error = 1;
    CalValueMsg default_value = 2;
    int64 new_version = 3;
}

// 获取历史
message GetHistoryRequest {
    string key = 1;       // 可选过滤
    int32 max_entries = 2;
}

message CalChangeMsg {
    string key = 1;
    CalValueMsg old_value = 2;
    CalValueMsg new_value = 3;
    int64 old_version = 4;
    int64 new_version = 5;
    int64 timestamp_us = 6;
    string changed_by = 7;
    string reason = 8;
}

message GetHistoryResponse {
    int32 error = 1;
    repeated CalChangeMsg changes = 2;
}

// 批量设置
message BatchSetRequest {
    map<string, CalValueMsg> params = 1;
    string reason = 2;
}

message BatchSetResponse {
    int32 error = 1;
    map<string, int64> new_versions = 2;
    repeated string failed_keys = 3;
}
```

---

## 六、存储格式

### 6.1 JSON 格式 (calibration.json)

```json
{
  "version": "1.0.0",
  "timestamp_us": 1713705600000000,
  "groups": {
    "camera": {
      "description": "相机标定参数",
      "params": {
        "fx": {
          "type": "double",
          "value": 1000.5,
          "default": 1000.0,
          "constraint": {
            "min": 100.0,
            "max": 5000.0
          },
          "description": "焦距 X (像素)",
          "version": 5,
          "confidence": 0.95
        },
        "fy": {
          "type": "double",
          "value": 1002.3,
          "default": 1000.0,
          "constraint": {
            "min": 100.0,
            "max": 5000.0
          },
          "description": "焦距 Y (像素)",
          "version": 5,
          "confidence": 0.95
        },
        "distortion_coeffs": {
          "type": "double_array",
          "value": [-0.1, 0.05, 0.001, -0.002, 0.0],
          "default": [0.0, 0.0, 0.0, 0.0, 0.0],
          "description": "畸变系数 [k1, k2, p1, p2, k3]",
          "version": 3,
          "confidence": 0.88
        }
      }
    },
    "lidar": {
      "description": "激光雷达标定参数",
      "params": {
        "extrinsic_matrix": {
          "type": "matrix",
          "value": [1, 0, 0, 0.1, 0, 1, 0, 0.2, 0, 0, 1, 0.3, 0, 0, 0, 1],
          "shape": [4, 4],
          "description": "外参矩阵 (lidar -> vehicle)",
          "version": 2,
          "confidence": 0.92
        }
      }
    }
  }
}
```

### 6.2 版本文件格式 (versions/v1.0.0.json)

```
calibration_data/
├── current.json          # 当前生效版本
├── history.jsonl         # 变更历史 (JSON Lines)
└── versions/
    ├── v1.0.0.json       # 版本快照
    ├── v1.0.1.json
    └── v1.1.0.json
```

---

## 七、使用示例

### 7.1 服务端

```cpp
#include <mcal/mcal.h>

int main() {
    // 创建标定服务
    auto server = mcal::CalServer::create(0x8001, 0x0001);

    // 注册参数
    mcal::CalParam fx_param;
    fx_param.key = "camera.fx";
    fx_param.group = "camera";
    fx_param.type = mcal::CalParamType::DOUBLE;
    fx_param.value = 1000.0;
    fx_param.default_value = 1000.0;
    fx_param.constraint.has_min = true;
    fx_param.constraint.min_value = 100.0;
    fx_param.constraint.has_max = true;
    fx_param.constraint.max_value = 5000.0;
    fx_param.description = "焦距 X (像素)";

    server->registry().register_param(fx_param);

    // 从文件加载
    server->load_from_file("config/calibration.json");

    // 启动服务
    server->init();
    server->start();

    // 自动保存
    server->enable_auto_save("config/calibration.json", 60);

    // 运行...
    std::this_thread::sleep_for(std::chrono::hours(1));

    server->stop();
    return 0;
}
```

### 7.2 客户端

```cpp
#include <mcal/mcal.h>

int main() {
    auto client = mcal::CalClient::create(0x8001, 0x0001);

    if (!client->connect()) {
        std::cerr << "连接失败" << std::endl;
        return 1;
    }

    // 获取参数
    auto fx = client->get_value<double>("camera.fx");
    if (fx) {
        std::cout << "当前焦距: " << *fx << std::endl;
    }

    // 设置参数
    auto err = client->set_value<double>("camera.fx", 1050.5,
                                         "调整焦距以匹配新镜头");
    if (err == mcal::CalError::OK) {
        std::cout << "设置成功" << std::endl;
    }

    // 监听变更
    client->subscribe("camera.fx", [](const mcal::CalChange& change) {
        std::cout << "参数变更: " << change.key << std::endl;
        std::cout << "  旧值: " << change.old_value << std::endl;
        std::cout << "  新值: " << change.new_value << std::endl;
    });

    // 列出所有参数
    auto params = client->list_by_group("camera");
    for (const auto& p : params) {
        std::cout << p.key << " = " << p.value << std::endl;
    }

    return 0;
}
```

---

## 八、里程碑计划

| 里程碑 | 内容 | 预计工期 |
|--------|------|----------|
| mcal-ms1 | 架构设计 + 骨架搭建 | 1 天 |
| mcal-ms2 | CalParam + CalRegistry 实现 | 2 天 |
| mcal-ms3 | CalServer + CalClient 实现 | 2 天 |
| mcal-ms4 | CalStorage + 版本管理 | 1 天 |
| mcal-ms5 | 测试验证 | 1 天 |

**总计**: 约 7 天

---

## 九、验收标准

- [ ] 支持基本参数类型 (INT, DOUBLE, BOOL, STRING, ARRAY, MATRIX)
- [ ] 在线参数调整 RPC 接口可用
- [ ] 参数约束验证
- [ ] 标定数据 JSON 持久化
- [ ] 版本管理 (保存/加载/回滚)
- [ ] 变更历史记录
- [ ] 参数变更订阅通知
- [ ] 单元测试覆盖核心功能
