# MService 模块设计文档

## 1. 概述

MService 是 MOSS 项目的服务调用模块，提供一对一同步 Request/Response 服务调用，支持超时和错误处理。基于现有 msomeip 协议栈，复用 ServiceProxy/ServiceSkeleton 模式。

### 1.1 设计目标

- **同步/异步调用**: 支持阻塞等待和 Future 异步模式
- **超时控制**: 可配置的服务调用超时
- **错误处理**: 完善的错误码和异常处理
- **轻量级**: 最小化依赖，适合嵌入式系统

### 1.2 术语表

| 术语 | 说明 |
|------|------|
| Request | 服务请求，包含方法ID和负载数据 |
| Response | 服务响应，包含返回数据和错误码 |
| ServiceClient | 服务客户端，发起请求 |
| ServiceServer | 服务端，处理请求并返回响应 |
| Method ID | 方法标识符 |

---

## 2. 架构设计

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        mservice 架构                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────┐         ┌─────────────────────┐          │
│  │   ServiceClient     │         │   ServiceServer     │          │
│  │                     │◀═══════▶│                     │          │
│  │  send_request()    │  SOME/IP│  handle_request()   │          │
│  │  async_call()      │  Method │  send_response()    │          │
│  └─────────────────────┘         └─────────────────────┘          │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    ServiceManager                            │   │
│  │  - Service Registry (service_id → handler)                  │   │
│  │  - Pending Request Map (request_id → promise)              │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 目录结构

```
mservice/
├── include/mservice/
│   ├── mservice.h              # 主头文件
│   ├── types.h                 # 类型定义
│   ├── service_client.h        # 服务客户端
│   ├── service_server.h        # 服务端
│   ├── service_manager.h       # 服务管理器
│   └── error.h                 # 错误码定义
├── src/
│   ├── service_client.cpp
│   ├── service_server.cpp
│   ├── service_manager.cpp
│   └── CMakeLists.txt
└── test/
    └── test_service.cpp
```

---

## 3. 核心设计

### 3.1 SOME/IP 消息映射

| mservice 操作 | SOME/IP MessageType | 说明 |
|--------------|---------------------|------|
| Request | REQUEST (0x00) | 带响应的请求 |
| RequestNoReturn | REQUEST_NO_RETURN (0x01) | 无响应请求 |
| Response | RESPONSE (0x80) | 成功响应 |
| Error | ERROR (0x81) | 错误响应 |

### 3.2 类型定义

```cpp
enum class ServiceError : uint8_t {
    OK = 0x00,
    TIMEOUT = 0x01,
    SERVICE_NOT_FOUND = 0x02,
    METHOD_NOT_FOUND = 0x03,
    INVALID_REQUEST = 0x04,
    INTERNAL_ERROR = 0x05,
    CANCELED = 0x06,
};

struct Request {
    RequestHeader header;
    std::vector<uint8_t> payload;
};

struct Response {
    ResponseHeader header;
    std::vector<uint8_t> payload;
};

using RequestHandler = std::function<Response(const Request& request)>;
```

### 3.3 ServiceClient 类

```cpp
class ServiceClient : public std::enable_shared_from_this<ServiceClient> {
public:
    ServiceClient(std::shared_ptr<msomeip::Application> app,
                  ServiceId service_id, InstanceId instance_id);
    ~ServiceClient();

    void init();

    // 同步调用
    std::optional<Response> send_request(
        MethodId method_id,
        const std::vector<uint8_t>& payload,
        std::chrono::milliseconds timeout = std::chrono::milliseconds(5000));

    // 异步调用
    std::future<std::optional<Response>> send_request_async(
        MethodId method_id,
        const std::vector<uint8_t>& payload,
        std::chrono::milliseconds timeout = std::chrono::milliseconds(5000));

    // 无返回调用
    bool send_request_no_return(MethodId method_id,
                                 const std::vector<uint8_t>& payload);

    bool is_service_available() const;
};
```

### 3.4 ServiceServer 类

```cpp
class ServiceServer : public std::enable_shared_from_this<ServiceServer> {
public:
    ServiceServer(std::shared_ptr<msomeip::Application> app,
                  ServiceId service_id, InstanceId instance_id);
    ~ServiceServer();

    void init();

    // 注册方法处理器
    void register_method(MethodId method_id, RequestHandler handler);
    void unregister_method(MethodId method_id);

    void offer();
    void stop_offer();

    void send_response(const Request& request,
                       const std::vector<uint8_t>& payload);
    void send_error_response(const Request& request, ServiceError error);
};
```

---

## 4. 使用示例

### 4.1 服务端示例

```cpp
class CalculatorServer {
public:
    CalculatorServer(std::shared_ptr<msomeip::Application> app)
        : server_(app, 0x1234, 0x0001) {}

    void init() {
        server_.register_method(0x0001, [this](const mservice::Request& req) {
            int32_t a = read_int32(req.payload, 0);
            int32_t b = read_int32(req.payload, 4);
            int32_t result = a + b;

            std::vector<uint8_t> resp;
            write_int32(resp, result);
            return mservice::Response{{0, 0x1234, 0x0001, 0x0001},
                                       mservice::ServiceError::OK, resp};
        });
        server_.init();
        server_.offer();
    }

private:
    mservice::ServiceServer server_;
};
```

### 4.2 客户端示例

```cpp
class CalculatorClient {
public:
    CalculatorClient(std::shared_ptr<msomeip::Application> app)
        : client_(app, 0x1234, 0x0001) {}

    void init() { client_.init(); }

    std::optional<int32_t> add(int32_t a, int32_t b) {
        std::vector<uint8_t> payload;
        write_int32(payload, a);
        write_int32(payload, b);

        auto response = client_.send_request(0x0001, payload);
        if (!response) return std::nullopt;
        return read_int32(response->payload, 0);
    }

private:
    mservice::ServiceClient client_;
};
```

---

## 5. 依赖关系

```
mservice
├── msomeip (必需)
│   ├── Application
│   └── ServiceProxy/ServiceSkeleton
└── mshm (可选，用于本地进程间通信)
```

---

## 6. 构建配置

```bash
cd mservice && mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j4
```
