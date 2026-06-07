# AOP 使用规范

> 适用于所有后端模块，与 backend-patterns.md 配合使用


## 一、AOP 适用场景

### 何时用 AOP vs 普通方法调用

| 场景 | 方案 | 判断依据 |
|------|------|---------|
| 横切关注点（日志、权限、限流、幂等） | **AOP** | 与业务逻辑无关，跨多个模块 |
| 业务流程中的公共步骤 | **普通方法/模板方法** | 属于业务逻辑的一部分 |
| 单个类内的公共逻辑 | **私有方法** | 不涉及跨类复用 |
| 多个类的公共数据处理 | **工具类/Helper** | 纯数据转换，无副作用 |

**一句话判断**：如果这个逻辑"跟业务无关、到处都要用" → AOP；"跟业务相关、只有特定流程需要" → 普通方法。

### 常见 AOP 场景

| 场景 | 注解 | 说明 |
|------|------|------|
| 操作日志 | `@Log` | 记录谁在什么时间做了什么操作 |
| 接口幂等 | `@Idempotent` | 防止重复提交（基于 Redis） |
| 接口限流 | `@RateLimiter` | 限制接口访问频率（滑动窗口） |
| 数据权限 | `@DataScope` | 自动追加数据过滤条件 |
| 分布式锁 | `@DistributedLock` | 方法级分布式锁 |
| 参数校验增强 | `@CheckParam` | 自定义复杂校验逻辑 |


## 二、自定义注解 + AOP 标准模板

### 通用结构（所有切面照此模板写）

```java
// ========== 第一步：定义注解 ==========
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RateLimiter {
    /** 每秒允许的请求数 */
    int value() default 10;
    /** 限流 key（支持 SpEL） */
    String key() default "";
    /** 超限提示信息 */
    String message() default "请求过于频繁，请稍后再试";
}

// ========== 第二步：定义切面 ==========
@Aspect
@Component
@Slf4j
public class RateLimiterAspect {

    @Resource
    private StringRedisTemplate redisTemplate;

    @Around("@annotation(rateLimiter)")
    public Object around(ProceedingJoinPoint point, RateLimiter rateLimiter) throws Throwable {
        // 1. 解析限流 key
        String key = resolveKey(point, rateLimiter);

        // 2. 执行限流判断
        boolean allowed = tryAcquire(key, rateLimiter.value());
        if (!allowed) {
            throw new BizException(rateLimiter.message());
        }

        // 3. 执行目标方法
        return point.proceed();
    }

    private String resolveKey(ProceedingJoinPoint point, RateLimiter rateLimiter) {
        // 优先用注解指定的 key
        if (StringUtils.isNotBlank(rateLimiter.key())) {
            return SpELUtil.parse(rateLimiter.key(), point);
        }
        // 默认：类名 + 方法名
        String className = point.getTarget().getClass().getSimpleName();
        String methodName = point.getSignature().getName();
        return "rate:" + className + ":" + methodName;
    }

    private boolean tryAcquire(String key, int limit) {
        // 滑动窗口限流（Lua 脚本实现，原子操作）
        String luaScript = """
            local key = KEYS[1]
            local limit = tonumber(ARGV[1])
            local window = 1000
            local now = tonumber(ARGV[2])
            redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
            local count = redis.call('ZCARD', key)
            if count < limit then
                redis.call('ZADD', key, now, now .. math.random())
                redis.call('EXPIRE', key, 1)
                return 1
            end
            return 0
            """;
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            List.of(key),
            String.valueOf(limit),
            String.valueOf(System.currentTimeMillis())
        );
        return Long.valueOf(1L).equals(result);
    }
}
```

### 使用方式

```java
@RestController
@RequestMapping("/api/order")
public class OrderController {

    @PostMapping("/create")
    @RateLimiter(value = 5, key = "#request.remoteAddr")  // 每个 IP 每秒最多 5 次
    public R<Long> createOrder(@RequestBody OrderDTO dto) {
        return R.ok(orderService.create(dto));
    }
}
```


## 三、切面执行顺序控制

多个切面同时作用于一个方法时，需要控制执行顺序。

### 方式一：@Order 注解（推荐）

```java
@Aspect
@Component
@Order(1)  // 数字越小越先执行（外层）
public class LogAspect { ... }

@Aspect
@Component
@Order(2)  // 后执行（内层）
public class AuthAspect { ... }

```

### 方式二：实现 Ordered 接口

```java
@Aspect
@Component
public class SecurityAspect implements Ordered {
    @Override
    public int getOrder() {
        return 1;  // 先执行
    }
}
```

### 执行顺序速查

| 场景 | 推荐 @Order |
|------|------------|
| 日志切面 | `@Order(1)`（最先记录，最后完成） |
| 权限/认证切面 | `@Order(2)` |
| 限流切面 | `@Order(3)` |
| 幂等切面 | `@Order(4)` |
| 数据权限切面 | `@Order(5)`（最内层） |


## 四、常用切面模板

### 1. 操作日志切面（@Log）

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Log {
    String module() default "";     // 模块名
    String action() default "";     // 操作描述
    boolean saveParams() default true;  // 是否保存请求参数
    boolean saveResult() default false; // 是否保存返回结果
}

@Aspect
@Component
@Slf4j
public class LogAspect {

    @Resource
    private OperateLogMapper operateLogMapper;

    @Around("@annotation(logAnnotation)")
    public Object around(ProceedingJoinPoint point, Log logAnnotation) throws Throwable {
        long startTime = System.currentTimeMillis();

        // 获取请求信息
        HttpServletRequest request = ((ServletRequestAttributes)
            RequestContextHolder.getRequestAttributes()).getRequest();

        OperateLog operateLog = new OperateLog();
        operateLog.setModule(logAnnotation.module());
        operateLog.setAction(logAnnotation.action());
        operateLog.setMethod(point.getSignature().getName());
        operateLog.setRequestUrl(request.getRequestURI());
        operateLog.setRequestMethod(request.getMethod());
        operateLog.setIp(IpUtil.getClientIp(request));
        operateLog.setOperateUser(SecurityUtil.getCurrentUserName());

        if (logAnnotation.saveParams()) {
            operateLog.setParams(JSONUtil.toJsonStr(point.getArgs()));
        }

        Object result;
        try {
            result = point.proceed();
            operateLog.setStatus(1);
        } catch (Exception e) {
            operateLog.setStatus(0);
            operateLog.setErrorMsg(e.getMessage());
            throw e;
        } finally {
            operateLog.setCostTime(System.currentTimeMillis() - startTime);
            if (logAnnotation.saveResult()) {
                operateLog.setResult(JSONUtil.toJsonStr(result));
            }
            // 异步保存日志，不阻塞主流程
            CompletableFuture.runAsync(() -> operateLogMapper.insert(operateLog));
        }
        return result;
    }
}
```

### 2. 接口幂等切面（@Idempotent）

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Idempotent {
    /** 幂等 key 的 SpEL 表达式，默认用 token + URL */
    String key() default "";
    /** 锁过期时间（秒） */
    int expireSeconds() default 5;
    /** 重复请求提示信息 */
    String message() default "请勿重复提交";
}

@Aspect
@Component
public class IdempotentAspect {

    @Resource
    private StringRedisTemplate redisTemplate;

    @Around("@annotation(idempotent)")
    public Object around(ProceedingJoinPoint point, Idempotent idempotent) throws Throwable {
        String key = buildKey(point, idempotent);

        // 尝试加锁（SETNX）
        Boolean success = redisTemplate.opsForValue()
            .setIfAbsent(key, "1", idempotent.expireSeconds(), TimeUnit.SECONDS);

        if (!success) {
            throw new BizException(idempotent.message());
        }

        try {
            return point.proceed();
        } finally {
            // 执行完成后删除锁（允许下次请求）
            redisTemplate.delete(key);
        }
    }

    private String buildKey(ProceedingJoinPoint point, Idempotent idempotent) {
        if (StringUtils.isNotBlank(idempotent.key())) {
            return "idempotent:" + SpELUtil.parse(idempotent.key(), point);
        }
        // 默认：用户 token + 请求 URL
        HttpServletRequest request = ((ServletRequestAttributes)
            RequestContextHolder.getRequestAttributes()).getRequest();
        String token = request.getHeader("Authorization");
        return "idempotent:" + DigestUtil.md5Hex(token + ":" + request.getRequestURI());
    }
}
```

### 3. 接口限流切面（@RateLimiter）

见上方"通用结构"中的完整代码。


## 五、参数获取技巧

### SpEL 表达式解析工具

```java
public class SpELUtil {

    private static final ExpressionParser PARSER = new SpelExpressionParser();

    /**
     * 解析 SpEL 表达式
     * 支持：#userId、#dto.name、#request.remoteAddr
     */
    public static String parse(String expression, JoinPoint point) {
        StandardEvaluationContext context = new StandardEvaluationContext();

        // 注入方法参数
        MethodSignature signature = (MethodSignature) point.getSignature();
        String[] paramNames = signature.getParameterNames();
        Object[] args = point.getArgs();
        for (int i = 0; i < paramNames.length; i++) {
            context.setVariable(paramNames[i], args[i]);
        }

        return PARSER.parseExpression(expression).getValue(context, String.class);
    }
}
```

### 获取请求信息

```java
// 获取 HttpServletRequest
HttpServletRequest request = ((ServletRequestAttributes)
    RequestContextHolder.getRequestAttributes()).getRequest();

// 获取方法注解
MethodSignature signature = (MethodSignature) point.getSignature();
Method method = signature.getMethod();
Log logAnnotation = method.getAnnotation(Log.class);

// 获取方法名、类名
String className = point.getTarget().getClass().getSimpleName();
String methodName = point.getSignature().getName();

// 获取参数
Object[] args = point.getArgs();
String[] paramNames = ((MethodSignature) point.getSignature()).getParameterNames();
```


## 六、Spring AOP 代理机制注意事项

| 注意点 | 说明 |
|--------|------|
| **同类内部调用不生效** | `this.method()` 绕过代理，AOP 不触发 |
| **private 方法不生效** | AOP 基于代理，private 方法无法被代理 |
| **final 类/方法不生效** | CGLIB 无法代理 final |
| **必须 Spring 管理的 Bean** | 手动 `new` 的对象没有 AOP 增强 |

**解决方案**：同类内部调用需要 AOP 时，注入自身代理：

```java
@Service
public class OrderService {

    @Resource
    private ApplicationContext context;

    public void createOrder(OrderDTO dto) {
        // 需要调用自己的 @Log 方法时
        OrderService proxy = context.getBean(OrderService.class);
        proxy.auditLog(dto);  // 通过代理调用，AOP 生效
    }

    @Log(module = "订单", action = "审计日志")
    public void auditLog(OrderDTO dto) { ... }
}
```


## 七、禁止事项

- **禁止在切面中写业务逻辑**（切面只做横切关注点，业务逻辑放 Service）
- **禁止切面嵌套过深**（最多 3 层，超过说明设计有问题）
- **禁止切面内吞掉异常**（catch 后必须重新抛出或处理，不能静默吞掉）
- **禁止 @Around 不调用 `point.proceed()`**（除非你明确要拦截阻止方法执行）
- **禁止在切面中调用耗时操作**（如发 HTTP 请求、查数据库，会拖慢业务方法）
- **禁止同类内部调用 AOP 方法而不做特殊处理**（AOP 不生效，要通过注入自身代理解决）
- **禁止 AOP 切面同时做多件事**（一个切面只关注一件事：日志就日志，限流就限流）

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
