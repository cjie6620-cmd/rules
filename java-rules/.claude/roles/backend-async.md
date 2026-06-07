# 异步编程、线程池、定时任务规范

> 适用于所有后端模块，与 backend-monolith.md / backend-microservice.md 配合使用


## 一、线程池规范（阿里规约重点）

### 1. 禁止 Executors 快捷创建线程池

| 快捷方法 | 问题 | 后果 |
|---------|------|------|
| `Executors.newFixedThreadPool` | 队列 `LinkedBlockingQueue` 无界 | 任务堆积 → OOM |
| `Executors.newCachedThreadPool` | 最大线程数 `Integer.MAX_VALUE` | 线程爆炸 → OOM |
| `Executors.newSingleThreadExecutor` | 队列无界 | 任务堆积 → OOM |
| `Executors.newScheduledThreadPool` | 最大线程数无界 | 延迟任务堆积 → OOM |

**必须用 `ThreadPoolExecutor` 显式指定全部参数。**

### 2. ThreadPoolExecutor 7 大参数 + 推荐配置

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    5,                                        // corePoolSize: 核心线程数
    10,                                       // maximumPoolSize: 最大线程数
    60, TimeUnit.SECONDS,                     // keepAliveTime: 非核心线程空闲存活时间
    new LinkedBlockingQueue<>(200),            // workQueue: 有界队列（必须指定容量！）
    new ThreadFactoryBuilder()                // threadFactory: 命名线程（便于排查问题）
        .setNameFormat("order-pool-%d")
        .build(),
    new ThreadPoolExecutor.CallerRunsPolicy()  // handler: 拒绝策略
);
```

**参数配置经验值**：

| 任务类型 | corePoolSize | maximumPoolSize | 判断依据 |
|---------|-------------|-----------------|---------|
| CPU 密集型（计算、加密） | CPU 核数 + 1 | CPU 核数 * 2 | 线程太多反而增加上下文切换 |
| IO 密集型（网络、DB） | CPU 核数 * 2 | CPU 核数 * 4 | 线程大部分时间在等待 IO |
| 混合型 | 按 IO 占比调整 | 按 IO 占比调整 | `线程数 = CPU核数 * (1 + IO等待时间/CPU计算时间)` |

**拒绝策略选择**：

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| `CallerRunsPolicy` | 调用线程自己执行 | 不丢任务（推荐默认） |
| `AbortPolicy` | 抛 RejectedExecutionException | 需要感知过载 |
| `DiscardPolicy` | 静默丢弃 | 允许丢失（日志、打点） |
| `DiscardOldestPolicy` | 丢弃队列最旧的任务 | 只关心最新数据 |

### 3. 线程池命名规范

```java
// 推荐：Google Guava ThreadFactoryBuilder
ThreadFactory factory = new ThreadFactoryBuilder()
    .setNameFormat("order-async-pool-%d")
    .setUncaughtExceptionHandler((t, e) -> log.error("线程 {} 异常", t.getName(), e))
    .build();

// 或者手写
public class NamedThreadFactory implements ThreadFactory {
    private final AtomicInteger counter = new AtomicInteger(0);
    private final String prefix;

    public NamedThreadFactory(String prefix) {
        this.prefix = prefix;
    }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r, prefix + "-" + counter.incrementAndGet());
        t.setDaemon(false);
        return t;
    }
}
```

### 4. Spring @Async 使用规范

**必须自定义线程池**（Spring 默认 `SimpleAsyncTaskExecutor` 每次创建新线程，不复用！）

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("asyncPool")
    public ThreadPoolTaskExecutor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(200);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("async-pool-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);  // 关闭时等待任务完成
        executor.setAwaitTerminationSeconds(60);
        executor.initialize();
        return executor;
    }
}
```

**使用注意**：
- 方法必须 `public`（AOP 代理要求）
- **禁止在本类内部调用**（绕过代理，@Async 不生效）
- 异步方法返回 `void` 或 `CompletableFuture<T>`

```java
@Service
public class NotifyService {

    @Async("asyncPool")
    public CompletableFuture<SendResult> sendSmsAsync(String phone, String content) {
        try {
            SendResult result = smsClient.send(phone, content);
            return CompletableFuture.completedFuture(result);
        } catch (Exception e) {
            log.error("短信发送失败, phone={}", phone, e);
            return CompletableFuture.failedFuture(e);
        }
    }
}
```

### 5. CompletableFuture 使用场景

**并行查询，汇总结果**：

```java
// 并行调用多个服务，汇总结果
CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(
    () -> userService.getById(userId), bizPool);

CompletableFuture<List<Order>> orderFuture = CompletableFuture.supplyAsync(
    () -> orderService.getByUserId(userId), bizPool);

CompletableFuture<List<Coupon>> couponFuture = CompletableFuture.supplyAsync(
    () -> couponService.getByUserId(userId), bizPool);

// 等待全部完成
CompletableFuture.allOf(userFuture, orderFuture, couponFuture).join();

// 组装结果
UserDetailVO vo = new UserDetailVO();
vo.setUser(userFuture.get());
vo.setOrders(orderFuture.get());
vo.setCoupons(couponFuture.get());
```

**超时控制**（必须！）：

```java
// 方式一：get 带超时
User user = userFuture.get(3, TimeUnit.SECONDS);

// 方式二：orTimeout（Java 9+，如果项目已升级）
User user = userFuture.orTimeout(3, TimeUnit.SECONDS).join();

// 方式三：completeOnTimeout（超时返回默认值）
User user = userFuture.completeOnTimeout(defaultUser, 3, TimeUnit.SECONDS).join();
```

**异常处理**：

```java
CompletableFuture.supplyAsync(() -> riskyOperation(), pool)
    .exceptionally(ex -> {
        log.error("异步任务失败", ex);
        return defaultValue;  // 降级返回
    })
    .thenAccept(result -> {
        // 正常处理结果
    });
```

### 6. 异步任务异常处理

**核心问题**：@Async 方法抛异常不会传播到调用方（调用方已经返回了）

```java
// 方案一：方法内部 try-catch（推荐）
@Async("asyncPool")
public void sendNotification(Long userId) {
    try {
        notificationService.send(userId);
    } catch (Exception e) {
        log.error("通知发送失败, userId={}", userId, e);
        // 重要任务：落库后续重试
        retryLogService.saveRetryLog("notification", userId, e.getMessage());
    }
}

// 方案二：全局异常处理器
@Configuration
public class AsyncExceptionConfig implements AsyncConfigurer {
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("异步方法异常, method={}, params={}", method.getName(), Arrays.toString(params), ex);
        };
    }
}
```

### 7. 线程池监控（可选）

```java
// 通过 Actuator + Micrometer 暴露指标
@Bean("asyncPool")
public ThreadPoolTaskExecutor asyncExecutor() {
    // ... 常规配置 ...
    // 注册 Micrometer 指标
    MeterRegistry registry = Metrics.globalRegistry;
    Gauge.builder("thread.pool.active", executor, ThreadPoolExecutor::getActiveCount)
        .tag("pool", "async-pool")
        .register(registry);
    Gauge.builder("thread.pool.queue.size", executor, e -> e.getQueue().size())
        .tag("pool", "async-pool")
        .register(registry);
    return executor;
}
```


## 二、定时任务规范

### 1. XXL-Job 使用规范

**任务分组规范**：
- 按业务模块分组：`order-group`、`member-group`、`report-group`
- 任务命名：`{模块}_{动作}_{频率}`，如 `order_timeout_cancel`、`report_daily_generate`

**路由策略选择**：

| 策略 | 适用场景 | 说明 |
|------|---------|------|
| 轮询（ROUND） | 均匀分配，默认选择 | 依次分配到不同执行器 |
| 故障转移（FAILOVER） | 对可靠性要求高的任务 | 自动切换到健康节点 |
| 分片广播（SHARDING_BROADCAST） | 大数据量并行处理 | 所有节点同时执行，按分片参数处理不同数据 |
| 一致性HASH | 需要固定节点执行 | 同一个任务始终在同一节点执行 |

**失败重试 + 告警**：
- 重试次数：3 次，间隔递增（1s → 5s → 30s）
- 必须配置邮件/钉钉告警
- 任务超时时间必须设置（防止任务挂死，默认 30 分钟）

**幂等保障**（必须！定时任务会被重试触发）：

```java
@XxlJob("orderTimeoutCancel")
public void execute() {
    String lockKey = "job:order_timeout_cancel:" + LocalDate.now();
    boolean locked = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, "1", 24, TimeUnit.HOURS);

    if (!locked) {
        log.info("任务已执行过，跳过");
        return;
    }

    try {
        List<Order> orders = orderMapper.selectTimeoutOrders();
        log.info("查询到 {} 笔超时订单", orders.size());

        for (Order order : orders) {
            orderService.cancel(order.getId(), "超时自动取消");
        }

        XxlJobHelper.log("处理完成，共取消 {} 笔订单", orders.size());
    } catch (Exception e) {
        log.error("超时订单处理失败", e);
        XxlJobHelper.log("处理异常: " + e.getMessage());
        throw e;  // 抛出异常触发重试
    }
}
```

**分片广播示例**（大数据量场景）：

```java
@XxlJob("reportDailyGenerate")
public void execute() {
    int shardIndex = XxlJobHelper.getShardIndex();  // 当前分片序号
    int shardTotal = XxlJobHelper.getShardTotal();   // 总分片数

    // 按用户 ID 取模分配数据
    List<Long> userIds = userService.getAllUserIds();
    List<Long> myUsers = userIds.stream()
        .filter(id -> id % shardTotal == shardIndex)
        .collect(Collectors.toList());

    log.info("分片 {}/{}, 处理 {} 个用户", shardIndex, shardTotal, myUsers.size());
    myUsers.forEach(this::generateReport);
}
```

### 2. @Scheduled 使用规范

**仅用于简单的本地定时任务**（不需要分布式调度、不需要管理界面）

```java
@Component
@EnableScheduling
public class LocalScheduledTasks {

    // cron 表达式：每天凌晨 2 点执行
    @Scheduled(cron = "0 0 2 * * ?")
    public void cleanExpiredData() {
        log.info("开始清理过期数据");
        // ...
    }

    // fixedDelay：上一次执行完成后延迟 60 秒再执行（适合有先后依赖的任务）
    @Scheduled(fixedDelay = 60000)
    public void syncData() {
        // ...
    }

    // fixedRate：每 30 秒执行一次（无论上一次是否完成）
    @Scheduled(fixedRate = 30000)
    public void heartbeat() {
        // ...
    }
}
```

**cron vs fixedDelay vs fixedRate 区别**：

| 属性 | 触发时机 | 适用场景 |
|------|---------|---------|
| `cron` | 按 cron 表达式定时触发 | 定时报表、定时清理 |
| `fixedDelay` | 上一次**完成**后延迟 N 毫秒 | 有先后依赖的轮询任务 |
| `fixedRate` | 上一次**开始**后 N 毫秒触发 | 心跳、状态同步 |

### 3. 任务执行日志规范

每个定时任务入口必须记录：

```java
@XxlJob("xxxTask")
public void execute() {
    long start = System.currentTimeMillis();
    log.info("===== 任务开始: xxxTask =====");

    try {
        int count = doBusiness();
        long cost = System.currentTimeMillis() - start;
        log.info("===== 任务完成: xxxTask, 处理 {} 条, 耗时 {}ms =====", count, cost);
    } catch (Exception e) {
        long cost = System.currentTimeMillis() - start;
        log.error("===== 任务异常: xxxTask, 耗时 {}ms =====", cost, e);
        throw e;
    }
}
```


## 三、禁止事项

- **禁止 Executors 快捷创建线程池**（OOM 风险，必须用 ThreadPoolExecutor 显式指定参数）
- **禁止 workQueue 不设容量**（`LinkedBlockingQueue` 默认 `Integer.MAX_VALUE` = OOM）
- **禁止无超时的异步调用**（`CompletableFuture.get()` 必须带 timeout）
- **禁止 @Async 方法内部调用本类方法**（AOP 代理不生效，异步不执行）
- **禁止在 Spring Bean 中手动 `new Thread()`**（不受容器管理，无法优雅关闭）
- **禁止定时任务无幂等处理**（MQ 重试 / XXL-Job 重试都会导致重复执行）
- **禁止任务内写死 `Thread.sleep()`**（用延时队列或定时调度替代）
- **禁止无超时控制的定时任务**（XXL-Job 必须设置超时时间）
- **禁止无监控的线程池**（线上必须有线程池活跃数、队列积压的监控告警）

---

## 开发规则整合

### 架构设计
- 优先采用当前主流且经过生产验证的企业级方案
- 以中型公司实际落地标准设计
- 满足业务需求即可，不允许过度设计

### 编码原则
- 使用最少代码完成需求
- 优先可读性，其次是代码量
- 避免重复代码（DRY）

### 代码要求
- 所有代码必须包含中文注释
- 必须进行必要的判空处理
- 必须进行必要的异常处理

### 性能原则
- 先保证正确性
- 再保证可维护性
- 最后再考虑性能优化
