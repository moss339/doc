# 日志模块设计文档

## 1. 概述

mlog 模块是 MOSS 项目的日志管理核心组件，提供高性能、高并发、线程安全的日志输出能力。

### 1.1 设计目标

- **高性能**: 异步日志写入，避免阻塞业务线程
- **高并发**: 支持多线程同时写入，消息不丢失、不乱序
- **输出有序**: 跨线程消息按全局序列号排序，保证日志完整性
- **多实例支持**: 工厂模式，一个进程可创建多个独立日志实例
- **配置灵活**: 支持 JSON 配置文件，热加载
- **格式统一**: 使用 fmt 库支持 `{}` 格式化风格

### 1.2 术语表

| 术语 | 说明 |
|------|------|
| Sink | 日志输出目标抽象接口 |
| FileSink | 文件输出，支持滚动 |
| ConsoleSink | 控制台输出 |
| AsyncQueue | 异步日志队列（MPSC 环冲区） |
| LogLevel | 日志级别枚举 |
| Formatter | 日志格式化器 |
| LoggerFactory | 日志实例工厂，支持多实例管理 |

---

## 2. 架构设计

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                              mlog                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    LoggerFactory                              │  │
│  │                  (实例工厂，支持多实例)                          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                               │                                     │
│         ┌─────────────────────┼─────────────────────┐             │
│         ▼                     ▼                     ▼             │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐     │
│  │ main_logger  │      │ perf_logger  │      │ debug_logger │     │
│  │  (实例1)     │      │  (实例2)     │      │  (实例3)     │     │
│  └──────┬───────┘      └──────┬───────┘      └──────┬───────┘     │
│         │                     │                     │              │
│         ▼                     ▼                     ▼              │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐     │
│  │  AsyncQueue  │      │  AsyncQueue  │      │  AsyncQueue  │     │
│  │  (独立队列)   │      │  (独立队列)   │      │  (独立队列)   │     │
│  └──────┬───────┘      └──────┬───────┘      └──────┬───────┘     │
│         │                     │                     │              │
│         ▼                     ▼                     ▼              │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐     │
│  │  FileSink   │      │  FileSink    │      │ ConsoleSink  │     │
│  │  app.log   │      │  perf.log    │      │   (stdout)   │     │
│  └──────────────┘      └──────────────┘      └──────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 目录结构

```
mlog/
├── include/mlog/
│   ├── mlog.h              # 主包含头文件
│   ├── logger.h            # Logger 主类
│   ├── sink.h              # Sink 接口
│   ├── file_sink.h         # 文件输出
│   ├── console_sink.h      # 控制台输出
│   ├── log_level.h         # 日志级别定义
│   ├── config.h            # 配置结构
│   ├── formatter.h         # 格式化器
│   └── async_queue.h       # 异步队列
├── src/
│   ├── logger.cpp
│   ├── file_sink.cpp
│   ├── console_sink.cpp
│   ├── formatter.cpp
│   └── async_queue.cpp
├── test/
│   └── test_logger.cpp
├── example/
│   └── basic_example.cpp
└── CMakeLists.txt
```

---

## 3. 核心组件设计

### 3.1 日志级别 (log_level.h)

```cpp
namespace mlog {

enum class LogLevel : uint8_t {
    TRACE = 0,
    DEBUG = 1,
    INFO = 2,
    WARN = 3,
    ERROR = 4,
    FATAL = 5
};

constexpr uint8_t LOG_LEVEL_COUNT = 6;

constexpr const char* log_level_to_string(LogLevel level) {
    switch (level) {
        case LogLevel::TRACE: return "TRACE";
        case LogLevel::DEBUG: return "DEBUG";
        case LogLevel::INFO:  return "INFO";
        case LogLevel::WARN:  return "WARN";
        case LogLevel::ERROR: return "ERROR";
        case LogLevel::FATAL: return "FATAL";
        default: return "UNKNOWN";
    }
}

constexpr LogLevel string_to_log_level(const std::string& str) {
    if (str == "TRACE") return LogLevel::TRACE;
    if (str == "DEBUG") return LogLevel::DEBUG;
    if (str == "INFO")  return LogLevel::INFO;
    if (str == "WARN")  return LogLevel::WARN;
    if (str == "ERROR") return LogLevel::ERROR;
    if (str == "FATAL") return LogLevel::FATAL;
    return LogLevel::INFO;
}

} // namespace mlog
```

### 3.2 Sink 接口 (sink.h)

```cpp
namespace mlog {

class Sink {
public:
    virtual ~Sink() = default;

    virtual void write(LogLevel level, const char* msg, size_t len) = 0;
    virtual void flush() = 0;
    virtual void set_level(LogLevel level) { level_ = level; }
    virtual LogLevel get_level() const { return level_; }

protected:
    Sink(LogLevel level = LogLevel::INFO) : level_(level) {}

    LogLevel level_;
};

using SinkPtr = std::unique_ptr<Sink>;

} // namespace mlog
```

### 3.3 文件输出 Sink (file_sink.h)

```cpp
namespace mlog {

class FileSink : public Sink {
public:
    struct Config {
        std::string file_path;
        size_t max_file_size = 10 * 1024 * 1024;  // 10MB
        size_t max_file_count = 5;
        bool flush_immediate = false;
    };

    explicit FileSink(const Config& config);
    ~FileSink() override;

    void write(LogLevel level, const char* msg, size_t len) override;
    void flush() override;

private:
    void rotate_if_needed();
    void perform_rotation();

    Config config_;
    std::FILE* file_;
    size_t current_size_;
    std::mutex write_mutex_;
};

} // namespace mlog
```

**滚动策略:**
- 当 `current_size_ >= max_file_size_` 时关闭当前文件
- 文件名滚动: `log.0 → log.1`, `log.1 → log.2`, 以此类推
- 超过 `max_file_count` 的文件被删除
- 创建新的 `log.0`

### 3.4 异步队列 (async_queue.h)

MPSC（多生产者单消费者）环缓冲区，核心性能组件。

```cpp
namespace mlog {

struct LogMessage {
    LogLevel level;
    std::chrono::steady_clock::time_point timestamp;
    uint64_t sequence;        // 全局序列号，保证顺序
    std::string thread_name;   // 线程名称
    std::string payload;       // 预格式化消息

    LogMessage(LogLevel lvl, uint64_t seq, std::string thread, std::string msg)
        : level(lvl), sequence(seq), thread_name(std::move(thread)), payload(std::move(msg)) {}
};

class AsyncQueue {
public:
    explicit AsyncQueue(size_t capacity);
    ~AsyncQueue();

    // 从任意线程推送（生产者）
    bool push(LogMessage&& msg);

    // 仅从消费者线程弹出（写入线程）
    bool pop(LogMessage& msg);

    // 批量弹出，提高效率
    size_t pop_batch(std::vector<LogMessage>& batch, size_t max_count);

    void stop();
    bool is_running() const { return running_.load(std::memory_order_acquire); }

    size_t size() const;
    bool is_empty() const;

private:
    struct Node {
        std::atomic<Node*> next;
        LogMessage message;
    };

    alignas(64) Node* head_;          // 生产者写入（原子操作）
    alignas(64) std::atomic<Node*> tail_;  // 消费者读取
    alignas(64) std::atomic<size_t> size_;

    std::vector<Node*> pool_;         // 预分配内存池
    std::atomic<bool> running_;
};

} // namespace mlog
```

**设计要点:**
- **Cache Line 对齐**: 64 字节对齐避免伪共享
- **预分配内存池**: 避免动态分配开销
- **原子操作**: 使用 `memory_order_acquire/release` 保证内存顺序
- **序列号**: 每个消息有单调递增序号，保证全局顺序

### 3.5 格式化器 (formatter.h)

```cpp
namespace mlog {

class Formatter {
public:
    struct FormatPattern {
        bool show_timestamp = true;
        bool show_level = true;
        bool show_thread = true;
        bool show_file_line = false;
        std::string timestamp_format = "%Y-%m-%d %H:%M:%S%.3f";
    };

    explicit Formatter(const FormatPattern& pattern = {}) : pattern_(pattern) {}

    std::string format(LogLevel level, const char* file, int line,
                       const char* function, const char* thread_name,
                       const std::string& message);

private:
    FormatPattern pattern_;
};

} // namespace mlog
```

**输出格式示例:**
```
2026-03-24 15:30:45.123 [INFO] [main] [example.cpp:42] Application started successfully
2026-03-24 15:30:45.124 [DEBUG] [worker-1] [example.cpp:55] Processing request id=1234
```

### 3.6 配置结构 (config.h)

```cpp
namespace mlog {

struct LoggerConfig {
    LogLevel level = LogLevel::INFO;
    bool async_mode = true;
    size_t queue_capacity = 1024;
    size_t batch_size = 32;
    std::chrono::milliseconds flush_interval{100};

    std::vector<SinkConfig> sinks;
    std::string file_path = "/var/log/moss/app.log";
    size_t max_file_size = 10 * 1024 * 1024;
    size_t max_file_count = 5;
    bool enable_console = true;
};

struct SinkConfig {
    std::string type;  // "file" or "console"
    LogLevel level = LogLevel::INFO;
    std::string file_path;
    size_t max_file_size = 10 * 1024 * 1024;
    size_t max_file_count = 5;
};

} // namespace mlog
```

### 3.7 Logger 主类与工厂 (logger.h)

采用**工厂模式**，支持同一进程内创建多个独立的日志实例。

```cpp
namespace mlog {

class Logger;
class LoggerFactory {
public:
    static LoggerFactory& instance();

    std::shared_ptr<Logger> create(const LoggerConfig& config);
    std::shared_ptr<Logger> create(const std::string& config_path);

    void destroy(const std::shared_ptr<Logger>& logger);
    void destroy_all();

private:
    LoggerFactory() = default;
    ~LoggerFactory();

    LoggerFactory(const LoggerFactory&) = delete;
    LoggerFactory& operator=(const LoggerFactory&) = delete;

    std::vector<std::weak_ptr<Logger>> loggers_;
    std::mutex mutex_;
};

class Logger : public std::enable_shared_from_this<Logger> {
public:
    ~Logger();

    bool init(const LoggerConfig& config);
    bool load_config(const std::string& config_path);

    // 日志方法（支持 fmt 格式化）
    void trace(const char* fmt, const fmt::format_args& args);
    void debug(const char* fmt, const fmt::format_args& args);
    void info(const char* fmt, const fmt::format_args& args);
    void warn(const char* fmt, const fmt::format_args& args);
    void error(const char* fmt, const fmt::format_args& args);
    void fatal(const char* fmt, const fmt::format_args& args);

    void log(LogLevel level, const char* file, int line,
             const char* function, const char* fmt, ...);

    void flush();
    void stop();

    void set_level(LogLevel level);
    LogLevel get_level() const { return level_; }

    void set_thread_name(const std::string& name);
    std::string get_current_thread_name() const;

    const std::string& name() const { return name_; }

private:
    friend class LoggerFactory;

    explicit Logger(std::string name);
    void worker_thread_loop();
    void process_messages();

    std::string name_;
    LoggerConfig config_;
    LogLevel level_ = LogLevel::INFO;

    std::vector<SinkPtr> sinks_;
    std::unique_ptr<AsyncQueue> queue_;

    std::thread worker_thread_;
    std::atomic<bool> running_{false};
    std::atomic<uint64_t> sequence_{0};

    static thread_local std::string thread_name_;
    static constexpr const char* DEFAULT_THREAD_NAME = "main";
};

} // namespace mlog
```

**工厂模式特点**:
- `LoggerFactory::instance()` 获取工厂单例
- `create()` 创建独立日志实例，每个实例有独立配置、队列和输出
- `destroy()` 优雅销毁指定实例
- `destroy_all()` 销毁所有实例
- 每个 Logger 可有独立名称，便于区分

---

## 4. JSON 配置文件格式

```json
{
    "log_level": "DEBUG",
    "async_mode": true,
    "queue_capacity": 2048,
    "batch_size": 32,
    "flush_interval_ms": 100,
    "sinks": [
        {
            "type": "console",
            "level": "INFO"
        },
        {
            "type": "file",
            "level": "DEBUG",
            "file_path": "/var/log/moss/app.log",
            "max_file_size": 10485760,
            "max_file_count": 5
        }
    ]
}
```

---

## 5. 性能设计

### 5.1 异步日志路径

1. **生产者（用户线程）**:
   - 本地格式化消息
   - 单原子操作推入 MPSC 队列
   - 立即返回（非阻塞）

2. **消费者（工作线程）**:
   - 批量从队列取出消息
   - 写入所有 Sink
   - 定期刷新

### 5.2 消息顺序保证

- 每个消息分配全局单调递增序列号
- 无论哪个线程产生，消息按序列顺序写入
- 避免不同线程日志行混合

### 5.3 队列背压策略

队列满时采用环形缓冲区策略：丢弃最旧消息。

```cpp
bool AsyncQueue::push(LogMessage&& msg) {
    if (!running_.load(std::memory_order_acquire)) {
        return false;
    }

    Node* node = acquire_node();
    if (!node) {
        // 队列满，丢弃旧消息
        return false;
    }
    // ... 原子链接插入
    return true;
}
```

### 5.4 批量写入

- 累积消息到缓冲区
- 批量写入（可配置 `batch_size`）
- 减少系统调用开销

---

## 6. API 设计

### 6.1 便捷宏

```cpp
// 使用指定 logger 实例
#define MLOG_TRACE(logger, ...) logger->trace(__FILE__, __LINE__, __PRETTY_FUNCTION__, __VA_ARGS__)
#define MLOG_DEBUG(logger, ...) logger->debug(__FILE__, __LINE__, __PRETTY_FUNCTION__, __VA_ARGS__)
#define MLOG_INFO(logger, ...)  logger->info(__FILE__, __LINE__, __PRETTY_FUNCTION__, __VA_ARGS__)
#define MLOG_WARN(logger, ...)  logger->warn(__FILE__, __LINE__, __PRETTY_FUNCTION__, __VA_ARGS__)
#define MLOG_ERROR(logger, ...) logger->error(__FILE__, __LINE__, __PRETTY_FUNCTION__, __VA_ARGS__)
#define MLOG_FATAL(logger, ...) logger->fatal(__FILE__, __LINE__, __PRETTY_FUNCTION__, __VA_ARGS__)
```

### 6.2 使用示例

```cpp
#include "mlog/mlog.h"

int main() {
    // 通过工厂创建多个独立日志实例
    auto& factory = mlog::LoggerFactory::instance();

    // 创建主日志实例
    auto main_logger = factory.create("main_logger");
    main_logger->load_config("/etc/myapp/main_logger.json");

    // 创建性能日志实例（独立文件输出）
    mlog::LoggerConfig perf_config;
    perf_config.name = "perf_logger";
    perf_config.file_path = "/var/log/myapp/perf.log";
    perf_config.level = mlog::LogLevel::INFO;
    auto perf_logger = factory.create(perf_config);

    // 设置线程名称
    main_logger->set_thread_name("main");

    // 日志输出
    MLOG_INFO(main_logger, "Application starting...");
    MLOG_DEBUG(main_logger, "Config loaded: {} items", config_items);
    MLOG_ERROR(main_logger, "Connection failed: {} (error code: {})", strerror(err), err_code);

    // 性能日志
    MLOG_INFO(perf_logger, "Request processed in {} ms", elapsed_ms);

    // 优雅关闭
    main_logger->flush();
    main_logger->stop();
    perf_logger->flush();
    perf_logger->stop();

    // 或一次性销毁所有实例
    factory.destroy_all();

    return 0;
}
```

### 6.3 多实例场景

```
┌─────────────────────────────────────────────────────────┐
│                      进程                                │
│                                                         │
│  ┌─────────────────┐    ┌─────────────────┐           │
│  │   main_logger   │    │   perf_logger   │           │
│  │  (级别: INFO)   │    │  (级别: DEBUG)  │           │
│  │  文件: app.log  │    │  文件: perf.log │           │
│  └────────┬────────┘    └────────┬────────┘           │
│           │                      │                     │
│           ▼                      ▼                     │
│     ┌──────────┐           ┌──────────┐               │
│     │ FileSink │           │ FileSink │               │
│     │ app.log  │           │ perf.log │               │
│     └──────────┘           └──────────┘               │
└─────────────────────────────────────────────────────────┘
```

---

## 7. 构建集成

### 7.1 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.14)
project(mlog VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Threads REQUIRED)

# fmt 库（头文件）
find_path(FMT_INCLUDE_DIR fmt/format.h)

set(MLOG_SOURCES
    src/logger.cpp
    src/file_sink.cpp
    src/console_sink.cpp
    src/formatter.cpp
    src/async_queue.cpp
)

add_library(mlog STATIC ${MLOG_SOURCES})

target_include_directories(mlog PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
)

target_link_libraries(mlog PUBLIC Threads::Threads)

if(FMT_INCLUDE_DIR)
    target_include_directories(mlog PRIVATE ${FMT_INCLUDE_DIR})
endif()

target_compile_options(mlog PRIVATE
    -Wall -Wextra -Wpedantic
    $<$<CONFIG:Release>:-O3>
)
```

---

## 8. 依赖关系

| 依赖 | 用途 | 说明 |
|------|------|------|
| fmt | 格式化输出 | 头文件库，支持 `{}` 格式 |
| Threads | 线程支持 | 标准 `std::thread` |

---

## 9. 关键实现文件

| 文件 | 用途 |
|------|------|
| `include/mlog/logger.h` | Logger 主类 |
| `include/mlog/async_queue.h` | MPSC 队列核心 |
| `include/mlog/sink.h` | Sink 接口 |
| `include/mlog/file_sink.h` | 文件滚动实现 |
| `include/mlog/config.h` | 配置结构 |
| `src/async_queue.cpp` | 无锁队列实现 |
| `src/file_sink.cpp` | 文件写入和滚动 |

---

## 10. 扩展功能（可选）

### 10.1 配置热加载

```cpp
class Logger {
public:
    bool reload_config(const std::string& config_path);
    void enable_config_watch();   // 使用 inotify/FSEvents
    void disable_config_watch();
};
```

### 10.2 共享内存 Sink

支持跨进程日志聚合，使用 mshm 模块。

### 10.3 网络 Sink

支持日志通过网络输出到远程服务器。
