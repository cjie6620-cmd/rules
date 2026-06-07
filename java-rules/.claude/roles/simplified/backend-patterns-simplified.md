# 设计模式、工具类封装、Client封装规范

> 适用于所有后端模块，与 backend-monolith.md / backend-microservice.md 配合使用


## 一、设计模式（只用在真正需要的地方，禁止为了用而用）

### 总原则

- **解决什么问题，就用什么模式**，不要上来就套模式
- 一个模式如果让代码更难理解，就不要用
- Spring 本身就是一堆设计模式的集合（IoC=工厂+单例、AOP=代理、Events=观察者），**优先用 Spring 提供的机制**，不要自己造轮子


### 1. 策略模式（Strategy）— 最常用，替代 if-else 地狱

**场景**：同一操作有多种实现方式，按类型选择不同逻辑（支付方式、消息通知渠道、导出格式、审批规则）

**判断标准**：if-else / switch 超过 3 个分支，且每个分支是独立的业务逻辑 → 用策略模式

```java
// ========== 1. 定义策略接口 ==========
public interface PayStrategy {
    /** 支付方式标识，如 "alipay"、"wechat" */
    String getType();
    /** 执行支付 */
    PayResult pay(PayRequest request);
}

// ========== 2. 各策略实现 ==========
@Component
public class AlipayStrategy implements PayStrategy {
    @Override
    public String getType() { return "alipay"; }

    @Override
    public PayResult pay(PayRequest request) {
        // 调用支付宝 SDK
    }
}

@Component
public class WechatPayStrategy implements PayStrategy {
    @Override
    public String getType() { return "wechat"; }

    @Override
    public PayResult pay(PayRequest request) {
        // 调用微信支付 SDK
    }
}

// ========== 3. 策略工厂（Spring 自动注入所有实现） ==========
@Component
public class PayStrategyFactory {

    private final Map<String, PayStrategy> strategyMap;

    // Spring 会自动把所有 PayStrategy 实现注入到 List 里
    public PayStrategyFactory(List<PayStrategy> strategies) {
        this.strategyMap = strategies.stream()
            .collect(Collectors.toMap(PayStrategy::getType, Function.identity()));
    }

    public PayStrategy getStrategy(String type) {
        PayStrategy strategy = strategyMap.get(type);
        if (strategy == null) {
            throw new BusinessException("不支持的支付方式: " + type);
        }
        return strategy;
    }
}

// ========== 4. 使用 ==========
@Service
@RequiredArgsConstructor
public class PayServiceImpl implements PayService {

    private final PayStrategyFactory payStrategyFactory;

    @Override
    public PayResult pay(String payType, PayRequest request) {
        PayStrategy strategy = payStrategyFactory.getStrategy(payType);
        return strategy.pay(request);
    }
}
```

**目录结构**：
```
modules/xxx/
├── strategy/                    # 策略目录
│   ├── PayStrategy.java         # 接口
│   ├── PayStrategyFactory.java  # 工厂
│   ├── AlipayStrategy.java      # 支付宝实现
│   └── WechatPayStrategy.java   # 微信实现
```


### 2. 模板方法模式（Template Method）— 流程相同，个别步骤不同

**场景**：多个业务有相同流程骨架（校验 → 处理 → 保存 → 通知），但个别步骤实现不同

```java
// ========== 抽象模板 ==========
public abstract class AbstractImportHandler<T> {

    /**
     * 导入数据的完整流程
     * @return 成功导入条数
     */
    public int importData(InputStream inputStream) {
        // 1. 解析文件（子类决定格式）
        List<T> dataList = parseFile(inputStream);
        // 2. 校验数据（子类决定规则）
        validateData(dataList);
        // 3. 保存数据（子类决定存哪张表）
        saveData(dataList);
        return dataList.size();
    }

    /** 解析文件：Excel / CSV 由子类实现 */
    protected abstract List<T> parseFile(InputStream inputStream);

    /** 校验数据：默认空实现，子类按需覆盖 */
    protected void validateData(List<T> dataList) {
        // 默认不做额外校验
    }

    /** 保存数据：子类实现具体持久化逻辑 */
    protected abstract void saveData(List<T> dataList);
}

// ========== 具体实现 ==========
@Component
public class UserImportHandler extends AbstractImportHandler<UserImportDTO> {

    private final UserMapper userMapper;

    @Override
    protected List<UserImportDTO> parseFile(InputStream inputStream) {
        // 用 EasyExcel 解析 Excel
        return EasyExcel.read(inputStream, UserImportDTO.class, null)
            .sheet().doReadSync();
    }

    @Override
    protected void validateData(List<UserImportDTO> dataList) {
        // 校验邮箱格式、手机号唯一性等
    }

    @Override
    protected void saveData(List<UserImportDTO> dataList) {
        List<User> users = dataList.stream()
            .map(this::convertToEntity)
            .collect(Collectors.toList());
        userMapper.insertBatch(users);
    }
}
```


### 3. 观察者模式 / Spring Events — 解耦"做完一件事后的后续操作"

**场景**：用户注册后发短信 + 发优惠券 + 写日志，这些后续操作不应该耦合在注册逻辑里

```java
// ========== 1. 定义事件 ==========
@Getter
public class UserRegisterEvent extends ApplicationEvent {
    private final Long userId;
    private final String phone;

    public UserRegisterEvent(Object source, Long userId, String phone) {
        super(source);
        this.userId = userId;
        this.phone = phone;
    }
}

// ========== 2. 发布事件（在 Service 里） ==========
@Service
@RequiredArgsConstructor
public class UserServiceImpl implements UserService {

    private final ApplicationEventPublisher eventPublisher;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void register(UserRegisterDTO dto) {
        // 1. 保存用户
        userMapper.insert(user);
        // 2. 发布事件，后续操作异步处理，不耦合
        eventPublisher.publishEvent(new UserRegisterEvent(this, user.getId(), dto.getPhone()));
    }
}

// ========== 3. 监听事件（各处理器独立，互不影响） ==========
@Component
@Slf4j
public class UserRegisterListener {

    @Async
    @EventListener
    public void sendWelcomeSms(UserRegisterEvent event) {
        smsService.send(event.getPhone(), "欢迎注册！");
    }

    @Async
    @EventListener
    public void sendCoupon(UserRegisterEvent event) {
        couponService.grantNewUserCoupon(event.getUserId());
    }

    @EventListener
    public void recordLog(UserRegisterEvent event) {
        log.info("用户注册: userId={}", event.getUserId());
    }
}
```

**好处**：新增"注册后送积分"只需要加一个 `@EventListener` 方法，不用改注册逻辑


### 4. 责任链模式（Chain of Responsibility）— 多个处理步骤按顺序执行

**场景**：审批流程（组长 → 经理 → 总监）、请求过滤器链、多步骤校验

```java
// ========== 1. 抽象处理器 ==========
public abstract class ApprovalHandler {

    private ApprovalHandler next;

    public ApprovalHandler setNext(ApprovalHandler next) {
        this.next = next;
        return next;
    }

    public final void handle(ApprovalContext context) {
        if (!canHandle(context)) {
            if (next != null) {
                next.handle(context);
            }
            return;
        }
        doHandle(context);
    }

    /** 当前节点是否能处理（根据金额、类型等判断） */
    protected abstract boolean canHandle(ApprovalContext context);

    /** 执行审批处理 */
    protected abstract void doHandle(ApprovalContext context);
}

// ========== 2. 具体处理器 ==========
@Component
public class TeamLeaderApprovalHandler extends ApprovalHandler {
    @Override
    protected boolean canHandle(ApprovalContext ctx) {
        return ctx.getAmount().compareTo(BigDecimal.valueOf(1000)) <= 0;
    }

    @Override
    protected void doHandle(ApprovalContext ctx) {
        ctx.setApprover("组长");
        ctx.setStatus(ApprovalStatus.APPROVED);
    }
}

@Component
public class ManagerApprovalHandler extends ApprovalHandler {
    @Override
    protected boolean canHandle(ApprovalContext ctx) {
        return ctx.getAmount().compareTo(BigDecimal.valueOf(10000)) <= 0;
    }

    @Override
    protected void doHandle(ApprovalContext ctx) {
        ctx.setApprover("经理");
        ctx.setStatus(ApprovalStatus.APPROVED);
    }
}

// ========== 3. 组装责任链（配置类） ==========
@Configuration
public class ApprovalChainConfig {

    @Bean
    public ApprovalHandler approvalChain(
            TeamLeaderApprovalHandler teamLeader,
            ManagerApprovalHandler manager,
            DirectorApprovalHandler director) {
        teamLeader.setNext(manager).setNext(director);
        return teamLeader;  // 返回链头
    }
}

// ========== 4. 使用 ==========
@Service
@RequiredArgsConstructor
public class ApprovalServiceImpl implements ApprovalService {

    private final ApprovalHandler approvalChain;

    @Override
    public void approve(ApprovalRequest request) {
        ApprovalContext ctx = new ApprovalContext(request);
        approvalChain.handle(ctx);
        // 保存审批结果
    }
}
```


### 5. 建造者模式（Builder）— 对象字段多、创建步骤复杂

**场景**：查询条件构造、复杂对象组装（搜索请求、报表配置）

**Spring Boot 项目中最常见的场景**：用 MyBatis-Plus 的 `LambdaQueryWrapper` 就是建造者模式

```java
// 自己写的 Builder 示例
SearchRequest request = SearchRequest.builder()
    .keyword("手机")
    .categoryId(100L)
    .priceRange(Range.closed(BigDecimal.valueOf(100), BigDecimal.valueOf(5000)))
    .sort(SortRule.priceAsc())
    .page(1)
    .size(20)
    .build();
```

**规则**：
- 字段超过 5 个且不是所有字段都必传 → 考虑 Builder
- Lombok 的 `@Builder` 够用就用，不需要手写
- DTO 如果用了 `@Builder`，同时加 `@NoArgsConstructor` + `@AllArgsConstructor`（保证 JSON 反序列化正常）


### 6. 工厂模式（Factory）— 用 Spring 容器替代手动 new

**场景**：根据条件创建不同类型的对象

```java
// 不需要自己写工厂类，直接用 Spring 的 @Autowired + Map 注入
@Component
@RequiredArgsConstructor
public class ExportServiceFactory {

    // key = Bean 名称，value = 对应实现
    private final Map<String, ExportService> exportServiceMap;

    public ExportService getExporter(String format) {
        String beanName = format.toLowerCase() + "ExportService";
        ExportService service = exportServiceMap.get(beanName);
        if (service == null) {
            throw new BusinessException("不支持的导出格式: " + format);
        }
        return service;
    }
}
```


### 设计模式使用决策表

| 你要解决的问题 | 用什么模式 | 一句话判断标准 |
|-------------|----------|-------------|
| if-else / switch 分支太多 | 策略模式 | 超过 3 个分支，每个分支是独立逻辑 |
| 多个流程骨架相同，个别步骤不同 | 模板方法 | 步骤固定，某些步骤由子类决定 |
| 做完一件事要触发多个后续操作 | Spring Events | 后续操作不需要返回值、不影响主流程 |
| 多个处理器按顺序处理 | 责任链 | 每个处理器决定"自己处理还是传给下一个" |
| 对象字段太多，创建过程复杂 | Builder | 字段 > 5 个，部分可选 |
| 根据类型创建不同对象 | 工厂 + Spring 容器 | 不要手动 new，让 Spring 自动管理 |

**禁止的用法**：
- 为了用模式而用模式（"加个设计模式让代码看起来高级"）
- 在只有 1~2 个实现时就建策略工厂（直接 if 就行）
- 在 Service 层手写单例（Spring Bean 天生单例）


## 二、工具类封装规范

### 总原则

- 工具类 = **无状态 + 静态方法 + 纯函数**，不持有 Spring Bean 依赖
- 如果工具方法需要依赖 Spring Bean（如 RedisTemplate），改用 **@Component 组件**，不要叫 XxxUtils
- 放在 `common/utils/` 目录下，按职责命名

### 工具类命名和目录

```
common/
├── utils/
│   ├── DateUtils.java          # 日期处理
│   ├── StringUtils.java        # 字符串处理
│   ├── MoneyUtils.java         # 金额计算（BigDecimal）
│   ├── IdCardUtils.java        # 身份证校验
│   ├── IpUtils.java            # IP 解析
│   └── TreeUtils.java          # 树形结构构建
├── helper/                     # 有状态、需要 Spring 依赖的"工具"组件
│   ├── RedisHelper.java        # Redis 操作封装（注入 RedisTemplate）
│   ├── ExcelHelper.java        # Excel 导入导出封装
│   └── SmsHelper.java          # 短信发送封装
```

**区分 Utils 和 Helper**：
| 类型 | 是否有状态 | 是否需要 Spring 依赖 | 命名后缀 | 示例 |
|------|----------|-------------------|---------|------|
| 工具类 | 无状态 | 不需要 | `Utils` | `DateUtils`、`MoneyUtils` |
| 帮助组件 | 有状态 | 需要 | `Helper` | `RedisHelper`、`ExcelHelper` |

### 工具类代码规范

```java
/**
 * 金额计算工具，所有金额运算必须用此工具，禁止用 double 运算
 */
public final class MoneyUtils {

    // 禁止实例化
    private MoneyUtils() {
        throw new UnsupportedOperationException("工具类不允许实例化");
    }

    /** 默认精度：2 位小数，四舍五入 */
    private static final int DEFAULT_SCALE = 2;
    private static final RoundingMode DEFAULT_MODE = RoundingMode.HALF_UP;

    /**
     * 金额加法
     * @param a 被加数
     * @param b 加数
     * @return 相加结果，保留 2 位小数
     */
    public static BigDecimal add(BigDecimal a, BigDecimal b) {
        return nullSafe(a).add(nullSafe(b)).setScale(DEFAULT_SCALE, DEFAULT_MODE);
    }

    /**
     * 金额乘法（计算总价等场景）
     * @param price 单价
     * @param quantity 数量
     * @return 金额
     */
    public static BigDecimal multiply(BigDecimal price, BigDecimal quantity) {
        return nullSafe(price).multiply(nullSafe(quantity)).setScale(DEFAULT_SCALE, DEFAULT_MODE);
    }

    /**
     * 元转分（支付接口通常用分）
     */
    public static long yuanToFen(BigDecimal yuan) {
        return nullSafe(yuan).multiply(BigDecimal.valueOf(100)).longValueExact();
    }

    /**
     * 分转元
     */
    public static BigDecimal fenToYuan(long fen) {
        return BigDecimal.valueOf(fen).divide(BigDecimal.valueOf(100), DEFAULT_SCALE, DEFAULT_MODE);
    }

    private static BigDecimal nullSafe(BigDecimal value) {
        return value != null ? value : BigDecimal.ZERO;
    }
}
```

**工具类规则**：
- 类必须 `final`，构造器 `private`
- 所有方法必须是 `public static`
- 参数必须做判空处理（返回安全默认值或抛明确异常）
- 必须写 Javadoc，说明参数含义和返回值
- 一个工具类只做一件事（单一职责）
- 禁止在工具类里注入 Spring Bean


## 三、Client 封装规范

### 总原则

- 所有**外部调用**（HTTP 接口、第三方 SDK、支付、短信、邮件）必须封装成独立的 Client 类
- **禁止在 Service 里直接用 RestTemplate / OkHttp / HttpClient 裸调外部接口**
- Client 统一放 `common/client/` 或 `modules/xxx/client/` 目录
- 调用方只关心方法名和参数，不关心底层用什么 HTTP 库

### 封装结构

```
common/
├── client/
│   ├── AbstractHttpClient.java     # 抽象基类（通用重试、日志、异常处理）
│   ├── config/
│   │   └── HttpClientConfig.java   # RestTemplate / OkHttp Bean 配置
│   ├── pay/
│   │   ├── AlipayClient.java       # 支付宝客户端
│   │   └── WechatPayClient.java    # 微信支付客户端
│   ├── sms/
│   │   └── AliyunSmsClient.java    # 阿里云短信客户端
│   └── oss/
│       └── AliyunOssClient.java    # 阿里云 OSS 客户端
```

### HTTP Client 基类封装

```java
/**
 * 外部 HTTP 调用基类，统一处理：重试、超时、日志、异常转换
 */
@Slf4j
public abstract class AbstractHttpClient {

    @Autowired
    protected RestTemplate restTemplate;

    /** 子类提供服务基础地址 */
    protected abstract String getBaseUrl();

    /** 默认请求超时时间（毫秒），子类可覆盖 */
    protected int getConnectTimeout() { return 3000; }

    /** 最大重试次数 */
    protected int getMaxRetries() { return 3; }

    /**
     * 发送 GET 请求
     * @param path    接口路径（不含 base url）
     * @param params  查询参数
     * @param respType 响应类型
     * @return 响应结果
     */
    protected <T> T get(String path, Map<String, Object> params, Class<T> respType) {
        String url = buildUrl(path, params);
        return executeWithRetry(() -> restTemplate.getForObject(url, respType));
    }

    /**
     * 发送 POST 请求
     */
    protected <T> T post(String path, Object body, Class<T> respType) {
        String url = buildUrl(path);
        return executeWithRetry(() -> restTemplate.postForObject(url, body, respType));
    }

    /**
     * 带重试的请求执行
     */
    private <T> T executeWithRetry(Callable<T> action) {
        Exception lastException = null;
        for (int i = 0; i <= getMaxRetries(); i++) {
            try {
                return action.call();
            } catch (Exception e) {
                lastException = e;
                log.warn("外部调用失败，第 {} 次重试: {}", i + 1, e.getMessage());
                if (i < getMaxRetries()) {
                    try { Thread.sleep((long) Math.pow(2, i) * 500); }
                    catch (InterruptedException ignored) { Thread.currentThread().interrupt(); }
                }
            }
        }
        throw new ExternalServiceException("外部服务调用失败: " + getBaseUrl(), lastException);
    }

    private String buildUrl(String path, Map<String, Object> params) {
        UriComponentsBuilder builder = UriComponentsBuilder.fromHttpUrl(getBaseUrl() + path);
        if (params != null) {
            params.forEach((k, v) -> {
                if (v != null) builder.queryParam(k, v);
            });
        }
        return builder.toUriString();
    }

    private String buildUrl(String path) {
        return getBaseUrl() + path;
    }
}
```

### 具体 Client 实现示例

```java
/**
 * 阿里云短信客户端
 * 配置项：aliyun.sms.access-key / secret-key / sign-name
 */
@Component
@Slf4j
public class AliyunSmsClient extends AbstractHttpClient {

    @Value("${aliyun.sms.access-key}")
    private String accessKey;

    @Value("${aliyun.sms.secret-key}")
    private String secretKey;

    @Value("${aliyun.sms.sign-name}")
    private String signName;

    @Override
    protected String getBaseUrl() {
        return "https://dysmsapi.aliyuncs.com";
    }

    /**
     * 发送短信验证码
     * @param phone       手机号
     * @param templateCode 模板编码
     * @param params       模板参数（如验证码）
     */
    public void sendSms(String phone, String templateCode, Map<String, String> params) {
        Map<String, Object> requestParams = new HashMap<>();
        requestParams.put("PhoneNumbers", phone);
        requestParams.put("SignName", signName);
        requestParams.put("TemplateCode", templateCode);
        requestParams.put("TemplateParam", JsonUtils.toJson(params));
        // 签名逻辑省略（实际需按阿里云文档计算）

        SmsResponse resp = get("/", requestParams, SmsResponse.class);
        if (!"OK".equals(resp.getCode())) {
            log.error("短信发送失败: phone={}, code={}, msg={}", phone, resp.getCode(), resp.getMessage());
            throw new ExternalServiceException("短信发送失败: " + resp.getMessage());
        }
        log.info("短信发送成功: phone={}, template={}", phone, templateCode);
    }
}
```

### RestTemplate 配置（只配一次，全局复用）

```java
@Configuration
public class HttpClientConfig {

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .setConnectTimeout(Duration.ofSeconds(3))
            .setReadTimeout(Duration.ofSeconds(10))
            .errorHandler(new CustomResponseErrorHandler())
            .build();
    }
}
```

### Client 封装规则清单

| 规则 | 说明 |
|------|------|
| **必须封装** | 所有外部 HTTP 调用、第三方 SDK 调用 |
| **禁止裸调** | Service 层禁止直接用 RestTemplate / OkHttp / HttpClient |
| **统一异常** | 外部调用失败一律抛 `ExternalServiceException`，不暴露第三方错误细节 |
| **配置外置** | URL、key、超时时间必须放 `application.yml`，用 `@Value` 注入 |
| **日志规范** | 入参 + 出参 + 耗时 + 异常，必须有日志，但敏感信息脱敏 |
| **超时控制** | 连接超时 3s，读取超时按业务场景设置（支付 10s，批量 60s） |
| **重试策略** | 幂等操作自动重试 3 次（指数退避），非幂等操作禁止自动重试 |
| **降级兜底** | 微服务场景用 Sentinel fallback，单体场景 try-catch 返回默认值 |

### 特殊 Client 场景

**Redis / ES / MQ 这类中间件**：不需要自己封装 Client，Spring Boot Starter 已经封装好了，直接注入使用即可

```java
// Redis 直接注入 StringRedisTemplate，不需要再包一层 Client
@RequiredArgsConstructor
public class CaptchaService {
    private final StringRedisTemplate redisTemplate;
}

// ES 直接注入 ElasticsearchRestTemplate
@RequiredArgsConstructor
public class SearchService {
    private final ElasticsearchRestTemplate esTemplate;
}
```


## 四、常用工具类参考（按需引入，不要重复造轮子）

| 工具 | 优先用谁 | 说明 |
|------|---------|------|
| 字符串处理 | Hutool `StrUtil` | `StrUtil.isBlank()`、`StrUtil.subBefore()` 等 |
| 日期处理 | Hutool `DateUtil` + Java 8 `LocalDateTime` | 不用 `SimpleDateFormat`（线程不安全） |
| 集合处理 | Hutool `CollUtil` + Java 8 Stream | `CollUtil.isEmpty()`、`stream().filter().collect()` |
| JSON 处理 | 项目统一用 Jackson（Spring 默认） | 禁止混用 Gson + Jackson |
| 金额计算 | 自定义 `MoneyUtils`（BigDecimal 封装） | 禁止用 `double` 算钱 |
| ID 生成 | 雪花算法（Hutool `IdUtil.getSnowflakeNextId()`） | 分布式环境必须用雪花 ID |
| Excel 导入导出 | EasyExcel（阿里开源） | 不用 POI 原生 API（内存占用大） |
| HTTP 调用 | 项目统一封装的 `AbstractHttpClient` | 禁止各模块各自引入不同 HTTP 库 |
| 加解密 | Hutool `SecureUtil` + `CryptoUtil` | AES / RSA / MD5 |
| 树形结构 | 自定义 `TreeUtils`（递归构建） | 部门、菜单、分类等树形数据 |

**禁止的用法**：
- 项目里同时存在 Hutool 工具 + 自己写的同功能工具类（重复造轮子）
- 自己封装 DateUtils 里全是 `SimpleDateFormat`（线程不安全）
- 算钱用 `double`（精度丢失）
- JSON 处理同时引入 Jackson 和 Gson


## 五、封装质量自检清单

每次封装工具类或 Client 时，对照以下清单自检：

- [ ] 类名清晰，看名字就知道做什么（`AliyunSmsClient` ✅ / `HttpClient2` ❌）
- [ ] 方法参数不超过 4 个，超过则用对象封装
- [ ] 所有 public 方法有 Javadoc
- [ ] 参数做了判空处理
- [ ] 异常被正确处理，不会抛出底层框架的原始异常给调用方
- [ ] 敏感信息（key、secret）从配置文件读取，不硬编码
- [ ] 有日志，能追踪调用链路（出参、入参、耗时）
- [ ] 如果是 Client：超时、重试、降级都考虑了
- [ ] 如果是工具类：无状态，不依赖 Spring 容器

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
