# ===== 缓存策略、Redis 使用规范 =====

> 适用于所有使用 Redis 的后端模块，与 backend-monolith.md / backend-microservice.md 配合使用

---

## 一、缓存策略

### 1. 缓存模式选择

| 模式 | 适用场景 | 一致性 | 复杂度 |
|------|---------|--------|--------|
| **Cache-Aside（旁路缓存）** | 读多写少，通用场景 | 最终一致 | 低 |
| Read-Through | 缓存层封装数据源访问 | 最终一致 | 中 |
| Write-Behind | 写多读少，允许短暂不一致 | 弱一致 | 高 |

**默认用 Cache-Aside**，流程：

```
读：先查缓存 → 命中返回 → 未命中查 DB → 写入缓存 → 返回
写：先更新 DB → 再删除缓存（不是更新缓存）
```

### 2. Spring @Cacheable 使用规范

**Key 命名规范**：`{业务}:{模块}:{标识}`

```java
// 正确：明确的业务语义 key
@Cacheable(value = "user:info", key = "#userId")
public UserVO getUserById(Long userId) {
    return userMapper.selectById(userId);
}

// 正确：复合 key
@Cacheable(value = "order:list", key = "#userId + ':' + #status")
public List<OrderVO> getOrders(Long userId, Integer status) {
    return orderMapper.selectByUserAndStatus(userId, status);
}

// 错误：无意义 key
@Cacheable(value = "cache1", key = "#id")  // "cache1" 没有业务含义
```

**@CacheEvict 使用**：删除/更新数据时必须同步清缓存

```java
@CacheEvict(value = "user:info", key = "#userId")
public void updateUser(Long userId, UserDTO dto) {
    userMapper.updateById(userId, dto);
}
```

**RedisCacheManager 配置模板**：

```java
@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        // 默认配置
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()))
            .prefixCacheNameWith("cache:");

        // 不同业务不同 TTL
        Map<String, RedisCacheConfiguration> configMap = new HashMap<>();
        configMap.put("user:info", defaultConfig.entryTtl(Duration.ofMinutes(10)));
        configMap.put("config:dict", defaultConfig.entryTtl(Duration.ofHours(24)));
        configMap.put("order:list", defaultConfig.entryTtl(Duration.ofMinutes(5)));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(configMap)
            .build();
    }
}
```

### 3. 缓存穿透防护

**问题**：查询不存在的数据，每次都打到 DB（如 `getUserById(-1)`）

**方案一：空值缓存**（推荐，简单有效）

```java
public UserVO getUserById(Long userId) {
    String key = "user:info:" + userId;
    String cached = redisTemplate.opsForValue().get(key);

    // 空值标记
    if ("NULL".equals(cached)) {
        return null;
    }
    if (cached != null) {
        return JSONUtil.parseObj(cached).toBean(UserVO.class);
    }

    UserVO user = userMapper.selectById(userId);
    if (user == null) {
        // 缓存空值，短 TTL 防止长期占用
        redisTemplate.opsForValue().set(key, "NULL", 2, TimeUnit.MINUTES);
    } else {
        redisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(user), 30, TimeUnit.MINUTES);
    }
    return user;
}
```

**方案二：布隆过滤器**（数据量大时）

```java
// 初始化时加载所有合法 ID
RBloomFilter<Long> filter = redissonClient.getBloomFilter("userFilter");
filter.tryInit(1000000L, 0.01);  // 预计100万数据，误判率1%

// 查询前先过滤
if (!filter.contains(userId)) {
    return null;  // 一定不存在，直接返回
}
// 可能存在，继续查缓存/DB
```

### 4. 缓存击穿防护

**问题**：热点 key 过期瞬间大量请求同时打到 DB

**方案：互斥锁**（只让一个线程查 DB，其他线程等待重试）

```java
public UserVO getUserById(Long userId) {
    String key = "user:info:" + userId;
    String cached = redisTemplate.opsForValue().get(key);
    if (cached != null) {
        return JSONUtil.parseObj(cached).toBean(UserVO.class);
    }

    String lockKey = "lock:user:info:" + userId;
    RLock lock = redissonClient.getLock(lockKey);
    try {
        // 只等 3 秒，获取不到锁直接返回降级数据
        if (lock.tryLock(3, 10, TimeUnit.SECONDS)) {
            // 双重检查（拿到锁后再查一次缓存，可能其他线程已经写入了）
            cached = redisTemplate.opsForValue().get(key);
            if (cached != null) {
                return JSONUtil.parseObj(cached).toBean(UserVO.class);
            }
            UserVO user = userMapper.selectById(userId);
            redisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(user), 30, TimeUnit.MINUTES);
            return user;
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } finally {
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
    // 获取锁失败，返回降级数据或抛异常
    return null;
}
```

### 5. 缓存雪崩防护

**问题**：大量 key 同时过期，DB 瞬间压力暴增

**方案一：过期时间加随机偏移**（推荐，零成本）

```java
// 基础 TTL + 随机偏移，让不同 key 的过期时间分散开
long baseTtl = 30;  // 基础 30 分钟
long randomOffset = ThreadLocalRandom.current().nextLong(0, 300);  // 随机 0~5 分钟
long finalTtl = baseTtl * 60 + randomOffset;
redisTemplate.opsForValue().set(key, value, finalTtl, TimeUnit.SECONDS);
```

**方案二：多级缓存**（高并发场景）

```
请求 → Caffeine（L1 本地，纳秒级） → Redis（L2 分布式） → DB
```

```java
// Caffeine L1 缓存（1000 条，5 分钟过期）
private final Cache<String, Object> localCache = Caffeine.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(5, TimeUnit.MINUTES)
    .build();

public UserVO getUserById(Long userId) {
    String key = "user:info:" + userId;

    // 1. 查 L1
    Object local = localCache.getIfPresent(key);
    if (local != null) {
        return (UserVO) local;
    }

    // 2. 查 L2
    String cached = redisTemplate.opsForValue().get(key);
    if (cached != null) {
        UserVO user = JSONUtil.parseObj(cached).toBean(UserVO.class);
        localCache.put(key, user);  // 回填 L1
        return user;
    }

    // 3. 查 DB
    UserVO user = userMapper.selectById(userId);
    if (user != null) {
        redisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(user), 30, TimeUnit.MINUTES);
        localCache.put(key, user);
    }
    return user;
}
```

### 6. 本地缓存 vs 分布式缓存选择

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 单机配置/字典数据 | Caffeine（本地） | 无网络开销，纳秒级响应 |
| 分布式 Session / 分布式锁 | Redis（分布式） | 跨节点共享 |
| 高频读 + 允许短暂不一致 | Caffeine L1 + Redis L2 | 兼顾性能和一致性 |
| 需要持久化 | Redis | 本地缓存重启丢失 |
| 排行榜/计数器 | Redis（ZSet/INCR） | 原子操作 + 持久化 |

### 7. 缓存一致性方案

**延迟双删**（适用于对一致性要求较高的场景）：

```java
public void updateUser(Long userId, UserDTO dto) {
    String key = "user:info:" + userId;

    // 1. 先删缓存
    redisTemplate.delete(key);

    // 2. 更新 DB
    userMapper.updateById(userId, dto);

    // 3. 延迟再删一次（防止并发读请求在步骤2之后把旧数据写回缓存）
    CompletableFuture.runAsync(() -> {
        try {
            Thread.sleep(500);
            redisTemplate.delete(key);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    });
}
```

**注意**：第二次删除失败要兜底（Canal 监听 binlog 或消息队列异步补偿）

---

## 二、Redis 使用规范

### 1. Key 设计规范

- **格式**：`{业务}:{模块}:{标识}`，如 `user:info:1001`、`order:lock:2001`、`config:dict:sys`
- **长度**：控制在 128 字节以内
- **禁止**：无意义 key（`key1`、`test`）、中文 key、特殊字符
- **对象存储**：优先用 Hash 结构（比 JSON String 更省内存，支持单字段更新）

```java
// 推荐：Hash 存储对象
redisTemplate.opsForHash().put("user:info:1001", "name", "张三");
redisTemplate.opsForHash().put("user:info:1001", "age", "25");

// 不推荐：JSON String 存对象（更新单个字段要读改写整条数据）
redisTemplate.opsForValue().set("user:info:1001", "{\"name\":\"张三\",\"age\":25}");
```

### 2. 数据结构选择决策表

| 需求 | 数据结构 | 典型场景 |
|------|---------|---------|
| 缓存对象 | Hash | 用户信息、配置项 |
| 排行榜/延时队列 | ZSet | 积分排行、延迟任务 |
| 消息队列（轻量） | List / Stream | 通知队列、日志缓冲 |
| 标签/去重 | Set | 用户标签、IP 黑名单 |
| 分布式锁 | String（SETNX） | 订单防重复提交 |
| 计数器 | String（INCR） | 访问量统计、限流计数 |
| 分布式限流 | String（Lua 脚本） | 接口限流（滑动窗口） |
| 最新动态/时间线 | List（LPUSH + LTRIM） | 动态流、最新消息 |

### 3. 大 Key 治理

**定义**：
- String 类型 value > 10KB
- Hash/Set/List/ZSet 元素数量 > 5000 个

**治理方案**：
- 拆分为多个小 key（如 `user:info:1001:base`、`user:info:1001:extra`）
- Hash 字段过多时按时间/业务拆分为多个 Hash
- 删除大 key 用 `UNLINK`（异步删除，不阻塞主线程），禁止用 `DEL`

```java
// 正确：异步删除大 key
redisTemplate.unlink("big:key");

// 错误：阻塞删除
redisTemplate.delete("big:key");
```

### 4. Pipeline 批量操作

**场景**：批量查询/写入 > 10 条以上，减少网络 RTT

```java
List<Object> results = redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
    for (Long userId : userIds) {
        connection.get(("user:info:" + userId).getBytes());
    }
    return null;
});
```

### 5. Lua 脚本使用场景

**场景一：分布式锁（原子性 check + set + expire）**

```java
String luaScript = """
    if redis.call('get', KEYS[1]) == ARGV[1] then
        return redis.call('del', KEYS[1])
    else
        return 0
    end
    """;

DefaultRedisScript<Long> script = new DefaultRedisScript<>(luaScript, Long.class);
redisTemplate.execute(script, List.of("lock:order:" + orderId), lockValue);
```

**场景二：滑动窗口限流器**

```java
String luaScript = """
    local key = KEYS[1]
    local limit = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])

    redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
    local count = redis.call('ZCARD', key)

    if count < limit then
        redis.call('ZADD', key, now, now .. math.random())
        redis.call('EXPIRE', key, window / 1000)
        return 1
    else
        return 0
    end
    """;
```

---

## 三、禁止事项

- **禁止 `KEYS *`**（线上致命，遍历所有 key 导致阻塞，用 `SCAN` 替代）
- **禁止无过期时间**（除系统配置类 key 外，所有缓存必须设 TTL）
- **禁止存大对象**（> 1MB 必须拆分或压缩）
- **禁止在 Redis 中存业务逻辑**（Redis 只存数据，不存计算逻辑）
- **禁止在循环中频繁调用 Redis**（用 Pipeline 或 Hash 批量操作）
- **禁止用 `SELECT` 切换数据库**（统一用 db0，多实例隔离而非多库）
- **禁止用 `DEL` 删除大 key**（用 `UNLINK` 异步删除）
- **禁止序列化方式不一致**（统一用 Jackson JSON，禁止混用 JDK 序列化和 JSON 序列化）
