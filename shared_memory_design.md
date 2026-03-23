# 跨平台共享内存模块设计文档 (一对多架构)

## 1. 概述

本模块提供统一的共享内存接口，支持 QNX 和 Linux 双平台运行。采用**一对多**架构设计，一个生产者创建共享内存，多个消费者可连接访问。

### 1.1 设计目标
- **一对多模式**：一个进程创建，多个进程共享
- **跨平台兼容**：统一接口，底层平台差异化实现
- **连接管理**：引用计数、自动清理、断线检测
- **高效通知**：eventfd 广播机制
- **进程安全**：Robust Mutex 崩溃自动恢复

### 1.2 术语表

| 术语 | 说明 |
|------|------|
| Producer (生产者) | 创建并拥有共享内存的进程 |
| Consumer (消费者) | 连接并使用共享内存的进程 |
| Handle (句柄) | 封装共享内存连接信息的 opaque 类型 |
| Control Block (控制块) | 共享内存开头的元数据区域 |
| Robust Lock (健壮锁) | 进程崩溃后能自动解锁的互斥锁 |

---

## 2. 架构设计

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          生产者 (Producer)                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │ shm_create() │  │ shm_notify() │  │ shm_lock/unlock()       │ │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │ 消费者1  │    │ 消费者2  │    │ 消费者N  │
        │shm_join()│    │shm_join()│    │shm_join()│
        │shm_wait()│    │shm_wait()│    │shm_wait()│
        └──────────┘    └──────────┘    └──────────┘
              │               │               │
              └───────────────┼───────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    共享内存 (Shared Memory)                            │
│  ┌─────────────────────────────┬───────────────────────────────────┐ │
│  │     控制块 (256 bytes)      │        数据区 (用户定义)           │ │
│  │  ┌─────────┐ ┌──────────┐  │                                   │ │
│  │  │ magic   │ │ ref_count│  │                                   │ │
│  │  │ version │ │lock.mutex│  │                                   │ │
│  │  │data_size│ │notify_fd │  │                                   │ │
│  │  └─────────┘ └──────────┘  │                                   │ │
│  └─────────────────────────────┴───────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 内存布局

```
共享内存文件结构 (mmap 映射后):
┌──────────────────────────────────────────────────────────────────┐
│                        控制块 (Control Block)                       │
│                        sizeof(shm_header_t) = 256 bytes            │
├──────────────────────────────────────────────────────────────────┤
│ Offset  │  Field           │  Description                        │
│--------─┼----------------──┼-------------------------------------│
│ 0x00    │  magic           │  0x53484D5F ('SHM_')               │
│ 0x04    │  version         │  0x00000001                        │
│ 0x08    │  data_size       │  用户数据大小                       │
│ 0x0C    │  ref_count       │  引用计数 (原子操作)                │
│ 0x10    │  connected_count │  当前连接数                        │
│ 0x14    │  notify_fd       │  eventfd 文件描述符                │
│ 0x18    │  interest_mask   │  关注的事件掩码                    │
│ 0x1C    │  pending_notify  │  待处理通知计数                    │
│ 0x20    │  lock.mutex      │  pthread_mutex_t (40-64 bytes)    │
│ 0x50    │  lock.owner      │  当前持有者PID                     │
│ 0x54    │  lock.state      │  锁状态                            │
│ 0x58+   │  reserved        │  对齐到 256 bytes                  │
├──────────────────────────────────────────────────────────────────┤
│                        数据区 (Data Section)                       │
│                        用户定义的数据结构                           │
│                        大小由 data_size 指定                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心数据结构

### 3.1 常量定义

```c
// shm_types.h
#define SHM_MAGIC           0x53484D5F    // 'SHM_'
#define SHM_VERSION         0x00000001    // 当前版本
#define SHM_NAME_MAX        64            // 最大名称长度
#define SHM_NAME_PREFIX     "/"           // POSIX共享内存路径前缀
#define SHM_HEADER_SIZE     256           // 控制块总大小 (必须对齐)
#define SHM_RESERVED_SIZE   128           // 保留空间大小
```

### 3.2 枚举类型

```c
// 权限
typedef enum {
    SHM_PERM_NONE   = 0,
    SHM_PERM_READ   = 1 << 0,
    SHM_PERM_WRITE  = 1 << 1,
} shm_permission_t;

// 标志
typedef enum {
    SHM_FLAG_NONE     = 0,
    SHM_FLAG_CREATE   = 1 << 0,   // 创建新的共享内存
    SHM_FLAG_EXCL     = 1 << 1,   // 已存在则失败
    SHM_FLAG_READONLY = 1 << 2,   // 只读模式映射
} shm_flags_t;

// 锁状态
typedef enum {
    SHM_LOCK_FREE    = 0,
    SHM_LOCK_BUSY    = 1,
    SHM_LOCK_RECOVER = 2,
} shm_lock_state_t;

// 事件类型
typedef enum {
    SHM_EVENT_NONE        = 0,
    SHM_EVENT_DATA_WRITE  = 1 << 0,
    SHM_EVENT_CONNECTED  = 1 << 1,
    SHM_EVENT_DISCONNECT = 1 << 2,
    SHM_EVENT_DESTROY    = 1 << 3,
} shm_event_type_t;
```

### 3.3 核心结构体

```c
// 进程间锁 (放在控制头中)
typedef struct shm_lock {
    pthread_mutex_t  mutex;           // 互斥锁
    volatile pid_t  owner;           // 当前持有者PID
    volatile int32_t state;          // 锁状态
} shm_lock_t;

// 控制头 (256 bytes, 8字节对齐)
typedef struct shm_header {
    uint32_t          magic;             // 魔数: 0x53484D5F
    uint32_t          version;           // 版本号
    uint32_t          data_size;         // 数据区大小
    volatile uint32_t ref_count;         // 引用计数
    volatile uint32_t connected_count;   // 当前连接数
    int               notify_fd;         // eventfd 文件描述符
    volatile uint32_t interest_mask;    // 关注的事件掩码
    volatile uint32_t pending_notify;    // 待处理通知数
    shm_lock_t        lock;              // 进程间锁
    uint8_t           reserved[SHM_RESERVED_SIZE];
} shm_header_t;
_Static_assert(sizeof(shm_header_t) == SHM_HEADER_SIZE,
               "shm_header_t must be 256 bytes");

// 句柄类型 (不透明指针)
typedef struct shm_handle_impl* shm_handle_t;

// 连接信息
typedef struct shm_connection {
    char     name[SHM_NAME_MAX];
    pid_t    pid;
    uint32_t flags;
    uint64_t connect_time;
} shm_connection_t;
```

---

## 4. 目录结构

```
shm/
├── include/
│   └── shm/
│       ├── shm_api.h          # 公共接口定义
│       ├── shm_types.h        # 类型和数据结构定义
│       ├── shm_errors.h       # 错误码定义
│       └── shm_adapter.h      # 平台适配选择
├── src/
│   ├── shm_common.c           # 公共实现
│   ├── shm_linux.c            # Linux 平台实现
│   ├── shm_qnx.c              # QNX 平台实现
│   └── CMakeLists.txt
├── test/
│   ├── producer_test.c
│   ├── consumer_test.c
│   ├── multi_consumer_test.c
│   ├── crash_recovery_test.c
│   └── CMakeLists.txt
└── CMakeLists.txt
```

---

## 5. API 设计

### 5.1 完整头文件 (shm_api.h)

```c
// shm_api.h
#ifndef SHM_API_H
#define SHM_API_H

#include "shm_types.h"
#include "shm_errors.h"
#include <stddef.h>
#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

// ========== 生命周期管理 ==========

SHM_API shm_error_t shm_create(const char* name,
                               size_t data_size,
                               shm_permission_t perm,
                               shm_flags_t flags,
                               shm_handle_t** out_handle);

SHM_API shm_error_t shm_join(const char* name,
                             shm_permission_t perm,
                             shm_handle_t** out_handle);

SHM_API shm_error_t shm_unmap(shm_handle_t* handle);
SHM_API shm_error_t shm_close(shm_handle_t** handle);
SHM_API shm_error_t shm_destroy(const char* name);

// ========== 数据访问 ==========

SHM_API void* shm_get_data_ptr(shm_handle_t* handle);
SHM_API const void* shm_get_data_ptr_const(shm_handle_t* handle);
SHM_API size_t shm_get_data_size(shm_handle_t* handle);

// ========== 通知机制 ==========

SHM_API shm_error_t shm_notify(shm_handle_t* handle);
SHM_API shm_error_t shm_wait(shm_handle_t* handle, int timeout_ms);
SHM_API int shm_get_notify_fd(shm_handle_t* handle);
SHM_API shm_error_t shm_consume_notify(shm_handle_t* handle);

// ========== 连接管理 ==========

SHM_API shm_error_t shm_get_connection_count(shm_handle_t* handle, uint32_t* count);
SHM_API bool shm_exists(const char* name);

// ========== 进程间锁 ==========

SHM_API shm_error_t shm_lock(shm_handle_t* handle, int timeout_ms);
SHM_API shm_error_t shm_lock_try(shm_handle_t* handle);
SHM_API shm_error_t shm_unlock(shm_handle_t* handle);
SHM_API bool shm_is_locked(shm_handle_t* handle);

// ========== 工具函数 ==========

SHM_API const char* shm_strerror(shm_error_t err);
SHM_API const char* shm_version(void);

#ifdef __cplusplus
}
#endif

#endif // SHM_API_H
```

### 5.2 API 速查表

| 分类 | 函数 | 说明 |
|------|------|------|
| **生命周期** | `shm_create()` | 生产者创建共享内存 |
| | `shm_join()` | 消费者加入共享内存 |
| | `shm_close()` | 关闭句柄 |
| | `shm_destroy()` | 销毁共享内存 (仅生产者) |
| **数据访问** | `shm_get_data_ptr()` | 获取写指针 |
| | `shm_get_data_ptr_const()` | 获取读指针 |
| **通知** | `shm_notify()` | 生产者通知消费者 |
| | `shm_wait()` | 消费者等待通知 |
| | `shm_get_notify_fd()` | 获取 pollfd |
| **锁** | `shm_lock()` | 加锁 (支持超时) |
| | `shm_lock_try()` | 尝试加锁 |
| | `shm_unlock()` | 解锁 |

---

## 6. 错误码 (shm_errors.h)

```c
// shm_errors.h
typedef enum {
    SHM_OK = 0,

    // 通用错误 (1-99)
    SHM_ERR_INVALID_PARAM    = 1,
    SHM_ERR_NOT_FOUND        = 2,
    SHM_ERR_ALREADY_EXISTS   = 3,
    SHM_ERR_PERMISSION       = 4,
    SHM_ERR_NO_MEMORY        = 5,
    SHM_ERR_SYSTEM           = 6,
    SHM_ERR_TIMEOUT          = 7,
    SHM_ERR_NOT_INITIALIZED  = 8,
    SHM_ERR_NOT_CONNECTED    = 9,
    SHM_ERR_STALE_HANDLE     = 10,
    SHM_ERR_VERSION_MISMATCH = 11,
    SHM_ERR_NAME_INVALID     = 12,
    SHM_ERR_SIZE_INVALID     = 13,

    // 锁相关错误 (100-199)
    SHM_ERR_LOCK_TIMEOUT     = 100,
    SHM_ERR_LOCK_DEAD        = 101,
    SHM_ERR_LOCK_RECOVERED   = 102,
    SHM_ERR_LOCK_NOT_OWNED   = 103,

    // 状态错误 (200-299)
    SHM_ERR_ALREADY_OPEN     = 200,
    SHM_ERR_NOT_OPEN         = 201,
    SHM_ERR_MAP_FAILED       = 202,
    SHM_ERR_UNMAP_FAILED     = 203,
    SHM_ERR_CLOSE_FAILED     = 204,
} shm_error_t;
```

---

## 7. 进程间锁机制

### 7.1 Robust Mutex 初始化

```c
int shm_lock_init(shm_header_t* header) {
    pthread_mutexattr_t attr;

    pthread_mutexattr_init(&attr);
    pthread_mutexattr_setpshared(&attr, PTHREAD_PROCESS_SHARED);
    pthread_mutexattr_setrobust(&attr, PTHREAD_MUTEX_ROBUST);

    pthread_mutex_init(&header->lock.mutex, &attr);
    pthread_mutexattr_destroy(&attr);

    return 0;
}
```

### 7.2 加锁实现

```c
shm_error_t shm_lock(shm_handle_t* handle, int timeout_ms) {
    shm_header_t* header = (shm_header_t*)handle->addr;
    int ret;

    if (timeout_ms < 0) {
        ret = pthread_mutex_lock(&header->lock.mutex);
    } else {
        struct timespec ts;
        clock_gettime(CLOCK_REALTIME, &ts);
        ts.tv_sec += timeout_ms / 1000;
        ts.tv_nsec += (timeout_ms % 1000) * 1000000;
        ret = pthread_mutex_timedlock(&header->lock.mutex, &ts);
    }

    if (ret == EOWNERDEAD) {
        pthread_mutex_consistent(&header->lock.mutex);
        return SHM_ERR_LOCK_RECOVERED;
    }

    return (ret == 0) ? SHM_OK : SHM_ERR_SYSTEM;
}
```

### 7.3 崩溃恢复流程

```
正常流程:
进程A 加锁 ──> 写入数据 ──> 解锁 ──> 进程B 获取锁

崩溃流程:
进程A 加锁 ──> 进程A 崩溃
                      │
                      ▼
        内核检测到 owner 已终止，自动解锁
                      │
                      ▼
进程B pthread_mutex_lock() 返回 EOWNERDEAD
                      │
                      ▼
进程B 调用 pthread_mutex_consistent() 恢复锁
                      │
                      ▼
进程B 清理崩溃进程残留状态
```

---

## 8. 通知机制

### 8.1 广播原理

```
生产者: eventfd_write(notify_fd, 1)  // 原子增加计数器
                │
                ▼
        ┌───────┴───────┐
        ▼               ▼
    消费者1          消费者2
    poll()返回       poll()返回
        │               │
        ▼               ▼
    eventfd_read()   eventfd_read()
    (消费1个计数)    (消费1个计数)
```

### 8.2 poll/select 集成

```c
struct pollfd pfd = {
    .fd = shm_get_notify_fd(handle),
    .events = POLLIN,
};

int ret = poll(&pfd, 1, 1000);
if (ret > 0 && (pfd.revents & POLLIN)) {
    shm_consume_notify(handle);
    // 处理数据...
}
```

---

## 9. 线程安全性

### 9.1 线程安全级别

| 对象 | 线程安全 | 说明 |
|------|---------|------|
| `shm_handle_t` | 线程安全 | 同一句柄可在多线程中使用 |
| 控制块 | 进程安全 | 跨进程需要锁 |
| 句柄实现 | 线程安全 | 内部有锁保护 |

### 9.2 嵌套锁禁止

```c
// 错误示例: 会导致死锁
shm_lock(handle, 1000);
shm_lock(handle, 1000);  // DEADLOCK!
shm_unlock(handle);
```

---

## 10. 平台差异

### 10.1 POSIX 兼容层

| 特性 | Linux | QNX |
|------|-------|-----|
| `shm_open()` | ✅ | ✅ |
| `shm_unlink()` | ✅ | ✅ |
| `mmap()` | ✅ | ✅ |
| `eventfd()` | ✅ (2.6.22+) | ✅ (6.4+) |
| `pthread_mutex` (robust) | ✅ (2.6.32+) | ✅ |

### 10.2 平台检测 (shm_adapter.h)

```c
#if defined(__linux__)
    #define SHM_PLATFORM_LINUX  1
    #define SHM_PLATFORM_QNX    0
#elif defined(__QNX__)
    #define SHM_PLATFORM_LINUX  0
    #define SHM_PLATFORM_QNX    1
#endif
```

---

## 11. 性能考虑

### 11.1 性能指标

| 操作 | 耗时 | 说明 |
|------|------|------|
| `shm_create()` | ~100-500μs | 首次创建 |
| `shm_join()` | ~50-200μs | 消费者连接 |
| `shm_notify()` | ~1-5μs | 事件通知 |
| `shm_lock()` | ~1-3μs | 快速路径 |
| 数据访问 | ~0ns | 直接内存访问 |

### 11.2 优化建议

1. **减少锁粒度**: 只在必要时加锁
2. **批量通知**: 多次写入后一次通知
3. **缓存友好**: 数据结构按 cache line 对齐

---

## 12. 构建配置

### 12.1 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)
project(shm C)

# 平台检测
if(CMAKE_SYSTEM_NAME MATCHES "Linux")
    set(PLATFORM "linux")
elseif(CMAKE_SYSTEM_NAME MATCHES "QNX")
    set(PLATFORM "qnx")
endif()

add_compile_definitions(SHM_PLATFORM_${PLATFORM})

include_directories(${CMAKE_SOURCE_DIR}/include)

set(SHM_SOURCES
    src/shm_common.c
    src/shm_${PLATFORM}.c
)

add_library(shm STATIC ${SHM_SOURCES})
target_link_libraries(shm rt pthread)

enable_testing()
add_subdirectory(test)
```

---

## 13. 使用示例

### 13.1 生产者

```c
#include "shm/shm_api.h"
#include <stdio.h>

typedef struct {
    float temperature;
    float humidity;
    uint64_t timestamp;
} SensorData;

int main(void) {
    shm_handle_t* handle = NULL;

    // 1. 创建共享内存
    shm_create("/sensor_shm", sizeof(SensorData),
               SHM_PERM_READ | SHM_PERM_WRITE,
               SHM_FLAG_CREATE | SHM_FLAG_EXCL,
               &handle);

    SensorData* data = (SensorData*)shm_get_data_ptr(handle);

    // 2. 写入数据 (带锁)
    for (int i = 0; i < 100; i++) {
        shm_lock(handle, 1000);
        data->temperature = 20.0f + (i % 10);
        data->humidity = 50.0f;
        data->timestamp = time(NULL);
        shm_unlock(handle);

        shm_notify(handle);  // 通知消费者
        usleep(100000);
    }

    shm_destroy("/sensor_shm");
    shm_close(&handle);
    return 0;
}
```

### 13.2 消费者

```c
#include "shm/shm_api.h"
#include <stdio.h>
#include <poll.h>

typedef struct {
    float temperature;
    float humidity;
    uint64_t timestamp;
} SensorData;

int main(void) {
    shm_handle_t* handle = NULL;

    // 1. 加入共享内存
    shm_join("/sensor_shm", SHM_PERM_READ, &handle);

    // 2. 使用 poll 异步等待
    struct pollfd pfd = {
        .fd = shm_get_notify_fd(handle),
        .events = POLLIN,
    };

    while (1) {
        int ret = poll(&pfd, 1, 1000);
        if (ret > 0 && (pfd.revents & POLLIN)) {
            shm_consume_notify(handle);

            shm_lock(handle, 500);
            const SensorData* data = shm_get_data_ptr_const(handle);
            printf("Temp: %.1f\n", data->temperature);
            shm_unlock(handle);
        }
    }

    shm_close(&handle);
    return 0;
}
```

### 13.3 多消费者

```c
int main(int argc, char* argv[]) {
    int num_consumers = (argc > 1) ? atoi(argv[1]) : 3;

    // 启动多个消费者进程
    for (int i = 0; i < num_consumers; i++) {
        if (fork() == 0) {
            shm_handle_t* h;
            shm_join("/test_shm", SHM_PERM_READ, &h);
            while (shm_wait(h, -1) == SHM_OK) {
                const int* data = shm_get_data_ptr_const(h);
                printf("[Consumer %d] %d\n", i, *data);
            }
            shm_close(&h);
            exit(0);
        }
    }

    // 父进程作为生产者
    sleep(1);
    shm_handle_t* h;
    shm_create("/test_shm", sizeof(int), SHM_PERM_READ|SHM_PERM_WRITE,
               SHM_FLAG_CREATE, &h);

    for (int i = 0; i < 10; i++) {
        int* data = shm_get_data_ptr(h);
        *data = i;
        shm_notify(h);
        sleep(1);
    }

    shm_destroy("/test_shm");
    shm_close(&h);
    return 0;
}
```

### 13.4 崩溃恢复

```c
shm_error_t err = shm_lock(handle, 2000);

switch (err) {
    case SHM_OK:
        printf("Lock acquired normally\n");
        break;
    case SHM_ERR_LOCK_RECOVERED:
        printf("Recovered from crashed process!\n");
        // 清理残留状态
        break;
    case SHM_ERR_LOCK_TIMEOUT:
        printf("Lock timeout (possible deadlock)\n");
        break;
}
```

---

## 14. 关键设计点总结

| 特性 | 实现方式 |
|------|---------|
| 一对多连接 | 引用计数 `ref_count` |
| 广播通知 | `eventfd` 计数器 |
| 阻塞等待 | `poll()` / `select()` 监听 `notify_fd` |
| 进程间锁 | Robust Mutex (`pthread_mutexattr_setrobust`) |
| 崩溃恢复 | `EOWNERDEAD` + `pthread_mutex_consistent` |
| 死锁预防 | 超时机制 |
| 跨平台 | 编译时平台检测 + POSIX API |
