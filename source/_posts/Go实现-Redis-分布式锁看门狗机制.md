---
title: Go实现 Redis 分布式锁看门狗机制
date: 2024-06-24 09:39:24
tags:
- Golang
- Redis
- 锁
categories:
- Golang
- Redis
index_img: https://github.com/whitestorm0316/picx-images-hosting/raw/master/image.491ifsl4mn.webp
---
# Go 实现 Redis 分布式锁看门狗机制

## 一、核心设计要点

### 1. 看门狗的三要素

```go
type Lock struct {
    key         string
    value       string    // 唯一标识
    expiration time.Duration
    ctx         context.Context
    cancel      context.CancelFunc
    redisClient *redis.Client
    mutex       sync.Mutex
    isHeld      bool
}
```

### 2. 自动续期流程

```text
┌─────────────┐       ┌──────────────┐
│ 获取锁成功     │       │  业务处理中    │
└──────┬──────┘       └──────┬───────┘
       │                     │
       ▼                     ▼
┌──────────────┐       ┌──────────────┐
│ 启动看门狗线程  │◀─────▶│ 定期续期锁    │
└──────────────┘       └──────────────┘
```

---

## 二、完整实现代码

```go
package main

import (
    "context"
    "fmt"
    "github.com/go-redis/redis/v8"
    "math/rand"
    "sync"
    "time"
)

const (
    lockCommand = `if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("PEXPIRE", KEYS[1], ARGV[2])
else
    return 0
end`
    watchdogInterval = 5 * time.Second  // 续期间隔
    defaultTTL      = 30 * time.Second // 默认锁超时
)

type RedisLock struct {
    client      *redis.Client
    key         string
    value       string
    ttl         time.Duration
    watchdogCtx context.Context
    cancelWatchdog context.CancelFunc
    mutex       sync.Mutex
}

func NewRedisLock(client *redis.Client, key string) *RedisLock {
    return &RedisLock{
        client: client,
        key:    key,
        value:  generateRandomValue(),
        ttl:    defaultTTL,
    }
}

func (l *RedisLock) Lock(ctx context.Context) (bool, error) {
    l.mutex.Lock()
    defer l.mutex.Unlock()

    // 尝试获取锁
    resp, err := l.client.SetNX(ctx, l.key, l.value, l.ttl).Result()
    if err != nil || !resp {
        return false, err
    }

    // 启动看门狗
    l.watchdogCtx, l.cancelWatchdog = context.WithCancel(context.Background())
    go l.startWatchdog()

    return true, nil
}

func (l *RedisLock) startWatchdog() {
    ticker := time.NewTicker(watchdogInterval)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            // 续期操作
            l.mutex.Lock()
            if !l.renewLock() {
                l.mutex.Unlock()
                return
            }
            l.mutex.Unlock()
        case <-l.watchdogCtx.Done():
            return
        }
    }
}

func (l *RedisLock) renewLock() bool {
    // 使用Lua脚本保证原子性
    result, err := l.client.Eval(context.Background(), lockCommand, 
        []string{l.key}, l.value, l.ttl.Milliseconds()).Int()
    return err == nil && result == 1
}

func (l *RedisLock) Unlock() error {
    l.mutex.Lock()
    defer l.mutex.Unlock()

    // 停止看门狗
    if l.cancelWatchdog != nil {
        l.cancelWatchdog()
    }

    // 使用Lua脚本释放锁
    script := redis.NewScript(`
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end`)
  
    _, err := script.Run(context.Background(), l.client, 
        []string{l.key}, l.value).Result()
    return err
}

func generateRandomValue() string {
    rand.Seed(time.Now().UnixNano())
    return fmt.Sprintf("%d-%d", time.Now().UnixNano(), rand.Intn(1000))
}
```

---

## 三、关键实现细节

### 1. 安全续期机制

```go
// 续期Lua脚本
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("PEXPIRE", KEYS[1], ARGV[2])
else
    return 0
end
```

**优势：**

- 保证检查值和续期的原子性
- 防止锁过期后的错误续期

### 2. 看门狗生命周期管理

```go
// 启动看门狗
ctx, cancel := context.WithCancel(context.Background())
go func() {
    for {
        select {
        case <-time.After(interval):
            renew()
        case <-ctx.Done():
            return
        }
    }
}()

// 停止看门狗
cancel()
```

### 3. 错误处理机制

```go
func (l *RedisLock) renewLock() bool {
    result, err := l.client.Eval(...)
    if err != nil {
        log.Printf("续期失败: %v", err)
        return false
    }
    return result == 1
}
```

---

## 四、使用示例

### 基本用法

```go
func main() {
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })

    lock := NewRedisLock(rdb, "my_resource_lock")
  
    // 获取锁
    acquired, err := lock.Lock(context.Background())
    if err != nil || !acquired {
        panic("获取锁失败")
    }
  
    // 执行业务逻辑
    defer lock.Unlock()
    processBusiness()
}

func processBusiness() {
    // 模拟长时间处理
    time.Sleep(1 * time.Minute)
    fmt.Println("业务处理完成")
}
```

---

## 五、性能优化建议

### 1. 动态调整续期间隔

```go
// 根据TTL自动计算间隔
func getRenewInterval(ttl time.Duration) time.Duration {
    return ttl / 3
}
```

### 2. 网络异常重试机制

```go
func (l *RedisLock) renewLockWithRetry() bool {
    for i := 0; i < 3; i++ {
        if success := l.renewLock(); success {
            return true
        }
        time.Sleep(100 * time.Millisecond)
    }
    return false
}
```

### 3. 监控集成

```go
type LockMetrics struct {
    RenewSuccess prometheus.Counter
    RenewFailure prometheus.Counter
    LockDuration prometheus.Histogram
}

var metrics LockMetrics

func init() {
    metrics.RenewSuccess = promauto.NewCounter(...)
    metrics.RenewFailure = promauto.NewCounter(...)
}
```

---

## 六、与其他方案的对比

| 特性         | 原生Redis锁 | Redisson看门狗 | 本实现方案 |
| ------------ | ----------- | -------------- | ---------- |
| 自动续期     | ❌          | ✅             | ✅         |
| 可重入锁     | ❌          | ✅             | ❌         |
| 网络抖动容错 | ❌          | ✅             | ✅         |
| 单点故障容错 | ❌          | ❌             | ❌         |
| 内存消耗     | 低          | 中             | 低         |
| Go支持       | ✅          | ❌             | ✅         |

---

## 七、注意事项

1. **时钟同步问题**：确保所有节点使用NTP时间同步
2. **资源泄漏防护**：必须搭配 `defer Unlock()`使用
3. **最大续期次数**：建议设置最大续期时间上限
4. **网络分区处理**：需要配合熔断机制使用
5. **锁粒度控制**：避免过大的锁范围影响性能

```go
// 错误用法示例：忘记解锁
func riskyOperation() {
    lock.Lock()
    // 没有解锁操作
}

// 正确用法
func safeOperation() {
    acquired, _ := lock.Lock()
    if !acquired {
        return
    }
    defer lock.Unlock()
    // 业务代码
}
```

---

## 八、扩展功能实现

### 1. 锁等待队列

```go
func (l *RedisLock) LockWithTimeout(ctx context.Context, timeout time.Duration) bool {
    deadline := time.Now().Add(timeout)
    for {
        acquired, err := l.Lock(ctx)
        if acquired {
            return true
        }
        if time.Now().After(deadline) {
            return false
        }
        time.Sleep(50 * time.Millisecond)
    }
}
```

### 2. 锁续期时间监控

```go
func (l *RedisLock) monitorTTL() {
    go func() {
        for {
            ttl, err := l.client.TTL(context.Background(), l.key).Result()
            if err == nil {
                metrics.LockDuration.Observe(ttl.Seconds())
            }
            time.Sleep(1 * time.Second)
        }
    }()
}
```

通过以上实现，可以构建出生产级可靠的 Redis 分布式锁看门狗机制。实际使用中需要根据具体业务场景调整续期间隔和错误处理策略。
