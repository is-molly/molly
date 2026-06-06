---
order: 2
date: 2026-06-03
category:
  - 后端
  - Golang
---

# Go操作Redis

<!-- more -->

## 一、Redis 简介

Redis（Remote Dictionary Server）是一个开源的、基于内存的键值对存储系统，通常用作数据库、缓存和消息中间件。

### 1.1 核心特点

- **基于内存**：数据存储在内存中，读写速度极快
- **持久化**：支持 RDB 快照和 AOF 日志两种持久化方式
- **丰富的数据结构**：支持 String、Hash、List、Set、Sorted Set 等
- **原子操作**：所有操作都是原子性的
- **发布/订阅**：支持消息的发布与订阅模式
- **支持过期**：可以为键设置过期时间

### 1.2 常用数据类型

| 类型 | 说明 | 常见场景 |
|------|------|----------|
| String | 字符串，二进制安全 | 缓存、计数器、分布式锁 |
| Hash | 哈希表，存储键值对集合 | 存储对象（如用户信息） |
| List | 双向链表 | 消息队列、最新列表 |
| Set | 无序集合，元素唯一 | 标签、共同好友 |
| Sorted Set | 有序集合，每个元素关联一个 score | 排行榜、延时队列 |

### 1.3 常用命令速览

```bash
# String 操作
SET key value           # 设置键值
GET key                 # 获取值
INCR key                # 自增 1
EXPIRE key seconds      # 设置过期时间（秒）
TTL key                 # 查看剩余过期时间

# Hash 操作
HSET key field value    # 设置哈希字段
HGET key field          # 获取哈希字段
HGETALL key             # 获取所有字段和值

# List 操作
LPUSH key value         # 从左侧插入
RPUSH key value         # 从右侧插入
LPOP key                # 从左侧弹出
LRANGE key start stop   # 获取范围元素

# Set 操作
SADD key member         # 添加成员
SMEMBERS key            # 获取所有成员
SISMEMBER key member    # 判断成员是否存在

# Sorted Set 操作
ZADD key score member   # 添加成员及分数
ZRANGE key start stop   # 按分数升序获取
ZREVRANGE key start stop # 按分数降序获取
```

## 二、Go Redis 客户端选择

Go 语言中使用最多的 Redis 客户端是 `go-redis`。

```bash
go get github.com/redis/go-redis/v9
```

| 客户端库 | 说明 |
|----------|------|
| [go-redis](https://github.com/redis/go-redis) | 功能最全面、社区最活跃的 Go Redis 客户端 |
| [redigo](https://github.com/gomodule/redigo) | 较早的 Go Redis 库，API 较底层 |

推荐使用 `go-redis/v9`，它支持：

- 所有 Redis 数据类型操作
- 连接池管理
- 集群、哨兵模式
- Pipeline（管道）
- Pub/Sub（发布/订阅）
- 事务（MULTI/EXEC）
- Lua 脚本

## 三、连接 Redis

### 3.1 单机连接

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/redis/go-redis/v9"
)

// RedisConfig Redis 连接配置
type RedisConfig struct {
    Addr     string // Redis 地址，如 "localhost:6379"
    Password string // Redis 密码，无密码时为空
    DB       int    // 数据库编号，默认 0
}

var rdb *redis.Client

func initRedis() {
    // 创建 Redis 客户端
    rdb = redis.NewClient(&redis.Options{
        Addr:     "localhost:6379", // Redis 地址
        Password: "",               // 密码，无密码留空
        DB:       0,                // 使用默认数据库

        // 连接池配置
        PoolSize:     10,  // 连接池大小，默认 10 * runtime.GOMAXPROCS
        MinIdleConns: 5,   // 最小空闲连接数
    })

    // 测试连接
    ctx := context.Background()
    if err := rdb.Ping(ctx).Err(); err != nil {
        log.Fatal("Redis 连接失败:", err)
    }
    fmt.Println("Redis 连接成功")
}

func main() {
    initRedis()
    defer rdb.Close()
}
```

### 3.2 哨兵模式连接

```go
// initSentinel 初始化 Redis 哨兵模式客户端
func initSentinel() *redis.Client {
    return redis.NewFailoverClient(&redis.FailoverOptions{
        MasterName:    "mymaster",
        SentinelAddrs: []string{":26379", ":26380", ":26381"},
        Password:      "",
    })
}
```

### 3.3 集群模式连接

```go
// initCluster 初始化 Redis 集群模式客户端
func initCluster() *redis.ClusterClient {
    return redis.NewClusterClient(&redis.ClusterOptions{
        Addrs:    []string{":7000", ":7001", ":7002"},
        Password: "",
    })
}
```

## 四、基本操作

### 4.1 String 操作

```go
// StringDemo String 类型操作示例
func StringDemo(ctx context.Context) error {
    // SET：设置键值，支持过期时间
    err := rdb.Set(ctx, "username", "张三", 10*time.Minute).Err()
    if err != nil {
        return fmt.Errorf("SET 失败: %w", err)
    }

    // GET：获取键值
    val, err := rdb.Get(ctx, "username").Result()
    if err != nil {
        if err == redis.Nil {
            fmt.Println("键不存在")
            return nil
        }
        return fmt.Errorf("GET 失败: %w", err)
    }
    fmt.Println("username:", val)

    // INCR：自增（原子操作，适合计数器）
    count, _ := rdb.Incr(ctx, "page_view").Result()
    fmt.Println("页面访问量:", count)

    // INCRBY：按指定值自增
    rdb.IncrBy(ctx, "page_view", 10)

    // DECR：自减
    rdb.Decr(ctx, "stock")

    // SETNX：仅当键不存在时设置（适合分布式锁）
    ok, _ := rdb.SetNX(ctx, "lock:order:1001", 1, 30*time.Second).Result()
    if ok {
        fmt.Println("获取锁成功")
        // 业务逻辑...
        rdb.Del(ctx, "lock:order:1001") // 释放锁
    } else {
        fmt.Println("获取锁失败")
    }

    // MGET：批量获取
    vals, _ := rdb.MGet(ctx, "key1", "key2", "key3").Result()
    for _, v := range vals {
        fmt.Println(v)
    }

    return nil
}
```

### 4.2 Hash 操作

```go
// UserInfo 用户信息结构体
type UserInfo struct {
    Name  string `json:"name"`
    Age   int    `json:"age"`
    Email string `json:"email"`
}

// HashDemo Hash 类型操作示例
func HashDemo(ctx context.Context) error {
    // HSET：设置单个字段
    rdb.HSet(ctx, "user:1001", "name", "张三")
    rdb.HSet(ctx, "user:1001", "age", 25)
    rdb.HSet(ctx, "user:1001", "email", "zhangsan@example.com")

    // HMSET：批量设置字段
    rdb.HMSet(ctx, "user:1002",
        "name", "李四",
        "age", 30,
        "email", "lisi@example.com",
    )

    // HGET：获取单个字段
    name, _ := rdb.HGet(ctx, "user:1001", "name").Result()
    fmt.Println("用户名:", name)

    // HGETALL：获取所有字段
    fields, _ := rdb.HGetAll(ctx, "user:1001").Result()
    for field, value := range fields {
        fmt.Printf("%s: %s\n", field, value)
    }

    // HEXISTS：判断字段是否存在
    exists, _ := rdb.HExists(ctx, "user:1001", "email").Result()
    fmt.Println("email 字段存在:", exists)

    // HINCRBY：字段值自增
    rdb.HIncrBy(ctx, "user:1001", "age", 1)

    // HDEL：删除字段
    rdb.HDel(ctx, "user:1001", "email")
    return nil
}
```

### 4.3 List 操作

```go
// ListDemo List 类型操作示例
func ListDemo(ctx context.Context) error {
    // LPUSH：从左侧插入元素
    rdb.LPush(ctx, "messages", "消息1", "消息2", "消息3")

    // RPUSH：从右侧插入元素
    rdb.RPush(ctx, "messages", "消息4")

    // LRANGE：获取指定范围元素（0 到 -1 表示所有）
    msgs, _ := rdb.LRange(ctx, "messages", 0, -1).Result()
    fmt.Println("所有消息:", msgs)

    // LPOP：从左侧弹出元素
    first, _ := rdb.LPop(ctx, "messages").Result()
    fmt.Println("弹出的第一条消息:", first)

    // LLEN：获取列表长度
    length, _ := rdb.LLen(ctx, "messages").Result()
    fmt.Println("列表长度:", length)

    // BRPOP：阻塞弹出（适合作为消息队列）
    val, _ := rdb.BRPop(ctx, 5*time.Second, "queue:task").Result()
    // val[0] = key, val[1] = value

    return nil
}
```

### 4.4 Set 操作

```go
// SetDemo Set 类型操作示例
func SetDemo(ctx context.Context) error {
    // SADD：添加成员
    rdb.SAdd(ctx, "tags:article:1", "Go", "Redis", "数据库")
    rdb.SAdd(ctx, "tags:article:2", "Go", "微服务", "gRPC")

    // SMEMBERS：获取所有成员
    members, _ := rdb.SMembers(ctx, "tags:article:1").Result()
    fmt.Println("文章1的标签:", members)

    // SISMEMBER：判断成员是否存在
    exists, _ := rdb.SIsMember(ctx, "tags:article:1", "Go").Result()
    fmt.Println("包含 Go 标签:", exists)

    // 集合运算
    // SINTER：交集（两篇文章的共同标签）
    intersect, _ := rdb.SInter(ctx, "tags:article:1", "tags:article:2").Result()
    fmt.Println("共同标签:", intersect)

    // SUNION：并集
    union, _ := rdb.SUnion(ctx, "tags:article:1", "tags:article:2").Result()
    fmt.Println("所有标签:", union)

    // SDIFF：差集
    diff, _ := rdb.SDiff(ctx, "tags:article:1", "tags:article:2").Result()
    fmt.Println("文章1独有的标签:", diff)

    // SREM：移除成员
    rdb.SRem(ctx, "tags:article:1", "数据库")
    return nil
}
```

### 4.5 Sorted Set 操作

```go
// SortedSetDemo Sorted Set 类型操作示例
func SortedSetDemo(ctx context.Context) error {
    // ZADD：添加成员及分数
    rdb.ZAdd(ctx, "rank:score", redis.Z{Score: 95, Member: "张三"})
    rdb.ZAdd(ctx, "rank:score", redis.Z{Score: 88, Member: "李四"})
    rdb.ZAdd(ctx, "rank:score", redis.Z{Score: 92, Member: "王五"})
    rdb.ZAdd(ctx, "rank:score", redis.Z{Score: 76, Member: "赵六"})

    // ZREVRANGE：按分数降序获取（排行榜）
    top3, _ := rdb.ZRevRangeWithScores(ctx, "rank:score", 0, 2).Result()
    fmt.Println("--- 排行榜 TOP 3 ---")
    for i, z := range top3 {
        fmt.Printf("第%d名: %s (分数: %.0f)\n", i+1, z.Member, z.Score)
    }

    // ZINCRBY：增加成员分数
    rdb.ZIncrBy(ctx, "rank:score", 5, "李四")

    // ZSCORE：获取成员分数
    score, _ := rdb.ZScore(ctx, "rank:score", "张三").Result()
    fmt.Printf("张三的分数: %.0f\n", score)

    // ZRANK：获取成员排名（从低到高）
    rank, _ := rdb.ZRevRank(ctx, "rank:score", "张三").Result()
    fmt.Printf("张三排名: %d\n", rank+1)

    // ZREM：移除成员
    rdb.ZRem(ctx, "rank:score", "赵六")
    return nil
}
```

## 五、高级特性

### 5.1 Pipeline（管道）

Pipeline 将多个命令打包发送，减少了网络往返次数，大幅提升批量操作性能。

```go
// PipelineDemo 管道操作示例
func PipelineDemo(ctx context.Context) error {
    pipe := rdb.Pipeline()

    // 将多个命令添加到管道
    pipe.Set(ctx, "key1", "value1", 0)
    pipe.Set(ctx, "key2", "value2", 0)
    pipe.Incr(ctx, "counter")
    pipe.Get(ctx, "key1")

    // 一次性执行所有命令
    cmders, err := pipe.Exec(ctx)
    if err != nil {
        return fmt.Errorf("Pipeline 执行失败: %w", err)
    }

    // 获取每个命令的结果
    for _, cmd := range cmders {
        fmt.Printf("命令: %s, 结果: %v\n", cmd.Name(), cmd.String())
    }
    return nil
}

// PipelineSet 使用 Pipelined 方法（推荐写法）
func PipelineSet(ctx context.Context, data map[string]string) error {
    _, err := rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
        for k, v := range data {
            pipe.Set(ctx, k, v, time.Hour)
        }
        return nil
    })
    return err
}
```

### 5.2 事务（MULTI/EXEC）

Redis 事务使用 `MULTI` 和 `EXEC` 命令，go-redis 通过 `TxPipelined` 方法实现。

```go
// TransactionDemo 事务操作示例
func TransactionDemo(ctx context.Context) error {
    // 使用 TxPipelined 确保原子执行
    _, err := rdb.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
        // 监视键（乐观锁）
        // 如果在 EXEC 之前被修改，事务会失败

        current, err := pipe.Get(ctx, "stock:1001").Int()
        if err != nil {
            return err
        }
        if current <= 0 {
            return fmt.Errorf("库存不足")
        }

        // 原子递减库存
        pipe.Decr(ctx, "stock:1001")
        // 添加订单
        pipe.SAdd(ctx, "orders", "order:12345")
        return nil
    })
    return err
}
```

对于需要乐观锁的场景，使用 WATCH：

```go
// OptimisticLockDemo 乐观锁示例
func OptimisticLockDemo(ctx context.Context) error {
    const maxRetries = 3
    for i := 0; i < maxRetries; i++ {
        err := rdb.Watch(ctx, func(tx *redis.Tx) error {
            // 读取当前库存
            stock, err := tx.Get(ctx, "stock:1001").Int()
            if err != nil && err != redis.Nil {
                return err
            }
            if stock <= 0 {
                return fmt.Errorf("库存不足")
            }

            // 开启事务
            _, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
                pipe.Decr(ctx, "stock:1001")
                return nil
            })
            return err
        }, "stock:1001")

        if err == nil {
            return nil // 成功
        }
        if err == redis.TxFailedErr {
            // 事务冲突，重试
            continue
        }
        return err
    }
    return fmt.Errorf("达到最大重试次数")
}
```

### 5.3 发布/订阅（Pub/Sub）

```go
// PubSubDemo 发布订阅示例
func PubSubDemo(ctx context.Context) error {
    // 订阅频道
    pubsub := rdb.Subscribe(ctx, "channel:news")
    defer pubsub.Close()

    // 在另一个 goroutine 中接收消息
    ch := pubsub.Channel()
    go func() {
        for msg := range ch {
            fmt.Printf("收到消息: 频道=%s, 内容=%s\n", msg.Channel, msg.Payload)
        }
    }()

    // 发布消息
    for i := 0; i < 3; i++ {
        err := rdb.Publish(ctx, "channel:news", fmt.Sprintf("消息 #%d", i+1)).Err()
        if err != nil {
            return err
        }
        time.Sleep(time.Second)
    }
    return nil
}
```

### 5.4 Lua 脚本

Lua 脚本在 Redis 中原子执行，适合需要多个命令组合的复杂操作。

```go
// LuaScriptDemo Lua 脚本操作示例
func LuaScriptDemo(ctx context.Context) error {
    // 定义一个 Lua 脚本：限制 API 访问频率
    limitScript := redis.NewScript(`
        local key = KEYS[1]
        local limit = tonumber(ARGV[1])
        local expire = tonumber(ARGV[2])

        local current = redis.call('INCR', key)
        if current == 1 then
            redis.call('EXPIRE', key, expire)
        end

        if current > limit then
            return 0
        end
        return 1
    `)

    // 执行 Lua 脚本
    key := "rate:limit:api:user:1001"
    limit := 10
    expire := 60 // 60 秒内最多 10 次

    allowed, err := limitScript.Run(ctx, rdb, []string{key}, limit, expire).Int()
    if err != nil {
        return fmt.Errorf("执行 Lua 脚本失败: %w", err)
    }
    if allowed == 1 {
        fmt.Println("允许访问")
    } else {
        fmt.Println("访问频率超限")
    }
    return nil
}
```

### 5.5 键过期与删除策略

```go
// KeyExpireDemo 键过期相关操作
func KeyExpireDemo(ctx context.Context) error {
    rdb.Set(ctx, "temp:key", "value", time.Minute) // 设置时指定过期时间

    // EXPIRE：设置过期时间（秒）
    rdb.Expire(ctx, "temp:key", 10*time.Minute)

    // TTL：查看剩余过期时间
    ttl, _ := rdb.TTL(ctx, "temp:key").Result()
    fmt.Println("剩余时间:", ttl)

    // PERSIST：移除过期时间，变为永久键
    rdb.Persist(ctx, "temp:key")

    // 批量删除匹配的键（生产环境慎用 KEYS，用 SCAN）
    var cursor uint64
    for {
        var keys []string
        var err error
        keys, cursor, err = rdb.Scan(ctx, cursor, "user:*", 100).Result()
        if err != nil {
            return err
        }
        if len(keys) > 0 {
            rdb.Del(ctx, keys...)
        }
        if cursor == 0 {
            break
        }
    }
    return nil
}
```

## 六、完整示例：缓存用户信息

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"

    "github.com/redis/go-redis/v9"
)

// User 用户模型
type User struct {
    ID       int    `json:"id"`       // 用户ID
    Username string `json:"username"` // 用户名
    Age      int    `json:"age"`      // 年龄
    Email    string `json:"email"`    // 邮箱
}

// UserCache 用户缓存服务
type UserCache struct {
    rdb        *redis.Client    // Redis 客户端
    expiration time.Duration    // 缓存过期时间
}

// NewUserCache 创建用户缓存服务实例
func NewUserCache(rdb *redis.Client) *UserCache {
    return &UserCache{
        rdb:        rdb,
        expiration: 30 * time.Minute, // 默认 30 分钟过期
    }
}

// Set 缓存用户信息
func (uc *UserCache) Set(ctx context.Context, user *User) error {
    key := fmt.Sprintf("user:%d", user.ID)

    // 序列化为 JSON
    data, err := json.Marshal(user)
    if err != nil {
        return fmt.Errorf("序列化用户数据失败: %w", err)
    }

    // 存入 Redis
    return uc.rdb.Set(ctx, key, data, uc.expiration).Err()
}

// Get 获取缓存的用户信息
func (uc *UserCache) Get(ctx context.Context, userID int) (*User, error) {
    key := fmt.Sprintf("user:%d", userID)

    // 从 Redis 获取
    data, err := uc.rdb.Get(ctx, key).Bytes()
    if err != nil {
        if err == redis.Nil {
            return nil, nil // 缓存未命中
        }
        return nil, fmt.Errorf("获取缓存失败: %w", err)
    }

    // 反序列化
    var user User
    if err := json.Unmarshal(data, &user); err != nil {
        return nil, fmt.Errorf("反序列化用户数据失败: %w", err)
    }
    return &user, nil
}

// Delete 删除用户缓存
func (uc *UserCache) Delete(ctx context.Context, userID int) error {
    key := fmt.Sprintf("user:%d", userID)
    return uc.rdb.Del(ctx, key).Err()
}

// GetOrLoad 缓存穿透保护：先查缓存，缓存未命中则从数据库加载
func (uc *UserCache) GetOrLoad(ctx context.Context, userID int, loader func(int) (*User, error)) (*User, error) {
    // 先查缓存
    user, err := uc.Get(ctx, userID)
    if err != nil {
        return nil, err
    }
    if user != nil {
        return user, nil
    }

    // 缓存未命中，从数据库加载
    user, err = loader(userID)
    if err != nil {
        return nil, err
    }
    if user == nil {
        return nil, nil
    }

    // 写入缓存
    if err := uc.Set(ctx, user); err != nil {
        log.Printf("写入缓存失败: %v", err)
    }
    return user, nil
}

func main() {
    // 连接 Redis
    rdb := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "",
        DB:       0,
    })

    ctx := context.Background()
    if err := rdb.Ping(ctx).Err(); err != nil {
        log.Fatal("Redis 连接失败:", err)
    }
    defer rdb.Close()

    cache := NewUserCache(rdb)

    // 示例：缓存用户信息
    user := &User{ID: 1, Username: "张三", Age: 25, Email: "zhangsan@example.com"}
    cache.Set(ctx, user)

    // 获取缓存
    cachedUser, _ := cache.Get(ctx, 1)
    fmt.Printf("缓存用户: %+v\n", cachedUser)
}
```

## 七、最佳实践

### 7.1 缓存穿透防护

```go
// GetWithNullCache 使用空值缓存防止缓存穿透
func GetWithNullCache(ctx context.Context, userID int) (*User, error) {
    key := fmt.Sprintf("user:%d", userID)
    data, err := rdb.Get(ctx, key).Bytes()
    if err == redis.Nil {
        return nil, nil // 缓存未命中
    }
    if err != nil {
        return nil, err
    }
    // 空值标记：防止不存在的数据反复查询数据库
    if string(data) == "NULL" {
        return nil, nil
    }
    var user User
    json.Unmarshal(data, &user)
    return &user, nil
}
```

### 7.2 分布式锁

```go
// DistributedLock 分布式锁
type DistributedLock struct {
    rdb *redis.Client
}

// Lock 获取锁
func (l *DistributedLock) Lock(ctx context.Context, key string, ttl time.Duration) (bool, error) {
    ok, err := l.rdb.SetNX(ctx, key, 1, ttl).Result()
    if err != nil {
        return false, err
    }
    return ok, nil
}

// Unlock 释放锁
func (l *DistributedLock) Unlock(ctx context.Context, key string) error {
    return l.rdb.Del(ctx, key).Err()
}
```

### 7.3 连接池配置建议

```go
rdb := redis.NewClient(&redis.Options{
    Addr:         "localhost:6379",
    Password:     "",
    DB:           0,
    PoolSize:     20,              // 连接池大小，根据并发量设置
    MinIdleConns: 10,              // 最小空闲连接数
    MaxRetries:   3,               // 最大重试次数
    DialTimeout:  5 * time.Second, // 连接超时
    ReadTimeout:  3 * time.Second, // 读超时
    WriteTimeout: 3 * time.Second, // 写超时
})
```

### 7.4 使用 SCAN 代替 KEYS

```go
// ❌ 错误：KEYS 会阻塞 Redis
keys, _ := rdb.Keys(ctx, "user:*").Result()

// ✅ 正确：使用 SCAN 迭代
var cursor uint64
for {
    keys, cursor, _ = rdb.Scan(ctx, cursor, "user:*", 100).Result()
    // 处理 keys...
    if cursor == 0 {
        break
    }
}
```

### 7.5 Redis 与数据库双写一致性

```go
// UpdateUserAndCache 更新数据库后同步更新缓存
// 推荐策略：先更新数据库，再删除缓存
func UpdateUserAndCache(ctx context.Context, user *User) error {
    // 1. 更新数据库
    if err := updateDB(user); err != nil {
        return err
    }
    // 2. 删除缓存（下次查询时自动重建）
    key := fmt.Sprintf("user:%d", user.ID)
    return rdb.Del(ctx, key).Err()
}
```

## 八、参考链接

- [Redis 官方文档](https://redis.io/docs/)
- [go-redis GitHub](https://github.com/redis/go-redis)
- [go-redis 中文文档](https://redis.uptrace.dev/zh/)
