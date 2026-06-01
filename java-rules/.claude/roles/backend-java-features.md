# ===== Java 8/21 特性、Stream API、国际化规范 =====

> 全局生效，所有 Java 模块必须遵守

---

## 一、Stream API 规范

### 1. 适用场景

| 场景 | 推荐写法 |
|------|---------|
| 集合过滤 + 转换 | `list.stream().filter().map().collect()` |
| 分组统计 | `list.stream().collect(Collectors.groupingBy())` |
| 提取某个字段列表 | `list.stream().map(Entity::getField).collect()` |
| 去重 | `list.stream().distinct().collect()` |
| 排序 | `list.stream().sorted().collect()` |
| 求和/最大/最小 | `list.stream().mapToLong().sum()` |

### 2. 禁止滥用

```java
// 错误：简单遍历不需要 Stream（性能反而更差）
list.stream().forEach(item -> {
    item.setStatus(1);
});

// 正确：简单遍历用 for
for (Item item : list) {
    item.setStatus(1);
}

// 错误：嵌套 Stream 难以阅读
list.stream()
    .filter(a -> a.getBs().stream()
        .anyMatch(b -> b.getCs().stream()
            .anyMatch(c -> c.getName().equals("xxx"))))
    .collect(Collectors.toList());

// 正确：拆成多步，赋中间变量
List<A> result = list.stream()
    .filter(this::hasMatchingChild)
    .collect(Collectors.toList());
```

### 3. Collectors 常用收集器速查

```java
// 转 List
.collect(Collectors.toList())

// 转 Map（value 取对象本身，key 冲突时取第一个）
.collect(Collectors.toMap(User::getId, Function.identity(), (a, b) -> a))

// 分组
.collect(Collectors.groupingBy(Order::getStatus))

// 分组 + 计数
.collect(Collectors.groupingBy(Order::getStatus, Collectors.counting()))

// 分组 + 提取字段列表
.collect(Collectors.groupingBy(Order::getUserId, Collectors.mapping(Order::getOrderId, Collectors.toList())))

// 分区（满足条件 / 不满足条件）
.collect(Collectors.partitioningBy(item -> item.getPrice() > 100))

// 连接字符串
.collect(Collectors.joining(",", "[", "]"))

// 统计摘要（count、sum、min、max、average）
.collect(Collectors.summarizingLong(Product::getPrice))
```

### 4. 并行流注意

```java
// 并行流：只在数据量大（> 10000）且无共享状态时使用
list.parallelStream()
    .filter(...)
    .map(...)
    .collect(Collectors.toList());

// 注意事项：
// 1. 必须无副作用（不能在 lambda 中修改外部变量）
// 2. 必须无顺序依赖（结果不依赖处理顺序）
// 3. collect 操作本身是线程安全的，但 forEach 不是
```

### 5. 性能对比

| 数据量 | 推荐方式 | 原因 |
|--------|---------|------|
| < 100 条 | 都可以 | 差距可忽略 |
| 100 ~ 10000 条 | Stream 或 for | 性能差距不大 |
| > 10000 条 | for 循环或并行流 | 普通 Stream 有创建/拆箱开销 |
| 需要短路操作（findFirst、anyMatch） | Stream | Stream 天然支持短路 |

---

## 二、Java 8 特性规范

### 1. Optional 使用

**适用场景**：方法返回值可能为空时

```java
// 推荐：返回 Optional 包装
public Optional<User> findById(Long id) {
    return Optional.ofNullable(userMapper.selectById(id));
}

// 使用链式调用
String userName = userService.findById(userId)
    .map(User::getName)
    .orElse("匿名用户");

// 提前返回
public String getDeptName(Long userId) {
    return userService.findById(userId)
        .map(User::getDeptId)
        .flatMap(deptService::findById)
        .map(Dept::getName)
        .orElseThrow(() -> new BizException("用户部门不存在"));
}
```

**禁止用法**：

```java
// 错误：orElse(null) 违背了 Optional 的意义
String name = optional.orElse(null);

// 错误：Optional 作为方法参数（设计意图不清）
public void process(Optional<String> name) { ... }

// 错误：Optional 作为字段类型（序列化问题）
private Optional<String> name;

// 错误：直接 get() 不先判断
optional.get();  // 可能 NoSuchElementException
```

### 2. LocalDateTime 替代 Date

**核心理由**：Date/SimpleDateFormat 线程不安全，LocalDateTime 线程安全 + API 更清晰

```java
// 创建
LocalDateTime now = LocalDateTime.now();
LocalDateTime specific = LocalDateTime.of(2024, 6, 1, 10, 30);

// 格式化（线程安全）
String formatted = now.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));

// 解析（线程安全）
LocalDateTime parsed = LocalDateTime.parse("2024-06-01 10:30:00",
    DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));

// 计算
LocalDateTime tomorrow = now.plusDays(1);
Duration duration = Duration.between(startTime, endTime);
long seconds = duration.getSeconds();

// 只有日期 / 只有时间
LocalDate today = LocalDate.now();
LocalTime time = LocalTime.now();

// 转时间戳（毫秒）
long timestamp = now.atZone(ZoneId.systemDefault()).toInstant().toEpochMilli();

// 时间戳转 LocalDateTime
LocalDateTime fromTimestamp = LocalDateTime.ofInstant(
    Instant.ofEpochMilli(timestamp), ZoneId.systemDefault());
```

**数据库映射**：

```yaml
# application.yml（MyBatis-Plus 自动映射）
mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true
  # LocalDateTime 自动映射到 MySQL DATETIME 字段，无需额外配置
```

**前端传参格式化**：

```java
// Controller 中 @JsonFormat 注解
@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
private LocalDateTime createTime;

// 全局配置（推荐）
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
```

### 3. 函数式接口速查

| 接口 | 方法 | 签名 | 场景 |
|------|------|------|------|
| `Function<T,R>` | `apply` | `T → R` | 类型转换（User → UserVO） |
| `Predicate<T>` | `test` | `T → boolean` | 过滤条件（filter） |
| `Consumer<T>` | `accept` | `T → void` | 消费数据（遍历处理） |
| `Supplier<T>` | `get` | `() → T` | 延迟创建（懒加载） |
| `BiFunction<T,U,R>` | `apply` | `(T,U) → R` | 两个参数的转换 |
| `UnaryOperator<T>` | `apply` | `T → T` | 同类型转换（如字符串处理） |

### 4. 方法引用使用场景

```java
// 静态方法引用
list.stream().map(Integer::parseInt)

// 实例方法引用（对象::方法）
list.stream().map(String::trim)

// 构造方法引用
list.stream().map(UserVO::new)

// 类::实例方法（第一个参数作为调用者）
list.stream().sorted(String::compareToIgnoreCase)
```

---

## 三、Java 21 特性（升级后使用）

> 以下特性在 Java 21 中可用。如项目仍为 Java 8，先升级再使用。

### 升级检查清单（Java 8 → 21）

- [ ] Spring Boot 升级到 3.x（2.7.x 不支持 Java 21）
- [ ] javax.* → jakarta.*（包名全部替换）
- [ ] MyBatis-Plus 升级到 3.5.5+（支持 Java 17+）
- [ ] Lombok 升级到 1.18.30+
- [ ] MapStruct 升级到 1.5.5+
- [ ] 所有依赖检查 Java 21 兼容性
- [ ] CI/CD 环境安装 JDK 21
- [ ] 全量回归测试

### 1. Record 类（不可变数据载体）

```java
// 替代 Lombok @Value / @Data 的不可变对象
public record UserDTO(Long id, String name, String email) {}

// 使用
UserDTO user = new UserDTO(1L, "张三", "zhangsan@example.com");
user.name();  // 自动生成的访问器（不是 getName()）

// 可以加方法
public record Money(BigDecimal amount, String currency) {
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("货币类型不一致");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

**适用场景**：DTO、VO、事件对象、方法返回值
**不适用场景**：Entity（需要 setter）、需要继承的类

### 2. Sealed Classes（限制继承）

```java
// 只允许指定的子类继承
public sealed class PaymentResult permits SuccessResult, FailResult, PendingResult {}

public record SuccessResult(String transactionId) extends PaymentResult {}
public record FailResult(String errorCode, String errorMessage) extends PaymentResult {}
public record PendingResult(String orderId) extends PaymentResult {}
```

**适用场景**：状态机、策略模式中需要穷举所有子类的情况

### 3. Pattern Matching

**instanceof 增强**：

```java
// 旧写法
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// 新写法（直接绑定变量）
if (obj instanceof String s) {
    System.out.println(s.length());
}

// 可以在条件中直接使用
if (obj instanceof String s && s.length() > 5) {
    System.out.println("长字符串: " + s);
}
```

**switch 模式匹配**（Java 21 正式特性）：

```java
// 替代 if-else instanceof 链
String describe(Object obj) {
    return switch (obj) {
        case Integer i   -> "整数: " + i;
        case String s    -> "字符串: " + s;
        case Long l      -> "长整数: " + l;
        case null        -> "空值";
        default          -> "未知类型";
    };
}
```

### 4. Virtual Threads（虚拟线程）

**适用场景**：IO 密集型任务（HTTP 调用、数据库查询、文件读写）

```java
// 创建虚拟线程
Thread.startVirtualThread(() -> {
    // IO 操作
    String result = httpClient.send(request, BodyHandlers.ofString());
});

// 虚拟线程池（代替传统线程池，用于 IO 密集型）
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> futures = urls.stream()
        .map(url -> executor.submit(() -> fetchUrl(url)))
        .toList();

    List<String> results = futures.stream()
        .map(f -> f.get(10, TimeUnit.SECONDS))
        .toList();
}

// Spring Boot 3.2+ 配置启用虚拟线程
spring:
  threads:
    virtual:
      enabled: true
```

**虚拟线程 vs 传统线程池**：

| 维度 | 传统线程池 | 虚拟线程 |
|------|-----------|---------|
| 适用场景 | 通用 | IO 密集型 |
| 线程数量 | 有限（通常几十~几百） | 数百万（极轻量） |
| 创建成本 | 高（OS 级线程） | 极低（JVM 级） |
| 阻塞影响 | 阻塞 OS 线程（昂贵） | 阻塞虚拟线程（廉价） |
| CPU 密集型 | 适合 | **不适合**（会阻塞 carrier thread） |

**注意**：
- CPU 密集型任务不要用虚拟线程（加密、压缩、大量计算）
- synchronized 块内不要做阻塞操作（会 pin carrier thread），改用 ReentrantLock
- 不要池化虚拟线程（每个任务创建一个，用完即弃）

### 5. Sequenced Collections

```java
// List、Set、Map 新增首尾元素访问方法
SequencedCollection<String> list = new ArrayList<>(List.of("a", "b", "c"));
list.getFirst();  // "a"
list.getLast();   // "c"
list.addFirst("x");
list.addLast("z");
list.reversed();  // 反转视图

// LinkedHashMap 也支持
SequencedMap<String, Integer> map = new LinkedHashMap<>();
map.putFirst("oldest", 1);
map.putLast("newest", 3);
```

---

## 四、国际化（i18n）

### 1. 后端配置

```java
@Configuration
public class I18nConfig {

    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource source = new ReloadableResourceBundleMessageSource();
        source.setBasename("classpath:i18n/messages");
        source.setDefaultEncoding("UTF-8");
        source.setDefaultLocale(Locale.CHINA);
        return source;
    }

    @Bean
    public LocaleResolver localeResolver() {
        // 从请求头 Accept-Language 或自定义 Header 解析语言
        AcceptHeaderLocaleResolver resolver = new AcceptHeaderLocaleResolver();
        resolver.setDefaultLocale(Locale.CHINA);
        return resolver;
    }
}
```

### 2. 消息文件结构

```
src/main/resources/i18n/
├── messages_zh_CN.properties    # 中文（默认）
├── messages_en_US.properties    # 英文
└── messages_ja_JP.properties    # 日文
```

```properties
# messages_zh_CN.properties
user.not_found=用户不存在
order.amount.exceed=订单金额超出限制
param.required={0}不能为空

# messages_en_US.properties
user.not_found=User not found
order.amount.exceed=Order amount exceeds limit
param.required={0} is required
```

### 3. 使用方式

```java
@Service
public class UserService {

    @Resource
    private MessageSource messageSource;

    public UserVO getById(Long id) {
        User user = userMapper.selectById(id);
        if (user == null) {
            // 获取当前 Locale 对应的国际化消息
            String message = messageSource.getMessage(
                "user.not_found", null, LocaleContextHolder.getLocale());
            throw new BizException(message);
        }
        return BeanUtil.copyProperties(user, UserVO.class);
    }
}
```

### 4. 全局异常处理集成

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @Resource
    private MessageSource messageSource;

    @ExceptionHandler(BizException.class)
    public R<?> handleBizException(BizException e) {
        // 根据错误码获取国际化消息
        String message = messageSource.getMessage(
            e.getCode(), e.getArgs(), LocaleContextHolder.getLocale());
        return R.fail(e.getCode(), message);
    }
}
```

### 5. 前端国际化（vue-i18n）

```typescript
// src/i18n/index.ts
import { createI18n } from 'vue-i18n'
import zhCN from './locales/zh-CN'
import enUS from './locales/en-US'

export const i18n = createI18n({
    legacy: false,
    locale: localStorage.getItem('lang') || 'zh-CN',
    fallbackLocale: 'zh-CN',
    messages: { 'zh-CN': zhCN, 'en-US': enUS }
})

// src/i18n/locales/zh-CN.ts
export default {
    common: {
        search: '搜索',
        reset: '重置',
        confirm: '确认',
        cancel: '取消'
    },
    user: {
        notFound: '用户不存在'
    }
}
```

```vue
<template>
  <!-- 使用 -->
  <a-button>{{ $t('common.search') }}</a-button>
  <!-- 带参数 -->
  <span>{{ $t('param.required', ['用户名']) }}</span>
</template>
```

---

## 五、禁止事项

- **禁止简单遍历使用 Stream**（`list.forEach()` 比 `list.stream().forEach()` 更简洁高效）
- **禁止 Stream 中嵌套 Stream 超过 2 层**（可读性极差，拆成多步）
- **禁止 `Optional.orElse(null)`**（违背 Optional 设计初衷，用 `orElse(defaultValue)` 或 `orElseThrow`）
- **禁止 Optional 作为方法参数或类字段**
- **禁止使用 `Date` 和 `SimpleDateFormat`**（线程不安全，用 `LocalDateTime` + `DateTimeFormatter`）
- **禁止时区硬编码**（统一用 `ZoneId.systemDefault()` 或配置 `spring.jackson.time-zone`）
- **禁止 Java 21 升级后还用 `javax.*`**（必须换成 `jakarta.*`）
- **禁止虚拟线程用于 CPU 密集型任务**（会阻塞 carrier thread，性能反而更差）
- **禁止 Record 类用于需要可变状态的实体**
